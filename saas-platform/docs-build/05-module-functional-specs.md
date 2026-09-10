# 05 — Module Functional Specifications

Format: **Purpose · Rules · Screens/behavior · Outputs/events**. Routes cross-ref 03; entities 02.

## 0. Core Platform

- **Tenant onboarding wizard:** org info → owner account → select module combo → plan → payment (Stripe) → provisioning (settings defaults, demo data optional).
- **Entitlements:** read record → build module list; module toggle only via billing (upgrade/downgrade) or platform admin override (audited).
- **Billing:** plans (08), Stripe intents/portal/webhooks, invoices list + PDF, `subscribed_until` enforcement (expired → read-only).
- **Notifications:** template engine (`{placeholders}`), channel dispatch (email/SMS/push), retry 3×, inbox viewer; logs audit.
- **Admin shell:** nav built from entitlements + role permissions; global search; per-tenant settings (general, branding, integrations).
- **Audit:** create/update/delete + auth/billing events; IP, user, module, entity, payload diff.

## 1. School module

**Purpose:** run classes, track students/teachers, attendance, fee billing, results.

Rules & behavior:
- Classes + academic year; sections; subjects per class.
- Student admission: roll auto next in class/section; unique admission_no; guardian info mandatory; promote (change class/section) keeping ledger history; statuses active → inactive/passed_out.
- Attendance daily grid (class/section/date) P/A/L; summary % per student/period; marked_by recorded.
- Fees: structures (name, amount, frequency) → generate monthly `fee_charges` (snapshot) with due_date; fine rules; waivers (percent/amount); partial vs full payments; **collect posts payment + marks `fee_charges` paid**; receipts (invoice_no) + gateway capture.
- Dues: filter by class/status; auto fine accumulate after due date; payment plan toggle.
- Exams: create exam (classes, subjects, total/pass), marks input per student/subject, auto grade+point from grade band, published results → report card PDF.
- Outputs: attendance reports, fee due/paid statements, result sheets, receipts; **events:** `StudentEnrolled` (on admission), `StudentGraduated` (on pass_out), `PaymentReceived:school`.

## 2. Rent module

**Purpose:** property/unit portfolio, leases, monthly rent collections, maintenance.

Rules & behavior:
- Properties (name/address/type/units); Units per property with rent_amount, deposit, status **vacant/occupied/maintenance** (capacity: 1 lease per occupied unit — conflicts rejected 409).
- Renters profile + lease: start/end dates, notice days, deposit; default invoice generation for period at lease start; renew/terminate workflows (status transitions + notifications).
- Invoices: monthly `lease_invoices`; **fine after due_date**; pay → `rent_payment` + receipt; overdue list.
- Maintenance: create request (unit/renter), priority, status flow, cost + charged_to; closed requests feed owner dashboard.
- Dashboard KPIs: occupancy %, units by status, rent due (current/overdue), maintenance open count, cash collected YTD.
- Outputs: rent roll statement, receipts, lease agreements (PDF). **Events:** `LeaseSigned`, `LeaseEnded` (on terminate/expire), `PaymentReceived:rent`.

## 3. Transport module

**Purpose:** fleet, drivers, routes/stops, trip scheduling, subscriber fares.

Rules & behavior:
- Vehicles (plate unique, seats, status/fuel); drivers + license + vehicle assignment; maintenance/fuel logs; expiry alerts (`insurance_date`, `reg_expiry`).
- Routes + ordered stops (lat/lng) + distance/ETA.
- Trips: schedule by date, assign route/vehicle/driver; start→complete/cancel; boarding roll-call per trip.
- Subscribers: external_ref (e.g., school.student) OR adhoc member; route/stop, period fee; monthly collections per subscriber (receipts, route/period reports).
- Integration (when School also entitled): `StudentEnrolled` listener auto-offers route → creates subscriber suggestion list; `StudentGraduated` removes/deactivates.
- Occupancy check: boardings ≤ vehicle seats.
- Outputs: trip sheet, collection report, route capacity report. **Events:** `TripCompleted` (consumed by School for attendance cross-check when entitled).

## 4. Cross-module integration behavior

| Publisher | Event | Consumer (opt-in, per entitlement) |
|---|---|---|
| School | StudentEnrolled | Transport → prepopulate subscriber pool |
| School | StudentGraduated | Transport → deactivate subscribers |
| Rent | LeaseSigned | Transport → optional move-in logistics |
| Rent | LeaseEnded | Transport → optionally end route subscription |
| Transport | TripCompleted | School → attendance cross-check |
| Core | EntitlementChanged | Any module → re-cache nav/config; disable module data views absent entitlement |

Implementation rule: listeners check consumer entitlement before acting; missing consumer = no-op (module independence guaranteed).

## 5. Notifications inventory

`invoice_created`, `payment_received`, `payment_failed`, `lease_due`, `rent_overdue`, `maintenance_update`, `trip_reminder`, `attendance_summary`, `result_published`, `module_suspended`, `welcome_tenant`, `password_reset`.