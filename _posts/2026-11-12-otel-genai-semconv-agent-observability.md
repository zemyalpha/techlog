---
title: "AI 에이전트 관측은 이제 표준으로 묶는다 — OpenTelemetry GenAI 규약과 Langfuse·Phoenix 실전 비교"
date: "2026-11-12"
keywords: ["OpenTelemetry", "GenAI 시맨틱 컨벤션", "AI 에이전트 옵저버빌리티", "Langfuse", "Arize Phoenix", "OpenInference"]
lang: "ko"
description: "AI 에이전트의 실패는 응답이 멀쩡해도 내부에서 일어난다. OpenTelemetry GenAI 시맨틱 컨벤션의 구조와 Langfuse·Phoenix 계측 실전 코드, 라이선스 함정까지 정리한다."
---

# AI 에이전트 관측은 이제 표준으로 묶는다 — OpenTelemetry GenAI 규약과 Langfuse·Phoenix 실전 비교

AI 에이전트는 정확한 답을 내놓고도 내부적으로 실패할 수 있다. 잘못된 도구를 호출했거나, 오래된 컨텍스트를 참조했거나, 같은 행동을 반복했거나, 승인 단계를 건너뛰었을 수 있다. 일반 애플리케이션 모니터링(APM)은 이런 요청을 "200 OK, 지연 시간 양호"로 기록한다. 즉, 정작 중요한 실패를 놓친다.

에이전트를 제대로 운영하려면 모델 호출 하나하나가 아니라 **실행 경로 전체** — 모델 호출, 검색, 도구 사용, 메모리, 상태 변경, 핸드오프 — 를 하나의 트레이스로 묶어 봐야 한다. 좋은 소식은 이걸 위한 계측 표준이 이미 존재하고, 도구 생태계도 빠르게 정리되고 있다는 것이다. 이 글에서는 OpenTelemetry GenAI 시맨틱 컨벤션의 현재 상태를 확인하고, 실제 코드로 계측하는 방법, 그리고 도구 선택 시 반드시 알아야 할 라이선스 함정까지 다룬다.

## 왜 일반 APM으로 에이전트를 볼 수 없는가

전통적인 APM이 보는 단위는 HTTP 요청이나 DB 쿼리다. 반면 에이전트 하나의 실행은 다음처럼 중첩된 구조다.

```
agent_run (사용자 목표 1개)
├── llm_inference (계획 수립)         ← 토큰 사용량, 모델명, 비용
├── retrieval (RAG 검색)              ← 쿼리, 검색 결과, 점수
├── execute_tool (외부 API 호출)       ← 입력, 출력, 에러
├── llm_inference (결과 종합)
└── agent_handoff (후속 처리 위임)
```

각 단계를 개별 로그로만 남기면 "3단계에서 검색된 문서가 오래돼서 최종 답이 틀렸다"는 인과 관계를 복원할 수 없다. span을 부모-자식으로 연결한 트레이스로 묶어야 비로소 "어느 단계에서 어떤 입력으로 무엇이 나왔나"를 한 화면에서 추적할 수 있다. 여기까지가 관측(observability)이고, 이 트레이스를 데이터셋으로 변환해 회귀 테스트와 평가(evals)에 재사용하는 것이 다음 단계다.

## OpenTelemetry GenAI 시맨틱 컨벤션 — 별도 리포로 분리된 표준

핵심부터 말하면, GenAI 관련 시맨틱 컨벤션은 이제 OpenTelemetry 본체 리포지토리에서 **별도 리포지토리(`open-telemetry/semantic-conventions-genai`)로 분리**되어 관리된다. 본체 문서의 GenAI 항목은 "Moved" 안내만 남고, 속성 레지스트리에서도 `gen_ai.*` 속성들이 전부 새 리포를 가리킨다.

분리된 리포가 다루는 범위는 생각보다 넓다:

- **클라이언트 스팬**: 추론(inference), 임베딩, 검색(retrieval), 도구 실행(execute tool), fetch response, memory
- **에이전트 스팬**: 에이전트 실행 단위 자체에 대한 규약
- **MCP(Model Context Protocol) 규약**: MCP 도구 호출에 대한 계측
- **A2A(Agent-to-Agent) 규약**: 에이전트 간 통신 계측
- **프로바이더별 규약**: OpenAI, Anthropic, AWS Bedrock, Azure AI

