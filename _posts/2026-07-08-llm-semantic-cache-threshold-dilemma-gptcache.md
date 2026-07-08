---
title: "LLM 의미론적 캐싱: 비용을 절반으로 줄이는 임계값의 딜레마"
date: "2026-07-08"
keywords: ["semantic cache", "GPTCache", "LLM 비용 최적화", "Redis vector cache", "캐싱 임계값"]
lang: "ko"
description: "LLM 의미론적 캐싱(Semantic Caching)으로 API 비용을 30~50% 줄이는 방법과, 단 하나의 숫자가 모든 것을 결정하는 임계값 트레이드오프를 정량 데이터로 분석한다."
---

# LLM 의미론적 캐싱: 비용을 절반으로 줄이는 임계값의 딜레마

고객 지원 봇을 운영한다고 가정해 보자. 하루에 10,000건의 질문이 들어오는데, 그중 상당수는 "비밀번호를 잊었어요", "비밀번호 어떻게 초기화하나요", "패스워드 리셋 방법"처럼 표현만 다를 뿐 같은 의미다. 이 매 요청마다 70B 파라미터 모델로 토큰을 생성하는 것은 낭비다. 더구나 H100 GPU 한 대의 시간당 비용이 2달러가 넘는 시장에서, 같은 답변을 반복 생성하는 것은 곧 직접적인 비용 손실이다.

의미론적 캐싱(Semantic Caching)은 이 문제를 해결한다. 들어오는 질문을 임베딩으로 변환하고, 과거 질문들과 코사인 유사도를 비교한다. 충분히 비슷한 질문이 있으면, LLM을 부르지 않고 저장된 답변을 3~8밀리초 만에 반환한다. Regmi와 Phakami Pun의 연구(arXiv:2411.05276)에 따르면, Redis 기반 의미론적 캐시로 API 호출을 최대 68.8%까지 줄이면서도 정확도 97% 이상을 유지할 수 있다고 한다.

하지만 여기에는 함정이 있다. "충분히 비슷하다"를 정의하는 단 하나의 숫자 — **유사도 임계값(threshold)** — 이 전체 시스템의 성패를 좌우한다. 이 글에서는 의미론적 캐싱이 어떻게 작동하는지, 임계값 설정이 왜 그렇게 까다로운지, 그리고 언제 도입해야 하고 언제 피해야 하는지를 정량 데이터와 코드로 살펴본다.

## 세 가지 캐싱 레이어: 의미론적, KV, 프롬프트

본론에 들어가기 전에, 흔히 혼동하는 개념부터 정리한다. LLM 캐싱에는 서로 다른 목적을 가진 세 가지 레이어가 있으며, 이들은 경쟁 관계가 아니라 보완 관계다.

**의미론적 캐시(Semantic Cache)**는 애플리케이션 계층에서 동작한다. 들어오는 질문을 임베딩하고, 과거 질문-답변 쌍을 벡터 저장소에 보관한다. 코사인 유사도가 임계값을 넘으면 저장된 응답 전체를 반환한다. LLM 자체를 건드리지 않으므로, 토큰 생성 비용과 지연 시간 모두 제로가 된다.

**KV 캐시**는 GPU 메모리 내부, 모델 추론 과정에서 동작한다. 어텐션(attention) 연산 중 계산된 Key-Value 텐서를 보관해, 이미 처리한 토큰을 재계산하지 않도록 한다. 긴 시스템 프롬프트나 멀티턴 대화에서 효과가 크다. 이는 vLLM, SGLang 같은 추론 프레임워크가 관리한다.

**프롬프트 캐시 / 접두어 캐시(Prefix Cache)**는 KV 캐시의 똑똑한 변형이다. 여러 요청에 걸쳐 동일한 프롬프트 접두어를 식별하고, 그 부분의 계산 결과를 재사용한다. SGLang의 RadixAttention이나 vLLM의 블록 레벨 접두어 캐싱이 여기에 해당한다. 일관된 시스템 프롬프트를 사용하는 팀이라면 이것만으로 입력 토큰 연산을 20~40% 줄일 수 있다.

