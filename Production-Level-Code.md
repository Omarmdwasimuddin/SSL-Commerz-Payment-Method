## Complete Production-Ready SSLCommerz Payment Workflow 

#### Architecture
```bash
payment-method/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── app/
│   ├── api/
│   │   └── payment/
│   │       ├── init/route.ts        → POST: create Payment, get SSLCommerz session
│   │       ├── success/route.ts     → POST: gateway redirect, validate + update, redirect
│   │       ├── fail/route.ts        → POST: gateway redirect, validate + update, redirect
│   │       ├── cancel/route.ts      → POST: gateway redirect, validate + update, redirect
│   │       ├── ipn/route.ts         → POST: webhook, validate + update, respond 200
│   │       └── reconcile/route.ts   → POST (internal): re-check stuck payments
│   ├── checkout/
│   │   ├── success/page.tsx         → re-fetches real status from DB, doesn't trust the URL
│   │   ├── fail/page.tsx
│   │   └── cancel/page.tsx
│   ├── layout.tsx
│   └── page.tsx                      → seeded/hardcoded "buy now" trigger for testing
│
├── lib/
│   ├── prisma.ts                     → Prisma client singleton
│   ├── logger.ts                     → Pino instance + redaction config
│   ├── env.ts                        → Zod-validated env, imported everywhere instead of raw process.env
│   ├── rate-limit.ts                 → rate limiter used by /api/payment/init
│   ├── errors/
│   │   └── handlePrismaError.ts
│   └── sslcommerz/
│       ├── client.ts                 → initSession(), validateTransaction() wrappers
│       └── types.ts                  → raw SSLCommerz request/response shapes
│
├── services/
│   ├── order.service.ts              → createOrder(), getOrderTotal()
│   └── payment.service.ts            → initiatePayment(), processGatewayCallback(),
│                                        processIpn(), reconcileStuckPayments()
│
├── validations/
│   ├── payment.schema.ts             → Zod schemas: initPayment, ipnPayload, callbackParams
│   ├── order.schema.ts
│   └── env.schema.ts
│
├── types/
│   └── sslcommerz.d.ts               → shared TS types for SSLCommerz payloads
│
├── .env
├── .env.example                       → committed, no real secrets — documents required vars
├── PROJECT-REQUIREMENT.md
└── package.json
```

---

