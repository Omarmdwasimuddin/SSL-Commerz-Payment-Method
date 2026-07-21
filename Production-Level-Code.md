## Complete Production-Ready SSLCommerz Payment Workflow 

#### Architecture
```bash
prisma/
- schema.prisma            → User + Product + Order + Transaction model

lib/
- prisma.ts                 → Prisma client singleton
- logger.ts                  → Structured logger
- validations/payment.schema.ts → Zod validation
- services/sslcommerz.service.ts → SSLCommerz API calls (init + validation)
- services/order.service.ts  → Shared transaction-finalize logic (success + ipn duitai use kore)

app/api/
- orders/route.ts             → Order create (checkout)
- payment/init/route.ts       → Payment session start (existing orderId diye)
- payment/success/route.ts    → Success callback (server-verified)
- payment/fail/route.ts       → Fail callback
- payment/cancel/route.ts     → Cancel callback
- payment/ipn/route.ts        → IPN (background, most reliable)
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

enum OrderStatus {
  PENDING   // order created, waiting for payment
  PAID      // payment verified (via Transaction)
  FAILED    // SSLCommerz fail callback
  CANCELLED // SSLCommerz cancel callback
}

enum TransactionStatus {
  INITIATED // SSLCommerz session create hoise, ekhono pay hoyni
  VALID     // Validation API confirm korse — final "paid" state
  FAILED
  CANCELLED
  EXPIRED
}

model User {
  id           String   @id @default(cuid())
  name         String
  email        String   @unique
  passwordHash String?   
  phone        String?
  address      String?
  city         String?

  orders       Order[]

  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  @@map("users")
}

model Product {
  id          String   @id @default(cuid())
  name        String
  description String?
  price       Decimal  @db.Decimal(10, 2)
  imageUrl    String?
  stock       Int      @default(100)

  orders      Order[]

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("products")
}

model Order {
  id              String        @id @default(cuid())

  userId          String
  user            User          @relation(fields: [userId], references: [id])

  // Relation to product
  productId       String
  product         Product       @relation(fields: [productId], references: [id])

  quantity        Int           @default(1)
  totalAmount     Decimal       @db.Decimal(10, 2)
  currency        String        @default("BDT")

  // Customer info snapshot — order er shomoy ja chilo, freeze kore rakha
  // (User profile porey update hole o order history accurate thake)
  customerName    String
  customerEmail   String
  customerPhone   String
  customerAddress String?
  customerCity    String?

  status          OrderStatus   @default(PENDING)

  transactions    Transaction[]

  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  @@index([userId])
  @@index([status])
  @@map("orders")
}

model Transaction {
  id                     String            @id @default(cuid())
  tranId                 String            @unique   // duplicate protection-er backbone

  orderId                String
  order                  Order             @relation(fields: [orderId], references: [id])

  amount                 Decimal           @db.Decimal(10, 2)
  currency               String            @default("BDT")
  status                 TransactionStatus @default(INITIATED)

  sessionKey             String?
  gatewayPageURL         String?

  valId                  String?
  bankTranId             String?
  cardType               String?           // e.g. VISA, brac_visa, bkash ইত্যাদি
  cardIssuer             String?
  storeAmount            Decimal?          @db.Decimal(10, 2)

  // Full raw payloads — audit trail o debugging er jonno
  rawInitResponse        Json?
  rawValidationResponse  Json?
  rawIpnPayload          Json?

  processedAt            DateTime?         // idempotency guard

  createdAt              DateTime          @default(now())
  updatedAt              DateTime          @updatedAt

  @@index([orderId])
  @@index([status])
  @@index([valId])
  @@map("transactions")
}
```

---

#### .env
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

const globalForPrisma = global as unknown as { 
    prisma: PrismaClient;
 };

 const adapter = new PrismaPg({
    connectionString: process.env.DATABASE_URL,
 });

 const prisma = globalForPrisma.prisma || new PrismaClient({ adapter, log: ['info', 'query', 'error', 'warn'] });

 if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

 export default prisma;
```

---

#### lib/logger.ts
```bash
import pino from "pino";

export const log = pino({
    level: process.env.LOG_LEVEL || "info",
    timestamp: pino.stdTimeFunctions.isoTime,

    transport:
        process.env.NODE_ENV === "development"
            ? {
                  target: "pino-pretty",
                  options: {
                      colorize: true,
                      translateTime: "SYS:standard",
                  },
              }
            : undefined,
});
```

---

#### lib/validations/payment.schema.ts
```bash
import { z } from "zod";

// Checkout — order create korar jonno. Price kokhono client theke nei na.
export const createOrderSchema = z.object({
    productId: z.string().min(1, "productId is required").cuid("Invalid productId"),
    quantity: z.number().int().min(1, "Quantity must be at least 1").max(100, "Quantity too high"),
    customerName: z.string().min(2, "Name is required"),
    customerEmail: z.string().email("Invalid email"),
    customerPhone: z.string().min(11, "Invalid phone number").max(15),
    customerAddress: z.string().optional(),
    customerCity: z.string().optional(),
});
export type CreateOrderInput = z.infer<typeof createOrderSchema>;

// Payment session start — shudhu existing orderId nei
export const initPaymentSchema = z.object({
    orderId: z.string().min(1, "orderId is required").cuid("Invalid orderId format"),
});
export type InitPaymentInput = z.infer<typeof initPaymentSchema>;

export const sslCallbackSchema = z.object({
    tran_id: z.string().min(1, "tran_id missing in callback"),
    val_id: z.string().optional(),
    amount: z.string().optional(),
    card_type: z.string().optional(),
    status: z.string().optional(),
});
export type SslCallbackInput = z.infer<typeof sslCallbackSchema>;

export const ipnPayloadSchema = z.object({
    tran_id: z.string().min(1, "tran_id missing in IPN"),
    val_id: z.string().min(1, "val_id missing in IPN"),
    status: z.string().optional(),
    amount: z.string().optional(),
});
export type IpnPayloadInput = z.infer<typeof ipnPayloadSchema>;
```

---

#### lib/services/sslcommerz.service.ts
```bash
import { z } from "zod";
import { log } from "@/lib/logger";

