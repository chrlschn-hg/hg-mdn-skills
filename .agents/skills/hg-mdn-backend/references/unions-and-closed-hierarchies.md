# Unions and Closed Hierarchies (C# 15)

**Expected outcomes are values, not exceptions.**  Core and library code returns a union (or a closed hierarchy) that names every outcome.  The compiler makes each caller handle all of them, and a new outcome breaks every `switch` that ignores it.  Exceptions stay for bugs and infrastructure failures (last section).

C# 15 is the default for `net11.0`: no `LangVersion`.  Every example here compiled and ran on SDK 11.0.100 with `TreatWarningsAsErrors`; the compiler codes are the ones the compiler printed.  Language reference: [unions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/union), [`closed`](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/closed).

## Choose

| The value is | Use | Example |
|---|---|---|
| one of several outcomes of an operation (success, not found, conflict, rejected) | **union** of small records | `CreateNoteOutcome` |
| one of several unrelated types in one slot | **union** | `NoteKeyParse(Guid, ShortId, InvalidKey)` |
| a variant with shared data or behavior: a state, an event, a message | **`closed` record hierarchy** | `NoteEvent`, `NotebookState` |
| a name from a fixed list, no payload | `enum` (switch names every member: `code-style.md`) | `OrderStatus` |
| a bug, a broken invariant, an infrastructure failure | exception | `ArgumentNullException`, `UnreachableException`, a dead database |

Union or closed: union cases need no common base (`Guid`, `string`, a third-party type) and share nothing.  Closed cases derive from one base, so they share its members and the base is what you pass around.  A closed hierarchy can grow inside its own assembly; a union is fixed by its declaration.  A state is closed; the **outcome** of a transition is a union (below).

## Unions

A library function that returns what happened instead of throwing or filling an `out` parameter:

`src/backend/packages/<PROJECT>.Notes/NoteKeys.cs`

```csharp
namespace <PROJECT>.Notes;

public sealed record ShortId(string Value);

public sealed record InvalidKey(string Reason);

// Cases need no common base and may be value types (`Guid` is boxed into `Value`).
public union NoteKeyParse(Guid, ShortId, InvalidKey);

public static class NoteKeys
{
    // Each arm converts to the union implicitly (the return type is the target type).
    public static NoteKeyParse Parse(string? text) =>
        text switch
        {
            null or "" => new InvalidKey("A note key is required."),
            _ when Guid.TryParse(text, out var id) => id,
            _ when text.Length <= 12 && text.All(char.IsAsciiLetterOrDigit) => new ShortId(text),
            _ => new InvalidKey("Neither a UUID nor a short id."),
        };
}
```

`src/backend/packages/<PROJECT>.Notes.Tests/NoteKeysTests.cs`

```csharp
namespace <PROJECT>.Notes.Tests;

public sealed class NoteKeysTests
{
    [Test]
    public async Task Parse_UuidText_ReturnsGuidValue()
    {
        var id = Guid.CreateVersion7();

        var parsed = NoteKeys.Parse(id.ToString());

        await Assert.That(parsed.Value).IsEqualTo(id);
    }

    [Test]
    [Arguments("Ab3dEf")]
    [Arguments("x")]
    public async Task Parse_ShortIdText_ReturnsShortIdValue(string text)
    {
        var parsed = NoteKeys.Parse(text);

        // `is` on a union tests the contents, so patterns read like the case they match.
        await Assert.That(parsed is ShortId(var value) && value == text).IsTrue();
    }

    [Test]
    [Arguments(null)]
    [Arguments("")]
    [Arguments("not a key, and far too long")]
    public async Task Parse_UnsupportedText_ReturnsInvalidKeyWithoutException(string? text)
    {
        var parsed = NoteKeys.Parse(text);

        await Assert.That(parsed.Value).IsTypeOf<InvalidKey>();
    }
}
```

The rules (each one verified):

