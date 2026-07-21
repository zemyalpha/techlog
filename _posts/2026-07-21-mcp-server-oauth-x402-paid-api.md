---
title: "유료 MCP 서버 설계: OAuth로 열고 x402로 받는 법"
date: "2026-07-21"
keywords: ["MCP", "x402", "OAuth 2.1", "유료 API", "AI 에이전트"]
lang: "ko"
description: "MCP의 OAuth 권한과 x402 결제를 분리해 유료 API를 MCP 서버 뒤에 붙이는 방법, 401·403·402 처리와 간단한 코드 예시를 정리합니다."
---

# 유료 MCP 서버 설계: OAuth로 열고 x402로 받는 법

AI 에이전트가 실제 업무에 들어오면, 가장 먼저 부딪히는 문제는 모델 성능이 아닙니다. **누가 접근했는지**, **무엇을 할 수 있는지**, **무엇에 돈을 낼지**가 한 덩어리로 섞여 버리는 점입니다. 그래서 MCP 서버를 유료화할 때는 인증, 권한, 결제를 한 계층으로 처리하면 안 됩니다.

최근 흐름도 이 방향을 밀고 있습니다. MCP의 권한 부여 문서는 HTTP 기반 전송에서 OAuth 2.1 계열 흐름을 사용해 제한된 MCP 서버에 접근하도록 설명합니다. x402 쪽은 Coinbase가 만든 HTTP 기반 결제 프로토콜로, 공식 문서와 Linux Foundation 발표가 모두 “인터넷 네이티브 결제”와 MCP 서버 연동을 전면에 내세웁니다. 즉, 이제는 **MCP가 도구 접근을 맡고, x402가 결제를 맡는 구조**가 훨씬 자연스럽습니다.

## 1. 먼저 역할을 나누십시오

유료 MCP 서버에서 가장 흔한 실수는 “로그인만 되면 결제도 된 것”처럼 다루는 것입니다. 하지만 실제로는 세 층이 다릅니다.

- **인증(Authentication)**: 이 요청이 누구의 것인지 확인합니다.
- **권한(Authorization)**: 이 사용자가 어떤 도구와 범위를 쓸 수 있는지 정합니다.
- **결제(Payment)**: 이 유료 자원에 접근할 비용이 지불됐는지 확인합니다.

MCP 문서가 강조하는 핵심도 최소 권한입니다. 클라이언트는 필요한 범위만 요청해야 하고, 서버는 제한된 자원만 열어야 합니다. 여기에 결제까지 붙으면, 결제 실패를 권한 실패와 같은 오류로 섞지 않는 것이 중요합니다.

## 2. 상태 코드를 분리하면 운영이 쉬워집니다

실전에서는 다음처럼 해석하면 깔끔합니다.

- **401**: 인증이 아직 안 됨
- **403**: 인증은 됐지만 scope가 부족함
- **402**: 접근은 가능하지만 결제가 필요함

이렇게 분리하면 로그도 훨씬 읽기 쉬워집니다. 예를 들어 401이 많다면 로그인 플로우를 고쳐야 하고, 403이 많다면 scope 설계가 과하고, 402가 많다면 가격이나 과금 단위가 지나치게 세분화됐을 가능성이 큽니다.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

def authenticated(req: Request) -> bool:
    return req.headers.get("authorization") is not None

def allowed(req: Request, scope: str) -> bool:
    return scope in req.headers.get("x-scopes", "")

def paid(req: Request, product: str) -> bool:
    return req.headers.get("x-paid-for") == product

@app.get("/mcp/search")
async def paid_search(req: Request, q: str):
    if not authenticated(req):
        return JSONResponse(status_code=401, content={"error": "authentication_required"})

    if not allowed(req, "search:paid"):
        return JSONResponse(status_code=403, content={"error": "insufficient_scope"})

    if not paid(req, "search:paid"):
        return JSONResponse(
            status_code=402,
            content={
                "error": "payment_required",
                "product": "search:paid",
                "price": "per-request",
            },
        )

    return {"query": q, "results": ["item-1", "item-2"]}
