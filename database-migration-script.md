Dưới đây là **DDL PostgreSQL** theo đúng thiết kế “database per service” (mỗi service 1 DB). Mình viết theo dạng **script riêng cho từng service DB**, gồm:

* tạo extension cần thiết
* tạo table
* index
* constraints (FK chỉ trong cùng DB)
* một số **partial unique index** để enforce business rule (đặc biệt cho slot/booking)

> Gợi ý chạy: mỗi block dưới đây chạy trong DB tương ứng (vd `booking_db`, `schedule_db`, …).
> Nếu bạn muốn 1 file tổng hợp theo folder `db/booking-service.sql`, `db/schedule-service.sql` mình cũng có thể gom lại.

---

# 1) BOOKING SERVICE DB (booking_db)

```sql
-- =========================
-- Booking Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) bookings
CREATE TABLE IF NOT EXISTS bookings (
  booking_id           uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id              uuid NOT NULL,
  spa_service_id       uuid NOT NULL,
  slot_id              uuid NOT NULL,

  status               varchar(20) NOT NULL,
  total_price          numeric(12,2) NOT NULL,
  currency             varchar(3) NOT NULL DEFAULT 'VND',

  note                 text NULL,

  booking_time         timestamptz NOT NULL DEFAULT now(),
  created_at           timestamptz NOT NULL DEFAULT now(),
  updated_at           timestamptz NOT NULL DEFAULT now(),

  cancelled_at         timestamptz NULL,
  cancellation_reason  varchar(255) NULL,

  CONSTRAINT chk_booking_status
    CHECK (status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED')),

  CONSTRAINT chk_total_price_non_negative
    CHECK (total_price > 0),

  CONSTRAINT chk_currency_len
    CHECK (char_length(currency) = 3)
);

-- Indexes for bookings
CREATE INDEX IF NOT EXISTS idx_bookings_user_id
  ON bookings(user_id);

CREATE INDEX IF NOT EXISTS idx_bookings_slot_id
  ON bookings(slot_id);

CREATE INDEX IF NOT EXISTS idx_bookings_status
  ON bookings(status);

CREATE INDEX IF NOT EXISTS idx_bookings_created_at
  ON bookings(created_at);

-- Business rule: one active booking per slot
-- Active statuses: PENDING, CONFIRMED
CREATE UNIQUE INDEX IF NOT EXISTS ux_bookings_slot_active
  ON bookings(slot_id)
  WHERE status IN ('PENDING','CONFIRMED');


-- 2) booking_status_history
CREATE TABLE IF NOT EXISTS booking_status_history (
  id           uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id   uuid NOT NULL,
  from_status  varchar(20) NULL,
  to_status    varchar(20) NOT NULL,
  changed_at   timestamptz NOT NULL DEFAULT now(),
  changed_by   uuid NULL,
  reason       varchar(255) NULL,

  CONSTRAINT fk_bsh_booking
    FOREIGN KEY (booking_id) REFERENCES bookings(booking_id)
    ON DELETE CASCADE,

  CONSTRAINT chk_bsh_to_status
    CHECK (to_status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED')),

  CONSTRAINT chk_bsh_from_status
    CHECK (from_status IS NULL OR from_status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED'))
);

CREATE INDEX IF NOT EXISTS idx_bsh_booking_id
  ON booking_status_history(booking_id);

CREATE INDEX IF NOT EXISTS idx_bsh_changed_at
  ON booking_status_history(changed_at);


-- 3) booking_outbox (Outbox pattern)
CREATE TABLE IF NOT EXISTS booking_outbox (
  event_id         uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  aggregate_type   varchar(50) NOT NULL DEFAULT 'Booking',
  aggregate_id     uuid NOT NULL,  -- booking_id
  event_type       varchar(100) NOT NULL,
  payload          jsonb NOT NULL,
  status           varchar(20) NOT NULL DEFAULT 'NEW',
  created_at       timestamptz NOT NULL DEFAULT now(),
  sent_at          timestamptz NULL,

  CONSTRAINT fk_outbox_booking
    FOREIGN KEY (aggregate_id) REFERENCES bookings(booking_id)
    ON DELETE CASCADE,

  CONSTRAINT chk_outbox_status
    CHECK (status IN ('NEW','PROCESSING','SENT','FAILED'))
);

CREATE INDEX IF NOT EXISTS idx_outbox_status_created
  ON booking_outbox(status, created_at);

CREATE INDEX IF NOT EXISTS idx_outbox_event_type
  ON booking_outbox(event_type);
```

