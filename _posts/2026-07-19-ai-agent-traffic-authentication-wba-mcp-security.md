---
title: "AI 에이전트 트래픽을 인증해야 하는 이유: WBA와 MCP 보안"
date: "2026-07-19"
keywords: ["AI 에이전트", "AWS WAF", "Web Bot Authentication", "MCP", "prompt injection"]
lang: "ko"
description: "Microsoft와 AWS 최신 자료를 바탕으로, AI 에이전트 트래픽 인증과 MCP 도구 보안을 어떻게 분리해야 하는지 실전 패턴과 코드 예시로 정리합니다."
---

# AI 에이전트 트래픽을 인증해야 하는 이유: WBA와 MCP 보안

AI 에이전트 보안은 이제 모델 답변의 품질만 보는 단계가 아닙니다. 최근 Microsoft는 에이전트가 단순히 콘텐츠를 읽는 수준을 넘어 실제로 행동을 수행하는 단계로 이동하고 있다고 설명했고, AWS는 AI 에이전트와 자동화 도구의 웹 트래픽을 **cryptographic signature**로 검증하는 Web Bot Authentication(WBA)을 공개했습니다. 흐름은 분명합니다. 앞으로 중요한 질문은 “이 모델이 똑똑한가?”가 아니라 “이 요청이 정말 우리가 허용한 에이전트에서 왔는가?”입니다.

이 변화가 중요한 이유는 공격면이 바뀌었기 때문입니다. 예전의 프롬프트 인젝션은 대개 잘못된 요약이나 왜곡된 텍스트를 만들었습니다. 하지만 에이전트가 MCP 도구를 호출하고, 메일을 보내고, 데이터를 조회하고, 외부 시스템을 건드리기 시작하면 같은 공격이 곧 **행동 오용**으로 바뀝니다. OWASP도 Prompt Injection, Supply Chain Vulnerabilities, Excessive Agency를 별도 위험으로 다루고 있습니다. 즉, 이제는 “출력 검증”만으로는 부족하고 **트래픽 인증, 도구 공급망 검증, 승인 경계**를 함께 설계해야 합니다.

## 1. 먼저 구분해야 할 두 개의 경계

첫 번째는 **요청이 누구의 것인지**입니다. AWS WAF Bot Control의 WBA는 검증된 봇과 에이전트 트래픽을 cryptographic signature로 확인합니다. AWS 문서와 공지에 따르면 WBA는 HTTP 메시지에 서명된 식별 정보를 사용하고, 검증 결과를 바탕으로 `verified`, `invalid`, `expired`, `unknown_bot` 같은 레이블을 붙입니다. 검증된 트래픽은 기본적으로 허용됩니다.

두 번째는 **도구가 무엇을 하게 만드는지**입니다. Microsoft 보안 블로그는 MCP 도구 설명이 변조되면 에이전트의 행동 자체가 바뀔 수 있다고 지적합니다. 다시 말해, 도구 이름만 믿으면 안 되고 도구 메타데이터와 변경 이력까지 보안 범위에 넣어야 합니다.

이 두 경계를 합치면 설계 원칙은 단순해집니다.

- 외부에서 들어오는 에이전트 요청은 서명으로 검증합니다.
- 내부에서 호출되는 도구는 allowlist로 제한합니다.
- 고위험 행동은 사람 승인으로 넘깁니다.
- 도구 설명, 버전, 변경 이력을 감사 로그로 남깁니다.

## 2. 실전 정책 예시

아래는 에이전트 권한을 최소화하는 간단한 정책 예시입니다.

```yaml
agent:
  name: support-orchestrator
  allowed_tools:
    - read_ticket
    - draft_reply
    - lookup_knowledge_base
  approval_required:
    - send_email
    - change_permissions
    - export_all_data
    - trigger_payment
    - deploy_service
  audit:
    - request_id
    - tool_name
    - tool_version
    - tool_description_hash
    - approver
```

핵심은 “무엇을 허용할지”보다 “무엇을 자동으로 하면 안 되는지”를 먼저 적는 것입니다. 에이전트가 똑똑해질수록 금지 목록은 더 중요해집니다.

```python
HIGH_RISK = {
    "send_email",
    "change_permissions",
    "export_all_data",
    "trigger_payment",
    "deploy_service",
}

def route_action(action: str, payload: dict) -> dict:
    if action in HIGH_RISK:
        return {
            "status": "PENDING_APPROVAL",
            "action": action,
            "payload": payload,
        }

    return {
        "status": "ALLOW",
        "action": action,
        "payload": payload,
    }
```

이 패턴은 단순하지만 효과가 큽니다. 에이전트가 자동으로 할 수 있는 일과 사람이 멈춰 세워야 하는 일을 분리해 주기 때문입니다.

## 3. WAF에서 먼저 막고, 애플리케이션에서 한 번 더 막기

에지에서 검증하고 애플리케이션에서 재검증하는 구조가 좋습니다. AWS 쪽에서는 WBA 레이블을 기반으로 다음처럼 정책을 잡는 식이 자연스럽습니다.

```yaml
rules:
  - name: allow_verified_agents
    match_label: "awswaf:managed:aws:bot-control:bot:web_bot_auth:verified"
    action: allow

  - name: block_invalid_agents
    match_label: "awswaf:managed:aws:bot-control:bot:web_bot_auth:invalid"
    action: block

  - name: alert_on_expired_keys
    match_label: "awswaf:managed:aws:bot-control:bot:web_bot_auth:expired"
    action: count
```

이렇게 하면 “정상 에이전트 트래픽”과 “의심스러운 트래픽”을 먼저 가를 수 있습니다. 그 다음 애플리케이션에서는 MCP 서버의 변경 이력, tool metadata, 승인 기록을 따로 확인하면 됩니다.

## 4. 지금 바로 적용할 체크리스트

- 에이전트 트래픽에 대해 신원 검증 레이어가 있는가
- MCP 서버와 tool description 변경이 승인 대상인가
- 고위험 action은 인간 승인으로 빠지는가
- tool call과 승인 로그를 분리 저장하는가
- verified / invalid / expired 같은 상태를 모니터링하는가

## 결론

AI 에이전트의 보안은 더 이상 “모델을 잘 고르는 문제”가 아닙니다. 요청의 출처를 증명하고, 도구 공급망을 통제하고, 행동 경계를 분명히 하는 문제입니다. Microsoft가 지적한 MCP 도구 오염과 AWS가 제시한 WBA는 같은 방향을 가리킵니다. **읽는 에이전트에서, 행동하는 에이전트로 넘어갈수록 인증과 승인, 감사는 선택이 아니라 기본값**이어야 합니다.

오늘의 첫 단계는 간단합니다. 여러분의 에이전트 정책에서 `send_email`, `change_permissions`, `export_all_data` 세 가지를 먼저 자동 실행 목록에서 빼 보십시오. 그 한 줄이 사고를 막는 가장 싼 방어선이 될 수 있습니다.
