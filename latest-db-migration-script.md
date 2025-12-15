Dưới đây là **bộ Liquibase DDL scripts** (tạo **tables + indexes + constraints**) theo **Phương án C (Keycloak, không Auth Service/DB)**, tách riêng cho từng service DB.

Mình cung cấp:

* Cấu trúc thư mục chuẩn cho **mỗi repo/service**
* `db.changelog-master.xml` (master)
* `V1__init.xml` (DDL chính)
* (Tuỳ chọn) `V2__triggers.xml` để auto-update `updated_at`
* Cấu hình Spring Boot để Liquibase chạy tự động

> ✅ Postgres compatible
> ✅ FK chỉ trong cùng DB (database-per-service)
> ✅ Partial unique index dùng `sql` (Liquibase không portable cho partial index)

---

# 0) Cấu trúc thư mục chung cho mỗi service repo

```text
<service-repo>/
  src/main/resources/
    db/changelog/
      db.changelog-master.xml
      changes/
        V1__init.xml
        V2__triggers.xml   (optional)
```

## Maven dependencies (mỗi service)

```xml
<dependency>
  <groupId>org.liquibase</groupId>
  <artifactId>liquibase-core</artifactId>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```

## application.yml (mỗi service)

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
```

---

# 1) USER SERVICE (`user_db`)

## `db.changelog-master.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="
    http://www.liquibase.org/xml/ns/dbchangelog
    https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <include file="db/changelog/changes/V1__init.xml" relativeToChangelogFile="false"/>
  <include file="db/changelog/changes/V2__triggers.xml" relativeToChangelogFile="false"/>
</databaseChangeLog>
```

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="
    http://www.liquibase.org/xml/ns/dbchangelog
    https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="user-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="user-010-users" author="system">
    <createTable tableName="users">
      <column name="user_id" type="uuid">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="full_name" type="varchar(150)">
        <constraints nullable="false"/>
      </column>

      <column name="email" type="varchar(255)"/>
      <column name="phone" type="varchar(30)"/>

      <column name="role_snapshot" type="varchar(20)">
        <constraints nullable="false"/>
      </column>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addCheckConstraint tableName="users"
      constraintName="chk_users_role"
      checkConstraint="role_snapshot IN ('CUSTOMER','STAFF','ADMIN')"/>

    <!-- email unique (case-insensitive), only when not null -->
    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_users_email
      ON users(lower(email))
      WHERE email IS NOT NULL;
    </sql>

    <!-- phone unique only when not null -->
    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_users_phone
      ON users(phone)
      WHERE phone IS NOT NULL;
    </sql>

    <createIndex indexName="idx_users_role" tableName="users">
      <column name="role_snapshot"/>
    </createIndex>
  </changeSet>

  <changeSet id="user-020-staff-profiles" author="system">
    <createTable tableName="staff_profiles">
      <column name="staff_id" type="uuid">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="specialty" type="varchar(100)"/>
      <column name="location_id" type="uuid"/>

      <column name="is_available" type="boolean" defaultValueBoolean="true">
        <constraints nullable="false"/>
      </column>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addForeignKeyConstraint
      constraintName="fk_staff_profiles_user"
      baseTableName="staff_profiles" baseColumnNames="staff_id"
      referencedTableName="users" referencedColumnNames="user_id"
      onDelete="CASCADE"/>

    <createIndex indexName="idx_staff_profiles_location" tableName="staff_profiles">
      <column name="location_id"/>
    </createIndex>

    <createIndex indexName="idx_staff_profiles_available" tableName="staff_profiles">
      <column name="is_available"/>
    </createIndex>
  </changeSet>

</databaseChangeLog>
```

## `changes/V2__triggers.xml` (optional)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="user-100-updated-at-trigger-fn" author="system">
    <sql>
      CREATE OR REPLACE FUNCTION set_updated_at()
      RETURNS trigger AS $$
      BEGIN
        NEW.updated_at = now();
        RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
    </sql>
  </changeSet>

  <changeSet id="user-110-users-updated-at-trigger" author="system">
    <sql>
      DROP TRIGGER IF EXISTS trg_users_updated_at ON users;
      CREATE TRIGGER trg_users_updated_at
      BEFORE UPDATE ON users
      FOR EACH ROW
      EXECUTE FUNCTION set_updated_at();
    </sql>
  </changeSet>

  <changeSet id="user-120-staff-profiles-updated-at-trigger" author="system">
    <sql>
      DROP TRIGGER IF EXISTS trg_staff_profiles_updated_at ON staff_profiles;
      CREATE TRIGGER trg_staff_profiles_updated_at
      BEFORE UPDATE ON staff_profiles
      FOR EACH ROW
      EXECUTE FUNCTION set_updated_at();
    </sql>
  </changeSet>
