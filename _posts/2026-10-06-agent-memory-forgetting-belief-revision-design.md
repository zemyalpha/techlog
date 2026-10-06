---
title: "에이전트는 어떻게 잊는가 — 2026년 장기기억 연구가 알려주는 설계 원칙 4가지"
date: "2026-10-06"
keywords: ["LLM 에이전트 장기기억", "agent memory", "망각 곡선", "메모리 통합", "LoCoMo", "LongMemEval"]
lang: "ko"
description: "벡터 DB에 대화를 쌓는 것은 기억이 아니다. 2026년 arXiv의 에이전트 메모리 논문 5편에서 온라인/오프라인 분리, 믿음 개정, 망각, 재통합이라는 공통 원칙을 추출하고 코드로 구현한다."
---

# 에이전트는 어떻게 잊는가 — 2026년 장기기억 연구가 알려주는 설계 원칙 4가지

개인 비서 에이전트를 한 달쯤 운영해본 사람이라면 누구나 같은 경험을 한다. 첫 주에는 감동적이다. 에이전트가 내 취향을 기억하고, 지난 대화를 인용하고, 반복 설명을 생략한다. 그런데 한 달쯤 지나면 이상해진다. 예전에 "라이트 모드를 싫어한다"고 했다가 나중에 "다크 테마는 눈이 아프다"고 바꾼 말을 두 개 다 기억해서 대답이 랜덤하게 바뀌고, 중요하지 않은 옛 대화가 여전히 검색 결과 상위에 떠서 최근 맥락을 덮어버린다.

문제는 저장이 아니라 **정리**다. 인간의 기억이 강력한 이유는 잘 보관해서가 아니라 잘 잊고 잘 다시 쓰기 때문이다. 2026년 arXiv에 올라온 에이전트 메모리 논문들을 읽으면 서로 다른 팀들이 전혀 다른 구조를 제안하면서도 놀랍게 같은 결론에 도착하는 것을 볼 수 있다. 이 글에서는 그 논문들에서 공통으로 나타나는 설계 원칙 4가지를 추출하고, 각각을 직접 구현해볼 수 있는 코드로 풀어본다.

## 원칙 1: 쓰기는 가볍게, 정리는 따로 — 온라인/오프라인 분리

LightMem(arXiv 2604.07798)는 이 문제를 가장 명확하게 정리한다. 메모리 조작을 대형 LLM 호출로 처리하면 정확하지만 턴마다 지연이 쌓이고, 순수 검색으로 처리하면 빠르지만 정확도가 흔들린다. 이 둘을 한 파이프라인에서 함께 하지 말고 **시간을 분리하라**는 것이다.

구체적으로 LightMem은 온라인 경로에서 소형 언어모델(SLM) 세 개만 돌린다. 쿼리를 의도 기반 가상 질의로 바꾸는 Controller, 후보를 검증·재랭킹하는 Selector, 요약을 쓰는 Writer. 무거운 추상화와 통합은 밤이나 유휴 시간에 오프라인으로 돌려서, 대화 중 지연을 상수로 유지한다. 논문의 보고 수치는 검색 지연 중앙값 83ms, 엔드투엔드 581ms이며 LoCoMo에서 A-MEM 대비 평균 F1 약 +2.5다.

이건 그냥 아키텍처 취향이 아니라 비용 구조의 문제다. 대화 턴마다 거대 모델을 불러 메모리를 정리하면, 한 달 치 대화에서 메모리 관리 비용이 응답 생성 비용을 추월한다. 사람이 잠자는 동안 기억을 통합하는 것처럼, 에이전트도 정리 작업을 유휴 시간으로 미룰 수 있어야 한다.

파이썬 의사코드로 구조를 잡으면 이렇다:

```python
# 온라인: 대화 턴 안에서 실행. SLM만 사용, 수백 ms 예산
def on_turn(user_msg, memory):
    queries = slm_controller.rewrite(user_msg)      # 의도 기반 가상 질의 생성
    candidates = memory.coarse_retrieve(queries)     # 벡터 코스 검색
    ranked = slm_selector.rerank(candidates)         # 의미 일관성 재랭킹
    reply = llm.generate(user_msg, context=ranked)
    slm_writer.append_summary(memory.mtm, user_msg, reply)
    return reply

# 오프라인: 크론잡/유휴 시간에 실행. 대형 모델 사용 허용
def nightly_consolidation(memory):
    batch = memory.mtm.pending_items()               # 새 항목 + 저빈도 압박 항목만
    knowledge = big_llm.abstract(batch)              # 개인 식별 정보 제거한 지식으로 증류
    memory.ltm.incremental_merge(knowledge)          # 전체 재구축 금지, 증분 병합만
```