실제 속성을 몇 개만 뽑아보면 이런 식이다.

| 속성 | 의미 | 예시 |
|------|------|------|
| `gen_ai.operation.name` | 수행한 연산 종류 | `chat`, `fetch_response` |
| `gen_ai.request.model` | 요청에 사용된 모델 | `gpt-4o` |
| `gen_ai.agent.id` / `gen_ai.agent.name` | 에이전트 식별자 | `asst_5j66...` / `Math Tutor` |
| `gen_ai.conversation.id` | 대화(세션/스레드) 식별자 | `conv_5j66...` |
| `gen_ai.usage.input_tokens` | 입력 토큰 수 | `1500` |

스팬 이름 규칙은 스팬 종류별로 조금씩 다르다. 클라이언트 추론 스팬은 `{gen_ai.operation.name} {gen_ai.request.model}` 형태(예: `chat gpt-4o`)를 따르고, 응답 식별자가 붙는 `fetch_response` 같은 스팬은 고카디널리티 문제 때문에 `{gen_ai.operation.name}`만 사용한다. 스팬 종류는 `CLIENT`를 권장한다. 재시도가 자동으로 일어났다면 스팬 하나가 재시도를 포함한 논리적 연산 전체를 덮는 것이 규칙이다.

**단, 반드시 알아야 할 한계가 있다.** 이 규약들의 공식 상태는 아직 **Development(개발 단계)**다. Stable이 아니므로 속성 이름이나 의미가 버전업되며 바뀔 수 있다. "이미 굳어진 표준"이 아니라 "굳어가는 중인 표준"이라는 인식으로 접근하고, 계측 라이브러리 버전을 고정해서 쓰는 것이 안전하다.

## 실전 계측 — 파이썬 코드 15줄로 시작하기

계측 경로는 크게 두 가지다. OpenTelemetry 커뮤니티의 공식 인스트루먼테이션을 쓰거나, Arize가 관리하는 OpenInference(Apache-2.0) 계열을 쓰는 것. OpenInference는 "OpenTelemetry Instrumentation for AI Observability"를 표방하며, OpenAI·LangChain 등 주요 프레임워크용 패키지를 PyPI에서 제공한다.

**방법 1: OpenInference + Langfuse로 트레이스 보내기**

```python
# pip install openinference-instrumentation-openai langfuse opentelemetry-sdk
from openinference.instrumentation.openai import OpenAIInstrumentor
from langfuse.exporters import LangfuseExporter  # OTel 익스포터
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

import openai

tracer_provider = TracerProvider()
tracer_provider.add_span_processor(SimpleSpanProcessor(LangfuseExporter()))

OpenAIInstrumentor().instrument(tracer_provider=tracer_provider)

client = openai.OpenAI()
resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "서울 날씨 알려줘"}],
)
# 이 호출 하나가 gen_ai 스팬으로 기록되어 Langfuse UI에서 추적 가능
```

**방법 2: OpenTelemetry 공식 인스트루먼테이션 사용**

```python
# pip install opentelemetry-instrumentation-openai
from opentelemetry.instrumentation.openai import OpenAIInstrumentor

OpenAIInstrumentor().instrument()
# 이후 OTel 익스포터(Console, OTLP, Collector)로 원하는 백엔드에 전송
```

에이전트 프레임워크를 쓴다면 `openinference-instrumentation-langchain` 같은 프레임워크용 계측이 더 편하다. LLM 호출 하나가 아니라 도구 호출·체인 실행까지 트리로 묶어 준다.

규모가 커지면 앱에서 백엔드로 직접 보내지 말고 **OTel Collector를 경유**하라. 계측 대상 앱과 저장 백엔드가 분리되어 나중에 도구를 갈아타도 앱 코드를 다시 배포할 필요가 없다.

## 도구 선택 — 그리고 반드시 확인해야 할 라이선스 함정

트레이스를 저장하고 보여주는 도구는 크게 3류로 정리된다.