</databaseChangeLog>
```

---

# 2) CATALOG SERVICE (`catalog_db`)

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="cat-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="cat-010-service-categories" author="system">
    <createTable tableName="service_categories">
      <column name="category_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="name" type="varchar(100)">
        <constraints nullable="false"/>
      </column>
      <column name="description" type="text"/>
      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_service_categories_name
      ON service_categories(lower(name));
    </sql>
  </changeSet>

  <changeSet id="cat-020-spa-services" author="system">
    <createTable tableName="spa_services">
      <column name="spa_service_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="name" type="varchar(150)">
        <constraints nullable="false"/>
      </column>
      <column name="description" type="text"/>

      <column name="duration_minutes" type="int">
        <constraints nullable="false"/>
      </column>

      <column name="price" type="numeric(12,2)">
        <constraints nullable="false"/>
      </column>

      <column name="currency" type="varchar(3)" defaultValue="VND">
        <constraints nullable="false"/>
      </column>

      <column name="is_active" type="boolean" defaultValueBoolean="true">
        <constraints nullable="false"/>
      </column>

      <column name="category_id" type="uuid"/>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addForeignKeyConstraint
      constraintName="fk_spa_services_category"
      baseTableName="spa_services" baseColumnNames="category_id"
      referencedTableName="service_categories" referencedColumnNames="category_id"
      onDelete="SET NULL"/>

    <addCheckConstraint tableName="spa_services"
      constraintName="chk_duration_positive"
      checkConstraint="duration_minutes > 0"/>

    <addCheckConstraint tableName="spa_services"
      constraintName="chk_price_positive"
      checkConstraint="price > 0"/>

    <addCheckConstraint tableName="spa_services"
      constraintName="chk_currency_len"
      checkConstraint="char_length(currency) = 3"/>

    <createIndex indexName="idx_spa_services_active" tableName="spa_services">
      <column name="is_active"/>
    </createIndex>

    <createIndex indexName="idx_spa_services_category" tableName="spa_services">
      <column name="category_id"/>
    </createIndex>

    <sql>
      CREATE INDEX IF NOT EXISTS idx_spa_services_name_lower
      ON spa_services(lower(name));
    </sql>
  </changeSet>

</databaseChangeLog>
```

*(V2 trigger giống user-service nếu bạn muốn auto updated_at.)*

---

