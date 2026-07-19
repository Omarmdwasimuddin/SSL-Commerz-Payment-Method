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
  PENDING
  PAID
  FAILED
  CANCELLED
}

enum TransactionStatus {
  INITIATED
  VALID
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

  productId       String
  product         Product       @relation(fields: [productId], references: [id])

  quantity        Int           @default(1)
  totalAmount     Decimal       @db.Decimal(10, 2)
  currency        String        @default("BDT")

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
  tranId                 String            @unique

  orderId                String
  order                  Order             @relation(fields: [orderId], references: [id])

  amount                 Decimal           @db.Decimal(10, 2)
  currency               String            @default("BDT")
  status                 TransactionStatus @default(INITIATED)

  sessionKey             String?
  gatewayPageURL         String?

  valId                  String?
  bankTranId             String?
  cardType               String?
  cardIssuer             String?
  storeAmount            Decimal?          @db.Decimal(10, 2)

  rawInitResponse        Json?
  rawValidationResponse  Json?
  rawIpnPayload          Json?

  processedAt            DateTime?

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
    val_id?: string;
    amount?: string;
    store_amount?: string;
    currency_amount?: string;
    bank_tran_id?: string;
    card_type?: string;
    card_issuer?: string;
    [key: string]: unknown;
}

// FIXED: age address1/postcode/country chilo, kintu Order model-e shudhu
// customerAddress ar customerCity ache — interface-ta match kore chotto kora hoise
interface CustomerInfo {
    name: string;
    email: string;
    phone: string;
    address?: string;
    city?: string;
}

// FIXED: productCategory/productProfile Order/Product kono model-e nai bole
// bad kora hoise — function-er ভিতরে hardcode kora hocche
interface OrderInfo {
    tranId: string;
    amount: number;
    currency: string;
    productName: string;
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
    formData.append("cus_add1", customer.address || "N/A");
    formData.append("cus_city", customer.city || "N/A");
    formData.append("cus_postcode", "1000");
    formData.append("cus_country", "Bangladesh");
    formData.append("cus_phone", customer.phone);

    formData.append("shipping_method", "NO");
    formData.append("num_of_item", "1");
    formData.append("product_name", order.productName);
    formData.append("product_category", "E-commerce");
    formData.append("product_profile", "general");

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

export async function validateTransaction(valId: string): Promise<SSLCommerzValidationResponse> {
    const { storeId, storePassword, apiBaseUrl } = getEnvConfig();

    const params = new URLSearchParams({ val_id: valId, store_id: storeId, store_passwd: storePassword, format: "json" });
    const response = await fetch(`${apiBaseUrl}/validator/api/validationserverAPI.php?${params}`, { method: "GET" });

    if (!response.ok) {
        log.error("SSLCommerz validation HTTP error", { httpStatus: response.status, valId });
        throw new Error(`Validation request failed with HTTP ${response.status}`);
    }
    return (await response.json()) as SSLCommerzValidationResponse;
}

export function isValidationAmountMatching(validation: SSLCommerzValidationResponse, expectedAmount: number): boolean {
    const receivedAmount = parseFloat(validation.currency_amount ?? validation.amount ?? "0");
    return Math.abs(receivedAmount - expectedAmount) < 0.01;
}

export function isValidationStatusSuccess(validation: SSLCommerzValidationResponse): boolean {
    return validation.status === "VALID" || validation.status === "VALIDATED";
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
import { NextRequest, NextResponse, userAgent } from "next/server";
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
        };

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

#### lib/auth.config.ts
```bash

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
import { prisma } from "@/lib/prisma";
import { log } from "@/lib/logger";
import { initPaymentSchema } from "@/lib/validations/payment.schema";
import { generateTranId, initiatePayment } from "@/lib/services/sslcommerz.service";
// import { auth } from "@/lib/auth";

const SESSION_VALIDITY_MINUTES = 55; // SSLCommerz session ~1hr e expire hoy, tai buffer rekhe 55 min

export async function POST(request: NextRequest) {
    try {
        // 1. Auth check
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

        // 3. Order + Product DB theke fetch (relation include kore)
        const order = await prisma.order.findUnique({
            where: { id: orderId },
            include: { product: true },
        });
        if (!order) return NextResponse.json({ status: "error", message: "Order not found" }, { status: 404 });
        if (order.userId !== userId) {
            log.warn("Order ownership mismatch", { orderId, userId });
            return NextResponse.json({ status: "error", message: "Forbidden" }, { status: 403 });
        }
        if (order.status !== "PENDING") {
            return NextResponse.json({ status: "error", message: `Order is already ${order.status.toLowerCase()}` }, { status: 409 });
        }

        // 4. Duplicate protection — recent (not expired) INITIATED session thakle reuse
        const sessionCutoff = new Date(Date.now() - SESSION_VALIDITY_MINUTES * 60 * 1000);
        const existingTransaction = await prisma.transaction.findFirst({
            where: { orderId: order.id, status: "INITIATED", createdAt: { gte: sessionCutoff } },
            orderBy: { createdAt: "desc" },
        });
        if (existingTransaction?.gatewayPageURL) {
            log.info("Reusing existing initiated transaction", { orderId, tranId: existingTransaction.tranId });
            return NextResponse.json({
                status: "success",
                data: { GatewayPageURL: existingTransaction.gatewayPageURL, tranId: existingTransaction.tranId },
            });
        }

        // 5. Unique tran_id
        let tranId = generateTranId();
        let attempt = 0;
        while (await prisma.transaction.findUnique({ where: { tranId } })) {
            if (++attempt > 5) {
                return NextResponse.json({ status: "error", message: "Could not generate unique tran_id" }, { status: 500 });
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
            log.error("SSLCommerz init failed", { orderId, tranId, reason: sslResponse.failedreason });
            await prisma.transaction.create({
                data: {
                    tranId, orderId: order.id, amount: order.totalAmount, currency: order.currency,
                    status: "FAILED", rawInitResponse: sslResponse as object,
                },
            });
            return NextResponse.json({ status: "error", message: sslResponse.failedreason || "Init failed" }, { status: 400 });
        }

        // 7. Transaction save
        await prisma.transaction.create({
            data: {
                tranId, orderId: order.id, amount: order.totalAmount, currency: order.currency,
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
            await prisma.$transaction([
                prisma.transaction.update({
                    where: { tranId: tran_id },
                    data: { status: "CANCELLED", processedAt: new Date(), rawIpnPayload: rawPayload },
                }),
                prisma.order.updateMany({
                    where: { id: transaction.orderId, status: "PENDING" },
                    data: { status: "CANCELLED" },
                }),
            ]);
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
