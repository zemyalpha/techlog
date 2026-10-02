---
title: "텍스트를 생성하지 않는 AI 모델이 등장했다 — Cloudflare Clef와 '결정 모델'이 에이전트 핫패스를 바꾸는 방식"
date: "2026-10-02"
keywords: ["결정 모델", "decision model", "Cloudflare Clef", "System One", "Jev", "TypeSafe AI", "AI 에이전트 라우팅", "분류 모델"]
lang: "ko"
description: "Cloudflare가 Apache 2.0으로 연 Clef·Clef-flash는 문자를 생성하지 않고 타입된 확률만 반환하는 '결정 모델'이다. LLM 호출 없이 에이전트의 분류·라우팅·게이팅을 밀리초에 처리하는 이 새로운 카테고리를 해부한다."
---

# 텍스트를 생성하지 않는 AI 모델이 등장했다 — Cloudflare Clef와 '결정 모델'이 에이전트 핫패스을 바꾸는 방식

2026년 10월 1일, Cloudflare가 자사 Workers AI 팀이 직접 학습한 첫 번째 모델을 공개했다. 이름은 Clef와 Clef-flash. 흥미로운 점은 파라미터 수도, 벤치마크 점수도 아니다. 이 모델은 **프롬프트를 받아도 문장을 한 개도 생성하지 않는다**는 것이다. 입력으로 상태(state)와 타입이 정의된 질문들을 받으면, 허용된 답변 각각에 대한 확률만 돌려준다. 채팅 응답도, 추론 토큰도, 파싱할 자유 서술도 없다.

이런 모델을 업계는 '결정 모델(decision model)'이라 부른다. TypeSafe AI가 2026년 9월 중순 첫 System One 모델 'Jev'를 내놓으면서 형성되기 시작한 카테고리인데, Cloudflare가 오픈 웨이트로 참전하면서 생태계 구도가 바뀌었다. 이 글에서는 결정 모델이 무엇이고, 왜 지금 필요한지, 그리고 기존 LLM 구조화 출력과 어떻게 다른지를 정리한다.

## 결정 모델은 무엇인가 — '상태 입력, 타입된 확률 출력'

결정 모델의 인터페이스는 LLM과 완전히 다르게 설계됐다. TypeSafe AI의 공식 문서에 따르면, 호출 구조는 다음과 같다.

1. **상태(state)**: 지금 무슨 일이 벌어지고 있는지를 담은 비정형 데이터. 고객 문의 텍스트, JSON 레코드, 현재 페이지 등
2. **질문(questions)**: 상태에 대해 물을 타입이 정의된 판단. "이 티켓은 어느 팀 소관인가?" 같은 것
3. **결정(decision)**: 사전에 정의된 답변 공간 안에서 각 후보에 대한 확률

TypeSafe는 질문 프리미티브를 세 가지로 정의한다. 여러 옵션 중 하나를 고르는 **Choice**(선택지당 확률 반환), 상태를 루브릭에 따라 평가하는 **Score**, 명제의 참 거짓 확률을 반환하는 **Noul**(0~1)이다. 핵심은 답변 공간이 호출 시점에 이미 닫혀 있다는 것. 모델이 만들 수 있는 출력은 개발자가 정의한 타입 안에서만 움직이므로, 구문 오류가 원천적으로 불가능하고 확률이 낮은 구간에서는 사람이나 추론 모델으로 에스컬레이션하는 코드를 그대로 짤 수 있다.

카테고리 이름인 'System One'은 대니얼 카너먼의 『생각에 관한 생각』에서 빌려왔다. 빠르고 직관적인 판단(System 1)과 느리고 숙고하는 판단(System 2)의 구분을 모델 계층으로 옮긴 것이다. 생성형 LLM이 System 2의 자리를 차지했다면, 결정 모델은 소프트웨어의 핫패스에 박히는 System 1을 표방한다.

## 왜 필요한가 — 에이전트의 시간과 돈은 '작은 판단'에서 새고 있다

AI 에이전트를 실제로 운영해본 사람이라면 안다. 에이전트의 실행 시간과 토큰 비용 대부분은 화려한 추론이 아니라 자잘한 판단에서 나온다. 이 메시지가 긴급한가? 이 요청을 어느 모델로 라우팅할까? 이 도구 호출 결과를 컨텍스트에 유지할까, 잘라낼까? 이 사용자 제출물이 정책 위반인가?

이런 판단들은 손으로 쓴 `if` 문으로 처리하기에는 애매하고, 프론티어 LLM을 부르기에는 과하다. TypeSafe는 이 간극을 "Too fuzzy for a hand-written if, too small for a frontier LLM"(손으로 쓴 if에는 너무 모호하고, 프론티어 LLM에는 너무 사소하다)이라고 표현한다. 결정 모델은 이 틈새를 노린다.

