# Modern C# Style

.NET 10 / C# 14 (LTS).  Prefer the newest language feature that makes the code shorter and clearer; do not write 2018 C#.  Formatting is CSharpier (default settings, 100 columns): run `csharpier format <files>` on every file you touch, and do not argue with it.

## Language features to use

`src/backend/packages/<PROJECT>.Notes/StyleProbe.cs`

```csharp
namespace <PROJECT>.Notes;

// 1. Extension members (C# 14): properties and methods in an `extension` block.
public static class StringExtensions
{
    extension(string? value)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(value);

        public string OrDefault(string fallback) => value.IsBlank ? fallback : value!;
    }
}

// 2. `field` keyword (C# 14): the compiler-synthesized backing field.
public sealed class Draft
{
    public required string Title
    {
        get;
        set => field = value.Trim();
    }

    public List<string>? Tags { get; set; }
}

public static class StyleProbe
{
    public static string Run(Draft? draft)
    {
        // 3. Null-conditional assignment (C# 14).
        draft?.Tags = ["a", "b"];

        // 4. Switch expression with relational + property patterns.
        var kind = draft switch
        {
            null => "none",
            { Tags: [] } => "untagged",
            { Tags.Count: > 1 } => "multi",
            _ => "single",
        };

        // 5. List pattern.
        int[] numbers = [1, 2, 3];
        var head = numbers is [var first, ..] ? first : -1;

        // 6. `with` on records, collection expressions, raw string literal.
        var request = new CreateNoteRequest("t", null) with
        {
            Body = "b",
        };
        string[] merged = [.. numbers.Select(n => n.ToString()), request.Title];
        var text = """
            line one
              indented
            """;

        // 7. Extension property + method from (1).
        return $"{kind}|{head}|{string.Join(',', merged)}|{text.Length}|{request.Body.OrDefault("x")}|{"  ".IsBlank}";
    }
}
```

Also standard practice here:

- **Primary constructors** for DI: `internal sealed class NoteStore(AppDbContext db) : INoteStore`.  Use the parameter directly; promote it to a field only when it must be mutated.
- **Records** for contracts and value-like types; `sealed record` by default.  Non-destructive edits with `with`.
- **Collection expressions** everywhere: `[]`, `[a, b]`, `[.. items.Select(...)]`.  Not `new List<T>()`, `Array.Empty<T>()`, `.ToList()` for a copy.
- **`required` and `init`** for properties that must be set at construction (not on domain entities, which keep public setters); no constructors that only assign.
- **`field`** for a property with light logic; no hand-written backing field.
- **Null handling:** nullable reference types are on; no `!` unless the invariant is documented next to it; `??`, `?.`, `??=`, and `?.x = y`.
- **Raw string literals** (`"""`) for multi-line text, JSON, SQL, and OpenAPI descriptions.
- **File-scoped namespaces**, `ImplicitUsings` on; add `GlobalUsings.cs` only for a slice that repeats the same usings everywhere.
- **`async` all the way:** `Task`/`ValueTask`, `CancellationToken` passed down from the endpoint.
- **`TimeProvider`** instead of `DateTime.UtcNow` when logic depends on time and a test needs to control it.

## Switch expressions and pattern matching

Prefer a switch expression to an `if`/`else` ladder: it is shorter, every arm is an expression, and the compiler checks the arms.

`src/backend/packages/<PROJECT>.Notes/PatternMatchingProbe.cs`