핵심 디테일은 오프라인 통합이 `rebuild()`가 아니라 `incremental_merge()`라는 점이다. 매일 밤 전체 기억을 다시 요약하면 오류가 누적되므로, LightMem은 새로 쓰였거나 검색으로 재활성화된 항목만 증분 처리한다.

## 원칙 2: 사실을 덮어쓰지 말고 믿음의 연쇄로 남겨라

"사용자가 커피를 좋아함" → "디카페으로 바꿈" → "요즘은 차만 마심". 대부분의 메모리 시스템은 이걸 잘못 처리한다. 최신 것만 남기면 시간 추론("예전엔 뭘 마셨지?")에 실패하고, 전부 남기면 충돌한다.

DCPM(arXiv 2606.09483)이 제안하는 구조는 **supersedes 체인**이다. 새 사실이 들어오면 기존 사실을 지우지 않고 이중 연결 포인터로 연결한다. 읽을 때는 벡터 조회 후 포인터만 따라가면 되므로 LLM 호출이 필요 없고, 사용자가 어떻게 변해왔는지(믿음 궤적) 자체가 질의 가능한 데이터가 된다. 논문은 이런 시간 추론이 필요한 PersonaMem-v2 벤치마크에서 비동기 추상화 엔진 활성화 시 최대 +5.20 향상을 보고했다.

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Fact:
    content: str
    created_at: datetime
    superseded_by: str | None = None   # 나를 대체한 사실의 ID
    supersedes: str | None = None      # 내가 대체한 사실의 ID
    id: str = field(default_factory=lambda: f"fact_{datetime.now().timestamp()}")

def revise(store, old_id: str, new_content: str) -> Fact:
    """기존 사실을 삭제하지 않고 체인으로 연결한다."""
    old = store[old_id]
    new = Fact(content=new_content, created_at=datetime.now(), supersedes=old_id)
    old.superseded_by = new.id        # 이중 연결 — 뒤로도 앞으로도 탐색 가능
    store[new.id] = new
    return new

def belief_trajectory(store, fact_id: str) -> list[str]:
    """한 주제에 대한 사용자의 믿음 변화 전체를 시간순으로 반환."""
    node = store[fact_id]
    while node.supersedes:            # 체인의 시작점까지 거슬러 올라감
        node = store[node.supersedes]
    chain = []
    while node:
        chain.append(f"{node.created_at:%Y-%m-%d}: {node.content}")
        node = store.get(node.superseded_by) if node.superseded_by else None
    return chain
```

이 패턴의 실무적 장점은 롤백이다. 에이전트가 잘못된 결론으로 사실을 "개정"했을 때, 체인이 있으면 이전 상태로 되돌릴 수 있다. 덮어썼다면 원본은 영원히 사라진다.

## 원칙 3: 잊음은 버그가 아니라 기능이다 — 엔트로피 감쇠

ZenBrain(arXiv 2604.23878)는 인지신경과학의 통찰 15가지(에빙하우스 망각 곡선, FSRS 간격 반복, 수면 중 재생 등)를 통합한 7층 구조를 제안하며, 흥미로운 실증 결과를 하나 남긴다. 정상 상황에서는 메커니즘들이 서로를 대신해주는 "협력적 은폐(cooperative masking)" 효과 때문에 개별 메커니즘을 빼도 성능이 잘 유지되는데, 스트레스 조건(감쇠율 0.25/일, 60일 운용)에서는 15개 중 9개가 개별적으로 필수가 된다는 것이다. 즉 **잊음 메커니즘은 단기 테스트에서는 효과가 안 보여도 장기 운용에서 시스템을 지탱하는 구조**다.

망각 구현 자체는 놀랍도록 단순하다. 항목마다 강도를 두고 시간에 따라 지수 감쇠시키면 된다:

```python
import math

def decay(strength: float, days: float, rate: float = 0.25) -> float:
    """에빙하우스 스타일 지수 감쇠. rate=0.25면 4일에 절반으로 줄어든다."""
    return strength * math.exp(-rate * days)

def maintenance_sweep(memory, threshold: float = 0.05, now=None):
    """유휴 시간에 실행. 임계치 밑으로 떨어진 항목은 보관소로 이동(즉시 삭제 아님)."""
    now = now or time_now()
    for item in memory.all():
        item.strength = decay(item.strength, days=(now - item.last_access).days)
        if item.strength < threshold:
            memory.archive(item)      # 삭제 대신 아카이브 — 필요 시 복원 가능
        elif item.recently_used:
            item.strength += 0.1      # 접근 시 강화 — 자주 쓰이는 기억은 오래감
