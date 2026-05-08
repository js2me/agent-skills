---
name: mobx-web-api
description: Reactive Web API with mobx-web-api — storage data persistence (localStorage/sessionStorage), URL params, cookies, and other browser APIs as observables
license: MIT
compatibility: opencode
---

# MobX Reactive Web API

When you need MobX-reactive bindings for browser Web APIs (storage, URL, cookies, etc.), use `mobx-web-api`.

Full documentation: https://js2me.github.io/mobx-web-api/llms-full.txt

## Client-Only Persistence

`storageData.key` provides a reactive wrapper around `localStorage` or `sessionStorage`:

```typescript
import { storageData } from 'mobx-web-api';

class SettingsStore {
  private readonly theme = storageData.key<string>(
    'theme',     // storage key
    'light',     // default value
    'local',     // 'local' | 'session'
  );

  constructor() {
    // Initial value is read from storage
    console.log(this.theme.value);
  }

  setTheme(value: string) {
    this.theme.value = value; // auto-syncs to storage
  }
}
```

The stored value is observable — any component or computed reading it will react to changes.

## When to Use

- Persisting user preferences (theme, language, filters)
- Session-scoped state (favorite IDs, draft data)
- Any browser storage that needs to be reactive in MobX

For full API reference (cookies, URL, etc.), consult the documentation above.
