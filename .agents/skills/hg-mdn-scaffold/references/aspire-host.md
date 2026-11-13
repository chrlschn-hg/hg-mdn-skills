# Aspire Host Templates (step 9)

The Aspire AppHost in `host/`: Postgres, the backend (with the CSharpRepl hook), the Vite front-end,
Kubb in watch mode, and LibreDB for browsing the database.  An optional YARP proxy gives the app one
local origin.  Add `host/host.csproj` to the solution.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

Notes:

- `AspireUseCliBundle` must be `true` (otherwise the build warns `ASPIRE010`); keep the SDK and
  package versions equal to the installed Aspire CLI's version line.
- The host starts under central package management (`host/Directory.Packages.props`) and
  `TreatWarningsAsErrors`, like `src/backend`.  Add a package with `dotnet add package <id> --project host`.
- `Projects.<PROJECT>_Runtime` is generated from the project reference; dots in the project name
  become underscores.
- **Ports:** fixed are the backend port (`launchSettings.json`, repeated as the `/api` proxy fallback in `vite.config.ts`; under Aspire the URL is injected) and the
  AppHost's dashboard, OTLP, and resource-service ports in `host/Properties/launchSettings.json`; change
  them when another AppHost on the machine already uses them.  The front-end and LibreDB ports are assigned
  by Aspire: never pin them (`aspire describe`).  The optional app proxy uses port 80 by default; only one checkout can use that port at a time.
