# hhbp-codex

[English](README.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Русский](README.ru.md) · [日本語](README.ja.md) · **简体中文** · [繁體中文](README.zh-TW.md) · [한국어](README.ko.md)

Heshuo Harness Best Practices —— 面向 Java Maven 微服务开发的 [OpenAI Codex](https://developers.openai.com/codex)
插件，提供符合 DDD 规范的技能（skills）、安全钩子（hooks）和 MCP 配置。

## 概述

`hhbp-codex` 是 [`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code) 的
Codex 兼容姊妹插件。两者都是围绕共享内容 **`hhbp-common`**（此处通过 git subtree 引入为 `common/`
目录）的轻量封装，以宿主环境的插件格式将其提供给该环境。

本封装遵循 [Codex 插件规范](https://developers.openai.com/codex/plugins/build)，提供：

- **技能** —— `common/skills/` 中的 22 个共享工作流（`SKILL.md` 定义），无需转换直接使用：
  - *多服务功能工作流* —— `springboot-feature-bootstrap` 编排 9 个阶段的链路：
    `service-discovery`、`feature-workspace-setup`、`contract-first-design`、
    `parallel-implementation`、`multi-service-integration-test`、`cross-service-review`、
    `multi-branch-merge`、`feature-cleanup`
  - *基础* —— `test-driven-development`、`writing-implementation-plan`、
    `subagent-task-execution`、`code-review-request`、`finishing-a-feature-branch`、
    `security-review`、`docs-lookup`、`language-setup`
  - *代码生成与 SQL* —— `generate-api`、`generate-code`、`remove-unused-class`、`sql-analyze`、
    `slow-sql-optimize`
- **钩子** —— SessionStart（`language.js`，应用你选择的语言）、
  PreToolUse（`block-dangerous.js`，拦截危险的 shell 命令）和
  PostToolUse（`git-workflow.js`，强制执行 git 提交规范）。清单指向 Codex 自有的
  `plugins/hhbp-codex/hooks/hooks.json`，而不是 `common/hooks/hooks.json`，因此 Codex 特有的调整
  （例如 Codex 尚不支持异步钩子）无需放进 `common`。钩子命令通过 Codex 为 `${PLUGIN_ROOT}` 提供的
  别名 `${CLAUDE_PLUGIN_ROOT}` 调用 `common/scripts/hooks/` 下的共享脚本。
- **MCP** —— `common/mcp/mcp.json` 中的服务器配置（例如用于查询最新文档的 `context7`）。

**不在范围内：** 本插件有意**不**提供代理（agents）和命令（commands）。

## 目录结构

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

本仓库是一个 Codex **插件市场（marketplace）**，其中唯一的插件位于 `plugins/hhbp-codex/`。
Codex 不会发现 `source.path` 指向市场根目录本身（`./`）的插件，因此插件必须放在子目录中——
与 `openai-curated` 的约定一致。清单中的 `skills`、`hooks` 和 `mcpServers` 字段直接指向
`./common/` subtree 中的目录——没有针对不同环境的重复副本。

## 安装

1. 添加插件市场并安装插件：

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. 在 Codex 的 `config.toml` 中启用 MCP 服务器：

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. 选择语言——参见[语言](#语言)。新的钩子需要先通过 `/hooks` 审核并信任，Codex 才会运行它们。

## 语言

插件可以用以下语言回复，并撰写计划、评审、报告、提交信息描述和代码注释：

| 代码 | 语言 |
|---|---|
| `en` | English（默认） |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

Codex 在安装时没有选项提示，因此插件将 `language-setup` 技能注册为引导技能
（`extensions.com.openai.onboardingSkill`）：插件设置时会运行它并请你选择语言。之后随时说
“更改 HHBP 语言” 即可再次运行。它会把你的选择保存到 `~/.hhbp/config.json`：

```json
{ "language": "zh-CN" }
```

环境变量 `HHBP_LANGUAGE`（例如 `HHBP_LANGUAGE=zh-CN codex`）优先于该文件。新语言从下一个会话
开始生效。

代码标识符、文件名、命令、提交前缀 `<type>:` 以及工作流协议中的标记（`feature-state.json`、
`REVIEW.md` 的结论）保持英文，以确保各技能正常运行。被拦截命令的提示信息也会被翻译。技能和
规范文档均以英文编写；语言设置只影响输出。

## 模型

技能使用 Codex 会话所用的模型运行，所有子代理也继承该模型。`common/skills` 中的 `model:`
frontmatter 仅适用于 Claude Code，Codex 会忽略它。请在启动会话时选择模型（`codex -m <model>`
或 `~/.codex/config.toml` 中的 `model`）。

该工作流会将子代理嵌套两层——实现流（`subagent-task-execution`）会为每个任务派发实现子代理和
评审子代理——因此请把 Codex 默认的派生深度 1 调高：

```toml
[agents]
max_depth = 2
```

## 更新 common

`plugins/hhbp-codex/common/` 通过 git subtree 从
[hhbp-common](https://github.com/rabbit4431/hhbp-common) 引入。拉取最新的共享内容：

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## 注意事项

- **MCP 包装键：** `common/mcp/mcp.json` 使用 `mcpServers` 包装键（与 Claude Code 共用）。Codex
  也接受不带包装的服务器映射或 `mcp_servers` 包装键。如果未来的 Codex 版本拒绝 camelCase 键，
  修复会在 `common` 中进行，以确保两个封装都能继续工作。
- **钩子的工具 matcher：** 钩子 matcher（`Bash`、`Write|Edit|MultiEdit`）以 Claude Code 的工具名
  命名。请确认它们在 Codex 的工具名下也能触发；任何 Codex 特有的 matcher 覆盖都应放在 Codex 自有的
  钩子文件中，而不是 `common` 中。