**Langfuse** — 현재 사실상의 오픈소스 대세. GitHub 스타 3.7만 개를 넘겼고, 코어는 MIT 라이선스(`ee/` 디렉토리의 엔터프라이즈 기능만 별도 라이선스)다. 셀프호스팅이 자유롭다. 참고로 2026년 초 **ClickHouse에 인수**되었는데, ClickHouse는 시리즈 D 4억 달러 투자 발표와 함께 Langfuse 인수를 공식 블로그로 발표했다. 오픈소스 정책 유지를 약속했지만, 회사 소유 구조가 바뀐 것은 사실이니 셀프호스팅 운영자라면 정책 변화를 주기적으로 확인하자.

**Arize Phoenix** — 관측과 평가(evals)를 함께 제공하는 강력한 옵션이지만, 여기에 함정이 있다. Phoenix의 라이선스는 **Elastic License 2.0(ELv2)**이다. 소스는 공개되어 있고 셀프호스팅은 자유롭지만, OSI 공식 오픈소스 정의에는 부합하지 않는다(관리형 서비스로 제공하는 것 등이 제한됨). "오픈소스니까 아무 제약 없다"고 가정하면 나중에 계약 문제가 생길 수 있다. 관리형이 필요하면 Arize AX를, 셀프호스팅이면 Phoenix OSS를 쓰는 구조다.

**LangSmith** — LangChain·LangGraph 팀이라면 가장 자연스러운 선택이지만 프로프라이어터리 SaaS다. 그 외에도 평가 중심의 Braintrust, AI 게이트웨이가 필요하면 Portkey, 기존 인프라 모니터링과의 상관 분석이 중요하면 Datadog이 각각 강점이 있다.

요약하면: **벤더 종속을 피하고 싶다면 계측은 OTel/OpenInference 표준으로 통일하고, 저장소만 상황에 맞게 골라라.** 계측이 표준을 따르면 도구는 언제든 갈아탈 수 있다.

## 주의사항 3가지

1. **규약은 아직 Development 단계다.** 버전을 고정하고, 업그레이드 시 속성 변경 내역을 확인하라.
2. **프롬프트와 응답 본문 기록은 선택이다.** 규약 문서 자체가 전체 콘텐츠를 속성에 기록하거나 외부 스토리지로 분리하는 옵션을 제공한다. PII가 섞이는 서비스라면 콘텐츠 마스킹 정책을 계측 단계에서부터 설계하라.
3. **카디널리티를 경계하라.** `gen_ai.conversation.id`처럼 고유값이 무한히 늘어나는 속성을 스팬 이름에 넣지 말 것. 규약 문서도 스팬 이름은 저카디널리티로 유지하라고 명시한다.

## 결론

- 에이전트 실패는 최종 응답이 멀쩡해도 내부에서 일어난다 — 트레이스로 실행 경로 전체를 묶어야 보인다
- GenAI 시맨틱 컨벤션은 별도 리포로 분리되어 있으며 MCP·A2A까지 다루지만, 아직 Development 단계다
- 계측은 `openinference-instrumentation-*` 또는 OTel 공식 인스트루먼테이션으로 15줄 안에 붙일 수 있다
- Langfuse(MIT 코어, ClickHouse 인수)와 Phoenix(ELv2 주의)는 라이선스 구조가 다르다 — 셀프호스팅/상업 제공 여부에 따라 선택하라
- 첫 단계는 간단하다: 오늘 에이전트 코드에 `OpenAIInstrumentor().instrument()` 한 줄을 붙이고 트레이스 하나를 직접 열어보는 것부터 시작하라

## 참고 자료

- OpenTelemetry GenAI 시맨틱 컨벤션 리포지토리: https://github.com/open-telemetry/semantic-conventions-genai
- OpenTelemetry GenAI 속성 레지스트리: https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/
- OpenInference: https://github.com/Arize-ai/openinference
- Langfuse "Langfuse joins ClickHouse": https://langfuse.com/blog/joining-clickhouse
- ClickHouse 인수 발표: https://clickhouse.com/blog/clickhouse-acquires-langfuse-open-source-llm-observability
- Arize 도구 비교 가이드: https://arize.com/blog/best-ai-observability-tools-for-autonomous-agents-in-2026/
