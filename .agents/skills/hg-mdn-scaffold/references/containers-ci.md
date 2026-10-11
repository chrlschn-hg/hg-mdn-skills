# Devcontainer, Dockerfile, and GitHub Actions (steps 5, 12, 13)

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

- The devcontainer installs tools through Mise (same versions as local development) and adds Docker
  (Aspire containers and Testcontainers need it).  Base image tags:
  <https://github.com/devcontainers/images/tree/main/src/dotnet>.
- The Dockerfile builds **one image for the whole application** from the repository root:
  `docker build -f build/Dockerfile -t <project> .` (`mise run image`).  A Node stage builds the front-end,
  the .NET stage copies its `dist/` into the runtime's `wwwroot` and publishes, and the runtime serves the
  API under `/api` and the front-end at `/` (same origin: no CORS, cookies work).  Serving the front-end
  elsewhere instead (static host, CDN) is the alternative recipe in `hg-mdn-frontend`,
  `references/config-deploy.md`.
- The version and commit the app reports come in as build args (`VERSION`, `GIT_SHA`): `docker-publish.yml`
  passes them, `mise run image` passes the commit, and a build without them reports `0.0.0-dev`
  (`hg-mdn-backend`, `references/build-info.md`, which also explains the `${VERSION:+...}` form).
- Workflows: `ci.yml` (build, test, lint, and an image job that builds the image and smoke-tests it with
  `build/smoke.sh`, on PRs and `main`) and `docker-publish.yml` (image to GHCR).
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

Applies migrations at startup and serves `/api/health` and the front-end; pass the connection string with `-e ConnectionStrings__<project>=...`.  The image runs Production: secured routes answer 401 until the real authentication scheme is registered (the front-end then sends every visitor to its login page).  Stage order is for caching: front-end dependencies and the backend restore are separate layers that change rarely.  Keep the `node` major in step with `.config/mise.toml`.  The Node stage assumes npm (`npm ci`, `package-lock.json`): for pnpm or yarn change the install lines and the lockfile it copies (variants: `frontend-templates.md`).

```dockerfile
# syntax=docker/dockerfile:1-labs
# Runtime image: the .NET API and the built front-end in one container.  Build from the repository
# root:
#   docker build -f build/Dockerfile -t <project> .
# Optional build args stamp the version the app reports on `/api/health` (see `BuildInfo`):
#   --build-arg VERSION=1.2.3 --build-arg GIT_SHA=$(git rev-parse HEAD)
# `--parents` (used below) needs the `dockerfile:1-labs` frontend declared on the first line.

#*----------------------------------------------------------------------------------
# Stage 1: build the front-end
#*----------------------------------------------------------------------------------
# Keep the Node major in step with `node` in `.config/mise.toml`.
FROM node:26 AS web
WORKDIR /srcroot/src/app

# Install first, with only the manifests, so this layer is cached until dependencies change.
COPY src/app/package.json src/app/package-lock.json ./
RUN npm ci

# `npm run build` regenerates the API client from the committed OpenAPI document (no .NET needed),
# bundles, and type-checks.  `.dockerignore` keeps local `node_modules` and `dist` out.
COPY src/app ./
RUN npm run build

#*----------------------------------------------------------------------------------
# Stage 2: build and publish the backend
#*----------------------------------------------------------------------------------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /srcroot

# Restore first, with only the files restore needs, so this layer is cached until a project
# file or a package version changes.  `.dockerignore` strips tests and build output.
COPY --parents ./src/backend/Directory.* ./
COPY --parents ./src/backend/**/*.csproj ./
RUN dotnet restore ./src/backend/runtime/<PROJECT>.Runtime.csproj

COPY ./src/backend ./src/backend

# The built front-end goes into the runtime's web root: `publish` carries `wwwroot` into the output
# and the runtime serves it (`Use<PROJECT>Spa`).
COPY --from=web /srcroot/src/app/dist ./src/backend/runtime/wwwroot

# Release skips OpenAPI document generation (see `OpenApiGenerateDocuments` in the runtime csproj).
#
# Both args are optional.  `${VAR:+...}` passes the property only when the arg is set: an empty
# `-p:Version=` would override the default version with nothing.  The image has no `.git`, so the
# commit comes in as `SourceRevisionId`; the SDK appends it to the informational version.
ARG VERSION
ARG GIT_SHA
RUN dotnet publish ./src/backend/runtime/<PROJECT>.Runtime.csproj \
  --configuration Release \
  --no-restore \
  --output /app/publish \
  ${VERSION:+-p:Version=$VERSION} \
  ${GIT_SHA:+-p:SourceRevisionId=$GIT_SHA}

#*----------------------------------------------------------------------------------
# Stage 3: runtime
#*----------------------------------------------------------------------------------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app

COPY --from=build /app/publish .

# The aspnet image listens on 8080 by default and ships a non-root `app` user.
EXPOSE 8080
USER $APP_UID

ENTRYPOINT ["dotnet", "<PROJECT>.Runtime.dll"]
```

### `build/smoke.sh`

