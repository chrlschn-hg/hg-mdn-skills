# TUnit Reference

The TUnit pieces beyond the everyday `[Test]` and `await value.Should()...`: data sources, hooks, parallelism, assertions, and the xUnit equivalents.  Suite layout, run commands, and fakes: `testing.md`.  Read <https://tunit.dev/llms.txt> before writing unusual tests (data sources, lifecycle, dependency injection, retries).

## Tour

`src/backend/packages/<PROJECT>.Notes.Tests/TUnitProbeTests.cs`

```csharp
namespace <PROJECT>.Notes.Tests;

[Property("Category", "Smoke")]
public sealed class TUnitProbeTests
{
    private readonly List<string> _log = [];

    // Runs before each test (a new class instance is created per test).
    [Before(Test)]
    public void Setup() => _log.Add("setup");

    [Test]
    [Arguments("a", 1)]
    [Arguments("bb", 2)]
    public async Task Length_InputString_ReturnsExpectedLength(string input, int expected)
    {
        await input.Length.Should().BeEqualTo(expected);
    }

    [Test]
    [MethodDataSource(nameof(Titles))]
    public async Task Trim_WhitespaceAroundText_ReturnsTrimmedText(string raw, string expected)
    {
        await raw.Trim().Should().BeEqualTo(expected);
    }

    public static IEnumerable<(string, string)> Titles()
    {
        yield return ("  x ", "x");
        yield return ("y", "y");
    }

    [Test]
    public async Task Assertions_ChainedAndGrouped_Pass()
    {
        var value = 3;

        await value.Should().BeGreaterThan(0).And.BeEqualTo(3);

        using (Assert.Multiple())
        {
            await value.Should().BeGreaterThan(1);
            await _log.Should().Contain("setup");
        }
    }

    [Test]
    public async Task Throw_SyncAndAsyncDelegates_MatchTypeAndMessage()
    {
        Action parse = () => int.Parse("nope");
        Func<Task> fail = async () =>
            await Task.FromException(new InvalidOperationException("boom"));

        await parse.Should().Throw<FormatException>();
        await fail.Should().Throw<InvalidOperationException>().WithMessageContaining("boom");
    }

    [Test]
    [Explicit]
    public async Task ExplicitTest_NameFilteredRun_Runs()
    {
        // [Explicit]: skipped by a normal run; run it by filtering for it by name.
        var ran = true;

        await ran.Should().BeTrue();
    }

    [Test]
    [NotInParallel]
    public async Task NotInParallelAttribute_MarkedTest_Runs()
    {
        var ran = true;

        await ran.Should().BeTrue();
    }
}
```

- `[Test]` covers both `[Fact]` and `[Theory]`.  Parameterize with `[Arguments(...)]` (the workhorse for table-driven cases) or `[MethodDataSource(nameof(...))]`; also `[ClassDataSource<T>]`, `[Matrix]`, `[CombinedDataSources]`.
- Hooks: `[Before(Test)]` / `[After(Test)]` per test, `[Before(Class)]` / `[After(Class)]`, `[Before(Assembly)]`, `[Before(TestSession)]`.  Hooks may be async.
- `[Property("Category", "Smoke")]` tags a class or test; filter with `[Category=Smoke]`.
- Tests run in parallel by default.  `[NotInParallel]` serializes a test (shared global state); `[ParallelLimiter<T>]` caps concurrency.
- `.And` / `.Or` cannot be mixed in one chain (build error `TUnitAssertions0001`); split into separate assertions.  A `Should()` chain that is not awaited is `TUnitAssertions0002`.
- `Assert.Multiple()` collects every failure instead of stopping at the first.
- Skipping: `[Skip("reason")]`; subclass `SkipAttribute` for conditional skips (for example infrastructure tests that cannot run in CI).
- `[Explicit]` excludes a test from normal runs; it runs only when a filter targets it by name (`--treenode-filter "/*/*/*/ExplicitTest_NameFilteredRun_Runs"`).  Use it for slow, costly, or environment-dependent checks (a probe against a live service).
- `TestContext` as a method parameter gives output (`context.Output.WriteLine`), the test name, and properties.

## Assertions

