---
name: mobx-stores
description: MobX store patterns — makeObservable, observable collections, async mutations, debounced sync, Globals central container, client-only persistence
license: MIT
compatibility: opencode
---

# MobX Store Patterns

## Store Declaration

```typescript
import { computed, makeObservable, observable } from 'mobx';

export class ExampleStore {
  // Observable primitives
  isLoading = true;
  error: string | null = null;

  // Observable collections
  private itemsMap = observable.map<number, ItemDC>();
  private selectedIds = observable.set<number>();

  constructor() {
    makeObservable(this, {
      isLoading: observable.ref,
      error: observable.ref,
      items: computed,
      headerCount: computed,
    });
  }
}
```

### Minimal Store Template Checklist

- Define observable state fields (primitives + collections).
- Call `makeObservable(this, { ... })` in constructor.
- Expose derived data via `computed` getters.
- Keep async loading/mutations in store methods.
- Keep network payload-to-state mapping in dedicated helper methods (`applyData`, `applyServerData`).

Key rules:
- No decorators — configure `useDecorators: false` in `ViewModelStoreBase` options
- `makeObservable` in constructor for stores, in `didCreate()` or `mount()` for ViewModels
- `observable.ref` — primitives and immutable plain objects
- `observable.shallow` — maps, sets
- `observable.set()` / `observable.map()` — reactive collections
- `computed` — derived values
- `computed.struct` — arrays/objects compared by value (not reference)

## Async Actions & Mutations

```typescript
class ExampleStore {
  load = async () => {
    this.isLoading = true;
    try {
      const data = await fetchData();
      this.applyData(data);
    } catch {
      this.error = 'Failed to load';
    } finally {
      this.isLoading = false;
    }
  };

  // Synchronous mutation
  toggleItem = (id: number) => {
    if (this.selectedIds.has(id)) {
      this.selectedIds.delete(id);
    } else {
      this.selectedIds.add(id);
    }
  };
}
```

- `runInAction` / `action` wrappers are NOT needed — see `mobx-general`

## Observable Collections

```typescript
// Map — ordered key-value with insertion order tracking
private itemsMap = observable.map<number, ItemDC>();

get items(): ItemInfo[] {
  return Array.from(this.itemsMap.values()).map(/* enrich */);
}

// Set — unique values
selectedIds = observable.set<number>();
addingIds = observable.set<number>();
```

## Debounced Server Sync Pattern

```typescript
import { debounce } from 'es-toolkit/function';

class Store {
  private readonly syncServer = debounce(async () => {
    this.syncInFlight++;
    const requestId = ++this.syncRequestId;
    try {
      const data = await putData(this.getPayload());
      if (requestId !== this.syncRequestId) return; // stale guard
      this.applyServerData(data);
    } finally {
      this.syncInFlight--;
    }
  }, 350);

  // Call syncServer() after every mutation
  addItem = async (id: number) => {
    // optimistic update
    this.itemsMap.set(id, newItem);
    this.syncServer(); // debounced
  };
}
```

No `runInAction` wrappers needed — MobX tracks mutations automatically when `enforceActions: 'never'`.

## Globals — Central Application Container

For SSR applications it's **strongly recommended** to have a central `Globals` class that holds all application context — router, stores, SSR API, and services:

```typescript
export class Globals {
  readonly isClient: boolean;
  readonly isServer: boolean;

  readonly router: Router;
  readonly stores: {
    items: ItemsStore;
    cart: CartStore;
    favorites: FavoritesStore;
    viewModels: ViewModelsStore;
  };

  constructor(params: GlobalsCreateParams) {
    this.isClient = typeof window !== 'undefined';
    this.isServer = !this.isClient;
    this.router = new Router(params.router);
    this.stores = {
      items: new ItemsStore(this.router),
      cart: new CartStore(this.router),
      favorites: new FavoritesStore(),
      viewModels: new ViewModelsStore(this, params.pageContexts),
    };
  }

  get ssr() {
    return this.params.ssr; // SSR API, only on server
  }

  static fromSnapshot(ssrData: AnyObject): Globals {
    return new Globals(ssrData);
  }

  toSnapshot() {
    return {
      ...this.params,
      pageContexts: this.stores.viewModels.loadedContexts,
    };
  }
}
```

Why `Globals` is essential:
- **SSR**: Each request gets its own instance with isolated stores and memory history
- **Hydration**: `Globals.fromSnapshot(window.__SSR_DATA__)` restores server state on client
- **DI without React context**: ViewModels receive `Globals` in constructor — no Context providers
- **Per-request isolation**: Fresh stores for every SSR request, no data leaks between users

For CSR-only projects `Globals` is optional but still useful as a single source of truth (can be a singleton instead of per-request).

## Store Composition

Stores are instantiated in the `Globals` constructor and accessed via `globals.stores.<name>`:

```typescript
// In any VM:
this.globals.stores.items.method()
this.globals.stores.cart.property
```

Stores receive only their direct dependencies. They can reference each other through the container.

## Client-Only Persistence

```typescript
import { storageData } from 'mobx-web-api';

class FavoritesStore {
  private readonly storedIds = storageData.key<number[]>(
    'favoriteIds',
    [],      // default
    'session', // storage type: 'session' | 'local'
  );

  constructor() {
    this.ids = [...this.storedIds.value];
  }

  private setIds(ids: number[]) {
    this.ids = ids;
    this.storedIds.value = ids; // auto-syncs to storage
  }
}
```
