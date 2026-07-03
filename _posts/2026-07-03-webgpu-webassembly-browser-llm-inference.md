---
title: "브라우저에서 LLM을 실행한다는 것: WebGPU와 WebAssembly로 만드는 서버리스 AI"
date: "2026-07-03"
keywords: ["WebGPU", "WebAssembly", "WebLLM", "브라우저 LLM", "클라이언트 사이드 AI"]
lang: "ko"
description: "WebGPU와 WebAssembly를 활용해 브라우저에서 직접 LLM을 실행하는 방법을 WebLLM, Transformers.js, ONNX Runtime Web의 실제 사용법과 함께 정리한다."
---

# 브라우저에서 LLM을 실행한다는 것: WebGPU와 WebAssembly로 만드는 서버리스 AI

LLM API 호출 비용은 트래픽이 늘수록 선형으로 증가한다. 수백만 건의 요청을 처리하는 서비스에서 토큰당 비용은 매월 수천만 원 규모로 자라난다. 그런데 사용자의 기기—데스크톱이든 모바일이든—에는 이미 GPU와 CPU가 있다. 이 자원을 활용해 LLM 추론을 클라이언트로 옮기면, 서버 비용은 0에 가까워지고 데이터는 기기를 떠나지 않는다.

이 글에서는 WebGPU와 WebAssembly를 활용해 브라우저 내에서 LLM을 실행하는 기술 스택을 정리한다. WebLLM, Transformers.js, ONNX Runtime Web이라는 세 가지 실용적인 도구를 비교하고, 각각의 코드를 작성하며, 어떤 시나리오에 적합한지 선택 기준을 제시한다.

## 브라우저에서 LLM이 가능해진 배경

2023년까지만 해도 브라우저에서 신경망을 실행한다는 것은 작은 분류 모델에 한정되는 이야기였다. JavaScript의 실행 속도 한계와 GPU 접근성 부족이 걸림돌이었다. 두 가지 기술이 이를 바꿨다.

**WebAssembly(Wasm)**는 C++, Rust, Python 같은 언어로 작성된 코드를 브라우저에서 네이티브에 가까운 속도로 실행할 수 있게 하는 바이너리 포맷이다. JavaScript보다 계산 집약적 작업에서 최대 수 배 빠른 성능을 낸다. SIMD(Single Instruction, Multiple Data) 명령어 지원으로 벡터 연산도 효율적으로 처리한다.

**WebGPU**는 2023년 Chrome 113에 기본 탑재되면서 브라우저에서 GPU 컴퓨팅에 직접 접근할 수 있게 한 그래픽스/컴퓨팅 API다. Vulkan, Metal, Direct3D 12 위에 구축되어 크로스 플랫폼 GPU 가속을 제공한다. WebGL이 렌더링에 초점을 맞췄다면, WebGPU는 범용 GPU 연산(Compute Shader)을 1등 시민으로 취급한다. Can I Use의 데이터에 따르면, 2026년 7월 현재 Chrome은 버전 113부터 안정 지원하고, Safari는 버전 26부터 부분 지원(partial support)을 시작했으나, Firefox는 여전히 플래그 뒤에 비활성화된 상태다.

이 둘이 결합하면서, 브라우저는 LLM 추론을 수행할 수 있는 실행 환경이 되었다. 모델 가중치를 양자화(quantization)해 메모리에 맞추면, 사용자 기기의 GPU가 실제로 추론을 수행한다. 서버는 정적 파일 호스팅만 담당한다.

## 실전 도구 3종: WebLLM, Transformers.js, ONNX Runtime Web

### WebLLM: OpenAI API 호환 브라우저 LLM 엔진

MLC-AI 팀이 개발한 WebLLM은 브라우저 내 LLM 추론에 특화된 고성능 엔진이다. CMU Catalyst, UW SAMPL, SJTU, OctoML 커뮤니티가 참여한 오픈소스 프로젝트로, WebAssembly와 WebGPU를 결합해 서버 없이 LLM을 구동한다. Llama 3, Phi 3, Gemma, Mistral, Qwen 등 주요 오픈소스 모델을 지원하며, OpenAI API와 호환되는 인터페이스를 제공한다.

