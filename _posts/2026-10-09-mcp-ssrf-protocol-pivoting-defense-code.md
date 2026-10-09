---
title: "구글·JP모건·프랑스 정부가 같은 실수를 했다 — MCP SSRF '프로토콜 피벗팅' 해부와 방어 코드"
date: "2026-10-09"
keywords: ["MCP 보안", "SSRF", "protocol pivoting", "Model Context Protocol", "AI 에이전트 보안", "CVE-2026-14540"]
lang: "ko"
description: "2026년 10월 공개된 MCP SSRF 다수 기관 동시 발견 사례와 '프로토콜 피벗팅' 공격 모델을 분석하고, Go·Python으로 바로 쓸 수 있는 SSRF 가드 구현과 점검 체크리스트를 정리한다."
---

# 구글·JP모건·프랑스 정부가 같은 실수를 했다 — MCP SSRF '프로토콜 피벗팅' 해부와 방어 코드

2026년 10월 초, 보안 커뮤니티에서 흥미로운 기사가 나왔다. 독립 보안 연구자 시드 아나스 모히우딘(Syed Anas Mohiuddin)이 5개월에 걸쳐 서로 아무 상관 없는 조직들의 MCP(Model Context Protocol) 서버에서 같은 결함을 반복해서 찾아냈다는 것이다. 구글, JP모건 체이스, 벡터 DB 회사 Weaviate, 프랑스 정부 디지털청(DINUM), 인도네시아 탕가랑 시청이 각각 자체 개발한 MCP 서버에서 같은 취약점 유형을 수정했다.

이 목록을 보고 "구글도 실수하는군"으로 넘기면 핵심을 놓친다. 이 다섯 곳은 코드도, 소유주도, 업계도, 국가도 공유하지 않는다. 그런데도 같은 버그가 나왔다. **공유하는 것은 MCP라는 프로토콜의 구조적 가정, 그 하나뿐이다.** 그리고 미국 정부(GSA 산하 TTS)의 연방 MCP 서버 5곳은 취약점이 보고된 지 6주가 지나도록 아직 수정되지 않은 상태라고 한다.

이 글에서는 이 사례의 기술적 원리인 SSRF(Server-Side Request Forgery), 여기서 한 단계 더 나아간 "프로토콜 피벗팅" 개념, 그리고 내 MCP 서버에 바로 적용할 수 있는 방어 코드를 다룬다.

## 왜 MCP 서버는 SSRF에 특히 취약한가

SSRF 자체는 20년 된 고전 취약점이다. 서버가 사용자 입력으로 받은 URL이나 경로를 검증 없이 요청에 사용하면, 공격자는 서버를 발판으로 내부망(예: 클라우드 메타데이터 엔드포인트 `http://169.254.169.254/`)에 요청을 보낼 수 있다.

문제는 **MCP의 아키텍처가 이 고전 취약점의 발생 조건을 구조적으로 만들어낸다**는 점이다.

일반적인 웹 API는 URL을 개발자가 정의한다. 반면 MCP 서버의 도구(tool)는 LLM 에이전트가 호출하고, **URL·경로·파라미터를 에이전트가 조립해 넘긴다**. 개발자 입장에서 보면 "내부적으로나 쓰일 값"이 사실상 외부 입력인 셈이다. 여기에 이중으로 위험한 지점이 있다:

1. **직접 주입** — 악의적 프롬프트가 에이전트를 속여 내부 주소로 향하는 파라미터를 만들게 한다.
2. **간접 주입** — 도구가 가져온 웹페이지·문서 안에 심어진 지시문이 에이전트를 조종한다.

실제로 구글의 MCP Toolbox for Databases(CVE-2026-14540, CVSS 8.0)는 정확히 이 그림이었다. NVD 기록에 따르면 HTTP 소스의 내부 클라이언트(`internal/sources/http/http.go`)가 `CheckRedirect` 정책 없이 초기화되어 있었고 대상 IP 검증도 없었다. 조작된 path 파라미터 하나면 툴박스가 리다이렉트를 따라 내부·외부 임의 엔드포인트로 요청을 날렸다.

나머지 사례도 패턴은 같다. JP모건의 문서 검색 MCP 서버는 두 도구 중 하나는 도메인 허용 목록을 검사했지만 다른 하나는 호출자가 준 URL을 그대로 가져왔다(흥미롭게도 원조인 AWS 프로젝트는 애초에 호출자 URL을 가져오는 기능이 없었다고 한다). 탕가랑 시청의 Wazuh MCP 서버는 "SSRF 방지 기능이 있다"고 광고했지만 문자열 그대로의 IP 주소만 거부했고 호스트명은 전혀 리졸브하지 않았다 — 문자열 검사로 IP를 막아도 `localhost`나 `127.0.0.1.nip.io` 같은 호스트명을 쓰면 우회된다는 기본을 보여주는 사례다.

## '프로토콜 피벗팅' — 신뢰가 프로토콜 경계에서 유출되는 지점

