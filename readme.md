# 🍜 Kopihub Queue & Ordering System Design (System Exam)

## Overview (C1: System Context)

ระบบออกแบบมาเพื่อจัดการคิวหน้าร้าน การสั่งอาหาร และการชำระเงิน PromptPay สำหรับร้าน Kopihub Dimsum Cafe โดยมีเป้าหมายหลักในการรองรับการเติบโต และการจัดการแบบ Real-time (Latency $\leq$ 1-2s)

**Key Flows:**
1. Customer scans QR/LINE to get a Queue Ticket.
2. Staff assigns a table using a Tablet app (Real-time floor plan).
3. Staff/Customer places an Order (Paper or Digital input).
4. POS processes PromptPay payment (Idempotent) and prints a receipt.

***

## 1. C2: Containers & Architecture (3-Tier)

ระบบใช้สถาปัตยกรรมแบบ **3-Tier** และแยกส่วนงานออกเป็น Containers เพื่อรองรับการ Scale และ Maintenance

| Container | Tier | Protocols หลัก | Key Responsibilities |
| :--- | :--- | :--- | :--- |
| **Customer Frontend** | Presentation | **HTTPS**, **WSS** | Queue Status Tracking, Menu Display |
| **Staff Tablet App** | Presentation | **HTTPS**, **WSS** | Real-time Table Management, Order Taking |
| **POS App (Cashier)** | Presentation | **HTTPS** | Billing, **Idempotent** Payment, Printing |
| **API & Realtime Server** | Application | **REST API (HTTPS), HTTP/HTTPS**  | Core Business Logic, Auth (RBAC/JWT), Idempotency Check |
| **Background Worker** | Application | **Queue, HTTPS (Webhook)** | Asynchronous Task: Printing, Retrying/Queue, Payment Webhook Processing |
| **Realtime Service** | Application | **WSS, Redis Pub/Sub**  | Fan-out Notifications (Queue/Table/Order Status $\leq 2s$) |
| **Database** | Data | **SQL** (PostgreSQL) | Persistent Data (e.g., Orders, Tickets, Tables, Audit Logs) |
| **Cache** | Data | **Redis Protocol** | Real-time Status Cache, **Rate Limiting** |

**Rationale Highlights:**
* **WSS (WebSocket):** ใช้สำหรับ Realtime Service เพื่อให้การอัปเดตสถานะคิว/โต๊ะมีความหน่วงต่ำ ($\leq$ 1-2s).
* **Worker/Queue:** ใช้สำหรับงาน **Printing** เพื่อ Decouple API จาก Printer Latency และเพิ่ม Fault Tolerance.

***

## 2. C3: API Server Component Breakdown

API Service ถูกแบ่งเป็น Microservices/Modules ภายใน Container เพื่อแบ่งขอบเขตความรับผิดชอบ (Separation of Concerns):

| Component | Responsibility (หน้าที่) | Interface / Calls | Dependency / Call Direction |
| :--- | :--- | :--- | :--- |
| **AuthService** | JWT Verification, **RBAC** (Role-Based Access Control) | Middleware | DB Layer, Cache (Rate Limit) |
| **QueueService** | จัดการคิว (จอง, เรียกคิว), **จัดโต๊ะ**, อัปเดตสถานะคิว | `/queue`, `/tables` (REST) | **Realtime Service**, DB Layer |
| **OrderService** | สร้าง/อัปเดต Order, Validate Order Items | `/orders`, `/order-items` (REST) | DB Layer, **Worker Service** (Printing Job) |
| **PaymentService** | เชื่อมต่อ **PromptPay GW**, Idempotency Check, Transaction | `/payments`, `/webhook` (REST) | External Payment GW, DB Layer |
| **ReportService** | ประมวลผล Report ยอดขาย/เวลารอ (Near Real-time) | `/reports` (REST) | DB Layer, Worker Service (Background Report) |
| **NotificationService** | จัดการการส่ง Realtime Event ผ่าน WebSocket | `emit(event_name)` | Realtime Service |
| **DB Layer** | ORM/SQL Access, Data Validation | SQL | PostgreSQL |

***

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

## 4. Sequence Diagram: PromptPay Payment Flow



**Key Steps:**
1.  **POS/Staff** $\to$ **API Gateway**: Sends `POST /payments` with `Idempotency-Key`.
2.  **API Gateway** $\to$ **Payment Service**: Sends `createPaymentOrder`.
3.  **Payment Service** $\to$ **Payment Gateway (PromptPay)**: Sends `PGW.createQR(amount, ref)`.
4.  **Payment Gateway** $\to$ **Payment Service**: Returns `QR payload`.
5.  **Payment Service** $\to$ **API Gateway**: Returns `payment_id + QR`.
6.  **API Gateway** $\to$ **POS/Staff**: Returns `201 Created (QR)`.
7.  **Payment Gateway** $\to$ **API Service (Webhook)**: Sends `Webhook: payment succeeded`(Asynchronous Callback).
8.  **API Service** $\to$ **SQL Database**: Sends `UPDATE payments=paid, order=closed`.
9.  **API Service** $\to$ **Background Worker**: Sends `print_receipt`.
10. **Background Worker** $\to$ **Receipt Printer**: Sends `publish.payment.paid`.
11. **API Service** $\to$ **Realtime (WS)**: Sends `publish.payment.paid` (Realtime Event).
12. **POS/Staff** สามารถดึงสถานะซ้ำได้ด้วย `GET /payments/{id}` หรือรับสถานะผ่าน Realtime.
***

## 5. Non-functional Requirements & Quality

| Requirement | Implementation Detail |
| :--- | :--- |
| **Latency $\le 2s$** | ใช้ **Redis** Cache + **WebSocket** (WSS) สำหรับ Realtime updates. |
| **SLA 99%** | ใช้ Load Balancer, Multi-AZ Deployment, Worker Queue (Asynchronous processing). |
| **PDPA** | Data Masking/Tokenization สำหรับข้อมูลส่วนบุคคล (เช่น LINE ID/เบอร์โทรศัพท์). |
| **RBAC** | ใช้ **JWT** Token ตรวจสอบสิทธิ์ (Role-Based Access Control) ใน **AuthService** Middleware. |
| **Audit Log** | บันทึกทุก Transaction/การเปลี่ยนสถานะสำคัญ (เช่น การจ่ายเงิน, การยกเลิกคิว) ใน DB Layer. |
| **Monitoring** | Prometheus/Grafana เพื่อติดตาม Latency, Error Rate, และ **Queue Length**. |

//