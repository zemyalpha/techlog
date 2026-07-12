---
title: "MCP 2026-07-28 스펙이 바꾸는 것: 세션리스 프로토콜과 OAuth 2.1 리소스 서버"
date: "2026-07-12"
keywords: ["MCP", "Model Context Protocol", "OAuth 2.1", "stateless protocol", "AI agent authentication"]
lang: "ko"
description: "2026년 7월 28일 최종 확정되는 MCP 스펙의 핵심 변화를 분석한다. 세션리스 프로토콜 전환, OAuth 2.1 리소스 서버 모델, 확장 프레임워크 도입이 프로덕션 환경에 미치는 영향과 마이그레이션 체크리스트를 정리했다."
---

# MCP 2026-07-28 스펙이 바꾸는 것: 세션리스 프로토콜과 OAuth 2.1 리소스 서버

2026년 5월 21일, Model Context Protocol의 수석 유지자 David Soria Parra와 Den Delimarsky는 "런치 이후 가장 큰 개정"이라고 부른 릴리스 후보(Release Candidate)를 공개했다. 최종 스펙은 2026년 7월 28일에 확정된다. 글을 쓰는 시점(7월 12일) 기준으로 16일 남았다.

이번 개정이 중요한 이유는 단순히 기능이 추가된 게 아니라 프로토콜의 기반을 뒤집기 때문이다. 세션 모델이 사라지고, 인증이 OAuth 2.1 표준으로 정렬되며, 확장 메커니즘이 정식으로 편입된다. MCP 서버를 프로덕션에서 운영 중인 팀이라면 이 변화가 인프라 구성과 보안 모델에 직접 영향을 미친다.

이 글에서는 공식 스펙 문서와 릴리스 후보 블로그 포스트를 기반으로, 실제 운영에 영향을 주는 핵심 변화를 정리하고 마이그레이션 관점에서 무엇을 해야 하는지를 다룬다.

## 세션이 사라진다: 스테이트리스 프로토콜 코어

가장 큰 변화는 프로토콜 레이어에서 세션을 완전히 제거한 것이다. 6개의 Specification Enhancement Proposal(SEP)이 함께 동작하여 기존 세션 모델을 해체한다.

### 무엇이 바뀌는가

기존(2025-11-25 스펙)에서 MCP 클라이언트가 원격 서버에 도구를 호출하려면, 먼저 `initialize` / `initialized` 핸드셰이크를 수행해야 했다. 이때 서버는 `Mcp-Session-Id` 헤더를 반환하고, 이후 모든 요청은 해당 세션 ID를携带해야 했다. 즉, 최초 핸드셰이크를 처리한 서버 인스턴스에 클라이언트가 고정(pinned)되는 구조였다.

새 스펙(2026-07-28)에서는 이 핸드셰이크가 완전히 제거된다(SEP-2575). `Mcp-Session-Id` 헤더도 사라진다(SEP-2567). 대신 프로토콜 버전, 클라이언트 정보, 클라이언트 capabilities가 매 요청의 `_meta` 필드에 포함되어 전달된다. 서버 capabilities는 필요할 때 새로운 `server/discover` 메서드로 가져올 수 있다.

실제 요청 구조를 비교하면 차이가 명확하다.

**이전 (2025-11-25):**

```http
POST /mcp HTTP/1.1
Mcp-Session-Id: 1868a90c-3a3f-4f5b
Content-Type: application/json

{"jsonrpc":"2.0","id":2,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"}}}
```

**이후 (2026-07-28):**

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"},
           "_meta":{"io.modelcontextprotocol/clientInfo":
                    {"name":"my-app","version":"1.0"}}}}
