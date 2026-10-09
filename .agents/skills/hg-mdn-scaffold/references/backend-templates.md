# Backend Templates (step 6)

Templates for the .NET web API runtime: shared build config, the infrastructure packages, the first
feature slice, and the runtime host.  Create `src/app/src/api/` first (its `README.txt` is in
`frontend-templates.md`), create the files, add every project to `<project>.slnx`, then `dotnet build`
(this writes the OpenAPI document into `src/app/src/api`).  Data access and the shared test fixtures
are in `data-templates.md`.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

```text
src/backend/
  Directory.Build.props
  Directory.Packages.props
  packages/
    <PROJECT>.Config/          AppSettings (strongly typed settings)
    <PROJECT>.Api/             IEndpoint, IEndpointHandler, endpoint discovery + mapping,
                               Security/ (secure-by-default route groups, admin group, route guards)
    <PROJECT>.Api.Tests/       lints the OpenAPI document; tests the route groups and guards in memory
    <PROJECT>.Telemetry/       Serilog + OpenTelemetry setup, tracing helpers
    <PROJECT>.Health/          first feature slice; copy its shape for new slices
    <PROJECT>.Health.Tests/
    <PROJECT>.Identity/        second slice: `GET /me` (who is signed in)
    <PROJECT>.Identity.Tests/
    <PROJECT>.Runtime.Tests/   lints the real routes: every one secured or listed as anonymous
  runtime/
    <PROJECT>.Runtime.csproj, Program.cs, appsettings*.json, Properties/launchSettings.json
```

## Shared build configuration

### `src/backend/Directory.Build.props`

`GenerateDocumentationFile` matters: the OpenAPI generator turns XML doc comments into schema and operation descriptions, so `///` comments on contracts reach the generated client.  `EnforceCodeStyleInBuild` makes `dotnet build` report the `.editorconfig` rules (braces, exhaustive enum switches).

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- XML doc comments feed the OpenAPI document (summaries, descriptions). -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);CS1591</NoWarn>
    <!-- Applies the .editorconfig style rules (braces, exhaustive switches) during `dotnet build`. -->
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

### `src/backend/Directory.Packages.props`

Central package management: versions live here only; csproj files use bare `<PackageReference Include="..." />`.  `TUnit.Core` needs its own entry because `<PROJECT>.Testing` references it.  `OpenTelemetry.Instrumentation.EntityFrameworkCore` is a pre-release package: keep it unless a stable one exists.  Drop packages for features not chosen (for example the Npgsql and EF Core entries when there is no database).

```xml
<Project>
  <!-- Central package management: versions live here only; csproj files use bare `PackageReference Include`. -->
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="EFCore.NamingConventions" Version="10.0.1" />
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.12" />
    <PackageVersion Include="Microsoft.AspNetCore.TestHost" Version="10.0.12" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.12">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageVersion>
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Relational" Version="10.0.12" />
    <PackageVersion Include="Microsoft.Extensions.ApiDescription.Server" Version="10.0.12">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageVersion>
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.3" />
    <PackageVersion Include="Npgsql.OpenTelemetry" Version="10.0.3" />
    <PackageVersion Include="OpenTelemetry" Version="1.19.1" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.19.1" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.19.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.19.0" />
    <PackageVersion
      Include="OpenTelemetry.Instrumentation.EntityFrameworkCore"
      Version="1.19.1-beta.1"
    />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.19.0" />
    <PackageVersion Include="Scalar.AspNetCore" Version="2.17.14" />
    <PackageVersion Include="Scrutor" Version="7.0.0" />
    <PackageVersion Include="Serilog" Version="4.4.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="10.0.0" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="6.1.1" />
    <PackageVersion Include="Serilog.Sinks.OpenTelemetry" Version="4.2.0" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="4.16.0" />
    <PackageVersion Include="TUnit" Version="1.73.19" />
    <PackageVersion Include="TUnit.Core" Version="1.73.19" />
  </ItemGroup>
</Project>
```

## Config package

### `src/backend/packages/<PROJECT>.Config/<PROJECT>.Config.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk"></Project>
```

### `src/backend/packages/<PROJECT>.Config/AppSettings.cs`

Settings that need a secret default to the command that sets them (`dotnet user-secrets set ...`), so a missing secret is self-documenting.

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

## Api package

### `src/backend/packages/<PROJECT>.Api/<PROJECT>.Api.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Scrutor" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Api/IEndpoint.cs`

```csharp
using Microsoft.AspNetCore.Routing;

namespace <PROJECT>.Api;

/// <summary>
/// Implemented once per feature slice to map its HTTP endpoints.  Discovered and registered
/// by `Add<PROJECT>Endpoints`; mapped in `Program.cs` via `Map<PROJECT>Endpoints`.
/// </summary>
public interface IEndpoint
{
    void MapEndpoints(IEndpointRouteBuilder endpoints);
}
```

### `src/backend/packages/<PROJECT>.Api/IEndpointHandler.cs`

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// Marker interface.  Handlers are registered by concrete type (transient) and injected into
/// the endpoint delegate as a parameter.
/// </summary>
public interface IEndpointHandler { }
```

### `src/backend/packages/<PROJECT>.Api/ApiError.cs`

Shared body for expected failures (`BadRequest<ApiError>`, `Conflict<ApiError>`, `NotFound<ApiError>`). Handlers return a distinct `Code` per failure so the front-end can react to each.

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// Error body returned by endpoints for expected failures (validation, conflicts, not found).
/// Unexpected failures use ASP.NET Core problem details instead.
/// </summary>
/// <param name="Code">Stable machine-readable code, `snake_case` (e.g. `invalid_title`).</param>
/// <param name="Message">Human-readable description.</param>
public sealed record ApiError(string Code, string Message);
```

### `src/backend/packages/<PROJECT>.Api/HttpExceptionHandler.cs`

Wired in `Program.cs` with `AddExceptionHandler<HttpExceptionHandler>().AddProblemDetails()` and `app.UseExceptionHandler()`.  Returns a problem-details 500 with `traceId` (exception text in Development only) and logs once.

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace <PROJECT>.Api;

/// <summary>
/// Converts unhandled exceptions into one consistent problem-details response.
/// </summary>
/// <remarks>
/// Minimal APIs have no MVC exception filters, so this runs at the middleware boundary
/// (`app.UseExceptionHandler()`).  Handlers stay focused on expected outcomes (`ApiError`);
/// anything unexpected is logged once (with the trace id), marked on the active span, and returned
/// as problem details; ASP.NET Core adds the `traceId` extension so a report can be matched to its
/// trace.  The exception text is exposed in Development only.
/// </remarks>
public sealed class HttpExceptionHandler(
    ILogger<HttpExceptionHandler> log,
    IHostEnvironment environment
) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken
    )
    {
        var traceId = Activity.Current?.TraceId.ToString() ?? httpContext.TraceIdentifier;

        log.LogError(
            exception,
            "Unhandled exception for {Method} {Path} (trace {TraceId})",
            httpContext.Request.Method,
            httpContext.Request.Path.Value,
            traceId
        );

        Activity.Current?.AddException(exception);
        Activity.Current?.SetStatus(ActivityStatusCode.Error);

        await Results
            .Problem(
                statusCode: StatusCodes.Status500InternalServerError,
                title: "An unexpected error occurred.",
                detail: environment.IsDevelopment()
                    ? exception.ToString()
                    : "An unexpected error occurred while processing the request.",
                instance: httpContext.Request.Path
            )
            .ExecuteAsync(httpContext);

        return true;
    }
}
```

### `src/backend/packages/<PROJECT>.Api/HttpSetup.cs`

Scrutor scans every assembly the app depends on, so a new feature slice needs only a `ProjectReference` from the runtime.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;

namespace <PROJECT>.Api;

public static class HttpSetup
{
    extension(IServiceCollection services)
    {
        /// <summary>
        /// Scans the application's assemblies and registers every <see cref="IEndpoint"/> (by
        /// interface) and <see cref="IEndpointHandler"/> (by concrete type) as transient.
        /// New feature slices are picked up automatically once referenced by the runtime project.
        /// </summary>
        public IServiceCollection Add<PROJECT>Endpoints() =>
            services.Scan(scan =>
                scan.FromApplicationDependencies()
                    .AddClasses(classes => classes.AssignableTo<IEndpoint>())
                    .AsImplementedInterfaces()
                    .WithTransientLifetime()
                    .AddClasses(classes => classes.AssignableTo<IEndpointHandler>())
                    .AsSelf()
                    .WithTransientLifetime()
            );
    }

    extension(WebApplication app)
    {
        /// <summary>
        /// Maps all registered <see cref="IEndpoint"/> instances, under `prefix` when given.
        /// </summary>
        /// <remarks>
        /// A prefix (`/api`) is needed when the same host also serves the front-end: SPA routes and
        /// API routes then share one path space, and the prefix keeps them apart.
        /// </remarks>
        public WebApplication Map<PROJECT>Endpoints(string? prefix = null)
        {
            // Secure by default: every route under the root group needs an authenticated user
            // unless it opts out with `AllowAnonymous()`.  The admin group sits under it.
            var root = app.MapApiRoot(prefix);
            var admin = root.MapAdminGroup();

            foreach (var endpoint in app.Services.GetRequiredService<IEnumerable<IEndpoint>>())
            {
                // Security by placement: the slice only ever sees the group it is mapped onto.
                endpoint.MapEndpoints(endpoint is IAdminEndpoint ? admin : root);
            }

            if (!string.IsNullOrEmpty(prefix))
            {
                // Anything under the prefix that no endpoint matched is an API 404, not a
                // front-end route: without this the SPA fallback would answer `/api/typo` with
                // index.html and a 200.
                root.Map(
                        "{**unmatched}",
                        () => Results.NotFound(new ApiError("not_found", "No such API route."))
                    )
                    .AllowAnonymous(); // a typo is a 404 for everyone, not a 401
            }

            return app;
        }
    }
}
```

