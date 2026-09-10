# 05 — Module Functional Specifications

Each entry: **Purpose · Key rules · Screens/behavior · Outputs**. Entities referenced from `entities.md`; routes from `03`.

## 1. Students Information

- Student CRUD with roll & admission number generation (auto next roll per class/section; admission number unique).
- Filters: class, section, keyword; server-paginated DataTable.
- Status lifecycle: active → passed_out/inactive.
- Migration: promote by year with target class/section/group (bulk or single); pushback reverts; migrated-list trackable by academic year/class/section.
- At-a-Glance: read-only grid of all students (reference seeded 100).

## 2. Staffs Information

- Staff & Teacher CRUD; teacher flagged for `/teachers` list; HR ID for payroll.
- Staff attendance by role+date grid (P/A/L), per staff row.

## 3. Student Attendance

- Daily grid class/section/date with quick P/A/L/E toggles; bulk submit.
- Exam attendance separately (exam+class+section+subject).
- Report: attendance % per student between dates (with optional >% filter); absent-fine report derives fines from configured rules.

## 4. QR Code Attendance

- Scanner UI: open camera (QR via lib) → decode student code → confirm → insert `qr_scans` (dedupe per student+date).
- Requires QR on student ID card. Manual fallback remains.

## 5. Academic Configuration

- Session(academic year) with current flag (all fee/exam configs keyed to it).
- Class→Sections(w/ groups, room)→Periods; Cascade select endpoints for forms.
- Subjects library; Subject Config wizard: pick class/group → multi-select subjects → set type/serial/merge; supports merged marking (e.g., two papers as one).
- Optional subject config (limit per class/group); assign to students.
- Exams list (name+code); student categories; departments (used for staff/ID cards); picklists (type/value/slug) for extensible dropdowns; signatures (principal etc.) for documents.

## 6. Fees Management

- Fee heads w/ sub-heads (serial); map each head+sub to accounting ledger & fund.
- Amount config keyed `class+group+section+category+head+period`; fine amount per head.
- Date config: payable date & fine-active date per year+head.
- Waivers: named waiver types; waiver config applies waiver per student/fee head (amount/percent).
- Attendance waiver: excused absences (skip fines).
- Quick/Smart collection: choose class/section → student rows with computed due (payable+fine−waiver−paid) → collect → creates fee payment + posts journal lines + updates fund/ledger balances; receipt print.
- Paid info (filters), Unpaid info (due report w/ details per head).
- Outcome: cash book / income statement reflect collected fees.

## 7. Accounts Management

- Ledger/Fund/Category/Group CRUD; chart-of-accounts tree (7 seeded groups, see schema).
- Vouchers: Payment (expense: date, paid-by ledger, fund, ref, lines[]ledgers+amounts), Receipt (income via `?type=receipt`), Contra (fund→fund), Journal (debit/credit lines), Fund Transfer.
- Posting rules: debits = credits per voucher; fund balance updated; ledger book fed.
- No negative fund balances.

## 8. Accounting Reports

- Balance sheet, trial balance, cash flow statement/details (by year), cash book & ledger book (date/payment-method filters), income statement & details, cash summary.
- Derived from `journal_lines`; consistent reporting month boundaries; totals displayed with debit/credit columns.

## 9. Payroll Management

- Salary heads (+/−); payroll mapping → ledger/fund.
- Payroll assign: staff salary structure cells (basic, allowance, festival, conveyance, etc. with +/−), auto net = plus − minus.
- Salary slips generation (period), payment process (marks paid, posts to payroll payment info), due tracking, advance payments and returns, salary statement.
- Payment info report: per staff invoice, payable/paid/due/advance/date.

## 10. Routine Management

- Syllabus (file upload per class/subject), Assignments, Class Routine grid (day×period, subject/teacher/room), Exam Routine, Admit Card & Seat Plan generation (roll-range → seats; print).

## 11. Library Management

- Categories, Books (+code/barcode, qty, available), Members (link to user), Issue/return flow (`books-ber-code-page`: search book & member, issue, mark return, overdue fine note), Issues report, Barcode label printing (`books-ber-code-print`).
- Stock check: cannot issue beyond available; return increments.

## 12. Exam Module

- Exam startup: per class build exams from ExamCodeList (code title/total/pass/acceptance) + GradeList; merit process type.
- Mark config: per-exam weight % and serial per class/group; calculation method (weighted/average).
- Remarks config (report-card comments per range).
- Mark input section-wise: subject sheets, per-student written/mcq/practical, compute totals → grade+point via grade list.
- Exam result: filter; send scope (class/section/all), notify-via (SMS/WhatsApp); result card + grand-final snapshot (gpa, position, failed flag).
- Tabulation/broad sheet, merit list sheet per class/section/exam.
- Result card settings drive card layout & signatures (see schema).
- Empty mark sheets (print), Assessment domains (co-curricular items+entry), Online exams (question window, timed, attempts, auto-score).

