# 02 — Database Schema (SQL Server)

Shared-schema multi-tenant **SQL Server**. Conventions:

- Every tenant-scoped table carries `institute_id BIGINT NOT NULL` (indexed; `InstituteScope` applies filtering in the generic repository).
- `id` = `BIGINT IDENTITY(1,1) PRIMARY KEY` unless shown.
- `created_at`/`updated_at` = `DATETIME2 NULL` (default `SYSUTCDATETIME()`), managed by EF Core on save.
- Money = `DECIMAL(18,2)`; percentages = `DECIMAL(5,2)`; booleans = `BIT`.
- Enums are stored as `NVARCHAR(n)` with a `CHECK` constraint listing the observed UI values. (Compact form below: `CHECK (col IN (...))`.)
- JSON columns are `NVARCHAR(MAX)` with `CHECK (ISJSON(col) = 1)` — shown as `(json)`.
- Text = `NVARCHAR(n)`; `id` references are `BIGINT`.

EF Core maps the PascalCase C# entities (`entities.md`) to these snake_case columns via explicit configuration in `MightySchool.Infrastructure/Configurations`.

**Example DDL (grounding for the shorthand below):**

```sql
CREATE TABLE students (
    id             BIGINT IDENTITY(1,1) PRIMARY KEY,
    institute_id   BIGINT NOT NULL,
    user_id        BIGINT NULL,
    image_path     NVARCHAR(255) NULL,
    roll           INT NOT NULL,
    admission_number NVARCHAR(32) NOT NULL,
    name           NVARCHAR(120) NOT NULL,
    father_name    NVARCHAR(120) NULL,
    mother_name    NVARCHAR(120) NULL,
    gender         NVARCHAR(8)  NOT NULL CONSTRAINT ck_students_gender   CHECK (gender   IN ('male','female','other')),
    dob            DATE NULL,
    class_id       BIGINT NOT NULL,
    section_id     BIGINT NULL,
    group_id       BIGINT NULL,
    category_id    BIGINT NULL,
    phone_no       NVARCHAR(32) NULL,
    g_mobile       NVARCHAR(32) NULL,
    email          NVARCHAR(120) NULL,
    address        NVARCHAR(255) NULL,
    blood_group    NVARCHAR(8) NULL,
    status         NVARCHAR(16) NOT NULL CONSTRAINT ck_students_status   CHECK (status IN ('active','inactive','passed_out')),
    is_deleted     BIT NOT NULL DEFAULT 0,
    created_at     DATETIME2 NULL DEFAULT (SYSUTCDATETIME()),
    updated_at     DATETIME2 NULL,
    CONSTRAINT uq_students_admission UNIQUE (institute_id, admission_number)
);
CREATE NONCLUSTERED INDEX ix_students_class ON students (institute_id, class_id, section_id);
```

---

## A. Platform / system (global, no `institute_id`)

```sql
institutes        id BIGINT IDENTITY PK, name NVARCHAR(120), type NVARCHAR(24) CHECK (type IN ('school','college','training_center')),
                  owner_user_id BIGINT NULL FK users, phone NVARCHAR(32), email NVARCHAR(120), default_domain NVARCHAR(255),
                  package_id BIGINT NULL FK packages, subscription_status NVARCHAR(16) CHECK (IN 'active','trial','expired','cancelled'),
                  status NVARCHAR(16) CHECK (IN 'pending','active','suspended'), start_date DATE, amount_paid DECIMAL(18,2),
                  payment_method NVARCHAR(32), demo_mode BIT

packages          id, name NVARCHAR(64), price DECIMAL(18,2), duration_days INT, module_limits NVARCHAR(MAX) (json), status BIT

branches          id, institute_id FK, name NVARCHAR(64), status BIT

custom_domains    id, institute_id FK, domain NVARCHAR(255) UNIQUE, note NVARCHAR(500), status NVARCHAR(16) CHECK (IN 'pending','verified'), submitted_at DATETIME2

institute_settings id, institute_id FK, key NVARCHAR(64), value NVARCHAR(MAX) (json)   -- flattened tenant settings (see 09)

users             id, institute_id BIGINT NULL (platform staff), profile_image NVARCHAR(255), name NVARCHAR(120), email NVARCHAR(120) UNIQUE, phone NVARCHAR(32),
                  password_hash NVARCHAR(255), user_type NVARCHAR(16) CHECK (IN 'admin','accountant','librarian','teacher','student','staff','super_admin'),
                  role_id FK NULL, is_active BIT, demo_locked BIT, last_login_at DATETIME2

roles             id, name NVARCHAR(64), permissions NVARCHAR(MAX) (json)

payment_gateways  id, name NVARCHAR(64), mode NVARCHAR(8) CHECK (IN 'test','live'), config NVARCHAR(MAX) (json), is_active BIT

system_settings   id, key NVARCHAR(64) UNIQUE, value NVARCHAR(MAX)   -- update server url, demo interval, flags

system_modules    id, module NVARCHAR(64) UNIQUE, version NVARCHAR(32), enabled BIT, dependencies NVARCHAR(MAX) (json)

system_updates    id, version NVARCHAR(32), archive_path NVARCHAR(255), status NVARCHAR(16), applied_at DATETIME2, applied_by BIGINT FK users

system_update_history id, version NVARCHAR(32), date DATETIME2, type NVARCHAR(16), status NVARCHAR(16), admin_id BIGINT FK users, duration_seconds INT
```

