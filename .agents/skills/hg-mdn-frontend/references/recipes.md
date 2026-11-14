# Recipes

Checklists in order of work.  Each names the reference for detail and, where a Notes example exists, the listing to copy (reference examples, not files in a new project: `project-structure.md`).  Finish every recipe with the **Hand-off checklist** at the end.

## Add a page that lists things

1. **Backend contract exists** (endpoint with `WithName`/`WithSummary`/`WithDescription`); `mise run openapi` (any backend build) refreshed `src/api/<project>-api.json`; commit it; `npm run generate`.  Read the generated function's JSDoc.
2. **Queries** `queries/<thing>.ts`: `<thing>Key` (the root key), `<thing>Query` with every input in its key, and a mutation composable per write that returns a `SaveResult` (`data-loading.md`; copy `queries/health.ts` and add the filter and the writes from the `queries/notes.ts` listing).
3. **Page** `pages/<things>/index.vue`: `useQuery(() => <thing>Query(filter.value))` (no `onMounted` load and no `watch` that loads: the query follows its key), `<AsyncContent :loading="isPending" :error="loadError" :empty>` around a `UTable`/list; `definePage({ meta: { title } })` (`ui-states.md`, `lists-tables.md`).
4. **Navigation entry** in the app's navigation data (the `layouts/app-navigation.ts` listing in `nuxt-ui.md`, once you have a layout).
5. **Tests**: queries (loads, error, a changed input, the write's `SaveResult`), page through the app (list, empty, error), all with the API mocked (`testing.md`).

## Build a complex screen (dashboard, listing, inbox, settings)

1. **Find the closest view** in `nuxt-ui-templates.md` ("Key views from `dashboard`"): dashboard (widgets with shared filters), listing (filter bar, selection, bulk actions), inbox (list plus detail), settings (sub-navigation plus sectioned forms).  Calendar, editor, and chat are in the same file.
2. Open that template's demo with the Playwright MCP, then read the matching source (`pages/` and its `components/` folder).  Take the component composition; leave the data layer, mock data, and `i-lucide-*` icons.
3. Build it here in slices: the queries and their states first (`data-loading.md`), then the layout from Nuxt UI parts, then the interactions.  Reuse `AsyncContent` (shipped) and the generated Zod schemas rather than the template's equivalents; `useClientList` and `useConfirm` are reference examples to copy (`lists-tables.md`, `overlays.md`).
4. Put the selection or filter state where the template does (one ref in the page), but prefer **ids and route params** over whole objects so the screen survives a reload and a refresh.
5. Check the responsive behavior in the browser: the side panel becomes a slideover, the table stays usable at ~390 px (`accessibility-responsive.md`).

## Add a create form (in a slideover)

1. Confirm the backend request schema generated (`src/api/generated/zod/<operation>RequestSchema.ts`).
2. **Form component** with `UForm` + the generated schema + UI-only refinements; emits `submit` with a normalized draft; exposes `showFailure`, `reset`, `isDirty` (`forms.md`, the `NoteForm.vue` listing).
3. **Slideover component** around it with `defineModel("open")` and the unsaved-changes confirmation (`overlays.md`, the `NoteFormSlideover.vue` listing).
4. **Mutation composable** `useCreate<Thing>()` in `queries/<thing>.ts`: `create(draft): Promise<SaveResult>` that invalidates `<thing>Key` on settle and returns the backend's error `code` (`data-loading.md`, the `useCreateNote` listing).
5. **Page** opens the slideover and calls `create`; success -> `reset()`, close, success toast; failure -> `showFailure(result)`, or an error toast if no field matches.
6. **Tests**: form (valid, each rule, contract limit, `showFailure`), slideover (open, dirty close asks, confirm closes), page (adds a note; field error keeps it open).

## Add a detail page with a route parameter

A sketch; no tested example in the references backs it.

1. File `pages/<things>/[id].vue`; read `useRoute("/<things>/[id]")` (typed `params.id`).  Run `npm run build` once so `typed-router.d.ts` knows the route.
2. Query `<thing>Query(id)`, read with `useQuery(() => <thing>Query(String(route.params.id)))` (`data-loading.md`).  An unknown id is an **expected** failure: show a `UEmpty` ("Not found", link back), not an error alert; use the backend's error code (`toApiError(error.value)?.code`) to tell it from a real failure.
3. Navigating between two ids reuses the component: the id is in the query key, so the query follows the route.  Write no `watch` and no `onMounted`.
4. Link from the list with `RouterLink :to="`/<things>/${id}`"`.
5. Test the page for a found id, an unknown id, and a param change (`routing.md`).

## Edit an existing item (reuse the form)

1. Give the form an optional `initial` prop (`forms.md`, "Create and edit in one form"); pass `:initial` through the slideover, which titles itself for editing.
2. Mutation composable `useUpdate<Thing>()`: `update(id, draft): Promise<SaveResult>` that invalidates `<thing>Key` on settle (the rows then show what the **server** saved) and returns the backend's error `code`.
3. The page keeps the selected item's **id** (not a copy), derives `initial` from the list query's `data` with a computed, opens the slideover, and on success closes it and toasts; a field error stays in the form.
4. Tests: the form fills from `initial` and is clean until changed; saving emits the draft; the page updates the row and keeps the form open on a field error.

## Add a public page, or an admin-only area

- **Public:** `definePage({ meta: { title: "...", public: true } })`; only the login page and similar.  The backend must agree (`AllowAnonymous()` for any API the page calls, with its lint list updated).
- **Admin-only:** a parent `pages/admin.vue` with `meta: { requiresAdmin: true }` that renders `<RouterView />`, children in `pages/admin/` (do not repeat the flag), the API as an `IAdminEndpoint` slice in the backend, and the navigation entry hidden from non-admins.  Guard tests: `routing.md`.

## Add a destructive action

1. Backend delete endpoint and generated client; a mutation composable `useDelete<Thing>()` with `remove(id): Promise<SaveResult>`: the row leaves the cached lists at once, and a failed delete puts it back (the undo in `onError`, then the refetch on settle reconciles; `data-loading.md`, "Optimistic updates", the `useDeleteNote` listing).
2. In the page, confirm first: `useConfirm` (`overlays.md`), or `PopConfirm` in a dense list.
3. Toast the result; a failed delete puts the row back (the undo in `onError`, then the refetch on settle).
4. Test: cancel does nothing, confirm calls `remove`, a failure restores the row.

## Handle a new backend error code

1. The backend documents it in the endpoint's `WithDescription` and returns `ApiError(code, message)`; regenerate.
2. If it belongs to a field, add it to the form's `fieldForCode`; otherwise the page toasts `result.message`.
3. If the UI must behave differently for it (redirect, disable a button), switch on `result.code` in the page: never on the message text.
4. Add a test of the mutation composable: the API mock rejects with a `ResponseError` and the write resolves to `{ ok: false, code }`; add a form/page test for the visible effect.

## React to a backend contract change

1. `mise run openapi` (the backend build rewrites the committed document; commit it), `npm run generate`, then `npm run build`: `vue-tsc` lists every call site and component that no longer fits.
2. Fix `src/queries` first (types flow from there; a new input to a read goes in its query key), then pages and forms; a changed limit needs no edit (the Zod schema regenerated).
3. Update mocks in tests (`vue-tsc` flags mismatches in typed fixtures); add a test for any new rule or status.
4. Read the new JSDoc: the behavior description may have changed.

## Add auto-refresh to a page

Copy `useAutoRefresh` (`data-loading.md`): `const query = useQuery(...)`, `useAutoRefresh(query)`, a `USwitch` bound to `enabled` (off by default), a Refresh button on `query.refetch()`, and `query.isLoading` as the "refreshing" mark.

## Add a shared component or composable

1. Component: `components/<Feature>Thing.vue`, flat and feature-prefixed (`pages/` is routes only; `project-structure.md`).  Type-only props and emits, no query, store, or API access, `defineModel` for two-way values, slots for content (`components-composables.md`).
2. Composable: `composables/useX.ts`, options object in, refs out, doc comment on the non-obvious decision.
3. Test it where it lives (`mountWithUi` for components; fake timers for time).
4. Register nothing: components in `src/components` are auto-registered; composables are imported explicitly.

## Hand-off checklist (definition of done)

- [ ] Loading, error, empty, and success states all work, plus feedback for writes (checked in the browser: stop the backend for the error state, clear the data for the empty state).
- [ ] Keyboard, light and dark, narrow and wide: `accessibility-responsive.md` ("Review checklist").
- [ ] No console errors; no network calls except to the API.
- [ ] Only Nuxt UI components, semantic color tokens, literal hugeicons names.
- [ ] Query and mutation tests (including a stale or failing response) and a page or component test.
- [ ] No hand edits under `src/api/generated`.
- [ ] `mise run check` passes (backend build, front-end build and type check, lint, format check, tests).  Front-end only, in `src/app`: `npm run lint && npm run fmt:check && npm run build && npm test`.
