---
title: "RAG가 '비슷한 문서'가 아니라 '맞는 문서'를 찾게 하는 한 겹: Cross-encoder Reranker 선택과 배포"
date: "2026-07-16"
keywords: ["Reranker", "Cross-encoder", "BGE-Reranker", "Cohere Rerank", "RAG 검색 품질", "NDCG"]
lang: "ko"
description: "Bi-encoder 검색이 놓치는 정밀 매칭을 Cross-encoder Reranker가 복구하는 원리와, BGE·Jina·Cohere 모델의 지연·비용·정확도 트레이드오프를 코드와 함께 정리한다."
---

# RAG가 '비슷한 문서'가 아니라 '맞는 문서'를 찾게 하는 한 겹: Cross-encoder Reranker 선택과 배포

RAG(Retrieval-Augmented Generation) 파이프라인이 처음 세팅되는 순간, 대부분의 팀은 같은 실수를 한다. 임베딩 모델 하나 골라 벡터 DB에 넣고, 상위 K개를 꺼내서 LLM에 주면 끝이라고 믿는 것이다. 실제로 돌려보면 결과는 꽤 괜찮아 보인다. 하지만 질문이 조금만 복잡해져도 — 부정어("A가 아닌 것"), 수식어("최근 3개월"), 다단계 조건("B를 만족하면서 C는 제외") — 검색이 어긋나기 시작한다. 임베딩 검색은 빠르지만, 질문과 문서가 서로를 직접 바라보지 못하기 때문이다.

이 간극을 메우는 것이 **Reranker(재순위 모델)**다. 검색 품질 개선에 있어 단일 단계로 가장 큰 ROI를 주는 컴포넌트로 평가받는다. 이 글에서는 Reranker가 왜 효과적인지, 주요 모델을 어떻게 선택하는지, 그리고 프로덕션에서 지연시간과 비용을 어떻게 다루는지를 코드와 함께 정리한다.

## 섹션 1: Bi-encoder가 놓치는 것, Cross-encoder가 보는 것

현재 대부분의 RAG가 사용하는 임베딩 검색은 **Bi-encoder**(이중 인코더) 구조다. 질문 Q와 문서 D를 각각 독립적으로 벡터로 변환한 뒤, 코사인 유사도로 점수를 매긴다.

```
점수 = cos(emb(Q), emb(D))
```

이 구조의 치명적 특징은, 질문 벡터를 만들 때 문서를 보지 못하고, 문서 벡터를 만들 때 질문을 보지 못한다는 점이다. 둘은 완전히 분리되어 인코딩된다. 그래서 "A가 아닌 것"이라는 부정어, "최근"이라는 시간 한정어, 토큰 간 미세한 대응 관계가 유실된다. 대신 문서 당 벡터 하나만 저장하면 되고, 검색이 1ms 이내에 끝나므로 수백만 문서에서 첫 단계 후보를 뽑는 데는 최적이다.

반면 **Cross-encoder**(교차 인코더)는 질문과 문서를 하나의 시퀀스로 합쳐서 `[CLS] 질문 [SEP] 문서 [SEP]` 형태로 트랜스포머에 통과시킨다. 질문의 모든 토큰이 문서의 모든 토큰에 주의(attention)를 주고, 그 풍부한 상호작용 끝에서 단일 점수를 낸다. 부정어, 수식어, 다단계 조건이 정확히 이 교차 주의(cross-attention)에서 처리된다.

대표적인 검색 벤치마크(BEIR)에서 두 단계를 비교하면 차이가 분명히 드러난다. 한 정리에 따르면 BM25 단독이 평균 NDCG@10 41.7, Bi-encoder(BGE-base)가 51.0, 여기에 Cross-encoder Reranker를 얹으면 56.5까지 올라간다. 약 5~7 NDCG 포인트의 향상은, RAG가 "그럴듯한 답"을 넘어 "정확한 답"을 내는 분기점이 되는 경우가 많다. 정리된 가이드들에서는 보통 +5~15 NDCG@10 향상을 기대할 수 있다고 본다.

전형적인 2단계 파이프라인은 이렇게 된다:

```
질문 → Bi-encoder 검색 (top 100) → Cross-encoder Reranker (top 10) → LLM
         ~1ms                           50~500ms
```

첫 단계는 넓게 후보를 거르고(정확도는 약간 포기하되 속도 확보), 두 번째 단계는 적은 후보에 대해 정밀하게 점수를 매겨 순위를 다시 만든다.

## 섹션 2: 2026년 주요 Reranker 모델 비교