/* ------------------------------------------------------------------ *
 * Constants
 * ------------------------------------------------------------------ */

const GATEWAY_STATUS = {
    SUCCESS: "SUCCESS",
    FAILED: "FAILED",
} as const;

const VALIDATION_STATUS = {
    VALID: "VALID",
    VALIDATED: "VALIDATED",
    INVALID_TRANSACTION: "INVALID_TRANSACTION",
    FAILED: "FAILED",
    EXPIRED: "EXPIRED",
} as const;

const DEFAULTS = {
    COUNTRY: "Bangladesh",
    POSTCODE: "1000",
    SHIPPING_METHOD: "NO",
    NUM_OF_ITEM: "1",
    PRODUCT_CATEGORY: "E-commerce",
    PRODUCT_PROFILE: "general",
    AMOUNT_EPSILON: 0.01,
} as const;

const HTTP = {
    INIT_TIMEOUT_MS: 10_000,
    VALIDATE_TIMEOUT_MS: 10_000,
    MAX_RETRIES: 2, // total attempts = MAX_RETRIES + 1
    BASE_BACKOFF_MS: 300,
} as const;

const GATEWAY_PATHS = {
    INIT: "/gwprocess/v4/api.php",
    VALIDATE: "/validator/api/validationserverAPI.php",
} as const;

/* ------------------------------------------------------------------ *
 * Errors
 *
 * Distinct classes let callers (API routes) branch on `instanceof`
 * instead of parsing message strings, and each carries only
 * non-sensitive context that is safe to log or return to a client.
 * ------------------------------------------------------------------ */

export class ConfigurationError extends Error {
    constructor(message: string) {
        super(message);
        this.name = "ConfigurationError";
    }
}

export class NetworkError extends Error {
    constructor(message: string, options?: { cause?: unknown }) {
        super(message, options);
        this.name = "NetworkError";
    }
}

export class TimeoutError extends Error {
    constructor(message: string) {
        super(message);
        this.name = "TimeoutError";
    }
}

export class PaymentGatewayError extends Error {
    readonly httpStatus?: number;
    readonly gatewayReason?: string;

    constructor(message: string, options?: { httpStatus?: number; gatewayReason?: string; cause?: unknown }) {
        super(message, { cause: options?.cause });
        this.name = "PaymentGatewayError";
        this.httpStatus = options?.httpStatus;
        this.gatewayReason = options?.gatewayReason;
    }
}

export class PaymentValidationError extends Error {
    readonly reason: string;

    constructor(message: string, reason: string) {
        super(message);
        this.name = "PaymentValidationError";
        this.reason = reason;
    }
}

/* ------------------------------------------------------------------ *
 * Types
 * ------------------------------------------------------------------ */

export interface CustomerInfo {
    readonly name: string;
    readonly email: string;
    readonly phone: string;
    readonly address?: string;
    readonly city?: string;
}

export interface OrderInfo {
    readonly tranId: string;
    readonly amount: number;
    readonly currency: string;
    readonly productName: string;
}

/* ------------------------------------------------------------------ *
 * Response schemas
 *
 * SSLCommerz is an external, untrusted boundary. Every field we rely
 * on downstream is parsed with Zod so malformed/unexpected payloads
 * fail fast instead of silently propagating `undefined` into payment
 * logic. Unknown extra fields are tolerated (`.passthrough()`-free —
 * we deliberately only declare what we use).
 * ------------------------------------------------------------------ */

const initResponseSchema = z.object({
    status: z.enum([GATEWAY_STATUS.SUCCESS, GATEWAY_STATUS.FAILED]),
    failedreason: z.string().optional(),
    sessionkey: z.string().optional(),
    GatewayPageURL: z.string().url().optional(),
});

export type SSLCommerzInitResponse = z.infer<typeof initResponseSchema>;

const validationResponseSchema = z.object({
    status: z.string(), // gateway uses a wider, less stable set here than init
    val_id: z.string().optional(),
    tran_id: z.string().optional(),
    amount: z.string().optional(),
    store_amount: z.string().optional(),
    currency_amount: z.string().optional(),
    currency_type: z.string().optional(),
    bank_tran_id: z.string().optional(),
    card_type: z.string().optional(),
    card_issuer: z.string().optional(),
});

export type SSLCommerzValidationResponse = z.infer<typeof validationResponseSchema>;

function parseGatewayResponse<T>(schema: z.ZodType<T>, raw: unknown, context: { tranId?: string; valId?: string }): T {
    const result = schema.safeParse(raw);
    if (!result.success) {
        log.error(
            {
            ...context,
            issues: result.error.issues.map((i) => ({ path: i.path.join("."), message: i.message })),
        },
        "SSLCommerz response failed schema validation"
    );
        throw new PaymentGatewayError("SSLCommerz returned an unexpected response shape", {
            gatewayReason: "SCHEMA_VALIDATION_FAILED",
        });
    }
    return result.data;
}

/* ------------------------------------------------------------------ *
 * Configuration
 *
 * Read and validate env vars once per call site instead of scattering
 * `process.env.X` (and its associated "did we check for undefined?"
 * question) throughout the file. Values are never logged.
 * ------------------------------------------------------------------ */

interface GatewayConfig {
    readonly storeId: string;
    readonly storePassword: string;
    readonly apiBaseUrl: string;
    readonly appUrl: string;
}

