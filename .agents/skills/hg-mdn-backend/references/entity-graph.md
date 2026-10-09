# Test Data: the Entity Graph Builder

Tests need coherent, related data: a notebook with notes, an organization with members.  Writing `new Note { ... }` by hand in every test is noisy, drifts when the model changes, and hides what each test is about.  The **entity graph builder** is the canonical way to set up test data: a terse, fluent builder that creates a seed, adds entities that hang off it, and returns them as a typed tuple.

```csharp
// Unit test: no database; entities exist in memory only
var (seed, notes) = await EntityGraph.AddGeneratedSeed().AddNotes(3).BuildAsync();

// Integration test: pass the test's DbContext and the graph is saved when BuildAsync() runs
var (seed, notes) = await EntityGraph.AddGeneratedSeed(_context).AddNotes(3).BuildAsync();
```

One mechanism feeds both kinds of test.  Use it for **every** test that needs more than a trivial entity.  Each test builds its own unique data (ids and names are generated), so tests never collide.

## The rules

1. **Customize the real entity with `Action<T>`.**  `AddNotes(note => note.Title = "x")`.  Do **not** create `*Prototype` classes that re-declare an entity's properties so a builder can copy them across: that duplicates every property and its defaults and drifts silently when the entity changes.  So entities used with the builder keep **public setters** (`{ get; set; }`; an `init`-only property cannot be customized).
2. **The seed is the ambient context.**  `SeedContext` holds the entities almost every test needs (here, one notebook).  Defaults for everything added later hang off it, so a bare `AddNotes(3)` is already a coherent graph.  Grow the seed as the domain grows (an organization, an owner, ...).
3. **Defaults are valid and unique.**  Ids come from the entity (`Guid.CreateVersion7()`), names are numbered by index, so unique constraints hold across `AddNotes(3)` and across tests.
4. **One extension block per slice** (`EntityGraph<Slice>Extensions.cs`) keeps the core generic.  Each entity gets the same overload set: `AddThings()`, `AddThings(int count)`, `AddThings(params Action<Thing>[] setup)`, and optionally `AddThings(params Action<Thing, EntityGraphContext>[] setup)` to read earlier entities.  Two `params` overloads make `AddThings()` ambiguous, so the no-argument call gets its own (`AddThings() => AddThings(1)`).
5. **A result record only when one call creates linked entities** and the test needs them back (`NotebookWithNotes`).  It implements `IEntityGraph` so the builder can save it.
6. **Options that are not entity properties are builder methods**: `.NotPersisted()` keeps what was just added in the result but does not save it (an "exists in memory, not in the database" case).
7. **Persistence is explicit.**  Pass the `DbContext` only in integration tests, and only the transactional base's `_context`, so everything is rolled back with the test (`testing.md`).

## When to use it, and when not to

The graph is for **preconditions**: rows the test needs but is not about.  It is not a second ORM and must not hide the behavior under test.

- **Use it** for the normal graph a test starts from: a seed, three notes, notes spread across notebooks.
- **Do not seed what the code under test creates.**  A create-path test (`CreateNote`) starts from an empty store and asserts what was created.
- **Keep direct construction when the entity is the input** to the method under test (`new Note { ... }` passed to `CreateAsync`).
- **Keep constraint tests direct.**  A foreign-key or unique-constraint test inserts the offending row by hand (a `Note` with an unknown `NotebookId`); a builder produces valid rows and would hide what is being proven.
- **Keep read models direct.**  The builder creates entities, not projections or response DTOs; build those in the test.
- **Prefer setup lambdas to post-build mutation** when a value belongs to the graph.
- **State the precondition you depend on** with an explicit assertion before the act step when it is not obvious from the setup (`await Assert.That(notes[1].NotebookId).IsEqualTo(other.Id);`).
- **One graph per logical scenario**: call `BuildAsync()` once.  Build the graph inside the test (or a per-test constructor), never shared between tests, and keep it small: only what the test asserts about.

## What you get back

`BuildAsync()` returns everything added, **in order, as a typed tuple**, with the seed first.  After `AddGeneratedSeed()` plus *k* `Add...` calls you get *k + 1* values:

| Call chain | `BuildAsync()` returns |
|---|---|
| `AddGeneratedSeed()` | `SeedContext` |
| `.AddNotes(3)` | `(SeedContext, Note[])` |
| `.AddNotebooks(2).AddNotes(1)` | `(SeedContext, Notebook[], Note[])` |
| `.AddNotebooksWithNotes(2)` | `(SeedContext, NotebookWithNotes[])` |

Deconstruct with the right arity (`var (seed, notebooks, notes) = ...`); discard what you do not need (`var (seed, _, notes)`).  The compiler tells you when the arity is wrong.  `graph.FindAll<Notebook>()` inside a context-aware setup returns **flat entities, seed first**: index 0 is the seed notebook, index 1 the first added.

## Using it in tests

**Handler unit tests**: build in memory and load a hand-written fake store from the graph (`FakeNoteStore` is in `testing.md`).

`src/backend/packages/<PROJECT>.Notes.Tests/ListNotesHandlerTests.cs`

```csharp
using <PROJECT>.Testing.EntityGraphs;

namespace <PROJECT>.Notes.Tests;

public sealed class ListNotesHandlerTests
{
    [Test]
    public async Task ListNotes_WithNotebookId_ReturnsOnlyThatNotebooksNotes()
    {
        // Three notes in the seed notebook, two in a second notebook; the graph reads top to bottom.
        var (seed, notebooks, seedNotes, otherNotes) = await EntityGraph
            .AddGeneratedSeed()
            .AddNotebooks()
            .AddNotes(3)
            .AddNotes(
                (note, graph) => note.NotebookId = graph.FindLast<Notebook>().Id,
                (note, graph) => note.NotebookId = graph.FindLast<Notebook>().Id
            )
            .BuildAsync();
        var store = new FakeNoteStore()
            .AddSeed(seed)
            .AddNotebooks(notebooks)
            .AddNotes(seedNotes)
            .AddNotes(otherNotes);
        var handler = new ListNotesHandler(store);

        var response = (await handler.HandleAsync(notebooks[0].Id, CancellationToken.None)).Value!;

        await Assert.That(response.Items.Count).IsEqualTo(2);
        await Assert.That(response.Items.All(item => item.NotebookId == notebooks[0].Id)).IsTrue();
    }

    [Test]
    public async Task ListNotes_WithoutNotebookId_ReturnsEveryNote()
    {
        var (seed, notes) = await EntityGraph.AddGeneratedSeed().AddNotes(4).BuildAsync();
        var store = new FakeNoteStore().AddSeed(seed).AddNotes(notes);
        var handler = new ListNotesHandler(store);

        var response = (await handler.HandleAsync(null, CancellationToken.None)).Value!;

        await Assert.That(response.Items.Count).IsEqualTo(4);
    }
}
```

**Store integration tests**: the graph saves the *setup*; the call under test does the rest.

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
        var (seed, _, other) = await EntityGraph
            .AddGeneratedSeed(_context)
            .AddNotes(2)
            .AddNotebooksWithNotes(3)
            .BuildAsync();
        _context.ChangeTracker.Clear();

        var inSeed = await new NoteStore(_context).ListAsync(
            seed.Notebook.Id,
            CancellationToken.None
        );
        var inOther = await new NoteStore(_context).ListAsync(
            other[0].Notebook.Id,
            CancellationToken.None
        );

        await Assert.That(inSeed.Count).IsEqualTo(2);
        await Assert.That(inOther.Count).IsEqualTo(3);
```

## The code

Copy the core into `<PROJECT>.Testing/EntityGraph/` (it references one project type, `SeedContext`).  The seed and the per-slice extensions are the only domain-specific files.

`src/backend/packages/<PROJECT>.Testing/EntityGraph/EntityGraphBuilder.cs`

```csharp
// The canonical way to set up test data: a terse, fluent object-graph builder.  No database for unit
// tests (pass nothing to `AddGeneratedSeed`); pass the test's `DbContext` and `BuildAsync()` persists.
// Rules and usage: the backend skill, `references/entity-graph.md`.

using System.Collections;
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;

namespace <PROJECT>.Testing.EntityGraphs;

