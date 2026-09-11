# 02 — Database Schema

One Microsoft SQL Server database, **schema-per-module**, every table carries `tenant_id`. EF Core migrations create schemas: `core`, `school`, `rent`, `transport`.

## 1. Conventions (Microsoft SQL Server)

- PK: `BIGINT IDENTITY(1,1) id`; tenant joins via indexed `tenant_id`.
- Money `NUMERIC(15,2)`; timestamps `DATETIME2` (UTC, `SYSUTCDATETIME()`); enums via `NVARCHAR + CHECK`.
- Unicode text: `NVARCHAR` everywhere (Bangladesh-first market — Bengali/Arabic names).
- JSON stored as `NVARCHAR(MAX)` (no native JSON type); enforce shape on key columns with `CHECK (ISJSON(<col>) = 1)`.
- `external_ref NVARCHAR(MAX) DEFAULT NULL` (JSON) for loose cross-module references — **never hard FKs between modules** (Core→module FKs allowed: users/tenants owned by Core).

```sql
-- bootstrap (run once) — SQL Server supports schemas natively
IF SCHEMA_ID('core')      IS NULL EXEC('CREATE SCHEMA core');
IF SCHEMA_ID('school')    IS NULL EXEC('CREATE SCHEMA school');
IF SCHEMA_ID('rent')      IS NULL EXEC('CREATE SCHEMA rent');
IF SCHEMA_ID('transport') IS NULL EXEC('CREATE SCHEMA transport');
```

## 2. Core

```sql
CREATE TABLE core.tenants (
  id BIGINT IDENTITY(1,1) PRIMARY KEY,
  uuid UNIQUEIDENTIFIER NOT NULL UNIQUE DEFAULT NEWID(),
  name NVARCHAR(160) NOT NULL,
  slug NVARCHAR(120) NOT NULL UNIQUE,
  email NVARCHAR(180), phone NVARCHAR(40),
  address NVARCHAR(MAX), logo_url NVARCHAR(MAX),
  status NVARCHAR(20) NOT NULL DEFAULT 'trial'   -- trial|active|suspended|expired
  CHECK (status IN ('trial','active','suspended','expired')),
  subscribed_until DATE,
  created_at DATETIME2 DEFAULT SYSUTCDATETIME(), updated_at DATETIME2 DEFAULT SYSUTCDATETIME()
);

CREATE TABLE core.users (
  id BIGINT IDENTITY(1,1) PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,
  name NVARCHAR(160) NOT NULL, email NVARCHAR(180) NOT NULL,
  phone NVARCHAR(40), password_hash NVARCHAR(MAX) NOT NULL,
  user_type NVARCHAR(40) NOT NULL DEFAULT 'member',  -- owner|admin|member|<module-role>
  status NVARCHAR(20) NOT NULL DEFAULT 'active',
  last_login_at DATETIME2,
  created_at DATETIME2 DEFAULT SYSUTCDATETIME(), updated_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  UNIQUE (tenant_id, email)
);
```

