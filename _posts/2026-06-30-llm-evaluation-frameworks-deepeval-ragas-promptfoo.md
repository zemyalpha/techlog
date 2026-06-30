---
title: "LLM 평가 프레임워크 3종 비교: DeepEval vs RAGAS vs Promptfoo를 선택하는 기준"
date: "2026-06-30"
keywords: ["LLM evaluation", "DeepEval", "RAGAS", "Promptfoo", "AI testing"]
lang: "ko"
description: "DeepEval, RAGAS, Promptfoo 세 LLM 평가 프레임워크의 차이점과 선택 기준을 정리한다. CI 통합, RAG 평가, 레드티밍 각 분야에서 어느 도구가 적합한지 코드 예시와 함께 설명한다."
---

# LLM 평가 프레임워크 3종 비교: DeepEval vs RAGAS vs Promptfoo를 선택하는 기준

LLM 기반 애플리케이션이 프로덕션에 들어가는 2026년, 가장 많이 묻는 질문은 "어떤 모델을 쓸까"가 아니라 "어떻게 검증할까"입니다. 비결정적 출력을 내는 시스템에서 "입력 A에 대해 출력 B가 나온다"는 기존 단위 테스트의 기본 가정이 무너졌기 때문입니다. 같은 프롬프트에도 매번 다른 답이 나오고, 정답 여부가 분포(distribution)로 표현되는 환경에서는 새로운 종류의 테스트 도구가 필요합니다.

이 글에서는 현재 GitHub에서 가장 활발히 유지보수되는 세 개의 LLM 평가 프레임워크 — DeepEval, RAGAS, Promptfoo — 의 차이를 실제 코드 예시와 함께 비교합니다. 각 도구가 해결하는 문제가 다르기 때문에, "어느 것이 최고다"라는 정답은 없습니다. 대신 "어떤 상황에서 어느 도구를 써야 하는지"를 구체적으로 정리합니다.

## 핵심 전제: LLM 평가는 왜 기존 테스트와 다른가

전통적인 소프트웨어 테스트는 결정적(deterministic)입니다. `assert add(1, 2) == 3`은 항상 참입니다. 하지만 LLM 출력은 확률적입니다. 같은 질문에 대해 응답의 품질, 정확도, 어조가 매번 달라집니다. 이런 환경에서는 다음과 같은 새로운 검증 방식이 필요합니다.

**첫째, 메트릭 기반 평가입니다.** 정답/오답의 이분법 대신 충실도(faithfulness), 관련성(relevancy), 정확성(correctness) 같은 정량적 점수를 사용합니다. "이 응답이 검색된 컨텍스트에 기반하고 있는가?"를 0~1 사이의 점수로 측정합니다.

**둘째, LLM-as-judge 방식입니다.** 더 강력한 모델(예: GPT-4o)이 더 약한 모델의 출력을 평가합니다. 평가 자체가 또 다른 LLM 호출이기 때문에, 평가자 모델의 품질이 전체 시스템의 신뢰성을 좌우합니다.

**셋째, 골든 데이터셋(golden dataset)입니다.** 큐레이션된 입력-기대출력 쌍을 만들어 회귀 테스트에 사용합니다. 프롬프트나 모델을 바꿨을 때 이 데이터셋에 대한 점수가 떨어지지 않는지 확인하는 것이 핵심입니다.

이 세 가지 요구사항을 각 프레임워크가 다르게 접근합니다.

## DeepEval: pytest 안에서 동작하는 LLM 평가 라이브러리

DeepEval은 Confident AI가 개발한 오픈소스 프로젝트로, Apache-2.0 라이선스로 배포됩니다. 핵심 철학은 "LLM 평가를 개발자가 이미 사용하는 pytest 생태계 안으로 가져오는 것"입니다.

### 설치와 기본 사용법

```bash
pip install deepeval
```

DeepEval의 가장 큰 특징은 평가 코드가 일반 pytest 테스트와 동일한 구조를 가진다는 점입니다.

```python
from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import HallucinationMetric, AnswerRelevancyMetric

def test_rag_pipeline():
    # 평가 대상: RAG 시스템의 실제 출력
    test_case = LLMTestCase(
        input="RAGAS의 핵심 메트릭은?",
        actual_output="Faithfulness, Answer Relevancy, Context Precision, Context Recall, Answer Correctness",
        retrieval_context=["RAGAS는 5가지 표준 메트릭을 제공한다..."]
    )

    # 메트릭 정의
    hallucination = HallucinationMetric(threshold=0.5)
    relevancy = AnswerRelevancyMetric(threshold=0.7)

    # assert_test가 pytest assert 역할
    assert_test(test_case, [hallucination, relevancy])
```

