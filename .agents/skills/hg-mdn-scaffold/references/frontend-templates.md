# Front-end Templates (steps 7 and 8)

Vue 3 + Nuxt UI + Vite + TypeScript in `src/app`, with Pinia, VueUse, file-based routing, Kubb
(OpenAPI → TypeScript client and Zod schemas), `oxlint`/`oxfmt`, and Vitest with `@vue/test-utils`.  Create the files, then `npm install`,
build the backend once (writes the OpenAPI document), and `npm run generate`.

Placeholders (`<PROJECT>`, `<project>`, `<BACKEND_API_PORT>`, `<GUID>`, ...): see `SKILL.md`.
Versions are a snapshot: replace each with the latest stable at scaffold time.

**TypeScript 6.x, not 7:** `vue-tsc` does not support 7 (`vue-tsc -b` fails with
`ERR_PACKAGE_PATH_NOT_EXPORTED`).  Reason and revisit condition: the front-end skill,
`references/project-structure.md`.

**Package manager variants** (default `npm`):

| | npm | pnpm | yarn |
|---|---|---|---|
| Aspire `AddViteApp` | `.WithNpm()` | `.WithPnpm()` | `.WithYarn()` |
| Kubb watch in AppHost | `command: "npm"`, `args: ["run", "generate:watch"]` | `command: "pnpm"`, `args: ["generate:watch"]` | `command: "yarn"`, `args: ["generate:watch"]` |
| CI install | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --immutable` |
| Mise tool | ships with `node` | `pnpm = "<version>"` | `yarn = "<version>"` |
| Extras | none | `pnpm-workspace.yaml` with `packages: ['.']` and `allowBuilds: { esbuild: true, vue-demi: true }` | `.yarnrc.yml` with `nodeLinker: node-modules` |

The baseline uses npm; the pnpm and yarn columns are not exercised by it.

**One origin:** the API paths start with `/api`, so the generated client calls relative URLs; the backend serves the built front-end in production and `vite.config.ts` proxies `/api` in development.  No CORS, no base URL (`hg-mdn-frontend`, `references/config-deploy.md`).

**Generated files:** `src/api/<project>-api.json` (OpenAPI document, written by the backend build) is
committed.  `src/api/generated/` (Kubb output) is git-ignored; `npm test` and `npm run build`
regenerate it first, and Aspire runs Kubb in watch mode.

`vite.config.ts` registers `ui(...)` from `@nuxt/ui/vite`, which wires `unplugin-auto-import` and
`unplugin-vue-components` internally: configure them through its `autoImport` and `components` options
instead of adding the plugins separately.

## Project files

### `src/app/package.json`

`pretest` and `build` run `kubb generate` first so a fresh clone works (`generated/` is git-ignored).

```json
{
  "name": "app",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "kubb generate && vite build && vue-tsc -b",
    "preview": "vite preview",
    "generate": "kubb generate",
    "generate:watch": "kubb generate --watch",
    "pretest": "kubb generate",
    "test": "vitest run",
    "lint": "oxlint",
    "lint:fix": "oxlint --fix",
    "fmt": "oxfmt",
    "fmt:check": "oxfmt --check"
  },
  "dependencies": {
    "@iconify-json/hugeicons": "^1.2.35",
    "@nuxt/ui": "^4.11.3",
    "@vueuse/core": "^15.0.0",
    "pinia": "^4.0.3",
    "tailwindcss": "^4.3.3",
    "vue": "^3.5.43",
    "vue-router": "^5.4.0",
    "zod": "^4.6.5"
  },
  "devDependencies": {
    "@kubb/plugin-fetch": "^5.5.6",
    "@kubb/plugin-ts": "^5.0.5",
    "@kubb/plugin-zod": "^5.4.3",
    "@types/node": "^26.6.4",
    "@vitejs/plugin-vue": "^6.0.9",
    "@vue/test-utils": "^2.5.1",
    "@vue/tsconfig": "^0.9.1",
    "happy-dom": "^20.14.6",
    "kubb": "^5.5.6",
    "oxfmt": "^0.72.0",
    "oxlint": "^1.87.0",
    "typescript": "~6.0.3",
    "unplugin-auto-import": "^21.1.0",
    "unplugin-vue-components": "^32.1.0",
    "vite": "^8.3.4",
    "vitest": "^5.0.3",
    "vue-tsc": "^3.3.12"
  }
}
```

### `src/app/vite.config.ts`

`vue-router/vite` provides file-based routing (the standalone `unplugin-vue-router` package is deprecated).  `VueRouter()` must come before `vue()`.  Imports `defineConfig` from `vitest/config` so the `test` block type-checks.  **Icons:** the `icons` map points Nuxt UI's default (lucide) slots at hugeicons and `icon.clientBundle` bundles them, so the shipped app makes no icon requests; icons built dynamically are invisible to the scan and go in `clientBundle.icons` (the front-end skill, `references/nuxt-ui.md`).  `base` honors `VITE_BASE` for sub-path hosting; `test.setupFiles` installs the network guard (`src/test/setup.ts`).  **`server.proxy`** forwards `/api` to the backend (`vite preview` reuses it); Aspire provides the URL as `services__app-backend__http__0` through `.WithReference(backend)` in `host/AppHost.cs`.

```ts
import path from "node:path";
import ui from "@nuxt/ui/vite";
import vue from "@vitejs/plugin-vue";
import { VueUseComponentsResolver } from "unplugin-vue-components/resolvers";
import { defineConfig } from "vitest/config";
// File-based routing ships with vue-router v5; the standalone `unplugin-vue-router`
// package is deprecated in favour of these entrypoints.
import { VueRouterAutoImports } from "vue-router/unplugin";
import VueRouter from "vue-router/vite";

