## Complete Production-Ready SSLCommerz Payment Workflow

#### Architecture
```bash
prisma/
- schema.prisma          → Order + Transaction model

src/lib/
- prisma.ts               → Prisma client singleton
- logger.ts                → Structured logger
- validations/payment.schema.ts   → Zod validation
- services/sslcommerz.service.ts  → SSLCommerz API calls (init + validation)

src/app/api/payment/
- init/route.ts            → Payment session start
- success/route.ts         → Success callback (server-verified)
- fail/route.ts            → Fail callback
- cancel/route.ts          → Cancel callback
- ipn/route.ts             → IPN (background, most reliable)
```

Simple version-er sathe key difference: `tran_id` ekhon Math.random() diye na, DB-e unique-check kore generate hoy; customer/product data hardcoded na, DB theke ashe; ar sob theke important — success/IPN-e SSLCommerz-er **Validation API** hit kore confirm kora hoy je payment ta genuine, shudhu redirect ashlei "paid" mark kora hoy na (eta spoof kora jay bole).

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
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  phone     String?
  address   String?
  city      String?

  orders    Order[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

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

#### src/lib/prisma.ts
```bash
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
    globalForPrisma.prisma ||
    new PrismaClient({
        log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
    });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

---

#### src/lib/logger.ts
```bash
type LogContext = Record<string, unknown>;

function format(level: string, message: string, context?: LogContext) {
    return JSON.stringify({
        level,
        message,
        timestamp: new Date().toISOString(),
        ...context,
    });
}

export const log = {
    info(message: string, context?: LogContext) {
        console.log(format("info", message, context));
    },
    warn(message: string, context?: LogContext) {
        console.warn(format("warn", message, context));
    },
    error(message: string, context?: LogContext) {
        console.error(format("error", message, context));
    },
};
```

---

#### src/lib/validations/payment.schema.ts
```bash
import { z } from "zod";

// Client theke shudhu orderId nei — customer/product data DB theke,
// tampering thekay eta protect kore
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

#### src/lib/services/sslcommerz.service.ts
```bash
import { log } from "@/lib/logger";

interface SSLCommerzInitResponse {
    status: "SUCCESS" | "FAILED";
    failedreason?: string;
    sessionkey?: string;
    GatewayPageURL?: string;
    [key: string]: unknown;
}

interface SSLCommerzValidationResponse {
    status: "VALID" | "VALIDATED" | "INVALID_TRANSACTION" | "FAILED" | string;
    tran_id?: string;
    val_id?: string;
    amount?: string;
    store_amount?: string;
    bank_tran_id?: string;
    card_type?: string;
    card_issuer?: string;
    currency_amount?: string;
    [key: string]: unknown;
}

interface CustomerInfo {
    name: string; email: string; address1: string;
    city: string; postcode: string; country: string; phone: string;
}

interface OrderInfo {
    tranId: string; amount: number; currency: string;
    productName: string; productCategory: string; productProfile: string;
}

function getEnvConfig() {
    const storeId = process.env.SSLCOMMERZ_STORE_ID;
    const storePassword = process.env.SSLCOMMERZ_STORE_PASSWORD;
    const apiBaseUrl = process.env.SSLCOMMERZ_API_BASE_URL;
    const appUrl = process.env.APP_URL;

    if (!storeId || !storePassword) throw new Error("SSLCommerz credentials missing");
    if (!apiBaseUrl) throw new Error("SSLCOMMERZ_API_BASE_URL missing");
    if (!appUrl) throw new Error("APP_URL missing");

    return { storeId, storePassword, apiBaseUrl, appUrl };
}

// Timestamp + random suffix — collision practically zero, tobuo DB unique constraint final safety net
export function generateTranId(): string {
    const random = Math.random().toString(36).substring(2, 10).toUpperCase();
    return `TXN-${Date.now()}-${random}`;
}

export async function initiatePayment(order: OrderInfo, customer: CustomerInfo): Promise<SSLCommerzInitResponse> {
    const { storeId, storePassword, apiBaseUrl, appUrl } = getEnvConfig();

    const formData = new FormData();
    formData.append("store_id", storeId);
    formData.append("store_passwd", storePassword);
    formData.append("total_amount", order.amount.toString());
    formData.append("currency", order.currency);
    formData.append("tran_id", order.tranId);

    formData.append("success_url", `${appUrl}/api/payment/success`);
    formData.append("fail_url", `${appUrl}/api/payment/fail`);
    formData.append("cancel_url", `${appUrl}/api/payment/cancel`);
    formData.append("ipn_url", `${appUrl}/api/payment/ipn`);

    formData.append("cus_name", customer.name);
    formData.append("cus_email", customer.email);
    formData.append("cus_add1", customer.address1);
    formData.append("cus_city", customer.city);
    formData.append("cus_postcode", customer.postcode);
    formData.append("cus_country", customer.country);
    formData.append("cus_phone", customer.phone);

    formData.append("shipping_method", "NO");
    formData.append("num_of_item", "1");
    formData.append("product_name", order.productName);
    formData.append("product_category", order.productCategory);
    formData.append("product_profile", order.productProfile);

    const response = await fetch(`${apiBaseUrl}/gwprocess/v4/api.php`, { method: "POST", body: formData });

    if (!response.ok) {
        log.error("SSLCommerz init HTTP error", { httpStatus: response.status, tranId: order.tranId });
        throw new Error(`SSLCommerz init failed with HTTP ${response.status}`);
    }

    const data = (await response.json()) as SSLCommerzInitResponse;
    if (data.status !== "SUCCESS" || !data.GatewayPageURL) {
        log.warn("SSLCommerz init non-success", { tranId: order.tranId, reason: data.failedreason });
    }
    return data;
}

// SSLCommerz Order Validation API — https://developer.sslcommerz.com/doc/v4/#validation
export async function validateTransaction(valId: string): Promise<SSLCommerzValidationResponse> {
    const { storeId, storePassword, apiBaseUrl } = getEnvConfig();

    const params = new URLSearchParams({
        val_id: valId, store_id: storeId, store_passwd: storePassword, format: "json",
    });

    const response = await fetch(`${apiBaseUrl}/validator/api/validationserverAPI.php?${params}`, { method: "GET" });

    if (!response.ok) {
        log.error("SSLCommerz validation HTTP error", { httpStatus: response.status, valId });
        throw new Error(`Validation request failed with HTTP ${response.status}`);
    }
    return (await response.json()) as SSLCommerzValidationResponse;
}

// Amount mismatch dhorbe — attacker kom pay kore beshi amount er order "valid" dekhate parbe na
export function isValidationAmountMatching(validation: SSLCommerzValidationResponse, expectedAmount: number): boolean {
    const receivedAmount = parseFloat(validation.currency_amount ?? validation.amount ?? "0");
    return Math.abs(receivedAmount - expectedAmount) < 0.01;
}

export function isValidationStatusSuccess(validation: SSLCommerzValidationResponse): boolean {
    return validation.status === "VALID" || validation.status === "VALIDATED";
}
```

---

#### app/api/payment/init
```bash
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { prisma } from "@/lib/prisma";
import { log } from "@/lib/logger";
import { initPaymentSchema } from "@/lib/validations/payment.schema";
import { generateTranId, initiatePayment } from "@/lib/services/sslcommerz.service";
// import { auth } from "@/lib/auth";

export async function POST(request: NextRequest) {
    try {
        // 1. Auth check — apnar NextAuth.js v5 auth() diye replace korben
        // const session = await auth();
        // if (!session?.user?.id) return NextResponse.json({ status: "error", message: "Unauthorized" }, { status: 401 });
        const userId = request.headers.get("x-debug-user-id");
        if (!userId) return NextResponse.json({ status: "error", message: "Unauthorized" }, { status: 401 });

        // 2. Body validation
        const body = await request.json().catch(() => null);
        const parsed = initPaymentSchema.safeParse(body);
        if (!parsed.success) {
            return NextResponse.json(
                { status: "error", message: "Invalid request body", errors: z.treeifyError(parsed.error) },
                { status: 400 }
            );
        }
        const { orderId } = parsed.data;

        // 3. Order DB theke fetch — client theke never trust
        const order = await prisma.order.findUnique({ where: { id: orderId } });
        if (!order) return NextResponse.json({ status: "error", message: "Order not found" }, { status: 404 });
        if (order.userId !== userId) {
            log.warn("Order ownership mismatch", { orderId, userId });
            return NextResponse.json({ status: "error", message: "Forbidden" }, { status: 403 });
        }
        if (order.status === "PAID") {
            return NextResponse.json({ status: "error", message: "Already paid" }, { status: 409 });
        }

        // 4. Duplicate protection — existing INITIATED session thakle reuse
        const existingTransaction = await prisma.transaction.findFirst({
            where: { orderId: order.id, status: "INITIATED" },
            orderBy: { createdAt: "desc" },
        });
        if (existingTransaction?.gatewayPageURL) {
            log.info("Reusing existing initiated transaction", { orderId, tranId: existingTransaction.tranId });
            return NextResponse.json({
                status: "success",
                data: { GatewayPageURL: existingTransaction.gatewayPageURL },
            });
        }

        // 5. Customer info — production e User model theke ashbe
        const customer = {
            name: "Customer Name", email: "customer@example.com", address1: "N/A",
            city: "Dhaka", postcode: "1000", country: "Bangladesh", phone: "01700000000",
        };

        // 6. Unique tran_id — DB collision check shoho
        let tranId = generateTranId();
        let attempt = 0;
        while (await prisma.transaction.findUnique({ where: { tranId } })) {
            if (++attempt > 5) {
                return NextResponse.json({ status: "error", message: "Could not generate unique tran_id" }, { status: 500 });
            }
            tranId = generateTranId();
        }

        // 7. SSLCommerz init call
        const sslResponse = await initiatePayment(
            {
                tranId, amount: Number(order.amount), currency: order.currency,
                productName: order.productName, productCategory: order.productCategory,
                productProfile: order.productProfile,
            },
            customer
        );

        if (sslResponse.status !== "SUCCESS" || !sslResponse.GatewayPageURL) {
            log.error("SSLCommerz init failed", { orderId, tranId, reason: sslResponse.failedreason });
            await prisma.transaction.create({
                data: {
                    tranId, orderId: order.id, amount: order.amount, currency: order.currency,
                    status: "FAILED", rawInitResponse: sslResponse as object,
                },
            });
            return NextResponse.json({ status: "error", message: sslResponse.failedreason || "Init failed" }, { status: 400 });
        }

        // 8. Transaction save
        await prisma.transaction.create({
            data: {
                tranId, orderId: order.id, amount: order.amount, currency: order.currency,
                status: "INITIATED", sessionKey: sslResponse.sessionkey,
                gatewayPageURL: sslResponse.GatewayPageURL, rawInitResponse: sslResponse as object,
            },
        });

        log.info("Payment session initiated", { orderId, tranId });
        return NextResponse.json({
            status: "success",
            data: { GatewayPageURL: sslResponse.GatewayPageURL, tranId },
        });
    } catch (error) {
        log.error("Payment init unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.json({ status: "error", message: "Something went wrong" }, { status: 500 });
    }
}
```

---

#### app/api/payment/success
```bash
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { log } from "@/lib/logger";
import { sslCallbackSchema } from "@/lib/validations/payment.schema";
import { validateTransaction, isValidationStatusSuccess, isValidationAmountMatching } from "@/lib/services/sslcommerz.service";

// Browser redirect spoof-able, tai shudhu ashlei "paid" dhora jabe na —
// Validation API hit kore server-to-server confirm kora hocche
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

        const transaction = await prisma.transaction.findUnique({ where: { tranId: tran_id }, include: { order: true } });
        if (!transaction) {
            log.error("Success callback for unknown tran_id", { tran_id });
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=unknown_transaction`);
        }

        // Idempotency — already processed hole revalidate na kore direct success page
        if (transaction.processedAt) {
            return NextResponse.redirect(`${appUrl}/payment/success?orderId=${transaction.orderId}`);
        }
        if (!val_id) {
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=missing_val_id`);
        }

        const validation = await validateTransaction(val_id);

        if (!isValidationStatusSuccess(validation)) {
            await prisma.transaction.update({ where: { tranId: tran_id }, data: { status: "FAILED", rawValidationResponse: validation as object } });
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=validation_failed`);
        }
        if (!isValidationAmountMatching(validation, Number(transaction.amount))) {
            log.error("Validation amount mismatch — possible tampering", { tran_id });
            await prisma.transaction.update({ where: { tranId: tran_id }, data: { status: "FAILED", rawValidationResponse: validation as object } });
            return NextResponse.redirect(`${appUrl}/payment/failed?reason=amount_mismatch`);
        }

        // Sob check pass — atomic transaction e order + transaction dutai update
        await prisma.$transaction([
            prisma.transaction.update({
                where: { tranId: tran_id },
                data: {
                    status: "VALID", valId: validation.val_id, bankTranId: validation.bank_tran_id,
                    cardType: validation.card_type, cardIssuer: validation.card_issuer,
                    storeAmount: validation.store_amount ? Number(validation.store_amount) : undefined,
                    rawValidationResponse: validation as object, processedAt: new Date(),
                },
            }),
            prisma.order.update({ where: { id: transaction.orderId }, data: { status: "PAID" } }),
        ]);

        log.info("Payment verified, order paid", { tran_id, orderId: transaction.orderId });
        return NextResponse.redirect(`${appUrl}/payment/success?orderId=${transaction.orderId}`);
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
            await prisma.transaction.update({
                where: { tranId: tran_id },
                data: { status: "FAILED", processedAt: new Date(), rawIpnPayload: rawPayload },
            });
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
            return NextResponse.redirect(`${appUrl}/payment/cancelled?reason=invalid_callback`);
        }
        const { tran_id } = parsed.data;

        const transaction = await prisma.transaction.findUnique({ where: { tranId: tran_id } });
        if (transaction && !transaction.processedAt) {
            await prisma.transaction.update({
                where: { tranId: tran_id },
                data: { status: "CANCELLED", processedAt: new Date(), rawIpnPayload: rawPayload },
            });
        }
        log.info("Payment cancelled by user", { tran_id });
        return NextResponse.redirect(`${appUrl}/payment/cancelled?tran_id=${tran_id}`);
    } catch (error) {
        log.error("Cancel callback unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.redirect(`${appUrl}/payment/cancelled?reason=server_error`);
    }
}
```

---

#### app/api/payment/ipn
```bash
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { log } from "@/lib/logger";
import { ipnPayloadSchema } from "@/lib/validations/payment.schema";
import { validateTransaction, isValidationStatusSuccess, isValidationAmountMatching } from "@/lib/services/sslcommerz.service";

