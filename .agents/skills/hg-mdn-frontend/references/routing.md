# Routing and Layouts

File-based routing from `vue-router` v5 (`vue-router/vite`); the standalone `unplugin-vue-router` package is deprecated.  A `.vue` file under `src/pages` is a route; the plugin generates the route table (`vue-router/auto-routes`) and typed paths (`typed-router.d.ts`, committed).

## Files to routes

| File | URL |
|---|---|
| `pages/index.vue` | `/` |
| `pages/login.vue` | `/login` (the baseline login page) |
| `pages/notes/index.vue` | `/notes` |
| `pages/notes/[id].vue` | `/notes/:id` |
| `pages/notes/[[page]].vue` | `/notes/:page?` (optional param) |
| `pages/[...path].vue` | catch-all (404 page) |
| `pages/(group)/about.vue` | `/about` (a `(group)` folder organizes files and is not part of the URL) |
| `pages/notes.vue` **and** `pages/notes/*.vue` | `notes.vue` is the parent route: it renders `<RouterView />` for the children |

- **Every `.vue` file under `pages/` becomes a route**, so a component placed there becomes a URL.  Non-route components live in `src/components` (`project-structure.md`).
- `VueRouter()` must come before `vue()` in `vite.config.ts`.
- The plugin rewrites `typed-router.d.ts` on `dev` and `build`.  If a new page's path is not recognized by the editor or `vue-tsc`, run `npm run build` once.
- Hosting under a sub-path (`VITE_BASE`): `config-deploy.md`.

## Page meta with `definePage`

`definePage` is a compile-time macro (auto-imported); use it for route name, meta, and aliases:

```vue
<script setup lang="ts">
definePage({ meta: { title: "Notes" } });                                   // signed-in users (the default)
// definePage({ meta: { title: "Sign in", public: true, guestOnly: true } }); // the login page
// definePage({ meta: { title: "Admin", requiresAdmin: true } });            // administrators only
</script>
```

Declare the meta keys once so they are typed (`RouteMeta` in `router/guards.ts`, below): a typo in a key is a type error.  **A route with no `public` flag requires a signed-in user**, like the backend's root group.  A test can assert the result: `router.currentRoute.value.meta.title`.

## Typed routes and params

```vue
<script setup lang="ts">
// The route name is the file path; params are typed from the file name.
const route = useRoute("/notes/[id]");
route.params.id; // string
</script>
```

```vue
<!-- Typed links: a path that matches no file is a type error in vue-tsc -->
<RouterLink :to="`/notes/${note.id}`">{{ note.title }}</RouterLink>
<UButton to="/notes" label="All notes" />
```

Programmatic: `const router = useRouter(); router.push("/notes")`.  A list page that links to these routes is the notes page in `lists-tables.md`.

## Guards: secure by default, like the backend

The router mirrors the backend's route groups (`hg-mdn-backend`, `references/route-security.md`).  The guard is UX (no flash of a page the user cannot use, a redirect with a way back); **the backend enforces**.

| Front-end | Backend |
|---|---|
| a route with no `public` flag needs a signed-in user | the root group's `RequireAuthorization()` |
| `public: true` (the login page) | `AllowAnonymous()` |
| `requiresAdmin: true` on a parent route | `IAdminEndpoint` and the `/admin` group |
| 401 from any call sends the user to login | the challenge: 401 |
| a non-admin on an admin route goes home | admin routes answer 404 (hidden) |

### Who is signed in

`stores/session-store.ts` (shipped) holds `user`, `checked` (the first check finished), `isSignedIn`, `isAdmin`, `load({ force })`, and `signedOut()`.

- The browser never reads a cookie or token; it asks the backend (`GET /me`).  **401 means nobody is signed in.**  Any other failure (a 500, the network) **keeps the previous state**: a blip must not sign someone out.
- Concurrent callers share one request, and a stale response never overwrites a newer one (`data-loading.md`).

### The guards

`src/app/src/router/guards.ts`

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

`src/app/src/router/return-url.ts`

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

Installed once in `main.ts`, **before** the router starts (installing the router starts the first navigation) and after Pinia:

`src/app/src/main.ts (excerpt)`

```ts
// Guards go in before the router starts (installing it starts the first navigation); Pinia is
// already installed, so the guards can read the session store.
installGuards(router);
redirectToLoginOnUnauthorized(router);
```

- **Ask once.**  The first navigation awaits `session.load()`; later navigations use the answer.  Changes arrive through the 401 handler or an explicit `session.load({ force: true })`.
- **`matched` includes parent routes**, so `requiresAdmin` on a parent (`pages/admin.vue` next to `pages/admin/`) protects every child; do not repeat it on the children.
- **Return a location to redirect** (an object or a string; a string keeps the query), `false` to cancel, `true` to continue; do not call `next()`.
- **`returnUrl` is untrusted input.**  Only a path inside the app is honored: `//evil.test`, `https://...`, backslash tricks, the login page itself, and repeated query parameters all fall back to `/`.  Never assign `window.location.href` from a query value: that is an open redirect.
- **A 401 mid-session** (an expired session, a sign-out in another tab): the API client dispatches `api:unauthorized` (`api-client.md`); `redirectToLoginOnUnauthorized` forgets the user and replaces the route with `/login?returnUrl=...`.  It ignores the event while the first check runs (the guard handles that 401) and on public routes (no loop).
- **Hide what the user cannot use**: filter navigation entries with a computed over the navigation data and `session.isAdmin`, so an administrator-only link is not offered to everyone.
- The **login page** (`pages/login.vue`) is a placeholder: replace it with your identity provider's flow, then call `session.load({ force: true })`.  It is `public` and `guestOnly`, so a signed-in user who opens it is sent onward.

### Testing guards

Create a router with `createMemoryHistory()` and push to it (`router/guards.test.ts` ships): a memory router with routes that carry the real meta keys (`public`, `guestOnly`, `requiresAdmin`), `getMe` mocked at the module boundary, and `router.push` for each case.  Cover: signed out is redirected with `returnUrl`; signed in passes and titles the page; public stays open; the login page bounces a signed-in user; an unsafe `returnUrl` goes home; a parent's `requiresAdmin` covers a child; the user is asked **once**; a 401 event returns the user to login (and does nothing on a public route or during the first check).

Test `safeReturnUrl` as a pure function (`return-url.test.ts`): every unsafe shape is a table row.  The whole-app smoke test pushes to the generated `routes` the same way (`testing.md`).

## Layouts

`App.vue` hosts `<UApp>` around `<RouterView />`.  When the app needs a shell (a header with navigation), add a layout component around `<RouterView />` in `App.vue`: the Notes example uses `layouts/AppLayout.vue` (`nuxt-ui.md`).

- A route that needs a different shell (a login screen without the header): give it `meta: { layout: "bare" }` (declare `layout` in `RouteMeta`) and have `App.vue` render the layout only when `route.meta.layout !== "bare"`.  Do not nest `UApp`.
- Navigation entries are data (`layouts/app-navigation.ts`), not template markup; add a page, add one entry.
