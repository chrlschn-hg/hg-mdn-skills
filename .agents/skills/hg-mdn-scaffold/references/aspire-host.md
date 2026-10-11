# Aspire Host Templates (step 9)

The Aspire AppHost in `host/`: Postgres, the backend (with the CSharpRepl hook), the Vite front-end,
Kubb in watch mode, and LibreDB for browsing the database.  Add `host/host.csproj` to the solution.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

Notes:

- `AspireUseCliBundle` must be `true` (otherwise the build warns `ASPIRE010`); keep the SDK and
  package versions equal to the installed Aspire CLI's version line.
- `Projects.<PROJECT>_Runtime` is generated from the project reference; dots in the project name
  become underscores.
- **Ports:** fixed are the backend port (`launchSettings.json`, repeated as the `/api` proxy fallback in `vite.config.ts`; under Aspire the URL is injected) and the
  AppHost's dashboard, OTLP, and resource-service ports in `host/Properties/launchSettings.json`; change
  them when another AppHost on the machine already uses them.  The front-end and LibreDB ports are assigned
  by Aspire: never pin them (`aspire describe`).
- **Resource names** are a contract (`aspire` commands, AGENTS.md, the backend skill's `references/debugging-aspire.md`):
  `postgres`, `app-backend`, `app-frontend`, `kubb-generate-watch`, `libredb`.
- LibreDB login: `admin@libredb.org` and the `libredb-admin-password` parameter (default `admin`).  The
  Postgres connection is seeded and managed (locked in the UI).  Seed `${VAR}` values are whole-value
  substitutions from the container environment; a connection with an undefined variable is skipped.
- `aspire run` finds the AppHost through `host/aspire.config.json`; run it from the repo root.
- Aspire prompts for parameters it has no value for; the defaults here are local-development values only.

### `host/host.csproj`

```xml
<Project Sdk="Aspire.AppHost.Sdk/13.6.1">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AspireUseCliBundle>true</AspireUseCliBundle>
    <UserSecretsId><GUID></UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\src\backend\runtime\<PROJECT>.Runtime.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.JavaScript" Version="13.6.1" />
    <PackageReference Include="Aspire.Hosting.PostgreSQL" Version="13.6.1" />
  </ItemGroup>
</Project>
```

### `host/aspire.config.json`

```json
{
  "appHost": {
    "path": "host.csproj"
  }
}
```

### `host/AppHost.cs`

```csharp
// Aspire host: orchestrates the local runtime.  Anything the app needs to run locally
// (databases, emulators, the front-end, code generation) is registered here.
// Docs: https://aspire.dev/llms.txt   Start: `mise run up`   Stop: `mise run down`
var builder = DistributedApplication.CreateBuilder(args);

// Postgres; fixed credentials are for local development only.
var username = builder.AddParameter("username", "<project>", secret: true);
var password = builder.AddParameter("password", "P@ssw0rd", secret: true);
var postgres = builder
    .AddPostgres("postgres", userName: username, password: password)
    .WithVolume("<project>-pg-data", "/var/lib/postgresql")
    .WithLifetime(ContainerLifetime.Persistent);

// The resource name is the connection string name the backend reads (`DataSetup.ConnectionStringName`).
var db = postgres.AddDatabase("<project>");

// Backend.  CSharpRepl: expose the startup-hook environment so `csharprepl connect` can attach.
var backend = builder
    .AddProject<Projects.<PROJECT>_Runtime>("app-backend")
    .WaitFor(db)
    .WithReference(db);

if (builder.ExecutionContext.IsRunMode)
{
    foreach (var (name, value) in ProcessUtils.TryGetCSharpReplConnectEnvironment())
    {
        backend.WithEnvironment(name, value);
    }
}

// Front-end (Vite).  Match `WithNpm()` to the package manager in `.config/mise.toml`.
// No fixed port: Aspire assigns a free one (shown in the dashboard), so projects never collide.
// `WithReference(backend)` hands Vite the backend URL (`services__app-backend__http__0`), which
// `vite.config.ts` uses as the target of the `/api` proxy: the browser only talks to Vite.
var frontend = builder
    .AddViteApp(name: "app-frontend", appDirectory: "../src/app")
    .WithNpm()
    .WithReference(backend)
    .WaitFor(backend);

// Kubb in watch mode: regenerates the TypeScript client when the backend rewrites the OpenAPI
// document (`src/app/src/api/<project>-api.json`) on build.
builder
    .AddExecutable(
        name: "kubb-generate-watch",
        command: "npm",
        workingDirectory: "../src/app",
        args: ["run", "generate:watch"]
    )
    .WaitFor(frontend);

// LibreDB (database UI).  Open the "libredb" resource endpoint in the Aspire dashboard.
//   Login:    admin@libredb.org
//   Password: the `libredb-admin-password` parameter (default below; override in AppHost user-secrets)
// The Postgres connection is seeded from `libredb/seed-connections.yaml` (managed; no setup in the UI).
var libredbPassword = builder.AddParameter("libredb-admin-password", "admin", secret: true);
var libredbJwtSecret = builder.AddParameter(
    "libredb-jwt-secret",
    "local-dev-only-jwt-secret-min-32-chars!!",
    secret: true
);

builder
    .AddContainer("libredb", "ghcr.io/libredb/libredb-studio", "latest")
    .WithHttpEndpoint(targetPort: 3000, name: "http") // host port assigned by Aspire; see the dashboard
    .WithEnvironment("ADMIN_EMAIL", "admin@libredb.org")
    .WithEnvironment("ADMIN_PASSWORD", libredbPassword)
    .WithEnvironment("JWT_SECRET", libredbJwtSecret)
    // The seed file interpolates `${VAR}` from the environment (whole values only).
    .WithEnvironment("SEED_CONFIG_PATH", "/app/config/seed-connections.yaml")
    .WithEnvironment("PG_HOST", postgres.Resource.Name) // container-to-container: the resource name is the hostname
    .WithEnvironment("PG_DATABASE", "<project>")
    .WithEnvironment("PG_USER", username)
    .WithEnvironment("PG_PASSWORD", password)
    .WithBindMount(
        "./libredb/seed-connections.yaml",
        "/app/config/seed-connections.yaml",
        isReadOnly: true
    )
    .WaitFor(db);

builder.Build().Run();
```

### `host/ProcessUtils.cs`

Asks `csharprepl connect init` for the startup-hook environment variables.  If `csharprepl` is missing the AppHost logs `Skipping CSharpRepl connect setup` and continues; it never blocks startup.

```csharp
using System.ComponentModel;
using System.Diagnostics;

/// <summary>
/// Runs small local processes needed by the AppHost.
/// </summary>
/// <remarks>
/// Used to ask CSharpRepl for the startup-hook environment variables that let it attach to the
/// running backend (see the `csharprepl` reference in the `hg-mdn-backend` skill).  Missing
/// tooling never blocks startup: the AppHost logs the skip and runs without REPL attachment.
/// See: https://www.nuget.org/packages/CSharpRepl
/// </remarks>
internal static class ProcessUtils
{
    public static IReadOnlyDictionary<string, string> TryGetCSharpReplConnectEnvironment()
    {
        try
        {
            return GetCSharpReplConnectEnvironment();
        }
        catch (Exception ex) when (ex is Win32Exception or InvalidOperationException)
        {
            Console.WriteLine($"Skipping CSharpRepl connect setup: {ex.Message}");
            return new Dictionary<string, string>();
        }
    }

    /// <summary>
    /// Runs `csharprepl connect init` and parses the shell exports it prints
    /// (`export DOTNET_STARTUP_HOOKS="..."`, etc.).
    /// </summary>
    private static IReadOnlyDictionary<string, string> GetCSharpReplConnectEnvironment()
    {
        var startInfo = new ProcessStartInfo("csharprepl", "connect init")
        {
            RedirectStandardError = true,
            RedirectStandardOutput = true,
            UseShellExecute = false,
        };

        using var process =
            Process.Start(startInfo)
            ?? throw new InvalidOperationException("Failed to start csharprepl connect init.");

        var outputTask = process.StandardOutput.ReadToEndAsync();
        var errorTask = process.StandardError.ReadToEndAsync();

        if (!process.WaitForExit(TimeSpan.FromSeconds(10)))
        {
            process.Kill(entireProcessTree: true);
            throw new InvalidOperationException("csharprepl connect init timed out.");
        }

        if (process.ExitCode != 0)
        {
            throw new InvalidOperationException(
                $"csharprepl connect init failed with exit code {process.ExitCode}: {errorTask.GetAwaiter().GetResult()}"
            );
        }

        return outputTask
            .GetAwaiter()
            .GetResult()
            .Split('\n', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
            .Where(line => line.StartsWith("export ", StringComparison.Ordinal))
            .Select(ParseShellExport)
            .ToDictionary(pair => pair.Name, pair => pair.Value, StringComparer.Ordinal);
    }

    /// <summary>
    /// Parses one `export NAME="VALUE"` line.  Deliberately strict: the tool version is pinned in
    /// `.config/mise.toml`, and a parse failure becomes a logged skip, never a startup crash.
    /// </summary>
    private static (string Name, string Value) ParseShellExport(string line)
    {
        var assignment = line["export ".Length..];
        var separatorIndex = assignment.IndexOf('=', StringComparison.Ordinal);

        if (separatorIndex < 1)
        {
            throw new InvalidOperationException($"Unexpected csharprepl export line: {line}");
        }

        var name = assignment[..separatorIndex];
        var value = assignment[(separatorIndex + 1)..].Trim();

        if (value.Length < 2 || value[0] != '"' || value[^1] != '"')
        {
            throw new InvalidOperationException($"Unexpected csharprepl export value: {line}");
        }

        return (name, value[1..^1]);
    }
}
```

### `host/libredb/seed-connections.yaml`

```yaml
# LibreDB seed connections (https://github.com/libredb/libredb-studio/blob/main/docs/SEED_CONNECTIONS.md)
# Mounted by host/AppHost.cs.  `${VAR}` values are whole-value substitutions from the container
# environment; a connection with an undefined variable is skipped.
version: "1"

connections:
  - id: "<project>-local"
    name: "<PROJECT> (local Postgres)"
    type: postgres
    host: "${PG_HOST}"
    port: 5432
    database: "${PG_DATABASE}"
    user: "${PG_USER}"
    password: "${PG_PASSWORD}"
    environment: local
    roles: ["admin", "user"]
    managed: true
```

### `host/appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Aspire.Hosting.Dcp": "Warning"
    }
  }
}
```

### `host/appsettings.Development.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### `host/Properties/launchSettings.json`

Dashboard `15100`, OTLP `19081`, resource service `20146`: change all three per project when they collide (`aspire ps` shows the dashboard URL of every running AppHost).

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:15100",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "DOTNET_ENVIRONMENT": "Development",
        "ASPIRE_DASHBOARD_OTLP_ENDPOINT_URL": "http://localhost:19081",
        "ASPIRE_RESOURCE_SERVICE_ENDPOINT_URL": "http://localhost:20146",
        "ASPIRE_ALLOW_UNSECURED_TRANSPORT": "true",
        "DOTNET_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS": "true"        
      }
    }
  }
}
```
