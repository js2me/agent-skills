---
name: mobx-ssr-hydration
description: Custom MobX SSR & hydration — PageVM (not mobx-view-model), onInit/initOnServer/initOnClient, ctx, ViewModelsStore + didCreate + suspendUntil, SSRApi, snapshot/hydration, guards
license: MIT
compatibility: opencode
---

# MobX SSR & hydration (custom project stack)

This skill describes a **project-specific** SSR integration: a **`PageVM`** base class, **`ViewModelsStore.createViewModel`** hooks, **`didCreate()`**, typed **`ctx`**, and in-process **`SSRApi`**. None of this is part of **`mobx-view-model`** or **`mobx-route`**.

## When to use this skill

- The codebase defines **`PageVM`**, server **`initOnServer`**, **`onInit(ssr => …)`**, and hydrates from a snapshot (`window.__SSR_DATA__` or equivalent).
- If the app is **CSR-only**, uses **RSC-only**, or SSR is wired **differently** — **ignore** this skill and use **`mvvm-architecture`** + your own data-loading story.

## Relation to other skills

| Topic | Skill |
|--------|--------|
| General MVVM, `withViewModel`, `willMount`, file layout | [`mvvm-architecture`](../mvvm-architecture/SKILL.md) |
| DC types, fetch helpers, SSR API method shapes | [`api-data-contracts`](../api-data-contracts/SKILL.md) |
| `Globals`, per-request roots | [`mobx-stores`](../mobx-stores/SKILL.md) |
| `mobx-route`, history, query | [`routing-navigation`](../routing-navigation/SKILL.md) |

---

## PageVM — responsibilities

Implement in **your** codebase (extends project `VM` → `ViewModelBase`).

Typical surface:

- Generic over **`TPageContext`** — data needed for the first paint.
- **`onInit(fn)`** — registers an init function receiving **`SSRApi | null`** (server) or **`null`** (client branch).
- **`initOnServer(ssr)`** — invoked on the server from the store; runs init and fills **`ctx`**.
- **`initOnClient()`** — invoked from **`mount()`** on the client; if **`ctx`** came from the snapshot, skip refetch; else run init with `ssr = null` (HTTP / client API).
- **`ctx`** — page context (from SSR or client).
- **`isInitializing`** — observable flag while client init runs.

Exact names may vary; align with the repo’s `PageVM` implementation.

---

## ViewModelsStore + `didCreate()` + `suspendUntil`

**`mobx-view-model`** exposes **`ViewModelStoreBase`** and **`mergeVMConfigs`**. Projects override **`createViewModel`** to:

1. **`new VMClass(…)`**
2. Call **`vm.didCreate()`** immediately (before React mount) so **`onInit`** is registered and **`makeObservable`** can run early.
3. If **`PageVM`** on server with empty **`ctx`**, call **`initOnServer(this.globals.ssr)`** and track loading (e.g. **`loadingContexts`**) until context is ready.
4. Use **`suspendUntil`** in **`vmConfig`** so SSR render waits until contexts finish loading.

```typescript
import {
  type AnyViewModel,
  mergeVMConfigs,
  type ViewModelCreateConfig,
  ViewModelStoreBase,
} from 'mobx-view-model';
import type { PageVM } from '../path/to/page-vm';
import type { VM } from '../path/to/vm';

class ViewModelsStore extends ViewModelStoreBase<VM> {
  createViewModel<TVM extends AnyViewModel>(
    config: ViewModelCreateConfig<AnyViewModel>,
  ): TVM {
    const Ctor = config.VM as new (...args: unknown[]) => VM | PageVM;
    const vm = new Ctor(this.globals, {
      ...config,
      vmConfig: mergeVMConfigs(this.vmConfig, config.vmConfig),
    });

    vm.didCreate();

    if (vm instanceof PageVM && !vm.ctx && this.globals.isServer) {
      this.loadingContexts.add(vm.id);
      void vm
        .initOnServer(this.globals.ssr)
        .then(() => {
          this.loadedContexts[vm.id] = vm.ctx;
        })
        .finally(() => {
          this.loadingContexts.delete(vm.id);
        });
    }

    return vm as TVM;
  }
}
```

Wire **`suspendUntil`** in **`ViewModelStoreBase`** options (e.g. **`when(() => !loadingContexts.size)`** on server) so HTML render waits for all **`initOnServer`** calls tracked in **`loadingContexts`**.

Adapt **`loadedContexts`**, **`loadingContexts`**, and **`when`** / **`suspendUntil`** to match your **`Globals`** and SSR pipeline.

### Why `didCreate()` for SSR (vs only `willMount()`)

- **`willMount()`** runs when the VM mounts with React — on SSR, data is often needed **before** that moment.
- **`didCreate()`** runs right after **`new VMClass()`** so **`onInit`** is registered and the store can start **`initOnServer`** before the tree renders, while **`suspendUntil`** blocks render until data is ready.

