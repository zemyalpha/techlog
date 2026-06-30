---
title: "Rust가 LLM 추론 파이프라인에 들어오는 방식: Candle·Burn·tch-rs 실전 비교"
date: "2026-06-30"
keywords: ["Rust", "LLM inference", "Candle", "Burn", "tch-rs"]
lang: "ko"
description: "HuggingFace Candle, Tracel-AI Burn, tch-rs 세 Rust ML 프레임워크를 LLM 추론 관점에서 비교한다. 바이너리 크기, GPU 백엔드, 서빙 시나리오별 선택 기준을 코드 예시와 함께 정리했다."
---

# Rust가 LLM 추론 파이프라인에 들어오는 방식: Candle·Burn·tch-rs 실전 비교

LLM을 프로덕션에서 서빙할 때, Python 런타임이 주는 오버헤드는 점점 눈에 띄고 있다. 콜드 스타트 지연, GIL 경합, 의존성 충돌, 컨테이너 이미지 크기 — 이런 문제들은 트래픽이 늘고 엣지 배포 요구가 커질수록 더 거슬린다. 그 사이 Rust 기반 ML 도구들은 조용히 성숙해졌다. HuggingFace Tokenizers가 Rust 코어를 채택한 지 수년이 지났고, 이제는 추론 런타임 자체도 Rust로 작성된 선택지가 실전에서 쓰이고 있다.

이 글에서는 현재 가장 활발히 유지되는 세 Rust ML 프레임워크 — Candle, Burn, tch-rs — 를 LLM 추론이라는 구체적 관점에서 비교한다. 각 도구의 설계 철학, GPU 지원 현황, 바이너리 크기, 그리고 어떤 서빙 시나리오에 맞는지를 정리한다.

## 왜 LLM 추론에 Rust인가

먼저 오해를 하나 짚자. Rust가 PyTorch를 대체하려는 게 아니다. 모델 훈련과 연구 실험에서 Python 생태계의 우위는 2026년 현재도 흔들리지 않는다. Rust가 파고드는 지점은 **핫 패스(hot path)** — 요청이 들어올 때마다 실행되는 토큰화, 전처리, 추론, 직렬화 계층이다.

이 계층에서 Rust가 주는 이점은 세 가지다:

- **예측 가능한 지연**: 가비지 컬렉터가 없으므로, p99 지연이 Python보다 안정적이다. 서빙에서 tail latency가 SLA를 결정할 때 이는 실질적 차이를 만든다.
- **단일 바이너리 배포**: Python 런타임과 의존성 트리를 통째로 컨테이너에 담을 필요 없이, 하나의 정적 바이너리로 배포할 수 있다. 서버리스 함수나 엣지 디바이스에서 특히 유리하다.
- **메모리 효율**: 할당을 세밀하게 제어할 수 있어, 메모리 사용량이 적고 스파이크가 적다.

HuggingFace Tokenizers는 이 접근의 가장 오래된 성공 사례다. Rust 코어에 Python 바인딩을 얹은 구조로, BPE/WordPiece 토큰화를 Python 순수 구현 대비 수십 배 빠르게 수행한다. 동일한 패턴 — Rust 코어, Python 바인딩 — 이 추론 런타임으로 확장되고 있다.

## Candle: 서버리스 추론을 겨냥한 HuggingFace의 Rust 런타임

### 설계 의도

Candle은 HuggingFace가 개발하는 Rust 네이티브 ML 프레임워크다. GitHub 저장소(`huggingface/candle`)는 20,500개 이상의 스타를 보유하고 있으며, 2026년 6월 현재 최신 버전은 0.11.0이다. crates.io 기준 월 약 78만 회 다운로드되며, 1,024개 이상의 크레이트가 의존하고 있다.

Candle의 README가 명시하는 핵심 동기는 두 가지다:

1. **바이너리 크기 축소** — PyTorch의 거대한 라이브러리 의존성 없이 추론 엔진을 작게 유지하여 서버리스 배포를 가능하게 한다.
2. **프로덕션에서 Python 제거** — 복잡한 워크플로에서 Python이 주는 오버헤드와 GIL 문제를 원천적으로 제거한다.

Candle은 PyTorch의 Rust 래퍼가 아니다. 텐서 연산 계층을 Rust로 처음부터 재구현했으며, CUDA와 Metal(Apple Silicon) 백엔드를 자체 구현했다. CPU, CUDA, Metal 세 가지 디바이스를 지원한다.

### 코드로 보는 Candle

Candle의 API는 의도적으로 PyTorch와 비슷하게 설계되었다. 기본 텐서 연산은 다음과 같다:

