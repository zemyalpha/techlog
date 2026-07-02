---
title: "pgvector에서 Qdrant로: 프로덕션 RAG가 벡터 DB를 갈아타는 분기점"
date: "2026-07-02"
keywords: ["벡터 데이터베이스", "pgvector", "Qdrant", "RAG", "HNSW"]
lang: "ko"
description: "pgvector iterative scan과 Qdrant 양자화가 2026년 프로덕션 RAG의 벡터 DB 선택 기준을 어떻게 바꾸는지, 실제 전환 시점과 마이그레이션 전략을 정리한다."
---

# pgvector에서 Qdrant로: 프로덕션 RAG가 벡터 DB를 갈아타는 분기점

2026년 중반, 프로덕션 RAG 시스템에서 벡터 데이터베이스를 선택하는 문제는 2년 전과 완전히 다른 양상을 보인다. 가장 큰 변화는 두 가지다. 첫째, pgvector가 필터링 + HNSW 결합의 치명적 약점이었던 "오버필터링" 문제를 iterative scan으로 해결했다. 둘째, Qdrant가 양자화(quantization)를 네이티브로 지원하면서 4~32배 메모리 압축을 기본 옵션으로 만들었다.

이 두 가지 변화가 만나는 지점에서, "pgvector로 시작하다가 Qdrant로 넘어간다"는 패턴이 프로덕션 RAG의 사실상 표준이 되고 있다. 이 글에서는 그 전환의 분기점을 기술적 기준으로 정리한다.

## 왜 pgvector로 시작하는가: 운영 복잡도 제로

대부분의 RAG 프로젝트는 Postgres를 이미 운영 중인 상태에서 시작한다. pgvector는 이 Postgres 인스턴스에 `CREATE EXTENSION vector;` 한 줄로 추가된다. 새로운 인프라를 띄울 필요도, 새로운 인증 체계를 도입할 필요도, 새로운 백업 전략을 짤 필요도 없다.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    embedding vector(1536),
    tenant_id UUID NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_documents_embedding
ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

pgvector의 진짜 강점은 속도가 아니다. 벡터 유사도 검색과 SQL 조건을 하나의 쿼리에서 결합할 수 있다는 것이다. 테넌트 격리, 시간 범위 필터, 메타데이터 조건을 모두 포함한 검색이 애플리케이션 레벨 조인 없이 한 번에 실행된다.

```sql
SELECT id, content, metadata,
       1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE tenant_id = $2
  AND metadata->>'category' = 'policy'
  AND created_at > NOW() - INTERVAL '90 days'
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

벡터 데이터와 관계형 데이터가 같은 트랜잭션에 들어간다. ACID 보장, 포인트인-타임 복구, 백업이 모두 Postgres의 기존 인프라를 그대로 탄다. 수백만 벡터 규모에서는 이 운영 단순성이 다른 모든 장점을 압도한다.

## 오버필터링: pgvector가 가졌던 치명적 약점

문제는 `WHERE` 절과 HNSW 인덱스를 결합할 때 발생했다. pgvector의 근사 최근접 이웃(ANN) 검색은 인덱스 스캔이 끝난 *후에* 필터를 적용한다. 기본 `ef_search=40`으로 40개 후보를 가져왔는데 필터가 90%를 제거하면, 결과는 4개뿐이다. `LIMIT 10`을 요청했어도.

이 문제를 "오버필터링(overfiltering)"이라 부른다. 선택적인 메타데이터 필터를 건 검색일수록 재현율이 급격히 떨어지는 현상이다. pgvector 0.7.x 시대에는 이를 우회하기 위해 `ef_search`를 수백으로 올리는 꼼수가 쓰였지만, 이는 레이턴시를 희생하는临时 방편이었다.

## iterative scan: pgvector 0.8.0의 해결책

pgvector 0.8.0(2024년 10월 출시)이 이 문제에 대한 공식 답을 내놓았다. iterative index scan이다. 초기 인덱스 스캔 결과가 `WHERE` 조건을 충족하지 못하면, pgvector가 자동으로 인덱스를 추가로 탐색한다. 충분한 결과가 나올 때까지, 또는 설정한 한계에 도달할 때까지.

```sql
-- HNSW iterative scan 활성화
SET hnsw.iterative_scan = strict_order;
SET hnsw.max_scan_tuples = 50000;

