---
title: "에이전트 프레임워크는 권한 변경을 기록하지 않는다 — Partnership on AI가 밝힌 6개 텔레메트리 구멍과 OTel로 막는 법"
date: "2026-10-15"
keywords: ["에이전트 관측성", "AI 에이전트 보안", "OpenTelemetry", "Partnership on AI", "텔레메트리", "Claude Agent SDK", "agent observability"]
lang: "ko"
description: "Partnership on AI가 2026년 10월 발표한 에이전트 관측성 격차 보고서의 6개 텔레메트리 갭을 분해하고, OpenTelemetry GenAI 규격과 Anthropic의 네이티브 OTel 익스포트로 직접 메우는 코드를 소개한다."
---

# 에이전트 프레임워크는 권한 변경을 기록하지 않는다

AI 에이전트를 프로덕션에 넣는 팀이라면 누구나 비슷한 질문을 받는다. "이 에이전트가 어제 밤에 정확히 무엇을 했고, 누가 승인했으며, 왜 그렇게 행동했나?" 놀랍게도 2026년 10월 현재, 가장 널리 쓰이는 에이전트 프레임워크 대부분은 이 질문에 답할 기록을 남기지 않는다.

Partnership on AI(PAI)가 2026년 10월에 발표한 보고서 **「The Observability Gap in AI Agents: What to Fix Before Agents Scale」**는 널리 쓰이는 4개 에이전트 프레임워크 — OpenAI Agents SDK, Anthropic Claude Agent SDK, LangGraph, CrewAI — 를 6개 안전 모니터링 목표에 대해 평가했다. 결론은 단순하다. 프레임워크가 내보내는 텔레메트리에는 일관되게 비어 있는 구멍이 6군데 있고, 그 구멍 때문에 기업은 에이전트의 권한이 언제 어떻게 바뀌었는지, 왜 그렇게 행동했는지 신뢰성 있게 추적할 수 없다는 것이다.

이 글은 그 6개 갭이 무엇인지, 왜 위험한지, 그리고 OpenTelemetry(OTel) GenAI 규격과 2026년 10월 14일 공개 베타로 나온 Anthropic의 네이티브 OTel 익스포트로 팀이 지금 당장 메울 수 있는 부분이 어디까지인지를 다룬다.

## 보고서가 발견한 6개 텔레메트리 갭

PAI 평가의 출발점은 명확하다. 모니터링 시스템은 그 밑에 깔린 데이터만큼만 좋다. 에이전트가 고위험 환경에서 돌 때 기업이 실시간으로 탐지해야 하는 것 — 미승인 행동, 권한 남용, 행동 드리프트 — 은 전부 프레임워크가 어떤 기록을 내보내느냐에 달려 있다. 보고서는 4개 프레임워크가 일관되게 내보내지 않는 텔레메트리를 여섯 범주로 묶었다.

- **에이전트 정체성(persistent agent identity)** — 어떤 기업 식별자 아래에서 행동했는지 트레이스에 연결되는 안정적인 참조가 없다.
- **권한 모드 변경(permission mode changes)** — 세션 중간에 인간 승인 요구가 풀리는 순간을 기록하는 프레임워크는 Claude Agent SDK 하나뿐이다. 다른 프레임워크에서는 내부자든 프롬프트 인젝션이든, 인간 개입을 제거하는 변경이 아무 흔적도 남기지 않는다.
- **휴먼 개입 기록(human intervention)** — 리뷰가 왜 촉발됐는지, 누가 승인했고 그 사람의 권한은 무엇이며 무엇을 결정했는지가 기록되지 않는다. 보고서는 이 때문에 "의미 있는 감독"과 "도장 찍기"를 구분할 수 없다고 지적한다.
- **메모리 변경(memory changes)** — 에이전트가 쌓아온 메모리가 행동을 바꾸는데, 그 변화 시점의 텔레메트리가 없다. OTel GenAI 규격에는 이제 메모리 변경 이벤트가 정의되어 있지만, 보고서 평가 대상 4개 중 3개 프레임워크가 아직 이를 내보내지 않는다.
- **사고 과정(chain-of-thought)** — 에이전트가 어떻게 결정에 도달했는지에 대한 기록.
- **토큰 단위 로그 확률(token-level logprobs)** — 구조화된 텔레메트리로 아옓 내보내지지 않아, 모델의 (불완전하나마 유용한) 신뢰도 신호 하나를 잃는다.

