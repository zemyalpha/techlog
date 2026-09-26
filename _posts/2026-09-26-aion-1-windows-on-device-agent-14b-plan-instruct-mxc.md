---
title: "윈도우에 14B 에이전트 모델이 들어온다 — Aion 1.0 Plan·Instruct와 MXC가 만드는 로컬 에이전트 스택의 실체"
date: "2026-09-26"
keywords: ["Aion 1.0", "온디바이스 SLM", "Windows AI", "로컬 에이전트", "MXC", "NPU", "Copilot+ PC"]
lang: "ko"
description: "마이크로소프트가 Build 2026에서 발표한 Aion 1.0 Plan(14B)·Instruct와 MXC 격리 계층을 분석한다. 하드웨어 요건, 실제 실행 방법, Ollama와의 차이까지."
---

# 윈도우에 14B 에이전트 모델이 들어온다 — Aion 1.0 Plan·Instruct와 MXC가 만드는 로컬 에이전트 스택의 실체

2026년 6월 2일, 마이크로소프트는 Build 2026에서 "윈도우에 내장되는 온디바이스 언어 모델"을 발표했다. Aion 1.0이다. 핵심은 두 가지 모델이다. **Aion 1.0 Instruct**는 요약·재작성 같은 일상 텍스트 작업용 소형 SLM이고, **Aion 1.0 Plan**은 파라미터 14B, 컨텍스트 32K의 추론·도구 호출 모델로, 파일 관리와 서브 에이전트 오케스트레이션까지 "완전 로컬 에이전트 루프"를 지향한다.

이 글은 공식 발표와 실제 배포 상태를 대조하면서, 개발자가 지금 당장 무엇을 할 수 있고 무엇을 기다려야 하는지를 정리한다. 결론부터 말하면 — Aion의 의미는 "빠른 로컬 챗봇"이 아니라 **OS·브라우저·격리 계층·엔터프라이즈 정책이 하나의 로컬 에이전트 스택으로 묶였다**는 점이다. 동시에, 현재 프리뷰는 ARM64 스냅드래곤 Copilot+ PC에서만 돌아가는 등 하드웨어 현실이 만만치 않다.

## 1. Aion 1.0 두 모델: Instruct와 Plan은 역할이 다르다

공식 Windows 개발자 블로그의 발표 문구를 그대로 옮기면, Aion 1.0 Instruct는 "더 작고, 빠르고, 똑똑한 온디바이스 SLM"이고, Aion 1.0 Plan은 "완전 로컬 에이전트 기능을 가능하게 하는 추론·도구 호출 모델"이다. 두 모델의 역할 분담은 명확하다.

**Aion 1.0 Instruct — 일상 텍스트 처리용**

- 요약, 재작성, 인텐트 감지, 접근성 작업용
- 이전 세대인 Edge의 Phi-4-mini(4B)를 대체한다. Edge 블로그는 2025년 Build에서 Phi-4-mini 기반 Prompt API·Writing Assistance API를 도입했었다고 밝히며, 이번에 Aion-1.0-Instruct의 개발자 프리뷰를 시작했다
- 공개 경로가 비교적 명확하다. 발표에 따르면 **2026년 7월 허깅페이스에 오픈소스로 공개될 예정**이다
- Edge 148에는 별개의 온디바이스 태스크 모델인 Language Detector API와 Translator API(145개 이상 언어 지원)도 포함됐다

**Aion 1.0 Plan — 로컬 에이전트의 두뇌**

- 14B 파라미터, 32K 컨텍스트
- 사용자 의도 추론, 도구 호출, 파일 관리, 서브 에이전트 오케스트레이션이 발표 범위에 명시돼 있다
- "역량 있는(capable) 윈도우 기기에 인박스로 제공"된다고 발표됐으며, 출시 시점은 "향후 몇 달 내"로만 공표된 상태다
- 중요한 미확정 지점: 발표 시점에 Plan의 모델 카드, 정확한 최소 하드웨어 요건, 양자화 포맷, 지연 시간 수치, 라이선스는 공개되지 않았다. Instruct의 오픈웨이트 공개 계획이 명시된 것과 대비된다

