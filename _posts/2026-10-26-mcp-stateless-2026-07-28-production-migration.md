---
title: "MCP에서 세션이 사라졌다 — 2026-07-28 스테이트리스 개정이 바꾸는 프로덕션 아키텍처와 마이그레이션 체크리스트"
date: "2026-10-26"
keywords: ["MCP", "Model Context Protocol", "스테이트리스", "에이전트 인프라", "Envoy", "OAuth 2.1", "MRTR"]
lang: "ko"
description: "MCP 2026-07-28 개정은 initialize 핸드셰이크와 세션을 삭제하고 모든 요청이 자기 자신을 설명하는 스테이트리스 프로토콜로 재설계했다. 무엇이 바뀌었고 프로덕션 마이그레이션은 어떻게 할지 정리한다."
---

# MCP에서 세션이 사라졌다 — 2026-07-28 스테이트리스 개정이 바꾸는 프로덕션 아키텍처와 마이그레이션 체크리스트

AI 에이전트가 외부 도구를 호출하는 사실상의 표준이 된 Model Context Protocol(MCP)이 2026년 7월 28일, 탄생 이후 가장 큰 구조적 재설계를 거쳤다. 새 기능이 추가된 수준이 아니라 **프로토콜의 기억 장치 자체가 삭제**된 것이다. `initialize` 핸드셰이크가 없어지고, `Mcp-Session-Id` 헤더가 없어지고, 서버가 클라이언트를 향해 먼저 요청을 보내던 채널도 없어졌다.

이 글은 공식 스펙 체인지로그와 Envoy 1.40 릴리스 노트를 기준으로, 무엇이 정확히 바뀌었는지, 그리고 MCP 서버를 프로덕션에서 운영하는 팀이 무엇을 지금 해야 하는지 정리한다. "MCP는 JSON-RPC에 init 댄스와 세션 헤더가 붙은 것"이라는 mental model을 가지고 있다면, 그 모델은 이제 죽었다.

## 1. 무엇이 삭제되었나

MCP는 2024-11-05 첫 버전 이후 2025-03-26, 2025-06-18, 2025-11-25를 거쳐 왔다. 이 다섯 개 revisions 모두 같은 핵심 설계를 유지했다. 클라이언트가 `initialize`를 보내고, 서버가 자신의 capabilities와 함께 `Mcp-Session-Id`를 발급하고, 이후 모든 요청은 그 세션에 묶이는 구조였다.

