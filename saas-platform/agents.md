# AGENTS.md

# Multi-Product SaaS Development Rules

This document defines the mandatory architecture and coding rules for the Multi-Product SaaS project.

The agent MUST follow these rules when creating, modifying, refactoring, or reviewing code.

---

# 1. Solution Structure

The solution contains exactly these five projects:

```text
MultiProduct.SaaS.sln
│
├── MultiProduct.Web
├── MultiProduct.Application
├── MultiProduct.Infrastructure
├── MultiProduct.Interfaces
└── MultiProduct.Entities
```

Product modules (School, Rent, Transport) are namespaces with these projects (e.g. `MultiProduct.Application.Modules.School`), NOT separate projects.

Do NOT create additional projects such as:

```text
MultiProduct.Domain
MultiProduct.Persistence
MultiProduct.Data
MultiProduct.Repositories
MultiProduct.Services
MultiProduct.School
```

unless explicitly requested by the developer.

---

# 2. Project Responsibilities

## MultiProduct.Web

ASP.NET Core MVC presentation project.

Contains:

```text
Controllers
Views
wwwroot
Program.cs
appsettings.json
```

Responsibilities:

* MVC Controllers
* Razor Views
* Tabulator
* AJAX endpoints
* ModelState handling
* Authentication/authorization UI concerns
* HTTP request/response handling
* Entitlement-driven shell (menu visibility by tenant subscription)

MultiProduct.Web MUST NOT:

* Access DbContext directly.
* Execute Stored Procedures.
* Execute SQL.
* Contain business logic.
* Contain repository implementation.
* Contain UnitOfWork implementation.

---

## MultiProduct.Application

Application/business-logic layer.

Contains:

```text
Services
Modules\School
Modules\Rent
Modules\Transport
Common
Documents
```

`Common` holds application-layer helpers shared by Services (e.g. `PasswordHasher.cs`, `CompanyScope.cs`).

Do NOT put domain/shared template data in MultiProduct.Application/Common when it must also be
used by MultiProduct.Infrastructure, because MultiProduct.Application and MultiProduct.Infrastructure
must never depend on each other. That data belongs in MultiProduct.Entities/Common instead.

Responsibilities:

* Business/application logic
* Service implementations
* Business validation
* Application workflows
* Entity/ViewModel mapping
* Document generation (Excel/CSV/PDF)
* Reusable application models/results

MultiProduct.Application MUST NOT:

* Access DbContext directly.
* Execute SQL.
* Execute Stored Procedures.
* Use ADO.NET directly.
* Use Dapper directly.
* Contain repository implementation.
* Contain UnitOfWork implementation.
* Contain EF Core packages.

---

## MultiProduct.Infrastructure

Data-access implementation layer.

Contains:

```text
Data
Configurations
Migrations
Repositories
UnitOfWork
```

Responsibilities:

* EF Core DbContext
* EF Core entity configurations
* EF Core migrations
* Generic Repository implementation
* UnitOfWork implementation
* Database access
* Stored Procedure execution

This is the only project where concrete database-access implementations belong.

MultiProduct.Infrastructure MUST NOT:

* Contain business logic.
* Contain MVC logic.
* Contain View/UI logic.
* Depend on MultiProduct.Web or MultiProduct.Application.

---

## MultiProduct.Interfaces

Contains interfaces/contracts only.

Contains:

```text
Services
Repositories
UnitOfWork
ViewModels
Documents
```

Examples:

```text
IGenericCrudService.cs
IStudentService.cs

IGenericRepository.cs

IUnitOfWork.cs
```

MultiProduct.Interfaces MUST NOT contain:

* Concrete implementations
* DbContext
* EF Core database code
* SQL execution
* Stored Procedure execution
* Business logic
* MVC code

---

## MultiProduct.Entities

Contains domain/database entities only.

Contains:

```text
Entities
Enums
Common
```

`Common` holds shared domain constant/template data that must be used by both
MultiProduct.Application and MultiProduct.Infrastructure (e.g. `PredefinedRoleTemplate.cs`).
These projects cannot depend on each other, so such shared data belongs here.
Do NOT put application use-case logic in MultiProduct.Entities/Common.

Examples:

```text
BaseEntity.cs
Company.cs
User.cs
Subscription.cs
```

MultiProduct.Entities MUST NOT depend on:

```text
MultiProduct.Web
MultiProduct.Application
MultiProduct.Infrastructure
MultiProduct.Interfaces
EF Core infrastructure
MVC
Controllers
Repositories
Services
UnitOfWork
```

Keep Entities clean.

---

# 3. Project Dependency Rules

Required dependency direction:

```text
MultiProduct.Web
   ↓
MultiProduct.Application          MultiProduct.Infrastructure
   ↓                                  ↓
MultiProduct.Interfaces               MultiProduct.Interfaces
   ↓
MultiProduct.Entities
```

More specifically:

```text
MultiProduct.Interfaces
    → MultiProduct.Entities

MultiProduct.Application
    → MultiProduct.Interfaces
    → MultiProduct.Entities

MultiProduct.Infrastructure
    → MultiProduct.Interfaces
    → MultiProduct.Entities

MultiProduct.Web
    → MultiProduct.Application
    → MultiProduct.Infrastructure
    → MultiProduct.Interfaces
    → MultiProduct.Entities (only when required)
```

MultiProduct.Application and MultiProduct.Infrastructure are siblings:

```text
MultiProduct.Application
    → MultiProduct.Infrastructure   FORBIDDEN

MultiProduct.Infrastructure
    → MultiProduct.Application      FORBIDDEN
```

