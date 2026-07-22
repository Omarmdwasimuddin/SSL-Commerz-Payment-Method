## Complete Production-Ready SSLCommerz Payment Workflow [source code](https://github.com/Omarmdwasimuddin/payment)

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
# Connect to Postgres via the shared transaction-mode pooler (IPv4-only)
DATABASE_URL=""

# Connect to Postgres via the shared session-mode pooler (used for migrations)
DIRECT_URL=""


// Sandbox Store ID
SSLCOMMERZ_STORE_ID=
SSLCOMMERZ_STORE_PASSWORD=
SSLCOMMERZ_API_BASE_URL=https://sandbox.sslcommerz.com
APP_URL=http://localhost:3000
RECONCILIATION_SECRET=dev-secret-change-me
SSLCOMMERZ_MODE=live


LOG_LEVEL=debug
NODE_ENV=production
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

#### eslint.config.mjs
```bash
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";

const eslintConfig = defineConfig([
  ...nextVitals,
  ...nextTs,
  // Override default ignores of eslint-config-next.
  globalIgnores([
    // Default ignores of eslint-config-next:
    ".next/**",
    "out/**",
    "build/**",
    "app/generated/prisma/**",
    "next-env.d.ts",
  ]),
]);

export default eslintConfig;

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
  APP_URL: z.url("APP_URL must be a valid absolute URL"),
  LOG_LEVEL: z.string().min(1).default("info"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  RECONCILIATION_SECRET: z.string().min(1, "RECONCILIATION_SECRET is required"),
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

#### services/order.service.ts
```bash
import { OrderStatus, Prisma } from "@/app/generated/prisma";
import prisma from "@/lib/prisma";
import {
  createOrderSchema,
  type CreateOrderInput,
} from "@/validations/order.schema";

type OrderDbClient = typeof prisma | Prisma.TransactionClient;

export async function createOrder(input: CreateOrderInput) {
  const parsed = createOrderSchema.parse(input);
  const unitPrice = new Prisma.Decimal(parsed.unitPrice);
  const amount = unitPrice.mul(parsed.quantity);

  return prisma.order.create({
    data: {
      amount,
      status: OrderStatus.PENDING,
      items: {
        create: {
          productName: parsed.productName,
          quantity: parsed.quantity,
          unitPrice,
        },
      },
    },
    include: {
      items: true,
    },
  });
}

export async function getOrderWithTotal(orderId: string) {
  const order = await prisma.order.findUniqueOrThrow({
    where: { id: orderId },
    include: {
      items: true,
    },
  });

  const total = order.items.reduce(
    (sum, item) => sum.plus(item.unitPrice.mul(item.quantity)),
    new Prisma.Decimal(0),
  );

  return { order, total };
}

export async function markOrderPaid(
  orderId: string,
  tx?: Prisma.TransactionClient,
) {
  return updateOrderStatus(orderId, OrderStatus.PAID, tx);
}

export async function markOrderFailed(
  orderId: string,
  tx?: Prisma.TransactionClient,
) {
  return updateOrderStatus(orderId, OrderStatus.FAILED, tx);
}

export async function markOrderCancelled(
  orderId: string,
  tx?: Prisma.TransactionClient,
) {
  return updateOrderStatus(orderId, OrderStatus.CANCELLED, tx);
}

function updateOrderStatus(
  orderId: string,
  status: OrderStatus,
  tx?: Prisma.TransactionClient,
) {
  const db: OrderDbClient = tx ?? prisma;

  return db.order.update({
    where: { id: orderId },
    data: { status },
  });
}
```
---

#### services/payment.service.ts
```bash
import { OrderStatus, PaymentStatus, Prisma, WebhookSource } from "@/app/generated/prisma";
import { env } from "@/lib/env";
import prisma from "@/lib/prisma";
import { initSession, validateTransaction } from "@/lib/sslcommerz/client";
import { getOrderWithTotal, markOrderPaid, markOrderFailed, markOrderCancelled } from "@/services/order.service";
import { validationApiResponseSchema, ipnPayloadSchema } from "@/validations/payment.schema";
import { log } from "@/lib/logger";

export type InitiatePaymentResult = {
  gatewayUrl: string;
  tranId: string;
};

export class PaymentInitError extends Error {
  constructor(
    message: string,
    public readonly status: number,
  ) {
    super(message);
    this.name = "PaymentInitError";
  }
}

