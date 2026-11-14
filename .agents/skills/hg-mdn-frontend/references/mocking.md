# Mocks from the contract (MSW + Faker)

Review the committed OpenAPI contract before handlers exist.  Run `mise run generate`.  Build queries, screens, and tests against generated handlers.  The backend removes `x-implementation: stub` when an operation is ready.  Regenerate the document and click through against the real backend.  A new field needs a contract change first.  See `hg-mdn-backend/references/openapi-contract.md` and `recipes.md`.

## Setup and selection

Use Kubb 5.20.1, `@kubb/plugin-fetch` 5.13.2, `@kubb/plugin-msw` 6.2.3, `@kubb/plugin-faker` 5.4.3, `msw` 3.0.5, and `@faker-js/faker` 10.6.0.  Pin the MSW and Faker plugins and MSW exactly and commit the lockfile.  Plugin peer ranges can allow broken pairs: MSW plugin 5.x fails with this Kubb version.  See `api-client.md` and `hg-mdn-scaffold/references/frontend-templates.md` for the configuration and complete files.

Kubb writes `src/api/generated/{handlers,mocks}`.  Never edit generated files.  Put overrides and state in `src/mocks`.  Kubb can rename path parameters.  The scaffold's `select.ts` matches the HTTP method and path shape, ignoring parameter names.  It preserves literal suffixes such as `.json`.  It selects operations marked `x-implementation: stub` and rejects unknown IDs and selected operations with no handler.  `VITE_MOCK_API=all` selects every generated handler; other nonempty values select stubs only.  Unselected requests reach the Vite proxy.

Run `npx msw init public/ --save` and commit `public/mockServiceWorker.js`.  Do not format or edit this vendor file.  Await the worker before mounting the app.  Start it only under `import.meta.env.DEV && import.meta.env.VITE_MOCK_API`.  The scaffold build plugin removes the worker asset from `dist`.  Check the production build with the flag set: `rg -il 'msw|faker' dist` prints nothing and `dist/mockServiceWorker.js` does not exist.  Service workers need a secure context and one origin.  Verify worker scope separately for sub-path hosting.  MSW 3 rejects repeated start or stop calls; avoid restarting the worker on each HMR update.  See `config-deploy.md`.

## Tests and data

Use MSW for tests that exercise the generated client and Colada together.  Use `vi.mock` for a focused wrapper or component boundary.  Keep a real Pinia and `PiniaColada` in both cases.  See `testing.md`.  MSW cannot detect a wrong generated URL when the handler shares the same contract.

The scaffold starts `setupServer` with generated handlers and `onUnhandledFrame: "error"`.  MSW 3 silently ignores the old `onUnhandledRequest` option.  `request:unhandled` events also fail `afterEach`, since application code can catch the fetch rejection.  A test expecting one calls `takeUnhandled()`.  Tests of HTTP behavior use MSW and never stub `fetch`.  A focused client configuration unit test may stub `fetch` only for that test and must restore it with `vi.unstubAllGlobals()`; the stub bypasses MSW while active.  MSW can log an `InternalError` for an expected unhandled request even when the test consumes it.  Set the happy-dom URL so relative paths match.  Await all requests before a test ends; in-flight requests can reach the next test.

The Faker printer uses words for plain strings and delegates patterns, formats, content, and minimum lengths to Kubb.  It slices words to honor `maxLength`.  The generated factories use `fakerEN`.  Kubb seeds it only at module load.  Reset `fakerEN.seed(1)` and `fakerEN.setDefaultRefDate("2026-06-01T00:00:00Z")` before each test.  Override IDs, dates, relationships, and visible strings used in assertions.  Generated fields are always present and non-null.  Pass explicit null or absent values to fixtures and test both cases.  A handler argument of `null` invokes the Faker fallback; use a resolver with `HttpResponse.json(null)` to send JSON null.  A 204 response must have no body.  Generated handlers choose one primary 2xx response; avoid multiple success statuses that can mismatch the body.  Use a generated status helper to return a declared error.

Cover empty and paged lists, enums, auth and validation failures, not-found, latency, and network failures.  Test optimistic updates and rollback with controlled promises; wait for each mutation to settle before the next assertion.  `server.use(a, b)` puts overrides first, and the first matching handler wins.  Reset handlers and stores after each test.  A `?mock=401` scenario flag does not survive a redirect unless the app explicitly carries it across.

## Stateful example

For behavior across requests, keep a small store outside generated code.  Put these handlers before generated ones and reset the store per test or browser scenario.  Validate request data and implement the contract's errors and ordering before relying on the fake.

```ts
import { HttpResponse } from "msw/http";
import { createNoteHandler } from "@/api/generated/handlers/createNoteHandler";
import { deleteNoteHandler } from "@/api/generated/handlers/deleteNoteHandler";
import { listNotesHandler } from "@/api/generated/handlers/listNotesHandler";
import { createNoteResponse } from "@/api/generated/mocks/createNoteResponse";
import type { NoteResponse } from "@/api/generated/types/NoteResponse";

const notes = new Map<string, NoteResponse>();
export function resetNotes() {
  notes.clear();
}
export const notesHandlers = [
  listNotesHandler(() => HttpResponse.json({ items: [...notes.values()] })),
  createNoteHandler(async ({ request }) => {
    const draft = await request.json();
    const note = createNoteResponse(draft);
    notes.set(note.id, note);
    return HttpResponse.json(note, { status: 201 });
  }),
  deleteNoteHandler(({ params }) => {
    notes.delete(String(params.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```
