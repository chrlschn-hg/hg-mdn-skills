# The Composition Root: `Program.cs` and `Setup/`

`Program.cs` is a table of contents.  It names each concern once, in the order a request meets it.  Every concern lives in its own static setup class.  The runtime holds no business logic (`feature-slices.md`).

## The shape

```text
src/backend/
  packages/
    <PROJECT>.Api/        HttpSetup (error handling, endpoint discovery and mapping), SpaSetup, Security/SecuritySetup
    <PROJECT>.Data/       DataSetup (DbContext, stores, migrations)
    <PROJECT>.Telemetry/  TelemetrySetup (Serilog, OpenTelemetry)
    <PROJECT>.<Slice>/    <Slice>Setup, only when the slice needs services of its own
  runtime/
    Program.cs
    Setup/                what only the runtime does: SettingsSetup, OpenApiSetup, StartupSetup
```

The full file is in `hg-mdn-scaffold`, `references/backend-templates.md`.

`src/backend/runtime/Program.cs`

```csharp
using <PROJECT>.Api;
using <PROJECT>.Data;
using <PROJECT>.Health;
using <PROJECT>.Runtime;
using <PROJECT>.Telemetry;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// What is running (version, commit, build time), stamped by the build.  Telemetry reports the
// version, Health reports all three, and the startup log prints them.
var buildInfo = BuildInfo.From(typeof(Program).Assembly);

builder
    .Services.Add<PROJECT>Settings(builder.Configuration)
    .AddSingleton(buildInfo)
    .Add<PROJECT>Telemetry(
        new()
        {
            ServiceName = <PROJECT>Telemetry.ActivitySourceName,
            ServiceVersion = buildInfo.Version,
            ActivitySourceName = <PROJECT>Telemetry.ActivitySourceName,
        }
    )
    .Add<PROJECT>ErrorHandling()
    .Add<PROJECT>OpenApi()
    .Add<PROJECT>Authorization()
    .Add<PROJECT>Authentication(builder.Environment, builder.Configuration)
    .Add<PROJECT>Endpoints()
    .Add<PROJECT>Data(builder.Configuration);

var app = builder.Build();

await app.Services.Use<PROJECT>StartupAsync();

// The order is the design.  Errors first; the front-end's static files are public and come before
// request logging (assets would flood the log) and security (the shell loads before anyone signs
// in); the API lives under /api so it cannot collide with front-end routes such as /notes; the
// SPA fallback comes last.  The front-end is same-origin (this host serves it in production, the
// Vite dev server proxies /api in development), so there is no CORS.
app.UseExceptionHandler();
app.Use<PROJECT>Spa();
app.UseSerilogRequestLogging();
app.Use<PROJECT>Security();
app.Map<PROJECT>OpenApi();
app.Map<PROJECT>Endpoints("/api");
app.Map<PROJECT>SpaFallback();

app.Run();
```

Read it top to bottom: settings, telemetry and logging, services, `Build()`, startup work, pipeline, `Run()`.

## The rules

- **One concern, one class.**  `internal static class <Concern>Setup` in `runtime/Setup/`, or `public static class` in a package.  Put a concern in a package when more than the runtime needs it, or when it belongs to what the package owns (`DataSetup` in `<PROJECT>.Data`).
- **One `extension` block per receiver type.**  `IServiceCollection` for services, `WebApplication` for the pipeline and endpoints, `IServiceProvider` for startup work.
- **Three verbs.**  `Add<PROJECT><Concern>` registers services and returns `IServiceCollection`, so calls chain.  `Use<PROJECT><Concern>` adds middleware.  `Map<PROJECT><Concern>` maps endpoints.  Async startup work is `Use<PROJECT><Concern>Async` on `IServiceProvider`.
- **A method owns its conditions.**  `Add<PROJECT>Authentication` takes the environment.  `Map<PROJECT>OpenApi` checks `IsDevelopment()`.  `Use<PROJECT>Spa` checks for `wwwroot/index.html`.  The caller never wraps them in `if`.
- **Order inside the chain does not matter** for registrations (the container resolves at the end), except where a setup reads another's result: `Add<PROJECT>Telemetry` sets `Log.Logger`.  **Order in the pipeline matters.**  The comment above it says why.
- **`Program.cs` never holds:** an `if`, a lambda that configures a service, an `AddOptions`, a constant, a connection string, or a second copy of a pipeline rule.  Move it into the setup class that owns the concern, with its comment.
- **A comment sits next to the code it explains.**  The reason a route is public sits on the route; the reason for a pipeline order sits above the pipeline.
- A slice registers its own extra services (`Add<PROJECT><Slice>()` in the slice).  The runtime only adds the call.

## Lifetimes

Each lives where it is registered.  Read the registration before you add a dependency: a longer-lived service must not hold a shorter-lived one.