## B. Academic core

```sql
academic_years   id, institute_id FK, session_name NVARCHAR(64), academic_year NVARCHAR(16), is_current BIT

shifts           id, institute_id FK, name NVARCHAR(32)

classes          id, institute_id FK, name NVARCHAR(64), numeric_value TINYINT, status BIT

sections         id, institute_id FK, class_ids NVARCHAR(MAX) (json m2m), section_name NVARCHAR(24), group_id FK NULL, room_no NVARCHAR(16)

student_groups   id, institute_id FK, group_name NVARCHAR(32)

periods          id, institute_id FK, name NVARCHAR(32), start_time TIME(7), end_time TIME(7), serial TINYINT

subjects         id, institute_id FK, name NVARCHAR(64), code NVARCHAR(16) UNIQUE (per institute), class_id FK NULL,
                 subject_type NVARCHAR(16) CHECK (IN 'compulsory','optional'), status BIT

subject_configs  id, institute_id FK, class_id FK, group_id FK, subject_ids NVARCHAR(MAX) (json m2m),
                 subject_type NVARCHAR(16) CHECK (IN 'compulsory','optional'), subject_serial TINYINT, merge_id NVARCHAR(16)

optional_subject_configs id, institute_id FK, name NVARCHAR(64), class_id FK, group_id FK, subject_ids NVARCHAR(MAX) (json), max_subjects TINYINT

exams            id, institute_id FK, name NVARCHAR(64), exam_code NVARCHAR(16)

student_categories id, institute_id FK, name NVARCHAR(64)

departments      id, institute_id FK, name NVARCHAR(64), priority SMALLINT

picklists        id, institute_id FK, type NVARCHAR(32), value NVARCHAR(64), slug NVARCHAR(64)

signatures       id, institute_id FK, place_at NVARCHAR(16) CHECK (IN 'principal','class_teacher','parent','other'), title NVARCHAR(64), image_path NVARCHAR(255)
```

## C. People: students & staff

```sql
students         id, institute_id FK, user_id FK NULL, image_path NVARCHAR(255), roll INT, admission_number NVARCHAR(32),
                 name NVARCHAR(120), father_name NVARCHAR(120), mother_name NVARCHAR(120), gender NVARCHAR(8) CHECK (IN 'male','female','other'), dob DATE,
                 class_id FK, section_id FK, group_id FK, category_id FK, phone_no NVARCHAR(32), g_mobile NVARCHAR(32),
                 email NVARCHAR(120), address NVARCHAR(255), blood_group NVARCHAR(8), status NVARCHAR(16) CHECK (IN 'active','inactive','passed_out'), created_at DATETIME2

staffs           id, institute_id FK, user_id FK NULL, hr_id NVARCHAR(32) UNIQUE, profile_image NVARCHAR(255),
                 name NVARCHAR(120), phone NVARCHAR(32), email NVARCHAR(120), designation NVARCHAR(64), department_id FK, blood_group NVARCHAR(8), status BIT
                 -- teachers: 'staffs' + flag is_teacher BIT (distinct admin list only; same entity)

migrations       id, institute_id FK, student_id FK, from_class_id, from_section_id, to_class_id, to_section_id,
                 migration_type NVARCHAR(16) CHECK (IN 'promote','demote','transfer'), academic_year_id FK, group_id FK,
                 status NVARCHAR(16) CHECK (IN 'pending','done','pushed_back'), created_at DATETIME2
```