핵심은 이 세 레이어가 독립적으로 동작한다는 점이다. 프롬프트 캐시로 접두어 연산을 아끼면서, 의미론적 캐시로 전체 응답을 재사용할 수 있다. 둘 다 적용해도 충돌하지 않는다.

## 의미론적 캐시는 어떻게 작동하는가

요청이 들어오면 세 단계를 거친다.

**1단계 — 임베딩.** 텍스트 임베딩 모델(`text-embedding-3-small`, `all-MiniLM-L6-v2` 등)로 질문을 벡터로 변환한다. OpenAI의 `text-embedding-ada-002`를 쓰면 1536차원 벡터가 나온다.

**2단계 — 유사도 검색.** 캐시에 저장된 모든 과거 질문 벡터와 코사인 유사도를 계산해, 가장 가까운 질문을 찾는다. 프로덕션에서는 pgvector, Redis Vector Search, Qdrant, Pinecone 같은 벡터 저장소를 쓴다.

**3단계 — 판정.** 최고 유사도가 임계값을 넘으면 저장된 응답을 반환하고 LLM 호출을 생략한다. 그렇지 않으면 LLM을 호출하고, 새 질문-답변 쌍을 캐시에 저장한다.

가장 단순한 형태의 코드는 이렇다.

```python
import numpy as np
from openai import OpenAI

client = OpenAI()
SIMILARITY_THRESHOLD = 0.95

def embed(text: str) -> np.ndarray:
    resp = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return np.array(resp.data[0].embedding)

def cosine(a: np.ndarray, b: np.ndarray) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def semantic_get(query: str, cache: list[dict]) -> str | None:
    if not cache:
        return None
    q_vec = embed(query)
    scores = [(cosine(q_vec, item["vec"]), item) for item in cache]
    scores.sort(key=lambda x: x[0], reverse=True)
    best_sim, best_match = scores[0]
    if best_sim >= SIMILARITY_THRESHOLD:
        return best_match["response"]
    return None
```

실제 프로덕션에서는 `list[dict]` 대신 벡터 저장소를 쓰고, 임계값, TTL, 캐시 무효화 전략을 별도로 관리해야 한다. 하지만 핵심 로직은 이게 전부다.

## 임계값의 딜레마: 하나의 숫자가 결정하는 것

여기가 이 글의 핵심이다. 임계값은 비용 절감액과 정확도를 동시에 결정하는 단일 변수다. 프로덕션 배포 데이터를 기반으로 한 임계값별 성능은 다음과 같다.

| 임계값 | 히트율 (고객지원 워크로드) | 오탐률(잘못된 답변 비율) |
|--------|--------------------------|------------------------|
| 0.99    | 1~3%     | 0.1% 미만           |
| 0.97    | 5~10%    | ~0.5%               |
| 0.95    | 15~25%   | 1~3%                |
| 0.93    | 25~40%   | 3~7%                |
| 0.90    | 35~55%   | 7~15%               |
| 0.85    | 45~70%   | 15~30%              |

> 위 수치는 Respan의 프로덕션 배포 관측 데이터를 기반으로 한다. Respan은 API 게이트웨이 서비스 기업으로, 고객사의 실제 배포에서 측정한 임계값-히트율-오탐률 상관관계를 공개했다. 워크로드 성격에 따라 실제 수치는 달라질 수 있다.

이 트레이드오프는 가혹하다. 임베딩 모델 운영 비용을 상쇄할 만한 히트율(보통 30% 이상)을 얻으려면 임계값을 0.93~0.95 수준으로 낮춰야 한다. 그런데 이 구간에서는 캐시 히트의 3~7%가 잘못된 답변을 반환한다.

구체적으로 계산해 보자. 하루 10,000건의 질문, 30% 히트율(임계값 0.93~0.95 수준)을 가정하면, 3,000건이 캐시에서 반환된다. 그중 3~7%인 90~210건이 틀린 답변이다. 시스템은 이것을 "정확한 답변"으로 간주하기 때문에, 사용자가 틀렸다는 것을 알아차리기 전까지 아무도 모른다.

이것이 공급자 프롬프트 캐시(provider prompt cache)와 의미론적 캐시의 근본적 차이다. 프롬프트 캐시는 정확한 접두어 매칭이므로 잘못된 히트가 사실상 불가능하다. 반면 의미론적 캐시는 "비슷한 의미"라는 확률적 판정에 의존하므로, 조용히 실패(silent failure)할 수 있다.