function getGatewayConfig(): GatewayConfig {
    const storeId = process.env.SSLCOMMERZ_STORE_ID;
    const storePassword = process.env.SSLCOMMERZ_STORE_PASSWORD;
    const apiBaseUrl = process.env.SSLCOMMERZ_API_BASE_URL;
    const appUrl = process.env.APP_URL;

    const missing: string[] = [];
    if (!storeId) missing.push("SSLCOMMERZ_STORE_ID");
    if (!storePassword) missing.push("SSLCOMMERZ_STORE_PASSWORD");
    if (!apiBaseUrl) missing.push("SSLCOMMERZ_API_BASE_URL");
    if (!appUrl) missing.push("APP_URL");

    if (missing.length > 0) {
        // Names of missing vars are safe to log; their values never are.
        throw new ConfigurationError(`Missing required environment variables: ${missing.join(", ")}`);
    }

    // Fail fast on an SSRF-adjacent misconfiguration: a base URL that
    // isn't actually a URL would otherwise surface as a confusing
    // fetch-time TypeError deep inside createGatewayUrl().
    try {
        new URL(apiBaseUrl!);
    } catch {
        throw new ConfigurationError("SSLCOMMERZ_API_BASE_URL is not a valid URL");
    }

    return { storeId: storeId!, storePassword: storePassword!, apiBaseUrl: apiBaseUrl!, appUrl: appUrl! };
}

/**
 * Builds a gateway URL via the URL class rather than string
 * concatenation, so a trailing/missing slash in the env var can never
 * produce a malformed or unexpectedly-redirected request.
 */
function createGatewayUrl(base: string, path: string, params?: URLSearchParams): string {
    const url = new URL(path, base.endsWith("/") ? base : `${base}/`);
    if (params) url.search = params.toString();
    return url.toString();
}

/* ------------------------------------------------------------------ *
 * HTTP helper: timeout + retry with exponential backoff
 *
 * Retries are limited to network-level failures and 5xx responses —
 * conditions where re-sending the *same* request is safe. 4xx
 * responses (bad request, auth) and successful HTTP responses with a
 * gateway-level failure are NOT retried, since retrying those either
 * can't succeed or risks duplicate payment-initiation side effects.
 * ------------------------------------------------------------------ */

interface FetchWithRetryOptions {
    readonly timeoutMs: number;
    readonly maxRetries: number;
    readonly context: Record<string, unknown>;
}

function isRetryableStatus(status: number): boolean {
    return status >= 500 && status < 600;
}

async function fetchWithRetry(input: string, init: RequestInit, options: FetchWithRetryOptions): Promise<Response> {
    const { timeoutMs, maxRetries, context } = options;
    let lastError: unknown;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        const controller = new AbortController();
        const timeout = setTimeout(() => controller.abort(), timeoutMs);

        try {
            const response = await fetch(input, { ...init, signal: controller.signal });
            clearTimeout(timeout);

            if (isRetryableStatus(response.status) && attempt < maxRetries) {
                log.warn(
                    { 
                        ...context, 
                        attempt, 
                        httpStatus: 
                        response.status 
                    },
                    "SSLCommerz request returned retryable status"
                );
                await backoff(attempt);
                continue;
            }

            return response;
        } catch (err) {
            clearTimeout(timeout);
            lastError = err;

            const timedOut = err instanceof Error && err.name === "AbortError";
            if (timedOut) {
                log.warn(
                    { 
                        ...context, 
                        attempt, 
                        timeoutMs
                     },
                     "SSLCommerz request timed out"
                );
            } else {
                log.warn(
                    { 
                        ...context, 
                        attempt, 
                        error: (err as Error)?.message 
                    },
                    "SSLCommerz request network error"
                );
            }

            if (attempt < maxRetries) {
                await backoff(attempt);
                continue;
            }

            if (timedOut) {
                throw new TimeoutError(`SSLCommerz request timed out after ${maxRetries + 1} attempt(s)`);
            }
            throw new NetworkError(`SSLCommerz request failed after ${maxRetries + 1} attempt(s)`, { cause: err });
        }
    }

    // Unreachable, but keeps TypeScript's control-flow analysis happy.
    throw new NetworkError("SSLCommerz request failed", { cause: lastError });
}

function backoff(attempt: number): Promise<void> {
    const delay = HTTP.BASE_BACKOFF_MS * 2 ** attempt;
    return new Promise((resolve) => setTimeout(resolve, delay));
}

/* ------------------------------------------------------------------ *
 * Request builders
 * ------------------------------------------------------------------ */

function buildInitFormData(config: GatewayConfig, order: OrderInfo, customer: CustomerInfo): FormData {
    const formData = new FormData();

    formData.append("store_id", config.storeId);
    formData.append("store_passwd", config.storePassword);
    formData.append("total_amount", order.amount.toString());
    formData.append("currency", order.currency);
    formData.append("tran_id", order.tranId);

    formData.append("success_url", createGatewayUrl(config.appUrl, "/api/payment/success"));
    formData.append("fail_url", createGatewayUrl(config.appUrl, "/api/payment/fail"));
    formData.append("cancel_url", createGatewayUrl(config.appUrl, "/api/payment/cancel"));
    formData.append("ipn_url", createGatewayUrl(config.appUrl, "/api/payment/ipn"));

    formData.append("cus_name", customer.name);
    formData.append("cus_email", customer.email);
    formData.append("cus_add1", customer.address || "N/A");
    formData.append("cus_city", customer.city || "N/A");
    formData.append("cus_postcode", DEFAULTS.POSTCODE);
    formData.append("cus_country", DEFAULTS.COUNTRY);
    formData.append("cus_phone", customer.phone);

    formData.append("shipping_method", DEFAULTS.SHIPPING_METHOD);
    formData.append("num_of_item", DEFAULTS.NUM_OF_ITEM);
    formData.append("product_name", order.productName);
    formData.append("product_category", DEFAULTS.PRODUCT_CATEGORY);
    formData.append("product_profile", DEFAULTS.PRODUCT_PROFILE);

    return formData;
}

/* ------------------------------------------------------------------ *
 * Public API
 * ------------------------------------------------------------------ */

export function generateTranId(): string {
    const random = Math.random().toString(36).substring(2, 10).toUpperCase();
    return `TXN-${Date.now()}-${random}`;
}

