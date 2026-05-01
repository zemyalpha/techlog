---
title: "MCP 서버 30분 만에 만들기: Claude Code와 연동하는 나만의 AI 도구"
date: "2026-04-27"
keywords: ["MCP", "Model Context Protocol", "Claude Code", "FastMCP", "AI 도구"]
lang: "ko"
description: "FastMCP 라이브러리로 30분 안에 첫 MCP 서버를 만들고 Claude Code와 연동하는 실전 가이드. 코드 예시와 설정 방법 포함."
---

# MCP 서버 30분 만에 만들기: Claude Code와 연동하는 나만의 AI 도구

2026년 4월 27일

---

## 1. 서론: MCP가 왜 지금 주목받는가

2024년 11월 Anthropic이 Model Context Protocol(이하 MCP)을 오픈소스로 공개한 이후, AI 에이전트 생태계는 빠르게 변화하고 있다. 2026년 3월 기준 MCP 공식 TypeScript SDK는 주간 npm 다운로드 3,300만 건을 돌파했고, MCP 공식 서버 레포지토리는 GitHub에서 84,600개 이상의 스타를 받았다. Anthropic 자체도 2025년 말 약 400억 달러(약 5조 3천억 원)의 기업가치 평가를 받으며 AI 인프라의 핵심 플레이어로 자리 잡았다.

이 변화의 중심에는 Claude Code가 있다. Claude Code는 GitHub에서 118,000개 이상의 스타를 받은 터미널 기반 에이전트 코딩 도구로, MCP를 통해 외부 도구와 직접 연동할 수 있다. 즉, MCP 서버를 만들면 Claude Code, Claude Desktop, Cursor 등 다양한 AI 클라이언트에서 내가 만든 도구를 즉시 사용할 수 있다.

이 글에서는 FastMCP 라이브러리를 사용해 30분 안에 동작하는 MCP 서버를 만들고, Claude Code와 연동하는 전 과정을 정리한다.

## 2. MCP란 무엇인가

### 프로토콜의 핵심 개념

MCP는 AI 모델과 외부 도구·데이터 소스 간의 통신을 표준화하는 오픈 프로토콜이다. 쉽게 말해 "AI가 세상과 소통하는 표준 언어"라고 볼 수 있다.

기존에는 AI 에이전트에 도구를 연결하려면 각 서비스마다 고유한 API 연동 코드를 작성해야 했다. OpenAI용 플러그인, Anthropic용 도구, 각종 커스텀 연동 코드가 제각각이었다. MCP는 이 문제를 해결한다. 한 번 MCP 서버를 만들면, MCP를 지원하는 모든 AI 클라이언트에서 그 도구를 사용할 수 있다.

### 아키텍처 한눈에 보기

```
┌──────────────┐     MCP      ┌──────────────┐
│  AI Client   │◄────────────►│  MCP Server  │
│ (Claude Code │   stdio/     │  (나만의     │
│  Cursor 등)  │   SSE        │   도구)      │
└──────────────┘              └──────────────┘
```

MCP 서버는 세 가지 핵심 기능을 제공한다:

- **Tools**: AI가 호출할 수 있는 함수 (예: 계산기, 데이터베이스 조회, API 호출)
- **Resources**: AI가 읽을 수 있는 데이터 (예: 파일, 데이터베이스 레코드)
- **Prompts**: 재사용 가능한 프롬프트 템플릿

이 중에서 개발자가 가장 먼저 다루게 되는 것은 Tools다. 이 글에서도 Tool 하나를 만들어보는 것으로 시작한다.

## 3. 실전: 첫 MCP 서버 만들기

### 환경 준비

Python 3.10 이상이 필요하다. FastMCP는 MCP 서버 개발을 극단적으로 단순화하는 파이썬 라이브러리로, 현재 GitHub에서 24,800개 이상의 스타를 받고 있다.

```bash
# 가상환경 생성 (권장)
python -m venv mcp-env
source mcp-env/bin/activate  # macOS/Linux
# mcp-env\Scripts\activate   # Windows

# FastMCP 설치
pip install fastmcp
```

### 계산기 MCP 서버 작성

`calculator_server.py` 파일을 만들고 다음 코드를 작성한다.