## GPTCache로 구현하기: 실전 설정

GPTCache는 이 분야에서 가장 널리 쓰이는 오픈소스 라이브러리다. Zilliz(Milvus 개발사)가 만들었고 GitHub에서 8,100개 이상의 스타를 받았으며, MIT 라이선스로 공개되어 있다. LangChain 및 LlamaIndex와 완전 통합되어 있고, FAISS, Qdrant, Redis, Milvus, Weaviate 등 다양한 벡터 저장소 백엔드를 지원한다.

가장 단순한 형태는 기본 설정으로 시작하는 것이다.

```python
# pip install gptcache
from gptcache import cache
from gptcache.adapter import openai

# 기본 설정: SQLite(메타데이터) + 임베딩 캐시
cache.init()
cache.set_openai_key()

# 이후 openai.ChatCompletion.create 호출은 자동으로 캐시를 거친다
# (GPTCache adapter가 OpenAI v0.x SDK 스타일을 래핑함)
response = openai.ChatCompletion.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "비밀번호를 잊었어요"}],
)
```

기본 설정은 정확 매칭(exact match) 캐시로 동작한다. 의미론적 매칭을 활성화하려면 임베딩 모델과 유사도 평가를 직접 설정해야 한다. GPTCache 공식 문서에서 권장하는 커스텀 설정은 이렇다.

```python
from gptcache import cache, Config
from gptcache.embedding import Onnx as Embedding
from gptcache.manager import get_data_manager, CacheBase, VectorBase
from gptcache.similarity_evaluation import SearchDistanceEvaluation

# 로컬 ONNX 임베딩 (OpenAI API 호출 없이 임베딩 생성)
embedding = Embedding(model_name="all-MiniLM-L6-v2")

# 백엔드: SQLite(메타데이터) + FAISS(벡터)
data_manager = get_data_manager(
    CacheBase("sqlite"),
    VectorBase("faiss", dimension=embedding.dimension),
)

cache.init(
    embedding_func=embedding.to_embeddings,
    data_manager=data_manager,
    similarity_evaluation=SearchDistanceEvaluation(
        max_distance=0.1,  # ← 핵심: 거리 임계값 (낮을수록 엄격)
        positive=False,
    ),
    config=Config(similarity_threshold=0.95),  # 의미론적 매칭 활성화
)
```

`max_distance`와 `similarity_threshold`가 이 설정의 핵심이다. 코사인 유사도 0.95는 거리(distance)로 환산하면 약 0.05에 해당한다. GPTCache의 `SearchDistanceEvaluation`은 거리 기반으로 동작하므로, 유사도 0.95를 원한다면 `max_distance`를 0.05~0.1 사이로 설정해야 한다. 임계값을 높이면(예: 유사도 0.99 = 거리 0.01) 히트율이 1~3%로 떨어져 캐싱의 의미가 사라진다. 반대로 낮추면(예: 유사도 0.90 = 거리 0.1) 히트율은 35~55%로 올라가지만, 잘못된 답변이 7~15%나 된다.

Redis를 백엔드로 쓰면 운영 환경에 더 적합하다. Redis Stack의 RediSearch 모듈은 코사인 유사도, 내적(IP), L2 거리를 모두 지원하며, 기존 Redis 인프라를 그대로 활용할 수 있다.

```python
from gptcache.manager import VectorBase

vector_base = VectorBase(
    "redis",
    host="localhost",
    port=6379,
    dimension=384,  # all-MiniLM-L6-v2 기준
    namespace="llm_cache",
)
```

> **참고**: GPTCache 공식 README에 따르면, API 형태가 계속 진화하고 있어 최신 설정 방법은 공식 문서를 참조해야 한다. 새로운 모델이나 API에 대한 어댑터 지원은 중단되었으며, 대신 `cache.get()` / `cache.put()` 직접 API 사용을 권장하고 있다.

## 언제 도입하고, 언제 피해야 하나

의미론적 캐싱은 워크로드 성격에 따라 효과가 극과 극으로 갈린다.

**도입을 권장하는 워크로드:**

