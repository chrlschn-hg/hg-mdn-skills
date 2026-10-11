# Static Analysis: Zero Warnings, Analyzers as Guard Rails

The compiler is the fastest reviewer an agent has: a rule it enforces fails `dotnet build` seconds after the mistake, with a file, a line, and a rule id, instead of surfacing in review or production.  So: **every warning is an error**, the built-in analyzers are on, and project-specific rules ("handlers never take `HttpContext`", "no `DateTime.UtcNow`") become analyzers rather than prose.

| Layer | Catches | Cost to add |
|---|---|---|
| `TreatWarningsAsErrors` | every compiler and analyzer warning, including nullable warnings | one property |
| `.editorconfig` severities + `EnforceCodeStyleInBuild` | style rules (braces, exhaustive switches: `code-style.md`) | one line per rule |
| `AnalysisLevel` (built-in .NET analyzers, `CAxxxx`) | correctness, performance, globalization, API design | one property, then a fix pass |
| `Microsoft.CodeAnalysis.BannedApiAnalyzers` | calls to APIs this project forbids | a package and a text file |
| A custom Roslyn analyzer | any rule you can state in terms of the code's structure | a small project |

## Zero tolerance: warnings are errors

```xml
<!-- src/backend/Directory.Build.props -->
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <!-- Optional, deliberate: the built-in analyzers at the recommended level (see below). -->
  <AnalysisLevel>latest-recommended</AnalysisLevel>
</PropertyGroup>
```

- **A warning that is tolerated is invisible.**  Once a build prints twenty warnings nobody, human or agent, notices the twenty-first.  With zero tolerance, a new warning is a failed build that names its cause.
- **Fix, do not silence.**  When a rule is wrong for this codebase, change its severity once in `.editorconfig` **with the reason as a comment**; never `#pragma warning disable` to make a build pass, and never a blanket `NoWarn` for a category.
- **NuGet warnings become errors too**: `NU1701` (a package restored for the wrong framework) and the vulnerability audit warnings (`NU1901`-`NU1904`) now fail the build.  That is the point: a newly published vulnerability stops the build until the package is upgraded.  Do not downgrade them to keep a build green.
- `CS1591` (missing XML doc) stays suppressed in the template: docs are required on public types and contract parameters by convention (`code-style.md`), not on every member.

## The built-in analyzers: `AnalysisLevel`

`latest-recommended` turns on a curated set of `CA` rules as warnings (so, with the setting above, errors).  Adopt it deliberately: the first build reports what the codebase violates, and each rule is either fixed or overridden with a reason.  Two rules that fight this codebase's conventions on purpose (both seen when enabling it):

```ini
# .editorconfig
[*.cs]
# Handlers and stores are instances resolved from DI; "mark it static" fights the design.
dotnet_diagnostic.CA1822.severity = none
```

```xml
<!-- Directory.Build.props: test names are Entity_State_Expected; CA1707 forbids the underscores. -->
<PropertyGroup Condition="$(MSBuildProjectName.EndsWith('.Tests'))">
  <NoWarn>$(NoWarn);CA1707</NoWarn>
</PropertyGroup>
```

Rules worth keeping that agents commonly trip: `CA1310`/`CA1309` (culture-sensitive string comparison: pass `StringComparison.Ordinal`), `CA2016` (forward the `CancellationToken`), `CA1849` (a synchronous call in an async method), `CA1848` (use source-generated logging on hot paths, if it fires).

## Banning APIs

The cheapest custom rule: list the symbols the project forbids, with the reason the developer sees.

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.CodeAnalysis.BannedApiAnalyzers" PrivateAssets="all" />
  <AdditionalFiles Include="BannedSymbols.txt" />