```csharp
using System.Diagnostics;

namespace <PROJECT>.Notes;

public enum OrderStatus
{
    Pending,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
}

public sealed record Address(string Country, string? Region);

public sealed record Customer(string Name, Address Address, int MonthsSubscribed, bool Blocked);

public abstract record Event;

public sealed record ItemAdded(int Amount) : Event;

public sealed record ItemRemoved(int Amount) : Event;

public sealed record ItemDeactivated : Event;

public static class PatternMatchingProbe
{
    // 1. Enum: name EVERY member.  The build warns (IDE0072) on a switch that leans on `_` for
    //    known members, so adding a member to the enum flags every switch that must handle it.
    //    The discard only guards values outside the enum and fails fast.
    public static string Describe(OrderStatus status) =>
        status switch
        {
            OrderStatus.Pending => "waiting to be picked",
            OrderStatus.Processing => "being prepared",
            OrderStatus.Shipped => "on its way",
            OrderStatus.Delivered => "arrived",
            OrderStatus.Cancelled => "cancelled",
            _ => throw new UnreachableException($"Unhandled {nameof(OrderStatus)}: {status}"),
        };

    // 2. Relational patterns with `or` / `and` / `not`: ranges read like the rule.
    public static string Bucket(int count) =>
        count switch
        {
            < 0 => throw new ArgumentOutOfRangeException(nameof(count)),
            0 => "none",
            1 or 2 => "few",
            > 2 and <= 10 => "several",
            _ => "many",
        };

    // 3. Property patterns, including nested ones (`Address.Country`) and `when` for the rest.
    //    Arms are tried top to bottom: put the specific cases first.
    public static string[] Channels(Customer customer) =>
        customer switch
        {
            { Blocked: true } => [],
            { MonthsSubscribed: < 3 } => ["comments"],
            { Address.Country: "DE", MonthsSubscribed: < 9 } => ["comments", "mention", "imprint"],
            { MonthsSubscribed: < 9 } => ["comments", "mention"],
            var c when c.Address.Region is null => ["comments", "mention", "ping", "region-prompt"],
            _ => ["comments", "mention", "ping"],
        };

    // 4. A tuple pattern flattens a nest of if/else on several values into one table.
    public static string Fee(bool member, int items) =>
        (member, items) switch
        {
            (true, 0) => "free",
            (true, < 5) => "reduced",
            (true, _) => "bulk",
            (false, 0) => "none",
            (false, _) => "standard",
        };

    // 5. Type patterns over a hierarchy; `null` and unknown types fail loudly.
    public static string Summarize(Event? e) =>
        e switch
        {
            ItemAdded added => $"added {added.Amount}",
            ItemRemoved removed => $"removed {removed.Amount}",
            ItemDeactivated => "deactivated",
            null => throw new ArgumentNullException(nameof(e)),
            _ => throw new UnreachableException($"Unhandled event: {e.GetType().Name}"),
        };

    // 6. List patterns: shape and contents of a sequence.  A reference-type input is nullable to
    //    the analyzer, so `null` is an arm too (here it shares the empty case with `or`).
    public static string Describe(int[]? numbers) =>
        numbers switch
        {
            null or [] => "empty",
            [var only] => $"just {only}",
            [var first, .., var last] when first == last => $"bookends of {first}",
            [var first, .., var last] => $"{first} to {last}",
        };

    // 7. `is` patterns in conditions: null checks, property checks, negation.
    public static bool HasRegion(Customer? customer) =>
        customer is { Address.Region: { Length: > 0 } };

    public static bool IsExternal(string? email) =>
        email is not null && !email.EndsWith("@example.test", StringComparison.Ordinal);
}
```

Rules:

- **Name every enum member.**  Keep a final `_ => throw new UnreachableException(...)` for values outside the enum, never a catch-all that quietly covers known members.  `.editorconfig` makes IDE0072, IDE0010 (a `switch` statement with only a `default`), and CS8524 (unnamed enum value) warnings, and `EnforceCodeStyleInBuild` makes `dotnet build` report them, so **adding an enum member flags every switch that has to decide what to do with it**.
- **Order arms from specific to general.**  An arm that an earlier arm already covers is a compile error (CS8510).
- **Reference-type input can be `null`**: give it an arm (`null => throw ...`, or `null or [] => ...`).
- **`when` only for what patterns cannot say** (comparing two properties, calling a method).  Relational, logical (`or`, `and`, `not`), property (including nested `{ Address.Country: "DE" }`), type, and list patterns cover the rest.
- A `switch` **statement** is for sequences of effects, not for choosing a value; it still needs a `default` that throws.

## Types, arguments, and tuples

