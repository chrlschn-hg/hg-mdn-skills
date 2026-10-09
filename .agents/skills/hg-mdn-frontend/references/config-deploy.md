# Configuration, Build, and Deployment

How the front-end is configured, built, and shipped.  Two deployment shapes work:

| Shape | API calls | Hosting | Use when |
|---|---|---|---|
| **Cross-origin** (the scaffold default) | absolute URL baked into the generated client (`http://localhost:<port>`); CORS on the backend (the template accepts any origin: restrict it before cookie auth) | static host or CDN for the SPA; the API elsewhere | the API is public or token-based, or a separate team deploys the SPA |
| **Same-origin, one container** | relative `/api/...` | the .NET runtime serves the API under `/api` and the built SPA from `wwwroot` | cookie auth, one deployable, no CORS, simplest operations |

## Environment variables

Vite reads `.env`, `.env.<mode>`, and `.env.local` files and the process environment.  Only variables prefixed **`VITE_`** reach the browser bundle, as `import.meta.env.VITE_X`, replaced **at build time**.

`src/app/src/env.d.ts`

```ts
/// <reference types="vite/client" />

// Typed `import.meta.env`.  Only `VITE_`-prefixed variables reach the browser bundle; anything
// else stays on the build machine.  They are public: never put a secret in one.
interface ImportMetaEnv {
  /** Backend origin for the generated client; unset keeps the URL baked in by Kubb. */
  readonly VITE_API_BASE_URL?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

- **They are public.**  Anything in the bundle can be read by every visitor: never put a secret, token, or private URL in a `VITE_` variable.
- **They are baked in at build.**  Changing one means rebuilding the front-end.  There is no runtime configuration of a static bundle; if values must differ per deployment without a rebuild, serve them from the backend (`GET /api/config`) and fetch at startup.
- Declare each variable the app code reads in `env.d.ts` so `import.meta.env.VITE_X` is typed (and `undefined` is visible in the type).  `VITE_BASE` is the exception: `vite.config.ts` reads it from `process.env` when the config loads.
- Built-ins: `import.meta.env.DEV`, `.PROD`, `.MODE`, `.BASE_URL`.
- `.env.local` and `*.local` are git-ignored: personal overrides.  Commit `.env` / `.env.production` only if they hold non-secret defaults.

## Pointing a built app at another backend

The generated client has its `baseURL` baked in from `kubb.config.ts` (the backend's launch-profile URL).  `configureApiClient` (`api-client.md`, "Configuring the client") lets `VITE_API_BASE_URL` re-point it at build time without regenerating:

```shell
VITE_API_BASE_URL=https://api.example.com npm run build     # a different backend
VITE_API_BASE_URL= npm run build                             # empty: same-origin (relative URLs)
```

`errors.test.ts` pins the behavior (`vi.stubEnv("VITE_API_BASE_URL", ...)`).

## Sub-path hosting

Served under `/web/` rather than `/`?  Set the Vite `base` and the router follows (`createWebHistory(import.meta.env.BASE_URL)`):

```shell
VITE_BASE=/web/ npm run build     # assets become /web/assets/...
```

The host must serve `index.html` for every path under the base (see the fallback below).

## Build

`npm run build` = `kubb generate` -> `vite build` -> `vue-tsc -b`.  It needs no .NET SDK or running backend: the OpenAPI document is committed, and Kubb generates from it.  Output is `dist/`: `index.html` plus hashed files in `assets/` (cache them forever; never cache `index.html`).  `npm run preview` serves `dist/` locally.

Keep the bundle honest: icons are bundled (`nuxt-ui.md`), routes are code-split per page, and a heavy dependency (a chart or editor library) should be imported in the page that needs it, not in `main.ts`.

## Same-origin, one container

The API and the SPA share one path space, so the API moves under a prefix, unmatched `/api/*` returns a JSON 404 (not `index.html`), and every other path falls back to `index.html` so the client router handles deep links.

**1. Backend: prefix, static files, fallback** (`hg-mdn-backend` skill; the `HttpSetup.cs` template is in `hg-mdn-scaffold`, `references/backend-templates.md`).  `Map<PROJECT>Endpoints("/api")` puts the secured root group under the prefix and adds an anonymous JSON 404 for anything under it that no endpoint matched.  In `Program.cs`, static files come before the endpoints and the fallback after them:

`src/backend/runtime/Program.cs (excerpt)`

```csharp
// The built front-end (`wwwroot`, copied in by the Docker build) is served by the same host.
app.UseDefaultFiles();
app.UseStaticFiles();

app.UseCors();
app.UseSerilogRequestLogging();

// Authentication, then authorization: before the endpoints they protect.
app.Use<PROJECT>Security();

// API under /api so it cannot collide with front-end routes such as /notes.
app.Map<PROJECT>Endpoints("/api");

// Client-side routes (/notes/123) are not files: serve index.html and let the router take over.
app.MapFallbackToFile("index.html");
```

`RouteSecurityLintTests` in `<PROJECT>.Runtime.Tests` must map with the same prefix; its allowlist of anonymous routes becomes `GET /api/health` and `ANY /api/{**unmatched}`.

**2. Front-end: relative URLs and a dev proxy.**  In `kubb.config.ts` delete `baseURL` from `pluginFetch`; the OpenAPI paths are now `/api/...`, so generated URLs are relative.  In development the Vite server forwards `/api` to the backend, which keeps the browser same-origin (and cookies working) there too:

```ts
// vite.config.ts
server: {
  // Aspire injects the backend URL when the front-end resource has `.WithReference(backend)`
  // in `host/AppHost.cs`; the launch-profile URL is the fallback when running Vite alone.
  proxy: { "/api": process.env["services__app-backend__http__0"] ?? "http://localhost:<BACKEND_API_PORT>" },
},
```

**3. Container: a Node stage feeds the .NET publish.**  The Dockerfile gains a front-end stage and one `COPY` into the backend stage:

`build/Dockerfile (excerpt)`

```dockerfile
FROM node:26 AS web
WORKDIR /srcroot/src/app

# Install first, with only the manifests, so this layer is cached until dependencies change.
COPY src/app/package.json src/app/package-lock.json ./
RUN npm ci

# `npm run build` regenerates the API client from the committed OpenAPI document (no .NET needed),
# bundles, and type-checks.
COPY src/app ./
RUN npm run build
```

`build/Dockerfile (excerpt)`

```dockerfile
# The static files the runtime serves; `publish` carries `wwwroot` into the output.
COPY --from=web /srcroot/src/app/dist ./src/backend/runtime/wwwroot
```

Run outside Docker, start the DLL from the project directory: the content root decides where `wwwroot` is.

Gotchas:

- Without the `/api` prefix, a page route like `/notes` and an API route `/notes` collide (the API wins and the page never loads).
- Without the JSON 404 for unmatched `/api/*`, a typo returns `index.html` with status 200, which hides API mistakes from the client.
- `app.UseStaticFiles()` and `UseDefaultFiles()` come before the endpoints; `MapFallbackToFile` after them.
- CORS is unnecessary in this shape; remove `AddCors`/`UseCors` if you do not need them.
- Cookie auth works because everything is one origin; set `credentials: "include"` only for cross-origin.
- **Authentication must be real before this ships.**  Outside Development the template's placeholder authentication answers 401 on every secured API route (only `/health` is public), and the SPA's router sends every visitor to the login page.  Register the real scheme in the backend (`hg-mdn-backend`, `references/route-security.md`) and replace the placeholder login page.

## Cache and security headers

- `index.html`: `Cache-Control: no-cache`; `assets/*`: `public, max-age=31536000, immutable` (the names are content-hashed).
- Set a Content-Security-Policy that allows only your own origin plus the API; because icons are bundled, no Iconify host is needed, and because fonts/images are local, none is allowed beyond `'self'`.
- Terminate TLS in front of the container; mark cookies `Secure` and `HttpOnly`.

## Before shipping

1. The hand-off checklist (`recipes.md`).
2. Open the production build (`npm run preview` or the container) and click through; check the network panel: no requests to third-party hosts, no failed calls.
3. Deep link and refresh on a nested route.
4. The `VITE_*` values in the bundle are the ones you meant (search `dist/assets/*.js`); no secrets in `dist/`.
