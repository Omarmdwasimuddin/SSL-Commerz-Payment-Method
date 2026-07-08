## heading...

#### Architecture
```bash
app/api/
- success/route.ts
- fail/route.ts
- cancel/route.ts
- payment/route.ts
- ipn/route.ts
```
---

#### app/api/payment
```bash
import { NextRequest, NextResponse } from "next/server";


export async function POST(request: NextRequest) {
    try {

        const tran_id = (Math.floor(100000 + Math.random() * 900000)).toString();
        const init_url = "https://sandbox.sslcommerz.com/gwprocess/v4/api.php";

        const formData = new FormData();
        formData.append("store_id", process.env.SSLCOMMERZ_STORE_ID as string);
        formData.append("store_passwd", process.env.SSLCOMMERZ_STORE_PASSWORD as string);
        formData.append("total_amount", "1000");
        formData.append("currency", "BDT");
        formData.append("tran_id", `${tran_id}`);
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

        const requestOptions: RequestInit = { method: 'POST', body: formData };
        let SSLResponse = await fetch(init_url, requestOptions);

        const SSLResponseJson = await SSLResponse.json();

        return NextResponse.json(
            {status: "success", data: SSLResponseJson},
            {status: 200}
        )
    } catch (error) {
        console.error("Error :", error);
        return NextResponse.json(
            {status: "error"},
            {status: 500}
        )
    }
}
```
---

#### app/api/success
```bash
import { NextRequest, NextResponse } from "next/server";


export async function POST(request: NextRequest) {
    let {searchParams} = new URL(request.url);
    let tran_id = searchParams.get("tran_id");

    /* 
        According to tran_id
        Update your payment status = "Success" in your database
    */

    return NextResponse.redirect(new URL('/success', request.url), 303)
}
```
---

#### app/api/cancel
```bash
import { NextRequest, NextResponse } from "next/server";


export async function POST(request: NextRequest) {
    let {searchParams} = new URL(request.url);
    let tran_id = searchParams.get("tran_id");

    /* 
        According to tran_id
        Update your payment status = "Cancel" in your database
    */

    return NextResponse.redirect(new URL('/cancel', request.url), 303)
}
```
---

#### app/api/fail
```bash
import { NextRequest, NextResponse } from "next/server";


export async function POST(request: NextRequest) {
    let {searchParams} = new URL(request.url);
    let tran_id = searchParams.get("tran_id");

    /* 
        According to tran_id
        Update your payment status = "Fail" in your database
    */

    return NextResponse.redirect(new URL('/fail', request.url), 303)
}
```
---

#### app/api/ipn
```bash
import { NextRequest, NextResponse } from "next/server";


export async function POST(request: NextRequest) {
    let json = await request.json();
    let tran_id = json['tran_id'];
    let val_id = json['val_id'];
    let status = json['status'];

    /* 
    
        According to tran_id
        Update your payment status = status
        Update your payment validation id = val_id
    
    */

    return NextResponse.json({ status: "success" });
}
```
---
