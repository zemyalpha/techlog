---
title: "자바스크립트 도구체인의 지반은 이미 Rust다 — Vite 8·Rolldown·tsrs가 바꾼 2026년 프론트엔드 빌드 판도"
date: "2026-10-29"
keywords: ["Rust", "Vite 8", "Rolldown", "Oxc", "tsrs", "TypeScript", "빌드 도구", "프론트엔드"]
lang: "ko"
description: "Vite 8의 Rolldown 전환, Cloudflare의 VoidZero 인수, TypeScript 컴파일러의 Rust 포트 tsrs까지 — 2026년 JS 도구체인이 Rust로 이동한 이유와 실전 마이그레이션 가이드"
---

# 자바스크립트 도구체인의 지반은 이미 Rust다

2026년 프론트엔드 개발자가 `npm run build`를 실행할 때, 그 명령 뒤에서 돌아가는 코드의 상당수는 더 이상 자바스크립트가 아니다. 번들러는 Rust로 작성된 Rolldown이고, 린터는 Rust 기반 Oxlint이며, 포매터도 Oxfmt이고, 심지어 TypeScript 컴파일러 자체의 Rust 포트(tsrs)까지 등장했다. 몇 년 전만 해도 "JS 생태계의 Rust화"는 실험적 프로젝트 몇 개로 점칠 수 있는 트렌드였다면, 지금은 완료된 사실에 가깝다.

이 글에서는 2026년에 이 전환이 어떤 경로로 완성되었는지, 각 층위의 도구가 실제로 얼마나 빨라졌는지, 그리고 현업 프로젝트에서 무엇을 지금 바꿔야 하고 무엇을 기다려야 하는지 정리한다.

## 1. 전환이 완성된 1년: Vite 8, VoidZero 인수, tsrs

타임라인부터 짚어보자. 핵심 사건은 세 가지다.

**첫째, Vite 8.0 정식 출시 (2026년 3월 12일).** Vite 팀 공식 발표에 따르면 Vite 8은 Rust로 작성된 Rolldown을 단일 통합 번들러로 탑재했다. 개발 서버와 프로덕션 빌드가 서로 다른 번들러(esbuild + Rollup)를 쓰던 구조를 없애고 하나의 Rust 번들러로 통합한, Vite 2 이후 가장 큰 아키텍처 변경이다. Vite 팀은 Rolldown 전환으로 기존 대비 10~30배 빠른 빌드를 보고했다.

**둘째, Cloudflare의 VoidZero 인수 (2026년 6월 4일).** Cloudflare는 Vite, Vitest, Rolldown, Oxc를 만든 VoidZero를 인수했다고 공식 발표했다. Evan You와 팀은 Cloudflare의 ETI(신기술 인큐베이션) 조직으로 합류하되, Vite·Vitest·Rolldown·Oxc·Vite+는 MIT 라이선스의 오픈소스·벤더 중립 상태를 유지하며, Cloudflare는 커뮤니티 메인테이너를 위한 독립적인 100만 달러 Vite 생태계 펀드를 조성했다. 참고로 VoidZero 발표에 따르면 Vite는 주간 다운로드 1억 회를 넘는, 사실상 모던 JS 개발의 표준 도구다.

**셋째, tsrs 공개 (2026년 10월).** Cloudflare와 VoidZero가 TypeScript 컴파일러를 Rust로 포트하는 tsrs를 발표했다. TypeScript 팀과 긴밀히 협업하며 Oxc 도구체인의 일부로 incubating된다는 점이 핵심이다. 초기 벤치마크에서 tsrs는 동일한 타입 검사 워커 수 기준 TypeScript 7.0 대비 최대 8배 빠르고 메모리는 약 70% 적게 쓴다고 한다.

인수 4개월 후 Cloudflare가 공개한 성과 보고에 따르면, 이 팀은 80회 이상의 릴리스와 1,200개 이상의 이슈 클로즈를 진행했고 Oxc React Compiler(React 앱 컴파일 10배 빠름), Vitest 5(4 대비 최대 50% 빠름), tsgolint 안정화(대형 코드베이스에서 ESLint 대비 최대 18배) 등을 출시했다.

## 2. 왜 Rust인가 — "사람이 아니라 에이전트가 도구를 돌린다"

속리 속리만 보면 이유는 단순하다. GC가 없고 예측 가능한 성격의 네이티브 코드가 계산 집약 작업(파싱, 변환, 번들링, 타입 검사)에 유리한 건 당연하다. 그런데 2026년에 이 전환이 급속도로 완성된 데는 새로운 변수가 있다. **AI 코딩 에이전트다.**

사람이 개발할 때는 하루에 빌드를 몇십 번 돌린다. 에이전트는 코드를 수정할 때마다 빌드·린트·테스트를 반복 실행한다. Cloudflare와 VoidZero 모두 "에이전트가 도구를 반복 실행하는 시대"를 인수의 배경으로 명시적으로 언급했다. 도구 실행 횟수가 기하급수적으로 늘면, 도구 한 번의 지연이 그대로 에이전트 루프 전체의 지연으로 곱해진다. 빌드가 30초에서 1초로 줄어들면 사람에게는 편리함이지만, 하루에 수백 번 루프를 도는 에이전트에게는 비용 구조 자체가 바뀐다.

