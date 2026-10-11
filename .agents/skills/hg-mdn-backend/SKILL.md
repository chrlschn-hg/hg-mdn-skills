---
name: hg-mdn-backend
description: Build and debug the modern .NET (C# 14) backend: feature slices, minimal API endpoints and handlers, EF Core data access, configuration, OpenTelemetry/Serilog, TUnit tests on Microsoft Testing Platform (not xUnit), the OpenAPI to Kubb contract, and debugging the running app with the Aspire CLI and CSharpRepl.  Use for any work in `src/backend`.
---

# Building the Modern .NET Backend

## Overview

The backend is a .NET 10 (C# 14) web API organized as **feature slices**: one project per capability holding its domain types, storage interface, endpoints, and handlers.  A lean runtime project wires the slices together; a data project implements their storage; an Aspire host runs it all locally.

- Minimal APIs; one `IEndpoint` per slice, one `IEndpointHandler` per operation, `TypedResults` for every response
- EF Core (Postgres by default), snake_case, store interfaces owned by the slices
- Serilog + OpenTelemetry with call-site tagging; logs and spans land in the Aspire dashboard
- The C# endpoints are the API contract: OpenAPI is generated at build and Kubb generates the TypeScript client
- **TUnit on Microsoft Testing Platform for tests.  Never xUnit.**  Real databases through Testcontainers
- Debugging by CLI: `aspire logs` / `aspire otel`, and CSharpRepl attached to the running process

If the repository is missing foundational pieces (Mise, the Aspire host, `src/backend` layout), use the `hg-mdn-scaffold` skill first.  The Notes example in the references is a worked example, not part of the scaffolded repository (`<PROJECT>.Health` and `<PROJECT>.Identity` are).

## When to Use

- Adding or changing endpoints, handlers, contracts, domain types, stores, or migrations
- Writing or running backend tests
- Adding configuration, secrets, logging, spans, or metrics
- Diagnosing a failing request, slow query, or startup problem in the running app
- Changing anything the front-end consumes (the OpenAPI contract)

## Instructions

<rules>
- Read the reference(s) for the work (index below) **before** writing code; copy the shape of an existing slice rather than inventing a layout
- Business logic goes in feature slices, never the runtime project.  Slices depend on `<PROJECT>.Api`, `<PROJECT>.Telemetry`, and interfaces they define; never on EF Core, `<PROJECT>.Data`, or another slice's internals
- **Security comes from the group** (`references/route-security.md`).  Slices map onto the secured root group (signed-in users; write no `RequireAuthorization()`), or onto `/admin` with `IAdminEndpoint` (404 for non-admins).  Tenant scope is a group guard; roles and state checks go on the route, cheap guard first.  Opening a route needs `AllowAnonymous()`, a `// NOTE:`, and a line in `RouteSecurityLintTests`
- **One origin:** the API is mapped under `/api` and the runtime serves the built front-end (`Use<PROJECT>Spa`, `Map<PROJECT>SpaFallback`): no CORS, one container image (`references/spa-hosting.md`).  The build stamps the version and commit the app reports (`references/build-info.md`)
- Authentication is a project choice behind one seam (`Add<PROJECT>Authentication`): Development signs in a configured user; elsewhere secured routes answer 401 until a real scheme replaces the placeholder
- Handlers return `Results<...>` of `TypedResults`; expected failures are `ApiError` results with a distinct `snake_case` code; validate with explicit guard clauses (DataAnnotations on contracts are OpenAPI metadata, not enforced)
- Every endpoint sets `WithName`, `WithSummary`, and `WithDescription`; `<PROJECT>.Api.Tests` fails `mise run test` and CI without them (`references/openapi-contract.md`)
- A model change needs a migration (`references/data-efcore.md`).  **Multi-tenant?  The tenant id is part of every primary key, composite foreign key, and index; stores take `tenantId`** (decide before the first entity).  UUID v7 is the internal key; where people see ids add a unique short **NanoID** (safe lowercase alphanumeric alphabet); where users reorder rows add a **LexoRank** column (`references/ids-and-ordering.md`)  After a contract change: `dotnet build`, then `mise run generate`
- Tests are TUnit: `[Test]`, `await Assert.That(...)` (always awaited), `[Arguments]`.  No `[Fact]`, `[Theory]`, `Assert.Equal`, or xUnit packages
- Test data comes from the **entity graph builder**: customize the real entity with `Action<T>`, never `*Prototype` classes (`references/entity-graph.md`).  Handler tests use hand-written fakes; store tests use the real database, never a mocked `DbContext`
- Logging: the per-class static logger with `Log.Here()`; spans: `using var activity = <PROJECT>Telemetry.Trace(...)`
- **Zero warnings: `TreatWarningsAsErrors`.**  A build error with a rule id is the instruction: fix the cause, never `#pragma`-disable it to finish a task; a rule that keeps coming up becomes a banned symbol or a custom Roslyn analyzer (`references/static-analysis.md`).  Mechanical, repeated code is generated: built-in generators first (`[GeneratedRegex]`, `[LoggerMessage]`, `JsonSerializerContext`, Mapperly), then an incremental generator (`references/source-generators.md`)
- Style is enforced by `.editorconfig` and the build (braces, exhaustive enum switches: name every member, end with `_ => throw new UnreachableException(...)`).  `csharpier format <files>` on everything you touch; `dotnet build` stays warning-free
- Prefer switch expressions and pattern matching, records for contracts, extension blocks for your own types; no `Task.Run` in request code
- Comments say *why*: XML-doc every public type and contract parameter; `<remarks>` for the flow, `// NOTE:` for traps
- Prefer the running app over guessing: read logs and traces with the Aspire CLI, or evaluate code in the process with CSharpRepl
- Definition of done: `mise run check`
</rules>

## Reference Index

| When you are | Read |
|---|---|
| adding a slice or deciding where code goes | `references/feature-slices.md` |
| writing endpoints, handlers, contracts, error results | `references/endpoints.md` |
| securing routes (groups, guards, roles, anonymous routes, the authentication seam) | `references/route-security.md` |
| writing endpoint descriptions, changing the API the front-end calls, or fixing a failing OpenAPI build | `references/openapi-contract.md` |
| serving the built front-end from the runtime (static files, SPA fallback, the `/api` prefix, cache headers, the container image) | `references/spa-hosting.md` |
| the running version, commit, and build time: stamping them in the build, `BuildInfo`, `/api/health`, logs, the Docker build args | `references/build-info.md` |
| touching entities, stores, EF configuration, or migrations; tenant-scoped (multi-tenant) keys | `references/data-efcore.md` |
| a short, user-facing id (NanoID) next to the UUID, or user-reorderable rows (LexoRank, O(1) moves) | `references/ids-and-ordering.md` |
| adding settings, secrets, or connection strings | `references/configuration.md` |
| adding logs, spans, or metrics | `references/telemetry-logging.md` |
| writing or running tests (test types, fakes, run commands) | `references/testing.md` |
| using TUnit features (data sources, hooks, assertions) or translating xUnit habits | `references/tunit.md` |
| creating test data and seeds | `references/entity-graph.md` |
| diagnosing the running app (logs, traces, rebuild, database reset) | `references/debugging-aspire.md` |
| inspecting or patching live code | `references/csharprepl.md` |
| writing C# (switch expressions, pattern matching, types, comments); side-effect-free core: pure decisions, effects passed as delegates | `references/code-style.md` |
| parallel or background work, producers and consumers, locks (default: plain `async`/`await`) | `references/concurrency.md` |
| zero-warning builds, `AnalysisLevel`, banned APIs, writing and testing a custom Roslyn analyzer | `references/static-analysis.md` |
| repetitive code: built-in source generators, writing and testing an incremental generator | `references/source-generators.md` |
| checking a library's real API, signatures, or breaking changes before coding against it: `dnx dotnet-inspect` | `references/inspecting-packages.md` |

## Commands

```shell
mise run up                 # start the Aspire stack (Postgres, backend, front-end, Kubb watch, LibreDB)
mise run down               # stop it
mise run build              # dotnet build (also refreshes the OpenAPI document)
mise run openapi            # refresh only the OpenAPI document (builds the runtime)
mise run generate           # regenerate the TypeScript client from the OpenAPI document
mise run test               # everything: backend + front-end tests
mise run format             # CSharpier + oxfmt
mise run lint               # checks (CI runs the same)
mise run check              # build, front-end build/type-check, lint, format check, tests

dotnet run --project src/backend/packages/<PROJECT>.<Slice>.Tests          # one test project
dotnet ef migrations add <Name> --project src/backend/packages/<PROJECT>.Data
aspire resource app-backend rebuild                                         # rebuild + restart the backend only
aspire logs app-backend -n 50 --search "error"
```

## Working Loop

1. Read the relevant reference(s); open the nearest existing slice.
2. Write or change the handler test first (`<Slice>.Tests`, fake store, data from the entity graph) and watch it fail.
3. Implement: contracts, handler, endpoint (with `WithName`, `WithSummary`, `WithDescription`), mapping; then the store and migration if data changes.
4. `csharpier format`, `dotnet build`, run the slice's tests (and `<PROJECT>.Data.Tests` if a store changed, `<PROJECT>.Api.Tests` for the descriptions and route security, `<PROJECT>.Runtime.Tests` for the routes lint).
5. If the contract changed: confirm the diff in `src/app/src/api/<project>-api.json`, then `mise run generate`.
6. Verify in the running app when it matters: `aspire resource app-backend rebuild`, call the endpoint, read `aspire logs` / `aspire otel traces`.
7. `mise run check` before handing work back.
