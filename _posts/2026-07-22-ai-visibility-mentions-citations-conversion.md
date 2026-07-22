---
title: "AI 가시성은 트래픽보다 언급·인용·전환으로 봐야 합니다"
date: "2026-07-22"
keywords: ["AI visibility", "AEO", "llms.txt", "publisher monetization", "answer engines"]
lang: "ko"
description: "AI 답변 엔진 시대에는 클릭보다 언급·인용·전환이 중요합니다. 최신 리서치 수치와 함께 측정·개선하는 방법을 정리합니다."
---

# AI 가시성은 트래픽보다 언급·인용·전환으로 봐야 합니다

예전 SEO는 꽤 단순했습니다. 검색결과에서 상위에 뜨면 클릭이 늘고, 클릭이 늘면 매출이 따라왔습니다. 그런데 AI 답변 엔진은 이 공식을 한 번에 바꿔버렸습니다. 사용자는 링크를 누르지 않고도 답을 얻고, 브랜드는 보이지만 유입은 남지 않는 경우가 많습니다.

그래서 지금 중요한 질문은 "몇 명이 들어왔나?"가 아닙니다. **"AI가 우리를 어떻게 읽고, 어떤 문장으로 소개하며, 실제로 전환까지 이어졌나?"** 입니다. 퍼블리셔, SaaS, 브랜드 모두 이제는 트래픽 리포트만 보지 말고 언급·인용·정확도·전환을 함께 봐야 합니다.

## 1) 왜 트래픽만 보면 안 되는가

최근 리서치들은 같은 방향을 가리킵니다. DataDome은 2026년 2분기 AI 에이전트 요청이 **177억 건**에 도달했고, 전분기 대비 **45% 증가**했다고 발표했습니다. 그런데 같은 보고서에서 ChatGPT는 AI referral traffic의 대부분을 차지하면서도, 크롤링 볼륨과 referral value가 완전히 같은 방향으로 움직이지 않는다고 지적했습니다. 즉, **더 많이 긁히는 것**과 **더 많이 보내는 것**은 다릅니다.

Ahrefs의 llms.txt 연구도 비슷합니다. 13만7천여 도메인을 분석했더니 llms.txt를 공개한 사이트는 **28%**였지만, 그중 **97%**는 2026년 5월에 **0 requests**를 기록했습니다. 파일을 올렸다는 사실만으로 AI가 읽어 주지는 않는다는 뜻입니다.

Webflow의 AEO 연구는 더 현실적입니다. 2,000개 웹사이트를 분석한 결과, median company는 답변 엔진 응답의 **16%**에만 등장했고, 링크로 인용된 비율은 **6%**에 불과했습니다. 기술 기본기에서도 **broken internal links 62%**, **missing SEO metadata 60%**가 드러났습니다. 결국 AI 가시성은 화려한 새 파일 하나가 아니라, 사이트 구조와 콘텐츠 운영의 총합입니다.

## 2) 실전에서는 무엇을 측정해야 하나

저는 AI 가시성을 아래 4개 축으로 나눠 보시길 권합니다.

- **Mention**: 답변 속에 우리 브랜드/도메인이 등장했는가
- **Citation**: 링크가 붙었는가
- **Accuracy**: 설명이 정확한가
- **Conversion**: 그 노출이 가입, 문의, 구매로 이어졌는가

이렇게 나누면 "AI가 우리를 봤다"와 "AI가 우리를 팔아줬다"를 분리할 수 있습니다. 둘은 전혀 같은 문제가 아닙니다.

### 간단한 수집 스크립트 예시

아래는 서버 로그나 리포트 CSV에서 AI 채널을 따로 집계하는 아주 단순한 예시입니다.

```python
import csv
from collections import Counter, defaultdict

AI_REFERRERS = {
    "chatgpt.com", "perplexity.ai", "claude.ai", "openai.com",
    "gemini.google.com", "copilot.microsoft.com"
}

summary = defaultdict(Counter)

with open("traffic.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        ref = row["referrer"].lower()
        page = row["landing_page"]
        channel = "ai" if any(domain in ref for domain in AI_REFERRERS) else "other"
        summary[page][channel] += 1

for page, counts in sorted(summary.items(), key=lambda x: x[1]["ai"], reverse=True):
    print(page, counts["ai"], counts["other"])
```

이 코드는 완벽하지 않지만 출발점으로 충분합니다. 다음 단계는 여기에 **mention/citation** 값을 붙이는 일입니다. AI 답변을 수동 샘플링하든, 외부 모니터링 도구를 쓰든, 결국 대시보드에는 같은 질문이 들어가야 합니다. "어떤 페이지가 인용되었고, 어떤 페이지가 무시되었는가?"

### 운영 팁

1. **llms.txt는 보조 수단으로만 보세요.** Ahrefs 데이터처럼 읽히지 않는 파일이 많습니다.
2. **기술 기본기를 먼저 고치세요.** 내부 링크, 메타데이터, 최신성은 여전히 강력합니다.
3. **대표 페이지를 정하세요.** 모든 글을 AI용으로 만들기보다, 인용될 만한 기준 페이지를 따로 두는 편이 좋습니다.
4. **전환 이벤트를 연결하세요.** AI 노출이 가입 폼, 문의, 데모 요청과 연결되는지 꼭 추적해야 합니다.

## 3) 지금 시장이 말해주는 것

세 자료를 같이 보면 패턴이 선명합니다. DataDome은 **agentic traffic의 급증**, Ahrefs는 **llms.txt의 낮은 실제 읽힘**, Webflow는 **answer engine에서의 낮은 인용률**을 보여줍니다. 즉, 시장은 이미 "검색 유입 최적화" 단계를 지나 "AI가 읽고 요약하고 재배포하는 과정"으로 이동했습니다.

그리고 여기서 중요한 건 공포가 아니라 설계입니다. 모든 봇을 막는 것보다, 신뢰 가능한 에이전트를 식별하고, 중요한 페이지를 구조화하고, 인용될 만한 출처를 만들어 두는 편이 실무적입니다. DataDome이 말한 것처럼 agent trust 정책을 도입한 고객이 이미 절반을 넘었다는 점도 같은 방향을 가리킵니다.

## 4) 흔히 하는 실수

- llms.txt만 올리면 끝난다고 생각하기
- AI 언급과 검색 유입을 같은 지표로 묶기
- "노출은 늘었는데 전환이 없다"를 그냥 넘어가기
- 오래된 콘텐츠와 깨진 링크를 방치하기

## 결론

AI 가시성은 이제 트래픽 게임이 아닙니다.

- **언급**이 있어야 발견됩니다.
- **인용**이 있어야 신뢰됩니다.
- **정확도**가 있어야 브랜드가 망가지지 않습니다.
- **전환**이 있어야 매출이 됩니다.

오늘 바로 할 첫 단계는 간단합니다. 최근 30일 동안 우리 사이트에서 가장 중요한 20개 페이지를 뽑고, 각 페이지에 대해 **AI 언급 / AI 인용 / 전환**을 따로 기록해 보세요. 그 표가 생기는 순간, AI 검색 시대의 운영이 시작됩니다.
