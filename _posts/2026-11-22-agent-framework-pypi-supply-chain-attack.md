---
title: "Microsoft agent-framework가 PyPI에서 해킹당한 45분 — .pth 훅 신원털이 공격 분석과 대응 체크리스트"
date: "2026-11-22"
keywords: ["공급망 공격", "PyPI", "agent-framework", "Microsoft", "credential stealer", "pth 파일"]
lang: "ko"
description: "2026년 11월 17일 Microsoft agent-framework PyPI 패키지 1.26.1/1.26.2에 신원털이 악성코드가 실렸다. .pth 시작 훅 기법, 훔치는 자격증명 목록, 그리고 개발자가 당장 실행해볼 대응 체크리스트를 정리했다."
---

# Microsoft agent-framework가 PyPI에서 해킹당한 45분 — .pth 훅 신원털이 공격 분석과 대응 체크리스트

2026년 11월 17일 밤(UTC 20:35~21:20), Microsoft의 AI 에이전트 SDK인 `agent-framework`가 PyPI에서 약 45분 동안 악성코드 배포 통로가 되었다. 오타 스쿼팅이나 사칭 패키지가 아니라, **Microsoft 계정과 연결된 공식 PyPI 퍼블리셔를 통해** 정식 버전 1.26.1과 1.26.2에 신원털이(credential stealer)가 실려 배포된 사건이다. 연구자들은 퍼블리셔 계정 침해로 보고 있으나, 정확한 게시 경위는 아직 공식 확인되지 않았다.

이 패키지는 월 약 19만 다운로드 규모이고, 이름 그대로 AI 에이전트를 만드는 개발자·CI 파이프라인이 주 사용층이다. 즉 공격자는 Azure CLI 로그인, GitHub Actions OIDC 토큰, OpenAI·Hugging Face API 키 같은 **AI 개발자의 환경을 정조준한 페이로드**를 심은 것이다. 왜 이런 공격이 가능했고, 어떻게 동작했으며, 지금 무엇을 점검해야 하는지 정리한다.

## 무슨 일이 일어났나 — 타임라인

Orca Security와 Chainguard의 분석을 교차 검증한 타임라인은 다음과 같다.

1. **20:35:20 UTC** — `agent-framework` 1.26.1이 Microsoft 계정으로부터 게시됨. `agent_framework_meta.pth`와 `_telemetry.py` 페이로드 포함.
2. **20:36:08 UTC** — Chainguard Libraries가 1.26.1을 차단 (게시 48초 후). Elastic도 게시 6분 만에 탐지했다고 밝혔다.
3. **PyPI가 1.26.1을 격리(quarantine)** 조치.
4. **21:08:39 UTC** — **격리 직후** 공격자가 1.26.2를 추가 게시. 이번엔 페이로드 파일명을 `af_runtime_diag.pth` / `af_runtime_diag.py`로 바꾸고, 정상 `__init__.py`를 바이트 단위로 원복시켜 diff 기반 탐지를 회피했다.
5. **약 21:20 UTC** — PyPI가 악성 버전을 제거하고 퍼블리셔 계정을 확보(secure)함.

핵심은 **공격이 패키징 레이어에서만 일어났다**는 점이다. agent-framework의 소스 코드 자체는 수정되지 않았고, 의존성에 악성 패키지를 끼워 넣지도 않았다. 그냥 배포 산출물(wheel)에 `.pth` 파일 두 개를 얹었을 뿐이다. 현재까지 CVE는 부여되지 않았다.

## .pth 시작 훅 — import하지 않아도 도는 맹독

이 공격의 기술적 핵심은 `.pth` 파일 남용이다. 많은 개발자가 "내 코드가 그 패키지를 import하지 않으면 안전하지 않나?"라고 생각하지만, `.pth`는 반대다.

Python은 가상환경·site-packages에 있는 `.pth` 파일을 **인터프리터 시작 시점**에 읽는다. 그리고 `.pth` 파일의 한 줄이 `import xxx` 형태면 그대로 **코드를 실행**한다. 공격자가 심은 `af_runtime_diag.pth`는 단 23바이트, `import af_runtime_diag` 한 줄이 전부였다.

