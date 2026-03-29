title: Beyond Roles: A Pragmatic ABAC Architecture for OData
date: 2026-03-29 14:50
tags: csharp, dotnet, odata
category: dotnet
slug: aspnetcore_odata_abac
author: Philipp Wagner
summary: Beyond Roles: A Pragmatic ABAC Architecture for OData

OData is a superpower for frontend developers. By exposing a single endpoint, you give clients the ultimate 
flexibility to query exactly what they need using `$filter`, shape the payload with `$select`, and fetch 
related data via `$expand`.

But this incredible power is a double-edged sword. When the client can query basically whatever they want, 
traditional authorization models start to crumble. Role-Based Access Control (RBAC) is far too coarse for 
this level of dynamic querying. 

In this article we'll take a look at implementing a pragmatic Attribute-based Access Control Architecture 
using PostgreSQL.

The code is available in a Git repository at:

* [https://github.com/bytefish/ODataSecurity](https://github.com/bytefish/ODataSecurity)

## Table of contents ##

[TOC]

## The Problem: Role Explosion ##

In most enterprise systems, we start with basic roles: `Admin`, `Manager`, `Employee`. This 
works fine for simple APIs. But OData allows users to query specific rows and columns across 
the entire database.

If you try to secure this with roles alone, you quickly hit **Role Explosion**. Imagine you 
have 50 departments and 10 countries. If a manager should only see data for their specific 
department in their specific country, you cannot practically create 500 different roles.

You'll need **Attributes**. You need a way to say: "You have the *role* of Manager, but your access is 
bounded by the *attributes* `Department: Sales` and `Country: Germany`."

## Why Application-Layer Proxies Fall Short ##

When we try to solve this fine-grained filtering in the application code, for example via a middleware, 
we usually hit two major walls:

### The "Shadow Database" Problem 

To secure a dynamic query in the application layer, you have to perfectly mimic the database's filtering logic. If 
your code and the SQL engine disagree on even a tiny edge case, like a null-check or a nested join, you've created 
a security hole. 

You essentially end up trying to rebuild a simplified version of your database engine in your C# code.

### The Audit Gap 

When security logic is hidden inside complex code blocks, interceptors, or dynamic query builders, it becomes 
a "black box". If an auditor asks, "What exactly can Jane Smith see?", you can't just give them a clear 
answer. 

You have to debug the application and trace execution paths. There is no single "Source of Truth" to verify.

## The Organizational Wall (Why Centralized Engines Fail) ##

Modern frameworks like OpenFGA or Open Policy Agent (OPA) are excellent in a vacuum, but 
in a standard enterprise, they often become a dead-end.

### A Choice Beyond the Developer

Deploying a centralized authorization engine is rarely just a "dev choice." It is an architectural commitment 
that requires **massive organizational buy-in**. These engines act as a global source of truth. If you 
can't get the entire company, including IT, Security, and Compliance to adopt the engine as the unified 
standard, you end up managing an isolated, highly complex "security island" just for your one project.

### The Gravity of Existing Infrastructure

Most companies have spent decades building their identity infrastructure around **Active Directory (AD)** or 
Entra ID. This is where users, departments, and basic groups live. Introducing a centralized engine requires 
you to synchronize this massive, shifting state into a new format (e.g. relationship tuples).

### The Filtering and Pagination Trap

OData is built for efficient data retrieval. When a user asks for "the first 50 employees I can see, sorted by 
hire date," a centralized engine hits a technical wall. Since the engine is outside the database, you can't 
easily `JOIN` permissions with your data. 

To solve this, you either:

**1. Fetch all IDs in your Database and Check** 

You fetch all IDs in your database and check them against the engine. This is an N+1 performance disaster.

**2. Fetch all IDs in your Centralized Engine**

You fetch all IDs a user is authorized for in your centralized engine. But listing all available objects is 
often very expensive and breaks these engines, when given lots of concurrent requests.

**3. Flatten your data** 

You could flatten your data and replicate the engine's relationships back into your SQL tables just so you 
can perform a join. This completely defeats the purpose of a "decoupled" engine, and introduces a whole new 
set of problems. 

## Why not native PostgreSQL RLS? ##

Since we are using PostgreSQL, the native Row-Level Security (RLS) feature seems like the logical 
choice. However, I found it a poor fit for two reasons:

**1. The Masking Challenge:** 

RLS is designed to hide or show rows entirely. It is remarkably difficult to implement "Masking", where a user sees the 
row but a specific column like `Salary` is returned as `NULL`, just by using RLS Policies.

**2. The Danger of Implicit Logic:** 

RLS is invisible. When a developer writes `SELECT * FROM Employees`, they expect to see the data. If the database 
silently drops rows based on a hidden policy, debugging becomes a nightmare. There is no visual difference in the 
code between a "full" query and a "secured" query. 

This lack of explicitness makes it incredibly hard for a team to reason about why certain data is missing or why a join is behaving unexpectedly.

## The Conceptual Solution: Security as a Database Session ##

The most pragmatic approach is to move the security boundary to where the data actually lives: the database session.

Instead of treating security as a secondary check in your C# code or an external engine, treat it as a **hardened boundary** 
within the database itself:

**1. Identify:** 

Extract the user identity and attributes from the incoming token or header.

**2. Inject:** 

Use an Interceptor to tell the database session exactly **who** is logged in via a session variable (`app.current_user`).

**3. Restrict:** 

The application queries **Secured Views** instead of physical tables. This makes the security context explicit: if you want secured data, you query the view.

## The Implementation ##

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

Now here is the trick: Never expose physical tables to the application. Always expose Views. This also 
solves the **auditability nightmare**: You can impersonate any user in a SQL tool and see exactly what they 
see.

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

That's it.

## ASP.NET Core OData API ##

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

You'll then write a `DbConnectionInterceptor` to pass the user into the PostgreSQL session as 
the `app.current_user` variable:

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
/// DbContext mapping the entities to the secure PostgreSQL views. The views implement Row-Level Security (RLS) and 
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

## Seeing it in action ##

Let's query the API as a Standard Employee, which should only return the 
public properties and mask everything else:

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

The Field-Level Security does not apply and we get

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

### The Trade-offs ###

Let's take a look at the Pro and Cons of the solution applied in this article.

**The Pros:**

- **Leak-Proof:** Fail-Closed by design. ORM mistakes or raw SQL cannot bypass the rules.
- **Auditability:** Permissions are deterministic. You can prove what a user sees with a simple SQL query.
- **Performance:** Filtering happens at the engine level, not in memory.

**The Cons:**

- **SQL Overhead:** You have to manage Views and Functions alongside your migrations.
- **Sync Responsibility:** You must sync identity attributes (from AD/Entra) into the database meta-tables.
- **Database Lock-in:** The logic is specific to your database provider (e.g., PostgreSQL).

### Closing Words ###

It was only a few lines of code in PostgreSQL to implement Attribute-based Access Control. I think it's a 
lean implementation, that everyone in a team is able to understand. You'll effictively avoid the Role Explosion, 
simplify your application code, and build a system that is fundamentally secure by default. 

Sometimes the most pragmatic solution isn't a new external engine, it's using your database engine.
