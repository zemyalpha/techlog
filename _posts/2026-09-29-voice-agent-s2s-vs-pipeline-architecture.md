---
title: "음성 AI 에이전트의 두 갈래 길: STT-LLM-TTS 파이프라인 vs Speech-to-Speech 아키텍처"
date: "2026-09-29"
keywords: ["음성 AI 에이전트", "speech to speech", "Realtime API", "STT LLM TTS 파이프라인", "voice agent 아키텍처"]
lang: "ko"
description: "2026년 음성 AI 에이전트는 캐스케이드 파이프라인(STT→LLM→TTS)과 단일 모델 Speech-to-Speech로 양분되었다. 지연 시간, 바지인, 운율, 비용 관점에서 두 아키텍처의 실전 차이를 코드와 함께 정리한다."
---

# 음성 AI 에이전트의 두 갈래 길: STT-LLM-TTS 파이프라인 vs Speech-to-Speech 아키텍처

전화를 받는 AI 상담원, 매장 예약을 도와주는 음성 비서, 실시간 통역 기능까지. 2026년 현재 프로덕션에서 돌아가는 음성 AI 에이전트는 거의 예외 없이 두 가지 아키텍처 중 하나다. 하나는 음성인식(STT) → LLM → 음성합성(TTS)을 순차적으로 연결하는 **캐스케이드 파이프라인**, 다른 하나는 오디오를 넣으면 오디오가 나오는 단일 멀티모달 모델인 **Speech-to-Speech(S2S)**다.

이 선택은 단순한 기술 취향이 아니다. 응답 지연 시간, 사용자가 말을 끊을 때(barge-in)의 반응성, 감정·억양 전달 능력, 분당 운영 비용이 모두 이 아키텍처 결정에서 파생된다. 이 글에서는 두 구조의 동작 원리, 실전 성능 차이, 그리고 각각이 유리한 상황을 정리한다.

## 1. 두 아키텍처의 구조적 차이

캐스케이드 파이프라인은 세 개의 독립 서비스를 연결한다.

```
발화자 오디오
    ↓
[STT] → 텍스트 전사
    ↓
[LLM] → 텍스트 응답
    ↓
[TTS] → 음성 응답
    ↓
발화자가 응답을 들음
```

서비스 세 개, 네트워크 홉 세 개, 그리고 지연과 오류와 정보 손실이 누적되는 지점 세 곳. 2024년 말 이전에는 프로덕션급 단일 음성 모델이 없었기 때문에 이것이 유일한 선택지였다. Deepgram·AssemblyAI·Whisper가 전사를, OpenAI·Anthropic·Google이 언어 처리를, ElevenLabs·Cartesia가 합성을 담당하고, Vapi·Retell 같은 오케스트레이션 레이어가 이를 접착하는 산업 구조가 형성됐다.

전환점은 2024년 10월 OpenAI가 DevDay에서 Realtime API를 공개한 것이었다(OpenAI 공식 발표 참조). 이후 Google이 Gemini Live API를, xAI가 오디오 지원 Grok을 출시하면서 단일 모델 기반 S2S가 프로덕션 선택지가 되었다. 2025년 8월에는 전용 모델인 gpt-realtime도 공개되었다.

```
발화자 오디오
    ↓
[멀티모달 모델] → 오디오 응답
    ↓
발화자가 응답을 들음
```

추론 한 번, 중간에 전사 텍스트 없음. 이 차이가 아래의 모든 차이를 만든다.

## 2. 지연 시간: 680ms가 의미하는 것

사람이 자연스럽다고 느끼는 대화 턴어라운드는 대략 1초 이내다. 음성 에이전트 업체 DestiLabs가 10개 이상의 실제 프로덕션 배포에서 텔레메트리를 수집해 공개한 2026년 벤치마크에 따르면, 전체 함대의 중간값은 p50 680ms, p95 1,180ms였다. 일관되게 1,200ms를 넘으면 발화자가 "여기 계세요?"라고 되묻거나 에이전트 말을 끊고 말하기 시작한다는 것이 이 보고서의 관찰이다(단일 업체 텔레메트리 기반이므로 참고 수치로 봐야 한다).

