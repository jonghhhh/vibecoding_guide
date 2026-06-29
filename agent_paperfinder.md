# AI와 함께하는 논문 검색·정리 (기초)

### Claude Code의 CLAUDE.md · 서브에이전트 · 스킬 · MCP를 가장 간단하게 경험하기

> 대상: 코딩 경험이 거의 없는 연구자·학생
> 도구: Claude Code (터미널 기반 AI 코딩 에이전트) + paper-search-mcp (논문 검색·PDF 다운로드 MCP)
> 목표: 검색어 한 마디로 관련 논문을 모아 → 관련도 선별 → 보기 좋은 HTML 한 장으로 정리. 그 과정에서 Claude Code의 네 기능을 직접 경험한다.
> 핵심 원칙: 파이썬 코드 0줄. AI가 찾고 만들고, 최종 확인은 사람이 한다.

---

## 1. 무엇을 만들고, 무엇을 경험하나

검색어 한 마디면 다음 순서로 진행됩니다.

```text
"<검색어>" 한 마디
   │
   ▼ 1. 논문 수집      → researcher 서브에이전트 (웹검색 + paper-search-mcp, PDF 다운로드)
   │
   ▼ 2. 관련도 선별    → 메인 세션이 직접 (가장 관련 높은 10~15편만)
   │
   ▼ 3. HTML 정리      → output/report.html (make-report 스킬: 인용 + 초록 + 링크)
```

경험하는 네 기능은 다음과 같습니다.

- CLAUDE.md: 프로젝트의 규칙·작업 순서·인용 규칙을 담아 두면, 매번 자동으로 적용됩니다.
- MCP(paper-search-mcp): 논문 검색과 PDF 다운로드를 해 주는 외부 도구입니다. 우리가 만들지 않고 **연결**만 합니다.
- 서브에이전트(researcher): 검색이라는 '시끄러운' 작업을 별도의 독립된 문맥에서 처리해, 메인 대화를 깨끗하게 유지합니다.
- 스킬(make-report): HTML로 정리하는 절차를 담아 두면, 정리 단계에서 자동으로 그 절차가 적용됩니다.

> 왜 검색을 서브에이전트로 두나요? 검색 한 번에 웹검색 수십 번 + MCP 호출 + PDF 다운로드로 메시지가 폭증합니다. 이 과정을 서브에이전트(독립 문맥)에 맡기면, 메인 대화는 결과 '목록'만 돌려받아 깔끔해집니다. 그래서 'Claude 자체 검색'은 서브에이전트로 두는 것이 좋습니다.

관련도 선별은 별도 파일 없이 메인 세션(기본 Claude)이 직접 합니다. 서브에이전트가 모아 온 목록을 보고, 주제에 가장 맞는 것만 고릅니다.

---

## 2. 폴더 구조 (최대한 간결하게)

```text
paper-finder/
├── CLAUDE.md          # 규칙 + 작업 순서 + 인용 규칙
├── .mcp.json          # MCP 연결 (paper-search-mcp)
├── output/            # 결과물 (report.html + 받은 PDF)
└── .claude/
    ├── agents/
    │   └── researcher.md             # 서브에이전트 1개 (논문 수집)
    └── skills/
        └── make-report/SKILL.md      # 스킬 1개 (HTML 정리)
```

파일은 사실상 CLAUDE.md, .mcp.json, 서브에이전트 1개, 스킬 1개 — 이렇게 네 개뿐입니다. 파이썬 파일도, 가상환경(.venv)도 없습니다.

---

## 3. 준비

### 3-1. Claude Code 실행

설치가 끝났다면(npm install -g @anthropic-ai/claude-code 후 Anthropic 계정 로그인), 폴더를 만들고 그 안에서 실행합니다.

```bash
mkdir paper-finder && cd paper-finder
claude
```

### 3-2. uv 설치 (paper-search-mcp 실행용)

paper-search-mcp는 `uvx` 명령으로 실행됩니다. `uvx`는 **설치 없이 그 자리에서 받아 실행**해 주므로 가상환경이 필요 없습니다. `uvx`를 쓰려면 `uv`만 한 번 깔면 됩니다.

