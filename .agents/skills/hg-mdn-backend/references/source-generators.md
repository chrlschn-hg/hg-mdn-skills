# Source Generators: Generate What Is Repetitive

When code is mechanical and repeats (a typed id per entity, a mapping per contract, a registration per handler, a serializer per type), **generate it at compile time** instead of writing it by hand or doing it with reflection at runtime.  Generated code is ordinary C# that the compiler checks, the IDE navigates, and trimming and AOT understand; the source stays terse because only the declaration is written.

**First, use the generators .NET already ships**; write your own only when none fits:

| Repetitive code | Built-in or established generator |
|---|---|
| logging calls on hot paths | `[LoggerMessage]` on a `partial` method (`Microsoft.Extensions.Logging`) |
| regular expressions | `[GeneratedRegex("...")]` on a `partial` method or property: compiled at build, no runtime `Regex` construction |
| JSON serialization | `JsonSerializerContext` with `[JsonSerializable(typeof(T))]` (System.Text.Json) |
| configuration binding | `<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>`: `Bind`/`Get<T>` without reflection |
| options validation | `[OptionsValidator]` on a `partial class : IValidateOptions<T>` (DataAnnotations checked at compile time) |
| minimal API endpoints | `<EnableRequestDelegateGenerator>true</EnableRequestDelegateGenerator>` (on by default when publishing AOT) |
| object-to-object mapping | [Mapperly](https://mapperly.riok.app) (`Riok.Mapperly`): `[Mapper] partial class` with `partial` methods; unmapped members are reported as diagnostics (raise their severity in `.editorconfig` to make them errors) |

Modern C# makes the declaration side clean: `partial` methods (with bodies supplied by a generator), **partial properties** (C# 13), and **partial constructors and events** (C# 14) are the shapes generators fill in.

## When to write one

Write a generator when the same shape is written for the third time and is fully determined by a declaration: strongly typed ids, `ToString`/equality for a family of types, registration tables (replacing an assembly scan), enum-to-string maps, test builders.  Do **not** generate business logic, or anything a generic method or a base class expresses in a few lines: a generator is a small compiler extension to maintain.

## A worked example: strongly typed ids

The declaration is one line; the generator supplies the UUID v7 value, `New()`, `Parse`, and `ToString`:

```csharp
[<PROJECT>.TypedId]
public readonly partial record struct NoteId;

var id = NoteId.New();   // a time-ordered UUID v7, typed: a NoteId cannot be passed where an OrderId is expected
```

**The project** (`src/backend/packages/<PROJECT>.Generators`): the same shape as an analyzer project (`static-analysis.md`): `netstandard2.0`, `IsRoslynComponent`, Roslyn `5.0.0` for the .NET 10 SDK.  `netstandard2.0` has no `IsExternalInit`, so records need a polyfill:

`<PROJECT>.Generators/IsExternalInit.cs`

```csharp
// netstandard2.0 lacks this type; records and init-only properties need it.
namespace System.Runtime.CompilerServices;

internal static class IsExternalInit;
```

**The generator**: always `IIncrementalGenerator` (the older `ISourceGenerator` is deprecated: it reruns everything on every edit):

`<PROJECT>.Generators/TypedIdGenerator.cs`

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text;

namespace <PROJECT>.Generators;

/// <summary>
/// Generates a strongly typed id for <c>[TypedId] public readonly partial record struct NoteId;</c>:
/// a UUID v7 value with <c>New()</c>, <c>Parse</c>, and <c>ToString</c>.
/// </summary>
[Generator(LanguageNames.CSharp)]
public sealed class TypedIdGenerator : IIncrementalGenerator
{
    private const string AttributeName = "<PROJECT>.TypedIdAttribute";

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 1. The marker attribute ships with the generator.
        context.RegisterPostInitializationOutput(output =>
            output.AddSource(
                "TypedIdAttribute.g.cs",
                """
                namespace <PROJECT>;

                /// <summary>Generates a UUID v7 strongly typed id for this partial record struct.</summary>
                [global::System.AttributeUsage(global::System.AttributeTargets.Struct)]
                internal sealed class TypedIdAttribute : global::System.Attribute;
                """
            )
        );

        // 2. Find the marked types.  ForAttributeWithMetadataName is the fast path: the compiler
        //    only looks at syntax carrying the attribute.  Project to a small equatable model
        //    (strings), never symbols, so unchanged inputs are cached.
        var ids = context.SyntaxProvider.ForAttributeWithMetadataName(
            AttributeName,
            predicate: static (node, _) => node is not null,
            transform: static (attribute, _) =>
                new TypedIdModel(
                    attribute.TargetSymbol.ContainingNamespace.IsGlobalNamespace
                        ? null
                        : attribute.TargetSymbol.ContainingNamespace.ToDisplayString(),
                    attribute.TargetSymbol.Name
                )
        );

        // 3. Emit one file per id.
        context.RegisterSourceOutput(ids, static (output, id) =>
            output.AddSource($"{id.Name}.g.cs", SourceText.From(Render(id), Encoding.UTF8))
        );
    }

    private static string Render(TypedIdModel id) =>
        $$"""
        // <auto-generated/>
        #nullable enable
        {{(id.Namespace is null ? "" : $"namespace {id.Namespace};")}}

        readonly partial record struct {{id.Name}}(global::System.Guid Value)
        {
            /// <summary>A new time-ordered id.</summary>
            public static {{id.Name}} New() => new(global::System.Guid.CreateVersion7());

            public static {{id.Name}} Parse(string value) => new(global::System.Guid.Parse(value));

            public override string ToString() => Value.ToString();
        }
        """;

    private sealed record TypedIdModel(string? Namespace, string Name);
}
```

The rules that keep a generator fast (the IDE runs it on every keystroke):

- **`ForAttributeWithMetadataName`** to find marked types: the compiler only visits syntax that carries the attribute.  Never scan all syntax nodes with `CreateSyntaxProvider` when an attribute can mark the target.
- **Project to a small, equatable model** (a `record` of strings and values) in `transform`.  Never put `ISymbol`, `SyntaxNode`, or `Compilation` in the pipeline's output: they are not equatable across edits, so nothing is cached and the generator reruns on every change.  For collections in the model use an equatable wrapper, not `ImmutableArray` (reference equality).
- **The marker attribute comes from `RegisterPostInitializationOutput`**, so the consuming project needs no extra reference.
- **Fully qualified names** (`global::System.Guid`) in generated code: it is compiled in the consumer's namespace context.
- Mark the output `// <auto-generated/>` and enable nullable explicitly (`#nullable enable`).

