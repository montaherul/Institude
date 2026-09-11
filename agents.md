# AGENTS.md

# Mighty School SaaS Development Rules

This document defines the mandatory architecture and coding rules for the Mighty School SaaS platform.

The agent MUST follow these rules when creating, modifying, refactoring, or reviewing code.

The platform is **Web API-first**: every consumer (admin SPA, public site, Flutter/PWA mobile app) talks
to the `MightySchool.Api` over REST/JSON. There is no server-rendered MVC UI in this stack.

---

# 1. Solution Structure

The solution contains exactly these five projects:

```text
MightySchool.SaaS.sln
│
├── MightySchool.Api
├── MightySchool.Application
├── MightySchool.Infrastructure
├── MightySchool.Interfaces
└── MightySchool.Entities
```

Product modules (School, Rent, Transport) are **namespaces inside the five shared projects**
(e.g. `MightySchool.Application.Modules.School`, `MightySchool.Api.Modules.School`), NOT separate projects.

Do NOT create additional projects such as:

```text
MightySchool.Domain
MightySchool.Persistence
MightySchool.Data
MightySchool.Repositories
MightySchool.Services
MightySchool.Web
MightySchool.School
MightySchool.Rent
MightySchool.Transport
```

unless explicitly requested by the developer.

---

# 2. Project Responsibilities

## MightySchool.Api

ASP.NET Core Web API presentation/protocol project.

Contains:

```text
Controllers
Program.cs
appsettings.json
wwwroot (static files served openly, e.g. public uploads)
```

Responsibilities:

* REST API Controllers (`[ApiController]`)
* JWT bearer / token authentication
* Request/response handling (REST/JSON)
* ModelState handling
* API versioning & route conventions
* Middleware (GlobalException, tenant resolution, audit pinning)
* Serving public static assets only where genuinely public

MightySchool.Api MUST NOT:

* Access DbContext directly.
* Execute Stored Procedures.
* Execute SQL.
* Contain business logic.
* Contain repository implementation.
* Contain UnitOfWork implementation.
* Render Razor/MVC views (this is an API-only project).

---

## MightySchool.Application

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

`Common` holds application-layer helpers shared by Services (e.g. `PasswordHasher.cs`, `InstituteScope.cs`).

Do NOT put domain/shared template data in MightySchool.Application/Common when it must also be
used by MightySchool.Infrastructure, because MightySchool.Application and MightySchool.Infrastructure
must never depend on each other. That data belongs in MightySchool.Entities/Common instead.

Responsibilities:

* Business/application logic
* Service implementations
* Business validation
* Application workflows
* Entity/DTO mapping
* Document generation (Excel/CSV/PDF)
* Reusable application models/results

MightySchool.Application MUST NOT:

* Access DbContext directly.
* Execute SQL.
* Execute Stored Procedures.
* Use ADO.NET directly.
* Use Dapper directly.
* Contain repository implementation.
* Contain UnitOfWork implementation.
* Contain EF Core packages.
* Reference JSON/DTO contracts of the HTTP layer (Api).

---

## MightySchool.Infrastructure

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

MightySchool.Infrastructure MUST NOT:

* Contain business logic.
* Contain HTTP/API logic.
* Contain View/UI logic.
* Depend on MightySchool.Api or MightySchool.Application.

---

## MightySchool.Interfaces

Contains interfaces/contracts only.

Contains:

```text
Services
Repositories
UnitOfWork
Dtos
Documents
```

Examples:

```text
IGenericCrudService.cs
IStudentService.cs

IGenericRepository.cs

IUnitOfWork.cs
```

MightySchool.Interfaces MUST NOT contain:

* Concrete implementations
* DbContext
* EF Core database code
* SQL execution
* Stored Procedure execution
* Business logic
* HTTP/MVC code

---

## MightySchool.Entities

Contains domain/database entities only.

Contains:

```text
Entities
Enums
Common
```

