---
title: "llama.cpp v1.0.0 출시 — 3.5년 만의 1.0 선언, FP8 KV 캐시와 MoE 캐시로 바뀐 것들"
date: "2026-11-14"
keywords: ["llama.cpp", "ggml", "FP8 KV 캐시", "MoE 캐시", "로컬 LLM 추론", "semantic versioning"]
lang: "ko"
description: "llama.cpp가 2026년 11월 13일 v1.0.0을 출시했다. API 안정화 선언의 의미, FP8 KV 캐시와 MoE 호스트 메모리 캐시, 서버 보안 강화까지 실전 관점에서 정리한다."
---

# llama.cpp v1.0.0 출시 — 3.5년 만의 1.0 선언, FP8 KV 캐시와 MoE 캐시로 바뀐 것들

2026년 11월 13일, llama.cpp가 v1.0.0을 출시했다. 2023년 3월 10일 Georgi Gerganov가 저장소를 만든 지 약 3년 반 만의 일이며, 텐서 라이브러리인 ggml도 함께 v1.0.0을 찍었다. 현재 GitHub 스타 약 13.5만 개(2026-11-14 기준 134,909개), MIT 라이선스의 프로젝트가 드디어 "1.0"을 선언한 것이다.

이 릴리스의 핵심은 화려한 신기능이 아니라 **API 안정성 약속**이다. llama.cpp를 라이브러리로 쓰는 수많은 다운스트림 프로젝트(Ollama, LM Studio류 앱, 각종 임베디드 포팅)에게 "이제 인터페이스가 흔들리지 않는다"고 공식화한 셈이다. 이 글에서는 v1.0.0에서 실제로 바뀐 것들을 릴리스 노트 기반으로 짚고, 로컬 추론 운영자 관점에서 당장 써먹을 수 있는 부분을 정리한다.

## 1.0의 의미: 빌드 번호에서 시맨틱 버저닝으로

지금까지 llama.cpp의 릴리스는 `b12375` 같은 빌드 번호 체계였다. 사실상 매일 올라오는 스냅샷일 뿐, 버전 번호로 하위 호환성을 판단할 수 없었다. v1.0.0부터는 시맨틱 버저닝 규칙이 공식화된다. 공식 발표 블로그(Hugging Face, 2026-11-13)에 따르면:

- **MAJOR**: `./include`의 공개 C API 파괴적 변경
- **MINOR**: 하위 호환되는 기능 추가, 백엔드 업데이트, 모델 지원
- **PATCH**: API 변경 없는 버그 수정

즉 앞으로 `2.0.0`이 나오면 공개 C API가 깨진다는 뜻이고, `1.x`에 머무는 한 라이브러리 통합이 안전하다. 배포 채널도 정리됐다. Homebrew(macOS/Linux), Docker, Debian, conda-forge, MacPorts, Winget(Windows), iOS/macOS용 XCFramework까지 공식 채널로 명시됐다. macOS 사용자는 이제 다음과 같이 설치하면 된다:

```bash
brew install llama.cpp
```

참고로 일일 빌드는 계속된다. v1.0.0 다음 날인 11월 14일에도 `b12375` 빌드가 올라왔다. 1.0은 개발 속도를 늦추는 게 아니라 버전 규칙을 붙인 것이다.

## API 변경: 35개 지원 중단 함수가 실제로 삭제됨

