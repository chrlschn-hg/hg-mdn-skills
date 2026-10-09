# Data Access with EF Core

All persistence lives in `<PROJECT>.Data`.  Slices define store interfaces and domain types; `Data` maps and implements them (`feature-slices.md` has the dependency direction).  Project templates are in the scaffold skill (`references/data-templates.md`); this file is the working guide.

## Conventions

- **One `AppDbContext`.**  Reach entities with `db.Set<T>()`; no `DbSet` properties required.
- **One `IEntityTypeConfiguration<T>` per entity** in `Configurations/`, picked up by `ApplyConfigurationsFromAssembly`.  `internal sealed` is fine.
- **snake_case** tables and columns from `UseSnakeCaseNamingConvention()` (`EFCore.NamingConventions`).  Set the table name and constraints in the configuration; do not name columns by hand.
- **Stores are `internal sealed`**, take `AppDbContext` through the primary constructor, are registered scoped in `DataSetup`, and return domain types.
- **Provider options in one place** (`Configure<PROJECT>Db`), shared by the runtime, the design-time factory, and the test fixture.
- **Ids:** `Guid.CreateVersion7()` (time-ordered, index-friendly) unless the domain supplies its own key.
- **Instants:** `DateTimeOffset`, named `...AtUtc`, always UTC.
- **Limits:** the slice's constants (`NoteLimits.MaxTitleLength`) in `HasMaxLength`, so contract, guard, and column agree (`endpoints.md`).

## Configuration and store

`src/backend/packages/<PROJECT>.Data/Configurations/NoteConfiguration.cs`

```csharp
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace <PROJECT>.Data.Configurations;

/// <summary>
/// EF Core mapping for <see cref="Note"/>.  Table and column names are snake_case by
/// convention (`UseSnakeCaseNamingConvention`); only the table and constraints are set here.
/// </summary>
internal sealed class NoteConfiguration : IEntityTypeConfiguration<Note>
{
    public void Configure(EntityTypeBuilder<Note> entity)
    {
        entity.ToTable("notes");
        entity.HasKey(note => note.Id);

        entity.Property(note => note.Title).IsRequired().HasMaxLength(NoteLimits.MaxTitleLength);

        // A note optionally belongs to a notebook.  Restrict: deleting a notebook that still has
        // notes is an explicit decision, not a cascade.
        entity
            .HasOne<Notebook>()
            .WithMany()
            .HasForeignKey(note => note.NotebookId)
            .OnDelete(DeleteBehavior.Restrict);

        // Supports `ListAsync` (newest first) and the per-notebook filter.
        entity.HasIndex(note => note.CreatedAtUtc).HasDatabaseName("ix_notes_created_at_utc");
        entity.HasIndex(note => note.NotebookId).HasDatabaseName("ix_notes_notebook_id");
    }
}
```

Relationships: configure the foreign key on the dependent entity (`HasOne<Notebook>().WithMany().HasForeignKey(...)`) even when the entities have no navigation properties; choose `OnDelete` deliberately (`Restrict` here: deleting a notebook that still has notes is an explicit decision), and index the foreign key column.

`src/backend/packages/<PROJECT>.Data/Notes/NoteStore.cs`

```csharp
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;

namespace <PROJECT>.Data.Notes;

/// <summary>
/// EF Core implementation of <see cref="INoteStore"/>.
/// </summary>
internal sealed class NoteStore(AppDbContext db) : INoteStore
{
    public async Task<Note> CreateAsync(Note note, CancellationToken ct)
    {
        db.Set<Note>().Add(note);
        await db.SaveChangesAsync(ct);

        return note;
    }

    public async Task<IReadOnlyList<Note>> ListAsync(Guid? notebookId, CancellationToken ct) =>
        await db.Set<Note>()
            .TagWithCallSite() // 👈 the call site appears in the SQL, visible in traces
            .Where(note => notebookId == null || note.NotebookId == notebookId)
            .OrderByDescending(note => note.CreatedAtUtc)
            .ToListAsync(ct);

    public async Task<Notebook> CreateNotebookAsync(Notebook notebook, CancellationToken ct)
    {
        db.Set<Notebook>().Add(notebook);
        await db.SaveChangesAsync(ct);

        return notebook;
    }

    public async Task<IReadOnlyList<Notebook>> ListNotebooksAsync(CancellationToken ct) =>
        await db.Set<Notebook>()
            .TagWithCallSite()
            .OrderByDescending(notebook => notebook.CreatedAtUtc)
            .ToListAsync(ct);

    public async Task<bool> NotebookExistsAsync(Guid notebookId, CancellationToken ct) =>
        await db.Set<Notebook>().AnyAsync(notebook => notebook.Id == notebookId, ct);
}
```

