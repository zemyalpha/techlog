---
title: "Codex의 Ona 인수가 말해주는 것: 에이전트는 클라우드 작업공간이 필요하다"
date: "2026-06-14"
keywords: ["OpenAI Codex", "Ona", "GitHub Copilot", "Claude Code", "MCP"]
lang: "ko"
description: "OpenAI의 Ona 인수, GitHub Copilot의 agent sessions, Claude Code의 project-level 작업 방식이 보여주는 클라우드 작업공간 전환을 정리한다."
---

# Codex의 Ona 인수가 말해주는 것: 에이전트는 클라우드 작업공간이 필요하다

코딩 에이전트를 둘러싼 최근 변화는 모델 경쟁보다 더 큰 방향 전환을 보여준다. 핵심은 더 똑똑한 답변이 아니라 **지속되는 작업공간**이다. OpenAI는 2026년 6월 11일 Ona 인수를 발표하며, Codex에 **secure, persistent cloud execution and orchestration**를 넣겠다고 밝혔다. 같은 시기 GitHub는 Copilot Chat이 **agent sessions**를 조회할 수 있게 만들었고, GitHub Agentic Workflows는 public preview에 들어가면서 `GITHUB_TOKEN` 기반 실행을 지원해 PAT 의존도를 줄였다. Anthropic의 Claude Code 역시 코드를 읽고, 여러 파일을 고치고, 테스트를 돌리고, 커밋된 결과를 내놓는 **project-level agentic system**으로 설명된다.

이 셋이 함께 말하는 바는 분명하다. 에이전트의 단위는 더 이상 “한 번의 프롬프트”가 아니다. 이제는 **상태를 유지하는 작업공간**, **세션 기록**, **검토 경로**가 있어야 실제 업무에 들어간다. 한마디로, 채팅창이 아니라 작업실이 필요하다.

## 1) 왜 채팅만으로는 부족한가

채팅은 질문과 답변에는 강하지만, 장시간 작업에는 약하다. 작업이 길어질수록 중요한 것은 문장 품질보다 다음 세 가지다.

- 무엇을 하기로 했는가
- 어디까지 실행했는가
- 어떤 근거로 멈추거나 되돌릴 것인가

그래서 좋은 에이전트 시스템은 대화창 하나로 끝나지 않는다. 계획은 문서로 남고, 실행은 격리된 workspace에서 이뤄지며, 결과는 로그와 diff로 검토된다. GitHub가 agent sessions를 다시 chat에서 보게 만든 이유도 여기에 있다. “무슨 말을 했는가”보다 **무슨 일이 실제로 일어났는가**가 더 중요해졌기 때문이다.

도구가 많아질수록 발견 계층도 필요해진다. MCP Registry 같은 공식 registry가 의미 있는 이유는, 도구를 단발성 스크립트가 아니라 **연결 가능한 인프라**로 바꿔 주기 때문이다.

## 2) 실전에서는 계획·실행·검토를 분리하라

가장 실용적인 구조는 단순하다. 에이전트에게 모든 권한을 주지 말고, 역할을 나눈다.

```yaml
workspace_contract:
  plan:
    input: "문제 정의, 목표, 제한조건"
    output: "작업 계획, 위험요소, 중단 조건"
  execute:
    input: "허용된 파일, 허용된 명령, 토큰 범위"
    output: "diff, 테스트 결과, 실행 로그"
  review:
    input: "변경 요약, 검증 근거, 남은 리스크"
    output: "승인, 수정 요청, 롤백"
```

이 구조는 거창하지 않지만, 실제로는 큰 차이를 만든다. 예를 들어 GitHub Actions에서 에이전트 작업을 돌릴 때도 같은 원칙이 적용된다.

```yaml
name: agent-task
on:
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Prepare workspace
        run: |
          mkdir -p .agent/session
          cat > .agent/session/brief.md <<'EOF'
          goal: fix the failing test
          stop: when tests pass
          review: attach diff and validation notes
          EOF
      - name: Record result
        run: |
          echo "save logs, diff, and summary here"
```

이런 식으로 만들면 에이전트가 “대화”를 남기는 것이 아니라 **작업 흔적**을 남긴다. 그리고 그 흔적이 다음 세션의 출발점이 된다.

## 3) 안전성은 기능이 아니라 기본값이어야 한다

최근 GitHub의 변화에서 특히 중요한 부분은 PAT 제거다. GitHub는 6월 11일, GitHub Agentic Workflows에서 `GITHUB_TOKEN`을 사용할 수 있게 했고, 그 결과 장기 PAT를 만들고 저장하는 운영 부담과 보안 위험을 줄일 수 있다고 설명했다. 이건 작은 편의 개선이 아니라, 에이전트 운영 방식의 기준이 바뀌고 있다는 신호다.

Claude Code도 같은 방향을 가리킨다. Anthropic은 Claude Code를 코드베이스를 읽고, 파일을 바꾸고, 테스트를 돌리고, 커밋된 코드를 내놓는 시스템으로 소개한다. 중요한 건 “자동으로 다 해준다”가 아니다. **무엇을 맡길지, 어디서 멈출지, 누가 최종 승인할지**를 정해 놓아야 실제로 안전하다는 점이다.

실무에서 바로 적용할 체크리스트는 이렇다.

- 장기 토큰 대신 짧은 수명 토큰이나 내장 토큰을 우선 쓴다
- 실행 workspace와 검토 workspace를 분리한다
- 세션 로그와 테스트 결과를 항상 남긴다
- PR 코멘트나 chat으로 최종 승인 경로를 고정한다
- 실패했을 때 되돌릴 기준을 미리 적어 둔다

## 결론: 에이전트는 이제 "답변 시스템"이 아니라 "작업공간"이다

핵심만 정리하면 이렇다.

- OpenAI는 Ona 인수를 통해 Codex에 지속형 클라우드 실행과 오케스트레이션을 넣으려 한다
- GitHub는 Copilot Chat과 agent sessions를 연결해 실행과 대화를 다시 묶고 있다
- Claude Code는 이미 코드베이스 단위의 작업 시스템으로 자리 잡았다
- 앞으로의 경쟁력은 모델 이름보다 **세션, 로그, 권한, 검토 경로**에 달려 있다
- 첫 단계는 에이전트를 똑똑하게 만드는 것이 아니라, **작업공간을 먼저 설계하는 것**이다

오늘 할 일은 하나면 충분하다. 다음 에이전트 작업 하나를 골라서, 목표·권한·중단 조건·검토 기준을 한 장짜리 문서로 먼저 써 보자. 그 문서가 쌓일수록 에이전트는 덜 떠들고 더 오래 일한다.
