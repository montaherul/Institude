# 02 — Database Schema

One PostgreSQL database, **schema-per-module**, every table carries `tenant_id`. Migrations create schemas: `core`, `school`, `rent`, `transport`.

## 1. Conventions

- PK: `BIGSERIAL id` or `BIGINT`; tenant joins via indexed `tenant_id`.
- Money `NUMERIC(15,2)`; timestamps `timestamptz`; enums via `VARCHAR + CHECK` (portable).
- `external_ref JSONB DEFAULT NULL` for loose cross-module references — **never hard FKs between modules** (Core→module FKs allowed: users/tenants owned by Core).

```sql
-- bootstrap
DO $$ BEGIN CREATE SCHEMA IF NOT EXISTS core; CREATE SCHEMA IF NOT EXISTS school;
CREATE SCHEMA IF NOT EXISTS rent; CREATE SCHEMA IF NOT EXISTS transport; END $$;
```

## 2. Core

```sql
CREATE TABLE core.tenants (
  id BIGSERIAL PRIMARY KEY,
  uuid UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
  name VARCHAR(160) NOT NULL,
  slug VARCHAR(120) NOT NULL UNIQUE,
  email VARCHAR(180), phone VARCHAR(40),
  address TEXT, logo_url TEXT,
  status VARCHAR(20) NOT NULL DEFAULT 'trial'   -- trial|active|suspended|expired
  CHECK (status IN ('trial','active','suspended','expired')),
  subscribed_until DATE,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now()
);

CREATE TABLE core.users (
  id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,
  name VARCHAR(160) NOT NULL, email VARCHAR(180) NOT NULL,
  phone VARCHAR(40), password_hash TEXT NOT NULL,
  user_type VARCHAR(40) NOT NULL DEFAULT 'member',  -- owner|admin|member|<module-role>
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  last_login_at timestamptz,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, email)
);
```

### Entitlements & plans
```sql
CREATE TABLE core.plans (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(60) NOT NULL UNIQUE,          -- free|growth|pro
  price_monthly NUMERIC(15,2) NOT NULL DEFAULT 0,
  modules JSONB NOT NULL,                    -- ["school","rent","transport"] allowed
  feature_limits JSONB NOT NULL DEFAULT '{}',-- e.g. {transport_vehicles: 5, rent_properties: 20}
  per_module_price NUMERIC(15,2) NOT NULL DEFAULT 0
);

CREATE TABLE core.entitlements (
  tenant_id BIGINT PRIMARY KEY REFERENCES core.tenants(id) ON DELETE CASCADE,
  modules JSONB NOT NULL DEFAULT '[]',       -- ['school','transport']
  plan VARCHAR(60) NOT NULL REFERENCES core.plans(name),
  status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active','pending','suspended')),
  subscribed_until DATE,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now()
);

CREATE TABLE core.invoices (
  id BIGSERIAL PRIMARY KEY,
  invoice_no VARCHAR(40) UNIQUE NOT NULL,
  tenant_id BIGINT NOT NULL REFERENCES core.tenants(id) ON DELETE CASCADE,
  plan VARCHAR(60) NOT NULL,
  period_start DATE NOT NULL, period_end DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','open','paid','void','past_due')),
  gateway VARCHAR(40), txn_id VARCHAR(120), paid_at timestamptz,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now()
);

CREATE TABLE core.payment_gateways (          -- platform's own billing creds
  id BIGSERIAL PRIMARY KEY, key VARCHAR(60) NOT NULL UNIQUE, label VARCHAR(120),
  is_default BOOLEAN DEFAULT true, is_test BOOLEAN DEFAULT true, config JSONB
);

CREATE TABLE core.role_permissions (          -- RBAC (Core-side)
  id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,  -- NULL = platform role
  name VARCHAR(60) NOT NULL,                   -- platform_admin | owner | admin | teacher | accountant…
  permissions JSONB NOT NULL DEFAULT '[]',     -- ["school.students.view", … module.*]
  UNIQUE (tenant_id, name)
);
CREATE TABLE core.user_roles ( user_id BIGINT REFERENCES core.users(id) ON DELETE CASCADE,
  role_id BIGINT REFERENCES core.role_permissions(id) ON DELETE CASCADE, PRIMARY KEY(user_id, role_id));
```

