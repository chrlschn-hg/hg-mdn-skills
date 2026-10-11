---
name: hg-mdn-scaffold
description: Scaffold a .NET 10 + Vue/Nuxt UI repository ready for development: Mise tasks, Aspire host, feature-slice backend, Kubb-generated API client, tests with real databases, CI, and agent skills.  Use in an empty repository or when a foundation (Mise, Aspire, `src/backend`, `src/app`, CI) is missing.
---

# Modern .NET Scaffold

## Overview

Scaffolds a repository for modern .NET development: Aspire host, feature-slice backend (.NET 10, C# 14), Vue + Nuxt UI front-end, generated API client, tests with real databases, CI, and agent skills for day-to-day work.  Use modern features whenever available (language, tooling, runtime, scaffolding such as `.slnx`).

- Mise (<https://mise.jdx.dev/llms.txt>): runtime dependencies (SDKs, CLIs, tools) and tasks (`mise run <task>`)
- Aspire (<https://aspire.dev/llms.txt>): local orchestration of containers and system dependencies (Postgres, SQL Server, Redis, emulators, runtime commands, package installs)
- Kubb.dev (<https://kubb.dev/llms.txt>): TypeScript client and Zod schemas from the OpenAPI document the backend build writes
- Scalar (<https://scalar.com/products/api-references/integrations/aspnetcore/integration>): API reference UI, Development only
- Testcontainers (<https://dotnet.testcontainers.org/>): integration test infrastructure
- One container image: the runtime serves the built front-end at `/` and the API under `/api`; the build stamps the version and commit the app reports
- TUnit (<https://tunit.dev/llms.txt>) on Microsoft Testing Platform (<https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro>) for tests; **never xUnit**
- CSharpier for C#; `oxlint` and `oxfmt` (<https://oxc.rs/llms.txt>) for the front-end

## When to Use

- The repository is empty or missing key foundational scaffolding
- The user wants scaffolding that was previously skipped

## Instructions

<rules>
- Run independent steps in parallel with sub-agents (order below); coordinate the results and continue
- When logical options exist, ask the user with short option descriptions; the user can defer any decision
- Replace every version in the templates with the latest stable at scaffold time (`mise latest <tool>`, `npm view <pkg> version`, `dotnet package search`); versions here and in `references/` are a snapshot
- Without permissions, request them or give the user the steps one by one
- Before a step that needs the user (install, login, credentials): list its sub-steps, ask if ready, then guide one at a time
- Skip steps already done or not requested
- For setup not documented here, follow the technology's docs (look for `llms.txt`)
- XML sections in this document are blocks that appear elsewhere in it
- Full file templates live in `references/`; each step names the file.  Create the files from the templates, replace the placeholders, adjust to the user's answers.  Do not invent a different layout
- Ports: pin only the backend port; Aspire assigns the front-end and LibreDB ports (`aspire describe`).  The AppHost's dashboard, OTLP, and resource-service ports are fixed in `host/Properties/launchSettings.json`: change them if they collide (`references/aspire-host.md`)
- Do not commit or push unless the user asks
- Leave terse, direct comments in the artifacts: the reasoning, where related files are, links
</rules>

Placeholders used in the templates:

- `<PROJECT>`: PascalCase project name for projects and namespaces (`Acme`)
- `<project>`: kebab-case project name for the solution file, database, image (`acme`)
- `<BACKEND_API_PORT>`: a free local port for the backend (`lsof -nP -iTCP:<port> -sTCP:LISTEN`); set in `launchSettings.json` and repeated in `vite.config.ts` (the `/api` proxy fallback): keep them equal
- `<GUID>`: a fresh GUID for each `UserSecretsId` (`uuidgen`)
- `<FEATURE_SLICE>`: name of a feature slice (`Health`, `Orders`, ...)
- `<DATABASE_PROVIDER>`: the Testcontainers module of the chosen database (`PostgreSql`, `MsSql`, ...)

Order: 1, then 2, then 3 and 4; then 5, the chain 6 → 9 → 10, and 7 in parallel; 8 needs 6 and 7; 11 and 12 need 6 to 9; 13 needs 11 and 12; 14 last.

0. Establish the context.  Ask once, up front (the user can defer any row):

   | Question | Default | Used in |
   |---|---|---|
   | What is being built (one line plus two sentences of context)? Project name (derive `<PROJECT>`, `<project>`; confirm) | none | 3 and every template |
   | Agents: Claude Code (always wired), Codex, GitHub Copilot | Claude Code | 3 |
   | .NET SDK | latest LTS (10.x) | 2, 6 |
   | Front-end package manager (`npm`, `pnpm`, `yarn`) | `npm` | 2, 7, 9 |
   | Database (Postgres, SQL Server, Redis, `none`) | Postgres | 2, 6, 9 |
   | Cloud CLIs (`aws-cli`, `gcloud`, `azure-cli`), other tools (`python`, `go`) | none | 2 |
   | `.mcp.json` (Aspire, Playwright, Nuxt UI) | yes | 3 |
   | Devcontainer or Codespaces (step 5) | yes | 5 |
   | How people sign in (cookie, JWT bearer, OpenID Connect) | later (placeholder) | 6 |
   | Tenancy: single-tenant, or multi-tenant (the tenant id is part of every key; `hg-mdn-backend` `references/data-efcore.md`) | single-tenant | 6 |
   | CSharpRepl hook (step 10), LibreDB | yes | 9, 10 |
   | Where the front-end is served: by the runtime in one container, or apart from the API (static host, CDN) | by the runtime | 6, 7, 12 |
   | GitHub Actions (step 13) and a deployment target | CI yes, deploy later | 13 |

1. Install Mise (<https://mise.jdx.dev/llms.txt> has the steps for the user's platform)
  - Install if missing; ensure `mise` is on `PATH` and the shell is activated
  - Recommend the VS Code extension `hverlin.mise-vscode`
  - `mise.toml` goes in `.config/mise.toml`, not the root
2. Set up Mise tooling (template: `references/root-files.md`, "Mise and .NET")
  - SDK: latest LTS (10.x) unless the user chose otherwise; pin it in `.config/mise.toml` and `global.json` (which also selects Microsoft Testing Platform).  Another SDK also changes every `net10.0` and `10.0` (`Directory.Build.props`, `host/host.csproj`, Dockerfile and devcontainer images, the `10.0.x` package versions, `dotnet-ef`, `AGENTS.md`)
  - Package manager: `npm` ships with `node`; `pnpm` and `yarn` get their own entries (variants: `references/frontend-templates.md`)
  - Database CLI: `postgres` (`psql`), `github:microsoft/go-sqlcmd` (`sqlcmd`), `redis` (`redis-cli`); check `mise registry`; swapping providers: `references/data-templates.md`
  - .NET tools: always <mise_config_default_dotnet_tools />
  - Stacked PRs: `github:cesarferreira/stax` (pinned; installs `st` and `stax`).  Then, per machine: `st setup --yes` and `st doctor` (setup, usage, and CI watching: `references/stax.md`)
  - Cloud CLIs (`aws-cli`, `gcloud`, `azure-cli`) with their config under `.config` (see the template); other tools asked for
  - Tasks `up`, `down`, `build`, `openapi`, `generate`, `test`, `image`, `format`, `lint`, `check` from the template; `mise tasks` lists them.  `mise run check` is the definition of done
  - Verify: `mise install`, `dotnet --version`, `mise tasks`
3. Set up skills, `AGENTS.md`, `README.md`, and editor config (templates: `references/root-files.md`)
  - VS Code (`.vscode/extensions.json`, `settings.json`, `tasks.json`): format on save with CSharpier (C#, XML) and `oxfmt` through the Oxc extension (TypeScript, Vue, CSS, JSON); `source.fixAll.oxc` for lint fixes; Markdown tools (markdownlint, Markdown All in One) but **no format on save for Markdown** (formatters pad tables and churn diffs; `.oxfmtrc.json` ignores `**/*.md` so the CLI agrees)
  - Vendor the stax agent skill: `mise run stax-skill` (writes `.agents/skills/stax/SKILL.md` from `st --skill`, matching the pinned version); list it in `AGENTS.md` (`references/stax.md`)
  - `.agents/skills` is the root.  Copy `hg-mdn-backend`, `hg-mdn-frontend`, and `hg-mdn-planning` into it (and `hg-mdn-scaffold` if the user wants it)
  - Symlinks, committed with the project (the baseline has none): `.claude/skills` -> `.agents/skills` and `./CLAUDE.md` -> `./AGENTS.md` for Claude Code, always; `.codex/skills` (Codex), `.github/skills` and `.github/copilot-instructions.md` (Copilot) only for the agents named in step 0
  - `./AGENTS.md` and `README.md` if absent: fill the title and context from the step 0 answers and adjust the rest of the template (package manager, skills actually present, agents); keep "Working agreements" (plans in `.agents/plans`, conventional commits, no commits unless asked, `mise run check`)
  - `.mcp.json` and the matching agent config (`.claude/settings.json`, `.codex/config.toml`) if wanted
  - `.vscode/extensions.json` and `settings.json` from the same template
4. Set up ignore files and style rules (template: `references/root-files.md`)
  - `dotnet new gitignore`, then append the project additions (they re-include `src/backend/packages/*` and ignore `generated/`, build, test, and scratch output); add entries for the tools chosen in step 2
  - `.csharpierignore`, `.dockerignore`, `nuget.config` (source mapping), and `.editorconfig` (C# style rules the build enforces)
5. Set up the devcontainer (optional; template: `references/containers-ci.md`)
  - Start from the `dotnet` devcontainer images (<https://raw.githubusercontent.com/devcontainers/images/refs/heads/main/src/dotnet/README.md>); add features from <https://containers.dev/features>; a custom `Dockerfile` only if needed
  - Mise installs the project tools inside the container; features add what Mise cannot (Docker for Aspire and Testcontainers, `gh`)
  - Skip if the user uses neither devcontainers nor Codespaces
6. Set up the .NET web API (templates: `references/backend-templates.md`, `references/data-templates.md`)
  - Create `src/app/src/api/` first (the template's `README.txt`): the build writes the OpenAPI document there
  - Central package management in `src/backend/Directory.Packages.props` (the package set; drop what the chosen database does not need); shared properties in `src/backend/Directory.Build.props`
  - Layout under `src/backend`, every project in `<project>.slnx` (folders mirror directories):
    - `runtime/<PROJECT>.Runtime.csproj`: the web host; keep it lean, business logic goes in feature packages; one `ProjectReference` per slice; writes OpenAPI (path relative to the csproj directory) and serves Scalar in Development
    - `packages/<PROJECT>.Config` (`AppSettings.cs`: the strongly typed `public record class` settings), `.Api` (endpoint contracts, discovery, `Security/`, `SpaSetup`), `.Telemetry`
    - `packages/<PROJECT>.Health` and `.Health.Tests`: the first slice, the worked example for every later `<PROJECT>.<FEATURE_SLICE>`; `.Identity` and `.Identity.Tests`: `GET /me`
    - `packages/<PROJECT>.Api.Tests` (lints the OpenAPI document; tests route groups, guards, and front-end hosting in memory) and `.Runtime.Tests` (lints every real route: secured or listed as anonymous; references the runtime)
    - Database only: `.Data`, `.Data.Tests`, `.Testing` (fixtures, `TruncateToPostgresPrecision`), `.Testing.Tests`
  - Secure by default (rules: `hg-mdn-backend`, `references/route-security.md`): `Health` is the only slice route that is `AllowAnonymous()` (plus the `/api` 404 catch-all `HttpSetup` adds)
  - One origin: the API is mapped under `/api` (`Map<PROJECT>Endpoints("/api")`), no CORS, and `Api/SpaSetup.cs` serves the built front-end from `wwwroot` when it is there (`hg-mdn-backend`, `references/spa-hosting.md`).  If the user chose to host the front-end apart, follow the alternative recipe in `hg-mdn-frontend`, `references/config-deploy.md`
  - Build info: `Health` reports the running version, commit, and build time (`BuildInfo`), also logged at startup and sent as `service.version` (`hg-mdn-backend`, `references/build-info.md`)
  - Sign-in is the step 0 answer.  Until a scheme is chosen keep the placeholder `Add<PROJECT>Authentication` (Development signs in a configured user; elsewhere secured routes answer 401); replacing it changes nothing else
  - Every endpoint gets `WithName`, `WithSummary`, and `WithDescription`; `<PROJECT>.Api.Tests` lints the OpenAPI document, so a missing one fails `mise run test` and CI, not `dotnet build` (`hg-mdn-backend`, `references/openapi-contract.md`)
  - `Program.cs` carries `HttpExceptionHandler` and the `isOpenApiGeneration` guard (the build runs it up to `app.Run()`; skip infrastructure in that mode)
  - No migration ships: the first entity needs `dotnet ef migrations add` (`hg-mdn-backend`, `references/data-efcore.md`), and the entity-graph test builder arrives with it (`references/entity-graph.md`)
  - Zero tolerance: `TreatWarningsAsErrors` in `Directory.Build.props`; raising `AnalysisLevel`, banned APIs, and custom analyzers are per-project choices (`hg-mdn-backend`, `references/static-analysis.md`)
  - Verify: `dotnet build` reports 0 warnings and writes `src/app/src/api/<project>-api.json`; `dotnet test --solution <project>.slnx` passes (database tests need Docker)
7. Set up the front-end in `src/app` (templates: `references/frontend-templates.md`)
  - Vite, Vue 3 (Composition API), Nuxt UI (<https://ui.nuxt.com/llms.txt>) through its Vite plugin, TypeScript 6.x only (`vue-tsc` does not support 7; `hg-mdn-frontend`, `references/project-structure.md`)
  - Pinia, VueUse, file-based routing (`vue-router` v5, `src/pages`); the unplugin auto-import and component plugins are wired through the Nuxt UI plugin's `autoImport` and `components` options
  - **Toolchain for speed and consistent formatting, from the first commit:** TypeScript 6.x (`~6.0`; `vue-tsc` does not support 7), `oxlint` for linting and `oxfmt` for formatting (the Rust-based Oxc tools, <https://oxc.rs/llms.txt>), with `.oxlintrc.json`, `.oxfmtrc.json`, the `lint`/`lint:fix`/`fmt`/`fmt:check` scripts, and VS Code formatting on save through `oxc.oxc-vscode`.  Do not add ESLint or Prettier: one linter and one formatter, fast enough to run on every save and in every check (a lint or format check of the template takes about a tenth of a second)
  - Run `npm run fmt` once after creating the files, so the first commit is already formatted, then verify `npm run lint && npm run fmt:check`
  - Icons are bundled, never fetched at runtime: the template's `vite.config.ts` maps the lucide slots to hugeicons (`hg-mdn-frontend`, `references/nuxt-ui.md`)
  - Vitest with `happy-dom` and `@vue/test-utils`; infrastructure in `src/test/` (`setup.ts` fails any unmocked `fetch`, `mount.ts`, `AppHarness.vue`); `App.test.ts` smoke-tests the whole app, including "no icon fetched"
  - Secure by default in the router too (`router/guards.ts`, `stores/session-store.ts`, `pages/login.vue`; rules: `hg-mdn-frontend`, `references/routing.md`); the login page is a placeholder for the chosen sign-in flow
  - `zod`, `AsyncContent.vue`, `src/api/errors.ts` (`SaveResult`), `src/api/setup.ts` (`configureApiClient`), `src/env.d.ts` (`VITE_API_BASE_URL`, only for a front-end hosted apart), `components/BuildInfoLabel.vue` (shows the running version)
  - Kubb (`kubb.config.ts`): TypeScript types, fetch client, Zod (`@kubb/plugin-zod`); no `baseURL` (the OpenAPI paths start with `/api`, so calls are relative); `vite.config.ts` proxies `/api` to the backend in development.  Scripts: `generate`, `generate:watch` (Aspire runs it standalone), and `pretest` and `build` run `kubb generate` first
  - The official Nuxt UI agent skill, after step 3's symlinks exist: `npx skills add nuxt/ui --skill nuxt-ui -a codex -y` (installs into `.agents/skills/nuxt-ui`, records `skills-lock.json`; tell the user to review it)
8. Generate the client and verify the front-end (needs steps 6 and 7)
  - In `src/app`: install with the chosen package manager; step 6's build already wrote the OpenAPI document; `npm run generate`
  - Verify: `npm run lint && npm run fmt:check && npm test && npm run build`
  - `AGENTS.md` ("API generation") already states the flow C# endpoints -> OpenAPI document -> Kubb; confirm it matches
9. Set up Aspire with C# (templates: `references/aspire-host.md`)
  - `host/` holds the AppHost; add it to the `.slnx`
  - Resources, whose names are a contract for the `aspire` commands and the skills: `postgres` (with its database), `app-backend`, `app-frontend` (Vite; match `WithNpm()` to the package manager; `.WithReference(backend)` gives its `/api` proxy the backend URL), `kubb-generate-watch`, `libredb` (database UI with a seeded connection; comment how to log in)
  - Aspire CLI: `aspire --version` in the same version line as the host SDK
  - Agent skills: `aspire agent init --skill-locations standard --skills aspire-orchestration,aspire-monitoring` (flags per `aspire agent init --help`; if a name is rejected, choose the orchestration and monitoring skills in the prompt)
  - `AGENTS.md` already carries the core Aspire commands and the rule that new runtime components are plugged into `host/AppHost.cs`; confirm
10. Set up the CSharpRepl hook (optional; skip: delete `src/backend/runtime/DebugRepl.cs` and the CSharpRepl block of `host/AppHost.cs`)
  - `host/ProcessUtils.cs` supplies the startup-hook environment to the backend in run mode only (`references/aspire-host.md`); `DebugRepl.cs` (bare `<PROJECT>` namespace) gives the REPL scope helpers for scoped services such as `DbContext`
  - Confirm: with the stack up, `csharprepl connect list` shows `<PROJECT>.Runtime`; usage: `hg-mdn-backend`, `references/csharprepl.md`
11. Set up CSharpier
  - `.config/dotnet-tools.json` mirrors the CSharpier version in `mise.toml` (for CI agents and the VS Code extension)
  - `mise run format` and `mise run lint` cover C# and the front-end; CSharpier also formats `.csproj`, `.props`, and `.slnx`; VS Code formats on save except Markdown (`.vscode` template)
  - `csharpier format src host` once before finishing; verify: `csharpier check src host` (CI runs it)
12. Set up the Dockerfile (template: `references/containers-ci.md`)
  - `build/Dockerfile` builds one image from the repo root: a Node stage builds the front-end, the .NET stage copies `dist/` into the runtime's `wwwroot` and publishes; restore and `npm ci` are separate layers for caching.  `VERSION` and `GIT_SHA` build args stamp the version the app reports (`mise run image` passes the commit)
  - `build/smoke.sh` checks a running container: `/api/health` (and its commit), the front-end, the client-side route fallback, real 404s.  `chmod +x` it
  - Verify: `mise run image`, run it against a throwaway Postgres on a user-defined network (the README template's "Container" section), then `build/smoke.sh http://localhost:8080`
  - The front-end hosted apart instead: drop the Node stage and the `wwwroot` copy, and trim the CI `image` job's smoke test and `SpaHostingTests` (alternative recipe: `hg-mdn-frontend`, `references/config-deploy.md`)
  - See <gotchas /> for what the image does at runtime
13. Set up GitHub Actions (optional; template: `references/containers-ci.md`)
  - `ci.yml`: backend format check, build, tests (Testcontainers needs Docker); front-end lint, format check, test, build; an image job that builds the Dockerfile and runs `build/smoke.sh` against it
  - `docker-publish.yml`: image to GHCR on `main` and version tags; `v1.2.3` reports version `1.2.3` (`VERSION` and `GIT_SHA` build args)
  - Deployment target (step 0): add deploy steps with OIDC federation, not long-lived secrets
14. Verify, then hand off.  Done when all hold:
  - `mise run check` passes: builds with 0 warnings, front-end type check, lint, format check, all tests (database tests need Docker; see <gotchas />)
  - After `mise run up`, `aspire describe` shows `postgres`, `app-backend`, `app-frontend`, `kubb-generate-watch`, and `libredb` running; `curl http://localhost:<BACKEND_API_PORT>/api/health` returns `Healthy` with the build (`0.0.0-dev`, and the commit once the repository has one); the front-end URL from `aspire describe` shows the health card; `csharprepl connect list` shows `<PROJECT>.Runtime`; LibreDB lists the seeded connection
  - No leftover placeholders: `grep -rnE '<(PROJECT|project|GUID|BACKEND_API_PORT|FEATURE_SLICE|DATABASE_PROVIDER)>|\(one line:' . --exclude-dir={node_modules,.git,.agents,bin,obj}` prints nothing
  - `mise run down`; remove leftover persistent containers and volumes if the user wants a clean slate (`ContainerLifetime.Persistent` outlives `aspire stop`)
  - Report: what was created, which questions were deferred, how to start (`mise run up`), and next steps (real authentication scheme, first migration, deployment)

## Snippets and Samples

<mise_config_default_dotnet_tools>

`dotnet:Aspire.Cli` (keep it on the host SDK's version line), `dotnet:CSharpier` (the same version as `.config/dotnet-tools.json`), `dotnet:CsharpRepl`, `dotnet:dotnet-ef` (the SDK's runtime line).  Versions: `references/root-files.md`, then replace with the latest stable.

</mise_config_default_dotnet_tools>

<gotchas>

- **Production answers 401.** The Docker image runs Production, where the placeholder authentication authenticates nobody: every secured route answers 401 (only `/api/health` works) until a project registers its real scheme.
- **The front-end is in the image.** `docker build` runs `npm run build` (it regenerates the API client from the committed OpenAPI document, so commit `src/app/src/api/<project>-api.json` after backend changes).  A local `wwwroot` (to try the hosted front-end without Docker) is git-ignored and not sent to the Docker build.
- **`Zero tests ran`** from `dotnet test`: the Microsoft Testing Platform pipe path is too long; see `hg-mdn-backend`, `references/testing.md`.

</gotchas>