핵심 특징:
- 모든 추론이 브라우저 내부에서 실행되어 데이터가 외부로 전송되지 않는다
- OpenAI API와 동일한 인터페이스(`engine.chat.completions.create`)를 사용한다
- 스트리밍, JSON 모드, WebWorker/ServiceWorker 실행을 지원한다

설치와 기본 사용법:

```bash
npm install @mlc-ai/web-llm
```

```javascript
import { CreateMLCEngine } from "@mlc-ai/web-llm";

const initProgressCallback = (progress) => {
  console.log(`로딩 중: ${Math.round(progress.progress * 100)}%`);
};

// Llama-3.1-8B를 INT4 양자화로 로드
const engine = await CreateMLCEngine(
  "Llama-3.1-8B-Instruct-q4f32_1-MLC",
  { initProgressCallback }
);

// OpenAI API와 동일한 호출 방식
const reply = await engine.chat.completions.create({
  messages: [
    { role: "system", content: "당신은 도움이 되는 AI 비서입니다." },
    { role: "user", content: "WebGPU가 뭔가요?" },
  ],
});
console.log(reply.choices[0].message.content);
```

스트리밍 응답도 지원한다:

```javascript
const chunks = await engine.chat.completions.create({
  messages: [{ role: "user", content: "React 훅을 설명해줘" }],
  stream: true,
});

for await (const chunk of chunks) {
  const delta = chunk.choices[0]?.delta?.content || "";
  process.stdout.write(delta); // 실시간 출력
}
```

WebLLM은 무거운 연산을 WebWorker로 분리할 수 있어 UI 스레드가 블록되지 않는다. ServiceWorker에 모델을 상주시키면 페이지 이동 간에도 모델을 다시 로드할 필요가 없다.

### Transformers.js: Hugging Face의 브라우저 ML 파이프라인

Hugging Face가 공식적으로 개발하는 Transformers.js는 Python `transformers` 라이브러리의 JavaScript 포팅이다. Hugging Face Hub에 있는 모델 중 ONNX 포맷으로 변환 가능한 것을 브라우저에서 직접 실행할 수 있다. 텍스트 분류, 임베딩, 음성 인식, 이미지 처리 등 광범위한 태스크를 커버한다.

WebGPU 가속을 지원하며, ONNX Runtime Web을 백엔드로 사용한다.

```bash
npm install @huggingface/transformers
```

```javascript
import { pipeline } from "@huggingface/transformers";

// 감정 분석 파이프라인 (브라우저에서 실행)
const classifier = await pipeline(
  "sentiment-analysis",
  "onnx-community/distilbert-base-uncased-finetuned-sst-2-english"
);

const result = await classifier("이 제품 정말 최고예요!");
console.log(result);
// [{ label: 'POSITIVE', score: 0.9998 }]
```

Transformers.js의 강점은 파이프라인 추상화다. `pipeline()` 함수 하나로 텍스트 분류, 토큰 분류, 번역, 요약, 임베딩 생성 등을 모두 처리할 수 있다. LLM 생성 작업도 지원한다:

```javascript
const generator = await pipeline(
  "text-generation",
  "onnx-community/Qwen2.5-0.5B-Instruct-ONNX",
  { device: "webgpu" } // WebGPU 가속
);

const output = await generator("자바스크립트의 클로저를 설명하면", {
  max_new_tokens: 100,
});
```

### ONNX Runtime Web: Microsoft의 범용 추론 엔진

ONNX Runtime Web은 Microsoft가 개발한 ONNX(Open Neural Network Exchange) 포맷 기반의 브라우저 추론 엔진이다. PyTorch, TensorFlow, Scikit-learn에서 학습한 모델을 ONNX로 변환하면 브라우저에서 실행할 수 있다.

WebGPU 백엔드와 WebAssembly 백엔드를 모두 지원하며, 2024년 2월 WebGPU를 통한 생성 AI 지원을 발표한 바 있다.

