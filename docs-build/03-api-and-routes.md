# 03 — API & Routes

Laravel web routes. CSRF required on every POST. `_method` spoof used for PUT/DELETE on forms.

## 1. Route grouping

| Prefix | Area | Middleware |
|---|---|---|
| `/` | Public site + CMS pages | `web` |
| `/login`, `/logout` | Auth | `web` (guest for login) |
| `/admin/*` (implied) | Back office | `web`, `auth`, `role` |
| `/api/*` | (Future JSON API for apps) | `auth:sanctum`, `throttle` |

Reference site uses flat routes (`/students`, `/fees`...) — recommended to keep flat web routes + add `/api` separately.

## 2. Public routes

```
GET  /                          → homepage (banners, why us, about, counters, teachers, staff, events, testimonials, faq)
GET  /page/{slug}               → CMS page (approach, career-counselling, facilities, houses-mentoring,
                                   mission, principal-message, school-uniform, student-council, vision)
GET  /academic-images           → gallery
GET  /event-details/{id}        → event detail
GET  /contact                   → contact page (map embed + form)
POST /contact-submit            → store contact message (name, email, phone, message) → redirect w/ flash
GET  /privacy-policy /terms-conditions /refund-policy /cookies-policy   → policy pages
GET  /login                     → login page (lists demo creds when demo_mode)
POST /login                     → authenticate (email, password, remember) → redirect dashboard
POST /logout                    → logout (csrf)
```

## 3. Shared / admin route conventions

```
GET    /{resource}                 → index (list + filters)
GET    /{resource}/create          → create form (rare; modeless forms used)
POST   /{resource}                 → store
GET    /{resource}/{id}            → show / edit (some screens)
POST   /{resource}/{id}            → update
POST   /{resource}/{id}  (_method=DELETE) → destroy
```

## 4. Admin route inventory (by group)

### Students Information
```
GET  /students                        (filter: class, section)
POST /students                        store
POST /students/{id}                   update
POST /students/{id}?delete            destroy
GET  /student-migration               list
POST /student-migration               migrate selected
GET  /student-migration-pushback      undo UI
POST /student-migration-pushback      pushback
GET  /migrated-list                   (filters academic_year,class,section)
POST /migrated-list                   filter submit
GET  /at-a-glance                     all-students grid
```

### Staffs Information
```
/staffs-attendance  GET list  POST save(role,date,statuses)
/teachers           CRUD
/staffs             CRUD
```

### Student Attendance
```
/student-attendance                 GET mark UI  POST save(class,section,date,statuses)
/exams-attendance                   GET  POST save
/exams-schedule                     GET list
/reports-student_attendance_date_to_date      GET form
/reports-student_attendance_date_to_date/view POST → rendered report
/absent-fine-report                 GET  POST filter
```

### QR Code Attendance
```
/qrattendance/scanner            GET (camera UI)  POST /qrattendance/scan (ajax)
```

### Academic Configuration
```
/academic-years /shift /class /sections /student-groups /periods /subjects
/subject-config/create  (step wizard: class→group→subjects→serial/merge)
/optional-subject-config CRUD
/student-optional-subject CRUD
/exams CRUD
/student-categories CRUD
/departments CRUD
/picklists CRUD (filtered by type)
/signatures CRUD
```

### Fees Management
```
/fee-head          CRUD (head + sub-head rows)
/fees-mapping      GET  POST store mapping
/amount-config     GET  POST store/update amounts
/date-config       GET  POST store dates
/attendance-waiver GET  POST save waivers
/waivers           CRUD
/waiver-config     GET  POST apply
/quick-collection  GET(class,section) POST collect (posts fee payment + journal lines)
/payment-fee-info  GET  POST filter
/unpaid-info       GET  POST filter
```

### Accounts Management (vouchers)
```
/accounting-ledgers   CRUD
/accounting-funds     CRUD
/accounting-categories CRUD
/accounting-groups    CRUD
/account-transaction-payment?type=payment | type=receipt   GET list, POST store voucher
/account-contra-transfers  GET  POST store
/journal-transactions     GET  POST store
/account-fund-transfers   GET  POST store
/chart-of-accounts        GET tree
```

### Accounting Reports
```
/balance-sheet   GET + POST filter
/trial-balance   GET + POST filter
/cash-flow-statement  GET + POST (year)
/cash-flow-details    GET + POST (year)
/cash-book-account    GET + POST (from,to,payment_method)
/ledger-book-account  GET + POST (from,to,payment_method)
/income-statement     GET
/income-statement-details GET + POST (from,to)
/cash-summary         GET
```

### Payroll Management
```
/salary-heads        CRUD
/payroll-mapping     GET  POST store(ledger,fund)
/payroll-assign      GET grid  POST save cells
/salary-create       GET + POST generate slips
/salary-payment-process  GET + POST process
/due-salary-payment       GET + POST pay
/advance-salary-payment   GET + POST pay advance
/return-salary-payment    GET + POST recover
/salary-statement         GET report
/payment-info             GET report
```

### Routine Management
```
/syllabus        CRUD
/assignments     CRUD
/class_routines  GET list(by class/section)  POST save grid
/exam-routines   GET  POST save
/exam-essentials GET form  POST generate admit/seating
```

