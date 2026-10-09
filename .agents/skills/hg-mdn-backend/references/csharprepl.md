# CSharpRepl: Evaluate Code in the Running Backend

CSharpRepl attaches to the running backend and evaluates C# inside it: real DI, real database, real queue.  Use it to inspect live state, call a handler while bypassing HTTP, or patch behavior without a restart.  Development only: anyone who can attach can run code in the process, so never enable the hook in production.

## How it is wired

- `dotnet:CsharpRepl` is pinned in `.config/mise.toml`.
- In run mode the AppHost calls `csharprepl connect init` (`host/ProcessUtils.cs`) and passes the resulting `DOTNET_STARTUP_HOOKS` and `ASPNETCORE_HOSTINGSTARTUPASSEMBLIES` to the backend resource only.  If `csharprepl` is missing the AppHost logs `Skipping CSharpRepl connect setup` and carries on.
- Never set those variables machine-wide; only on the process to inspect.  Outside Aspire:

  ```bash
  eval "$(csharprepl connect init | grep '^export ')"   # in the launching shell only
  dotnet run --project src/backend/runtime
  ```

## Connect

| Command | Purpose |
|---|---|
| `csharprepl connect list` | Connectable processes with PIDs. |
| `csharprepl connect <pid>` | Interactive session (needs a real TTY). |
| `--eval <code>` | Evaluate one snippet, print, exit.  Exit code is non-zero on any error. |
| `--eval-file <path>` | Evaluate a `.csx` or `.cs` file, print the last expression, exit. |
| `--streamPipedInput` | Treat piped stdin as one submission per line (scripting). |
| `-u <ns>` / `--using <ns>` | **No effect in connect mode.**  Write `using X;` inside the code instead. |

```bash
csharprepl connect list                  # pick the backend PID
# Under Aspire the process is named <PROJECT>.Runtime; launched with `dotnet X.dll` it shows as `dotnet`.
# The "hint" line may point at the CSharpRepl CLI itself: ignore it.
NO_COLOR=1 csharprepl connect 98990 --eval 'services is not null'       # => true
```

Use `NO_COLOR=1` for parseable output.  Multi-line C#, `using` directives, and top-level `await` all work inside the string.

## Globals and resolving services

- `services`: the root `IServiceProvider`.  `Get<T>()` is `services.GetRequiredService<T>()`: **singletons and transients only**.
- Scoped services (DbContext, stores) fail with `Cannot resolve scoped service ... from root provider`.  Use the scope helpers from `DebugRepl.cs` (`src/backend/runtime`, a scaffold template; `using <PROJECT>;` first):

| Helper | Use for |
|---|---|
| `services.Use<TSvc, T>(svc => ...)` | one scoped service, sync |
| `await services.UseAsync<TSvc, T>(async svc => ...)` | one scoped service, async |
| `await services.UseAsync<TSvc1, TSvc2, T>(async (a, b) => ...)` | two services in **one** scope |
| `await services.UseScopedAsync<T>(async sp => ...)` | raw scoped provider: more than two services, keyed services |
| `... .AsJsonAsync()` / `.AsJsonIndentedAsync()` | print a result as unescaped JSON |

Specify every type argument: C# has no partial type inference, so `UseAsync<AppDbContext>` does not compile; the return type `T` is explicit.  Return plain data, not the service: the scope ends when the callback returns.  Do not nest `UseAsync` calls to get two services: each call makes its own scope; use the two-service overload or `UseScopedAsync`.

```bash
# transient handler, no scope needed
NO_COLOR=1 csharprepl connect $PID --eval 'Get<<PROJECT>.Health.HealthHandler>().HandleAsync().Result.Value.Status'

# scoped DbContext
NO_COLOR=1 csharprepl connect $PID --eval 'using <PROJECT>; await services.UseAsync<<PROJECT>.Data.AppDbContext, bool>(async db => await db.Database.CanConnectAsync())'

# query through the process's own EF mappings (not psql) -> JSON
NO_COLOR=1 csharprepl connect $PID --eval 'using <PROJECT>; using Microsoft.EntityFrameworkCore; await services.UseAsync<<PROJECT>.Data.AppDbContext, List<string>>(async db => (await db.Database.GetAppliedMigrationsAsync()).ToList()).AsJsonAsync()'
```

## Patching live code

`#replace`, `#wrap`, `#patches`, `#revert` change behavior for all callers, process-wide, with no rebuild.

- `#replace <Type>.<Member> with <fn>`: swap the implementation.  Instance members take the instance first: `(instance, ...params) => ...`.
- `#wrap <Type>.<Member> with <fn>`: like replace but calls through; first parameter is `orig`: `(orig, instance, ...params) => { ...; return orig(instance, ...params); }`.  No `ref`/`out`/`in` (replace supports them).
- `#patches` lists active patches with ids; `#revert <id>` or `#revert all` undoes them.
- Patches live in the target process, not the CLI session: they persist across `connect` calls and keep affecting real traffic until reverted or the process restarts.  **End every patching session with `#patches` then `#revert all`.**
- Patching an already-patched member replaces the earlier patch.
- Private static methods often fail to patch (`CS8917`): call through a public entry point instead.