```html
<script src="https://cdn.jsdelivr.net/npm/onnxruntime-web/dist/ort.webgpu.min.js"></script>
<script>
  async function runInference() {
    // WebGPU 세션 생성
    const session = await ort.InferenceSession.create("./model.onnx", {
      executionProviders: ["webgpu", "wasm"],
    });

    // 입력 텐서 준비
    const input = new ort.Tensor("float32", [1, 3, 224, 224].flat(), [1, 3, 224, 224]);
    const feeds = { input: input };

    // 추론 실행
    const results = await session.run(feeds);
    console.log(results);
  }
</script>
```

ONNX Runtime Web의 장점은 프레임워크 중립성이다. 학습 프레임워크에 구애받지 않고 모든 모델을 동일한 인터페이스로 실행할 수 있다.

## 세 도구 비교: 무엇을 언제 써야 할까

| 항목 | WebLLM | Transformers.js | ONNX Runtime Web |
|------|--------|-----------------|------------------|
| 개발사 | MLC-AI | Hugging Face | Microsoft |
| 주요 타겟 | 대화형 LLM (7B~8B급) | 다양한 ML 태스크 전반 | 범용 추론 |
| API 형태 | OpenAI 호환 | pipeline 추상화 | 저수준 세션 API |
| GPU 가속 | WebGPU | WebGPU | WebGPU + WASM |
| 모델 포맷 | MLC 포맷 | ONNX | ONNX |
| 적합한 규모 | 7B~8B LLM | 소~중형 모델 | 소~중형 모델 |
| 학습 곡선 | 낮음 (OpenAI 경험 직결) | 중간 | 높음 |

선택 기준은 단순하다:

1. **대화형 챗봇이 필요하다** → WebLLM. OpenAI API와 동일한 인터페이스로 마이그레이션 부담이 가장 적다.
2. **임베딩, 분류, 음성 인식 등 다양한 ML 태스크가 필요하다** → Transformers.js. Hugging Face 생태계의 방대한 모델을 그대로 활용할 수 있다.
3. **커스텀 학습 모델을 브라우저에 배포해야 한다** → ONNX Runtime Web. PyTorch/TensorFlow 모델을 ONNX로 변환해서 직접 실행한다.

## 모델 압축: 브라우저 메모리 한계를 넘는 법

브라우저에서 LLM을 실행할 때 가장 큰 제약은 메모리다. Llama-3.1-8B 모델의 FP16 가중치는 약 16GB인데, 이를 그대로 브라우저에 올리는 것은 불가능하다. 세 가지 압축 기법이 이 문제를 해결한다.

**양자화(Quantization)**는 가중치의 정밀도를 낮춰 메모리를 줄인다. FP16(16비트)에서 INT4(4비트)로 변환하면 메모리 사용량이 약 75% 감소한다. Llama-3.1-8B의 경우 INT4 양자화 시 약 4.5GB로 줄어들어 평균적인 소비자 GPU에서 실행 가능해진다. WebLLM의 모델 ID에서 `q4f32_1` 접미사가 INT4 양자화를 의미한다.

**프루닝(Pruning)**은 모델에서 중요도가 낮은 가중치를 제거한다. 구조적 프루닝(structured pruning)은 채널이나 헤드 단위로 제거하여 실제 연산량을 줄인다.

**지식 증류(Knowledge Distillation)**는 큰 교사(teacher) 모델이 작은 학생(student) 모델을 훈련시키는 기법이다. 가장 잘 알려진 사례는 DistilBERT로, arXiv 1910.01108 논문(Victor Sanh et al., 2019)에 따르면 BERT-base 대비 40% 작은 크기로 60% 빠른 추론 속도를 달성하면서 GLUE 벤치마크에서 97%의 성능을 유지한다.

실제 적용에서는 이 세 기법을 조합한다. 2025년 발표된 압축 순서 연구에 따르면, 프루닝 → 지식 증류 → 양자화(P-KD-Q) 순서로 적용할 때 최적의 압축-성능 균형을 달성한다고 보고되었다. 이 순서가 중요한 이유는, 양자화를 증류보다 먼저 적용하면 perplexity가 한 자릿수에서 두 자릿수로 급증하기 때문이다.

