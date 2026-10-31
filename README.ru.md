# hhbp-codex

[English](README.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · **Русский** · [日本語](README.ja.md) · [简体中文](README.zh-CN.md) · [繁體中文](README.zh-TW.md) · [한국어](README.ko.md)

Heshuo Harness Best Practices — плагин для [OpenAI Codex](https://developers.openai.com/codex)
с навыками (skills) в соответствии со стандартами DDD, хуками безопасности и конфигурацией MCP
для разработки микросервисов на Java и Maven.

## Обзор

`hhbp-codex` — совместимый с Codex «брат» плагина
[`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code). Оба являются тонкими
обёртками вокруг общего содержимого **`hhbp-common`** (встроенного здесь в каталог `common/`
с помощью git subtree) и предоставляют его среде-хосту в её формате плагинов.

Эта обёртка следует
[спецификации плагинов Codex](https://developers.openai.com/codex/plugins/build). Она предоставляет:

- **Навыки** — 22 общих рабочих процесса из `common/skills/` (определения `SKILL.md`),
  используемые напрямую без преобразования:
  - *Рабочий процесс для функциональности в нескольких сервисах* — `springboot-feature-bootstrap`
    управляет цепочкой из 9 фаз: `service-discovery`, `feature-workspace-setup`,
    `contract-first-design`, `parallel-implementation`, `multi-service-integration-test`,
    `cross-service-review`, `multi-branch-merge`, `feature-cleanup`
  - *Основа* — `test-driven-development`, `writing-implementation-plan`,
    `subagent-task-execution`, `code-review-request`, `finishing-a-feature-branch`,
    `security-review`, `docs-lookup`, `language-setup`
  - *Генерация кода и SQL* — `generate-api`, `generate-code`, `remove-unused-class`,
    `sql-analyze`, `slow-sql-optimize`
- **Хуки** — SessionStart (`language.js`, применяет выбранный язык),
  PreToolUse (`block-dangerous.js`, блокирует опасные команды оболочки) и
  PostToolUse (`git-workflow.js`, следит за соблюдением соглашений о коммитах git). Манифест
  указывает на собственный файл Codex `plugins/hhbp-codex/hooks/hooks.json`, а не на
  `common/hooks/hooks.json`, чтобы специфичные для Codex изменения (например, Codex пока не
  поддерживает асинхронные хуки) никогда не приходилось вносить в `common`. Команды хуков вызывают
  общие скрипты из `common/scripts/hooks/` через псевдоним `${CLAUDE_PLUGIN_ROOT}`, который Codex
  предоставляет для `${PLUGIN_ROOT}`.
- **MCP** — конфигурация серверов в `common/mcp/mcp.json` (например, `context7` для получения
  актуальной документации).

**Вне рамок:** агенты и команды этим плагином намеренно **не** предоставляются.

## Структура каталогов

```
hhbp-codex/                              # marketplace root (the dir you `add`)
├── .agents/
│   └── plugins/
│       └── marketplace.json            # Codex marketplace metadata
└── plugins/
    └── hhbp-codex/                      # the plugin itself
        ├── .codex-plugin/
        │   └── plugin.json             # Codex plugin manifest
        ├── hooks/
        │   └── hooks.json              # Codex-owned hook registrations
        └── common/                     # vendored via git subtree from hhbp-common
            ├── skills/                 # shared skill definitions
            ├── hooks/hooks.json        # shared hook registrations
            ├── mcp/mcp.json            # shared MCP server config
            └── scripts/                # hook implementations
```

Репозиторий является **маркетплейсом** Codex, единственный плагин которого находится в
`plugins/hhbp-codex/`. Codex не обнаруживает плагин, у которого `source.path` указывает на сам
корень маркетплейса (`./`), поэтому плагин должен находиться в подкаталоге — в соответствии с
соглашением `openai-curated`. Поля манифеста `skills`, `hooks` и `mcpServers` указывают прямо на
каталоги subtree `./common/` — без дублирующих копий для отдельных сред.

## Установка

1. Добавьте маркетплейс и установите плагин:

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Включите MCP-сервер в `config.toml` Codex:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. Выберите язык — см. раздел [Язык](#язык). Новые хуки нужно проверить и одобрить через
   `/hooks`, прежде чем Codex начнёт их выполнять.

## Язык

Плагин может отвечать и писать планы, ревью, отчёты, описания коммитов и комментарии в коде
на следующих языках:

| Код | Язык |
|---|---|
| `en` | English (по умолчанию) |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

В Codex нет запроса параметров при установке, поэтому плагин регистрирует навык `language-setup`
как навык первоначальной настройки (`extensions.com.openai.onboardingSkill`): настройка плагина
запускает его и просит выбрать язык. Его можно запустить снова в любой момент, попросив «сменить
язык HHBP». Навык сохраняет ваш выбор в `~/.hhbp/config.json`:

```json
{ "language": "ru" }
```

Переменная окружения `HHBP_LANGUAGE` (например, `HHBP_LANGUAGE=ru codex`) имеет приоритет над этим
файлом. Новый язык применяется со следующей сессии.

Идентификаторы в коде, имена файлов, команды, префикс коммита `<type>:` и токены протокола
рабочего процесса (`feature-state.json`, вердикты `REVIEW.md`) остаются на английском, чтобы
навыки продолжали работать. Сообщения о заблокированных командах тоже переводятся. Навыки и
спецификации написаны на английском; настройка языка влияет только на вывод.

## Модели

Навыки выполняются на модели, которую использует сессия Codex, и каждый субагент наследует её.
Frontmatter `model:` в `common/skills` поддерживается только в Claude Code; Codex его игнорирует.
Выберите модель при запуске сессии (`codex -m <model>` или `model` в `~/.codex/config.toml`).

Рабочий процесс вкладывает субагентов на два уровня — потоки реализации
(`subagent-task-execution`) запускают для каждой задачи субагентов-исполнителей и субагентов-
ревьюеров, — поэтому увеличьте глубину порождения Codex, которая по умолчанию равна 1:

```toml
[agents]
max_depth = 2
```

## Обновление common

`plugins/hhbp-codex/common/` встроен из
[hhbp-common](https://github.com/rabbit4431/hhbp-common) с помощью git subtree. Чтобы получить
последнюю версию общего содержимого:

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## Примечания

- **Ключ-обёртка MCP:** `common/mcp/mcp.json` использует ключ-обёртку `mcpServers` (общий с
  Claude Code). Codex также принимает «голую» таблицу серверов или обёртку `mcp_servers`. Если
  будущая версия Codex отклонит ключ в camelCase, исправление будет внесено в `common`, чтобы обе
  обёртки продолжали работать.
- **Matcher'ы инструментов в хуках:** matcher'ы хуков (`Bash`, `Write|Edit|MultiEdit`) названы по
  инструментам Claude Code. Убедитесь, что они срабатывают с именами инструментов Codex; любые
  переопределения matcher'ов для Codex должны находиться в собственном файле хуков Codex, а не в
  `common`.
