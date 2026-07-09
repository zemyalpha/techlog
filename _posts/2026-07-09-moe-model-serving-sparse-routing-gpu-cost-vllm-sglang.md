---
title: "MoE 모델을 프로덕션에 서빙할 때: 희소 라우팅이 GPU 비용 구조를 바꾸는 방식"
date: "2026-07-09"
keywords: ["Mixture of Experts", "MoE 서빙", "DeepSeek V3", "vLLM", "SGLang", "expert parallelism"]
lang: "ko"
description: "MoE 모델이 2026년 오픈웨이트 표준이 된 지금, 희소 라우팅이 추론 비용과 VRAM 요구사항을 어떻게 재구성하는지, 그리고 vLLM과 SGLang 중 무엇을 선택해야 하는지를 정리한다."
---

# MoE 모델을 프로덕션에 서빙할 때: 희소 라우팅이 GPU 비용 구조를 바꾸는 방식

2024년 말 DeepSeek V3가 등장했을 때, 혼자 서버에 올려 쓰던 엔지니어들이 가장 먼저 부딪힌 문제는 "671B 파라미터인데 왜 VRAM이 이렇게 많이 필요하지?"였다. 활성 파라미터는 37B에 불과한데, 모든 전문가(expert) 가중치를 메모리에 올려야 하니 실제 VRAM 요구량은 700B 급 밀집 모델과 비슷했다. 바로 이 지점이 Mixture-of-Experts(MoE) 모델을 프로덕션에 서빙할 때 겪는 가장 근본적인 딜레마다.

이 글에서는 MoE 아키텍처가 추론 비용 구조를 어떻게 바꾸는지, 라우팅 전략에 따라 서빙 난이도가 어떻게 달라지는지, 그리고 2026년 현재 사용할 수 있는 서빙 프레임워크 중 어떤 것을 선택해야 하는지를 정리한다.

## MoE가 비용 구조를 바꾸는 핵심 메커니즘

### 밀집 모델과의 결정적 차이: 활성 파라미터 vs 총 파라미터

표준 트랜스포머에서 모든 토큰은 동일한 피드포워드 레이어를 통과한다. 70B 밀집 모델이라면, 모든 토큰이 70B 파라미터 전체를 거친다. 한 토큰을 처리하는 데 70B 분량의 연산이 필요하다.

MoE 레이어는 이 단일 피드포워드 블록을 여러 개의 병렬 "전문가"와 작은 라우터 네트워크로 교체한다. 각 토큰마다 라우터가 어떤 전문가(일반적으로 수십~수백 개 중 1~8개)를 활성화할지 결정하고, 선택된 전문가만 연산에 참여한다.

결과적으로 1T 파라미터 MoE 모델이 32B 활성 파라미터를 가진다면, 토큰 생성 속도와 GPU 활용률은 대략 32B 밀집 모델과 비슷하다. 하지만 모델은 1T 파라미터의 용량을 사용할 수 있고, 라우터는 서로 다른 유형의 토큰을 서로 다른 특화된 전문가로 보낸다.

**핵심 트레이드오프**: 총 메모리 사용량은 활성 파라미터가 아닌 총 파라미터에 비례한다. 모든 전문 가중치를 VRAM에 올려야 하지만, 토큰당 활성화되는 것은 그 중 일부다. 이것이 MoE 모델이 동일 추론 비용의 밀집 모델보다 더 많은 VRAM을 요구하는 이유다.

### 스파시티 비율: 2024년에서 2026년으로의 압축

모델 간 비교에서 가장 중요한 지표는 "스파시티 비율(sparsity ratio)"이다. 총 파라미터 대비 활성 파라미터의 비율로, 이 값이 낮을수록 토큰당 연산이 싸다.

| 모델 | 총 파라미터 | 활성 파라미터 | 스파시티 | 전문가 구성 |
|------|-----------|------------|---------|-----------|
| Mixtral 8x7B (2023) | 46.7B | 12.9B | ~28% | 8 experts, top-2 |
| Mixtral 8x22B (2024) | 141B | 39B | ~28% | 8 experts, top-2 |
| Qwen3-235B-A22B (2025) | 235B | 22B | ~9.4% | 128 experts, top-8 + shared |
| DeepSeek V3 (2024) | 671B | 37B | ~5.4% | 256+1 shared, top-8 |

> 위 수치는 각 모델의 공식 기술 보고서(arXiv 2412.19437 DeepSeek V3, arXiv 2505.09388 Qwen3, Mistral 공식 모델 카드)에 근거한다.

Mixtral 시대에는 8개 전문가에 top-2 라우팅이 표준이었다. 2024년 말부터 DeepSeek V3와 Qwen3이 도입한 "fine-grained MoE" 패턴은 전문가 수를 크게 늘리고(128~256개), 각 전문가를 더 작게 만들고, 토큰당 더 많은 전문가를 활성화한다(top-8). 동시에 "공유 전문가(shared expert)"를 배치해 모든 토큰이 공통적으로 처리해야 할 패턴을 담당하게 한다. 이 설계는 동일 활성 파라미터 대비 더 나은 품질을 보여준다.

