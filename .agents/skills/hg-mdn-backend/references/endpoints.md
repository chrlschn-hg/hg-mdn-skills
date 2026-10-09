# Endpoints and Handlers

Minimal APIs, organized as one `IEndpoint` per slice and one `IEndpointHandler` per operation.  Registration is automatic (`Add<PROJECT>Endpoints` scans assemblies; `Map<PROJECT>Endpoints` maps them).  Contracts flow to the front-end through OpenAPI (`openapi-contract.md`).

## Responsibilities

| Piece | Owns | Does not own |
|---|---|---|
| `<Slice>Endpoints` | route, verb, `WithName`, `WithSummary`, `WithDescription`, tags, route guards and roles (on top of the inherited authentication) | behavior |
| Handler | validation, orchestration, choosing the result; takes bound parameters (and a `ClaimsPrincipal` when authenticated), **never `HttpContext`**, so tests call `HandleAsync` directly | routing, SQL, JSON |
| Store | persistence | validation, HTTP |
| Mapping | domain <-> contract | decisions |

## The endpoint

`src/backend/packages/<PROJECT>.Notes/NoteEndpoints.cs (excerpt)`

```csharp
        var notes = endpoints.MapGroup("notes").WithTags("Notes");

        notes
            .MapGet(
                "/",
                (Guid? notebookId, ListNotesHandler handler, CancellationToken ct) =>
                    handler.HandleAsync(notebookId, ct)
            )
            .WithName("ListNotes")
            .WithSummary("List notes.")
            .WithDescription(
                """
                Returns notes newest first. Pass `notebookId` to list only the notes filed in that
                notebook; omit it to list every note. A notebook with no notes, or one that does
                not exist, yields an empty list rather than an error.
                """
            );
```

- `WithName(...)` becomes the OpenAPI `operationId`, which Kubb turns into the client function (`ListNotes` -> `listNotes`).  Always set it; keep it a unique `VerbNoun`.
- `WithSummary(...)` is the one-line summary; `WithDescription("""...""")` is the **API documentation**.  Every endpoint has both; format and content: `openapi-contract.md`.
- Group by resource with `MapGroup("notes").WithTags("Notes")`.  Route guards attach to the group.
- **Security comes from the group you are given.**  `MapEndpoints` receives the secured root group (every route needs a signed-in user), or the `/admin` group when the slice implements `IAdminEndpoint`.  Write no `RequireAuthorization()` per route; opt **out** with `AllowAnonymous()` and a `// NOTE:`.  Layers, guards, and tests: `route-security.md`.
- The handler is a delegate parameter; ASP.NET Core resolves it from DI (registered by the scan).  `CancellationToken` binds automatically; pass it down.
- No `Produces<T>()` is needed: the handler's `Results<...>` return type is the source of the response metadata.  It says *which* statuses exist, not *why*: the description says why ("responds 404 with `notebook_not_found` when ...").
- XML docs on the handler method are for developers; they do not become the operation description.

## The handler

`src/backend/packages/<PROJECT>.Notes/NoteEndpoints.Handler.CreateNote.cs`

```csharp
using <PROJECT>.Api;
using <PROJECT>.Telemetry;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Http.HttpResults;

namespace <PROJECT>.Notes;

/// <summary>
/// Handler for creating a note.
/// </summary>
public sealed class CreateNoteHandler(INoteStore store) : IEndpointHandler
{
    private static readonly Serilog.ILogger Log = Serilog.Log.ForContext<CreateNoteHandler>();

    /// <summary>
    /// Validates the request and persists the note.  Validation is explicit (guard clauses); each
    /// guard returns a distinct error code so the front-end can give targeted feedback.
    /// </summary>
    public async Task<
        Results<Created<NoteResponse>, BadRequest<ApiError>, NotFound<ApiError>>
    > HandleAsync(CreateNoteRequest request, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(request.Title))
        {
            return TypedResults.BadRequest(new ApiError("invalid_title", "Title is required."));
        }

        if (request.Title.Length > NoteLimits.MaxTitleLength)
        {
            return TypedResults.BadRequest(
                new ApiError(
                    "title_too_long",
                    $"Title may be at most {NoteLimits.MaxTitleLength} characters."
                )
            );
        }

        if (request.Body?.Length > NoteLimits.MaxBodyLength)
        {
            return TypedResults.BadRequest(
                new ApiError(
                    "body_too_long",
                    $"Body may be at most {NoteLimits.MaxBodyLength} characters."
                )
            );
        }

        if (
            request.NotebookId is { } notebookId
            && !await store.NotebookExistsAsync(notebookId, ct)
        )
        {
            return TypedResults.NotFound(
                new ApiError("notebook_not_found", "The notebook does not exist.")
            );
        }

        using var activity = <PROJECT>Telemetry.Trace(
            tags: [("note.title.length", request.Title.Length)]
        );

        var note = await store.CreateAsync(request.ToNote(), ct);

        Log.Here().Information("Created note {NoteId}", note.Id);

        return TypedResults.Created($"/notes/{note.Id}", note.ToResponse());
    }
}
```

Rules:

- **Return `Results<A, B, ...>` of `TypedResults`.**  The union documents every status code in OpenAPI and gives the generated client typed errors.  A handler with one outcome returns `Ok<T>` (`ListNotesHandler`).
- **Validate explicitly with guard clauses**, in a fixed order, each returning a distinct `ApiError` code (`snake_case`).  The front-end switches on `Code`; the message is for humans.
- **Expected failures are results, not exceptions.**  Validation -> `BadRequest<ApiError>`; duplicate or limit -> `Conflict<ApiError>`; a row that is missing or that the caller may not see -> `NotFound`, so existence is not leaked.  Which status means what: `route-security.md`, "Status codes at a glance".
- **Unexpected failures throw** and are handled once, at the boundary, by `HttpExceptionHandler` (see "Error shape").  Handlers do not catch to log.
- Create the span after validation succeeds, with cheap, non-sensitive tags; log with `Log.Here()` (`telemetry-logging.md`).
- Keep handlers short.  A handler that needs branching business rules gets a domain service in the slice.

## Contracts

`src/backend/packages/<PROJECT>.Notes/NoteEndpoints.Contracts.cs (excerpt)`

```csharp
/// <param name="Title">Short title (required).</param>
/// <param name="Body">Optional body text.</param>
/// <param name="NotebookId">Optional notebook to file the note in.</param>
public sealed record CreateNoteRequest(
    [property: Required, MaxLength(NoteLimits.MaxTitleLength)] string Title,
    [property: MaxLength(NoteLimits.MaxBodyLength)] string? Body = null,
    Guid? NotebookId = null
);
```

- Records with XML docs on every parameter: the docs become OpenAPI `description`s and reach the generated TypeScript types.
- **Optional means a default value.**  `string? Body = null` is optional in the contract; `string? Body` is required-but-nullable, so the generated client forces callers to pass `null`.
- **Validation attributes are OpenAPI metadata only.**  `[Required]` and `[MaxLength]` reach the schema and the generated client but are never enforced on request records (not even with `AddValidation()`).  Back every constraint with a guard clause.
- Define limits once (`NoteLimits`) and use the constant in the attribute, the guard, and the EF mapping.
- Instants are `DateTimeOffset` named `...AtUtc` (`data-efcore.md`).  Enums: switch over them exhaustively (`code-style.md`); string serialization is under "JSON options".

## Mapping

`src/backend/packages/<PROJECT>.Notes/NoteMapping.cs`

```csharp
namespace <PROJECT>.Notes;

/// <summary>
/// Maps between domain types and API contracts.  All mapping lives here so handlers stay
/// focused on validation and orchestration.
/// </summary>
internal static class NoteMapping
{
    extension(Note note)
    {
        public NoteResponse ToResponse() =>
            new(note.Id, note.NotebookId, note.Title, note.Body, note.CreatedAtUtc);
    }

    extension(CreateNoteRequest request)
    {
        public Note ToNote() =>
            new()
            {
                Title = request.Title.Trim(),
                Body = request.Body ?? "",
                NotebookId = request.NotebookId,
            };
    }

    extension(Notebook notebook)
    {
        public NotebookResponse ToResponse() =>
            new(notebook.Id, notebook.Name, notebook.CreatedAtUtc);
    }

    extension(CreateNotebookRequest request)
    {
        public Notebook ToNotebook() => new() { Name = request.Name.Trim() };
    }
}
```

## Error shape

Expected errors use the shared `ApiError(Code, Message)` record from `<PROJECT>.Api`.  Unexpected errors use problem details, written by `HttpExceptionHandler`, which also logs with the trace id and marks the active span as an error:

```json
{
  "status": 500,
  "title": "An unexpected error occurred.",
  "instance": "/notes",
  "traceId": "00-239044669adc343bdf1928e9fb2e38f8-36ee816d5a2427c2-00"
}
```

`detail` carries the exception text in Development only.  The trace id is the second segment of `traceId`: search it with `aspire otel traces` (`debugging-aspire.md`).

## JSON options

Defaults are camelCase.  Not in the template: configure `JsonStringEnumConverter` via `ConfigureHttpJsonOptions` when the first enum crosses the wire, so the OpenAPI schema and the wire agree.

## Checklist for a new endpoint

1. Contract records (with docs) and a guard clause for every constraint.
2. **Handler test first**: each guard and the happy path, fake store, data from the entity graph (`testing.md`, `entity-graph.md`).
3. Handler returning `Results<...>`; register nothing by hand.
4. `MapGet/MapPost(...)` with `WithName`, `WithSummary`, and `WithDescription` in the slice's `IEndpoint`.  Security: nothing to add for "signed-in users only"; otherwise a role, a group guard, `AllowAnonymous` with its comment and a line in the lint test, or `IAdminEndpoint` (`route-security.md`).  Name the role in `WithDescription` when it is not the default (signed in).
5. `dotnet build`, then read the operation in `src/app/src/api/<project>-api.json` or the Scalar UI: does the description tell a caller what they need to know?  Then `mise run generate`.
6. Run `<PROJECT>.Api.Tests` (descriptions lint, route-security tests) and `<PROJECT>.Runtime.Tests` (every route secured or listed as anonymous).