// Nuxt UI's default icon slots (loading spinner, chevrons, close, ...) are lucide icons.  Point every
// slot at the installed hugeicons set and bundle them (`icon.clientBundle`), so the app never
// fetches icons from the public Iconify API at runtime.  Add literal icon names used in `.ts`
// files (like `app-navigation.ts`) automatically: the scan below reads `.vue` and `.ts`.
const icons = {
  arrowDown: "i-hugeicons-arrow-down-01",
  arrowLeft: "i-hugeicons-arrow-left-01",
  arrowRight: "i-hugeicons-arrow-right-01",
  arrowUp: "i-hugeicons-arrow-up-01",
  caution: "i-hugeicons-alert-02",
  check: "i-hugeicons-tick-02",
  chevronDoubleLeft: "i-hugeicons-arrow-left-double",
  chevronDoubleRight: "i-hugeicons-arrow-right-double",
  chevronDown: "i-hugeicons-arrow-down-01",
  chevronLeft: "i-hugeicons-arrow-left-01",
  chevronRight: "i-hugeicons-arrow-right-01",
  chevronUp: "i-hugeicons-arrow-up-01",
  close: "i-hugeicons-cancel-01",
  copy: "i-hugeicons-copy-01",
  copyCheck: "i-hugeicons-copy-check",
  dark: "i-hugeicons-moon-02",
  drag: "i-hugeicons-drag-drop-vertical",
  ellipsis: "i-hugeicons-more-horizontal",
  error: "i-hugeicons-cancel-circle",
  external: "i-hugeicons-link-square-01",
  eye: "i-hugeicons-view",
  eyeOff: "i-hugeicons-view-off",
  file: "i-hugeicons-file-01",
  folder: "i-hugeicons-folder-01",
  folderOpen: "i-hugeicons-folder-open",
  hash: "i-hugeicons-hashtag",
  info: "i-hugeicons-information-circle",
  light: "i-hugeicons-sun-03",
  loading: "i-hugeicons-loading-03",
  menu: "i-hugeicons-menu-01",
  minus: "i-hugeicons-minus-sign",
  panelClose: "i-hugeicons-layout-left",
  panelOpen: "i-hugeicons-sidebar-left",
  plus: "i-hugeicons-plus-sign",
  reload: "i-hugeicons-reload",
  search: "i-hugeicons-search-01",
  stop: "i-hugeicons-square",
  success: "i-hugeicons-checkmark-circle-02",
  system: "i-hugeicons-computer",
  tip: "i-hugeicons-bulb",
  upload: "i-hugeicons-upload-01",
  warning: "i-hugeicons-alert-01",
};

// The backend the `/api` proxy forwards to.  Aspire injects the URL when the front-end resource has
// `.WithReference(backend)` (`host/AppHost.cs`); the fallback is `applicationUrl` in the backend's
// `launchSettings.json`, for running Vite alone.
const apiTarget = process.env["services__app-backend__http__0"] ?? "http://localhost:<BACKEND_API_PORT>";

// https://vite.dev/config/
export default defineConfig({
  // Hosted under a sub-path (for example `/web/`)?  Build with `VITE_BASE=/web/ npm run build`.
  base: process.env.VITE_BASE ?? "/",
  plugins: [
    // Must come before `vue()` so route blocks in SFCs are handled first.
    VueRouter(),
    vue(),
    // Nuxt UI wires unplugin-auto-import and unplugin-vue-components internally;
    // configure both through the `autoImport` and `components` options here.
    ui({
      icon: {
        clientBundle: {
          scan: { globInclude: ["**/*.{vue,ts}"] },
          icons: Object.values(icons),
        },
      },
      autoImport: {
        imports: ["vue", "pinia", VueRouterAutoImports, "@vueuse/core"],
      },
      components: {
        resolvers: [VueUseComponentsResolver()],
      },
      ui: {
        colors: {
          primary: "cyan",
          neutral: "zinc",
        },
        icons,
      },
    }),
  ],
  resolve: {
    alias: {
      // `@/` -> `src/`
      "@": path.resolve(import.meta.dirname, "src"),
    },
  },
  // The browser only ever talks to Vite; `/api` is forwarded to the backend, so development is
  // same-origin like production (no CORS, cookies behave).  `vite preview` reuses this proxy.
  server: { proxy: { "/api": apiTarget } },
  test: {
    environment: "happy-dom",
    include: ["src/**/*.test.ts"],
    setupFiles: ["./src/test/setup.ts"],
    restoreMocks: true,
    clearMocks: true,
  },
});
```

### `src/app/tsconfig.json`

```json
{
  "files": [],
  "references": [{ "path": "./tsconfig.app.json" }, { "path": "./tsconfig.node.json" }]
}
```

### `src/app/tsconfig.app.json`

```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "types": ["vite/client"],
    "allowArbitraryExtensions": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["./src/*"],
      "#build/ui/*": ["./node_modules/.nuxt-ui/ui/*"]
    }
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "typed-router.d.ts",
    "auto-imports.d.ts",
    "components.d.ts"
  ]
}
```

### `src/app/tsconfig.node.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "es2023",
    "lib": ["ES2023"],
    "types": ["node"],
    "skipLibCheck": true,
    "module": "nodenext",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "#build/ui": ["./node_modules/.nuxt-ui/ui"]
    }
  },
  "include": ["vite.config.ts"]
}
```

### `src/app/src/env.d.ts`

Types `import.meta.env` (`VITE_API_BASE_URL` only, and only needed when the front-end is hosted apart from the API; `VITE_BASE` is read in `vite.config.ts`).  Only `VITE_`-prefixed variables reach the browser; they are public.

```ts
/// <reference types="vite/client" />

// Typed `import.meta.env`.  Only `VITE_`-prefixed variables reach the browser bundle; anything
// else stays on the build machine.  They are public: never put a secret in one.
interface ImportMetaEnv {
  /** Backend origin for the generated client; unset (the default) means same-origin `/api`. */
  readonly VITE_API_BASE_URL?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### `src/app/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title><project></title>
  </head>
  <body>
    <div id="app" class="isolate"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### `src/app/.gitignore`

```gitignore
# Logs
logs
*.log
npm-debug.log*

node_modules
dist
dist-ssr
*.local

# Editor directories and files
.idea
.DS_Store

# Auto-generated declarations (rebuilt by Vite on dev/build)
auto-imports.d.ts
components.d.ts
```

## Lint and format (oxlint, oxfmt)

### `src/app/.oxlintrc.json`

Lints `.ts` and `.vue`.  `oxlint` prints nothing and exits 0 when clean; exits 1 on errors.  The three `rules` enforce the TypeScript conventions: braces always, no `any`, `for...of` instead of `forEach`.

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "vue", "unicorn", "oxc"],
  "categories": {
    "correctness": "error"
  },
  "ignorePatterns": ["dist", "src/api/generated", "*.d.ts"],
  "rules": {
    "curly": "error",
    "typescript/no-explicit-any": "error",
    "unicorn/no-array-for-each": "error"
  }
}
```

### `src/app/.oxfmtrc.json`

```json
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "ignorePatterns": ["dist", "src/api/generated", "*.d.ts", "src/api/*.json", "**/*.md"]
}
```

## Kubb

### `src/app/kubb.config.ts`

There is no `baseURL`: the OpenAPI paths already start with `/api`, so the generated calls are relative and go to whichever origin served the app (the backend in production, the Vite proxy in development).  Hosting the front-end apart from the API is the alternative recipe in the front-end skill, `references/config-deploy.md`.  `pluginZod` generates form-validation schemas from the same OpenAPI constraints, so limits cannot drift from the backend.

```ts
import { defineConfig } from "kubb";
import { pluginTs } from "@kubb/plugin-ts";
import { pluginFetch } from "@kubb/plugin-fetch";
import { pluginZod } from "@kubb/plugin-zod";