```

주의할 점 하나: ZenBrain식 구조에서 절대 감쇠하지 않는 계층(core memory — 사용자 신원, 극히 안정적인 선호)을 따로 두는 것이 핵심이다. 전체를 감쇠시키면 중요한 것과 사소한 것이 함께 죽는다.

한편 RPMem(arXiv 2609.23466)은 텍스트 저장을 아예 벗어나는 길을 보여준다. 세션을 잠재 메모리로 컴파일해 LoRA 파라미터로 매핑하는 방식으로, Qwen3-8B 기준 PERMA 벤치마크 85.52%, 텍스트 기반 최강 baseline 대비 +12.98pp를 보고했다. 다만 이 접근은 학습 파이프라인이 필요해 아직 연구 단계에 가깝고, 운영 환경에서는 텍스트 기반 구조가 당분간 실용적 선택이다.

## 원칙 4: 검색은 끝이 아니라 되먹임의 시작이다

가장 흥미로운 방향은 REALM(arXiv 2609.16053)이 밀고 있는 **재통합(reconsolidation)**이다. 신경과학에서 기억은 꺼낼 때마다 다시 쓰인다 — 검색이 곧 수정 기회라는 것. REALM은 태스크가 끝날 때마다 방금 검색에 쓰인 서브그래프의 연결 구조를 업데이트해서, 자주 함께 쓰이는 기억 경로는 강화되고 비효과적 경로는 약해지게 만든다. LoCoMo 평균 정확도 75.97%, LongMemEval 65.11%로 최강 baseline 대비 각각 +7.17, +1.31점이다.

운영 관점에서 이 아이디어는 저렴하게 훔쳐올 수 있다. 전체 인지 그래프가 아니라 **검색 로그에 통계만 얹는 것**으로 시작하라:

```python
from collections import defaultdict

class UsageAwareMemory:
    def __init__(self, store):
        self.store = store
        self.co_access = defaultdict(int)   # (a, b) 쌍이 함께 검색에 쓰인 횟수

    def retrieve(self, query, top_k=5):
        hits = self.store.search(query, top_k=top_k)
        for i, a in enumerate(hits):
            for b in hits[i+1:]:
                self.co_access[(a.id, b.id)] += 1
        return hits

    def boost(self, results, base_score_key="sim"):
        """함께 자주 쓰인 기억 묶음에 보너스 — 검증 전 가설이지만 구현은 10줄."""
        for r in results:
            r.score = r.score + 0.05 * max(
                self.co_access[(r.id, o.id)] for o in results if o.id != r.id
            )
        return sorted(results, key=lambda r: -r.score)
```

REALM의 성능 향상 원인을 논문은 "기억 단위들이 점점 더 응집된 국소 구조로 재조직된다"고 설명한다. 위의 통계적 근사는 그 이점의 일부만 줄 수 있지만, 그래프 재구성 인프라 없이 오늘 바로 적용 가능하다.

## 결론: 저장 용량이 아니라 정리 정책의 싸움

5편의 논문에서 반복적으로 나타난 공통 원칙을 정리하면:

- **온라인/오프라인 분리** — 대화 중에는 가볍게 쓰고, 정리는 유휴 시간에 증분으로(LightMem: 검색 지연 83ms)
- **믿음 개정은 체인으로** — 사실을 덮어쓰지 말고 supersedes 포인터로 변화 궤적을 보존(DCPM: PersonaMem-v2 +5.20)
- **적극적 망각** — 감쇠는 단기 테스트에 안 보여도 장기 운용의 필수 조건(ZenBrain), 단 core memory는 감쇠 제외
- **검색 후 재통합** — 검색 로그 자체를 기억 구조 개선의 신호로 사용(REALM: LoCoMo 75.97%)

당장 시도해볼 첫 단계를 하나만 고르라면, 2번이다. 벡터 스토어에 저장하는 레코드에 `superseded_by` 필드 하나를 추가하고, 같은 주제의 새 사실이 들어올 때 삭제 대신 체인으로 연결하는 것. 하루 치 작업이고, 기존 RAG 파이프라인을 바꿀 필요도 없다. 그리고 다음 날 에이전트에게 물어보라 — "내가 커피에 대해 뭐라고 했지?" 대답이 달라져 있을 것이다.

*(본문의 벤치마크 수치는 모두 각 논문의 arXiv abstract 및 본문에 자가 보고된 값이며, 독립 재현 검증은 아니라는 점을 유의하자. 특히 LoCoMo/LongMemEval 계열 벤치마크는 평가 프로토콜이 논문마다 달라 직접 비교에는 한계가 있다.)*
