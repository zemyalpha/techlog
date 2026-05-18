---
title: "Python 코드에서 자주 발생하는 보안 취약점과 실전 대응 전략"
date: "2026-05-18"
keywords: ["Python 보안", "웹 취약점", "코드 인젝션", "의존성 관리", "보안 코딩"]
lang: "ko"
description: "Python 애플리케이션에서 발생하기 쉬운 주요 보안 취약점과 이를 방지하기 위한 실전 대응 전략을 코드 예시와 함께 설명합니다."
---

# Python 코드에서 자주 발생하는 보안 취약점과 실전 대응 전략

최근 AI와 웹 자동화의 확산으로 Python은 데이터 분석에서 웹 백엔드, 인프라 자동화까지 폭넓게 사용되고 있습니다. 하지만 인기만큼이나 공격자들의 표적이 되는 언어이기도 합니다. 공식 PyPI 저장소에는 매일 수천 개의 새로운 패키지가 등록되며, 이 중 일부는 악성 의도를 지닌 패키지(의도적 공격)이거나, 오랜 시간 업데이트되지 않아 보안 패치가 누락된 패키지(기회적 공격)입니다. 이 글에서는 실제 프로젝트에서 흔히 발생하는 Python 관련 보안 취약점을 중심으로, 어떻게 예방하고 대응해야 하는지 실질적인 전략을 제시합니다.

## 1. 코드 인젝션과 서식 문자열 취약점 (Python)

가장 대표적인 공격 방식은 외부에서 주입된 데이터를 신뢰하지 않고 사용하는 경우입니다. 특히 `eval()` 및 `exec()` 함수는 입력 값에 따라 임의의 코드를 실행할 수 있어 극도로 위험합니다.

**위험한 예시:**
```python
user_input = "__import__('os').system('rm -rf /')"
eval(user_input)  # 시스템 파괴 명령 실행
```

**안전한 대안:**
- `eval()`이나 `exec()`를 **절대 사용하지 마세요.** 구성 파일, 계산식 처리 등 다양한 상황에서 필요할 수 있지만, 대안이 항상 존재합니다.
- 계산식 처리는 `asteval` 같은 전용 라이브러리를 사용하세요.
- 구성 데이터는 안전한 형식의 YAML, JSON을 파싱하는 것이 좋습니다.

또한 `.format()` 또는 f-string을 사용할 때, 사용자가 제공한 포맷 문자열을 직접 사용하면 정보 유출로 이어질 수 있습니다.

**위험한 예시:**
```python
user_format = "{config.__init__.__globals__[SECRET]}.secret".format()
```

**안전한 대안:**
입력된 값은 `.format()`의 인자로 전달하고, 포맷 문자열은 개발자가 정의해야 합니다.

```python
safe_output = "User said: {}".format(user_input)  # 안전함
```

## 2. 악성 패키지와 의존성 관리

PyPI는 수많은 훌륭한 오픈소스 라이브러리의 보고이지만, 동시에 공격자가 "의사(Py-thons)"를 숨겨놓는 황량한 바위장입니다. 유사한 이름을 가진 패키지(타이포스크와칭)를 업로드하거나, 무기명 계정으로 빠른 업데이트를 반복하다가 나중에 악성 코드를 삽입하는 사례가 빈번합니다.

악성 패키지가 PyPI에 게시되는 사례는 지속적으로 보고되고 있으며, Open Source Security Foundation(OSSF)의 Malicious Packages 리포지토리 등에서 지속적으로 모니터링되고 있습니다. 공격자들은 타이포스크와칭(typosquatting) 등을 통해 이러한 패키지를 유포하므로, 설치 시 출처를 철저히 확인하는 것이 중요합니다.

**실전 대응 전략:**

1.  **`requirements.txt` 대신 `pyproject.toml`과 `poetry` 사용:** 더 엄격한 의존성 고정과 해시 기반 검증을 가능하게 합니다.
    ```toml
    [tool.poetry.dependencies]
    requests = {version = "^2.28.0", checksum = "sha256:{해시값}"}
    ```

2.  **의존성 스캔 도구 정기적 사용:**
    - `pip-audit`: 알려진 CVE가 포함된 패키지를 탐지합니다.
    - `safety check`: 공개된 보안 취약점 데이터베이스와 비교합니다.
    ```bash
    poetry export --format=requirements.txt --without-hashes | pip-audit
    ```

3.  **신뢰할 수 있는 출처 확인:** 패키지의 GitHub 저장소, 기여자 수, 이슈 트래커의 활발함 등을 확인하세요. 다운로드 수만으로 신뢰해서는 안 됩니다.

## 3. 웹 프레임워크에서의 보안 (Django와 Flask)

웹 프레임워크에서도 주의해야 할 고전적인 취약점들이 존재합니다.

**크로스 사이트 스크립팅(XSS):**
Django 템플릿은 기본적으로 HTML 이스케이프를 수행하므로 안전하지만, `|safe` 필터를 무분별하게 사용하면 취약점이 생깁니다.
```django
<!-- 위험 -->
{{ user_content|safe }}

<!-- 안전 -->
{{ user_content }}  <!-- 자동 이스케이프 됨 -->
```

**크로스 사이트 요청 위조(CSRF):**
Flask는 CSRF 보호를 기본으로 제공하지 않기 때문에 `Flask-WTF` 또는 `Flask-CSRF` 같은 확장 프로그램을 사용해야 합니다.

Django는 기본적으로 CSRF 보호를 활성화하며, `{% csrf_token %}` 템플릿 태그를 사용하여 폼에 토큰을 포함시켜야 요청이 유효합니다.

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect()
csrf.init_app(app)
```

**SQL 인젝션:**
`raw()` 쿼리를 사용하거나, ORM을 제대로 활용하지 않으면 발생할 수 있습니다. Django ORM은 파라미터화된 쿼리를 사용하여 기본적으로 안전합니다.
```python
# 절대 금지
MyModel.objects.extra(where=[f"field='{user_input}'"])

# 안전
MyModel.objects.filter(field=user_input)
```

## 4. 환경 변수와 민감 정보 관리

API 키, 데이터베이스 비밀번호와 같은 민감 정보는 코드에 하드코딩해서는绝不(절대) 안 됩니다.

**실전 패턴:** `.env` 파일과 `python-dotenv` 라이브러리 사용.

```python
# .env 파일
SECRET_KEY=your-very-secret-key-here
DATABASE_URL=postgresql://user:pass@localhost/db

# 코드에서 로드
from dotenv import load_dotenv
import os

load_dotenv()

secret_key = os.getenv("SECRET_KEY")  # 코드 안에는 키값 X
```

`.env` 파일은 반드시 `.gitignore`에 추가하여 Git 저장소에 올라가지 않도록 하세요.

## 결론

Python으로 개발할 때 보안은 기능 개발 이후에 고려하는 것이 아니라, 개발의 첫 번째 원칙이 되어야 합니다. `eval()` 사용 금지, 의존성 스캐닝, 민감 정보 관리 등 기본적인 원칙을 철저히 지키는 것만으로도 대부분의 공격을 차단할 수 있습니다. 오늘 소개한 전략을 개발 프로세스에 빌드하세요. CI/CD 파이프라인에 `pip-audit`를 추가하고, 코드 리뷰에서 `.env` 파일과 `eval()` 사용 여부를 반드시 확인하세요.

> 본 내용은 실제 PyPI 보고서, OWASP Top Ten, Python 공식 문서를 참고하여 작성되었으며, 작성된 사실은 이후 별도의 팩트체크 과정을 거칠 예정입니다.