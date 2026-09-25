---
title: "GPT-4 심판은 진짜 80% 맞출까 — LLM-as-a-Judge 편향 4종과 OTel로 심사관까지 감사하는 법"
date: "2026-09-25"
keywords: ["LLM-as-a-Judge", "에이전트 평가", "G-Eval", "MT-Bench", "OpenTelemetry", "LLM 평가"]
lang: "ko"
description: "LLM 심판의 인간 일치율 80%라는 숫자의 실제 의미, 검증된 편향 4종과 완화법, 그리고 심판 호출 자체를 OpenTelemetry GenAI 트레이싱으로 감사하는 실전 구성을 다룬다."
---

# GPT-4 심판은 진짜 80% 맞출까 — LLM-as-a-Judge 편향 4종과 OTel로 심사관까지 감사하는 법

LLM 애플리케이션을 운영하다 보면 가장 난감한 질문에 부딪힌다. "이번에 프롬프트 바꿨는데, 좋아진 게 맞나?" 사람이 100개 응답을 일일이 읽고 점수를 매기는 건 현실적으로 불가능하고, BLEU나 ROUGE 같은 전통 지표는 "의미가 맞는지"는 전혀 재지 못한다. 그래서 등장한 것이 LLM-as-a-Judge, 즉 또 다른 LLM에게 채점을 맡기는 방식이다.

문제는 심판이 LLM이라는 점이다. 심판도 환각을 하고, 편향을 가지고, 비용을 발생시킨다. 이 글에서는 (1) "GPT-4 심판이 인간과 80% 일치한다"는 유명한 주장이 정확히 무엇을 뜻하는지, (2) 논문으로 검증된 편향 4종과 완화법, (3) 심판 호출 자체를 OpenTelemetry로 관측해 "심판의 실수"를 프로덕션 장애처럼 추적하는 구성까지 다룬다.

## 1. "80% 일치"의 정확한 의미를 먼저 파고들기

이 숫자의 출처는 Zheng 등의 NeurIPS 2023 논문 "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"(arXiv 2306.05685)다. 논문 초록은 GPT-4 심판이 통제된 환경의 인간 선호 및 크라우드소싱 인간 선호와 모두 잘 일치하며 **80%를 넘는 일치율**을 달성했고, 이는 **인간 간 일치율과 같은 수준**이라고 명시한다.

여기서 흔히 놓치는 뉘앙스가 두 가지 있다.

- **일치의 단위는 페어와이즈 승부다.** 두 응답 중 어느 쪽이 나은지 고르는 비교 판정에서 80%이지, 10점 만점 점수를 소수점까지 정확히 맞춘다는 뜻이 아니다. 동전 던지기가 50%라는 점을 감안하면 80%는 유의미하지만 "심판이 인간을 대체한다"는 해석은 과하다.
- **심판 모델은 당대 최강 모델이었다.** 약한 모델로 심판을 세우면 일치율은 급격히 떨어진다. 논문 자체가 "강한 LLM 심판(strong LLM judges)"이라고 한정해서 말한다.

단일 출력 채점 방향의 근거는 G-Eval(Liu 등, EMNLP 2023, arXiv 2303.16634)에서 나온다. GPT-4를 백본으로 CoT(Chain-of-Thought)와 폼 필링을 결합한 G-Eval는 요약 태스크에서 인간 평가와 **Spearman 상관계수 0.514**를 기록해 기존 방법들을 크게 앞섰다. 요약이라는 주관적 태스크에서 0.514가 "인간 수준"은 아니지만, 운영 관점에서 회귀 테스트의 신호로는 충분히 쓸 만한 수준이라는 게 이후 업계의 실전 합의다.

즉 정리하면: **비교 판정에는 강한 심판을, 절대 채점에는 루브릭 기반 CoT를** 쓰는 것이 검증된 출발점이다.

## 2. 논문이 밝힌 편향 4종 — 알고 쓰면 절반은 막힌다

MT-Bench 논문은 LLM 심판의 한계로 포지션, 장황함(verbosity), 자기강화(self-enhancement) 편향을 정식으로 다뤘고, 이후 연구(arXiv 2410.21819 등)는 자기선호(self-preference) 편향을 추가로 측정했다.