결과적으로 이 환경에서는:

- `pip`, `pytest` 실행 시
- IDE(PyCharm, VS Code)의 언어 서버가 뜰 때
- agent-framework와 전혀 상관없는 그냥 `python myscript.py`를 돌릴 때

매번 신원털이 코드가 함께 실행됐다. import 여부와 무관하다는 점이 이 기법이 무서운 이유다. 페이로드는 `_AF_TEL=1` 플래그로 중복 실행을 막고, 분리된 자식 프로세스를 fork해(실패 시 데몬 스레드) 조용히 동작했다.

## 무엇을 훔쳤나 — AI 개발자 정조준 페이로드

훔치는 대상 목록을 보면 이것이 어설픈 대량 살포형이 아니라는 걸 알 수 있다. Orca와 Chainguard 분석에 따르면:

**환경변수**: `TOKEN`, `SECRET`, `PASSWORD`, `KEY`, `AZURE`, `AWS`, `GITHUB`, `OPENAI`, `PIP`, `NPM`, `NUGET` 등 민감 키워드가 포함된 환경변수를 현재 프로세스뿐 아니라 `/proc/*/environ`을 통해 다른 최대 300개 프로세스에서도 긁어간다.

**클라우드 신원 토큰**: 저장된 비밀만 복사하는 게 아니라 **새로 발급**받는다. Azure Instance Metadata Service(IMDS)를 통해 Azure 관리 ID(managed identity) 토큰을, Azure DevOps의 `SYSTEM_ACCESSTOKEN`과 페더레이션 OIDC 토큰을, 그리고 1.26.2에서는 `id-token: write` 권한이 있는 GitHub Actions 워크플로우의 OIDC 토큰까지 요청한다. CI 러너에서 실행됐다면 저장소 비밀을 안 훔쳐도 **클라우드에 직접 로그인할 수 있는 임시 신원**을 손에 쥐게 된다는 뜻이다.

**파일 자격증명**: Azure CLI 상태(msToken), `pip.conf`, `.npmrc`, `.netrc`, `.git-credentials`, Docker `config.json`, Hugging Face 토큰, `.env` 파일까지 쓸어담는다.

수집한 데이터는 zlib으로 압축하고 base64로 인코딩해 JSON으로 묶은 뒤, `webhook.site` URL로 HTTPS POST했다. User-Agent는 `runtime-diagnostics/1`이라는 평범한 척 이름이다.

## 내가 당했는지 확인하는 방법

다행히 영향 범위는 명확하다. **`agent-framework` 1.26.1과 1.26.2 단 두 버전**이며, 마지막 정상 버전은 1.26.0이다. `agent-framework-core`와 나머지 22개 형제 패키지는 확인 결과 무결하다. 다만 `agent-framework>=1.26` 같은 느슨한 버전 핀이라면 노출 창(11/17 20:35~21:20 UTC) 사이에 설치했을 수 있으니 반드시 점검한다.

**1단계: 버전 확인**

```bash
# lockfile과 캐시 뒤지기
grep -rn "agent-framework" --include="*.lock" --include="requirements*.txt" --include="pyproject.toml" .

# 활성 환경에 설치된 버전
pip show agent-framework | grep -E "^(Name|Version)"

# 이미 지웠더라도 설치 이력이 있는지
grep -rn "agent.framework" ~/.pip/pip.log 2>/dev/null
```

**2단계: .pth 파일 탐지**

```bash
python -c "import site; print(site.getsitepackages())"
find $(python -c "import site; print(site.getsitepackages()[0])") \
  -name "*.pth" | xargs grep -l "af_runtime_diag\|_telemetry" 2>/dev/null
```

`agent_framework_meta.pth`, `af_runtime_diag.pth`가 보이면 오염된 것이다.

**3단계: 네트워크 흔적(IOCs)**

프록시·DNS 로그에서 `webhook.site`로의 아웃바운드 연결, User-Agent `runtime-diagnostics/1` 또는 `af-runtime-diagnostics/1`, 그리고 비정상적인 IMDS 토큰 요청 버스트를 찾는다.

