---
title: "에이전트 트래픽은 라운드로빈 로드밸런서로 못 버틴다 — llm-d 1.0 프리픽스 캐시 라우팅 실전 가이드"
date: "2026-10-23"
keywords: ["llm-d", "Kubernetes", "분산 추론", "prefix cache", "에이전트 인프라", "vLLM", "KV cache"]
lang: "ko"
description: "CNCF 프로젝트 llm-d 1.0이 에이전트 워크로드용 추론 인프라를 어떻게 바꾸는지, 프리픽스 캐시 라우팅·P/D 분리·플로우 컨트롤의 원리와 실제 배포 명령어까지 정리했다."
---

# 에이전트 트래픽은 라운드로빈 로드밸런서로 못 버틴다 — llm-d 1.0 프리픽스 캐시 라우팅 실전 가이드

2026년 10월 20일, PyTorch Conference North America에서 오픈소스 분산 추론 스택 **llm-d 1.0**이 발표됐다. Red Hat·IBM·Google Cloud·CoreWeave·NVIDIA가 2025년 공동 창립하고 현재 CNCF 샌드박스 프로젝트로 운영되는 이 프로젝트는, 쿠버네티스 위에서 vLLM/SGLang 모델 서버들을 하나의 조율된 서빙 풀로 묶어 주는 역할을 한다.

흥미로운 지점은 성능 숫자 자체보다 **문제 정의**다. Google Cloud 발표 블로그는 이렇게 말한다. 1년 전만 해도 추론 트래픽 대부분은 챗봇이었다 — 프롬프트 하나 들어와서 응답 하나 나가는. 그런데 지금은 다중 단계 에이전트와 강화학습 롤아웃이 도구를 가로질러 확장되고(fan-out), 긴 프리픽스를 공유하고, 버스트로 몰려온다. 이 트래픽을 라운드로빈 로드밸런서 뒤에서 서빙하면 어떻게 되는지, 그리고 llm-d가 이걸 어떻게 푸는지 실제 배포 명령어와 함께 정리한다.

## 에이전트 워크로드가 추론 인프라를 부러뜨리는 세 가지

llm-d 공식 가이드(optimized-baseline)는 전통적 HTTP 요청과 LLM 요청의 차이를 명확히 짚는다. 전통적 요청은 **빠르고, 균일하고, 저렴**해서 라운드로빈 스케줄링이 잘 작동한다. LLM 요청은 이 세 가정을 전부 깬다.

1. **멀티턴** — 대화와 에이전트 도구 루프는 같은 프리픽스가 점점 길어진 채로 반복해서 들어온다. 매 턴마다 시스템 프롬프트, 도구 정의, 지금까지의 대화 이력을 다시 보낸다.
2. **느림** — 단일 요청이 토큰을 생성하는 데 1분 이상 걸릴 수 있다.
3. **비균일** — 1,000토큰대 추론부터 100k+ 컨텍스트 입력까지 편차가 크다.

여기에 라운드로빈을 걸면 참사가 벌어진다. 같은 대화의 다음 턴이 **캐시를 갖고 있지 않은 다른 레플리카**로 라우팅되고, 그 레플리카는 긴 프리픽스를 처음부터 다시 프리필(prefill)한다. IBM 발표의 표현을 빌리면, 컨텍스트 관리가 최종 출력 생성보다 더 많은 자원을 소모하는 시대가 됐다는 것. 에이전트 하나가 과업을 수행하며 같은 코드·문서·지시문·대화 이력을 몇 번이고 다시 읽는다면, 이미 처리한 작업에 반복해서 비용을 지불하는 셈이다.

## llm-d의 해법: 캐시를 알고 있는 라우터

llm-d는 모델 서버(vLLM, SGLang)를 대체하는 게 아니라, 그 **위에** 있는 오케스트레이션 계층이다. 핵심 기술은 다섯 가지다.

