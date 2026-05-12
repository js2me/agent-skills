---
name: mobx-stores
description: MobX domain/state classes — makeObservable, observable collections, async mutations, debounced sync, Globals central container, client-only persistence
license: MIT
compatibility: opencode
---

# MobX domain state classes

MobX-backed classes that hold app state and side effects are often called “stores” in docs. **Class names should reflect what they represent or do** (`Cart`, `Products`, `Favorites`, `SessionSettings`) — **not** a mechanical `SomethingStore` suffix.

## Declaration

```typescript
import { computed, makeObservable, observable } from 'mobx';

export class Products {
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

### Checklist

- Define observable state fields (primitives + collections).
- Call `makeObservable(this, { ... })` in constructor.
- Expose derived data via `computed` getters.
- Keep async loading/mutations in class methods.
- Keep network payload-to-state mapping in dedicated helper methods (`applyData`, `applyServerData`).

Key rules:
- No decorators — configure `useDecorators: false` in `ViewModelStoreBase` options
- `makeObservable` in constructor for these classes; for ViewModels use **`willMount()`** (or your base’s **`mount()`**). SSR stacks that call **`didCreate()`** before mount are covered in **`mobx-ssr-hydration`**.
- `observable.ref` — primitives and immutable plain objects
- `observable.shallow` — maps, sets
- `observable.set()` / `observable.map()` — reactive collections
- `computed` — derived values
- `computed.struct` — arrays/objects compared by value (not reference)

## Async actions & mutations

```typescript
class Products {
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

## Observable collections

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

## Debounced server sync pattern

```typescript
import { debounce } from 'es-toolkit/function';

class Workspace {
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

## Globals — central application container

For SSR applications it's **strongly recommended** to have a central `Globals` class that holds all application context — router, state roots, SSR API, and services:

```typescript
export class Globals {
  readonly isClient: boolean;
  readonly isServer: boolean;

  readonly router: Router;
  readonly stores: {
    products: Products;
    cart: Cart;
    favorites: Favorites;
    viewModels: ViewModels;
  };

  constructor(params: GlobalsCreateParams) {
    this.isClient = typeof window !== 'undefined';
    this.isServer = !this.isClient;
    this.router = new Router(params.router);
    this.stores = {
      products: new Products(this.router),
      cart: new Cart(this.router),
      favorites: new Favorites(),
      viewModels: new ViewModels(this, params.pageContexts),
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
- **SSR**: Each request gets its own instance with isolated state and memory history
- **Hydration**: `Globals.fromSnapshot(window.__SSR_DATA__)` restores server state on client
- **DI without React context**: ViewModels receive `Globals` in constructor — no Context providers
- **Per-request isolation**: Fresh roots for every SSR request, no data leaks between users

For CSR-only projects `Globals` is optional but still useful as a single source of truth (can be a singleton instead of per-request).

The `stores` field name is a common bucket for these roots; keys (`products`, `cart`, …) should match **domain** names, not `productsStore`.

## Composition & access

Roots are created in the `Globals` constructor and reached via `globals.stores.<key>` (or rename the bucket to `roots` / `models` if the project prefers):

```typescript
// In any VM:
this.globals.stores.products.method()
this.globals.stores.cart.property
```

Each class receives only its direct dependencies. They can reference each other through the container.

## Client-only persistence

```typescript
import { storageData } from 'mobx-web-api';

class Favorites {
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
