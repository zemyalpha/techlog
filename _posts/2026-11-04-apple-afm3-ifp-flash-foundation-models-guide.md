---
title: "아이폰 안에 200억 파라미터를 넣는 법 — Apple AFM 3의 플래시 기반 sparse 아키텍처와 Foundation Models 프레임워크 실전 가이드"
date: "2026-11-04"
keywords: ["Apple AFM 3", "Foundation Models 프레임워크", "온디바이스 AI", "Instruction-Following Pruning", "Private Cloud Compute", "Swift"]
lang: "ko"
description: "WWDC26에서 발표된 Apple 파운데이션 모델 3세대의 200억 파라미터 온디바이스 모델이 플래시 메모리와 sparse 라우팅으로 동작하는 원리와, Swift Foundation Models 프레임워크로 무료 온디바이스 LLM을 앱에 넣는 실전 방법을 정리했다."
---

# 아이폰 안에 200억 파라미터를 넣는 법 — Apple AFM 3의 플래시 기반 sparse 아키텍처와 Foundation Models 프레임워크 실전 가이드

2026년 6월 8일 WWDC26에서 Apple은 파운데이션 모델 3세대(AFM 3)를 발표했다. 이 발표에서 가장 눈에 띄는 것은 화려한 Siri도, Google과의 협업도 아니다. **아이폰 같은 소비자 하드웨어의 DRAM 용량을 넘어서는 200억 파라미터 모델을 온디바이스로 돌리겠다는 시도**다. 일반적으로 온디바이스 LLM은 파라미터 수가 한 자릿수(수십억)에 머물렀다는 점을 생각하면 방향 자체가 다르다.

이 글은 두 가지를 다룬다. 먼저 AFM 3 Core Advanced가 어떤 기술로 200억 파라미터를 아이폰에 들여놓는지(공식 기술 블로그 기준), 그리고 iOS 앱 개발자가 Foundation Models 프레임워크로 이 모델을 실제로 어떻게 쓰는지다. API 키도, 네트워크 왕복도, 토큰 과금도 없다.

## AFM 3 라인업: 5개 모델의 분업

Apple Machine Learning Research의 공식 글에 따르면 3세대는 Google과 협업해 제작된 5개 모델로 구성된다.

- **AFM 3 Core** — 30억 파라미터 dense 모델. 전 세대(2025년 발표)의 직계 후속으로 품질이 한 단계 올랐다.
- **AFM 3 Core Advanced** — 온디바이스 최상위 모델. 200억 파라미터지만 sparse 구조로 요청에 따라 10~40억 파라미터만 활성화한다. 네이티브 멀티모달이라 표정 있는 음성, 고정밀 받아쓰기 같은 기능을 담당하며, 가장 성능이 좋은 Apple 실리콘에서만 잠금 해제된다.
- **AFM 3 Cloud** — Private Cloud Compute(PCC)에서 도는 서버 측 주력 모델. 속도와 효율에 최적화됐고, 전 세대에서 도입한 PT-MoE(Parallel-Track Mixture-of-Experts) 구조를 개선했다.
- **ADM 3 Cloud (Image)** — 이미지 생성·편집 전용 디퓨전 모델. 사진 편집 도구와 새로워진 Image Playground를 뒷받침한다.
- **AFM 3 Cloud Pro** — 가장 강력한 서버 모델로 에이전틱 도구 사용과 복잡한 추론을 담당한다. 흥미로운 점은 **NVIDIA GPU 위에서 구글 클라우드에서 돌아간다는 것**. PCC가 처음으로 외부 인프라로 확장됐으며, Apple은 공급망 공격 방지를 위해 PCC 함대에 속한 모든 Google Cloud 하드웨어의 암호학적으로 검증 가능한 append-only 원장을 유지한다고 밝혔다.

훈련 측면에서는 5개 모델이 공통 기반 사전학습을 공유한 뒤 각자의 아키텍처와 용도로 특화됐고, 최신 세대 클라우드 TPU로 사전학습을 대폭 확대한 뒤 지도 파인튜닝과 다단계 강화학습을 결합했다. 양자화 인식 훈련(Quantization Aware Training)으로 정확도 손실을 최소화하며 모델을 압축했다. 사람 평가자 대비 실험에서는 AFM 3 Core가 전 세대 대비 일반 텍스트 능력에서 45.6%의 프롬프트에서 선호되는 등 세대 간 향상을 보였다고 공식 블로그는 전한다.

## 핵심: 200억 파라미터를 어떻게 아이폰에 넣나

전통적인 LLM — dense든 MoE든 — 은 모든 가중치가 활성 메모리(DRAM)에 상주해야 한다. 200억 파라미터 모델을 4비트로 양자화해도 가중치만 수십 GB에 달하므로, 소비자 기기에서는 원천적으로 불가능하다.

