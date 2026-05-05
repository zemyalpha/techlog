---
title: "LLM 메모리를 지식 그래프로 구조화하는 기술 — 프로젝트·논문·온톨로지 총정리"
date: "2026-05-05"
keywords: ["LLM memory", "knowledge graph", "personal knowledge graph", "온톨로지", "Graphiti", "Mem0", "HippoRAG"]
lang: "ko"
description: "LLM이 기억하는 개인 정보를 타입화된 지식 그래프로 변환하는 오픈소스 프로젝트 13개, 핵심 논문 13편, 온톨로지 구축 방법론을 총정리한다."
---

# LLM 메모리를 지식 그래프로 구조화하는 기술 — 프로젝트·논문·온톨로지 총정리

ChatGPT, Claude, Gemini와 대화하다 보면 놀랍게도 AI가 당신을 꽤 잘 알고 있다. 직업, 취향, 최근 고민거리까지. 하지만 이 "기억"은 사실 납작한 텍스트 덩어리일 뿐이다. 같은 사실이 5번 반복되면 5개의 증거처럼 보이지만, 실제로는 동일한 정보의 재탕이다.

이 문제를 해결하려는 움직임이 2024~2026년 사이 폭발적으로 늘었다. **LLM의 메모리를 구조화된 지식 그래프(Knowledge Graph)로 변환**하는 것이다. 이 글에서는 현재 존재하는 주요 프로젝트, 논문, 그리고 온톨로지 구축 방법론을 체계적으로 정리한다.

## 왜 "평면 메모리"가 문제인가

대부분의 LLM 메모리 시스템은 키-값(key-value) 형태다. 예를 들어:

```
사용자 선호: Node.js가 편함
프로젝트 상태: 포시냥 부품 주문 완료
```

이 구조의 치명적 한계는 세 가지다:

1. **출처 추적 불가** — 이 정보를 언제, 어느 대화에서 알게 됐는지 모른다
2. **신뢰도 구분 불가** — 사용자가 직접 말한 사실과 AI가 추론한 해석이 같은 레벨에 있다
3. **관계 파악 불가** — 두 정보 사이의 인과관계나 패턴을 발견할 수 없다

이걸 해결하려면 메모리를 **노드(사실)와 엣지(관계)**로 구성된 그래프로 재구성해야 한다.

## 메이저 오픈소스 프로젝트 5선

### 1. Mem0 — 에이전트 메모리의 사실상 표준

