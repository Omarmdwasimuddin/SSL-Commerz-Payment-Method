## SSL Commerz Test with Postman

#### Postman e URL daw---> https://sandbox.sslcommerz.com/gwprocess/v4/api.php --->Method hobe: POST

#### Body ---> form-data --->
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
---

![](https://imgur.com/sxRsiMA.png)
![](https://imgur.com/7YtUFzF.png)

---

#### redirectGatewayURL Ctrl chepe click korle---> OTP page e niye jabe --->Success button click korle success url e niye jabe abar failed button e click korle fail url e niye jabe
![](https://imgur.com/ioio1r8.png)

#### payment success hole---> ssl commerz er transections e record hobe.
![](https://imgur.com/87j3lvC.png)
