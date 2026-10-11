# TUnit Reference

The TUnit pieces beyond the everyday `[Test]` and `await Assert.That(...)`: data sources, hooks, parallelism, assertions, and the xUnit equivalents.  Suite layout, run commands, and fakes: `testing.md`.  Read <https://tunit.dev/llms.txt> before writing unusual tests (data sources, lifecycle, dependency injection, retries).

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
    public async Task Length_OfInput_MatchesExpected(string input, int expected)
    {
        await Assert.That(input.Length).IsEqualTo(expected);
    }

    [Test]
    [MethodDataSource(nameof(Titles))]
    public async Task Trim_RemovesWhitespace(string raw, string expected)
    {
        await Assert.That(raw.Trim()).IsEqualTo(expected);
    }

    public static IEnumerable<(string, string)> Titles()
    {
        yield return ("  x ", "x");
        yield return ("y", "y");
    }

    [Test]
    public async Task Assertions_CombineAndCollect()
    {
        var value = 3;

        await Assert.That(value).IsPositive().And.IsEqualTo(3);

        using (Assert.Multiple())
        {
            await Assert.That(value).IsGreaterThan(1);
            await Assert.That(_log).Contains("setup");
        }
    }

    [Test]
    public async Task Exceptions_AreAsserted()
    {
        await Assert.That(() => int.Parse("nope")).Throws<FormatException>();
        await Assert
            .That(async () => await Task.FromException(new InvalidOperationException("boom")))
            .Throws<InvalidOperationException>()
            .WithMessageContaining("boom");
    }

    [Test]
    [Explicit]
    public async Task Explicit_Test_RunsOnlyWhenTargeted()
    {
        // [Explicit]: skipped by a normal run; run it by filtering for it by name.
        var ran = true;

        await Assert.That(ran).IsTrue();
    }

    [Test]
    [NotInParallel]
    public async Task Serial_Test()
    {
        var ran = true;

        await Assert.That(ran).IsTrue();
    }
}
```

- `[Test]` covers both `[Fact]` and `[Theory]`.  Parameterize with `[Arguments(...)]` (the workhorse for table-driven cases) or `[MethodDataSource(nameof(...))]`; also `[ClassDataSource<T>]`, `[Matrix]`, `[CombinedDataSources]`.
- Hooks: `[Before(Test)]` / `[After(Test)]` per test, `[Before(Class)]` / `[After(Class)]`, `[Before(Assembly)]`, `[Before(TestSession)]`.  Hooks may be async.
- `[Property("Category", "Smoke")]` tags a class or test; filter with `[Category=Smoke]`.
- Tests run in parallel by default.  `[NotInParallel]` serializes a test (shared global state); `[ParallelLimiter<T>]` caps concurrency.
- `.And` / `.Or` chains cannot be mixed in one chain (`MixedAndOrAssertionsException`); split into separate `Assert.That` calls.
- `Assert.Multiple()` collects every failure instead of stopping at the first.
- Skipping: `[Skip("reason")]`; subclass `SkipAttribute` for conditional skips (for example infrastructure tests that cannot run in CI).
- `[Explicit]` excludes a test from normal runs; it runs only when a filter targets it by name (`--treenode-filter "/*/*/*/Explicit_Test_RunsOnlyWhenTargeted"`).  Use it for slow, costly, or environment-dependent checks (a probe against a live service).
- `TestContext` as a method parameter gives output (`context.Output.WriteLine`), the test name, and properties.

## Assertions

Docs: [getting started](https://tunit.dev/docs/assertions/getting-started), [equality and comparison](https://tunit.dev/docs/assertions/equality-and-comparison), [combining](https://tunit.dev/docs/assertions/combining-assertions), [exceptions](https://tunit.dev/docs/assertions/exceptions).

```csharp
await Assert.That(result).IsEqualTo(10);
await Assert.That(score).IsGreaterThan(70);
await Assert.That(percentage).IsBetween(0, 100);
await Assert.That(actual).IsEqualTo(expected).Within(0.001);     // double/float/decimal tolerance
await Assert.That(instance).IsSameReferenceAs(other);            // reference, not value, equality
await Assert.That(notes.Select(n => n.Title)).IsEquivalentTo(["newer", "older"], CollectionOrdering.Matching);   // collections, in this order

await Assert.That(() => int.Parse("x")).Throws<FormatException>();                 // exact type or subclass
await Assert.That(() => Validate(null!)).Throws<ArgumentNullException>().WithParameterName("user");
await Assert.That(async () => await FailingAsync()).Throws<HttpRequestException>();
```

**`IsEquivalentTo` ignores order by default**: an assertion that a list is sorted passes for any order.  Add `CollectionOrdering.Matching` (`using TUnit.Assertions.Enums;`) whenever order is the point; a test of sorting that cannot fail is worse than none.

`IsEqualTo` uses `Equals`/`==`: records and structs get value equality for free.  Message matchers: `WithMessage` (exact), `WithMessageContaining`, `WithMessageNotContaining`, `WithMessageMatching` (regex), `.IgnoringCase()`.

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
| `Assert.Equal(expected, actual)` | `await Assert.That(actual).IsEqualTo(expected)` |
| `Assert.Throws<T>(() => ...)` | `await Assert.That(() => ...).Throws<T>()` |
| `ITestOutputHelper` | `TestContext` method parameter, `.Output` |
| `dotnet test --filter "Category=X"` | `--treenode-filter "/*/*/*/*[Category=X]"` |