For VMs **without** this SSR pipeline, use **`willMount()`** only (see **`mvvm-architecture`**).

---

## End-to-end lifecycle

```
1. ViewModelsStore.createViewModel(config)
2.   → new VMClass(globals, config)
3.   → vm.didCreate() — register onInit, makeObservable
4.   → if PageVM + server + no ctx: vm.initOnServer(ssr)
5.     → onInit(ssr) → data layer / SSRApi → set ctx, head meta
6.   → store waits until loading contexts drain (suspendUntil / when)
7.   → React renders page
8. Client: vm.mount() → initOnClient()
9.   → if ctx from snapshot — skip fetch; else onInit(null) / HTTP
```

---

## Data flow in `onInit`

### Server path (`ssr` is defined)

```typescript
didCreate(): void {
  makeObservable(this, { /* … */ });

  this.onInit(async (ssr) => {
    if (ssr) {
      const item = await ssr.getItemById(this.id);
      const related = await ssr.getRelatedById(item!.relatedId);

      ssr.head.title = item!.title;
      ssr.head.ogTitle = item!.title;

      this.ctx = { item, related };
      return;
    }

    if (!this.ctx) {
      const item = await loadItemById(this.id);
      this.ctx = { item, related: await loadRelated(item!.relatedId) };
    }
  });
}
```

### Client `mount()` / `initOnClient`

- If **`ctx`** was restored from the hydration snapshot → **do not** refetch the same payload immediately.
- If no **`ctx`** (client navigation to this page) → run client/API loading and set **`ctx`**.
- Toggle **`isInitializing`** while async work runs (if your `PageVM` exposes it).

### Rendering guard

```tsx
// In withViewModel view or route view
if (model.isInitializing && !model.item) {
  return <LoadingSkeleton />;
}
if (!model.item) {
  return <NotFound />;
}
return <FullPage />;
```

Use **`withViewModel(..., { fallback: () => … })`** where appropriate for suspense-style loading.

---

## SSRApi (in-process server API)

Shape is **project-defined**; keep methods typed and DC-backed (see **`api-data-contracts`**).

```typescript
interface SSRApi {
  head: HeadApi; // title, ogTitle, ogDescription, ogImage, ogUrl, ogType, …
  getItemById(id: number): Promise<ItemDC | null>;
  getRelatedById(id: number): Promise<RelatedDC | null>;
  getSystemInfo(): { date: string };
}
```

Set head meta only when **`ssr`** is present:

```typescript
ssr.head.title = 'Page Title';
ssr.head.ogTitle = 'OG Title';
ssr.head.ogDescription = 'Description';
ssr.head.ogImage = '/path/to/image';
ssr.head.ogUrl = `/items/${id}`;
ssr.head.ogType = 'product';
```

---

## Server-side caching

SSR API often uses an **in-memory cache** keyed by session (or similar):

```typescript
const CACHE_TTL_MS = 3 * 24 * 60 * 60 * 1000;
// Map<sessionId, { value: SSRApi; expiresAt: number }>
```

Reuse one API instance per session across SSR requests where appropriate.

---

## Hydration and snapshot

```typescript
// Client entry (names illustrative)
const globals = Globals.fromSnapshot((globalThis as unknown as { __SSR_DATA__: unknown }).__SSR_DATA__);

hydrateRoot(document.getElementById('root')!, <App globals={globals} />);
```

Snapshot is produced on the server (e.g. **`globals.toSnapshot()`**) and should include **`loadedContexts`** / **`pageContexts`** from **`ViewModelsStore`** so client VMs restore **`ctx`** without duplicate SSR fetches.

---

## PageVM template

```typescript
interface MyPageContext {
  data: SomeDC;
}

export class MyPageVM extends PageVM<MyPageContext | null> {
  didCreate(): void {
    makeObservable(this, {
      /* … */
    });

    this.onInit(async (ssr) => {
      if (ssr) {
        const data = await ssr.getSomeData();
        ssr.head.title = 'Page Title';
        this.ctx = { data };
        return;
      }
      if (!this.ctx) {
        const data = await loadSomeData();
        this.ctx = { data };
      }
    });
  }
}
```

---

## Anti-patterns

- Refetching the same page data on hydration when **`ctx`** already came from the snapshot.
- Using **`window`**, **`localStorage`**, or other browser-only APIs in the **`ssr`** branch of **`onInit`**.
- Shared **singleton** mutable state on the server — use a **per-request** `Globals` (or equivalent) so sessions do not leak.
- Confusing this **`PageVM`** with **`RouteViewModel` / `PageViewModel`** from **`mobx-route`** (different pattern; see **`routing-navigation`** + **`mvvm-architecture`** for route-bound VMs).