#### prisma/schema.prisma
```bash
generator client {
  provider = "prisma-client-js"
  output   = "../app/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

// --------------------------------------------------------------------------
// Enums
// --------------------------------------------------------------------------

enum OrderStatus {
  PENDING // order created, no successful payment yet
  PAID // a Payment reached SUCCESS and was validated
  FAILED // payment attempt(s) failed, order not fulfilled
  CANCELLED // user cancelled before/during payment
}

enum PaymentGateway {
  SSLCOMMERZ
  // add more here later (BKASH, STRIPE, etc.) if the project ever expands
}

enum PaymentStatus {
  INITIATED // session created with gateway, user not redirected/paid yet
  PENDING // user redirected to gateway, awaiting callback/IPN
  SUCCESS // independently verified via Validation API
  FAILED // gateway or validation reported failure
  CANCELLED // user cancelled at the gateway
}

// Where a status-changing signal came from — critical for debugging
// "why did this payment change state" during a real incident.
enum WebhookSource {
  REDIRECT_SUCCESS
  REDIRECT_FAIL
  REDIRECT_CANCEL
  IPN
  VALIDATION_API // manual/reconciliation re-check
}

// --------------------------------------------------------------------------
// Core models
// --------------------------------------------------------------------------

// Deliberately minimal — this project is not about auth/user management.
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())

  orders Order[]
}

model Order {
  id     String @id @default(cuid())
  userId String?
  user   User?  @relation(fields: [userId], references: [id])

  // Decimal, never Float — money must never use floating point.
  amount   Decimal @db.Decimal(10, 2)
  currency String  @default("BDT")

  status OrderStatus @default(PENDING)

  items    OrderItem[]
  payments Payment[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([status])
  @@index([userId])
}

// Hardcoded/seeded product data is fine per project scope — the schema
// still models it properly so the amount-calculation logic is real.
model OrderItem {
  id      String @id @default(cuid())
  orderId String
  order   Order  @relation(fields: [orderId], references: [id], onDelete: Cascade)

  productName String
  quantity    Int     @default(1)
  unitPrice   Decimal @db.Decimal(10, 2)

  @@index([orderId])
}

// The heart of this project. One Order can have multiple Payment attempts
// (e.g. first attempt failed, user retried) — never overwrite/reuse a row
// across attempts, always create a new one with a new tran_id.
model Payment {
  id      String @id @default(cuid())
  orderId String
  order   Order  @relation(fields: [orderId], references: [id])

  gateway PaymentGateway @default(SSLCOMMERZ)

  // Your own generated transaction id (UUID) — sent to SSLCommerz as tran_id.
  tranId String @unique

  // Returned by SSLCommerz after a successful transaction; used to call
  // the Validation API. Nullable because it doesn't exist until the
  // gateway redirects/IPNs back.
  valId String? @unique

  sessionKey String? // returned by sessionapi at init time
  gatewayUrl String? // GatewayPageURL returned at init time

  // Snapshot of amount/currency at the time this Payment was created —
  // always compared against the live Order total during validation to
  // catch tampering or race conditions (e.g. order edited mid-payment).
  amount   Decimal @db.Decimal(10, 2)
  currency String  @default("BDT")

  status PaymentStatus @default(INITIATED)

  // Useful gateway metadata worth persisting for support/audit purposes.
  cardType   String?
  bankTranId String?

  // Full raw responses from SSLCommerz — never discard these. When a
  // payment dispute happens weeks later, this is what you go back to.
  rawInitResponse       Json?
  rawValidationResponse Json?

  failReason String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  events PaymentEvent[]

  @@index([orderId])
  @@index([status])
}

// Append-only audit trail of every callback/IPN/validation call this
// Payment has ever received. This is what makes a payment system
// debuggable — without it, "what actually happened to this transaction"
// is unanswerable after the fact.
model PaymentEvent {
  id        String  @id @default(cuid())
  paymentId String
  payment   Payment @relation(fields: [paymentId], references: [id], onDelete: Cascade)

  source    WebhookSource
  requestId String? // correlates back to the API route's log lines

  rawPayload Json // exactly what was received, unmodified

  // What this event resulted in, filled in after processing —
  // e.g. "validated_success", "duplicate_ignored", "amount_mismatch".
  resultStatus String?
  processedAt  DateTime?

  createdAt DateTime @default(now())

  @@index([paymentId])
  @@index([source])
}

// Guards POST /api/payment/init against duplicate submissions
// (double-click, network retry, etc.) producing duplicate Payment rows
// for the same order/request.
model IdempotencyKey {
  id     String  @id @default(cuid())
  key    String  @unique // e.g. hash of (orderId + userId) or a client-supplied key
  orderId String?

  // Cache the response so a retried request with the same key gets the
  // exact same result instead of re-running side effects.
  responseBody Json?

  createdAt DateTime  @default(now())
  expiresAt DateTime?

  @@index([expiresAt])
}
```

---

#### .env.example
```bash
SSLCOMMERZ_STORE_ID=your_store_id
SSLCOMMERZ_STORE_PASSWORD=your_store_password

# sandbox: https://sandbox.sslcommerz.com
# live:    https://securepay.sslcommerz.com
SSLCOMMERZ_API_BASE_URL=https://sandbox.sslcommerz.com

APP_URL=http://localhost:3000
```

---

#### lib/prisma.ts
```bash
import { PrismaClient } from "@/app/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
import { env } from "@/lib/env";

const globalForPrisma = globalThis as unknown as {
  prisma?: PrismaClient;
};

const adapter = new PrismaPg({
  connectionString: env.DATABASE_URL,
});

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    adapter,
    log: ["info", "warn", "error"],
  });

if (env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}

export default prisma;

```

---

#### lib/logger.ts
```bash
import pino from "pino";
import { env } from "@/lib/env";

export const log = pino({
  level: env.LOG_LEVEL,
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ["cus_email", "cus_phone", "store_passwd", "card", "card.*"],
    censor: "[redacted]",
  },
});

```
---

