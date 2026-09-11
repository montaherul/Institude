# 04 — Authentication & RBAC

## 1. Authentication

- **Mode:** cookie authentication (`AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)`) in `MightySchool.Web` + antiforgery.
- **Login screen:** `/login` — accepts **email OR phone** + password; `remember` support (`AuthenticationProperties.IsPersistent`).
- **Login page behavior in demo:** renders quick-fill role buttons that auto-fill demo credentials and submit (reference embeds them in JS).
- **Rate limiting:** at least 5 attempts/minute on `/login` (return 429).
- **Session/identity:** the principal carries `UserId`, `RoleId`, `RoleScope` (Platform/Institute), and `InstituteId` claims; `InstituteScope` (Application/Common) resolves the current institute + demo flag. Idle timeout default 120 min, absolute 8 h.
- **Password policy:** min 8, hashed with `PasswordHasher` (PBKDF2) in `MightySchool.Application/Common`; optional reset via `password_resets` table.

## 2. Roles

| Role key | Notes |
|---|---|
| `super_admin` (Platform Admin) | Platform owner: institutes, packages, users(= tenants), roles, system settings/updates/modules, custom domains, payment gateways. Detected via `RoleScope = Platform` claim. |
| `admin` / `institute_admin` | Tenant owner: everything inside the tenant |
| `accountant` | fees, accounting, payroll, reports |
| `librarian` | library module |
| `teacher` | marks, attendance, assignments, routines, meets |
| `student` | student self-service views (results, fees, meetings) |
| `staff` | HR staff view (attendance, payroll slips) |

Demo accounts on reference: superadmin, accountant, librarian, teacher1, student (password `12345678`).

Institutes seed identically-named operational roles; the **Platform Admin role** is the single global role and is the RBAC source of truth (never match by name — see AGENTS §40).

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
system.*          system settings/updates/modules (super_admin only)
logs.view         user-activity
```

### Enforcement
- Controller gate: `[MenuAuthorize]` attribute (`Permission = "Create" | "Edit" | "Delete" | "Print" | "Export"`) — coarse server-side backstop.
- UI gate (belt-and-suspenders): `IPermissionService.GetAccessAsync(roleId, controller)` → `PageAccessVM`; views call `User.GetPageAccessAsync(...)` via `PageAccessExtensions` to hide buttons per role.
- Platform Admin → `PageAccessVM.FullAccess`; institute roles map to their `RoleWiseMenuAccess` row (seeded per institute).
- Controllers stay thin: they call services; permission checks live in `PermissionService` (Application) + `MenuAuthorize` (Web).

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

## 6. Session & antiforgery details

- Antiforgery applies to all state-changing requests; Razor forms render it automatically (`@Html.AntiForgeryToken()`); AJAX reads the token from `<meta name="csrf-token">` and sends `RequestVerificationToken` header.
- Logout: `POST /logout` (antiforgery protected, not GET) — per reference.
- `sessions` table tracks device; `user_activity_logs` records IP + action (see admin "User Activities").

## 7. Security-first notes (attacks)

- Login throttle + lockout.
- No role/IDOR: `InstituteScope` ensures the object's `institute_id` equals the current tenant on every fetch (generic repository default filter).
- ViewModel binding (no mass assignment) + DataAnnotations validation for all inputs.
- Print/export routes under auth + tenant check.
- Secrets (SMTP/SMS/Google/AI keys) masked on read, never echoed in logs.
- Reuse demo guard for any "locked" environment.