`src/backend/packages/<PROJECT>.Notes/TypeChoicesProbe.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>A positional record: an immutable value with a constructor, equality, and deconstruction.</summary>
public sealed record Contact(string Name, string Email);

public static class TypeChoicesProbe
{
    // Pick the type for the job:
    //   class   -> EF entities and anything with identity and mutable state
    //   record  -> DTOs, settings, jsonb payloads, value-like data (positional = deconstructible)
    //   (named) tuple -> a private helper returning several values, no type declaration needed
    //   anonymous type -> a payload that never leaves the method
    public static string Run()
    {
        var contact = new Contact("Ada", "ada@example.test");

        var (name, email) = contact; // records deconstruct
        var (_, domain) = SplitEmail(contact); // so do named tuples

        var summary = new
        {
            name,
            domain,
            length = email.Length,
        };

        // Named arguments: use them for literals (`true`, `30`) and for more than three
        // parameters, so the call reads without opening the signature.
        return Describe(title: summary.name, pinned: true, retentionDays: 30, tags: [domain]);
    }

    private static (string Local, string Domain) SplitEmail(Contact contact)
    {
        var at = contact.Email.IndexOf('@');

        return (contact.Email[..at], contact.Email[(at + 1)..]);
    }

    private static string Describe(
        string title,
        bool pinned,
        int retentionDays,
        IReadOnlyList<string> tags
    ) => $"{title}|{(pinned ? "pinned" : "loose")}|{retentionDays}d|{string.Join(',', tags)}";

    // Side-effect-free core: the decision is a pure function of its inputs, and the effect arrives
    // as a delegate.  A test passes a lambda that records the calls; no mocking library, no I/O.
    // The caller (a handler or job) supplies the real `archiveAsync`.
    public static async Task<int> ArchiveStaleAsync(
        IReadOnlyList<Note> notes,
        DateTimeOffset now,
        TimeSpan maxAge,
        Func<Note, CancellationToken, Task> archiveAsync,
        CancellationToken ct = default
    )
    {
        var stale = notes.Where(note => now - note.CreatedAtUtc > maxAge).ToList();

        foreach (var note in stale)
        {
            await archiveAsync(note, ct);
        }

        return stale.Count;
    }
}
```

- **`class`** for EF entities and anything with identity and mutable state.  **`record`** (positional where it reads well) for DTOs, contracts, settings, `jsonb` payloads, and value-like data.  A **named tuple** for a private helper that returns several values.  An **anonymous type** only for a payload that never leaves the method.
- **Named arguments** for `bool` and number literals (`true`, `30`) and for any call with more than three parameters: the call reads without opening the signature.
- **Extension members** (`extension(T value) { ... }`, properties and static members included) for your own types and collections (`IReadOnlyList<Note>`), grouped by receiver in one `static class`; not static helper classes.  Extend a primitive (`string`, `int`) only when the member is globally applicable (`IsBlank`).
- **Builders for complex construction**, combined with extension members for a fluent, composable call (the entity-graph builder in `entity-graph.md` is one).

## Side-effect-free core

Decide in pure functions and push effects to the edge.  When logic needs an effect (archive a note, call a service), take it as a delegate (`ArchiveStaleAsync` above is the shape): a test passes a lambda that records the calls, with no mocking library and no I/O.  The caller (a handler or job) supplies the real implementation, which an integration test covers.  Use this for logic with one or two effects; an interface you own (`INoteStore`) is better when there are many operations.

## Design habits

- Small types, one per file; file name equals type name.  Partial classes only to group a slice's endpoint, contracts, and handlers by name (`NoteEndpoints.Handler.CreateNote.cs`) or to split `AppSettings` by area.
- `public` only what crosses a project boundary; `internal sealed` for implementations (stores, mappers) and `InternalsVisibleTo` for tests.
- Depend on interfaces you own (`INoteStore`); do not wrap framework types in interfaces just to mock them.
- Guard clauses and early returns; the happy path stays at the left margin.
- Prefer immutable data and pure functions; keep side effects in handlers and stores.
- Static state is for constants and loggers (`private static readonly Serilog.ILogger Log`); everything else is injected.
- Exceptions are for unexpected failures; expected outcomes are `TypedResults` (`endpoints.md`).
- No `#region`, no commented-out code, no `TODO` without a tracking reference.
- **Always brace** `if`/`else`/`for`/`foreach`/`while` bodies, even one-liners (`using var` declarations are the exception).  `csharp_prefer_braces` and IDE0011 are warnings in `.editorconfig`.
- Use `using Some.Namespace;` instead of writing full namespaces in code (aliases for the rare collision).
- No `Task.Run` or `.Result` / `.Wait()` in request code: `concurrency.md`.