파이프라인에서는 각 단계가 순차 대기한다. STT가 전사를 마쳐야 LLM이 시작되고, LLM의 텍스트가 나와야 TTS가 합성한다. 여기에 각 서비스 간 네트워크 왕복이 더해진다. 반면 S2S는 추론 한 번으로 끝난다. Leadlock 같은 S2S 기반 업체는 Grok 기반으로 라이브 턴 461개의 중간값 약 300ms 응답을 주장하지만(역시 벤더 자체 측정), 방향성은 명확하다. 같은 지연 예산 안에서 파이프라인은 세 단계에 나눠 써야 하고 S2S는 한 번에 쓸 수 있다.

특히 중요한 것은 중간값이 아니라 꼬리(p95)다. 650ms p50에 2,500ms p95인 시스템이 800ms의 균일한 응답보다 체감상 더 나쁘다. 예측 불가능한 긴 정적이 대화 리듬을 깨기 때문. DestiLabs 보고서는 테일의 주범으로 동기 도구 호출(느린 CRM 조회), 콜드스타트 LLM 요청, 긴 응답의 TTS를 꼽았고, 최고 효율 개선책은 스트리밍 TTS로 첫 문장부터 재생을 시작하는 것이라고 정리했다.

## 3. 바지인과 운율: 파이프라인이 구조적으로 못하는 것

**바지인(barge-in)** — 사용자가 에이전트 말 중간에 끼어드는 것 — 은 파이프라인에서 비싸다. 현재 재생 중인 TTS 버퍼, 생성 중인 LLM, 이미 돌기 시작한 STT 세 곳의 취소를 조율하고 새 입력으로 파이프라인을 재시작해야 한다. 결과는 "잠시만요, 다시 말씀해 주시겠어요?"라는 어색한 복구. S2S에서는 응답을 생성하는 모델이 동시에 입력 오디오를 받고 있으므로, 끼어듦을 감지하고 즉시 멈추고 반응한다. 취소 캐스케이드 자체가 없다.

**운율(prosody)과 감정**은 더 근본적인 차이다. STT가 오디오를 텍스트로 바꾸는 순간, 억양·말속도·망설임·역조롱 같은 준언어적 신호는 전부 사라진다. LLM이 보는 것은 "네 관심 있어요"라는 문자열뿐이고, 진심인지 건성인지 구분할 수 없다. 파이프라인은 감정 분석을 덧붙여 보완하지만 이미 손실된 정보의 근사치일 뿐이다. S2S 모델은 오디오 토큰을 직접 처리하므로 발화자의 톤을 응답 생성에 반영할 수 있고, 자신의 응답도 텍스트가 아니라 오디오로 생성하기 때문에 운율이 담긴다.

## 4. 실전 구현: 두 아키텍처의 코드 모습

### 파이프라인 방식 (Python 의사코드)

```python
import asyncio

async def pipeline_turn(audio_chunk):
    # 1단계: STT — 전사가 끝나야 다음 단계
    text = await stt_client.transcribe(audio_chunk)

    # 2단계: LLM — 텍스트 응답 생성 (도구 호출 포함 가능)
    reply = await llm_client.chat(
        messages=history + [{"role": "user", "content": text}],
        tools=[calendar_tool, crm_tool],
    )

    # 3단계: TTS — 첫 문장부터 스트리밍 재생이 테일 지연 완화의 핵심
    async for sentence in split_sentences(reply):
        audio = await tts_client.synthesize(sentence)
        await play_stream(audio)  # barge-in 시 여기서 취소 필요
```

각 단계가 독립 서비스라 모델 교체가 자유롭고, 도구 호출·RAG·가드레일을 LLM 단계에 표준적인 방식으로 끼워 넣을 수 있다.

### S2S 방식 (OpenAI Realtime API)