- **Conversion is implicit from each case type**; nested unions do not flatten: `Outer o = (Pet)dog` needs the inner union first.
- **Exhaustive `switch`.**  A missing case is `CS8509` (an error under `TreatWarningsAsErrors`): "the pattern 'C' is not covered".  Never write `_` or `var` as the last arm: they accept every future case and the compiler stops warning.
- **`default(U)` has a `null` `Value`.**  When every case is non-nullable the compiler does *not* ask for a `null` arm, and a switch without one throws `SwitchExpressionException` at run time.  The template's **IDE0072** (a build error) does ask for it.  Give the switch a `null` arm that states the bug (`null => throw new InvalidOperationException(...)`).
- **`is` and `switch` patterns test the contents**: `outcome is Note n`, `outcome is Note { Title: "x" }`.  A pattern for the union itself (`outcome is CreateNoteOutcome`) is `CS8121`; test `is not null` or use `var`.
- **A union is a struct**: `Value` is `object?`; value-type cases are boxed; `ToString()` returns the type name, so log `outcome.Value`.
- **The case list cannot repeat a type** (`CS0111`), and a generic `T` that equals another case fails where it is used (`CS0457`, ambiguous conversion): do not build a generic `Result<T, TError>`; name each operation's outcomes.
- A body holds methods and properties, no instance fields (`Value` is the only state).

## Outcomes instead of exceptions

A core operation: one union per operation, the success case is the domain type, every failure is a small `sealed` record named for what happened and carrying what the caller needs:

`src/backend/packages/<PROJECT>.Notes/CreateNoteOutcome.cs`

```csharp
namespace <PROJECT>.Notes;

public sealed record TitleRequired;

public sealed record TitleTooLong(int Max);

public sealed record BodyTooLong(int Max);

public sealed record NotebookNotFound(Guid NotebookId);

/// <summary>Every way <see cref="NoteCreator.CreateAsync"/> can end.  This list is the contract.</summary>
public union CreateNoteOutcome(Note, TitleRequired, TitleTooLong, BodyTooLong, NotebookNotFound);
```

`src/backend/packages/<PROJECT>.Notes/NoteCreator.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// The note-creation rules, free of HTTP, so a handler, an import, and a job share them.
/// </summary>
public sealed class NoteCreator(INoteStore store)
{
    public async Task<CreateNoteOutcome> CreateAsync(
        string? title,
        string? body,
        Guid? notebookId,
        CancellationToken ct
    )
    {
        // No `throw`, no `catch`: a rule that fails is a return value.
        if (string.IsNullOrWhiteSpace(title))
        {
            return new TitleRequired();
        }

        if (title.Length > NoteLimits.MaxTitleLength)
        {
            return new TitleTooLong(NoteLimits.MaxTitleLength);
        }

        if (body?.Length > NoteLimits.MaxBodyLength)
        {
            return new BodyTooLong(NoteLimits.MaxBodyLength);
        }

        if (notebookId is { } id && !await store.NotebookExistsAsync(id, ct))
        {
            return new NotebookNotFound(id);
        }

        var note = await store.CreateAsync(
            new Note
            {
                Title = title.Trim(),
                Body = body ?? "",
                NotebookId = notebookId,
            },
            ct
        );

        return note;
    }
}
```

The handler is the only code that knows HTTP.  It replaces the guard-clause handler in `endpoints.md` once the rules live in core code.  It maps each outcome to a `TypedResults` (the same `Results<...>`), and the switch is exhaustive, so adding an outcome to the union fails the build here until a status is chosen:

`src/backend/packages/<PROJECT>.Notes/NoteEndpoints.Handler.CreateNote.cs (variant)`

```csharp
using <PROJECT>.Api;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Notes;

public sealed class CreateNoteHandler(NoteCreator creator) : IEndpointHandler
{
    public async Task<
        Results<Created<NoteResponse>, BadRequest<ApiError>, NotFound<ApiError>>
    > HandleAsync(CreateNoteRequest request, CancellationToken ct)
    {
        var outcome = await creator.CreateAsync(
            request.Title,
            request.Body,
            request.NotebookId,
            ct
        );

        // The arms are target-typed to `Results<...>`; the arms and the return type must agree.
        return outcome switch
        {
            Note note => TypedResults.Created($"/notes/{note.Id}", note.ToResponse()),
            TitleRequired => TypedResults.BadRequest(
                new ApiError("invalid_title", "Title is required.")
            ),
            TitleTooLong(var max) => TypedResults.BadRequest(
                new ApiError("title_too_long", $"Title may be at most {max} characters.")
            ),
            BodyTooLong(var max) => TypedResults.BadRequest(
                new ApiError("body_too_long", $"Body may be at most {max} characters.")
            ),
            NotebookNotFound => TypedResults.NotFound(
                new ApiError("notebook_not_found", "The notebook does not exist.")
            ),
            null => throw new InvalidOperationException("default(CreateNoteOutcome) is a bug."),
        };
    }
}
```