## D. Attendance

```sql
student_attendance id, institute_id FK, class_id FK, section_id FK, student_id FK, date DATE,
                 status NVARCHAR(2) CHECK (IN 'P','A','L','E'), marked_by BIGINT FK users, created_at DATETIME2   -- UNIQUE(institute_id,student_id,date)

exam_attendance  id, institute_id FK, exam_id FK, class_id FK, section_id FK, subject_id FK, student_id FK, status NVARCHAR(2) CHECK (IN 'P','A')

staff_attendance id, institute_id FK, role NVARCHAR(8) CHECK (IN 'teacher','staff'), staff_id FK, date DATE, status NVARCHAR(2) CHECK (IN 'P','A','L'), check_in DATETIME2, check_out DATETIME2

absent_fines     id, institute_id FK, student_id FK, class_id FK, section_id FK, days_absent INT, fine DECIMAL(18,2), period NVARCHAR(16), status NVARCHAR(16)

qr_scans         id, institute_id FK, student_id FK, date DATE, time DATETIME2, device NVARCHAR(64), status NVARCHAR(16)
```

## E. Fees

```sql
fee_heads        id, institute_id FK, parent_id BIGINT NULL (sub-head, self FK), name NVARCHAR(64), serial INT, is_active BIT

fee_mappings     id, institute_id FK, fee_head_id FK, fee_sub_head_ids NVARCHAR(MAX) (json), ledger_id FK, fund_id FK

fee_amount_config id, institute_id FK, class_id FK, group_id FK, section_id FK, student_category_id FK,
                 fee_head_id FK, fee_amount DECIMAL(18,2), fine_amount DECIMAL(18,2), fund_id FK, period NVARCHAR(16), amount DECIMAL(18,2)

fee_date_config  id, institute_id FK, academic_year_id FK, fee_head_id FK, fee_sub_head NVARCHAR(64), payable_date DATE, fine_active_date DATE

attendance_wavers id, institute_id FK, class_id FK, section_id FK, student_id FK, days_waived INT

wavers           id, institute_id FK, name NVARCHAR(64)                          -- entity: Waiver

waiver_configs   id, institute_id FK, academic_year_id FK, group_id FK, class_id FK, section_id FK,
                 student_category_id FK, student_id FK, roll INT, fee_head_id FK, waiver_id FK, amount DECIMAL(18,2)

fee_payments     id, institute_id FK, invoice_no NVARCHAR(32), student_id FK, academic_year_id FK, fee_head_id FK,
                 amount DECIMAL(18,2), fine DECIMAL(18,2), waiver_discount DECIMAL(18,2), paid_by NVARCHAR(16) CHECK (IN 'admin','accountant','online'),
                 payment_method_id FK, fund_id FK, transaction_date DATE, note NVARCHAR(500), created_at DATETIME2
                 -- source of PaidInfo / UnpaidInfo (posts journal lines into accounting)
```

## F. Accounting (double-entry)

```sql
account_categories id, institute_id FK, name NVARCHAR(64), code NVARCHAR(32), type NVARCHAR(16) CHECK (IN 'asset','liability','income','expense')

account_groups   id, institute_id FK, category_id FK, name NVARCHAR(64)

account_ledgers  id, institute_id FK, name NVARCHAR(64), group_id FK, category_id FK, nature NVARCHAR(8) CHECK (IN 'debit','credit'), opening_balance DECIMAL(18,2)

account_funds    id, institute_id FK, name NVARCHAR(64), serial INT, opening_balance DECIMAL(18,2)

vouchers         id, institute_id FK, type NVARCHAR(16) CHECK (IN 'payment','receipt','contra','journal','fund_transfer'), voucher_no NVARCHAR(32),
                 voucher_date DATE, from_fund_id FK, to_fund_id FK, from_ledger_id FK, to_ledger_id FK,
                 reference NVARCHAR(64), description NVARCHAR(500), status NVARCHAR(8) CHECK (IN 'draft','posted'), created_by BIGINT FK users, posted_at DATETIME2

journal_lines    id, institute_id FK, voucher_id FK, ledger_id FK, fund_id FK, debit DECIMAL(18,2), credit DECIMAL(18,2), note NVARCHAR(500)

chart_of_accounts id, institute_id FK, code INT, name NVARCHAR(64), category_id FK, parent_id BIGINT NULL   -- seeded tree of 7 groups

-- Derived reports (views/stored procedures, no storage)
balance_sheet / trial_balance / cash_flow / cash_book / ledger_book /
income_statement / cash_summary   → SP aggregates of journal_lines by ledger/fund/date
```

