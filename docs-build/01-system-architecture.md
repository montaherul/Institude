# 01 — System Architecture

## 1. Stack (ASP.NET Core + SQL Server)

| Layer | Choice | Why |
|---|---|---|
| Backend | **ASP.NET Core 8 MVC** — `MightySchool.Web` | Razor views + Tag Helpers; AH. `agents.md` five-project architecture |
| DB | **SQL Server** (shared schema, `institute_id` discriminator) | transactional; supports multi-tenant + double-entry |
| Front public | Tailwind CSS + Alpine.js + vanilla JS | SEO-friendly server-rendered Razor |
| Admin UI | Metronic-style theme (v8 class conventions) + jQuery 3.7 + Alpine + Select2 + **Tabulator** | reference-compatible feel; Tabulator for server-side grids |
| Auth | Cookie authentication + antiforgery; RBAC via `RoleWiseMenuAccess` + `[MenuAuthorize]` | role gate + UI permission hiding |
| Caching | `IMemoryCache` (or Redis) with tenant-scoped keys | hot: sessions, settings, students list |
| Jobs/Queue | `IHostedService` background workers (in-process) | SMS/WhatsApp/Meet notifications, imports |
| Files | Local `wwwroot/uploads` + optional S3-compatible | images, documents, certificates |
| Edge | Cloudflare proxy/WAF | robots content-signals, email obfuscation |
| Search/pluggables | Select2/Tabulator on client; no ES needed v1 | |

## 2. Logical layers

```
Browser (public SSR)         Browser (admin Tabulator grids via AJAX)
        |                                |
        +------------ Cloudflare -------+
                        |
               ASP.NET Core pipeline (MightySchool.Web)
                 - Cookie auth + Antiforgery
                 - GlobalExceptionMiddleware (AJAX=JSON / other=ServerError)
                 - MenuAuthorize (RoleWiseMenuAccess) + InstituteScope resolve institute/demo_mode
                 |        |         |
        Controllers → Application Services → IUnitOfWork → UnitOfWork
                 |        |                     |
        Razor Views (admin/public)      GenericRepository<T> → EF Core or SP
                 |                                   |
        Background services → Third-party (SMS, WhatsApp, Gmail/Meet, Zoom, Anthropic, Gateways)

                                            SQL Server
```

Higher layers never touch `DbContext` directly (see `AGENTS.md` §4).

## 3. Multi-tenancy model

- **Single DB, shared schema** with `institute_id` discriminator on tenant-scoped tables (simplest, matches SaaS demo). Alternative: schema-per-tenant (documented in `08`).
- `InstituteScope` (application-layer helper) + a generic repository default filter inject `institute_id` scoping automatically.
- `System` tables are global (no tenant): `institutes`, `packages`, `users`, `roles`, `permissions`, `institute_settings`.
- Tenant tables (students, fees, accounts …) carry an indexed `institute_id` column.
- `demo_mode` flag per institute toggles lock behavior.

## 4. Key domain patterns

### Class → Section → Group cascade
Class drives section & group select options. Endpoints:
- `GET /sections-section-group-wise` → sections per class (JSON AJAX)
- `GET /groups-class-section-wise` → groups per section (JSON AJAX)

### Double-entry accounting
Ledger + Fund core. Vouchers (cash payment/receipt, contra, journal, fund-transfer) post journal lines with `debit`/`credit` sides inside a `UnitOfWork` transaction. Reports derive from posted lines; Cash & Fund balances derived from running balance. Service decides the operation; Generic Repository executes it.

### Fees pipeline
`FeeHead`(+sub-heads) → map to Ledger/Fund (`fees-mapping`) → `FeeAmountConfig` (class/section/category amounts per head+period) → `FeeDateConfig` (payable/fine dates) → collection (payment rows posting to ledgers) → `PaidInfo`/`UnpaidInfo` read-models (SP-backed).

### Exam pipeline
`ExamCodeList` + `GradeList` → `ExamStartup` (per class) → `MarkConfig` (per-exam % weights) → `ExamMark` rows → `ExamResult`/`GrandFinal` → tabulation/merit/result-card + certificates. Offline tally: reports compute on demand (SP query, no materialized tables).

