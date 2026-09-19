---
title: "컴퓨터 유즈 에이전트 순위는 1.1%p 차이다 — OSWorld·WebArena 점수로 실무 모델을 고르는 4축 프레임워크"
date: "2026-09-19"
keywords: ["컴퓨터 유즈 에이전트", "OSWorld", "WebArena", "computer use agent", "CUA 벤치마크", "GUI 자동화"]
lang: "ko"
description: "OSWorld-Verified 상위 3개 모델의 격차는 1.1%p에 불과하다. 벤치마크 점수 대신 GUI·웹·CLI·하이브리드 4개 축으로 나눠 컴퓨터 유즈 에이전트를 선택하는 실전 가이드."
---

# 컴퓨터 유즈 에이전트 순위는 1.1%p 차이다 — OSWorld·WebArena 점수로 실무 모델을 고르는 4축 프레임워크

2026년 9월 18일 기준 OSWorld-Verified 리더보드에서 1위 Qwen3.8 Max(86.1%)와 3위 Claude Mythos 5(85%)의 격차는 1.1 percentage point다. 2년 전만 해도 최고 에이전트가 15%도 못 넘기던 벤치마크가 포화 상태에 접어들었다는 신호다. 문제는 여기서 시작한다. 헤드라임 스코어가 모두 비슷해지면 순위표는 더 이상 선택 기준이 아니게 되고, 실무 트래픽을 돌려본 팀은 자기 워크로드가 리더보드 1위 모델에서 가장 잘 안 풀리는 경우를 발견하게 된다.

이 글은 "컴퓨터 유즈(computer use)"라는 하나의 라벨 아래 묶여 있는 능력이 실제로는 4개의 서로 다른 축이라는 점을 정리하고, 축별로 어떤 벤치마크를 봐야 하고 어디서 무너지는지를 최신 리더보드 수치와 함께 분해한다.

## "컴퓨터 유즈"는 하나의 능력이 아니다

흔히 CUA(Computer-Use Agent)로 통칭되는 영역을 작업 인터페이스 기준으로 나누면 다음 4개 축이 된다.

- **GUI 내비게이션** — 데스크톱 애플리케이션(오피스, 레거시 ERP, 디자인 도구) 안에서 클릭·스크롤·폼 입력을 수행
- **웹 자동화** — 브라우저에서 DOM 접근, 자바스크립트 헤비 페이지, 인증 세션이 필요한 작업
- **CLI 실행** — 셸 명령 생성, 스크립트 실행, 파일 I/O 중심의 터미널 워크플로우
- **하이브리드 오케스트레이션** — "메일에서 인보이스를 꺼내 스프레드시트를 갱신하고 Slack에 요약을 올려라"처럼 둘 이상의 축을 잇는 멀티앱 파이프라인

각 축은 자기만의 벤치마크, 자기만의 선두 모델, 그리고 결정적으로 자기만의 실패 양상을 갖는다. 이걸 하나의 통합 능력으로 취급하는 것이 기업 도입 검토에서 가장 비싼 실수다.

## 벤치마크 지형: 어느 숫자를 믿을 것인가

현재 측정의 중심은 세 개다.

**OSWorld / OSWorld-Verified.** OSWorld는 NeurIPS 2024에서 xlang-ai가 발표한 평가로, 샌드박스된 Ubuntu 환경에서 LibreOffice·VS Code·Chrome·파일 매니저 등 실제 데스크톱 앱 369개 태스크를 수행시킨다. OSWorld-Verified는 2025년 7월에 태스크 기술과 평가기를 수리한 재출시 버전이다. 2026년 9월 기준 상위권은 다음과 같다(BenchLM 집계, 제공사 자가보고 포함).

| 모델 | 개발사 | OSWorld-Verified |
|------|--------|------------------|
| Qwen3.8 Max | Alibaba | 86.1% |
| Claude Fable 5 | Anthropic | 85% |
| Claude Mythos 5 | Anthropic | 85% |
| Qwen3.8-27B | Alibaba | 84.3% |
| Claude Opus 4.8 | Anthropic | 83.4% |
| GPT-5.5 | OpenAI | 78.7% |