[공식 체인지로그](https://modelcontextprotocol.io/specification/2026-07-28/changelog)에 따르면 2026-07-28 개정에서 삭제된 것은 다음과 같다.

- **`initialize` / `notifications/initialized` 핸드셰이크** — 버전과 capabilities 협상이 세션 시작 시점에 일어나던 메커니즘 자체가 사라짐
- **`Mcp-Session-Id` 헤더와 프로토콜 수준의 세션** — Streamable HTTP에서 세션 개념 제거
- **HTTP GET 엔드포인트와 `resources/subscribe` / `resources/unsubscribe`** — 서버가 단독으로 열던 채널 제거. 변경 알림은 새로운 `subscriptions/listen`(단일 장기 POST-응답 스트림, 클라이언트가 유형별로 opt-in)으로 대체
- **`ping`, `logging/setLevel`, `notifications/roots/list_changed`** — 로그 레벨은 요청별 `_meta`의 `io.modelcontextprotocol/logLevel`로 이동
- **SSE 스트림 재개(`Last-Event-ID`)** — 끊긴 스트림은 재개되지 않고, 클라이언트가 새 요청 ID로 재발행해야 함

왜 이렇게까지 했을까. 이전 설계는 본질적으로 데스크톱 앱 프로토콜이었다. Claude Desktop이나 VS Code가 로컬 stdio 서버를 띄우고 긴 세션을 유지하는 구조에서는 문제가 없었다. 그러나 MCP 서버를 여러 레플리카로 확장하는 순간, 세션은 곧바로 운영 부채가 된다. 로드밸런서에 스티키 세션이 강제되고, 레플리카 하나가 재배포되면 진행 중이던 모든 세션이 404를 뱉는다.

## 2. 새 모델: 모든 요청이 자기 자신을 설명한다

2026-07-28의 핵심은 한 문장으로 요약된다. **모든 요청이 자신을 처리하는 데 필요한 모든 것을 스스로 실고 다닌다.**

이전에 세션 시작 때 한 번 정하던 정보들이 이제 모든 요청의 `_meta` 블록에 담긴다.

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "check_pending_invoices",
    "arguments": { "date": "2026-10-26" },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": { "elicitation": {} },
      "io.modelcontextprotocol/clientInfo": { "name": "my-client", "version": "1.0.0" }
    }
  }
}
```

`protocolVersion`과 `clientCapabilities`는 필수다. 둘 중 하나라도 빠지면 서버는 `-32602` 에러와 함께 HTTP 400으로 거절해야 한다. 서버 측에서 지원하지 않는 버전이 오면 `UnsupportedProtocolVersionError`(에러 코드 `-32022`)와 함께 자신이 지원하는 버전 목록을 돌려주고, 클라이언트는 호환되는 버전으로 재시도하면 된다. 핸드셰이크 없는 버전 협상이다.

사전에 상대 capabilities를 알고 싶다면 새로 추가된 `server/discover`를 호출하면 된다. 서버는 이 RPC의 구현이 의무이며, 지원 버전·capabilities·신원을 한 번에 cacheable한 응답으로 돌려준다. 버전 협상뿐 아니라 `tools/list`, `resources/list` 같은 목록 응답에도 `ttlMs`와 `cacheScope`(`public`/`private`) 캐시 힌트가 붙는다. HTTP 캐시 시맨틱이 프로토콜 안으로 들어온 것이다.

라우팅 관점에서 중요한 변화도 있다. 메서드 이름과 도구 이름이 HTTP 헤더로도 노출된다.

```text
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: get_weather
```

게이트웨이는 본문을 파싱하지 않고도 이 헤더만으로 라우팅과 인가를 할 수 있다. 단, 본문이 여전히 source of truth이며, 서버는 헤더와 본문이 불일치하면 요청을 거절해야 한다. 게이트웨이가 A 작업으로 인가했는데 서버가 B 작업을 실행하는 변조를 막는 장치다.

## 3. 서버가 먼저 묻던 시대의 끝 — MRTR 패턴

구조 변경 중 실제 코드에 가장 많은 영향을 주는 것은 서버 주도 요청(server-initiated requests)의 제거다. 이전까지 서버는 진행 중인 작업에 추가 정보가 필요하면 열려 있는 스트림을 타고 거꾸로 요청을 보냈다. 사용자에게 되묻는 elicitation, 호스트의 LLM을 빌려 쓰는 sampling, 워크스페이스 폴더를 묻는 roots가 그것이다. 이 설계는 양방향 열린 채널을 요구했고, 결국 같은 서버 인스턴스가 작업 전체를 처리해야 한다는 뜻이었다.

대체재는 **Multi Round-Trip Requests(MRTR)** 다. 서버는 묻지 않고, **요청을 "질문으로 끝냄"으로써** 필요한 것을 받는다.

1. 클라이언트가 `tools/call`을 보낸다
2. 서버는 최종 결과 대신 `resultType: "input_required"` 결과를 반환한다. `inputRequests` 필드에 필요한 정보의 목록이, 선택적으로 `requestState`(불투명한 blob)에 재개에 필요한 맥락이 담긴다
3. 클라이언트는 답을 모아 **원래 요청을 새 요청 ID로 재시도**한다. `inputResponses`와 `requestState`를 그대로 실어서
4. 어느 레플리카든 `requestState`를 검증하고 작업을 마친다

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "io.modelcontextprotocol/elicitation": {
      "message": "어느 날짜의 미지급 송장을 조회할까요?"
    }
  },
  "requestState": "gAAAAABl..."
}
```

파괴적 작업의 확인(concern confirmation)에도 같은 패턴이 쓰인다. `delete_resources(resource_ids=[...])` 호출에 대해 서버가 "53개 리소스 삭제 확인"을 `input_required`로 돌려주고, 클라이언트가 확인 후 재시도하는 식이다.

MRTR은 `tools/call`, `resources/read`, `prompts/get` 세 연산에만 허용된다. 재시도는 새로운 JSON-RPC ID를 써야 하며, 서버는 멱등성, 재시도 시 재인가, 라운드 횟수 제한, `requestState` 만료 설계를 직접 책임진다. `requestState`는 변조 방지가 필요하다면 서버 측 상태를 가리키는 불투명 핸들이나 MAC으로 보호된 값으로 만들라 것이 스펙의 권고다.

## 4. "스테이트리스"의 진짜 의미 — 숨은 상태가 명시적 상태로

