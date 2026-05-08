# AGENTS.md — работа с этим репозиторием

Этот репозиторий содержит **набор навыков (skills)** для ИИ-агентов: каждый навык — отдельная папка с файлом `SKILL.md` и YAML frontmatter (`name`, `description`, и т.д.).

## Как пользоваться навыками

1. **Перед задачей** определи тему (UI, роутинг, SSR, MobX, API).
2. **Открой соответствующий** `skills/<topic>/SKILL.md` и следуй описанным там паттернам и запретам.
3. Если темы пересекаются (например, страница + SSR + роутинг), прочитай **несколько** навыков в порядке: архитектура → данные → конкретная область.

## Оглавление навыков

| Путь | Когда читать |
|------|----------------|
| [`skills/mvvm-architecture/SKILL.md`](skills/mvvm-architecture/SKILL.md) | MVVM, `withViewModel`, иерархия VM/PageVM, lifecycle, структура файлов |
| [`skills/mobx-general/SKILL.md`](skills/mobx-general/SKILL.md) | Базовые правила MobX: `enforceActions: 'never'`, без лишних `action`/`runInAction` |
| [`skills/mobx-stores/SKILL.md`](skills/mobx-stores/SKILL.md) | Сторы: `makeObservable`, коллекции, async, debounced sync, `Globals` |
| [`skills/mobx-web-api/SKILL.md`](skills/mobx-web-api/SKILL.md) | Реактивные обёртки над Web API (storage, URL, cookies) |
| [`skills/api-data-contracts/SKILL.md`](skills/api-data-contracts/SKILL.md) | DC-типы, `fetch`, SSR API, добавление эндпоинтов, маппинг в UI |
| [`skills/ssr-hydration/SKILL.md`](skills/ssr-hydration/SKILL.md) | SSR/гидратация, PageVM, `SSRApi`, снапшот, антипаттерны |
| [`skills/routing-navigation/SKILL.md`](skills/routing-navigation/SKILL.md) | `mobx-route`, история, query params, навигация, 404 |
| [`skills/ui-components/SKILL.md`](skills/ui-components/SKILL.md) | Презентационные компоненты, `ClientOnly`, ActionButton, стили |

## Сквозные правила (кратко)

- **Бизнес-логика** — во ViewModel/Store, не в React-компонентах.
- **Хуки React** — только для чистого UI (refs, анимации, измерение DOM, паттерн `ClientOnly` и аналоги), без доменной логики.
- **MobX**: по умолчанию без обёрток `action`/`runInAction` при принятой конфигурации проекта (см. `mobx-general`).
- **API**: типы ответов с суффиксом `DC`; при неуспешном HTTP — явная обработка ошибок (см. примеры в `api-data-contracts`).

## Дополнительные документы внутри навыков

Часть навыков сошлась на полную документацию библиотек по URL (`llms-full.txt`). При необходимости уточняй детали API там — после прочтения базового `SKILL.md`.

## Изменение навыков

При правках сохраняй:

- единый стиль с существующими `SKILL.md`;
- согласованность между навыками (нет противоречий по хукам, SSR, MobX);
- актуальные примеры кода и явные антипаттерны там, где это уже принято в проекте.
