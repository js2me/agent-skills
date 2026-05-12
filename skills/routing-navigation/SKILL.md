---
name: routing-navigation
description: Routing with mobx-route and mobx-location-history — routeConfig (history, mergeQuery, formatLinkHref, fallbackPath), createRoute/extend/groupRoutes, createRouter, QueryParams, RouteView (view, children, fallback), RouteViewGroup (otherwise, useLastOpened, nav options), Link, navigation, VirtualRoute, 404
license: MIT
compatibility: opencode
---

# Routing & Navigation (mobx-route)

Full documentation:

- mobx-route: https://js2me.github.io/mobx-route/llms-full.txt
- mobx-location-history: https://js2me.github.io/mobx-location-history/llms-full.txt

Paths use [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) syntax (optional segments, rest params, and so on). Example for a catch-all segment: `createRoute('/services/:serviceId{/*rest}')`.

## Global setup: `routeConfig` and history

Routes read **`routeConfig.history`** (and optional **`routeConfig.queryParams`**) for navigation. Initialize once at app bootstrap:

```typescript
import {
  routeConfig,
  createBrowserHistory,
  createMemoryHistory,
} from 'mobx-route';
// Same factories are re-exported from mobx-route; you can import from mobx-location-history instead.

const history =
  typeof window === 'undefined'
    ? createMemoryHistory({
        // SSR: seed stack with this request’s path + search (field names depend on your server)
        initialEntries: [incomingPathWithSearch],
      })
    : createBrowserHistory();

routeConfig.update({
  history,
  baseUrl: '/', // prefix when app is not served from domain root
  mergeQuery: false, // false = replace whole query on route.open; true = merge with current search
});
```

Optional **`routeConfig`** fields from the library (see core docs): **`formatLinkHref`** (transform `href` before it is applied in **`Link`**), global **`createUrl`** (same idea as per-route `createUrl` on **`Route`**), **`fallbackPath`** (used when path compilation fails, default `'/'`).

For **non-browser** environments (SSR, tests, RN-style shells), use *Memory routing*: `routeConfig.update({ history: createMemoryHistory(...) })`.

For **hash** history: `createHashHistory()` from **`mobx-route`** / **`mobx-location-history`**, then `routeConfig.update({ history: createHashHistory() })`.

## Declaring routes

Prefer **`groupRoutes()`** for nested trees. Use **`extend()`** on a parent **`Route`** to append a relative segment (merged path, shared hierarchy). **`extend()`** does **not** inherit the parent’s **`index`**, **`params`**, or **`exact`** — set them on the child if needed.

```typescript
import { createRoute, createVirtualRoute, groupRoutes } from 'mobx-route';

const home = createRoute('/', { exact: true });
const profile = createRoute('/profile', { exact: true });
const users = createRoute('/users', { exact: true });
const userDetails = users.extend('/:userId');

const items = groupRoutes({
  list: createRoute('/items', { index: true }),
  details: createRoute('/items/:itemId'),
});
const notFound = createVirtualRoute();

export const routes = {
  home,
  profile,
  users,
  userDetails,
  items,
  notFound,
};
```

`{ index: true }` marks an index route inside a group (used by `group.open()` and matching). **`groupRoutes`** also exists as the **`RouteGroup`** class. Useful group APIs: **`isOpened`**, **`indexRoute`** (first route with `index: true`), **`open()`** (opens index route if present, else tries the last nested group; can pass params for an index path like `/user/:id`).

`{ exact: true }` keeps a prefix route from staying “open” when a longer path matches another route — pick per route based on behavior you need.

## Optional: `createRouter` (central `Router`)

For a single object that exposes **`routes`**, **`history`**, **`location`**, **`query`**, and **`navigate()`**:

```typescript
import {
  createBrowserHistory,
  createRoute,
  createRouter,
  routeConfig,
} from 'mobx-route';
import { createQueryParams } from 'mobx-location-history';

const history = createBrowserHistory();
const queryParams = createQueryParams({ history });

routeConfig.update({ history, queryParams });

export const router = createRouter({
  routes: { home: createRoute('/') /* ... */ },
  history,
  queryParams,
});

router.routes.home.open();
router.navigate(router.routes.home.createUrl());
router.history.back();
// Reactive search string object:
router.query.data;
router.query.update({ tab: 'settings' }); // merges; use undefined value to drop a key
```

**`new QueryParams({ history })`** from **`mobx-route`** is the same class; **`createQueryParams`** is the factory from **`mobx-location-history`** (see Query parameters section).

Whether you keep a **`Router`** instance on `globals` or only export **`routes`** + **`routeConfig`** depends on the project; APIs below work on **route entities** (`Route`, `VirtualRoute`, `RouteGroup`) the same way.

