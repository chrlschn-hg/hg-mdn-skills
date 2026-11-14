# Client State with Pinia

**Pinia holds client state: what the browser knows and the server does not.**  Selection, view preferences, a wizard's draft, the open panel.  **Server state** (anything the backend owns: lists, records) belongs to Pinia Colada queries, not to a store: `data-loading.md`.  Colada's cache is itself a Pinia store, so the two share one `createPinia()` instance (`main.ts` installs both).

| It is | It lives in |
|---|---|
| a list or record from the API | a query in `src/queries/<resource>.ts` (`data-loading.md`) |
| the signed-in user | `session-store.ts`, the one store that wraps a query (below) |
| a selection, a filter the user set, a view mode, a multi-step draft | a Pinia setup store, or a local `ref` when only one component needs it |
| a form's field values | local `ref`s in the form component (`forms.md`) |
| the URL's params and query string | the router (`route.params`, `route.query`) |

Two rules keep the split clean:

- **Never copy query data into a store `ref`.**  A copy goes stale, and the cache already shares one entry between components.  Derive with `computed` from the query's `data`.
- **A store never calls the generated client.**  Reads are queries; writes are mutations (`data-loading.md`).  Only `src/queries` imports a client (and `session-store.ts` through `queries/session.ts`).

## The shape of a store

A setup store: `ref` is state, `computed` is a getter, a function is an action.

`src/app/src/stores/notes-view-store.ts (example)`

```ts
import { acceptHMRUpdate, defineStore } from "pinia";

/** How the Notes page is laid out and what is selected.  Server data comes from `queries/notes.ts`. */
export const useNotesViewStore = defineStore("notes-view", () => {
  // Persisted per browser (VueUse `useStorage`, auto-imported); namespaced key `<feature>:<thing>`.
  const layout = useStorage<"table" | "cards">("notes:layout", "table");
  const selectedId = ref<string | null>(null);

  function select(id: string | null) {
    selectedId.value = id;
  }

  return { layout, selectedId, select };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useNotesViewStore, import.meta.hot));
}
```

Rules:

- **Setup syntax** (`defineStore("id", () => { ... })`).  Return everything components use.
- **Id** is a stable kebab-case string; the file is `<thing>-store.ts` and the export `use<Thing>Store`.
- **Keep a store small and named for the UI concern** (`notes-view`, `checkout-draft`), not for a resource: the resource is a query.  Prefer a local `ref` or a composable until two unrelated components need the same state.
- **Persisted preferences** use `useStorage` with namespaced keys (`<feature>:<thing>`); never tokens or secrets, and treat stored data as untrusted (an older version may have written it).
- **HMR:** end each store file with the `acceptHMRUpdate` block (copy it from `session-store.ts`).  Stores import `defineStore` and `acceptHMRUpdate` from `pinia` explicitly; `ref`, `computed`, and `storeToRefs` are auto-imported.
- A store may use another store **inside an action or computed**, never at module top level; avoid cycles.

## Using a store in a component

```vue
<script setup lang="ts">
import { useNotesViewStore } from "@/stores/notes-view-store";

const view = useNotesViewStore();
// Destructure state with storeToRefs (keeps reactivity); actions come straight off the store.
const { layout, selectedId } = storeToRefs(view);
</script>
```

- Destructure **state and getters** with `storeToRefs`; call **actions** on the store (`view.select(id)`).
- **Pages and the app shell (layout, router guards) read stores; everything else gets props** and emits events, so it stays testable and reusable (`components-composables.md`).

## The session store: the one store that owns a query

The signed-in user lives as long as the app runs, and the router guard needs it outside any component.  `session-store.ts` (it ships in the baseline) wraps `useQuery(sessionQuery)` and exposes `user`, `checked`, `isSignedIn`, `isAdmin`, `error`, `load()`, and `signedOut()`.  `queries/session.ts` turns a 401 into `null` data, so "nobody signed in" is a normal result and not an error.

- `load()` joins the check in flight; `load({ force: true })` starts a new one (after signing in elsewhere).  The guard calls `load()`, so every route change shares one request.
- `signedOut()` cancels a check in flight, then writes `null` into the cache (`setQueryData`): the API answered 401 mid-session, or the user signed out.  The cancel matters: without it a response that left before sign-out writes the old user back.
- A failure other than 401 (a 500, the network) keeps the previous user: a blip on the server must not sign someone out.
- Do not add another store like it.  A page that needs a resource calls `useQuery` itself.

## What does not belong in a store

- Anything the backend owns (`data-loading.md`) or the router holds (`route.params`, `route.query`).
- Form field state for a single form (local `ref`s in the form component: `forms.md`).
- Pure functions (`utils/`), and reusable stateful behavior without shared state (a composable: `components-composables.md`).
