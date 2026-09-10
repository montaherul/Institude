# 00 — Project Overview

## 1. Product

**Mighty School** — white-label SaaS for schools, colleges, and training centers. One codebase, many tenants (institutes), with a public marketing site per tenant and a full back-office management suite.

Reference instance: `https://institute.bdboibazer.com/` (demo tenant, brand default).

## 2. Goals

- Manage students, staff, classes, subjects, exams and results.
- Bill & collect fees; full double-entry accounting; payroll with salary heads.
- Run daily ops: attendance (manual + QR), library, hostel, transport, inventory, meals.
- Communicate: SMS, WhatsApp, email, notices, Google Meet live classes.
- Present a CMS-driven public website (home, gallery, events, academics pages, contact, legal).
- Sell as SaaS: institutes + packages + custom domains + auto-reset demo tenants.

## 3. Personas

| Persona | Role in system |
|---|---|
| Super Admin | Owns platform + all tenants (system settings, packages, updates) |
| Institute Admin | Runs a tenant; manages everything for it |
| Accountant | Fees, accounting, payroll, reports |
| Librarian | Books, members, issues |
| Teacher | Marks, attendance, assignments, classes, meets |
| Student | View results, classes, fees status |
| Public visitor | Browse marketing site, contact, apply for admission |

## 4. Module map (sidebar groups)

1. Students Information
2. Staffs Information
3. Student Attendance
4. QR Code Attendance
5. Academic Configuration
6. Fees Management
7. Accounts Management
8. Accounting Reports
9. Payroll Management
10. Routine Management
11. Library Management
12. Exam Module
13. Layout & Certificates
14. SMS Module
15. Administrator
16. System
17. Master Configuration (multi-tenant)
18. CMS Management
19. WhatsApp
20. Hostel Management
21. Inventory
22. Transport Management
23. Google Meet
24. AI Assistant

## 5. User stories (MVP cut)

- As Admin: configure academic session, classes, sections, subjects per class/group (with optional subjects).
- As Admin: add 100+ students with roll/admission numbers, migrate across years.
- As Teacher: take daily attendance, exam attendance; enter marks section-wise.
- As Accountant: define fee heads, map to ledgers/funds, set amounts & due dates, collect via quick collection, report paid/unpaid.
- As Accountant: run P&L, balance sheet, cash flow, cash summary.
- As Admin/Accountant: payroll heads → mapping → assign → slips → process payments (incl. advance/due/return).
- As Exam team: startup exams (code+grade lists), mark config weights, input marks, results, tabulation, merit list, result-card print, certificates.
- As Librarian: categories, books+barcodes, members, issue/return.
- As Admin: hostel buildings/floors/rooms/beds, members, meals, bills, collections; transport buses/drivers/routes/stops, members, collections.
- As Admin: inventory categories/items/sales.
- As Admin: notices, events, contact messages, user logs, ID-card printing (student/teacher/staff).
- As Communicator: SMS templates/phonebook/compose/purchase/report; WhatsApp settings/templates/logs.
- As Admin: Google Meet scheduling with OAuth + recurrence + notifications.
- As Admin: CMS (pages, banners, FAQs, gallery, testimonials, policies, admission forms, mobile app sections, why-choose-us, ready-to-join-us).
- As Admin: AI assistant (chat, writer, insights) on Anthropic.
- Super Admin: institutes (install w/ package + payment), branches, users, roles+permissions, custom domains, payment gateways, system settings, updates, modules.

## 6. Non-functional requirements

- Multi-tenant isolation, tenant-aware caching.
- Demo-mode per tenant (lock profile/settings/upgrades).
- Bangla + English i18n.
- Print-ready documents (certificates, ID cards, mark sheets, admit cards, barcodes).
- Rate-limit auth; CSRF everywhere; role middleware.
- Cloudflare-friendly (robots content-signals, WAF).

## 7. Out of scope for v1

- Native mobile apps (only CMS content for apps + store links).
- Multi-language question banks for online exams.
- KPI dashboards beyond attendance/fees/income aggregates.