---
title: "JSON 파싱 실패는 이제 선택 문제다 — 컨스트레인드 디코딩과 XGrammar-2 구조적 이해"
date: "2026-09-27"
keywords: ["컨스트레인드 디코딩", "constrained decoding", "structured output", "XGrammar", "XGrammar-2", "JSON Schema", "vLLM"]
lang: "ko"
description: "LLM이 잘못된 JSON을 뱉는 근본 원인과 토큰 마스킹으로 이를 원천 차단하는 컨스트레인드 디코딩의 구조, 그리고 에이전트 시대의 XGrammar-2까지 실무 관점으로 정리한다."
---

# JSON 파싱 실패는 이제 선택 문제다 — 컨스트레인드 디코딩과 XGrammar-2 구조적 이해

LLM API로 데이터 파이프라인을 짜본 사람이라면 누구나 이 경험을 한다. "JSON 형식으로 답해줘"라고 프롬프트에 적었는데 어느 날 답변 앞에 "Sure! Here is your JSON:"이라는 인사말이 붙어있고, 파서는 토큰 400번째쯤에서 괄호 하나가 어긋난 채 죽는다. 재시도 루프를 달고, 검증 라이브러리를 붙이고, "반드시 유효한 JSON만 출력하라"고 프롬프트를 다듬는 이 모든 작업은 사실 증상 치료다. 근본 해결책은 잘못된 출력을 확률적으로 줄이는 게 아니라, 토큰 레벨에서 불가능하게 만드는 것이다.

이 글에서는 그 메커니즘인 컨스트레인드 디코딩(constrained decoding)이 실제로 어떻게 동작하는지, 왜 나이브한 구현이 프로덕션에서 쓸 수 없는지, 그리고 2026년 에이전트 워크로드를 위해 등장한 XGrammar-2의 구조까지 정리한다.

## 컨스트레인드 디코딩은 토큰 마스크다

LLM은 토큰을 하나씩 생성한다. 각 스텝에서 모델은 어휘(vocabulary) 전체에 대한 확률 분포를 뱉고, 샘플러가 그중 하나를 고른다. 이 분포에는 JSON 문법이니 정규식이니 하는 개념이 전혀 없다. "JSON으로 답하라"는 프롬프트는 모델에게 스타일 선호 정도로만 해석될 뿐, 하드 제약이 아니다.

컨스트레인드 디코딩은 모델과 샘플러 사이에 개입한다. 목표 구조(JSON 스키마, 문맥 자유 문법, 정규식)를 상태 기계로 컴파일하고, 지금까지 생성된 토큰에 비추어 도달 가능한 상태를 추적한 뒤, **토큰 마스크**를 계산한다. 구조를 위반할 모든 토큰의 확률을 0으로 만드는 것이다. 모델은 여전히 유효한 토큰 중 무엇을 낼지 스스로 결정하지만, 물리적으로 무효한 토큰은 낼 수 없다. 출력 정확성이 확률적 주장에서 구조적 보장으로 바뀌는 순간이다.

한 가지 중요한 뉘앙스가 있다. 제약은 **형식을 잡지 의미를 잡지 못한다**는 점이다. 문법은 모델이 도시 이름 문자열을 강제로 뱉게 할 수 있지만, 그 도시가 실제 존재하는지는 보장하지 못한다. 다만 실무에서는 이 한계가 생각보다 아프지 않다. 형식 오류가 진짜 능력을 가리고 있었던 경우가 많아서, 형식 실패가 사라지고 나면 도구 호출 벤치마크의 측정 정확도가 오히려 올라가는 현상이 관찰된다. 파싱 실패로 버려지던 응답이 없어지기 때문이다.

## 왜 나이브한 구현은 느린가

정석대로 구현하면 비용이 크다. 문맥 자유 문법을 어휘 전체 토큰(수만~수십만 개)에 대해 매 디코딩 스텝마다, 요청마다 실행하는 것이다. 이렇게 하면 문법 검사 비용이 트랜스포머 forward pass 자체를 능가할 수 있다.

XGrammar 논문(MLSys 2025, arXiv 2411.15100)이 이 문제를 여러 아이디어로 공략했고, 이 설계는 지금 대부분의 서빙 엔진에 반영되어 있다.

