# 10 — Security & Compliance

## 1. AuthN/AuthZ

- Cookie authentication + antiforgery (ASP.NET Core defaults). Login matrix: email-or-phone, remember; rate-limit 5/min.
- RBAC: `roles` + `role_wise_menu_access` (per institute) + `[MenuAuthorize]`; permission-enforced menus & routes; Platform Admin bypass (isolated section via `RoleScope`).
- Password `PasswordHasher` (PBKDF2) min 8; `password_resets` single-use token, 60 min expiry.

## 2. Tenant isolation (OWASP-Critical)

- `InstituteScope` + generic repository default filter apply `institute_id` to **every** tenant query (no per-controller `Where` needed).
- Cross-tenant object access denied (404/403) — never leak existence.
- Custom-domain + sub-path both verified; host resolution whitelist.
- Cache keys namespaced by institute (`institute:{id}:...`) to prevent cross-tenant cache bleed; secret caches include per-tenant salt.

## 3. Web app hardening

- ViewModel binding (no mass assignment — entities never bound directly); DataAnnotations + business rules in Services for all input.
- Razor `@`-encodes output by default; no `@Html.Raw(...)` for user strings (certificates render via safe layout tokens only).
- IDOR checks in Show/Edit/Delete; file uploads: whitelist extensions + mime sniff + storage outside webroot, randomized names.
- Print/export/download routes auth + tenant-checked.
- Rate limits on `/login`, SMS compose, AI endpoints.
- Demo-mode mutation guard (see 04 §4) — security control AND product feature; session cookies terminated on role change.
- CSP & secure headers where proxy allows; TLS enforced.
- Signed URLs for external webhook callbacks where used.

## 4. Secrets management

- Credentials (SMTP/SMS/gateway/Anthropic/Google OAuth/barcodes) masked in UI; only last-4 shown; never in logs (stripped by Serilog filters).
- Stored encrypted where practical (Data Protection / `IDataProtector` or Azure Key Vault option).
- Rotations supported without downtime (dual-key window columns where feasible).
- `appsettings.json` holds no real secrets; `appsettings.Development.json`/user-secrets document local values; a committed `appsettings.example.json` documents required vars (equivalent of `.env.example`).

## 5. Audit & logging

- User activity logs (who/what/IP/time) for create/update/delete across admin.
- Status transitions logged (student migration, results publish, payment reversals, package enrollment, update history).
- Notification & gateway logs with retry/error content (no credentials).

## 6. Data protection

- PII minimization; student/parent records retained per tenant policy (privacy-policy page published).
- Backups encrypted; production DB access least-privilege; downloadable exports restricted to permitted roles.
- Right to erasure: hook on institute deletion; custom-domain removal nukes DNS binding.
- Encrypt secrets at rest with `IDataProtector`; connection strings never logged by `GlobalExceptionMiddleware`.

## 7. Compliance notes (Bangladesh context)

- Provide policy pages: Privacy, Terms, Refund, Cookies (routes exist); contact-form consent checkbox.
- SMS/WhatsApp compliance: consent opt-in recorded on phone book sync; unsubscribe link/HASH per message.
- Payment records immutable append-only ledger rows (`journal_lines`, voucher no duplicates).
- Financial transparency: every fee/salary/voucher posting produces two-sided accounting (audit trail); trial-balance must reconcile.

## 8. Test checklist (security)

- [ ] Antiforgery token on every POST incl. AJAX
- [ ] Login throttle + lockout enforcement
- [ ] Cross-tenant object fetch → 403/404
- [ ] IDOR on all `{id}` routes
- [ ] Privilege escalation via role/permission tamper
- [ ] XSS (Razor `@` encoding & certificate token whitelist)
- [ ] Upload mime/extension bypass attempts
- [ ] Command injection vectors on gateway/server settings
- [ ] Demo-mode locked mutations return errors
- [ ] Secrets never present in HTML/network/logs