Dưới đây là **kế hoạch thiết kế CSDL** dựa trên business Spa Booking (microservices). Vì theo microservices, mình sẽ trình bày theo **database-per-service**: mỗi service sở hữu DB riêng (hoặc ít nhất schema riêng), **không join cross-service**.

> Ghi chú: ID dùng `UUID` (PostgreSQL: `uuid`, SQL Server: `uniqueidentifier`).

---

# A. BOOKING SERVICE DB

## 1) Bảng cần có & mục đích

1. **bookings**
   Lưu “giao dịch đặt lịch” (transaction). Đây là core table của booking-service.

2. **booking_status_history** (khuyến nghị)
   Audit lịch sử trạng thái booking (PENDING → CONFIRMED → CANCELLED…), phục vụ debug & tracking.

3. **booking_outbox** (khuyến nghị nếu event-driven)
   Implement **Outbox Pattern** để publish event an toàn (BookingCreated/Confirmed/Cancelled) tránh mất event khi DB commit thành công mà publish fail.

---

## 2) Define cột

### Table: `bookings`

| Column              | Type          | Null | Mục đích                             |
| ------------------- | ------------- | ---: | ------------------------------------ |
| booking_id          | uuid (PK)     |   NO | ID booking                           |
| user_id             | uuid          |   NO | Logical ref tới user-service         |
| spa_service_id      | uuid          |   NO | Logical ref tới catalog-service      |
| slot_id             | uuid          |   NO | Logical ref tới schedule-service     |
| status              | varchar(20)   |   NO | PENDING/CONFIRMED/CANCELLED/REJECTED |
| total_price         | numeric(12,2) |   NO | Snapshot giá tại thời điểm đặt       |
| currency            | varchar(3)    |   NO | VND/USD…                             |
| note                | text          |  YES | Ghi chú khách                        |
| booking_time        | timestamptz   |   NO | Thời điểm khách tạo booking          |
| created_at          | timestamptz   |   NO | Audit                                |
| updated_at          | timestamptz   |   NO | Audit                                |
| cancelled_at        | timestamptz   |  YES | Audit khi hủy                        |
| cancellation_reason | varchar(255)  |  YES | lý do hủy                            |

**Indexes khuyến nghị**

* idx_bookings_user_id (user_id)
* idx_bookings_slot_id (slot_id)
* idx_bookings_status (status)
* Unique (slot_id) **có cân nhắc**:

    * Nếu bạn đảm bảo 1 slot chỉ có 1 booking active, bạn có thể unique theo `(slot_id)` hoặc **unique partial**: `(slot_id) WHERE status IN ('PENDING','CONFIRMED')` (Postgres hỗ trợ tốt).

### Table: `booking_status_history`

| Column      | Type                | Null | Mục đích         |
| ----------- | ------------------- | ---: | ---------------- |
| id          | uuid (PK)           |   NO | record id        |
| booking_id  | uuid (FK->bookings) |   NO | booking          |
| from_status | varchar(20)         |  YES | trạng thái trước |
| to_status   | varchar(20)         |   NO | trạng thái sau   |
| changed_at  | timestamptz         |   NO | thời điểm đổi    |
| changed_by  | uuid                |  YES | user/admin id    |
| reason      | varchar(255)        |  YES | ghi chú          |

Index: idx_bsh_booking_id (booking_id)

### Table: `booking_outbox`

| Column         | Type         | Null | Mục đích                           |
| -------------- | ------------ | ---: | ---------------------------------- |
| event_id       | uuid (PK)    |   NO | id event                           |
| aggregate_type | varchar(50)  |   NO | "Booking"                          |
| aggregate_id   | uuid         |   NO | booking_id                         |
| event_type     | varchar(100) |   NO | BookingCreated/Confirmed/Cancelled |
| payload        | jsonb        |   NO | dữ liệu event                      |
| status         | varchar(20)  |   NO | NEW/PROCESSING/SENT/FAILED         |
| created_at     | timestamptz  |   NO | audit                              |
| sent_at        | timestamptz  |  YES | audit                              |

Index: idx_outbox_status_created_at (status, created_at)

---

## 3) Quan hệ giữa các bảng (Booking DB)

* bookings (1) —— (*) booking_status_history
* bookings (1) —— (*) booking_outbox *(logical: aggregate_id)*

> user_id / spa_service_id / slot_id **không FK sang DB khác** (chỉ logical ref).

---

## 4) Thứ tự thiết kế (Booking DB)

1. `bookings`
2. `booking_status_history`
3. `booking_outbox`

---

# B. SCHEDULE SERVICE DB

## 1) Bảng cần có & mục đích

1. **staff_schedules**
   Lịch làm việc của nhân viên (ca làm, ngày làm, quy tắc tạo slot).

