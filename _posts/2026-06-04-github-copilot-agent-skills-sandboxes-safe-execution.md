---
title: "Agent skills는 지식, sandboxes는 실행: GitHub Copilot를 안전하게 굴리는 법"
date: "2026-06-04"
keywords: ["GitHub Copilot", "agent skills", "sandboxes", "gh skill", "AI security"]
lang: "ko"
description: "GitHub Copilot의 agent skills와 로컬·클라우드 sandboxes를 분리해서 쓰는 이유와, 안전한 자동화 루프를 만드는 실전 방법을 정리한다."
---

# Agent skills는 지식, sandboxes는 실행: GitHub Copilot를 안전하게 굴리는 법

AI 코딩 에이전트가 강해질수록 오히려 더 중요해지는 건 “더 많은 일을 시키는 것”이 아니라 “어디까지 맡길지 경계를 정하는 것”입니다. 최근 GitHub는 이 경계를 나누는 두 장치를 빠르게 밀고 있습니다. 하나는 **agent skills**이고, 다른 하나는 **cloud/local sandboxes**입니다.

agent skills는 반복 작업을 위한 지식 묶음입니다. 반면 sandbox는 그 지식이 실제로 실행되는 장소입니다. 이 둘을 섞어 버리면, 에이전트가 똑똑해질수록 오히려 더 위험해집니다. 반대로 분리해 두면, 같은 자동화라도 재현성과 통제가 훨씬 좋아집니다.

## 1. agent skills는 “작업 방식”을 담는 포장지다

GitHub Docs는 agent skills를 프로젝트나 개인 범위에 둘 수 있다고 설명합니다. 프로젝트 스킬은 저장소 안의 `.github/skills`, `.claude/skills`, `.agents/skills`에 두고, 개인 스킬은 `~/.copilot/skills` 또는 `~/.agents/skills`에 둡니다. 핵심 파일 이름은 항상 `SKILL.md`입니다.

GitHub가 밝힌 정의도 명확합니다. skills는 **instructions, scripts, resources**를 담는 폴더이고, GitHub Copilot은 물론 Claude Code, Cursor, Codex, Gemini CLI 같은 여러 host와 함께 쓸 수 있습니다. 즉, 특정 제품에 묶인 비밀 레시피가 아니라 재사용 가능한 작업 단위에 가깝습니다.

실전에서는 이렇게 시작하면 됩니다.

```text
.github/skills/
  release-note-checker/
    SKILL.md
    scripts/
      extract_changes.py
    resources/
      checklist.md
```

```md
---
name: release-note-checker
description: 변경 요약과 사용자 영향만 정리하는 릴리스 노트 보조 스킬
---

- 사실이 확실하지 않으면 추측하지 말 것
- 변경 요약, 영향 범위, 남은 확인 항목만 분리할 것
- 최종 답변은 5줄 이내로 먼저 제시할 것
```

이런 식으로 skills를 만들면, 에이전트에게 “무엇을 해야 하는지”를 매번 장황하게 설명하지 않아도 됩니다.

## 2. gh skill은 skills를 ‘설치 가능한 단위’로 바꾼다

GitHub는 2026년 4월 `gh skill`을 공개했습니다. GitHub CLI **v2.90.0 이상**에서 사용할 수 있고, skills를 **discover, install, manage, publish**하는 데 쓰입니다. 설치 예시는 공식 changelog에 그대로 나와 있습니다.

```bash
gh skill install github/awesome-copilot
gh skill install github/awesome-copilot documentation-writer
gh skill install github/awesome-copilot documentation-writer@v1.2.0
gh skill search mcp-apps
```

여기서 중요한 포인트는 두 가지입니다. 첫째, skills는 설치형 자산이기 때문에 버전 고정이 가능하다는 점입니다. 둘째, GitHub는 skills가 **prompt injection**이나 **malicious scripts**를 포함할 수 있다고 경고합니다. 그래서 설치 전에 `gh skill preview`로 내용을 먼저 확인하라고 권장합니다.

즉, agent skills는 편하지만 “무조건 믿어도 되는 매크로”는 아닙니다. 설치 가능한 자동화일수록, 공급망 관점의 검토가 필요합니다.

## 3. sandbox는 “에이전트가 실제로 움직이는 공간”이다

2026년 6월 GitHub는 Copilot이 **secure, isolated sandboxes** 안에서 실행될 수 있다고 발표했습니다. 로컬에서도, 클라우드에서도 가능합니다. 그리고 이 환경은 Copilot의 tool execution을 격리된 상태로 돌리며, 코드·도구·filesystem·network를 **사용자가 정의한 policies** 안에서 다룰 수 있게 합니다.

이 발표가 중요한 이유는 간단합니다. skills는 “어떻게 일할지”를 정하고, sandbox는 “어디까지 움직일지”를 제한하기 때문입니다. agent가 파일을 읽고, 명령을 실행하고, 수정안을 만들 수 있게 되면, 그다음 질문은 늘 같습니다.

- 읽기만 허용할 것인가?
- 임시 디렉터리만 쓸 것인가?
- 네트워크 접근이 꼭 필요한가?
- 수정은 어떤 브랜치에만 허용할 것인가?

이 네 가지를 먼저 정해 두지 않으면, 자동화는 빨라져도 안전하지 않습니다.

## 4. 내가 추천하는 최소 운영 방식

가장 단순하면서도 효과적인 구조는 이겁니다.

1. **Skill**로 작업 절차를 문서화한다.
2. `gh skill`로 설치 전에 내용을 미리 확인한다.
3. **Sandbox** 안에서만 tool execution을 허용한다.
4. 결과는 diff와 로그만 보고 사람이 최종 승인한다.

예를 들면 이런 식입니다.

```yaml
workflow:
  skill: release-note-checker
  execution:
    mode: sandboxed
    network: on-demand
    filesystem: workspace-only
  review:
    require_diff: true
    require_human_approval: true
```

이건 실제 제품 설정 파일이 아니라 운영 원칙을 적은 예시입니다. 하지만 방향은 분명합니다. 지식은 skill에, 실행은 sandbox에, 책임은 사람에게 남겨 두는 것. 이 세 층을 분리할수록 자동화는 더 신뢰할 수 있습니다.

## 결론: 자동화의 다음 단계는 “더 강한 에이전트”가 아니다

- agent skills는 반복 작업을 위한 portable instructions, scripts, resources다.
- `gh skill`은 skills를 설치·관리·버전 고정하는 CLI 레이어다.
- Copilot의 local/cloud sandboxes는 tool execution을 격리된 공간에서 돌린다.
- skills는 믿되, 설치 전에는 반드시 preview하고 검토해야 한다.
- 자동화의 품질은 성능보다 경계 설정에서 먼저 갈린다.

오늘 당장 할 일은 하나면 충분합니다. 자주 반복하는 작업 하나를 골라 `SKILL.md`로 정리하고, 그 작업을 sandbox 안에서만 돌려 보세요. 그 순간부터 AI 코딩은 “대화”가 아니라 “운영 가능한 시스템”에 가까워집니다.

## 참고 자료

- GitHub Docs: About agent skills
- GitHub Docs: Adding agent skills for GitHub Copilot
- GitHub Changelog: Manage agent skills with GitHub CLI
- GitHub Changelog: Cloud and local sandboxes for GitHub Copilot now in public preview
