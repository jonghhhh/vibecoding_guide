# AI와 함께하는 논문 검색·정리

### 검색어 한 마디로 논문을 모아 HTML 한 장으로

> **대상**: 코딩 경험이 거의 없는 연구자·학생<br>
> **도구**: Claude Code + **paper-search-mcp** (논문 검색·PDF 다운로드 MCP)<br>
> **목표**: 검색어 한 마디 → 논문 수집 → 관련도 선별 → **보기 좋은 HTML 한 장**<br>
> **핵심 원칙**: 파이썬 코드 0줄. AI가 찾고 만들고, **최종 확인은 사람**이 합니다.

---

## 1. 무엇을 만드나

검색어 한 마디면 이 순서로 진행됩니다.

```text
"<검색어>" 한 마디
   │
   ▼ 1. 논문 수집      → researcher 서브에이전트 (웹검색 + paper-search-mcp, PDF 다운로드)
   │
   ▼ 2. 관련도 선별    → 메인 세션이 직접 (가장 관련 높은 10~15편만)
   │
   ▼ 3. HTML 정리      → output/report.html (make-report 스킬: 인용 + 초록 + 링크)
```

이 과정에서 심화 가이드의 네 기능을 실제로 써 봅니다.

| 기능 | 하는 일 | 이 실습에서 |
|---|---|---|
| **CLAUDE.md** | 규칙을 매 세션 자동으로 읽힘 | 작업 순서·인용 규칙 |
| **MCP** | 외부 도구를 AI에 연결 | paper-search-mcp (우리가 만들지 않고 **연결만**) |
| **서브에이전트** | 독립된 문맥에서 따로 일함 | 검색이라는 '시끄러운' 작업 담당 |
| **스킬** | 절차를 저장해 두면 자동 적용 | HTML 정리 절차 |

> **왜 검색을 서브에이전트로 두나요?** 검색 한 번에 웹검색 수십 번 + MCP 호출 + PDF 다운로드로 메시지가 폭증합니다. 이 과정을 독립 문맥에 맡기면 **메인 대화는 결과 '목록'만** 돌려받아 깔끔하게 유지됩니다.

---

## 2. 폴더 구조

```text
paper-finder/
├── CLAUDE.md          # 규칙 + 작업 순서 + 인용 규칙
├── .mcp.json          # MCP 연결 (paper-search-mcp)
├── .venv/             # 가상환경 (3-2에서 만듭니다)
├── output/            # 결과물 (report.html + 받은 PDF)
└── .claude/
    ├── agents/
    │   └── researcher.md             # 서브에이전트 1개 (논문 수집)
    └── skills/
        └── make-report/SKILL.md      # 스킬 1개 (HTML 정리)
```

우리가 만드는 건 **설정 파일 네 개**뿐입니다. 파이썬 파일은 한 개도 쓰지 않습니다.

---

## 3. 준비

### 3-1. Claude Code 실행

```bash
# macOS · Linux · WSL
curl -fsSL https://claude.ai/install.sh | bash
```

```powershell
# Windows · PowerShell
irm https://claude.ai/install.ps1 | iex
```

폴더를 만들고 그 안에서 실행합니다.

```bash
mkdir paper-finder && cd paper-finder
claude
```

### 3-2. paper-search-mcp 설치 (pip) {#pip}

`paper-search-mcp`는 **평범한 파이썬 패키지**입니다. 가상환경을 만들고 `pip`로 설치하면 됩니다.

```bash
# ① 가상환경 만들기 (Windows는 python, macOS/Linux는 python3)
python -m venv .venv

# ② 활성화 — Windows PowerShell
.venv\Scripts\Activate.ps1
# ② 활성화 — macOS / Linux
source .venv/bin/activate

# ③ 설치
pip install "paper-search-mcp" "mcp[cli]<2"
```

> ⚠️ **`"mcp[cli]<2"`를 반드시 함께 적으세요.** paper-search-mcp는 MCP 라이브러리 버전을 열어 둔 채 배포됐는데, 그 라이브러리가 2.0으로 올라가면서 내부 경로가 바뀌었습니다. 그냥 `pip install paper-search-mcp`만 하면 실행할 때 `ModuleNotFoundError: No module named 'mcp.server.fastmcp'`가 납니다. 위처럼 버전을 묶어 설치하면 정상 동작합니다.

설치가 됐는지 확인합니다.

```bash
python -c "import paper_search_mcp.server; print('설치 OK')"
```

> ✅ **체크포인트**: 경고 몇 줄이 지나간 뒤 `설치 OK`가 찍히면 성공입니다. (`No CORE API key provided` 같은 경고는 **정상**입니다 — 선택 사항인 유료 소스를 안 쓴다는 안내일 뿐입니다.)

