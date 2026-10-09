# Concurrency

Default to `async`/`await`; I/O-bound work (HTTP, database) needs nothing more.  Use the tools below only for a measured reason.  The template has no concurrency code of its own; these are the shapes to copy when you need it.

## Choosing

| Situation | Use |
|---|---|
| I/O-bound, one thing after another | `await` |
| Many independent items, bounded parallelism | `Parallel.ForEachAsync` with `MaxDegreeOfParallelism` and the `CancellationToken` |
| Several producers, one writer (or one `DbContext`) | a **channel**: producers write, a single consumer merges |
| Shared mutable state that cannot be avoided | `Interlocked`, a concurrent collection, or a `Lock` |
| Background work that outlives a request | a hosted service reading from a channel |

Bounded data parallelism with a shared result (`Interlocked`, never a plain `+=`):

`src/backend/packages/<PROJECT>.Notes/ConcurrencyProbe.cs (excerpt)`

```csharp
        var total = 0;

        await Parallel.ForEachAsync(
            words,
            new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
            async (word, _) =>
            {
                await Task.Yield();

                Interlocked.Add(ref total, word.Count(c => "aeiou".Contains(c)));
            }
        );

        return total;
```

Producers and one consumer.  The producers compute facets (messages) concurrently and write them to a bounded channel; one consumer merges them, so the merge state needs no lock:

`src/backend/packages/<PROJECT>.Notes/ConcurrencyProbe.cs (excerpt)`

```csharp
        var channel = Channel.CreateBounded<Facet>(
            new BoundedChannelOptions(capacity: 32) { SingleReader = true }
        );

        // 👇 Start the consumer first and do not await it yet.
        var consumer = ConsumeAsync(channel.Reader, ct);

        try
        {
            await Parallel.ForEachAsync(
                recordIds,
                new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
                async (recordId, token) =>
                {
                    foreach (var facet in produce(recordId))
                    {
                        await channel.Writer.WriteAsync(facet, token);
                    }
                }
            );

            channel.Writer.Complete();
        }
        catch (Exception ex)
        {
            // NOTE: complete the writer with the error, or the consumer waits forever.
            channel.Writer.Complete(ex);

            throw;
        }

        return await consumer;
```

## Rules

- **Avoid locks.**  Prefer immutable data, a single owner of the state (the channel consumer), `Interlocked`, or a concurrent collection.  When a lock is unavoidable use `System.Threading.Lock` (`private readonly Lock _gate = new();`, `lock (_gate) { ... }`), not `lock (object)`.  `await` is not allowed inside a `lock` body: keep the locked section synchronous and short.
- **A channel replaces synchronization.**  A `DbContext` is not thread-safe, so let producers compute and have **one consumer** do all the writes instead of one context per producer.
- **Bound the channel** (`Channel.CreateBounded`) so a slow consumer pushes back on producers instead of growing memory; `SingleReader = true` when one task reads.
- **Start the consumer first**; do not await it until the producers finish.
- **Always complete the writer**, and complete it **with the exception** when a producer fails (`Writer.Complete(ex)`); otherwise the consumer waits forever.
- **Read with `await foreach (... in reader.ReadAllAsync(ct))`**: it ends when the writer completes.
- **Model the message as a closed set of records** (an abstract record with sealed subtypes) and handle it with a switch that throws on unknown types: the compiler cannot prove exhaustiveness over an open hierarchy, so the `_` arm is required.
- **Thread-safe collection for shared results** (`ConcurrentDictionary`, `ConcurrentBag`) when `Parallel.ForEachAsync` bodies write to one; a plain `List<T>` or `+=` races.
- **Pass the `CancellationToken`** to every awaited call and to `ParallelOptions`.
- **No `Task.Run`** in request code: it only moves I/O-bound work to another thread, and work that outlives the request runs after its scoped services and cancellation token are gone.  Background work belongs in a hosted service reading from a channel.  **No `.Result` / `.Wait()`** outside the REPL and tests.
- Measure before parallelizing: parallelism adds failure modes, and the database is usually the bottleneck.

## Testing concurrent code

Assert on the **merged result**, never on ordering or timing: sort for comparison and use enough items to make a race likely (a thousand increments, not three).  Do not `Task.Delay` to "let it finish": await the task.