# 3) SCHEDULE SERVICE (`schedule_db`)

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="sch-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="sch-010-staff-schedules" author="system">
    <createTable tableName="staff_schedules">
      <column name="schedule_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="staff_id" type="uuid">
        <constraints nullable="false"/>
      </column>
      <column name="work_date" type="date">
        <constraints nullable="false"/>
      </column>
      <column name="start_time" type="time">
        <constraints nullable="false"/>
      </column>
      <column name="end_time" type="time">
        <constraints nullable="false"/>
      </column>
      <column name="break_start" type="time"/>
      <column name="break_end" type="time"/>
      <column name="location_id" type="uuid"/>
      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addCheckConstraint tableName="staff_schedules"
      constraintName="chk_schedule_time_range"
      checkConstraint="start_time < end_time"/>

    <addCheckConstraint tableName="staff_schedules"
      constraintName="chk_schedule_break_range"
      checkConstraint="(break_start IS NULL AND break_end IS NULL)
        OR (break_start IS NOT NULL AND break_end IS NOT NULL AND break_start < break_end)"/>

    <addCheckConstraint tableName="staff_schedules"
      constraintName="chk_schedule_break_within_shift"
      checkConstraint="(break_start IS NULL AND break_end IS NULL)
        OR (break_start >= start_time AND break_end <= end_time)"/>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_staff_schedules_staff_date
      ON staff_schedules(staff_id, work_date);
    </sql>

    <createIndex indexName="idx_staff_schedules_work_date" tableName="staff_schedules">
      <column name="work_date"/>
    </createIndex>
  </changeSet>

  <changeSet id="sch-020-slots" author="system">
    <createTable tableName="slots">
      <column name="slot_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="staff_id" type="uuid">
        <constraints nullable="false"/>
      </column>
      <column name="work_date" type="date">
        <constraints nullable="false"/>
      </column>
      <column name="start_time" type="time">
        <constraints nullable="false"/>
      </column>
      <column name="end_time" type="time">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)" defaultValue="AVAILABLE">
        <constraints nullable="false"/>
      </column>
      <column name="held_until" type="timestamptz"/>
      <column name="booking_id" type="uuid"/>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addCheckConstraint tableName="slots"
      constraintName="chk_slot_time_range"
      checkConstraint="start_time < end_time"/>

    <addCheckConstraint tableName="slots"
      constraintName="chk_slot_status"
      checkConstraint="status IN ('AVAILABLE','HELD','BOOKED')"/>

    <addCheckConstraint tableName="slots"
      constraintName="chk_slot_hold_consistency"
      checkConstraint="(status = 'HELD' AND held_until IS NOT NULL)
        OR (status <> 'HELD' AND held_until IS NULL)"/>

    <addCheckConstraint tableName="slots"
      constraintName="chk_slot_booking_consistency"
      checkConstraint="(status = 'BOOKED' AND booking_id IS NOT NULL)
        OR (status <> 'BOOKED' AND booking_id IS NULL)"/>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_slots_staff_date_time
      ON slots(staff_id, work_date, start_time, end_time);
    </sql>

    <createIndex indexName="idx_slots_staff_date" tableName="slots">
      <column name="staff_id"/>
      <column name="work_date"/>
    </createIndex>

    <createIndex indexName="idx_slots_status" tableName="slots">
      <column name="status"/>
    </createIndex>

    <createIndex indexName="idx_slots_booking_id" tableName="slots">
      <column name="booking_id"/>
    </createIndex>

    <sql>
      CREATE INDEX IF NOT EXISTS idx_slots_held_until
      ON slots(held_until)
      WHERE status = 'HELD';
    </sql>
  </changeSet>

  <changeSet id="sch-030-slot-holds" author="system">
    <createTable tableName="slot_holds">
      <column name="hold_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="slot_id" type="uuid">
        <constraints nullable="false"/>
      </column>
      <column name="booking_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="held_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="expires_at" type="timestamptz">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)" defaultValue="ACTIVE">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addForeignKeyConstraint
      constraintName="fk_slot_holds_slot"
      baseTableName="slot_holds" baseColumnNames="slot_id"
      referencedTableName="slots" referencedColumnNames="slot_id"
      onDelete="CASCADE"/>

    <addCheckConstraint tableName="slot_holds"
      constraintName="chk_slot_hold_status"
      checkConstraint="status IN ('ACTIVE','EXPIRED','RELEASED')"/>

    <addCheckConstraint tableName="slot_holds"
      constraintName="chk_slot_hold_expiry"
      checkConstraint="expires_at > held_at"/>

    <createIndex indexName="idx_slot_holds_slot_status" tableName="slot_holds">
      <column name="slot_id"/>
      <column name="status"/>
    </createIndex>

    <createIndex indexName="idx_slot_holds_expires_at" tableName="slot_holds">
      <column name="expires_at"/>
    </createIndex>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_slot_holds_slot_active
      ON slot_holds(slot_id)
      WHERE status = 'ACTIVE';
    </sql>
  </changeSet>

