---
name: mvvm-architecture
description: MVVM patterns with mobx-view-model — ViewModel hierarchy, withViewModel wrapper, file structure, willMount lifecycle, class composition, container access
license: MIT
compatibility: opencode
---

# MVVM Architecture with mobx-view-model

## Core Rule: Business Logic NEVER in React Components

React components are pure presentation layers. All business logic, state, computed values, and side effects belong in ViewModel classes. React hooks (`useState`, `useEffect`, `useCallback`, etc.) are banned for state management — only allowed for pure UI concerns (e.g., refs, animations).

## ViewModel hierarchy

```
ViewModelBase (mobx-view-model)
  └── VM (optional project base class)
```

Many projects add **`VM`**: extends **`ViewModelBase`**, holds **`Globals`** (or similar), overrides **`viewModels`** getter.

**SSR-specific `PageVM`**, **`didCreate()`** in **`ViewModelStore`**, **`initOnServer`**, **`ctx`**, snapshots — **not** from `mobx-view-model`. If the repo uses that stack, read **[`mobx-ssr-hydration`](../mobx-ssr-hydration/SKILL.md)** instead of inferring it from this file.

### Choosing a base class

When creating a new ViewModel:

- If the project has a custom base (**`VM`**, **`AppVM`**, …) — **extend it**.
- Otherwise extend **`ViewModelBase`** from **`mobx-view-model`**.

```typescript
import { VM } from '../shared/vm';
export class MyVM extends VM { ... }

import { ViewModelBase } from 'mobx-view-model';
export class MyVM extends ViewModelBase { ... }
```

### VM — typical project base

- Reference to the central container (stores, router, …).
- Constructor: **`(globals, ViewModelParams)`** (or your project’s shape).

## Lifecycle: `willMount()`

**`mobx-view-model`** provides **`willMount()`** — run before the VM mounts. Use it for **`makeObservable`**, subscriptions, and normal client-side setup.

```typescript
import { ViewModelBase, makeObservable } from 'mobx-view-model';
import { computed, observable } from 'mobx';

export class MyVM extends ViewModelBase {
  willMount(): void {
    makeObservable(this, {
      count: observable,
      doubled: computed,
    });
  }
}
```

Do **not** assume **`didCreate()`** or **`PageVM`** unless the project defines them for SSR — see **`mobx-ssr-hydration`**.

## File structure convention

```
my-comp/
├── model/
│   ├── index.ts        # Main VM class entry
│   ├── list.ts         # Optional: sub-models for class composition
│   └── types.ts        # Optional: VM-specific types/interfaces
├── index.tsx           # Component (withViewModel + JSX)
└── components/         # Optional
    └── my-comp-header.tsx
```

Simple case:

```
my-comp/
├── model.ts
└── index.tsx
```

## Model composition (not monolithic)

Split logic into focused classes; the VM wires them up.

```typescript
// model/index.ts
import { makeObservable } from 'mobx';
import { VM } from '../path/to/vm';
import { ProductsList } from './products-list';

export class MyCompVM extends VM {
  productsList = new ProductsList(this.globals);

  willMount(): void {
    // wire-up only
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

Sub-models take **direct dependencies** (usually **`Globals`**) — not the parent VM — unless you intentionally use nested VMs (see below).

## `withViewModel` pattern

```typescript
import { type ViewModelProps, withViewModel } from 'mobx-view-model-react'; // v10+
// v9: import { withViewModel, ViewModelProps } from 'mobx-view-model';

export interface MyCompProps extends ViewModelProps<MyCompVM> {
  children: ReactNode;
}

export const MyComp = withViewModel(
  MyCompVM,
  ({ model, children }: MyCompProps) => {
    return <div>{children}</div>;
  },
  {
    fallback: () => <div>Loading...</div>, // optional: async VM / suspense-style
  },
);
```

- **v10+**: `mobx-view-model-react` for **`withViewModel`** / **`ViewModelProps`**.
- **v9+**: same symbols from **`mobx-view-model`**.
- Prefer the **latest** **`mobx-view-model`** from npm for new work.

**When to add a VM**: only for components with meaningful own logic; dumb presentational pieces stay plain components.

## Container access

```typescript
import { ViewModelBase, makeObservable } from 'mobx-view-model';
import { computed } from 'mobx';

class ExampleVM extends ViewModelBase {
  get items() {
    return this.globals.stores.products.items;
  }

  willMount(): void {
    makeObservable(this, {
      items: computed,
    });
  }
}
```

Stores: **`this.globals.stores.<name>`**; router: **`this.globals.router`** (names depend on the project).

## Nested ViewModels (child VM with parent)

```typescript
import { ViewModelBase } from 'mobx-view-model';

export class ChildVM extends ViewModelBase<{}, ParentVM> {
  get parentData() {
    return this.parentViewModel.someQuery.data;
  }
}
```

Prefer **composition** + **`Globals`** when possible — fewer hard dependencies on a specific parent VM class.

### Avoid deep `parentViewModel` chains

Long chains like **`this.parentViewModel.parentViewModel.parentViewModel`** are a smell: they encode **implicit navigation** through the UI tree, break when the hierarchy moves, and couple the child to many concrete parent types.

Prefer one of these instead:

- **One hop only**: if a child VM needs the parent, use **`this.parentViewModel`** for the **immediate** parent and expose deeper data **through that parent** (methods or small read-only props), not by walking the chain.
- **`Globals` / stores**: shared reads and writes live in **`this.globals.stores…`** (or services on **`Globals`**) so any VM depth can access the same source of truth without knowing ancestors.
- **Explicit injection**: pass **`Globals`**, a **narrow interface** (callbacks or facades), or a **dedicated coordinator** into the child’s constructor when wiring sub-models — same idea as “constructor deps, not locator chains.”
- **Lift coordination**: if several nested VMs must collaborate, move orchestration to a **store** or a **single parent VM** that owns the subtree; children stay shallow.

Full **`mobx-view-model`** reference: https://js2me.github.io/mobx-view-model/llms-full.txt

## Prohibited patterns

- ❌ `useState` / `useEffect` / `useCallback` for business logic  
- ❌ Fetching or mutating domain state in components  
- ❌ Derived state in components  
- ❌ Deep **`parentViewModel.parentViewModel…`** chains — use **`Globals`/stores**, **explicit deps**, or **one parent** that exposes what children need  
- ✅ State, async work, and computations in the VM  