export async function initiatePayment(
  orderId: string,
  idempotencyKey?: string,
): Promise<InitiatePaymentResult> {
  const { order, total } = await getOrderWithTotal(orderId);

  if (order.status !== OrderStatus.PENDING) {
    throw new PaymentInitError("Order is not payable", 409);
  }

  if (idempotencyKey) {
    const cached = await prisma.idempotencyKey.findUnique({
      where: { key: idempotencyKey },
    });

    if (cached?.responseBody) {
      return cached.responseBody as InitiatePaymentResult;
    }

    if (cached) {
      throw new PaymentInitError("Payment initiation is already in progress", 409);
    }
  }

  const existingPayment = await prisma.payment.findFirst({
    where: {
      orderId,
      status: {
        in: [PaymentStatus.INITIATED, PaymentStatus.PENDING],
      },
    },
    orderBy: {
      createdAt: "desc",
    },
  });

  if (existingPayment) {
    if (!existingPayment.gatewayUrl) {
      throw new PaymentInitError(
        "Payment initiation is already in progress",
        409,
      );
    }

    // Reuse an active SSLCommerz session instead of invalidating/recreating it:
    // a refresh or retry should send the customer to the same unpaid attempt.
    const response = {
      gatewayUrl: existingPayment.gatewayUrl,
      tranId: existingPayment.tranId,
    };

    if (idempotencyKey) {
      await cacheIdempotentResponse(idempotencyKey, orderId, response);
    }

    return response;
  }

  if (idempotencyKey) {
    try {
      await prisma.idempotencyKey.create({
        data: {
          key: idempotencyKey,
          orderId,
        },
      });
    } catch (error) {
      if (isUniqueConstraintError(error)) {
        const cached = await prisma.idempotencyKey.findUnique({
          where: { key: idempotencyKey },
        });

        if (cached?.responseBody) {
          return cached.responseBody as InitiatePaymentResult;
        }

        throw new PaymentInitError(
          "Payment initiation is already in progress",
          409,
        );
      }

      throw error;
    }
  }

  const tranId = crypto.randomUUID();
  const payment = await prisma.payment.create({
    data: {
      orderId,
      tranId,
      amount: total,
      currency: order.currency,
      status: PaymentStatus.INITIATED,
    },
  });

  const initResponse = await initSession({
    tranId,
    amount: total.toFixed(2),
    currency: order.currency,
    ...buildCallbackUrls(),
    product: buildProductSnapshot(order.items),
  });

  const gatewayUrl = initResponse.GatewayPageURL;

  if (!gatewayUrl) {
    throw new PaymentInitError(
      initResponse.failedreason ?? "SSLCommerz did not return a gateway URL",
      502,
    );
  }

  await prisma.payment.update({
    where: { id: payment.id },
    data: {
      sessionKey: initResponse.sessionkey,
      gatewayUrl,
      rawInitResponse: toPrismaJson(initResponse),
    },
  });

  const response = { gatewayUrl, tranId };

  if (idempotencyKey) {
    await cacheIdempotentResponse(idempotencyKey, orderId, response);
  }

  return response;
}

export type ProcessGatewayCallbackResult = {
  paymentStatus: PaymentStatus | "not_found";
};

const TERMINAL_PAYMENT_STATUSES: PaymentStatus[] = [
  PaymentStatus.SUCCESS,
  PaymentStatus.FAILED,
  PaymentStatus.CANCELLED,
];

export async function processGatewayCallback(
  tranId: string,
  outcome: "success" | "fail" | "cancel",
  requestId: string,
  valId?: string,
): Promise<ProcessGatewayCallbackResult> {
  const payment = await prisma.payment.findUnique({
    where: { tranId },
  });

  if (!payment) {
    log.warn(
      { requestId, tranId, outcome },
      "Gateway callback received for unknown tran_id",
    );
    return { paymentStatus: "not_found" };
  }

  const source = getWebhookSource(outcome);

  if (TERMINAL_PAYMENT_STATUSES.includes(payment.status)) {
    await prisma.paymentEvent.create({
      data: {
        paymentId: payment.id,
        source,
        requestId,
        rawPayload: buildRawCallbackPayload(tranId, outcome, valId),
        resultStatus: "duplicate_ignored",
      },
    });

    log.info(
      { requestId, paymentId: payment.id, tranId, status: payment.status },
      "Duplicate gateway callback ignored",
    );

    return { paymentStatus: payment.status };
  }

  if (outcome === "cancel") {
    const cancelResult = await prisma.$transaction(async (tx) => {
      const updateResult = await tx.payment.updateMany({
        where: { id: payment.id, status: { notIn: TERMINAL_PAYMENT_STATUSES } },
        data: { status: PaymentStatus.CANCELLED },
      });

      if (updateResult.count === 0) {
        return { raced: true };
      }

      await markOrderCancelled(payment.orderId, tx);
      return { raced: false };
    });

    if (cancelResult.raced) {
      await prisma.paymentEvent.create({
        data: {
          paymentId: payment.id,
          source: WebhookSource.REDIRECT_CANCEL,
          requestId,
          rawPayload: buildRawCallbackPayload(tranId, outcome, valId),
          resultStatus: "duplicate_ignored",
        },
      });

      log.info(
        { requestId, paymentId: payment.id, tranId, status: payment.status },
        "Duplicate gateway cancel ignored",
      );

      return { paymentStatus: payment.status };
    }

    await prisma.paymentEvent.create({
      data: {
        paymentId: payment.id,
        source: WebhookSource.REDIRECT_CANCEL,
        requestId,
        rawPayload: buildRawCallbackPayload(tranId, outcome, valId),
        resultStatus: "cancelled",
      },
    });

    log.info(
      { requestId, paymentId: payment.id, tranId },
      "Payment cancelled via gateway redirect",
    );

    return { paymentStatus: PaymentStatus.CANCELLED };
  }

  const { paymentStatus, failReason } = await applyValidationOutcome(
    payment,
    valId,
    requestId,
  );

  await prisma.paymentEvent.create({
    data: {
      paymentId: payment.id,
      source,
      requestId,
      rawPayload: buildRawCallbackPayload(tranId, outcome, valId),
      resultStatus: failReason,
    },
  });

  return { paymentStatus };
}