**① 포지션 편향 (Position Bias)** — 두 응답을 비교할 때 답변 순서에 따라 승자가 바뀐다. 완화법은 단순하다. A-B, B-A 양방향으로 두 번 심판하고 결과가 다르면 무승부로 처리하는 것이다. 심판 호출 비용이 2배가 되지만, 비교 평가의 신뢰도가 가장 확실하게 오른다.

**② 장황함 편향 (Verbosity Bias)** - 같은 내용이라도 길고 자세한 응답에 후한 점수를 준다. 루브릭에 "간결성" 항목을 명시적으로 넣고, 채점 기준에 "길이 자체는 품질이 아니다"를 문자로 박아넣어야 한다.

**③ 자기강화/자기선호 편향 (Self-Enhancement / Self-Preference)** — 심판 모델이 자기 계열 모델의 출력을 선호하는 경향. 같은 회사 모델끼리 심판-피심사자 관계가 되면 점수가 인플레이션된다. 실무에서는 **피심사 모델과 다른 계열의 심판**을 고르는 것이 정석이다.

**④ 제한된 추론 능력** — 심판도 틀린다. 특히 정답이 필요한 수학·코드 채점에서 심판이 그럴듯한 오답을 정답으로 인정하는 사례가 보고됐다. 정답이 존재하는 영역은 exact match나 유닛 테스트가 1차이고, 심판은 그 다음 2차 필터로 쓰는 게 맞다.

## 3. 실전 구성 1: 루브릭 기반 심판 만들기

G-Eval의 핵심 아이디어는 두 가지다. (1) 사람이 쓴 채점 기준을 프롬프트로 주고, (2) 점수 근거를 먼저 생성(CoT)하게 한 뒤 점수를 뽑는 것. 이 구조를 최소한의 코드로 옮기면 다음과 같다.

```python
import json
from openai import OpenAI

client = OpenAI()

RUBRIC = """당신은 고객지원 응답 품질 심사관입니다.
다음 기준으로 1~5점을 매기세요.
1. 정확성: 환불 정책(30일 전액 환불) 사실과 일치하는가
2. 간결성: 불필요한 군더더기 없이 3문장 이내인가
3. 어조: 정중하고 브랜드 보이스에 맞는가
먼저 각 항목별 근거를 쓰고, 마지막에 JSON으로 출력하세요:
{"accuracy": 1-5, "conciseness": 1-5, "tone": 1-5, "reason": "..."}"""

def judge(question: str, answer: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4.1",  # 피심사 모델보다 강하고 다른 계열 권장
        messages=[
            {"role": "system", "content": RUBRIC},
            {"role": "user", "content": f"[질문]\n{question}\n\n[응답]\n{answer}"},
        ],
        response_format={"type": "json_object"},
    )
    return json.loads(resp.choices[0].message.content)

score = judge("신발이 안 맞는데 환불 가능한가요?",
              "30일 이내 전액 환불해 드립니다. 신발 카테고리만 가능합니다.")
print(score["accuracy"], score["conciseness"], score["tone"])
```

포지션 편향까지 막으려면 비교 심판을 양방향으로 돌리면 된다.

```python
def judge_pairwise(judge_model: str, q: str, a: str, b: str) -> str:
    def ask(first, second):
        resp = client.chat.completions.create(
            model=judge_model,
            messages=[{"role": "user", "content":
                f"[질문]{q}\n[응답A]{first}\n[응답B]{second}\n"
                "더 나은 응답 하나만 'A' 또는 'B'로 답하라."}],
        )
        return resp.choices[0].message.content.strip()
    r1, r2 = ask(a, b), ask(b, a)   # B-A 순서에서는 답문자를 뒤집어 해석
    r2 = {"A": "B", "B": "A"}.get(r2, r2)
    return r1 if r1 == r2 else "TIE"  # 양방향 불일치 = 무효 처리
```

## 4. 실전 구성 2: 심판 호출도 LLM 호출이다 — OTel로 묶어 보기

여기서부터가 이 글의 핵심 주장이다. **심판 호출은 애플리케이션의 다른 LLM 호출과 똑같은 비용·지연시간·실패 확률을 가진 프로덕션 호출이다.** 심판이 타임아웃해서 회귀 테스트가 조용히 스킵되거나, 심판 프롬프트 변경으로 점수 분포가 미끄러지는 일은 "평가의 문제"가 아니라 "장애"다. 그래서 관측 대상에 심판을 포함시켜야 한다.

