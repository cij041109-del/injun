# MCP 실습 세팅 가이드 (campus-helper)

공개SW실무 · MCP 프로그래밍 발표 실습용 저장소입니다.
Python으로 작은 MCP 서버(`campus-helper`)를 만들고, VS Code의 AI 채팅이 그 서버의 도구를 호출하는 것까지 해 봅니다.

- 환경: Windows 10/11, VS Code(GitHub Copilot 채팅 사용 가능), PowerShell
- 소요: 설치 포함 약 20~30분

## 실습 흐름

```
1. 프로젝트 만들기  →  2. server.py 작성  →  3. Inspector로 확인  →  4. VS Code에 연결  →  5. AI에게 질문
```

먼저 서버만 따로 확인(3)하고, 그다음 AI에 연결(4~5)합니다. 문제가 생겼을 때 서버 문제인지 연결 문제인지 나눠서 볼 수 있습니다.

## 이 실습으로 알 수 있는 것

1. MCP 서버는 파이썬 함수 몇 개 + `@mcp.tool()` 만으로 만들 수 있다.
2. AI는 서버가 알려 준 도구 이름·설명·입력 형식만 보고 도구를 고르고 호출한다. 한 질문에 도구 여러 개도 연속으로 쓴다.
3. 서버 단독 확인(Inspector)과 AI 연결 확인(VS Code)을 나눠서 보면 문제를 빨리 찾는다.
4. AI가 접근하는 범위는 서버가 만든 도구로 제한되고, 실행 전에 사용자 승인을 거친다.

---

## 0. 사전 준비

PowerShell에서 아래를 확인합니다.

```powershell
node --version      # v18 이상이면 OK (Inspector 실행에 필요)
python --version    # Python 3.12.x 가 나오면 OK
```

`python --version`에서 `Python ` 만 나오고 버전이 안 보이면 Microsoft Store 가짜 실행 파일입니다. 아래처럼 직접 설치하세요.

```powershell
winget install Python.Python.3.12
```

설치 후 **VS Code를 완전히 종료했다가 다시 실행**해야 PATH가 갱신됩니다. (터미널만 새로 여는 것으로는 부족할 수 있습니다.)

Node.js가 없다면 https://nodejs.org 에서 LTS 버전을 설치합니다.

## 1. uv 설치와 프로젝트 만들기

`pip install uv`가 연결 오류로 실패해서 winget으로 설치했습니다.

```powershell
winget install --id=astral-sh.uv -e
```

설치 후 VS Code를 다시 시작하고 폴더를 만듭니다.

```powershell
mkdir mcp-lab
cd mcp-lab
uv init
uv add "mcp[cli]"
```

VS Code에서 `File > Open Folder`로 `mcp-lab` 폴더를 엽니다.

> 참고: `uv init`은 `src/mcp_lab/__init__.py` 구조를 만듭니다. 서버 파일은 `src` 안이 아니라 **프로젝트 맨 바깥(`mcp-lab` 바로 아래)** 에 만드세요.

### mcp 버전 주의

수업 자료의 `from mcp.server.fastmcp import FastMCP` 코드는 mcp **2.x** 에서 동작하지 않습니다. 2.x에서는 `FastMCP`가 `MCPServer`로 바뀌었습니다.

| mcp 버전 | import |
| --- | --- |
| 2.x (이 가이드, 2.3.0에서 확인) | `from mcp.server.mcpserver import MCPServer` |
| 1.x | `from mcp.server.fastmcp import FastMCP` |

수업 자료 코드를 그대로 쓰고 싶다면 1.x로 고정할 수 있습니다.

```powershell
uv add "mcp[cli]<2"
```

## 2. server.py 작성

`mcp-lab` 폴더 바로 아래에 `server.py` 파일을 새로 만들고 아래를 붙여 넣습니다.
(데이터는 예시 값입니다.)

