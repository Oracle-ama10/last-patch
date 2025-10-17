## 3. API Specification & Idempotency (`API_Spec.md`)

### 3.1. Core Endpoints (6+ Examples)

| Method | Endpoint | Description | Business Rules |
| :--- | :--- | :--- | :--- |
| **POST** | `/queue-tickets` | ลูกค้าขอรับหมายเลขคิวใหม่ | ต้องระบุ `party_size` $\ge$ 1. |
| **GET** | `/queue-tickets/{id}` | ตรวจสอบสถานะและตำแหน่งคิว | ใช้สำหรับลูกค้าติดตามคิว. |
| **GET** | `/tables` | ดึงสถานะผังโต๊ะทั้งหมด | ต้องมี **Auth: Staff/Manager**. |
| **POST** | `/orders` | สร้างออร์เดอร์หลักใหม่ | ต้องมี **Auth**. ต้องมี `table_id` หรือ `is_takeaway: true`. |
| **POST** | `/orders/{id}/items` | เพิ่ม/อัปเดตรายการอาหารในออร์เดอร์ | Order Status: `PENDING_ITEMS` หรือ `SERVING`. |
| **POST** | `/payments` | สร้างรายการชำระเงิน (PromptPay) | ต้องมี **Header: `X-Idempotency-Key`**. Order Status: `CHECKOUT`. |

### 3.2. Idempotency for POST /payments

* **Mechanism:** Client (POS) ต้องส่ง **`X-Idempotency-Key`** (UUID) ใน Header.
* **API Action:** API จะตรวจสอบ Key ก่อนประมวลผล. หาก Key เคยใช้สำเร็จแล้ว จะคืนค่า Transaction เดิม **(`200 OK` / `202 Accepted`)** โดยไม่สร้าง Payment ซ้ำ.
* **Response Code:** **`202 Accepted`** (เมื่อสร้าง QR Code สำเร็จ) แสดงว่า Request ได้รับการยอมรับและกำลังรอผลยืนยัน (Webhook).

### 3.3. Realtime Events (WSS)

| Event Name | Audience | Description | Trigger Condition |
| :--- | :--- | :--- | :--- |
| **`queue.updated`** | Customer, Staff Tablet | แจ้งเตือนเมื่อสถานะคิวเปลี่ยน หรือมีการเรียกคิวใหม่ | เมื่อคิวถูกเรียก (**`CALLED`**) หรือสถานะคิวเปลี่ยน. |
| **`table.status_changed`** | Staff Tablet, POS | แจ้งเตือนเมื่อสถานะโต๊ะเปลี่ยน | เมื่อพนักงานเปลี่ยนสถานะโต๊ะ (เช่น `SERVING` $\to$ `CHECKOUT` $\to$ `VACANT`). |
| **`order.status_changed`** | POS, Kitchen Display | แจ้งเตือนเมื่อออร์เดอร์เปลี่ยนสถานะ เช่น การชำระเงินสำเร็จ (PAID) |เมื่อPaymentGatewayส่งCallback ยืนยันการชำระเงินเสร็จ. |


### 3.4. รูปแบบ Response และ Error (Consistency)

 Success Response (2xx)
 โครงสร้างสม่ำเสมอ: success: true และมี Block data

 ```
 // ตัวอย่าง: HTTP 201 Created สำหรับ /queue-tickets
{
  "success": true,
  "data": {
    "ticket_id": "A007",
    "status": "WAITING",
    "eta_minutes": 15
  },
  "metadata": {
    "timestamp": "2025-10-17T11:00:00Z"
  }
}
```


 Error Response (4xx, 5xx)
 โครงสร้างสม่ำเสมอ: success: false และมี Block error พร้อม code ที่ชัดเจน

 ```
 // ตัวอย่าง: HTTP 409 Conflict (Business Rule Violation)
{
  "success": false,
  "error": {
    "code": "ORDER_LOCKED",
    "message": "Cannot process payment. Order is not in CHECKOUT status.",
    "http_status": 409
  }
}
```

***