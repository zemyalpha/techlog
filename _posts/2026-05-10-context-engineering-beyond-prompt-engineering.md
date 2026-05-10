---
title: "프롬프트 엔지니어링은 끝났다 — 컨텍스트 엔지니어링이 AI 에이전트의 새로운 패러다임이다"
date: "2026-05-10"
keywords: ["컨텍스트 엔지니어링", "AI 에이전트", "프롬프트 엔지니어링", "LLM", "context management"]
lang: "ko"
description: "Anthropic이 제시한 컨텍스트 엔지니어링 개념을 중심으로, AI 에이전트의 컨텍스트 윈도우를 실전에서 관리하는 4가지 전략과 코드를 정리한다."
---

# 프롬프트 엔지니어링은 끝났다 — 컨텍스트 엔지니어링이 AI 에이전트의 새로운 패러다임이다

챗봇 하나 만들 때는 프롬프트만 잘 쓰면 됐다. 하지만 에이전트는 다르다. 에이전트는 수십 번의 추론 턴을 반복하고, 도구를 호출하고, 외부 데이터를 끌어오면서 컨텍스트가 끝없이 부풀어 오른다. 어느 순간 모델은 사용자가 3번째 메시지에서 한 지시를 잊어버리고, 비용은 턴당 $0.50씩 치솟고, 응답 품질은 급락한다.

Anthropic은 2025년 9월 "Effective context engineering for AI agents"라는 글에서 이 문제에 정면으로 접근했다. 핵심 메시지는 간단하다: **프롬프트 엔지니어링의 시대는 지났고, 컨텍스트 엔지니어링의 시대가 왔다.** 이 글에서는 그 의미를 풀고, 실전에서 즉시 써먹을 수 있는 구체적인 전략과 코드를 정리한다.

## 컨텍스트 엔지니어링이 프롬프트 엔지니어링과 다른 점

프롬프트 엔지니어링은 "어떤 단어와 문장을 쓰면 모델이 원하는 출력을 내는가"에 집중한다. 시스템 프롬프트를 다듬고, few-shot 예시를 넣고, temperature를 조정하는 것이 전부였다.

컨텍스트 엔지니어링은 한 차원 위의 질문을 던진다: **"모델이 추론할 때 컨텍스트 윈도우 안에 어떤 정보 구성을 넣을 것인가?"** 여기에는 시스템 프롬프트뿐 아니라 도구 정의, MCP 서버가 제공하는 외부 데이터, 이전 대화 이력, 검색 결과, 에이전트가 스스로 생성한 중간 결과물이 모두 포함된다.

결정적인 차이는 **반복**이다. 프롬프트 엔지니어링은 한 번 작성하면 끝나는 정적인 작업이지만, 컨텍스트 엔지니어링은 에이전트가 매 추론 턴마다 수행해야 하는 동적 최적화다. 에이전트 루프가 돌 때마다 새로운 정보가 쌓이고, 그중 무엇을 컨텍스트에 유지하고 무엇을 버릴지 결정해야 한다.

## 왜 컨텍스트 관리가 에이전트의 생사를 가르는가

Anthropic이 지적한 핵심 개념은 **컨텍스트 부패(context rot)**다. 컨텍스트 윈도우가 길어질수록 모델의 정보 회상 능력이 저하되는 현상이다. needle-in-a-haystack 벤치마크에서 확인된 이 특성은 모든 모델에서 공통으로 나타난다.

원인은 트랜스포머 아키텍처 자체에 있다. 트랜스포머는 모든 토큰 쌍 사이의 어텐션을 계산하므로 n개 토큰에 대해 n²의 관계를 처리해야 한다. 컨텍스트가 길어지면 이 관계망이 희박해지고, 모델은 "중간에 있는" 정보를 놓치기 쉬워진다. 위치 인코딩 보간 기법으로 긴 시퀀스를 처리할 수는 있지만, 토큰 위치 이해도에는 저하가 생긴다.

실전에서는 세 가지 실패 모드가 반복해서 나타난다:

1. **지시 망각** — 사용자가 "pytest만 써"라고 3번째 턴에서 말했지만, 50번째 턴에서 모델이 unittest를 import한다
2. **중복 응답** — 이미 20턴 전에 설명한 내용을 다시 설명한다. 모델이 사용자가 이미 아는 것을 잊었기 때문이다
3. **반복 루프** — 최근 컨텍스트가 같은 패턴을 반복 강화하면서 모델이 똑같은 요약이나 제안을 계속 출력한다

