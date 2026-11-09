---
title: "Claude API Message Threads 실전 가이드 — 긴 에이전트 세션의 첫 토큰 대기 35% 줄이기"
date: "2026-11-09"
keywords: ["Anthropic", "Message Threads", "Claude API", "AI 에이전트", "TTFT", "프롬프트 캐싱"]
lang: "ko"
description: "2026년 11월 2일 공개 베타된 Anthropic Message Threads의 동작 원리, Python SDK와 cURL 실전 코드, 5분 만료와 재생(replay) 처리, 캐싱 제약까지 정리한다."
---

# Claude API Message Threads 실전 가이드 — 긴 에이전트 세션의 첫 토큰 대기 35% 줄이기

코딩 에이전트를 직접 만들어 본 사람이라면 누구나 이 문제를 겪는다. 에이전트 루프 한 세션에서 수백 번의 모델 호출이 일어나는데, 매 요청마다 지금까지의 대화 전체 — 시스템 프롬프트, 툴 정의, 모든 툴 실행 결과 — 를 다시 업로드해야 한다. 대화가 수십만 토큰으로 불어난 시점에는 모델이 첫 토큰을 뱉기도 전에 전체 히스토리를 검증하고 토크나이즈하는 데 드는 시간이 체감될 정도로 길어진다.

Anthropic이 2026년 11월 2일 Messages API에 공개 베타로 내놓은 **Message Threads**는 이 낭비를 구조적으로 없앤다. 전체 대화를 한 번만 보내고, 이후에는 새 메시지와 직전 응답의 id만 보내면 서버가 대화를 재구성해서 모델을 호출한다. Anthropic 발표 기준으로 50만~100만 토큰 요청에서 **첫 토큰까지의 대기 시간(TTFT) 중앙값이 약 35% 줄었고**, 이미 Claude Code 사용자 대부분에게 적용되어 있다고 한다. 토큰 과금과 레이트 리밋은 그대로다 — 시간을 줄이는 기능이지 비용을 줄이는 기능은 아니다.

이 글은 공식 문서를 기준으로 실전 도입 시 반드시 알아야 할 설계 규칙과 함정을 정리한다.

## 동작 원리: create, continue, replay

Message Threads의 전체 메커니즘은 세 단어로 요약된다.

1. **Create** — 첫 요청은 평소처럼 전체 대화와 `system`, `tools`를 담아 보내되 `thread: {"type": "create"}`만 추가한다. API가 Claude의 응답까지 포함해 대화를 응답 id 아래 저장한다.
2. **Continue** — 이후 요청에는 직전 응답 이후의 새 메시지만 담고 `thread: {"type": "continue", "previous_message_id": "<직전 응답 id>"}`를 붙인다. `system`과 `tools`는 **보내면 안 된다**(null로 보내도 400 에러). 스레드가 create 시점의 값을 유지한다.
3. **Replay** — 스레드는 베스트에포트(best-effort)다. 마지막 요청 후 **5분이 지나면** 만료되고, 마지막 사용 후 최대 1시간 안에 삭제된다. continue 요청이 404(`thread_not_found`)를 반환하면 전체 대화를 새 create 요청으로 다시 보내면 된다.

여기서 중요한 설계 원칙이 나온다. **대화의 원본 출처(source of truth)는 여전히 내 애플리케이션이다.** API는 create든 continue든 "이 턴을 저장했다"고 알려주지 않는다. 저장 여부는 다음 continue 요청이 실패해야 알 수 있다. 그러므로 모든 메시지의 사본을 클라이언트가 따로 유지하고, 처음부터 재생 경로를 만들어 둬야 한다.

## 실전 코드

### Python SDK — 스레드 헬퍼

SDK에는 재생 처리까지 알아서 해주는 헬퍼가 들어 있다. create/continue 구분, 베타 헤더 추가, 404 시 재생을 전부 대신한다.

```python
import anthropic

client = anthropic.Anthropic()

# 헬퍼 생성 — 이 시점에는 요청이 나가지 않는다
thread = client.beta.messages.thread(
    model="claude-opus-5-5",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # 턴마다 프롬프트 캐싱 (선택)
    system="You are a support agent for Acme. Keep replies short.",
)

# 턴 1: 헬퍼가 스레드를 create하고 전체 대화를 보낸다
thread.send({"role": "user", "content": "Hi! I have a question about an order."})

# 턴 2: 새 메시지만 보낸다. continue는 헬퍼가 처리
second = thread.send(
    {"role": "user", "content": "How long does standard shipping usually take?"}
)
for block in second.content:
    if block.type == "text":
        print(block.text)
```