---

# 2) SCHEDULE SERVICE DB (schedule_db)

```sql
-- =========================
-- Schedule Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) staff_schedules
CREATE TABLE IF NOT EXISTS staff_schedules (
  schedule_id   uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  staff_id      uuid NOT NULL,
  work_date     date NOT NULL,

  start_time    time NOT NULL,
  end_time      time NOT NULL,

  break_start   time NULL,
  break_end     time NULL,

  location_id   uuid NULL,

  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_schedule_time_range
    CHECK (start_time < end_time),

  CONSTRAINT chk_schedule_break_range
    CHECK (
      (break_start IS NULL AND break_end IS NULL)
      OR (break_start IS NOT NULL AND break_end IS NOT NULL AND break_start < break_end)
    ),

  CONSTRAINT chk_schedule_break_within_shift
    CHECK (
      (break_start IS NULL AND break_end IS NULL)
      OR (break_start >= start_time AND break_end <= end_time)
    )
);

-- Unique schedule per staff per day
CREATE UNIQUE INDEX IF NOT EXISTS ux_staff_schedules_staff_date
  ON staff_schedules(staff_id, work_date);

CREATE INDEX IF NOT EXISTS idx_staff_schedules_work_date
  ON staff_schedules(work_date);


-- 2) slots
CREATE TABLE IF NOT EXISTS slots (
  slot_id      uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  staff_id     uuid NOT NULL,
  work_date    date NOT NULL,

  start_time   time NOT NULL,
  end_time     time NOT NULL,

  status       varchar(20) NOT NULL DEFAULT 'AVAILABLE',
  held_until   timestamptz NULL,

  booking_id   uuid NULL, -- logical ref to booking-service

  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_slot_time_range
    CHECK (start_time < end_time),

  CONSTRAINT chk_slot_status
    CHECK (status IN ('AVAILABLE','HELD','BOOKED')),

  CONSTRAINT chk_slot_hold_consistency
    CHECK (
      (status = 'HELD' AND held_until IS NOT NULL)
      OR (status <> 'HELD' AND held_until IS NULL)
    ),

  CONSTRAINT chk_slot_booking_consistency
    CHECK (
      (status = 'BOOKED' AND booking_id IS NOT NULL)
      OR (status <> 'BOOKED' AND booking_id IS NULL)
    )
);

-- Avoid duplicate time blocks for same staff/day
CREATE UNIQUE INDEX IF NOT EXISTS ux_slots_staff_date_time
  ON slots(staff_id, work_date, start_time, end_time);

CREATE INDEX IF NOT EXISTS idx_slots_staff_date
  ON slots(staff_id, work_date);

CREATE INDEX IF NOT EXISTS idx_slots_status
  ON slots(status);

CREATE INDEX IF NOT EXISTS idx_slots_booking_id
  ON slots(booking_id);

-- Helpful for cleaning expired holds:
CREATE INDEX IF NOT EXISTS idx_slots_held_until
  ON slots(held_until)
  WHERE status = 'HELD';


-- 3) slot_holds
CREATE TABLE IF NOT EXISTS slot_holds (
  hold_id     uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  slot_id     uuid NOT NULL,
  booking_id  uuid NOT NULL, -- booking pending

  held_at     timestamptz NOT NULL DEFAULT now(),
  expires_at  timestamptz NOT NULL,

  status      varchar(20) NOT NULL DEFAULT 'ACTIVE',

  CONSTRAINT fk_slot_holds_slot
    FOREIGN KEY (slot_id) REFERENCES slots(slot_id)
    ON DELETE CASCADE,

  CONSTRAINT chk_slot_hold_status
    CHECK (status IN ('ACTIVE','EXPIRED','RELEASED')),

  CONSTRAINT chk_slot_hold_expiry
    CHECK (expires_at > held_at)
);

CREATE INDEX IF NOT EXISTS idx_slot_holds_slot_status
  ON slot_holds(slot_id, status);

CREATE INDEX IF NOT EXISTS idx_slot_holds_expires_at
  ON slot_holds(expires_at);

-- Business rule: only 1 ACTIVE hold per slot at a time
CREATE UNIQUE INDEX IF NOT EXISTS ux_slot_holds_slot_active
  ON slot_holds(slot_id)
  WHERE status = 'ACTIVE';
```