## Api package: security (route groups and guards)

Secure by default: slices are mapped onto a group that requires a signed-in user, admin-only slices onto an `/admin` group, and guards (`IEndpointFilter`) attach to groups or routes.  Rules, layers, and testing: the backend skill, `references/route-security.md`.

### `src/backend/packages/<PROJECT>.Api/Security/RouteGroups.cs`

`MapApiRoot` is the secured root group; `MapAdminGroup` is the `/admin` subgroup under it.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Routing;

namespace <PROJECT>.Api;

/// <summary>
/// The route groups that carry the security rules.  Slices never map onto the bare application:
/// they receive a group, so the rules apply to every route they add.
/// </summary>
public static class RouteGroups
{
    extension(IEndpointRouteBuilder app)
    {
        /// <summary>
        /// The root group (under `prefix` when given).  Secure by default: every route needs an
        /// authenticated user unless it opts out with <c>AllowAnonymous()</c>.
        /// </summary>
        public RouteGroupBuilder MapApiRoot(string? prefix = null) =>
            app.MapGroup(prefix ?? "").RequireAuthorization();
    }

    extension(RouteGroupBuilder root)
    {
        /// <summary>
        /// The `/admin` group under the root: requires the admin policy, and failures answer 404
        /// (<see cref="HiddenRouteMetadata"/>).  Slices opt in with <see cref="IAdminEndpoint"/>.
        /// </summary>
        public RouteGroupBuilder MapAdminGroup() =>
            root.MapGroup("/admin")
                .WithMetadata(new HiddenRouteMetadata())
                .RequireAuthorization(Policies.Admin);
    }
}
```

### `src/backend/packages/<PROJECT>.Api/Security/Policies.cs`

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// Names of the authorization policies the route groups apply.
/// </summary>
public static class Policies
{
    /// <summary>
    /// Applied to the `/admin` route group (see <see cref="RouteGroups.MapAdminGroup"/>).
    /// </summary>
    public const string Admin = "admin";
}

/// <summary>
/// Role names the policies check.  The authentication scheme in use puts them in the user's
/// role claims.
/// </summary>
public static class Roles
{
    public const string Admin = "admin";

    public const string Member = "member";
}
```

### `src/backend/packages/<PROJECT>.Api/Security/IAdminEndpoint.cs`

Marker: a slice that implements it is mapped onto the admin group (security by placement).

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// Marks an endpoint slice whose routes are admin-only.  <see cref="HttpSetup.Map<PROJECT>Endpoints"/>
/// maps it onto the `/admin` group, which already carries the admin policy and hides itself from
/// everyone else (404).
/// </summary>
/// <remarks>
/// Security by placement: an admin endpoint receives the secured group and maps paths relative to
/// it (`endpoints.MapGet("/stats", ...)` becomes `GET /admin/stats`).  It must not state its own,
/// weaker authorization, and cannot loosen the group's.
/// </remarks>
public interface IAdminEndpoint : IEndpoint;
```

### `src/backend/packages/<PROJECT>.Api/Security/HiddenRouteMetadata.cs`

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// Endpoint metadata that cloaks authorization failures as 404, so a route's existence is not
/// advertised to callers who may not use it.  Attached to the `/admin` group.
/// </summary>
/// <remarks>
/// Read by <see cref="HiddenRouteAuthorizationResultHandler"/>; routes without it keep the
/// framework's 401 (challenge) and 403 (forbid).
/// </remarks>
public sealed class HiddenRouteMetadata;
```

### `src/backend/packages/<PROJECT>.Api/Security/HiddenRouteAuthorizationResultHandler.cs`

Answers 404 for a failed authorization on routes that carry the metadata (the admin group); every other route keeps 401 and 403.

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Authorization.Policy;
using Microsoft.AspNetCore.Http;

namespace <PROJECT>.Api;

/// <summary>
/// Turns a failed authorization into 404 for routes that carry <see cref="HiddenRouteMetadata"/>,
/// and defers to the framework for everything else.
/// </summary>
/// <remarks>
/// Deliberately narrow: only a matched endpoint with the metadata changes behavior, so ordinary
/// routes keep their 401 and 403.
/// </remarks>
public sealed class HiddenRouteAuthorizationResultHandler : IAuthorizationMiddlewareResultHandler
{
    private readonly AuthorizationMiddlewareResultHandler _default = new();

    public Task HandleAsync(
        RequestDelegate next,
        HttpContext context,
        AuthorizationPolicy policy,
        PolicyAuthorizationResult authorizeResult
    )
    {
        var hidden = context.GetEndpoint()?.Metadata.GetMetadata<HiddenRouteMetadata>() is not null;

        if (!hidden || authorizeResult.Succeeded)
        {
            return _default.HandleAsync(next, context, policy, authorizeResult);
        }

        context.Response.StatusCode = StatusCodes.Status404NotFound;

        return Task.CompletedTask;
    }
}
```

### `src/backend/packages/<PROJECT>.Api/Security/RequireRouteMatchesClaimFilter.cs`

A stateless route guard: tenant isolation.  400 no route value, 403 no claim or a different one.  Never 401: the secured group guarantees a signed-in caller, and the front-end reads every 401 as "nobody is signed in".

```csharp
using Microsoft.AspNetCore.Http;

namespace <PROJECT>.Api;

/// <summary>
/// Route guard: a route value must equal one of the caller's claims.  The usual use is tenant
/// isolation: `tenants/{tenantId}/...` is only for a caller whose `tenant` claim is that tenant.
/// </summary>
/// <remarks>
/// Answers 400 when the route has no such value, and 403 when the caller has no such claim or a
/// different one.  Never 401: the secured root group already guarantees a signed-in caller, and
/// the front-end treats every 401 as "nobody is signed in" (it would send the user to login in a
/// loop).  Cheap on purpose (no I/O): register it first, before guards that read state.
/// </remarks>
/// <param name="routeKey">The route value to read, for example `tenantId`.</param>
/// <param name="claimType">The claim that must carry the same value, for example `tenant`.</param>
public sealed class RequireRouteMatchesClaimFilter(string routeKey, string claimType)
    : IEndpointFilter
{
    public ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next
    )
    {
        var routeValue = context.HttpContext.Request.RouteValues[routeKey]?.ToString();
        var claimValue = context.HttpContext.User.FindFirst(claimType)?.Value;

        if (string.IsNullOrWhiteSpace(routeValue))
        {
            return ValueTask.FromResult<object?>(TypedResults.BadRequest());
        }

        // A signed-in caller without the claim is not allowed here: 403, not 401.
        if (
            string.IsNullOrWhiteSpace(claimValue)
            || !string.Equals(routeValue, claimValue, StringComparison.Ordinal)
        )
        {
            return ValueTask.FromResult<object?>(TypedResults.Forbid());
        }

        return next(context);
    }
}
```

### `src/backend/packages/<PROJECT>.Api/Security/RouteGuardExtensions.cs`

One generic extension block serves groups and single routes (C# 14).

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;

namespace <PROJECT>.Api;

/// <summary>
/// Guards as extension methods, so a slice reads <c>group.RequireRouteMatchesClaim(...)</c>.
/// One generic block serves groups and single routes alike.
/// </summary>
public static class RouteGuardExtensions
{
    extension<TBuilder>(TBuilder builder)
        where TBuilder : IEndpointConventionBuilder
    {
        /// <summary>
        /// Requires the route value `routeKey` to equal the caller's `claimType` claim
        /// (see <see cref="RequireRouteMatchesClaimFilter"/>).
        /// </summary>
        public TBuilder RequireRouteMatchesClaim(string routeKey, string claimType) =>
            builder.AddEndpointFilter(new RequireRouteMatchesClaimFilter(routeKey, claimType));
    }
}
```

### `src/backend/packages/<PROJECT>.Api/Security/DevelopmentUserOptions.cs`

```csharp
namespace <PROJECT>.Api;

/// <summary>
/// The user the Development scheme signs in (`Security:DevelopmentUser` in configuration).
/// </summary>
public sealed class DevelopmentUserOptions
{
    public string Subject { get; set; } = "dev-user";

    public string Name { get; set; } = "Development User";

    // NOTE: empty by default on purpose.  Configuration binding appends to an array that has a
    // default, so a default of [member] plus a configured [member, admin] would give
    // [member, member, admin].  `DevelopmentAuthenticationHandler` supplies `member` when none.
    public string[] Roles { get; set; } = [];
}
```

### `src/backend/packages/<PROJECT>.Api/Security/PlaceholderAuthenticationHandlers.cs`

Placeholders only: Development signs in a configured user; every other environment authenticates nobody (401 on secured routes).  Replace with your real scheme.

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace <PROJECT>.Api;

/// <summary>
/// Development only: authenticates every request as the configured <see cref="DevelopmentUserOptions"/>,
/// so the API and the front-end work locally without an identity provider.  Never registered
/// outside the Development environment.
/// </summary>
internal sealed class DevelopmentAuthenticationHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder,
    IOptions<DevelopmentUserOptions> user
) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var dev = user.Value;
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, dev.Subject),
            new(ClaimTypes.Name, dev.Name),
        };
        string[] roles = dev.Roles.Length > 0 ? dev.Roles : [Roles.Member];
        claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, Scheme.Name));

        return Task.FromResult(
            AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name))
        );
    }
}