-- 이제 WHERE 필터가 선택적이어도 재현율이 유지된다
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE tenant_id = $2
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY embedding <=> $1::vector
LIMIT 20;
```

`strict_order` 모드는 정확한 거리 순서를 보장하지만 재현율이 약간 낮고, `relaxed_order` 모드는 순서가 미세하게 어긋나는 대신 더 높은 재현율을 제공한다. 실무에서는 `relaxed_order`가 대부분의 필터링 워크로드에 적합하다.

이 개선으로 pgvector는 "메타데이터 필터가 많은 프로덕션 RAG에서는 쓸 수 없다"는 이전의 평가에서 벗어났다. PostgreSQL 공식 발표([postgresql.org](https://www.postgresql.org/about/news/pgvector-080-released-2952/))와 pgvector GitHub README 모두 이 기능을 명시적으로 문서화하고 있다.

## pgvector의 한계가 시작되는 지점

iterative scan으로 필터링 문제는 해결됐지만, pgvector의 근본적 한계는 여전히 존재한다.

**첫째, 메모리 압축이 없다.** pgvector는 float32(4바이트)와 halfvec(float16, 2바이트)를 지원한다. halfvec로 2배 압축은 가능하지만, int8 수준의 4배 압축은 아직 실험적이다. 1000차원 벡터 1억 개를 float32로 저장하면 약 400GB. 이것이 전부 RAM에 올라가야 HNSW가 제 성능을 낸다.

**둘째, 양자화된 벡터와 원본을 병렬로 관리하는 메커니즘이 없다.** 검색은 양자화된 벡터로 빠르게 하고, 최종 후보만 원본 벡터로 재평가(rescoring)하는 패턴은 대규모 배포의 표준이다. pgvector에는 이 이중 저장 구조가 없다.

**셋째, 수평 확장이 어렵다.** Postgres는 단일 노드 시스템이다. Citus로 샤딩은 가능하지만, 벡터 검색에 특화된 분산 쿼리 최적화는 없다. 벡터 수가 단일 인스턴스의 메모리를 넘어서면 읽기 복제본, 파티셔닝, 또는 전환을 검토해야 한다.

실무 경험에 따르면, 대략 **단일 Postgres 인스턴스 기준 1,000만~2,000만 벡터**가 pgvector의 실용적 한계로 보고된다. 이를 넘어서면 인덱스 빌드 시간이 급증하고, VACUUM이 쿼리 트래픽과 경쟁하기 시작한다.

## Qdrant가 제공하는 것: 양자화와 분산

Qdrant는 Rust로 작성된 오픈소스 벡터 데이터베이스다. pgvector를 넘어서는 핵심 이유는 양자화(quantization)의 네이티브 구현에 있다.

Qdrant는 세 가지 양자화 방식을 지원한다 (공식 문서 [qdrant.tech](https://qdrant.tech/documentation/manage-data/quantization/) 기준):

| 방식 | 압축비 | 정확도 | 도입 버전 |
|------|--------|--------|-----------|
| Scalar (int8) | 4x | 99%+ | 1.1.0 |
| Binary (1-bit) | 32x | 95% (호환 모델) | — |
| Product Quantization | 4~64x | 70~97% | 1.2.0 |

가장 실용적인 것은 scalar quantization이다. float32(4바이트)를 int8(1바이트)로 매핑하여 4배 메모리 압축을 달성하면서 정확도 손실을 1% 이내로 유지한다. SIMD 명령어를 활용해 연산 속도도 최대 2배 빨라진다.

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="documents",
    vectors_config=models.VectorParams(
        size=1536,
        distance=models.Distance.COSINE,
        on_disk=True,  # 원본 벡터는 디스크로
    ),
    quantization_config=models.ScalarQuantization(
        scalar=models.ScalarQuantizationConfig(
            type=models.ScalarType.INT8,
            quantile=0.99,  # 극단값 1% 제외
            always_ram=True,  # 양자화 벡터는 RAM 상주
        )
    ),
)
```

위 설정의 핵심은 `on_disk=True` + `always_ram=True` 조합이다. 원본 벡터는 디스크에, 양자화된 벡터는 RAM에 둔다. 검색은 RAM의 양자화 벡터로 빠르게 수행하고, 최종 후보만 디스크에서 원본을 읽어 정확한 재평가를 한다. 이 이중 저장 전략으로 RAM 사용량을 획기적으로 줄이면서 레이턴시 손실을 최소화한다.

추가로, Qdrant v1.15.0부터는 1.5-bit와 2-bit binary quantization이 도입되어, 기존 binary(1-bit)와 scalar(int8) 사이의 중간 지점을 제공한다.

## 인그래프 필터링: Qdrant의 두 번째 무기

Qdrant의 또 다른 차별점은 메타데이터(payload) 필터를 HNSW 그래프 탐색 *내부*에서 수행한다는 것이다. pgvector는 인덱스 스캔 후에 필터를 적용하지만, Qdrant는 그래프를 순회하면서 동시에 필터 조건을 검사한다.

선택적인 필터가 걸린 대규모 컬렉션에서는 이 차이가 결정적이다. pgvector의 iterative scan이 사후 탐색 확장으로 문제를 완화한다면, Qdrant는 처음부터 그래프 내부에서 불필요한 경로를 가지치기한다.

```python
results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    query_filter=models.Filter(
        must=[
            models.FieldCondition(
                key="category", match=models.MatchValue(value="policy")
            ),
            models.FieldCondition(
                key="tenant", match=models.MatchValue(value="acme-corp")
            ),
            models.FieldCondition(
                key="created_timestamp",
                range=models.Range(gte=1704067200)
            ),
        ]
    ),
    limit=5,
    with_payload=True,
)
```