```powershell
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
```cmd
# Windows (Command Prompt / cmd)
powershell -ExecutionPolicy ByPass -Command "irm https://astral.sh/uv/install.ps1 | iex"
```
```bash
# macOS / Linux (Bash / Zsh)
curl -LsSf https://astral.sh/uv/install.sh | sh
```

새 터미널에서 `uv --version`이 찍히면 성공입니다.

> **설치가 막히면 Claude Code에 맡기세요.** 위 명령이 권한·실행정책·PATH 문제로 잘 안 되면, Claude Code 안에서 이렇게 부탁하면 됩니다: **"uv가 안 깔려. 내 OS(Windows)에 맞게 uv를 설치하고, `uv --version`으로 확인까지 해줘."** Claude Code가 알맞은 설치 명령을 직접 실행하고, 설치 후 `uvx.exe`의 전체 경로까지 찾아 줍니다(이 경로는 바로 아래 `.mcp.json`에 필요합니다).

---

## 4. 설정 파일 만들기

아래 내용을 그대로 저장합니다. (Claude Code 안에서 "아래 내용으로 OO 파일을 만들어 줘"라고 시켜도 됩니다.)

### 4-1. CLAUDE.md

```markdown
# 논문 검색 → HTML 정리 프로젝트 규칙

## 역할
- 너는 학술 리서치 도우미다. 메타데이터·초록을 지어내지 않는다. 모르면 비워 둔다.
- 모든 설명은 한국어로 한다.

## 입력/출력
- 입력: 사용자가 주는 검색어(국문/영문)
- 출력: output/report.html  (받은 PDF도 output/에)

## 작업 순서 ("OO 논문 찾아서 정리해줘"라고 하면 이 순서대로)
1. researcher 서브에이전트로 관련 논문을 수집한다(웹검색 + paper-search-mcp, 가능하면 PDF 다운로드).
2. 돌아온 목록에서 주제에 가장 관련 높은 10~15편을 고른다. 곁가지는 버린다.
3. make-report 스킬로 output/report.html 을 만든다.
각 단계마다 무엇을 했는지 한 줄로 보고한다.

## 인용·표기 규칙
- 영문 논문: APA 7th.
- 한국 논문: 저자는 한글 성명만(영문 병기 제거), 한국언론학보식(저자들을 가운뎃점 ·으로 구분, 제목 무표시).
- 각 논문에 제목·저자·연도·출처(저널)·초록·링크(PDF 우선, 없으면 사이트)를 담는다.

