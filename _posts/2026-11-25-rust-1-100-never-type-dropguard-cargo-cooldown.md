---
title: "Rust 1.100이 10년 묵은 never type의 숙제를 풀었다 — DropGuard, Cargo 쿨다운까지 업그레이드 체크리스트"
date: "2026-11-25"
keywords: ["Rust 1.100", "never type", "DropGuard", "min-publish-age", "공급망 보안", "Allocator API"]
lang: "ko"
description: "Rust 1.100.0(2026-11-12)의 핵심 변화를 정리한다 — 10년 만에 안정화된 never type, core::mem::DropGuard, Cargo 의존성 쿨다운(min-publish-age)과 업그레이드 시 실제로 부딪히는 호환성 주의점."
---

# Rust 1.100이 10년 묵은 never type의 숙제를 풀었다 — DropGuard, Cargo 쿨다운까지 업그레이드 체크리스트

2026년 11월 12일 발표된 Rust 1.100.0은 단순한 기능 추가 릴리스가 아니었다. 언어 설계자들이 10년 넘게 끌어안고 있던 숙제(never type)를 마무리했고, 동시에 Cargo에는 공급망 공격에 대한 실질적 방어선(의존성 쿨다운)이 들어갔다. 언어의 근본과 생태계의 안전성이라는 서로 다른 축의 변화가 하나의 버전에 함께 도착한 셈이다.

이 글은 공식 릴리스 발표문과 2026 프로젝트 로드맵을 기반으로, 1.100에서 실제로 무엇이 바뀌었는지 그리고 기존 프로젝트를 업그레이드할 때 어디를 확인해야 하는지를 정리한다.

## never type: 10년과 5번의 실패 끝의 안정화

`!`로 표기되는 never type은 "절대 값을 만들어내지 않는 계산"의 타입이다. `return`, `break`, `std::process::exit` 같은 제어 흐름을 종료하는 표현식이 이 타입을 가지며, 값이 존재할 수 없으므로 어떤 타입으로든 강제 변환(coercion)된다.

사실 이 개념 자체는 이미 코드에서 쓰이고 있었다. `match`의 한 분기에서 `return`을 써도 타입 검사가 통과했던 것이 그 증거다. 1.100에서 달라진 점은 **어디서든 `!`를 이름으로 쓸 수 있게 되었다**는 것이다:

```rust
fn infallible_unwrap<T>(res: Result<T, !>) -> T {
    // 기존에는 Result<_, !> 표기가 불가능했다
    let Ok(v) = res;
    v
}
```

또한 기존의 `Infallible`이 `!`의 별칭(alias)으로 변경되어, 옛 API들이 그대로 호환된다. 공식 발표문은 이 안정화가 "10년과 5번의 실패한 안정화 시도 끝"에 이루어졌다고 밝히고 있다. 왜 이렇게 오래 걸렸는지가 이번 릴리스의 실질적인 교훈이다.

### 조용한 breaking change: never type fallback

`Infallible` 변경을 위해 감수해야 했던 것이 있다. **2024 에디션 이전(2015, 2018, 2021)에서 never type fallback이 `()`에서 `!`로 바뀌었다.** 대부분의 코드는 변화를 느끼지 못하지만, 제네릭 파라미터를 반환하는 함수의 결과를 무시하는 코드에서 타입 추론 결과가 달라질 수 있고, 컴파일러가 더 많은 코드를 도달 불가능(unreachable)로 판정할 수 있다.

공식 발표문은 이 호환성 경고를 상당히 이른 시점에 미리 제공했고, 깨진 공개 크레이트에는 수정본을 이미 릴리스했다고 밝혔다. 따라서 업그레이드 후 의존성 빌드가 never type 관련으로 실패한다면, **먼저 해당 크레이트 업데이트를 시도**하는 것이 정답이다.

## DropGuard: 실패 경로의 롤백을 표준 라이브러리로