```

### 인프라에 미치는 영향

세션이 사라진다는 것은 운영 관점에서 즉각적인 이점이 있다. 기존에는 원격 MCP 서버를 수평 확장하려면 sticky session(세션 친화성 라우팅), 공유 세션 저장소, 그리고 게이트웨이에서 JSON-RPC 바디를 까봐야 하는 deep packet inspection이 필요했다. 새 스펙에서는 이런 것들이 프로토콜 레이어에서 더 이상 필요하지 않다.

새로 도입된 `Mcp-Method`와 `Mcp-Name` 헤더(SEP-2243)는 로드밸런서와 API 게이트웨이가 JSON-RPC 바디를 파싱하지 않고도 요청을 라우팅할 수 있게 한다. 서버는 헤더와 바디가 일치하지 않는 요청을 거부하도록 명시되어 있어, 라우팅/보안 불일치 문제도 차단된다. 또한 `tools/list` 같은 응답에 `ttlMs`와 `cacheScope` 필드가 추가되어, HTTP의 `Cache-Control`과 유사한 방식으로 클라이언트 캐싱이 표준화된다.

분산 추적도 개선된다. W3C Trace Context가 `_meta`의 고정 키 이름으로 전파되어, OpenTelemetry 호환 분산 추적이 SDK와 게이트웨이 전반에서 기본 동작한다.

### 스테이트리스 프로토콜, 스테이트풀 애플리케이션

프로토콜 레이어에서 세션이 사라졌다고 해서 애플리케이션이 무상태여야 하는 것은 아니다. 상태가 필요한 서버는 HTTP API가 오랫동안 해왔던 것처럼 명시적 핸들(explicit handle) 패턴을 사용한다. 도구가 `basket_id`나 `browser_id` 같은 식별자를 반환하고, 모델이 이후 호출에서 그 식별자를 일반 인자로 전달하는 방식이다.

공식 블로그 포스트는 이 패턴이 단순한 세션 상태 대체 이상이라고 설명한다. 모델이 핸들을 직접 다루기 때문에, 핸들을 여러 도구에 걸쳐 조합하거나, 추론하거나, 단계 간에 전달하는 것이 가능해진다. 이는 외부 세션 상태에 숨겨져 있을 때보다 모델의 다단계 추론 신뢰성을 높이는 효과가 있다.

## 인증이 OAuth 2.1 표준으로 정렬된다

이전 MCP 스펙의 인증 접근은 관대하게 말해 "토큰을 알아서 가져오라" 수준이었다. 2026-07-28 스펙은 MCP 인증을 OAuth 2.1과 OpenID Connect 배포 패턴에 맞춰 대폭 강화한다.

### MCP 서버 = OAuth 2.1 리소스 서버

MCP 서버는 공식적으로 OAuth 2.1 리소스 서버로 분류된다. 이는 2025-11-25 스펙에서 도입되었으나, 2026-07-28에서 대폭 강화되었다. 핵심 요구사항은 다음과 같다:

**1. Protected Resource Metadata (RFC 9728) 구현 의무화**

MCP 서버는 RFC 9728에 정의된 OAuth 2.0 Protected Resource Metadata를 반드시 구현해야 한다. RFC 9728은 2025년 4월 IETF Standards Track으로 발행된 문서로(M. Jones, P. Hunt, A. Parecki 저), 보호된 리소스가 자신에 대한 메타데이터를 표준 위치(`/.well-known/oauth-protected-resource`)에 게시하여 클라이언트가 올바른 인가 서버를 자동 발견할 수 있게 하는 메커니즘이다. MCP 클라이언트는 이 메타데이터를 통해 어느 인가 서버에서 토큰을 받아야 하는지 알 수 있다.

**2. Resource Indicators (RFC 8707) 의무화**

MCP 클라이언트는 RFC 8707에 정의된 Resource Indicators를 반드시 구현해야 한다. RFC 8707은 2020년 2월 IETF Standards Track으로 발행되었으며(B. Campbell, J. Bradley, H. Tschofenig 저), 클라이언트가 액세스 토큰을 요청할 때 그 토큰을 사용할 대상 리소스(protected resource)를 명시적으로 지정하는 `resource` 파라미터를 정의한다. 이를 통해 악의적인 MCP 서버가 다른 서버용으로 발급된 토큰을 가로채서 사용하는 token mis-redemption 공격을 방지한다. MCP가 다수의 서버와 동시에 통신하는 에이전트 패턴을 장려하는 점을 고려하면, 특히 중요한 보안 조치다.

**3. 인가 서버 발급자 검증 (RFC 9207)**

클라이언트는 인가 응답의 `iss` 파라미터를 RFC 9207에 따라 검증해야 한다. 이는 mix-up 공격을 완화하는 조치로, 클라이언트가 여러 인가 서버와 통신할 때 응답이 어느 서버에서 왔는지 확인하여 중간자 공격을 방지한다. 등록된 크리덴셜은 발급한 인가 서버의 issuer에 바인딩되며, 리소스가 다른 인가 서버로 마이그레이션되면 클라이언트는 재등록해야 한다(SEP-2352).

**4. Dynamic Client Registration에서 Client ID Metadata Documents로 전환**

기존에 사용되던 Dynamic Client Registration(RFC 7591)은 더 이상 권장되지 않는다(deprecated). 새 스펙은 OAuth Client ID Metadata Documents(CIMD)를 선호하는 클라이언트 등록 방식으로 권장한다. RFC 7591은 하위 호환성을 위해 유지되지만, CIMD를 지원하지 않는 인가 서버에서만 사용해야 한다.

**5. 리프레시 토큰 및 스코프 처리 정식화**

이전 스펙에서는 리프레시 토큰 동작이 정의되지 않아 구현마다 달랐다. 새 스펙은 OpenID Connect 스타일 인가 서버에서 리프레시 토큰을 요청하는 방법을 문서화하고(SEP-2207), step-up authorization 중 스코프 누적 동작을 명확히 했다(SEP-2350). 클라이언트는 등록 시 OpenID Connect `application_type`을 선언해야 하며(SEP-837), 이는 인가 서버가 데스크톱/CLI 클라이언트를 "web"으로 기본 설정한 후 localhost 리다이렉트 URI를 거부하는 흔한 실패 모드를 해결한다.

### 실제 401 응답 예시

새 스펙에서 MCP 서버는 스코프 가이드를 포함한 401 응답을 반환할 수 있다:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                          scope="files:read"
```

