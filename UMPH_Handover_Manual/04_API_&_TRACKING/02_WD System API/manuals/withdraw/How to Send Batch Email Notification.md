---
type: manual
system: reg-withdraw
aliases: [Batch Email, Send Email Notification, ส่งอีเมลแจ้งเตือน]
tags: [guide, workflow]
created: 2026-03-30
---

# 📖 How to Send Batch Email Notification

> [!abstract] Overview (ภาพรวม)
> คู่มือนี้อธิบายขั้นตอนการที่เจ้าหน้าที่ทะเบียนส่ง batch email เพื่อเตือนอาจารย์/ที่ปรึกษา/คณบดีที่มีคำร้องค้างการพิจารณา และแจ้งนักศึกษาเกี่ยวกับผลการถอน  
> ใช้เมื่อ: ต้องการ remind อาจารย์ก่อน due date หรือแจ้งผลนักศึกษาเป็น batch

## ✅ Prerequisites (สิ่งที่ต้องเตรียมก่อนเริ่มงาน)

- [ ] มีสิทธิ์เจ้าหน้าที่ทะเบียน (Staff)
- [ ] DB มีข้อมูล DUEDATE ใน `SEND_AUTO_EMAIL` procedure
- [ ] SMTP Gmail account `coursewithdraw.reg@mfu.ac.th` ใช้งานได้
- [ ] (หมายเหตุ: Endpoint เหล่านี้ไม่มี JWT authentication)

## ⚙️ Step-by-Step Guide (ขั้นตอนการทำงาน)

### 🖱️ วิธีการกดส่งเมลด้วยมือผ่าน Postman (Manual Email Trigger)
ในช่วงที่มีการเปิดให้ถอนรายวิชา เจ้าหน้าที่ต้องทำการส่ง batch email แจ้งเตือนผู้เกี่ยวข้อง **ทุกเช้า** ผ่านโปรแกรม Postman โดยมีขั้นตอนดังนี้:

#### 1. ส่ง Batch Email แจ้งเตือนผู้พิจารณา (อาจารย์ / ที่ปรึกษา / คณบดี)
ส่งเมลเตือนอาจารย์ผู้สอน, ที่ปรึกษา, และคณบดีที่มีคำร้องค้างการพิจารณาอยู่ในระบบ:

*   **HTTP Method & Endpoint:**
    ```text
    ┌───────┬──────────────────────────────────────────────────────────────┐
    │ POST  │ {API_BASE_URL}/reg/withdraw/staff/sendmail                   │
    └───────┴──────────────────────────────────────────────────────────────┘
    ```
*   **ขั้นตอนการดำเนินงาน:**
    1. เปิดโปรแกรม **Postman**
    2. ตั้งค่า Method เป็น **POST**
    3. ระบุ URL: `{API_BASE_URL}/reg/withdraw/staff/sendmail` *(ตัวอย่าง: `http://localhost:5000/reg/withdraw/staff/sendmail`)*
    4. คลิกปุ่ม **Send**
*   **การทำงานเบื้องหลัง:** ระบบดึงข้อมูลจาก `SEND_AUTO_EMAIL` 3 รอบ (type 1=ผู้สอน, 2=ที่ปรึกษา, 3=คณบดี) เพื่อส่งอีเมลแจ้งว่ามีคำร้องรอพิจารณาจำนวนเท่าใดภายใน Due Date

---

#### 2. ส่ง Batch Email แจ้งเตือนนักศึกษา
ส่งเมลแจ้งผลการพิจารณาการถอนและ Due Date ให้นักศึกษาทราบ:

*   **HTTP Method & Endpoint:**
    ```text
    ┌───────┬──────────────────────────────────────────────────────────────┐
    │ POST  │ {API_BASE_URL}/reg/withdraw/staff/sendmailstudent            │
    └───────┴──────────────────────────────────────────────────────────────┘
    ```
*   **ขั้นตอนการดำเนินงาน:**
    1. ตั้งค่า Method เป็น **POST** ใน Postman
    2. ระบุ URL: `{API_BASE_URL}/reg/withdraw/staff/sendmailstudent`
    3. คลิกปุ่ม **Send**
*   **การทำงานเบื้องหลัง:** ระบบดึงข้อมูลจาก `SEND_AUTO_EMAIL` type 4 ส่งอีเมลแจ้งเตือนนักศึกษา

---

#### 3. ตรวจสอบผลการส่ง (Response Checking)
สังเกตผลตอบกลับจาก Postman (Response Body และ Status Code):