여섯 갭 중 앞의 넷은 "에이전트가 무엇을 했는가"에 대한 결정론적 기록이라 프레임워크가 내보내기만 하면 된다. 반면 CoT와 logprobs는 모델 제공자가 공개해야 하는 신호라, 표준화는 시기상조라는 게 보고서의 판단이다.

## 왜 이 구멍이 위험한가: 드리프트는 행동보다 먼저 상태에서 시작된다

이 갭들이 실제로 어떻게 사고로 이어지는지 보여주는 것이 드리프트 감지 문제다. 행동 드리프트는 정의상 기준선에서의 이탈인데, 기준선을 세우려면 여러 실행에 걸친 텔레메트리가 필요하다. 그런데 프레임워크가 구조화된 입출력만 남기고 메모리·설정 변화를 남기지 않으면, 드리프트는 이미 상태에 스며든 뒤에야 겨우 행동으로 드러난다.

보고서가 인용한 연구에 따르면, 메모리를 가진 에이전트는 태스크를 거듭하며 메모리가 쌓일수록 안전하지 않은 행동의 비율이 올라가며, 그 위험은 에이전트가 만들어내는 결과물에 나타나기 전에 메모리 상태에서 먼저 감지할 수 있다. 즉 메모리 변경 텔레메트리는 결과물 기반 모니터링보다 한 발 앞선 경보인데, 대부분의 프레임워크가 바로 그 신호를 버리고 있는 것이다.

권한 모드 갭은 더 직설적이다. 세션 중간에 "인간 승인 필요" 모드가 풀리는 이벤트가 기록되지 않으면, 사후 감사는 물론 실시간 이상 탐지의 재료 자체가 없다. PAI는 Claude Agent SDK가 이미 권한 모드 변경 이벤트와 도구 사용 승인/거부 결정 이벤트를 내보낸다는 점에서, 이 필드들이 기술적으로 충분히 구현 가능하다는 근거로 제시한다.

## 지금 메우는 법 1: Anthropic 네이티브 OTel 익스포트 켜기

갭을 메우는 가장 빠른 길은 플랫폼이 이미 내보내는 트레이스를 받는 것이다. Anthropic은 2026년 10월 14일 Claude Managed Agents, Messages API, Claude SDK에서 OpenTelemetry 트레이스를 내보내는 기능을 공개 베타로 출시했다. 트레이스는 OTel GenAI 시맨틱 컨벤션을 따르므로 Grafana, Honeycomb, Datadog, Langfuse, LangSmith 등 기존 관측성 도구에서 다른 서비스의 트레이스 옆에 나란히 놓인다.

설정은 두 갈래다. Managed Agents와 Messages API는 Claude Console에서 트레이스 목적지(destination)를 추가하면 되고, Claude SDK는 별도 설정 없이 트레이스를 자동으로 익스포트한다. 공식 문서의 안내는 다음과 같다.

```
# Claude Console → Settings → OpenTelemetry
# destination 추가 (OTLP endpoint + 헤더)
# 대상: Managed Agents 세션, Messages API 요청
# SDK(claude-agent-sdk 등)는 애플리케이션 쪽에서 자동 익스포트
```

트레이스에는 각 에이전트 턴, 모델 호출, 도구 사용별로 지연 시간·토큰 수·식별자가 담긴다. Anthropic은 고객사 Ramp의 사례로, Langfuse 트레이스를 채점해 반복 실패를 찾아내는 내부 시스템 "Reflect"를 통해 일부 에이전트에서 토큰 사용량을 10~20%, 세션 시간을 30% 줄였다고 밝혔다(업체 발표 인용).

## 지금 메우는 법 2: 프레임워크가 안 남기는 이벤트를 직접 span으로 남기기

플랫폼 트레이스는 "무슨 일이 있었나"를 채우지만, 권한 변경·휴먼 개입·메모리 변경은 여전히 대부분 애플리케이션 몫이다. OTel로 이벤트를 남기는 최소한의 패턴은 다음과 같다.