// Reads the OpenAPI document that the backend build writes (see `OpenApiDocumentsDirectory` in
// `<PROJECT>.Runtime.csproj`) and generates typed fetch clients.  Run: `npm run generate`.
export default defineConfig({
  input: "./src/api/<project>-api.json",
  output: {
    path: "./src/api/generated",
    clean: true,
  },
  plugins: [
    pluginTs({
      output: {
        path: "./types",
        mode: "directory",
      },
      enum: {
        type: "asConst",
      },
      optionalType: "questionTokenAndUndefined",
    }),
    // Zod schemas from the same OpenAPI constraints (maxLength, required, ...): form validation
    // that cannot drift from the backend.  Import from `@/api/generated/zod/<operation>Schema`.
    pluginZod({ output: { path: "./zod", mode: "directory" } }),
    // https://kubb.dev/plugins/plugin-fetch/reference/options
    // No `baseURL`: the OpenAPI paths already start with `/api`, so the generated calls are
    // relative and the SPA talks to whatever origin served it (the backend in production, the
    // Vite dev proxy locally; see `vite.config.ts`).
    pluginFetch({
      output: {
        path: "./clients",
        mode: "directory",
      },
    }),
  ],
});
```

### `src/app/src/api/README.txt`

```text
Generated API files live here.

  <project>-api.json   OpenAPI document; written by the backend build (src/backend/runtime); COMMITTED
  generated/           TypeScript types + fetch clients; written by Kubb (`npm run generate`); git-ignored

Do not edit by hand.
Flow: C# endpoints -> OpenAPI document (`dotnet build`) -> Kubb -> generated/.  Aspire runs Kubb in watch mode.
After a fresh clone, run `npm run generate` (or `mise run generate`) before building the front-end.
```

## Application

### `src/app/src/main.ts`

```ts
import "./assets/css/main.css";

import ui from "@nuxt/ui/vue-plugin";
import { createPinia } from "pinia";
import { createApp } from "vue";
import { createRouter, createWebHistory } from "vue-router";
import { routes } from "vue-router/auto-routes";
import { configureApiClient } from "@/api/setup";
import { installGuards, redirectToLoginOnUnauthorized } from "@/router/guards";
import App from "./App.vue";

configureApiClient();

const app = createApp(App);

const router = createRouter({
  routes,
  history: createWebHistory(import.meta.env.BASE_URL),
});

app.use(createPinia());

// Guards go in before the router starts (installing it starts the first navigation); Pinia is
// already installed, so the guards can read the session store.
installGuards(router);
redirectToLoginOnUnauthorized(router);

app.use(router);
app.use(ui);

app.mount("#app");
```

### `src/app/src/App.vue`

```vue
<template>
  <UApp>
    <RouterView />
  </UApp>
</template>
```

### `src/app/src/assets/css/main.css`

```css
@import "tailwindcss";
@import "@nuxt/ui";
```

### `src/app/src/pages/index.vue`

Shows the backend health and, under it, the running build (`BuildInfoLabel`).  In a real app shell put the label where people look when reporting a problem: the sidebar footer or the user menu.

```vue
<script setup lang="ts">
// `ref`, `onMounted`, `storeToRefs`, `useDark` etc. are auto-imported.
import { useHealthStore } from "@/stores/health-store";

const store = useHealthStore();
const { health, loading, error } = storeToRefs(store);

onMounted(() => store.refresh());
</script>

<template>
  <main class="flex min-h-svh items-center justify-center p-6">
    <UCard class="w-full max-w-md">
      <template #header>
        <div class="flex items-center gap-2">
          <UIcon name="i-hugeicons-sparkles" class="size-5 text-primary" />
          <h1 class="text-lg font-semibold text-highlighted"><PROJECT></h1>
        </div>
      </template>

      <!-- Every page that loads data renders the four states: loading, error, empty, content. -->
      <AsyncContent
        :loading="loading && !health"
        :error="error"
        :empty="!health"
        empty-title="No result yet"
      >
        <p class="text-muted">
          Backend is <UKbd :value="health?.status" /> as of {{ health?.checkedAtUtc }}.
        </p>
        <BuildInfoLabel v-if="health" :build="health.build" class="mt-2" />
      </AsyncContent>

      <template #footer>
        <UButton icon="i-hugeicons-reload" :loading="loading" @click="store.refresh()">
          Refresh
        </UButton>
      </template>
    </UCard>
  </main>
</template>
```

### `src/app/src/stores/health-store.ts`

The pattern: components use a Pinia store; the store calls the generated client.

```ts
import { acceptHMRUpdate, defineStore } from "pinia";
import { getHealth } from "@/api/generated/clients/getHealth";
import type { HealthResponse } from "@/api/generated/types/HealthResponse";
import { errorMessage } from "@/api/errors";

/**
 * Backend health.  Stores own the API calls; components consume the store (never the generated
 * client directly) so loading/error state lives in one place.
 */
export const useHealthStore = defineStore("health", () => {
  const health = ref<HealthResponse | null>(null);
  const loading = ref(false);
  const error = ref<string | null>(null);

  /** Fetches the current health; records the error message instead of throwing. */
  async function refresh() {
    loading.value = true;
    error.value = null;

    try {
      health.value = await getHealth().unwrap();
    } catch (err: unknown) {
      error.value = errorMessage(err, "Health check failed.");
    } finally {
      loading.value = false;
    }
  }

  return { health, loading, error, refresh };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useHealthStore, import.meta.hot));
}
```

### `src/app/src/stores/health-store.test.ts`

```ts
import { createPinia, setActivePinia } from "pinia";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { useHealthStore } from "./health-store";

const apiMocks = vi.hoisted(() => ({ getHealth: vi.fn() }));

vi.mock("@/api/generated/clients/getHealth", () => ({ getHealth: apiMocks.getHealth }));

describe("useHealthStore", () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it("stores the health response on refresh", async () => {
    const response = {
      status: "Healthy",
      checkedAtUtc: "2026-10-09T00:00:00Z",
      build: { version: "1.4.0", commit: "3f2a1bc", builtAtUtc: null },
    };
    apiMocks.getHealth.mockReturnValue({ unwrap: () => Promise.resolve(response) });

    const store = useHealthStore();
    await store.refresh();

    expect(store.health).toEqual(response);
    expect(store.error).toBeNull();
    expect(store.loading).toBe(false);
  });

  it("records the error message when the call fails", async () => {
    apiMocks.getHealth.mockReturnValue({ unwrap: () => Promise.reject(new Error("boom")) });

    const store = useHealthStore();
    await store.refresh();

    expect(store.health).toBeNull();
    expect(store.error).toBe("boom");
  });
});
```

### `src/app/src/stores/session-store.ts`

Who is signed in, from `GET /me`: 401 means nobody; other failures keep the previous state.

```ts
import { acceptHMRUpdate, defineStore } from "pinia";
import { isUnauthorized, errorMessage } from "@/api/errors";
import { getMe } from "@/api/generated/clients/getMe";
import type { MeResponse } from "@/api/generated/types/MeResponse";

/**
 * Who is signed in.  The backend is the authority (`GET /me`): the browser never reads the
 * session cookie or a token.  A 401 means nobody is signed in.  The router guard consults this
 * store; the guard is a convenience, the backend enforces every route.
 */