`Common` holds shared domain constant/template data that must be used by both
MightySchool.Application and MightySchool.Infrastructure (e.g. `PredefinedRoleTemplate.cs`).
These projects cannot depend on each other, so such shared data belongs here.
Do NOT put application use-case logic in MightySchool.Entities/Common.

Examples:

```text
BaseEntity.cs
Institute.cs
User.cs
Student.cs
```

MightySchool.Entities MUST NOT depend on:

```text
MightySchool.Api
MightySchool.Application
MightySchool.Infrastructure
MightySchool.Interfaces
EF Core infrastructure
HTTP/MVC
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
MightySchool.Api
   ↓
MightySchool.Application          MightySchool.Infrastructure
   ↓                                  ↓
MightySchool.Interfaces               MightySchool.Interfaces
   ↓
MightySchool.Entities
```

More specifically:

```text
MightySchool.Interfaces
    → MightySchool.Entities

MightySchool.Application
    → MightySchool.Interfaces
    → MightySchool.Entities

MightySchool.Infrastructure
    → MightySchool.Interfaces
    → MightySchool.Entities

MightySchool.Api
    → MightySchool.Application
    → MightySchool.Infrastructure
    → MightySchool.Interfaces
    → MightySchool.Entities (only when required)
```

MightySchool.Application and MightySchool.Infrastructure are siblings:

```text
MightySchool.Application
    → MightySchool.Infrastructure   FORBIDDEN

MightySchool.Infrastructure
    → MightySchool.Application      FORBIDDEN
```

They communicate through the contracts in MightySchool.Interfaces.
Dependency wiring happens only in MightySchool.Api.

Never create circular dependencies.

Forbidden:

```text
MightySchool.Entities → MightySchool.Application
MightySchool.Entities → MightySchool.Api
MightySchool.Entities → MightySchool.Infrastructure

MightySchool.Interfaces → MightySchool.Application
MightySchool.Interfaces → MightySchool.Api
MightySchool.Interfaces → MightySchool.Infrastructure

MightySchool.Application → MightySchool.Api
MightySchool.Infrastructure → MightySchool.Api
```

---

# 4. Mandatory Application Flow

The standard runtime flow is:

```text
API Controller                (MightySchool.Api)
    ↓
Service                       (MightySchool.Application)
    ↓
IUnitOfWork                   (MightySchool.Interfaces)
    ↓
UnitOfWork                    (MightySchool.Infrastructure)
    ↓
GenericRepository             (MightySchool.Infrastructure)
    ↓
EF Core / Stored Procedure    (MightySchool.Infrastructure)
    ↓
SQL Server
```

Never bypass this flow.

---

# 5. API Controller Rules

API Controllers must remain thin.

Controllers may:

* Receive HTTP requests.
* Bind DTOs.
* Validate ModelState.
* Call Services.
* Return responses (200/201/204/400/404/403…).
* Map `IResult`/`Ok`/`Created`/`BadRequest`.

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
    .ExecuteSpAsync<StudentListDto>(...);
```

The Controller must call a Service instead.

Correct:

```csharp
var result = await _studentService.GetListAsync(request);

return Ok(result);
```

---

# 6. Service Rules

The Service layer owns ALL business/application logic.

Services are responsible for:

* Business rules
* Application workflows
* Business validation
* Entity/DTO mapping
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
    .ExecuteSpAsync<StudentListDto>(
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
MightySchool.Infrastructure
└── UnitOfWork
    └── UnitOfWork.cs

MightySchool.Interfaces
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
* HTTP logic

UnitOfWork coordinates data access.

---

# 8. Generic Repository Rules

There is ONE Generic Repository.

Structure:

```text
MightySchool.Interfaces
└── Repositories
    └── IGenericRepository.cs

MightySchool.Infrastructure
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

