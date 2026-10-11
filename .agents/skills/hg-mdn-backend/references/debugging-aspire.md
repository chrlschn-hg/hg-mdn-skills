# Debugging with the Aspire CLI

Read logs, spans, and traces of the running app from the terminal: no dashboard UI needed.  The stack is started with `mise run up` (detached) and stopped with `mise run down`.  Resource names come from `host/AppHost.cs`: `app-backend`, `app-frontend`, `kubb-generate-watch`, `libredb`, `postgres`.

## Commands

| Command | Purpose |
|---|---|
| `aspire ps` | List running AppHosts: path, SDK, PID, dashboard URL. |
| `aspire describe` | Every resource with state, health, and URLs (front-end and LibreDB ports are assigned dynamically). |
| `aspire logs <resource> -n <N> --search <text>` | Tail and filter log lines. |
| `aspire logs <resource> -f` | Stream logs live. |
| `aspire otel spans <resource> --search <text>` | Individual spans with attributes and source location. |
| `aspire otel spans <resource> --has-error true` | Only error spans. |
| `aspire otel traces <resource> --search <text>` | One row per trace with span count and status. |
| `aspire otel traces <resource> --trace-id <id> --format json` | Full detail for one trace. |
| `aspire resource app-backend rebuild` | Rebuild and restart the backend after source changes (the full stack keeps running). |
| `aspire stop --apphost host/host.csproj` | Stop this repo's stack when several run (otherwise `mise run down`). |

All of `logs`, `otel spans`, and `otel traces` take `--search <text>` (server-side full-text and field filter; syntax: <https://aka.ms/aspire/cli-search>) and `--format Json|Table`.

**Starting and stopping.**  `mise run up` and `mise run down` (`aspire stop`) start and stop a stack you started.  `aspire ps` shows every AppHost on the machine; with more than one running, pass `--apphost host/host.csproj` to `stop`, `logs`, `describe`, and the others, so you never act on another project's stack.  Do not stop a stack you did not start: a foreground host belongs to the person you are working with, so ask first.  Prefer `aspire resource <name> rebuild` or `restart` to restarting the host.  Smoke-test reachability with the backend's `/api/health` endpoint.

## `--dashboard-url`

`aspire otel ...` needs the dashboard URL.  Auto-discovery sometimes fails with `Could not fetch telemetry data from the dashboard. The dashboard is not available.`  Pass the URL from `aspire ps` explicitly; it also gives clean stdout.  The dashboard is on `15100`, the template's fixed default (change it per project if it collides); the other resource ports are assigned (`aspire describe`).

```bash
aspire otel spans app-backend --search HealthCheck --dashboard-url http://localhost:15100 --format json
aspire otel traces app-backend --dashboard-url http://localhost:15100
```

## Output banners

Without `--nologo`, commands print banner lines to stdout *before* the data, which breaks naive `| jq`:

- `aspire logs --format json` prints two lines first: `Scanning for running AppHosts...` and `Getting logs...`.
- `aspire otel ... --format json` prints `Scanning for running AppHosts...` when `--dashboard-url` is omitted.

```bash
# skip the banner lines yourself
aspire logs app-backend --search "Health" --format json | tail -n +3 | jq .
# or pass --dashboard-url (otel) for clean output
```

## JSON shapes

- `logs`: `{"logs":[{"resourceName","content","isError"}]}`.  No structured timestamp, level, or location: everything is in `content`, in the console format (`telemetry-logging.md`).
- `otel spans`: a flat array.  Each span: `traceId`, `spanId`, `kind`, `name`, `source`, `durationMs`, `timestamp`, `attributes`, `dashboardUrl`.  Spans from `<PROJECT>Telemetry.Trace` carry structured `code.file_path`, `code.member`, `code.line_number` plus your tags: parse them directly instead of regexing log text.
- `otel traces`: the table is a summary (timestamp, name, span count, duration, status); use `otel spans` or `--trace-id ... --format json` for attributes.

```bash
# every span of one trace with its source location
aspire otel spans app-backend --trace-id <traceId> --dashboard-url http://localhost:15100 --format json \
  | jq '.[] | {name, durationMs, file: .attributes["code.file_path"], line: .attributes["code.line_number"]}'
```

## Workflows

**"Why did this request fail?"**  Reproduce it, then `aspire logs app-backend -n 50 --search "Unhandled"`.  The line carries the trace id; pass it to `aspire otel traces ... --trace-id`.  The problem-details response body has the same trace id in its `traceId` field.

**"Where is the time going?"**  `aspire otel traces app-backend` for the slow trace, then `otel spans ... --trace-id`.  EF Core child spans show the SQL (with the `TagWithCallSite` comment pointing at the store line).

**After editing backend code.**  `aspire resource app-backend rebuild`.  Do not restart the whole stack: it is slow.  Changes to the AppHost itself need `mise run down && mise run up` (for a stack you started).

**Evaluating live code.**  Use the REPL (see `csharprepl.md`) to inspect DI state or patch a method without rebuilding.

## Looking at the database

Use `psql` (installed by Mise) for read-only questions the logs and traces cannot answer; the connection details are the Postgres resource's values in `aspire describe postgres --format json`.  **Never update or delete unless asked**: describe the statement and the rows it would affect, and wait for confirmation.  For inspecting through the app's own mappings, the REPL is often better (`csharprepl.md`).

## Reset the database

The Postgres container is persistent (`ContainerLifetime.Persistent`): it and its volume outlive `aspire stop` and `mise run down`.  To start from an empty database, remove the Postgres container and its volume (find them with `docker ps -a` and `docker volume ls`), then `mise run up`; migrate-on-start rebuilds the schema.  This deletes all local data.

## Aspire skills

`aspire agent init` installs the `aspire-orchestration` and `aspire-monitoring` agent skills (the scaffold does this); they cover the rest of the CLI (wiring new resources, deployment).  Use them for anything beyond debugging the local stack.