export async function initiatePayment(order: OrderInfo, customer: CustomerInfo): Promise<SSLCommerzInitResponse> {
    const config = getGatewayConfig();
    const formData = buildInitFormData(config, order, customer);
    const url = createGatewayUrl(config.apiBaseUrl, GATEWAY_PATHS.INIT);

    const response = await fetchWithRetry(
        url,
        { method: "POST", body: formData },
        { timeoutMs: HTTP.INIT_TIMEOUT_MS, maxRetries: HTTP.MAX_RETRIES, context: { tranId: order.tranId, op: "initiatePayment" } },
    );

    if (!response.ok) {
        log.error(
            { 
                tranId: order.tranId, 
                httpStatus: response.status 
            },
            "SSLCommerz init HTTP error"
        );
        throw new PaymentGatewayError(`SSLCommerz init failed with HTTP ${response.status}`, { httpStatus: response.status });
    }

    const raw = await response.json();
    const data = parseGatewayResponse(initResponseSchema, raw, { tranId: order.tranId });

    if (data.status !== GATEWAY_STATUS.SUCCESS || !data.GatewayPageURL) {
        // Not thrown here: callers that only need the raw gateway
        // response (e.g. to show the user a specific failure reason)
        // still get it. Business-layer code decides whether a
        // non-success init should become a hard error.
        log.warn(
            { 
                tranId: order.tranId, 
                gatewayStatus: data.status, 
                reason: data.failedreason 
            },
            "SSLCommerz init non-success"
        );
    }

    return data;
}

export async function validateTransaction(valId: string): Promise<SSLCommerzValidationResponse> {
    if (!valId) {
        throw new PaymentValidationError("val_id is required", "MISSING_VAL_ID");
    }

    const config = getGatewayConfig();
    const params = new URLSearchParams({
        val_id: valId,
        store_id: config.storeId,
        store_passwd: config.storePassword,
        format: "json",
    });
    const url = createGatewayUrl(config.apiBaseUrl, GATEWAY_PATHS.VALIDATE, params);

    const response = await fetchWithRetry(
        url,
        { method: "GET" },
        { timeoutMs: HTTP.VALIDATE_TIMEOUT_MS, maxRetries: HTTP.MAX_RETRIES, context: { valId, op: "validateTransaction" } },
    );

    if (!response.ok) {
        log.error(
            { 
                valId, 
                httpStatus: response.status 
            },
            "SSLCommerz validation HTTP error"
        );
        throw new PaymentGatewayError(`SSLCommerz validation failed with HTTP ${response.status}`, { httpStatus: response.status });
    }

    const raw = await response.json();
    return parseGatewayResponse(validationResponseSchema, raw, { valId });
}

export function isValidationStatusSuccess(validation: SSLCommerzValidationResponse): boolean {
    return validation.status === VALIDATION_STATUS.VALID || validation.status === VALIDATION_STATUS.VALIDATED;
}

export function isValidationAmountMatching(validation: SSLCommerzValidationResponse, expectedAmount: number): boolean {
    const raw = validation.currency_amount ?? validation.amount;
    if (!raw) return false;

    const receivedAmount = Number(raw);
    if (!Number.isFinite(receivedAmount)) return false;

    return Math.abs(receivedAmount - expectedAmount) < DEFAULTS.AMOUNT_EPSILON;
}

/**
 * Full defensive check for a validated transaction: status, amount,
 * transaction id, and (when the caller has it) currency all have to
 * agree with what was recorded at order-creation time. Use this
 * instead of `isValidationStatusSuccess` alone before marking an
 * order as paid — status success on its own does not prove the
 * amount or tran_id line up, which is what makes payment tampering
 * (a forged success callback for a different/cheaper order) possible.
 */
export function assertPaymentIsLegitimate(
    validation: SSLCommerzValidationResponse,
    expected: { tranId: string; amount: number; currency?: string },
): void {
    if (!isValidationStatusSuccess(validation)) {
        throw new PaymentValidationError("Transaction status is not valid", "STATUS_NOT_VALID");
    }

    if (validation.tran_id && validation.tran_id !== expected.tranId) {
        throw new PaymentValidationError("Transaction id mismatch", "TRAN_ID_MISMATCH");
    }

    if (!isValidationAmountMatching(validation, expected.amount)) {
        throw new PaymentValidationError("Transaction amount mismatch", "AMOUNT_MISMATCH");
    }

    if (expected.currency && validation.currency_type && validation.currency_type !== expected.currency) {
        throw new PaymentValidationError("Transaction currency mismatch", "CURRENCY_MISMATCH");
    }

    if (!validation.val_id) {
        throw new PaymentValidationError("Missing val_id on validated transaction", "MISSING_VAL_ID");
    }
}
```

---

#### lib/services/order.service.ts
```bash
import  prisma  from "@/lib/prisma";
import { log } from "@/lib/logger";
import { validateTransaction, isValidationStatusSuccess, isValidationAmountMatching } from "@/lib/services/sslcommerz.service";

/**
 * success/route.ts ar ipn/route.ts duitai ei ekই verification logic follow kore,
 * tai ekhane ekbar likhe dutai reuse kora hocche.
 */
export type FinalizeOutcome =
    | { outcome: "not_found" }
    | { outcome: "already_processed"; orderId: string }
    | { outcome: "missing_val_id"; orderId: string }
    | { outcome: "validation_failed"; orderId: string }
    | { outcome: "amount_mismatch"; orderId: string }
    | { outcome: "success"; orderId: string };

