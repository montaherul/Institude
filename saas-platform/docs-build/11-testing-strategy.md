# 11 — Testing Strategy

## 1. Stack

- PHPUnit + Laravel feature/unit tests; Dusk for shell/nav entitlement E2E; factories per entity.
- Fake drivers for Stripe, SMS, email, event bus (recording outbox).

## 2. Pyramid

| Layer | Scope | Focus |
|---|---|---|
| Unit | service math (fines, pts, occupancy, balances) | fee fine calc, rent fine + deposit, transport seats, subscription math |
| Feature | controllers, validation, transactions | collect fee→ledger consistency; lease→invoice; trip schedule→attendance; entitlement gate |
| Integration | services + webhooks + outbox→listener | Stripe↔entitlement flip; SMS/email send+retry; EventBus wiring |
| E2E | shell nav, onboarding, checkout | signup→module pick→pay→nav shows modules only |

## 3. Critical tests (write first — CI gate)

1. **Tenancy isolation matrix** — tenant A cannot read/write tenant B in every module + core (substrate: generate catalog from route list).
2. **Entitlement gate** — non-entitled module route → 404; frontend nav hides only entitled.
3. **Billing** — webhook completes checkout → entitlement gains module; upgrade/downgrade flows; failed payment → suspend; **idempotent webhook replay**.
4. **Money** — fee charge/collect sums ≡ receipts; rent invoice/payment reconcile; subscription charge no double-post.
5. **Event bus** — publish in same txn as change; rollout: dispatch succeeds, listener no-op when consumer absent, dead-letter on failure.
6. **Integration** — StudentEnrolled → transport pool (only when transport entitled); StudentGraduated → deactivate.
7. **RBAC** — role matrix × module endpoints (allowed/denied).
8. **Media/upload** — extension/mime/size guard.
9. **Rate limits** — login, sms, ai.

## 4. Fixtures & seeds

- `faker`-based tenants (2 tenants), owner/admin/operator users; module fixtures: school (120 students/2 classes), rent (3 properties/12 units/several leases), transport (5 vehicles/3 routes/40 subscribers).
- Freeze time for fine/period logic · `RefreshDatabase` per suite.

## 5. CI pipeline

`composer lint` (Pint) · `phpstan level 6` · `npm build` · `php artisan test --testsuite=isolation,billing,money,modules` · artifact coverage ≥ 85%.

- Seeded "isolation" suite runs on every PR; billing/money on webhook/trigger PRs.

## 6. Manual QA checklist (release)

- [ ] Fresh onboarding: Free→Growth→Pro combos (all 7 combos)
- [ ] Same-tenant multi-module nav + cross-integration live
- [ ] Downgrade keeps data locked + restore works
- [ ] Permission instruct display per role
- [ ] i18n EN/BN; dark/light theme; mobile
- [ ] Receipt/PDF printing (fee, rent, report cards)
- [ ] All webhook flows via Stripe test mode