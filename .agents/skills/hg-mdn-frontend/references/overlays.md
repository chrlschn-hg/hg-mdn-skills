# Overlays: Slideovers, Modals, Confirmations

Nuxt UI overlays (`USlideover`, `UModal`, `UDrawer`, `UPopover`) render in a **portal**: their content is teleported to `<body>`, outside the component that declares them.  `UApp` (the app root) hosts the programmatic ones.  Three patterns cover almost everything:

| Need | Pattern |
|---|---|
| A panel with a form or detail that belongs to one page | **declarative** `USlideover`/`UModal` with `v-model:open`, wrapped in its own component |
| "Are you sure?" before something destructive or lossy | **programmatic** `useConfirm()` -> `await confirm({...})` |
| A small inline confirmation in a list row | `PopConfirm` (a `UPopover`; below) |

Prefer a **slideover** for forms and detail (the page stays visible and the form has room), a **modal** for short, blocking decisions, and a **popover** for a quick inline choice.  Toasts are not overlays you manage: `ui-states.md`.

## A slideover component

The slideover is presentational: data in as props, decisions out as events.  The page owns the store.

`src/app/src/components/NoteFormSlideover.vue`

```vue
<script setup lang="ts">
import NoteForm from "@/components/NoteForm.vue";
import { useConfirm } from "@/composables/useConfirm";
import type { NoteDraft } from "@/stores/notes-store";

/**
 * Slideover around `NoteForm`.  Presentational: it takes data as props and reports decisions as
 * events; the page owns the store.  Closing with unsaved text asks first.
 */
defineProps<{
  saving?: boolean;
  description?: string;
  initial?: { title: string; body?: string };
}>();

const open = defineModel<boolean>("open", { default: false });

const emit = defineEmits<{ submit: [draft: NoteDraft] }>();

const form = useTemplateRef("form");
const confirm = useConfirm();

/** Closing with unsaved text asks first; opening, and closing a clean form, just happen. */
async function onUpdateOpen(next: boolean) {
  if (!next && form.value?.isDirty) {
    const discard = await confirm({
      title: "Discard this note?",
      description: "What you typed will be lost.",
      confirmLabel: "Discard",
      confirmColor: "error",
    });

    if (!discard) {
      return;
    }

    form.value.reset();
  }

  open.value = next;
}

defineExpose({
  showFailure: (failure: { code: string | null; message: string }) =>
    form.value?.showFailure(failure) ?? false,
  reset: () => form.value?.reset(),
});
</script>

<template>
  <USlideover
    :open="open"
    :title="initial ? 'Edit note' : 'New note'"
    :description="description"
    @update:open="onUpdateOpen"
  >
    <template #body>
      <NoteForm ref="form" :saving="saving" :initial="initial" @submit="emit('submit', $event)" />
    </template>
  </USlideover>
</template>
```

- **`defineModel<boolean>("open")`** gives `v-model:open` with no prop/emit boilerplate; the parent opens it with a ref (`slideoverOpen = true`).
- **Content goes in the `#body` slot** (and `#footer` for actions); `title` and `description` are props.  Nuxt UI adds the close button.
- The slideover emits `submit` and exposes `reset`/`showFailure` so the page can react to the save result (`forms.md`); a template ref (`useTemplateRef("form")`) reaches the child form's exposed members, which the child must `defineExpose`.
- **Intercept closing** by binding `:open` and handling `@update:open` yourself (above) instead of `v-model:open`: the handler decides whether the change happens.

## Unsaved changes: which mechanism

- **Closing a slideover or modal over one form:** intercept `@update:open` and `await confirm(...)` when the form's `isDirty` is true (the slideover above).  Reset the form only after the user confirms or the save succeeds.
- **Leaving the route, or switching the selected item, with an editor open:** `onBeforeRouteLeave` for the route; for an in-page switch, hold the pending action and ask save / discard / cancel (a small composable; the caller renders the dialog).  Neither is in the examples: write it when you need it.

## A confirmation you can `await`

`useOverlay()` creates overlays from components and returns a promise that resolves when the component emits `close`.  Wrap it once:

`src/app/src/components/ConfirmDialog.vue`

