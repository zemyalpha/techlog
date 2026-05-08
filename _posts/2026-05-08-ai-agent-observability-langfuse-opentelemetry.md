---
title: "AI 에이전트가 왜 그런 대답을 했는지 추적하는 법: Langfuse와 OpenTelemetry로 옵저버빌리티 구축하기"
date: "2026-05-08"
keywords: ["AI 에이전트", "옵저버빌리티", "Langfuse", "OpenTelemetry", "트레이싱", "LLM 디버깅"]
lang: "ko"
description: "프로덕션 AI 에이전트의 비결정적 실패를 디버깅하기 위한 옵저버빌리티 전략을 Langfuse와 OpenTelemetry를 중심으로 실전 코드와 함께 설명합니다."
---

# AI 에이전트가 왜 그런 대답을 했는지 추적하는 법: Langfuse와 OpenTelemetry로 옵저버빌리티 구축하기

프로덕션에 AI 에이전트를 배포한 경험이 있다면 이런 상황을 겪었을 것이다. 사용자가 "회의 요약해 줘"라고 했는데, 에이전트가 아주 그럴듯한 문장으로 답했다. 그런데 자세히 보니 회의에 없었던 사람 이름이 들어 있고, 결정된 적 없는 사항이 포함되어 있으며, 날짜까지 틀렸다. 겉보기엔 완벽해 보이지만 전부 거짓이다.

로그를 확인해 보니 프롬프트도 있고 최종 응답도 있다. 하지만 그 사이에 일어난 16번의 도구 호출, 3번의 재시도, 더 싼 모델로의 자동 폴백, 컨텍스트가 잘린 지점 두 곳은 어디에도 기록되어 있지 않다. 버그는 프롬프트에 있지 않았다. 에이전트가 거치는 중간 단계들을 관찰할 수 없었던 게 진짜 문제였다.

이 글에서는 AI 에이전트의 옵저버빌리티를 구축하는 실전 방법을 다룬다. Langfuse, OpenTelemetry, 그리고 이 둘을 연결하는 구체적인 코드까지 함께 살펴보자.

## 에이전트 옵저버빌리티가 일반 로깅과 다른 점

전통적인 웹 서비스는 요청이 들어오면 정해진 코드 경로를 따라 응답이 나간다. 입력과 출력이 1:1로 매핑되고, 예외가 발생하면 스택 트레이스가 알려준다. 하지만 AI 에이전트는 다르다.

에이전트 하나의 요청 안에는 LLM 호출, 도구 사용, 조건 분기, 상태 전이, 메모리 읽기/쓰기가 뒤섞인다. 같은 질문을 넣어도 매번 다른 경로를 탈 수 있다. 이런 비결정적 시스템에서는 "어떤 경로를 거쳤는지"를 아는 게 "결과가 뭐였는지" 아는 것만큼 중요하다.

에이전트 옵저버빌리티가 해결하려는 문제는 다음과 같다:

- **경로 추적**: 에이전트가 어떤 도구를 호출했고, 각 단계에서 어떤 결정을 내렸는가
- **원인 분석**: 최종 응답이 틀렸을 때, 어느 단계에서 잘못되었는가
- **비용 추적**: 토큰 사용량, 모델별 비용, 도구 호출 횟수는 얼마인가
- **품질 측정**: 실제 사용 데이터를 바탕으로 에이전트 성능을 평가하는가

## 핵심 개념: 트레이스, 스팬, 세션

옵저버빌리티를 구축하려면 먼저 데이터 모델을 이해해야 한다. OpenTelemetry의 개념을 LLM 세계에 맞게 변형한 구조다.

**트레이스(Trace)**: 사용자의 하나의 요청에서 시작된 전체 실행 흐름. 하나의 트레이스 안에는 여러 개의 스팬이 포함된다.

**스팬(Span)**: 트레이스 안의 개별 단계. LLM 호출 한 번, 도구 호출 한 번, 검색 한 번 각각이 스팬이 된다. 각 스팬에는 입력, 출력, 소요 시간, 메타데이터가 기록된다.

**세션(Session)**: 여러 트레이스를 묶는 단위. 대화형 에이전트라면 하나의 대화 세션이 여러 턴(각 턴이 하나의 트레이스)으로 구성된다.

이 구조를 시각화하면 다음과 같다:

```
Session: 사용자 대화 #42
├── Trace: 턴 1 "회의 요약해 줘"
│   ├── Span: LLM 호출 (GPT-4o)
│   ├── Span: 도구 호출 - calendar_lookup
│   ├── Span: 도구 호출 - document_search
│   └── Span: LLM 호출 (최종 응답 생성)
├── Trace: 턴 2 "거기서 누가 발표했어?"
│   ├── Span: LLM 호출
│   ├── Span: 도구 호출 - context_lookup
│   └── Span: LLM 호출
```

