---
title: "온디바이스 LLM이 서버를 대체할 수 있는가: Gemma 4·MLX M5·NPU 가속의 현실적 한계와 가능"
date: "2026-07-14"
keywords: ["온디바이스 LLM", "Gemma 4", "Gemini Nano", "Apple MLX", "NPU 가속", "edge inference"]
lang: "ko"
description: "Android AICore Gemma 4와 Apple MLX M5 벤치마크로 보는 2026년 온디바이스 LLM 추론의 현재 위치 — 무엇이 가능하고 어디까지 한계인지 구체적 수치로 분석한다."
---

# 온디바이스 LLM이 서버를 대체할 수 있는가: Gemma 4·MLX M5·NPU 가속의 현실적 한계와 가능

2026년 7월, 온디바이스 LLM은 "가능한 이야기"에서 "쓸모 있는 도구"로 넘어가는 전환점에 있다. Google이 AICore Developer Preview로 Gemma 4를 공개하고, Apple이 MLX 프레임워크로 M5의 Neural Accelerator를 활용한 LLM 추론 벤치마크를 발표했다. Qualcomm은 모바일 CPU에서 Llama 계열 모델을 가속하는 QMX(Qualcomm Matrix Extension)를 선보였다. 서버 없이 폰과 노트북에서 모델이 돈다는 것 자체는 더 이상 뉴스가 아니다. 진짜 질문은 "어떤 작업까지 온디바이스에서 처리할 수 있는가"이다.

이 글에서는 2026년 중반 공개된 세 가지 데이터 — Android AICore의 Gemma 4, Apple MLX의 M5 벤치마크, 모바일 CPU/NPU 추론의 메모리 한계 — 를 교차 검증하며 온디바이스 LLM이 실제로 어디에 쓸모 있고 어디서 여전히 서버가 필요한지 정리한다.

## 핵심 배경: 온디바이스 LLM이 "빠르게" 발전하는 이유

온디바이스 추론이 2026년 의미 있는 단계에 들어선 데는 세 가지 기술적 요인이 겹쳤다.

**첫째, 소형 모델의 품질 상승이다.** 1~4B 파라미터 모델이 구조적 추론·수학·OCR 등에서 실사용 가능한 수준에 도달했다. Google의 Gemma 4 E2B(최적화된 속도형)와 E4B(추론 중심)는 140개 이상 언어를 네이티브로 지원하고, 텍스트·이미지·오디오를 멀티모달로 처리한다. 1~4B 크기에서 이 수준의 다국어·멀티모달 능력은 2년 전에는 서버급 모델에서나 기대할 수 있었다.

**둘째, 하드웨어 가속기의 일반화다.** Apple M5의 Neural Accelerator, Qualcomm Snapdragon의 NPU, Google Tensor의 TPU — 이 세 가지가 공통적으로 "행렬 곱셈에 특화된 전용 연산 유닛"을 제공한다. LLM 추론에서 가장 비싼 연산이 바로 행렬 곱셈이므로, 이 연산을 전용 하드웨어로 오프로드하면 성능이 극적으로 올라간다.

**셋째, 양자화 기술의 성숙이다.** 4-bit 양자화로 모델 크기를 BF16 대비 약 4배 줄이면서도 품질 저하가 미미하다는 것이 검증되었다. 덕분에 24GB 메모리 노트북에서 30B MoE 모델(4-bit)이 18GB 이내에 들어가고, 모바일 기기에서도 2~8B 모델을 실행할 수 있게 되었다.

## Android AICore: Gemma 4와 Gemini Nano의 경로

Google은 2026년 4월 Android Developers Blog에서 AICore Developer Preview를 통해 Gemma 4를 공개했다. 이 모델은 Gemini Nano 4의 기반이 되며, AICore Developer Preview에서 테스트한 코드는 향후 Gemini Nano 4가 탑재된 기기에서 그대로 동작한다고 한다.

Gemma 4 on Android는 두 가지 사이즈로 제공된다:

- **E4B**: 더 강력한 추론을 위한 모델. 복잡한 태스크에 적합.
- **E2B**: 최대 속도에 최적화. Google 발표에 따르면 E4B 대비 **3배 빠른** 추론 속도.

