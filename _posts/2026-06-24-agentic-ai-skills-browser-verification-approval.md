---
title: "에이전틱 AI 스택이 skills, 브라우저 검증, 승인으로 나뉘는 이유"
date: "2026-06-24"
keywords: ["agentic AI", "GitHub agent skills", "Playwright MCP", "CISA", "browser verification"]
lang: "ko"
description: "CISA의 agentic AI 가이드, GitHub의 agent skills, Playwright MCP를 엮어 에이전틱 AI를 안전하고 재현 가능하게 배포하는 3층 구조를 정리합니다."
---

# 에이전틱 AI 스택이 skills, 브라우저 검증, 승인으로 나뉘는 이유

에이전틱 AI를 쓰는 방식이 빠르게 바뀌고 있습니다. 예전에는 “좋은 프롬프트를 어떻게 쓰느냐”가 중심이었다면, 지금은 “작업을 어떤 단위로 나누고, 어디서 검증하며, 어떤 행동은 인간이 멈춰 세울 것이냐”가 더 중요해졌습니다. 이 흐름은 감상이 아니라 문서로도 보입니다. CISA는 미국 및 국제 파트너와 함께 agentic AI의 안전한 채택 가이드를 내놨고, GitHub Docs는 agent skills를 Copilot이 특화 작업을 수행하도록 돕는 기능으로 설명합니다. Playwright MCP는 브라우저 자동화를 접근성 스냅샷 중심으로 다루며, 코드 에이전트라면 CLI+SKILLS가 더 효율적일 수 있다고까지 적고 있습니다.

제가 보기에 이 세 문서는 같은 방향을 가리킵니다. 에이전틱 AI는 더 큰 모델 하나로 해결되는 문제가 아니라, **작업 지식, 실행, 검증, 승인**을 분리한 스택으로 다뤄야 한다는 뜻입니다.

## 1. agentic AI의 핵심은 “똑똑함”보다 “분리”입니다

에이전트가 위험해지는 이유는 단순합니다. 한 번의 요청이 곧 여러 행동으로 번지기 때문입니다. 문서 작성, 브라우저 클릭, 배포, 외부 API 호출, 파일 수정이 한 줄로 이어지면, 문제의 원인도 결과도 흐려집니다. 그래서 실무에서는 먼저 “무엇을 할 수 있는가”보다 “무엇을 한 번에 묶지 않을 것인가”를 정해야 합니다.

여기서 agent skills가 유용합니다. GitHub Docs의 정의대로라면 skills는 Copilot이 특화된 작업을 수행하도록 돕는 지침 묶음입니다. 즉, skill은 모델 자체가 아니라 **작업을 포장하는 방식**입니다. 예를 들어 블로그 발행 skill, 릴리스 노트 skill, QA 체크 skill을 따로 두면, 에이전트는 매번 긴 지시문을 새로 해석하지 않아도 됩니다. 반복되는 판단 기준이 파일 단위로 고정되기 때문입니다.

이 접근의 장점은 두 가지입니다. 첫째, 작업 재현성이 좋아집니다. 둘째, “이 행동을 왜 했는지”를 나중에 추적하기 쉬워집니다. CISA가 말하는 secure adoption도 결국 이 방향과 닿아 있습니다. 에이전트를 쓰지 말라는 뜻이 아니라, **통제 가능한 방식으로 채택하라**는 뜻에 가깝습니다.

## 2. 실전 구조는 skills, execution, verification 세 층이면 충분합니다

가장 단순한 구조는 세 층입니다.

- **Skills layer**: 작업 규칙과 체크리스트를 담습니다.
- **Execution layer**: 실제로 명령을 실행합니다.
- **Verification layer**: 결과가 맞는지 다시 확인합니다.

예를 들어 “게시글 발행” 작업을 생각해 보겠습니다. skill에는 제목 규칙, frontmatter 형식, 금지 표현, 발행 전 확인 항목을 넣습니다. execution layer에서는 파일 생성과 커밋을 수행합니다. verification layer에서는 브라우저에서 실제 렌더링을 확인하거나, CI에서 링크와 빌드 오류를 잡습니다.

