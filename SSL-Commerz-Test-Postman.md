# SSLCommerz Test with Postman

একটি স্টেপ-বাই-স্টেপ গাইড যা দেখাবে কিভাবে Postman দিয়ে SSLCommerz Sandbox Payment Gateway API টেস্ট করা যায়।

---

## ধাপ ১: URL ও Method সেট করা

Postman এ নিচের URL দাও এবং Method হিসেবে **POST** সিলেক্ট করো:

```
https://sandbox.sslcommerz.com/gwprocess/v4/api.php
```

---

## ধাপ ২: Body তে form-data দেওয়া

Body ট্যাবে গিয়ে **form-data** সিলেক্ট করো এবং নিচের key-value গুলো বসাও:

```bash
store_id = paste store_id
store_passwd = paste store_passwd
total_amount = 1000
currency = BDT
tran_id = XYZ-123-SECRET
success_url = http://localhost:3000/api/success
fail_url = http://localhost:3000/api/fail
cancel_url = http://localhost:3000/api/cancel
ipn_url = http://localhost:3000/api/ipn
cus_name = Wasim
cus_email = mdwasimu015@gmail.com
cus_add1 = Dhaka
cus_add2 = Dhaka
cus_city = Dhaka
cus_state = Dhaka
cus_postcode = 1000
cus_country = Bangladesh
cus_phone = 01711111111 
cus_fax = 01711111111
ship_name = Wasim
ship_add1 = Dhaka
ship_add2 = Dhaka
ship_city = Dhaka
ship_state = Dhaka
ship_postcode = 1000
ship_country = Bangladesh
multi_card_name = mastercard
value_a = ref001_A
value_b = ref002_B
value_c = ref003_C
value_d = ref004_D
```

![Step 2 - Request Setup](https://imgur.com/bQNf5ew.png)
![Step 2 - Response](https://imgur.com/7YtUFzF.png)

---

## ধাপ ৩: redirectGatewayURL এ যাওয়া

Response এ পাওয়া **redirectGatewayURL** এ Ctrl চেপে ক্লিক করলে OTP পেজে নিয়ে যাবে।

- **Success** বাটনে ক্লিক করলে → `success_url` এ নিয়ে যাবে
- **Failed** বাটনে ক্লিক করলে → `fail_url` এ নিয়ে যাবে

![Step 3 - OTP & Redirect](https://imgur.com/ioio1r8.png)

---

## ধাপ ৪: Transaction Record চেক করা

Payment success হলে SSLCommerz এর Transactions এ রেকর্ড হবে।

![Step 4 - Transaction Record](https://imgur.com/87j3lvC.png)

---

✅ **Postman দিয়ে SSLCommerz Payment Gateway Test সম্পন্ন!**
