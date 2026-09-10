# 02 — Database Schema

Shared-schema multi-tenant MySQL. Conventions:

- Every tenant-scoped table carries `institute_id BIGINT unsigned` (indexed; global applied scope).
- `id` = `BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT` unless shown.
- `created_at`/`updated_at` timestamps implicit.
- Money = `DECIMAL(15,2)`; percentages `DECIMAL(5,2)`; booleans `TINYINT(1)`.
- `enum` values are the observed UI options.

---

## A. Platform / system (global, no `institute_id`)

```sql
institutes       id, name, type ENUM('school','college','training_center'), owner_user_id FK,
                 phone, email, default_domain, package_id FK, subscription_status ENUM('active','trial','expired','cancelled'),
                 status ENUM('pending','active','suspended'), start_date DATE, amount_paid DECIMAL, payment_method, demo_mode BOOL

packages         id, name, price DECIMAL, duration_days INT, module_limits JSON, status BOOL

branches         id, institute_id FK, name, status BOOL

custom_domains   id, institute_id FK, domain VARCHAR(255) UNIQUE, note, status ENUM('pending','verified'), submitted_at

institute_settings id, institute_id FK, key VARCHAR(64), value JSON   -- flattened tenant settings (see 09)

users            id, institute_id FK NULL(sys staff), profile_image, name, email UNIQUE, phone,
                 password_hash, user_type ENUM('admin','accountant','librarian','teacher','student','staff','super_admin'),
                 role_id FK, is_active BOOL, demo_locked BOOL, last_login_at

roles            id, name, permissions JSON

payment_gateways id, name, mode ENUM('test','live'), config JSON, is_active BOOL

system_settings  id, key VARCHAR(64) UNIQUE, value TEXT   -- update server url, demo interval, flags

system_modules   id, module VARCHAR(64) UNIQUE, version, enabled BOOL, dependencies JSON

system_updates   id, version, archive_path, status, applied_at, applied_by FK users

system_update_history id, version, date, type, status, admin_id FK, duration_seconds
```

## B. Academic core

```sql
academic_years   id, institute_id FK, session_name VARCHAR(64), academic_year VARCHAR(16), is_current BOOL

shifts           id, institute_id FK, name VARCHAR(32)

classes          id, institute_id FK, name VARCHAR(64), numeric_value TINYINT, status BOOL

sections         id, institute_id FK, class_ids JSON(m2m), section_name VARCHAR(16), group_id FK NULL, room_no VARCHAR(16)

student_groups   id, institute_id FK, group_name VARCHAR(32)

periods          id, institute_id FK, name VARCHAR(32), start_time TIME, end_time TIME, serial TINYINT

subjects         id, institute_id FK, name VARCHAR(64), code VARCHAR(16) UNIQUE(per institute), class_id FK NULL,
                 subject_type ENUM('compulsory','optional'), status BOOL

subject_configs  id, institute_id FK, class_id FK, group_id FK, subject_ids JSON(m2m),
                 subject_type ENUM('compulsory','optional'), subject_serial TINYINT, merge_id VARCHAR(16)

optional_subject_configs id, institute_id FK, name, class_id FK, group_id FK, subject_ids JSON, max_subjects TINYINT

exams            id, institute_id FK, name VARCHAR(64), exam_code VARCHAR(16)

student_categories id, institute_id FK, name VARCHAR(64)

departments      id, institute_id FK, name VARCHAR(64), priority SMALLINT

picklists        id, institute_id FK, type VARCHAR(32), value VARCHAR(64), slug VARCHAR(64)

signatures       id, institute_id FK, place_at ENUM('principal','class_teacher','parent','other'), title, image_path
```

## C. People: students & staff

```sql
students         id, institute_id FK, user_id FK NULL, image_path, roll INT, admission_number VARCHAR(32),
                 name, father_name, mother_name, gender ENUM('male','female','other'), dob DATE,
                 class_id FK, section_id FK, group_id FK, category_id FK, phone_no, g_mobile,
                 email, address, blood_group, status ENUM('active','inactive','passed_out'), created_at

staffs           id, institute_id FK, user_id FK NULL, hr_id VARCHAR(32) UNIQUE, profile_image,
                 name, phone, email, designation, department_id FK, blood_group, status BOOL
                 -- teachers: 'staffs' + flag is_teacher BOOL (distinct admin list only; same entity)

migrations       id, institute_id FK, student_id FK, from_class_id, from_section_id, to_class_id, to_section_id,
                 migration_type ENUM('promote','demote','transfer'), academic_year_id FK, group_id FK,
                 status ENUM('pending','done','pushed_back'), created_at
```

