# hhbp-codex

Heshuo Harness Best Practices — [OpenAI Codex](https://developers.openai.com/codex)
plugin providing DDD-standard skills, safety hooks, and MCP configuration
for Java Maven microservices development.

## Overview

`hhbp-codex` is the Codex-compatible sibling of
[`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code). Both are thin
wrappers around the shared **`hhbp-common`** content (vendored here as the `common/`
directory via git subtree), exposing it to a host harness via that harness's plugin format.

This wrapper follows the
[Codex plugin spec](https://developers.openai.com/codex/plugins/build). It surfaces:

- **Skills** — the shared `common/skills/` workflows (`SKILL.md` definitions), used
  directly with no conversion.
- **Hooks** — PreToolUse (`block-dangerous.js`, blocks dangerous shell commands) and
  PostToolUse (`git-workflow.js`, enforces git commit conventions). The manifest points
  at a Codex-owned `plugins/hhbp-codex/hooks/hooks.json` rather than
  `common/hooks/hooks.json`, so Codex-specific adjustments (for example, Codex does not
  support async hooks yet) never have to go into `common`. The hook commands invoke the
  shared scripts under `common/scripts/hooks/` via the `${CLAUDE_PLUGIN_ROOT}` alias
  Codex provides for `${PLUGIN_ROOT}`.
- **MCP** — the server configuration in `common/mcp/mcp.json` (e.g. `context7` for live
  documentation lookup).

**Out of scope:** agents and commands are intentionally **not** exposed by this plugin.

## Directory structure

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

The repo is a Codex **marketplace** whose single plugin lives under
`plugins/hhbp-codex/`. Codex does not discover a plugin whose `source.path` is the
marketplace root itself (`./`), so the plugin must sit in a subdirectory — matching the
`openai-curated` convention. The manifest's `skills`, `hooks`, and `mcpServers` fields
point straight at the `./common/` subtree directories — no duplicated, harness-specific
copies.

## Installation

1. Add the marketplace and install the plugin:

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Enable the MCP server in your Codex `config.toml`:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

## Model tiers

Skills declare a model tier in `SKILL.md` frontmatter — `metadata.hhbp-phase` (`design` / `impl`)
and `metadata.codex-model`:

| Tier | Codex model | Skills |
|---|---|---|
| design | `gpt6-astra` | service-discovery, contract-first-design, writing-implementation-plan, cross-service-review, code-review-request, security-review |
| impl | `gpt6.1-sol` | all other workflow, codegen, and test skills (`sql-analyze` and `slow-sql-optimize` keep the session model) |

Codex has no per-skill model setting and cannot change a session's model mid-session, so the
switch works like this:

- **Inside the `springboot-feature-bootstrap` workflow — automatic.** Run the session on the impl
  model. At each phase transition, the dispatcher runs a design phase (discovery, contracts,
  planning, review) in a subagent spawned on `gpt6-astra`, then resumes in the main thread.
  Approval gates still pause in the main thread. See "Model routing" in
  `common/skills/springboot-feature-bootstrap/references/phase-dispatcher.md`.
- **A skill used on its own — pick a profile.** `codex --profile hhbp-design` or
  `codex --profile hhbp-impl`.

Add to `~/.codex/config.toml`:

```toml
# workflow sessions (orchestrator + impl phases) run on the impl model
model = "gpt6.1-sol"

[agents]
default_subagent_model = "gpt6.1-sol"
max_depth = 2   # a phase subagent dispatches its own reviewer/task subagents

[profiles.hhbp-design]
model = "gpt6-astra"

[profiles.hhbp-impl]
model = "gpt6.1-sol"
```

The automatic switch relies on Codex honoring the model the dispatcher requests at spawn time —
it is an instruction to the agent, not a config-enforced setting. Open a subagent thread (`o`) to
confirm which model it runs on.

## Updating common

`plugins/hhbp-codex/common/` is vendored from
[hhbp-common](https://github.com/rabbit4431/hhbp-common) via git subtree. To pull the
latest shared content:

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## Notes

- **MCP wrapper key:** `common/mcp/mcp.json` uses the `mcpServers` wrapper key (shared
  with Claude Code). Codex also accepts a bare server map or the `mcp_servers` wrapper.
  If a future Codex version rejects the camelCase key, the fix is applied inside
  `common` so both wrappers keep working.
- **Hook tool matchers:** hook matchers (`Bash`, `Write|Edit|MultiEdit`) are named for
  Claude Code tools. Confirm they fire under Codex's tool names; any Codex-specific
  matcher overrides belong in a codex-owned hooks file, not in `common`.
