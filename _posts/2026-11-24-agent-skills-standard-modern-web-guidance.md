---
title: "에이전트의 낡은 지식을 치료하는 폴더 하나 — Agent Skills 표준과 구글 Modern Web Guidance 1.0 분석"
date: "2026-11-24"
keywords: ["Agent Skills", "Modern Web Guidance", "SKILL.md", "코딩 에이전트", "프롬프트 엔지니어링", "Claude Code", "웹 플랫폼"]
lang: "ko"
description: "SKILL.md 파일 하나로 에이전트에 전문 지식을 주입하는 Agent Skills 오픈 표준과, 크롬·파이어폭스·엣지가 함께 만든 Modern Web Guidance 1.0의 구조와 실전 설치법을 분석한다."
---

# 에이전트의 낡은 지식을 치료하는 폴더 하나 — Agent Skills 표준과 구글 Modern Web Guidance 1.0 분석

2026년 11월, GitHub 트렌딩 상위권을 에이전트 "스킬" 저장소들이 휩쓸었다. 이전까지 에이전트 생태계의 화두는 MCP·A2A 같은 *연결 프로토콜*이었다면, 이번 달의 화두는 **지식 패키징**이다. "에이전트를 어디에 연결할 것인가"에서 "에이전트에게 무엇을 알려줄 것인가"로 무게중심이 이동한 것이다.

이 흐름의 중심에 두 가지가 있다. 하나는 폴더 하나로 에이전트에 전문 지식을 주입하는 **Agent Skills 오픈 표준**이고, 다른 하나는 구글 크롬 팀이 이 표준 위에 웹 플랫폼 지식을 얹어 만든 **Modern Web Guidance 1.0**이다. 이 글에서는 둘의 구조와 동작 원리를 뜯어보고, 실제로 설치해서 쓰는 방법까지 정리한다.

## 1. 문제: 모델은 똑똑한데 웹은 옛날 코드를 뱉는다