- **프리픽스 캐시 인지 라우팅(prefix-cache-aware routing)** — 요청을 이미 해당 컨텍스트의 캐시를 보유한 레플리카로 보낸다. "끈적끈적한(sticky)" 엔드포인트 우선 + 포화 시 부하 분산 오버라이드.
- **KV 캐시 오프로딩** — GPU 메모리가 부족하면 KV 캐시를 CPU·디스크 계층으로 내려 멀티턴 요청의 유효 워킹셋을 키운다.
- **프리필/디코드 분리(P/D disaggregation)** — 프롬프트를 읽는 작업과 응답을 생성하는 작업을 전용 인스턴스로 나눈다.
- **와이드 익스퍼트 병렬성(wide EP)** — 대형 MoE 모델을 여러 노드의 가속기에 분산한다.
- **배치/비동기 인퍼런스** — OpenAI 호환 Batch API로 오프라인 대량 추론을 효율화한다.

그리고 1.0에서 새로 추가된 것이 "Day 2 운영" 기능들이다. **멀티테넌트 QoS**(우선순위·공정성 기반 플로우 컨트롤로 버스티한 에이전트 트래픽이 지연 민감 요청을 굶기지 않게 함), **제로 다운타임 운영**(라우터 HA 핫 스탠바이, 롤링 업데이트), 프리필/디코드 레플리카를 하나의 논리 단위로 묶는 **DisaggregatedSet**, 안정화된 **v1 설정 API**, 그리고 **에이전트 인지 서빙**(세션 인지 라우팅, 턴 우선순위 스케줄링, Responses API 상태 네이티브 지원)이다.

## 실전 배포: Well-Lit Path 가이드

llm-d는 문서화·테스트·벤치마크된 레시피를 "well-lit path"라는 이름으로 제공한다. 기본 시작점은 optimized-baseline — 프리픽스 캐시 + 부하 인지 라우팅이 켜진 vLLM 배포다. 공식 가이드의 실제 흐름을 옮기면:

```bash
git clone https://github.com/llm-d/llm-d.git && cd llm-d
export REPO_ROOT=$(realpath $(git rev-parse --show-toplevel))
export GUIDE_NAME=optimized-baseline
export NAMESPACE=llm-d-optimized-baseline
export ACCELERATOR_TYPE=gpu   # gpu, amd, xpu, tpu/v5p, tpu/v6, tpu/v7, npu, cpu 등
export MODEL_SERVER=vllm      # vllm, sglang, trtllm, atom
export INFRA_PROVIDER=base    # base, gke, amd-ci
export MODEL=Qwen/Qwen3-32B   # 가속기별 검증된 모델 지정
source ${REPO_ROOT}/guides/env.sh
```

라우터는 쿠버네티스 게이트웨이 API의 인퍼런스 확장(GAIE) 위에 올라간다:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/${GAIE_URL}/v1-manifests.yaml
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic llm-d-hf-token \
  --from-literal="HF_TOKEN=${HF_TOKEN}" \
  --namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -

helm install ${GUIDE_NAME} \
  ${ROUTER_STANDALONE_CHART} \
  -f ${ROUTER_BASE_VALUES} \
  -f ${ROUTER_VALUES} \
  ${ACCELERATOR_VALUES:-} \
  -n ${NAMESPACE} --version ${ROUTER_CHART_VERSION}

# 모델 서버 배포 (kustomize)
kubectl apply -n ${NAMESPACE} \
  -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/${ACCELERATOR_TYPE}/${MODEL_SERVER}/${INFRA_PROVIDER}/
```

배포 후에는 프리픽스 캐시 효과를 직접 확인할 수 있다. 약 2k 토큰 프리픽스를 공유하는 요청 10개를 보내고, 각 파드의 메트릭을 읽는 것까지 가이드에 포함되어 있다:

```bash
for pod in $(kubectl get pods -n ${NAMESPACE} -l llm-d.ai/guide=${GUIDE_NAME} \
  -o jsonpath='{.items[*].metadata.name}'); do
  echo "== ${pod}"
  kubectl get --raw "/api/v1/namespaces/${NAMESPACE}/pods/${pod}:8000/proxy/metrics" \
    | grep -E '^(vllm:prefix_cache_(hits|queries)_total|vllm:request_success_total)' || true
