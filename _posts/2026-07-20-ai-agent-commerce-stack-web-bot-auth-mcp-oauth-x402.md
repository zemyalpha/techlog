---
title: "AI 에이전트 상거래 스택 3층 구조: Web Bot Auth, MCP OAuth, x402"
date: "2026-07-20"
keywords: ["AI 에이전트", "Web Bot Auth", "MCP OAuth", "x402", "agentic commerce"]
lang: "ko"
description: "Web Bot Auth, MCP OAuth, x402를 기준으로 AI 에이전트의 신원·권한·결제를 분리 설계하는 방법과 구현 예시를 정리합니다."
---

# AI 에이전트 상거래 스택 3층 구조: Web Bot Auth, MCP OAuth, x402

AI 에이전트를 실제 서비스에 붙여 보면 금방 느끼게 되는 점이 하나 있습니다. 문제는 모델 하나가 아니라, **신원 확인, 권한 부여, 결제**가 한 흐름으로 엉켜 있다는 점입니다. 에이전트가 웹을 읽고, 도구를 호출하고, 유료 데이터까지 구매하는 순간에는 “누가 요청했는가”, “무엇을 해도 되는가”, “무엇에 돈을 내는가”를 서로 다른 레이어로 분리해야 합니다.

최근 흐름을 보면 이 분리가 더 이상 선택이 아닙니다. Cloudflare는 Web Bot Auth를 기반으로 signed agents를 소개하며, 자동화 트래픽도 암호학적으로 식별할 수 있어야 한다고 봤습니다. MCP는 HTTP 기반 전송에서 OAuth 계열 권한 부여를 전면에 두고, x402는 HTTP 402를 이용해 기계가 직접 결제하는 흐름을 제안합니다. 세 가지는 서로 경쟁하는 기능이 아니라, 에이전트 운영을 위한 서로 다른 경계입니다.

## 1. 신원: 이 요청이 정말 우리 에이전트인가

Web Bot Auth의 핵심은 단순합니다. 에이전트가 요청을 보낼 때 HTTP 메시지 자체에 서명을 넣고, 목적지 서버가 그 서명을 검증하게 하자는 것입니다. IETF draft는 자동화 클라이언트가 HTTP 요청을 암호학적으로 서명하고, 서버가 그 신원을 검증하는 아키텍처를 설명합니다. Cloudflare는 이를 signed agents로 확장해, 단순한 User-Agent 문자열보다 더 신뢰도 높은 분류를 시도합니다.

실전에서 중요한 점은 여기입니다.

- User-Agent는 위조될 수 있습니다.
- IP allowlist만으로는 운영 주체를 정확히 드러내기 어렵습니다.
- 서명은 요청 단위로 추적 가능하므로 감사와 차단이 쉽습니다.

즉, 에이전트의 “정체성”은 로그인 세션과 별개로 관리하는 편이 안전합니다. 브라우저 렌더링, 스크래핑, 자동 구매 같은 작업은 모두 같은 신원 계층을 공유할 수 있지만, 키는 역할별로 분리하는 편이 좋습니다.

## 2. 권한: 무엇을 해도 되는가

신원을 확인했다고 해서 모든 행동을 허용하면 안 됩니다. MCP authorization 스펙은 HTTP 기반 MCP 서버를 OAuth 2.1 resource server로 다루고, 클라이언트는 protected resource metadata로 권한 서버를 찾습니다. 그리고 범위가 부족하면 `WWW-Authenticate` 헤더의 scope challenge로 한 단계 더 높은 권한을 요청하게 만듭니다.

이 구조의 장점은 분명합니다. 에이전트가 “읽기”와 “쓰기”, “조회”와 “전송”, “테스트”와 “실거래”를 같은 토큰으로 뭉뚱그리지 않아도 됩니다. 예를 들어 다음처럼 권한을 나눌 수 있습니다.

```yaml
agent_policy:
  identity:
    require_signed_requests: true
    accepted_headers:
      - Signature
      - Signature-Input
      - Signature-Agent
  authorization:
    transport: http
    protocol: oauth2
    discovery: protected_resource_metadata
    scopes:
      read: ["read:*", "search:*"]
      write: ["write:ticket", "tool:send_email"]
      payment: ["pay:premium-data"]
```

