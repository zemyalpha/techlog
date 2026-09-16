---
title: "Claude Agent SDK 실전 가이드 — 권한·훅·서브에이전트로 프로덕션 에이전트 만들기"
date: "2026-09-16"
keywords: ["Claude Agent SDK", "AI 에이전트", "hooks", "subagents", "permission", "Claude Code"]
lang: "ko"
description: "Claude Agent SDK의 권한 평가 순서, 훅 시스템, 서브에이전트 구성을 공식 문서 기반으로 정리한다. Python/TypeScript 코드 예시와 함께 프로덕션 에이전트의 안전장치 설계 방법을 설명한다."
---

# Claude Agent SDK 실전 가이드 — 권한·훅·서브에이전트로 프로덕션 에이전트 만들기

"파일을 읽고, 명령을 실행하고, 코드를 수정하는 에이전트"를 직접 만들고 싶다면, 2026년 현재 가장 빠른 출발점 중 하나가 Claude Agent SDK다. 이 SDK는 Claude Code CLI를 라이브러리 형태로 감싼 것으로, Claude Code가 실제로 사용하는 도구들(파일 읽기, 셸 명령, 코드 편집), 에이전트 루프, 컨텍스트 관리를 그대로 Python과 TypeScript에서 프로그래밍할 수 있게 해준다.

도구 실행 로직을 직접 구현할 필요가 없다는 점이 핵심이다. `query()` 함수에 프롬프트와 옵션만 넘기면 SDK가 내장된 Claude Code 바이너리를 실행하고, 에이전트가 알아서 파일을 탐색하고 수정한다. 문제는 반대 방향이다 — 도구가 강력할수록 **무엇을 못 하게 막을지**가 설계의 본질이 된다. 이 글에서는 공식 문서를 기준으로 권한(permission), 훅(hooks), 서브에이전트(subagents) 세 축을 정리한다.

## 1단계: 설치와 첫 실행

TypeScript와 Python 모두 지원한다. 두 SDK 모두 플랫폼별 Claude Code 바이너리를 번들하므로 Claude Code를 따로 설치할 필요가 없다.

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python (3.10 이상 필요)
pip install claude-agent-sdk
```

API 키는 환경변수로만 전달한다. SDK가 `.env` 파일을 자동으로 로드하지 않으므로, `.env`를 쓴다면 실행 전에 직접 로드해야 한다.

```bash
export ANTHROPIC_API_KEY=your-api-key
```

가장 단순한 에이전트는 다음과 같다. Python 예시:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"],
        ),
    ):
        print(message)

asyncio.run(main())
```

`allowed_tools`에 이름을 적은 도구는 사전 승인되어 확인 프롬프트 없이 실행된다. AWS 환경이라면 Amazon Bedrock(`CLAUDE_CODE_USE_BEDROCK=1`)이나 Claude Platform on AWS(`CLAUDE_CODE_USE_ANTHROPIC_AWS=1`)를 통한 인증도 지원한다.

참고로 비용 구조가 바뀌었다. 2026년 6월 15일부터 구독 플랜에서 Agent SDK와 `claude -p` 사용량은 대화형 사용량과 분리된 별도의 월간 Agent SDK 크레딧에서 차감된다. API 키 결제가 아닌 구독으로 SDK를 돌리는 경우 예상치 못한 제한에 부딪히지 않도록 이 변경을 숙지해야 한다.

## 2단계: 권한 — 평가 순서를 알아야 막을 수 있다

Agent SDK의 권한 시스템은 단일 스위치가 아니라 **평가 파이프라인**이다. 순서는 다음과 같다.

1. **훅(hooks)** — 도구 호출 요청을 가로채 거부하거나 수정할 수 있다
2. **deny 규칙** — `disallowed_tools`로 지정
3. **ask 규칙** — 사용자 확인 요구
4. **권한 모드(permission mode)** — `default`, `acceptEdits`, `bypassPermissions` 등
5. **allow 규칙** — `allowed_tools`로 지정
6. **`canUseTool` 콜백** — 앞 단계에서 결정되지 않은 호출의 최종 판단

