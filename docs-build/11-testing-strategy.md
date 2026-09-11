# 11 — Testing Strategy

## 1. Stack

- xUnit + Moq/NSubstitute for unit tests; `WebApplicationFactory<Program>` for HTTP feature tests.
- Playwright (optional) for the heavy Metronic admin UI flows that rely on cascading selects/Tabulator.
- Entity builders/fixtures (Bogus optional) per model; a 100–150 student + 20 staff seed for realistic report/tabulation tests.
- Test DB: SQL Server LocalDB (or containerized SQL Server per CI matrix) migrated with EF Core (`dotnet ef database update`) + seeded.

## 2. Pyramid

| Layer | Scope | Example cases |
|---|---|---|
| Unit | math: fees calc, fine, GPA, payroll net, seat allocation | waiver(fee) exact vs percent; grade boundary; advance return math |
| Feature | Controllers → Services → UnitOfWork → repositories (real EF against test DB) | collection posts journal lines + balances; balance-sheet totals; attendance save; migration reversion |
| Integration | services: SMS/WhatsApp/Meet/gateway (fake `HttpMessageHandler` drivers) | retry cap, status, no double-charge, OAuth refresh |
| E2E (Playwright) | login → section-cascade → quick-collection → receipt; result publish flow | whole happy paths |

## 3. Test harness

- Fake providers with recording (invoices, `IMailService → AssertSent`), gateway sandbox modes, mocked `HttpClient` (`HttpMessageHandler`) for Anthropic/Google/WhatsApp.
- Freeze time via `TimeProvider`/`ITimeclock` abstraction (or `Microsoft.Extensions.Time.Testing` FakeTimeProvider) for fine windows/result scoring.

## 4. Critical test matrix (write first)

1. Tenant isolation: user A (institute 1) cannot select/render institute-2 student/fee/ledger on any route (matrix over 8 modules).
2. Money integrity: `debits == credits` after payment, contra, journal, salary; fund & ledger updated; trial balance reconciles.
3. Fee pipeline: amount config × period × waiver × fine × paid → expected due on smart collection.
4. Exam pipeline: startup → mark config percentages → input → grand-final → merit/tabulation snapshots stable on re-run.
5. Attendance & migration: promote school-year keeps roll/fees references intact; pushback restores.
6. RBAC: role matrix (super_admin/admin/accountant/librarian/teacher/student/staff) × every `[MenuAuthorize]` route → allowed/denied + UI (`PageAccessVM`) flags.
7. Demo mode: profile email/password, general settings, system update all blocked.
8. Notifications: SMS ≤300 chars, template params resolve, failed→retry→fail cap; WhatsApp log & retry.
9. QR attendance: dup scan rejected; missing student code error path.
10. Certificate/layout: render 10 layout types + custom template for HTML print validity.

## 5. Fixtures & seed

- Seeders: platform (packages, modules, system settings), institute-owner (classes→sections→groups, subjects, exams with grade lists, fee heads + configs, ledger seed data, payroll heads, library books, hostel/transport/inventory, notices/events/banners, 100 students w/ users).
- Feature tests: ephemeral SQL Server test DB created per run (migrate + seed via `Program` + test host).

## 6. CI

- GitHub Actions: `dotnet restore` + `dotnet build`, `npm ci && npm run build` (frontend bundles), `dotnet test`, .NET analyzers/SonarAnalyzer (level: warnings-as-errors), `dotnet format --verify-no-changes`, and a `dotnet ef database update -- --seed` smoke job.
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