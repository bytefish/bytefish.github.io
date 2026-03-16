title: Introducing the new PostgreSQLCopyHelper 3.0.0 API
date: 2026-03-16 14:21
tags: dotnet, postgres, bulk
category: dotnet
slug: postgresqlcopyhelper_3_api
author: Philipp Wagner
summary: Introducing the new PostgreSQLCopyHelper 3.0.0 API.

After re-architecting the PgBulkInsert library, it's now time to also update its 
.NET counterpart, PostgreSQLCopyHelper, because it suffered from just the same 
design flaws as PgBulkInsert. 

By putting the application language in the middle, we had a bloated API surface and 
it was hard to use Postgres rich type system. This again was a modeling error on my 
side, that existed from the very start. 

I think it's time to flip the mental model around and turn the library into a Database-first centric approach.

All code can be found in a Git Repository at:

* [https://github.com/bytefish/PostgreSQLCopyHelper](https://github.com/bytefish/PostgreSQLCopyHelper)

## Introducing the 3.0.0 API ##

The new API strictly separates the **What** (Structure and Mapping) from the **How** (Execution and I/O). It flips 
the mental model by having a Database-first API, which solves a lot of problem with the previous API and allows for 
composing complex types more easily. 

## Quick Start ##

### 1. Define your Data Model ###

The library works perfectly with modern C# record types, structs, or traditional classes.

```csharp
public record UserSession(
    Guid Id,
    string? UserAgent,   // Nullable Reference Type
    DateTime CreatedAt,  // Precise Timestamp
    int[] Tags,          // Array
    NpgsqlRange<int> ActiveRange // Native Range Support
);
```

### 2. Define your Mapping (Stateless & Thread-Safe) ###

The `PgMapper<T>` is the heart of the library. It is completely stateless after configuration and should be 
instantiated only once (e.g., as a `static readonly` field or Singleton).

```csharp
private static readonly PgMapper<UserSession> SessionMapper = 
    new PgMapper<UserSession>("public", "user_sessions")
        .Map("id", PostgresTypes.Uuid, x => x.Id)
                
        // SAFE STRINGS: Strips invalid \u0000 characters to prevent pipeline crashes
        .Map("user_agent", PostgresTypes.Text.NullCharacterHandling(""), x => x.UserAgent)
        
        // TIME TYPES: Native support for Npgsql's DateTime semantics
        .Map("created_at", PostgresTypes.TimestampTz, x => x.CreatedAt)
        
        // ARRAYS: Compose base types natively
        .Map("tags", PostgresTypes.Array(PostgresTypes.Integer), x => x.Tags)

        // RANGES: Native Postgres range types
        .Map("active_range", PostgresTypes.IntegerRange, x => x.ActiveRange);
```

### 3. Execute the Bulk Insert ###

The `PgBulkWriter<T>` is a lightweight, transient executor that takes your mapper and streams the data to the 
database using `ValueTask` and asynchronous I/O.

```csharp
public async Task SaveSessionsAsync(NpgsqlConnection conn, List<UserSession> sessionList)
{
    var writer = new PgBulkWriter<UserSession>(SessionMapper);
    
    ulong insertedCount = await writer.SaveAllAsync(conn, sessionList);

    Console.WriteLine($"Successfully inserted {insertedCount} sessions.");
}
```

## Streaming and Lazy Evaluation ##

One of the key strengths of the `SaveAllAsync` method is that it accepts an `IEnumerable<T>`. This means you 
are never forced to load your entire dataset into memory.

If you are yielding data from a stream, a file parser, or another database, the writer will pull the data lazily:

```csharp
IEnumerable<UserSession> massiveDataStream = ReadMassiveDataFromCsv();

// Data is streamed directly to PostgreSQL on-the-fly. Memory consumption remains flat.
await writer.SaveAllAsync(connection, massiveDataStream);
```

## Mastering the Fluent API ##

The API is designed around `PostgresTypes`. This class serves as your single entry point for all PostgreSQL data types.

When you map a property, the compiler automatically detects if your `struct` is nullable (`int?`) or non-nullable (`int`):

```csharp
// The compiler routes this to the high-performance, non-allocating path
.Map("mandatory_id", PostgresTypes.Integer, x => x.Id) // Id is 'int'

// The compiler routes this to the null-safe path automatically!
.Map("optional_bonus", PostgresTypes.Integer, x => x.Bonus) // Bonus is 'int?'
```

## Advanced Type Mapping ##

### Arrays and Lists ###

You can compose any base type into an array or list using the `Array()` or `List()` composition functions:

```csharp
// Maps a C# List<string> to a Postgres text[]
.Map("nicknames", PostgresTypes.List(PostgresTypes.Text), x => x.Nicknames)

// Maps a C# int[] to a Postgres int4[]
.Map("scores", PostgresTypes.Array(PostgresTypes.Integer), x => x.Scores)
```

### Ranges ###

PostgreSQL's powerful range types are fully supported via `NpgsqlRange<T>`:

```csharp
// Using predefined common ranges
.Map("age_limit", PostgresTypes.IntegerRange, x => x.AgeRange)

// Composing custom ranges dynamically (e.g. for custom PostgreSql Range Types)
.Map("custom_range", PostgresTypes.Range(PostgresTypes.DoublePrecision), x => x.CustomRange)
```

## Conclusion ##

Re-architecting an established library from the ground up is never an easy decision and I didn't 
take it lightly. But moving PostgreSQLCopyHelper to a more functional, Database-first design was 
the only way to fix its inherent design flaws and maybe keep it alive for the decade to come. 

I know, that migrating takes a lot of effort. So if you need help migrating your existing mappings, let me know.

Finally a **big thank you** to everyone who has used, tested, and contributed to PostgreSQLCopyHelper over the past decade. 

The 3.0.0 release is out now on NuGet.