```python
from opentelemetry import trace

tracer = trace.get_tracer("agent.governance")

def record_permission_change(agent_id: str, old_mode: str, new_mode: str,
                             actor: str):
    with tracer.start_as_current_span("agent.permission_mode_change") as span:
        span.set_attribute("agent.id", agent_id)
        span.set_attribute("permission.old_mode", old_mode)
        span.set_attribute("permission.new_mode", new_mode)
        span.set_attribute("permission.changed_by", actor)
        # 인간 승인 제거는 항상 경보 대상
        if "human" in old_mode and "human" not in new_mode:
            span.set_attribute("security.alert", True)

def record_human_intervention(agent_id: str, trigger: str,
                              reviewer: str, authority: str, decision: str):
    with tracer.start_as_current_span("agent.human_intervention") as span:
        span.set_attribute("agent.id", agent_id)
        span.set_attribute("review.trigger_reason", trigger)
        span.set_attribute("review.reviewer", reviewer)
        span.set_attribute("review.reviewer_authority", authority)
        span.set_attribute("review.decision", decision)
```

핵심은 PAI가 지적한 세 필드 — 왜 리뷰가 촉발됐는지, 누가 응답했고 그 사람의 권한은 무엇인지, 무엇을 결정했는지 — 를 하나의 이벤트로 묶는 것이다. 이 세 필드가 없으면 승인 로그는 승인자 이름만 남긴 도장에 불과하다. 이벤트를 OTel로 남기면 어느 백엔드에서든 "권한 모드 변경 → 직후 도구 호출" 같은 시퀀스를 규칙으로 탐지할 수 있다.

프레임워크 선택 시에도 이제 체크리스트가 생겼다. 트레이스를 OTel로 내보내는가, 권한 모드 변경 이벤트가 있는가, 휴먼 개입을 구조화해 기록하는가, 메모리 변경 이벤트를 내보내는가 — 네 질문 중 첫 번째는 쉽게 통과하지만 나머지는 아직 대부분 "직접 구현"이 답이다.

## 생태계는 이미 움직이고 있다

PAI의 권고는 규격과 프레임워크 양쪽을 겨냥한다. 결정론적 갭 대부분은 OTel GenAI 시맨틱 컨벤션 확장 — 검증 가능한 에이전트 기업 식별자, 세션 수준 권한 모드와 변경 이벤트, 휴먼 개입 필드 — 로 닫을 수 있고, 나머지는 프레임워크 개발자가 그 규격을 채택하면 된다.

같은 달 생태계의 움직임도 빠르다. Anthropic의 OTel 공개 베타(10월 14일)에 앞서 Google은 10월 8일 프로덕션 에이전트의 트레이토리를 주기적으로 훑어 실패를 군집화·검증하고 배포 시점 소스 스냅샷에 루트 원인을 정렬해 주는 "AQuA(Ambient Quality Agent)"를 공개했다. 관측성이 "트레이스를 보는 것"에서 "트레이스를 자동 진단하는 것"으로 넘어가는 중간 지점이다.

## 결론

- PAI 보고서는 4개 주요 에이전트 프레임워크에서 6개 텔레메트리 갭(정체성, 권한 모드 변경, 휴먼 개입, 메모리 변경, CoT, logprobs)을 발견했다.
- 권한 모드 변경을 기록하는 프레임워크는 Claude Agent SDK 하나뿐이며, 메모리 변경 이벤트는 규격에 정의됐음에도 4개 중 3개가 내보내지 않는다.
- 드리프트는 행동이 아니라 상태(메모리·설정)에서 먼저 시작되므로, 상태 변경 텔레메트리는 결과물 모니터링보다 앞선 경보다.
- 당장 할 수 있는 일: Anthropic 플랫폼을 쓴다면 OTel 익스포트 공개 베타를 켜고, 모든 스택에서 권한 변경·휴먼 개입 이벤트를 OTel span으로 직접 남긴다.
- 첫 단계는 크지 않아도 된다. 오늘 소개한 span 두 개 — `permission_mode_change`와 `human_intervention` — 를 승인 게이트에 심는 것만으로 감사 질문의 절반에 답할 수 있다.

**참고자료**: [Partnership on AI, The Observability Gap in AI Agents (2026-10)](https://partnershiponai.org/resource/the-observability-gap-in-ai-agents/) · [Anthropic, Agent observability on the Claude Platform (2026-10-14)](https://claude.com/resources/articles/agent-observability-on-the-claude-platform) · [Claude API OpenTelemetry 문서](https://platform.claude.com/docs/en/manage-claude/opentelemetry) · [Google, AQuA 발표 (2026-10-08)](https://developers.googleblog.com/the-outer-loop-insights-first-an-ambient-quality-agent-that-diagnoses-your-production-agent/)
