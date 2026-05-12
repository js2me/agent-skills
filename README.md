# agent-skills

Здесь лежат **навыки для ИИ-агентов** — папки в `skills/`, в каждой файл `SKILL.md` с правилами и примерами кода (MVVM, MobX, SSR, роутинг, UI, API).

Ниже — два простых способа пользоваться репозиторием.

---

## Способ 1: положить рядом с проектом и читать файлы

1. Склонируйте репозиторий или скопируйте каталог `skills/` к себе в проект.
2. В чате с агентом попросите открыть нужный файл, например  
   `skills/mvvm-architecture/SKILL.md`.

Так можно пользоваться без установки чего-либо глобально.

---

## Способ 2: установить через `npx skills`

Подходит, если хотите, чтобы Cursor, Codex и другие агенты подхватили навыки автоматически (CLI из [vercel-labs/skills](https://github.com/vercel-labs/skills)).

**С GitHub** ([js2me/agent-skills](https://github.com/js2me/agent-skills)) — выполните в терминале из любой папки (например из корня вашего проекта):

```bash
npx skills add js2me/agent-skills --list
npx skills add js2me/agent-skills -a cursor -a codex -y
```

Полная ссылка то же самое:

```bash
npx skills add https://github.com/js2me/agent-skills -a cursor -y
```

**Из локального клона** этого репозитория вместо `js2me/agent-skills` можно указать `.`:

```bash
npx skills add . --list
npx skills add . -a cursor -a codex -y
```

---

## Какие навыки есть

| Папка | О чём |
|-------|--------|
| [mvvm-architecture](skills/mvvm-architecture/SKILL.md) | MVVM, ViewModel, `withViewModel` |
| [mobx-ssr-hydration](skills/mobx-ssr-hydration/SKILL.md) | кастомный SSR и гидратация: PageVM, снапшот |
| [mobx-general](skills/mobx-general/SKILL.md) | базовые правила MobX |
| [mobx-stores](skills/mobx-stores/SKILL.md) | сторы, Globals |
| [mobx-web-api](skills/mobx-web-api/SKILL.md) | storage и др. в браузере |
| [api-data-contracts](skills/api-data-contracts/SKILL.md) | типы DC, fetch, API |
| [routing-navigation](skills/routing-navigation/SKILL.md) | роутинг |
| [ui-components](skills/ui-components/SKILL.md) | UI-компоненты и стили |

---

Если вы **редактируете** навыки в этом репозитории — загляните в [AGENTS.md](AGENTS.md).

Лицензия навыков в frontmatter: MIT.
