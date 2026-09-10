# 04 — Authentication & RBAC

## 1. Authentication

- **Web shell:** session + CSRF (Laravel `web` guard).
- **API:** Sanctum tokens; `Bearer` + `X-Tenant` (slug) headers; CSRF not needed on token API.
- Login accepts email (unique per tenant) + password; optional phone login flag per tenant setting.
- Rate limit login: 5/min/IP (+ per-email lockout).
- Password: Argon2id/bcrypt, min 8; `password_resets` 60-min tokens; optional 2FA (TOTP) flag per tenant.
- `last_login_at` + audit record on login/logout.

## 2. Tenant resolution

- Web: host slug (`acme.myplatform.com` or custom domain) → `core.tenants.slug` in session/context.
- API: `X-Tenant: <slug-or-uuid>` header; validated before auth.
- Unknown/inactive tenant → 404 (avoid enumeration); suspended tenant → 403 `tenant_suspended`.
- Every authenticated request sets scoped context: `tenant_id`, `entitlements`, `features`.

## 3. Users & roles

Roles live in two scopes:
- **Platform** (`core.role_permissions` with `tenant_id NULL`): `platform_admin`, `support`, `billing`.
- **Tenant** (`tenant_id set`): `owner`, `admin`, plus per-module: School (`teacher`, `accountant`, `librarian`, `student`), Rent (`property_manager`, `renter`), Transport (`dispatcher`, `driver`).

`core.user_roles` maps users → role. A user can hold multiple roles (e.g., teacher + dispatcher when School+Rent).

## 4. Permission keys

Permissions are prefixed by module to keep namespaces clean:

```
core:       core.dashboard, core.billing.view/.manage, core.users.manage, core.settings,
            core.audit.view, core.notifications.view
school:     school.classes.*, school.students.*, school.teachers.*, school.attendance.*,
            school.fees.*, school.exams.*, school.results.view
rent:       rent.properties.*, rent.renters.*, rent.leases.*, rent.invoices.*,
            rent.maintenance.*, rent.reports.view
transport:  transport.vehicles.*, transport.drivers.*, transport.routes.*, transport.trips.*,
            transport.subscribers.*, transport.collect.*, transport.reports.view
```
Defaults: `*` wildcard inheritance for owner/admin on entitled modules. Operator roles get scoped `view/create` on their module only.

## 5. Enforcement

- Route middleware `permission:{key}` (Rails-style `@can`/Gate in controllers).
- Menu/shell reads entitlements + user roles → hides unauthorized items (defense in depth: server 403 still enforced).
- Platform-admin guard isolated from tenant modules; cannot impersonate by ID (uses explicit tenant switch).

## 6. Account lifecycle

- Signup: tenant → owner user auto-created in `active`; onboarding sets module choices.
- Invite: owner/admin invites → email w/ signed URL → sets password → role assigned.
- Suspend: billing failure or tenant suspension → owner keeps login, operators blocked (`403 tenant_suspended`).
- Deletion: tenant deletion cascades module data (module records then Core cascades) after 30-day grace.

## 7. Demo/onboarding mode (mirrors previous product behavior convention)

- Onboarded "trial" tenants get sample data generators per module; sensitive settings guarded until plan active.
- Trial tenants can't change billing/entitlement → prompt to subscribe.

Security notes (cross-ref 10): sessions table wired for device listing; logout everywhere on password change; role revocation terminates active tokens.