### cURL — thread 필드 직접 구성

헬퍼 없이 직접 보내려면 모든 요청에 `message-threads-2026-08-12` 베타 헤더를 붙인다.

```bash
# 턴 1: 전체 대화 + create
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2026-10-21" \
  -H "anthropic-beta: message-threads-2026-08-12" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 1024,
    "system": "You are a support agent for Acme.",
    "messages": [{"role": "user", "content": "Hi!"}],
    "thread": {"type": "create"}
  }'

# 턴 2: 새 메시지 + 직전 응답 id만. system/tools는 생략
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2026-10-21" \
  -H "anthropic-beta: message-threads-2026-08-12" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "How long does shipping take?"}],
    "thread": {"type": "continue", "previous_message_id": "msg_01ABC..."}
  }'
```

## 필드 규칙 — 여기서 400 에러가 가장 많이 난다

공식 문서의 필드 분류를 그대로 옮기면 다음과 같다.

| 분류 | 필드 | 규칙 |
|------|------|------|
| create 전용 | `system`, `tools` | create에만 포함. continue에 null로라도 보내면 400 |
| 고정 | `model`, `tool_choice`, `thinking`, `mcp_servers`(토큰 제외), 최상위 `cache_control` 등 | 모든 요청에 동일 값으로 포함. 다르면 `thread_fingerprint_mismatch` 400 |
| 자유 | `max_tokens`, `stop_sequences`, `stream`, `metadata` 등 | 턴마다 변경 가능 |

주의할 예외가 하나 있다. MCP 서버 URL과 인증 토큰, 컨테이너 id는 API가 저장하지 않으므로 **사용하는 모든 요청에 다시 보내야** 한다. 인증 토큰은 요청 사이에 바뀔 수 있어서 의도된 설계다.

## 도입 전 반드시 알아야 할 함정

**1. 5분 만료와 무음 실패.** 사용자 입력을 기다리는 챗봇처럼 턴 간격이 5분을 넘는 서비스라면 거의 매 턴이 재생(replay)이 된다. 재생은 "언제나 안전하다"는 게 공식 문서의 입장이지만, 전체 대화를 다시 보내므로 요청 크기 이점은 사라진다. 턴 간격이 짧은 자율 에이전트 루프에 최적화된 기능이다.

**2. 캐시 브레이크포인트는 최대 4개.** 스레드 요청에 `cache_control`을 4개 초과 붙이면 `thread_unsupported_request` 400 에러가 난다. 최상위 `cache_control`이 1개를 차지하고, create 시점의 `system`/`tools`에 붙은 브레이크포인트는 이후 모든 continue에서 다시 카운트된다. 스레드가 프롬프트 캐싱을 대신 해주는 것은 아니며, 캐싱은 평소처럼 브레이크포인트 배치로 직접 제어한다.

**3. append-only — 편집·분기·요약 압축 불가.** 각 응답 id는 한 번만 continue할 수 있고, 항상 최신 응답 id에서 이어야 한다. 이전 메시지를 편집하거나 분기하려면 새 스레드를 만들어야 한다. 컨텍스트 압축(compaction)도 스레드와 함께 쓸 수 없다 — 요약으로 대화를 줄이려면 요약본을 첫 메시지로 하는 새 create 요청을 보내야 한다.

**4. pause_turn 은 빈 메시지로 재개.** 응답이 `stop_reason: "pause_turn"`으로 끝나면 그 응답 id에서 `messages: []`로 continue한다. 404가 뜨면 일반 재생 경로로.

**5. 스트리밍 중단 시 재시도 판단.** continue 스트림이 `message_stop` 전에 끊기면 같은 요청을 다시 보낼 수 있지만, `thread_already_continued`나 `thread_continue_in_flight` 에러가 뜨면 전체 재생만이 안전한 경로다. 응답 id는 `message_start`에 오지만, continue는 `message_stop`을 받은 뒤에만 시도해야 한다.