Apple의 해법은 **Instruction-Following Pruning(IFP)** 이다. 이 기술은 Apple 연구진이 2025년에 공개한 논문(arXiv:2501.02086, ICML 2025 포스터)에서 나왔다. 동작 방식은 공식 설명을 정리하면 이렇다.

1. **전체 모델은 플래시 메모리(NAND)에 저장한다.** DRAM에는 상주하지 않는다.
2. NAND→DRAM 대역폭은 토큰 단위로 가중치를 스와핑하기엔 너무 느리다. 그래서 일반 MoE처럼 토큰마다 라우팅하는 대신, **프롬프트 단위로 라우팅을 결정**한다. 가벼운 dense 블록이 초기 처리 시점에 고정된 전문가(expert) 집합을 선택하고, 생성 도중 주기적으로 재선택한다.
3. 데이터 이동을 줄이기 위해 **항상 활성 상태인 공유 전문가(shared experts)** 비중을 높이고, 입력 의존적 라우팅 전문가(routed experts)만 필요할 때 DRAM으로 올린다. 대부분의 파라미터는 트랜스포머 FFN 블록에 묶인 전문가 가중치인데, 선택된 소수의 전문가를 공유 정적 가중치로 패치해 DRAM 안에 dense 모델을 완성한다.

여기서 흥미로운 파생 효과가 하나 더 있다. 활성 파라미터 수를 용도별로 미리 정할 수 있어서, **요청 난이도에 따라 로드할 가중치 규모를 탄력적으로 조절**할 수 있다. 쉬운 요청은 10억, 어려운 요청은 40억을 활성화하는 식이다. 이것이 "200억 파라미터 모델이 10~40억을 활성화한다"는 문장의 실제 의미다.

정리하면 IFP는 "모델을 작게 만드는" 기술이 아니라 "DRAM이라는 벽을 플래시와 프롬프트 단위 라우팅으로 우회하는" 기술이다. 같은 문제를 푸는 접근으로 양자화·증류가 있지만, 이들은 모델 전체를 여전히 DRAM에 올려야 한다는 제약을 벗어나지 못한다는 점에서 방향이 다르다.

## 개발자가 쓰는 쪽: Foundation Models 프레임워크

모델만 세워둔 게 아니라, 2025년부터 Swift API로 열어온 Foundation Models 프레임워크도 이번에 크게 확장됐다. TechCrunch의 키노트 보도에 따르면 이미지 입력과 서버 모델 지원이 추가됐고, 개발자가 원하는 클라우드 모델 제공자와도 통합할 수 있게 됐다.

### 온디바이스 모델: 토큰 예산부터 확인

기본 진입점은 `SystemLanguageModel`이다. 온디바이스 모델의 세션 컨텍스트 윈도우는 공식 문서 기준 **세션당 4,096 토큰**이다. 프롬프트, 인스트럭션, 도구 정의와 입출력, `@Generable` 스키마까지 전부 이 예산에서 나간다.

```swift
import FoundationModels

let model = SystemLanguageModel()
print(model.contextSize) // 공식 문서 기준 4096 (기기/OS 버전에 따라 다를 수 있어 런타임 확인 권장)

let count = try await model.tokenCount(
    for: "종이학 접는 방법을 세 문장으로 설명해줘"
)
```

참고로 WWDC26 세션과 일부 보도에서는 8K 등 다른 수치가 언급되기도 했다. 그래서인지 프레임워크는 `contextSize`와 `tokenCount(for:)`로 런타임에 실제 예산을 확인하도록 설계돼 있다. 하드코딩하지 말고 이 값으로 "온디바이스에서 처리할지, 서버로 넘길지"를 코드로 판단하는 것이 권장 패턴이다.

### 구조화 출력: @Generable

JSON 스키마를 두고 파싱 에러와 씨름하는 대신, Swift 타입 자체를 스키마로 쓴다.

```swift
@Generable
struct RecipeDraft {
    var title: String
    @Guide(description: "재료 목록. 최대 10개까지", .maximumCount(10))
    var ingredients: [String]
}

let session = LanguageModelSession()
let draft: RecipeDraft = try await session.respond(
    to: "김치찌개 레시피 초안을 만들어줘"
).content
```

`@Guide` 매크로로 최대 개수 등 제약을 걸 수 있어 토큰 예산 낭비도 줄인다.

### 에이전틱 패턴: DynamicProfile

올해 추가된 DynamicProfile은 세션의 인스트럭션과 도구를 선언적으로 묶고, **대화 이력을 보존한 채 모드가 바뀔 때 프로파일을 교체**할 수 있게 한다. 프로파일 분기마다 모델과 추론 수준을 다르게 지정할 수 있어서, 빠른 분석은 온디바이스 모델에, 개방형 브레인스토밍은 PCC의 깊은 추론에 맡기는 라우팅이 한 세션 안에서 완성된다.