Playwright MCP가 여기서 중요한 이유가 있습니다. 공식 README는 이 서버가 Playwright를 이용한 browser automation capabilities를 제공하고, LLM이 웹페이지와 structured accessibility snapshots로 상호작용하게 해준다고 설명합니다. 즉, 스크린샷만 보는 방식보다 DOM 구조와 접근성 상태를 더 잘 읽을 수 있습니다. 에이전트가 “보긴 봤는데 실제로 동작했는지 모르는” 상태를 줄여주는 셈입니다.

아래는 브라우저 검증을 붙일 때의 가장 단순한 설정 예시입니다.

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

이 설정만으로 끝내면 안 됩니다. 중요한 건 도구를 붙이는 것이 아니라, **언제 검증하고 언제 멈출지**를 정하는 일입니다.

## 3. 사람은 승인, 에이전트는 초안, 브라우저는 증거를 맡아야 합니다

실무에서 가장 흔한 실수는 에이전트에게 모든 것을 맡기고 “결과가 이상하면 다시 물어보지 뭐”라고 생각하는 것입니다. 그런데 에이전트가 잘못했을 때 가장 비싼 부분은 대개 되돌림입니다. 그래서 저는 다음 규칙이 가장 현실적이라고 봅니다.

1. 에이전트는 초안을 만든다.
2. 브라우저는 실제 화면과 동작을 검증한다.
3. 삭제, 전송, 배포, 권한 변경처럼 되돌리기 어려운 행동은 인간 승인으로 묶는다.

예를 들어 블로그 자동화라면, 에이전트는 새 글을 쓰고 링크를 검사할 수 있습니다. Playwright MCP는 렌더링과 버튼 상태를 확인할 수 있습니다. 하지만 `git push`나 원격 배포는 승인 큐를 거치게 하는 편이 안전합니다. CISA의 secure adoption 가이드가 던지는 신호도 비슷합니다. 에이전틱 AI는 ‘자동화’가 아니라 ‘통제된 자동화’로 접근해야 합니다.

아래처럼 아주 작은 승인 정책만 두어도 사고를 줄일 수 있습니다.

```yaml
agent:
  name: post-publisher
  allowed:
    - write_draft
    - validate_links
    - run_preview
  requires_approval:
    - git_push
    - deploy
    - send_notification
    - delete_file
  evidence:
    - browser_check
    - diff_summary
    - validation_log
```

이 정책의 핵심은 복잡함이 아니라 경계입니다. 에이전트가 할 수 있는 일을 늘리는 것보다, **할 수 없는 일을 분명히 적어 두는 것**이 더 중요합니다.

## 4. 왜 지금 이 구조가 더 맞는가

이유는 단순합니다. 에이전트가 길어질수록 컨텍스트는 빨리 소모되고, 실패 원인은 더 추적하기 어려워집니다. 반대로 skill과 verification을 분리하면 각 단계의 책임이 명확해집니다. 특히 Playwright MCP README가 coding agent의 경우 CLI+SKILLS가 더 token-efficient할 수 있다고 적는 대목은 중요합니다. 모든 일을 MCP로 처리하는 대신, 반복 작업은 skill/CLI로 묶고, 브라우저 확인이 필요한 구간만 MCP로 넘기라는 뜻으로 읽힙니다.

정리하면 이렇습니다.

- **skills**는 작업 지식을 재사용 가능하게 만든다.
- **브라우저 검증**은 “보였다”와 “동작했다”를 분리한다.
- **승인**은 고위험 행동을 멈출 수 있게 만든다.
- **CISA의 가이드**는 이 조합이 단순한 취향이 아니라 안전한 채택 전략임을 보여준다.

오늘 당장 해볼 수 있는 첫 단계는 하나입니다. 지금 쓰는 에이전트 작업 하나를 골라서, 그 작업을 **skill / verification / approval** 세 파일로 쪼개 보십시오. 그 순간부터 에이전틱 AI는 마법이 아니라 운영 시스템이 됩니다.

## 참고한 공식 문서

- CISA, US and International Partners Release Guide to Secure Adoption of Agentic AI
- GitHub Docs: About agent skills
- Microsoft Playwright MCP README
- GitHub: microsoft/playwright-mcp