- **FAQ / 고객지원 봇** — 사용자들이 비슷한 질문을 다른 말로 한다. "비밀번호를 잊었어요"와 "패스워드 초기화 방법"을 같은 답변으로 처리할 수 있다. 히트율 50~70%가 현실적이다.
- **에이전트 프레임워크의 도구 호출** — LangChain이나 LlamaIndex 기반 에이전트는 매 호출마다 동일한 도구 설명과 스키마를 보낸다. 시스템 프롬프트와 도구 스키마가 수천 요청에 걸쳐 거의 동일하므로, 히트율이 60%를 넘기도 한다.
- **정적 문서 기반 RAG** — 문서 코퍼스가 자주 바뀌지 않으면, 비슷한 질문이 비슷한 청크를 검색하고 비슷한 답변을 생성한다. 히트율 30~50% 수준.
- **Mixture of Agents의 제안 모델** — 동일한 사용자 질문이 결정론적 제안 출력으로 이어지므로, 제안 계층에서 캐싱하면 반복 질문의 GPU 시간을 60~80% 줄일 수 있다.

**도입을 피해야 하는 워크로드:**

- **창의적 생성 (temperature > 0.5)** — 매 요청이 의도적으로 다르다. 캐시 미스율이 95% 이상이며, 임베딩 오버헤드만 낭비다.
- **상태 기반 멀티턴 대화** — 각 턴이 전체 대화 기록에 의존하므로, 질문이 구조적으로 고유해진다. 히트율이 5~15%에 그친다.
- **개인화 추천** — 프롬프트에 사용자별 데이터가 인코딩되므로, 질문은 구조적으로 비슷하지만 의미적으로 다르다. 다른 사용자의 추천을 반환할 위험이 있다.
- **규제 산업 (의료, 금융)** — "자신만만하게 틀린 답변"이 규제 위반이 되는 분야에서는, 3%의 오탐률도 감당할 수 없다.

## 평가 루프 없이 배포하지 마라

의미론적 캐시의 가장 위험한 특성은 **조용한 실패**다. 정확 매칭 캐시나 프롬프트 캐시와 달리, 의미론적 캐시의 잘못된 히트는 로그에 에러로 잡히지 않는다. 시스템은 그것을 "정상적인 캐시 히트"로 기록한다.

따라서 배포 전에 평가 루프를 반드시 구축해야 한다. 샘플링한 캐시 히트를 사람이나 별도 LLM으로 채점해, 오탐률을 실시간으로 추적해야 한다. Respan의 권장 경로는 이렇다.

1. **정확 매칭 캐시 먼저** — 정확도 리스크가 제로이고, 구현이 단순하다.
2. **공급자 프롬프트 캐시** — 긴 접두어에 대해 무료로 적용되는 윈윈.
3. **의미론적 캐시는 마지막에** — 위 두 단계로 충분하지 않을 때, 그리고 평가 루프를 운영할 수 있을 때만.

## 결론: 임계값은 기술이 아니라 제품 결정이다

의미론적 캐싱은 강력한 도구다. 논문 데이터에 따르면 API 호출을 68.8%까지 줄일 수 있고, 응답 시간은 500~2000밀리초에서 3~8밀리초로 단축된다. 하지만 그 대가는 정확도 저하 가능성이다.

- **임계값 0.95 이상**을 유지하면 오탐률을 1~3%로 낮출 수 있지만, 히트율은 15~25%로 제한된다.
- **임계값을 낮추면** 비용 절감은 커지지만, 하루에 90~210명의 사용자가 틀린 답변을 받을 수 있다.
- **평가 루프 없이 의미론적 캐시를 배포하지 마라.** 조용한 실패는 눈에 보이지 않는다.
- **정확 매칭과 프롬프트 캐시로 먼저 충분한지 확인하라.** 그 두 가지는 정확도 리스크 없이 비용을 줄인다.

의미론적 캐싱에서 임계값은 단순한 하이퍼파라미터가 아니다. "몇 명의 사용자가 틀린 답변을 받는 것을 감수할 것인가"라는 제품 결정이다. 그 숫자를 의식적으로 선택하는 팀만이 이 기술의 이점을 안전하게 누릴 수 있다.
