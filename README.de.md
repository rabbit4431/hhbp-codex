# hhbp-codex

[English](README.md) · [Français](README.fr.md) · **Deutsch** · [Русский](README.ru.md) · [日本語](README.ja.md) · [简体中文](README.zh-CN.md) · [繁體中文](README.zh-TW.md) · [한국어](README.ko.md)

Heshuo Harness Best Practices — [OpenAI Codex](https://developers.openai.com/codex)-Plugin
mit DDD-konformen Skills, Sicherheits-Hooks und MCP-Konfiguration
für die Entwicklung von Java-Maven-Microservices.

## Überblick

`hhbp-codex` ist das Codex-kompatible Schwesterplugin von
[`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code). Beide sind schlanke
Wrapper um die gemeinsamen Inhalte von **`hhbp-common`** (hier als Verzeichnis `common/` per
git subtree eingebunden) und stellen sie einer Host-Umgebung in deren Plugin-Format bereit.

Dieser Wrapper folgt der
[Codex-Plugin-Spezifikation](https://developers.openai.com/codex/plugins/build). Er stellt bereit:

- **Skills** — die 22 gemeinsamen Workflows aus `common/skills/` (`SKILL.md`-Definitionen),
  direkt und ohne Konvertierung verwendet:
  - *Multi-Service-Feature-Workflow* — `springboot-feature-bootstrap` orchestriert die
    9-phasige Kette: `service-discovery`, `feature-workspace-setup`, `contract-first-design`,
    `parallel-implementation`, `multi-service-integration-test`, `cross-service-review`,
    `multi-branch-merge`, `feature-cleanup`
  - *Grundlagen* — `test-driven-development`, `writing-implementation-plan`,
    `subagent-task-execution`, `code-review-request`, `finishing-a-feature-branch`,
    `security-review`, `docs-lookup`, `language-setup`
  - *Codegenerierung & SQL* — `generate-api`, `generate-code`, `remove-unused-class`,
    `sql-analyze`, `slow-sql-optimize`
- **Hooks** — SessionStart (`language.js`, wendet die gewählte Sprache an),
  PreToolUse (`block-dangerous.js`, blockiert gefährliche Shell-Befehle) und
  PostToolUse (`git-workflow.js`, setzt Git-Commit-Konventionen durch). Das Manifest verweist
  auf eine Codex-eigene `plugins/hhbp-codex/hooks/hooks.json` statt auf
  `common/hooks/hooks.json`, damit Codex-spezifische Anpassungen (Codex unterstützt zum Beispiel
  noch keine asynchronen Hooks) nie in `common` landen müssen. Die Hook-Befehle rufen die
  gemeinsamen Skripte unter `common/scripts/hooks/` über den Alias `${CLAUDE_PLUGIN_ROOT}` auf,
  den Codex für `${PLUGIN_ROOT}` bereitstellt.
- **MCP** — die Serverkonfiguration in `common/mcp/mcp.json` (z. B. `context7` für aktuelle
  Dokumentation).

**Nicht im Umfang:** Agents und Commands werden von diesem Plugin bewusst **nicht** bereitgestellt.

## Verzeichnisstruktur

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

Das Repository ist ein Codex-**Marketplace**, dessen einziges Plugin unter
`plugins/hhbp-codex/` liegt. Codex erkennt kein Plugin, dessen `source.path` das
Marketplace-Stammverzeichnis selbst (`./`) ist; das Plugin muss daher in einem Unterverzeichnis
liegen — entsprechend der `openai-curated`-Konvention. Die Manifestfelder `skills`, `hooks` und
`mcpServers` verweisen direkt auf die Verzeichnisse des Subtrees `./common/` — ohne doppelte,
umgebungsspezifische Kopien.

## Installation

1. Marketplace hinzufügen und Plugin installieren:

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Den MCP-Server in Ihrer Codex-`config.toml` aktivieren:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. Eine Sprache wählen — siehe [Sprache](#sprache). Neue Hooks müssen mit `/hooks` geprüft und
   freigegeben werden, bevor Codex sie ausführt.

## Sprache

Das Plugin kann in folgenden Sprachen antworten und darin Pläne, Reviews, Berichte,
Commit-Beschreibungen und Codekommentare verfassen:

| Code | Sprache |
|---|---|
| `en` | English (Standard) |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

Codex bietet bei der Installation keine Optionsabfrage; daher registriert das Plugin den Skill
`language-setup` als Onboarding-Skill (`extensions.com.openai.onboardingSkill`): Die
Plugin-Einrichtung führt ihn aus und fragt nach Ihrer Sprache. Sie können ihn jederzeit erneut
ausführen, indem Sie „HHBP-Sprache ändern“ anfragen. Er speichert Ihre Wahl in `~/.hhbp/config.json`:

```json
{ "language": "de" }
```

Die Umgebungsvariable `HHBP_LANGUAGE` (zum Beispiel `HHBP_LANGUAGE=fr codex`) hat Vorrang vor
dieser Datei. Die neue Sprache gilt ab der nächsten Sitzung.

Code-Bezeichner, Dateinamen, Befehle, das Commit-Präfix `<type>:` und die Tokens des
Workflow-Protokolls (`feature-state.json`, `REVIEW.md`-Urteile) bleiben auf Englisch, damit die
Skills weiterhin funktionieren. Meldungen zu blockierten Befehlen werden ebenfalls übersetzt.
Skills und Spezifikationen sind auf Englisch verfasst; die Spracheinstellung betrifft nur die Ausgabe.

## Modelle

Skills laufen mit dem Modell der Codex-Sitzung, und jeder Subagent erbt es. Das `model:`-Frontmatter
in `common/skills` gilt nur für Claude Code; Codex ignoriert es. Wählen Sie das Modell beim Start der
Sitzung (`codex -m <model>` oder `model` in `~/.codex/config.toml`).

Der Workflow verschachtelt Subagents zwei Ebenen tief — Implementierungsstränge
(`subagent-task-execution`) starten pro Aufgabe Implementierungs- und Review-Subagents — erhöhen Sie
daher die standardmäßige Spawn-Tiefe von Codex (1):

```toml
[agents]
max_depth = 2
```

## common aktualisieren

`plugins/hhbp-codex/common/` wird aus
[hhbp-common](https://github.com/rabbit4431/hhbp-common) per git subtree eingebunden. So holen Sie
die neuesten gemeinsamen Inhalte:

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## Hinweise

- **MCP-Wrapper-Schlüssel:** `common/mcp/mcp.json` verwendet den Wrapper-Schlüssel `mcpServers`
  (gemeinsam mit Claude Code). Codex akzeptiert auch eine reine Servertabelle oder den Wrapper
  `mcp_servers`. Falls eine künftige Codex-Version den camelCase-Schlüssel ablehnt, wird die
  Korrektur in `common` vorgenommen, damit beide Wrapper weiter funktionieren.
- **Tool-Matcher der Hooks:** Die Hook-Matcher (`Bash`, `Write|Edit|MultiEdit`) sind nach den
  Tools von Claude Code benannt. Prüfen Sie, ob sie unter den Tool-Namen von Codex auslösen;
  Codex-spezifische Matcher-Überschreibungen gehören in eine Codex-eigene Hooks-Datei, nicht in
  `common`.
