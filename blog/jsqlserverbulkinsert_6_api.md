title: Introducing the new JSqlServerBulkInsert 6.0.0 API
date: 2026-03-20 23:49
tags: java, sqlserver, jsqlserverbulkinsert
category: java
slug: jsqlserverbulkinsert_6_api
author: Philipp Wagner
summary: Introducing the new JSqlServerBulkInsert 6.0.0 API.

After re-architecting the PgBulkInsert library, it is now time to update its SQL Server counterpart, JSqlServerBulkInsert. Just like the 
PgBulkInsert API, the previous versions of JSqlServerBulkInsert suffered from significant design flaws caused by an overly Java-centric 
perspective.

By putting the application language in the middle, we ended up with a bloated API surface that struggled to map correctly to SQL Server's 
specific type system. This was a modeling error on my part that had existed from the very beginning of the library.

With Version 6.0.0, I am flipping the mental model to a Database-first centric approach.

All code can be found in the Git Repository at:

* [https://github.com/bytefish/JSqlServerBulkInsert](https://github.com/bytefish/JSqlServerBulkInsert)

## Setup ##

JSqlServerBulkInsert is designed for Java 17+ and the Microsoft JDBC Driver for SQL Server (version 12.6.5.jre11 or higher).

Add the following dependency to your pom.xml:

```xml
<dependency>
    <groupId>de.bytefish</groupId>
    <artifactId>jsqlserverbulkinsert</artifactId>
    <version>6.0.0</version>
</dependency>
```

## Introducing the 6.0.0 API ##

The new API strictly separates the **What** (Structure and Mapping) from the **How** (Execution and I/O). 

By adopting a Database-first API, we solve the mapping ambiguities of previous versions and make it much 
easier to handle SQL Server-specific requirements like precision metadata and reserved keywords.

## Quick Start ##

### 1. Define your Data Model ###

The library works perfectly with modern Java record types and traditional POJOs.

```java
public record SensorData(
    UUID id,
    String name,
    int temperature,
    String payload,      // Large content
    OffsetDateTime time  // Precise Timestamp
) {}
```

### 2. Define your Mapping (Stateless & Thread-Safe) ###

The `SqlServerMapper<T>` is the heart of the library. It is completely stateless after configuration and should be 
instantiated only once (e.g., as a `static final` field or a Singleton).

```java
private static final SqlServerMapper<SensorData> MAPPER = 
    SqlServerMapper.forClass(SensorData.class)
        .map("Id", SqlServerTypes.UNIQUEIDENTIFIER.from(SensorData::id))
        
        // BRACKETING: Automatically wraps column names in [] to prevent keyword conflicts
        .map("Name", SqlServerTypes.NVARCHAR.from(SensorData::name))
        
        // PRIMITIVE SUPPORT: Specialized functional interfaces for direct mapping
        .map("Temperature", SqlServerTypes.INT.primitive(SensorData::temperature))
        
        // MAX TYPES: Easily map to NVARCHAR(MAX) or VARBINARY(MAX)
        .map("Payload", SqlServerTypes.NVARCHAR.max().from(SensorData::payload))
        
        // TIME TYPES: Native support for java.time with optimized binary transfer
        .map("Timestamp", SqlServerTypes.DATETIMEOFFSET.offsetDateTime(SensorData::time));
```

### 3. Execute the Bulk Insert ###

The `SqlServerBulkWriter<T>` is a lightweight, transient executor that takes your mapper and streams the 
data to the database using the native `ISQLServerBulkData` API.

```java
public void saveAll(Connection conn, List<SensorData> data) {
    var writer = new SqlServerBulkWriter<>(MAPPER)
        .withBatchSize(1000)
        .withTableLock(true);
    
    BulkInsertResult result = writer.saveAll(conn, "dbo", "Sensors", data);
    
    if (result.success()) {
        System.out.println("Successfully inserted " + result.rowsAffected() + " rows.");
    }
}
```

## Streaming and Lazy Evaluation ##

The `saveAll` method accepts an `Iterable<T>`. This ensures you are never forced to load your entire dataset into memory at once.

If you are yielding data from a file parser, a stream, or a reactive source, the writer will pull the data lazily, keeping your 
memory footprint flat regardless of the dataset size.

```java
Iterable<SensorData> massiveDataStream = readMassiveCsvFile();

// Data is streamed directly to SQL Server on-the-fly.
writer.saveAll(connection, "dbo", "MassiveTable", massiveDataStream);
```

## Mastering the SQL-Centric API ##

The API is designed around `SqlServerTypes`. This class serves as your single entry point for all SQL Server data types, 
ensuring that the library provides the exact metadata (precision and scale) the database expects.

### Bracketing by Default ###

SQL Server developers often run into syntax errors when using reserved keywords like `DATE`, `TIME`, or `USER` as column names. JSqlServerBulkInsert 6.0.0 
introduces automatic bracketing: every schema, table, and column name is automatically wrapped in brackets.

### Optimized Binary Transfer for Time API ###

One of the biggest pain points in the old API was the conversion of `java.time` objects. The driver would often fall back to string-based transfers, 
leading to "Conversion failed" errors. By mandating specific precision metadata for temporal types, we force the JDBC driver to use the native 
binary transfer path.

## Conclusion ##

Re-architecting an established library is a difficult decision, but moving JSqlServerBulkInsert to a more functional, Database-first design was the 
only way to fix its inherent design flaws and ensure long-term stability. 

I know that migrating existing code requires effort. And if you encounter issues along the way, feel free to ask in the issue tracker.

Finally thank you to everyone who has supported these libraries over the years.

The 6.0.0 release is out now.