Tests assert with the fluent **`Should()`** syntax: `await value.Should().BeEqualTo(x)`.  It comes from the optional `TUnit.Assertions.Should` package, which is published only as a **beta** (`1.84.0-beta`) and generates each method from a TUnit assertion by renaming it: `Is*` becomes `Be*`, `IsNot*` `NotBe*`, `Has*` `Have*`, `DoesNotContain` `NotContain`, `Contains` `Contain`, `Throws` `Throw`.  The scaffold's `Directory.Build.targets` adds the package and its two namespaces (`TUnit.Assertions.Should`, `TUnit.Assertions.Should.Extensions`) to every project that references `TUnit`; a project that adds it by hand needs both namespaces as usings.

Docs: [Should syntax](https://tunit.dev/docs/assertions/should-syntax) (this page), [equality and comparison](https://tunit.dev/docs/assertions/equality-and-comparison), [combining](https://tunit.dev/docs/assertions/combining-assertions), [exceptions](https://tunit.dev/docs/assertions/exceptions) (these four use the underlying `Assert.That(...)` names; apply the renaming above).

```csharp
await result.Should().BeEqualTo(10);
await score.Should().BeGreaterThan(70);
await percentage.Should().BeBetween(0, 100);
await actual.Should().BeCloseTo(expected, 0.001); // double/float/decimal tolerance
await instance.Should().BeSameReferenceAs(other); // reference, not value, equality
await notes
    .Select(n => n.Title)
    .Should()
    .BeEquivalentTo(["newer", "older"], CollectionOrdering.Matching); // collections, in this order

Action parse = () => int.Parse("x");
Action validate = () => Validate(null!);
Func<Task> failing = async () => await FailingAsync();

await parse.Should().Throw<FormatException>(); // the type or a subclass; ThrowExactly<T>() for the exact type
await validate
    .Should()
    .Throw<ArgumentNullException>()
    .WithMessageContaining("(Parameter 'user')");
await failing.Should().Throw<HttpRequestException>();
await notes.Should().HaveSingleItem();
await note.Title.Should().BeEqualTo("Hello").Because("the title is trimmed, not rewritten");
```

**`BeEquivalentTo` ignores order by default**: an assertion that a list is sorted passes for any order.  Add `CollectionOrdering.Matching` (`using TUnit.Assertions.Enums;`) whenever order is the point; a test of sorting that cannot fail is worse than none.

`BeEqualTo` uses `Equals`/`==`: records and structs get value equality for free.  Message matchers (after `Throw<T>()`): `WithMessage` (exact), `WithMessageContaining`, `WithMessageNotContaining`, `WithMessageMatching` (a wildcard pattern).  There is no `WithParameterName`: match `(Parameter 'name')` in the message.

**Not available on `Should()`**: `IsPositive()` (use `BeGreaterThan(0)`), `IsEqualTo(x).Within(t)` (use `BeCloseTo(x, t)`), and the generic `IsTypeOf<T>()` (use `BeOfType(typeof(T))`).  For anything missing, `Assert.That(...)` still works in the same file.

## xUnit to TUnit

| xUnit | TUnit |
|---|---|
| `[Fact]` / `[Theory]` | `[Test]` |
| `[InlineData(...)]` | `[Arguments(...)]` |
| `[MemberData(nameof(...))]` | `[MethodDataSource(nameof(...))]` |
| `[Trait("key","value")]` | `[Property("key","value")]` |
| constructor / `IDisposable` | constructor / `[Before(Test)]` / `[After(Test)]` |
| `IAsyncLifetime` | `[Before(Test)]` / `[After(Test)]` (async-native) |
| `IClassFixture<T>` | `[ClassDataSource<T>(Shared = SharedType.PerClass)]` |
| shared across the run | `[ClassDataSource<T>(Shared = SharedType.PerTestSession)]` |
| `Assert.Equal(expected, actual)` | `await actual.Should().BeEqualTo(expected)` |
| `Assert.Throws<T>(() => ...)` | `Action act = () => ...;` then `await act.Should().Throw<T>()` |
| `ITestOutputHelper` | `TestContext` method parameter, `.Output` |
| `dotnet test --filter "Category=X"` | `--treenode-filter "/*/*/*/*[Category=X]"` |