#### lib/errors/handlePrismaError.ts
```bash
import { Prisma } from "@/app/generated/prisma";

export function handlePrismaError(error: unknown): {
  status: number;
  message: string;
} {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    switch (error.code) {
      case "P2002":
        return {
          status: 409,
          message: "A record with this value already exists",
        };
      case "P2003":
        return { status: 400, message: "Referenced record does not exist" };
      case "P2025":
        return { status: 404, message: "Record not found" };
      default:
        return { status: 500, message: "Something went wrong" };
    }
  }

  return { status: 500, message: "Something went wrong" };
}

```
---

#### lib/sslcommerz/types.ts
```bash
type SslCommerzPrimitive = string | number | boolean | null;

export type SessionRequestPayload = {
  store_id: string;
  store_passwd: string;
  total_amount: string;
  currency: string;
  tran_id: string;
  success_url: string;
  fail_url: string;
  cancel_url: string;
  ipn_url: string;
  product_name: string;
  product_category: string;
  product_profile: string;
  cus_name: string;
  cus_email: string;
  cus_add1: string;
  cus_city: string;
  cus_postcode: string;
  cus_country: string;
  cus_phone: string;
  shipping_method: string;
  num_of_item: string;
  ship_name: string;
  ship_add1: string;
  ship_city: string;
  ship_postcode: string;
  ship_country: string;
  value_a?: string;
  value_b?: string;
  value_c?: string;
  value_d?: string;
};

export type SessionResponse = {
  status?: string;
  failedreason?: string;
  sessionkey?: string;
  GatewayPageURL?: string;
  storeBanner?: string;
  storeLogo?: string;
  desc?: SslCommerzPrimitive[];
  is_direct_pay_enable?: string;
  directPaymentURL?: string;
  redirectGatewayURL?: string;
  redirectGatewayURLFailed?: string;
  GatewayPageURLFailed?: string;
  [key: string]: unknown;
};

export type ValidationResponse = {
  status?: string;
  tran_date?: string;
  tran_id?: string;
  val_id?: string;
  amount?: string;
  store_amount?: string;
  currency?: string;
  bank_tran_id?: string;
  card_type?: string;
  card_no?: string;
  card_issuer?: string;
  card_brand?: string;
  card_issuer_country?: string;
  card_issuer_country_code?: string;
  currency_type?: string;
  currency_amount?: string;
  currency_rate?: string;
  base_fair?: string;
  value_a?: string;
  value_b?: string;
  value_c?: string;
  value_d?: string;
  risk_title?: string;
  risk_level?: string;
  validated_on?: string;
  verify_sign?: string;
  verify_key?: string;
  error?: string;
  [key: string]: unknown;
};

export type IpnPayload = {
  status?: string;
  tran_date?: string;
  tran_id?: string;
  val_id?: string;
  amount?: string;
  store_amount?: string;
  currency?: string;
  bank_tran_id?: string;
  card_type?: string;
  card_no?: string;
  card_issuer?: string;
  card_brand?: string;
  card_issuer_country?: string;
  card_issuer_country_code?: string;
  currency_type?: string;
  currency_amount?: string;
  currency_rate?: string;
  base_fair?: string;
  value_a?: string;
  value_b?: string;
  value_c?: string;
  value_d?: string;
  risk_title?: string;
  risk_level?: string;
  verify_sign?: string;
  verify_key?: string;
  [key: string]: string | undefined;
};

export type InitSessionParams = {
  tranId: string;
  amount: string | number;
  currency: string;
  successUrl: string;
  failUrl: string;
  cancelUrl: string;
  ipnUrl: string;
  product?: {
    name?: string;
    category?: string;
    profile?: string;
    quantity?: number;
  };
  customer?: {
    name?: string;
    email?: string;
    address?: string;
    city?: string;
    postcode?: string;
    country?: string;
    phone?: string;
  };
  shipping?: {
    method?: string;
    name?: string;
    address?: string;
    city?: string;
    postcode?: string;
    country?: string;
  };
};
```
---

