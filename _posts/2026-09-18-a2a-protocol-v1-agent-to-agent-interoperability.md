---
title: "A2A 프로토콜 v1.0 완전 해부 — 에이전트끼리 대화하는 표준은 MCP와 무엇이 다른가"
date: "2026-09-18"
keywords: ["A2A 프로토콜", "Agent2Agent", "MCP", "에이전트 상호운용성", "멀티 에이전트", "Agent Card"]
lang: "ko"
description: "Google이 만들고 Linux Foundation이 키운 A2A 프로토콜 v1.0의 핵심 개념, MCP와의 차이, Agent Card·Task 모델, 실전 코드와 생태계 현황을 정리한다."
---

# A2A 프로토콜 v1.0 완전 해부 — 에이전트끼리 대화하는 표준은 MCP와 무엇이 다른가

AI 에이전트가 실무에 들어오면서 생긴 가장 큰 골칫거리는 지능이 아니라 **연결**이다. LangGraph로 만든 에이전트와 CrewAI로 만든 에이전트, 우리 회사 서버에 있는 에이전트와 협력사 클라우드에 있는 에이전트가 일을 시키려면 매번 커스텀 통합 코드를 짜야 했다. 쌍(pair)이 늘어날 때마다 통합 비용이 선형으로 늘어나는 구조다.

A2A(Agent2Agent) 프로토콜은 이 문제를 푸는 개방형 표준이다. Google이 2025년 4월 처음 발표했고, 같은 해 6월 Linux Foundation에 기부되어 중립 재단 산하에서 운영되고 있다. 2026년 4월 기준 지원 기관은 150개를 넘었고, Microsoft·AWS·Google 모두 자사 플랫폼에 A2A를 내장했다. 2026년 8월에는 Linux Foundation 산하의 Agentic AI Foundation(AAIF)에 Growth Stage 프로젝트로 편입되면서 MCP, goose, AGENTS.md와 같은 스택을 이루게 되었다.

이 글에서는 A2A가 무엇이고, 무엇이 아니며, 어떻게 쓰는지를 정리한다.

## A2A와 MCP는 경쟁 관계가 아니다

가장 흔한 오해부터 정리하자. A2A와 MCP(Model Context Protocol)는 같은 층위에서 경쟁하는 프로토콜이 아니다. 공식 문서는 이 둘을 이렇게 구분한다.

- **MCP = 수직 연결(에이전트 ↔ 도구)**: 하나의 에이전트가 GitHub 리포지토리, SQL 데이터베이스 같은 도구와 데이터에 붙는 방식을 표준화한다.
- **A2A = 수평 연결(에이전트 ↔ 에이전트)**: 서로 다른 프레임워크, 서로 다른 조직이 만든 독립적인 에이전트끼리 발견하고, 위임하고, 결과를 주고받는 방식을 표준화한다.

비유하자면 MCP가 "직원과 사내 시스템(ERP, 그룹웨어)을 잇는 API"라면, A2A는 "협력사와 주고받는 표준 거래 문서"에 가깝다. 실제 아키텍처에서는 한 에이전트가 MCP로 자기 도구에 붙고, A2A로 다른 에이전트와 협업하는 그림이 표준적인 사용법이다.

공식 문서가 명시한 "A2A가 아닌 것"도 짚을 가치가 있다.

1. LangGraph, CrewAI 같은 **에이전트 개발 프레임워크가 아니다** — 어느 프레임워크로 만든 에이전트든 통신 층으로 쓸 뿐이다.
2. **서브에이전트·도구 호출 프로토콜이 아니다** — 한 에이전트 내부의 하위 에이전트 통신은 각 프레임워크의 고유 기능이나 MCP가 담당한다.
3. **MCP의 대체재가 아니다** — 위에서 봤듯 보완 관계다.

## 핵심 개념 3가지: Agent Card, Task, 불투명성

A2A의 동작 원리는 세 가지 개념으로 요약된다.

**① Agent Card — 자기소개서.** 각 에이전트는 잘 알려진 경로(well-known URI)에 자신의 능력, 입출력 모달리티, 보안 스킴을 담은 JSON 메타데이터를 게시한다. 다른 에이전트는 이 카드를 읽고 "이 일을 이 에이전트에게 맡길 수 있는가"를 판단한다. v1.0에서는 **Signed Agent Card**가 도입되어 JWS(JSON Web Signature, RFC 7515)로 카드에 서명하고, 정규화는 RFC 8785(JCS)를 따른다. 즉 에이전트 신원을 암호학적으로 검증할 수 있게 됐다.

**② Task — 작업의 단위.** A2A의 통신 단위는 채팅 메시지가 아니라 작업이다. 작업은 `submitted → working → input-required → completed/failed` 상태를 가지며, 장시간 실행도 지원한다. 통신은 JSON-RPC 2.0 over HTTP(S)를 기반으로 하고, 동기 요청/응답, SSE 스트리밍, 비동기 푸시 알림 세 가지 상호작용 모드를 제공한다.

**③ 불투명성(opacity) — 내부를 안 보여준다.** A2A 에이전트는 협업할 때 내부 메모리, 도구 구현, 독점 로직을 노출하지 않는다. 이건 기술적 제약이 아니라 설계 철학이다. 지식재산을 보호하고, 공격 표면을 줄이며, 조직 경계를 넘는 협업을 가능하게 만든다.

## 실전: A2A 에이전트 서버 띄우기

Python SDK 기준으로 A2A 에이전트를 띄우는 코드는 공식 튜토리얼에서 가져왔다 (v1.0 기준).

