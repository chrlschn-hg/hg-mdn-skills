# Lists, Tables, Search, and Pagination

A list page has four jobs: **show** rows, **narrow** them (filter, search), **page** through them, and **keep them fresh**.  Decide first *where* each job happens:

| Job | In the browser | In the backend |
|---|---|---|
| Filter by a field (notebook, status) | only for small, fully loaded data | **preferred**: the backend filters (`?notebookId=`), the filter is part of the query key |
| Free-text search | fine for hundreds of rows (`useClientList`) | needed for thousands, or when search covers fields not loaded |
| Pagination | fine for hundreds of rows | needed for thousands (cursor or offset in the API) |
| Sorting | fine for hundreds of rows | needed when paging server-side (sort must precede the page) |

Rule of thumb: **filter by an entity relationship in the backend; search and page in memory until the data outgrows it.**  When a list can reach thousands of rows, add paging to the API first (an OpenAPI change), regenerate, and put the page in the query key: the component code below barely changes.

## The page

The notes page composes everything: a backend filter (notebook select), debounced in-memory search, the four states, a table with pagination, a creation slideover, an inline delete, and optional auto-refresh.

`src/app/src/pages/notes/index.vue`

```vue
<script setup lang="ts">
import type { TableColumn } from "@nuxt/ui";
import { useQuery } from "@pinia/colada";
import { errorText } from "@/api/errors";
import NoteFormSlideover from "@/components/NoteFormSlideover.vue";
import { useAutoRefresh } from "@/composables/useAutoRefresh";
import { useClientList } from "@/composables/useClientList";
import { notebooksQuery } from "@/queries/notebooks";
import { notesQuery, useCreateNote, useDeleteNote, type NoteDraft } from "@/queries/notes";
import { formatDateTime } from "@/utils/format";

definePage({ meta: { title: "Notes" } });

const toast = useToast();

// `ALL` stands for "no filter".  A string sentinel is unambiguous (`null` is also how the select
// says "nothing chosen") and must sit outside the id alphabet, so no notebook id can equal it.
const ALL = "__all__";
const notebookFilter = ref(ALL);
const selectedNotebookId = computed(() =>
  notebookFilter.value === ALL ? null : notebookFilter.value,
);

// The filter is in the key: each notebook has its own cache entry, and a late response for the
// old filter cannot overwrite the new one (`data-loading.md`).  No watch, no `onMounted`.
const notesList = useQuery(() => notesQuery(selectedNotebookId.value));
const { data, error, isPending, isLoading, isPlaceholderData, refetch } = notesList;
const { enabled: autoRefresh } = useAutoRefresh(notesList);
const { data: notebookList } = useQuery(notebooksQuery);
const { create, saving } = useCreateNote();
const { remove } = useDeleteNote();

const notes = computed(() => data.value?.items ?? []);
const notebooks = computed(() => notebookList.value?.items ?? []);
const loadError = computed(() => errorText(error.value, "Could not load notes."));

const selectedNotebook = computed(() =>
  notebooks.value.find((n) => n.id === selectedNotebookId.value),
);
const notebookItems = computed(() => [
  { label: "All notebooks", value: ALL },
  ...notebooks.value.map((notebook) => ({ label: notebook.name, value: notebook.id })),
]);

// Search and pagination over the loaded notes; the backend does the notebook filter.
const { search, page, pageSize, total, pageItems } = useClientList(notes, {
  searchText: (note) => `${note.title} ${note.body ?? ""}`,
  pageSize: 8,
});

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
  { id: "actions", header: "" },
];

const slideoverOpen = ref(false);
const slideover = useTemplateRef("slideover");

/** Saves into the selected notebook.  A field error stays in the form; anything else is a toast. */
async function onSubmit(draft: NoteDraft) {
  const result = await create({ ...draft, notebookId: selectedNotebookId.value });

  if (result.ok) {
    slideover.value?.reset();
    slideoverOpen.value = false;
    toast.add({ title: "Note added", icon: "i-hugeicons-checkmark-circle-02", color: "success" });
    return;
  }

  if (!slideover.value?.showFailure(result)) {
    toast.add({ title: "Could not save the note", description: result.message, color: "error" });
  }
}

/** The row leaves at once (`useDeleteNote` is optimistic); a failure puts it back. */
async function onDelete(id: string) {
  const result = await remove(id);

  if (!result.ok) {
    toast.add({ title: "Could not delete the note", description: result.message, color: "error" });
  }
}
</script>

<template>
  <section class="space-y-6">
    <header class="flex flex-wrap items-center justify-between gap-3">
      <h1 class="text-2xl font-semibold text-highlighted">Notes</h1>
      <UButton icon="i-hugeicons-plus-sign" @click="slideoverOpen = true">New note</UButton>
    </header>

    <div class="flex flex-wrap items-center gap-3">
      <USelect v-model="notebookFilter" :items="notebookItems" class="w-48" aria-label="Notebook" />
      <SearchBox v-model="search" label="Search notes" placeholder="Search notes" />
      <div class="ml-auto flex items-center gap-3 text-sm text-muted">
        <USwitch v-model="autoRefresh" label="Auto-refresh" />
        <UButton
          icon="i-hugeicons-reload"
          color="neutral"
          variant="ghost"
          aria-label="Refresh"
          :loading="isLoading"
          @click="refetch()"
        />
      </div>
    </div>

    <AsyncContent
      :loading="isPending"
      :error="loadError"
      :empty="!total"
      empty-title="No notes"
      :empty-description="search ? 'No note matches your search.' : 'Add your first note.'"
    >
      <!-- Dim the previous notebook's rows while the new one loads. -->
      <UTable :data="rows" :columns="columns" :class="{ 'opacity-60': isPlaceholderData }">
        <template #title-cell="{ row }">
          <RouterLink :to="`/notes/${row.original.id}`" class="font-medium text-highlighted">
            {{ row.original.title }}
          </RouterLink>
        </template>
        <template #notebookName-cell="{ row }">
          <UBadge
            v-if="row.original.notebookName"
            :label="row.original.notebookName"
            variant="subtle"
          />
          <span v-else class="text-muted">None</span>
        </template>
        <template #actions-cell="{ row }">
          <PopConfirm
            title="Delete this note?"
            :body="`'${row.original.title}' will be deleted.`"
            icon="i-hugeicons-delete-02"
            color="neutral"
            variant="ghost"
            aria-label="Delete note"
            @confirm="onDelete(row.original.id)"
          />
        </template>
      </UTable>

      <UPagination
        v-if="total > pageSize"
        class="mt-4"
        :page="page"
        :items-per-page="pageSize"
        :total="total"
        @update:page="page = $event"
      />
    </AsyncContent>

    <NoteFormSlideover
      ref="slideover"
      v-model:open="slideoverOpen"
      :saving="saving"
      :description="
        selectedNotebook ? `Filed in ${selectedNotebook.name}` : 'Not filed in a notebook'
      "
      @submit="onSubmit"
    />
  </section>
</template>
```