2. **slots**
   Các khung giờ có thể đặt. Đây là core table của schedule-service.

3. **slot_holds** (khuyến nghị)
   Giữ slot tạm thời cho booking pending (TTL). Giúp quản lý hold & auto release dễ hơn.

---

## 2) Define cột

### Table: `staff_schedules`

| Column      | Type        | Null | Mục đích                         |
| ----------- | ----------- | ---: | -------------------------------- |
| schedule_id | uuid (PK)   |   NO | id                               |
| staff_id    | uuid        |   NO | logical ref user-service (staff) |
| work_date   | date        |   NO | ngày làm                         |
| start_time  | time        |   NO | giờ bắt đầu                      |
| end_time    | time        |   NO | giờ kết thúc                     |
| break_start | time        |  YES | nghỉ trưa                        |
| break_end   | time        |  YES | nghỉ trưa                        |
| location_id | uuid        |  YES | chi nhánh (nếu multi-branch)     |
| created_at  | timestamptz |   NO | audit                            |
| updated_at  | timestamptz |   NO | audit                            |

Unique: (staff_id, work_date)

### Table: `slots`

| Column     | Type        | Null | Mục đích                                 |
| ---------- | ----------- | ---: | ---------------------------------------- |
| slot_id    | uuid (PK)   |   NO | id slot                                  |
| staff_id   | uuid        |   NO | nhân viên                                |
| work_date  | date        |   NO | ngày                                     |
| start_time | time        |   NO | bắt đầu                                  |
| end_time   | time        |   NO | kết thúc                                 |
| status     | varchar(20) |   NO | AVAILABLE/HELD/BOOKED                    |
| held_until | timestamptz |  YES | TTL hold                                 |
| booking_id | uuid        |  YES | logical ref booking-service (khi BOOKED) |
| created_at | timestamptz |   NO | audit                                    |
| updated_at | timestamptz |   NO | audit                                    |

Indexes:

* idx_slots_staff_date (staff_id, work_date)
* idx_slots_status (status)
* idx_slots_booking_id (booking_id)

Unique:

* (staff_id, work_date, start_time, end_time) để tránh slot trùng.

### Table: `slot_holds`

| Column     | Type             | Null | Mục đích                |
| ---------- | ---------------- | ---: | ----------------------- |
| hold_id    | uuid (PK)        |   NO | id                      |
| slot_id    | uuid (FK->slots) |   NO | slot                    |
| booking_id | uuid             |   NO | booking pending         |
| held_at    | timestamptz      |   NO | giữ lúc nào             |
| expires_at | timestamptz      |   NO | hết hạn                 |
| status     | varchar(20)      |   NO | ACTIVE/EXPIRED/RELEASED |

Index: idx_slot_holds_slot_id_status (slot_id, status)
Unique (slot_id) WHERE status='ACTIVE' (nếu Postgres)

---

## 3) Quan hệ giữa các bảng (Schedule DB)

* staff_schedules (1) —— (*) slots (theo staff_id + work_date)
* slots (1) —— (*) slot_holds

---

## 4) Thứ tự thiết kế (Schedule DB)

1. `staff_schedules`
2. `slots`
3. `slot_holds`

---

# C. CATALOG SERVICE DB

## 1) Bảng cần có & mục đích

1. **spa_services**: danh mục dịch vụ
2. **service_categories** (optional): phân loại dịch vụ

## 2) Define cột

### `service_categories` (optional)

| Column      | Type         | Null |
| ----------- | ------------ | ---: |
| category_id | uuid (PK)    |   NO |
| name        | varchar(100) |   NO |
| description | text         |  YES |

### `spa_services`

| Column           | Type                          | Null |
| ---------------- | ----------------------------- | ---: |
| spa_service_id   | uuid (PK)                     |   NO |
| name             | varchar(150)                  |   NO |
| description      | text                          |  YES |
| duration_minutes | int                           |   NO |
| price            | numeric(12,2)                 |   NO |
| currency         | varchar(3)                    |   NO |
| is_active        | boolean                       |   NO |
| category_id      | uuid (FK->service_categories) |  YES |
| created_at       | timestamptz                   |   NO |
| updated_at       | timestamptz                   |   NO |

Indexes: is_active, category_id, name (search)

## 3) Quan hệ

* service_categories (1) —— (*) spa_services

## 4) Thứ tự thiết kế

1. `service_categories`
2. `spa_services`

---

# D. AUTH SERVICE DB

## 1) Bảng cần có & mục đích

1. **user_credentials**: username/password hash, role
2. **refresh_tokens** (nếu có refresh)

## 2) Define cột

### `user_credentials`