이 순서에서 자주 오해하는 지점 두 가지를 짚는다.

**첫째, `allowed_tools`는 `bypassPermissions`를 제약하지 않는다.** `bypassPermissions` 모드는 모드 단계에 도달한 모든 호출을 승인하므로, `allowed_tools=["Read"]`를 지정해도 `Bash`, `Write`, `Edit`이 전부 승인된다. 전부 허용하되 특정 도구만 막고 싶다면 `disallowed_tools`를 써야 한다.

**둘째, deny 규칙의 형태에 따라 동작이 다르다.**

```python
# 도구 정의 자체를 제거 — Claude가 Bash를 시도조차 못 한다
disallowed_tools=["Bash"]

# Bash는 유지하되 rm 호출만 모든 모드에서 거부
disallowed_tools=["Bash(rm *)"]
```

또한 `rm`/`rmdir`로 지정된 크리티컬 경로 삭제는 어떤 모드·어떤 allow 규칙으로도 자동 승인되지 않고 반드시 확인 단계로 넘어간다. 최후의 안전장치가 코드 밖에 있는 셈이다.

권한 모드는 5가지다. `default`(표준), `dontAsk`(확인 대신 거부), `acceptEdits`(파일 편집·파일시스템 명령 자동 승인, 단 작업 디렉토리 내부로 한정), `bypassPermissions`(거의 전부 승인), `plan`(편집 없이 탐색·계획만). 서버에 붙여 24시간 돌리는 무인 에이전트라면 `default` + allow/deny 규칙 + 훅 조합이, 통제된 컨테이너 안의 CI 잡이라면 `bypassPermissions`가 현실적인 선택이 된다.

## 3단계: 훅 — 도구 호출 전후에 내 코드를 끼워 넣기

훅은 에이전트 실행의 특정 시점에 실행되는 콜백이다. 가장 많이 쓰이는 것은 `PreToolUse`(도구 호출 직전, 차단·수정 가능)와 `PostToolUse`(도구 실행 후)다.

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher

# .env 파일 보호 훅: Edit/Write가 .env를 건드리면 거부
async def protect_env_files(input, tool_use_id, extra):
    file_path = input.get("tool_input", {}).get("file_path", "")
    if file_path.endswith(".env"):
        return {"hookSpecificOutput": {
            "permissionDecision": "deny",
            "permissionDecisionReason": ".env 파일 수정 금지",
        }}
    return {}

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(hooks=[protect_env_files])]}
)
```

핵심 규칙 세 가지:

- **훅의 deny는 `bypassPermissions`에서도 적용된다.** 모든 권한 평가보다 먼저 실행되기 때문이다. "전부 허용하는 모드지만 이것만은 절대 안 된다"는 요구사항은 훅으로만 구현할 수 있다.
- **여러 훅이 매칭되면 병렬로 실행되고, 가장 제한적인 결과가 적용된다.** 하나라도 deny를 반환하면 다른 훅이 allow를 반환했어도 차단된다.
- **도구 입력 수정도 가능하다.** `updatedInput`을 반환하면 호출 내용을 고쳐서 진행할 수 있다. 단 `permissionDecision: 'defer'`와 함께 쓰면 수정이 무시되니 주의.

이벤트 종류는 `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `SubagentStop`, `PreCompact`, `Notification` 등이 Python과 TypeScript 공통이고, `SessionStart`/`SessionEnd`, `PostToolBatch`, `PreModelSwitch` 등은 TypeScript 전용이다. Python에서 세션 시작/종료 훅이 필요하면 SDK 콜백 대신 `.claude/settings.json`의 셸 커맨드 훅으로 정의해야 한다.