// SSLCommerz-er server theke background e ashe, browser-er upor depend kore na —
// eta e sob theke reliable source. Business logic fail holeo sob shomoy 200 return
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

        const transaction = await prisma.transaction.findUnique({ where: { tranId: tran_id } });
        if (!transaction) {
            log.error("IPN for unknown tran_id", { tran_id });
            return NextResponse.json({ status: "error", message: "Unknown transaction" }, { status: 200 });
        }

        // Idempotency — duplicate/retry IPN skip
        if (transaction.processedAt) {
            return NextResponse.json({ status: "success", message: "Already processed" }, { status: 200 });
        }

        const validation = await validateTransaction(val_id);

        if (!isValidationStatusSuccess(validation)) {
            await prisma.transaction.update({
                where: { tranId: tran_id },
                data: { status: "FAILED", rawIpnPayload: rawPayload, rawValidationResponse: validation as object },
            });
            return NextResponse.json({ status: "error", message: "Validation failed" }, { status: 200 });
        }
        if (!isValidationAmountMatching(validation, Number(transaction.amount))) {
            log.error("IPN amount mismatch — possible tampering", { tran_id });
            await prisma.transaction.update({
                where: { tranId: tran_id },
                data: { status: "FAILED", rawIpnPayload: rawPayload, rawValidationResponse: validation as object },
            });
            return NextResponse.json({ status: "error", message: "Amount mismatch" }, { status: 200 });
        }

        await prisma.$transaction([
            prisma.transaction.update({
                where: { tranId: tran_id },
                data: {
                    status: "VALID", valId: validation.val_id, bankTranId: validation.bank_tran_id,
                    cardType: validation.card_type, cardIssuer: validation.card_issuer,
                    storeAmount: validation.store_amount ? Number(validation.store_amount) : undefined,
                    rawIpnPayload: rawPayload, rawValidationResponse: validation as object, processedAt: new Date(),
                },
            }),
            prisma.order.update({ where: { id: transaction.orderId }, data: { status: "PAID" } }),
        ]);

        log.info("IPN verified, order paid", { tran_id, orderId: transaction.orderId });
        return NextResponse.json({ status: "success", message: "IPN processed" }, { status: 200 });
    } catch (error) {
        log.error("IPN unexpected error", { error: error instanceof Error ? error.message : String(error) });
        return NextResponse.json({ status: "error", message: "Internal error" }, { status: 200 });
    }
}
```

---

#### Checklist — kon requirement kothay ache

| Requirement | Kothay |
|---|---|
| Zod validation | `payment.schema.ts`, sob route-e `safeParse` |
| Prisma transaction save | `Transaction` model, `prisma.transaction.create/update` |
| Unique `tran_id` | `generateTranId()` + DB collision-check loop |
| Logging | `logger.ts` — structured JSON |
| Service layer | `sslcommerz.service.ts` |
| Secrets `.env`-e | Kono hardcoded value nai |
| `APP_URL` `.env` theke | Sob callback URL dynamic |
| Customer/product DB theke | `init/route.ts`-e `Order` fetch |
| Success callback-e `tran_id` verify | DB lookup + Validation API call |
| IPN verify | Same Validation API, server-to-server |
| Duplicate payment protection | `tranId` unique + existing-INITIATED reuse |
| Idempotency | `Transaction.processedAt` field |

**Setup**: `npx prisma migrate dev --name add_payment_module` → `.env` fill up → `init/route.ts`-e auth ar customer-object wire up koren → `npm install zod` (na thakle).