export const useSessionStore = defineStore("session", () => {
  const user = ref<MeResponse | null>(null);
  /** True once the first `load()` finished, whatever its outcome: the guard waits for it once. */
  const checked = ref(false);
  const error = ref<string | null>(null);

  const isSignedIn = computed(() => user.value !== null);
  const isAdmin = computed(() => user.value?.isAdmin === true);

  let inFlight: Promise<void> | null = null;
  let latestLoad = 0;

  /**
   * Asks the backend who is signed in.  401 clears the user.  Any other failure (a 500, the
   * network) keeps the previous state: a blip on the server must not sign someone out.
   * Concurrent callers share one request; `force` starts a new one (after signing in elsewhere).
   */
  function load(options: { force?: boolean } = {}): Promise<void> {
    if (inFlight && !options.force) {
      return inFlight;
    }

    const request = ++latestLoad;
    const promise = (async () => {
      error.value = null;

      try {
        const me = await getMe().unwrap();

        if (request === latestLoad) {
          user.value = me;
        }
      } catch (err: unknown) {
        if (request === latestLoad) {
          if (isUnauthorized(err)) {
            user.value = null;
          } else {
            error.value = errorMessage(err, "Could not check who is signed in.");
          }
        }
      } finally {
        if (request === latestLoad) {
          checked.value = true;
          inFlight = null;
        }
      }
    })();

    inFlight = promise;

    return promise;
  }

  /** Forgets the user (the API answered 401 mid-session, or the user signed out). */
  function signedOut() {
    user.value = null;
  }

  return { user, checked, error, isSignedIn, isAdmin, load, signedOut };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useSessionStore, import.meta.hot));
}
```

### `src/app/src/stores/session-store.test.ts`

```ts
import { createPinia, setActivePinia } from "pinia";
import { beforeEach, describe, expect, it, vi } from "vitest";
import { ResponseError } from "@/api/generated/.kubb/client";
import { useSessionStore } from "./session-store";

const api = vi.hoisted(() => ({ getMe: vi.fn() }));
vi.mock("@/api/generated/clients/getMe", () => ({ getMe: api.getMe }));

const ok = <T>(value: T) => ({ unwrap: () => Promise.resolve(value) });
const fails = (error: unknown) => ({ unwrap: () => Promise.reject(error) });
const failure = (status: number) =>
  new ResponseError({
    data: undefined,
    status,
    statusText: "",
    request: new Request("http://localhost/me"),
    response: new Response(null, { status }),
  });
const ada = { subject: "u1", name: "Ada", roles: ["member"], isAdmin: false };

describe("session store", () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it("is signed in after a successful check, and reports admin from the backend", async () => {
    api.getMe.mockReturnValue(ok({ ...ada, roles: ["member", "admin"], isAdmin: true }));
    const store = useSessionStore();

    await store.load();

    expect(store.isSignedIn).toBe(true);
    expect(store.isAdmin).toBe(true);
    expect(store.checked).toBe(true);
  });

  it("treats a 401 as nobody signed in, without an error", async () => {
    api.getMe.mockReturnValue(fails(failure(401)));
    const store = useSessionStore();

    await store.load();

    expect(store.isSignedIn).toBe(false);
    expect(store.error).toBeNull();
    expect(store.checked).toBe(true);
  });

  it("keeps the signed-in user when a later check fails for another reason", async () => {
    const store = useSessionStore();
    api.getMe.mockReturnValueOnce(ok(ada));
    await store.load();

    api.getMe.mockReturnValueOnce(fails(failure(500)));
    await store.load({ force: true });

    expect(store.isSignedIn).toBe(true);
    expect(store.error).not.toBeNull();
  });

  it("shares one request between concurrent callers", async () => {
    api.getMe.mockClear();
    api.getMe.mockReturnValue(ok(ada));
    const store = useSessionStore();

    await Promise.all([store.load(), store.load(), store.load()]);

    expect(api.getMe).toHaveBeenCalledTimes(1);
  });

  it("lets only the newest check write state", async () => {
    const store = useSessionStore();
    let releaseSlow: (() => void) | undefined;
    api.getMe.mockReturnValueOnce({
      unwrap: () =>
        new Promise((resolve) => {
          releaseSlow = () => resolve(ada);
        }),
    });
    const slow = store.load();

    // A forced check that finds nobody signed in lands first, then the stale slow one resolves.
    api.getMe.mockReturnValueOnce(fails(failure(401)));
    await store.load({ force: true });
    releaseSlow?.();
    await slow;

    expect(store.isSignedIn).toBe(false);
  });

  it("forgets the user on signedOut()", async () => {
    api.getMe.mockReturnValue(ok(ada));
    const store = useSessionStore();
    await store.load();

    store.signedOut();

    expect(store.isSignedIn).toBe(false);
  });
});
```

### `src/app/src/router/guards.ts`

Secure by default, like the backend: a route needs a signed-in user unless its meta says `public`; `requiresAdmin` on a parent covers its children; `redirectToLoginOnUnauthorized` handles a 401 mid-session.  Guards are UX; the backend enforces.

```ts
import type { Router } from "vue-router";
import { safeReturnUrl } from "@/router/return-url";
import { useSessionStore } from "@/stores/session-store";

// Typed route meta: `definePage({ meta: { public: true } })` is checked against this.
declare module "vue-router" {
  interface RouteMeta {
    /** Browser tab and header title. */
    title?: string;
    /** Reachable without signing in (the login page).  Every other route requires sign-in. */
    public?: boolean;
    /** With `public`: a signed-in user is sent onward instead (the login page). */
    guestOnly?: boolean;
    /** Only administrators.  Set it on a parent route and every child is protected. */
    requiresAdmin?: boolean;
  }
}

/**
 * Installs the navigation guards.  **Secure by default, like the backend:** a route requires a
 * signed-in user unless its meta says `public`.  These guards are UX; the backend decides.
 */
export function installGuards(router: Router) {
  router.beforeEach(async (to) => {
    const session = useSessionStore();

    // Ask who is signed in once, on the first navigation.  Later changes arrive through
    // `redirectToLoginOnUnauthorized` or an explicit `session.load({ force: true })`.
    if (!session.checked) {
      await session.load();
    }

    if (to.meta.public) {
      // A guest-only page (login) has nothing to offer a signed-in user: continue to where they
      // were headed.  Returning a string redirects, and keeps the query string.
      return to.meta.guestOnly && session.isSignedIn ? safeReturnUrl(to.query.returnUrl) : true;
    }

    if (!session.isSignedIn) {
      return { path: "/login", query: { returnUrl: to.fullPath } };
    }

    // `matched` holds the route and its parents, so `requiresAdmin` on a parent covers its children.
    if (to.matched.some((record) => record.meta.requiresAdmin) && !session.isAdmin) {
      return "/";
    }

    return true;
  });

  router.afterEach((to) => {
    document.title = to.meta.title ? `${to.meta.title} · <PROJECT>` : "<PROJECT>";
  });
}

