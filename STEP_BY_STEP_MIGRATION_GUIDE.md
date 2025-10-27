# Step-by-Step Migration Guide: .NET Framework 4.8 to .NET 8

This guide provides detailed, actionable steps to migrate the ContosoUniversity application from .NET Framework 4.8 to .NET 8.0.

---

## Table of Contents

1. [Pre-Migration Phase (Week 0-1)](#phase-0-pre-migration)
2. [Testing Foundation (Week 1-2)](#phase-1-testing-foundation)
3. [Framework Migration (Week 3-4)](#phase-2-framework-migration)
4. [Azure Integration (Week 5-6)](#phase-3-azure-integration)
5. [Deployment & Testing (Week 7-8)](#phase-4-deployment--testing)

---

## Phase 0: Pre-Migration (Week 0-1)

### Step 0.1: Environment Setup

**Time Estimate:** 2-4 hours

1. **Install Required Tools**

   ```powershell
   # Install .NET 8 SDK
   winget install Microsoft.DotNet.SDK.8
   
   # Verify installation
   dotnet --version
   # Should show 8.0.x
   
   # Install .NET Upgrade Assistant
   dotnet tool install -g upgrade-assistant
   
   # Install try-convert (optional)
   dotnet tool install -g try-convert
   
   # Install Entity Framework Core tools
   dotnet tool install -g dotnet-ef
   ```

2. **Set Up Azure Account**

   ```bash
   # Install Azure CLI
   winget install Microsoft.AzureCLI
   
   # Login to Azure
   az login
   
   # Set default subscription
   az account set --subscription "Your-Subscription-Name"
   
   # Create resource group
   az group create --name contoso-university-rg --location eastus
   ```

3. **Create Git Branch for Migration**

   ```bash
   cd ContosoUniversity
   git checkout -b feature/migrate-to-net8
   git push -u origin feature/migrate-to-net8
   ```

### Step 0.2: Backup Current State

**Time Estimate:** 1 hour

1. **Backup Database**

   ```powershell
   # Export current database
   $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
   $backupPath = ".\Backups\ContosoUniversity_$timestamp.bak"
   
   # Create backup directory
   New-Item -Path ".\Backups" -ItemType Directory -Force
   
   # Backup LocalDB database
   sqlcmd -S "(LocalDb)\MSSQLLocalDB" -Q "BACKUP DATABASE ContosoUniversityNoAuthEFCore TO DISK = '$backupPath'"
   ```

2. **Document Current Configuration**

   ```powershell
   # Copy current Web.config
   Copy-Item Web.config ".\Backups\Web.config.backup_$timestamp"
   
   # Export NuGet packages list
   nuget list -Source packages.config > ".\Backups\packages_before_migration_$timestamp.txt"
   ```

3. **Create Migration Log**

   Create a file `MIGRATION_LOG.md` to track progress:

   ```markdown
   # Migration Log
   
   ## Pre-Migration Checklist
   - [ ] Environment setup complete
   - [ ] Database backup created
   - [ ] Configuration backed up
   - [ ] Git branch created
   
   ## Issues Encountered
   
   ## Decisions Made
   ```

### Step 0.3: Analyze Current Dependencies

**Time Estimate:** 2 hours

1. **Run .NET Portability Analyzer**

   ```powershell
   # Install API Port tool
   dotnet tool install -g Microsoft.DotNet.ApiPort.Tool
   
   # Analyze the assembly
   ApiPort.exe analyze -f ContosoUniversity.dll -t ".NET Core 8.0" -r HTML -o PortabilityReport.html
   
   # Review the report
   start PortabilityReport.html
   ```

2. **Document API Compatibility Issues**

   Create `API_COMPATIBILITY_ISSUES.md` and list any incompatible APIs found.

### Step 0.4: Upgrade Entity Framework Core

**Time Estimate:** 4-6 hours

**⚠️ CRITICAL: Do this BEFORE migrating to .NET 8**

1. **Update packages.config**

   Edit `packages.config`:

   ```xml
   <!-- Change from: -->
   <package id="Microsoft.EntityFrameworkCore" version="3.1.32" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.Abstractions" version="3.1.32" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.SqlServer" version="3.1.32" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.Relational" version="3.1.32" targetFramework="net482" />
   
   <!-- Change to: -->
   <package id="Microsoft.EntityFrameworkCore" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.Abstractions" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.SqlServer" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.EntityFrameworkCore.Relational" version="8.0.0" targetFramework="net482" />
   ```

2. **Update Microsoft.Data.SqlClient**

   ```xml
   <!-- Change from: -->
   <package id="Microsoft.Data.SqlClient" version="2.1.4" targetFramework="net482" />
   
   <!-- Change to: -->
   <package id="Microsoft.Data.SqlClient" version="5.1.2" targetFramework="net482" />
   ```

3. **Update Extension Packages**

   ```xml
   <!-- Update all Microsoft.Extensions.* packages from 3.1.32 to 8.0.0 -->
   <package id="Microsoft.Extensions.Caching.Abstractions" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.Extensions.Caching.Memory" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.Extensions.Configuration" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.Extensions.DependencyInjection" version="8.0.0" targetFramework="net482" />
   <package id="Microsoft.Extensions.Logging" version="8.0.0" targetFramework="net482" />
   <!-- etc. -->
   ```

4. **Restore and Build**

   ```powershell
   # Restore packages
   nuget restore ContosoUniversity.sln
   
   # Build
   msbuild ContosoUniversity.sln /p:Configuration=Release
   
   # Check for breaking changes
   # Review build warnings and errors
   ```

5. **Update DbContext Configuration**

   No major changes needed for EF Core 3.1 → 8.0, but review:
   - Date/time handling (already configured for datetime2)
   - Null reference warnings (enable nullable reference types later)

6. **Test Thoroughly**

   ```powershell
   # Run the application
   # Test all CRUD operations
   # Verify database operations work
   # Check for any runtime errors
   ```

7. **Commit Changes**

   ```bash
   git add .
   git commit -m "chore: Upgrade Entity Framework Core from 3.1.32 to 8.0.0"
   git push
   ```

---

## Phase 1: Testing Foundation (Week 1-2)

### Step 1.1: Create Test Project Structure

**Time Estimate:** 2 hours

1. **Create Unit Test Project**

   ```powershell
   # Navigate to solution directory
   cd ContosoUniversity
   
   # Create test project
   dotnet new xunit -n ContosoUniversity.UnitTests -f net48
   
   # Add to solution
   dotnet sln ContosoUniversity.sln add ContosoUniversity.UnitTests/ContosoUniversity.UnitTests.csproj
   
   # Add project reference
   cd ContosoUniversity.UnitTests
   dotnet add reference ../ContosoUniversity.csproj
   ```

2. **Install Testing Packages**

   ```powershell
   # Install Moq for mocking
   dotnet add package Moq --version 4.20.69
   
   # Install FluentAssertions (optional, but recommended)
   dotnet add package FluentAssertions --version 6.12.0
   
   # Install EF Core InMemory for database testing
   dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 8.0.0
   
   # Install test utilities
   dotnet add package Microsoft.NET.Test.Sdk --version 17.8.0
   ```

3. **Create Test Directory Structure**

   ```powershell
   New-Item -Path "Controllers" -ItemType Directory
   New-Item -Path "Services" -ItemType Directory
   New-Item -Path "Data" -ItemType Directory
   New-Item -Path "TestUtilities" -ItemType Directory
   ```

### Step 1.2: Write Controller Tests

**Time Estimate:** 16-20 hours

1. **Create Test Utilities**

   Create `TestUtilities/TestDbContextFactory.cs`:

   ```csharp
   using ContosoUniversity.Data;
   using Microsoft.EntityFrameworkCore;
   using System;
   
   namespace ContosoUniversity.UnitTests.TestUtilities
   {
       public class TestDbContextFactory
       {
           public static SchoolContext CreateInMemoryContext()
           {
               var options = new DbContextOptionsBuilder<SchoolContext>()
                   .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
                   .Options;
               
               return new SchoolContext(options);
           }
       }
   }
   ```

2. **Create StudentsController Tests**

   Create `Controllers/StudentsControllerTests.cs`:

   ```csharp
   using ContosoUniversity.Controllers;
   using ContosoUniversity.Data;
   using ContosoUniversity.Models;
   using ContosoUniversity.UnitTests.TestUtilities;
   using Microsoft.AspNetCore.Mvc;
   using System;
   using System.Linq;
   using System.Threading.Tasks;
   using Xunit;
   
   namespace ContosoUniversity.UnitTests.Controllers
   {
       public class StudentsControllerTests
       {
           [Fact]
           public async Task Index_ReturnsViewWithStudents()
           {
               // Arrange
               using var context = TestDbContextFactory.CreateInMemoryContext();
               context.Students.Add(new Student 
               { 
                   FirstMidName = "John", 
                   LastName = "Doe",
                   EnrollmentDate = DateTime.Now
               });
               await context.SaveChangesAsync();
               
               var controller = new StudentsController(context);
               
               // Act
               var result = await controller.Index(null, null, null, 1);
               
               // Assert
               var viewResult = Assert.IsType<ViewResult>(result);
               Assert.NotNull(viewResult.Model);
           }
           
           [Fact]
           public async Task Create_ValidStudent_AddsToDatabase()
           {
               // Arrange
               using var context = TestDbContextFactory.CreateInMemoryContext();
               var controller = new StudentsController(context);
               
               var student = new Student
               {
                   FirstMidName = "Jane",
                   LastName = "Smith",
                   EnrollmentDate = DateTime.Now
               };
               
               // Act
               var result = await controller.Create(student);
               
               // Assert
               Assert.Equal(1, context.Students.Count());
               var savedStudent = context.Students.First();
               Assert.Equal("Jane", savedStudent.FirstMidName);
           }
           
           [Fact]
           public async Task Details_ExistingId_ReturnsStudent()
           {
               // Arrange
               using var context = TestDbContextFactory.CreateInMemoryContext();
               var student = new Student 
               { 
                   FirstMidName = "Test", 
                   LastName = "User",
                   EnrollmentDate = DateTime.Now
               };
               context.Students.Add(student);
               await context.SaveChangesAsync();
               
               var controller = new StudentsController(context);
               
               // Act
               var result = await controller.Details(student.ID);
               
               // Assert
               var viewResult = Assert.IsType<ViewResult>(result);
               var model = Assert.IsType<Student>(viewResult.Model);
               Assert.Equal("Test", model.FirstMidName);
           }
           
           [Fact]
           public async Task Details_NonExistingId_ReturnsNotFound()
           {
               // Arrange
               using var context = TestDbContextFactory.CreateInMemoryContext();
               var controller = new StudentsController(context);
               
               // Act
               var result = await controller.Details(999);
               
               // Assert
               Assert.IsType<NotFoundResult>(result);
           }
       }
   }
   ```

3. **Create Tests for Other Controllers**

   Repeat similar patterns for:
   - `CoursesControllerTests.cs`
   - `InstructorsControllerTests.cs`
   - `DepartmentsControllerTests.cs`

4. **Run Tests**

   ```powershell
   dotnet test
   ```

### Step 1.3: Write Service Tests

**Time Estimate:** 4-6 hours

Create `Services/NotificationServiceTests.cs`:

```csharp
using ContosoUniversity.Services;
using ContosoUniversity.Models;
using System;
using Xunit;

namespace ContosoUniversity.UnitTests.Services
{
    public class NotificationServiceTests
    {
        [Fact]
        public void SendNotification_ValidInput_DoesNotThrow()
        {
            // Arrange
            var service = new NotificationService();
            
            // Act & Assert
            var exception = Record.Exception(() => 
                service.SendNotification("Student", "123", EntityOperation.CREATE, "TestUser"));
            
            Assert.Null(exception);
        }
        
        [Fact]
        public void SendNotification_NullEntityType_DoesNotThrow()
        {
            // Arrange
            var service = new NotificationService();
            
            // Act & Assert
            // Service should handle errors gracefully
            var exception = Record.Exception(() => 
                service.SendNotification(null, "123", EntityOperation.CREATE));
            
            Assert.Null(exception); // Should not throw, errors are logged
        }
    }
}
```

### Step 1.4: Measure Code Coverage

**Time Estimate:** 2 hours

1. **Install Coverage Tool**

   ```powershell
   dotnet tool install -g dotnet-coverage
   ```

2. **Run Tests with Coverage**

   ```powershell
   dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura
   
   # Or using dotnet-coverage
   dotnet-coverage collect "dotnet test" -f xml -o coverage.xml
   ```

3. **Generate Coverage Report**

   ```powershell
   # Install ReportGenerator
   dotnet tool install -g dotnet-reportgenerator-globaltool
   
   # Generate HTML report
   reportgenerator -reports:coverage.xml -targetdir:coverage-report -reporttypes:Html
   
   # Open report
   start coverage-report/index.html
   ```

4. **Document Coverage**

   Update `MIGRATION_LOG.md`:

   ```markdown
   ## Test Coverage
   - Controllers: 65%
   - Services: 80%
   - Data Layer: 70%
   - Overall: 68%
   ```

### Step 1.5: Commit Test Suite

**Time Estimate:** 30 minutes

```bash
git add .
git commit -m "test: Add unit test suite with 68% code coverage"
git push
```

---

## Phase 2: Framework Migration (Week 3-4)

### Step 2.1: Convert Project to SDK-Style Format

**Time Estimate:** 4-6 hours

1. **Backup Current .csproj**

   ```powershell
   Copy-Item ContosoUniversity.csproj ContosoUniversity.csproj.old
   ```

2. **Use try-convert Tool**

   ```powershell
   # Navigate to project directory
   cd ContosoUniversity
   
   # Run try-convert
   try-convert -p ContosoUniversity.csproj -t web
   
   # Review the converted project file
   ```

3. **Manual Conversion (if try-convert fails)**

   Create new `ContosoUniversity.csproj`:

   ```xml
   <Project Sdk="Microsoft.NET.Sdk.Web">
   
     <PropertyGroup>
       <TargetFramework>net8.0</TargetFramework>
       <Nullable>enable</Nullable>
       <ImplicitUsings>enable</ImplicitUsings>
       <RootNamespace>ContosoUniversity</RootNamespace>
       <AssemblyName>ContosoUniversity</AssemblyName>
     </PropertyGroup>
   
     <ItemGroup>
       <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
       <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
       <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />
       <!-- Add other packages as needed -->
     </ItemGroup>
   
   </Project>
   ```

4. **Remove Old Files**

   ```powershell
   # Remove packages.config (no longer needed)
   Remove-Item packages.config
   
   # Remove Properties/AssemblyInfo.cs (handled by SDK)
   Remove-Item Properties/AssemblyInfo.cs
   ```

5. **Build and Fix Issues**

   ```powershell
   dotnet build
   
   # Address any build errors
   ```

### Step 2.2: Migrate from ASP.NET MVC to ASP.NET Core MVC

**Time Estimate:** 16-24 hours

1. **Update NuGet Packages**

   Update `ContosoUniversity.csproj`:

   ```xml
   <ItemGroup>
     <!-- ASP.NET Core -->
     <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.2.0" />
     
     <!-- Entity Framework Core -->
     <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
     <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
     <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />
     
     <!-- Remove these - no longer needed in .NET Core -->
     <!-- System.Web.Mvc -->
     <!-- Microsoft.AspNet.* packages -->
   </ItemGroup>
   ```

2. **Create Program.cs**

   Create `Program.cs`:

   ```csharp
   using ContosoUniversity.Data;
   using Microsoft.EntityFrameworkCore;
   
   var builder = WebApplication.CreateBuilder(args);
   
   // Add services to the container
   builder.Services.AddControllersWithViews();
   
   // Configure Entity Framework
   builder.Services.AddDbContext<SchoolContext>(options =>
       options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
   
   // Add other services
   builder.Services.AddScoped<NotificationService>();
   
   var app = builder.Build();
   
   // Configure the HTTP request pipeline
   if (!app.Environment.IsDevelopment())
   {
       app.UseExceptionHandler("/Home/Error");
       app.UseHsts();
   }
   
   app.UseHttpsRedirection();
   app.UseStaticFiles();
   
   app.UseRouting();
   
   app.UseAuthorization();
   
   app.MapControllerRoute(
       name: "default",
       pattern: "{controller=Home}/{action=Index}/{id?}");
   
   // Initialize database
   using (var scope = app.Services.CreateScope())
   {
       var services = scope.ServiceProvider;
       var context = services.GetRequiredService<SchoolContext>();
       DbInitializer.Initialize(context);
   }
   
   app.Run();
   ```

3. **Create appsettings.json**

   Create `appsettings.json`:

   ```json
   {
     "Logging": {
       "LogLevel": {
         "Default": "Information",
         "Microsoft.AspNetCore": "Warning",
         "Microsoft.EntityFrameworkCore": "Warning"
       }
     },
     "AllowedHosts": "*",
     "ConnectionStrings": {
       "DefaultConnection": "Server=(LocalDb)\\MSSQLLocalDB;Database=ContosoUniversityNoAuthEFCore;Trusted_Connection=True;MultipleActiveResultSets=true"
     },
     "NotificationQueuePath": ".\\Private$\\ContosoUniversityNotifications"
   }
   ```

4. **Update Controllers**

   Update all controllers to use ASP.NET Core patterns:

   **Before (StudentsController.cs):**
   ```csharp
   using System.Web.Mvc;
   
   public class StudentsController : BaseController
   {
       public ActionResult Index()
       {
           return View(students);
       }
   }
   ```

   **After (StudentsController.cs):**
   ```csharp
   using Microsoft.AspNetCore.Mvc;
   
   public class StudentsController : Controller
   {
       private readonly SchoolContext _context;
       private readonly NotificationService _notificationService;
       
       public StudentsController(SchoolContext context, NotificationService notificationService)
       {
           _context = context;
           _notificationService = notificationService;
       }
       
       public async Task<IActionResult> Index()
       {
           return View(await students.ToListAsync());
       }
   }
   ```

5. **Update Global.asax Functionality**

   Delete `Global.asax` and `Global.asax.cs` - functionality moved to `Program.cs`.

6. **Update SchoolContextFactory**

   Update `Data/SchoolContextFactory.cs`:

   ```csharp
   using Microsoft.EntityFrameworkCore;
   using Microsoft.EntityFrameworkCore.Design;
   
   namespace ContosoUniversity.Data
   {
       public class SchoolContextFactory : IDesignTimeDbContextFactory<SchoolContext>
       {
           public SchoolContext CreateDbContext(string[] args)
           {
               var optionsBuilder = new DbContextOptionsBuilder<SchoolContext>();
               optionsBuilder.UseSqlServer("Server=(LocalDb)\\MSSQLLocalDB;Database=ContosoUniversityNoAuthEFCore;Trusted_Connection=True;MultipleActiveResultSets=true");
               
               return new SchoolContext(optionsBuilder.Options);
           }
       }
   }
   ```

7. **Update Razor Views**

   Update `Views/_ViewImports.cshtml`:

   ```cshtml
   @using ContosoUniversity
   @using ContosoUniversity.Models
   @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
   ```

8. **Update Model Binding**

   Change from:
   ```csharp
   [HttpPost]
   [ValidateAntiForgeryToken]
   public ActionResult Create([Bind(Include = "FirstMidName,LastName,EnrollmentDate")] Student student)
   ```

   To:
   ```csharp
   [HttpPost]
   [ValidateAntiForgeryToken]
   public async Task<IActionResult> Create([Bind("FirstMidName,LastName,EnrollmentDate")] Student student)
   ```

### Step 2.3: Update File Upload Functionality

**Time Estimate:** 4 hours

Update file upload in CoursesController:

**Before:**
```csharp
public ActionResult Create(Course course, HttpPostedFileBase file)
{
    if (file != null && file.ContentLength > 0)
    {
        var fileName = Path.GetFileName(file.FileName);
        var path = Path.Combine(Server.MapPath("~/Uploads/TeachingMaterials"), fileName);
        file.SaveAs(path);
    }
}
```

**After:**
```csharp
public async Task<IActionResult> Create(Course course, IFormFile file)
{
    if (file != null && file.Length > 0)
    {
        var fileName = Path.GetFileName(file.FileName);
        var path = Path.Combine(_webHostEnvironment.WebRootPath, "Uploads", "TeachingMaterials", fileName);
        
        using var stream = new FileStream(path, FileMode.Create);
        await file.CopyToAsync(stream);
    }
}
```

Add IWebHostEnvironment injection:

```csharp
public class CoursesController : Controller
{
    private readonly SchoolContext _context;
    private readonly IWebHostEnvironment _webHostEnvironment;
    
    public CoursesController(SchoolContext context, IWebHostEnvironment webHostEnvironment)
    {
        _context = context;
        _webHostEnvironment = webHostEnvironment;
    }
}
```

### Step 2.4: Build and Test

**Time Estimate:** 4-6 hours

1. **Build the Project**

   ```powershell
   dotnet build
   
   # Fix all compilation errors
   ```

2. **Update Tests**

   Update test project to target .NET 8:

   ```xml
   <Project Sdk="Microsoft.NET.Sdk">
     <PropertyGroup>
       <TargetFramework>net8.0</TargetFramework>
     </PropertyGroup>
   </Project>
   ```

3. **Run Tests**

   ```powershell
   dotnet test
   
   # All tests should pass
   ```

4. **Run Application**

   ```powershell
   dotnet run
   
   # Test in browser
   # Navigate to https://localhost:5001
   ```

5. **Verify All Features**

   - [ ] Home page loads
   - [ ] Student CRUD operations work
   - [ ] Course CRUD operations work
   - [ ] Instructor CRUD operations work
   - [ ] Department CRUD operations work
   - [ ] File uploads work
   - [ ] Database operations work

### Step 2.5: Commit Migration

**Time Estimate:** 30 minutes

```bash
git add .
git commit -m "feat: Migrate from .NET Framework 4.8 to .NET 8.0"
git push
```

---

## Phase 3: Azure Integration (Week 5-6)

### Step 3.1: Migrate to Azure SQL Database

**Time Estimate:** 4-6 hours

1. **Create Azure SQL Database**

   ```bash
   # Set variables
   RESOURCE_GROUP="contoso-university-rg"
   SQL_SERVER="contoso-university-sql-$(date +%s)"
   DATABASE="ContosoUniversity"
   ADMIN_USER="sqladmin"
   ADMIN_PASSWORD="YourSecurePassword123!"
   LOCATION="eastus"
   
   # Create SQL Server
   az sql server create \
     --name $SQL_SERVER \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --admin-user $ADMIN_USER \
     --admin-password $ADMIN_PASSWORD
   
   # Configure firewall
   az sql server firewall-rule create \
     --resource-group $RESOURCE_GROUP \
     --server $SQL_SERVER \
     --name AllowAzureServices \
     --start-ip-address 0.0.0.0 \
     --end-ip-address 0.0.0.0
   
   # Create database
   az sql db create \
     --resource-group $RESOURCE_GROUP \
     --server $SQL_SERVER \
     --name $DATABASE \
     --service-objective S0 \
     --backup-storage-redundancy Local
   ```

2. **Update Connection String**

   Update `appsettings.json`:

   ```json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=tcp:contoso-university-sql-xxxxx.database.windows.net,1433;Initial Catalog=ContosoUniversity;Persist Security Info=False;User ID=sqladmin;Password=YourSecurePassword123!;MultipleActiveResultSets=True;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
     }
   }
   ```

3. **Migrate Schema and Data**

   ```powershell
   # Update database with migrations
   dotnet ef database update
   
   # Or export/import data
   # Export from LocalDB
   sqlcmd -S "(LocalDb)\MSSQLLocalDB" -d ContosoUniversityNoAuthEFCore -Q "SELECT * FROM Student" -o students.csv -s"," -w 700
   
   # Import to Azure SQL
   # Use SQL Server Management Studio or Azure Data Studio
   ```

4. **Test Connection**

   ```powershell
   dotnet run
   
   # Verify application connects to Azure SQL
   ```

### Step 3.2: Replace MSMQ with Azure Service Bus

**Time Estimate:** 8-12 hours

1. **Create Azure Service Bus**

   ```bash
   # Create Service Bus namespace
   SERVICEBUS_NAMESPACE="contoso-university-sb-$(date +%s)"
   
   az servicebus namespace create \
     --name $SERVICEBUS_NAMESPACE \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --sku Standard
   
   # Create queue
   az servicebus queue create \
     --namespace-name $SERVICEBUS_NAMESPACE \
     --resource-group $RESOURCE_GROUP \
     --name notifications \
     --max-size 1024
   
   # Get connection string
   az servicebus namespace authorization-rule keys list \
     --resource-group $RESOURCE_GROUP \
     --namespace-name $SERVICEBUS_NAMESPACE \
     --name RootManageSharedAccessKey \
     --query primaryConnectionString -o tsv
   ```

2. **Install Azure Service Bus Package**

   ```powershell
   dotnet add package Azure.Messaging.ServiceBus --version 7.17.0
   ```

3. **Update NotificationService**

   Create new `Services/NotificationService.cs`:

   ```csharp
   using Azure.Messaging.ServiceBus;
   using ContosoUniversity.Models;
   using Newtonsoft.Json;
   using System;
   using System.Threading.Tasks;
   
   namespace ContosoUniversity.Services
   {
       public class NotificationService : IAsyncDisposable
       {
           private readonly ServiceBusSender _sender;
           private readonly ServiceBusClient _client;
           
           public NotificationService(IConfiguration configuration)
           {
               var connectionString = configuration["ServiceBus:ConnectionString"];
               _client = new ServiceBusClient(connectionString);
               _sender = _client.CreateSender("notifications");
           }
           
           public async Task SendNotificationAsync(string entityType, string entityId, EntityOperation operation, string userName = null)
           {
               await SendNotificationAsync(entityType, entityId, null, operation, userName);
           }
           
           public async Task SendNotificationAsync(string entityType, string entityId, string entityDisplayName, EntityOperation operation, string userName = null)
           {
               try
               {
                   var notification = new Notification
                   {
                       EntityType = entityType,
                       EntityId = entityId,
                       Operation = operation.ToString(),
                       Message = GenerateMessage(entityType, entityId, entityDisplayName, operation),
                       CreatedAt = DateTime.UtcNow,
                       CreatedBy = userName ?? "System",
                       IsRead = false
                   };
                   
                   var jsonMessage = JsonConvert.SerializeObject(notification);
                   var message = new ServiceBusMessage(jsonMessage)
                   {
                       Subject = $"{entityType} {operation}",
                       MessageId = Guid.NewGuid().ToString()
                   };
                   
                   await _sender.SendMessageAsync(message);
               }
               catch (Exception ex)
               {
                   // Log error but don't break the main operation
                   Console.WriteLine($"Failed to send notification: {ex.Message}");
               }
           }
           
           private string GenerateMessage(string entityType, string entityId, string entityDisplayName, EntityOperation operation)
           {
               var displayText = !string.IsNullOrWhiteSpace(entityDisplayName) 
                   ? $"{entityType} '{entityDisplayName}'" 
                   : $"{entityType} (ID: {entityId})";
               
               return operation switch
               {
                   EntityOperation.CREATE => $"New {displayText} has been created",
                   EntityOperation.UPDATE => $"{displayText} has been updated",
                   EntityOperation.DELETE => $"{displayText} has been deleted",
                   _ => $"{displayText} operation: {operation}"
               };
           }
           
           public async ValueTask DisposeAsync()
           {
               await _sender.DisposeAsync();
               await _client.DisposeAsync();
           }
       }
   }
   ```

4. **Update Controllers**

   Change all notification calls from synchronous to async:

   ```csharp
   // Before
   SendEntityNotification("Student", student.ID.ToString(), EntityOperation.CREATE);
   
   // After
   await _notificationService.SendNotificationAsync("Student", student.ID.ToString(), EntityOperation.CREATE);
   ```

5. **Update Configuration**

   Update `appsettings.json`:

   ```json
   {
     "ServiceBus": {
       "ConnectionString": "Endpoint=sb://contoso-university-sb-xxxxx.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=xxxxx"
     }
   }
   ```

6. **Register Service**

   Update `Program.cs`:

   ```csharp
   builder.Services.AddSingleton<NotificationService>();
   ```

### Step 3.3: Migrate to Azure Blob Storage

**Time Estimate:** 6-8 hours

1. **Create Storage Account**

   ```bash
   # Create storage account
   STORAGE_ACCOUNT="contosouniversitystorage$(date +%s | cut -c 6-)"
   
   az storage account create \
     --name $STORAGE_ACCOUNT \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --sku Standard_LRS
   
   # Create container
   az storage container create \
     --name teaching-materials \
     --account-name $STORAGE_ACCOUNT \
     --public-access off
   
   # Get connection string
   az storage account show-connection-string \
     --name $STORAGE_ACCOUNT \
     --resource-group $RESOURCE_GROUP \
     --query connectionString -o tsv
   ```

2. **Install Azure Storage Package**

   ```powershell
   dotnet add package Azure.Storage.Blobs --version 12.19.0
   ```

3. **Create Blob Storage Service**

   Create `Services/BlobStorageService.cs`:

   ```csharp
   using Azure.Storage.Blobs;
   using Azure.Storage.Blobs.Models;
   using System.IO;
   using System.Threading.Tasks;
   
   namespace ContosoUniversity.Services
   {
       public class BlobStorageService
       {
           private readonly BlobServiceClient _blobServiceClient;
           private readonly string _containerName = "teaching-materials";
           
           public BlobStorageService(IConfiguration configuration)
           {
               var connectionString = configuration["AzureStorage:ConnectionString"];
               _blobServiceClient = new BlobServiceClient(connectionString);
           }
           
           public async Task<string> UploadFileAsync(Stream fileStream, string fileName, string contentType)
           {
               var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
               await containerClient.CreateIfNotExistsAsync();
               
               var blobClient = containerClient.GetBlobClient(fileName);
               
               await blobClient.UploadAsync(fileStream, new BlobHttpHeaders { ContentType = contentType });
               
               return blobClient.Uri.ToString();
           }
           
           public async Task DeleteFileAsync(string fileName)
           {
               var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
               var blobClient = containerClient.GetBlobClient(fileName);
               
               await blobClient.DeleteIfExistsAsync();
           }
       }
   }
   ```

4. **Update CoursesController**

   ```csharp
   public class CoursesController : Controller
   {
       private readonly SchoolContext _context;
       private readonly BlobStorageService _blobStorageService;
       
       public CoursesController(SchoolContext context, BlobStorageService blobStorageService)
       {
           _context = context;
           _blobStorageService = blobStorageService;
       }
       
       [HttpPost]
       [ValidateAntiForgeryToken]
       public async Task<IActionResult> Create(Course course, IFormFile file)
       {
           if (file != null && file.Length > 0)
           {
               var fileName = $"course_{course.CourseID}_{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
               
               using var stream = file.OpenReadStream();
               var blobUrl = await _blobStorageService.UploadFileAsync(stream, fileName, file.ContentType);
               
               course.TeachingMaterialImagePath = blobUrl;
           }
           
           // ... rest of create logic
       }
   }
   ```

5. **Update Configuration**

   Update `appsettings.json`:

   ```json
   {
     "AzureStorage": {
       "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=contosouniversitystoragxxxxx;AccountKey=xxxxx;EndpointSuffix=core.windows.net"
     }
   }
   ```

6. **Register Service**

   Update `Program.cs`:

   ```csharp
   builder.Services.AddScoped<BlobStorageService>();
   ```

### Step 3.4: Test Azure Integration

**Time Estimate:** 4 hours

```powershell
dotnet run

# Test:
# - Database operations with Azure SQL
# - Notifications via Service Bus
# - File uploads to Blob Storage
```

### Step 3.5: Commit Azure Integration

**Time Estimate:** 30 minutes

```bash
git add .
git commit -m "feat: Integrate Azure SQL, Service Bus, and Blob Storage"
git push
```

---

## Phase 4: Deployment & Testing (Week 7-8)

### Step 4.1: Containerize Application

**Time Estimate:** 4-6 hours

1. **Create Dockerfile**

   Create `Dockerfile`:

   ```dockerfile
   FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
   WORKDIR /app
   EXPOSE 80
   EXPOSE 443
   
   FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
   WORKDIR /src
   COPY ["ContosoUniversity.csproj", "."]
   RUN dotnet restore "ContosoUniversity.csproj"
   COPY . .
   RUN dotnet build "ContosoUniversity.csproj" -c Release -o /app/build
   
   FROM build AS publish
   RUN dotnet publish "ContosoUniversity.csproj" -c Release -o /app/publish /p:UseAppHost=false
   
   FROM base AS final
   WORKDIR /app
   COPY --from=publish /app/publish .
   ENTRYPOINT ["dotnet", "ContosoUniversity.dll"]
   ```

2. **Create .dockerignore**

   Create `.dockerignore`:

   ```
   **/.git
   **/.vs
   **/.vscode
   **/bin
   **/obj
   **/Uploads
   **/.gitignore
   **/Dockerfile*
   **/docker-compose*
   ```

3. **Build Docker Image**

   ```bash
   docker build -t contoso-university:v1 .
   ```

4. **Test Locally**

   ```bash
   docker run -p 8080:80 \
     -e ConnectionStrings__DefaultConnection="your-azure-sql-connection-string" \
     -e ServiceBus__ConnectionString="your-service-bus-connection-string" \
     -e AzureStorage__ConnectionString="your-storage-connection-string" \
     contoso-university:v1
   ```

### Step 4.2: Deploy to Azure Container Apps

**Time Estimate:** 4-6 hours

1. **Create Container Registry**

   ```bash
   ACR_NAME="contosouniversityacr$(date +%s | cut -c 6-)"
   
   az acr create \
     --resource-group $RESOURCE_GROUP \
     --name $ACR_NAME \
     --sku Basic
   ```

2. **Push Image to ACR**

   ```bash
   # Login to ACR
   az acr login --name $ACR_NAME
   
   # Tag image
   docker tag contoso-university:v1 $ACR_NAME.azurecr.io/contoso-university:v1
   
   # Push image
   docker push $ACR_NAME.azurecr.io/contoso-university:v1
   ```

3. **Create Container Apps Environment**

   ```bash
   az containerapp env create \
     --name contoso-env \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION
   ```

4. **Deploy Container App**

   ```bash
   az containerapp create \
     --name contoso-university-app \
     --resource-group $RESOURCE_GROUP \
     --environment contoso-env \
     --image $ACR_NAME.azurecr.io/contoso-university:v1 \
     --target-port 80 \
     --ingress external \
     --registry-server $ACR_NAME.azurecr.io \
     --cpu 0.5 --memory 1Gi \
     --min-replicas 1 --max-replicas 3
   ```

5. **Configure Secrets**

   ```bash
   # Set secrets
   az containerapp secret set \
     --name contoso-university-app \
     --resource-group $RESOURCE_GROUP \
     --secrets \
       sqlconnection="your-sql-connection-string" \
       sbconnection="your-servicebus-connection-string" \
       storageconnection="your-storage-connection-string"
   
   # Set environment variables
   az containerapp update \
     --name contoso-university-app \
     --resource-group $RESOURCE_GROUP \
     --set-env-vars \
       "ConnectionStrings__DefaultConnection=secretref:sqlconnection" \
       "ServiceBus__ConnectionString=secretref:sbconnection" \
       "AzureStorage__ConnectionString=secretref:storageconnection"
   ```

6. **Get App URL**

   ```bash
   az containerapp show \
     --name contoso-university-app \
     --resource-group $RESOURCE_GROUP \
     --query properties.configuration.ingress.fqdn -o tsv
   ```

### Step 4.3: Set Up CI/CD Pipeline

**Time Estimate:** 4-6 hours

Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy to Azure Container Apps

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  ACR_NAME: contosouniversityacrxxxxx
  RESOURCE_GROUP: contoso-university-rg
  CONTAINER_APP_NAME: contoso-university-app
  IMAGE_NAME: contoso-university

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Test
      run: dotnet test --no-build --configuration Release --verbosity normal
    
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Build and push Docker image
      run: |
        az acr login --name ${{ env.ACR_NAME }}
        docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
        docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
    
    - name: Deploy to Container Apps
      run: |
        az containerapp update \
          --name ${{ env.CONTAINER_APP_NAME }} \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --image ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

### Step 4.4: Testing and Validation

**Time Estimate:** 8-12 hours

1. **Functional Testing**

   - [ ] All pages load correctly
   - [ ] Student CRUD operations work
   - [ ] Course CRUD operations work
   - [ ] Instructor CRUD operations work
   - [ ] Department CRUD operations work
   - [ ] File uploads to Blob Storage work
   - [ ] Notifications via Service Bus work
   - [ ] Database operations with Azure SQL work

2. **Performance Testing**

   ```bash
   # Install Apache Bench
   sudo apt-get install apache2-utils
   
   # Run load test
   ab -n 1000 -c 10 https://your-app-url.azurecontainerapps.io/
   ```

3. **Security Testing**

   - [ ] HTTPS enforced
   - [ ] No secrets in code
   - [ ] SQL injection protection (parameterized queries)
   - [ ] CSRF protection enabled
   - [ ] Authentication working (if implemented)

### Step 4.5: Final Commit and Documentation

**Time Estimate:** 2 hours

```bash
git add .
git commit -m "feat: Complete Azure deployment with CI/CD pipeline"
git push
```

Update `README.md` with deployment instructions and URLs.

---

## Post-Migration Checklist

- [ ] All code migrated to .NET 8
- [ ] All tests passing (80%+ coverage)
- [ ] Application deployed to Azure
- [ ] CI/CD pipeline operational
- [ ] Database migrated to Azure SQL
- [ ] MSMQ replaced with Service Bus
- [ ] File storage migrated to Blob Storage
- [ ] Performance testing completed
- [ ] Security review completed
- [ ] Documentation updated
- [ ] Team trained on new architecture
- [ ] Monitoring and alerts configured

---

## Troubleshooting Common Issues

### Build Errors

**Issue:** `The type or namespace name 'System.Web' could not be found`

**Solution:** Remove all `using System.Web.*` statements and replace with ASP.NET Core equivalents.

### Runtime Errors

**Issue:** `Unable to connect to Azure SQL Database`

**Solution:** Check firewall rules and connection string. Ensure IP is allowed.

**Issue:** `Service Bus authentication failed`

**Solution:** Verify connection string and ensure namespace exists.

### Deployment Issues

**Issue:** Container app won't start

**Solution:** Check logs:
```bash
az containerapp logs show \
  --name contoso-university-app \
  --resource-group contoso-university-rg \
  --follow
```

---

## Additional Resources

- [.NET 8 Migration Guide](https://docs.microsoft.com/dotnet/core/porting/)
- [ASP.NET Core Documentation](https://docs.microsoft.com/aspnet/core/)
- [Azure Container Apps Documentation](https://docs.microsoft.com/azure/container-apps/)
- [Entity Framework Core 8](https://docs.microsoft.com/ef/core/)

---

**Document Version:** 1.0  
**Last Updated:** October 27, 2025  
**Estimated Total Time:** 240-320 hours (6-8 weeks for a team of 2-3 developers)