혼동하기 쉬운 지점을 짚자. 2026-07-28은 **애플리케이션 상태를 금지하는 게 아니라, 전송 계층에 숨어 있던 상태를 명시적으로 만드는 것**이다.

서버가 호출 사이에 실제 상태가 필요하다면(작업 워크스페이스, 장바구니, 분석 컨텍스트), 이제 스스로 식별자를 발급하고 클라이언트가 그것을 **평범한 도구 인자**로 다시 전달하게 하면 된다.

```text
create_workspace()
→ { "workspace_id": "ws_7f3..." }

add_file(workspace_id="ws_7f3...", path="report.md", content=...)
run_analysis(workspace_id="ws_7f3...")
```

운영 관점에서 이것은 명백한 이득이다. 세션 어피니티 어노테이션도, 세션 소유용 StatefulSet도 필요 없어진다. 쿠버네티스의 평범한 `Deployment` + `Service`로, 라운드 로빈 로드밸런싱 아래에서 수평 확장된다. 단, 핸들 설계의 책임은 서버 개발자에게 넘어온다. 핸들마다 소유권 검증, 테넌트 격리, 만료, 리플레이 방지가 필요하다. 세션이 감춰주던 일이 드러났을 뿐이다.

## 5. 12개월의 시계 — Deprecation 타임라인

이번 개정과 함께 MCP는 공식 **기능 라이프사이클 정책**(Active / Deprecated / Removed, 폐기 전 최소 12개월 유예)과 공개 deprecation 레지스트리를 도입했다. 이제 마이그레이션은 소문이 아니라 날짜로 계획할 수 있다.

2026-07-28 기준 deprecated, 최소 제거 시점 2027-07-28:

- **Sampling** — 서버가 호스트의 LLM을 빌려 쓰던 기능. 서버가 LLM 호출이 필요하면 이제 LLM provider API를 직접 호출해야 함
- **Roots** — 경로는 도구 인자나 설정으로 전달
- **Logging 유틸리티** — stderr나 OpenTelemetry로 대체
- **Dynamic Client Registration(RFC 7591)** — Client ID Metadata Documents로 대체

인증 이야기가 나왔으니 짚자. MCP 인가는 OAuth 2.1 관습을 따르며, 미인증 요청에 `WWW-Authenticate` 헤더와 함께 RFC 9728 Protected Resource Metadata 문서(`/.well-known/oauth-protected-resource`)를 가리키는 표준 웹 플로우로 정리됐다. 인가 서버는 RFC 9207의 `iss` 값을 포함해야 하고 클라이언트는 코드 교환 전 발급자를 검증해야 한다. 자격 증명은 발급자 식별자별로 저장되며 인가 서버 간 재사용이 금지된다.

한 가지 더. MCP는 이미 Anthropic 단독 프로젝트가 아니다. Anthropic은 MCP를 리눅스 재단 산하 **Agentic AI Foundation**(Anthropic·Block·OpenAI 공동 설립, Google·Microsoft·AWS 등 지원)에 기증했고([Anthropic 공식 발표](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)), 이번 개정이 인프라 표준다운 문체인 것도 그 중립적 거버넌스의 영향으로 읽힌다.

## 6. Envoy 1.40: 에이전트 트래픽도 이제 프록시의 시민

