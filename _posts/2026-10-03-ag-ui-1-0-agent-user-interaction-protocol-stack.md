---
title: "에이전트와 UI 사이의 마지막 빈 자리가 채워졌다 — AG-UI 1.0 스테이블 스펙 설계와 마이그레이션 실전"
date: "2026-10-03"
keywords: ["AG-UI", "에이전트 프로토콜", "MCP", "A2A", "CopilotKit", "SSE", "AI 에이전트 UI"]
lang: "ko"
description: "2026년 9월 30일 AG-UI 1.0이 스키마 락 stable 스펙으로 확정됐다. MCP-A2A-AG-UI 3층 프로토콜 스택의 완성 의미, 인터럽트·서브에이전트 이벤트 설계, 그리고 실전 마이그레이션 체크리스트를 정리한다."
---

# 에이전트와 UI 사이의 마지막 빈 자리가 채워졌다 — AG-UI 1.0 스테이블 스펙 설계와 마이그레이션 실전

에이전트 백엔드를 만드는 일은 어려워졌지만, 정작 프론트엔드와 붙이는 일은 여전히 재량에 맡겨져 있었다. 토큰을 스트리밍하고, 도구 호출을 중간에 보여주고, 상태를 동기화하는 코드는 프레임워크마다 전부 달랐고, LangGraph로 짠 에이전트를 React 화면에 붙이는 방식과 Mastra로 짠 에이전트를 붙이는 방식은 서로 호환되지 않았다. 에이전트 하나를 다른 프레임워크로 갈아끼우면 UI 계층의 상당 부분을 다시 짜야 했다.

2026년 9월 30일, 이 자리를 채우는 스펙이 확정됐다. CopilotKit 팀이 주도해온 **AG-UI(Agent-User Interaction Protocol) 1.0**이 스키마 락(stable) 버전으로 발표된 것. 발표문에 따르면 Google, Microsoft, Amazon, Oracle이 채택했고 LangChain, Google ADK, OpenAI Agents SDK, AWS Strands, Microsoft Agent Framework, Pydantic AI, Mastra 등 주요 에이전트 프레임워크가 지원 목록에 들어가 있다. GitHub 리포에서는 Oracle Agent Spec과 Amazon Bedrock AgentCore가 퍼스트파티 지원으로 명시돼 있다.

이 글에서는 AG-UI 1.0이 정확히 무엇을 표준화했는지, 기존 0.x에서 무엇이 바뀌는지, 그리고 실제 코드 레벨에서 어떻게 다루는지를 정리한다.

## 1. 3층 프로토콜 스택: MCP, A2A, 그리고 AG-UI

2026년 내내 에이전트 프로토콜 생태계는 세 층으로 정리되어 왔다.

- **MCP (Model Context Protocol)** — 에이전트와 도구(툴)를 연결한다. 파일 시스템, 데이터베이스, 외부 API에 접근하는 표준 인터페이스.
- **A2A (Agent-to-Agent)** — 에이전트끼리 협업하고 메시지를 주고받는 계층.
- **AG-UI** — 에이전트와 사용자가 보는 애플리케이션 사이의 양방향 연결.

앞의 두 층은 이미 버전이 붙은 운영급 스펙으로 자리 잡았지만, 세 번째 층은 그때까지 0.x 버전이었다. "스펙에는 이렇게 돼 있는데 SDK는 저렇게 동작한다"는 불일치가 실제 제품에 치명적이었기 때문에, 많은 팀이 프로덕션 도입을 미루고 있었다. AG-UI 1.0의 핵심 선언은 바로 이것이다: **이제 스펙이 스키마로 락됐고, SDK는 그 스키마에서 직접 생성된다.**

이 시점에 프로토콜 스택의 수렴은 AG-UI 하나로 끝나지 않는다. 같은 주에 일어난 두 사건이 같은 방향을 가리킨다.

**Pi 1.0의 방향 전환.** 터미널 코딩 에이전트 Pi는 10월 1일 1.0을 내면서 MCP 지원을 기본으로 탑재했다. Pi의 개발자 마리오 제크너(Mario Zechner)는 2025년 11월 "What if you don't need MCP?"라는 글로 MCP 불필요론을 폈던 당사자다. Pi를 인수한 이어렌딜(Earendil, 아르민 로나처와 콜린 데이먼드 해나가 설립)은 "You said no, MCP!"라는 제목의 글로 입장을 공식 철회했다. 이유는 단순했다. MCP 자체가 좋아진 것도 있지만, MCP 통신에 필요한 샌드박스·인터프리터 계층(코드모드)이 다른 기능에도 그대로 쓸모 있었다는 것. 프로토콜 생태계가 성숙하면 "내 프레임워크만의 방식"을 고수할 실익이 사라진다는 신호다.

