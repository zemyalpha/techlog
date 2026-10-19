---
title: "한국어 오픈소스 TTS의 기준이 바뀌었다 — Qwen3-TTS 12Hz 실전 가이드와 Voxtral TTS 비교"
date: "2026-10-19"
keywords: ["Qwen3-TTS", "오픈소스 TTS", "한국어 TTS", "Voxtral TTS", "보이스 클로닝", "음성 합성"]
lang: "ko"
description: "아파치 라이선스로 풀린 Qwen3-TTS(0.6B/1.7B)의 한국어 합성 품질과 실전 사용법, 그리고 한국어를 지원하지 않는 Mistral Voxtral TTS와의 정확한 비교."
---

# 한국어 오픈소스 TTS의 기준이 바뀌었다 — Qwen3-TTS 12Hz 실전 가이드와 Voxtral TTS 비교

로컬에서 돌리는 오픈소스 TTS는 오랫동안 두 가지 아쉬움을 안고 있었습니다. 하나는 한국어 품질이 영어·중국어에 비해 확실히 뒤떨어진다는 것, 다른 하나는 상업적 사용이 막힌 라이선스가 많다는 것이었습니다. 2026년 들어 이 구도를 흔드는 발표가 두 번 있었습니다. 알리바이 클라우드의 **Qwen3-TTS**(1월 22일 공개)와 미스트랄의 **Voxtral TTS**(3월 23일 공개)입니다.

두 모델은 언뜻 비슷해 보이지만 한국어 사용자에게는 결정적으로 다릅니다. Qwen3-TTS는 한국어를 정식 지원하고 Apache-2.0 라이선스로 상업적 사용까지 열어뒀습니다. 반면 Voxtral TTS는 9개 언어만 지원하는데 그 목록에 한국어가 없고, 공개된 가중치는 CC BY-NC 4.0, 즉 비상업 용도로 제한됩니다. 이 글에서는 두 모델의 실제 차이를 공식 문서 기반으로 짚고, Qwen3-TTS를 로컬에서 바로 돌려보는 코드까지 정리합니다.

## 2026년 오픈소스 TTS 지형: 두 개의 큰 물결

### Qwen3-TTS — 한국어 포함 10개 언어, Apache-2.0

Qwen 팀은 2026년 1월 22일 Qwen3-TTS 시리즈(0.6B/1.7B)를 공개했습니다. GitHub 저장소(QwenLM/Qwen3-TTS)는 글 작성 시점 기준 약 13,900스타를 기록 중이며, 라이선스는 Apache-2.0입니다. 지원 언어는 중국어, 영어, 일본어, **한국어**, 독일어, 프랑스어, 러시아어, 포르투갈어, 스페인어, 이탈리아어의 10개입니다.

모델은 용도별로 다섯 가지로 나뉩니다.

| 모델 | 역할 | 스트리밍 | 명령 제어 |
|---|---|---|---|
| Qwen3-TTS-12Hz-1.7B-VoiceDesign | 텍스트 설명으로 목소리 자체를 설계 | 지원 | 지원 |
| Qwen3-TTS-12Hz-1.7B-CustomVoice | 9종 프리미엄 음색 + 스타일 지시 | 지원 | 지원 |
| Qwen3-TTS-12Hz-1.7B-Base | 3초 오디오로 보이스 클론, 파인튜닝 베이스 | 지원 | 미지원 |
| Qwen3-TTS-12Hz-0.6B-CustomVoice | 경량 커스텀 보이스 | 지원 | 미지원 |
| Qwen3-TTS-12Hz-0.6B-Base | 경량 클론/파인튜닝 베이스 | 지원 | 미지원 |

CustomVoice 모델에 내장된 9개 음색 중에는 한국어 원어 화자인 **Sohee**(풍부한 감정의 따뜻한 한국어 여성 음성)가 포함되어 있어, 참고 오디오 없이도 바로 한국어 합성이 가능합니다.

