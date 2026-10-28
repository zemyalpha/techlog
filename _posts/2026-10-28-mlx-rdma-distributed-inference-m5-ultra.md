---
title: "816B 모델을 책상 위에서 돌리는 법 — MLX RDMA 분산 추론과 M5 Ultra 벤치마크 완전 분석"
date: "2026-10-28"
keywords: ["MLX", "M5 Ultra", "Mac Studio", "RDMA", "Thunderbolt 5", "로컬 LLM", "분산 추론"]
lang: "ko"
description: "Apple이 공개한 MLX + RDMA over Thunderbolt 분산 추론 벤치마크를 분석한다. 816B MoE 모델이 4대의 Mac Studio에서 초당 36 토큰으로 도는 구조, mlx-lm 실전 명령어, 비용과 전력 관점까지."
---

# 816B 모델을 책상 위에서 돌리는 법 — MLX RDMA 분산 추론과 M5 Ultra 벤치마크 완전 분석

2026년 10월 15일, Apple 머신러닝 리서치 팀은 꽤 도발적인 결과를 발표했다. 총 8,160억 개 파라미터의 MoE(믹스처 오브 익스퍼트) 모델을 **Mac Studio 4대**로 구성한 클러스터에서 초당 36 토큰 이상의 속도로 구동했다는 것. 데이터센터 HBM 서버 수백 대가 아니라, 책상 위의 미니 팩토리 네 대로 프론티어급 모델을 프라이빗하게 돌리는 시대가 열리고 있다.

이 글에서는 이 벤치마크의 수치를 하나씩 뜯어보고, mlx-lm으로 직접 재현하는 방법, 그리고 "과연 이게 GPU 서버 대비 현실적인 대안인가"를 전력·비용 관점에서 따져본다.

## 1. 왜 갑자기 Mac이 LLM 머신이 되었나 — 두 가지 하드웨어 변화

이번 결과는 마법이 아니라 두 가지 구체적인 하드웨어 변화의 산물이다.

**첫째, M5 세대의 GPU Neural Accelerator.** M5 칩에 들어간 Neural Accelerator는 행렬 곱셈(matrix multiplication) 전용 유닛이다. LLM 추론에서 첫 토큰까지의 시간(TTFT)은 연산량에 병목이 걸리는데, 이 전용 유닛이 딱 그 부하를 받아낸다. MLX는 Metal 4의 TensorOps와 Metal Performance Primitives 프레임워크를 통해 이 유닛을 활용한다. Apple의 2025년 11월 벤치마크에서도 MacBook Pro M5(24GB)는 M4 대비 TTFT가 **3.3~4.1배** 빨라졌다. 14B 밀집 모델의 첫 토큰이 10초 안에, 30B MoE는 3초 안에 나오는 수준이다.

**둘째, 메모리 대역폭의 도약.** 토큰 생성(decode) 속도는 연산이 아니라 메모리 대역폭에 묶인다. Mac Studio M5 Ultra는 819GB/s였던 M3 Ultra 대비 약 47% 늘어난 **1.2TB/s**의 통합 메모리 대역폭을 제공한다. Apple의 측정에서 M5 Ultra는 M3 Ultra 대비 decode가 1.6배 빨랐다. 대역폭 증가율(약 1.46배)과 실제 속도 향상(1.6배)이 거의 정비례한다는 점이, "decode는 메모리 바운드다"라는 LLM 추론의 기본 공식을 그대로 보여주는 사례다.

여기에 통합 메모리 구조가 더해진다. Mac Studio M5 Ultra는 최대 512GB의 통합 메모리를 GPU가 전부 쓸 수 있다. VRAM 512GB짜리 GPU 서버를 떠올려보면 가격이 어느 정도인지 감이 온다.

## 2. 핵심 벤치마크 수치 뜯어보기

Apple이 공개한 측정 환경은 다음과 같다. 모든 수치는 `mlx_lm.benchmark`로 측정했으며, 컨텍스트 8,192토큰과 65,536토큰 두 가지 설정을 사용했다.

| 모델 | 총 파라미터 | 활성 파라미터 | 평가 정밀도 |
|------|------------|--------------|------------|
| Muse Spark Open (MoE) | 816B | 42B | NVFP4 |
| Mistral Medium 3.5 (밀집) | 128B | 128B | FP8 |
| Qwen 3.8 (밀집) | 27B | 27B | BF16 |

눈여겨볼 지점 세 가지:

**MoE의 실용성.** 816B 모델이 돌아가는 이유는 추론 시 활성 파라미터가 42B에 불과하기 때문이다. 가중치가 통합 메모리에 다 들어가기만 하면(4비트 NVFP4 양자화로 압축), 연산 부하는 42B 모델 수준으로 줄어든다. 512GB 한 대로는 부족해 4대 분산이 필요했지만, 그 대가로 프론티어급 품질을 로컬에서 얻는다.

**큰 컨텍스트에서도 실용적.** 65,536 토큰 컨텍스트에서 Qwen 3.8은 최대 68GB, Mistral Medium 3.5는 158GB의 메모리를 사용했다. 둘 다 512GB Mac Studio 한 대에서 여유 있게 돌아간다. 에이전트 코딩처럼 긴 컨텍스트를 물고 늘어지는 워크로드에 유리한 구성이다. 그리고 이런 대형 컨텍스트에서 M5 Ultra의 TTFT 우위가 최대 4.6배까지 벌어진다 — 긴 프롬프트 프리필이 연산 병목이기 때문이다.