</ItemGroup>
```

```text
P:System.DateTime.UtcNow;Use TimeProvider (inject it) so tests control time
P:System.DateTimeOffset.UtcNow;Use TimeProvider (inject it) so tests control time
```

A use fails the build: `error RS0030: The symbol 'DateTimeOffset.UtcNow' is banned in this project: Use TimeProvider (inject it) so tests control time`.  Symbol ids use the XML-doc format (`T:` type, `M:` method, `P:` property, `F:` field, `N:` namespace).  Put the package and the file in `Directory.Build.props` for every project, or only in the slices a rule applies to.  Adopt a ban only after migrating the existing uses (the template's entities and `HealthHandler` use `DateTimeOffset.UtcNow`).

## A custom analyzer

When a rule is about the code's structure ("a type implementing `IEndpointHandler` must not take `HttpContext`"), write an analyzer.  It lives in its own project, ships with the solution, and runs in every build and in the IDE.

**The project** (`src/backend/packages/<PROJECT>.Analyzers`): `netstandard2.0` is required (the compiler loads it), and the Roslyn version must not be newer than the SDK's compiler, so use the version that shipped with the SDK's major release (`5.0.0` for the .NET 10 SDK):

`<PROJECT>.Analyzers/<PROJECT>.Analyzers.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <IsRoslynComponent>true</IsRoslynComponent>
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="5.0.0" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.11.0" PrivateAssets="all" />
  </ItemGroup>
  <ItemGroup>
    <AdditionalFiles Include="AnalyzerReleases.Shipped.md" />
    <AdditionalFiles Include="AnalyzerReleases.Unshipped.md" />
  </ItemGroup>
</Project>
```

- `EnforceExtendedAnalyzerRules` plus `Microsoft.CodeAnalysis.Analyzers` check the analyzer itself (no file I/O, no banned APIs inside the compiler).
- **Release tracking** (`RS2008`) requires every rule id to be listed in `AnalyzerReleases.Unshipped.md` (and moved to `Shipped.md` on release); both are `AdditionalFiles`:

`<PROJECT>.Analyzers/AnalyzerReleases.Unshipped.md`

```markdown
; Unshipped analyzer release
; https://github.com/dotnet/roslyn-analyzers/blob/main/src/Microsoft.CodeAnalysis.Analyzers/ReleaseTrackingAnalyzers.Help.md

### New Rules

Rule ID | Category | Severity | Notes
--------|----------|----------|-------
<PREFIX>0001 | Design | Error | HandlerTakesHttpContextAnalyzer
```

**The analyzer** (symbol-based: resolve the types once per compilation, then inspect each method):

`<PROJECT>.Analyzers/HandlerTakesHttpContextAnalyzer.cs`

```csharp
using System.Collections.Immutable;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Diagnostics;

namespace <PROJECT>.Analyzers;

/// <summary>
/// <PREFIX>0001: an endpoint handler's method takes <c>HttpContext</c>.  Handlers take bound
/// parameters so tests can call them directly; HTTP stays in the endpoint mapping.
/// </summary>
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public sealed class HandlerTakesHttpContextAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor Rule = new(
        id: "<PREFIX>0001",
        title: "Handlers do not take HttpContext",
        messageFormat: "'{0}' takes HttpContext; bind the values it needs as parameters instead",
        category: "Design",
        defaultSeverity: DiagnosticSeverity.Error,
        isEnabledByDefault: true,
        description: "Handlers are called directly by tests; the endpoint binds HTTP inputs."
    );

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();

        context.RegisterCompilationStartAction(start =>
        {
            // Resolve the types once per compilation; no rule applies if either is missing.
            var handler = start.Compilation.GetTypeByMetadataName("<PROJECT>.Api.IEndpointHandler");
            var httpContext = start.Compilation.GetTypeByMetadataName(
                "Microsoft.AspNetCore.Http.HttpContext"
            );

            if (handler is null || httpContext is null)
            {
                return;
            }

            start.RegisterSymbolAction(
                symbol => AnalyzeMethod(symbol, handler, httpContext),
                SymbolKind.Method
            );
        });
    }

    private static void AnalyzeMethod(
        SymbolAnalysisContext context,
        INamedTypeSymbol handler,
        INamedTypeSymbol httpContext
    )
    {
        var method = (IMethodSymbol)context.Symbol;

        if (
            method.DeclaredAccessibility != Accessibility.Public
            || !method.ContainingType.AllInterfaces.Contains(handler, SymbolEqualityComparer.Default)
        )
        {
            return;
        }

        foreach (var parameter in method.Parameters)
        {
            if (SymbolEqualityComparer.Default.Equals(parameter.Type, httpContext))
            {
                context.ReportDiagnostic(
                    Diagnostic.Create(Rule, parameter.Locations[0], method.Name)
                );
            }
        }
    }
}
```

- Prefix the ids with the project's own letters (`<PREFIX>0001`); write the message for the person who hits it: what is wrong and what to do instead.
- **Symbols over syntax**: compare `ISymbol`s with `SymbolEqualityComparer.Default`, resolved by metadata name in `RegisterCompilationStartAction`; return early when the types are absent (a project the rule does not apply to).
- `EnableConcurrentExecution()` and `ConfigureGeneratedCodeAnalysis(None)` always.
- Severity `Error` for rules that must hold; `Warning` would become an error anyway under zero tolerance, but `Error` states the intent.

**Wire it into every project that should obey it** (in `Directory.Build.props`, or per project):

```xml
<ProjectReference Include="..\<PROJECT>.Analyzers\<PROJECT>.Analyzers.csproj"
                  OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