export async function finalizeTransactionPayment(tranId: string, valId: string | undefined): Promise<FinalizeOutcome> {
    const transaction = await prisma.transaction.findUnique({ where: { tranId } });

    if (!transaction) {
        log.error({ tranId }, "Callback for unknown tranId");
        return { outcome: "not_found" };
    }

    // Idempotency — ei transaction already process hoye gele (success + IPN duitai
    // kache-kache ashte pare) revalidate na kore shorashori result ferot dei
    if (transaction.processedAt) {
        log.info({ tranId }, "Transaction already processed, skipping re-verification");
        return { outcome: "already_processed", orderId: transaction.orderId };
    }

    if (!valId) {
        log.warn({ tranId }, "Callback missing val_id");
        return { outcome: "missing_val_id", orderId: transaction.orderId };
    }

    const validation = await validateTransaction(valId);

    if (!isValidationStatusSuccess(validation)) {
        log.warn({ tranId, validation }, "Validation API returned non-success status");
        await prisma.transaction.update({
            where: { tranId },
            data: { status: "FAILED", rawValidationResponse: validation as object, processedAt: new Date() },
        });
        // Note: Order status ekhane change kora hoy na — user chaile abar
        // /api/payment/init call kore notun Transaction diye retry korte parbe
        return { outcome: "validation_failed", orderId: transaction.orderId };
    }

    if (!isValidationAmountMatching(validation, Number(transaction.amount))) {
        log.error({
            tranId, expected: transaction.amount, received: validation.currency_amount ?? validation.amount,
        }, "Validation amount mismatch — possible tampering");
        await prisma.transaction.update({
            where: { tranId },
            data: { status: "FAILED", rawValidationResponse: validation as object, processedAt: new Date() },
        });
        return { outcome: "amount_mismatch", orderId: transaction.orderId };
    }

    // Sob check pass — Transaction VALID, Order PAID (shudhu PENDING thakle),
    // ar Product stock decrement — sob ekshathe atomic
    const order = await prisma.order.findUnique({ where: { id: transaction.orderId } });
    if (!order) {
        log.error({ tranId, orderId: transaction.orderId }, "Transaction points to a missing order");
        return { outcome: "not_found" };
    }

    const [, orderUpdateResult, stockUpdateResult] = await prisma.$transaction([
        prisma.transaction.update({
            where: { tranId },
            data: {
                status: "VALID",
                valId: validation.val_id,
                bankTranId: validation.bank_tran_id,
                cardType: validation.card_type,
                cardIssuer: validation.card_issuer,
                storeAmount: validation.store_amount ? Number(validation.store_amount) : undefined,
                rawValidationResponse: validation as object,
                processedAt: new Date(),
            },
        }),
        // Guard: shudhu ekhono PENDING thakle-i PAID e move hobe — double-processing thekay eta second layer of protection
        prisma.order.updateMany({
            where: { id: order.id, status: "PENDING" },
            data: { status: "PAID" },
        }),
        // Guard: stock thakle-i decrement hobe, noyto oversell hobe na
        prisma.product.updateMany({
            where: { id: order.productId, stock: { gte: order.quantity } },
            data: { stock: { decrement: order.quantity } },
        }),
    ]);

    if (orderUpdateResult.count === 0) {
        log.warn({
            orderId: order.id, tranId,
        }, "Order was not in PENDING state during finalize — likely already paid via another transaction");
    }
    if (stockUpdateResult.count === 0) {
        log.error({
            orderId: order.id, productId: order.productId,
        }, "Stock decrement failed after payment — possible oversell, needs manual review");
    }

    log.info({ tranId, orderId: order.id }, "Payment verified and order finalized");
    return { outcome: "success", orderId: order.id };
}
```

---

#### app/api/orders  (checkout — Order create)
```bash
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import  prisma  from "@/lib/prisma";
import { log } from "@/lib/logger";
import { createOrderSchema } from "@/lib/validations/payment.schema";
import { handlePrismaError } from "@/lib/errors/handlePrismaError";
import { auth } from "@/lib/auth"; 

export async function POST(request: NextRequest) {
    const requestId = crypto.randomUUID();
    try {
        // 1. Auth check
        const session = await auth();
        if (!session?.user?.id) {
            log.warn(
                {
                    requestId,
                    ip: request.headers.get("x-forwarded-for") ?? "unknown",
                    userAgent: request.headers.get("user-agent"),
                },
                "Unauthorized order creation attempt"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Unauthorized" }, 
                { status: 401 }
            );
        }

        const userId = session.user.id;

        // 2. Body validation
        const body = await request.json().catch(() => null);
        const parsed = createOrderSchema.safeParse(body);

        if (!parsed.success) {
            log.warn(
                {
                    requestId,
                    userId,
                    validationErrors: z.treeifyError(parsed.error)
                },
                "Order validation failed"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Invalid request body", errors: z.treeifyError(parsed.error) },
                { status: 400 }
            );
        }
        const { productId, quantity, customerName, customerEmail, customerPhone, customerAddress, customerCity } = parsed.data;

        // 3. Product DB theke fetch — price client theke kokhono trust kora hoy na
        const product = await prisma.product.findUnique({
             where: { id: productId } 
            });

        if (!product) {
            log.warn(
                {
                    requestId,
                    userId,
                    productId,
                },
                "Product not found"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Product not found" },
                { status: 404 }
            );
        }
        if (product.stock < quantity) {
            log.warn(
                {
                    requestId,
                    userId,
                    productId,
                    requestedQuantity: quantity,
                    availableStock: product.stock,
                },
                "Insufficient stock"
            )
            return NextResponse.json(
                { status: "error", message: "Insufficient stock" },
                { status: 409 }
            );
        }

        // 4. Server-side price calculation
        const unitPrice = Number(product.price);
        const totalAmount = Math.round(unitPrice * quantity * 100) / 100;

        // 5. Order create — status PENDING, payment ekhono shuru hoyni
        const order = await prisma.order.create({
            data: {
                userId, productId, quantity, totalAmount,
                customerName, customerEmail, customerPhone, customerAddress, customerCity,
                status: "PENDING",
            },
        });

        log.info(
            {
                requestId,
                orderId: order.id,
                productId, 
                userId,
                quantity,
                totalAmount,
            }, 
            "Order created"
        );

        return NextResponse.json(
            { status: "success", requestId, message: "Order created successfully", data: { orderId: order.id, totalAmount }},
            { status: 201 }
        );
    } catch (error) {
        const { status, message } = handlePrismaError(error);
        log.error(
        { 
            requestId,
            err: error instanceof Error 
                ? { message: error.message, stack: error.stack, name: error.name }
                : String(error),
        },
        message
    )
    return NextResponse.json(
        { status: "fail", requestId, message },
        { status }
    )
    }
}
```

---

#### Package Install
```bash
npm install next-auth@beta
```
---

#### /lib/auth.ts
```bash
import NextAuth from "next-auth";
import { authConfig } from "@/lib/auth.config"; 

export const { handlers, auth, signIn, signOut } = NextAuth(authConfig);
```
---

#### lib/auth/password.ts
```bash
import bcrypt from "bcryptjs";

