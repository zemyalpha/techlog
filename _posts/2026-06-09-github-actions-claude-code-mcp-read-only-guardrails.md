---
title: "GitHub Actions에 AI 에이전트를 넣을 때: Claude Code와 MCP를 분리하는 안전한 설계"
date: "2026-06-09"
keywords: ["GitHub Actions", "Claude Code", "MCP", "read-only", "agentic workflows"]
lang: "ko"
description: "GitHub Agentic Workflows, Claude Code, MCP 보안 원칙을 묶어 읽기 전용 분석과 승인 기반 쓰기 작업을 분리하는 실전 설계를 정리한다."
---

# GitHub Actions에 AI 에이전트를 넣을 때: Claude Code와 MCP를 분리하는 안전한 설계

AI 에이전트가 코드베이스를 읽고, 이슈를 분류하고, 보고서를 만들고, 심지어 PR 초안까지 준비하는 흐름은 더 이상 실험실 이야기만은 아니다. GitHub는 Agentic Workflows를 통해 이런 작업을 GitHub Actions 안에서 돌리는 방향을 제시했고, Anthropic의 Claude Code는 "채팅형 자동완성"이 아니라 작업을 맡기는 에이전트형 도구로 포지셔닝하고 있다. 동시에 MCP 보안 문서는 로컬 서버가 사실상 강한 권한을 가진 실행 환경이 될 수 있다고 경고한다.

그래서 핵심은 하나다. **읽기, 판단, 쓰기를 한 프로세스에 섞지 말고, 역할을 분리해야 한다.** 이 글은 그 분리 원칙을 GitHub Actions, Claude Code, MCP 세 축으로 나눠 설명한다.

## 1) 왜 지금 이 분리가 중요한가

GitHub의 Agentic Workflows는 GitHub Actions에서 실행되는 의도 기반 워크플로우로 설명되며, 기본적으로 읽기 전용 권한을 사용하고, 쓰기 작업은 안전한 출력(safe outputs)을 통해 검토 가능한 GitHub 작업으로 제한한다. 즉, 에이전트가 마음대로 레포를 흔드는 것이 아니라, 사람이 검토할 수 있는 경로를 남겨두는 구조다.

Claude Code 역시 중요한 힌트를 준다. 제품 페이지는 이를 "agentic, not autocomplete"라고 설명한다. 다시 말해, 에이전트는 단순히 다음 줄을 예측하는 도구가 아니라, 빌드·테스트·수정·배포 같은 작업을 맡길 수 있는 실행 파트너다. 이런 도구를 쓸수록 더 중요한 건 능력이 아니라 경계선이다.

MCP 쪽은 더 분명하다. 공식 보안 문서는 새 로컬 MCP 서버를 연결할 때 **정확한 명령을 잘라내지 말고 보여주라**, **명시적 사용자 승인을 받아라**, **sudo나 rm -rf 같은 위험 패턴을 경고하라**, **파일 시스템과 네트워크 접근을 최소 권한으로 제한하라**고 권고한다. 한마디로, MCP 서버는 편리한 플러그인이 아니라 잠재적으로 강한 실행 권한을 가진 도구다.

## 2) 실전 아키텍처: 분석은 읽기 전용, 쓰기는 승인 후

가장 안정적인 패턴은 두 단계다.

1. **분석 단계**: GitHub Actions가 저장소를 읽고, 이슈·PR·로그를 수집해 Markdown 보고서를 만든다.
2. **쓰기 단계**: 사람이 보고서를 확인한 뒤, 별도 작업이 PR 생성 또는 댓글 작성 같은 제한된 작업만 수행한다.

이때 첫 단계의 권한은 최대한 좁혀야 한다. 아래처럼 시작하면 좋다.

```yaml
name: daily-repo-report
on:
  schedule:
    - cron: '0 1 * * *'
  workflow_dispatch:

permissions:
  contents: read
  issues: read
  pull-requests: read

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build report
        run: python scripts/build_repo_report.py > report.md
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: repo-report
          path: report.md
```

이 구조의 장점은 명확하다. 에이전트가 결과를 만들 수는 있지만, 바로 레포를 바꾸지 못한다. 인간이 아티팩트를 확인한 뒤, 다음 단계에서만 쓰기 권한을 열어준다.

## 3) Claude Code는 어디에 두는 게 좋은가

Claude Code는 코드 탐색, 의존성 추적, 파일 수정처럼 넓은 범위의 작업에 강하다. 하지만 GitHub Actions 런너 안에서 바로 실행하면 권한과 네트워크 범위가 넓어지기 쉽다. 그래서 나는 보통 다음처럼 둔다.

- **Actions**: 데이터 수집과 결과물 생성
- **Claude Code**: 로컬 또는 별도 작업공간에서 초안 작성과 리팩터링
- **MCP**: 꼭 필요한 도구만, 최소 권한으로 연결

로컬에서 검증할 때는 컨테이너로 경계를 한 번 더 치는 편이 낫다.

```bash
docker run --rm -it \
  --network none \
  -v "$PWD:/workspace" \
  -w /workspace \
  python:3.11-slim \
  python scripts/build_repo_report.py
```

이렇게 하면 네트워크가 없는 환경에서 결과를 먼저 확인할 수 있다. 에이전트가 외부로 새는 대신, 내부 산출물만 만든다.

## 4) MCP를 붙일 때 자주 놓치는 함정

MCP는 유용하지만, "연결만 하면 끝"이 아니다. 보안 문서가 강조하듯 로컬 서버는 클라이언트와 같은 권한으로 동작할 수 있고, 잘못 연결하면 명령 실행, 데이터 유출, 파일 손상으로 이어질 수 있다.

실무에서 특히 조심할 지점은 세 가지다.

- **명령 노출**: 사용자가 실행될 명령을 완전히 볼 수 있어야 한다.
- **권한 분리**: 파일 시스템, 네트워크, 민감 디렉터리는 기본 차단이 좋다.
- **승인 절차**: 위험한 동작은 "에이전트가 알아서"가 아니라 "사용자가 승인"해야 한다.

즉, MCP는 에이전트의 손발이 아니라, 통제 가능한 도구 상자여야 한다.

## 결론: 에이전트의 성능보다 중요한 건 경계선이다

오늘의 실전 요약은 간단하다.

- GitHub의 Agentic Workflows는 읽기 전용 기본값과 안전한 출력으로 제어를 우선한다.
- Claude Code는 채팅 보조가 아니라 작업을 맡기는 에이전트형 도구로 쓰는 편이 맞다.
- MCP는 강력하지만, 승인·샌드박스·권한 제한이 없으면 위험해진다.
- 가장 좋은 구조는 분석과 쓰기를 분리하고, 쓰기는 사람의 확인 후에만 열어두는 것이다.
- 자동화의 목표는 "더 많이 바꾸는 것"이 아니라 "안전하게 반복하는 것"이다.

다음 단계는 어렵지 않다. 지금 운영 중인 저장소 하나를 골라, 읽기 전용 리포트 워크플로우를 먼저 만든 뒤, 그 결과를 사람이 검토하는 승인 단계까지 붙여보자. 그 순간부터 에이전트는 장난감이 아니라 운영 도구가 된다.
