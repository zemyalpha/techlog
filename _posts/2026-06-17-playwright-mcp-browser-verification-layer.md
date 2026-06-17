---
title: "Playwright MCP를 브라우저 검증 레이어로 써야 하는 이유"
date: "2026-06-17"
keywords: ["Playwright MCP", "browser automation", "MCP", "browser verification", "agent workflow"]
lang: "ko"
description: "Playwright MCP를 생성용 도구가 아니라 브라우저 검증 레이어로 쓰는 이유, 설정법, 운영 팁을 정리한다."
---

# Playwright MCP를 브라우저 검증 레이어로 써야 하는 이유

요즘 에이전트 워크플로우를 보면, "브라우저를 열 수 있느냐"보다 "브라우저에서 본 것을 다시 검증할 수 있느냐"가 더 중요하다. 특히 폼 제출, 로그인, 리다이렉트, 버튼 활성화 같은 작업은 화면 캡처 한 장으로는 부족하다. 그래서 나는 Playwright MCP를 **생성 도구**가 아니라 **검증 레이어**로 두는 편이 훨씬 낫다고 본다.

공식 README가 이 생각을 뒷받침한다. Playwright MCP는 Playwright를 이용한 browser automation capabilities를 제공하고, 스크린샷보다 structured accessibility snapshots를 중심으로 페이지를 다룬다. 즉, 사람 눈에 보이는 이미지를 흉내 내기보다 페이지 구조와 역할(role), 상태(state)를 더 잘 읽게 해준다. 이 차이가 에이전트 디버깅에서는 꽤 크다.

## 1) Playwright MCP는 무엇이 다른가

일반적인 브라우저 자동화는 보통 "열기 → 클릭 → 스크린샷"으로 흘러간다. 그런데 에이전트 입장에서는 이 흐름이 애매하다. 화면이 비슷해 보여도 실제로는 버튼이 비활성화되어 있거나, 라벨이 잘못 연결되어 있거나, 에러 메시지가 DOM에만 남아 있을 수 있기 때문이다.

Playwright MCP는 이런 문제를 구조적으로 다룬다. 페이지를 이미지가 아니라 접근성 중심의 구조로 보게 하니, "보였다"와 "작동했다"를 분리해서 판단하기 쉽다. 그리고 공식 README는 Node.js 18 이상을 요구하고, 기본적으로는 headed 모드로 동작하며 `--headless` 옵션도 지원한다고 적고 있다. 로컬에서는 눈으로 확인하고, CI에서는 headless로 돌리는 식의 분리가 깔끔하다.

## 2) 가장 단순한 설정은 이거다

먼저 표준 MCP 설정을 넣으면 된다. 공식 README가 제시하는 기본 형태는 아래와 같다.

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

헤드리스 실행이 필요하면 인자만 추가하면 된다.

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

Claude Code를 쓰는 경우에는 README에 나온 그대로 다음 명령으로 추가할 수도 있다.

```bash
claude mcp add playwright npx @playwright/mcp@latest
```

이 정도면 브라우저 검증용 서버는 바로 붙는다. 중요한 건 "붙였다"가 아니라, 무엇을 검증할지 규칙을 먼저 정하는 것이다.

## 3) 실전에서는 생성과 검증을 분리해야 한다

내가 추천하는 패턴은 아주 단순하다.

1. 에이전트가 먼저 코드를 수정한다.
2. Playwright MCP로 브라우저에서 실제 상태를 확인한다.
3. 같은 시나리오를 Playwright 테스트로 고정한다.

예를 들어, 랜딩 페이지가 정말 정상인지 확인하려면 최소한 제목, 주요 CTA, 폼 에러 메시지 정도는 봐야 한다. 아래처럼 Playwright 테스트를 하나 두면, 나중에 에이전트가 UI를 바꿔도 회귀를 잡기 쉽다.

```ts
import { test, expect } from '@playwright/test';

test('landing page keeps the primary CTA visible', async ({ page }) => {
  await page.goto('https://example.com');

  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
  await expect(page.getByRole('link', { name: /get started|시작/i })).toBeVisible();
});
```

이 방식의 장점은 분명하다. 에이전트는 MCP로 탐색하고, 사람은 테스트로 고정한다. 이렇게 나누면 "한 번은 됐는데 다음엔 안 됨" 같은 얘기를 줄일 수 있다.

## 4) 주의할 점: MCP가 늘 최고의 실행 경로는 아니다

여기서 재미있는 대목이 하나 있다. 공식 README는 Playwright MCP를 소개하면서도, **coding agent**를 쓴다면 오히려 CLI+SKILLS가 더 token-efficient할 수 있다고 적는다. 이 말은 곧, MCP가 모든 브라우저 작업의 정답은 아니라는 뜻이다.

내 해석은 이렇다.

- 반복적인 배치 작업은 CLI가 더 낫다.
- 사람이 함께 보며 상태를 확인해야 하면 MCP가 편하다.
- 최종 회귀 방지는 결국 Playwright test 같은 고정된 테스트가 맡아야 한다.

즉, Playwright MCP는 "모든 걸 대신하는 브라우저"가 아니라, **검증을 빠르게 붙여주는 관측 레이어**에 가깝다. 이 역할을 분명히 하면 도구 선택이 훨씬 단순해진다.

## 결론: 에이전트에게 필요한 건 더 많은 클릭이 아니라 더 나은 검증이다

- Playwright MCP는 Playwright 기반 브라우저 자동화용 MCP 서버다.
- 스크린샷보다 접근성 구조를 중심으로 페이지를 읽게 해준다.
- 기본 설정은 간단하고, `--headless`로 CI에도 붙이기 쉽다.
- 하지만 반복 실행형 작업은 CLI나 Playwright 테스트가 더 적합할 수 있다.
- 그래서 가장 좋은 쓰임새는 생성이 아니라 검증이다.

오늘 바로 해볼 일은 하나다. 지금 쓰는 에이전트 흐름에 "브라우저로 확인하는 단계"를 하나 추가해 보자. 그 한 단계만 있어도, 에이전트는 훨씬 덜 믿음직한 대신 훨씬 더 재현 가능해진다.