```

이 코드는 x402의 실제 헤더 이름을 복제한 예제가 아니라, **역할을 분리하는 구조**를 보여 주는 예시입니다. 핵심은 결제 로직을 MCP 툴 내부에 묻어 두지 말고, 게이트웨이에서 분리하는 것입니다.

## 3. MCP는 도구 접근, x402는 과금 게이트로 두십시오

x402 문서는 MCP 서버를 통해 유료 API 요청을 만드는 가이드를 따로 제공합니다. 이 말은 곧, 유료 API를 MCP 서버 뒤에 두는 패턴이 이미 문서화되어 있다는 뜻입니다. Cloudflare와 Linux Foundation 자료도 x402를 HTTP 위의 공개 결제 표준으로 설명하며, AI 에이전트와 애플리케이션이 데이터 교환처럼 결제도 주고받을 수 있게 하려는 방향을 보여 줍니다.

그래서 설계는 단순해야 합니다.

1. MCP 서버는 도구 정의와 scope를 관리합니다.
2. 결제 게이트웨이는 402 응답과 재시도 흐름을 담당합니다.
3. 실제 과금 단위는 모델 호출이 아니라 **도구 호출 단위**로 잡습니다.

특히 에이전트는 같은 작업도 여러 번 호출할 수 있으니, 월 구독보다 per-request나 credit 방식이 더 잘 맞는 경우가 많습니다. 중요한 것은 “에이전트에게 돈을 내라고 말하는 것”이 아니라, **결제 실패를 기계적으로 복구 가능한 상태로 만드는 것**입니다.

```python
resp = client.get(url, headers=auth_headers)

if resp.status_code == 402:
    # x402 흐름에서는 여기서 결제 챌린지를 해석하고
    # 지불 증빙을 붙여 다시 요청합니다.
    payment_headers = build_payment_headers(resp)
    resp = client.get(url, headers={**auth_headers, **payment_headers})
```

## 4. 실제 운영에서 꼭 지켜야 할 팁

- **결제 실패와 권한 실패를 같은 에러로 처리하지 마십시오.**
- **scope는 기능 단위로, 결제는 상품 단위로 설계하십시오.**
- **MCP 프롬프트 안에 가격 규칙을 넣지 마십시오.** 정책은 서버에서 관리해야 합니다.
- **감사 로그에 인증, 권한, 결제 이벤트를 따로 남기십시오.**
- **유료 도구는 먼저 작은 범위로 시작하십시오.** 검색, 요약, 리포트처럼 재시도 비용이 낮은 것부터 좋습니다.

## 결론

유료 MCP 서버를 만들 때 기억할 것은 하나입니다. **MCP는 무엇을 할 수 있는지, OAuth는 누가 할 수 있는지, x402는 무엇에 돈을 낼지**를 각각 담당해야 합니다.

- MCP는 도구와 컨텍스트를 연결합니다.
- OAuth는 제한된 접근을 관리합니다.
- x402는 HTTP 위에서 결제를 붙입니다.
- 401, 403, 402를 분리하면 운영이 훨씬 쉬워집니다.

지금 바로 해볼 첫 단계는 간단합니다. 기존 MCP 서버에서 가장 자주 쓰이는 도구 하나를 골라, 그 앞단에 **“401/403/402를 분리하는 게이트웨이”**를 먼저 붙여 보십시오. 그 한 단계만으로도 유료화의 구조가 훨씬 선명해집니다.

## 참고 자료

- [MCP Authorization specification](https://modelcontextprotocol.io/specification/draft/basic/authorization)
- [x402: MCP Server with x402](https://docs.x402.org/guides/mcp-server-with-x402)
- [Coinbase Developer Documentation: x402 Overview](https://docs.cdp.coinbase.com/x402/welcome)
- [Linux Foundation: x402 Foundation operational launch](https://www.linuxfoundation.org/press/linux-foundation-announces-operational-launch-of-x402-foundation-to-standardize-internet-native-payments-for-ai-agents-and-applications)
- [Descope: The Developer’s Guide to Agentic Commerce](https://www.descope.com/blog/post/developer-guide-agentic-commerce)
