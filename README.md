# MCP 실습 세팅 가이드 (campus-helper)

공개SW실무 · MCP 프로그래밍 발표 실습용 저장소입니다.
내 시간표를 데이터로 넣은 작은 MCP 서버(`campus-helper`)를 Python으로 만들고, VS Code의 AI 채팅이 그 서버의 도구를 호출하는 것까지 해 봅니다.

- 환경: Windows 10/11, VS Code(GitHub Copilot 채팅 사용 가능)
- 소요: 설치 포함 약 20~30분

## 실습 흐름

```
1. 준비·프로젝트 만들기  →  2. server.py 작성  →  3. Inspector로 확인  →  4. VS Code에 연결  →  5. AI에게 질문
```

먼저 서버만 따로 확인(3)하고, 그다음 AI에 연결(4~5)합니다. 문제가 생겼을 때 서버 문제인지 연결 문제인지 나눠서 볼 수 있습니다.

## 이 실습으로 알 수 있는 것

1. MCP 서버는 파이썬 함수 몇 개 + `@mcp.tool()` 만으로 만들 수 있다.
2. AI는 서버가 알려 준 도구 이름·설명·입력 형식만 보고 도구를 고르고 호출한다. 한 질문에 도구 여러 개도 연속으로 쓴다.
3. 서버 단독 확인(Inspector)과 AI 연결 확인(VS Code)을 나눠서 보면 문제를 빨리 찾는다.
4. 예시 데이터에 **내 시간표**를 넣으면, AI가 내 강의실과 시험 일정을 대신 찾아 준다.
5. AI가 접근하는 범위는 서버가 만든 도구로 제한되고, 실행 전에 사용자 승인을 거친다.

---

## 1. 준비 (VS Code 터미널에서)

VS Code를 실행하고 상단 메뉴 **Terminal > New Terminal** 로 터미널을 엽니다.

### 1-1. 설치 확인

```powershell
node --version
python --version
uv --version
```

- `node`: `v18` 이상이 나오면 OK
- `python`: `Python 3.12.x` 가 나오면 OK
- `uv`: 버전이 나오면 OK

### 1-2. 안 되면 이렇게 설치

명령어가 인식되지 않거나 `python --version`이 `Python ` 만 출력하면(Microsoft Store 가짜 실행 파일) 아래를 터미널에 입력하세요.

```powershell
winget install Python.Python.3.12
winget install OpenJS.NodeJS.LTS
winget install --id=astral-sh.uv -e
```

> 설치가 끝나면 **VS Code를 완전히 종료했다가 다시 실행**해야 합니다. 그래야 방금 설치한 명령어를 인식합니다.

## 2. 프로젝트 만들고 VS Code로 열기

VS Code 터미널에서 순서대로 입력합니다.

```powershell
mkdir mcp-lab
cd mcp-lab
uv init
uv add "mcp[cli]"
code .
```

`code .` 를 입력하면 현재 폴더(`mcp-lab`)가 새 VS Code 창으로 바로 열립니다.

**`code .` 가 안 될 때**: VS Code 메뉴 **File > Open Folder** 로 `mcp-lab` 폴더를 직접 선택해서 엽니다.

> 꼭 `mcp-lab` 폴더 자체를 열어야 합니다. 그 위 폴더를 열면 5단계 연결에서 Error가 납니다.

> 참고: `uv init`은 `src/mcp_lab/__init__.py` 구조를 만듭니다. 서버 파일은 `src` 안이 아니라 **프로젝트 맨 바깥(`mcp-lab` 바로 아래)** 에 만드세요.

## 3. server.py 작성

`mcp-lab` 폴더 바로 아래에 `server.py` 파일을 새로 만들고 아래를 붙여 넣습니다.

예시 데이터(`COURSES`)에 들어 있는 과목·강의실·시험 일정은 **작성자의 시간표**입니다. **각자 자기 시간표(과목명, 강의실, 시험 일정 등)로 바꿔 넣으면 내 일정을 알려 주는 서버**가 됩니다.