구조화된 출력(structured output)으로도 같은 문제를 풀 수 있지 않냐는 반론이 있다. 그러나 결정 모델과는 근본적인 차이가 있다. 구조화 출력은 여전히 **토큰을 순차적으로 생성하는 LLM**이다. JSON 스키마로 출력 공간을 제약(constrained decoding)해 파싱 실패는 없앨 수 있지만, 디코딩 자체는 토큰을 하나씩 만드는 순차 과정이고, 긴 프롬프트에 대한 프리필 비용도 그대로 지불한다. 반면 결정 모델은 답변 확률 분포를 계산하는 것이 전부이므로, TypeSafe의 주장에 따르면 동일 수준의 판단 작업에서 생성형 LLM 대비 두 자릿수 배수의 속도 차이가 난다. 물론 이 수치는 벤더 자체 측정이므로 실제 도입 시에는 자기 워크로드로 검증해야 한다.

## Cloudflare Clef — 첫 오픈 웨이트 결정 모델

여기까지는 TypeSafe의 Jev(호스티드 API 전용, 웨이트 비공개)가 만든 시장이었다. Cloudflare의 Clef 발표가 구도를 바꾼 지점은 두 가지다.

**첫째, Apache 2.0 오픈 웨이트다.** 두 모델의 가중치는 Hugging Face에 공개됐고, 자체 호스팅이 가능하다. 자체 데이터로 파인튜닝해 프라이빗 환경에 넣고 싶은 기업 입장에서는 닫힌 API뿐이던 선택지에 열린 대안이 생긴 셈이다.

**둘째, 에지 인프라에 얹혔다.** Clef는 Workers AI에서 `@cf/cloudflare/clef`(27B, 정밀도 우선)와 `@cf/cloudflare/clef-flash`(9B, 지연 우선)로 서빙된다. Cloudflare의 전 세계 GPU 네트워크에서 사용자 근처에서 실행되므로 네트워크 왕복이 짧다. Cloudflare changelog가 제시한 지연 수치는 다음과 같다(43회 벤치마크 런 기준, 벤더 측정).

| 지연 | Clef (27B) | Clef-flash (9B) | Jev |
|------|-----------|-----------------|-----|
| 미디언 | 209.3ms | 38.8ms | 524.1ms |
| p95 | 238.6ms | 122.4ms | 536.0ms |

MarkTechPost 보도에 따르면 Clef는 Qwen3.8-27B, Clef-flash는 Qwen3.5-9B를 백본으로 하며, 두 모델 모두 64K 토큰 컨텍스트와 이미지 입력(최대 4장, 비전 인코더 포함)을 지원한다. 특히 이미지 입력은 텍스트만 받는 Jev와의 명시적 차이점이다. 보도된 호스티드 가격은 Clef가 입력 100만 토큰당 $0.24, Clef-flash가 $0.09 수준으로 알려져 있으나, 정확한 최신 가격은 Cloudflare의 공식 가격 페이지에서 확인하기 바란다.

Cloudflare는 자사 위협 인텔리전스 팀의 도메인 분류 작업에 Clef를 내부 테스트했다고 밝혔다. 외부 벤치마크에서도 Decision Index 0.2.1 스위트의 10개 결정 벤치마크 중 7개에서 Clef 계열 모델이 최고 점수를 기록했다고 발표했다. 다만 주의할 점이 있다. 지식 집약 과제는 여전히 다른 영역이다. MarkTechPost가 인용한 모델 카드 수치에서 Jev는 GPQA Diamond(78.3 vs 48.0), MMLU-Pro(82.7 vs 65.9), BBH(92.9 vs 73.7)에서 Clef를 크게 앞선다. 결정 모델은 만능 대체재가 아니라 **분업의 한 층**이다.

## 실전 배치 — 결정 모델 + LLM의 2층 구조

결정 모델의 실전 배치 패턴은 명확하다. 핫패스 판단은 결정 모델에, 깊은 작업은 LLM에 맡기는 것이다. Cloudflare Workers AI에서는 다음과 같이 호출한다.

```typescript
// Workers AI 바인딩으로 Clef 호출
const decision = await env.AI.run("@cf/cloudflare/clef", {
  state: {
    ticket: "결제가 두 번 됐어요. 환불해주세요.",
    customer_tier: "free",
  },
  questions: {
    intent: ["refund", "bug", "billing", "other"],
    needs_human: "bool",
    urgency: { type: "score", max: 3 },
  },
});

// 반환값은 텍스트가 아니라 타입된 확률
// 예: { intent: { choice: "refund", probabilities: {...} },
//       needs_human: { noul: 0.12 },
//       urgency: { score: 2, confidence: 0.9 } }

if (decision.needs_human.noul > 0.8) {
  return escalateToHuman(ticket);
}
if (decision.intent.choice === "refund") {
  return runAgent(REFUND_SPECIALIST_LLM); // 비싼 LLM은 여기서만
}
```

