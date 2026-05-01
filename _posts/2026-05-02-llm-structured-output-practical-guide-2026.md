---
title: "LLM 구조화 출력 실전 가이드: 더 이상 정규식으로 JSON 파싱하지 마라"
date: "2026-05-02"
keywords: ["구조화 출력", "Structured Output", "JSON Schema", "OpenAI", "Claude", "제약 디코딩"]
lang: "ko"
description: "OpenAI, Anthropic, Gemini의 네이티브 구조화 출력 API를 비교하고, 제약 디코딩 원리부터 프로덕션 함정까지 실전 코드로 설명하는 2026년 가이드"
---

# LLM 구조화 출력 실전 가이드: 더 이상 정규식으로 JSON 파싱하지 마라

LLM에 "JSON으로 줘"라고 했더니, 마크다운 코드 블록으로 감싸서 온다. 친절하게 설명까지 한 줄 덧붙이고. 그래서 코드 블록 벗기는 정규식을 하나 짠다. 해설 떼는 정규식도 하나 짠다. 어느 날은 `{"result": ...}`로 한 겹 더 감싸서 오고, 어느 날은 `score` 필드가 문자열 `"0.85"`인지 숫자 `0.85`인지도 모른 채 데이터베이스에 밀어 넣는다.

이건 2024년까지의 이야기다. 2026년 현재, 메이저 LLM 프로바이더 전부가 **네이티브 구조화 출력(Structured Output)** API를 제공한다. 출력이 생성되는 그 순간에 스키마를 강제하는, 정규식 따위가 필요 없는 방식이다. 이 글에서는 세 프로바이더의 구조화 출력이 어떻게 다르고, 내부에서 무슨 일이 일어나며, 프로덕션에서 어떤 함정이 기다리는지를 실전 코드와 함께 정리한다.

## 구조화 출력의 세 가지 레벨

구조화 출력이라는 말 아래에 사실 세 가지 완전히 다른 기술이 섞여 있다. 이걸 구분하지 않으면 버그 추적에 며칠을 쓰게 된다.

**레벨 1: 프롬프트 엔지니어링** — "다음 JSON 스키마에 맞춰서 응답해줘"라고 프롬프트에 적는 방식이다. 80~95%는 작동하지만, 엣지 케이스에서 조용히 실패한다. `score`를 `"0.85"`(문자열)로 주거나, 요청하지 않은 `confidence` 필드를 추가로 만들어낸다. 타입 보장이 없다.

**레벨 2: JSON 모드** — `response_format: { type: "json_object" }` 같은 설정으로, 출력이 문법적으로 유효한 JSON임은 보장한다. 하지만 **키와 값은 모델 마음대로**다. 수프 레시피를 JSON으로 달라고 했는데 송장 데이터를 JSON으로 줘도 JSON 모드는 통과시킨다. 이게 가장 위험한 레벨이다. "작동하는 것 같다"는 착각을 준다.

**레벨 3: 스키마 제약 디코딩** — JSON Schema를 전달하면, **토큰이 생성되는 시점에** 스키마를 위반하는 토큰을 원천 차단한다. 100% 스키마 준수를 보장하며, 타입과 값의 범위까지 강제한다. 프로덕션이라면 이 레벨을 써야 한다.

## 제약 디코딩: 출력이 생성되는 순간에 무슨 일이 일어나는가

LLM이 텍스트를 생성할 때, 매 스텝마다 어휘 전체(약 10만 개 토큰)에서 다음 토큰을 선택한다. 제약 디코딩은 이 선택 과정에 **유한 상태 머신(FSM)**을 끼워 넣는다.

```
일반 생성:
  토큰 확률: {"hello": 0.3, "{": 0.1, "The": 0.2, ...}
  → 어떤 토큰이든 선택 가능

제약 생성 (JSON 객체 시작을 기대하는 상태):
  토큰 확률: {"hello": 0.3, "{": 0.1, "The": 0.2, ...}
  마스크:    {"hello": 0,    "{": 1,    "The": 0,   ...}
  → "{"와 공백 토큰만 유효 → 반드시 "{"로 시작
```

`{"name": string, "age": integer}` 스키마의 상태 머신을 따라가 보면:

1. 시작 → `{` 기대
2. `{"name"` 기대
3. `:` 기대
4. 문자열 값 기대 (따옴표 안의 내용)
5. `,` 또는 `}` 기대
6. `,`면 → `"age"` 기대 → `:` 기대 → 정수 값 기대
7. `}` → 완료

