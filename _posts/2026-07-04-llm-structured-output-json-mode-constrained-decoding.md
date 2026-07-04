---
title: "LLM이 JSON을 깨트릴 때: 구조화 출력 3단계와 프로덕션에서 실제로 부서지는 지점"
date: "2026-07-04"
keywords: ["구조화 출력", "Structured Outputs", "JSON Mode", "constrained decoding", "LLM"]
lang: "ko"
description: "LLM에서 JSON을 안정적으로 받아내는 세 가지 방법(JSON Mode, Function Calling, Structured Outputs)의 차이와 제약 디코딩 원리, 그리고 프로덕션에서 실제로 부서지는 지점을 정리한다."
---

# LLM이 JSON을 깨트릴 때: 구조화 출력 3단계와 프로덕션에서 실제로 부서지는 지점

LLM에 "JSON 형식으로 답해"라고 적었는데 모델이 마음대로 필드를 추가하거나, enum에 없는 값을 집어넣거나, 아예 마크다운 코드블록으로 감싸버리는 경험을 해본 적이 있을 것이다. 이 문제는 단순한 불편함이 아니다. 파서가 실패하고, 파이프라인이 멈추고, 데이터가 조용히 사라진다.

이 글에서는 LLM에서 구조화된 출력을 얻는 세 가지 방법(JSON Mode, Function Calling, Structured Outputs)의 차이를 정리하고, 그 기반이 되는 **제약 디코딩(constrained decoding)**의 원리를 설명한 뒤, 프로덕션 환경에서 실제로 출력이 부서지는 지점과 대응 패턴을 다룬다.

## 구조화 출력의 3단계: JSON Mode → Function Calling → Structured Outputs

세 가지 방법은 "얼마나 강하게 제약하는가"로 구분된다. 이해하기 쉽게 각각이 무엇을 보장하고 무엇을 보장하지 않는지 정리한다.

### JSON Mode: 문법만 보장한다

OpenAI, Mistral, Groq 등에서 지원하는 JSON Mode(`response_format: {"type": "json_object"}`)는 모델의 토크나이저가 유효한 JSON 문법에 해당하는 토큰만 생성하도록 제한한다. 괄호 짝이 맞고, 따옴표가 닫히고, 후행 콤마가 없는 것까지는 보장한다.

하지만 **스키마 일치는 보장하지 않는다.** 모델이 당신이 정의하지 않은 필드를 추가할 수도, 필수 필드를 빼먹을 수도, 숫자 자리에 문자열을 넣을 수도 있다.

```python
# OpenAI JSON mode
response = client.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system", "content": "상품 정보를 추출하라. name, price, category 필드를 포함한 JSON으로 답하라."},
        {"role": "user", "content": product_description}
    ]
)
# 보장: 유효한 JSON
# 미보장: name/price/category가 실제로 있는지, price가 숫자인지, category가 지정한 enum인지
```

### Function Calling: 스키마를 채우도록 유도한다

Function Calling(또는 Tool Use)은 API 호출에 JSON Schema를 정의하고, 모델이 그 스키마에 맞춰 출력을 생성하도록 한다. OpenAI는 `tools`, Anthropic은 `tool_use`라는 이름으로 부르지만 동작 원리는 같다.

```python
# OpenAI function calling
response = client.chat.completions.create(
    model="gpt-4o",
    tools=[{
        "type": "function",
        "function": {
            "name": "extract_product",
            "parameters": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "price": {"type": "number", "minimum": 0},
                    "category": {"type": "string", "enum": ["electronics", "clothing", "food", "other"]},
                },
                "required": ["name", "price", "category"],
                "additionalProperties": False,
            }
        }
    }],
    tool_choice={"type": "function", "function": {"name": "extract_product"}},
    messages=[{"role": "user", "content": product_description}]
)
```

스키마에 정의된 필드를 채우도록 강제하지만, 100%는 아니다. 모호한 입력에서는 enum에 없는 값을 넣거나, 깊이 중첩된 스키마에서 필드를 빼먹는 경우가 여전히 발생한다. 한 프로덕션 환경에서의 실측 데이터에 따르면(GPT-4o 기준), Function Calling의 스키마 위반율은 약 1.2% 수준이었다.

### Structured Outputs: 토큰 수준에서 스키마를 강제한다

2024년 8월, OpenAI가 발표한 Structured Outputs는 Function Calling의 `strict: true` 모드로 동작한다. 모델이 스키마에 없는 속성을 추가할 수 없고, 모든 필수 필드가 반드시 존재해야 하며, enum 값이 토큰 수준에서 강제된다. 현재 사용 가능한 가장 강력한 제약 방식이다.