/// <summary>
/// Implemented by a result record that is several linked entities created together, so the builder
/// can persist them.  Needed only when one `Add...` call creates more than one entity.
/// </summary>
public interface IEntityGraph
{
    /// <summary>The entities in foreign-key-safe insertion order.</summary>
    IReadOnlyList<object> GetEntities();
}

/// <summary>
/// Typed access to the entities already added to a graph.
/// </summary>
public sealed record EntityGraphContext(IReadOnlyList<object> Entities)
{
    public T Find<T>() =>
        Entities.OfType<T>().FirstOrDefault()
        ?? throw new InvalidOperationException($"No entity of type {typeof(T).Name} in the graph.");

    public T FindLast<T>() =>
        Entities.OfType<T>().LastOrDefault()
        ?? throw new InvalidOperationException($"No entity of type {typeof(T).Name} in the graph.");

    public IReadOnlyList<T> FindAll<T>() => Entities.OfType<T>().ToArray();
}

/// <summary>
/// Fluent builder for test entity graphs.  Each `Add...` pushes its result into a nested tuple
/// (<typeparamref name="TState"/>) so `BuildAsync()` can return everything added, in order, as a
/// typed tuple: `var (seed, notes) = await EntityGraph.AddGeneratedSeed().AddNotes(3).BuildAsync();`.
/// </summary>
public sealed record EntityGraphBuilder<TState>(
    SeedContext Seed,
    TState State,
    IReadOnlyList<object> Entities,
    DbContext? Database = null,
    IReadOnlyList<object>? NonPersistentEntities = null
)
{
    /// <summary>
    /// Adds one entity.  `setup` customizes the entity itself (its own properties).
    /// </summary>
    public EntityGraphBuilder<(TState Previous, TEntity Entity)> Add<TEntity>(
        Func<SeedContext, TEntity> entityFactory,
        Action<TEntity>? setup = null
    )
    {
        var entity = entityFactory(Seed);
        setup?.Invoke(entity);
        return this.Push(entity);
    }

    /// <summary>
    /// Adds one entity whose defaults depend on what was added before (the current state).
    /// </summary>
    public EntityGraphBuilder<(TState Previous, TEntity Entity)> Add<TEntity>(
        Func<SeedContext, TState, TEntity> entityFactory,
        Action<TEntity>? setup = null
    )
    {
        var entity = entityFactory(Seed, State);
        setup?.Invoke(entity);
        return this.Push(entity);
    }

    /// <summary>
    /// Adds `count` default entities.
    /// </summary>
    public EntityGraphBuilder<(TState Previous, TEntity[] Entity)> AddMany<TEntity>(
        Func<SeedContext, int, TEntity> entityFactory,
        int count
    )
    {
        ArgumentOutOfRangeException.ThrowIfNegative(count);

        var entities = new TEntity[count];

        for (var i = 0; i < entities.Length; i++)
        {
            entities[i] = entityFactory(Seed, i);
        }

        return this.Push(entities);
    }

    /// <summary>
    /// Adds one entity per setup action; no actions means one default entity.
    /// </summary>
    public EntityGraphBuilder<(TState Previous, TEntity[] Entity)> AddMany<TEntity>(
        Func<SeedContext, int, TEntity> entityFactory,
        params Action<TEntity>[] setupActions
    )
    {
        if (setupActions.Length == 0)
        {
            setupActions = [_ => { }];
        }

        var entities = new TEntity[setupActions.Length];

        for (var i = 0; i < entities.Length; i++)
        {
            entities[i] = entityFactory(Seed, i);
            setupActions[i].Invoke(entities[i]);
        }

        return this.Push(entities);
    }

    /// <summary>
    /// Adds one entity per setup action, where each action can read the entities added so far
    /// (for example to file a note in "the second notebook").
    /// </summary>
    public EntityGraphBuilder<(TState Previous, TEntity[] Entity)> AddMany<TEntity>(
        Func<SeedContext, int, TEntity> entityFactory,
        params Action<TEntity, EntityGraphContext>[] setupActions
    )
    {
        var entities = new TEntity[setupActions.Length];
        var context = Context;

        for (var i = 0; i < entities.Length; i++)
        {
            entities[i] = entityFactory(Seed, i);
            setupActions[i].Invoke(entities[i], context);
        }

        return this.Push(entities);
    }

    /// <summary>
    /// Everything in the graph so far as flat entities, seed entities first.  Arrays and graph
    /// records are flattened, so `FindAll&lt;Notebook&gt;()` sees the seed notebook and every added one.
    /// </summary>
    public EntityGraphContext Context =>
        new([.. Seed.GetSeedEntities(), .. Entities.SelectMany(EntityGraph.Flatten)]);

    /// <summary>
    /// Keeps what was just added in the result but does not save it: for "exists in memory, not in
    /// the database" cases such as a not-found test.
    /// </summary>
    public EntityGraphBuilder<TState> NotPersisted() =>
        this with
        {
            NonPersistentEntities =
            [
                .. NonPersistentEntities ?? [],
                .. EntityGraph.Flatten(Entities[^1]),
            ],
        };

    /// <summary>
    /// Saves the seed and every added entity when a database was supplied; otherwise does nothing.
    /// </summary>
    public async Task PersistStateAsync()
    {
        if (Database is null)
        {
            return;
        }

        var seedEntities = Seed.GetSeedEntities();
        Database.AddRange(seedEntities);

        HashSet<object> skip = new(
            [.. seedEntities, Seed, .. NonPersistentEntities ?? []],
            ReferenceEqualityComparer.Instance
        );

        foreach (
            var entity in Entities.SelectMany(EntityGraph.Flatten).Where(e => !skip.Contains(e))
        )
        {
            Database.Add(entity);
        }

        await Database.SaveChangesAsync();
    }
}