```python
import uvicorn

from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.routes import (
    create_agent_card_routes,
    create_jsonrpc_routes,
)
from a2a.server.tasks import InMemoryTaskStore
from a2a.types import (
    AgentCapabilities,
    AgentCard,
    AgentInterface,
    AgentSkill,
)
from agent_executor import HelloWorldAgentExecutor
from starlette.applications import Starlette

skill = AgentSkill(
    id='echo_bot',
    name='Echo Bot',
    description='요청을 받아 인사를 반환하는 예제 에이전트',
    input_modes=['text/plain'],
    output_modes=['text/plain'],
    tags=['a2a', 'echo-example'],
)
```

v1.0을 쓴다면 이전 버전과의 차이를 반드시 알아야 한다. 메서드 이름이 dot 표기에서 카멜케이스로 바뀌었다 (`message/send` → `SendMessage`, `tasks/get` → `GetTask` 등). 또한 Part 타입 통합, 스트림 이벤트 판별자 패턴 변경, 에러 처리의 `google.rpc.Status` 표준화 같은 브레이킹 체인지가 있어서, v0.3 기반 코드는 마이그레이션이 필요하다. 공식 문서는 호환 레이어 → 듀얼 지원 → v1.0 단독의 3단계 전환 전략을 권장한다.

설치와 실행은 다음과 같다.

```bash
pip install a2a-sdk

# 공식 샘플 실행 (a2a-samples 리포지토리에서)
python samples/python/agents/helloworld/__main__.py
# INFO: Uvicorn running on http://127.0.0.1:9999
```

SDK는 Python 외에도 JavaScript(`@a2a-js/sdk`), Java, Go, .NET, Rust(`a2a-lf`)를 공식 지원한다. 1년 전만 해도 Python 하나뿐이었다는 점을 생각하면 생태계가 빠르게 성숙하고 있다.

## 1년 만에 프로덕션으로 간 생태계

A2A가 단순한 스펙 놀음으로 끝나지 않은 이유는 클라우드 플랫폼의 직접 내장이다.

- **Microsoft**: Azure AI Foundry와 Copilot Studio에 A2A 통합
- **AWS**: Amazon Bedrock AgentCore Runtime에서 지원
- **Google Cloud**: ADK(Agent Development Kit)를 통한 지원 — 원래 발표 주체이기도 하다

산업별로는 공급망, 금융, 보험, IT 운영에서 프로덕션 배포가 진행 중이며, ServiceNow·Salesforce·Atlassian·SAP 같은 엔터프라이즈 SaaS도 워크플로 연결에 A2A를 쓰고 있다. 결제 영역까지 확장 중인데, 에이전트 간 거래를 위한 **AP2(Agent Payments Protocol)** 확장에는 이미 60개 이상의 결제·금융 기관이 참여하고 있다.

개발자 생태계 쪽에서는 LangGraph, CrewAI, Pydantic AI, AG2, IBM BeeAI 등 주요 프레임워크가 A2A 지원을 갖췄다. GitHub 리포지토리 스타는 글 작성 시점 기준 약 2.5만 개다.

## 주의사항: 통합 전에 확인할 것

A2A를 도입 검토한다면 몇 가지를 점검하자.

- **버전을 명확히 하라.** v0.3와 v1.0은 호환되지 않는 지점이 많다. 새로 시작한다면 무조건 v1.0부터.
- **Agent Card의 신뢰 문제.** Signed Agent Card로 신원 검증은 되지만, 카드에 적힌 능력이 실제로 정직한지는 별개 문제다. 외부 조직의 에이전트와 통신할 때는 타임아웃·재시도·예산 상한을 반드시 설계에 넣어야 한다.
- **거버넌스 변화를 따라가라.** 2026년 8월 AAIF 편입으로 로드맵 주체가 재정리되었다. 중요한 결정을 스펙에 묶는다면 상향 버전 정책과 확장(extension) 거버넌스 문서를 확인하자.
- **내부 서브에이전트에는 쓰지 마라.** 한 시스템 안의 오케스트레이션은 프레임워크 고유 기능이 더 단순하다. A2A의 가치는 조직·벤더 경계를 넘을 때 나온다.

## 결론

- A2A는 에이전트-에이전트 수평 통신, MCP는 에이전트-도구 수직 통신. 둘은 함께 쓰는 것이다.
- v1.0에서 메서드 체계, 타입 안전성, Signed Agent Card, 멀티테넌시가 정리되어 프로덕션 수준이 됐다.
- 150개 이상 조직과 3대 클라우드의 직접 내장으로 사실상의 표준 경쟁에서 앞서 나가고 있다.
- 도입은 "다른 팀/다른 회사의 에이전트와 협업이 필요한 지점"부터 시작하는 게 합리적이다.

당장 해볼 수 있는 첫 걸음은 `pip install a2a-sdk` 후 공식 helloworld 샘플을 띄워보고, 브라우저에서 `http://127.0.0.1:9999/.well-known/agent-card.json`(에이전트 카드)을 열어보는 것이다. 에이전트의 자기소개서가 어떻게 생겼는지 직접 눈으로 확인하는 것만으로도 이 프로토콜의 설계 철학이 감이 온다.

## 참고 자료

- A2A Protocol 공식 문서: https://a2a-protocol.org/latest/
- What's New in v1.0: https://a2a-protocol.org/latest/whats-new-v1/
- Linux Foundation 보도자료 (2026-04-09): 150개 조직 돌파 및 v1.0 발표
- Google Developers Blog (2025-06-23): Google Cloud donates A2A to Linux Foundation
- A2A GitHub 리포지토리: https://github.com/a2aproject/A2A
