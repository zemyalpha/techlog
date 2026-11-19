---
title: "LLM은 어제 대화를 기억하지 못한다 — LongMemEval이 밝힌 30~60% 성능 붕괴와 에이전트 메모리 설계 법칙 4가지"
date: "2026-11-19"
keywords: ["에이전트 메모리", "LongMemEval", "LoCoMo", "LangGraph", "AI 에이전트", "장기 기억"]
lang: "ko"
description: "1.5M 토큰 대화 이력에서 롱컨텍스트 LLM조차 30~60% 성능이 무너진다는 LongMemEval 논문의 실험 결과와, 인덱싱·검색·리딩 3단계 메모리 설계 법칙, LangGraph 체크포인터 실전 코드까지 정리한다."
---

# LLM은 어제 대화를 기억하지 못한다 — LongMemEval이 밝힌 30~60% 성능 붕괴와 에이전트 메모리 설계 법칙 4가지

"컨텍스트 윈도우가 100만 토큰인데 굳이 메모리 시스템이 필요한가?" — 2026년 말, 이 질문에 대한 가장 정확한 답은 벤치마크 논문에 나와 있다. **롱컨텍스트 LLM조차 긴 상호작용 이력에서는 30~60% 성능이 하락한다**는 것이 실측 데이터로 확인됐기 때문이다([LongMemEval, arXiv 2410.10813](https://arxiv.org/abs/2410.10813)).

이 글에서는 LoCoMo와 LongMemEval 두 벤치마크가 실제로 측정한 것, 논문이 발견한 메모리 설계 법칙 4가지, 그리고 이를 오늘 바로 적용할 수 있는 코드(LLM API 메모리 도구 + LangGraph 체크포인터)까지 정리한다. 메모리 프레임워크 간 비교 평가가 아니라, "왜 필요한가"와 "어떻게 설계하는가"에 초점을 맞췄다.

## 1. 문제의 크기: 벤치마크가 실측한 '기억력 붕괴'

### LoCoMo — 35개 세션짜리 대화에서 인간을 따라잡지 못하는 모델들

[LoCoMo](https://arxiv.org/abs/2402.17753)(Snap Research·UNC·USC, TMLR 게재)는 "매우 긴 대화"를 평가하기 위해 만든 데이터셋이다. 페르소나와 시간 순 이벤트 그래프를 부여한 두 LLM 에이전트가 대화를 생성하고, 사람 주석자가 장기 일관성을 검수하는 파이프라인으로 만들어졌다. 규모는 이렇다.

- 각 대화 **평균 300턴, 약 9K 토큰**, 최대 **35개 세션**에 걸친 이력
- 평가 과제는 질의응답, 이벤트 요약, 멀티모달 대화 생성 3종
- 질문은 단일 홉/다중 홉/시간 추론 등 여러 추론 유형으로 분류

논문의 결론은 냉정하다. 롱컨텍스트 LLM이나 RAG로 성능을 끌어올릴 수는 있지만, **인간 수준에는 크게 미치지 못한다**는 것이다. 5개 세션 이하 대화만 다루던 기존 벤치마크로는 이 문제가 보이지 않았다.

### LongMemEval — 롱컨텍스트 모델이 30~60% 무너지는 지점

[LongMemEval](https://arxiv.org/abs/2410.10813)(Tencent AI Lab·UCLA·UCSD, ICLR 2025)은 챗 어시스턴트의 장기 기억을 정밀하게 측정한다. 핵심 수치는 다음과 같다(모두 논문 및 [프로젝트 페이지](https://xiaowu0162.github.io/long-mem-eval/)에 명시된 값이다).

- **500개 질문**, 7개 유형으로 **5가지 능력** 측정: 정보 추출, 멀티세션 추론, 지식 갱신(knowledge update), 시간 추론, 기피 응답(abstention)
- **LongMemEval-S**: 질문당 약 115K 토큰(30~40세션) 이력 / **LongMemEval-M**: 약 500세션(~1.5M 토큰) 이력
- **롱컨텍스트 LLM이 LongMemEval-S에서 30~60% 성능 하락**
- GPT-4o 같은 상용 시스템도 LongMemEval-S보다 훨씬 쉬운 세팅에서 **30~70% 정확도**에 그침

특히 주목할 유형은 **지식 갱신**과 **시간 추론**이다. 사용자가 "3월에는 다이어트 중이라더니 6월에는 베이커리를 열었다"처럼 정보를 바꾸거나, "작년에" "지난주에" 같은 시점을 묻는 질문에서 단순 유사도 검색 기반 메모리가 반복적으로 무너진다. "컨텍스트에 다 넣으면 된다"는 전략은 비용 이전에 정확도에서 이미 실패하는 셈이다.

참고로 메모리 인프라 업체 Mem0는 자체 보고로 LoCoMo 92.5점, LongMemEval 94.4점을 주장하지만([Mem0 블로그](https://mem0.ai/blog/state-of-ai-agent-memory-2026)), 이는 **벤더 자체 측정치**라는 점을 감안해야 한다. 벤치마크 채점 방식(모델, 검색 깊이, 채점 LLM)에 따라 독립 평가 결과는 크게 달라질 수 있다.

## 2. 설계 법칙 4가지 — 논문이 실험으로 증명한 것

LongMemEval 논문의 진짜 가치는 벤치마크 자체가 아니라, 메모리 시스템을 **인덱싱 → 검색 → 리딩** 3단계로 분해하고 각 단계의 설계 선택을 통제 실험한 데 있다. 4가지 핵심 발견을 법칙으로 정리하면 다음과 같다.

**법칙 1 — 저장 단위는 '세션'도 '팩트'도 아닌 '라운드'다.**
대화 전체 세션을 통째로 저장하면 검색이 흐려지고, 반대로 개별 사실로 압축하면 정보가 손실된다. 논문 실험에서는 **라운드(질문-응답 한 쌍) 단위**가 전반적으로 가장 좋았고, 팩트 압축은 다중 세션 추론에는 도움이 되지만 전체 성능은 오히려 떨어뜨렸다.

**법칙 2 — 검색 키에 '사용자 팩트'를 덧붙여라.**
메모리 원문 자체를 임베딩 키로 쓰는 플랫 인덱스가 강력한 기본값이지만, 여기에 **추출된 사용자 사실로 키를 확장**하면 recall@k가 +4%, 다운스트림 QA 정확도가 +5% 올라간다. 대화 원문("그때 그랬었지")에는 없는 표면 키워드("다이어트", "베이커리")를 색인에 심어두는 효과다.

**법칙 3 — 시간 인덱싱과 시간 인지 쿼리 확장은 선택이 아니다.**
시간 추론 질문에서 단순 설계는 크게 실패한다. 저장 시 **타임스탬프를 인덱스에 포함**하고, 검색 시 질문의 시간 표현("작년에", "최근에")을 명시적 날짜 범위로 확장하면 시간 추론 관련 recall이 **7~11%** 개선된다.

**법칙 4 — 가장 강력한 모델도 메모리 없이는 1.5M 토큰 이력을 못 버틴다.**
LongMemEval-M 수준이 되면 롱컨텍스트도 답이 아니다. 끊임없이 자라는 이력에는 외부 메모리 메커니즘이 필수라는 것이 논문의 명시적 결론이다.

연구 프론티어는 한 걸음 더 나아가 있다. [MAGMA](https://arxiv.org/abs/2601.03236)(ACL 2026)는 메모리 항목을 **의미·시간·인과·엔티티 4종류의 직교 그래프**로 표현하고, 검색을 그래프 순회 정책으로 정식화해 LoCoMo·LongMemEval에서 기존 에이전트 메모리 시스템을 능가했다고 보고한다. 단일 유사도 색인이 한계에 도달했다는 신호로 읽힌다.

## 3. 실전 구현 ① — LLM API의 메모리 도구를 직접 다루기

플랫폼 벤더도 같은 방향으로 움직이고 있다. Anthropic의 Claude API는 [메모리 도구(memory tool)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)를 제공하는데, 설계가 앞의 법칙들과 정확히 맞닿아 있다.

- Claude가 태스크 시작 전 **메모리 디렉터리(`/memories`)를 자동으로 확인**하고, 학습한 내용을 파일로 기록했다가 이후 대화에서 다시 읽는다(just-in-time 검색).
- **클라이언트 측 방식**: Claude는 파일 연산을 "요청"만 하고, 실제 저장은 개발자 애플리케이션이 자기 인프라(사용자별 디렉터리, DB 키 등)에서 수행한다. 데이터가 벤더에 종속되지 않는 구조다.

애플리케이션 측 핸들러의 뼈대는 이렇게 시작한다(Python, 의사 구조):

```python
# memory_tool 요청을 받아 앱 쪽 저장소에서 실행하는 핸들러 예시
import json, pathlib

MEMORY_ROOT = pathlib.Path("./user_memory/alice")

def handle_memory_op(op: dict) -> str:
    # op 예: {"command": "view", "path": "/memories/preferences.md"}
    rel = op["path"].removeprefix("/memories/").lstrip("/")
    target = (MEMORY_ROOT / rel).resolve()

    # 보안: 메모리 루트 밖 탈출 차단 (공식 문서 권고 사항)
    if not str(target).startswith(str(MEMORY_ROOT.resolve())):
        return json.dumps({"error": "path outside memory root"})

    if op["command"] == "view":
        return target.read_text(encoding="utf-8") if target.exists() else ""
    if op["command"] == "write":
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(op["content"], encoding="utf-8")
        return "ok"
    return json.dumps({"error": f"unsupported: {op['command']}"})
```

여기에 법칙 1~3을 얹으면 이런 저장 포맷이 된다. 라운드 단위 저장 + 팩트 확장 키 + 타임스탬프 인덱스를 JSONL 하나로 구현한 예시다:

```python
import datetime, json

def make_memory_record(round_text: str, extracted_facts: list[str]) -> dict:
    return {
        "ts": datetime.datetime.now(datetime.timezone.utc).isoformat(),
        # 법칙 2: 원문 + 추출 팩트를 합쳐 임베딩 키로 사용
        "embed_input": round_text + "\n" + "\n".join(extracted_facts),
        "facts": extracted_facts,        # 예: ["사용자가 베이커리 창업 준비 중"]
        "raw": round_text,               # 법칙 1: 라운드 원문 보존 (압축 금지)
    }

def expand_query_time(question: str, now: datetime.datetime) -> dict:
    # 법칙 3: 시간 표현을 날짜 범위로 확장해 검색 범위 축소
    if "작년" in question:
        return {"gte": now.replace(year=now.year - 1).date().isoformat()}
    if "최근" in question:
        return {"gte": (now - datetime.timedelta(days=14)).date().isoformat()}
    return {}
```

## 4. 실전 구현 ② — LangGraph 체크포인터: 단기 vs 장기 메모리 분리

프레임워크 생태계에서도 메모리는 정식 계층이 됐다. LangChain의 구형 `ConversationBufferMemory` 계열은 **공식적으로 deprecated**됐고, 현재 지원되는 방식은 LangGraph의 두 축이다([공식 문서](https://docs.langchain.com/oss/python/langgraph/add-memory)).

- **단기 기억 = 체크포인터(checkpointer)**: 스레드 단위 대화 상태를 DB에 저장
- **장기 기억 = 스토어(store)**: 사용자/앱 단위 데이터를 세션을 넘어 보관

프로덕션 코드는 문서의 권장 패턴 그대로다:

```python
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:***@localhost:5432/postgres?sslmode=disable"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 최초 1회 필수

    def call_model(state: MessagesState):
        return {"messages": init_chat_model(model="claude-haiku-4-5").invoke(state["messages"])}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "user-alice"}}
    graph.invoke({"messages": [{"role": "user", "content": "나 베이커리 열 거야"}]}, config)
```

같은 `thread_id`로 다시 호출하면 이전 대화가 복원된다. 단기(스레드)와 장기(사용자)를 나누는 이 시점이 앞서 본 법칙 1(라운드 단위 저장)과 스토어의 팩트 확장(법칙 2)을 결합하기 좋은 지점이다.

## 5. 남은 어려운 문제들 — 튜토리얼이 말해주지 않는 것

3단계 설계를 적용해도 풀리지 않는 문제들이 여전히 있다. Mem0의 2026년 현황 보고가 지적하는 난제들은 실무자라면 공감할 목록이다: **시간 추론, 메모리 신선도(staleness), 세션 간 사용자 식별, 프라이버시와 삭제**. 특히 삭제는 "그 사용자의 데이터를 지웠는가"가 아니라 "그 사용자에 대해 *추론되어* 저장된 팩트까지 지웠는가"의 문제라 훨씬 까다롭다. 벤치마크 점수를 비교하기 전에, 내 서비스가 이 네 가지 중 무엇에 노출돼 있는지부터 따져보는 게 순서다.

## 결론

- 롱컨텍스트는 메모리의 대체재가 아니다. LongMemEval 실측상 롱컨텍스트 LLM도 115K 토큰 이력에서 30~60% 성능이 하락한다.
- 메모리는 인덱싱→검색→리딩 3단계로 설계하라. 저장은 라운드 단위, 키는 팩트로 확장, 검색은 시간 범위로 좁혀라 — 각각 논문 실험에서 +4~11%의 검증된 개선효과가 있다.
- 벤더 프리미티브(Anthropic 메모리 도구 등)와 LangGraph 체크포인터/스토어는 이미 프로덕션 수준이다. 직접 만들기 전에 이 계층 위에 얹는 것이 낫다.
- 벤더 자체 벤치마크 점수는 비교 근거로 삼지 마라. 채점 방식에 따라 순위가 쉽게 뒤집힌다.

첫 단계는 작게 시작하면 된다. 오늘 소개한 JSONL 저장 포맷(타임스탬프 + 팩트 확장 키 + 라운드 원문)과 LangGraph 체크포인터 세팅, 두 코드 블록을 자기 에이전트에 붙이는 것만으로 논문이 증명한 개선치의 대부분에 접근할 수 있다. 그다음은 사용자 이력이 실제로 115K 토큰을 넘는 시점에 LongMemEval-S 형식으로 자체 평가셋을 만들어보는 것 — 그때 비로소 내 메모리 시스템의 점수를 알게 된다.
