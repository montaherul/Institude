# 09 — System Settings & Operations

## 1. Settings storage

- `core.system_settings` — platform-level (default currency, billing provider, demo interval, update server URL).
- `core.settings` — per-tenant JSON blob `{values}` grouped by tab: General, Branding, Modules flags, Integrations (SMS/email/maps/AI), Billing. Secrets stored masked (see 10).

## 2. Tenant settings tabs

| Tab | Keys |
|---|---|
| General | org name, slug, address, phones, timezone, currency, language |
| Branding | logo (header/footer/favicon light+dark), primary color, tagline |
| Modules | per-module switches (read-only display; enforced by entitlements), default module landing |
| Integrations | SMTP, SMS gateway creds, maps key, WhatsApp business id, AI key |
| Billing | payment gateway selection per module (fee/invoice/transport collections) |

## 3. Platform admin operations

- **Plans & pricing management** (CRUD `core.plans`).
- **Tenant management:** create, override entitlements, suspend/extend, export data, terminate.
- **Notification templates** (email/SMS/push bodies).
- **Module registry:** registered modules list with min-version + flags; module feature toggles.

## 4. Scheduled jobs (cron → `schedule:run`)

| Job | Schedule | Behavior |
|---|---|---|
| Fee/invoice generation | daily | School fee_charges for period due; Rent monthly invoices due at period start |
| Fine accrual | daily | apply late fines per module rules |
| Notification dispatch | every 1 min | drained queue; retries (cap 3); dead-letter on fail |
| Subscription checks | hourly | expire `subscribed_until`, flag tenants suspended/past_due; notify |
| Demo data reset | per interval | wipe demo tenants' module data (trial only) |
| Backups | daily 02:00 | encrypted DB dump + media sync (see 12 §7) |
| Session GC | daily | prune stale sessions |

## 5. Audit logging

- Middleware writes audit on mutations (module/action/entity/ip). Billing webhooks write `billing` events. Search/filter UI for owner/admin/platform_admin.
- Sensitive payloads scrubbed (password fields, keys).

## 6. Monitoring hooks

- Outbox depth + notification failure rates → metrics exporter (Prometheus) or log-based alerts.
- Tenant quota usage vs `feature_limits` → preemptive upsell emails.
- Gateway webhook error dashboard (Stripe events failing → support queue).

## 7. Data hygiene

- Media GC (unreferenced uploads), log rotation (30d), event outbox cleanup of `dispatched` rows (30d), soft-delete where needed (students/leases/vehicles) for audit.