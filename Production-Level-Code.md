## SSLCommerz Production Payment Module

### Structure
```bash
prisma/schema-addition.prisma        → Order + Transaction model (apnar schema.prisma e merge korben)
.env.example                          → Environment variables list
src/lib/prisma.ts                     → Prisma client singleton
src/lib/logger.ts                     → Structured logger
src/lib/validations/payment.schema.ts → Zod validation schemas
src/lib/services/sslcommerz.service.ts→ SSLCommerz API calls (init + validation)
src/app/api/payment/init/route.ts     → Payment session start
src/app/api/payment/success/route.ts  → Success callback (server-side verify kore)
src/app/api/payment/fail/route.ts     → Fail callback
src/app/api/payment/cancel/route.ts   → Cancel callback
src/app/api/payment/ipn/route.ts      → IPN (background verification, most reliable source)
```
---
