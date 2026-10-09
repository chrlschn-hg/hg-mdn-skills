# Data Templates (step 6, when a database was chosen)

EF Core data access, shared test fixtures, and one fixture-backed test.  Templates target Postgres.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

**No database chosen:** skip this file and remove the rest of the data wiring: the `<PROJECT>.Data`,
`.Data.Tests`, `.Testing`, and `.Testing.Tests` projects (and their `.slnx` entries); the data lines in
`Program.cs` and the `<PROJECT>.Data` reference in the runtime csproj; `AddNpgsqlInstrumentation` and the
Npgsql and EF Core packages in `<PROJECT>.Telemetry` and `Directory.Packages.props`; the Postgres, database, and
LibreDB resources in the AppHost; `postgres` in `mise.toml`; the `Migrations` line in `.csharpierignore`.

Swapping the database flavor:

| Concern | Postgres (template) | Other providers |
|---|---|---|
| EF Core provider | `Npgsql.EntityFrameworkCore.PostgreSQL`, `UseNpgsql` | `Microsoft.EntityFrameworkCore.SqlServer`, `UseSqlServer`, etc. (in `ConfigureDb` only) |
| Aspire hosting | `Aspire.Hosting.PostgreSQL`, `AddPostgres` | `Aspire.Hosting.SqlServer`, `AddSqlServer`, etc. |
| Testcontainers | `Testcontainers.PostgreSql`, `PostgreSqlBuilder` | `Testcontainers.MsSql`, etc. |
| Telemetry | `Npgsql.OpenTelemetry`, `AddNpgsqlInstrumentation` | provider instrumentation package |
| CLI in `mise.toml` | `postgres` (provides `psql`) | `github:microsoft/go-sqlcmd` (`sqlcmd`), `redis` (`redis-cli`), `usql` (universal) |

After creating the projects: add `<PROJECT>.Data` to the runtime (already in the runtime template),
add all three projects to the solution, then create the first migration when the first entity exists:

```shell
dotnet ef migrations add Initial --project src/backend/packages/<PROJECT>.Data
```

The design-time factory means no startup project is needed.  Migrations are applied at app startup
(`UseDataAsync` in `Program.cs`); `.csharpierignore` already skips the generated `Migrations` folder.

## Data package

### `src/backend/packages/<PROJECT>.Data/<PROJECT>.Data.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <InternalsVisibleTo Include="<PROJECT>.Data.Tests" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="EFCore.NamingConventions" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Relational" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" />
    <!-- Required for the EF Core CLI (`dotnet ef migrations add`, etc.) -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Data/AppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;

namespace <PROJECT>.Data;

/// <summary>
/// The EF Core `DbContext`.  Entity types live in their feature slice's project; each is
/// configured by an `IEntityTypeConfiguration&lt;T&gt;` in `Configurations/` (applied below).
/// Reach entities via `Set&lt;T&gt;()`.
/// </summary>
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Picks up every IEntityTypeConfiguration<T> in this assembly.
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### `src/backend/packages/<PROJECT>.Data/DataSetup.cs`

`Configure<PROJECT>Db` is the one place provider options are set; the runtime, the design-time factory, and the test fixture all call it so the model is built identically.

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace <PROJECT>.Data;

public static class DataSetup
{
    /// <summary>
    /// Name of the Aspire database resource (see `host/AppHost.cs`); Aspire injects the connection
    /// string as `ConnectionStrings:<project>`.
    /// </summary>
    public const string ConnectionStringName = "<project>";

    /// <summary>
    /// Applies the shared EF Core options.  Used by the runtime, the design-time factory, and
    /// the test fixture so all three build the model the same way.
    /// </summary>
    public static DbContextOptionsBuilder Configure<PROJECT>Db(
        this DbContextOptionsBuilder options,
        string connectionString
    ) => options.UseNpgsql(connectionString).UseSnakeCaseNamingConvention();

    extension(IServiceCollection services)
    {
        /// <summary>
        /// Registers <see cref="AppDbContext"/> using the Aspire-provided connection string.
        /// </summary>
        public IServiceCollection Add<PROJECT>Data(IConfiguration configuration)
        {
            var connectionString =
                configuration.GetConnectionString(ConnectionStringName)
                ?? throw new InvalidOperationException(
                    $"Connection string '{ConnectionStringName}' is missing; run the app via Aspire (`mise run up`)."
                );

            return services.AddDbContext<AppDbContext>(options =>
                options.Configure<PROJECT>Db(connectionString)
            );
        }
    }