/**
 * When any API call answers 401 mid-session (the session expired, the user signed out in another
 * tab), forget the user and send them to the login page with a way back.  Wire it once in
 * `main.ts`; the client dispatches the event (`configureApiClient`).  Returns an unsubscribe.
 */
export function redirectToLoginOnUnauthorized(router: Router) {
  const onUnauthorized = () => {
    const session = useSessionStore();

    // The first check runs inside the guard and handles its own 401.
    if (!session.checked) {
      return;
    }

    session.signedOut();

    const current = router.currentRoute.value;

    if (!current.meta.public) {
      void router.replace({ path: "/login", query: { returnUrl: current.fullPath } });
    }
  };

  window.addEventListener("api:unauthorized", onUnauthorized);

  return () => window.removeEventListener("api:unauthorized", onUnauthorized);
}
```

### `src/app/src/router/return-url.ts`

Only a path inside the app is honored as `returnUrl` (no open redirect).

```ts
/**
 * A `returnUrl` query value that is safe to navigate to after signing in: a path inside this app.
 * Anything else (another site, a protocol-relative `//host`, a backslash trick, the login page
 * itself, a non-string) becomes `fallback`.  Without this check `?returnUrl=https://evil.test` is
 * an open redirect.
 */
export function safeReturnUrl(value: unknown, fallback = "/"): string {
  if (typeof value !== "string") {
    return fallback;
  }

  const isLocalPath = value.startsWith("/") && !value.startsWith("//") && !value.includes("\\");
  const isLoginPage = value === "/login" || value.startsWith("/login?");

  return isLocalPath && !isLoginPage ? value : fallback;
}
```

### `src/app/src/router/guards.test.ts`

```ts
import { createPinia, setActivePinia } from "pinia";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { createMemoryHistory, createRouter } from "vue-router";
import { ResponseError } from "@/api/generated/.kubb/client";
import { useSessionStore } from "@/stores/session-store";
import { installGuards, redirectToLoginOnUnauthorized } from "./guards";

const api = vi.hoisted(() => ({ getMe: vi.fn() }));
vi.mock("@/api/generated/clients/getMe", () => ({ getMe: api.getMe }));

const Page = { template: "<div />" };
const ok = <T>(value: T) => ({ unwrap: () => Promise.resolve(value) });
const unauthorized = {
  unwrap: () =>
    Promise.reject(
      new ResponseError({
        data: undefined,
        status: 401,
        statusText: "",
        request: new Request("http://localhost/me"),
        response: new Response(null, { status: 401 }),
      }),
    ),
};
const member = { subject: "u1", name: "Ada", roles: ["member"], isAdmin: false };
const admin = { ...member, roles: ["member", "admin"], isAdmin: true };

/** The routes the guards care about; the same meta keys a real page sets with `definePage`. */
function createTestRouter() {
  const router = createRouter({
    history: createMemoryHistory(),
    routes: [
      { path: "/", component: Page, meta: { title: "Home" } },
      { path: "/notes", component: Page, meta: { title: "Notes" } },
      { path: "/login", component: Page, meta: { public: true, guestOnly: true } },
      { path: "/about", component: Page, meta: { public: true } },
      {
        // `requiresAdmin` on the parent protects every child.
        path: "/admin",
        component: Page,
        meta: { requiresAdmin: true },
        children: [{ path: "users", component: Page }],
      },
    ],
  });

  installGuards(router);

  return router;
}

describe("installGuards", () => {
  beforeEach(() => {
    setActivePinia(createPinia());
    api.getMe.mockReset();
  });

  it("sends a signed-out user to the login page and remembers where they were going", async () => {
    api.getMe.mockReturnValue(unauthorized);
    const router = createTestRouter();

    await router.push("/notes?page=2");

    expect(router.currentRoute.value.path).toBe("/login");
    expect(router.currentRoute.value.query.returnUrl).toBe("/notes?page=2");
  });

  it("lets a signed-in user through and titles the page", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();

    await router.push("/notes");

    expect(router.currentRoute.value.path).toBe("/notes");
    expect(document.title).toBe("Notes · <PROJECT>");
  });

  it("leaves a public route open to a signed-out user", async () => {
    api.getMe.mockReturnValue(unauthorized);
    const router = createTestRouter();

    await router.push("/about");

    expect(router.currentRoute.value.path).toBe("/about");
  });

  it("sends a signed-in user from the login page to where they were going", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();

    await router.push("/login?returnUrl=/notes%3Fpage%3D2");

    expect(router.currentRoute.value.fullPath).toBe("/notes?page=2");
  });

  it("ignores an unsafe returnUrl and goes home", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();

    await router.push({ path: "/login", query: { returnUrl: "https://evil.test/steal" } });

    expect(router.currentRoute.value.path).toBe("/");
  });

  it("keeps a non-admin out of a child of an admin-only parent", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();

    await router.push("/admin/users");

    expect(router.currentRoute.value.path).toBe("/");
  });

  it("lets an admin into the admin routes", async () => {
    api.getMe.mockReturnValue(ok(admin));
    const router = createTestRouter();

    await router.push("/admin/users");

    expect(router.currentRoute.value.path).toBe("/admin/users");
  });

  it("asks the backend who is signed in once, not on every navigation", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();

    await router.push("/notes");
    await router.push("/");
    await router.push("/notes");

    expect(api.getMe).toHaveBeenCalledTimes(1);
  });
});

describe("redirectToLoginOnUnauthorized", () => {
  let stop: (() => void) | undefined;

  beforeEach(() => {
    setActivePinia(createPinia());
    api.getMe.mockReset();
  });

  afterEach(() => stop?.());

  it("signs the user out and returns them to login, keeping the way back", async () => {
    api.getMe.mockReturnValue(ok(member));
    const router = createTestRouter();
    stop = redirectToLoginOnUnauthorized(router);
    await router.push("/notes");

    window.dispatchEvent(new CustomEvent("api:unauthorized"));
    await vi.waitFor(() => expect(router.currentRoute.value.path).toBe("/login"));

    expect(router.currentRoute.value.query.returnUrl).toBe("/notes");
    expect(useSessionStore().isSignedIn).toBe(false);
  });

  it("does nothing while the first check is still running (the guard handles that 401)", async () => {
    const router = createTestRouter();
    stop = redirectToLoginOnUnauthorized(router);

    window.dispatchEvent(new CustomEvent("api:unauthorized"));

    expect(router.currentRoute.value.path).toBe("/");
    expect(useSessionStore().checked).toBe(false);
  });

  it("does not redirect when the user is already on a public route", async () => {
    api.getMe.mockReturnValue(unauthorized);
    const router = createTestRouter();
    stop = redirectToLoginOnUnauthorized(router);
    await router.push("/about");

    window.dispatchEvent(new CustomEvent("api:unauthorized"));
    await new Promise((resolve) => setTimeout(resolve, 10));

    expect(router.currentRoute.value.path).toBe("/about");
  });
});
```

### `src/app/src/router/return-url.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { safeReturnUrl } from "./return-url";