이건 환각이 아니다. 컨텍스트 관리 실패다. 토큰은 기술적으로 윈도우 안에 있지만, 모델의 어텐션이 닿지 않는 것이다.

## 실전 전략 4가지와 구현 코드

### 전략 1: 잘라내기 (Truncation)

가장 단순하고 예측 가능한 전략. 토큰 수가 임계치를 넘으면 가장 오래된 메시지부터 버린다.

```python
def truncate_history(messages: list[dict], max_tokens: int, count_tokens) -> list[dict]:
    system = [m for m in messages if m['role'] == 'system']
    rest = [m for m in messages if m['role'] != 'system']
    
    total = sum(count_tokens(m) for m in system)
    kept = []
    
    for msg in reversed(rest):
        total += count_tokens(msg)
        if total > max_tokens:
            break
        kept.append(msg)
    
    return system + list(reversed(kept))
```

시스템 프롬프트는 항상 보존하고, 나머지는 최신 메시지부터 거꾸로 세어서 예산 안에 들어오는 만큼만 유지한다. "이 버그 수정하고 테스트 돌려" 같은 순차적 태스크에 적합하다. 한 번 해결된 이전 단계의 대화는 더 이상 필요 없기 때문이다.

**한계**: 버려진 정보는 영영 사라진다. 이전 맥락이 계속 중요한 대화에서는 위험하다.

### 전략 2: 요약 (Summarization)

주기적으로 이전 대화의 절반을 요약된 문단으로 교체한다. 정보의 핵심은 유지하면서 토큰을 크게 줄인다.

```python
import anthropic

client = anthropic.Anthropic()

def summarize_history(messages: list[dict], keep_recent: int = 6) -> list[dict]:
    system = [m for m in messages if m['role'] == 'system']
    rest = [m for m in messages if m['role'] != 'system']
    
    if len(rest) <= keep_recent:
        return messages
    
    to_summarize = rest[:-keep_recent]
    recent = rest[-keep_recent:]
    
    # 대화를 텍스트로 변환
    conversation_text = "\n".join(
        f"{m['role']}: {m['content']}" for m in to_summarize
    )
    
    summary_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": f"다음 대화의 핵심 결정, 사용자 요구사항, 이미 해결된 문제를 "
                       f"간결하게 요약하세요. 기술적 세부사항을 보존하세요:\n\n{conversation_text}"
        }]
    )
    
    summary = summary_response.content[0].text
    
    return system + [{
        "role": "user",
        "content": f"[이전 대화 요약]\n{summary}"
    }, {
        "role": "assistant",
        "content": "이전 대화 내용을 숙지했습니다. 계속 진행하겠습니다."
    }] + recent
```

LLM을 한 번 더 호출하는 비용이 발생하지만, 180K 토큰의 원본을 2K 토큰의 요약으로 압축하면 이후 매 턴마다 절약되는 비용이 훨씬 크다.

### 전략 3: RAG 기반 선택적 검색

컨텍스트에 모든 것을 넣는 대신, 현재 질문에 관련 있는 정보만 검색해서 동적으로 주입한다.

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

class ContextRetriever:
    def __init__(self, embedding_model="text-embedding-3-small"):
        self.embedding_model = embedding_model
        self.memory_store = []  # (embedding, text, metadata)
    
    def store_message(self, text: str, metadata: dict):
        embedding = client.embeddings.create(
            input=text, model=self.embedding_model
        ).data[0].embedding
        self.memory_store.append((np.array(embedding), text, metadata))
    
    def retrieve_relevant(self, query: str, top_k: int = 5) -> list[str]:
        query_emb = np.array(
            client.embeddings.create(
                input=query, model=self.embedding_model
            ).data[0].embedding
        )
        
        scores = []
        for emb, text, meta in self.memory_store:
            similarity = np.dot(query_emb, emb) / (
                np.linalg.norm(query_emb) * np.linalg.norm(emb)
            )
            scores.append((similarity, text))
        
        scores.sort(reverse=True)
        return [text for _, text in scores[:top_k]]
