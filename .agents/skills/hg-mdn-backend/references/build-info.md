# Build Info: Version, Commit, Build Time

The running app can say which build it is.  The build stamps three values, `BuildInfo` reads them back, and they appear in four places: `GET /api/health`, the startup log (`Starting 1.2.3 (commit 3f2a1bc, built 2026-10-11 19:19:56Z)`), OpenTelemetry's `service.version` on every log record, span, and metric, and the UI (`BuildInfoLabel`, `hg-mdn-frontend`).

| Value | Local build | Release image |
|---|---|---|
| Version | `0.0.0-dev` (`VersionPrefix` and `VersionSuffix` in `Directory.Build.props`) | `-p:Version=1.2.3` from the `VERSION` build arg; the publish workflow sets it from a `v1.2.3` tag, other builds keep `0.0.0-dev` |
| Commit | the SDK reads it from git and appends it to the informational version (`0.0.0-dev+<sha>`), no setup; a repository with no commit has none | the image has no `.git`: `-p:SourceRevisionId=<sha>` from the `GIT_SHA` build arg |
| Build time | not stamped | `AssemblyMetadata("BuildTimeUtc", UtcNow)`, Release only |

Why the build time is Release only: a timestamp changes the assembly on every build, so every Debug build would recompile the runtime project.

`src/backend/Directory.Build.props`

```xml
<PropertyGroup>
  <!-- Version of a build that is not given one.  Releases pass `-p:Version=1.2.3` (and the commit
       as `-p:SourceRevisionId=<sha>`); the runtime reports both (`BuildInfo`). -->
  <VersionPrefix>0.0.0</VersionPrefix>
  <VersionSuffix>dev</VersionSuffix>
</PropertyGroup>
```

`src/backend/runtime/<PROJECT>.Runtime.csproj`

```xml
  <ItemGroup Condition="'$(Configuration)' == 'Release'">
    <!-- Read back by `BuildInfo`.  Release only: a timestamp changes the assembly on every build,
         which would make every Debug build recompile the runtime project. -->
    <AssemblyMetadata Include="BuildTimeUtc" Value="$([System.DateTime]::UtcNow.ToString('o'))" />
  </ItemGroup>
```

## Reading it

`BuildInfo` (`<PROJECT>.Health/BuildInfo.cs`, in the Health slice so it has a test project; file in `hg-mdn-scaffold`, `references/backend-templates.md`) parses the informational version at the first `+` and reads the metadata; a missing or malformed stamp degrades to `unknown` or null instead of throwing.  The runtime registers one instance in `Program.cs`, passes its version to telemetry (`TelemetryConfig.ServiceVersion`), and logs it at startup:

```csharp
var buildInfo = BuildInfo.From(typeof(Program).Assembly);
builder.Services.AddSingleton(buildInfo);
```

The health handler maps it onto its own contract record (`BuildResponse`), so the API shape does not depend on how the stamp is stored.

## In the image

`ARG VERSION` and `ARG GIT_SHA` sit right before the publish `RUN` in `build/Dockerfile` (full file: `hg-mdn-scaffold`, `references/containers-ci.md`): an arg change invalidates the cache of everything after it, and restore must stay cached.

- **`${VERSION:+-p:Version=$VERSION}`, not `-p:Version=$VERSION`.**  An unset arg would pass an empty `-p:Version=`, which overrides the default with nothing.  The `:+` form passes the property only when the arg is set.  The same for `GIT_SHA` and `SourceRevisionId`.
- Changing `Version` between restore and publish (`--no-restore`) is fine.
- `mise run image` passes the commit with `git rev-parse --verify -q HEAD || true`: a repository with no commit yet gives an empty arg, where plain `git rev-parse HEAD` would print the literal `HEAD`.
- `docker/metadata-action` already labels the image (`org.opencontainers.image.version`, `.revision`) in CI.

The publish workflow derives the build args from the ref: the commit always, the version from a `v1.2.3` tag (`GITHUB_REF_NAME` without the `v`); a branch push has no version and reports `0.0.0-dev`.  Releasing is tagging: `git tag v1.2.3 && git push origin v1.2.3`.

## Exposure

`GET /api/health` is anonymous (probes and "what is deployed?" need it), so it shows the version and commit to anyone.  That is usually fine for an internal or SaaS app; if it must not be public, move `Build` to a secured route (`GET /api/about`, the root group secures it) and show it in the UI after sign-in, keeping `/api/health` to the status.

## Tests

| Test | Proves |
|---|---|
| `Parse_SplitsTheVersionFromTheCommit` | `1.4.0-rc.2+<sha>` gives version, full commit, short commit, build time |
| `Parse_WithoutACommit_ReportsNoCommit`, `Parse_WithoutAVersion_ReportsUnknown`, `Parse_IgnoresATimestampItCannotRead` | missing or malformed stamps degrade instead of throwing |
| `From_ReadsTheStampOfTheAssemblyTheBuildProduced` | the real attribute is read |
| `Health_ReportsTheRunningBuild` | the handler maps `BuildInfo` onto the contract |
| `build/smoke.sh <url> <commit>` | a built image reports the commit it was built from (CI passes `$GITHUB_SHA`) |

The OpenAPI document's own `info.version` (`1.0.0`) is the API contract's version, not the application's.