Checks a running container: the API answers under `/api` (and reports the expected commit), the front-end is served, a client-side route falls back to `index.html`, and misses are real 404s (JSON under `/api`, no `index.html` for a missing asset).  CI runs it against the image it just built; run it by hand against any deployment: `build/smoke.sh https://host`.  It assumes this project's shape: the front-end's root element is `id="app"` and `/api/me` is secured (401 in Production); change those two lines otherwise.  Make it executable (`chmod +x`).  Running a container to try it: the README's "Container" section (`references/root-files.md`).

```bash
#!/usr/bin/env bash
# Smoke test for a running container (CI runs it against the image it just built):
#   build/smoke.sh http://localhost:8080 [expected-commit]
# Checks that the API answers under /api, the front-end is served, client-side routes fall back to
# index.html, and misses are real 404s.  With `expected-commit`, also that the build is stamped.
set -euo pipefail

base="${1:?usage: smoke.sh <base-url> [expected-commit]}"
expected_commit="${2:-}"

fail() { echo "FAIL: $*" >&2; exit 1; }

# status <path> -> the HTTP status code
status() { curl -s -o /dev/null -w '%{http_code}' "$base$1"; }

# Wait for the app to start (it applies migrations first).
for _ in $(seq 1 60); do
  [ "$(status /api/health)" = 200 ] && break
  sleep 1
done
[ "$(status /api/health)" = 200 ] || fail "/api/health did not answer 200 within 60 seconds"

health="$(curl -fsS "$base/api/health")"
echo "$health" | grep -q '"status":"Healthy"' || fail "health is not Healthy: $health"
echo "$health" | grep -q '"version":"' || fail "health reports no build version: $health"
if [ -n "$expected_commit" ]; then
  echo "$health" | grep -q "\"commit\":\"$expected_commit\"" || fail "health does not report commit $expected_commit: $health"
fi

curl -fsS "$base/" | grep -q 'id="app"' || fail "/ is not the front-end's index.html"
curl -fsS "$base/notes/123" | grep -q 'id="app"' || fail "a client-side route does not fall back to index.html"

[ "$(status /api/me)" = 401 ] || fail "/api/me should answer 401 until real authentication is configured"
[ "$(status /api/typo)" = 404 ] || fail "an unknown API route should answer 404"
curl -s "$base/api/typo" | grep -q '"not_found"' || fail "an unknown API route should answer a JSON 404, not index.html"
[ "$(status /assets/missing-000000.js)" = 404 ] || fail "a missing asset should answer 404, not index.html"

echo "OK: $health"
```

## GitHub Actions

### `.github/workflows/ci.yml`

Backend tests use Testcontainers (Docker is preinstalled on `ubuntu-latest`).  `--report-trx` and `--results-directory` are Microsoft Testing Platform options.  The `image` job builds the Dockerfile (a broken image is found on the pull request, not at publish time) and runs `build/smoke.sh` against it with a Postgres service; the app applies its migrations on start.

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

      # The build rewrites the OpenAPI document; the front-end client and the Docker image are
      # generated from the committed copy, so a difference is a contract change nobody committed.
      - name: OpenAPI document is committed
        run: git diff --exit-code -- src/app/src/api/<project>-api.json

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

  # Builds the image (API + front-end) and starts it, so a broken Dockerfile or a front-end that
  # is not served is found on the pull request, not when publishing.
  image:
    name: Image
    runs-on: ubuntu-latest
    # The app applies its migrations at startup, so the container needs a database.
    services:
      postgres:
        image: postgres:18
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - uses: actions/checkout@v7

      - uses: docker/setup-buildx-action@v4

      - uses: docker/build-push-action@v7
        with:
          context: .
          file: build/Dockerfile
          load: true
          tags: <project>:ci
          build-args: GIT_SHA=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # `build/smoke.sh` checks /api/health (and that it reports this commit), the front-end, the
      # client-side route fallback, and real 404s.  Host networking reaches the database service.
      - name: Smoke test
        run: |
          docker run -d --name app --network host \
            -e "ConnectionStrings__<project>=Host=localhost;Port=5432;Database=<project>;Username=postgres;Password=postgres" \
            <project>:ci
          build/smoke.sh http://localhost:8080 "$GITHUB_SHA" || { docker logs app; exit 1; }
```

### `.github/workflows/docker-publish.yml`

The `build-info` step turns the ref into build args: the commit always, the version on `v*` tags (`v1.2.3` reports `1.2.3`); other pushes report `0.0.0-dev` plus the commit.

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

      # What the app reports on /api/health: the commit always, the version on `v*` tags (`v1.2.3`
      # reports `1.2.3`); other builds report `0.0.0-dev`.
      - id: build-info
        run: |
          echo "commit=$GITHUB_SHA" >> "$GITHUB_OUTPUT"
          if [[ "$GITHUB_REF" == refs/tags/v* ]]; then echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"; fi

      - uses: docker/build-push-action@v7
        with:
          context: .
          file: build/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            GIT_SHA=${{ steps.build-info.outputs.commit }}
            VERSION=${{ steps.build-info.outputs.version }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