**Wire it in** like an analyzer:

```xml
<ProjectReference Include="..\<PROJECT>.Generators\<PROJECT>.Generators.csproj"
                  OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
```

**See what it generated**: `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>` writes the files to `obj/<Configuration>/<tfm>/generated/<Generator.Assembly>/<Generator.Type>/` (here `NoteId.g.cs`); the IDE also shows them under Dependencies > Analyzers.

## Testing a generator

Run it with `CSharpGeneratorDriver` against an in-memory compilation (reference assemblies from `Basic.Reference.Assemblies.Net100`), under TUnit:

`<PROJECT>.Analyzers.Tests/GeneratorTests.cs`

```csharp
using <PROJECT>.Generators;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;

namespace <PROJECT>.Analyzers.Tests;

/// <summary>Generator tests: run the generator with a driver and inspect the output.</summary>
public sealed class TypedIdGeneratorTests
{
    private static CSharpCompilation Compile(string source) =>
        CSharpCompilation.Create(
            "Probe",
            [CSharpSyntaxTree.ParseText(source)],
            Basic.Reference.Assemblies.Net100.References.All,
            new CSharpCompilationOptions(OutputKind.DynamicallyLinkedLibrary)
        );

    [Test]
    public async Task Generates_ACompilingTypedId()
    {
        var compilation = Compile(
            """
            namespace Shop;

            [<PROJECT>.TypedId]
            public readonly partial record struct OrderId;

            public static class Use
            {
                public static OrderId Make() => OrderId.New();
            }
            """
        );
        GeneratorDriver driver = CSharpGeneratorDriver.Create(new TypedIdGenerator());

        driver = driver.RunGeneratorsAndUpdateCompilation(compilation, out var output, out _);

        var generated = driver.GetRunResult().GeneratedTrees.Select(t => t.FilePath).ToList();
        var errors = output.GetDiagnostics().Where(d => d.Severity == DiagnosticSeverity.Error);

        await Assert.That(generated.Any(path => path.EndsWith("OrderId.g.cs", StringComparison.Ordinal))).IsTrue();
        await Assert.That(errors).IsEmpty();
    }

    [Test]
    public async Task UnrelatedEdits_AreServedFromTheCache()
    {
        var compilation = Compile("[<PROJECT>.TypedId] public readonly partial record struct NoteId;");
        GeneratorDriver driver = CSharpGeneratorDriver.Create(
            [new TypedIdGenerator().AsSourceGenerator()],
            driverOptions: new GeneratorDriverOptions(
                IncrementalGeneratorOutputKind.None,
                trackIncrementalGeneratorSteps: true
            )
        );
        driver = driver.RunGenerators(compilation);

        // Add an unrelated file: the marked type did not change, so its output is cached.
        var edited = compilation.AddSyntaxTrees(CSharpSyntaxTree.ParseText("class Other { }"));
        driver = driver.RunGenerators(edited);

        var outputs = driver
            .GetRunResult()
            .Results[0]
            .TrackedOutputSteps.SelectMany(step => step.Value)
            .SelectMany(step => step.Outputs)
            .ToList();

        await Assert.That(outputs).IsNotEmpty(); // the check below must not pass vacuously
        await Assert
            .That(outputs.All(o => o.Reason is IncrementalStepRunReason.Cached or IncrementalStepRunReason.Unchanged))
            .IsTrue();
    }
}
```

- **The output compiles**: no error diagnostics in the updated compilation, and the expected file exists.
- **Caching works**: rerun after an unrelated edit with `trackIncrementalGeneratorSteps: true` and assert every output step is `Cached` or `Unchanged`.  This test catches the most common generator bug (a symbol in the model) before it slows the IDE.  It asserts the outputs are not empty, so it cannot pass vacuously.
- For larger generators, snapshot-test the generated text (Verify with `Verify.SourceGenerators`).

## Checklist

1. A built-in generator does not already cover it.
2. `IIncrementalGenerator`, `ForAttributeWithMetadataName`, an equatable model, fully qualified generated code.
3. The marker attribute is post-initialization output.
4. Tests: the output compiles, and unrelated edits are cached.
5. The generated code passes the same zero-warning build as handwritten code.
