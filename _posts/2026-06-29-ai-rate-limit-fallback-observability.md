---
title: "AI 기능은 모델보다 실패 처리가 먼저다: 429·fallback·관측성 설계"
date: "2026-06-29"
keywords: ["rate limits", "fallback", "observability", "AI Gateway", "OpenAI"]
lang: "ko"
description: "Gemini 접근 제한과 OpenAI·Cloudflare 문서를 바탕으로, AI 기능을 429·fallback·관측성 관점에서 설계하는 실전 패턴을 정리합니다."
---

# AI 기능은 모델보다 실패 처리가 먼저다: 429·fallback·관측성 설계

AI 기능을 넣은 팀이 가장 늦게 배우는 사실은, 모델 품질보다 **실패를 어떻게 다루느냐**가 사용자 경험을 더 크게 좌우한다는 점입니다. 2026년 6월 28일에는 CNBC가 FT 보도를 인용해 Google이 Meta의 Gemini 사용을 제한했다고 전했고, Reuters도 같은 내용을 보도했습니다. 이유는 단순합니다. 수요가 공급을 앞지르면, 더 좋은 모델보다 먼저 **안정적인 실패 처리**가 필요해집니다.

OpenAI 문서는 rate limit을 “정해진 기간 안에 사용자가 서비스에 접근할 수 있는 횟수 제한”으로 설명하고, RPM·RPD·TPM·TPD·IPM 같은 지표를 함께 봐야 한다고 말합니다. Cloudflare AI Gateway와 Kong AI Gateway도 같은 방향입니다. 둘 다 관측, 캐시, rate limiting, retry, fallback을 AI 인프라의 기본 기능으로 다룹니다. 이제 AI 앱은 “요청을 보내는 코드”가 아니라 “실패를 운영하는 코드”가 되어야 합니다.

## 1. 429는 에러가 아니라 운영 신호입니다

429를 받았다고 해서 무조건 재시도하면 안 됩니다. 먼저 확인할 것은 세 가지입니다.

- 지금 막힌 이유가 **요청 수(RPM)** 인지, **토큰 수(TPM)** 인지
- 이 요청이 **즉시 응답**이어야 하는지, **대기열 처리**로 바꿀 수 있는지
- fallback이 **품질 하락**을 감수할 만큼 중요한 경로인지

```python
import random
import time

class RateLimitError(Exception):
    pass

def backoff_seconds(attempt: int, base: float = 0.5, cap: float = 8.0) -> float:
    delay = min(cap, base * (2 ** attempt))
    jitter = 0.5 + random.random() * 0.5
    return delay * jitter


def call_with_retry(callable_fn, max_attempts: int = 5):
    last_err = None
    for attempt in range(max_attempts):
        try:
            return callable_fn()
        except RateLimitError as err:
            last_err = err
            time.sleep(backoff_seconds(attempt))
    raise last_err
```

핵심은 무한 재시도가 아니라 **지수 백오프 + 지터**입니다. OpenAI 문서가 이 패턴을 권장하는 이유도, 동시에 몰린 요청이 더 큰 병목을 만들기 때문입니다.

## 2. fallback은 “싸게”가 아니라 “안전하게” 설계해야 합니다

모델을 하나 더 붙였다고 자동으로 안전해지지 않습니다. fallback은 보통 세 단계로 나눕니다.

```yaml
ai_request_policy:
  criticality:
    realtime_chat: primary_only
    internal_summarize: fallback_allowed
    batch_enrichment: queue_first
  retry:
    on: [429, 503]
    max_attempts: 4
    strategy: exponential_backoff_jitter
  fallback_chain:
    - primary_model
    - cheaper_model
    - cached_answer
```

여기서 중요한 건 순서입니다. 실시간 대화는 무리해서라도 품질을 지켜야 하지만, 내부 요약이나 배치 보강은 대체 모델로 내려보내도 됩니다. 반대로 대체 모델로도 품질이 크게 무너지면, 그 기능은 fallback보다 **비동기 큐**가 더 낫습니다.

## 3. 관측성은 비용 절감보다 먼저 붙여야 합니다

Cloudflare AI Gateway와 Kong AI Gateway가 공통으로 강조하는 건 관측입니다. 요청 수, 에러, 캐시 적중률, 모델별 비용을 한 곳에서 봐야 합니다. 그래야 “왜 느린가”가 아니라 “어디서 새는가”를 찾을 수 있습니다.

실무 체크포인트는 이렇습니다.

- 요청 로그에 **모델명, 토큰 수, 응답 시간, 실패 코드**를 남긴다
- 캐시가 있으면 **캐시 히트/미스**를 분리한다
- fallback이 작동하면 **원래 모델과 대체 모델의 비용 차이**를 기록한다
- rate limit이 자주 걸리면 **사용자별/기능별 쿼터**를 따로 둔다

## 결론: AI 앱의 진짜 경쟁력은 복구력입니다

- OpenAI 문서처럼 rate limit은 RPM, TPM 같은 운영 지표로 봐야 합니다.
- Cloudflare AI Gateway와 Kong AI Gateway는 관측·캐시·retry·fallback을 기본기라고 봅니다.
- Gemini 접근 제한 보도는 “모델 공급”이 제품 리스크가 될 수 있음을 보여 줍니다.
- 따라서 AI 기능은 먼저 실패 처리, 그다음 품질 최적화 순서로 설계하는 편이 안전합니다.

오늘 바로 할 일은 하나입니다. 여러분의 AI 기능에서 **429가 났을 때의 경로**를 한 장의 다이어그램으로 그려 보십시오. 재시도, 대기열, fallback, 중단 중 무엇을 택할지 정해 두면, 운영은 훨씬 덜 흔들립니다.

## 참고

- OpenAI API Docs: [Rate limits](https://developers.openai.com/api/docs/guides/rate-limits)
- Cloudflare Docs: [AI Gateway](https://developers.cloudflare.com/ai-gateway/)
- Kong Docs: [AI Gateway](https://developer.konghq.com/ai-gateway/)
- CNBC: [Google limits Meta’s use of its Gemini AI models, FT reports](https://www.cnbc.com/2026/06/28/google-limits-metas-use-of-its-gemini-ai-models-ft-reports.html)
- Reuters: [Google limits Meta’s use of its Gemini AI models, FT reports](https://www.reuters.com/business/google-limits-metas-use-its-gemini-ai-models-ft-reports-2026-06-28/)
