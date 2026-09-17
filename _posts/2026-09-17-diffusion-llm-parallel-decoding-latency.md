---
title: "확산 언어 모델은 왜 초당 1,000 토큰을 뽑는가 — LLaDA에서 DiffusionGemma까지 병렬 디코딩의 원리"
date: "2026-09-17"
keywords: ["확산 언어 모델", "diffusion LLM", "DiffusionGemma", "LLaDA", "병렬 디코딩", "dLLM"]
lang: "ko"
description: "자기회귀 LLM을 대신해 한 번의 순전파로 256 토큰을 만들어내는 확산 언어 모델(dLLM)의 원리와 LLaDA·Mercury·DiffusionGemma·Nemotron Diffusion의 실측 성능, 그리고 정직한 트레이드오프를 정리한다."
---

# 확산 언어 모델은 왜 초당 1,000 토큰을 뽑는가 — LLaDA에서 DiffusionGemma까지 병렬 디코딩의 원리

LLM 추론의 병목은 연산량이 아니라 메모리 대역폭인 경우가 많다. 자기회귀(Autoregressive, AR) 모델은 토큰 하나를 만들 때마다 모델 전체를 통과해야 하고, 배치 크기가 작은 로컬 추론 환경에서는 GPU가 다음 "키 입력"을 기다리며 놀고 있다. Google이 2026년 6월에 공개한 [DiffusionGemma](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/)는 이 낭비를 뒤집는 접근이다. 단일 H100에서 초당 1,000 토큰 이상, 가정용 RTX 5090에서도 초당 700 토큰 이상을 내는 실험적 오픈 모델이다.

이 글에서는 확산 언어 모델(diffusion LLM, dLLM)이 정확히 무엇이고, 어떤 모델들이 실제로 출시됐으며, 무엇을 포기하고 무엇을 얻는 기술인지 정리한다.

## 마스킹 확산: BERT와 닮았지만 생성 모델이다

dLLM의 핵심 아이디어는 이미지 확산 모델의 "노이즈를 걷어내며 복원한다"는 절차를 이산적인 텍스트 토큰 위에 옮긴 것이다. 텍스트에는 가우시안 노이즈를 더할 수 없으므로, 대신 토큰을 `[MASK]`로 점차 바꾸는 **마스킹 확산(masked diffusion)** 이 표준이 됐다. D3PM 계열의 이산 확산 이론 위에 세워진 방식이다.

동작은 단순하다:

1. 답변 영역 전체를 마스크로 채운다.
2. 트랜스포머가 마스킹된 모든 위치의 토큰을 **한 번에** 예측한다.
3. 신뢰도가 높은 토큰부터 확정하고(저신뢰도 재마스킹), 애매한 토큰은 다시 마스크로 돌려 다음 단계에서 다듬는다.
4. 마스크가 없어질 때까지 반복한다.

BERT의 마스크 언어 모델링과 비슷해 보이지만 결정적 차이가 있다. BERT는 약 15%를 고정 비율로 가려 한 번에 맞히는 표현 학습 목표인 반면, 마스킹 확산은 마스킹 비율을 0에서 1 사이로 무작위로 바꾸고 여기에 1/t 가중치를 곱한 교차 엔트로피 손실을 쓴다. 이 목적 함수는 음의 로그 가능도에 대한 이론적 상한이므로(ELBO의 재구성), 빈 문장에서 완결된 텍스트를 샘플링할 수 있는 **원리적인 생성 모델**이 된다. BERT는 빈칸 채우기는 잘하지만 문장을 만들지 못한다.

마스킹 확산 디코딩의 핵심 루프는 Python 의사코드로 20줄 안에 쓸 수 있다:

```python
# 마스킹 확산 디코딩 (개념 코드)
def diffuse_generate(model, prompt, n_steps=32, gen_len=256):
    tokens = prompt + [MASK] * gen_len
    for t in reversed(range(n_steps, 0, -1)):
        logits = model(tokens)                    # 마스크 위치 전부 동시 예측
        pred = logits.argmax(dim=-1)
        conf = logits.softmax(dim=-1).max(dim=-1)
        budget = gen_len * t // n_steps           # 이번 단계에 확정할 개수
        # 신뢰도 상위 budget개만 확정, 나머지는 재마스킹
        keep = topk_indices(conf, budget)
        for i in keep:
            tokens[i] = pred[i]
    return tokens[len(prompt):]
```

한 번의 순전파에서 여러 토큰을 만들어 내므로, 남는 연산 자원을 지연 시간 감소로 바꿀 수 있다. 이것이 AR 대비 속도 우위의 원천이다.

## 실제로 출시된 모델들: 연구에서 상용까지

### LLaDA — "확장성은 자기회귀가 아니라 생성 원리에서 나온다"

