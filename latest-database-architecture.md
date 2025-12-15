Dưới đây là **kế hoạch thiết kế cơ sở dữ liệu cho toàn hệ thống** sau khi chốt **Phương án Keycloak (không Auth Service, không Auth DB)**. Mình trình bày theo đúng 4 mục bạn yêu cầu và theo chuẩn **database-per-service** (mỗi service 1 DB/schema riêng).

> Nguyên tắc chung:
>
> * **Identity source of truth**: Keycloak
> * `user_id` trong toàn hệ thống = **JWT `sub`** (UUID)
> * Không lưu password/refresh token trong hệ thống
> * Không join DB giữa services; chỉ “logical reference” bằng UUID

---

# A) Tổng quan phân chia DB theo Service

* `user_db` (User Service)
* `catalog_db` (Catalog Service)
* `schedule_db` (Schedule Service)
* `booking_db` (Booking Service)
* `payment_db` (Payment Service)
* `notification_db` (Notification Service – optional)
* (Gateway không cần DB hoặc chỉ config/metrics; Keycloak có DB riêng của Keycloak)

---

# 1) Cần những bảng nào? Mục đích từng bảng

## 1.1 User Service DB (`user_db`)

1. **users**
   Lưu **profile nội bộ** của người dùng (mapping với Keycloak `sub`).

2. **staff_profiles** *(optional)*
   Thông tin nghiệp vụ riêng cho nhân viên spa (specialty, availability, location…).

3. **user_role_snapshots** *(optional/khuyến nghị)*
   Snapshot role tại thời điểm lưu profile (phục vụ reporting), dù role “source” là token.

---

## 1.2 Catalog Service DB (`catalog_db`)

1. **service_categories** *(optional)*
   Phân loại dịch vụ (massage, facial…).

2. **spa_services**
   Danh mục dịch vụ: tên, thời lượng, giá, trạng thái.

3. **price_history** *(optional, nếu muốn audit giá)*
   Lịch sử thay đổi giá (giúp audit/analytics).

---

## 1.3 Schedule Service DB (`schedule_db`)

1. **staff_schedules**
   Ca làm/khung làm việc của staff theo ngày.

2. **slots**
   Khung giờ có thể đặt: AVAILABLE/HELD/BOOKED.

3. **slot_holds** *(khuyến nghị mạnh)*
   Quản lý hold slot có TTL, tránh double-booking.

4. **slot_events** *(optional)*
   Audit mọi thay đổi slot (held/confirmed/released).

---

## 1.4 Booking Service DB (`booking_db`)

1. **bookings**
   Transaction đặt lịch: tham chiếu `slot_id`, `spa_service_id`, `user_id=sub`.

2. **booking_status_history** *(khuyến nghị)*
   Audit lịch sử trạng thái booking.

3. **booking_outbox** *(khuyến nghị nếu event-driven)*
   Outbox pattern để publish event tin cậy.

---

## 1.5 Payment Service DB (`payment_db`)

1. **payment_transactions**
   Lưu giao dịch thanh toán theo `booking_id`.

2. **payment_outbox** *(optional)*
   Nếu Payment phát event ra broker (PaymentSuccess/Failed).

---

## 1.6 Notification Service DB (`notification_db`) – optional

1. **notification_logs**
   Audit việc gửi email/SMS/push.

---

# 2) Define các cột của mỗi bảng

## 2.1 `user_db`

### Table: `users`

* `user_id` (uuid, PK) — **= Keycloak `sub`**
* `full_name` (varchar(150), not null)
* `email` (varchar(255), null, unique if not null)
* `phone` (varchar(30), null, unique if not null)
* `role_snapshot` (varchar(20), not null) — CUSTOMER/STAFF/ADMIN (snapshot)
* `created_at`, `updated_at` (timestamptz)

### Table: `staff_profiles` (optional)

* `staff_id` (uuid, PK, FK → users.user_id)
* `specialty` (varchar(100), null)
* `location_id` (uuid, null)
* `is_available` (boolean, not null default true)
* `created_at`, `updated_at`