</databaseChangeLog>
```

---

# 4) BOOKING SERVICE (`booking_db`)

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="bok-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="bok-010-bookings" author="system">
    <createTable tableName="bookings">
      <column name="booking_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="user_id" type="uuid">
        <constraints nullable="false"/>
      </column>
      <column name="spa_service_id" type="uuid">
        <constraints nullable="false"/>
      </column>
      <column name="slot_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)">
        <constraints nullable="false"/>
      </column>

      <column name="total_price" type="numeric(12,2)">
        <constraints nullable="false"/>
      </column>
      <column name="currency" type="varchar(3)" defaultValue="VND">
        <constraints nullable="false"/>
      </column>

      <column name="note" type="text"/>

      <column name="booking_time" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>

      <column name="cancelled_at" type="timestamptz"/>
      <column name="cancellation_reason" type="varchar(255)"/>
    </createTable>

    <addCheckConstraint tableName="bookings"
      constraintName="chk_booking_status"
      checkConstraint="status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED')"/>

    <addCheckConstraint tableName="bookings"
      constraintName="chk_total_price_positive"
      checkConstraint="total_price > 0"/>

    <addCheckConstraint tableName="bookings"
      constraintName="chk_currency_len"
      checkConstraint="char_length(currency) = 3"/>

    <createIndex indexName="idx_bookings_user_id" tableName="bookings">
      <column name="user_id"/>
    </createIndex>

    <createIndex indexName="idx_bookings_slot_id" tableName="bookings">
      <column name="slot_id"/>
    </createIndex>

    <createIndex indexName="idx_bookings_status" tableName="bookings">
      <column name="status"/>
    </createIndex>

    <createIndex indexName="idx_bookings_created_at" tableName="bookings">
      <column name="created_at"/>
    </createIndex>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_bookings_slot_active
      ON bookings(slot_id)
      WHERE status IN ('PENDING','CONFIRMED');
    </sql>
  </changeSet>

  <changeSet id="bok-020-booking-status-history" author="system">
    <createTable tableName="booking_status_history">
      <column name="id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="booking_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="from_status" type="varchar(20)"/>
      <column name="to_status" type="varchar(20)">
        <constraints nullable="false"/>
      </column>

      <column name="changed_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>

      <column name="changed_by" type="uuid"/>
      <column name="reason" type="varchar(255)"/>
    </createTable>

    <addForeignKeyConstraint
      constraintName="fk_bsh_booking"
      baseTableName="booking_status_history" baseColumnNames="booking_id"
      referencedTableName="bookings" referencedColumnNames="booking_id"
      onDelete="CASCADE"/>

    <addCheckConstraint tableName="booking_status_history"
      constraintName="chk_bsh_to_status"
      checkConstraint="to_status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED')"/>

    <addCheckConstraint tableName="booking_status_history"
      constraintName="chk_bsh_from_status"
      checkConstraint="from_status IS NULL OR from_status IN ('PENDING','CONFIRMED','CANCELLED','REJECTED')"/>

    <createIndex indexName="idx_bsh_booking_id" tableName="booking_status_history">
      <column name="booking_id"/>
    </createIndex>

    <createIndex indexName="idx_bsh_changed_at" tableName="booking_status_history">
      <column name="changed_at"/>
    </createIndex>
  </changeSet>

  <changeSet id="bok-030-booking-outbox" author="system">
    <createTable tableName="booking_outbox">
      <column name="event_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="aggregate_type" type="varchar(50)" defaultValue="Booking">
        <constraints nullable="false"/>
      </column>

      <column name="aggregate_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="event_type" type="varchar(100)">
        <constraints nullable="false"/>
      </column>

      <column name="payload" type="jsonb">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)" defaultValue="NEW">
        <constraints nullable="false"/>
      </column>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
      <column name="sent_at" type="timestamptz"/>
    </createTable>

    <addForeignKeyConstraint
      constraintName="fk_outbox_booking"
      baseTableName="booking_outbox" baseColumnNames="aggregate_id"
      referencedTableName="bookings" referencedColumnNames="booking_id"
      onDelete="CASCADE"/>

    <addCheckConstraint tableName="booking_outbox"
      constraintName="chk_outbox_status"
      checkConstraint="status IN ('NEW','PROCESSING','SENT','FAILED')"/>

    <createIndex indexName="idx_outbox_status_created" tableName="booking_outbox">
      <column name="status"/>
      <column name="created_at"/>
    </createIndex>

    <createIndex indexName="idx_outbox_event_type" tableName="booking_outbox">
      <column name="event_type"/>
    </createIndex>
  </changeSet>

</databaseChangeLog>
```

---