They communicate through the contracts in MultiProduct.Interfaces.
Dependency wiring happens only in MultiProduct.Web.

Never create circular dependencies.

Forbidden:

```text
MultiProduct.Entities → MultiProduct.Application
MultiProduct.Entities → MultiProduct.Web
MultiProduct.Entities → MultiProduct.Infrastructure

MultiProduct.Interfaces → MultiProduct.Application
MultiProduct.Interfaces → MultiProduct.Web
MultiProduct.Interfaces → MultiProduct.Infrastructure

MultiProduct.Application → MultiProduct.Web
MultiProduct.Infrastructure → MultiProduct.Web
```

---

# 4. Mandatory Application Flow

The standard architecture is:

```text
Controller                    (MultiProduct.Web)
    ↓
Service                       (MultiProduct.Application)
    ↓
IUnitOfWork                   (MultiProduct.Interfaces)
    ↓
UnitOfWork                    (MultiProduct.Infrastructure)
    ↓
GenericRepository             (MultiProduct.Infrastructure)
    ↓
EF Core / Stored Procedure    (MultiProduct.Infrastructure)
    ↓
SQL Server
```

Never bypass this flow.

---

# 5. Controller Rules

Controllers must remain thin.

Controllers may:

* Receive HTTP requests.
* Bind ViewModels.
* Validate ModelState.
* Call Services.
* Return Views.
* Return JSON.
* Redirect.

Controllers MUST NOT:

```text
Access DbContext
Execute SQL
Execute Stored Procedures
Access GenericRepository directly
Access UnitOfWork directly
Contain business logic
Contain database queries
Contain transaction logic
```

Bad:

```csharp
var items = await _context.Students.ToListAsync();
```

Bad:

```csharp
await connection.QueryAsync(...);
```

Bad:

```csharp
await _unitOfWork.Repository<Student>()
    .ExecuteSpAsync<StudentListVM>(...);
```

The Controller must call a Service instead.

Correct:

```csharp
var result = await _studentService.GetListAsync(model);

return Json(result);
```

---

# 6. Service Rules

The Service layer owns ALL business/application logic.

Services are responsible for:

* Business rules
* Application workflows
* Business validation
* Entity/ViewModel mapping
* Preparing repository parameters
* Coordinating repository operations
* Deciding which data operation is required
* Calling UnitOfWork
* Coordinating transactions when necessary

Services MUST NOT:

```text
Execute SQL
Execute Stored Procedures directly
Use ADO.NET directly
Use Dapper directly
Use DbContext directly
Use DbSet directly
Manage raw database connections
```

The Service may REQUEST a repository operation.

Example:

```csharp
var result = await _unitOfWork
    .Repository<Student>()
    .ExecuteSpAsync<StudentListVM>(
        "sp_StudentList",
        parameters);
```

This is allowed.

The actual Stored Procedure execution MUST happen inside the Repository implementation.

---

# 7. UnitOfWork Rules

UnitOfWork is a separate component.

Structure:

```text
MultiProduct.Infrastructure
└── UnitOfWork
    └── UnitOfWork.cs

MultiProduct.Interfaces
└── UnitOfWork
    └── IUnitOfWork.cs
```

UnitOfWork is responsible for:

* Providing repositories
* Coordinating repositories
* SaveChangesAsync
* BeginTransactionAsync
* CommitTransactionAsync
* RollbackTransactionAsync

UnitOfWork MUST NOT contain:

* Business logic
* SQL queries
* Stored Procedure implementation
* Entity-specific business rules
* MVC logic

UnitOfWork coordinates data access.

---

# 8. Generic Repository Rules

There is ONE Generic Repository.

Structure:

```text
MultiProduct.Interfaces
└── Repositories
    └── IGenericRepository.cs

MultiProduct.Infrastructure
└── Repositories
    └── GenericRepository.cs
```

The Generic Repository handles:

## EF Core

* Create
* Read
* Update
* Delete
* GetById
* GetAll
* Exists

## Stored Procedures

* Complex listing
* Complex read
* Search
* Filtering
* Sorting
* Pagination
* Reporting
* Aggregation
* Performance-sensitive queries

There is NO separate StoredProcedureRepository by default.

Do NOT create:

```text
IStoredProcedureRepository
StoredProcedureRepository
```

unless explicitly requested.

---

# 9. Generic CRUD Rule

ALL standard CRUD MUST be generic.

Use:

```text
IGenericRepository<TEntity>
GenericRepository<TEntity>

IGenericCrudService<TEntity, TVM, TListVM>
GenericCrudService<TEntity, TVM, TListVM>
```

## 9.1 Generic CRUD Service (the ONE generic service)

`GenericCrudService<TEntity, TVM, TListVM>`
(`MultiProduct.Application/Services/GenericCrudService.cs`) is the project's
single reusable generic CRUD implementation. It powers ALL simple
CRUD/master-data modules (School classes/subjects, Rent properties/units,
Transport vehicles/routes).

Contract (`MultiProduct.Interfaces/Services/IGenericCrudService.cs`):

```text
TabulatorResponse<TListVM> GetListAsync(TabulatorRequest)     // SP-backed paged listing
TListVM? GetByIdAsync(int id)                                 // scoped
List<MasterDataDropdownVM> GetActiveAsync(int? companyId = null)
(bool Success, string Message) CreateAsync(TVM model, int? createdBy = null)
(bool Success, string Message) UpdateAsync(TVM model, int? updatedBy = null)
(bool Success, string Message) DeleteAsync(int id, int? updatedBy = null)  // soft delete
```

Generic constraints:

```text
TEntity : BaseEntity, IMasterDataEntity, new()
TVM     : class, IMasterDataViewModel, new()
TListVM : class, IHasTotalCount, new()
```