#### lib/sslcommerz/client.ts
```bash
import { env } from "@/lib/env";
import { log } from "@/lib/logger";
import type {
  InitSessionParams,
  SessionRequestPayload,
  SessionResponse,
  ValidationResponse,
} from "@/lib/sslcommerz/types";

const sslCommerzBaseUrls = {
  sandbox: "https://sandbox.sslcommerz.com",
  live: "https://securepay.sslcommerz.com",
} as const;

function getBaseUrl() {
  return sslCommerzBaseUrls[env.SSLCOMMERZ_MODE];
}

function stringifyAmount(amount: string | number) {
  return typeof amount === "number" ? amount.toFixed(2) : amount;
}

function redactStorePassword<T extends { store_passwd?: unknown }>(payload: T) {
  return {
    ...payload,
    store_passwd:
      payload.store_passwd === undefined ? undefined : "[redacted]",
  };
}

function toFormBody(payload: SessionRequestPayload) {
  const body = new URLSearchParams();

  for (const [key, value] of Object.entries(payload)) {
    if (value !== undefined) {
      body.set(key, value);
    }
  }

  return body;
}

async function readJsonResponse<T>(response: Response, context: string): Promise<T> {
  const text = await response.text();

  if (!response.ok) {
    throw new Error(
      `${context} failed with HTTP ${response.status}: ${text || response.statusText}`,
    );
  }

  try {
    return JSON.parse(text) as T;
  } catch {
    throw new Error(`${context} returned invalid JSON`);
  }
}

function buildSessionPayload(params: InitSessionParams): SessionRequestPayload {
  const customer = params.customer ?? {};
  const shipping = params.shipping ?? {};
  const product = params.product ?? {};

  return {
    store_id: env.SSLCOMMERZ_STORE_ID,
    store_passwd: env.SSLCOMMERZ_STORE_PASSWORD,
    total_amount: stringifyAmount(params.amount),
    currency: params.currency,
    tran_id: params.tranId,
    success_url: params.successUrl,
    fail_url: params.failUrl,
    cancel_url: params.cancelUrl,
    ipn_url: params.ipnUrl,
    product_name: product.name ?? "Demo Product",
    product_category: product.category ?? "General",
    product_profile: product.profile ?? "general",
    cus_name: customer.name ?? "Demo Customer",
    cus_email: customer.email ?? "customer@example.com",
    cus_add1: customer.address ?? "Dhaka",
    cus_city: customer.city ?? "Dhaka",
    cus_postcode: customer.postcode ?? "1207",
    cus_country: customer.country ?? "Bangladesh",
    cus_phone: customer.phone ?? "01700000000",
    shipping_method: shipping.method ?? "NO",
    num_of_item: String(product.quantity ?? 1),
    ship_name: shipping.name ?? customer.name ?? "Demo Customer",
    ship_add1: shipping.address ?? customer.address ?? "Dhaka",
    ship_city: shipping.city ?? customer.city ?? "Dhaka",
    ship_postcode: shipping.postcode ?? customer.postcode ?? "1207",
    ship_country: shipping.country ?? customer.country ?? "Bangladesh",
  };
}

export async function initSession(
  params: InitSessionParams,
): Promise<SessionResponse> {
  const endpoint = `${getBaseUrl()}/gwprocess/v4/api.php`;
  const payload = buildSessionPayload(params);

  log.info(
    { endpoint, payload: redactStorePassword(payload) },
    "Calling SSLCommerz session API",
  );

  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      "content-type": "application/x-www-form-urlencoded",
    },
    body: toFormBody(payload),
  });

  const data = await readJsonResponse<SessionResponse>(
    response,
    "SSLCommerz session API",
  );

  log.info({ endpoint, response: data }, "Received SSLCommerz session response");

  return data;
}

export async function validateTransaction(
  valId: string,
): Promise<ValidationResponse> {
  const endpoint = new URL(
    `${getBaseUrl()}/validator/api/validationserverAPI.php`,
  );

  endpoint.searchParams.set("val_id", valId);
  endpoint.searchParams.set("store_id", env.SSLCOMMERZ_STORE_ID);
  endpoint.searchParams.set("store_passwd", env.SSLCOMMERZ_STORE_PASSWORD);
  endpoint.searchParams.set("v", "1");
  endpoint.searchParams.set("format", "json");

  log.info(
    {
      endpoint: endpoint.origin + endpoint.pathname,
      query: redactStorePassword(Object.fromEntries(endpoint.searchParams)),
    },
    "Calling SSLCommerz validation API",
  );

  const response = await fetch(endpoint);
  const data = await readJsonResponse<ValidationResponse>(
    response,
    "SSLCommerz validation API",
  );

  log.info(
    { endpoint: endpoint.origin + endpoint.pathname, response: data },
    "Received SSLCommerz validation response",
  );

  return data;
}

```
---