- **어휘 분할(Vocabulary partitioning)** — 대부분의 토큰은 '문맥 독립적'이다. 합법 여부가 주변 파싱 문맥이 아니라 현재 문법 상태만으로 결정된다. 이런 토큰들은 컴파일 타임에 한 번 미리 검사해 둔다. 런타임에 해석이 필요한 건 문맥 의존적 토큰이라는 더 작은 집합뿐이다.
- **문법 확장(Grammar expansion)** — 컴파일 전에 문법을 변형해 더 많은 토큰을 문맥 독립 범주로 밀어 넣는다.
- **영속 스택(Persistent stack)** — 문맥 의존적 검사가 파싱 스택 상태를 스텝 간에 재사용하여, 반복 작업을 값싼 백트래킹으로 바꾼다.
- **엔진 동시설계(Engine co-design)** — 마스크 생성을 GPU 실행과 겹쳐서(overlap) 문법 작업이 forward pass 뒤에 숨겨지게 한다.

이 조합의 효과는 기존 방식 대비 최대 100배 속도 향상으로 보고되었고(XGrammar 논문), 이것이 컨스트레인드 디코딩을 연구 데모에서 vLLM·SGLang·TensorRT-LLM·MLC-LLM의 기본 백엔드 자리로 끌어올렸다.

## 에이전트가 단일 스키마의 벽을 부쉈다

요청당 JSON 스키마 하나면 되던 시절은 지났다. 현대 에이전트의 한 턴은 단일 구조가 아니다. 자유 서술로 reasoning을 하다가 모델 고유의 와이어 포맷으로 도구를 호출하고, 다시 자유 텍스트로 돌아와 또 다른 도구를 호출한다. 모델 패밀리마다 포맷이 다르고(OpenAI의 harmony 형식, 오픈소스 모델 각자의 툴 콜 방언), 그 명세는 종종 문서조차 불완전하다.

XGrammar-2(2026년 5월 발표, arXiv 2601.04426)는 이 문제를 **Structural Tag**로 풀었다. "응답 전체가 이 스키마와 일치해야 한다"가 아니라, 구조의 시퀀스를 기술하는 컴포저블 JSON 프로토콜이다. 예를 들어 `<answer>` 태그 안의 내용이 특정 JSON 스키마를 따르도록 강제하는 최소 예제는 다음과 같다.

```json
{
  "type": "structural_tag",
  "format": {
    "type": "tag",
    "begin": "<answer>",
    "content": {
      "type": "json_schema",
      "json_schema": {
        "type": "object",
        "properties": {
          "status": { "type": "string" },
          "message": { "type": "string" }
        },
        "required": ["status", "message"]
      }
    },
    "end": "</answer>"
  }
}
```

태그는 합성된다. 시퀀스로 섹션을 이어붙이고, 트리거 태그로 모델이 자유 텍스트를 쓰다가 스스로 도구 호출을 시작하게 하며, JSON 스키마 조각·정규식·리터럴 문자열·토큰 ID 같은 원자 타입이 서로 중첩된다. 엔진이 주요 모델 포맷용 structural tag를 내장하고 있어 서빙 레이어가 모델 패밀리마다 파서를 손으로 유지할 필요가 없어졌다.

에이전트 규모의 워크로드는 효율 문제도 드러냈다. 하나의 에이전트 세션이 수십~수백 개의 도구 스키마를 들고 다니는데, 대부분 같은 하위 구조를 공유한다(모든 오브젝트 스키마는 같은 문자열 필드 기계를 포함한다). XGrammar-2의 **크로스-그래머 캐시**가 공유 하위 구조를 한 번만 컴파일해 재사용하고, 반복 상태 압축과 배칭·스페큘러티브 디코딩 지원으로 거대한 문법에서도 오버헤드를 0에 가깝게 유지한다. XGrammar-2는 XGrammar 대비 최대 80배 효율 개선을 보고했고(MLC 공식 발표), xAI·Databricks·DeepSeek 등이 제품에 채택했으며 SGLang·vLLM·TensorRT-LLM·MLC-LLM이 통합했다.

## 실무에서는 어떻게 쓰나 — 세 가지 신뢰성 레벨

구조화 출력의 신뢰성은 세 단계로 나뉘고, 이 레벨을 잘못 고르는 것이 프로덕션 사고의 흔한 원인이라는 분석이 있다.

| 레벨 | 방법 | 신뢰성 | 적합한 용도 |
|------|------|--------|-------------|
| 1 | 프롬프트 엔지니어링 | 80~95% | 프로토타입, 일회성 스크립트 |
| 2 | 함수 호출 / 툴 사용 | 95~99% | 일반 API 앱, 대부분의 에이전트 |
| 3 | 네이티브 구조화 출력(strict) | 99% 이상 | 파이프라인, 프로덕션 데이터 추출 |