*   **กรณีส่งสำเร็จ (Success):**
    *   **Status Code:** `200 OK`
    *   **Response Body:**
        ```json
        { "code": 200, "message": { "en": "success" } }
        ```
*   **กรณีส่งไม่สำเร็จ (Failure):**
    *   **Status Code:** `500 Internal Server Error` (หาก SMTP มีปัญหา) หรือค้างส่งไม่ได้ ให้ตรวจสอบตามหัวข้อ Troubleshooting ด้านล่าง

> [!IMPORTANT]
> **ข้อพึงระวัง:** ให้ทำการกดส่งทั้ง 2 เส้นทางนี้ **ทุกวันตอนเช้า** ในช่วงระยะเวลาที่เปิดถอนรายวิชา เพื่อประสิทธิผลของระบบแจ้งเตือน

---

**โครงสร้างข้อมูลฟังก์ชัน `mail_teacher()` (สำหรับใช้อ้างอิง):**
```python
mail_teacher(
    to=i['OFF_EMAIL'],
    subject='Notification for course withdraw',
    ccode=i['COURSECODE'],
    cname=i['COURSENAMEENG'],
    offname='',
    offnameeng='',
    stdamt=i['COUNTOFSTUDENT'],
    ttype="1",         # 1=instructor, 2=advisor, 3=dean
    duedate=i['DUEDATE']
)
```

## ⚠️ Troubleshooting & Pitfalls (ข้อควรระวัง/ปัญหาที่พบบ่อย)

> [!CAUTION] กรณีส่งเมลไม่ได้ (SMTP / Email Delivery Issues)
> หากกดส่งผ่าน Postman แล้วขึ้น Error หรืออีเมลไม่ยอมถูกส่งออกไป ให้ตรวจทานตามขั้นตอนเหล่านี้:
> 
> 1. **ตรวจสอบความถูกต้องของอีเมลผู้รับ (Email Validation Check):**
>    - เข้าไปเช็คในระบบว่าอีเมลของอาจารย์หรือนักศึกษาคนดังกล่าว **มีสถานะขึ้นตัวหนังสือ Validate สีแดง** หรือมีรูปแบบไม่ถูกต้อง (เช่น สะกดผิด, มีช่องว่างเกิน) หรือไม่
> 
> 2. **ตรวจสอบการบล็อคบัญชีผู้ส่งของ Google (Allow Less Secure Apps):**
>    - บัญชีอีเมลระบบส่ง `coursewithdraw.reg@mfu.ac.th` อาจถูกระบบความปลอดภัยของ Google สกัดกั้น
>    - ให้เปิดเว็บเบราว์เซอร์ ล็อกอินด้วยเมลระบบ และเข้าลิงก์ด้านล่างเพื่อเปิดใช้งานสิทธิ์ **Less Secure Apps** (หรือตรวจสอบและตั้งค่า App Passwords ใหม่):
>      👉 [Google Account Less Secure Apps Settings](https://myaccount.google.com/lesssecureapps?pli=1&rapt=AEjHL4Pfvk0lug5Y1EOWsGQg2CuklFdebC4FFLOC_cs9BtMN83tuZJzRTT4fDwa4wab8Sb4BmyPXS08ieW584wFQi5HKxfbgog)

> [!warning] Authentication ถูก Disabled
> `/staff/sendmail` และ `/staff/sendmailstudent` ไม่มี `@jwt_required`  
> ใครก็ได้ที่รู้ URL สามารถ trigger batch email ได้ — ควรเปิด authentication ใน production

> [!warning] SMTP Credentials Hardcoded
> `coursewithdraw.reg@mfu.ac.th` + App Password อยู่ใน `routes.py` โดยตรง  
> หาก App Password หมดอายุหรือถูก revoke ทุก email function จะหยุดทำงาน  
> **แก้ไข:** ย้าย credentials ไป environment variable + ตรวจสอบอายุ App Password เป็นประจำ

> [!warning] ไม่มี retry mechanism
> หาก SMTP timeout ใน loop อีเมลบางรายการอาจไม่ถูกส่ง แต่ response ยังเป็น 200  
> **ตรวจสอบ:** ดู server log หรือ SMTP error ใน exception handler

## 🔗 Related

- [[Withdraw Email Notification]] — รายละเอียด email template และ positions
- [[Withdraw Staff Management]] — ความสามารถอื่นๆ ของ Staff

Source: `misapi/reg/withdraw/routes.py`