#### validations/order.schema.ts
```bash
import { z } from "zod";

const decimalStringSchema = z
  .string()
  .regex(/^\d+(\.\d{1,2})?$/, "Must be a decimal amount with up to 2 places");

export const createOrderSchema = z.object({
  productName: z.string().trim().min(1, "Product name is required"),
  quantity: z.coerce.number().int().positive(),
  unitPrice: z.union([
    decimalStringSchema,
    z.coerce.number().positive().finite(),
  ]),
});

export type CreateOrderInput = z.infer<typeof createOrderSchema>;

```
---

#### validations/payment.schema.ts
```bash
import { z } from "zod";

const decimalStringSchema = z
  .string()
  .regex(/^\d+(\.\d{1,2})?$/, "Must be a decimal amount with up to 2 places");

const gatewayStatusSchema = z.string().trim().min(1);
const validGatewayStatusSchema = z.enum(["VALID", "VALIDATED"]);

// Unknown keys are stripped on purpose: callers may send an amount, but the
// service must only use the server-side Order/Payment amount from the database.
export const initPaymentSchema = z
  .object({
    orderId: z.string().cuid(),
    idempotencyKey: z.string().trim().min(1).max(255).optional(),
  })
  .strip();

export type InitPaymentInput = z.infer<typeof initPaymentSchema>;

export const callbackParamsSchema = z.object({
  tran_id: z.string().uuid(),
});

export type CallbackParams = z.infer<typeof callbackParamsSchema>;

export const ipnPayloadSchema = z
  .object({
    tran_id: z.string().min(1),
    val_id: z.string().min(1).optional(),
    status: gatewayStatusSchema,
    amount: decimalStringSchema,
    store_amount: decimalStringSchema.optional(),
    currency: z.string().min(1),
    card_type: z.string().min(1).optional(),
    bank_tran_id: z.string().min(1).optional(),
  })
  .passthrough();

export type IpnPayloadInput = z.infer<typeof ipnPayloadSchema>;

export const validationApiResponseSchema = z
  .object({
    status: validGatewayStatusSchema,
    tran_id: z.string().min(1),
    val_id: z.string().min(1),
    amount: decimalStringSchema,
    store_amount: decimalStringSchema.optional(),
    currency: z.string().min(1),
    card_type: z.string().min(1).optional(),
    bank_tran_id: z.string().min(1).optional(),
  })
  .passthrough();

export type ValidationApiResponseInput = z.infer<
  typeof validationApiResponseSchema
>;

```
---

#### validations/env.schema.ts
```bash
import { z } from "zod";

export const envSchema = z.object({
  DATABASE_URL: z.string().min(1, "DATABASE_URL is required"),
  SSLCOMMERZ_STORE_ID: z.string().min(1, "SSLCOMMERZ_STORE_ID is required"),
  SSLCOMMERZ_STORE_PASSWORD: z
    .string()
    .min(1, "SSLCOMMERZ_STORE_PASSWORD is required"),
  SSLCOMMERZ_MODE: z.enum(["sandbox", "live"], {
    error: "SSLCOMMERZ_MODE must be either 'sandbox' or 'live'",
  }),
  LOG_LEVEL: z.string().min(1).default("info"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
});

export type Env = z.infer<typeof envSchema>;

```
---

#### lib/env.ts
```bash
import "dotenv/config";
import { z } from "zod";
import { envSchema } from "@/validations/env.schema";

const parsedEnv = envSchema.safeParse(process.env);

if (!parsedEnv.success) {
  const missingOrInvalid = z.prettifyError(parsedEnv.error);

  throw new Error(
    `Invalid environment configuration:\n${missingOrInvalid}`,
  );
}

// Import this validated object from server-side code instead of reading
// process.env directly, so missing payment/database secrets fail at startup.
export const env = parsedEnv.data;

```
---
