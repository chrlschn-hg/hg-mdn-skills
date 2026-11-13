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

## Add a slice

A slice that needs only endpoints and handlers needs no `Setup` code: `Add<PROJECT>Endpoints` finds them (`feature-slices.md`).

1. Add the `ProjectReference` from the runtime csproj to the slice.
2. The slice needs services of its own (options, a client, a background job)?  Add `<Slice>Setup.cs` to the slice, with `Add<PROJECT><Slice>()`:

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

3. Add `.Add<PROJECT>Invoicing()` to the chain in `Program.cs`.
4. The slice needs a store?  The line `AddScoped<I<Thing>Store, <Thing>Store>()` goes in `DataSetup`, not in the slice (`data-efcore.md`).
5. The slice needs a setting?  Add the class to `<PROJECT>.Config`, then one `AddOptions<T>()` to `SettingsSetup` (`configuration.md`).

## OpenAPI generation

`dotnet build` writes the OpenAPI document into `src/app/src/api` (`openapi-contract.md`).  The tool (`Microsoft.Extensions.ApiDescription.Server`) runs the **whole** entry point under an assembly named `GetDocument.Insider`.  It runs `Build()`, the startup code, hosted services, and `app.Run()`, with a no-op server and no database.  It writes the document once the host reports started.

- **Registration needs no infrastructure.**  `Add<PROJECT>Data` reads the connection string when the first `DbContext` is built, not when it registers.  Do the same for any client or option that needs a secret or a network.
- **Startup work needs one guard.**  `StartupSetup` returns first when `OpenApiSetup.IsGeneratingDocument` is true.  This is the only branch on it in the runtime.  Put every step after it.
- **Why not skip in `Program.cs`?**  A guard around each call spreads and gets forgotten.  A hosted service is the same: it runs in generation too, so it either checks `IsGeneratingDocument` or is registered behind a method that does.

Check with `dotnet build src/backend/runtime` (or `mise run openapi`) and read the diff of `<project>-api.json`.

## Scalar

`Map<PROJECT>OpenApi()` (`OpenApiSetup`) serves the OpenAPI document (`/openapi/v1.json`) and Scalar (`/scalar`, which redirects to `/scalar/`) in Development only.  Both sit outside the secured groups and are public by design, so no deployed environment may serve them.  `AddOpenApi` and Scalar live in the one class so a change to the document and to its viewer is one edit.

Options set, and why:

| Option | Reason |
|---|---|
| `WithTitle("<PROJECT> API")` | The browser tab and the page header name the product, not `v1`. |
| `ShowOperationId()` | The operation id is the `WithName` value, which is the name of the generated client function.  A reader finds the call to use in the front-end. |

Add an option only with a reason.  The other 2.x options (`ExpandAllTags()`, `WithDefaultHttpClient(...)`, `SortTagsAlphabetically()`, `ForceDarkMode()`, `AddServer(...)`) are cosmetic or answer a question this API does not ask.  Full list: <https://scalar.com/products/api-references/integrations/aspnetcore/integration>.

**"Try it out" works in Development.**  The placeholder authentication signs in the configured development user on every request (`Security:DevelopmentUser`), so Scalar calls `/api/me` with no token and gets 200.  Outside Development the same call answers 401, and Scalar is not served there.

**When a real authentication scheme replaces the placeholder:** declare the scheme in the OpenAPI document with a document transformer.  Scalar only offers schemes the document defines.  Then name it with `AddPreferredSecuritySchemes("<scheme>")` in `Map<PROJECT>OpenApi`.  Do not pre-fill a token with `AddHttpAuthentication`; the page shows it to anyone who can open it.

**The Aspire dashboard:** the AppHost gives the `app-backend` resource a link to `/scalar` (`aspire-host.md`).  Change both together if the route moves.

## What goes wrong

| Symptom | Cause and fix |
|---|---|
| `dotnet build` fails in `GenerateOpenApiDocuments` with `Connection string '<project>' is missing` | Startup work runs without the guard.  Put the step in `StartupSetup` after the `IsGeneratingDocument` return. |
| A test host in Development fails to build with `Unable to resolve service for type 'BuildInfo' ... HealthHandler` | Development validates the whole container on build, and the test registers only one concern.  Set `builder.Host.UseDefaultServiceProvider(o => o.ValidateOnBuild = false)` in that test host. |
| `/scalar` answers 404 | The environment is not Development.  This is the rule, not a defect. |
| `Serilog.Log.ForContext<T>()` does not compile in a setup class | `T` cannot be a static class.  Use `Serilog.Log.ForContext(typeof(<Class>))`. |