v1.0.0에서 가장 직접적인 영향은 **지원 중단(deprecated)으로 표시돼 있던 공개 API 35개와 `llama_set_warmup`, `defrag_thold`가 삭제**된 것이다 (PR #30565). llama.cpp를 소스로 링크해 쓰는 프로젝트라면 빌드가 깨질 수 있으므로 마이그레이션이 필요하다.

주요 변경 두 가지:

- NextN/스페큘러티브 디코딩 API가 `llama-ext.h`에서 `llama.h`로 승격되고, `llama_set_embd_state_src`가 `llama_set_ctx_state_src`로 이름이 바뀌었다 (PR #30565)
- `llama_context_params`에 `moe_cache_size`와 `moe_cache_cpu_share`가 추가됐다 (PR #30966)

또한 비전 전용 헤더였던 `clip.h`가 제거되고 멀티모달 처리가 `mtmd`로 통합됐다 (PR #31865). 비전 파이프라인을 직접 붙였던 프로젝트는 이 부분도 확인 대상이다.

## 추론 성능: FP8 KV 캐시와 MoE 호스트 메모리 캐시

긴 컨텍스트 처리에서 KV 캐시 메모리는 항상 병목이었다. v1.0.0은 **OCP FP8(e4m3) KV 캐시**를 추가했고 (PR #28392), 옵션으로 텐서별 정적 스케일도 지원한다. 사용법은 간단하다:

```bash
llama-server \
  -m model-Q4_K_M.gguf \
  -c 131072 \
  -ctk f8_e4m3 -ctv f8_e4m3 \
  --port 8080 --host 127.0.0.1
```

`-ctk/-ctv`에 `f8_e4m3`를 주면 KV 캐시가 FP16 대비 정확히 절반 수준(1바이트 vs 2바이트)으로 줄어든다. 128K 컨텍스트를 잡는 순간 수십 GB로 불어나던 KV 캐시를, 품질 저하를 감수할 만한 워크로드(배치 요약, 코드 인덱싱 등)에서 훨씬 작은 메모리로 돌릴 수 있게 된 셈이다.

MoE 모델용 **호스트 메모리 전문가(expert) 캐시**도 눈에 띈다 (PR #30966, #31233, #31888). 전문가 가중치 전부를 VRAM에 올리지 못하는 상황에서, 자주 쓰이는 전문가를 호스트 메모리에 캐시하고 미스(miss) 발생 시 CPU가 처리할 비율을 `moe_cache_cpu_share`로 조절한다. 교체 정책은 LRFU/ClassLRU이며, 캐시에 이미 있는 전문가는 CPU를 기다리지 않는다고 릴리스 노트는 밝힌다. "MoE 모델을 로컬 GPU에서 돌리면 전문가 로드 때문에 토큰이 뚝뚝 끊긴다"는 경험을 겪어본 사람이라면 이 변경의 무게를 알 것이다.

## 서버와 생태계: 보안 강화, TTS 엔드포인트, 새 아키텍처

`llama-server`는 이번에 보안 대대 정비를 거쳤다. CORS 프록시 SSRF, DNS 리바인딩, `--cors-origins` 처리, 차일드 인스턴스 접근 등이 수정됐고 (PR #32098, #32115, #32120), MCP 도구 호출은 클라이언트 연결 끊김·타임아웃 시 취소되고 MCP 서버에 통보되도록 바뀌었다 (PR #32025). 도구를 무인증으로 열려면 이제 `--allow-unauthenticated-tools`를 명시적으로 켜야 한다 (PR #32113). `llama-server`를 로컬 네트워크에 노출하고 MCP 도구를 물려둔 구성이라면 v1.0.0으로 올리면서 플래그를 점검해야 한다.

기능 추가도 있다:

- OpenAI 호환 `/v1/audio/speech` TTS 엔드포인트 (PR #31763)와 drama-3 TTS 지원 (PR #31613). 로컬 전용 TTS 스택을 llama-server 하나로 묶을 수 있게 됐다:

```bash
curl http://localhost:8080/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model": "drama-3", "input": "안녕하세요, llama.cpp 1.0입니다."}' \
  --output speech.wav
```

- 새 텍스트 아키텍처 3종: Kimi-K3.1-Pro(MLA 전체 어텐션 층을 DSA로 교체), 신규 dense 65B 구조인 Ripple-65B-Preview, mHC+MLA+MoE+MTP 조합의 xing4_0
- `clip.h` 통합 외에도 서브컨텍스트 간 컴퓨트 버퍼 공유로 비전+오디오+생성 통합 모델의 메모리 절약 (PR #31945)
- 웹 UI 전면 개편: 채팅 그룹, 마크다운 에디터, 모바일 네이티브 레이아웃

모델 커버리지 측면에서 llama.cpp는 공식 블로그 기준 150개 이상의 아키텍처 변형과 주류 스페큘러티브 디코딩 방식을 지원하며, NVIDIA·AMD·Qualcomm·Arm·IBM·Intel의 하드웨어 지원을 받아 셀프호스티드 러너를 운영 중이다. ggml이 "사실상 모든 하드웨어"를 겨냥한다는 프로젝트의 원래 목표는 1.0에 와서도 변하지 않았다.

## 마이그레이션 체크리스트

정리하면, 지금 점검할 것은 다음과 같다:

- **소스 링크 프로젝트**: 삭제된 35개 지원 중단 함수 사용 여부 확인. NextN 사용 코드는 `llama_set_embd_state_src` → `llama_set_ctx_state_src` 이름 변경 반영
- **비전 파이프라인**: `clip.h` 의존 제거, `mtmd` 통합 API로 이전
- **`llama-server` 운영자**: 보안 기본값 강화로 인해 MCP 도구·미디어 프록시 동작이 달라졌는지 플래그 점검 (`--allow-unauthenticated-tools`, `--media-allow-private-urls`)
- **긴 컨텍스트 워크로드**: `-ctk/-ctv f8_e4m3`로 KV 캐시 메모리 절감 실험
- **MoE 로컬 실행**: 새 MoE 캐시 파라미터로 전문가 스래싱 완화 여부 테스트
- **패키지 관리자**: Homebrew·Docker·conda-forge 등 공식 채널 설치가 이제 문서화된 지원 경로. 시맨틱 버전으로 고정 가능

3년 반 동안 "빌드 번호"로만 존재하던 프로젝트가 1.0을 선언했다는 것은, 로컬 AI 인퍼런스가 실험 단계를 지나 인프라의 자리에 앉았다는 신호로 읽힌다. API가 안정적으로 잠기면 그 위에 올라가는 것이 늘어난다. 다운스트림 생태계의 반응이 다음 관전 포인트다.

**참고 자료:**
- 공식 발표: [llama.cpp v1.0.0: A stronger foundation for the local AI ecosystem (Hugging Face 블로그, 2026-11-13)](https://huggingface.co/blog/ggml-org/llama-cpp-v1)
- 릴리스 노트: [GitHub ggml-org/llama.cpp v1.0.0](https://github.com/ggml-org/llama.cpp/releases/tag/v1.0.0)
- 저장소: [github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)
