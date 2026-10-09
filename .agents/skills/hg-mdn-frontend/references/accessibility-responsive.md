# Accessibility and Responsive Design

Nuxt UI is built on Reka UI primitives, so focus management, keyboard handling, and ARIA roles for its components come for free **if you use the components and give them names**.  Most accessibility defects here are missing names and mouse-only custom controls.  Responsive design is mobile-first Tailwind plus the occasional behavior switch.

## Accessibility rules

1. **Everything interactive has an accessible name.**  An icon-only button needs `aria-label` (`<UButton icon="i-hugeicons-reload" aria-label="Refresh" />`); an input needs a `UFormField label` or `aria-label`; a placeholder is not a name.
2. **Use the real component.**  Navigation is a link (`RouterLink`, `UButton to`), an action is a `UButton`, a choice is a `USelect`/`UCheckbox`/`URadioGroup`.  Never a clickable `div`.
3. **Headings are an outline**: one `h1` per page, then `h2`, `h3` in order; style with classes, do not skip levels for looks.
4. **Color is never the only signal.**  An error shows an icon and text, not just red; use semantic tokens (`text-error`, `text-muted`) so contrast holds in light and dark.
5. **Keyboard**: every flow works with Tab, Shift+Tab, Enter, Space, and Escape; the focus ring stays visible (do not remove outlines); overlays trap and restore focus (Reka UI does it).
6. **Announce changes**: error alerts have `role="alert"` (`AsyncContent` does); busy regions have `aria-busy`; toasts are announced by Nuxt UI.
7. **Titles on containers**: modals and slideovers have a `title` (it is their accessible name); give a `description` when the purpose needs it.
8. **Motion is optional**: skeletons and spinners are fine; avoid large autoplaying animation; respect `prefers-reduced-motion` for anything custom.
9. **Language and document**: `<html lang>` is set in `index.html`; set `document.title` per route (`definePage` meta + the `afterEach` guard in `routing.md`).
10. **Lists are lists**: use `ul`/`li` (or `UTable`), not stacks of `div`s, so assistive tech announces the count.

Check with the **Playwright MCP's page snapshot** (`testing.md`), which is the accessibility tree: if a control is missing from it, has no name, or has the wrong role, a screen reader has the same problem.  Tests can query by name too (`[aria-label='Clear search']`, the button's visible text), which both documents and enforces the contract.

## Responsive layout

Tailwind is **mobile-first**: unprefixed classes apply everywhere, `md:`/`lg:` add behavior for wider screens.

```vue
<div class="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">...</div>
<UContainer class="py-6 md:py-8">...</UContainer>
```

- Breakpoints (Tailwind): `sm` 640, `md` 768, `lg` 1024, `xl` 1280, `2xl` 1536.  Design the phone layout first.
- **Change layout with classes**, not JavaScript: flex direction, grid columns (`grid-cols-1 md:grid-cols-3`), hidden elements (`hidden md:block`), spacing.
- **Change behavior with `useIsMobile`** only when the interaction itself differs (a slideover on phones, a popover on desktop):

`src/app/src/composables/useIsMobile.ts`

```ts
import { breakpointsTailwind } from "@vueuse/core";

/**
 * True below Tailwind's `md` breakpoint (768px).  Use it to change *behavior* (a slideover on
 * phones, a modal on desktop); change *layout* with Tailwind classes (`md:flex-row`) instead.
 * `useBreakpoints` is auto-imported; VueUse constants such as `breakpointsTailwind` are not.
 */
export function useIsMobile() {
  return useBreakpoints(breakpointsTailwind).smaller("md");
}
```

`src/app/src/composables/useIsMobile.test.ts`

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { useIsMobile } from "./useIsMobile";

/** happy-dom has no layout; make `matchMedia` answer for a given viewport width. */
function stubViewport(width: number) {
  vi.stubGlobal("matchMedia", (query: string) => {
    const max = /max-width:\s*([\d.]+)px/.exec(query);
    const min = /min-width:\s*([\d.]+)px/.exec(query);
    const matches =
      (max ? width <= Number(max[1]) : true) && (min ? width >= Number(min[1]) : true);

    return {
      matches,
      media: query,
      addEventListener() {},
      removeEventListener() {},
      addListener() {},
      removeListener() {},
    };
  });
}

afterEach(() => vi.unstubAllGlobals());

describe("useIsMobile", () => {
  it("is true on a phone-sized viewport", () => {
    stubViewport(390);

    expect(useIsMobile().value).toBe(true);
  });

  it("is false on a desktop viewport", () => {
    stubViewport(1280);

    expect(useIsMobile().value).toBe(false);
  });
});
```

  happy-dom has no layout, so tests stub `matchMedia` for the width they need.
- **Touch targets** are at least ~44 px: use Nuxt UI's default button sizes for primary actions; reserve `size="xs"` for dense desktop tables.
- **Tables** are the usual phone problem: show fewer columns, or let the table scroll horizontally; check at ~390 px.
- **Text**: do not fix widths in pixels; use `max-w-*`, `w-full`, and `truncate` with a `title`/tooltip for long names.
- **Images and media**: set `width`/`height` (or an aspect ratio) so the layout does not jump while loading.

Dark mode comes with the semantic tokens (`nuxt-ui.md`, "Theming"): never hard-code `bg-white`/`text-black`/hex colors, and check every new screen in both modes.

## Review checklist

1. Tab through the page: order sensible, focus visible, no traps, Escape closes overlays.
2. Every icon-only control has a name; every field has a label.
3. Page snapshot (Playwright MCP) shows sensible roles and names.
4. 390 px wide and 1280 px wide both usable; no horizontal page scroll.
5. Light and dark both readable; errors not conveyed by color alone.
6. Zoom to 200% without losing function.
