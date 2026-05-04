---
title: "2026년 오픈소스 LLM 선택 가이드: 코딩·추론·자가호스팅 용도별 비교"
date: "2026-05-04"
keywords: ["오픈소스 LLM", "DeepSeek V4", "Llama 4", "Qwen 3.5", "Gemma 4", "GLM-5.1"]
lang: "ko"
description: "2026년 주요 오픈소스 LLM 5종(DeepSeek V4, Llama 4, Qwen 3.5, Gemma 4, GLM-5.1)을 코딩, 추론, 자가호스팅, 가격 기준으로 비교하고 용도별 선택 가이드를 제공합니다."
---

# 2026년 오픈소스 LLM 선택 가이드: 용도별로 어떤 모델을 쓸 것인가

2026년 4월, 오픈소스 AI 모델의 지형이 완전히 바뀌었다. 1년 전만 해도 GPT-4나 Claude를 대체할 만한 오픈 모델을 찾기 어려웠지만, 지금은 DeepSeek V4, Llama 4, Qwen 3.5, Gemma 4, GLM-5.1이 각각 다른 영역에서 상용 모델을 압도하거나 필적하는 성능을 보여주고 있다.

문제는 "가장 좋은 모델"이 하나가 아니라는 점이다. 코딩에 강한 모델, 추론에 뛰어난 모델, 가볍게 로컬에서 돌릴 수 있는 모델, 초장문 컨텍스트가 필요한 작업에 적합한 모델이 각각 다르다. 이 글에서는 각 모델의 실제 벤치마크 수치, 아키텍처 특징, 자가 호스팅 요구사항, API 가격을 비교하고, 용도별로 어떤 모델을 선택해야 하는지 실용적인 가이드를 제공한다.

## 1. 2026년 주요 오픈소스 LLM 한눈에 보기

비교 대상은 현재 가장 활발하게 사용되는 5개 모델 패밀리다.

| 모델 | 개발사 | 총 파라미터 | 활성 파라미터 | 라이선스 | 컨텍스트 |
|------|--------|------------|-------------|---------|---------|
| DeepSeek V4-Pro | DeepSeek | ~1.6조 | ~49B | MIT | 1M |
| Llama 4 Maverick | Meta | 400B | 17B | Llama 4 커뮤니티 | 1M |
| Llama 4 Scout | Meta | 109B | 17B | Llama 4 커뮤니티 | 10M |
| Qwen 3.5 (397B) | Alibaba | 397B | 17B | Apache 2.0 | 128K |
| Gemma 4 (31B) | Google | 31B | 31B (dense) | Apache 2.0 | 256K |
| GLM-5.1 | Zhipu AI | 754B | ~45B | MIT | 200K |

여기서 주목할 점이 있다. 대부분의 모델이 **Mixture-of-Experts(MoE)** 아키텍처를 채택했다는 것. 총 파라미터는 수백억~조 단위지만, 실제로 각 토큰 처리에 활성화되는 파라미터는 17B~45B 수준이다. 이 덕분에 과거보다 훨씬 적은 GPU 메모리로 대형 모델을 실행할 수 있다.

예외는 Gemma 4 31B. 이 모델은 dense(밀집) 구조를 유지하면서도 31B 파라미터로 400B+ MoE 모델들과 경쟁하는 성능을 보여준다.

## 2. 코딩 벤치마크: SWE-bench, HumanEval 기준

개발자에게 가장 중요한 지표는 코딩 능력이다. 실제 소프트웨어 엔지니어링 작업을 평가하는 SWE-bench Verified 기준으로 보자.

| 모델 | SWE-bench Verified | LiveCodeBench v6 | HumanEval |
|------|-------------------|-----------------|-----------|
| DeepSeek V4-Pro | 83.7%* | — | 90.0% |
| DeepSeek V4-Pro (독립검증) | 80.6% | 93.5% | — |
| GLM-5.1 | ~78% | — | — |
| Qwen 3.5 (35B-A3B) | 73.4% | 80.4% | — |
| Gemma 4 31B | 52.0% | 80.0% | — |
| Llama 4 Maverick | ~65% | — | 82.4% |