type ApplyValidationResult = {
  paymentStatus: PaymentStatus;
  failReason: string;
};
export type ProcessIpnResult = {
  status: "success" | "fail";
};

export async function processIpn(
  rawPayload: string,
  requestId: string,
): Promise<ProcessIpnResult> {
  // PaymentEvent.paymentId is a required FK, so a payload that never
  // resolves to a known Payment (bad tran_id, unparseable JSON) can only
  // be logged via Pino here. The tran_id lookup therefore has to happen
  // before any PaymentEvent write, not after.
  let parsedPayload: Record<string, unknown>;
  try {
    parsedPayload = JSON.parse(rawPayload) as Record<string, unknown>;
  } catch {
    log.error(
      { requestId, rawPayload },
      "IPN payload is not valid JSON — cannot extract tran_id, skipping PaymentEvent creation",
    );
    return { status: "fail" };
  }

  // Step 2: Lightweight extraction of tran_id before any validation
  const tranId = parsedPayload.tran_id;
  if (typeof tranId !== "string" || !tranId) {
    log.warn(
      { requestId, tranId },
      "IPN payload missing or invalid tran_id — cannot look up Payment",
    );
    return { status: "fail" };
  }

  // Step 3: Look up Payment by tran_id
  const payment = await prisma.payment.findUnique({
    where: { tranId },
  });

  if (!payment) {
    log.warn(
      { requestId, tranId },
      "IPN received for unknown tran_id — cannot create PaymentEvent (FK constraint)",
    );
    return { status: "fail" };
  }

  // Step 4: Persist raw payload to PaymentEvent NOW (before schema validation)
  const paymentEvent = await prisma.paymentEvent.create({
    data: {
      paymentId: payment.id,
      source: WebhookSource.IPN,
      requestId,
      rawPayload: JSON.parse(JSON.stringify(parsedPayload)) as Prisma.InputJsonValue,
    },
  });

  // Step 5: Validate payload shape with ipnPayloadSchema
  const payloadValidation = ipnPayloadSchema.safeParse(parsedPayload);
  if (!payloadValidation.success) {
    await prisma.paymentEvent.update({
      where: { id: paymentEvent.id },
      data: { resultStatus: "invalid_payload", processedAt: new Date() },
    });
    log.warn(
      { requestId, paymentId: payment.id, tranId, issues: payloadValidation.error.issues },
      "IPN payload failed schema validation",
    );
    return { status: "fail" };
  }

  const ipnPayload = payloadValidation.data;

  // Step 6: If Payment status is already terminal, mark as duplicate and return
  if (TERMINAL_PAYMENT_STATUSES.includes(payment.status)) {
    await prisma.paymentEvent.update({
      where: { id: paymentEvent.id },
      data: { resultStatus: "duplicate_ignored", processedAt: new Date() },
    });
    log.info(
      { requestId, paymentId: payment.id, tranId, status: payment.status },
      "Duplicate IPN ignored — Payment already terminal",
    );
    return { status: "fail" };
  }

  // Step 7: Validate via Validation API and apply outcome
  const { paymentStatus, failReason } = await applyValidationOutcome(
    payment,
    ipnPayload.val_id,
    requestId,
  );

  // Step 8: Update PaymentEvent with final result
  await prisma.paymentEvent.update({
    where: { id: paymentEvent.id },
    data: { resultStatus: failReason, processedAt: new Date() },
  });

  return { status: paymentStatus === PaymentStatus.SUCCESS ? "success" : "fail" };
}

function getWebhookSource(
  outcome: "success" | "fail" | "cancel",
): WebhookSource {
  switch (outcome) {
    case "success":
      return WebhookSource.REDIRECT_SUCCESS;
    case "fail":
      return WebhookSource.REDIRECT_FAIL;
    case "cancel":
      return WebhookSource.REDIRECT_CANCEL;
    default:
      return WebhookSource.REDIRECT_FAIL;
  }
}

function buildRawCallbackPayload(
  tranId: string,
  outcome: string,
  valId?: string,
  extra?: Record<string, unknown>,
): Prisma.InputJsonValue {
  const payload: Record<string, unknown> = {
    tran_id: tranId,
    outcome,
    ...(valId ? { val_id: valId } : {}),
    ...extra,
  };
  return JSON.parse(JSON.stringify(payload)) as Prisma.InputJsonValue;
}

async function applyAtomicUpdate(
  payment: { id: string; orderId: string },
  status: PaymentStatus,
  failReason: string,
): Promise<ApplyValidationResult> {
  const result = await prisma.$transaction(async (tx) => {
    const updateResult = await tx.payment.updateMany({
      where: { id: payment.id, status: { notIn: TERMINAL_PAYMENT_STATUSES } },
      data: { status, failReason },
    });

    if (updateResult.count === 0) {
      return { raced: true };
    }

    if (status === PaymentStatus.SUCCESS) {
      await markOrderPaid(payment.orderId, tx);
    } else {
      await markOrderFailed(payment.orderId, tx);
    }

    return { raced: false };
  });

  if (result.raced) {
    return { paymentStatus: status, failReason: "duplicate_ignored" };
  }

  return { paymentStatus: status, failReason };
}

