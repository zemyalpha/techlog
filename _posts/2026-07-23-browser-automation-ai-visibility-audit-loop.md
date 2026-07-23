---
title: "브라우저 자동화로 AI 가시성 점검 루프를 만드는 법"
date: "2026-07-23"
keywords: ["AI visibility", "browser automation", "AEO", "Playwright", "Search Console"]
lang: "ko"
description: "AI 답변 엔진 시대에는 llms.txt만으로 부족합니다. 브라우저 자동화와 로그·전환 데이터를 묶어 AI 가시성을 반복 점검하는 방법을 정리합니다."
---

# 브라우저 자동화로 AI 가시성 점검 루프를 만드는 법

AI 검색 시대에는 "우리 브랜드가 검색되나"보다 "AI가 우리를 어떻게 읽고, 어떻게 인용하고, 실제로 전환까지 만들었나"가 더 중요합니다. 지난 며칠 사이 공개된 리서치만 봐도 방향은 분명합니다. DataDome은 2026년 2분기 AI agent 요청이 177억 건에 도달했고 전분기 대비 45% 늘었다고 발표했습니다. Ahrefs는 137,210개 도메인을 분석해 llms.txt 파일의 97%가 2026년 5월에 한 번도 읽히지 않았다고 보고했습니다. Webflow는 2,000개 웹사이트를 조사한 결과, median company가 답변 엔진 응답의 16%에만 등장하고 링크 인용은 6%에 그친다고 밝혔습니다.

이 숫자들이 말하는 바는 간단합니다. 파일 하나 올리고 끝낼 수 있는 문제가 아니라는 뜻입니다. 브라우저 자동화, 로그 분석, 전환 추적이 함께 묶인 점검 루프가 필요합니다. Browserless도 2026년 "AI & Browser Automation" 현황을 다루며 이 영역이 실험 단계에서 운영 단계로 이동하고 있음을 보여줍니다.

## 1) 먼저 무엇을 측정할지 정하세요

AI 가시성은 아래 4개 축으로 쪼개면 실무적으로 다루기 쉽습니다.

- **Mention**: 답변 속에 브랜드나 제품명이 나오는가
- **Citation**: 링크가 붙는가
- **Accuracy**: 설명이 정확한가
- **Conversion**: 그 노출이 가입, 문의, 구매로 이어지는가

이렇게 나누면 "보였다"와 "팔렸다"를 분리할 수 있습니다. 답변 엔진에서 16% 등장, 6% 인용이라는 Webflow 수치는 특히 중요합니다. 노출과 전환은 같은 축이 아니기 때문입니다.

## 2) 브라우저 자동화는 수집기일 뿐, 답은 아닙니다

브라우저 자동화의 역할은 AI 답변을 매일 같은 조건으로 캡처하는 데 있습니다. 중요한 것은 완벽한 봇이 아니라 **반복 가능한 샘플링**입니다. 예를 들어 주요 키워드 20개에 대해 매주 같은 질문을 던지고, 결과 텍스트와 스크린샷을 저장하면 됩니다.

```python
# collect_visibility.py
from pathlib import Path
import json
from playwright.sync_api import sync_playwright

prompts = json.loads(Path("prompts.json").read_text(encoding="utf-8"))
Path("snapshots").mkdir(exist_ok=True)
Path("data").mkdir(exist_ok=True)

rows = []
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page(viewport={"width": 1440, "height": 1800})

    for item in prompts:
        page.goto(item["url"], wait_until="networkidle", timeout=45000)
        body_text = page.locator("body").inner_text()
        rows.append({
            "id": item["id"],
            "url": item["url"],
            "title": page.title(),
            "text": body_text[:12000],
        })
        page.screenshot(path=f"snapshots/{item['id']}.png", full_page=True)

    browser.close()

Path("data/observations.jsonl").write_text(
    "\n".join(json.dumps(row, ensure_ascii=False) for row in rows),
    encoding="utf-8"
)
```

이 스크립트는 AI 서비스를 직접 "해석"하지 않습니다. 대신 보이는 결과를 저장합니다. 점검 루프의 핵심은 과장된 추론이 아니라, 같은 프롬프트를 같은 방식으로 반복해 차이를 보는 데 있습니다.

## 3) 전환과 연결해야 진짜 지표가 됩니다

AI 가시성은 결국 매출과 연결되어야 의미가 있습니다. 아래처럼 간단한 점수표를 만들 수 있습니다.

```python
# score_visibility.py
import json
from collections import Counter, defaultdict
from pathlib import Path

observations = [json.loads(line) for line in Path("data/observations.jsonl").read_text(encoding="utf-8").splitlines()]
metrics = defaultdict(Counter)

for row in observations:
    text = row["text"].lower()
    metrics[row["id"]]["mention"] += int("mybrand" in text)
    metrics[row["id"]]["citation"] += int("http" in text)
    metrics[row["id"]]["accuracy"] += int("pricing" in text or "docs" in text)

for k, v in sorted(metrics.items()):
    score = v["mention"] * 2 + v["citation"] * 3 + v["accuracy"]
    print(k, dict(v), "score=", score)
```

이 점수는 완벽한 벤치마크가 아니라 운영용 경보판입니다. 점수가 떨어진 페이지는 보통 세 가지 중 하나입니다. 내부 링크가 끊겼거나, 메타데이터가 비어 있거나, 문서의 핵심 문장이 오래됐습니다. Webflow가 지적한 broken internal links 62%, missing SEO metadata 60%는 바로 이 기본기의 중요성을 보여줍니다.

## 4) 우선순위는 llms.txt가 아닙니다

Ahrefs의 결과처럼 llms.txt가 있다고 해서 자동으로 읽히는 것은 아닙니다. 그래서 순서는 분명해야 합니다.

1. 대표 페이지를 정리합니다.
2. 내부 링크와 메타데이터를 고칩니다.
3. AI 답변 캡처 루프를 만듭니다.
4. 전환 이벤트를 붙입니다.
5. llms.txt는 마지막에 보조 수단으로 둡니다.

이 순서를 거꾸로 하면 파일은 생기는데 가시성은 안 오릅니다. 반대로 기본기를 먼저 고치면, AI가 요약할 문장도 더 선명해지고 링크를 붙일 이유도 늘어납니다.

## 결론

오늘부터 할 일은 복잡하지 않습니다.

- 주요 키워드 20개를 정합니다.
- 각 키워드에 대한 AI 답변 샘플을 주 1회 캡처합니다.
- Mention, Citation, Accuracy, Conversion을 같은 표에 넣습니다.
- 깨진 링크와 빈 메타데이터부터 고칩니다.
- llms.txt는 그 다음입니다.

AI 가시성의 핵심은 더 많은 트래픽을 쫓는 것이 아니라, 더 자주 인용되고 더 정확하게 설명되며 더 잘 전환되는 구조를 만드는 데 있습니다. 브라우저 자동화는 그 구조를 매주 확인하는 가장 현실적인 도구입니다.
