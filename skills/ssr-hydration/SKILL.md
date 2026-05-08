---
name: ssr-hydration
description: SSR and hydration patterns with PageVM — initOnServer/initOnClient flow, SSRApi interface, rendering guards, server caching, hydration entry
license: MIT
compatibility: opencode
---

# SSR & Hydration Strategy

## Architecture Overview

```
Server Request
  → Session management
  → SSR API with in-memory cache
  → Central container (per-request) with memory history
  → React renders → stream to HTML
  → `window.__SSR_DATA__` (snapshot)

Client Hydration
  → Restore container from snapshot
  → hydrateRoot(<App />)
  → VMs re-init with pre-loaded context (skip fetch)
```

## Data Flow Per Page (PageVM)

### 1. Server: initOnServer(ssr)
```typescript
didCreate(): void {
  this.onInit(async (ssr) => {
    if (ssr) {
      // SSR path: use SSR API (in-process data, no HTTP)
      const item = await ssr.getItemById(this.id);
      const related = await ssr.getRelatedById(item!.relatedId);

      // Set head meta
      ssr.head.title = `${item!.title}`;
      ssr.head.ogTitle = item!.title;

      // Set context for hydration
      this.ctx = { item, related };
      return;
    } else if (!this.ctx) {
      // Client-side navigation (no SSR context) — use HTTP fetch
      const item = await loadItemById(this.id);
      this.ctx = { item, related: await loadRelated(item.relatedId) };
    }
  });
}
```

### 2. Client: initOnClient()
- Called in `mount()` → `initOnClient()`
- If `this.ctx` already populated from SSR snapshot → skips fetch
- If no context → runs initFn with `ssr=null` → fetches via HTTP API
- Sets `isInitializing = true` during fetch

### 3. Rendering Guard
```typescript
// In withViewModel component
if (model.isInitializing && !model.item) {
  return <LoadingSkeleton />;  // or options.fallback
}
if (!model.item) {
  return <NotFound />;
}
return <FullPage />;
```

## SSR API Interface

```typescript
interface SSRApi {
  head: HeadApi; // title, ogTitle, ogDescription, ogImage, etc.
  getItemById(id: number): Promise<ItemDC | null>;
  getRelatedById(id: number): Promise<RelatedDC | null>;
  getSystemInfo(): { date: string };
}
```

Set head meta inside `onInit` when `ssr` is available:
```typescript
ssr.head.title = 'Page Title';
ssr.head.ogTitle = 'OG Title';
ssr.head.ogDescription = 'Description';
ssr.head.ogImage = '/path/to/image';
ssr.head.ogUrl = `/items/${id}`;
ssr.head.ogType = 'product';
```

## Server-Side Caching Pattern

SSR API typically has an in-memory cache keyed by session ID:

```typescript
const CACHE_TTL_MS = 3 * 24 * 60 * 60 * 1000;
// Map<sessionId, { value: SSRApi, expiresAt: number }>
```

Cache is created once per session and reused across all SSR requests for that session.

## Hydration Entry

```typescript
// Client entry
const container = Container.fromSnapshot(
  (globalThis as any).__SSR_DATA__,
);
hydrateRoot(
  document.getElementById('root')!,
  <App container={container} />,
);
```

Snapshot is created via `container.toSnapshot()` which serializes loaded contexts from ViewModelsStore.

## Anti-Patterns to Avoid

- Don't fetch the same page data both in SSR and immediately again on hydration; check `ctx` first.
- Don't access browser-only APIs (`window`, `localStorage`) in server init path.
- Don't mutate shared singleton state on server; keep per-request container isolation.

## PageVM Template

```typescript
interface MyPageContext {
  data: SomeDC;
}

class MyPageVM extends PageVM<MyPageContext | null> {
  didCreate(): void {
    makeObservable(this, {
      // ...
    });

    this.onInit(async (ssr) => {
      if (ssr) {
        const data = await ssr.getSomeData();
        ssr.head.title = `Page Title`;
        this.ctx = { data };
        return;
      } else if (!this.ctx) {
        const data = await loadSomeData();
        this.ctx = { data };
      }
    });
  }
}
```