Rust의 `Drop` 트레이트는 값이 스코프를 벗어날 때 `return`이든 `?`든 `panic`이든 상관없이 정리 코드를 실행해 준다. 다만 정리용 타입을 매번 직접 구현하는 것은 번거로운 일이었다. 1.100은 이를 위한 표준 헬퍼 `core::mem::DropGuard`를 추가했다.

`DropGuard`는 변수와 클로저를 받아 `Deref`/`DerefMut`로 변수를 그대로 쓰게 해 주고, 드롭될 때 클로저를 실행하며 변수를 소비한다. `dismiss` 연관 함수로 정상 경로에서는 가드를 해제할 수도 있다. 공식 문서의 파서 예제가 사용 패턴을 잘 보여준다:

```rust
use std::mem::DropGuard;
use std::num::ParseIntError;

/// 모든 문자열을 파싱해 nums에 push한다.
/// 하나라도 실패하면 nums는 변경되지 않은 채로 남는다.
fn parse_all(nums: &mut Vec<u32>, strings: &[&str]) -> Result<(), ParseIntError> {
    let len = nums.len();
    let mut nums = DropGuard::new(nums, move |nums| {
        nums.truncate(len); // 1. 실패 시 벡터를 원상복구
    });
    for s in strings {
        nums.push(s.parse()?); // 2. ?가 에러를 반환하면 롤백
    }
    DropGuard::dismiss(nums); // 3. 성공 시 가드 해제
    Ok(())
}
```

올오어낫싱(all-or-nothing) 트랜잭션 API를 만들 때마다 반복하던 보일러플레이트가 표준 타입 하나로 대체된다. 참고로 `dismiss`가 메서드가 아니라 연관 함수로 설계된 것은, `Deref` 경유의 메서드 해석이 내부 타입의 메서드와 충돌하지 않게 하기 위한 의도적인 선택이다.

함께 추가된 `std::mem::conjure_zst`는 unsafe 코드에서 크기 0 타입의 값을 만드는 표준적 방법을 제공한다. `transmute::<(), T>`나 `MaybeUninit::uninit().assume_init()` 같은 관용구를 대체하며, PyO3의 `Python<'py>` 토큰처럼 "생성 불가능성"이 안전성 보장인 타입이 있기 때문에 이 함수는 `unsafe`다.

## Cargo: 공급망 방어가 도구로 굳어지다

### 의존성 쿨다운 (min-publish-age)

악성 버전의 패키지는 대개 게시 후 몇 시간에서 며칠 안에 자동 스캐너나 커뮤니티에 의해 발각된다. 1.100에서 안정화된 min-publish-age 인식 의존성 해석은 이 사실을 이용한다 — **새로 게시된 버전을 지정한 기간만큼 기다렸다가 받는 것**이다:

```toml
# .cargo/config.toml
[registry]
global-min-publish-age = "3 days"
```

설정 후 `cargo update`를 실행하면 Cargo가 보류한 버전을 명시적으로 알려준다:

```
$ cargo update
    Updating crates.io index
 Locking 2 packages to highest compatible version as of min-publish-age
 Adding cc v1.11.0 (available: v1.18.0, published 3 hours ago)
 Adding clap 4.6.7 (available: 4.6.9, published 2 days ago)
```

긴급하게 새 버전이 필요하면 환경변수로 단건 상향이 가능하다:

```
$ CARGO_RESOLVER_INCOMPATIBLE_PUBLISH_AGE=allow cargo update clap --precise 4.6.9
```

알아둘 제약도 있다. 인덱스에 게시 시각(`pubtime`)이 없는 버전은 보류되지 않고, `cargo install`은 이 규칙을 무시한다(필요하면 `cargo install foo@1.2.3`으로 고정). 공식 발표문도 강조하듯 쿨다운은 일부 공급망 공격의 위험을 줄일 뿐 완전한 방어가 아니며, 의존성 업데이트 리뷰는 계속해야 한다. 이 기능은 RFC 3923으로 설계되었고 crates.io 공급망 강화 프로젝트의 일부다.

### 미사용 의존성 경고와 캐시 충돌 해소