두 모델은 경쟁사 로컬 모델과 배포 계층에서 차이가 난다. Google Gemma 4나 QVAC TurboQuant는 개발자가 허깅페이스나 SDK로 "가져오는" 모델이다. Aion은 **윈도우라는 OS 설치 기반이 배포 채널이 되는 모델**이다. 이 차이가 이번 발표의 본질이다.

## 2. 지금 당장 실행해 보는 방법 — 그리고 그 조건

Aion 1.0 Plan은 아직 출시 전이지만, Aion Instruct Preview는 이미 공식 GitHub 저장소([microsoft/Aion-Instruct-Preview-Sample](https://github.com/microsoft/Aion-Instruct-Preview-Sample))로 공개돼 있다. README를 직접 확인한 결과, 실행 조건이 상당히 제한적이다.

**하드웨어 조건 (2026년 9월 기준):**

- ARM64 Copilot+ PC, 즉 스냅드래곤 기기에서만 실행된다. QNN 실행 프로바이더로 NPU를 직접 쓴다
- **CPU 폴백이 없다.** 인증된 NPU 실행 프로바이더가 필수다
- x64(인텔/AMD) 지원은 "coming soon" 상태다

**실행 절차:**

```powershell
git clone https://github.com/microsoft/Aion-Instruct-Preview-Sample
cd Aion-Instruct-Preview-Sample
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\Bootstrap.ps1
```

Bootstrap.ps1은 개발자 모드 활성화, 프리뷰 프레임워크 MSIX 설치, ARM64에서 QNN 실행 프로바이더 준비, 빌드와 실행까지 한 번에 처리한다. 첫 실행 시 NPU용으로 모델을 컴파일하는 데 3~5분이 걸리고, 이후 프롬프트는 첫 토큰까지 1초 미만이라고 README에 명시돼 있다. 샘플 앱은 WinUI 3 챗 앱으로, 토큰이 스트리밍되며 초당 토큰 수와 첫 토큰 지연시간이 표시된다.

웹 개발자라면 Edge 쪽을 먼저 볼 수 있다. Edge Canary/Dev 채널에서 온디바이스 음성 인식을 실험 중이며, 기존 Web Speech API 인터페이스에 `processLocally` 플래그를 추가하는 방식이다:

```javascript
const recognition = new SpeechRecognition();
recognition.lang = 'en-US';
recognition.processLocally = true;
recognition.start();
```

네트워크 왕복 없이 브라우저가 음성 인식을 처리하는 경로다. 다만 아직 프리뷰·채널 한정 기능이므로, 프로덕션 웹앱에서는 미지원 브라우저 폴백과 권한 UX를 반드시 함께 설계해야 한다.

## 3. 하드웨어의 벽: 40 TOPS와 메모리 대역폭

여기서부터가 각종 호화 보도가 생략하는 부분이다. Aion의 NPU 경로는 Copilot+ PC 요건을 상속받는데, 이는 **NPU 40 TOPS**가 최소선이다. 이 숫자 하나로 기존 노트북의 접근 가능 여부가 갈린다.

- 스냅드래곤 X 엘리트(45 TOPS), 인텔 Lunar Lake(45~48 TOPS)는 요건을 충족한다
- 인텔 메테오 레이크(1세대 Core Ultra)는 NPU가 약 10 TOPS에 불과하다(칩 분석 사이트 Chips and Cheese의 NPU 3720 계산치는 9.5 TOPS). 스펙시트의 "34 TOPS"는 NPU+iGPU+CPU를 합산한 패키지 수치다. 즉 2024년형 "AI PC" 상당수는 NPU 기준 미달이다
- AMD 라이젠 AI 300/스트릭스 할로의 XDNA 2 NPU는 50 TOPS로 충분하지만, 마이크로소프트는 론칭 시점에 AMD 지원을 "coming later"로 연기했다. 실리콘은 되고 소프트웨어 검증이 늦은 상태다

더 근본적인 문제는 TOPS가 토큰 생성 속도를 보장하지 않는다는 점이다. LLM 디코드(토큰을 하나씩 뽑아내는 단계)는 **연산량이 아니라 메모리 대역폭**에 병목이 걸린다. TOPS는 프리필(프롬프트 처리)에는 유리하지만, 디코드 속도는 매 토큰마다 모델 가중치를 메모리에서 꺼내는 속도에 의해 결정된다.

RunAIHome의 독립 측정 기준으로는 스냅드래곤 X 엘리트 NPU에서 Llama 3.1 8B가 대략 초당 5 토큰 수준이고, 반면 중고 RTX 3090(936GB/s 대역폭)은 7B 모델을 초당 95 토큰 가까이 처리한다. 14B 추론 모델인 Plan이 응답 전에 800토큰의 사고 과정을 뽑는다면 NPU에서는 1분 이상 기다리게 되는 셈이다. 사람이 편안하게 읽는 속도가 초당 7~10 토큰임을 고려하면, NPU 기반 14B 추론 모델은 실시간 읽기 속도보다 느리다. (이 수치들은 단일 소스 측정값이므로 참고치로 봐야 한다.)

그렇다면 NPU의 승부처는 어디인가. **와트당 토큰**이다. NPU는 10~25W로 이 작업을 수행하고, RTX 3090은 시스템 전체로 285~350W를 소모한다. 파일을 요약하고 답장 초안을 쓰는 상시 대기 에이전트에는 5 tok/s도 충분히 유용하지만, 사용자가 결과를 기다리는 인터랙티브 작업에는 NPU가 적합하지 않다. "unmetered intelligence(측정되지 않는 지능)"라는 마이크로소프트의 표현은 무료를 뜻하지 않는다 — 토큰 과금이 사라지는 대신 비용이 기기 메모리·배터리·발열·지원 부담으로 이동할 뿐이다.

## 4. MXC: 로컬 에이전트는 모델만으로 안전해지지 않는다

같은 발표에서 마이크로소프트는 **Microsoft Execution Containers(MXC)** SDK를 얼리 프리뷰로 공개했다(공식 저장소: [github.com/microsoft/mxc](https://github.com/microsoft/mxc)). MXC는 개발자가 에이전트의 파일·네트워크 접근 범위를 선언하면 런타임이 그 경계를 강제하는, 정책 기반 실행 계층이다. 프로세스 격리와 세션 격리로 에이전트 실행을 사용자의 데스크톱·클립보드·입력 장치와 분리하고, 강한 사용자 신원에 실행을 바인딩한다.

이유는 명확하다. 에이전트가 파일을 읽고 셸 명령을 실행하는 순간, 공격 표면은 클라우드 API 호출이 아니라 **사용자의 물리적 기기**로 옮겨 온다. 로컬 추론이 프라이버시를 개선할지언정, 권한 설계 없이는 보안을 개선하지 못한다.

주목할 점은 파트너 인용의 방향이다. Nous Research와 OpenAI 모두 "더 똑똑한 모델"이 아니라 "통제된 실행 환경" 쪽으로 초점을 옮기는 발언을 냈다. 에이전트가 코드를 실행하는 시대의 제품 표면은 모델 API만큼이나 OS 정책이라는 신호다. 함께 발표된 Windows 365 for Agents(에이전트에 관리형 클라우드 PC를 주는 서비스)의 일반 가용 전환도 같은 맥락이다.

## 5. Ollama·dGPU 조합과 비교: Aion을 언제 쓸 것인가

"이미 Ollama로 Qwen 14B를 돌리는데 왜 Aion인가"라는 질문이 당연히 나온다. 실용적인 답은 세 가지 트레이드오프로 정리된다.

**배포와 설정.** Aion은 인박스로 제공된다. 설치도, 모델 다운로드도, 양자화 선택도 없다. 터미널을 열지 않을 95%의 윈도우 사용자에게는 이것이 결정적 차이다. 단, 인박스 모델이 모든 기기에 자동 다운로드되는 것은 아니고 앱이 요청할 때 획득된다는 점도 공지돼 있다 — 디스크와 기업 fleet 정책에 유의해야 한다.

**실행 백엔드.** Ollama의 윈도우 경로는 Vulkan/CUDA를 타므로 Copilot+ 노트북에서는 주로 iGPU를 쓴다. Aion은 Windows ML 라우팅으로 NPU·iGPU·dGPU 중 선택된다. 흥미롭게도 같은 기기에서 디코드 속도만 보면 iGPU 경로가 NPU보다 빠른 경우도 있다 — iGPU가 더 많은 메모리 대역폭을 쓸 수 있기 때문이다. 속도가 목표라면 NPU가 자동으로 정답이 아니다.

**통합 vs 제어.** Aion Plan은 파일 접근·도구 호출·서브 에이전트 오케스트레이션, 그리고 MXC 격리라는 OS 수준의 에이전트 스택에 묶여 있다. Ollama는 OS 통합은 없지만 모델·양자화·데이터에 대한 완전한 통제를 준다.

당장의 실용적 설계는 **계층형 에이전트**다. 저위험·고빈도 작업(요약, 재작성, 번역, 음성인식, 분류, 인텐트 감지)은 로컬 모델에 맡기고, 고위험 추론·코드 수정·장기 계획은 프론티어 클라우드 모델에 맡긴다. Aion은 이 아키텍처의 로컬 슬롯을 늘려주는 발표이지, 클라우드 모델을 없애는 발표가 아니다.

## 결론

- Aion 1.0은 "또 하나의 로컬 모델"이 아니라 OS·브라우저·하드웨어·엔터프라이즈 정책이 묶인 로컬 에이전트 플랫폼의 첫 조각이다. 윈도우 설치 기반 자체가 모델 배포 채널이 됐다
- Plan(14B·32K·도구 호출)은 발표만 된 상태다. 모델 카드·하드웨어 요건·라이선스가 공개되기 전까지 "역량 있는 기기"는 미확정 약속이다. Instruct는 허깅페이스 오픈웨이트 공개(7월 예정)가 공표됐다
- 지금 실행 가능한 것은 ARM64 스냅드래곤 한정의 Instruct Preview(GitHub 샘플)와 Edge 148의 Translator/Language Detector API다. CPU 폴백이 없다는 점을 반드시 기억하자
- NPU는 속도가 아니라 와트당 토큰의 승부다. 인터랙티브 작업은 여전히 dGPU가 압도적이며, TOPS 수치는 디코드 속도를 예측하지 못한다
- 로컬 에이전트의 보안은 모델이 아니라 권한 설계에서 나온다. MXC 같은 격리 계층을 선언·테스트·감사하는 것이 에이전트 제품의 필수 공정이 된다

당장 취할 첫 단계가 있다면 이것이다 — 스냅드래곤 기기가 있다면 GitHub 샘플을 돌려 첫 토큰 지연과 tok/s를 직접 측정해 보고, 없다면 Edge 148의 Translator API로 온디바이스 경로의 폴백 설계를 연습해 보라. 로컬 에이전트 시대의 첫 조각은 이미 설치돼 있다.

**참고 자료:**
- [Build 2026: Furthering Windows as the trusted platform for development — Windows Developer Blog](https://blogs.windows.com/windowsdeveloper/2026/06/02/build-2026-furthering-windows-as-the-trusted-platform-for-development/)
- [Expanding on-device AI in Microsoft Edge — Microsoft Edge Blog](https://blogs.windows.com/msedgedev/2026/06/02/expanding-on-device-ai-in-microsoft-edge-new-models-and-apis-for-the-web/)
- [microsoft/Aion-Instruct-Preview-Sample — GitHub](https://github.com/microsoft/Aion-Instruct-Preview-Sample)
- [microsoft/mxc — GitHub](https://github.com/microsoft/mxc)
