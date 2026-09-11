# 08 — Multi-Tenancy & Billing

## 1. Model

- **Shared database + scoped rows** (single Schema; `institute_id` column on every tenant table, index included — see 02).
- `Institutes` table: name, email, phone, address, status (`active`/`suspended`/`expired`), `demo_mode`, start date, renewal dates, subscribed until.
- One pivot `institute_packages` (package + start_date + amount_paid).

## 2. Tenant resolution

- Custom domain resolution: `custom_domains` (domain, institute_id, status, verified). Requested in `/custom-domain`, verified super admin side.
- Fallback: sub-path or `subdomain.domain.com`.
- Cookie session claims carry `InstituteId` (+ `RoleId`, `RoleScope`); `InstituteScope` middleware binds the institute and injects a tenant-scoped cache key.
- `InstituteCache` cleared via `/institute-cache-clear`.

## 3. Packages

| Package | Scope |
|---|---|
| Trial | demo, limited, auto reset |
| Standard/Pro | full module set per edition (BUP) |

- Price assignment stored in `Packages` (via super admin package CRUD implied by enrollment form).
- Enrollment admin UI posts: package, start date, amount paid, payment method → creates `institute_packages` + marks status active → confirmation requiring typed text "Institute 1" (destructive confirm pattern).

## 4. DB backups & expiry

- Backup task automation assumed (recommend: daily encrypted dump to S3).
- Subscription expiry: marked `expired` → read-only view with banner + renewal link; `demo_mode` institutes auto-reset on interval (`system.settings.demo_reset_interval_hours`).

## 5. Billing record

- `institute_packages` holds amount_paid; super admin sees payment method + start date (this anchors billing reconciliation w/ gateway `transactions` where logged).
- Future: invoice + subscription schedule.

## 6. Platform tables (super admin scope)

- `Packages`, `UserTypes`/Roles, `Institutes`, `institute_packages`, `custom_domains`, `modules` + `institute_modules` (feature toggles per tenant), `system_settings`, `general_settings` (per institute), `update histories`, `payment_gateway` per institute.
- Tenant tables get seeded defaults at creation (Admin user, settings, ledger seeded data, subscriptions rows).

## 7. Enforcement

- Every repository query runs through `InstituteScope` (generic repository default filter: `x => x.InstituteId == _instituteScope.InstituteId`); services never bypass it for tenant tables.
- Feature module toggles gate menus/routes per institute (`institute_modules`).
- Roles enforce within tenant; the Platform Admin (`RoleScope` claim) bypasses tenant scope to all institutes (explicit separate guard/section).

## 8. Administration checklist

1. Super admin creates Institute → auto-create owner admin user + default settings.
2. Enroll package (start date, amount, method, confirm).
3. Verify custom domain.
4. Tenant admin configures General Settings (logo, theme, SMS/SMTP, gateways, Meet/AI).
5. Seed master data (academic years, classes, sections, groups, subjects, fee heads, vouchers).
6. Onboarding demo seed available.