## Langfuse로 트레이싱 시작하기

Langfuse는 LLM 애플리케이션을 위한 오픈소스 옵저버빌리티 플랫폼이다. 트레이싱, 평가(Evaluation), 프롬프트 관리, 비용 추적을 한 곳에서 제공한다. 직접 호스팅할 수도 있고 클라우드 버전을 쓸 수도 있다.

### 기본 설정

```python
# pip install langfuse
from langfuse import Langfuse

langfuse = Langfuse(
    public_key="pk-...",
    secret_key="sk-...",
    host="https://cloud.langfuse.com"  # 또는 자체 호스팅 URL
)
```

### 에이전트 실행 추적하기

간단한 RAG 에이전트를 예로 들어 보자. 사용자 질문이 들어오면 검색하고, 컨텍스트와 함께 LLM을 호출하는 흐름이다.

```python
from langfuse.decorators import observe, langfuse_context

@observe(as_type="generation")
def call_llm(prompt: str, model: str = "gpt-4o") -> str:
    """LLM 호출을 추적하는 래퍼 함수"""
    # Langfuse가 자동으로 입력/출력/토큰/비용을 기록
    response = openai_client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content

@observe(as_type="span")
def search_documents(query: str, top_k: int = 5) -> list[dict]:
    """문서 검색 스팬"""
    # 검색 결과를 스팬의 출력으로 기록
    results = vector_store.similarity_search(query, k=top_k)
    return [{"content": r.page_content, "score": r.score} for r in results]

@observe(as_type="trace")
def answer_question(user_question: str) -> str:
    """전체 에이전트 실행을 하나의 트레이스로 추적"""
    # 1단계: 검색
    docs = search_documents(user_question)
    
    # 2단계: 컨텍스트 구성
    context = "\n".join([d["content"] for d in docs])
    prompt = f"""다음 문서를 참고해서 질문에 답해라.
    
문서:
{context}

질문: {user_question}"""

    # 3단계: LLM 호출
    answer = call_llm(prompt)
    
    # 커스텀 메타데이터 추가
    langfuse_context.update_current_trace(
        metadata={
            "user_question": user_question,
            "retrieved_doc_count": len(docs),
            "top_score": docs[0]["score"] if docs else 0,
        }
    )
    
    return answer
```

`@observe` 데코레이터만으로 각 함수의 입력, 출력, 실행 시간이 자동으로 기록된다. Langfuse 대시보드에서 트리 형태로 각 단계를 시각적으로 확인할 수 있다.

### 비용과 토큰 추적

LLM 호출 시 토큰 사용량을 명시적으로 기록하면 모델별 비용 추적이 가능하다:

```python
@observe(as_type="generation")
def call_llm_with_cost(prompt: str, model: str = "gpt-4o") -> str:
    response = openai_client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
    
    # 토큰 사용량을 Langfuse에 기록
    langfuse_context.update_current_observation(
        model=model,
        usage={
            "input": response.usage.prompt_tokens,
            "output": response.usage.completion_tokens,
            "total": response.usage.total_tokens,
        },
    )
    
    return response.choices[0].message.content
```

## OpenTelemetry와 연동하기: 표준의 힘

Langfuse SDK만으로도 충분하지만, 이미 OpenTelemetry를 사용 중인 인프라라면 OTLP(OpenTelemetry Protocol)를 통해 Langfuse로 트레이스를 직접 보낼 수 있다. Langfuse는 OTLP 백엔드로 동작한다.

### OpenTelemetry로 LLM 스팬 정의하기

OpenTelemetry의 GenAI 시맨틱 컨벤션(Semantic Conventions)은 LLM 호출을 표준화된 속성으로 기록하는 방법을 정의한다. 이 컨벤션에 따르면 LLM 스팬은 다음 속성을 포함한다:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Langfuse를 OTLP 백엔드로 설정
otlp_exporter = OTLPSpanExporter(
    endpoint="https://cloud.langfuse.com/api/public/otel",
    headers={
        "Authorization": "Basic <base64(pk:sk)>",
    },
)
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("my-agent")
```

### GenAI 시맨틱 컨벤션에 따른 스팬 생성

```python
from opentelemetry.semconv.attributes.error_attributes import ERROR_TYPE

TRACER = trace.get_tracer("agent-service")