이제 **가상환경 안의 파이썬 전체 경로**를 알아 둡니다. 다음 단계 `.mcp.json`에 넣어야 합니다.

```bash
# Windows PowerShell
(Get-Command python).Source
# macOS / Linux
which python
```

> **막히면 Claude Code에 맡기세요** — "paper-search-mcp를 .venv에 `mcp[cli]<2`와 함께 설치하고, 가상환경 파이썬의 전체 경로를 알려 줘."

> **설치 없이 쓰고 싶다면 (선택)** — [uv](https://astral.sh/uv)를 쓰면 가상환경 없이 그 자리에서 받아 실행할 수 있습니다. 다만 위와 같은 버전 문제가 있어 `--with` 옵션을 함께 줘야 합니다. 이 경우 `.mcp.json`의 `command`는 `uvx`(Windows는 `uvx.exe` 전체 경로), `args`는 아래 명령의 뒷부분이 됩니다.

```bash
uvx --with "mcp[cli]<2" --from paper-search-mcp python -m paper_search_mcp.server
```

---

## 4. 설정 파일 만들기

네 파일을 만듭니다. Claude Code 안에서 "이 문서의 4-1~4-4 내용대로 파일을 만들어 줘"라고 시켜도 됩니다.

### 4-1. `CLAUDE.md`

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
- 정식 절차를 벗어난 경로로 논문 원문을 내려받지 않는다.
```

### 4-2. `.mcp.json` — MCP 연결

`command`에 **3-2에서 알아낸 가상환경 파이썬 전체 경로**를 넣습니다.

```jsonc
// Windows 예시
{
  "mcpServers": {
    "paper-search-mcp": {
      "command": "C:\\Users\\사용자명\\paper-finder\\.venv\\Scripts\\python.exe",
      "args": ["-m", "paper_search_mcp.server"],
      "env": {
        "PAPER_SEARCH_MCP_UNPAYWALL_EMAIL": "your@email.com"
      }
    }
  }
}
```

```jsonc
// macOS / Linux 예시
{
  "mcpServers": {
    "paper-search-mcp": {
      "command": "/Users/사용자명/paper-finder/.venv/bin/python",
      "args": ["-m", "paper_search_mcp.server"],
      "env": {
        "PAPER_SEARCH_MCP_UNPAYWALL_EMAIL": "your@email.com"
      }
    }
  }
}
```

- **경로는 반드시 전체 경로로** 적습니다. 그냥 `python`이라고 쓰면 시스템 파이썬이 잡혀 패키지를 못 찾습니다.
- Windows의 `\\`(역슬래시 두 개)는 JSON에서 경로를 쓰는 정상 표기입니다.
- 이메일만 본인 것으로 바꾸세요. 무료 오픈액세스 PDF를 찾는 데 쓰이며, **API 키는 필요 없습니다.**

저장한 뒤 Claude Code에서 **`/mcp`**를 입력해 연결을 확인합니다. 처음 도구를 쓸 때 권한을 물으면 **허용(allow)**을 누릅니다.

> ✅ **체크포인트**: `/mcp`에 `paper-search-mcp`가 **connected**로 뜨고 도구 수십 개가 보이면 성공입니다.

> **paper-search-mcp란?** arXiv·PubMed·bioRxiv·medRxiv·Google Scholar·OpenAlex·Crossref·DOAJ·Semantic Scholar 등 **여러 학술 데이터베이스를 한 번에 검색**하고, 오픈액세스 논문은 **PDF까지 내려받아 주는** 오픈소스 MCP 서버입니다.
>
> - **다중 소스 통합 검색** — 소스마다 따로 들어갈 필요가 없습니다. 특히 **Google Scholar 검색**이 강력해 해외 저널은 물론 **국내 논문도 잘 잡힙니다.**
> - **PDF 자동 다운로드** — 오픈액세스(arXiv·PubMed Central 등) 논문은 `output/` 폴더로 바로 받아 줍니다.
> - **전문 읽기** — 받은 논문 본문에서 초록을 직접 읽어 와, 지어내지 않고 원문 그대로 옮깁니다.
> - **무료** — 기본 검색에 별도 API 키가 필요 없습니다(이메일만 입력).

### 4-3. 서브에이전트 — `.claude/agents/researcher.md`

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
6. 정식 절차를 벗어난 경로의 다운로드 도구는 쓰지 않는다.

결과는 번호를 매긴 논문 목록으로만 보고한다. HTML은 만들지 않는다(메인 세션 몫).
```

> `tools` 줄을 적지 않으면 서브에이전트가 **메인 세션의 모든 도구**(웹검색·MCP 포함)를 그대로 씁니다. 그래서 researcher는 paper-search-mcp를 바로 쓸 수 있습니다.

### 4-4. 스킬 — `.claude/skills/make-report/SKILL.md`

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

### 4-5. `.claude/settings.local.json` — 만들 필요 없습니다

`/mcp`로 서버를 켜고 권한 창에서 **허용**을 누르면 Claude Code가 알아서 채워 줍니다. 개인 설정이라 보통 git에 올리지 않습니다.

```json
{
  "enabledMcpjsonServers": ["paper-search-mcp"],
  "permissions": {
    "allow": ["mcp__paper-search-mcp__search_google_scholar"]
  }
}
```

- `enabledMcpjsonServers`: 이 프로젝트에서 **켤 서버** 목록.
- `permissions.allow`: 매번 묻지 않고 **자동 허용할 도구** 목록. 권한 창에서 "항상 허용"을 고를 때마다 여기에 쌓입니다.

---

## 5. 실행

`<검색어>`만 바꿔서 한 마디 하면 됩니다.

```text
"<검색어>" 논문 찾아서 HTML로 정리해줘. CLAUDE.md 순서대로 수집 → 선별 → report.html 까지 진행하고, 단계마다 한 줄로 보고해.
```

> ✅ **체크포인트**: `output/report.html`이 생기고, 브라우저로 열었을 때 논문 카드가 10편 이상 보이면 성공입니다.

단계를 끊어서 시키면 어느 기능이 언제 작동하는지 눈으로 볼 수 있습니다.

```text
researcher 서브에이전트로 "<검색어>" 관련 논문을 모아줘.
```

```text
방금 모은 목록에서 관련도 높은 12편을 골라, make-report 스킬로 output/report.html 을 만들어줘.
```

**예시 검색어**: `생성형 AI 저널리즘`, `뉴스 품질 평가`, `large language model fact-checking`

> **큰 작업 전에는 `Shift+Tab`으로 플랜 모드**에 들어가세요. 어떤 서브에이전트·스킬이 언제 작동하는지 계획으로 먼저 보여 줍니다.

---

## 6. 자주 막히는 곳

| 증상 | 해결 |
|---|---|
| `ModuleNotFoundError: No module named 'mcp.server.fastmcp'` | 버전 문제입니다. `pip install "paper-search-mcp" "mcp[cli]<2"`로 다시 설치하세요([3-2절](#pip)) |
| `/mcp`에 서버가 안 보임 | `.mcp.json`의 `command` 경로가 **가상환경 파이썬 전체 경로**인지 확인. 고친 뒤 Claude Code 재시작 |
| 서버가 `failed`로 뜸 | 터미널에서 `python -m paper_search_mcp.server`를 직접 실행해 오류 메시지를 확인하세요 |
| `No CORE API key provided` 경고 | **정상입니다.** 선택 사항인 유료 소스를 안 쓴다는 안내일 뿐, 검색은 됩니다 |
| PDF가 거의 안 받아짐 | 유료(페이월) 논문은 받을 수 없습니다. 링크만 남는 것이 정상입니다 |
| 한국 논문이 적게 잡힘 | "웹검색으로 한국 논문을 더 찾아서 추가해줘"라고 하면 Claude 자체 검색이 보완합니다 |

---

## 7. 잊지 말 것

**파이썬 파일은 0개**입니다. 우리가 만든 건 설정 파일 네 개뿐이고, 검색·다운로드·HTML 작성은 모두 Claude가 합니다.

**PDF는 대부분 오픈액세스**(arXiv·PubMed Central 등)입니다. 유료 논문은 사이트·DOI 링크만 남습니다. 강의 전에 검색어 한두 개로 미리 돌려 몇 편이나 받아지는지 확인해 두면 안전합니다.

> ⚠️ **정식 경로로만 받으세요.** paper-search-mcp에는 정식 구독·오픈액세스 경로를 벗어나 논문을 받는 도구도 섞여 있습니다. 저작권 문제가 있으니 쓰지 마세요. 위 `CLAUDE.md`와 서브에이전트 지침에 이미 금지 문구를 넣어 두었고, 권한 창에서 그런 도구를 물으면 **거부**하면 됩니다.

**윤리**: 초록·인용·수치는 사람이 1차 출처로 다시 확인합니다. AI는 찾기와 초안을 돕는 도구이며, **최종 책임은 사람**에게 있습니다.

> 도구 명령·패키지 버전은 2026년 8월 기준이며 자주 바뀝니다. 막히면 Claude Code에서 `/help`를 먼저 확인하세요.
