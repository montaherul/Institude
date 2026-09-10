# 11 — Testing Strategy

## 1. Stack

- PHPUnit + Laravel BrowserKit/HTTP feature tests; Pest optional (keep PHPUnit for parity).
- Dusk (optional) for the heavy Metronic admin UI flows that rely on cascading selects/DataTables.
- Factories + faker per entity; a 100–150 student + 20 staff seed for realistic report/tabulation tests.

## 2. Pyramid

| Layer | Scope | Example cases |
|---|---|---|
| Unit | math: fees calc, fine, GPA, payroll net, seat allocation | waiver(fee) exact vs percent; grade boundary; advance return math |
| Feature | route/controllers/validation/transactions | collection posts journal lines + balances; balance-sheet totals; attendance save; migration reversion |
| Integration | services: SMS/WhatsApp/Meet/gateway (fake drivers) | retry cap, status, no double-charge, OAuth refresh |
| E2E (Dusk) | login → section-cascade → quick-collection → receipt; result publish flow | whole happy paths |

## 3. Doctrine harness

- Fake providers with recording (invoices, `mailer→assertSent`), gateway sandbox modes, `Http::fake` for Anthropic/Google.
- Freeze time (`Carbon::setTestNow`) for fine windows/result scoring.

## 4. Critical test matrix (write first)

1. Tenant isolation: user A (institute 1) cannot select/render institute-2 student/fee/ledger on any route (matrix over 8 modules).
2. Money integrity: `debits == credits` after payment, contra, journal, salary; fund & ledger updated; trial balance reconciles.
3. Fee pipeline: amount config × period × waiver × fine × paid → expected due on smart collection.
4. Exam pipeline: startup → mark config percentages → input → grand-final → merit/tabulation snapshots stable on re-run.
5. Attendance & migration: promote school-year keeps roll/fees references intact; pushback restores.
6. RBAC: role matrix (super_admin/admin/accountant/librarian/teacher/student/staff) × every menu route → allowed/denied.
7. Demo mode: profile email/password, general settings, system update all blocked.
8. Notifications: SMS ≤300 chars, template params resolve, failed→retry→fail cap; WhatsApp log & retry.
9. QR attendance: dup scan rejected; missing student code error path.
10. Certificate/layout: render 10 layout types + custom template for HTML print validity.

## 5. Fixtures & seed

- `Seeders`: platform (packages, modules, system settings), institute-owner (classes→sections→groups, subjects, exams with grade lists, fee heads + configs, ledger seed data, payroll heads, library books, hostel/transport/inventory, notices/events/banners, 100 students w/ users).
- Feature test DB: `RefreshDatabase`.

## 6. CI

- GitHub Actions: `composer install` + `npm ci` + `npm run build`, `php artisan test`, PHPStan level 6, Pint style, `php artisan migrate --seed` smoke.
- Coverage gating per module ≥ 85%.
- Nightly full test on demo-like resource DB.

## 7. Manual QA checklist (release)

- [ ] Onboarding wizard for new institute
- [ ] Twin demo accounts login parity
- [ ] EN/BN switch persists
- [ ] All 25 sidebar groups expand + each route 200
- [ ] Dark/light theme & mobile responsiveness
- [ ] System update via archive applies cleanly
- [ ] General settings tabs save + reload