OpenTelemetry의 GenAI 시맨틱 컨벤션은 이제 LLM 호출의 모델명, 토큰 수, 지연시간을 표준 속성으로 기록한다. 공식 문서에 따르면 스팬 속성 `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, 메트릭 `gen_ai.client.operation.duration` 등이 정의되어 있고, 토큰 사용량 메트릭은 공식 블로그 시점에 `gen_ai.client.token.usage`로 안내됐지만 최신 컨벤션 리포(semantic-conventions-genai)에서는 `gen_ai.client.inference.usage.*` 계열로 이름이 이동하는 중이니 도입 시점의 리포를 확인해야 한다. 또한 VS Code Copilot, OpenAI Codex, Claude Code 같은 도구들도 이미 OTel 내보내기를 지원한다.

파이썬 앱에서 심판까지 포함해 계측하는 최소 구성은 이렇다.

```python
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider(resource=Resource.create({"service.name": "my-llm-app"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

def judge_traced(question: str, answer: str) -> dict:
    with tracer.start_as_current_span("llm-judge.geval") as span:
        span.set_attribute("gen_ai.system", "openai")
        span.set_attribute("gen_ai.request.model", "gpt-4.1")
        span.set_attribute("judge.rubric_version", "v3")  # 커스텀 속성
        result = judge(question, answer)                  # 위의 judge()
        span.set_attribute("judge.score.accuracy", result["accuracy"])
        return result
```

이렇게 해두면 어떤 질문에 심판 점수가 급락했는지, 심판 호출 지연이 회귀 테스트 총 소요시간의 병목인지, 루브릭 v2에서 v3로 바꾼 뒤 점수 분포가 어떻게 이동했는지를 일반 APM 대시보드에서 그대로 볼 수 있다. Aspire Dashboard(무료, 오픈소스, Docker로 단일 컨테이너 실행) 같은 OTLP 호환 뷰어면 별도 클라우드 계약 없이도 로컬에서 바로 확인 가능하다.

핵심 설계 원칙은 이것이다: **피심사 호출과 심판 호출을 같은 트레이스 트리에 두되, 심판 스팬은 별도 이름과 커스텀 속성(rubric 버전, 판정 유형)으로 구분하라.** 그래야 "응답 품질 저하"와 "심판 고장"을 로그에서 한눈에 구분할 수 있다.

> 참고: 위 코드의 속성명은 OTel 공식 블로그(2026년 5월) 기준이며, GenAI 시맨틱 컨벤션은 현재 별도 리포(semantic-conventions-genai)에서 활발히 개정 중이다. 도입 시점에 최신 속성명을 리포에서 직접 확인하라.

## 5. 운영자를 위한 체크리스트

- **심판은 강하게, 계열은 다르게.** 피심사 모델과 같은 계열이면 자기선호 편향 위험이 커진다.
- **비교는 양방향, 채점은 루브릭+CoT.** MT-Bench와 G-Eval이 각각 검증한 표준 기법이다.
- **정답 있는 문제는 심판 앞세우지 마라.** 테스트·exact match가 1차, 심판은 주관 품질의 2차.
- **심판 프롬프트도 버전 관리하라.** 루브릭 변경은 점수 분포를 바꾼다. OTel 커스텀 속성에 버전을 남겨라.
- **심판 실패는 알림 대상이다.** 타임아웃·JSON 파싱 실패가 쌓이면 회귀 테스트가 무의미해진다.

## 결론

- "GPT-4 심판 80% 일치"는 페어와이즈 비교 판정 기준이며, 인간 간 일치율과 동급이라는 게 논문의 정확한 주장이다.
- 절대 채점은 G-Eval 스타일(루브릭 + CoT + 구조화 출력)이 검증된 기본값이고, 요약 태스크 인간 상관계수 0.514가 그 근거다.
- 포지션·장황함·자기선호·추론 한계의 편향 4종은 알려진 완화법(양방향 비교, 간결성 루브릭, 타 계열 심판, 정답 우선)이 존재한다.
- 심판 호출은 프로덕션 LLM 호출이다. OTel GenAI 컨벤션으로 피심사·심판을 한 트리에 계측하면 "품질 저하"와 "심판 고장"을 구분할 수 있다.

당장 해볼 첫 단계: 기존 평가 스크립트의 심판 함수 하나에 스팬을 감싸고 루브릭 버전을 속성으로 남기는 것. 오늘 소개한 코드 20줄이면 충분하다.