**Microsoft Agent Framework 1.20.0.** Microsoft는 10월 2일 파이썬 라이브러리 1.20.0을 내면서 Foundry 호스팅 재설계, DuckDB·SQL Server 벡터 스토어 커넥터, 세션 단위 파일 접근 격리를 추가했다. 대형 플랫폼 벤더조차 자체 규격이 아니라 열린 프로토콜(MCP, AG-UI) 위에 기능을 얹는 쪽으로 이동하고 있다.

## 2. AG-UI 1.0의 설계: 스키마 퍼스트, 그리고 두 가지 신기능

### 스키마 퍼스트 생성

1.0의 가장 중요한 아키텍처 변경은 모든 이벤트 타입이 JSON Schema로 정의되고, TypeScript·Python·.NET SDK가 그 스키마에서 직접 생성된다는 점이다. 사람이 스펙 문서를 따라 SDK를 손으로 만들던 시기의 "스펙-구현 불일치" 계열 버그가 구조적으로 제거된다. .NET SDK는 Microsoft가 기여했고 프로토콜 1.0과 함께 NuGet에 1.0으로 올라왔다.

### 첫 번째 신기능: 인터럽트(Interrupt)

1.0에서 프로덕션 관점으로 가장 중요한 추가는 1급 인터럽트 지원이다. 에이전트가 실행 중간에 멈추고, 선택지·폼·승인 요청 같은 구조화된 프롬프트를 프론트엔드에 밀어준 뒤 사용자 응답을 기다린다. 재개(resume) 호출은 `interruptId`, `status`(resolved 또는 cancelled), 선택적 `payload`를 실어 나르고 실행이 이어진다.

왜 중요한가. 배포 실행, 금전 작업, 데이터 변경 같은 민감한 오퍼레이션은 지금까지 폴링 루프 위에 어거지로 승인 절차를 얹는 방식이 일반적이었다. AG-UI 1.0은 이 휴먼 인 더 루프 패턴을 스펙 자체에 넣었다. Mastra도 9월 30일 `@ag-ui/mastra@1.1.5`를 내면서 네이티브 툴 승인을 AG-UI 인터럽트로 자동 노출하고 있다.

### 두 번째 신기능: 서브에이전트 이벤트

부모 에이전트가 자식 에이전트에게 작업을 위임할 때, 프론트엔드가 전체 이벤트 트리를 받아 서브에이전트별 진행 상황을 따로 렌더링할 수 있게 됐다. 이전까지는 멀티에이전트 UI를 만들려면 커스텀 이벤트를 발명하거나 사이드 스테이트를 따로 유지해야 했다.

## 3. 실전: 이벤트 스트림은 어떻게 흐르는가

AG-UI의 기본 전제는 단순하다. 에이전트 백엔드는 실행 중에 표준 이벤트를 발생시키고, 어떤 클라이언트든 그 이벤트를 읽을 수 있다. 전송 계층은 SSE, 웹소켓, 웹훅 어느 것이든 붙을 수 있는 미들웨어 구조다.

가장 단순한 실행 흐름은 대략 이렇다:

```text
RUN_STARTED
  ├─ TEXT_MESSAGE_START        (메시지 시작)
  ├─ TEXT_MESSAGE_CONTENT      (토큰 단위 스트리밍)
  ├─ TEXT_MESSAGE_END
  ├─ TOOL_CALL_START / END     (도구 호출 가시화)
  ├─ STATE_DELTA               (JSON Patch 기반 상태 동기화)
  └─ RUN_FINISHED
```

도구 호출이 UI에 실시간으로 보이고, 상태 변경이 JSON Patch 델타로 전달된다는 점이 핵심이다. 프론트엔드는 매초 전체 상태를 다시 받는 게 아니라 패치를 적용하기만 하면 된다.

React 계열에서는 CopilotKit이 제공하는 AG-UI 클라이언트로 이벤트를 구독한다. 공식 문서 기준 패턴은 다음과 같다:

```javascript
import { useEffect } from "react";
import { useAgent } from "@copilotkit/react-core/v2";

function EventLog() {
  const { agent } = useAgent({ agentId: "research-agent" });

  useEffect(() => {
    const subscription = agent.subscribe({
      onTextMessageContentEvent({ textMessageBuffer }) {
        console.log("스트리밍 텍스트:", textMessageBuffer);
      },
      onToolCallEndEvent({ toolCallName, toolCallArgs }) {
        console.log("도구 호출 완료:", toolCallName, toolCallArgs);
      },
      onStateChanged({ agent }) {
        console.log("상태 변경:", agent.state);
      },
    });
    return () => subscription.unsubscribe();
  }, [agent]);
}
```