```vue
<script setup lang="ts">
/**
 * Confirmation modal for `useOverlay`: it resolves the overlay's promise with `true` or `false`
 * by emitting `close`.  Open it with `await confirm({ title, ... })` (see `useConfirm`).
 */
withDefaults(
  defineProps<{
    title: string;
    description?: string;
    confirmLabel?: string;
    cancelLabel?: string;
    /** Use `error` for destructive confirmations. */
    confirmColor?: "primary" | "error" | "neutral";
  }>(),
  {
    description: undefined,
    confirmLabel: "Confirm",
    cancelLabel: "Cancel",
    confirmColor: "primary",
  },
);

const emit = defineEmits<{ close: [confirmed: boolean] }>();
</script>

<template>
  <!-- `dismissible` false: leaving by Escape or outside click would never resolve the promise -->
  <UModal :title="title" :description="description" :dismissible="false">
    <template #footer>
      <div class="flex w-full justify-end gap-2">
        <UButton
          :label="cancelLabel"
          color="neutral"
          variant="outline"
          @click="emit('close', false)"
        />
        <UButton :label="confirmLabel" :color="confirmColor" @click="emit('close', true)" />
      </div>
    </template>
  </UModal>
</template>
```

`src/app/src/composables/useConfirm.ts`

```ts
import ConfirmDialog from "@/components/ConfirmDialog.vue";

type ConfirmOptions = {
  title: string;
  description?: string;
  confirmLabel?: string;
  cancelLabel?: string;
  confirmColor?: "primary" | "error" | "neutral";
};

/**
 * Promise-based confirmation: `if (await confirm({ title: "Discard changes?" })) { ... }`.
 * Needs a `<UApp>` ancestor (the app root has one): overlays render there, outside the page.
 */
export function useConfirm() {
  const overlay = useOverlay();
  const dialog = overlay.create(ConfirmDialog, { destroyOnClose: true });

  return (options: ConfirmOptions): Promise<boolean> => dialog.open(options);
}
```

```ts
const confirm = useConfirm();

if (await confirm({ title: "Delete this note?", confirmLabel: "Delete", confirmColor: "error" })) {
  await store.remove(id);
}
```

- The dialog **emits `close` with a boolean**: that value is what `await` returns.  `:dismissible="false"` stops Escape/outside-click from closing it without resolving the promise (the awaiting code would hang).
- `{ destroyOnClose: true }` frees the instance after each use.
- **Needs `<UApp>` as an ancestor.**  The app root has one; a component test mounts inside `src/test/AppHarness.vue`.  Overlays are mounted outside the page, so injected values from a page are not available inside them: pass data as props.
- Use `confirmColor: "error"` for destructive actions and name the action in the button (`Delete`, not `OK`).
- Do not ask twice, and do not confirm reversible actions (offer undo through a toast instead).

## Inline confirmation: `PopConfirm`

In a dense list use a popover instead of a modal.  The component is a thin wrapper over a Nuxt UI trigger button: it emits `confirm`/`cancel` and the parent does the deletion.

`src/app/src/components/PopConfirm.vue`

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false });

withDefaults(
  defineProps<{
    title: string;
    body: string;
    confirmLabel?: string;
    confirmColor?: "error" | "neutral";
  }>(),
  { confirmLabel: "Delete", confirmColor: "error" },
);

const emit = defineEmits<{
  confirm: [];
  cancel: [];
}>();

const open = ref(false);

// Button attrs (icon, color, variant, disabled, ...) belong on the trigger; the confirm and
// cancel listeners are declared component events and are emitted below.
const triggerAttrs = useAttrs();

function cancel() {
  open.value = false;
  emit("cancel");
}

function confirm() {
  open.value = false;
  emit("confirm");
}
</script>

<template>
  <!-- Inline confirmation for destructive actions; keeps table and list rows compact. -->
  <UPopover v-model:open="open">
    <UButton v-bind="triggerAttrs" />

    <template #content>
      <UCard class="w-72" :ui="{ header: 'flex items-center gap-2' }">
        <template #header>
          <UIcon name="i-hugeicons-alert-02" class="text-warning" />
          <span class="font-medium">{{ title }}</span>
        </template>

        <p class="text-sm text-muted">{{ body }}</p>

        <template #footer>
          <div class="flex justify-end gap-2">
            <UButton label="Cancel" size="xs" color="neutral" variant="ghost" @click="cancel" />
            <UButton :label="confirmLabel" size="xs" :color="confirmColor" @click="confirm" />
          </div>
        </template>
      </UCard>
    </template>
  </UPopover>