## D. Attendance

```sql
student_attendance id, institute_id FK, class_id FK, section_id FK, student_id FK, date DATE,
                 status ENUM('P','A','L','E'), marked_by FK users, created_at   -- UNIQUE(student_id,date)

exam_attendance  id, institute_id FK, exam_id FK, class_id FK, section_id FK, subject_id FK, student_id FK, status ENUM('P','A')

staff_attendance id, institute_id FK, role ENUM('teacher','staff'), staff_id FK, date DATE, status ENUM('P','A','L'), check_in, check_out

absent_fines     id, institute_id FK, student_id FK, class_id FK, section_id FK, days_absent INT, fine DECIMAL, period, status

qr_scans         id, institute_id FK, student_id FK, date DATE, time, device, status
```

## E. Fees

```sql
fee_heads        id, institute_id FK, parent_id FK NULL(sub-head), name, serial, is_active BOOL

fee_mappings     id, institute_id FK, fee_head_id FK, fee_sub_head_ids JSON, ledger_id FK, fund_id FK

fee_amount_config id, institute_id FK, class_id FK, group_id FK, section_id FK, student_category_id FK,
                 fee_head_id FK, fee_amount DECIMAL, fine_amount DECIMAL, fund_id FK, period, amount DECIMAL

fee_date_config  id, institute_id FK, academic_year_id FK, fee_head_id FK, fee_sub_head, payable_date DATE, fine_active_date DATE

attendance_wavers id, institute_id FK, class_id FK, section_id FK, student_id FK, days_waived INT

wavers           id, institute_id FK, name VARCHAR(64)

waiver_configs   id, institute_id FK, academic_year_id FK, group_id FK, class_id FK, section_id FK,
                 student_category_id FK, student_id FK, roll INT, fee_head_id FK, waiver_id FK, amount DECIMAL

fee_payments     id, institute_id FK, invoice_no VARCHAR(32), student_id FK, academic_year_id FK, fee_head_id FK,
                 amount DECIMAL, fine DECIMAL, waiver_discount DECIMAL, paid_by ENUM('admin','accountant','online'),
                 payment_method_id FK, fund_id FK, transaction_date DATE, note, created_at
                 -- source of PaidInfo / UnpaidInfo (post ledger lines in fees_paid_transactions)
```

## F. Accounting (double-entry)

```sql
account_categories id, institute_id FK, name, code, type ENUM('asset','liability','income','expense')

account_groups   id, institute_id FK, category_id FK, name

account_ledgers  id, institute_id FK, name, group_id FK, category_id FK, nature ENUM('debit','credit'), opening_balance DECIMAL

account_funds    id, institute_id FK, name, serial, opening_balance DECIMAL

vouchers         id, institute_id FK, type ENUM('payment','receipt','contra','journal','fund_transfer'), voucher_no,
                 voucher_date DATE, from_fund_id FK, to_fund_id FK, from_ledger_id FK, to_ledger_id FK,
                 reference, description, status ENUM('draft','posted'), created_by FK users, posted_at

journal_lines    id, institute_id FK, voucher_id FK, ledger_id FK, fund_id FK, debit DECIMAL, credit DECIMAL, note

chart_of_accounts id, institute_id FK, code INT, name, category_id FK, parent_id FK   -- seeded tree of 7 groups

-- Derived reports (views, no storage)
balance_sheet / trial_balance / cash_flow / cash_book / ledger_book /
income_statement / cash_summary   → aggregate of journal_lines by ledger/fund/date
```

## G. Payroll

```sql
salary_heads     id, institute_id FK, name, nature ENUM('plus','minus')

payroll_mappings id, institute_id FK, ledger_id FK, fund_id FK

payroll_assigns  id, institute_id FK, staff_id FK, effective_date DATE,
                 md DECIMAL, basic DECIMAL, allowance DECIMAL, early_leave_fine DECIMAL,
                 festival_allowance DECIMAL, welfare_fund DECIMAL, professional_tax DECIMAL,
                 conveyance DECIMAL, exam_hall_duty DECIMAL, incentive DECIMAL, medical DECIMAL, net_salary DECIMAL

salary_slips     id, institute_id FK, staff_id FK, payroll_assign_id FK, period, net_salary, generated_at

salary_payments  id, institute_id FK, invoice_no, staff_id FK, hr_id, period, net_salary DECIMAL,
                 payable_salary DECIMAL, paid DECIMAL, due DECIMAL, advance DECIMAL,
                 payment_type ENUM('monthly','due','advance','advance_return'), paid_status ENUM('unpaid','paid'),
                 transaction_date DATE, payment_method_id FK

advance_salary   id, institute_id FK, staff_id FK, amount DECIMAL, date DATE, installments TINYINT, remaining DECIMAL
```