### Entitlements & plans
```sql
CREATE TABLE core.plans (
  id BIGINT IDENTITY(1,1) PRIMARY KEY,
  name NVARCHAR(60) NOT NULL UNIQUE,          -- free|growth|pro
  price_monthly NUMERIC(15,2) NOT NULL DEFAULT 0,
  modules NVARCHAR(MAX) NOT NULL CHECK (ISJSON(modules)=1),       -- ["school","rent","transport"] allowed
  feature_limits NVARCHAR(MAX) NOT NULL DEFAULT '{}' CHECK (ISJSON(feature_limits)=1), -- e.g. {transport_vehicles: 5, rent_properties: 20}
  per_module_price NUMERIC(15,2) NOT NULL DEFAULT 0
);

CREATE TABLE core.entitlements (
  tenant_id BIGINT PRIMARY KEY REFERENCES core.tenants(id) ON DELETE CASCADE,
  modules NVARCHAR(MAX) NOT NULL DEFAULT '[]' CHECK (ISJSON(modules)=1), -- ['school','transport']
  plan NVARCHAR(60) NOT NULL REFERENCES core.plans(name),
  status NVARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active','pending','suspended')),
  subscribed_until DATE,
  created_at DATETIME2 DEFAULT SYSUTCDATETIME(), updated_at DATETIME2 DEFAULT SYSUTCDATETIME()
);

CREATE TABLE core.invoices (
  id BIGINT IDENTITY(1,1) PRIMARY KEY,
  invoice_no NVARCHAR(40) UNIQUE NOT NULL,
  tenant_id BIGINT NOT NULL REFERENCES core.tenants(id) ON DELETE CASCADE,
  plan NVARCHAR(60) NOT NULL,
  period_start DATE NOT NULL, period_end DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, currency NVARCHAR(3) NOT NULL DEFAULT 'USD',
  status NVARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','open','paid','void','past_due')),
  gateway NVARCHAR(40), txn_id NVARCHAR(120), paid_at DATETIME2,
  created_at DATETIME2 DEFAULT SYSUTCDATETIME(), updated_at DATETIME2 DEFAULT SYSUTCDATETIME()
);

CREATE TABLE core.payment_gateways (          -- platform's own billing creds
  id BIGINT IDENTITY(1,1) PRIMARY KEY, key NVARCHAR(60) NOT NULL UNIQUE, label NVARCHAR(120),
  is_default BIT DEFAULT 1, is_test BIT DEFAULT 1, config NVARCHAR(MAX) CHECK (ISJSON(config)=1)
);

CREATE TABLE core.role_permissions (          -- RBAC (Core-side)
  id BIGINT IDENTITY(1,1) PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,  -- NULL = platform role
  name NVARCHAR(60) NOT NULL,                   -- platform_admin | owner | admin | teacher | accountant…
  permissions NVARCHAR(MAX) NOT NULL DEFAULT '[]' CHECK (ISJSON(permissions)=1), -- ["school.students.view", … module.*]
  UNIQUE (tenant_id, name)
);
CREATE TABLE core.user_roles ( user_id BIGINT REFERENCES core.users(id) ON DELETE CASCADE,
  role_id BIGINT REFERENCES core.role_permissions(id) ON DELETE CASCADE, PRIMARY KEY(user_id, role_id));
```

### Notifications, audit, settings, assets
```sql
CREATE TABLE core.notification_templates ( id BIGINT IDENTITY(1,1) PRIMARY KEY,
  key NVARCHAR(80) UNIQUE NOT NULL, channel NVARCHAR(20) NOT NULL,   -- email|sms|push
  subject NVARCHAR(200), body NVARCHAR(MAX) NOT NULL, is_active BIT DEFAULT 1 );

CREATE TABLE core.notifications ( id BIGINT IDENTITY(1,1) PRIMARY KEY,
  tenant_id BIGINT REFERENCES core.tenants(id) ON DELETE CASCADE,
  channel NVARCHAR(20) NOT NULL, template_key NVARCHAR(80),
  recipient NVARCHAR(180) NOT NULL, payload NVARCHAR(MAX) CHECK (ISJSON(payload)=1),
  status NVARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','sent','failed','dropped')),
  attempts SMALLINT DEFAULT 0, sent_at DATETIME2, error NVARCHAR(MAX), created_at DATETIME2 DEFAULT SYSUTCDATETIME() );

CREATE TABLE core.audit_logs ( id BIGINT IDENTITY(1,1) PRIMARY KEY,
  tenant_id BIGINT, user_id BIGINT, module NVARCHAR(40) NOT NULL,
  action NVARCHAR(40) NOT NULL, entity_type NVARCHAR(80), entity_id NVARCHAR(80),
  detail NVARCHAR(MAX) CHECK (ISJSON(detail)=1), ip NVARCHAR(64), created_at DATETIME2 DEFAULT SYSUTCDATETIME() );

CREATE TABLE core.event_outbox ( id BIGINT IDENTITY(1,1) PRIMARY KEY,
  tenant_id BIGINT NOT NULL, module NVARCHAR(40) NOT NULL,
  event_type NVARCHAR(120) NOT NULL, event_key NVARCHAR(200) UNIQUE,
  payload NVARCHAR(MAX) NOT NULL CHECK (ISJSON(payload)=1),
  status NVARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending','dispatched','failed')),
  attempts SMALLINT DEFAULT 0, dispatched_at DATETIME2, created_at DATETIME2 DEFAULT SYSUTCDATETIME() );

CREATE TABLE core.settings ( tenant_id BIGINT PRIMARY KEY REFERENCES core.tenants(id) ON DELETE CASCADE,
  values NVARCHAR(MAX) NOT NULL DEFAULT '{}' CHECK (ISJSON(values)=1) );
CREATE TABLE core.system_settings ( key NVARCHAR(120) PRIMARY KEY, value NVARCHAR(MAX) CHECK (ISJSON(value)=1) );
CREATE TABLE core.media ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT,
  module NVARCHAR(40), model_type NVARCHAR(80), model_id NVARCHAR(80),
  path NVARCHAR(MAX) NOT NULL, mime NVARCHAR(100), size_bytes BIGINT, created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE core.sessions ( id NVARCHAR(255) PRIMARY KEY, user_id BIGINT,
  ip NVARCHAR(64), user_agent NVARCHAR(512), payload NVARCHAR(MAX),
  last_activity DATETIME2, tenant_id BIGINT);
```