*DeepSeek 자체 발표 수치. 독립 검증에서는 80.6%로 측정됨.

코딩 분야에서 DeepSeek V4-Pro가 확실히 선두다. SWE-bench Verified에서 80.6~83.7%, HumanEval에서 90%를 기록했다. 특히 주목할 것은 LiveCodeBench v6에서 93.5%를 기록했다는 점인데, 이는 Claude Opus 4.6의 88.8%를 상회하는 수치다. 가격은 API 기준 입력 $1.74/M, 출력 $3.48/M으로, Claude Opus 4.6의 입력 $5/M, 출력 $25/M과 비교하면 **약 3~7배 저렴**하다.

GLM-5.1은 SWE-bench Pro(더 어려운 난이도)에서 58.4%를 기록하며 이 부문 최고 점수를 보여준다. 복잡한 멀티스텝 코딩 작업에 강점이 있다.

Gemma 4 31B는 SWE-bench에서 상대적으로 낮은 52%를 보이지만, 31B 파라미터 모델임을 감안하면 인상적이다. 경량 코딩 보조 도구로는 충분히 실용적이다.

## 3. 추론 및 지식 벤치마크

코딩 외에 수학, 과학, 일반 지식 추론 능력도 중요하다.

| 모델 | GPQA Diamond | MMLU | AIME 2026 |
|------|-------------|------|-----------|
| DeepSeek V4-Pro | — | — | — |
| Gemma 4 31B | 84.3% | — | 89.2% |
| Llama 4 Maverick | — | 85.2% | — |
| Qwen 3.5 (397B) | — | — | — |

Gemma 4 31B가 AIME 2026(수학 경시대회)에서 89.2%를 기록하며 수학 추론에서 강력한 모습을 보인다. GPQA Diamond(대학원 수준 과학 문제)에서도 84.3%로 준수하다. 31B 모델이 이 정도 성능이라는 건 로컬 호스팅 관점에서 매우 매력적이다.

Llama 4 Maverick은 MMLU 85.2%로 일반 지식 분야에서 견고하다. 하지만 전문 추론 벤치마크에서는 DeepSeek V4나 GLM-5.1에 뒤처지는 경향이 있다.

## 4. 자가 호스팅: 실제 하드웨어 요구사항

직접 모델을 호스팅하려면 하드웨어가 핵심 제약이다. 양자화(Q4_K_M) 기준으로 정리했다.

| 모델 | Q4_K_M 크기 | 최소 RAM/VRAM | 추천 하드웨어 |
|------|-----------|-------------|-------------|
| Gemma 4 31B | ~18GB | 24GB VRAM | RTX 4090, Mac M2 Ultra |
| Qwen 3.5 (35B-A3B) | ~4GB | 8GB VRAM | RTX 4070, Mac M1 16GB |
| Llama 4 Scout | ~25GB | 32GB RAM | Mac M2 Max, 2×RTX 4070 |
| Llama 4 Maverick | ~60GB | 64GB+ RAM | 2×A100, Mac M2 Ultra |
| DeepSeek V4 | ~300GB+ | 멀티 GPU | 4~8×H100 클러스터 |
| GLM-5.1 | ~150GB+ | 멀티 GPU | 4×H100 클러스터 |

**가장 실용적인 선택**: Qwen 3.5 35B-A3B는 활성 파라미터가 3B에 불과해 **RTX 4070이나 Mac M1 16GB에서도 실행** 가능하다. 그러면서도 SWE-bench 73.4%, LiveCodeBench 80.4%를 기록하는 준수한 성능을 보여준다. 개인 개발자의 로컬 코딩 어시스턴트로는 최적의 선택이다.

Gemma 4 31B는 RTX 4090 한 장으로 실행 가능하면서 수학 추론 89.2%를 보여준다. 연구나 데이터 분석 작업에 활용하기 좋다.

DeepSeek V4나 GLM-5.1은 조직 단위에서 GPU 클러스터를 운영할 수 있어야 현실적이다.

### Ollama로 직접 실행해보기

