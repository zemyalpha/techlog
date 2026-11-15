---
title: "1조 파라미터 모델을 무료로 다운로드하는 시대, 진짜 문제는 서빙이다 — Mistral Large 4 오픈 웨이트와 llm-d 1.0 분산 추론 가이드"
date: "2026-11-15"
keywords: ["Mistral Large 4", "llm-d", "오픈 웨이트", "분산 추론", "vLLM", "Kubernetes", "자체 호스팅"]
lang: "ko"
description: "2026년 10월, 1조 파라미터 오픈 웨이트 모델(Mistral Large 4)과 CNCF 분산 추론 스택(llm-d 1.0)이 열흘 간격으로 나왔다. 프런티어급 모델을 직접 호스팅하려는 팀을 위한 실전 가이드."
---

# 1조 파라미터 모델을 무료로 다운로드하는 시대, 진짜 문제는 서빙이다 — Mistral Large 4 오픈 웨이트와 llm-d 1.0 분산 추론 가이드

2026년 10월은 자체 호스팅(self-hosting) 인프라를 고민하는 팀에게 기념비적인 달이었다. **10월 20일**, CNCF 샌드박스 프로젝트 **llm-d**가 쿠버네티스 네이티브 분산 추론 스택의 첫 정식 버전(1.0)을 PyTorch Conference에서 발표했다. 그리고 열흘 뒤 **10월 29일**, Mistral AI가 약 1조 파라미터 규모의 플래그십 모델 **Mistral Large 4(코드명 Le Chonk)**의 웨이트를 허깅페이스에 공개했다.

이 두 사건을 함께 봐야 하는 이유는 명확하다. 오픈 웨이트 모델이 이제 프런티어급 규모에 도달했지만, 그런 모델은 단일 머신의 `vllm serve` 한 방으로는 안정적으로 서빙할 수 없다. **모델은 공짜로 받을 수 있지만, 서빙 아키텍처는 여전히 여러분의 몫**이다. 이 글에서는 무엇이 바뀌었는지, 그리고 실제로 어떻게 시작하는지 다룬다.

## 1. Mistral Large 4: 무엇이 공개되었나

허깅페이스 모델 카드(`mistralai/Mistral-Large-4-1T-A52B`)와 Mistral 공식 발표에 따르면 스펙은 다음과 같다.

- **총 파라미터 약 1.05조**, 토큰당 활성 파라미터 49B(임베딩·출력 레이어 포함 시 52B)의 MoE 구조 — 라우팅 전문가 128개 중 토큰마다 4개 활성 + 공유 전문가 1개
- **100만 토큰 컨텍스트**, 텍스트·이미지 입력 → 텍스트 출력의 네이티브 멀티모달
- 160개 이상 언어 지원(모든 EU 공식 언어 포함)
- Mistral 자체 유럽 데이터센터의 NVIDIA Grace Blackwell GPU 3,800대로 처음부터 학습
- 비용-성능 트레이드오프를 조절하는 **3단계 추론 모드**(low/medium/high, 기본값 high)

성능은 '공개 웨이트 중 코딩 최강'을 내세운다. 아래는 Mistral 발표 및 Artificial Analysis 측정치다.

| 벤치마크 | 점수 | 비고 |
|---|---|---|
| FrontierCode 2.0 (Cognition) | 19.7% | 공개 웨이트 1위 (GLM-5.3 17.8%, Kimi K3 17.3%) |
| AA Coding Agent Index | 48% | GLM-5.3와 동률, 평균 47% 적은 토큰 사용 |
| Terminal-Bench 4.0 | 41% | 프리뷰 대비 +13%p, 출력 토큰 30% 감소 |
| AA Cyber Index | 51% | 방어적 보안 태스크 |