## 3. School module (`school`)

```sql
CREATE TABLE school.classes ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name NVARCHAR(120) NOT NULL, code NVARCHAR(40), academic_year NVARCHAR(20) NOT NULL,
  capacity SMALLINT DEFAULT 0, created_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  UNIQUE (tenant_id, academic_year, name) );
CREATE TABLE school.sections ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT REFERENCES school.classes(id) ON DELETE CASCADE,
  name NVARCHAR(80) NOT NULL, room NVARCHAR(60),
  UNIQUE (tenant_id, class_id, name) );
CREATE TABLE school.subjects ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT REFERENCES school.classes(id) ON DELETE CASCADE,
  name NVARCHAR(120) NOT NULL, code NVARCHAR(40), units SMALLINT DEFAULT 1 );
CREATE TABLE school.students ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, class_id BIGINT, section_id BIGINT,
  admission_no NVARCHAR(60) UNIQUE, roll SMALLINT, name NVARCHAR(160) NOT NULL,
  dob DATE, gender NVARCHAR(20), email NVARCHAR(180), phone NVARCHAR(40),
  guardian_name NVARCHAR(160), guardian_phone NVARCHAR(40), address NVARCHAR(MAX),
  status NVARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','inactive','passed_out')),
  external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1), created_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  UNIQUE (tenant_id, admission_no) );
CREATE TABLE school.teachers ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name NVARCHAR(160) NOT NULL, phone NVARCHAR(40), email NVARCHAR(180),
  designation NVARCHAR(120), departments NVARCHAR(MAX) DEFAULT '[]' CHECK (ISJSON(departments)=1),
  subjects NVARCHAR(MAX) DEFAULT '[]' CHECK (ISJSON(subjects)=1),
  hire_date DATE, external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1) );
CREATE TABLE school.attendance ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT REFERENCES school.students(id) ON DELETE CASCADE,
  class_id BIGINT, section_id BIGINT, date DATE NOT NULL,
  status NVARCHAR(20) NOT NULL CHECK (status IN ('present','absent','late','leave')),
  marked_by BIGINT, note NVARCHAR(255), created_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  UNIQUE (tenant_id, student_id, date) );
CREATE TABLE school.fee_structures ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  class_id BIGINT, section_id BIGINT, name NVARCHAR(120) NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  frequency NVARCHAR(20) DEFAULT 'monthly' CHECK (frequency IN ('monthly','term','yearly','one_time')) );
CREATE TABLE school.fee_charges ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT REFERENCES school.students(id) ON DELETE CASCADE,
  fee_structure_id BIGINT, period NVARCHAR(40) NOT NULL, due_date DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, fine NUMERIC(15,2) DEFAULT 0,
  status NVARCHAR(20) DEFAULT 'due' CHECK (status IN ('due','partial','paid','waived')),
  paid_amount NUMERIC(15,2) DEFAULT 0, paid_at DATETIME2 );
CREATE TABLE school.fee_payments ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  student_id BIGINT, invoice_no NVARCHAR(60) UNIQUE, amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  method NVARCHAR(40), gateway NVARCHAR(40), txn_id NVARCHAR(120), receipt_no NVARCHAR(60),
  paid_at DATETIME2 DEFAULT SYSUTCDATETIME(), note NVARCHAR(MAX), external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1) );
CREATE TABLE school.exams ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name NVARCHAR(120) NOT NULL, academic_year NVARCHAR(20), class_ids NVARCHAR(MAX) CHECK (ISJSON(class_ids)=1),
  status NVARCHAR(20) DEFAULT 'draft', result_goal NVARCHAR(20) DEFAULT 'pass' );
CREATE TABLE school.results ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  exam_id BIGINT REFERENCES school.exams(id) ON DELETE CASCADE,
  student_id BIGINT, subject_id BIGINT,
  marks NUMERIC(8,2), grade NVARCHAR(10), point NUMERIC(4,2), obtained_total NUMERIC(10,2),
  UNIQUE (tenant_id, exam_id, student_id, subject_id) );
```

## 4. Rent module (`rent`)

