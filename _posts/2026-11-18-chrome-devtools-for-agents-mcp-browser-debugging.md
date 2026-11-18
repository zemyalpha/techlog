---
title: "AI 코딩 에이전트가 브라우저를 직접 들여다본다 — Chrome DevTools for agents 실전 가이드"
date: "2026-11-18"
keywords: ["Chrome DevTools MCP", "코딩 에이전트", "MCP", "브라우저 디버깅", "성능 트레이스", "chrome-devtools-mcp"]
lang: "ko"
description: "chrome-devtools-mcp 1.13의 구조(MCP 서버·CLI·스킬)와 설치법, get_css_styles·1.2GB 청크 트레이스 파서 등 2026년 가을 업데이트, 그리고 보안 주의사항까지 정리한다."
---

# AI 코딩 에이전트가 브라우저를 직접 들여다본다 — Chrome DevTools for agents 실전 가이드

AI 코딩 에이전트는 하루가 다르게 코드를 잘 쓰게 되었지만, 정작 그 코드가 브라우저에서 **실제로 어떻게 동작하는지는 보지 못했다**. 컴포넌트를 고쳤는데 화면이 왜 깨지는지, 네트워크 요청이 왜 실패하는지, LCP가 왜 느린지는 개발자가 직접 DevTools를 열어 확인한 뒤 에이전트에게 다시 설명해야 했다. 구글이 2025년 9월 Chrome DevTools MCP 서버를 공개 프리뷰로 내놓으며 이 문제를 "눈가리개를 쓴 프로그래밍"이라고 표현한 것도 같은 맥락이다.

1년이 지난 지금, 이 프로젝트는 'Chrome DevTools for agents'라는 이름의 정식 도구 모음으로 자리 잡았다. MCP 서버, CLI, 에이전트 스킬 세 가지로 구성되어 있고, npm 패키지 `chrome-devtools-mcp`는 2026년 11월 11일자로 1.13.0 버전까지 올라왔다. 이 글에서는 이 도구가 무엇인지, 어떻게 설치하는지, 최근 업데이트에서 뭐가 달라졌는지, 그리고 쓸 때 무엇을 조심해야 하는지를 정리한다.

## 구조: MCP 서버 + CLI + 에이전트 스킬

Chrome DevTools for agents는 세 가지 요소로 이루어져 있다.

- **MCP 서버** — 코딩 에이전트(Claude Code, Cursor, Gemini CLI, Copilot, Codex 등)가 Model Context Protocol로 실제 Chrome 브라우저 인스턴스를 제어하고 검사한다. 자동화는 Puppeteer 위에 얹혀 있어서 요소를 클릭하거나 폼을 채운 뒤 결과를 기다리는 동작이 안정적이다.
- **CLI** — MCP 클라이언트 없이 터미널에서 직접 브라우저를 조작한다. 셸 스크립트로 브라우저 작업을 자동화할 때 유용하다.
- **에이전트 스킬** — 에이전트에게 DevTools 사용법을 가르치는 전문 지침 세트다. `a11y-debugging`(접근성), `cookie-debugging`(쿠키·세션), `debug-optimize-lcp`(LCP 개선), `memory-leak-debugging`(메모리 누수), `troubleshooting`(연결 문제) 등 7종이 패키지에 포함되어 있다.

핵심 차별점은 "HTML을 긁어오는 것"이 아니라는 점이다. 문서화되지 않은 실제 페이지를 라이브로 탐색하고, 성능 트레이스를 기록하고, 힙 스냅샷을 분석한다는 점에서 단순 웹 스크래핑 도구와는 다르다.

## 설치와 첫 실행

MCP를 지원하는 도구라면 어디든 붙일 수 있다. 공통 설정은 다음과 같다.

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

에이전트별로는 공식 설치 경로가 조금씩 다르다.

```bash
# Claude Code — 플러그인 마켓플레이스 방식 (스킬까지 함께 설치)
/plugin marketplace add ChromeDevTools/chrome-devtools-mcp
/plugin install chrome-devtools-mcp@chrome-devtools-plugins

# Codex CLI
codex mcp add chrome-devtools -- npx chrome-devtools-mcp@latest

# Gemini CLI — MCP + 스킬을 확장으로 설치 (자동 업데이트)
gemini extensions install --auto-update https://github.com/ChromeDevTools/chrome-devtools-mcp
```

Antigravity 2.0에는 아예 사전 번들되어 있어서 별도 설치 없이 `/browser` 슬래시 커맨드로 브라우저 서브에이전트를 바로 쓸 수 있다. 설치 후 동작 확인은 이 프롬프트 하나면 충분하다.

```
Check the performance of https://developers.chrome.com
```

에이전트가 브라우저를 띄우고 성능 트레이스를 기록·분석하면 성공이다. MCP 서버는 브라우저가 필요한 도구를 처음 호출하는 시점에 Chrome을 자동으로 시작한다.

CLI만 따로 쓰고 싶다면 전역 설치 후 이렇게 한다.

```sh
npm i chrome-devtools-mcp@latest -g

chrome-devtools navigate_page 1 --url "https://developers.chrome.com"
chrome-devtools take_screenshot 1 --filePath screenshot.png
chrome-devtools stop
```

## 2026년 가을 업데이트: 에이전트를 위한 DevTools가 갖춰가는 것들

### CSS 검사 — get_css_styles와 페이지네이션

