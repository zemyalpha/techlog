---
title: "Agent skills는 지식, sandbox는 실행: GitHub Copilot과 Claude Code를 같이 쓰는 법"
date: "2026-06-06"
keywords: ["agent skills", "Claude Code sandbox", "GitHub Copilot", "gh skill", "AI coding agents"]
lang: "ko"
description: "포터블 agent skills와 OS 수준 sandbox를 분리하면 AI 코딩 에이전트의 재사용성과 안전성을 동시에 올릴 수 있다. GitHub CLI와 Claude Code 기준으로 실전 루틴을 정리한다."
---

# Agent skills는 지식, sandbox는 실행: GitHub Copilot과 Claude Code를 같이 쓰는 법

요즘 AI 코딩 도구를 보면 공통점이 있다. 모델이 똑똑해지는 것보다, **지식을 패키징하는 방식**과 **실행을 격리하는 방식**이 더 중요해졌다는 점이다. GitHub는 `gh skill`로 agent skills를 GitHub CLI에서 설치·관리·배포할 수 있게 했고, Anthropic은 Claude Code에 OS 수준 sandbox를 붙여 파일시스템과 네트워크를 제한한다. 한쪽은 “무엇을 할지”를 표준화하고, 다른 쪽은 “어디까지 할 수 있는지”를 제한한다.

이 둘을 같이 보면 에이전트 운영의 답이 꽤 선명해진다. **skills는 재사용 가능한 작업 지식**, **sandbox는 사고를 막는 실행 경계**다. 둘을 섞어버리면 프롬프트만 길어지고 통제는 약해진다.

## 1) agent skills는 프롬프트가 아니라 배포 가능한 지식이다

GitHub Changelog에 따르면 `gh skill`은 GitHub 저장소에서 agent skills를 **discover, install, manage, publish**하는 새 명령이다. 또 agent skills는 instructions, scripts, resources로 이루어진 포터블 묶음이며, GitHub Copilot, Claude Code, Cursor, Codex, Gemini CLI 같은 여러 host에서 동작한다. Visual Studio의 2026년 5월 업데이트에는 workspace와 user profile에서 발견된 skill을 보여주는 Skills 패널도 들어갔다. 즉, skills는 더 이상 “누가 채팅창에 적어 둔 요령”이 아니라, IDE와 CLI 양쪽에서 재사용되는 자산이 됐다.

실전에서는 이렇게 시작하면 된다.

```bash
# GitHub CLI는 v2.90.0 이상으로 업데이트
gh skill search mcp-apps

# Claude Code host에 user scope로 설치
gh skill install github/awesome-copilot documentation-writer --agent claude-code --scope user

# 특정 버전에 고정해서 설치
gh skill install github/awesome-copilot documentation-writer@v1.2.0 --pin
```

여기서 중요한 포인트는 버전 고정이다. skills는 실행 가능한 지식이기 때문에, 조용히 바뀌면 바로 공급망 리스크가 된다. “어제 잘 되던 에이전트가 오늘 이상하다”는 문제의 상당수는 모델이 아니라 skill 변경에서 시작한다.

## 2) Claude Code의 sandbox는 권한 프롬프트를 줄이는 기능이 아니다

Claude Code 문서는 sandboxed Bash tool이 OS 프리미티브를 이용해 Bash 명령의 **filesystem과 network access를 제한**한다고 설명한다. macOS에서는 Seatbelt, Linux와 WSL2에서는 bubblewrap을 사용한다. 기본 동작은 작업 디렉터리에 쓰기를 허용하고, **새 네트워크 도메인이 필요할 때 처음 한 번 묻는 것**이다. Anthropic의 엔지니어링 글도 같은 핵심을 강조한다. filesystem isolation만으로는 부족하고, network isolation까지 함께 있어야 SSH 키 유출이나 외부 서버로의 호출을 막을 수 있다.

그래서 `/sandbox`는 “귀찮은 승인 창을 줄이는 옵션”이 아니라, **에이전트가 어디까지 닿을 수 있는지 정하는 경계선**이다.

```text
/sandbox
```

이 경계를 켜 두면, 에이전트가 코드를 수정하더라도 작업 디렉터리 밖으로 쉽게 벗어나지 못하고, 처음 보는 네트워크 목적지는 다시 확인을 요구한다. 실무에서는 이 한 가지가 로그 신뢰성과 재현성을 크게 올린다.

## 3) 실전 조합은 간단하다: skill로 방법을 넣고, sandbox로 범위를 묶는다

내가 권하는 운영 순서는 이렇다.

1. `gh skill`로 작업별 skill을 설치한다.
2. `--pin`으로 버전을 고정한다.
3. Claude Code에서는 `/sandbox`를 먼저 켠다.
4. 에이전트가 수정한 결과를 사람이나 CI가 다시 검증한다.

이 패턴의 장점은 분명하다. skills는 팀의 “일하는 방식”을 코드처럼 배포하게 해 주고, sandbox는 그 일이 다른 곳으로 새지 않게 막아 준다. 다시 말해, **지식의 재현성과 실행의 안전성**을 분리하는 셈이다.

## 결론: 에이전트 운영의 핵심은 똑똑함보다 경계다

- `gh skill`은 agent skills를 설치·관리·배포하는 표준 진입점이다.
- agent skills는 GitHub Copilot, Claude Code, Cursor, Codex, Gemini CLI 등 여러 host에서 재사용된다.
- Claude Code sandbox는 Seatbelt와 bubblewrap 위에서 filesystem/network를 제한한다.
- 기본 작업 디렉터리 쓰기 허용과 새 도메인 승인 덕분에, 자동화와 통제가 같이 간다.
- 가장 안전한 조합은 “skill은 핀으로 고정하고, 실행은 sandbox로 감싼 뒤, 결과는 다시 검증”하는 방식이다.

오늘 바로 할 일은 하나다. 지금 쓰는 에이전트 작업 하나를 골라서, 그 작업이 **지식 문제인지, 실행 문제인지** 나눠 보자. 지식이면 skill로 패키징하고, 실행이면 sandbox로 가두면 된다.
