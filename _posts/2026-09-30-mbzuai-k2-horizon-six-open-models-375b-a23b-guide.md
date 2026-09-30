---
title: "가중치만이 아니라 레시피까지 연다 — MBZUAI K2 Horizon 0.9B~375B 6형제 분해와 사이즈 선택 가이드"
date: "2026-09-30"
keywords: ["K2 Horizon", "MBZUAI", "오픈소스 LLM", "MoE", "Apache 2.0", "IFM", "오픈 웨이트"]
lang: "ko"
description: "MBZUAI IFM이 2026년 9월 3일 공개한 K2 Horizon 6개 오픈 모델(0.9B~375B-A23B)의 구성, 벤치마크, vLLM/SGLang 실행법, 그리고 '완전 공개'의 실체와 주의점을 정리한다."
---

# 가중치만이 아니라 레시피까지 연다 — MBZUAI K2 Horizon 0.9B~375B 6형제 분해와 사이즈 선택 가이드

오픈소스 LLM 시장에서 "오픈"이라는 말은 오랫동안 반쪽짜리였다. 대부분의 공개 모델은 가중치만 내놓고, 훈련 데이터와 학습 코드는 비공개로 남긴다. 그래서 벤치마크 주장을 재현할 수도, 오염 여부를 감사할 수도 없었다. 2026년 9월 3일, 아부다비 MBZUAI 산하 Institute of Foundation Models(IFM)이 발표한 **K2 Horizon**은 이 관행을 깨는 시도다. 0.9B부터 375B까지 여섯 개 모델을 가중치·훈련 코드·훈련 데이터·레시피와 함께 Apache 2.0으로 공개한 것이다.

이 글에서는 K2 Horizon의 라인업 구성, 아키텍처 특징, 발표된 벤치마크, 실제 실행 방법, 그리고 "완전 공개"라는 주장 뒤에 숨은 주의점까지 정리한다. 특히 6개 사이즈 중 자신의 하드웨어와 용도에 맞는 모델을 고르는 기준에 초점을 맞췄다.

## 여섯 개 모델, 하나의 아키텍처

K2 Horizon의 핵심 설계는 "하나의 패밀리"다. 여섯 모델이 공통 아키텍처, 어휘(vocabulary), 훈련 방법론, 배포 도구를 공유한다. 0.9B로 프로토타입을 만들고 375B로 확장할 때 배포 코드를 다시 쓰지 않아도 된다는 것이 IFM의 주장이다. 단, 공식 발표에 따르면 0.9B 모델만은 나머지 다섯 모델보다 작은 어휘를 사용한다.

| 모델 | 구조 | 활성 파라미터 | IFM 제시 배치 대상 |
|------|------|------------|----------------|
| 0.9B | Dense | 0.9B | 워치, 안경 등 초저사양 엣지 |
| 3.7B | Dense | 3.7B | 폰, 파인튜닝 실험 |
| 7B | Dense | 7B | 온디바이스 앱, 소프트웨어 엔지니어링 |
| 32B | Dense | 32B | 노트북, 온프레미스 서버 |
| 36B-A4B | Sparse (MoVA) | 4B | 비용 효율적 로컬 호스팅 |
| 375B-A23B | Sparse MoE | 23B | 엔터프라이즈 추론 워크로드 |

흥미로운 지점은 두 개의 희소(sparse) 모델이다. 375B-A23B는 375B 파라미터를 저장하지만 토큰당 23B만 활성화하는 MoE 구조로, 미드트레이닝 단계부터 524,288 토큰(512K) 컨텍스트를 지원한다. 36B-A4B는 **MoVA(Mixture of Value Attention)**라는 새로운 구조를 쓴다. 일반 MoE가 피드포워드 층의 전문가를 라우팅한다면, MoVA는 어텐션의 value 프로젝션에 희소성을 적용해 연산량을 늘리지 않고 추론 능력을 끌어올리는 것을 목표로 한다. 워크스테이션급 GPU 한 장으로 돌릴 수 있는 "로컬 호스팅용 특화 모델"이라는 포지션이다.