```rust
use candle_core::{Tensor, Device};

fn main() -> anyhow::Result<()> {
    let device = Device::Cpu;

    let a = Tensor::arange(0f32, 6f32, &device)?.reshape((2, 3))?;
    let b = Tensor::arange(0f32, 12f32, &device)?.reshape((3, 4))?;
    let c = a.matmul(&b)?;

    println!("Result:\n{}", c);
    Ok(())
}
```

LLM 추론을 위해서는 `candle-transformers` 크레이트를 사용한다. Llama, Whisper, Falcon 등 주요 transformer 모델의 구현체를 제공한다. `Cargo.toml`에 다음 의존성을 추가한다:

```toml
[dependencies]
candle-core = "0.11"
candle-nn = "0.11"
candle-transformers = "0.11"
tokenizers = "0.21"
safetensors = "0.4"
hf-hub = "0.3"
```

Hugging Face Hub에서 모델을 직접 다운로드해 추론하는 예시는 다음과 같다:

```rust
use candle_core::Device;
use candle_transformers::models::llama;
use hf_hub::api::sync::Api;
use tokenizers::Tokenizer;

fn main() -> anyhow::Result<()> {
    let device = Device::Cuda(0); // 또는 Device::Metal(0), Device::Cpu

    let api = Api::new()?;
    let repo = api.model("meta-llama/Llama-3.2-1B".to_string());

    let tokenizer = Tokenizer::from_file(repo.get("tokenizer.json")?)?;
    let weights = repo.get("model.safetensors")?;
    let config = llama::Config::default();

    let model = llama::Model::load(&device, &config, &weights)?;

    let encoded = tokenizer.encode("Rust로 LLM을 서빙하는 이유는", true)?;
    let tokens = encoded.get_ids().to_vec();

    // 추론 수행
    let logits = model.forward(&tokens, 0)?;
    let next_token = logits.argmax(candle_core::D::Minus1)?;

    println!("Next token id: {:?}", next_token);
    Ok(())
}
```

### Candle의 한계

Candle은 PyTorch의 완전한 대체가 아니다. 연산자 커버리지가 좁고, autograd는 초기 단계이며, 커스텀 커널 작성이 PyTorch C++ 확장보다 까다롭다. 하지만 LLM 추론처럼 필요한 연산자가 잘 정의된 영역에서는 프로덕션 사용이 가능하다.

## Burn: 백엔드 교환이 가능한 텐서 프레임워크

### 설계 의도

Burn은 Tracel-AI가 개발하는 순수 Rust 딥러닝 프레임워크다. 2026년 5월 현재 최신 버전은 0.21.0이며, crates.io에서 월 약 11만 회 다운로드된다. 306개 이상의 크레이트가 의존하고 있다.

Burn의 핵심 차별점은 **백엔드 추상화**다. `Backend` trait을 중심으로 설계되어, 동일한 모델 코드를 CUDA, ROCm, Metal, Vulkan, WebGPU, LibTorch 백엔드에서 실행할 수 있다. 백엔드는 런타임이 아닌 컴파일 타임에 결정된다.

Burn의 가장 독특한 설계는 **Autodiff 백엔드 데코레이터**다. Autodiff는 독립 백엔드가 아니라 다른 백엔드를 감싸는 데코레이터로 작동한다. 기본 백엔드를 Autodiff로 감싸면 자동 미분 기능이 투명하게 추가된다:

```rust
use burn::backend::{Autodiff, Wgpu};
use burn::tensor::{Distribution, Tensor};

fn main() {
    type Backend = Autodiff<Wgpu>;
    let device = Default::default();

    let x: Tensor<Backend, 2> = Tensor::random([2, 3], Distribution::Default, &device);
    let weights: Tensor<Backend, 2> = Tensor::random([3, 4], Distribution::Default, &device);

    let output = x.matmul(&weights);
    let grads = output.backward();

    println!("Gradients computed");
}
```

### Burn이 지원하는 백엔드

Burn의 백엔드 지원은 넓다:

| 백엔드 | Nvidia GPU | AMD GPU | Apple | Intel | Qualcomm | Wasm |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|
| CUDA | ✅ | — | — | — | — | — |
| ROCm | — | ✅ | — | — | — | — |
| Metal | — | — | ✅ | — | — | — |
| Vulkan | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| WebGPU | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| LibTorch | ✅ | ✅ | ✅ | — | — | — |

WebGPU 백엔드는 브라우저 내 실행까지 커버한다. Vulkan 백엔드는 다양한 GPU 벤더를 단일 코드 경로로 지원한다는 점에서 실용적이다.

### Burn의 현재 위치

Burn은 훈련 프레임워크를 지향하지만, 2026년 현재로는 소규모 모델(ResNet 수준) 훈련에 실용적이다. 대규모 LLM 훈련에서 PyTorch를 대체하기에는 아직 이르다. 다만, 추론 관점에서는 백엔드 교환 가능성이 큰 장점이다. 개발 환경에서는 Wgpu로 빠르게 프로토타입하고, 프로덕션에서는 CUDA나 LibTorch 백엔드로 전환하는 전략이 가능하다.

