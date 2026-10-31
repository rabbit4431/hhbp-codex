# hhbp-codex

[English](README.md) · **Français** · [Deutsch](README.de.md)

Heshuo Harness Best Practices — plugin [OpenAI Codex](https://developers.openai.com/codex)
fournissant des skills conformes au DDD, des hooks de sécurité et une configuration MCP
pour le développement de microservices Java Maven.

## Présentation

`hhbp-codex` est le jumeau compatible Codex de
[`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code). Tous deux sont de fines
enveloppes autour du contenu partagé **`hhbp-common`** (intégré ici dans le répertoire `common/`
via git subtree), qui l'exposent à un environnement hôte dans le format de plugin de cet environnement.

Cette enveloppe suit la
[spécification des plugins Codex](https://developers.openai.com/codex/plugins/build). Elle expose :

- **Skills** — les 22 workflows partagés de `common/skills/` (définitions `SKILL.md`), utilisés
  directement sans conversion :
  - *Workflow de fonctionnalité multi-services* — `springboot-feature-bootstrap` orchestre la
    chaîne en 9 phases : `service-discovery`, `feature-workspace-setup`, `contract-first-design`,
    `parallel-implementation`, `multi-service-integration-test`, `cross-service-review`,
    `multi-branch-merge`, `feature-cleanup`
  - *Fondations* — `test-driven-development`, `writing-implementation-plan`,
    `subagent-task-execution`, `code-review-request`, `finishing-a-feature-branch`,
    `security-review`, `docs-lookup`, `language-setup`
  - *Génération de code & SQL* — `generate-api`, `generate-code`, `remove-unused-class`,
    `sql-analyze`, `slow-sql-optimize`
- **Hooks** — SessionStart (`language.js`, applique la langue choisie),
  PreToolUse (`block-dangerous.js`, bloque les commandes shell dangereuses) et
  PostToolUse (`git-workflow.js`, applique les conventions de commit git). Le manifeste pointe
  vers un fichier `plugins/hhbp-codex/hooks/hooks.json` propre à Codex plutôt que vers
  `common/hooks/hooks.json`, afin que les ajustements spécifiques à Codex (par exemple, Codex ne
  prend pas encore en charge les hooks asynchrones) n'aient jamais à être placés dans `common`. Les
  commandes des hooks appellent les scripts partagés de `common/scripts/hooks/` via l'alias
  `${CLAUDE_PLUGIN_ROOT}` que Codex fournit pour `${PLUGIN_ROOT}`.
- **MCP** — la configuration des serveurs dans `common/mcp/mcp.json` (par ex. `context7` pour la
  consultation de documentation à jour).

**Hors périmètre :** les agents et les commandes ne sont volontairement **pas** exposés par ce plugin.

## Structure des répertoires

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

Le dépôt est une **marketplace** Codex dont l'unique plugin se trouve dans
`plugins/hhbp-codex/`. Codex ne découvre pas un plugin dont le `source.path` est la racine de la
marketplace elle-même (`./`) ; le plugin doit donc se trouver dans un sous-répertoire, conformément
à la convention `openai-curated`. Les champs `skills`, `hooks` et `mcpServers` du manifeste pointent
directement vers les répertoires du subtree `./common/` — sans copies dupliquées propres à un
environnement.

## Installation

1. Ajoutez la marketplace et installez le plugin :

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Activez le serveur MCP dans votre `config.toml` Codex :

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. Choisissez une langue — voir [Langue](#langue). Les nouveaux hooks doivent être examinés et
   approuvés avec `/hooks` avant que Codex ne les exécute.

## Langue

Le plugin peut répondre, et rédiger les plans, revues, rapports, descriptions de commit et
commentaires de code, dans les langues suivantes :

| Code | Langue |
|---|---|
| `en` | English (par défaut) |
| `fr` | Français |
| `de` | Deutsch |

Codex ne propose pas d'options à l'installation ; le plugin enregistre donc le skill
`language-setup` comme skill d'accueil (`extensions.com.openai.onboardingSkill`) : la configuration
du plugin l'exécute et vous demande de choisir une langue. Vous pouvez le relancer à tout moment en
demandant « changer la langue de HHBP ». Il enregistre votre choix dans `~/.hhbp/config.json` :

```json
{ "language": "fr" }
```

La variable d'environnement `HHBP_LANGUAGE` (par exemple `HHBP_LANGUAGE=de codex`) a priorité sur
ce fichier. La nouvelle langue s'applique à partir de la session suivante.

Les identifiants de code, noms de fichiers, commandes, le préfixe de commit `<type>:` et les jetons
du protocole de workflow (`feature-state.json`, verdicts de `REVIEW.md`) restent en anglais pour que
les skills continuent de fonctionner. Les messages des commandes bloquées sont également traduits.
Les skills et spécifications sont rédigés en anglais ; le réglage de langue ne concerne que les sorties.

## Modèles

Les skills s'exécutent avec le modèle utilisé par la session Codex, et chaque sous-agent en hérite.
Le frontmatter `model:` de `common/skills` est propre à Claude Code ; Codex l'ignore. Choisissez le
modèle au démarrage de la session (`codex -m <model>` ou `model` dans `~/.codex/config.toml`).

Le workflow imbrique les sous-agents sur deux niveaux — les flux d'implémentation
(`subagent-task-execution`) lancent des sous-agents d'implémentation et de revue pour chaque tâche —
augmentez donc la profondeur de lancement par défaut de Codex, qui est de 1 :

```toml
[agents]
max_depth = 2
```

## Mise à jour de common

`plugins/hhbp-codex/common/` est intégré depuis
[hhbp-common](https://github.com/rabbit4431/hhbp-common) via git subtree. Pour récupérer le
dernier contenu partagé :

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## Remarques

- **Clé d'enveloppe MCP :** `common/mcp/mcp.json` utilise la clé d'enveloppe `mcpServers`
  (partagée avec Claude Code). Codex accepte aussi une table de serveurs nue ou l'enveloppe
  `mcp_servers`. Si une future version de Codex rejette la clé en camelCase, la correction sera
  appliquée dans `common` afin que les deux enveloppes continuent de fonctionner.
- **Matchers d'outils des hooks :** les matchers des hooks (`Bash`, `Write|Edit|MultiEdit`) portent
  les noms des outils de Claude Code. Vérifiez qu'ils se déclenchent avec les noms d'outils de Codex ;
  toute surcharge de matcher spécifique à Codex doit aller dans un fichier de hooks propre à Codex,
  pas dans `common`.
