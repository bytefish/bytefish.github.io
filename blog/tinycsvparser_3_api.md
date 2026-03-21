title: Introducing the new TinyCsvParser 3.0.0 API
date: 2026-03-21 11:18
tags: csharp, dotnet, csv
category: dotnet
slug: tinycsvparser_3_api
author: Philipp Wagner
summary: Introducing the new TinyCsvParser 3.0.0 API.

Something I've been working on the past weeks is a complete redesign of TinyCsvParser, which is 
my Open Source library for CSV Parsing with .NET. It's a decade old, born when .NET Core was at 
version 1.4 and it shows its age.

The CSV Parser allocates way too much memory, probably making Garbage Collectors all around the 
world sweat. It wasn't able to handle multi-line CSV at all, and the Result and Error Handling 
always felt off.

Thanks to [@Externius](https://github.com/Externius) for supporting, helping me and igniting my interest.

## Table of contents ##

[TOC]

## Why all these Breaking Changes? ##

Re-architecting an established library from the ground up is never an easy decision, and I didn't 
make it lightly. But moving TinyCsvParser to a modern and functional API maybe keeps the project 
alive for the decade to come. 

I know, that migrating takes a lot of effort, especially with these kind of breaking changes. So if 
you need help migrating your existing mappings, let me know.

## What is TinyCsvParser? ##

TinyCsvParser is a high-performance CSV parsing library for .NET. This documentation explains the usage, 
configuration, and extensibility of the library through practical examples.

> Upgrading from a previous version? Check out the [Migration Guide from 2.x to 3.x](#8-migration-from-2x-to-3x)

## 1. Setup ##

To include TinyCsvParser in your project, install the NuGet package using the .NET CLI:

```powershell
dotnet add package TinyCsvParser
```

Alternatively, you can use the NuGet Package Manager in Visual Studio:

```
Install-Package TinyCsvParser
```

## 2. Quick Start ##

To parse a CSV file, you need a target model and a mapping definition.

### 2.1 The Model ###

The target of your parsing operation should be a class with a parameterless constructor.

```csharp
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

### 2.2 The Mapping ###

Create a class inheriting from `CsvMapping<T>` and define the relationship between CSV columns and model properties.

**Option A: Mapping by Header Name (Recommended)**

This approach is flexible as it doesn't depend on the order of columns in the CSV file. The 
parser automatically resolves the names to indices.

```csharp
public class PersonMapping : CsvMapping<Person>
{
    public PersonMapping()
    {
        MapProperty("ID", x => x.Id);
        MapProperty("Full Name", x => x.Name);
    }
}
```

**Option B: Mapping by Index**

Use this for files without headers or for maximum performance.

```csharp
public class PersonMappingByIndex : CsvMapping<Person>
{
    public PersonMappingByIndex()
    {
        // 0-based index: ID is column 0, Name is column 1
        MapProperty(0, x => x.Id);
        MapProperty(1, x => x.Name);
    }
}
```

### 2.3 Handling Quotes ###

TinyCsvParser automatically handles fields wrapped in quotes. This is essential when your data 
or your header names contain the delimiter character or line breaks.

```csharp
// Example CSV: "ID";"Full Name"
// The parser strips the quotes automatically. 
// You map using the clean name:
MapProperty("Full Name", x => x.Name);
```

Quoted fields can contain the delimiter (e.g., "Doe, John") or even escaped quotes (e.g., "The ""Great"" Gatsby"), which 
the parser resolves before passing the value to the mapping.


## 3. Configuring and Running the Parser ##

The CsvParser is the central engine. It is stateless and can be reused for multiple parsing operations.

### 3.1 Setup the Parser ###

First, you combine your `CsvOptions` and your `CsvMapping` to create the parser instance.

```csharp
// 1. Define the technical format
CsvOptions options = new(
    Delimiter: ';', 
    QuoteChar: '"', 
    EscapeChar: '"', 
    SkipHeader: true,
    CommentCharacter: '#'
);

// 2. Instantiate your mapping logic
PersonMapping mapping = new();

// 3. Create the parser (Stateless and reusable)
CsvParser<Person> parser = new(options, mapping);
```

### 3.2 Execute the Parsing ###

The parser supports reading from strings, streams, or files. Crucially, the parsing process uses deferred execution (lazy loading). This means the file is not loaded into memory all at once; it is read line-by-line as you iterate through the results.

```csharp
// Calling ReadFromFile does NOT start the parsing yet.
// It returns an Enumerable that waits for a foreach loop.
IEnumerable<CsvMappingResult<Person>> results = parser.ReadFromFile("data.csv");

// The actual parsing happens here, one record at a time.
foreach (CsvMappingResult<Person> result in results)
{
    // Every result encapsulates Success, Error, or Comment states.
    if (result.IsSuccess)
    {
        Person person = result.Result;
        Console.WriteLine($"Parsed: {person.Name}");
    }
}
```

## 4. Core Concept: Row and Line Tracking ##

TinyCsvParser distinguishes between two types of indices. This distinction is necessary because CSV 
files often deviate from a simple "one line equals one record" structure.

### 4.1 LineNumber vs. RecordIndex ###

* `LineNumber`: Refers to the physical line in the source file (1-based).
* `RecordIndex`: Refers to the logical data entity (0-based).

### 4.2 Reasoning for the Distinction ###

* **Quoted Newlines**: If a CSV field contains a newline (e.g., a description field), a single logical record spans multiple physical lines. In this case, the LineNumber will point to the start of the record, but the next record's LineNumber will jump several lines ahead.
* **Comments**: If CommentCharacter is set, comment lines occupy a physical LineNumber but do not increment the RecordIndex.
* **Header**: The header row consumes a LineNumber but is not counted as a data RecordIndex.

**Usage Tip**: Always use LineNumber when reporting errors to users, as it corresponds directly to what they see in a text editor!

## 5. Result Handling: Success, Error, and Comment ##

The `CsvMappingResult<T>` captures every possible state of a row. The `Switch` method ensures 
all states are handled correctly.

```csharp
foreach (CsvMappingResult<Person> item in parser.ReadFromStream(stream))
{
    item.Switch(
        onSuccess: (Person entity) => 
            Console.WriteLine($"[Record {item.RecordIndex}] Imported: {entity.Name}"),
            
        onFailure: (CsvMappingError error) => 
            Console.WriteLine($"[Line {item.LineNumber}] Error in Column {error.ColumnIndex}: {error.Value}"),
            
        onComment: (string comment) => 
            Console.WriteLine($"[Line {item.LineNumber}] Meta-Info: {comment}")
    );
}
```

## 6. Advanced Usage: Accessing CsvRow ##

For complex scenarios, `MapUsing` provides direct access to the `ref struct CsvRow`. This is useful for mapping multiple columns 
into a single property or performing manual validation.

```csharp
public class AdvancedMapping : CsvMapping<Person>
{
    public AdvancedMapping()
    {
        MapUsing((Person entity, ref CsvRow row) =>
        {
            // row.Count checks the number of columns found
            if (row.Count < 2) return false;

            // row.GetSpan() returns ReadOnlySpan<char> (Zero-Allocation)
            if (!int.TryParse(row.GetSpan(0), out int id)) return false;
            entity.Id = id;
            
            // row.GetString() handles unescaping of quotes automatically
            entity.Name = row.GetString(1);

            return true;
        });
    }
}
```

## 7. TypeConverters ##

### 7.1 Configuring Existing Converters ###

You can pass specific parameters (like date formats) to built-in converters during mapping.

```csharp
DateTimeConverter dateConverter = new("yyyy-MM-dd");

MapProperty("BirthDate", x => x.BirthDate, dateConverter);
```

### 7.2 Writing a Custom Converter ###

Inherit from `NonNullableConverter<T>` to implement custom parsing logic directly on the memory spans.

```csharp
public class YesNoConverter : NonNullableConverter<bool>
{
    protected override bool InternalConvert(ReadOnlySpan<char> value, out bool result)
    {
        if (value.Equals("Yes".AsSpan(), StringComparison.OrdinalIgnoreCase))
        {
            result = true;
            return true;
        }
        result = false;
        return false;
    }
}
```

## 8. Migration from 2.x to 3.x ##

### 8.1 Data Access ###

In Version 2.x, custom logic used a `string[]`. In Version 3.0, it uses `ref CsvRow`. This allows the library to work 
with `ReadOnlySpan<char>`, significantly reducing memory allocations.

### 8.2 Result Pattern ###

The addition of the Comment state means that Match and Switch now require a third functional argument. Use these overloads to handle metadata rows found in the CSV.

### 8.3 Error Metadata ###

Error objects in Version 3.0 now contain both `RecordIndex` and `LineNumber`. If you previously relied on indices for debugging, ensure 
you switch to `LineNumber` for file-based troubleshooting. This is what the user sees in their CSV file.