    extension(IServiceProvider services)
    {
        /// <summary>
        /// Applies pending EF Core migrations.  Called once at startup after the host is built.
        /// </summary>
        public async Task Use<PROJECT>DataAsync()
        {
            using var scope = services.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            await db.Database.MigrateAsync();
        }
    }
}
```

### `src/backend/packages/<PROJECT>.Data/DesignTimeDbContextFactory.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace <PROJECT>.Data;

/// <summary>
/// Lets the EF Core CLI build <see cref="AppDbContext"/> without starting the app:
/// `dotnet ef migrations add Name --project src/backend/packages/<PROJECT>.Data`.
/// </summary>
/// <remarks>
/// Connection string priority: Aspire-injected `ConnectionStrings__<project>`, then a local fallback.
/// Migrations only need the model, so the fallback never has to be reachable.
/// </remarks>
public sealed class DesignTimeDbContextFactory : IDesignTimeDbContextFactory<AppDbContext>
{
    private const string FallbackConnectionString =
        "Host=localhost;Port=5432;Database=<project>;Username=postgres;Password=postgres";

    public AppDbContext CreateDbContext(string[] args)
    {
        var connectionString =
            Environment.GetEnvironmentVariable(
                $"ConnectionStrings__{DataSetup.ConnectionStringName}"
            ) ?? FallbackConnectionString;

        var options = new DbContextOptionsBuilder<AppDbContext>();
        options.Configure<PROJECT>Db(connectionString);

        return new AppDbContext(options.Options);
    }
}
```

### `src/backend/packages/<PROJECT>.Data/Configurations/README.md`

```markdown
# Entity configurations

One `IEntityTypeConfiguration<T>` per entity; `AppDbContext` applies them automatically.
See the `hg-mdn-backend` skill: `references/data-efcore.md`.
```

## Shared test fixtures

### `src/backend/packages/<PROJECT>.Testing/<PROJECT>.Testing.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Testcontainers.PostgreSql" />
    <PackageReference Include="TUnit.Core" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Data\<PROJECT>.Data.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Testing/PgDatabaseFixture.cs`

One container per test session.  `EnsureCreatedAsync()` builds the schema from the current model (no migrations), so a model change needs no extra step in tests.

```csharp
using <PROJECT>.Data;
using DotNet.Testcontainers.Builders;
using Microsoft.EntityFrameworkCore;
using Testcontainers.PostgreSql;
using TUnit.Core.Interfaces;

namespace <PROJECT>.Testing;

/// <summary>
/// Boots one Postgres container for the test session.  Share it with
/// `[ClassDataSource&lt;PgDatabaseFixture&gt;(Shared = SharedType.PerTestSession)]`.
/// </summary>
/// <remarks>
/// Faster to start than the Aspire stack; the container is removed on dispose.  The schema is
/// built from the current model with `EnsureCreatedAsync()` (not migrations), so a model change
/// is picked up by the next run with no extra step.
/// </remarks>
public class PgDatabaseFixture : IAsyncInitializer, IAsyncDisposable
{
    private PostgreSqlContainer? _container;

    private DbContextOptions<AppDbContext>? _options;

    /// <summary>
    /// Creates a new context against the test database (caller disposes).
    /// </summary>
    public AppDbContext CreateContext() => new(_options!);

    public async Task InitializeAsync()
    {
        _container = new PostgreSqlBuilder("postgres:18")
            .WithPortBinding(5432, true)
            .WithWaitStrategy(
                Wait.ForUnixContainer()
                    .UntilMessageIsLogged("database system is ready to accept connections")
                    .UntilInternalTcpPortIsAvailable(5432)
            )
            .WithAutoRemove(true)
            .Build();

        await _container.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>();
        options
            .Configure<PROJECT>Db(_container.GetConnectionString())
            .EnableDetailedErrors()
            .EnableSensitiveDataLogging();
        _options = options.Options;

        await using var context = CreateContext();
        await context.Database.EnsureCreatedAsync();
    }