## H. Routine & academics content

```sql
syllabus         id, institute_id FK, title, description, class_id FK, subject_id FK, file_path, status

assignments      id, institute_id FK, title, description, class_id FK, section_id FK, subject_id FK, file_path, due_date

class_routines   id, institute_id FK, class_id FK, section_id FK, day ENUM('Sat'..'Fri'),
                 period_id FK, subject_id FK, teacher_id FK, room, start_time, end_time

exam_routines    id, institute_id FK, exam_id FK, class_id FK, group_id FK, subject_id FK, date DATE, start_time, end_time, room

admit_cards      id, institute_id FK, exam_id FK, class_id FK, section_id FK, student_id FK, seat_no, printed BOOL
```

## I. Exams & results

```sql
exam_code_lists  id, institute_id FK, code_title, total_marks DECIMAL, pass_mark DECIMAL, acceptance DECIMAL

grade_lists      id, institute_id FK, grade CHAR(2), min_mark DECIMAL, max_mark DECIMAL, point DECIMAL

exam_startups    id, institute_id FK, class_id FK, exam_code_list_id FK, grade_list_id FK, exam_id FK,
                 merit_process_type ENUM('gpa','total','percentage')

exam_mark_configs id, institute_id FK, class_id FK, group_id FK, calculation_method ENUM('weighted','average'),
                 exam_id FK, percentage DECIMAL, exam_serial TINYINT

exam_remarks     id, institute_id FK, title, text

exam_marks       id, institute_id FK, exam_id FK, class_id FK, section_id FK, subject_id FK, student_id FK,
                 written DECIMAL, mcq DECIMAL, practical DECIMAL, total DECIMAL, grade, point,
                 UNIQUE(exam_id,subject_id,student_id)

result_cards     id, institute_id FK, exam_id FK, class_id FK, section_id FK, student_id FK, gpa DECIMAL,
                 position INT, % DECIMAL, is_failed BOOL, created_at

grand_final_results id, institute_id FK, class_id FK, section_id FK, student_id FK, total_grade_point,
                 merit_type, position_determinant, created_at   -- computed snapshot

assessment_domains    id, institute_id FK, domain ENUM('cognitive','affective','psychomotor'), name, display_order
assessment_domain_items id, institute_id FK, domain_id FK, label, max_value TINYINT, type ENUM('score','grade')
assessment_entries    id, institute_id FK, class_id FK, exam_id FK, section_id FK, student_id FK,
                 domain_item_id FK, score DECIMAL, grade

online_exams    id, institute_id FK, name, exam_id FK NULL, class_id FK, subject_id FK, questions JSON,
                window_start_at, window_end_at, duration_min, status ENUM('draft','published','closed')

online_exam_attempts id, institute_id FK, online_exam_id FK, student_id FK, answers JSON, score DECIMAL, started_at, submitted_at

result_card_settings id, institute_id FK, result_title, show_photo BOOL, show_position BOOL, show_gpa BOOL,
                 show_percentage BOOL, show_failed_subjects BOOL, show_attendance BOOL, show_remarks BOOL,
                 show_cognitive BOOL, show_affective BOOL, show_psychomotor BOOL,
                 show_teacher_sig BOOL, show_principal_sig BOOL, show_parent_sig BOOL,
                 class_teacher_label, principal_label, parent_label,
                 class_teacher_sig_path, principal_sig_path,
                 failed_ranking ENUM('after_all','with_all'), primary_color, accent_color,
                 watermark_opacity DECIMAL(2,1), show_banner_watermark BOOL
```

## J. Certificates

```sql
certificate_templates id, institute_id FK, name, type ENUM('general','testimonial','attendance','hsc',
                 'transfer','abroad','character','study','bonafide','migration'),
                 orientation ENUM('portrait','landscape'), colors JSON, background_path, placeholders JSON, status BOOL

certificates     id, institute_id FK, template_id FK, type, student_id FK, class_id FK, section_id FK,
                 data JSON(merged fields), printed BOOL, issued_at
```