done
```

`vllm:prefix_cache_hits_total`이 라우팅 덕에 실제로 올라가는지 눈으로 확인하는 것이, 이 스택을 도입할 가치가 있는지 판단하는 가장 빠른 방법이다.

## 숫자로 보는 효과 — 그리고 어디까지 믿을 것인가

IBM·Red Hat 발표자료와 프로젝트 README가 제시하는 수치들이다. 전부 **특정 시연/벤치마크 조건에서의 결과**라는 점을 전제로 읽어야 한다.

- IBM과 Red Hat은 오픈 웨이트 모델 **GLM 5.2(약 7,530억 파라미터)**를 기존 NVIDIA GPU 플릿에 llm-d로 배포해, 수백 개 에이전트 세션이 병렬로 도는 워크로드에서 **분당 660만 출력 토큰 이상**, 최대 **3,000개 동시 코딩 에이전트**를 선점(preemption) 없이 서빙했다. 상용 API 대비 **토큰당 5~10배 낮은 비용**이라고 주장한다.
- IBM Research의 에이전트 코딩 워크로드 테스트에서 프리픽스 인지 라우터는 **이용 가능한 캐시 재사용의 97%**를 잡아냈다.
- README의 파트너 벤치마크: 라운드로빈 대비 **3배 출력 처리량·2배 빠른 TTFT**(Llama 3.1 70B, 4× AMD MI300X, Tesla/Red Hat), Google의 예측 지연 스케줄링으로 **TTFT·ITL 40% 감소**, AWS의 P/D 분리로 GPT-OSS on B200에서 **최대 70% 높은 토큰/초**, 계층적 KV 오프로딩으로 250 동시 사용자에서 **13.9배 처리량**(4× H100).
- Google은 PyTorch Conference 키노트에서 GKE 위 Agent Substrate의 **샌드박스 에이전트 1,000개**가 vLLM·SGLang·TPU/GPU 혼합 플릿에 걸린 단일 llm-d 엔드포인트를 치는 시연을 보였다.

내 환경에서 재현하려면 문서의 캘리브레이션 레시피로 자체 하드웨어의 `peakPrefillThroughput`을 측정해 필터에 설정하라는 것이 공식 가이드의 권고다. 남의 벤치마크 숫자를 그대로 기대하면 안 된다는 뜻이다.

## 주의사항

- **CNCF 샌드박스 단계다.** 500명 이상의 기여자와 25개 이상 조직이 참여하고 있지만 아직 인큐베이팅 단계이므로, 조직 표준으로 삼기 전에 자체 검증이 필요하다.
- **1.0의 안정성 약속은 v1 설정 API 기준이다.** 컴포넌트별 릴리스 노트를 확인하며 올라가야 한다.
- **라우터가 주는 이득은 트래픽 프로파일에 따라 달라진다.** 프리픽스 공유가 적은 1회성 단발 트래픽이 대부분이라면 캐시 인지 라우팅의 이득이 크지 않다. 반대로 에이전트·멀티턴·긴 컨텍스트가 많을수록 이득이 커진다.

## 결론

- 에이전트 시대의 추론 비용 문제는 모델 성능이 아니라 **같은 컨텍스트를 반복해서 다시 계산하는** 인프라 낭비에서 온다.
- llm-d 1.0은 프리픽스 캐시 인지 라우팅, KV 캐시 오프로딩, P/D 분리, 멀티테넌트 플로우 컨트롤을 하나의 쿠버네티스 네이티브 스택으로 통합했다.
- well-lit path 가이드를 따라 optimized-baseline을 띄우고 `vllm:prefix_cache_hits_total` 메트릭으로 자기 트래픽에서 캐시 히트가 실제로 오르는지 확인하는 것이 첫 단계다.
- 벤치마크 수치(3배 처리량, 5~10배 비용 절감 등)는 어디까지나 발표 측 시연 조건의 결과며, 도입 전 자체 캘리브레이션이 필수다.

시작해보고 싶다면 [llm-d 저장소](https://github.com/llm-d/llm-d)의 quickstart와 [optimized-baseline 가이드](https://github.com/llm-d/llm-d/tree/main/guides/optimized-baseline)부터 보면 된다.

**참고 자료**
- IBM Newsroom, "llm-d 1.0 Advances Open, Cost-Efficient AI Inference for Production Scale" (2026-10-20)
- IBM Research Blog, "llm-d 1.0 hardens open-source AI inference for day 2 production operations" (2026-10-20)
- Google Cloud Blog, "llm-d 1.0, an open source stack for distributed inference" (2026-10-21)
- llm-d GitHub 저장소 README 및 guides/optimized-baseline 문서 (2026-10-23 확인)
