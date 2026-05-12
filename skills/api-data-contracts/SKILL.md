---
name: api-data-contracts
description: API data contract conventions — DC suffix types, fetch-based client functions, SSR API interface, endpoint addition workflow, data mapping to UI types
license: MIT
compatibility: opencode
---

# API & Data Contracts

> **This skill applies only if the project does NOT use `mobx-tanstack-query` or `mobx-tanstack-query-api`.**  
> If those libraries are present, API calls and caching are handled by them — follow their documentation:
> - mobx-tanstack-query: https://js2me.github.io/mobx-tanstack-query/llms-full.txt
> - mobx-tanstack-query-api: https://js2me.github.io/mobx-tanstack-query-api/llms-full.txt

## Data Contract (DC) Convention

All API response types are suffixed with `DC` to clearly distinguish API data shapes from UI/domain types:

```typescript
export interface ProductDC {
  id: number;
  title: string;
  price: number;
  rating: number;
  images?: string[];
}

export interface CartItemDC {
  productId: number;
  title: string;
  price: CartItemPriceDC;
  quantity: CartItemQuantityDC;
}
```

DC types are pure data shapes from the server. Extend/enrich them in stores or VMs.

## Client-Side API Functions

Place all API functions in a dedicated file. Use pure `fetch`:

```typescript
export const loadItems = async (
  params: LoadParams,
): Promise<ItemsChunkDC> => {
  const searchParams = new URLSearchParams({ ... });
  const response = await fetch(`/api/items?${searchParams}`);
  if (!response.ok) {
    throw new Error(`loadItems failed: ${response.status}`);
  }
  return response.json();
};

export const putItem = async (
  payload: SyncPayload,
): Promise<ItemDC> => {
  const response = await fetch('/api/items', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
    credentials: 'include',
  });
  if (!response.ok) {
    throw new Error(`putItem failed: ${response.status}`);
  }
  return response.json();
};
```

Patterns:
- GET params via `URLSearchParams`
- Mutations via PUT with `credentials: 'include'`
- Typed request/response with DC interfaces
- All functions are async, throw on non-ok response

## Server-Side SSR API (when using SSR)

How **`PageVM`**, **`onInit(ssr => …)`**, snapshot, and **`ViewModelsStore`** fit together is described in **[`mobx-ssr-hydration`](../mobx-ssr-hydration/SKILL.md)**. This section only covers **contract shapes**.

In-process data access (no HTTP) via an SSR API interface:

```typescript
interface SSRApi {
  head: HeadApi;
  getItemById(id: number): Promise<ItemDC | null>;
}

interface HeadApi {
  title: string;
  ogTitle?: string;
  ogDescription?: string;
  ogImage?: string;
  ogUrl?: string;
  ogType?: string;
}
```

SSR API calls the data layer directly, optionally with caching.

## Adding a New API Endpoint

1. **Data Contract**: Add `XxxDC` interface
2. **Client function**: Add `loadXxx` / `putXxx` using fetch
3. **Server endpoint**: Add handler on the server
4. **SSR API** (if needed): Add method to SSR interface, create fetch function with caching
5. **Store/VM**: Client fetch in VMs; SSR methods inside **`onInit(ssr => …)`** when using the stack from **`mobx-ssr-hydration`**

## Data Mapping

Convert DC types to UI-friendly types in stores or VMs:

```typescript
get items(): ItemInfo[] {
  return rawItems.map((item) => ({
    ...item,
    href: this.router.routes.item.createUrl({ itemId: item.id }),
    isSelected: this.selectedIds.has(item.id),
  }));
}
```

Helper functions keep mapping logic reusable:
```typescript
export const mapItemToViewModel = (item: ItemDC): ItemVM => ({ ... });
```