Test the operation (no HTTP) and the mapping (one test per outcome):

`src/backend/packages/<PROJECT>.Notes.Tests/NoteCreatorTests.cs`

```csharp
using <PROJECT>.Api;
using <PROJECT>.Testing;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Notes.Tests;

public sealed class NoteCreatorTests
{
    [Test]
    public async Task CreateAsync_ValidRequest_ReturnsStoredNote()
    {
        var store = new FakeNoteStore();

        var outcome = await new NoteCreator(store).CreateAsync(
            "  Hello ",
            null,
            null,
            CancellationToken.None
        );

        // Narrow with a pattern, then assert on the payload.
        await Assert.That(outcome is Note { Title: "Hello" }).IsTrue();
        await Assert.That(store.Notes).HasSingleItem();
    }

    [Test]
    public async Task CreateAsync_UnknownNotebook_ReturnsNotFoundOutcome()
    {
        TestBiome.Start(out _).AddNotebook(out var notebook);
        var store = new FakeNoteStore().AddNotebooks(notebook);
        var missing = Guid.NewGuid();

        var outcome = await new NoteCreator(store).CreateAsync(
            "Hello",
            null,
            missing,
            CancellationToken.None
        );

        await Assert.That(outcome.Value).IsEqualTo(new NotebookNotFound(missing));
        await Assert.That(store.Notes).IsEmpty();
    }

    [Test]
    public async Task HandleAsync_OverlongTitle_ReturnsBadRequestWithTitleTooLongCode()
    {
        var handler = new CreateNoteHandler(new NoteCreator(new FakeNoteStore()));

        var result = await handler.HandleAsync(
            new(new string('x', NoteLimits.MaxTitleLength + 1)),
            CancellationToken.None
        );

        var badRequest = result.Result as BadRequest<ApiError>;

        await Assert.That(badRequest).IsNotNull();
        await Assert.That(badRequest!.Value!.Code).IsEqualTo("title_too_long");
    }
}
```

## Translate at the edge, once

A third-party exception that means an expected outcome becomes a union case in the **one** adapter that knows it, and nowhere else.  A unique violation is the common one.  The slice declares the outcome and the store contract:

`src/backend/packages/<PROJECT>.Notes/RenameNotebookOutcome.cs`

```csharp
namespace <PROJECT>.Notes;

public sealed record NameTaken(string Name);

// `NotebookNotFound` is shared with `CreateNoteOutcome`: a case is a plain record, reuse it.
public union RenameNotebookOutcome(Notebook, NotebookNotFound, NameTaken);
```

`src/backend/packages/<PROJECT>.Notes/INoteStore.cs (excerpt)`

```csharp
    /// <summary>Renames a notebook.  A taken name is an outcome, not an exception.</summary>
    Task<RenameNotebookOutcome> RenameNotebookAsync(Guid id, string name, CancellationToken ct);
```

Adding a method to `INoteStore` breaks `FakeNoteStore` (`CS0535`) until it implements it: a fake returns the same cases.  `Data` owns the constraint and the translation:

`src/backend/packages/<PROJECT>.Data/Configurations/NotebookConfiguration.cs`

```csharp
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace <PROJECT>.Data.Configurations;

internal sealed class NotebookConfiguration : IEntityTypeConfiguration<Notebook>
{
    public const string UniqueName = "ux_notebooks_name";

    public void Configure(EntityTypeBuilder<Notebook> entity)
    {
        entity.ToTable("notebooks");
        entity.HasKey(notebook => notebook.Id);
        entity.Property(notebook => notebook.Name).HasMaxLength(NoteLimits.MaxNotebookNameLength);
        entity.HasIndex(notebook => notebook.Name).IsUnique().HasDatabaseName(UniqueName);
    }
}
```

`src/backend/packages/<PROJECT>.Data/Notes/NoteStore.cs (excerpt)`

