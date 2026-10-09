# Route Security: Groups and Guards

Security is applied by **route groups** and **guards**, not by decorating each endpoint.  A group carries the rules for every route mapped onto it; a guard is a small reusable check (an `IEndpointFilter`) attached to a group or one route.  How users *sign in* (cookie, JWT bearer, OpenID Connect) is a project decision that plugs in at one seam (`Add<PROJECT>Authentication`); everything here works with any scheme.

The classes live in `<PROJECT>.Api/Security` (full files: `hg-mdn-scaffold`, `references/backend-templates.md`).  Tests: `RouteSecurityTests`, `RouteGuardPipelineTests` and `PlaceholderAuthenticationTests` in `<PROJECT>.Api.Tests`; `RouteSecurityLintTests` in `<PROJECT>.Runtime.Tests`.

## The layers

| Layer | Applied at | Mechanism | Example |
|---|---|---|---|
| **Authenticated by default** | the root group | `MapApiRoot()` = `MapGroup(prefix).RequireAuthorization()` | every route |
| **Explicit exceptions** | one route | `.AllowAnonymous()` + a `// NOTE:` | `GET /health` |
| **Admin only** | the `/admin` subgroup, by *placement* | admin policy + hidden 404 | `IAdminEndpoint` |
| **Feature or tenant scope** | the slice's own group | a group guard | `RequireRouteMatchesClaim` |
| **Roles and policies** | one route | `.RequireAuthorization(p => p.RequireRole(...))` | `/reports` in `ProbeEndpoints` (test project) |
| **State checks** | one route (usually mutations) | a guard resolved from DI | `RequireActiveTenantFilter` (test project) |

Put on the **group** what is true of every route in it; put on the **route** what is specific to it.

## Secure by default

`Map<PROJECT>Endpoints` (`HttpSetup.cs`) builds the root group and hands *that group*, never the bare application, to every slice:

`src/backend/packages/<PROJECT>.Api/HttpSetup.cs (excerpt)`

```csharp
            // Secure by default: every route under the root group needs an authenticated user
            // unless it opts out with `AllowAnonymous()`.  The admin group sits under it.
            var root = app.MapApiRoot(prefix);
            var admin = root.MapAdminGroup();
```

`src/backend/packages/<PROJECT>.Api/Security/RouteGroups.cs`

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

- A slice's `MapEndpoints` receives `root` (or `admin` when it implements `IAdminEndpoint`), so `endpoints.MapGet("/notes", ...)` is authenticated with no attribute.  Securing a route cannot be forgotten; **opening** one is deliberate.
- Mapped outside the root group and public on purpose: `MapOpenApi` and the Scalar UI (Development only, in `Program.cs`).  Same-origin hosting adds the SPA fallback (`hg-mdn-frontend`, `references/config-deploy.md`).
- When the API is mapped under a prefix (same-origin hosting), an unmatched-route catch-all under it is `AllowAnonymous()`, so a typo is a 404 for everyone, not a 401.

## Opting out: `AllowAnonymous`

`HealthEndpoints.cs` is the template's one public route:

`src/backend/packages/<PROJECT>.Health/HealthEndpoints.cs (excerpt)`

```csharp
            .MapGet("/health", (HealthHandler handler) => handler.HandleAsync())
            // NOTE: anonymous on purpose.  Platform probes and smoke tests call it before any
            // sign-in, and it reveals nothing.  Every other route inherits the root group's
            // `RequireAuthorization()`; opening one is a deliberate act like this one.
            .AllowAnonymous()
```

- Say **why** in a `// NOTE:` next to the call: a public route is a security decision a reviewer should see.
- `RouteSecurityLintTests` keeps a list of the anonymous routes and fails when the real set differs, so adding one cannot slip through unreviewed (see Testing).

## Admin routes: security by placement

A slice that implements the marker interface is mapped onto the `/admin` group instead of the root:

`src/backend/packages/<PROJECT>.Api/Security/IAdminEndpoint.cs`

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

- The `/admin` group carries the admin policy **and** `HiddenRouteMetadata`.  An admin endpoint maps paths relative to the group (`MapGet("/stats", ...)` becomes `GET /admin/stats`) and states no authorization of its own, so it cannot loosen the rule.
- **Hide what you protect.**  `HiddenRouteAuthorizationResultHandler` answers a failed authorization on a hidden route with **404**, so an admin route is not advertised to anonymous or ordinary users.  Every other route keeps the framework's 401 and 403.  It is narrow on purpose: it changes the result only for endpoints that carry the metadata.
- The policy (`Policies.Admin`) requires the `admin` role, so admin routes need that role in the principal (see "The authentication seam").
- **If admin rights can be revoked, do not rely on a role baked into a token or cookie**: evaluate the requirement live (an `AuthorizationHandler<TRequirement>` that checks the current configuration or store on every request), so a revocation takes effect immediately.