**전력 효율.** 피크 전력이 M3 Ultra Mac Studio는 약 200W, M5 Ultra는 약 300W다. Apple은 이를 "일반적인 토스터기가 쓰는 전력의 일부"라고 표현했다. HBM GPU 서버 한 대가 kW 단위를 소비하는 것과 비교하면 한 자릿수 차이며, 4대 클러스터 전체도 1.2kW 수준에 그친다.

## 3. 직접 해보기 — mlx-lm 실전 가이드

재현 장벽은 생각보다 낮다. 전부 오픈소스로 공개되어 있다.

**Step 1: 설치 및 단일 모델 실행**

```bash
pip install mlx-lm

# Hugging Face 모델과 바로 채팅
mlx_lm.chat --model mlx-community/Qwen3-8B-4bit
```

**Step 2: 벤치마크 재현** — Apple 포스팅과 동일한 방식:

```bash
mlx_lm.benchmark --model Qwen/Qwen3.8-27B -p 8192 -g 128 -b 1
# -p: 프롬프트 길이, -g: 생성 토큰 수, -b: 배치 크기
```

**Step 3: 직접 양자화** — Hugging Face 모델을 MLX용 4비트로 변환:

```bash
mlx_lm.convert \
  --hf-path mlx-community/Mistral-7B-Instruct-v0.3 \
  -q \
  --upload-repo my-org/mistral-7b-4bit
```

**Step 4: 멀티 Mac 분산 (RDMA over Thunderbolt)** — MLX 최신 버전에 포함된 RDMA over Thunderbolt 드라이버로 Mac 여러 대를 Thunderbolt 5 케이블로 연결하면, 파이프라인 병렬화 또는 텐서 병렬화로 모델을 쪼개 실행할 수 있다. Apple의 측정에서 816B MoE는 텐서 병렬화 4대 구성에서 초당 36 토큰 이상을 기록했다. 사람의 평균 독서 속도보다 빠르다.

## 4. 그래서 GPU 서버를 버려야 하나? — 현실적인 판단 기준

장밋빛만 있는 것은 아니다. 냉정하게 조건을 나눠보자.

**Mac Studio + MLX가 유리한 경우:**
- 데이터가 절대 밖으로 나가면 안 되는 의료·법무·금융 워크로드 (프라이빗 추론)
- 에이전트 코딩처럼 긴 컨텍스트 + 낮은 동시성(배치 1) 패턴
- 전기·냉각 인프라가 약한 사무실 환경, 24시간 재택 워크스테이션
- 실험과 파인튜닝을 조용히 반복하는 리서치 환경

**여전히 GPU 서버가 유리한 경우:**
- 높은 동시성 서빙. 배치 크기를 키워 처리량을 뽑는 서비스에는 NVIDIA의 텐서코어 + HBM 조합이 앞선다
- 학습(파인튜닝 이상). 대규모 학습은 여전히 데이터센터 GPU의 영역이다
- 초저지연 TTFT가 SLA인 실시간 서비스

또 한 가지 기억할 점: decode 속도가 대역폭에 비례한다는 사실은 역으로 153GB/s 수준의 MacBook Pro(M5)에서는 큰 밀집 모델이 느리다는 뜻이기도 하다. 노트북은 30B급 MoE 양자화 모델까지가 실용적 영역이고, 밀집 100B급 이상은 워크스테이션의 영역이다.

## 결론

- Apple의 10월 벤치마크는 816B MoE 모델을 4대의 Mac Studio(초당 36+ 토큰)로 구동했고, 이는 공식 1차 출처로 확인된 수치다
- 성능의 핵심은 M5의 GPU Neural Accelerator(TTFT 최대 4.6배)와 1.2TB/s 통합 메모리 대역폭(decode 1.6배)이다
- `mlx_lm.chat`, `mlx_lm.benchmark`, `mlx_lm.convert` 명령 세 개면 누구나 오늘 재현할 수 있다
- 다중 Mac 분산은 RDMA over Thunderbolt로 해결됐고, 전력은 4대 클러스터 합계 1.2kW 수준이다
- 단, 고동시성 서빙과 학습은 여전히 데이터센터 GPU의 영역 — "로컬 프라이빗 추론"이 이 조합의 정확한 포지션이다

당장 시도해볼 첫 단계는 간단하다. 맥이 있다면 `pip install mlx-lm` 후 4비트 양자화된 8B 모델 하나를 돌려보는 것. 자기 하드웨어가 어느 지점까지 버틸 수 있는지 감을 잡는 데 10분이면 충분하다.

**참고 자료:**
- Apple ML Research, "Local and Distributed Large LLM Inference with MLX and RDMA on Mac Studio" (2026-10-15) — https://machinelearning.apple.com/research/mlx-distributed-inference
- Apple ML Research, "Exploring LLMs with MLX and the Neural Accelerators in the M5 GPU" (2025-11-19) — https://machinelearning.apple.com/research/exploring-llms-mlx-m5
- Apple Newsroom, "Apple introduces new Mac Studio with M5 Max and M5 Ultra" (2026-08) — https://www.apple.com/newsroom/2026/08/apple-introduces-new-mac-studio-with-m5-max-and-m5-ultra/
- mlx-lm GitHub — https://github.com/ml-explore/mlx-lm
