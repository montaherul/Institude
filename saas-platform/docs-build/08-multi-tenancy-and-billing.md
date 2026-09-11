# 08 — Multi-Tenancy & Billing

## 1. Tenancy model

- Single SQL Server database, shared schemas, **row-level tenant scoping** — every table carries `tenant_id`.
- Tenant resolved from host slug or custom domain; context bound per request.
- No cross-tenant FK; module tables never read other tenants (scoped queries + repository scoping + tests).

## 2. Plans (recommended baseline — adjust to pricing decision)

| Plan | Modules | Price (USD, illustrative) | Limits |
|---|---|---|---|
| Free/trial | any 1 module | $0 (14 days) | 30 students / 10 properties / 5 vehicles |
| Growth | any module | $25/mo per module | 500 students / 100 properties / 25 vehicles |
| Pro | all 3 | $60/mo | unlimited base + integrations (GPS, WhatsApp, AI) |

- `core.plans` stores allowed `modules` + `feature_limits` JSON — single source for both entitlement AND pricing UI.
- Per-module add-on: `per_module_price` (So All-three vs 1+1+1 configurable by business).

## 3. Entitlement record (single source of truth)

```
tenant_id   → acme-school-001
modules     → ["school","transport"]
plan        → growth
status      → active
subscribed_until → 2026-10-01
```
- Read on every request (cached, invalidated by `EntitlementChanged` event / cache clear).
- **Frontend shell** same record → menus; API gateway middleware (entitlement check) on module route groups.
- Safe mutation only via: billing success (webhook) or `platform_admin` override (audited + logged).

## 4. Add / remove a module ⇒ config only

1. Tenant goes to Billing → "Add Transport" → Stripe checkout for module price (+ plan delta).
2. Webhook `checkout.session.completed` → `EntitlementService.GrantModule(tenant, "transport")` → record modules JSON + audit + notify owner.
3. Next page loads: shell nav includes Transport, `transport.*` routes now pass the gate. **No redeploy, no new infra.**

Downgrade path: webhook/customer portal cancel → grace period (till period end) → module removed; module data retained but gated behind reactivation (no destructive deletes).

## 5. Invoicing & lifecycle

- `core.invoices` generated at plan/module change; PDF receipt; payment method Stripe; next billing date + outstanding shown in billing page.
- States: draft→open→paid/past_due/void. Failed payment: notify ×3, retries via Stripe, then `suspended` (read-only) with portal resume.

## 6. Tenants operations (platform admin)

| Ops | What happens |
|---|---|
| Force module add/remove | entitlement row update + `EntitlementChanged` + audit |
| Suspend/expire | shell read-only banner; operators blocked; job rechecks daily |
| Provision demo tenant | trial entitlement + demo generator per selected module |
| Delete tenant | grace 30d → cascade module rows → core rows → custom domain unlinked |

## 7. Sizing & caching

- Scoped queries cache per-tenant key: `t:{tenant_id}:entitlements`, `:settings`, `:roles`.
- Clear: `POST /api/v1/tenants/current/cache-clear` (owner/admin), or event-driven invalidation.
- Tables indexed on `tenant_id` (SQL Server query optimizer picks index ranges; grows horizontally until service split — see 12 §8).