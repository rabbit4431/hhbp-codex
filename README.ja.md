# hhbp-codex

[English](README.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Русский](README.ru.md) · **日本語** · [简体中文](README.zh-CN.md) · [繁體中文](README.zh-TW.md) · [한국어](README.ko.md)

Heshuo Harness Best Practices — Java Maven マイクロサービス開発向けに、DDD 標準に沿ったスキル、
安全性のためのフック、MCP 設定を提供する [OpenAI Codex](https://developers.openai.com/codex) プラグインです。

## 概要

`hhbp-codex` は [`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code) の
Codex 互換の兄弟プラグインです。どちらも共有コンテンツ **`hhbp-common`**（ここでは git subtree で
`common/` ディレクトリに取り込み）を包む薄いラッパーで、ホスト環境のプラグイン形式でそれを公開します。

このラッパーは [Codex プラグイン仕様](https://developers.openai.com/codex/plugins/build)に
従っており、次のものを提供します。

- **スキル** — `common/skills/` にある 22 の共有ワークフロー（`SKILL.md` 定義）を変換せずに
  そのまま使用します:
  - *マルチサービス機能ワークフロー* — `springboot-feature-bootstrap` が 9 フェーズのチェーンを
    統括します: `service-discovery`、`feature-workspace-setup`、`contract-first-design`、
    `parallel-implementation`、`multi-service-integration-test`、`cross-service-review`、
    `multi-branch-merge`、`feature-cleanup`
  - *基盤* — `test-driven-development`、`writing-implementation-plan`、
    `subagent-task-execution`、`code-review-request`、`finishing-a-feature-branch`、
    `security-review`、`docs-lookup`、`language-setup`
  - *コード生成と SQL* — `generate-api`、`generate-code`、`remove-unused-class`、`sql-analyze`、
    `slow-sql-optimize`
- **フック** — SessionStart（`language.js`、選択した言語を適用）、
  PreToolUse（`block-dangerous.js`、危険なシェルコマンドをブロック）、
  PostToolUse（`git-workflow.js`、git のコミット規約を徹底）。マニフェストは
  `common/hooks/hooks.json` ではなく Codex 専用の `plugins/hhbp-codex/hooks/hooks.json` を
  参照しているため、Codex 固有の調整（たとえば Codex はまだ非同期フックに対応していません）を
  `common` に入れる必要はありません。フックのコマンドは、Codex が `${PLUGIN_ROOT}` の別名として
  提供する `${CLAUDE_PLUGIN_ROOT}` を通じて、`common/scripts/hooks/` の共有スクリプトを呼び出します。
- **MCP** — `common/mcp/mcp.json` のサーバー設定（例: 最新ドキュメントを参照するための `context7`）。

**対象外:** エージェントとコマンドは、このプラグインでは意図的に**公開していません**。

## ディレクトリ構成

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

このリポジトリは Codex の**マーケットプレイス**で、唯一のプラグインは `plugins/hhbp-codex/` に
あります。Codex は `source.path` がマーケットプレイスのルートそのもの（`./`）であるプラグインを
検出しないため、プラグインはサブディレクトリに置く必要があります（`openai-curated` の慣例と同じ）。
マニフェストの `skills`、`hooks`、`mcpServers` フィールドは `./common/` subtree のディレクトリを
直接参照しており、環境ごとの重複コピーはありません。

## インストール

1. マーケットプレイスを追加し、プラグインをインストールします:

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Codex の `config.toml` で MCP サーバーを有効にします:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. 言語を選びます — [言語](#言語)を参照してください。新しいフックは、Codex が実行する前に
   `/hooks` で確認して信頼する必要があります。

## 言語

プラグインは次の言語で回答し、計画、レビュー、レポート、コミットメッセージの説明、
コードコメントを書くことができます:

| コード | 言語 |
|---|---|
| `en` | English（デフォルト） |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

Codex にはインストール時にオプションを尋ねる仕組みがないため、プラグインは `language-setup`
スキルをオンボーディングスキル（`extensions.com.openai.onboardingSkill`）として登録しています。
プラグインのセットアップ時にこのスキルが実行され、言語の選択を求められます。「HHBP の言語を
変更して」と頼めば、いつでも再実行できます。選択内容は `~/.hhbp/config.json` に保存されます:

```json
{ "language": "ja" }
```

環境変数 `HHBP_LANGUAGE`（例: `HHBP_LANGUAGE=ja codex`）はこのファイルより優先されます。
新しい言語は次のセッションから適用されます。

コードの識別子、ファイル名、コマンド、コミットの `<type>:` プレフィックス、ワークフロー
プロトコルのトークン（`feature-state.json`、`REVIEW.md` の判定）は、スキルが正しく動作し続ける
よう英語のままです。ブロックされたコマンドのメッセージも翻訳されます。スキルと仕様書は英語で
書かれています。言語設定が変えるのは出力だけです。

## モデル

スキルは Codex セッションが使用するモデルで実行され、すべてのサブエージェントもそれを引き継ぎます。
`common/skills` の `model:` frontmatter は Claude Code 専用で、Codex は無視します。モデルは
セッション開始時に選んでください（`codex -m <model>`、または `~/.codex/config.toml` の `model`）。

このワークフローはサブエージェントを 2 階層にネストします — 実装ストリーム
（`subagent-task-execution`）がタスクごとに実装担当とレビュー担当のサブエージェントを起動します —
そのため、Codex の既定の起動深度 1 を引き上げてください:

```toml
[agents]
max_depth = 2
```

## common の更新

`plugins/hhbp-codex/common/` は [hhbp-common](https://github.com/rabbit4431/hhbp-common) から
git subtree で取り込まれています。最新の共有コンテンツを取得するには:

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## 注意事項

- **MCP のラッパーキー:** `common/mcp/mcp.json` は（Claude Code と共通の）`mcpServers`
  ラッパーキーを使用しています。Codex はラッパーなしのサーバーマップや `mcp_servers` ラッパーも
  受け付けます。将来の Codex が camelCase のキーを拒否するようになった場合は、両方のラッパーが
  動作し続けるよう `common` 側で修正します。
- **フックのツール matcher:** フックの matcher（`Bash`、`Write|Edit|MultiEdit`）は Claude Code の
  ツール名に基づいています。Codex のツール名でも発火することを確認してください。Codex 固有の
  matcher の上書きは `common` ではなく、Codex 専用のフックファイルに置いてください。
