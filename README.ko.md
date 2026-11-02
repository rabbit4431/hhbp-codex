# hhbp-codex

[English](README.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [Русский](README.ru.md) · [日本語](README.ja.md) · [简体中文](README.zh-CN.md) · [繁體中文](README.zh-TW.md) · **한국어**

Heshuo Harness Best Practices — Java Maven 마이크로서비스 개발을 위해 DDD 표준에 맞춘 스킬,
안전 훅, MCP 구성을 제공하는 [OpenAI Codex](https://developers.openai.com/codex) 플러그인입니다.

## 개요

`hhbp-codex`는 [`hhbp-claude-code`](https://github.com/rabbit4431/hhbp-claude-code)의
Codex 호환 자매 플러그인입니다. 둘 다 공유 콘텐츠 **`hhbp-common`**(여기서는 git subtree로
`common/` 디렉터리에 포함)을 감싸는 얇은 래퍼이며, 호스트 환경의 플러그인 형식으로 이를 제공합니다.

이 래퍼는 [Codex 플러그인 명세](https://developers.openai.com/codex/plugins/build)를 따르며
다음을 제공합니다.

- **스킬** — `common/skills/`의 공유 워크플로 22개(`SKILL.md` 정의)를 변환 없이 그대로 사용합니다:
  - *멀티 서비스 기능 워크플로* — `springboot-feature-bootstrap`이 9단계 체인을 조율합니다:
    `service-discovery`, `feature-workspace-setup`, `contract-first-design`,
    `parallel-implementation`, `multi-service-integration-test`, `cross-service-review`,
    `multi-branch-merge`, `feature-cleanup`
  - *기반* — `test-driven-development`, `writing-implementation-plan`,
    `subagent-task-execution`, `code-review-request`, `finishing-a-feature-branch`,
    `security-review`, `docs-lookup`, `language-setup`
  - *코드 생성 및 SQL* — `generate-api`, `generate-code`, `remove-unused-class`, `sql-analyze`,
    `slow-sql-optimize`
- **훅** — SessionStart(`language.js`, 선택한 언어 적용),
  PreToolUse(`block-dangerous.js`, 위험한 셸 명령 차단),
  PostToolUse(`git-workflow.js`, git 커밋 규칙 적용). 매니페스트는 `common/hooks/hooks.json`
  대신 Codex 전용 `plugins/hhbp-codex/hooks/hooks.json`을 가리키므로, Codex 전용 조정(예: Codex는
  아직 비동기 훅을 지원하지 않음)을 `common`에 넣을 필요가 없습니다. 훅 명령은 Codex가
  `${PLUGIN_ROOT}`의 별칭으로 제공하는 `${CLAUDE_PLUGIN_ROOT}`를 통해 `common/scripts/hooks/`의
  공유 스크립트를 호출합니다.
- **MCP** — `common/mcp/mcp.json`의 서버 구성(예: 최신 문서 조회용 `context7`).

**범위 밖:** 에이전트와 명령은 이 플러그인에서 의도적으로 **제공하지 않습니다**.

## 디렉터리 구조

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

이 저장소는 Codex **마켓플레이스**이며, 유일한 플러그인은 `plugins/hhbp-codex/`에 있습니다.
Codex는 `source.path`가 마켓플레이스 루트 자체(`./`)인 플러그인을 인식하지 않으므로, 플러그인은
`openai-curated` 관례에 따라 하위 디렉터리에 있어야 합니다. 매니페스트의 `skills`, `hooks`,
`mcpServers` 필드는 `./common/` subtree 디렉터리를 직접 가리키며, 환경별 중복 사본은 없습니다.

## 설치

1. 마켓플레이스를 추가하고 플러그인을 설치합니다:

   ```bash
   codex plugin marketplace add rabbit4431/hhbp-codex
   codex plugin add hhbp-codex@hhbp-codex
   ```

2. Codex `config.toml`에서 MCP 서버를 활성화합니다:

   ```toml
   [plugins."hhbp-codex".mcp_servers.context7]
   enabled = true
   ```

3. 언어를 선택합니다 — [언어](#언어)를 참고하세요. 새 훅은 Codex가 실행하기 전에 `/hooks`에서
   검토하고 신뢰해야 합니다.

## 언어

플러그인은 다음 언어로 답변하고 계획, 리뷰, 보고서, 커밋 메시지 설명, 코드 주석을 작성할 수
있습니다:

| 코드 | 언어 |
|---|---|
| `en` | English (기본값) |
| `fr` | Français |
| `de` | Deutsch |
| `ru` | Русский |
| `ja` | 日本語 |
| `zh-CN` | 简体中文 |
| `zh-TW` | 繁體中文 |
| `ko` | 한국어 |

Codex에는 설치 시 옵션을 묻는 기능이 없으므로, 플러그인은 `language-setup` 스킬을 온보딩 스킬
(`extensions.com.openai.onboardingSkill`)로 등록합니다. 플러그인 설정 시 이 스킬이 실행되어
언어를 선택하도록 요청합니다. "HHBP 언어 변경"이라고 요청하면 언제든 다시 실행할 수 있습니다.
선택한 언어는 `~/.hhbp/config.json`에 저장됩니다:

```json
{ "language": "ko" }
```

환경 변수 `HHBP_LANGUAGE`(예: `HHBP_LANGUAGE=ko codex`)가 이 파일보다 우선합니다. 새 언어는
다음 세션부터 적용됩니다.

코드 식별자, 파일 이름, 명령, 커밋 `<type>:` 접두사, 워크플로 프로토콜 토큰
(`feature-state.json`, `REVIEW.md` 판정)은 스킬이 계속 작동하도록 영어로 유지됩니다. 차단된
명령의 메시지도 번역됩니다. 스킬과 명세는 영어로 작성되어 있으며, 언어 설정은 출력에만 영향을
줍니다.

## 모델

스킬은 Codex 세션이 사용하는 모델로 실행되며, 모든 서브에이전트도 이를 상속합니다.
`common/skills`의 `model:` frontmatter는 Claude Code 전용이며 Codex는 무시합니다. 세션을 시작할
때 모델을 선택하세요(`codex -m <model>` 또는 `~/.codex/config.toml`의 `model`).

이 워크플로는 서브에이전트를 두 단계로 중첩합니다 — 구현 스트림(`subagent-task-execution`)이
작업마다 구현 담당과 리뷰 담당 서브에이전트를 실행합니다 — 따라서 Codex의 기본 생성 깊이 1을
높이세요:

```toml
[agents]
max_depth = 2
```

## common 업데이트

`plugins/hhbp-codex/common/`은 [hhbp-common](https://github.com/rabbit4431/hhbp-common)에서
git subtree로 가져옵니다. 최신 공유 콘텐츠를 받으려면:

```bash
# one-time: register the upstream remote
git remote add common https://github.com/rabbit4431/hhbp-common.git

# pull the latest common content into the common/ subtree
git subtree pull --prefix=plugins/hhbp-codex/common common main --squash
```

## 참고 사항

- **MCP 래퍼 키:** `common/mcp/mcp.json`은 (Claude Code와 공유하는) `mcpServers` 래퍼 키를
  사용합니다. Codex는 래퍼 없는 서버 맵이나 `mcp_servers` 래퍼도 허용합니다. 향후 Codex 버전이
  camelCase 키를 거부하면, 두 래퍼가 계속 작동하도록 `common`에서 수정합니다.
- **훅 도구 matcher:** 훅 matcher(`Bash`, `Write|Edit|MultiEdit`)는 Claude Code 도구 이름을
  따릅니다. Codex 도구 이름에서도 실행되는지 확인하세요. Codex 전용 matcher 재정의는 `common`이
  아닌 Codex 전용 훅 파일에 두어야 합니다.
