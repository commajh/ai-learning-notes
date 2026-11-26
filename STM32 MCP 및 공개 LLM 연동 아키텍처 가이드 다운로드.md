\# 공개 LLM 기반 STM32 MCP 개발 환경 구성 가이드



> 작성일: 2026-11-27  

> 범위: 지금까지 논의한 Python MCP 서버, stdio, Tool 메타데이터/결과, MCP Client/Host, 공개 LLM 연동, SKILL.md 및 오픈소스 Host 선택



\## 1. 목표 및 권장 아키텍처



직접 구현한 `stm32\_mcp` 서버를 공개 LLM과 연결하여 STM32 펌웨어 빌드, 분석, 디버깅 등의 작업을 자연어로 수행한다.



```text

사용자 (CLI / IDE / Web UI)

&#x20;         |

&#x20;         v

MCP Host / Agent (예: OpenCode, Goose, LibreChat)

&#x20; |       - 대화 이력 및 Context 관리

&#x20; |       - SKILL.md 탐색/선택/로딩

&#x20; |       - Tool 목록을 LLM API 형식으로 변환

&#x20; |       - Tool 호출/결과 처리 Agent Loop

&#x20; |

&#x20; +---- HTTP API ----> LLM Server (Ollama / vLLM 등)

&#x20; |

&#x20; +---- MCP Client -- stdio/HTTP --> stm32\_mcp Server

&#x20;                                     |

&#x20;                                     +-- STM32CubeIDE 빌드

&#x20;                                     +-- ELF/메모리 분석

&#x20;                                     +-- ST-Link/디버깅

```



\- \*\*LLM Server\*\*: 모델 추론과 Tool 호출 의도 생성. MCP Tool을 직접 실행하지 않음.

\- \*\*MCP Host\*\*: LLM, 사용자, Skills, MCP Client를 조정하는 애플리케이션.

\- \*\*MCP Client\*\*: Host가 서버별로 생성하는 연결/세션. `initialize`, `tools/list`, `tools/call` 등 처리.

\- \*\*MCP Server\*\*: STM32 작업을 수행하고 Tool, Resource, Prompt를 제공.

\- \*\*Skill\*\*: 언제 어떤 순서로 도구를 사용할지 안내하는 재사용 가능한 작업 지침.



기존 MCP Host를 사용하면 Client와 Agent Loop를 별도로 개발할 필요가 없다. 자체 애플리케이션을 만든다면 Host 로직을 직접 구현해야 한다.



\## 2. Python 패키지 실행: `python -m stm32\_mcp`



예시 구조:



```text

D:\\Projects\\

&#x20; stm32\_mcp\\

&#x20;   \_\_init\_\_.py

&#x20;   \_\_main\_\_.py

&#x20;   server.py

&#x20;   tools\\

&#x20;     \_\_init\_\_.py

&#x20;     build.py

```



`stm32\_mcp/\_\_main\_\_.py`:



```python

from .server import main



if \_\_name\_\_ == "\_\_main\_\_":

&#x20;   main()

```



`stm32\_mcp/server.py`:



```python

from mcp.server.fastmcp import FastMCP



mcp = FastMCP("stm32\_mcp")



@mcp.tool()

def get\_mcu\_info(mcu\_name: str) -> str:

&#x20;   """Return MCU information for the given part number."""

&#x20;   return f"MCU: {mcu\_name}"  # 설명용 가상 결과



def main() -> None:

&#x20;   mcp.run(transport="stdio")

```



실행:



```powershell

cd D:\\Projects

python -m stm32\_mcp

```



다른 작업 디렉터리에서 실행할 경우:



```powershell

$env:PYTHONPATH = "D:\\Projects"

python -m stm32\_mcp

```



\- `-m` 뒤에는 \*\*파일 경로가 아닌 모듈/패키지 이름\*\*을 지정한다.

\- `PYTHONPATH`는 \*\*패키지 검색 경로\*\*이며 프로세스의 현재 작업 디렉터리를 변경하지 않는다.

\- 운영 환경에서는 가상환경의 Python 절대 경로 또는 패키지 설치 방식도 고려한다.



\## 3. stdio 통신의 의미



`mcp.run(transport="stdio")`는 서버가 프로세스의 표준 입출력을 통해 MCP JSON-RPC 메시지를 주고받는다는 의미다.



| 채널 | 역할 |

|---|---|

| stdin | Client → Server 요청 |

| stdout | Server → Client 응답/프로토콜 메시지 |

| stderr | 일반 로그와 진단 출력 |



일반적으로 Host가 `python -m stm32\_mcp`를 \*\*자식 프로세스로 시작\*\*하고 파이프를 연결한다. 따라서 stdio 서버를 수동으로 미리 실행할 필요가 없다. HTTP 방식은 이미 실행 중인 독립 서버에 접속할 수 있다.



