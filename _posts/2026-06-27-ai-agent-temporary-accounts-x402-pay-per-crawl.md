---
title: "AI 에이전트는 이제 계정·결제·접근을 따로 갖는다: temporary accounts, x402, pay-per-crawl"
date: "2026-06-27"
keywords: ["AI 에이전트", "temporary accounts", "x402", "pay-per-crawl", "Cloudflare Workers"]
lang: "ko"
description: "Cloudflare temporary accounts, x402, pay-per-crawl을 묶어 AI 에이전트의 계정·결제·접근 레이어를 분리하는 실전 설계 패턴을 설명합니다."
---

# AI 에이전트는 이제 계정·결제·접근을 따로 갖는다: temporary accounts, x402, pay-per-crawl

AI 에이전트 제품을 만들 때 가장 흔한 실수는 모든 문제를 하나의 로그인이나 하나의 API 키로 해결하려는 것입니다. 그런데 2026년의 흐름을 보면, 이 방식은 점점 맞지 않습니다. 에이전트는 **먼저 실행되어야 하고**, 필요한 것만 **유료로 접근해야 하며**, 실제로 무엇을 **읽고/쓸 수 있는지**는 별도 규칙으로 제한되어야 합니다.

최근 발표를 묶어 보면 이 경향이 꽤 분명합니다. Cloudflare는 에이전트가 먼저 배포를 시도할 수 있도록 `wrangler deploy --temporary` 기반의 temporary accounts를 내놨고, Cloudflare pay-per-crawl은 AI 크롤러 접근을 가격화할 수 있게 했습니다. 여기에 Coinbase의 x402는 HTTP 402 Payment Required를 이용해 기계가 읽을 수 있는 결제 흐름을 제안합니다. 저는 이 셋을 각각 **계정**, **결제**, **접근** 레이어로 보는 편이 가장 실용적이라고 봅니다.

## 1) temporary accounts는 “먼저 실행하고 나중에 소유권을 가져오는” 방식입니다

Cloudflare의 temporary accounts for AI agents는 배경 실행 중인 에이전트가 브라우저 로그인, 복붙, 짧은 승인 창에 막히지 않도록 만든 장치입니다. Cloudflare 문서에 따르면 에이전트는 `wrangler deploy --temporary`로 Worker를 바로 배포할 수 있고, 이 임시 배포는 **60분 동안 유지**된 뒤 사람이 claim하면 영구 소유권으로 넘어갑니다.

이건 단순한 편의 기능이 아닙니다. 에이전트 제품에서 계정 생성은 종종 가장 큰 마찰입니다. “누가 이 배포의 주인인가?”를 너무 일찍 묻지 말고, 먼저 일을 시켜 보고 나중에 소유권을 정하자는 발상입니다.

```bash
wrangler deploy --temporary
# 60분 안에 claim해서 영구 계정으로 전환
```

이 패턴은 특히 다음 상황에 잘 맞습니다.

- 에이전트가 데모용 사이트를 즉시 띄워야 할 때
- 사람이 마지막 승인만 하면 되는 내부 자동화일 때
- trial-and-error가 많은 배포/실험 흐름일 때

## 2) x402는 “요청 → 402 → 결제 → 재시도”를 HTTP에 붙입니다

x402는 Coinbase가 만든 오픈 결제 프로토콜로, **HTTP 위에서 즉시 자동 stablecoin 결제**를 가능하게 하는 것을 목표로 합니다. 공식 문서는 x402가 계정, 세션, 복잡한 인증 없이도 서비스가 API와 디지털 콘텐츠를 monetization할 수 있게 해준다고 설명합니다. 핵심은 `402 Payment Required`를 “막는 응답”이 아니라 **결제를 유도하는 응답**으로 쓰는 것입니다.

클라이언트 흐름은 대략 이렇습니다.

```ts
async function fetchPaidResource(url: string) {
  const res = await fetch(url);

  if (res.status === 402) {
    const price = res.headers.get("X-Price") ?? "unknown";
    throw new Error(`payment required: ${price}`);
  }

  return await res.text();
}
```

이 예시는 단순하지만 핵심은 분명합니다. 유료 API는 “토큰이 있으면 통과” 같은 내부 규칙보다, **HTTP 표준 응답으로 가격 신호를 내보내는 쪽**이 에이전트에게 더 읽기 쉽습니다. 인간도 이해하고, 봇도 이해합니다.

