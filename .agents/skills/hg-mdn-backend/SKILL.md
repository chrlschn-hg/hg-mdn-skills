---
name: hg-mdn-backend
description: Build and debug the modern .NET 11 (C# 15) backend: feature slices, the composition root (`Program.cs`, `Setup/`, DI lifetimes), minimal API endpoints and handlers, route security, unions and `closed` hierarchies instead of exceptions, EF Core data access, configuration, OpenTelemetry/Serilog, the OpenAPI to Kubb contract with Scalar, TUnit tests on Microsoft Testing Platform (not xUnit) with Zeeq Biome test data, zero-warning builds, central package management, and debugging the running app with the Aspire CLI and CSharpRepl.  Use for any work in `src/backend`.
---

# Building the Modern .NET Backend

## Overview

The backend is a .NET 11 (C# 15) web API organized as **feature slices**: one project per capability holding its domain types, storage interface, endpoints, and handlers.  A lean runtime project wires the slices together; a data project implements their storage; an Aspire host runs it all locally.

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
- Read the reference for the work (index below) **before** writing code; copy the shape of the nearest slice
- **Slices hold the business logic**, never the runtime project.  A slice depends on `<PROJECT>.Api`, `<PROJECT>.Telemetry`, and interfaces it defines; never on EF Core, `<PROJECT>.Data`, or another slice's internals (`references/feature-slices.md`)
- **`Program.cs` is a table of contents:** one `Add<PROJECT>X` / `Use<PROJECT>X` / `Map<PROJECT>X` call per concern, each in its own static setup class; no `if`, no inline configuration.  Registrations need no infrastructure and startup work sits behind `StartupSetup`'s guard, because the OpenAPI build runs the whole host without a database (`references/composition-root.md`)
- **Security comes from the group.**  Slices map onto the secured root group (write no `RequireAuthorization()`) or onto `/admin` with `IAdminEndpoint`.  Opening a route needs `AllowAnonymous()`, a `// NOTE:`, and a line in `RouteSecurityLintTests`.  Authentication is one seam, `Add<PROJECT>Authentication` (`references/route-security.md`)
- **One origin:** the API is under `/api` and the runtime serves the built front-end: no CORS, one container image (`references/spa-hosting.md`).  The build stamps the version and commit the app reports (`references/build-info.md`)
- Handlers return `Results<...>` of `TypedResults`; expected failures are `ApiError` results with a distinct `snake_case` code; explicit guard clauses validate (`references/endpoints.md`)
- **Expected outcomes are values, not exceptions.**  Core code returns a **union** (C# 15) or a **`closed` hierarchy**; the handler maps it with an exhaustive `switch`: a `null` arm, no `_`.  Throw only for bugs and infrastructure failures; never `catch` to choose a branch (`references/unions-and-closed-hierarchies.md`)
- Every endpoint sets `WithName`, `WithSummary`, and `WithDescription`; `<PROJECT>.Api.Tests` fails `mise run test` and CI without them.  After a contract change: `dotnet build`, then `mise run generate` (`references/openapi-contract.md`)
- **Contract first when the front-end builds in parallel:** publish the contract before the handlers.  A stub operation has the final records, endpoint, descriptions, and security, and a handler that returns `Stub` as the last arm of its `Results<...>`; never `NotImplementedException`.  Each stub is listed in `StubOperations` (`StubLintTests`); a release fails while one exists.  The switch removes the `Stub` arm and the list entry; the document diff is the removed `x-implementation` (`references/openapi-contract.md`, "Contract first: stub operations"; front-end half: `hg-mdn-frontend`, `references/mocking.md`)
- A model change needs a migration.  **Multi-tenant?  The tenant id is part of every key, foreign key, and index**: decide before the first entity (`references/data-efcore.md`).  Short public ids (NanoID) and reorderable rows (LexoRank): `references/ids-and-ordering.md`
- **Tests are TUnit on Microsoft Testing Platform.  Never xUnit.**  `[Test]`, `await value.Should().BeEqualTo(x)` (always awaited), `[Arguments]`; names `Unit_State_Expected`.  No `[Fact]`, `[Theory]`, `Assert.Equal`, or xUnit packages (`references/testing.md`, `references/tunit.md`)
- Test data comes from **Zeeq Biome**: customize the real entity with `Action<T>`, never `*Prototype` classes.  Handler tests use hand-written fakes; store tests use the real database, never a mocked `DbContext` (`references/zeeq-biome.md`)
- Logging: the per-class static logger with `Log.Here()`; spans: `using var activity = <PROJECT>Telemetry.Trace(...)` (`references/telemetry-logging.md`)
- **Zero warnings: `TreatWarningsAsErrors`.**  Never dismiss a warning by default.  A build error with a rule id is the instruction: fix the cause.  Suppress only case by case, when the rule is wrong *here*: one id, narrowest scope (`#pragma` pair, `.editorconfig` severity, or `NoWarn`), with the reason; never to finish a task, never turn warnings-as-errors off.  A rule that keeps coming up becomes a banned symbol or a custom Roslyn analyzer (`references/static-analysis.md`)
- Mechanical, repeated code is generated: built-in generators first (`[GeneratedRegex]`, `[LoggerMessage]`, `JsonSerializerContext`, Mapperly), then an incremental generator (`references/source-generators.md`)
- **Central package management.**  Versions live only in `Directory.Packages.props` (`src/backend` and `host/` each have one).  Add a package with `dotnet add package <id> --project <csproj>`: it writes the `PackageVersion` entry and a bare `PackageReference`.  Never write an inline `Version` (`NU1008`); `VersionOverride` only with a reason (`hg-mdn-scaffold`, `references/backend-templates.md`)
- Style is enforced by `.editorconfig` and the build: enum switches name every member and end with `_ => throw new UnreachableException(...)`.  `csharpier format <files>` on everything you touch.  Prefer switch expressions, pattern matching, records for contracts; no `Task.Run` in request code.  Comments say *why*; XML-doc every public type and contract parameter (`references/code-style.md`, `references/concurrency.md`)
- Prefer the running app over guessing: read logs and traces with the Aspire CLI, or evaluate code in the process with CSharpRepl (`references/debugging-aspire.md`, `references/csharprepl.md`)
- Definition of done: `mise run check`
</rules>

## Reference Index

| When you are | Read |
|---|---|
| adding a slice (including the contract-first variant with stub handlers) or deciding where code goes | `references/feature-slices.md` |
| wiring: `Program.cs` and `Setup/`, service lifetimes, adding a concern or a slice's services, startup work, a build that fails in OpenAPI generation mode | `references/composition-root.md` |
| writing endpoints, handlers, contracts, error results | `references/endpoints.md` |
| securing routes (groups, guards, roles, anonymous routes, the authentication seam) | `references/route-security.md` |
| writing summaries and descriptions, changing the API the front-end calls, Scalar (`/scalar`) and its options, a missing or stale OpenAPI document; publishing the contract before the handlers exist (stub operations, `StubOperations`, the release gate) | `references/openapi-contract.md` |
| serving the built front-end from the runtime (static files, SPA fallback, the `/api` prefix, cache headers, the container image) | `references/spa-hosting.md` |
| the running version, commit, and build time: stamping them in the build, `BuildInfo`, `/api/health`, logs, the Docker build args | `references/build-info.md` |
| touching entities, stores, EF configuration, or migrations; tenant-scoped (multi-tenant) keys | `references/data-efcore.md` |
| a short, user-facing id (NanoID) next to the UUID, or user-reorderable rows (LexoRank, O(1) moves) | `references/ids-and-ordering.md` |
| adding settings, secrets, or connection strings | `references/configuration.md` |
| adding logs, spans, or metrics | `references/telemetry-logging.md` |
| writing or running tests: test types, names, fakes, store tests, run commands, CI | `references/testing.md` |
| using TUnit features (data sources, hooks, assertions) or translating xUnit habits | `references/tunit.md` |
| creating test data: Zeeq Biome seeds, blueprints, chains, sequences, time, tenancy | `references/zeeq-biome.md` |
| diagnosing the running app (logs, traces, rebuild, database reset) | `references/debugging-aspire.md` |
| evaluating C# inside the running backend: live DI state, scoped services, `#replace`/`#wrap` patches | `references/csharprepl.md` |
| writing C# (switch expressions, pattern matching, types, comments); side-effect-free core: pure decisions, effects passed as delegates | `references/code-style.md` |
| returning outcomes instead of throwing; modeling states, events, and fixed variants: C# 15 unions and `closed` hierarchies | `references/unions-and-closed-hierarchies.md` |
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
2. Write or change the handler test first (`<Slice>.Tests`, fake store, data from a Zeeq Biome chain) and watch it fail.  Against a stub it fails on the `Stub` result.
3. Implement: contracts, handler, endpoint (with `WithName`, `WithSummary`, `WithDescription`), mapping; then the store and migration if data changes.
4. `csharpier format`, `dotnet build`, run the slice's tests (and `<PROJECT>.Data.Tests` if a store changed, `<PROJECT>.Api.Tests` for the descriptions and route security, `<PROJECT>.Runtime.Tests` for the routes lint).
5. If the contract changed: confirm the diff in `src/app/src/api/<project>-api.json`, then `mise run generate`.
6. Verify in the running app when it matters: `aspire resource app-backend rebuild`, call the endpoint, read `aspire logs` / `aspire otel traces`.
7. `mise run check` before handing work back.
