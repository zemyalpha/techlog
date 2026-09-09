---
title: "OpenTelemetry GenAI 스펙은 아직 전부 Development다: 대시보드가 까맣게 변하는 마이그레이션 함정 피하기"
date: "2026-09-09"
keywords: ["OpenTelemetry", "GenAI", "observability", "LLM tracing", "에이전트 관측가능성", "OTLP"]
lang: "ko"
description: "2026년 6월 v1.42.0에서 gen_ai 네임스페이스 전체가 전용 리포로 이동하며 63개 속성 전부 Development 상태가 된 OpenTelemetry GenAI 컨벤션의 현주소와, 계측을 안전하게 설계하는 실전 가이드."
---

# OpenTelemetry GenAI 스펙은 아직 전부 Development다: 대시보드가 까맣게 변하는 함정 피하기

LLM 애플리케이션에 관측가능성(observability)을 붙이려는 팀 대부분이 첫 번째 질문으로 "OpenTelemetry 쓰면 되죠?"라고 묻는다. 맞다. 그런데 두 번째 질문인 "그럼 스키마를 뭘로 고정하지?"에서부터 문제가 시작된다. 2026년 9월 현재 OpenTelemetry의 GenAI 시맨틱 컨벤션은 **안정(Stable) 상태가 단 하나도 없다.** 레지스트리 집계에 따르면 63개의 `gen_ai.*` 속성 키 전부가 Development 등급이다.

이건 단순한 사실 관계가 아니라 실무에서 다이어그램보다 먼저 부딪히는 문제다. 계측 라이브러리를 업그레이드했더니 어제까지 잘 나오던 토큰 사용량 대시보드가 하루아침에 비어버리는 일이 실제로 벌어지고 있고, 원인은 대부분 스펙 개정에 따른 속성 이름 변경이다. 이 글에서는 현재 스펙의 정확한 상태, 속성 이름이 어떤 주기로 바뀌어 왔는지, 그리고 다음 릴리스에도 깨지지 않는 계측 설계법을 정리한다.

## 스펙의 현주소: 전용 리포로 이동했지만 버전 태그가 없다

2026년 6월 12일, OpenTelemetry 메인 시맨틱 컨벤션 리포의 v1.42.0 릴리스는 모든 `gen_ai.*` 속성·메트릭·이벤트·스팬을 **deprecated 처리하고 전용 리포로 통째로 옮겼다.** 이후 v1.43.0(7월), v1.44.0(8월)에는 GenAI 관련 내용이 더 이상 포함되지 않는다. 이 사실은 GitHub 릴리스 노트에서 직접 확인할 수 있다.

새 거처는 `open-telemetry/semantic-conventions-genai` 리포인데, 여기서 실무자를 당황시키는 지점이 두 가지 있다.

1. **태그된 릴리스가 아직 없다.** 스키마를 특정 버전으로 핀(pinning)하려 해도 핀할 대상이 존재하지 않는다. 커밋 해시나 날짜 스냅샷을 기준으로 삼을 수 있을 뿐이다.
2. **README의 Schema URL 섹션이 아직 TODO다.** 리포를 직접 들어가 보면 확인할 수 있다.

독립적으로 이 상태를 검증한 분석도 최소 두 곳 이상에서 나왔다. 2026년 8월 13일 기준 GenAI 레지스트리의 63개 `gen_ai` 속성 키 전부가 Development이며, 이넘(enum) 멤버 선언까지 합친 109개 안정성 선언 전부가 development로 표기되어 있다는 것이다. 별도의 7월 분석도 "GenAI 고유 스팬·이벤트·메트릭·속성 중 Stable로 표기된 것은 없다"고 결론짓는다. 스펙 문서 본문 첫 줄의 Status도 Development다. 공유 핵심 속성인 `error.type`이나 `server.address` 등은 Stable이지만, GenAI 고유 표면은 하나도 아니다.

## 속성 이름은 계속 바뀌어 왔다: 개정 연대기

"Development라도 일단 쓰면 되지 않나?"라고 넘어가면 안 되는 이유는 과거 개정 이력이 보여준다. 최근 2년간 주요 변경만 추려도 이렇다.

| 릴리스 | 변경 | 실무 영향 |
|---------|------|-----------|
| v1.27.0 (2024-08) | `gen_ai.usage.prompt_tokens` / `completion_tokens` → `input_tokens` / `output_tokens` | 토큰 대시보드가 이중 필드 쿼리 또는 마이그레이션 로직 필요 |
| v1.37.0 (2025-08) | `gen_ai.system` → `gen_ai.provider.name`, 메시지별 이벤트가 `gen_ai.input.messages` 등 속성으로 재편 | 구버전 프레임워크 텔레메트리와 신버전의 가장 큰 단절점 |
| v1.38.0 (2025-10) | `gen_ai.evaluation.result` 이벤트 추가 | 평가(evals) 결과를 트레이싱과 상관 관계로 연결 가능 |
| v1.40.0 (2026-02) | 리트리벌 스팬, 캐시 토큰 속성, `gen_ai.agent.version` 추가 | RAG·에이전트 텔레메트리 풍부해짐 |
| v1.41.0 (2026-04) | `invoke_agent`가 클라이언트/내부 스팬으로 분리, 스트리밍 지연 메트릭 추가 | 인프로세스 에이전트 프레임워크의 스팬 구조 변경 |
| v1.42.0 (2026-06) | GenAI 전체를 전용 리포로 이동·deprecated | 권위 있는 원천 자체가 바뀜 |