## Guards: `IEndpointFilter`

A guard is an endpoint filter, exposed as an extension method so a slice reads like configuration.

**A stateless guard** (no I/O): a route value must equal one of the caller's claims, the usual shape of tenant isolation.  `RequireRouteMatchesClaimFilter` answers 400 when the route has no such value and 403 when the caller has no such claim or a different one (never 401: the root group already guarantees a signed-in caller).

`src/backend/packages/<PROJECT>.Api/Security/RouteGuardExtensions.cs`

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

One **generic** extension block (`where TBuilder : IEndpointConventionBuilder`) serves a whole group and a single route alike; no duplicate `RouteGroupBuilder` and `RouteHandlerBuilder` overloads.  Guards return their answers as `TypedResults`; what each status means is in "Status codes at a glance".

**A stateful guard** needs a service (a store, a cache), so it is resolved from DI per request: `AddEndpointFilter<TFilter>()` constructs the filter through DI, and its constructor can take `ITenantDirectory`, a store, or a `HybridCache`.  `RequireActiveTenantFilter` (in `ProbeEndpoints.cs`, test project) maps the tenant's state to a result:

`src/backend/packages/<PROJECT>.Api.Tests/ProbeEndpoints.cs (excerpt)`

```csharp
            null => ValueTask.FromResult<object?>(TypedResults.NotFound()),
            false => ValueTask.FromResult<object?>(
                TypedResults.Conflict(new ApiError("tenant_suspended", "This tenant is suspended."))
            ),
            true => next(context),
```

Apply a stateful guard to the **routes that change things** (`.RequireActiveTenant()` on the mutation), not to a whole read-mostly group, and cache the lookup briefly if it is hot.

**Order matters.**  Filters added to a *group* run before filters added to a *route*, and filters on the same builder run **in the order they were added**.  Register the cheap, I/O-free guard first and the stateful one after it: a caller who fails the cheap check (403) must never reach the lookup, so they cannot probe which tenants exist.

## Structuring a slice

A feature maps one group and layers the rules, general to specific:

```csharp
var documents = endpoints
    .MapGroup("tenants/{tenantId}/documents")          // the feature group
    .WithTags("Documents")
    .RequireRouteMatchesClaim("tenantId", "tenant");   // every route: caller belongs to the tenant

documents.MapGet("/", ...);                            // reads: authenticated + tenant (inherited)

documents
    .MapPost("/", ...)                                 // mutations: add a role and a state check
    .RequireAuthorization(p => p.RequireRole("editor"))
    .RequireActiveTenant();
```

Name the role in the endpoint's `WithDescription` when it is not the default (signed in), so the generated API documentation and client JSDoc say who may call it (`openapi-contract.md`).

## The authentication seam

`SecuritySetup.cs` holds three calls:

- `Add<PROJECT>Authorization()` registers the admin policy and the hidden-route result handler: everything the groups need, independent of how people sign in.
- `Add<PROJECT>Authentication(environment, configuration)` is a **placeholder**.  In Development it signs in the user configured under `Security:DevelopmentUser` (`appsettings.Development.json`: subject, name, roles) so the API and the front-end work locally with no identity provider; remove `admin` from the roles to try the non-admin experience.  In every other environment it registers a scheme that authenticates nobody, so every secured route answers **401** (not the framework's 500 for "no authentication scheme") until you replace the call with your real scheme as the default.
- `Use<PROJECT>Security()` adds authentication then authorization to the pipeline.

Replacing the placeholder changes nothing else: the groups, guards, and policies already apply.  The scheme must produce the role and tenant claims the policies and guards read.

- **Pipeline order** in `Program.cs`: exception handler, (static files), CORS, `Use<PROJECT>Security()`, then the mapped endpoints.
- **Cookie auth with a cross-origin front-end** needs credentialed CORS and the same-origin option: `hg-mdn-frontend`, `references/config-deploy.md`.
- **Roles that can change mid-session:** load them per request in middleware between `UseAuthentication()` and `UseAuthorization()` and add them to the principal; never trust a stale token.  Do not reject there: policies and guards reject, so recovery routes (`GET /me`) stay reachable.

## Testing

Three layers, fastest first.  Run one class with `--treenode-filter "/*/*/<Class>/*"`.

**1. Unit-test a guard directly** (`RouteGuardUnitTests`).  Build a `DefaultHttpContext` with route values and a principal, call `InvokeAsync`, assert the result type and whether `next` ran.  No host.  Covers each status path, including ones a real request cannot produce (a missing route value):

`src/backend/packages/<PROJECT>.Api.Tests/RouteGuardUnitTests.cs (excerpt)`

```csharp
        // A signed-in caller without the claim: 403.  A 401 would tell the front-end that nobody
        // is signed in and send the user to the login page again.
        var (result, nextCalled) = await Run(routeTenant: "t1", claimTenant: null);

        await Assert.That(result).IsTypeOf<ForbidHttpResult>();
        await Assert.That(nextCalled).IsFalse();
```

**2. Pipeline tests with an in-memory host** (`Microsoft.AspNetCore.TestHost`): real routing, authentication, authorization, and filters, no Kestrel, no database.  `ProbeHost` and the probe endpoints (`ProbeEndpoints`, `ProbeAdminEndpoints`) are in the test project; a header-driven scheme stands in for sign-in, and the probe endpoints go through the same endpoint scan as real slices, so the test exercises `Map<PROJECT>Endpoints` itself.

`src/backend/packages/<PROJECT>.Api.Tests/RouteSecurityTests.cs (excerpt)`

```csharp
        var signedInOnly = await host.Client.SendAsync(Get("/api/reports", user: "ada"));
        var analyst = await host.Client.SendAsync(
            Get("/api/reports", user: "ada", roles: "analyst")
        );

        // Signed in but missing the role: 403, not 404 (only admin routes hide themselves).
        await Assert.That(signedInOnly.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
        await Assert.That(analyst.StatusCode).IsEqualTo(HttpStatusCode.OK);
```

| Class | Pins |
|---|---|
| `RouteSecurityTests` | 401 without a user, 200 with one, the anonymous opt-out, a role layered on the group (403, not 404), admin routes (404 for anonymous and non-admins, 200 for admins), the unmatched-route 404 |
| `RouteGuardPipelineTests` | the stateless guard (400, 403, 200), a group guard running before the route's own filters, filters running in registration order, the stateful DI guard after the cheap one (a stranger gets 403, never a lookup), a guard on a single route |
| `PlaceholderAuthenticationTests` | Development signs in the configured user (roles from `Security:DevelopmentUser`, `member` by default); Production authenticates nobody: 401, never a 500 |

Assert the **status code** and, for ordering, a side effect (a response header) that proves a filter did or did not run.

**3. Lint the real routes** (`RouteSecurityLintTests`): map every real slice and check each route's metadata.  "Secure by default" is a claim; this test makes it a checked fact.  Opening a route means editing the list below, which a reviewer sees:

`src/backend/packages/<PROJECT>.Runtime.Tests/RouteSecurityLintTests.cs (excerpt)`

```csharp
    // Routes that are public on purpose, as "METHOD pattern".  Adding one is a security decision:
    // the test fails until it is listed here, so the change shows up in review.
    private static readonly string[] AnonymousRoutes = ["GET /health"];
```

- `EveryRoute_RequiresAuthorization_OrIsExplicitlyAnonymous` fails if a route has neither authorization nor an explicit opt-out.
- `AnonymousRoutes_AreExactlyTheReviewedOnes` fails when the anonymous set differs from the list.
- `TheLintSeesTheRealRoutes` guards the lint itself: an empty route list would make the other two pass for nothing.
- The test project references the runtime, so endpoint discovery sees every slice; keep its `Map<PROJECT>Endpoints(...)` prefix the same as `Program.cs`.

## Status codes at a glance

| Situation | Answer |
|---|---|
| Nobody is signed in | **401** (the front-end sends the user to sign in) |
| Signed in, lacks the role | **403** |
| Admin route, not an admin | **404** (hidden) |
| Route value is another tenant's, or the caller has no such claim | **403** from the group guard, before any lookup |
| Route has no such value | **400** |
| Row that does not exist, or that the caller may not see | **404** from the handler (existence is not leaked) |
| Exists, wrong state (suspended) | **409** with an `ApiError` code |
| Typo under the API prefix | **404** JSON, never 401 |

## Checklist: a new route or slice

1. Map onto the group you are given; add no `RequireAuthorization()` just to be safe (it is already there).
2. Admin-only?  Implement `IAdminEndpoint`; map relative paths.
3. Scoped to a tenant or owner?  A group guard (`RequireRouteMatchesClaim`) on the feature group.
4. A role, or a state check?  On the route (mutations), cheap guard first.
5. Public?  `AllowAnonymous()` with a `// NOTE:` saying why, **and** add it to the list in `RouteSecurityLintTests`.
6. Name the role in `WithDescription` when it is not the default (signed in).
7. Add pipeline-test cases for any new guard or rule; the lint runs for every route automatically.