```

이 패턴은 장기 세션에서 빛을 발한다. 사용자가 "아까 말한 그 데이터베이스 마이그레이션 이슈 어떻게 됐어?"라고 물으면, 임베딩 유사도로 해당 대화를 찾아 현재 컨텍스트에 삽입할 수 있다.

### 전략 4: 구조화된 컨텍스트 아키텍처

Anthropic이 권장하는 방식. 컨텍스트를 명확한 섹션으로 나누고, 각 섹션에 XML 태그나 마크다운 헤더를 사용해 모델이 구조를 인식하게 한다.

```python
def build_structured_context(
    system_prompt: str,
    tools: list[dict],
    recent_messages: list[dict],
    retrieved_context: list[str],
    agent_scratchpad: str
) -> list[dict]:
    enriched_system = f"""{system_prompt}

<available_context>
<retrieved_documents>
{chr(10).join(f'<doc>{c}</doc>' for c in retrieved_context)}
</retrieved_documents>

<agent_working_notes>
{agent_scratchpad}
</agent_working_notes>
</available_context>"""
    
    return [{"role": "system", "content": enriched_system}] + recent_messages
```

핵심은 **최소 정보 원칙**이다. 원하는 동작을 이끌어내는 데 필요한 최소한의 토큰 세트만 구성하는 것. Anthropic은 "최소가 반드시 짧다는 뜻은 아니다"라고 강조한다. 에이전트가 제대로 동작하려면 충분한 정보가 필요하지만, 그중 불필요한 것은 과감히 제거해야 한다.

## 실전에서는 전략을 조합해 쓴다

단일 전략만으로는 부족하다. 실제 프로덕션 에이전트에서는 다음과 같이 계층적으로 조합한다:

```
┌─────────────────────────────────────┐
│  시스템 프롬프트 (항상 유지)          │
├─────────────────────────────────────┤
│  이전 대화 요약 (요약 전략)           │
├─────────────────────────────────────┤
│  RAG 검색 결과 (선택적 검색)          │
├─────────────────────────────────────┤
│  최근 N턴 대화 (잘라내기 전략)        │
├─────────────────────────────────────┤
│  에이전트 작업 노트 (구조화)          │
├─────────────────────────────────────┤
│  도구 정의 및 MCP 컨텍스트            │
└─────────────────────────────────────┘
```

이 계층 구조에서 각 레이어는 독립적으로 관리된다. 요약은 주기적으로 갱신되고, RAG 결과는 매 질문마다 교체되며, 최근 대화는 토큰 예산에 맞게 잘라낸다.

## 컨텍스트 엔지니어링의 함정

**과도한 압축은 독이다.** 핵심 정보를 잃은 요약은 180K 토큰의 원본보다 더 위험하다. 모델이 잘못된 요약을 사실로 받아들이면, 차라리 아예 없는 편이 낫다.

**검색 품질이 전체 품질을 결정한다.** RAG 기반 선택 전략에서 임베딩 모델이 관련 문서를 찾지 못하면, 컨텍스트에는 쓸모없는 정보만 들어간다. 검색 성능을 지속적으로 모니터링해야 한다.

**컨텍스트 격리를 잊지 마라.** 멀티 에이전트 시스템에서 한 에이전트의 컨텍스트가 다른 에이전트에 간섭하면 예측 불가능한 동작이 발생한다. 각 에이전트의 컨텍스트는 독립적으로 관리되어야 한다.

## 핵심 요약

- **컨텍스트 엔지니어링**은 프롬프트를 넘어 에이전트가 매 턴마다 처리하는 전체 정보를 최적화하는 작업이다
- **컨텍스트 부패**는 모든 모델에 나타나는 현상으로, 토큰이 많다고 좋은 게 아니다
- **4가지 전략** — 잘라내기, 요약, RAG 선택적 검색, 구조화 — 을 상황에 맞게 조합하라
- **최소 정보 원칙**: 원하는 동작을 만드는 데 필요한 최소 토큰 세트를 찾는 것이 핵심이다
- **지금 당장 할 일**: 현재 에이전트의 평균 컨텍스트 길이를 측정하고, 50턴 이상에서 품질 저하가 있는지 확인하라. 대부분의 에이전트는 아무런 관리 없이 돌아가고 있다