## G. Payroll

```sql
salary_heads     id, institute_id FK, name NVARCHAR(64), nature NVARCHAR(8) CHECK (IN 'plus','minus')

payroll_mappings id, institute_id FK, ledger_id FK, fund_id FK

payroll_assigns  id, institute_id FK, staff_id FK, effective_date DATE,
                 md DECIMAL(18,2), basic DECIMAL(18,2), allowance DECIMAL(18,2), early_leave_fine DECIMAL(18,2),
                 festival_allowance DECIMAL(18,2), welfare_fund DECIMAL(18,2), professional_tax DECIMAL(18,2),
                 conveyance DECIMAL(18,2), exam_hall_duty DECIMAL(18,2), incentive DECIMAL(18,2), medical DECIMAL(18,2), net_salary DECIMAL(18,2)

salary_slips     id, institute_id FK, staff_id FK, payroll_assign_id FK, period NVARCHAR(16), net_salary DECIMAL(18,2), generated_at DATETIME2

salary_payments  id, institute_id FK, invoice_no NVARCHAR(32), staff_id FK, hr_id NVARCHAR(32), period NVARCHAR(16), net_salary DECIMAL(18,2),
                 payable_salary DECIMAL(18,2), paid DECIMAL(18,2), due DECIMAL(18,2), advance DECIMAL(18,2),
                 payment_type NVARCHAR(16) CHECK (IN 'monthly','due','advance','advance_return'), paid_status NVARCHAR(8) CHECK (IN 'unpaid','paid'),
                 transaction_date DATE, payment_method_id FK

advance_salary   id, institute_id FK, staff_id FK, amount DECIMAL(18,2), date DATE, installments TINYINT, remaining DECIMAL(18,2)
```

## H. Routine & academics content

```sql
syllabus         id, institute_id FK, title NVARCHAR(120), description NVARCHAR(500), class_id FK, subject_id FK, file_path NVARCHAR(255), status NVARCHAR(16)

assignments      id, institute_id FK, title NVARCHAR(120), description NVARCHAR(500), class_id FK, section_id FK, subject_id FK, file_path NVARCHAR(255), due_date DATE

class_routines   id, institute_id FK, class_id FK, section_id FK, day NVARCHAR(8) CHECK (IN 'Sat','Sun','Mon','Tue','Wed','Thu','Fri'),
                 period_id FK, subject_id FK, teacher_id FK, room NVARCHAR(32), start_time TIME(7), end_time TIME(7)

exam_routines    id, institute_id FK, exam_id FK, class_id FK, group_id FK, subject_id FK, date DATE, start_time TIME(7), end_time TIME(7), room NVARCHAR(32)

admit_cards      id, institute_id FK, exam_id FK, class_id FK, section_id FK, student_id FK, seat_no NVARCHAR(16), printed BIT
```

## I. Exams & results

