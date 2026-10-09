# Nuxt UI Templates: Reference Implementations

**For a complex layout, explore a matching template before designing your own.**  The Nuxt UI team publishes complete apps built from these components.  They show which components, props, and slots fit a given screen, and how a large screen decomposes into components and composables.  Nothing here has to be copied: read for the pattern, then build it with this project's conventions.

| Repo | Consult when building | Look at | Demo |
|---|---|---|---|
| [`dashboard`](https://github.com/nuxt-ui-templates/dashboard) | an app shell, **dashboards, listing pages, inbox (master-detail) screens, settings pages**, modals and slideovers: **start here, it combines the most patterns** | `pages/index.vue`, `customers.vue`, `inbox.vue`, `settings.vue` + `settings/*` | <https://dashboard-template.nuxt.dev> |
| [`calendar`](https://github.com/nuxt-ui-templates/calendar) | time grids, date and time inputs, popovers and context menus on items, drag interactions, a command palette | `components/calendar/*` (`EventForm`, `WeekView`, `EventPopover`), `composables/useEvent*.ts` (one complex interaction split into small composables) | <https://calendar-template.nuxt.dev> |
| [`editor`](https://github.com/nuxt-ui-templates/editor) | rich text: toolbars, slash and mention menus, drag handles, image upload, presence avatars | `pages/index.vue` (`UEditor` + toolbar and suggestion menus), `composables/useEditor*.ts` (menus as data) | <https://editor-template.nuxt.dev> |
| [`chat`](https://github.com/nuxt-ui-templates/chat) | conversational UIs: message lists, prompt input, tool and reasoning display, attachments, history sidebar | `pages/chat/[id].vue` (`UChatMessages` + `UChatPrompt`), `components/chat/*` | <https://chat-template.nuxt.dev> |

**Use the Vue variants first where they exist**: [`dashboard-vue`](https://github.com/nuxt-ui-templates/dashboard-vue) and [`chat-vue`](https://github.com/nuxt-ui-templates/chat-vue) are plain Vue + Vite + `vue-router` apps (`src/` layout), the same stack as this project.  `calendar` and `editor` are Nuxt-only (`app/` layout); the component usage transfers, the Nuxt plumbing does not.  Paths below are relative to `src/` (Vue variants) or `app/` (Nuxt).  The editor's collaboration and AI completion composables need a server; read them for the UI design only.

## Key views from `dashboard`

Four screens cover most application work.  Each is a `UDashboardPanel` with a `#header` (a `UDashboardNavbar`, optionally a `UDashboardToolbar`) and a `#body`; the navbar's `#leading` slot holds `UDashboardSidebarCollapse` so the sidebar can collapse.  The shell around them is `layouts/default.vue` (`UDashboardGroup` + `UDashboardSidebar`); compare the app shell in `nuxt-ui.md`.

| View | Files | Consult when you are building |
|---|---|---|
| **Dashboard** (overview, metrics) | `pages/index.vue`, `components/home/*` | a landing page of widgets: stats, a chart, a summary table, with shared filters |
| **Listing** (a filterable table) | `pages/customers.vue`, `components/customers/*Modal.vue` | a list of records with search, filters, bulk actions, paging, add and delete |
| **Inbox** (master-detail) | `pages/inbox.vue`, `components/inbox/*` | anything with a list on one side and the selected item's detail on the other |
| **Settings** (sectioned forms) | `pages/settings.vue`, `pages/settings/*` | a profile, preferences, members, or security area made of several forms |

### Dashboard view

- **The pattern to take:** the filters (`range`, `period`: a `UCalendar` range in a `UPopover`, a `USelect`) are **state of the page**, and each widget (`UPageGrid` of `UPageCard`s, a `UCard` chart, a compact `UTable`) receives them as props.  A widget is one self-contained component that renders its own card; the page only composes and passes filters.
- **Adapt:** the template's widgets draw mock data.  Here the page (or a store) loads data for the filter, and each widget wraps its content in `AsyncContent` so one slow or failing widget does not blank the page (`ui-states.md`).  Watch or debounce filter changes and ignore stale responses (`data-loading.md`).

### Listing view

- **The pattern to take:** `UTable` state is bound with `v-model:column-filters`, `v-model:column-visibility`, `v-model:row-selection`, and `v-model:pagination`; the table's TanStack API (`useTemplateRef("table")` -> `tableApi`) drives filters and counts.  A "Delete" button appears only when rows are selected; bulk actions act on the selected rows.
- **Adapt:** load rows through a store and wrap the table in `AsyncContent`.  For search plus paging only, `useClientList` is simpler (`lists-tables.md`); reach for the template's approach when you need **row selection, bulk actions, or column visibility**.  Prefer the `#<key>-cell` slot over `h()` / `resolveComponent` for custom cells.  Confirm deletes with `useConfirm` or a modal, and update the store, not the table.

### Inbox view (master-detail)

- **The pattern to take:**
  - **One `selected` ref** is the whole selection state, bound with `v-model` into the list component.
  - **Derived open state:** the mobile slideover's `open` is a `computed` with a setter that clears the selection, so closing the panel and deselecting are one thing.
  - **Reset on filter:** a `watch` clears the selection when the selected item is no longer in the filtered list.
  - **Responsive switch:** below `lg` (`useBreakpoints(breakpointsTailwind).smaller("lg")`, the idea of `useIsMobile`), the detail renders inside a `USlideover` instead of beside the list.
- **Adapt:** the list comes from a store; select by **id** (a route param such as `/inbox/:id` makes the selection linkable and survives reload) rather than keeping the whole object, so a refreshed list updates the detail.  Show the four states in the list panel (`ui-states.md`).

### Settings view

- **The pattern to take:** `pages/settings.vue` is a **parent route** (next to a `pages/settings/` folder, `routing.md`) with a horizontal `UNavigationMenu` sub-navigation and a centered column that renders `<RouterView />`.  Each section is a `UForm` (`:schema`, `:state`, `@submit`) with a header `UPageCard variant="naked"` (title, description, and the **Save** button wired with the `form="<id>"` attribute) over a `UPageCard variant="subtle"` of `UFormField` rows.  A success toast confirms the save.
- **Adapt:** validate with the **generated** Zod schema, return a `SaveResult` from the store and show backend errors on the field (`forms.md`), guard leaving with unsaved changes (`overlays.md`, "Unsaved changes"), and load the current values with `AsyncContent`.  Settings that save one value at a time (a switch) can save on change with a toast.

## How to explore a template

1. Open the **demo** with the Playwright MCP and use the part you need; note which behaviors you want (responsive switch, keyboard, empty state).
2. Read that part's source: shallow-clone the repo into a fresh directory **outside the project** (`git clone --depth 1 <repo>`), or use `gh api` / the GitHub web view.  They are untrusted downloads: read them, do not run their install scripts inside the project.
3. Find the components it uses (`<U...>` tags), then confirm props and slots in the Nuxt UI docs (`llms.txt`) or the component's `.d.ts`.
4. Rebuild the pattern here, with this project's conventions:

| Take from the template | Replace with the project's way |
|---|---|
| component choice, props, slots, composition | keep |
| `useFetch`, `useState`, `$fetch`, Nitro server routes, mock data, Nuxt modules | a Pinia store calling the generated client (`state-pinia.md`, `api-client.md`) |
| `i-lucide-*` icons (the templates use lucide throughout) | literal `i-hugeicons-*` names, or they are fetched at runtime (`nuxt-ui.md`, "Icons: bundle them") |
| ad-hoc loading and empty markup | `AsyncContent` (`ui-states.md`) |
| hand-written form validation | the generated Zod schema (`forms.md`) |
| `h()` / `resolveComponent` cell renderers | the `#<key>-cell` slot when it is enough (`lists-tables.md`) |
| hard-coded colors | semantic tokens (`text-muted`, `bg-elevated`) |