| Column        | Type         | Null |                          |
| ------------- | ------------ | ---: | ------------------------ |
| credential_id | uuid (PK)    |   NO |                          |
| user_id       | uuid         |   NO | logical ref user-service |
| username      | varchar(100) |   NO |                          |
| password_hash | varchar(255) |   NO |                          |
| role          | varchar(20)  |   NO |                          |
| is_active     | boolean      |   NO |                          |
| created_at    | timestamptz  |   NO |                          |
| updated_at    | timestamptz  |   NO |                          |

Unique: username, user_id

### `refresh_tokens`

| Column     | Type         | Null |
| ---------- | ------------ | ---: |
| token_id   | uuid (PK)    |   NO |
| user_id    | uuid         |   NO |
| token_hash | varchar(255) |   NO |
| expires_at | timestamptz  |   NO |
| revoked_at | timestamptz  |  YES |

---

# E. USER SERVICE DB

## 1) Bảng cần có & mục đích

1. **users**: thông tin profile
2. **staff_profiles** (optional): thông tin nhân viên spa

## 2) Define cột

### `users`

| Column     | Type         | Null |
| ---------- | ------------ | ---: |
| user_id    | uuid (PK)    |   NO |
| full_name  | varchar(150) |   NO |
| phone      | varchar(30)  |  YES |
| email      | varchar(255) |  YES |
| role       | varchar(20)  |   NO |
| created_at | timestamptz  |   NO |
| updated_at | timestamptz  |   NO |

Unique: email (nếu bắt buộc), phone (nếu bắt buộc)

### `staff_profiles` (optional)

| Column       | Type                         | Null |
| ------------ | ---------------------------- | ---: |
| staff_id     | uuid (PK, FK->users.user_id) |   NO |
| specialty    | varchar(100)                 |  YES |
| location_id  | uuid                         |  YES |
| is_available | boolean                      |   NO |

---

# F. PAYMENT SERVICE DB

## 1) Bảng cần có & mục đích

1. **payment_transactions**: payment theo booking

## 2) Define cột

### `payment_transactions`

| Column     | Type          | Null |                             |
| ---------- | ------------- | ---: | --------------------------- |
| payment_id | uuid (PK)     |   NO |                             |
| booking_id | uuid          |   NO | logical ref booking-service |
| amount     | numeric(12,2) |   NO |                             |
| currency   | varchar(3)    |   NO |                             |
| provider   | varchar(50)   |   NO | MOCK/STRIPE/...             |
| status     | varchar(20)   |   NO | INIT/SUCCESS/FAILED         |
| created_at | timestamptz   |   NO |                             |
| updated_at | timestamptz   |   NO |                             |

Indexes: booking_id, status

---

# G. NOTIFICATION SERVICE DB (optional)

Nếu chỉ listen event và gửi mail thì DB có thể không cần. Nếu muốn audit:

## 1) Bảng

1. **notification_logs**: lưu log gửi thông báo

## 2) Define cột

### `notification_logs`

| Column          | Type        | Null |             |
| --------------- | ----------- | ---: | ----------- |
| notification_id | uuid (PK)   |   NO |             |
| user_id         | uuid        |   NO |             |
| channel         | varchar(20) |   NO | EMAIL/SMS   |
| template        | varchar(50) |   NO |             |
| payload         | jsonb       |   NO |             |
| status          | varchar(20) |   NO | SENT/FAILED |
| sent_at         | timestamptz |  YES |             |
| created_at      | timestamptz |   NO |             |

---

# H. Quan hệ tổng thể giữa các service (logical references)

```text
User Service: users.user_id
   ↑
Auth Service: user_credentials.user_id (logical)

Catalog Service: spa_services.spa_service_id
   ↑
Booking Service: bookings.spa_service_id (logical)

Schedule Service: slots.slot_id
   ↑
Booking Service: bookings.slot_id (logical)

Booking Service: bookings.booking_id
   ↑
Payment Service: payment_transactions.booking_id (logical)
   ↑
Schedule Service: slots.booking_id (logical when BOOKED)
```

---

# I. Thứ tự thiết kế toàn hệ thống (khuyến nghị)

Để triển khai mượt (dev/test) bạn nên thiết kế theo dependency:

1. **User Service DB** (users) – vì mọi thứ đều reference userId
2. **Auth Service DB** (credentials) – cần để login
3. **Catalog Service DB** (spa_services) – cần để chọn dịch vụ
4. **Schedule Service DB** (staff_schedules, slots, holds) – cần để có slot
5. **Booking Service DB** (bookings, history, outbox) – core transaction
6. **Payment Service DB** (payment_transactions)
7. **Notification Service DB** (optional: logs)

---

Nếu bạn nói rõ bạn dùng **PostgreSQL hay SQL Server**, mình có thể:

* Viết luôn **DDL script tạo bảng + index + constraints** cho từng service DB
* Đề xuất **strategy khoá slot** (unique partial / optimistic lock / transaction isolation) chi tiết cho Schedule Service
