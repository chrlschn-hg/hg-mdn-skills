# Loading and Saving Data

Use contract-generated MSW handlers to test queries and mutations through the real client.  See [mocking.md](mocking.md).

Server state is anything the backend owns: lists, records, the signed-in user.  **Read it with `useQuery`, write it with `useMutation`**, both from [Pinia Colada](https://pinia-colada.esm.dev/llms.txt).  The library keeps a cache, shares one request between callers, aborts a request once nothing waits for it, and keeps old data on screen while it refreshes.  Do not write that code by hand.

Random-looking front-end bugs are usually load concurrency: a slow response overwrites a newer one, three components fire the same request, a poll piles up behind a slow server.  The library removes each of them when the **query key** holds every input.  Client state (selection, preferences) is a different thing: `state-pinia.md`.

## Setup (ships in the baseline)

- `main.ts` installs Pinia first, then `PiniaColada`: the query cache is a Pinia store.
- `src/api/colada.d.ts` types every query and mutation `error` as the generated `ResponseError` (`status`, `data`).  A network failure is a plain `Error` at runtime (`status` is `undefined`): show errors with `errorText` and `toApiError`, which handle both.
- **Defaults** (from the library source): `staleTime` 5 s, `gcTime` 5 minutes; data older than `staleTime` refetches on mount, window focus, and reconnect.  **No retry.**  Change one query with its options; a retry policy (never retry a 4xx) is the opt-in `@pinia/colada-plugin-retry`.

## 1. Query options: `queries/<resource>.ts`

**Only `src/queries` (and the session store through `queries/session.ts`) imports a generated client.**  One options factory per read; components call `useQuery(options)` and never the client.  This is the boundary tests mock (`testing.md`).

`src/app/src/queries/notes.ts (excerpt)`

```ts
import { defineQueryOptions } from "@pinia/colada";
import { listNotes } from "@/api/generated/clients/listNotes";
import type { NoteListResponse } from "@/api/generated/types/NoteListResponse";

/** Root key of the Notes tag: a write invalidates everything under it. */
export const notesKey = ["notes"] as const;

/**
 * Notes, newest first; only those in `notebookId` when given.  The filter is part of the key, so
 * each filter has its own cache entry, and a response that lands late can only fill the entry it
 * was asked for.
 */
export const notesQuery = defineQueryOptions((notebookId: string | null = null) => ({
  key: [...notesKey, { notebookId }],
  query: ({ signal }) =>
    listNotes({ query: notebookId ? { notebookId } : undefined, signal }).unwrap(),
  // Keep the previous filter's rows on screen while the new filter loads.
  placeholderData: (previous: NoteListResponse | undefined) => previous,
}));
```

- **The key holds every input the query reads**: `[resource, { filters }]`.  Same key, same cache entry, one request.  A different key is a different entry, so a slow response for the old filter cannot overwrite the new one.  If you want a `latestLoad` counter, the key is missing an input.
- Keys are arrays; the first item is the resource root (`notesKey`), so one prefix invalidates the tag (section 3).
- **Forward `signal`**: the library aborts a request when the last component using that key leaves, and on `refetch()` or a cancel.  It does not abort while another component still uses the key.  The browser drops the response; the server may still finish the work.  `.unwrap()` throws `ResponseError` on a non-2xx (`api-client.md`).
- `staleTime` per query: long for data that rarely changes (`notebooks`: 5 minutes), short for live data.  Fresh data is not refetched on mount.
- Options as a function (`defineQueryOptions((id) => ({ ... }))`) when the query takes parameters; a plain object when it does not (`healthQuery`).
- Read the cache with `queryCache.getQueryData(key)`.  Write it with `setQueryData` only when the client already knows the answer: an optimistic change (section 4) or a sign-out (`state-pinia.md`).  `setQueryData` does not cancel a request in flight: call `cancelQueries` first, or the late response overwrites it.

## 2. In a component

`useQuery` loads on mount and shares the request with every other caller.  There is no `onMounted(load)` and no `watch(filter, load, { immediate: true })`: pass a getter and the query follows its inputs.

```ts
import { useQuery } from "@pinia/colada";
import { errorText } from "@/api/errors";
import { notesQuery } from "@/queries/notes";

const notebookId = ref<string | null>(null);
const { data, error, isPending, isLoading, isPlaceholderData, refetch } = useQuery(() =>
  notesQuery(notebookId.value),
);
const loadError = computed(() => errorText(error.value, "Could not load notes."));
```

```vue
<AsyncContent :loading="isPending" :error="loadError" :empty="!data?.items.length">
  <!-- rows; dim them while `isPlaceholderData` -->
</AsyncContent>
<UButton :loading="isLoading" @click="refetch()">Refresh</UButton>
```

| Flag | True when | Use it for |
|---|---|---|
| `isPending` | `status` is `"pending"`: nothing has loaded for this key and nothing has failed yet | `AsyncContent :loading`: skeletons only on the first load (`ui-states.md`) |
| `isLoading` | a request is running (first load, refresh, refetch) | a spinner on the Refresh button |
| `isPlaceholderData` | the rows belong to the previous key | dimming the table while a filter loads |
| `error` | the last request failed (`ResponseError`); `data` keeps the last good value | `errorText(error.value, fallback)` for `AsyncContent :error` |
| `status` | `"pending"`, `"success"`, or `"error"` | narrowing in TypeScript |

- `isPending` is not "a request is running" (that is `isLoading`).  A first load that fails is `status: "error"`, so `isPending` is false and `error` is set.  A query with `enabled: false` stays pending with no request.  With `placeholderData`, a new key is `success` at once: `isPending` false, `isPlaceholderData` true.
- **`refresh()`** joins a request that is running and skips fresh data.  **`refetch()`** always starts a new request and cancels the one in flight.  A Refresh button calls `refetch()`; a timer calls `refresh()` (section 6).
- A dependent or paused query sets `enabled` in its options: `useQuery(() => ({ ...noteQuery(id.value), enabled: !!id.value }))`.
- A route param is the key: `useQuery(() => noteQuery(String(route.params.id)))`.
- A router guard needs no query of its own: it calls the session store (`routing.md`).

## 3. Writes: `useMutation` that returns a `SaveResult`

A write the caller reacts to returns a `SaveResult` (`api-client.md`), so a form can show the backend's message on the right field (`forms.md`).  The mutation composable lives next to the queries it changes.

`src/app/src/queries/notes.ts (excerpt)`

```ts
import { useMutation, useQueryCache } from "@pinia/colada";
import { toSaveFailure, type SaveResult } from "@/api/errors";
import { createNote } from "@/api/generated/clients/createNote";
import { invalidate } from "@/queries/invalidate";

/** What the create form submits. */
export type NoteDraft = { title: string; body?: string; notebookId?: string | null };

/** Creates a note.  The failure carries the backend's error `code`; the list refetches. */
export function useCreateNote() {
  const queryCache = useQueryCache();
  const { mutateAsync, isLoading } = useMutation({
    mutation: (draft: NoteDraft) => createNote({ body: draft }).unwrap(),
    onSettled: () => invalidate(queryCache, notesKey),
  });

  async function create(draft: NoteDraft): Promise<SaveResult> {
    try {
      await mutateAsync(draft);
      return { ok: true };
    } catch (err: unknown) {
      return toSaveFailure(err, "Could not save the note.");
    }
  }

  return { create, saving: isLoading };
}
```

`src/app/src/queries/invalidate.ts`

```ts
import type { EntryKey, useQueryCache } from "@pinia/colada";

/**
 * Marks everything under `key` stale and refetches what is on screen, and waits for it, so a form
 * can stay "saving" until the list is fresh.  A failed refetch shows on that query's own `error`;
 * it must not fail the write that already succeeded, so it is swallowed here.
 */
export async function invalidate(queryCache: ReturnType<typeof useQueryCache>, key: EntryKey) {
  await queryCache.invalidateQueries({ key }).catch(() => undefined);
}
```

- **Refetch, do not patch.**  After a write, invalidate the root key: the list shows what the server saved, not what the request said.  Default to this; optimistic updates are section 4.
- **Use `mutateAsync` in a wrapper**, not `mutate`: `mutate` swallows the error, and the form needs the `SaveResult`.
- **Call `useQueryCache()` in the composable body** (an injection context), and use the handle inside callbacks.
- **Always call `invalidate`, never `queryCache.invalidateQueries` directly in `onSettled`.**  The mutation waits for the promise you return, and `invalidateQueries` rejects when a refetch fails: a saved note would report "could not save".
- Invalidate the narrowest root that covers the change: `notesKey` covers every filter of the list; a different tag stays untouched.  Inactive entries are marked stale and refetch when they are used again.
- `saving` is the mutation's `isLoading`: it stays true until the refetch finishes.

## 4. Optimistic updates

Show the change before the server confirms **only where a failure is handled**: toggles, reorders, deletes.  Default to refetching (section 3); optimism is not worth it for creates.

The helper edits every loaded list under a key and returns the undo.

`src/app/src/queries/optimistic.ts`

```ts
import type { EntryKey, useQueryCache } from "@pinia/colada";

/**
 * Applies `update` to every loaded list under `key` and returns a function that undoes it.  Call
 * the undo from `onError`.  It restores a list only if nothing else changed that list since
 * (another optimistic edit, a refetch): then the refetch on settle is the truth.  The refetch
 * alone is not enough: it skips an inactive or disabled query, and it can fail.
 */
export function optimisticUpdate<T>(
  queryCache: ReturnType<typeof useQueryCache>,
  key: EntryKey,
  update: (previous: T) => T,
) {
  // Loaded lists only: a list on its first load has nothing to edit, and cancelling that request
  // would leave it idle until the mutation settles.
  const loaded = { key, status: "success" } as const;

  // Compute every edit before writing one: an `update` that throws leaves all lists untouched.
  const edits = queryCache.getEntries(loaded).map((entry) => {
    const before = entry.state.value.data as T;
    return { entryKey: entry.key, before, after: update(before) };
  });

  // A refetch that left before the edit would overwrite it with old data.
  queryCache.cancelQueries(loaded);
  for (const { entryKey, after } of edits) {
    queryCache.setQueryData(entryKey, after);
  }

  return () => {
    for (const { entryKey, before, after } of edits) {
      if (queryCache.getQueryData(entryKey) === after) {
        queryCache.setQueryData(entryKey, before);
      }
    }
  };
}
```

`src/app/src/queries/notes.ts (excerpt)`

```ts
import { optimisticUpdate } from "@/queries/optimistic";

/** Deletes a note.  The row leaves every loaded list at once; a failed delete puts it back. */
export function useDeleteNote() {
  const queryCache = useQueryCache();
  const { mutateAsync } = useMutation({
    mutation: (id: string) => deleteNote({ path: { id } }).unwrap(),
    onMutate: (id) => ({
      rollback: optimisticUpdate<NoteListResponse>(queryCache, notesKey, (list) => ({
        ...list,
        items: list.items.filter((note) => note.id !== id),
      })),
    }),
    // The context is empty if `onMutate` threw.
    onError: (_error, _id, context) => context.rollback?.(),
    onSettled: () => invalidate(queryCache, notesKey),
  });

  async function remove(id: string): Promise<SaveResult> {
    try {
      await mutateAsync(id);
      return { ok: true };
    } catch (err: unknown) {
      return toSaveFailure(err, "Could not delete the note.");
    }
  }

  return { remove };
}
```

- **Snapshot and undo on error, then refetch.**  The undo puts the rows back at once, even when the settle refetch fails, and for a list no component shows (a refetch skips those).  The refetch on settle then reconciles with the server.
- The undo skips a list that changed after the edit: the refetch decides.  Two overlapping writes to one list can leave a row missing in a list no component shows, until that list is used again (it is stale, so it refetches then).  Disable the control while a write runs if that matters.
- The helper edits only **loaded** lists, and computes every edit before writing one.  A list on its first load or in an error state is not edited: the refetch on settle fixes it.  The cancel is scoped the same way: cancelling a first load would leave it idle until the mutation settles.
- A failed write still returns a `SaveResult`: show the toast.  Colada's own recipe: [optimistic updates](https://pinia-colada.esm.dev/guide/optimistic-updates.md).

## 5. Shared data: one request, many readers

Several components call `useQuery(notebooksQuery)` on mount (the page, the filter, the form).  The library sends one request and gives all of them the same data.  Nothing to write: no `loaded` flag, no shared promise.

`src/app/src/queries/notebooks.ts (excerpt)`

```ts
/** Notebooks: read by several components, changed rarely, so a long `staleTime` skips refetches. */
export const notebooksQuery = defineQueryOptions({
  key: notebooksKey,
  query: ({ signal }) => listNotebooks({ signal }).unwrap(),
  staleTime: 5 * 60_000,
});
```

Data the whole app needs for as long as it runs (the signed-in user) may live in a store that wraps the query: `session-store.ts` (`state-pinia.md`, `routing.md`).  Everything else is called from the page.

## 6. Polling and auto-refresh

Refreshing on a timer must never **overlap** (a slow server would accumulate requests) and must stop when the page goes away.  The composable polls with `refresh`: a tick joins the request in flight.

`src/app/src/composables/useAutoRefresh.ts`

```ts
import type { UseQueryReturn } from "@pinia/colada";

type AutoRefreshOptions = {
  /** Time between automatic refreshes while enabled.  Keep it at or above the query's `staleTime`. */
  intervalMs?: number;
};

/**
 * Refreshes a query on a timer the user can switch on and off.  Pass what `useQuery` returned.
 *
 * Uses `refresh`, never `refetch`: a tick while a request is in flight joins it, so a slow
 * backend does not build a queue; `refetch` would cancel and restart it, and a backend slower than
 * the interval would never answer.  `refresh` skips data that is still fresh, hence the note on
 * `intervalMs`.  The timer stops when the owning component unmounts.
 */
export function useAutoRefresh(
  query: Pick<UseQueryReturn, "refresh">,
  options: AutoRefreshOptions = {},
) {
  const enabled = ref(false);

  // `useIntervalFn` (VueUse) is auto-imported and stops itself when its scope is disposed.
  const intervalMs = options.intervalMs ?? 30_000;
  const { pause, resume } = useIntervalFn(() => void query.refresh(), intervalMs, {
    immediate: false,
  });

  watch(enabled, (on) => (on ? resume() : pause()), { immediate: true });

  return { enabled };
}
```

- Use it: `const query = useQuery(...); const { enabled: autoRefresh } = useAutoRefresh(query);`.  Show `isLoading` as the "refreshing" mark.
- Offer the control to the user (`USwitch` "Auto-refresh", default **off**).  Polling by default hammers the backend and surprises people.
- A tick fetches only when the data is older than `staleTime` or the last request failed: keep `intervalMs` at or above `staleTime`, or set a shorter `staleTime` on that query.
- Under refresh the data stays on screen (`ui-states.md`, "The four states"): `isPending` is false once the first load finished.
- A manual `refetch()` during a poll aborts the request in the browser; the server may still finish it, so the no-overlap promise covers requests, not server work.
- Pause polling while the user is interacting with the data (dragging, editing) if a refresh would disrupt them.
- For real-time needs beyond polling (hundreds of clients, sub-second updates) the backend should push (SSE or WebSockets): a backend design decision first.
- Test with fake timers (`testing.md`).

## 7. Debounce what is typed

Search boxes and filters that trigger work wait for a pause after the last keystroke: `refDebounced(search, 250)` for a value that others derive from, `useDebounceFn(fn, 250)` for a function.  Put the debounced value in the query key: each pause is one request, and the library aborts the previous one when nothing else uses its key.

## Pitfalls

- **A query runs when it is created**: on mount, and when a store that holds one is first used.  In a test, arrange the mocked response before that moment (`testing.md`).
- **The cache is a Pinia store.**  `createTestingPinia` stubs the actions it needs: tests use a real Pinia plus `PiniaColada` (`testing.md`).
- `invalidateQueries` marks everything under a key stale, but it **never refetches a query with `enabled: false`**.
- **Do not copy query data into a store `ref`.**  Two sources of truth go stale.  Derive with `computed` from the query's `data`.
- Window focus refetches stale data after 5 s.  Raise `staleTime` on an expensive query; do not turn the trigger off globally.
- A catch-all `catch` around `useQuery` finds nothing: the query never throws.  Read `error`.