```sql
exam_code_lists  id, institute_id FK, code_title NVARCHAR(64), total_marks DECIMAL(18,2), pass_mark DECIMAL(18,2), acceptance DECIMAL(18,2)

grade_lists      id, institute_id FK, grade NVARCHAR(4), min_mark DECIMAL(18,2), max_mark DECIMAL(18,2), point DECIMAL(5,2)

exam_startups    id, institute_id FK, class_id FK, exam_code_list_id FK, grade_list_id FK, exam_id FK,
                 merit_process_type NVARCHAR(16) CHECK (IN 'gpa','total','percentage')

exam_mark_configs id, institute_id FK, class_id FK, group_id FK, calculation_method NVARCHAR(16) CHECK (IN 'weighted','average'),
                 exam_id FK, percentage DECIMAL(5,2), exam_serial TINYINT

exam_remarks     id, institute_id FK, title NVARCHAR(64), text NVARCHAR(500)

exam_marks       id, institute_id FK, exam_id FK, class_id FK, section_id FK, subject_id FK, student_id FK,
                 written DECIMAL(18,2), mcq DECIMAL(18,2), practical DECIMAL(18,2), total DECIMAL(18,2), grade NVARCHAR(4), point DECIMAL(5,2),
                 UNIQUE(exam_id, subject_id, student_id)

result_cards     id, institute_id FK, exam_id FK, class_id FK, section_id FK, student_id FK, gpa DECIMAL(5,2),
                 position INT, pct DECIMAL(5,2), is_failed BIT, created_at DATETIME2

grand_final_results id, institute_id FK, class_id FK, section_id FK, student_id FK, total_grade_point DECIMAL(5,2),
                 merit_type NVARCHAR(16), position_determinant NVARCHAR(32), created_at DATETIME2   -- computed snapshot

assessment_domains    id, institute_id FK, domain NVARCHAR(16) CHECK (IN 'cognitive','affective','psychomotor'), name NVARCHAR(64), display_order INT
assessment_domain_items id, institute_id FK, domain_id FK, label NVARCHAR(64), max_value TINYINT, type NVARCHAR(8) CHECK (IN 'score','grade')
assessment_entries    id, institute_id FK, class_id FK, exam_id FK, section_id FK, student_id FK,
                 domain_item_id FK, score DECIMAL(18,2), grade NVARCHAR(4)

online_exams    id, institute_id FK, name NVARCHAR(64), exam_id FK NULL, class_id FK, subject_id FK, questions NVARCHAR(MAX) (json),
                window_start_at DATETIME2, window_end_at DATETIME2, duration_min INT, status NVARCHAR(16) CHECK (IN 'draft','published','closed')

online_exam_attempts id, institute_id FK, online_exam_id FK, student_id FK, answers NVARCHAR(MAX) (json), score DECIMAL(18,2), started_at DATETIME2, submitted_at DATETIME2

result_card_settings id, institute_id FK, result_title NVARCHAR(64), show_photo BIT, show_position BIT, show_gpa BIT,
                 show_percentage BIT, show_failed_subjects BIT, show_attendance BIT, show_remarks BIT,
                 show_cognitive BIT, show_affective BIT, show_psychomotor BIT,
                 show_teacher_sig BIT, show_principal_sig BIT, show_parent_sig BIT,
                 class_teacher_label NVARCHAR(64), principal_label NVARCHAR(64), parent_label NVARCHAR(64),
                 class_teacher_sig_path NVARCHAR(255), principal_sig_path NVARCHAR(255),
                 failed_ranking NVARCHAR(16) CHECK (IN 'after_all','with_all'), primary_color NVARCHAR(16), accent_color NVARCHAR(16),
                 watermark_opacity DECIMAL(2,1), show_banner_watermark BIT
```

## J. Certificates

```sql
certificate_templates id, institute_id FK, name NVARCHAR(64), type NVARCHAR(16) CHECK (IN 'general','testimonial','attendance','hsc',
                 'transfer','abroad','character','study','bonafide','migration'),
                 orientation NVARCHAR(16) CHECK (IN 'portrait','landscape'), colors NVARCHAR(MAX) (json), background_path NVARCHAR(255), placeholders NVARCHAR(MAX) (json), status BIT

certificates     id, institute_id FK, template_id FK, type NVARCHAR(16), student_id FK, class_id FK, section_id FK,
                 data NVARCHAR(MAX) (json, merged fields), printed BIT, issued_at DATETIME2
```

## K. Library

```sql
book_categories id, institute_id FK, name NVARCHAR(64)

books           id, institute_id FK, name NVARCHAR(120), code NVARCHAR(32), barcode NVARCHAR(64) UNIQUE, author NVARCHAR(64),
                category_id FK, quantity INT, available INT, rack NVARCHAR(32), status BIT

library_members id, institute_id FK, library_member_id NVARCHAR(32), user_id FK, name NVARCHAR(120), image_path NVARCHAR(255),
                member_type NVARCHAR(16) CHECK (IN 'student','teacher','staff'), status BIT

book_issues     id, institute_id FK, book_id FK, member_id FK, issue_date DATE, return_date DATE, returned BIT, status NVARCHAR(16)
```