```sql
CREATE TABLE rent.properties ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name NVARCHAR(160) NOT NULL, address NVARCHAR(MAX), type NVARCHAR(40),
  units_count SMALLINT DEFAULT 0, total_monthly_rent NUMERIC(15,2) DEFAULT 0,
  status NVARCHAR(20) DEFAULT 'active', created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE rent.units ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  property_id BIGINT REFERENCES rent.properties(id) ON DELETE CASCADE,
  unit_no NVARCHAR(60), floor SMALLINT, size NVARCHAR(40),
  rent_amount NUMERIC(15,2) NOT NULL DEFAULT 0, deposit_amount NUMERIC(15,2) DEFAULT 0,
  status NVARCHAR(20) DEFAULT 'vacant' CHECK (status IN ('vacant','occupied','maintenance')),
  external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1), created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE rent.renters ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name NVARCHAR(160) NOT NULL, phone NVARCHAR(40), email NVARCHAR(180),
  national_id NVARCHAR(60), emergency_phone NVARCHAR(40), address NVARCHAR(MAX), notes NVARCHAR(MAX),
  created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE rent.leases ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  unit_id BIGINT REFERENCES rent.units(id), renter_id BIGINT REFERENCES rent.renters(id),
  start_date DATE NOT NULL, end_date DATE, notice_days SMALLINT DEFAULT 30,
  rent_amount NUMERIC(15,2) NOT NULL DEFAULT 0, deposit NUMERIC(15,2) DEFAULT 0,
  status NVARCHAR(20) DEFAULT 'active' CHECK (status IN ('draft','active','expired','terminated','renewed')),
  signed_at DATETIME2, notes NVARCHAR(MAX), external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1),
  created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE rent.rent_payments ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  lease_id BIGINT REFERENCES rent.leases(id), period_start DATE NOT NULL, period_end DATE NOT NULL,
  amount NUMERIC(15,2) NOT NULL DEFAULT 0, late_fee NUMERIC(15,2) DEFAULT 0,
  method NVARCHAR(40), gateway NVARCHAR(40), txn_id NVARCHAR(120), paid_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  receipt_no NVARCHAR(60), notes NVARCHAR(MAX), external_ref NVARCHAR(MAX) CHECK (ISJSON(external_ref)=1) );
CREATE TABLE rent.lease_invoices ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  lease_id BIGINT, invoice_no NVARCHAR(60) UNIQUE NOT NULL, period_start DATE NOT NULL,
  period_end DATE NOT NULL, amount NUMERIC(15,2) DEFAULT 0, due_date DATE NOT NULL,
  fine NUMERIC(15,2) DEFAULT 0, status NVARCHAR(20) DEFAULT 'open'
    CHECK (status IN ('open','paid','overdue','void')) );
CREATE TABLE rent.maintenance_requests ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  unit_id BIGINT, renter_id BIGINT, title NVARCHAR(160) NOT NULL, description NVARCHAR(MAX),
  status NVARCHAR(20) DEFAULT 'open' CHECK (status IN ('open','in_progress','done','cancelled')),
  priority NVARCHAR(20) DEFAULT 'medium', cost NUMERIC(15,2) DEFAULT 0,
  charged_to NVARCHAR(20) DEFAULT 'owner' CHECK (charged_to IN ('owner','renter')),
  completed_at DATETIME2, created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
```

## 5. Transport module (`transport`)