매 상태마다 FSM이 스키마를 어기는 토큰을 마스킹한다. 구조는 확실히 보장하면서도, 모델은 유효한 토큰 중 가장 확률이 높은 것을 고를 수 있어 출력 품질도 유지된다.

## 프로바이더별 구조화 출력 API 비교

### OpenAI: `response_format` + `json_schema`

```python
from openai import OpenAI
from pydantic import BaseModel

class Invoice(BaseModel):
    buyer: str
    seller: str
    amount: float
    currency: str = "KRW"

client = OpenAI()
response = client.responses.parse(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "송장 정보를 추출하세요."},
        {"role": "user", "content": "삼성전자가 LG전자에 500만 원어치 부품을 납품했습니다."}
    ],
    text_format=Invoice,
)

invoice = response.output_parsed
print(invoice.buyer)    # "LG전자"
print(invoice.seller)   # "삼성전자"
print(invoice.amount)   # 5000000.0
```

OpenAI는 2024년 8월에 Structured Outputs를 정식 출시했다. `response_format: { type: "json_schema", json_schema: {...} }` 형태로 JSON Schema를 직접 전달하거나, 최신 `responses.parse()` API에서는 Pydantic 모델을 직접 넘길 수 있다. 스키마 준수를 100% 보장한다.

주의점: OpenAI는 스키마에 `additionalProperties: false`를 요구하고, 최대 깊이와 키 개수에 제한이 있다. 선택적 필드는 반드시 기본값을 가져야 한다.

### Anthropic: `output_format` (베타) 또는 `tool_use`

```python
from anthropic import Anthropic
from pydantic import BaseModel

class Invoice(BaseModel):
    buyer: str
    seller: str
    amount: float
    currency: str = "KRW"

client = Anthropic()

# 방법 1: output_format 사용 (2025년 11월 베타 출시)
# 베타 헤더 필요: anthropic-beta: structured-outputs-2025-11-13
response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "삼성전자가 LG전자에 500만 원어치 부품을 납품했습니다."}
    ],
    output_format={
        "type": "json",
        "schema": Invoice.model_json_schema()
    },
    betas=["structured-outputs-2025-11-13"]
)

import json
invoice = Invoice.model_validate_json(response.content[0].text)

# 방법 2: tool_use를 활용한 방식 (베타 이전부터 사용 가능)
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[{
        "name": "extract_invoice",
        "description": "송장 정보를 추출합니다",
        "input_schema": Invoice.model_json_schema()
    }],
    tool_choice={"type": "tool", "name": "extract_invoice"},
    messages=[
        {"role": "user", "content": "삼성전자가 LG전자에 500만 원어치 부품을 납품했습니다."}
    ]
)
```

Anthropic은 2025년 11월 14일에 구조화 출력을 공개 베타로 출시했다. `output_format` 파라미터로 JSON Schema를 전달하면 제약 디코딩이 적용된다. 스키마는 24시간 캐시되어 반복 요청 시 오버헤드가 줄어든다. 현재 Claude Sonnet 4.5와 Opus 4.1을 지원하며, 베타 헤더(`structured-outputs-2025-11-13`)가 필요하다. 베타 이전부터 tool_use를 통한 간접 방식도 가능했고, 여전히 유효하다.

### Google Gemini: `response_schema`

```python
import google.generativeai as genai
from pydantic import BaseModel

class Invoice(BaseModel):
    buyer: str
    seller: str
    amount: float
    currency: str = "KRW"

genai.configure(api_key="YOUR_KEY")

response = genai.GenerativeModel("gemini-2.0-flash").generate_content(
    "삼성전자가 LG전자에 500만 원어치 부품을 납품했습니다.",
    generation_config=genai.GenerationConfig(
        response_mime_type="application/json",
        response_schema=Invoice
    )
)

invoice = Invoice.model_validate_json(response.text)
```

Gemini는 `generation_config`에 `response_schema`를 직접 전달한다. Pydantic 모델을 그대로 넘길 수 있어 코드가 가장 간결하다. 다만 일부 복잡한 스키마(예: 중첩된 `anyOf`)에서 제한이 있을 수 있다.

## 프로덕션 함정: 스키마는 맞았는데 값이 틀리면

구조화 출력이 스키마 준수를 100% 보장한다는 건 알겠다. 하지만 **스키마는 맞으면서 의미적으로 틀린 값**은 잡아주지 않는다.

```python
# 스키마는 완벽하게 준수하지만, buyer와 seller가 뒤바뀐 케이스
{
    "buyer": "삼성전자",    # 실제로는 seller
    "seller": "LG전자",     # 실제로는 buyer
    "amount": 5000000.0,
    "currency": "KRW"
}
```

