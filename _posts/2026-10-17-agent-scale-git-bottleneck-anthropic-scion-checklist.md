---
title: "하루 2,500만 번의 체크아웃이 Git을 무너뜨렸다 — Anthropic Scion 사례로 보는 에이전트 시대의 코드 인프라"
date: "2026-10-17"
keywords: ["Anthropic Scion", "버전 관리", "AI 코딩 에이전트", "GitHub Actions lockfile", "Agentic Autofix", "Git 병목"]
lang: "ko"
description: "Anthropic이 CI 체크아웃 하루 2,500만 회로 Git을 버리고 자체 버전 관리 시스템 Scion을 만든 사례와, GitHub의 lockfile·Agentic Autofix 발표에서 일반 팀이 지금 당장 챙겨야 할 에이전트 시대 코드 인프라 전략을 정리한다."
---

# 하루 2,500만 번의 체크아웃이 Git을 무너뜨렸다 — Anthropic Scion 사례로 보는 에이전트 시대의 코드 인프라

2026년 10월 13일, Anthropic은 엔지니어링 블로그에 [Redesigning source control](https://www.anthropic.com/engineering/redesigning-source-control)이라는 글을 올렸다. 요약하면 이렇다. **"에이전트가 쓰는 코드의 양이 늘면서 소스 컨트롤이 개발의 가장 큰 병목이 되었고, 우리는 Git을 버리고 5개월 만에 자체 버전 관리 시스템으로 완전 이전했다."** 코딩 에이전트의 성능 논쟁이 모델 벤치마크에만 머물러 있다면, 이 글이 보여주는 것은 그 다음 단계의 문제다. 에이전트가 코드를 잘 쓰게 되자, 이번에는 **그 코드를 저장하고, 검증하고, 병합하는 인프라**가 무너지기 시작했다.

이 글에서는 Anthropic이 공개한 구체적 수치와 설계를 짚고, 같은 주에 GitHub이 발표한 lockfile과 Agentic Autofix까지 함께 읽어서, 우리 같은 일반 팀이 에이전트 시대에 무엇을 준비해야 하는지 정리한다.

## Git이 어디서 무너졌나: Anthropic이 공개한 숫자

Anthropic 엔지니어링 블로그가 밝힌 모노레포 규모는 이렇다.

- 메인 브랜치 히스토리만 **약 100 GB**, 압축된 working copy가 **25 GB 이상**
- CI만 하루 **약 2,500만 번**의 코드 체크아웃을 수행 (개발 환경, 에이전트 샌드박스, 리서치 잡은 별도)
- 이걸 순진하게(naive) fetch하면 하루 **600 PB 이상**의 Git 트래픽이 발생
- 2026년 1월~9월 사이 **하루 병합량이 10배 이상** 증가

Git이 무너진 지점은 용량 그 자체보다 **설계 가정**이다. Git은 "클라이언트가 오프라인으로 작업하고 리포지토리 전체 복사본을 갖는다"는 전제 위에 서 있다. 그런데 Anthropic의 클라이언트는 이제 거의 다 에이전트와 CI 잡이다. 이들은 오프라인 작업을 하지 않고, 리포지토리 전체가 필요 없고, 짧은 수명의 태스크를 초 단위로 반복한다. 수천 개의 에이전트가 같은 브랜치에서 몇 개의 파일만 고치고 몇 초 만에 랜딩하려 하는 순간, 전체 클론과 글로벌 머지 큐는 그 자체가 병목이 된다.

## Scion은 뭘 바꿨나: 서버 중심 설계의 핵심

Scion의 설계를 블로그에서 발췌해 정리하면 다음과 같다.

**1. 클라이언트-서버 비대칭.** Git에서는 모든 클라이언트가 잠재적 서버다. Scion에서는 서버가 파일 히스토리와 blame을 답하고, 커밋을 만들고, 머지와 리베이스를 수행한다. 클라이언트는 필요할 때 필요한 것만 fetch한다. "거의 모든 클라이언트가 에이전트와 잡"이므로 서버가 항상 reachable하다는 가정이 오히려 자연스럽다.

**2. Changeset = 쓰기 선행 로그(write-ahead log) 항목.** Git 커밋이 트리 전체에 대한 포인터를 저장한다면, Scion의 changeset은 **첫 번째 부모 기준으로 변경된 파일 목록**과 콘텐츠 어드레스 저장소 참조만 담는다. 브랜치 헤드 이동은 Spanner 한 행의 compare-and-swap으로 처리된다.

**3. 머지 큐를 포기하고 병합 후 테스트로.** 이것이 가장 흥미로운 결정이다. "수백~수천 개의 변경이 테스트가 돌아가는 동안 병합될 수 있다면 머지 큐가 무슨 의미인가"라는 질문에 대해, Anthropic은 업계 표준인 글로벌 리포지토리 락을 에이전트 규모에서 재현하는 것을 포기했다. 대신 **병합 후·배포 전 자동 테스트**에 투자했다. 동시 변경이 충돌하면? "Claude가 인간이 충돌을 인지하기도 전에 수정한다"고 적혀 있다.

**4. 결과.** CI와 리서치 잡의 체크아웃이 분 단위에서 **초 단위**로 빨라졌고, CI만 **25% CPU 용량**을 절감했다. 기존 Git 미러링 인프라 대비 CPU/메모리 요구량은 **200배 작다**.

**5. 사람 7명, 5개월.** Scion과 코드 리뷰 도구 Antique, 그리고 관련 시스템 반 조각을 7명이 만들어 5개월 만에 완전 이전했다. 블로그는 "이런 빠르고 민첩한 이전이 앞으로 더 흔해질 것"이라고 말한다. 또 컴포넌트별로 Claude를 "superfan"으로 배치해 텔레메트리를 지켜보게 하는 운영 방식도 소개했다.

주의할 점: Scion은 공개되지 않았다. Anthropic은 GitHub, GitLab, 오픈소스 유지자들과 배운 것을 공유하겠다고만 했다. 커뮤니티에서 공개된 설계를 바탕으로 한 재구현 시도(예: open_scion)가 하루 만에 스타 1,000개를 모았다는 소식이 있으나, 이들은 검증 초기 단계이므로 프로덕션 도입 검토 대상으로 보긴 이르다.

## 같은 주, GitHub은 다른 방향에서 같은 문제를 팠다

Git 자체를 교체하는 대신, GitHub은 **에이전트가 만드는 코드의 신뢰성** 쪽을 파고 있다. 2026년 10월 중순 GitHub Changelog의 발표 두 건이 그 증거다.

**GitHub Actions 워크플로우 lockfile (10월 14일, public preview).** `.github/workflows/actions.lock` 파일이 워크플로우의 모든 액션과 전이적 의존성을 특정 커밋 SHA로 고정한다. 워크플로우에는 `actions/checkout@v6` 같은 읽기 쉬운 레퍼런스를 그대로 쓰되, 러너에서는 lockfile에 기록된 정확한 커밋이 실행된다. 액션 태그가 밀리거나 계정이 탈취돼도 고정된 코드가 계속 돈다. `gh actions-lock` CLI 확장으로 생성하며, "Require lockfile" 정책도 함께 나왔다.

```yaml
# .github/workflows/actions.lock (gh actions-lock이 자동 생성)
version: 'v0.0.2'
workflows:
    '.github/workflows/ci.yml':
        - 'octo-org/deploy-action@v3.1.0'
dependencies:
    'octo-org/deploy-action@v3.1.0':
        ref: 'v3.1.0'
        commit: 'sha1-...'
        owner_id: 44036562
        repo_id: 197814629
```

**Agentic Autofix GA (10월 15일).** 코드 스캐닝 얼러트를 Copilot에 어사인하면, 에이전트가 관련 파일을 돌아보고 → 수정안을 만들고 → **CodeQL을 다시 돌려** 얼러트가 닫히는지 검증하고 → 필요하면 반복한 뒤 드래프트 PR을 연다. 최대 25개 얼러트를 한 PR에서 처리할 수 있다. 단, 디폴트 브랜치 얼러트만 대상이고 Enterprise Cloud의 Code Security/Advanced Security 고객 전용이며 AI 크레딧을 소모한다. 같은 날 GA된 **AI Scan for PR**은 CodeQL이 커버하지 않는 PHP, Bash, Terraform, Dockerfile 등을 검사한다(단, 자문(advisory)용이고 머지를 막을 수는 없다).

이 두 발표를 Scion과 나란히 놓으면 하나의 그림이 나온다. **쓰는 쪽(에이전트)의 속도가 폭증하자, 저장(Scion)과 검증(lockfile, Autofix) 인프라가 뒤따라 재설계되고 있다.**

## 일반 팀을 위한 실전 체크리스트

우리는 Anthropic이 아니다. Git을 버릴 일도, 800B 파라미터 모델을 돌릴 일도 없다. 그러나 에이전트 트래픽이 기하급수적으로 늘어난다는 **패턴**은 똑같이 적용된다. 지금 할 수 있는 것들:

**1. 체크아웃 비용부터 측정하라.** CI 잡 하나가 몇 번 클론하고, 각각 몇 분 걸리는지부터 세라. 그리고 즉시 적용할 수 있는 것:

```bash
# 얕은 클론 + 일부 경로만 체크아웃
git clone --filter=blob:none --no-checkout https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout set services/api libs/core
git checkout main
```

blob 없이 트리만 받는 partial clone(`--filter=blob:none`)과 sparse-checkout 조합만으로도 대형 리포의 체크아웃 비용이 극적으로 줄어든다. 이것이 Anthropic이 말한 "Git이 리포의 일부분만 다루는 도구를 제공한다"의 실전 버전이다.

**2. 병렬 에이전트에는 worktree를 써라.** 에이전트 여럿이 하나의 working copy를 두고 경쟁하면 충돌은 코드가 아니라 파일시스템에서 난다. 태스크마다 격리된 체크아웃을 주는 git worktree 패턴이 업계 표준으로 자리 잡고 있다:

```bash
git worktree add ../task-$(date +%s) -b agent/task-42
# 에이전트는 ../task-xxx 에서 작업, 완료 후 브랜치만 머지
git worktree list
```

**3. lockfile을 켜라.** 에이전트가 PR을 수정하는 워크플로우에서 서플라이체인은 어제보다 위험하다. GitHub Actions를 쓴다면 public preview라도 `gh actions-lock`으로 actions.lock을 만들고, 셀프호스티드 환경이라면 기존의 SHA 핀닝(`uses: actions/checkout@<sha>`)을 정책화하라.

**4. "병합 전 검증"을 "병합 후 빠른 롤백"과 섞어라.** Anthropic의 통찰은 머지 큐를 완벽하게 유지하는 게 불가능해지는 지점이 온다는 것이다. 그 전에 빠른 post-merge 테스트 + 원인 커밋 자동 리버트(bisect) 파이프라인을 만들어 두면, 검증 병목 없이 처리량을 늘릴 수 있다.

**5. 검증 자체도 에이전트에 위임하되, 결정권은 남겨라.** GitHub의 Agentic Autofix가 좋은 설계인 이유는 "CodeQL 재실행이라는 결정적 검증 + 드래프트 PR이라는 인간 게이트"를 조합했기 때문이다. 도구가 무엇이든 이 구조(자율 탐색 → 결정적 검증 → 인간 승인)를 베끼면 된다.

## 결론

- Anthropic의 하루 2,500만 회 CI 체크아웃, 10배 늘어난 병합량이 Git의 근본 가정을 무너뜨렸고, 7명이 5개월 만에 Scion으로 이전했다(체크아웃 분→초, CI CPU 25% 절감).
- Scion의 핵심 설계는 서버 중심 changeset 로그, 그리고 **머지 큐 포기와 병합 후 테스트**라는 과감한 트레이드오프다.
- GitHub은 같은 주에 lockfile(actions.lock)과 Agentic Autofix로 "에이전트가 만든 코드의 신뢰성" 문제에 답했다.
- 일반 팀이 당장 할 일: 체크아웃 비용 측정 → partial clone/sparse-checkout → worktree로 에이전트 격리 → actions.lock → post-merge 자동 롤백 파이프라인.

오늘 당장 시도할 첫 단계는 하나면 충분하다. **가장 느린 CI 잡의 체크아웃 시간을 재고, `--filter=blob:none` 클론으로 바꿔서 얼마나 줄어드는지 확인하는 것.** 에이전트 시대의 병목은 모델이 아니라 이런 지루한 지점에서 먼저 온다.

**참고 자료**
- [Redesigning source control at Anthropic (Anthropic Engineering, 2026-10-13)](https://www.anthropic.com/engineering/redesigning-source-control)
- [GitHub Actions workflow lockfiles in public preview (GitHub Changelog, 2026-10-14)](https://github.blog/changelog/2026-10-14-github-actions-workflow-lockfiles-in-public-preview/)
- [Agentic autofix for code scanning alerts general availability (GitHub Changelog, 2026-10-15)](https://github.blog/changelog/2026-10-15-agentic-autofix-for-code-scanning-alerts-general-availability)
- [Workflow dependency lockfiles — GitHub Docs](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)