이 코드는 `pytest test_rag.py`로 실행할 수 있으며, 결과는 일반 pytest 결과와 동일한 형식으로 표시됩니다. CI/CD 파이프라인에서 기존 테스트와 함께 실행되는 것이 핵심 장점입니다.

### DeepEval의 메트릭 구성

DeepEval이 제공하는 메트릭은 여러 종류가 있으며, 가장 자주 사용되는 것들은 다음과 같습니다.

| 메트릭 | 측정 대상 | 활용 시나리오 |
|--------|-----------|---------------|
| `HallucinationMetric` | 출력이 검색된 컨텍스트에 기반하는지 | RAG 환각 감지 |
| `AnswerRelevancyMetric` | 출력이 질문에 직접적으로 답하는지 | 챗봇 응답 품질 |
| `FaithfulnessMetric` | 출력이 참조 텍스트와 일치하는지 | 요약, 번역 |
| `ContextualPrecisionMetric` | 검색된 컨텍스트의 정밀도 | RAG 검색 품질 |
| `ContextualRecallMetric` | 필요한 컨텍스트를 모두 검색했는지 | RAG 검색 품질 |
| `BiasMetric` | 인구통계적 편향 여부 | 컴플라이언스 |
| `GEval` | 사용자 정의 평가 기준 (LLM-as-judge) | 맞춤형 기준 |

`GEval`은 특히 유용한데, 개발자가 자연어로 평가 기준을 작성하면 DeepEval이 이를 자동으로 LLM-as-judge 프롬프트로 변환합니다.

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

