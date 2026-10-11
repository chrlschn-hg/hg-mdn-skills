# Serving the Front-end from the Runtime

One image, one origin.  The Docker build copies the front-end's `dist/` into the runtime's `wwwroot`; the runtime serves it at `/` and the API under `/api`.  No CORS, cookie authentication just works, one thing to deploy.  (The front-end side, the dev proxy, and hosting the front-end apart instead: `hg-mdn-frontend`, `references/config-deploy.md`.)

| Piece | Where | Does |
|---|---|---|
| API prefix | `Map<PROJECT>Endpoints("/api")` | the API and the SPA share one path space; the prefix keeps them apart, and adds an anonymous JSON 404 under it (`HttpSetup`) |
| Static files | `Use<PROJECT>Spa()` (`<PROJECT>.Api/SpaSetup.cs`) | serves the files in `wwwroot`, with cache headers |
| Fallback | `Map<PROJECT>SpaFallback()` | `/` and any path that is not a file: answer `index.html`, the client router takes over |
| Image | `build/Dockerfile` | a Node stage builds `dist/`, the .NET stage copies it to `wwwroot` and publishes |
| Development | Vite `server.proxy` | the browser talks to Vite; `/api` is forwarded to the backend |

`SpaSetup.cs` is in `hg-mdn-scaffold`, `references/backend-templates.md`.  The pipeline in `Program.cs`, which is also the order that matters (static files before request logging and security; the fallback after the API):

```csharp
app.UseExceptionHandler();

// The built front-end, when it is in `wwwroot` (the Docker image).  Public, and before request
// logging so assets do not flood the log.
app.Use<PROJECT>Spa();
app.UseSerilogRequestLogging();

// Authentication, then authorization: before the endpoints they protect.
app.Use<PROJECT>Security();

// The API lives under /api so it cannot collide with front-end routes such as /notes.
app.Map<PROJECT>Endpoints("/api");

// Anything else that is not a file is a client-side route: index.html.
app.Map<PROJECT>SpaFallback();
```

## Rules

- **Both methods do nothing without `wwwroot/index.html`.**  Local development is untouched (Vite serves the front-end), so there is no environment check to keep in sync.  To try the hosted front-end without Docker: `npm run build` in `src/app`, copy `dist/*` to `src/backend/runtime/wwwroot` (git-ignored), run the runtime from its project directory (the content root decides where `wwwroot` is).
- **`nonfile` is required on the fallback.**  Without it the fallback endpoint also matches `/assets/app.js`, and the static-file middleware skips any request that already has an endpoint, so no asset is served at all.  With it, a path with an extension is a real 404: a stale chunk name after a deploy must not come back as `index.html` with a 200, which the browser fails to parse as a script.  **Side effect:** a client route that contains a dot (`/users/john.smith`) counts as a file and answers 404 on reload or deep link; keep dots out of route params (ids, slugs).
- **Public on purpose.**  The shell must load before anyone signs in, so static files and the fallback sit outside the secured root group, like `MapOpenApi` and Scalar.  The API stays secured by group (`references/route-security.md`); the route lint covers the API routes, not these.
- **Under `/api` everything is the API's.**  A typo is a JSON 404, never `index.html`.  The catch-all answers every method, so a wrong method on a real route (`POST /api/health`) is that 404 too, not a 405.
- **Cache:** fingerprinted files under `assets/` are `public, max-age=31536000, immutable`; everything else (`index.html`, the favicon) is `no-cache`, so a release shows up on the next load.  The rule reads the request path, and the fallback serves `index.html` with it rewritten, so a page route like `/assets/overview` is revalidated too (a test pins it).
- **Sub-path hosting is not part of the template and untested.**  It needs `VITE_BASE` for the build, `RequestPath` on the static files and a prefixed fallback pattern (`"/web/{**path:nonfile}"`), and the cache rule's `/assets` check adjusted to the prefix.

## Tests

`SpaHostingTests` (`<PROJECT>.Api.Tests`) runs the real pipeline in memory over a temporary web root; `ProbeHost.StartWithSpaAsync(webRoot)` wires it like `Program.cs`.

| Test | Proves |
|---|---|
| `Root_ServesIndexHtml_AndRevalidates`, `ClientSideRoute_ServesIndexHtml`, `ClientRouteUnderAssets_IsIndexHtml_AndRevalidated` | `/`, `/notes/123`, and `/assets/overview` answer `index.html` with `no-cache` |
| `HashedAsset_IsCachedForever`, `OtherFiles_AreRevalidated` | the cache headers |
| `MissingAsset_Is404_NotIndexHtml` | the `nonfile` constraint (with it removed this and `HashedAsset_IsCachedForever` fail) |
| `UnknownApiRoute_Is404Json_NotIndexHtml`, `ApiRoutes_KeepTheirSecurity` | the SPA does not swallow the API: a typo is a JSON 404, secured routes still answer 401 |
| `WithoutABuiltFrontEnd_NothingIsServed` | no `index.html` in the web root: nothing is served, the API still works |

`build/smoke.sh <base-url> [commit]` checks the same things against a running container (CI runs it on the image it builds; run it against any deployment).  It also asserts the commit the app reports.