```python
# OpenAI Structured Outputs (strict mode)
"function": {
    "name": "extract_product",
    "strict": True,  # 이것이 핵심 차이
    "parameters": { ... }
}
```

Anthropic은 2025년 11월부터 Claude API에서 Structured Outputs를 공식 지원하기 시작했다(`structured-outputs-2025-11-13` 베타 헤더로 시작, 이후 GA). 이전에는 `tool_choice`로 특정 도구를 강제하고 `additionalProperties: false`를 설정해 유사한 효과를 얻을 수 있었다. Google Gemini는 `response_schema` 파라미터로 유사한 기능을 제공하며, 2024년 중반경 발표된 것으로 알려져 있다.

## 제약 디코딩: 토큰 확률을 마스킹하는 원리

세 가지 방법 중 Structured Outputs가 100%에 가까운 정확도를 내는 이유는 **제약 디코딩(constrained decoding)**이라는 기술 때문이다. 이 원리를 이해하면 왜 특정 스키마에서 출력이 부서지는지 알 수 있다.

LLM은 매 시점마다 어휘(vocabulary) 전체에 대한 확률 분포(logit)를 계산한다. 제약 디코딩은 모델 출력과 샘플링 단계 사이에 **logit processor**를 끼워 넣어, 현재 문법 상태에서 유효하지 않은 토큰의 확률을 0(음의 무한대)으로 만드는 방식이다.

구체적인 과정은 다음과 같다:

1. **오프라인(스키마당 1회):** JSON Schema를 정규식 또는 문맥 자유 문법(CFG)으로 변환하고, 상태 기계(FSM) 또는 푸시다운 오토마톤(PDA)을 구축한다. 각 상태에서 어떤 토큰이 유효한 전환인지 미리 계산한다.
2. **온라인(토큰마다):** 현재 상태를 해시맵에서 조회(O(1))하고, 유효하지 않은 토큰을 마스킹한 뒤 샘플링하고, 상태 기계를 다음 상태로 이동시킨다.

이 접근의 기초는 Willard와 Louf가 2023년에 발표한 Outlines 라이브러리에서 정립되었다. FSM 방식은 단순한 스키마에서는 빠르지만, JSON의 재귀적 구조(객체 안의 객체, 배열 안의 배열)를 표현할 수 없다는 한계가 있다. 정규식은 재귀를 표현할 수 없기 때문이다.

