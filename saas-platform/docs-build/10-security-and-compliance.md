# 10 — Security & Compliance

## 1. Tenancy isolation (most important)

- Every module query scoped `tenant_id` (repository scoping + scoped services). IDOR test matrix across all module routes.
- Cross-tenant access → 404 (no existence leak). `X-Tenant` header validated; host resolution whitelist.
- Cache keys namespaced `t:{id}:{key}`; entitlements/settings cached per tenant; secrets never cached in shared keys.
- **Demo/demo-like** tenants share no internal data path (generators only).

## 2. Authentication & session

- Argon2id / ASP.NET Core Identity (PBKDF2) min 8; login throttle 5/min; 2FA TOTP optional per tenant.
- Sessions table for device revocation; token/cookie rotation on password change.
- CSRF (antiforgery tokens) on web; JWT Bearer tokens on API; webhook authenticity via Stripe signatures + idempotency.

## 3. Authorization

- Entitlement gate (module-level) + RBAC (key-level) layered; `platform_admin` isolated scope.
- Plan-limit enforcement server-side (402) — never rely on frontend hiding alone.
- Role changes revoke active tokens; suspended tenants blocked middleware-wide.

## 4. Web app hardening

- ViewModel binding only (no mass assignment); DataAnnotations validation on ViewModels (typed, numbered); Razor HTML-encodes output by default; no raw user HTML except sanitized rich-text editor output.
- Uploads: whitelist + mime sniff + storage outside webroot + random names; size caps.
- Rate limits on login, SMS, and AI routes; CSP + HSTS + X-Frame-Options DENY + Referrer-Policy at CDN/LB.
- Signed URLs for any reset/download links.

## 5. Secrets

- Stripe/keys/SMTP/SMS/AI masked in UI (last4), stripped from logs & audit payloads; dual-key rotation; secrets via .NET user-secrets (dev) and a secret store (prod) — never committed.
- Gateway config (tenant's module payment) masked; test/live separated.

## 6. Financial integrity (money flows)

- Both-sided ledger mental model: fee/invoice charges vs payments tracked with statuses; receipts immutable (invoice_no); refunds recorded not deleted.
- Stripe events idempotent; invoice status qualifies by `txn_id`; `past_due`/`suspended` lifecycle audited.
- All money columns `NUMERIC(15,2)`; sums computed via SQL not float.

## 7. Data protection & retention

- PII minimized (guardian/renter only what's needed); 30-day soft-delete grace; export before cascade delete; policy pages (privacy/terms/refund) published.
- Right-to-erasure hook per tenant; billing records retained per law.
- Consent flags captured at onboarding for SMS/push per module (unsubscribe honored).

## 8. Audit & compliance checklist

- [ ] Tenant isolation test matrix (all modules, all roles) — CI gate
- [ ] Entitlement gate cannot be bypassed via direct deep links
- [ ] IDOR matrix clean
- [ ] XSS/XSS in import & template renderer
- [ ] Upload bypass attempts tested
- [ ] Stripe webhook replay-safe
- [ ] Money write transactions rollback on failure (fee collect, lease invoice, subscriptions)
- [ ] Secrets absent from HTML/network/logs
- [ ] Rate-limit enforcement verified
- [ ] Retroactive module downgrade verified to leave data intact but inaccessible