핵심은 이 중 어느 것도 "선택 사항"이 아니었다는 점이다. 계측 라이브러리를 업그레이드하면 따라온다. 특히 v1.37.0의 재편은 현재 시점에 구버전 속성을 뿜는 프레임워크와 신버전 컨벤션을 뿜는 프레임워크가 동시에 존재하는 원인이 되어, "OpenTelemetry를 쓴다"고 해도 서로 다른 세대의 속성이 섞여 들어오는 상황을 만들었다.

## 대시보드가 까매지는 진짜 원인: 패키지 스왑

실무에서 가장 자주 겪는 장애는 스펙 이동 자체가 아니라 **Python 계측 패키지 교체**에서 발생한다. 업계 분석 보고에 따르면, 구 패키지는 안정화 옵트인(`OTEL_SEMCONV_STABILITY_OPT_IN`)을 설정하지 않으면 v1.30.0 시대의 구 속성명을 뿜은 반면, 이를 대체하는 신 패키지(`opentelemetry-python-genai`, 현재 베타 버전으로 배포 중)는 옵트아웃 없이 최신 실험 컨벤션을 무조건 내보낸다. 결과적으로 패키지만 바꿔도 대시보드 쿼리가 바라보던 필드 이름이 통째로 바뀌어 차트가 빈 상태가 된다.

## 다음 릴리스에도 깨지지 않는 계측 설계

Development 스펙 위에서 안정적으로 운영하려면 설계 원칙이 필요하다. 실전에서 검증된 전략을 정리하면 다음과 같다.

**1. 비용·지연 대시보드는 스팬 속성이 아니라 메트릭에 세운다.** 클라이언트 메트릭은 속성 이름 변경의 영향권에서 상대적으로 안전하다. 속성명을 직접 참조하는 대시보드 패널일수록 다음 개정에 취약하다.

**2. 에이전트·툴 패널은 '임시'로 취급한다.** 이미 다섯 건의 브레이킹 체인지가 릴리스되지 않은 상태로 큐에 대기 중이라는 보고가 있다. 이 영역 스키마에 비즈니스 로직을 묶지 말 것.

**3. 프롬프트 본문은 스팬에 실어 보내지 않는다.** 규제 환경이라면 외부 업로드 훅을 써서 메시지 본문은 자체 스토리지에 남기고 스팬에는 참조만 실린다. 프롬프트 유출은 관측가능성 도입이 만드는 새로운 공격면이다.

**4. 수집기 경유로 버퍼를 둔다.** 앱 → OTel Collector → 백엔드 구조로 중간 다리를 두면, 속성명 변경 시 앱 재배포 없이 Collector의 속성 프로세서로 매핑을 흡수할 수 있다:

```yaml
# otel-collector-config.yaml (발췌)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  memory_limiter:
    limit_mib: 1500
  batch: {}
exporters:
  otlphttp:
    endpoint: https://your-langfuse-host/api/public/otel
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp]
```

이 구성의 엔드포인트(`/api/public/otel`)는 Langfuse 공식 문서가 안내하는 OTLP 수집 경로다. Langfuse는 코어가 MIT 라이선스인 오픈코어 프로젝트로(기업용 `ee/` 디렉터리만 별도 상업 라이선스), 자체 호스팅으로 프롬프트 데이터를 내부망에 두고 싶은 팀의 현실적 선택지다. 코드에서 트레이서를 초기화할 때는 스팬 속성을 하드코딩하기보다 SDK 상수를 참조하게 해서, 패키지 업그레이드 시 컴파일 시점에 변경을 감지할 수 있게 한다:

```python
from opentelemetry import trace
from opentelemetry.sdk._logs import LoggingHandler

# 속성 이름을 문자열 리터럴로 흩뿌리지 말고 한 곳에서 정의하라
ATTRS = {
    "model": "gen_ai.request.model",       # Development 등급
    "input_tokens": "gen_ai.usage.input_tokens",   # v1.27+ 이름
    "provider": "gen_ai.provider.name",    # v1.37+ 이름
}

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("chat_completion") as span:
    span.set_attribute(ATTRS["model"], "gpt-4o")
    span.set_attribute(ATTRS["provider"], "openai")
    # ... 호출 후 토큰 사용량 기록
```

## 요약: 지금 취해야 할 자세

- OpenTelemetry GenAI 컨벤션의 63개 속성 전부가 Development이며, 2026년 6월 v1.42.0부터 전용 리포에서 관리된다. 단 그 리포에는 아직 버전 태그가 없다.
- 대시보드 장애의 주범은 스펙 이동이 아니라 계측 패키지 교체에 따른 속성명 변경이다. 비용·지연 지표는 메트릭 기반으로 세워라.
- 에이전트·툴 영역 스키마는 추가 브레이킹 체인지가 예고된 상태이므로 비즈니스 로직과 분리하라.
- 프롬프트 본문은 외부로 실어 보내지 말고, 자체 호스팅 백엔드 + OTel Collector 구조로 데이터 주도권을 가져라.

첫 단계는 간단하다. 지금 자신의 대시보드 쿼리를 열어 `gen_ai.usage.prompt_tokens` 같은 v1.27 이전 속성명을 하드코딩한 패널이 있는지 확인하는 것부터 시작하면 된다. 다음 패키지 업그레이드 전에 미리 고쳐둘 수 있는 유일한 곳이 바로 그곳이다.