```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async with client.realtime.connect(
    model="gpt-realtime",
) as session:
    await session.session.update(session={
        "voice": "alloy",
        "input_audio_transcription": {"model": "whisper-1"},
        "turn_detection": {"type": "server_vad"},  # 서버 측 VAD로 barge-in 처리
    })

    async def stream_mic():
        async for chunk in mic_stream():          # 마이크 → WebSocket
            await session.input_audio_buffer.append(chunk)

    async def play_response():
        async for event in session:               # 오디오 토큰 수신 즉시 재생
            if event.type == "response.audio.delta":
                await speaker.play(event.delta)

    await asyncio.gather(stream_mic(), play_response())
```

하나의 세션에서 입력 오디오와 출력 오디오가 같은 모델을 통과한다. 전사·생성·합성의 경계가 없고, 서버 측 VAD(voice activity detection)가 발화 시작과 끝을 판정한다.

## 5. 파이프라인이 여전히 이기는 지점

S2S가 모든 면에서 우월한 것은 아니다. 2026년 현재 파이프라인이 유리한 조건은 명확하다.

- **대규모 트래픽의 비용**: DestiLabs 벤치마크에서도 캐스케이드 스택이 규모의 경제에서 유리하다고 정리했다. STT와 TTS는 경쟁이 치열해 단가가 낮고, 저렴한 모델로 조합을 최적화할 수 있다.
- **제어력과 디버깅**: 각 단계의 입출력이 텍스트로 관측되므로 실패 지점을 특정하기 쉽다. S2S는 중간 표현이 없어 무엇이 잘못됐는지 들여다보기 어렵다.
- **도구 생태계**: 함수 호출, RAG, 구조화된 출력 등 LLM 생태계의 성숙한 도구를 그대로 쓸 수 있다.
- **로컬·오픈소스 대안**: GLM-4-Voice(9B, zai-org 공개)처럼 오픈소스 E2E 음성 모델도 존재하지만, 운영 난이도를 고려하면 Whisper + 로컬 LLM + 오픈소스 TTS 조합이 여전히 현실적인 자가호스팅 경로다.

즉, **대화의 자연스러움이 제품의 핵심이면 S2S, 비용·제어·안정성이 우선이면 파이프라인**이라는 것이 2026년의 실무적 결론이다.

## 결론

- 음성 에이전트의 아키텍처는 캐스케이드 파이프라인(STT→LLM→TTS)과 단일 모델 S2S로 양분되어 있으며, 이 결정이 지연·바지인·운율·비용을 좌우한다.
- S2S의 전환점은 2024년 10월 OpenAI Realtime API였고, 2025년 8월 gpt-realtime, Google의 Gemini Live API가 뒤를 이었다.
- 실무 기준선으로 p95 1,400ms 이내가 권장되며, 지연 테일의 최대 개선책은 스트리밍 TTS로 첫 문장부터 재생하는 것이다.
- 파이프라인은 비용·제어력·도구 생태계에서, S2S는 대화 자연스러움에서 각각 우위다. 트래픽 규모와 제품 요구에 따라 선택이 갈린다.

첫 단계로 권하는 것은 기존 파이프라인 에이전트가 있다면 지연 시간을 p50/p95로 측정해 보는 것이다. 꼬리가 길다면 스트리밍 TTS 도입부터, 대화 몰입이 제품 차별점이라면 Realtime API 소규모 스파이크를 검토할 가치가 있다.

## 참고 자료

- [Introducing the Realtime API — OpenAI](https://openai.com/index/introducing-the-realtime-api/)
- [Introducing gpt-realtime and Realtime API updates — OpenAI](https://openai.com/index/introducing-gpt-realtime/)
- [Gemini Live API overview — Google AI for Developers](https://ai.google.dev/gemini-api/docs/live-api)
- [GLM-4-Voice — zai-org (GitHub)](https://github.com/zai-org/GLM-4-Voice)
- [2026 AI Voice Agent Benchmark: Latency & Cost per Minute — DestiLabs](https://www.destilabs.com/blog/ai-voice-agent-benchmark-2026)
- [Speech-to-Speech vs Pipeline Voice Agents — Leadlock AI](https://www.leadlock.ai/blog/speech-to-speech-vs-pipeline-voice-agents/)
