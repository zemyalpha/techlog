---
title: "WASI 0.3으로 서버 사이드 WebAssembly가 바뀌었다 — 네이티브 비동기와 Wasmtime 46 실전 가이드"
date: "2026-10-31"
keywords: ["WASI 0.3", "WebAssembly", "Wasmtime", "컴포넌트 모델", "네이티브 비동기", "WIT"]
lang: "ko"
description: "2026년 6월 승인된 WASI 0.3이 네이티브 비동기(async func·stream·future)로 서버 사이드 Wasm을 어떻게 바꾸는지, WIT 계약 작성부터 wasmtime serve 실행, 생태계 마이그레이션 전략까지 코드와 함께 정리했다."
---

# WASI 0.3으로 서버 사이드 WebAssembly가 바뀌었다 — 네이티브 비동기와 Wasmtime 46 실전 가이드

서버에서 WebAssembly(Wasm)를 쓰려던 개발자들이 0.2 시절 가장 자주 부딪힌 벽은 비동기였다. 파일 읽기 하나, HTTP 요청 하나를 처리하려도 컴포넌트 전체가 블로킹되거나, 호스트가 폴링 루프를 직접 돌려야 했다. I/O 집약적인 마이크로서비스에는 치명적인 제약이었다.

2026년 6월 11일, WASI 서브그룹 표결로 **WASI 0.3.0이 정식 승인**되면서 이 벽이 무너졌다. 비동기가 라이브러리 레벨 합의가 아니라 컴포넌트 모델의 Canonical ABI 자체로 내려왔다. 그로부터 넉 달이 지난 지금, Wasmtime 46이 0.3을 기본 활성화했고, wasmCloud 2.5.0이 프로덕션 플랫폼에 0.3을 탑재했으며, 10월 13일에는 패치 릴리스 0.3.2까지 나왔다. 이제는 "미래 기술"이 아니라 마이그레이션 계획을 세워야 할 시점이다.

이 글에서는 WASI 0.3이 정확히 뭘 바꿨는지, 실제로 컴포넌트를 만들어 돌려보는 방법, 그리고 생태계가 어디까지 와 있는지를 공식 문서 기반으로 정리한다.

## 0.2에서 0.3으로: 무엇이 달라졌나

WASI 0.2는 컴포넌트 모델의 기반을 다진 릴리스였다. WIT(WebAssembly Interface Types)로 언어 중립적 인터페이스를 정의하고, 리소스 타입으로 상태ful 상호작용을 다뤘다. 다만 I/O는 여전히 동기 블로킹 시맨틱에 의존했거나, 복잡한 폴링 루프로 우회해야 했다.

WASI 0.3의 핵심 변화는 하나로 요약된다. **비동기가 WASI 인터페이스 위의 라이브러리가 아니라, 컴포넌트 모델 자체의 기본 요소가 되었다.** 구체적으로 세 가지 프리미티브가 WIT 언어와 Canonical ABI에 추가됐다.

- `async func` — 함수 자체를 비동기로 선언한다. 스케줄링과 재개(suspension/resume)는 런타임이 관리한다.
- `stream<T>` — 백프레셔를 인식하는 논블로킹 스트림. 컴포넌트 간 데이터 흐름을 표현한다.
- `future<T>` — 단일 값의 비동기 완료. 호스트의 이벤트 루프(Tokio, libuv 등)와 자연스럽게 통합된다.

왜 이게 중요한지는 wasi.dev의 릴리스 노트가 설명하는 `A → B → Host` 구성 문제에서 드러난다. 0.2에서 A 컴포넌트가 B를 거쳐 호스트 I/O를 호출하는 그래프가 있으면, 비동기 완료 알림을 각 단계마다 손으로 전파해야 했다. 0.3은 스케줄링과 웨이크업 전파를 런타임이 소유하기 때문에, 생산자와 소비자 사이에 컴포넌트가 몇 개나 끼어 있든 비동기가 올바르게 동작한다.

부수적으로 눈에 띄는 변화는 `wasi:io` 패키지가 통째로 사라졌다는 점이다. 스트림과 폴링 관련 기능이 Canonical ABI로 흡수되면서, 인터페이스 정의가 더 단순해졌다. 실제 WIT 시그니처는 이렇게 바뀌었다.