async function applyValidationOutcome(
  payment: { id: string; orderId: string; amount: Prisma.Decimal; currency: string; status: PaymentStatus },
  valId: string | undefined,
  requestId: string,
): Promise<ApplyValidationResult> {
  if (!valId) {
    return applyAtomicUpdate(payment, PaymentStatus.FAILED, "validation_failed");
  }

  let validationResult: unknown = null;
  try {
    validationResult = await validateTransaction(valId);
  } catch (error) {
    log.error(
      { requestId, paymentId: payment.id, error: serializeError(error) },
      "Validation API call failed",
    );
  }

  const parsed = validationApiResponseSchema.safeParse(validationResult);
  if (!parsed.success) {
    return applyAtomicUpdate(payment, PaymentStatus.FAILED, "validation_failed");
  }

  const data = parsed.data;
  const isValid = data.status === "VALID" || data.status === "VALIDATED";
  const amountMatches = new Prisma.Decimal(data.amount).equals(payment.amount);
  const currencyMatches = data.currency === payment.currency;

  if (isValid && amountMatches && currencyMatches) {
    return applyAtomicUpdate(payment, PaymentStatus.SUCCESS, "validated_success");
  }

  const failReason = !isValid ? "validation_failed" : "amount_mismatch";
  return applyAtomicUpdate(payment, PaymentStatus.FAILED, failReason);
}

export type ReconcileSummary = {
  checked: number;
  success: number;
  failed: number;
  unresolved: number;
};

export async function reconcileStuckPayments(
  olderThanMinutes: number = 30,
  limit: number = 100,
): Promise<ReconcileSummary> {
  olderThanMinutes = Math.max(5, Math.min(1440, olderThanMinutes));
  limit = Math.max(1, Math.min(100, limit));

  const cutoffDate = new Date(Date.now() - olderThanMinutes * 60_000);
  const noValIdCutoff = new Date(Date.now() - olderThanMinutes * 2 * 60_000);

  const stuckPayments = await prisma.payment.findMany({
    where: {
      status: { in: [PaymentStatus.INITIATED, PaymentStatus.PENDING] },
      createdAt: { lt: cutoffDate },
    },
    take: limit,
    orderBy: { createdAt: "asc" },
  });

  let checked = 0;
  let success = 0;
  let failed = 0;
  let unresolved = 0;

  for (const payment of stuckPayments) {
    checked++;

    if (payment.valId) {
      const result = await applyValidationOutcome(
        payment,
        payment.valId,
        crypto.randomUUID(),
      );
      if (result.paymentStatus === PaymentStatus.SUCCESS) success++;
      else if (result.paymentStatus === PaymentStatus.FAILED) failed++;
      else unresolved++;
    } else {
      // No valId means the user never completed the gateway checkout,
      // so the Validation API has nothing to validate. But some late
      // redirects or IPNs may still deliver valId, so we only auto-fail
      // payments that crossed a 2x timeout threshold. This gives the
      // async callback pipeline time to resolve the payment before
      // reconciliation declares it abandoned.
      if (payment.createdAt < noValIdCutoff) {
        const result = await applyAtomicUpdate(
          payment,
          PaymentStatus.FAILED,
          "no_val_id_stuck",
        );
        if (result.paymentStatus === PaymentStatus.FAILED) failed++;
        else unresolved++;
      } else {
        unresolved++;
      }
    }
  }

  return { checked, success, failed, unresolved };
}

export async function getPaymentWithOrder(tranId: string) {
  const payment = await prisma.payment.findUnique({
    where: { tranId },
    include: {
      order: {
        select: {
          id: true,
          amount: true,
          currency: true,
          status: true,
          items: {
            select: {
              productName: true,
            },
            take: 1,
          },
        },
      },
    },
  });

  if (!payment) {
    return null;
  }

  return {
    payment,
    productName: payment.order.items[0]?.productName ?? "Demo Product",
    amount: payment.amount,
    currency: payment.currency,
    tranId: payment.tranId,
  };
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}

async function cacheIdempotentResponse(
  key: string,
  orderId: string,
  response: InitiatePaymentResult,
) {
  await prisma.idempotencyKey.upsert({
    where: { key },
    create: {
      key,
      orderId,
      responseBody: response as Prisma.InputJsonValue,
    },
    update: {
      responseBody: response as Prisma.InputJsonValue,
    },
  });
}

function isUniqueConstraintError(error: unknown): boolean {
  return error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002";
}

function buildCallbackUrls() {
  return {
    successUrl: new URL("/api/payment/success", env.APP_URL).toString(),
    failUrl: new URL("/api/payment/fail", env.APP_URL).toString(),
    cancelUrl: new URL("/api/payment/cancel", env.APP_URL).toString(),
    ipnUrl: new URL("/api/payment/ipn", env.APP_URL).toString(),
  };
}

