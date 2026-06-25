---
title: "AI 에이전트에 402를 돌려주는 법: x402와 pay-per-crawl"
date: "2026-06-25"
keywords: ["AI 에이전트", "x402", "pay-per-crawl", "MCP", "402 Payment Required"]
lang: "ko"
description: "MCP로 권한을 나누고, x402로 결제 게이트를 만들고, Cloudflare pay-per-crawl과 Sumsub MCP 사례로 AI 에이전트 과금·감사 구조를 설계하는 법을 정리합니다."
---

# AI 에이전트에 402를 돌려주는 법: x402와 pay-per-crawl

AI 에이전트를 쓰는 방식이 한 단계 더 바뀌고 있습니다. 예전에는 “어떤 도구를 붙일까”가 핵심이었다면, 이제는 “누가 접근할 수 있고, 무엇에 비용을 붙이며, 어떤 행동을 감사할 것인가”가 더 중요합니다. 특히 콘텐츠, API, 컴플라이언스처럼 값이 분명한 작업은 더 이상 공짜로 열어둘 이유가 없습니다.

최근 흐름이 이 점을 잘 보여줍니다. Cloudflare는 콘텐츠 소유자가 AI 크롤러의 접근에 가격을 붙일 수 있는 pay-per-crawl을 발표했고, x402는 HTTP 402 Payment Required를 기계가 읽는 결제 핸드셰이크로 바꾸려는 인터넷 결제 표준을 내세웁니다. 동시에 MCP는 에이전트가 외부 시스템을 안전하게 다루는 인터페이스로 자리 잡고 있습니다. 한 줄로 요약하면, **MCP는 행동의 입구를 정하고, x402는 그 입구에 가격을 붙이며, pay-per-crawl은 그 모델이 실제 시장에 필요하다는 신호**입니다.

## 1) MCP는 권한, x402는 결제, pay-per-crawl은 시장 신호입니다

세 개를 섞어 보면 역할이 꽤 분명합니다.

- **MCP**: 에이전트가 무엇을 읽고, 무엇을 바꿀 수 있는지 정합니다.
- **x402**: 그 접근이 무료인지 유료인지 기계가 판독할 수 있게 만듭니다.
- **pay-per-crawl**: 웹 콘텐츠도 이제는 “허용/차단”만이 아니라 “허용 후 과금”으로 설계될 수 있음을 보여줍니다.

이 구분이 중요한 이유는 실무에서 보안과 과금이 자주 뒤섞이기 때문입니다. 인증이 끝났다고 해서 비용 문제가 끝나는 것은 아니고, 비용을 냈다고 해서 모든 행위가 허용되는 것도 아닙니다. 그래서 저는 다음처럼 분리하는 편이 좋다고 봅니다.

1. **인증(Authentication)**: 누구인지 확인한다.
2. **인가(Authorization)**: 무엇을 할 수 있는지 정한다.
3. **결제(Payment)**: 얼마를 내야 하는지 정한다.
4. **감사(Audit)**: 누가 언제 무엇을 했는지 남긴다.

이 4개를 한 덩어리로 묶으면 운영이 무너집니다. 반대로 분리하면, 에이전트는 더 유연해지고 사람은 통제권을 유지할 수 있습니다.

## 2) 실전 구조: 402 응답을 “거절”이 아니라 “요금 안내”로 쓰기

x402가 흥미로운 이유는 단순한 차단이 아니라 **재시도 가능한 요금 안내**를 만든다는 점입니다. x402 사이트와 GitHub README는 클라이언트가 HTTP 요청을 보내고, 서버가 `402 Payment Required`와 가격 정보를 돌려준 뒤, 클라이언트가 결제하고 다시 시도하는 흐름을 보여줍니다. 또 `@x402/express`, `@x402/next`, `@x402/mcp` 같은 SDK도 제공합니다.

서버는 대략 이런 식으로 설계할 수 있습니다.

```ts
import express from "express";

const app = express();

function verifyPaymentProof(req) {
  const proof = req.header("X-Payment-Proof");
  return Boolean(proof && proof.length > 20);
}

app.get("/report", (req, res) => {
  if (!verifyPaymentProof(req)) {
    return res.status(402).json({
      code: "PAYMENT_REQUIRED",
      resource: "/report",
      price: "0.10 USDC",
      hint: "결제 후 X-Payment-Proof 헤더와 함께 다시 요청하세요",
    });
  }

  res.json({
    ok: true,
    data: "유료 리포트 본문",
  });
});
```

이 코드의 포인트는 화려한 결제 SDK가 아닙니다. 핵심은 **무료와 유료를 같은 엔드포인트 안에서 구분하는 것**입니다. 그러면 같은 API라도 세 가지 계층으로 나뉩니다.

- 미결제: 요약 정보만 제공
- 결제 완료: 원문/고해상도/대량 조회 허용
- 고위험 작업: 결제와 별도로 MCP 권한 확인

