# Loading Data Correctly

Random-looking front-end bugs are usually load concurrency: a slow response overwrites a newer one, three components fire the same request, a poll piles up behind a slow server.  This file owns those patterns; store basics are in `state-pinia.md`.

## 1. Ignore stale responses

A user changes a filter twice quickly; the first request is slow and lands **after** the second.  Without a guard the list shows results for the filter the user already left.  Number each request and let only the newest write state:

`src/app/src/stores/notes-store.ts`

```ts
import { acceptHMRUpdate, defineStore } from "pinia";
import { createNote } from "@/api/generated/clients/createNote";
import { listNotes } from "@/api/generated/clients/listNotes";
import type { NoteResponse } from "@/api/generated/types/NoteResponse";
import { errorMessage, toSaveFailure, type SaveResult } from "@/api/errors";

/** What the create form submits. */
export type NoteDraft = { title: string; body?: string; notebookId?: string | null };

/**
 * Notes.  The store owns the API calls and the loading/error state; components read it and call
 * its actions.  Read actions record errors; write actions return a {@link SaveResult}.
 */
export const useNotesStore = defineStore("notes", () => {
  const notes = ref<NoteResponse[]>([]);
  const loading = ref(false);
  const saving = ref(false);
  const error = ref<string | null>(null);

  // Each `load` takes a number; only the newest request may write state.  Without this, a slow
  // response for an older filter can arrive last and overwrite the list the user is looking at.
  let latestLoad = 0;

  /** Loads notes, newest first; only those in `notebookId` when given. */
  async function load(notebookId?: string | null) {
    const request = ++latestLoad;
    loading.value = true;
    error.value = null;

    try {
      const response = await listNotes({
        query: notebookId ? { notebookId } : undefined,
      }).unwrap();

      if (request === latestLoad) {
        notes.value = response.items;
      }
    } catch (err: unknown) {
      if (request === latestLoad) {
        error.value = errorMessage(err, "Could not load notes.");
      }
    } finally {
      if (request === latestLoad) {
        loading.value = false;
      }
    }
  }

  /** Creates a note and puts it first.  The failure carries the backend's error `code`. */
  async function create(draft: NoteDraft): Promise<SaveResult> {
    saving.value = true;

    try {
      const created = await createNote({ body: draft }).unwrap();
      notes.value = [created, ...notes.value];
      return { ok: true };
    } catch (err: unknown) {
      return toSaveFailure(err, "Could not save the note.");
    } finally {
      saving.value = false;
    }
  }

  return { notes, loading, saving, error, load, create };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useNotesStore, import.meta.hot));
}
```