### Table: `user_role_snapshots` (optional)

* `id` (uuid, PK)
* `user_id` (uuid, FK → users.user_id)
* `role` (varchar(20))
* `captured_at` (timestamptz)

---

## 2.2 `catalog_db`

### Table: `service_categories` (optional)

* `category_id` (uuid, PK)
* `name` (varchar(100), not null, unique case-insensitive)
* `description` (text, null)
* `created_at`, `updated_at`

### Table: `spa_services`

* `spa_service_id` (uuid, PK)
* `name` (varchar(150), not null)
* `description` (text, null)
* `duration_minutes` (int, not null, >0)
* `price` (numeric(12,2), not null, >0)
* `currency` (varchar(3), not null)
* `is_active` (boolean, not null default true)
* `category_id` (uuid, FK → service_categories.category_id, null)
* `created_at`, `updated_at`

### Table: `price_history` (optional)

* `id` (uuid, PK)
* `spa_service_id` (uuid, FK → spa_services)
* `old_price` (numeric)
* `new_price` (numeric)
* `changed_at` (timestamptz)
* `changed_by` (uuid, null) — admin userId snapshot

---

## 2.3 `schedule_db`

### Table: `staff_schedules`

* `schedule_id` (uuid, PK)
* `staff_id` (uuid, not null) — logical ref to users (staff)
* `work_date` (date, not null)
* `start_time` (time, not null)
* `end_time` (time, not null)
* `break_start`, `break_end` (time, null)
* `location_id` (uuid, null)
* `created_at`, `updated_at`
* Unique: `(staff_id, work_date)`

### Table: `slots`

* `slot_id` (uuid, PK)
* `staff_id` (uuid, not null)
* `work_date` (date, not null)
* `start_time` (time, not null)
* `end_time` (time, not null)
* `status` (varchar(20), not null) — AVAILABLE/HELD/BOOKED
* `held_until` (timestamptz, null) — chỉ khi HELD
* `booking_id` (uuid, null) — logical ref booking-service, chỉ khi BOOKED
* `created_at`, `updated_at`
* Unique: `(staff_id, work_date, start_time, end_time)`

### Table: `slot_holds`

* `hold_id` (uuid, PK)
* `slot_id` (uuid, FK → slots.slot_id)
* `booking_id` (uuid, not null)
* `held_at` (timestamptz, not null)
* `expires_at` (timestamptz, not null)
* `status` (varchar(20), not null) — ACTIVE/EXPIRED/RELEASED
* Unique partial: `slot_id WHERE status='ACTIVE'`

### Table: `slot_events` (optional)

* `event_id` (uuid, PK)
* `slot_id` (uuid)
* `event_type` (varchar(50)) — HELD/CONFIRMED/RELEASED/EXPIRED
* `payload` (jsonb)
* `created_at` (timestamptz)

---

## 2.4 `booking_db`

### Table: `bookings`

* `booking_id` (uuid, PK)
* `user_id` (uuid, not null) — **Keycloak sub**
* `spa_service_id` (uuid, not null) — logical ref catalog
* `slot_id` (uuid, not null) — logical ref schedule
* `status` (varchar(20), not null) — PENDING/CONFIRMED/CANCELLED/REJECTED
* `total_price` (numeric(12,2), not null)
* `currency` (varchar(3), not null)
* `note` (text, null)
* `booking_time` (timestamptz)
* `created_at`, `updated_at`
* `cancelled_at` (timestamptz, null)
* `cancellation_reason` (varchar(255), null)
* Unique partial: `slot_id WHERE status in ('PENDING','CONFIRMED')`

### Table: `booking_status_history`

* `id` (uuid, PK)
* `booking_id` (uuid, FK → bookings.booking_id)
* `from_status` (varchar(20), null)
* `to_status` (varchar(20), not null)
* `changed_at` (timestamptz)
* `changed_by` (uuid, null) — user/admin snapshot
* `reason` (varchar(255), null)

### Table: `booking_outbox`