const SALT_ROUNDS = 12; // production standard — 10-12 recommended

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(
  hashValue: string,
  password: string
): Promise<boolean> {
  return bcrypt.compare(password, hashValue);
}
```
---

#### lib/auth.config.ts
```bash
import type { NextAuthConfig } from "next-auth";
import Credentials from "next-auth/providers/credentials";
import { verifyPassword } from "@/lib/auth/password";
import prisma from "@/lib/prisma";
import { log } from "@/lib/logger";
import { loginSchema } from "@/lib/validations/auth.schema";

export const authConfig: NextAuthConfig = {
  providers: [
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },

      authorize: async (credentials) => {
        const requestId = crypto.randomUUID();

        const parsed = loginSchema.safeParse(credentials);

        if (!parsed.success) {
          log.warn(
            { 
                requestId,
                errors: parsed.error.flatten(), 
            },
             "Login validation failed"
            );
          return null;
        }
        const { email, password } = parsed.data;

        const user = await prisma.user.findUnique({
             where: { email }, 
            });

        if (!user) {
          log.warn(
            { 
                requestId,
                email, 
            }, 
            "Login attempt for non-existent email"
        );
          return null;
        }

        if (!user.passwordHash) {
          log.warn(
            {
              requestId,
              userId: user.id,
            },
            "No password hash for user"
          );
          return null;
        }

        const isValidPassword = await verifyPassword(user.passwordHash, password);

        if (!isValidPassword) {
          log.warn(
            { 
                requestId,
                userId: user.id 
            }, 
            "Invalid password attempt"
        );
          return null;
        }

        log.info(
            { 
                requestId,
                userId: user.id 
            }, 
            "User authenticated successfully"
        );

        return {
          id: user.id,
          email: user.email,
          name: user.name,
        };
      },
    }),
  ],

  session: {
    strategy: "jwt",
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },

  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.sub = user.id;
      }
      return token;
    },
    session({ session, token }) {
      if (token?.sub && session.user) {
        session.user.id = token.sub;
      }
      return session;
    },
  },

  pages: {
    signIn: "/login",
    error: "/login",
  },

  useSecureCookies: process.env.NODE_ENV === "production",
  trustHost: true,
};
```
---

#### app/api/auth/[...nextauth]/route.ts
```bash
import { handlers } from "@/lib/auth";
export const { GET, POST } = handlers;
```
---

#### app/api/payment/init
```bash
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import  prisma  from "@/lib/prisma";
import { log } from "@/lib/logger";
import { initPaymentSchema } from "@/lib/validations/payment.schema";
import { generateTranId, initiatePayment } from "@/lib/services/sslcommerz.service";
import { handlePrismaError } from "@/lib/errors/handlePrismaError";
import { auth } from "@/lib/auth";

const SESSION_VALIDITY_MINUTES = 55; // SSLCommerz session ~1hr e expire hoy, tai buffer rekhe 55 min

export async function POST(request: NextRequest) {
    const requestId = crypto.randomUUID();
    try {
        // 1. Auth check
        const session = await auth();
        const userId = session?.user?.id;
        if (!userId) {
            log.warn(
                {
                    requestId,
                    ip: request.headers.get("x-forwarded-for") ?? "unknown",
                    userAgent: request.headers.get("user-agent"),
                },
                "Unauthorized payment initialization attempt"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Unauthorized" }, 
                { status: 401 });
        }


        // 2. Body validation
        const body = await request.json().catch(() => null);
        const parsed = initPaymentSchema.safeParse(body);

        if (!parsed.success) {
            log.warn(
                {
                    requestId,
                    userId,
                    validationErrors: z.treeifyError(parsed.error),
                },
                "Invalid payment initialization request"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Invalid request body", errors: z.treeifyError(parsed.error) },
                { status: 400 }
            );
        }

        const { orderId } = parsed.data;

        // 3. Order + Product DB theke fetch (relation include kore)
        const order = await prisma.order.findUnique({
            where: { id: orderId },
            include: { product: true },
        });
        
        if (!order) {
            log.warn(
                {
                    requestId,
                    userId,
                    orderId,
                },
                "Order not found"
            )
            return NextResponse.json(
                { status: "error", requestId, message: "Order not found" }, 
                { status: 404 }
            );
        }

        if (order.userId !== userId) {
            log.warn(
                { 
                    requestId,
                    orderId, 
                    userId,
                    orderOwnerId: order.userId, 
                },
                "Order ownership mismatch"
            );
            return NextResponse.json(
                { status: "error", requestId, message: "Forbidden" }, 
                { status: 403 }
            );
        }
        if (order.status !== "PENDING") {
            log.warn(
                {
                    requestId,
                    orderId,
                    userId,
                    orderStatus: order.status,
                },
                "Payment initialization attempted for non-pending order"
            )
            return NextResponse.json(
                { status: "error", requestId, message: `Order is already ${order.status.toLowerCase()}` }, 
                { status: 409 }
            );
        }

        // 4. Duplicate protection — recent (not expired) INITIATED session thakle reuse
        const sessionCutoff = new Date(Date.now() - SESSION_VALIDITY_MINUTES * 60 * 1000);

        const existingTransaction = await prisma.transaction.findFirst({
            where: { orderId: order.id, status: "INITIATED", createdAt: { gte: sessionCutoff } },
            orderBy: { createdAt: "desc" },
        });

        if (existingTransaction?.gatewayPageURL) {
            log.info(
                { 
                    requestId,
                    orderId, 
                    tranId: existingTransaction.tranId,
                    transactionId: existingTransaction.id,
                },
                "Reusing existing initiated transaction"
            );
            return NextResponse.json(
                { status: "success", requestId, data: { GatewayPageURL: existingTransaction.gatewayPageURL, tranId: existingTransaction.tranId },}
            );
        }

        // 5. Unique tran_id
        let tranId = generateTranId();
        let attempt = 0;
        while (await prisma.transaction.findUnique({ where: { tranId } })) {
            if (++attempt > 5) {
                return NextResponse.json(
                    { status: "error", requestId, message: "Could not generate unique tran_id" }, 
                    { status: 500 }
                );
            }
            tranId = generateTranId();
        }

        // 6. SSLCommerz init call — Order-er customer info + product.name use kora hocche
        const sslResponse = await initiatePayment(
            {
                tranId,
                amount: Number(order.totalAmount),
                currency: order.currency,
                productName: order.product.name,
            },
            {
                name: order.customerName,
                email: order.customerEmail,
                phone: order.customerPhone,
                address: order.customerAddress ?? undefined,
                city: order.customerCity ?? undefined,
            }
        );

        if (sslResponse.status !== "SUCCESS" || !sslResponse.GatewayPageURL) {
            log.error(
                { 
                    requestId,
                    orderId, 
                    tranId, 
                    status: sslResponse.status,
                    failedReason: sslResponse.failedreason, 
                },
                "SSLCommerz payment initialization failed"
            );
            await prisma.transaction.create({
                data: {
                    tranId, orderId: order.id, amount: order.totalAmount, currency: order.currency,
                    status: "FAILED", rawInitResponse: sslResponse as object,
                },
            });
            return NextResponse.json(
                { status: "error", requestId, message: sslResponse.failedreason || "Init failed" }, 
                { status: 400 }
            );
        }

        // 7. Transaction save
        await prisma.transaction.create({
            data: {
                tranId, orderId: order.id, amount: order.totalAmount, currency: order.currency,
                status: "INITIATED", sessionKey: sslResponse.sessionkey,
                gatewayPageURL: sslResponse.GatewayPageURL, rawInitResponse: sslResponse as object,
            },
        });

        log.info(
            { 
                requestId,
                orderId,
                userId, 
                tranId,
                amount: order.totalAmount,
                currency: order.currency,
            },
            "Payment initialized successfully"
        );
        return NextResponse.json({
            status: "success", requestId,
            data: { GatewayPageURL: sslResponse.GatewayPageURL, tranId },
        });
    } catch (error) {
        const { status, message } = handlePrismaError(error);
                log.error(
                { 
                    requestId,
                    err: error instanceof Error 
                        ? { message: error.message, stack: error.stack, name: error.name }
                        : String(error),
                },
                message
            )
            return NextResponse.json(
                { status: "fail", requestId, message },
                { status }
            )
    }
}
```

---

#### app/api/payment/success
```bash
import { NextRequest, NextResponse } from "next/server";
import { log } from "@/lib/logger";
import { sslCallbackSchema } from "@/lib/validations/payment.schema";
import { finalizeTransactionPayment } from "@/lib/services/order.service";