여기서 핵심은 scope가 결제를 대체하지 않는다는 점입니다. 권한은 “할 수 있는 일”을 정하고, 결제는 “유료 자원에 접근할 조건”을 정합니다.

## 3. 결제: 무엇에 돈을 내는가

x402는 이 지점을 깔끔하게 분리합니다. 서버가 리소스를 내주기 전에 `402 Payment Required`를 반환하고, 클라이언트는 결제 정보를 붙여 다시 요청합니다. 공식 문서와 Coinbase 문서는 공통적으로 “계정, 세션, API 키 없이” 프로그램적으로 유료 자원에 접근하는 흐름을 강조합니다. Cloudflare의 x402 문서도 동일하게 402 응답, `PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE` 흐름을 설명합니다.

이 레이어는 MCP와 잘 맞습니다. 예를 들어 MCP 도구가 외부 데이터 API를 호출할 때, 권한이 있더라도 유료 엔드포인트라면 x402로 한 번 더 결제를 요구할 수 있습니다. 그러면 에이전트는 “접근 권한”과 “지불 의사”를 별개로 처리하게 됩니다.

```python
def handle_response(resp):
    if resp.status_code == 401:
        return "AUTHENTICATE"
    if resp.status_code == 403:
        return "STEP_UP_SCOPE"
    if resp.status_code == 402:
        return "PAY_AND_RETRY"
    return "OK"
```

이런 분기만 있어도 운영이 훨씬 단순해집니다. 401은 신원 문제, 403은 권한 문제, 402는 가격 문제로 읽히기 때문입니다.

## 4. 실전 설계: 한 번에 묶지 말고, 세 번 나눠라

가장 흔한 실수는 이 세 층을 하나의 “AI 인증”으로 뭉뚱그리는 것입니다. 그러면 감사 로그가 섞이고, revocation 기준이 흐려지고, 사고가 났을 때 원인을 분리하기 어렵습니다. 저는 다음 순서를 권합니다.

1. **서명 검증**으로 요청의 출처를 확인합니다.
2. **OAuth scope**로 에이전트의 행동 범위를 제한합니다.
3. **x402 결제**로 유료 리소스 접근을 별도 관리합니다.
4. 세 단계의 거절 사유를 로그에서 서로 다르게 저장합니다.

이렇게 하면 에이전트가 어떤 이유로 멈췄는지 바로 알 수 있습니다. 인증 실패인지, 권한 부족인지, 비용 부족인지가 구분되기 때문입니다.

## 결론

AI 에이전트가 커질수록 보안 문제는 모델 문제가 아니라 **시스템 경계 문제**가 됩니다.

- Web Bot Auth는 “누가 요청했는가”를 다룹니다.
- MCP OAuth는 “무엇을 해도 되는가”를 다룹니다.
- x402는 “무엇에 돈을 내는가”를 다룹니다.
- 이 셋을 분리하면 감사, 차단, 정산이 모두 쉬워집니다.

다음 단계는 간단합니다. 에이전트 설계서에서 “인증”, “권한”, “결제”를 한 칸에 적지 마시고, 아예 서로 다른 정책 파일로 분리해 보시면 됩니다. 그 순간부터 에이전트는 더 똑똑해지는 것이 아니라, 더 운영 가능해집니다.

## 참고한 공식 자료

- [Cloudflare: The age of agents: cryptographically recognizing agent traffic](https://blog.cloudflare.com/signed-agents/)
- [IETF draft: HTTP Message Signatures for Bots](https://datatracker.ietf.org/doc/html/draft-meunier-web-bot-auth-architecture)
- [MCP Authorization specification](https://modelcontextprotocol.io/specification/draft/basic/authorization)
- [x402 Docs: Introduction](https://docs.x402.org/introduction)
- [Coinbase: Introducing x402](https://www.coinbase.com/developer-platform/discover/launches/x402)
