title: Dynamic Mappings with the TinyCsvParser 3.0.0 API
date: 2026-03-23 10:43
tags: csharp, dotnet, csv
category: dotnet
slug: tinycsvparser_3_dynamic_mappings
author: Philipp Wagner
summary: Dynamic Mappings with the TinyCsvParser 3.0.0 API.

Now that TinyCsvParser is finally updated, we can add all kinds of features, that 
had been missing in previous iterations of the library. The source code and documentation 
for TinyCsvParser can be found at:

* [https://github.com/TinyCsvParser/TinyCsvParser](https://github.com/TinyCsvParser/TinyCsvParser)

One of things, that had been missing in TinyCsvParser had been dynamic mappings, so you 
don't need to create dedicated classes to parse some CSV data.

## Table of contents ##

[TOC]

## What is TinyCsvParser? ##

TinyCsvParser is a high-performance CSV parsing library for .NET. This documentation explains the usage, 
configuration, and extensibility of the library through practical examples.

> Upgrading from a previous version? Check out the [Migration Guide from 2.x to 3.x](https://www.bytefish.de/blog/tinycsvparser_3_api.html#8-migration-from-2x-to-3x)

## Dynamic Mapping (Dictionary & ExpandoObject) ##

When the CSV schema is only known at runtime, or you want to avoid creating dedicated classes for 
simple scripts, you can parse rows directly into dynamic structures (`Dictionary<string, object?>` or 
`ExpandoObject`).

For performance it's maybe better to map to a `Dictionary`, as it avoids Dynamic Language Runtime overhead.

### Inline Configuration ###

Use the static factory methods on the `CsvParser` class. The schema is configured inline using a delegate.

```csharp
using TinyCsvParser;

CsvOptions options = new(Delimiter: ';', QuoteChar: '"', EscapeChar: '"', SkipHeader: false);

// Create the parser and configure the schema in one go
var parser = CsvParser.CreateDictionaryParser(options, schema => 
{
    schema.Add<int>("Id");       // Resolves Int32Converter automatically
    schema.Add<double>("Price"); // Resolves DoubleConverter automatically
});

foreach (var result in parser.ReadFromFile("products.csv"))
{
    if (result.IsSuccess)
    {
        Dictionary<string, object?> row = result.Result;
        Console.WriteLine($"Item {row["Id"]} costs {row["Price"]}");
    }
}
```

> Note: Use `CsvParser.CreateExpandoParser(...)` if you prefer to access fields via the dynamic keyword like `row.Id`.

### Explicit Converters & Custom Providers ###

While `Add<T>` is the most convenient method, you can pass explicit converter instances if you need special configurations 
(e.g., `date formats`).

```csharp
var parser = CsvParser.CreateDictionaryParser(options, schema => 
{
    schema.Add<int>("Id");
    schema.Add("BirthDate", new DateTimeConverter("yyyy-MM-dd")); // Explicit Converter
});
```

### Fallback Behavior ###

Any column present in the CSV header that is not mapped in your `CsvSchema` will automatically be 
parsed as a raw `string`. This prevents data loss while maintaining strict typing for the columns 
you care about.