[Mem0](https://github.com/mem0ai/mem0)는 현재 생태계에서 가장 성숙한 AI 메모리 레이어다. ⭐54,800개의 스타가 말해주듯, 커뮤니티 지지가 압도적이다.

대화에서 자동으로 메모리를 추출·저장·검색하며, 최근 그래프 기반 메모리를 지원하기 시작했다. MCP(Model Context Protocol) 플러그인으로 Claude Desktop, Cursor 등에 바로 연결할 수 있다.

**핵심 차별점:** 범용성. 특정 도메인에 국한되지 않고 모든 AI 에이전트에 메모리 레이어를 제공한다.

### 2. Microsoft GraphRAG — 문서 이해의 game changer

[GraphRAG](https://github.com/microsoft/graphrag)는 마이크로소프트가 만든 그래프 기반 RAG 시스템으로 ⭐32,800개의 스타를 보유하고 있다.

비정형 텍스트에서 자동으로 엔티티와 관계를 추출해 지식 그래프를 구축한다. 커뮤니티 감지(Community Detection) 알고리즘으로 자동으로 주제별 클러스터를 형성하고, 글로벌/로컬 검색 모드를 제공한다. v3까지 발전했다.

**한계:** 개인 메모리보다는 문서 분석에 특화되어 있다.

### 3. Graphiti (Zep) — 시간이 흐르는 지식 그래프

[Graphiti](https://github.com/getzep/graphiti)는 ⭐25,700스타로, **시간적(temporal) 지식 그래프** 구축에 특화되어 있다. 에피소드(대화나 이벤트)에서 자동으로 엔티티와 관계를 추출하고, Neo4j에 저장한다.

가장 흥미로운 점은 **에피소드 메모리 → 의미 메모리** 변환이 자동으로 이루어진다는 것. "어제 대화에서 사용자가 이직을 고민한다"는 에피소드가 "사용자는 현재 경력 전환기를 맞이하고 있다"는 의미로 압축된다. 중복 제거와 병합도 자동 처리된다.

**핵심 차별점:** 시간 축. 정보가 언제 생성되고 언제 만료되는지 추적한다.

### 4. Letta (구 MemGPT) — LLM을 운영체제처럼

[Letta](https://github.com/letta-ai/letta)는 ⭐22,400스타로, LLM이 스스로 메모리를 관리하는 혁신적 아키텍처를 제안한다.

컨텍스트 윈도우를 넘어서는 장기 메모리를 위해 **코어 메모리 / 아카이브 메모리 / 리콜 메모리**의 3계층 구조를 사용한다. 마치 운영체제의 RAM/디스크/캐시 계층과 유사하다.

**핵심 차별점:** LLM 스스로가 무엇을 기억하고 무엇을 잊을지 결정하는 자율 메모리 관리.

### 5. Cognee — 6줄 코드로 KG 파이프라인

[Cognee](https://github.com/topoteretes/cognee)는 ⭐17,000스타로, 비정형 데이터를 지식 그래프로 변환하는 파이프라인을 단 6줄의 코드로 구현할 수 있다.

```python
import cognee

await cognee.add("data://example.txt")
await cognee.cognify()
results = await cognee.search("QUERY", "GRAPH_COMPLETION")
```

Neo4j, 버터DB 등 다양한 그래프 스토어를 지원하고 MCP 서버도 포함되어 있다.

## 학술 연구에 가까운 프로젝트들

### HippoRAG — 해마 기반 RAG+KG

[HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG) (⭐3,500, NeurIPS 2024)는 인간 해마의 장기 기억 메커니즘에서 영감을 받은 프레임워크다.

문서에서 지식 그래프를 지속적으로 구축하고, **Personalized PageRank**로 그래프를 검색한다. 다중 홉(multi-hop) 추론에 특히 강점이 있다. "A는 B를 알고, B는 C에서 일한다 → A와 C의 관계는?" 같은 질문에 기존 RAG보다 훨씬 정확하다.

HippoRAG 2에서는 개인화와 시간 정보가 추가되었다.

### MCP Knowledge Graph — Claude 전용 로컬 KG

[MCP Knowledge Graph](https://github.com/shaneholloman/mcp-knowledge-graph) (⭐848)는 외부 DB 없이 로컬 파일 기반으로 동작하는 경량 지식 그래프다. 엔티티-관계-관찰(observation)의 3층 구조를 사용한다.

### Memento MCP — Neo4j + 시간적 엔티티

[Memento MCP](https://github.com/gannonh/memento-mcp) (⭐418)는 Neo4j 기반 MCP 서버로, 시간적 엔티티(temporal entities)를 지원한다. Claude Desktop이나 Cursor에서 바로 사용할 수 있다.

## 온톨로지 — "어떻게 구조화할 것인가"의 설계도

여기까지는 "저장소"에 대한 이야기였다. 그렇다면 **무엇을 어떻게 저장할지의 규칙**, 즉 온톨로지(ontology)는 어떻게 만들까?

### KNOW 온톨로지 — LLM용 실세계 지식 모델

[KNOW: A Real-World Ontology for Knowledge Capture with Large Language Models](https://arxiv.org/abs/2405.19877) (2024)는 LLM이 개인 정보를 체계적으로 캡처하기 위해 설계된 온톨로지다.

사람, 장소, 이벤트, 조직 등 일상 지식을 모델링하며, 12개 프로그래밍 언어용 코드 생성 라이브러리를 제공한다. **개인 AI 어시스턴트용으로 설계되었다는 점**이 핵심이다.

### ndoli — RDF/OWL/SHACL 기반 개인 KG

[ndoli](https://github.com/SteveHedden/ndoli)는 W3C 표준인 RDF, OWL, SHACL을 사용해 개인 지식 그래프를 구축하는 프로젝트다. schema.org 온톨로지를 기반으로 하며, Claude Code와 통합되어 작동한다.

### 자동 온톨로지 생성 — LLM이 온톨로지를 만든다

[automatic-KG-creation-with-LLM](https://github.com/fusion-jena/automatic-KG-creation-with-LLM) (⭐341)은 가장 인기 있는 자동 온톨로지 생성 프로젝트다. NER(개체명 인식) → 온톨로지 생성 → 지식 그래프 구축 → 평가의 전체 파이프라인을 제공한다.

논문 [Automatic Ontology Construction Using LLMs as an External Layer of Memory, Verification, and Planning for Hybrid Intelligent Systems](https://arxiv.org/abs/2604.20795) (2026)는 RDF/OWL 기반 지식 그래프 구축 자동화 파이프라인을 제안한다. NER → 관계 추출 → 정규화 → triple 생성 → **SHACL/OWL 검증** → 지속적 업데이트의 완전한 파이프라인이다.

## 반드시 읽어야 할 핵심 논문

### 1. Generative Agents (Stanford, 2023)

[Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) — LLM 기반 에이전트가 인간 같은 행동을 시뮬레이션하는 이정표 논문. **관찰 → 반성 → 계획**의 3단계 메모리 구조를 제안했다. 25개의 에이전트가 자율적으로 발렌타인데이 파티를 기획하는 실험은 유명하다.

### 2. MemGPT (UC Berkeley, 2023)

[MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — LLM을 운영체제처럼 다루는 혁신적 접근. 가상 컨텍스트 관리 기법으로 제한된 컨텍스트 윈도우를 극복한다. 현재 Letta 오픈소스 프로젝트로 발전했다.

### 3. The AI Hippocampus (2026 서베이)

[The AI Hippocampus: How Far are We From Human Memory?](https://arxiv.org/abs/2601.09113) — LLM 메모리 메커니즘의 **가장 포괄적인 서베이**. 암묵적 메모리 → 명시적 메모리 → 에이전트 메모리의 3분류 체계를 제안한다.

### 4. PKG 생태계 서베이 (2023)

[An Ecosystem for Personal Knowledge Graphs: A Survey and Research Roadmap](https://arxiv.org/abs/2304.09572) — PKG 연구의 바이블. PKG의 정의, 구축 방법론, 응용 분야, 연구 로드맵을 체계적으로 정리했다.

### 5. GAAMA (2026)

[GAAMA: Graph Augmented Associative Memory for Agents](https://arxiv.org/abs/2603.27910) — Generative Agents의 메모리 스트림을 그래프 구조로 확장. 다중 세션 대화에서 영속적 장기 메모리를 유지한다.

### 6. 메모리 보안 서베이 (2026)

[A Survey on the Security of Long-Term Memory in LLM Agents: Toward Mnemonic Sovereignty](https://arxiv.org/abs/2604.16548) — **"기억 주권(Mnemonic Sovereignty)"**이라는 개념을 제안하며, 공격자가 에이전트의 메모리를 조작하는 시나리오를 분석한다. 63페이지 분량의 심층 연구.

## 포지셔닝 — 어디에 초점을 맞출 것인가

전체 생태계를 놓고 보면, 각 프로젝트가 해결하려는 문제가 다르다:

- **저장소 중심:** Mem0, Cognee, MCP Knowledge Graph — "어떻게 저장할까"
- **이해 중심:** GraphRAG, HippoRAG — "어떻게 검색·추론할까"
- **아키텍처 중심:** Letta/MemGPT — "어떻게 관리할까"
- **신뢰도 중심:** Know Thyself, KNOW 온톨로지 — "이 정보가 왜 믿을 수 있는가"

이 중에서 **개인 AI 비서**에 가장 적합한 조합은:

1. **Graphiti**로 시간적 KG 기반 저장
2. **KNOW 온톨로지**로 개인 정보 타입 정의
3. **HippoRAG**의 PageRank로 그래프 검색
4. **Know Thyself의 provenance + confidence tier**로 신뢰도 추적

이 네 가지를 결합하면, 기존 어느 단일 프로젝트보다 강력한 개인 메모리 시스템을 구축할 수 있다.

## 결론

LLM 메모리를 단순한 텍스트에서 구조화된 그래프로 변환하는 기술은 2024~2026년 사이 급격히 성숙했다. Mem0, Graphiti, Letta 같은 프로젝트는 이미 프로덕션 수준이며, HippoRAG, KNOW 온톨로지 같은 연구成果는 학술적 기반을 제공한다.

다음 단계는 이것들을 **실제로 통합하는 것**이다. 오픈소스 조각들은 있지만, 이걸 end-to-end로 묶어 개인 AI 비서에 적용한 사례는 아직 드물다. 바로 여기가 기회다.

---

**참고 링크 모음:**

- [Mem0](https://github.com/mem0ai/mem0)
- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
- [Graphiti (Zep)](https://github.com/getzep/graphiti)
- [Letta (MemGPT)](https://github.com/letta-ai/letta)
- [Cognee](https://github.com/topoteretes/cognee)
- [HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG)
- [MCP Knowledge Graph](https://github.com/shaneholloman/mcp-knowledge-graph)
- [Memento MCP](https://github.com/gannonh/memento-mcp)
- [Know Thyself](https://github.com/parrik/know-thyself)
- [ndoli (RDF/OWL 개인 KG)](https://github.com/SteveHedson/ndoli)
- [automatic-KG-creation-with-LLM](https://github.com/fusion-jena/automatic-KG-creation-with-LLM)
- [KNOW 온톨로지 논문](https://arxiv.org/abs/2405.19877)
- [Generative Agents 논문](https://arxiv.org/abs/2304.03442)
- [MemGPT 논문](https://arxiv.org/abs/2310.08560)
- [The AI Hippocampus 서베이](https://arxiv.org/abs/2601.09113)
- [PKG 생태계 서베이](https://arxiv.org/abs/2304.09572)
- [GAAMA 논문](https://arxiv.org/abs/2603.27910)
- [메모리 보안 서베이](https://arxiv.org/abs/2604.16548)
- [LLM 자동 온톨로지 구축 논문](https://arxiv.org/abs/2604.20795)