## L. Hostel

```sql
hostels         id, institute_id FK, name NVARCHAR(64), type NVARCHAR(8) CHECK (IN 'boys','girls','mixed'), address NVARCHAR(255), status NVARCHAR(16)

hostel_categories id, institute_id FK, hostel_id FK, standard NVARCHAR(32), fee DECIMAL(18,2), note NVARCHAR(500)

hostel_members  id, institute_id FK, student_id FK, hostel_id FK, category_id FK, class_id FK, fee DECIMAL(18,2), status NVARCHAR(16)

hostel_buildings id, institute_id FK, hostel_id FK, name NVARCHAR(64)

hostel_floors   id, institute_id FK, building_id FK, name NVARCHAR(64)

rooms           id, institute_id FK, hostel_id FK, floor_id FK, number NVARCHAR(16), hostel_category_id FK, capacity TINYINT

beds            id, institute_id FK, room_id FK, bed_number NVARCHAR(8), status NVARCHAR(16) CHECK (IN 'free','assigned','maintenance')

room_members    id, institute_id FK, student_id FK, room_id FK, bed_id FK, hostel_category_id FK, assigned_at DATETIME2

meals           id, institute_id FK, name NVARCHAR(64), type NVARCHAR(16) CHECK (IN 'breakfast','lunch','dinner','snack')

meal_plans      id, institute_id FK, student_id FK, meal_id FK, date DATE

meal_entries    id, institute_id FK, student_id FK, meal_id FK, date DATE, price DECIMAL(18,2)

hostel_bills    id, institute_id FK, student_id FK, period NVARCHAR(16), hostel_fee DECIMAL(18,2), meal_fee DECIMAL(18,2),
                total DECIMAL(18,2), due_date DATE, status NVARCHAR(8) CHECK (IN 'unpaid','partial','paid')

hostel_leaves   id, institute_id FK, student_id FK, room_id FK, from DATE, to DATE, reason NVARCHAR(255), status NVARCHAR(16) CHECK (IN 'pending','approved','rejected')

hostel_collections id, institute_id FK, invoice NVARCHAR(32), student_id FK, room_id FK, bed_id FK, month NVARCHAR(16),
                amount DECIMAL(18,2), paid DECIMAL(18,2), due DECIMAL(18,2), date DATE
```

## M. Transport

```sql
buses           id, institute_id FK, bus_number NVARCHAR(32), model NVARCHAR(64), capacity TINYINT, registration NVARCHAR(32), status BIT

drivers         id, institute_id FK, name NVARCHAR(120), phone NVARCHAR(32), license_no NVARCHAR(32), assigned_bus_id FK, status BIT

helpers         id, institute_id FK, name NVARCHAR(120), phone NVARCHAR(32), assigned_bus_id FK, status BIT

vehicle_types   id, institute_id FK, name NVARCHAR(64), status BIT

vehicle_categories id, institute_id FK, name NVARCHAR(64), status BIT

bus_routes      id, institute_id FK, route_name NVARCHAR(64), start_location NVARCHAR(128), end_location NVARCHAR(128), distance DECIMAL(8,2), estimated_time INT (min), status BIT

bus_stops       id, institute_id FK, route_id FK, stop_name NVARCHAR(64), latitude DECIMAL(10,7), longitude DECIMAL(10,7), order_pos TINYINT

transport_members id, institute_id FK, student_id FK, route_id FK, stop_id FK, fare DECIMAL(18,2), status BIT

transport_collections id, institute_id FK, invoice NVARCHAR(32), student_id FK, route_id FK, stop_id FK, month NVARCHAR(16),
                fare DECIMAL(18,2), paid DECIMAL(18,2), due DECIMAL(18,2), date DATE
```

## N. Inventory