오픈소스와 API 기반 모델이 섞여 있어 선택이 까다롭다. 아래는 MTEB Rerank 리더보드 및 각 모델의 Hugging Face 모델 카드를 기준으로 정리한 비교다(MTEB 점수는 벤치마크 버전·설정에 따라 변동할 수 있으므로 참고치로 볼 것).

### 오픈소스 (자체 호스팅)

| 모델 | 파라미터 | 라이선스 | MTEB Rerank | 특징 |
|------|---------|---------|-------------|------|
| BGE-Reranker-v2-m3 | 568M | MIT | 60.4 | 다국어 기본 선택 |
| BGE-Reranker-v2-Gemma | 2.5B | Apache 2.0 | 64.5 | 최고 품질, LLM 기반 구조 |
| BGE-Reranker-v2-MiniCPM | 2.7B | Apache 2.0 | 62.3 | 품질·속도 균형 |
| Jina Reranker v2 (multilingual) | 278M | Apache 2.0 | 56.8 | 저지연 |
| mxbai-rerank-large-v1 | 435M | Apache 2.0 | 59.4 | 영어, 빠름 |

**BGE-Reranker-v2-m3**가 2026년 현재 가장 합리적인 기본값이다. 100개 이상 언어를 지원하고, MIT 라이선스로 상용에 제약이 없으며, 568M 파라미터라 GPU에서 50~100ms 안에 처리된다. 다국어 한국어 환경에서도 검증된 선택지다.

품질이 최우선이고 지연시간을 감당할 수 있다면 **BGE-Reranker-v2-Gemma**(약 2.5B)가 정상권이다. MTEB Rerank 64.5로 오픈소스 최상위권이며, 일반 Cross-encoder와 달리 Gemma LLM을 기반(text-generation 아키텍처)으로 하여 더 깊은 문맥 이해가 가능하다. 다만 2.5B 규모라 BGE-v2-m3보다 추론 비용이 크다.

**Jina Reranker v2**는 278M 파라미터로 지연에 민감한 서비스에 적합하다. 다만 정확도는 BGE v2 계열보다 낮으므로, 1차 후보를 많이(예: 100~200개) 잡아 정밀 보정이 덜 중요한 경우에 맞다.

### API 기반 (관리형)

호스팅 인프라를 직접 운영하기 부담스럽다면 API가 깔끔하다.

| 서비스 | 지연시간 | 비용 | 특징 |
|--------|---------|------|------|
| Cohere Rerank 3 (v3.5) | 100~150ms | $2.00 / 1M 토큰 | 안정성 보장, 주요 언어 지원 |
| Voyage Rerank 2 | API 의존 | API 의존 | MTEB 61.5 수준 |
| Jina Rerank API | 100~200ms | API 의존 | 오픈소스 모델의 관리형 버전 |

Cohere Rerank는 2026년 7월 기준 **$2.00 / 1M 토큰**의 토큰 기반 요금제다(검색 입력 토큰 기준). 예를 들어 쿼리당 100개 문서 × 문서당 평균 200토큰이라면, 1M 토큰으로 약 50쿼리를 처리할 수 있어 쿼리당 약 $0.04에 해당한다. 하루 1만 쿼리 미만이라면 관리형이 운영 부담이 적지만, 그 이상에서는 자체 호스팅 BGE가 며칠 안에 비용을 상환하는 경우가 많다.

## 섹션 3: 실전 배포 — sentence-transformers와 TEI

### sentence-transformers로 빠르게 프로토타입

가장 빠른 진입은 `sentence-transformers`의 `CrossEncoder` 클래스다.

```python
from sentence_transformers import CrossEncoder
import time

# BGE-Reranker-v2-m3 로드
reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")

query = "A 기능을 지원하지 않는 플랜은?"
docs = [
    "Pro 플랜은 A 기능을 지원합니다.",
    "Free 플랜은 A 기능을 지원하지 않습니다.",
    "Pro 플랜 가격은 월 $20입니다.",
]

start = time.time()
scores = reranker.predict([[query, doc] for doc in docs])
print(f"지연: {(time.time() - start) * 1000:.0f}ms")

# 점수 높은 순 정렬
ranked = sorted(zip(scores, docs), reverse=True)
for score, doc in ranked:
    print(f"{score:.4f}  {doc}")
```

