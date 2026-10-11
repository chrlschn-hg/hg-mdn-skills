# Root Files (steps 1 to 4 and 10)

Mise configuration, SDK pinning, package source, ignore files, editor and agent configuration,
`AGENTS.md`, `README.md`.  Sections marked *(optional)* depend on the answers in step 0.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

Symlinks (create with `ln -s`, then commit them; the baseline project has none, and `.agents/skills`
is filled in step 3):

```shell
mkdir -p .agents/skills .claude .codex
ln -s ../.agents/skills .claude/skills          # Claude Code
ln -s ../.agents/skills .codex/skills           # Codex
ln -s AGENTS.md CLAUDE.md                       # Claude Code reads CLAUDE.md
# GitHub Copilot (optional):
mkdir -p .github && ln -s ../AGENTS.md .github/copilot-instructions.md && ln -s ../.agents/skills .github/skills
```

## Mise and .NET

### `.config/mise.toml`

Lives in `.config/`, not the root.  Tasks run from the repo root.  `check` is the definition of done (step 14, AGENTS.md, the planning skill); `image` builds the container (`references/containers-ci.md`).  `npm` ships with `node`; add `pnpm` or `yarn` entries only when chosen.  Add cloud CLIs (`aws-cli`, `azure-cli`, `gcloud`) and point their config at `.config` with an `[env]` block, for example `CLOUDSDK_CONFIG = "{{config_root}}/.config/.gcloud-config"`.  Mise tool names: check `mise registry`.

```toml
# Runtime dependencies (SDKs, CLIs, tools) and tasks for the repository.
#   Install:  mise install        Run a task:  mise run <task>        List tasks:  mise tasks
# Docs: https://mise.jdx.dev/llms.txt
# NOTE: this file lives in `.config/`, not the repo root; tasks run from the repo root.

[tools]
dotnet = "10.0.401"
node = "26"
postgres = "18" # provides `psql` (CLI only; the server runs in Aspire)
"dotnet:Aspire.Cli" = "13.6"
"dotnet:CSharpier" = "1.3.0"
"dotnet:CsharpRepl" = "0.9.2"
"dotnet:dotnet-ef" = "10.0.12"
"github:cesarferreira/stax" = "0.115.0" # stacked PRs: `st` / `stax` (references/stax.md)

[tasks.stax-skill]
description = "Refresh the vendored stax agent skill from the pinned stax version"
run = "mkdir -p .agents/skills/stax && st --skill > .agents/skills/stax/SKILL.md"

[tasks.up]
description = "Start the Aspire stack (detached, non-interactive)"
run = "aspire run --detach --non-interactive"

[tasks.down]
description = "Stop the Aspire stack"
run = "aspire stop"

[tasks.build]
description = "Build the solution (also refreshes the OpenAPI document in src/app/src/api)"
run = "dotnet build <project>.slnx"

[tasks.openapi]
description = "Refresh the OpenAPI document from the C# endpoints (a build of the runtime writes it)"
run = "dotnet build src/backend/runtime"

[tasks.generate]
description = "Regenerate the TypeScript API client from the committed OpenAPI document"
run = "npm run generate --prefix src/app"

[tasks.test]
description = "Run every test (backend via Microsoft Testing Platform, then front-end; `npm test` regenerates the client first)"
run = ["dotnet test --solution <project>.slnx", "npm test --prefix src/app"]

[tasks.image]
description = "Build the container image (API + built front-end), stamped with the current commit (none before the first commit)"
run = "docker build -f build/Dockerfile --build-arg GIT_SHA=$(git rev-parse --verify -q HEAD || true) -t <project> ."

[tasks.format]
description = "Format backend (CSharpier) and front-end (oxfmt)"
run = ["csharpier format src host", "npm run fmt --prefix src/app"]

[tasks.lint]
description = "Lint the front-end (oxlint) and check formatting"
run = ["npm run lint --prefix src/app", "npm run fmt:check --prefix src/app", "csharpier check src host"]

[tasks.check]
description = "Everything that must pass before handing work back (CI runs the same): builds, lint, format check, all tests"
run = [
  "dotnet build <project>.slnx",
  "npm run build --prefix src/app",
  "npm run lint --prefix src/app",
  "npm run fmt:check --prefix src/app",
  "csharpier check src host",
  "dotnet test --solution <project>.slnx",
  "npm test --prefix src/app",
]
```

