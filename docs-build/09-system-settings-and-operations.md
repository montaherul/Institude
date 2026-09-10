# 09 — System Settings & Operations

## 1. Settings storage

- `system_settings` (platform, JSON values) — read via cached helper `option('key', default)`.
- `general_settings` (single row per institute, ~150 JSON keys, grouped) — accessed via dedicated service `InstitutionSettings::get(key)`.
- Keys surfaced on UI without echo of secrets (mask).

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

## 3. Platform operations (super admin)

- **Modules registry:** `modules` list w/ `institute_modules` enable/disable per institute → menus hide.
- **System Update:** upload update archive → validate zip → extract into `shared package` → migrations → Vite build → history log (`system/update/history`, `update_history` table: version, updated at, whether full installed).
- **System information:** Laravel/php/DB version, duplicate-file checks, new version notification from remote (update server URL configured).
- **Automations (scheduled, via cron)**
  - Attendance/fee fine application on due dates (date-config).
  - Notification queues (SMS/WhatsApp/email) dispatch + retry on logs.
  - Session cleanup; backups; demo auto-reset.
  
## 4. Admin tooling

- `/institute-cache-clear` (per tenant) purges cached settings/menus.
- `/user-logs` — activity audit (user, action, detail, IP).
- `/profile` — name/email/photo/password; blocked in demo mode (only photo).
- Banco: printed documents print-preview routes are GET (CSRF not required).

## 5. Scheduled jobs spec

| Job | Schedule | Notes |
|---|---|---|
| Fine application | daily | compute fines vs attendance waivers + date-config |
| Notification retry | every 15m | bump retries until cap then mark failed |
| DB backup | daily 02:00 | encrypted, retain 30d |
| Session GC | daily | prune expired sessions |
| Demo reset | per interval | recreate/wipe muted tenants (`InstituteReset`) |
| Rental check | hourly | mark expired institutes read-only |

## 6. Logging & observability

- Laravel log channels: daily tenant, error-events w/ context.
- Request logging (middleware) into `user_activity_logs` for admin actions; `sms_sent_logs`, `whats_app_logs`, `subscription` invocations logged.
- Monolog context always includes institute_id for cross-tenant audit.

## 7. Environment & deployment config

- `.env`: APP_URL, DB_*, CACHE/SESSION (redis recommended for multi-tenancy), QUEUE (redis/beanstalk), MAIL_*, SMS API keys, ANTHROPIC_API_KEY, STORAGE driver, APP_TIMEZONE=DEFAULT Asia/Dhaka.
- Optional: `QUEUE_CONNECTION=database` fallback; queue workers required for notifications batch.