> 참고: 2026년 2분기 기준으로 일부 기술 블로그들은 DeepSeek V4-Pro가 1.6T 총 / 49B 활성(약 3.1% 스파시티)이라고 보도하고 있다. 하지만 이 수치는 공식 논문이 아닌 단일 소스에 근거하므로, 확인될 때까지 참고용으로만 활용한다.

## 라우팅 전략이 서빙 난이도를 결정한다

### Top-K 라우팅 (Mixtral 방식)

가장 단순한 방식이다. 각 토큰을 고정된 K개의 전문가에게 라우팅한다(Mixtral의 경우 K=2). 구현이 쉽고 병렬 처리가 가능하다. 하지만 부하가 불균형해지는 문제가 있다. 일부 전문가에 토큰이 몰리면, 그 전문가가 병목이 되어 전체 처리 속도가 느려진다.

이를 완화하기 위해 훈련 단계에서 "부하 균형 손실(load balancing loss)"을 추가해 라우터가 토큰을 고르게 분배하도록 유도한다.

### Fine-Grained 라우팅 (DeepSeek, Qwen 방식)

더 많은 전문가(128~256개)를 더 작은 크기로 만들고, 토큰당 더 많은 전문가(8개)를 활성화한다. 각 전문가가 더 좁은 영역에 특화되며, 공유 전문가가 공통 지식을 담당한다.

서빙 관점에서 이 방식은 새로운 문제를 만든다. 256개 전문가가 여러 GPU에 분산되어 있을 때, 토큰 하나가 활성화해야 하는 8개의 전문가가 서로 다른 GPU에 있을 수 있다. 이때 발생하는 GPU 간 통신(all-to-all communication)이 새로운 병목이다.

## 서빙 스택 선택: vLLM, SGLang, TensorRT-LLM

MoE 모델을 서빙하려면 전문가 병렬 처리(expert parallelism)를 지원하는 추론 엔진이 필수다. 2026년 현재 프로덕션에서 사용되는 주요 엔진 4종을 비교한다.

### vLLM: 기본 선택

UC Berkeley에서 시작된 오픈소스 추론 엔진이다. PagedAttention을 통해 KV 캐시를 페이지 단위로 관리해 메모리 단편화를 줄이고, continuous batching으로 요청을 동적으로 추가·제거한다.

```bash
# DeepSeek V3를 vLLM으로 서빙 (8x H100 필요)
pip install vllm

vllm serve deepseek-ai/DeepSeek-V3 \
    --tensor-parallel-size 8 \
    --max-model-len 65536 \
    --enable-expert-parallel \
    --gpu-memory-utilization 0.90
```

`--enable-expert-parallel` 플래그는 MoE 전문가를 GPU 간에 분산시킨다. 텐서 병렬화(TP)만으로는 전문가가 한 GPU에 몰리는 문제를 해결할 수 없기 때문에, MoE 모델에서는 이 옵션이 필수적이다.

vLLM의 강점은 새 모델 지원 속도와 하드웨어 호환성이다. NVIDIA, AMD, Intel GPU를 모두 지원하며, 새 모델이 발표되면 며칠 내로 지원이 추가된다.

### SGLang: 공유 접두사가 많은 워크로드에 유리

역시 UC Berkeley에서 나온 엔진으로, RadixAttention이라는 기술이 핵심이다. RadixAttention은 라딕스 트리(radix tree)를 기반으로 KV 캐시를 관리해, 여러 요청이 공통 접두사(시스템 프롬프트 등)를 공유할 때 KV 캐시를 재사용한다.

이 특성은 RAG 파이라인, 멀티턴 채팅, 에이전트 루프처럼 동일한 컨텍스트가 반복되는 워크로드에서 특히 효과적이다. 또한 JSON 스키마, 정규표현식 기반 구조화 출력(structured output)을 네이티브로 지원한다.

```python
# SGLang으로 MoE 모델 서빙 + 구조화 출력
# sglang 서버 실행
# python -m sglang.launch_server \
#     --model-path deepseek-ai/DeepSeek-V3 \
#     --tp 8 --expert-parallel-size 8

import openai

client = openai.Client(base_url="http://localhost:30000/v1", api_key="none")

response = client.chat.completions.create(
    model="default",
    messages=[
        {"role": "system", "content": "당신은 기술 분석 에이전트입니다."},
        {"role": "user", "content": "다음 텍스트를 JSON으로 구조화해라: ..."}
    ],
    extra_body={
        "regex": r'\{.*\}',  # 정규표현식으로 출력 형식 강제
    }
)
```

### TensorRT-LLM: NVIDIA 환경에서 최고 성능

NVIDIA가 개발한 엔진으로, 모델을 특정 GPU(H100, H200, Blackwell)에 최적화된 커널로 컴파일한다. NVIDIA 하드웨어에서 최고 처리량을 내지만, 컴파일 단계가 필요하고 NVIDIA GPU에서만 동작한다.

> 특정 벤치마크에서 TensorRT-LLM이 vLLM보다 약 10% 높은 처리량을 보인다고 보고된 바 있다(예: Llama-3-70B FP8 기준 단일 H200에서 추정치). 단, 이 수치는 하드웨어, 배치 크기, 모델에 따라 크게 변동하므로 자체 환경에서 벤치마크하는 것이 필수적이다.