\*\*주의:\*\* `print()` 등으로 stdout에 일반 로그를 출력하면 프로토콜을 손상시킬 수 있다. 로그는 stderr에 기록한다.



```python

import logging

import sys

logging.basicConfig(stream=sys.stderr, level=logging.INFO)

```



\## 4. FastMCP와 Tool 등록



`from mcp.server.fastmcp import FastMCP`는 \*\*공식 `mcp` Python SDK의 FastMCP API\*\*를 가져오는 구문이다. 별도 프로젝트인 `from fastmcp import FastMCP`와 혼동하지 않는다. 설치된 SDK 버전에 따라 API가 달라질 수 있으므로 의존성 버전을 고정한다.



```powershell

python -m pip install "mcp>=1,<2"

```



```python

@mcp.tool()

def build\_project(project\_path: str, configuration: str = "Debug") -> dict:

&#x20;   """Build an STM32 firmware project.



&#x20;   Use when the user requests firmware compilation.



&#x20;   Args:

&#x20;       project\_path: Absolute path to the STM32 project.

&#x20;       configuration: Debug or Release.



&#x20;   Returns:

&#x20;       Build status and diagnostics.

&#x20;   """

&#x20;   # 실제 빌드 로직은 별도로 구현해야 한다.

&#x20;   return {"status": "not\_implemented"}

```



\- 함수 이름: Tool 이름의 기본값.

\- \*\*Docstring(`"""..."""`)\*\*: 첫 줄뿐 아니라 여러 줄 설명이 Tool description에 사용될 수 있다. SDK가 `Args`를 파라미터 설명으로 별도 추출할 수 있다.

\- 일반 `#` 주석: Tool description으로 자동 공개되지 않는다.

\- Type Hint, 기본값: 입력 JSON Schema 생성에 활용.

\- `@mcp.tool(description="...")`: Tool 설명을 명시적으로 덮어쓸 수 있다.



Tool 설명에는 \*\*What / When / Input / Constraints / Output\*\*을 분명하게 적는다. Tool의 실제 기능은 Python 구현이 담당하며 Docstring만으로 기능이 생기지 않는다.



\## 5. Tool 정보가 LLM에 전달되는 과정



1\. Host가 MCP Client를 만들고 서버와 `initialize` 및 초기화 완료 절차를 수행한다.

2\. Client가 `tools/list`로 Tool 이름, 설명, `inputSchema` 등을 조회한다.

3\. Host가 이 정보를 LLM 서버의 Tool Calling API 형식으로 변환한다.

4\. LLM은 사용자 요청과 Tool 설명을 보고 적절한 Tool과 인수를 제안한다.

5\. Host가 실행 권한을 검사한 뒤 MCP Client의 `tools/call`로 실행한다.

6\. 서버의 결과가 Host를 거쳐 LLM의 다음 입력으로 전달된다.



MCP 응답의 개념적 예:



```json

{

&#x20; "jsonrpc": "2.0",

&#x20; "id": 3,

&#x20; "result": {

&#x20;   "content": \[{"type": "text", "text": "Build completed"}],

&#x20;   "isError": false

&#x20; }

}

```



LLM은 \*\*Python 소스 자체를 전달받는 것이 아니라\*\*, 공개된 Tool 메타데이터와 실행 결과를 받는다. 실제 LLM 입력에 어떤 정보를 포함할지는 Host 구현과 모델 API에 따라 달라진다.



\## 6. Tool 결과와 오류 설계



가능하면 구조화된 결과를 제공한다. 예:



```json

{

&#x20; "status": "build\_failed",

&#x20; "project": "Haircare",

&#x20; "configuration": "Debug",

&#x20; "build": {"exit\_code": 1, "errors": 2, "warnings": 3},

&#x20; "diagnostics": \[

&#x20;   {

&#x20;     "severity": "error",

&#x20;     "file": "main.c",

&#x20;     "line": 125,

&#x20;     "message": "undefined reference to HAL\_I2C\_Init"

&#x20;   }

&#x20; ]

}

```



\- Tool 반환값은 SDK와 타입 선언에 따라 MCP `content` 및/또는 `structuredContent`로 표현될 수 있다.

\- \*\*작업 실패\*\*(컴파일 실패)는 정상적으로 실행된 Tool의 데이터에서 `status=build\_failed`로 나타낼 수 있다.

\- \*\*Tool 실행 실패\*\*(실행 파일 없음, 파라미터 오류 등)는 `isError=true`로 나타낼 수 있다.

\- `status`, `exit\_code`, 파일/라인, 오류 개수, 단위는 서버에서 확정해 반환한다. LLM이 로그만 보고 성공 여부를 추측하도록 만들지 않는다.