프론티어 코딩 모델은 충분히 뛰어나다. 하지만 크롬 팀의 공식 블로그는 냉정하게 지적한다. "웹 개발 전문가에게 AI가 생성한 코드를 보여주면, 거의 항상 더 빠르고 접근성 있고 안전하게 만들 방법을 찾아낸다."([Modern Web Guidance 1.0 발표문](https://developer.chrome.com/blog/modern-web-guidance-v1))

원인은 단순하다. **학습 데이터에는 낡은 패턴이 넘쳐난다.** 자바스크립트 라이브러리로 툴팁을 그리던 시절의 코드, `setTimeout`으로 스크롤 애니메이션을 흉내내던 코드가 웹에 수억 개 존재한다. 반면 최신 웹 플랫폼 API — Popover API, CSS Anchor Positioning, `scheduler.yield`, `field-sizing: content` — 는 상대적으로 문서화된 사례가 적다. 모델은 "알고 있는" 2019년식 패턴으로 코드를 쓰고, 결과물은 동작하지만 낡은 웹이 되는 것이다.

크롬 팀의 평가에서 프론티어 모델들이 여전히 놓친다는 영역은 구체적이다.

- **접근성**: 키보드 포커스와 `inert` 관리, `prefers-reduced-motion` 대응, 접근성 있는 폼 검증
- **사용자 경험**: 터치 제스처, 다이얼로그·팝오버 전환 애니메이션, 라이트/다크 테마 구현
- **성능**: 메인 스레드 작업 스케줄링, 오프스크린 렌더링 격리, 레이아웃 시프트 방지
- **보안**: 안전한 HTML 살균(sanitization), 엄격한 CSP, 패스키 자격증명 관리

이건 모델이 멍청해서가 아니라 **컨텍스트에 최신 지식이 없어서** 생기는 문제다. 그래서 해법도 파인튜닝이 아니라 컨텍스트 주입이다.

## 2. Agent Skills: SKILL.md 파일 하나로 만드는 지식 패키지

Agent Skills의 정의는 놀랄 만큼 단순하다. **폴더 하나, 그 안에 `SKILL.md` 파일 하나.**

```
my-skill/
├── SKILL.md          # 필수: 메타데이터 + 지시사항
├── scripts/          # 선택: 실행 가능한 코드
├── references/       # 선택: 참고 문서
└── assets/           # 선택: 템플릿, 리소스
```

`SKILL.md`은 YAML 프론트매터와 마크다운 본문으로 구성된다. 프론트매터에는 `name`(최대 64자, 소문자·숫자·하이픈)과 `description`(최대 1024자)이 필수고, `license`, `compatibility`, `metadata`, `allowed-tools`는 선택이다.

```markdown
---
name: tooltip-anchoring
description: CSS Anchor Positioning으로 자바스크립트 없이 툴팁과 팝오버를 배치하는 방법. 툴팁, 메뉴, 팝오버 위치 지정 작업에 사용.
license: Apache-2.0
metadata:
  author: my-org
  version: "1.0"
---

# 툴팁 앵커링

1. 대상 요소에 `anchor-name`을 지정한다.
2. 툴팁에 `position-anchor`와 `position-area`를 설정한다.
3. 자세한 폴백 전략은 references/fallbacks.md를 참고한다.
```

이 단순함이 핵심 설계다. 핵심 메커니즘은 **점진적 공개(progressive disclosure)** 3단계다.

1. **발견(Discovery)**: 에이전트 구동 시 모든 스킬의 `name`과 `description`만 로드한다. 스킬당 약 100 토큰 수준.
2. **활성화(Activation)**: 작업이 description과 매칭되면 그때 전체 `SKILL.md`를 컨텍스트에 넣는다. 권장 상한 5,000 토큰 미만.
3. **실행(Execution)**: 본문에서 참조하는 `scripts/`, `references/` 파일은 실제로 필요할 때만 읽는다.

덕분에 수십 개의 스킬을 상시 장착해도 초기 컨텍스트 비용은 미미하다. OpenAI의 Codex 문서는 이 초기 스킬 목록을 "컨텍스트 윈도우의 약 2%, 크기를 알 수 없을 때는 8,000자"로 제한한다고 명시한다. 스킬이 많아지면 description을 먼저 줄이고, 그래도 넘치면 일부 스킬을 목록에서 생략한다.

### 표준의 연대기

- **2025년 10월**: Anthropic이 Claude에 스킬 시스템을 도입했다. 사내 코드는 잘 다루지만 실무 절차 지식이 부족한 에이전트의 한계를 해결하기 위해서였고, Claude Code 개발 중 탄생했다.([Anthropic 연구 글](https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills))
- **2025년 12월**: Agent Skills를 별도 GitHub 조직과 도메인(agentskills.io)으로 분리한 오픈 표준으로 공개했다. 출시 몇 주 만에 개발자들이 수천 개의 스킬을 GitHub에 올리고 Codex·Cursor·OpenCode용 로더를 만들었다.
- **2026년 10월 22일**: Anthropic이 표준을 **Agentic AI Foundation(AAIF)**에 기증했다. AAIF는 Anthropic이 Block, OpenAI와 함께 설립하고 MCP를 기증한 리눅스 재단 산하 중립 재단이다.

Anthropic 발표에 따르면 표준 공개 후 10개월 동안 **70개 이상의 에이전트 플랫폼**(ChatGPT, Cursor, GitHub Copilot, VS Code 등)이 Agent Skills를 통합했고, **약 100만 개의 스킬이 게시되어 거의 2억 8천만 회 설치**됐다. 특정 회사의 사양이 아니라 사실상 업계 공통 포맷이 된 셈이다. 구현체별 문서는 [OpenAI Codex 스킬 문서](http://developers.openai.com/codex/skills)처럼 각사가 직접 유지하고 있다.

## 3. Modern Web Guidance 1.0: 브라우저 3사가 함께 만드는 스킬

Agent Skills 표준 위에 얹힌 가장 큰 공식 패키지가 구글 크롬 팀의 **Modern Web Guidance**다. 2026년 5월 Google I/O에서 초기 공개됐고, **2026년 11월 10일 버전 1.0 정식 버전**이 나왔다.

이 프로젝트의 특별한 점은 거버넌스다. 1.0부터 **Firefox(Mozilla)와 Edge(Microsoft) 팀이 참여하는 크로스 브라우저 협력**으로 전환됐다. 모든 가이드에 Baseline 호환성 데이터와 폴백이 붙는다. 브라우저 벤더들이 자기들 지식을 에이전트용 스킬로 패키징해 배포하는, 웹의 역사에서 처음 있는 구도다.

### 동작 구조

Modern Web Guidance는 단순한 마크다운 묶음이 아니라 CLI가 붙은 스킬이다. 공식 문서가 설명하는 흐름은 다음과 같다.

1. **활성화**: 코딩 에이전트가 웹 작업을 만나면 `modern-web-guidance` 스킬을 활성화한다.
2. **로컬 의미 검색**: 에이전트가 `modern-web-guidance search "<query>"`를 실행한다. 오프라인 임베딩 모델 + BM25 검색엔진으로 최적의 가이드를 찾는다. **네트워크 호출도, API 키도 없다.**
3. **가이드 가져오기**: `modern-web-guidance retrieve <guide-id>`로 해당 가이드를 컨텍스트에 주입한다.

검색 쿼리는 "2~5단어의 동작 중심 쿼리"가 권장된다. 예를 들어 `dialog`처럼 짧게 쓰지 말고 `light dismiss dialog`처럼 쓰고, `responsive table dark mode` 같은 복합 쿼리는 두 번으로 나누라는 식이다. LLM이 검색을 잘하게 만드는 프롬프트 설계가 스킬 본문에 박제되어 있다.

### 설치와 현대화 워크플로우

설치는 프로젝트 디렉터리에서 한 줄이다.

```bash
npx modern-web-guidance@latest install
```

Claude Code, Codex, Cursor, Antigravity, Copilot, WebStorm 등의 플러그인 마켓플레이스로도 설치할 수 있다. 1.0에서 새로 생긴 **현대화 워크플로우**는 기존 HTML·CSS·JavaScript·`package.json`을 스캔해서 안전하게 업그레이드할 지점을 찾고, 에이전트가 단계별로 변경을 수행하도록 안내한다. 레거시 코드베이스를 에이전트에게 맡겨 정리할 때 유용하다.

1.0의 구체적 변화는 네 가지다.

- **200개 이상의 가이드**: 성능, 접근성, 보안, 프라이버시, 폼, 패스키, WebMCP, 빌트인 AI 등 영역별 전문가 검수를 거친 가이드 데이터베이스
- **크로스 브라우저 기본**: Firefox·Edge 팀 협업으로 모든 가이드에 Baseline 호환성 데이터와 폴백 포함
- **공개 평가 대시보드**: 수천 번의 평가 실행 결과를 공개([대시보드](https://googlechrome.github.io/modern-web-guidance-src/)), 스킬 유무에 따른 통과율 차이를 볼 수 있다
- **응답 투명성**: 스킬이 가이드를 사용한 모든 응답 끝에 "어떤 가이드를 적용했고, 어떤 API를 썼고, 어떤 문제를 예방하는지" 목록을 붙이도록 지시한다

마지막 항목이 흥미롭다. 스킬이 단순히 지식을 주입하는 게 아니라 **출처를 밝히게 만들어서** 환각과 구분되는 검증 가능한 출력을 만드는 것이다.

## 4. 이 흐름이 의미하는 것

### 지식이 코드보다 병목이 되는 시대

지난 1년간 에이전트 인프라 논의는 도구 연결(MCP)과 에이전트 간 통신(A2A)에 집중됐다. 그런데 연결이 해결되고 나니 남는 병목은 **에이전트가 쓰는 지식의 신선도**였다. Agent Skills는 이 병목을 "폴더 + 마크다운 + 점진적 로드"라는 지극히 단순한 형태로 풀었다. 파인튜닝보다 저렴하고, RAG보다 예측 가능하고, 프롬프트에 박는 것보다 재사용 가능하다.

### 공급자가 지식을 내려주는 구조

Modern Web Guidance의 진짜 의미는 성능 개선 수치가 아니라 **브라우저 벤더가 퍼스트파티 지식을 에이전트용으로 패키징했다**는 점이다. 웹 플랫폼의 정확한 지식은 사실 브라우저 벤더만이 보증할 수 있다. 그들이 문서를 스킬로 내놓기 시작하면, 에코시스템 전체의 지식 신선도가 벤더 업데이트 주기를 따라간다. 다른 도메인 — 데이터베이스 벤더, 클라우드 프로바이더, 보안 표준 기구 — 에서도 같은 패턴이 복제될 가능성이 크다.

### 직접 스킬을 만들어야 할 때

여기서 개인 개발자가 당장 취할 행동은 명확하다. **자신의 절차 지식을 스킬로 만드는 것.** 회사의 배포 절차, 개인의 글쓰기 워크플로우, 반복되는 코드 리뷰 체크리스트 — 에이전트가 매번 새로 설명해야 했던 것들이 전부 스킬 후보다. 형식은 위에서 본 것처럼 단순하고, 한 번 만들면 표준을 지원하는 어느 에이전트에서든 쓸 수 있다.

## 결론

- Agent Skills는 `SKILL.md` 파일 하나로 에이전트에 절차 지식을 주입하는 오픈 표준이다. 2025년 12월 표준 공개 후 70개 이상 플랫폼이 통합했고, 2026년 10월 리눅스 재단 산하 AAIF로 소유권이 이전됐다.
- 핵심 메커니즘은 점진적 공개 3단계다. 메타데이터(~100 토큰) → 본문(<5,000 토큰 권장) → 참조 파일(필요 시). 컨텍스트 비용을 최소화하면서 무제한에 가까운 지식을 장착할 수 있다.
- 구글의 Modern Web Guidance 1.0은 이 표준 위에 웹 플랫폼 지식 200개 이상의 가이드를 얹은 것이고, Firefox·Edge가 참여하는 크로스 브라우저 프로젝트가 됐다. `npx modern-web-guidance@latest install` 한 줄로 시작한다.
- 모델의 낡은 학습 데이터 문제는 파인튜닝이 아니라 컨텍스트 주입으로 푸는 게 현재의 정답처럼 보인다. 벤더가 퍼스트파티 지식을 스킬로 내려주는 패턴은 다른 도메인으로도 확산 중이다.

지금 당장 해볼 일: 웹 프로젝트 하나에서 `npx modern-web-guidance@latest install`을 실행하고, 에이전트에게 팝오버나 다이얼로그 컴포넌트를 하나 만들어보라. 응답 끝에 붙는 "적용한 가이드" 목록에서 2019년식 코드와 2026년식 코드의 차이를 직접 확인할 수 있다.

## 참고 자료

- [Modern Web Guidance 1.0 발표 (Chrome for Developers, 2026-11-10)](https://developer.chrome.com/blog/modern-web-guidance-v1)
- [Modern Web Guidance 시작하기 (Chrome 공식 문서)](https://developer.chrome.com/docs/modern-web-guidance/get-started)
- [Agent Skills 사양 (agentskills.io)](https://agentskills.io/specification)
- [Agent Skills의 AAIF 기증 발표 (Anthropic, 2026-10-22)](https://www.anthropic.com/news/donating-agent-skills-to-the-agentic-ai-foundation)
- [Equipping agents for the real world with Agent Skills (Anthropic, 2025-10-16)](https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills)
- [Agent Skills – Codex (OpenAI 개발자 문서)](http://developers.openai.com/codex/skills)
- [GoogleChrome/modern-web-guidance (GitHub)](https://github.com/GoogleChrome/modern-web-guidance)