레벨 1이 실패하는 이유는 근본적이다. 하루 만 건을 처리하는 파이프라인에서 5% 실패는 500건의 깨진 추출을 의미한다. 레벨 3의 구현체들이 각 사의 API에서 어떻게 노출되는지는 다음과 같다.

**OpenAI**는 2024년 8월 6일 gpt-4o-2024-08-06과 함께 Structured Outputs를 출시했다. 디코드 타임에 컴파일된 문법으로 스키마를 위반하는 토큰을 마스킹하며, 지원 모델에서 100% 스키마 준수를 주장한다. 주의점이 두 가지 있다. `response_format`과 `tools`는 한 호출에서 함께 쓸 수 없어서 둘 다 필요하면 스키마를 강제 도구(forced tool)로 감싸야 하고, 선택적 필드는 `required`에서 빼는 게 아니라 `["string", "null"]` 유니온으로 표현해야 한다. 스키마 제약(첫 호출 시 100개 속성, 5단계 중첩 등)도 있다.

```python
from openai import OpenAI
from pydantic import BaseModel

class Extract(BaseModel):
    name: str
    age: int

client = OpenAI()
resp = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[{"role": "user", "content": "프로필을 추출해줘: 김철수, 32세"}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "extract",
            "strict": True,
            "schema": Extract.model_json_schema(),
        },
    },
)
```

**직접 서빙하는 경우**에는 vLLM의 구조화 출력 기능이 대표적이다. 문법 백엔드로 XGrammar·Outlines·Guidance 중 선택 가능하다.

```python
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")
schema = '{"type":"object","properties":{"sentiment":{"type":"string","enum":["pos","neg","neu"]}},"required":["sentiment"],"additionalProperties":false}'

out = llm.chat(
    messages=[{"role": "user", "content": "리뷰 감성 분류: 배송도 빠르고 좋았어요"}],
    sampling_params=SamplingParams(temperature=0.1),
    chat_template_kwargs={"add_generation_prompt": True},
    extra_body={"guided_json": schema},
)
```

**앱 레이어에서 프로바이더 중립적으로** 쓰려면 Instructor가 사실상 표준이다. Pydantic 모델을 정의하면 검증 실패 시 자동 재시도까지 붙여주는 라이브러리로, 한 분석에서 월 300만 다운로드를 넘겼다고 한다(2026년 기준, 단일 출처라 참고 수치로 봐야 한다).

```python
import instructor
from pydantic import BaseModel

class Sentiment(BaseModel):
    label: str
    score: float

client = instructor.from_openai(OpenAI())
result = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    response_model=Sentiment,
    messages=[{"role": "user", "content": "리뷰 감성 분석: 최악이었어요"}],
)
```

로컬 모델을 통째로 제어하고 싶다면 Outlines(dottxt-ai가 유지보수)가 정규식·JSON 스키마·CFG 기반 구조화 생성을 지원한다.

## 결론: 이제 실패는 아키텍처의 선택이다

- LLM의 JSON 실패는 모델의 실수가 아니라, 하드 제약 없이 부드러운 지시만 전달한 설계의 결과다.
- 컨스트레인드 디코딩은 문법을 상태 기계로 컴파일해 토큰 마스크를 씌우는 기술이고, 어휘 분할·스택 재사용으로 오버헤드를 0에 가깝게 만들었다.
- 에이전트 시대의 혼합 구조(자유 텍스트 + 도구 호출 + reasoning 채널)는 XGrammar-2의 Structural Tag가 통합적으로 다루며, 주요 서빙 엔진이 이미 채택했다.
- 신뢰성은 레벨 1(프롬프트) → 레벨 2(함수 호출) → 레벨 3(네이티브 strict)로 갈수록 올라간다. 다운스트림 시스템이 출력을 소비한다면 레벨 3이 유일하게 방어 가능한 선택이다.
- 단, 제약은 형식만 보장한다. 값의 의미적 정확성은 여전히 프롬프트·모델·후처리의 몫이다.

당장 시도해볼 첫 단계는 간단하다. 기존 파이프라인에서 JSON 재시도 루프가 있는 지점을 찾아, 해당 호출에 `response_format` strict 모드(또는 vLLM의 `guided_json`)를 붙여보라. 재시도 코드가 통째로 사라지는 경험을 하게 될 것이다.