| Service | Lifetime | Registered in | Why |
|---|---|---|---|
| `IOptions<AppSettings>` | singleton | `SettingsSetup` | Read once; validated at startup.  Use `IOptionsMonitor` when values change at runtime. |
| `BuildInfo` | singleton | `Program.cs` chain | The stamp cannot change while the process runs. |
| `IEndpoint` | transient | `HttpSetup` (assembly scan) | Resolved once at startup from the root provider, only to map routes.  Must not depend on scoped services. |
| `IEndpointHandler` (concrete type) | transient | `HttpSetup` (assembly scan) | One per request.  It may hold scoped services (stores) because it lives inside the request scope. |
| `I<Thing>Store` | scoped | `DataSetup` | Wraps the request's `DbContext`.  Slices never reference EF Core, so the line is in `<PROJECT>.Data`. |
| `AppDbContext` | scoped | `DataSetup` | EF Core's unit of work; one per request. |

Never make a handler a singleton: it would hold a scoped store for the life of the process.  Development validates the container at build, so the host refuses to start.

## Add a concern

A concern that only the runtime needs: a hosted service, a health-check set, an HTTP client.

1. Create `runtime/Setup/<Concern>Setup.cs`.  Add one `extension(IServiceCollection services)` block with `Add<PROJECT><Concern>()`.  Add a `WebApplication` block if it has pipeline or routes.  Put the reasoning in the XML docs.
2. Add one call to the chain in `Program.cs`, and one line to the pipeline if the concern has a middleware or routes.  Mind the order.
3. Needs infrastructure at startup (a migration, a cache warm-up, a connectivity check)?  Add the step to `StartupSetup.Use<PROJECT>StartupAsync`, after its guard.
4. Used by a second host or a test?  Move the class to a package.
5. Test it with a small host that calls only that concern (`OpenApiSetupTests` is the model).

## A slice's own services

A slice that needs only endpoints and handlers needs no `Setup` code: `Add<PROJECT>Endpoints` finds them.  The steps to add a slice are in `feature-slices.md` ("Add a slice").

A slice that needs services of its own (options, a client, a background job) has `<Slice>Setup.cs` with `Add<PROJECT><Slice>()`, and the runtime adds one `.Add<PROJECT>Invoicing()` call to the chain in `Program.cs`:

```csharp
public static class InvoicingSetup
{
    extension(IServiceCollection services)
    {
        /// <summary>Registers the invoice number generator.</summary>
        public IServiceCollection Add<PROJECT>Invoicing() =>
            services.AddSingleton<IInvoiceNumbers, InvoiceNumbers>();
    }
}
```

- A store: the line `AddScoped<I<Thing>Store, <Thing>Store>()` goes in `DataSetup`, not in the slice (`data-efcore.md`).
- A setting: add the class to `<PROJECT>.Config`, then one `AddOptions<T>()` to `SettingsSetup` (`configuration.md`).

## OpenAPI generation

`dotnet build` writes the OpenAPI document into `src/app/src/api` (`openapi-contract.md`).  The tool (`Microsoft.Extensions.ApiDescription.Server`) runs the **whole** entry point under an assembly named `GetDocument.Insider`.  It runs `Build()`, the startup code, hosted services, and `app.Run()`, with a no-op server and no database.  It writes the document once the host reports started.

- **Registration needs no infrastructure.**  `Add<PROJECT>Data` reads the connection string when the first `DbContext` is built, not when it registers.  Do the same for any client or option that needs a secret or a network.  Handlers are fine: they resolve at request time.
- **Startup work needs one guard.**  `StartupSetup` returns first when `OpenApiSetup.IsGeneratingDocument` is true.  This is the only branch on it in the runtime.  Put every step after it.
- **Why not skip in `Program.cs`?**  A guard around each call spreads and gets forgotten.  A hosted service is the same: it runs in generation too, so it either checks `IsGeneratingDocument` or is registered behind a method that does.

Check with `dotnet build src/backend/runtime` (or `mise run openapi`) and read the diff of `<project>-api.json`.

`Map<PROJECT>OpenApi()` serves the document and Scalar in Development only: `openapi-contract.md` ("Scalar").

## What goes wrong

| Symptom | Cause and fix |
|---|---|
| `dotnet build` fails in `GenerateOpenApiDocuments` with `Connection string '<project>' is missing` | Startup work runs without the guard.  Put the step in `StartupSetup` after the `IsGeneratingDocument` return. |
| A test host in Development fails to build with `Unable to resolve service for type 'BuildInfo' ... HealthHandler` | Development validates the whole container on build, and the test registers only one concern.  Set `builder.Host.UseDefaultServiceProvider(o => o.ValidateOnBuild = false)` in that test host. |
| `Serilog.Log.ForContext<T>()` does not compile in a setup class | `T` cannot be a static class.  Use `Serilog.Log.ForContext(typeof(<Class>))`. |
