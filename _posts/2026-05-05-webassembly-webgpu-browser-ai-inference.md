---
title: "브라우저에서 AI 모델 직접 돌리기: WebAssembly + WebGPU 실전 가이드"
date: "2026-05-05"
keywords: ["WebAssembly", "WebGPU", "브라우저 AI", "Transformers.js", "ONNX Runtime Web", "WebLLM"]
lang: "ko"
description: "2026년 WebGPU 전 브라우저 지원과 Transformers.js v4 릴리스로 브라우저 기반 AI 추론이 프로덕션 단계에 진입했다. 실전 코드와 성능 비교로 브라우저 AI 도입법을 정리한다."
---

# 브라우저에서 AI 모델 직접 돌리기: WebAssembly + WebGPU 실전 가이드

서버에 AI 모델을 두고 API로 호출하는 것이 당연한 시대다. 하지만 매 요청마다 네트워크 왕복이 발생하고, API 비용은 누적되며, 민감한 데이터를 외부로 보내야 한다. 2026년 들어 이 패러다임에 균열이 생기고 있다. 브라우저 자체에서 AI 모델을 실행하는 기술이 프로덕션 수준에 도달했다.

핵심은 두 기술의 결합이다. **WebAssembly(WASM)** 가 이식성과 샌드박스 보안을 제공하고, **WebGPU** 가 GPU 연산을 브라우저로 가져온다. 이 글에서는 이 기술 스택이 2026년 어디까지 왔는지, 실제 어떻게 쓰는지를 코드와 함께 정리한다.

## 1. 왜 브라우저에서 AI를 돌려야 하는가

브라우저 기반 AI 추론이 의미 있는 상황은 구체적이다.

**첫째, API 비용 문제.** 이미지 분류, 텍스트 임베딩, 감정 분석 같은 경량 작업을 매번 클라우드 API로 처리하면 비용이 빠르게 누적된다. 이런 작업은 브라우저에서 로컬로 처리하면 서버 측 비용이 0원이 된다.

**둘째, 지연 시간.** 서버 왕복이 200ms 걸리는 작업을 로컬에서 50ms 이내에 끝낼 수 있다면, 사용자 경험이 근본적으로 달라진다. 실시간 인터랙션이 필요한 애플리케이션에서 이 차이는 결정적이다.

**셋째, 데이터 프라이버시.** 의료, 금융, 개인 문서 등 민감한 데이터를 외부 서버로 전송하지 않고 브라우저 내에서 처리할 수 있다. 규제가 엄격한 산업에서 특히 중요하다.

물론 제약도 있다. 브라우저 메모리는 보통 2~4GB로 제한되고, 모델 크기에 따라 초기 로딩 시간이 길어진다. GPT-4 급 대형 모델은 아직 브라우저에서 돌리기 어렵지만, 3B~7B 파라미터 모델은 충분히 현실적이다.

## 2. 2026년 기술 환경: 무엇이 달라졌나

### WebGPU, 전 브라우저 지원 달성

2026년 1월, WebGPU가 마지막 보루였던 Firefox와 Safari에서 지원을 활성화하면서 모든 주요 브라우저에서 사용 가능해졌다. Firefox 147이 1월 13일 WebGPU를 탑재했고, Safari는 iOS 26과 macOS Tahoe 26에서 기본 활성화되었다. Chrome과 Edge는 이미 2023년부터 지원했다. 전체 브라우저 커버리지는 약 70%에 달한다.

WebGPU가 WebGL과 근본적으로 다른 점은 GPU를 저수준에서 직접 제어할 수 있다는 것이다. WebGL이 고수준 상태 기계 API였다면, WebGPU는 비동기 멀티스레드 명령 버퍼를 통해 GPU 병렬 처리를 가능하게 한다. 컴퓨트 워크로드 기준으로 WebGL 대비 15~30배 성능 향상이 보고되었다.

### Transformers.js v4 릴리스

Hugging Face의 Transformers.js는 2026년 2월 v4를 릴리스했다. 가장 큰 변화는 WebGPU 런타임을 C++로 완전히 재작성한 것이다. ONNX Runtime 팀과 협력해 약 200개 모델 아키텍처에서 테스트를 마쳤다.

