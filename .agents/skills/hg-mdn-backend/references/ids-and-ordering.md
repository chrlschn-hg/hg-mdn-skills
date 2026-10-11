# Public Ids and User Ordering

Two additions to a table that already has a UUID v7 key.  Neither replaces it: the UUID stays the primary key and the target of every foreign key (`data-efcore.md`).

| Need | Add | Library |
|---|---|---|
| People read, type, or share the id (URLs, tickets, order numbers) | a short, unique **public id** column | `Nanoid` |
| Users reorder rows and a move must not rewrite the list | a string **rank** column | `Innoloft.LexoRank` |

Both are optional and per table: add them when the table needs them, not by default.

## Public ids: a short NanoID next to the UUID

A UUID is a poor thing to show: 36 characters, unreadable aloud, awkward in a URL.  Keep it internal and add a **second, unique column** for what people see.  It is never a key and never a foreign key target.

- **Library:** [`Nanoid`](https://github.com/codeyu/nanoid-net) (MIT): `Nanoid.Generate(alphabet, size)`.  `dotnet add package Nanoid`; with central package management add the version to `Directory.Packages.props`.
- **Use a custom alphabet, not the default.**  The default mixes cases, digits, `_` and `-`: hard to read aloud, case-sensitive, and it can spell words.  Use **safe lowercase alphanumeric** (`23456789bcdfghjkmnpqrstvwxz`, 27 characters): digits `2`-`9` and lowercase consonants.  No vowels, so an id cannot spell a word; no lookalikes (`0`/`o`, `1`/`l`/`i`, and no `y`); no capitals, so it is not case-sensitive.
- **Length:** 12 characters from 27 is 27^12 (about 1.5e17).  The chance that a new id already exists is about `rows / 1.5e17`, so even a billion rows collide rarely.  Longer ids cost URL length, shorter ones cost retries (10 characters is 2e14: fine below about a hundred million rows).
- **Generate in one helper** (below), in a shared project once a second slice needs it.  Check the shape at the edge (`IsValid`) before querying: a route value that cannot be an id is a 404.

`src/backend/packages/<PROJECT>.Notes/PublicId.cs`

```csharp
using System.Buffers;
using NanoidDotNet;

namespace <PROJECT>.Notes;

/// <summary>
/// The short, user-facing id: an extra unique column next to the internal UUID v7 key, for URLs,
/// support tickets, and anything a person reads or types.  The UUID stays the primary key and the
/// target of foreign keys; this id is never one.
/// </summary>
/// <remarks>
/// Safe lowercase alphanumeric: digits 2-9 and consonants, without the lookalikes (`0`/`o`,
/// `1`/`l`/`i`) and without vowels, so an id is easy to read aloud and cannot spell a word.  12
/// characters from 27 is 27^12 (about 1.5e17) values: the chance that a new id already exists is
/// rows / 1.5e17, so a unique index with a retry is the guarantee and a collision is a rare event,
/// not a design problem.  Keep this in one shared project once a second slice needs it.
/// </remarks>
public static class PublicId
{
    public const string Alphabet = "23456789bcdfghjkmnpqrstvwxz";

    public const int Length = 12;

    private static readonly SearchValues<char> AlphabetChars = SearchValues.Create(Alphabet);

    /// <summary>A new random id (cryptographically strong; no coordination needed).</summary>
    public static string New() => Nanoid.Generate(Alphabet, Length);

    /// <summary>
    /// True when `value` has the shape of an id.  Check it at the edge (a route value) before
    /// querying: anything else cannot exist.
    /// </summary>
    public static bool IsValid(string? value) =>
        value is { Length: Length } && value.AsSpan().IndexOfAnyExcept(AlphabetChars) < 0;

    /// <summary>
    /// Runs `insertAsync` with a fresh id; when it fails with a collision on the id's unique index
    /// (`isCollision` says so), tries again with another id, up to `attempts` times.
    /// </summary>
    public static async Task<T> InsertWithFreshIdAsync<T>(
        Func<string, Task<T>> insertAsync,
        Func<Exception, bool> isCollision,
        int attempts = 3,
        Func<string>? newId = null
    )
    {
        newId ??= New;

        for (var attempt = 1; ; attempt++)
        {
            try
            {
                return await insertAsync(newId());
            }
            catch (Exception ex) when (attempt < attempts && isCollision(ex))
            {
                // Another id will do: loop.
            }
        }
    }
}
```

**Uniqueness comes from a unique index, not from the generator.**  Generation is random and needs no coordination; the index decides, and the insert is retried with another id:

`src/backend/packages/<PROJECT>.Data.Tests/CardsProbeModel.cs`

```csharp
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;

namespace <PROJECT>.Data.Tests;

/// <summary>
/// A row with three identities: the internal UUID v7 key, the short public id for URLs and
/// support, and a rank for user ordering.
/// </summary>
public sealed class Card
{
    /// <summary>Internal: the primary key and the target of foreign keys.  Time-ordered.</summary>
    public Guid Id { get; set; } = Guid.CreateVersion7();

    /// <summary>External: unique, short, safe to read aloud (<see cref="PublicId"/>).</summary>
    public required string PublicId { get; set; }

    /// <summary>Position among its siblings (<see cref="RowRank"/>); sorted ordinally.</summary>
    public required string Rank { get; set; }

    public required string Title { get; set; }
}

/// <summary>A test-only model of the id and ordering rules (its own schema, like the tenancy probe).</summary>
internal sealed class CardsProbeContext(DbContextOptions<CardsProbeContext> options)
    : DbContext(options)
{
    public const string PublicIdIndex = "ix_cards_public_id";

    protected override void OnModelCreating(ModelBuilder model)
    {
        model.HasDefaultSchema("cards_probe");

        model.Entity<Card>(card =>
        {
            card.HasKey(c => c.Id);

            // The uniqueness guarantee for the public id: generation may collide (rarely); the
            // index decides, and the insert is retried with another id.
            card.Property(c => c.PublicId).HasMaxLength(PublicId.Length);
            card.HasIndex(c => c.PublicId).IsUnique().HasDatabaseName(PublicIdIndex);

            // Ordinal collation: the database orders ranks byte by byte, exactly as the rank
            // library compares them, whatever the database's locale.  Hard length cap: a rank
            // this long means a rebalance was missed.
            card.Property(c => c.Rank).HasMaxLength(128).UseCollation("C");

            // In a real model the index is per list: (ListId, Rank) (and (TenantId, ListId, Rank)).
            card.HasIndex(c => c.Rank);
        });
    }
}
```

`src/backend/packages/<PROJECT>.Data.Tests/CardsProbeStore.cs`

```csharp
using <PROJECT>.Notes;
using Microsoft.EntityFrameworkCore;
using Npgsql;

namespace <PROJECT>.Data.Tests;

/// <summary>
/// What a store does with the three identities: inserts with a fresh public id (retrying a
/// collision), moves a row by computing one rank, and lists in order.
/// </summary>
internal static class CardsProbeStore
{
    /// <summary>The database error for a duplicate public id: what the retry reacts to.</summary>
    public static bool IsPublicIdCollision(Exception ex) =>
        ex
            is DbUpdateException
            {
                InnerException: PostgresException
                {
                    SqlState: PostgresErrorCodes.UniqueViolation,
                    ConstraintName: CardsProbeContext.PublicIdIndex,
                },
            };

    /// <summary>Adds a card with a fresh public id; a collision on the unique index is retried.</summary>
    public static Task<Card> AddAsync(
        CardsProbeContext db,
        string title,
        string rank,
        Func<string>? newId = null
    ) =>
        PublicId.InsertWithFreshIdAsync(
            async id =>
            {
                var card = new Card
                {
                    PublicId = id,
                    Rank = rank,
                    Title = title,
                };
                db.Add(card);

                try
                {
                    await db.SaveChangesAsync();

                    return card;
                }
                catch
                {
                    // Forget the failed insert, or the retry would try to save it again.
                    db.Entry(card).State = EntityState.Detached;

                    throw;
                }
            },
            IsPublicIdCollision,
            newId: newId
        );

    /// <summary>
    /// Moves a card between its new neighbors (null: the start or the end of the list): one new
    /// rank, one row updated (returns the number of rows written).  The caller passes the
    /// neighbors' ranks, never a position.
    /// </summary>
    public static async Task<int> MoveAsync(
        CardsProbeContext db,
        Card card,
        string? beforeRank,
        string? afterRank
    )
    {
        card.Rank = RowRank.Between(beforeRank, afterRank);

        return await db.SaveChangesAsync();
    }

    /// <summary>The list in order: by rank (ordinal), then by the time-ordered id for equal ranks.</summary>
    public static Task<List<Card>> ListAsync(CardsProbeContext db) =>
        db.Set<Card>().AsNoTracking().OrderBy(c => c.Rank).ThenBy(c => c.Id).ToListAsync();
}
```

- Recognize the collision by SQLSTATE `23505` **and the index name**, so an unrelated unique violation is not retried.
- A failed insert must be **detached** before the retry, or the next `SaveChanges` tries to save it again.
- Inside a surrounding transaction EF wraps `SaveChanges` in a savepoint, so a collision does not poison the transaction and the retry works there too (the test pins it).
- **Multi-tenant:** make the unique index `(TenantId, PublicId)`: ids only need to be unique within a tenant (`data-efcore.md`, "Multi-tenant data").
- **At the API**, routes and responses use the public id; the UUID stays out of contracts (`endpoints.md`).  A random id is not enumerable but it is not a secret either: authorize every request as usual.

## Reordering rows: LexoRank

Integer positions mean a move rewrites every row between the old and the new place.  **LexoRank** gives each row a string rank that sorts lexicographically; a move computes **one** rank between the row's new neighbors and updates **one** row, an O(1) operation however long the list.

- **Library:** [`Innoloft.LexoRank`](https://github.com/innoloft/lexorank) (MIT, targets .NET 10), an implementation of Atlassian's algorithm.  It is a small, young project: pin the version and keep it behind **one helper** (below), so replacing it touches one file.  Avoid the older `LexoRank` package (last release 2019, no license).
- A rank looks like `0|hzzzzz:`.  `Between(before, after)` takes the neighbors' ranks, `null` for the start or the end of the list.

`src/backend/packages/<PROJECT>.Notes/RowRank.cs`

```csharp
using Rank = LexoRank.LexoRank;

namespace <PROJECT>.Notes;

/// <summary>
/// Rank strings for rows a user can reorder (LexoRank).  Moving a row computes one new rank
/// between its new neighbors and updates that one row: O(1), no other row is rewritten.
/// Sort by the rank column with an ordinal collation, then by the row's id.
/// </summary>
/// <remarks>
/// The only place that touches the LexoRank library, so it can be swapped.  A rank looks like
/// `0|hzzzzz:`.  Ranks stay short unless inserts keep landing between the same two neighbors; then
/// they grow, and <see cref="NeedsRebalance"/> says when to rewrite the list's ranks with
/// <see cref="Spread"/> (a rare, per-list operation).
/// </remarks>
public static class RowRank
{
    /// <summary>A rank longer than this means the list should be rebalanced.</summary>
    public const int RebalanceAt = 32;

    /// <summary>The rank for the first row of an empty list.</summary>
    public static string First() => Rank.Middle().ToString();

    /// <summary>
    /// A rank that sorts between `before` (null: the start of the list) and `after` (null: the
    /// end).  Pass the ranks of the row's new neighbors.
    /// </summary>
    public static string Between(string? before, string? after) =>
        Rank.CalculateBetween(before, after).ToString();

    /// <summary>True when `rank` has grown long enough that the list should be rebalanced.</summary>
    public static bool NeedsRebalance(string rank) => rank.Length > RebalanceAt;

    /// <summary>
    /// `count` evenly spaced ranks, in order: seeds a list, or rebalances one (assign them to the
    /// rows in their current order, in one transaction).
    /// </summary>
    public static IReadOnlyList<string> Spread(int count)
    {
        var ranks = new List<string>(count);

        Fill(Rank.Min(), Rank.Max(), count, ranks);

        return ranks;
    }

    // Recursive midpoints: even spacing, so later inserts have room before ranks grow.
    private static void Fill(Rank low, Rank high, int count, List<string> into)
    {
        if (count == 0)
        {
            return;
        }

        var middle = low.Between(high);
        var left = (count - 1) / 2;

        Fill(low, middle, left, into);
        into.Add(middle.ToString());
        Fill(middle, high, count - 1 - left, into);
    }
}
```

**The column:**

- **Ordinal collation** (`UseCollation("C")` in EF): the database compares bytes, exactly as the library does, whatever the database's locale.  Under a locale collation (the default `en_US.utf8`) `a` sorts before `B`; ordinal puts `B` first.  Ranks from the library are lowercase base36 plus `|` and `:`, and 3,400 generated ranks sorted identically under both, so this guards against a different locale or a library change rather than a bug seen in practice.
- **Index `(ListId, Rank)`** (and `(TenantId, ListId, Rank)` when multi-tenant); read with `ORDER BY rank, id`.  The row's UUID v7 id **breaks ties**: two clients dropping into the same gap at once get equal ranks, the order stays stable, and a later move separates them.
- A hard length cap (`HasMaxLength(128)`) turns a forgotten rebalance into a loud error instead of silent growth.

**The move:**

- The **client sends the new neighbors** (`beforeId`, `afterId`: public ids), never a rank or an index.  The server loads the two ranks (checking both belong to the same list and tenant), computes the new one, and updates the moved row only.  Return the moved item with its new rank, or the new order.
- First row of an empty list: `First()`.  Append: `Between(lastRank, null)`.  Prepend: `Between(null, firstRank)`.

**Rebalance:** ranks stay short unless inserts keep landing between the same two neighbors (for example, always dropping an item at the top).  Measured: a rank reached 410 characters after about 2,000 such inserts.  When a move returns `NeedsRebalance(rank)` (over 32 characters), rewrite that **one list**: `Spread(count)` assigned to its rows in their current order, in one transaction.  It is rare and bounded by the list's size.

**Do not use it** for a small fixed set, or when the order is a field (a date, a name): sort by the field.

## What the tests pin

`PublicIdTests` and `RowRankTests` (no database), `CardsProbeTests` (real Postgres; the probe model is test-only, in its own schema, created inside the test's transaction):

| Test | Proves |
|---|---|
| `New_HasTheConfiguredShape`, `..NeverContainsAVowel..`, `..NeverContainsALookalike..`, `..UsesBothDigitsAndLetters`, `..IsUniqueAcrossAManySamples` | length, alphabet, no vowels, no `0 1 o l i y`, digits and letters both appear, 50,000 samples without a duplicate |
| `IsValid_AcceptsDigitsAndLetters`, `IsValid_RejectsAnythingElse` | the shape check: wrong length, capitals, a vowel, a lookalike, a symbol, `null` |
| `InsertWithFreshId_*` | a collision retries with another id; gives up after the last attempt; other failures are not retried |
| `PublicId_IsUniqueByIndex_AndTheCollisionIsRecognizable` | the unique index rejects a duplicate, and the predicate recognizes it (SQLSTATE and index name) |
| `Add_RetriesWhenTheUniqueIndexRejectsTheId` | a real collision, then a retry that succeeds, inside a transaction |
| `Moving_ARow_UpdatesExactlyThatRow_AndTheOrderFollows` | a move writes one row and the list order follows |
| `Ranks_AreComparedOrdinally_NotByTheDatabasesLocale` | with `UseCollation("C")` `B` sorts before `a`; without it the test fails |
| `EqualRanks_AreOrderedByTheV7Id_SoTheOrderIsStable` | equal ranks fall back to the time-ordered id |
| `Spread_*`, `Between_*`, `InsertingAtTheSameSpotAgainAndAgain_*` | even short ranks; neighbors respected; growth at a hot spot, then a rebalance that shortens them |

**Assert order with `IsEquivalentTo(expected, CollectionOrdering.Matching)`.**  Plain `IsEquivalentTo` ignores order, so an assertion that "the list is in order" passes for any order (`tunit.md`).
