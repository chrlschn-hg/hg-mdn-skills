# Nuxt UI

[Nuxt UI](https://ui.nuxt.com/llms.txt) v4: 125+ accessible Vue components (built on Reka UI) with Tailwind CSS theming.  Used here in a plain Vite app through `@nuxt/ui/vite` and `@nuxt/ui/vue-plugin`, not Nuxt.

**Sources of truth, in order:**

1. The official `nuxt-ui` agent skill in `.agents/skills/nuxt-ui` (component selection, forms, layouts, recipes).  Install: `npx skills add nuxt/ui --skill nuxt-ui -a codex -y` (`.claude/skills` is a symlink to `.agents/skills`).  Review installed skills before use.
2. <https://ui.nuxt.com/llms.txt>: dense index of every component; <https://ui.nuxt.com/llms-full.txt>: full docs (download and search).
3. The Nuxt UI MCP server (`https://ui.nuxt.com/mcp`, configured in `.mcp.json`).
4. The component's `.d.ts` in `node_modules/@nuxt/ui` for the exact props.
5. For **complex layouts**, the official Nuxt UI templates: working apps that show how components combine (`nuxt-ui-templates.md`).

## Rules

- **Components first.**  Before writing markup, check whether Nuxt UI has the component (`UCard`, `UTable`, `UForm`, `UModal`, `UDropdownMenu`, `UTabs`, `UBadge`, `UAlert`, ...).  Write a custom component only when none fits, and build it from Nuxt UI parts.
- **Slots before custom layout.**  Put content in the component's slots (`#header`, `#footer`, `#content`, `#leading`, ...) instead of wrapping it in extra elements.
- **Props before overrides.**  Use the available props (`color`, `variant`, `size`, `icon`, `loading`, `disabled`, `label`) before the `ui` prop (a slot class map), then Tailwind classes, then scoped CSS (`project-structure.md`, "Conventions").  A component-level change that should apply everywhere goes through `ui.<component>` in `vite.config.ts`, not one-off class soup.
- **Theme through tokens**, not hard-coded colors: `text-muted`, `text-highlighted`, `text-default`, `bg-elevated`, `border-default`, `text-primary`, `text-error`.
- **One `UApp` at the root** (`App.vue`): it provides toasts, tooltips, and overlays.
- **Icons: `hugeicons` only** (`i-hugeicons-<name>`), browsable at <https://icon-sets.iconify.design/hugeicons/>.  Never mix icon sets.
- Do not import Nuxt UI components: they are auto-registered.  This is not Nuxt: no `@nuxt/*` modules or Nuxt composables.

## Icons: bundle them

Nuxt UI's default icon slots (the `loading` spinner on `UButton`, chevrons in selects, the `close` icon, ...) are **lucide** icons, and any icon whose collection is not bundled is fetched from the public Iconify API at runtime: a network dependency, a privacy and CSP concern, and a failure when offline.  `vite.config.ts` fixes it in three parts:

1. `@iconify-json/hugeicons` is installed (a dependency).
2. `ui.icons` maps every default slot to hugeicons (the `icons` object below lists them all).
3. `icon.clientBundle` bundles those icons plus every literal `i-hugeicons-*` name in `.vue` and `.ts` files (`scan.globInclude`).

`src/app/vite.config.ts (excerpt)`

```ts
const icons = {
  arrowDown: "i-hugeicons-arrow-down-01",
  arrowLeft: "i-hugeicons-arrow-left-01",
  arrowRight: "i-hugeicons-arrow-right-01",
  arrowUp: "i-hugeicons-arrow-up-01",
```

(the real map lists all the slots) and, inside the `ui({ ... })` call:

`src/app/vite.config.ts (excerpt)`

```ts
      ui: {
        colors: {
          primary: "cyan",
          neutral: "zinc",
        },
        icons,
```

`src/app/vite.config.ts (excerpt)`

```ts
          scan: { globInclude: ["**/*.{vue,ts}"] },
          icons: Object.values(icons),
```

Rules that follow:

- Write icon names as **literal strings**: `icon="i-hugeicons-plus-sign"`.  A dynamic name (`` `i-hugeicons-${name}` ``) is invisible to the scan; add it to `icon.clientBundle.icons` by hand.  Names in `.ts` files (navigation data) are scanned; names fetched from an API at runtime are not.
- Verify: the baseline smoke test spies on `node:https` (the path Iconify uses; a failed icon fetch is silent, so only a spy catches it; `testing.md`), and DevTools network shows no request to `api.iconify.design`.

## Theming

Colors are configured in `vite.config.ts` (`ui.colors`): `primary`, `neutral`, plus `secondary`, `info`, `success`, `warning`, `error`.  To add a color token (for example `tertiary`), list it in `theme.colors` and map it in `ui.colors`:

```ts
ui({
  theme: { colors: ["primary", "secondary", "tertiary", "info", "success", "warning", "error"] },
  ui: { colors: { primary: "cyan", neutral: "zinc", tertiary: "purple" } },
});
```

Dark mode is automatic with the semantic tokens: `UColorModeButton` toggles it and `useDark()` (VueUse) reads it.  Never hard-code `bg-white`/`text-black`/hex colors; use `bg-default`, `bg-elevated`, `text-default`, `text-muted`, `border-default`.  Check every new screen in both modes.

## Patterns used in this project

- **App shell:** `UHeader` + `UNavigationMenu` + `UMain` + `UContainer` (the Notes example's `layouts/AppLayout.vue` and `app-navigation.ts`, below).
- **Loading, error, empty, content:** wrap in `AsyncContent` (`ui-states.md`); lists of records are a `UTable` or `UCard`s inside it, with search and paging from `useClientList` (`lists-tables.md`).
- **Forms:** `UForm` with the generated Zod schema, in a `USlideover` for create flows (`forms.md`, `overlays.md`).
- **Confirmations:** `useConfirm()` for a modal question, `PopConfirm` for a compact one in a list (`overlays.md`).
- **Feedback:** `useToast()` for transient results, `UAlert` for persistent page errors (`ui-states.md`).
- **Navigation:** `to="/path"` on `UButton`/`UNavigationMenu`, which renders a router link (typed routes check the path).

`src/app/src/layouts/AppLayout.vue`

```vue
<script setup lang="ts">
import { navigationItems } from "./app-navigation";
</script>

<template>
  <UHeader title="<PROJECT>">
    <UNavigationMenu :items="navigationItems" />

    <template #right>
      <UColorModeButton />
    </template>
  </UHeader>

  <UMain>
    <UContainer class="py-8">
      <slot />
    </UContainer>
  </UMain>
</template>
```

`src/app/src/layouts/app-navigation.ts`

```ts
import type { NavigationMenuItem } from "@nuxt/ui";

/** Top-level navigation.  Add an entry per page; `to` is a typed route path. */
export const navigationItems: NavigationMenuItem[] = [
  { label: "Home", icon: "i-hugeicons-home-01", to: "/" },
  { label: "Notes", icon: "i-hugeicons-note-01", to: "/notes" },
];
```

## Where to go next

| For | Read |
|---|---|
| loading, error, empty states, and toasts | `ui-states.md` |
| forms and validation | `forms.md` |
| slideovers, modals, confirmations | `overlays.md` |
| tables, search, paging | `lists-tables.md` |
| keyboard, screen readers, phones | `accessibility-responsive.md` |
| a dashboard, listing, inbox, or settings screen; a calendar, editor, or chat | `nuxt-ui-templates.md` |
| step-by-step feature work and the hand-off checklist | `recipes.md` |
