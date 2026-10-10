---
title: "GPU 한 장씩 걸어서 405B를 돌린다 — BloomBee가 인터넷 위 분산 LLM 추론의 통공 병목을 푸는 방식"
date: "2026-10-10"
keywords: ["분산 LLM 추론", "BloomBee", "Petals", "decentralized inference", "pipeline parallelism", "speculative decoding", "P2P GPU"]
lang: "ko"
description: "인터넷으로 연결된 이기종 GPU에서 대형 LLM을 서빙하는 분산 추론 프레임워크 BloomBee를 해부한다. 홉 수·데이터량·디코딩 스텝을 동시에 줄이는 3차원 통신 최적화와 실전 실행 코드를 정리했다."
---

# GPU 한 장씩 걸어서 405B를 돌린다 — BloomBee가 인터넷 위 분산 LLM 추론의 통신 병목을 푸는 방식

H100 클러스터를 못 빌리는 팀에게 405B급 모델은 여전히 닫힌 문이다. 양자화를 잘하면 소비자용 GPU 몇 장에도 올릴 수 있지만, 품질 손실 없이 풀 정밀도로 서빙하려면 결국 메모리를 여러 장에 걸쳐야 한다. 그런데 그 GPU들이 같은 데이터센터가 아니라 인터넷으로 연결된 집, 사무실, 학교 서버라면 이야기가 달라진다. 데이터센터 내부의 NVLink와 InfiniBand가 없는 환경, 즉 '인터넷 스케일'에서 분산 추론을 하려면 통신이 모든 것을 결정한다.

BloomBee는 이 문제를 정면에서 다루는 오픈소스 프레임워크다. 2026년 4월 arXiv에 공개된 논문 "Distributed Generative Inference of LLM at Internet Scales with Multi-Dimensional Communication Optimization"(arXiv:2604.21072)과 함께 GitHub(ai-decentralized/BloomBee)에 약 3만 1천 줄의 코드로 공개되어 있다. 기존 Petals를 확장한 시스템으로, 논문 보고 기준 Petals와 Helix 같은 기존 분산 추론 시스템 대비 서비스 처리량을 최대 1.76배 높이고 평균 지연을 최대 43.20% 낮췄다.

## 왜 인터넷 위 추론은 통신이 전부인가

같은 rack 안에서라면 GPU 간 통신은 GPUDirect RDMA로 거의 직통으로 흐른다. 그런데 열린 인터넷에서는 RDMA가 사실상 불가능하다. RDMA는 무손실(lossless) 패브릭을 요구하는데 인터넷 라우팅은 혼잡, 패킷 로스, 가변 MTU를 기본 전제로 하기 때문이다. 게다가 인터넷 통신에 필요한 TLS·IPsec·QUIC 같은 보안 계층은 NIC이나 GPU가 아니라 CPU에서 처리된다. 결국 어떤 최적화를 하든 활성화(activation) 텐서는 반드시 CPU 메모리를 거쳐가고, 여기서 네트워크 전송이 GPU 계산과 분리된다.

이 구조적 제약 아래에서 통신 오버헤드는 두 차원으로 나뉜다.

1. **홉(hop)의 수** — 파이프라인에서 활성화가 노드 사이를 몇 번 건너뛰는가
2. **홉당 데이터량** — 각 전송이 몇 바이트를 실어 나르는가

골치 아운 지점은 이 둘이 얽혀 있다는 것이다. 노드당 더 많은 레이어를 배치하면 홉 수는 줄지만 노드의 GPU 메모리가 부족해진다. 마이크로배치를 작게 잡으면 통신과 계산을 겹칠 수 있지만 파이프라인 버블이 커진다. BloomBee의 핵심 기여는 이 세 기법을 개별적으로 적용하는 게 아니라, GPU 메모리 예산 제약 아래에서 하나의 최적화 문제로 묶어 동적 계획법(dynamic programming)으로 푸는 것이다.

## 3차원 통신 최적화: 레이어 배치, 마이크로배칭, 텐서 오프로딩

