# 01 — System Architecture

## 1. Stack (reference-equivalent)

| Layer | Choice | Why |
|---|---|---|
| Backend | **Laravel (PHP 8.2+)** | Matches reported framework; Blade + Eloquent + Passport/Sanctum |
| DB | **MySQL 8** | transactional; supports multi-tenant + double-entry |
| Front public | Tailwind CSS + Alpine.js + vanilla JS | SEO-friendly SSR |
| Admin UI | Metronic-style theme (v8 class conventions) + jQuery 3.7 + Alpine + Select2 + DataTables | reference-compatible feel |
| Auth | Session-based + CSRF; roles via Spatie permissions or custom | simple role gate |
| Caching | Redis (tenant-scoped keys) | hot: sessions, settings, students list |
| Jobs/Queue | Redis driver + Horizon | SMS/WhatsApp/meet notifications, imports |
| Files | Local disk `storage/` + optional S3 | images, documents, certificates |
| Edge | Cloudflare proxy/WAF | robots.txt content-signals, email obfuscation |
| Search/pluggables | Select2 on client; no ES needed v1 | |

## 2. Logical layers

```
Browser (public SSR)         Browser (admin SPA-ish via server-rendered pages + Alpine)
        |                                |
        +------------ Cloudflare -------+
                        |
                 Laravel HTTP Kernel (web middleware)
                   - StartSession, CSRF, VerifyCsrfToken
                   - RoleMiddleware(super_admin|admin|accountant|librarian|teacher|student)
                   - TenantMiddleware(resolve institute + demo_mode)
                 |        |         |
        Controllers/FormRequests → Services → Repositories → Models/Eloquent
                 |        |         |
            Blade views (admin/public)   Eloquent → MySQL (tenant tables)
                 |
        Queues → Jobs → Third-party (SMS, WhatsApp, Gmail/GoogleMeet, Zoom, Anthropic, Gateways)
```

## 3. Multi-tenancy model

- **Single DB, shared schema** with `institute_id` discriminator on tenant-scoped tables (simplest, matches SaaS demo). Alternative: schema-per-tenant (documented in `08`).
- A global `TenantScope`/global scope injects `institute_id` filters automatically.
- `System` tables are global (no tenant): `institutes`, `packages`, `users`, `roles`, `permissions`, `institute_settings` map.
- Tenant tables (students, fees, accounts …) carry `institute_id` index.
- `demo_mode` flag per institute toggles lock behavior.

## 4. Key domain patterns

### Class → Section → Group cascade
Class drives section & group select options. Endpoints:
- `GET /sections-section-group-wise` → sections per class
- `GET /groups-class-section-wise` → groups per section

### Double-entry accounting
Ledger + Fund core. Vouchers (cash payment/receipt, contra, journal, fund-transfer) post `journal_lines` with `debit`/`credit` sides. Reports derive from posted lines; Cash & Fund balances derived from running balance.

### Fees pipeline
`FeeHead`(+sub-heads) → map to Ledger/Fund (`fees-mapping`) → `AmountConfig` (class/section/category amounts per head+period) → `DateConfig` (payable/fine dates) → collection (payment rows posting to ledgers) → `PaidInfo`/`UnpaidInfo` read-models.

### Exam pipeline
`ExamCodeList` + `GradeList` → `ExamStartup` (per class) → `MarkConfig` (per-exam % weights) → `Mark` rows → `ExamResult`/`GrandFinal` → tabulation/merit/result-card + certificates. Offline tally: reports compute on demand (query, no materialized tables).

### Attendance pipeline
Daily rows `(student,date,status)`; report aggregates presence%; exam attendance separate; staff attendance role+date. QR scanner parses student QR → insert row.

### Certificate layouts
Typed layout screens `layout-cert?type={slug}` render printable HTML+CSS → print view; `certificate-templates` supports custom templates via placeholder maps.

## 5. Conventions

- Routing: resource-style web routes `GET/POST /{resource}` and `POST /{resource}/{id}` with `_method` spoofing.
- CSRF: `_token` hidden + `VerifyCsrfToken`; meta csrf-token for JS.
- Validation: FormRequest per create/update.
- Naming: snake_case columns, singular models, plural routes.
- Money: integer minor units or decimal(15,2) — pick once, enforce in casts.
- Dates: store UTC, present tenant timezone.
- i18n: `lang/en` + `lang/bn` JSON keys.

## 6. Cross-cutting services

| Service | Responsibility |
|---|---|
| `TenantContext` | current institute, session, demo flag |
| `SettingsService` | merged tenant settings (cached, per tenant) |
| `FeeProcessor` | compute payable/fine/waiver per student, post to ledger |
| `AccountingService` | post journal lines, balances, reports |
| `ResultEngine` | mark aggregation, graders, grand-final, positions |
| `NotifyDispatcher` | SMS/WhatsApp/Email via a single `notifications.via` enum |
| `GoogleMeetService` | OAuth token mgmt, calendar events, recurrence |
| `AiService` | chat/writer/insights via Anthropic |
| `SmsGatewayManager` | multi-provider adapters (Bangladeshi APIs + Twilio) |
| `WhatsApp` | Cloud API adapter, templates, retries |
| `PdfService` | certificates, ID cards, slips, admit cards |

## 7. Directory layout (Laravel)

```
app/
  Http/Controllers/{Admin,Public,Api}
  Models/          (one per entity in 02-database-schema.md)
  Services/        (the cross-cutting list above)
  Jobs/            (SmsJob, WhatsAppJob, MeetSyncJob, ResultNotifyJob)
  Providers/
  Console/Commands (demo:reset, enroll:seed)
database/
  migrations/  seeders/  factories/
resources/
  views/  public/  langs/{en,bn}.json
routes/
  web.php  public.php  admin.php  api.php
storage/  ({banners,users,events,...})
```

## 8. Performance notes

- Students list, at-a-glance: query with filters + pagination (DataTables server-side).
- Accounting reports: aggregate with indexes on `journal_lines(institute_id, ledger_id, date)`.
- Attendance report: index `(class_id,section_id,date)`.
- Dashboard KPIs: cached aggregates keyed `tenant:{id}:dash`.
- Certificate print: build HTML once, cache per student/exam.