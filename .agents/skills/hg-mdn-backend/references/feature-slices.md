# Feature Slices

A feature slice is one project that owns a capability end to end: domain types, the storage abstraction, HTTP endpoints, handlers, and contracts.  The runtime stays lean; everything else is a slice or an infrastructure package.  `<PROJECT>.Health` and `<PROJECT>.Identity` (from the scaffold) are the smallest slices.  `<PROJECT>.Notes` below is a worked example with persistence: it lives in these references, not in the scaffolded repository.

## Dependency direction

```text
Runtime ──► slices, Api, Telemetry, Config, Data   (wires everything; no business logic)
Data    ──► slices                                 (implements the stores that slices define)
slices  ──► Api, Telemetry                         (never Data, never Runtime, never another slice's internals)
tests   ──► the project under test (+ <PROJECT>.Testing: entity graph, fixtures)
```

- Slices depend on **interfaces** (`I<Thing>Store`), so they never reference EF Core and handler tests use a fake.
- The slice defines what it needs; `Data` provides it.
- If two slices need the same domain types, or `Data` should not pull in ASP.NET, split the shared types into `<PROJECT>.<Slice>.Core` (domain + interfaces, no ASP.NET reference) and keep endpoints in `<PROJECT>.<Slice>`.  Do this when needed, not up front.

## Anatomy of a slice

```text
src/backend/packages/<PROJECT>.Notes/
  <PROJECT>.Notes.csproj
  Note.cs, Notebook.cs                 domain types (no framework or persistence dependencies; public setters)
  NoteLimits.cs                        constants shared by contracts, guards, and the database mapping
  INoteStore.cs                        storage abstraction; implemented in <PROJECT>.Data
  NoteEndpoints.cs                     IEndpoint: routes, names, summaries, descriptions (the API documentation)
  NoteEndpoints.Contracts.cs           request/response records (XML docs become OpenAPI descriptions)
  NoteEndpoints.Handler.CreateNote.cs  one IEndpointHandler per operation: validation + orchestration
  NoteEndpoints.Handler.ListNotes.cs   (also CreateNotebook, ListNotebooks)
  NoteMapping.cs                       domain <-> contract mapping (extension members)
src/backend/packages/<PROJECT>.Notes.Tests/
  FakeNoteStore.cs, CreateNoteHandlerTests.cs, ListNotesHandlerTests.cs
src/backend/packages/<PROJECT>.Testing/EntityGraph/
  EntityGraphNotesExtensions.cs        the slice's test-data helpers (see entity-graph.md)
```

Conventions:

- One handler class per operation, named `<Verb><Thing>Handler`, in `<Slice>Endpoints.Handler.<Verb><Thing>.cs`.
- Handlers take a store interface through the primary constructor and return `Results<...>` built with `TypedResults`.
- Contracts are `sealed record`s with XML docs on every parameter.
- Mapping lives in one `internal static class` of extension members; handlers never build contracts by hand.
- `InternalsVisibleTo` the tests project instead of making internals public.
- Slice names are nouns (`Notes`, `Billing`).

## Add a slice

Test first: the first test of a slice is a handler test.

1. Create `<PROJECT>.<Slice>/<PROJECT>.<Slice>.csproj` (template below) and the `.Tests` project; add both to `<project>.slnx`.
2. Add a `ProjectReference` to the slice in `<PROJECT>.Runtime.csproj`.  Nothing else is needed for routing: `Add<PROJECT>Endpoints` scans for `IEndpoint` and `IEndpointHandler`, and security comes with the group the slice is mapped onto (`route-security.md`).
3. Write the domain type and `I<Thing>Store`.
4. Add `EntityGraph<Slice>Extensions.cs` to `<PROJECT>.Testing` (it needs a `ProjectReference` to the slice; `entity-graph.md`), then write the failing handler test with a fake store (`testing.md`).
5. Write contracts, handler, mapping, and the endpoint with `WithName`, `WithSummary`, `WithDescription` (`endpoints.md`, `openapi-contract.md`) until the test passes.
6. In `<PROJECT>.Data`: add the `IEntityTypeConfiguration<T>`, the store implementation, one `AddScoped<I<Thing>Store, <Thing>Store>()` line in `DataSetup`, and a `ProjectReference` to the slice.
7. Create the migration (`data-efcore.md`), then write the store integration tests against the database fixture.
8. `dotnet build`, then `mise run generate` (`openapi-contract.md`).

## The Notes slice

`src/backend/packages/<PROJECT>.Notes/<PROJECT>.Notes.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <InternalsVisibleTo Include="<PROJECT>.Notes.Tests" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Innoloft.LexoRank" />
    <PackageReference Include="Nanoid" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Api\<PROJECT>.Api.csproj" />
    <ProjectReference Include="..\<PROJECT>.Telemetry\<PROJECT>.Telemetry.csproj" />
  </ItemGroup>
</Project>
```

`src/backend/packages/<PROJECT>.Notes/Note.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// A short note, optionally filed in a <see cref="Notebook"/>.  Domain type: no framework or
/// persistence dependencies.  Properties have public setters so tests can customize generated
/// instances (see the entity-graph builder).
/// </summary>
public sealed class Note
{
    /// <summary>Time-ordered id (UUID v7); sortable and index-friendly.</summary>
    public Guid Id { get; set; } = Guid.CreateVersion7();

    /// <summary>The notebook the note is filed in; <c>null</c> when it is not filed anywhere.</summary>
    public Guid? NotebookId { get; set; }

    public required string Title { get; set; }

    public string Body { get; set; } = "";

    public DateTimeOffset CreatedAtUtc { get; set; } = DateTimeOffset.UtcNow;
}
```

`src/backend/packages/<PROJECT>.Notes/Notebook.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// A named group of notes.
/// </summary>
public sealed class Notebook
{
    /// <summary>Time-ordered id (UUID v7).</summary>
    public Guid Id { get; set; } = Guid.CreateVersion7();

    public required string Name { get; set; }

    public DateTimeOffset CreatedAtUtc { get; set; } = DateTimeOffset.UtcNow;
}
```

`src/backend/packages/<PROJECT>.Notes/NoteLimits.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// Limits shared by the contracts (OpenAPI metadata), the handler guards, and the database
/// mapping, so a limit is defined once.
/// </summary>
public static class NoteLimits
{
    public const int MaxTitleLength = 200;

    public const int MaxBodyLength = 20_000;

    public const int MaxNotebookNameLength = 100;
}
```

`src/backend/packages/<PROJECT>.Notes/INoteStore.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// Storage for notes and notebooks.  Handlers depend on this abstraction; `<PROJECT>.Data` implements
/// it, so the slice never references EF Core and handler tests can use a fake.
/// </summary>
public interface INoteStore
{
    Task<Note> CreateAsync(Note note, CancellationToken ct);

    /// <summary>
    /// Lists notes, newest first; only those filed in <paramref name="notebookId"/> when it is
    /// given.
    /// </summary>
    Task<IReadOnlyList<Note>> ListAsync(Guid? notebookId, CancellationToken ct);

    Task<Notebook> CreateNotebookAsync(Notebook notebook, CancellationToken ct);

    /// <summary>Lists notebooks, newest first.</summary>
    Task<IReadOnlyList<Notebook>> ListNotebooksAsync(CancellationToken ct);

    Task<bool> NotebookExistsAsync(Guid notebookId, CancellationToken ct);
}
```

Endpoints, contracts, handlers, and mapping are in `endpoints.md`; the store implementation and EF mapping are in `data-efcore.md`; tests are in `testing.md`.
