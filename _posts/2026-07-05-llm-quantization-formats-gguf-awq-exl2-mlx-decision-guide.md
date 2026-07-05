---
title: "GGUF, AWQ, EXL2, MLX: 하드웨어별 LLM 양자화 포맷 선택 가이드"
date: "2026-07-05"
keywords: ["LLM 양자화", "GGUF", "AWQ", "EXL2", "MLX", "quantization"]
lang: "ko"
description: "GGUF, AWQ, EXL2, MLX, bitsandbytes의 차이를 파일 포맷과 알고리즘으로 구분하고, 하드웨어별로 올바른 양자화 포맷을 선택하는 실전 의사결정 가이드를 제공한다."
---

# GGUF, AWQ, EXL2, MLX: 하드웨어별 LLM 양자화 포맷 선택 가이드

70억 개 매개변수(7B) 모델 하나가 FP16 정밀도로 디스크에 저장되면 약 14GB를 차지한다. 이를 4비트로 압축하면 4GB 안팎으로 줄어든다. 단일 소비자용 GPU에서 실행할 수 없던 모델이 양자화 하나로 실행 가능해지는 순간이다.

문제는 "어떤 양자화를 쓸 것인가"라는 질문에 답이 너무 많다는 점이다. GGUF, AWQ, GPTQ, EXL2, MLX, bitsandbytes — 이름만 보면 모두 비슷해 보이지만, 이들은 카테고리 자체가 다르다. 잘못 고르면 하드웨어에서 아예 실행되지 않거나, 실행되더라도 속도가 절반 이하로 떨어진다.

이 글에서는 이 여섯 가지를 세 가지 카테고리로 정리하고, 각 하드웨어(NVIDIA GPU, Apple Silicon, CPU, 모바일)에서 어떤 선택을 해야 하는지 실전 의사결정 기준을 제시한다. 검색 결과 링크를 나열하는 대신, 각 포맷이 왜 그 하드웨어에 맞는지 설명하고 실제로 사용하는 명령어와 코드를 포함한다.

## 1단계: 포맷인가, 알고리즘인가, 온더플라이인가

가장 흔한 실수는 이 여섯 가지를 동일선상에 놓고 비교하는 것이다. 실제로는 세 가지 다른 종류의 것이 섞여 있다.

**파일 포맷(file format)**은 다운로드한 파일 자체가 모델이다. 가중치, 토크나이저, 메타데이터가 하나의 파일에 들어 있고, 특정 런타임에 묶여 있다. GGUF, EXL2, MLX가 여기에 해당한다. GGUF는 llama.cpp 전용, EXL2는 ExLlamaV2 전용, MLX는 mlx-lm과 Ollama의 Apple Silicon 백엔드 전용이다.

**양자화 알고리즘(quantization algorithm)**은 파일 형식이 아니라 절차다. 출력물은 일반적인 Hugging Face safetensors 샤드와 config 파일이다. GPTQ와 AWQ가 여기에 속한다. "AWQ 모델"을 다운로드하면 받는 것은 다른 Hugging Face 모델과 동일한 구조의 디렉토리다.

**온더플라이(on-the-fly)**는 사전에 양자화된 파일이 아예 없다. 모델을 로드하는 시점에 가중치를 압축하고, 순전파마다 필요할 때 해제한다. bitsandbytes가 이 방식이며, 이 때문에 QLoRA 파인튜닝이 가능하다 — 얼려진 4비트 기반 모델 위에 소형 LoRA 어댑터를 훈련하기 때문이다.

이 구분을 먼저 확립하지 않으면 "GGUF를 vLLM에 올리려면 어떻게 하나요?" 같은 질문이 나온다. GGUF는 llama.cpp/Ollama/LM Studio 런타임용이고, vLLM은 safetensors 기반의 AWQ나 GPTQ를 로드한다.

## 2단계: 각 포맷의 핵심 설계 철학

### GGUF — 만능 포맷, CPU가 첫 시민

GGUF는 "Georgi Gerganov Universal Format"의 약자로, llama.cpp 창시자인 Georgi Gerganov가 2023년 8월에 설계했다. 기존 GGML 형식을 대체하며, 단일 파일에 가중치·토크나이저·메타데이터·양자화 매개변수를 모두 묶어 넣는다.

가장 큰 특징은 하드웨어 비의존성이다. CPU(llama.cpp), NVIDIA GPU(CUDA), AMD GPU(ROCm), Apple Silicon(Metal)을 모두 지원하며, 모델의 레이어를 CPU와 GPU에 나누어 배치하는 부분 오프로드(partial offload)도 가능하다. VRAM에 모델이 조금 모자랄 때 일부 레이어만 CPU에서 실행하는 식이다. 이 범용성 때문에 "어떤 기계에서든 실행해야 할 때"의 기본 선택이 된다.