```bash
# Qwen 3.5 — 가장 가벼운 옵션, 8GB VRAM에서도 가능
ollama pull qwen3:8b
ollama run qwen3:8b "파이썬으로 이진 탐색 트리를 구현해줘"

# Gemma 4 — 24GB VRAM 필요, 수학/추론에 강점
ollama pull gemma4:31b
ollama run gemma4:31b "n-queen 문제를 백트래킹으로 풀어줘"

# Llama 4 Scout — 10M 컨텍스트, 32GB RAM
ollama pull llama4-scout:q4_k_m
ollama run llama4-scout:q4_k_m "이 코드베이스 전체에서 아키텍처 문제를 찾아줘"
```

## 5. API 가격 비교: 언제 API를 쓰고 언제 직접 호스팅할까

자가 호스팅은 하드웨어 비용과 관리 부담이 있으므로, 소규모 사용에는 API가 더 경제적일 수 있다.

| 모델 | API 제공자 | 입력 가격 | 출력 가격 | 참고 |
|------|----------|---------|---------|------|
| DeepSeek V4-Flash | DeepSeek API | $0.14/M | $0.28/M | 최저가 |
| DeepSeek V4-Pro | DeepSeek API | $1.74/M | $3.48/M | 고품질 저가 |
| Llama 4 Maverick | Together AI | ~$0.10/M | ~$0.49/M | 오픈웨이트 경쟁 |
| Llama 4 Scout | Together AI | ~$0.10/M | ~$0.15/M | 가장 저렴 |
| GPT-5.4 | OpenAI | ~$2.50/M | ~$10.00/M | 참고용 |
| Claude Opus 4.6 | Anthropic | $5.00/M | $25.00/M | 참고용 |

DeepSeek V4-Flash는 입력 $0.14/M, 출력 $0.28/M로 압도적인 가격 경쟁력을 보여준다. Claude Opus 4.6($5/$25)과 비교하면 **입력 기준 약 35배, 출력 기준 약 90배 저렴**하다. 반복 프롬프트에 적용되는 캐시 적중 가격($0.028/M)을 활용하면 비용이 더욱 낮아진다. DeepSeek V4-Pro도 $1.74/$3.48로 Claude Opus 대비 약 3~7배 저렴하면서 코딩 벤치마크에서 동급 이상의 성능을 보여준다.

Llama 4 Scout는 Together AI 기준 $0.15/M로, 10M 컨텍스트를 제공하면서도 가장 저렴하다. 대규모 문서 분석 파이프라인에 적합하다.

### Python으로 API 호출하기

```python
import openai

# DeepSeek V4-Flash — 가장 저렴한 옵션
client = openai.OpenAI(
    api_key="your-deepseek-api-key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=[
        {"role": "system", "content": "당신은 전문 Python 개발자입니다."},
        {"role": "user", "content": "FastAPI로 JWT 인증 미들웨어를 구현해줘"}
    ],
    temperature=0.3,
    max_tokens=4000
)
print(response.choices[0].message.content)
```

```python
# Llama 4 Scout — 10M 컨텍스트로 대규모 코드 분석
from openai import OpenAI

client = OpenAI(
    api_key="your-together-api-key",
    base_url="https://api.together.xyz"
)

# 전체 코드베이스를 컨텍스트에 넣고 질문
with open("entire_codebase.txt") as f:
    codebase = f.read()

response = client.chat.completions.create(
    model="meta-llama/Llama-4-Scout",
    messages=[
        {"role": "user", "content": f"다음 코드베이스를 분석하고 아키텍처 개선점을 제안해줘:\n\n{codebase}"}
    ],
    max_tokens=4000
)
```

## 6. 용도별 선택 가이드

지금까지의 분석을 종합해 실제 용도별 추천을 정리한다.

### 코딩 어시스턴트 (일상적 사용)

**1순위: DeepSeek V4-Flash → 2순위: Qwen 3.5 35B-A3B (로컬)**

API로 쓸 때는 DeepSeek V4-Flash의 가격대 성능비가 압도적이다. 로컬에서 쓰고 싶다면 Qwen 3.5 35B-A3B가 일반 GPU에서도 돌아가면서 코딩 성능이 준수하다.

### 수학/과학 연구

**1순위: Gemma 4 31B → 2순위: DeepSeek V4-Pro**

