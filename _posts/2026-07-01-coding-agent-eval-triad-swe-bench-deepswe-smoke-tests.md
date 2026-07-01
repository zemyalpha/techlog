---
title: "코딩 에이전트 성능평가, SWE-bench만 보면 놓치는 3가지"
date: "2026-07-01"
keywords: ["코딩 에이전트", "SWE-bench", "DeepSWE", "벤치마크", "소프트웨어 엔지니어링"]
lang: "ko"
description: "SWE-bench와 DeepSWE를 함께 쓰고, 내 저장소 스모크 테스트까지 묶어 코딩 에이전트를 더 현실적으로 평가하는 방법을 정리합니다."
---

# 코딩 에이전트 성능평가, SWE-bench만 보면 놓치는 3가지

요즘 코딩 에이전트는 단순한 자동완성 도구를 넘어, 저장소를 읽고 수정하고 테스트까지 돌리는 쪽으로 빠르게 진화하고 있습니다. Anthropic은 Claude Sonnet 5를 “coding and everyday professional work”에 강한 “most agentic Sonnet”이라고 소개했고, SWE-bench 쪽도 original, verified, multilingual, multimodal, lite 같은 여러 축으로 평가 체계를 넓혀 왔습니다. DeepSWE 역시 frontier coding agents를 original, long-horizon software engineering tasks로 측정한다고 밝힙니다.

문제는, 이런 움직임이 곧바로 “좋은 에이전트”를 뜻하지는 않는다는 점입니다. 한 개의 리더보드 점수만 보고 도입하면, 데모는 멋지지만 실제 저장소에서는 브레이크를 밟는 에이전트를 뽑게 됩니다. 그래서 저는 코딩 에이전트 평가를 **벤치마크 게이트 + 장기 작업 스트레스 테스트 + 내 저장소 스모크 테스트**의 3단계로 나누는 편이 훨씬 낫다고 봅니다.

## 1) 왜 SWE-bench 한 방으로는 부족한가

SWE-bench는 분명 유용합니다. 최소한 “코드 수정 후 테스트를 통과하는 능력”을 공통 언어로 비교하게 해 주니까요. 하지만 실제 제품 도입에서는 다음이 따로 중요합니다.

- 저장소 구조를 얼마나 오래 기억하는가
- 한 번의 수정이 아니라 여러 파일에 걸친 연쇄 변경을 안정적으로 처리하는가
- 실패했을 때 되돌아오거나 재시도하는가
- 모델이 똑똑한지보다, 실패했을 때 서비스가 무너지지 않는가

즉, 벤치마크 점수는 **입장권**일 뿐입니다. 운영 적합성은 그 다음 단계에서 확인해야 합니다.

## 2) 추천하는 3단계 평가 파이프라인

### Step 1. 짧은 비교: SWE-bench 계열로 기본 체력 보기

처음에는 SWE-bench Verified나 Lite처럼 비교 비용이 낮은 축으로 시작합니다. 목적은 “이 에이전트가 코드 수정형 과제를 아예 못 푸는가?”를 빠르게 거르는 것입니다.

```yaml
evaluation:
  gate:
    benchmark: SWE-bench Verified
    metric: pass@1
    threshold: 0.30
  smoke:
    commands:
      - pytest -q
      - ruff check .
```

여기서 중요한 건 절대 점수보다 **같은 조건에서의 상대 비교**입니다. 모델 A가 B보다 5점 높아도, 실제 업무에선 B가 훨씬 적은 파일을 건드리고 더 적게 깨뜨릴 수 있습니다.

### Step 2. 장기 작업: DeepSWE 스타일의 오래 걸리는 과제 보기

DeepSWE가 강조하는 지점은 original, long-horizon software engineering tasks입니다. GitHub README 기준으로는 113개 과제가 TypeScript, Go, Python, JavaScript, Rust에 걸쳐 있고, 격리된 실행 환경과 program-based verifier를 사용합니다. 이건 아주 중요합니다. 실전 에이전트는 단일 함수 수정보다, 맥락을 유지한 채 여러 차례 판단을 내리는 일이 더 많기 때문입니다.