## tch-rs: PyTorch 모델을 Rust 서빙으로 옮기는 가장 짧은 경로

### 설계 의도

tch-rs는 PyTorch C++ API(LibTorch)의 Rust 바인딩이다. Laurent Mazare가 개발했으며(Candle의 핵심 기여자이기도 하다), 2026년 3월 출시된 0.24.0이 최신 버전이다.

tch-rs의 가치는 단순하다. **PyTorch로 훈련한 모델을 Rust 서빙 계층에서 재구현 없이 그대로 실행할 수 있다.** TorchScript나 `torch.jit.trace`로 직렬화한 모델을 tch-rs에서 로드하면 된다.

```rust
use tch::{Cuda, Device, Tensor};

fn main() -> anyhow::Result<()> {
    let device = Device::Cuda(0);

    let model = tch::CModule::load("traced_model.pt")?;
    let input = Tensor::from_slice(&[1.0f32, 2.0, 3.0]).to(device);

    let output = model.forward_ts(&[input]);
    println!("Output: {:?}", output);

    Ok(())
}
```

### tch-rs의 트레이드오프

tch-rs는 가장 빠르게 도입할 수 있는 선택지지만, 두 가지 비용이 있다:

1. **LibTorch 의존성**: CUDA 빌드 기준 LibTorch 공유 라이브러리가 약 2GB에 달한다. 이는 "10MB 바이너리를 Lambda에 배포"라는 목표와 정면으로 충돌한다.
2. **Rust 이점의 일부 상실**: 실제 연산은 LibTorch C++ 코드에서 수행되므로, Rust의 메모리 안전성과 단일 바이너리 배포 이점을 온전히 누리지 못한다.

따라서 tch-rs는 "장기 실행 컨테이너에서 PyTorch 모델을 서빙하되 Python 런타임은 빼고 싶을 때" 적합하다. 서버리스나 엣지 배포에는 무겁다.

## 세 프레임워크 선택 기준

지금까지의 내용을 실전 의사결정 기준으로 정리한다:

| 조건 | 추천 | 이유 |
|------|------|------|
| 서버리스/엣지 배포, 작은 바이너리 | **Candle** | ~40MB 이하 바이너리, Python 의존성 없음 |
| 다양한 GPU 백엔드 지원 필요 | **Burn** | CUDA, ROCm, Metal, Vulkan, WebGPU 전부 지원 |
| 기존 PyTorch 모델을 빠르게 Rust 서빙 | **tch-rs** | 모델 재구현 불필요, LibTorch 직접 호출 |
| 브라우저 내 추론 | **Burn (WebGPU)** | Wasm + WebGPU 백엔드 |
| LLM 추론 (Llama, Whisper 등) | **Candle** | `candle-transformers`가 주요 모델 구현 제공 |
| 훈련까지 Rust에서 | **Burn** | 유일하게 훈련 프레임워크 지향 (아직 소규모 한계) |

중요한 점은 이 세 가지가 배타적 선택이 아니라는 것이다. 동일한 파이프라인에서 Tokenizers(Rust)로 토큰화하고, Candle로 추론하고, 결과를 Rust 웹 프레임워크(axum 등)로 응답하는 구성이 자연스럽다. Python은 모델 훈련과 실험에 남기고, 서빙 핫 패스만 Rust로 교체하는 하이브리드 접근이 2026년 현재 가장 현실적이다.

## 결론

Rust가 ML 인프라에 들어오는 방식은 한꺼번에 Python을 대체하는 게 아니라, 성능에 민감한 계층부터 조용히 교체하는 것이다. 세 프레임워크를 요약하면:

- **Candle** — HuggingFace가 만든 서버리스 추론 런타임. 작은 바이너리, LLM 모델 지원, CPU/CUDA/Metal 백엔드. LLM 추론 서빙의 첫 선택지.
- **Burn** — 백엔드 교환이 가능한 텐서 프레임워크. 가장 넓은 GPU 지원(CUDA, ROCm, Metal, Vulkan, WebGPU). 훈련까지 목표하지만 아직 초기.
- **tch-rs** — LibTorch 바인딩. 기존 PyTorch 모델을 재작성 없이 Rust 서빙으로 옮길 때. 단, LibTorch 의존성으로 바이너리가 큼.

Rust 추론 스택을 처음 도입한다면, Candle로 소규모 LLM 서빙을 시작해보길 권한다. `candle-transformers` 예제 중 하나를 골라 로컬에서 실행해보면, Rust 바이너리 하나로 LLM 추론이 가능하다는 것을 직접 확인할 수 있다. 거기서부터 백엔드 교환이나 브라우저 실행이 필요해지면 Burn으로 확장하는 경로가 자연스럽다.
