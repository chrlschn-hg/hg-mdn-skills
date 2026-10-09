# Devcontainer, Dockerfile, and GitHub Actions (steps 5, 12, 13)

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

- The devcontainer installs tools through Mise (same versions as local development) and adds Docker
  (Aspire containers and Testcontainers need it).  Base image tags:
  <https://github.com/devcontainers/images/tree/main/src/dotnet>.
- The Dockerfile builds the backend API image from the repository root:
  `docker build -f build/Dockerfile -t <project> .`.  It does not bundle the front-end (the baseline
  front-end is cross-origin).  To serve the SPA from the runtime in one container, follow the same-origin
  recipe in the `hg-mdn-frontend` skill, `references/config-deploy.md`: API under `/api` with a JSON 404 for
  unmatched paths, static files and an SPA fallback in `Program.cs`, a Node stage in the Dockerfile, and no
  Kubb `baseURL`.  Also change the prefix in `RouteSecurityLintTests` (see its note).
- Workflows: `ci.yml` (build, test, lint on PRs and `main`) and `docker-publish.yml` (image to GHCR).
  Deployment depends on the cloud platform chosen in step 2: add deploy steps after publish, using OIDC
  federation instead of long-lived credentials.
- Action versions are a snapshot: use the latest major of each action.

## Devcontainer

### `.devcontainer/devcontainer.json`

`postCreateCommand` trusts and installs the Mise config.  Not built in a container yet: test it once (`Dev Containers: Rebuild Container`).

```jsonc
// Format: https://aka.ms/devcontainer.json    Base images: https://github.com/devcontainers/images/tree/main/src/dotnet
// Mise (`.config/mise.toml`) installs the project's tools; the features below provide what Mise cannot (Docker for Aspire and Testcontainers).
{
  "name": "<PROJECT> (.NET)",
  "image": "mcr.microsoft.com/devcontainers/dotnet:10.0-noble",
  "hostRequirements": {
    "cpus": 4,
    "memory": "16gb",
    "storage": "32gb"
  },
  "features": {
    "ghcr.io/devcontainers-extra/features/mise:1": {},
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  // Aspire containers publish ports on the container host; share its network so the dashboard links work.
  "runArgs": ["--network=host"],
  "customizations": {
    "vscode": {
      "extensions": [
        "hverlin.mise-vscode",
        "ms-dotnettools.csdevkit",
        "ms-dotnettools.csharp",
        "microsoft-aspire.aspire-vscode",
        "k--kato.docomment",
        "aaron-bond.better-comments",
        "eamodio.gitlens",
        "csharpier.csharpier-vscode",
        "jeremycaron.csharp-organize-usings",
        "Vue.volar",
        "bradlc.vscode-tailwindcss",
        "oxc.oxc-vscode"
      ]
    }
  },
  "remoteUser": "vscode",
  "postCreateCommand": "mise trust --yes && mise install"
}
```

## Dockerfile

### `build/Dockerfile`

Applies migrations at startup and serves `/health`; pass the connection string with `-e ConnectionStrings__<project>=...`.  The image runs Production: secured routes answer 401 until the real authentication scheme is registered.

```dockerfile
# syntax=docker/dockerfile:1-labs
# Runtime image for the backend API.  Build from the repository root:
#   docker build -f build/Dockerfile -t <project> .
# `--parents` (used below) needs the `dockerfile:1-labs` frontend declared on the first line.

#*----------------------------------------------------------------------------------
# Stage 1: build and publish
#*----------------------------------------------------------------------------------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /srcroot

# Restore first, with only the files restore needs, so this layer is cached until a project
# file or a package version changes.  `.dockerignore` strips tests and build output.
COPY --parents ./src/backend/Directory.* ./
COPY --parents ./src/backend/**/*.csproj ./
RUN dotnet restore ./src/backend/runtime/<PROJECT>.Runtime.csproj

COPY ./src/backend ./src/backend

# Release skips OpenAPI document generation (see `OpenApiGenerateDocuments` in the runtime csproj).
RUN dotnet publish ./src/backend/runtime/<PROJECT>.Runtime.csproj \
  --configuration Release \
  --no-restore \
  --output /app/publish

#*----------------------------------------------------------------------------------
# Stage 2: runtime
#*----------------------------------------------------------------------------------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

COPY --from=build /app/publish .

# The aspnet image listens on 8080 by default and ships a non-root `app` user.
EXPOSE 8080
USER $APP_UID

ENTRYPOINT ["dotnet", "<PROJECT>.Runtime.dll"]
```

## GitHub Actions

### `.github/workflows/ci.yml`

Backend tests use Testcontainers (Docker is preinstalled on `ubuntu-latest`).  `--report-trx` and `--results-directory` are Microsoft Testing Platform options.

```yaml
# Build and test on every pull request and on pushes to main.
#
# Backend integration tests use Testcontainers, which needs Docker; ubuntu runners have it.
# Tools (.NET SDK, Node, ...) come from `.config/mise.toml` so CI matches local development.

name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  # Needed by the test report action to write the check run and PR comment.
  checks: write
  pull-requests: write

env:
  DOTNET_NOLOGO: true
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true

jobs:
  backend:
    name: Backend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      # Installs every tool in `.config/mise.toml` and caches them.
      - uses: jdx/mise-action@v5

      - name: Check formatting
        run: csharpier check src host

      - name: Restore
        run: dotnet restore

      # Build once so compile errors are reported separately from test failures.
      - name: Build
        run: dotnet build --no-restore

      # Microsoft.Testing.Platform (see `global.json`) uses reporters instead of VSTest loggers;
      # `--report-trx` writes the TRX consumed by the report action below.
      - name: Test
        run: dotnet test --solution <project>.slnx --no-build --disable-logo --results-directory TestResults --report-trx

      - name: Upload test results
        if: ${{ always() }}
        uses: actions/upload-artifact@v7
        with:
          name: backend-test-results
          path: TestResults/**/*.trx
          if-no-files-found: warn

      - name: Publish test report
        if: ${{ always() }}
        uses: EnricoMi/publish-unit-test-result-action@v2
        with:
          files: TestResults/**/*.trx

  frontend:
    name: Front-end
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: src/app
    steps:
      - uses: actions/checkout@v7

      - uses: jdx/mise-action@v5

      # Fails if the lockfile is out of date.
      - name: Install
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Check formatting
        run: npm run fmt:check

      # `generated/` is git-ignored: `npm test` and `npm run build` regenerate the API client
      # from the committed OpenAPI document first (`pretest` / `build` scripts).
      - name: Test
        run: npm test

      # Builds and type-checks (vue-tsc).
      - name: Build
        run: npm run build
```

### `.github/workflows/docker-publish.yml`

```yaml
# Build the runtime image and push it to GitHub Container Registry (ghcr.io/<owner>/<repo>)
# on pushes to main and on version tags.  Add deployment steps here (or a downstream workflow)
# for your target platform.

name: Publish image

on:
  push:
    branches: [main]
    tags: ["v*"]

permissions:
  contents: read
  packages: write

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Tags: `latest` on main, the version on `v*` tags, and the commit SHA on every push.
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/build-push-action@v7
        with:
          context: .
          file: build/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