TypeSafe의 라우팅 패턴 문서도 같은 구조를 권한다. 분류와 복잡도 채점을 결정 모델 호출 한 번으로 처리한 뒤, 확신도(confidence)가 낮은 케이스만 사람에게 넘기고, 나머지는 각 인텐트에 맞는 전문 LLM로 보낸다. 컨텍스트 관리에도 쓸 수 있다. 오래된 도구 호출 결과를 컨텍스트에 유지할지, 잘라낼지, 버릴지를 결정 모델에 물으면 세션 재작성 없이 정확한 판단으로 컨텍스트를 줄일 수 있다는 것이 TypeSafe의 설명이다.

Cloudflare는 여기에 한 발 더 나아가 Clef를 고객의 프라이빗 데이터로 강화학습(RL) 파인튜닝해주는 서비스도 함께 발표했다. AI Gateway, Workers AI, Containers와 새 Trainer 컴포넌트를 결합한 파이프라인으로, 현재는 포워드 디플로이드 엔지니어를 통한 디자인 파트너 방식으로 시작해 추후 셀프서비스로 확대할 예정이다.

## 도입 전 점검할 4가지

1. **확률 보정(calibration)을 먼저 검증하라.** 결정 모델의 가치는 확률이 신뢰할 만하다는 데 있다. 자기 데이터에서 상위 20% 신뢰 구간의 실제 정확도가 실제로 높은지 확인하지 않으면 임계값 기반 자동화는 성급하다.
2. **답변 공간 설계가 절반이다.** 옵션 목록이 모호하게 겹치면 확률이 뭉개진다. 상호배타적이고 빠짐없는 선택지를 설계하는 것이 프롬프트 엔지니어링보다 중요하다.
3. **생성이 필요한 곳에는 쓰지 마라.** 요약, 코드 작성, 장문 추론은 여전히 LLM의 영역이다. 결정 모델은 설명문을 생성하지 않는다. 왜 그렇게 판단했는지 이유가 필요한 워크플로우라면 구조가 다르다.
4. **벤더 수치는 벤더 측정이다.** 이 글에 인용된 지연·벤치마크 수치는 Cloudflare와 TypeSafe가 각자 자사 환경에서 측정해 발표한 값이다. 실제 도입 시에는 자체 트래픽으로 A/B를 돌려 확인하는 것이 원칙이다.

## 결론

- 결정 모델은 텍스트 생성을 포기하는 대신 타입된 확률과 밀리초 단위 지연을 얻은 새로운 모델 계층이다. 에이전트의 라우팅·게이팅·분류라는 핫패스을 겨냥한다.
- Cloudflare Clef·Clef-flash는 이 카테고리의 첫 오픈 웨이트(Apache 2.0) 진입자다. Clef-flash의 미디언 38.8ms는 에지 네트워크와 결합되어 요청 경로 안에 넣을 수 있는 수준이다.
- 다만 지식 집약 벤치마크에서는 생성형·호스티드 모델이 여전히 앞선다. 결정 모델은 LLM을 대체하는 것이 아니라 LLM 호출 앞단의 비용과 지연을 걷어내는 분업 층이다.
- 당장 시도해볼 첫 단계는 간단하다. 자기 서비스에서 가장 빈도 높은 분기 판단 하나(긴급도, 인텐트, 정책 위반 여부 등)를 골라 Workers AI 무료 티어로 Clef-flash를 호출해보고, 기존 LLM 호출과 확신도·지연·비용을 비교하는 것이다. 하루면 충분하다.

**참고 자료**
- Cloudflare 공식 블로그: Introducing Clef: our open-source decision models — https://blog.cloudflare.com/clef-decision-models/
- Cloudflare Developers Changelog (2026-10-01): Clef on Workers AI — https://developers.cloudflare.com/changelog/post/2026-10-01-clef-workers-ai/
- TypeSafe AI 공식 문서: System One — https://docs.typesafe.ai/concepts/system-one
- TypeSafe AI 블로그: Introducing System One Models & Jev — https://typesafe.ai/blog/introducing-system-one-models-and-jev
- MarkTechPost (2026-10-01): Cloudflare Releases Clef and Clef-flash — https://www.marktechpost.com/2026/10/01/cloudflare-releases-clef-and-clef-flash/