스테이트리스 개정의 수혜자는 프록시 생태계이기도 하다. [Envoy 1.40](https://github.com/envoyproxy/envoy/releases/tag/v1.40.0)(2026년 10월 13일 릴리스)은 MCP를 네이티브로 지원한다. [공식 문서](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/mcp_filter)에 따르면 두 가지 확장이 제공된다.

- **MCP 필터**(`envoy.filters.http.mcp`) — MCP 서버 앞단에 놓여 요청을 검증하고, 메서드·도구 이름을 추출해 표준 Envoy 라우팅·RBAC·ext_authz·액세스 로그에 노출
- **MCP JSON/REST 브리지**(`envoy.filters.http.mcp_json_rest_bridge`) — 기존 REST API를 설정만으로 MCP 서버처럼 노출

핵심 활용은 `Mcp-Method`와 `Mcp-Name` 헤더 기반 라우팅이다. 도구별로 별도 클러스터로 라우팅하거나, 비싼 도구에 별도 rate limit을 걸거나, RBAC으로 "누가 어떤 도구를 호출할 수 있는지"를 게이트웨이에서 통제할 수 있다. 1.40.0 릴리스 노트에는 MCP 필터에 per-route 설정, 프로토콜 버전 강제, 필수 클라이언트 capabilities 지정이 추가됐고, `_meta` 필드 이름의 점(`io.modelcontextprotocol/protocolVersion`)이 올바르게 파싱되도록 수정되었다는 항목도 있다. 새 스펙 지원이 릴리스 시점에 완성되어 가는 중이라는 뜻이다.

구버전 클라이언트가 걱정된다면, Envoy 한 리스너에서 새·구 프로토콜을 동시에 서빙하는 것도 가능하다. `max_supported_protocol_version` 설정으로 수용 버전을 정하면 된다.

## 7. 지금 할 일 — 마이그레이션 체크리스트

### 로컬 stdio 서버만 운영한다면

급할 것이 없다. 이전 revision도 여전히 유효하고, 클라이언트·SDK가 협상하는 버전으로 동작한다. 다른 이유로 서버를 건드릴 때 SDK를 함께 업데이트하면 된다. 현재 TypeScript·Python·Go·C# 공식 SDK가 새 스펙을 지원한다(Rust는 베타).

### 원격 HTTP로 MCP 서버를 운영한다면

1. **세션 의존 제거** — `Mcp-Session-Id`, 서버 메모리의 연결별 상태에 의존하는 코드를 제거하고, 크로스콜 상태를 도메인 핸들(`workspace_id` 등)의 도구 인자로 전환
2. **새 요청 형식 구현** — 모든 요청의 `_meta`에 `protocolVersion`·`clientCapabilities` 포함, `server/discover` 구현, 버전 불일치 시 `-32022` 응답
3. **MRTR 전환** — elicitation/sampling에 의존하던 흐름을 `input_required` → 재시도 패턴으로 재설계. 재시도 멱등성과 재인가 포함
4. **인증 정리** — RFC 9728 PRM 노출, RFC 9207 `iss` 검증, 발급자별 자격 증명 저장. Dynamic Client Registration에 붙어 있다면 Client ID Metadata Documents로
5. **deprecation 인벤터리** — Sampling/Roots/Logging 사용처를 지금 조사. 제거 창은 2027-07-28부터 열린다
6. **관측 지표 교체** — HTTP 상태 코드만 보지 말 것. MCP는 실패한 도구 호출도 `200 OK`에 `isError: true`로 돌려줄 수 있다. 골든 시그널은 `tools/call` 처리량, `isError` 비율, 도구별 p99 지연, `UnsupportedProtocolVersionError` 발생 수

### 한 가지 주의: era-mixing

헤더 없이 날아드는 트래픽은 일부 SDK에서 구버전(2025-11-25) 세션 전송으로 조용히 라우팅될 수 있다. "스테이트리스로 전환했다"고 믿는 플릿의 로그에 "Missing session ID" 에러가 보인다면, 누군가(손으로 짠 클라이언트, 낡은 SDK) 아직 구버전을 말하고 있다는 신호다. 이 문자열을 로그에서 grep하는 것이 era 마이그레이션의 화재 감지기다.

## 결론

- MCP 2026-07-28 개정은 `initialize` 핸드셰이크와 세션을 삭제하고, 모든 요청이 버전·capabilities·신원을 `_meta`로 스스로 실고 다니는 스테이트리스 프로토콜이 되었다
- 서버 주도 요청은 MRTR 패턴(`input_required` + `requestState` + 재시도)으로 대체됐고, 애플리케이션 상태는 도구 인자로 전달되는 명시적 핸들이 된다
- Sampling·Roots·Logging·DCR은 12개월 유예로 deprecated되었고, 최소 제거 시점은 2027-07-28이다
- 운영상 이득은 구체적이다. 스티키 세션 없는 수평 확장, 헤더 기반 게이트웨이 라우팅(Envoy 1.40이 네이티브 지원), 표준 OAuth 2.1과 OpenTelemetry
- 첫 단계는 간단하다. 원격 MCP 서버를 운영 중이라면 오늘 `server/discover`를 구현하는 것부터. 로컬 stdio라면 다음 SDK 업데이트 때 자연스럽게 따라가면 된다

**참고 자료**
- [MCP Specification 2026-07-28 Changelog (공식)](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [Envoy v1.40.0 Release Notes (공식)](https://github.com/envoyproxy/envoy/releases/tag/v1.40.0)
- [Envoy MCP Filter 문서 (공식)](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/mcp_filter)
- [Anthropic — Donating MCP to the Agentic AI Foundation (공식)](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)
