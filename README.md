# agent-skills

Набор **Agent Skills** — текстовых инструкций для ИИ-агентов (MVVM, MobX, SSR, роутинг, UI, контракты API). Каждый навык — папка с `SKILL.md` и YAML frontmatter (`name`, `description`, …).

Подробная памятка для агентов и авторов навыков: **[AGENTS.md](AGENTS.md)**.

## Быстрый старт

### 1. Использовать как есть в своём проекте

- Склонируйте репозиторий или добавьте его **git submodule** в корень целевого проекта.
- В промпте агента укажите: прочитать нужный файл `skills/<имя>/SKILL.md` перед изменением кода.
- Либо скопируйте только нужные папки из `skills/` в свой репозиторий (например `docs/agent-skills/` или рядом с кодом).

Структура:

```text
skills/
  mvvm-architecture/SKILL.md
  mobx-general/SKILL.md
  mobx-stores/SKILL.md
  mobx-web-api/SKILL.md
  api-data-contracts/SKILL.md
  ssr-hydration/SKILL.md
  routing-navigation/SKILL.md
  ui-components/SKILL.md
```

### 2. Через `npx skills` (CLI экосистемы Agent Skills)

Да, так можно. Навыки лежат в каталоге `skills/` с полноценным `SKILL.md` — это как раз формат, который подхватывает официальный CLI [**vercel-labs/skills**](https://github.com/vercel-labs/skills): он находит навыки в `skills/` и ставит их в каталоги выбранных агентов (symlink или копия).

Из каталога с клоном этого репозитория:

```bash
# посмотреть, какие навыки обнаружены (без установки)
npx skills add . --list

# установить все навыки в текущий проект (интерактивно выберите агентов)
npx skills add .

# только Cursor и Codex
npx skills add . -a cursor -a codex -y

# глобально для пользователя
npx skills add . -g -y
```

После публикации репозитория на GitHub:

```bash
npx skills add OWNER/agent-skills --list
npx skills add OWNER/agent-skills --skill mvvm-architecture --skill mobx-stores -a cursor -y
```

Один навык по прямой ссылке на подпапку (пример URL — замените на свой):

```bash
npx skills add https://github.com/OWNER/agent-skills/tree/main/skills/ui-components
```

Полный список команд и путей установки по агентам см. в README проекта [skills](https://github.com/vercel-labs/skills).

### 3. Cursor

- Установка через **`npx skills add`** (см. выше) кладёт навыки туда, где их ждёт Cursor согласно CLI (например `.agents/skills/` в проекте или `~/.cursor/skills/` глобально — см. таблицу агентов в документации CLI).
- Либо откройте проект, где лежит каталог `skills/`, и явно попросите агента следовать соответствующему `SKILL.md`.
- Чтобы правила подтягивались постоянно, добавьте в **Rules** (`.cursor/rules/`) короткое правило со ссылкой на нужные навыки или скопируйте ключевые пункты из `SKILL.md` в `.mdc`-правило с нужным `globs`.

Точный каталог «skills» в Cursor зависит от версии и настроек; универсально работает хранение навыков **внутри репозитория** и явное чтение файлов по пути.

### 4. Codex (OpenAI Codex CLI)

Проще всего использовать тот же **`npx skills add … -a codex`** — он разложит файлы в ожидаемые пути для Codex.

Альтернатива: установщик навыков из документации Codex с указанием репозитория и пути к навыку (`skills/mvvm-architecture`, …). После любой установки при необходимости перезапустите Codex.

### 5. Ручное копирование

Скопируйте одну или несколько папок из `skills/<имя>/` в каталог skills вашего окружения так, чтобы у каждого навыка остался файл `SKILL.md` в корне папки навыка.

## Навыки (кратко)

| Навык | Содержание |
|--------|------------|
| [mvvm-architecture](skills/mvvm-architecture/SKILL.md) | MVVM, `withViewModel`, VM/PageVM, lifecycle |
| [mobx-general](skills/mobx-general/SKILL.md) | MobX: `enforceActions`, без лишних `action` |
| [mobx-stores](skills/mobx-stores/SKILL.md) | Сторы, коллекции, async, `Globals` |
| [mobx-web-api](skills/mobx-web-api/SKILL.md) | Реактивные Web API (storage и др.) |
| [api-data-contracts](skills/api-data-contracts/SKILL.md) | DC-типы, `fetch`, SSR API |
| [ssr-hydration](skills/ssr-hydration/SKILL.md) | SSR, гидратация, PageVM |
| [routing-navigation](skills/routing-navigation/SKILL.md) | `mobx-route`, query, 404 |
| [ui-components](skills/ui-components/SKILL.md) | UI-компоненты, стили, `ClientOnly` |

В нескольких навыках есть ссылки на полную документацию библиотек (`llms-full.txt`) — её имеет смысл открывать после базового `SKILL.md`.

## Лицензия

В frontmatter навыков указано `license: MIT` — при распространении сохраняйте условия MIT для производных работ, если копируете или переупаковываете материалы.

## Участие

При изменении навыков следуйте рекомендациям в [AGENTS.md](AGENTS.md): единый стиль, согласованность между файлами и актуальные примеры кода.