describe("safeReturnUrl", () => {
  it("keeps a path inside the app, with its query and hash", () => {
    expect(safeReturnUrl("/notes?page=2#top")).toBe("/notes?page=2#top");
  });

  it.each([
    ["another site", "https://evil.test/steal"],
    ["a protocol-relative URL", "//evil.test"],
    ["a backslash trick", "/\\evil.test"],
    ["a javascript URL", "javascript:alert(1)"],
    ["the login page itself", "/login?returnUrl=/notes"],
    ["a relative path", "notes"],
    ["an empty string", ""],
  ])("falls back for %s", (_, value) => {
    expect(safeReturnUrl(value)).toBe("/");
  });

  it("falls back for a non-string (a repeated query parameter is an array)", () => {
    expect(safeReturnUrl(["/a", "/b"])).toBe("/");
    expect(safeReturnUrl(undefined, "/home")).toBe("/home");
  });
});
```

### `src/app/src/pages/login.vue`

A placeholder: replace it with your identity provider's flow, then call `session.load({ force: true })`.

```vue
<script setup lang="ts">
import { safeReturnUrl } from "@/router/return-url";
import { useSessionStore } from "@/stores/session-store";

// Public, and only for people who are not signed in (the guard sends everyone else onward).
definePage({ meta: { title: "Sign in", public: true, guestOnly: true } });

const session = useSessionStore();
const route = useRoute("/login");
const router = useRouter();
const checking = ref(false);

/** After signing in elsewhere (the provider's page, another tab): ask again and continue. */
async function retry() {
  checking.value = true;
  await session.load({ force: true });
  checking.value = false;

  if (session.isSignedIn) {
    await router.replace(safeReturnUrl(route.query.returnUrl));
  }
}
</script>

<template>
  <section class="mx-auto max-w-md space-y-4 py-16 text-center">
    <h1 class="text-2xl font-semibold text-highlighted">Sign in</h1>

    <!-- Placeholder: replace with your identity provider's flow (a redirect to its sign-in page,
         or a form that posts to your login endpoint), then call `session.load({ force: true })`. -->
    <p class="text-muted">You need to sign in to continue.</p>

    <UAlert
      v-if="session.error"
      color="error"
      variant="subtle"
      :description="session.error"
      role="alert"
    />

    <UButton :loading="checking" @click="retry">I have signed in</UButton>
  </section>
</template>
```

### `src/app/src/App.test.ts`

Whole-app smoke test: mounts the app (router, Nuxt UI, Pinia) under happy-dom with the API mocked at the generated-client boundary.  Catches broken wiring that unit tests miss.

```ts
import ui from "@nuxt/ui/vue-plugin";
import https from "node:https";
import { createPinia } from "pinia";
import { describe, expect, it, vi } from "vitest";
import { createApp } from "vue";
import { createMemoryHistory, createRouter } from "vue-router";
import { routes } from "vue-router/auto-routes";
import App from "./App.vue";

// Mock the API at the generated-client boundary: the smoke test never touches the network.
const api = vi.hoisted(() => ({ getHealth: vi.fn() }));
vi.mock("@/api/generated/clients/getHealth", () => ({ getHealth: api.getHealth }));

/** Mounts the whole app (router, Nuxt UI, Pinia) at `path` and waits for the route to render. */
async function mountAt(path: string) {
  const root = document.createElement("div");
  document.body.append(root);

  const router = createRouter({ routes, history: createMemoryHistory() });
  const app = createApp(App).use(createPinia()).use(router).use(ui);

  await router.push(path);
  await router.isReady();
  app.mount(root);
  await vi.waitFor(() => expect(root.textContent).not.toBe(""));

  return { root, unmount: () => app.unmount() };
}

describe("App", () => {
  it("renders the home page with the backend health", async () => {
    api.getHealth.mockReturnValue({
      unwrap: () =>
        Promise.resolve({
          status: "Healthy",
          checkedAtUtc: "2026-10-09T00:00:00Z",
          build: { version: "0.0.0-dev", commit: null, builtAtUtc: null },
        }),
    });

    const { root, unmount } = await mountAt("/");

    await vi.waitFor(() => expect(root.textContent).toContain("Healthy"));
    expect(root.textContent).toContain("<PROJECT>");
    expect(root.textContent).toContain("Version 0.0.0-dev");

    unmount();
  });

  it("makes no outbound HTTPS requests: icons are bundled, not fetched from Iconify", async () => {
    // NOTE: an icon that is not bundled is fetched from api.iconify.design through `node:https`
    // (not `fetch`), and a failed fetch is silent, so only this spy notices.
    const request = vi.spyOn(https, "request");
    api.getHealth.mockReturnValue({
      unwrap: () =>
        Promise.resolve({
          status: "Healthy",
          checkedAtUtc: "2026-10-09T00:00:00Z",
          build: { version: "0.0.0-dev", commit: null, builtAtUtc: null },
        }),
    });

    const { root, unmount } = await mountAt("/");
    await vi.waitFor(() => expect(root.textContent).toContain("Healthy"));
    await new Promise((resolve) => setTimeout(resolve, 100)); // let icon loading start

    expect(request).not.toHaveBeenCalled();

    unmount();
  });
});
```

## API plumbing

### `src/app/src/api/errors.ts`

Turns thrown values into something a store can show: the backend's `ApiError` (stable `code`, display `message`) or a fallback; `isUnauthorized` recognizes a 401.  Write actions return a `SaveResult` instead of throwing.

```ts
import { ResponseError } from "@/api/generated/.kubb/client";

/**
 * The backend's expected-failure body (the `ApiError` record in `<PROJECT>.Api`): a stable
 * machine-readable `code` to switch on and a `message` fit for display.  Declared here rather
 * than imported from `generated/` because the generated type only exists once an endpoint
 * returns it.
 */
export type ApiError = { code: string; message: string };

/** True when `data` has the shape of the backend's expected-failure body (`ApiError`). */
function isApiError(data: unknown): data is ApiError {
  return typeof data === "object" && data !== null && "code" in data && "message" in data;
}

/**
 * The backend's `ApiError` (`{ code, message }`) when `err` is a failed API call that carries
 * one; `null` for anything else (network failures, unexpected 500s).  Switch on `code`, not on
 * the message.
 */
export function toApiError(err: unknown): ApiError | null {
  return err instanceof ResponseError && isApiError(err.data) ? err.data : null;
}

/** True when `err` is a failed API call that answered 401: nobody is signed in. */
export function isUnauthorized(err: unknown): boolean {
  return err instanceof ResponseError && err.status === 401;
}

/** A user-facing message for any thrown value, preferring the backend's own message. */
export function errorMessage(err: unknown, fallback: string): string {
  return toApiError(err)?.message ?? (err instanceof Error ? err.message : fallback);
}