```

A violation then fails the build: `error <PREFIX>0001: 'Handle' takes HttpContext; bind the values it needs as parameters instead`.

**Test it** with Roslyn's testing library under TUnit.  `DefaultVerifier` throws on a mismatch, so no xUnit adapter is involved; `{|<PREFIX>0001:...|}` marks where the diagnostic must appear:

`<PROJECT>.Analyzers.Tests/AnalyzerTests.cs`

```csharp
using <PROJECT>.Analyzers;
using Microsoft.CodeAnalysis.CSharp.Testing;
using Microsoft.CodeAnalysis.Testing;

namespace <PROJECT>.Analyzers.Tests;

/// <summary>
/// Analyzer tests with Roslyn's testing library: `DefaultVerifier` throws on a mismatch, so it
/// works under TUnit (no xUnit adapter).  `{|<PREFIX>0001:...|}` marks where the diagnostic must be.
/// </summary>
public sealed class HandlerTakesHttpContextAnalyzerTests
{
    private const string Stubs = """
        namespace Microsoft.AspNetCore.Http { public class HttpContext { } }
        namespace <PROJECT>.Api { public interface IEndpointHandler { } }
        """;

    private static Task VerifyAsync(string source) =>
        new CSharpAnalyzerTest<HandlerTakesHttpContextAnalyzer, DefaultVerifier>
        {
            TestState = { Sources = { Stubs, source } },
        }.RunAsync();

    [Test]
    public Task Handler_TakingHttpContext_IsReported() =>
        VerifyAsync(
            """
            public sealed class Bad : <PROJECT>.Api.IEndpointHandler
            {
                public string Handle(Microsoft.AspNetCore.Http.HttpContext {|<PREFIX>0001:context|}) => "";
            }
            """
        );

    [Test]
    public Task Handler_TakingBoundValues_IsClean() =>
        VerifyAsync(
            """
            public sealed class Good : <PROJECT>.Api.IEndpointHandler
            {
                public string Handle(string title) => title;
            }
            """
        );

    [Test]
    public Task NonHandler_TakingHttpContext_IsNotTheRulesBusiness() =>
        VerifyAsync(
            """
            public sealed class Middleware
            {
                public void Invoke(Microsoft.AspNetCore.Http.HttpContext context) { }
            }
            """
        );
}
```

- Test project packages: `Microsoft.CodeAnalysis.CSharp.Analyzer.Testing` **and** `Microsoft.CodeAnalysis.CSharp.Workspaces` at the analyzer's Roslyn version.  Without the second, the testing package restores Workspaces `1.0.1` (a .NET Framework package) and the build fails on `NU1701`.
- Stub the types the rule looks for (`Stubs`) instead of referencing ASP.NET Core in the test compilation.
- Cover: the violation (with its location), the compliant case, and a near miss the rule must ignore.

## Which tool for which rule

| The rule | Use |
|---|---|
| a formatting or style preference | CSharpier, `.editorconfig` |
| "never call X" | `BannedSymbols.txt` |
| "every X must have Y" about code structure | a custom analyzer |
| "every endpoint is described / secured" (runtime metadata) | a test over the OpenAPI document or the endpoint data source (`openapi-contract.md`, `route-security.md`) |
| a rule that needs judgment | a review checklist, not an analyzer |

## Checklist

1. `TreatWarningsAsErrors` is on and the build is clean; no `#pragma` disables.
2. Every severity override in `.editorconfig` has a comment with its reason.
3. A rule that keeps coming up in review becomes a banned symbol or an analyzer, with tests.
4. Agents: a build error with a rule id is the instruction.  Read the message, fix the cause, and never suppress the rule to finish a task.