The implementation already provides:

* Company-scoped filtering (e.g. CompanyScope)
* Duplicate code/name validation per Company
* Optional per-module validator delegate
* Soft delete (`IsActive = false`, `IsDeleted = true`)
* Audit log records
* `IsDefault` single-default-per-company handling
* Entity/VM/list-VM mapping

Per-module configuration is supplied through DI in `MultiProduct.Web/Program.cs`
(`RegisterGenericCrudService<TEntity, TVM, TListVM>`): entity type, VM types,
list SP name, display name, optional validator factory. NEVER create a separate
service class per simple CRUD module.

Forbidden names (removed/obsolete, do not reintroduce):

```text
MasterDataService
GenericService
CompanyService
```

Simple CRUD modules MUST flow through:

```text
Simple CRUD module
    ↓
IGenericCrudService<TEntity, TVM, TListVM>
    ↓
GenericCrudService<TEntity, TVM, TListVM>
    ↓
IUnitOfWork
    ↓
IGenericRepository<TEntity>
    ↓
EF Core / Stored Procedure
```

Modules with genuine business workflows (e.g. Student, Lease)
MUST NOT be forced into GenericCrudService — they get a dedicated
service when required.

Do NOT duplicate standard CRUD implementations.

Forbidden unless explicitly justified:

```text
StudentRepository
StudentService
```

just for basic CRUD.

---

# 10. Generic Repository Contract

The repository should follow this general pattern:

```csharp
public interface IGenericRepository<TEntity>
    where TEntity : class
{
    Task<TEntity?> GetByIdAsync(int id);

    Task<List<TEntity>> GetAllAsync();

    Task AddAsync(TEntity entity);

    void Update(TEntity entity);

    void Delete(TEntity entity);

    Task<bool> ExistsAsync(int id);

    Task<List<TResult>> ExecuteSpAsync<TResult>(
        string procedureName,
        object? parameters = null);

    Task<TResult?> ExecuteSpSingleAsync<TResult>(
        string procedureName,
        object? parameters = null);
}
```

Do not add repository methods without a real reusable requirement.

---

# 11. Stored Procedure Rules

Stored Procedures are preferred for:

* Complex listing
* Complex search
* Multiple filters
* Complex JOINs
* Server-side pagination
* Server-side sorting
* Reports
* Aggregation
* Dashboard queries
* Performance-sensitive queries

Stored Procedures should generally NOT be used for simple CRUD.

Normal CRUD uses EF Core.

---

# 12. Stored Procedure Execution Rule

This rule is mandatory:

```text
Service
    ↓
UnitOfWork
    ↓
GenericRepository
    ↓
Stored Procedure
```

The Service does NOT execute the Stored Procedure.

The Repository executes the Stored Procedure.

Correct responsibility:

```text
Service:
"What data does the application need?"

Repository:
"How do I retrieve that data from the database?"
```

---

# 13. EF Core Rules

EF Core is the default for normal transactional CRUD.

Use EF Core for:

```text
Create
Read By ID
Update
Delete
Simple Read
Simple GetAll
```

EF Core implementation belongs in:

```text
MultiProduct.Infrastructure
```

The DbContext belongs in:

```text
MultiProduct.Infrastructure
└── Data
    └── ApplicationDbContext.cs
```

Entity configurations belong in:

```text
MultiProduct.Infrastructure
└── Configurations
```

Migrations belong in:

```text
MultiProduct.Infrastructure
└── Migrations
```

---

# 14. DbContext Rules

Only MultiProduct.Infrastructure (Repository/UnitOfWork infrastructure) may use DbContext.

Do NOT inject:

```text
ApplicationDbContext
```

into:

```text
Controllers
Services
ViewModels
Entities
```

Services access database operations through UnitOfWork/Repository.

---

# 15. Specific Services

Specific Services are allowed when real business logic exists.

Example:

```text
IStudentService
StudentService
```

is appropriate when Student contains:

* Role rules
* Permission rules
* Company restrictions
* Student-specific validation
* Complex workflows
* Complex listing orchestration

Do NOT create a specific Service just to wrap generic CRUD without adding business logic.

---

# 16. Specific Repositories

Specific Repositories are NOT the default.

Prefer:

```text
GenericRepository<TEntity>
```

for:

* CRUD
* Simple reads
* Stored Procedure execution

Only create a specific Repository when there is a genuine database-access requirement
that cannot reasonably be handled by the Generic Repository.

---

# 17. Tabulator Rules

Tabulator belongs to MultiProduct.Web.

Tabulator code belongs in:

```text
MultiProduct.Web
├── Views
└── wwwroot/js
```

Tabulator communicates with MVC through AJAX.

Required flow:

```text
Tabulator
    ↓ AJAX
Controller
    ↓
Service
    ↓
UnitOfWork
    ↓
GenericRepository
    ↓
Stored Procedure
    ↓
SQL Server
    ↓
Controller JSON
    ↓
Tabulator
```

Tabulator MUST NOT communicate directly with the database.

---

# 18. Tabulator Controller Rules

Tabulator endpoints must remain thin.

Example:

```csharp
[HttpGet]
public async Task<IActionResult> GetList(
    StudentSearchVM model)
{
    var result =
        await _studentService.GetListAsync(model);

    return Json(result);
}
```

The Controller must not:

* Execute SQL
* Execute Stored Procedures
* Query DbContext
* Perform database pagination
* Perform database filtering
* Perform database sorting

---

# 19. Tabulator Server-Side Processing

For large tables, use server-side:

* Pagination
* Filtering
* Sorting
* Searching

