# UI States and Feedback

Every load has four user-visible states: **loading**, **error**, **empty**, **content**.  A write adds feedback (a toast or a field error).  A missing state is the usual bug in data pages: a blank screen while loading, a spinner that never ends after an error, an empty table with no explanation.

## The four states: `AsyncContent`

One component renders them in the right priority so pages do not repeat the `v-if` chain (it ships in the baseline):

`src/app/src/components/AsyncContent.vue`

```vue
<script setup lang="ts">
/**
 * Renders the four states of anything that loads: error, loading, empty, or content.  Pages use
 * it instead of repeating the same `v-if` chain; each state has a sensible default that a slot
 * can replace.
 */
withDefaults(
  defineProps<{
    loading: boolean;
    /** A message to show instead of the content; `null` when there is no error. */
    error?: string | null;
    /** True when the load finished and there is nothing to show. */
    empty: boolean;
    emptyTitle?: string;
    emptyDescription?: string;
    emptyIcon?: string;
    skeletonRows?: number;
  }>(),
  {
    error: null,
    emptyTitle: "Nothing here yet",
    emptyDescription: undefined,
    emptyIcon: "i-hugeicons-folder-01",
    skeletonRows: 3,
  },
);

defineSlots<{
  default(): unknown;
  loading?(): unknown;
  empty?(): unknown;
}>();
</script>

<template>
  <UAlert
    v-if="error"
    color="error"
    variant="subtle"
    icon="i-hugeicons-alert-02"
    :title="error"
    role="alert"
  />

  <div v-else-if="loading" class="space-y-3" aria-busy="true" aria-label="Loading">
    <slot name="loading">
      <USkeleton v-for="row in skeletonRows" :key="row" class="h-12 w-full" />
    </slot>
  </div>

  <slot v-else-if="!empty" />

  <slot v-else name="empty">
    <UEmpty :icon="emptyIcon" :title="emptyTitle" :description="emptyDescription" />
  </slot>
</template>
```

Priority: **error**, then **loading**, then **content**, then **empty**.  A failed load never shows a spinner forever or stale emptiness.

```vue
<AsyncContent
  :loading="firstLoad"
  :error="error"
  :empty="!total"
  empty-title="No notes"
  :empty-description="search ? 'No note matches your search.' : 'Add your first note.'"
>
  <UTable :data="rows" :columns="columns" />
</AsyncContent>
```

- **`firstLoad`** is a `computed` (`loading && !notes.length`): skeletons only for the *first* load.  On a refresh the rows stay on screen while the new data arrives (no flash of skeletons under auto-refresh).
- **`:empty`** is true when the load finished and there is nothing to show.  Make the message depend on *why* it is empty (no data yet vs. nothing matches the search): `UEmpty`'s `description` and `actions` (a "Create" button) turn a dead end into the next step.
- **Slots replace the defaults** when the page needs something specific: `#loading` (skeleton shaped like the content), `#empty` (a custom call to action).
- It is generic over content: tables, lists, cards, detail panels.
- **An error replaces the content with the alert**, even when rows are already loaded.  To keep the last good rows visible through a failed refresh, pass `:error="null"` and render a `UAlert v-if="error"` above `AsyncContent` yourself; decide per page whether stale rows are acceptable.

The shipped `AsyncContent.test.ts` covers each state; copy its shape for a new state component.

## Which Nuxt UI component for which state

| State | Component | Notes |
|---|---|---|
| Loading, first load | `USkeleton` (shaped like the content) | `aria-busy` on the wrapper; never a bare spinner for a whole page |
| Loading, an action | `loading` prop on `UButton` | the button disables itself while loading |
| Loading, refresh | keep the content; a small spinner or the button's `loading` | do not blank the page |
| Error, page-level | `UAlert color="error" variant="subtle"` with `role="alert"` | the store's `error` string |
| Error, field | the field's own message (`UFormField`) | see `forms.md` |
| Error, action | toast (`useToast`) | transient; the user is not blocked |
| Empty | `UEmpty` (icon, title, description, actions) | explain and offer the next step |
| Success, action | toast | brief, `color: "success"` |
| Confirm a destructive action | `useConfirm` (overlay) | see `overlays.md` |

## Feedback: toasts

`useToast()` (auto-imported) shows transient messages; `UApp` (in `App.vue`) hosts them.  Use them for the result of an action the user just took, not for page state.

```ts
const toast = useToast();

toast.add({ title: "Note added", icon: "i-hugeicons-checkmark-circle-02", color: "success" });
toast.add({ title: "Could not save the note", description: result.message, color: "error" });
```

- **Success:** short title, `color: "success"`, a literal hugeicons icon (so it is bundled).
- **Failure:** `color: "error"` and the backend's message as `description`; keep the title about the *action* ("Could not save the note"), not the cause.
- A failure that belongs to **one field stays on the field** (the form shows it); only failures with no field home become toasts (`forms.md`).
- Do not toast every successful load, and do not toast what the user can already see changed.
- Toasts disappear: anything the user must act on goes in the page (`UAlert`) or a dialog.

## Rules

1. Every data-loading page uses `AsyncContent` (or renders all four states by hand with the same priority).
2. Stores expose `loading`, `error`, and (for writes) `saving`; **components never invent their own flags** for the same thing (`state-pinia.md`).
3. Disable and spin the control that started the action (`:loading="saving"`); never leave a double-submit possible.