모히우딘이 이름 붙인 더 큰 공격 클래스는 "프로토콜 피벗팅(protocol pivoting)"이다. 그의 정의를 Ars Technica 보도에서 그대로 옮기면, **"한 프로토콜로 초기 접근권을 얻은 뒤, 프로토콜 사이의 신뢰 가정을 악용해 다른 프로토콜에서만 접근 가능한 권한으로 escalate하는 다단계 공격"**이다.

구체적인 흐름은 이렇다:

1. 공격자는 MCP 도구가 반환하는 콘텐츠(웹페이지, 이슈, 문서) 안에, 구글 A2A(Agent-to-Agent) 프로토콜 작업 형식처럼 보이는 텍스트를 심는다.
2. 오케스트레이터 에이전트가 이를 정상 위임 작업으로 착각하고 서브 에이전트에게 전달한다.
3. 서브 에이전트는 오케스트레이터를 신뢰하므로 지시를 실행한다. 그리고 MCP 서버는 종종 해당 에이전트의 자격증명을 보유하고 있다.

핵심 대사는 Rapid7의 더글러스 맥키(Douglas McKee)가 Ars Technica에 한 다음 문장이다: **"각 프로토콜은 자기 혼자 존재한다고 가정하고 설계됐기 때문에 각자 자기 현관문만 검사하고, 그 사이의 복도를 아무도 감시하지 않는다."** 각 구성 요소는 설계대로 정확히 작동했다는 점이 이 공격을 잡기 어렵게 만든다.

독일 보안업체 X41 D-Sec의 마르쿠스 베르비어(Markus Vervier)는 이를 굳이 새 이름을 붙일 것 없이 간접 프롬프트 주입의 한 하위류로 보아야 한다고 지적했다. 프롬프트 주입이 A2A 같은 다른 프로토콜을 거쳐 들어오는지 여부는 공격 성립에 필수가 아니라는 것이다. 두 관점 모두 타당하지만, 실무자에게 중요한 교훈은 같다. **LLM이 도구에 넘기는 모든 값은 '인터넷의 낯선 사람이 보낸 입력'으로 취급하라** — 프롬프트 주입 시나리오에서는 사실상 그렇기 때문이다. 그 아래 깔린 버그는 주입과 SSRF라는 20년 된 고전이고, 수정 방법도 20년째 변하지 않았다.

## 방어 코드 1: Go — 리다이렉트와 IP를 함께 검사하는 HTTP 클라이언트

