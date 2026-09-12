# IEnumerable vs. IQueryable in C#

*A deep-dive walkthrough of `IEnumerable<T>` and `IQueryable<T>` in C# — covering what each interface actually contains, why `IQueryable<T>` builds on top of `IEnumerable<T>` rather than replacing it, expression trees vs. delegates as the mechanical difference between them, where a query actually executes in each case, the provider model that lets `IQueryable<T>` target SQL, Cosmos DB, or anything else, and the specific, common bugs (accidental early materialization, mixed in-memory/server-side filtering) that come directly from confusing the two.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Core Distinction, Stated Precisely](#1-the-core-distinction-stated-precisely)
3. [IEnumerable&lt;T&gt;: The Foundation](#2-ienumerablet-the-foundation)
4. [IQueryable&lt;T&gt;: Built on Top of IEnumerable&lt;T&gt;](#3-iqueryablet-built-on-top-of-ienumerablet)
5. [Delegates vs. Expression Trees: The Actual Mechanical Difference](#4-delegates-vs-expression-trees-the-actual-mechanical-difference)
6. [Where Execution Actually Happens](#5-where-execution-actually-happens)
7. [The Provider Model: How IQueryable&lt;T&gt; Targets Different Data Sources](#6-the-provider-model-how-iqueryablet-targets-different-data-sources)
8. [The Classic Bug: Accidental Early Materialization](#7-the-classic-bug-accidental-early-materialization)
9. [The Classic Bug: Un-Translatable Expressions](#8-the-classic-bug-un-translatable-expressions)
10. [Composing IQueryable&lt;T&gt; Queries Across Methods](#9-composing-iqueryablet-queries-across-methods)
11. [AsEnumerable, AsQueryable, and Deliberately Crossing the Boundary](#10-asenumerable-asqueryable-and-deliberately-crossing-the-boundary)
12. [Performance: Why the Difference Is Not Academic](#11-performance-why-the-difference-is-not-academic)
13. [How to Tell Which One You Actually Have](#12-how-to-tell-which-one-you-actually-have)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

`IEnumerable<T>` and `IQueryable<T>` look almost identical from the outside — both represent "a sequence of `T` you can query with LINQ," and the same `Where`, `Select`, `OrderBy` syntax works on either one. But they represent two fundamentally different execution models, and confusing them is one of the most common, costly mistakes in real-world C# code working against a database: `IEnumerable<T>` filters, sorts, and projects data that's already in application memory, using ordinary compiled delegates; `IQueryable<T>` builds up a data structure describing what you *want* to happen, which a provider then translates into the target data source's own query language — SQL, for Entity Framework — and executes remotely, pulling back only the result. This guide, building directly on this series' LINQ guide's introduction of expression trees, goes deep on exactly how and why these two interfaces differ, and the concrete bugs that follow from treating them as interchangeable.

```plaintext
IEnumerable<T> query = inMemoryList.Where(p => p.Price > 100);
   → compiles the lambda to a DELEGATE → filters objects ALREADY IN MEMORY, in .NET

IQueryable<T> query = dbContext.Products.Where(p => p.Price > 100);
   → compiles the lambda to an EXPRESSION TREE → a PROVIDER translates it to SQL
   → "SELECT * FROM Products WHERE Price > 100" runs ON THE DATABASE SERVER
   → only the matching ROWS are ever transferred back into .NET memory
```

---

## 1. The Core Distinction, Stated Precisely

### The one-line summary, and why it deserves a full guide's worth of unpacking

```plaintext
IEnumerable<T>: "here is a sequence; whatever you ask of it happens IN MEMORY, in .NET."
IQueryable<T>:  "here is a sequence; whatever you ask of it becomes a DESCRIPTION that
                  gets TRANSLATED and executed somewhere else — typically a database."
```

This distinction sounds simple stated this way, but it has real, concrete consequences for where computation happens, how much data crosses the network, what C# expressions are even legal to write, and a specific, common class of bug that this guide exists to make unmistakable rather than a vague intuition.

### Both represent "a sequence you can query" — that's what makes them easy to confuse

```csharp
IEnumerable<Product> a = someList.Where(p => p.Price > 100);       // in-memory
IQueryable<Product> b = dbContext.Products.Where(p => p.Price > 100); // database-translated

// Both of these compile and LOOK identical at the call site:
foreach (var product in a) Console.WriteLine(product.Name);
foreach (var product in b) Console.WriteLine(product.Name);
```

The syntax is deliberately, genuinely identical — this is LINQ's whole design point, per this series' LINQ guide's introduction — which is exactly why it's so easy to write code that works correctly against one and then breaks, silently or loudly, when the underlying type turns out to be the other.

---

## 2. IEnumerable&lt;T&gt;: The Foundation

### The interface itself is almost nothing — a single method

```csharp
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<out T> : IDisposable, IEnumerator
{
    T Current { get; }
    bool MoveNext();
    void Reset();
}
```

`IEnumerable<T>` itself defines almost no behavior — just "give me something that can produce a `Current` item and advance via `MoveNext()`." Everything that feels like "LINQ functionality" (`Where`, `Select`, `OrderBy`, and the rest) is not part of this interface at all — as this series' LINQ guide's Section 6 covers, those are extension methods defined separately in `System.Linq.Enumerable`, working generically against any `IEnumerable<T>`.

### LINQ operators on IEnumerable&lt;T&gt; compile lambdas to real delegates

```csharp
public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate)
{
    foreach (var item in source)
        if (predicate(item)) yield return item; // predicate is an ORDINARY, callable delegate
}
```

This is the crucial mechanical fact this whole guide builds on: `Enumerable.Where`'s second parameter is `Func<TSource, bool>` — a genuine, JIT-compiled delegate, per this series' Delegates guide. When you write `list.Where(p => p.Price > 100)` against an `IEnumerable<T>`, the C# compiler produces ordinary executable code for that lambda, and `Where`'s implementation calls it directly, once per element, entirely within the running .NET process.

### Everything happens against objects already sitting in memory

```csharp
List<Product> products = LoadAllProductsFromSomewhere(); // however these got here, they're in memory NOW
var expensive = products.Where(p => p.Price > 100); // filters objects ALREADY resident in .NET memory
```

By the time any `IEnumerable<T>` LINQ operator runs, the objects it's operating on already exist as real .NET objects in the process's memory — `Where`, `OrderBy`, and the rest are simply iterating and inspecting objects that are already there; there's no translation step, no remote execution, nothing beyond ordinary, direct C# code running against real objects.

---

## 3. IQueryable&lt;T&gt;: Built on Top of IEnumerable&lt;T&gt;

### The interface declaration — note it inherits from IEnumerable&lt;T&gt; directly

```csharp
public interface IQueryable<out T> : IEnumerable<T>, IQueryable, IEnumerable
{
    // (inherited members only — IQueryable<T> itself adds no NEW members beyond IQueryable's)
}

public interface IQueryable : IEnumerable
{
    Type ElementType { get; }
    Expression Expression { get; }   // the EXPRESSION TREE representing this query, built up so far
    IQueryProvider Provider { get; } // the PROVIDER responsible for translating and executing it
}
```

This is worth sitting with directly: `IQueryable<T>` *is* an `IEnumerable<T>` — every `IQueryable<T>` can be used anywhere an `IEnumerable<T>` is expected, and it inherits that same `GetEnumerator()` capability. What it adds is exactly two things: an `Expression` (the query, represented as data, built up incrementally) and a `Provider` (the object that knows how to turn that expression into something executable against a specific data source).

### LINQ operators on IQueryable&lt;T&gt; compile lambdas to expression trees instead

```csharp
public static IQueryable<TSource> Where<TSource>(this IQueryable<TSource> source, Expression<Func<TSource, bool>> predicate)
{
    // Note the parameter type: Expression<Func<TSource, bool>>, NOT Func<TSource, bool> directly.
    // This method doesn't loop and call predicate() at all — it builds a NEW Expression combining
    // source.Expression with this Where call and this predicate, and returns a new IQueryable<T>
    // wrapping that combined expression. NOTHING has executed yet.
}
```

This is the single most important mechanical fact in this entire guide: `Queryable.Where` (the `IQueryable<T>` version, distinct from `Enumerable.Where`) takes an `Expression<Func<TSource, bool>>`, not a plain `Func<TSource, bool>`. The `Expression<...>` wrapper is what tells the C# compiler: "don't compile this lambda into executable code — compile it into a data structure describing the lambda's logic instead." Calling `Where` here doesn't filter anything; it constructs a bigger expression tree representing "the previous query, plus this additional filter condition," and hands back a new `IQueryable<T>` wrapping that combined tree.

### Every `IQueryable<T>` LINQ call builds up a bigger tree — nothing runs until enumeration

```csharp
IQueryable<Product> query = dbContext.Products;               // Expression: "Products table"
query = query.Where(p => p.Price > 100);                       // Expression: "Products table, WHERE Price > 100"
query = query.OrderBy(p => p.Name);                             // Expression: "..., ORDER BY Name"
query = query.Select(p => p.Name);                               // Expression: "..., SELECT Name"

var results = query.ToList(); // ONLY NOW does the Provider translate the WHOLE tree and execute it
```

Just as this series' LINQ guide's Section 3 covers for deferred execution generally, each of these calls returns a *new* `IQueryable<T>` describing a progressively larger query — nothing about the actual data has been touched yet. It's only when something forces enumeration (`ToList()`, `foreach`, `Count()`, and similar) that the `Provider` steps in, translates the complete, final expression tree into the target query language, sends it off, and materializes the results.

---

## 4. Delegates vs. Expression Trees: The Actual Mechanical Difference

### Same C# syntax, two entirely different things the compiler produces

```csharp
Func<Product, bool> asDelegate = p => p.Price > 100;
// Compiled to: a real, callable block of IL/machine code. You can invoke it: asDelegate(someProduct) → bool.

Expression<Func<Product, bool>> asExpressionTree = p => p.Price > 100;
// Compiled to: a DATA STRUCTURE. asExpressionTree.Body is an object graph representing
// "a BinaryExpression, Operator=GreaterThan, Left=(MemberAccess 'Price' on parameter p), Right=(Constant 100)"
```

This is the exact same lambda, written identically in C# — the *only* difference is which type it's being assigned to. `Func<Product, bool>` tells the compiler "compile this to executable code, right now." `Expression<Func<Product, bool>>` tells the compiler "compile this to a tree of objects that *describes* the code, so something else can inspect, transform, or translate it later." This single compiler behavior is the entire mechanical foundation this whole guide rests on.

### Inspecting an expression tree directly, to make this concrete

```csharp
Expression<Func<Product, bool>> expr = p => p.Price > 100;

var body = (BinaryExpression)expr.Body;
Console.WriteLine(body.NodeType);                    // GreaterThan
Console.WriteLine(((MemberExpression)body.Left).Member.Name); // "Price"
Console.WriteLine(((ConstantExpression)body.Right).Value);    // 100
```

You can genuinely walk this structure at runtime, node by node — it's real, inspectable data, not a black box. This is exactly what Entity Framework's SQL provider does internally: walk the tree, recognize the `GreaterThan` node, recognize the `Price` member access maps to a specific database column, and emit the equivalent `WHERE Price > 100` SQL fragment — a mechanical translation process, not magic, and one that only works because the lambda was captured as data instead of compiled directly into unreadable machine code.

### Why `IEnumerable<T>` couldn't work this way even if it wanted to

```plaintext
There is no general way to take an arbitrary, already-JIT-compiled block of
  machine code (a Func<T, bool> delegate) and reverse-engineer what SQL it
  corresponds to — the information needed to do that translation (which
  property was compared, to what value, with which operator) is exactly
  the information an EXPRESSION TREE preserves as inspectable data, and
  exactly the information a compiled DELEGATE discards once it's just
  runnable code.
```

This is the deep reason `IEnumerable<T>` genuinely cannot be translated to SQL, and it isn't a limitation someone forgot to fix — a compiled delegate is opaque by design; an expression tree is transparent by design. `IQueryable<T>` exists specifically because someone needed a LINQ-compatible type that preserves the "what" instead of just the "how to run it."

---

## 5. Where Execution Actually Happens

### IEnumerable&lt;T&gt;: every operator runs in the current .NET process, against in-memory objects

```csharp
List<Product> allProducts = LoadEntireProductCatalogIntoMemory(); // however this happened, ALL of it is now in RAM
var expensive = allProducts.Where(p => p.Price > 100).ToList();   // filters in-process, over objects already loaded
```

There is no "elsewhere" here — `Where`'s delegate runs directly, in the calling thread (or wherever the enumeration happens), against objects that already exist fully in memory. If `allProducts` held a million rows, all one million were already loaded before this filter ever ran.

### IQueryable&lt;T&gt;: the operator chain becomes a REMOTE query, and only the RESULT crosses back

```csharp
var expensive = dbContext.Products.Where(p => p.Price > 100).ToList();
// The WHERE clause executes ON THE DATABASE SERVER. If there are a million rows total but
// only 200 have Price > 100, ONLY THOSE 200 ROWS are ever transferred over the network or
// loaded into .NET memory at all — the other 999,800 rows never leave the database.
```

This is the practical payoff that makes the distinction matter enormously for real applications: `IQueryable<T>`'s translation-and-remote-execution model means filtering, sorting, and even some projection can happen where the data already lives — on the database server — with only the relevant, already-reduced result set actually making the trip into your application's memory.

### A visual side-by-side of the two execution paths

```plaintext
IEnumerable<T> path:
  Database → [ENTIRE TABLE pulled into .NET memory] → Where/OrderBy/Select run IN-PROCESS → result

IQueryable<T> path:
  C# lambda → EXPRESSION TREE → Provider translates → SQL sent to database →
  database executes Where/OrderBy/Select ITSELF → only matching rows returned → result
```

Every operator placed *before* the point of materialization in an `IQueryable<T>` chain gets a chance to be translated and pushed down to the data source; every operator placed *after* materialization (Section 7 covers exactly how this happens by accident) necessarily runs as ordinary `IEnumerable<T>` LINQ, in memory, over whatever was already pulled back.

---

## 6. The Provider Model: How IQueryable&lt;T&gt; Targets Different Data Sources

### `IQueryProvider` is the piece that actually knows how to translate and execute

```csharp
public interface IQueryProvider
{
    IQueryable CreateQuery(Expression expression);
    IQueryable<TElement> CreateQuery<TElement>(Expression expression);
    object Execute(Expression expression);
    TResult Execute<TResult>(Expression expression);
}
```

Every `IQueryable<T>` carries a reference to a specific `Provider` (Section 3) — this is the object that receives the final, complete expression tree at enumeration time and is responsible for making sense of it in whatever way is appropriate for its specific target. Entity Framework Core has one `IQueryProvider` implementation that translates to SQL; Azure Cosmos DB's SDK has an entirely different one translating to its own query language; an in-memory testing implementation might just interpret the tree directly.

### This is exactly why the SAME LINQ syntax works across totally different data sources

```csharp
dbContext.Products.Where(p => p.Price > 100).ToList();      // → translated to SQL by EF Core's provider
cosmosContainer.GetItemLinqQueryable<Product>()
    .Where(p => p.Price > 100).ToList();                     // → translated to Cosmos DB's SQL-like query language
```

The C# code you write is nearly identical in both cases — what differs entirely is which `IQueryProvider` receives the expression tree at execution time, and therefore what target language it gets translated into. This is the concrete mechanism behind LINQ's "one syntax, many data sources" promise this series' LINQ guide introduces — it's not that LINQ magically knows SQL and Cosmos DB's query language both; it's that each provider independently implements the translation logic for its own target.

### Custom `IQueryable<T>` providers are possible, though rarely written by hand

```plaintext
Writing your OWN IQueryProvider from scratch — walking an arbitrary
  expression tree and translating it correctly for a genuinely custom
  data source — is a real, substantial undertaking, which is why most
  applications consume an existing provider (EF Core, a specific NoSQL
  SDK) rather than building one; it's worth knowing this is what's
  happening underneath, even if you'll rarely implement it yourself.
```

Worth knowing this exists as a real extensibility point in the LINQ design, even though the overwhelming majority of developers only ever *consume* a provider someone else wrote (EF Core, MongoDB's driver, Cosmos DB's SDK) rather than writing one — understanding that a provider is doing real translation work is what makes Section 8's "not everything translates" limitation make sense rather than feeling arbitrary.

---

## 7. The Classic Bug: Accidental Early Materialization

### Calling `.ToList()` (or `.AsEnumerable()`) before filtering silently switches to IEnumerable&lt;T&gt;

```csharp
// ❌ BUG: pulls the ENTIRE Products table into memory FIRST, then filters in .NET
var expensive = dbContext.Products.ToList().Where(p => p.Price > 100);

// ✅ CORRECT: the filter is part of the expression tree BEFORE materialization,
//    so it's translated to SQL and only matching rows are ever transferred
var expensive2 = dbContext.Products.Where(p => p.Price > 100).ToList();
```

`dbContext.Products` is `IQueryable<Product>`; calling `.ToList()` on it immediately forces materialization — the *entire* table gets pulled into memory at that exact point, and only *then* does the subsequent `.Where(...)` run, now against a plain in-memory `List<Product>`, using `Enumerable.Where` (Section 2) rather than `Queryable.Where` (Section 3). The bug is easy to miss because both versions compile without error and produce *the same result* — the difference is entirely in performance and network cost, invisible until the table is large enough to make the difference painfully obvious.

### The type actually changes at the point `.ToList()` is called — this is the mechanical root cause

```csharp
IQueryable<Product> query = dbContext.Products; // still IQueryable — expression tree, not yet executed
List<Product> materialized = query.ToList();     // NOW it's a List<Product> — plain, in-memory
var filtered = materialized.Where(p => p.Price > 100); // this Where is Enumerable.Where — IEnumerable<T>, in-memory
```

Once you have a `List<T>` (or any concrete, already-populated collection), every subsequent LINQ call against it is unavoidably an `IEnumerable<T>` operation — there is no way to "push a filter back down" to a database query that has already fully executed and returned its results. The fix (Section 9) is disciplinary: always compose every filter, sort, and projection you actually need *before* the operation that materializes the query.

---

## 8. The Classic Bug: Un-Translatable Expressions

### A provider can only translate what it knows how to translate

```csharp
public bool PassesCustomBusinessRule(Product p) => SomeComplexLogic(p);

// ❌ Throws at RUNTIME (typically a NotSupportedException or similar, provider-dependent):
//    EF Core has no idea how to turn a call to an arbitrary C# method into SQL
var results = dbContext.Products.Where(p => PassesCustomBusinessRule(p)).ToList();
```

Section 6 established that translation is real, mechanical work a provider does by walking the expression tree — and that work is fundamentally limited to expression shapes the provider was written to recognize. An arbitrary method call, especially one wrapping non-trivial logic, has no general mapping to SQL (or whatever the target query language is), so the provider either throws an exception at the point of execution, or — in some cases, depending on the specific provider and its version — falls back to evaluating that part client-side, which reintroduces exactly the "pull more than you need into memory" cost Section 7 covers, just less visibly.

### Keeping `IQueryable<T>` predicates within what's genuinely translatable

```csharp
// ✅ Simple property comparisons, standard operators, and built-in functions typically translate cleanly
var results = dbContext.Products
    .Where(p => p.Price > 100 && p.Category == "Electronics" && p.Name.Contains("Pro"))
    .ToList(); // Contains() here maps to SQL LIKE — this specific pattern IS supported by EF Core's provider
```

The practical discipline this implies: keep `IQueryable<T>` predicates to expressions built from simple property access, comparisons, and the specific subset of .NET methods a given provider documents as translatable (EF Core, for instance, supports a meaningful subset of `string` and `DateTime` methods specifically because its provider has explicit translation logic for them) — anything genuinely custom or complex should either be expressed differently, or deliberately materialized first (accepting the cost) and then filtered in-memory afterward.

---

## 9. Composing IQueryable&lt;T&gt; Queries Across Methods

### Passing an `IQueryable<T>` through several methods, each adding to the eventual query

```csharp
public IQueryable<Product> FilterByCategory(IQueryable<Product> query, string? category)
{
    if (!string.IsNullOrEmpty(category))
        query = query.Where(p => p.Category == category); // still building the tree, nothing executes
    return query;
}

public IQueryable<Product> FilterByPrice(IQueryable<Product> query, decimal? minPrice)
{
    if (minPrice.HasValue)
        query = query.Where(p => p.Price >= minPrice.Value);
    return query;
}

// Composing the pieces:
IQueryable<Product> query = dbContext.Products;
query = FilterByCategory(query, categoryFilter);
query = FilterByPrice(query, minPriceFilter);
var results = query.ToList(); // ONE combined SQL query, with whichever conditions actually applied
```

This is exactly this series' LINQ guide's Section 10 conditional-query-building pattern, but worth restating specifically for `IQueryable<T>`: because each method receives and returns an `IQueryable<T>` (never forcing materialization along the way), the whole chain — across as many separate methods as needed — still results in exactly one SQL query at the end, with every applicable filter correctly combined. The moment any of these methods returned `IEnumerable<T>` instead (by calling `.ToList()` internally, even "just to be safe"), this composition would break down into Section 7's early-materialization bug.

### Why the parameter and return types matter as much as the logic inside the methods

```csharp
// ❌ This signature FORCES materialization the moment it's called, breaking composition for any caller
public List<Product> FilterByCategory(IQueryable<Product> query, string category) => query.Where(...).ToList();

// ✅ This signature preserves IQueryable<T> all the way through, letting callers keep composing
public IQueryable<Product> FilterByCategory(IQueryable<Product> query, string category) => query.Where(...);
```

This is a genuinely important API design detail specific to working with `IQueryable<T>`: a method's declared return type isn't just a style choice — returning `List<T>` (or any materialized type) from a method meant to be part of a larger, composable query chain forces execution at exactly that point, regardless of what the caller actually wanted to do next, which is precisely why data-access layer methods intended to support further filtering by their callers should return `IQueryable<T>`, reserving `.ToList()`/`.ToArray()` for the actual endpoint of the query, chosen deliberately by whichever caller is finally ready to execute it.

---

## 10. AsEnumerable, AsQueryable, and Deliberately Crossing the Boundary

### `AsEnumerable()`: deliberately dropping into IEnumerable&lt;T&gt; semantics, mid-query

```csharp
var results = dbContext.Products
    .Where(p => p.Price > 100)         // translated to SQL — runs on the database
    .AsEnumerable()                     // ⚠️ deliberate boundary: everything AFTER this runs IN-MEMORY
    .Where(p => PassesCustomBusinessRule(p)) // this Where is now Enumerable.Where — a real, callable delegate
    .ToList();
```

`AsEnumerable()` is the *deliberate*, intentional version of Section 7's accidental bug — sometimes you genuinely need to combine a translatable filter (push it to the database) with a non-translatable one (Section 8's custom business logic), and `AsEnumerable()` is the explicit, visible way to say "translate and execute everything up to this point on the server, then finish the rest in memory" — as opposed to `.ToList()` doing the same thing implicitly and, often, unintentionally.

### `AsQueryable()`: the reverse — wrapping an in-memory sequence as IQueryable&lt;T&gt;

```csharp
List<Product> inMemoryProducts = GetTestData();
IQueryable<Product> asQueryable = inMemoryProducts.AsQueryable(); // wraps it, doesn't translate to anything remote
```

This is primarily useful for testing code that's written against `IQueryable<T>` (like a repository method expecting `IQueryable<Product>`) without needing an actual database — `AsQueryable()` provides a real `IQueryable<T>` (using an in-memory, no-op provider) that the test can pass in, letting the exact same production code path be exercised against in-memory test data.

---

## 11. Performance: Why the Difference Is Not Academic

### Network transfer volume is often the dominant cost, and IQueryable&lt;T&gt; minimizes it directly

```plaintext
A Products table with 2 million rows, where only 4,000 have Price > 100:
IEnumerable<T> (materialize-then-filter): transfers 2,000,000 rows over the network
IQueryable<T> (filter-then-materialize):  transfers only 4,000 rows over the network
```

This isn't a subtle, theoretical difference — for any table of meaningful size, the gap between "the database does the filtering" and "your application does the filtering after receiving everything" is frequently the single largest, most consequential performance decision in a whole application's data-access layer, dwarfing most micro-optimizations elsewhere in the code.

### Database-side execution can also use indexes, which in-memory filtering never benefits from

```plaintext
A WHERE Price > 100 clause translated to SQL can use an INDEX on the Price
  column, letting the database find matching rows without scanning the
  entire table — an in-memory Where over a List<T> the application already
  materialized has no equivalent optimization available; it's always an
  O(n) linear scan over whatever was already pulled into memory.
```

This is a second, independent reason `IQueryable<T>`'s server-side translation matters beyond just network transfer volume — the database's own query optimizer and indexing infrastructure can only help if the filtering condition is actually part of the SQL query it receives, which requires the condition to be translated (Section 6), which requires it to still be part of the expression tree at execution time (Section 7's whole point).

---

## 12. How to Tell Which One You Actually Have

### Check the declared type — IDE tooling makes this immediate

```csharp
var query = dbContext.Products.Where(p => p.Price > 100); // hover in your IDE: IQueryable<Product>
var list = dbContext.Products.ToList();                     // hover in your IDE: List<Product>
var filteredList = list.Where(p => p.Price > 100);          // hover in your IDE: IEnumerable<Product>
```

Most modern C# IDEs show the inferred type directly on hover — this is the most reliable, immediate way to check whether a given LINQ expression is still `IQueryable<T>` (still translatable, still deferred against the remote source) or has already become `IEnumerable<T>` (already in memory, from this point on). Making a habit of checking this at each step of a data-access method, especially one composing several conditional filters, is a cheap, effective way to catch Section 7's bug before it ships.

### A quick mental checklist for any LINQ chain against a database context

```plaintext
1. Does this chain call .ToList()/.ToArray()/.AsEnumerable() ANYWHERE before the LAST operator
   that genuinely needs to run in-memory? → check whether that's intentional (Section 10) or accidental (Section 7).
2. Does any predicate reference a custom method, rather than simple property access
   and provider-supported built-ins? → check it will actually translate (Section 8), or that
   AsEnumerable() has been used deliberately if it can't.
3. Is this method's return type IQueryable<T>, or has it been narrowed to a concrete,
   materialized collection type? → check whether that narrowing is intentional or would
   break composition for a caller who needed to add more filters (Section 9).
```

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Calling `.ToList()` before filtering/sorting an `IQueryable<T>` source | Pulls the entire table into memory first; every subsequent LINQ operator runs in-process instead of translating to SQL | Compose all filters, sorts, and projections before the final materializing call (Section 7) |
| Using a custom or complex C# method inside an `IQueryable<T>` predicate | The provider often cannot translate an arbitrary method call into SQL, causing a runtime exception or unexpected in-memory fallback | Keep predicates to simple property access and provider-supported built-in methods (Section 8); use `AsEnumerable()` deliberately when custom logic is genuinely needed |
| Returning a concrete, materialized collection type from a data-access method meant to support further filtering | Forces execution at that exact point, breaking composition for any caller wanting to add more conditions | Return `IQueryable<T>` from composable data-access methods; materialize only at the true endpoint of the query (Section 9) |
| Assuming `IEnumerable<T>` and `IQueryable<T>` are interchangeable because the syntax looks identical | The execution model, translatability, and performance characteristics are fundamentally different, even though both support the same LINQ syntax | Actively check the declared type (Section 12) at each step of a data-access chain, especially against a database context |
| Treating `AsEnumerable()`/`.ToList()` as functionally equivalent ways to "just get the data" | One is a deliberate, visible boundary marker (`AsEnumerable()`); the other is easy to place accidentally in the wrong spot | Use `AsEnumerable()` specifically when deliberately switching to in-memory semantics mid-query, distinct from `.ToList()`'s role as final materialization |
| Assuming an in-memory `Where` benefits from database indexes the same way a translated `WHERE` clause does | In-memory filtering is always a linear scan over already-materialized objects; it can never use a database index | Push filtering into the `IQueryable<T>` portion of the query, before materialization, so the database's indexes and optimizer can actually help |
| Writing a custom `IQueryProvider` from scratch without recognizing the real complexity involved | Correctly walking and translating arbitrary expression trees for a new target is a substantial undertaking, easy to underestimate | Prefer consuming an existing, well-tested provider (EF Core, a vendor's SDK) over building one, unless there's a genuinely compelling reason not to (Section 6) |
| Assuming `IQueryable<T>` is strictly "better" than `IEnumerable<T>` and reaching for it even for in-memory collections | `IQueryable<T>` adds real overhead (expression tree construction, provider translation) with zero benefit when the data source is already an in-memory collection | Use `IEnumerable<T>` for genuinely in-memory sequences; reserve `IQueryable<T>` for sources that actually benefit from translation and remote execution |

---

## Quick Reference Table

| Concept | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Inherits from | `IEnumerable` | `IEnumerable<T>` (and thus `IEnumerable`) |
| Lambda compiled to | `Func<T, TResult>` — a real, callable delegate | `Expression<Func<T, TResult>>` — an inspectable expression tree |
| Where filtering happens | In the current .NET process, over already-loaded objects | Wherever the `Provider` translates and sends the query — typically a database server |
| Typical source | `List<T>`, arrays, in-memory sequences | `DbSet<T>` (EF Core), other ORM/NoSQL query roots |
| Key added members over `IEnumerable<T>` | — | `Expression`, `Provider` |
| Can use database indexes | No — always a linear, in-memory scan | Yes, if the translated query can leverage them |
| Translatability limits | None — any valid C# delegate works | Limited to what the specific `Provider` knows how to translate |
| Deliberately switching to the other | `AsQueryable()` (wraps as `IQueryable<T>`, no real translation) | `AsEnumerable()` (materializes remaining logic to run in-memory) |

---

## Conclusion

`IEnumerable<T>` and `IQueryable<T>` present the same LINQ surface deliberately, but underneath that surface they represent two entirely different answers to "where does this query actually run": `IEnumerable<T>` compiles your lambdas into real delegates and executes them directly, in-process, against objects already sitting in memory; `IQueryable<T>` compiles the exact same-looking lambdas into expression trees — inspectable data describing what you asked for — which a provider then translates into the target data source's own query language and executes remotely, pulling back only the reduced result. Understanding this is what turns "LINQ against a database" from a syntax you've memorized into something you can reason about correctly: knowing that composing filters before materialization keeps a query translatable and pushed down to the server, that a custom method in a predicate might silently fail to translate, and that a method's return type is a real architectural decision about whether callers can keep composing the query further.

The bugs this guide spends the most time on — accidental early materialization, and predicates that don't translate — both come from the same root cause: losing track of which of the two interfaces you're actually holding at a given point in a LINQ chain. Checking the declared type, being deliberate about `AsEnumerable()`/`AsQueryable()` as explicit boundary markers rather than interchangeable "just get me the data" calls, and keeping data-access methods returning `IQueryable<T>` until the genuine end of a query, are what turn this from a source of silent, expensive network round trips into a correctly and deliberately designed data-access layer.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the ".ToList() one line too early pulled the whole table across the network" incident that made the IEnumerable/IQueryable distinction click far better than any interface diagram ever could.*