def trace_llm_call(prompt: str, model: str, system_prompt: str = ""):
    with TRACER.start_as_current_span("gen_ai.chat.completion") as span:
        # GenAI 시맨틱 컨벤션 속성 설정
        span.set_attribute("gen_ai.system", "openai")
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.request.max_tokens", 4096)
        span.set_attribute("gen_ai.prompt", prompt[:1000])  # 토큰 제한
        
        try:
            response = openai_client.chat.completions.create(
                model=model,
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": prompt},
                ],
            )
            
            # 응답 메타데이터 기록
            span.set_attribute("gen_ai.response.finish_reason", 
                             response.choices[0].finish_reason)
            span.set_attribute("gen_ai.usage.input_tokens", 
                             response.usage.prompt_tokens)
            span.set_attribute("gen_ai.usage.output_tokens", 
                             response.usage.completion_tokens)
            
            return response.choices[0].message.content
            
        except Exception as e:
            span.set_attribute(ERROR_TYPE, type(e).__name__)
            span.set_status(trace.StatusCode.ERROR, str(e))
            raise
```

이렇게 하면 기존 인프라의 앱 성능 모니터링(APM)과 LLM 호출 트레이스를 하나의 대시보드에서 볼 수 있다. 데이터베이스 쿼리, API 호출, LLM 호출이 하나의 트레이스 안에 연결되어 전체 요청 흐름을 파악할 수 있다.

## 실전 팁: 트레이스 볼륨 관리와 샘플링

에이전트가 프로덕션에서 초당 수십~수백 건의 요청을 처리하면 트레이스 데이터가 급격히 늘어난다. Langfuse 클라우드 플랜의 경우 스토리지 비용도 문제지만, 더 큰 문제는 수천 개의 트레이스 속에서 의미 있는 것을 찾기 어려워진다는 점이다.

### 샘플링 전략

```python
import random

TRACE_SAMPLE_RATE = 0.1  # 10%만 트레이싱

def should_trace() -> bool:
    # 항상 트레이싱해야 하는 케이스
    if is_high_value_user():
        return True
    if has_error_in_recent_calls():
        return True
    # 일반 요청은 샘플링
    return random.random() < TRACE_SAMPLE_RATE
```

### 스코어 기반 필터링

Langfuse에서는 트레이스에 스코어를 매길 수 있다. 낮은 스코어의 트레이스를 우선적으로 확인하는 방식으로 디버깅 효율을 높인다:

```python
@observe(as_type="trace")
def answer_question_scored(user_question: str) -> str:
    answer = answer_question(user_question)
    
    # 응답 품질 자동 평가
    score = evaluate_answer_quality(user_question, answer)
    
    langfuse_context.update_current_trace(
        scores={"answer_quality": score}
    )
    
    # 낮은 스코어만 알림
    if score < 0.5:
        send_alert(f"Low quality answer detected: score={score}")
    
    return answer
```

## 평가(Evaluation) 파이프라인 구축하기

옵저버빌리티의 궁극적 목표는 관찰 그 자체가 아니라 개선이다. 프로덕션 트레이스를 기반으로 평가 데이터셋을 구축하고, 에이전트 성능을 지속적으로 측정하는 파이프라인이 필요하다.

### 1단계: 프로덕션 트레이스에서 데이터셋 생성

Langfuse 대시보드에서 좋은 트레이스와 나쁜 트레이스를 직접 골라 데이터셋을 만들 수 있다. API로도 가능하다:

```python
from langfuse import Langfuse

langfuse = Langfuse()

# 낮은 스코어의 트레이스 수집
low_score_traces = langfuse.get_traces(
    tags=["production"],
    score_threshold=0.3,
    limit=50,
)

# 데이터셋 생성
dataset = langfuse.create_dataset(
    name="rag-agent-regression-v2",
    description="낮은 품질의 프로덕션 응답 기반 회귀 테스트"
)

for trace in low_score_traces:
    input_msg = trace.input
    expected = trace.output  # 정답은 직접 수정 가능
    langfuse.create_dataset_item(
        dataset_name="rag-agent-regression-v2",
        input=input_msg,
        expected_output=expected,
    )
```

### 2단계: 자동 평가 실행

```python
@observe(as_type="trace")
def run_eval(dataset_name: str):
    dataset = langfuse.get_dataset(dataset_name)
    
    for item in dataset.items:
        # 에이전트 실행
        actual = answer_question(item.input)
        
        # LLM-as-judge로 평가
        eval_prompt = f"""다음 질문에 대한 두 답변을 비교해라.
        
질문: {item.input}
기대 답변: {item.expected_output}
실제 답변: {actual}

정확성, 완전성, 관련성을 각각 1~5점으로 평가하고 JSON으로 응답해라."""
        
        eval_result = call_llm(eval_prompt)
        scores = parse_eval_json(eval_result)
        
        # Langfuse에 평가 결과 기록
        langfuse_context.update_current_trace(
            scores=scores
        )