    public async ValueTask DisposeAsync()
    {
        if (_container is not null)
        {
            await _container.DisposeAsync();
        }

        GC.SuppressFinalize(this);
    }
}
```

### `src/backend/packages/<PROJECT>.Testing/PgTransactionalTestBase.cs`

```csharp
using System.Data;
using <PROJECT>.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;

namespace <PROJECT>.Testing;

/// <summary>
/// Base class for database tests.  Each test gets its own <see cref="AppDbContext"/> inside an
/// explicit transaction that is always rolled back, so nothing a test writes is ever committed.
/// </summary>
public abstract class PgTransactionalTestBase(PgDatabaseFixture postgres)
{
    private IDbContextTransaction? _transaction;

    /// <summary>
    /// The context for the current test; write test bodies against this.
    /// </summary>
    protected AppDbContext _context = postgres.CreateContext();

    /// <remarks>
    /// `RepeatableRead` keeps parallel tests from observing each other's committed rows
    /// between assertions.
    /// </remarks>
    [Before(Test)]
    public async Task Before() =>
        _transaction = await _context.Database.BeginTransactionAsync(IsolationLevel.RepeatableRead);

    [After(Test)]
    public async Task Cleanup()
    {
        if (_transaction is not null)
        {
            await _transaction.RollbackAsync();
            await _transaction.DisposeAsync();
        }

        await _context.DisposeAsync();
    }
}
```

### `src/backend/packages/<PROJECT>.Testing/DateTimeOffsetExtensions.cs`

Apply to the *expected* value when asserting a timestamp that round-tripped through Postgres (microsecond precision; Linux `UtcNow` has 100 ns ticks, so a raw comparison passes on macOS and fails on Linux CI).

```csharp
namespace <PROJECT>.Testing;

/// <summary>
/// Test helpers for timestamps persisted through PostgreSQL.
/// </summary>
public static class DateTimeOffsetExtensions
{
    /// <summary>
    /// Truncates to PostgreSQL `timestamp with time zone` microsecond precision (a .NET tick is
    /// 100 ns), so an equality assertion survives a database round-trip.
    /// </summary>
    public static DateTimeOffset TruncateToPostgresPrecision(this DateTimeOffset value) =>
        new(value.Ticks - (value.Ticks % 10), value.Offset);
}
```

### `src/backend/packages/<PROJECT>.Testing.Tests/<PROJECT>.Testing.Tests.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Testing\<PROJECT>.Testing.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Testing.Tests/DateTimeOffsetExtensionsTests.cs`

Test the test helpers.  The entity-graph builder's tests live here too (see the backend skill, `references/entity-graph.md`).

```csharp
namespace <PROJECT>.Testing.Tests;

/// <summary>
/// Tests for the shared test helpers.  Run: dotnet run --project src/backend/packages/<PROJECT>.Testing.Tests
/// </summary>
public sealed class DateTimeOffsetExtensionsTests
{
    private const long MicrosecondAlignedTicks = 638_000_000_000_000_000L;

    [Test]
    [Arguments(0L)]
    [Arguments(1L)]
    [Arguments(9L)]
    public async Task TruncateToPostgresPrecision_DropsSubMicrosecondTicks(long extraTicks)
    {
        var value = new DateTimeOffset(MicrosecondAlignedTicks + extraTicks, TimeSpan.Zero);

        var truncated = value.TruncateToPostgresPrecision();

        await Assert.That(truncated.Ticks).IsEqualTo(MicrosecondAlignedTicks);
    }
}
```

## Fixture-backed test

### `src/backend/packages/<PROJECT>.Data.Tests/<PROJECT>.Data.Tests.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Testing\<PROJECT>.Testing.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Data.Tests/DatabaseFixtureTests.cs`

Requires Docker.  Run with `dotnet run --project src/backend/packages/<PROJECT>.Data.Tests`.

```csharp
using <PROJECT>.Testing;

namespace <PROJECT>.Data.Tests;

[Property("integration", "true")]
[Property("testcontainer", "true")]
[ClassDataSource<PgDatabaseFixture>(Shared = SharedType.PerTestSession)]
public sealed class DatabaseFixtureTests(PgDatabaseFixture pg) : PgTransactionalTestBase(pg)
{
    [Test]
    public async Task Database_Fixture_CanConnect()
    {
        var canConnect = await _context.Database.CanConnectAsync();

        await Assert.That(canConnect).IsTrue();
    }
}
```
