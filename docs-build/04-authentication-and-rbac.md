# 04 — Authentication & RBAC

## 1. Authentication

- **Mode:** session-based cookies (Laravel `web` guard) + CSRF.
- **Login screen:** `/login` — accepts **email OR phone** + password; `remember` token support.
- **Login page behavior in demo:** renders quick-fill role buttons that auto-fill demo credentials and submit (reference embeds them in JS).
- **Rate limiting:** at least 5 attempts/minute on `/login` (return 429).
- **Session:** tenant-aware; storing `institute_id` + `demo_mode`; idle & absolute timeout configurable (default 120 min idle, 8 h absolute).
- **Password policy:** min 8, hashed Argon2id/bcrypt, optional reset via `password_resets`.

## 2. Roles

| Role key | Notes |
|---|---|
| `super_admin` | Platform owner: institutes, packages, users(= tenants), roles, system settings/updates/modules, custom domains, payment gateways |
| `admin` / `institute_admin` | Tenant owner: everything inside the tenant |
| `accountant` | fees, accounting, payroll, reports |
| `librarian` | library module |
| `teacher` | marks, attendance, assignments, routines, meets |
| `student` | student self-service views (results, fees, meetings) |
| `staff` | HR staff view (attendance, payroll slips) |

Demo accounts on reference: superadmin, accountant, librarian, teacher1, student (password `12345678`).

## 3. Permission model

Reference shows a **"Role with Permissions"** screen (`/roles`) listing permissions per role. Recommended permission keys:

```
dashboard.view
students.*        student.view/create/update/delete/migrate
staff.*           staff.view/create/update/delete/attendance
attendance.view/update  exams.view
academics.*       class/section/subject/config.view+manage
fees.*            fee-head/amount/waiver/collect/paid/unpaid
accounts.*        ledger/fund/voucher/view/chart  reports.view
payroll.*         heads/mapping/assign/process/payment
routine.*         syllabus/assignment/class-routine/exam-routine
library.*         category/book/member/issue
exam.*            startup/mark-config/mark-input/result/tabulation/merit/online
certificate.*     templates/issue
sms.*            sms-templates/phonebook/compose/purchase
whatsapp.*       settings/templates/logs
hostel.*          category/member/building/room/bed/meal/bill/collection
transport.*       bus/driver/route/stop/member/collection
inventory.*      category/item/sale
meet.*            view/create/delete/notify
cms.*             pages/banners/faq/gallery/testimonials/policies/admission
users.*           tenant users & roles
system.*          system settings/updates/modules (super_admin)
logs.view         user-activity
```

### Enforcement
- Middleware chain: `auth` → `role:{role}` → permissions checked inline (Gate/`@can`).
- `super_admin` inherits all.
- Controllers use `authorize()`; admin UI hides/disabled nav items not permitted (menu rendered from permission set).

## 4. Demo mode guard

When `institute.demo_mode = 1`:
- Block: `profile` email/password change, `general_settings` update, `system/update`, signup of real users.
- Allow: all read + data-create (so visitors can try the flows).
- Dashboard banner: "Demo Mode: Password, email, profile, system settings, and system upgrades cannot be changed."

## 5. Account provisioning

- Super admin creates institutes → creates first admin user (owner) for the tenant.
- Tenant admin creates users of other roles (`/users`) and assigns `role_id`.
- Student/staff records create (or link to) `users` with `user_type` matching role; library member links to user.
- Online admission applications become pending students until approved (`/admission-forms`).

## 6. Session & CSRF details

- `VerifyCsrfToken` applies to all POSTs; token exposed via `<meta name="csrf-token">` for AJAX.
- Logout: `POST /logout` (CSRF protected, not GET) — per reference.
- `Sessions` table tracks device; `user_activity_logs` records IP + action (see admin "User Activities").

## 7. Security-first notes (attacks)

- Login throttle + lockout.
- No role/IDOR: verify object `institute_id` equals session tenant on every fetch.
- Mass assignment protection & request validation for all inputs.
- Print/export routes under auth.
- Secrets (SMTP/SMS/Google/AI keys) masked on read, never echoed in logs.
- Reuse demo guard for any "locked" environment.