export async function POST(request: NextRequest) {
    const appUrl = process.env.APP_URL ?? "";
    try {
        const formData = await request.formData();
        const rawPayload = Object.fromEntries(formData.entries());
        const parsed = sslCallbackSchema.safeParse(rawPayload);

        if (!parsed.success) {
            log.warn("Invalid success callback payload", { rawPayload });
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=invalid_callback`);
        }

        const { tran_id, val_id } = parsed.data;
        const result = await finalizeTransactionPayment(tran_id, val_id);

        switch (result.outcome) {
            case "not_found":
                return NextResponse.redirect(`${appUrl}/payment/failed?reason=unknown_transaction`);
            case "already_processed":
            case "success":
                return NextResponse.redirect(`${appUrl}/payment/success?orderId=${result.orderId}`);
            case "missing_val_id":
                return NextResponse.redirect(`${appUrl}/payment/failed?reason=missing_val_id&orderId=${result.orderId}`);
            case "validation_failed":
                return NextResponse.redirect(`${appUrl}/payment/failed?reason=validation_failed&orderId=${result.orderId}`);
            case "amount_mismatch":
                return NextResponse.redirect(`${appUrl}/payment/failed?reason=amount_mismatch&orderId=${result.orderId}`);
        }
    } catch (error) {
        log.error("Success callback unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.redirect(`${appUrl}/payment/failed?reason=server_error`);
    }
}
```

---

#### app/api/payment/fail
```bash
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { log } from "@/lib/logger";
import { sslCallbackSchema } from "@/lib/validations/payment.schema";

export async function POST(request: NextRequest) {
    const appUrl = process.env.APP_URL ?? "";
    try {
        const formData = await request.formData();
        const rawPayload = Object.fromEntries(formData.entries());
        const parsed = sslCallbackSchema.safeParse(rawPayload);
        if (!parsed.success) {
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=invalid_callback`);
        }
        const { tran_id } = parsed.data;

        const transaction = await prisma.transaction.findUnique({ where: { tranId: tran_id } });
        if (transaction && !transaction.processedAt) {
            await prisma.$transaction([
                prisma.transaction.update({
                    where: { tranId: tran_id },
                    data: { status: "FAILED", processedAt: new Date(), rawIpnPayload: rawPayload },
                }),
                prisma.order.updateMany({
                    where: { id: transaction.orderId, status: "PENDING" },
                    data: { status: "FAILED" },
                }),
            ]);
        }

        log.info("Payment failed callback received", { tran_id });
        return NextResponse.redirect(`${appUrl}/payment/failed?tran_id=${tran_id}`);
    } catch (error) {
        log.error("Fail callback unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.redirect(`${appUrl}/payment/failed?reason=server_error`);
    }
}
```

---

#### app/api/payment/cancel
```bash
import { NextRequest, NextResponse } from "next/server";
import  prisma  from "@/lib/prisma";
import { log } from "@/lib/logger";
import { sslCallbackSchema } from "@/lib/validations/payment.schema";