필터에 사용할 필드는 사전에 인덱스를 생성해야 최적의 성능이 나온다.

```python
client.create_payload_index(
    collection_name="documents",
    field_name="category",
    field_schema="keyword",
)
```

## 하이브리드 서치: 2026년의 표준

순수 벡터 검색은 키워드 정확 매칭이 필요한 쿼리에서 실패한다. 제품 코드, 고유명사, 약어 등은 의미적 유사도로 잡히지 않는다. 2026년 프로덕션 RAG에서 BM25(키워드)와 dense(의미) 검색을 결합하는 하이브리드 서치는 선택이 아닌 기본이다.

**pgvector**에서는 Postgres의 `tsvector` 전문 검색과 벡터 검색을 RRF(Reciprocal Rank Fusion)로 결합한다. SQL 레벨에서 직접 조합해야 하므로 코드가 길어지지만, 두 검색이 같은 쿼리 안에서 실행된다.

**Qdrant**에서는 sparse vector와 dense vector를 같은 컬렉션에 저장하고, 쿼리 시점에 융합한다.

```python
client.query_points(
    collection_name="docs",
    prefetch=[
        models.Prefetch(query=dense_vec, using="dense", limit=50),
        models.Prefetch(query=sparse_vec, using="bm25", limit=50),
    ],
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=20,
)
```

## 언제 전환해야 하는가: 실용적 분기점

여러 실무 보고와 업계 관행을 종합하면, pgvector에서 Qdrant(또는 다른 전용 벡터 DB)로의 전환 분기점은 다음과 같다.

**pgvector를 유지해도 되는 경우:**
- Postgres를 이미 운영 중이고 벡터 수가 1,000만 이하
- 벡터 데이터와 관계형 데이터의 트랜잭션 일관성이 중요
- 운영 팀에 Docker/Kubernetes 경험이 없거나 인력이 부족
- RAG의 정확도를 결정하는 요인이 DB가 아니라 청킹 전략과 임베딩 모델이라는 점을 이해하는 경우

**Qdrant로 전환을 검토해야 하는 경우:**
- 벡터 수가 1,000만~5,000만을 넘어서며 단일 Postgres 인스턴스의 메모리가 부족
- 선택적인 메타데이터 필터가 대부분의 쿼리에 포함되며 레이턴시 요구사항이 엄격
- RAM 비용이 인프라 비용의 주된 원인이 되어 4배 이상 압축이 필요
- 하이브리드 서치가 핵심 기능이며, SQL 레벨 조합의 복잡도가 부담

**Pinecone을 선택하는 경우:**
- 운영 인력 없이 관리형 서비스로만 가야 하는 상황
- 1억 벡터 이상에서 서브 20ms p95가 하드 요구사항
- 비용보다 운영 단순성이 우선

## 마이그레이션은 생각보다 저렴하다

벡터 DB 간 마이그레이션은 일반적으로 "한 Python 스크립트"로 요약된다. 한쪽에서 임베딩을 읽어 다른 쪽에 쓰면 된다. 벡터는 결국 부동소수점 배열일 뿐이다. 진짜 비용은 쿼리 로직 재작성에 있다. 필터 문법, 멀티테넌시 구현, 에러 처리가 DB마다 다르기 때문이다.

따라서 실용적 조언은 명확하다. **pgvector로 시작하고, 실제로 한계에 부딪혔을 때 Qdrant로 이동하라.** 미리 최적화하느라 시간을 쓰는 것보다, pgvector로 1~2년을 보내면서 실제 워크로드 패턴을 파악한 뒤 전환하는 것이 더 빠르고 더 저렴하다. 대부분의 RAG 프로젝트는 pgvector의 한계에 도달하기 전에 끝난다.

## 핵심 요약

- pgvector 0.8.0의 iterative scan이 메타데이터 필터링 + HNSW의 오버필터링 문제를 해결했다. 프로덕션 RAG의 기본 선택으로 충분하다.
- Qdrant의 네이티브 양자화(scalar int8 = 4배 압축, 99% 정확도)와 인그래프 필터링은 1,000만 벡터 이상에서 결정적 이점이 된다.
- 2026년 프로덕션 RAG에서 하이브리드 서치(BM25 + dense)는 표준이다. pgvector는 SQL로, Qdrant는 sparse vector로 구현한다.
- 전환 분기점은 벡터 수(약 1,000만~2,000만)와 메모리 비용, 필터 선택도로 결정된다.
- pgvector로 시작하고, 한계에 부딪히면 Qdrant로 이동하는 것이 가장 안전한 경로다. 마이그레이션 비용은 생각보다 낮다.
- RAG 정확도를 좌우하는 것은 벡터 DB가 아니라 청킹 전략, 임베딩 모델, 리랭커다. DB 선택에 에너지를 쏟기 전에 검색 파이프라인 전체를 먼저 점검하라.
