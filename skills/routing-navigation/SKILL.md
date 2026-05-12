---
name: routing-navigation
description: Routing with mobx-route and mobx-location-history — routeConfig, history, createRoute/groupRoutes/extend, createRouter, QueryParams (.data/.update), RouteViewGroup.otherwise, Link, navigation and URLs
license: MIT
compatibility: opencode
---

# Routing & Navigation (mobx-route)

Full documentation:

- mobx-route: https://js2me.github.io/mobx-route/llms-full.txt
- mobx-location-history: https://js2me.github.io/mobx-location-history/llms-full.txt

Paths use [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) syntax (including optional segments, rest params, etc.).

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

For **non-browser** environments (SSR, tests, RN-style shells), see the library recipe *Memory routing*: `routeConfig.update({ history: createMemoryHistory(...) })`.

## Declaring routes

Prefer **`groupRoutes()`** for nested trees; use **`extend()`** to attach child paths to a parent route (merged path, shared hierarchy).

```typescript
import { createRoute, createVirtualRoute, groupRoutes } from 'mobx-route';

const home = createRoute('/', { exact: true });
const profile = createRoute('/profile', { exact: true });
const items = groupRoutes({
  list: createRoute('/items', { index: true }),
  details: createRoute('/items/:itemId'),
});
const notFound = createVirtualRoute();

export const routes = {
  home,
  profile,
  items,
  notFound,
};
```

`{ index: true }` marks an index route inside a group (used by `group.open()` and matching). `{ exact: true }` keeps a prefix route from staying “open” when a longer path matches another route — pick one style per route based on docs / behavior you need.

## Optional: `createRouter` (central `Router`)

For a single object that exposes **`routes`**, **`history`**, **`location`**, **`query`**, and **`navigate()`**:

```typescript
import {
  createBrowserHistory,
  createRoute,
  createRouter,
  createQueryParams,
  routeConfig,
} from 'mobx-route';

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

Whether you keep a **`Router`** instance on `globals` or only export **`routes`** + **`routeConfig`** depends on the project; APIs below work on **route entities** (`Route`, `VirtualRoute`, `RouteGroup`) the same way.

## Query parameters (`QueryParams`)

Use **`createQueryParams({ history })`** (often wired through **`routeConfig`** / **`createRouter`**). The observable snapshot of the current search string is **`queryParams.data`** (plain object of string values), **not** `.params`.

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
import { routes } from '@/shared/config/routing';

<RouteViewGroup layout={Layout}>
  <RouteView route={routes.home} view={HomePage} />
  <RouteView route={routes.profile} view={ProfilePage} />
  <RouteView
    route={routes.items.routes.details}
    loadView={async () => (await import('@/pages/item')).ItemPage}
    loading={GlobalLoader}
  />
  <RouteView route={routes.notFound} view={NotFoundPage} />
</RouteViewGroup>

<Link to={routes.items.routes.details} params={{ itemId: item.id }} query={{ ref: 'list' }}>
  Open item
</Link>
```

- **`loadView`** — async code-splitting; optional **`loading`** UI.
- **`Link`**: `to` (route or string), `params`, `query`, `replace`, `mergeQuery`, `state`, `asChild` — see library docs.

## Navigation (from VM / domain classes)

Use **`route.open()`** (or **`router.navigate(url)`**). Do not push raw strings from UI components unless intentional.

```typescript
// Signature (see Route docs): open(params?, options?) — options: query, replace, state, mergeQuery
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
// Third argument: boolean mergeQuery, or { omitQuery: true } to strip search
```

## Path params

On a matched route, read **`route.params`** (library examples often use this in `observer` components). With **`createRouter`**, same via `router.routes...`.

```typescript
// Example: custom page VM (only if your project uses SSR PageVM — see mobx-ssr-hydration)
class ItemPageVM extends PageVM {
  get itemId(): string | null {
    return this.globals.router.routes.items.routes.details.params?.itemId ?? null;
  }
}
```

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

**Redirect** (e.g. to home) — virtual route whose `open()` replaces URL and returns `false` so the virtual route does not stay “open”; still use `otherwise={notFoundRoute}`. See mobx-route recipe *Not found routing*.

`RouteViewGroup` without `otherwise` can use a **last non-`RouteView` child** as static fallback UI; with `otherwise` set to a **route or URL string**, the group triggers that navigation when no declared route child is open.

## Anti-patterns

- Do not call **`history.push`** / **`replace`** ad hoc from components for normal in-app navigation; use **`route.open`**, **`router.navigate`**, or **`<Link>`**.
- Do not parse **`window.location.search`** by hand for app state; use **`QueryParams`** (`createQueryParams` / `router.query`).
- Do not duplicate path strings; define routes once and use **`createUrl`** / **`Link to={route}`**.

## Adding a new route

1. Add **`createRoute`** / **`groupRoutes`** / **`extend`** entry (and types if needed).
2. Call **`routeConfig.update`** / **`createRouter`** if history or route tree shape changes.
3. Add **`RouteView`** (or nested group) and optional **`Link`** targets.
4. Add page VM + view files if the project uses that layout.