/// <summary>
/// The default scheme everywhere except Development until the project registers a real one:
/// nobody is authenticated, so every secured route answers 401 and every forbid answers 403
/// (instead of the framework's 500 for "no authentication scheme").
/// </summary>
internal sealed class UnconfiguredAuthenticationHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder
) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync() =>
        Task.FromResult(AuthenticateResult.NoResult());
}
```

### `src/backend/packages/<PROJECT>.Api/Security/SecuritySetup.cs`

`Add<PROJECT>Authorization` (policies, result handler), `Add<PROJECT>Authentication` (the placeholder: replace this call with cookie, JWT bearer, or OpenID Connect), `Use<PROJECT>Security` (authentication, then authorization).

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace <PROJECT>.Api;

public static class SecuritySetup
{
    public const string DevelopmentScheme = "Development";

    public const string UnconfiguredScheme = "Unconfigured";

    extension(IServiceCollection services)
    {
        /// <summary>
        /// The policies and the hidden-route result handler: everything the route groups need,
        /// independent of how users sign in.
        /// </summary>
        public IServiceCollection Add<PROJECT>Authorization()
        {
            services.AddAuthorization(options =>
                options.AddPolicy(Policies.Admin, policy => policy.RequireRole(Roles.Admin))
            );
            services.AddSingleton<
                IAuthorizationMiddlewareResultHandler,
                HiddenRouteAuthorizationResultHandler
            >();

            return services;
        }

        /// <summary>
        /// Placeholder authentication.  Development signs in the configured user
        /// (`Security:DevelopmentUser`); every other environment denies until the project replaces
        /// this call with its real scheme (cookie, JWT bearer, OpenID Connect) as the default.
        /// </summary>
        public IServiceCollection Add<PROJECT>Authentication(
            IHostEnvironment environment,
            IConfiguration configuration
        )
        {
            if (environment.IsDevelopment())
            {
                services
                    .AddOptions<DevelopmentUserOptions>()
                    .Bind(configuration.GetSection("Security:DevelopmentUser"));
                services
                    .AddAuthentication(DevelopmentScheme)
                    .AddScheme<AuthenticationSchemeOptions, DevelopmentAuthenticationHandler>(
                        DevelopmentScheme,
                        null
                    );

                return services;
            }

            services
                .AddAuthentication(UnconfiguredScheme)
                .AddScheme<AuthenticationSchemeOptions, UnconfiguredAuthenticationHandler>(
                    UnconfiguredScheme,
                    null
                );

            return services;
        }
    }

    extension(WebApplication app)
    {
        /// <summary>
        /// Adds authentication then authorization to the pipeline.  Call after the exception
        /// handler and CORS, before mapping endpoints.
        /// </summary>
        public WebApplication Use<PROJECT>Security()
        {
            app.UseAuthentication();
            app.UseAuthorization();

            return app;
        }
    }
}
```

## Api tests: the contract lint and route security

### `src/backend/packages/<PROJECT>.Api.Tests/<PROJECT>.Api.Tests.csproj`

