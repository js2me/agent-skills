---
name: mvvm-architecture
description: MVVM patterns with mobx-view-model — ViewModel hierarchy, withViewModel wrapper, file structure, lifecycle hooks (willMount vs didCreate), class composition, container access
license: MIT
compatibility: opencode
---

# MVVM Architecture with mobx-view-model

## Core Rule: Business Logic NEVER in React Components

React components are pure presentation layers. All business logic, state, computed values, and side effects belong in ViewModel classes. React hooks (`useState`, `useEffect`, `useCallback`, etc.) are banned for state management — only allowed for pure UI concerns (e.g., refs, animations).

## ViewModel Hierarchy

```
ViewModelBase (mobx-view-model)
  └── VM (project base class)
        └── PageVM (page-level with SSR)
```

It's common to create project-level base classes:

- **`VM`** — extends `ViewModelBase`, adds a reference to a central DI container (typically called `Globals` or similar), overrides `viewModels` getter
- **`PageVM`** — extends `VM`, adds SSR lifecycle (`onInit`, `initOnServer`, `initOnClient`), typed `ctx`, `isInitializing` flag

### Choosing a Base Class

When creating a new ViewModel:

- If the project has a custom base class (e.g. `VM`, `AppVM`, `OwnVM` extending `ViewModelBase`) — **extend that class** to stay consistent with the project's conventions
- If no custom base class exists — extend `ViewModelBase` directly from `mobx-view-model`

```typescript
// Project has a custom base — use it
import { VM } from '../shared/vm';

export class MyVM extends VM { ... }

// No custom base — use ViewModelBase directly
import { ViewModelBase } from 'mobx-view-model';

export class MyVM extends ViewModelBase { ... }
```

### VM — base class for all ViewModels
- Holds reference to a central container (stores, router, etc.)
- Has empty `didCreate()` hook for subclasses (custom lifecycle, not part of mobx-view-model)
- Constructor typically accepts the container + `ViewModelParams`

### PageVM — page-level ViewModel with SSR support
- Generic over `TPageContext` (data loaded during SSR)
- `onInit(fn)` — registers init function receiving SSR API or null
- `initOnServer(ssr)` — called during SSR, returns context
- `initOnClient()` — called in `mount()` on client, re-runs init if no cached context
- `ctx` — holds page context data from SSR or client fetch
- `isInitializing` — observable flag

### Lifecycle Hooks: `willMount()` vs `didCreate()`

mobx-view-model provides the standard lifecycle hook `willMount()` — called before the VM mounts.

Some projects additionally define a `didCreate()` hook called immediately after construction (inside a custom `ViewModelStore.createViewModel`). This is **not** a standard mobx-view-model API — it's a custom convention for projects that need to run initialization code before `mount()` (e.g. for SSR data fetching).

### How `didCreate()` is implemented

```typescript
import { ViewModelBase, ViewModelCreateConfig, ViewModelStoreBase, mergeVMConfigs } from 'mobx-view-model';

class ViewModelsStore extends ViewModelStoreBase {
  createViewModel<TVM extends AnyViewModel>(
    config: ViewModelCreateConfig<AnyViewModel>,
  ): TVM {
    const vm = new VMClass(this.globals, {
      ...config,
      vmConfig: mergeVMConfigs(this.vmConfig, config.vmConfig),
    });

    // didCreate() runs immediately after construction —
    // before mount(), before React renders
    vm.didCreate();

    // For SSR: kick off async data loading right away
    if (vm instanceof PageVM && !vm.ctx && this.globals.isServer) {
      vm.initOnServer(this.globals.ssr).then(() => {
        // context is now ready for render
      });
    }

    return vm;
  }
}
```

### Why `didCreate()` instead of `willMount()`?

- **`willMount()`** runs later, during React's component mount phase. By that time, SSR needs the data to already be loaded.
- **`didCreate()`** runs synchronously right after `new VMClass()` — this allows the VM to register its `onInit` callback and for `ViewModelStore` to immediately start SSR data fetching before React begins rendering the component tree.

The pattern is: `didCreate()` registers init functions → `createViewModel` sees the VM needs SSR data → starts async fetch → rendering waits via `suspendUntil`. Without `didCreate()`, the SSR data fetch would only start during mount, which is too late for server-side rendering.

```typescript
// Standard mobx-view-model — use willMount()
import { ViewModelBase, makeObservable } from 'mobx-view-model';

export class MyVM extends ViewModelBase {
  willMount(): void {
    makeObservable(this, { ... });
    // initialization logic
  }
}

// Project with custom base that supports didCreate()
import { VM } from '../shared/vm';

export class MyVM extends VM {
  didCreate(): void {
    makeObservable(this, { ... });
    // initialization logic
  }
}
```

Check the project's base class to determine which hook to use. If extending `ViewModelBase` directly, use `willMount()`.

## File Structure Convention

Components with a ViewModel follow a strict directory layout:

```
my-comp/
├── model/
│   ├── index.ts        # Main VM class entry
│   ├── list.ts         # Optional: sub-models for class composition
│   └── types.ts         # Optional: VM-specific types/interfaces
├── index.tsx            # Component (withViewModel wrapper + JSX)
└── components/          # Optional: sub-components for this feature
    └── my-comp-header.tsx
```

For simple components the model can be a single file:

```
my-comp/
├── model.ts             # Single file VM
└── index.tsx            # Component
```

## Model Composition (not monolithic)

`model.ts` / `model/index.ts` is NOT a single file where you dump everything. Use class composition to split logic into focused sub-classes:

```typescript
// model/index.ts
import { makeObservable } from 'mobx';
import { VM } from '../path/to/vm';
import { ProductsList } from './products-list';

export class MyCompVM extends VM {
  // Delegate to a dedicated sub-class with only its dependencies
  productsList = new ProductsList(this.globals);

  willMount(): void {
    // VM only wires up sub-classes, all logic lives in them
  }
}
```

```typescript
// model/products-list.ts
import { computed, makeObservable, observable } from 'mobx';
import type { Globals } from '../path/to/globals';

export class ProductsList {
  items: Item[] = [];
  isLoading = false;

  get hasItems(): boolean {
    return this.items.length > 0;
  }

  constructor(private globals: Globals) {
    makeObservable(this, {
      items: observable.ref,
      isLoading: observable.ref,
      hasItems: computed,
    });
  }

  load = async () => {
    this.isLoading = true;
    try {
      this.items = await fetchItems();
    } finally {
      this.isLoading = false;
    }
  };
}
```

Sub-classes receive only their direct dependencies (typically the central container) — never the parent VM. This keeps them decoupled, testable, and reusable across different VMs.

## withViewModel Pattern

Always wrap components that need a ViewModel:

```typescript
import { type ViewModelProps, withViewModel } from 'mobx-view-model-react'; // v10+
// OR for v9+: import { withViewModel, ViewModelProps } from 'mobx-view-model';

// Component props interface
export interface MyCompProps extends ViewModelProps<MyCompVM> {
  children: ReactNode;
}

// Component with ViewModel
export const MyComp = withViewModel(
  MyCompVM,
  ({ model, children }: MyCompProps) => {
    // Pure JSX only — no business logic
    // Access all state/actions via `model`
    return <div>{children}</div>;
  },
  {
    // Optional: skeleton/loading fallback for SSR
    fallback: () => <div>Loading...</div>,
  },
);
```

Key points:
- `withViewModel` and `ViewModelProps` imports depend on mobx-view-model version:
  - **10+**: `import { withViewModel } from 'mobx-view-model-react'`
  - **9+**: `import { withViewModel } from 'mobx-view-model'`
- When starting a new project, always use the **latest published version** from npm (check `npm view mobx-view-model version`)
- Component receives `{ model }` as prop (typed as the VM class)
- 3rd arg `options.fallback` renders during SSR suspense

**When to create a VM**: Only for components with significant own business logic. Simple presentational components stay as plain functions/props — no `withViewModel` wrapper needed.

## Page VM Lifecycle (when using PageVM)

```
1. ViewModelsStore.createViewModel(config)
2.   → new VMClass(globals, config)
3.   → vm.didCreate() (for SSR code only) or vm.willMount() — register init fn, makeObservable
4.   → if PageVM + server: vm.initOnServer(ssr)
5.     → run initFn(ssr) → fetch data → set ctx → set head
6.   → ViewModelsStore waits for all loading contexts
7. React renders component → user sees page
8. On client hydration: vm.mount() → makeObservable → initOnClient()
9.   → if context exists from SSR skip, else fetch
```

## Container Access Pattern

```typescript
class ExampleVM extends ViewModelBase {
  get items() {
    return this.globals.stores.someStore.items;
  }

  willMount(): void {
    makeObservable(this, {
      items: computed,
    });
  }
}
```

All stores are accessed via the central container (e.g. `this.globals.stores.<name>`). Router via `this.globals.router`.

## Nested ViewModels (Child VM with Parent)

When a sub-component needs its own business logic but depends on page state:

```typescript
import { ViewModelBase } from 'mobx-view-model';

export class ChildVM extends ViewModelBase<{}, ParentVM> {
  // Access parent data via this.parentViewModel
  get parentData() {
    return this.parentViewModel.someQuery.data;
  }
}
```

For most cases prefer **class composition** (see above) — sub-classes receive the central container directly. This is more flexible and avoids coupling to a specific VM class.

Full mobx-view-model documentation: https://js2me.github.io/mobx-view-model/llms-full.txt

## Prohibited Patterns

- ❌ `useState` / `useEffect` / `useCallback` for business logic
- ❌ Fetching data inside components
- ❌ Computed/derived state in components
- ❌ Mutating state in components
- ✅ All state, computations, async operations in VM