### Voxtral TTS — 강력하지만 한국어는 없다

미스트랄이 3월 23일 공개한 Voxtral TTS는 다른 의미에서 인상적입니다. 4B 파라미터(3.4B 트랜스포머 디코더 + 390M flow-matching 어쿠스틱 모델, Ministral 3B 기반)로 사람 평가에서 ElevenLabs v2.5 Flash를 자연성에서 앞선다는 결과를 냈고, 10초 오디오·500자 입력 기준 모델 지연 70ms, 실시간 배수(RTF) 약 9.7배라는 스트리밍 성능을 보여줍니다. 공식 발표에 따르면 3초 참고 음성만으로 화자의 억양, 억누르지 못하는 말버릇까지 반영하는 적응이 가능하고, 영어 텍스트에 프랑스어 음성 프롬프트를 주면 프랑스 억양 영어가 나오는 제로샷 교차 언어 적응도 보여줍니다.

문제는 한국어입니다. 공식 지원 언어는 영어, 프랑스어, 독일어, 스페인어, 네덜란드어, 포르투갈어, 이탈리아어, 힌디어, 아랍어 9개뿐이며 한국어는 포함되지 않았습니다. 여기에 Hugging Face에 공개된 가중치(mistralai/Voxtral-4B-TTS-2603)의 라이선스가 **CC BY-NC 4.0**이라 상업적 이용이 불가합니다. 상업 용도로 쓰려면 문자당 과금되는 미스트랄 API(1천 자당 $0.016부터)를 쓰는 구조입니다. 즉, 한국어 사용자에게 Voxtral은 "기술적으로 흥미롭지만 바로 쓸 수는 없는" 모델입니다.

## Qwen3-TTS의 설계: 왜 12Hz가 중요한가

Qwen3-TTS를 이해하려면 이름에 붙은 "12Hz"부터 봐야 합니다. 이는 음성을 1초에 12개 토큰으로 압축하는 자체 토크나이저(Qwen3-TTS-Tokenizer-12Hz)를 뜻합니다. 낮은 프레임 레이트로 압축하되 운율·감정 같은 준언어 정보와 음향 환경 정보를 보존하는 방향으로 설계됐다고 공식 문서는 설명합니다.

두 번째 특징은 **discrete multi-codebook LM 구조의 종단간(end-to-end) 아키텍처**입니다. 기존의 "언어 모델 + DiT(Diffusion Transformer)" 2단계 구성은 음질을 좇으면 지연이 커지고, 단계 사이에서 정보 병목과 오류 누적이 발생합니다. Qwen3-TTS는 DiT 없이 가벼운 디코더로 고음질 재구성을 한다는 점을 강점으로 내세웁니다.

세 번째는 **Dual-Track 하이브리드 스트리밍**입니다. 하나의 모델이 스트리밍과 비스트리밍을 모두 지원하며, 입력 첫 글자 직후 첫 오디오 패킷을 내보낼 수 있고 종단간 합성 지연이 최소 97ms라고 합니다. 실시간 통화 에이전트를 만들 때 체감 지연을 좌우하는 숫자입니다.

## 실전: 10분 안에 로컬 한국어 TTS 띄우기

### 설치

공식 권장은 격리된 파이썬 3.12 환경입니다.

```bash
conda create -n qwen3-tts python=3.12 -y
conda activate qwen3-tts
pip install -U qwen-tts

# GPU 메모리 절약을 위해 권장 (선택)
pip install -U flash-attn --no-build-isolation
```

가중치는 모델명을 지정하면 로드 시 자동 다운로드됩니다. 오프라인 환경이라면 `huggingface-cli download Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice --local-dir ./Qwen3-TTS-12Hz-1.7B-CustomVoice`처럼 미리 받아두면 됩니다.

### 한국어 합성: 내장 음성 Sohee 사용