function buildProductSnapshot(
  items: { productName: string; quantity: number }[],
) {
  const totalQuantity = items.reduce((sum, item) => sum + item.quantity, 0);
  const name = items[0]?.productName ?? "Demo Product";
  return {
    name,
    category: "General",
    profile: "general",
    quantity: totalQuantity,
  };
}

function toPrismaJson<T>(value: T): Prisma.InputJsonValue {
  return JSON.parse(JSON.stringify(value));
}
```
---

#### app/api/payment/init/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { z } from "zod";
import { handlePrismaError } from "@/lib/errors/handlePrismaError";
import { log } from "@/lib/logger";
import { checkRateLimit, getClientIp } from "@/lib/rate-limit";
import {
  initiatePayment,
  PaymentInitError,
} from "@/services/payment.service";
import { initPaymentSchema } from "@/validations/payment.schema";

export async function POST(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "Payment init request received");

  try {
    const body = await request.json();
    const input = initPaymentSchema.parse(body);
    const rateLimitKey = getClientIp(request.headers);

    if (checkRateLimit(rateLimitKey)) {
      log.info({ requestId, path, method, rateLimitKey }, "Rate limit exceeded");

      return NextResponse.json(
        {
          status: "fail",
          requestId,
          message: "Too many payment initiation requests",
        },
        { status: 429 },
      );
    }

    const data = await initiatePayment(input.orderId, input.idempotencyKey);

    log.info(
      { requestId, path, method, orderId: input.orderId, tranId: data.tranId },
      "Payment init request completed",
    );

    return NextResponse.json({
      status: "success",
      requestId,
      message: "Payment initiated",
      data,
    });
  } catch (error) {
    const handled = toHttpError(error);

    log.error(
      { requestId, path, method, error: serializeError(error) },
      "Payment init request failed",
    );

    return NextResponse.json(
      {
        status: "fail",
        requestId,
        message: handled.message,
      },
      { status: handled.status },
    );
  }
}

function toHttpError(error: unknown) {
  if (error instanceof z.ZodError) {
    return { status: 400, message: "Invalid request body" };
  }

  if (error instanceof SyntaxError) {
    return { status: 400, message: "Invalid JSON body" };
  }

  if (error instanceof PaymentInitError) {
    return { status: error.status, message: error.message };
  }

  return handlePrismaError(error);
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }

  return { error };
}
```
---

#### app/api/payment/success/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { log } from "@/lib/logger";
import {
  processGatewayCallback,
  type ProcessGatewayCallbackResult,
} from "@/services/payment.service";
import { callbackParamsSchema } from "@/validations/payment.schema";

export async function GET(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "Payment callback received");

  try {
    const searchParams = request.nextUrl.searchParams;
    const rawParams = Object.fromEntries(searchParams.entries());
    const { tran_id } = callbackParamsSchema.parse(rawParams);

    const valId = searchParams.get("val_id") ?? undefined;

    const result = await processGatewayCallback(tran_id, "success", requestId, valId);

    log.info(
      { requestId, path, method, tranId: tran_id, status: result.paymentStatus },
      "Payment callback processed",
    );

    const redirectPath = getRedirectPath(result.paymentStatus);
    const url = new URL(redirectPath, request.url);
    url.searchParams.set("tran_id", tran_id);
    return NextResponse.redirect(url);
  } catch (error) {
    log.error(
      { requestId, path, method, error: serializeError(error) },
      "Payment callback failed",
    );

    return NextResponse.redirect(new URL("/checkout/fail", request.url));
  }
}

function getRedirectPath(
  status: ProcessGatewayCallbackResult["paymentStatus"],
): string {
  switch (status) {
    case "SUCCESS":
      return "/checkout/success";
    case "CANCELLED":
      return "/checkout/cancel";
    case "FAILED":
    case "not_found":
    default:
      return "/checkout/fail";
  }
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}
```
---

#### app/api/payment/fail/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { log } from "@/lib/logger";
import {
  processGatewayCallback,
  type ProcessGatewayCallbackResult,
} from "@/services/payment.service";
import { callbackParamsSchema } from "@/validations/payment.schema";

export async function GET(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "Payment callback received");

  try {
    const searchParams = request.nextUrl.searchParams;
    const rawParams = Object.fromEntries(searchParams.entries());
    const { tran_id } = callbackParamsSchema.parse(rawParams);

    const valId = searchParams.get("val_id") ?? undefined;

    const result = await processGatewayCallback(tran_id, "fail", requestId, valId);

    log.info(
      { requestId, path, method, tranId: tran_id, status: result.paymentStatus },
      "Payment callback processed",
    );

    const redirectPath = getRedirectPath(result.paymentStatus);
    const url = new URL(redirectPath, request.url);
    url.searchParams.set("tran_id", tran_id);
    return NextResponse.redirect(url);
  } catch (error) {
    log.error(
      { requestId, path, method, error: serializeError(error) },
      "Payment callback failed",
    );

    return NextResponse.redirect(new URL("/checkout/fail", request.url));
  }
}

function getRedirectPath(
  status: ProcessGatewayCallbackResult["paymentStatus"],
): string {
  switch (status) {
    case "SUCCESS":
      return "/checkout/success";
    case "CANCELLED":
      return "/checkout/cancel";
    case "FAILED":
    case "not_found":
    default:
      return "/checkout/fail";
  }
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}
```
---

