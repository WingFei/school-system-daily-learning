# New School Management System — Phase 1 Foundation

**Added:** 8 October 2026  
**Status:** Architecture and implementation tutorial; sample code has not been compiled or deployed.  
**Target:** Singapore schools, tuition centres, private education institutions and training providers.

## Technology baseline

| Layer | Technology |
| --- | --- |
| Language | C# |
| Runtime | .NET 10 LTS / ASP.NET Core |
| UI model | Blazor Web App with Interactive Server render mode |
| Components | DevExpress Blazor |
| Persistence | Entity Framework Core |
| Database | MariaDB |
| EF Core MariaDB provider | Pomelo.EntityFrameworkCore.MySql |
| Authentication (next phase) | ASP.NET Core Identity and authorization policies |
| Deployment (planned) | IIS or Linux |
| Source control | GitHub |

**Version compatibility:** Verify the latest DevExpress / .NET / EF Core / Pomelo compatibility matrix before installation. An initial compatible combination is .NET 10, EF Core 9 packages and Pomelo 9, with an appropriate DevExpress Blazor version supporting .NET 10. Do not install EF Core 10 alongside Pomelo 9 expecting compatibility. The version numbers below are examples, not evidence of an executed build.

## Architectural decisions

Start as a **modular monolith**. Keep UI, business logic, entities and persistence separate; avoid putting EF Core queries directly into Razor components. Design future tenant isolation and per-school access controls *before* deploying real student data.

```text
SchoolManagement.sln
├── SchoolManagement.Web              # Blazor, DevExpress UI, Program.cs
├── SchoolManagement.Application      # DTOs, service contracts, use cases
├── SchoolManagement.Domain           # Entities, domain rules
├── SchoolManagement.Infrastructure   # EF Core, MariaDB, services
└── SchoolManagement.Tests            # Unit/integration tests (recommended)
```

References:
- Web → Application + Infrastructure
- Application → Domain
- Infrastructure → Application + Domain
- Domain → none

## 1. Install prerequisites

Install a Visual Studio release that supports .NET 10, .NET 10 SDK, MariaDB 11.4 LTS, and licensed DevExpress Blazor packages. Choose the **ASP.NET and web development** workload.

Verify:

```powershell
dotnet --list-sdks
dotnet --version
```

## 2. Create the solution

Create **Blazor Web App** named `SchoolManagement.Web`, solution `SchoolManagement`, target `.NET 10`, render mode **Server**, interactivity **Global**, HTTPS enabled, Authentication **None** for local prototype only.

Add three .NET class libraries named `SchoolManagement.Domain`, `SchoolManagement.Application`, and `SchoolManagement.Infrastructure`. Configure project references as above.

## 3. Install NuGet packages

From the solution folder:

```powershell
dotnet add SchoolManagement.Infrastructure package Pomelo.EntityFrameworkCore.MySql --version 9.0.0
dotnet add SchoolManagement.Web package Microsoft.EntityFrameworkCore.Design --version 9.0.10
dotnet tool install --global dotnet-ef --version 9.0.10
```

Use a consistent supported EF Core 9 patch version. If the EF tool already exists, use `dotnet tool update --global dotnet-ef --version 9.0.10`. Register the authenticated DevExpress NuGet source through official account instructions, then install `DevExpress.Blazor` in Web. Never commit a private feed key.

## 4. Create local MariaDB database

```sql
CREATE DATABASE schoolmanagementdb
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'sms_dev'@'localhost'
  IDENTIFIED BY 'REPLACE_WITH_STRONG_PASSWORD';

GRANT ALL PRIVILEGES ON schoolmanagementdb.*
  TO 'sms_dev'@'localhost';
```

This development-only user can run migrations; create restricted separate accounts for production.

Right-click Web → **Manage User Secrets**, then configure:

```json
{
  "ConnectionStrings": {
    "SchoolManagement": "Server=localhost;Port=3306;Database=schoolmanagementdb;User=sms_dev;Password=YOUR_LOCAL_PASSWORD;"
  }
}
```

## 5. Domain student entity

`SchoolManagement.Domain/Entities/Student.cs`

```csharp
namespace SchoolManagement.Domain.Entities;

public class Student
{
    public long StudentId { get; set; }
    public string StudentNo { get; set; } = string.Empty;
    public string FullName { get; set; } = string.Empty;
    public string? Email { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAtUtc { get; set; } = DateTime.UtcNow;
}
```

This is a **prototype, single-institution** entity. Add SchoolId, branch rules, guardian relationships, audit records and appropriate data retention policy before production.

## 6. EF Core DbContext

