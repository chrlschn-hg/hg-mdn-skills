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

Lives in `.config/`, not the root.  Tasks run from the repo root.  `check` is the definition of done (step 14, AGENTS.md, the planning skill).  `npm` ships with `node`; add `pnpm` or `yarn` entries only when chosen.  Add cloud CLIs (`aws-cli`, `azure-cli`, `gcloud`) and point their config at `.config` with an `[env]` block, for example `CLOUDSDK_CONFIG = "{{config_root}}/.config/.gcloud-config"`.  Mise tool names: check `mise registry`.

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

# Front-end build output
node_modules/
dist/

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
- Route security is by group: `Map<PROJECT>Endpoints` maps slices onto a root group that requires a signed-in user (`Health` is the only anonymous route); admin-only slices implement `IAdminEndpoint`; guards are `IEndpointFilter` extensions.  Opening a route means `AllowAnonymous()` and updating the list in `<PROJECT>.Runtime.Tests` (`RouteSecurityLintTests`).  Development signs in a configured user; other environments answer 401 until a real authentication scheme replaces the placeholder (`Add<PROJECT>Authentication`)
- `host`: Aspire host; a new runtime component (database, emulator, worker, etc.) must be plugged into `host/AppHost.cs`

## Skills

- `hg-mdn-backend`: building the .NET backend (feature slices, endpoints, data, telemetry, testing)
- `hg-mdn-frontend`: building the Vue front-end (Nuxt UI, Pinia, API client)
- `hg-mdn-planning`: planning a feature or edit (requirements, draft, spec, and work log in `.agents/plans`)
- `hg-mdn-scaffold`: setting up foundational scaffolding that is missing

## Working agreements

- Plans, specs, and work logs live in `.agents/plans`; throwaway agent output goes in `.agents/scratch` (git-ignored)
- Do not commit or push unless asked.  Commit messages are conventional: `feat(area): Sentence-case description` (also `fix`, `chore`, `refactor`, `docs`, `test`); one dense word for the area
- Before handing work back: `mise run check` passes (build, front-end type check, lint, format check, all tests; database tests need Docker) and the build reports no new warnings (it enforces braces and exhaustive enum switches)

## Tooling

Tools and tasks are managed by Mise (`.config/mise.toml`).

```shell
mise install      # install the tools
mise tasks        # list tasks
mise run up       # start the Aspire stack (detached)
mise run down     # stop it
mise run build    # build the solution (backend)
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
    "ms-dotnettools.csdevkit",
    "ms-dotnettools.csharp",
    "microsoft-aspire.aspire-vscode",
    "k--kato.docomment",
    "aaron-bond.better-comments",
    "csharpier.csharpier-vscode",
    "jeremycaron.csharp-organize-usings",
    "Vue.volar",
    "bradlc.vscode-tailwindcss",
    "oxc.oxc-vscode"
  ]
}
```

### `.vscode/settings.json`

```json
{
  "[csharp]": {
    "editor.defaultFormatter": "csharpier.csharpier-vscode",
    "editor.formatOnSave": true
  },
  "[xml]": {
    "editor.defaultFormatter": "csharpier.csharpier-vscode",
    "editor.formatOnSave": true
  },
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
  "[typescript]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[vue]": { "editor.defaultFormatter": "oxc.oxc-vscode", "editor.formatOnSave": true },
  "[json]": { "editor.defaultFormatter": "oxc.oxc-vscode" },
  "[jsonc]": { "editor.defaultFormatter": "oxc.oxc-vscode" },
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