/**
 * The outcome of a write action that the caller must react to (close a form, mark a field).
 * Actions return this instead of throwing: `ok` on success; otherwise the backend's stable error
 * `code` (when it sent one) and a message fit for display.
 */
export type SaveResult = { ok: true } | { ok: false; code: string | null; message: string };

/** Converts a thrown value into the failure side of a {@link SaveResult}. */
export function toSaveFailure(err: unknown, fallback: string): SaveResult & { ok: false } {
  return {
    ok: false,
    code: toApiError(err)?.code ?? null,
    message: errorMessage(err, fallback),
  };
}
```

### `src/app/src/api/setup.ts`

Called once from `main.ts`.  `VITE_API_BASE_URL` (unset by default: same-origin) re-points a built app at another backend without regenerating the client; safe to call twice (replaces its own interceptor).  Any 401 dispatches `api:unauthorized`, which the router's login redirect listens for.

```ts
import { client } from "@/api/generated/.kubb/client";

// Id of the interceptor this module registered, so configuring twice (HMR, tests) replaces it
// instead of stacking a second copy.
let unauthorizedInterceptor: number | undefined;

/**
 * One-time configuration of the generated client; call it from `main.ts` before the app mounts.
 * Safe to call again: it replaces what it registered before.
 */
export function configureApiClient() {
  // The generated calls are relative (`/api/...`): same-origin in production (the backend serves
  // the SPA) and in development (the Vite proxy).  Only when the SPA is hosted apart from the API
  // set `VITE_API_BASE_URL` when building (see "Hosting apart from the API" in `config-deploy.md`:
  // it also needs CORS on the backend, and `credentials: "include"` for cookie authentication).
  const baseURL = import.meta.env.VITE_API_BASE_URL;

  if (baseURL !== undefined) {
    client.setConfig({ ...client.getConfig(), baseURL });
  }

  // Interceptors run in registration order.  Error interceptors see every non-2xx `ResponseError`
  // before it is thrown: the place for cross-cutting reactions such as "401 -> sign in".
  if (unauthorizedInterceptor !== undefined) {
    client.interceptors.error.eject(unauthorizedInterceptor);
  }

  unauthorizedInterceptor = client.interceptors.error.use((error) => {
    if (error.status === 401) {
      window.dispatchEvent(new CustomEvent("api:unauthorized"));
    }
    return error;
  });
}
```

### `src/app/src/api/errors.test.ts`

Runs the real generated client with only `fetch` stubbed.  `setConfig` merges, so the cleanup names `baseURL` explicitly; otherwise a test that sets it leaks into the next.

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { client } from "@/api/generated/.kubb/client";
import { getHealth } from "@/api/generated/clients/getHealth";
import { errorMessage, toApiError } from "./errors";
import { configureApiClient } from "./setup";

/** Stubs only `fetch`: everything above it is the real generated client. */
function respondWith(status: number, body: unknown) {
  vi.stubGlobal(
    "fetch",
    vi.fn(
      async () =>
        new Response(JSON.stringify(body), {
          status,
          headers: { "content-type": "application/json" },
        }),
    ),
  );
}

// `configureApiClient` mutates the shared client: put its config back after every test.
let originalConfig: ReturnType<typeof client.getConfig>;

beforeEach(() => (originalConfig = { ...client.getConfig() }));

afterEach(() => {
  // `setConfig` merges, so a key the test added survives unless it is named: restore `baseURL`
  // explicitly (it is unset by default), or the next test inherits the previous test's backend.
  client.setConfig({ ...originalConfig, baseURL: originalConfig.baseURL });
  vi.unstubAllEnvs();
  vi.unstubAllGlobals();
});

describe("API errors from the real generated client", () => {
  it("exposes the backend's ApiError on an expected failure", async () => {
    respondWith(400, { code: "invalid_thing", message: "That is not valid." });

    const error = await getHealth()
      .unwrap()
      .catch((e) => e);

    expect(toApiError(error)).toEqual({ code: "invalid_thing", message: "That is not valid." });
    expect(errorMessage(error, "fallback")).toBe("That is not valid.");
  });

  it("falls back for an unexpected failure with no ApiError body", async () => {
    respondWith(500, { title: "An unexpected error occurred." });

    const error = await getHealth()
      .unwrap()
      .catch((e) => e);

    expect(toApiError(error)).toBeNull();
    expect(errorMessage(error, "Could not check health.")).not.toBe("");
  });
});

/** Stubs `fetch` with a healthy response; the generated client hands it a `Request`. */
function stubHealthFetch() {
  const fetchMock = vi.fn(
    async () =>
      new Response(JSON.stringify({ status: "Healthy", checkedAtUtc: "x" }), {
        status: 200,
        headers: { "content-type": "application/json" },
      }),
  );
  vi.stubGlobal("fetch", fetchMock);

  return () =>
    new URL((fetchMock.mock.calls[0] as unknown as [Request])[0].url, window.location.href);
}

describe("configureApiClient", () => {
  it("calls a relative /api URL by default: same-origin, with no base URL configured", async () => {
    configureApiClient();
    const requestUrl = stubHealthFetch();

    await getHealth().unwrap();

    expect(requestUrl().origin).toBe(window.location.origin);
    expect(requestUrl().pathname).toBe("/api/health");
  });

  it("points the client at VITE_API_BASE_URL when it is set", async () => {
    vi.stubEnv("VITE_API_BASE_URL", "https://api.example.test");
    configureApiClient();
    const requestUrl = stubHealthFetch();

    await getHealth().unwrap();

    expect(requestUrl().href).toBe("https://api.example.test/api/health");
  });

  it("treats an empty VITE_API_BASE_URL as same-origin", async () => {
    vi.stubEnv("VITE_API_BASE_URL", "");
    configureApiClient();
    const requestUrl = stubHealthFetch();

    await getHealth().unwrap();

    expect(requestUrl().origin).toBe(window.location.origin);
    expect(requestUrl().pathname).toBe("/api/health");
  });

  it("runs the error interceptor for a 401, once even when configured twice", async () => {
    configureApiClient();
    configureApiClient();
    const unauthorized = vi.fn();
    window.addEventListener("api:unauthorized", unauthorized);
    respondWith(401, { code: "unauthorized", message: "Sign in." });

    await getHealth()
      .unwrap()
      .catch(() => {});

    expect(unauthorized).toHaveBeenCalledOnce();
    window.removeEventListener("api:unauthorized", unauthorized);
  });
});
```

## Shared UI

### `src/app/src/components/AsyncContent.vue`

The four states of anything that loads (error, loading, empty, content) in one place; pages use it instead of repeating a `v-if` chain.

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

### `src/app/src/components/AsyncContent.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { h } from "vue";
import { mountWithUi } from "@/test/mount";
import AsyncContent from "./AsyncContent.vue";

const content = () => h("p", { "data-test": "content" }, "the content");