`SchoolManagement.Infrastructure/Data/SchoolDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SchoolManagement.Domain.Entities;

namespace SchoolManagement.Infrastructure.Data;

public class SchoolDbContext(DbContextOptions<SchoolDbContext> options)
    : DbContext(options)
{
    public DbSet<Student> Students => Set<Student>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Student>(entity =>
        {
            entity.ToTable("Students");
            entity.HasKey(x => x.StudentId);
            entity.Property(x => x.StudentNo).HasMaxLength(30).IsRequired();
            entity.HasIndex(x => x.StudentNo).IsUnique();
            entity.Property(x => x.FullName).HasMaxLength(200).IsRequired();
            entity.Property(x => x.Email).HasMaxLength(255);
            entity.Property(x => x.DateOfBirth).HasColumnType("date");
            entity.Property(x => x.CreatedAtUtc).HasColumnType("datetime(6)");
        });
    }
}
```

## 7. Application DTO and contract

`SchoolManagement.Application/DTOs/StudentDto.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace SchoolManagement.Application.DTOs;

public class StudentDto
{
    public long StudentId { get; set; }

    [Required, StringLength(30)]
    public string StudentNo { get; set; } = "";

    [Required, StringLength(200)]
    public string FullName { get; set; } = "";

    [EmailAddress, StringLength(255)]
    public string? Email { get; set; }

    public DateTime? DateOfBirth { get; set; }
    public bool IsActive { get; set; } = true;
}
```

`SchoolManagement.Application/Interfaces/IStudentService.cs`

```csharp
using SchoolManagement.Application.DTOs;

namespace SchoolManagement.Application.Interfaces;

public interface IStudentService
{
    Task<List<StudentDto>> GetAllAsync();
    Task<StudentDto?> GetByIdAsync(long id);
    Task<long> CreateAsync(StudentDto dto);
    Task UpdateAsync(StudentDto dto);
    Task DeleteAsync(long id);
}
```

## 8. Student service implementation

`SchoolManagement.Infrastructure/Services/StudentService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SchoolManagement.Application.DTOs;
using SchoolManagement.Application.Interfaces;
using SchoolManagement.Domain.Entities;
using SchoolManagement.Infrastructure.Data;

namespace SchoolManagement.Infrastructure.Services;

public class StudentService(IDbContextFactory<SchoolDbContext> factory)
    : IStudentService
{
    public async Task<List<StudentDto>> GetAllAsync()
    {
        await using var db = await factory.CreateDbContextAsync();
        return await db.Students.AsNoTracking()
            .OrderBy(x => x.FullName)
            .Select(x => new StudentDto {
                StudentId = x.StudentId,
                StudentNo = x.StudentNo,
                FullName = x.FullName,
                Email = x.Email,
                DateOfBirth = x.DateOfBirth,
                IsActive = x.IsActive
            })
            .ToListAsync();
    }

    public async Task<StudentDto?> GetByIdAsync(long id)
    {
        await using var db = await factory.CreateDbContextAsync();
        return await db.Students.AsNoTracking()
            .Where(x => x.StudentId == id)
            .Select(x => new StudentDto {
                StudentId = x.StudentId,
                StudentNo = x.StudentNo,
                FullName = x.FullName,
                Email = x.Email,
                DateOfBirth = x.DateOfBirth,
                IsActive = x.IsActive
            })
            .FirstOrDefaultAsync();
    }

    public async Task<long> CreateAsync(StudentDto dto)
    {
        await using var db = await factory.CreateDbContextAsync();
        var student = new Student {
            StudentNo = dto.StudentNo.Trim(),
            FullName = dto.FullName.Trim(),
            Email = dto.Email?.Trim(),
            DateOfBirth = dto.DateOfBirth,
            IsActive = dto.IsActive
        };
        db.Students.Add(student);
        await db.SaveChangesAsync();
        return student.StudentId;
    }

    public async Task UpdateAsync(StudentDto dto)
    {
        await using var db = await factory.CreateDbContextAsync();
        var student = await db.Students.FindAsync(dto.StudentId)
            ?? throw new KeyNotFoundException("Student not found.");
        student.StudentNo = dto.StudentNo.Trim();
        student.FullName = dto.FullName.Trim();
        student.Email = dto.Email?.Trim();
        student.DateOfBirth = dto.DateOfBirth;
        student.IsActive = dto.IsActive;
        await db.SaveChangesAsync();
    }

    public async Task DeleteAsync(long id)
    {
        await using var db = await factory.CreateDbContextAsync();
        var student = await db.Students.FindAsync(id);
        if (student is null) return;
        db.Students.Remove(student);
        await db.SaveChangesAsync();
    }
}
```