---

# 3) CATALOG SERVICE DB (catalog_db)

```sql
-- =========================
-- Catalog Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) service_categories (optional)
CREATE TABLE IF NOT EXISTS service_categories (
  category_id   uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  name          varchar(100) NOT NULL,
  description   text NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_service_categories_name
  ON service_categories(lower(name));


-- 2) spa_services
CREATE TABLE IF NOT EXISTS spa_services (
  spa_service_id     uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  name               varchar(150) NOT NULL,
  description        text NULL,

  duration_minutes   int NOT NULL,
  price              numeric(12,2) NOT NULL,
  currency           varchar(3) NOT NULL DEFAULT 'VND',

  is_active          boolean NOT NULL DEFAULT true,
  category_id        uuid NULL,

  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT fk_spa_services_category
    FOREIGN KEY (category_id) REFERENCES service_categories(category_id)
    ON DELETE SET NULL,

  CONSTRAINT chk_duration_positive
    CHECK (duration_minutes > 0),

  CONSTRAINT chk_price_positive
    CHECK (price > 0),

  CONSTRAINT chk_currency_len
    CHECK (char_length(currency) = 3)
);

CREATE INDEX IF NOT EXISTS idx_spa_services_active
  ON spa_services(is_active);

CREATE INDEX IF NOT EXISTS idx_spa_services_category
  ON spa_services(category_id);

-- Search optimization (simple):
CREATE INDEX IF NOT EXISTS idx_spa_services_name_lower
  ON spa_services(lower(name));
```

---

# 4) AUTH SERVICE DB (auth_db)

```sql
-- =========================
-- Auth Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) user_credentials
CREATE TABLE IF NOT EXISTS user_credentials (
  credential_id   uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         uuid NOT NULL, -- logical ref to user-service

  username        varchar(100) NOT NULL,
  password_hash   varchar(255) NOT NULL,
  role            varchar(20) NOT NULL,
  is_active       boolean NOT NULL DEFAULT true,

  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_auth_role
    CHECK (role IN ('CUSTOMER','STAFF','ADMIN'))
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_user_credentials_username
  ON user_credentials(lower(username));

CREATE UNIQUE INDEX IF NOT EXISTS ux_user_credentials_user_id
  ON user_credentials(user_id);

CREATE INDEX IF NOT EXISTS idx_user_credentials_role
  ON user_credentials(role);

CREATE INDEX IF NOT EXISTS idx_user_credentials_active
  ON user_credentials(is_active);


-- 2) refresh_tokens (optional)
CREATE TABLE IF NOT EXISTS refresh_tokens (
  token_id      uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id       uuid NOT NULL,
  token_hash    varchar(255) NOT NULL,
  expires_at    timestamptz NOT NULL,
  revoked_at    timestamptz NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_refresh_expires_future
    CHECK (expires_at > created_at)
);

CREATE INDEX IF NOT EXISTS idx_refresh_tokens_user_id
  ON refresh_tokens(user_id);

CREATE INDEX IF NOT EXISTS idx_refresh_tokens_expires_at
  ON refresh_tokens(expires_at);

-- Ensure only one non-revoked token_hash if you want (optional):
-- CREATE UNIQUE INDEX ... ON refresh_tokens(token_hash);
```

---

# 5) USER SERVICE DB (user_db)