또 하나, IFM은 **diffusion distillation** 기법을 도입해 토큰 블록을 병렬로 생성함으로써 약 3배의 생성 속도 향상이 있다고 주장한다. 텍스트 디퓨전 생성 자체는 Mercury, LLaDA 등으로 이미 연구되어 온 영역인데, 이를 프로덕션 모델 패밀리에 증류로 넣었다는 점이 공학적으로 의미 있다. 다만 이 3배 수치는 하드웨어나 컨텍스트 길이 조건이 명시되지 않은 발표사 자체 측정이며, 독립 검증은 아직 없다.

## 벤치마크: 강한 부분과 그렇지 않은 부분

K2 Horizon의 성능 수치는 전부 IFM 발표 값(issuer-reported)이라는 점을 먼저 밝힌다. 독립 재현 결과는 아직 공개되지 않았다. 375B-A23B 모델카드 기준 주요 결과는 다음과 같다 (출처: [Hugging Face 모델카드](https://huggingface.co/IFM/K2-Horizon-375B-A23B)).

| 벤치마크 | K2 375B-A23B | Nemotron 3 Ultra (550B) | MiniMax-M3 (428B) | GLM 5.2 max (753B) | GPT 5.6 Terra (high) |
|------|------|------|------|------|------|
| GDPVal-AA (에이전트, Elo) | 1,441 | 1,162 | 1,380 | 1,498 | 1,503 |
| Toolathlon Verified | 65.3 | 34.3 | 53.7 | 59.9 | 28.7* |
| Terminal-Bench 2.1 | 70.2 | 53.9 | 65.2 | 77.9 | 80.5 |
| MCPMark | 67.7 | 45.7 | 48.8 | 72.4 | 65.3 |
| GPQA Diamond | 87.3 | 86.7 | 89.5 | 91.1 | 91.1 |
| SWE Bench Pro (strict) | 42.6 | 38.7 | 43.8 | 46.7 | -- |

*GPT 5.6 Terra의 Toolathlon 수치이며, Luna(max)는 67.5.

표를 읽으면 그림이 명확해진다. **에이전트형 도구 사용(task tool use)에서 K2는 같은 세대 오픈 MoE 모델들보다 확실히 앞서지만, GLM 5.2나 GPT 5.6 같은 상위 모델과는 벤치마크마다 우열이 갈린다.** Toolathlon에서는 비교군 전체에서 최상위권(65.3)이고, 자동화 벤치마크(Automation Bench Public 25.3, Apex-Agents 24.8)에서도 오픈 모델 중 선두다. 반면 Terminal-Bench 2.1(70.2)과 MCPMark(67.7), GPQA Diamond(87.3)에서는 GLM 5.2가 앞서 있다. IFM이 "자기 크기의 2.6배까지의 오픈 웨이트 MoE와 매치하거나 능가한다"고 주장하는 것과 일치하지만, 클로즈드 프론티어를 넘었다는 이야기는 아니라는 뜻이다.

소형 모델 쪽 주장도 비슷한 결이다. IFM 발표에 따르면 0.9B 모델은 AIME 2026에서 48.5, HumanEval+ 79.9, LiveCodeBench v6 37.4를 기록해 동급 최고 수준이라고 한다. 7B는 "10B 미만 최강", 3.7B는 "4B 미만 최강"을 표방한다. 다만 이 역시 자체 측정값이고, Qwen3 계열 등 경쟁 모델과의 독립 비교는 아직 부족하다.

## 실행: vLLM과 SGLang 레시피

K2 Horizon은 vLLM과 SGLang 공식 지원으로 출시됐고, API로는 Cerebras, AWS, Nebius 등에서 제공된다. 375B-A23B는 8-way 텐서 병렬 + 전문가 병렬이 권장 구성이다.

```shell
# vLLM (공식 레시피 기준)
vllm serve IFM/K2-Horizon-375B-A23B \
  --revision main \
  --tensor-parallel-size 8 \
  --enable-expert-parallel \
  --trust-remote-code \
  --dtype bfloat16 \
  --reasoning-parser k2_horizon \
  --tool-call-parser k2_horizon \
  --enable-auto-tool-choice
```

추론 API는 OpenAI 호환 인터페이스를 따른다. 권장 설정은 `temperature=1.0`, `top_p=0.95`이고, 추론 깊이는 요청별로 `reasoning_effort`로 조절한다. 툴 호출 포맷은 `json`, `xml`, `xml_typed` 중 선택 가능하며 기본값은 `xml`이다.

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:30000/v1", api_key="EMPTY")
response = client.chat.completions.create(
    model="IFM/K2-Horizon-375B-A23B",
    messages=[{"role": "user", "content": "결과를 단계별로 설명해줘."}],
    temperature=1.0,
    top_p=0.95,
    max_tokens=32768,
    extra_body={"chat_template_kwargs": {
        "reasoning_effort": "high",
        "tool_call_format": "xml",
    }},
)
msg = response.choices[0].message
print("추론:", getattr(msg, "reasoning_content", None))
print("답변:", msg.content)
```

이 구조가 실무적으로 중요한 이유가 하나 있다. thinking이 `reasoning_content`로, 최종 답이 `content`로 분리되어 반환되므로 로깅 파이프라인에서 추론 과정과 결과를 분리해 저장할 수 있다. 긴 추론을 유도하는 에이전트 워크로드에서 토큰 비용 추적을 잡는 데 유용하다.

여기에 "완전 공개"의 진짜 가치가 있다. 모델카드에 따르면 중간 체크포인트도 공개되어 있어, 훈련이 진행되며 능력이 어떻게 변하는지 단일 시점이 아니라 학습 전반에 걸쳐 추적할 수 있다. vLLM의 `--revision pretrain_ph1_211000`처럼 특정 학습 단계 체크포인트를 지정해 서빙하는 것도 가능하다. 연산 예산이 있는 팀이라면 능력 형성 과정 자체를 연구 대상으로 삼을 수 있다는 뜻이다.

## 주의점: "완전 공개"의 롤아웃은 아직 진행 중

뉴스 보도만 보고 바로 도입을 결정하기 전에 확인할 점들이 있다.

**첫째, 공개 범위가 발표와 실제 저장소 사이에 차이가 있다.** 발표에서는 "가중치, 코드, 훈련 데이터, 방법론 전부 공개"라고 했지만, Data Phoenix가 9월 10~11일자 저장소 목록을 확인한 바에 따르면 모델카드 3개와 Apache 2.0 라이선스 표기는 확인되나, 일부 코드 저장소와 기술 보고서는 여전히 작업 중(pending) 상태다. 일부 전문가 체크포인트도 미공개로 남아 있다. "역사상 최대 완전 공개"라는 포지셔닝을 평가하려면 전체 인벤토리가 닫히기를 기다려야 한다.

**둘째, 신규 아키텍처의 도구 지원 폭.** MoVA는 새로운 구조라 vLLM과 SGLang 외 런타임(llama.cpp, MLX 등)의 지원은 늦어질 수 있다. 맥이나 로컬 GPU에서 GGUF로 돌리고 싶은 사람은 36B-A4B가 아니라 dense 모델(7B, 32B)을 먼저 보는 게 현실적이다.

**셋째, 벤치마크 해석.** 위 표의 모든 수치는 IFM 자체 측정이다. 특히 GDPVal-AA 같은 Elo 기반 평가는 평가 하네스 구성에 따라 흔들린다. 모델카드에도 BrowseComp는 DeepSeek-V3.2 기술보고서의 Discard-all@95k 방식을 썼다는 등 세부 조건이 달라 비교군과 완전히 동일하지 않은 측면이 명시되어 있다. 도입 결정 전 자기 워크로드로 스모크 테스트하는 것이 원칙이다.

**넷째, 라이선스는 진짜 깔끔하다.** Llama 커뮤니티 라이선스와 달리 Apache 2.0에는 매출 임계치도, 허용 용도 제한도 없다. 규제 산업에서 법무 검토를 거쳐야 하는 팀에게는 이 점 하나만으로도 검토 가치가 있다.

## 사이즈 선택: 용도별 추천

6개 모델을 실무 기준으로 정리하면 이렇다.

- **엣지/워치·안경 (0.9B)**: 배터리와 메모리가 병목인 기기. 단, 어휘가 다르므로 상위 모델과의 토큰 호환은 기대하지 말 것.
- **모바일·파인튜닝 베이스 (3.7B)**: IFM이 명시적으로 파인튜닝 친화적으로 설계했다. 도메인 특화 파생 모델 베이스로 가장 먼저 검토할 대상.
- **온디바이스 코딩 (7B)**: 노트북에서 돌리는 코드 어시스턴트 용도. 10B 미만 최강 주장의 검증 대상.
- **온프레 단일 GPU (32B)**: H100/A100 한 장이면 bfloat16으로 돌릴 수 있는 마지막 사이즈. 검색 가능한 규정 준수 환경에 적합.
- **로컬 MoE (36B-A4B)**: 활성 4B라 지연이 낮고 메모리는 36B치를 쓴다. MoVA 지원이 vLLM/SGLang에 안착한 뒤 본격 채택 검토.
- **엔터프라이즈 (375B-A23B)**: 8×H200급. 에이전트 도구 사용 워크로드에서 오픈 진영 최상위권 성능이 필요할 때.

## 결론

- K2 Horizon은 0.9B~375B까지 여섯 사이즈를 **가중치·데이터·코드·레시피 포함 Apache 2.0**으로 공개한 패밀리로, "오픈"의 기준을 한 단계 끌어올렸다.
- 375B-A23B는 **에이전트형 도구 사용 벤치마크에서 오픈 MoE 최상위권**이지만, GLM 5.2·GPT 5.6과는 벤치마크별로 우열이 갈린다. 모든 수치는 아직 IFM 자체 발표 값이다.
- diffusion distillation(약 3배 속도)과 MoVA는 흥미롭지만 **독립 검증과 런타임 생태계 확산이 남아 있다.**
- "완전 공개"는 발표 시점 기준으로도 **일부 코드·보고서가 진행 중**이므로, 도입 전 저장소 인벤토리를 직접 확인하라.
- 당장 해볼 것: 자신의 하드웨어에 맞는 사이즈 하나를 골라 vLLM 공식 레시피로 띄우고, 실제 워크로드 20~30건으로 스모크 테스트를 돌려보는 것. 벤치마크 숫자보다 그 결과가 더 많은 것을 말해줄 것이다.

## 참고 자료

- [K2-Horizon-375B-A23B 모델카드 (Hugging Face)](https://huggingface.co/IFM/K2-Horizon-375B-A23B) — 벤치마크 표, vLLM/SGLang 레시피, API 사용법
- [AlphaSignal: MBZUAI Releases K2 Horizon](https://alphasignal.ai/news/mbzuai-releases-k2-horizon-the-largest-fully-open-source-ai-fleet-ever) — 라인업·라이선스·MoVA/diffusion distillation 요약
- [Data Phoenix: MBZUAI launches six K2 Horizon models](https://dataphoenix.info/news/mbzuai-k2-horizon-open-models) — 저장소 롤아웃 상태, issuer-reported 수치 주의점
- [LLM Reference Changelog 2026-09](https://www.llmreference.com/changelog/2026-09) — 출시일(2026-09-03) 및 파라미터 사양