IGenericCrudService<TEntity, TDto, TListDto>
GenericCrudService<TEntity, TDto, TListDto>
```

## 9.1 Generic CRUD Service (the ONE generic service)

`GenericCrudService<TEntity, TDto, TListDto>`
(`MightySchool.Application/Services/GenericCrudService.cs`) is the platform's
single reusable generic CRUD implementation. It powers ALL simple
CRUD/master-data modules (School classes/subjects, Rent properties/units,
Transport vehicles/routes).

Contract (`MightySchool.Interfaces/Services/IGenericCrudService.cs`):

```text
PagedResult<TListDto> GetListAsync(PagedRequest)           // SP-backed paged listing
TListDto? GetByIdAsync(int id)                             // scoped
List<MasterDataDropdownDto> GetActiveAsync(int? instituteId = null)
(bool Success, string Message) CreateAsync(TDto model, int? createdBy = null)
(bool Success, string Message) UpdateAsync(TDto model, int? updatedBy = null)
(bool Success, string Message) DeleteAsync(int id, int? updatedBy = null)  // soft delete
```

Generic constraints:

```text
TEntity : BaseEntity, IMasterDataEntity, new()
TDto    : class, IMasterDataDto, new()
TListDto : class, IHasTotalCount, new()
```

The implementation already provides:

* Institute-scoped filtering (e.g. InstituteScope)
* Duplicate code/name validation per Institute
* Optional per-module validator delegate
* Soft delete (`IsActive = false`, `IsDeleted = true`)
* Audit log records
* `IsDefault` single-default-per-institute handling
* Entity/DTO/list-DTO mapping

Per-module configuration is supplied through DI in `MightySchool.Api/Program.cs`
(`RegisterGenericCrudService<TEntity, TDto, TListDto>`): entity type, DTO types,
list SP name, display name, optional validator factory. NEVER create a separate
service class per simple CRUD module.

Forbidden names (removed/obsolete, do not reintroduce):

```text
MasterDataService
GenericService
InstituteService
```

Simple CRUD modules MUST flow through:

```text
Simple CRUD module
    ↓
IGenericCrudService<TEntity, TDto, TListDto>
    ↓
GenericCrudService<TEntity, TDto, TListDto>
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
MightySchool.Infrastructure
```

The DbContext belongs in:

```text
MightySchool.Infrastructure
└── Data
    └── ApplicationDbContext.cs
```

Entity configurations belong in:

```text
MightySchool.Infrastructure
└── Configurations
```

Migrations belong in:

```text
MightySchool.Infrastructure
└── Migrations
```

---

# 14. DbContext Rules

Only MightySchool.Infrastructure (Repository/UnitOfWork infrastructure) may use DbContext.

Do NOT inject:

```text
ApplicationDbContext
```

into:

```text
Controllers
Services
Dtos
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
* Institute restrictions
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

# 17. Data Grid Rules (Tabulator-Class Clients)

Grids live on the **client** (admin SPA/web front using Tabulator or a similar grid).
The API serves server-side pagination/filtering/sort via SP-backed listing endpoints.

Required flow:

```text
Tabulator
    ↓ AJAX (JSON)
API Controller
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
JSON response
    ↓
Tabulator
```

The grid MUST NOT communicate directly with the database.

---

# 18. Listing Endpoint Rules

Listing endpoints must remain thin.

Example:

```csharp
[HttpGet]
public async Task<IActionResult> GetList([FromQuery] StudentSearchRequest model)
{
    var result =
        await _studentService.GetListAsync(model);

    return Ok(result);
}
```

The Controller must not:

* Execute SQL
* Execute Stored Procedures
* Query DbContext
* Perform database pagination/filtering/sorting

---

# 19. Server-Side Processing

For large tables, use server-side:

* Pagination
* Filtering
* Sorting
* Searching

Prefer Stored Procedures for complex server-side operations.

Do not load thousands of records into memory just to let the client paginate them.

---

# 20. DTO Rules (ViewModels / Contracts)