Google의 공식 발표에서 강조하는 성능 개선은 두 가지다:

> "The new model is up to **4x faster** than previous versions and uses up to **60% less battery**."

또한 Gemma 4는 140개 이상 언어를 네이티브로 지원하며, 멀티모달 이해(텍스트, 이미지, 오디오)를 제공한다. 실제 개발자 코드는 다음과 같은 구조로 작성된다:

```kotlin
// AICore Developer Preview — Gemma 4 모델 설정
val previewConfig = generationConfig {
    modelConfig = ModelConfig {
        releaseTrack = ModelReleaseTrack.PREVIEW
        preference = ModelPreference.FULL  // E4B (full) 또는 E2B (fast)
    }
}

val previewModel = GenerativeModel.getClient(previewConfig)

// 모델 가용성 확인 후 추론
val status = previewModel.checkStatus()
if (status == FeatureStatus.AVAILABLE) {
    val response = previewModel.generateContent(
        "26 paychecks per year로 $10,000 저축하려면 매번 얼마를 저금해야 하나?"
    )
}
```

AICore의 중요한 점은 추론이 "특정 하드웨어"에 종속되지 않는다는 것이다. Google, MediaTek, Qualcomm의 최신 AI 가속기가 탑재된 기기에서는 하드웨어 가속으로 실행되고, 그렇지 않은 기기에서는 CPU 폴백으로 동작한다(다만 CPU 모드는 "최종 프로덕션 성능을 대표하지 않는다"고 명시되어 있다).

## Apple MLX + M5 Neural Accelerator: 노트북에서 30B 모델까지

Apple은 2025년 10월 M5 칩을 탑재한 MacBook Pro를 발표하며, MLX 프레임워크가 M5의 Neural Accelerator를 활용한다고 밝혔다(MLX 연구 글은 같은 해 11월에 게재). 핵심은 M5 GPU에 추가된 "Neural Accelerator"가 행렬 곱셈에 특화된 전용 연산을 제공한다는 것이다.

Apple의 공식 MLX 벤치마크(MacBook Pro 24GB, M5 vs M4)에서 주목할 수치들:

| 모델 | M5 vs M4 TTFT 향상 | 생성 속도 향상 | 메모리 (GB) |
|------|-------------------|---------------|------------|
| Qwen3-1.7B BF16 | 3.57x | 1.27x | 4.40 |
| Qwen3-8B BF16 | 3.62x | 1.24x | 17.46 |
| Qwen3-8B 4-bit | 3.97x | 1.24x | 5.61 |
| Qwen3-14B 4-bit | 4.06x | 1.19x | 9.16 |
| GPT-OSS-20B MXFP4 | 3.33x | 1.24x | 12.08 |
| Qwen3-30B-A3B MoE 4-bit | 3.52x | 1.25x | 17.31 |

이 데이터가 중요한 이유는 두 가지 성능 병목을 명확히 보여주기 때문이다.

**첫 번째 병목: Time-To-First-Token(TTFT)** — 첫 토큰을 생성하기 전까지는 연산량이 크므로 compute-bound다. Neural Accelerator가 이 단계를 3.3~4.06배 가속한다. 14B 모델의 프롬프트 처리(4096 토큰)를 10초 이내에 끝내고, 30B MoE는 3초 이내에 첫 토큰을 낸다.

**두 번째 병목: 생성 속도(tokens/s)** — 첫 토큰 이후에는 메모리 대역폭이 병목이다. 베이스 M4는 120GB/s, M5는 153.6GB/s(LPDDR5X 9600MT/s)로 약 28% 높아졌으며, 이것이 그대로 19~27%의 생성 속도 향상으로 이어진다. Neural Accelerator가 아무리 빨라도 생성 단계에서는 메모리 대역폭이 벽이 된다. (Apple은 M5의 전체 AI 성능이 전세대 대비 최대 3.5배라고 발표했다.)

MLX의 사용은 매우 간단하다:

```bash
# MLX 설치 및 모델 로드
pip install mlx-lm

# Hugging Face 모델 4-bit 양자화 및 업로드
mlx_lm.convert \
    --hf-path Qwen/Qwen3-8B \
    -q \
    --upload-repo mlx-community/Qwen3-8B-4bit

# 채팅 시작
mlx_lm.chat --model mlx-community/Qwen3-8B-4bit
```