## Comments and docs

Terse, direct, and about *why*.  Depth follows complexity: a one-line summary for the obvious, more for anything that spans files or surprises a reader.

- XML docs (`///`) on every public type and every contract parameter; on members when not obvious.  They feed the OpenAPI document, IntelliSense, and the generated TypeScript.  Every **type** says its purpose.
- `<summary>` is brief and to the point.  `<remarks>` carries what a reader cannot see: the **why**, business context, where the method sits in the larger flow (`<see cref="..."/>` to the caller or the related type), the setting that controls it, and the decisions behind it.
- `<param>` states constraints and expected shape, with an example for strings that have a format (`some::string::format`); `<returns>` says what comes back.  Skip a tag that only repeats the name.
- Private members get a comment when they are non-obvious; a method that builds a string shows an example of the result.
- `// NOTE:` marks a non-obvious decision, an ordering constraint, or a trap; say what breaks if it is changed.
- Pointer comments help travelers: `// 👇 Start a span here`, `// 👈 From the launch profile for the backend`.  Use sparingly, only where the next line needs a flag.
- Link the reference for anything non-obvious (`// See: https://...`).
- Do not narrate what the code says; delete comments that go stale.

Illustrative shape (not compiled):

```csharp
/// <summary>Moves an imported document into its library.</summary>
/// <remarks>
/// Runs after <see cref="ParseAsync"/> and relies on its normalized path (`/guides/intro.md`),
/// because the library keys documents by path.  The size limit comes from
/// <see cref="ImportSettings.MaxBytes"/>.
/// </remarks>
/// <param name="path">Normalized path, lowercase with hyphens: `/guides/getting-started.md`.</param>
public async Task<ImportResult> ImportAsync(string path, CancellationToken ct)
{
    // NOTE: the check must precede the write: the store enforces no size limit of its own.
    ...
}
```

## Whitespace and readability

Blank lines separate ideas; dense runs of related one-line declarations stay together.  CSharpier keeps the blank lines you write, so the choice is yours:

- **Together:** single-line declarations of the same concern (`var identity = ...; var logger = ...;`).
- **Apart:** a multi-line declaration, the move from declarations to a call or a condition, a call to an unrelated call, and **always a blank line before `return`** when the method has more than a line or two.

Illustrative (not compiled):

```csharp
var ttl = TimeSpan.FromMinutes(1);
var key = $"notes:{notebookId}";

var cached = await cache.GetAsync(key, ct);
if (cached is not null)
{
    return cached;
}

var notes = await store.ListAsync(notebookId, ct);
await cache.SetAsync(key, notes, ttl, ct);

return notes;
```

## Naming

- Types and members `PascalCase`; locals and parameters `camelCase`; private fields `_camelCase`; constants `PascalCase`.
- Async methods end in `Async`; the cancellation token parameter is `ct`.
- Interfaces `IThing`; implementations named for the technology or role (`NoteStore`), not `ThingImpl`.
- Handlers `<Verb><Thing>Handler`; endpoints `<Slice>Endpoints`; stores `<Thing>Store`; settings `<Area>Settings`.
- Tests `Entity_State_Expected` (`testing.md`).
- Database objects are `snake_case` (applied by convention); indexes `ix_<table>_<columns>`.

## Tooling

- `csharpier format src host` formats C# and the `.csproj`/`.props`/`.slnx` XML; `csharpier check` is what CI runs.  VS Code formats on save through the CSharpier extension (`.vscode/settings.json`).
- Analyzer warnings are real: fix them or suppress with a reason; do not blanket-disable.
- Warnings are errors (`TreatWarningsAsErrors`); banned APIs and custom analyzers: `static-analysis.md`.  Repeated, mechanical code: `source-generators.md`.
- `.editorconfig` raises the rules that matter to warnings (IDE0011 braces, IDE0072/IDE0010 exhaustive switches) and `EnforceCodeStyleInBuild` in `Directory.Build.props` makes the build report them.  Add a rule the same way: `dotnet_diagnostic.IDEnnnn.severity = warning`.
- Look up an unfamiliar API instead of guessing: `inspecting-packages.md`.
- `dotnet build` must be warning-free before you hand work back.