**레이어 배치(layer assignment)**는 트랜스포머 블록을 어느 노드에 몇 개씩 놓을지 정한다. 현대 LLM이 동일한 블록의 반복이라는 점을 이용해, 각 노드에 연속된 블록 구간을 할당한다. 이기종 GPU와 네트워크 대역폭의 이질성을 식에 넣고, 제약된 형태의 문제라 가벼운 동적 계획법으로 풀 수 있다.

**텐서 오프로딩(tensor offloading)**은 반대 방향의 트릭이다. 배치 크기와 시퀀스 길이에 비례해 선형적으로 자라는 KV 캐시를 GPU가 부족할 때 CPU 메모리로 내보낸다. 이렇게 비워진 GPU 용량에 블록을 더 올리면 전체 홉 수가 줄어든다. 즉 "메모리를 희생해 통신을 산다"는 거래를 최적화 문제 안에서 자동으로 조정한다.

**마이크로배치 파이프라이닝**은 남은 차원, 즉 홉당 데이터량과 대기 시간을 공략한다. 요청 배치를 배치 차원으로 쪼개 작은 마이크로배치로 만들면, 한 노드가 마이크로배치 하나를 계산하는 동안 다른 마이크로배치의 활성화가 네트워크를 흐른다. 앞서 설명했듯 활성화가 어차피 CPU 메모리를 경유하므로, GPU가 계산을 마치면 DMA로 결과를 CPU에 던져놓고 곧바로 다음 마이크로배치를 시작할 수 있다. 파이프라인이 차오르면 모든 스테이지가 서로 다른 마이크로배치를 동시에 처리한다.

## 저대역폭용 추측 디코딩 — 로컬에서 약이 분산에서 독이 되는 순간

흥미로운 부분은 추측 디코딩(speculative decoding)의 재해석이다. 로컬 추론에서 추측 디코딩은 드래프트 모델이 여러 토큰을 한 번에 제안하고 대상 모델이 병렬로 검증하는 가속 기법이다. 그런데 파이프라인이 인터넷으로 흩어져 있으면 토큰 하나를 검증하는 것조차 여러 홉을 왕복해야 한다. 추측 디코딩은 이 왕복 횟수 자체를 줄여주므로 분산에서도 유효하다.

문제는 검증할 후보 토큰이 늘어나면 홉당 전송량도 커진다는 것이다. BloomBee는 여기에 가벼운 분류기를 붙여 네트워크로 보내기 전에 드래프트 토큰을 가지치기(pruning)한다. 논문 설명에 따르면 토큰 수용률(acceptance rate)을 크게 해치지 않으면서 대역폭 요구를 줄이는 방향으로 설계됐다. 여기에 무손실 압축까지 더해져 홉당 바이트 수를 추가로 줄인다. 요약하면 BloomBee는 **홉 수 × 홉당 데이터량 × 디코딩 스텝 수**라는 세 인자를 동시에 최소화하는 설계다.

## 직접 돌려보기: P2P 네트워크에 레이어를 올리는 법

BloomBee는 Petals처럼 HuggingFace Transformers 인터페이스를 그대로 쓴다. 내부적으로 hivemind(libp2p 기반 P2P)와 FlexLLMGen(오프로딩) 위에 만들어졌고, DHT가 어떤 워커가 어떤 블록을 호스팅하는지 추적한다.

워커 실행 — 각 서버가 모델의 일부 블록을 맡는다:

```bash
# Worker 1: 트랜스포머 블록 16개 호스팅
python -m bloombee.cli.run_server meta-llama/Llama-2-7b-hf \
  --initial_peers $BBSERVER \
  --num_blocks 16 \
  --identity_path worker_1.id

# Worker 2: 나머지 16개 호스팅
python -m bloombee.cli.run_server meta-llama/Llama-2-7b-hf \
  --initial_peers $BBSERVER \
  --num_blocks 16 \
  --identity_path worker_2.id
```