실제로는 여기서 로그도 함께 남겨야 합니다. 어떤 클라이언트가 어떤 리소스에 몇 번 402를 받았는지 보면, 가격이 너무 높았는지, 봇이 과도하게 긁는지, 또는 정상 사용자가 결제 플로우에서 막혔는지 알 수 있습니다.

## 3) MCP는 “무엇을 할 수 있는가”를 좁혀준다

Sumsub의 MCP 서버 문서는 AI 에이전트가 단일 서버 URL로 계정에 연결되고, 역할 권한 안에서 작업할 수 있다고 설명합니다. 또 온보딩, 컴플라이언스 운영, 거래 모니터링, 설정 탐색 같은 워크플로를 에이전트 기반으로 묶을 수 있다고 말합니다. Google Cloud도 공식 MCP 지원을 발표하면서, MCP가 단순한 실험이 아니라 서비스 연결 방식으로 퍼지고 있음을 보여줬습니다.

여기서 중요한 교훈은 하나입니다. **결제는 접근을 통제하고, MCP는 행동을 통제한다**는 점입니다.

예를 들어 이런 식의 정책이 필요합니다.

```yaml
mcp:
  server: compliance
  role: analyst
  allowed_actions:
    - read_policy
    - draft_workflow
    - inspect_risk_rules
  requires_human_approval:
    - publish_rules
    - change_thresholds
    - delete_audit_log
payment:
  mode: pay_per_request
  protected_resources:
    - /report
    - /bulk-export
    - /historical-data
audit:
  log_402: true
  log_mcp_actions: true
  retain_days: 90
```

이렇게 나누면 좋은 점이 있습니다. 에이전트가 실수했을 때 복구 범위가 작아집니다. 그리고 운영자가 나중에 “왜 이 요청은 돈을 받았는지”, “왜 이 설정은 변경되지 않았는지”를 추적할 수 있습니다.

## 4) 앞으로의 제품 설계는 ‘기능’보다 ‘계층’이 더 중요합니다

제가 보기엔 앞으로의 에이전트 제품은 기능 목록보다 계층 설계가 더 중요해질 겁니다.

- 공개 읽기 레이어: 검색, 요약, 미리보기
- 유료 읽기 레이어: 원문, 대량 조회, 고급 필터
- 실행 레이어: MCP로 제한된 작업 수행
- 승인 레이어: 삭제, 배포, 정책 변경

이 구조는 웹사이트에도, SaaS에도, 내부 업무 자동화에도 그대로 적용됩니다. Cloudflare의 pay-per-crawl은 웹 콘텐츠 쪽에서 이 방향을 보여주고, x402는 결제의 언어를 HTTP로 옮기려 하고, MCP는 에이전트의 행동 범위를 제한합니다. 세 개를 한꺼번에 보면, AI 에이전트 시대의 진짜 인프라는 모델이 아니라 **접근·결제·감사 레이어**라는 결론에 도달합니다.

## 결론: 에이전트는 똑똑해지는 만큼 더 잘 과금되어야 합니다

- Cloudflare의 pay-per-crawl은 AI 시대에 콘텐츠 접근을 가격화하는 흐름을 보여줍니다.
- x402는 HTTP 402를 이용해 기계가 읽을 수 있는 결제 흐름을 제안합니다.
- MCP는 에이전트가 외부 시스템을 만질 때 권한 범위를 좁혀 줍니다.
- 좋은 설계는 인증, 결제, 권한, 감사를 한 줄로 묶지 않고 분리합니다.
- 가장 먼저 할 일은 “무료/유료/승인 필요”를 엔드포인트와 작업 단위로 나누는 것입니다.

오늘 당장 해볼 수 있는 첫 단계는 하나입니다. 여러분이 운영하는 API나 콘텐츠 중 하나를 골라서, **읽기 전용 / 유료 읽기 / 승인 필요 작업**으로 나눠 보십시오. 그 순간부터 AI 에이전트는 단순한 자동화 도구가 아니라, 수익과 통제가 공존하는 시스템이 됩니다.

## 참고한 자료

- Cloudflare Blog: [Introducing pay per crawl: Enabling content owners to charge AI crawlers for access](https://blog.cloudflare.com/introducing-pay-per-crawl/)
- TechCrunch: [Cloudflare launches a marketplace that lets websites charge AI bots for scraping](https://techcrunch.com/2025/07/01/cloudflare-launches-a-marketplace-that-lets-websites-charge-ai-bots-for-scraping/)
- x402: [Internet-Native Payments Standard](https://www.x402.org/)
- x402 GitHub: [x402-foundation/x402](https://github.com/x402-foundation/x402)
- Sumsub Newsroom: [Sumsub Becomes First Verification Platform to Enable AI Agents to Build Compliance Setup](https://sumsub.com/newsroom/sumsub-becomes-first-verification-platform-to-enable-ai-agents-to-build-compliance-setup/)
- Sumsub Docs: [MCP server](https://docs.sumsub.com/docs/mcp-server)
- Google Cloud Blog: [Announcing official MCP support for Google services](https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services)