Points worth copying:

- **The backend filter is in the query key:** `useQuery(() => notesQuery(selectedNotebookId.value))` loads on mount and on every change; no `watch`, no `onMounted`.  The newest key wins over a slow older response (`data-loading.md`).
- **Flags drive the states:** `isPending` feeds `AsyncContent :loading` (skeletons on the first load only), `isLoading` spins the Refresh button, `isPlaceholderData` dims the previous notebook's rows.  The button calls `refetch()`; `useAutoRefresh(notesList)` calls `refresh()` on its timer.
- **Writes are mutation composables:** `useCreateNote().create` and `useDeleteNote().remove` return a `SaveResult` and refetch the list; the delete removes the row at once (`data-loading.md`, "Optimistic updates").  `PopConfirm` asks first (`overlays.md`).
- **Search and paging are `useClientList`** over `notes`; the page reads `pageItems`, `total`, `page`, `pageSize`.
- **Destructure composable results** (`const { search, page, pageItems } = useClientList(...)`): refs only unwrap in templates when they are top-level bindings.
- **`UTable` renders** `:data="rows"` (a computed view model, `components-composables.md`) with TanStack-style `columns`; custom cells use the `#<accessorKey>-cell` slot (`#title-cell="{ row }"`, the record is `row.original`).
- **Row links** use `RouterLink` (typed routes), not click handlers.
- **`UPagination` only when needed** (`v-if="total > pageSize"`).

## `useClientList`

`src/app/src/composables/useClientList.ts`

```ts
type ClientListOptions<T> = {
  /** Text the search box matches against, for one item. */
  searchText: (item: T) => string;
  pageSize?: number;
  /** Pause after the last keystroke before the filter applies. */
  debounceMs?: number;
};

/**
 * Search and pagination over a list that is already in memory: the right tool for hundreds of
 * rows.  (Thousands of rows, or data the backend must filter, need server-side paging instead.)
 *
 * `search` is bound to the input; `query` is the debounced value the filter uses, so typing does
 * not re-filter on every keystroke.  `ref`, `computed`, `watch`, `refDebounced`, `toValue` and
 * `MaybeRefOrGetter` are auto-imported.
 */
export function useClientList<T>(
  items: MaybeRefOrGetter<readonly T[]>,
  options: ClientListOptions<T>,
) {
  const search = ref("");
  const query = refDebounced(search, options.debounceMs ?? 250);
  const page = ref(1);
  const pageSize = ref(options.pageSize ?? 10);

  const filtered = computed(() => {
    const all = toValue(items);
    const needle = query.value.trim().toLowerCase();

    return needle
      ? all.filter((item) => options.searchText(item).toLowerCase().includes(needle))
      : all;
  });

  const total = computed(() => filtered.value.length);
  const pageCount = computed(() => Math.max(1, Math.ceil(total.value / pageSize.value)));
  const pageItems = computed(() => {
    const start = (page.value - 1) * pageSize.value;
    return filtered.value.slice(start, start + pageSize.value);
  });

  // A new search starts at page 1; a shrinking list must not strand the user on an empty page.
  watch(query, () => (page.value = 1));
  watch(pageCount, (count) => {
    if (page.value > count) {
      page.value = count;
    }
  });

  return { search, query, page, pageSize, total, pageCount, pageItems };
}
```

