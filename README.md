# hhbp-codex

Heshuo Harness Best Practices — [OpenAI Codex](https://developers.openai.com/codex)
plugin providing DDD-standard skills, session lifecycle hooks, and MCP configuration
for Java Maven microservices development.

## Overview

`hhbp-codex` is the Codex-compatible sibling of
[`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code). Both are thin
wrappers around the shared **`hhbp-common`** content (vendored here as the `common/`
git submodule), exposing it to a host harness via that harness's plugin format.

This wrapper follows the
[Codex plugin spec](https://developers.openai.com/codex/plugins/build). It surfaces:

- **Skills** — the shared `common/skills/` workflows (`SKILL.md` definitions), used
  directly with no conversion.
- **Hooks** — the session lifecycle hooks (SessionStart, PreToolUse, PostToolUse,
  SessionEnd, Stop). The manifest points at a Codex-owned
  `plugins/hhbp-codex/hooks/hooks.json` rather than `common/hooks/hooks.json`: it is
  identical except the `async` flag is removed, because Codex does not support async
  hooks yet (it would otherwise skip the activity-tracker and session-end hooks). The
  hook commands still invoke the shared scripts under `common/scripts/hooks/` via the
  `${CLAUDE_PLUGIN_ROOT}` alias Codex provides for `${PLUGIN_ROOT}`.
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
        │   └── hooks.json              # Codex-owned hooks (async removed)
        └── common/                     # git submodule → hhbp-common
            ├── skills/                 # shared skill definitions
            ├── hooks/hooks.json        # shared hook registrations
            ├── mcp/mcp.json            # shared MCP server config
            └── scripts/                # hook implementations + shared libs
```

The repo is a Codex **marketplace** whose single plugin lives under
`plugins/hhbp-codex/`. Codex does not discover a plugin whose `source.path` is the
marketplace root itself (`./`), so the plugin must sit in a subdirectory — matching the
`openai-curated` convention. The manifest's `skills`, `hooks`, and `mcpServers` fields
point straight at the `./common/` submodule directories — no duplicated, harness-specific
copies.

## Installation

1. Clone with the submodule:

   ```bash
   git clone --recurse-submodules <repo-url> hhbp-codex
   # or, after a plain clone:
   git submodule update --init
   ```

2. Add the marketplace and install the plugin:

   ```bash
   codex plugin marketplace add ./hhbp-codex      # path to this repo
   codex plugin add hhbp-codex@hhbp-codex
   ```

3. Enable the MCP server in your Codex `config.toml`:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

## Notes

- **MCP wrapper key:** `common/mcp/mcp.json` uses the `mcpServers` wrapper key (shared
  with Claude Code). Codex also accepts a bare server map or the `mcp_servers` wrapper.
  If a future Codex version rejects the camelCase key, the fix is applied inside
  `common` so both wrappers keep working.
- **Hook tool matchers:** hook matchers (`Bash`, `Write|Edit|MultiEdit`) are named for
  Claude Code tools. Confirm they fire under Codex's tool names; any Codex-specific
  matcher overrides belong in a codex-owned hooks file, not in `common`.