MLX의 강력한 점은 Apple Silicon의 "통합 메모리(Unified Memory)" 구조를 활용한다는 것이다. CPU와 GPU가 같은 메모리를 공유하므로 데이터 복사 오버헤드가 없다. 24GB 메모리 노트북에서 8B BF16(17.5GB)이나 30B MoE 4-bit(17.3GB)를 실행할 수 있는 이유다.

## NPU 가속의 현실: 모바일 CPU에서 Llama가 돈다

Qualcomm은 Snapdragon 모바일 CPU에서 Llama 계열 모델의 성능을 높이기 위해 QMX(Qualcomm Matrix Extension)를 개발했다. ARM 기반 모바일 CPU는 기본적으로 행렬 곱셈에 비효율적인데, QMX는 NEON 명령어를 확장하여 저비트 양자화된 행렬 연산을 가속한다.

한편, llama.cpp는 크로스 플랫폼 C/C++ 기반 추론 엔진으로 GGUF 형식의 양자화 모델을 실행한다. 모바일(Android/iOS), 데스크톱, 서버 모두에서 동작하며, 가장 널리 쓰이는 온디바이스 LLM 실행 경로다.

```bash
# llama.cpp로 모바일 기기에서 Qwen3-1.7B 실행 (Android Termux)
# 1. 모델 다운로드 (4-bit 양자화 GGUF)
wget https://huggingface.co/Qwen/Qwen3-1.7B-GGUF/resolve/main/qwen3-1.7b-q4_k_m.gguf

# 2. llama.cpp CLI로 추론
./llama-cli \
    -m qwen3-1.7b-q4_k_m.gguf \
    -p "온디바이스 LLM의 장점 3가지를 설명해줘" \
    -n 256 \
    -t 4  # CPU 스레드 수
```

모바일 환경에서 1~2B 모델은 일반적으로 초당 10~30 토큰 수준의 생성 속도를 보인다(기기 및 양자화 수준에 따라 편차 큼). 체감상 읽는 속도와 비슷하므로 채팅형 인터페이스에서 사용 가능한 수준이다.

## 온디바이스 vs 서버: 무엇을 옮기고 무엇을 남길 것인가

이제 실전적인 질문으로 돌아오자. 온디바이스 LLM이 실제로 서버를 대체할 수 있는 영역은 어디까지인가?

**온디바이스에 적합한 작업:**

- **요약 및 분류** — 텍스트/이미지 분류, 댓글 검열, 감정 분석. Gemma 4의 예시에서 보듯 "이 댓글이 커뮤니티 가이드라인을 통과하는가?" 같은 판단 작업
- **개인정보가 민감한 작업** — 건강 데이터, 메시지 내용, 사진 OCR. 서버로 데이터를 보내지 않는 것이 핵심 가치
- **오프라인 작업** — 비행기 모드, 네트워크 불안정 환경에서의 기본 AI 기능
- **단순 수학/시간 추론** — "월 26회 급여로 연 $10,000 저축하려면 한 번에 얼마?" 같은 계산
- **저지연 인터페이스** — 실시간 자동완성, 음성 명령 처리. 왕복 네트워크 지연(100~300ms) 없이 즉시 응답

**서버가 여전히 필요한 작업:**

- **복잡한 추론 및 코딩** — 1~4B 모델은 간단한 논리에는 강하지만, 다단계 추론이나 코드 생성에서는 70B+ 모델에 비해 품질이 떨어진다
- **대규모 RAG** — 수백만 건의 문서 검색+생성은 메모리와 연산량 면에서 기기 밖에서 처리해야 한다
- **도구 호출 오케스트레이션** — 여러 API를 연속 호출하며 중간 결과를 추적하는 작업은 더 큰 컨텍스트 윈도우가 필요하다
- **최신 지식이 필요한 작업** — 온디바이스 모델은 학습 시점으로 고정되므로 실시간 정보는 서버가 필요하다

핵심 원칙은 **"지연/프라이버시가 중요하면 기기로, 품질/지식이 중요하면 서버로"**다.

## 실전 아키텍처: 하이브리드 라우팅

