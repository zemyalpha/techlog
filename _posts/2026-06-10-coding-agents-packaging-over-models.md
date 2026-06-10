---
title: "코딩 에이전트의 경쟁력은 모델보다 패키징에 있다"
date: "2026-06-10"
keywords: ["GitHub Agentic Workflows", "Claude Code", "subagents", "skills", "GitHub Actions"]
lang: "ko"
description: "GitHub Agentic Workflows와 Claude Code의 subagents·skills를 묶어, 코딩 에이전트를 모델이 아니라 패키징과 경계로 설계하는 방법을 설명한다."
---

# 코딩 에이전트의 경쟁력은 모델보다 패키징에 있다

많은 팀이 아직도 코딩 에이전트의 성능을 모델 이름으로만 비교한다. 하지만 최근 문서들을 보면 실제 차이는 “무슨 모델이냐”보다 “어떤 일을 맡길 수 있게 패키징했느냐”에서 더 크게 벌어진다. GitHub Agentic Workflows는 표준 GitHub Actions 워크플로우 위에서 돌아가되, 읽기 전용 권한과 승인된 safe output 경로를 기본으로 둔다. Anthropic의 Claude Code 문서도 같은 방향을 보여 준다. skills는 `SKILL.md` 하나로 정의하고, subagents는 YAML frontmatter가 붙은 Markdown 파일로 정의한다.

즉, 에이전트의 핵심은 채팅 한 번에 다 시키는 것이 아니라, 반복되는 행동을 재사용 가능한 작업 단위로 포장하는 데 있다.

## 1. 왜 모델보다 패키징이 먼저인가

Claude Code 문서에 따르면 skills는 “Claude가 할 수 있는 일을 확장”하는 장치다. `SKILL.md`를 만들면 Claude가 필요할 때만 그 지침을 불러오고, 긴 참고 자료도 실제로 쓰일 때만 비용을 치른다. subagents 역시 마찬가지다. 공식 문서는 subagents를 “Markdown 파일 + YAML frontmatter”로 정의하며, `/agents` 명령으로 만들 수도 있다고 설명한다.

이 구조가 중요한 이유는 간단하다. 모델은 문장을 잘 생성하지만, 패키징은 행동의 경계를 만든다. GitHub 쪽은 오케스트레이션, Claude 쪽은 전문화에 가깝다. 하나는 저장소 수준의 흐름을 조율하고, 다른 하나는 코드 리뷰·요약·릴리즈 노트 같은 좁은 업무를 맡는다. 이 둘을 섞으면 편해 보이지만, 실제로는 리뷰와 책임 범위가 흐려진다.

## 2. 실전 구성: 오케스트레이터와 전문 에이전트 분리하기

아래는 구조를 보여 주는 예시다. 핵심은 GitHub Actions가 읽기 전용으로 분석을 돌리고, 전문 작업은 별도 패키지로 넘기는 것이다.

```yaml
name: repo-orchestrator
on:
  workflow_dispatch:

permissions:
  contents: read
  issues: read
  pull-requests: read

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build report
        run: python tools/build_report.py > report.md
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: repo-report
          path: report.md
```

이런 식으로 먼저 “읽기”를 고정해 두면, 결과물은 남고 권한은 넓어지지 않는다. GitHub가 말하는 safe output의 철학도 결국 여기에 있다. 쓰기 작업은 사람이 검토할 수 있는 경로로만 열어 두는 것이다.

Claude 쪽에서는 반복 지침을 `SKILL.md`와 subagent로 쪼갠다.

```md
---
name: release-note
description: PR diff를 읽고 사용자 영향 중심의 릴리즈 노트를 만든다
---

- 변경 의도와 사용자 영향을 먼저 요약한다.
- 실패 시나리오와 롤백 힌트를 짧게 적는다.
- 내부 구현 설명보다 외부 변화에 집중한다.
```

```md
---
name: code-reviewer
description: 코드 리뷰와 리스크 지점을 찾는 전용 subagent
---

- 변경 범위와 테스트 결과를 먼저 읽는다.
- 스타일보다 회귀 가능성을 우선한다.
- 수정이 필요하면 짧은 diff 후보만 제시한다.
```

## 3. 이 조합이 강한 이유

GitHub Agentic Workflows는 sandboxing, permissions, control, review를 붙여 에이전트를 저장소 운영에 넣는다. GitHub blog는 이런 워크플로우가 설정에 따라 Copilot CLI, Claude Code, OpenAI Codex 같은 서로 다른 엔진을 사용할 수 있다고 설명한다. 반면 Claude Code의 skills와 subagents는 “같은 말을 계속 반복하지 않게” 해 준다. 공식 문서가 Claude Code를 터미널, IDE, 데스크톱 앱, 브라우저에서 쓸 수 있는 도구로 설명하는 이유도 여기에 있다. 행동 패키지가 작고 명확하면, 실행 환경이 바뀌어도 같은 기준을 유지하기 쉽다.

## 4. 흔한 실수

- 거대한 프롬프트 하나에 분석, 수정, 검토를 모두 넣기
- 읽기 전용 단계와 쓰기 단계를 분리하지 않기
- `SKILL.md`를 정적 문서처럼 두고 실제 호출 조건을 적지 않기
- subagent의 범위를 너무 넓게 잡아 결국 “만능 봇”으로 되돌리기

에이전트가 강해질수록 중요한 건 더 큰 권한이 아니라 더 작은 경계다.

## 결론: 다음에 바꿀 것은 모델이 아니라 구조다

- GitHub는 오케스트레이션 레이어로 두고, 읽기 전용을 기본값으로 삼자.
- Claude Code의 `SKILL.md`는 반복 절차를 패키징하는 가장 싼 방법이다.
- subagents는 좁고 명확한 전문 업무에 넣을수록 효과가 크다.
- write 작업은 사람이 확인한 뒤에만 열어 두는 편이 안전하다.
- 다음 단계는 하나다. 지금 가장 자주 반복하는 작업 하나를 골라 `SKILL.md`로 먼저 빼 보자.

모델은 계속 바뀌지만, 좋은 패키징은 오래 남는다. 코딩 에이전트의 경쟁력은 결국 그 차이에서 결정된다.