주목할 점은 콜백 이름이 AG-UI 이벤트 타입에 그대로 대응한다는 것(`onStateDeltaEvent`, `onCustomEvent` 등). 프레임워크가 LangGraph든 Mastra든, 프론트엔드가 보는 계약은 동일하다. 같은 에이전트를 웹, 모바일, 터미널, 슬랙·텔레그램 봇 등 어떤 표면에서든 띄울 수 있는 근거가 여기 있다.

백엔드 쪽에서는 프레임워크 브리지가 자기 네이티브 출력을 AG-UI 이벤트로 변환해 엔드포인트에서 스트리밍한다. 즉 LangGraph 에이전트를 새로 짤 때 AG-UI "용" 코드를 따로 짜는 게 아니라, 기존 에이전트에 브리지를 얹는 것으로 끝난다.

## 4. 0.x에서 1.0으로: 마이그레이션 체크리스트

AG-UI 1.0은 하위 호환을 유지한다. 0.x 에이전트는 1.0 클라이언트와, 그 반대도 함께 동작하므로 양쪽을 따로 업그레이드할 수 있다. 다만 마이너한 브레이킹 체인지는 있다:

- **TypeScript**: 선택적 프로토콜 필드는 `null`이 아니라 아예 없어야 한다. `rawEvent: null` 같은 코드는 제거할 것.
- **커스텀 필드**: 이벤트에 붙이던 커스텀 필드는 `metadata`로 이동.
- **`SubAgentInfo` → `SubagentInfo`**: 대소문자 변경. 문자열로 타입을 비교하는 코드가 있다면 깨진다.
- **Python**: JSON Patch 엔트리가 dict에서 타입 객체로 바뀌었다. `patch["path"]`는 `patch.path`로.

공식 마이그레이션 가이드는 세 SDK(TypeScript, Python, .NET)를 모두 다루고 있으며, 대부분의 기존 코드베이스는 열 개 남짓의 수정으로 끝난다고 밝히고 있다. 실무 권장사항을 정리하면:

1. 의존성에 `ag-ui-protocol >= 1.0.0`을 고정한다.
2. 커스텀 이벤트 필드를 전부 `metadata`로 옮긴다.
3. 민감한 도구(배포, 결제, 데이터 변경)에 인터럽트 핸들러를 붙여 승인 게이트를 만든다.
4. 멀티에이전트 UI라면 서브에이전트 이벤트로 사이드 스테이트 유지 코드를 걷어낸다.

## 5. 이름 비슷한 것들 조심하기

프로토콜 이름이 알파벳 조합이라 헷갈리기 쉬운데, 정리하면 이렇다.

- **A2UI** — Google의 제너레이티브 UI 명세. 에이전트가 UI 위젯을 전달하는 방식을 정의한다. AG-UI와 경쟁 관계가 아니라 함께 쓰는 것.
- **MCP Apps** — MCP 진영의 제너레이티브 UI 확장. AG-UI는 이를 지원 대상으로 명시하고 있다.
- **A2A** — 에이전트 간 통신. AG-UI와의 통합도 공식 지원 목록에 있다.

"에이전트→도구(MCP), 에이전트→에이전트(A2A), 에이전트→사용자(AG-UI)"라는 세 축만 기억하면 된다.

## 결론

AG-UI 1.0의 의미를 한 문장으로 압축하면, **에이전트 프로토콜 스택의 마지막 미완성 층이 버전이 붙은 스펙으로 닫혔다**는 것이다. 정리하면:

- AG-UI 1.0은 2026년 9월 30일 스키마 락 stable로 발표됐고, 모든 이벤트가 JSON Schema로 정의되며 SDK가 스키마에서 생성된다.
- 1급 인터럽트 지원으로 휴먼 인 더 루프 승인이 스펙 내장이 됐다. 서브에이전트 이벤트로 멀티에이전트 UI가 표준화됐다.
- 하위 호환은 유지되며, 마이그레이션은 대부분 열 개 남짓의 수정으로 끝난다(커스텀 필드 `metadata` 이동, `SubagentInfo`改名, Python 패치 객체화).
- Pi 1.0의 MCP 기본 채택과 Microsoft Agent Framework 1.20의 프로토콜 수렴은 같은 흐름의 방증이다. 자체 스트리밍 규격을 만들 시대는 끝났다.

당장 취할 수 있는 첫 단계는 간단하다. 기존 에이전트의 커스텀 스트리밍 코드가 어느 정도인지 세어보고, 프레임워크의 AG-UI 브리지로 교체 가능한지 확인하는 것. GitHub 리포(ag-ui-protocol/ag-ui)에 지원 프레임워크별 동작 예제가 준비돼 있으니, 자기가 쓰는 스택의 예제부터 열어보길 권한다.