## 오염됐다면 — Chainguard가 권장하는 대응 순서

Chainguard의 조언에서 중요한 디테일: 오염된 호스트에서는 **Python을 다시 시작하지 말 것**이다. `.pth` 훅이 인터프리터 시작마다 재실행되기 때문이다.

1. **호스트 격리** 후 셸 도구로 `.pth`와 페이로드 모듈 파일을 직접 삭제
2. **닿을 수 있었던 모든 자격증명 로테이션** — Azure 서비스 프린서펄, Azure DevOps PAT·서비스 커넥션, GitHub 토큰, npm/PyPI/Hugging Face 토큰, Docker 레지스트리 자격증명, `.env`의 모든 비밀
3. GitHub Actions / Azure DevOps의 **런타임 신원(OIDC) 발급 이력 폐기**
4. `agent-framework==1.26.0`으로 해시 검증 핀 후 클린 환경에서 재빌드
5. 11월 17일 이후 Entra ID 로그인 로그, Azure DevOps 감사 로그, GitHub Actions 워크플로우 실행 기록을 들여다봐 허가받지 않은 접근이 있었는지 감사

## 이 사건이 남긴 교훈 세 가지

**공식 퍼블리셔도 신뢰 축이 아니다.** 지금까지의 PyPI 공급망 공격은 주로 오타 스쿼팅이나 유지관리자 계정 탈취였다면, 이번엔 Microsoft 공식 게시 계정이 뚫렸다. "publisher verified 배지가 있으니 안전하다"는 가정이 무너진 사례다.

**AI 에이전트 스택은 이제 최우선 표적이다.** 페이로드가 OpenAI·Hugging Face 키와 Azure AI 관련 관리 ID를 정조준한다는 점은, 공격자들이 에이전트 개발 환경의 자격증명 구조를 정확히 공부했음을 보여준다. `id-token: write` 권한을 디폴트로 붙여두는 GitHub Actions 관행도 재점검할 때다 — 필요한 워크플로우에만 최소 권한으로 부여하자.

**탐지 속도가 피해 크기를 정한다.** Chainguard는 48초, Elastic은 6분 만에 잡았지만, 조직의 자체 파이프라인은? 새 패키지 버전이 올라오면 해시를 검증하고 알려진 IOC와 대조하는 자동화 게이트가 없다면 이번 45분 창을 스스로 메울 방법이 없다. `pip install --require-hashes`나 lockfile 기반 재현 가능 빌드를 습관화하는 것이 개인 개발자에게 현실적인 방어선이다.

## 핵심 요약

- 2026-11-17 20:35~21:20 UTC, Microsoft 공식 PyPI 계정이 침해되어 `agent-framework` 1.26.1/1.26.2에 신원털이가 실림 (월 ~19만 다운로드 패키지)
- 페이로드는 `.pth` 시작 훅 — agent-framework를 import하지 않아도 Python 인터프리터를 켤 때마다 실행됨
- 환경변수·자격증명 파일에 더해 IMDS 관리 ID 토큰, Azure DevOps·GitHub Actions OIDC 토큰을 **새로 발급해서** 훔쳐 webhook.site로 유출
- 대상은 1.26.1/1.26.2 두 버전뿐. 마지막 정상 버전은 1.26.0, 형제 패키지 22개는 무결
- 오염 호스트는 Python 재시작 금지 → 파일 삭제 → 전면 자격증명 로테이션 → 클라우드 감사 로그 확인

당장 할 수 있는 첫 단계는 이것이다: 오늘 내 프로젝트의 lockfile에서 `agent-framework`를 grep해보고, 없다면 다음으로 내 CI 워크플로우에 `id-token: write`가 어디에 열려 있는지 확인하라. 이 두 가지가 이번 사건이 우리에게 남긴 실질적인 점검 항목이다.

## 참고 자료

- Orca Security — Microsoft agent-framework PyPI Supply-Chain Attack 분석 (2026-11-18)
- Chainguard — agent-framework compromised on PyPI: 1.26.1 and 1.26.2 (2026-11-17)
- Elastic Security Labs — 탐지 6분 기록 블로그
