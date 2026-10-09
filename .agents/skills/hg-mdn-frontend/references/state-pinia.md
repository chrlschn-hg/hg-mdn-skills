# State with Pinia

Pinia setup stores hold server state and shared UI state.  **Stores are the only code that calls the generated API client**: one place for loading, saving, and error state per resource, one boundary to mock in tests (`testing.md`), and a contract change that touches the store and the type checker rather than every component.  Components read store state and call store actions.

## The shape of a store

State and the flags a component binds to (`notes-store.ts`, the Notes example; `health-store.ts` ships in the baseline):

`src/app/src/stores/notes-store.ts (excerpt)`

```ts
export const useNotesStore = defineStore("notes", () => {
  const notes = ref<NoteResponse[]>([]);
  const loading = ref(false);
  const saving = ref(false);
  const error = ref<string | null>(null);
```

A write returns a `SaveResult` and updates state from the server's response; the setup function returns everything components need:

`src/app/src/stores/notes-store.ts (excerpt)`

```ts
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
```

Rules:

- **Setup syntax** (`defineStore("id", () => { ... })`): `ref` is state, `computed` is a getter, functions are actions.
- **Id** is a stable kebab-case string; the file is `<thing>-store.ts` and the export `use<Thing>Store`.
- **One store per resource**, roughly one per API tag (`notes-store.ts` for the Notes tag).  The signed-in user is the shipped `session-store.ts`; a feature's UI state goes in that feature's store.
- **Expose** `loading` and `error` (a message string or `null`) for loads, and a separate `saving` when the UI shows a write in progress.  Components bind to these (`:loading="saving"`, `<UAlert v-if="error">`).
- **Actions record errors; they do not throw.**  Loads set `error`.  A write the caller must react to returns a `SaveResult` (`api-client.md`), so a form can show the backend's message on the right field (`forms.md`).  Throw only when the caller cannot continue (for example a failed startup load a guard depends on).
- **Reset `error` at the start** of each action; clear `loading`/`saving` in `finally`.
- **Use the backend's message** for expected failures (`ApiError`), a fallback for everything else: `errorMessage(err, "Could not save the note.")`.
- **Concurrency** (stale responses, shared in-flight requests, optimistic updates): `data-loading.md`.
- **HMR:** end each store file with the `acceptHMRUpdate` block (copy it from `health-store.ts`).  Stores import `defineStore` and `acceptHMRUpdate` from `pinia` explicitly; `ref`, `computed`, and `storeToRefs` are auto-imported.
- **Persisted preferences** use VueUse's `useStorage` inside the store with namespaced keys (`<feature>:<thing>`); never tokens or secrets, and treat stored data as untrusted (it may come from an older version).
- A store may use another store **inside an action or computed**, never at module top level; avoid cycles.

## Using a store in a component

```vue
<script setup lang="ts">
import { useNotesStore } from "@/stores/notes-store";

const store = useNotesStore();
// Destructure state with storeToRefs (keeps reactivity); actions come straight off the store.
const { notes, loading, error } = storeToRefs(store);

onMounted(() => store.load());
</script>
```

- Destructure **state and getters** with `storeToRefs`; call **actions** on the store (`store.load()`).
- Load in `onMounted` when the input is fixed; when it can change (a filter, a route param) use `watch(source, load, { immediate: true })` instead, never both.  A router guard loads data a page cannot render without.
- **Pages and the app shell (layout, router guards) consume stores; everything else gets props** and emits events, so it stays testable and reusable (`components-composables.md`).

## What does not belong in a store

- Form field state for a single form (local `ref`s in the form component: `forms.md`).
- Pure functions (`utils/`), and reusable stateful behavior without a server resource (a composable: `components-composables.md`).
- Anything the router already holds (`route.params`, `route.query`).