```csharp
    public async Task<RenameNotebookOutcome> RenameNotebookAsync(
        Guid id,
        string name,
        CancellationToken ct
    )
    {
        var notebook = await db.Set<Notebook>().FindAsync([id], ct);

        if (notebook is null)
        {
            return new NotebookNotFound(id);
        }

        notebook.Name = name;

        try
        {
            await db.SaveChangesAsync(ct);
            return notebook;
        }
        // Translate the one constraint this method owns.  Every other `DbUpdateException`
        // (a dead connection, a lock timeout) is not an outcome: it propagates.
        catch (DbUpdateException ex)
            when (ex.InnerException
                    is PostgresException { SqlState: PostgresErrorCodes.UniqueViolation } pg
                && pg.ConstraintName == NotebookConfiguration.UniqueName
            )
        {
            // The failed save leaves the entity tracked and modified: drop it so the
            // context is reusable.
            db.ChangeTracker.Clear();

            return new NameTaken(name);
        }
    }
```

The store test: two notebooks, rename the second to the first's name.  Postgres aborts the surrounding transaction on a violation, so the outcome is the last thing the test reads:

`src/backend/packages/<PROJECT>.Data.Tests/RenameNotebookTests.cs`

```csharp
using <PROJECT>.Data.Notes;
using <PROJECT>.Notes;
using <PROJECT>.Testing;

namespace <PROJECT>.Data.Tests;

[Property("integration", "true")]
[Property("testcontainer", "true")]
[ClassDataSource<PgDatabaseFixture>(Shared = SharedType.PerTestSession)]
public sealed class RenameNotebookTests(PgDatabaseFixture pg) : PgTransactionalTestBase(pg)
{
    [Test]
    public async Task RenameNotebookAsync_TakenName_ReturnsNameTakenOutcome()
    {
        await TestBiome.Start(_context, out _).AddNotebooks(out var notebooks, 2).BuildAsync();
        var store = new NoteStore(_context);

        var outcome = await store.RenameNotebookAsync(
            notebooks[1].Id,
            notebooks[0].Name,
            CancellationToken.None
        );

        await Assert.That(outcome.Value).IsEqualTo(new NameTaken(notebooks[0].Name));
    }

    [Test]
    public async Task RenameNotebookAsync_MissingNotebook_ReturnsNotFoundOutcome()
    {
        var store = new NoteStore(_context);
        var missing = Guid.NewGuid();

        var outcome = await store.RenameNotebookAsync(missing, "Anything", CancellationToken.None);

        await Assert.That(outcome.Value).IsEqualTo(new NotebookNotFound(missing));
    }
}
```

## Closed hierarchies

`closed` on a class (or record) limits its **direct** subtypes to the declaring assembly, so the compiler knows the full list.  Use it where the variants share data or behavior:

`src/backend/packages/<PROJECT>.Notes/NoteEvent.cs`

```csharp
namespace <PROJECT>.Notes;

// Shared data sits on the base.  Leaves are `sealed`: another assembly may not extend them.
public closed record class NoteEvent(Guid NoteId, DateTimeOffset AtUtc);

public sealed record NoteCreated(Guid NoteId, DateTimeOffset AtUtc, string Title)
    : NoteEvent(NoteId, AtUtc);

public sealed record NoteMoved(Guid NoteId, DateTimeOffset AtUtc, Guid? From, Guid? To)
    : NoteEvent(NoteId, AtUtc);

public sealed record NoteDeleted(Guid NoteId, DateTimeOffset AtUtc) : NoteEvent(NoteId, AtUtc);

public static class NoteActivity
{
    // Exhaustive with no `_`: add a NoteEvent subtype and this stops compiling (CS8509).
    // The `null` arm is the caller's bug, not a catch-all (the template's IDE0072 asks for it).
    public static string Describe(NoteEvent e) =>
        e switch
        {
            NoteCreated(_, _, var title) => $"created \"{title}\"",
            NoteMoved { From: null, To: { } to } => $"filed in {to}",
            NoteMoved moved => $"moved from {moved.From} to {moved.To}",
            NoteDeleted => "deleted",
            null => throw new ArgumentNullException(nameof(e)),
        };
}
```

A state is closed, the outcome of changing it is a union, and the transition is a switch over the state:

`src/backend/packages/<PROJECT>.Notes/NotebookState.cs`

