# Mighty School — Full Project Build

A white-label, multi-tenant **institute / LMS management system** (schools, colleges, training centers) — spec'd from a read-only audit of the live reference deployment at `https://institute.bdboibazer.com/`.

## What is this repo?

This is the **specification / build documentation set**. It contains everything needed to rebuild the full product:

| Doc | Purpose |
|---|---|
| [DOCUMENTATION.md](./DOCUMENTATION.md) | Read-only audit of reference site (routes, screens, integrations) |
| [entities.md](./entities.md) | ~190 .NET entity inventory (behavioral data model, presented as ASP.NET Core models) |
| [AGENTS.md](./AGENTS.md) | Mandatory architecture & coding rules (ASP.NET Core + SQL Server) |
| [docs-build/00-project-overview.md](./docs-build/00-project-overview.md) | Product scope, personas, modules, roadmap |
| [docs-build/01-system-architecture.md](./docs-build/01-system-architecture.md) | Tech stack, layered architecture, patterns |
| [docs-build/02-database-schema.md](./docs-build/02-database-schema.md) | Full relational schema (tables, columns, keys, indexes) |
| [docs-build/03-api-and-routes.md](./docs-build/03-api-and-routes.md) | All web routes + endpoints (+ REST API contract) |
| [docs-build/04-authentication-and-rbac.md](./docs-build/04-authentication-and-rbac.md) | Auth, roles, permissions, demo logins |
| [docs-build/05-module-functional-specs.md](./docs-build/05-module-functional-specs.md) | Functional spec per module (25 groups / 180+ screens) |
| [docs-build/06-ui-ux-design-system.md](./docs-build/06-ui-ux-design-system.md) | Design system, layouts, components, i18n |
| [docs-build/07-integrations.md](./docs-build/07-integrations.md) | SMS, WhatsApp, email, payments, Google Meet, Zoom, AI, analytics |
| [docs-build/08-multi-tenancy-and-billing.md](./docs-build/08-multi-tenancy-and-billing.md) | Tenants, packages, custom domains, demo controls |
| [docs-build/09-system-settings-and-operations.md](./docs-build/09-system-settings-and-operations.md) | Tenant settings, system settings, updates, cache, language |
| [docs-build/10-security-and-compliance.md](./docs-build/10-security-and-compliance.md) | Security controls, CSRF, RBAC, secrets, compliance |
| [docs-build/11-testing-strategy.md](./docs-build/11-testing-strategy.md) | Unit/feature/E2E test strategy + fixtures |
| [docs-build/12-deployment-and-devops.md](./docs-build/12-deployment-and-devops.md) | Environments, CI/CD, infra, observability |

## Reference product facts (from audit)

- Front office: Tailwind + Alpine marketing site (Home/Gallery/Events/Academics/Contact/Legal)
- Back office: Metronic-style admin, 25 sidebar module groups, ~180–190 screens
- Roles: Super Admin, Accountant, Librarian, Teacher, Student (+ Staff)
- Backend capabilities: double-entry accounting, payroll, exams & results, library, hostel, transport, inventory, fees & waivers, SMS/WhatsApp, Google Meet, certificate printing, AI assistant
- Demo constraints on reference tenant: password/email/profile/system-settings/upgrades locked

## Build order

1. Foundation: ASP.NET Core MVC app (MightySchool.Web) + auth/RBAC + multi-tenancy + settings
2. Academic core: config (year/shift/class/section/group/subject), students, staff, attendance
3. Money core: fees, accounting, payroll
4. Academics: routine/syllabus, exams/results/certificates
5. Campus: library, hostel, transport, inventory
6. Communication: SMS, WhatsApp, notifications, Google Meet
7. Growth: CMS front office, AI assistant, mobile-app content
8. Ops: system settings, updates, caching, languages

## Licensing & note

Spec derived from a public demo by FueDevs LTD (product "Mighty School"). Rebuilds must respect applicable licensing/ownership. Contents are for engineering/study purposes.