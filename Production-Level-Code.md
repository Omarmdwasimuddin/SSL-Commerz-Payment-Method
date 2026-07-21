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