/// <summary>
/// Entry points and tuple plumbing for <see cref="EntityGraphBuilder{TState}"/>.
/// </summary>
public static class EntityGraph
{
    /// <summary>
    /// Starts a graph from a generated seed, pushed first into the result tuple.  Pass the test's
    /// `DbContext` to persist on `BuildAsync()`; omit it for pure in-memory unit tests.
    /// </summary>
    public static EntityGraphBuilder<(SeedContext Previous, SeedContext Current)> AddGeneratedSeed(
        DbContext? database = null,
        Action<Notebook>? notebookSetup = null
    )
    {
        var seed = SeedContext.Generate(notebookSetup);
        return new EntityGraphBuilder<SeedContext>(seed, seed, [], database).Push(seed);
    }

    /// <summary>
    /// Starts a graph from a seed you built yourself.
    /// </summary>
    public static EntityGraphBuilder<SeedContext> SeedWith(
        SeedContext context,
        DbContext? database = null
    ) => new(context, context, [], database);

    /// <summary>
    /// Pushes a value into the builder state and the entity list.
    /// </summary>
    public static EntityGraphBuilder<(TState Previous, TNext Current)> Push<TState, TNext>(
        this EntityGraphBuilder<TState> builder,
        TNext next
    ) =>
        new(
            builder.Seed,
            (builder.State, next),
            [.. builder.Entities, next!],
            builder.Database,
            builder.NonPersistentEntities
        );

    /// <summary>
    /// Flattens a pushed value (an entity, an array, or an <see cref="IEntityGraph"/>) into entities.
    /// </summary>
    internal static IEnumerable<object> Flatten(object value) =>
        value switch
        {
            string => [value],
            IEntityGraph graph => graph.GetEntities().SelectMany(Flatten),
            IEnumerable items => items.Cast<object?>().OfType<object>().SelectMany(Flatten),
            _ => [value],
        };

    // `BuildAsync` returns everything pushed, as a flat typed tuple.  One overload per arity: the
    // nested-tuple shapes are mechanical; add a fifth when a test needs one.

    public static async Task<T1> BuildAsync<TInitial, T1>(
        this EntityGraphBuilder<(TInitial InitialEntity, T1 Entity1)> builder
    )
    {
        await builder.PersistStateAsync();
        return builder.State.Entity1;
    }

    public static async Task<(T1 Entity1, T2 Entity2)> BuildAsync<TInitial, T1, T2>(
        this EntityGraphBuilder<((TInitial InitialEntity, T1 Entity1) Previous, T2 Entity2)> builder
    )
    {
        await builder.PersistStateAsync();
        return (builder.State.Previous.Entity1, builder.State.Entity2);
    }

