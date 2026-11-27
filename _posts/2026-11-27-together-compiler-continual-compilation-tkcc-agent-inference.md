---
title: "에이전트가 재시작 없이 추론 엔진을 다시 쓴다 — Together Compiler '지속 컴파일' 설계 해부"
date: "2026-11-27"
keywords: ["Together Compiler", "TKCC", "지속 컴파일", "continual compilation", "LLM 추론 최적화", "persistent engine", "InferenceX", "AI 에이전트"]
lang: "ko"
description: "Together AI가 2026년 11월 공개한 Together Compiler(TKCC)의 2계층 영속 엔진, 467초→9초 핫 리로드, 252배 실험 처리량의 원리와 self-hosting 운영자가 가져갈 설계 교훈을 분석한다."
---

# 에이전트가 재시작 없이 추론 엔진을 다시 쓴다 — Together Compiler '지속 컴파일' 설계 해부

2026년 11월 4일, Together AI는 "지난해엔 메가커널을 만들었고, 올해는 그걸 지웠다"는 문장으로 시작하는 발표를 내놨다. [Together Compiler(TKCC)](https://www.together.ai/blog/together-compiler-continual-compilation-for-ai-inference)다. 6일 뒤 같은 팀이 논문을 [arXiv:2611.14681](https://arxiv.org/abs/2611.14681)로 올렸고, 이 시스템은 이미 하루 수조 토큰 단위의 프로덕션 트래픽을 서빙 중이라고 밝혔다.

이 발표가 흥미로운 이유는 숫자 자체보다 방향 전환이다. 지금까지 LLM 추론 최적화의 정석은 "잘 쓴 커널 하나(메가커널)" 또는 "잘 튜닝한 엔진 설정 하나"였다. TKCC는 그 정적 최적화를 포기하고, **트래픽이 변할 때마다 에이전트가 엔진을 다시 컴파일하는 루프**를 상시로 돌린다. 컴파일러가 산출물이 아니라 컴파일 과정 자체가 24시간 돌아가는 시스템인 것이다.

이 글은 TKCC가 풀려고 한 문제, 3가지 핵심 설계(2계층 영속 엔진·멀티 에이전트 메시지 보드·비용 모델), 그리고 검증된 성능 수치를 정리하고, vLLM/SGLang으로 self-hosting하는 운영자가 내일부터 적용할 수 있는 교훈으로 정리한다.

## 1. 문제: 왜 '한 번 최적화'가 더 이상 성립하지 않는가

TKCC 논문의 출발점은 Together AI가 실제 트래픽에서 관측한 세 가지 변동성이다.

- **아키텍처 변동성**: Transformer 등장 약 11년이 지났지만 OpenRouter 상위 10개 오픈소스 LLM 중 **같은 아키텍처를 쓰는 모델은 하나도 없다**고 한다. MoE 구성, 어텐션 희소성, 하이브리드 레이어가 제각각이므로 하나의 만능 커널이 존재할 수 없다.
- **워크로드 변동성**: 국가별로 프롬프트 길이 분포가 다르다(한 국가의 근무시간 트래픽이 97k 토큰, 다른 국가는 70k로 약 1.4배 차이). 신모델 출시 2주 뒤엔 사용자가 챗 평가에서 에이전트 워크로드로 넘어가며 **프롬프트 길이가 80% 증가**한다. 캐시 적중률도 버스트성 트래픽에 뒤흔들린다.
- **하드웨어 변동성**: H100, B200, B300, NVL72 등 플랫폼마다 최적 전략이 다르다.

여기에 기존 접근의 구조적 딜레마가 겹친다. 논문은 이를 **측정 처리량(measurement throughput) vs 측정 충실도(measurement fidelity)의 트레이드오프**로 정리한다.

1. **커널 단위 에이전트 최적화**: forward pass의 개별 커널을 떼어내 오프라인 벤치마크로 최적화한다. 반복은 빠르지만 MoE 라우팅이나 희소 어텐션처럼 **입력 데이터 분포에 따라 성능이 달라지는 프리미티브**에서는 타이밍이 왜곡된다. 실제 트래픽에선 다른 커널이 이긴다.
2. **엔진 단위 에이전트 최적화**: vLLM·SGLang·TRT-LLM을 통째로 띄우고 end-to-end로 측정한다. 충실도는 높지만, 실험마다 엔진을 재시작해야 한다. 논문의 표 1에 따르면 상용 엔진 하나가 서빙 준비까지 걸리는 시간은 **467초**다. 프로세스 초기화 93초, 가중치 로딩 50초, KV 캐시 사이징 21초, JIT 워밍업 93초, 오토튜닝 193초가 쌓인 결과다. 하루에 돌릴 수 있는 실험 수 자체가 바닥난다.

즉, "커널만 테스트하면 거짓이 나오고, 엔진을 통째로 테스트하면 느리다"는 딜레마. TKCC는 이 두 축을 동시에 푸는 것이 본체다.

## 2. 핵심 설계 ①: 2계층 영속 엔진 (Tiered Persistent Engine)

TKCC의 첫 번째 무기는 **영속 엔진(persistent engine)**이다. 엔진을 한 번만 띄우고, 에이전트가 제안하는 최적화 후보 forward pass를 **재시작 없이 핫 리로드**한다. 재컴파일이 필요한 지점(예: CUDA 그래프 캡처)만 7초 남기고 나머지 부팅 비용을 전부 제거해, 서빙 준비 시간이 표 1 기준 **467초에서 9.0초**로 압축된다.

여기에 충실도 문제를 풀기 위해 엔진을 두 계층으로 나눈다.

- **빠른 엔진(fast engine)**: 레이어 몇 개만 로드. 반복이 빠르고 타이밍 분산이 낮아 아이디어 빠른 스케닝용.
- **느린 엔진(slower engine)**: 레이어를 많이 로드. 실제 데이터 분포에서 최적화가 살아남는지 검증용.

에이전트는 빠른 엔진에서 후보를 대량 거른 뒤, 살아남은 후보만 느린 엔진에서 실데이터 검증을 받는다. CI의 빠른 테스트와 풀 통합 테스트를 나누는 것과 같은 논리를 추론 컴파일에 적용한 셈이다.

이 구조가 실험 처리량에 주는 영향이 크다. Together AI 발표 기준 TKCC는 기존 에이전트 기반 최적화 시스템 대비 **252배 높은 실험 처리량**을 낸다. 에이전트의 문제 해결 능력이 아니라 "에이전트가 실험을 얼마나 빨리 돌릴 수 있는가"가 병목이었다는 진단의 결과물이다.

## 3. 핵심 설계 ②: 메시지 보드 기반 멀티 에이전트 + ③: 병렬 전략 비용 모델

영속 엔진이 계속 새 실험을 먹게 하려면 후보 생성 속도도 따라와야 한다. TKCC는 **여러 에이전트를 상시 스폰하고 메시지 보드로 통신**하게 해 아이디어 생성을 병렬화한다. 각 에이전트는 서로의 진행 상황을 보드에서 읽어 중복을 피하고, 언덕오르기(hill-climbing)를 분산 수행한다. 논문은 에이전트 컨텍스트 관리에 대한 소거 실험(ablation)도 포함해, "무엇을 에이전트 맥락에 넣고 언제 지우는가"조차 실험 대상이었음을 보여준다.

세 번째 요소는 **병렬 구성 선택 비용 모델**이다. TP(텐서 병렬), EP(전문가 병렬), DCP(디코드 컨텍스트 병렬) 등의 조합은 하드웨어 스펙만으로 고르면 함정에 빠진다. MoE 전문가 불균형이나 프롬프트 길이 편차 같은 데이터 의존적 특성이 최적 배치를 바꾸기 때문이다. TKCC는 하드웨어 스펙 → 마이크로벤치마크 → 실측 forward pass의 3단 계층 측정으로 탐색 공간을 빠르게 가지치기한다. 발표의 그림 4는 "하드웨어 스펙이 고른 배치를 실측이 이기고, 정답 배치가 동시성 수준에 따라 바뀐다"는 결과를 보여준다. 즉 **정답이 고정값이 아니라 트래픽의 함수**라는 것을 시스템적으로 인정한 설계다.

전체 루프를 개념적으로 표현하면 다음과 같다(논문 구조를 설명하기 위한 의사코드이지 실제 구현이 아니다):

```python
# 개념 코드: TKCC의 지속 컴파일 루프 (의사코드)
engine = TieredPersistentEngine(model, hardware)  # 1회 기동, 이후 재시작 없음

while True:  # 트래픽이 흐르는 동안 계속
    shape = traffic_monitor.snapshot()            # 배치 크기, 시퀀스 길이, MoE 불균형, 캐시 적중률
    if drift_detector(shape) > threshold:         # 트래픽 드리프트 감지
        seeds = cost_model.prune(shape, hardware) # TP/EP/DCP 조합을 3단 계층 측정으로 가지치기
        board.publish_task(seeds)                 # 에이전트들에게 아이디어 생성 과제 배포
        for candidate in board.collect_proposals():
            timing = engine.fast_reload(candidate, layers="few")      # 빠른 엔진: 1차 스케닝
            if timing.passes(tolerance):
                timing2 = engine.slow_reload(candidate, layers="all") # 느린 엔진: 실데이터 검증
                board.report(candidate, timing2)
        best = board.select_best()
        if best.beats(current):
            engine.promote(best)                  # 핫 스왑: 다운타임 없이 교체
```

## 4. 결과: 24시간 만의 2.6배 — 그리고 검증 지점들

논문과 블로그가 보고하는 성능은 다음과 같다. (수치는 모두 Together AI 자체 발표 기준이며, [arXiv:2611.14681](https://arxiv.org/abs/2611.14681)과 [공식 블로그](https://www.together.ai/blog/together-compiler-continual-compilation-for-ai-inference)에서 교차 확인했다.)

| 항목 | 결과 |
|---|---|
| 최적화 소요 시간 | 24시간 미만 |
| 프론티어 오픈 모델 TPS | 최대 2.6배 향상 (Kimi K3, GLM 5.3, MiniMax M3.1-Flash, DeepSeek v4.1-Flash) |
| InferenceX 저지연(bs=1) | 1111 TPS (M3.1-Flash), 1205 TPS (DeepSeek v4.1-Flash) — 발표 기준 state-of-the-art |
| 메가커널 대비 | Llama 1B, batch size 1에서 1.7배 |
| 실험 처리량 | 기존 에이전트 최적화 대비 252배 |
| 엔진 서빙 준비 시간 | 467초 → 9.0초 |

주목할 디테일이 두 가지 있다. 첫째, 메가커널 대비 승부처로 고른 곳이 "Llama 1B batch size 1"이라는 점이다. 인간 전문가와 전통적 ML 컴파일러가 가장 잘 최적화한 정전영역(canonical workload)에서 이겼다는 상징적 의미가 크다. 둘째, 1111/1205 TPS 결과는 InferenceX 기본 구성이 아니라 **자체 비용 모델이 고른 병렬 구성**을 썼다고 명시한다. 즉 성능의 일부는 새 커널이 아니라 구성 선택에서 나온다는 것 — 이것이 "컴파일"의 실체다.

또한 TKCC의 원리는 LLM 밖으로도 이월된다. 날씨 모델링, PDE, 시계열, 바이오 분야의 오픈소스 코드베이스에서, 단순히 Claude Code의 `/goal` 명령으로 최적화한 베이스라인 대비 지오메트릭 평균 가속을 냈다고 한다(네 모델 중 세 모델에서 더 큰 가속). "에이전트로 코드를 최적화한다"는 일반 작업에서도 영속 엔진 + 계층 측정 구조가 유효하다는 방증이다.

## 5. Self-hosting 운영자가 가져갈 것 3가지

TKCC는 Together AI 규모의 시스템이지만, 설계 원리는 vLLM/SGLang 단일 노드 운영자에게도 적용된다.

**① 엔진 재시작은 측정의 적이다.** 튜닝 실험마다 `vllm serve`를 다시 띄우는 습관은 한 번에 5~8분을 태운다. 이탈 관점(핫 리로드, 체크포인트 보존)을 실험 파이프라인에 넣는 것만으로 하루 실험 수가 자릿수 단위로 바뀐다. 예컨대 튜닝 스크립트를 파라미터 스왑마다 재부팅하는 대신, 서빙 설정 중 실행 시점에 바꿀 수 있는 것(vLLM 기준 일부 스케줄러·캐시 파라미터)과 재시작이 필요한 것(텐서 병렬도, 양자화 포맷)을 분리해서 실험 매트릭스를 재설계할 수 있다.

```yaml
# 실험 매트릭스 분리 예시 (개념)
# 재시작 필요 그룹: 한 세트당 ~7분 → 하루 최대 ~200 실험
restart_group: [tensor_parallel_size, quantization, kv_cache_dtype]
# 핫 스왑 가능 그룹: 초 단위 순환 → 하루 수천 실험
hot_group: [max_num_seqs, prefix_caching, scheduling_policy, gpu_memory_utilization]
# 실험 순서: hot_group을 restart_group 고정값별로 먼저 소진
```

**② 오프라인 벤치마크를 믿지 마라.** MoE 전문가 불균형은 랜덤 입력으로는 재현되지 않는다. 커널이나 설정을 고르기 전에 실제 트래픽 샘플(실제 프롬프트 길이 분포, 실제 반복 프리픽스)으로 재측정하는 단계를 반드시 둘 것. TKCC의 "빠른 엔진/느린 엔진" 2단 구성은 이를 자동화한 것이다.

**③ 트래픽 드리프트를 모니터링 지표로.** "모델 출시 2주 뒤 프롬프트 길이 80% 증가"는 self-hosting에서도 똑같이 일어난다(신모델 붙이고 처음엔 챗 테스트, 이후 에이전트가 붙는 패턴). 평균 프롬프트 길이, 프리픽스 캐시 적중률, MoE 로드 밸런스 지표를 대시보드에 올려두고, 임계값을 넘으면 재튜닝을 트리거하는 구조가 TKCC의 '지속' 부분을 소규모로 흉내 내는 방법이다.

덧붙여 Together AI의 관련 연구 흐름을 같이 보면 맥이 잡힌다. 같은 팀은 2026년 7월 [ThunderAgent](https://alphasignal.ai/news/together-ai-s-thunderagent-doubles-gpu-throughput-running-192-agents-at-once)(ICML 2026 Spotlight)로 에이전트 워크플로의 KV 캐시 스래싱 문제를 스케줄러 계층에서 풀었고, 11월 TKCC로는 컴파일 계층을 자동화했다. ThunderAgent가 "프로그램 단위 스케줄링"이라면 TKCC는 "프로그램 단위 최적화 루프"다. 에이전트 시대의 추론 인프라가 요청 단위(request-level) 관점을 버려가는 과정으로 읽힌다.

## 결론

- Together Compiler는 최적화 결과물이 아니라 **최적화 루프 자체를 24시간 상시 가동**하는 '지속 컴파일' 시스템이다. 2계층 영속 엔진이 엔진 부팅 467초를 9초로 압축했고, 이것이 252배 실험 처리량과 24시간 만의 최대 2.6배 TPS로 이어졌다.
- 기술적 핵심은 측정 딜레마의 해소다. 커널 단위 측정의 빠름과 엔진 단위 측정의 정확함을 **빠른/느린 엔진 2계층**으로, 병렬 구성 선택은 **스펙·마이크로벤치·실측 3단 비용 모델**로 해결했다.
- 성능 수치(최대 2.6배, 1111/1205 TPS, 1.7배, 252배)는 Together AI 자체 발표 기준이므로 절대치보다 설계 방향을 보는 게 맞다. 특히 1111/1205 TPS가 자체 비용 모델의 구성 선택에서 나왔다는 점은 "컴파일러가 실제로 하는 일"의 실체를 보여준다.
- Self-hosting 운영자가 당장 가져갈 첫 단계는 이것이다: **튜닝 실험 매트릭스에서 '재시작 필요 파라미터'와 '핫 스왑 가능 파라미터'를 오늘 분리하라.** 그것이 TKCC의 영속 엔진 아이디어를 하루 만에 자기 파이프라인에 접목하는 가장 싼 방법이다.

**참고 자료**
- Together AI 블로그: [Together Compiler: Continual Compilation for AI inference](https://www.together.ai/blog/together-compiler-continual-compilation-for-ai-inference) (2026-11-04)
- 논문: [arXiv:2611.14681](https://arxiv.org/abs/2611.14681) (2026-11-10 제출, Reyna Abhyankar 외 12인)
- ThunderAgent 관련: [AlphaSignal 보도](https://alphasignal.ai/news/together-ai-s-thunderagent-doubles-gpu-throughput-running-192-agents-at-once) (ICML 2026 Spotlight)
