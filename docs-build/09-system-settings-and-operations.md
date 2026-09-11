# 09 — System Settings & Operations

## 1. Settings storage

- `system_settings` (platform, JSON values) — read via cached `SettingsService` (e.g. `GetAsync(key)`), multi-level cache (`IMemoryCache`/Redis) keyed globally.
- `general_settings` (single row per institute, ~150 JSON keys, grouped) — accessed via `TenantSettingsService` (Application); tenant cache key `institute:{id}:settings`.
- Keys surfaced on UI without echo of secrets (mask; only last-4 shown).

## 2. General Settings modules (per-tenant)

| Tab | Keys |
|---|---|
| General | site_title, tagline, address, phone, email, currency, academic_year default, timezone, demo_mode |
| SEO | meta title/desc/keywords, og image |
| Social | FB/Twitter/Instagram/YouTube/LinkedIn |
| Mail | SMTP host/port/username/password/secure/from |
| SMS | provider, SMS API key, phone number, sender |
| Payment | gateway creds + test/live mode |
| App | store links |
| Theme | primary/secondary/text/sidebar colors, font, logo (header/footer light/dark), favicon |
| (Meet/Zoom) | OAuth creds |

Controllers render the tabbed form; `TenantSettingsService.UpdateAsync(tab, model)` (Application) persists the single row; `MightySchool.Web` never writes DbContext directly.

## 3. Platform operations (super admin)

- **Modules registry:** `modules` list + `institute_modules` enable/disable per institute → menus hide (checked by `PermissionService`/nav rendering).
- **System Update:** upload update archive (`SystemUpdate`) → validate zip → extract into `MightySchool.Web`/Application assemblies → apply `dotnet ef database update` migrations → restart workers → history log (`/system/update/history`, `system_update_history` table: version, applied time, whether full install).
- **System information:** .NET runtime / SQL Server version, duplicate-file checks, new version notification from remote (update server URL configured).
- **Automations (`IHostedService` background workers)**
  - Attendance/fee fine application on due dates (date-config).
  - Notification queues (SMS/WhatsApp/email) dispatch + retry on logs.
  - Session cleanup; backups; demo auto-reset.
  - Scheduling: hosted-service timers (or Hangfire) with per-job intervals; production uses one worker process per replica.

## 4. Admin tooling

- `/institute-cache-clear` (per tenant) purges cached settings/menus.
- `/user-logs` — activity audit (user, action, detail, IP) via `UserActivityLogService`.
- `/profile` — name/email/photo/password; blocked in demo mode (only photo).
- Print-preview routes are GET (antiforgery not required for idempotent GET print views).

## 5. Scheduled jobs spec

| Job | Schedule | Notes |
|---|---|---|
| Fine application | daily | compute fines vs attendance waivers + date-config |
| Notification retry | every 15m | bump retries until cap then mark failed |
| DB backup | daily 02:00 | `BACKUP DATABASE` / `sqlcmd`, encrypted, retain 30d |
| Session GC | daily | prune expired `sessions` rows |
| Demo reset | per interval | recreate/wipe muted tenants (`DemoResetJob`) |
| Rental check | hourly | mark expired institutes read-only |

## 6. Logging & observability

- `ILogger` across all layers; optional **Serilog** sinks (file daily rotation/Console/Application Insights).
- Request logging (middleware) into `user_activity_logs` for admin actions; `sms_sent_logs`, `whats_app_logs`, subscription invocations logged.
- Structured logging always includes `institute_id` in scope for cross-tenant audit (never passwords/tokens).
- `GlobalExceptionMiddleware` (Web) intercepts AJAX exceptions → JSON `{ error }`; non-AJAX → `/Home/ServerError`.

## 7. Environment & deployment config

- `appsettings.json` + `appsettings.Production.json` (see 12), overridable via environment variables / user-secrets in dev:
  - `ConnectionStrings:Default` (SQL Server)
  - `Cache:Redis` (optional; in-memory fallback), `Caching` tenant keys
  - `Mail:SMTP_*`, `Sms:Gateway/Key`, `WhatsApp:AccessKey`, `Anthropic:ApiKey`, `GoogleMeet:ClientId/Secret`, `Payment:*`
  - `Storage:Provider` (local `wwwroot/uploads` | S3), `App:Timezone` (default Asia/Dhaka)
- Background worker runs from the same web application (`IHostedService`); no separate queue daemon required for v1.