```csharp
namespace <PROJECT>.Notes;

public closed record class NotebookState;

public sealed record Active : NotebookState;

public sealed record Archived(DateTimeOffset AtUtc) : NotebookState;

public sealed record Deleted(DateTimeOffset AtUtc) : NotebookState;

public sealed record Rejected(string Reason);

public union ArchiveOutcome(Archived, Rejected);

public static class NotebookLifecycle
{
    public static ArchiveOutcome Archive(NotebookState state, TimeProvider time) =>
        state switch
        {
            Active => new Archived(time.GetUtcNow()),
            Archived => new Rejected("Already archived."),
            Deleted => new Rejected("A deleted notebook cannot be archived."),
            null => throw new ArgumentNullException(nameof(state)),
        };
}
```

The rules (each one verified):

- **A `closed` type is implicitly abstract**: `closed abstract` is `CS9384`; `closed interface` is `CS0106` (classes and records only).
- **Derivation outside the assembly is `CS9382`**, but the rule is not transitive: a non-`sealed`, non-`closed` leaf can still be extended elsewhere, so seal every leaf.
- **Nesting works**: a `closed` intermediate (`closed record Rounded : Shape`) lets a switch name either `Rounded` or its children.
- **Generic closed types work** (`closed record class Tree<T>`, `Leaf<T>`, `Node<T>`).  A derived type may not add a type parameter its closed base does not use (`CS9383`).
- **A union may name a closed type as a case**, and the switch is exhaustive over the closed type's leaves; omit one and `CS8509` names it.
- **A switch over a closed type needs a `null` arm too**: the compiler accepts the switch without one, IDE0072 does not.  `null => throw new ArgumentNullException(nameof(e))` is a bug report, not a catch-all, so a missing subclass is still `CS8509`.
- `closed` gives exhaustiveness only.  It does **not** make JSON polymorphic: a hierarchy sent over the wire still needs `[JsonPolymorphic]` and `[JsonDerivedType]`.

## At the HTTP boundary

Verified on ASP.NET Core 11 with `System.Text.Json`:

| A union is... | Result |
|---|---|
| returned from an endpoint | `200` and the JSON of the case value only (no discriminator); the OpenAPI schema is `anyOf` the cases |
| a property of a response record | the same, nested |
| in a request body | cases of distinct JSON kinds deserialize (`string` + object, `int` + object, `bool`); two object-shaped cases, or a string beside a number case (the web defaults read quoted numbers), throw `JsonException` ("ambiguous ... specify a custom type classifier") |

So **map a domain union to explicit contract types in the handler** (`Results<...>` above) and keep the union inside.  A union in a contract is for cases a client can tell apart by JSON kind or by shape; give object-shaped cases a `kind` property.

## What stays an exception

| Throw | Because |
|---|---|
| `ArgumentNullException.ThrowIfNull`, `ArgumentOutOfRangeException` | the caller is wrong: a bug |
| `UnreachableException` (the `_` arm of an enum switch), `InvalidOperationException` (the `null` arm of a union) | the code reached a state its types forbid |
| `OperationCanceledException` | cancellation propagates; it is never an outcome |
| a dead database, a timeout, a failed deploy | infrastructure; `HttpExceptionHandler` answers once at the boundary (`endpoints.md`) |

Do not:

- `catch` to choose a branch, or `throw` for "not found", "invalid", or "conflict".
- Return `null` when there is more than one way to fail.  A single, obvious absence is still `Task<Note?> FindAsync(...)`.
- Hand-write `Result<T>` or `Either<L, R>` classes, or share one error base across slices: each operation names its own outcomes.
- End a switch over a union with `_` or `var`.

## Compiler and tooling notes

| You see | Cause | Fix |
|---|---|---|
| `CS8509` on a switch | a case is not handled | add the arm; never `_` |
| `IDE0072` on a switch over a union or a closed type | no `null` arm | add `null => throw ...` (a bug), not `_` |
| `CS0457` ambiguous conversion | a generic `T` case equals another case | name the cases; no generic `Result<T>` |
| `CS8121` | `x is TheUnit` | `x is not null`, or a case pattern |
| `CS9384`, `CS0106` | `closed abstract`; `closed` on an interface | `closed class` or `closed record` only |
| `CS9382` | a subtype of a closed type in another assembly | add the subtype beside the base, or use a union |
| `;` expected on `outer: ...`, `union` | CSharpier older than 1.4.0 | pin CSharpier 1.4.0 or newer |

Assert on a union through `outcome.Value`: TUnit's `IsTypeOf<T>()` and `IsEqualTo(...)` work on it as on any `object?`.