## 4단계: 서브에이전트 — 컨텍스트 격리와 병렬화

서브에이전트는 메인 에이전트가 파생시키는 별도의 에이전트 인스턴스다. 정의 방법은 세 가지다.

1. **프로그래밍 방식** — `query()` 옵션의 `agents` 파라미터 (SDK 애플리케이션에 권장)
2. **파일시스템 방식** — `.claude/agents/` 디렉토리에 마크다운 파일로 정의
3. **내장 general-purpose 서브에이전트** — 정의 없이 `Agent` 도구로 즉시 사용

```python
from claude_agent_sdk import AgentDefinition

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Glob", "Agent"],
    agents={
        "code-reviewer": AgentDefinition(
            description="코드 리뷰 전문가. 품질·보안·유지보수성 검토에 사용.",
            # 이 서브에이전트는 읽기 도구만 허용 → 수정 불가능
        ),
    },
)
```

서브에이전트가 주는 이점은 4가지로 정리된다.

- **컨텍스트 격리** — 서브에이전트가 수십 개 파일을 읽어도 그 내용은 서브에이전트 대화 안에 머물고, 부모에게는 최종 요약만 전달된다. 메인 컨텍스트가 검색 결과로 오염되는 것을 막는다.
- **병렬화** — `style-checker`, `security-scanner`, `test-coverage` 서브에이전트를 동시에 돌려 순차 실행의 합이 아니라 가장 느린 하나의 시간으로 끝낸다.
- **전문화** — 데이터베이스 마이그레이션 서브에이전트에 롤백 전략과 무결성 검사 지침만 넣어, 메인 프롬프트를 어지럽히지 않는다.
- **도구 제한** — 문서 리뷰 서브에이전트를 `Read`/`Grep`으로 제한하면 실수로 파일을 수정할 일이 원천적으로 없다.

주의점도 있다. 여러 서브에이전트를 띄우면 각자 권한 확인을 요청할 수 있어 프롬프트가 늘어난다. 이때는 `PreToolUse` 훅으로 특정 도구를 자동 승인하거나 권한 규칙을 정리하라 — 서브에이전트는 부모 대화의 권한 설정을 상속받는다. 또한 `agents` 파라미터로 전달한 프로그래밍 방식 에이전트는 같은 이름의 파일시스템 에이전트를 오버라이드한다.

## 결론: 안전장치 3층 구조로 시작하라

Claude Agent SDK의 설계 철학은 요약하면 "강력한 도구는 기본 제공, 제어는 계층으로"다. 프로덕션에 넣기 전에 다음 구조를 권한다.

- **1층 — deny 규칙**: `disallowed_tools`로 절대 금지 행위(예: `Bash(rm *)`)를 선언적으로 차단한다
- **2층 — PreToolUse 훅**: 동적 판단이 필요한 정책(파일 경로 패턴, 시간대 제한)을 코드로 구현한다. `bypassPermissions`에서도 살아있는 유일한 차단 계층이다
- **3층 — 서브에이전트 도구 제한**: 위험한 작업을 읽기 전용 서브에이전트에 격리해 폭발 반경을 줄인다

첫 단계는 간단하다. `pip install claude-agent-sdk` 후 버그 수정 에이전트를 `default` 모드로 돌려보고, 실행 로그의 도구 호출 패턴을 관찰한 뒤, 그 패턴에 맞는 deny 규칙과 훅을 한 개씩 추가하면 된다. 에이전트를 신뢰하는 것은 출력 품질이 아니라 제어 구조의 완성도에서 나온다.

## 참고 자료

- Agent SDK 개요 — https://code.claude.com/docs/en/agent-sdk/overview
- 권한 구성 — https://code.claude.com/docs/en/agent-sdk/permissions
- 훅 가이드 — https://code.claude.com/docs/en/agent-sdk/hooks
- 서브에이전트 — https://code.claude.com/docs/en/agent-sdk/sub-agents