### `global.json`

Pins the SDK and selects Microsoft Testing Platform for `dotnet test` (required for TUnit).  Match the SDK version to `mise.toml`.

```json
{
  "sdk": {
    "version": "10.0.401",
    "rollForward": "latestFeature"
  },
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

### `.config/dotnet-tools.json`

Mirrors the CSharpier pin so `dotnet tool restore` (CI agents, the VS Code CSharpier extension) works without Mise.  Keep versions in sync with `mise.toml`.

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "csharpier": {
      "version": "1.3.0",
      "commands": ["csharpier"],
      "rollForward": false
    }
  }
}
```

### `nuget.config`

Restricts packages to nuget.org with source mapping (supply-chain hygiene).

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="https://api.nuget.org/v3/index.json" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="https://api.nuget.org/v3/index.json">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

## Ignore files

**`.gitignore`**: run `dotnet new gitignore`, then append the block below.  The stock template ignores any `packages` directory, which would silently swallow `src/backend/packages/*`; the `!src/backend/packages/*` line fixes that.

```gitignore
# --- Project additions (append to the output of `dotnet new gitignore`) ---

# The stock template ignores any `packages` directory (NuGet restore layout); ours holds source.
!src/backend/packages/*

# Kubb output; regenerated from the committed OpenAPI document (`npm run generate`)
generated/

# Front-end build output (the Docker build copies `dist/` into the runtime's `wwwroot`; a local
# copy there, to try the hosted front-end without Docker, is not committed either)
node_modules/
dist/
src/backend/runtime/wwwroot/

# Test results (Microsoft Testing Platform)
.artifacts/
TestResults/

# Mise VS Code extension symlinks
.vscode/mise-tools/

# Agent scratch space and Playwright MCP output (local only)
.agents/scratch/
.playwright-mcp/
```

### `.csharpierignore`

The `Migrations` line belongs to `<PROJECT>.Data`; drop it when there is no database.

```gitignore
# Generated; never hand-formatted.
src/backend/packages/<PROJECT>.Data/Migrations
```

### `.dockerignore`

```gitignore
**/bin/
**/obj/
**/node_modules/
**/*.Tests/
.git/
.config/
host/
src/app/dist/
src/backend/runtime/wwwroot/
```

## AGENTS.md and README.md

### `AGENTS.md`

Fill the title line and the context sentences from the user's answers; the rest already describes the template (tasks, Aspire, route security): adjust it to the step 0 answers (package manager, skills present, agents) and keep it current.

````markdown
# <PROJECT>: (one line: what this application does)

(Two or three sentences: who it is for, what problem it solves, key constraints.)

- `src/backend`: .NET 10, C# 14; use modern language features (primary constructors, collection expressions, pattern matching, switch expressions, extension members, etc.)
    - `packages/`: library projects; one per concern or feature slice (`<PROJECT>.<Feature>` with `<PROJECT>.<Feature>.Tests`)
    - `runtime/`: the web API host; keep it lean, business logic lives in feature packages
- `src/app`: Vue 3, Nuxt UI, Vite, TypeScript, Tailwind CSS; package manager is `npm`
    - `src/app/src/api`: OpenAPI document (written by the backend build) and Kubb output in `generated/`; never edit by hand
    - One origin: the API is under `/api`; in production the runtime serves the built front-end from `wwwroot` (one container, `build/Dockerfile`), in development Vite proxies `/api` to the backend.  No CORS, no base URL in the client
- Route security is by group: `Map<PROJECT>Endpoints("/api")` maps slices onto a root group that requires a signed-in user (`GET /api/health` and the `/api` 404 catch-all are the only anonymous routes); admin-only slices implement `IAdminEndpoint`; guards are `IEndpointFilter` extensions.  Opening a route means `AllowAnonymous()` and updating the list in `<PROJECT>.Runtime.Tests` (`RouteSecurityLintTests`).  Development signs in a configured user; other environments answer 401 until a real authentication scheme replaces the placeholder (`Add<PROJECT>Authentication`)
- The build stamps the running version, commit, and build time (`BuildInfo` in `<PROJECT>.Health`): released images get them from the `VERSION` and `GIT_SHA` build args, local builds report `0.0.0-dev`.  They show on `GET /api/health`, in the startup log, as OpenTelemetry `service.version`, and in the UI (`BuildInfoLabel`)
- `host`: Aspire host; a new runtime component (database, emulator, worker, etc.) must be plugged into `host/AppHost.cs`