## 금지
- 없는 논문, 가짜 초록, 추측한 메타데이터. 모르면 비운다.
```

### 4-2. .mcp.json  (MCP 연결)

> **paper-search-mcp란?** arXiv·PubMed·bioRxiv·medRxiv·Google Scholar 등 **여러 학술 데이터베이스를 한 번에 검색**하고, 오픈액세스 논문은 **PDF까지 내려받아 주는** 오픈소스 MCP 서버입니다. 우리가 코드를 짜지 않고 **연결만** 하면, Claude가 이 도구로 검색·다운로드·전문 읽기를 대신합니다.
>
> 우수한 점:
> - **다중 소스 통합 검색** — 소스마다 따로 들어갈 필요 없이 한 도구로 arXiv·PubMed·Google Scholar 등을 함께 훑습니다. 특히 **Google Scholar 검색**이 강력해, 해외 저널은 물론 **국내 논문(DBpia·학교 리포지터리 등)도 잘 잡힙니다**(이번 '의제설정' 수집에서 가장 많이 건진 소스).
> - **PDF 자동 다운로드** — 오픈액세스(arXiv·PubMed Central 등) 논문은 `output/` 폴더로 바로 받아 줍니다.
> - **전문 읽기** — 받은 논문 본문에서 초록·내용을 직접 읽어 와, 초록을 지어내지 않고 원문 그대로 옮길 수 있습니다.
> - **무료·키 불필요** — 기본 검색에 별도 API 키가 없어도 되고(이메일만 입력), `uvx`로 설치 없이 그 자리에서 실행됩니다.

```json
{
  "mcpServers": {
    "paper-search-mcp": {
      "command": "C:\\Users\\user\\.local\\bin\\uvx.exe",
      "args": ["--from", "paper-search-mcp", "python", "-m", "paper_search_mcp.server"],
      "env": {
        "PAPER_SEARCH_MCP_UNPAYWALL_EMAIL": "your@email.com"
      }
    }
  }
}
```

`uvx`로 paper-search-mcp 서버를 띄워 연결한다는 뜻입니다. 이메일만 본인 것으로 바꾸세요(무료 오픈액세스 PDF 검색에 쓰입니다).

> **Windows에서 실제로 통한 설정입니다.** 위 예시처럼 두 가지를 권장합니다.
> 1. **`command`는 `uvx.exe`의 전체 경로**로 적습니다(예: `C:\\Users\\<사용자명>\\.local\\bin\\uvx.exe`). PATH에 잡히지 않아 `uvx`만으로는 서버가 안 뜨는 경우가 많습니다. uv 설치 위치를 모르면 Claude Code에 "uvx.exe 전체 경로를 찾아서 .mcp.json에 넣어줘"라고 부탁하세요. (macOS/Linux는 보통 `uvx`만으로도 동작합니다.)
> 2. **`args`는 `["--from", "paper-search-mcp", "python", "-m", "paper_search_mcp.server"]`** 형태가 안정적입니다. `["paper-search-mcp"]` 단순형은 실행 진입점을 못 찾아 실패하는 경우가 있어, 모듈을 직접 실행(`python -m paper_search_mcp.server`)하도록 명시했습니다.
> `\\`(역슬래시 두 개)는 JSON에서 경로 구분자를 쓰는 정상 표기입니다.

### 4-3. 서브에이전트 — .claude/agents/researcher.md

```markdown
---
name: researcher
description: 검색어로 관련 학술 논문을 수집할 때 사용. 웹검색과 paper-search-mcp로 논문을 찾고 가능하면 PDF를 받아, 정리된 목록을 돌려준다.
---
너는 학술 검색 담당이다. 받은 검색어로 관련 논문을 모아 '정리된 목록'만 돌려준다.

순서:
1. 먼저 WebSearch로 주제·핵심 논문·표준 용어(국문/영문)를 파악한다.
2. paper-search-mcp 도구로 관련 논문을 검색한다(여러 소스).
3. 받을 수 있는 논문은 paper-search-mcp로 PDF를 output/ 폴더에 내려받는다.
4. 관련도 높은 논문 위주로, 각 논문의 제목·저자·연도·저널·초록·링크·PDF경로를 정리한다.
5. 한국 논문은 저자를 한글로, 영문 병기는 뺀다. 모르는 값은 비운다(추측 금지).

결과는 번호를 매긴 논문 목록으로만 보고한다. HTML은 만들지 않는다(메인 세션 몫).
```

> tools 줄을 적지 않으면 서브에이전트가 메인 세션의 모든 도구(웹검색·MCP 포함)를 그대로 씁니다. 그래서 researcher는 paper-search-mcp를 바로 쓸 수 있습니다.

### 4-4. 스킬 — .claude/skills/make-report/SKILL.md

```markdown
---
name: make-report
description: 수집된 논문 목록을 보기 좋은 HTML 한 장으로 정리할 때 사용한다. 카드마다 인용·초록·링크를 넣는다.
---
# 논문 → HTML 정리 절차
1. 입력은 researcher가 모은 논문 목록이다. 목록에 있는 사실만 쓴다(없는 값은 비운다).
2. 관련도 높은 10~15편만 카드로 만든다.
3. 카드마다 다음을 넣는다.
   - 인용: 영문=APA 7th, 한국=한국언론학보식(저자들을 가운뎃점 ·으로, 영문 병기 제거).
   - 제목 / 초록(없으면 "초록 없음").
   - 하단 링크: PDF(다운로드된 파일 우선, 없으면 OA URL) · 사이트 · DOI.
