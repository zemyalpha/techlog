---
title: "맥에서 LLM 돌리기 2026: MLX가 M5에서 실제로 빨라지는 지점과 WWDC 2026의 변화"
date: "2026-09-10"
keywords: ["MLX", "Apple Silicon", "로컬 LLM", "mlx-lm", "M5 Neural Accelerators", "Foundation Models"]
lang: "ko"
description: "Apple MLX로 맥에서 LLM을 실행·파인튜닝하는 실전 가이드. M5 Neural Accelerators의 공식 벤치마크 수치, mlx-lm 명령어, WWDC 2026 Foundation Models 통합까지 검증된 사실만 정리했다."
---

# 맥에서 LLM 돌리기 2026: MLX가 M5에서 실제로 빨라지는 지점과 WWDC 2026의 변화

맥북에서 로컬 LLM을 돌리는 방법은 크게 두 갈래다. llama.cpp 계열(Ollama, LM Studio 포함)과 Apple이 직접 만든 MLX다. 2026년 들어 이 균형이 MLX 쪽으로 기우는 이유가 생겼다. M5 칩의 GPU Neural Accelerators가 MLX에 특화된 연산 유닛이고, WWDC 2026에서 Foundation Models 프레임워크가 MLX 백엔드를 정식으로 받아들였기 때문이다.

이 글은 Apple이 공식 발표한 벤치마크 수치와 실제 문서를 기반으로, MLX가 무엇이고 어디서 빨라지며 무엇이 여전히 한계인지 정리한다. 벤더 발표 수치는 출처를 명시했다.

## MLX는 모델 동물원이 아니라 배열 프레임워크다

MLX에 대해 가장 흔한 오해는 "Apple의 로컬 LLM 런처"라고 생각하는 것이다. MLX는 NumPy나 JAX처럼 배열 연산 프레임워크이며, 우리가 쓰는 `mlx-lm`은 그 위에 얹힌 LLM 도구일 뿐이다. Apple Machine Learning Research가 2023년 12월 5일 공개했고, 현재 GitHub ml-explore/mlx 저장소에서 오픈소스로 관리된다.

구조적으로 중요한 점은 두 가지다.

**첫째, 통합 메모리(unified memory)를 설계부터 활용한다.** 일반적인 GPU 서버에서는 CPU RAM과 GPU VRAM이 분리되어 있어 모델을 GPU로 옮길 때 복사 비용이 발생한다. Apple Silicon은 물리적으로 하나의 메모리 풀을 CPU와 GPU가 공유하므로, MLX의 연산은 디바이스 간 이동 없이 디스패치된다. 모델 가중치, KV 캐시, 활성화 값이 같은 풀에 살기 때문에 "VRAM 부족으로 못 돌린다"는 제약 자체가 사라진다. 32GB 맥에서 4비트 양자화된 14B 모델을 돌리는 것이 가능한 이유다.

**둘째, 지연 평가(lazy evaluation)로 연산을 융합한다.** 배열이 실제로 필요한 시점에만 구체화되므로, 하드웨어에 연산이 전달되기 전에 여러 오퍼레이션을 합쳐 메모리 할당 오버헤드를 줄인다. Python API는 NumPy와 거의 같은 모양이고, 동일한 인터페이스의 Swift, C++, C 바인딩도 제공한다.

## mlx-lm: pip 한 줄로 끝나는 도구 체인

MLX 위에서 LLM을 실행하려면 `mlx-lm` 패키지 하나면 충분하다. 텍스트 생성, 대화형 채팅, 양자화, LoRA 파인튜닝, OpenAI 호환 HTTP 서버가 전부 포함돼 있다.

```bash
# 설치
pip install mlx-lm

# 대화형 채팅 시작 (모델은 자동 다운로드)
mlx_lm.chat --model mlx-community/Qwen3-4B-4bit

# 일회성 생성
mlx_lm.generate \
  --model mlx-community/Qwen3-4B-4bit \
  --prompt "맥에서 로컬 LLM을 돌릴 때 장점을 설명해줘"
```

Hugging Face의 `mlx-community` 오가니제이션에는 이미 변환된 MLX 모델이 약 4,800개 정도 있어서, 대부분의 유명 모델은 변환 과정 없이 바로 쓸 수 있다.

**양자화도 몇 초면 된다.** Apple 문서에 나온 예시 그대로다:

```bash
mlx_lm.convert \
  --hf-path mistralai/Mistral-7B-Instruct-v0.3 \
  -q \
  --upload-repo mlx-community/Mistral-7B-Instruct-v0.3-4bit
```

`-q` 플래그로 4비트 양자화를 수행하고, `--upload-repo`를 붙이면 결과를 Hugging Face에 올릴 수도 있다.

**OpenAI 호환 서버**도 필요하면 바로 띄운다:

```bash
mlx_lm.server --model mlx-community/Qwen3-4B-4bit --port 8080
```

이후 `http://localhost:8080/v1`을 OpenAI API 엔드포인트처럼 쓰면 된다. 기존 코드의 base_url만 바꾸면 로컬 모델로 전환되는 구조다.

## M5는 어디서 빨라지고 어디서 안 빨라지나

Apple이 2025년 11월 공식 발표한 벤치마크가 이 질문에 가장 정확한 답을 준다. MacBook Pro M5(24GB)와 동일 사양의 M4를 비교했고, 프롬프트 길이는 4,096 토큰, 128 토큰을 생성하며 측정했다.

| 모델 | TTFT 속도향상 | 생성 속도향상 | 메모리 (GB) |
|------|-------------|-------------|------------|
| Qwen3-1.7B-MLX-bf16 | 3.57x | 1.27x | 4.40 |
| Qwen3-8B-MLX-bf16 | 3.62x | 1.24x | 17.46 |
| Qwen3-8B-MLX-4bit | 3.97x | 1.24x | 5.61 |
| Qwen3-14B-MLX-4bit | 4.06x | 1.19x | 9.16 |
| gpt-oss-20b-MXFP4-Q4 | 3.33x | 1.24x | 12.08 |
| Qwen3-30B-A3B-MLX-4bit | 3.52x | 1.25x | 17.31 |

(출처: Apple Machine Learning Research, "Exploring LLMs with MLX and the Neural Accelerators in the M5 GPU", 2025-11-19)

이 표가 말해주는 핵심은 **속도 향상이 단계적으로 갈린다**는 점이다.

**첫 토큰까지의 시간(TTFT)은 3.3~4.06배 빨라진다.** 프롬프트 처리(프리필)는 대규모 행렬 곱셈으로 연산 집약적(compute-bound)이고, M5의 Neural Accelerators가 바로 이 행렬 곱셈 전용 유닛이기 때문이다. 4,096 토큰짜리 긴 프롬프트를 넣는 RAG나 문서 요약 워크로드에서 체감이 커진다. Apple 발표 기준 14B 밀집 모델의 TTFT가 10초 미만, 30B MoE는 3초 미만이다.

**첫 토큰 이후의 토큰 생성은 1.19~1.27배만 빨라진다.** 디코딩은 연산이 아니라 메모리 대역폭에 묶여(bandwidth-bound) 있어서다. M4의 120GB/s 대비 M5가 153GB/s로 28% 높은데, 실제 생성 속도 향상(19~27%)은 거의 이 대역폭 차이를 그대로 따라간다. 즉 토큰 생성 속도는 칩 연산력이 아니라 메모리 대역폭이 결정하며, 이는 고성능 NVIDIA GPU와의 격차가 여전히 남아 있는 부분이다.

**메모리 여유도 공식 수치로 확인된다.** 24GB 맥에서 8B BF16 모델(17.46GB)과 4비트 30B MoE(17.31GB)를 각각 18GB 미만의 메모리로 구동했다. 통합 메모리의 실질적 이점은 속도가 아니라 이 수용 능력이다. 참고로 M5의 Neural Accelerators를 활용하려면 macOS 26.2 이상이 필요하다.

## WWDC 2026: Foundation Models가 MLX를 정식 품었다

2026년 가장 구조적으로 중요한 변화는 이것이다. WWDC 2026에서 Apple은 Foundation Models 프레임워크에 `LanguageModel` 프로토콜을 도입해, 모델 선택이 백엔드 교체 문제가 아니라 한 줄 변경 문제로 만들었다.

기존에는 Foundation Models가 Apple Intelligence 내장 모델에 잠겨 있었다. 이제 `SystemLanguageModel`(Apple Intelligence), `PrivateCloudComputeLanguageModel`, Anthropic/Google Swift 패키지, 그리고 `MLXLanguageModel`이 모두 같은 인터페이스를 구현한다. mlx-swift-lm 패키지를 추가하면 된다:

```swift
import FoundationModels
import MLXFoundationModels

let model = MLXLanguageModel(modelID: "mlx-community/Qwen3-4B-4bit")
let session = LanguageModelSession(model: model)
let response = try await session.respond(to: "Swift actors를 설명해줘")
print(response.content)
```

모델은 최초 사용 시 Hugging Face에서 내려받아 디스크에 캐시된다. 스트리밍, 툴 콜링, 구조화 출력(`@Generable`), 멀티턴 세션이 내장 모델과 동일하게 동작한다는 것이 핵심이다. 이 구조의 실용적 의미는 "벤더를 고르는" 것이 아니라 "정책을 고르는" 것이 된다는 점이다. 문서 요약 같은 워크로드는 로컬 4B 모델로 무료로 돌리고, 프론티어 품질이 필요한 쿼리만 클라우드로 라우팅하는 하이브리드가 API 재작성 없이 가능해진다.

## 모델 고르기와 흔한 실수

mlx-community의 약 4,800개 모델 중 실무에서 쓸 만한 기본 선택지는 다음과 같다:

- **Qwen3-4B-4bit (약 2.3GB)** — 기본 픽. 8GB 맥에서도 무리 없다
- **Llama-3.2-3B-Instruct-4bit (약 1.8GB)** — 최소 사양 또는 첫 토큰 지연이 중요할 때
- **Qwen3-8B-4bit (약 4.6GB)** — 품질 업그레이드. 16GB 이상 권장
- **Qwen3-14B-4bit / gpt-oss-20b** — 24GB 이상 맥에서 본격 활용

흔히 하는 실수 세 가지:

1. **메모리 계산을 안 한다.** 맥은 OS와 다른 앱도 같은 통합 메모리를 쓴다. Apple 벤치마크에서도 24GB 머신에 18GB짜리 워크로드를 띄웠지, 32GB짜리 모델을 올리지는 않았다. 모델 크기 + KV 캐시(컨텍스트 길이에 비례) + 시스템 여유분을 계산하라.
2. **BF16과 양자화를 구분 안 한다.** 같은 8B 모델이 BF16에서는 17.46GB, 4비트에서는 5.61GB다. 실험용이 아니라면 4비트가 Apple Silicon의 실용적 스위트 스팟이다.
3. **생성 속도 기대치를 높게 잡는다.** M5로 갈아타도 토큰 생성 속도는 기껏해야 1.2배다. 긴 응답을 뽑는 챗봇 용도라면 M4와 M5의 체감 차이가 TTFT만큼 크지 않다. 빨라지는 것은 긴 프롬프트를 넣는 프리필 heavy 워크로드다.

## 결론

- MLX는 통합 메모리를 설계부터 활용하는 배열 프레임워크이며, `mlx-lm` 하나로 실행·양자화·파인튜닝·서빙이 전부 된다
- M5의 Neural Accelerators는 TTFT를 3.3~4.06배 끌어올리지만, 토큰 생성 속도는 메모리 대역폭(120→153GB/s)이 결정하므로 1.19~1.27배에 그친다 (Apple 공식 벤치마크 기준)
- 맥의 진짜 강점은 속도가 아니라 수용 능력이다. 24GB로 30B MoE 4비트를 돌릴 수 있는 것이 소비자 GPU로는 불가능한 조합이다
- WWDC 2026의 `MLXLanguageModel`는 로컬/클라우드 하이브리드 라우팅을 코드 한 줄 수준으로 만들었다
- 당장 해볼 것: `pip install mlx-lm` 후 `python -m mlx_lm.chat --model mlx-community/Qwen3-4B-4bit`로 5분 안에 시작할 수 있다

## 참고 자료

- Apple Machine Learning Research — Exploring LLMs with MLX and the Neural Accelerators in the M5 GPU (2025-11-19): https://machinelearning.apple.com/research/exploring-llms-mlx-m5
- GitHub — ml-explore/mlx: https://github.com/ml-explore/mlx
- GitHub — ml-explore/mlx-lm (LoRA 문서 포함): https://github.com/ml-explore/mlx-lm
- Apple Developer — WWDC26 Machine Learning 가이드: https://developer.apple.com/wwdc26/guides/machine-learning/