custom_metric = GEval(
    name="전문성",
    criteria="평가 대상 텍스트가 기술 블로그 독자에게 적절한 전문성 수준을 보이는지",
    evaluation_params=[LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.6
)
```

### DeepEval이 적합한 경우

- 기존 pytest 기반 CI/CD 파이프라인이 있는 팀
- RAG뿐 아니라 다양한 LLM 애플리케이션(챗봇, 요약, 분류 등)을 평가해야 하는 팀
- 코드로 평가 로직을 관리하고 버전 관리하려는 팀
- Confident AI의 상용 플랫폼으로 팀 단위 대시보드가 필요한 경우

## RAGAS: RAG 평가의 사실상 표준

RAGAS(Retrieval-Augmented Generation Assessment)는 Apache-2.0 라이선스로 배포되며, DeepEval이 "광범위한 LLM 평가"를 목표로 한다면, RAGAS는 **RAG 시스템 평가에 특화**되어 있습니다.

### RAGAS의 5대 표준 메트릭

RAGAS가 제공하는 5개 메트릭은 학계와 산업계에서 RAG 평가의 공통 어휘로 자리 잡았습니다. 2026년 기준 많은 RAG 관련 논문과 벤더 벤치마크가 이 메트릭 이름을 사용합니다.

| 메트릭 | 질문 | 계산 방식 |
|--------|------|-----------|
| **Faithfulness** | 답변이 검색된 컨텍스트에 근거하는가? | 답변의 각 문장이 컨텍스트에서 지지되는지 LLM이 판단 |
| **Answer Relevancy** | 답변이 질문에 직접적으로 답하는가? | 답변에서 역으로 질문을 생성해 원본 질문과의 유사도 측정 |
| **Context Precision** | 검색된 컨텍스트가 질문에 관련이 있는가? | 컨텍스트 순위의 정밀도 |
| **Context Recall** | 필요한 정보를 모두 검색했는가? | 정답에 필요한 정보가 컨텍스트에 포함되는지 |
| **Answer Correctness** | 답변이 정답과 일치하는가? | 의미적 유사도 + 사실 정확도 |

### RAGAS 사용법

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 평가 데이터셋 구성
eval_data = {
    "question": ["RAGAS의 Faithfulness 메트릭은 무엇인가?"],
    "answer": ["답변이 검색된 컨텍스트에 기반하는지 측정하는 메트릭입니다."],
    "contexts": [["Faithfulness는 RAGAS의 핵심 메트릭 중 하나로..."]],
    "ground_truth": ["Faithfulness는 RAG 답변이 검색된 컨텍스트에 근거하는지를 측정한다."]
}
dataset = Dataset.from_dict(eval_data)

# 평가 실행
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)
print(results)
# {'faithfulness': 0.92, 'answer_relevancy': 0.85, 'context_precision': 0.78, 'context_recall': 0.90}
```

RAGAS의 특징은 `ground_truth`(정답) 없이도 `faithfulness`와 `answer_relevancy`를 계산할 수 있다는 점입니다. 이는 실제 프로덕션 환경에서 큰 장점이 됩니다. 모든 질문에 대해 사람이 정답을 작성하는 것은 현실적으로 불가능하기 때문입니다.

### RAGAS가 적합한 경우

- RAG 파이프라인의 검색(retrieval) 품질을 정량적으로 측정해야 하는 경우
- `context_precision`/`context_recall`로 리랭킹(reranking) 모델의 효과를 비교해야 하는 경우
- RAG 논문을 작성하거나 벤치마크를 공표해야 하는 경우 (메트릭 이름이 학계 표준)
- 정답 라벨 없이 대규모 평가를 자동화해야 하는 경우

## Promptfoo: CI/CD를 위한 선언적 테스트

Promptfoo는 MIT 라이선스로 배포되며, 핵심 강점은 **YAML 기반 선언적 설정**과 **CLI 중심의 CI/CD 통합**입니다.

### Promptfoo의 차별점: 코드 없는 테스트 정의

DeepEval과 RAGAS가 Python 코드로 평가를 작성하는 반면, Promptfoo는 YAML 파일로 테스트를 정의합니다.

```yaml
# promptfooconfig.yaml
description: "RAG 응답 품질 회귀 테스트"

providers:
  - openai:gpt-4o
  - anthropic:claude-sonnet
  - ollama:llama3:8b

prompts:
  - "다음 컨텍스트를 바탕으로 질문에 답하라:\n{{context}}\n\n질문: {{question}}"

tests:
  - vars:
      context: "DeepEval은 Apache-2.0 라이선스 LLM 평가 프레임워크다."
      question: "DeepEval의 라이선스는?"
    assert:
      - type: contains
        value: "Apache"
      - type: llm-rubric
        value: "답변이 정확하고 간결하며, 컨텍스트에서만 정보를 도출한다"
      - type: latency
        threshold: 3000  # 3초 이내 응답

  - vars:
      context: "RAGAS는 RAG 평가에 특화된 도구다."
      question: "RAGAS의 주요 용도는?"
    assert:
      - type: llm-rubric
        value: "RAG 평가라는 용도를 명확히 설명한다"
```

이 YAML 파일 하나로 다음 작업이 가능합니다.

- **멀티 모델 비교**: GPT-4o, Claude, 로컬 Llama를 같은 프롬프트로 동시 테스트
- **assert 타입**: `contains`(문자열 포함), `llm-rubric`(LLM 판정), `latency`(응답 속도), `icontains`(대소문자 무시)
- **매트릭스 테스트**: 모델 × 프롬프트 × 테스트 케이스의 모든 조합을 자동 실행

### CLI 실행과 CI/CD 통합

```bash
# 평가 실행
promptfoo eval

# 웹 대시보드에서 결과 확인
promptfoo view

# JSON 결과 파일 출력 (CI에서 파싱)
promptfoo eval --output results.json
```

Promptfoo의 가장 강력한 기능 중 하나는 **레드티밍(red-teaming)**입니다. 프롬프트 인젝션, 탈옥(jailbreak), 민감 정보 유출 등의 공격 시나리오를 자동으로 생성하고 테스트합니다.

```bash
# 자동 레드팀 테스트 생성 및 실행
promptfoo redteam \
  --purpose "고객 지원 챗봇" \
  --plugins prompt-injection,jailbreak,pii \
  --output redteam.yaml
promptfoo eval redteam.yaml
```

### Promptfoo가 적합한 경우

- CI/CD 파이프라인에서 프롬프트 변경 시 자동 회귀 테스트를 실행해야 하는 경우
- 여러 모델을 동일한 테스트 셋으로 비교해야 하는 경우
- 보안 팀과 협업하여 레드티밍 테스트를 자동화해야 하는 경우
- Python 환경 없이 Node.js/CLI 환경에서 테스트를 실행해야 하는 경우

## 세 프레임워크 비교 요약

| 구분 | DeepEval | RAGAS | Promptfoo |
|------|----------|-------|-----------|
| **GitHub 활동** | 활발함 | 활발함 | 매우 활발함 |
| **라이선스** | Apache-2.0 | Apache-2.0 | MIT |
| **주 언어** | Python | Python | TypeScript/Node.js |
| **테스트 정의** | Python 코드 (pytest) | Python 코드 | YAML (선언적) |
| **강점** | 광범위한 메트릭, pytest 통합 | RAG 특화, 학계 표준 메트릭 | CI/CD 통합, 레드티밍, 멀티 모델 비교 |
| **적용 분야** | 일반 LLM 애플리케이션 | RAG 시스템 | 프롬프트 엔지니어링, 보안 테스트 |
| **관측성** | Confident AI 플랫폼 (상용) | 자체 대시보드 없음 | 내장 웹 UI |

## 실제 팀에서는 두 개를 같이 쓴다

가장 성숙한 AI 팀은 하나의 프레임워크만 사용하지 않습니다. 개발 시점 평가와 프로덕션 모니터링을 분리하여 두 개 이상을 병행하는 것이 일반적인 패턴입니다.

**개발 시점 평가 파이프라인 예시:**

```
Promptfoo (프롬프트 회귀 테스트)
       ↓
DeepEval (pytest 통합 단위/통합 평가)
       ↓
RAGAS (RAG 검색 품질 벤치마크)
       ↓
배포
       ↓
Arize Phoenix / LangSmith (프로덕션 관측성)
```

- Promptfoo로 프롬프트나 모델 변경 시 기존 응답 품질이 떨어지지 않는지 확인
- DeepEval로 개별 기능(요약, 분류, RAG 답변)의 pytest 기반 회귀 테스트
- RAGAS로 검색 파이프라인의 정밀도/재현율 추적
- 프로덕션에서는 Arize Phoenix나 LangSmith로 드리프트와 환각률을 지속 모니터링

## CI/CD에서 비결정적 출력을 다루는 팁

LLM 출력은 매 실행마다 달라지기 때문에, 그대로 CI에 넣으면 빌드가 불안정해집니다. 이를 방지하는 실전 팁은 다음과 같습니다.

**허용 오차(tolerance band)를 사용하라.** 정확한 임계값 대신 점수 구간을 설정합니다. `faithfulness > 0.75` 대신 `faithfulness가 0.70~0.80 구간이면 통과`로 설정하면, LLM 출력의 자연스러운 변동에 의한 빌드 실패를 줄일 수 있습니다.

**평가자 모델을 고정하라.** LLM-as-judge에 사용하는 모델을 버전까지 명시하여 고정합니다. 이렇게 하면 평가 결과의 변동을 줄이고 회귀 비교가 쉬워집니다.

**안정적인 골든 테스트셋을 유지하라.** 프로덕션에서 무작위로 샘플링한 테스트 케이스는 매번 바뀌기 때문에 회귀를 감지하기 어렵습니다. 50~200개 정도의 고정된 테스트 셋을 큐레이션하여 매번 같은 셋으로 평가해야 합니다.

## 결론: 선택의 기준

세 프레임워크는 경쟁 관계가 아니라 보완 관계입니다. 상황에 따른 선택 기준을 정리하면 다음과 같습니다.

- **RAG 시스템을 구축하고 있다면** → RAGAS로 시작하라. `context_precision`과 `context_recall`은 검색 품질을 정량화하는 가장 빠른 방법이다.
- **기존 pytest CI 파이프라인에 LLM 평가를 통합해야 한다면** → DeepEval을 선택하라. 기존 테스트 인프라를 그대로 활용할 수 있다.
- **프롬프트 변경 시 자동 회귀 테스트와 레드티밍이 필요하다면** → Promptfoo를 선택하라. YAML 기반 설정으로 코드 수정 없이 CI에서 실행할 수 있다.

처음 시작하는 팀이라면, 가장 먼저 30~50개의 골든 데이터셋을 만드는 것부터 시작하길 권합니다. 어떤 프레임워크를 쓰든, 좋은 평가의 출발점은 "무엇을 평가할 것인가"를 정의하는 골든 셋입니다. 도구는 그다음 문제입니다.
