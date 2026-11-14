# Front-end Testing

Use MSW for tests that exercise the generated client and Colada.  Keep `vi.mock` for focused boundaries.  The scaffold setup rejects unhandled requests, including swallowed rejections.  Read [mocking.md](mocking.md) for setup, deterministic data, and scenarios.

[Vitest](https://vitest.dev) with `happy-dom` and `@vue/test-utils` for unit, component, and wiring tests; the **Playwright MCP** for looking at the real running UI.  (Backend tests are TUnit: see the `hg-mdn-backend` skill.)  This file owns the test setup, the API mock boundary, page tests, fake timers, and browser verification; forms, overlays, routing, and the other topic references show their own tests.

## Setup

`vite.config.ts` carries the test config (it imports `defineConfig` from `vitest/config`):

```ts
test: {
  environment: "happy-dom",
  environmentOptions: { happyDOM: { url: "http://localhost:3000" } },
  include: ["src/**/*.test.ts"],
  setupFiles: ["./src/test/setup.ts"],
  restoreMocks: true,
  clearMocks: true,
},
```

- Tests run through the same Vite config, so auto-imports (`ref`, `defineStore`, `storeToRefs`, ...), the `@/` alias, and typed routes work in them.  `@pinia/colada` is not auto-imported: import `useQuery` explicitly.
- `npm test` runs `vitest run` (after regenerating the API client: `pretest`).  Watch mode: `npx vitest`.  One file: `npx vitest run src/queries/notes.test.ts`; by name: `-t "refetches the list"`.
- Tests sit beside the code: `thing.ts` and `thing.test.ts`.

## Test infrastructure (`src/test/`)

Four small files every test benefits from (they ship in the baseline):

`src/app/src/test/setup.ts`

```ts
import { fakerEN } from "@faker-js/faker";
import { afterAll, afterEach, beforeAll, beforeEach } from "vitest";
import { server, takeUnhandled } from "@/test/server";

// An unmatched request fails the test with MSW's message (method + URL + what to add), so a
// forgotten handler is loud.  MSW patches `fetch` itself: never stub it in this setup,
// because a stub replaces MSW's patch.  MSW 3 names the option `onUnhandledFrame`
// (`onUnhandledRequest` of MSW 2 is silently ignored).
beforeAll(() => server.listen({ onUnhandledFrame: "error" }));
afterEach(() => {
  server.resetHandlers();
  // The rejected `fetch` can be swallowed by app code (a query's error state); this cannot.
  const unhandled = takeUnhandled();
  if (unhandled.length > 0) {
    throw new Error(`Request(s) without a handler: ${unhandled.join(", ")}`);
  }
});
afterAll(() => server.close());

// The generated factories use the `fakerEN` instance, not the default `faker` export.  Kubb seeds it
// once at import; reset it per test so a test's data does not depend on the tests before it, and
// pin the reference date because `faker.date.anytime()` is relative to "now".
beforeEach(() => {
  fakerEN.seed(1);
  fakerEN.setDefaultRefDate("2026-06-01T00:00:00Z");
});
```

`src/app/src/test/mount.ts`

```ts
import ui from "@nuxt/ui/vue-plugin";
import { PiniaColada } from "@pinia/colada";
import { mount, type ComponentMountingOptions } from "@vue/test-utils";
import { createPinia } from "pinia";
import { defineComponent, type Component } from "vue";

/**
 * `mount` with what every component here needs: the Nuxt UI plugin (theme and injections), a
 * fresh Pinia, and its query cache (`PiniaColada`).  Pass `attachTo: document.body` when the
 * component teleports (slideovers, modals, popovers) and the test queries `document.body`.
 */
export function mountWithUi<T extends Component>(
  component: T,
  options: ComponentMountingOptions<T> = {},
) {
  return mount(component, {
    ...options,
    global: {
      ...options.global,
      plugins: [ui, createPinia(), PiniaColada, ...(options.global?.plugins ?? [])],
    },
  } as ComponentMountingOptions<T>);
}

/**
 * Runs a composable (or anything that needs a component's injections, like `useQuery` and
 * `useMutation`) inside a mounted component, and returns what it returned.  Unmount the wrapper
 * when done.
 */
export function runInSetup<T>(fn: () => T) {
  let result!: T;
  const wrapper = mountWithUi(
    defineComponent({
      setup() {
        result = fn();
        return () => null;
      },
    }),
  );

  return { result, wrapper };
}
```

`src/app/src/test/pinia.ts`

```ts
import { PiniaColada } from "@pinia/colada";
import { createPinia, setActivePinia } from "pinia";
import { createApp } from "vue";

/**
 * A fresh Pinia **and query cache** for a test that uses stores or queries outside a component.
 * The cache is a Pinia store, so a new Pinia means no data left over from the previous test.
 * It is installed in an app so `useQuery` finds its injections.  Never use `createTestingPinia`:
 * it stubs the actions the query cache relies on.
 */
export function installTestPinia() {
  const pinia = createPinia();
  createApp({}).use(pinia).use(PiniaColada);
  setActivePinia(pinia);

  return pinia;
}
```

`src/app/src/test/AppHarness.vue`

```vue
<script setup lang="ts">
// Hosts overlays: `useOverlay` (confirmation dialogs) renders into `UApp`, so a component that
// opens one must be tested inside it.
</script>

<template>
  <UApp>
    <slot />
  </UApp>
</template>
```

- **`setup.ts` is a network guard.**  MSW reports unhandled requests even when the caller catches the rejection.
- **`mountWithUi`** is `mount` plus the Nuxt UI plugin, a fresh Pinia, and its query cache (`PiniaColada`): what every component needs.
- **`runInSetup`** runs a query, a mutation, or a composable that uses them inside a mounted component: `useQuery` and `useMutation` need a component's injections.  Each call gets a fresh Pinia and cache; unmount the wrapper after the test.
- **`installTestPinia`** gives a test that uses a store outside a component a fresh Pinia and cache, attached to an app.  **Never `createTestingPinia`**: it stubs the actions the query cache relies on.
- **`AppHarness`** is a `UApp` host for anything that teleports or needs `UApp`: overlays opened through `useOverlay` (confirmations), toasts, `USlideover`/`UModal`/`UPopover`.  Mount the component in its default slot, with `attachTo: document.body`; the content is then in `document.body`, not in `wrapper`.
- Principle: **use MSW for client integration and `vi.mock` for focused boundaries; everything else is real.**

## What to test

| Subject | How | Example |
|---|---|---|
| **Queries and mutations** | use MSW with the real generated clients; run in `runInSetup`; assert `data`, `error`, flags, and the `SaveResult` | `queries/notes.msw.test.ts` |
| **Query unit boundary** | mock the generated client for a focused test | `queries/notes.test.ts` |
| **Stores** | `installTestPinia()` in `beforeEach`; arrange mocks before the first use of a store that wraps a query | `stores/session-store.test.ts` |
| **Composables** | call them directly with plain refs, or in `runInSetup` when they use a query; fake timers for time | `composables/useAutoRefresh.test.ts` |
| **Utils** | plain input and output | any `utils/*.test.ts` |
| **Components with logic** | `mountWithUi`; props, events, slots | `components/SearchBox.test.ts`, `NoteForm.test.ts` (`forms.md`) |
| **Overlays** | `AppHarness` + `attachTo: document.body` (`overlays.md`, "Testing overlays") | `components/NoteFormSlideover.test.ts` |
| **API error mapping, client config** | focused unit tests may stub `fetch` and restore it; use MSW for HTTP integration | `api/errors.test.ts` |
| **Router guards** | memory router and `router.push` (`routing.md`) | `router/guards.test.ts` |
| **Pages and whole-app wiring** | mount `App` with router, Nuxt UI, Pinia, `PiniaColada`; API mocked | `App.test.ts`, `pages/notes/index.test.ts` |

Test behavior a user or a caller can see.  Do not test Nuxt UI itself, and do not assert on markup details that a restyle would change.  Query by **text, role, or `aria-label`**, which doubles as an accessibility check.

Every page covers: the list, the **empty** state, the **error** state, the main action's success, and its failure.  Every query covers loading, error, success, and a stale or failing response; every mutation covers success and the backend's `code`.

## Mock at the generated-client boundary

For focused unit tests, mock the generated client boundary.  Use `vi.hoisted` so the mock exists before the module under test is imported:

`src/app/src/queries/notes.test.ts`

```ts
import { useQuery, useQueryCache } from "@pinia/colada";
import { afterEach, describe, expect, it, vi } from "vitest";
import { ResponseError } from "@/api/generated/.kubb/client";
import { runInSetup } from "@/test/mount";
import { notesKey, notesQuery, useCreateNote, useDeleteNote } from "./notes";

// Mock the generated clients at the module boundary; queries and mutations never see HTTP.
const api = vi.hoisted(() => ({ listNotes: vi.fn(), createNote: vi.fn(), deleteNote: vi.fn() }));
vi.mock("@/api/generated/clients/listNotes", () => ({ listNotes: api.listNotes }));
vi.mock("@/api/generated/clients/createNote", () => ({ createNote: api.createNote }));
vi.mock("@/api/generated/clients/deleteNote", () => ({ deleteNote: api.deleteNote }));

const note = (id: string, title = id) => ({
  id,
  notebookId: null,
  title,
  body: "",
  createdAtUtc: "2026-10-09T00:00:00Z",
});
const ok = <T>(value: T) => ({ unwrap: () => Promise.resolve(value) });
const fails = (error: unknown) => ({ unwrap: () => Promise.reject(error) });
const rejection = (status: number, data: unknown) =>
  new ResponseError({
    data,
    status,
    statusText: "",
    request: new Request("http://localhost/api/notes"),
    response: new Response(null, { status }),
  });

// `runInSetup` mounts a host component with a fresh Pinia and query cache: no data leaks between
// tests.  Unmount it so no query outlives its test.
let unmount: () => void = () => {};
afterEach(() => unmount());

describe("notesQuery", () => {
  it("loads notes", async () => {
    // A query fires when it is created: arrange the response first.
    api.listNotes.mockReturnValue(ok({ items: [note("1")] }));

    const { result: query, wrapper } = runInSetup(() => useQuery(notesQuery));
    unmount = () => wrapper.unmount();

    await vi.waitFor(() => expect(query.data.value).toEqual({ items: [note("1")] }));
    expect(query.error.value).toBeNull();
    expect(query.isPending.value).toBe(false);
  });

  it("asks the backend for one notebook's notes", async () => {
    api.listNotes.mockReturnValue(ok({ items: [] }));

    const { result: query, wrapper } = runInSetup(() => useQuery(() => notesQuery("nb-1")));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(query.status.value).toBe("success"));

    // `objectContaining`: the query also passes its abort `signal`.
    expect(api.listNotes).toHaveBeenCalledWith(
      expect.objectContaining({ query: { notebookId: "nb-1" } }),
    );
  });

  it("reports a failed load as the error and keeps no data", async () => {
    api.listNotes.mockReturnValue(fails(rejection(500, undefined)));

    const { result: query, wrapper } = runInSetup(() => useQuery(notesQuery));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(query.status.value).toBe("error"));

    expect(query.error.value).toBeInstanceOf(ResponseError);
    expect(query.data.value).toBeUndefined();
  });

  it("shares one request between components asking for the same filter", async () => {
    api.listNotes.mockReturnValue(ok({ items: [note("1")] }));

    const { result, wrapper } = runInSetup(() => [useQuery(notesQuery), useQuery(notesQuery)]);
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result[0].data.value?.items).toHaveLength(1));

    expect(api.listNotes).toHaveBeenCalledTimes(1);
    expect(result[1].data.value).toBe(result[0].data.value);
  });

  it("shows the newest filter when an older response lands last", async () => {
    let finishOlder!: (value: unknown) => void;
    api.listNotes
      .mockReturnValueOnce({ unwrap: () => new Promise((resolve) => (finishOlder = resolve)) })
      .mockReturnValueOnce(ok({ items: [note("newer")] }));

    const notebookId = ref<string | null>("a");
    const { result: query, wrapper } = runInSetup(() =>
      useQuery(() => notesQuery(notebookId.value)),
    );
    unmount = () => wrapper.unmount();
    await nextTick();

    notebookId.value = "b";
    await vi.waitFor(() => expect(query.data.value?.items.map((n) => n.id)).toEqual(["newer"]));

    finishOlder({ items: [note("older")] }); // arrives last, but belongs to the other filter
    await new Promise((resolve) => setTimeout(resolve, 20));

    expect(query.data.value?.items.map((n) => n.id)).toEqual(["newer"]);
  });
});

describe("useCreateNote", () => {
  it("saves, then refetches the list that is on screen", async () => {
    api.listNotes.mockReturnValueOnce(ok({ items: [] }));
    api.createNote.mockReturnValue(ok(note("2")));
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      write: useCreateNote(),
    }));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result.list.data.value).toEqual({ items: [] }));

    api.listNotes.mockReturnValueOnce(ok({ items: [note("2")] })); // what the server saved
    const saved = await result.write.create({ title: "2" });

    expect(saved).toEqual({ ok: true });
    expect(result.list.data.value?.items.map((n) => n.id)).toEqual(["2"]);
    expect(result.write.saving.value).toBe(false);
  });

  it("returns the backend's error code and message when the API rejects the note", async () => {
    api.createNote.mockReturnValue(
      fails(
        rejection(400, { code: "title_too_long", message: "Title may be at most 200 characters." }),
      ),
    );
    const { result, wrapper } = runInSetup(() => useCreateNote());
    unmount = () => wrapper.unmount();

    const saved = await result.create({ title: "x" });

    expect(saved).toEqual({
      ok: false,
      code: "title_too_long",
      message: "Title may be at most 200 characters.",
    });
  });

  it("does not fail a saved note when the refetch afterwards fails", async () => {
    api.listNotes.mockReturnValueOnce(ok({ items: [] }));
    api.createNote.mockReturnValue(ok(note("2")));
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      write: useCreateNote(),
    }));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result.list.status.value).toBe("success"));

    api.listNotes.mockReturnValueOnce(fails(rejection(500, undefined)));
    const saved = await result.write.create({ title: "2" });

    expect(saved).toEqual({ ok: true });
    await vi.waitFor(() => expect(result.list.error.value).not.toBeNull()); // the list shows it
  });
});

describe("useDeleteNote", () => {
  const ids = (list: { items: { id: string }[] } | undefined) => list?.items.map((n) => n.id);

  it("removes the row at once and puts it back when the delete fails", async () => {
    api.listNotes.mockReturnValueOnce(ok({ items: [note("a"), note("b")] }));
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result.list.data.value?.items).toHaveLength(2));

    let failDelete!: (error: unknown) => void;
    api.deleteNote.mockReturnValue({
      unwrap: () => new Promise((_, reject) => (failDelete = reject)),
    });
    api.listNotes.mockReturnValueOnce(ok({ items: [note("a"), note("b")] })); // the truth after settle
    const removing = result.write.remove("a");

    await vi.waitFor(() => expect(ids(result.list.data.value)).toEqual(["b"]));

    failDelete(rejection(500, undefined));
    expect((await removing).ok).toBe(false);
    await vi.waitFor(() => expect(ids(result.list.data.value)).toEqual(["a", "b"]));
  });

  // The refetch on settle is not the rollback: it can fail, and it skips an unwatched list.
  it("puts the row back even when the refetch afterwards fails", async () => {
    api.listNotes.mockReturnValueOnce(ok({ items: [note("a"), note("b")] }));
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result.list.data.value?.items).toHaveLength(2));

    api.deleteNote.mockReturnValue(fails(rejection(500, undefined)));
    api.listNotes.mockReturnValueOnce(fails(rejection(500, undefined)));
    const removed = await result.write.remove("a");

    expect(removed.ok).toBe(false);
    await vi.waitFor(() => expect(result.list.error.value).not.toBeNull());
    expect(ids(result.list.data.value)).toEqual(["a", "b"]);
  });

  it("restores a cached list that no component is showing", async () => {
    const { result, wrapper } = runInSetup(() => ({
      cache: useQueryCache(),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();
    const key = notesQuery("x").key; // no component tracks this entry: no refetch will run
    result.cache.setQueryData(key, { items: [note("a"), note("b")] });

    api.deleteNote.mockReturnValue(fails(rejection(500, undefined)));
    await result.write.remove("a");

    expect(ids(result.cache.getQueryData(key))).toEqual(["a", "b"]);
  });

  it("does not stall a list that is still on its first load", async () => {
    let finishLoad!: (value: unknown) => void;
    api.listNotes.mockReturnValueOnce({
      unwrap: () => new Promise((resolve) => (finishLoad = resolve)),
    });
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();

    api.deleteNote.mockReturnValue({ unwrap: () => new Promise(() => {}) }); // never settles
    void result.write.remove("a");
    finishLoad({ items: [note("a")] });

    await vi.waitFor(() => expect(ids(result.list.data.value)).toEqual(["a"]));
  });

  it("edits no list when the update fails on one of them", async () => {
    const { result, wrapper } = runInSetup(() => ({
      cache: useQueryCache(),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();
    const readable = notesQuery("x").key;
    result.cache.setQueryData(readable, { items: [note("a"), note("b")] });
    result.cache.setQueryData([...notesKey, { notebookId: "y" }], undefined); // loaded, nothing to filter

    const removed = await result.write.remove("a");

    expect(removed.ok).toBe(false);
    expect(ids(result.cache.getQueryData(readable))).toEqual(["a", "b"]);
  });

  it("leaves a list alone when another change touched it after the delete", async () => {
    api.listNotes.mockReturnValueOnce(ok({ items: [note("a"), note("b")] }));
    const { result, wrapper } = runInSetup(() => ({
      list: useQuery(notesQuery),
      cache: useQueryCache(),
      write: useDeleteNote(),
    }));
    unmount = () => wrapper.unmount();
    await vi.waitFor(() => expect(result.list.data.value?.items).toHaveLength(2));

    let failDelete!: (error: unknown) => void;
    api.deleteNote.mockReturnValue({
      unwrap: () => new Promise((_, reject) => (failDelete = reject)),
    });
    api.listNotes.mockReturnValueOnce(fails(rejection(500, undefined))); // no refetch to reconcile
    const removing = result.write.remove("a");
    await vi.waitFor(() => expect(ids(result.list.data.value)).toEqual(["b"]));

    result.cache.setQueryData(notesQuery().key, { items: [note("c")] }); // someone else wrote
    failDelete(rejection(500, undefined));
    await removing;

    expect(ids(result.list.data.value)).toEqual(["c"]); // not overwritten with the old snapshot
  });
});
```

- Return `{ unwrap: () => Promise.resolve(value) }` (success) or `{ unwrap: () => Promise.reject(error) }` (failure): exactly what the query functions use.  Small helpers (`ok(value)`, `fails(error)`) keep tests readable.
- For failures build a real `ResponseError` (as above) so the error-mapping code is exercised and the `code` reaches the `SaveResult`.
- **Arrange the mock before the query exists.**  A query fires when it is created: on mount in `runInSetup`, or on the first use of a store that wraps one.
- `runInSetup` per test: a fresh Pinia and query cache, so no cached data leaks.  Unmount in `afterEach`.  Never `createTestingPinia`.
- Wait with `vi.waitFor(() => expect(...))`; the query resolves after the call returns.
- Assert what the caller sees (`data`, `error`, `status`, `isPending`, the `SaveResult`), not that a function was called with particular arguments, unless the arguments are the point (a request body, a filter).
- Test the **concurrency** the key handles: the newest key wins over a slow older response, one key shares one request, a write refetches the list, a failed optimistic change rolls back even when the refetch fails too (`data-loading.md`).
- A store that wraps a query (`session-store.test.ts`) calls `installTestPinia()` in `beforeEach` and creates the store after it arranges the mock.
- A test that needs HTTP behavior uses MSW handlers (`mocking.md`).

## Page tests: through the whole app

Unit tests miss broken wiring: a missing plugin, a bad route, a layout that throws.  A page test mounts the **real app** (router, layout, Nuxt UI, Pinia, query cache) and mocks only the API.  Arrange the mocked responses first: the page's queries fire on mount.  Mount at a route:

`src/app/src/pages/notes/index.test.ts (excerpt)`

```ts
  const root = document.createElement("div");
  document.body.append(root);
  const router = createRouter({ routes, history: createMemoryHistory() });
  const app = createApp(App).use(createPinia()).use(PiniaColada).use(router).use(ui);

  await router.push("/notes");
  await router.isReady();
  app.mount(root);
  await flushPromises();

  unmount = () => app.unmount();
```

Then drive the UI like a user and assert what it shows:

`src/app/src/pages/notes/index.test.ts (excerpt)`

```ts
    click("New note");
    await flushPromises();
    type("input[placeholder='What to remember']", "Buy milk");
    document.body
      .querySelector("form")!
      .dispatchEvent(new Event("submit", { bubbles: true, cancelable: true }));

    await vi.waitFor(() => expect(text()).toContain("Note added"));
    expect(api.createNote).toHaveBeenCalledWith({
      body: { title: "Buy milk", body: undefined, notebookId: null },
    });
    expect(text()).toContain("Buy milk");
```

- `createMemoryHistory()` avoids the browser location; `router.push(path)`, `router.isReady()`, then `app.mount(root)`.
- Everything renders in `document.body` (overlays teleport there), so query `document.body`, not only the root.
- `vi.waitFor(() => expect(...))` waits for async rendering and data; `flushPromises()` settles pending promises.
- Click a button by its label, set an input's value and dispatch `input`, dispatch `submit` on the form.  The toast text, the table rows, and the field message are the assertions.
- Clean up: `app.unmount()` and `document.body.innerHTML = ""` in `afterEach`.

The baseline smoke test (`App.test.ts`) is the same shape for the home page.  It also guards **icon bundling**: an unbundled icon is fetched from Iconify through `node:https`, silently, so the test spies on `https.request`:

`src/app/src/App.test.ts (excerpt)`

```ts
    // NOTE: an icon that is not bundled is fetched from api.iconify.design through `node:https`
    // (not `fetch`), and a failed fetch is silent, so only this spy notices.
    const request = vi.spyOn(https, "request");
    api.getHealth.mockReturnValue({
      unwrap: () =>
        Promise.resolve({
          status: "Healthy",
          checkedAtUtc: "2026-10-09T00:00:00Z",
          build: { version: "0.0.0-dev", commit: null, builtAtUtc: null },
        }),
    });

    const { root, unmount } = await mountAt("/");
    await vi.waitFor(() => expect(root.textContent).toContain("Healthy"));
    await new Promise((resolve) => setTimeout(resolve, 100)); // let icon loading start

    expect(request).not.toHaveBeenCalled();
```

## Time, environment, and globals

- **Fake timers** for debounce, polling, and timeouts: `vi.useFakeTimers()` in `beforeEach`, `vi.useRealTimers()` in `afterEach`, and `await vi.advanceTimersByTimeAsync(ms)` (runs the timers *and* the promises they start; use it, not the sync version, whenever the callback is async).
- **Polling:** call `useAutoRefresh` with its query in `runInSetup`, arrange the mock first, and count calls to the generated function after each `advanceTimersByTimeAsync` (`composables/useAutoRefresh.test.ts`).
- **Environment variables:** `vi.stubEnv("VITE_API_BASE_URL", "")` (Vitest restores them), as `api/errors.test.ts` does.
- **Browser APIs happy-dom lacks** (`matchMedia` layout, `ResizeObserver`): stub them in the test that needs them (`vi.stubGlobal`); `useIsMobile.test.ts` shows `matchMedia` (`accessibility-responsive.md`).
- Tests import generated modules like any other; `npm test` regenerates them first.

## Debugging a failing test or a UI problem

1. Run the one file: `npx vitest run <path> -t "<name>"`.
2. `Request(s) without a handler`: add an MSW handler or mock the client boundary.
3. `Cannot find ... in document`: the content teleported; query `document.body`, mount with `attachTo`.
4. Assertion before the data arrived: wrap in `vi.waitFor` or `await flushPromises()`.  The mock returns `undefined` or the wrong value: the query fired when it was created, before the test arranged the mock.
5. State leaking between tests: fresh Pinia and query cache per test (`runInSetup`, `mountWithUi`, `installTestPinia`; never `createTestingPinia`), unmount and clear the body, restore global stubs in `afterEach` with `vi.unstubAllGlobals()`.
6. A confirmation or toast never appears: the component needs `UApp` (`AppHarness`).
7. In the browser, wrong data: is the right function called with the right arguments (a test with the mock asserting the call tells you), and does the query key hold every input?  Wrong render: check the four-state priority (`ui-states.md`) and which query flag (`isPending`, `isLoading`, `isPlaceholderData`) drives it.
8. A route or component is "not found" by the editor: `npm run build` once regenerates `typed-router.d.ts`, `components.d.ts`, and `auto-imports.d.ts`.

## Playwright MCP: look at the real UI

Tests prove logic; they do not prove the UI looks and works right.  For any UI work:

1. **Get browser access first**, before writing code: confirm the Playwright MCP is connected (`.mcp.json`).  If the app needs a signed-in session, pause and ask the user to authenticate.
2. Get the front-end URL from `aspire describe` (the port is assigned) and open it.
3. Drive the feature: click, type, submit; try the empty, loading, and error states (stop the backend, send invalid input).
4. Check **dark mode and a narrow viewport**; read the browser console and network panel for errors and unexpected requests (none should go to `api.iconify.design`).
5. Read the **page snapshot** (the accessibility tree): every control should have a sensible role and name (`accessibility-responsive.md`).
6. Fix and re-check, then finish with the hand-off checklist in `recipes.md`.

## Test HTTP behavior with MSW

This test uses the real generated client and Colada.  See [mocking.md](mocking.md) for the server and fixture lifecycle.

`src/app/src/queries/session.msw.test.ts`

```ts
import { useQuery } from "@pinia/colada";
import { HttpResponse } from "msw/http";
import { expect, it, vi } from "vitest";
import { getMeHandler } from "@/api/generated/handlers/getMeHandler";
import { server } from "@/test/server";
import { runInSetup } from "@/test/mount";
import { sessionQuery } from "./session";

it("treats a 401 as a signed-out session", async () => {
  server.use(getMeHandler(() => new HttpResponse(null, { status: 401 })));
  const { result, wrapper } = runInSetup(() => useQuery(sessionQuery));
  await vi.waitFor(() => expect(result.status.value).toBe("success"));
  expect(result.data.value).toBeNull();
  wrapper.unmount();
});
```

## Notes query through MSW

The handler observes the request from the real generated client.  `runInSetup` installs a real Pinia and query cache.  See [mocking.md](mocking.md).

`src/app/src/queries/notes.msw.test.ts (example)`

```ts
import { useQuery } from "@pinia/colada";
import { HttpResponse } from "msw/http";
import { expect, it, vi } from "vitest";
import { listNotesHandler } from "@/api/generated/handlers/listNotesHandler";
import { runInSetup } from "@/test/mount";
import { server } from "@/test/server";
import { notesQuery } from "./notes";

it("loads notes through the generated client", async () => {
  const requests: string[] = [];
  server.use(
    listNotesHandler(({ request }) => {
      requests.push(new URL(request.url).searchParams.get("notebookId") ?? "");
      return HttpResponse.json({ items: [] });
    }),
  );
  const { result, wrapper } = runInSetup(() => useQuery(notesQuery("notebook-1")));
  await vi.waitFor(() => expect(result.data.value).toEqual({ items: [] }));
  expect(requests).toEqual(["notebook-1"]);
  wrapper.unmount();
});
```