성능 개선도 눈에 띈다. `com.microsoft.MultiHeadAttention` 오퍼레이터를 도입해 BERT 기반 임베딩 모델에서 약 4배 속도 향상을 달성했다. 또한 Node.js, Bun, Deno 같은 서버 사이드 런타임에서도 WebGPU 가속 모델을 실행할 수 있게 되었다. 브라우저와 서버에서 동일한 코드가 돌아가는 건 의미 있는 변화다.

## 3. 실전: 프레임워크별 브라우저 AI 추론

현재 브라우저 AI 추론의 3대 프레임워크는 Transformers.js, ONNX Runtime Web, WebLLM이다. 각각의 특징과 실제 코드를 비교해본다.

### Transformers.js — 가장 쉬운 시작점

Transformers.js는 Hugging Face 생태계와 직접 연동되어 수천 개의 사전 학습 모델을 브라우저에서 바로 사용할 수 있다.

```bash
npm install @huggingface/transformers
```

감정 분석 파이프라인 예시:

```javascript
import { pipeline } from "@huggingface/transformers";

// 파이프라인 생성 (첫 실행 시 모델 다운로드, 이후 캐시)
const classifier = await pipeline("sentiment-analysis");

const result = await classifier("이 제품 정말 좋아요!");
console.log(result);
// [{ label: "POSITIVE", score: 0.9998 }]
```

텍스트 임베딩으로语义 검색 구현:

```javascript
import { pipeline } from "@huggingface/transformers";

const embedder = await pipeline("feature-extraction", 
  "BAAI/bge-small-en-v1.5", 
  { dtype: "fp32" }
);

const embeddings = await embedder("검색할 문장", {
  pooling: "mean",
  normalize: true,
});
// 384차원 벡터 반환 → 코사인 유사도로 문서 검색
```

v4에서는 모델 레지스트리(ModelRegistry)가 추가되어 모델 캐시와 버전 관리가 개선되었다. 환경 설정(Environment Settings)으로 백엔드(WASM 또는 WebGPU)를 명시적으로 선택할 수도 있다.

### ONNX Runtime Web — 정밀한 제어가 필요할 때

Microsoft의 ONNX Runtime Web은 ONNX 포맷 모델을 브라우저에서 실행한다. WASM 백엔드와 WebGPU 백엔드를 모두 지원하며, SIMD와 스레딩을 활용한 최적화가 잘 되어 있다.

```html
<script src="https://cdn.jsdelivr.net/npm/onnxruntime-web/dist/ort.all.min.js"></script>
<script>
  // WebGPU 백엔드 사용 설정
  ort.env.wasm.numThreads = navigator.hardwareConcurrency || 4;
  
  const session = await ort.InferenceSession.create(
    "model.onnx",
    { executionProviders: ["webgpu", "wasm"] }
  );

  // 입력 텐서 생성
  const inputTensor = new ort.Tensor("float32", 
    new Float32Array([/* 데이터 */]), 
    [1, 3, 224, 224]
  );

  const results = await session.run({ input: inputTensor });
  console.log(results.output.data);
</script>
```

ONNX Runtime Web의 장점은 PyTorch, TensorFlow, scikit-learn 등 다양한 프레임워크에서 학습한 모델을 ONNX로 변환만 하면 바로 사용할 수 있다는 것이다. Transformers.js가 Hugging Face 생태계에 종속적인 것과 대비된다.

### WebLLM — 브라우저에서 LLM 실행

MLC AI의 WebLLM은 LLaMA, Mistral, Gemma, Phi, Qwen 같은 LLM을 브라우저에서 직접 실행한다. WebGPU를 활용해 네이티브 성능의 약 80%에 도달한다고 보고되었다.

```javascript
import { CreateMLCEngine } from "@mlc-ai/web-llm";

// LLM 엔진 초기화 (모델 다운로드 포함)
const engine = await CreateMLCEngine("Llama-3.2-1B-Instruct-q4f16_1-MLC");

const reply = await engine.chat.completions.create({
  messages: [
    { role: "system", content: "한국어로 답변하세요." },
    { role: "user", content: "WebAssembly를 한 줄로 설명해줘." },
  ],
  temperature: 0.7,
  max_tokens: 256,
});

console.log(reply.choices[0].message.content);
```

WebLLM은 OpenAI 호환 API를 제공하는 것이 특징이다. 기존에 OpenAI API를 사용하던 코드에서 `baseUrl`만 WebLLM 엔드포인트로 바꾸면 로컬 추론으로 전환할 수 있다. 마이그레이션 비용이 거의 없다.

## 4. 성능 비교와 선택 기준