export async function POST(request: NextRequest) {
    const appUrl = process.env.APP_URL ?? "";
    try {
        const formData = await request.formData();
        const rawPayload = Object.fromEntries(formData.entries());
        // Normalize FormData values to plain strings for JSON storage
        const rawPayloadJson: Record<string, string> = Object.fromEntries(
            Object.entries(rawPayload).map(([k, v]) => [k, typeof v === "string" ? v : (v as File).name || ""])
        );
        const parsed = sslCallbackSchema.safeParse(rawPayload);
        if (!parsed.success) {
            return NextResponse.redirect(`${appUrl}/payment/cancelled?reason=invalid_callback`);
        }
        const { tran_id } = parsed.data;

        const transaction = await prisma.transaction.findUnique({ where: { tranId: tran_id } });
        if (transaction && !transaction.processedAt) {
            await prisma.$transaction([
                prisma.transaction.update({
                    where: { tranId: tran_id },
                    data: { status: "CANCELLED", processedAt: new Date(), rawIpnPayload: rawPayloadJson },
                }),
                prisma.order.updateMany({
                    where: { id: transaction.orderId, status: "PENDING" },
                    data: { status: "CANCELLED" },
                }),
            ]);
        }

        log.info({ tran_id }, "Payment cancelled by user");
        return NextResponse.redirect(`${appUrl}/payment/cancelled?tran_id=${tran_id}`);
    } catch (error) {
        log.error({ error: error instanceof Error ? error.message : String(error) }, "Cancel callback unexpected error");
        return NextResponse.redirect(`${appUrl}/payment/cancelled?reason=server_error`);
    }
}
```

---

#### app/api/payment/ipn
```bash
import { NextRequest, NextResponse } from "next/server";
import { log } from "@/lib/logger";
import { ipnPayloadSchema } from "@/lib/validations/payment.schema";
import { finalizeTransactionPayment } from "@/lib/services/order.service";

// SSLCommerz-er server theke background e ashe, browser-er upor depend kore na —
// sob theke reliable source. Business logic fail holeo sob shomoy HTTP 200 return
// kori, noyto SSLCommerz infinite retry korte thakbe.
export async function POST(request: NextRequest) {
    try {
        const formData = await request.formData();
        const rawPayload = Object.fromEntries(formData.entries());
        const parsed = ipnPayloadSchema.safeParse(rawPayload);

        if (!parsed.success) {
            log.warn("Invalid IPN payload", { rawPayload });
            return NextResponse.json({ status: "error", message: "Invalid IPN payload" }, { status: 200 });
        }

        const { tran_id, val_id } = parsed.data;
        const result = await finalizeTransactionPayment(tran_id, val_id);

        if (result.outcome === "not_found") {
            return NextResponse.json({ status: "error", message: "Unknown transaction" }, { status: 200 });
        }
        if (result.outcome === "success" || result.outcome === "already_processed") {
            return NextResponse.json({ status: "success", message: "IPN processed" }, { status: 200 });
        }
        return NextResponse.json({ status: "error", message: result.outcome }, { status: 200 });
    } catch (error) {
        log.error("IPN unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.json({ status: "error", message: "Internal error" }, { status: 200 });
    }
}
```

#### app/api/auth/signup/route.ts
```bash
import { NextRequest, NextResponse } from "next/server";
import prisma from "@/lib/prisma";
import { log } from "@/lib/logger";
import { handlePrismaError } from "@/lib/errors/handlePrismaError";
import { SignupSchema } from "@/lib/validations/auth.schema";
import { hashPassword } from "@/lib/auth/password";


export async function POST(request: NextRequest) {
    const requestId = crypto.randomUUID();
    try {
        log.info(
            { 
                requestId, 
                path: request.nextUrl.pathname, 
                method: request.method 
            },
            "User data received"
        )

        const jsonBody = await request.json();
        const validation = SignupSchema.safeParse(jsonBody);

        if (!validation.success) {
            log.warn(
                { 
                    requestId, 
                    errors: validation.error.flatten().fieldErrors 
                },
                "Invalid User Input"
            )
            return NextResponse.json(
                { status: "fail", requestId, message: "Invalid user input", errors: validation.error.flatten().fieldErrors },
                { status: 400 }
            )
        }

        const { password, ...userData } = validation.data;
        const passwordHash = await hashPassword(password);


        const result = await prisma.user.create({
            data: {
                ...userData,
                passwordHash,
            },
        })


        log.info(
            { 
                requestId, 
                name: result.name, 
                email: result.email 
            },
            "User created"
        )

        return NextResponse.json(
            {
                status: "success",
                requestId,
                message: "Account created.",
                data: { id: result.id, name: result.name, email: result.email },
            },
            { status: 201 }
        )
    } catch (error) {
        const { status, message } = handlePrismaError(error);
        log.error(
            {
                requestId,
                err: error instanceof Error
                    ? { message: error.message, stack: error.stack, name: error.name }
                    : String(error),
            },
            message
        )
        return NextResponse.json(
            { status: "fail", requestId, message },
            { status }
        )
    }
}
```
---

---

#### Checklist — kon requirement kothay ache

| Requirement | Kothay |
|---|---|
| Zod validation | `payment.schema.ts` — `createOrderSchema`, `initPaymentSchema`, `sslCallbackSchema`, `ipnPayloadSchema` |
| Prisma transaction save | `Transaction` model, sob route-e `create/update` |
| Unique `tran_id` | `generateTranId()` + DB collision-check loop |
| Logging | `logger.ts` — structured JSON |
| Service layer | `sslcommerz.service.ts` (SSLCommerz API) + `order.service.ts` (business logic) |
| Secrets `.env`-e | Kono hardcoded value nai |
| `APP_URL` `.env` theke | Sob callback URL dynamic |
| Customer/product DB theke | `orders/route.ts`-e Order create, `init/route.ts`-e Order+Product fetch |
| Order create + price integrity | `orders/route.ts` — `Product.price × quantity` server-side calculate |
| Stock management | `order.service.ts` — payment verified hole atomic-e `Product.stock` decrement |
| Success callback-e `tran_id` verify | `order.service.ts` → `finalizeTransactionPayment()` |
| IPN verify | Same function, IPN route-e reuse |
| Duplicate payment protection | `tranId` unique + non-expired INITIATED transaction reuse |
| Idempotency | `Transaction.processedAt` + `Order.updateMany` PENDING-guard (double layer) |

**Setup**: `npx prisma migrate dev --name init_production_schema` → `.env` fill → `orders`/`init` route-e auth wire up koren → `npm install zod` (na thakle).