Gemma 4 31B는 AIME 2026 89.2%, GPQA Diamond 84.3%로 수학·과학 추론에서 특히 강하다. RTX 4090 한 장으로 실행 가능하다는 점도 연구자에게 매력적이다.

### 대규모 코드베이스/문서 분석

**1순위: Llama 4 Scout (10M 컨텍스트)**

10M 토큰 컨텍스트는 경쟁 모델의 10~100배에 해당한다. 전체 코드베이스나 수백 페이지의 문서를 한 번에 처리해야 할 때 유일한 선택지다.

### 자율 에이전트/도구 호출

**1순위: GLM-5.1 → 2순위: DeepSeek V4-Pro**

GLM-5.1은 600+ 반복 최적화 루프를 설계할 정도로 장기 에이전트 작업에 특화되어 있다. MIT 라이선스로 상업적 제한도 없다. DeepSeek V4-Pro도 Engram 조건부 메모리 모듈로 지속적 컨텍스트 유지에 강하다.

### 모바일/엣지 디바이스

**1순위: Gemma 4 E2B/E4B → 2순위: Qwen 3.5 소형 변종**

Gemma 4 E2B는 Android AICore를 통해 스마트폰에서 직접 실행된다. 오프라인 환경이나 지연 시간이 민감한 애플리케이션에 적합하다.

### 최소 비용으로 최대 처리량

**1순위: DeepSeek V4-Flash**

입력 $0.14/M, 출력 $0.28/M. 대량 배치 처리나 프로토타이핑에서 비용을 최소화해야 할 때 선택의 여지가 없다.

## 7. 주의사항과 한계

벤치마크 수치를 맹신하면 안 된다. 몇 가지 주의할 점이 있다.

**벤치마크 과적합 의심**: SWE-bench나 HumanEval 같은 공개 벤치마크는 모델 훈련 데이터에 포함되었을 가능성이 있다. 특히 새로운 벤치마크가 아닌 오래된 벤치마크일수록 이 위험이 크다. 실제 프로젝트에서는 벤치마크보다 10~20% 낮은 성능을 보이는 경우가 많다.

**자체 발표 vs 독립 검증**: DeepSeek V4-Pro의 SWE-bench 점수는 자체 발표에서 83.7%, 독립 검증에서 80.6%로 차이가 있다. 모델 개발사가 발표한 수치는 항상 독립 검증과 비교해보아야 한다.

**라이선스 확인**: Llama 4는 월간 활성 사용자 7억 명 제한이 있다. B2C 서비스에서 대규모로 사용할 경우 라이선스 위반 가능성을 검토해야 한다. 반면 Gemma 4, Qwen 3.5, GLM-5.1은 Apache 2.0이나 MIT로 제한이 거의 없다.

**컨텍스트 길이와 실제 품질**: Llama 4 Scout의 10M 컨텍스트는 길이만 긴 것이 아니라, 그 길이 내에서도 품질이 유지되어야 한다. 컨텍스트가 길어질수록 중간 정보를 잊는 "lost in the middle" 현상이 발생할 수 있으므로, 실제 사용 시에는 필요한 만큼만 컨텍스트를 제공하는 것이 좋다.

## 결론

2026년 오픈소스 LLM 생태계를 한 줄로 요약하면: **"상용 모델을 대체할 수 있는 모델이 하나가 아니라 여럿이다."** 핵심은 자신의 용도에 맞는 모델을 선택하는 것이다.

- **코딩**: DeepSeek V4 (API) 또는 Qwen 3.5 (로컬)
- **수학/과학**: Gemma 4 31B
- **초장문 처리**: Llama 4 Scout
- **에이전트 작업**: GLM-5.1
- **모바일/엣지**: Gemma 4 E2B
- **최저가 대량 처리**: DeepSeek V4-Flash

당장 시도해볼 추천: Ollama로 Qwen 3.5를 설치해 로컬 코딩 어시스턴트로 써보라. 8GB VRAM만 있으면 되고, 설치는 `ollama pull qwen3:8b` 한 줄이다. 상용 API에 의존하지 않고도 강력한 AI 코딩 보조를 경험할 수 있다.
