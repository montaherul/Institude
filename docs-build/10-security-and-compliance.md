# 10 — Security & Compliance

## 1. AuthN/AuthZ

- Session + CSRF (Laravel defaults). Login matrix: email-or-phone, remember; throttle 5/min.
- RBAC via roles table + permissions; permission-enforced menus & routes; `super_admin` bypass (isolated section).
- Password Argon2id/bcrypt min 8; `password_resets` token expiry 60m.

## 2. Tenant isolation (OWASP-Critical)

- `institute_id` scoping on **every** tenant query (controller-level + repository default where).
- Cross-tenant object access denied (404/403) — never leak existence.
- Custom-domain + sub-path both verified; host resolution whitelist.
- Cache keys namespaced by institute (`institute:{id}:...`) to prevent cross-tenant cache bleed; secret caches include per-tenant salt.

## 3. Web app hardening

- Mass assignment off via `$guarded`; FormRequest validation for all input; `old()` not echoed without escaping.
- Blade auto-escapes; no `{!! $u !!}` for user strings (certificates render via safe layout tokens only).
- IDOR checks in Show/Edit/Delete; file uploads: whitelist extensions + mime sniff + storage outside webroot, randomize names.
- Print/export/download routes auth + tenant-checked.
- Rate limits on `/login`, SMS compose, AI endpoints.
- Demo-mode mutation guard (see 04 §4) — security control AND product feature; sessions terminated on role change.
- CSP & secure headers where proxy allows; TLS enforced.
- Signed URLs for external webhook callbacks where used.

## 4. Secrets management

- Credentials (SMTP/SMS/gateway/Anthropic/Google OAuth/String barcodes) masked in UI; only last-4 shown; never in logs (strip from logs).
- Rotations supported without downtime (dual-key window columns where feasible).
- `.env` never committed; `.env.example` documents required vars.

## 5. Audit & logging

- User activity logs (who/what/IP/time) for create/update/delete across admin.
- Status transitions logged (student migration, results publish, payment reversals, package enrollment, update history).
- Notification & gateway logs with retry/error content (no credentials).

## 6. Data protection

- PII minimization; student/parent records retained per tenant policy (privacy-policy page published).
- Backups encrypted; production DB access least-privilege; downloadable exports restricted to permitted roles.
- Right to erasure: hook on institute deletion; custom-domain removal nukes DNS binding.

## 7. Compliance notes (Bangladesh context)

- Provide policy pages: Privacy, Terms, Refund, Cookies (routes exist); contact-form consent checkbox.
- SMS/Microsoft compliance: consent opt-in recorded on phone book sync; unsubcribe link/HASH per message.
- Payment records immutable append-only ledger rows (`journal_lines`, voucher no duplicates).
- Financial transparency: every fee/salary/voucher posting produces two-sided accounting (audit trail); trial-balance must reconcile.

## 8. Test checklist (security)

- [ ] CSRF on every POST incl. AJAX
- [ ] Login throttle + lockout enforcement
- [ ] Cross-tenant object fetch → 403/404
- [ ] IDOR on all `{id}` routes
- [ ] Privilege escalation via role/permission tamper
- [ ] XSS (Blade escape & certificate token whitelist)
- [ ] Upload mime/extension bypass attempts
- [ ] Command injection vectors on gateway/server settings
- [ ] Demo-mode locked mutations return errors
- [ ] Secrets never present in HTML/network/logs