가장 현실적인 접근은 온디바이스와 서버를 작업 특성에 따라 라우팅하는 것이다:

```python
# 하이브리드 라우팅 의사코드
def route_request(query, context):
    # 1. 프라이버시 민감 데이터 감지
    if contains_pii(context):
        return on_device_inference(query, model="gemma4-e2b")

    # 2. 오프라인 상태
    if not network_available():
        return on_device_inference(query, model="gemma4-e4b")

    # 3. 복잡도 추정 (토큰 수, 도구 호출 필요성 등)
    complexity = estimate_complexity(query)
    if complexity < THRESHOLD_SIMPLE:
        # 단순 작업은 온디바이스에서 즉시 응답
        return on_device_inference(query, model="gemma4-e2b")
    else:
        # 복잡한 작업은 서버로 (70B+ 모델)
        return server_inference(query, model="claude-opus")
```

이 패턴의 장점은 세 가지다. 첫째, 단순 작업(약 60~70%의 요청)을 온디바이스에서 처리하여 서버 비용을 크게 줄인다. 둘째, 프라이버시 민감 데이터는 절대 서버로 가지 않는다. 셋째, 네트워크 장애 시에도 기본 기능이 동작한다.

## 메모리 예산 계산: 내 기기에서 무엇이 돌아가는가

온디바이스 LLM을 도입하기 전에 반드시 계산해야 할 것은 "메모리 예산"이다. Apple MLX 벤치마크에서 확인된 값들을 기준으로 정리하면:

| 모델 크기 | 정밀도 | 메모리 | 실행 가능 기기 |
|----------|--------|--------|--------------|
| 1~2B | 4-bit | 1~2GB | 대부분의 모바일/태블릿 |
| 7~8B | 4-bit | 4~6GB | 최신 플래그십 폰, M1+ Mac |
| 7~8B | BF16 | 14~18GB | M2+ Mac (16GB+) |
| 13~14B | 4-bit | 8~10GB | 최신 태블릿, M1+ Mac |
| 20~30B MoE | 4-bit | 12~18GB | M2+ Mac (24GB+) |

주의할 점은 이 메모리가 "모델 전용"이라는 것이다. 실제로는 OS, 앱, KV 캐시(컨텍스트 길이에 비례)가 추가로 필요하므로, 실사용 가능한 모델 크기는 이론적 최대치보다 한 단계 작아진다.

## 결론: 2026년 온디바이스 LLM의 현재 위치

2026년 중반, 온디바이스 LLM은 명확한 위치를 잡았다.

- **Gemma 4 / Gemini Nano 4**는 1~4B 크기에서 다국어·멀티모달·기본 추론을 제공하며, 이전 세대 대비 최대 4배 빠르고 60% 적은 배터리를 소모한다. Android AICore를 통해 개발자가 테스트할 수 있는 환경이 갖춰졌다.
- **Apple MLX + M5 Neural Accelerator**는 노트북에서 30B MoE 모델을 실행 가능하게 만들었고, TTFT에서 3~4배 가속을 달성했다. 다만 생성 속도는 메모리 대역폭(M5 153GB/s)에 의해 제약된다.
- **하이브리드 라우팅**이 현실적 최선책이다. 단순/프라이버시/저지연 작업은 기기에서, 복잡 추론/대규모 검색/최신 지식은 서버에서 처리하는 구조.

온디바이스 LLM이 서버를 "완전히" 대체할 수는 없다. 하지만 트래픽의 상당 부분을 기기로 옮기고, 서버는 진짜로 큰 모델이 필요한 작업에만 집중시키는 구조는 이미 충분히 실현 가능하다. 당장 시도해볼 첫 단계는 간단하다:

1. **MLX 또는 llama.cpp를 설치**하고, 4-bit 양자화된 7~8B 모델을 로컬에서 실행해본다.
2. **애플리케이션의 요청을 복잡도별로 분류**하여, 단순 요청의 비율을 측정한다.
3. **단순 요청을 온디바이스로 라우팅**하는 프로토타입을 만들고, 비용 절감과 지연 감소를 측정한다.

서버가 사라지는 것이 아니라, 서버가 "필요한 순간에만" 쓰이는 시설이 되는 것이다.