이건 JSON Schema 레벨에서는 절대 잡을 수 없다. 해결 방법은 세 가지다.

**첫째, 프롬프트에 필드 정의를 명확히 적는다.** "buyer는 물품을 구매하는 측, seller는 물품을 판매하는 측"처럼 각 필드의 의미를 프롬프트에 적어야 한다. 구조화 출력이 제약 디코딩을 사용하더라도, 모델은 프롬프트를 기반으로 "어떤 값을 넣을지"를 결정한다.

**둘째, 검증 로직을 별도로 둔다.** Pydantic validator나 별도 검증 단계에서 비즈니스 로직을 확인한다.

```python
from pydantic import BaseModel, field_validator

class Invoice(BaseModel):
    buyer: str
    seller: str
    amount: float
    currency: str = "KRW"

    @field_validator("amount")
    @classmethod
    def amount_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("금액은 양수여야 합니다")
        return v
```

**셋째, 평가(eval)를 구축한다.** 구조화 출력도 평가 대상이다. 정확한 필드 매핑률, 값의 정확도를 지속적으로 측정해야 한다.

## Instructor: 모든 프로바이더를 하나의 인터페이스로

여러 프로바이더를 동시에 쓰는 환경이라면 [Instructor](https://python.useinstructor.com/) 라이브러리가 유용하다. Pydantic 모델 하나로 OpenAI, Anthropic, Gemini, Ollama 등을 모두 지원한다.

```python
import instructor
from openai import OpenAI
from anthropic import Anthropic
from pydantic import BaseModel

class Analysis(BaseModel):
    summary: str
    sentiment: float  # -1.0 ~ 1.0
    keywords: list[str]

# OpenAI
openai_client = instructor.from_openai(OpenAI())
result = openai_client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "오늘 날씨가 좋아서 기분이 최고야"}],
    response_model=Analysis,
)

# Anthropic — 모델만 바꾸면 된다
claude_client = instructor.from_anthropic(Anthropic())
result = claude_client.messages.create(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "오늘 날씨가 좋아서 기분이 최고야"}],
    response_model=Analysis,
    max_tokens=1024,
)
```

Instructor는 내부적으로 각 프로바이더의 구조화 출력 API를 최우선으로 사용하고, 지원하지 않는 모델에서는 함수 호출(function calling)로 폴백한다. 재시도 로직도 내장되어 있어, 파싱 실패 시 자동으로 재요청한다.

## 언제 어떤 레벨을 써야 하나

간단한 결정 기준을 정리하면:

| 상황 | 추천 레벨 | 이유 |
|------|-----------|------|
| 프로토타입, 내부 도구 | 프롬프트 엔지니어링 | 빠르게 구현, 실패해도 치명적이지 않음 |
| 단순한 JSON 파싱 | JSON 모드 | 문법적 유효성만 보장돼도 충분한 경우 |
| 프로덕션 데이터 파이프라인 | 스키마 제약 디코딩 | 100% 스키마 보장 필수 |
| 다중 프로바이더 환경 | Instructor + Pydantic | 통일된 인터페이스 |
| 스트리밍이 필요한 경우 | 프로바이더 네이티브 API | Instructor는 스트리밍 제약이 있을 수 있음 |

## 핵심 요약

- **정규식으로 LLM 출력을 파싱하는 시대는 끝났다.** 세 메이저 프로바이더 모두 네이티브 구조화 출력을 제공한다.
- **JSON 모드와 스키마 제약 디코딩은 다르다.** JSON 모드는 문법만 보장하고, 스키마 제약은 구조와 타입까지 보장한다. 프로덕션에서는 반드시 스키마 제약을 사용하라.
- **제약 디코딩은 마법이 아니라 FSM이다.** 토큰 생성 시점에 유한 상태 머신이 스키마 위반 토큰을 마스킹하는 원리다.
- **스키마가 맞다고 값이 맞은 건 아니다.** buyer/seller 뒤바뀜 같은 의미적 오류는 프롬프트 명확화, 검증 로직, eval로 잡아야 한다.
- **Instructor 라이브러리로 멀티 프로바이더 환경을 단순화하라.** Pydantic 모델 하나로 교체 가능한 아키텍처를 만들 수 있다.

당장 해볼 것: 기존 프로젝트에서 `json.loads()` + 정규식으로 LLM 출력을 파싱하는 부분을 찾아서, 해당 프로바이더의 네이티브 구조화 출력 API로 교체해 보라. 보통 15분 안에 끝나고, 그 즉시 묻어있던 파싱 버그들이 사라진다.