```wit
// WASI 0.2
read-via-stream: func() -> result<input-stream, error-code>;

// WASI 0.3
read-via-stream: func() -> tuple<stream<u8>, future<result<_, error-code>>>;
```

## 실전: 비동기 HTTP 핸들러 컴포넌트 만들기

간단한 예제로 흐름을 짚어보자. 목표는 WASI 0.3의 `wasi:http` 세계(world)를 export하는 비동기 핸들러 컴포넌트다.

**Step 1 — WIT 계약 작성.** 인터페이스의 단일 진실 원천은 WIT 파일이다:

```wit
package example:metrics@0.1.0;

interface processor {
  record Reading {
    sensor-id: string,
    value: float64,
  }

  // 0.3의 핵심: 함수 자체가 async
  process: async func(readings: list<Reading>) -> result<list<Reading>, string>;
}

world metrics-app {
  export processor;
}
```

`async func`로 선언하면 바인딩 생성기가 언어별 네이티브 비동기로 변환해 준다. Rust는 `async fn`, JavaScript는 `Promise`, Python은 코루틴으로 매핑된다.

**Step 2 — Rust 구현.** Rust는 현재 0.3 지원이 가장 앞서 있다. `wasip3` 크레이트(2026년 10월 27일 기준 최신 0.10.0+wasi-0.3.2)가 `wasi:cli/command`와 `wasi:http/service` 월드의 바인딩을 제공한다.

```bash
cargo add wasip3
cargo build --target wasm32-wasip3   # 또는 wasm32-wasip2 조합
```

일반 비즈니스 로직 컴포넌트라면 `wit-bindgen`으로 앞서 정의한 WIT에서 바인딩을 생성하는 방식이 표준이다:

```rust
wit_bindgen::generate!({
    world: "metrics-app",
    async: true,  // 0.3 네이티브 비동기 바인딩
});
```

참고로 타깃 상태는 이행 중이다. `wasm32-wasip2`는 Tier 2로 스테이블이고, `wasm32-wasip3`는 rustc 표준 문서(main 브랜치) 기준 Rust 1.100.0부터 스테이블 채널에서 사용 가능하도록 Tier 2 승격이 진행됐다. 다만 wasi.dev 언어 페이지에는 아직 Tier 3/nightly로 표기돼 있어 문서 간 시차가 있으니, 프로젝트 시작 시점에 실제 툴체인 상태를 확인하는 것이 안전하다.

**Step 3 — Wasmtime 46으로 실행.** Wasmtime 46은 최종 WASI 0.3.0 명세를 구현한 첫 릴리스이며, 0.3과 `component-model-async` 기능이 기본 활성화돼 있다:

```bash
wasmtime serve app.wasm          # 0.3 컴포넌트: wasi:http 0.3 service 월드 사용
wasmtime serve legacy.wasm       # 0.2 컴포넌트도 같은 바이너리로 실행됨
```

중요한 점은 `wasmtime serve`가 0.3 `service` 월드를 export하지 않는 컴포넌트는 자동으로 0.2 `wasi:http/proxy` 월드로 폴백한다는 것이다. 즉 런타임 교체 없이 이전 자산과 신규 컴포넌트를 같이 굴릴 수 있다.

## 생태계 현황: 어디까지 왔나

2026년 10월 말 기준, 0.3 지원은 "Rust 앞, 나머지 뒤" 국면이다.

- **Wasmtime 46+** — WASI 0.3 기본 활성화. 41~45는 RC 스냅샷 구현이며 `-Sp3 -W component-model-async=y` 플래그가 필요했다. 46에서 플래그 없이 동작한다.
- **wasmCloud 2.5.0** — Wasmtime 46으로 올라가며 WASI 0.3이 기본·유일 타깃. `wash-runtime`의 `wasip3` 피처 플래그가 제거됐고, Rust MSRV가 1.94.0으로 올랐다. `(implements ...)` 어노테이션으로 같은 인터페이스의 멀티플렉싱 라우팅도(피처 플래그 뒤에서) 들어왔다.
- **jco (JS/TS)** — `preview3-shim` 패키지로 Node.js 환경의 실험적 0.3 구현 제공.
- **Rust `wasip3` 크레이트** — 정식 0.3 바인딩. 0.3.0(6월), 0.3.1(8월)에 이어 0.3.2 버전이 10월 27일에 발행됐다.
- **그 외 언어** — C/C++(wasi-sdk), Python(componentize-py), Go(TinyGo), C#(componentize-dotnet) 등 대다수 툴체인은 아직 0.2 타깃이다. Java·Swift는 로드맵 단계.

