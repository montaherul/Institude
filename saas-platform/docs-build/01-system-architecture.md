# 01 — System Architecture

## 1. Topology (modular monolith)

```
┌─────────────────────────────── CORE PLATFORM (shared) ───────────────────────────────┐
│  Auth & Tenants │ Billing │ Entitlements │ Notifications │ Admin shell │ Audit        │
└───────────────────────────────────────────┬──────────────────────────────────────────┘
                                            │
                               API Gateway / Router
                               (tenant resolution + entitlement check)
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              ▼                             ▼                             ▼
   ┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
   │  School module   │          │   Rent module    │          │ Transport module │
   │  school.* schema │          │   rent.* schema  │          │ transport.* schema│
   └──────────────────┘          └──────────────────┘          └──────────────────┘
              │                             │                             │
              └─────────────────────────────┼─────────────────────────────┘
                                            ▼
                              Event Bus (async, outbox + Redis)
                              e.g. StudentEnrolled → Transport assigns route
```

One codebase, one deployment, one PostgreSQL instance with **schema-per-module**.

## 2. Requirement mapping (from spec §2.1)

| Component | Where it lives | Responsibility |
|---|---|---|
| Auth & Tenants | Core (`core.users`, `core.tenants`) | Login, accounts, roles, tenant records |
| Billing | Core (`core.invoices`, `core.plans`, `core.payment_gateways`) | Plans, invoices, provider integration |
| Entitlements | Core (`core.entitlements`) | Active modules + plan tier per tenant |
| Notifications | Core (`core.notifications`, templates) | Email/SMS/push, shared delivery |
| Admin/Dashboard shell | Core | UI shell; nav adapts to entitlements |
| Audit & Settings | Core (`core.audit_logs`, `core.settings`, `core.system_settings`) | Activity logs, tenant config |

## 3. Layers (Laravel 11)

```
web/ + api/ routes  →  middleware stack
    1. TenantResolution  (db connection stays same schema-scoped; sets tenant context)
    2. EntitlementGate   (deny non-entitled module routes → 404/403 "not available")
    3. Auth              (session or sanctum)
    4. Role/Permission   (Core RBAC)
  →  Controllers (module namespaced)
  →  Services (business rules)  ──publish──▶ EventBus (outbox → Redis → listeners)
  →  Models (module tables) → PostgreSQL (schema-name qualified)
```

Folder layout (`app/Modules/{School,Rent,Transport}` + `app/Core`):

```
app/
├─ Core/
│  ├─ Controllers/ (Auth, Tenants, Billing, Entitlements, Dashboard, Settings)
│  ├─ Services/    (BillingService, EntitlementService, NotifyService, AuditService)
│  ├─ Models/      (User, Tenant, Entitlement, Plan, Invoice, Notification, AuditLog)
│  ├─ Middleware/  (TenantResolution, EntitlementGate, SetupTenantRelations)
│  └─ Events/      (outbox publishing)
└─ Modules/
   ├─ School/     Controllers/ Models/ Services/ Events/ Listeners/ routes.php migrations/ database/schema
   ├─ Rent/       same
   └─ Transport/  same
database/migrations/core/  … school/ … rent/ … transport/   (create schema if not exists)
```

## 4. Entitlement flow (per request)

1. Resolve tenant (host `/subdomain` or custom domain → `core.tenants`).
2. Load `core.entitlements` into tenant context (cached, evicted on billing change).
3. Route matched; if it belongs to a module not entitled → **409/404 "module not available"** (frontend hides it; API returns error).
4. Proceed under `tenant_id` scope + RBAC.

Plan tier adds feature toggles inside an entitled module (e.g., Transport Pro unlocks GPS; School Pro unlocks results).

## 5. Event bus (integration contract)

- **Publisher** writes to `core.event_outbox` (same DB transaction as the business write → no lost events).
- **Dispatcher** (queued job, retry w/ backoff) publishes to Redis and forwards to subscribers.
- **Listeners** are small service classes per module, registered centrally (`$listen` map) so a module can be present without listeners.
- If no subscriber: event simply discarded. Shared events published only by publishers when integration matters (see 07 §1).

## 6. Domain patterns

- **Money:** `DECIMAL(15,2)`; all charge/payment writes transactional; no negative balances.
- **Loose cross-module refs:** `external_ref` column (JSON: `{module, type, id}`) e.g., `transport.subscribers.external_ref` → school student; no FK.
- **Tenant scoping:** default query scopes / routes bound to `tenant_id`.
- **Snapshots:** report tables (results, due lists) built by jobs, queried read-only.
- **Outbox = source of truth for integration**, giving exactly-once-ish dispatch via dedupe `event_key`.

## 7. Scale path (later, not now)

Split when: heavy isolated load (Telemetry/GPS), independent scaling need, fully separate deploys, or data-residency. Boundaries (schemas + events) already match service seams — migration is mechanical.