읽기 전에 알아둘 것: BenchLM의 검증 등록부 기준으로 32개 행 중 30개가 제공사 자가보고다. 독립 실행은 2개뿐이다.

**WebArena / WebArena-Verified.** WebArena는 셀프호스팅한 실제 동작하는 웹사이트 복제본 안에서 812개 장기 태스크를 수행시키고 최종 상태를 검사한다. Verified판은 비결정적 판정을 결정적 검사로 교체한 감사판이다. 2026년 9월 기준 공개 스냅샷은 행이 3개뿐인데, Muse Spark 1.1(Meta) 69%, Qwen3.8 Max 66.8%, Qwen3.8-27B 64.8%다. 행이 적어 시장 전체의 선두를 확정할 수 없다는 한계를 BenchLM 스스로 명시하고 있다.

**긴 태스크(50-step) 평가.** 여기서 진짜 격차가 드러난다. OSWorld-Verified에서 70~80%대를 찍는 에이전트들도 50스텝 멀티앱 태스크에서는 크게 무너진다. 오픈소스 진영의 참고점으로, Simular의 Agent S2(arXiv 2504.00906)는 OSWorld 15-step과 50-step 평가에서 당시 최강 baseline 대비 각각 18.9%, 32.7%의 상대적 개선을 기록하며 구성적 일반리스트-스페셜리스트 프레임워크의 유효성을 보였다. 짧은 태스크 점수가 비슷할수록 긴 태스크 성공률이 실질적인 변별력이 된다.

## 축별 해부: 어디서 누가 강하고 어디서 모두가 무너지나

**GUI 축.** 스크린샷+마우스/키보드 인터페이스로 OS에 구애받지 않는 접근이 유리하다. 실제로 OSWorld-Verified 상위권은 Claude 계열과 Qwen 계열이 양분한다. 크로스플랫폼 데스크톱 자동화가 요구사항이라면 2026년 현재 이 둘이 경험적 답이다.

**웹 축.** 브라우저 네이티브 작업은 OSWorld 점수가 아니라 WebArena 계열로 봐야 한다. 보도 기준으로 OpenAI의 초기 Operator는 WebArena 58.1% 대비 OSWorld 38.1%로, 같은 모델에서 20 point 가까운 축 간 격차를 보여준다. 웹 업무(CRM 입력, SaaS 파이프라인)에 OSWorld 순위로 모델을 고르면 반대로 선택하게 된다.

**CLI 축.** 측정 인프라가 가장 얇고 기업 가치는 가장 큰 축이다. Terminal-Bench 계열 평가가 있지만 OSWorld만한 트랙션은 아직이다. 핵심은 SWE-bench 같은 PR 형식 평가가 터미널 능력의 나쁜 대리변수라는 점이다. DevOps·플랫폼 팀이라면 리더보드 대신 자기 저장소 기반 스모크 테스트가 답이다.

**하이브리드 축.** 모든 에이전트가 가장 약한 곳이다. GUI+웹+CLI를 잇는 조율 작업은 멀티앱 평가에서 성공률이 크게 떨어지며, 이는 모델 능력보다 아키텍처 문제, 즉 컨텍스트 전환에 걸친 상태 추적과 지속 메모리 부재 탓이 크다.

## 실전: 워크로드를 축으로 분류하는 셀프 평가

도입 검토 시 가장 먼저 할 일은 후보 태스크 20~30개를 축으로 태깅하는 것이다. 프롬프트로 자동 분류하면 이렇다.

