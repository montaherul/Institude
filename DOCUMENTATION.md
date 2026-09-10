# Full Website Documentation — `https://institute.bdboibazer.com/`

> **Audit type:** read-only. **No data was created, modified, or deleted** on the target. All inspection was performed with authenticated `GET` requests after logging in as the demo superadmin.
>
> **Doc date:** 2026-09-11

---

## Table of Contents

1. [Overview](#1-overview)
2. [Product Identity & Branding](#2-product-identity--branding)
3. [Tech Stack & Architecture](#3-tech-stack--architecture)
4. [Access & Demo Credentials](#4-access--demo-credentials)
5. [Public-Facing Website](#5-public-facing-website)
6. [Admin Panel (Mighty School)](#6-admin-panel-mighty-school)
   - [Dashboard](#61-dashboard)
   - [Roles & Access Control](#62-roles--access-control)
   - [Module-by-module breakdown](#63-module-by-module-breakdown)
7. [Integrations](#7-integrations)
8. [Multi-Tenancy, Billing & Deployment](#8-multi-tenancy-billing--deployment)
9. [System Operations](#9-system-operations)
10. [Audit Summary & Security Observations](#10-audit-summary--security-observations)
11. [Master Route / Page Index](#11-master-route--page-index)

---

## 1. Overview

| Property | Value |
|---|---|
| Canonical URL | `https://institute.bdboibazer.com/` |
| Platform / product | **Mighty School** — a white-label SaaS institute/LMS management suite |
| Vendor | FueDevs LTD (footer: "© Copyright 2025. All Rights Reserved by FueDevs LTD") |
| Instance type | **Demo / sandbox** instance (public demo credentials; demo-mode restrictions active) |
| Public brand on the site | "Fudevs School" (banner hero) / "Demo Title" (default placeholders) |
| Hosting / edge | Cloudflare (WAF, managed `robots.txt` content-signals, email obfuscation, beacon analytics) |
| Language | English UI + installed Bangla language pack (`/language/bn`) |

The site is a **multi-tenant school management system**: the same code powers "schools, colleges, and training centers" (per footer tagline). What you browse at the frontend is a showcase website; the real product is the authenticated admin back office accessed via `/login`.

**Scope of what exists:**

- Public marketing/corporate front office (home, gallery, events, academics CMS pages, contact, legal pages).
- A large admin back office with **~25 sidebar module groups** containing **~180 distinct screens** (setup/config CRUD, attendance, fees, accounts, payroll, exams, library, hostel, transport, inventory, certificates, SMS/WhatsApp, Google Meet, AI, CMS, multi-tenancy, system maintenance).

---

## 2. Product Identity & Branding

| Element | Observed value |
|---|---|
| Site title (footer tagline) | "Demo Title — Modern SaaS-based institute management system for schools, colleges, and training centers." |
| Brand mark | `storage/institute_image_settings/2026-07-08-6a4ddd012d929.png` (header + footer logo) |
| Hero headline | "Learn Today, Lead Tomorrow" |
| Secondary banner | "Fudevs School - Excellence in Education, Brighter Futures Ahead!" |
| Contact | Phone `+123456789`, email (Cloudflare-obfuscated), address "2/A NewYork, USA" |
| Social links | Facebook, YouTube (external placeholders) |
| Admin app name | **Mighty School** (`<title>Mighty School</title>` across admin screens) |
| Copyright | Footer `© 2025 FueDevs LTD`; admin footer `© 2026 Mighty School` |

The demo content is **unfinished/placeholder-heavy** (Academics pages say "content goes here...", FAQ repeated 3× with identical Q&A, duplicated "About" section title "About Us" appears twice on the homepage, "What is an academic program?" repeated). Brand defaults were not customized, confirming it is a stock demo tenant.

---

## 3. Tech Stack & Architecture

Inferred from page source, headers, assets and endpoints:

| Layer | Evidence |
|---|---|
| Backend framework | **Laravel** (Blade templates, `_token` CSRF fields, `_method` HTTP method spoofing, `POST form action=".../route/1"` + `DELETE`, sessions) |
| Frontend (public) | Custom **Tailwind CSS** layout, Alpine.js, custom scrollbar/gradients, lightbox gallery |
| Admin frontend | **Metronic-style theme (v8 "keen"/"kt_" classes)**: `app-sidebar`, `menu-accordion`, `menu-sub-accordion`, drawer, toggle + jQuery 3.7.1, Alpine.js, **Select2**, **DataTables** (`.dataTables_info`, `dataTables` plugin JS), Font Awesome |
| Fonts/styling | Inter (local `assets/vendor/fonts/inter-local.css`), `assets/css/tailwind-app.css`, `assets/plugins/font-awesome/...` |
| AJAX helpers | Class→Section→Group cascading selects hitting `/sections-section-group-wise` and `/groups-class-section-wise` |
| Storage | Local disk `/storage/...` (banners, users, events, testimonials, institute settings images) |
| Email/SMS/WhatsApp | Configurable gateways (SMTP, SMS gateway + Twilio, WhatsApp Cloud API providers) |
| AI | Admin "AI Assistant" feature (chat / content writer / data insights) configured with **Anthropic API key** |
| Edge | Cloudflare (managed robots content-signal block on GPTBot, ClaudeBot, CCBot, etc.; JS beacon; email decode) |
| Auth | Session-cookie + CSRF; single `/login` for all roles; role-based middleware |
| Databases implied | Relational (likely MySQL) based on models mirrored by screen set (students, fee collections, ledger accounting, payroll, library, hostel…) |

No source code is accessible from the outside, so this documentation is based on **behavioral analysis** of the running site.

---

## 4. Access & Demo Credentials

These credentials are **publically embedded** on the login page itself (`/login` page JavaScript `demoUsers` object) — documenting them introduces no new risk.

| Role | Email | Password | Icon on login |
|---|---|---|---|
| Super Admin | `superadmin@gmail.com` | `12345678` | Admin (user-shield) |
| Accountant | `accountant@gmail.com` | `12345678` | Accountant (calculator) |
| Librarian | `librarian@gmail.com` | `12345678` | Librarian (book) |
| Teacher | `teacher1@gmail.com` | `12345678` | Teacher (black-tie) |
| Student | `student@gmail.com` | `12345678` | Student (graduation cap) |

**Login flow:** `GET /login` → parse CSRF `_token` → `POST /login` with `email`, `password`, `remember` → redirect to `/dashboard`. Logout via `POST /logout` (CSRF protected).

**Demo-mode restrictions observed** (banner in dashboard):
> "Demo Mode: Password, email, profile, system settings, and system upgrades cannot be changed."

i.e. attempts to alter password/email/profile/system settings/upgrades are expected to fail in this environment.

---

## 5. Public-Facing Website

### 5.1 Navigation structure

Top bar: Logo · Menu (☰) · Phone `+123456789` · **Login** button.

Primary nav:
- **Home** `/`
- **About** (anchor `#about` on homepage)
- **Gallery** `/academic-images`
- **Academics** (dropdown, 9 static CMS pages):
  - Approach → `/page/approach`
  - Career Counselling → `/page/career-counselling`
  - Facilities → `/page/facilities`
  - Houses & Mentoring → `/page/houses-mentoring`
  - Mission → `/page/mission`
  - Principal Message → `/page/principal-message`
  - School Uniform → `/page/school-uniform`
  - Student Council → `/page/student-council`
  - Vision → `/page/vision`
- **Events** (anchor `#events` on homepage)
- **FAQ** (anchor `#faq` on homepage)
- **Contact** `/contact`

Footer: Quick Links (Home, Galleries), Legal (Privacy Policy `/privacy-policy`, Terms & Conditions `/terms-conditions`, Refund Policy `/refund-policy`, Cookies Policy `/cookies-policy`).

### 5.2 Homepage sections (`/`)

1. **Hero/rotating banner carousel** — two banners (stored under `/storage/banners/`):
   - "Learn Today, Lead Tomorrow" — CTA "Get Admission Today" (`#`)
   - "Fudevs School — Excellence in Education, Brighter Futures Ahead!" — CTA "Enroll Now" (`#`)
2. **Why Choose Us** — 4 cards (image + title + text): Personalized Education, Experienced Faculty, State-of-the-Art Facilities, Holistic Development.
3. **About Us** — "Welcome to Fudevs School" intro + image.
4. **School Achievements / counter strip** — "Trusted By Thousands": Expert Teachers 4+, Total Students 102+, School Events 3+, Happy Reviews 4+.
5. **Our Faculty — Teachers**:
   - Dakota Love — Director of administration and student guidance
   - Nash Santana — Professor
   - Sanjay — Lecturer (has avatar `/storage/users/2026-08-24-6a8ca60d3ca80.png`)
   - hamza — Lecturer
6. **Our Faculty — Staffs**: Luk unami — Department Head.
7. **Upcoming Activities / Events** (3 seeded events, see 5.4).
8. **Testimonials** carousel:
   - Principal — "Our institute focuses on quality education and student development." ★★★★★
   - Vice Principal — "We maintain discipline and academic excellence for all students." ★★★★★
   - Teacher — "Teaching here is a wonderful experience with motivated students." ★★★★☆
   - Student Parent — "My child has improved a lot after joining this institute." ★★★★★
9. **FAQ** accordion — single demo Q&A "What is an academic program?" repeated 3 times.
10. Footer.

### 5.3 Static / CMS pages

- **Academic gallery** `/academic-images` — "Explore school activities, events & memories"; image lightbox. Seeded images point at `storage/institute_image_settings/...` (currently returns the logo placeholder).
- **Academics pages** `/page/{slug}` — generic CMS "pages" module; all seeded with placeholder body "… content goes here...". Admin-managed (Pages module).
- **Legal pages** — placeholders:
  - Privacy Policy: LMS privacy statement blurb.
  - Terms & Conditions: usage/disputes blurb.
  - Refund Policy: only heading "Refund".
  - Cookies Policy: analytics/personalization blurb.
- **Event detail** `/event-details/{id}` — featured image, date, location, description (see events data).

### 5.4 Public forms & endpoints

| Form | Method | Action endpoint | Fields / behavior |
|---|---|---|---|
| Login | POST | `/login` | `_token`, `email`, `password`, `remember` |
| Contact us | POST | `/contact-submit` | `_token`, `name`, `email`, `phone`, `message` → stored, visible in admin **Contact Message** module |
| (CTA) Get Admission Today / Enroll Now | – | `#` | Links are dead anchors in demo |
| Gallery | – | lightbox only | no form |

### 5.5 Seeded public content inventory

| Entity | Records |
|---|---|
| Banners | 2 (slide 1 "Learn Today…", slide 2 "Fudevs School…") |
| Events | 3: **Annual Sports Day** (23 Aug 2026, School Playground), **Science Fair 2026** (02 Sep 2026, Main Hall), **Cultural Program** (12 Sep 2026, Auditorium) |
| Testimonials | 4 (Principal, Vice Principal, Teacher, Student Parent) |
| Teachers | 4; Staff 1; FAQ 3 (all identical); Why-choose-us 4; Achievement counters 4 |
| Academics pages | 9 (all placeholder) |

---

## 6. Admin Panel (Mighty School)

Reached via `/login`. Layout: Metronic-style top bar (logo, language switch EN/BN, notifications, user dropdown, logout), collapsible left sidebar (accordion menu groups), content area.

### 6.1 Dashboard

`GET /dashboard`

- **Demo-mode alert** banner (restrictions listed in §4).
- "Welcome back, Admin" + subtitle "Track real-time school statistics and key performance metrics."
- **Quick Action** shortcut.
- **KPI cards:** Total Admin **4+**, Total Students **102+**, Total Teachers **2+**, Total Staffs **0+**.
- **Attendance Summary:** total students 102, gender distribution **56 Male (54.9%) / 46 Female (45.1%)**.
- **Fees Collection Overview** (collected vs pending).
- **Income & Expenses Overview** (financial performance).
- **Live Class** (Google Meet widget — "No live classes found").
- **Notice Board:** "Admission Notice 2026" (13 Aug 2024).
- Sidebar is the module launcher (groups below).

### 6.2 Roles & Access Control

Single-user login supports role-based dashboards: **Super Admin, Accountant, Librarian, Teacher, Student**. Admin area includes a **Roles with Permissions** module (`/roles`) and **Users** module (`/users`). There is no public registration — the only intake path is **Admission Forms** (CMS) / student creation.

### 6.3 Module-by-module breakdown

Legend: **Σ** = summary/report screen, **CRUD** = list + add/edit/delete, **CFG** = configuration, **S-A** = super-admin-only (multi-tenant/system).

---

#### 6.3.1 Students Information

| Screen | Route (GET) | Type | Purpose / details |
|---|---|---|---|
| Students List | `/students` | CRUD | Master student register. Filters: Class, Section. Columns: Image, Roll, Admission Number, Name, Class, Section, Group, Phone No, Status, Action |
| Student Migration | `/student-migration` | Σ/CFG | Promote/move students across academic sessions. Filters: Select Class, Select Section, Migration Type, Year, Group |
| Migration Pushback | `/student-migration-pushback` | Σ | Undo migrations |
| Migrated List | `/migrated-list` | Σ | Who moved. Filters: Academic Year, Class, Section (posts to `/migrated-list`) |
| All Students View List (At a Glance) | `/at-a-glance` | Σ | Compact student grid; shows "Student List, Total Found: 100"; columns: Student ID, Roll No., Admission Number, Name, Class, Section, Gender, G.Mobile |

#### 6.3.2 Staffs Information

| Screen | Route | Type | Details |
|---|---|---|---|
| Staff Attendance | `/staffs-attendance` | CRUD/Σ | Role + Date based daily attendance marking |
| Teachers List | `/teachers` | CRUD | Columns: SL No., Profile, Name, Phone, Email, Department, Designation, Blood Group, Status, Action |
| Staffs List | `/staffs` | CRUD | Columns: Profile, Name, Phone, Email, Designation, Blood Group, Status, Action |

#### 6.3.3 Student Attendance

| Screen | Route | Type | Details |
|---|---|---|---|
| Student Attendance | `/student-attendance` | CRUD | Daily attendance by Class, Section, Date |
| Exam Attendance | `/exams-attendance` | CRUD | Class, Section, Subject, Exam |
| Exam Schedule | `/exams-schedule` | Σ | Exam, Exam Start At, Class, Group |
| Attendance Report | `/reports-student_attendance_date_to_date` | Σ | Date-to-date report; filter to `/reports-student_attendance_date_to_date/view`; add % >= optional |
| Absent Fine | `/absent-fine-report` | Σ | Class, Section, From, To |

#### 6.3.4 QR Code Attendance

| Screen | Route | Type | Details |
|---|---|---|---|
| QR Attendance | `/qrattendance/scanner` | Σ | QR scanner UI for attendance punch-in |

#### 6.3.5 Academic Configuration

| Screen | Route | Type | Details |
|---|---|---|---|
| Academic Session | `/academic-years` | CRUD | Session/Academic Year (Session Name, Academic Year) |
| Shift | `/shift` | CRUD | Name → Shift |
| Class | `/class` | CRUD | Name → Class |
| Sections | `/sections` | CRUD | Class(es), Section Name, Group Name, Room No/Name |
| Groups | `/student-groups` | CRUD | Group Name |
| Periods | `/periods` | CRUD | Period */
| Subjects | `/subjects` | CRUD | Subject Name, Subject Code, Class; filter by class |
| Subject Config | `/subject-config/create` | CFG | Assign subjects per class/group from full subject library (Bangla BAN101…, English ENG101, ICT, Religion, Physics, Chemistry, Higher Mathematics, Biology, History, Geography, Civics, Economics, Accounting, Finance, Business Entrepreneurship, Statistics — BAN201/ENG201 second versions, etc.); columns: Subject, Subject Type, Subject Serial, Marge ID |
| Optional Subject Configs | `/optional-subject-config` | CRUD | Optional subject rules per class/group (Name, Class, Group, Subjects, Limit) |
| Student Optional Subjects | `/student-optional-subject` | CRUD | Assign optional subjects to students (Student, Subject, Config) |
| Exam | `/exams` | CRUD | Name + Exam Code |
| Student Categories | `/student-categories` | CRUD | Category names (e.g., scholarship buckets) |
| Departments | `/departments` | CRUD | Department Name + Priority |
| Picklist | `/picklists` | CRUD | Reusable dropdown options (Type, Value, Slug) |
| Principal Signature | `/signatures` | CRUD | Signature images per "Place At" + Title (Principal etc.) |

#### 6.3.6 Fees Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Fees StartUp | `/fee-head` | CRUD | Fee heads + sub-heads (Name, Serial) |
| Fees Mapping | `/fees-mapping` | CFG | Map Fee Head + Sub-Heads → Ledger & Fund |
| Amount Config | `/amount-config` | CFG | Fee amount + fine per Class, Group, Section, Category, Fee Head, Fund, Period, Amount |
| Date Config | `/date-config` | CFG | Fee payable date + fine active date per Academic Year & Fee Head |
| Fine Waiver (attendance) | `/attendance-waiver` | CFG | Waive absent fines by class/section |
| Waiver | `/waivers` | CRUD | Waiver names |
| Waiver Config | `/waiver-config` | CFG | Apply waivers per student/roll/fee-head (Academic Year, Group, Class, Section, Student Category) |
| Smart Collection | `/quick-collection` | Σ/S | Fee collection screen by Class, Section |
| Paid Info | `/payment-fee-info` | Σ | Fetched payments: From Date, To Date, Class, Section |
| Unpaid Info | `/unpaid-info` | Σ | Due list: SL, Student, Roll, Due Details, Total Due |

#### 6.3.7 Accounts Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Ledger | `/accounting-ledgers` | CRUD | Category + Group + Ledger Name (SL No., Ledger Name, Account Group, Account Category, Nature) |
| Fund | `/accounting-funds` | CRUD | Fund Name + Serial; shows Amount In / Out / Balance |
| Category | `/accounting-categories` | CRUD | Accounting Category (Name, Code, Type) |
| Group | `/accounting-groups` | CRUD | Account Category + Group Name |
| Payment | `/account-transaction-payment` | CRUD | Expense voucher: Payment Date, Payment By, Fund, Ref., Description; detail rows ledger_ids[]/amounts[] |
| Receipt | `/account-transaction-payment?type=receipt` | CRUD | Income voucher (Receipt Date, Receipt Type, Fund, Ref., Description) |
| Contra | `/account-contra-transfers` | CRUD | Internal cash/fund moves: Transfer Date, From, To, Amount, Ref., Description |
| Journal | `/journal-transactions` | CRUD | Journal voucher: Journal Date, Fund, Description, Ref.; Debit/Credit rows |
| Fund Transfer | `/account-fund-transfers` | CRUD | Payment Date, Transfer From, Transfer To, Amount, Description |
| Chart Of Accounts | `/chart-of-accounts` | Σ/CFG | Pre-seeded tree: 1. Cash & Cash Equivalence → Assets; 2. Current Liabilities → Liabilities; 3. Non-Current Liabilities → Liabilities; 4. Owner's Equity → Liabilities; 5. Fees Related Income → Income; 6. Others Income → Income; 7. General Expenses → Expense |

#### 6.3.8 Accounting Reports

| Screen | Route | Type | Details |
|---|---|---|---|
| Balance Sheet | `/balance-sheet` | Σ | From/To range |
| Trial Balance | `/trial-balance` | Σ | From/To range |
| Cash Flow Statement | `/cash-flow-statement` | Σ | Year filter |
| Cash Flow Details | `/cash-flow-details` | Σ | Year filter |
| Cash Book Account | `/cash-book-account` | Σ | From Date, To Date, Payment Method |
| Ledger Book Account | `/ledger-book-account` | Σ | From Date, To Date, Payment Method |
| Income Statement | `/income-statement` | Σ | Ledger-Income List (Debit/Credit), Ledger-Expense List, Ledger-Profit/Loss |
| Income Statement Details | `/income-statement-details` | Σ | From/To |
| Cash Summary | `/cash-summary` | Σ | Ledger-Income List, Income List, Ledger-Expense List, Expense List |

#### 6.3.9 Payroll Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Payroll Start Up | `/salary-heads` | CRUD | Salary heads + Nature (+/−) |
| Payroll Mapping | `/payroll-mapping` | CFG | Map Payroll → Ledger & Fund |
| Payroll Assign | `/payroll-assign` | CFG | Assign heads to staff; breakdown columns: Net Salary, md (+), Basic (+), Allowance (+), Early Leave Fine (−), Festival Allowance (+), Welfare Fund (−), Professional Tax (−), Conveyance (+), Exam Hall Duty (+), Incentive (+), Medical (+) |
| Salary Slip | `/salary-create` | Σ/S | Generate salary slips |
| Salary | `/salary-payment-process` | Σ/S | Process salary payments |
| Due | `/due-salary-payment` | Σ/S | Due salary (HR ID lookup) |
| Advance | `/advance-salary-payment` | Σ/S | Advance payments (HR ID lookup) |
| Return Advance Payment | `/return-salary-payment` | Σ/S | Recover advances (HR ID lookup) |
| Salary Statement | `/salary-statement` | Σ | Monthly statement |
| Payment Info | `/payment-info` | Σ | Columns: HR ID, Name, Invoice ID, Paid Status, Payment Type, Net Salary, Payable Salary, Paid, Due, Advance, Payment Date |

#### 6.3.10 Routine Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Syllabus | `/syllabus` | CRUD | Title, Description, Class, File |
| Assignments | `/assignments` | CRUD | Title, Description, Class, Section, Subject |
| Class Routine | `/class_routines` | CRUD | Per Class/Section timetable builder |
| Exam Routine | `/exam-routines` | Σ | Exam, Class, Group |
| Admit & Seat Plan | `/exam-essentials` | Σ | Generate Seat Plan / Admit Card (Class, Section, From/To Roll optional) |

#### 6.3.11 Library Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Book Categories | `/book-categories` | CRUD | Category Name |
| Books | `/books` | CRUD | Book Name, Code, Author, Quantity |
| Members / Library ID | `/librarymembers` | CRUD | Library Member ID, Name, Image, Member Type |
| Books Issue | `/books-ber-code-page` | Σ/S | Issue UI with Books Search + Member Search; columns SL, Name, Code, Writer, Quantity, Return Date |
| Book Issues Report | `/bookissues` | Σ | Library Id filter |
| Barcode Books Print | `/books-ber-code-print` | Σ | Books Code → print barcodes |

#### 6.3.12 Exam Module

| Screen | Route | Type | Details |
|---|---|---|---|
| Exam StartUp | `/semester-exam-settings-exam-startup` | CFG | Merge global exam code list + grade list into class exams; Merit Process Type |
| Mark Config | `/semester-exam-settings-mark-config` | CFG | Class/Group/Calculation Method/Exam; per-exam percentage & serial; grand-final mark update per class (Class 9, Class 10, Computer) |
| Remarks Config | `/remarks-config` | CFG | Remark Title + Remarks |
| Mark Input | `/mark-input-section-wise` | Σ | Class List → section-wise mark entry |
| Exam Result | `/exam-result-view` | Σ | Filter Results + "Send Result Notifications" (Send Scope, Classes, Notify via); also reused for SMS result send |
| Grand Final Result | `/grand-final-result` | Σ | Filter Students (Class, Section, Roll range) |
| Tabulation / Broad Sheet | `/tabulation-sheet` | Σ | Generate Broad Sheet (Class, Section, Exam, Sheet Type) |
| Merit List Sheet | `/merit-list-sheet` | Σ | Generate Merit List (Class, Section, Exam) |
| Result Card Settings | `/result-card-settings` | CFG | Card visuals & flags: Result Title, show photo/position/GPA/percentage/failed subjects/attendance/remarks/cognitive+affective+psychomotor domains/teacher+principal+parent signatures; labels; failed-student ranking; Primary/Accent color; watermark opacity; institute watermark |
| Empty Mark Sheet | `/empty-mark-sheet` | Σ | Blank mark input sheets to print (Class, Section, Exam, #students) |
| Assessment Domains | `/assessment-domains` | CFG | Cognitive / Affective / Psychomotor domains + items (also drives report-card "co-curricular" columns) |
| Domain Assessment Entry | `/assessment-entry` | Σ | Score domains per Class, Exam |
| Online Examination | `/online-exams` | CRUD | Online tests: Name, Exam, Class, Subject, Window, Status |

#### 6.3.13 Layout & Certificates

All certificate designs use `/layout-cert?type={slug}`, each with Class / Section / Student selectors and a live layout preview + print:

- `general-certificate` (General Recommendation Letter)
- `testimonial` (Testimonial)
- `attendance-certificate` (Attendance Certificate)
- `hsc-recommendation` (HSC Recommendation Letter)
- `transfer-certificate` (Transfer Certificate)
- `abroad-recommendation` (Abroad Letter)
- `character-certificate` (Character Certificate)
- `study-certificate` (Study Certificate)
- `bonafide-certificate` (Bonafide Certificate)
- `migration-certificate` (Migration Certificate)

| Screen | Route | Type | Details |
|---|---|---|---|
| Manage Templates | `/certificate-templates` | CRUD | Custom templates: Name, Certificate Type, Orientation, Colors, Status |

#### 6.3.14 SMS Module

| Screen | Route | Type | Details |
|---|---|---|---|
| SMS Template | `/sms-template` | CRUD | Template Title + Message |
| Phone Book Category | `/phone-book-category` | CRUD | Category Title + Description |
| Phone Book | `/phone-book` | CRUD | Contacts (Name, Phone, Category, Class, Section, Note); bulk sync via `POST /phone-book-sync` |
| SMS Sent | `/sms-compose` | Σ/S | Compose one-off: Select Class, SMS Template, Section, Number; Message (max 300) |
| Exam Result SMS | `/exam-result-view` | Σ/S | Sends result SMS (same screen as result) |
| Purchase SMS | `/sms-purchase` | CFG | Quantity, Price, Transaction date, Masking Type, SMS Gateway |
| SMS Report | `/send-sms-report` | Σ | From Date, To Date |

#### 6.3.15 Administrator

| Screen | Route | Type | Details |
|---|---|---|---|
| Assign Shift | `/assign-shifts` | CRUD | Teacher ↔ Shift |
| Assign Subject | `/assign-subjects` | CRUD | Per Class/Section: Subject ↔ Teacher |
| Assign Class | `/assign-class` | CRUD | Teacher ↔ Class |
| Notice | `/notices` | CRUD | Image, Title, User Type, Notice |
| Events | `/events` | CRUD | Image, Start/End Date, Name, Location |
| Contact Message | `/contact-message` | Σ | Submissions from `/contact-submit`: Name, Phone, Email, Message, Date, Status |
| User Activities | `/user-logs` | Σ | SL No., Name, IP Address, Action, Detail, Create Time, Update Time |
| Student ID Cards | `/student-id-cards` | Σ/S | Class, Group, Section, Card Validity Date → print |
| Teacher ID Cards | `/teacher-id-cards` | Σ/S | Department, Card Validity Date → print |
| Staff ID Cards | `/staff-id-cards` | Σ/S | Department, Card Validity Date → print |

#### 6.3.16 System

| Screen | Route | Type | Details |
|---|---|---|---|
| System Information | `/system/information` | Σ | App/server info panel |
| System Update | `/system/update` | S-A | Manual update upload |
| Update History | `/system/update/history` | S-A | Version, Date, Type, Status, Admin, Duration |
| Modules | `/system/modules` | S-A | Module Management: Module, Version, Status, Dependencies, Actions |
| System Settings | `/system/settings` | S-A | Update Server URL, Demo Reset Interval (hours), Enable remote update check, Enable demo auto reset |

#### 6.3.17 Master Configuration (multi-tenant)

| Screen | Route | Type | Details |
|---|---|---|---|
| System Settings (tenant) | `/administration/general_settings` | CFG | Giant settings form (see 6.4) |
| Custom Domain | `/custom-domain` | S-A | Request/attach custom domain (Domain, Note, Status, Submitted) |
| Roles | `/roles` | CRUD | Role with Permissions |
| Users | `/users` | CRUD | Profile, Name, Email, Phone, User Type |
| Institutes | `/institutes` | S-A | Create/approve tenants: confirm box "Type Institute 1 to confirm", Select Package, Start Date, Amount Paid, Payment Method; table: Institute Name, Type, Owner, Phone, Domain, Package, Subscription, Status, Created, Action |
| Branches | `/branches` | S-A | Institute → Branch |
| Payment Gateways | `/payment-gateways` | CFG | Gateway, Mode, Status |

#### 6.3.18 CMS Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Admission Forms | `/admission-forms` | Σ | Online admission applications: Student, Class, Contact, Guardian, Status, Date |
| Pages | `/pages` | CRUD | Title, Slug, Status (powers `/page/{slug}` frontend) |
| Banners | `/banners` | CRUD | Title, Image, button_name, button_link, description |
| About Us | `/about-us` | CRUD | Title, Image, Short Description |
| FAQs | `/faqs` | CRUD | Question, Answer, Status |
| Gallery Images | `/gallery-images` | CRUD | Title, Heading, Image, Status (front: `/academic-images`) |
| Mobile App Sections | `/mobile-app-sections` | CRUD | Title, Heading, Image, Features, Store Links |
| Why Choose Us | `/why-choose-us` | CRUD | Title, Icon, Description |
| Policies | `/policies` | CRUD | Policy pages (Privacy/Terms/Refund/Cookies frontend): Type, Status, Description |
| Ready to Join Us | `/ready-to-join-us` | CRUD | Icon, Title, Description, Button |
| Testimonials | `/testimonials` | CRUD | Image, Name, Rating, Status |
| Image Setting | `/institute-image-settings` | CRUD | Header Light/Dark, Footer Light/Dark, Favicon, Status |

#### 6.3.19 WhatsApp

| Screen | Route | Type | Details |
|---|---|---|---|
| WhatsApp Settings | `/whats-app-settings` | CFG | Provider, Phone ID, Business ID, Access Key, Language, Status |
| WhatsApp Templates | `/whats-app-templates` | CRUD | WhatsApp message templates (Name, Event, Message, Status) |
| WhatsApp Logs | `/whats-app-logs` | Σ | Message, Phone, Retries, Sent At, Status, Student; filters From Date/Status/To Date |

#### 6.3.20 Hostel Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Hostel Categories | `/hostel-categories` | CRUD | Hostel, Standard, Fee, Note |
| Hostel Members | `/hostel-members` | CRUD | Student, Class, Hostel, Category, Fee |
| Buildings | `/hostel-buildings` | CRUD | Hostel → Building Name |
| Floors | `/hostel-floors` | CRUD | Building → Floor Name |
| Rooms | `/rooms` | CRUD | Room Number, Hostel Category, Capacity |
| Beds | `/beds` | CRUD | Room, Bed Number, Status |
| Room Members | `/room-members` | CRUD | Student, Phone, Room, Hostel Category |
| Meals | `/meals` | CRUD | Meal Name, Type |
| Meal Plans | `/meal-plans` | CRUD | Student, Meal, Date |
| Meal Entries | `/meal-entries` | CRUD | Student, Meal, Date, Price |
| Hostel Bills | `/hostel-bills` | CRUD | Student, Hostel Fee, Meal Fee, Total, Due Date |
| Hostel Leaves | `/hostel-leaves` | CRUD | Student, Room, From/To, Reason, Status |
| Hostel Dashboard | `/hostel-dashboard` | Σ | KPIs (occupancy %, 0% / 3/3 / 1 / 0.00 seeded) |
| Seat Map | `/hostel-seat-map` | Σ | Visual bed assignment grid |
| Hostel Fee Collection | `/hostel-collections` | Σ/S | Month + Student/Room search; This Month + All Outstanding Due |
| Hostel Reports | `/hostel-reports/collection` | Σ | Collection report (Invoice, Student, Room, Paid, Due, Date) |

#### 6.3.21 Inventory

| Screen | Route | Type | Details |
|---|---|---|---|
| Categories | `/inventory-categories` | CRUD | Name, Note |
| Items | `/inventory-items` | CRUD | Name, Category, SKU, Cost Price, Selling Price, Stock, Status |
| Sales | `/inventory-sales` | CRUD | Invoice, Student, Date, Payable, Paid, Due |

#### 6.3.22 Transport Management

| Screen | Route | Type | Details |
|---|---|---|---|
| Buses | `/buses` | CRUD | Bus Number, Model, Capacity, Status |
| Drivers | `/drivers` | CRUD | Name, Phone, License No, Assigned Bus, Status |
| Bus Routes | `/bus-routes` | CRUD | Route Name, Start/End Location, Distance, Estimated Time, Status |
| Bus Stops | `/bus-stops` | CRUD | Stop Name, Route Name, Latitude, Longitude, Order |
| Transport Members | `/transport-members` | CRUD | Student, Route, Stop, Fare, Status |
| Vehicle Types | `/vehicle-types` | CRUD | Name, Status |
| Vehicle Categories | `/vehicle-categories` | CRUD | Name, Status |
| Helpers | `/helpers` | CRUD | Name, Phone, Assigned Bus, Status |
| Transport Dashboard | `/transport-dashboard` | Σ | KPIs (1/ 0/ 0/0 / 0.00 seeded) |
| Transport Fee Collection | `/transport-collections` | Σ/S | Month; Student, Route, Stop, Fare, This Month |
| Transport Reports | `/transport-reports/collection` | Σ | Collection report (Invoice, Student, Route, Month, Paid, Due, Date) |

#### 6.3.23 Google Meet

| Screen | Route | Type | Details |
|---|---|---|---|
| All Meetings | `/google-meet` | CRUD | Title, Teacher, Class, Section, Date, Time, Status, Meet Link |
| Create Meeting | `/google-meet/create` | CFG | Meeting Title, Teacher, Description, Class, Section, Group, Subject, Start Date/Time, End Time, Duration (minutes), Notes/Attachments, Visibility, Recipients (Student/Guardian), Recurring (Enable, Repeat, Until) |
| Test connection | `POST /google-meet/test-connection` | S-A | Verify Google OAuth credentials |

#### 6.3.24 AI Assistant (AI)

| Screen | Route | Type | Details |
|---|---|---|---|
| Chat | `/ai/chat` | Σ | Conversational assistant over institute data |
| Content Writer | `/ai/writer` | Σ | Generate content by `Content Type` + instruction |
| Data Insights | `/ai/insights` | Σ | Ask questions over reports (Report, Class optional, From, To, ask a question) |
| Settings | `/ai/settings` | CFG | **Anthropic API Key** (LLM backend) |

#### 6.3.25 Miscellaneous / Utility routes

| Route | Purpose |
|---|---|
| `/profile` | Update Full Name, Email Address, Change Photo, Current/New Password |
| `/language/en`, `/language/bn` | Language switch (English / বাংলা) |
| `/institute-cache-clear` | Clear app cache (route only; not in sidebar) |
| `/sections-section-group-wise`, `/groups-class-section-wise` | AJAX cascade select endpoints (class → section/group) (not in sidebar) |
| `/contact-submit` | Public contact form handler (frontend) |
| `/login`, `/logout` | Auth |

### 6.4 System Settings (tenant) — `/administration/general_settings`

A single huge tabbed settings screen saving via `POST /administration/general_settings/update`. Field inventory by tab:

- **General:** Institute Name, Site Title, Tagline / Footer Subtitle, Phone, Email, Address, EIIN, Institute ID, CopyRight.
- **SEO:** SEO Meta Title/Description/Keywords, Social Share (OG) Image URL, Google Analytics 4 Measurement ID, Google Tag Manager ID, Meta (Facebook) Pixel ID.
- **Results/fees:** Tuition Fee, Exam and result (display flags), Attendance OUT After (Device), Transfer Certificate Fee, Live Notice On Header.
- **Online admission/app:** Online Admission Display Status, Exam Results Display Status, App Version, App URL, Play Store Link, App Store Link.
- **Mail:** Mail Type, From Email, From Name, SMTP Host/Port/Username/Password/Encryption.
- **SMS:** SMS Gateway, SMS Test Mode, API Key, Sender ID, User Name, Sender Name, Twilio SID/Token, From Number, Base URL, Originator.
- **Social:** Google Map + "on google map" toggle, Facebook, Google Plus, Youtube, WhatsApp, Twitter, LinkedIn.
- **Zoom:** Zoom Account ID, Client Key, Client Secret.
- **Google Meet:** Enable toggle, Client ID/Secret, Project ID, Redirect URI, Service Account JSON, Calendar ID, Default Duration (minutes), Default Timezone, Default Visibility.
- **Theme:** Primary Color, Secondary Color, Text Color, Sidebar Color.

---

## 7. Integrations

| Integration | Evidence | Configured at |
|---|---|---|
| SMTP email | Mail settings + From Email/Name | System Settings |
| SMS gateways | SMS Gateway + Twilio SID/Token, SMS purchase & reporting | System Settings / SMS module |
| WhatsApp Cloud API | Phone ID, Business ID, Access Key; templates + logs | WhatsApp module |
| Payment gateways | Gateway list, Mode (test/live), Status | Master Config → Payment Gateways |
| Google Meet (OAuth) | Full OAuth + calendar service account | System Settings / Google Meet |
| Zoom | OAuth/API keys | System Settings |
| Google Analytics / GTM / Meta Pixel | Measurement ID / container / pixel IDs | System Settings → SEO |
| LLM (Anthropic) | AI Chat / Writer / Insights powered by Anthropic API key | AI → Settings |
| Google Maps | Embedded map on Contact page & field | System Settings |
| Mobile apps | App version, App/Play Store/App Store links, Mobile App Sections CMS | System Settings / CMS |
| Cloudflare | Edge proxy, emails, WAF, bots | Infra (external) |

---

## 8. Multi-Tenancy, Billing & Deployment

- **Tenant objects:** `Institutes` (name, type, owner, phone, domain, **Package**, **Subscription**, status) and `Branches`. New tenants can be created by superadmin (payment method capture; "Amount Paid"; install/package select).
- **Packages / subscriptions:** institutes carry a package + subscription; install flow is part of the Superadmin area.
- **Custom domains:** tenants may request custom domains (`.bdboibazer.com` subdomain default; e.g., `institute.bdboibazer.com`).
- **Demo controls (SaaS):** `system/settings` exposes Demo Reset Interval + auto-reset flag; the running tenant actively enforces demo mode (credentials/settings/profile/upgrades locked).
- **Updates:** remote update server URL + manual update + update history; cache clear endpoint.

---

## 9. System Operations

- `GET /system/information` — environment/server details.
- `GET /system/update` & `/system/update/history` — patching.
- `GET /system/modules` — enable/disable/version modules.
- `GET /institute-cache-clear` — flush cache.
- `GET|POST /language/{en|bn}` — locale switch (i18n pack installed).
- Standard REST-ish route naming observed:
  - List/screen `GET /{resource}`
  - Create/update `POST /{resource}`
  - Update detail `POST /{resource}/{id}` (with `_method` spoof e.g. `DELETE`)
  - AJAX cascade `GET /sections-section-group-wise`, `/groups-class-section-wise`

---

## 10. Audit Summary & Security Observations

**What was done:** Logged in as `superadmin@gmail.com` (demo creds), visited all ~185 admin screens, all frontend pages, extracted menus, forms, table schemas and endpoints. **All operations were read-only GET/POST-login; nothing was created, edited, or deleted.** The login session was the only side effect (a demo session cookie).

**Notable observations (informational, no action taken):**

1. **Demo credentials exposed in public JavaScript** on the login page (5 role accounts, password `12345678`). Expected for a public demo, but the same pattern must never ship in production.
2. **Demo-mode guard banner** present in dashboard — tenant enforces read-only on password/email/profile/system settings/upgrades.
3. **Placeholder content** throughout frontend (Academics pages, FAQ ×3 identical, duplicate "About", refund page blank) — CMS is functional, seeded data is just incomplete.
4. **HTTP endpoints observed** (`/contact-submit`, cascading select AJAX, certificate print layouts, ID-card print, barcode print) — no CSRF on GET analytics, standard Laravel CSRF on all POST.
5. **Cloudflare bot protection** active; `robots.txt` explicitly blocks major AI crawlers (GPTBot, ClaudeBot, CCBot, Amazonbot…) and sets `Content-Signal: search=yes, ai-train=no, use=reference`.
6. **Sensitive config present in UI** for tenant admin (SMTP password, SMTP/Twilio/Zoom keys, Google service account JSON, Anthropic API key, SMS credentials). These fields are masked only at input level — a hardcoded/demo credential or public API key here would be a real exposure.
7. **Large new-feature surface** unusual for a "school" demo: AI assistant (Anthropic), online exams, QR attendance, Google Meet scheduling, hostel/transport/inventory with fee engines, full double-entry accounting, payroll, certificates, SMS/WhatsApp marketing.
8. No `sitemap.xml` (404); no public registration; single sign-in for all roles; admission intake only via Admission Forms module.

---

## 11. Master Route / Page Index

Complete list of distinct routes observed (public + authenticated). Frontend pages are marked ⚪, everything else is behind login.

### Public (frontend)
```
/
/academic-images
/contact
/event-details/1   /event-details/2   /event-details/3
/page/approach /page/career-counselling /page/facilities /page/houses-mentoring
/page/mission /page/principal-message /page/school-uniform /page/student-council /page/vision
/privacy-policy /terms-conditions /refund-policy /cookies-policy
/login   POST /login   POST /contact-submit
```

### Admin navigation (sidebar groups → routes)

**Students Information**
```
/students /student-migration /student-migration-pushback /migrated-list /at-a-glance
```
**Staffs Information**
```
/staffs-attendance /teachers /staffs
```
**Student Attendance**
```
/student-attendance /exams-attendance /exams-schedule /reports-student_attendance_date_to_date /absent-fine-report
```
**QR Code Attendance** — `/qrattendance/scanner`
**Academic Configuration**
```
/academic-years /shift /class /sections /student-groups /periods /subjects
/subject-config/create /optional-subject-config /student-optional-subject /exams
/student-categories /departments /picklists /signatures
```
**Fees Management**
```
/fee-head /fees-mapping /amount-config /date-config /attendance-waiver /waivers
/waiver-config /quick-collection /payment-fee-info /unpaid-info
```
**Accounts Management**
```
/accounting-ledgers /accounting-funds /accounting-categories /accounting-groups
/account-transaction-payment  /account-transaction-payment?type=receipt
/account-contra-transfers /journal-transactions /account-fund-transfers /chart-of-accounts
```
**Accounting Reports**
```
/balance-sheet /trial-balance /cash-flow-statement /cash-flow-details /cash-book-account
/ledger-book-account /income-statement /income-statement-details /cash-summary
```
**Payroll Management**
```
/salary-heads /payroll-mapping /payroll-assign /salary-create /salary-payment-process
/due-salary-payment /advance-salary-payment /return-salary-payment /salary-statement /payment-info
```
**Routine Management**
```
/syllabus /assignments /class_routines /exam-routines /exam-essentials
```
**Library Management**
```
/book-categories /books /librarymembers /books-ber-code-page /bookissues /books-ber-code-print
```
**Exam Module**
```
/semester-exam-settings-exam-startup /semester-exam-settings-mark-config /remarks-config
/mark-input-section-wise /exam-result-view /grand-final-result /tabulation-sheet /merit-list-sheet
/result-card-settings /empty-mark-sheet /assessment-domains /assessment-entry /online-exams
```
**Layout & Certificates**
```
/layout-cert?type=general-certificate | test&v=testimonial | attendance-certificate | hsc-recommendation
/layout-cert?type=transfer-certificate | abroad-recommendation | character-certificate | study-certificate
/layout-cert?type=bonafide-certificate | migration-certificate
/certificate-templates
```
**SMS Module**
```
/sms-template /phone-book-category /phone-book /sms-compose /exam-result-view /sms-purchase /send-sms-report
```
**Administrator**
```
/assign-shifts /assign-subjects /assign-class /notices /events /contact-message /user-logs
/student-id-cards /teacher-id-cards /staff-id-cards
```
**System**
```
/system/information /system/update /system/update/history /system/modules /system/settings
```
**Master Configuration**
```
/administration/general_settings /custom-domain /roles /users /institutes /branches /payment-gateways
```
**CMS Management**
```
/admission-forms /pages /banners /about-us /faqs /gallery-images /mobile-app-sections
/why-choose-us /policies /ready-to-join-us /testimonials /institute-image-settings
```
**WhatsApp (module pages, not in sidebar menu)**
```
/whats-app-settings /whats-app-templates /whats-app-logs
```
**Hostel Management**
```
/hostel-categories /hostel-members /hostel-buildings /hostel-floors /rooms /beds /room-members
/meals /meal-plans /meal-entries /hostel-bills /hostel-leaves /hostel-dashboard /hostel-seat-map
/hostel-collections /hostel-reports/collection
```
**Inventory**
```
/inventory-categories /inventory-items /inventory-sales
```
**Transport Management**
```
/buses /drivers /bus-routes /bus-stops /transport-members /vehicle-types /vehicle-categories
/helpers /transport-dashboard /transport-collections /transport-reports/collection
```
**Google Meet**
```
/google-meet /google-meet/create
```
**AI Assistant**
```
/ai/chat /ai/writer /ai/insights /ai/settings
```
**Other authenticated / utility**
```
/dashboard /profile /logout
/language/en /language/bn /institute-cache-clear
```

*End of documentation — generated by a read-only audit. No data was changed on the target.*