# Serving the Front-end from the Runtime

One image, one origin.  The Docker build copies the front-end's `dist/` into the runtime's `wwwroot`; the runtime serves it at `/` and the API under `/api`.  No CORS, cookie authentication just works, one thing to deploy.  (The front-end side, the dev proxy, and hosting the front-end apart instead: `hg-mdn-frontend`, `references/config-deploy.md`.)

| Piece | Where | Does |
|---|---|---|
| API prefix | `Map<PROJECT>Endpoints("/api")` | the API and the SPA share one path space; the prefix keeps them apart, and adds an anonymous JSON 404 under it (`HttpSetup`) |
| Static files | `Use<PROJECT>Spa()` (`<PROJECT>.Api/SpaSetup.cs`) | serves the files in `wwwroot`, with cache headers |
| Fallback | `Map<PROJECT>SpaFallback()` | `/` and any path that is not a file: answer `index.html`, the client router takes over |
| Image | `build/Dockerfile` | a Node stage builds `dist/`, the .NET stage copies it to `wwwroot` and publishes |
| Development | Vite `server.proxy` | the browser talks to Vite; `/api` is forwarded to the backend |
| Optional same-origin development | YARP in the AppHost | routes `/api`, `/scalar`, and `/openapi` to the backend and the rest to Vite (`hg-mdn-scaffold`, `references/aspire-host.md`) |

`SpaSetup.cs` is in `hg-mdn-scaffold`, `references/backend-templates.md`.  Order matters in the `Program.cs` pipeline (`composition-root.md`): `Use<PROJECT>Spa()` comes before request logging, so assets do not flood the log, and before security, because it is public.  `Map<PROJECT>SpaFallback()` comes last, after `Map<PROJECT>Endpoints("/api")`: anything that is not a file or an API route is a client route and answers `index.html`.

## Rules

- **Both methods do nothing without `wwwroot/index.html`.**  Local development is untouched (Vite serves the front-end), so there is no environment check to keep in sync.  To try the hosted front-end without Docker: `npm run build` in `src/app`, copy `dist/*` to `src/backend/runtime/wwwroot` (git-ignored), run the runtime from its project directory (the content root decides where `wwwroot` is).
- **`nonfile` is required on the fallback.**  Without it the fallback endpoint also matches `/assets/app.js`, and the static-file middleware skips any request that already has an endpoint, so no asset is served at all.  With it, a path with an extension is a real 404: a stale chunk name after a deploy must not come back as `index.html` with a 200, which the browser fails to parse as a script.  **Side effect:** a client route that contains a dot (`/users/john.smith`) counts as a file and answers 404 on reload or deep link; keep dots out of route params (ids, slugs).
- **Public on purpose.**  The shell must load before anyone signs in, so static files and the fallback sit outside the secured root group, like `MapOpenApi` and Scalar (`openapi-contract.md`).  The API stays secured by group (`references/route-security.md`); the route lint covers the API routes, not these.
- **Under `/api` everything is the API's.**  A typo is a JSON 404, never `index.html`.  The catch-all answers every method, so a wrong method on a real route (`POST /api/health`) is that 404 too, not a 405.
- **Cache:** fingerprinted files under `assets/` are `public, max-age=31536000, immutable`; everything else (`index.html`, the favicon) is `no-cache`, so a release shows up on the next load.  The rule reads the request path, and the fallback serves `index.html` with it rewritten, so a page route like `/assets/overview` is revalidated too (a test pins it).
- **Sub-path hosting is not part of the template and untested.**  It needs `VITE_BASE` for the build, `RequestPath` on the static files and a prefixed fallback pattern (`"/web/{**path:nonfile}"`), and the cache rule's `/assets` check adjusted to the prefix.

## Tests

`SpaHostingTests` (`<PROJECT>.Api.Tests`) runs the real pipeline in memory over a temporary web root; `ProbeHost.StartWithSpaAsync(webRoot)` wires it like `Program.cs`.

| Test | Proves |
|---|---|
| `SpaRoot_ValidRequest_ReturnsIndexHtmlWithNoCache`, `SpaFallback_ClientRoute_ReturnsIndexHtmlWithNoCache`, `SpaFallback_AssetsPathRoute_ReturnsIndexHtmlWithNoCache` | `/`, `/notes/123`, and `/assets/overview` answer `index.html` with `no-cache` |
| `HashedAsset_FingerprintedPath_ReturnsImmutableCacheHeaders`, `StaticFile_NonHashedPath_ReturnsNoCache` | the cache headers |
| `HashedAsset_StaleChunkPath_ReturnsNotFoundWithoutIndexHtml` | the `nonfile` constraint (with it removed this and `HashedAsset_FingerprintedPath_ReturnsImmutableCacheHeaders` fail) |
| `ApiRoute_UnknownPath_ReturnsJsonNotFound`, `ApiGroup_SecuredAndOpenRequests_PreservesAccessRules` | the SPA does not swallow the API: a typo is a JSON 404, secured routes still answer 401 |
| `SpaHosting_EmptyWebRoot_ServesOnlyTheApi` | no `index.html` in the web root: nothing is served, the API still works |

`build/smoke.sh <base-url> [commit]` checks the same things against a running container (CI runs it on the image it builds; run it against any deployment).  It also asserts the commit the app reports.