### Attendance pipeline
Daily rows `(student,date,status)`; report aggregates presence%; exam attendance separate; staff attendance role+date. QR scanner parses student QR → insert row.

### Certificate layouts
Typed layout screens `layout-cert?type={slug}` render printable HTML+CSS → print view; `certificate-templates` supports custom templates via placeholder maps. PDF generation lives in `MightySchool.Application/Documents`.

## 5. Conventions

- Routing: attribute/conventional routes `GET/POST/DELETE /{resource}`; Tabulator/AJAX endpoints on the same Controllers; real HTTP verbs for DELETE.
- Antiforgery: `@Html.AntiForgeryToken()` in forms; `[AutoValidateAntiforgeryToken]`; AJAX sends `RequestVerificationToken` header.
- Validation: ViewModel DataAnnotations + ModelState in the Web layer; business validation in Application Services.
- Naming: PascalCase C# models (`entities.md`), snake_case SQL columns that mirror EF entities.
- Money: `decimal` → SQL `DECIMAL(18,2)` — pick once, enforce everywhere.
- Dates: store UTC (`DateTime`), present tenant timezone (Asia/Dhaka default).
- i18n: `Resources/en.json` + `Resources/bn.json` via `IStringLocalizer`.

## 6. Cross-cutting services (MightySchool.Application)

| Service | Responsibility |
|---|---|
| `InstituteScope` (Common) | current institute, session claims, demo flag |
| `TenantSettingsService` | merged tenant settings (cached, per tenant) |
| `FeeProcessor` | compute payable/fine/waiver per student, post to ledger |
| `AccountingService` | post journal lines, balances, reports |
| `ResultEngine` | mark aggregation, graders, grand-final, positions |
| `NotifyDispatcher` | SMS/WhatsApp/Email via a single `notifications.via` enum |
| `GoogleMeetService` | OAuth token mgmt, calendar events, recurrence |
| `AiService` | chat/writer/insights via Anthropic |
| `SmsGatewayManager` | multi-provider adapters (Bangladeshi APIs + Twilio) |
| `WhatsAppCloudApi` | Cloud API adapter, templates, retries |
| `PdfService` | certificates, ID cards, slips, admit cards |
| `PermissionService` | `GetAccessAsync(roleId, controller)` → `PageAccessVM` for UI gating |

## 7. Directory layout (ASP.NET Core solution)

```
MightySchool.sln
├── MightySchool.Web                     (ASP.NET Core MVC)
│   ├── Controllers/  Views/  wwwroot/   (JS + Tabulator + bundles)
│   ├── Common/       (PageAccessExtensions, ExceptionHelper, AjaxRequests)
│   ├── Program.cs    appsettings.json / appsettings.Development.json
│
├── MightySchool.Application              (business logic, no EF)
│   ├── Services/     (GenericCrudService, PermissionService, module services)
│   ├── Common/       (InstituteScope, ExceptionHelper, PasswordHasher)
│   └── Documents/    (Excel/CSV/PDF generation)
│
├── MightySchool.Infrastructure           (data access only)
│   ├── Data/ApplicationDbContext.cs   Configurations/   Migrations/
│   ├── Repositories/GenericRepository.cs    UnitOfWork/UnitOfWork.cs
│   └── Database/StoredProcedures/ (sp_StudentList, sp_* per module)
│
├── MightySchool.Interfaces               (contracts + ViewModels)
│   └── Services/ Repositories/ UnitOfWork/ ViewModels/ Documents/
│
└── MightySchool.Entities                 (POCO entities per entities.md)
    └── Entities/ Enums/ Common/
```

## 8. Performance notes

- Students list, at-a-glance: Tabulator AJAX → Service → `sp_StudentList` (filter/paginate/sort in SQL Server).
- Accounting reports: aggregate with indexes on `journal_lines(institute_id, ledger_id, date)`.
- Attendance report: index `(class_id, section_id, date)`.
- Dashboard KPIs: cached aggregates keyed `tenant:{id}:dash`.
- Certificate print: build HTML once, cache per student/exam.