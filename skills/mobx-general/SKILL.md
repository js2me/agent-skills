---
name: mobx-general
description: MobX general rules — configure enforceActions never, no action/runInAction wrappers needed, direct observable mutations in async code
license: MIT
compatibility: opencode
---

# MobX General: No action / runInAction Wrappers

## enforceActions: 'never'

The project should use `configure({ enforceActions: 'never' })` at the application bootstrap:

```typescript
import { configure } from 'mobx';

configure({ enforceActions: 'never' });
```

This means **MobX does not require `action` or `runInAction` wrappers**. All observable mutations work directly, including inside async functions.

## Rule: default to no action / runInAction

```typescript
// ❌ WRONG — runInAction is redundant
class Store {
  load = async () => {
    runInAction(() => { this.isLoading = true; });
    try {
      const data = await fetch();
      runInAction(() => { this.data = data; });
    } finally {
      runInAction(() => { this.isLoading = false; });
    }
  };
}

// ✅ CORRECT — mutate observable directly
class Store {
  load = async () => {
    this.isLoading = true;
    try {
      this.data = await fetch();
    } finally {
      this.isLoading = false;
    }
  };
}
```

## Why you might see runInAction in existing code

Some codebases may contain `runInAction` from before this config was set. Write **new code** without it:

```typescript
// Legacy style — do NOT repeat
runInAction(() => {
  this.items.set(id, item);
});

// Correct new code:
this.items.set(id, item);
```

## Exceptions

`action` / `runInAction` is only needed if you explicitly pass a mutator function externally (e.g., as a callback outside the reactive context). But within MVVM architecture, all mutations stay inside the VM/Store, so they are almost never required.

## Why this works

By default MobX strict mode requires `action`/`runInAction` to ensure predictable changes and prevent mutations outside action context. `enforceActions: 'never'` disables this check, allowing direct mutations from anywhere. This simplifies code (especially async operations) without losing reactivity.
