# hhbp-codex

[English](README.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Русский](README.ru.md) · [日本語](README.ja.md) · [简体中文](README.zh-CN.md) · **繁體中文** · [한국어](README.ko.md)

Heshuo Harness Best Practices —— 為 Java Maven 微服務開發打造的 [OpenAI Codex](https://developers.openai.com/codex)
外掛，提供符合 DDD 規範的技能（skills）、安全掛鉤（hooks）與 MCP 設定。

## 概覽

`hhbp-codex` 是 [`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code) 的
Codex 相容姊妹外掛。兩者都是包覆共用內容 **`hhbp-common`**（此處透過 git subtree 引入為
`common/` 目錄）的輕量封裝，以宿主環境的外掛格式將其提供給該環境。

本封裝遵循 [Codex 外掛規格](https://developers.openai.com/codex/plugins/build)，提供：

- **技能** —— `common/skills/` 中的 22 個共用工作流程（`SKILL.md` 定義），無需轉換直接使用：
  - *多服務功能工作流程* —— `springboot-feature-bootstrap` 協調 9 個階段的流程鏈：
    `service-discovery`、`feature-workspace-setup`、`contract-first-design`、
    `parallel-implementation`、`multi-service-integration-test`、`cross-service-review`、
    `multi-branch-merge`、`feature-cleanup`
  - *基礎* —— `test-driven-development`、`writing-implementation-plan`、
    `subagent-task-execution`、`code-review-request`、`finishing-a-feature-branch`、
    `security-review`、`docs-lookup`、`language-setup`
  - *程式碼產生與 SQL* —— `generate-api`、`generate-code`、`remove-unused-class`、
    `sql-analyze`、`slow-sql-optimize`
- **掛鉤** —— SessionStart（`language.js`，套用你選擇的語言）、
  PreToolUse（`block-dangerous.js`，封鎖危險的 shell 指令）與
  PostToolUse（`git-workflow.js`，強制執行 git 提交規範）。資訊清單指向 Codex 自有的
  `plugins/hhbp-codex/hooks/hooks.json`，而非 `common/hooks/hooks.json`，因此 Codex 特有的調整
  （例如 Codex 尚不支援非同步掛鉤）不必放進 `common`。掛鉤指令透過 Codex 為 `${PLUGIN_ROOT}`
  提供的別名 `${CLAUDE_PLUGIN_ROOT}` 呼叫 `common/scripts/hooks/` 下的共用指令碼。
- **MCP** —— `common/mcp/mcp.json` 中的伺服器設定（例如用於查詢最新文件的 `context7`）。

**不在範圍內：** 本外掛刻意**不**提供代理（agents）與指令（commands）。

## 目錄結構

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

本儲存庫是一個 Codex **外掛市集（marketplace）**，其中唯一的外掛位於 `plugins/hhbp-codex/`。
Codex 不會偵測 `source.path` 指向市集根目錄本身（`./`）的外掛，因此外掛必須放在子目錄中——
與 `openai-curated` 的慣例一致。資訊清單中的 `skills`、`hooks` 與 `mcpServers` 欄位直接指向
`./common/` subtree 中的目錄——沒有針對不同環境的重複副本。

## 安裝

1. 新增外掛市集並安裝外掛：

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. 在 Codex 的 `config.toml` 中啟用 MCP 伺服器：

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. 選擇語言——請參閱[語言](#語言)。新的掛鉤需要先透過 `/hooks` 審查並信任，Codex 才會執行。

## 語言

外掛可以用以下語言回覆，並撰寫計畫、審查、報告、提交訊息描述與程式碼註解：

| 代碼 | 語言 |
|---|---|
| `en` | English（預設） |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

Codex 在安裝時沒有選項提示，因此外掛將 `language-setup` 技能註冊為引導技能
（`extensions.com.openai.onboardingSkill`）：外掛設定時會執行它並請你選擇語言。之後隨時說
「變更 HHBP 語言」即可再次執行。它會將你的選擇儲存到 `~/.hhbp/config.json`：

```json
{ "language": "zh-TW" }
```

環境變數 `HHBP_LANGUAGE`（例如 `HHBP_LANGUAGE=zh-TW codex`）優先於該檔案。新語言從下一個工作
階段開始生效。

程式碼識別字、檔案名稱、指令、提交前綴 `<type>:` 以及工作流程協定中的標記
（`feature-state.json`、`REVIEW.md` 的結論）維持英文，以確保各技能正常運作。被封鎖指令的提示
訊息也會翻譯。技能與規範文件皆以英文撰寫；語言設定只影響輸出。

## 模型

技能以 Codex 工作階段所用的模型執行，所有子代理也會沿用。`common/skills` 中的 `model:`
frontmatter 僅適用於 Claude Code，Codex 會忽略它。請在啟動工作階段時選擇模型
（`codex -m <model>` 或 `~/.codex/config.toml` 中的 `model`）。

此工作流程會將子代理巢狀兩層——實作流程（`subagent-task-execution`）會為每個任務派發實作子代理
與審查子代理——因此請把 Codex 預設的衍生深度 1 調高：

```toml
[agents]
max_depth = 2
```

## 更新 common

`plugins/hhbp-codex/common/` 透過 git subtree 從
[hhbp-common](https://github.com/rabbit4431/hhbp-common) 引入。取得最新的共用內容：

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## 注意事項

- **MCP 包裝鍵：** `common/mcp/mcp.json` 使用 `mcpServers` 包裝鍵（與 Claude Code 共用）。Codex
  也接受未包裝的伺服器對應表或 `mcp_servers` 包裝鍵。若未來的 Codex 版本拒絕 camelCase 鍵，
  修正會在 `common` 中進行，以確保兩個封裝都能繼續運作。
- **掛鉤的工具 matcher：** 掛鉤 matcher（`Bash`、`Write|Edit|MultiEdit`）以 Claude Code 的工具
  名稱命名。請確認它們在 Codex 的工具名稱下也會觸發；任何 Codex 特有的 matcher 覆寫都應放在
  Codex 自有的掛鉤檔案中，而非 `common`。