독립 평가 기관 [Artificial Analysis의 지표(VentureBeat 보도 기준)](https://venturebeat.com/technology/mistral-large-4-le-chonk-arrives-as-the-top-open-model-outside-china-with-improved-coding-performance-and-63-lower-cost-per-task)도 이를 뒷받침한다. 다만 전체 지능 지수(Intelligence Index v5.0.1)에서는 26.1점으로 **미국·유럽 출신 공개 모델 중 최고일 뿐**, GLM-5.3(31.7)이나 Kimi K3.1 Pro(46.5)에는 뒤진다. "공개 웨이트 전체 1위"가 아니라 "서방권 공개 모델 1위 + 코딩 특화"라는 정확한 이해가 필요하다. 같은 지표에서 태스크당 비용은 프리뷰 시절 $5.54에서 $2.06으로 63% 내려갔다.

### 라이선스의 함정 — "사실상 MIT"라고 읽으면 안 된다

배포 라이선스는 **Mistral Ecosystem Open-Weight License**로, MIT에 가깝지만 조건이 하나 붙어 있다.

> 모델(및 파생 모델)을 **Model-as-a-Service(MaaS)로 판매하는 사업자**는 Mistral과 별도 상업 라이선스를 체결해야 한다.

즉 내부 용도, 애플리케이션에 내장해 판매하는 용도, 파인튜닝 후 자사 제품에 쓰는 용도는 무료다. 하지만 "우리 API로 ML4를 재판매합니다"라는 비즈니스라면 이야기가 다르다. revenue 기준 같은 문턱은 없고 MaaS 여부로만 판정된다는 점이 특징이다. 검토 시 원문 라이선스를 직접 읽어야 한다.

### API 가격과 직접 호스팅

API로 쓰면 백만 토큰당 입력 $0.70 / 캐시 입력 $0.07 / 출력 $2.10이다. Mistral Studio, OpenRouter, Microsoft Foundry 등에서 이용 가능하다.

직접 돌리려면 현실적인 숫자를 봐야 한다. Blackwell GPU용 **NVFP4 양자화 버전도 약 610GB로 보고된다**. 소비자용 GPU 몇 장으로는 불가능하고, 데이터센터급 NVIDIA 하드웨어 + vLLM 또는 SGLang이 기본 요건이다. "공짜 모델"과 "공짜 인프라"는 다른 문제라는 뜻이다.

## 2. 왜 단일 서버 서빙이 무너지는가 — 에이전트 시대의 추론 트래픽

llm-d를 이해하려면 추론 트래픽의 변화를 먼저 봐야 한다. [Google Cloud 블로그](https://cloud.google.com/blog/topics/ai-infrastructure/llm-d-1-0-an-open-source-stack-for-distributed-inference)의 진단은 이렇다. 1년 전 추론 트래픽의 대부분은 "프롬프트 하나, 응답 하나"인 채팅이었다. 지금은 **다중 단계 에이전트와 RL 롤아웃**이 도구를 향해 부채꼴로 퍼지고(fan-out), 긴 프리픽스를 공유하고, 버스트로 몰려온다.

이 트래픽을 라운드 로빈 로드밸런서 뒤에 두면 세 가지가 일어난다.

1. **같은 프롬프트를 레플리카마다 재계산**한다 — 프리픽스 캐시 히트가 레플리카마다 따로 놀게 된다
2. **가속기 메모리가 중복 상태로 채워진다** — KV 캐시가 레플리카별로 복제된다
3. **버스티한 에이전트 루프 하나가 대화형 사용자를 밀어낸다**

즉 문제는 모델 크기만이 아니라 **트래픽 패턴**이다. llm-d가 푸는 것이 바로 이 지점이다.

## 3. llm-d 1.0: 쿠버네티스 위의 조율된 서빙 풀

[llm-d](https://github.com/llm-d/llm-d)는 2025년 Red Hat, Google Cloud, IBM Research, CoreWeave, NVIDIA가 공동 창립하고 2026년 3월 CNCF 샌드박스로 들어간 프로젝트로, 1.0 기준 25개 이상 조직의 기여자 500명 이상이 참여했다. vLLM과 SGLang 위에서 동작하며 NVIDIA·AMD·Intel GPU와 Google TPU를 하나의 엔드포인트 뒤에 묶는다.

핵심 컴포넌트를 구조로 이해하면 이렇다.

```
                    ┌─────────────────────────────┐
 에이전트/앱 ──────▶ │  llm-d 라우터 (Gateway API    │
                    │  Inference 기반)             │
                    │  · 프리픽스 캐시 인지 라우팅   │
                    │  · 로드/지연 예측 기반 선택    │
                    │  · flow control (우선순위)   │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
        prefill 풀        decode 풀         비동기 프로세서
        (연산 집중)       (메모리 집약)     (Pub/Sub, Redis)
              │                │
              └───── KV 캐시 계층화 ─────┘
              HBM → 호스트 RAM → SSD/Lustre
              (OffloadingConnector / Mooncake / HiCache)
```

1.0에서 새로 안정화된 능력은 다음 네 가지다.

- **멀티테넌트 QoS** — 우선순위·공정성 기반 flow control로 지연 민감 요청과 버스티한 에이전트 워크로드가 한 플릿을 SLO 안에서 공유한다
- **무중단 운영** — 라우터 HA(hot standby)와 `DisaggregatedSet`(LeaderWorkerSet 기반)가 prefill/decode 풀을 하나의 논리 단위로 관리하며 롤링 업데이트 중 컨텍스트를 잃지 않는다
- **안정적인 v1 설정 API** — 여러 클라우드와 가속기에서 검증된 요청 경로
- **에이전트 인지 서빙** — 세션 인지 라우팅, 턴 우선순위 스케줄링, Responses API 상태 지원으로 멀티턴 컨텍스트를 재사용한다

### 숫자로 보는 효과 (모두 공식 발표 기준)

| 워크로드 | 효과 |
|---|---|
| Ironwood TPU 에이전트 코딩 | 중간 TTFT 85%+ 감소, 총 처리량 2.4배 (일반 K8s Service 대비) |
| Vertex AI(Gemini 플랫폼) 프로덕션 | 프리픽스 캐시 히트율 35%→70%, 코딩 에이전트 TTFT 35%+ 감소 |
| 17,000노드 멀티리전 (MoE) | 3클러스터 간 처리량 2.9배, 성공률 99.9%, 라우팅 오버헤드 1% 미만 |
| A3 Mega 70B, 5만 토큰 프롬프트 | TTFT 약 40% 개선, 종단 지연 30% 감소, GPU 시간 약 60% 절감 |
| RL 학습 중 time-slicing | 가속기 듀티 사이클 약 40%→70%+ |

Snap은 프리픽스 인지 라우팅으로 캐시 히트율 75~80%를, ZenteiQ는 TPU v6e 1,000개당 일일 토큰 생성량을 14.7억→59.7억(4배)을 달성했다고 각각 밝혔다.

## 4. 실전: 10분 안에 시작하는 llm-d

llm-d는 Helm 차트로 배포되며, 공식 quickstart 저장소가 전 과정을 스크립트로 제공한다. GPU가 달린 쿠버네티스 클러스터가 있다면 다음이 실제 시작점이다.

```bash
# 저장소 클론 (공식 quickstart)
git clone https://github.com/llm-d/llm-d-deployer.git
cd llm-d-deployer/quickstart

# 필요 도구 설치 (yq, jq, helm, kustomize, kubectl)
./install-deps.sh

# 배포 — 샘플 모델 기반, 네임스페이스 지정
export HF_TOKEN=hf_xxx   # 모델 다운로드 권한이 있는 토큰
./llmd-installer.sh --namespace llm-d
```

주요 플래그도 알아두면 좋다.

| 플래그 | 역할 |
|---|---|
| `--download-model` | 허깅페이스에서 모델을 PVC로 다운로드 |
| `--storage-size 15Gi` | 저장 볼륨 크기 지정 |
| `--storage-class <name>` | 스토리지 클래스 지정 |
| `--skip-infra` | 인프라 컴포넌트 설치 생략 |
| `--uninstall` | 설치 제거 |

기본 quickstart는 소형 모델(Llama-3.2-3B)로 파이프라인을 검증하는 용도다. Mistral Large 4급을 올리려면 prefill/decode 분산과 전문가 병렬(expert parallelism)이 필요하므로, 공식 문서의 "well-lit paths" 가이드와 `llm-d-benchmark`의 표준 레시피를 따르는 것이 좋다. 특히 `llm-d-benchmark`는 로컬 Kind 클러스터에서 GPU 없이도 배포 파이프라인을 리허설할 수 있어, 프로덕션 투입 전 검증 도구로 쓸 만하다.

## 5. 주의사항 — 도입 전에 체크할 것

- **라이선스 조건 확인**: 앞서 설명한 MaaS 조항은 Mistral Large 4에만 있는 게 아니라 오픈 웨이트 라이선스의 일반적 트렌드다. GLM-5.3도 유사한 맞춤 라이선스(매출 100억 달러 초과 기업의 MaaS는 별도 보안 검토)를 적용한다. "오픈 웨이트 = 아무 조건 없음"이라는 가정은 버려야 한다.
- **하드웨어 현실 인정**: 610GB(NVFP4) 체크포인트는 개인이나 소규모 팀의 영역이 아니다. 이 규모는 클라우드 GPU 플릿이나 학교/기업 공유 클러스터 전제다.
- **API 프리뷰 상태**: 2026년 11월 기준 Mistral API의 ML4 모델 카드는 아직 Public Preview(v26.10)로 표시된다. GA 일정은 발표되지 않았다.
- **벤치마크 맥락 읽기**: Mistral 발표의 "최고" 주장은 대부분 코딩·보안 벤치마크 한정이다. 범용 지능에서는 중국 계열 공개 모델이 앞서므로, 용도에 맞는 지표를 볼 것.

## 결론

- Mistral Large 4로 **프런티어급 1조 파라미터 모델이 무료 웨이트로 풀렸지만**, 라이선스에 MaaS 조항이 있고 하드웨어 요구량은 데이터센터급이다
- 에이전트·RL 트래픽 시대에는 **프리픽스 인지 라우팅, 계층화 KV 캐시, prefill/decode 분리**가 선택이 아니라 기본이 됐다
- llm-d 1.0은 이것을 쿠버네티스 표준(Gateway API Inference, LeaderWorkerSet) 위에 올린 **CNCF 프로젝트**로, vLLM/SGLang을 이미 쓰고 있다면 부품만 조립해 도입할 수 있다
- 공개된 숫자들은 대부분 공급자 발표 기준이므로, 도입 전 `llm-d-benchmark`로 자기 워크로드에서 재측정하는 것이 원칙이다
- 당장의 첫 단계: `git clone llm-d-deployer` → 로컬/소형 클러스터에서 quickstart 실행 → 자기 트래픽 패턴(멀티턴 비율, 프리픽스 공유율)부터 측정해보라

**참고 자료**
- [Mistral Large 4 오픈 웨이트 공식 발표](https://mistral.ai/news/mistral-large-4-oss/) / [허깅페이스 모델 카드](https://huggingface.co/mistralai/Mistral-Large-4-1T-A52B)
- [llm-d v1.0 발표문](https://llm-d.ai/blog/llm-d-v1.0-state-of-the-art-inference-ready-for-day-2) / [llm-d GitHub](https://github.com/llm-d/llm-d) / [llm-d quickstart](https://github.com/llm-d/llm-d-deployer/blob/main/quickstart/README.md)
- [Google Cloud: llm-d 1.0 on TPUs and GPUs](https://cloud.google.com/blog/topics/ai-infrastructure/llm-d-1-0-an-open-source-stack-for-distributed-inference) / [IBM Research 발표](https://research.ibm.com/blog/llmd-1-0-is-ready-for-production-inference-serving) / [Red Hat 보도자료](https://www.redhat.com/en/about/press-releases/llm-d-1-advances-open-cost-efficient-ai-inference-production-scale)
- [VentureBeat: Mistral Large 4 분석 (Artificial Analysis 수치)](https://venturebeat.com/technology/mistral-large-4-le-chonk-arrives-as-the-top-open-model-outside-china-with-improved-coding-performance-and-63-lower-cost-per-task)
