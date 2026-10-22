---
title: "프롬프트로 'JSON으로 답해'라고 쓰지 마세요 — LLM 구조화 출력의 3단계 신뢰성과 3사 API 실전 코드"
date: "2026-10-22"
keywords: ["구조화 출력", "structured outputs", "constrained decoding", "JSON Schema", "OpenAI", "Claude API", "Zod"]
lang: "ko"
description: "프롬프트 기반 JSON 추출이 프로덕션에서 무너지는 이유와, OpenAI·Anthropic·Gemini 구조화 출력 API의 차이, Zod/Pydantic으로 스키마를 단일 소스로 만드는 실전 패턴까지."
---

# 프롬프트로 'JSON으로 답해'라고 쓰지 마세요 — LLM 구조화 출력의 3단계 신뢰성과 3사 API 실전 코드

LLM 파이프라인을 만들 때 가장 자주하는 실수는 "다음 JSON 형식으로 답해"라는 문장을 프롬프트에 넣는 것입니다. 프로토타입에서는 대부분 동작하고, 트래픽이 늘면 조용히 무너집니다. 숫자 자리에 문자열이 오고, 필수 필드가 빠지고, 유효한 JSON을 마크다운 코드 펜스로 감싸서 파서를 깨뜨립니다. 한 명의 개발자가 자사 파이프라인에서 실측한 바로는 스키마 강제가 없는 JSON 모드는 스키마 복잡도에 따라 8~15%가량 실패했다고 합니다. 하루 1만 건을 처리하면 하루에 수백 건의 조용한 데이터 오염입니다.

2026년 현재 이 문제의 정답은 정해졌습니다. **구조화 출력(structured outputs)** — 스키마를 문법(grammar)으로 컴파일해서 토큰 생성 단계에서 위반을 원천 차단하는 방식입니다. 이 글에서는 신뢰성의 3단계 계층, OpenAI·Anthropic·Gemini 구현의 실제 차이, 그리고 스키마를 코드의 단일 소스로 만드는 패턴까지 정리합니다.

## 신뢰성은 3단계 계층이다

구조화 출력 접근법은 신뢰성 수준에 따라 세 층으로 나뉩니다. 어느 층을 쓰느냐를 잘못 고르는 것이 프로덕션 사고의 흔한 원인입니다.

| 단계 | 방법 | 실패율 (체감) | 적합한 용도 |
|------|------|------|------|
| 1 | 프롬프트로 JSON 요청 | 5~20% | 프로토타입, 일회성 스크립트 |
| 2 | 함수 호출 / 툴 사용 | 1~5% | 일반 API 앱, 대부분의 에이전트 |
| 3 | 네이티브 구조화 출력 (strict) | 사실상 0% | 데이터 파이프라인, 금융/의료, 라우팅 에이전트 |

1단계의 문제는 "JSON으로 답해"가 **소프트 지시**라는 점입니다. 모델은 이를 스타일 선호 정도로 취급하고, 입력이 복잡하거나 컨텍스트가 길어지면 형식 준수를 우선순위에서 밀어냅니다. 2단계 함수 호출은 훨씬 낫지만 여전히 "잘 맞춰 보려는" 수준입니다. 3단계는 다릅니다. **위반 토큰이 샘플링 자체에서 마스킹되기 때문에, 잘못된 출력이 생성 후 걸러지는 게 아니라 생성 자체가 불가능합니다.** OpenAI는 공식 문서에서 구조화 출력을 "모델이 항상 제공된 JSON Schema를 준수하는 응답을 생성하게 보장하는 기능"으로 설명합니다.

## 동작 원리: 스키마는 문법이 된다

핵심 메커니즘은 **제약 디코딩(constrained decoding)**입니다. 요청에 실린 JSON Schema가 정규 문법으로 컴파일되고, 매 스텝 토큰을 뽑기 전에 그 문법을 위반하게 될 후보 토큰 전부가 로짓 단계에서 마스킹됩니다. Anthropic 공식 문서도 이를 "constrained decoding을 통해 스키마를 준수하는 응답을 보장한다"고 명시합니다.

한 줄로 요약하면 이렇습니다: **유효성 검사는 생성 후가 아니라 생성 중에 일어난다.** `JSON.parse()`가 실패하는 지점이 아니라 샘플러 안에서 이미 끝나 있는 것입니다.

