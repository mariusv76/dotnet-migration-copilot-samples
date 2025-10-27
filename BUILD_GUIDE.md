# Build Guide for ContosoUniversity Application

This guide provides detailed instructions for building the ContosoUniversity application in its current state (.NET Framework 4.8).

---

## Current Build Status

**Platform:** Windows Only  
**Build Tool:** MSBuild (Visual Studio)  
**Framework:** .NET Framework 4.8  
**Status:** ⚠️ Cannot build on Linux/macOS

### Why Can't This Build on Linux?

The project uses the old-style `.csproj` format that imports Visual Studio-specific build targets:

```xml
<Import Project="$(VSToolsPath)\WebApplications\Microsoft.WebApplication.targets" />
```

These targets are part of Visual Studio and are **not available** in the cross-platform .NET SDK. This is by design for .NET Framework projects.

---

## Prerequisites

### Required Software

1. **Windows Operating System**
   - Windows 10 (version 1809 or later)
   - Windows 11
   - Windows Server 2016 or later

2. **Visual Studio 2019 or 2022**
   - Download: [https://visualstudio.microsoft.com/](https://visualstudio.microsoft.com/)
   - Edition: Community (free), Professional, or Enterprise
   
   **Required Workloads:**
   - ✅ ASP.NET and web development
   - ✅ .NET desktop development (optional, but recommended)

3. **.NET Framework 4.8 Developer Pack**
   - Usually included with Visual Studio
   - Direct download: [https://dotnet.microsoft.com/download/dotnet-framework/net48](https://dotnet.microsoft.com/download/dotnet-framework/net48)
   - Verify installation:
     ```powershell
     Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full" | Select-Object Version
     # Should show 4.8.xxxxx
     ```

4. **SQL Server LocalDB**
   - Included with Visual Studio "Data storage and processing" workload
   - Or download SQL Server Express: [https://www.microsoft.com/sql-server/sql-server-downloads](https://www.microsoft.com/sql-server/sql-server-downloads)
   - Verify installation:
     ```powershell
     sqllocaldb info
     # Should list available instances, including MSSQLLocalDB
     ```

5. **Microsoft Message Queue (MSMQ) Server**
   - Required for the notification system
   - Installation instructions below

### Installing MSMQ

MSMQ is a Windows Feature that must be enabled:

**Windows 10/11:**
1. Open **Settings** → **Apps** → **Optional Features** → **More Windows features**
2. Or search for "Turn Windows features on or off"
3. Navigate to **Microsoft Message Queue (MSMQ) Server**
4. Expand and check:
   - ✅ **Microsoft Message Queue (MSMQ) Server Core** (required)
   - ❌ MSMQ Active Directory Domain Services Integration (not needed)
   - ❌ MSMQ HTTP Support (not needed)
5. Click **OK** and wait for installation
6. Restart if prompted

**Windows Server:**
1. Open **Server Manager**
2. Click **Add Roles and Features**
3. Navigate to **Features** → **Message Queuing**
4. Select **Message Queuing Services** → **Message Queuing Server**
5. Complete the wizard

**Verify MSMQ Installation:**
```powershell
# Check if MSMQ service is running
Get-Service MSMQ
# Status should be "Running"

# Or open Computer Management
compmgmt.msc
# Navigate to Services and Applications → Message Queuing
# You should see Private Queues folder
```

---

## Building the Application

### Method 1: Visual Studio (Recommended)

This is the easiest and most reliable method.

1. **Open the Solution**
   ```
   Double-click: ContosoUniversity\ContosoUniversity.sln
   ```

2. **Restore NuGet Packages**
   - Visual Studio should automatically restore packages on open
   - If not: Right-click solution → **Restore NuGet Packages**
   - Or: **Tools** → **NuGet Package Manager** → **Package Manager Console**
     ```powershell
     Update-Package -Reinstall
     ```

3. **Build the Solution**
   - **Build** → **Build Solution** (or press `Ctrl+Shift+B`)
   - Or right-click solution → **Build Solution**

4. **Check Output Window**
   ```
   ========== Build: 1 succeeded, 0 failed, 0 up-to-date, 0 skipped ==========
   ```

5. **Run the Application**
   - Press `F5` (Debug mode) or `Ctrl+F5` (without debugging)
   - Application will launch in your default browser
   - Database will be automatically created on first run

### Method 2: MSBuild Command Line

For automated builds or CI/CD on Windows agents.

1. **Open Developer Command Prompt for Visual Studio**
   - Start Menu → Visual Studio 2022 → Developer Command Prompt for VS 2022
   - Or open PowerShell and run:
     ```powershell
     & "C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\Tools\Launch-VsDevShell.ps1"
     ```

2. **Navigate to Project Directory**
   ```powershell
   cd C:\path\to\dotnet-migration-copilot-samples\ContosoUniversity
   ```

3. **Restore NuGet Packages**
   ```powershell
   nuget restore ContosoUniversity.sln
   ```

4. **Build with MSBuild**
   ```powershell
   msbuild ContosoUniversity.sln /p:Configuration=Release /p:Platform="Any CPU"
   ```

5. **Expected Output**
   ```
   Build succeeded.
       0 Warning(s)
       0 Error(s)
   
   Time Elapsed 00:00:15.45
   ```

### Method 3: Using dotnet CLI (Limited Support)

**⚠️ Warning:** The `dotnet` CLI has limited support for .NET Framework projects and may not work correctly.

```powershell
# This will likely fail with the same error as on Linux
dotnet build ContosoUniversity.sln

# Error:
# error MSB4019: The imported project "Microsoft.WebApplication.targets" was not found
```

**Do not use this method** for .NET Framework 4.8 projects.

---

## Build Output

### Output Directory Structure

After a successful build:

```
ContosoUniversity\
└── bin\
    ├── ContosoUniversity.dll
    ├── ContosoUniversity.pdb
    ├── Web.config
    ├── Microsoft.Data.SqlClient.SNI.x64.dll
    ├── Microsoft.Data.SqlClient.SNI.x86.dll
    ├── x64\
    │   └── Microsoft.Data.SqlClient.SNI.dll
    ├── x86\
    │   └── Microsoft.Data.SqlClient.SNI.dll
    └── [NuGet package assemblies]
```

### Key Build Artifacts

- **ContosoUniversity.dll**: Main application assembly
- **ContosoUniversity.pdb**: Debug symbols
- **Web.config**: Configuration file (transformed for Release build)
- **SQL Client Native DLLs**: Required for Azure SQL connectivity

---

## Running the Application

### Using Visual Studio

1. Set **ContosoUniversity** as startup project (right-click → Set as Startup Project)
2. Press **F5** to run with debugging
3. Browser opens automatically to `https://localhost:44300/`
4. First run will create the database and seed initial data

### Using IIS Express (Command Line)

```powershell
cd C:\path\to\ContosoUniversity

# Start IIS Express
"C:\Program Files\IIS Express\iisexpress.exe" /path:%CD% /port:44300 /systray:false
```

### First-Run Checklist

When you run the application for the first time:

- [ ] Database `ContosoUniversityNoAuthEFCore` is created in LocalDB
- [ ] Sample data is seeded (students, courses, instructors, departments)
- [ ] MSMQ queue `.\Private$\ContosoUniversityNotifications` is created
- [ ] Upload directory `/Uploads/TeachingMaterials/` is accessible
- [ ] Application loads without errors

---

## Common Build Issues

### Issue 1: NuGet Package Restore Fails

**Symptoms:**
```
error : Unable to find version 'x.x.x' of package 'PackageName'
```

**Solutions:**
```powershell
# Clear NuGet cache
nuget locals all -clear

# Restore again
nuget restore ContosoUniversity.sln -Source https://api.nuget.org/v3/index.json

# Or in Visual Studio
Tools → Options → NuGet Package Manager → Package Sources
# Ensure "nuget.org" source is enabled
```

### Issue 2: Missing .NET Framework 4.8

**Symptoms:**
```
error MSB3644: The reference assemblies for framework ".NETFramework,Version=v4.8" were not found
```

**Solutions:**
1. Install .NET Framework 4.8 Developer Pack
2. Verify installation:
   ```powershell
   Get-ChildItem "C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.8"
   ```

### Issue 3: SQL Server LocalDB Not Found

**Symptoms:**
- Application starts but crashes when accessing database
- Error: "A network-related or instance-specific error occurred"

**Solutions:**
```powershell
# Check LocalDB installation
sqllocaldb info

# If not installed, download SQL Server Express with LocalDB:
# https://www.microsoft.com/sql-server/sql-server-downloads

# Create instance if needed
sqllocaldb create MSSQLLocalDB
sqllocaldb start MSSQLLocalDB
```

### Issue 4: MSMQ Queue Creation Fails

**Symptoms:**
- Application runs but notifications don't work
- Error in debug output: "Message Queuing is not installed"

**Solutions:**
1. Verify MSMQ is installed (see Prerequisites)
2. Check MSMQ service is running:
   ```powershell
   Get-Service MSMQ
   Start-Service MSMQ  # If stopped
   ```
3. Manually create queue:
   - Open Computer Management (`compmgmt.msc`)
   - Navigate to Message Queuing → Private Queues
   - Right-click → New → Private Queue
   - Name: `ContosoUniversityNotifications`

### Issue 5: File Upload Permissions

**Symptoms:**
- Cannot upload teaching material images
- Error: "Access to the path is denied"

**Solutions:**
```powershell
# Grant write permissions to upload directory
$uploadPath = "C:\path\to\ContosoUniversity\Uploads\TeachingMaterials"
$acl = Get-Acl $uploadPath
$rule = New-Object System.Security.AccessFile.FileSystemAccessRule("IIS_IUSRS", "Modify", "Allow")
$acl.SetAccessRule($rule)
Set-Acl $uploadPath $acl
```

### Issue 6: Visual Studio Build Targets Not Found

**Symptoms:**
```
error MSB4019: The imported project ".../Microsoft.WebApplication.targets" was not found
```

**Solutions:**
1. Ensure Visual Studio "ASP.NET and web development" workload is installed
2. Repair Visual Studio installation
3. For CI/CD, use Windows build agents with Visual Studio Build Tools

---

## CI/CD on Windows

### GitHub Actions (Windows Runner)

```yaml
name: Build .NET Framework App

on: [push, pull_request]

jobs:
  build:
    runs-on: windows-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup MSBuild
      uses: microsoft/setup-msbuild@v1
    
    - name: Setup NuGet
      uses: nuget/setup-nuget@v1
    
    - name: Restore NuGet packages
      run: nuget restore ContosoUniversity/ContosoUniversity.sln
    
    - name: Build
      run: msbuild ContosoUniversity/ContosoUniversity.sln /p:Configuration=Release /p:Platform="Any CPU"
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-artifacts
        path: ContosoUniversity/bin/
```

### Azure DevOps (Windows Agent)

```yaml
trigger:
  - main

pool:
  vmImage: 'windows-latest'

steps:
- task: NuGetToolInstaller@1

- task: NuGetCommand@2
  inputs:
    command: 'restore'
    restoreSolution: 'ContosoUniversity/ContosoUniversity.sln'

- task: VSBuild@1
  inputs:
    solution: 'ContosoUniversity/ContosoUniversity.sln'
    platform: 'Any CPU'
    configuration: 'Release'
```

---

## Database Setup

### LocalDB Connection String

Default connection string in `Web.config`:

```xml
<connectionStrings>
  <add name="DefaultConnection" 
       connectionString="Data Source=(LocalDb)\MSSQLLocalDB;
                        Initial Catalog=ContosoUniversityNoAuthEFCore;
                        Integrated Security=True;
                        MultipleActiveResultSets=True" />
</connectionStrings>
```

### Manual Database Creation

If automatic database creation fails:

```sql
-- Connect to LocalDB
sqlcmd -S "(LocalDb)\MSSQLLocalDB"

-- Create database
CREATE DATABASE ContosoUniversityNoAuthEFCore;
GO

-- The application will create tables on first run
```

### Reset Database

To start fresh:

```powershell
# Stop application in Visual Studio

# Delete database
sqllocaldb stop MSSQLLocalDB
sqllocaldb delete MSSQLLocalDB
sqllocaldb create MSSQLLocalDB
sqllocaldb start MSSQLLocalDB

# Or via SQL
sqlcmd -S "(LocalDb)\MSSQLLocalDB" -Q "DROP DATABASE ContosoUniversityNoAuthEFCore"

# Restart application - database will be recreated
```

---

## Development Environment Setup

### Recommended Visual Studio Extensions

- **Web Essentials** - Enhanced web development features
- **ReSharper** - Code analysis and refactoring (optional, paid)
- **CodeMaid** - Code cleanup and organization
- **.NET Framework Analyzer** - Framework API guidance

### Visual Studio Configuration

**Recommended Settings:**
1. **Tools → Options → Text Editor → C# → Advanced**
   - Enable "Generate XML documentation comments"
   - Enable "Place 'System' directives first when sorting usings"

2. **Tools → Options → Web Projects**
   - Use the 64-bit version of IIS Express

3. **Tools → Options → Debugging**
   - Enable "Just My Code" for easier debugging

---

## Performance Tips

### Build Performance

1. **Disable Antivirus Scanning** for project directories
   - Add exclusions for:
     - Project folder
     - `bin/` and `obj/` directories
     - Visual Studio installation folder

2. **Use SSD** for source code and Visual Studio

3. **Close Unnecessary Visual Studio Windows** during build

4. **Disable MSBuild Node Reuse** if experiencing issues:
   ```powershell
   # Add to environment variables
   MSBUILDDISABLENODEREUSE=1
   ```

### Runtime Performance

1. **Use Release Configuration** for performance testing
   - Debug builds have optimization disabled

2. **Enable Browser Caching** in Web.config

3. **Use Production LocalDB** instance:
   ```powershell
   sqllocaldb create ProductionDB
   # Update connection string to use ProductionDB
   ```

---

## Next Steps

After successfully building the application:

1. **Run the Application** and verify all features work
2. **Review MIGRATION_ASSESSMENT.md** for modernization recommendations
3. **Add Unit Tests** before migration (see MIGRATION_ASSESSMENT.md Section 4)
4. **Plan Migration to .NET 8** for cross-platform support

### For Migration

Once you're ready to migrate to .NET 8 (for cross-platform builds):

1. **Install .NET 8 SDK**
   ```powershell
   winget install Microsoft.DotNet.SDK.8
   ```

2. **Use .NET Upgrade Assistant**
   ```powershell
   dotnet tool install -g upgrade-assistant
   upgrade-assistant upgrade ContosoUniversity.csproj
   ```

3. **Follow MIGRATION_ASSESSMENT.md** for detailed migration steps

---

## Additional Resources

### Documentation
- [ASP.NET MVC 5 Documentation](https://docs.microsoft.com/aspnet/mvc/overview/)
- [Entity Framework Core 3.1](https://docs.microsoft.com/ef/core/)
- [MSBuild Reference](https://docs.microsoft.com/visualstudio/msbuild/)

### Tools
- [Visual Studio Download](https://visualstudio.microsoft.com/)
- [.NET Framework Developer Pack](https://dotnet.microsoft.com/download/dotnet-framework)
- [SQL Server Downloads](https://www.microsoft.com/sql-server/sql-server-downloads)

### Migration Resources
- [.NET Upgrade Assistant](https://dotnet.microsoft.com/platform/upgrade-assistant)
- [Porting to .NET 8](https://docs.microsoft.com/dotnet/core/porting/)
- [MIGRATION_ASSESSMENT.md](./MIGRATION_ASSESSMENT.md) - Comprehensive migration guide

---

## Support

For issues or questions:

1. Check this guide's "Common Build Issues" section
2. Review the [MIGRATION_ASSESSMENT.md](./MIGRATION_ASSESSMENT.md)
3. Check application-specific README files:
   - [ContosoUniversity/README.md](./ContosoUniversity/README.md)
   - [ContosoUniversity/NOTIFICATION_SYSTEM_README.md](./ContosoUniversity/NOTIFICATION_SYSTEM_README.md)
   - [ContosoUniversity/SETUP_TESTING_GUIDE.md](./ContosoUniversity/SETUP_TESTING_GUIDE.md)

---

**Document Version:** 1.0  
**Last Updated:** October 27, 2025  
**Platform:** Windows Only (.NET Framework 4.8)  
**Build Tool:** MSBuild / Visual Studio 2019+