# 5) PAYMENT SERVICE (`payment_db`)

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="pay-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="pay-010-payment-transactions" author="system">
    <createTable tableName="payment_transactions">
      <column name="payment_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="booking_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="amount" type="numeric(12,2)">
        <constraints nullable="false"/>
      </column>

      <column name="currency" type="varchar(3)" defaultValue="VND">
        <constraints nullable="false"/>
      </column>

      <column name="provider" type="varchar(50)" defaultValue="MOCK">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)" defaultValue="INIT">
        <constraints nullable="false"/>
      </column>

      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>

      <column name="updated_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addCheckConstraint tableName="payment_transactions"
      constraintName="chk_payment_amount_positive"
      checkConstraint="amount > 0"/>

    <addCheckConstraint tableName="payment_transactions"
      constraintName="chk_payment_currency_len"
      checkConstraint="char_length(currency) = 3"/>

    <addCheckConstraint tableName="payment_transactions"
      constraintName="chk_payment_status"
      checkConstraint="status IN ('INIT','SUCCESS','FAILED')"/>

    <createIndex indexName="idx_payment_booking_id" tableName="payment_transactions">
      <column name="booking_id"/>
    </createIndex>

    <createIndex indexName="idx_payment_status" tableName="payment_transactions">
      <column name="status"/>
    </createIndex>

    <sql>
      CREATE UNIQUE INDEX IF NOT EXISTS ux_payment_one_success_per_booking
      ON payment_transactions(booking_id)
      WHERE status = 'SUCCESS';
    </sql>
  </changeSet>

</databaseChangeLog>
```

---

# 6) NOTIFICATION SERVICE (`notification_db`) — optional

## `changes/V1__init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

  <changeSet id="noti-001-ext-uuid" author="system">
    <sql>CREATE EXTENSION IF NOT EXISTS "uuid-ossp";</sql>
  </changeSet>

  <changeSet id="noti-010-notification-logs" author="system">
    <createTable tableName="notification_logs">
      <column name="notification_id" type="uuid" defaultValueComputed="uuid_generate_v4()">
        <constraints primaryKey="true" nullable="false"/>
      </column>

      <column name="user_id" type="uuid">
        <constraints nullable="false"/>
      </column>

      <column name="channel" type="varchar(20)">
        <constraints nullable="false"/>
      </column>

      <column name="template" type="varchar(50)">
        <constraints nullable="false"/>
      </column>

      <column name="payload" type="jsonb">
        <constraints nullable="false"/>
      </column>

      <column name="status" type="varchar(20)" defaultValue="SENT">
        <constraints nullable="false"/>
      </column>

      <column name="sent_at" type="timestamptz"/>
      <column name="created_at" type="timestamptz" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <addCheckConstraint tableName="notification_logs"
      constraintName="chk_notification_channel"
      checkConstraint="channel IN ('EMAIL','SMS','PUSH')"/>

    <addCheckConstraint tableName="notification_logs"
      constraintName="chk_notification_status"
      checkConstraint="status IN ('SENT','FAILED')"/>

    <createIndex indexName="idx_notification_user_id" tableName="notification_logs">
      <column name="user_id"/>
    </createIndex>

    <createIndex indexName="idx_notification_created_at" tableName="notification_logs">
      <column name="created_at"/>
    </createIndex>

    <createIndex indexName="idx_notification_status" tableName="notification_logs">
      <column name="status"/>
    </createIndex>
  </changeSet>

</databaseChangeLog>
```

---

# 7) Gợi ý “master” cho các service còn lại

Bạn chỉ cần copy `db.changelog-master.xml` như user-service, đổi include đúng path.

Ví dụ `db.changelog-master.xml` tối giản:

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
  <include file="db/changelog/changes/V1__init.xml" relativeToChangelogFile="false"/>
</databaseChangeLog>
```

---

## Ghi chú quan trọng (để chạy production “đúng bài”)

* **Không manage Keycloak DB bằng Liquibase của bạn**. Keycloak tự quản schema riêng (Postgres riêng hoặc schema riêng).
* **Partial unique index**: mình dùng `<sql>` vì đây là đúng cách đơn giản và chính xác nhất cho PostgreSQL.
* Nếu bạn triển khai event-driven, có thể thêm `*_outbox` cho payment/notification tương tự booking.

---

Nếu bạn muốn, mình có thể tiếp tục “đóng gói đúng multi-repo” bằng cách:

* tạo **template repo layout** + commit-ready content cho từng service (`src/main/resources/db/...`)
* hoặc viết sẵn `Docker Compose + init` để dựng Postgres từng DB và chạy Liquibase khi service start.