```sql
CREATE TABLE transport.vehicles ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  plate NVARCHAR(40) NOT NULL, name NVARCHAR(120), make_model NVARCHAR(120), year SMALLINT,
  seats SMALLINT DEFAULT 0, fuel NVARCHAR(20),
  status NVARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','maintenance','retired')),
  insurance_date DATE, reg_expiry DATE, created_at DATETIME2 DEFAULT SYSUTCDATETIME(),
  UNIQUE (tenant_id, plate) );
CREATE TABLE transport.drivers ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  user_id BIGINT, name NVARCHAR(160) NOT NULL, phone NVARCHAR(40), email NVARCHAR(180),
  license_no NVARCHAR(60), assigned_vehicle_id BIGINT REFERENCES transport.vehicles(id),
  status NVARCHAR(20) DEFAULT 'active',
  UNIQUE (tenant_id, license_no) );
CREATE TABLE transport.routes ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  name NVARCHAR(120) NOT NULL, distance_km NUMERIC(8,2), eta_min SMALLINT,
  status NVARCHAR(20) DEFAULT 'active',
  UNIQUE (tenant_id, name) );
CREATE TABLE transport.stops ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  route_id BIGINT REFERENCES transport.routes(id) ON DELETE CASCADE,
  stop_no SMALLINT NOT NULL, name NVARCHAR(160) NOT NULL, lat NUMERIC(9,6), lng NUMERIC(9,6),
  UNIQUE (tenant_id, route_id, stop_no) );
CREATE TABLE transport.trips ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  route_id BIGINT, vehicle_id BIGINT, driver_id BIGINT,
  trip_date DATE NOT NULL, start_time TIME, end_time TIME,
  status NVARCHAR(20) DEFAULT 'scheduled'
    CHECK (status IN ('scheduled','in_progress','completed','cancelled')),
  boardings SMALLINT DEFAULT 0, created_at DATETIME2 DEFAULT SYSUTCDATETIME() );
CREATE TABLE transport.subscribers ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  external_ref NVARCHAR(MAX) NOT NULL CHECK (ISJSON(external_ref)=1),  -- {module:'school', type:'student', id: 123}  OR null (adhoc)
  member_name NVARCHAR(160) NOT NULL, contact NVARCHAR(40),
  route_id BIGINT REFERENCES transport.routes(id), stop_id BIGINT,
  fee_amount NUMERIC(15,2) DEFAULT 0, period NVARCHAR(40),
  status NVARCHAR(20) DEFAULT 'active' CHECK (status IN ('active','inactive')),
  UNIQUE (tenant_id, external_ref, period) );
CREATE TABLE transport.trip_attendance ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  trip_id BIGINT REFERENCES transport.trips(id) ON DELETE CASCADE,
  subscriber_id BIGINT, boarded BIT, boarded_at DATETIME2,
  UNIQUE (tenant_id, trip_id, subscriber_id) );
CREATE TABLE transport.fuel_logs ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  vehicle_id BIGINT, date DATE NOT NULL, liters NUMERIC(10,2), cost NUMERIC(15,2), odometer INT );
CREATE TABLE transport.vehicle_maintenance ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  vehicle_id BIGINT, date DATE NOT NULL, type NVARCHAR(60), cost NUMERIC(15,2), notes NVARCHAR(MAX),
  next_due DATE );
CREATE TABLE transport.collections ( id BIGINT IDENTITY(1,1) PRIMARY KEY, tenant_id BIGINT NOT NULL,
  subscriber_id BIGINT, period NVARCHAR(40), amount NUMERIC(15,2) NOT NULL DEFAULT 0,
  method NVARCHAR(40), paid_at DATETIME2 DEFAULT SYSUTCDATETIME(), receipt_no NVARCHAR(60) );
```

## 6. Indexes (mandatory)

```sql
CREATE NONCLUSTERED INDEX ix_users_tenant ON core.users(tenant_id);
CREATE NONCLUSTERED INDEX ix_students_tenant ON school.students(tenant_id);
CREATE NONCLUSTERED INDEX ix_students_class_section ON school.students(class_id, section_id);
CREATE NONCLUSTERED INDEX ix_attendance_student_date ON school.attendance(student_id, date);
CREATE NONCLUSTERED INDEX ix_fee_charges_student_status ON school.fee_charges(student_id, status);
CREATE NONCLUSTERED INDEX ix_fee_charges_tenant_due ON school.fee_charges(tenant_id, due_date);
CREATE NONCLUSTERED INDEX ix_results_exam ON school.results(exam_id);
CREATE NONCLUSTERED INDEX ix_units_property ON rent.units(property_id);
CREATE NONCLUSTERED INDEX ix_units_status ON rent.units(status);
CREATE NONCLUSTERED INDEX ix_leases_unit_status ON rent.leases(unit_id, status);
CREATE NONCLUSTERED INDEX ix_lease_invoices_lease ON rent.lease_invoices(lease_id);
CREATE NONCLUSTERED INDEX ix_rent_payments_lease ON rent.rent_payments(lease_id);
CREATE NONCLUSTERED INDEX ix_subscribers_route ON transport.subscribers(route_id);
CREATE NONCLUSTERED INDEX ix_stops_route ON transport.stops(route_id);
CREATE NONCLUSTERED INDEX ix_trips_date ON transport.trips(trip_date);
CREATE NONCLUSTERED INDEX ix_trip_attendance_trip ON transport.trip_attendance(trip_id);
CREATE NONCLUSTERED INDEX ix_event_outbox_status ON core.event_outbox(status);
-- Every module table's tenant_id pair is covered by the PK/unique patterns above (or the explicit indexes).
```

## 7. Seeded data

- `core.plans`: free(0, one module trial) / growth(per-module pricing) / pro(all 3 + extras) — see 08.
- `core.role_permissions`: platform_admin; tenant owner/admin; per-module roles.
- `core.notification_templates`: invoice_created, payment_received, lease_due, attendance_summary, trip_reminder.
- `core.system_settings`: default currency, time zone, billing provider.