One line per store in `DataSetup.cs`:

`src/backend/packages/<PROJECT>.Data/DataSetup.cs (excerpt)`

```csharp
            // Stores: one line per slice store interface.
            services.AddScoped<INoteStore, NoteStore>();
```

Query habits:

- `TagWithCallSite()` puts the calling file and line in the SQL comment, so a slow query in a trace points at code.
- Read-only queries: add `AsNoTracking()`.  Writes that do not need entities: `ExecuteUpdateAsync` / `ExecuteDeleteAsync`.
- Prefer Npgsql's LINQ translations (JSON, full-text search, arrays, ranges) to raw SQL; `FromSql` only when none exists.
- Multi-tenant data: tenant key first in every index, query filter, and foreign key.
- Project to what the caller needs; do not return tracked entities across a layer you do not control.
- Catch `DbUpdateException` only to translate a specific constraint (for example a unique violation, SQL state `23505`) into a domain exception; let the rest reach the exception handler.
- Time-partitioned tables, unlogged tables, and advisory locks suit special cases; propose them when the data warrants it and **confirm with the user first**, because they are hard to reverse.

## Migrations

The app applies pending migrations at startup (`Use<PROJECT>DataAsync`).  The scaffold ships **none**: create one whenever the model changes.

```shell
# After adding or changing an entity or configuration
dotnet ef migrations add AddNotesAndNotebooks --project src/backend/packages/<PROJECT>.Data

# CI check: fails when the model has changes without a migration
dotnet ef migrations has-pending-model-changes --project src/backend/packages/<PROJECT>.Data

# Undo the last migration (before it is applied or shared)
dotnet ef migrations remove --project src/backend/packages/<PROJECT>.Data
```

`DesignTimeDbContextFactory` builds the context without starting the app, so no `--startup-project` is needed and no database has to be reachable.

Symptoms and causes:

| Symptom | Cause |
|---|---|
| `relation "notes" does not exist` (500 on first call) | entity added, no migration created |
| warning `No instantiatable types implementing IEntityTypeConfiguration were found` at startup | no configurations exist yet; disappears with the first entity |
| error-level `Failed executing DbCommand ... FROM "__EFMigrationsHistory"` on a brand-new database | EF probing for the history table before creating it; expected once |
| app exits at startup with `Failed to connect` | migrate-on-start needs the database; start it first (`mise run up`) |

Rules: never edit an applied migration; one logical change per migration with a descriptive name; review the generated `Up()` (tables, foreign keys such as `fk_notes_notebooks_notebook_id`, indexes); exclude `Migrations/` from CSharpier (`.csharpierignore`).

Production: migrate-on-start suits a single instance.  With several replicas or restricted database permissions, run migrations as a deployment step (`dotnet ef migrations bundle`) and drop `Use<PROJECT>DataAsync` from startup.

## Testing the data layer

Store tests run against a real database with `PgTransactionalTestBase`, each test in a rolled-back transaction: `testing.md`.

## Other databases

Swap the provider package, `UseNpgsql`, the Aspire hosting package, the Testcontainers module, and the telemetry instrumentation (table in the scaffold's `references/data-templates.md`).  Nothing in slices changes: they only see store interfaces.