```python
AXES = {
    "gui": "데스크톱 앱 창, 메뉴, 다이얼로그 조작 (LibreOffice, ERP, 디자인 도구)",
    "web": "브라우저 내 사이트 조작 (폼, 검색, 예약, 로그인 세션)",
    "cli": "터미널 명령, 스크립트, 파일 시스템 작업",
    "hybrid": "둘 이상의 인터페이스를 잇는 멀티앱 파이프라인",
}

def classify_task(task_desc: str) -> str:
    keywords = {
        "gui": ["앱", "창", "메뉴", "스프레드시트", "ERP"],
        "web": ["사이트", "브라우저", "로그인", "폼", "장바구니"],
        "cli": ["셸", "스크립트", "배포", "로그", "git"],
    }
    scores = {ax: sum(k in task_desc for k in kws)
              for ax, kws in keywords.items()}
    hits = [ax for ax, s in scores.items() if s > 0]
    return "hybrid" if len(hits) > 1 else (hits[0] if hits else "gui")
```

분류 결과의 주 축이 `gui`나 `web`이면 해당 축 리더보드 상위 후보 3개, `cli`나 `hybrid`이면 리더보드가 아니라 직접 평가로 간다. 후보마다 실제 태스크 10개씩을 돌리는 미니 평가 해니스는 이렇게 구성한다.

```python
import json, statistics

def smoke_eval(agent, tasks, axis, passes=3):
    results = []
    for t in tasks:  # axis에 속한 실제 업무 태스크 10개
        ok = sum(agent.run(t.goal, t.setup) for _ in range(passes))
        results.append({
            "task": t.id,
            "axis": axis,
            "success_rate": ok / passes,
            "steps": agent.last_step_count,
        })
    rate = statistics.mean(r["success_rate"] for r in results)
    return {"axis": axis, "mean_success": rate, "detail": results}
```

`passes=3`으로 반복하는 이유는 CUA 성공이 확률적이기 때문이다. 1회 성공률이 아니라 통계로 비교해야 하고, 평균 스텝 수를 함께 재면 "성공은 하지만 200스텝을 도는" 에이전트를 걸러낼 수 있다.

## 주의사항 세 가지

1. **자가보고 편향.** OSWorld-Verified 행의 대부분이 제공사 자가보고다. 서로 다른 에이전트 스캐폴드·스텝 예산·평가 정책으로 낸 숫자를 동률로 비교하면 안 된다. WebArena-Verified의 BenchLM 편집 리뷰도 환경 리비전·스캐폴드·도구 호출 예산·pass@k 정책이 일치할 때만 행 간 비교가 성립한다고 못박는다.
2. **고정 환경의 한계.** WebArena는 셀프호스팅 복제본에서 돈다. 실제 오픈 웹의 레이아웃 드리프트, 안티봇, 프로덕션 인증 흐름은 측정 범위 밖이다. 벤치마크 통과가 실웹 안정성을 보장하지 않는다.
3. **시스템 점수, 모델 점수.** CUA 벤치마크 점수는 모델이 아니라 모델+스캐폴드+브라우저 인터페이스를 합친 시스템의 점수다. 같은 모델도 스캐폴드를 바꾸면 점수가 달라진다.

## 결론

- OSWorld-Verified 상위 3개 격차는 1.1%p — 순위표는 포화됐고, 변별력은 축 매칭과 긴 태스크로 이동했다
- GUI는 OSWorld 계열(Claude·Qwen 선두), 웹은 WebArena 계열(Verified 선두 Muse Spark 1.1 69%), CLI와 하이브리드는 아직 리더보드가 아니라 직접 평가의 영역이다
- 같은 에이전트의 축 간 격차는 20 point에 달할 수 있다(Operator: WebArena 58.1% vs OSWorld 38.1%, 보도 기준)
- 벤치마크 점수의 대부분은 제공사 자가보고이며, 시스템 전체의 점수지 모델의 점수가 아니다

당장 할 첫 단계는 간단하다. 자동화하려는 업무 20개를 위의 분류기로 축 태깅하고, 지배적 축 하나를 정한 뒤 그 축의 후보 3개에 대해 실제 태스크 10개짜리 스모크 평가를 3회 반복해서 돌려보는 것. 리더보드 1위를 찾는 시간보다 이 30분이 더 나은 선택을 보장한다.
