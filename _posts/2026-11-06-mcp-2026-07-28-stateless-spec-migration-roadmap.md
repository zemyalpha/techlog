---
title: "MCP가 세션을 버렸다 — 2026-07-28 스테이트리스 스펙 마이그레이션 가이드와 로드맵 5개 우선영역"
date: "2026-11-06"
keywords: ["MCP", "Model Context Protocol", "스테이트리스", "Streamable HTTP", "SEP-2575", "server/discover", "에이전트 프로토콜"]
lang: "ko"
description: "2026-07-28 MCP 스펙이 프로토콜 세션과 initialize 핸드셰이크를 제거하고 스테이트리스로 전환했습니다. 무엇이 바뀌었고 기존 서버를 어떻게 마이그레이션할지, 그리고 2026 로드맵 5개 우선영역까지 정리합니다."
---

# MCP가 세션을 버렸다 — 2026-07-28 스테이트리스 스펙 마이그레이션 가이드와 로드맵 5개 우선영역

MCP(Model Context Protocol)를 프로덕션에서 운영하고 있다면 2026년은 그냥 지나칠 수 없는 해입니다. 2026년 7월 28일자 스펙 개정에서 프로토콜 레벨 세션과 `initialize` 핸드셰이크가 통째로 사라졌고, 원격 MCP 서버는 이제 "그냥 평범한 HTTP 워크로드"가 되었습니다. 공식 로드맵 문서의 표현을 그대로 가져오면 "The 2026-07-28 release made a remote MCP server a normal HTTP workload"입니다.

이 변경은 사소한 문법 수정이 아닙니다. 스티키 세션, 세션 스토어, 핸드셰이크 상태 머신으로 감싸던 기존 구현 전반이 영향을 받습니다. 반면 로드밸런서 뒤에서 수평 확장하던 운영자에게는 오랫동안 골치였던 문제가 원칙 차원에서 해소된 셈입니다. 이 글에서는 (1) 실제로 무엇이 바뀌었는지, (2) 기존 서버·클라이언트를 어떻게 마이그레이션할지, (3) 앞으로 오는 것이 무엇인지(2026 로드맵 5개 우선영역)를 다룹니다. 모든 사실 관계는 공식 스펙 changelog와 로드맵 문서를 기준으로 확인했습니다.

## 1. 무엇이 바뀌었나: 2026-07-28 스펙의 메이저 변경

공식 changelog에 따르면 2026-07-28 개정은 이전 개정(2025-11-25) 이후 다음과 같은 메이저 변경을 담고 있습니다.

**① 프로토콜 세션 제거.** Streamable HTTP 트랜스포트에서 프로토콜 레벨 세션과 `Mcp-Session-Id` 헤더가 제거되었습니다. 이전에는 클라이언트가 서버에 연결하면 세션 ID를 발급받고, 이후 모든 요청에 이 ID를 실어 보내며 서버는 그 세션에 상태를 묶어 두었습니다. 이제 그 개념 자체가 없습니다.