Prefer Stored Procedures for complex server-side operations.

Do not load thousands of records into memory just to let Tabulator paginate them.

---

# 20. ViewModel Rules

ViewModels belong in MultiProduct.Interfaces (with the application contracts).

Use ViewModels for:

* Form input
* Search/filter input
* Tabulator rows
* Report results
* UI-specific data

Do not expose database entities directly to the UI when a ViewModel is appropriate.

Example:

```csharp
public class StudentListVM
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    // ... additional display fields
}
```

---

# 21. Entity Rules

Entities belong to:

```text
MultiProduct.Entities
```

Entities must remain independent of:

* MVC
* Controllers
* Services
* Repositories
* UnitOfWork

Do not put business workflows inside entities unless the project explicitly adopts
a domain-model pattern requiring it.

---

# 22. Mapping Rules

Mapping must not be placed in Controllers.

Use the Service/application layer or the established mapping mechanism.

Typical flow:

```text
ViewModel
    ↓
Service
    ↓
Entity
```

and:

```text
Entity / SP Result
    ↓
Service
    ↓
ViewModel
```

Do not duplicate identical mapping logic across Controllers.

---

# 23. Validation Rules

Basic UI/input validation:

```text
ViewModel
+
DataAnnotations
+
ModelState
```

Business validation:

```text
Service
```

Example:

```text
Required field
    → ViewModel validation

User cannot assign restricted role
    → Service business rule
```

---

# 24. Transaction Rules

Use UnitOfWork transactions when a business operation modifies multiple related records.

Example:

```text
Create Student
    +
Assign Guardian
    +
Assign Permissions
```

Flow:

```text
Service
    ↓
BeginTransaction
    ↓
Repository operations
    ↓
SaveChanges
    ↓
Commit
```

If an operation fails:

```text
Rollback
```

Transactions must not be implemented in Controllers.

---

# 25. Async Rules

Use asynchronous APIs for database operations.

Prefer:

```text
GetByIdAsync
GetAllAsync
AddAsync
CreateAsync
UpdateAsync
DeleteAsync
ExecuteSpAsync
SaveChangesAsync
```

Avoid blocking calls such as:

```csharp
.Result
.Wait()
```

Do not introduce synchronous database calls when an async API is available.

---

# 26. SQL Safety

Never concatenate user input into SQL.

Bad:

```csharp
$"EXEC sp_StudentList '{search}'"
```

Use parameterized execution.

Stored Procedure parameters must be passed separately from the SQL/procedure name.

---

# 27. No Architecture Drift

The agent MUST NOT automatically introduce:

```text
CQRS
MediatR
Generic UnitOfWork wrappers
Repository per entity
Service per entity
Specification Pattern
Auto-generated factories
Domain Events
Additional Domain project
Additional Persistence project
```

unless explicitly requested.

Do not over-engineer the project.

---

# 28. No Duplicate Abstractions

Before creating a new:

```text
Interface
Repository
Service
Helper
Manager
Wrapper
Base class
```

the agent MUST first check whether an existing generic/reusable implementation
can satisfy the requirement.

Prefer reuse.

---

# 29. Error Handling

Do not silently swallow exceptions.

Bad:

```csharp
try
{
    ...
}
catch
{
}
```

Exceptions should be:

* Handled meaningfully at the appropriate boundary.
* Logged when appropriate.
* Converted to a suitable application response where required.

Do not put global exception handling into every Controller action.

---

# 30. Logging

Use the application's configured logging infrastructure.

Do not use:

```csharp
Console.WriteLine(...)
```

as permanent application logging.

Do not log:

* Passwords
* Tokens
* Secrets
* Connection strings
* Sensitive user information

---

# 31. Configuration

Keep application configuration in:

```text
MultiProduct.Web
├── appsettings.json
└── appsettings.Development.json
```

Do not hardcode:

* Connection strings
* API keys
* Passwords
* Secrets
* Environment-specific values

---

# 32. Database Scripts

Stored Procedures belong under the project's database script area.

Organize by module:

```text
Database
└── StoredProcedures
    ├── School
    ├── Rent
    ├── Transport
    └── Billing
```

Use clear names:

```text
sp_StudentList
sp_RentCollectionReport
sp_TripCollectionReport
```

Do not place SQL execution code in Controllers or Services.

---

# 33. Naming Conventions

Use clear .NET naming conventions.

Examples:

```text
StudentController

StudentVM
StudentListVM
StudentSearchVM

IGenericRepository
GenericRepository

IUnitOfWork
UnitOfWork

IGenericCrudService
GenericCrudService

IStudentService
StudentService
```

Async methods must end with:

```text
Async
```

Examples:

```text
GetByIdAsync
CreateAsync
UpdateAsync
DeleteAsync
ExecuteSpAsync
SaveChangesAsync
```

---

# 34. Code Quality

When modifying code:

1. Reuse existing abstractions.
2. Keep methods focused.
3. Keep Controllers thin.
4. Keep Services responsible for business logic.
5. Keep Repositories responsible for database access.
6. Keep UnitOfWork responsible for coordination and transactions.
7. Avoid unnecessary abstractions.
8. Avoid duplicated code.
9. Prefer clear code over clever code.
10. Preserve existing conventions unless explicitly asked to change them.

---

# 35. Mandatory Architecture Diagram

The agent must preserve this architecture:

```text
                    MultiProduct.Web
                       │
                       ▼
                  Controller
                       │
                       ▼
              MultiProduct.Application
                   Service
                       │
                       ▼
              MultiProduct.Interfaces
                   IUnitOfWork
                       │
                       ▼
             MultiProduct.Infrastructure
                   UnitOfWork
                       │
                       ▼
              Generic Repository
                  │         │
                  │         │
                  ▼         ▼
               EF Core      SP
                  │         │
                  └────┬────┘
                       ▼
                   SQL Server
```

