# Test Data: Zeeq Biome

Tests need coherent, related data: a notebook with notes, an organization with members.  Hand-written `new Note { ... }` in every test is noisy, drifts when the model changes, and hides what each test is about.  [**Zeeq Biome**](https://zeeq-ai.github.io/zeeq-biome/llms.txt) is the canonical way to set it up.  A **biome** is the app's entity graph in miniature: just the rows one test needs, wired together correctly.  You declare a **blueprint** per entity once, with valid, uninteresting defaults; a test describes its scenario in one fluent chain and sets only the property it is about.

```csharp
// Unit test: no database; the entities exist as soon as each Add returns
TestBiome.Start(out _).AddNotebook(out var notebook).AddNotes(out var notes, 3);

// Integration test: pass the test's DbContext; BuildAsync() saves everything once
await TestBiome.Start(_context, out _).AddNotebook(out var notebook).AddNotes(out var notes, 3).BuildAsync();
```

Use it for **every** test that needs more than a trivial entity.  Each `Start` is a new session with its own ids, sequences, and clock, so tests never collide and run in parallel.

| Term | Meaning |
|---|---|
| **Seed** | the baseline every test starts from (an owner, a tenant id); any shape |
| **Blueprint** | the recipe for one entity type: a factory with valid defaults, plus traits and derived defaults; a root, a child of a typed parent, or built from an input |
| **Graft** | the generated `AddNotebook(out var notebook)` / `AddNotes(out var notes, 3)` methods, from a `[BiomeGraft]` catalog of blueprints |
| **Anchor** | the entities the next Add attaches to; every Add re-anchors to its own output |
| **Store** | where `BuildAsync()` saves: nothing (the default), EF Core (`_context`), or your own `IBiomeStore` |

This file is the integration guide.  The library's own docs (<https://zeeq-ai.github.io/zeeq-biome/llms.txt>) are the API reference, and it ships two agent skills for the day-to-day: `zeeq-biome-tests` (writing chains, fixing compile and runtime errors) and `zeeq-biome-blueprints` (writing seeds, blueprints, options); install them with the project (below).

## Setup (with the first entity)

Two test-only packages in `<PROJECT>.Testing`: `Zeeq.Biome` (core, no dependencies; the graft generator ships inside it as an analyzer) and `Zeeq.Biome.EntityFrameworkCore` (`EfCoreStore`, `db.AsBiomeStore()`).  Keep the two versions equal; input blueprints need 1.1.0 or later.

```xml
<!-- Directory.Packages.props -->
<PackageVersion Include="Zeeq.Biome" Version="1.1.0" />
<PackageVersion Include="Zeeq.Biome.EntityFrameworkCore" Version="1.1.0" />
```

```xml
<!-- <PROJECT>.Testing.csproj: next to Testcontainers and TUnit.Core -->
<PackageReference Include="Zeeq.Biome" />
<PackageReference Include="Zeeq.Biome.EntityFrameworkCore" />
```

`<PROJECT>.Testing` references every slice it builds.  Then the files, in `<PROJECT>.Testing/Biome/`; the seed and `TestBiome` once, a blueprint catalog per slice:

`src/backend/packages/<PROJECT>.Testing/Biome/AppSeed.cs`

```csharp
namespace <PROJECT>.Testing;

/// <summary>
/// The baseline every test starts from.  Empty until the app has something every scenario needs:
/// an owner, an organization, a tenant id.  Biome never inspects its shape; blueprints read it
/// through `ctx.Seed`.
/// </summary>
public sealed record AppSeed;
```

`src/backend/packages/<PROJECT>.Testing/Biome/TestBiome.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Zeeq.Biome;
using Zeeq.Biome.EntityFrameworkCore;

namespace <PROJECT>.Testing;

/// <summary>
/// The one entry point for test data: <c>TestBiome.Start(out var seed)</c> for a test without a
/// database, <c>TestBiome.Start(_context, out var seed)</c> to save through the test's
/// `DbContext`.  Rules and usage: the backend skill, `references/zeeq-biome.md`.
/// </summary>
public static class TestBiome
{
    // Options are shared by every test and freeze on the first `Start`: configure them here,
    // once, never inside a test.  Each `Start` is a new session, so parallel tests are safe.
    private static readonly BiomeOptions<AppSeed> Options = new();

    /// <summary>A session with no database: entities exist as soon as each Add returns.</summary>
    public static BiomeBuilder<AppSeed, AppSeed> Start(
        out AppSeed seed,
        IBiomeStore? store = null,
        TimeProvider? time = null
    )
    {
        seed = new AppSeed();

        return Biome.Start(seed, Options, store, time);
    }

    /// <summary>A session that saves through <paramref name="db"/> on `BuildAsync()`.</summary>
    public static BiomeBuilder<AppSeed, AppSeed> Start(
        DbContext db,
        out AppSeed seed,
        TimeProvider? time = null
    ) => Start(out seed, db.AsBiomeStore(), time);
}
```

`src/backend/packages/<PROJECT>.Testing/Biome/NotesBlueprints.cs`

```csharp
using <PROJECT>.Notes;
using Zeeq.Biome;

namespace <PROJECT>.Testing;

/// <summary>
/// Blueprints for the Notes slice: valid, uninteresting defaults, so a test sets only the
/// property it is about.  `[BiomeGraft]` makes the generator write `AddNotebook(out var notebook)`,
/// `AddNotes(out var notes, 3)` and the like (`NotesBlueprintsGrafts`).
/// </summary>
[BiomeGraft]
public static class NotesBlueprints
{
    public static readonly Blueprint<AppSeed, Notebook> Notebook = Blueprint.For<AppSeed, Notebook>(
        ctx => new() { Name = $"Notebook {ctx.Index + 1}", CreatedAtUtc = ctx.Now }
    );

    // A note belongs to a notebook: the typed parent, so a chain reads top to bottom.
    public static readonly ChildBlueprint<AppSeed, Notebook, Note> Note = Blueprint.ChildOf<
        AppSeed,
        Notebook,
        Note
    >(
        (notebook, ctx) =>
            new()
            {
                Title = $"Note {ctx.Index + 1}",
                NotebookId = notebook.Id,
                CreatedAtUtc = ctx.Now,
            }
    );

    // A note that is not filed anywhere (`NotebookId` is optional): a root, so it needs no notebook.
    [Graft("UnfiledNote", Plural = "UnfiledNotes")]
    public static readonly Blueprint<AppSeed, Note> Unfiled = Blueprint.For<AppSeed, Note>(ctx =>
        new() { Title = $"Note {ctx.Index + 1}", CreatedAtUtc = ctx.Now }
    );
}
```

- **Namespace `<PROJECT>.Testing`**, not `<PROJECT>.Testing.Biome`: a namespace named `Biome` would hide the `Biome` type.
- The entity already supplies its id (`Guid.CreateVersion7()`), so the blueprint leaves it.  Where it does not, use `ctx.NewId("note")` (`note_<uuidv7>`), or `Ids(...)` in the options for another shape.
- A seed with rows (an owner, a tenant) lists them in `SeedEntities`: they are saved first (below).
- Then the agent skills: `npx skills add zeeq-ai/zeeq-biome --skill zeeq-biome-tests --skill zeeq-biome-blueprints -a codex -y` (installs into `.agents/skills`, records `skills-lock.json`; review them first, as with any skill).

## The rules

1. **Customize the real entity.**  `AddNotes(out var notes, n => n.Title = "x")`.  No `*Prototype` classes that mirror an entity's properties (they duplicate every property and drift).  A customizer assigns, so a property the tests vary needs a **setter**; if the model must stay `init`-only, do not loosen it: build the row from an **input blueprint** (below).
2. **A blueprint is valid and uninteresting.**  Every required column gets a default no test should care about.  Time comes from `ctx.Now` / `ctx.Ago(t)` / `ctx.FromNow(t)`, never `DateTimeOffset.UtcNow`.  The parent comes from the typed argument, a grandparent from `ctx.Ancestor<T>()`; never wire relationships with `ctx.Last<T>()` or `ctx.All<T>()`, which break with siblings and `.Under`.
3. **Set only the property under test.**  A repeated default looks as deliberate as a chosen value; leave it in the blueprint.  A choice several tests share becomes a **trait**, a named `Action<T>` (below).
4. **Derived values are fill-only.**  `.Derive(e => e.Prop, (e, ctx) => value)` assigns only while the property is unset (`default`: `null`, `0`, `false`; **`""` counts as set**), after traits and customizers, so a test's value wins.  Leave the property unset in the factory, or `Derive` never fires.
5. **Numbers and order come from a sequence**, never `ctx.Index + 1` (it restarts on every Add and produced duplicate numbers): `.Derive(e => e.Number, Numbers)` (below).  A globally unique value uses `ctx.NewId` or a per-session sequence, never a per-parent one.
6. **Grafts, not hand-written `AddX`.**  Mark the catalog `[BiomeGraft]`; `[Graft("Name", Plural = "Names")]` renames, `[NoGraft]` excludes.  A one-off context-aware customizer (`(note, ctx) => ...`) goes through the core `Add` / `AddOne` with the blueprint, and needs `using Zeeq.Biome;`.
7. **Persistence is explicit.**  Pass the `DbContext` only in integration tests, and only the transactional base's `_context`, so everything rolls back with the test (`testing.md`).  `BuildAsync()` once, at the end of setup; then `_context.ChangeTracker.Clear()` so the code under test reads rows, not tracked instances.
8. **Options are configured once**, in `TestBiome`, and freeze on the first `Start`: changing them in a test throws.  A test that needs different options builds its own (a `NewOptions()` factory) rather than touching the shared ones.

## Reading a chain

- The anchor starts at the seed.  **Every Add re-anchors to its own output**, so a chain reads top to bottom: each child belongs to the line above.  A child Add **compiles only when the anchor is its parent type**.
- `.Under(x)` jumps back to an earlier Add output or a seed row and adds nothing: the next Add is a sibling or starts another branch.  It accepts an array too.
- **Fan-out is per parent**: `AddNotebooks(out var notebooks, 2).AddNotes(out var notes, 3)` is six notes, `notes[0..2]` in `notebooks[0]`.  Customizers fan out the same way; a comma makes a new entity per parent.
- Entities are **created immediately**, so `out` variables are usable at once; `BuildAsync()` saves.  Every Add takes an `out` (`out _` when unused).  Prefer the singular graft (`AddNote(out var note)`) to `notes[0]`.
- A **trait** is a named `Action<T>` passed as a customizer.  **A comma makes a new entity; `+` stacks traits on one entity.**

```csharp
public static class NoteTraits
{
    public static readonly Action<Note> Unfiled = note => note.NotebookId = null;
    public static readonly Action<Note> LongBody = note => note.Body = new string('x', 5_000);
}

// two notes: the first with a long body, the second with a long body and no notebook
.AddNotes(out var notes, NoteTraits.LongBody, NoteTraits.LongBody + NoteTraits.Unfiled)
```

- Biome never creates a missing ancestor.  A chain that repeats across tests becomes a **scenario helper** on the seed, with every step visible in its definition:

```csharp
public static class NotesScenarioHelpers
{
    extension(BiomeBuilder<AppSeed, AppSeed> b)
    {
        public BiomeBuilder<AppSeed, Note[]> AddNotebookWithNotes(
            out Notebook notebook,
            out Note[] notes,
            int count = 3
        ) => b.AddNotebook(out notebook).AddNotes(out notes, count);
    }
}
```

## In tests

**Handler tests**: no store, no `BuildAsync`; load a hand-written fake (`FakeNoteStore` is in `testing.md`) from what the chain returned.  TUnit creates a new class instance per test, so the constructor can start the common graph and each test extends it:

`src/backend/packages/<PROJECT>.Notes.Tests/ListNotesHandlerTests.cs`

```csharp
using <PROJECT>.Testing;
using Zeeq.Biome;

namespace <PROJECT>.Notes.Tests;

/// <summary>
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Notes.Tests --treenode-filter "/*/*/ListNotesHandlerTests/*"
/// </summary>
public sealed class ListNotesHandlerTests
{
    // TUnit creates a new instance per test, so every test gets its own session: its own seed,
    // entities, and sequences.  The constructor builds the common start; each test extends it.
    private readonly BiomeBuilder<AppSeed, Notebook[]> _start;
    private readonly Notebook _notebook;

    public ListNotesHandlerTests()
    {
        _start = TestBiome.Start(out _).AddNotebook(out _notebook);
    }

    [Test]
    public async Task ListNotes_NotebookIdSpecified_ReturnsMatchingNotes()
    {
        // Three notes in the first notebook, two in a second one.
        _start
            .AddNotes(out var inNotebook, 3)
            .AddNotebook(out var other)
            .AddNotes(out var inOther, 2);
        var store = new FakeNoteStore()
            .AddNotebooks(_notebook, other)
            .AddNotes(inNotebook)
            .AddNotes(inOther);
        var handler = new ListNotesHandler(store);

        var response = (await handler.HandleAsync(other.Id, CancellationToken.None)).Value!;

        await Assert.That(response.Items.Count).IsEqualTo(2);
        await Assert.That(response.Items.All(item => item.NotebookId == other.Id)).IsTrue();
    }

    [Test]
    public async Task ListNotes_NoNotebookId_ReturnsEveryNote()
    {
        // Notes in two notebooks (fan-out: three per parent) and one that is filed nowhere.
        TestBiome
            .Start(out _)
            .AddNotebooks(out var notebooks, 2)
            .AddNotes(out var filed, 3)
            .AddUnfiledNote(out var unfiled);
        var store = new FakeNoteStore().AddNotebooks(notebooks).AddNotes(filed).AddNotes(unfiled);
        var handler = new ListNotesHandler(store);

        var response = (await handler.HandleAsync(null, CancellationToken.None)).Value!;

        await Assert.That(response.Items.Count).IsEqualTo(7);
    }
}
```

**Store integration tests**: the chain saves the *setup*; the call under test does the rest.

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
[Test]
public async Task ListAsync_NotebookIdSpecified_ReturnsMatchingNotes()
{
    await TestBiome
        .Start(_context, out _)
        .AddNotebook(out var first)
        .AddNotes(out _, 2)
        .AddNotebook(out var second)
        .AddNotes(out _, 3)
        .BuildAsync();
    _context.ChangeTracker.Clear();

    var inFirst = await new NoteStore(_context).ListAsync(first.Id, CancellationToken.None);
    var inSecond = await new NoteStore(_context).ListAsync(second.Id, CancellationToken.None);

    await Assert.That(inFirst.Count).IsEqualTo(2);
    await Assert.That(inSecond.Count).IsEqualTo(3);
}
```

## Time

One clock per session: `ctx.Now`, `ctx.Ago(...)`, `ctx.FromNow(...)`.  Without a provider it is **frozen** when `Start` runs, so every row built in a test shares one `Now`; a test about order needs distinct times, which is a context-aware customizer (the core `Add`).  Pass a `FakeTimeProvider` (`Microsoft.Extensions.TimeProvider.Testing`) as `time` to read live and advance it, and pass the **same** instance to the code under test.  Biome truncates its readings to **1 µs**, Postgres's precision, so a value it built equals the value read back; override with `TimePrecision` in the options.

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
[Test]
public async Task ListAsync_DistinctCreatedAtUtc_ReturnsNewestFirst()
{
    // All notes share one frozen `Now`, so the order under test needs distinct times:
    // `ctx.Ago(...)` needs the context-aware form, which goes through the core `Add`.
    await TestBiome
        .Start(_context, out _)
        .AddNotebook(out _)
        .Add(
            NotesBlueprints.Note,
            out var notes,
            (note, ctx) => note.CreatedAtUtc = ctx.Ago(TimeSpan.FromMinutes(10)),
            (note, ctx) => note.CreatedAtUtc = ctx.Ago(TimeSpan.FromMinutes(1)),
            (note, ctx) => note.CreatedAtUtc = ctx.Ago(TimeSpan.FromMinutes(5))
        )
        .BuildAsync();
    _context.ChangeTracker.Clear();

    var listed = await new NoteStore(_context).ListAsync(null, CancellationToken.None);

    await Assert
        .That(listed.Select(note => note.Id))
        .IsEquivalentTo([notes[1].Id, notes[2].Id, notes[0].Id], CollectionOrdering.Matching);
}
```

```csharp
[Test]
public async Task Note_CreatedAtUtc_SurvivesTheDatabaseRoundTrip()
{
    // An instant with 100 ns ticks, below Postgres's microsecond precision.  Biome rounds the
    // clock it hands out, so the value it built equals the value read back.
    var time = new FakeTimeProvider(
        new DateTimeOffset(2026, 10, 11, 12, 0, 0, TimeSpan.Zero).AddTicks(1_234_567)
    );
    await TestBiome
        .Start(_context, out _, time)
        .AddNotebook(out _)
        .AddNote(out var note)
        .BuildAsync();
    _context.ChangeTracker.Clear();

    var saved = (await new NoteStore(_context).ListAsync(null, CancellationToken.None)).Single();

    await Assert.That(saved.CreatedAtUtc).IsEqualTo(note.CreatedAtUtc);
}
```

A timestamp the code under test creates (a store that stamps `UtcNow`) is not Biome's: truncate the expected side (`testing.md`, "Timestamps through the database").

## Sequences

Declare the sequence once, above the blueprint that uses it (static fields initialize top to bottom; keep a blueprint and its variants in one file), and fill the property with `Derive`.  `.Per<TAncestor>()` scopes it to the nearest ancestor of a type, `.PerSession()` is the default.

An example catalog member (a ticket numbered within its project):

```csharp
// Declared above the blueprint that uses it: static fields initialize top to bottom.
public static readonly Sequence<int> TicketNumbers = Sequence.Of(1, n => n + 1).PerParent(); // restarts per project

public static readonly ChildBlueprint<AppSeed, Project, Ticket> Ticket = Blueprint
    .ChildOf<AppSeed, Project, Ticket>(
        (project, ctx) => new() { ProjectId = project.Id, Title = "A ticket" }
    )
    .Derive(ticket => ticket.Number, TicketNumbers); // fill-only: a test can set the number
```

```csharp
TestBiome
    .Start(out _)
    .AddProject(out var project)
    .AddTickets(out var first, 2) // numbers 1, 2
    .Under(project)
    .AddTicket(out var third) // 3, not 1: the sequence continues
    .Under(project)
    .AddTicket(out var four, t => t.Number = 4) // explicit: kept, and skipped from now on
    .Under(project)
    .AddTicket(out var next) // 5
    .AddProject(out _) // another project...
    .AddTicket(out var otherFirst); // ...restarts at 1
```

Explicit values are reserved and skipped within the step, so set a required number in the same step as the defaults or earlier.  A non-numeric sequence works too (a rank: `Sequence.Of("a", r => r + "a").PerParent()`; leave the property `null`, since `""` counts as set).

## Transients: the input of the call under test

Build an entity with valid defaults without saving it: `.Transient()` right after the step it marks (or `blueprint.AsTransient()` for a type that is always an input).  It is still returned and inspectable (`biome.All<T>()`).

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
[Test]
public async Task CreateAsync_TransientBiomeInput_PersistsNote()
{
    // The note is the input of the call under test: built with valid defaults, never saved by
    // the chain (`.Transient()` marks the step above it).
    await TestBiome
        .Start(_context, out _)
        .AddNotebook(out var notebook)
        .AddNote(out var input, note => note.Title = "Hello")
        .Transient()
        .BuildAsync();
    var store = new NoteStore(_context);

    await store.CreateAsync(input, CancellationToken.None);
    _context.ChangeTracker.Clear();

    var saved = (await store.ListAsync(notebook.Id, CancellationToken.None)).Single();

    await Assert.That(saved.Title).IsEqualTo("Hello");
}
```

EF saves anything reachable through a navigation from a saved row, transient or not: do not link persisted rows to a transient input.

## Seeds, tenancy, and defaults

The seed is created outside the session (so it cannot use `ctx.Now`) and can be any shape.  Rows in it are saved before the created entities if listed in `SeedEntities`, in foreign-key-safe order, and then count as parents for `.Under(seed.Tenant)`.  A **multi-tenant** app puts the tenant in the seed and every blueprint stamps it (`data-efcore.md`, "Multi-tenant data"):

```csharp
public sealed record TenantSeed(Tenant Tenant); // the row, and the id every blueprint stamps

// In TestBiome: the seed's row is saved first, and `.Under(seed.Tenant)` can anchor on it.
private static readonly BiomeOptions<TenantSeed> Options =
    new BiomeOptions<TenantSeed>().SeedEntities(seed => [seed.Tenant]);
// Start: `seed = new TenantSeed(new Tenant()); return Biome.Start(seed, Options, store);`
```

```csharp
public static readonly Blueprint<TenantSeed, User> User = Blueprint.For<TenantSeed, User>(ctx =>
    new() { TenantId = ctx.Seed.Tenant.Id, Email = $"user{ctx.Index + 1}@example.test" }
);
```

Options also hold `Default<TBase, TProp>(e => e.Prop, (e, ctx) => ...)`, a fill-only default for every entity of a base type or interface (every `IUpdatedEntity` gets `UpdatedAtUtc`), and `BeforeSave` / `AfterSave` hooks that see the final row list.  Rows added in a hook skip derives and defaults, and hidden rows make a test harder to read: keep hooks for rules the app always requires.

## Init-only entities: input blueprints

A customizer cannot assign an `init` property (`CS8852`), and production models should not change to suit tests.  An **input blueprint** moves customization earlier: the test customizes a small settable input, `Derive` fills the input, and the factory builds the row.  It also builds one input into several rows (an aggregate: an output that implements `IBiomeAggregate`).  Needs `Zeeq.Biome` 1.1.0; use a plain blueprint for a settable entity.

```csharp
public sealed class Receipt // init-only: a customizer cannot assign these
{
    public required Guid Id { get; init; }
    public required decimal Total { get; init; }
}

public sealed class ReceiptInput // construction data: settable, nothing required
{
    public decimal Subtotal { get; set; }
    public decimal TaxRate { get; set; }
}

// in a [BiomeGraft] catalog (ReceiptBlueprints):
public static readonly InputBlueprint<AppSeed, ReceiptInput, Receipt> Receipt = Blueprint
    .FromInput<AppSeed, ReceiptInput, Receipt>(
        () => new(),
        (input, _) =>
            new() { Id = Guid.CreateVersion7(), Total = input.Subtotal * (1 + input.TaxRate) }
    )
    .Derive(input => input.Subtotal, (_, _) => 100m); // fills the input, in order
```

```csharp
TestBiome.Start(out _).AddOne(ReceiptBlueprints.Receipt, out var receipt, i => i.TaxRate = 0.25m); // Total 125
```

`Derive` treats `default` as unset on an input too: an explicit `0` or `false` on a plain `int` or `bool` is refilled.

## When to use it, and when not to

Biome is for **preconditions**: rows the test needs but is not about.  It is not a second ORM and must not hide the behavior under test.

- **Do not build what the code under test creates.**  A create-path test starts from an empty store and asserts what was created; build only what the call needs (its notebook), or the input, as a transient.
- **Keep construction direct when the entity is the input** and its exact values are the test (`new Note { Title = "   " }` for a guard test).
- **Keep constraint tests direct.**  A foreign-key or unique-constraint test inserts the offending row by hand; a blueprint builds valid rows and would hide what is proven.
- **Keep read models direct.**  Biome builds entities, not projections or response DTOs.
- **State a precondition you depend on** with an assertion before the act step when the chain does not make it obvious.
- **One session per scenario**: one `Start`, one `BuildAsync()`, small, built inside the test or its constructor, never shared between tests.
- **Test the blueprints once**: a store test that saves each blueprint's defaults (`NotesBlueprintsTests`) fails once when a new required column appears, instead of in every test.

## Mistakes

| Symptom | Cause and fix |
|---|---|
| `CS1929 ... requires a receiver of type 'BiomeBuilder<AppSeed, Notebook[]>'` | a child graft exists only after its parent: add the parent, `.Under(parent)`, or use a scenario helper |
| `CS0411` on `.Add(Blueprint, out ..., (x, ctx) => ...)` | the core child `Add` / `AddOne` are extension methods in `Zeeq.Biome`: add `using Zeeq.Biome;` |
| `CS1593 Delegate 'Action<T>' does not take 2 arguments` | one- and two-parameter lambdas mixed in a call, or a `(x, ctx)` lambda given to a graft: write every customizer in the `(x, ctx)` form and call the core `Add` |
| `CS8852` init-only property | the property is `init`: an input blueprint, not a setter |
| `Parent at index N ... is not owned by this biome session` | `.Under` got an object this session did not build: use an `out` from an Add or a `SeedEntities` row |
| `A singular child AddOne requires exactly one parent` | the anchor holds several parents: the plural Add, or `.Under(parents[0])` |
| `No ancestor of type X exists` | the chain skipped X: build the full chain, or use a scenario helper |
| every later call throws, with the first exception inside | an Add failed and the session is faulted: fix the first exception, start a new session |
| `BuildAsync may only be called once` | a second build, or an Add after it: all setup before one `BuildAsync()` |
| `Biome options are frozen` | a test changed shared options |
| a unique violation on a number | an explicit value in a **later** step than the defaults that used it: set it in the same step or earlier |
| a timestamp differs by microseconds after reading back | `DateTimeOffset.UtcNow` in setup: use `ctx.Now` / `ctx.Ago`, or a `FakeTimeProvider` shared with the code under test |
| a `.Transient()` row was saved | a saved row references it through a navigation (EF follows it) |
| `.Transient()` marked the wrong rows | it marks the session's most recent Add step: put it straight after the step |
| `biome.All<T>()` has unexpected rows | it includes transients and aggregate wrappers, not seed rows: assert on `out` variables |

## Moving from a hand-written builder

A hand-rolled entity-graph builder (a fluent builder with `*Prototype` classes or a `SeedContext`) maps one to one:

| Hand-written builder | Zeeq Biome |
|---|---|
| `AddGeneratedSeed(_context)` and a `SeedContext` | `TestBiome.Start(_context, out var seed)`; the old seed can be the seed type |
| a `*Prototype` class plus `Create*` and overloads | one blueprint, `Derive`s, and generated grafts; an input blueprint where the entity is `init`-only |
| `FindLatest<T>(...)` to find a parent | a child blueprint's typed parent and `ctx.Ancestor<T>()` |
| `index + 1` numbering | a sequence with `Derive` |
| `PersistOnBuild = false` / `NotPersisted()` | `.Transient()` |
| `var (seed, notes) = await ...BuildAsync()` | `out` variables; `BuildAsync()` returns nothing |
| a result record that implements an `IEntityGraph` | an input blueprint with an `IBiomeAggregate` output |
| `DateTimeOffset.UtcNow` and `TruncateToPostgresPrecision()` | `ctx.Now` (rounded) or a shared `FakeTimeProvider` |

Both can live side by side until the last call site has moved.  The library's `zeeq-biome-blueprints` skill has a step-by-step migration guide (its `migration-from-entitygraph` reference).  Then delete the builder, its extensions, and its tests.