## Query parameters (`QueryParams`)

Use **`createQueryParams({ history })`** or **`new QueryParams({ history })`**, often wired through **`routeConfig`** / **`createRouter`**. The observable snapshot of the current search string is **`queryParams.data`** (plain object of string values), **not** `.params`.

```typescript
import { createQueryParams, createBrowserHistory } from 'mobx-location-history';

const history = createBrowserHistory();
const query = createQueryParams({ history });

// Read (reactive)
query.data.page; // e.g. '2'

// Replace whole query object (optional second arg: use history.replace)
query.set({ page: '3', search: 'hello' });
query.set({ page: '2' }, true);

// Merge into current query; set field to undefined to remove it
query.update({ page: '3' });
query.update({ page: undefined }); // removes `page`

// Helpers
query.toString();
query.createUrl({ page: '2' }); // same path, new search
```

For **typed single-parameter** helpers, see `createQueryParam` / `createQueryParamFromPreset` in the mobx-location-history docs.

## React: `RouteView`, `RouteViewGroup`, `Link`

```tsx
import { RouteView, RouteViewGroup, Link } from 'mobx-route/react';
import type { RouteViewProps } from 'mobx-route/react';
import { routes } from '@/shared/config/routing';

<RouteViewGroup layout={Layout}>
  <RouteView route={routes.home} view={HomePage} />
  <RouteView route={routes.profile} view={ProfilePage} />
  <RouteView route={routes.items.routes.details} fallback={null}>
    {(params) => <ItemPage itemId={params.itemId} />}
  </RouteView>
  <RouteView route={routes.notFound} view={NotFoundPage} />
</RouteViewGroup>

<Link to={routes.items.routes.details} params={{ itemId: item.id }} query={{ ref: 'list' }}>
  Open item
</Link>
<Link href="https://example.com" target="_blank" rel="noreferrer">
  External
</Link>
```

- **`view`** — component; use **`RouteViewProps<typeof route>`** for typed **`params`** (and optional **`children`** from **`RouteView`**).
- **`children`** — static node or render function **`(params, route) => ...`**. If **`route`** is omitted, **`children`** always renders (function form without arguments).
- **`fallback`** — UI when the route is not open (instead of **`null`**).
- **`Link`**: **`to`** (route or string), optional **`href`** (direct URL, alternative to **`to`**), **`params`**, **`query`**, **`replace`**, **`mergeQuery`**, **`state`**, **`asChild`** — see library docs.

### Lazy-loading the view for a route

**`RouteView`** only needs a normal React component for **`view`** (or content in **`children`**). Splitting is done outside the library:

1. **`React.lazy` + `Suspense`** — define `const Page = lazy(() => import('./Page').then((m) => ({ default: m.Page })))`, then either wrap in **`<Suspense fallback={...}>`** where you render **`RouteView`**, or pass **`view`** as a thin wrapper that renders **`<Suspense><Page ... /></Suspense>`** so the fallback stays local to that route.
2. **Loadable-style helpers** (`@loadable/component`, `react-loadable`, project-specific wrappers) — resolve dynamic **`import()`** to a component and pass **that** reference as **`view`** (same as any synchronous page component).

The *Routing declarations* recipe in mobx-route docs shows exporting a loadable component from the feature folder and wiring it in **`RouteView`** once at the app shell.

### `RouteViewGroup` behavior

- Picks **one** child: first opened **`RouteView`** by default, or **last** opened when **`useLastOpened`** is set.
- If no route child is open: last **non-**`RouteView` child is static fallback UI, or **`otherwise`** runs.
- **`otherwise`**: a **route entity** or a **URL string** (e.g. `"/404"`). When set and no route is active, the group triggers navigation and renders **`null`** while that runs. You can pass **`params`**, **`query`**, **`replace`**, **`state`** on **`RouteViewGroup`** when **`otherwise`** is a route (see docs).

## Navigation (from VM / domain classes)

Use **`route.open()`** (or **`router.navigate(url, { query, replace, ... })`**). Do not push raw strings from UI components unless intentional.

**`open`** signatures (see **`Route`** / **`VirtualRoute`** docs):

```ts
open(params?, { query?, replace?, state?, mergeQuery? }): Promise<void>;
open(params?, replace?, query?): Promise<void>; // legacy positional
```

```typescript
await this.globals.router.routes.items.routes.details.open(
  { itemId: id },
  { query: { from: 'list' }, replace: false },
);

this.globals.router.routes.home.open();
```

## URL strings (`createUrl`)

```typescript
const url = this.globals.router.routes.items.routes.details.createUrl(
  { itemId: item.id },
  { ref: 'email' }, // query object
);
// Third argument: boolean mergeQuery, or { mergeQuery?, omitQuery? } — omitQuery strips search
```