릴리스 자체는 이제 예측 가능한 리듬을 갖는다. WASI 0.3 패치 릴리스는 **2개월마다 둘째 주 화요일**에 출시되는 "릴리스 트레인" 모델을 따른다. 0.3.0(6월 11일)만 표결에 의한 예외였고, 0.3.1이 8월 11일, 0.3.2가 10월 13일에 나왔으며, 다음은 0.3.3(12월 8일 예정)이다. 0.3.1에서는 `map<K, V>` 타입과 `implements`·`external-id` 어노테이션이 채택됐다.

컨테이너 대비 낮은 콜드스타트와 작은 메모리 풋프린트는 Wasm의 오랜 강점인데, 0.3으로 비동기라는 마지막 취약점이 채워진 만큼 엣지·플러그인 시스템·고밀도 샌드박스 워크로드에서 채택이 빨라질 관측이다. 다만 구체적 수치는 환경과 워크로드에 따라 편차가 크니, 자체 워크로드로 재는 것을 권한다.

## 마이그레이션 전략과 주의사항

**급할 필요 없다 — 하지만 방향은 정해야 한다.** wasi.dev 로드맵은 "0.3 지원 런타임(Wasmtime, jco)은 0.2 컴포넌트도 실행한다"를 명시한다. 0.2를 0.3 위에서 가상화(폴리필)하는 것도 허용되는 경로다. 신규 WIT 설계만 0.3으로 잡고 기존 자산은 점진적으로 옮기면 된다.

실무에서 자주 걸리는 함정 네 가지:

1. **버전 핀닝** — WIT 인식 도구들이 아직 버전 인식 링킹을 완전히 지원하지 않는다. 관련 도구 전체에서 동일한 버전(예: `0.3.0`)을 맞춰라. 불일치는 인스턴스화 시점의 애매한 `wrong type` 에러로 터진다.
2. **언어별 지원 편차** — Rust만 0.3 정식 바인딩이 있다. 파이썬·Go·C# 컴포넌트가 섞인 프로젝트라면 아직 0.2로 통일하는 게 맞다.
3. **스레딩은 아직 아니다** — 0.3의 후속 후보로 threading built-ins, stackful async lift 등이 게이트 뒤에서 개발 중이다. Rust 표준 문서도 `wasm32-wasip3`에서 `std::thread` 지원이 아직 완성 전임을 명시한다(2026년 9월 초 기준). 스레드 의존 설계는 0.3.x 후속을 기다려야 한다.
4. **Wasmtime 버전 확인** — 45 이하는 RC 스냅샷 구현이라 최종 명세와 다를 수 있다. 프로덕션은 46 이상으로.

## 결론

- WASI 0.3.0(2026년 6월 11일 승인)은 `async func`·`stream<T>`·`future<T>`를 컴포넌트 모델의 기본 요소로 만들어, 서버 사이드 Wasm의 비동기 결함을 구조적으로 해결했다.
- Wasmtime 46이 0.3을 기본 활성화했고, wasmCloud 2.5.0이 프로덕션 플랫폼에 탑재했다. 패치 릴리스는 2개월 주기의 트레인 모델로 0.3.2(10월 13일)까지 순항 중이다.
- 언어 지원은 Rust가 앞선다. `wasip3` 크레이트와 `wasm32-wasip3` 타깃(Tier 2 승격 진행, Rust 1.100.0~)이 준비돼 있고, 다른 툴체인은 대부분 아직 0.2다.
- 마이그레이션은 급하지 않다. Wasmtime은 0.2·0.3 컴포넌트를 동시에 실행한다. 신규 WIT만 0.3으로 잡는 전략이 합리적 출발점이다.

당장 시도해볼 첫 단계는 간단하다. `cargo add wasip3`로 바인딩을 추가하고, WIT에 `async func` 하나를 선언한 뒤, Wasmtime 46의 `wasmtime serve`로 돌려보는 것. 컴포넌트 경계를 넘는 비동기가 플래그 없이 그냥 동작하는 경험이 0.2 시절과 얼마나 다른지 직접 확인할 수 있다.
