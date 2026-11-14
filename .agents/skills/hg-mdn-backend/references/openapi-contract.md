# The API Contract: C# to OpenAPI to TypeScript

The C# endpoints are the single source of truth for the API.  Everything the front-end knows about the backend is generated:

```text
C# endpoints + contract records (with XML docs)
   │  dotnet build  (Microsoft.Extensions.ApiDescription.Server)
   ▼
src/app/src/api/<project>-api.json         OpenAPI document; COMMITTED
   │  kubb generate  (npm run generate; watch mode under Aspire)
   ▼
src/app/src/api/generated/                 TypeScript types + fetch clients; git-ignored, never hand-edited
```

Scalar (`/scalar`, Development only) renders the same document for humans; `/openapi/v1.json` serves it live.  Setup and options: "Scalar" below.

## Changing the contract

1. Edit the endpoint, handler, or contract record (and its XML docs); write or update `WithSummary` and `WithDescription`.
2. `dotnet build` (or `mise run openapi`): rewrites `src/app/src/api/<project>-api.json`.  Under Aspire a watching build does this on save.
3. `mise run generate` (`npm run generate` in `src/app`; or let `kubb-generate-watch` do it): regenerates `generated/`.
4. Use the new client function or type in a query or mutation in `src/queries` (`hg-mdn-frontend`, `references/data-loading.md`, `references/api-client.md`).  Commit the OpenAPI document with the C# change.

Review the diff of `<project>-api.json`: an unintended contract change shows up there.

## What shapes the generated client

| In C# | In OpenAPI | In TypeScript |
|---|---|---|
| `.WithName("CreateNote")` | `operationId: CreateNote` | `createNote(...)` in `clients/createNote.ts` |
| `.WithSummary("Create a note.")` | operation `summary` | JSDoc `@summary` |
| `.WithDescription("""...""")` | operation `description` | JSDoc `@description`, shown on hover at every call site |
| `.WithTags("Notes")` | `tags` | grouping |
| record `<param>` docs | schema property `description` | JSDoc on the type's fields |
| `Results<Created<NoteResponse>, BadRequest<ApiError>>` | `201` and `400` responses with schemas | typed success and error unions |
| `string Title` | `required` | `title: string` |
| `string? Body` (no default) | **still `required`**, nullable | `body: string \| null`: callers must pass `null` |
| `string? Body = null` | optional, nullable | `body?: string \| null` |
| `[Required]`, `[MaxLength(n)]` | `required`, `maxLength` | documentation only (not enforced; `endpoints.md`) |

Give every optional record parameter a default (`= null`, `= []`, ...); without one the schema marks it required.

`WithName` is unique across the API and `VerbNoun` (`ListNotes`, `CreateNote`, `GetNote`); without it the operation id is derived from the route and the client names get ugly.

## Descriptions are the API documentation

Every operation has a **summary** and a **description**.  They are what a caller reads: in Scalar, in the editor (as JSDoc on the generated client function), and in any tool or agent that consumes the OpenAPI document.  The types say *what shape*; the description says *what it is for and how it behaves*.  The contract lint (`<PROJECT>.Api.Tests`) fails `mise run test` and CI when one is missing or malformed.

**Summary** (`WithSummary`): one short imperative sentence ending in a period that names the action and the thing: `Create a note.`, `List notebooks.`, `Replace a library's source.`

**Description** (`WithDescription`): **one to two paragraphs, one to six sentences in total**, describing the purpose and the high-level domain behavior.  Most need one paragraph; add a second only for a separate caveat.

Write it for the caller, in the present tense, third person, opening with the verb that says what the operation does (`Returns`, `Creates`, `Replaces`, `Queues`, `Removes`).  Cover, in roughly this order:

1. **What it does and where**: the thing, its scope, and its ordering (`Returns notes newest first.`, `...in the caller's active organization.`).
2. **The domain rules a client must know**: uniqueness, limits, immutability, defaults, one-time secrets (`returned only in this response`), side effects (`queues a sync`), idempotency.  **Name the role when it is not the default (signed in)**: "Requires the `editor` role."  Authentication failures are not listed per operation: 401 means nobody is signed in, 403 a missing role, and admin routes answer 404 (`route-security.md`).
3. **How the main parameters change behavior**: ``Pass `notebookId` to list only that notebook; omit it to list every note.``
4. **Domain-level failures**: which situation produces which status and code.  The `Results<...>` type lists the statuses; only the description says why (``Responds 404 with `notebook_not_found` when the notebook does not exist.``).
5. **A second paragraph, only when needed**, for the less common mode, a caveat, a required follow-up call, or an environment restriction (`Development only`).

Do **not**:

- restate the verb and route, or document every field (the `<param>` XML docs and the schema do that);
- describe the implementation (tables, caches, classes, queries);
- use bullet lists, headings, or marketing language (prose only; backticks for fields, parameters, codes, and routes; bold only for a must-not-miss caveat);
- exceed six sentences.  A longer description means the endpoint does too much, or the text is documenting internals: split the endpoint or cut.

Format: a raw string literal with the text starting on the next line, wrapped near 100 columns:

```csharp
.WithName("CreateNote")
.WithSummary("Create a note.")
.WithDescription(
    """
    Creates a note with a title and optional body, and returns it. The title is trimmed,
    required, and limited to 200 characters; a blank or overlong title is rejected with a
    specific error code.

    Pass `notebookId` to file the note in an existing notebook. Responds with 404 and the code
    `notebook_not_found` when that notebook does not exist; omit it to leave the note unfiled.
    """
)
```

| Instead of | Write |
|---|---|
| `Gets notes.` | the `ListNotes` description in `endpoints.md`: what it returns, in what order, what the parameter changes, what the empty case yields |
| `This endpoint queries the notes table ordered by created_at descending and maps rows to DTOs.` | the same sentence: behavior, not mechanism |
| `Creates a notebook.` (as the whole description) | `Creates a notebook and returns it. The name is trimmed, required, and limited to 100 characters; names are not required to be unique.` |

Check your work by reading the operation as someone who has never seen the code: can they call it correctly, and predict each non-2xx response?  The lint test enforces the mechanical rules (present, 1 to 2 paragraphs, 1 to 6 sentences, summary ends in a period); it cannot judge the content, so review it.

## Scalar

`Map<PROJECT>OpenApi()` (`OpenApiSetup`, `composition-root.md`) serves the OpenAPI document (`/openapi/v1.json`) and Scalar (`/scalar`, which redirects to `/scalar/`) in Development only.  Both sit outside the secured groups and are public by design, so no deployed environment may serve them.  `AddOpenApi` and Scalar live in the one class so a change to the document and to its viewer is one edit.

Options set, and why:

| Option | Reason |
|---|---|
| `WithTitle("<PROJECT> API")` | The browser tab and the page header name the product, not `v1`. |
| `ShowOperationId()` | The operation id is the `WithName` value, which is the name of the generated client function.  A reader finds the call to use in the front-end. |

Add an option only with a reason.  The other 2.x options (`ExpandAllTags()`, `WithDefaultHttpClient(...)`, `SortTagsAlphabetically()`, `ForceDarkMode()`, `AddServer(...)`) are cosmetic or answer a question this API does not ask.  Full list: <https://scalar.com/products/api-references/integrations/aspnetcore/integration>.

**"Try it out" works in Development.**  The placeholder authentication signs in the configured development user on every request (`Security:DevelopmentUser`), so Scalar calls `/api/me` with no token and gets 200.  Outside Development the same call answers 401, and Scalar is not served there.

**When a real authentication scheme replaces the placeholder:** declare the scheme in the OpenAPI document with a document transformer.  Scalar only offers schemes the document defines.  Then name it with `AddPreferredSecuritySchemes("<scheme>")` in `Map<PROJECT>OpenApi`.  Do not pre-fill a token with `AddHttpAuthentication`; the page shows it to anyone who can open it.

**The Aspire dashboard:** the AppHost gives the `app-backend` resource a link to `/scalar` (`hg-mdn-scaffold`, `references/aspire-host.md`).  Change both together if the route moves.

## Project settings and the `GetDocument.Insider` trap

The scaffold sets these (`hg-mdn-scaffold`, `references/backend-templates.md`): `OpenApiDocumentsDirectory` (relative to the runtime csproj, pointing at `src/app/src/api`), the file name (`--file-name <project>-api`, which `kubb.config.ts` `input` must match), and `GenerateDocumentationFile` (lets XML docs reach the schema).  Release builds skip generation, so container and CI builds do not need `src/app`.

Document generation runs the whole host under an entry assembly named `GetDocument.Insider`, so registrations need no infrastructure and startup work sits behind a guard: `composition-root.md` ("OpenAPI generation").

## Hosting

The API is mapped under `/api` and shares an origin with the front-end (the runtime serves it in production, the Vite dev server proxies `/api`), so there is no CORS and the generated client has no base URL: `references/spa-hosting.md`.  Hosting the front-end apart from the API (CORS, a base URL): `hg-mdn-frontend`, `references/config-deploy.md`.

## Compatibility

The document is regenerated on every build, so front-end and backend in one repo never drift.  For an API consumed by others, publish under a versioned prefix, add fields as optional first, and treat removed fields and changed types as breaking.

## Failure modes

| Symptom | Cause |
|---|---|
| `dotnet build` fails inside `GenerateOpenApiDocuments` | startup work or a registration needs infrastructure in generation mode (`composition-root.md`, "OpenAPI generation") |
| the JSON never appears in `src/app/src/api` | wrong relative `OpenApiDocumentsDirectory`, or a Release build |
| Kubb: input not found | `kubb.config.ts` `input` file name differs from `--file-name` |
| client function is named oddly | missing `WithName` |
| `/scalar` answers 404 | the environment is not Development; this is the rule, not a defect |
| `Api.Tests` fails: no description, too many sentences or paragraphs, or a summary that is not one sentence ending in a period | add or trim `WithDescription` / `WithSummary` |
| a handler's error type is `unknown` in TypeScript | the handler returns a non-typed result (`IResult`); use `Results<...>` of `TypedResults` |
| types are stale in the editor | `generated/` not regenerated; run `mise run generate` |