## 3) pay-per-crawl은 공개 웹의 읽기 권한에 가격을 붙입니다

Cloudflare의 pay-per-crawl은 콘텐츠 소유자가 AI 크롤러의 접근에 가격을 붙일 수 있게 합니다. Cloudflare 문서와 블로그는 AI crawler가 요청할 때 **request headers로 payment intent를 제시해 200을 받거나**, 아니면 **402 Payment Required와 가격 정보**를 받는다고 설명합니다. 또한 Cloudflare가 pay-per-crawl의 merchant of record 역할을 맡습니다.

이건 x402와 닮았지만 완전히 같지는 않습니다. x402는 범용 결제 프로토콜이고, pay-per-crawl은 **웹 콘텐츠 접근 정책**에 더 가깝습니다. 즉, x402는 결제 언어이고 pay-per-crawl은 그 언어를 콘텐츠 접근에 적용한 사례라고 보면 이해가 쉽습니다.

## 4) 실전 설계는 세 레이어를 분리하는 것입니다

저라면 AI 에이전트 서비스를 아래처럼 나눕니다.

```yaml
agent_stack:
  identity:
    mode: temporary_accounts
    purpose: "먼저 실행, 나중에 claim"
  payment:
    mode: x402
    purpose: "유료 API와 디지털 콘텐츠 결제"
  access:
    mode: pay_per_crawl
    purpose: "읽기 트래픽의 허용/과금/차단"
  audit:
    logs:
      - payment_events
      - crawl_events
      - deployment_claims
```

이렇게 분리하면 운영이 훨씬 단순해집니다.

- 계정 레이어는 소유권 문제만 다룹니다.
- 결제 레이어는 돈의 흐름만 다룹니다.
- 접근 레이어는 무엇을 읽고 쓸 수 있는지만 다룹니다.
- 감사 레이어는 모든 행동을 추적합니다.

가장 중요한 점은, 이 셋을 하나의 로그인이나 하나의 비밀키에 묶지 않는 것입니다. 그렇게 하면 보안도 흐려지고, 과금도 흐려지고, 운영 로그도 흐려집니다.

## 결론: 에이전트 제품의 경쟁력은 모델보다 경계 설계입니다

- Cloudflare temporary accounts는 에이전트가 **먼저 실행**할 수 있게 해줍니다.
- x402는 유료 접근을 **HTTP 결제 흐름**으로 바꿉니다.
- Cloudflare pay-per-crawl은 공개 콘텐츠 접근을 **가격화**할 수 있게 해줍니다.
- 셋을 합치면, AI 에이전트 제품은 계정·결제·접근을 분리한 구조로 진화합니다.

오늘 당장 해볼 일은 하나입니다. 여러분 서비스에서 “로그인”, “유료 접근”, “실제 행동 권한”을 같은 층에 두고 있지 않은지 확인해 보십시오. 만약 섞여 있다면, 먼저 세 개를 분리하는 것만으로도 에이전트 제품의 품질이 한 단계 올라갑니다.

## 참고 자료

- Cloudflare Blog: [Temporary Cloudflare Accounts for AI agents](https://blog.cloudflare.com/temporary-accounts/)
- Cloudflare Docs: [Temporary accounts for AI agent deployments](https://developers.cloudflare.com/changelog/post/2026-06-19-temporary-accounts-for-agents/)
- Cloudflare Docs: [Claim deployments (temporary accounts)](https://developers.cloudflare.com/workers/platform/claim-deployments/)
- Cloudflare Blog: [Introducing pay per crawl: Enabling content owners to charge AI crawlers for access](https://blog.cloudflare.com/introducing-pay-per-crawl/)
- Cloudflare Docs: [What is Pay Per Crawl?](https://developers.cloudflare.com/ai-crawl-control/features/pay-per-crawl/what-is-pay-per-crawl/)
- Coinbase Docs: [x402 Overview](https://docs.cdp.coinbase.com/x402/welcome)
- x402: [Internet-Native Payments Standard](https://www.x402.org/)
- Sumsub PRNewswire: [Sumsub Becomes First Verification Platform to Enable AI Agents to Build Compliance Setup](https://www.prnewswire.com/news-releases/sumsub-becomes-first-verification-platform-to-enable-ai-agents-to-build-compliance-setup-302804191.html)
- Model Context Protocol: [What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro)