### Notifications, audit, settings, assets
```sql
CREATE TABLE core.notification_templates ( id BIGSERIAL PRIMARY KEY,
  key VARCHAR(80) UNIQUE NOT NULL, channel VARCHAR(20) NOT NULL,   -- email|sms|push
  subject VARCHAR(200), body TEXT NOT NULL, is_active BOOLEAN DEFAULT true );

CREATE TABLE core.notifications ( id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,
  channel VARCHAR(20) NOT NULL, template_key VARCHAR(80),
  recipient VARCHAR(180) NOT NULL, payload JSONB,
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','sent','failed','dropped')),
  attempts SMALLINT DEFAULT 0, sent_at timestamptz, error TEXT, created_at timestamptz DEFAULT now() );

CREATE TABLE core.audit_logs ( id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT, user_id BIGINT, module VARCHAR(40) NOT NULL,
  action VARCHAR(40) NOT NULL, entity_type VARCHAR(80), entity_id VARCHAR(80),
  detail JSONB, ip VARCHAR(64), created_at timestamptz DEFAULT now() );

CREATE TABLE core.event_outbox ( id BIGSERIAL PRIMARY KEY,
  tenant_id BIGINT NOT NULL, module VARCHAR(40) NOT NULL,
  event_type VARCHAR(120) NOT NULL, event_key VARCHAR(200) UNIQUE,
  payload JSONB NOT NULL,
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','dispatched','failed')),
  attempts SMALLINT DEFAULT 0, dispatched_at timestamptz, created_at timestamptz DEFAULT now() );

CREATE TABLE core.settings ( tenant_id BIGINT PRIMARY KEY REFERENCES core.tenants(id) ON DELETE CASCADE,
  values JSONB NOT NULL DEFAULT '{}' );
CREATE TABLE core.system_settings ( key VARCHAR(120) PRIMARY KEY, value JSONB );
CREATE TABLE core.media ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT,
  module VARCHAR(40), model_type VARCHAR(80), model_id VARCHAR(80),
  path TEXT NOT NULL, mime VARCHAR(100), size_bytes BIGINT, created_at timestamptz DEFAULT now() );
CREATE TABLE core.sessions ( id VARCHAR(255) PRIMARY KEY, user_id BIGINT, ip, user_agent, payload, last_activity, tenant_id BIGINT);
```

## 3. School module (`school`)

```sql
CREATE TABLE school.classes ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name VARCHAR(120) NOT NULL, code VARCHAR(40), academic_year VARCHAR(20) NOT NULL,
  capacity SMALLINT DEFAULT 0, created_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, academic_year, name) );
CREATE TABLE school.sections ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT REFERENCES school.classes(id) ON DELETE CASCADE,
  name VARCHAR(80) NOT NULL, room VARCHAR(60),
  UNIQUE (tenant_id, class_id, name) );
CREATE TABLE school.subjects ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT REFERENCES school.classes(id) ON DELETE CASCADE,
  name VARCHAR(120) NOT NULL, code VARCHAR(40), units SMALLINT DEFAULT 1 );
CREATE TABLE school.students ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, class_id BIGINT, section_id BIGINT,
  admission_no VARCHAR(60) UNIQUE, roll SMALLINT, name VARCHAR(160) NOT NULL,
  dob DATE, gender VARCHAR(20), email VARCHAR(180), phone VARCHAR(40),
  guardian_name VARCHAR(160), guardian_phone VARCHAR(40), address TEXT,
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','inactive','passed_out')),
  external_ref JSONB, created_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, admission_no) );
CREATE TABLE school.teachers ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name VARCHAR(160) NOT NULL, phone VARCHAR(40), email VARCHAR(180),
  designation VARCHAR(120), departments JSONB DEFAULT '[]', subjects JSONB DEFAULT '[]',
  hire_date DATE, external_ref JSONB );
CREATE TABLE school.attendance ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT REFERENCES school.students(id) ON DELETE CASCADE,
  class_id BIGINT, section_id BIGINT, date DATE NOT NULL,
  status VARCHAR(20) NOT NULL CHECK (status IN ('present','absent','late','leave')),
  marked_by BIGINT, note VARCHAR(255), created_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, student_id, date) );
CREATE TABLE school.fee_structures ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT, section_id BIGINT, name VARCHAR(120) NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  frequency VARCHAR(20) DEFAULT 'monthly' CHECK (frequency IN ('monthly','term','yearly','one_time')) );
CREATE TABLE school.fee_charges ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT REFERENCES school.students(id) ON DELETE CASCADE,
  fee_structure_id BIGINT, period VARCHAR(40) NOT NULL, due_date DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, fine NUMERIC(15,2) DEFAULT 0,
  status VARCHAR(20) DEFAULT 'due' CHECK (status IN ('due','partial','paid','waived')),
  paid_amount NUMERIC(15,2) DEFAULT 0, paid_at timestamptz );
CREATE TABLE school.fee_payments ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT, invoice_no VARCHAR(60) UNIQUE, amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  method VARCHAR(40), gateway VARCHAR(40), txn_id VARCHAR(120), receipt_no VARCHAR(60),
  paid_at timestamptz DEFAULT now(), note TEXT, external_ref JSONB );
CREATE TABLE school.exams ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name VARCHAR(120) NOT NULL, academic_year VARCHAR(20), class_ids JSONB,
  status VARCHAR(20) DEFAULT 'draft', result_goal VARCHAR(20) DEFAULT 'pass' );
CREATE TABLE school.results ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  exam_id BIGINT REFERENCES school.exams(id) ON DELETE CASCADE,
  student_id BIGINT, subject_id BIGINT,
  marks NUMERIC(8,2), grade VARCHAR(10), point NUMERIC(4,2), obtained_total NUMERIC(10,2),
  UNIQUE (tenant_id, exam_id, student_id, subject_id) );
```

