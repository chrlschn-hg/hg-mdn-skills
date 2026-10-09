# Front-end Testing

[Vitest](https://vitest.dev) with `happy-dom` and `@vue/test-utils` for unit, component, and wiring tests; the **Playwright MCP** for looking at the real running UI.  (Backend tests are TUnit: see the `hg-mdn-backend` skill.)  This file owns the test setup, the API mock boundary, page tests, fake timers, and browser verification; forms, overlays, routing, and the other topic references show their own tests.

## Setup

`vite.config.ts` carries the test config (it imports `defineConfig` from `vitest/config`):

```ts
test: {
  environment: "happy-dom",
  include: ["src/**/*.test.ts"],
  setupFiles: ["./src/test/setup.ts"],
  restoreMocks: true,
  clearMocks: true,
},
```

- Tests run through the same Vite config, so auto-imports (`ref`, `defineStore`, `storeToRefs`, ...), the `@/` alias, and typed routes work in them.
- `npm test` runs `vitest run` (after regenerating the API client: `pretest`).  Watch mode: `npx vitest`.  One file: `npx vitest run src/stores/notes-store.test.ts`; by name: `-t "puts a created note first"`.
- Tests sit beside the code: `thing.ts` and `thing.test.ts`.

## Test infrastructure (`src/test/`)

Three small files every test benefits from (they ship in the baseline):

`src/app/src/test/setup.ts`

```ts
import { afterEach, beforeEach, vi } from "vitest";

/**
 * Tests never reach a real backend.  Every API function a test touches must be mocked (the
 * generated clients, at the module boundary); anything that slips through would otherwise try
 * `localhost:<port>` and fail with a confusing `ECONNREFUSED`.  This turns it into a clear error.
 * Tests that exercise the real client stub `fetch` themselves (`vi.stubGlobal`), which wins.
 */
beforeEach(() => {
  vi.stubGlobal("fetch", (input: RequestInfo | URL) => {
    throw new Error(
      `Unexpected network request in a test: ${String(input)}. Mock the API function.`,
    );
  });
});

afterEach(() => vi.unstubAllGlobals());
```

`src/app/src/test/mount.ts`

```ts
import ui from "@nuxt/ui/vue-plugin";
import { mount, type ComponentMountingOptions } from "@vue/test-utils";
import { createPinia } from "pinia";
import type { Component } from "vue";

/**
 * `mount` with what every component here needs: the Nuxt UI plugin (theme and injections) and a
 * fresh Pinia.  Pass `attachTo: document.body` when the component teleports (slideovers, modals,
 * popovers) and the test queries `document.body`.
 */
export function mountWithUi<T extends Component>(
  component: T,
  options: ComponentMountingOptions<T> = {},
) {
  return mount(component, {
    ...options,
    global: { ...options.global, plugins: [ui, createPinia(), ...(options.global?.plugins ?? [])] },
  } as ComponentMountingOptions<T>);
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

- **`setup.ts` is a network guard.**  Any `fetch` a test did not stub throws `Unexpected network request in a test: <url>. Mock the API function.` instead of failing with a confusing `ECONNREFUSED`.  Tests that exercise the real client stub `fetch` themselves with `vi.stubGlobal`, which wins.
- **`mountWithUi`** is `mount` plus the Nuxt UI plugin and a fresh Pinia: what every component needs.
- **`AppHarness`** is a `UApp` host for anything that teleports or needs `UApp`: overlays opened through `useOverlay` (confirmations), toasts, `USlideover`/`UModal`/`UPopover`.  Mount the component in its default slot, with `attachTo: document.body`; the content is then in `document.body`, not in `wrapper`.
- Principle: **mock only the API (at the generated-client module); everything else is real.**

## What to test

| Subject | How | Example |
|---|---|---|
| **Stores** | mock the generated clients; assert state, flags, and errors | `stores/notes-store.test.ts` |
| **Composables** | call them directly with plain refs; fake timers for time | `composables/useAutoRefresh.test.ts` |
| **Utils** | plain input and output | any `utils/*.test.ts` |
| **Components with logic** | `mountWithUi`; props, events, slots | `components/SearchBox.test.ts`, `NoteForm.test.ts` (`forms.md`) |
| **Overlays** | `AppHarness` + `attachTo: document.body` (`overlays.md`, "Testing overlays") | `components/NoteFormSlideover.test.ts` |
| **API error mapping, client config** | real generated client, stub `fetch` | `api/errors.test.ts` |
| **Router guards** | memory router and `router.push` (`routing.md`) | `router/guards.test.ts` |
| **Pages and whole-app wiring** | mount `App` with router, Nuxt UI, Pinia; API mocked | `App.test.ts`, `pages/notes/index.test.ts` |

Test behavior a user or a caller can see.  Do not test Nuxt UI itself, and do not assert on markup details that a restyle would change.  Query by **text, role, or `aria-label`**, which doubles as an accessibility check.

Every page covers: the list, the **empty** state, the **error** state, the main action's success, and its failure.  Every store covers loading, error, success, and a stale or failing response.

## Mock at the generated-client boundary

Stores call generated functions, so that is the one seam to mock.  Use `vi.hoisted` so the mock exists before the module under test is imported:

`src/app/src/stores/notes-store.test.ts`

```ts
import { createPinia, setActivePinia } from "pinia";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { ResponseError } from "@/api/generated/.kubb/client";
import { useNotesStore } from "./notes-store";

// Mock the generated clients at the module boundary; stores never see HTTP.
const api = vi.hoisted(() => ({ listNotes: vi.fn(), createNote: vi.fn() }));
vi.mock("@/api/generated/clients/listNotes", () => ({ listNotes: api.listNotes }));
vi.mock("@/api/generated/clients/createNote", () => ({ createNote: api.createNote }));

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
    request: new Request("http://localhost/notes"),
    response: new Response(),
  });

describe("useNotesStore", () => {
  beforeEach(() => setActivePinia(createPinia()));

  it("loads notes", async () => {
    api.listNotes.mockReturnValue(ok({ items: [note("1")] }));

    const store = useNotesStore();
    await store.load();

    expect(store.notes).toEqual([note("1")]);
    expect(store.error).toBeNull();
    expect(store.loading).toBe(false);
  });

  it("asks the backend for one notebook's notes", async () => {
    api.listNotes.mockReturnValue(ok({ items: [] }));

    await useNotesStore().load("nb-1");

    expect(api.listNotes).toHaveBeenCalledWith({ query: { notebookId: "nb-1" } });
  });

  it("records the error message when loading fails", async () => {
    api.listNotes.mockReturnValue(fails(new Error("boom")));

    const store = useNotesStore();
    await store.load();

    expect(store.error).toBe("boom");
    expect(store.loading).toBe(false);
  });

  it("ignores a slow response for an older request", async () => {
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
    expect(store.loading).toBe(false);
  });

  it("puts a created note first and reports success", async () => {
    api.listNotes.mockReturnValue(ok({ items: [note("1")] }));
    api.createNote.mockReturnValue(ok(note("2")));

    const store = useNotesStore();
    await store.load();
    const result = await store.create({ title: "Newer" });

    expect(result).toEqual({ ok: true });
    expect(store.notes.map((n) => n.id)).toEqual(["2", "1"]);
    expect(store.saving).toBe(false);
  });

  it("returns the backend's error code and message when the API rejects the note", async () => {
    api.createNote.mockReturnValue(
      fails(
        rejection(400, { code: "title_too_long", message: "Title may be at most 200 characters." }),
      ),
    );

    const result = await useNotesStore().create({ title: "x" });

    expect(result).toEqual({
      ok: false,
      code: "title_too_long",
      message: "Title may be at most 200 characters.",
    });
  });

  it("returns a null code for a failure that is not an ApiError", async () => {
    api.createNote.mockReturnValue(fails(new Error("network down")));

    const result = await useNotesStore().create({ title: "x" });

    expect(result).toEqual({ ok: false, code: null, message: "network down" });
  });
});
```

- Return `{ unwrap: () => Promise.resolve(value) }` (success) or `{ unwrap: () => Promise.reject(error) }` (failure): exactly what the stores use.  Small helpers (`ok(value)`, `fails(error)`) keep tests readable.
- For failures build a real `ResponseError` (as above) so the error-mapping code is exercised and the `code` reaches the store.
- `setActivePinia(createPinia())` in `beforeEach`: a fresh store per test.
- Assert the **state after** the action (`store.notes`, `store.error`, `store.loading`), not that a function was called with particular arguments, unless the arguments are the point (a request body, a filter).
- Test the **concurrency** a store handles: a slow response arriving after a newer one, two callers sharing one request (`data-loading.md`).
- A test that needs HTTP behavior stubs `fetch` (`api-client.md`); the guard in `setup.ts` makes any other network call loud.

## Page tests: through the whole app

Unit tests miss broken wiring: a missing plugin, a bad route, a layout that throws.  A page test mounts the **real app** (router, layout, Nuxt UI, Pinia, stores) and mocks only the API.  Mount at a route:

`src/app/src/pages/notes/index.test.ts (excerpt)`

```ts
  const root = document.createElement("div");
  document.body.append(root);
  const router = createRouter({ routes, history: createMemoryHistory() });
  const app = createApp(App).use(createPinia()).use(router).use(ui);

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
      unwrap: () => Promise.resolve({ status: "Healthy", checkedAtUtc: "2026-10-09T00:00:00Z" }),
    });

    const { root, unmount } = await mountAt("/");
    await vi.waitFor(() => expect(root.textContent).toContain("Healthy"));
    await new Promise((resolve) => setTimeout(resolve, 100)); // let icon loading start

    expect(request).not.toHaveBeenCalled();
```

## Time, environment, and globals

- **Fake timers** for debounce, polling, and timeouts: `vi.useFakeTimers()` in `beforeEach`, `vi.useRealTimers()` in `afterEach`, and `await vi.advanceTimersByTimeAsync(ms)` (runs the timers *and* the promises they start; use it, not the sync version, whenever the callback is async).
- **Environment variables:** `vi.stubEnv("VITE_API_BASE_URL", "")` (Vitest restores them), as `api/errors.test.ts` does.
- **Browser APIs happy-dom lacks** (`matchMedia` layout, `ResizeObserver`): stub them in the test that needs them (`vi.stubGlobal`); `useIsMobile.test.ts` shows `matchMedia` (`accessibility-responsive.md`).
- Tests import generated modules like any other; `npm test` regenerates them first.

## Debugging a failing test or a UI problem

1. Run the one file: `npx vitest run <path> -t "<name>"`.
2. `Unexpected network request in a test: <url>`: mock that API function (or stub `fetch` if the real client is the subject).
3. `Cannot find ... in document`: the content teleported; query `document.body`, mount with `attachTo`.
4. Assertion before the data arrived: wrap in `vi.waitFor` or `await flushPromises()`.
5. State leaking between tests: fresh Pinia per test, unmount and clear the body, `afterEach` restores stubs (`restoreMocks`/`unstubAllGlobals` are on).
6. A confirmation or toast never appears: the component needs `UApp` (`AppHarness`).
7. In the browser, wrong data: is the right function called with the right arguments (a test with the mock asserting the call tells you), and is a stale response overwriting state?  Wrong render: check the four-state priority (`ui-states.md`) and which store flag drives it.
8. A route or component is "not found" by the editor: `npm run build` once regenerates `typed-router.d.ts`, `components.d.ts`, and `auto-imports.d.ts`.

## Playwright MCP: look at the real UI

Tests prove logic; they do not prove the UI looks and works right.  For any UI work:

1. **Get browser access first**, before writing code: confirm the Playwright MCP is connected (`.mcp.json`).  If the app needs a signed-in session, pause and ask the user to authenticate.
2. Get the front-end URL from `aspire describe` (the port is assigned) and open it.
3. Drive the feature: click, type, submit; try the empty, loading, and error states (stop the backend, send invalid input).
4. Check **dark mode and a narrow viewport**; read the browser console and network panel for errors and unexpected requests (none should go to `api.iconify.design`).
5. Read the **page snapshot** (the accessibility tree): every control should have a sensible role and name (`accessibility-responsive.md`).
6. Fix and re-check, then finish with the hand-off checklist in `recipes.md`.
