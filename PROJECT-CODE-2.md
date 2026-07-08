## 

```bash
import { NextRequest, NextResponse } from "next/server";

interface SSLCommerzInitResponse {
    status: "SUCCESS" | "FAILED";
    failedreason?: string;
    sessionkey?: string;
    GatewayPageURL?: string;
    [key: string]: unknown;
}

export async function POST(request: NextRequest) {
    try {
        // Env var guard — na thakle "undefined" string SSLCommerz e chole jabe, tai check kora hocche
        if (!process.env.SSLCOMMERZ_STORE_ID || !process.env.SSLCOMMERZ_STORE_PASSWORD) {
            console.error("SSLCommerz env vars missing");
            return NextResponse.json(
                { status: "error", message: "Payment gateway not configured" },
                { status: 500 }
            );
        }

        const init_url = "https://sandbox.sslcommerz.com/gwprocess/v4/api.php";
        const tran_id = `TXN-${Date.now()}-${Math.floor(1000 + Math.random() * 9000)}`;

        const formData = new FormData();
        formData.append("store_id", process.env.SSLCOMMERZ_STORE_ID);
        formData.append("store_passwd", process.env.SSLCOMMERZ_STORE_PASSWORD);
        formData.append("total_amount", "1000");
        formData.append("currency", "BDT");
        formData.append("tran_id", tran_id);
        formData.append("success_url", `http://localhost:3000/api/success?tran_id=${tran_id}`);
        formData.append("fail_url", `http://localhost:3000/api/fail?tran_id=${tran_id}`);
        formData.append("cancel_url", `http://localhost:3000/api/cancel?tran_id=${tran_id}`);
        formData.append("ipn_url", "http://localhost:3000/api/ipn");
        formData.append("cus_name", "Wasim Uddin");
        formData.append("cus_email", "wasim.uddin@example.com");
        formData.append("cus_add1", "Dewanhat Chittagong");
        formData.append("cus_add2", "Dewanhat Chittagong");
        formData.append("cus_city", "Chittagong");
        formData.append("cus_state", "Chittagong");
        formData.append("cus_postcode", "4000");
        formData.append("cus_country", "Bangladesh");
        formData.append("cus_phone", "01700000000");
        formData.append("cus_fax", "01700000000");
        formData.append("shipping_method", "Yes");
        formData.append("ship_name", "Wasim Uddin");
        formData.append("ship_add1", "Dewanhat Chittagong");
        formData.append("ship_add2", "Dewanhat Chittagong");
        formData.append("ship_city", "Chittagong");
        formData.append("ship_state", "Chittagong");
        formData.append("ship_postcode", "4000");
        formData.append("ship_country", "Bangladesh");
        formData.append("product_name", "Computer");
        formData.append("product_category", "Electronic");
        formData.append("product_profile", "general");
        formData.append("product_amount", "3");

        const requestOptions: RequestInit = { method: "POST", body: formData };
        const SSLResponse = await fetch(init_url, requestOptions);

        if (!SSLResponse.ok) {
            console.error("SSLCommerz init HTTP error:", SSLResponse.status);
            return NextResponse.json(
                { status: "error", message: "Payment gateway request failed" },
                { status: 502 }
            );
        }

        const SSLResponseJson = (await SSLResponse.json()) as SSLCommerzInitResponse;

        // Actual gateway status check — age eta na thakay always "success" return hoto
        if (SSLResponseJson.status !== "SUCCESS" || !SSLResponseJson.GatewayPageURL) {
            console.error("SSLCommerz init failed:", SSLResponseJson.failedreason);
            return NextResponse.json(
                {
                    status: "error",
                    message: SSLResponseJson.failedreason || "Failed to initiate payment session",
                    data: SSLResponseJson,
                },
                { status: 400 }
            );
        }

        return NextResponse.json(
            {
                status: "success",
                message: "Payment session initiated successfully",
                data: SSLResponseJson,
            },
            { status: 200 }
        );
    } catch (error) {
        console.error("SSLCommerz init error:", error);
        return NextResponse.json(
            {
                status: "error",
                message:
                    error instanceof Error
                        ? error.message
                        : "Something went wrong while initiating payment",
            },
            { status: 500 }
        );
    }
}
```
---