Per-route or global **`createUrl`** hooks can adjust **`params`** / **`query`** before URLs are built (see **`Route`** configuration **`createUrl`** and **`routeConfig.createUrl`**).

## Path params and matching

On a matched route, read **`route.params`** (`null` when closed). **`route.path`** is the matched path segment; **`matchPath(optionalPath?)`** returns parsed **`params`** / **`path`** or **`null`**. With **`createRouter`**, same via **`router.routes...`**.

```typescript
// Example: custom page VM (only if your project uses SSR PageVM — see mobx-ssr-hydration)
class ItemPageVM extends PageVM {
  get itemId(): string | null {
    return this.globals.router.routes.items.routes.details.params?.itemId ?? null;
  }
}
```

## Guards and custom open/close

- **`Route`**: **`beforeOpen`** (return **`false`** to block, or **`{ url, state?, replace? }`** to redirect), **`checkOpened`**, **`afterOpen`** / **`afterClose`**, optional **`params()`** mapper (return **`null`** / **`false`** to treat route as not open). See *Protected routes* recipe in mobx-route docs.
- **`Route`**: **`confirmOpening`** / **`confirmClosing`** for pipeline hooks around navigation (advanced; subclassing).
- **`VirtualRoute`**: **`beforeOpen`**, **`open`**, **`close`**, **`checkOpened`**, **`getAutoOpenParams`**, etc. — modals and non-URL state (see *Modal routes* recipe).

## `RouteViewModel` (mobx-view-model integration)

**`RouteViewModel`** from **`mobx-route/view-model`** binds a route to a view model (**`pathParams`**, **`query`**, **`isMounted`** gated by **`route.isOpened`**). This is **not** the same as a project **`PageVM`** / SSR stack — see **`mvvm-architecture`** and **`mobx-ssr-hydration`** for the distinction.

## 404 / fallback (library recipe)

Prefer **`RouteViewGroup`** + **`otherwise`**, not ad-hoc `reaction` over “all routes”.

**Dedicated not-found page** — `VirtualRoute` + `RouteView` + `otherwise`:

```typescript
import { createVirtualRoute } from 'mobx-route';

export const notFound = createVirtualRoute({});
```

```tsx
<RouteViewGroup otherwise={routes.notFound}>
  <RouteView route={routes.home} view={HomePage} />
  <RouteView route={routes.notFound} view={NotFoundPage} />
</RouteViewGroup>
```

**Redirect** (e.g. to home) — virtual route whose **`open()`** replaces URL and returns **`false`** so the virtual route does not stay “open”; still use **`otherwise={notFoundRoute}`**. See mobx-route recipe *Not found routing*.

`RouteViewGroup` without `otherwise` can use a **last non-`RouteView` child** as static fallback UI; with `otherwise` set to a **route or URL string**, the group triggers that navigation when no declared route child is open.

## Route lifetime, MobX, and `destroy()`

Route entities (**`Route`**, groups, **`VirtualRoute`**) are **MobX-aware**: **`isOpened`**, **`params`**, matching, and navigation tie into observables and internal **`reaction`**s (see mobx-route source/docs).

- **One static route map for the whole app** (usual case: `routes.ts` imported once). Instances live until reload; **you do not need** **`destroy()`** — there is nothing to “turn off”.
- **Routes created at runtime** (plugins, dynamic segments of the tree, tests that construct routes per case). When an instance is **no longer used** and nothing should subscribe to it, call **`destroy()`** on **`Route`** instances (stops reactions and clears subscriptions). Omitting this can **leak** MobX disposers. If your version exposes **`destroy()`** on other entity types you allocate dynamically, apply the same rule there.

Before **`destroy()`**, ensure nothing still references that instance (**`RouteView`**, **`Link to={...}`**, globals) so you do not read disposed state.

## Utilities

- **`isRouteEntity(value)`** — type guard when a value may be a route-like object.

## Anti-patterns

- Do not call **`history.push`** / **`replace`** ad hoc from components for normal in-app navigation; use **`route.open`**, **`router.navigate`**, or **`<Link>`**.
- Do not parse **`window.location.search`** by hand for app state; use **`QueryParams`** (`createQueryParams` / **`router.query`**).
- Do not duplicate path strings; define routes once and use **`createUrl`** / **`Link to={route}`**.

## Adding a new route

1. Add **`createRoute`** / **`extend`** / **`groupRoutes`** entry (and types if needed).
2. Call **`routeConfig.update`** / **`createRouter`** if history or route tree shape changes.
3. Add **`RouteView`** (or nested group) with **`view`**, **`children`**, or a lazy wrapper; optional **`Link`** targets.
4. Add page VM + view files if the project uses that layout.