```sql
-- =========================
-- User Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) users
CREATE TABLE IF NOT EXISTS users (
  user_id      uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  full_name    varchar(150) NOT NULL,
  phone        varchar(30) NULL,
  email        varchar(255) NULL,
  role         varchar(20) NOT NULL,

  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_user_role
    CHECK (role IN ('CUSTOMER','STAFF','ADMIN'))
);

-- optional uniqueness (tùy policy của bạn)
CREATE UNIQUE INDEX IF NOT EXISTS ux_users_email
  ON users(lower(email))
  WHERE email IS NOT NULL;

CREATE UNIQUE INDEX IF NOT EXISTS ux_users_phone
  ON users(phone)
  WHERE phone IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_users_role
  ON users(role);


-- 2) staff_profiles (optional)
CREATE TABLE IF NOT EXISTS staff_profiles (
  staff_id       uuid PRIMARY KEY,
  specialty      varchar(100) NULL,
  location_id    uuid NULL,
  is_available   boolean NOT NULL DEFAULT true,

  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT fk_staff_profiles_user
    FOREIGN KEY (staff_id) REFERENCES users(user_id)
    ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_staff_profiles_location
  ON staff_profiles(location_id);

CREATE INDEX IF NOT EXISTS idx_staff_profiles_available
  ON staff_profiles(is_available);
```

---

# 6) PAYMENT SERVICE DB (payment_db)

```sql
-- =========================
-- Payment Service DB
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1) payment_transactions
CREATE TABLE IF NOT EXISTS payment_transactions (
  payment_id    uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id    uuid NOT NULL, -- logical ref to booking-service

  amount        numeric(12,2) NOT NULL,
  currency      varchar(3) NOT NULL DEFAULT 'VND',
  provider      varchar(50) NOT NULL DEFAULT 'MOCK',
  status        varchar(20) NOT NULL DEFAULT 'INIT',

  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_payment_amount_positive
    CHECK (amount > 0),

  CONSTRAINT chk_payment_currency_len
    CHECK (char_length(currency) = 3),

  CONSTRAINT chk_payment_status
    CHECK (status IN ('INIT','SUCCESS','FAILED'))
);

CREATE INDEX IF NOT EXISTS idx_payment_booking_id
  ON payment_transactions(booking_id);

CREATE INDEX IF NOT EXISTS idx_payment_status
  ON payment_transactions(status);

-- optional: allow only one SUCCESS per booking
CREATE UNIQUE INDEX IF NOT EXISTS ux_payment_one_success_per_booking
  ON payment_transactions(booking_id)
  WHERE status = 'SUCCESS';
```

---

# 7) NOTIFICATION SERVICE DB (notification_db) — optional

```sql
-- =========================
-- Notification Service DB (optional)
-- =========================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS notification_logs (
  notification_id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         uuid NOT NULL, -- logical ref user-service
  channel         varchar(20) NOT NULL, -- EMAIL/SMS
  template        varchar(50) NOT NULL,
  payload         jsonb NOT NULL,
  status          varchar(20) NOT NULL DEFAULT 'SENT',
  sent_at         timestamptz NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),

  CONSTRAINT chk_notification_channel
    CHECK (channel IN ('EMAIL','SMS','PUSH')),

  CONSTRAINT chk_notification_status
    CHECK (status IN ('SENT','FAILED'))
);

CREATE INDEX IF NOT EXISTS idx_notification_user_id
  ON notification_logs(user_id);

CREATE INDEX IF NOT EXISTS idx_notification_created_at
  ON notification_logs(created_at);

CREATE INDEX IF NOT EXISTS idx_notification_status
  ON notification_logs(status);
```

---

## Lưu ý quan trọng về “updated_at”

DDL ở trên chưa auto-update `updated_at` khi UPDATE. Nếu bạn muốn DB tự cập nhật, có thể dùng trigger chung cho mỗi DB (Postgres). Nói là làm luôn:

```sql
-- Add once per DB if you want automatic updated_at:
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Example: for bookings
DROP TRIGGER IF EXISTS trg_bookings_updated_at ON bookings;
CREATE TRIGGER trg_bookings_updated_at
BEFORE UPDATE ON bookings
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

Bạn có thể lặp trigger cho các bảng có `updated_at`.

---

Nếu bạn muốn, mình có thể tiếp tục theo 2 hướng (bạn chọn 1):

1. Viết **docker-compose.yml** dựng toàn bộ các DB + init scripts tự chạy DDL theo từng service.
2. Viết DDL thành **các file .sql tách theo repo/service** kèm versioning kiểu migration (Flyway/Liquibase).