- **Resource names** are a contract (`aspire` commands, AGENTS.md, the backend skill's `references/debugging-aspire.md`):
  `postgres`, `app-backend`, `app-frontend`, `kubb-generate-watch`, `libredb`; `app-proxy` appears when enabled.
- LibreDB login: `admin@libredb.org` / `P@ssw0rd`.  The credentials are fixed on purpose, like the Postgres ones:
  LibreDB is local-only and the values are public.  The Postgres connection is seeded and managed (locked in the UI).
- LibreDB is local-only.  It can read and modify the database.  Never expose its port beyond the machine.
- LibreDB uses a pinned multi-architecture image digest.  Look up the current index with
  `docker buildx imagetools inspect ghcr.io/libredb/libredb-studio:latest` and copy the top-level `Digest` into
  `WithImageSHA256`; update the pin on purpose.
- Set `LIBREDB_ALLOWED_ORIGINS` only when a reverse proxy rewrites `Host` and does not send `x-forwarded-host`.
  Use comma-separated full origins or bare hosts.  Leave it unset for the direct Aspire endpoint.
- Enable the same-origin app proxy with `APP_ORIGIN_PROXY=1`.  It routes `/api`, `/scalar`, and `/openapi` to the
  backend and all other paths to Vite.  The dashboard's API reference link follows the proxy when enabled.
  The default `http://<project>.localhost` origin uses port 80.  Set `APP_ORIGIN_PROXY_PORT` when another checkout
  needs a different port; the origin then includes that port.  In a Linux dev container, add `<project>.localhost`
  to `/etc/hosts` because Node may not resolve `*.localhost`.  Browser-only Codespaces uses the forwarded HTTPS URL.
- `aspire run` finds the AppHost through `host/aspire.config.json`; run it from the repo root.
- Aspire prompts for parameters it has no value for; the defaults here are local-development values only.

### `host/host.csproj`

`host/` sits outside `src/backend`, so `src/backend/Directory.Build.props` and `Directory.Packages.props` do not
apply to it.  The host repeats both defaults: zero warnings and central package management.

```xml
<Project Sdk="Aspire.AppHost.Sdk/13.6.5">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net11.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AspireUseCliBundle>true</AspireUseCliBundle>
    <!-- Same zero tolerance as src/backend/Directory.Build.props. -->
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <UserSecretsId><GUID></UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\src\backend\runtime\<PROJECT>.Runtime.csproj" />
  </ItemGroup>

  <ItemGroup>
    <!-- Bare references: versions live in host/Directory.Packages.props (an inline Version fails with NU1008). -->
    <PackageReference Include="Aspire.Hosting.JavaScript" />
    <PackageReference Include="Aspire.Hosting.PostgreSQL" />
    <PackageReference Include="Aspire.Hosting.Yarp" />
  </ItemGroup>
</Project>
```

### `host/Directory.Packages.props`

The `Sdk="Aspire.AppHost.Sdk/<version>"` attribute cannot be centralized.  Bump it and every
`Aspire.Hosting.*` entry together, and keep them equal to the Aspire CLI's version line.

```xml
<Project>
  <!-- Central package management for the AppHost: the SDK and every Aspire.Hosting.* package share one version. -->
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Aspire.Hosting.JavaScript" Version="13.6.5" />
    <PackageVersion Include="Aspire.Hosting.PostgreSQL" Version="13.6.5" />
    <PackageVersion Include="Aspire.Hosting.Yarp" Version="13.6.5" />
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
using System.ComponentModel;
using System.Diagnostics;
using Aspire.Hosting.Yarp;
using Aspire.Hosting.Yarp.Transforms;

var builder = DistributedApplication.CreateBuilder(args);

// Opt in when the app needs a stable same-origin URL. Port 80 is shared across checkouts.
var useAppOriginProxy = builder.Configuration["APP_ORIGIN_PROXY"] == "1";
var appOriginProxyPort = int.TryParse(
    builder.Configuration["APP_ORIGIN_PROXY_PORT"],
    out var configuredAppOriginProxyPort
)
    ? configuredAppOriginProxyPort
    : 80;
string? appOrigin = null;
if (useAppOriginProxy)
{
    var portSuffix = appOriginProxyPort == 80 ? "" : $":{appOriginProxyPort}";
    appOrigin = $"http://<project>.localhost{portSuffix}";
    if (builder.Configuration["CODESPACE_NAME"] is { Length: > 0 } codespaceName)
    {
        var domain = builder.Configuration["GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN"]
            ?? "app.github.dev";
        appOrigin = $"https://{codespaceName}-{appOriginProxyPort}.{domain}";
    }
}

// Postgres; fixed credentials are for local development only.
var username = builder.AddParameter("username", "<project>", secret: true);
var password = builder.AddParameter("password", "P@ssw0rd", secret: true);
var postgres = builder
    .AddPostgres("postgres", userName: username, password: password)
    .WithVolume("<project>-pg-data", "/var/lib/postgresql")
    .WithLifetime(ContainerLifetime.Persistent)
    .WithRepl();

// The resource name is the connection string name the backend reads (`DataSetup.ConnectionStringName`).
var db = postgres.AddDatabase("<project>");

// Backend.  CSharpRepl: in run mode, pass the startup-hook environment so `csharprepl connect` can attach.
var backend = builder
    .AddProject<Projects.<PROJECT>_Runtime>("app-backend")
    .WaitFor(db)
    .WithReference(db)
    .WithUrlForEndpoint("http", url =>
    {
        url.DisplayText = "API reference (Scalar)";
        url.Url = appOrigin is null ? "/scalar" : $"{appOrigin}/scalar";
    });

if (builder.ExecutionContext.IsRunMode)
{
    foreach (var (name, value) in CSharpReplEnvironment())
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

// A single browser origin matches the production layout. Vite remains available for HMR.
if (useAppOriginProxy)
{
    frontend.WithUrl(appOrigin!, "App (<project>.localhost)");
    string[] backendPaths =
    [
        "/api/{**catch-all}",
        "/scalar/{**catch-all}",
        "/openapi/{**catch-all}",
    ];
    builder
        .AddYarp("app-proxy")
        .WithHostPort(appOriginProxyPort)
        .WithEndpoint("http", endpoint => endpoint.IsProxied = false)
        .WithConfiguration(yarp =>
        {
            var backendCluster = yarp.AddCluster(backend.GetEndpoint("http"));
            foreach (var path in backendPaths)
            {
                yarp.AddRoute(path, backendCluster).WithTransformUseOriginalHostHeader();
            }
            yarp.AddRoute("/{**catch-all}", frontend.GetEndpoint("http"))
                .WithTransformUseOriginalHostHeader();
        });
}

// LibreDB (database UI).  Open the "libredb" resource endpoint in the Aspire dashboard.
//   Login: admin@libredb.org / P@ssw0rd
// Fixed credentials, like Postgres above: LibreDB is local-only, so a known value costs nothing and nobody
// has to look a generated one up.
// The connection is seeded from `libredb/seed-connections.yaml`; SQLite stores saved queries and history.
var libredbPassword = builder.AddParameter("libredb-admin-password", "P@ssw0rd", secret: true);
var libredbJwtSecret = builder.AddParameter(
    "libredb-jwt-secret",
    "local-dev-only-jwt-secret-min-32-chars!!",
    secret: true
);

var libredb = builder
    .AddContainer("libredb", "ghcr.io/libredb/libredb-studio")
    .WithImageSHA256("3521f28fc2732a040715015669dfb475a1f2bc60b31025cb58ef472379ca74c7") // multi-arch index, checked 2026-11-13
    .WithHttpEndpoint(targetPort: 3000, name: "http")
    .WithEnvironment("ADMIN_EMAIL", "admin@libredb.org")
    .WithEnvironment("ADMIN_PASSWORD", libredbPassword)
    .WithEnvironment("JWT_SECRET", libredbJwtSecret)
    .WithEnvironment("AUTH_BOOTSTRAP", "off")
    .WithEnvironment("AUTH_COOKIE_SECURE", "false")
    .WithEnvironment("SEED_CONFIG_PATH", "/app/config/seed-connections.yaml")
    .WithEnvironment("APP_DB_URI", db.Resource.UriExpression)
    .WithEnvironment("STORAGE_PROVIDER", "sqlite")
    .WithEnvironment("STORAGE_SQLITE_PATH", "/app/data/libredb-storage.db")
    .WithBindMount(
        "./libredb/seed-connections.yaml",
        "/app/config/seed-connections.yaml",
        isReadOnly: true
    )
    .WithParentRelationship(db)
    .WaitFor(db);
libredb.WithVolume(VolumeNameGenerator.Generate(libredb, "data"), "/app/data");

// LibreDB checks the request origin against Host. Set this only when a proxy rewrites Host and drops x-forwarded-host.
if (builder.Configuration["LIBREDB_ALLOWED_ORIGINS"] is { Length: > 0 } allowedOrigins)
{
    libredb.WithEnvironment("ALLOWED_ORIGINS", allowedOrigins);
}

builder.Build().Run();

// CSharpRepl: `csharprepl connect init` prints `export NAME="VALUE"` lines (DOTNET_STARTUP_HOOKS and
// ASPNETCORE_HOSTINGSTARTUPASSEMBLIES).  Parse them all: the tool owns the names.
// A missing or failing tool skips the hook; it never blocks the AppHost.
// See: https://www.nuget.org/packages/CSharpRepl
static IReadOnlyList<(string Name, string Value)> CSharpReplEnvironment()
{
    try
    {
        using var process = Process.Start(
            new ProcessStartInfo("csharprepl", "connect init")
            {
                RedirectStandardOutput = true,
                UseShellExecute = false,
            }
        )!;
        var output = process.StandardOutput.ReadToEnd();
        process.WaitForExit();

        var exports = output
            .Split('\n', StringSplitOptions.TrimEntries)
            .Where(line => line.StartsWith("export ", StringComparison.Ordinal))
            .Select(line => line["export ".Length..].Split('=', 2))
            .Select(pair => (pair[0], pair[1].Trim('"')))
            .ToList();

        if (exports.Count == 0)
        {
            Console.WriteLine("Skipping CSharpRepl: `csharprepl connect init` printed no exports.");
        }
        return exports;
    }
    catch (Win32Exception)
    {
        Console.WriteLine("Skipping CSharpRepl: `csharprepl` is not on PATH.");
        return [];
    }
}
```

### `host/libredb/seed-connections.yaml`

```yaml
# LibreDB seed connections (https://github.com/libredb/libredb-studio/blob/main/docs/SEED_CONNECTIONS.md)
# Mounted by host/AppHost.cs. Aspire supplies the Postgres URI over the container network.
version: "1"

connections:
  - id: "<project>-local"
    name: "<PROJECT> local Postgres"
    type: postgres
    connectionString: "${APP_DB_URI}"
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