- `latestLoad` increments per call; every state write checks `request === latestLoad`, including the `loading` flag in `finally` (an old request must not clear the newer one's spinner).
- Apply it to **any** load whose input can change while it is in flight: filters, search, paging, route params.
- `AbortController` can also cancel the old request (the generated functions accept `{ signal }`), which saves bandwidth, but the guard is still needed: cancellation is best-effort and the response may already be on its way.

Tested with a deliberately late response (the body of the store test in `testing.md`):

`src/app/src/stores/notes-store.test.ts (excerpt)`

```ts
    let finishOlder!: (value: unknown) => void;
    api.listNotes
      .mockReturnValueOnce({ unwrap: () => new Promise((resolve) => (finishOlder = resolve)) })
      .mockReturnValueOnce(ok({ items: [note("newer")] }));

    const store = useNotesStore();
    const older = store.load("a");
    await store.load("b");

    finishOlder({ items: [note("older")] }); // arrives last, but is stale
    await older;

    expect(store.notes.map((n) => n.id)).toEqual(["newer"]);
```

## 2. Share one in-flight request

Several components call `load()` on mount (the page, the header, a filter).  Without care that is several identical HTTP calls.  Keep the promise and hand it to concurrent callers; skip the call once the data is loaded:

`src/app/src/stores/notebooks-store.ts`

```ts
import { acceptHMRUpdate, defineStore } from "pinia";
import { createNotebook } from "@/api/generated/clients/createNotebook";
import { listNotebooks } from "@/api/generated/clients/listNotebooks";
import type { NotebookResponse } from "@/api/generated/types/NotebookResponse";
import { errorMessage, toSaveFailure, type SaveResult } from "@/api/errors";

/** Notebooks.  Loaded once and shared; concurrent `load()` calls share one request. */
export const useNotebooksStore = defineStore("notebooks", () => {
  const notebooks = ref<NotebookResponse[]>([]);
  const loading = ref(false);
  const loaded = ref(false);
  const error = ref<string | null>(null);

  // In-flight request, so several components calling `load()` on mount cost one HTTP call.
  let inFlight: Promise<void> | null = null;

  /** Loads notebooks.  Skips the call when already loaded unless `force` is set. */
  function load(options: { force?: boolean } = {}): Promise<void> {
    if (inFlight) {
      return inFlight;
    }

    if (loaded.value && !options.force) {
      return Promise.resolve();
    }

    loading.value = true;
    error.value = null;

    inFlight = (async () => {
      try {
        notebooks.value = (await listNotebooks().unwrap()).items;
        loaded.value = true;
      } catch (err: unknown) {
        error.value = errorMessage(err, "Could not load notebooks.");
      } finally {
        loading.value = false;
        inFlight = null;
      }
    })();

    return inFlight;
  }

  /** Creates a notebook and puts it first. */
  async function create(name: string): Promise<SaveResult> {
    try {
      const created = await createNotebook({ body: { name } }).unwrap();
      notebooks.value = [created, ...notebooks.value];
      return { ok: true };
    } catch (err: unknown) {
      return toSaveFailure(err, "Could not create the notebook.");
    }
  }

  return { notebooks, loading, loaded, error, load, create };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useNotebooksStore, import.meta.hot));
}
```

- `inFlight` is plain module/closure state (not reactive); it is cleared in `finally`.
- `loaded` distinguishes "never loaded" from "loaded and empty", so an empty list is not re-fetched forever.
- `load({ force: true })` is the refresh path; a normal `load()` is cheap to call anywhere.
- For data that goes stale (a dashboard), store a timestamp and reload when older than N seconds instead of a boolean.
- `session-store.ts` (`routing.md`) combines this with the stale guard.

## 3. Writes and optimistic updates

A write the caller reacts to returns a `SaveResult` (`api-client.md`) and updates state from the server's **response** (`notes = [created, ...notes]`), not from the request, so the list shows what was actually saved.

**Optimistic updates** (show the change before the server confirms) only where the failure path is handled: remember the previous value, apply the change, and on failure restore it and report the error.  Default to the response; optimism is worth it for toggles and reorders, not for creates.

## 4. Polling and auto-refresh

Refreshing on a timer must never **overlap** (a slow server would accumulate requests) and must stop when the page goes away:

`src/app/src/composables/useAutoRefresh.ts`

```ts
type AutoRefreshOptions = {
  /** Time between automatic refreshes while enabled. */
  intervalMs?: number;
};

/**
 * Refreshes data on a timer the user can switch on and off, plus an on-demand `refreshNow`.
 *
 * Never overlaps: a tick (or click) while a refresh is still running is skipped, so a slow
 * backend does not build a queue of requests.  `lastUpdatedAt` feeds an "updated 12s ago" label.
 * The timer stops when the owning component unmounts.
 *
 * @param refresh - Reloads whatever the page shows; awaited, errors are the callee's to handle.
 */
export function useAutoRefresh(refresh: () => Promise<void>, options: AutoRefreshOptions = {}) {
  const enabled = ref(false);
  const refreshing = ref(false);
  const lastUpdatedAt = ref<Date | null>(null);

  async function refreshNow() {
    if (refreshing.value) {
      return;
    }

    refreshing.value = true;

    try {
      await refresh();
      lastUpdatedAt.value = new Date();
    } finally {
      refreshing.value = false;
    }
  }

  // `useIntervalFn` (VueUse) is auto-imported and stops itself when its scope is disposed.
  const { pause, resume } = useIntervalFn(refreshNow, options.intervalMs ?? 30_000, {
    immediate: false,
  });

  watch(enabled, (on) => (on ? resume() : pause()), { immediate: true });

  return { enabled, refreshing, lastUpdatedAt, refreshNow };
}
```

- `useIntervalFn` (VueUse, auto-imported) is the timer; it pauses and resumes and stops with its scope (the unmounted component), so there is no leak to clean up.
- The overlap guard (`refreshing`) also makes the manual **Refresh** button safe to click during a tick.
- Offer the control to the user (`USwitch` "Auto-refresh", default **off**) and show the last update time (`lastUpdatedAt`).  Polling by default hammers the backend and surprises people.
- Under refresh keep the data on screen (`ui-states.md`, "first load").
- Pause polling while the user is interacting with the data (dragging, editing) if a refresh would disrupt them.
- For real-time needs beyond polling (hundreds of clients, sub-second updates) the backend should push (SSE or WebSockets): a backend design decision first.
- Test with fake timers (`testing.md`).

## 5. Debounce what is typed

Search boxes and filters that trigger work wait for a pause after the last keystroke: `refDebounced(search, 250)` for a value that others derive from, `useDebounceFn(fn, 250)` for a function.  Cancel or ignore the previous request on each change (patterns 1 and 4).