```python
# PyTorch에서 ONNX로 변환하는 기본 예시
import torch

# 학습된 모델 로드
model = MyModel.load("model.pt")
model.eval()

# ONNX로 내보내기
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={"input": {0: "batch"}, "output": {0: "batch"}},
)
```

## 실제 시나리오: 언제 브라우저 LLM이 서버 LLM보다 나은가

브라우저에서 LLM을 실행하는 것이 항상 정답은 아니다. 대규모 모델(GPT-5, Claude Sonnet급)의 추론 품질을 브라우저에서 재현하는 것은 현재 기술로 불가능하다. 하지만 특정 시나리오에서는 클라이언트 사이드 실행이 명확한 이점을 갖는다.

**프라이버시가 최우선인 경우.** 의료, 법률, 금융 데이터를 다루는 애플리케이션에서는 사용자 입력이 서버로 전송되는 것 자체가 리스크다. 브라우저에서 실행하면 데이터가 기기를 한 번도 떠나지 않는다. WebLLM이나 Transformers.js로 임베딩을 생성하고, 그 결과를 로컬 벡터 검색에 활용하는 패턴이 이에 해당한다.

**오프라인 동작이 필요한 경우.** 네트워크가 불안정하거나 오프라인 환경에서도 AI 기능이 동작해야 하는 애플리케이션. ServiceWorker에 모델을 캐시하면 네트워크 연결 없이도 추론이 가능하다.

**API 비용을 최소화해야 하는 경우.** 자동완성, 텍스트 분류, 간단한 요약 같이 빈번하지만 복잡도가 낮은 작업은 브라우저에서 처리하고, 복잡한 추론만 서버 API로 라우팅하는 하이브리드 아키텍처가 효과적이다.

반대로 브라우저 LLM이 부적합한 경우도 명확하다:
- 최고 수준의 추론 품질이 필요한 경우 (코딩 에이전트, 복잡한 분석)
- 모바일 환경에서 대규모 모델을 실행해야 하는 경우 (메모리와 발열 제약)
- 실시간 다국어 번역처럼 지연 시간이 수백 밀리초 이내여야 하는 대규모 모델 작업

## 결론

브라우저에서 LLM을 실행하는 기술은 2026년 실용 단계에 진입했다. WebGPU가 Chrome과 Firefox에 기본 탑재되고, WebLLM이 OpenAI API와 호환되는 인터페이스를 제공하며, Transformers.js가 Hugging Face 생태계를 브라우저로 가져왔다. 이 조합은 프라이버시, 비용, 오프라인 동작이라는 세 가지 문제를 동시에 해결한다.

핵심 요약:

- **WebGPU + WebAssembly**가 브라우저에서 GPU 가속 LLM 추론을 가능하게 했다. Chrome은 안정 지원, Safari는 부분 지원, Firefox는 아직 플래그 비활성화 상태다.
- **WebLLM**은 대화형 LLM, **Transformers.js**는 다양한 ML 태스크, **ONNX Runtime Web**은 커스텀 모델에 각각 적합하다.
- 모델 크기는 **INT4 양자화**로 약 75%까지 줄일 수 있다. 8B급 모델이 평균적 소비자 GPU에서 실행 가능해진다.
- 브라우저 LLM은 **프라이버시, 오프라인, 비용 절감** 시나리오에서 서버 API를 대체할 수 있다.
- 서버 API와의 **하이브리드 라우팅**(단순 작업은 브라우저, 복잡한 작업은 서버)이 현실적인 최적 전략이다.

지금 당장 해볼 수 있는 첫 단계: WebLLM의 공식 데모([webllm.mlc.ai](https://webllm.mlc.ai/))에 접속해 Chrome에서 Llama 3를 브라우저에서 실행해보자. 서버 없이 8B 모델이 토큰을 생성하는 것을 직접 확인하면, 이 기술의 잠재력이 실감된다.