`Microsoft.AspNetCore.TestHost` hosts the pipeline in memory (no Kestrel, no database).

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.AspNetCore.TestHost" />
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Api\<PROJECT>.Api.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Api.Tests/OpenApiDocumentTests.cs`

Lints the committed OpenAPI document: every operation has an `operationId`, a short summary ending in a period, and a description of one to six sentences in at most two paragraphs.  It reads `src/app/src/api/*-api.json`, which the runtime build writes, so run it after a build (`dotnet test --solution` builds first).  A violation fails `mise run test` and CI, not `dotnet build`.

```csharp
using System.Text.Json;
using System.Text.RegularExpressions;

namespace <PROJECT>.Api.Tests;

/// <summary>
/// Lints the committed OpenAPI document (written by the runtime build into the front-end's
/// `src/api`): every operation is named, summarized, and described.  The descriptions are the API
/// documentation, shown in Scalar and as JSDoc on every generated client function.
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Api.Tests
/// </summary>
public sealed partial class OpenApiDocumentTests
{
    private static readonly string[] Methods = ["get", "put", "post", "delete", "patch"];

    [Test]
    public async Task Operations_HaveAnOperationId()
    {
        var offenders = Operations()
            .Where(op => string.IsNullOrWhiteSpace(op.Id))
            .Select(op => op.Label);

        await Assert.That(offenders).IsEmpty();
    }

    [Test]
    public async Task Operations_HaveAShortSummaryEndingInAPeriod()
    {
        var offenders = Operations()
            .Where(op => string.IsNullOrWhiteSpace(op.Summary) || !op.Summary.EndsWith('.'))
            .Select(op => $"{op.Label}: '{op.Summary}'");

        await Assert.That(offenders).IsEmpty();
    }

    [Test]
    public async Task Operations_HaveADescriptionOfOneToSixSentencesInAtMostTwoParagraphs()
    {
        var offenders = Operations()
            .Where(op => !IsWellFormedDescription(op.Description))
            .Select(op => $"{op.Label}: {Describe(op.Description)}");

        await Assert.That(offenders).IsEmpty();
    }

    private static bool IsWellFormedDescription(string? description)
    {
        if (string.IsNullOrWhiteSpace(description))
        {
            return false;
        }

        var paragraphs = ParagraphBreak().Split(description.Trim()).Length;
        var sentences = CountSentences(description);

        return paragraphs <= 2 && sentences is >= 1 and <= 6;
    }

    private static string Describe(string? description) =>
        string.IsNullOrWhiteSpace(description)
            ? "no description"
            : $"{ParagraphBreak().Split(description.Trim()).Length} paragraph(s), {CountSentences(description)} sentence(s)";

    /// <summary>
    /// Counts sentence terminators, ignoring code spans and common abbreviations.
    /// </summary>
    private static int CountSentences(string text)
    {
        var plain = CodeSpan().Replace(text, "x");
        plain = Abbreviation().Replace(plain, "");

        return SentenceEnd().Count(plain);
    }

    private static List<(
        string Label,
        string? Id,
        string? Summary,
        string? Description
    )> Operations()
    {
        using var document = JsonDocument.Parse(File.ReadAllText(FindOpenApiDocument()));
        var operations = new List<(string, string?, string?, string?)>();

        foreach (var path in document.RootElement.GetProperty("paths").EnumerateObject())
        {
            foreach (
                var operation in path.Value.EnumerateObject().Where(o => Methods.Contains(o.Name))
            )
            {
                string? Text(string name) =>
                    operation.Value.TryGetProperty(name, out var value) ? value.GetString() : null;

                operations.Add(
                    (
                        $"{operation.Name.ToUpperInvariant()} {path.Name}",
                        Text("operationId"),
                        Text("summary"),
                        Text("description")
                    )
                );
            }
        }

        return operations;
    }

    /// <summary>
    /// Finds `src/app/src/api/*-api.json` by walking up from the test binary to the solution root.
    /// </summary>
    private static string FindOpenApiDocument()
    {
        for (
            var dir = new DirectoryInfo(AppContext.BaseDirectory);
            dir is not null;
            dir = dir.Parent
        )
        {
            if (dir.GetFiles("*.slnx").Length == 0)
            {
                continue;
            }

            var apiDir = Path.Combine(dir.FullName, "src", "app", "src", "api");

            return Directory.GetFiles(apiDir, "*-api.json").Single();
        }

        throw new InvalidOperationException("Could not find the solution root (*.slnx).");
    }

    [GeneratedRegex(@"\n\s*\n")]
    private static partial Regex ParagraphBreak();

    [GeneratedRegex(@"`[^`]*`")]
    private static partial Regex CodeSpan();

    [GeneratedRegex(@"\b(e\.g|i\.e|etc|vs)\.", RegexOptions.IgnoreCase)]
    private static partial Regex Abbreviation();

    [GeneratedRegex(@"[.!?](?=\s|$)")]
    private static partial Regex SentenceEnd();
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/RouteSecurityTests.cs`

The route-group rules through the real pipeline: 401, 403, hidden 404, anonymous opt-out.

```csharp
using System.Net;
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace <PROJECT>.Api.Tests;

/// <summary>
/// The route-group rules through a real (in-memory) pipeline: routing, authentication,
/// authorization.  No database and no Kestrel.  Users are simulated with request headers
/// (`X-User`, `X-Roles`) by a test authentication scheme (`ProbeHost`).
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Api.Tests --treenode-filter "/*/*/RouteSecurityTests/*"
/// </summary>
public sealed class RouteSecurityTests
{
    private static HttpRequestMessage Get(
        string path,
        string? user = null,
        string? roles = null,
        string? tenant = null
    ) => ProbeHost.Get(path, user, roles, tenant);

    [Test]
    public async Task SecuredRoute_WithoutAUser_Answers401()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/secret"));

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.Unauthorized);
    }

    [Test]
    public async Task SecuredRoute_WithAUser_Answers200()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/secret", user: "ada"));

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task AnonymousRoute_WithoutAUser_Answers200()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/open"));

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task EndpointRoleRequirement_IsLayeredOnTopOfTheGroup()
    {
        await using var host = await ProbeHost.StartAsync();

        var signedInOnly = await host.Client.SendAsync(Get("/api/reports", user: "ada"));
        var analyst = await host.Client.SendAsync(
            Get("/api/reports", user: "ada", roles: "analyst")
        );

        // Signed in but missing the role: 403, not 404 (only admin routes hide themselves).
        await Assert.That(signedInOnly.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(analyst.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    [Arguments(null, null)]
    [Arguments("ada", "member")]
    public async Task AdminRoute_ForAnyoneButAnAdmin_LooksLikeItDoesNotExist(
        string? user,
        string? roles
    )
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/admin/stats", user, roles));

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.NotFound);
    }

    [Test]
    public async Task AdminRoute_ForAnAdmin_Answers200()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(
            Get("/api/admin/stats", user: "ada", roles: "member,admin")
        );

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task UnmatchedApiRoute_IsA404ForEveryone_NotA401()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/typo"));

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.NotFound);
        await Assert.That(await response.Content.ReadAsStringAsync()).Contains("not_found");
    }
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/RouteGuardPipelineTests.cs`

Route guards through the pipeline: stateless and stateful guards, group-before-route and registration order.

```csharp
using System.Net;
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace <PROJECT>.Api.Tests;

/// <summary>
/// Route guards through the real pipeline: the stateless and stateful guards, the order filters
/// run in, and a guard on a single route.  Routes are in `ProbeEndpoints`.
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Api.Tests --treenode-filter "/*/*/RouteGuardPipelineTests/*"
/// </summary>
public sealed class RouteGuardPipelineTests
{
    private static HttpRequestMessage Get(
        string path,
        string? user = null,
        string? roles = null,
        string? tenant = null
    ) => ProbeHost.Get(path, user, roles, tenant);

    [Test]
    public async Task RouteGuard_ChecksTheRouteValueAgainstTheClaim()
    {
        await using var host = await ProbeHost.StartAsync();

        var noClaim = await host.Client.SendAsync(Get("/api/tenants/t1/items", user: "ada"));
        var other = await host.Client.SendAsync(
            Get("/api/tenants/t1/items", user: "ada", tenant: "t2")
        );
        var own = await host.Client.SendAsync(
            Get("/api/tenants/t1/items", user: "ada", tenant: "t1")
        );

        // Signed in without the claim is 403, not 401: the front-end reads 401 as "nobody".
        await Assert.That(noClaim.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(other.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(own.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task GroupGuard_RunsBeforeTheEndpointsOwnFilters()
    {
        await using var host = await ProbeHost.StartAsync();

        // The endpoint's filter stamps a header when it runs.  A rejected request must never
        // reach it: the cheap group guard is the first gate.
        var rejected = await host.Client.SendAsync(
            Get("/api/tenants/t1/stamped", user: "ada", tenant: "t2")
        );
        var allowed = await host.Client.SendAsync(
            Get("/api/tenants/t1/stamped", user: "ada", tenant: "t1")
        );

        await Assert.That(rejected.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(rejected.Headers.Contains("X-Endpoint-Filter")).IsFalse();
        await Assert.That(allowed.Headers.Contains("X-Endpoint-Filter")).IsTrue();
    }

    [Test]
    public async Task Filters_RunInTheOrderTheyAreRegistered()
    {
        await using var host = await ProbeHost.StartAsync();

        var response = await host.Client.SendAsync(Get("/api/ordered", user: "ada"));

        await Assert.That(string.Join(",", response.Headers.GetValues("X-Order"))).IsEqualTo("a,b");
    }

    [Test]
    public async Task StatefulGuard_ResolvesItsServiceFromDi_AndRunsAfterTheCheapGuard()
    {
        await using var host = await ProbeHost.StartAsync();

        var active = await host.Client.SendAsync(
            Get("/api/tenants/t1/export", user: "ada", tenant: "t1")
        );
        var suspended = await host.Client.SendAsync(
            Get("/api/tenants/t2/export", user: "ada", tenant: "t2")
        );
        var unknown = await host.Client.SendAsync(
            Get("/api/tenants/t9/export", user: "ada", tenant: "t9")
        );
        var someoneElses = await host.Client.SendAsync(
            Get("/api/tenants/t9/export", user: "ada", tenant: "t1")
        );

        await Assert.That(active.StatusCode).IsEqualTo(HttpStatusCode.OK);
        await Assert.That(suspended.StatusCode).IsEqualTo(HttpStatusCode.Conflict);
        await Assert.That(unknown.StatusCode).IsEqualTo(HttpStatusCode.NotFound);
        // The cheap guard answers first (403), so a stranger cannot probe which tenants exist.
        await Assert.That(someoneElses.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
    }

    [Test]
    public async Task RouteGuard_AlsoWorksOnASingleRoute()
    {
        await using var host = await ProbeHost.StartAsync();

        var other = await host.Client.SendAsync(
            Get("/api/orgs/o1/info", user: "ada", tenant: "o2")
        );
        var own = await host.Client.SendAsync(Get("/api/orgs/o1/info", user: "ada", tenant: "o1"));

        await Assert.That(other.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(own.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/PlaceholderAuthenticationTests.cs`

Development signs in the configured user; other environments authenticate nobody (401).

```csharp
using System.Net;
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace <PROJECT>.Api.Tests;

/// <summary>
/// The scaffold's placeholder authentication: Development signs in the configured user; every
/// other environment authenticates nobody (401 on secured routes, never a 500).
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Api.Tests --treenode-filter "/*/*/PlaceholderAuthenticationTests/*"
/// </summary>
public sealed class PlaceholderAuthenticationTests
{
    private static HttpRequestMessage Get(
        string path,
        string? user = null,
        string? roles = null,
        string? tenant = null
    ) => ProbeHost.Get(path, user, roles, tenant);

    [Test]
    public async Task PlaceholderAuthentication_OutsideDevelopment_DeniesEverySecuredRoute()
    {
        await using var host = await ProbeHost.StartAsync("Production", placeholder: true);

        var secured = await host.Client.SendAsync(Get("/api/secret"));
        var open = await host.Client.SendAsync(Get("/api/open"));

        // Nobody can sign in until the project registers a real scheme: 401, not a 500.
        await Assert.That(secured.StatusCode).IsEqualTo(HttpStatusCode.Unauthorized);
        await Assert.That(open.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task PlaceholderAuthentication_InDevelopment_SignsInTheConfiguredUser()
    {
        await using var host = await ProbeHost.StartAsync(
            "Development",
            placeholder: true,
            "member",
            "admin"
        );

        var secured = await host.Client.SendAsync(Get("/api/secret"));
        var admin = await host.Client.SendAsync(Get("/api/admin/stats"));

        await Assert.That(secured.StatusCode).IsEqualTo(HttpStatusCode.OK);
        await Assert.That(admin.StatusCode).IsEqualTo(HttpStatusCode.OK);
    }

    [Test]
    public async Task PlaceholderAuthentication_InDevelopment_DefaultsToTheMemberRole()
    {
        await using var host = await ProbeHost.StartAsync("Development", placeholder: true);

        var secured = await host.Client.SendAsync(Get("/api/secret"));
        var admin = await host.Client.SendAsync(Get("/api/admin/stats"));

        await Assert.That(secured.StatusCode).IsEqualTo(HttpStatusCode.OK);
        await Assert.That(admin.StatusCode).IsEqualTo(HttpStatusCode.NotFound);
    }

    [Test]
    public async Task PlaceholderAuthentication_InDevelopment_HonorsTheConfiguredRoles()
    {
        await using var host = await ProbeHost.StartAsync(
            "Development",
            placeholder: true,
            "member"
        );

        var admin = await host.Client.SendAsync(Get("/api/admin/stats"));

        await Assert.That(admin.StatusCode).IsEqualTo(HttpStatusCode.NotFound);
    }
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/ProbeHost.cs`

An in-memory host with a header-driven test scheme (`X-User`, `X-Roles`, `X-Tenant`).

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace <PROJECT>.Api.Tests;

/// <summary>An in-memory host that maps the probe endpoints (`ProbeEndpoints.cs`) through the real wiring.</summary>
internal sealed class ProbeHost : IAsyncDisposable
{
    private readonly WebApplication _app;

    private ProbeHost(WebApplication app) => _app = app;

    /// <summary>A GET request as the given simulated user (headers read by the test scheme).</summary>
    public static HttpRequestMessage Get(
        string path,
        string? user = null,
        string? roles = null,
        string? tenant = null
    )
    {
        var request = new HttpRequestMessage(HttpMethod.Get, path);

        if (user is not null)
        {
            request.Headers.Add("X-User", user);
        }

        if (roles is not null)
        {
            request.Headers.Add("X-Roles", roles);
        }

        if (tenant is not null)
        {
            request.Headers.Add("X-Tenant", tenant);
        }

        return request;
    }

    public HttpClient Client => _app.GetTestClient();

    /// <param name="environment">The hosting environment name: `Development` or `Production`.</param>
    /// <param name="placeholder">
    /// Use the scaffold's placeholder authentication for `environment` instead of the header
    /// scheme (Development signs in `Security:DevelopmentUser`; anything else denies).
    /// </param>
    /// <param name="developmentRoles">Roles for the Development user (placeholder only).</param>
    public static async Task<ProbeHost> StartAsync(
        string environment = "Development",
        bool placeholder = false,
        params string[] developmentRoles
    )
    {
        var builder = WebApplication.CreateBuilder(
            new WebApplicationOptions { EnvironmentName = environment }
        );
        builder.WebHost.UseTestServer();
        builder.Logging.ClearProviders();

        builder.Services.Add<PROJECT>Authorization();

        if (placeholder)
        {
            for (var i = 0; i < developmentRoles.Length; i++)
            {
                builder.Configuration[$"Security:DevelopmentUser:Roles:{i}"] = developmentRoles[i];
            }

            builder.Services.Add<PROJECT>Authentication(builder.Environment, builder.Configuration);
        }
        else
        {
            // A header-driven scheme stands in for sign-in.
            builder
                .Services.AddAuthentication(HeaderAuthenticationHandler.SchemeName)
                .AddScheme<AuthenticationSchemeOptions, HeaderAuthenticationHandler>(
                    HeaderAuthenticationHandler.SchemeName,
                    null
                );
        }

        builder.Services.AddSingleton<ITenantDirectory, ProbeTenantDirectory>();
        builder.Services.Add<PROJECT>Endpoints(); // finds the probe endpoints

        var app = builder.Build();
        app.Use<PROJECT>Security();
        app.Map<PROJECT>Endpoints("/api");

        await app.StartAsync();

        return new ProbeHost(app);
    }

    public async ValueTask DisposeAsync() => await _app.DisposeAsync();
}

/// <summary>Signs in whoever the request headers describe; nobody when `X-User` is absent.</summary>
internal sealed class HeaderAuthenticationHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder
) : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string SchemeName = "Headers";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-User", out var user))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        List<Claim> claims = [new(ClaimTypes.NameIdentifier, user.ToString())];

        foreach (
            var role in Request
                .Headers["X-Roles"]
                .ToString()
                .Split(',', StringSplitOptions.RemoveEmptyEntries)
        )
        {
            claims.Add(new(ClaimTypes.Role, role));
        }

        if (Request.Headers.TryGetValue("X-Tenant", out var tenant))
        {
            claims.Add(new("tenant", tenant.ToString()));
        }

        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, Scheme.Name));

        return Task.FromResult(
            AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name))
        );
    }
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/ProbeEndpoints.cs`

Probe routes for each rule, plus a stateful (DI) guard example; found by the same endpoint scan as real slices.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

namespace <PROJECT>.Api.Tests;

/// <summary>Routes that exercise each rule; mapped through `Map<PROJECT>Endpoints` like a real slice.</summary>
public sealed class ProbeEndpoints : IEndpoint
{
    public void MapEndpoints(IEndpointRouteBuilder endpoints)
    {
        endpoints.MapGet("/open", () => "open").AllowAnonymous();

        // Inherits the root group: authenticated users only.
        endpoints.MapGet("/secret", () => "secret");

        // A role on top of the group's authentication.
        endpoints
            .MapGet("/reports", () => "reports")
            .RequireAuthorization(p => p.RequireRole("analyst"));

        // A group guard (tenant isolation) for every route of the feature.
        var tenant = endpoints
            .MapGroup("tenants/{tenantId}")
            .RequireRouteMatchesClaim("tenantId", "tenant");

        tenant.MapGet("/items", () => "items");

        // A stateful guard on the routes that change things, after the group's cheap guard.
        tenant.MapGet("/export", () => "export").RequireActiveTenant();
        tenant
            .MapGet("/stamped", () => "stamped")
            .AddEndpointFilter(
                async (context, next) =>
                {
                    context.HttpContext.Response.Headers["X-Endpoint-Filter"] = "ran";

                    return await next(context);
                }
            );

        // The same guard on a single route.
        endpoints
            .MapGet("/orgs/{orgId}/info", () => "info")
            .RequireRouteMatchesClaim("orgId", "tenant");

        // Filters run in the order they are added.
        endpoints
            .MapGet("/ordered", () => "ordered")
            .AddEndpointFilter(
                async (context, next) =>
                {
                    context.HttpContext.Response.Headers.Append("X-Order", "a");

                    return await next(context);
                }
            )
            .AddEndpointFilter(
                async (context, next) =>
                {
                    context.HttpContext.Response.Headers.Append("X-Order", "b");

                    return await next(context);
                }
            );
    }
}

/// <summary>An admin-only slice: mapped onto the `/admin` group by placement alone.</summary>
public sealed class ProbeAdminEndpoints : IAdminEndpoint
{
    public void MapEndpoints(IEndpointRouteBuilder endpoints) =>
        endpoints.MapGet("/stats", () => "stats");
}

/// <summary>What the stateful guard asks: is this tenant known, and is it active?</summary>
public interface ITenantDirectory
{
    /// <summary>True when active, false when suspended, null when unknown.</summary>
    bool? IsActive(string tenantId);
}

internal sealed class ProbeTenantDirectory : ITenantDirectory
{
    private readonly Dictionary<string, bool> _tenants = new() { ["t1"] = true, ["t2"] = false };

    public bool? IsActive(string tenantId) =>
        _tenants.TryGetValue(tenantId, out var active) ? active : null;
}

/// <summary>
/// A stateful guard: it needs a service, so it is resolved from DI per request
/// (`AddEndpointFilter&lt;T&gt;()`).  Unknown tenant: 404.  Suspended: 409.  Register it after the
/// cheap route-matches-claim guard so strangers learn nothing about which tenants exist.
/// </summary>
public sealed class RequireActiveTenantFilter(ITenantDirectory tenants) : IEndpointFilter
{
    public ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next
    )
    {
        var tenantId = context.HttpContext.Request.RouteValues["tenantId"]?.ToString() ?? "";

        return tenants.IsActive(tenantId) switch
        {
            null => ValueTask.FromResult<object?>(TypedResults.NotFound()),
            false => ValueTask.FromResult<object?>(
                TypedResults.Conflict(new ApiError("tenant_suspended", "This tenant is suspended."))
            ),
            true => next(context),
        };
    }
}

public static class ProbeGuardExtensions
{
    extension<TBuilder>(TBuilder builder)
        where TBuilder : IEndpointConventionBuilder
    {
        /// <summary>Requires the route's tenant to be known and active.</summary>
        public TBuilder RequireActiveTenant() =>
            builder.AddEndpointFilter<TBuilder, RequireActiveTenantFilter>();
    }
}
```

### `src/backend/packages/<PROJECT>.Api.Tests/RouteGuardUnitTests.cs`

A guard called directly: build a context, call `InvokeAsync`, assert.

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Api.Tests;

/// <summary>
/// A guard tested directly: build a context with route values and a principal, call
/// `InvokeAsync`, assert the result and whether `next` ran.  No host, no pipeline.
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Api.Tests --treenode-filter "/*/*/RouteGuardUnitTests/*"
/// </summary>
public sealed class RouteGuardUnitTests
{
    private static readonly RequireRouteMatchesClaimFilter Filter = new("tenantId", "tenant");

    private static async Task<(object? Result, bool NextCalled)> Run(
        string? routeTenant,
        string? claimTenant
    )
    {
        var http = new DefaultHttpContext();

        if (routeTenant is not null)
        {
            http.Request.RouteValues["tenantId"] = routeTenant;
        }

        http.User = new ClaimsPrincipal(
            new ClaimsIdentity(
                claimTenant is null ? [] : [new Claim("tenant", claimTenant)],
                "Test"
            )
        );

        var nextCalled = false;
        var result = await Filter.InvokeAsync(
            new DefaultEndpointFilterInvocationContext(http),
            _ =>
            {
                nextCalled = true;

                return ValueTask.FromResult<object?>("next");
            }
        );

        return (result, nextCalled);
    }

    [Test]
    public async Task Guard_WhenRouteMatchesClaim_CallsNext()
    {
        var (result, nextCalled) = await Run(routeTenant: "t1", claimTenant: "t1");

        await Assert.That(result).IsEqualTo("next");
        await Assert.That(nextCalled).IsTrue();
    }

    [Test]
    public async Task Guard_WhenRouteDiffersFromClaim_Forbids_WithoutCallingNext()
    {
        var (result, nextCalled) = await Run(routeTenant: "t1", claimTenant: "t2");

        await Assert.That(result).IsTypeOf<ForbidHttpResult>();
        await Assert.That(nextCalled).IsFalse();
    }

    [Test]
    public async Task Guard_WhenTheCallerHasNoClaim_Forbids_NeverAnswers401()
    {
        // A signed-in caller without the claim: 403.  A 401 would tell the front-end that nobody
        // is signed in and send the user to the login page again.
        var (result, nextCalled) = await Run(routeTenant: "t1", claimTenant: null);

        await Assert.That(result).IsTypeOf<ForbidHttpResult>();
        await Assert.That(nextCalled).IsFalse();
    }

    [Test]
    public async Task Guard_WhenTheRouteHasNoSuchValue_Answers400()
    {
        var (result, _) = await Run(routeTenant: null, claimTenant: "t1");

        await Assert.That(result).IsTypeOf<BadRequest>();
    }
}
```

## Telemetry package

### `src/backend/packages/<PROJECT>.Telemetry/<PROJECT>.Telemetry.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <!-- Logging -->
    <PackageReference Include="Serilog" />
    <PackageReference Include="Serilog.AspNetCore" />
    <PackageReference Include="Serilog.Sinks.Console" />
    <PackageReference Include="Serilog.Sinks.OpenTelemetry" />

    <!-- OpenTelemetry -->
    <PackageReference Include="Npgsql.OpenTelemetry" />
    <PackageReference Include="OpenTelemetry" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" />
    <PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Telemetry/<PROJECT>Telemetry.cs`

The single entry point feature slices call.  It lives here (not in the runtime) so slices can use it without a circular reference.

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using System.Runtime.CompilerServices;

namespace <PROJECT>.Telemetry;

/// <summary>
/// Telemetry entry point for the application: owns the `ActivitySource` and `Meter`.  Feature
/// slices call <see cref="Trace"/> / <see cref="AddEvent"/>; extension methods in
/// <see cref="TelemetryExtensions"/> add `SetTags` and friends.
/// </summary>
public static class <PROJECT>Telemetry
{
    public const string ActivitySourceName = "<PROJECT>";

    public static readonly ActivitySource Tracer = new(ActivitySourceName);

    public static readonly Meter Metrics = new(ActivitySourceName);

    public static Activity? AddEvent(
        (string Key, object? Value)[] tags,
        string? eventName = null,
        [CallerMemberName] string name = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0
    ) => Activity.Current?.AddEvent(tags, eventName, name, filePath, lineNumber);

    public static Activity? Trace(
        (string Key, object? Value)[] tags,
        string? traceName = null,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0
    ) => Tracer.Trace(tags, traceName, memberName, filePath, lineNumber);
}
```

### `src/backend/packages/<PROJECT>.Telemetry/TelemetrySetup.cs`

`AddNpgsqlInstrumentation` is Postgres-specific; swap it for the chosen provider's instrumentation (or remove it when there is no database).

```csharp
using Microsoft.Extensions.DependencyInjection;
using Npgsql;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using Serilog;
using Serilog.Events;

namespace <PROJECT>.Telemetry;

public static class TelemetrySetup
{
    extension(IServiceCollection services)
    {
        /// <summary>
        /// Configures Serilog (console + OTLP) and OpenTelemetry (traces, metrics, logs).  The
        /// OTLP endpoint is supplied by Aspire (`OTEL_EXPORTER_OTLP_ENDPOINT`) in local dev.
        /// </summary>
        public IServiceCollection Add<PROJECT>Telemetry(TelemetryConfig config)
        {
            Dictionary<string, object> attributes = new() { ["service"] = config.ServiceName };

            // Logging
            Log.Logger = new LoggerConfiguration()
                .WriteTo.Console(
                    outputTemplate: "[{Timestamp:HH:mm:ss.fff} {Level:u3}] {Message:lj} ({Here}){NewLine}{Exception}"
                )
                .WriteTo.OpenTelemetry(options => options.ResourceAttributes = attributes)
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .MinimumLevel.Override("Microsoft.Hosting.Lifetime", LogEventLevel.Information)
                .CreateLogger();

            services.AddSerilog();

            // OpenTelemetry
            services
                .AddOpenTelemetry()
                .ConfigureResource(res =>
                    res.AddService(config.ServiceName).AddAttributes(attributes)
                )
                .WithTracing(builder =>
                    builder
                        .AddSource(config.ActivitySourceName)
                        .AddAspNetCoreInstrumentation(options => options.RecordException = true)
                        .AddHttpClientInstrumentation()
                        .AddEntityFrameworkCoreInstrumentation()
                )
                .WithMetrics(builder =>
                    builder
                        .AddMeter("*")
                        .AddAspNetCoreInstrumentation()
                        .AddHttpClientInstrumentation()
                        .AddNpgsqlInstrumentation()
                )
                .WithLogging()
                .UseOtlpExporter();

            return services;
        }
    }
}
```

### `src/backend/packages/<PROJECT>.Telemetry/TelemetryConfig.cs`

```csharp
namespace <PROJECT>.Telemetry;

/// <summary>
/// Payload for `Add<PROJECT>Telemetry`.
/// </summary>
public record TelemetryConfig
{
    /// <summary>Reported as `service.name` on every span, metric, and log.</summary>
    public required string ServiceName { get; init; }

    /// <summary>Must match the `ActivitySource` name used by <see cref="<PROJECT>Telemetry"/>.</summary>
    public required string ActivitySourceName { get; init; }
}
```

### `src/backend/packages/<PROJECT>.Telemetry/LoggerExtensions.cs`

```csharp
using System.Runtime.CompilerServices;
using ILogger = Serilog.ILogger;

namespace <PROJECT>.Telemetry;

/// <summary>
/// Captures the call site (member, file, line) on a log event.  The console template renders
/// `{Here}`; the OpenTelemetry sink exports the other properties.
/// </summary>
public static class LoggerExtensions
{
    public static ILogger Here(
        this ILogger logger,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string sourceFilePath = "",
        [CallerLineNumber] int sourceLineNumber = 0
    )
    {
        var srcFile = Path.GetFileName(sourceFilePath);
        var here = $" {srcFile}:{memberName}@{sourceLineNumber}";

        return logger
            .ForContext("Here", here)
            .ForContext("MemberName", memberName)
            .ForContext("FilePath", sourceFilePath)
            .ForContext("LineNumber", sourceLineNumber);
    }
}
```

### `src/backend/packages/<PROJECT>.Telemetry/TelemetryExtensions.cs`

```csharp
using System.Diagnostics;
using System.Runtime.CompilerServices;

namespace <PROJECT>.Telemetry;

/// <summary>
/// Convenience methods over `Activity` / `ActivitySource`.  Call sites are captured
/// automatically as `code.member`, `code.file_path`, and `code.line_number` tags.
/// </summary>
public static class TelemetryExtensions
{
    // Extension block for the ambient activity; access via `Activity.Current`
    extension(Activity? activity)
    {
        /// <summary>
        /// Set tags on the activity.
        /// </summary>
        /// <param name="tags">Key-value pairs to attach.</param>
        /// <returns>The current activity with the applied tags.</returns>
        public Activity? SetTags(params (string Key, object? Value)[] tags)
        {
            foreach (var (key, value) in tags)
            {
                activity?.SetTag(key, value);
            }

            return activity;
        }

        /// <summary>
        /// Attach a timestamped, point-in-time event to the activity.
        /// </summary>
        /// <param name="tags">Key-value pairs to attach to the event.</param>
        /// <param name="eventName">The event name; defaults to `{member}@{file}:{line}`.</param>
        /// <param name="name">Caller member name (set automatically; do not pass).</param>
        /// <param name="filePath">Caller file path (set automatically; do not pass).</param>
        /// <param name="lineNumber">Caller line number (set automatically; do not pass).</param>
        /// <returns>The current activity with the added event.</returns>
        public Activity? AddEvent(
            (string Key, object? Value)[] tags,
            string? eventName = null,
            [CallerMemberName] string name = "",
            [CallerFilePath] string filePath = "",
            [CallerLineNumber] int lineNumber = 0
        )
        {
            var effectiveName = eventName ?? $"{name}@{Path.GetFileName(filePath)}:{lineNumber}";

            activity?.AddEvent(
                new ActivityEvent(
                    effectiveName,
                    tags:
                    [
                        .. tags.Select(tag => new KeyValuePair<string, object?>(
                            tag.Key,
                            tag.Value
                        )),
                        new KeyValuePair<string, object?>("code.member", name),
                        new KeyValuePair<string, object?>("code.file_path", filePath),
                        new KeyValuePair<string, object?>("code.line_number", lineNumber),
                    ]
                )
            );

            return activity;
        }
    }

    extension(ActivitySource source)
    {
        /// <summary>
        /// Start a child span under `Activity.Current`.  Use as
        /// `using var activity = <PROJECT>Telemetry.Trace(tags: [("key", "value")]);`.
        /// The `using` is required to finalize the span's duration.
        /// </summary>
        /// <param name="tags">Tags to attach to the span.</param>
        /// <param name="traceName">The span name; defaults to `{file}#{member}`.</param>
        /// <param name="memberName">Caller member name (set automatically; do not pass).</param>
        /// <param name="filePath">Caller file path (set automatically; do not pass).</param>
        /// <param name="lineNumber">Caller line number (set automatically; do not pass).</param>
        public Activity? Trace(
            (string Key, object? Value)[] tags,
            string? traceName = null,
            [CallerMemberName] string memberName = "",
            [CallerFilePath] string filePath = "",
            [CallerLineNumber] int lineNumber = 0
        ) =>
            source.StartActivity(
                traceName ?? $"{Path.GetFileName(filePath)}#{memberName}",
                kind: ActivityKind.Internal,
                parentContext: Activity.Current?.Context ?? default,
                tags:
                [
                    .. tags.Select(tag => new KeyValuePair<string, object?>(tag.Key, tag.Value)),
                    new KeyValuePair<string, object?>("code.member", memberName),
                    new KeyValuePair<string, object?>("code.file_path", filePath),
                    new KeyValuePair<string, object?>("code.line_number", lineNumber),
                ]
            );
    }
}
```

## First feature slice: Health

### `src/backend/packages/<PROJECT>.Health/<PROJECT>.Health.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <InternalsVisibleTo Include="<PROJECT>.Health.Tests" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Api\<PROJECT>.Api.csproj" />
    <ProjectReference Include="..\<PROJECT>.Telemetry\<PROJECT>.Telemetry.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Health/HealthEndpoints.cs`

The one anonymous route: `AllowAnonymous()` with its reason in a `// NOTE:`, and listed in `RouteSecurityLintTests`.  Every endpoint needs `WithName` (the `operationId` and the client function name), `WithSummary`, and `WithDescription`; the rule and wording are in the backend skill, `references/openapi-contract.md`.

```csharp
using <PROJECT>.Api;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

namespace <PROJECT>.Health;

/// <summary>
/// Liveness endpoint.  First feature slice; copy this shape for new slices.
/// </summary>
public sealed class HealthEndpoints : IEndpoint
{
    public void MapEndpoints(IEndpointRouteBuilder endpoints)
    {
        endpoints
            .MapGet("/health", (HealthHandler handler) => handler.HandleAsync())
            // NOTE: anonymous on purpose.  Platform probes and smoke tests call it before any
            // sign-in, and it reveals nothing.  Every other route inherits the root group's
            // `RequireAuthorization()`; opening one is a deliberate act like this one.
            .AllowAnonymous()
            .WithName("GetHealth")
            .WithTags("Health")
            .WithSummary("Check service health.")
            .WithDescription(
                """
                Reports whether the service is up and answering requests, and when the check ran.
                It does not probe dependencies such as the database, so it is suitable for liveness
                checks and smoke tests but not as a readiness signal.
                """
            );
    }
}
```

### `src/backend/packages/<PROJECT>.Health/HealthHandler.cs`

Shape to copy for new slices: one `<Slice>Endpoints : IEndpoint`, one `IEndpointHandler` per operation, contract records with XML docs, a static per-class logger, and `Trace` for spans.

```csharp
using <PROJECT>.Api;
using <PROJECT>.Telemetry;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Health;

/// <summary>
/// Response for a health check.
/// </summary>
/// <param name="Status">The health status of the service.</param>
/// <param name="CheckedAtUtc">When the check was performed.</param>
public sealed record HealthResponse(string Status, DateTimeOffset CheckedAtUtc);

public sealed class HealthHandler : IEndpointHandler
{
    // One logger per class: scoped to the type, no DI needed.  Field name shadows `Serilog.Log`.
    private static readonly Serilog.ILogger Log = Serilog.Log.ForContext<HealthHandler>();

    public Task<Ok<HealthResponse>> HandleAsync()
    {
        // 👇 Start a span here
        using var activity = <PROJECT>Telemetry.Trace(
            tags: [("endpoint", "health")],
            traceName: "HealthCheck"
        );

        // 👇 This log is linked to the span
        Log.Here().Information("Health check requested!");

        return Task.FromResult(
            TypedResults.Ok(new HealthResponse("Healthy", DateTimeOffset.UtcNow))
        );
    }
}
```

### `src/backend/packages/<PROJECT>.Health.Tests/<PROJECT>.Health.Tests.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Health\<PROJECT>.Health.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Health.Tests/HealthHandlerTests.cs`

```csharp
namespace <PROJECT>.Health.Tests;

public sealed class HealthHandlerTests
{
    [Test]
    public async Task Health_WhenCalled_ReturnsHealthy()
    {
        var handler = new HealthHandler();

        var result = await handler.HandleAsync();

        await Assert.That(result.Value).IsNotNull();
        await Assert.That(result.Value!.Status).IsEqualTo("Healthy");
    }
}
```

## Second slice: Identity (who is signed in)

### `src/backend/packages/<PROJECT>.Identity/<PROJECT>.Identity.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <InternalsVisibleTo Include="<PROJECT>.Identity.Tests" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Api\<PROJECT>.Api.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Identity/MeEndpoints.cs`

No `RequireAuthorization()`: the root group provides it.  The front-end calls `GET /me`; a 401 means nobody is signed in.

```csharp
using System.Security.Claims;
using <PROJECT>.Api;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;

namespace <PROJECT>.Identity;

/// <summary>
/// Identity slice: tells the caller who they are.  The front-end uses it to find out whether
/// anyone is signed in.
/// </summary>
public sealed class MeEndpoints : IEndpoint
{
    public void MapEndpoints(IEndpointRouteBuilder endpoints)
    {
        // Secured by the root group: no `RequireAuthorization()` here, and none is needed.
        endpoints
            .MapGet("/me", (ClaimsPrincipal user, GetMeHandler handler) => handler.Handle(user))
            .WithName("GetMe")
            .WithTags("Identity")
            .WithSummary("Get the signed-in user.")
            .WithDescription(
                """
                Returns the signed-in user's subject, display name, and roles, and whether they are
                an administrator. Responds 401 when nobody is signed in, which is how the front-end
                learns that it must send the user to sign in.
                """
            );
    }
}
```

### `src/backend/packages/<PROJECT>.Identity/GetMeHandler.cs`

```csharp
using System.Security.Claims;
using <PROJECT>.Api;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Identity;

/// <summary>
/// The signed-in user, as the front-end needs to see them.
/// </summary>
/// <param name="Subject">Stable identifier of the user from the identity provider.</param>
/// <param name="Name">Display name; empty when the provider sends none.</param>
/// <param name="Roles">Every role claim the user carries, for example `member`, `admin`.</param>
/// <param name="IsAdmin">Whether the user has the admin role (the `/admin` routes accept them).</param>
public sealed record MeResponse(
    string Subject,
    string Name,
    IReadOnlyList<string> Roles,
    bool IsAdmin
);

public sealed class GetMeHandler : IEndpointHandler
{
    public Ok<MeResponse> Handle(ClaimsPrincipal user)
    {
        var roles = user.FindAll(ClaimTypes.Role).Select(claim => claim.Value).ToArray();

        return TypedResults.Ok(
            new MeResponse(
                Subject: user.FindFirstValue(ClaimTypes.NameIdentifier) ?? "",
                Name: user.Identity?.Name ?? "",
                Roles: roles,
                IsAdmin: roles.Contains(Roles.Admin)
            )
        );
    }
}
```

### `src/backend/packages/<PROJECT>.Identity.Tests/<PROJECT>.Identity.Tests.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\<PROJECT>.Identity\<PROJECT>.Identity.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Identity.Tests/GetMeHandlerTests.cs`

```csharp
using System.Security.Claims;

namespace <PROJECT>.Identity.Tests;

/// <summary>
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Identity.Tests
/// </summary>
public sealed class GetMeHandlerTests
{
    private static ClaimsPrincipal User(params string[] roles)
    {
        List<Claim> claims =
        [
            new(ClaimTypes.NameIdentifier, "user-1"),
            new(ClaimTypes.Name, "Ada"),
            .. roles.Select(role => new Claim(ClaimTypes.Role, role)),
        ];

        return new ClaimsPrincipal(new ClaimsIdentity(claims, "Test"));
    }

    [Test]
    public async Task Me_ForAMember_ReturnsTheUserWithoutAdmin()
    {
        var result = new GetMeHandler().Handle(User("member"));

        await Assert.That(result.Value!.Subject).IsEqualTo("user-1");
        await Assert.That(result.Value!.Name).IsEqualTo("Ada");
        await Assert.That(result.Value!.Roles).IsEquivalentTo(["member"]);
        await Assert.That(result.Value!.IsAdmin).IsFalse();
    }

    [Test]
    public async Task Me_ForAnAdmin_FlagsAdmin()
    {
        var result = new GetMeHandler().Handle(User("member", "admin"));

        await Assert.That(result.Value!.IsAdmin).IsTrue();
    }
}
```

## Runtime tests: the routes lint

### `src/backend/packages/<PROJECT>.Runtime.Tests/<PROJECT>.Runtime.Tests.csproj`

References the runtime, so endpoint discovery sees every slice.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.AspNetCore.TestHost" />
    <PackageReference Include="TUnit" />
  </ItemGroup>
  <ItemGroup>
    <!-- The runtime references every slice, so endpoint discovery sees the real endpoint set. -->
    <ProjectReference Include="..\..\runtime\<PROJECT>.Runtime.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/packages/<PROJECT>.Runtime.Tests/RouteSecurityLintTests.cs`

Every real route has authorization or an explicit `AllowAnonymous()`, and the anonymous set equals the reviewed list.  Keep the `Map<PROJECT>Endpoints` prefix the same as `Program.cs`: with the same-origin recipe (`Map<PROJECT>Endpoints("/api")`) change it here too, and the list then needs `GET /api/health` and the catch-all route `HttpSetup` adds under the prefix (print `Routes()` for the exact strings).

```csharp
using <PROJECT>.Api;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.DependencyInjection;

namespace <PROJECT>.Runtime.Tests;

/// <summary>
/// Secure by default, verified: maps the application's real endpoints (every slice the runtime
/// references) and checks each route's security metadata.  Nothing is requested and no database
/// is touched.
/// Run: dotnet run --project src/backend/packages/<PROJECT>.Runtime.Tests
/// </summary>
public sealed class RouteSecurityLintTests
{
    // Routes that are public on purpose, as "METHOD pattern".  Adding one is a security decision:
    // the test fails until it is listed here, so the change shows up in review.
    private static readonly string[] AnonymousRoutes = ["GET /health"];

    private static List<(string Route, bool Secured, bool Anonymous)> Routes()
    {
        var builder = WebApplication.CreateBuilder();
        builder.WebHost.UseTestServer();
        builder.Services.Add<PROJECT>Authorization();
        builder.Services.Add<PROJECT>Endpoints();

        var app = builder.Build();
        app.Map<PROJECT>Endpoints(); // same prefix as `Program.cs`

        // The route builder's data sources hold every mapped endpoint, with group metadata applied.
        return
        [
            .. ((IEndpointRouteBuilder)app)
                .DataSources.SelectMany(source => source.Endpoints)
                .OfType<RouteEndpoint>()
                .Select(endpoint =>
                {
                    var method =
                        endpoint
                            .Metadata.GetMetadata<HttpMethodMetadata>()
                            ?.HttpMethods.FirstOrDefault()
                        ?? "ANY";

                    return (
                        Route: $"{method} {endpoint.RoutePattern.RawText}",
                        Secured: endpoint.Metadata.GetMetadata<IAuthorizeData>() is not null,
                        Anonymous: endpoint.Metadata.GetMetadata<IAllowAnonymous>() is not null
                    );
                }),
        ];
    }

    [Test]
    public async Task TheLintSeesTheRealRoutes()
    {
        // Guards the lint itself: an empty route list would make the checks below pass vacuously.
        await Assert.That(Routes().Select(r => r.Route)).Contains("GET /health");
        await Assert.That(Routes().Select(r => r.Route)).Contains("GET /me");
    }

    [Test]
    public async Task EveryRoute_RequiresAuthorization_OrIsExplicitlyAnonymous()
    {
        var offenders = Routes().Where(r => !r.Secured && !r.Anonymous).Select(r => r.Route);

        await Assert.That(offenders).IsEmpty();
    }

    [Test]
    public async Task AnonymousRoutes_AreExactlyTheReviewedOnes()
    {
        var anonymous = Routes().Where(r => r.Anonymous).Select(r => r.Route);

        await Assert.That(anonymous).IsEquivalentTo(AnonymousRoutes);
    }
}
```

## Runtime

### `src/backend/runtime/<PROJECT>.Runtime.csproj`

`OpenApiDocumentsDirectory` is relative to this csproj's directory (`src/backend/runtime`), so `..\..\app\src\api` resolves to `src/app/src/api`.  Release builds skip document generation, so container builds do not need `src/app`.  Add one `ProjectReference` per feature slice.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <RootNamespace><PROJECT>.Runtime</RootNamespace>
    <UserSecretsId><GUID></UserSecretsId>
    <!-- OpenAPI config: the build writes the document into the front-end, where Kubb reads it. -->
    <OpenApiDocumentsDirectory>..\..\app\src\api</OpenApiDocumentsDirectory>
    <OpenApiGenerateDocumentsOptions>--file-name <project>-api</OpenApiGenerateDocumentsOptions>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)' == 'Release'">
    <!-- Suppress OpenAPI document generation in Release; the target checks this property -->
    <OpenApiGenerateDocuments>false</OpenApiGenerateDocuments>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" />
    <PackageReference Include="Microsoft.Extensions.ApiDescription.Server">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Scalar.AspNetCore" />
  </ItemGroup>
  <ItemGroup>
    <!-- Infrastructure packages -->
    <ProjectReference Include="..\packages\<PROJECT>.Api\<PROJECT>.Api.csproj" />
    <ProjectReference Include="..\packages\<PROJECT>.Config\<PROJECT>.Config.csproj" />
    <ProjectReference Include="..\packages\<PROJECT>.Data\<PROJECT>.Data.csproj" />
    <ProjectReference Include="..\packages\<PROJECT>.Telemetry\<PROJECT>.Telemetry.csproj" />
    <!-- Feature slices: add one reference per slice -->
    <ProjectReference Include="..\packages\<PROJECT>.Health\<PROJECT>.Health.csproj" />
    <ProjectReference Include="..\packages\<PROJECT>.Identity\<PROJECT>.Identity.csproj" />
  </ItemGroup>
</Project>
```

### `src/backend/runtime/Program.cs`

**`isOpenApiGeneration`:** build-time OpenAPI generation (`Microsoft.Extensions.ApiDescription.Server`) runs this entry point under the `GetDocument.Insider` assembly and executes everything up to `app.Run()`.  Anything that needs infrastructure (database connection, migrations, network calls) must be skipped in that mode or `dotnet build` fails.  Remove the data lines if no database was chosen.  Security: `Add<PROJECT>Authorization().Add<PROJECT>Authentication(...)` and `Use<PROJECT>Security()` (before the endpoints); replace the placeholder authentication with the real scheme.  CORS is open to any origin: restrict it before using cookie authentication.

```csharp
using System.Reflection;
using <PROJECT>.Api;
using <PROJECT>.Config;
using <PROJECT>.Data;
using <PROJECT>.Telemetry;
using Scalar.AspNetCore;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Build-time OpenAPI generation (Microsoft.Extensions.ApiDescription.Server) runs this entry
// point under `GetDocument.Insider` and executes everything up to `app.Run()`.  Anything that
// needs infrastructure (database, etc.) must be skipped in that mode or the build fails.
var isOpenApiGeneration = Assembly.GetEntryAssembly()?.GetName().Name == "GetDocument.Insider";

// Settings into the DI container
builder
    .Services.AddOptions<AppSettings>()
    .Bind(builder.Configuration.GetSection(nameof(AppSettings)))
    .ValidateOnStart();

// Unhandled exceptions -> problem details (see `HttpExceptionHandler`)
builder.Services.AddExceptionHandler<HttpExceptionHandler>().AddProblemDetails();

// OpenAPI and CORS.  The Vite dev server calls the API cross-origin, so CORS is open to any
// origin here; restrict the origins before you use cookie authentication.
builder
    .Services.AddOpenApi()
    .AddCors(options =>
        options.AddDefaultPolicy(policy =>
            policy.AllowAnyHeader().AllowAnyMethod().AllowAnyOrigin()
        )
    );

// Logging and telemetry
builder.Services.Add<PROJECT>Telemetry(
    new()
    {
        ServiceName = <PROJECT>Telemetry.ActivitySourceName,
        ActivitySourceName = <PROJECT>Telemetry.ActivitySourceName,
    }
);

// Security: policies and route-group rules (`<PROJECT>.Api`), plus placeholder authentication.
// Development signs in a configured user (`Security:DevelopmentUser`); everywhere else every
// secured route answers 401 until you replace `Add<PROJECT>Authentication` with your real scheme.
builder
    .Services.Add<PROJECT>Authorization()
    .Add<PROJECT>Authentication(builder.Environment, builder.Configuration);

// Endpoint discovery
builder.Services.Add<PROJECT>Endpoints();

// Data access (needs the Aspire-provided connection string)
if (!isOpenApiGeneration)
{
    builder.Services.Add<PROJECT>Data(builder.Configuration);
}

var app = builder.Build();

if (!isOpenApiGeneration)
{
    // Applies pending migrations.
    await app.Services.Use<PROJECT>DataAsync();
}

// OpenAPI document + Scalar UI (dev only; the schema is also generated at build time)
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
}

app.UseExceptionHandler();
app.UseCors();
app.UseSerilogRequestLogging();

// Authentication, then authorization: before the endpoints they protect.
app.Use<PROJECT>Security();

app.Map<PROJECT>Endpoints();

app.Run();
```

### `src/backend/runtime/appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### `src/backend/runtime/appsettings.Development.json`

`Security:DevelopmentUser` is who the Development placeholder signs in (remove `admin` from the roles to try the non-admin experience).

```json
{
  "Security": {
    "DevelopmentUser": {
      "Subject": "dev-user",
      "Name": "Development User",
      "Roles": ["member", "admin"]
    }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### `src/backend/runtime/DebugRepl.cs`

CSharpRepl helpers (step 10).  `Get<T>()` in the REPL resolves singletons/transients only; these give scoped services (DbContext, stores) a scope.  Remove with step 10 if the REPL hook is declined.

```csharp
using System.Text.Json;

// NOTE: Deliberately the bare `<PROJECT>` namespace rather than `<PROJECT>.Runtime`.  These members are
// typed into REPL eval strings by hand, and extension members need a `using` directive (they
// cannot be called fully qualified), so the shortest import is `using <PROJECT>;`.
namespace <PROJECT>;

/// <summary>
/// Scoped-service helpers on <see cref="IServiceProvider"/> for CSharpRepl diagnostics.
/// </summary>
/// <remarks>
/// CSharpRepl attaches to the running backend (see the `csharprepl` reference in the
/// `hg-mdn-backend` skill) and exposes the root provider as the global `services`.  `Get&lt;T&gt;()`
/// resolves singletons and transients only; scoped services (DbContext, stores) need a scope, so
/// these helpers wrap "create scope, resolve, invoke, dispose".  Inert unless called; anyone who
/// can evaluate code in the process can already create scopes.
/// </remarks>
public static class DebugReplExtensions
{
    private static readonly JsonSerializerOptions Indented = new() { WriteIndented = true };

    extension(IServiceProvider provider)
    {
        /// <summary>
        /// Resolves <typeparamref name="TService"/> in a fresh scope, invokes the callback, and
        /// disposes the scope.  Return plain data, not the service: the scope ends on return.
        /// The return type must be explicit (C# has no partial type inference):
        /// `services.Use&lt;AppDbContext, int&gt;(db =&gt; db.Database.GetMigrations().Count())`.
        /// </summary>
        public T Use<TService, T>(Func<TService, T> action)
            where TService : notnull
        {
            using var scope = provider.CreateScope();

            return action(scope.ServiceProvider.GetRequiredService<TService>());
        }

        /// <summary>
        /// Async variant of <c>Use</c>; the scope stays alive until the callback's task completes.
        /// </summary>
        public async Task<T> UseAsync<TService, T>(Func<TService, Task<T>> action)
            where TService : notnull
        {
            using var scope = provider.CreateScope();

            // The await is load-bearing: returning the task directly would dispose the scope
            // before the callback completes.
            return await action(scope.ServiceProvider.GetRequiredService<TService>());
        }

        /// <summary>
        /// Two services in one shared scope (nesting two <c>UseAsync</c> calls would create two scopes).
        /// </summary>
        public async Task<T> UseAsync<TService1, TService2, T>(
            Func<TService1, TService2, Task<T>> action
        )
            where TService1 : notnull
            where TService2 : notnull
        {
            using var scope = provider.CreateScope();
            var sp = scope.ServiceProvider;

            return await action(
                sp.GetRequiredService<TService1>(),
                sp.GetRequiredService<TService2>()
            );
        }

        /// <summary>
        /// Hands the raw scoped provider to the callback: keyed services, more than two services,
        /// or <c>IEnumerable&lt;T&gt;</c> resolution.
        /// </summary>
        public async Task<T> UseScopedAsync<T>(Func<IServiceProvider, Task<T>> action)
        {
            using var scope = provider.CreateScope();

            return await action(scope.ServiceProvider);
        }
    }

    /// <summary>
    /// Awaits the task and serializes the result to compact JSON; <see cref="RawOutput"/> prints
    /// unescaped in the REPL (no `\"` around every quote).
    /// </summary>
    public static async Task<RawOutput> AsJsonAsync<T>(this Task<T> task) =>
        new(JsonSerializer.Serialize(await task));

    /// <summary>
    /// Same as <see cref="AsJsonAsync{T}"/>, indented.
    /// </summary>
    public static async Task<RawOutput> AsJsonIndentedAsync<T>(this Task<T> task) =>
        new(JsonSerializer.Serialize(await task, Indented));

    /// <summary>
    /// Wraps a string so the REPL pretty-printer renders it as-is.
    /// </summary>
    public readonly struct RawOutput(string value)
    {
        public override string ToString() => value;
    }
}
```

### `src/backend/runtime/Properties/launchSettings.json`

`applicationUrl` is the single source of the backend port: it feeds `kubb.config.ts` (`baseURL`).  Pick a free port per project.

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": false,
      "applicationUrl": "http://localhost:<BACKEND_API_PORT>",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

## Solution

### `<project>.slnx`

Add every new project; keep the folder structure mirroring the directories.

```xml
<Solution>
  <Folder Name="/src/backend/packages/">
    <Project Path="src/backend/packages/<PROJECT>.Api/<PROJECT>.Api.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Api.Tests/<PROJECT>.Api.Tests.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Config/<PROJECT>.Config.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Data/<PROJECT>.Data.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Data.Tests/<PROJECT>.Data.Tests.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Health/<PROJECT>.Health.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Health.Tests/<PROJECT>.Health.Tests.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Identity/<PROJECT>.Identity.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Identity.Tests/<PROJECT>.Identity.Tests.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Runtime.Tests/<PROJECT>.Runtime.Tests.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Telemetry/<PROJECT>.Telemetry.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Testing/<PROJECT>.Testing.csproj" />
    <Project Path="src/backend/packages/<PROJECT>.Testing.Tests/<PROJECT>.Testing.Tests.csproj" />
  </Folder>
  <Folder Name="/src/backend/runtime/">
    <Project Path="src/backend/runtime/<PROJECT>.Runtime.csproj" />
  </Folder>
  <Project Path="host/host.csproj" />
</Solution>
```