* `event_id` (uuid, PK)
* `aggregate_type` (varchar(50)) default ‘Booking’
* `aggregate_id` (uuid, FK → bookings.booking_id)
* `event_type` (varchar(100)) — BookingCreated/Confirmed/Cancelled
* `payload` (jsonb)
* `status` (varchar(20)) — NEW/PROCESSING/SENT/FAILED
* `created_at`, `sent_at`

---

## 2.5 `payment_db`

### Table: `payment_transactions`

* `payment_id` (uuid, PK)
* `booking_id` (uuid, not null) — logical ref booking
* `amount` (numeric(12,2), not null)
* `currency` (varchar(3), not null)
* `provider` (varchar(50), not null) — MOCK/STRIPE/…
* `status` (varchar(20), not null) — INIT/SUCCESS/FAILED
* `created_at`, `updated_at`
* Unique partial: `booking_id WHERE status='SUCCESS'`

### Table: `payment_outbox` (optional)

* tương tự booking_outbox

---

## 2.6 `notification_db` (optional)

### Table: `notification_logs`

* `notification_id` (uuid, PK)
* `user_id` (uuid, not null)
* `channel` (varchar(20)) — EMAIL/SMS/PUSH
* `template` (varchar(50))
* `payload` (jsonb)
* `status` (varchar(20)) — SENT/FAILED
* `sent_at` (timestamptz, null)
* `created_at` (timestamptz)

---

# 3) Quan hệ giữa các bảng

## Trong từng DB (FK thật)

* `user_db`:

  * `staff_profiles.staff_id` → `users.user_id`
  * `user_role_snapshots.user_id` → `users.user_id`

* `catalog_db`:

  * `spa_services.category_id` → `service_categories.category_id`
  * `price_history.spa_service_id` → `spa_services.spa_service_id`

* `schedule_db`:

  * `slot_holds.slot_id` → `slots.slot_id`

* `booking_db`:

  * `booking_status_history.booking_id` → `bookings.booking_id`
  * `booking_outbox.aggregate_id` → `bookings.booking_id`

* `payment_db`: *(không FK cross DB)*

* `notification_db`: *(không FK cross DB)*

## Quan hệ logical (cross-service, không FK)

* `bookings.user_id` → Keycloak `sub` (và user_db.users.user_id)
* `bookings.slot_id` → schedule_db.slots.slot_id
* `bookings.spa_service_id` → catalog_db.spa_services.spa_service_id
* `payment_transactions.booking_id` → booking_db.bookings.booking_id
* `schedule_db.slots.booking_id` → booking_db.bookings.booking_id (chỉ khi BOOKED)

---

# 4) Thứ tự thiết kế các bảng (toàn hệ thống)

Mình ưu tiên theo dependency nghiệp vụ + dễ chạy integration test:

## Phase 1 (nền tảng identity & catalog)

1. **Keycloak** (realm/client/roles) *(không phải DB của bạn, nhưng phải có trước)*
2. `user_db.users`
3. `user_db.staff_profiles` (optional)
4. `catalog_db.service_categories` (optional)
5. `catalog_db.spa_services`
6. `catalog_db.price_history` (optional)

## Phase 2 (schedule resource)

7. `schedule_db.staff_schedules`
8. `schedule_db.slots`
9. `schedule_db.slot_holds`
10. `schedule_db.slot_events` (optional)

## Phase 3 (transaction booking)

11. `booking_db.bookings`
12. `booking_db.booking_status_history`
13. `booking_db.booking_outbox`

## Phase 4 (payment + notification)

14. `payment_db.payment_transactions`
15. `payment_db.payment_outbox` (optional)
16. `notification_db.notification_logs` (optional)

---

Nếu bạn muốn mình “đóng gói” luôn:

* tạo bộ **DDL scripts** (`/db/user_db.sql`, `/db/booking_db.sql`, …)
* kèm **Flyway migrations** versioned (`V1__init.sql`, `V2__indexes.sql`)
* kèm `docker-compose` dựng toàn bộ Postgres + Keycloak + gateway

…thì mình có thể soạn theo đúng cấu trúc multi-repo bạn đang theo.
