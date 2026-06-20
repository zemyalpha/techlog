---
title: "세션 로그와 브라우저 증거로 에이전트를 검증하는 법"
date: "2026-06-20"
keywords: ["GitHub Copilot app", "/chronicle", "Playwright MCP", "worktree", "browser verification"]
lang: "ko"
description: "GitHub Copilot app의 worktree, /chronicle, Playwright MCP를 묶어 에이전트 실행·기록·검증을 분리하는 실전 패턴을 정리한다."
---

# 세션 로그와 브라우저 증거로 에이전트를 검증하는 법

코딩 에이전트를 오래 써보면 한 가지 패턴이 보인다. 작업 자체는 빠른데, 나중에 보면 왜 그렇게 고쳤는지 설명이 남지 않는다. 그래서 지금 필요한 건 더 많은 프롬프트가 아니라, **실행·기록·검증을 분리하는 구조**다. GitHub Copilot app, /chronicle, Playwright MCP를 같이 보면 이 구조가 꽤 선명해진다.

GitHub의 공식 설명에 따르면 Copilot app은 에이전트가 쓰는 데스크톱 경험이고, 각 세션은 별도의 git worktree에서 돈다. 즉, 병렬 세션이 서로 브랜치를 망가뜨리지 않도록 처음부터 격리해 둔 셈이다. 여기에 /chronicle가 붙으면 각 Copilot 세션의 히스토리를 조회 가능한 형태로 바꿔 주고, 요약이나 개인화된 팁으로 다시 돌려준다. 세션이 끝난 뒤에도 "무슨 일이 있었는가"를 다시 묻을 수 있다는 뜻이다.

문제는 여기서 끝나지 않는다. 세션 기록이 아무리 잘 남아도, 실제로 웹 UI가 정상인지까지는 보장하지 못한다. 로그인, 버튼 활성화, 에러 메시지, 리다이렉트 같은 건 브라우저에서 직접 확인해야 한다. 이 지점에서 Playwright MCP가 유용하다. 공식 README는 이것을 Playwright 기반의 MCP 서버로 설명하며, 스크린샷 대신 **structured accessibility snapshots**를 쓰고, Node.js 18 이상을 요구한다고 적는다. 즉, "보였다"가 아니라 "구조적으로 읽혔다"를 기준으로 검증할 수 있다.

## 1) 권장 흐름: 세션은 worktree, 회고는 /chronicle, 확인은 브라우저

내가 추천하는 순서는 단순하다.

1. Copilot app에서 작업을 시작한다. 세션은 별도 worktree로 격리한다.
2. /chronicle로 지난 세션의 실패 패턴을 요약한다.
3. Playwright MCP로 실제 브라우저 상태를 확인한다.
4. 마지막에는 Playwright 테스트로 회귀를 고정한다.

이렇게 하면 에이전트가 남긴 흔적이 단순 로그가 아니라, 다음 실행을 개선하는 재료가 된다.

## 2) Playwright MCP는 이렇게 붙이면 된다

공식 README의 기본 예시는 간단하다.

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

헤드리스가 필요하면 인자만 하나 더 붙이면 된다.

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

그리고 README에는 coding agent를 쓸 때는 MCP보다 CLI+SKILLS가 더 token-efficient할 수 있다는 설명도 있다. 이 말은 MCP가 쓸모없다는 뜻이 아니라, **역할이 다르다**는 뜻에 가깝다. 대량 반복 작업은 CLI가 낫고, 화면 구조를 보면서 검증해야 하는 구간은 MCP가 낫다.

## 3) 실제 검증은 테스트로 고정해야 한다

브라우저에서 한 번 성공한 화면은 금방 깨진다. 그래서 확인한 경로는 테스트로 박아 두는 편이 좋다.

```ts
import { test, expect } from '@playwright/test';

test('landing page keeps the main CTA visible', async ({ page }) => {
  await page.goto('https://example.com');

  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
  await expect(page.getByRole('link', { name: /get started|시작/i })).toBeVisible();
});
```

이 테스트는 화려하지 않지만 유용하다. 에이전트가 UI를 바꿔도 핵심 경로가 살아 있는지 바로 드러난다. 결국 좋은 에이전트 운영은 "잘 고쳤다"가 아니라 "다음에도 다시 검증할 수 있다"로 끝나야 한다.

## 결론: 에이전트의 진짜 자산은 결과물이 아니라 재현성이다

- Copilot app은 세션을 worktree로 분리해 병렬 작업을 안전하게 만든다.
- /chronicle는 그 세션을 나중에 다시 조회할 수 있게 만든다.
- Playwright MCP는 브라우저 상태를 구조적으로 확인하게 해 준다.
- Playwright 테스트는 그 확인 과정을 회귀 방지로 고정한다.

오늘 바로 해볼 일은 하나다. 다음 에이전트 작업 하나에만이라도 "기록"과 "브라우저 확인" 단계를 붙여 보자. 그 순간부터 에이전트는 단발성 자동화가 아니라, 다시 돌릴 수 있는 운영 절차가 된다.