Responsibilities:

```text
MultiProduct.Web
    = MVC / UI / Tabulator / HTTP

MultiProduct.Application
    = Business Logic + Services + Document Generation

MultiProduct.Infrastructure
    = Data Access + EF Core + Stored Procedure Execution

MultiProduct.Interfaces
    = Contracts + ViewModels

MultiProduct.Entities
    = Entities
```

---

# 36. Absolute Golden Rules

These rules have priority over convenience.

1. **All business logic belongs in Services.**
2. **All actual database execution belongs in Repositories.**
3. **All standard CRUD must use Generic Repository.**
4. **EF Core handles normal transactional CRUD.**
5. **Generic Repository executes Stored Procedures.**
6. **Stored Procedures handle complex reads/listing/reporting/filtering/aggregation/performance-sensitive operations.**
7. **Service may request an SP operation but must never execute the SP itself.**
8. **UnitOfWork is separate and handles repository coordination and transactions.**
9. **Controllers remain thin.**
10. **Tabulator belongs to MultiProduct.Web.**
11. **Tabulator communicates through MVC AJAX endpoints.**
12. **Do not access DbContext directly from Controllers or Services.**
13. **Do not create duplicate CRUD repositories/services.**
14. **Specific Services are created only for genuine business logic.**
15. **Specific Repositories are created only for genuine database-access requirements.**
16. **Do not create additional projects without explicit approval.**
17. **Do not bypass architectural layers.**
18. **Do not over-engineer.**
19. **Prefer existing generic abstractions over new duplicated abstractions.**
20. **Preserve this architecture when implementing new modules.**
21. **Repositories, UnitOfWork, DbContext, Configurations, Migrations, and EF packages belong in MultiProduct.Infrastructure.**
22. **Services, business logic, and document generation belong in MultiProduct.Application.**
23. **MultiProduct.Application and MultiProduct.Infrastructure must never depend on each other; they communicate only through MultiProduct.Interfaces.**
24. **Modules (School, Rent, Transport) share the five projects; they integrate only through the event bus — never direct cross-module queries.**

---

# 37. New Module Rule

When adding a new module, follow this order:

```text
1.  Add Entity (MultiProduct.Entities)
2.  Add ViewModel(s) (MultiProduct.Interfaces)
3.  Determine whether generic CRUD is sufficient
4.  Use GenericCrudService for standard CRUD (MultiProduct.Application)
5.  Create Specific Service only if business logic requires it (MultiProduct.Application)
6.  Use Generic Repository for database access (MultiProduct.Infrastructure)
7.  Use EF Core for normal CRUD (MultiProduct.Infrastructure)
8.  Add Stored Procedure only for complex operations
9.  Execute the SP through Generic Repository (MultiProduct.Infrastructure)
10. Add Controller (MultiProduct.Web) — CRUD modules use the combined
    CreateEdit(int? id) GET + CreateEdit(VM) POST actions with the
    CanUseFormAsync(isEdit) permission gate (§39.5.1)
11. Add Razor Views (MultiProduct.Web) — one combined CreateEdit.cshtml per
    module; NO separate Create.cshtml/Edit.cshtml
12. Add Tabulator only when server-side listing requires it (MultiProduct.Web)
```

Never automatically create:

```text
NewEntityRepository
NewEntityService
NewEntityUnitOfWork
NewEntityStoredProcedureRepository
```

for every module.

---

# 38. Final Decision Rule

When unsure where code belongs, use this:

```text
Is it HTTP/UI?
    → MultiProduct.Web / Controller / View

Is it business logic?
    → MultiProduct.Application / Service

Is it a contract?
    → MultiProduct.Interfaces

Is it an entity?
    → MultiProduct.Entities

Is it database execution?
    → MultiProduct.Infrastructure / Repository

Is it transaction/repository coordination?
    → MultiProduct.Infrastructure / UnitOfWork

Is it DbContext / Configurations / Migrations?
    → MultiProduct.Infrastructure / Data

Is it normal CRUD?
    → Generic Repository + EF Core (MultiProduct.Infrastructure)

Is it complex listing/report/filter/aggregation?
    → Generic Repository + Stored Procedure (MultiProduct.Infrastructure)

Is it document generation (Excel/CSV/PDF)?
    → MultiProduct.Application / Documents

Is it Tabulator?
    → MultiProduct.Web

Is it UI permission/button hiding?
    → MultiProduct.Web / View (+ PageAccessExtensions in MultiProduct.Web/Common),
      contract in MultiProduct.Interfaces (PageAccessVM, IPermissionService.GetAccessAsync),
      logic in MultiProduct.Application (PermissionService)

Is it a cross-module integration (e.g. School → Transport)?
    → Async event via the event bus; NEVER a direct module query
```

---

# 39. Permission-Based Conditional UI Rules

Server-side authorization (`MenuAuthorize`) is the backstop; the UI must ALSO
hide the actions the current user cannot perform (belt-and-suspenders approach).

## 39.1 When to Apply

Every view that renders Create/Edit/Delete/Assign/Print/Export UI — Index
listings, Details pages, form pages — MUST hide buttons/actions the current user
is not permitted to perform. Showing a button that leads to "Access Denied" is
considered bad UX and is forbidden.

## 39.2 Contract: PageAccessVM (MultiProduct.Interfaces)