즉, Rust 전환은 단순한 성능 경쟁이 아니라 "에이전트 시대의 도구 호출 단가"를 낮추는 인프라 투자다.

## 3. 층위별 현황: 어디까지 Rust가 됐나

2026년 10월 현재 주요 층위는 다음과 같다.

| 층위 | 기존 (JS) | 현재 (Rust) | 발표된 성능 |
|---|---|---|---|
| 번들러 | Rollup, esbuild* | Rolldown (Vite 8 기본) | 빌드 10~30배 (Vite 팀 발표) |
| 린터 | ESLint | Oxlint / tsgolint | 대형 프로젝트 18~50배 이상 |
| 포매터 | Prettier | Oxfmt | 7~30배 |
| 테스트 러너 | Vitest 4 | Vitest 5 (핵심 Rust화) | 최대 50% |
| 타입 체커 | TypeScript (JS, Go 포트 진행) | tsrs (개발 중) | 최대 8배, 메모리 ~70% 절감 |

*esbuild 자체도 Go로 작성됐다. 즉 JS로 된 번들러는 사실상 Rollup이 마지막이었고, 그마저 Rolldown이 API 호환성을 목표로 대체하고 있다.

주의할 점: 위 수치는 전부 각 도구 팀이 자체 벤치마크로 발표한 값이다. 특히 "10~30배" 같은 폭이 큰 수치는 프로젝트 규모와 구성에 따라 편차가 크다. 실제 프로젝트에서는 하단 끝값 근처에서 시작한다고 예상하는 게 현실적이다.

## 4. 실전 마이그레이션: 지금 할 수 있는 것

### Vite 7 → Vite 8

Vite 8 마이그레이션의 핵심은 번들러 교체다. 커뮤니티 마이그레이션 가이드들이 권장하는 검증 경로는 `rolldown-vite` 패키지로 먼저 호환성을 확인하는 것이다.

```bash
# 1단계: 번들러만 교체해 호환성 확인
npm remove vite
npm install rolldown-vite

# 2단계: 빌드가 통과하면 정식 Vite 8로 전환
npm remove rolldown-vite
npm install vite@8
```

Vite 8에서 조용히 동작이 바뀌는 설정 키들이 있다고 커뮤니티 가이드들이 지적하므로, 마이그레이션 후에는 빌드 산출물 크기와 청크 구성을 반드시 비교해야 한다.

### ESLint → Oxlint

Oxlint는 ESLint 플러그인 호환성을 갖추고, tsgo 기반의 타입 인식 린팅(tsgolint)까지 지원한다.

```bash
npm install -D oxlint
npx oxlint --version
```

현실적인 접근은 완전 교체가 아니라 이원화다. 빠른 규칙은 Oxlint가 CI와 에디터에서, 복잡한 커스텀 규칙은 기존 ESLint가 담당하는 구조로 시작해 점진적으로 옮기는 것이다.

### Prettier → Oxfmt

Oxfmt는 Prettier 호환을 목표로 하며, JSON·CSS·SCSS·GraphQL·YAML 포매터까지 Rust로 재작성되어 Cloudflare 발표 기준 7배 빠르다. 포매팅은 원래 결과물이 결정적(deterministic)이므로, 마이그레이션 리스크가 가장 낮은 영역이다. 다만 프리티어 플러그인 의존(예: organize-imports 서드파티)이 있다면 호환 여부를 먼저 확인해야 한다.

## 5. 기다려야 하는 것: tsrs

tsrs는 아직 incubating 단계다. TypeScript 팀이 언어 설계의 결정권을 계속 갖고, tsrs는 그 결정을 엄격히 따르는 구현으로 유지된다는 방침이다. 이는 현명한 설계다 — 언어의 사실상 표준 구현이 특정 업체의 속도에 끌려가면 생태계 전체가 흔들린다.

실무 관점에서는 TypeScript 7(Go 기반 네이티브 포트)의 안정화가 먼저고, tsrs는 그 다음 세대의 선택지로 보는 게 맞다. 다만 방향성은 분명하다. 타입 체크가 빌드의 병목이던 시대가 끝나가고 있다.

## 결론

- Vite 8의 Rolldown 통합(2026년 3월)으로 JS 번들링의 Rust 전환은 사실상 완료됐다.
- Cloudflare의 VoidZero 인수(6월)는 이 도구체인이 "에이전트가 쓰는 인프라"로 재정의됐음을 보여준다.
- tsrs(10월)는 컴파일러라는 마지막 성벽까지 Rust가 밀고 들어가는 중이라는 신호다.
- 현업에서 지금 할 것: Vite 8 마이그레이션 검증, Oxlint/Oxfmt 이원화 도입.
- 기다릴 것: tsrs 안정화. TypeScript 7 정착 이후의 이야기다.

가장 중요한 관점 변화는 이것이다. 도구의 속도는 이제 개발자 경험(DX)의 문제이기 이전에 **에이전트 운영 비용**의 문제다. 하루에 몇 번 빌드하던 시대의 도구 선택 기준과, 하루에 수천 번 빌드하는 에이전트 시대의 기준은 같을 수 없다. 자바스크립트 생태계는 이미 답을 내렸고, 그 답은 Rust였다.
