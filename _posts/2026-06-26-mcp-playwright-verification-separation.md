---
title: "MCP는 연결, Playwright는 검증: 코딩 에이전트의 프로덕션 분리 설계"
date: "2026-06-26"
keywords: ["코딩 에이전트", "MCP", "Playwright MCP", "브라우저 검증", "Google Cloud"]
lang: "ko"
description: "Google의 공식 MCP 지원과 Playwright MCP를 바탕으로, 연결·검증·승인을 분리해 코딩 에이전트를 프로덕션에 올리는 방법을 정리합니다."
---

# MCP는 연결, Playwright는 검증: 코딩 에이전트의 프로덕션 분리 설계

코딩 에이전트가 빨라질수록 오히려 더 중요한 질문이 하나 생깁니다. **무엇을 연결할 것인가**보다 **무엇을 증명할 것인가**입니다. LLM이 사내 서비스, 웹 브라우저, 문서 저장소를 전부 건드릴 수 있게 만들면 데모는 쉬워집니다. 하지만 프로덕션에서는 연결, 검증, 승인 세 단계를 분리하지 않으면 금방 무너집니다.

최근 흐름은 이 분리가 단순한 취향이 아니라는 점을 보여줍니다. Google Cloud는 2025년 12월 11일 공식적으로 Google 서비스용 MCP 지원을 발표했고, 문서에서는 원격 MCP 서버를 HTTP로 제공한다고 설명합니다. 한편 Playwright MCP 문서는 브라우저 자동화를 MCP로 제공하면서도, 코딩 에이전트라면 CLI+SKILLS가 더 토큰 효율적일 수 있다고 직접 적고 있습니다. 요약하면, **MCP는 연결에 강하고, 브라우저 검증은 별도 레이어로 다루는 편이 더 실전적**입니다.

## 1) MCP가 잘하는 일: 서비스 연결과 구조화된 입력

Google의 공식 MCP 지원이 중요한 이유는 “에이전트가 쓸 수 있는 커넥터”가 이제 실험이 아니라는 신호이기 때문입니다. 문서 검색, 캘린더, 내부 도구처럼 구조화된 데이터 소스는 MCP로 붙이기 좋습니다. 에이전트는 자연어로 묻고, MCP 서버는 정형화된 결과를 돌려줍니다.

이때 핵심은 범위를 넓히지 않는 것입니다. MCP는 에이전트에게 세상을 주는 도구가 아니라, **필요한 데이터만 꺼내오는 인터페이스**여야 합니다.

```json
{
  "mcpServers": {
    "google-services": {
      "url": "https://YOUR_GOOGLE_MCP_ENDPOINT"
    },
    "internal-docs": {
      "url": "https://YOUR_INTERNAL_MCP_ENDPOINT"
    }
  }
}
```

이런 구성이 좋습니다.
- 읽기 전용 데이터는 MCP로 가져온다
- 쓰기 작업은 별도 승인 단계로 넘긴다
- 에이전트가 직접 관리자 API를 호출하게 두지 않는다

## 2) Playwright가 잘하는 일: 화면이 아니라 증거를 남기는 검증

Playwright MCP는 브라우저 자동화를 제공하고, 구조화된 accessibility snapshot을 통해 웹 페이지와 상호작용하도록 돕습니다. 여기서 중요한 포인트는 “브라우저를 움직일 수 있다”가 아니라 **UI가 실제로 기대한 상태인지 확인할 수 있다**는 점입니다.

다만 Playwright 쪽 README는 한 걸음 더 나아갑니다. 코딩 에이전트라면 MCP 대신 CLI+SKILLS를 쓰는 편이 더 토큰 효율적일 수 있다고 밝힙니다. 저는 이 문장을 아주 중요하게 봅니다. 즉, 브라우저 검증이 필요하더라도, 항상 MCP가 정답은 아닙니다. 반복적인 회귀 테스트나 고빈도 확인은 CLI와 테스트 러너가 더 나을 수 있습니다.

```ts
import { test, expect } from '@playwright/test';

test('dashboard renders with real data', async ({ page }) => {
  await page.goto(process.env.APP_URL!);

  await expect(page.getByRole('heading', { name: /dashboard/i })).toBeVisible();
  await expect(page.getByText(/last updated/i)).toBeVisible();
});
```

이 테스트의 역할은 단순합니다. 모델이 “아마 됐을 것”이라고 말하는 대신, 브라우저에서 실제로 보이는 증거를 남깁니다.

## 3) 실전 구조: 연결, 검증, 승인을 한 줄로 묶지 마십시오

프로덕션에서 가장 좋은 패턴은 다음 3단계입니다.

1. **연결 계층**: MCP로 Google 서비스와 내부 시스템을 읽는다.
2. **검증 계층**: Playwright로 브라우저 상태와 사용자 경로를 확인한다.
3. **승인 계층**: 배포, 삭제, 결제 같은 되돌리기 어려운 작업은 사람이 승인한다.

```yaml
pipeline:
  fetch:
    source: mcp
    mode: read_only
  verify:
    tool: playwright
    evidence:
      - visible_heading
      - form_state
      - navigation_path
  publish:
    requires_human_approval: true
    audit_log: true
```

이렇게 나누면 좋은 점이 많습니다. 실패 원인이 명확해집니다. 연결 문제인지, UI 문제인지, 승인 문제인지가 분리되기 때문입니다. 또 에이전트가 잘못 동작했을 때 손상 범위도 작아집니다.

## 4) 흔히 하는 실수

- **MCP 하나로 다 해결하려는 것**: 연결과 검증을 섞으면 디버깅이 어려워집니다.
- **브라우저 검증을 스크린샷 한 장으로 끝내는 것**: 텍스트, 역할, 폼 상태를 함께 봐야 합니다.
- **쓰기 작업을 검증 없이 자동화하는 것**: 삭제·배포·결제는 항상 승인 로그가 필요합니다.
- **고빈도 검증에 무거운 도구를 쓰는 것**: Playwright README가 말하듯, 상황에 따라 CLI+SKILLS가 더 효율적일 수 있습니다.

## 결론: 에이전트의 경쟁력은 더 많은 도구가 아니라 더 좋은 분리다

- Google Cloud의 공식 MCP 지원은 서비스 연결이 표준화되고 있음을 보여줍니다.
- Playwright MCP는 브라우저 자동화와 검증을 MCP로 가져옵니다.
- 하지만 Playwright 자체도 코딩 에이전트에게는 CLI+SKILLS가 더 효율적일 수 있다고 말합니다.
- 그래서 프로덕션 설계의 핵심은 “모든 걸 하나로”가 아니라 “연결·검증·승인을 분리”하는 것입니다.
- 가장 먼저 할 일은 에이전트가 읽는 것, 증명하는 것, 바꾸는 것을 서로 다른 레이어로 나누는 것입니다.

오늘 바로 해볼 수 있는 첫 단계는 간단합니다. 여러분의 에이전트 워크플로우를 적어 보고, 각 단계 옆에 `MCP`, `Playwright`, `Human Approval` 중 하나를 붙여 보십시오. 그 순간부터 데모용 자동화가 아니라 운영 가능한 시스템 설계가 시작됩니다.

## 참고 자료

- Google Cloud Blog: [Announcing official MCP support for Google services](https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services)
- Google Cloud Docs: [Google Cloud MCP servers release notes](https://docs.cloud.google.com/mcp/release-notes)
- Playwright Docs: [Playwright MCP](https://playwright.dev/docs/getting-started-mcp)
- Microsoft GitHub: [microsoft/playwright-mcp README](https://github.com/microsoft/playwright-mcp)