`PageAccessVM` (`MultiProduct.Interfaces/ViewModels/PageAccessVM.cs`) carries the
per-role flags for one controller/page:

```csharp
public class PageAccessVM
{
    public bool CanView { get; set; }
    public bool CanCreate { get; set; }
    public bool CanEdit { get; set; }
    public bool CanDelete { get; set; }
    public bool CanPrint { get; set; }
    public bool CanExport { get; set; }

    public static PageAccessVM FullAccess => new() { ... };
}
```

## 39.3 Service Contract: GetAccessAsync

`IPermissionService.GetAccessAsync(int userRoleId, string controller)` returns
`Task<PageAccessVM?>`.

`PermissionService.GetAccessAsync` (MultiProduct.Application/Services):

* Returns `PageAccessVM.FullAccess` for the Platform Admin role.
* Otherwise maps the role's `RoleWiseMenuAccess` row for the controller.
* Returns `null` when no role or access row exists.

## 39.4 View Helper: PageAccessExtensions (MultiProduct.Web/Common)

`User.GetPageAccessAsync(IPermissionService, ViewContext)` is an extension on
`ClaimsPrincipal` living in `MultiProduct.Web/Common/PageAccessExtensions.cs`
(namespace `MultiProduct.Web.Common`, imported into views via `_ViewImports`).

It:

* Returns `PageAccessVM.FullAccess` for admin-level roles
  (resolved through `PermissionService` from the `RoleId` claim).
* Reads the `RoleId` claim.
* Resolves the controller name from `ViewContext.RouteData.Values["controller"]`.
* Never returns `null` (defaults to an empty `PageAccessVM`).

## 39.5 Mandatory View Pattern

Index/Details views compute permissions at the top:

```cshtml
@inject MultiProduct.Interfaces.Services.IPermissionService PermissionService
@{
    ViewData["Title"] = "Student";
    var access = await User.GetPageAccessAsync(PermissionService, ViewContext);
    var canCreate = access.CanCreate;
    var canEdit = access.CanEdit;
    var canDelete = access.CanDelete;
}
```

Create button:

```cshtml
@if (canCreate)
{
    <a asp-action="CreateEdit" class="btn btn-primary btn-sm">
        <i class="bi bi-plus-lg"></i> Create Student
    </a>
}
```

Actions column formatter (view always; edit/delete gated by `@if`):

```js
formatter: function (cell) {
    var id = parseInt(cell.getRow().getData().id);
    var actions = `
        <a class="btn btn-sm btn-info me-1 view-btn" href='@Url.Action("Details", "Student")/${id}'>
            <i class="bi bi-eye"></i>
        </a>`;
    @if (canEdit)
    {
        <text>actions += `<button class="btn btn-sm btn-warning me-1 edit-btn"><i class="bi bi-pencil"></i></button>`;</text>
    }
    @if (canDelete)
    {
        <text>actions += `<button class="btn btn-sm btn-danger delete-btn"><i class="bi bi-trash"></i></button>`;</text>
    }
    return actions;
},
```

## 39.5.1 Mandatory Combined Create/Edit Pattern (CreateEdit)

Every CRUD module MUST use ONE combined create/edit form — there are NO separate
`Create`/`Edit` actions or views:

* Controller: `CreateEdit(int? id)` (GET) + `CreateEdit(TVM model)` (POST).
  Mode = `id > 0` / `model.Id > 0`; pass `ViewBag.IsEdit` to the view.
* Permission gate inside BOTH actions via a private `CanUseFormAsync(isEdit)`
  helper → `_permissionService.CanCreateAsync(roleId, "Student")` /
  `CanEditAsync(...)` from the `RoleId` claim; redirect to
  `Account/AccessDenied` when denied. Keep plain `[MenuAuthorize]` on the
  action as the coarse backstop.
* View: single `CreateEdit.cshtml` rendering both modes (title/layout switch on
  `ViewBag.IsEdit`); form posts to `asp-action="CreateEdit"` with a hidden `Id`.
* Do NOT reintroduce separate `Create.cshtml`/`Edit.cshtml` views or
  permission-specific GET form actions for CRUD modules.

## 39.6 Rules

1. Do NOT compute these permission flags in controllers — use the view extension
   (single source of truth, no controller churn).
2. Hiding a button MUST NOT be the only protection. `MenuAuthorize` must still
   deny direct URL access; the Access Denied page remains the backstop.
3. Do NOT gate the read/View affordance behind anything other than `CanView`.
4. Razor conditionals inside Tabulator JS are allowed and are the established
   pattern; emit booleans or use `@if (flag) { <text>...</text> }` blocks.
5. Keep the same pattern in every new view (Index, Details, Assign) — do not
   create a per-controller/per-view permission helper.

---

# 40. Multi-Level RBAC Hierarchy

Access control has 2 distinct, non-interchangeable levels:

```text
LEVEL 1 — Platform (global, multi-tenant)
    Platform Admin = the single global UserRole (Scope = Platform,
    detected via Users.PlatformRoleId)
      ├── Plan & Module Management
      ├── Company (Tenant) Management
      └── Billing & Invoicing

LEVEL 2 — Company (per-tenant)
    Company Admin (implicit full access within its own company only)
    └── 8 operational roles
          → governed by RoleWiseMenuAccess rows seeded per company
```

## 40.1 Mandatory Rules

1. **Platform detection is role-only.** The global **Platform Admin role**
   is the RBAC source of truth. Platform scope is detected via the
   `RoleScope` claim. NEVER use role name matching for top-level checks
   — every company may seed identically-named roles.