## 4. Rent module (`rent`)

```sql
CREATE TABLE rent.properties ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name VARCHAR(160) NOT NULL, address TEXT, type VARCHAR(40),
  units_count SMALLINT DEFAULT 0, total_monthly_rent NUMERIC(15,2) DEFAULT 0,
  status VARCHAR(20) DEFAULT 'active', created_at timestamptz DEFAULT now() );
CREATE TABLE rent.units ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  property_id BIGINT REFERENCES rent.properties(id) ON DELETE CASCADE,
  unit_no VARCHAR(60), floor SMALLINT, size VARCHAR(40),
  rent_amount NUMERIC(15,2) NOT NULL DEFAULT 0, deposit_amount NUMERIC(15,2) DEFAULT 0,
  status VARCHAR(20) DEFAULT 'vacant' CHECK (status IN ('vacant','occupied','maintenance')),
  external_ref JSONB, created_at timestamptz DEFAULT now() );
CREATE TABLE rent.renters ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name VARCHAR(160) NOT NULL, phone VARCHAR(40), email VARCHAR(180),
  national_id VARCHAR(60), emergency_phone VARCHAR(40), address TEXT, notes TEXT,
  created_at timestamptz DEFAULT now() );
CREATE TABLE rent.leases ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  unit_id BIGINT REFERENCES rent.units(id), renter_id BIGINT REFERENCES rent.renters(id),
  start_date DATE NOT NULL, end_date DATE, notice_days SMALLINT DEFAULT 30,
  rent_amount NUMERIC(15,2) NOT NULL DEFAULT 0, deposit NUMERIC(15,2) DEFAULT 0,
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('draft','active','expired','terminated','renewed')),
  signed_at timestamptz, notes TEXT, external_ref JSONB,
  created_at timestamptz DEFAULT now() );
CREATE TABLE rent.rent_payments ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  lease_id BIGINT REFERENCES rent.leases(id), period_start DATE NOT NULL, period_end DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, late_fee NUMERIC(15,2) DEFAULT 0,
  method VARCHAR(40), gateway VARCHAR(40), txn_id VARCHAR(120), paid_at timestamptz DEFAULT now(),
  receipt_no VARCHAR(60), notes TEXT, external_ref JSONB );
CREATE TABLE rent.lease_invoices ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  lease_id BIGINT, invoice_no VARCHAR(60) UNIQUE NOT NULL, period_start DATE NOT NULL,
  period_end DATE NOT NULL, amount NUMERIC(15,2) DEFAULT 0, due_date DATE NOT NULL,
  fine NUMERIC(15,2) DEFAULT 0, status VARCHAR(20) DEFAULT 'open'
    CHECK (status IN ('open','paid','overdue','void')) );
CREATE TABLE rent.maintenance_requests ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  unit_id BIGINT, renter_id BIGINT, title VARCHAR(160) NOT NULL, description TEXT,
  status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open','in_progress','done','cancelled')),
  priority VARCHAR(20) DEFAULT 'medium', cost NUMERIC(15,2) DEFAULT 0,
  charged_to VARCHAR(20) DEFAULT 'owner' CHECK (charged_to IN ('owner','renter')),
  completed_at timestamptz, created_at timestamptz DEFAULT now() );
```

## 5. Transport module (`transport`)