실제 프로덕션 환경에서 측정된 대략적인 성능 수치를 정리한다. (모델과 하드웨어에 따라 편차가 크므로 참고 수치로 활용할 것)

| 작업 | 모델 크기 | WASM + SIMD | WebGPU | 비고 |
|------|-----------|-------------|--------|------|
| 감정 분석 | ~60MB | ~30ms | ~10ms | BERT-tiny 기준 |
| 텍스트 임베딩 | ~130MB | ~80ms | ~20ms | bge-small 기준 |
| 이미지 분류 | ~100MB | ~150ms | ~40ms | MobileNet-v3 기준 |
| 텍스트 생성 | ~1.5GB | ~15 tok/s | ~45 tok/s | Llama-3.2-1B q4 기준 |
| 텍스트 생성 | ~4GB | 불가능 | ~20 tok/s | Llama-3.2-3B q4 기준 |

프레임워크 선택 기준은 다음과 같다.

- **빠른 프로토타이핑** → Transformers.js. 파이프라인 API 한 줄로 대부분의 작업이 가능하다.
- **기존 ONNX 모델 활용** → ONNX Runtime Web. PyTorch에서 학습한 모델을 그대로 브라우저로 가져올 수 있다.
- **LLM 채팅/생성** → WebLLM. OpenAI 호환 API로 마이그레이션이 쉽고, WebGPU 가속으로 실용적인 속도가 나온다.

## 5. 프로덕션 도입 시 주의사항

**모델 로딩 시간.** 브라우저에서 모델을 처음 다운로드할 때 시간이 걸린다. 1GB 모델은 빠른 네트워크에서도 5~10초가 소요된다. Cache API나 IndexedDB를 활용해 모델을 로컬에 캐시하는 것이 필수적이다.

```javascript
// Transformers.js v4에서는 내부적으로 캐시를 관리하지만,
// 명시적 제어가 필요하면 환경 설정 활용
import { env } from "@huggingface/transformers";

env.allowLocalModels = false;  // 로컬 파일 시스템 접근 비활성화
env.useBrowserCache = true;    // 브라우저 캐시 사용 (기본값)
```

**메모리 관리.** 브라우저 탭 하나가 사용할 수 있는 메모리는 제한적이다. 모델을 여러 개 로드하면 탭이 크래시될 수 있다. 사용하지 않는 모델은 명시적으로 dispose해야 한다.

**WebGPU 미지원 환경 폴백.** WebGPU 커버리지가 70%라도 나머지 30% 사용자를 무시할 수는 없다. WASM 백엔드로 폴백하는 로직이 필요하다.

```javascript
// WebGPU 지원 확인 후 백엔드 선택
const useWebGPU = typeof navigator !== "undefined" && 
  !!navigator.gpu;

const session = await ort.InferenceSession.create("model.onnx", {
  executionProviders: useWebGPU ? ["webgpu", "wasm"] : ["wasm"],
});
```

**모바일 환경.** 모바일 브라우저에서는 메모리와 GPU 성능이 더 제한적이다. 모바일 타겟이라면 1B 이하 모델과 양자화(q4, q8)를 적극 활용해야 한다.

## 결론: 지금 브라우저 AI를 시도해야 할 순간

WebGPU의 전 브라우저 지원, Transformers.js v4의 WebGPU 런타임 재작성, WebLLM의 실용적 성능 달성 — 이 세 가지가 2026년 상반기에 동시에 이루어졌다. 브라우저 AI 추론은 더 이상 실험 단계가 아니다.

당장 시도해볼 수 있는 첫 단계를 제안한다.

- **가장 빠른 시작:** Transformers.js로 감정 분석이나 텍스트 임베딩 파이프라인을 하나 만들어 본다. 10줄 안팎의 코드로 로컬 AI를 경험할 수 있다.
- **기존 모델 마이그레이션:** PyTorch 모델을 ONNX로 변환한 후 ONNX Runtime Web으로 브라우저 배포를 테스트한다.
- **LLM 로컬 채팅:** WebLLM으로 소형 LLM(Llama-3.2-1B)을 브라우저에서 돌려본다. API 키 없이, 서버 없이, 완전히 로컬에서 동작하는 챗봇을 만들 수 있다.

서버 없이, API 키 없이, 데이터가 외부로 나가지 않으면서도 실용적인 속도로 AI가 동작하는 시대가 열렸다. 한 번 직접 경험해보면 가능성이 보일 것이다.
