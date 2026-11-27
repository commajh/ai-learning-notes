# 공개 MCP로 웹 검색과 기술 문서 조회하기

웹에서 최신 정보를 찾거나 제조사 데이터시트에서 특정 사양을 확인할 때는 출처에 따라 MCP를 선택한다. 검색 API 키 없이 로컬 PC의 Chrome 또는 Edge를 사용하려면 **Playwright MCP**가 적합하다. 부품 번호를 알고 있다면 **제조사 문서 전용 MCP**로 공식 PDF를 찾고, PDF 링크나 파일을 이미 확보했다면 **MarkItDown MCP**로 본문을 읽을 수 있다.

## 웹 검색 API를 호출하는 MCP

아래 프로젝트의 MCP 서버 코드는 MIT 라이선스다. 실행 시 사용하는 검색 서비스 또는 인스턴스는 프로젝트마다 다르다.

| MCP | 기능 | 실행 조건 | 출처 |
| --- | --- | --- | --- |
| Brave Search MCP | 웹·뉴스 검색, `freshness`를 이용한 기간 필터 | Brave Search API 키 필요 | [GitHub](https://github.com/brave/brave-search-mcp-server) |
| Tavily MCP | 웹 검색, URL 본문 추출, 사이트 크롤링 | 로컬 서버는 Tavily API 키 사용. 원격 서버는 API 키 또는 OAuth 지원 | [GitHub](https://github.com/tavily-ai/tavily-mcp) |
| Firecrawl MCP | 검색, 검색 결과의 Markdown 본문 수집, 사이트 크롤링 | 클라우드 API에는 키 필요. 자체 호스팅 API를 `FIRECRAWL_API_URL`로 연결 가능 | [GitHub](https://github.com/firecrawl/firecrawl-mcp-server) |
| SearXNG MCP | 여러 검색 엔진의 결과를 모아 검색하고 URL을 텍스트나 Markdown으로 읽기 | 검색 서비스 API 키 없이 구성 가능. JSON 검색을 활성화한 SearXNG 인스턴스 필요 | [GitHub](https://github.com/ihor-sokoliuk/mcp-searxng) · [SearXNG 검색 API](https://docs.searxng.org/dev/search_api.html) |

## 로컬 Chrome 또는 Edge로 검색하기

Playwright MCP는 로컬에서 브라우저를 실행해 Google이나 Bing의 검색 화면을 조작한다. 브라우저에서 검색 결과와 원문을 읽으면, MCP 호스트의 AI가 날짜 필터링, 중복 제거, 요약 및 Markdown 표나 JSON 출력을 처리한다. [Playwright MCP](https://github.com/microsoft/playwright-mcp) · [브라우저 옵션](https://github.com/microsoft/playwright.dev/blob/main/mcp/configuration/options.mdx)

Node.js 20 이상과 Edge가 설치된 PC에서 `mcpServers` 형식을 사용하는 클라이언트의 설정에 다음 항목을 추가한다. Chrome을 사용하려면 `--browser=chrome`을 지정한다. `--isolated`는 브라우저 세션에 임시 프로필을 사용한다. [Playwright MCP 설치 및 프로필 설정](https://github.com/microsoft/playwright-mcp)

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest", "--browser=msedge", "--isolated"]
    }
  }
}
```

> Edge에서 최근 30일간의 Playwright 릴리스를 검색해 줘. 공식 릴리스 페이지에서 게시일과 변경 사항을 확인하고, 중복을 제거한 뒤 날짜·버전·주요 변경 사항·원문 링크를 Markdown 표로 정리해 줘.

출력 형식이 요청마다 달라지면 프롬프트에서 지정한다. 같은 JSON 스키마나 CSV를 반복해서 생성한다면 검색·추출·정렬 규칙을 전용 MCP 도구에 고정할 수 있다.

## 제조사 데이터시트와 개발 문서 조회하기

| MCP | 기능 | 실행 및 인증 | 출처 |
| --- | --- | --- | --- |
| Electronics Docs MCP | TI, STMicroelectronics, Analog Devices의 부품 번호로 공식 PDF를 찾고, 로컬 SQLite에 색인해 페이지 단위로 조회 | 로컬 `stdio` 설치 예시에 API 키 설정이 없음. 공개 저장소의 라이선스는 확인되지 않음 | [GitHub](https://github.com/flaco-source/mcp-docs) |
| PCB Parts MCP | 제조사 부품 번호나 LCSC 코드로 부품 사양과 데이터시트 링크 조회 | 공개 서버는 키 없이 사용 가능. 자체 호스팅에서 Mouser·DigiKey 조회를 사용하려면 각 서비스 자격 증명 필요 | [GitHub](https://github.com/Averyy/pcbparts-mcp) |
| MarkItDown MCP | 이미 확보한 PDF URL이나 로컬 파일을 Markdown으로 변환 | 로컬 실행. 기본 PDF 변환은 API 키 없이 사용 가능 | [GitHub](https://github.com/microsoft/markitdown/tree/main/packages/markitdown-mcp) |
| Microsoft Learn MCP | Microsoft 공식 기술 문서 및 코드 샘플 검색, 문서 본문 조회 | 인증 없이 연결하는 원격 서버 | [Microsoft Learn](https://learn.microsoft.com/en-us/training/support/mcp) |
| Context7 | 라이브러리·프레임워크·SDK의 버전별 문서와 코드 예제 조회 | Context7 서비스에 연결. 공식 가이드의 로컬 `stdio` 설정은 API 키 인증을 사용 | [설정 문서](https://github.com/upstash/context7/blob/master/docs/resources/all-clients.mdx) |

### 최신 데이터시트 확인 순서

1. 정확한 제조사 부품 번호로 제품 페이지의 문서 목록을 확인한다. Electronics Docs MCP가 지원하는 제조사라면 `search_docs`로 PDF 링크를 찾을 수 있다. 그 외 제조사는 Playwright MCP로 공식 제품 페이지를 열어 확인한다.
2. 제조사 제품 페이지의 문서 목록과 PDF에 표시된 **문서 번호·리비전·개정일**을 대조한다. 로컬 색인을 사용했다면 제조사 페이지에서 현재 게시본을 확인한다.
3. Electronics Docs MCP의 `read_doc`와 `read_doc_page`, 또는 MarkItDown MCP로 필요한 본문을 읽는다. 사양에는 단위, 시험 조건, 원문 URL 및 페이지 번호를 함께 기록한다. PDF 변환 중 표 구조가 흐트러졌다면 원문 표에서 수치와 조건을 확인한다.

> STM32 부품 번호에 해당하는 ST 공식 데이터시트를 찾아 줘. 제조사 제품 페이지와 PDF의 리비전 및 개정일을 대조하고, 동작 전압 범위를 추출해 줘. 조건과 단위, 원문 URL, PDF 페이지 번호를 함께 적어 줘.
