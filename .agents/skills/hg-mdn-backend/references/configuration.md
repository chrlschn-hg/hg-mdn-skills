# Configuration and Secrets

One strongly typed `AppSettings` record, bound from the `AppSettings` configuration section.  Everything else (connection strings, OTLP endpoint, ports) is injected by Aspire in local development and by environment variables in deployment.

## Where values come from

Later sources override earlier ones:

1. `appsettings.json`, then `appsettings.<Environment>.json` (committed; no secrets)
2. User secrets (Development only; per machine; `dotnet user-secrets`)
3. Environment variables (`AppSettings__Section__Key`: double underscore for nesting)
4. Aspire (`WithEnvironment`, `WithReference`): connection strings arrive as `ConnectionStrings__<name>`

## The settings record

`src/backend/packages/<PROJECT>.Config/AppSettings.cs`

```csharp
namespace <PROJECT>.Config;

/// <summary>
/// Strongly typed application settings; bound from the `AppSettings` configuration section
/// (appsettings*.json, environment variables, user-secrets).
/// </summary>
public record class AppSettings
{
    /// <summary>
    /// (A comment here explaining this setting and a sample value)
    /// </summary>
    public string SomeSetting { get; init; } = "";

    /// <summary>
    /// (Setting that requires a secret value; set via user-secrets.  The default is the command to run.)
    /// </summary>
    public string SettingWithSecret { get; init; } =
        "dotnet user-secrets set AppSettings:SettingWithSecret YOUR_API_KEY_HERE";
}
```

- `public record class AppSettings` with `init` properties and defaults.  A setting that needs a secret **defaults to the command that sets it** (`dotnet user-secrets set AppSettings:SettingWithSecret YOUR_API_KEY_HERE`), so a missing secret fails loudly and tells the next person what to do.
- XML-doc every property: what it is, a sample value, where it is set.
- No secrets in `appsettings*.json`, in source, or in Aspire parameter defaults that reach production.

### Growing: area partials

When the record grows past a screenful, make it `public sealed partial record AppSettings` and give each area its own file with a nested settings record:

`src/backend/packages/<PROJECT>.Config/AppSettings.Limits.cs`

```csharp
namespace <PROJECT>.Config;

/// <summary>
/// Area partial: each area of settings lives in its own <c>AppSettings.{Area}.cs</c> file.
/// </summary>
public sealed partial record AppSettings
{
    /// <summary>
    /// Limits for the application (`AppSettings:Limits` in configuration).
    /// </summary>
    public LimitsSettings Limits { get; init; } = new();
}

/// <summary>
/// Limits settings.
/// </summary>
public record LimitsSettings
{
    /// <summary>
    /// Maximum notes a user may keep.  Override with the environment variable
    /// `AppSettings__Limits__MaxNotes`.
    /// </summary>
    public int MaxNotes { get; init; } = 100;
}
```

Configuration (`appsettings.json` or environment):

```json
{
  "AppSettings": {
    "Limits": { "MaxNotes": 250 }
  }
}
```

```bash
AppSettings__Limits__MaxNotes=250        # same setting from the environment
```

VS Code nests `AppSettings.*.cs` under `AppSettings.cs` (see the `.vscode/settings.json` template).

## Binding and validation

`Program.cs` binds the section and validates **at startup**, so a bad configuration stops the app immediately instead of failing on first use:

```csharp
// Program.cs
builder
    .Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection(nameof(AppSettings)))
    .Validate(
        settings => settings.Limits.MaxNotes > 0,
        "AppSettings:Limits:MaxNotes must be greater than zero."
    )
    .ValidateOnStart();
```

- `ValidateOnStart()` is what makes `Validate(...)` eager.
- One predicate and one clear message per rule (`AppSettings:Limits:MaxNotes must be greater than zero.`).
- Inject `IOptions<AppSettings>` (or `IOptionsMonitor<AppSettings>` when values may change at runtime) into handlers and services; do not read `IConfiguration` outside startup code.
- Needed before the container exists (for example to choose a provider)?  `builder.Configuration.GetSection(nameof(AppSettings)).Get<AppSettings>()` once in `Program.cs`.

## Secrets

```bash
# per developer machine (Development)
dotnet user-secrets set AppSettings:SettingWithSecret "value" --project src/backend/runtime
dotnet user-secrets list --project src/backend/runtime
```

- The runtime csproj has a `UserSecretsId`; the Aspire host has its own, for host parameters.
- Aspire parameters (`builder.AddParameter("name", secret: true)`) prompt for or read a value from the host's user secrets; defaults in `AppHost.cs` are local-development values only.
- Deployed environments get secrets from the platform's secret store as environment variables.  Prefer federated identity (OIDC, workload identity) over long-lived keys.

## Connection strings

The database resource name in `AppHost.cs` is the connection string name: `AddDatabase("<project>")` becomes `ConnectionStrings:<project>`, read with `configuration.GetConnectionString("<project>")` (`DataSetup.ConnectionStringName`).  Do not duplicate it in `AppSettings`.

## Environments

`ASPNETCORE_ENVIRONMENT` is `Development` locally (launch profile) and `Production` by default elsewhere.  Gate development-only behavior on `app.Environment.IsDevelopment()` (Scalar, OpenAPI endpoints, detailed errors), never on a custom flag.

## Ports

The backend port lives in `Properties/launchSettings.json` (`applicationUrl`); `vite.config.ts` repeats it as the `/api` dev proxy fallback for Vite without Aspire (keep them equal; under Aspire the URL is injected).  Pick a free port per project (`lsof -nP -iTCP:<port> -sTCP:LISTEN`); the dashboard uses the template's fixed `15100` and the other resource ports are assigned by Aspire (`debugging-aspire.md`).