**6. 규제·보안 제약.** 스레드는 메모리에 서버 관리 키로 암호화되어 보관되고 디스크에는 기록되지 않지만, **ZDR(제로 데이터 리텐션) 및 HIPAA 대상이 아니다**. HIPAA 준비를 켠 조직, Inference hooks를 켠 조직은 아예 사용할 수 없다. Message Batches, 토큰 카운팅, 컨텍스트 에디팅도 미지원. 스레드를 조회·삭제하는 API는 존재하지 않는다.

**7. 서버사이드 폴백과의 상호작용.** 폴백 모델이 쓴 응답은 스레드에 저장되지 않는다. `usage.iterations`의 `fallback_message` 항목을 매 턴 확인해서, 폴백 응답이 있으면 전체 대화를 새 create로 보내야 한다. 폴백은 모든 요청에 넣거나 모든 요청에서 빼야 한다(추가·제거 시 지문 불일치 400).

## 같은 주간의 흐름 — 에이전트 인프라의 시대

Message Threads가 나온 같은 주(11월 1~7일)에 세 개 대형 랩이 내놓은 발표에는 공통점이 있다. 범용 플래그십 모델은 없었고, **모델 주변의 배관(plumbing)**만 있었다. Anthropic은 11월 6일 에이전트 메모리의 4계층 구조(세션 기록, 모델이 작성한 메모, 위키 같은 연결 지식, 세션 시작 시 로드되는 컨텍스트 블록)를 다루는 엔지니어링 포스팅을 공개했고, OpenAI는 11월 2일 모노레포용 Git 호환 백엔드 **Pando**를 발표했다. OpenAI 발표 기준으로 Pando는 브랜치 단위로 쓰기를 처리해 무관한 브랜치의 push가 서로를 기다리지 않게 만들었고, 중앙값 push 지연 시간을 약 5분의 1로 줄이며 하루 약 1PB의 Git 데이터를 처리하고 있다.

무거운 에이전트 워크로드가 늘면서 토큰 전송, Git push, 메모리 계층 같은 "지루한" 계층이 병목으로 떠오르고, 랩들이 그 병목을 직접 깨고 있는 것이다. Message Threads도 같은 맥락에서 봐야 한다 — 모델이 아니라 모델로 가는 길을 빠르게 만든 발표다.

## 결론

- 에이전트 루프에서 매턴 전체 히스토리 재전송은 TTFT 병목이며, Message Threads는 이를 "한 번 전송 + 증분" 구조로 바꾼다 (Anthropic 발표 기준 50만~100만 토큰 요청 TTFT 중앙값 약 35% 감소).
- 요금과 레이트 리밋은 변함없다. 절약은 시간이지 비용이 아니다 — 비용은 여전히 프롬프트 캐싱으로 관리한다.
- 도입의 핵심은 재생 경로다. 클라이언트가 항상 대화 사본을 들고 있어야 하며, 404를 만나면 전체 재생이 기본 동작이어야 한다.
- 캐시 브레이크포인트 4개 한도, append-only 제약, ZDR/HIPAA 비대상은 아키텍처 결정 전에 반드시 확인할 것.
- 턴 간격이 5분 이상인 대화형 서비스보다는, 연속 호출이 많은 자율 에이전트·코딩 에이전트가 1차 수혜자다.

당장 해볼 첫 단계는 간단하다. 기존 에이전트 루프의 모델 호출 부분을 `client.beta.messages.thread()` 헬퍼로 바꾸고, 긴 세션에서 TTFT가 실제로 얼마나 줄어드는지 직접 측정해 보는 것이다. 베타 기능이므로 헤더 하나(`anthropic-beta: message-threads-2026-08-12`)로 on/off가 가능하다.

**참고 자료**
- Anthropic, "Message threads" — Claude Platform Docs (platform.claude.com/docs/en/build-with-claude/message-threads)
- The Next Web, "Anthropic's message threads let Claude agents skip resending chats" (2026-11-02)
- OpenAI, "Scaling Git at OpenAI" (openai.com/index/scaling-git-at-openai/, 2026-11-02)
- The New Stack, "OpenAI built itself a new Git backend, too" (2026-11-02)
- Promptea, "AI weekly recap Nov 1–7" (2026-11-07)