10월 20일 발행된 공식 DevTools 블로그(Chrome 155/156 기능 소개)에 따르면, 에이전트용 DevTools에 `get_css_styles` 도구가 추가되었다. 계산된 스타일과 매칭 규칙을 검사하되 `pageSize`·`pageIdx`로 **페이지네이션을 지원**하고, 매칭 규칙·상속 규칙·가상 요소·`@keyframes`·`@position-try` 같은 @규칙까지 정리된 형태로 돌려준다. LLM 컨텍스트 창은 한정되어 있으니, 거대한 스타일 시트를 통째로 밀어 넣는 대시 잘게 나눠 주는 설계다. 스크린샷으로는 잡히지 않는 "이 요소에 왜 이 스타일이 적용되는가"를 에이전트가 스스로 추적할 수 있게 된 셈이다.

### 성능 트레이스 — 1.2GB 청크 파서

성능 녹화 파일은 수백 MB에서 GB 단위로 커진다. 이제 대용량 녹화를 청크 단위로 파싱하는 트레이스 버퍼 파서가 들어갔고 기본 버퍼 크기가 1.2GB다. 파싱할 때마다 트레이스 엔진 인스턴스를 스코프로 분리해 메모리 누수도 막는다. "페이지가 느리다"고 에이전트에게 던지면, 트레이스를 기록해 병목을 찾고 개선안까지 제안하는 워크플로가 실제로 돌아가는 수준이 되었다는 뜻이다.

### 그 외의 변화

같은 블로그 포스트와 저장소 체인지로그를 종합하면 다음과 같다.

- **설정 파일 지원** — 서버 플래그를 전용 설정 파일로 관리하고, 파일 시스템 루트를 지정할 수 있다. CLI에서는 `--allow-unrestricted-paths`가 기본값이 됐다.
- **다이얼로그 자동 처리** — 입력 도구 실행 중 뜨는 JavaScript 다이얼로그(alert 등)를 자동으로 처리하고, 콘솔 메시지 이력은 같은 문서 내 탐색에서 유지된다.
- **v1.12.0 (10월 22일)** — 브라우저 연결 없이 힙 스냅샷 분석, 유지할 탐색 이력 수를 정하는 `--maxNavigationSaved`, 설정 파일 변경을 실시간 반영하는 `--watchConfig`.
- **v1.13.0 (11월 11일)** — DevTools BreakpointManager 기반 로그포인트, 페이지 단위 네트워크 목(mock), 상위(업스트림) 네트워크 목 응답 지원.

네트워크 목 기능은 특히 실용적이다. 결제 API가 실패하는 상황을 재현하려고 실제 서버를 조작할 필요 없이, 에이전트가 특정 요청을 가로채 원하는 응답을 내려주게 만들면 된다.

## 쓰기 전에 알아야 할 것들

강력한 만큼 노출도 크다. 공식 문서의 경고를 그대로 옮기면 이 도구는 **브라우저 인스턴스의 콘텐츠 전부를 MCP 클라이언트에 노출**한다. 로그인 세션이 살아 있는 브라우저를 연결하면 에이전트가 사실상 나 대신 행동할 수 있다. 민감한 계정이 있는 프로필은 분리하고, 가능하면 격리된 실행(`--isolated`)이나 헤드리스(`--headless`)를 쓰는 편이 안전하다.

알아둘 점이 두 가지 더 있다.

- **사용 통계가 기본 수집된다.** 도구 호출 성공률·지연시간·환경 정보를 구글이 수집하며 옵트아웃은 `--no-usage-statistics` 플래그나 `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS` 환경변수로 한다. `CI` 환경변수가 설정되어 있으면 수집이 비활성화된다.
- **공식 지원은 Chrome과 Chrome for Testing뿐이다.** 다른 Chromium 계열 브라우저는 동작할 수 있지만 보장되지 않는다. 성능 도구는 실사용자 데이터를 얻기 위해 트레이스 URL을 CrUX API로 보낼 수 있는데, 원하지 않으면 `--no-performance-crux`로 끈다.

도구 수가 많아 토큰이 아까운 단순 자동화라면 `--slim` 플래그로 축소된 도구 세트만 노출하는 것도 방법이다.

## 마치며: 디버깅의 관점이 에이전트로 넘어온다

정리하면 이렇다.

- Chrome DevTools for agents는 MCP 서버·CLI·스킬 3종 구성이며, 에이전트가 라이브 브라우저를 직접 검사·제어하게 해준다.
- 올가을 업데이트로 CSS 규칙 검사(get_css_styles), 1.2GB 청크 트레이스 파서, 네트워크 목, 로그포인트가 갖춰져 실전 디버깅 워크플로가 가능해졌다.
- 브라우저 전체가 에이전트에 노출되는 만큼 프로필 분리·격리 모드·옵트아웃 플래그는 기본으로 챙겨야 한다.
- 설치는 JSON 한 줄 또는 각 에이전트의 공식 커맨드로 끝난다.

당장 시도해볼 첫 단계는 간단하다. 진행 중인 프로젝트 하나에 MCP 서버를 붙이고 "localhost:3000이 느리다. 성능 트레이스를 기록해 병목을 찾고 고쳐라"라고 맡겨보자. 내가 DevTools를 열어 설명해 주던 작업이 어디까지 에이전트로 넘어가는지 직접 체감할 수 있을 것이다.