```bash
# replace: verify with a real HTTP call, then revert
NO_COLOR=1 csharprepl connect $PID --eval '#replace <PROJECT>.Health.HealthHandler.HandleAsync with (instance) => Task.FromResult(Microsoft.AspNetCore.Http.TypedResults.Ok(new <PROJECT>.Health.HealthResponse("Healthy (replaced!)", DateTimeOffset.UtcNow)))'
curl -s http://localhost:<BACKEND_API_PORT>/health        # => {"status":"Healthy (replaced!)", ...}
NO_COLOR=1 csharprepl connect $PID --eval '#revert all'

# wrap: temporary debug logging without a rebuild; search the logs for the marker, revert
NO_COLOR=1 csharprepl connect $PID --eval '#wrap <PROJECT>.Health.HealthHandler.HandleAsync with (orig, instance) => { Serilog.Log.Information("repl-wrap-marker note={Note}", "hello"); return orig(instance); }'
curl -s localhost:<BACKEND_API_PORT>/health
aspire logs app-backend --non-interactive --search 'repl-wrap-marker'      # => [app-backend] [11:32:29.590 INF] repl-wrap-marker note=hello
NO_COLOR=1 csharprepl connect $PID --eval '#revert all'
```

Wrapping an async method (`Task<Ok<T>>`) works when the compiler can infer the delegate types from the method's signature; with complex generics or overloads it cannot (`CS8917`): call the service through `services.UseAsync` instead of intercepting it.

## Debugging recipes

Substitute your PID; `NO_COLOR=1` keeps output parseable.

**Discover a type's real shape** when a property guess fails with `CS1061`; list the property names, then fix the query in one step:

```bash
NO_COLOR=1 csharprepl connect $PID --eval 'string.Join(", ", typeof(<PROJECT>.Health.HealthResponse).GetProperties().Select(p => p.Name))'   # => "Status, CheckedAtUtc"
```

**Read the configuration the process actually loaded**, not what you think it loaded:

```bash
NO_COLOR=1 csharprepl connect $PID --eval 'Get<Microsoft.Extensions.Configuration.IConfiguration>()["ASPNETCORE_ENVIRONMENT"]'   # => "Development"
```

**Fault injection.**  Force a failure to see how callers, the exception handler, and retries really behave, without a throwaway `throw` in the code.  This hits real traffic until reverted:

```bash
NO_COLOR=1 csharprepl connect $PID --eval '#replace <PROJECT>.Health.HealthHandler.HandleAsync with (instance) => throw new System.InvalidOperationException("repl-fault-injection")'
curl -s -w ' [%{http_code}]\n' localhost:<BACKEND_API_PORT>/health     # => problem-details 500 with a traceId
NO_COLOR=1 csharprepl connect $PID --eval '#revert all'
curl -s -w ' [%{http_code}]\n' localhost:<BACKEND_API_PORT>/health     # => 200 again
```

**A reusable probe script.**  For a diagnostic you run repeatedly, put it in a `.csx` in a scratch directory (not in the repo): `using` lines at the top, awaits, one result expression last.  If a probe earns permanence, promote it to a development-only diagnostic endpoint.

```csharp
// probe.csx  (run: csharprepl connect $PID --eval-file probe.csx)
using <PROJECT>;
using Microsoft.EntityFrameworkCore;

var env = Get<Microsoft.Extensions.Hosting.IHostEnvironment>().EnvironmentName;
var migrations = await services.UseAsync<<PROJECT>.Data.AppDbContext, int>(async db => (await db.Database.GetAppliedMigrationsAsync()).Count());

$"env={env} migrations={migrations}"      // => "env=Development migrations=0"
```

## Input modes and gotchas

| Mode or symptom | Behavior and fix |
|---|---|
| `--eval` | One expression or a few statements; `#`-commands work.  A `using` line before `#wrap` in the same string fails (`CS1040`): write fully qualified names. |
| `--eval-file` | The whole file is one submission; the last expression prints.  A `#`-command works only as the **only line** in the file: one file per `#`-command. |
| `--streamPipedInput` | Stdin is split per line, so `#`-commands and C# mix freely: `printf '%s\n' '#patches' '#revert all' \| NO_COLOR=1 csharprepl connect $PID --streamPipedInput`.  Without the flag the whole stdin is one submission and `#`-commands fail (`CS1024`). |
| top-level `using var scope = ...` or `try`/`catch` in a `.csx` | Fails (`CS1002`): use the scope helpers; wrap the body in an async lambda inside `UseScopedAsync`. |
| a handler ignores your arguments | It may generate its own correlation ids: read the id from its result, then search the logs for it. |
| `connect list` is empty | The hook did not load: `aspire logs app-backend --search "Skipping CSharpRepl connect setup"` explains why (usually `csharprepl` missing from `PATH`: `mise install`). |
