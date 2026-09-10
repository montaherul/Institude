# 03 — API & Routes

Laravel 11. CSRF on web forms; `api` routes use Sanctum tokens. Entitlement middleware applies to module route groups.

## 1. Route matrix

| Group | Prefix | Middleware | Purpose |
|---|---|---|---|
| Auth / tenant | `POST /login`, `POST /logout`, `GET /onboarding/*` | guest | login, signup + onboarding |
| Core API | `/api/v1/me`, `/api/v1/billing/*`, `/api/v1/settings` | auth, tenant | self-service |
| Module School | `/api/v1/school/*` | auth, tenant, entitlement:school | module endpoints |
| Module Rent | `/api/v1/rent/*` | auth, tenant, entitlement:rent | module endpoints |
| Module Transport | `/api/v1/transport/*` | auth, tenant, entitlement:transport | module endpoints |
| Public shell assets | `/`, `/login`, `/signup`, `/pricing` | guest | landing + shell |

## 2. Tenant + entitlement middleware

```php
// EntitlementGate: applies per module route group
Route::any('/school/{any}', fn() => null)
  ->where('any', '.*')->middleware(['auth:sanctum','tenant','entitlement:school']);

// logic:
$e = tenant()->entitlements;               // core.entitlements record (cached)
if (!$e->hasModule($module) ) abort(404, 'Module not available');
if (plan limits hit) abort(402, 'Plan limit reached');
```
Frontend fetches the same record → builds nav (see 06 §4). A Rent-only tenant receives 404 for `/school/*` and never renders those menus.

## 3. Core routes (public + self-service)

```
POST /api/v1/auth/login            {email, password}
POST /api/v1/auth/logout
GET  /api/v1/me                    → profile + entitlements (modules, plan, status)
GET  /api/v1/tenants/current       → tenant info + settings (public keys only)
POST /api/v1/billing/subscribe     {plan, billing_period, payment_method} → Stripe intent
GET  /api/v1/billing/intent        → client secret (Stripe PaymentIntent)
POST /api/v1/billing/portal        → Stripe customer portal access
GET  /api/v1/billing/invoices      → list core.invoices (filter status, period)
POST /api/v1/webhooks/stripe       (public, Stripe-signed) → billing events
GET  /api/v1/settings              → tenant settings (masked secrets)
POST /api/v1/settings              {key:value…} (validation per tab)
GET  /api/v1/notifications         → tenant notification inbox/logs
GET  /api/v1/audit                 → tenant audit logs (owner/admin only)
POST /api/v1/users                 → invite member  | GET list | PATCH role | DELETE
GET  /api/v1/roles                 → role catalog + candidates for module
POST /api/v1/onboarding           → step wizard: tenant, owner, plan, payment
```

## 4. School routes

```
GET  /api/v1/school/classes?academic_year=        POST create   PATCH /{id}   DELETE
GET  /api/v1/school/classes/{id}/sections          POST create
GET  /api/v1/school/students?class=&section=&q=    POST create   PATCH /{id}  DELETE
GET  /api/v1/school/students/{id}                  (profile + fee ledger)
GET  /api/v1/school/teachers                       POST          PATCH/DELETE
GET  /api/v1/school/attendance?date=&class=&section=    POST save {rows:[{student_id,status}]}
GET  /api/v1/school/fees/structures                POST create
POST /api/v1/school/fees/generate                  {period} → fee_charges snapshot
GET  /api/v1/school/fees/due?grade=&status=        (due list w/ fine calc)
POST /api/v1/school/fees/collect                   {student_id, period, method} → payment + receipt
GET  /api/v1/school/fees/payments                  (receipts, filter)
GET  /api/v1/school/exams                          POST create  (publish → results)
POST /api/v1/school/exams/{id}/results/input       bulk marks
GET  /api/v1/school/exams/{id}/results             (report card per student)
POST /api/v1/school/students/{id}/report-card      (generate/print)
```

## 5. Rent routes

```
GET  /api/v1/rent/properties        POST  PATCH/{id}  DELETE     (+ units nested)
POST /api/v1/rent/properties/{id}/units        {unit_no, rent_amount, deposit} 
GET  /api/v1/rent/units?status=     (vacant/occupied list)
GET  /api/v1/rent/renters           POST  PATCH/{id}  DELETE
POST /api/v1/rent/leases            {unit_id, renter_id, dates, rent, deposit} → creates lease + first invoice
PATCH /api/v1/rent/leases/{id}      (terminate/renew → status change + event)
GET  /api/v1/rent/leases?status=active&overdue=
GET  /api/v1/rent/invoices?status=  (render due/fine by lease)
POST /api/v1/rent/invoices/{id}/pay {method, amount} → rent_payment + receipt
POST /api/v1/rent/maintenance       {unit_id, title, …}  PATCH status
GET  /api/v1/rent/dashboard         (occupancy %, rent due, maintenance open)
```

## 6. Transport routes

```
GET  /api/v1/transport/vehicles    POST  PATCH/{id}  DELETE
GET  /api/v1/transport/drivers     POST  PATCH/{id}  (assign vehicle)
GET  /api/v1/transport/routes      POST  PATCH/{id}  DELETE
POST /api/v1/transport/routes/{id}/stops         {stop_no,name,lat,lng}
GET  /api/v1/transport/trips?date= POST schedule {route_id,vehicle_id,driver_id,start}
PATCH /api/v1/transport/trips/{id}/status (start/complete/cancel)
GET  /api/v1/transport/subscribers?route=&period=   POST subscribe {external_ref, …}
POST /api/v1/transport/subscribers/{id}/collect   {period, amount, method}
GET  /api/v1/transport/reports/collection   (per route/period totals)
POST /api/v1/transport/trips/{id}/attendance     {subscriber_id, boarded}
```

## 7. Webhooks / events

| Webhook | From | Handler |
|---|---|---|
| `checkout.session.completed` | Stripe | create invoice(paid) → update entitlement → emit `EntitlementChanged` |
| `invoice.payment_succeeded` | Stripe | mark paid, extend `subscribed_until`, notify |
| `invoice.payment_failed` | Stripe | mark past_due, notify, hold plan until retry |

Internal events (outbox, see 07 §1): `StudentEnrolled`, `StudentGraduated`, `LeaseSigned`, `LeaseEnded`, `TripCompleted`, `PaymentReceived:*`.

## 8. Error contract

| Code | Meaning |
|---|---|
| 200 / 201 | success / created |
| 401 | unauthenticated |
| 403 | not permitted by RBAC |
| 404 | not found **or module not entitled** (same signal to avoid enumeration) |
| 402 | plan/entitlement limit exceeded |
| 409 | state conflict (unit occupied, already collected) |
| 422 | validation (`{errors}`) |
| 429 | rate limited |

## 9. Conventions

- Pagination: `?page=1&per_page=20&search=`.
- Filters: named query params per list (documented on each endpoint).
- Responses: `{data: […] , meta: {pagination}}`; errors `{error: {code, message, details}}`.
- Tenant scope: never pass `tenant_id` from client; resolved server-side.