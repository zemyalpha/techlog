---
title: "Gemini 4 Argon 1M 토큰 출력 실전 가이드 — 12월 1일 가격 인상 전에 잡아야 할 것들"
date: "2026-11-07"
keywords: ["Gemini 4 Argon", "Gemini API", "long decode", "Google DeepMind", "LLM 가격", "AI 코딩"]
lang: "ko"
description: "Gemini 4 Argon의 1M 토큰 long decode를 Python으로 쓰는 법과 11월 30일까지 유효한 introductory 가격의 실제 비용을 공식 문서 기반으로 정리했다."
---

# Gemini 4 Argon 1M 토큰 출력 실전 가이드 — 12월 1일 가격 인상 전에 잡아야 할 것들

Google DeepMind는 2026년 9월 30일 차세대 플래십 모델 Gemini 4 Argon을 발표하고, 10월 13일부터 AI Studio와 Gemini platform API, Google Antigravity, Gemini 앱 전반에 전면 공개했다. 이 모델의 핵심 차별점은 두 가지다. 첫째, 입력과 출력, 심지어 내부 추론(thinking) 토큰까지 하나의 창을 공유하는 **통합 1M(1,048,576) 토큰 컨텍스트**다. 둘째, 응답 하나에서 수십만 토큰의 추론을 수행하는 "long decode" 워크로드를 염두에 둔 설계다.

그런데 놓치기 쉬운 것이 하나 있다. 현재 적용되는 introductory 가격 — 입력 100만 토큰당 $2, 출력 $10 — 은 **2026년 11월 30일까지만 유효**하고, 12월 1일부터 입력 $4, 출력 $20으로 정확히 두 배 인상된다(공식 가격 페이지 기준). 오늘 기준으로 3주 남았다. 이 글에서는 long decode를 실제로 쓰는 코드, 벤치마크의 진짜 의미, 그리고 인상 전후 비용 차이를 공식 문서와 제3자 데이터 기반으로 정리한다.

## Argon의 스펙에서 실무자가 봐야 할 부분

모델 코드는 `gemini-4-argon`이며 텍스트·이미지·오디오·비디오·PDF를 입력받고 텍스트를 출력한다. 흥미로운 점은 "지원하지 않는 것" 목록이 꽤 길다는 사실이다.

지원: thinking(low/medium/high), function calling, computer use, code execution, file search, URL context, search grounding, structured outputs, implicit caching, batch/flex/priority inference
미지원: Live API(실시간 스트리밍), 이미지 생성, 오디오 생성, **explicit caching**

두 가지가 실무에 바로 영향을 준다. 첫째, 캐싱은 **implicit(자동) 방식만** 지원한다. 기존에 explicit cache 객체를 만들어 관리하던 파이프라인이 있다면 그 코드는 무의미해진다. 대신 반복되는 프롬프트 접두사는 자동으로 캐시되어 캐시 입력 가격($0.10/M, 12월 1일부터 $0.20/M)이 적용된다. 둘째, **무료 티어가 없다.** 공식 문서는 "처리되는 모든 토큰이 과금되며, 여기에는 thinking 토큰과 그라운딩 사용 시 내부 도구 시스템 명령·함수 선언도 포함된다"고 명시한다. 즉 이 모델은 실험이 아니라 예산 관리 대상이다.

## Long decode 실전: 기본 64K를 넘는 방법

기본적으로 Argon은 응답당 thinking과 출력을 합쳐 65,536 토큰까지만 생성한다. 대부분의 요청은 이 안에서 끝난다. 문제는 "소수정리의 완전한 증명"이나 "대규모 리팩토링 계획 수립"처럼 모델이 64K 전체를 thinking으로 소진하고 답을 내기 전에 끊기는 경우다.

Long decode는 `max_output_tokens`를 65,536 초과로 설정하면 활성화된다. 수십만 토큰 생성은 단일 HTTP 요청이 열려 있을 수 있는 시간보다 길기 때문에, 생성은 **hop 단위로 분할**된다 — 백그라운드/스트리밍 hop당 64K, 비스트리밍 hop당 32K씩 이전 지점에서 이어서 생성한다. 공식 문서의 Python 예시(Interactions API)를 기반으로 정리하면 다음과 같다.

```python
import time
from google import genai

client = genai.Client()

interaction = client.interactions.create(
    model="gemini-4-argon",
    input="소수정리의 완전하고 엄밀한 증명을 작성하라.",
    background=True,
    generation_config={
        "max_output_tokens": 262144,   # 64K 초과 → long decode
        "thinking_level": "high",
    },
)

while interaction.status in ("in_progress", "queued"):
    time.sleep(5)
    interaction = client.interactions.get(id=interaction.id)

# hop마다 model_output step이 추가되므로 전부 이어 붙인다
text = "".join(
    content.text
    for step in interaction.steps
    if step.type == "model_output"
    for content in step.content
    if content.type == "text"
)
print("Status:", interaction.status)
print("Usage:", interaction.usage)
```

핵심 디테일 세 가지:

1. **`max_output_tokens`는 반드시 65,536 초과로.** 공식 문서는 에이전트 프레임워크나 평가 도구에 기본으로 숨어 있는 낮은 값(예: 8,192)이 thinking 도중 모델을 끊어버리는 흔한 원인이라고 경고한다.
2. **출력 한도는 "1,048,576 − 입력 토큰"이다.** 입력을 900K 넣으면 출력은 100K 남짓밖에 못 쓴다.
3. **쓸데없이 켜지 마라.** 공식 가이드 기준으로 long decode가 필요한 것은 어려운 수학·형식 증명, 최상위 난도 코딩/에이전트 태스크, 출력 한도에서 끊기는 평가 도구뿐이다. 채팅·요약·분류는 32K 안에 끝나고, 툴을 짧게 여러 번 호출하는 에이전트는 오히려 긴 출력이 공유 컨텍스트 창을 빠르게 채운다.