DTOs belong in MightySchool.Interfaces.

Use DTOs for:

* Request/input contracts
* Search/filter input
* Grid rows
* Report results
* API-specific data

Do not expose database entities directly to the API when a DTO is appropriate.

Example:

```csharp
public class StudentListDto
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
MightySchool.Entities
```

Entities must remain independent of:

* HTTP/MVC
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
Dto
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
Dto
```

Do not duplicate identical mapping logic across Controllers.

---

# 23. Validation Rules

Basic input validation:

```text
Dto
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
    → Dto validation

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
* Converted to a suitable API response where required (`ProblemDetails`).

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
MightySchool.Api
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

StudentDto
StudentListDto
StudentSearchDto

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
                 MightySchool.Api (REST/JSON)
                       │
                       ▼
                  Controller
                       │
                       ▼
              MightySchool.Application
                   Service
                       │
                       ▼
              MightySchool.Interfaces
                   IUnitOfWork
                       │
                       ▼
             MightySchool.Infrastructure
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
MightySchool.Api
    = REST/JSON HTTP, auth, middleware

MightySchool.Application
    = Business Logic + Services + Document Generation

MightySchool.Infrastructure
    = Data Access + EF Core + Stored Procedure Execution

MightySchool.Interfaces
    = Contracts + DTOs

MightySchool.Entities
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
10. **The platform is Web API-first; JSON responses define the contract.**
11. **Grids communicate through API AJAX endpoints.**
12. **Do not access DbContext directly from Controllers or Services.**
13. **Do not create duplicate CRUD repositories/services.**
14. **Specific Services are created only for genuine business logic.**
15. **Specific Repositories are created only for genuine database-access requirements.**
16. **Do not create additional projects without explicit approval.**
17. **Do not bypass architectural layers.**
18. **Do not over-engineer.**
19. **Prefer existing generic abstractions over new duplicated abstractions.**
20. **Preserve this architecture when implementing new modules.**
21. **Repositories, UnitOfWork, DbContext, Configurations, Migrations, and EF packages belong in MightySchool.Infrastructure.**
22. **Services, business logic, and document generation belong in MightySchool.Application.**
23. **MightySchool.Application and MightySchool.Infrastructure must never depend on each other; they communicate only through MightySchool.Interfaces.**
24. **Modules (School, Rent, Transport) share the five projects as namespaces; they integrate only through services/event bus — never direct cross-module DB coupling.**

---

# 37. New Module Rule

When adding a new module, follow this order:

```text
1.  Add Entity (MightySchool.Entities)
2.  Add DTO(s) (MightySchool.Interfaces)
3.  Determine whether generic CRUD is sufficient
4.  Use GenericCrudService for standard CRUD (MightySchool.Application)
5.  Create Specific Service only if business logic requires it (MightySchool.Application)
6.  Use Generic Repository for database access (MightySchool.Infrastructure)
7.  Use EF Core for normal CRUD (MightySchool.Infrastructure)
8.  Add Stored Procedure only for complex operations
9.  Execute the SP through Generic Repository (MightySchool.Infrastructure)
10. Add API Controller (MightySchool.Api) — standard REST, no MVC views
11. Add Stored Procedures under Database/StoredProcedures/<Module>
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
Is it HTTP/JSON contract?
    → MightySchool.Api / Controller / DTO binding

Is it business logic?
    → MightySchool.Application / Service

Is it a contract?
    → MightySchool.Interfaces

Is it an entity?
    → MightySchool.Entities

Is it database execution?
    → MightySchool.Infrastructure / Repository

Is it transaction/repository coordination?
    → MightySchool.Infrastructure / UnitOfWork

Is it DbContext / Configurations / Migrations?
    → MightySchool.Infrastructure / Data

Is it normal CRUD?
    → Generic Repository + EF Core (MightySchool.Infrastructure)

Is it complex listing/report/filter/aggregation?
    → Generic Repository + Stored Procedure (MightySchool.Infrastructure)

Is it document generation (Excel/CSV/PDF)?
    → MightySchool.Application / Documents

Is it a grid/table UI?
    → client (Tabulator), consuming the API

Is it UI permission/button hiding?
    → client UI, resolved from the API permission contract (PageAccessDto,
      IPermissionService.GetAccessAsync); logic in MightySchool.Application (PermissionService)

Is it a cross-module integration (e.g. School → Transport)?
    → Service-based orchestration or async event; NEVER a direct module DB query
```

