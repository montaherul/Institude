# Multi-Product SaaS Platform — Build Documentation

Three independently sellable SaaS products — **School**, **Rent**, **Transport** — built and deployed as **one modular monolith** with a shared Core Platform. Clients subscribe to 1, 2, or 3 modules; toggling modules is a configuration/billing change, never new infrastructure.

## What is this repo?

The complete build specification for the platform (architecture, schema, APIs, module specs, design system, integrations, tenancy/billing, security, testing, deployment). Everything needed for a small team to implement it on Laravel 11 + PostgreSQL.

| Doc | Purpose |
|---|---|
| [docs-build/00-project-overview.md](./docs-build/00-project-overview.md) | Product goals, personas, module map, MVP, build order |
| [docs-build/01-system-architecture.md](./docs-build/01-system-architecture.md) | Modular monolith: Core vs modules, event bus, layers |
| [docs-build/02-database-schema.md](./docs-build/02-database-schema.md) | Single DB, schema-per-module: `core`, `school`, `rent`, `transport` (full SQL) |
| [docs-build/03-api-and-routes.md](./docs-build/03-api-and-routes.md) | Routes, API gateway / entitlement middleware, REST contract |
| [docs-build/04-authentication-and-rbac.md](./docs-build/04-authentication-and-rbac.md) | Auth, tenants, roles, permissions, tenant scoping |
| [docs-build/05-module-functional-specs.md](./docs-build/05-module-functional-specs.md) | Functional spec: Core, School, Rent, Transport |
| [docs-build/06-ui-ux-design-system.md](./docs-build/06-ui-ux-design-system.md) | Shell app, entitlement-driven navigation, tokens, components |
| [docs-build/07-integrations.md](./docs-build/07-integrations.md) | Event bus, payments, SMS/email, maps, notifications |
| [docs-build/08-multi-tenancy-and-billing.md](./docs-build/08-multi-tenancy-and-billing.md) | Plans & pricing, entitlement record, upgrades, invoicing |
| [docs-build/09-system-settings-and-operations.md](./docs-build/09-system-settings-and-operations.md) | Settings, scheduled jobs, audit, module config |
| [docs-build/10-security-and-compliance.md](./docs-build/10-security-and-compliance.md) | Tenant isolation, entitlement enforcement, hardening |
| [docs-build/11-testing-strategy.md](./docs-build/11-testing-strategy.md) | Unit/feature/E2E strategy, isolation & event-bus tests |
| [docs-build/12-deployment-and-devops.md](./docs-build/12-deployment-and-devops.md) | One-deployment model, CI/CD, mono→microservices path |

## Core decisions (see 01 for details)

- 1 codebase, 1 deployment, 1 database → **modular monolith**.
- `core.*` schema (shared) + `school.*`, `rent.*`, `transport.*` (independent).
- **Entitlements** record per tenant decides which modules/frontend menus are live.
- Modules integrate only via an **event bus** (async, never direct DB coupling).
- Cross-module references are loose (`external_ref`), never hard FKs.

## Stack

- **Backend:** Laravel 11, PHP 8.3, PostgreSQL 16 (single instance, schemas), Redis (cache/queue).
- **Frontend:** Blade + Alpine shell; module areas lazy-rendered by entitlement.
- **Events:** internal queue-backed outbox (Redis) — Redis Pub/Sub or RabbitMQ only when scale requires.
- **Billing:** Stripe provider + local invoices; webhook flips entitlements.

## Build order (from 00)

1. Core Platform (tenants, auth, entitlements, billing, notifications).
2. Build first module end-to-end against core (validates plug-in pattern).
3. Extract shared components (reporting, invoicing, uploads).
4. Add event bus when ≥2 modules exist with a real integration need.

## Licensing & note

Spec written for engineering/study purposes. Payment/tier details are suggestions — align pricing with business decisions before launch.