이 응답을 받은 클라이언트는 `resource_metadata` URL에서 Protected Resource Metadata를 가져오고, 그 안에 명시된 인가 서버로 이동하여 `files:read` 스코프를 요청한다. 이때 `resource` 파라미터(RFC 8707)로 이 MCP 서버의 URI를 명시하여, 발급되는 토큰이 이 서버에서만 사용 가능하도록 제한한다.

## 확장 프레임워크가 정식으로 편입된다

2026-07-28 스펙에서 확장(extensions)은 비공식적 관례에서 정식 거버넌스 시스템으로 승격된다. 핵심 요소는 다음과 같다:

- **역순 DNS 식별자** — 확장은 `io.modelcontextprotocol.ext.tasks` 같은 역순 DNS 식별자를 사용한다
- **capability 협상** — `extensions` 맵을 통해 클라이언트와 서버가 지원 확장을 협상한다
- **독립 버전 관리** — 확장은 코어 스펙과 별도로 버전이 관리된다
- **전담 유지자** — 각 확장은 `ext-*` 저장소에서 위임된 유지자가 관리한다

두 가지 확장이 특히 주목된다:

**MCP Apps** — 서버가 렌더링하는 대화형 HTML 인터페이스로, 샌드박스된 iframe에서 실행된다. 도구는 UI 템플릿을 미리 선언하며, 클라이언트는 렌더링 전에 프리패치하고 보안 검토를 수행할 수 있다. UI 액션은 일반 도구 호출과 동일한 JSON-RPC 채널을 통해 전달된다.

**Tasks** — 실험적 기능에서 정식 확장으로 승격되었다. 새 라이프사이클은 스테이트리스 설계로 동작한다: `tools/call`이 태스크 핸들을 반환하고, 클라이언트가 `tasks/get`, `tasks/update`, `tasks/cancel`로 진행 상황을 관리한다. 세션이 없으므로 `tasks/list`는 제거되었다 — 모든 태스크를 나열하는 것은 안전하지 않은 연산으로 간주되기 때문이다.

## 3가지 기능이 더 이상 사용되지 않는다 (Deprecated)

다음 세 기능이 공식적으로 deprecation되며, 최소 12개월의 유예 기간 후 제거된다:

| 기능 | 대체 방안 |
|------|-----------|
| Roots | 도구 파라미터, 리소스 URI, 또는 서버 설정 |
| Sampling | LLM 제공자 API와의 직접 통합 |
| Logging | stdio 트랜스포트의 stderr, 또는 OpenTelemetry 구조화 관측성 |

현재는 어노테이션만 추가된 상태로 메서드 자체는 유예 기간 동안 계속 동작한다. 하지만 마이그레이션 계획을 세울 시점이다.

## JSON Schema 2020-12 전체 지원

도구의 `inputSchema`와 `outputSchema`가 JSON Schema 2020-12 어휘를 완전히 지원한다. 입력 스키마에서 `oneOf`, `anyOf`, `allOf` 같은 조합(composition)과 조건부 로직을 사용할 수 있고, 다른 스키마를 참조할 수도 있다. 출력 스키마는 사실상 제한이 없으며, `structuredContent`는 모든 JSON 값을 가질 수 있다.