구글의 실제 수정(PR #3448, 2026년 6월 병합)에서 직접 배울 점이 있다. 커밋 요약에 따르면 DNS 리바인딩(TOCTOU) 방지용 `SSRFGuard`, 사설 네트워크 허용 여부와 IP 대역 허용/차단 설정, 그리고 요청 시점이 아니라 **시작 시점에 BaseURL을 검사해 조기 실패(fast-fail)**시키는 구조다.

핵심 원리만 추리면 다음과 같은 형태가 된다:

```go
// net.Resolver를 고정 IP로 재정의해 DNS 리바인딩(TOCTOU)을 차단하는 다이얼러
var ssrfResolver = &net.Resolver{
	PreferGo: true,
	Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
		d := net.Dialer{}
		return d.DialContext(ctx, "udp", "8.8.8.8:53") // 검증·요청에 동일 DNS 사용
	},
}

// 대상 주소가 사설/링크로컬/루프백인지 검사
func isBlockedIP(ip net.IP) bool {
	blocked := []net.IPNet{
		{IP: net.IPv4zero, Mask: net.CIDRMask(8, 32)},          // 0.0.0.0/8
		{IP: net.ParseIP("10.0.0.0"), Mask: net.CIDRMask(8, 32)},
		{IP: net.ParseIP("172.16.0.0"), Mask: net.CIDRMask(12, 32)},
		{IP: net.ParseIP("192.168.0.0"), Mask: net.CIDRMask(16, 32)},
		{IP: net.ParseIP("169.254.0.0"), Mask: net.CIDRMask(16, 32)}, // 링크로컬(메타데이터)
		{IP: net.ParseIP("127.0.0.0"), Mask: net.CIDRMask(8, 32)},
	}
	for _, n := range blocked {
		if n.Contains(ip) {
			return true
		}
	}
	return ip.IsLoopback() || ip.IsLinkLocalUnicast()
}

client := &http.Client{
	// 리다이렉트를 따라가기 전에 매번 최종 호스트를 재검사 (CVE-2026-14540의 직접 원인)
	CheckRedirect: func(req *http.Request, via []*http.Request) error {
		host := req.URL.Hostname()
		ips, err := ssrfResolver.LookupIP(context.Background(), "ip", host)
		if err != nil {
			return fmt.Errorf("SSRF guard: host lookup failed: %w", err)
		}
		for _, ip := range ips {
			if isBlockedIP(net.ParseIP(ip.String())) {
				return fmt.Errorf("SSRF guard: redirect to blocked IP %s", ip)
			}
		}
		return nil
	},
}
```

빠뜨리기 쉬운 두 가지를 짚는다:

- **문자열 검사 금지** — 탕가랑 사례가 보여주듯 리터럴 IP 문자열만 거부하는 검사는 호스트명으로 우회된다. 반드시 DNS 리졸브 뒤 IP를 검사한다.
- **리다이렉트마다 재검사** — 최초 URL만 검사하면 안전한 도메인→사설 IP로 연결되는 302 한 번으로 무너진다. `CheckRedirect` 훅에서 매 hop을 검사해야 한다.
- **DNS 리바인딩 대비** — 검증 시점과 요청 시점에 DNS 응답이 다를 수 있으므로(TOCTOU), 커스텀 `DialContext`에서 검증된 IP로 직접 연결하는 방식이 안전하다.

## 방어 코드 2: Python — 자체 MCP 서버의 fetch 도구에 거는 허용 목록

Python으로 MCP 서버를 짰다면 URL을 만지는 모든 도구 입구에서 동일한 원칙을 적용한다:

```python
import ipaddress, socket, urllib.parse
from functools import lru_cache

ALLOWED_HOSTS = {"api.example.com", "cdn.example.com"}

@lru_cache(maxsize=1024)
def _resolve_ips(host: str) -> tuple:
    return tuple(socket.getaddrinfo(host, None, socket.AF_UNSPEC, socket.SOCK_STREAM))

def assert_safe_url(url: str) -> None:
    p = urllib.parse.urlparse(url)
    if p.scheme not in ("http", "https"):
        raise ValueError(f"scheme not allowed: {p.scheme!r}")
    if p.hostname not in ALLOWED_HOSTS:                # 1) 도메인 허용 목록
        raise ValueError(f"host not in allowlist: {p.hostname!r}")
    for info in _resolve_ips(p.hostname):              # 2) 리졸브 결과 IP 검사
        ip = ipaddress.ip_address(info[4][0])
        if ip.is_private or ip.is_loopback or ip.is_link_local or ip.is_reserved:
            raise ValueError(f"resolves to blocked IP: {ip}")
```

MCP 컨텍스트에서 한 층 더해야 할 것:

- **"읽기 전용"을 과신하지 않기** — 웹을 읽는 도구도 SSRF 봉쇄 없이는 내부 메타데이터를 읽는 쓰기급 위력을 가진다. 안전 분류는 도구가 아니라 도구의 접근 범위가 결정한다.
- **출력도 입력이다** — 도구가 반환한 텍스트에 다음 도구 호출을 조종하는 지시문이 섞일 수 있다(간접 프롬프트 주입). 반환 콘텐츠를 그대로 신뢰하는 구조 자체를 재점검한다.
- **도구 권한 분리** — 하나의 MCP 서버에 모든 자격증명을 몰아넣지 않는다. 피벗팅은 "한 에이전트의 신뢰 → 전체 자격증명" 경로를 노린다.

## 내 MCP 서버 10분 점검 체크리스트

1. URL·경로·호스트를 파라미터로 받는 도구가 있는가? 있다면 허용 목록이 있는가?
2. HTTP 클라이언트의 리다이렉트 정책이 설정돼 있는가? 리다이렉트마다 대상을 재검사하는가?
3. 호스트명을 리졸브해서 IP 기준으로 사설 대역을 차단하는가? (문자열 비교가 아닌)
4. `169.254.169.254`(클라우드 메타데이터) 등 링크로컬 대역 차단이 빠져 있지 않은가?
5. BaseURL/기본 경로를 시작 시점에 검증해 조기 실패시키는가?
6. 에이전트 간 위임(A2A 등)으로 들어온 작업 지시를 무검증으로 실행하는 경로가 있는가?
7. 도구 응답에 자격증명·PII가 로그로 남고 있지 않은가? (미수정 상태로 남은 VA 사례의 교훈)

## 결론

- 서로 무관한 5개 조직에서 같은 취약점이 나왔다는 것은 우연이 아니라 MCP 구조가 만드는 **체계적 오류**다. 에이전트가 조립하는 입력은 전부 외부 입력이다.
- 프로토콜 피벗팅은 새로운 마법이 아니라 **간접 프롬프트 주입 + 고전 SSRF**의 조합이며, 각 프로토콜이 자기 현관문만 검사하는 사이의 복도를 노린다.
- 구글의 실제 수정은 다르지 않다: 리다이렉트 정책, IP 대역 허용/차단, DNS 리바인딩 가드, 시작 시점 fast-fail. 20년 된 SSRF 방어 수칙의 MCP 버전이다.
- 오늘 할 일 하나를 꼽자면: URL을 받는 내 MCP 도구에 `assert_safe_url` 같은 허용 목록 검사를 다는 것. 30분이면 되고, 다섯 글로벌 조직이 놓쳤던 바로 그 지점을 막는다.

**참고 자료**
- Ars Technica, "MCP for agent-to-agent comms may be the riskiest protocol you've never heard of" (2026-10-05, Dan Goodin)
- The Next Web, "Google, JPMorgan and two governments fixed the same MCP flaw" (2026-10-06)
- NVD CVE-2026-14540 (CVSS 4.0 8.0, googleapis/mcp-toolbox 0.3.0–1.4.0, finder: Syed Anas Mohiuddin)
- googleapis/mcp-toolbox PR #3448 "fix(source/http): implement SSRF guard" (2026-06-18 병합)