품질 측면에서 llama.cpp의 K-quant(Q4_K_M, Q5_K_M 등)는 슈퍼블록 구조를 사용해 기존 Q4_0보다 같은 평균 비트수에서 더 낮은 양자화 오차를 달성한다. 2026년 1월 arXiv에 보고된 Llama-3.1-8B-Instruct 기준 독립 평가에서, Q4_K_S는 F16 대비 약 71% 크기 절감에도 불구하고 종합 점수 69.17로 F16의 69.47과 거의 차이가 없었다.

실제 사용 예시 — llama.cpp에서 GGUF 모델 실행:

```bash
# 모델 다운로드 후 실행 (모든 레이어를 GPU로 오프로드)
./llama-cli \
  -m ./models/llama-3-8b-Q4_K_M.gguf \
  -p "한국어로 자기소개를 해줘" \
  -n 256 \
  -ngl 99 \
  --temp 0.7
```

```bash
# OpenAI 호환 API 서버로 실행
./llama-server \
  -m ./models/llama-3-8b-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 \
  -ngl 99 -c 4096 \
  --parallel 4 --cont-batching
```

### AWQ — 중요한 1%를 보호하라

AWQ(Activation-aware Weight Quantization)는 MIT의 Song Han(한송) 교수 지도 하에 Lin Ji 등이 개발했으며, 2023년 6월 arXiv에 게재되었다(arXiv:2306.00978). MLSys 2024에서 Best Paper Award를 받았다.

핵심 통찰은 단순하다: 모든 가중치가 동등하게 중요하지 않다. 가중치 값 자체가 아니라 활성화(activation) 크기를 기준으로 상위 1%의 "현저한(salient)" 가중치를 식별하고, 이 부분만 보호하면 양자화 오차가 크게 줄어든다. 역전파나 가중치 재구성이 필요 없기 때문에 quantization 속도도 빠르다.

INT4, 그룹 크기 128 기준에서 AWQ는 양자화로 인한 perplexity 페널티를 4.57에서 1.17로 약 74% 줄였다고 논문에서 보고한다. 동일 4비트에서 GPTQ보다 추론 및 지시 수행 벤치마크에서 앞선다는 점도 AWQ 논문이 제시하는 근거다. FP16 Hugging Face 구현 대비 약 3배 이상의 속도 향상 역시 AWQ 논문에서 보고한 수치다.

2025~2026년 들어 새 모델이 출시될 때 AWQ 또는 GGUF 버전이 먼저 올라오는 경우가 많아졌다. vLLM, Hugging Face TGI, NVIDIA TensorRT-LLM, LMDeploy에서 1차 지원한다.

실제 사용 예시 — vLLM에서 AWQ 모델 서빙:

```python
from vllm import LLM, SamplingParams

# AWQ 양자화 모델 로드
llm = LLM(
    model="TheBloke/Llama-2-13B-AWQ",
    quantization="awq",
    dtype="float16",
    gpu_memory_utilization=0.9,
)

sampling = SamplingParams(temperature=0.7, max_tokens=256)
output = llm.generate("딥러닝을 한 줄로 설명해줘", sampling)
print(output[0].outputs[0].text)
```

### EXL2 — VRAM에 맞춰 비트수를 미세 조정

EXL2는 ExLlamaV2(turboderp 제작)에서 사용하는 포맷으로, 독특한 점은 정수 비트수에 얽매이지 않는다는 것이다. 단일 모델 내에서 양자화 수준을 섞어, 2비트부터 8비트 사이의 소수점 평균 비트수(예: 4.65 bpw)를 만들 수 있다. 알고리즘이 각 매트릭스별로 양자화 오차를 측정한 뒤 가장 민감한 레이어에 더 많은 비트를 할당한다.

대가는 호환성이다. EXL2는 NVIDIA GPU 전용이다. AMD, Apple Silicon, CPU는 지원하지 않는다. 대신 단일 GPU에서의 토큰 생성 속도는 매우 빠르다 — 단일 NVIDIA GPU에서 표준 로더 대비 2~5배 속도 향상이 사용자 보고에서 나온다(다중 사용자나 배치 추론 환경에서는 더 낮을 수 있다).

적합한 사례는 "단일 NVIDIA GPU 하나로 최대한 많은 토큰/초를 끌어내고 싶을 때"다. 예를 들어 RTX 4090(24GB)에서 70B 모델을 2.55 bpw로 맞추면 약 38 tokens/sec로 생성할 수 있다고 한다. 단, 이는 단일 배치 최적 조건에서의 벤치마크 수치이므로 실제 환경에서는 더 낮을 수 있다.