## Skills

- `hg-mdn-backend`: building the .NET backend (feature slices, endpoints, data, telemetry, testing)
- `hg-mdn-frontend`: building the Vue front-end (Nuxt UI, Pinia, API client)
- `hg-mdn-planning`: planning a feature or edit (requirements, draft, spec, and work log in `.agents/plans`)
- `stax`: stacked branches and PRs with `st` (vendored from `st --skill`; refresh with `mise run stax-skill`)
- `hg-mdn-scaffold`: setting up foundational scaffolding that is missing

## Working agreements

- Plans, specs, and work logs live in `.agents/plans`; throwaway agent output goes in `.agents/scratch` (git-ignored)
- Do not commit or push unless asked.  Stacked branches and PRs go through `st` (stax: `st create -am`, `st modify -a`, `st submit`; see the `stax` skill); `st ci --watch --strict` watches CI.  Commit messages are conventional: `feat(area): Sentence-case description` (also `fix`, `chore`, `refactor`, `docs`, `test`); one dense word for the area
- Before handing work back: `mise run check` passes (build, front-end type check, lint, format check, all tests; database tests need Docker) and the build reports no new warnings (it enforces braces and exhaustive enum switches)

## Tooling

Tools and tasks are managed by Mise (`.config/mise.toml`).

```shell
mise install      # install the tools
mise tasks        # list tasks
mise run up       # start the Aspire stack (detached)
mise run down     # stop it
mise run build    # build the solution (backend)
mise run image    # build the container image (API + front-end)
mise run check    # everything CI checks: builds, type check, lint, format check, tests
mise run test     # backend + front-end tests
mise run format   # CSharpier + oxfmt
mise run lint     # oxlint, oxfmt --check, csharpier --check
```

### API generation

C# endpoints → OpenAPI document (`dotnet build` writes `src/app/src/api/<project>-api.json`; committed) → Kubb (`npm run generate` in `src/app`) → `src/app/src/api/generated` (git-ignored). Aspire runs `generate:watch` continuously; after a fresh clone run `mise run generate`.

### Aspire CLI

- Host project is in `host/`; use the `aspire-monitoring` and `aspire-orchestration` skills; project specifics (resource names, `--dashboard-url`, several stacks): `hg-mdn-backend`, `references/debugging-aspire.md`

```shell
aspire -h

# What is running (resources, dashboard URL)
aspire ps

# Avoid restarting the full stack; just rebuild the backend
aspire resource app-backend rebuild

# Logs and telemetry without opening the dashboard
aspire logs app-backend -n 50 --search "error"
aspire otel traces app-backend --search "Health"

# Restart the stack if needed
mise run down && mise run up
```

### Playwright MCP

Use the Playwright MCP to run the app in a browser and test the UI.  For UI work, get access first thing **before** you start!

### Formatting

Format what you touch: `csharpier format <file_list_or_dir>` for C#, `npm run fmt` (in `src/app`) for the front-end; `mise run format` does both.
````

### `README.md`

Replace the one-line description.

````markdown
# <PROJECT>

(One paragraph: what this is.)

## Setup

```bash
# Install mise (https://mise.jdx.dev)
curl https://mise.run | sh

# Install the tools (.NET SDK, Node, Aspire CLI, ...)
mise install

# Run the whole stack (Postgres, backend, front-end, code generation, LibreDB)
mise run up

# Stop it
mise run down

# Everything CI checks (builds, type check, lint, format check, tests; tests need Docker)
mise run check
```

Open the Aspire dashboard (URL printed by `aspire run`) for the front-end, API docs (Scalar), and database UI links.

## Container

One image holds the API and the built front-end (`build/Dockerfile`).  The app applies its migrations at startup, so it needs a database; a user-defined network lets the containers find each other on macOS, Windows, and Linux:

```bash
mise run image
docker network create <project>-net
docker run -d --name <project>-pg --network <project>-net -e POSTGRES_PASSWORD=pw postgres:18
docker run -d --name <project>-app --network <project>-net -p 8080:8080 \
  -e "ConnectionStrings__<project>=Host=<project>-pg;Database=<project>;Username=postgres;Password=pw" <project>
build/smoke.sh http://localhost:8080   # the API, the front-end, and the version it reports
docker rm -f <project>-app <project>-pg && docker network rm <project>-net
```

The image runs in Production: every secured API route answers 401 until a real authentication scheme replaces the placeholder (`Add<PROJECT>Authentication`).  Release builds pass `--build-arg VERSION=1.2.3 --build-arg GIT_SHA=<sha>`; the running version and commit show on `/api/health` and in the app.
````

## Code style enforcement

### `.editorconfig`

Scoped to `[*.cs]` so it does not change how CSharpier or oxfmt format other files.  IDE0011 (braces) and IDE0072/IDE0010 (switches name every enum member) are warnings, reported by `dotnet build` because of `EnforceCodeStyleInBuild`.  Add a rule with `dotnet_diagnostic.IDEnnnn.severity = warning`.

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4

# Style rules that the build enforces (`EnforceCodeStyleInBuild` in src/backend/Directory.Build.props).
# Always brace statements, even one-liners.
csharp_prefer_braces = true:warning
dotnet_diagnostic.IDE0011.severity = warning