---

# 39. Permission-Based Conditional API / UI Rules

Server-side authorization is the backstop; the API MUST also enforce fine-grained
access per action (belt-and-suspenders approach). The client hides actions via the
permission contract the API returns.

## 39.1 When to Apply

Every endpoint that creates/edits/deletes/prints/exports resources MUST be guarded.
The client MUST hide the actions the current user cannot perform.

## 39.2 Contract: PageAccessDto (MightySchool.Interfaces)

`PageAccessDto` (`MightySchool.Interfaces/Dtos/PageAccessDto.cs`) carries the
per-role flags for one controller/page:

```csharp
public class PageAccessDto
{
    public bool CanView { get; set; }
    public bool CanCreate { get; set; }
    public bool CanEdit { get; set; }
    public bool CanDelete { get; set; }
    public bool CanPrint { get; set; }
    public bool CanExport { get; set; }

    public static PageAccessDto FullAccess => new() { ... };
}
```

## 39.3 Service Contract: GetAccessAsync

`IPermissionService.GetAccessAsync(int userRoleId, string controller)` returns
`Task<PageAccessDto?>`.

`PermissionService.GetAccessAsync` (MightySchool.Application/Services):

* Returns `PageAccessDto.FullAccess` for the Platform Admin role.
* Otherwise maps the role's `RoleWiseMenuAccess` row for the controller.
* Returns `null` when no role or access row exists.

## 39.4 Authorization

* Coarse gate: `[Authorize]` + a platform/module policy or `MenuAuthorizeAttribute`
  (declarative permission such as `Permission = "Edit"`).
* Fine gate in services: business/permission checks before mutation.
* The client resolves the `GetAccessAsync` contract to hide buttons.

## 39.5 Rules

1. Hiding a button on the client MUST NOT be the only protection — the API must
   still deny direct calls; return 403 when denied.
2. Do NOT gate read/list endpoints behind anything other than `CanView`.
3. Permission resolution logic lives in `PermissionService` (Application) only.

---

# 40. Multi-Level RBAC Hierarchy

Access control has 2 distinct, non-interchangeable levels:

```text
LEVEL 1 — Platform (global, multi-tenant)
    Platform Admin = the single global UserRole (Scope = Platform,
    detected via Users.PlatformRoleId)
      ├── Institute & Tenant CRUD
      ├── User & Role Management
      ├── Plan & Module Management (School/Rent/Transport entitlements)
      └── System Settings & Season Checks

LEVEL 2 — Institute (per-tenant)
    Institute Admin (implicit full access within its own institute only)
    └── 8 operational roles
          → governed by RoleWiseMenuAccess rows seeded per institute
```

## 40.1 Mandatory Rules

1. **Platform detection is role-only.** The global **Platform Admin role**
   is the RBAC source of truth. Platform scope is detected via the
   `RoleScope` claim. NEVER use role name matching for top-level checks
   — every institute may seed identically-named roles.
2. **Platform-only menus.** Menus with `IsPlatformOnly = true` MUST:
   - never be added to the default role template;
   - be filtered out of the role-access Assign UI;
   - be denied by `PermissionService` for EVERY institute role;
   - be visible only to the Platform Admin.
3. **Defense in depth.** Even behind the authorization attribute,
   admin-level controllers require the Platform check in services.