run_eval("rag-agent-regression-v2")
```

## 아키텍처 선택 가이드: Langfuse vs LangSmith vs 자체 구축

에이전트 옵저버빌리티 도구를 선택할 때 고려할 점을 정리했다.

| 기준 | Langfuse | LangSmith | 자체 구축 (OTel + Grafana) |
|------|----------|-----------|--------------------------|
| 오픈소스 | ✅ 오픈소스 (Apache 2.0) | ❌ 독점 | ✅ |
| 자체 호스팅 | ✅ Docker로 간단 | ❌ 클라우드만 | ✅ |
| OTel 통합 | ✅ OTLP 백엔드 | ❌ 자체 포맷 | ✅ 네이티브 |
| 프롬프트 관리 | ✅ | ✅ | ❌ 별도 도구 |
| 평가 파이프레인 | ✅ 내장 | ✅ 내장 | ❌ 직접 구현 |
| LangChain 통합 | ✅ | ✅ (1차 파티) | ⚠️ 수동 |
| 비용 | 무료(자체) / 종량제 | 종량제 | 인프라 비용만 |

빠르게 시작하려면 Langfuse가 가장 무난하다. 이미 Grafana/Prometheus 스택을 쓰고 있다면 OTel 트레이스를 기존 인프라로 보내는 것도 좋은 선택이다.

## 자주 하는 실수와 해결책

**1. 로그와 트레이스를 혼동하기**: `print()`나 `logging`으로 남기는 로그는 트레이스가 아니다. 트레이스는 구조화된 데이터로, 각 스팬의 관계(부모-자식)가 명확해야 한다.

**2. 프롬프트 전체를 기록하기**: 프롬프트에 사용자 개인정보가 포함될 수 있다. PII 마스킹을 적용하거나, 최소한 프롬프트의 앞부분만 기록하는 전략이 필요하다:

```python
def sanitize_for_logging(text: str, max_len: int = 500) -> str:
    """PII를 마스킹하고 길이를 제한하는 유틸리티"""
    # 이메일, 전화번호 등 패턴 마스킹
    text = re.sub(r'\b[\w.-]+@[\w.-]+\.\w+\b', '[EMAIL]', text)
    text = re.sub(r'\b\d{3}[-.]?\d{3,4}[-.]?\d{4}\b', '[PHONE]', text)
    return text[:max_len]
```

**3. 모든 트레이스를 동등하게 취급하기**: 에러가 발생한 트레이스, 높은 비용이 든 트레이스, 낮은 평가 점수의 트레이스를 우선적으로 분석해야 한다.

**4. 개발 환경만 트레이싱하기**: 가장 많이 배우는 건 프로덕션 트레이스다. 실 사용자 데이터를 기반으로 문제를 발견하고, 그 트레이스를 테스트 케이스로 변환하는 루프가 핵심이다.

## 결론: 관찰에서 개선으로

AI 에이전트 옵저버빌리티는 단순히 "무슨 일이 일어났는지"를 아는 것으로 끝나지 않는다. 진짜 가치는 관찰 → 분석 → 평가 → 개선의 사이클을 만드는 데 있다.

핵심 요약:

- **에이전트는 비결정적이다** — 같은 입력에도 매번 다른 경로를 탄다. 전체 실행 경로를 추적해야 원인을 파악할 수 있다
- **Langfuse로 빠르게 시작하라** — `@observe` 데코레이터 몇 줄로 트레이싱, 비용 추적, 평가를 한 번에 구축할 수 있다
- **OpenTelemetry와 연동하면 기존 인프라와 통합된다** — GenAI 시맨틱 컨벤션을 따르면 APM 대시보드에서 LLM 호출까지 한눈에 본다
- **프로덕션 트레이스를 테스트 데이터셋으로 변환하라** — 실제 실패 사례를 기반으로 회귀 테스트를 만들면 에이전트 품질이 지속적으로 개선된다
- **샘플링과 스코어링으로 노이즈를 줄여라** — 모든 트레이스를 같은 우선순위로 보면 정작 중요한 걸 놓친다

당장 해볼 수 있는 첫 단계: 기존 에이전트 코드에 Langfuse SDK를 추가하고 `@observe` 데코레이터를 메인 함수에 붙여 보라. 그 다음 요청 하나를 실행하고 Langfuse 대시보드를 열면, 에이전트가 실제로 무슨 일을 하고 있었는지 처음으로 명확하게 보게 될 것이다.