# `switch` over an enum names every member, so adding a member produces a warning at each switch
# (a catch-all `_` arm does not silence it; keep one that throws for unexpected values).
dotnet_diagnostic.IDE0072.severity = warning
dotnet_diagnostic.IDE0010.severity = warning
```

## VS Code

### `.vscode/extensions.json`

```json
{
  "recommendations": [
    "hverlin.mise-vscode",
    "editorconfig.editorconfig",
    "ms-dotnettools.csdevkit",
    "ms-dotnettools.csharp",
    "microsoft-aspire.aspire-vscode",
    "csharpier.csharpier-vscode",
    "k--kato.docomment",
    "aaron-bond.better-comments",
    "jeremycaron.csharp-organize-usings",
    "Vue.volar",
    "bradlc.vscode-tailwindcss",
    "oxc.oxc-vscode",
    "davidanson.vscode-markdownlint",
    "yzhang.markdown-all-in-one"
  ]
}
```

| Extension | For |
|---|---|
| `hverlin.mise-vscode` | Mise tools, tasks, and environment in the editor |
| `editorconfig.editorconfig` | honors `.editorconfig` for every file type (the C# extension reads it itself) |
| `ms-dotnettools.csdevkit`, `ms-dotnettools.csharp` | C# language support, test explorer, solution explorer |
| `microsoft-aspire.aspire-vscode` | the Aspire host and dashboard |
| `csharpier.csharpier-vscode` | C# and XML formatting; finds the pinned tool in `.config/dotnet-tools.json` |
| `Vue.volar` | Vue, TypeScript in `.vue` files |
| `oxc.oxc-vscode` | `oxlint` diagnostics and fixes, `oxfmt` formatting (needs `oxlint`/`oxfmt` installed in `src/app`: `npm install`) |
| `bradlc.vscode-tailwindcss` | Tailwind class completion |
| `davidanson.vscode-markdownlint` | Markdown lint (warnings only; it never rewrites a file) |
| `yzhang.markdown-all-in-one` | preview, table of contents, list continuation, table formatting **on demand** |

### `.vscode/settings.json`

**Format on save** runs CSharpier for C# and XML (`.csproj`, `.props`, `.slnx`) and `oxfmt` (through the Oxc extension) for TypeScript, Vue, CSS, and JSON, the same tools `mise run format` and CI use.  **Markdown is deliberately excluded**: a formatter pads tables and rewrites list markers, which makes diffs noisy.  Format a Markdown file on demand with the Markdown All in One command (**Format Tables**) if you want to.

```json
{
  "editor.formatOnSaveMode": "file",
  "files.insertFinalNewline": true,
  "files.trimTrailingWhitespace": true,

  "[csharp]": {
    "editor.defaultFormatter": "csharpier.csharpier-vscode",
    "editor.formatOnSave": true
  },
  "[xml]": {
    "editor.defaultFormatter": "csharpier.csharpier-vscode",
    "editor.formatOnSave": true
  },

  "[typescript]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[javascript]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[vue]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[css]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[json]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[jsonc]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "editor.codeActionsOnSave": { "source.fixAll.oxc": "explicit" },

  "[markdown]": {
    "editor.formatOnSave": false,
    "editor.formatOnPaste": false,
    "editor.wordWrap": "on",
    "editor.quickSuggestions": { "comments": "off", "strings": "off", "other": "off" },
    "files.trimTrailingWhitespace": false
  },
  "markdownlint.config": {
    "MD013": false,
    "MD033": false,
    "MD041": false,
    "MD024": { "siblings_only": true }
  },

  "dotnet.defaultSolution": "<project>.slnx",
  "files.associations": {
    "*.css": "tailwindcss",
    "appsettings*.json": "jsonc",
    "*.csproj": "xml",
    "*.props": "xml"
  },
  "editor.quickSuggestions": {
    "strings": "on"
  },
  "tailwindCSS.classAttributes": ["class", "ui"],
  "tailwindCSS.classFunctions": ["defineAppConfig"],
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/bin/**": true,
    "**/obj/**": true,
    "src/app/src/api/generated/**": true,
    "src/app/dist/**": true
  },
  "explorer.fileNesting.enabled": true,
  "explorer.fileNesting.patterns": {
    "AppSettings.cs": "AppSettings.*.cs",
    "appsettings.json": "appsettings.*.json"
  }
}
```

- **Why each Markdown setting:** `files.trimTrailingWhitespace` is off because two trailing spaces are a Markdown line break; word wrap is on because prose lines are long; `MD013` (line length), `MD033` (inline HTML, used in skills), and `MD041` (first line a heading, not true of front matter files) are the lint rules that fight this repository's docs; duplicate headings are fine under different parents.
- **The CLI agrees with the editor:** `.oxfmtrc.json` ignores `**/*.md` (`frontend-templates.md`), so `npm run fmt` and `mise run format` never touch Markdown either.  `oxfmt` otherwise pads tables and rewrites `*` bullets to `-`.
- **CSharpier** finds its version from `.config/dotnet-tools.json`, which mirrors the pin in `mise.toml`: keep the two in sync (it is what makes the extension, `mise run format`, and CI agree).
- `source.fixAll.oxc` applies safe `oxlint` fixes on save; drop it if you prefer fixes by hand (`npm run lint:fix`).
- Run once after cloning: `mise install`, `npm install --prefix src/app`, then **Oxc: Restart oxfmt Server** if the extension was already open.

### `.vscode/tasks.json`

Mise tasks from the command palette (**Tasks: Run Task**):

```json
{
  "version": "2.0.0",
  "tasks": [
    { "label": "up", "type": "shell", "command": "mise run up", "problemMatcher": [] },
    { "label": "down", "type": "shell", "command": "mise run down", "problemMatcher": [] },
    { "label": "check", "type": "shell", "command": "mise run check", "group": { "kind": "build", "isDefault": true }, "problemMatcher": [] },
    { "label": "format", "type": "shell", "command": "mise run format", "problemMatcher": [] }
  ]
}
```

## Agent configuration (optional)

### `.mcp.json`

Aspire MCP (resources, logs, telemetry), Playwright (drive the UI), Nuxt UI docs.  Drop servers the user does not want.

```json
{
  "mcpServers": {
    "aspire": {
      "type": "stdio",
      "command": "aspire",
      "args": ["agent", "mcp"],
      "env": {}
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest", "--extension"],
      "env": {}
    },
    "nuxt-ui-remote": {
      "type": "http",
      "url": "https://ui.nuxt.com/mcp"
    }
  }
}
```

### `.claude/settings.json`

```json
{
  "enabledMcpjsonServers": ["aspire", "playwright", "nuxt-ui-remote"]
}
```

### `.codex/config.toml`

```toml
[mcp_servers.aspire]
command = "aspire"
args = ["agent", "mcp"]

[mcp_servers.nuxt-ui-remote]
enabled = true
url = "https://ui.nuxt.com/mcp"

[mcp_servers.playwright]
command = "npx"
args = ["@playwright/mcp@latest", "--extension"]
```