같은 릴리스에서 Cargo는 빌드 타깃 전체에서 쓰이지 않는 의존성을 경고하는 기능(`[lints.cargo]` 테이블로 제어)과, 빌드 디렉터리에 빌드 설정 해시를 일관되게 적용해 불필요한 재빌드와 캐시 충돌을 줄이는 변경도 포함했다. 후자는 내부 변경이지만, 과거의 불완전한 동작에 대한 워크어라운드를 쓰고 있던 빌드 스크립트나 테스트는 깨질 수 있으니 이상 동작 시 cargo 이슈 #17287을 참고하자.

## 업그레이드 체크리스트

지금까지의 내용을 실무 관점으로 압축하면 다음과 같다:

1. **에디션 확인** — 2015/2018/2021 에디션 프로젝트는 never type fallback 변경(`()` → `!`)으로 타입 추론·도달성 판정이 미세하게 달라질 수 있다.
2. **의존성 빌드 실패 시** — never type 관련 에러라면 크레이트 업데이트부터. 수정본이 이미 나와 있을 가능성이 높다.
3. **빌드 디렉터리 워크어라운드 점검** — 빌드 스크립트가 내부 산출물 경로를 참조한다면 1.100의 레이아웃 변경과 충돌할 수 있다.
4. **쿨다운 설정 도입** — `.cargo/config.toml`에 `global-min-publish-age = "3 days"` 한 줄로 시작할 수 있다. CI에서는 `CARGO_RESOLVER_INCOMPATIBLE_PUBLISH_AGE=allow`로 긴급 상향 경로도 미리 정해두자.
5. **DropGuard로 롤백 보일러플레이트 교체** — 수제 가드 타입을 쓰고 있었다면 표준 타입으로 통일할 시점이다.

## 다음 숙제: 2026 로드맵이 예고하는 것

1.100의 변화는 우연이 아니라 로드맵의 결과다. 공식 프로젝트 골스 사이트가 정리한 2026 로드맵을 보면 방향이 읽힌다:

- **Allocators 1.0** — 이번에 안정화된 `Allocator` 트레이트 코어가 출발점. 커스텀 할당자를 더 많은 컬렉션에서 쓸 수 있게 확장된다.
- **Fast Builds** — 클린 빌드와 증분 빌드 모두의 컴파일 속도 자체를 공략한다. 이번 Cargo 캐시 구조 정리도 그 기반 작업이다.
- **Just add async** — 트레이트·클로저·드롭·스코프 태스크가 async에서도 동기와 같은 패턴으로 동작하게 만든다.
- **Project Zero** — 알려진 타입 시스템 불건전성(unsoundness)을 전부 수정해 안전성 보장을 문자 그대로 신뢰할 수 있게 만든다.
- **Rust for Linux** — 안정 기능만으로 리눅스 커널을 빌드하는 것을 목표로 한다.

never type이 10년 걸린 숙제였듯이, 이 로드맵들도 수년 단위의 작업다. 다만 1.100에서 확인되는 것은 그 숙제들이 실제로 결실을 맺는 속도가 빨라지고 있다는 점이다.

## 결론

- Rust 1.100.0(2026-11-12)은 never type을 10년 만에 안정화했고, 그 대가로 구 에디션에 조용한 breaking change가 들어갔다.
- `core::mem::DropGuard`는 실패 경로 롤백 코드를 표준화했고, `conjure_zst`는 unsafe ZST 생성의 관용구를 통일했다.
- Cargo의 `min-publish-age`는 "게시 후 초기 기간이 가장 위험하다"는 공급망 공격의 실제 패턴을 활용하는 실용적 방어다. 설정 한 줄로 도입할 수 있다.
- 업그레이드 시 확인할 것은 세 가지: 구 에디션의 타입 추론 변화, 의존성 크레이트 업데이트, 빌드 디렉터리 레이아웃 변경.
- 당장 할 일: `rustup update stable` 후 `.cargo/config.toml`에 의존성 쿨다운을 켜보자. 가장 저렴한 보안 투자다.