function render(props: { loading: boolean; error?: string | null; empty: boolean }) {
  return mountWithUi(AsyncContent, { props, slots: { default: content } });
}

describe("AsyncContent", () => {
  it("shows the content when loaded and not empty", () => {
    const wrapper = render({ loading: false, empty: false });

    expect(wrapper.find("[data-test=content]").exists()).toBe(true);
  });

  it("shows skeletons while loading, instead of the content", () => {
    const wrapper = render({ loading: true, empty: false });

    expect(wrapper.find("[aria-busy=true]").exists()).toBe(true);
    expect(wrapper.find("[data-test=content]").exists()).toBe(false);
  });

  it("shows the empty state when there is nothing to show", () => {
    const wrapper = mountWithUi(AsyncContent, {
      props: { loading: false, empty: true, emptyTitle: "No notes yet" },
      slots: { default: content },
    });

    expect(wrapper.text()).toContain("No notes yet");
    expect(wrapper.find("[data-test=content]").exists()).toBe(false);
  });

  it("shows the error in place of everything else", () => {
    const wrapper = render({ loading: true, empty: true, error: "Could not load notes." });

    expect(wrapper.find("[role=alert]").text()).toContain("Could not load notes.");
    expect(wrapper.find("[aria-busy=true]").exists()).toBe(false);
  });

  it("lets a slot replace the empty state", () => {
    const wrapper = mountWithUi(AsyncContent, {
      props: { loading: false, empty: true },
      slots: { default: content, empty: () => h("p", "Create your first note") },
    });

    expect(wrapper.text()).toContain("Create your first note");
  });
});
```

### `src/app/src/components/BuildInfoLabel.vue`

The running build, from `GET /api/health` (the version, the commit, and the build time the backend stamped; `hg-mdn-backend`, `references/build-info.md`).  The commit and the build time are optional: a build without them shows only the version.

```vue
<script setup lang="ts">
import type { BuildResponse } from "@/api/generated/types/BuildResponse";

// Which build is running: the version, commit, and build time the backend reports on
// `/api/health` (stamped by the build, see `BuildInfo` in the backend).  Show it wherever people
// report problems from: a footer, an about dialog, the user menu.
const props = defineProps<{ build: BuildResponse }>();

const shortCommit = computed(() => props.build.commit?.slice(0, 7) ?? null);
const builtAtLabel = computed(() =>
  props.build.builtAtUtc
    ? new Intl.DateTimeFormat(undefined, { dateStyle: "medium", timeStyle: "short" }).format(
        new Date(props.build.builtAtUtc),
      )
    : null,
);
</script>

<template>
  <p class="flex flex-wrap items-center gap-x-2 text-xs text-muted">
    <span>Version {{ build.version }}</span>
    <code v-if="shortCommit" :title="build.commit ?? undefined">{{ shortCommit }}</code>
    <time v-if="build.builtAtUtc" :datetime="build.builtAtUtc">Built {{ builtAtLabel }}</time>
  </p>
</template>
```

### `src/app/src/components/BuildInfoLabel.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { mountWithUi } from "@/test/mount";
import BuildInfoLabel from "./BuildInfoLabel.vue";

describe("BuildInfoLabel", () => {
  it("shows the version, the short commit, and the build time", () => {
    const wrapper = mountWithUi(BuildInfoLabel, {
      props: {
        build: {
          version: "1.4.0",
          commit: "3f2a1bc9d0e84a5b6c7d8e9f00112233445566ff",
          builtAtUtc: "2026-10-11T14:02:03Z",
        },
      },
    });

    expect(wrapper.text()).toContain("Version 1.4.0");
    expect(wrapper.get("code").text()).toBe("3f2a1bc");
    expect(wrapper.get("code").attributes("title")).toBe(
      "3f2a1bc9d0e84a5b6c7d8e9f00112233445566ff",
    );
    expect(wrapper.get("time").attributes("datetime")).toBe("2026-10-11T14:02:03Z");
    expect(wrapper.get("time").text()).toMatch(/^Built .*2026/);
  });

  it("shows only the version for a build with no stamp (no git, no build arg)", () => {
    const wrapper = mountWithUi(BuildInfoLabel, {
      props: { build: { version: "0.0.0-dev", commit: null, builtAtUtc: null } },
    });

    expect(wrapper.text()).toBe("Version 0.0.0-dev");
    expect(wrapper.find("code").exists()).toBe(false);
    expect(wrapper.find("time").exists()).toBe(false);
  });
});
```

## Test infrastructure

### `src/app/src/test/setup.ts`

Fails any real HTTP request with a clear message.  Without it an unmocked API call tries `localhost:<port>` and fails with `ECONNREFUSED`.

```ts
import { afterEach, beforeEach, vi } from "vitest";

/**
 * Tests never reach a real backend.  Every API function a test touches must be mocked (the
 * generated clients, at the module boundary); anything that slips through would otherwise try
 * `localhost:<port>` and fail with a confusing `ECONNREFUSED`.  This turns it into a clear error.
 * Tests that exercise the real client stub `fetch` themselves (`vi.stubGlobal`), which wins.
 */
beforeEach(() => {
  vi.stubGlobal("fetch", (input: RequestInfo | URL) => {
    throw new Error(
      `Unexpected network request in a test: ${String(input)}. Mock the API function.`,
    );
  });
});

afterEach(() => vi.unstubAllGlobals());
```

### `src/app/src/test/mount.ts`

`mount` with the Nuxt UI plugin and a fresh Pinia: component tests use this.

```ts
import ui from "@nuxt/ui/vue-plugin";
import { mount, type ComponentMountingOptions } from "@vue/test-utils";
import { createPinia } from "pinia";
import type { Component } from "vue";

/**
 * `mount` with what every component here needs: the Nuxt UI plugin (theme and injections) and a
 * fresh Pinia.  Pass `attachTo: document.body` when the component teleports (slideovers, modals,
 * popovers) and the test queries `document.body`.
 */
export function mountWithUi<T extends Component>(
  component: T,
  options: ComponentMountingOptions<T> = {},
) {
  return mount(component, {
    ...options,
    global: { ...options.global, plugins: [ui, createPinia(), ...(options.global?.plugins ?? [])] },
  } as ComponentMountingOptions<T>);
}
```

### `src/app/src/test/AppHarness.vue`

Hosts overlays: components that open a confirmation (`useOverlay`) must be mounted inside `UApp`.

```vue
<script setup lang="ts">
// Hosts overlays: `useOverlay` (confirmation dialogs) renders into `UApp`, so a component that
// opens one must be tested inside it.
</script>

<template>
  <UApp>
    <slot />
  </UApp>
</template>
```

## Directories to create (empty)

Create `src/composables`, `src/layouts`, and `src/utils` when the first file needs them (git does not track empty directories; the baseline has none).  Copy `public/favicon.svg` (any SVG).  Conventions: the `hg-mdn-frontend` skill.
