# Entity Inventory — `https://institute.bdboibazer.com/` (Mighty School)

> **Derived from** read-only audit of the live demo (login as superadmin). No database/source access exists from outside, so this is a **behavioral entity model**: every entity + property is inferred from admin screens, forms, table columns and endpoints.
>
> The inventory is presented as **ASP.NET Core EF Core entity models** for the `MightySchool.Entities` project (see `AGENTS.md`). Every entity is a C# POCO class, attributes are C# PascalCase properties, FKs are `TypeId`/`TypeId` references, and stored/read-model/transient entities are distinguished. SQL Server is the target database.
>
> Property types are best-effort from input types (text/number/date/select/file). Keys: `PK` primary key, `FK` reference, `enum` dropdown, `m2m` many-to-many, `file` upload, `percent`, `bool`, `auto`. All entities inherit `BaseEntity` (`Id`, `CreatedAt`, `UpdatedAt`, `IsActive`, `IsDeleted`) unless stated; only extra properties are listed where `Id`/audit fields are implied.

**Scope: ~190 entities across 27 domains** (25 admin sidebar groups + public/tenant + auth). See [Complete Model Listing](#appendix-complete-model-listing) for the full class index.

---

## Table of Contents

1. [Tenant & Core Platform](#1-tenant--core-platform)
2. [Students Information](#2-students-information)
3. [Staffs Information](#3-staffs-information)
4. [Student Attendance](#4-student-attendance)
5. [QR Code Attendance](#5-qr-code-attendance)
6. [Academic Configuration](#6-academic-configuration)
7. [Fees Management](#7-fees-management)
8. [Accounts Management](#8-accounts-management)
9. [Accounting Reports](#9-accounting-reports)
10. [Payroll Management](#10-payroll-management)
11. [Routine Management](#11-routine-management)
12. [Library Management](#12-library-management)
13. [Exam Module](#13-exam-module)
14. [Layout & Certificates](#14-layout--certificates)
15. [SMS Module](#15-sms-module)
16. [Administrator](#16-administrator)
17. [System](#17-system)
18. [Master Configuration](#18-master-configuration)
19. [CMS Management](#19-cms-management)
20. [WhatsApp](#20-whatsapp)
21. [Hostel Management](#21-hostel-management)
22. [Inventory](#22-inventory)
23. [Transport Management](#23-transport-management)
24. [Google Meet](#24-google-meet)
25. [AI Assistant](#25-ai-assistant)
26. [Auth, Profile & Public Forms](#26-auth-profile--public-forms)
27. [Cross-cutting / shared entities](#27-cross-cutting--shared-entities)
- [Appendix: Complete Model Listing](#appendix-complete-model-listing)

---

## 1. Tenant & Core Platform

### Institute
- **Model:** `MightySchool.Entities/Entities/Institute.cs`
- **Screen:** `/institutes`
- **Purpose:** Top-level tenant (school/college/training center) in the SaaS.
- **Properties:**
  - `int Id` (PK)
  - `string InstituteName`
  - `InstituteType Type` (enum: school/college/training-center…)
  - `int? OwnerId` (FK → User)
  - `string? Phone`
  - `string? Email`
  - `string? Domain` — default subdomain or custom
  - `int? PackageId` (FK → Package)
  - `SubscriptionStatus SubscriptionStatus` (enum)
  - `TenantStatus Status` (enum: pending/active/suspended)
  - `DateTime? StartDate`
  - `decimal AmountPaid` (money)
  - `PaymentMethod PaymentMethod` (enum)
  - `DateTime CreatedAt` (auto)

### Package
- **Model:** `MightySchool.Entities/Entities/Package.cs`
- **Screen:** `/institutes` (selector)
- **Purpose:** Subscription pack per tenant.
- **Properties:** `int Id`, `string Name`, `decimal Price`, `int Duration`, `string FeatureLimits` (json), `bool? Status`

### Branch
- **Model:** `MightySchool.Entities/Entities/Branch.cs`
- **Screen:** `/branches`
- **Purpose:** Branch under an institute.
- **Properties:** `int Id`, `int InstituteId` (FK → Institute), `string Name`, `bool? Status`, `DateTime CreatedAt`

### CustomDomain
- **Model:** `MightySchool.Entities/Entities/CustomDomain.cs`
- **Screen:** `/custom-domain`
- **Purpose:** Tenant custom-domain requests.
- **Properties:** `int Id`, `int InstituteId` (FK → Institute), `string Domain`, `string? Note`, `DomainStatus Status` (enum: pending/verified), `DateTime? SubmittedAt`

### TenantGeneralSetting
- **Model:** `MightySchool.Entities/Entities/TenantGeneralSetting.cs`
- **Screen:** `/administration/general_settings`
- **Purpose:** Single per-tenant settings record (saved via `POST /administration/general_settings/update`).
- **Properties (by tab):**
  - General: `string InstituteName`, `string SiteTitle`, `string? Tagline`, `string? Phone`, `string? Email`, `string? Address`, `string? Ein`, `int InstituteId` (FK), `string? Copyright`
  - SEO: `string? MetaTitle`, `string? MetaDescription`, `string? MetaKeywords`, `string? OgImageUrl`, `string? Ga4MeasurementId`, `string? GtmContainerId`, `string? MetaPixelId`
  - Fee/exam: `decimal TuitionFee`, `bool ExamResultVisible`, `int? AttendanceOutAfter` (device), `decimal TransferCertificateFee` (money), `bool LiveNoticeOnHeader`
  - Admission/app: `bool OnlineAdmissionStatus`, `bool ExamResultsDisplayStatus`, `string? AppVersion`, `string? AppUrl`, `string? PlayStoreLink`, `string? AppStoreLink`
  - Mail: `string? MailType`, `string? FromEmail`, `string? FromName`, `string? SmtpHost`, `int? SmtpPort`, `string? SmtpUsername`, `string? SmtpPassword` (secret), `string? SmtpEncryption`
  - SMS: `string? SmsGateway`, `bool SmsTestMode`, `string? ApiKey` (secret), `string? SenderId`, `string? UserName`, `string? SenderName`, `string? TwilioSid`, `string? TwilioToken` (secret), `string? FromNumber`, `string? BaseUrl`, `string? Originator`
  - Social/map: `string? GoogleMap`, `bool OnGoogleMap`, `string? FacebookLink`, `string? GooglePlusLink`, `string? YoutubeLink`, `string? WhatsappLink`, `string? TwitterLink`, `string? LinkedinLink`
  - Zoom: `string? ZoomAccountId`, `string? ZoomClientKey`, `string? ZoomClientSecret` (secret)
  - Google Meet: `bool GoogleMeetEnabled`, `string? ClientId`, `string? ClientSecret` (secret), `string? ProjectId`, `string? RedirectUri`, `string? ServiceAccountJson` (file/json, secret), `string? CalendarId`, `int? DefaultDuration`, `string? DefaultTimezone`, `string? DefaultVisibility`
  - Theme: `string? PrimaryColor`, `string? SecondaryColor`, `string? TextColor`, `string? SidebarColor`

### InstituteImageSetting
- **Model:** `MightySchool.Entities/Entities/InstituteImageSetting.cs`
- **Screen:** `/institute-image-settings`
- **Purpose:** Branding images per tenant (used in headers/footers/favicon).
- **Properties:** `int Id`, `string? HeaderLight` (file), `string? HeaderDark` (file), `string? FooterLight` (file), `string? FooterDark` (file), `string? Favicon` (file), `bool? Status`

### PaymentGateway
- **Model:** `MightySchool.Entities/Entities/PaymentGateway.cs`
- **Screen:** `/payment-gateways`
- **Purpose:** Payment processor registry.
- **Properties:** `int Id`, `string Gateway`, `GatewayMode Mode` (enum: test/live), `string Credentials` (json), `bool? Status`

---

## 2. Students Information

### Student
- **Model:** `MightySchool.Entities/Entities/Student.cs`
- **Screen:** `/students`
- **Purpose:** Core student record.
- **Properties:**
  - `int Id` (PK)
  - `string? Image` (file)
  - `int Roll`
  - `string AdmissionNumber`
  - `string Name`
  - `int? ClassId` (FK → Class)
  - `int? SectionId` (FK → Section)
  - `int? GroupId` (FK → StudentGroup)
  - `string? PhoneNo` / `string? GMobile`
  - `Gender Gender` (enum: male/female)
  - `bool? Status` (active/inactive)
  - `DateTime CreatedAt`

### StudentMigration
- **Model:** `MightySchool.Entities/Entities/StudentMigration.cs`
- **Screens:** `/student-migration`, `/student-migration-pushback`, `/migrated-list`
- **Purpose:** Promote/lift students across sessions or revert.
- **Properties:** `int Id`, `int StudentId` (FK), `int? FromClassId`, `int? FromSectionId`, `int? ToClassId`, `int? ToSectionId`, `MigrationType MigrationType` (enum), `int? AcademicYearId` (FK → AcademicYear), `int? GroupId` (FK → StudentGroup), `bool? Status`, `DateTime CreatedAt`

### StudentAtAGlance (read model)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/at-a-glance`
- **Purpose:** Flat read-model of all students.
- **Properties:** `int StudentId`, `int RollNo`, `string AdmissionNumber`, `string Name`, `string Class`, `string Section`, `string Gender`, `string? GMobile` (seeded 100 rows)

---

## 3. Staffs Information

### Staff
- **Model:** `MightySchool.Entities/Entities/Staff.cs`
- **Screen:** `/staffs`
- **Purpose:** Non-teaching staff.
- **Properties:** `int Id`, `string? ProfileImage` (file), `string Name`, `string? Phone`, `string? Email`, `string? Designation`, `int? DepartmentId` (FK), `BloodGroup BloodGroup` (enum), `bool? Status`, `string? HrId` (used in payroll screens)

### Teacher
- **Model:** `MightySchool.Entities/Entities/Teacher.cs`
- **Screens:** `/teachers`
- **Purpose:** Teaching staff (subtype of staff with academic role).
- **Properties:** `int Id`, `string? ProfileImage`, `string Name`, `string? Phone`, `string? Email`, `int? DepartmentId` (FK), `string Designation` (enum: Professor/Lecturer…), `BloodGroup? BloodGroup`, `bool? Status`

### StaffAttendance
- **Model:** `MightySchool.Entities/Entities/StaffAttendance.cs`
- **Screens:** `/staffs-attendance`
- **Purpose:** Daily staff attendance.
- **Properties:** `int Id`, `string Role` (enum), `int StaffId` (FK), `DateTime Date`, `AttendanceStatus Status` (enum: present/absent/late/leave), `TimeSpan? CheckIn`, `TimeSpan? CheckOut`

---

## 4. Student Attendance

### StudentAttendanceRecord
- **Model:** `MightySchool.Entities/Entities/StudentAttendanceRecord.cs`
- **Screens:** `/student-attendance`
- **Purpose:** Daily marking per class/section.
- **Properties:** `int Id`, `int? ClassId` (FK), `int? SectionId` (FK), `int StudentId` (FK), `DateTime Date`, `StudentAttendanceStatus Status` (enum: P/A/L/E), `int MarkedBy` (FK → User), `DateTime CreatedAt`

### ExamAttendance
- **Model:** `MightySchool.Entities/Entities/ExamAttendance.cs`
- **Screen:** `/exams-attendance`
- **Purpose:** Attendance inside exams.
- **Properties:** `int Id`, `int? ExamId` (FK), `int? ClassId` (FK), `int? SectionId` (FK), `int? SubjectId` (FK), `int StudentId` (FK), `bool? Status`

### ExamSchedule
- **Model:** `MightySchool.Entities/Entities/ExamSchedule.cs`
- **Screen:** `/exams-schedule`
- **Purpose:** Timetable of exam sessions.
- **Properties:** `int Id`, `int ExamId` (FK), `int ClassId` (FK), `int? GroupId` (FK), `DateTime ExamStartAt`, `int SubjectId` (FK), `int? Duration` (minutes)

### AttendanceReportSummary (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/reports-student_attendance_date_to_date` (→ `/view`)
- **Purpose:** Aggregated attendance (filter by class/section/range).
- **Properties:** `string Student`, `int PresentDays`, `int AbsentDays`, `decimal Percentage`

### AbsentFineReport (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/absent-fine-report`
- **Purpose:** Fines charged for absences.
- **Properties:** `string Student`, `int DaysAbsent`, `decimal FineAmount`

---

## 5. QR Code Attendance

### QRAttendanceScannerSession
- **Model:** `MightySchool.Entities/Entities/QRAttendanceScannerSession.cs`
- **Screen:** `/qrattendance/scanner`
- **Purpose:** Camera/QR based punch-in.
- **Properties:** `int Id`, `int StudentId` (FK), `DateTime Date`, `TimeSpan? Time`, `string? Device`, `bool? Status`

---

## 6. Academic Configuration

### AcademicYear
- **Model:** `MightySchool.Entities/Entities/AcademicYear.cs`
- **Screen:** `/academic-years`
- **Properties:** `int Id`, `string SessionName`, `string AcademicYearLabel`, `bool IsCurrent`

### Shift
- **Model:** `MightySchool.Entities/Entities/Shift.cs`
- **Screen:** `/shift`
- **Properties:** `int Id`, `string Name` (e.g. Morning/Day)

### Class
- **Model:** `MightySchool.Entities/Entities/Class.cs`
- **Screen:** `/class`
- **Properties:** `int Id`, `string Name`, `int? NumericValue`, `bool? Status`

### Section
- **Model:** `MightySchool.Entities/Entities/Section.cs`
- **Screen:** `/sections`
- **Properties:** `int Id`, `ICollection<int> ClassIds` (FK, m2m classes), `string SectionName`, `int? GroupId` (FK), `string? RoomNo`

### StudentGroup
- **Model:** `MightySchool.Entities/Entities/StudentGroup.cs`
- **Screen:** `/student-groups`
- **Properties:** `int Id`, `string GroupName` (e.g. Science/Commerce/Arts)

### Period
- **Model:** `MightySchool.Entities/Entities/Period.cs`
- **Screen:** `/periods`
- **Properties:** `int Id`, `string PeriodName`, `TimeSpan? StartTime`, `TimeSpan? EndTime`, `int? Serial`, `int? ClassId` (FK)

### Subject
- **Model:** `MightySchool.Entities/Entities/Subject.cs`
- **Screen:** `/subjects`
- **Properties:** `int Id`, `string SubjectName`, `string SubjectCode` (unique, e.g. BAN101), `int? ClassId` (FK), `SubjectType SubjectType` (enum: compulsory/optional), `bool? Status`

### SubjectConfig
- **Model:** `MightySchool.Entities/Entities/SubjectConfig.cs`
- **Screen:** `/subject-config/create`
- **Purpose:** Which subjects a class/group studies; merge groups of subjects.
- **Properties:** `int Id`, `int ClassId` (FK), `int? GroupId` (FK), `ICollection<int> SubjectIds` (m2m → Subject), `int? SubjectSerial`, `int? MergeId`, `SubjectType SubjectType`

### OptionalSubjectConfig
- **Model:** `MightySchool.Entities/Entities/OptionalSubjectConfig.cs`
- **Screen:** `/optional-subject-config`
- **Properties:** `int Id`, `string Name`, `int ClassId` (FK), `int? GroupId` (FK), `ICollection<int> SubjectIds` (m2m), `int? MaxLimit`

### StudentOptionalSubject
- **Model:** `MightySchool.Entities/Entities/StudentOptionalSubject.cs`
- **Screen:** `/student-optional-subject`
- **Properties:** `int Id`, `int StudentId` (FK), `int SubjectId` (FK), `int ConfigId` (FK → OptionalSubjectConfig)

### Exam
- **Model:** `MightySchool.Entities/Entities/Exam.cs`
- **Screen:** `/exams`
- **Properties:** `int Id`, `string Name` (e.g. Half Yearly/Final), `string? ExamCode`

### StudentCategory
- **Model:** `MightySchool.Entities/Entities/StudentCategory.cs`
- **Screen:** `/student-categories`
- **Properties:** `int Id`, `string StudentCategoryName` (e.g. Regular/Talentpool)

### Department
- **Model:** `MightySchool.Entities/Entities/Department.cs`
- **Screen:** `/departments`
- **Properties:** `int Id`, `string DepartmentName`, `int? Priority`

### Picklist
- **Model:** `MightySchool.Entities/Entities/Picklist.cs`
- **Screen:** `/picklists`
- **Purpose:** Reusable dropdown option sets.
- **Properties:** `int Id`, `string Type`, `string Value`, `string? Slug`

### Signature
- **Model:** `MightySchool.Entities/Entities/Signature.cs`
- **Screen:** `/signatures`
- **Purpose:** Authority signatures for documents.
- **Properties:** `int Id`, `SignaturePlaceAt PlaceAt` (enum: principal/class-teacher/parent…), `string Title`, `string? SignatureImage` (file)

---

## 7. Fees Management

### FeeHead
- **Model:** `MightySchool.Entities/Entities/FeeHead.cs`
- **Screen:** `/fee-head`
- **Purpose:** Fee heads and sub-heads.
- **Properties:** `int Id`, `int? FeeHeadId` (FK, self-parent), `string Name`, `int? Serial`, `string Type`

### FeeMapping
- **Model:** `MightySchool.Entities/Entities/FeeMapping.cs`
- **Screen:** `/fees-mapping`
- **Purpose:** Link fee heads to accounting ledgers/funds.
- **Properties:** `int Id`, `int FeeHeadId` (FK), `ICollection<int> FeeSubHeadIds` (m2m), `int? LedgerId` (FK → AccountingLedger), `int? FundId` (FK → AccountingFund)

### FeeAmountConfig
- **Model:** `MightySchool.Entities/Entities/FeeAmountConfig.cs`
- **Screen:** `/amount-config`
- **Properties:** `int Id`, `int? ClassId` (FK), `int? GroupId` (FK), `int? SectionId` (FK), `int? StudentCategoryId` (FK), `int? FeeHeadId` (FK), `decimal FeeAmount` (money), `decimal FineAmount` (money), `int? FundId` (FK), `string? Period`, `decimal Amount` (money)

### FeeDateConfig
- **Model:** `MightySchool.Entities/Entities/FeeDateConfig.cs`
- **Screen:** `/date-config`
- **Properties:** `int Id`, `int? AcademicYearId` (FK), `int? FeeHeadId` (FK), `string? FeeSubHead`, `DateTime? FeePayableDate`, `DateTime? FineActiveDate`

### AttendanceFineWaiver
- **Model:** `MightySchool.Entities/Entities/AttendanceFineWaiver.cs`
- **Screen:** `/attendance-waiver`
- **Properties:** `int Id`, `int? ClassId` (FK), `int? SectionId` (FK), `int? StudentId` (FK), `int DaysWaived`

### Waiver
- **Model:** `MightySchool.Entities/Entities/Waiver.cs`
- **Screen:** `/waivers`
- **Properties:** `int Id`, `string WaiverName`

### WaiverConfig
- **Model:** `MightySchool.Entities/Entities/WaiverConfig.cs`
- **Screen:** `/waiver-config`
- **Purpose:** Per-student fee waivers.
- **Properties:** `int Id`, `int? AcademicYearId` (FK), `int? GroupId` (FK), `int? ClassId` (FK), `int? SectionId` (FK), `int? StudentCategoryId` (FK), `int? StudentId` (FK), `int? Roll`, `int? FeeHeadId` (FK), `int WaiverId` (FK → Waiver), `decimal Amount` (money)

### FeeCollection
- **Model:** `MightySchool.Entities/Entities/FeeCollection.cs`
- **Screen:** `/quick-collection`
- **Purpose:** Capture student fee payments; also exposes per-student paid status.
- **Properties:** `int Id`, `int StudentId` (FK), `string? InvoiceId`, `int? AcademicYearId` (FK), `int? FeeHeadId` (FK), `decimal Amount` (money), `decimal Fine` (money), `decimal DiscountOrWaiver` (money), `PaidBy PaidBy` (enum), `int? PaymentMethodId` (FK → PaymentGateway), `int? FundId` (FK → AccountingFund), `DateTime? Date`, `PaymentStatus Status`

### PaymentFeeInfo (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/payment-fee-info`
- **Purpose:** Read-model of collected payments by filter.
- **Properties:** `string Student`, `string Class`, `string Section`, `decimal Amount`, `decimal Discount`, `decimal Fine`, `PaymentStatus PaidStatus`, `DateTime? Date`

### UnpaidFeeInfo (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/unpaid-info`
- **Purpose:** Due ledger per student.
- **Properties:** `int Sl`, `string Student`, `int Roll`, `string DueDetails`, `decimal TotalDue`

---

## 8. Accounts Management

### AccountingCategory
- **Model:** `MightySchool.Entities/Entities/AccountingCategory.cs`
- **Screen:** `/accounting-categories`
- **Properties:** `int Id`, `string Name`, `string Code`, `AccountingCategoryType Type` (enum)

### AccountingGroup
- **Model:** `MightySchool.Entities/Entities/AccountingGroup.cs`
- **Screen:** `/accounting-groups`
- **Properties:** `int Id`, `int AccountCategoryId` (FK), `string Name`

### AccountingLedger
- **Model:** `MightySchool.Entities/Entities/AccountingLedger.cs`
- **Screen:** `/accounting-ledgers`
- **Properties:** `int Id`, `string LedgerName`, `int? AccountGroupId` (FK), `int? AccountCategoryId` (FK), `LedgerNature Nature` (enum: debit/credit)

### AccountingFund
- **Model:** `MightySchool.Entities/Entities/AccountingFund.cs`
- **Screen:** `/accounting-funds`
- **Properties:** `int Id`, `string Name`, `int? Serial`, `decimal OpeningBalance`, `decimal AmountIn` (derived), `decimal AmountOut` (derived), `decimal Balance` (derived)

### CashTransaction
- **Model:** `MightySchool.Entities/Entities/CashTransaction.cs`
- **Screens:** `/account-transaction-payment` (+ `?type=receipt`)
- **Purpose:** Payment (expense) & receipt (income) vouchers; receipt shares table with `type` discriminator.
- **Properties:** `int Id`, `CashTransactionType Type` (enum: payment/receipt), `DateTime TransactionDate`, `string? PaidBy`, `string? ReceiptType`, `int? FundId` (FK), `string? Reference`, `string? Description`, `ICollection<CashTransactionLine> Lines` (ledger ids + amounts, m2m)

### ContraTransfer
- **Model:** `MightySchool.Entities/Entities/ContraTransfer.cs`
- **Screen:** `/account-contra-transfers`
- **Purpose:** Movement between internal accounts.
- **Properties:** `int Id`, `DateTime TransferDate`, `int TransferFromId` (FK → AccountingLedger/AccountingFund), `int TransferToId` (FK), `decimal Amount`, `string? Reference`, `string? Description`

### JournalTransaction
- **Model:** `MightySchool.Entities/Entities/JournalTransaction.cs`
- **Screen:** `/journal-transactions`
- **Properties:** `int Id`, `DateTime JournalDate`, `int? FundId` (FK), `string? Description`, `string? Reference`, `ICollection<JournalLine> Lines` (debit/credit)

### FundTransfer
- **Model:** `MightySchool.Entities/Entities/FundTransfer.cs`
- **Screen:** `/account-fund-transfers`
- **Properties:** `int Id`, `DateTime PaymentDate`, `int TransferFromId` (FK → AccountingFund), `int TransferToId` (FK), `decimal Amount`, `string? Description`

### ChartOfAccounts (read model)
- **Model:** read-model tree (SP-backed, not stored)
- **Screen:** `/chart-of-accounts`
- **Purpose:** Read-model tree; seeded parents:
  - 1. Cash & Cash Equivalence → Assets
  - 2. Current Liabilities → Liabilities
  - 3. Non-Current Liabilities → Liabilities
  - 4. Owner's Equity → Liabilities
  - 5. Fees Related Income → Income
  - 6. Others Income → Income
  - 7. General Expenses → Expense

---

## 9. Accounting Reports

> All are **read-only view projections** (SP-backed) — no stored attributes beyond filters.

| Entity (report) | Screen | Filters |
|---|---|---|
| BalanceSheet | `/balance-sheet` | from, to |
| TrialBalance | `/trial-balance` | from, to |
| CashFlowStatement | `/cash-flow-statement` | year |
| CashFlowDetail | `/cash-flow-details` | year |
| CashBook | `/cash-book-account` | from_date, to_date, payment_method |
| LedgerBook | `/ledger-book-account` | from_date, to_date, payment_method |
| IncomeStatement | `/income-statement` | ledger_income_list, income_list, ledger_expense_list, expense_list, profit/loss |
| IncomeStatementDetail | `/income-statement-details` | from, to |
| CashSummary | `/cash-summary` | ledger_income_list, income_list, ledger_expense_list, expense_list |

---

## 10. Payroll Management

### SalaryHead
- **Model:** `MightySchool.Entities/Entities/SalaryHead.cs`
- **Screen:** `/salary-heads`
- **Properties:** `int Id`, `string SalaryHeadName` (Basic, Allowance, Welfare Fund, Professional Tax…), `SalaryNature Nature` (enum: plus/minus)

### PayrollMapping
- **Model:** `MightySchool.Entities/Entities/PayrollMapping.cs`
- **Screen:** `/payroll-mapping`
- **Purpose:** Map payroll account to ledger/fund.
- **Properties:** `int Id`, `int? LedgerId` (FK → AccountingLedger), `int? FundId` (FK → AccountingFund)

### PayrollAssign
- **Model:** `MightySchool.Entities/Entities/PayrollAssign.cs`
- **Screen:** `/payroll-assign`
- **Purpose:** Salary structure per staff.
- **Properties:** `int Id`, `int StaffId` (FK), `DateTime EffectiveDate`, `decimal NetSalary` (money), `decimal Md` (money, +), `decimal Basic` (money, +), `decimal Allowance` (money, +), `decimal EarlyLeaveFine` (money, −), `decimal FestivalAllowance` (money, +), `decimal WelfareFund` (money, −), `decimal ProfessionalTax` (money, −), `decimal Conveyance` (money, +), `decimal ExamHallDuty` (money, +), `decimal Incentive` (money, +), `decimal Medical` (money, +)

### SalarySlip
- **Model:** 'generated' document (PDF), not stored per se
- **Screen:** `/salary-create`
- **Purpose:** Generated monthly pay slips per staff/period.

### SalaryPayment
- **Model:** `MightySchool.Entities/Entities/SalaryPayment.cs`
- **Screens:** `/salary-payment-process`, `/payment-info`
- **Properties:** `int Id`, `int StaffId` (FK), `string? HrId`, `string? InvoiceId`, `string MonthPeriod`, `decimal NetSalary`, `decimal PayableSalary`, `decimal Paid`, `decimal Due`, `decimal Advance`, `string? PaymentType`, `PaymentStatus PaidStatus`, `DateTime? PaymentDate`

### DueSalaryPayment
- **Model:** `MightySchool.Entities/Entities/DueSalaryPayment.cs`
- **Screen:** `/due-salary-payment`
- **Properties:** `int Id`, `string? HrId`, `string Month`, `decimal DueAmount`, `decimal Paid`, `bool? Status`

### AdvanceSalaryPayment
- **Model:** `MightySchool.Entities/Entities/AdvanceSalaryPayment.cs`
- **Screen:** `/advance-salary-payment`
- **Properties:** `int Id`, `int StaffId` (FK), `string? HrId`, `decimal Amount`, `DateTime? Date`, `int? Installments`, `decimal Remaining`

### ReturnAdvancePayment
- **Model:** `MightySchool.Entities/Entities/ReturnAdvancePayment.cs`
- **Screen:** `/return-salary-payment`
- **Purpose:** Track advances recovered from salary.
- **Properties:** `int Id`, `int AdvanceId` (FK → AdvanceSalaryPayment), `decimal Amount`, `DateTime? Date`

### SalaryStatement (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/salary-statement`
- **Purpose:** Monthly aggregate report. View: staff, month, earnings, deductions, net, paid, due.

---

## 11. Routine Management

### Syllabus
- **Model:** `MightySchool.Entities/Entities/Syllabus.cs`
- **Screen:** `/syllabus`
- **Properties:** `int Id`, `string Title`, `string? Description`, `int? ClassId` (FK), `int? SubjectId` (FK), `string? File` (file), `bool? Status`

### Assignment
- **Model:** `MightySchool.Entities/Entities/Assignment.cs`
- **Screen:** `/assignments`
- **Properties:** `int Id`, `string Title`, `string? Description`, `int? ClassId` (FK), `int? SectionId` (FK), `int? SubjectId` (FK), `string? File` (file), `DateTime? DueDate`

### ClassRoutine
- **Model:** `MightySchool.Entities/Entities/ClassRoutine.cs`
- **Screen:** `/class_routines`
- **Purpose:** Weekly timetable grid.
- **Properties:** `int Id`, `int ClassId` (FK), `int SectionId` (FK), `DayOfWeek Day` (enum), `int PeriodId` (FK), `int SubjectId` (FK), `int TeacherId` (FK), `string? Room`

### ExamRoutine
- **Model:** `MightySchool.Entities/Entities/ExamRoutine.cs`
- **Screen:** `/exam-routines`
- **Properties:** `int Id`, `int ExamId` (FK), `int ClassId` (FK), `int? GroupId` (FK), `int SubjectId` (FK), `DateTime? Date`, `TimeSpan? StartTime`, `TimeSpan? EndTime`, `string? Room` (seat plan areas)

### AdmitAndSeatPlan
- **Model:** generated artifact (admit cards / seating plans), not stored
- **Screen:** `/exam-essentials`
- **Purpose:** Generated admit cards / seating plans (class, section, roll range).

---

## 12. Library Management

### BookCategory
- **Model:** `MightySchool.Entities/Entities/BookCategory.cs`
- **Screen:** `/book-categories`
- **Properties:** `int Id`, `string CategoryName`

### Book
- **Model:** `MightySchool.Entities/Entities/Book.cs`
- **Screen:** `/books`
- **Properties:** `int Id`, `string BookName`, `string? Code` / `string? Barcode`, `string? Author`, `int CategoryId` (FK), `int Quantity`, `int Available`, `string? RackShelf`, `bool? Status`

### LibraryMember
- **Model:** `MightySchool.Entities/Entities/LibraryMember.cs`
- **Screen:** `/librarymembers`
- **Properties:** `int Id`, `string? LibraryMemberId`, `string Name`, `string? Image`, `LibraryMemberType MemberType` (enum: student/teacher/staff), `int? UserId` (FK)

### BookIssue
- **Model:** `MightySchool.Entities/Entities/BookIssue.cs`
- **Screens:** `/books-ber-code-page` (issue UI), `/bookissues` (report)
- **Properties:** `int Id`, `int BookId` (FK), `int MemberId` (FK → LibraryMember), `string? LibraryId`, `DateTime IssueDate`, `DateTime? ReturnDate`, `bool Returned`, `bool? Status`

### BarcodePrintJob
- **Model:** transient print artifact (not stored)
- **Screen:** `/books-ber-code-print`
- **Properties:** `ICollection<string> BookCodes` → printed labels (transient)

---

## 13. Exam Module

### ExamStartup
- **Model:** `MightySchool.Entities/Entities/ExamStartup.cs`
- **Screen:** `/semester-exam-settings-exam-startup`
- **Purpose:** Compose exams for a class from global code/grade lists.
- **Properties:** `int Id`, `int ClassId` (FK), `int? GlobalExamCodeListId` (FK → ExamCodeList), `int? GlobalExamGradeListId` (FK → GradeList), `int? ExamId` (FK), `MeritProcessType MeritProcessType` (enum)

### ExamCodeList
- **Model:** `MightySchool.Entities/Entities/ExamCodeList.cs`
- **Screen:** `/semester-exam-settings-exam-startup`
- **Properties:** `int Id`, `string CodeTitle`, `int TotalMarks`, `int PassMark`, `string? Acceptance`

### GradeList
- **Model:** `MightySchool.Entities/Entities/GradeList.cs`
- **Screen:** `/semester-exam-settings-exam-startup`
- **Properties:** `int Id`, `string Grade`, `string GradeRange` (from–to), `decimal Point`

### ExamMarkConfig
- **Model:** `MightySchool.Entities/Entities/ExamMarkConfig.cs`
- **Screen:** `/semester-exam-settings-mark-config`
- **Purpose:** Weight of each exam in grand-final; per class.
- **Properties:** `int Id`, `int ClassId` (FK), `int? GroupId` (FK), `CalculationMethod CalculationMethod` (enum), `int ExamId` (FK), `string? ExamName`, `decimal Percentage` (percent), `int? ExamSerial`

### ExamRemark
- **Model:** `MightySchool.Entities/Entities/ExamRemark.cs`
- **Screen:** `/remarks-config`
- **Properties:** `int Id`, `string RemarkTitle`, `string? RemarkText`

### ExamMark
- **Model:** `MightySchool.Entities/Entities/ExamMark.cs`
- **Screen:** `/mark-input-section-wise`
- **Purpose:** Subject-wise mark entry per class/section.
- **Properties:** `int Id`, `int ExamId` (FK), `int ClassId` (FK), `int SectionId` (FK), `int SubjectId` (FK), `int StudentId` (FK), `decimal? Written`, `decimal? Mcq`, `decimal? Practical`, `decimal Total`, `string? Grade`, `decimal? Point`

### ExamResult
- **Model:** computed result (SP-backed projection) + notification dispatch
- **Screen:** `/exam-result-view`
- **Purpose:** Computed results (view) + notification dispatch (send scope/classes/notify-via).

### GrandFinalResult
- **Model:** computed result (SP-backed projection)
- **Screen:** `/grand-final-result`
- **Purpose:** Blended multi-exam result per student (class, section, roll range).

### TabulationSheet (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/tabulation-sheet`
- **Properties (view):** class, section, exam/term, sheet_type
- **Columns:** `string Student`, `int Roll`, per-subject marks, `decimal Total`, `decimal Gpa`, `string Grade`, `int Position`

### MeritListSheet (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/merit-list-sheet`
- **Purpose:** Ranked merit list per class/section/exam.

### ResultCardSetting
- **Model:** `MightySchool.Entities/Entities/ResultCardSetting.cs`
- **Screen:** `/result-card-settings`
- **Properties:**
  - `string ResultTitle`
  - toggles: `bool ShowPhoto`, `bool ShowPosition`, `bool ShowGpa`, `bool ShowPercentage`, `bool ShowFailedSubjects`, `bool ShowAttendance`, `bool ShowRemarks`, `bool ShowCognitiveDomain`, `bool ShowAffectiveDomain`, `bool ShowPsychomotorDomain`, `bool ShowTeacherSignature`, `bool ShowPrincipalSignature`, `bool ShowParentSignature`
  - labels: `string? ClassTeacherLabel`, `string? PrincipalLabel`, `string? ParentLabel`
  - signatures: `string? ClassTeacherSignature` (file), `string? PrincipalSignature` (file)
  - `FailedStudentRanking FailedStudentRanking` (enum)
  - `string? PrimaryColor`, `string? AccentColor`, `decimal WatermarkOpacity` (0–1), `bool ShowInstituteBannerWatermark`

### EmptyMarkSheet
- **Model:** printable artifact (not stored)
- **Screen:** `/empty-mark-sheet`
- **Purpose:** Printable blank mark sheets (class, section, exam, optional count).

### AssessmentDomain
- **Model:** `MightySchool.Entities/Entities/AssessmentDomain.cs`
- **Screen:** `/assessment-domains`
- **Purpose:** Co-curricular skill domains.
- **Properties:** `int Id`, `AssessmentDomainType Domain` (enum: cognitive/affective/psychomotor), `string Name`, `int DisplayOrder`
- **Child (AssessmentDomainItem):** `int Id`, `int DomainId` (FK), `string Item`, `int Max`, `string Type`

### DomainAssessmentEntry
- **Model:** `MightySchool.Entities/Entities/DomainAssessmentEntry.cs`
- **Screen:** `/assessment-entry`
- **Properties:** `int Id`, `int ClassId` (FK), `int ExamId` (FK), `int SectionId` (FK), `int StudentId` (FK), `int DomainItemId` (FK → AssessmentDomainItem), `decimal? Score`, `string? Grade`

### OnlineExam
- **Model:** `MightySchool.Entities/Entities/OnlineExam.cs`
- **Screen:** `/online-exams`
- **Properties:** `int Id`, `string Name`, `int? ExamId` (FK), `int? ClassId` (FK), `int? SubjectId` (FK), `string? QuestionBank` (json), `string? Window` (timed), `DateTime? StartAt`, `DateTime? EndAt`, `bool? Status`

---

## 14. Layout & Certificates

### CertificateTemplate
- **Model:** `MightySchool.Entities/Entities/CertificateTemplate.cs`
- **Screen:** `/certificate-templates`
- **Properties:** `int Id`, `string Name`, `CertificateType CertificateType` (enum, see layouts), `CertificateOrientation Orientation` (enum: portrait/landscape), `string? Colors`, `string? Background` (file), `string? Fields` (json of placeholders), `bool? Status`

### CertificateLayout
- **Model:** `MightySchool.Entities/Entities/CertificateLayout.cs`
- **Screen:** `/layout-cert?type={slug}` (shared entity, discriminated by `type`)
- **Types:** `general-certificate`, `testimonial`, `attendance-certificate`, `hsc-recommendation`, `transfer-certificate`, `abroad-recommendation`, `character-certificate`, `study-certificate`, `bonafide-certificate`, `migration-certificate`
- **Properties (request):** class, section, student, certificate fields, layout preview → PDF print.

---

## 15. SMS Module

### SmsTemplate
- **Model:** `MightySchool.Entities/Entities/SmsTemplate.cs`
- **Screen:** `/sms-template`
- **Properties:** `int Id`, `string Title`, `string Message`

### PhoneBookCategory
- **Model:** `MightySchool.Entities/Entities/PhoneBookCategory.cs`
- **Screen:** `/phone-book-category`
- **Properties:** `int Id`, `string CategoryTitle`, `string? Description`

### PhoneBookContact
- **Model:** `MightySchool.Entities/Entities/PhoneBookContact.cs`
- **Screen:** `/phone-book`
- **Properties:** `int Id`, `string Name`, `string Phone`, `int? CategoryId` (FK → PhoneBookCategory), `int? ClassId` (FK), `int? SectionId` (FK), `string? Note`
- Sync: `POST /phone-book-sync`

### SmsSend
- **Model:** `MightySchool.Entities/Entities/SmsSend.cs`
- **Screen:** `/sms-compose`
- **Properties:** `int Id`, `int? ClassId` (FK), `int? SectionId` (FK), `string? Number` (or from phone-book), `int? TemplateId` (FK), `string Message` (≤300), `string? Gateway`, `DateTime? SentAt`, `bool? Status`
- **SmsSentLog:** `int Id`, `string To`, `string Message`, `string? GatewayResponse`, `bool? Status`, `DateTime? SentAt`

### SmsPurchase
- **Model:** `MightySchool.Entities/Entities/SmsPurchase.cs`
- **Screen:** `/sms-purchase`
- **Properties:** `int Id`, `string SmsGateway` (FK), `int Quantity`, `decimal Price`, `DateTime? TransactionDate`, `MaskingType MaskingType` (enum: masking/non-masking), `bool? Status`

### SmsReport (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/send-sms-report`
- **Purpose:** Sent-vs-delivered aggregate (from/to).

---

## 16. Administrator

### ShiftAssignment
- **Model:** `MightySchool.Entities/Entities/ShiftAssignment.cs`
- **Screen:** `/assign-shifts`
- **Properties:** `int Id`, `int TeacherId` (FK), `int ShiftId` (FK)

### SubjectAssignment
- **Model:** `MightySchool.Entities/Entities/SubjectAssignment.cs`
- **Screen:** `/assign-subjects`
- **Properties:** `int Id`, `int ClassId` (FK), `int SectionId` (FK), `int SubjectId` (FK), `int TeacherId` (FK)

### ClassAssignment
- **Model:** `MightySchool.Entities/Entities/ClassAssignment.cs`
- **Screen:** `/assign-class`
- **Properties:** `int Id`, `int TeacherId` (FK), `int ClassId` (FK)

### Notice
- **Model:** `MightySchool.Entities/Entities/Notice.cs`
- **Screen:** `/notices`
- **Properties:** `int Id`, `string? Image` (file), `string Title`, `AudienceType UserType` (enum audience), `string? NoticeText` (rich text), `bool? Status`, `DateTime CreatedAt`

### Event
- **Model:** `MightySchool.Entities/Entities/Event.cs`
- **Screen:** `/events`
- **Properties:** `int Id`, `string? Image`, `string Name`, `string? Location`, `DateTime? StartDate`, `DateTime? EndDate`, `string? Description`, `bool? Status` (public: `/event-details/{id}`)

### ContactMessage
- **Model:** `MightySchool.Entities/Entities/ContactMessage.cs`
- **Screen:** `/contact-message`
- **Purpose:** Submissions from the public contact form.
- **Properties:** `int Id`, `string Name`, `string? Phone`, `string? Email`, `string Message`, `DateTime? Date`, `ContactStatus Status` (enum: new/replied/closed)

### UserActivityLog
- **Model:** `MightySchool.Entities/Entities/UserActivityLog.cs`
- **Screen:** `/user-logs`
- **Properties:** `int Id`, `int UserId` (FK), `string? Name`, `string? IpAddress`, `string? Action`, `string? Detail`, `DateTime CreatedAt`, `DateTime? UpdatedAt`

### StudentIdCardBatch
- **Model:** print batch artifact (generated PDF)
- **Screen:** `/student-id-cards`
- **Purpose:** Bulk ID card generation.
- **Properties:** `int ClassId` (FK), `int? GroupId` (FK), `int? SectionId` (FK), `DateTime? CardValidityDate`, `string? Layout`, output PDF

### TeacherIdCardBatch
- **Model:** print batch artifact (generated PDF)
- **Screen:** `/teacher-id-cards`
- **Properties:** `int? DepartmentId` (FK), `DateTime? CardValidityDate`, output

### StaffIdCardBatch
- **Model:** print batch artifact (generated PDF)
- **Screen:** `/staff-id-cards`
- **Properties:** `int? DepartmentId` (FK), `DateTime? CardValidityDate`, output

---

## 17. System

### SystemInformation
- **Model:** env/server detail read-model (read-only)
- **Screen:** `/system/information`
- **Purpose:** Env/server detail panel (read-only).

### ModuleRegistry
- **Model:** `MightySchool.Entities/Entities/ModuleRegistry.cs`
- **Screen:** `/system/modules`
- **Purpose:** Installable/enableable feature modules.
- **Properties:** `int Id`, `string Module`, `string Version`, `bool? Status`, `string? Dependencies`, `string? Actions`

### SystemUpdate
- **Model:** `MightySchool.Entities/Entities/SystemUpdate.cs`
- **Screen:** `/system/update`
- **Properties:** `int Id`, `string Version`, `string? Archive` (file), `DateTime? AppliedAt`, `bool? Status`

### UpdateHistory
- **Model:** `MightySchool.Entities/Entities/UpdateHistory.cs`
- **Screen:** `/system/update/history`
- **Properties:** `string Version`, `DateTime? Date`, `string Type`, `bool? Status`, `int? AdminId` (FK → User), `string? Duration`

### SystemSetting
- **Model:** `MightySchool.Entities/Entities/SystemSetting.cs`
- **Screen:** `/system/settings`
- **Properties:** `string? UpdateServerUrl`, `int? DemoResetIntervalHours`, `bool EnableRemoteUpdateCheck`, `bool EnableDemoAutoReset`

---

## 18. Master Configuration

### Role
- **Model:** `MightySchool.Entities/Entities/Role.cs`
- **Screen:** `/roles`
- **Properties:** `int Id`, `string RoleName`, `string Permissions` (json/m2m)

### User
- **Model:** `MightySchool.Entities/Entities/User.cs`
- **Screens:** `/users`, `/profile`
- **Properties:** `int Id`, `string? ProfileImage`, `string Name`, `string? Email`, `string? Phone`, `string PasswordHash` (hashed — never stored/logged plaintext), `UserType UserType` (enum: admin/accountant/librarian/teacher/student/staff), `int? RoleId` (FK), `int? InstituteId` (FK), `bool IsActive`, `DateTime? LastLogin`

### Tenant Admin (superadmin) extension
- Provided via seeded demo users: superadmin, accountant, librarian, teacher1, student

---

## 19. CMS Management

### AdmissionApplication
- **Model:** `MightySchool.Entities/Entities/AdmissionApplication.cs`
- **Screen:** `/admission-forms`
- **Purpose:** Online admission requests.
- **Properties:** `int Id`, `string StudentName`, `int? ClassId` (FK), `string? Contact`/`string? Phone`, `string? Guardian`, `string? Email`, `string? Address`, `string? Documents`, `AdmissionStatus Status` (enum: pending/approved/rejected), `DateTime? Date`

### CmsPage
- **Model:** `MightySchool.Entities/Entities/CmsPage.cs`
- **Screen:** `/pages` → public `/page/{slug}`
- **Properties:** `int Id`, `string Title`, `string Slug`, `string Content` (rich text), `string? SeoMeta` (json), `bool? Status`

### Banner
- **Model:** `MightySchool.Entities/Entities/Banner.cs`
- **Screen:** `/banners`
- **Properties:** `int Id`, `string? Title`, `string? Image`, `string? ButtonName`, `string? ButtonLink`, `string? Description`, `int? Serial`, `bool? Status`

### AboutUsItem
- **Model:** `MightySchool.Entities/Entities/AboutUsItem.cs`
- **Screen:** `/about-us`
- **Properties:** `int Id`, `string? Title`, `string? Image`, `string? ShortDescription`, `bool? Status`

### Faq
- **Model:** `MightySchool.Entities/Entities/Faq.cs`
- **Screen:** `/faqs`
- **Properties:** `int Id`, `string Question`, `string Answer`, `bool? Status`, `int? Serial`

### GalleryImage
- **Model:** `MightySchool.Entities/Entities/GalleryImage.cs`
- **Screen:** `/gallery-images` → public `/academic-images`
- **Properties:** `int Id`, `string? Title`, `string? Heading`, `string? Image`, `bool? Status`

### MobileAppSection
- **Model:** `MightySchool.Entities/Entities/MobileAppSection.cs`
- **Screen:** `/mobile-app-sections`
- **Properties:** `int Id`, `string? Title`, `string? Heading`, `string? Image`, `string? Features` (list), `string? StoreLinks` (Android/iOS), `bool? Status`

### WhyChooseUsItem
- **Model:** `MightySchool.Entities/Entities/WhyChooseUsItem.cs`
- **Screen:** `/why-choose-us`
- **Properties:** `int Id`, `string? Title`, `string? Icon`, `string? Description`, `int? Serial`

### Policy
- **Model:** `MightySchool.Entities/Entities/Policy.cs`
- **Screen:** `/policies` → public `/privacy-policy`, `/terms-conditions`, `/refund-policy`, `/cookies-policy`
- **Properties:** `int Id`, `PolicyType Type` (enum: privacy/terms/refund/cookies), `bool? Status`, `string Description` (rich text)

### ReadyToJoinUsItem
- **Model:** `MightySchool.Entities/Entities/ReadyToJoinUsItem.cs`
- **Screen:** `/ready-to-join-us`
- **Properties:** `int Id`, `string? Icon`, `string? Title`, `string? Description`, `string? Button`, `bool? Status`

### Testimonial
- **Model:** `MightySchool.Entities/Entities/Testimonial.cs`
- **Screen:** `/testimonials`
- **Properties:** `int Id`, `string? Image`, `string? Name`, `string? Designation`, `string? Review`, `int Rating` (1–5), `bool? Status`

### AchievementCounter
- **Model:** static content block (seeded under content settings)
- **Screens:** homepage section; stored under content settings.
- **Values seeded:** expert teachers 4+, total students 102+, school events 3+, happy reviews 4+.

---

## 20. WhatsApp

### WhatsAppSetting
- **Model:** `MightySchool.Entities/Entities/WhatsAppSetting.cs`
- **Screen:** `/whats-app-settings`
- **Properties:** `int Id`, `string? Provider`, `string? PhoneId`, `string? BusinessId`, `string? AccessKey` (secret), `string? Language`, `bool? Status`

### WhatsAppTemplate
- **Model:** `MightySchool.Entities/Entities/WhatsAppTemplate.cs`
- **Screen:** `/whats-app-templates`
- **Properties:** `int Id`, `string Name`, `WhatsAppEvent Event` (enum: exam/admission/fee…), `string? Message`, `bool? Status`

### WhatsAppLog
- **Model:** `MightySchool.Entities/Entities/WhatsAppLog.cs`
- **Screen:** `/whats-app-logs`
- **Properties:** `int Id`, `int? StudentId` (FK), `string? Phone`, `string? Message`, `int Retries`, `DateTime? SentAt`, `bool? Status`

---

## 21. Hostel Management

### Hostel
- **Model:** `MightySchool.Entities/Entities/Hostel.cs`
- **Screen:** `/hostel-categories`
- **Properties:** `int Id`, `string Name`, `HostelType HostelType` (enum), `string? Address`, `bool? Status`

### HostelCategory
- **Model:** `MightySchool.Entities/Entities/HostelCategory.cs`
- **Screen:** `/hostel-categories`
- **Properties:** `int Id`, `int HostelId` (FK), `string Standard` (class level), `decimal Fee` (money), `string? Note`

### HostelMember
- **Model:** `MightySchool.Entities/Entities/HostelMember.cs`
- **Screen:** `/hostel-members`
- **Properties:** `int Id`, `int StudentId` (FK), `int? ClassId` (FK), `int HostelId` (FK), `int CategoryId` (FK → HostelCategory), `decimal Fee` (money), `bool? Status`

### HostelBuilding
- **Model:** `MightySchool.Entities/Entities/HostelBuilding.cs`
- **Screen:** `/hostel-buildings`
- **Properties:** `int Id`, `int HostelId` (FK), `string BuildingName`

### HostelFloor
- **Model:** `MightySchool.Entities/Entities/HostelFloor.cs`
- **Screen:** `/hostel-floors`
- **Properties:** `int Id`, `int BuildingId` (FK), `string FloorName`

### HostelRoom
- **Model:** `MightySchool.Entities/Entities/HostelRoom.cs`
- **Screen:** `/rooms`
- **Properties:** `int Id`, `string RoomNumber`, `int HostelCategoryId` (FK), `int FloorId` (FK), `int Capacity`, `int Occupied` (derived)

### Bed
- **Model:** `MightySchool.Entities/Entities/Bed.cs`
- **Screen:** `/beds`
- **Properties:** `int Id`, `int RoomId` (FK), `string BedNumber`, `BedStatus Status` (enum: free/assigned/maintenance)

### RoomMember
- **Model:** `MightySchool.Entities/Entities/RoomMember.cs`
- **Screen:** `/room-members`
- **Properties:** `int Id`, `int StudentId` (FK), `string? Phone`, `int RoomId` (FK), `int? BedId` (FK), `int? HostelCategoryId` (FK), `DateTime? AssignedAt`

### Meal
- **Model:** `MightySchool.Entities/Entities/Meal.cs`
- **Screen:** `/meals`
- **Properties:** `int Id`, `string MealName`, `MealType Type` (enum: breakfast/lunch/dinner), `DateTime CreatedAt`

### MealPlan
- **Model:** `MightySchool.Entities/Entities/MealPlan.cs`
- **Screen:** `/meal-plans`
- **Properties:** `int Id`, `int StudentId` (FK), `int MealId` (FK), `DateTime? Date`

### MealEntry
- **Model:** `MightySchool.Entities/Entities/MealEntry.cs`
- **Screen:** `/meal-entries`
- **Properties:** `int Id`, `int StudentId` (FK), `int MealId` (FK), `DateTime? Date`, `decimal Price` (money)

### HostelBill
- **Model:** `MightySchool.Entities/Entities/HostelBill.cs`
- **Screen:** `/hostel-bills`
- **Properties:** `int Id`, `int StudentId` (FK), `decimal HostelFee` (money), `decimal MealFee` (money), `decimal TotalAmount` (derived), `DateTime? DueDate`, `string Period` (month), `bool? Status`

### HostelLeave
- **Model:** `MightySchool.Entities/Entities/HostelLeave.cs`
- **Screen:** `/hostel-leaves`
- **Properties:** `int Id`, `int StudentId` (FK), `int RoomId` (FK), `DateTime? From`, `DateTime? To`, `string? Reason`, `LeaveStatus Status` (enum: pending/approved/rejected)

### HostelCollection
- **Model:** `MightySchool.Entities/Entities/HostelCollection.cs`
- **Screen:** `/hostel-collections`
- **Properties:** `int Id`, `string? Invoice`, `int StudentId` (FK), `int? RoomId` (FK), `int? BedId` (FK), `string Month`, `decimal ThisMonth` (money), `decimal AllOutstandingDue` (money), `decimal Paid` (money), `decimal Due` (money), `DateTime? Date`

### HostelSeatMap (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/hostel-seat-map`
- **Purpose:** Visual bed allocation view (read-only aggregate).

---

## 22. Inventory

### InventoryCategory
- **Model:** `MightySchool.Entities/Entities/InventoryCategory.cs`
- **Screen:** `/inventory-categories`
- **Properties:** `int Id`, `string Name`, `string? Note`

### InventoryItem
- **Model:** `MightySchool.Entities/Entities/InventoryItem.cs`
- **Screen:** `/inventory-items`
- **Properties:** `int Id`, `string Name`, `int CategoryId` (FK → InventoryCategory), `string? Sku`, `decimal CostPrice` (money), `decimal SellingPrice` (money), `int Stock`, `bool? Status`

### InventorySale
- **Model:** `MightySchool.Entities/Entities/InventorySale.cs`
- **Screen:** `/inventory-sales`
- **Properties:** `int Id`, `string? Invoice`, `int StudentId` (FK), `DateTime? Date`, `decimal Payable` (money), `decimal Paid` (money), `decimal Due` (derived)
- **InventorySaleLine:** `int Id`, `int SaleId` (FK), `int ItemId` (FK), `int Qty`, `decimal UnitPrice`, `decimal Total`

---

## 23. Transport Management

### Bus
- **Model:** `MightySchool.Entities/Entities/Bus.cs`
- **Screen:** `/buses`
- **Properties:** `int Id`, `string BusNumber`, `string? Model`, `int Capacity`, `string? Registration`, `int? DriverId` (FK), `bool? Status`

### Driver
- **Model:** `MightySchool.Entities/Entities/Driver.cs`
- **Screen:** `/drivers`
- **Properties:** `int Id`, `string Name`, `string? Phone`, `string? LicenseNo`, `int? AssignedBusId` (FK → Bus), `bool? Status`

### BusRoute
- **Model:** `MightySchool.Entities/Entities/BusRoute.cs`
- **Screen:** `/bus-routes`
- **Properties:** `int Id`, `string RouteName`, `string? StartLocation`, `string? EndLocation`, `string? Distance`, `string? EstimatedTime`, `bool? Status`

### BusStop
- **Model:** `MightySchool.Entities/Entities/BusStop.cs`
- **Screen:** `/bus-stops`
- **Properties:** `int Id`, `string StopName`, `int RouteId` (FK → BusRoute), `decimal? Latitude`, `decimal? Longitude`, `int? Order`

### TransportMember
- **Model:** `MightySchool.Entities/Entities/TransportMember.cs`
- **Screen:** `/transport-members`
- **Properties:** `int Id`, `int StudentId` (FK), `int RouteId` (FK), `int StopId` (FK), `decimal Fare` (money), `bool? Status`

### VehicleType
- **Model:** `MightySchool.Entities/Entities/VehicleType.cs`
- **Screen:** `/vehicle-types`
- **Properties:** `int Id`, `string Name`, `bool? Status`

### VehicleCategory
- **Model:** `MightySchool.Entities/Entities/VehicleCategory.cs`
- **Screen:** `/vehicle-categories`
- **Properties:** `int Id`, `string Name`, `bool? Status`

### TransportHelper
- **Model:** `MightySchool.Entities/Entities/TransportHelper.cs`
- **Screen:** `/helpers`
- **Properties:** `int Id`, `string Name`, `string? Phone`, `int? AssignedBusId` (FK), `bool? Status`

### TransportCollection
- **Model:** `MightySchool.Entities/Entities/TransportCollection.cs`
- **Screen:** `/transport-collections` (+ report `/transport-reports/collection`)
- **Properties:** `int Id`, `string? Invoice`, `int StudentId` (FK), `int RouteId` (FK), `int? StopId` (FK), `string Month`, `decimal Fare`, `decimal ThisMonth` (money), `decimal Paid`, `decimal Due`, `DateTime? Date`

### TransportDashboard (view)
- **Model:** read-model projection (SP-backed, not stored)
- **Screen:** `/transport-dashboard`
- **Purpose:** KPI read-model (buses, routes, members, collections).

---

## 24. Google Meet

### GoogleMeetSession
- **Model:** `MightySchool.Entities/Entities/GoogleMeetSession.cs`
- **Screens:** `/google-meet`, `/google-meet/create`
- **Properties:**
  - `string Title`
  - `int TeacherId` (FK)
  - `int? ClassId` (FK), `int? SectionId` (FK), `int? GroupId` (FK), `int? SubjectId` (FK)
  - `string? Description`
  - `DateTime? StartDate`, `TimeSpan? StartTime`, `TimeSpan? EndTime`, `int Duration` (min)
  - `MeetVisibility Visibility` (enum: everyone/class/private)
  - `bool Enable`
  - recipients: `ICollection<int> StudentIds`, `ICollection<int> GuardianUserIds` (m2m notify)
  - recurring: `bool Recurring`, `RepeatType? Repeat` (enum), `DateTime? Until`
  - `string? MeetLink`, `bool? Status`
  - `string? Attachments` (file)

### GoogleCalendarOAuthSetting
- **Model:** stored inside `TenantGeneralSetting` (Google Meet tab), exposed via `POST /google-meet/test-connection`
- **Stored:** client credentials, service account JSON, calendar id (see tenant settings entity)

---

## 25. AI Assistant

### AiAssistantSetting
- **Model:** `MightySchool.Entities/Entities/AiAssistantSetting.cs`
- **Screen:** `/ai/settings`
- **Properties:** `int Id`, `string? AnthropicApiKey` (secret), `string? Model` (default), `bool Enabled`

### AiChatSession
- **Model:** `MightySchool.Entities/Entities/AiChatSession.cs`
- **Screen:** `/ai/chat`
- **Purpose:** Conversational assistant over institute data.
- **Properties:** `int Id`, `int UserId` (FK), `string? Thread`, `DateTime CreatedAt`

### AiChatMessage
- **Model:** `MightySchool.Entities/Entities/AiChatMessage.cs`
- **Screen:** `/ai/chat`
- **Properties:** `int Id`, `int SessionId` (FK → AiChatSession), `ChatRole Role` (enum: user/assistant), `string Content`, `DateTime CreatedAt`

### AiContentWriterOutput
- **Model:** request/response artifact (transient unless saved)
- **Screen:** `/ai/writer`
- **Purpose:** Generated text by type.
- **Properties (request):** `string ContentType`, `string? Prompt`, `string? Result`

### AiDataInsights
- **Model:** request/response read-model (SP-backed query)
- **Screen:** `/ai/insights`
- **Purpose:** NL queries over reports.
- **Properties (request):** `string Report`, `int? ClassId`, `DateTime? From`, `DateTime? To`, `string Question`, `string? Summary`

---

## 26. Auth, Profile & Public Forms

### AuthenticatedSession
- **Endpoints:** `POST /login`, `POST /logout`
- **Mechanics:** ASP.NET Core antiforgery cookie session (cookie auth) + antiforgery token on every state-changing request (equivalent of the audit `_token` CSRF).

### UserProfile
- **Model:** handled by `User` (see §18) + `UserProfileVM` in the Web layer
- **Screen:** `/profile`
- **Editable:** `string Name` (Full Name), `string? Email`, `string? Photo`, `string Password`, `string PasswordConfirmation` (demo-mode locked on this tenant)

### PublicContactSubmission
- **Model:** same record as admin `ContactMessage` (§16)
- **Endpoint:** `POST /contact-submit`
- **Properties:** `int Id`, `string Name`, `string? Email`, `string? Phone`, `string Message`, `DateTime CreatedAt`, `ContactStatus Status`

### PublicEvent (frontend mirror)
- **Route:** `/event-details/{id}`
- **Source entity:** `Event` (admin)

---

## 27. Cross-cutting / shared entities

| Entity | Used by | Notes |
|---|---|---|
| `AcademicYear` | Fees, Waivers, Migrations, Exams | current-year flag |
| `Class` / `Section` / `StudentGroup` | nearly every domain | cascading via `/sections-section-group-wise`, `/groups-class-section-wise` |
| `User` (+Rolable) | Auth, logs, notices audience | single sign-in for all roles |
| `Role` + `Permission` | RBAC | `/roles` |
| `AccountingLedger`/`AccountingFund` | Fees mapping, payroll mapping, accounts | double-entry backbone |
| `PaymentGateway` | Fee, hostel, transport, payroll collections | mode test/live |
| `SmsGateway` | SMS module | per-gateway creds |
| `AiAssistantSetting` | AI module | Anthropic key (secret) |

---

## Appendix: Complete Model Listing

Every **class** (C# model) in this inventory, grouped by module. Stored entities live in `MightySchool.Entities/Entities`; read-model/view names are SP-backed projections for Tabulator/AJAX listings.

### §1 Tenant & Core Platform
1. `Institute`
2. `Package`
3. `Branch`
4. `CustomDomain`
5. `TenantGeneralSetting`
6. `InstituteImageSetting`
7. `PaymentGateway`

### §2 Students Information
8. `Student`
9. `StudentMigration`
10. `StudentAtAGlance` (view)

### §3 Staffs Information
11. `Staff`
12. `Teacher`
13. `StaffAttendance`

### §4 Student Attendance
14. `StudentAttendanceRecord`
15. `ExamAttendance`
16. `ExamSchedule`
17. `AttendanceReportSummary` (view)
18. `AbsentFineReport` (view)

### §5 QR Code Attendance
19. `QRAttendanceScannerSession`

### §6 Academic Configuration
20. `AcademicYear`
21. `Shift`
22. `Class`
23. `Section`
24. `StudentGroup`
25. `Period`
26. `Subject`
27. `SubjectConfig`
28. `OptionalSubjectConfig`
29. `StudentOptionalSubject`
30. `Exam`
31. `StudentCategory`
32. `Department`
33. `Picklist`
34. `Signature`

### §7 Fees Management
35. `FeeHead`
36. `FeeMapping`
37. `FeeAmountConfig`
38. `FeeDateConfig`
39. `AttendanceFineWaiver`
40. `Waiver`
41. `WaiverConfig`
42. `FeeCollection`
43. `PaymentFeeInfo` (view)
44. `UnpaidFeeInfo` (view)

### §8 Accounts Management
45. `AccountingCategory`
46. `AccountingGroup`
47. `AccountingLedger`
48. `AccountingFund`
49. `CashTransaction` (+ `CashTransactionLine`)
50. `ContraTransfer`
51. `JournalTransaction` (+ `JournalLine`)
52. `FundTransfer`

### §9 Accounting Reports (all views)
53. `BalanceSheet` (view)
54. `TrialBalance` (view)
55. `CashFlowStatement` (view)
56. `CashFlowDetail` (view)
57. `CashBook` (view)
58. `LedgerBook` (view)
59. `IncomeStatement` (view)
60. `IncomeStatementDetail` (view)
61. `CashSummary` (view)

### §10 Payroll Management
62. `SalaryHead`
63. `PayrollMapping`
64. `PayrollAssign`
65. `SalaryPayment`
66. `DueSalaryPayment`
67. `AdvanceSalaryPayment`
68. `ReturnAdvancePayment`
69. `SalaryStatement` (view)

### §11 Routine Management
70. `Syllabus`
71. `Assignment`
72. `ClassRoutine`
73. `ExamRoutine`

### §12 Library Management
74. `BookCategory`
75. `Book`
76. `LibraryMember`
77. `BookIssue`

### §13 Exam Module
78. `ExamStartup`
79. `ExamCodeList`
80. `GradeList`
81. `ExamMarkConfig`
82. `ExamRemark`
83. `ExamMark`
84. `ExamResult` (computed)
85. `GrandFinalResult` (computed)
86. `TabulationSheet` (view)
87. `MeritListSheet` (view)
88. `ResultCardSetting`
89. `AssessmentDomain` (+ `AssessmentDomainItem`)
90. `DomainAssessmentEntry`
91. `OnlineExam`

### §14 Layout & Certificates
92. `CertificateTemplate`
93. `CertificateLayout` (10 types, discriminant)

### §15 SMS Module
94. `SmsTemplate`
95. `PhoneBookCategory`
96. `PhoneBookContact`
97. `SmsSend` (+ `SmsSentLog`)
98. `SmsPurchase`
99. `SmsReport` (view)

### §16 Administrator
100. `ShiftAssignment`
101. `SubjectAssignment`
102. `ClassAssignment`
103. `Notice`
104. `Event`
105. `ContactMessage`
106. `UserActivityLog`
107. `StudentIdCardBatch` (print artifact)
108. `TeacherIdCardBatch` (print artifact)
109. `StaffIdCardBatch` (print artifact)

### §17 System
110. `ModuleRegistry`
111. `SystemUpdate`
112. `UpdateHistory`
113. `SystemSetting`

### §18 Master Configuration
114. `Role`
115. `User`

### §19 CMS Management
116. `AdmissionApplication`
117. `CmsPage`
118. `Banner`
119. `AboutUsItem`
120. `Faq`
121. `GalleryImage`
122. `MobileAppSection`
123. `WhyChooseUsItem`
124. `Policy`
125. `ReadyToJoinUsItem`
126. `Testimonial`
127. `AchievementCounter` (static content block)

### §20 WhatsApp
128. `WhatsAppSetting`
129. `WhatsAppTemplate`
130. `WhatsAppLog`

### §21 Hostel Management
131. `Hostel`
132. `HostelCategory`
133. `HostelMember`
134. `HostelBuilding`
135. `HostelFloor`
136. `HostelRoom`
137. `Bed`
138. `RoomMember`
139. `Meal`
140. `MealPlan`
141. `MealEntry`
142. `HostelBill`
143. `HostelLeave`
144. `HostelCollection`
145. `HostelSeatMap` (view)

### §22 Inventory
146. `InventoryCategory`
147. `InventoryItem`
148. `InventorySale` (+ `InventorySaleLine`)

### §23 Transport Management
149. `Bus`
150. `Driver`
151. `BusRoute`
152. `BusStop`
153. `TransportMember`
154. `VehicleType`
155. `VehicleCategory`
156. `TransportHelper`
157. `TransportCollection`
158. `TransportDashboard` (view)

### §24 Google Meet
159. `GoogleMeetSession`

### §25 AI Assistant
160. `AiAssistantSetting`
161. `AiChatSession` (+ `AiChatMessage`)

### §26 Auth, Profile & Public Forms
162. `User` (also §18)
163. `ContactMessage` (public mirror of §16)

### §27 Cross-cutting / shared (referenced above)
- `AcademicYear`, `Class`, `Section`, `StudentGroup`, `User`, `Role`, `Permission`, `AccountingLedger`, `AccountingFund`, `PaymentGateway`, `SmsGateway`, `AiAssistantSetting`

---

*End of entity inventory. ~190 entities inferred behaviorally from the UI; no database access performed — read-only audit, nothing changed on the target. Presented as .NET EF Core models targeting `MightySchool.Entities` + SQL Server.*