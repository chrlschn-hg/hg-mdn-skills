# Components and Composables

Where behavior goes: **stores** own server state, **composables** own reusable stateful behavior, **components** own presentation, **utils** own pure functions.  Where files live, SFC order, styling, and the template rule: `project-structure.md`.

## Data flow: the page is the hub

State flows **down as props** and decisions flow **up as events**; the page (the route component) is where state meets the stores.

```text
store state --storeToRefs--> page ----props----> child component
                              ^  |                   |
        calls store actions --+  +-- v-model --------+  (defineModel: two-way)
                              +--------- emit --------+  (a user decision: "submit", "confirm")
```

- **Pages and the app shell (layout, router guards) consume stores** (`storeToRefs` for state, actions straight off the store); every other component gets props.  The page handles events that need a store and passes state down.
- **Events, not callbacks:** `emit("submit", draft)`, never `:onSave="save"`: events are typed in the component's signature and compose with `v-model`.
- **Two-way binding with `defineModel`** when a child edits a value the parent owns (`v-model="search"`, `v-model:open="slideoverOpen"`).
- **`defineExpose` is the one sanctioned downward command**, for results that arrive after an emit (a form's `showFailure()` and `reset()`); call it through a template ref (`useTemplateRef`).  Anything declarative is a prop.
- Split a page or component when it mixes concerns (a filter bar, a table, a form): extract a child or a composable.
- **The notes page is the canonical example** (`lists-tables.md`): it reads two stores, owns the filter state, passes rows and flags to `AsyncContent`, `UTable`, and the slideover, and handles the slideover's `submit` event.

## Components

Components in `src/components/` are auto-registered (typed in `components.d.ts`: no import); pages and layouts import theirs.  Name a component for what it is (`NoteForm`, `NoteList`) with a feature prefix.

- One responsibility per component.
- `v-for` always has a stable `:key` (a real id, never the index for mutable lists).
- No direct DOM access; use template refs (`useTemplateRef`) and VueUse (`useElementSize`, `onClickOutside`, ...).
- Prefer Nuxt UI's `v-model` bindings (`v-model:open`, `v-model="value"`) over manual prop/emit pairs.
- Props are read-only: copy to local state (`ref(props.value)`) or emit; never mutate.
- **Slots over props for content**, and Nuxt UI parts for layout.
- **A thin wrapper over a Nuxt UI component** (`PopConfirm`, full listing in `overlays.md`): `defineOptions({ inheritAttrs: false })` plus `useAttrs()` forwards the caller's `icon`, `color`, `variant`, `label` to the trigger `UButton`, so the wrapper adds behavior without re-declaring the button's API:

`src/app/src/components/PopConfirm.vue (excerpt)`

```vue
// Button attrs (icon, color, variant, disabled, ...) belong on the trigger; the confirm and
// cancel listeners are declared component events and are emitted below.
const triggerAttrs = useAttrs();
```

## View models: precompute what the template renders

A Vue `computed` is cached: it reruns only when its inputs change.  Calling a function from the template (`{{ formatDateTime(row.createdAtUtc) }}`, `v-if="notebookName(row.notebookId)"`) runs it on every render, for every row.  Project the data into the shape the template needs **once**, in a computed, and let the template just print it:

`src/app/src/pages/notes/index.vue (excerpt)`

```ts
/** What the table renders: the text is prepared here, once, so the template only prints it. */
type NoteRow = { id: string; title: string; notebookName: string | null; createdLabel: string };

const notebookNames = computed(() => new Map(notebooks.value.map((n) => [n.id, n.name])));

// A computed view model (cached, recomputed only when its inputs change) instead of calling
// `formatDateTime` and a notebook lookup from the template for every row on every render.
const rows = computed<NoteRow[]>(() =>
  pageItems.value.map((note) => ({
    id: note.id,
    title: note.title,
    notebookName: note.notebookId ? (notebookNames.value.get(note.notebookId) ?? null) : null,
    createdLabel: formatDateTime(note.createdAtUtc),
  })),
);

const columns: TableColumn<NoteRow>[] = [
  { accessorKey: "title", header: "Title" },
  { accessorKey: "notebookName", header: "Notebook" },
  { accessorKey: "createdLabel", header: "Created" },
];
```

- The **view model** is a small local `type` (`NoteRow`): the API type with display text already prepared (`createdLabel`, `notebookName`).  It is also a natural place for derived flags (`isOverdue`).
- Build lookups once (`notebookNames` is a `Map`) instead of searching an array per row.
- The columns and the cell slots then use the view model's keys (`#notebookName-cell`), and the template stays free of calls and lookups.
- Do the same for `v-for` lists: iterate a computed array of ready-to-print objects with a stable `:key`.

## Two-way values with `defineModel`

`defineModel` replaces the `modelValue` prop plus `update:modelValue` emit pair; the parent uses a plain `v-model`.  Name a model for components with several (`defineModel<boolean>("open", { required: true })` -> `v-model:open`).  `SearchBox` is a drop-in `v-model` input:

`src/app/src/components/SearchBox.vue`

```vue
<script setup lang="ts">
/**
 * A search input with a clear button.  `defineModel` is the two-way binding: the parent writes
 * `v-model="search"`, and assigning to `query` here updates the parent.  A single `defineModel()`
 * replaces the old `modelValue` prop plus `update:modelValue` emit pair.
 */
const query = defineModel<string>({ default: "" });

defineProps<{ label?: string; placeholder?: string }>();
</script>

<template>
  <UInput
    v-model="query"
    icon="i-hugeicons-search-01"
    :placeholder="placeholder ?? 'Search'"
    :aria-label="label ?? placeholder ?? 'Search'"
    class="w-64"
  >
    <template v-if="query" #trailing>
      <UButton
        icon="i-hugeicons-cancel-01"
        color="neutral"
        variant="link"
        size="sm"
        aria-label="Clear search"
        @click="query = ''"
      />
    </template>
  </UInput>
</template>
```

`src/app/src/components/SearchBox.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { mountWithUi } from "@/test/mount";
import SearchBox from "./SearchBox.vue";

describe("SearchBox", () => {
  it("updates the bound value as the user types", async () => {
    const wrapper = mountWithUi(SearchBox, { props: { modelValue: "" } });

    await wrapper.find("input").setValue("milk");

    expect(wrapper.emitted("update:modelValue")).toEqual([["milk"]]);
  });

  it("clears the value with the clear button, which only shows when there is text", async () => {
    const wrapper = mountWithUi(SearchBox, { props: { modelValue: "milk" } });

    await wrapper.find("[aria-label='Clear search']").trigger("click");

    expect(wrapper.emitted("update:modelValue")).toEqual([[""]]);
    await wrapper.setProps({ modelValue: "" });
    expect(wrapper.find("[aria-label='Clear search']").exists()).toBe(false);
  });
});
```

- Test it through the model: pass `modelValue`, interact, assert `emitted("update:modelValue")`; `setProps` simulates the parent updating.
- Type slots with `defineSlots<{ default(props: { item: T }): unknown }>()` when a component passes data to its slot.
- A form that edits an entity takes an optional `initial` prop: `forms.md`, "Create and edit in one form".

## Common mistakes

| Mistake | Instead |
|---|---|
| a child imports a store or the API client | the page passes state as props and handles the child's events |
| a callback passed as a prop (`:onSave="save"`) | `defineEmits` and `emit("save", ...)` |
| functions and lookups called from the template per row | a computed view model |
| mutating a prop (`props.items.push(...)`) | emit, or copy to local state |
| `.value` in the template | use the ref's name (refs unwrap in templates) |
| one component that loads, filters, edits, and renders | split into children and composables |
| custom CSS or heavy Tailwind to restyle a component | the component's props, slots, and `ui` prop |
| a `watch` that only derives a value | a `computed` |

## Composables

A composable is a function named `useThing` that uses Vue reactivity and returns refs and functions.  Use one when behavior is **stateful, reusable, and not a server resource** (that is a store).  Worked examples: `useClientList` (`lists-tables.md`), `useAutoRefresh` (`data-loading.md`), `useIsMobile` (`accessibility-responsive.md`).

- **Name** `useThing`; **file** `useThing.ts` in `src/composables/`, test beside it.
- **Options object in, plain object out.**  Accept `MaybeRefOrGetter<T>` for inputs that may be reactive and unwrap with `toValue()`; return refs, not unwrapped values.
- **Single purpose**; compose small composables rather than growing one.
- **No UI inside**: a composable returns state; the caller renders dialogs and chooses wording (a pending-action composable for save / discard / cancel works this way).
- **Clean up** side effects (`onScopeDispose`, or use VueUse building blocks, which clean up for you).
- Check [VueUse](https://vueuse.org/functions.html) first: `useStorage`, `useDebounceFn`, `useClipboard`, `useDark`, `useEventListener`, `useIntervalFn`, and more are already auto-imported.
- Document the non-obvious decision in the doc comment.

Pure, framework-free functions go in `src/utils/` (formatting, parsing, small domain helpers); unit test them directly, no Pinia or Vue needed.