인민대학과 Ant Group의 [LLaDA](https://arxiv.org/abs/2502.09992)(arXiv 2502.09992)는 사전학습부터 SFT까지 표준 LLM 절차 그대로 밑바닥부터 학습한 8B 마스킹 확산 모델이다. 논문의 핵심 주장은 도발적이다. 문맥 내 학습, 지시 따르기 같은 LLM의 핵심 능력은 자기회귀라는 특정 구조가 아니라 생성 모델링 원리 자체에서 나온다는 것. 실제로 LLaDA 8B는 문맥 내 학습에서 LLaMA3 8B와 경쟁 가능한 성능을 보였고, "A는 B다"를 배워도 "B는 A다"를 못 맞히는 AR 모델의 역전의 저주(reversal curse) 과제에서는 시 완성 역방향 프롬프트에서 GPT-4o를 능가했다고 보고한다.

### Mercury — 첫 상용 규모 dLLM

Inception Labs의 [Mercury](https://arxiv.org/abs/2506.17298)(arXiv 2506.17298)는 코드 특화 모델(Mercury Coder Mini/Small)로, 속도-품질 프론티어를 목표로 한다. Artificial Analysis의 독립 측정 기준으로 H100에서 Mini 초당 1,109 토큰, Small 초당 737 토큰을 기록했으며, 속도 최적화 프론티어 모델 대비 평균 최대 10배 빠르다. 개발자 실전 평가장인 Copilot Arena에서는 품질 공동 2위, 속도 전체 1위였다. 다만 매개변수 규모와 학습 데이터 구성은 공개되지 않은 상용 보고서라는 점은 감안해야 한다.

### DiffusionGemma — vLLM이 네이티브 지원한 첫 dLLM

[DiffusionGemma](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/)는 Google DeepMind가 Apache 2.0 라이선스로 공개한 26B MoE(활성 3.8B) 모델이다. 256 토큰짜리 "캔버스"를 무작위 토큰으로 초기화한 뒤 디코더가 캔버스 전체를 병렬 복원하고, 수렴하면 인코더가 결과를 커밋해 다음 캔버스로 넘어가는 블록 확산 구조를 쓴다. 캔버스 안은 병렬, 캔버스 사이는 자기회귀인 셈이다.

[vLLM 팀의 통합 블로그](https://vllm.ai/blog/2026-06-10-diffusion-gemma)에 따르면 DiffusionGemma는 vLLM이 네이티브로 지원한 최초의 dLLM이며, FP8 기준 H200에서 초당 1,288 토큰, H100에서 초당 1,008 토큰을 배치 크기 1에서 기록했다. 흥미로운 설계는 각 캔버스를 speculative decoding의 드래프트 집합처럼 다뤄 기존 추측 디코딩 데이터 경로를 재사용했다는 점이다. 인코더가 AR과 동일하게 KV 캐시를 쓰므로 접두사 캐싱도 그대로 동작한다.

### Nemotron Diffusion — AR과 확산을 한 모델에

NVIDIA의 [Nemotron-Labs-Diffusion](https://github.com/NVlabs/Nemotron-Labs-Diffusion)(arXiv 2607.05722)은 3B/8B/14B 규모로, 어텐션 패턴만 바꿔 자기회귀·확산·자기추측(self-speculation) 세 가지 디코딩 모드를 모두 지원한다. 자기추측 디코딩이 특히 실용적인데, 같은 모델이 확산 모드로 여러 토큰을 병렬 초안으로 뽑고 AR 모드로 검증하므로 별도의 드래프트 모델이 필요 없다. 기술 보고서에 따르면 Qwen3-8B 대비 순전파당 약 5.9배 많은 토큰을 같은 정확도로 생성하며, GB200 동시성 1 기준으로 AR의 초당 256 토큰을 자기추측으로 초당 851 토큰까지 끌어올렸다고 한다.

| 모델 | 개발사 | 규모 | 핵심 특징 | 검증된 속도 |
|------|--------|------|-----------|------------|
| LLaDA 8B | 인민대·Ant Group | 8B dense | 밑바닥부터 학습, 역전의 저주 극복 | — (연구 모델) |
| Mercury Coder | Inception Labs | 비공개 | 첫 상용 dLLM, OpenAI 호환 API | H100 초당 1,109 토큰 (Mini) |
| DiffusionGemma | Google DeepMind | 26B MoE / 활성 3.8B | Apache 2.0, vLLM 네이티브 지원 | H100 초당 1,000+ 토큰 |
| Nemotron Diffusion | NVIDIA | 3B/8B/14B | AR+확산+자기추측 3모드 | GB200 초당 851 토큰 |

## 정직한 트레이드오프: 품질이 아니라 지연 시간을 위한 기술

dLLM을 "AR보다 똑똑한 모델"로 오해하면 안 된다. Google 스스로 명시하듯 DiffusionGemma의 전체 출력 품질은 표준 Gemma 4보다 낮으며, 최대 품질이 필요한 용도에는 Gemma 4를 권장한다. 모델 카드 기준으로 MMLU Pro 77.6% 대 82.6%, AIME 2026(도구 미사용) 69.1% 대 88.3% 등 주요 벤치마크에서 AR 쪽이 앞선다.

대신 얻는 것은 저동시성 환경의 지연 시간이다. 클라우드에서는 수천 요청을 배칭해 GPU를 채울 수 있지만, 로컬 단일 사용자 추론에서 AR 모델은 메모리 대역폭에 발목 잡힌다. dLLM은 병목을 메모리 대역폭에서 연산으로 옮겨, 한 번의 순전파로 텍스트 블록 전체를 초안 작성한다. 인라인 편집, 코드 인필링, 빠른 반복 프로토타이핑 같은 속도 민감 워크플로우가 주 타깃이다.

양방향 어텐션이 주는 독특한 부수효과도 있다. 각 토큰이 미래 토큰을 참조할 수 있어, Google이 예시로 든 것처럼 Unsloth가 파인튜닝한 DiffusionGemma가 스도쿠를 푸는 데 유리하다. 모든 칸이 서로 의존하는 과제는 왼쪽에서 오른쪽으로만 읽는 AR 모델이 구조적으로 불리하기 때문이다.

## 디코딩 최적화: 정답은 이미 절반 지점에 나와 있다

dLLM의 속도는 디코딩 전략에 따라 크게 달라진다. 두 연구가 대표적이다.

**[Prophet](https://arxiv.org/abs/2508.19982)**(arXiv 2508.19982)은 "dLLM은 디코딩이 끝나기 전에 이미 정답을 안다"는 관찰에서 출발한다. 연구에 따르면 LLaDA 8B와 Dream 7B에서 무작위 재마스킹 기준으로 GSM8K는 최대 97%, MMLU는 최대 99%의 사례가 전체 복원 단계의 절반만으로 정답에 도달한다. Prophet은 답변 영역의 신뢰도 격차가 임계값을 넘으면 남은 마스크를 한 번에 확정하고 종료하는 방식으로, 재학습 없이 복원 단계를 최대 약 3.4배 줄이면서 품질을 유지했다. KV 캐시 최적화와 곱셈적으로 결합되어 Fast-dLLM 단독 약 6.82배 가속이 결합 시 약 7.66배로 늘어난다는 보고도 있다.

**CCD(Coherent Contextual Decoding)**는 한 단계의 국소적 확신도 대신 최근 여러 단계에 걸친 예측의 일관성을 본다. 여러 단계에 걸쳐 꾸준히 같은 위치를 확신하는 토큰만 확정하는 궤적 정정 방식으로, Dream 기준 Trip Planning 과제에서 단계 수를 256에서 약 75로 줄이면서 점수를 오히려 높였다는 결과가 보고됐다.

## 직접 실행해 보기

DiffusionGemma는 vLLM 네이티브 지원 덕분에 기존 서빙 스택에서 바로 돌려볼 수 있다. 모델은 [Hugging Face](https://huggingface.co/google/diffusiongemma-26B-A4B-it)에서 `google/diffusiongemma-26B-A4B-it`로 공개돼 있다.

```bash
pip install -U vllm
vllm serve google/diffusiongemma-26B-A4B-it
# OpenAI 호환 API로 교체 가능 — 기존 클라이언트 코드 수정 불필요
```

양자화 시 상단 소비자 GPU의 18GB VRAM 한도 안에 들어온다고 Google은 명시했다. 로컬에서 초당 수백 토큰의 인터랙티브한 초안 생성 워크플로우를 실험해 볼 수 있는 최초의 "실용적" dLLM이라는 의미가 크다.

## 결론

- dLLM은 마스킹 확산으로 문장 전체를 병렬 복원하는 생성 모델이다. BERT의 빈칸 채우기가 아니라, 가능도 상한을 최적화하는 원리적인 생성기다.
- 속도는 실측으로 검증됐다. Mercury 초당 1,109 토큰, DiffusionGemma H100 초당 1,000+ 토큰, Nemotron 자기추측 초당 851 토큰 — 모두 제3자 또는 공개 벤치마크 기준이다.
- 그러나 품질에서 AR을 앞서는 기술이 아니다. Google 스스로 "최대 품질이 필요하면 Gemma 4를 쓰라"고 권한다. dLLM은 저동시성·지연 민감 워크로드를 위한 기술이다.
- 디코딩 최적화(Prophet, CCD)만으로도 수 배의 추가 가속이 가능하며, 아키텍처 개선과 곱셈적으로 결합된다.
- 관심 있다면 vLLM으로 DiffusionGemma를 직접 띄워 보는 것이 가장 빠른 시작점이다.

**참고 자료**: [LLaDA (arXiv 2502.09992)](https://arxiv.org/abs/2502.09992) · [Mercury (arXiv 2506.17298)](https://arxiv.org/abs/2506.17298) · [DiffusionGemma 발표 (Google)](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/) · [vLLM 통합 블로그](https://vllm.ai/blog/2026-06-10-diffusion-gemma) · [Nemotron-Labs-Diffusion (GitHub)](https://github.com/NVlabs/Nemotron-Labs-Diffusion) · [Prophet (arXiv 2508.19982)](https://arxiv.org/abs/2508.19982) · [A Survey on Diffusion Language Models (arXiv 2508.10875)](https://arxiv.org/abs/2508.10875) · [PyTorchKR 연구 정리](https://discuss.pytorch.kr/t/diffusion-llm-llada-diffusiongemma/11311)