```sql
CREATE TABLE transport.vehicles ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  plate VARCHAR(40) NOT NULL, name VARCHAR(120), make_model VARCHAR(120), year SMALLINT,
  seats SMALLINT DEFAULT 0, fuel VARCHAR(20),
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','maintenance','retired')),
  insurance_date DATE, reg_expiry DATE, created_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, plate) );
CREATE TABLE transport.drivers ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name VARCHAR(160) NOT NULL, phone VARCHAR(40), email VARCHAR(180),
  license_no VARCHAR(60), assigned_vehicle_id BIGINT REFERENCES transport.vehicles(id),
  status VARCHAR(20) DEFAULT 'active',
  UNIQUE (tenant_id, license_no) );
CREATE TABLE transport.routes ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name VARCHAR(120) NOT NULL, distance_km NUMERIC(8,2), eta_min SMALLINT,
  status VARCHAR(20) DEFAULT 'active',
  UNIQUE (tenant_id, name) );
CREATE TABLE transport.stops ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  route_id BIGINT REFERENCES transport.routes(id) ON DELETE CASCADE,
  stop_no SMALLINT NOT NULL, name VARCHAR(160) NOT NULL, lat NUMERIC(9,6), lng NUMERIC(9,6),
  UNIQUE (tenant_id, route_id, stop_no) );
CREATE TABLE transport.trips ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  route_id BIGINT, vehicle_id BIGINT, driver_id BIGINT,
  trip_date DATE NOT NULL, start_time TIME, end_time TIME,
  status VARCHAR(20) DEFAULT 'scheduled'
    CHECK (status IN ('scheduled','in_progress','completed','cancelled')),
  boardings SMALLINT DEFAULT 0, created_at timestamptz DEFAULT now() );
CREATE TABLE transport.subscribers ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  external_ref JSONB NOT NULL,          -- {module:'school', type:'student', id: 123}  OR null (adhoc)
  member_name VARCHAR(160) NOT NULL, contact VARCHAR(40),
  route_id BIGINT REFERENCES transport.routes(id), stop_id BIGINT,
  fee_amount NUMERIC(15,2) DEFAULT 0, period VARCHAR(40),
  status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','inactive')),
  UNIQUE (tenant_id, external_ref, period) );
CREATE TABLE transport.trip_attendance ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  trip_id BIGINT REFERENCES transport.trips(id) ON DELETE CASCADE,
  subscriber_id BIGINT, boarded BOOLEAN, boarded_at timestamptz,
  UNIQUE (tenant_id, trip_id, subscriber_id) );
CREATE TABLE transport.fuel_logs ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  vehicle_id BIGINT, date DATE NOT NULL, liters NUMERIC(10,2), cost NUMERIC(15,2), odometer INT );
CREATE TABLE transport.vehicle_maintenance ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  vehicle_id BIGINT, date DATE NOT NULL, type VARCHAR(60), cost NUMERIC(15,2), notes TEXT,
  next_due DATE );
CREATE TABLE transport.collections ( id BIGSERIAL PRIMARY KEY, tenant_id BIGINT NOT NULL,
  subscriber_id BIGINT, period VARCHAR(40), amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  method VARCHAR(40), paid_at timestamptz DEFAULT now(), receipt_no VARCHAR(60) );
```

## 6. Indexes (mandatory)

```sql
CREATE INDEX IF NOT EXISTS idx_tenant ON core.users(tenant_id);
CREATE INDEX ON school.students(tenant_id);        ON school.students(class_id, section_id);
CREATE INDEX ON school.attendance(student_id, date); ON school.fee_charges(student_id, status);
CREATE INDEX ON school.fee_charges(tenant_id, due_date); ON school.results(exam_id);
CREATE INDEX ON rent.units(property_id);  ON rent.units(status); ON rent.leases(unit_id, status);
CREATE INDEX ON rent.lease_invoices(lease_id); ON rent.rent_payments(lease_id);
CREATE INDEX ON transport.subscribers(route_id); ON transport.stops(route_id);
CREATE INDEX ON transport.trips(trip_date); ON transport.trip_attendance(trip_id);
CREATE INDEX ON core.event_outbox(status);
-- Every module table's tenant_id + id pair should be indexed via its PK/unique patterns above.
```

## 7. Seeded data

- `core.plans`: free(0, one module trial) / growth(per-module pricing) / pro(all 3 + extras) — see 08.
- `core.role_permissions`: platform_admin; tenant owner/admin; per-module roles.
- `core.notification_templates`: invoice_created, payment_received, lease_due, attendance_summary, trip_reminder.
- `core.system_settings`: default currency, time zone, billing provider.