```sql
inventory_categories id, institute_id FK, name NVARCHAR(64), note NVARCHAR(500)

inventory_items id, institute_id FK, name NVARCHAR(120), category_id FK, sku NVARCHAR(32), cost_price DECIMAL(18,2),
                selling_price DECIMAL(18,2), stock INT, status BIT

inventory_sales id, institute_id FK, invoice NVARCHAR(32), student_id FK, date DATE, payable DECIMAL(18,2), paid DECIMAL(18,2), due DECIMAL(18,2)
inventory_sale_lines id, institute_id FK, sale_id FK, item_id FK, qty INT, unit_price DECIMAL(18,2), total DECIMAL(18,2)
```

## O. Communication (SMS / WhatsApp / notices)

```sql
sms_templates    id, institute_id FK, title NVARCHAR(64), message NVARCHAR(MAX)

phone_book_categories id, institute_id FK, title NVARCHAR(64), description NVARCHAR(500)

phone_book_contacts id, institute_id FK, name NVARCHAR(120), phone NVARCHAR(32), category_id FK, class_id FK, section_id FK, note NVARCHAR(500)

sms_messages     id, institute_id FK, class_id BIGINT NULL, section_id BIGINT NULL, to_number NVARCHAR(32), template_id FK,
                 message NVARCHAR(300), gateway NVARCHAR(32) CHECK (IN 'sms_gateway','twilio'), status NVARCHAR(8) CHECK (IN 'queued','sent','failed'),
                 sent_at DATETIME2, gateway_response NVARCHAR(MAX)

sms_purchases    id, institute_id FK, gateway NVARCHAR(64), quantity INT, price DECIMAL(18,2), transaction_date DATE,
                 masking_type NVARCHAR(16) CHECK (IN 'masking','non_masking'), status NVARCHAR(16)

-- sms_report: SP view — aggregate sent/delivered by date range

whatsapp_settings id, institute_id FK, provider NVARCHAR(32), phone_id NVARCHAR(64), business_id NVARCHAR(64), access_key NVARCHAR(255) (secret), language NVARCHAR(8), status BIT
whatsapp_templates id, institute_id FK, name NVARCHAR(64), event NVARCHAR(32), message NVARCHAR(MAX), status BIT
whatsapp_logs    id, institute_id FK, student_id FK, phone NVARCHAR(32), message NVARCHAR(MAX), retries TINYINT, sent_at DATETIME2, status NVARCHAR(16)

notices          id, institute_id FK, image_path NVARCHAR(255), title NVARCHAR(120), audience NVARCHAR(16) CHECK (IN 'all','student','teacher','staff','parent'),
                 body NVARCHAR(MAX), status BIT, created_at DATETIME2

notifications    id, institute_id FK, user_id FK, channel NVARCHAR(16) CHECK (IN 'sms','whatsapp','email','meet','app'), title NVARCHAR(120), body NVARCHAR(MAX),
                 payload NVARCHAR(MAX) (json), read_at DATETIME2, sent_at DATETIME2, status NVARCHAR(16)
```

## P. Google Meet

```sql
google_meet_sessions id, institute_id FK, title NVARCHAR(120), teacher_id FK, class_id FK, section_id FK, group_id BIGINT NULL,
                 subject_id BIGINT NULL, description NVARCHAR(MAX), start_at DATETIME2, end_at DATETIME2, duration_min INT,
                 visibility NVARCHAR(16) CHECK (IN 'private','class','group','everyone'), meet_link NVARCHAR(255), status NVARCHAR(16),
                 recurring BIT, repeat NVARCHAR(8) CHECK (IN 'daily','weekly','monthly'), until DATE NULL,
                 created_by BIGINT FK users              -- notifications target m2m: meet_recipients(meet_id, role NVARCHAR(16) CHECK IN ('student','guardian'))
google_meet_credentials -- part of institute_settings JSON
```

## Q. AI

```sql
ai_settings      id, institute_id FK, anthropic_api_key NVARCHAR(255) (secret), model NVARCHAR(32), enabled BIT

ai_chat_sessions id, institute_id FK, user_id FK, thread_id NVARCHAR(64), created_at DATETIME2
ai_chat_messages id, institute_id FK, session_id FK, role NVARCHAR(16) CHECK (IN 'user','assistant'), content NVARCHAR(MAX), created_at DATETIME2

ai_writer_outputs id, institute_id FK, user_id FK, content_type NVARCHAR(32), prompt NVARCHAR(MAX), result NVARCHAR(MAX), created_at DATETIME2
ai_insight_runs  id, institute_id FK, user_id FK, report NVARCHAR(64), question NVARCHAR(500), answer NVARCHAR(MAX), created_at DATETIME2
```