    public static async Task<(T1 Entity1, T2 Entity2, T3 Entity3)> BuildAsync<TInitial, T1, T2, T3>(
        this EntityGraphBuilder<(
            ((TInitial InitialEntity, T1 Entity1) Previous1, T2 Entity2) Previous0,
            T3 Entity3
        )> builder
    )
    {
        await builder.PersistStateAsync();
        return (
            builder.State.Previous0.Previous1.Entity1,
            builder.State.Previous0.Entity2,
            builder.State.Entity3
        );
    }

    public static async Task<(T1 Entity1, T2 Entity2, T3 Entity3, T4 Entity4)> BuildAsync<
        TInitial,
        T1,
        T2,
        T3,
        T4
    >(
        this EntityGraphBuilder<(
            (
                ((TInitial InitialEntity, T1 Entity1) Previous2, T2 Entity2) Previous1,
                T3 Entity3
            ) Previous0,
            T4 Entity4
        )> builder
    )
    {
        await builder.PersistStateAsync();
        return (
            builder.State.Previous0.Previous1.Previous2.Entity1,
            builder.State.Previous0.Previous1.Entity2,
            builder.State.Previous0.Entity3,
            builder.State.Entity4
        );
    }
}
```

`src/backend/packages/<PROJECT>.Testing/EntityGraph/SeedContext.cs`

```csharp
using <PROJECT>.Notes;

namespace <PROJECT>.Testing.EntityGraphs;

/// <summary>
/// The ambient data every graph starts from: the entities a test almost always needs.  Extend it
/// as the domain grows (an organization, an owner, ...); everything added later hangs off it.
/// </summary>
public sealed class SeedContext
{
    private SeedContext(Notebook notebook) => Notebook = notebook;

    /// <summary>
    /// The notebook that generated notes are filed in by default.
    /// </summary>
    public Notebook Notebook { get; }

    /// <summary>
    /// Generates a seed.  Pass an action to customize the real entity (no wrapper type needed).
    /// </summary>
    public static SeedContext Generate(Action<Notebook>? notebookSetup = null)
    {
        var notebook = new Notebook { Name = "Test Notebook" };
        notebookSetup?.Invoke(notebook);

        return new SeedContext(notebook);
    }

    /// <summary>
    /// Seed entities in foreign-key-safe insertion order (parents before children).
    /// </summary>
    public IReadOnlyList<object> GetSeedEntities() => [Notebook];
}
```

`src/backend/packages/<PROJECT>.Testing/EntityGraph/EntityGraphNotesExtensions.cs`

```csharp
using <PROJECT>.Notes;

namespace <PROJECT>.Testing.EntityGraphs;

/// <summary>
/// One notebook with its notes: the result of an `Add...` call that creates several linked
/// entities.  It is an output type (needed to hand back more than one entity), not a mirror of
/// entity properties: customize the real `Notebook` and `Note` instances inside it.
/// </summary>
public sealed record NotebookWithNotes(Notebook Notebook, Note[] Notes) : IEntityGraph
{
    public IReadOnlyList<object> GetEntities() => [Notebook, .. Notes];
}

/// <summary>
/// Notes-slice helpers for the entity graph.  One block per slice keeps the builder core generic.
/// </summary>
/// <remarks>
/// Rules for adding helpers:
/// <list type="bullet">
///   <item>Customize with <c>Action&lt;TEntity&gt;</c> on the real entity; no `*Prototype` classes.</item>
///   <item>Defaults are valid and unique (ids from the entity, names numbered by index).</item>
///   <item>Defaults hang off the seed (`seed.Notebook`), so a bare `AddNotes(3)` is a coherent graph.</item>
///   <item>Return an <see cref="IEntityGraph"/> record only when one call creates linked entities.</item>
/// </list>
/// </remarks>
public static class EntityGraphNotesExtensions
{
    extension<TState>(EntityGraphBuilder<TState> builder)
    {
        /// <summary>Adds one default notebook.</summary>
        public EntityGraphBuilder<(TState Previous, Notebook[] Notebooks)> AddNotebooks() =>
            builder.AddNotebooks(1);