```python
from mcp.server.mcpserver import MCPServer

mcp = MCPServer("campus-helper")

# 예시 데이터: 본인 시간표로 바꿔 쓰면 된다
COURSES = {
    "공개SW실무": {
        "room": "제5공학관 4층 Y5420",
        "exam": "중간 시험 없음 (발표 10/19)",
    },
    "선형대수학개론": {
        "room": "함박관 5층 Y9503",
        "exam": "시험 10/19(월) 오후 5:30 ~ 7:00, 함박관 Y9010",
    },
    "웹프로그래밍": {
        "room": "제5공학관 4층 Y5445",
        "exam": "시험 10/27(화) 오후 7:00 ~ 7:50, Y5445 (장소 추후 공지)",
    },
    "객체지향프로그래밍1": {
        "room": "제5공학관 4층 Y5401",
        "exam": "시험 10/26",
    },
    "발표와토의": {
        "room": "참조예술관 5층 Y2511",
        "exam": "시험 없음",
    },
    "채플": {
        "room": "60주년채플관 2층 Y22217",
        "exam": "시험 없음",
    },
}


@mcp.tool()
def list_courses() -> str:
    """List the names of all courses in the timetable."""
    return ", ".join(COURSES.keys())


@mcp.tool()
def get_classroom(course: str) -> str:
    """Return the classroom of a course."""
    info = COURSES.get(course.strip())
    return info["room"] if info else "No matching course"


@mcp.tool()
def get_exam_schedule(course: str) -> str:
    """Return the exam schedule (date, time, place) of a course."""
    info = COURSES.get(course.strip())
    return info["exam"] if info else "No matching course"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

핵심은 평범한 함수 위에 `@mcp.tool()` 한 줄을 붙이는 것입니다. 함수 이름, docstring(설명), 매개변수 타입이 AI에게 보이는 "사용 설명서"가 됩니다.

## 4. Inspector로 서버 확인

```powershell
uv run mcp dev server.py
```

브라우저에서 MCP Inspector가 열립니다. (처음 실행할 때 Node 패키지를 내려받아 시간이 걸릴 수 있습니다.)

1. **Connect** 클릭
2. **Tools** 탭으로 이동(목록이 비어 있으면 List Tools 버튼 클릭) → 도구 3개(`list_courses`, `get_classroom`, `get_exam_schedule`)가 보이면 성공
3. `get_exam_schedule`을 선택하고 `course`에 `선형대수학개론` 입력 후 실행 → 결과 `시험 10/19(월) 오후 5:30 ~ 7:00, 함박관 Y9010`

확인이 끝나면 터미널에서 `Ctrl + C`로 종료합니다.

## 5. VS Code에 연결

프로젝트 폴더에 `.vscode` 폴더를 만들고 `.vscode/mcp.json` 파일을 만듭니다.

```powershell
mkdir .vscode
```

```json
{
  "servers": {
    "campus-helper": {
      "type": "stdio",
      "command": "uv",
      "args": ["run", "python", "server.py"]
    }
  }
}
```

파일을 **저장**(Ctrl+S)하면 `"campus-helper"` 위에 **Start / Running / Error** 표시가 나타납니다.

### 꼭 먼저 할 것: 채팅 세션 만들기

Start를 눌렀는데 Error가 나고 출력 로그에 아래 문장이 있다면 서버 문제가 아닙니다.

```
Cannot start an MCP server before the session runtime has been created. Send a message first.
```

Copilot 채팅 창에서 **아무 메시지(예: "안녕")를 한 번 보낸 뒤** 다시 **Start**를 누르면 `Running`이 됩니다.

### 위 설정이 안 될 때 (uv를 못 찾는 경우)

`uv` 명령어를 VS Code가 찾지 못하면 가상환경의 파이썬을 직접 지정합니다.

Windows

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

Mac / Linux: `command`를 `${workspaceFolder}/.venv/bin/python` 으로 바꿉니다.

## 6. AI에게 질문하기

1. Copilot 채팅을 **Agent** 모드로 둡니다.
2. 아래처럼 질문합니다. (본인 시간표의 과목명으로 바꿔도 됩니다.)

```
선형대수학개론 시험 일정이랑 강의실을 같이 알려줘.
```

3. 도구 실행 확인 창이 뜨면 내용을 확인하고 **Allow in this Session**을 누릅니다.
4. AI가 `get_exam_schedule`, `get_classroom` 두 도구를 연속으로 호출하고 답변합니다.

다른 질문도 해 보세요.

```
내 시간표에 있는 과목을 전부 알려줘.
웹프로그래밍 시험은 언제 어디서 봐?
공개SW실무는 시험 있어?
```

호출 과정을 보려면 답변 위의 `Completed 2 steps` 항목을 펼쳐 보세요.

---

## 막혔을 때

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| `python --version`에 `Python `만 출력 | Microsoft Store 가짜 실행 파일 | `winget install Python.Python.3.12` 후 VS Code 완전 재시작 |
| 설치 후에도 명령어를 못 찾음 | PATH가 아직 갱신되지 않음 | VS Code를 완전히 종료 후 다시 실행 |
| `code .` 가 안 됨 | `code` 명령어가 PATH에 없음 | File > Open Folder 로 폴더 열기 |
| `mcp dev`가 실행되지 않음 | Node.js 없음 | `winget install OpenJS.NodeJS.LTS` 후 VS Code 재시작 |
| 도구가 안 보이거나 파일을 못 찾음 | `server.py`가 `src/` 안에 있음 | 프로젝트 맨 바깥으로 이동 |
| mcp.json에 Start가 안 보임 | 파일 미저장(탭의 ●) | Ctrl+S |
| Start 후 Error, 출력은 비어 있음 | 채팅 세션이 아직 없음 | 채팅에 메시지를 먼저 보내고 다시 Start |
| Start 후 Error, 로그에 `uv`를 못 찾는다는 내용 | VS Code가 uv 경로를 모름 | 5단계의 `.venv` 파이썬 경로 방식 사용 |
| Start 후 Error, 파일을 못 찾음 | `mcp-lab`이 아닌 다른 폴더를 열어 둠 | `mcp-lab` 폴더 자체를 열기 |

## 참고

- MCP 공식 문서: https://modelcontextprotocol.io
- VS Code MCP 서버 가이드: https://code.visualstudio.com/docs/agent-customization/mcp-servers
- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk

## 참고

- MCP 공식 문서: https://modelcontextprotocol.io
- VS Code MCP 서버 가이드: https://code.visualstudio.com/docs/agent-customization/mcp-servers
- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