```python
import torch
import soundfile as sf
from qwen_tts import Qwen3TTSModel

model = Qwen3TTSModel.from_pretrained(
    "Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice",
    device_map="cuda:0",
    dtype=torch.bfloat16,
    attn_implementation="flash_attention_2",
)

wavs, sr = model.generate_custom_voice(
    text="오늘도 좋은 하루 되세요. 곧 점심시간이에요.",
    language="Korean",          # "Auto"로 두면 자동 판별
    speaker="Sohee",            # 한국어 원어 여성 음성
    instruct="차분하고 부드러운 톤으로",  # 생략 가능
)
sf.write("output.wav", wavs[0], sr)
```

`model.get_supported_speakers()`와 `get_supported_languages()`로 현재 모델이 지원하는 음성·언어 목록을 런타임에 확인할 수 있습니다.

### 3초 보이스 클론

Base 모델은 참고 오디오 3초면 새 화자의 목소리로 임의의 텍스트를 합성합니다.

```python
model = Qwen3TTSModel.from_pretrained(
    "Qwen/Qwen3-TTS-12Hz-1.7B-Base",
    device_map="cuda:0",
    dtype=torch.bfloat16,
    attn_implementation="flash_attention_2",
)

wavs, sr = model.generate_voice_clone(
    text="안녕하세요, 제 목소리로 합성된 문장입니다.",
    language="Korean",
    ref_audio="my_voice_3sec.wav",   # 로컬 경로, URL, base64 모두 가능
    ref_text="참고 오디오의 원문 스크립트",
)
sf.write("clone.wav", wavs[0], sr)
```

같은 화자로 여러 문장을 뽑을 때는 `create_voice_clone_prompt()`로 프롬프트를 한 번 만들어 재사용하면 참고 음성 특징 추출을 반복하지 않아도 됩니다. 참고 스크립트가 없는 오디오라면 `x_vector_only_mode=True` 옵션으로 화자 임베딩만 쓸 수 있지만, 공식 문서도 클론 품질이 낮아질 수 있다고 명시하니 가급적 원문을 함께 제공하는 편이 좋습니다.

저장소에는 `finetuning/` 디렉터리가 포함되어 있어 Base 모델을 자체 데이터로 파인튜닝하는 것도 공식 경로로 지원되고, vLLM으로 서빙하는 방법도 문서화되어 있습니다.

## 벤치마크: 한국어에서 어느 정도일까

주의할 점부터 짚으면, 아래 수치는 Qwen이 자체 공개한 벤치마크(공식 README)입니다. 독립 검증이 아니라는 점은 염두에 두고 읽어야 합니다.

중국어·영어 Seed-TTS 테스트셋의 콘텐츠 일관성(Word Error Rate, 낮을수록 좋음)에서 Qwen3-TTS-12Hz-1.7B-Base는 중국어 0.77, 영어 1.24를 기록했고, 영어 1.24는 테이블 전체에서 가장 낮은 값이었습니다. 비교 대상 중 CosyVoice 3는 중국어 0.71로 중국어 최고 성적을 유지했습니다.

다국어 테스트셋의 한국어 WER은 이렇습니다.

| 모델 | 한국어 WER (↓) |
|---|---|
| Qwen3-TTS-25Hz-0.6B-Base | 2.852 |
| Qwen3-TTS-25Hz-1.7B-Base | 2.631 |
| Qwen3-TTS-12Hz-0.6B-Base | **1.741** |
| Qwen3-TTS-12Hz-1.7B-Base | 1.755 |
| MiniMax | 1.747 |
| ElevenLabs | 1.865 |

흥미로운 점은 12Hz-**0.6B**가 1.7B보다 한국어에서 미세하게 앞선다는 것입니다. 벤치마크 표에서도 12Hz-0.6B의 1.741이 최고치로 볼드 처리되어 있습니다. 작은 모델이 한국어에서는 더 나은 결과를 보인다는 자체 벤치마크 수치인 만큼, "무조건 큰 모델"보다는 실제 오디오를 듣고 고르는 편이 좋겠습니다. 다만 차이가 0.014포인트 수준이므로 통계적으로 유의미하다고 단정하기는 어렵습니다.