4. 상단에 검색어·날짜·총 건수를 넣는다. CSS는 파일 안에 인라인으로 넣어 보기 좋게.
5. output/report.html 한 파일로 저장한다.
```

### 4-5. .claude/settings.local.json  (권한·MCP 활성화 — 대부분 자동 생성)

이 파일은 **직접 만들 필요가 없습니다.** `/mcp`로 서버를 켜고, 도구 권한을 물을 때 **허용(allow)** 을 누르면 Claude Code가 알아서 아래 같은 내용을 채워 줍니다. (개인 설정이라 보통 git에 올리지 않습니다.)

```json
{
  "enabledMcpjsonServers": [
    "paper-search-mcp"
  ],
  "permissions": {
    "allow": [
      "mcp__paper-search-mcp__read_arxiv_paper"
    ]
  }
}
```

- `enabledMcpjsonServers`: `.mcp.json`에 적은 서버 중 **이 프로젝트에서 켤 서버** 목록입니다. `/mcp`에서 서버를 활성화하면 여기에 추가됩니다.
- `permissions.allow`: 매번 묻지 않고 **자동 허용할 도구** 목록입니다. 위 예시는 arXiv 논문 읽기 도구를 허용해 둔 상태입니다. 권한 창에서 "이번 세션 항상 허용"을 고르면 `mcp__paper-search-mcp__search_google_scholar` 같은 항목이 이 목록에 더 쌓입니다.

> 권한을 손으로 미리 넣어 두고 싶으면 Claude Code에 "paper-search-mcp 검색·다운로드 도구를 settings.local.json에서 자동 허용으로 추가해줘"라고 부탁해도 됩니다.

저장한 뒤 Claude Code에서 `/mcp`를 입력해 paper-search-mcp가 연결됐는지 확인하세요. 처음 도구를 쓸 때 권한을 물으면 **허용(allow)** 을 누릅니다.

---

## 5. 실행

준비가 끝나면 한 마디면 됩니다. `<검색어>`만 바꾸세요.

```text
"<검색어>" 논문 찾아서 HTML로 정리해줘. CLAUDE.md 순서대로 수집 → 선별 → report.html 까지 진행하고, 단계마다 한 줄로 보고해.
```

기능을 하나씩 체감하고 싶을 때는 단계를 끊어서 시키면 됩니다.

```text
researcher 서브에이전트로 "<검색어>" 관련 논문을 모아줘.
```

```text
방금 모은 목록에서 관련도 높은 12편을 골라, make-report 스킬로 output/report.html 을 만들어줘.
```

큰 작업 전에 Shift+Tab으로 플랜 모드에 들어가면, 어떤 서브에이전트·스킬이 언제 작동하는지 계획으로 먼저 보여 주어 교육용으로 좋습니다.

예시 검색어: `생성형 AI 저널리즘`, `뉴스 품질 평가`, `large language model fact-checking`

---

## 6. PDF에 대해

paper-search-mcp가 받는 PDF는 대부분 오픈액세스(arXiv·PubMed Central 등)입니다. 유료(페이월) 논문은 직접 받을 수 없고, 그 경우 사이트·DOI 링크가 남습니다(정상). 강의 전에 검색어 한두 개로 미리 돌려, 몇 편이나 PDF가 받아지는지 확인해 두면 안전합니다.

한국 논문은 해외 소스 중심인 paper-search-mcp에서 적게 잡힐 수 있습니다. 이럴 때는 "웹검색으로 한국 논문을 더 찾아서 추가해줘"라고 말하면, Claude 자체 검색이 보완합니다.

---

## 7. 잊지 말 것

- 파이썬 파일·가상환경(.venv)은 0개입니다. 우리가 만든 건 설정 파일 네 개뿐이고, 검색·다운로드·HTML 작성은 모두 Claude가 합니다.
- MCP가 `/mcp`에 안 보이면 `uv --version`을 먼저 확인하세요(uv가 있어야 `uvx`가 동작합니다). 그래도 안 되면 Claude Code를 재시작합니다.
- 윤리: 초록·인용·수치는 사람이 1차 출처로 다시 확인합니다. AI는 찾기와 초안을 돕는 도구이며, 최종 책임은 사람에게 있습니다.

> 도구 명령·기본 모델은 2026년 6월 기준이며 자주 바뀝니다. 막히면 Claude Code에서 /help 를 먼저 확인하세요.

AI와 함께하는 논문 검색·정리 입문 슬림판 · 도구 버전·모델은 2026년 6월 기준이며 자주 바뀝니다.