## R. CMS / public content

```sql
cms_pages        id, institute_id FK, title NVARCHAR(120), slug NVARCHAR(128) UNIQUE (per institute), content NVARCHAR(MAX), seo NVARCHAR(MAX) (json), status BIT

banners          id, institute_id FK, title NVARCHAR(120), image_path NVARCHAR(255), button_name NVARCHAR(64), button_link NVARCHAR(255), description NVARCHAR(500), serial INT, status BIT

about_us_items   id, institute_id FK, title NVARCHAR(120), image_path NVARCHAR(255), short_description NVARCHAR(500), status BIT

faqs             id, institute_id FK, question NVARCHAR(500), answer NVARCHAR(MAX), serial INT, status BIT

gallery_images   id, institute_id FK, title NVARCHAR(120), heading NVARCHAR(120), image_path NVARCHAR(255), status BIT

mobile_app_sections id, institute_id FK, title NVARCHAR(120), heading NVARCHAR(120), image_path NVARCHAR(255), features NVARCHAR(MAX) (json), store_links NVARCHAR(MAX) (json), status BIT

why_choose_us    id, institute_id FK, title NVARCHAR(120), icon NVARCHAR(64), description NVARCHAR(500), serial INT, status BIT

policies         id, institute_id FK, type NVARCHAR(16) CHECK (IN 'privacy','terms','refund','cookies'), description NVARCHAR(MAX), status BIT

ready_to_join_us id, institute_id FK, icon NVARCHAR(64), title NVARCHAR(120), description NVARCHAR(500), button NVARCHAR(64), status BIT

testimonials     id, institute_id FK, image_path NVARCHAR(255), name NVARCHAR(120), designation NVARCHAR(64), review NVARCHAR(MAX), rating TINYINT (1-5), status BIT

admission_applications id, institute_id FK, student_name NVARCHAR(120), class_id FK, contact_phone NVARCHAR(32), email NVARCHAR(120), guardian NVARCHAR(120),
                 address NVARCHAR(255), documents NVARCHAR(MAX) (json), status NVARCHAR(16) CHECK (IN 'pending','approved','rejected'), submitted_at DATETIME2

contact_messages id, institute_id FK, name NVARCHAR(120), phone NVARCHAR(32), email NVARCHAR(120), message NVARCHAR(MAX), status NVARCHAR(16) CHECK (IN 'new','replied','closed'), created_at DATETIME2

user_activity_logs id, institute_id FK, user_id FK, name NVARCHAR(120), ip_address NVARCHAR(64), action NVARCHAR(64), detail NVARCHAR(MAX), created_at DATETIME2
```

## S. Auth / session

```sql
password_resets        email NVARCHAR(120), token NVARCHAR(255), created_at DATETIME2 -- single-use, 60 min expiry
sessions               id BIGINT IDENTITY PK, user_id FK, ip_address NVARCHAR(64), user_agent NVARCHAR(500), payload NVARCHAR(MAX), last_activity DATETIME2
refresh_tokens         (if JWT added for future API) id BIGINT IDENTITY PK, user_id FK, family_hash NVARCHAR(255), expires_at DATETIME2, revoked_at DATETIME2 NULL
```

## Indexes & constraints to add

- `UNIQUE(students.institute_id, admission_number)`, `UNIQUE(institute_id, roll, class_id)`
- `UNIQUE(student_attendance.institute_id, student_id, date)`
- `UNIQUE(books.institute_id, code)`, `UNIQUE(books.institute_id, barcode)`
- `UNIQUE(book_issues.institute_id, book_id, member_id, issue_date)`
- `UNIQUE(journal_lines per voucher)`
- `CREATE NONCLUSTERED INDEX` on: `fee_payments(student_id, academic_year_id)`, `exam_marks(exam_id, class_id, section_id, subject_id)`, `student_attendance(class_id, section_id, date)`, `journal_lines(ledger_id, date)`, plus a running-balance index on `account_funds`.
- FK `ON DELETE NO ACTION` everywhere (audit integrity); parent deletes cascade only for line/child config tables.