#### app/api/payment/cancel/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { log } from "@/lib/logger";
import {
  processGatewayCallback,
  type ProcessGatewayCallbackResult,
} from "@/services/payment.service";
import { callbackParamsSchema } from "@/validations/payment.schema";

export async function GET(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "Payment callback received");

  try {
    const searchParams = request.nextUrl.searchParams;
    const rawParams = Object.fromEntries(searchParams.entries());
    const { tran_id } = callbackParamsSchema.parse(rawParams);

    const result = await processGatewayCallback(tran_id, "cancel", requestId);

    log.info(
      { requestId, path, method, tranId: tran_id, status: result.paymentStatus },
      "Payment callback processed",
    );

    const redirectPath = getRedirectPath(result.paymentStatus);
    const url = new URL(redirectPath, request.url);
    url.searchParams.set("tran_id", tran_id);
    return NextResponse.redirect(url);
  } catch (error) {
    log.error(
      { requestId, path, method, error: serializeError(error) },
      "Payment callback failed",
    );

    return NextResponse.redirect(new URL("/checkout/fail", request.url));
  }
}

function getRedirectPath(
  status: ProcessGatewayCallbackResult["paymentStatus"],
): string {
  switch (status) {
    case "SUCCESS":
      return "/checkout/success";
    case "CANCELLED":
      return "/checkout/cancel";
    case "FAILED":
    case "not_found":
    default:
      return "/checkout/fail";
  }
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}
```
---

#### app/api/payment/ipn/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { log } from "@/lib/logger";
import { processIpn } from "@/services/payment.service";

export async function POST(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "IPN received");

  try {
    const rawBody = await request.text();
    const result = await processIpn(rawBody, requestId);

    log.info(
      { requestId, path, method, status: result.status },
      "IPN processed",
    );

    return NextResponse.json(
      { status: result.status, requestId },
      { status: 200 },
    );
  } catch (error) {
    log.error(
      { requestId, path, method, error: serializeError(error) },
      "IPN handler failed",
    );

    return NextResponse.json(
      { status: "fail", requestId, message: "IPN processing failed" },
      { status: 200 },
    );
  }
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}
```
---

#### app/api/payment/reconcile/route.ts
```bash
import { NextResponse, type NextRequest } from "next/server";
import { timingSafeEqual } from "crypto";
import { log } from "@/lib/logger";
import { env } from "@/lib/env";
import { reconcileStuckPayments } from "@/services/payment.service";
import { reconcileSchema } from "@/validations/payment.schema";

export async function GET(request: NextRequest) {
  const requestId = crypto.randomUUID();
  const path = request.nextUrl.pathname;
  const method = request.method;

  log.info({ requestId, path, method }, "Reconciliation request received");

  const secret = request.headers.get("x-reconcile-secret");
  const expected = Buffer.from(env.RECONCILIATION_SECRET);
  const provided = Buffer.from(secret ?? "");

  const maxLength = Math.max(expected.length, provided.length);
  const expectedPadded = Buffer.alloc(maxLength);
  expected.copy(expectedPadded);
  const providedPadded = Buffer.alloc(maxLength);
  provided.copy(providedPadded);

  let isMatch = false;
  try {
    isMatch = timingSafeEqual(expectedPadded, providedPadded);
  } catch {
    // Swallow unexpected errors from timingSafeEqual.
  }

  if (!isMatch) {
    log.warn({ requestId, path, method }, "Reconciliation unauthorized");
    return NextResponse.json(
      {
        status: "fail",
        requestId,
        message: "Unauthorized",
      },
      { status: 401 },
    );
  }

  try {
    const params = {
      olderThanMinutes: Number(request.nextUrl.searchParams.get("olderThanMinutes") ?? 30),
      limit: Number(request.nextUrl.searchParams.get("limit") ?? 100),
    };

    const parsed = reconcileSchema.parse(params);

    const summary = await reconcileStuckPayments(
      parsed.olderThanMinutes,
      parsed.limit,
    );

    log.info(
      { requestId, path, method, summary },
      "Reconciliation completed",
    );

    return NextResponse.json({
      status: "success",
      requestId,
      message: "Reconciliation completed",
      data: summary,
    });
  } catch (error) {
    log.error(
      { requestId, path, method, error: serializeError(error) },
      "Reconciliation failed",
    );

    return NextResponse.json(
      {
        status: "fail",
        requestId,
        message: "Reconciliation failed",
      },
      { status: 500 },
    );
  }
}

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}
```
---