## K. Library

```sql
book_categories id, institute_id FK, name

books           id, institute_id FK, name, code VARCHAR(32), barcode VARCHAR(64) UNIQUE, author,
                category_id FK, quantity INT, available INT, rack, status BOOL

library_members id, institute_id FK, library_member_id VARCHAR(32), user_id FK, name, image_path,
                member_type ENUM('student','teacher','staff'), status BOOL

book_issues     id, institute_id FK, book_id FK, member_id FK, issue_date DATE, return_date DATE, returned BOOL, status
```

## L. Hostel

```sql
hostels         id, institute_id FK, name, type ENUM('boys','girls','mixed'), address, status

hostel_categories id, institute_id FK, hostel_id FK, standard VARCHAR(32), fee DECIMAL, note

hostel_members  id, institute_id FK, student_id FK, hostel_id FK, category_id FK, class_id FK, fee DECIMAL, status

hostel_buildings id, institute_id FK, hostel_id FK, name

hostel_floors   id, institute_id FK, building_id FK, name

rooms           id, institute_id FK, hostel_id FK, floor_id FK, number VARCHAR(16), hostel_category_id FK, capacity TINYINT

beds            id, institute_id FK, room_id FK, bed_number VARCHAR(8), status ENUM('free','assigned','maintenance')

room_members    id, institute_id FK, student_id FK, room_id FK, bed_id FK, hostel_category_id FK, assigned_at

meals           id, institute_id FK, name, type ENUM('breakfast','lunch','dinner','snack')

meal_plans      id, institute_id FK, student_id FK, meal_id FK, date DATE

meal_entries    id, institute_id FK, student_id FK, meal_id FK, date DATE, price DECIMAL

hostel_bills    id, institute_id FK, student_id FK, period, hostel_fee DECIMAL, meal_fee DECIMAL,
                total DECIMAL, due_date DATE, status ENUM('unpaid','partial','paid')

hostel_leaves   id, institute_id FK, student_id FK, room_id FK, from DATE, to DATE, reason, status ENUM('pending','approved','rejected')

hostel_collections id, institute_id FK, invoice, student_id FK, room_id FK, bed_id FK, month,
                amount DECIMAL, paid DECIMAL, due DECIMAL, date DATE
```

## M. Transport

```sql
buses           id, institute_id FK, bus_number, model, capacity TINYINT, registration, status BOOL

drivers         id, institute_id FK, name, phone, license_no, assigned_bus_id FK, status BOOL

helpers         id, institute_id FK, name, phone, assigned_bus_id FK, status BOOL

vehicle_types   id, institute_id FK, name, status BOOL

vehicle_categories id, institute_id FK, name, status BOOL

bus_routes      id, institute_id FK, route_name, start_location, end_location, distance DECIMAL, estimated_time INT(min), status BOOL

bus_stops       id, institute_id FK, route_id FK, stop_name, latitude DECIMAL(10,7), longitude DECIMAL(10,7), order_pos TINYINT

transport_members id, institute_id FK, student_id FK, route_id FK, stop_id FK, fare DECIMAL, status BOOL

transport_collections id, institute_id FK, invoice, student_id FK, route_id FK, stop_id FK, month,
                fare DECIMAL, paid DECIMAL, due DECIMAL, date DATE
```

## N. Inventory

```sql
inventory_categories id, institute_id FK, name, note

inventory_items id, institute_id FK, name, category_id FK, sku VARCHAR(32), cost_price DECIMAL,
                selling_price DECIMAL, stock INT, status BOOL

inventory_sales id, institute_id FK, invoice, student_id FK, date DATE, payable DECIMAL, paid DECIMAL, due DECIMAL
inventory_sale_lines id, institute_id FK, sale_id FK, item_id FK, qty INT, unit_price DECIMAL, total DECIMAL
```

## O. Communication (SMS / WhatsApp / notices)