## 13. Layout & Certificates

- 10 layout generators (general, testimonial, attendance, hsc, transfer, abroad, character, study, bonafide, migration) each class/section/student → printable letter.
- Certificate templates: custom layout, orientation, colors, backgrounds, placeholders, status; signatures merged in.

## 14. SMS Module

- Templates; phone book categories & contacts (+sync from students/staff); compose (class/section or numbers; message ≤300), log sent; exam-result SMS dispatch reuse; SMS purchase records per gateway; report by date.
- Multi-gateway manager (Bangladeshi gateway API + Twilio) with test mode.

## 15. Administrator

- Assignments: teacher↔shift, teacher↔subject(class/section), teacher↔class.
- Notices (audience-targeted, image), Events (public on homepage/`event-details/{id}`).
- Contact messages inbox (status workflow).
- User activity log (IP, action, detail, timestamps).
- ID cards: student (class/group/section/validity), teacher & staff (department/validity) → batch PDF.

## 16. System (super admin)

- Information panel; Modules registry enable/disable; System update upload/apply + history; System settings (update server URL, remote check flag, demo auto reset interval + flag).

## 17. Master Configuration (multi-tenant)

- Roles+permissions; Users; Institutes enrollment (confirm "Type Institute 1", package, start date, paid amount, payment method, status/subscription); Branches; Payment gateways test/live; Custom domains request/verify.
- Tenant onboarding sets default settings + admin user + seeds demo content.

## 18. CMS Management

- Admission forms (approve→student); Pages (title/slug/content → `/page/{slug}`); Banners (homepage carousel); About Us; FAQs; Gallery images; Mobile app sections (features + store links); Why Choose Us; Policies (4 legal types → public routes); Ready To Join Us; Testimonials; Institute image settings (header/footer/favicon light&dark).

## 19. WhatsApp

- Settings (provider/phone id/business id/access key/lang/status); Templates; Logs (message/phone/retries/sent/status/student) with filters & retry; dispatch via Notification service.

## 20. Hostel Management

- Hostels→categories(fee/standard)→buildings→floors→rooms(capacity)→beds(free/assigned/maintenance).
- Members (student→hostel/category/fee), room members (assign bed, capacity check).
- Meals + meal plans (per student per date) + meal entries (price).
- Bills (hostel fee + meal fee per month → due dates), collections, leaves (approval workflow).
- Hostel dashboard KPIs (occupancy %), seat map visual, collection report.

## 21. Inventory

- Categories; Items (SKU, cost/selling, stock); Sales to students (invoice, payable/paid/due), stock decrement on sale lines.

## 22. Transport Management

- Buses (capacity), drivers (+license, assigned bus), helpers; vehicle types/categories; routes (distance/ET), stops (geo coords/order); members (route/stop/fare); monthly collections; dashboard; reports.

## 23. Google Meet

- CRUD sessions with class/section/group/subject, start/end, duration, visibility, recipients (students/guardians), recurrence (repeat until), enable; OAuth service (test-connection); creates Google Calendar events with meet link; status/link columns in list; notify recipients.

## 24. AI Assistant

- Chat: conversation over institute data (RAG over reports) — sessions/messages.
- Content writer: type+prompt→draft text (templates/notices/letters).
- Data insights: report + question (class/date filters)→ natural-language summary.
- Settings: Anthropic key, model, enabled toggle. Keys masked; disable in demo-lock.

## 25. Admin dashboard (global)

KPI cards (total admin/students/teachers/staffs), attendance summary (total/gender split), fees collection overview, income vs expenses, live classes widget, notice board quick list; click-through to modules.

---

## Cross-module business rules (critical)

1. **Multi-tenant isolation:** every query scoped by `institute_id`; IDs never trusted cross-tenant.
2. **Money integrity:** all fee/salary/collection writes inside transactions; accounting lines balance.
3. **Academic year scoping:** fee/exam/waiver configs resolve current `academic_year` unless specified.
4. **Stock/bed/capacity checks** before allocate/issue/sell (no negatives).
5. **Notifications:** one dispatcher; channels via `notifications.via`; failures logged (SMS/WhatsApp logs) with retries.
6. **Demo mode:** block sensitive mutation (see 04 §4).
7. **Audit log** on admin mutate actions (create/update/delete) with user+IP.