# Configuration, Build, and Deployment

How the front-end is configured, built, and shipped.  **One origin by default:** the API lives under `/api`, the .NET runtime serves the built front-end at `/` (one container), and in development Vite proxies `/api` to the backend.  The browser never calls across origins, so there is no CORS, cookie authentication just works, and the generated client has no base URL.  Hosting the front-end apart from the API is the alternative (below).

| Shape | API calls | Hosting | Use when |
|---|---|---|---|
| **One origin, one container** (the scaffold default) | relative `/api/...` | the runtime serves the API under `/api` and the built SPA from `wwwroot` | cookie auth, one deployable, simplest operations |
| **Hosted apart** | an absolute URL (`VITE_API_BASE_URL`); CORS on the backend | a static host or CDN for the SPA; the API elsewhere | a public or token-based API, or a separate team deploys the SPA |

## Environment variables

Vite reads `.env`, `.env.<mode>`, and `.env.local` files and the process environment.  Only variables prefixed **`VITE_`** reach the browser bundle, as `import.meta.env.VITE_X`, replaced **at build time**.

`src/app/src/env.d.ts`

```ts
/// <reference types="vite/client" />

// Typed `import.meta.env`.  Only `VITE_`-prefixed variables reach the browser bundle; anything
// else stays on the build machine.  They are public: never put a secret in one.
interface ImportMetaEnv {
  /** Backend origin for the generated client; unset (the default) means same-origin `/api`. */
  readonly VITE_API_BASE_URL?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

- **They are public.**  Anything in the bundle can be read by every visitor: never put a secret, token, or private URL in a `VITE_` variable.
- **They are baked in at build.**  Changing one means rebuilding the front-end.  There is no runtime configuration of a static bundle; if values must differ per deployment without a rebuild, serve them from the backend (`GET /api/config`) and fetch at startup.
- Declare each variable the app code reads in `env.d.ts` so `import.meta.env.VITE_X` is typed (and `undefined` is visible in the type).  `VITE_BASE` is the exception: `vite.config.ts` reads it from `process.env` (shell or CI only, not `.env`; see "Sub-path hosting").
- Built-ins: `import.meta.env.DEV`, `.PROD`, `.MODE`, `.BASE_URL`.
- `.env` and `*.local` are git-ignored (the root `.gitignore`, `src/app/.gitignore`): personal overrides.  Commit `.env.production` only if it holds non-secret defaults.

## One origin, one container

```text
production:   browser -> runtime  (/api/* -> endpoints;  /, /assets/*, /notes/123 -> wwwroot)
development:  browser -> Vite     (/api/* -> proxy -> backend;  everything else -> Vite)
```

**Front-end.**  Nothing to configure: Kubb turns the OpenAPI paths (`/api/health`) into relative calls (`kubb.config.ts` has no `baseURL`), and `vite.config.ts` proxies them in development.  Aspire passes the backend URL (`services__app-backend__http__0`) because the front-end resource has `.WithReference(backend)` in `host/AppHost.cs`; the fallback, for Vite alone, is the backend's port from `launchSettings.json`, repeated in `vite.config.ts` (keep them equal).  Only `/api` is proxied: anything else the backend serves on its own (`/scalar`, `/openapi/v1.json`, an OpenID Connect callback) is opened on the backend's URL, or its path is added to `server.proxy`.

`src/app/vite.config.ts (excerpt)`

```ts
// The backend the `/api` proxy forwards to.  Aspire injects the URL when the front-end resource has
// `.WithReference(backend)` (`host/AppHost.cs`); the fallback is `applicationUrl` in the backend's
// `launchSettings.json`, for running Vite alone.
const apiTarget = process.env["services__app-backend__http__0"] ?? "http://localhost:<BACKEND_API_PORT>";

export default defineConfig({
  // ...
  // The browser only ever talks to Vite; `/api` is forwarded to the backend, so development is
  // same-origin like production (no CORS, cookies behave).  `vite preview` reuses this proxy.
  server: { proxy: { "/api": apiTarget } },
  // ...
});
```

**Backend.**  `Map<PROJECT>Endpoints("/api")`, `Use<PROJECT>Spa()` before the endpoints, `Map<PROJECT>SpaFallback()` after them.  The rules (the `nonfile` constraint, order, cache headers, tests) are in `hg-mdn-backend`, `references/spa-hosting.md`.

**Container.**  A Node stage builds `dist/`, the .NET stage copies it into the runtime's `wwwroot`, and `publish` carries it into the image.  Full file: `hg-mdn-scaffold`, `references/containers-ci.md`.

`build/Dockerfile (excerpt)`

```dockerfile
FROM node:26 AS web
WORKDIR /srcroot/src/app

# Install first, with only the manifests, so this layer is cached until dependencies change.
COPY src/app/package.json src/app/package-lock.json ./
RUN npm ci

# `npm run build` regenerates the API client from the committed OpenAPI document (no .NET needed),
# bundles, and type-checks.  `.dockerignore` keeps local `node_modules` and `dist` out.
COPY src/app ./
RUN npm run build
```

```dockerfile
# The built front-end goes into the runtime's web root: `publish` carries `wwwroot` into the output
# and the runtime serves it (`Use<PROJECT>Spa`).
COPY --from=web /srcroot/src/app/dist ./src/backend/runtime/wwwroot
```

**An out-of-date `src/app/src/api/<project>-api.json` ships an out-of-date client**: the image generates the client from the committed document.  `mise run openapi` (any backend build) rewrites it; commit it with the backend change.  CI's backend job fails when the committed copy differs from what the build writes.  Keep the `node` major in the Dockerfile in step with `.config/mise.toml`.

To try the hosted front-end without Docker, see `hg-mdn-backend`, `references/spa-hosting.md`, "Rules".

Gotchas:

- Without the `/api` prefix a page route like `/notes` and an API route `/notes` collide (the API wins and the page never loads).
- An unknown path under `/api` is a JSON 404, never `index.html`: a typo must not look like a successful response.
- After a deploy, a tab still running the old bundle asks for chunks that no longer exist and fails with `Failed to fetch dynamically imported module`.  The runtime answers those with a real 404 (not `index.html`, which the browser would try to parse as a script); a reload fixes it.  To do it automatically: `window.addEventListener("vite:preloadError", () => location.reload())`.
- **Authentication must be real before this ships.**  Outside Development the template's placeholder authentication answers 401 on every secured API route (only `/api/health` is public), and the SPA's router sends every visitor to the login page.  Register the real scheme in the backend (`hg-mdn-backend`, `references/route-security.md`) and replace the placeholder login page.

## Hosting apart from the API (the alternative)

Use it when the SPA is deployed on its own (a static host or CDN).  The API keeps its `/api` prefix; what changes:

1. **Build with the API's origin:** `VITE_API_BASE_URL=https://api.example.com npm run build` (the calls become `https://api.example.com/api/...`).  `configureApiClient` applies it; `errors.test.ts` pins the behavior with `vi.stubEnv`.  An empty value means same-origin.
2. **Backend:** register CORS for the SPA's origin (that origin, not any origin) before the security middleware.  Cookie auth also needs `AllowCredentials()` and `credentials: "include"` in `configureApiClient`; on a sibling subdomain (`app.x.com` calling `api.x.com`) the cookies are same-site, otherwise `SameSite=None; Secure`, which browsers' third-party-cookie blocking makes fragile: prefer the same site, or bearer tokens.  The scaffold registers none of this: add it.
3. **Image:** drop the Node stage and the `wwwroot` copy; the image is the API alone (`Use<PROJECT>Spa()` and `Map<PROJECT>SpaFallback()` then do nothing).  Trim what assumes the front-end in the image: the CI `image` job's smoke test (`build/smoke.sh` checks `/` and `/notes/123`) and `SpaHostingTests`.
4. **Static host:** answer every path with `index.html` (the router handles it) and apply the cache rules below.

## Sub-path hosting

Served under `/web/` rather than `/`?  Build with `VITE_BASE=/web/` and the router follows (`createWebHistory(import.meta.env.BASE_URL)`).  Set it in the shell or CI environment: `vite.config.ts` reads `process.env`, and Vite does not load `.env` files before evaluating the config, so a `VITE_BASE` in `.env.production` is ignored.  The single container has no sub-path support (untested): it needs a `VITE_BASE` build arg in the Dockerfile's web stage and the backend's side (`hg-mdn-backend`, `references/spa-hosting.md`, "Rules").  API calls keep their absolute `/api/...` paths, which `base` does not rewrite; a proxy that mounts the container under `/web` also needs `VITE_API_BASE_URL=/web`.

## Build

`npm run build` = `kubb generate` -> `vite build` -> `vue-tsc -b`.  It needs no .NET SDK or running backend: the OpenAPI document is committed, and Kubb generates from it.  Output is `dist/`: `index.html` plus hashed files in `assets/` (cache them forever; never cache `index.html`).  `npm run preview` serves `dist/` locally with the same `/api` proxy (to the fallback backend URL: Aspire's variable is not set there).

Keep the bundle honest: icons are bundled (`nuxt-ui.md`), routes are code-split per page, and a heavy dependency (a chart or editor library) should be imported in the page that needs it, not in `main.ts`.

## Showing the version in the UI

The backend reports the running build on `GET /api/health` (`version`, `commit`, `builtAtUtc`), stamped by the build (`hg-mdn-backend`, `references/build-info.md`).  `BuildInfoLabel.vue` takes it as a `build` prop and shows `Version 1.4.0`, the short commit, and the build time when the build has them (a local build is `0.0.0-dev` plus the commit).  The scaffold's home page renders it under the health status.

Put it where people look when reporting a problem: the sidebar footer, an about dialog, or the user menu.  `health-store` loads only when a page asks, so a label in the app shell needs the shell to call `refresh()` itself.  It shows the backend that answered; a tab left open across a deploy still runs the old bundle until it reloads (`index.html` is `no-cache`, so the next load gets the new one).

## Cache and security headers

- `index.html`: `Cache-Control: no-cache`; `assets/*`: `public, max-age=31536000, immutable` (the names are content-hashed).  The runtime sets exactly this; a CDN or static host needs the same rules.
- Set a Content-Security-Policy that allows only your own origin plus the API; because icons are bundled, no Iconify host is needed, and because fonts/images are local, none is allowed beyond `'self'`.
- Terminate TLS in front of the container; mark cookies `Secure` and `HttpOnly`.

## Before shipping

1. The hand-off checklist (`recipes.md`).
2. Build the image and run `build/smoke.sh` against it (`mise run image`; CI does the same on every pull request): API, front-end, deep link, JSON 404 under `/api`, real 404 for a missing asset, and the commit the app reports.
3. Open the running container and click through; check the network panel: no requests to third-party hosts, no failed calls.
4. Deep link and refresh on a nested route.
5. The `VITE_*` values in the bundle are the ones you meant (search `dist/assets/*.js`); no secrets in `dist/`.