\- 로그가 크면 요약과 진단 목록, 로그 파일 경로를 분리한다. 긴 결과는 Context를 소모한다.



\## 7. MCP Client와 Host의 역할



| 구분 | MCP Client | MCP Host |

|---|---|---|

| 연결 | 서버별 연결/세션 관리 | Client 생성과 수명 관리 |

| 프로토콜 | `initialize`, `tools/list`, `tools/call` | LLM용 Tool 목록으로 변환 |

| 실행 | 서버에 호출 전달, 결과 수신 | 호출 승인, 재시도/제한, 후속 LLM 요청 |

| Context | 자체적으로 대화 추론하지 않음 | 사용자 입력, 대화 이력, Skills, Tool 결과 관리 |



Python SDK로 직접 Client를 만들 때 핵심 API:



```python

from mcp import ClientSession, StdioServerParameters

from mcp.client.stdio import stdio\_client



\# stdio\_client(...) -> ClientSession(...) -> initialize()

\# -> list\_tools() -> call\_tool(...)

```



VS Code, Codex CLI, OpenCode 등 기존 MCP Host를 사용하는 경우 이 코드를 직접 작성할 필요가 없다.



\## 8. 공개 LLM을 직접 구동하는 경우



예시 구성:



\- LLM Server: Ollama 또는 vLLM (Tool Calling 지원 모델 사용)

\- Host: OpenCode, Goose, LibreChat 또는 직접 작성한 Python Agent

\- MCP Server: `stm32\_mcp` (stdio)

\- UI: CLI, IDE 또는 Web



직접 Host를 개발할 경우 핵심은 \*\*Agent Loop\*\*다.



```text

사용자 입력

&#x20; -> LLM API 호출 (Tool 목록 + 대화 Context)

&#x20; -> LLM의 Tool Call 여부 검사

&#x20;    -> Tool Call 있음: 이름/인수 검증 -> 승인 -> MCP tools/call

&#x20;                      -> 결과를 대화에 추가 -> LLM 재호출

&#x20;    -> Tool Call 없음: 사용자에게 최종 응답

```



필수 고려사항: 호출 횟수 제한, 타임아웃, 모델의 Tool Calling 지원 여부, Tool 이름 충돌, 파라미터 검증, 결과 크기 제한, 승인 정책, 대화 이력 관리. LLM API의 Tool Calling 형식은 모델 서버별로 차이가 있을 수 있다.



\## 9. `SKILL.md`와 MCP Tool의 차이



| 항목 | 역할 |

|---|---|

| MCP Tool | 실제 빌드/다운로드/분석 등 실행 기능 |

| SKILL.md | Tool 사용 시점, 작업 순서, 검증 기준을 설명하는 지침 |

| Host | Skill 탐색/선택/로딩, Tool 실행 및 Context 조합 |



예시 위치(OpenCode):



```text

HaircareProject/

&#x20; opencode.json

&#x20; .opencode/

&#x20;   skills/

&#x20;     stm32-build/

&#x20;       SKILL.md

```



예시 `SKILL.md`:



```markdown

\---

name: stm32-build

description: Build STM32 firmware and analyze compilation diagnostics.

\---



\# STM32 Build Workflow



1\. Identify the project and configuration.

2\. Invoke the STM32 build MCP tool.

3\. Check the returned status and exit code.

4\. Analyze errors and warnings with file/line references.

5\. Report results without inventing missing diagnostics.

6\. Request approval before modifying source or flashing hardware.

```



많은 Host는 Skill의 \*\*이름/설명\*\*을 먼저 LLM에 노출하고, 필요할 때 `SKILL.md` 본문을 로드한다. 이는 모든 Skill 본문을 매 요청에 넣는 것과 다르다. 실제 로딩 정책은 Host마다 다르다.



참고로 MCP Skills Extension은 \*\*MCP 서버가 Skill을 공개하는 별도 확장 규격\*\*이다. 일반적인 Host 로컬 디렉터리의 `SKILL.md` 로딩과 구분하며 지원 여부를 확인해야 한다.



\## 10. 오픈소스 MCP Host 후보



| 프로젝트 | 적합한 목적 | 특징/유의사항 |

|---|---|---|

| \*\*OpenCode\*\* | STM32 코딩 Agent | MCP와 `SKILL.md` 지원, 프로젝트 단위 작업에 적합 |

| \*\*Goose\*\* | MCP 중심 Agent 및 구조 연구 | MCP 확장, Agent Skills 지원 |

| \*\*LibreChat\*\* | 웹 UI 및 다중 사용자 | MCP와 Skills 지원; Docker 배포 시 Windows 개발 도구 접근 경로 주의 |

| Continue | IDE 중심 워크플로 | 사용 버전의 Skill 지원 및 MCP 연동 범위 확인 필요 |



공식 문서:

\- \[MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)

\- \[MCP 아키텍처](https://modelcontextprotocol.io/specification/2025-11-25/architecture)

\- \[OpenCode MCP 설정](https://dev.opencode.ai/docs/ko/mcp-servers/)

\- \[OpenCode Skills](https://opencode.ai/docs/skills/)

\- \[Goose Agent Skills](https://goose-docs.ai/docs/guides/context-engineering/using-skills/)

\- \[LibreChat Skills](https://www.librechat.ai/docs/features/skills)

\- \[LibreChat MCP 설정](https://www.librechat.ai/docs/configuration/librechat\_yaml/object\_structure/mcp\_servers)

\- \[MCP Skills Extension](https://modelcontextprotocol.io/extensions/skills/overview)



\## 11. OpenCode 연결 예시



`opencode.json` (로컬 stdio 서버):



```json

{

&#x20; "$schema": "https://opencode.ai/config.json",

&#x20; "mcp": {

&#x20;   "stm32": {

&#x20;     "type": "local",

&#x20;     "command": \["python", "-m", "stm32\_mcp"],

&#x20;     "environment": {

&#x20;       "PYTHONPATH": "D:\\\\Projects"

&#x20;     },

&#x20;     "enabled": true

&#x20;   }

&#x20; }

}

```



별도로 OpenCode의 모델 공급자를 로컬 Ollama/vLLM에 연결하고, \*\*Tool Calling을 지원하는 모델\*\*을 선택해야 한다. Python이 여러 개라면 `command`에 해당 가상환경의 `python.exe` 절대 경로를 지정하는 편이 안전하다.



\## 12. STM32 MCP 개발 시 권장 모듈 분리



```text

stm32\_mcp/

&#x20; \_\_init\_\_.py

&#x20; \_\_main\_\_.py

&#x20; server.py             # MCP 생성/실행

&#x20; tools/

&#x20;   build.py            # LLM에 공개할 인터페이스

&#x20;   flash.py

&#x20;   analyze.py

&#x20; core/

&#x20;   cubeide.py          # 실제 빌드 명령 실행

&#x20;   stlink.py           # 실제 장치 접근

&#x20;   elf\_parser.py

```



Tool 인터페이스와 실제 실행 로직을 분리하면 단위 테스트, CLI 테스트, Host 교체가 쉬워진다.



\*\*안전 정책 권장\*\*

\- 읽기/분석 Tool과 수정/Flash Erase/다운로드 Tool을 구분한다.

\- 프로젝트 경로 허용 범위와 명령 인수를 검증한다.

\- 장치 변경 작업은 승인 절차와 명확한 대상 식별을 요구한다.

\- 빌드/다운로드/검증 결과를 별도 상태값으로 반환한다.

\- 타임아웃, 취소, 실패 복구, 중복 실행 방지를 설계한다.



\## 13. 권장 구축 순서



1\. `stm32\_mcp`에 `\_\_main\_\_.py` 및 `stdio` 실행 경로 추가.

2\. 단일 읽기 전용 Tool 등록 후 MCP Inspector 또는 SDK Client로 `tools/list`/`tools/call` 시험.

3\. Tool Docstring, 입력 스키마, 결과 구조, 오류 상태 정리.

4\. OpenCode에 `stm32\_mcp` 연결 및 Tool 탐색 확인.

5\. Ollama 또는 vLLM의 Tool Calling 지원 모델 연결.

6\. `stm32-build/SKILL.md` 추가 및 선택/로딩 동작 확인.

7\. 실제 빌드 실행, 오류 반환, 후속 LLM 분석까지 End-to-End 검증.

8\. 소스 수정/Flash 다운로드 Tool은 승인과 보호 장치 마련 후 단계적으로 추가.



\## 14. 최종 결론



\- \*\*MCP 서버\*\*는 실제 STM32 기능과 명확한 Tool 인터페이스를 구현한다.

\- \*\*MCP Client\*\*는 Host가 서버별로 생성하며 프로토콜 통신을 수행한다.

\- \*\*MCP Host\*\*는 사용자 입력, LLM, Skill, Tool 결과를 조합하고 Agent Loop를 운영한다.

\- \*\*LLM 서버\*\*는 Tool 호출을 결정하고 결과를 해석하지만 실제 작업은 수행하지 않는다.

\- 현재 목적에는 \*\*OpenCode + 공개 LLM 서버 + `stm32\_mcp` + 프로젝트별 `SKILL.md`\*\* 구성이 가장 간단한 출발점이다.



> \*\*주의:\*\* 예제의 MCU 정보, 빌드 결과 및 경로는 설명용이며 실제 장치/빌드 작업을 수행하지 않는다. SDK와 Host의 버전별 API 및 설정 차이는 적용 시 확인해야 한다.