2. **Platform-only menus.** Menus with `IsPlatformOnly = true` MUST:
   - never be added to the default role template;
   - be filtered out of the role-access Assign UI;
   - be denied by `PermissionService` for EVERY company role;
   - be visible only to the Platform Admin.
3. **Defense in depth.** Even behind `[MenuAuthorize]`,
   admin-level controllers require the Platform check in services.
4. **Audit pinning.** Audit log pins company scope for non-Platform callers.
5. **Export/Print granularity.** `MenuAuthorizeAttribute.Permission` supports
   `"Create" | "Edit" | "Delete" | "Print" | "Export"`.
6. **Seed identity range.** New companies seed access rows from a
   safe starting Id — the table has unique indexes, so seed rows must never be
   renumbered in place.

---

# 61. SOLID Principles

The project MUST follow SOLID principles while preserving the
five-project architecture.

SOLID principles must be applied pragmatically.

Do NOT introduce unnecessary abstractions merely to demonstrate SOLID.

---

## 61.1 Single Responsibility Principle (SRP)

Every class MUST have one clear responsibility.

### Controller

Responsible for:

- HTTP requests
- Model binding
- ModelState validation
- Calling Services
- Returning Views/JSON/Redirects

Controller MUST NOT contain:

- Business logic
- Database logic
- SQL
- Stored Procedure execution
- Transaction management

---

### Service

Responsible for:

- Business/application logic
- Business validation
- Application workflows
- Mapping
- Coordinating repositories through UnitOfWork

Service MUST NOT be responsible for:

- Actual SQL execution
- Stored Procedure execution
- Direct DbContext access
- HTTP response handling
- Razor/View logic

---

### UnitOfWork

Responsible for:

- Repository coordination
- SaveChanges
- Transaction management

UnitOfWork MUST NOT contain business logic.

---

### GenericRepository

Responsible for:

- Database access
- EF Core CRUD
- Stored Procedure execution
- Database result retrieval

Repository MUST NOT contain:

- Business rules
- MVC logic
- View logic
- HTTP logic

---

## 61.2 Open/Closed Principle (OCP)

Classes should be open for extension and closed for unnecessary modification.

The Generic Repository should support new entities without requiring
a new repository implementation.

Do NOT modify GenericRepository every time a new entity is added
unless a genuinely reusable capability is required.

Similarly, GenericCrudService should support new entities without
duplicating CRUD implementations.

---

## 61.3 Liskov Substitution Principle (LSP)

Implementations MUST correctly honor their interfaces.

Do not create implementations that:

* Throw `NotSupportedException` for normal contract operations without a valid reason.
* Change the expected behavior of the interface.
* Return incompatible results.
* Introduce unexpected side effects.

---

## 61.4 Interface Segregation Principle (ISP)

Interfaces MUST remain focused.

Do NOT create large interfaces containing unrelated responsibilities.

Prefer focused interfaces over one large "God interface".

Specific interfaces should contain only operations relevant to their responsibility.

Do NOT add methods to an interface simply because another implementation might need them.

---

## 61.5 Dependency Inversion Principle (DIP)

High-level application code MUST depend on abstractions rather than
concrete implementations.

Controllers should depend on Service interfaces:

```csharp
public class StudentController : Controller
{
    private readonly IStudentService _studentService;

    public StudentController(IStudentService studentService)
    {
        _studentService = studentService;
    }
}
```

Services should depend on `IUnitOfWork` and other required interfaces.

The architecture should therefore use:

```text
Controller
    ↓
IService
    ↓
IUnitOfWork
    ↓
IGenericRepository
    ↓
Database
```

---

# 62. SOLID + Project Architecture

SOLID MUST work together with the existing architecture.

The required relationship is:

```text
MultiProduct.Web
│
├── Controllers
│       ↓
│   Service Interfaces
│
└── Views / Tabulator


MultiProduct.Application
│
├── Services
│       ↓
│   IUnitOfWork
│
└── Documents


MultiProduct.Infrastructure
│
├── UnitOfWork
│       ↓
│   IGenericRepository<TEntity>
│
├── Repositories
│       ↓
│   EF Core / Stored Procedures
│
└── Data
        ↓
    ApplicationDbContext


MultiProduct.Interfaces
│
├── Services
├── Repositories
├── UnitOfWork
├── ViewModels
└── Documents


MultiProduct.Entities
│
└── Entities
```

---

# 63. SOLID Does Not Mean More Classes

The agent MUST NOT interpret SOLID as:

```text
One entity → One Repository → One Service → One Manager → One Factory → One Helper
```

That is NOT required.

For normal CRUD, this is preferred:

```text
Controller
    ↓
IGenericCrudService<TEntity, TVM, TListVM>
    ↓
IUnitOfWork
    ↓
IGenericRepository<TEntity>
    ↓
EF Core
```

For a module with genuine business logic:

```text
Controller
    ↓
IStudentService
    ↓
IUnitOfWork
    ↓
IGenericRepository<Student>
    ↓
EF Core / Stored Procedure
```

---

# 64. SOLID and Generic Repository

The Generic Repository MUST remain generic.

Do NOT violate SRP by putting business rules into the repository.

Correct:

```text
StudentService
    ↓
Business validation
    ↓
UnitOfWork
    ↓
GenericRepository<Student>
```

The Repository only handles database access.

---

# 65. SOLID and Stored Procedures

Stored Procedure execution remains a Repository responsibility.

Do NOT create a new class for every Stored Procedure.

The Generic Repository should provide reusable Stored Procedure execution.

---

# 66. SOLID and UnitOfWork

UnitOfWork must have one clear responsibility:

```text
Repository coordination + Transaction coordination
```

It MUST NOT become a "God class".

