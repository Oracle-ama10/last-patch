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

| Event Name | Audience | Trigger Condition |
| :--- | :--- | :--- |
| **`queue.updated`** | Customer, Staff Tablet | เมื่อคิวถูกเรียก (**`CALLED`**) หรือสถานะคิวเปลี่ยน. |
| **`table.status_changed`** | Staff Tablet, POS | เมื่อพนักงานเปลี่ยนสถานะโต๊ะ (เช่น `SERVING` $\to$ `CHECKOUT` $\to$ `VACANT`). |

***