**Blazor Interactive Server requirement:** Use `IDbContextFactory<SchoolDbContext>` and a new context per operation rather than sharing one across long-lived UI circuits. Hard deletes are for this disposable prototype only.

## 9. Dependency injection in Web

In `SchoolManagement.Web/Program.cs`, preserve template routing and middleware and register:

```csharp
using DevExpress.Blazor;
using Microsoft.EntityFrameworkCore;
using SchoolManagement.Application.Interfaces;
using SchoolManagement.Infrastructure.Data;
using SchoolManagement.Infrastructure.Services;

builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();
builder.Services.AddDevExpressBlazor();

var connectionString = builder.Configuration
    .GetConnectionString("SchoolManagement")
    ?? throw new InvalidOperationException("Missing SchoolManagement connection string.");

builder.Services.AddDbContextFactory<SchoolDbContext>(options =>
    options.UseMySql(connectionString,
        new MariaDbServerVersion(new Version(11, 4, 0))));

builder.Services.AddScoped<IStudentService, StudentService>();
```

These registrations belong **before** `var app = builder.Build();`. Keep `app.UseAntiforgery();`, `app.MapStaticAssets();`, and `app.MapRazorComponents<App>().AddInteractiveServerRenderMode();` as appropriate for the generated .NET 10 Blazor template.

## 10. DevExpress UI setup

Add `@using DevExpress.Blazor` to `Components/_Imports.razor` and register the DevExpress theme and scripts in the root `Components/App.razor` using the DevExpress 26.1 documentation (for example, `DxResourceManager.RegisterTheme(Themes.Fluent)` and `DxResourceManager.RegisterScripts()`). Retain the template's Blazor Web script and Interactive Server render mode.

Build `Components/Pages/Students.razor` using a `DxGrid` with:
- `Data` bound to `IStudentService.GetAllAsync()`
- `KeyFieldName="StudentId"`
- `DxGridDataColumn` for StudentNo, FullName, Email, IsActive
- `ShowSearchBox="true"` and `ShowFilterRow="true"`
- `EditForm` + `DataAnnotationsValidator` with DevExpress `DxTextBox`, `DxDateEdit`, `DxCheckBox`, `DxButton`
- Service-based create/update/delete, and refresh the grid after writes

Add a `/students` link in `Components/Layout/NavMenu.razor`.

**Security note:** Never publish an unauthenticated student CRUD page. Add authorization, tenant filtering, server-side validation and a delete confirmation first.

## 11. Migrations

From solution root:

```powershell
dotnet ef migrations add InitialCreate --project SchoolManagement.Infrastructure --startup-project SchoolManagement.Web --context SchoolDbContext
dotnet ef database update --project SchoolManagement.Infrastructure --startup-project SchoolManagement.Web --context SchoolDbContext
```

Inspect MariaDB:

```sql
USE schoolmanagementdb;
SHOW TABLES;
DESCRIBE Students;
SELECT * FROM Students ORDER BY StudentId DESC;
```

Future schema changes should use reviewed EF Core migrations, not manual production table edits.

## 12. Local verification checklist

- [ ] Solution builds
- [ ] Blazor application opens on HTTPS
- [ ] DevExpress controls render and respond
- [ ] Database connection succeeds
- [ ] EF Core creates Students and __EFMigrationsHistory
- [ ] Student create/list/edit/delete works
- [ ] Grid search/filter works
- [ ] Duplicate StudentNo produces a handled error
- [ ] Secrets are not committed
- [ ] No unauthenticated deployment with real personal data

## Next phases

**Phase 2:** ASP.NET Core Identity, authorization policies, multi-tenant schools and branches, audit logging, and database-enforced access isolation.

**Phase 3:** Reusable DevExpress shell, sidebar, responsive forms, validation patterns, loading states and standardized grids.

**Phase 4:** Enquiries, admissions, guardian records, student lifecycle, course/intake enrolment and attendance.

## Primary documentation

- [.NET downloads](https://dotnet.microsoft.com/en-us/download)
- [Blazor render modes](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/render-modes)
- [EF Core DbContext lifetime](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [EF Core migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Pomelo MariaDB/MySQL provider](https://github.com/PomeloFoundation/Pomelo.EntityFrameworkCore.MySql)
- [DevExpress Blazor documentation](https://docs.devexpress.com/Blazor/)
- [MariaDB documentation](https://mariadb.com/docs/)

This document records the selected direction and Phase 1 implementation steps. It does **not** claim that a runnable solution has been committed to the repository.