Do NOT put business logic, SQL queries, or SP implementations inside UnitOfWork.

---

# 67. SOLID and Services

Services should contain application/business logic.

However, Services MUST NOT become God classes.

Prefer focused services when genuine business logic exists.

Standard CRUD should remain generic.

---

# 68. SOLID and Controllers

Controllers must follow SRP.

A Controller should coordinate HTTP interaction only.

The Controller does not:

```text
Validate business rules
Query database
Execute SP
Manage transactions
Map database records
```

---

# 69. SOLID and Dependency Injection

All dependencies should be injected.

Do NOT instantiate infrastructure manually:

```csharp
new GenericRepository<Student>()
new UnitOfWork()
new ApplicationDbContext()
```

inside Services or Controllers.

Concrete implementations live in MultiProduct.Infrastructure and are
registered through DI in MultiProduct.Web.

---

# 70. SOLID Review Rule for AI Agents

Before creating or modifying a class, the agent MUST ask:

1. What is this class responsible for?
2. Does it have more than one unrelated responsibility?
3. Can an existing abstraction handle this requirement?
4. Am I introducing a new abstraction unnecessarily?
5. Does this depend on an abstraction rather than a concrete implementation?
6. Am I putting business logic in a database-access class?
7. Am I putting database logic in a business class?
8. Am I creating duplication that a generic component already handles?
9. Does this code belong in MultiProduct.Web, MultiProduct.Application, MultiProduct.Infrastructure, MultiProduct.Interfaces, or MultiProduct.Entities?

If a new class is not justified, do not create it.

---

# 71. SOLID Compliance Checklist

Before completing a change, verify:

### SRP

* [ ] Controller only handles MVC/HTTP.
* [ ] Service only handles application/business logic.
* [ ] UnitOfWork only coordinates repositories/transactions.
* [ ] Repository only handles database access.

### OCP

* [ ] Existing generic components are reused.
* [ ] New entities do not require duplicated CRUD repositories.
* [ ] New functionality does not unnecessarily modify generic infrastructure.

### LSP

* [ ] Implementations correctly honor their interfaces.
* [ ] No unexpected contract violations.

### ISP

* [ ] Interfaces remain focused.
* [ ] No unnecessary methods were added.
* [ ] No God interfaces were introduced.

### DIP

* [ ] Controllers depend on Service interfaces.
* [ ] Services depend on abstractions.
* [ ] UnitOfWork exposes repository abstractions.
* [ ] Concrete implementations are registered through DI.

---

# 72. SOLID Priority Rule

SOLID principles MUST be followed, but they MUST NOT be used as a
reason to over-engineer the application.

The preferred design is:

```text
Simple + Reusable + Testable + Maintainable + SOLID
```

NOT:

```text
Maximum number of abstractions
```

When two designs both satisfy SOLID, prefer the simpler design.

---

# 73. Final SOLID Architecture

```text
                         MultiProduct.WEB
                            │
                            ▼
                       Controller
                            │
                            │ depends on
                            ▼
                       IService
                            │
                            ▼
                    MultiProduct.APPLICATION
                            │
                            ▼
                        Service
                            │
                            │ depends on
                            ▼
                      IUnitOfWork
                            │
                            ▼
                 MultiProduct.INFRASTRUCTURE
                            │
                            ▼
                       UnitOfWork
                            │
                            │ exposes
                            ▼
                  IGenericRepository<T>
                            │
                            ▼
                   GenericRepository<T>
                        │           │
                        │           │
                     EF Core       SP
                        │           │
                        └─────┬─────┘
                              ▼
                          SQL Server
```

The architecture MUST preserve:

```text
SRP → One clear responsibility
OCP → Reuse generic infrastructure
LSP → Honor contracts
ISP → Focused interfaces
DIP → Depend on abstractions
```

# END OF SOLID RULES

---

# 74. Exception Handling Rules

## 74.1 ExceptionHelper

Two identical copies exist:

```text
MultiProduct.Application/Common/ExceptionHelper.cs   → used by Services
MultiProduct.Web/Common/ExceptionHelper.cs            → used by Controllers + Middleware
```

Both provide `ExceptionHelper.BuildMessage(Exception?)` which:

* Walks the exception chain to the innermost message (the real cause).
* Returns a safe, user-facing string.
* NEVER exposes stack traces, SQL fragments, or server paths.

The Application copy is for the Service layer.
The Web copy is for Controllers and Middleware.
Do NOT merge them — they exist in separate projects that must not depend on each other.

## 74.2 GlobalExceptionMiddleware

Registered in `Program.cs` **before** all other middleware:

```text
app.UseMiddleware<GlobalExceptionMiddleware>();   // Tier 1
app.UseExceptionHandler("/Home/ServerError");     // Tier 2
app.UseStatusCodePagesWithReExecute(...);         // Tier 3
```

Behavior:

* **AJAX / JSON requests** → intercepted, logged, returns `{ "error": "..." }` with HTTP 500.
* **Non-AJAX requests** → re-throw so `UseExceptionHandler` renders the `ServerError` view.

The middleware uses `AjaxRequests.IsAjax(request)` to detect AJAX calls.

## 74.3 Three-Tier Exception Strategy

| Tier | Mechanism | Target | Behavior |
|------|-----------|--------|----------|
| 1 | `GlobalExceptionMiddleware` | AJAX/JSON (Tabulator, fetch) | Returns `{ error: "..." }` JSON, HTTP 500 |
| 2 | `UseExceptionHandler("/Home/ServerError")` | Non-AJAX | Renders ServerError view |
| 3 | `UseStatusCodePagesWithReExecute` | 404/403 etc. | Renders status-specific page |

---