```python
from fastmcp import FastMCP

# MCP 서버 인스턴스 생성
mcp = FastMCP("Calculator")

@mcp.tool()
def add(a: float, b: float) -> float:
    """두 숫자를 더합니다."""
    return a + b

@mcp.tool()
def subtract(a: float, b: float) -> float:
    """첫 번째 숫자에서 두 번째 숫자를 뺍니다."""
    return a - b

@mcp.tool()
def multiply(a: float, b: float) -> float:
    """두 숫자를 곱합니다."""
    return a * b

@mcp.tool()
def divide(a: float, b: float) -> float:
    """첫 번째 숫자를 두 번째 숫자로 나눕니다.
    
    Args:
        a: 나누어지는 수 (분자)
        b: 나누는 수 (분모). 0이 될 수 없습니다.
    
    Returns:
        나눗셈 결과
    
    Raises:
        ValueError: b가 0인 경우
    """
    if b == 0:
        raise ValueError("0으로 나눌 수 없습니다.")
    return a / b

@mcp.tool()
def power(base: float, exponent: float) -> float:
    """거듭제곱을 계산합니다. base^exponent를 반환합니다."""
    return base ** exponent

# 서버 실행 (stdio 모드 - AI 클라이언트와 통신용)
if __name__ == "__main__":
    mcp.run()
```

이게 전부다. `@mcp.tool()` 데코레이터 하나로 일반 파이썬 함수가 MCP Tool이 된다.

### 로컬에서 테스트하기

서버를 실행하기 전에, FastMCP가 제공하는 개발용 inspector로 테스트해볼 수 있다.

```bash
# MCP Inspector로 서버 테스트 (브라우저가 열리며 UI에서 테스트 가능)
fastmcp dev calculator_server.py
```

Inspector가 열리면 등록된 Tool 목록을 확인하고, 직접 파라미터를 입력해 결과를 볼 수 있다.

서버를 직접 실행하려면:

```bash
# stdio 모드로 실행 (AI 클라이언트 연결용)
fastmcp run calculator_server.py
```

### 왜 FastMCP인가

공식 MCP Python SDK(`modelcontextprotocol/python-sdk`, GitHub 22,700+ 스타)도 있지만, FastMCP가 더 인기 있는 이유는 명확하다.

**공식 SDK 방식:**
```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import mcp.server.stdio

server = Server("calculator")

@server.list_tools()
async def list_tools():
    return [
        Tool(name="add", description="두 숫자를 더합니다.",
             inputSchema={"type": "object", "properties": 
                         {"a": {"type": "number"}, "b": {"type": "number"}},
                         "required": ["a", "b"]})
    ]

@server.call_tool()
async def call_tool(name, arguments):
    if name == "add":
        return [TextContent(type="text", text=str(arguments["a"] + arguments["b"]))]
```

**FastMCP 방식:**
```python
from fastmcp import FastMCP
mcp = FastMCP("Calculator")

@mcp.tool()
def add(a: float, b: float) -> float:
    """두 숫자를 더합니다."""
    return a + b
```

함수 시그니처와 독스트링만으로 Tool의 스키마와 설명이 자동 생성된다. 보일러플레이트 코드가 거의 사라진다.

## 4. Claude Code와 연동하는 방법

MCP 서버를 만들었으니, 이제 Claude Code에서 사용해보자.

### 설정 파일 위치

Claude Code의 MCP 설정은 `claude_desktop_config.json` 파일에 등록한다. 플랫폼별 위치는 다음과 같다:

| 플랫폼 | 경로 |
|--------|------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

Claude Code CLI를 사용하는 경우 프로젝트 루트에 `.mcp.json` 파일을 만들 수도 있다.

### 설정 파일 작성

`claude_desktop_config.json` 파일을 열고(없으면 생성) 다음 내용을 추가한다:

```json
{
  "mcpServers": {
    "calculator": {
      "command": "python",
      "args": ["/absolute/path/to/calculator_server.py"],
      "env": {}
    }
  }
}
```

가상환경을 사용하는 경우 `python` 대신 가상환경의 파이썬 경로를 지정해야 한다:

```json
{
  "mcpServers": {
    "calculator": {
      "command": "/absolute/path/to/mcp-env/bin/python",
      "args": ["/absolute/path/to/calculator_server.py"],
      "env": {}
    }
  }
}
```

`uv`를 사용하는 경우 더 간단하다:

```json
{
  "mcpServers": {
    "calculator": {
      "command": "uv",
      "args": [
        "--directory",
        "/absolute/path/to/project",
        "run",
        "calculator_server.py"
      ]
    }
  }
}
```

### Claude Code CLI에서 설정

Claude Code CLI에서는 명령어로 바로 추가할 수도 있다:

```bash
# Claude Code에 MCP 서버 추가
claude mcp add calculator -- python /absolute/path/to/calculator_server.py

# 등록된 MCP 서버 목록 확인
claude mcp list

# 특정 서버 삭제
claude mcp remove calculator
```

### 연동 확인

Claude Desktop을 재시작한 후, 채팅창에서 다음과 같이 입력해보자:

> "123 * 456 계산해줘"

Claude가 `multiply` Tool을 자동으로 호출해서 결과를 반환할 것이다. 수동으로 Tool 선택 UI에서 "calculator" 서버의 도구들을 확인할 수도 있다.