WER은 발화 내용의 정확성만 측정합니다. 자연스러움, 감정 표현, 화자 유사도는 별도 평가가 필요하며, 이 영역에서는 사람 평가가 여전히 기준입니다. Voxtral TTS 발표에서 미스트랄이 "자동 지표는 자연스러움을 측정할 수 없다"며 원어민 인간 평가를 강조한 것도 같은 맥락입니다.

## 선택 가이드: 어떤 상황에서 뭘 쓸까

- **한국어 오디오북·내레이션** — Qwen3-TTS가 사실상 유망한 선택지입니다. 한국어 정식 지원 + Apache-2.0 + 파인튜닝 코드 제공이라 자체 데이터로 품질을 끌어올릴 수 있습니다.
- **한국어 실시간 음성 에이전트** — Qwen3-TTS의 스트리밍(첫 패킷 지연 최소 97ms)이 설계 목표에 명시되어 있습니다. 다만 97ms는 공식 발표 최솟값이므로 실제 하드웨어에서 측정 후 판단해야 합니다.
- **영어·유럽어 상용 서비스** — Voxtral TTS 가중치는 비상업(CC BY-NC 4.0) 제한이 있으니, 상업 용도라면 미스트랄 API를 쓰거나 Qwen3-TTS(Apache-2.0)를 검토하세요.
- **라이선스 확인은 본인 책임** — Qwen3-TTS 코드·가중치의 Apache-2.0은 확인됐지만, 실제 서비스 적용 전 각 모델 페이지의 최신 라이선스를 다시 확인하는 것이 안전합니다.

한 가지 짚을 점은 "오픈소스 TTS가 상용 API를 완전히 대체했다"는 식의 결론은 아직 이르다는 것입니다. ElevenLabs 같은 상용 서비스는 다국어 혼합, 장문 안정성, 노이즈 취급 등 실전 영역에서 여전히 강점이 있습니다. 오픈소스의 의미는 "충분히 쓸 만한 기본기가 갖춰졌고, 데이터와 파인튜닝으로 내 것으로 만들 수 있는 길이 생겼다"는 데 있습니다.

## 결론

- Qwen3-TTS는 한국어를 정식 지원하는 10개 언어 오픈소스 TTS로, Apache-2.0 라이선스로 상업적 사용이 가능합니다.
- 자체 벤치마크에서 한국어 WER 1.741(12Hz-0.6B-Base)로, 발표 시점 비교군(MiniMax 1.747, ElevenLabs 1.865)과 근접하거나 소폭 앞선 수치를 보였습니다. 단, 자체 평가라는 한계는 있습니다.
- `pip install qwen-tts` 후 열 몇 줄의 코드로 한국어 합성과 3초 보이스 클론이 가능하고, 파인튜닝·vLLM 서빙 경로도 공식 제공됩니다.
- Voxtral TTS는 자연성과 지연(70ms)에서 인상적이지만 한국어 미지원 + 비상업 라이선스라 한국어 프로젝트에는 바로 쓸 수 없습니다.
- 첫 단계는 간단합니다. 커스텀 보이스 모델을 받아 Sohee 음성으로 자기 문장을 합성해 직접 들어보는 것부터 시작하세요. 벤치마크 숫자보다 자기 귀가 더 정확한 판단 기준입니다.

**참고 자료**: [QwenLM/Qwen3-TTS GitHub 저장소](https://github.com/QwenLM/Qwen3-TTS)(모델 구성·벤치마크·코드 예시), [Speaking of Voxtral — Mistral AI 공식 발표](https://mistral.ai/news/voxtral-tts/)(Voxtral TTS 성능·라이선스·언어 목록), [mistralai/Voxtral-4B-TTS-2603 모델 카드](https://huggingface.co/mistralai/Voxtral-4B-TTS-2603)(CC BY-NC 4.0 라이선스)
