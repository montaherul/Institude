# Entity Inventory — `https://institute.bdboibazer.com/` (Mighty School)

> **Derived from** read-only audit of the live demo (login as superadmin). No database/source access exists from outside, so this is a **behavioral entity model**: every entity + attribute is inferred from admin screens, forms, table columns and endpoints.
>
> Attribute types are best-effort from input types (text/number/date/select/file). Keys: `PK` primary-ish, `FK` reference, `enum` dropdown, `m2m` many-to-many, `file` upload, `percent`, `bool`, `auto`.

**Scope: ~190 entities across 27 domains** (25 admin sidebar groups + public/tenant + auth).

---

## Table of Contents

1. [Tenant & Core Platform](#1-tenant--core-platform)
2. [Students Information](#2-students-information)
3. [Staffs Information](#3-staffs-information)
4. [Student Attendance](#4-student-attendance)
5. [QR Code Attendance](#5-qr-code-attendance)
6. [Academic Configuration](#6-academic-configuration)
7. [Fees Management](#7-fees-management)
8. [Accounts Management](#8-accounts-management)
9. [Accounting Reports](#9-accounting-reports)
10. [Payroll Management](#10-payroll-management)
11. [Routine Management](#11-routine-management)
12. [Library Management](#12-library-management)
13. [Exam Module](#13-exam-module)
14. [Layout & Certificates](#14-layout--certificates)
15. [SMS Module](#15-sms-module)
16. [Administrator](#16-administrator)
17. [System](#17-system)
18. [Master Configuration](#18-master-configuration)
19. [CMS Management](#19-cms-management)
20. [WhatsApp](#20-whatsapp)
21. [Hostel Management](#21-hostel-management)
22. [Inventory](#22-inventory)
23. [Transport Management](#23-transport-management)
24. [Google Meet](#24-google-meet)
25. [AI Assistant](#25-ai-assistant)
26. [Auth, Profile & Public Forms](#26-auth-profile--public-forms)
27. [Cross-cutting / shared entities](#27-cross-cutting--shared-entities)

---

## 1. Tenant & Core Platform

### Institute
- **Screen:** `/institutes`
- **Purpose:** Top-level tenant (school/college/training center) in the SaaS.
- **Attributes:**
  - `id` (PK)
  - `institute_name` (string)
  - `type` (enum: school/college/training-center…)
  - `owner_id` (FK → User)
  - `phone` (string)
  - `email` (string)
  - `domain` (string) — default subdomain or custom
  - `package_id` (FK → Package)
  - `subscription_status` (enum)
  - `status` (enum: pending/active/suspended)
  - `start_date` (date)
  - `amount_paid` (money)
  - `payment_method` (enum)
  - `created_at` (auto)

### Package
- **Screen:** `/institutes` (selector)
- **Purpose:** Subscription pack per tenant.
- **Attributes:** `id`, `name`, `price`, `duration`, `feature_limits`, `status`

### Branch
- **Screen:** `/branches`
- **Purpose:** Branch under an institute.
- **Attributes:** `id`, `institute_id` (FK), `name`, `status`, `created_at`

### Custom Domain
- **Screen:** `/custom-domain`
- **Purpose:** Tenant custom-domain requests.
- **Attributes:** `id`, `institute_id` (FK), `domain` (string), `note`, `status` (enum: pending/verified), `submitted_at`

### Tenant General Setting
- **Screen:** `/administration/general_settings`
- **Purpose:** Single per-tenant settings record (saved via `POST /administration/general_settings/update`).
- **Attributes (by tab):**
  - General: `institute_name`, `site_title`, `tagline`, `phone`, `email`, `address`, `eiin`, `institute_id`, `copyright`
  - SEO: `meta_title`, `meta_description`, `meta_keywords`, `og_image_url`, `ga4_measurement_id`, `gtm_container_id`, `meta_pixel_id`
  - Fee/exam: `tuition_fee`, `exam_result_visible` (bool), `attendance_out_after` (int-device), `transfer_certificate_fee` (money), `live_notice_on_header` (bool)
  - Admission/app: `online_admission_status` (bool), `exam_results_display_status` (bool), `app_version`, `app_url`, `play_store_link`, `app_store_link`
  - Mail: `mail_type`, `from_email`, `from_name`, `smtp_host`, `smtp_port`, `smtp_username`, `smtp_password`, `smtp_encryption`
  - SMS: `sms_gateway`, `sms_test_mode` (bool), `api_key`, `sender_id`, `user_name`, `sender_name`, `twilio_sid`, `twilio_token`, `from_number`, `base_url`, `originator`
  - Social/map: `google_map`, `on_google_map` (bool), `facebook_link`, `google_plus_link`, `youtube_link`, `whatsapp_link`, `twitter_link`, `linkedin_link`
  - Zoom: `zoom_account_id`, `zoom_client_key`, `zoom_client_secret`
  - Google Meet: `google_meet_enabled` (bool), `client_id`, `client_secret`, `project_id`, `redirect_uri`, `service_account_json` (file/json), `calendar_id`, `default_duration` (int), `default_timezone`, `default_visibility`
  - Theme: `primary_color`, `secondary_color`, `text_color`, `sidebar_color`

### Institute Image Setting
- **Screen:** `/institute-image-settings`
- **Purpose:** Branding images per tenant (used in headers/footers/favicon).
- **Attributes:** `id`, `header_light` (file), `header_dark` (file), `footer_light` (file), `footer_dark` (file), `favicon` (file), `status`, `action`

### Payment Gateway
- **Screen:** `/payment-gateways`
- **Purpose:** Payment processor registry.
- **Attributes:** `id`, `gateway` (string), `mode` (enum: test/live), `credentials` (json), `status`

---

## 2. Students Information

### Student
- **Screen:** `/students`
- **Purpose:** Core student record.
- **Attributes:**
  - `id` (PK)
  - `image` (file)
  - `roll` (int)
  - `admission_number` (string)
  - `name` (string)
  - `class_id` (FK → Class)
  - `section_id` (FK → Section)
  - `group_id` (FK → StudentGroup)
  - `phone_no` / `g_mobile` (string)
  - `gender` (enum: male/female)
  - `status` (enum: active/inactive)
  - `created_at`

### Student Migration
- **Screens:** `/student-migration`, `/student-migration-pushback`, `/migrated-list`
- **Purpose:** Promote/lift students across sessions or revert.
- **Attributes:** `id`, `student_id` (FK), `from_class`, `from_section`, `to_class`, `to_section`, `migration_type` (enum), `year` (FK → AcademicYear), `group_id` (FK), `status`, `created_at`

### Migrated Student View (At-a-Glance)
- **Screen:** `/at-a-glance`
- **Purpose:** Flat read-model of all students.
- **Columns:** student_id, roll_no, admission_number, name, class, section, gender, g_mobile (seeded 100 rows)

---

## 3. Staffs Information

### Staff
- **Screen:** `/staffs`
- **Purpose:** Non-teaching staff.
- **Attributes:** `id`, `profile_image` (file), `name`, `phone`, `email`, `designation`, `department_id` (FK), `blood_group` (enum), `status`, `hr_id` (used in payroll screens)

### Teacher
- **Screens:** `/teachers`
- **Purpose:** Teaching staff (subtype of staff with academic role).
- **Attributes:** `id`, `profile_image`, `name`, `phone`, `email`, `department_id` (FK), `designation` (enum: Professor/Lecturer…), `blood_group`, `status`

### Staff Attendance
- **Screens:** `/staffs-attendance`
- **Purpose:** Daily staff attendance.
- **Attributes:** `id`, `role` (enum), `staff_id` (FK), `date`, `status` (enum: present/absent/late/leave), `check_in`, `check_out`

---

## 4. Student Attendance

### Student Attendance Record
- **Screens:** `/student-attendance`
- **Purpose:** Daily marking per class/section.
- **Attributes:** `id`, `class_id` (FK), `section_id` (FK), `student_id` (FK), `date`, `status` (enum: P/A/L/E), `marked_by` (FK → User), `created_at`

### Exam Attendance
- **Screen:** `/exams-attendance`
- **Purpose:** Attendance inside exams.
- **Attributes:** `id`, `exam_id` (FK), `class_id` (FK), `section_id` (FK), `subject_id` (FK), `student_id` (FK), `status`

### Exam Schedule
- **Screen:** `/exams-schedule`
- **Purpose:** Timetable of exam sessions.
- **Attributes:** `id`, `exam_id` (FK), `class_id` (FK), `group_id` (FK), `exam_start_at` (datetime), `subject_id` (FK), `duration`

### Attendance Report Summary
- **Screen:** `/reports-student_attendance_date_to_date` (→ `/view`)
- **Purpose:** Aggregated attendance (filter by class/section/range). View only: `student`, `present_days`, `absent_days`, `percentage`

### Absent Fine Report
- **Screen:** `/absent-fine-report`
- **Purpose:** Fines charged for absences. View: `student`, `days_absent`, `fine_amount`

---

## 5. QR Code Attendance

### QR Attendance Scanner Session
- **Screen:** `/qrattendance/scanner`
- **Purpose:** Camera/QR based punch-in.
- **Attributes:** `id`, `student_id` (FK), `date`, `time`, `device`, `status`

---

## 6. Academic Configuration

### Academic Year / Session
- **Screen:** `/academic-years`
- **Attributes:** `id`, `session_name` (string), `academic_year` (string/date), `is_current` (bool)

### Shift
- **Screen:** `/shift`
- **Attributes:** `id`, `name` (e.g. Morning/Day)

### Class
- **Screen:** `/class`
- **Attributes:** `id`, `name`, `numeric_value`, `status`

### Section
- **Screen:** `/sections`
- **Attributes:** `id`, `class_id` (FK, m2m classes), `section_name`, `group_id` (FK), `room_no`

### Student Group
- **Screen:** `/student-groups`
- **Attributes:** `id`, `group_name` (e.g. Science/Commerce/Arts)

### Period
- **Screen:** `/periods`
- **Attributes:** `id`, `period_name`, `start_time`, `end_time`, `serial`, `class_id` (FK)

### Subject
- **Screen:** `/subjects`
- **Attributes:** `id`, `subject_name`, `subject_code` (unique, e.g. BAN101), `class_id` (FK), `subject_type` (enum: compulsory/optional), `status`

### Subject Config
- **Screen:** `/subject-config/create`
- **Purpose:** Which subjects a class/group studies; merge groups of subjects.
- **Attributes:** `id`, `class_id` (FK), `group_id` (FK), `subjects[]` (m2m → Subject), `subject_serial`, `merge_id`, `subject_type`

### Optional Subject Config
- **Screen:** `/optional-subject-config`
- **Attributes:** `id`, `name`, `class_id` (FK), `group_id` (FK), `subjects[]` (m2m), `limit` (int)

### Student Optional Subject
- **Screen:** `/student-optional-subject`
- **Attributes:** `id`, `student_id` (FK), `subject_id` (FK), `config_id` (FK → OptionalSubjectConfig)

### Exam
- **Screen:** `/exams`
- **Attributes:** `id`, `name` (e.g. Half Yearly/Final), `exam_code`

### Student Category
- **Screen:** `/student-categories`
- **Attributes:** `id`, `student_category` (string, e.g. Regular/Talentpool)

### Department
- **Screen:** `/departments`
- **Attributes:** `id`, `department_name`, `priority`

### Picklist
- **Screen:** `/picklists`
- **Purpose:** Reusable dropdown option sets.
- **Attributes:** `id`, `type`, `value`, `slug`

### Signature
- **Screen:** `/signatures`
- **Purpose:** Authority signatures for documents.
- **Attributes:** `id`, `place_at` (enum: principal/class-teacher/parent…), `title`, `signature` (file)

---

## 7. Fees Management

### Fee Head
- **Screen:** `/fee-head`
- **Purpose:** Fee heads and sub-heads.
- **Attributes:** `id`, `fee_head_id` (FK, parent), `name`, `serial`, `type`

### Fee Mapping
- **Screen:** `/fees-mapping`
- **Purpose:** Link fee heads to accounting ledgers/funds.
- **Attributes:** `id`, `fee_head_id` (FK), `fee_sub_heads[]` (m2m), `ledger_id` (FK), `fund_id` (FK)

### Fee Amount Config
- **Screen:** `/amount-config`
- **Attributes:** `id`, `class_id` (FK), `group_id` (FK), `section_id` (FK), `student_category_id` (FK), `fee_head_id` (FK), `fee_amount` (money), `fine_amount` (money), `fund_id` (FK), `period` (string), `amount` (money)

### Fee Date Config
- **Screen:** `/date-config`
- **Attributes:** `id`, `academic_year_id` (FK), `fee_head_id` (FK), `fee_sub_head`, `fee_payable_date` (date), `fine_active_date` (date)

### Attendance / Fine Waiver
- **Screen:** `/attendance-waiver`
- **Attributes:** `id`, `class_id` (FK), `section_id` (FK), `student_id` (FK), `days_waived`

### Waiver
- **Screen:** `/waivers`
- **Attributes:** `id`, `waiver_name`

### Waiver Config
- **Screen:** `/waiver-config`
- **Purpose:** Per-student fee waivers.
- **Attributes:** `id`, `academic_year_id` (FK), `group_id` (FK), `class_id` (FK), `section_id` (FK), `student_category_id` (FK), `student_id` (FK), `roll`, `fee_head_id` (FK), `waiver_id` (FK → Waiver), `amount` (money)

### Fee Collection (Quick / Smart Collection)
- **Screen:** `/quick-collection`
- **Purpose:** Capture student fee payments; also exposes per-student paid status.
- **Attributes:** `id`, `student_id` (FK), `invoice_id` (string), `academic_year_id` (FK), `fee_head_id` (FK), `amount` (money), `fine` (money), `discount/waiver` (money), `paid_by` (enum), `payment_method_id` (FK → PaymentGateway), `fund_id` (FK → AccountingFund), `date`, `status`

### Payment Fee Info
- **Screen:** `/payment-fee-info`
- **Purpose:** Read-model of collected payments by filter.
- **Columns:** student, class, section, amount, discount, fine, paid_status, date

### Unpaid Fee Info
- **Screen:** `/unpaid-info`
- **Purpose:** Due ledger per student.
- **Columns:** sl, student, roll, due_details, total_due

---

## 8. Accounts Management

### Accounting Category
- **Screen:** `/accounting-categories`
- **Attributes:** `id`, `name`, `code`, `type` (enum)

### Accounting Group
- **Screen:** `/accounting-groups`
- **Attributes:** `id`, `account_category_id` (FK), `name`

### Accounting Ledger
- **Screen:** `/accounting-ledgers`
- **Attributes:** `id`, `ledger_name`, `account_group_id` (FK), `account_category_id` (FK), `nature` (enum: debit/credit)

### Accounting Fund
- **Screen:** `/accounting-funds`
- **Attributes:** `id`, `name`, `serial`, `opening_balance`, `amount_in` (derived), `amount_out` (derived), `balance` (derived)

### Cash Transaction (Payment/Receipt voucher)
- **Screens:** `/account-transaction-payment` (+ `?type=receipt`)
- **Purpose:** Payment (expense) & receipt (income) vouchers; receipt shares table with `type` discriminator.
- **Attributes:** `id`, `type` (enum: payment/receipt), `transaction_date`, `paid_by` / `receipt_type`, `fund_id` (FK), `reference`, `description`, `lines[]` (ledger_ids[] + amounts[], m2m)

### Contra Transfer
- **Screen:** `/account-contra-transfers`
- **Purpose:** Movement between internal accounts.
- **Attributes:** `id`, `transfer_date`, `transfer_from` (FK → Ledger/Fund), `transfer_to` (FK), `amount`, `reference`, `description`

### Journal Transaction
- **Screen:** `/journal-transactions`
- **Attributes:** `id`, `journal_date`, `fund_id` (FK), `description`, `reference`, `lines[]` (debit[] / credit[])

### Fund Transfer
- **Screen:** `/account-fund-transfers`
- **Attributes:** `id`, `payment_date`, `transfer_from` (FK → Fund), `transfer_to` (FK), `amount`, `description`

### Chart of Accounts
- **Screen:** `/chart-of-accounts`
- **Purpose:** Read-model tree; seeded parents:
  - 1. Cash & Cash Equivalence → Assets
  - 2. Current Liabilities → Liabilities
  - 3. Non-Current Liabilities → Liabilities
  - 4. Owner's Equity → Liabilities
  - 5. Fees Related Income → Income
  - 6. Others Income → Income
  - 7. General Expenses → Expense

---

## 9. Accounting Reports

> All are read-only (view) entities — no stored attributes beyond filters.

| Entity (report) | Screen | Filters |
|---|---|---|
| BalanceSheet | `/balance-sheet` | from, to |
| TrialBalance | `/trial-balance` | from, to |
| CashFlowStatement | `/cash-flow-statement` | year |
| CashFlowDetail | `/cash-flow-details` | year |
| CashBook | `/cash-book-account` | from_date, to_date, payment_method |
| LedgerBook | `/ledger-book-account` | from_date, to_date, payment_method |
| IncomeStatement | `/income-statement` | ledger_income_list, income_list, ledger_expense_list, expense_list, profite/loss |
| IncomeStatementDetail | `/income-statement-details` | from, to |
| CashSummary | `/cash-summary` | ledger_income_list, income_list, ledger_expense_list, expense_list |

---

## 10. Payroll Management

### Salary Head
- **Screen:** `/salary-heads`
- **Attributes:** `id`, `salary_head` (string: Basic, Allowance, Welfare Fund, Professional Tax…), `nature` (enum: plus/minus)

### Payroll Mapping
- **Screen:** `/payroll-mapping`
- **Purpose:** Map payroll account to ledger/fund.
- **Attributes:** `id`, `ledger_id` (FK), `fund_id` (FK)

### Payroll Assign
- **Screen:** `/payroll-assign`
- **Purpose:** Salary structure per staff.
- **Attributes:** `id`, `staff_id` (FK), `effective_date`, cell columns:
  - `net_salary` (money)
  - `md` (money, +)
  - `basic` (money, +)
  - `allowance` (money, +)
  - `early_leave_fine` (money, −)
  - `festival_allowance` (money, +)
  - `welfare_fund` (money, −)
  - `professional_tax` (money, −)
  - `conveyance` (money, +)
  - `exam_hall_duty` (money, +)
  - `incentive` (money, +)
  - `medical` (money, +)

### Salary Slip
- **Screen:** `/salary-create`
- **Purpose:** Generated monthly pay slips per staff/period.

### Salary Payment
- **Screens:** `/salary-payment-process`, `/payment-info`
- **Attributes:** `id`, `staff_id` (FK), `hr_id`, `invoice_id`, `month/period`, `net_salary`, `payable_salary`, `paid`, `due`, `advance`, `payment_type`, `paid_status`, `payment_date`

### Due Salary Payment
- **Screen:** `/due-salary-payment`
- **Attributes:** `id`, `hr_id`, `month`, `due_amount`, `paid`, `status`

### Advance Salary Payment
- **Screen:** `/advance-salary-payment`
- **Attributes:** `id`, `staff_id` (FK), `hr_id`, `amount`, `date`, `installments`, `remaining`

### Return Advance Payment
- **Screen:** `/return-salary-payment`
- **Purpose:** Track advances recovered from salary.
- **Attributes:** `id`, `advance_id` (FK), `amount`, `date`

### Salary Statement
- **Screen:** `/salary-statement`
- **Purpose:** Monthly aggregate report. View only: staff, month, earnings, deductions, net, paid, due.

---

## 11. Routine Management

### Syllabus
- **Screen:** `/syllabus`
- **Attributes:** `id`, `title`, `description`, `class_id` (FK), `subject_id` (FK), `file` (file), `status`

### Assignment
- **Screen:** `/assignments`
- **Attributes:** `id`, `title`, `description`, `class_id` (FK), `section_id` (FK), `subject_id` (FK), `file`, `due_date`

### Class Routine
- **Screen:** `/class_routines`
- **Purpose:** Weekly timetable grid.
- **Attributes:** `id`, `class_id` (FK), `section_id` (FK), `day` (enum), `period_id` (FK), `subject_id` (FK), `teacher_id` (FK), `room`

### Exam Routine
- **Screen:** `/exam-routines`
- **Attributes:** `id`, `exam_id` (FK), `class_id` (FK), `group_id` (FK), `subject_id` (FK), `date`, `start_time`, `end_time`, `room` (seat plan areas)

### Admit & Seat Plan
- **Screen:** `/exam-essentials`
- **Purpose:** Generated admit cards / seating plans (class, section, roll range).

---

## 12. Library Management

### Book Category
- **Screen:** `/book-categories`
- **Attributes:** `id`, `category_name`

### Book
- **Screen:** `/books`
- **Attributes:** `id`, `book_name`, `code` / `barcode`, `author`, `category_id` (FK), `quantity`, `available`, `rack/shelf`, `status`

### Library Member
- **Screen:** `/librarymembers`
- **Attributes:** `id`, `library_member_id`, `name`, `image`, `member_type` (enum: student/teacher/staff), `user_id` (FK)

### Book Issue
- **Screens:** `/books-ber-code-page` (issue UI), `/bookissues` (report)
- **Attributes:** `id`, `book_id` (FK), `member_id` (FK → LibraryMember), `library_id`, `issue_date`, `return_date`, `returned` (bool), `status`

### Barcode Print Job
- **Screen:** `/books-ber-code-print`
- **Attributes:** `book_code` set → printed labels (transient)

---

## 13. Exam Module

### Exam Startup
- **Screen:** `/semester-exam-settings-exam-startup`
- **Purpose:** Compose exams for a class from global code/grade lists.
- **Attributes:** `id`, `class_id` (FK), `global_exam_code_list_id` (FK→ExamCode), `global_exam_grade_list_id` (FK→Grade), `exam_id` (FK), `merit_process_type` (enum)

### Exam Code List
- **Screen:** `/semester-exam-settings-exam-startup`
- **Attributes:** `id`, `code_title`, `total_marks`, `pass_mark`, `acceptance`

### Grade List
- **Screen:** `/semester-exam-settings-exam-startup`
- **Attributes:** `id`, `grade`, `grade_range` (from–to), `point`

### Exam Mark Config
- **Screen:** `/semester-exam-settings-mark-config`
- **Purpose:** Weight of each exam in grand-final; per class.
- **Attributes:** `id`, `class_id` (FK), `group_id` (FK), `calculation_method` (enum), `exam_id` (FK), `exam_name`, `percentage` (percent), `exam_serial`

### Exam Remark
- **Screen:** `/remarks-config`
- **Attributes:** `id`, `remark_title`, `remark_text`

### Exam Mark (Mark Input)
- **Screen:** `/mark-input-section-wise`
- **Purpose:** Subject-wise mark entry per class/section.
- **Attributes:** `id`, `exam_id` (FK), `class_id` (FK), `section_id` (FK), `subject_id` (FK), `student_id` (FK), `written`, `mcq`, `practical`, `total`, `grade`, `point`

### Exam Result
- **Screen:** `/exam-result-view`
- **Purpose:** Computed results (view) + notification dispatch (send scope/classes/notify-via).

### Grand Final Result
- **Screen:** `/grand-final-result`
- **Purpose:** Blended multi-exam result per student (class, section, roll range).

### Tabulation / Broad Sheet
- **Screen:** `/tabulation-sheet`
- **Attributes (view):** class, section, exam/term, sheet_type
- **Columns:** student, roll, per-subject marks, total, gpa, grade, position

### Merit List Sheet
- **Screen:** `/merit-list-sheet`
- **Purpose:** Ranked merit list per class/section/exam.

### Result Card Setting
- **Screen:** `/result-card-settings`
- **Attributes:**
  - `result_title`
  - toggles: `show_photo`, `show_position`, `show_gpa`, `show_percentage`, `show_failed_subjects`, `show_attendance`, `show_remarks`, `show_cognitive_domain`, `show_affective_domain`, `show_psychomotor_domain`, `show_teacher_signature`, `show_principal_signature`, `show_parent_signature`
  - labels: `class_teacher_label`, `principal_label`, `parent_label`
  - signatures: `class_teacher_signature` (file), `principal_signature` (file)
  - `failed_student_ranking` (enum)
  - `primary_color`, `accent_color`, `watermark_opacity` (0–1), `show_institute_banner_watermark` (bool)

### Empty Mark Sheet
- **Screen:** `/empty-mark-sheet`
- **Purpose:** Printable blank mark sheets (class, section, exam, optional count).

### Assessment Domain
- **Screen:** `/assessment-domains`
- **Purpose:** Co-curricular skill domains.
- **Attributes:** `id`, `domain` (enum: cognitive/affective/psychomotor), `name`, `display_order`
- **Assessment Domain Item:** `id`, `domain_id` (FK), `item`, `max` (int), `type`

### Domain Assessment Entry
- **Screen:** `/assessment-entry`
- **Attributes:** `id`, `class_id` (FK), `exam_id` (FK), `section_id` (FK), `student_id` (FK), `domain_item_id` (FK), `score`, `grade`

### Online Exam
- **Screen:** `/online-exams`
- **Attributes:** `id`, `name`, `exam_id` (FK), `class_id` (FK), `subject_id` (FK), `question_bank` (json), `window` (timed), `start_at`, `end_at`, `status`

---

## 14. Layout & Certificates

### Certificate Template
- **Screen:** `/certificate-templates`
- **Attributes:** `id`, `name`, `certificate_type` (enum, see layouts), `orientation` (enum: portrait/landscape), `colors`, `background` (file), `fields` (json of placeholders), `status`

### Certificate Layout (10 types)
- **Screen:** `/layout-cert?type={slug}` (shared entity, discriminated by `type`)
- **Types:** `general-certificate`, `testimonial`, `attendance-certificate`, `hsc-recommendation`, `transfer-certificate`, `abroad-recommendation`, `character-certificate`, `study-certificate`, `bonafide-certificate`, `migration-certificate`
- **Attributes (request):** class, section, student, certificate fields, layout preview → PDF print.

---

## 15. SMS Module

### SMS Template
- **Screen:** `/sms-template`
- **Attributes:** `id`, `title`, `message`

### Phone Book Category
- **Screen:** `/phone-book-category`
- **Attributes:** `id`, `category_title`, `description`

### Phone Book Contact
- **Screen:** `/phone-book`
- **Attributes:** `id`, `name`, `phone`, `category_id` (FK → PhoneBookCategory), `class_id` (FK), `section_id` (FK), `note`
- Sync: `POST /phone-book-sync`

### SMS Send (Compose)
- **Screen:** `/sms-compose`
- **Attributes:** `id`, `class_id` (FK), `section_id` (FK), `number` (or from phone-book), `template_id` (FK), `message` (≤300), `gateway`, `sent_at`, `status`
- **SMS Sent Log:** `id`, `to`, `message`, `gateway_response`, `status`, `sent_at`

### SMS Purchase
- **Screen:** `/sms-purchase`
- **Attributes:** `id`, `sms_gateway` (FK), `quantity`, `price`, `transaction_date`, `masking_type` (enum: masking/non-masking), `status`

### SMS Report
- **Screen:** `/send-sms-report`
- **Purpose:** Sent-vs-delivered aggregate (from/to).

---

## 16. Administrator

### Shift Assignment
- **Screen:** `/assign-shifts`
- **Attributes:** `id`, `teacher_id` (FK), `shift_id` (FK)

### Subject Assignment
- **Screen:** `/assign-subjects`
- **Attributes:** `id`, `class_id` (FK), `section_id` (FK), `subject_id` (FK), `teacher_id` (FK)

### Class Assignment
- **Screen:** `/assign-class`
- **Attributes:** `id`, `teacher_id` (FK), `class_id` (FK)

### Notice
- **Screen:** `/notices`
- **Attributes:** `id`, `image` (file), `title`, `user_type` (enum audience), `notice` (rich text), `status`, `created_at`

### Event
- **Screen:** `/events`
- **Attributes:** `id`, `image`, `name`, `location`, `start_date`, `end_date`, `description`, `status` (public: `/event-details/{id}`)

### Contact Message
- **Screen:** `/contact-message`
- **Purpose:** Submissions from the public contact form.
- **Attributes:** `id`, `name`, `phone`, `email`, `message`, `date`, `status` (enum: new/replied/closed)

### User Activity Log
- **Screen:** `/user-logs`
- **Attributes:** `id`, `user_id` (FK), `name`, `ip_address`, `action`, `detail`, `created_at`, `updated_at`

### Student ID Card Batch
- **Screen:** `/student-id-cards`
- **Purpose:** Bulk ID card generation.
- **Attributes:** `class_id` (FK), `group_id` (FK), `section_id` (FK), `card_validity_date`, `layout`, output PDF

### Teacher ID Card Batch
- **Screen:** `/teacher-id-cards`
- **Attributes:** `department_id` (FK), `card_validity_date`, output

### Staff ID Card Batch
- **Screen:** `/staff-id-cards`
- **Attributes:** `department_id` (FK), `card_validity_date`, output

---

## 17. System

### System Information
- **Screen:** `/system/information`
- **Purpose:** Env/server detail panel (read-only).

### Module Registry
- **Screen:** `/system/modules`
- **Purpose:** Installable/enablable feature modules.
- **Attributes:** `id`, `module`, `version`, `status`, `dependencies`, `actions`

### System Update
- **Screen:** `/system/update`
- **Attributes:** `id`, `version`, `archive` (file), `applied_at`, `status`

### Update History
- **Screen:** `/system/update/history`
- **Attributes:** `version`, `date`, `type`, `status`, `admin` (FK → User), `duration`

### System Setting (SaaS)
- **Screen:** `/system/settings`
- **Attributes:** `update_server_url`, `demo_reset_interval_hours`, `enable_remote_update_check` (bool), `enable_demo_auto_reset` (bool)

---

## 18. Master Configuration

### Role
- **Screen:** `/roles`
- **Attributes:** `id`, `role_name`, `permissions` (json/m2m)

### User
- **Screens:** `/users`, `/profile`
- **Attributes:** `id`, `profile_image`, `name`, `email`, `phone`, `password` (hashed), `user_type` (enum: admin/accountant/librarian/teacher/student/staff), `role_id` (FK), `institute_id` (FK), `is_active`, `last_login`

### Tenant Admin (superadmin) extension
- **Provided via seeded demo users:** superadmin, accountant, librarian, teacher1, student

---

## 19. CMS Management

### Admission Application
- **Screen:** `/admission-forms`
- **Purpose:** Online admission requests.
- **Attributes:** `id`, `student_name`, `class_id` (FK), `contact`/`phone`, `guardian`, `email`, `address`, `documents`, `status` (enum: pending/approved/rejected), `date`

### CMS Page
- **Screen:** `/pages` → public `/page/{slug}`
- **Attributes:** `id`, `title`, `slug`, `content` (rich text), `seo_meta`, `status`

### Banner
- **Screen:** `/banners`
- **Attributes:** `id`, `title`, `image`, `button_name`, `button_link`, `description`, `serial`, `status`

### About Us Item
- **Screen:** `/about-us`
- **Attributes:** `id`, `title`, `image`, `short_description`, `status`

### FAQ
- **Screen:** `/faqs`
- **Attributes:** `id`, `question`, `answer`, `status`, `serial`

### Gallery Image
- **Screen:** `/gallery-images` → public `/academic-images`
- **Attributes:** `id`, `title`, `heading`, `image`, `status`

### Mobile App Section
- **Screen:** `/mobile-app-sections`
- **Attributes:** `id`, `title`, `heading`, `image`, `features` (list), `store_links` (Android/iOS), `status`

### Why Choose Us Item
- **Screen:** `/why-choose-us`
- **Attributes:** `id`, `title`, `icon`, `description`, `serial`

### Policy
- **Screen:** `/policies` → public `/privacy-policy`, `/terms-conditions`, `/refund-policy`, `/cookies-policy`
- **Attributes:** `id`, `type` (enum: privacy/terms/refund/cookies), `status`, `description` (rich text)

### Ready To Join Us Item
- **Screen:** `/ready-to-join-us`
- **Attributes:** `id`, `icon`, `title`, `description`, `button`, `status`

### Testimonial
- **Screen:** `/testimonials`
- **Attributes:** `id`, `image`, `name`, `designation`, `review`, `rating` (1–5), `status`

### Achievement Counter (static on homepage)
- **Screens:** homepage section; stored under content settings.
- **Values seeded:** expert teachers 4+, total students 102+, school events 3+, happy reviews 4+.

---

## 20. WhatsApp

### WhatsApp Setting
- **Screen:** `/whats-app-settings`
- **Attributes:** `id`, `provider`, `phone_id`, `business_id`, `access_key`, `language`, `status`

### WhatsApp Template
- **Screen:** `/whats-app-templates`
- **Attributes:** `id`, `name`, `event` (enum: exam/admission/fee…), `message`, `status`

### WhatsApp Log
- **Screen:** `/whats-app-logs`
- **Attributes:** `id`, `student_id` (FK), `phone`, `message`, `retries` (int), `sent_at`, `status`

---

## 21. Hostel Management

### Hostel
- **Screen:** `/hostel-categories`
- **Attributes:** `id`, `name`, `hostel_type` (enum), `address`, `status`

### Hostel Category
- **Screen:** `/hostel-categories`
- **Attributes:** `id`, `hostel_id` (FK), `standard` (string/class level), `fee` (money), `note`

### Hostel Member
- **Screen:** `/hostel-members`
- **Attributes:** `id`, `student_id` (FK), `class_id` (FK), `hostel_id` (FK), `category_id` (FK → HostelCategory), `fee` (money), `status`

### Hostel Building
- **Screen:** `/hostel-buildings`
- **Attributes:** `id`, `hostel_id` (FK), `building_name`

### Hostel Floor
- **Screen:** `/hostel-floors`
- **Attributes:** `id`, `building_id` (FK), `floor_name`

### Hostel Room
- **Screen:** `/rooms`
- **Attributes:** `id`, `room_number`, `hostel_category_id` (FK), `floor_id` (FK), `capacity`, `occupied` (derived)

### Bed
- **Screen:** `/beds`
- **Attributes:** `id`, `room_id` (FK), `bed_number`, `status` (enum: free/assigned/maintenance)

### Room Member
- **Screen:** `/room-members`
- **Attributes:** `id`, `student_id` (FK), `phone`, `room_id` (FK), `bed_id` (FK), `hostel_category_id` (FK), `assigned_at`

### Meal
- **Screen:** `/meals`
- **Attributes:** `id`, `meal_name`, `type` (enum: breakfast/lunch/dinner), `created_at`

### Meal Plan
- **Screen:** `/meal-plans`
- **Attributes:** `id`, `student_id` (FK), `meal_id` (FK), `date`

### Meal Entry
- **Screen:** `/meal-entries`
- **Attributes:** `id`, `student_id` (FK), `meal_id` (FK), `date`, `price` (money)

### Hostel Bill
- **Screen:** `/hostel-bills`
- **Attributes:** `id`, `student_id` (FK), `hostel_fee` (money), `meal_fee` (money), `total_amount` (derived), `due_date`, `period` (month), `status`

### Hostel Leave
- **Screen:** `/hostel-leaves`
- **Attributes:** `id`, `student_id` (FK), `room_id` (FK), `from`, `to`, `reason`, `status` (enum: pending/approved/rejected)

### Hostel Collection
- **Screen:** `/hostel-collections`
- **Attributes:** `id`, `invoice`, `student_id` (FK), `room_id` (FK), `bed_id` (FK), `month`, `this_month` (money), `all_outstanding_due` (money), `paid` (money), `due` (money), `date`

### Hostel Seat Map
- **Screen:** `/hostel-seat-map`
- **Purpose:** Visual bed allocation view (read-only aggregate).

---

## 22. Inventory

### Inventory Category
- **Screen:** `/inventory-categories`
- **Attributes:** `id`, `name`, `note`

### Inventory Item
- **Screen:** `/inventory-items`
- **Attributes:** `id`, `name`, `category_id` (FK → InventoryCategory), `sku`, `cost_price` (money), `selling_price` (money), `stock` (int), `status`

### Inventory Sale
- **Screen:** `/inventory-sales`
- **Attributes:** `id`, `invoice`, `student_id` (FK), `date`, `payable` (money), `paid` (money), `due` (derived)
- **Inventory Sale Line:** `id`, `sale_id` (FK), `item_id` (FK), `qty`, `unit_price`, `total`

---

## 23. Transport Management

### Bus
- **Screen:** `/buses`
- **Attributes:** `id`, `bus_number`, `model`, `capacity`, `registration`, `driver_id` (FK), `status`

### Driver
- **Screen:** `/drivers`
- **Attributes:** `id`, `name`, `phone`, `license_no`, `assigned_bus_id` (FK → Bus), `status`

### Bus Route
- **Screen:** `/bus-routes`
- **Attributes:** `id`, `route_name`, `start_location`, `end_location`, `distance`, `estimated_time`, `status`

### Bus Stop
- **Screen:** `/bus-stops`
- **Attributes:** `id`, `stop_name`, `route_id` (FK → BusRoute), `latitude`, `longitude`, `order`

### Transport Member
- **Screen:** `/transport-members`
- **Attributes:** `id`, `student_id` (FK), `route_id` (FK), `stop_id` (FK), `fare` (money), `status`

### Vehicle Type
- **Screen:** `/vehicle-types`
- **Attributes:** `id`, `name`, `status`

### Vehicle Category
- **Screen:** `/vehicle-categories`
- **Attributes:** `id`, `name`, `status`

### Transport Helper
- **Screen:** `/helpers`
- **Attributes:** `id`, `name`, `phone`, `assigned_bus_id` (FK), `status`

### Transport Collection
- **Screen:** `/transport-collections` (+ report `/transport-reports/collection`)
- **Attributes:** `id`, `invoice`, `student_id` (FK), `route_id` (FK), `stop_id` (FK), `month`, `fare`, `this_month` (money), `paid`, `due`, `date`

### Transport Dashboard
- **Screen:** `/transport-dashboard`
- **Purpose:** KPI read-model (buses, routes, members, collections).

---

## 24. Google Meet

### Google Meet Session
- **Screens:** `/google-meet`, `/google-meet/create`
- **Attributes:**
  - `title`
  - `teacher_id` (FK)
  - `class_id` (FK), `section_id` (FK), `group_id` (FK), `subject_id` (FK)
  - `description`
  - `start_date`, `start_time`, `end_time`, `duration` (int min)
  - `visibility` (enum: everyone/class/private)
  - `enable` (bool)
  - recipients: `students[]`, `guardians[]` (m2m notify)
  - `recurring`: `recurring` (bool), `repeat` (enum), `until` (date)
  - `meet_link`, `status`
  - `notes/attachments`

### Google Calendar OAuth/Settings
- **Screen:** `/administration/general_settings` (Google Meet tab) + `POST /google-meet/test-connection`
- **Stored:** client credentials, service account JSON, calendar id (see tenant settings entity)

---

## 25. AI Assistant

### AI Assistant Setting
- **Screen:** `/ai/settings`
- **Attributes:** `id`, `anthropic_api_key`, `model` (default), `enabled`

### AI Chat Session / Message
- **Screen:** `/ai/chat`
- **Purpose:** Conversational assistant over institute data.
- **Attributes (session):** `id`, `user_id` (FK), `thread`, `created_at`
- **Attributes (message):** `id`, `session_id` (FK), `role` (enum: user/assistant), `content`, `created_at`

### AI Content Writer Output
- **Screen:** `/ai/writer`
- **Purpose:** Generated text by type.
- **Attributes (request):** `content_type`, `prompt`, `result` (transient unless saved)

### AI Data Insights
- **Screen:** `/ai/insights`
- **Purpose:** NL queries over reports.
- **Attributes (request):** `report`, `class_id` (optional), `from`, `to`, `question`, `summary`

---

## 26. Auth, Profile & Public Forms

### Authenticated Session
- **Endpoints:** `POST /login`, `POST /logout`
- **Mechanics:** Laravel cookie session + CSRF `_token`.

### User Profile
- **Screen:** `/profile`
- **Editable:** `name` (Full Name), `email`, `photo`, `password`, `password_confirmation` (demo-mode locked on this tenant)

### Public Contact Submission
- **Endpoint:** `POST /contact-submit`
- **Attributes:** `id`, `name`, `email`, `phone`, `message`, `created_at`, `status` (same record surfaced in admin `/contact-message`)

### Public Event (frontend mirror)
- **Route:** `/event-details/{id}`
- **Source entity:** Event (admin)


## 27. Cross-cutting / shared entities

| Entity | Used by | Notes |
|---|---|---|
| `AcademicYear` | Fees, Waivers, Migrations, Exams | current-year flag |
| `Class` / `Section` / `StudentGroup` | nearly every domain | cascading via `/sections-section-group-wise`, `/groups-class-section-wise` |
| `User` (+Rolable) | Auth, logs, notices audience | single sign-in for all roles |
| `Role` + `Permission` | RBAC | `/roles` |
| `Ledger`/`Fund` | Fees mapping, payroll mapping, accounts | double-entry backbone |
| `PaymentGateway` | Fee, hostel, transport, payroll collections | mode test/live |
| `SmsGateway` | SMS module | per-gateway creds |
| `AiSetting` | AI module | Anthropic key (secret) |

---

*End of entity inventory. ~190 entities inferred behaviorally from the UI; no database access performed. Read-only audit — nothing changed on the target.*