```swift
struct CraftProfile: LanguageModelSession.DynamicProfile {
    var body: some DynamicProfile {
        Profile {
            Instructions {
                """
                너는 공예 전문가 조수다.
                이미지 분석 결과는 recordImageAnalysis 도구로 기록한다.
                """
            }
            RecordImageAnalysisTool()
        }
    }
}

let session = LanguageModelSession(profile: CraftProfile())
```

서버 측으로 올리고 싶으면 모델만 바꾸면 된다.

```swift
let session = LanguageModelSession(model: PrivateCloudComputeLanguageModel())
```

이미지 이해는 프롬프트에 `Attachment`를 붙이면 된다 — 전부 온디바이스에서 처리된다. 보도 기준 PCC 모델의 컨텍스트는 32K로 온디바이스보다 훨씬 넉넉하며, 사용량(캐시·추론 토큰 포함)도 응답에서 조회할 수 있다.

## 비용 구조가 바뀌었다: 200만 다운로드까지 무료 PCC

TechCrunch 보도에 따르면 Apple은 **첫 다운로드 200만 회 미만의 앱에 한해 PCC에서 도는 파운데이션 모델 사용료를 면제**했다. API 키나 계정 설정 없이 온디바이스와 같은 세션 API로 접근되므로, 인디 개발자 입장에서는 클라우드 LLM API 비용이라는 진입 장벽이 사실상 사라졌다. 커미션 감면 소상공인 프로그램(Small Business Program)의 AI 버전이라고 보면 된다.

여러 매체 보도에 따르면 프레임워크 자체를 오픈소스로 공개하는 것도 발표됐다. 애플 모델뿐 아니라 `LanguageModel` 프로토콜만 맞추면 어떤 모델이든 세션 뒤에 붙을 수 있고, Anthropic과 Google이 같은 프로토콜을 준수하는 Swift 패키지를 내놓았다고 알려져 있다. 모델 교체가 코드 수정이 아니라 패키지 의존성 변경이 되는 셈이다. 다만 이 부분은 공식 문서가 아닌 매체 보도 기반이므로 실제 적용 전 원문 확인을 권한다.

## 주의할 점

- **온디바이스 컨텍스트는 작다.** 세션당 4,096 토큰(공식 문서 기준)은 생각보다 빨리 찬다. 멀티턴 대화에서는 Xcode Instruments의 Foundation Models 템플릿으로 토큰 사용을 프로파일링하고, 찼으면 세션을 새로 만들어 필요한 상태만 이관하자.
- **파인튜닝한 커스텀 가중치는 못 올린다.** 온디바이스 런타임은 Apple이 제공하는 모델로 제한된다. 도메인 특화 모델이 필요하면 별도 추론 스택의 영역이다.
- **AFM 3 Core Advanced는 최상위 실리콘 전용.** 전 기기에서 쓸 수 있는 것은 아니다. `SystemLanguageModel`의 가용 모델 감지 API로 기기별 분기를 처리해야 한다.
- **평가 수치는 베타 기준.** Apple도 공식 글에서 "현재 개발 단계 기준 평가"라고 못박았다. 45.6% 선호율 같은 수치는 확정값이 아니라는 점을 염두에 두자.

## 결론

- AFM 3세대의 본질은 200억 파라미터 온디바이스 모델이고, 그 핵심은 전체 가중치를 NAND 플래시에 두고 프롬프트 단위로 전문가를 골라 DRAM에 올리는 IFP다.
- 서버 측에서도 PCC가 NVIDIA GPU + Google Cloud로 확장됐지만, 검증 가능한 원장과 다중 루트 오브 트러스트로 프라이버시 보장을 유지하려 한다.
- 개발자에게는 토큰 과금 없는 온디바이스 LLM, 200만 다운로드까지 무료인 PCC, 그리고 `@Generable`과 DynamicProfile이라는 실용적 API가 열렸다.
- 먼저 해볼 일은 분명하다. Xcode에서 `LanguageModelSession` 하나 만들고, `tokenCount(for:)`로 내 앱의 대표 프롬프트가 몇 토큰인지 측정하는 것부터다.

**참고 자료**
- Apple ML Research, "Introducing the Third Generation of Apple's Foundation Models" (2026-06-08)
- Apple Developer Documentation, "Foundation Models" / "Managing the context window"
- 9to5Mac, "Apple's third-generation Foundation Models explained" (2026-06-11)
- TechCrunch, "Apple bets cheaper AI will woo small developers" (2026-06-08)
- arXiv:2501.02086, "Instruction-Following Pruning for Large Language Models" (ICML 2025)