부정어("지원하지 않습니다")가 포함된 문서가 정확히 상위로 올라오는 것을 볼 수 있다. 이게 Bi-encoder가 놓치는 교차 주의의 힘이다. BAAI 공식 패키지인 `FlagEmbedding`의 `FlagReranker`를 써도 같은 모델을 로드할 수 있으며, 디바이스(`use_fp16`) 등 세부 설정이 더 편하다.

### Text Embeddings Inference (TEI)로 프로덕션 서빙

`sentence-transformers`는 단일 프로세스에서 좋지만, 동시 요청이 늘면 병목이 생긴다. Hugging Face의 **TEI(Text Embeddings Inference)**는 고성능 추론 서버로, 내부에 Candle(Rust 기반 ML 프레임워크), Flash Attention, cuBLASLt를 활용해 Reranker를 포함한 임베딩·분류 모델을 빠르게 서빙한다.

```bash
# TEI로 BGE-Reranker-v2-m3 서빙 (GPU)
docker run --gpus all -p 8080:80 \
  -v $PWD/data:/data \
  ghcr.io/huggingface/text-embeddings-inference:1.5 \
  --model-id BAAI/bge-reranker-v2-m3 \
  --max-batch-tokens 16384
```

```bash
# REST 호출
curl http://localhost:8080/rerank \
  -H "Content-Type: application/json" \
  -d '{
    "query": "A 기능을 지원하지 않는 플랜은?",
    "texts": [
      "Pro 플랜은 A 기능을 지원합니다.",
      "Free 플랜은 A 기능을 지원하지 않습니다."
    ]
  }'
```

TEI는 내부적으로 동적 배칭을 해서, 여러 요청의 후보 문서를 한 번에 처리하며 GPU 활용도를 높인다. Reranker는 문서마다 독립 추론이 필요하므로 배치 효율이 지연시간과 직결된다.

## 섹션 4: K(재순위 대상 수)와 지연시간 다루기

Reranker 도입에서 가장 많이 헷갈리는 결정이 두 개의 K다.

- **K1 (1차 검색 후보)**: Bi-encoder가 가져올 후보 수. 너무 적으면 정답이 후보에 없고, 너무 많으면 Reranker 부하가 커진다. 보통 50~100.
- **K2 (Reranker 후 최종)**: LLM에 넘길 문서 수. 보통 5~10. 그 이상은 컨텍스트 비용만 키우고 정확도는 미미하다.

지연시간은 후보 수와 Reranker 모델 크기에 거의 선형으로 비례한다. BGE-Reranker-v2-m3 기준 GPU에서 후보 100개 처리에 약 50~100ms, 후보 50개면 절반 이하로 줄어든다. CPU에서는 200~400ms로 실시간 서비스에 부적합하므로, GPU가 없다면 API 기반(Cohere, Jina)이나 더 가벼운 모델(mxbai-rerank-base)을 선택해야 한다.

실전 팁을 정리하면:

1. **먼저 K1=100, K2=10으로 세팅**하고 NDCG@10로 측정한 뒤, K1을 점차 줄여 지연을 최적화한다.
2. **지연 예산이 200ms**라면, GPU에서 BGE-v2-m3(후보 100)가 들어맞는다. 예산이 50ms라면 mxbai-rerank-base나 API를 검토한다.
3. **캐싱을 잊지 말 것**. 자주 들어오는 질문-문서 쌍의 Reranker 점수는 캐싱하면 비용과 지연을 동시에 줄인다. (이전에 다룬 의미론적 캐싱과 결합하기 좋다.)

## 결론

Reranker는 RAG 파이프라인에서 가장 비용 효율적인 단일 개선책이다. 정리하면:

- **Bi-encoder는 속도, Cross-encoder는 정확도**다. 둘을 2단계로 결합하는 것이 표준.
- **BGE-Reranker-v2-m3**가 2026년 다국어 환경의 안전한 기본값이다. MIT 라이선스, GPU 50~100ms 지연, 100개 이상 언어 지원.
- **자체 호스팅 vs API**는 하루 1만 쿼리 정도가 분기점이다. 그 이하면 Cohere·Jina API가 운영이 편하고, 이상이면 TEI로 BGE를 직접 굴리는 것이 비용이 유리하다.
- **K1(1차 후보) 50~100, K2(최종) 5~10**에서 시작해 벤치마크로 조정하라.

지금 RAG의 검색 품질에 만족하지 못하고 있다면, 가장 먼저 해볼 일은 `sentence-transformers`로 BGE-Reranker-v2-m3를 얹어보는 것이다. 십중팔구, 답변 품질의 가장 큰 단일 도약을 그 한 겹에서 얻게 될 것이다.