문맥 자유 문법(CFG)은 스택을 가진 푸시다운 오토마톤을 사용해 재귀를 처리한다. XGrammar(Dong et al., MLSys 2025, [arXiv:2411.15100](https://arxiv.org/abs/2411.15100))는 어휘 토큰을 문맥 독립(99%)과 문맥 의존(1%)으로 분리해, 기존 방식 대비 최대 100배의 속도 향상을 달성했다. XGrammar는 2024년 12월 vLLM, 2024년 11월 SGLang, 2025년 1월 TensorRT-LLM에 기본 구조화 생성 백엔드로 통합되었다. Microsoft의 llguidance는 Rust 기반 Earley 파서를 사용하며, 업계 보도에 따르면 OpenAI가 자사 Structured Outputs의 기반이 되는 작업으로 llguidance를 언급한 바 있다.

> **참고:** 제약 디코딩의 오버헤드는 토큰당 50마이크로초 미만으로, 모델 추론 시간(토큰당 10~50ms)에 비하면 무시할 수 있는 수준이다. 스키마 컴파일은 최초 1회만 발생하며, 이후 요청에서는 캐싱된다.

## 프로덕션에서 실제로 부서지는 지점

제약 디코딩은 문법적 정확성을 보장하지만, **의미론적 정확성**까지 보장하지는 않는다. 구조는 완벽한데 값이 틀리는 경우가 실전에서 가장 까다롭다. 구체적으로 어떤 패턴에서 출력이 부서지는지 살펴본다.

### 1단계: 깊이 중첩된 스키마

3단계 이상 중첩된 스키마는 평평한(flat) 스키마보다 위반율이 3~5배 높다(단일 프로덕션 환경 측정 기준). 3단계까지는 기준선에 가깝지만, 4단계에서 대략 두 배로 증가하고, 5단계 이상에서는 8~12%에 달할 수 있다.

```python
# 이 정도는 괜찮다: 2단계
class Address(BaseModel):
    street: str
    city: str
    country: str

class Contact(BaseModel):
    name: str
    address: Address

# 이것은 위반율이 올라간다: 4단계
class LineItem(BaseModel):
    product: Product          # 1
    quantity: int
    pricing: PricingDetail    # 2

class PricingDetail(BaseModel):
    base: float
    discounts: list[Discount] # 3

class Discount(BaseModel):
    rule: DiscountRule        # 4 — 이 단계에서 성능 저하가 뚜렷함
    amount: float
```

**대응:** 스키마를 최대한 평평하게 유지하라. 깊이 중첩된 데이터가 필요하면 여러 추출 호출로 분할하는 것이 더 안정적이다.

### 2단계: Nullable + Enum 조합

`Optional[Literal[...]]` 조합은 일관된 문제 지점이다. 모델이 상태를 확신하지 못할 때 `null` 대신 `"unknown"`이나 `"n/a"`를 반환하는 경우가 있다. 문법적으로는 합리적이지만 스키마 검증에는 실패한다.

```python
# 사후 처리 검증기로 보정
STATUS_COERCION = {
    "unknown": None,
    "n/a": None,
    "not specified": None,
    "unavailable": None,
}

def coerce_nullable_enum(value, valid_values):
    if value is None:
        return None
    if value in valid_values:
        return value
    return STATUS_COERCION.get(value.lower(), None)
```

이러한 보정 계층 하나만 추가해도 enum 관련 위반의 약 70%를 줄일 수 있다.

### 3단계: 긴 문서 + 많은 필드

15개 이상의 필드를 가진 스키마를 3,000 토큰 이상의 긴 문서에 적용하면 위반율이 올라간다. 모델이 스키마의 후반부 필드를 "잃어버리는" 현상이다. 이는 문맥 주의력(context attention) 문제이지 스키마 문제가 아니다.

**대응:** 가장 중요한 필드를 스키마 정의의 앞쪽에 배치하라. 큰 스키마를 분할해 동일한 문서에 대해 두 번의 추출을 실행하라. 한 번의 API 호출이 추가되지만(약 150~250ms), 각 하위 스키마의 위반율은 기준선으로 돌아간다.

## 의미론적 드리프트: 가장 조용한 실패

여기까지의 문제는 스키마 검증으로 잡을 수 있다. 진짜 까다로운 문제는 **의미론적 정확성**이다. 모델이 스키마에 완벽히 부합하는 JSON을 반환하지만, 값이 틀린 경우다. 에러도 없고, Pydantic 검증도 통과하고, 애플리케이션은 잘못된 데이터를 받아들인다.

예를 들어 계약 조항 추출 파이프라인에서 `risk_level: Literal["low", "medium", "high"]` 필드를 정의했다고 하자. 모델은 항상 스키마에 맞는 값을 반환하지만, 모호한 조항에서는 실행할 때마다 다른 위험도를 반환한다. 모델의 "중위험" 개념이 법무팀의 정의와 일치하지 않는 것이다.

이 문제는 스키마 검증으로 잡을 수 없다. **의미론적 검증**이 필요하다.

```python
from pydantic import BaseModel, field_validator
from typing import Literal

class ContractClause(BaseModel):
    clause_text: str
    risk_level: Literal["low", "medium", "high"]
    justification: str  # 모델에게 이유를 강제로 작성하게 한다

    @field_validator("risk_level")
    @classmethod
    def validate_risk_consistency(cls, v, values):
        # "high"인데 근거가 짧으면 뭔가 잘못된 것
        if v == "high" and "justification" in values.data:
            if len(values.data["justification"]) < 50:
                raise ValueError("고위험 분류에는 구체적 근거가 필요합니다")
        return v
```

`justification` 필드를 추가하면 두 가지 효과가 있다. 첫째, 모델이 자신의 추론을 명시적으로 작성해야 하므로 정확도가 올라간다(구조화 출력 안에 내장된 chain-of-thought). 둘째, 짧은 근거는 불확실한 분류의 신호가 되므로, 잘못된 값을 전파되기 전에 감지할 수 있다. 실제 측정에서는 분류 출력에 근거 또는 신뢰도 필드를 추가하면 의미론적으로 잘못된 분류의 30~40%를 사전에 잡을 수 있다.

> **주의:** 제약 디코딩은 출력 형식의 보장이지, 내용의 정확성 보장이 아니다. Park et al.(NeurIPS 2024)은 높은 확률의 토큰을 마스킹하면 모델의 확률 분포가 왜곡되어, 문법적으로는 맞지만 의미론적으로는 저하된 출력이 나올 수 있음을 보였다. 구조화 출력을 "전송 보장"으로 취급하고, 비즈니스 로직을 위한 Pydantic 검증자와 의미론적 품질 모니터링을 별도로 추가하라.

## 재시도 패턴: 0%가 아닌 실패율에 대응하기

Structured Outputs(strict mode)를 사용해도 0.2% 수준의 위반율이 남는다. 표준적인 대응은 검증 에러를 다음 시도에 주입하는 재시도 패턴이다.

```python
async def extract_with_retry(document, schema, max_retries=2):
    for attempt in range(max_retries + 1):
        response = await call_with_function_calling(document, schema)
        try:
            return schema.model_validate(response)
        except ValidationError as e:
            if attempt == max_retries:
                logger.error("추출 실패 (재시도 %d회): %s", max_retries, e)
                return None
            # 다음 시도에 검증 에러를 주입
            error_feedback = format_validation_errors(e)
            document = f"{document}\n\n[이전 응답: {response}. 검증 에러: {error_feedback}. 수정하라.]"
    return None
```

재시도 패턴의 핵심 원칙 세 가지:

1. **에러 피드백 주입은 실제로 효과가 있다.** 모델은 자신의 이전 출력과 검증 에러를 읽고 대개 두 번째 시도에서 문제를 수정한다. 첫 시도 실패의 약 80%가 재시도에서 해결된다.
2. **재시도는 2회로 제한하라.** 세 번째 재시도는 거의 도움이 되지 않으며 지연만 두 배로 늘린다. 두 번 실패하면 입력이 모호하거나 예상 분포 밖일 확률이 높다.
3. **절대 유효하지 않은 데이터를 조용히 반환하지 마라.** 부분적으로 유효한 객체를 반환하는 유혹이 있지만, 유효한 데이터를 기대하는 다운스트림 코드가 이상한 방식으로 실패한다. 명시적인 `None`은 눈에 보이는 버그이지만, 미묘하게 잘못된 객체는 몇 주간 숨어 있는 버그가 된다.

## Pydantic + Instructor: 멀티 프로바이더 추상화

OpenAI, Anthropic, Google 등 여러 프로바이더를 오가며 작업해야 한다면, Instructor 라이브러리가 가장 실용적인 추상화 계층을 제공한다. Instructor는 LLM 클라이언트 SDK를 패치해 Pydantic 모델을 직접 받아들이도록 만든다.

```python
import instructor
from openai import OpenAI

# OpenAI
client = instructor.from_openai(OpenAI())
result = client.chat.completions.create(
    model="gpt-4o",
    response_model=ReviewAnalysis,
    messages=[{"role": "user", "content": review_text}]
)

# Anthropic으로 전환 — 클라이언트 생성자만 바꾸면 된다
import instructor
from anthropic import Anthropic

client = instructor.from_anthropic(Anthropic())
result = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    response_model=ReviewAnalysis,
    max_tokens=1024,
    messages=[{"role": "user", "content": review_text}]
)
```

Instructor는 스키마 변환, 검증, 검증 실패 시 자동 재시도를 처리한다. 동일한 Pydantic 모델이 OpenAI, Anthropic, Google, Mistral, Cohere, Ollama, DeepSeek 등 15개 이상의 프로바이더에서 작동한다. 단, 재시도 메커니즘이 근본적으로 깨진 스키마에서는 루프에 빠질 수 있으므로 `max_retries`를 반드시 설정하라(복잡한 스키마는 2~3 권장).

## 핵심 요약

- **JSON Mode는 문법만 보장한다.** 스키마 일치가 필요하면 절대 프로덕션에 쓰지 마라. 프로덕션 실측에서 위반율이 9~12%에 달한다(단일 환경 기준).
- **Structured Outputs(strict mode)가 기본 선택이다.** 0.2% 수준의 위반율로 대부분의 프로덕션 워크로드에 충분하다.
- **제약 디코딩은 토큰 확률을 마스킹해 스키마를 강제한다.** FSM은 단순 스키마에, CFG(XGrammar, llguidance)는 재귀 스키마에 적합하다.
- **깊이 중첩, nullable+enum, 긴 문서는 위반율을 높인다.** 스키마를 평평하게 유지하고, 중요 필드를 앞에 배치하라.
- **문법적 정확성 ≠ 의미론적 정확성.** 근거 필드를 추가하고, Pydantic 검증자로 비즈니스 로직을 보호하라.
- **재시도는 2회까지만.** 에러 피드백 주입으로 첫 실패의 약 80%를 해결할 수 있다.
- **Instructor로 멀티 프로바이더 추상화.** 동일한 Pydantic 모델을 OpenAI, Anthropic, Google 등에서 재사용하라.

당장 시도해볼 첫 단계: 기존에 JSON Mode나 프롬프트 기반 JSON 추출을 사용하고 있다면, Pydantic 스키마를 정의하고 Structured Outputs(strict mode)로 전환하라. 그 다음, 추출 파이프라인에 재시도 래퍼와 의미론적 검증 필드를 추가하는 것으로 시작하라.