### Library Management
```
/book-categories    CRUD
/books              CRUD
/librarymembers     CRUD
/books-ber-code-page GET issue UI POST issue POST return
/bookissues         GET report(filter library_id)
/books-ber-code-print GET barcode labels
```

### Exam Module
```
/semester-exam-settings-exam-startup      GET  POST store per class
/semester-exam-settings-mark-config       GET  POST store percentages
/remarks-config        CRUD
/mark-input-section-wise                  GET class list → per section POST marks
/exam-result-view                         GET + POST filter; POST notification dispatch
/grand-final-result                       GET + POST filter/rebuild snapshot
/tabulation-sheet                         GET + POST generate
/merit-list-sheet                         GET + POST generate
/result-card-settings                     GET  POST save
/empty-mark-sheet                         GET + POST print
/assessment-domains                       GET  POST domain/item CRUD
/assessment-entry                         GET  POST scores
/online-exams                             CRUD (+ attempts API)
```

### Layout & Certificates
```
/layout-cert?type={slug}    GET with class/section/student → live layout + print
/certificate-templates      CRUD
```

### SMS Module
```
/sms-template         CRUD
/phone-book-category  CRUD
/phone-book           CRUD + POST /phone-book-sync (import numbers)
/sms-compose          GET  POST send (number/template, max 300)
/exam-result-view     (reused) POST send result SMS
/sms-purchase         GET  POST record purchase
/send-sms-report      GET  POST filter
```

### Administrator
```
/assign-shifts        CRUD(teacher↔shift)
/assign-subjects      CRUD(class/section → subject↔teacher)
/assign-class         CRUD(teacher↔class)
/notices              CRUD
/events               CRUD
/contact-message      index + POST change status
/user-logs            index (view)
/student-id-cards     GET form POST generate
/teacher-id-cards     GET form POST generate
/staff-id-cards       GET form POST generate
```

### System (super admin)
```
/system/information        GET info
/system/modules            GET list  POST toggle
/system/update             GET  POST upload+apply
/system/update/history     GET
/system/settings           GET  POST save(update server url, demo reset interval, flags)
```

### Master Configuration
```
/administration/general_settings   GET  POST /administration/general_settings/update (multi-tab)
/custom-domain                     GET  POST request domain
/roles                             CRUD (permissions)
/users                             CRUD
/institutes                        CRUD + POST enroll(package, start_date, amount_paid, payment_method) + confirm
/branches                          CRUD
/payment-gateways                  CRUD
```

### CMS Management
```
/admission-forms     GET list  POST status change
/pages               CRUD (title, slug, content, status)
/banners             CRUD
/about-us            CRUD
/faqs                CRUD
/gallery-images      CRUD
/mobile-app-sections CRUD
/why-choose-us       CRUD
/policies            CRUD
/ready-to-join-us    CRUD
/testimonials        CRUD
/institute-image-settings CRUD
```

### WhatsApp
```
/whats-app-settings         GET  POST save
/whats-app-templates        CRUD
/whats-app-logs             GET filter(from,status,to) + retry
```

### Hostel Management
```
/hostel-categories CRUD  /hostel-members CRUD  /hostel-buildings CRUD  /hostel-floors CRUD
/rooms CRUD  /beds CRUD  /room-members CRUD  /meals CRUD  /meal-plans CRUD
/meal-entries CRUD  /hostel-bills CRUD  /hostel-leaves CRUD
/hostel-dashboard GET  /hostel-seat-map GET
/hostel-collections GET + POST collect  /hostel-reports/collection GET + POST filter
```

### Inventory
```
/inventory-categories CRUD   /inventory-items CRUD   /inventory-sales CRUD
```

### Transport Management
```
/buses CRUD  /drivers CRUD  /helpers CRUD  /bus-routes CRUD  /bus-stops CRUD
/vehicle-types CRUD  /vehicle-categories CRUD  /transport-members CRUD
/transport-dashboard GET  /transport-collections GET+POST  /transport-reports/collection GET+POST
```

### Google Meet
```
/google-meet        CRUD list
/google-meet/create GET form  POST store
/google-meet/test-connection  POST (verify OAuth)
POST /google-meet/{id}/join / cancel / notify
```

### AI Assistant
```
/ai/chat      GET UI  POST message  POST new session
/ai/writer    GET   POST generate(content_type,prompt)
/ai/insights  GET   POST run(report,class_id?,from,to,question)
/ai/settings  GET   POST save(anthropic_api_key,model,enabled)
```

### Other
```
/profile            GET  POST update(name,email,photo,password...)
/language/{en|bn}   GET switch locale (session)
/institute-cache-clear  GET flush tenant cache
/sections-section-group-wise   GET ?class_id= → sections JSON (cascade)
/groups-class-section-wise     GET ?class_id=&section_id= → groups JSON
```

## 5. JSON API (recommended for future mobile apps)

- `POST /api/v1/auth/login` (guest) → token
- `GET /api/v1/me` (auth:sanctum)
- `GET /api/v1/dashboard/summary`
- `GET /api/v1/student/results` / `attendance` / `fees`
- `GET /api/v1/meetings`
- `GET /api/v1/notices`
- Provide `Accept: application/json`; all errors `{error: {...}}`.

## 6. Error & status codes

- 200 success, 302 redirects (forms), 401 unauthenticated, 403 role denied, 404 not found, 419 CSRF, 422 validation.