한 가지 주의할 breaking change: "resource not found" 에러가 커스텀 코드 `-32002`에서 표준 JSON-RPC `-32602`(Invalid Params)로 이동한다. 서버나 클라이언트가 `-32002`를 하드코딩하고 있다면 반드시 업데이트해야 한다.

## 마이그레이션 체크리스트

최종 스펙이 7월 28일에 확정되지만, SDK 유지자와 서버 구현자를 위한 10주 검증 창이 이미 진행 중이다. Tier 1 SDK는 이 기간 내에 지원을 제공할 것으로 예상된다. 프로덕션에서 MCP 서버를 운영 중인 팀이 점검해야 할 항목을 정리한다:

**인프라 (즉시 적용 가능):**
1. `Mcp-Session-Id` 기반 sticky session 설정 제거 — 라운드 로빈 로드밸런서로 전환 가능
2. `Mcp-Method` / `Mcp-Name` 헤더를 라우팅에 활용하도록 API 게이트웨이 규칙 업데이트
3. `tools/list` 응답의 `ttlMs`를 활용한 클라이언트 캐싱 도입

**인증 (마이그레이션 창 내):**
4. `/.well-known/oauth-protected-resource` 엔드포인트 구현 (RFC 9728)
5. 클라이언트에 `resource` 파라미터 추가 (RFC 8707)
6. 인가 응답의 `iss` 검증 로직 추가 (RFC 9207)
7. Dynamic Client Registration에서 Client ID Metadata Documents로 전환 검토

**코드 (breaking change 대응):**
8. `-32002` 에러 코드 하드코딩 제거, `-32602`로 변경
9. `initialize` / `initialized` 핸드셰이크 로직 제거, `_meta` 기반 메타데이터 전송으로 전환
10. `tasks/list` 사용 코드 제거 (실험적 Tasks API 사용 시)
11. Roots, Sampling, Logging 사용 부분의 마이그레이션 계획 수립

## 핵심 요약

- **세션리스 전환은 인프라를 단순화한다.** Sticky session, 공유 세션 저장소, deep packet inspection이 더 이상 프로토콜 레이어에서 필요하지 않다. 라운드 로빈 로드밸런서만으로 수평 확장이 가능하다.

- **OAuth 2.1 정렬은 엔터프라이즈 배포의 문을 연다.** RFC 9728, RFC 8707, RFC 9207을 따르면 Okta, Azure AD, Google Workspace 같은 기존 IdP와의 통합이 스펙 수준에서 정의된다.

- **확장 프레임워크는 진화를 안정화한다.** 코어를 건드리지 않고 새 기능이 확장으로 추가될 수 있으며, Feature Lifecycle Policy(Active → Deprecated → Removed, 각 단계 최소 12개월)가 breaking change를 예측 가능하게 만든다.

- **7월 28일 이전에 검증하라.** 최종 스펙 확정까지 10주 검증 창이 있다. 이 기간에 SDK를 업데이트하고, 인증 흐름을 테스트하고, breaking change(`-32002` → `-32602`, 핸드셰이크 제거)를 대응해야 한다.

공식 스펙 문서와 릴리스 후보 블로그 포스트는 글 말미에 링크했으니, 직접 확인하기 바란다. 스펙 변경의 세부 사항은 원문을 참조하는 것이 가장 정확하다.

## 참조

- [MCP 2026-07-28 Release Candidate 공식 블로그 포스트](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/) — David Soria Parra, Den Delimarsky (2026-05-21)
- [MCP Authorization Specification (Draft)](https://modelcontextprotocol.io/specification/draft/basic/authorization) — 공식 스펙 문서
- [RFC 9728: OAuth 2.0 Protected Resource Metadata](https://www.rfc-editor.org/rfc/rfc9728.html) — IETF, 2025년 4월
- [RFC 8707: Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707.html) — IETF, 2020년 2월
- [RFC 9207: OAuth 2.0 Authorization Server Issuer Identification](https://www.rfc-editor.org/rfc/rfc9207.html) — IETF
- [WorkOS: MCP 2026 Spec Agent Authentication 분석](https://workos.com/blog/mcp-2026-spec-agent-authentication) — 2026년 6월 18일