        /// <summary>Adds `count` default notebooks.</summary>
        public EntityGraphBuilder<(TState Previous, Notebook[] Notebooks)> AddNotebooks(
            int count
        ) => builder.AddMany(CreateNotebook, count);

        /// <summary>Adds one notebook per setup action.</summary>
        public EntityGraphBuilder<(TState Previous, Notebook[] Notebooks)> AddNotebooks(
            params Action<Notebook>[] setup
        ) => builder.AddMany(CreateNotebook, setup);

        /// <summary>Adds one default note, filed in the seed notebook.</summary>
        public EntityGraphBuilder<(TState Previous, Note[] Notes)> AddNotes() =>
            builder.AddNotes(1);

        /// <summary>Adds `count` default notes, filed in the seed notebook.</summary>
        public EntityGraphBuilder<(TState Previous, Note[] Notes)> AddNotes(int count) =>
            builder.AddMany(CreateNote, count);

        /// <summary>Adds one note per setup action, filed in the seed notebook unless changed.</summary>
        public EntityGraphBuilder<(TState Previous, Note[] Notes)> AddNotes(
            params Action<Note>[] setup
        ) => builder.AddMany(CreateNote, setup);

        /// <summary>
        /// Adds one note per setup action; each action can read what was added so far, for example
        /// `(note, graph) => note.NotebookId = graph.FindAll&lt;Notebook&gt;()[1].Id`.
        /// </summary>
        public EntityGraphBuilder<(TState Previous, Note[] Notes)> AddNotes(
            params Action<Note, EntityGraphContext>[] setup
        ) => builder.AddMany(CreateNote, setup);

        /// <summary>
        /// Adds notebooks that each come with their own notes.  Customize the generated graph, which
        /// holds the real entities.
        /// </summary>
        public EntityGraphBuilder<(
            TState Previous,
            NotebookWithNotes[] Notebooks
        )> AddNotebooksWithNotes(int notesEach, params Action<NotebookWithNotes>[] customize)
        {
            if (customize.Length == 0)
            {
                customize = [_ => { }];
            }

            var graphs = new NotebookWithNotes[customize.Length];

            for (var i = 0; i < graphs.Length; i++)
            {
                var notebook = CreateNotebook(builder.Seed, i);
                Note[] notes =
                [
                    .. Enumerable
                        .Range(0, notesEach)
                        .Select(n => new Note
                        {
                            Title = $"{notebook.Name} Note {n + 1}",
                            NotebookId = notebook.Id,
                        }),
                ];

                graphs[i] = new NotebookWithNotes(notebook, notes);
                customize[i].Invoke(graphs[i]);
            }

            return builder.Push(graphs);
        }
    }

    // Index 0 is the seed notebook, so added notebooks are numbered from 2.
    private static Notebook CreateNotebook(SeedContext seed, int index) =>
        new() { Name = $"Test Notebook {index + 2}" };

    private static Note CreateNote(SeedContext seed, int index) =>
        new() { Title = $"Test Note {index + 1}", NotebookId = seed.Notebook.Id };
}
```

## Adding it to a project or slice

1. **First entity in the project:** create `<PROJECT>.Testing/EntityGraph/` with the core and a `SeedContext` for the ambient entities, plus `<PROJECT>.Testing.Tests` for the builder's own tests (one per overload that matters: persisted, defaults, setup applied, context-aware, not persisted).  `<PROJECT>.Testing` references every slice it seeds.
2. **New slice:** add `EntityGraph<Slice>Extensions.cs` with one block of `Add<Things>` overloads (rule 4) and a `Create<Thing>(SeedContext seed, int index)` factory with valid, unique defaults.
3. Extend the seed if most tests of the slice need an ambient entity.
4. When a test needs more entities than the `BuildAsync` overloads cover (the template has up to four), add the next arity: the nested-tuple shape is mechanical.

## Pitfalls

- **Order matters, but EF sorts inserts.**  The builder adds the seed first and the rest in call order; EF orders the inserts by the model's foreign keys.  Entities without a mapped relationship rely on `GetSeedEntities()`/call order being parent-first.
- **Timestamps** that went through the database need `TruncateToPostgresPrecision()` on the expected value (`testing.md`).