Claude Code CLI에서는 `/mcp` 명령으로 연결 상태를 확인할 수 있다.

## 5. 생태계 등록: Smithery와 Glama

MCP 서버를 만들고 나면, 다른 사람들도 쉽게 사용할 수 있도록 생태계 레지스트리에 등록하는 것이 좋다.

### Smithery (smithery.ai)

Smithery는 MCP 서버 디렉토리로, 검색과 설치가 가능하다.

1. **GitHub 저장소 준비**: 서버 코드를 GitHub에 푸시한다
2. **`smithery.yaml` 파일 작성** (프로젝트 루트):
   ```yaml
   name: my-calculator-mcp
   description: 계산기 MCP 서버 - 사칙연산 및 거듭제곱 지원
   version: 1.0.0
   runtime: python
   entrypoint: calculator_server.py
   ```
3. **Smithery에 제출**: Smithery 웹사이트에서 "Submit Server"를 통해 GitHub 저장소 URL을 등록한다

등록되면 사용자가 다음 명령 하나로 설치할 수 있다:

```bash
npx @smithery/cli install my-calculator-mcp --client claude
```

### Glama (glama.ai)

Glama 역시 MCP 서버 레지스트리를 제공한다.

1. **Glama 웹사이트**에서 "Publish MCP Server" 메뉴 선택
2. GitHub 저장소 URL 또는 npm 패키지 이름 입력
3. 메타데이터(설명, 카테고리, 스크린샷 등) 작성
4. 검수 후 공개

### 기타 등록 채널

- **MCP Get (mcp.get)**: `npx mcp-get` 명령으로 설치 가능한 MCP 서버 목록 관리
- **npm 배포**: TypeScript MCP 서버의 경우 npm 패키지로 배포하면 설치가 편하다
- **Docker Hub**: 서버를 컨테이너 이미지로 배포하면 환경 의존성 문제를 해결할 수 있다

### 패키징 팁

등록 전에 다음 사항을 정비하면 채택률이 높아진다:

- `README.md`에 설치 방법과 사용 예시 명확히 작성
- `pyproject.toml` 또는 `package.json`에 의존성 명시
- 환경변수가 필요한 경우 `.env.example` 파일 포함
- MCP Inspector로 테스트한 스크린샷 첨부

## 6. 결론: 핵심 요약과 다음 단계

### 핵심 요약

| 항목 | 내용 |
|------|------|
| MCP | AI 클라이언트와 외부 도구를 연결하는 표준 프로토콜 |
| FastMCP | 파이썬에서 MCP 서버를 만드는 가장 쉬운 방법 (24,800+ GitHub 스타) |
| 개발 시간 | 기본 서버 기준 30분 이내 |
| 연동 | Claude Code, Claude Desktop, Cursor 등 다양한 클라이언트 지원 |
| 배포 | Smithery, Glama 등 레지스트리를 통해 공유 가능 |

### 다음 단계

이 글에서는 기본적인 계산기 서버를 만들어보았다. 여기서 한 단계 더 나아가려면:

1. **실용적인 도구 만들기**: 데이터베이스 조회, Slack 알림, Jira 티켓 생성 등 실무에서 바로 쓸 수 있는 도구를 MCP 서버로 만들어보자.

2. **Resource와 Prompt 활용**: Tool 외에도 Resource(파일·데이터 제공)와 Prompt(재사용 프롬프트 템플릿)를 등록하면 서버의 활용도가 크게 올라간다.

3. **TypeScript SDK로 전환**: 대규모 서비스라면 공식 TypeScript SDK(`@modelcontextprotocol/sdk`, 주간 3,300만 npm 다운로드)를 사용하는 것이 생태계 호환성에 유리하다.

4. **인증과 보안**: 실서비스 환경에서는 API 키 관리, 접근 제어, 감사 로깅이 필요하다. FastMCP의 미들웨어 기능과 환경변수 설정을 활용하자.

5. **SSE 모드로 배포**: stdio 모드 대신 Server-Sent Events(SSE) 모드를 사용하면 원격 서버로 배포가 가능하다. `mcp.run(transport="sse")` 한 줄로 전환할 수 있다.

MCP는 AI 에이전트가 현실 세계의 도구와 데이터에 접근하는 방식을 근본적으로 바꾸고 있다. 이 표준이 자리 잡으면, AI 도구 개발은 오늘날의 모바일 앱 개발처럼 일상적인 영역이 될 것이다. 30분 투자로 그 흐름의 시작점에 서보자.

---

**참고 자료:**
- MCP 공식 문서: https://modelcontextprotocol.io
- FastMCP GitHub: https://github.com/jlowin/fastmcp
- MCP TypeScript SDK: https://github.com/modelcontextprotocol/typescript-sdk
- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
- Claude Code: https://github.com/anthropics/claude-code
- Smithery: https://smithery.ai