</template>
```

Usage (no import needed in `src/components`): `<PopConfirm title="Discard this note?" body="..." icon="i-hugeicons-delete-02" color="error" label="Discard" @confirm="discard" />`

## Rules

1. **One owner per overlay.**  The page (or the component that opened it) holds `open` state; the overlay never reaches for global state, and takes props and emits events like any presentational component.
2. **Guard data loss** (above).
3. **Focus and escape are free**: Nuxt UI (Reka UI) traps focus and handles Escape; do not rebuild them.  Give the container a `title` (it is the accessible name).
4. **No more than one overlay deep**: a confirmation over a slideover is the limit.

## Testing overlays

Teleported content lives in `document.body`: mount with `attachTo: document.body`, host in `UApp` when `useOverlay`/`useToast` is involved (`AppHarness`), and query the document, not the wrapper:

`src/app/src/components/NoteFormSlideover.test.ts`

```ts
import { flushPromises, type VueWrapper } from "@vue/test-utils";
import { afterEach, describe, expect, it } from "vitest";
import { h } from "vue";
import AppHarness from "@/test/AppHarness.vue";
import { mountWithUi } from "@/test/mount";
import NoteFormSlideover from "./NoteFormSlideover.vue";

let wrapper: VueWrapper | undefined;

afterEach(() => {
  wrapper?.unmount();
  document.body.innerHTML = "";
});

/** Mounts the slideover open, inside `UApp` (needed for the confirmation overlay). */
function mountOpen(props: Record<string, unknown> = {}) {
  wrapper = mountWithUi(AppHarness, {
    attachTo: document.body, // slideovers and modals teleport to <body>
    slots: { default: () => h(NoteFormSlideover, { open: true, ...props }) },
  });
  return wrapper;
}

/** The teleported content is in `document.body`, not inside the wrapper. */
const body = () => document.body;
const button = (label: string) =>
  [...body().querySelectorAll("button")].find((b) => b.textContent?.trim() === label);

async function typeTitle(value: string) {
  const input = body().querySelector("input") as HTMLInputElement;
  input.value = value;
  input.dispatchEvent(new Event("input", { bubbles: true }));
  await flushPromises();
}

async function pressClose() {
  (body().querySelector("[aria-label=Close]") as HTMLElement).click();
  await flushPromises();
}

describe("NoteFormSlideover", () => {
  it("renders the form inside the slideover", async () => {
    mountOpen();
    await flushPromises();

    expect(body().textContent).toContain("New note");
    expect(body().querySelector("form")).not.toBeNull();
  });

  it("edits a note: titled for editing and filled from the note", async () => {
    mountOpen({ initial: { title: "Groceries" } });
    await flushPromises();

    expect(body().textContent).toContain("Edit note");
    expect((body().querySelector("input") as HTMLInputElement).value).toBe("Groceries");
  });

  it("closes without asking when nothing was typed", async () => {
    mountOpen();
    await flushPromises();

    await pressClose();

    expect(body().textContent).not.toContain("Discard this note?");
    expect(wrapper!.findComponent(NoteFormSlideover).emitted("update:open")).toEqual([[false]]);
  });

  it("asks before discarding unsaved text, and stays open when the user cancels", async () => {
    mountOpen();
    await flushPromises();
    await typeTitle("half-written");

    await pressClose();

    expect(body().textContent).toContain("Discard this note?");

    button("Cancel")!.click();
    await flushPromises();

    expect(wrapper!.findComponent(NoteFormSlideover).emitted("update:open")).toBeUndefined();
  });

  it("closes after the user confirms the discard", async () => {
    mountOpen();
    await flushPromises();
    await typeTitle("half-written");
    await pressClose();

    button("Discard")!.click();
    await flushPromises();

    expect(wrapper!.findComponent(NoteFormSlideover).emitted("update:open")).toEqual([[false]]);
  });
});
```

- `flushPromises()` after each interaction: overlays open and close asynchronously.
- Find buttons by their visible label (`[...document.body.querySelectorAll("button")].find(...)`), as a user would.
- Clean up in `afterEach` (`wrapper.unmount(); document.body.innerHTML = ""`) so one test's overlay never leaks into the next.
- Assert what the **user sees** (the prompt text, the event the parent gets), not Reka UI internals.
