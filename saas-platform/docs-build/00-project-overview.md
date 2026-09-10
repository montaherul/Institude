# 00 — Project Overview

## 1. Product goals

1. A client can buy **any combination** of School, Rent, Transport.
2. Adding/removing a module for a client = **configuration change**, never code or infrastructure.
3. Shared concerns (login, tenants, billing, notifications, users) built **once**.
4. Modules may **optionally** integrate (School + Transport shares students) without depending on each other.
5. Simple enough for a small team to build and operate; clear path to scale later.

## 2. Modules & buyers

| Module | Core value | Primary buyers |
|---|---|---|
| **School** | Students, classes, teachers, attendance, fees, results | Schools, coaching/training centers |
| **Rent** | Properties, units, renters, leases, rent payments, maintenance | Landlords, property managers |
| **Transport** | Vehicles, drivers, routes, stops, trips, subscribers/fares | Schools (bus), logistics, fleet owners |

**Combos sold:** School-only · Rent-only · Transport-only · School+Rent · School+Transport · Rent+Transport · All three.

## 3. Personas

- **Platform admin** — manages tenants, plans, billing, modules toggle, system settings.
- **Tenant admin / owner** — subscribes, configures their modules, invites staff.
- **Module operators** — Per-module roles: School (teacher, accountant, student), Rent (property manager, renter), Transport (dispatcher, driver, owner).
- **End customers** — students/guardians, renters, transport riders (limited self-serve views).

## 4. MVP scope (v1)

- Core: tenant signup/onboarding, auth + RBAC, entitlements, plan checkout (Stripe) + webhook, notifications (email/SMS), settings, audit log, admin shell with entitlement-filtered nav.
- School: classes/sections, students, teachers, attendance, fee structures + payments, results (basic).
- Rent: properties/units, renters, leases, rent payments, maintenance requests.
- Transport: vehicles/drivers, routes/stops, trip scheduling, subscriber management + monthly fares.
- Event bus: internal outbox + Redis Pub/Sub; seed at least `StudentEnrolled → Transport` integration.

### Out of MVP (v1.1+)
Live GPS tracking, driver mobile app, renter portal, payroll, library/hostel (School), tenant analytics dashboards, multi-currency.

## 5. Non-functional requirements

- **Multi-tenancy:** strict row isolation via `tenant_id` on every table; entitlements checked at gateway level.
- **Modularity:** no cross-module imports/dependencies; module deletion = config, not refactor.
- **Scalability:** modular monolith on one host first; extract services only on load/team demand.
- **Reliability:** async integrations must never break core flows (outbox + retries).
- **Security:** OWASP baseline, tenant-isolation test suite mandatory.
- **Performance:** p95 < 300ms for module CRUD; DataTable-style pagination.

## 6. Delivery milestones

| M | Milestone | Exits |
|---|---|---|
| 1 | Core Platform | tenant onboarding, auth, entitlements middleware, billing webhook |
| 2 | First module (pick highest-demand) | full CRUD, entitlement toggling live |
| 3 | Second module + event bus | School→Transport or Rent→Transport integration working |
| 4 | Third module + billing polish | all combos sellable; admin dashboard |
| 5 | Hardening | security tests, load test, docs, deployment runbook |

## 7. Risks

- Scope creep building all three at once → MVP = 1 module against proven core.
- Tight coupling via "quick" shared queries → enforce event bus from day one.
- Billing → entitlement drift → single source of truth (`entitlements` row) + webhook handler idempotent.