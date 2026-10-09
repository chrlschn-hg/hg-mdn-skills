# Telemetry and Logging

Two complementary systems, both wired once in `Program.cs` (`Add<PROJECT>Telemetry`) and both tagged with the source location of the call.  Use **logs** for discrete "what happened" messages and **spans** for "where did the time go" and request flow.

| | Logging (Serilog) | Telemetry (OpenTelemetry span) |
|---|---|---|
| API | `Serilog.ILogger` + `.Here()` | `<PROJECT>Telemetry.Trace/SetTags/AddEvent` |
| Unit | point-in-time message | duration-bearing span, parent/child |
| Use for | human-readable events, warnings, errors | latency diagnosis, request flow across DB/HTTP calls |
| Source location | `{Here}` suffix on the line | `code.file_path`, `code.member`, `code.line_number` attributes |
| Auto-instrumented | no: call it explicitly | yes: ASP.NET Core, `HttpClient`, EF Core, Npgsql; custom spans need `Trace` |

They correlate automatically: a log written while an `Activity` is running (inside a `using var activity = ...Trace(...)` block) is exported with that span's trace id and appears linked to it in the dashboard.  `Activity.Current` is ambient (`AsyncLocal`); there is no trace-id plumbing.

```csharp
// In a handler: a span, a linked log, and an auto-instrumented child span
using var activity = <PROJECT>Telemetry.Trace(tags: [("endpoint", "health")], traceName: "HealthCheck");
Log.Here().Information("Health check requested!");       // linked to the HealthCheck span
await db.Database.ExecuteSqlRawAsync("SELECT 1");         // EF Core instrumentation emits a child span
```

## Logging

Each class declares its own static logger, scoped to the type, instead of injecting `ILogger<T>` or calling the static `Serilog.Log`:

```csharp
private static readonly Serilog.ILogger Log = Serilog.Log.ForContext<CreateNoteHandler>();
```

- `ForContext<T>()` binds the global logger (configured once) to a `SourceContext` of the full type name: built at class load, no DI, no per-request allocation, and per-class filtering.
- `.Here()` adds the per-call-site layer (`[CallerMemberName]`, `[CallerFilePath]`, `[CallerLineNumber]`): chain it on every call: `Log.Here().Information(...)`.
- The field is named `Log` on purpose: it shadows the `Serilog.Log` static class, so call sites need no prefix.
- Console output: `[HH:mm:ss.fff LVL] message ( File.cs:Member@Line)`.  Source location appears only on lines written through `.Here()`; framework lines show an empty `()`.
- Use message templates with named holes (`"Created note {NoteId}"`), never string interpolation: the properties stay queryable.
- Levels: `Debug` for developer detail, `Information` for business events, `Warning` for recoverable oddities, `Error` for failures that need a human.  Do not log and rethrow; do not log secrets, tokens, or request bodies.
- Framework noise is filtered in `TelemetrySetup` (`Microsoft` at Warning); change the overrides there.

Framework code that takes `ILogger<T>` (for example `HttpExceptionHandler`) also reaches Serilog: `services.AddSerilog()` replaces the logging pipeline.

## Spans

`src/backend/packages/<PROJECT>.Telemetry/<PROJECT>Telemetry.cs`

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using System.Runtime.CompilerServices;

namespace <PROJECT>.Telemetry;

/// <summary>
/// Telemetry entry point for the application: owns the `ActivitySource` and `Meter`.  Feature
/// slices call <see cref="Trace"/> / <see cref="AddEvent"/>; extension methods in
/// <see cref="TelemetryExtensions"/> add `SetTags` and friends.
/// </summary>
public static class <PROJECT>Telemetry
{
    public const string ActivitySourceName = "<PROJECT>";

    public static readonly ActivitySource Tracer = new(ActivitySourceName);

    public static readonly Meter Metrics = new(ActivitySourceName);

    public static Activity? AddEvent(
        (string Key, object? Value)[] tags,
        string? eventName = null,
        [CallerMemberName] string name = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0
    ) => Activity.Current?.AddEvent(tags, eventName, name, filePath, lineNumber);

    public static Activity? Trace(
        (string Key, object? Value)[] tags,
        string? traceName = null,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0
    ) => Tracer.Trace(tags, traceName, memberName, filePath, lineNumber);
}
```

- `Trace(tags, traceName?)` starts a child `Activity` under `Activity.Current`, tagged with `tags` plus the captured call site.  It **must** be `using`'d or the duration is never recorded.  The default name is `{File}#{Member}`; pass `traceName` for a stable, searchable name.
- `SetTags(params tags)` (on `Activity.Current`) adds tags to the running span without creating a new one.
- `AddEvent(tags, eventName?)` adds a timestamped, point-in-time event inside the current span.
- Tags are `(string Key, object? Value)` tuples.  Keep them low-cardinality and non-sensitive: ids and sizes, not payloads.
- Span names: noun-ish and stable (`HealthCheck`, `ImportDocuments`); put variable data in tags.

```csharp
<PROJECT>Telemetry.AddEvent([("retry", attempt)], "ImportRetry");
Activity.Current.SetTags(("note.count", notes.Count));
```

## Metrics

`<PROJECT>Telemetry.Metrics` is the application `Meter`; `AddMeter("*")` in `TelemetrySetup` exports every meter.  Create each instrument once, as a static field:

`src/backend/packages/<PROJECT>.Notes/MetricsProbe.cs`

```csharp
using System.Diagnostics.Metrics;
using <PROJECT>.Telemetry;

namespace <PROJECT>.Notes;

internal static class NoteMetrics
{
    // One instrument per measurement, created once.  `AddMeter("*")` in TelemetrySetup exports it.
    private static readonly Counter<long> Created = <PROJECT>Telemetry.Metrics.CreateCounter<long>(
        "notes.created",
        unit: "{note}",
        description: "Notes created."
    );

    public static void NoteCreated(string source) =>
        Created.Add(1, new KeyValuePair<string, object?>("source", source));
}
```

Counters for events (`notes.created`), histograms for durations and sizes, observable gauges for current state.  Names are lowercase, dot-separated; units in UCUM braces (`{note}`).

## Exceptions

Unexpected exceptions are handled once, at the boundary, by `HttpExceptionHandler` (`endpoints.md`, "Error shape"): it logs with the trace id and marks the active span as error.  Handlers do not catch to log.

## Where it goes

- Local development: Aspire sets `OTEL_EXPORTER_OTLP_ENDPOINT`; logs, traces, and metrics land in the Aspire dashboard.
- Read them from the CLI with `aspire logs` and `aspire otel` (see `debugging-aspire.md`).
- Production: point the same OTLP variables at a collector (the exporter reads standard `OTEL_*` environment variables); no code change.
- Resource attributes: `service.name` comes from `TelemetryConfig.ServiceName`; set one per deployable.