```sql
sms_templates    id, institute_id FK, title, message

phone_book_categories id, institute_id FK, title, description

phone_book_contacts id, institute_id FK, name, phone, category_id FK, class_id FK, section_id FK, note

sms_messages     id, institute_id FK, class_id FK NULL, section_id FK NULL, to_number, template_id FK,
                 message, gateway ENUM('sms_gateway','twilio'), status ENUM('queued','sent','failed'),
                 sent_at, gateway_response

sms_purchases    id, institute_id FK, gateway, quantity INT, price DECIMAL, transaction_date DATE,
                 masking_type ENUM('masking','non_masking'), status

sms_report       -- view: aggregate sent/delivered by date range

whatsapp_settings id, institute_id FK, provider, phone_id, business_id, access_key, language, status BOOL
whatsapp_templates id, institute_id FK, name, event, message, status BOOL
whatsapp_logs    id, institute_id FK, student_id FK, phone, message, retries TINYINT, sent_at, status

notices          id, institute_id FK, image_path, title, audience ENUM('all','student','teacher','staff','parent'),
                 body, status BOOL, created_at

notifications    id, institute_id FK, user_id FK, channel ENUM('sms','whatsapp','email','meet','app'), title, body,
                 payload JSON, read_at, sent_at, status
```

## P. Google Meet

```sql
google_meet_sessions id, institute_id FK, title, teacher_id FK, class_id FK, section_id FK, group_id FK NULL,
                 subject_id FK NULL, description, start_at DATETIME, end_at DATETIME, duration_min INT,
                 visibility ENUM('private','class','group','everyone'), meet_link, status,
                 recurring BOOL, repeat ENUM('daily','weekly','monthly'), until DATE NULL,
                 created_by FK users              -- notifications target m2m: meet_recipients(meet_id, role ENUM('student','guardian'))
google_meet_credentials -- part of institute_settings JSON
```

## Q. AI

```sql
ai_settings      id, institute_id FK, anthropic_api_key, model VARCHAR(32), enabled BOOL

ai_chat_sessions id, institute_id FK, user_id FK, thread_id, created_at
ai_chat_messages id, institute_id FK, session_id FK, role ENUM('user','assistant'), content TEXT, created_at

ai_writer_outputs id, institute_id FK, user_id FK, content_type, prompt, result, created_at
ai_insight_runs  id, institute_id FK, user_id FK, report, question, answer TEXT, created_at
```

## R. CMS / public content

```sql
cms_pages        id, institute_id FK, title, slug VARCHAR(128) UNIQUE(per institute), content LONGTEXT, seo JSON, status BOOL

banners          id, institute_id FK, title, image_path, button_name, button_link, description, serial, status BOOL

about_us_items   id, institute_id FK, title, image_path, short_description, status BOOL

faqs             id, institute_id FK, question, answer, serial, status BOOL

gallery_images   id, institute_id FK, title, heading, image_path, status BOOL

mobile_app_sections id, institute_id FK, title, heading, image_path, features JSON, store_links JSON, status BOOL

why_choose_us    id, institute_id FK, title, icon, description, serial, status BOOL

policies         id, institute_id FK, type ENUM('privacy','terms','refund','cookies'), description LONGTEXT, status BOOL

ready_to_join_us id, institute_id FK, icon, title, description, button, status BOOL

testimonials     id, institute_id FK, image_path, name, designation, review, rating TINYINT(1-5), status BOOL

admission_applications id, institute_id FK, student_name, class_id FK, contact_phone, email, guardian,
                 address, documents JSON, status ENUM('pending','approved','rejected'), submitted_at

contact_messages id, institute_id FK, name, phone, email, message, status ENUM('new','replied','closed'), created_at

user_activity_logs id, institute_id FK, user_id FK, name, ip_address, action, detail, created_at
```

## S. Auth / session

```sql
password_resets   email, token, created_at
sessions          id, user_id FK, ip_address, user_agent, payload, last_activity
personal_access_tokens (if Sanctum for future API) id, tokenable_id, name, token, abilities, last_used_at, expires_at
```

## Indexes & constraints to add

- `UNIQUE(students.institute_id, admission_number)`, `UNIQUE(institute_id, roll, class_id)`
- `UNIQUE(student_attendance.institute_id, student_id, date)`
- `UNIQUE(books.institute_id, code)`, `UNIQUE(books.institute_id, barcode)`
- `UNIQUE(book_issues.institute_id, book_id, member_id, issue_date)`
- `UNIQUE(journal_lines per voucher)`
- Indexes: `fee_payments(student_id, academic_year_id)`, `exam_marks(exam_id,class_id,section_id,subject_id)`, `attendance(class_id,section_id,date)`, `journal_lines(ledger_id,date)`, `account_funds running balance`.
- FK `ON DELETE RESTRICT` everywhere (audit integrity); parent deletes cascade only for lines/child configs.