4. **Audit pinning.** Audit log pins institute scope for non-Platform callers.
5. **Permission granularity.** The permission attribute supports
   `"Create" | "Edit" | "Delete" | "Print" | "Export"`.
6. **Entitlement gating.** Product modules (School/Rent/Transport) are live for a
   tenant only when the tenant's subscription/entitlement includes them.
7. **Seed identity range.** New institutes seed access rows from a
   safe starting Id — the table has unique indexes, so seed rows must never be
   renumbered in place.

---

# 50. Authentication (Web API)

* Token-based auth: **JWT bearer** for API clients; refresh-token rotation supported.
* Cookie auth may be used only for the same-origin admin/dashboard client if adopted.
* All protected endpoints require a valid token; identity claims include
  `UserId`, `RoleId`, `RoleScope`, `InstituteId` (when institute-scoped).
* Passwords hashed with `PasswordHasher` (PBKDF2/Argon2) in Application/Common.
* API keys/secrets belong in `appsettings.Development.json` or user-secrets, never in code.

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
- Model binding/ModelState
- Calling Services
- Returning results/status codes

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
public class StudentController : ControllerBase
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
MightySchool.Api
│
├── Controllers
│       ↓
│   Service Interfaces
│
└── Middleware / Auth


MightySchool.Application
│
├── Services
│       ↓
│   IUnitOfWork
│
└── Documents


MightySchool.Infrastructure
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


MightySchool.Interfaces
│
├── Services
├── Repositories
├── UnitOfWork
├── Dtos
└── Documents


MightySchool.Entities
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
IGenericCrudService<TEntity, TDto, TListDto>
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

Concrete implementations live in MightySchool.Infrastructure and are
registered through DI in MightySchool.Api.

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
9. Does this code belong in MightySchool.Api, MightySchool.Application, MightySchool.Infrastructure, MightySchool.Interfaces, or MightySchool.Entities?

If a new class is not justified, do not create it.

---

# 71. SOLID Compliance Checklist

Before completing a change, verify:

### SRP

* [ ] Controller only handles HTTP/JSON.
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
                         MightySchool.API
                            │
                            ▼
                       Controller
                            │
                            │ depends on
                            ▼
                       IService
                            │
                            ▼
                    MightySchool.APPLICATION
                            │
                            ▼
                        Service
                            │
                            │ depends on
                            ▼
                      IUnitOfWork
                            │
                            ▼
                 MightySchool.INFRASTRUCTURE
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
MightySchool.Application/Common/ExceptionHelper.cs   → used by Services
MightySchool.Api/Common/ExceptionHelper.cs            → used by Controllers + Middleware
```

Both provide `ExceptionHelper.BuildMessage(Exception?)` which:

* Walks the exception chain to the innermost message (the real cause).
* Returns a safe, user-facing string.
* NEVER exposes stack traces, SQL fragments, or server paths.

The Application copy is for the Service layer.
The Api copy is for Controllers and Middleware.
Do NOT merge them — they exist in separate projects that must not depend on each other.

## 74.2 GlobalExceptionMiddleware

Registered in `Program.cs` **before** all other middleware:

```text
app.UseMiddleware<GlobalExceptionMiddleware>();
app.UseExceptionHandler(...);          // produces ProblemDetails
app.UseStatusCodePages();
```

Behavior:

* **API requests** → intercepted, logged, returns `ProblemDetails` (`{ "title", "status" }`) with HTTP 500.
* The middleware logs the exception (without secrets) and converts to a safe response.

## 74.3 Exception Strategy

| Tier | Mechanism | Target | Behavior |
|------|-----------|--------|----------|
| 1 | `GlobalExceptionMiddleware` | All API requests | ProblemDetails JSON, HTTP 500 |
| 2 | Authorization failure (policies/attribute) | Secured endpoints | HTTP 401/403 |
| 3 | ModelState / validation | Request body/query | HTTP 400 with problem details |

---