- `search` is bound to the input; `query` is its **debounced** copy (`refDebounced`, 250 ms) that the filter uses, so typing does not re-filter per keystroke.
- A new search returns to **page 1**; a list that shrinks below the current page moves back to the last page (never strand the user on an empty page).
- Pure and testable with fake timers (`testing.md`); no component involved.
- To **sort** in memory, sort the filtered array before slicing (inside `filtered`) and let a header control set the key; do not rely on `UTable`'s own sorting with manual paging, which would sort only the visible page.

## Search input

`SearchBox` is a drop-in `v-model` component (listing in `components-composables.md`, "Two-way values"); the clear button shows only when there is text.  Always give the input an accessible name (`label`/`aria-label`).

## Tables with `UTable`

```ts
const columns: TableColumn<NoteRow>[] = [
  { accessorKey: "title", header: "Title" },
  { accessorKey: "notebookName", header: "Notebook" },
  { accessorKey: "createdLabel", header: "Created" },
];
```

- `columns` are TanStack column definitions: `accessorKey` (a key of the row type, checked by TypeScript), `header` (string or render function).  `TableColumn` comes from `@nuxt/ui`.
- Cells default to the raw value; prepare dates, money, and ids in a view-model `computed` (`components-composables.md`) with a util (`formatDateTime`).
- Use `UTable` for tabular data with several attributes per row; use a plain list of `UCard`s for rich, variable rows, and `UTabs`/`USelect` to switch between views of the same data.
- Check wide tables at phone width: show fewer columns or allow horizontal scrolling (`accessibility-responsive.md`).
- Timestamps arrive as ISO strings: format at the edge, never in the query:

`src/app/src/utils/format.ts`

```ts
/** Formats an ISO timestamp (or Date) for display: "Oct 9, 2026, 2:35 PM". */
export function formatDateTime(
  value: string | Date,
  options: { locale?: string; timeZone?: string } = {},
): string {
  return new Intl.DateTimeFormat(options.locale, {
    dateStyle: "medium",
    timeStyle: "short",
    timeZone: options.timeZone,
  }).format(new Date(value));
}
```

## Selection, bulk actions, and column visibility

`useClientList` covers search and paging.  When a table needs **row selection, bulk actions, or a column chooser**, use `UTable`'s TanStack state (`v-model:row-selection`, `v-model:column-visibility`, `v-model:column-filters`, `v-model:pagination`, and `useTemplateRef("table")` for `tableApi`).  The Nuxt UI `dashboard` template's customers page is the reference for that layout (`nuxt-ui-templates.md`, "Listing view").  Keep writes in **mutation composables** (delete the selected ids through one, then let it invalidate the list) and confirm destructive ones.

## Server-side paging (when the data outgrows memory)

A sketch; no example in the references backs it.

1. Add paging to the endpoint (`?cursor=` / `?pageSize=`, a `NextCursor` in the response) and **describe it** in `WithDescription`; regenerate the client.
2. The page (or cursor) is part of the query key: `[...notesKey, { notebookId, page }]`.  Set `placeholderData: (previous) => previous` to keep the rows on screen while the next page loads.
3. `UPagination`'s `page` is a `ref` the key reads; `total` comes from the response (`data.value?.total`).  `@update:page` sets the `ref`; the query follows.
4. Search and sort become query parameters, debounced (`refDebounced`), in the key; each change resets to the first page.
5. No stale-response guard to write: each page has its own key, so a slow page-3 response cannot replace a newer page-1 result.

## Reordering (drag and drop)

When users reorder rows, the backend ranks them (`hg-mdn-backend`, `references/ids-and-ordering.md`).  Send the dragged row's **new neighbors** (`beforeId`, `afterId`: the neighbors' public ids, `null` at a list end) to the move endpoint, never an index or a rank; show the new order optimistically (`data-loading.md`, "Optimistic updates") and keep what the server returns: the moved row with its new rank, or the new order.  Two people moving rows into the same gap at once is fine: the order stays stable and a later move separates them.

## Rules

1. Every list renders the four states (`ui-states.md`).
2. Debounce anything typed that triggers work (`data-loading.md`).
3. Do not mutate query data from the page; call a mutation composable.
