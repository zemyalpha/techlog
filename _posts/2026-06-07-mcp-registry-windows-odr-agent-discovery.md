---
title: "왜 MCP Registry와 Windows ODR가 중요한가: 에이전트 도구가 '발견 가능한 인프라'가 되는 순간"
date: "2026-06-07"
keywords: ["MCP Registry", "Windows ODR", "Model Context Protocol", "mcp-publisher", "agent connectors"]
lang: "ko"
description: "MCP Registry와 Windows ODR를 공식 문서 기준으로 정리한다. MCP 서버를 찾고, 버전을 조회하고, Windows에서 안전하게 등록하는 실전 흐름을 설명한다."
---

# 왜 MCP Registry와 Windows ODR가 중요한가: 에이전트 도구가 '발견 가능한 인프라'가 되는 순간

MCP는 AI 애플리케이션이 외부 시스템과 연결되는 오픈 표준이다. 그런데 실무에서 더 중요한 문제는 규격 자체가 아니라, **어떤 서버를 어디서 찾고, 어떻게 배포하고, 누가 허용하느냐**다. 최근 공식 MCP Registry와 Windows의 ODR(On-device Agent Registry)을 보면, MCP는 더 이상 개별 앱의 설정 파일만으로 관리하는 기술이 아니다. 이제는 **발견 가능성(discoverability)** 과 **관리 가능성(manageability)** 의 문제다.

예전에는 각 팀이 MCP 서버 주소를 README나 위키에 따로 적어 두고, 클라이언트마다 JSON 설정을 손으로 맞췄다. 하지만 레지스트리가 들어오면 클라이언트는 서버 목록을 조회하고, 최신 버전과 변경 이력을 따라가고, 관리자는 정책으로 접근을 통제할 수 있다. 즉, MCP 서버는 “숨겨진 설정”이 아니라 “조회 가능한 인프라”가 된다.

## 1) Registry가 바꾸는 것

MCP Registry의 핵심은 단순하다. 공식 문서와 저장소 설명은 이 레지스트리를 MCP 서버 목록으로 보며, 앱스토어 같은 발견 경험을 제공한다고 설명한다. 공식 API에는 `GET /v0.1/servers`로 목록을 조회하고, `GET /v0.1/servers/{serverName}/versions`로 버전 이력을 보는 흐름이 있다. 페이지네이션은 커서 기반이므로 `nextCursor`를 그대로 다음 요청에 넘겨야 한다. 커서는 사람이 조립하는 문자열이 아니다.

```bash
curl "https://registry.modelcontextprotocol.io/v0.1/servers?search=filesystem&version=latest"
```

이 한 줄로도 실무 감각이 달라진다. 더 이상 “이 서버가 아직 살아 있나?”를 README에서 추측하지 않고, 검색 조건으로 바로 확인할 수 있다. 공식 레지스트리는 `search`, `updated_since`, `version`, `include_deleted` 같은 쿼리도 제공하므로, 간단한 탐색과 증분 동기화를 같은 API에서 처리할 수 있다.

```python
import json
import urllib.parse
import urllib.request


def fetch_servers(search, cursor=None):
    params = {"search": search, "version": "latest", "limit": "10"}
    if cursor:
        params["cursor"] = cursor
    url = "https://registry.modelcontextprotocol.io/v0.1/servers?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
    with urllib.request.urlopen(req, timeout=20) as resp:
        return json.load(resp)


data = fetch_servers("filesystem")
print(data["metadata"]["nextCursor"])
```

이 패턴의 장점은 명확하다. 검색, 최신 버전 선택, 증분 동기화가 모두 같은 스키마로 정리된다.

## 2) 배포는 `mcp-publisher`로, 메타데이터는 `server.json`으로

공식 quickstart는 `mcp-publisher init`, `login github`, `publish` 흐름을 안내한다. 중요한 포인트는 레지스트리가 아티팩트를 호스팅하는 게 아니라 **메타데이터를 호스팅**한다는 점이다. 그래서 실제 패키지 배포와 레지스트리 등록을 분리해서 생각해야 한다.

```json
{
  "name": "io.github.my-username/weather",
  "description": "An MCP server for weather information.",
  "repository": {
    "url": "https://github.com/my-username/mcp-weather-server",
    "source": "github"
  },
  "version": "1.0.0",
  "packages": [
    {
      "registryType": "npm",
      "identifier": "@my-username/mcp-weather-server",
      "version": "1.0.0",
      "transport": { "type": "stdio" }
    }
  ]
}
```

여기서 `name`은 패키지의 `mcpName`과 맞아야 한다. 또한 공식 레지스트리는 `io.github.*` 네임스페이스에 GitHub OAuth를 사용한다. 즉, “아무나 아무 이름으로 올리는 구조”가 아니라, 네임스페이스 단위 검증이 들어간다.

## 3) Windows ODR가 왜 다르냐

Microsoft는 Windows의 MCP 문서에서 ODR을 “local apps와 remote servers의 agent connector를 발견하고 사용하는 secure, manageable interface”라고 설명한다. 이게 중요한 이유는 데스크톱 에이전트가 더 이상 개인 설정 파일만 보고 움직이지 않기 때문이다. Windows 쪽에서는 발견성, 보안, 사용자/관리자 제어, 로깅과 감사 가능성을 같이 다룬다.

특히 문서가 강조하는 차이는 분명하다. 패키지 identity가 있는 앱은 설치/제거 시 서버가 자동 등록·해제될 수 있지만, `.exe`나 MSI 같은 package identity 없는 배포는 직접 설치가 가능하더라도 안전하게 격리된 agent process에서 동작하지 않으며, Windows Settings에서 보호 수준을 낮추지 않으면 ODR에 노출되지 않는다. 이건 편의성보다 보안을 우선한 설계다.

## 4) 실전 팁

- 탐색은 `search`, 동기화는 `updated_since`, 최신 선택은 `version=latest`로 나눠라.
- 페이지네이션의 `cursor`는 절대 가공하지 말고 그대로 넘겨라.
- 레지스트리 등록과 패키지 배포는 분리해서 설계하라.
- Windows에서는 “무조건 등록”보다 “package identity로 자동 관리”가 더 안전하다.

## 결론: MCP는 이제 설정이 아니라 유통과 통제의 문제다

- MCP Registry는 MCP 서버를 찾고 동기화하는 공식 인덱스다.
- 공식 API는 `/v0.1/servers`와 `/versions`를 제공하고, 커서 기반 페이지네이션을 쓴다.
- `mcp-publisher`는 서버 메타데이터를 레지스트리에 올리는 공식 경로다.
- Windows ODR는 로컬/원격 MCP 서버를 발견·통제·감사하는 OS 수준 인터페이스다.
- 직접 설치형 번들은 편하지만, ODR의 안전한 경로와는 다를 수 있다.

오늘 바로 할 일은 하나다. 지금 운영 중인 MCP 서버 하나를 골라서, README 링크 대신 레지스트리 기준으로 검색 가능한 이름과 버전 정보를 정리해 보자. 그 순간부터 MCP는 “숨겨진 설정”이 아니라 “발견 가능한 제품”이 된다.