**② 완전한 스테이트리스화.** `initialize` / `notifications/initialized` 핸드셰이크가 사라졌습니다. 대신 모든 요청이 자신의 프로토콜 버전과 클라이언트 역량(capabilities)을 `_meta` 필드에 직접 실어 보냅니다. 핵심 필드는 다음과 같습니다.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_docs",
    "arguments": { "query": "stateless mcp" },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": { "roots": {} },
      "io.modelcontextprotocol/clientInfo": { "name": "my-agent", "version": "1.4.0" }
    }
  }
}
```

버전이 맞지 않으면 `UnsupportedProtocolVersionError`가 반환됩니다(SEP-2575).

**③ server/discover RPC 추가.** 서버는 이 RPC를 반드시(MUST) 구현해서 지원 프로토콜 버전, 역량, 신원 정보를 광고해야 합니다. 클라이언트는 첫 요청 전에 호출해 버전을 사전 협상하거나, STDIO에서 하위호환 프로브로 쓸 수 있습니다.

```json
{ "jsonrpc": "2.0", "id": 0, "method": "server/discover", "params": {} }
```

**④ 구독 모델 교체.** HTTP GET 엔드포인트와 `resources/subscribe`/`resources/unsubscribe`가 `subscriptions/listen`으로 통합됐습니다. 클라이언트가 옵트인한 유형(`toolsListChanged`, `promptsListChanged` 등)의 변경 알림을 하나의 장수명(long-lived) POST-응답 스트림으로 받는 구조입니다.

**⑤ 상태는 핸들로.** 호출 사이 상태가 필요한 서버는 이제 세션이 아니라 명시적인 서버 발급 핸들(handle)을 일반 도구 인자로 주고받습니다(SEP-2567). `tools/list`, `resources/list`, `prompts/list` 같은 목록 엔드포인트는 더 이상 커넥션별로 다른 결과를 반환하지 않습니다.

이와 함께 deprecation도 대거 이뤄졌습니다. Roots, Sampling, Logging이 deprecated되었고(SEP-2577) 최소 12개월간은 동작하지만 신규 구현은 채택하지 말라는 것이 공식 입장입니다. 구버전 HTTP+SSE 트랜스포트도 1년의 유예 기간을 두고 공식 폐기 대상이 됐습니다. 한편 Tasks는 실험적 코어에서 `io.modelcontextprotocol/tasks` 익스텐션으로 이동하며 폴링 기반 `tasks/get`과 새 `tasks/update`를 갖췄습니다(SEP-2663).

SDK 지원도 빠릅니다. 공식 발표에 따르면 4개 Tier 1 SDK(TypeScript, Python, Go, C#)가 릴리스 당일부터 2026-07-28 스펙을 지원하며, Rust SDK는 베타 상태입니다. Tier 1 SDK 전체가 월 5억 다운로드에 근접하고 TypeScript·Python은 누적 10억 다운로드를 넘겼다고 합니다.

## 2. 마이그레이션 가이드: 무엇이 깨지고 어떻게 고치나

기존 구현에서 실제로 깨지는 지점과 대응을 정리하면 다음과 같습니다.

**Step 1 — 세션 저장소 제거.** Redis 등에 `Mcp-Session-Id` → 상태 매핑을 저장하고 있었다면 그 계층을 걷어냅니다. 세션 스티키니스(stickiness)를 위해 로드밸런서에 설정해 둔 세션 어피니티도 더 이상 프로토콜 요구사항이 아닙니다. 진짜 상태(커서, 페이지네이션 토큰, 업로드 컨텍스트 등)는 불투명한 핸들 문자열로 만들어 도구 인자로 돌려보내고, 서버 쪽에서는 그 핸들을 키로 상태를 보관하면 됩니다.

**Step 2 — 핸드셰이크 상태 머신 삭제.** 클라이언트라면 `initialize` → `initialized` 흐름 대신, 모든 요청의 `_meta`에 프로토콜 버전과 클라이언트 정보를 싣도록 바꿉니다. 서버라면 요청마다 `_meta`에서 버전을 읽고 불일치 시 `UnsupportedProtocolVersionError`를 반환하는 방어 코드를 넣습니다.

**Step 3 — server/discover 구현.** 서버는 이 RPC를 필수로 지원해야 하므로, 자신의 지원 버전 목록·역량·서버 정보를 반환하도록 구현합니다. 라우팅 레이어에서 `/mcp` 단일 진입점을 유지하되, 요청별 자기 식별이 가능해졌으므로 게이트웨이에서 버전 협상을 먼저 떼어낼 수도 있습니다.

**Step 4 — 구독 재배선.** GET 폴링이나 `resources/subscribe`를 쓰고 있었다면 `subscriptions/listen` 스트림으로 옮깁니다. 서버 푸시가 필요한 워크로드라면 로드맵의 웹훅 방향(아래 3절)까지 함께 검토할 시점입니다.

한 가지 주의할 점: 세션 제거는 "상태가 필요 없어졌다"가 아니라 "상태의 소유권이 프로토콜에서 애플리케이션으로 넘어갔다"는 뜻입니다. 멀티플레이 세션형 도구를 만들고 있었다면 핸들 설계가 오히려 더 중요해집니다.

## 3. 다음에 오는 것: 2026 로드맵 5개 우선영역

공식 로드맵 문서(2026-08-22 기준)는 다음 다섯 가지를 우선영역으로 명시합니다. 각 영역의 SEP는 신속 심사 대상이 되고 코어 메인테이너 역량이 집중됩니다.

**① 에이전트 메시징 프리미티브.** "서버가 아직 안 끝났다"는 상황에 대한 답이 Tasks, 구독/listen, 진행 알림 등 여러 곳에 흩어져 있는 문제를 해결합니다. 생명주기·취소·오류 표면을 공유하도록 조합(compose)하는 것이 목표이며, Tasks(SEP-2663)를 코어 프로토콜로 편입하는 작업이 진행 중입니다. 웹훅을 포함한 서버 주도 푸시 전달도 이 영역의 과제입니다.

**② HTTP 네이티브 트랜스포트 통합.** 가장 흥미로운 방향은 **HTTP over stdio**입니다. 로컬 서버의 stdin/stdout 위에서 HTTP/2를 실어 멀티플렉싱된 트랜스포트를 얻고, 동시에 서브프로세스의 보안·생명주기 보장은 유지하는 구상입니다. HTTP용 기능마다 stdio용 설계가 하나씩 더 필요한 이중 파이프라인 문제를 하나의 트랜스포트 모델로 해소하려는 것입니다. 캐싱도 이 영역인데, 최근 개정에서 `ttlMs`와 `cacheScope`가 목록 결과·리소스 읽기에 추가됐고(SEP-2549), 다음은 ETag로 도구 호출 결과까지 버저닝하는 방향입니다.

**③ 에이전트 신원과 엔터프라이즈 보안.** 현재 MCP 인가 모델은 "브라우저 앞에 사람이 있는" 동의(consent) 시점을 가정하지만, 실제 호출자는 점점 사용자가 자리에 없는 클라우드 워크로드인 에이전트입니다. 붙여넣은 API 키와 장수명 리프레시 토큰 대신, DPoP(Demonstrating Proof of Possession) 스펙 확정과 에이전트 신원·위임(delegation) 표준화가 진행됩니다. 하위 에이전트가 부모보다 좁은 권한만 받는 스코핑도 다뤄집니다.

**④ 프리미티브 개선.** `tools/call`이 콘텐츠와 `structuredContent`를 동시에 반환할 수 있어 구현이 갈렸던 문제를 정리해 반환 타입의 충실도를 맞춥니다. 수백 개 도구를 한 번에 다 내려주는 대신 필요할 때 점진적으로 발견하는 progressive discovery 작업도 여기서 시작됩니다. 콘텐츠 어노테이션(SEP-2200 관련)은 채택이 저조하면 폐기(deprecation)까지 검토 대상입니다.

**⑤ SDK 개발자 경험.** 스펙과 사람이 검토하는 적합성(conformance) 테스트 스위트를 단일 진실 공급원으로 삼아, SDK와 퀵스타트를 릴리스마다 재생성·재검증하는 "generated-artifacts 실험"을 진행합니다. 손으로 유지하던 SDK를 코드젠으로 바꾸는 것이 목표입니다.

## 4. 운영자 관점에서 미리 준비할 것

- **캐싱 전략을 세우세요.** `ttlMs`/`cacheScope`가 이미 들어왔고 ETag가 다음 단계입니다. 도구 목록처럼 잘 변하지 않는 응답에 캐시 범위를 지정해 두면 로드맵의 후속 기능과 자연스럽게 맞물립니다.
- **핸들 설계에 공을 들이세요.** 세션이 사라진 만큼 상태 인코딩은 애플리케이션의 몫입니다. 핸들에 만료 시간과 서명을 포함시키면 무상태 서버에서도 남용을 방어할 수 있습니다.
- **STDIO 서버도 HTTP 마인드로.** HTTP over stdio가 현실화되면 로컬과 원격의 구현 차이가 줄어듭니다. 지금부터 Streamable HTTP 단일 바인딩으로 구조를 잡아두면 전환 비용이 작습니다.
- **SEP 동향을 팔로우하세요.** 우선영역에 속한 SEP는 가장 빨리, 가장 잘 받아들여집니다. 신규 제안을 할 계획이라면 먼저 해당 워킹그룹과 논의해 지지를 얻는 것이 공식 권장 흐름입니다.

## 결론

- 2026-07-28 개정에서 MCP는 프로토콜 세션과 `initialize` 핸드셰이크를 제거하고 완전한 스테이트리스 프로토콜이 되었습니다. 버전·역량 정보는 모든 요청의 `_meta`로 이동했습니다(SEP-2575).
- `server/discover` RPC가 필수가 됐고, 구독은 `subscriptions/listen` 단일 스트림으로 통합됐습니다. 상태는 세션이 아니라 서버 발급 핸들로 주고받습니다(SEP-2567). Roots·Sampling·Logging과 구버전 HTTP+SSE는 deprecated되었으니 신규 구현은 피하세요.
- 2026 로드맵의 5개 우선영역은 에이전트 메시징, HTTP 네이티브 트랜스포트 통합(HTTP over stdio, ETag 캐싱), 에이전트 신원·DPoP, 프리미티브 개선, SDK 코드젠 실험입니다.
- 마이그레이션의 첫 단계는 명확합니다. 세션 스토어와 세션 어피니티 설정을 걷어내고, 요청별 `_meta` 자기 식별과 `server/discover`를 구현하는 것부터 시작하면 됩니다.

## 참고 자료

- [The 2026-07-28 Specification — MCP 공식 블로그](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [2026-07-28 스펙 Key Changes(changelog)](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [MCP 공식 로드맵(2026-08-22 업데이트)](https://modelcontextprotocol.io/development/roadmap)
- [The 2026 MCP Roadmap — MCP 공식 블로그(2026-03-09)](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