## 선택 결정 트리

```
NVIDIA 전용, 최고 성능이 필수인가?
├── Yes → TensorRT-LLM (단, 컴파일 복잡도 감수)
└── No → 공유 접두사가 많은 워크로드인가? (RAG, 멀티턴, 에이전트)
    ├── Yes → SGLang (RadixAttention으로 KV 캐시 재사용)
    └── No → vLLM (가장 넓은 생태계, 가장 빠른 신모델 지원)
```

대부분의 팀에서 vLLM이 기본 선택이다. RAG나 에이전트 워크로드라면 SGLang을 시도해볼 만하다. TensorRT-LLM은 NVIDIA 클러스터에서 마지막 10~20% 성능이 필요할 때 선택한다.

## 프로덕션 배포 시 주의사항

### VRAM 계산: 활성이 아닌 총 파라미터 기준

가장 흔히 하는 실수는 "활성 파라미터가 37B니까 70B 밀집 모델과 비슷하게 서빙할 수 있겠지"라고 가정하는 것이다. DeepSeek V3(671B 총 / 37B 활성)를 FP8로 서빙하려면 약 700GB VRAM이 필요하다. H100 80GB 8장으로는 간신히 올라간다.

```python
# VRAM 요구량 추정 (FP8 기준, 대략적)
def estimate_moe_vram(total_params_b, quantization="fp8"):
    bytes_per_param = {"fp16": 2, "bf16": 2, "fp8": 1, "int4": 0.5}
    b = bytes_per_param.get(quantization, 2)
    weight_vram_gb = total_params_b * b
    # KV 캐시, 활성화 메모리 등 추가 (컨텍스트 길이에 따라 변동)
    overhead_gb = weight_vram_gb * 0.15  # 대략적 추정
    return weight_vram_gb + overhead_gb

# DeepSeek V3 FP8 예시
print(f"DeepSeek V3 FP8: ~{estimate_moe_vram(671, 'fp8'):.0f} GB")
# 출력: DeepSeek V3 FP8: ~772 GB → H100 80GB x 10장 또는 최적화 시 8장
```

### 전문가 병렬 처리와 GPU 간 통신

fine-grained MoE 모델에서 256개 전문가가 8개 GPU에 분산되어 있을 때, 토큰 하나가 활성화해야 하는 8개 전문가가 최대 8개의 서로 다른 GPU에 있을 수 있다. 각 토큰마다 라우팅 결정 후 해당 GPU로 데이터를 보내고(all-to-all), 연산 결과를 다시 받아와야 한다.

이 all-to-all 통신이 병목이 되면, 활성 파라미터는 적은데 실제 지연 시간이 길어진다. NVLink나 InfiniBand 같은 고속 인터커넥트가 필수적이며, 이것이 MoE 모델이 단일 노드보다 멀티 노드 클러스터에서 운영되어야 하는 이유 중 하나다.

### 부하 불균형 모니터링

top-K 라우팅에서 특정 전문가에 토큰이 몰리면, 해당 GPU가 병목이 된다. vLLM과 SGLang은 Prometheus 메트릭을 통해 전문가별 활성 빈도를 노출하므로, 이를 모니터링해야 한다.

실제 운영에서 부하 불균형이 심하면, 서빙 엔진의 전문가 배치 전략을 조정하거나 라우팅 임계값을 튜닝해야 한다. 일부 엔진은 "전문가 수용량(expert capacity)"을 설정해 한 전문가가 처리할 수 있는 최대 토큰 수를 제한하기도 한다.

## 결론: MoE 서빙은 연산 비용을 메모리 비용으로 교환하는 일

- **MoE는 토큰당 연산(FLOPs)을 70~95% 줄이지만, 총 파라미터 전체를 VRAM에 올려야 한다.** 연산 비용은 줄었지만 메모리 비용은 늘었다.
- **스파시티 비율이 핵심 비교 지표다.** Mixtral의 28%에서 DeepSeek V3의 5.4%로, 2년 만에 토큰당 연산 비용이 대폭 낮아졌다. 하지만 VRAM 요구량은 늘었다.
- **fine-grained 라우팅(128~256 전문가)은 더 나은 품질을 제공하지만, GPU 간 통신 비용을 만든다.** all-to-all 통신이 새로운 병목이다.
- **서빙 엔진 선택은 워크로드에 달려 있다.** vLLM(기본), SGLang(공유 접두사), TensorRT-LLM(NVIDIA 최고 성능).
- **가장 먼저 확인할 것은 VRAM 예산이다.** 활성 파라미터가 아니라 총 파라미터 기준으로 GPU를 계획해야 한다.

MoE 모델을 처음 서빙하는 팀이라면, 가장 작은 fine-grained MoE 모델인 Qwen3-30B-A3B(30B 총 / 3B 활성)로 시작하는 것을 권한다. 단일 GPU에서 실행 가능하면서도, fine-grained 라우팅의 모든 특성을 경험할 수 있다.
