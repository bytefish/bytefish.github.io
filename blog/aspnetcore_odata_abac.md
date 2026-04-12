title: Securing an ASP.NET Core OData Service using Attribute-based Access Control
date: 2026-04-12 14:50
tags: csharp, dotnet, odata
category: dotnet
slug: aspnetcore_odata_abac
author: Philipp Wagner
summary: This article shows how to secure an OData Service using Attribute-based Access Control. 

OData is a superpower for frontend developers. By exposing a single endpoint, you give clients the ultimate 
flexibility to query exactly what they need using `$filter`, shape the payload with `$select`, and fetch 
related data via `$expand`.

But when a client can query basically whatever they want, then... how on earth do you secure access to 
your data? And how do you prevent leaking sensitive information? How do you conditionally show or mask 
data?

In this article we'll take a look at using Attribute-based Access Control to secure an ASP.NET Core 
OData Service. It is built for PostgreSQL and SQL Server, so you are covered with both database 
systems.

The code is available in a Git repository at:

* [https://github.com/bytefish/ODataSecurity](https://github.com/bytefish/ODataSecurity)

## Table of contents ##

[TOC]

## The Problem ##

My previous approaches at authorizing OData services have been misguided and flawed. They operated 
at the application layer and left the database wide open. This is wrong.

Authorizing access at the application layer, is a bit like wearing a trench coat with nothing 
underneath: One loose button and you're exposing everything. Authorization at the database layer 
is like putting on your pants. Even if you application trips, you aren't showing the world all 
your private data.

By moving authorization into the database itself, you ensure that the data remains protected at 
its core, regardless of which application knocks at the door. 

## Attribute-based Access Control

To secure dynamic data access using OData, we'll need to have very fine-grained authorization, 
that allows us to have Row Level Security and even Field Level Security. We will take a look 
at Attribute-based Access Control (ABAC) to provide this level of granularity.

Attribute-based Access Control decides wether a user can access something based on attributes, instead 
of roles. The attributes are properties of the users, resources, actions, and the environment. Instead 
of asking *"What role does the user have?"*, ABAC asks *"Do all required attributes match a rule?"*.

In an ABAC system there are usually four types of attributes to attributes to configure access:

* User Attributes
    * Department, Job Title, Location
* Resource Attributes 
    * Document Type, Sensitivy, Owner
* Action Attributes
    * Read, Write, Delete
* Context Attributes
    * Time of day, device type,  

These attributes are then combined into a policy, which is evaluated using a Policy engine. 

I won't go deep into theory here. There are way better resources available, than anything I'd 
come up with. Let's just solve the problem at hand, which is securing our data using the database.

## ASP.NET Core OData API ##

The idea is pretty simple. The database only exposes views to the user. The views are using a session 
variable to filter out or mask data for a given user, the filtering being based on attributes and 
permissions synced to the database.

I won't paste the entire code. But let's outline the most important parts. 

You'll start by extracting the user off a token, or in our simplified example from the 
HTTP Header. We can add a small middleware to extract it and write it to the HttpContext 
Items:

```csharp
// Use a middleware to extract the user identity from the request headers and store it in the HttpContext.Items. This
// allows us to access the current user's identity in the PostgresSecurityInterceptor when setting the session variable
// for RLS and FLS in PostgreSQL.
app.Use(async (context, next) =>
{
    var userId = context.Request.Headers["X-User-Id"].FirstOrDefault() ?? "anonymous";
    context.Items["CurrentUserId"] = userId;
    await next();
});
```

Next we create an `ICurrentUserService`, which is scoped to the request lifetime. It uses 
the `IHttpContextAccessor` to extract the username off the `HttpContext.Items`:

```csharp
/// <summary>
/// Abstraction for getting the current user identity.
/// </summary>
public interface ICurrentUserService
{
    string GetCurrentUserId();
}

/// <summary>
/// HTTP-specific implementation of the CurrentUserService.
/// </summary>
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public string GetCurrentUserId()
    {
        return _httpContextAccessor.HttpContext?.Items["CurrentUserId"] as string ?? "anonymous";
    }
}
```

We'll then write a `DbConnectionInterceptor` for PostgreSQL to pass the user into the PostgreSQL 
session as the `app.current_user` variable:

```
/// <summary>
/// PostgreSQL-specific interceptor that sets a session variable with the current user's identity.
/// </summary>
public class PostgresSecurityInterceptor : DbConnectionInterceptor
{
    private readonly ICurrentUserService _currentUserService;

    public PostgresSecurityInterceptor(ICurrentUserService currentUserService)
    {
        _currentUserService = currentUserService;
    }

    public override async Task ConnectionOpenedAsync(DbConnection connection, ConnectionEndEventData eventData, CancellationToken cancellationToken)
    {
        await SetPostgresSessionVariableAsync(connection, cancellationToken);
    }

    public override void ConnectionOpened(DbConnection connection, ConnectionEndEventData eventData)
    {
        SetPostgresSessionVariableAsync(connection, CancellationToken.None).GetAwaiter().GetResult();
    }

    private async Task SetPostgresSessionVariableAsync(DbConnection connection, CancellationToken cancellationToken)
    {
        string userId = _currentUserService.GetCurrentUserId();

        // Create a command to set the session variable in PostgreSQL. This variable will be used
        // for RLS and FLS in the database views.
        using DbCommand cmd = connection.CreateCommand();

        // Sets the session variable 'app.current_user' to the current user's ID. This variable is then used in
        // the PostgreSQL views for RLS and FLS.
        cmd.CommandText = "SELECT set_config('app.current_user', @userId, false)";

        // Create and Add the parameter to prevent SQL injection and ensure proper handling of special characters
        DbParameter param = cmd.CreateParameter();
        
        param.ParameterName = "@userId";
        param.Value = userId;
        
        cmd.Parameters.Add(param);

        await cmd.ExecuteNonQueryAsync(cancellationToken);
    }
}
```

And we'll implement a similiar interceptor for SQL Server, that uses the `sp_set_session_context` to pass the 
user into the SQL Server session as `app.current_user`:


```csharp
/// <summary>
/// SQL Server-specific interceptor that sets a session context with the current user's identity. This is used for RLS and FLS in SQL Server.
/// </summary>
public class SqlServerSecurityInterceptor : DbConnectionInterceptor
{
    private readonly ICurrentUserService _currentUserService;

    public SqlServerSecurityInterceptor(ICurrentUserService currentUserService)
    {
        _currentUserService = currentUserService;
    }

    public override async Task ConnectionOpenedAsync(DbConnection connection, ConnectionEndEventData eventData, CancellationToken cancellationToken)
    {
        await SetSessionContextAsync(connection, cancellationToken);
    }

    public override void ConnectionOpened(DbConnection connection, ConnectionEndEventData eventData)
    {
        SetSessionContextAsync(connection, CancellationToken.None).GetAwaiter().GetResult();
    }

    private async Task SetSessionContextAsync(DbConnection connection, CancellationToken cancellationToken)
    {
        var userId = _currentUserService.GetCurrentUserId();
        using var cmd = connection.CreateCommand();
        
        // SQL Server uses sp_set_session_context to safely store context
        cmd.CommandText = "EXEC sp_set_session_context @key = N'app.current_user', @value = @userId;";

        var param = cmd.CreateParameter();
        
        param.ParameterName = "@userId";
        param.Value = userId;
        
        cmd.Parameters.Add(param);

        await cmd.ExecuteNonQueryAsync(cancellationToken);
    }
}
```

Next we'll define the application domain model, which is a one to one mapping to the secure views, we 
have defined in SQL: 

```csharp
/// <summary>
/// An Employee entity representing an employee in the company. The security for this entity is handled at the 
/// database level through the view 'vw_Employee_Secure', which implements Row-Level Security (RLS) and 
/// Field-Level Security (FLS).
/// </summary>
public class Employee
{
    public int Id { get; set; }
    
    public string Name { get; set; } = null!;

    public string Department { get; set; } = null!;

    public decimal? AnnualSalary { get; set; }

    public string? BonusGoal { get; set; }

    public List<BonusPayment> BonusPayments { get; set; } = [];
}

/// <summary>
/// A BonusPayment entity representing a bonus payment made to an employee. This is related to 
/// the Employee entity via the EmployeeId foreign key. The security for this entity is also 
/// handled at the database level through the view 'vw_BonusPayment_Secure'.
/// </summary>
public class BonusPayment
{
    public int Id { get; set; }

    public int EmployeeId { get; set; }

    public decimal Amount { get; set; }

    public string? Reason { get; set; }

    public Employee? Employee { get; set; }
}
```

The `DbContext` now maps to the secure views, instead of raw tables:

```csharp
/// <summary>
/// DbContext mapping the entities to the secure views. The views implement Row-Level Security (RLS) and 
/// Field-Level Security (FLS) based on the session variable 'app.current_user' that we set in the 
/// PostgresSecurityInterceptor.
/// </summary>
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Employee>().ToTable("vw_Employee_Secure");

        modelBuilder.Entity<BonusPayment>().ToTable("vw_BonusPayment_Secure");

        modelBuilder.Entity<Employee>()
            .HasMany(e => e.BonusPayments)
            .WithOne(b => b.Employee)
            .HasForeignKey(b => b.EmployeeId);
    }
}
```

And the `ODataController` now simply becomes:

```csharp
public class EmployeesController : ODataController
{
    private readonly AppDbContext _context;

    public EmployeesController(AppDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    [EnableQuery]
    public IActionResult Get()
    {
        return Ok(_context.Set<Employee>().AsNoTracking());
    }
}
```

And that's it!

## PostgreSQL Database ##

### Designing the Meta-Schema

Start by mapping organizational roles to technical permissions and attributes directly in the database.

```sql
CREATE TABLE IF NOT EXISTS "Role" (
    "RoleName" VARCHAR(50) PRIMARY KEY,
    "Description" TEXT
);

CREATE TABLE IF NOT EXISTS "Role_Permission" (
    "RoleName" VARCHAR(50) REFERENCES "Role"("RoleName"),
    "Permission" VARCHAR(100),
    PRIMARY KEY ("RoleName", "Permission")
);

CREATE TABLE IF NOT EXISTS "User_Attribute" (
    "UserId" VARCHAR(100), 
    "AttributeKey" VARCHAR(50), 
    "AttributeValue" VARCHAR(100), 
    PRIMARY KEY ("UserId", "AttributeKey", "AttributeValue")
);

CREATE TABLE IF NOT EXISTS "User_Role" (
    "UserId" VARCHAR(100),
    "RoleName" VARCHAR(50) REFERENCES "Role"("RoleName"),
    PRIMARY KEY ("UserId", "RoleName")
);
```

### Defining Helper Functions to check for permissions

Next we'll define 3 Helper functions to avoid repetitive SQL Queries. By marking them as 
`STABLE` they are very efficient and optimized:

```sql
CREATE OR REPLACE FUNCTION has_permission(p_permission VARCHAR) 
RETURNS BOOLEAN 
LANGUAGE sql STABLE 
AS $$
    SELECT EXISTS (
        SELECT 1
        FROM "User_Role" ur
        JOIN "Role_Permission" rp ON rp."RoleName" = ur."RoleName"
        WHERE ur."UserId" = COALESCE(current_setting('app.current_user', true), 'anonymous')
          AND rp."Permission" = p_permission
    );
$$;

CREATE OR REPLACE FUNCTION has_department_access(p_department VARCHAR) 
RETURNS BOOLEAN 
LANGUAGE sql STABLE 
AS $$
    SELECT EXISTS (
        SELECT 1
        FROM "User_Attribute" ua
        WHERE ua."UserId" = COALESCE(current_setting('app.current_user', true), 'anonymous')
          AND ua."AttributeKey" = 'Department'
          AND (ua."AttributeValue" = '*' OR ua."AttributeValue" = p_department)
    );
$$;

CREATE OR REPLACE FUNCTION mask_if_not(val anyelement, condition boolean) 
RETURNS anyelement 
LANGUAGE sql IMMUTABLE 
AS $$
    SELECT CASE WHEN condition THEN val ELSE NULL END;
$$;
```

### Creating the Application Tables and Secured Views 

In the example we have a HR application to manage employees and their Bonus Payments. A Standard User 
is not allowed to access the annual salary, bonus goal and is not allowed to list the list of bonus 
payments for an employee.

Only the Boss of HR is allowed to read the salary and see bonus payments.

```sql
CREATE TABLE IF NOT EXISTS "Employee" (
    "Id" SERIAL PRIMARY KEY,
    "Name" VARCHAR(200) NOT NULL,
    "Department" VARCHAR(100) NOT NULL,
    "AnnualSalary" DECIMAL(10,2),
    "BonusGoal" TEXT
);

CREATE TABLE IF NOT EXISTS "BonusPayment" (
    "Id" SERIAL PRIMARY KEY,
    "EmployeeId" INTEGER NOT NULL REFERENCES "Employee"("Id"),
    "Amount" DECIMAL(10,2) NOT NULL,
    "Reason" TEXT
);
```

We can then seed the database with some sample data:

```sql
INSERT INTO "Role" ("RoleName", "Description") VALUES 
    ('Standard_User', 'Normal Employee'),
    ('HR_Manager', 'Human Resources Manager')
ON CONFLICT ("RoleName") DO NOTHING;

INSERT INTO "Role_Permission" ("RoleName", "Permission") VALUES 
    ('Standard_User', 'Employee:Read_Public'),
    ('HR_Manager', 'Employee:Read_Public'),
    ('HR_Manager', 'Salary:Read')
ON CONFLICT ("RoleName", "Permission") DO NOTHING;

INSERT INTO "User_Attribute" ("UserId", "AttributeKey", "AttributeValue") VALUES 
    ('jane.smith@firma.de', 'Department', 'IT'),
    ('john.doe@firma.de', 'Department', 'Sales'),
    ('hr.boss@firma.de', 'Department', '*')
ON CONFLICT ("UserId", "AttributeKey", "AttributeValue") DO NOTHING;

INSERT INTO "User_Role" ("UserId", "RoleName") VALUES 
    ('jane.smith@firma.de', 'Standard_User'),
    ('john.doe@firma.de', 'Standard_User'),
    ('hr.boss@firma.de', 'HR_Manager')
ON CONFLICT ("UserId", "RoleName") DO NOTHING;
```

Now here is the trick: Never expose physical tables to the application. Always expose Views.

For the Employee access, we define a View `vw_Employee_Secure`. A user with the Permission `Employee:Read_Public` 
is allowed to see the rows it. But only a User with the Permission `Salary:Read` is allowed to see the annual 
salary and bonus goal.

```sql
CREATE OR REPLACE VIEW "vw_Employee_Secure" (
    "Id", 
    "Name", 
    "Department", 
    "AnnualSalary", 
    "BonusGoal"
) WITH (security_barrier = true) AS 
SELECT 
    e."Id", 
    e."Name", 
    e."Department",
    
    -- FIELD-LEVEL SECURITY
    mask_if_not(e."AnnualSalary", has_permission('Salary:Read') AND has_department_access(e."Department")),
    mask_if_not(e."BonusGoal", has_permission('Salary:Read'))

FROM "Employee" e
-- ROW-LEVEL SECURITY
WHERE has_permission('Employee:Read_Public');
```

For the Bonus Payments we need to be even more strict. You should only be able to see bonus 
payments, if you have the permission `Salary:Read` and you are the departments boss.

```sql
CREATE OR REPLACE VIEW "vw_BonusPayment_Secure" (
    "Id",
    "EmployeeId",
    "Amount",
    "Reason"
) WITH (security_barrier = true) AS
SELECT
    bp."Id",
    bp."EmployeeId",
    
    -- FIELD-LEVEL SECURITY: Only visible if the user has Salary:Read AND department access to the parent Employee
    mask_if_not(bp."Amount", has_permission('Salary:Read') AND has_department_access(e."Department")),
    mask_if_not(bp."Reason", has_permission('Salary:Read') AND has_department_access(e."Department"))

FROM "BonusPayment" bp
JOIN "Employee" e ON bp."EmployeeId" = e."Id"
-- ROW-LEVEL SECURITY: Bonus payments are completely filtered out if the user
-- does not have the "Salary:Read" permission OR lacks access to the employee's department.
WHERE has_permission('Salary:Read') AND has_department_access(e."Department");
```

Finally you should create a user `app_user`, which is used by the OData Service to connect with. Make sure, that 
this user has access to the Views only, so the application is not allowed to bypass the secured views, for example 
by running raw queries on the physical tables.

```sql
DO
$do$
BEGIN
   IF NOT EXISTS (
      SELECT FROM pg_catalog.pg_roles
      WHERE  rolname = 'app_user') THEN
      CREATE ROLE app_user LOGIN PASSWORD 'app_user';
   END IF;
END
$do$;

-- Grant basic schema access
GRANT USAGE ON SCHEMA public TO app_user;

-- Grant explicit SELECT access ONLY to the secure views
-- The user has NO access to the physical tables "Employee" or "BonusPayment"
GRANT SELECT ON "vw_Employee_Secure" TO app_user;
GRANT SELECT ON "vw_BonusPayment_Secure" TO app_user;

-- Grant read access to the ABAC metadata tables.
-- This is required because the helper functions (has_permission, etc.)
-- are executed in the context of the invoker (SECURITY INVOKER by default).
GRANT SELECT ON "Role" TO app_user;
GRANT SELECT ON "Role_Permission" TO app_user;
GRANT SELECT ON "User_Attribute" TO app_user;
GRANT SELECT ON "User_Role" TO app_user;
```

## SQL Server Database ##

### Designing the Meta-Schema

Start by mapping organizational roles to technical permissions and attributes directly in the database.

```sql
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[Role]') AND type in (N'U'))
BEGIN
    CREATE TABLE [dbo].[Role] (
        [RoleName] NVARCHAR(50) PRIMARY KEY,
        [Description] NVARCHAR(MAX)
    );
END

IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[Role_Permission]') AND type in (N'U'))
BEGIN
    CREATE TABLE [dbo].[Role_Permission] (
        [RoleName] NVARCHAR(50) FOREIGN KEY REFERENCES [dbo].[Role]([RoleName]),
        [Permission] NVARCHAR(100),
        PRIMARY KEY ([RoleName], [Permission])
    );
END

IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[User_Role]') AND type in (N'U'))
BEGIN
    CREATE TABLE [dbo].[User_Role] (
        [UserId] NVARCHAR(100),
        [RoleName] NVARCHAR(50) FOREIGN KEY REFERENCES [dbo].[Role]([RoleName]),
        PRIMARY KEY ([UserId], [RoleName])
    );
END

IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[User_Attribute]') AND type in (N'U'))
BEGIN
    CREATE TABLE [dbo].[User_Attribute] (
        [UserId] NVARCHAR(100), 
        [AttributeKey] NVARCHAR(50), 
        [AttributeValue] NVARCHAR(100), 
        PRIMARY KEY ([UserId], [AttributeKey], [AttributeValue])
    );
END

GO
```

### Defining Helper Functions to check for permissions

We define a set of helper functions to check, if the user is authorized and has the sufficient set 
of attributes to access a row or field. It looks a bit more verbose, than PostgreSQL, but I think 
it's still a nice implementation.

```sql
-- Check if the current session user has a specific permission
CREATE OR ALTER FUNCTION [dbo].[fn_HasPermission] (@Permission NVARCHAR(100))
RETURNS BIT
AS
BEGIN
    DECLARE @CurrentUserId NVARCHAR(100) = CAST(SESSION_CONTEXT(N'app.current_user') AS NVARCHAR(100));
    
    IF EXISTS (
        SELECT 1 FROM [dbo].[User_Role] ur
        JOIN [dbo].[Role_Permission] rp ON rp.[RoleName] = ur.[RoleName]
        WHERE ur.[UserId] = ISNULL(@CurrentUserId, 'anonymous')
          AND rp.[Permission] = @Permission
    )
        RETURN 1;

    RETURN 0;
END;
GO


-- Check if the current session user has a matching attribute (supports wildcards)
CREATE OR ALTER FUNCTION [dbo].[fn_HasAttrAccess] (@Key NVARCHAR(50), @Val NVARCHAR(100))
RETURNS BIT
AS
BEGIN
    DECLARE @CurrentUserId NVARCHAR(100) = CAST(SESSION_CONTEXT(N'app.current_user') AS NVARCHAR(100));
    
    IF EXISTS (
        SELECT 1 FROM [dbo].[User_Attribute] ua
        WHERE ua.[UserId] = ISNULL(@CurrentUserId, 'anonymous')
          AND ua.[AttributeKey] = @Key
          AND (ua.[AttributeValue] = @Val OR ua.[AttributeValue] = '*')
    )
        RETURN 1;

    RETURN 0;
END;
GO

-- Checks if a user is authorized based on a permission and optional attribute
CREATE OR ALTER FUNCTION [dbo].[fn_Auth] (
    @Permission NVARCHAR(100),
    @AttrKey    NVARCHAR(50) = NULL,
    @AttrValue  NVARCHAR(100) = NULL
)
RETURNS BIT
AS
BEGIN
    -- 1. Must have the base permission
    IF [dbo].[fn_HasPermission](@Permission) = 0
        RETURN 0;

    -- 2. If an attribute check is requested, must have matching attribute
    IF @AttrKey IS NOT NULL AND [dbo].[fn_HasAttrAccess](@AttrKey, @AttrValue) = 0
        RETURN 0;

    RETURN 1;
END;
GO
```

### Creating the Application Tables and Secured Views 

In the example we have a HR application to manage employees and their Bonus Payments. A Standard User 
is not allowed to access the annual salary, bonus goal and is not allowed to list the list of bonus 
payments for an employee.

Only the Boss of HR is allowed to read the salary and see bonus payments.

```sql
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[Employee]') AND type in (N'U'))
BEGIN
CREATE TABLE [dbo].[Employee] (
    [Id] INT IDENTITY(1,1) PRIMARY KEY,
    [Name] NVARCHAR(200) NOT NULL,
    [Department] NVARCHAR(100) NOT NULL,
    [AnnualSalary] DECIMAL(18, 2),
    [BonusGoal] NVARCHAR(2000),
    [Region] NVARCHAR(50) 
);
END

IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[BonusPayment]') AND type in (N'U'))
BEGIN
    CREATE TABLE [dbo].[BonusPayment] (
        [Id] INT IDENTITY(1,1) PRIMARY KEY,
        [EmployeeId] INT NOT NULL FOREIGN KEY REFERENCES [dbo].[Employee]([Id]),
        [Amount] DECIMAL(18, 2) NOT NULL,
        [Reason] NVARCHAR(2000)
    );
END
GO
```

We can then seed the database with some sample data:

```sql
-- Roles
IF NOT EXISTS (SELECT 1 FROM [dbo].[Role] WHERE [RoleName] = 'Standard_User')
BEGIN
    INSERT INTO [dbo].[Role] ([RoleName], [Description]) VALUES 
    ('Standard_User', 'Normal Employee'),
    ('HR_Manager', 'Human Resources Manager');
END

-- Permissions
IF NOT EXISTS (SELECT 1 FROM [dbo].[Role_Permission] WHERE [RoleName] = 'Standard_User' AND [Permission] = 'Employee:Read_Public')
BEGIN
    INSERT INTO [dbo].[Role_Permission] ([RoleName], [Permission]) VALUES 
    ('Standard_User', 'Employee:Read_Public'),
    ('HR_Manager', 'Employee:Read_Public'),
    ('HR_Manager', 'Salary:Read');
END

-- User Attributes (for the Department context)
IF NOT EXISTS (SELECT 1 FROM [dbo].[User_Attribute] WHERE [UserId] = 'jane.smith@firma.de')
BEGIN
    INSERT INTO [dbo].[User_Attribute] ([UserId], [AttributeKey], [AttributeValue]) VALUES 
    ('jane.smith@firma.de', 'Department', 'IT'),
    ('john.doe@firma.de', 'Department', 'Sales'),
    ('hr.boss@firma.de', 'Department', '*');
END

-- User Roles
IF NOT EXISTS (SELECT 1 FROM [dbo].[User_Role] WHERE [UserId] = 'jane.smith@firma.de')
BEGIN
    INSERT INTO [dbo].[User_Role] ([UserId], [RoleName]) VALUES 
    ('jane.smith@firma.de', 'Standard_User'),
    ('john.doe@firma.de', 'Standard_User'),
    ('hr.boss@firma.de', 'HR_Manager');
END

-- Employees
IF NOT EXISTS (SELECT 1 FROM [dbo].[Employee] WHERE [Id] IN (1, 2))
BEGIN
    SET IDENTITY_INSERT [dbo].[Employee] ON;
    INSERT INTO [dbo].[Employee] ([Id], [Name], [Department], [AnnualSalary], [BonusGoal], [Region]) VALUES 
    (1, 'Jane Smith', 'IT', 82000, 'System Uptime', 'North'),
    (2, 'John Doe', 'Sales', 65000, '10% Sales Increase', 'South');
    SET IDENTITY_INSERT [dbo].[Employee] OFF;
END

-- Bonus Payments
IF NOT EXISTS (SELECT 1 FROM [dbo].[BonusPayment] WHERE [Id] IN (1, 2))
BEGIN
    SET IDENTITY_INSERT [dbo].[BonusPayment] ON;
    INSERT INTO [dbo].[BonusPayment] ([Id], [EmployeeId], [Amount], [Reason]) VALUES 
    (1, 1, 5000.00, 'Excellent Uptime'),
    (2, 2, 3000.00, 'Q1 Target Met');
    SET IDENTITY_INSERT [dbo].[BonusPayment] OFF;
END
GO
```

Now here is the trick: Never expose physical tables to the application. Always expose Views.

For the Employee access, we define a View `vw_Employee_Secure`. A user with the Permission `Employee:Read_Public` 
is allowed to see the rows it. But only a User with the Permission `Salary:Read` is allowed to see the annual 
salary and bonus goal.

```sql
-- Secure view for Employees
CREATE OR ALTER VIEW [dbo].[vw_Employee_Secure]
AS
SELECT 
    e.[Id], 
    e.[Name], 
    e.[Department],
    
    -- FIELD-LEVEL SECURITY
    -- Mask AnnualSalary if lacks Salary:Read OR lack Department access
    IIF([dbo].[fn_Auth]('Salary:Read', 'Department', e.[Department]) = 1, e.[AnnualSalary], NULL) AS [AnnualSalary],
    
    -- Mask BonusGoal based on base permission
    IIF([dbo].[fn_Auth]('Salary:Read', NULL, NULL) = 1, e.[BonusGoal], NULL) AS [BonusGoal]

FROM [dbo].[Employee] e
-- ROW-LEVEL SECURITY
WHERE [dbo].[fn_Auth]('Employee:Read_Public', NULL, NULL) = 1;
GO
```

For the Bonus Payments we need to be even more strict. You should only be able to see bonus 
payments, if you have the permission `Salary:Read` and you are the departments boss.

```sql
-- Secure view for Bonus Payments with relationship security
CREATE OR ALTER VIEW [dbo].[vw_BonusPayment_Secure]
AS
SELECT
    bp.[Id],
    bp.[EmployeeId],
    
    -- FIELD-LEVEL SECURITY: Only visible if user has Salary:Read AND access to parent Employee's department
    IIF([dbo].[fn_Auth]('Salary:Read', 'Department', e.[Department]) = 1, bp.[Amount], NULL) AS [Amount],
    IIF([dbo].[fn_Auth]('Salary:Read', 'Department', e.[Department]) = 1, bp.[Reason], NULL) AS [Reason]

FROM [dbo].[BonusPayment] bp
JOIN [dbo].[Employee] e ON bp.[EmployeeId] = e.[Id]
-- ROW-LEVEL SECURITY: Bonus payments are filtered out if lacks Salary:Read OR lacks access to the employee's department.
WHERE [dbo].[fn_Auth]('Salary:Read', 'Department', e.[Department]) = 1;
GO
```

Finally you should create a user `ODataApiUser`, which is used by the OData Service to connect with. Make sure, that 
this user has access to the Views only, so the application is not allowed to bypass the secured views, for example 
by running raw queries on the physical tables.

```sql
-- Create a login and user for the API Application
IF NOT EXISTS (SELECT * FROM sys.server_principals WHERE name = 'ODataApiLogin')
BEGIN
    CREATE LOGIN [ODataApiLogin] WITH PASSWORD = 'YourStrong!AppPassword123';
END
GO

IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = 'ODataApiUser')
BEGIN
    CREATE USER [ODataApiUser] FOR LOGIN [ODataApiLogin];
END
GO

-- Grant EXECUTE on the security functions so the views can evaluate them
GRANT EXECUTE ON [dbo].[fn_HasPermission] TO [ODataApiUser];
GRANT EXECUTE ON [dbo].[fn_HasAttrAccess] TO [ODataApiUser];
GRANT EXECUTE ON [dbo].[fn_Auth] TO [ODataApiUser];

-- Grant SELECT ONLY on the secure views
GRANT SELECT ON [dbo].[vw_Employee_Secure] TO [ODataApiUser];
GRANT SELECT ON [dbo].[vw_BonusPayment_Secure] TO [ODataApiUser];

-- Explicitly ensure NO access to the raw tables
DENY SELECT ON [dbo].[Employee] TO [ODataApiUser];
DENY SELECT ON [dbo].[BonusPayment] TO [ODataApiUser];

-- Grant SELECT on metadata tables required for the functions to work
GRANT SELECT ON [dbo].[Role] TO [ODataApiUser];
GRANT SELECT ON [dbo].[Role_Permission] TO [ODataApiUser];
GRANT SELECT ON [dbo].[User_Role] TO [ODataApiUser];
GRANT SELECT ON [dbo].[User_Attribute] TO [ODataApiUser];
GO
```

## Docker Compose Files


We add a docker file and put the SQL Statements for both database into a `sql` folder:
```
./
│   docker-compose.yml
│
└───sql
    ├───postgres
    │       create-database.sql
    │
    └───sqlserver
            create-database.sql
```

And then we add two profiles for PostgreSQL and SQL Server:

```yml
version: '3.8'

services:
  postgres-db:
    image: postgres:18-alpine
    container_name: odata_security_postgres
    profiles:
      - postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ODataSecurityDemo
    ports:
      - "54320:5432"
    volumes:
      - ./sql/postgres:/docker-entrypoint-initdb.d/
    restart: unless-stopped
    
  sql-server-db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: odata_security_sqlserver
    profiles:
      - sqlserver
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=YourStrong!Pass123
      - MSSQL_PID=Developer
    ports:
      - "14330:1433"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "/opt/mssql-tools18/bin/sqlcmd", "-S", "localhost", "-U", "sa", "-P", "YourStrong!Pass123", "-C", "-Q", "SELECT 1"]
      interval: 10s
      timeout: 3s
      retries: 10
      start_period: 10s    
      
  sql-server-init:
    image: mcr.microsoft.com/mssql-tools:latest
    profiles:
      - sqlserver
    depends_on:
      sql-server-db:
        condition: service_healthy
    volumes:
      - ./sql/sqlserver/create-database.sql:/init.sql
    entrypoint: /opt/mssql-tools/bin/sqlcmd -S sql-server-db -U sa -P YourStrong!Pass123 -C -i /init.sql
```

The SQL Scripts are automatically executed for both systems.

There are two Docker Profiles `postgres` and `sqlserver`.

Let's boot into the Postgres one:

```
docker-compose --profile postgres up
```


## Seeing it in action


In Visual Studio select `ODataDemo.Postgres` in the Launch settings. This sets the `ASPNETCORE_ENVIRONMENT` 
environment variable to `Postgres`, so the correct connection string is resolved.

Now let's query the API using the `ODataSecurity.http` file coming with the Solution.

As a Standard Employee, we should only see the public properties and mask everything else:

```
### Simple request as a standard employee (Masking applies)
# Expectation: Returns records, but AnnualSalary and BonusGoal are NULL.
GET {{HostAddress}}/odata/Employees
X-User-Id: jane.smith@firma.de
```

And we get the employees data, with sensitive properties being masked:

```json
{
  "@odata.context": "https://localhost:5000/odata/$metadata#Employees",
  "value": [
    {
      "Id": 1,
      "Name": "Jane Smith",
      "Department": "IT",
      "AnnualSalary": null,
      "BonusGoal": null
    },
    {
      "Id": 2,
      "Name": "John Doe",
      "Department": "Sales",
      "AnnualSalary": null,
      "BonusGoal": null
    }
  ]
}
```

If we do the same as the HR Boss:

```
### Simple request as HR Boss (Masking does NOT apply)
# Expectation: Returns records, with AnnualSalary and BonusGoal
GET {{HostAddress}}/odata/Employees
X-User-Id: hr.boss@firma.de
```

The Field-Level Security does not apply and we get:

```json
{
  "@odata.context": "https://localhost:5000/odata/$metadata#Employees",
  "value": [
    {
      "Id": 1,
      "Name": "Jane Smith",
      "Department": "IT",
      "AnnualSalary": 82000.00,
      "BonusGoal": "System Uptime"
    },
    {
      "Id": 2,
      "Name": "John Doe",
      "Department": "Sales",
      "AnnualSalary": 65000.00,
      "BonusGoal": "10% Sales Increase"
    }
  ]
}
```

### Does it work with $expand? ###

Now let's check the OData `$expand` feature.

An Employee can have a list of Bonus Payments having been paid out. These bonus 
payments shouldn't be available to standard employees, so we check if a standard 
user can `$expand` the property:

```
### Simple request as a standard employee and $expand (Masking applies)
# Expectation: Returns records, but AnnualSalary and BonusGoal are NULL and no Bonus Payments.
GET {{HostAddress}}/odata/Employees?$expand=BonusPayments
X-User-Id: jane.smith@firma.de
```

And we'll see, that the `BonusPayments` property is empty:

```
{
  "@odata.context": "https://localhost:5000/odata/$metadata#Employees(BonusPayments())",
  "value": [
    {
      "Id": 1,
      "Name": "Jane Smith",
      "Department": "IT",
      "AnnualSalary": null,
      "BonusGoal": null,
      "BonusPayments": []
    },
    {
      "Id": 2,
      "Name": "John Doe",
      "Department": "Sales",
      "AnnualSalary": null,
      "BonusGoal": null,
      "BonusPayments": []
    }
  ]
}
```

The HR Boss on the other hand:

```
### Request with $expand (Securing related tables)
# Expectation: EF Core generates a LEFT JOIN. Since the database connection already knows the User-ID,
# the Row-Level Security and Field-Level Security are automatically applied to
# the view 'vw_BonusPayment_Secure'.
GET {{HostAddress}}/odata/Employees?$expand=BonusPayments
X-User-Id: hr.boss@firma.de
```

Is authorized to see all Bonus Payments for his employees:

```json
{
  "@odata.context": "https://localhost:5000/odata/$metadata#Employees(BonusPayments())",
  "value": [
    {
      "Id": 1,
      "Name": "Jane Smith",
      "Department": "IT",
      "AnnualSalary": 82000.00,
      "BonusGoal": "System Uptime",
      "BonusPayments": [
        {
          "Id": 1,
          "EmployeeId": 1,
          "Amount": 5000.00,
          "Reason": "Excellent Uptime"
        }
      ]
    },
    {
      "Id": 2,
      "Name": "John Doe",
      "Department": "Sales",
      "AnnualSalary": 65000.00,
      "BonusGoal": "10% Sales Increase",
      "BonusPayments": [
        {
          "Id": 2,
          "EmployeeId": 2,
          "Amount": 3000.00,
          "Reason": "Q1 Target Met"
        }
      ]
    }
  ]
}
```

## Conclusion ##

The selling point of this solution is, that it's Fail-Closed by design. ORM mistakes or raw SQL cannot bypass 
the rules. Your data is protected at its core, and it doesn't matter which application is querying it. You are 
safe wether it's OData, or an ORM.

While everything looks nice for this simple example, it's probably getting hairy as soon as the requirements 
and rules get more complex. I will see how it turns out to work in larger projects and then adjust.

Anyways, I think it's a good start towards a flexible authorization implementation!
