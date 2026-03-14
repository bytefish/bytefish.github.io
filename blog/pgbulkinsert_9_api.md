title: Introducing the new PgBulkInsert 9.0.0 API
date: 2026-03-14 17:21
tags: java, postgres, pgbulkinsert
category: java
slug: pgbulkinsert_9_api
author: Philipp Wagner
summary: Introducing the new PgBulkInsert 9.0.0 API.

PgBulkInsert was written in 2016 and I am happy it has gained some traction in 
the past 10 years. There are quite a few companies using it "under the hood" and 
it probably eats terabytes day by day.

So why did I redesign the API and introduced a major breaking change? Mainly because 
the PgBulkInsert API never sat right with me, and I think it was fundamentally broken 
ever since being released.

The API didn't need a minor refactoring. 

It needed a completely inverted mental model.

All code can be found in a Git repository at:

* [https://github.com/bytefish/PgBulkInsert](https://github.com/bytefish/PgBulkInsert)


## Table of contents ##

[TOC]

## The Major Design Flaw: A Java-centric approach ##

The API always had a fatal design flaw: It was **Java-centric**. Putting the 
application language (Java) in the center, when working with a database, is 
a modeling error. 

###  API Surface Explosion ###

Think about the problem at hand: If the API's starting point is the Java type, how 
does it handle the fact that a single Java `String` can be mapped to half a dozen 
different PostgreSQL types?

The only workaround in the old architecture? Brute-force it by adding a specific 
mapping method for every single database type. We couldn't just have a `mapString()` 
method. 

To support the database properly, the library had to expose `mapText()`, `mapVarchar()`, 
`mapJsonb()`, `mapUuid()`, `mapXml()`, ... and that's just a single Java type. We 
ended up with a god-class of dozens of `mapX` methods.

### Inheritance Over Composition ###

At work I am preaching **Composition over Inheritance** to each and everyone. And 
in PgBulkInsert? It heavily relied on class Inheritance and forced you to write an 
`BaseValueHandler<T>`. 

That meant: If you wanted to support a new type, you had to write a brand-new 
class. There was no way to compose data types and bulk copy something like a 
`int4range[]` to the database.

You probably had to write something like an `IntegerRangeArrayValueHandler`, which 
comes suspicously close to a meme-esque `FactoryFactory` Java Enterprise Code. 

This is wrong. 

We need to compose types to echo the rich and flexible Postgres type-system.

### Issues, Issues, ...  ###

I could go on and talk about the illusion of type-safety, but I cut it here ...

## The Design Fix: A Postgres-centric approach ##

The new API flips the mental model.

The entry point is now anchored to the database schema. The API asks: "What does your PostgreSQL column expect?" 

That means you start the mapping by defining the truth of the database: `PostgresTypes.INT4` or `PostgresTypes.JSONB`. By 
putting the Postgres type in the middle, the API can strictly dictate which Java types are legally allowed to fulfill 
that contract.

### Anatomy of the New API ###

The whole new API is rebuilt around a few core components. If you are migrating to 9.0.0, these are the new tools in your belt:

#### 1. `PostgresTypes`: The Starting Point ####

This is the new heart of the library. `PostgresTypes` is a registry containing every supported PostgreSQL data type—from standard 
`INT4` and `TEXT` to complex types like `JSONB`, `TSRANGE` and multidimensional arrays.

Instead of guessing which mapping method to use, you always start here. When you type `PostgresTypes.INT4.`, your IDE immediately 
understands the binary contract and offers you the exact functional methods allowed for that specific PostgreSQL type.

#### 2. The Extraction Methods (`.primitive()`, `.boxed()`, `.from()`) ####

Once you select `INT4` as your Postgres type, you tell the library how to extract that data from your Java entity using functional 
Method References. This forces explicit null-safety and unlocks our zero-allocation performance:

* `.primitive(...)`: The high-performance path. If your Java object uses a primitive int, long, or double, the value goes straight into the binary stream. No heap allocation, no autoboxing, no garbage collection overhead.
* `.boxed(...)`: The null-safe path. If your database column is nullable and your Java POJO uses an Integer or Long wrapper, you explicitly declare it here.
* `.from(...)`: The standard extraction path for complex objects like String, UUID, Instant, or custom geometries.

#### 3. `PgMapper`: The Lean Orchestrator ####

With this change I was able to delete 50+ mapping methods from the old API. The new `PgMapper` is now entirely 
stateless, thread-safe and lean. It now relies on exactly one method: `.map(columnName, typeDefinition)`. And 
because the mapping action is now decoupled from the type itself, the `PgMapper` is infinitely extensible. 

You can map a basic string (`PostgresTypes.TEXT.from(...)`) or a complex nested array (`PostgresTypes.array(PostgresTypes.INT4RANGE).from(...)`) 
using the exact same API surface.

#### 4. `PgBulkWriter`: The Execution Engine ####

In the new architecture, the *What* (your mapping definition) is strictly separated from the *How* (the database execution). 

Once your `PgMapper` is defined, you pass it to the `PgBulkWriter`. It manages the low-level `COPY BINARY` protocol, allocating I/O buffers and 
streaming your Iterable or Stream directly over the JDBC connection.

### From Runtime Explosions to Compile-Time Guarantees ###

Let's look at how this changes your daily workflow.

If we previously had to map a `User` class we probably wrote something like this:

```java
public class UserMapping extends AbstractMapping<User>
{
    public UserMapping() {
        // What if "id" was actually a UUID in the DB, this compiled 
        // fine but crashed at runtime.
        mapText("id", User::getId);
        
        // This also compiled perfectly, but caused huge GC pressure 
        // due to autoboxing, if you accidentally used the wrong 
        // map Method.
        mapInteger("visits", User::getVisits);
        
    }
}
```

Now in the new API there is only one `.map()` method. You declare the Postgres type first, and 
the compiler restricts what Java types you can pass to it.

```java
PgMapper<User> mapper = PgMapper.forClass(User.class)
    // The compiler enforces that the getter must match the Postgres UUID protocol
    .map("id", PostgresTypes.UUID.from(User::getId))
    
    // Explicitly mapping to VARCHAR
    .map("username", PostgresTypes.VARCHAR.from(User::getUsername))
    
    // ZERO ALLOCATION: The IDE only exposes primitive functional interfaces for INT4
    .map("visits", PostgresTypes.INT4.primitive(User::getVisits))
    
    // Complex types are natively supported via composition
    .map("active_hours", PostgresTypes.array(PostgresTypes.INT4RANGE).from(User::getActiveHours));
```

If you accidentally try to map a `String` getter (e.g., `User::getUsername`) into a `PostgresTypes.INT4` column, 
your Java code will physically not compile. The IDE will flag it with a red underline immediately.

## Conclusion ##

Re-architecting an established library from the ground up is never an easy decision. 

But moving PgBulkInsert to a more functional, Database-first design was the only way to fix 
its inherent design flaws and maybe keep it alive for the decade to come. 

I know, that migrating takes a lot of effort. So if you need help migrating your existing mappings, let me know.

Finally a **big thank you** to everyone who has used, tested, and contributed to PgBulkInsert over the past decade. 

The 9.0.0 release is out now on Maven Central.