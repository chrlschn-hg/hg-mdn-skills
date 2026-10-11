# Testing: TUnit on Microsoft Testing Platform

**Tests use [TUnit](https://tunit.dev/llms.txt) on the [Microsoft Testing Platform](https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro) (MTP).**  Never xUnit, NUnit, or MSTest: no `xunit`, `Microsoft.NET.Test.Sdk`, or `coverlet` packages; no `[Fact]`, `[Theory]`, or `Assert.Equal`.  TUnit API (data sources, hooks, assertions) and the xUnit cheat sheet: `tunit.md`.

## The shape of the suite

Most tests are **fast, in-process, and call code directly**: no host-level tests (no `WebApplicationFactory`); handlers are called with a fake store, stores are called against a real database, and the database tests share one container.

| Test type | Project | Subject | Data |
|---|---|---|---|
| **Handler tests** (the bulk) | `<PROJECT>.<Slice>.Tests` | `HandleAsync`: guards, result selection, mapping | entity graph in memory + a hand-written fake store |
| **Store integration tests** | `<PROJECT>.Data.Tests` | the real store on real Postgres: queries, ordering, constraints, round-trips | entity graph persisted through the test's `DbContext` |
| **Test-infrastructure tests** | `<PROJECT>.Testing.Tests` | the entity-graph builder, fixtures, helpers | the builder itself |
| **Contract lint** | `<PROJECT>.Api.Tests` | the committed OpenAPI document: every operation named, summarized, described | the JSON file the build writes |
| **Route-security tests** | `<PROJECT>.Api.Tests` | route groups and guards through an in-memory pipeline: 401, 403, hidden 404, filter order | probe endpoints, header-driven test scheme |
| **Route-security lint** | `<PROJECT>.Runtime.Tests` | every real route is secured or listed as anonymous | the real endpoint set, never requested |
| **Front-end hosting tests** | `<PROJECT>.Api.Tests` | the runtime serving the built front-end: files, fallback, cache headers, the line to the API (`spa-hosting.md`) | a temporary web root, the in-memory pipeline |
| **Build-info tests** | `<PROJECT>.Health.Tests` | parsing the version stamp the build leaves in the assembly (`build-info.md`) | strings, no host |
| **App smoke tests** | `src/app` (Vitest) | the whole front-end mounted with the API mocked | see `hg-mdn-frontend` |

Rules of thumb:

- A rule or branch in a handler gets a **handler test**.  A query, ordering, filter, or constraint gets a **store integration test**.  Do not test the same thing at both levels.
- A fake proves the handler's logic; only the database proves the store's behavior.  A fake never stands in for a query you have not tested for real.
- Prefer many small tests with one clear reason to fail over a few large scenarios.
- Security rules are tested at three levels (`route-security.md`); handler tests do not repeat them, because a handler never sees an unauthenticated request.

## What a test project is

A TUnit test project is a self-executing console app (`OutputType` `Exe`) that references only `TUnit`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Notes\<PROJECT>.Notes.csproj" />
    <ProjectReference Include="..\<PROJECT>.Testing\<PROJECT>.Testing.csproj" />
  </ItemGroup>
</Project>
```

- `global.json` selects MTP for `dotnet test`: `"test": { "runner": "Microsoft.Testing.Platform" }`.
- Shared fixtures and the entity-graph builder go in `<PROJECT>.Testing`, which references `TUnit.Core` (not `TUnit`) and the slices it seeds.
- Tests live next to what they test: `<PROJECT>.<Slice>.Tests` for slices, `<PROJECT>.Data.Tests` for stores.

## Run

One test project: `dotnet run --project ...` (the TUnit-native way; it takes the filter flags directly).  Everything: `mise run test` (backend `dotnet test --solution`, then front-end).

| Command | Purpose |
|---|---|
| `dotnet run --project src/backend/packages/<PROJECT>.Notes.Tests` | Run one project. |
| `... --treenode-filter "/*/*/CreateNoteHandlerTests/*"` | One class. |
| `... --treenode-filter "/*/*/*/CreateNote_WithBlankTitle_ReturnsBadRequest"` | One test method. |
| `... --treenode-filter "/*/*/*/*[integration=true]"` | By `[Property("integration", "true")]`. |
| `... --treenode-filter "/*/<PROJECT>.Notes.Tests/*/*"` | By namespace. |
| `... --list-tests` | List discovered tests without running them. |
| `... --fail-fast` | Stop at the first failure. |
| `... --report-trx` | Write a TRX report (path printed at the end). |
| `dotnet test --solution <project>.slnx` | Every test project (CI; `--report-trx --results-directory TestResults`). |

Filter path shape: `/<assembly>/<namespace>/<class>/<method>`, with `*` wildcards and `[Property=value]` suffixes.  Docs: [test filters](https://tunit.dev/docs/execution/test-filters), [command-line flags](https://tunit.dev/docs/reference/command-line-flags).

Put the run line in the test class's doc comment (`/// Run: dotnet run --project ... --treenode-filter "/*/*/MyTests/*"`) so the next person can copy it.

MTP exit codes: `0` success, `2` test failure, `5` zero tests ran (<https://aka.ms/testingplatform/exitcodes>).  Treat `5` as a failure in CI.

**Database tests need Docker** (Testcontainers).  The first run pulls the Postgres image.  Tag them, so `--treenode-filter "/*/*/*/*[integration=true]"` runs just those (attributes in the store test below).

**Named-pipe path too long.**  `dotnet test` talks to the test app over a Unix socket whose path is limited to ~103 bytes; a deep working directory or `TMPDIR` makes the app abort with `The named pipe path ... is 150 bytes long`, and the run can look like `Zero tests ran`.  Fix: `export TESTINGPLATFORM_PIPE_DIRECTORY=/tmp/mtp` (a short existing directory), or use `dotnet run --project`.

## Naming and shape

- Name tests `EntityOrScope_StateUnderTest_ExpectedBehavior`: `CreateNote_WithBlankTitle_ReturnsBadRequest`, `Note_List_ReturnsNewestFirst`.  The name alone should say what broke.
- **High signal, low noise.**  Assert what the test is about; do not pad with unrelated checks or re-verify what another test covers.
- **Test invariants**, not implementation: "a saved note round-trips", not "the store called `SaveChanges` once".  No call-count assertions on internals.
- **Name the invariant in a comment** at the top of a non-obvious test (`// Guards that a blank title is rejected before the store is touched`).
- **Skip the bare write-then-read.**  It proves little unless the mapping, query, ordering, or constraint is what could break; then say which in the name and assert that.
- Arrange, act, assert, with a blank line between each.  A short comment above a non-obvious setup says *why* (`// Three notes here, two in a second notebook`).
- TUnit creates **a new class instance per test**, so fields and constructor state never leak between tests.  Per-test setup: constructor or `[Before(Test)]`.  Expensive shared resources (containers): a `[ClassDataSource<T>]` fixture.
- **Every `Assert.That(...)` must be awaited.**  An unawaited assertion silently does nothing.

## Test data

Build entities with the **entity graph builder**, not hand-written `new Note { ... }` (`entity-graph.md`: rules, when not to use it, the code).

## Handler tests: no host, no database

Call `HandleAsync` directly with a fake store.  Fast, deterministic, and they cover the guard clauses and result mapping.  A guard test starts from an empty fake:

`src/backend/packages/<PROJECT>.Notes.Tests/CreateNoteHandlerTests.cs (excerpt)`

```csharp
        var store = new FakeNoteStore();
        var handler = new CreateNoteHandler(store);

        var result = await handler.HandleAsync(new("   "), CancellationToken.None);

        var badRequest = result.Result as BadRequest<ApiError>;

        await Assert.That(badRequest).IsNotNull();
        await Assert.That(badRequest!.Value!.Code).IsEqualTo("invalid_title");
        await Assert.That(store.Notes).IsEmpty();
```

When the handler needs existing rows, load the fake from a graph built in memory:

`src/backend/packages/<PROJECT>.Notes.Tests/CreateNoteHandlerTests.cs (excerpt)`

```csharp
        var seed = await EntityGraph.AddGeneratedSeed().BuildAsync();
        var store = new FakeNoteStore().AddSeed(seed);
        var handler = new CreateNoteHandler(store);

        var result = await handler.HandleAsync(
            new("Hello", null, Guid.NewGuid()),
            CancellationToken.None
        );

        var notFound = result.Result as NotFound<ApiError>;

        await Assert.That(notFound).IsNotNull();
        await Assert.That(notFound!.Value!.Code).IsEqualTo("notebook_not_found");
        await Assert.That(store.Notes).IsEmpty();
```

Cast `result.Result` to the expected `TypedResults` type to assert the status (`result.Result as NotFound<ApiError>`).  Assert the **error code**, not the message.  A handler that takes a `ClaimsPrincipal` gets one from a small private helper that builds a `ClaimsIdentity` with the claims the code reads (`GetMeHandlerTests` in `<PROJECT>.Identity.Tests`); promote it to `<PROJECT>.Testing` when several test projects need it.

## Fakes and doubles

- **Hand-write fakes for interfaces you own** (`INoteStore`).  A fake is a few `List<T>` properties plus the methods the handlers use; it reads better and fails more clearly than a mocking setup.
- **Fluent seeding:** `AddSeed(seed)`, `AddNotes(...)`, ... return `this`, so setup reads `new FakeNoteStore().AddSeed(seed).AddNotes(notes)`.
- **Say what it does not cover.**  Put a `<remarks>` on every fake: it models only the surface the handlers need; the store integration tests cover the real behavior.
- **Expose knobs for failure and timing** when tests need them: a public `Exception? ThrowOnCall`, a recorded `List<string> Requests`, a `Func<Task>? Gate` to hold calls open.  Doubles for external services (an LLM client, an email sender) belong in `<PROJECT>.Testing` so every test project shares them.
- **A mocking library only for third-party interfaces** you do not control and where a fake would be noise (add NSubstitute when first needed).  Never to mock `DbContext`: use the database.
- **Time:** inject `TimeProvider` and use `FakeTimeProvider` (add `Microsoft.Extensions.TimeProvider.Testing` when first needed):

  ```csharp
  var time = new FakeTimeProvider(DateTimeOffset.Parse("2026-01-01T00:00:00Z"));

  time.Advance(TimeSpan.FromMinutes(5));      // no sleeping, no flakiness
  ```

`FakeNoteStore` shows the shape:

`src/backend/packages/<PROJECT>.Notes.Tests/FakeNoteStore.cs`

```csharp
using <PROJECT>.Testing.EntityGraphs;

namespace <PROJECT>.Notes.Tests;

/// <inheritdoc cref="INoteStore" />
/// <remarks>
/// Minimal in-memory store for handler tests.  It models only the small persistence surface the
/// handlers need to prove validation and result selection; the real queries and constraints are
/// covered by `NoteStoreIntegrationTests` against Postgres.  Fill it from an entity graph.
/// </remarks>
internal sealed class FakeNoteStore : INoteStore
{
    public List<Note> Notes { get; } = [];

    public List<Notebook> Notebooks { get; } = [];

    /// <summary>Adds the seed's entities (fluent, for test setup).</summary>
    public FakeNoteStore AddSeed(SeedContext seed)
    {
        Notebooks.AddRange(seed.GetSeedEntities().OfType<Notebook>());
        return this;
    }

    public FakeNoteStore AddNotebooks(params IEnumerable<Notebook> notebooks)
    {
        Notebooks.AddRange(notebooks);
        return this;
    }

    public FakeNoteStore AddNotes(params IEnumerable<Note> notes)
    {
        Notes.AddRange(notes);
        return this;
    }

    public Task<Note> CreateAsync(Note note, CancellationToken ct)
    {
        Notes.Add(note);
        return Task.FromResult(note);
    }

    public Task<IReadOnlyList<Note>> ListAsync(Guid? notebookId, CancellationToken ct) =>
        Task.FromResult<IReadOnlyList<Note>>([
            .. Notes
                .Where(note => notebookId == null || note.NotebookId == notebookId)
                .OrderByDescending(note => note.CreatedAtUtc),
        ]);

    public Task<Notebook> CreateNotebookAsync(Notebook notebook, CancellationToken ct)
    {
        Notebooks.Add(notebook);
        return Task.FromResult(notebook);
    }

    public Task<IReadOnlyList<Notebook>> ListNotebooksAsync(CancellationToken ct) =>
        Task.FromResult<IReadOnlyList<Notebook>>([.. Notebooks]);

    public Task<bool> NotebookExistsAsync(Guid notebookId, CancellationToken ct) =>
        Task.FromResult(Notebooks.Any(notebook => notebook.Id == notebookId));
}
```

## Store tests: a real database

Use the shared fixture and transactional base rather than standing up your own container or mocking `DbContext`:

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
[Property("integration", "true")]
[Property("testcontainer", "true")]
[ClassDataSource<PgDatabaseFixture>(Shared = SharedType.PerTestSession)]
public sealed class NoteStoreIntegrationTests(PgDatabaseFixture pg) : PgTransactionalTestBase(pg)
```

`src/backend/packages/<PROJECT>.Data.Tests/NoteStoreIntegrationTests.cs (excerpt)`

```csharp
        // The graph persists the seed notebook; the store call under test is the write.
        var seed = await EntityGraph.AddGeneratedSeed(_context).BuildAsync();
        var store = new NoteStore(_context);

        var created = await store.CreateAsync(
            new()
            {
                Title = "Hello",
                Body = "World",
                NotebookId = seed.Notebook.Id,
            },
            CancellationToken.None
        );
        _context.ChangeTracker.Clear(); // 👈 force a real read, not the tracked instance

        var notes = await store.ListAsync(null, CancellationToken.None);

        await Assert.That(notes.Count).IsEqualTo(1);
        await Assert.That(notes[0].Id).IsEqualTo(created.Id);
        await Assert.That(notes[0].Title).IsEqualTo("Hello");
        await Assert.That(notes[0].NotebookId).IsEqualTo(seed.Notebook.Id);
        await Assert
            .That(notes[0].CreatedAtUtc)
            .IsEqualTo(created.CreatedAtUtc.TruncateToPostgresPrecision());
```

- `[ClassDataSource<PgDatabaseFixture>(Shared = SharedType.PerTestSession)]` boots **one** Postgres container for the whole run, reused by every class that declares it (`PgDatabaseFixture` in `<PROJECT>.Testing`).
- `PgTransactionalTestBase` opens a `RepeatableRead` transaction in `[Before(Test)]` and always rolls it back in `[After(Test)]`: nothing a test writes is committed, so tests need no cleanup and never collide.
- Write against `_context`.  Anything that bypasses its transaction (a second connection, raw SQL on another context, un-awaited background work) escapes the rollback and can leak into later tests.
- Use the **entity graph** for the rows the test needs; the store call under test is then the only thing that is not setup.
- `_context.ChangeTracker.Clear()` between the write and the read, or the test reads the tracked instance and proves nothing.
- The schema is built once per container with `EnsureCreatedAsync()` from the current model (no migrations): schema changes need a new container, not a new test.  Migrations are verified separately (`dotnet ef migrations has-pending-model-changes`).
- A base class per area is fine when many classes share helpers on top of `PgTransactionalTestBase`.

## Timestamps through the database

Postgres keeps `timestamptz` to the microsecond; .NET ticks are 100 ns.  A round-tripped timestamp compares equal on macOS (microsecond clock) but not on Linux CI.  Truncate the *expected* side with `TruncateToPostgresPrecision()` (`<PROJECT>.Testing`), as in the last assertion above.

## Contract lint: descriptions are enforced

`<PROJECT>.Api.Tests` reads the committed OpenAPI document and fails when an operation has no `operationId`, no summary ending in a period, or a description that is not one to two paragraphs of one to six sentences (`openapi-contract.md`).  An endpoint without `WithDescription` therefore fails `mise run test` and CI: the descriptions are the API documentation.

## CI

`.github/workflows/ci.yml` runs `dotnet test --solution <project>.slnx --report-trx --results-directory TestResults` and publishes the TRX.  The runner is Linux: write tests that pass there (timestamps above, case-sensitive paths).