클라이언트는 임베딩과 LM 헤드만 로컬로 실행하고, 나머지 레이어는 DHT로 발견한 원격 피어들을 경유한다:

```python
from transformers import AutoTokenizer
from bloombee import AutoDistributedModelForCausalLM

model = AutoDistributedModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    initial_peers=["/ip4/YOUR_IP/tcp/31340/p2p/Qm..."],
)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b-hf")

inputs = tokenizer("The quick brown fox", return_tensors="pt")
outputs = model.generate(**inputs, max_new_tokens=50)
print(tokenizer.decode(outputs[0]))
```

이기종 GPU 환경에서는 `pip install torch`가 함정이 될 수 있다. BloomBee는 로컬 NVIDIA 드라이버의 CUDA 버전과 GPU 컴퓨트 capability를 읽어 호환되는 PyTorch 휠을 골라주는 스크립트를 제공한다. 예컨대 Tesla P100(파스칼, SM 6.0) 호스트는 CUDA 12.1 휠에 고정되고, 새 드라이버를 가진 최신 GPU는 더 새 휠을 쓴다:

```bash
python scripts/install_compatible_torch.py --dry-run  # 결정 미리보기
python scripts/install_compatible_torch.py && pip install -e .
```

## 기대치 조정: 벤치마크가 말해주는 것과 말해주지 않는 것

논문의 1.76배 처리량 개선과 43.20% 지연 감소는 Petals와 Helix를 기준 시스템으로 삼은 결과다. 즉 "분산 추론 시스템끼리의 비교"이지, 데이터센터의 vLLM 같은 중앙집중 서빙과의 비교가 아니다. 인터넷 대역폭이 병목인 이상, 아무리 잘 최적화해도 같은 건물 안 H100 팜과의 물리적 격차는 좁히기 어렵다. 선행 연구인 Petals가 커뮤니티 네트워크에서 Llama 2 70B 싱글 배치 추론을 초당 수 토큰 수준으로 서빙했다는 사실은, 이 접근의 실용 영역이 채팅 인터페이스 정도임을 보여준다.

그럼에도 이 방향이 의미 있는 경우는 명확하다. 첫째, 취미·연구 목적으로 유휴 GPU를 모아 프론티어급 오픈 모델을 풀 정밀도로 써보고 싶을 때. 둘째, 프라이버시나 데이터 주권 때문에 외부 API에 의존할 수 없을 때. 셋째, 단일 벤더 의존을 피하면서 유휴 컴퓨팅을 pooling하는 인프라를 실험할 때다.

## 결론

- 인터넷 스케일 분산 추론의 병목은 계산이 아니라 통신이며, 통신은 **홉 수, 홉당 데이터량, 디코딩 스텝 수**의 곱으로 볼 수 있다.
- BloomBee는 레이어 배치, 마이크로배칭, KV 캐시 오프로딩을 GPU 메모리 예산 안에서 동적 계획법으로 jointly 최적화하고, 저대역폭에 맞게 추측 디코딩(드래프트 토큰 가지치기)과 무손실 압축을 얹는다.
- 논문 보고 기준으로 기존 분산 추론 시스템(Petals, Helix) 대비 최대 1.76배 처리량, 최대 43.20% 지연 감소를 달성했으며, 코드와 논문이 모두 공개되어 있다.
- 사용법은 Transformers의 `AutoDistributedModelForCausalLM` 하나로 요약된다. 워커 두 개를 띄워 직접 검증해볼 수 있는 진입 장벽이다.
- 다만 중앙집중 서빙을 대체하는 기술은 아니다. 느린 링크 위에서 '그래도 쓸만한' 수준까지 끌어올리는 기술로 이해해야 한다.

직접 확인해보고 싶다면 GitHub 저장소(ai-decentralized/BloomBee)를 클론하고, GPU 한 장짜리 머신 두 대로 7B 모델을 16블록씩 나눠 띄워보는 것이 가장 빠른 첫걸음이다. 논문은 arXiv:2604.21072에서 읽을 수 있다.
