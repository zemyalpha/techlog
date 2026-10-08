---
title: "구글이 NPU 설계도를 통째로 공개한 이유 — Coral NPU 아키텍처 해부와 2026 엣지 AI 실리콘 지형"
date: "2026-10-08"
keywords: ["Coral NPU", "RISC-V", "오픈소스 실리콘", "엣지 AI", "NPU", "Coralboard", "TinyGemma"]
lang: "ko"
description: "구글이 2025년 10월 오픈소스로 공개한 RISC-V 기반 Coral NPU의 아키텍처를 라인 단위로 해부하고, VeriSilicon·Synaptics·MIPS까지 2026년 엣지 AI 실리콘 생태계가 어떻게 재편되는지 분석한다."
---

# 구글이 NPU 설계도를 통째로 공개한 이유 — Coral NPU 아키텍처 해부와 2026 엣지 AI 실리콘 지형

2025년 10월 15일, 구글 리서치는 자사 NPU(신경처리장치) IP의 전체 설계를 Apache 2.0 라이선스로 공개했다. [google-coral/coralnpu](https://github.com/google-coral/coralnpu) 리포지토리에는 RTL(HDL) 코드부터 컴파일러 툴체인, 시뮬레이터, FPGA 검증 환경, UVM 테스트벤치까지 들어 있다. 흔히 '오픈소스'라고 하면 소프트웨어를 떠올리지만, 이건 회로 설계도 — 반도체 회사가 라이선스 비용을 받고 팔던 그 지식 자산 — 그 자체다.

이 글에서는 Coral NPU가 정확히 어떤 구조인지 공식 데이터시트와 리포지토리 기준으로 해부하고, 왜 구글이 이걸 무료로 풀었는지, 그리고 이 결정이 2026년 현재 엣지 AI 칩 시장을 어떻게 바꾸고 있는지 정리한다.

## Coral NPU는 정확히 무엇인가

Coral NPU는 웨어러블(이어버드, AR 글래스, 스마트워치)을 타깃으로 하는 초저전력 SoC에 통합하기 위한 ML 추론 가속기다. 첫인상은 "구글이 만든 또 다른 AI 가속기"지만, 설계 철학이 기존 NPU와 다르다.

핵심은 **C 프로그래밍이 가능한(progradable) 가속기**라는 점이다. 전통적인 NPU는 고정된 연산 그래프를 돌리는 전용 기계라 유연성이 없고, CPU는 유연하지만 AI 연산의 에너지 효율이 낮다. 구글의 공식 문서는 이 트레이드오프를 명시적으로 언급하며, Coral NPU는 이 둘 사이의 절충안 — 도메인 최적화된 명령어 집합을 갖추되 C 컴파일러로 프로그래밍할 수 있는 구조 — 을 택했다.

### 아키텍처: 하나의 칩, 세 개의 프로세서

Coral NPU는 단일 가속기가 아니라 세 개의 처리 컴포넌트가 하나의 파이프라인을 공유하는 구조다.

| 컴포넌트 | 역할 |
|---------|------|
| **Scalar** | RV32IMF 명령어를 실행하며 전체 흐름 제어. 백엔드(ML·SIMD)에 명령 큐를 공급 |
| **Vector (SIMD)** | 128-bit 벡터 파이프라인. FP32/FP16/BF16 지원, 32개의 128-bit 벡터 레지스터 |
| **Matrix** | 양자화 외적(outer-product) 기반 MAC 엔진. int8/int16 곱셈 누적 처리 |

세부 스펙을 공식 데이터시트에서 그대로 가져오면 다음과 같다.

- **ISA**: `rv32imf_zve32x_zicsr_zifencei_zbb` — 32-bit RISC-V에 정수 곱셈/나눗셈(M), 단정도 부동소수점(F), 임베디드 벡터 부분집합(Zve32x) 확장
- **파이프라인**: 4단계, 인오더 발급(in-order dispatch)·아웃오오더 리타이어(out-of-order retire)
- **발급 폭**: 스칼라 4-way, 벡터 2-way
- **메모리**: 명령용 8KB ITCM + 데이터용 32KB DTCM. 둘 다 캐시가 아닌 단일 사이클 지연 SRAM (틀리지 말 것 — 캐시가 아니다)
- **버스**: AXI4 인터페이스로 외부 메모리 접근 및 외부 CPU의 NPU 설정 모두 지원 (매니저/서브오디네이트 겸용)
- **데이터 타입**: BFloat16 네이티브 지원 — 트랜스포머 모델용

눈여겨볼 디테일은 두 가지다. 첫째, 캐시를 아예 없애고 TCM(tightly-coupled memory)을 쓴 것은 결정론적 실행 지연이 웨어러블 always-on 워크로드에서 캐시 히트율 관리보다 중요하기 때문이다. 둘째, 벡터 레지스터 폭(VLEN)이 파라미터라이즈 되어 있어 기본 128-bit에서 최대 1024-bit까지 확장 가능하며, 구글은 이 확장 경로가 "TinyGemma 270M 같은 LLM급 워크로드를 여는 길"이라고 공식 문서에 명시했다. 웨어러블용 저전력 코어 설계에 처음부터 온디바이스 LLM 실행을 염두에 둔 것이다.

### 도구줄까지 공개

하드웨어 IP는 검증 환경이 없으면 무용지물이다. 리포지토리에는 실제로 다음이 포함되어 있다:

```bash
# 요구사항: Bazel 8.6.0, Python 3.9–3.13
git clone https://github.com/google-coral/coralnpu.git
cd coralnpu

# 1. Cocotb 테스트 스위트 실행 (RTL 검증)
bazel run //tests/cocotb:core_mini_axi_sim_cocotb

# 2. 예제 바이너리 빌드
bazel build //examples:coralnpu_v2_hello_world_add_floats

# 3. Verilator 시뮬레이터 빌드 후 바이너리 실행
bazel build //tests/verilator_sim:core_mini_axi_sim
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
  --binary bazel-out/k8-fastbuild/bin/examples/coralnpu_v2_hello_world_add_floats.elf
```

즉, 실리콘 없이도 FPGA나 시뮬레이터에서 이 NPU에 C 코드를 올려 돌려볼 수 있다. 컴파일러 스택은 MLIR/IREE 기반이며 JAX·PyTorch·TFLite(LiteRT)를 지원한다. 행동 수준 시뮬레이터([MPACT-CoralNPU](https://developers.google.com/coral))는 별도 리포에서 제공된다.

## 왜 구글은 이걸 무료로 풀었나

NPU IP를 공개한다는 건 반도체 사업 모델 관점에서는 역행처럼 보인다. ARM이나 Ceva 같은 IP 사업자는 설계 라이선스로 수익을 올린다. 하지만 구글의 이득 구조는 다르다.

1. **모델 진입장벽 제거**: 구글은 Gemma 같은 오픈 모델을 갖고 있고, 그 모델이 돌아가는 칩이 많아질수록 구글 생태계가 넓어진다. 칩 설계도를 풀어서 '어떤 반도체 회사든 저전력 Gemma 실행 칩을 만들 수 있는 길'을 연 것이다.
2. **RISC-V 생태계 공략**: Coral NPU의 모든 커스텀 확장은 RISC-V 표준 확장 조합(rv32imf + Zve32x + Zbb) 위에 얹혀 있다. 독자 아키텍처가 아니라 표준을 따르므로, 기존 RISC-V 툴체인 투자가 그대로 재사용된다.
3. **소프트웨어 퍼스트**: 구글의 이익은 칩 판매가 아니라 그 위에서 도는 소프트웨어·모델·서비스다. 하드웨어 설계의 개방은 소프트웨어 지배력을 위한 투자다.

## 2026년, 생태계가 실제로 움직이고 있다

선언에 그치지 않고 1년 남짓 사이에 상업적 파이프라인이 구체화됐다.

**VeriSilicon(芯原) 상용화 — 2025년 11월 13일.** 구글과 공동 발표를 통해 VeriSilicon이 Coral NPU IP의 엔터프라이즈급 상용 버전을 제공하고 원스톱 커스텀 실리콘 서비스를 제공하기로 했다. VeriSilicon은 현재 Coral NPU 기반 검증 칩을 AI/AR 글래스와 스마트홈 애플리케이션 대상으로 개발 중이다.

**Synaptics Coralboard — 2026년 5월 19일 발표.** Synaptics Astra SL2619 SoC에 1 TOPS Coral NPU와 2GB DDR4를 탑재한 개발보드로, Google I/O 2026에서 Gemma 3 270M 기반 초소형 모델(TinyGemma 270M)을 클라우드 없이 완전히 온디바이스로 실행하는 데모를 선보였다. 구글 공식 문서에 따르면 2026년 7월부터 DigiKey를 통해 예약 판매가 시작됐다.

**MIPS S8200 — 2026년 1월 5일.** 같은 '오픈 RISC-V NPU' 흐름의 다른 축이다. GlobalFoundries 산하의 MIPS는 S8200 RISC-V NPU IP를 발표했고, Lockheed Martin의 반도체 자회사 ForwardEdge ASIC이 자율 플랫폼용 ASIC에 채택했다. MIPS는 2027년 S8200 탑재 실리콘 레퍼런스 플랫폼 샘플을 계획 중이다. MIPS의 접근은 Coral NPU가 겨냥하는 웨어러블(밀리와트급)과 달리 자율주행·로보틱스 엣지(멀티와트급)를 타깃으로 한다는 점에서 보완적이다.

정리하면 2026년 엣지 AI 실리콘 지형은 "데이터센터 GPU → 엣지 NPU → 오픈 NPU IP"로 수직 내려오는 중이고, 그 최하단에서 구글은 설계도 공개라는 비전통적 무기로 표준 선점을 하고 있다.

## 주의할 점 — 이것이 만능은 아니다

Coral NPU를 실제 도입을 검토하는 관점에서 몇 가지 냉정한 제약을 짚어둔다.

- **TOPS가 아니라 효율의 게임이다.** Coralboard의 1 TOPS는 데이터센터 GPU와 단위만 같은 숫자다. 이 칩의 설계 목표는 초저전력 always-on 추론이며, MobileNet급 분류·검출이나 270M 파라미터급 초소형 LM이 실용적 상한에 가깝다.
- **ITCM 8KB / DTCM 32KB는 의도된 제약이다.** 모델 가중치와 활성값은 AXI4로 외부 메모리에 두고, TCM에는 핫한 코드·데이터만 올리는 메모리 계획이 필수다. 32-bit 주소 공간 역시 대규모 모델 상주에는 제약이다.
- **상용 통합에는 여전히 전문성이 필요하다.** IP가 공개됐다고 해서 검증·DFT·물리 설계가 공짜가 되는 건 아니다. VeriSilicon이 '상용 준비된 엔터프라이즈급 버전'을 별도로 파는 이유가 그것이다.
- **컴파일러 성숙도.** MLIR/IREE 스택이 지원한다고는 하지만 임베디드 툴체인 특유의 최적화·디버깅(GDB 지원은 된다) 노력은 여전히 개발자 몫이다.

## 결론

- Coral NPU는 2025년 10월 15일 Apache 2.0으로 공개된 구글 리서치의 오픈소스 NPU IP로, RISC-V(rv32imf_zve32x) 기반에 scalar·vector·matrix 세 컴포넌트를 갖춘 C 프로그래밍 가능 가속기다.
- 캐시 대신 단일 사이클 TCM, BFloat16 네이티브 지원, VLEN 확장 경로(최대 1024-bit) 등 웨어러블 온디바이스 LLM을 겨냥한 설계 선택이 읽힌다.
- VeriSilicon(상용화·검증 칩), Synaptics(Coralboard, TinyGemma 270M 온디바이스 실행), MIPS S8200(자율 엣지)까지, 오픈 RISC-V NPU 생태계가 1년 만에 실제 파이프라인을 갖췄다.
- 구글의 계산은 명확하다 — 하드웨어 설계도를 풀어 Gemma가 도는 칩의 수를 늘리고, 그 위에서 소프트웨어 생태계의 지배력을 굳힌다.
- 당장 해볼 수 있는 것: [리포지토리](https://github.com/google-coral/coralnpu)를 클론해서 Verilator 시뮬레이터로 헬로월드 바이너리를 돌려보는 것부터. Bazel 8.6.0과 Python 3.9–3.13만 있으면 실리콘 없이 NPU 개발 경험을 시작할 수 있다.

## 참고 자료

- [Coral NPU GitHub 리포지토리](https://github.com/google-coral/coralnpu) — Apache 2.0, RTL/툴체인/테스트 전체 포함
- [Coral NPU 데이터시트 (Google 공식)](https://developers.google.com/coral/guides/hardware/datasheet)
- [Coral 최신 업데이트 (Google 공식)](https://developers.google.com/coral/news/announcements)
- [VeriSilicon–Google Coral NPU IP 공동 발표 (2025-11-13)](https://www.verisilicon.com/en/PressRelease/CoralNPU)
- [MIPS S8200 발표 (2026-01-05)](https://mips.com/press-releases/mips-s8200-delivers-software-first-risc-v-npu-to-enable-physical-ai-at-the-autonomous-edge/)