장기 과제에서는 아래 항목을 꼭 기록해야 합니다.

- 첫 응답까지 걸린 시간
- 중간에 잃어버린 파일 수
- 불필요한 리팩터링 횟수
- 테스트 실패 후 복구 성공 여부

즉, “최종 성공”만 보지 말고 **과정 품질**을 점수화해야 합니다.

### Step 3. 내 저장소 스모크 테스트: 진짜 운영 적합성 확인

가장 현실적인 평가는 내 코드베이스에서 나옵니다. 다음처럼 아주 작은 시나리오를 만들어 두면 좋습니다.

```python
import subprocess
from pathlib import Path

TESTS = [
    ["pytest", "-q"],
    ["ruff", "check", "."],
]

def run(cmd, cwd):
    p = subprocess.run(cmd, cwd=cwd, text=True, capture_output=True)
    return {
        "cmd": " ".join(cmd),
        "returncode": p.returncode,
        "stdout": p.stdout[-4000:],
        "stderr": p.stderr[-4000:],
    }


def smoke_test(repo_path: str):
    repo = Path(repo_path)
    results = [run(cmd, repo) for cmd in TESTS]
    passed = sum(r["returncode"] == 0 for r in results)
    return {
        "repo": str(repo),
        "passed": passed,
        "total": len(results),
        "results": results,
    }
```

이 스모크 테스트는 거창할 필요가 없습니다. 오히려 작아야 합니다. 핵심은 에이전트가 **당신의 규칙**을 지키는지 보는 것입니다. 예를 들어, 테스트만 통과하고 코드 스타일을 망가뜨리거나, 반대로 린트는 깨끗하지만 기능을 망치는 에이전트는 운영용으로 부적합합니다.

## 3) 실전에서 자주 하는 실수

### 1. 벤치마크 점수와 제품 품질을 동일시하기

점수가 높아도, 프롬프트 구조가 조금만 바뀌면 성능이 급락할 수 있습니다. 평가할 때는 “한 번 잘했다”가 아니라 “조건이 달라도 버티는가”를 봐야 합니다.

### 2. 실패 로그를 남기지 않기

에이전트는 실패할 수 있습니다. 중요한 건 실패를 숨기지 않는 것입니다. 최소한 아래는 저장하세요.

- 입력 프롬프트
- 생성된 패치
- 실행한 명령어
- 테스트 결과
- 재시도 횟수

이 다섯 개가 남아 있어야 나중에 원인을 추적할 수 있습니다.

### 3. 긴 작업에서 중간 점검 없이 끝까지 맡기기

장기 작업은 중간 중간 체크포인트가 있어야 합니다. 1시간 작업이라면 10분 단위로 상태를 기록하고, 위험한 변경은 자동 적용 대신 검토 대기열로 보내는 편이 안전합니다.

## 결론: 코딩 에이전트는 "점수"보다 "복원력"을 봐야 합니다

핵심만 정리하면 이렇습니다.

- SWE-bench 계열은 기본 비교 기준으로 유용합니다.
- DeepSWE 스타일의 long-horizon 과제는 실제 업무 적합성을 더 잘 드러냅니다.
- 내 저장소 스모크 테스트는 최종 합격선입니다.
- 실패 로그와 재시도 기록이 있어야 운영이 가능합니다.
- 모델이 똑똑해 보여도, 복원력이 없으면 프로덕션에서는 위험합니다.

저라면 새 코딩 에이전트를 도입할 때 먼저 **SWE-bench 계열로 빠르게 거르고**, 그 다음 **장기 작업으로 버티는 힘을 확인하고**, 마지막으로 **내 저장소 스모크 테스트에서 실제로 안 깨지는지**를 봅니다. 이 세 단계를 통과한 에이전트라면, 적어도 데모용이 아니라 업무용 후보로는 충분히 논의해 볼 수 있습니다.

검증에 사용한 공식 자료: SWE-bench 리더보드, DeepSWE 공식 페이지, Anthropic Claude Sonnet 5 소개 페이지.