## 3사 구현 비교 — 파라미터 하나하나가 다르다

셋 다 제약 디코딩을 제공하지만, 방언(dialect) 차이가 실제 디버깅 시간을 잡아먹습니다.

### OpenAI — `strict: true`

Responses API에서 스키마와 함께 `strict: true`를 전달하면 제약 디코딩이 활성화됩니다. Python SDK는 Pydantic 모델을 직접 받아 파싱까지 끝내줍니다(공식 문서 예제 기준):

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

response = client.responses.parse(
    model="gpt-6-astra",
    input=[
        {"role": "system", "content": "Extract the event information."},
        {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."},
    ],
    text_format=CalendarEvent,
)
event = response.output_parsed
```

strict 모드의 규칙이 엄격해서 마이그레이션할 때 자주 걸립니다. 모든 객체에 `additionalProperties: false`가 강제되고, **모든 키가 `required`에 들어가야** 하므로 "선택적 필드"는 널러블 유니온으로 표현해야 합니다. 또한 `response_format: {type: "json_object"}`로 쓰던 구 JSON 모드는 문법상 유효한 JSON만 보장할 뿐 스키마 준수는 전혀 보장하지 않으므로 신규 코드에 쓰면 안 됩니다.

### Anthropic — `output_config.format`과 strict tool use

Claude는 구조화 출력을 두 가지 보완 기능으로 제공합니다(공식 문서 기준). **JSON 출력**은 `output_config.format`에 `type: "json_schema"` 스키마를 실어 보내는 방식이고, **strict tool use**는 툴 이름과 입력에 대해 `strict: true`로 스키마 검증을 보장하는 방식입니다. 둘은 독립적으로, 또는 같은 요청에서 함께 쓸 수 있습니다.

Python SDK는 OpenAI와 거의 같은 경험을 제공합니다:

```python
from anthropic import Anthropic
from pydantic import BaseModel

class ContactInfo(BaseModel):
    name: str
    email: str
    plan_interest: str
    demo_requested: bool

client = Anthropic()
response = client.messages.parse(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Extract the key information from this email..."}],
    output_format=ContactInfo,
)
print(response.parsed_output)
```

주의할 점은 SDK의 스키마 변환 방식입니다. Anthropic API가 `minimum`, `maximum`, `minLength`, `maxLength` 같은 제약을 지원하지 않기 때문에, SDK는 이를 **아무 경고 없이 제거하고 필드 설명(description)으로 옮겨 담습니다.** 즉 `Field(ge=0, le=1)`을 붙여도 그 범위가 모델에게 강제되지 않으므로 응답 후 클라이언트 측 재검증이 필수입니다.

### Google Gemini — `responseSchema`

Gemini는 `generationConfig.responseSchema`로 JSON 스키마를 전달하는 방식입니다. OpenAPI 서브셋에서 출발한 탓에 깊게 중첩된 유니온에서 간헐적으로 강제(coercion) 이상 동작이 보인다는 실전 보고가 있으니, 복잡한 스키마는 되도록 평평하게 유지하는 것이 안전합니다.

## 스키마의 단일 소스: Zod 4와 Pydantic

문제는 이제 "API에 보내는 JSON Schema와 애플리케이션 코드의 타입이 어긋나는" 순간 발생합니다. TypeScript 진영의 답은 **Zod 4의 네이티브 `z.toJSONSchema()`**입니다. Zod 공식 문서에 따르면 4버전부터 JSON Schema 변환이 코어에 내장되어, 타입을 정의한 그 스키마 객체가 그대로 API가 강제하는 스키마가 됩니다:

```typescript
import { z } from "zod";

const ReplySchema = z.object({
  sentiment: z.enum(["positive", "negative", "neutral", "unknown"]),
  confidence: z.number().describe("0.0 ~ 1.0"),
});

// 스키마 하나가 타입이자, API 페이로드이자, 응답 검증기
const jsonSchema = z.toJSONSchema(ReplySchema);
```

Python 진영에서는 Instructor가 사실상 표준입니다. PyPI 통계 기준 월 850만 회 이상 다운로드되는 이 라이브러리는, Pydantic 검증 실패 시 에러 메시지를 대화 이력에 append해서 모델에게 "무엇이 틀렸는지" 알려준 뒤 재시도하는 루프를 돌립니다. 블라인드 재시도와 달리 모델의 자기수정 능력을 활용하는 방식으로, 실전에서 대부분의 검증 실패를 1~2회 재시도 안에 해소합니다. OpenAI, Anthropic, Gemini, Ollama 등 열두 개 이상 프로바이더를 같은 인터페이스로 묶어주므로 프로바이더 교체가 한 줄 변경으로 끝납니다.

그리고 규칙이 하나 더 있습니다: **프로바이더가 문법을 강제하더라도 클라이언트 측 `safeParse`를 반드시 실행할 것.** 위에서 본 Anthropic의 min/max 제거 사례처럼, 방언이 지원하지 않는 제약은 응답 도착 후에만 검증할 수 있기 때문입니다.

## 함정 4가지 — "유효한 JSON"은 "올바른 데이터"가 아니다

제약 디코딩이 만능은 아닙니다. 구조화 출력 실전 보고서들이 공통적으로 지적하는 실패 지점은 다음 네 곳입니다.

**1. 형식은 보장되지만 내용은 보장되지 않는다.** 이것이 가장 위험한 오해입니다. enum을 강제하면 모델은 정답이 없을 때도 목록에서 하나를 골라 냅니다. 필수 필드는 항상 채워지는데, 근거가 없으면 그럴듯한 지어내기로 채워집니다. 제약은 모델의 지식을 제약하지 않으며, **보이는 실패를 보이지 않는 실패로 바꿔버립니다.** 처방은 스키마 설계에 있습니다 — enum에 `unknown` 옵션을 넣고, 부재가 정당한 답인 필드는 선택형으로 만들고, 모델이 채울 근거가 없는 필수 필드를 만들지 말 것.

**2. 추론이 필요한 작업에서 정확도가 떨어진다.** 첫 토큰부터 스키마 모양의 토큰을 뽑도록 강제하면 중간 사고의 공간이 줄어듭니다. 한 실전 계측에서는 관점 기반 감성 분석에서 직접 스키마 추출이 자유 서술 후 추출보다 약 4점 낮았다고 합니다. 처방은 **투콜 패턴**입니다 — 먼저 산문으로 추론하게 하고, 그 산문에서 두 번째 호출로 구조화 추출하는 것.

**3. 깊은 중첩과 긴 enum은 품질을 갉아먹는다.** 구조는 유지되더라도 어텐션 예산이 문법 충족에 소모되어 내용 품질이 떨어지고, enum 후보가 길어지면 선택 정확도도 떨어집니다. 스키마는 평평하게, enum은 짧게.

**4. 첫 요청에 문법 컴파일 비용이 있다.** 새 스키마는 첫 호출에서 컴파일 비용을 물고, 주요 프로바이더들이 컴파일 결과를 약 하루 정도 캐시한다는 보고가 있습니다. 스키마를 자주 바꾸는 개발 단계에서 첫 토큰 지연이 커 보인다면 이것이 원인일 수 있습니다.

## 결론

- 구조화 출력 신뢰성은 3단계 계층이며, 다른 시스템으로 흘러가는 출력은 3단계(제약 디코딩)만이 방어 가능한 선택입니다.
- 원리는 스키마를 문법으로 컴파일해 위반 토큰을 샘플링 단계에서 마스킹하는 것 — 검증은 생성 후가 아니라 생성 중에 일어납니다.
- 세 프로바이더의 방언이 다릅니다. OpenAI는 `strict: true` + 전체 키 `required`, Claude는 `output_config.format`/strict tool use이며 SDK가 min/max 제약을 조용히 제거하므로 클라이언트 재검증 필수, Gemini는 `responseSchema`에 평평한 스키마를.
- 스키마는 Zod 4의 `z.toJSONSchema()` 또는 Pydantic+Instructor로 코드의 단일 소스로 만들 것.
- 가장 중요한 것: 구조화 출력은 형식을 보장하지 내용을 보장하지 않습니다. 불확실성을 표현할 길이 없는 스키마는 "확신에 찬 오답"을 양산하도록 설계된 것입니다.

당장 시도해볼 첫 단계는 간단합니다. 지금 "JSON으로 답해"라는 문장이 든 프롬프트 하나를 골라 Pydantic 모델 하나로 바꾸고 `strict` 모드로 돌려보는 것. 파서 방어 코드가 몇 줄 줄어드는 것을 확인하면, 돌아갈 이유가 없어집니다.