## 비용 계산: 12월 1일이 실제로 의미하는 것

Introductory 가격과 인상 후 가격을 같은 워크로드로 비교해보자. 예를 들어 "입력 50K + thinking·출력 합계 150K"를 하루 100회 실행하는 배치 작업을 가정하면 하루 입력 5M, 출력 15M 토큰이다.

- 11월 30일까지: 5M × $2 + 15M × $10 = **$160/일**
- 12월 1일부터: 5M × $4 + 15M × $20 = **$320/일**

월 환산으로는 약 $4,800 → $9,600이다. 여기서 빼먹지 말아야 할 것이 캐싱이다. 프롬프트 접두사(시스템 프롬프트, 레퍼런스 문서 등)가 매 요청에 반복된다면 그 부분은 $0.10/M(12월 이후 $0.20/M)으로 떨어진다. 입력 50K 중 40K가 반복 접두사라면 위 계산에서 입력 비용은 절반 이하로 줄어든다. **Argon에서 비용 최적화의 1차 관문은 implicit 캐시 히트율**이라고 봐도 된다.

참고로 제3자 추적 사이트 Artificial Analysis는 Argon(High)의 Intelligence Index 평균 태스크 비용을 $9.95로 측정했다. 고성능 모델군의 중앙값 대비 비싼 편이며, 태스크당 평균 27.7분, 첫 토큰까지 38.73초, 출력 70 tok/s로 측정되었다. 즉 Argon은 **배치형 심층 작업용**이지, 사용자 앞에서 대기하는 인터랙티브 용도로는 느리다는 이야기다.

## 벤치마크 숫자를 제대로 읽기

Google 발표 기준(Argon 블로그 포스트) 주요 수치는 다음과 같다. 모두 Google 자체 발표이므로 출처를 달아두었다.

- **DeepSWE v1.1 77.9%** — 실제 장기 소프트웨어 엔지니어링 태스크 측정, 신기록(SOTA) 주장
- **Vals Index 1위** — 미국 GDP 기여도로 가중한 금융·코딩·법무·세무 성능 지수
- **AutomationBench 51.3% (Zapier)** — 비즈니스 기능별 종단 실행 벤치마크 1위
- **LVBench 91.7%** — 롱 비디오 이해 부문 SOTA 주장
- **CWE-bench v1 68%** — 취약점 수정 능력, 공동 1위

내부 사례도 인상적이다. Chrome 팀은 4,000만 줄 코드베이스에서 약 600만 줄짜리 레거시 브라우저 기능 폴더를 코어에서 분리했고, libgav1의 Rust 포트에서 3.2만 줄 SIMD 코드를 재작성해 기존 Rust 포트 대비 2.7배 빠른 메모리 안전 디코더를 만들었다고 한다. Fuchsia Zircon 커널의 C/C++→Rust 마이그레이션은 80만 줄 이상 규모로 진행 중이라고 밝혔다. 다만 이는 전부 **Google 자체 발표 사례**이며 독립 검증된 수치가 아니라는 점은 유념해야 한다. 특히 "Antigravity에서 에이전트 코드 제출 35% 증가, 롤백 1/3 감소" 같은 수치는 측정 방법론이 공개되지 않았다.

한 가지 짚을 점: Artificial Analysis의 Intelligence Index에서 Argon(High)은 50점으로 중앙값(37)보다 훨씬 높지만 전체 69개 모델 중 15위다. 특정 도메인(장기 코딩, 사이버 방어)에 특화된 모델이라는 게 내 판단이며, 범용 챗봇 지표만 보고 평가절하하는 것도, 발표 자료의 SOTA 주장을 그대로 신뢰하는 것도 둘 다 위험하다.

## 결론

- Gemini 4 Argon은 통합 1M 토큰 창과 long decode로 "한 번의 궤적에서 깊게 푸는" 워크로드에 특화된 모델이다.
- **12월 1일부터 입력 $4/출력 $20으로 두 배 인상**된다. 대량 워크로드라면 지금이 예산 측정과 캐싱 전략 점검의 마지막 기한이다.
- long decode는 `max_output_tokens` 65,536 초과 + 백그라운드 실행으로 쓴다. 단, 대부분의 태스크는 64K 안에 끝나니 남발하지 말 것.
- implicit caching만 지원하므로 비용 절감의 핵심은 반복 프롬프트 접두사의 캐시 히트율이다.
- 벤치마크 SOTA 주장은 Google 자체 발표임을 감안하고, 실제 도입 전에는 본인의 워크로드로 직접 평가하라.

당장 해볼 것: AI Studio에서 `gemini-4-argon`으로 현재 사용 중인 가장 어려운 태스크 하나를 thinking level별(low/high)로 돌려보고, 11월 30일 전에 인상 후 가격 기준으로 월 비용을 계산해보라. 두 배 인상이 감당 가능한지 여부가 이 모델 채택의 실질적 기준이 될 것이다.

**주요 출처**
- Google 블로그: Gemini 4 Argon 발표(2026-09-30) 및 전면 공개(2026-10-13)
- Gemini API 공식 문서: 모델 페이지, pricing, long decode, thinking
- llm-stats.com, Artificial Analysis(제3자 가격·성능 추적)