### MLX — Apple Silicon의 통합 메모리를 위해

MLX는 Apple이 2023년 11월에 공개한 오픈소스 머신러닝 프레임워크로, Apple Silicon의 핵심인 통합 메모리 아키텍처(unified memory)에 맞춰 설계되었다. CPU, GPU, Neural Engine이 하나의 물리적 메모리 풀을 공유하기 때문에, PCIe 버스를 통한 텐서 복사가 필요 없다. 진정한 제로카피(zero-copy) 텐서 연산이 가능하다.

이 아키텍처가 Apple Silicon이 대형 모델에서 무게를 발휘하는 이유다. M4 Max는 최대 128GB 통합 메모리를 546 GB/s 대역폭으로 처리할 수 있다. 이는 비슷한 가격대의 개별 GPU가 담을 수 없는 용량이다.

2026년에 Ollama가 Apple Silicon 백엔드를 MLX로 전환했다는 보도가 있다. 4비트 MLX는 풀정밀도 MMLU 점수의 약 97%를 유지한다는 커뮤니티 벤치마크도 있으나, 이는 2차 출처이므로 참고 수치로만 활용해야 한다. 안정적인 결론은 단순하다: M 시리즈 Mac에서는 MLX가 기본 경로이며, 같은 파일을 CPU 박스나 PC GPU에서도 돌려야 하는 게 아니라면 GGUF보다 MLX가 우선이다.

### bitsandbytes — 유일한 트레이닝 가능 포맷

bitsandbytes는 이 목록에서 유일하게 "추론이 아니라 훈련을 위해 만들어진" 존재다. NF4(4-bit Normal Float)와 LLM.int8()을 제공하며, 사전 양자화 파일이 필요 없다. 모델 로드 시점에 압축하고 순전파마다 해제한다.

QLoRA는 이 NF4를 기반으로 한 파인튜닝 기법이다. 4비트로 얼려진 기반 모델 위에 소형 LoRA 어댑터를 훈련한다. Tim Dettmers 등이 2023년 5월에 발표한 QLoRA 논문(arXiv:2305.14314, NeurIPS 2023)에 따르면, 650억 매개변수(65B) 모델을 단일 48GB GPU에서 풀 16비트 파인튜닝과 동등한 성능으로 훈련할 수 있다. 기존 16비트 파인튜닝이 780GB 이상의 GPU 메모리를 요구하던 것과 비교하면 약 16배의 메모리 절감 효과다.

실제 사용 예시 — QLoRA로 7B 모델 파인튜닝:

```python
import torch
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

# 4비트 NF4 양자화 설정
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,  # 이중 양자화
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-8B",
    quantization_config=bnb_config,
    device_map="auto",
)

model = prepare_model_for_kbit_training(model)

# LoRA 어댑터 추가 (이 부분은 16비트로 훈련됨)
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# 출력: trainable params: ~42M || all params: ~8B || trainable%: 0.52
```

`trainable%: 0.52` — 전체 매개변수의 0.5%만 훈련하면서 풀 파인튜닝과 비슷한 결과를 낸다. NVIDIA GPU(Turing 아키텍처, 즉 RTX 20 시리즈 또는 T4 이상)가 필요하다.

## 3단계: 하드웨어별 의사결정표

여섯 포맷을 하드웨어 기준으로 정리하면 다음과 같다.

| 포맷 | 하드웨어 | 주요 런타임 | 비트 깊이 | 훈련 가능 | 특징 |
|------|----------|-------------|-----------|-----------|------|
| **GGUF** | CPU · NVIDIA · AMD · Apple Silicon (+ 부분 오프로드) | llama.cpp, Ollama, LM Studio | 2~8비트 | 아니오 | K-quant 품질 우수, 이식성 최강 |
| **AWQ** | NVIDIA GPU (엣지·모바일 GPU 포함) | vLLM, HF TGI, TensorRT-LLM, LMDeploy | INT4 4비트 | 아니오 | GPTQ보다 추론 우수, 프로덕션 GPU 기본값 |
| **GPTQ** | NVIDIA GPU | vLLM(Marlin/Machete 커널), GPTQModel | 2~4비트 | 아니오 | 레거시 표준, 기존 체크포인트용 |
| **EXL2** | NVIDIA GPU 전용 | ExLlamaV2 + TabbyAPI | 2~8 bpw(소수점 가능) | 아니오 | 단일 GPU 토큰/초 최강 |
| **MLX** | Apple Silicon 전용 | mlx-lm, Ollama(macOS) | 3~8비트 | 부분(mlx-lm) | 통합 메모리 활용, Mac 기본값 |
| **bitsandbytes NF4** | NVIDIA(Turing/T4 이상) | Hugging Face Transformers | 4비트 NF4 / 8비트 INT8 | 예(QLoRA) | 파인튜닝 전용, 추론 속도는 느림 |

