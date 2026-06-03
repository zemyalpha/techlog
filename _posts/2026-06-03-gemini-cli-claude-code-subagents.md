---
title: "Gemini CLI와 Claude Code에서 서브에이전트를 나누는 법"
date: "2026-06-03"
keywords: ["Gemini CLI", "Claude Code", "subagents", "context window", "parallel execution"]
lang: "ko"
description: "Gemini CLI와 Claude Code의 서브에이전트 기능을 비교하고, 컨텍스트 오염을 줄이는 실전 분업 구조를 정리한다."
---

# Gemini CLI와 Claude Code에서 서브에이전트를 나누는 법

AI 코딩 도구가 빨라질수록 오히려 더 중요해지는 건 “한 번에 얼마나 많이 시키느냐”가 아니라 “어떻게 나눠서 시키느냐”입니다. 같은 작업이라도 메인 대화에 로그, 검색 결과, 수정안, 검토 의견이 한꺼번에 섞이면 컨텍스트가 금세 지저분해집니다. 이때 서브에이전트는 단순한 편의 기능이 아니라, 작업을 격리하고 요약만 회수하기 위한 구조적 장치가 됩니다.

최근 Google은 Gemini CLI의 subagents를 소개하면서, 이들이 **전용 컨텍스트 창**에서 복잡하고 반복적인 작업을 맡고, **병렬 실행**으로 시간을 줄이며, 메인 에이전트의 **context rot**를 줄인다고 설명했습니다. Claude Code 문서도 비슷한 방향입니다. 서브에이전트는 메인 대화가 감당하기 어려운 검색 결과, 로그, 파일 내용을 별도 컨텍스트에서 처리하고, 필요한 요약만 돌려주는 방식으로 설계됩니다.

## 1. 서브에이전트는 “더 똑똑한 챗봇”이 아니라 “격리된 작업자”다

서브에이전트를 잘 쓰는 팀은 공통적으로 역할을 분리합니다.

- **메인 에이전트**: 목표를 정하고, 판단하고, 최종 결론을 낸다.
- **리서치 서브에이전트**: 문서, 릴리스 노트, 오류 메시지를 읽고 핵심만 요약한다.
- **구현 서브에이전트**: 특정 파일이나 모듈만 수정한다.
- **리뷰 서브에이전트**: 안전성, 스타일, 누락된 테스트를 검사한다.

이 구조의 장점은 명확합니다. 메인 대화는 길어지지 않고, 각 서브에이전트는 자기 일에만 집중합니다. 특히 Gemini CLI처럼 여러 서브에이전트를 **동시에** 돌릴 수 있으면, 서로 독립적인 조사나 리팩터링을 병렬로 처리할 수 있습니다. Claude Code 역시 같은 맥락에서, “한 번 더 묻는 것”보다 “같은 종류의 작업자를 반복 생성하는 것”에 서브에이전트를 쓰라고 안내합니다.

## 2. 실전 분업: planner / researcher / executor / reviewer

가장 실용적인 패턴은 네 단계입니다.

1. **Planner**가 작업 목표를 짧게 정의한다.
2. **Researcher**가 공식 문서와 로그만 읽고 사실관계를 정리한다.
3. **Executor**가 바뀌어야 할 파일만 수정한다.
4. **Reviewer**가 변경 이유와 부작용을 점검한다.

이때 중요한 건 서브에이전트가 “모든 걸 다 아는” 존재가 아니라는 점입니다. 오히려 범위를 좁힐수록 결과가 좋아집니다. 예를 들어 문서 조사 서브에이전트에는 쓰기 권한이 필요 없습니다. 반대로 구현 서브에이전트에는 과도한 외부 도구보다 코드 수정 권한과 최소한의 읽기 권한만 주는 편이 안전합니다.

## 3. 바로 쓸 수 있는 최소 설정 예시

Gemini CLI 쪽은 Markdown 파일과 YAML frontmatter만으로 시작할 수 있습니다. Google은 서브에이전트를 `~/.gemini/agents`에 두면 개인용으로, `.gemini/agents`에 두면 프로젝트 단위로 재사용할 수 있다고 설명합니다.

```md
---
name: docs-researcher
description: 공식 문서에서 변경 사항만 찾아 요약하는 리서치 전용 에이전트
---

공식 문서, 릴리스 노트, 오류 메시지만 읽어라.
추측하지 말고, 확인된 사실만 5줄 이내로 요약해라.
```

Claude Code는 `/agents` 명령으로 서브에이전트를 만들 수 있고, 파일 기반 서브에이전트는 `~/.claude/agents/` 또는 `.claude/agents/`에 저장합니다. 문서에는 `description`, `prompt`, `tools`, `disallowedTools`, `model` 같은 필드를 지원한다고 나와 있습니다.

```md
---
name: code-reviewer
description: 변경된 코드의 안전성과 테스트 누락을 검토하는 리뷰어
tools: ["Read", "Grep", "Glob", "Bash"]
model: sonnet
---

변경된 코드의 위험 지점만 찾아라.
새로운 버그 가능성, 테스트 누락, 권한 과다를 우선 확인해라.
결과는 문제점 / 근거 / 수정 제안 순서로 정리해라.
```

## 4. 자주 하는 실수

가장 흔한 실수는 서브에이전트에게 메인 에이전트와 같은 권한을 주는 것입니다. 그러면 격리의 이점이 사라집니다. 두 번째 실수는 서브에이전트 결과를 원문 그대로 붙여넣는 것입니다. 서브에이전트는 “요약을 돌려주는 엔진”으로 쓰는 편이 훨씬 낫습니다. 세 번째는 병렬 실행을 무조건 쓰는 것입니다. 서로 의존하는 작업은 순서대로 돌려야 합니다.

## 결론: 분업이 곧 품질이다

- Gemini CLI subagents는 별도 컨텍스트 창과 병렬 실행에 강점이 있다.
- Claude Code subagents는 작업별 역할 분리와 권한 제어가 탄탄하다.
- 둘 다 핵심은 “메인 대화는 판단, 서브에이전트는 처리”로 나누는 데 있다.
- 검색, 로그 분석, 코드 리뷰처럼 반복되는 일을 먼저 분리하면 효과가 크다.
- 오늘 당장 하나의 리서치 작업과 하나의 리뷰 작업만 서브에이전트로 빼 보자.

이 글의 첫 번째 실전 과제는 간단합니다. 자주 반복하는 작업 하나를 골라, 리서치용과 리뷰용 서브에이전트를 각각 하나씩 만들어 보세요. 대화창이 짧아질수록, 오히려 작업 품질은 더 선명해집니다.

## 참고 자료

- Google Developers Blog: Subagents have arrived in Gemini CLI — https://developers.googleblog.com/subagents-have-arrived-in-gemini-cli/
- Google Blog: Google announces Gemini CLI: your open-source AI agent — https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemini-cli-open-source-ai-agent/
- Claude Code Docs: Create custom subagents — https://code.claude.com/docs/en/sub-agents
- Anthropic: Claude Code | Anthropic's agentic coding system — https://www.anthropic.com/product/claude-code