#### app/checkout/success/page.tsx
```bash
import { PaymentStatus } from "@/app/generated/prisma";
import { getPaymentWithOrder } from "@/services/payment.service";

function getStatusLabel(status: PaymentStatus | undefined): string {
  switch (status) {
    case PaymentStatus.SUCCESS:
      return "Success";
    case PaymentStatus.FAILED:
      return "Failed";
    case PaymentStatus.CANCELLED:
      return "Cancelled";
    case PaymentStatus.PENDING:
      return "Pending";
    case PaymentStatus.INITIATED:
      return "Initiated";
    default:
      return "Unknown";
  }
}

export default async function CheckoutSuccessPage({
  searchParams,
}: {
  searchParams: Promise<{ tran_id?: string }>;
}) {
  const { tran_id } = await searchParams;
  const data = tran_id ? await getPaymentWithOrder(tran_id) : null;
  const paymentStatus = data?.payment.status;
  const isSuccess = paymentStatus === PaymentStatus.SUCCESS;
  const isMismatch = !isSuccess;

  return (
    <div className="min-h-screen flex items-center justify-center p-8">
      <div className="max-w-md w-full border rounded-lg p-6 shadow-sm">
        <h1 className="text-2xl font-bold mb-2">
          {isMismatch ? "Payment Status Updated" : "Payment Success"}
        </h1>

        {isMismatch && (
          <div className="mb-4 p-3 bg-yellow-50 dark:bg-yellow-900/20 border border-yellow-200 dark:border-yellow-800 rounded">
            <p className="text-sm font-medium text-yellow-800 dark:text-yellow-200">
              The URL shows success, but your actual payment status is:
            </p>
            <p className="text-lg font-bold mt-1">{getStatusLabel(paymentStatus)}</p>
          </div>
        )}

        <div className="space-y-3">
          <div>
            <p className="text-sm text-zinc-500">Product</p>
            <p className="font-medium">{data?.productName ?? "Unknown"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Amount</p>
            <p className="font-medium">{data ? `${data.amount} ${data.currency}` : "—"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Payment Status</p>
            <p className="font-medium">{getStatusLabel(paymentStatus)}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Transaction ID</p>
            <p className="font-mono text-sm break-all">{data?.tranId ?? "—"}</p>
          </div>
        </div>
      </div>
    </div>
  );
}
```
---

#### app/checkout/fail/page.tsx
```bash
import { PaymentStatus } from "@/app/generated/prisma";
import { getPaymentWithOrder } from "@/services/payment.service";

function getStatusLabel(status: PaymentStatus | undefined): string {
  switch (status) {
    case PaymentStatus.SUCCESS:
      return "Success";
    case PaymentStatus.FAILED:
      return "Failed";
    case PaymentStatus.CANCELLED:
      return "Cancelled";
    case PaymentStatus.PENDING:
      return "Pending";
    case PaymentStatus.INITIATED:
      return "Initiated";
    default:
      return "Unknown";
  }
}

export default async function CheckoutFailPage({
  searchParams,
}: {
  searchParams: Promise<{ tran_id?: string }>;
}) {
  const { tran_id } = await searchParams;
  const data = tran_id ? await getPaymentWithOrder(tran_id) : null;
  const paymentStatus = data?.payment.status;
  const isFailed = paymentStatus === PaymentStatus.FAILED;
  const isMismatch = !isFailed;

  return (
    <div className="min-h-screen flex items-center justify-center p-8">
      <div className="max-w-md w-full border rounded-lg p-6 shadow-sm">
        <h1 className="text-2xl font-bold mb-2">
          {isMismatch ? "Payment Status Updated" : "Payment Failed"}
        </h1>

        {isMismatch && (
          <div className="mb-4 p-3 bg-yellow-50 dark:bg-yellow-900/20 border border-yellow-200 dark:border-yellow-800 rounded">
            <p className="text-sm font-medium text-yellow-800 dark:text-yellow-200">
              The URL shows fail, but your actual payment status is:
            </p>
            <p className="text-lg font-bold mt-1">{getStatusLabel(paymentStatus)}</p>
          </div>
        )}

        <div className="space-y-3">
          <div>
            <p className="text-sm text-zinc-500">Product</p>
            <p className="font-medium">{data?.productName ?? "Unknown"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Amount</p>
            <p className="font-medium">{data ? `${data.amount} ${data.currency}` : "—"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Payment Status</p>
            <p className="font-medium">{getStatusLabel(paymentStatus)}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Transaction ID</p>
            <p className="font-mono text-sm break-all">{data?.tranId ?? "—"}</p>
          </div>
        </div>
      </div>
    </div>
  );
}
```
---

#### app/checkout/cancel/page.tsx
```bash
import { PaymentStatus } from "@/app/generated/prisma";
import { getPaymentWithOrder } from "@/services/payment.service";

function getStatusLabel(status: PaymentStatus | undefined): string {
  switch (status) {
    case PaymentStatus.SUCCESS:
      return "Success";
    case PaymentStatus.FAILED:
      return "Failed";
    case PaymentStatus.CANCELLED:
      return "Cancelled";
    case PaymentStatus.PENDING:
      return "Pending";
    case PaymentStatus.INITIATED:
      return "Initiated";
    default:
      return "Unknown";
  }
}

export default async function CheckoutCancelPage({
  searchParams,
}: {
  searchParams: Promise<{ tran_id?: string }>;
}) {
  const { tran_id } = await searchParams;
  const data = tran_id ? await getPaymentWithOrder(tran_id) : null;
  const paymentStatus = data?.payment.status;
  const isCancelled = paymentStatus === PaymentStatus.CANCELLED;
  const isMismatch = !isCancelled;

  return (
    <div className="min-h-screen flex items-center justify-center p-8">
      <div className="max-w-md w-full border rounded-lg p-6 shadow-sm">
        <h1 className="text-2xl font-bold mb-2">
          {isMismatch ? "Payment Status Updated" : "Payment Cancelled"}
        </h1>

        {isMismatch && (
          <div className="mb-4 p-3 bg-yellow-50 dark:bg-yellow-900/20 border border-yellow-200 dark:border-yellow-800 rounded">
            <p className="text-sm font-medium text-yellow-800 dark:text-yellow-200">
              The URL shows cancel, but your actual payment status is:
            </p>
            <p className="text-lg font-bold mt-1">{getStatusLabel(paymentStatus)}</p>
          </div>
        )}

        <div className="space-y-3">
          <div>
            <p className="text-sm text-zinc-500">Product</p>
            <p className="font-medium">{data?.productName ?? "Unknown"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Amount</p>
            <p className="font-medium">{data ? `${data.amount} ${data.currency}` : "—"}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Payment Status</p>
            <p className="font-medium">{getStatusLabel(paymentStatus)}</p>
          </div>
          <div>
            <p className="text-sm text-zinc-500">Transaction ID</p>
            <p className="font-mono text-sm break-all">{data?.tranId ?? "—"}</p>
          </div>
        </div>
      </div>
    </div>
  );
}
```
---