이 표를 보는 순서가 중요하다. 먼저 "내 하드웨어가 무엇인가"로 행을 고르면 메뉴의 대부분이 사라진다. 그 다음에 비트 깊이와 품질을 고민하면 된다.

**NVIDIA GPU(A100/H100 등)** — 새 모델이라면 AWQ가 기본값이다. 단일 소비자 GPU(RTX 4090 등)에서 최대 속도가 필요하면 EXL2. 기존 GPTQ 체크포인트가 있다면 그대로 GPTQ 사용. 파인튜닝이 목적이면 bitsandbytes NF4.

**Apple Silicon(M1~M5)** — MLX가 기본 경로이며 Ollama의 기본 백엔드이기도 하다. 같은 모델 파일을 CPU 박스나 PC GPU에서도 실행해야 하면 GGUF.

**CPU 전용 서버/랩탑** — GGUF가 사실상 유일한 선택. Q4_K_M(균형) 또는 Q5_K_M(품질 우선)로 시작할 것.

**파인튜닝** — bitsandbytes NF4 한 가지뿐이다. 추론용으로는 GGUF나 AWQ로 변환 후 서빙한다.

## 4단계: 실제 선택 시 주의할 점

**4비트 이하로 내려가면 품질이 급격히 떨어진다.** GPTQ의 2비트나 삼진(ternary) 양자화는 논문에서는 가능하다고 되어 있지만, 커뮤니티 테스트에서는 4비트 미만에서 품질이 눈에 띄게 저하된다. 프로덕션에서는 4비트 이상을 유지할 것.

**"무손실(lossless)" 마케팅 표현을 문자 그대로 믿지 말 것.** AWQ의 무손실 주장은 VILA 같은 비전-언어 모델에 한정된 것이다. 텍스트 LLM에서 AWQ 4비트는 "사실상 무손실에 가깝다(near-lossless)"로 이해해야 하며, 실제로는 작은 품질 저하가 있다. 배포 전에 자신의 프롬프트로 벤치마크할 것.

**같은 모델이라도 양자화 방식에 따라 결과가 다를 수 있다.** 2026년 1월 arXiv 평가에서는 5비트 Q5_0가 특정 집계 점수에서 FP16을 근소하게 앞선 결과가 나왔다. 양자화 노이즈가 일종의 정규화 효과를 낸 것으로 추정된다. 이를 보편적 규칙(5비트가 항상 풀정밀도보다 낫다)으로 해석하면 안 된다. 검증 가능한 특정 모델·태스크 조합에서의 결과로 이해할 것.

**EXL2 벤치마크 수치는 최적 조건 기준이다.** 제작자가 보고한 토큰/초는 단일 배치 최적 생성 기준이며, 다중 사용자나 배치 추론 환경에서는 더 낮다. 서빙 환경에서는 실제 부하로 테스트해야 한다.

## 결론: 핵심 요약

- **포맷은 하드웨어 결정이다.** 비트 깊이가 아니라 타겟 하드웨어가 먼저다. CPU 박스에서 EXL2 파일을 받으면 실행조차 안 된다.
- **GGUF는 만능 기본값.** CPU, GPU, Apple Silicon 모두에서 실행되며 부분 오프로드도 지원한다. 혼합 환경이거나 타겟 하드웨어가 불확실하면 GGUF로 시작할 것.
- **NVIDIA 프로덕션은 AWQ.** vLLM, TGI, TensorRT-LLM에서 1차 지원하며, GPTQ보다 같은 4비트에서 더 나은 추론 품질을 보인다. 새 모델은 AWQ로 먼저 출시되는 추세다.
- **Apple Silicon은 MLX.** 통합 메모리를 활용한 제로카피 연산이 핵심이며, 2026년부터 Ollama의 기본 백엔드다.
- **파인튜닝은 bitsandbytes NF4 하나뿐.** QLoRA로 4비트 기반 모델 위에 LoRA 어댑터를 훈련한 뒤, 서빙을 위해 GGUF나 AWQ로 변환하는 것이 일반적 워크플로다.

가장 먼저 해볼 일: 사용 중인 하드웨어에서 GGUF Q4_K_M 버전의 7B 모델을 다운로드해 `llama-server`로 띄워보는 것이다. 한 줄 명령으로 로컬 OpenAI 호환 API를 얻을 수 있고, 그 경험이 "내가 왜 이 포맷을 선택하는가"에 대한 감각을 만들어준다.