```python
from mcp.server.mcpserver import MCPServer

mcp = MCPServer("campus-helper")

# 예시 데이터
COURSES = {
    "공개SW실무": {"room": "S1234", "exam": "12/14", "assignment": "10/19"},
    "자료구조": {"room": "S2345", "exam": "12/16", "assignment": "11/02"},
}


@mcp.tool()
def list_courses() -> str:
    """List the names of all available courses."""
    return ", ".join(COURSES.keys())


@mcp.tool()
def get_classroom(course: str) -> str:
    """Return the classroom of a course."""
    info = COURSES.get(course.strip())
    return info["room"] if info else "No matching course"


@mcp.tool()
def get_assignment_deadline(course: str) -> str:
    """Return the assignment deadline of a course."""
    info = COURSES.get(course.strip())
    return info["assignment"] if info else "No matching course"


@mcp.tool()
def get_exam_schedule(course: str) -> str:
    """Return the exam date of a course."""
    info = COURSES.get(course.strip())
    return info["exam"] if info else "No matching course"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

핵심은 평범한 함수 위에 `@mcp.tool()` 한 줄을 붙이는 것입니다. 함수 이름, docstring(설명), 매개변수 타입이 AI에게 보이는 "사용 설명서"가 됩니다.

## 3. Inspector로 서버 확인

```powershell
uv run mcp dev server.py
```

브라우저에서 MCP Inspector가 열립니다. (처음 실행할 때 Node 패키지를 내려받아 시간이 걸릴 수 있습니다.)

1. **Connect** 클릭
2. **Tools** 탭으로 이동(목록이 비어 있으면 List Tools 버튼 클릭) → 도구 4개(`list_courses`, `get_classroom`, `get_assignment_deadline`, `get_exam_schedule`)가 보이면 성공
3. `get_exam_schedule`을 선택하고 `course`에 `공개SW실무` 입력 후 실행 → 결과 `12/14`

확인이 끝나면 터미널에서 `Ctrl + C`로 종료합니다.

## 4. VS Code에 연결

프로젝트 폴더에 `.vscode` 폴더를 만들고 `.vscode/mcp.json` 파일을 만듭니다.

```powershell
mkdir .vscode
```

```json
{
  "servers": {
    "campus-helper": {
      "type": "stdio",
      "command": "${workspaceFolder}/.venv/Scripts/python.exe",
      "args": ["${workspaceFolder}/server.py"]
    }
  }
}
```

파일을 **저장**(Ctrl+S)하면 `"campus-helper"` 위에 **Start / Running / Error** 표시(CodeLens)가 나타납니다.

> 대신 uv로 실행하는 설정(`"command": "uv"`, `"args": ["run", "python", "server.py"]`)도 쓸 수 있지만, 이 가이드에서는 위의 `.venv` 파이썬 경로 방식으로 확인했습니다.

### 꼭 먼저 할 것: 채팅 세션 만들기

Start를 눌렀는데 Error가 나고 출력 로그에 아래 문장이 있다면 서버 문제가 아닙니다.

```
Cannot start an MCP server before the session runtime has been created. Send a message first.
```

Copilot 채팅 창에서 **아무 메시지(예: "안녕")를 한 번 보낸 뒤** 다시 **Start**를 누르면 `Running`이 됩니다.

## 5. AI에게 질문하기

1. Copilot 채팅을 **Agent** 모드로 둡니다.
2. 아래처럼 질문합니다.

```
공개SW실무 시험 일정이랑 강의실을 같이 알려줘.
```

3. 도구 실행 확인 창이 뜨면 내용을 확인하고 **Allow in this Session**을 누릅니다.
4. AI가 `get_exam_schedule`, `get_classroom` 두 도구를 연속으로 호출하고 답변합니다.

예상 결과

```
시험일: 12월 14일
강의실: S1234
```

호출 과정을 보려면 답변 위의 `Completed 2 steps` 항목을 펼쳐 보세요.

---

## 막혔을 때

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| `python --version`에 `Python `만 출력 | Microsoft Store 가짜 실행 파일 | `winget install Python.Python.3.12` 후 VS Code 완전 재시작 |
| 설치 후에도 명령어를 못 찾음 | PATH가 아직 갱신되지 않음 | VS Code를 완전히 종료 후 다시 실행 |
| `pip install uv` 연결 오류 | 네트워크·환경 문제 | `winget install --id=astral-sh.uv -e` |
| `ImportError: FastMCP` | mcp 2.x에서 이름 변경 | `MCPServer`로 바꾸거나 `uv add "mcp[cli]<2"` |
| `mcp dev`가 실행되지 않음 | Node.js 없음 | Node.js LTS 설치 |
| 도구가 안 보이거나 파일을 못 찾음 | `server.py`가 `src/` 안에 있음 | 프로젝트 맨 바깥으로 이동 |
| mcp.json에 Start가 안 보임 | 파일 미저장(탭의 ●) | Ctrl+S |
| Start 후 Error, 출력은 비어 있음 | 채팅 세션이 아직 없음 | 채팅에 메시지를 먼저 보내고 다시 Start |

## 참고

- MCP 공식 문서: https://modelcontextprotocol.io
- VS Code MCP 서버 가이드: https://code.visualstudio.com/docs/agent-customization/mcp-servers
- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
