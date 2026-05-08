---
name: routing-navigation
description: Routing and navigation with mobx-route and mobx-location-history — route definition, createQueryParams for reactive URL params, RouteView, navigation actions, 404 handling
license: MIT
compatibility: opencode
---

# Routing & Navigation (mobx-route)

Full documentation:
- mobx-route: https://js2me.github.io/mobx-route/llms-full.txt
- mobx-location-history: https://js2me.github.io/mobx-location-history/llms-full.txt

## Route Definition

Routes are defined using `mobx-route`:

```typescript
import { createRoute, createVirtualRoute, Router as RouterLib } from 'mobx-route';

const defineRoutes = () => ({
  home: createRoute('/', { exact: true }),
  profile: createRoute('/profile', { exact: true }),
  item: createRoute('/items/:itemId', { exact: true }),
  notFound: createVirtualRoute(),
});
```

### History
- Server: `createMemoryHistory` with `initialEntries: [req.path]`
- Client: `createBrowserHistory`

Both from `mobx-location-history`.

### Query Params

For reading and updating query parameters, use `createQueryParams` from `mobx-location-history`. It provides reactive observable access to URL search params:

```typescript
import { createQueryParams } from 'mobx-location-history';

this.query = createQueryParams({ history });

// Read params (reactive — component re-renders on change)
this.query.params.page    // → "2"
this.query.params.search  // → "hello"

// Update params
this.query.setParam('page', '3');
this.query.setParams({ page: '3', search: 'hello' });
this.query.removeParam('page');

// Replace all params at once
this.query.replaceParams({ search: 'new' });
```

`query.params` is an observable — any component or computed that reads it will react to URL changes automatically.

## Route Registration

```typescript
import { RouteView, RouteViewGroup } from 'mobx-route/react';

<RouteViewGroup layout={Layout}>
  <RouteView route={router.routes.home} view={HomePage} />
  <RouteView route={router.routes.profile} view={ProfilePage} />
  <RouteView route={router.routes.item} view={ItemPage} />
  <RouteView route={router.routes.notFound} view={NotFoundPage} />
</RouteViewGroup>
```

Pages are lazy-loaded with `React.lazy()`.

## Navigation Actions

```typescript
// In a ViewModel:
class ExampleVM extends VM {
  goToItem = (id: string) => {
    this.globals.router.routes.item.open({ itemId: id });
  };

  goHome = () => {
    this.globals.router.routes.home.open();
  };
}
```

## URL Creation (for hrefs)

```typescript
// Generate URL string for links
const url = this.globals.router.routes.item.createUrl({
  itemId: item.id,
});
// → "/items/42"
```

## Route Parameters Access

```typescript
class ExamplePageVM extends PageVM {
  get itemId(): string | null {
    return this.globals.router.routes.item.params?.itemId ?? null;
  }
}
```

## 404 Handling

Automatic via `reaction` in Router constructor:

```typescript
reaction(
  () => routeKeys.every(key => !this.routes[key].isOpening && !this.routes[key].isOpened),
  (isAllClosed) => {
    if (isAllClosed) this.routes.notFound.open();
  },
  { fireImmediately: true },
);
```

### Anti-Patterns to Avoid

- Don't call `history.push()` directly in components; navigate via route objects (`route.open`) from VM/store.
- Don't parse `window.location.search` manually; use `createQueryParams` for reactive params.
- Don't hardcode route strings in multiple places; define once in router and use `createUrl()`.

## Adding a New Route

1. Add route definition
2. Create page files with VM + component
3. Register in route view tree