#### app/actions.ts
```bash
"use server";

import { headers } from "next/headers";
import { checkRateLimit, getClientIp } from "@/lib/rate-limit";
import { log } from "@/lib/logger";
import { PaymentInitError } from "@/services/payment.service";
import { createOrder } from "@/services/order.service";
import { initiatePayment, type InitiatePaymentResult } from "@/services/payment.service";

function serializeError(error: unknown) {
  if (error instanceof Error) {
    return {
      name: error.name,
      message: error.message,
      stack: error.stack,
    };
  }
  return { error };
}

export async function buyNowAction(): Promise<InitiatePaymentResult> {
  const requestId = crypto.randomUUID();

  const requestHeaders = await headers();
  const rateLimitKey = getClientIp(requestHeaders);

  log.info({ requestId, rateLimitKey }, "Buy now action started");

  if (checkRateLimit(rateLimitKey)) {
    log.info({ requestId, rateLimitKey }, "Buy now rate limit exceeded");

    throw new PaymentInitError("Too many payment initiation requests", 429);
  }

  const order = await createOrder({
    productName: "Demo Product",
    quantity: 1,
    unitPrice: "10.00",
  });

  try {
    const result = await initiatePayment(order.id);

    log.info(
      { requestId, orderId: order.id, tranId: result.tranId },
      "Buy now action completed",
    );

    return result;
  } catch (error) {
    log.error(
      { requestId, orderId: order.id, error: serializeError(error) },
      "Buy now action failed",
    );

    throw error;
  }
}
```
---

#### lib/rate-limit.ts
```bash
const RATE_LIMIT_WINDOW_MS = 60_000;
const RATE_LIMIT_MAX_REQUESTS = 5;
const rateLimitBuckets = new Map<string, { count: number; resetAt: number }>();

export function getClientIp(
  headers: { get(name: string): string | null },
): string {
  const forwardedFor = headers.get("x-forwarded-for");
  if (forwardedFor) {
    return forwardedFor.split(",")[0].trim();
  }
  return headers.get("x-real-ip") ?? "unknown";
}

export function checkRateLimit(key: string): boolean {
  const now = Date.now();
  const bucket = rateLimitBuckets.get(key);

  if (!bucket || bucket.resetAt <= now) {
    rateLimitBuckets.set(key, {
      count: 1,
      resetAt: now + RATE_LIMIT_WINDOW_MS,
    });

    return false;
  }

  bucket.count += 1;
  return bucket.count > RATE_LIMIT_MAX_REQUESTS;
}
```
---

#### app/page.tsx
```bash
"use client";

import { buyNowAction } from "./actions";

export default function Home() {
  return (
    <div className="flex flex-col flex-1 items-center justify-center bg-zinc-50 font-sans dark:bg-black">
      <main className="flex flex-1 w-full max-w-3xl flex-col items-center justify-between py-32 px-16 bg-white dark:bg-black sm:items-start">
        <div className="flex flex-col items-center gap-6 text-center sm:items-start sm:text-left w-full">
          <div className="max-w-md w-full border rounded-lg p-6 shadow-sm">
            <div className="flex items-center gap-4">
              <div className="w-16 h-16 bg-zinc-200 dark:bg-zinc-800 rounded-lg flex items-center justify-center text-2xl font-bold">
                P
              </div>
              <div>
                <h2 className="text-lg font-semibold">Demo Product</h2>
                <p className="text-sm text-zinc-500">BDT 10.00</p>
              </div>
            </div>
            <button
              onClick={async () => {
                try {
                  const result = await buyNowAction();
                  window.location.href = result.gatewayUrl;
                } catch {
                  alert("Payment initiation failed. Please try again.");
                }
              }}
              className="w-full mt-6 bg-black text-white rounded-full h-10 font-medium transition-colors hover:bg-zinc-800 dark:bg-white dark:text-black dark:hover:bg-zinc-200"
            >
              Buy Now
            </button>
          </div>
        </div>
      </main>
    </div>
  );
}
```
---
