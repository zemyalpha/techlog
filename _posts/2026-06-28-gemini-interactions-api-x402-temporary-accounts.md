---
title: "Gemini Interactions API와 x402로 보는 AI 에이전트의 상태·배포·결제 분리"
date: "2026-06-28"
keywords: ["Gemini Interactions API", "temporary accounts", "x402", "pay-per-crawl", "AI 에이전트"]
lang: "ko"
description: "Google Gemini Interactions API, Cloudflare temporary accounts, x402, pay-per-crawl을 묶어 AI 에이전트의 상태·배포·결제 레이어를 분리하는 실전 설계를 설명합니다."
---

# Gemini Interactions API와 x402로 보는 AI 에이전트의 상태·배포·결제 분리

요즘 AI 에이전트 이야기를 보면 모델 성능보다 더 중요한 질문이 하나 보입니다. **이 에이전트는 어디에 상태를 두고, 어디서 배포되며, 무엇을 어떤 방식으로 결제하는가**입니다.

최근 발표들을 함께 보면 방향이 꽤 선명합니다. Google은 Gemini용 **Interactions API**를 일반 공개했고, 서버사이드 상태와 background execution, Managed Agents를 강조했습니다. Cloudflare는 AI 에이전트를 위한 **temporary accounts**를 내놔서, 사람이 먼저 계정을 만들지 않아도 에이전트가 배포를 시작할 수 있게 했습니다. Coinbase의 **x402**는 HTTP 402 Payment Required를 이용해 기계가 읽을 수 있는 결제 흐름을 제안하고, Cloudflare의 **pay-per-crawl**은 AI 크롤러의 접근 자체를 가격화합니다.

이 네 가지를 한 문장으로 묶으면 이렇습니다. **상태는 런타임이 맡고, 배포는 임시 자격이 맡고, 결제는 HTTP가 맡고, 접근 정책은 크롤링 레이어가 맡는다.**

## 1) 상태는 프롬프트가 아니라 런타임에 두는 게 낫습니다

Google 문서와 블로그에 따르면 Interactions API는 Gemini 모델과 에이전트를 위한 기본 인터페이스이며, 서버사이드 상태와 background execution을 지원합니다. 이 말은 실무적으로 중요합니다. 매 요청마다 대화 이력을 통째로 다시 보내는 구조보다, **작업 단위 상태를 서버에 남기는 구조**가 훨씬 덜 취약하기 때문입니다.

```yaml
runtime:
  model_api: Gemini Interactions API
  state: server_side
  execution: background
  use_case:
    - planner
    - tool runner
    - long task orchestration
```

핵심은 “모델을 한 번 더 부른다”가 아니라 “작업을 이어서 굴린다”입니다. 상태가 서버에 있으면 재시도, 중단 복구, 장시간 작업 분리가 쉬워집니다.

## 2) 배포는 계정보다 먼저, 소유권은 나중에

Cloudflare의 temporary accounts는 이 문제를 정면으로 다룹니다. 에이전트가 배포를 시도할 때 브라우저 OAuth, 토큰 복붙, MFA 같은 인간용 절차에서 멈추지 않게 하려는 것입니다.

```bash
wrangler deploy --temporary
```

이 패턴의 장점은 단순합니다. 에이전트가 먼저 실행해 보고, 사람이 나중에 claim하면 됩니다. 즉, **“누가 소유자인가?”보다 “지금 이 일을 자동으로 돌릴 수 있는가?”를 먼저 묻는 구조**입니다.

## 3) 결제는 x402, 접근은 pay-per-crawl로 분리합니다

x402는 Coinbase가 제안한 오픈 결제 프로토콜로, HTTP 위에서 자동 stablecoin 결제를 가능하게 하려는 표준입니다. 공식 문서와 프로젝트 사이트는 x402가 `402 Payment Required`를 이용해 서비스가 계정이나 세션 없이도 유료 접근을 제공할 수 있게 한다고 설명합니다.

Cloudflare pay-per-crawl은 API 결제라기보다 **웹 콘텐츠 접근 정책**에 가깝습니다. 콘텐츠 소유자는 AI 크롤러에게 가격을 붙일 수 있습니다.

```ts
async function fetchPaidResource(url: string) {
  const res = await fetch(url);

  if (res.status === 402) {
    return { ok: false, reason: "payment_required" };
  }

  return { ok: true, body: await res.text() };
}
```

이 코드는 단순한 예시지만 메시지는 분명합니다. **유료 API와 유료 콘텐츠는 ‘토큰이 있으면 통과’보다 ‘HTTP 상태로 가격 신호를 주는 방식’이 에이전트 친화적**입니다.

## 4) 실전 아키텍처는 네 칸으로 나누면 됩니다

```yaml
agent_stack:
  state:
    layer: runtime
    choice: Gemini Interactions API
  identity:
    layer: deployment
    choice: temporary accounts
  payment:
    layer: api_access
    choice: x402
  access:
    layer: content_control
    choice: pay-per-crawl
```

이렇게 나누면 운영 포인트가 아주 명확해집니다.

- 상태 레이어는 대화·툴 실행·재시도를 담당합니다.
- 배포 레이어는 임시 권한과 소유권 이전을 담당합니다.
- 결제 레이어는 API 과금을 담당합니다.
- 접근 레이어는 공개 웹의 읽기 권한을 담당합니다.

제가 보기엔 2026년의 AI 에이전트 경쟁력은 모델 이름보다 **경계 설계**에 더 가깝습니다. 어떤 모델을 쓰느냐보다, 상태를 어디에 두고, 배포를 어떻게 시작하고, 무엇을 돈 받고 열어줄지 결정하는 쪽이 제품 차이를 만듭니다.

## 결론: 에이전트 제품은 “한 번에 다”가 아니라 “레이어별 책임 분리”가 답입니다

- Interactions API는 에이전트의 **상태를 런타임으로 이동**시킵니다.
- Temporary accounts는 에이전트가 **먼저 배포하고 나중에 소유권을 확정**하게 합니다.
- x402는 유료 API 접근을 **HTTP 결제 흐름**으로 바꿉니다.
- pay-per-crawl은 공개 콘텐츠의 접근을 **가격화**할 수 있게 합니다.

오늘 바로 점검할 질문은 하나입니다. 여러분의 에이전트는 아직도 로그인, 결제, 실행 권한을 한 층에 묶고 있지 않습니까? 그렇다면 먼저 세 칸으로 쪼개 보십시오. 그 순간부터 운영이 훨씬 단순해집니다.

## 참고

- Google AI Blog: [Google AI Studio’s Interactions API for Gemini models and agents](https://blog.google/innovation-and-ai/technology/developers-tools/interactions-api-general-availability/)
- Cloudflare Blog: [Temporary Cloudflare Accounts for AI agents](https://blog.cloudflare.com/temporary-accounts/)
- Coinbase Docs: [x402 Overview](https://docs.cdp.coinbase.com/x402/welcome)
- x402 Project: [Internet-Native Payments Standard](https://www.x402.org/)
- Cloudflare Blog: [Introducing pay per crawl](https://blog.cloudflare.com/introducing-pay-per-crawl/)

