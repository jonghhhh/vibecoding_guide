# AI와 함께하는 엔드투엔드 기사 작성

### Claude Code의 CLAUDE.md · 서브에이전트 · 스킬 활용 (PDF 자료 대응)

> 대상: 코딩 경험이 적은 기자
> 도구: Claude Code (터미널 기반 AI 코딩 에이전트)
> 목표: 보도자료·리포트·공시 같은 원자료(대부분 PDF)를 넣으면 PDF 변환 → 핵심 추출 → 검색 보완 → 내 문체로 기사 작성 → 사실 검증까지 한 흐름으로 진행한다.
> 핵심 원칙: AI는 보조 기자이며, 최종 판단과 책임은 사람 기자에게 있다.

---

## 1. 이 실습이 만드는 흐름

자료 여러 개(주로 PDF)를 넣고, 다음 단계를 한 프로젝트 안에서 이어서 진행합니다.

```text
[원본 PDF]  보도자료 · 애널리스트 리포트 · 공시 · 보고서
   │
   ▼ 0. PDF 변환        → sources_md/*.md          (표·다단·스캔까지 살린 마크다운)
   │
   ▼ 1. 핵심 추출        → output/fact_ledger.md     (사실 대장)
   │
   ▼ 2. 검색 보완        → output/research_notes.md  (URL 출처 포함)
   │
   ▼ 3. 기사 작성        → output/drafts/article_v1.md (내 문체)
   │
   ▼ 4. 팩트체크(검증)   → output/factcheck_report.md  (문장별 판정)
   │
   ▼ 5. 수정 반영        → output/drafts/article_v2.md (변경 요약 포함)
```

이 실습의 목적은 단순히 기사를 빨리 뽑는 것이 아니라, Claude Code의 세 가지 기능을 직접 체감하는 것입니다.

---

## 2. 핵심 아이디어 — 사실 대장(fact ledger)

이 방법의 중심에는 사실 대장이 있습니다. 원자료에서 뽑은 모든 사실을 출처에 묶어 표로 정리한 중간 산출물입니다.

사실 대장이 있으면, 기사에 들어간 모든 문장이 "어느 자료의 어느 부분에서 왔는지" 추적됩니다. 그래서 마지막 검증 단계가 의미를 갖고, 근거 없는 문장(이른바 환각)을 걸러낼 수 있습니다. 기자 입장에서는 "AI가 지어낸 것"과 "자료에 근거한 것"을 명확히 구분하는 장치입니다.

PDF가 입력일 때 특히 중요한 점이 있습니다. PDF는 표·다단·스캔 때문에 텍스트가 흐트러지기 쉽습니다. 그래서 사실 대장의 모든 수치는 "변환본"이 아니라 "원본 PDF의 페이지"까지 출처로 적어, 나중에 사람이 원문으로 되짚을 수 있게 합니다.

---

## 3. 세 기능과 단계의 연결

| 단계 | 하는 일 | 체감하는 기능 |
|---|---|---|
| 0. 규칙·문체 | 뉴스룸 원칙과 기자 문체를 매번 자동 적용 | CLAUDE.md |
| 0.5 PDF 변환 | PDF를 표·스캔까지 살려 마크다운으로 | 스킬(pdf-to-md) |
| 1. 핵심 추출 | 변환본에서 사실 대장 작성 | 스킬 + 추출 서브에이전트 |
| 2. 검색 보완 | 빈틈을 웹 검색으로 채움 | 서브에이전트(WebSearch) |
| 3. 기사 작성 | 뉴스가치로 앵글을 정하고 사실 대장만 근거로 내 문체로 작성 | 스킬 + CLAUDE.md 문체 |
| 4. 검토(팩트체크) | 문장별 사실·수치·인용 대조 검증 | 서브에이전트(독립 문맥) |

각 기능을 간단히 정리하면 다음과 같습니다.

- CLAUDE.md: 프로젝트의 "근무 수칙 + 기자 문체". 매 작업마다 자동으로 읽혀 반복 설명이 필요 없습니다.
- 서브에이전트: 추출 담당, 리서치 담당, 검증 담당을 각각 독립된 두뇌(별도 문맥)로 둡니다. 특히 검증자가 작성자의 판단에 오염되지 않는 점이 팩트체크 신뢰의 핵심입니다.
- 스킬: 반복되는 표준 작업 절차(PDF 변환, 사실 대장 만들기, 기사 작성 규칙)를 정리해 두면, 상황에 맞게 적용됩니다.

---

## 4. 사전 준비

### 4-1. Claude Code 실행

설치가 끝났다면(npm install -g @anthropic-ai/claude-code 후 Anthropic 계정 로그인), 작업 폴더를 만들고 그 안에서 실행합니다.

```bash
mkdir agent_newsmaking && cd agent_newsmaking
claude
```

PDF 변환에 파이썬을 쓰므로, 이 폴더 안에 가상환경(.venv)을 만들어 패키지를 격리하는 것을 권합니다. (전역 설치는 버전 충돌·재현성 문제가 생기기 쉽습니다.)

```bash
# 가상환경 생성
python -m venv .venv
# 활성화
source .venv/bin/activate       # macOS/Linux
.venv\Scripts\activate          # Windows(PowerShell)
```

`.venv/` 는 자료·산출물과 함께 공유되지 않도록 `.gitignore` 에 넣어 둡니다.

```bash
printf ".venv/\noutput/\nsources/\n" > .gitignore
```

### 4-2. 폴더 구조

```text
agent_newsmaking/
├── .venv/                          # 파이썬 가상환경 (PDF 변환용, gitignore)
├── .gitignore
├── CLAUDE.md                       # 뉴스룸 규칙 + 기자 문체
├── sources/                        # 원본 PDF (보도자료·리포트·공시)
├── sources_md/                     # PDF에서 변환된 마크다운 (추출은 여기서)
├── style/my-articles/              # 기자의 과거 기사 (문체 학습용)
├── output/                         # 중간 산출물 + 최종 기사
└── .claude/
    ├── agents/                     # 서브에이전트
    │   ├── extractor.md
    │   ├── researcher.md
    │   └── fact-checker.md
    └── skills/                     # 스킬
        ├── pdf-to-md/SKILL.md
        ├── pdf-to-md/convert.py
        ├── extract-key-points/SKILL.md
        └── write-article/SKILL.md
```

원본 PDF는 sources/ 에, 기자의 과거 기사 몇 편은 style/my-articles/ 에 넣어 둡니다. 추출은 변환된 sources_md/ 를 기준으로 하되, 수치·표는 원본 PDF로 교차 확인합니다.

### 4-3. PDF 자료 다루기 (핵심)

PDF는 그대로도 일부 읽히지만, 공시·리포트의 표와 수치, 스캔 문서는 변환 단계를 거쳐야 정확합니다. 두 가지 방법 중 하나를 고르세요.

방법 A. 변환 우선 (권장·기본). PDF를 마크다운으로 먼저 바꿔 sources_md/ 에 둡니다. pymupdf4llm을 쓰면 표·다단·자동 OCR까지 처리됩니다. 위에서 만든 .venv를 활성화한 상태에서 최초 1회만 설치하면 됩니다.

```bash
source .venv/bin/activate            # macOS/Linux (Windows: .venv\Scripts\activate)
pip install -U pymupdf pymupdf4llm
```

방법 B. 대용량·스캔 PDF가 많을 때 (pdf-mcp). 큰 PDF에서 필요한 페이지만 의미·키워드로 찾아 읽고 표·스캔 텍스트를 뽑아 주는 MCP 서버입니다. 한국어 다단도 지원합니다.

```bash
pip install -U pdf-mcp
claude mcp add pdf -- pdf-mcp
```

> 스캔 PDF(텍스트 층이 없는 이미지 PDF)는 OCR이 필요합니다. 한국어 OCR을 위해 tesseract와 한국어 데이터(kor)를 함께 설치하세요. (예: macOS는 brew install tesseract tesseract-lang, Ubuntu는 apt install tesseract-ocr tesseract-ocr-kor)
> 한글(.hwp) 파일은 한컴오피스에서 PDF로 저장한 뒤 위 절차를 따르는 것이 가장 간단합니다.

---

## 5. 설정 파일 만들기

아래 내용을 그대로 해당 경로에 저장합니다. (Claude Code 안에서 "아래 내용으로 CLAUDE.md를 만들어 줘"라고 시켜도 됩니다.)

### 5-1. CLAUDE.md

```markdown
# 뉴스 작성 프로젝트 규칙

## 역할과 원칙
- 너는 우리 뉴스룸의 보조 기자다. 최종 판단과 책임은 사람 기자에게 있다.
- 모든 산출물은 한국어로 작성한다.
- 사실·수치·인용은 절대 지어내지 않는다. 근거가 없으면 "[확인 필요]"로 표시한다.
- 모든 사실은 출처(문서명+페이지 또는 URL)에 연결되어야 한다.
- 추론·해석은 본문에서 [추론] 표기로 사실과 구분한다.

## 입력/출력 위치
- 원본 PDF: sources/
- 변환 마크다운(추출 기준): sources_md/
- 문체 샘플: style/my-articles/
- 산출물: output/

## PDF 자료 취급
- 추출은 sources_md/의 변환본을 기준으로 한다.
- 표·핵심 수치는 변환 오류 가능성이 있으므로 원본 PDF의 해당 페이지로 반드시 교차 확인한다.
- 변환본이 비어 있거나 깨진 문서는 스캔본일 수 있으니 사람에게 알린다.

## 작업 흐름
PDF 변환 → 추출 → 검색 보완 → 기사 작성 → 팩트체크 → 수정.
각 단계 결과는 output/에 파일로 남긴다.

## 내 기사 문체(요약)
- 리드는 한 문장, 핵심 수치를 앞세운다.
- 문장은 짧게, 피동·번역투를 피한다.
- 전문용어는 괄호로 짧게 풀어준다.
- 단정 대신 출처를 밝히는 화법("~에 따르면").
- 상세 문체는 style/my-articles/ 샘플을 우선한다.

## 뉴스가치 판단(기사 작성 시)
- 무엇을 리드로 올리고 무엇을 강조할지는 뉴스가치로 정한다.
- 기준: 새로움, 중요성, 사회적 파장, 유명성, 갈등, 그리고 근접성·이례성.
- 가장 가치 높은 사실을 앞세우되, 가치는 '선택과 강조'에만 쓰고 사실을 과장·왜곡하지 않는다.

## 금지
- 출처 없는 단정, 가짜 인용, 수치 왜곡(단위·기간 누락), 낚시성 제목.
```

### 5-2. 서브에이전트 (.claude/agents)

extractor.md

```markdown
---
name: extractor
description: sources_md/의 변환된 원자료(보도자료·리포트·공시)를 읽고 핵심 사실을 구조화해 추출할 때 사용.
tools: Read, Grep, Glob
model: sonnet
---
너는 자료 추출 전문 기자다. sources_md/의 문서에서 핵심 사실(5W1H), 모든 수치(단위·기준기간·전년比),
직접 인용과 발화자, 각 항목의 출처(문서명+페이지)를 빠짐없이 뽑아라.
수치와 표는 값이 변환 과정에서 어긋날 수 있으니, 의심되면 원본 sources/의 PDF로 확인한다.
문서에 없으면 비워 두고, 지어내지 마라. 결과는 '사실 대장' 표로 정리한다.
```

researcher.md

```markdown
---
name: researcher
description: 추출된 사실의 빈틈(배경·맥락·최신 수치·반론)을 웹 검색으로 보완할 때 사용.
tools: WebSearch, WebFetch, Read, Write
model: sonnet
---
너는 리서치 전문 기자다. 사실 대장에서 보완이 필요한 항목을 골라,
WebSearch로 신뢰할 1차 출처(정부·기관·기업 공식·학술)를 우선 찾고 WebFetch로 확인한다.
보완한 사실마다 URL과 확인 날짜를 달고, 확인 안 되면 [미확인]으로 남긴다.
원자료와 상충하는 내용은 별도 표시한다. 추측으로 메우지 않는다.
```

fact-checker.md

```markdown
---
name: fact-checker
description: 기사 초안의 사실·수치·인용을 사실 대장·리서치 노트·원문에 대조해 검증할 때 사용.
tools: Read, Grep, WebSearch, WebFetch
model: opus
---
너는 깐깐한 팩트체크 데스크다. 기사 초안의 검증 가능한 모든 문장을 하나씩 점검한다.
각 문장: 근거 출처 매칭 → 판정([확인]/[출처 불일치]/[근거 없음]/[오래된 정보]/[과장]).
인용은 원문과 글자 단위로, 수치는 단위·기간·반올림을 대조한다. 수치·표는 원본 sources/의 PDF까지 확인한다.
문제 항목은 무엇이 틀렸는지와 수정안을 함께 제시한다.
결과는 '문장 / 판정 / 근거 / 수정안' 표로 내고, 통과 못한 항목을 먼저 경고한다.
```

fact-checker에는 일부러 Edit·Write 권한을 주지 않았습니다. 검증자는 고치지 않고 지적만 하게 해서, 작성과 검증을 분리합니다.

### 5-3. 스킬 (.claude/skills)

pdf-to-md/SKILL.md

```markdown
---
name: pdf-to-md
description: sources/의 PDF(보도자료·공시·리포트)를 표·다단·스캔까지 살려 마크다운으로 변환할 때 사용한다.
---
# PDF → 마크다운 변환 절차
1. 최초 1회, 프로젝트의 .venv에 변환 도구를 설치한다(설치돼 있으면 건너뜀).
   .venv/bin/python -m pip install -U pymupdf pymupdf4llm
   (Windows: .venv\Scripts\python -m pip install -U pymupdf pymupdf4llm)
2. .venv의 파이썬으로 sources/의 모든 PDF를 sources_md/ 에 .md로 변환한다.
   .venv/bin/python .claude/skills/pdf-to-md/convert.py
   (Windows: .venv\Scripts\python .claude/skills/pdf-to-md/convert.py)
3. 변환 결과를 점검한다.
   - 내용이 비어 있는 파일은 스캔본일 수 있다. tesseract 한국어 OCR 설치 후 다시 변환하거나 사람에게 알린다.
   - 표는 파이프(|) 마크다운으로 변환된다. 병합 셀은 값이 복제될 수 있으니, 핵심 수치는 원본 PDF 페이지와 교차 확인한다.
4. 이후 추출 단계는 sources_md/ 를 기준으로 진행한다.
```

pdf-to-md/convert.py

```python
import pathlib
import pymupdf4llm

SRC = pathlib.Path("sources")
OUT = pathlib.Path("sources_md")
OUT.mkdir(exist_ok=True)

pdfs = sorted(SRC.glob("*.pdf"))
if not pdfs:
    print("sources/ 에 PDF가 없습니다.")

for pdf in pdfs:
    # 다단 레이아웃과 표를 살리고, 필요한 페이지는 자동 OCR
    md = pymupdf4llm.to_markdown(str(pdf))
    target = OUT / (pdf.stem + ".md")
    target.write_text(md, encoding="utf-8")
    status = "OK" if md.strip() else "빈 결과(스캔본일 수 있음 → OCR 필요)"
    print(f"{pdf.name} -> {target.name} [{status}]")
```

extract-key-points/SKILL.md

```markdown
---
name: extract-key-points
description: 보도자료·보고서·애널리스트 리포트·공시(변환본)에서 핵심을 뽑아 '사실 대장'을 만들 때 사용한다.
---
# 사실 대장(fact ledger) 작성 절차
1. sources_md/의 각 문서를 읽는다.
2. 문서별로 핵심 사실을 아래 표에 한 행씩, id를 붙여 정리한다.

| id | 사실 | 유형(사실/수치/인용) | 수치·단위·기간 | 발화자 | 출처(문서·페이지) | 비고(잠정/확정) |
|----|------|------|------|------|------|------|

3. 수치는 반드시 단위·기준기간을 함께 적는다(예: "매출 1.2조원, 2025년 연간, 전년比 +8%").
4. 표에서 뽑은 수치는 원본 PDF 페이지로 한 번 더 확인하고, 출처에 페이지 번호를 적는다.
5. 문서에 없는 내용은 적지 않고, 확인이 필요한 항목은 '검색 필요' 목록으로 따로 뺀다.
6. 결과를 output/fact_ledger.md 로 저장한다.
```

write-article/SKILL.md

```markdown
---
name: write-article
description: 사실 대장과 리서치 노트만 근거로, 내 문체로 기사 초안을 쓸 때 사용한다.
---
# 기사 작성 절차
1. 근거는 output/fact_ledger.md 와 output/research_notes.md 에 있는 것만 쓴다.
2. 뉴스가치를 판단해 무엇을 앞세울지 정한다. 사실 대장의 항목들을 다음 기준으로 평가한다.
   - 새로움(이전에 알려지지 않은 사실인가, 시의성이 있는가)
   - 중요성(영향을 받는 사람·규모가 큰가)
   - 사회적 파장(정책·시장·여론에 미치는 후속 영향이 큰가)
   - 유명성(관련 인물·기관·기업이 저명한가)
   - 갈등(대립·쟁점·이해충돌이 있는가)
   - 그 밖에 근접성(우리 독자와 가까운가), 이례성(드물거나 의외인가)
   가장 강한 뉴스가치를 가진 사실을 리드로 올리고, 기사 전체의 각도(앵글)도 거기에 맞춘다.
3. 구조: 리드(핵심 한 문장, 가장 뉴스가치 높은 사실) → 핵심 사실 → 인용 → 배경 → 전망.
4. 문체는 CLAUDE.md와 style/my-articles/ 샘플을 따른다.
5. 모든 핵심 문장 끝에 근거 id를 단다(예: [S3], [R2]).
6. 근거 없는 문장은 쓰지 않고, 해석은 [추론]으로 표시한다.
7. 뉴스가치를 높이려고 사실을 과장·왜곡하거나 낚시성 제목을 달지 않는다. 가치 판단은 '선택과 강조'에만 쓰고, 사실 자체는 그대로 둔다.
8. output/drafts/article_v1.md 로 저장한다(기본 800자, 지정 시 그 분량).
```

---

## 6. 엔드투엔드 실행 — 프롬프트

준비가 끝나면, Claude Code 입력창에 아래 프롬프트를 순서대로 입력합니다. 대괄호는 그 단계에서 체감하는 기능을 표시한 것입니다.

### 단계 0. PDF 변환 — [스킬(pdf-to-md)]

```text
pdf-to-md 스킬로 sources/ 의 모든 PDF를 sources_md/ 에 마크다운으로 변환해.
변환이 끝나면 파일별 상태(정상/빈 결과)를 알려 주고, 빈 결과가 있으면 스캔본 가능성을 표시해 줘.
```

산출물: sources_md/*.md (표·다단까지 살린 변환본)

### 단계 1. 문체 학습 (선택) — [CLAUDE.md 보강]

```text
style/my-articles/ 의 기사들을 읽고 내 글쓰기 스타일을 분석해,
문장 길이·어휘·리드 방식·인용 처리 규칙으로 정리한 뒤 output/style_guide.md 로 저장해.
CLAUDE.md 문체 요약과 충돌하면 샘플을 우선해.
```

산출물: output/style_guide.md (기자 문체를 규칙으로 정리)

### 단계 2. 핵심 추출 — [스킬 + 서브에이전트]

```text
sources_md/ 의 변환된 자료를 extractor 서브에이전트로 읽고,
extract-key-points 스킬에 따라 사실 대장을 만들어 output/fact_ledger.md 로 저장해.
각 사실에 id와 출처(문서·페이지)를 반드시 표기하고, 표의 수치는 원본 sources/의 PDF로 교차 확인해.
확인이 더 필요한 항목은 '검색 필요'로 따로 빼 줘.
```

산출물: output/fact_ledger.md (출처가 달린 사실 표)

### 단계 3. 검색 보완 — [서브에이전트 + WebSearch]

```text
fact_ledger.md 의 '검색 필요' 항목과 배경·최신 수치를 researcher 서브에이전트에게 맡겨 웹에서 보완해.
보완한 사실마다 URL과 확인 날짜를 달고, 원자료와 상충하면 표시해서
output/research_notes.md 로 정리해.
```

산출물: output/research_notes.md (URL 출처가 달린 보완 사실)

### 단계 4. 기사 작성 — [스킬 + CLAUDE.md 문체]

```text
fact_ledger.md 와 research_notes.md 만 근거로,
write-article 스킬과 내 문체(CLAUDE.md / style_guide.md)로 800자 기사 초안을
output/drafts/article_v1.md 로 써.
먼저 사실들을 뉴스가치(새로움·중요성·사회적 파장·유명성·갈등 등)로 평가해,
가장 가치 높은 사실을 리드와 앵글로 삼아. 단, 가치는 선택·강조에만 쓰고 사실은 과장하지 마.
모든 핵심 문장에 근거 id를 달고, 근거 없는 문장은 쓰지 마.
```

산출물: output/drafts/article_v1.md (근거 id가 달린 초안)

### 단계 5. 팩트체크 — [서브에이전트, 독립 검증]

```text
fact-checker 서브에이전트로 article_v1.md 의 모든 사실·수치·인용을
fact_ledger.md, research_notes.md, 그리고 원문(sources_md/ 와 원본 sources/ PDF)에 대조해 검증해.
문장별 판정·근거·수정안을 표로 만들어 output/factcheck_report.md 로 내고,
통과 못한 항목을 먼저 요약해 줘.
```

산출물: output/factcheck_report.md (문장별 검증 결과)

### 단계 6. 수정 반영

```text
factcheck_report.md 의 지적을 모두 반영해 article_v2.md 를 만들고,
무엇을 왜 고쳤는지 변경 요약을 맨 위에 붙여 줘.
남은 [확인 필요]는 사람 기자 확인용 목록으로 정리해.
```

산출물: output/drafts/article_v2.md (수정본 + 변경 요약)

---

## 7. 운영·검증·윤리 팁

큰 단계 전에는 Shift+Tab으로 플랜 모드에 들어가, AI가 무엇을 할지 계획을 먼저 보여주게 하면 안전합니다. 권한은 처음엔 승인 모드로 두고, 파일 쓰기·검색·스크립트 실행 허용 범위를 직접 확인하세요.

PDF는 변환본을 기준으로 작업하되, 핵심 수치·표·인용은 반드시 원본 PDF 페이지로 교차 확인합니다. 변환본이 비어 있으면 스캔본이므로 OCR을 거쳐야 합니다. 작은 텍스트 PDF는 Claude Code의 Read로 바로 읽을 수도 있지만, 표·수치 정확도와 대용량 처리를 위해 변환을 권장합니다.

검색은 Claude Code 내장 WebSearch/WebFetch로 충분합니다. 더 정밀한 검색이 필요하면 claude mcp add 명령으로 Brave나 Tavily 같은 검색 MCP를 붙일 수 있습니다. 다만 WebFetch는 페이지를 요약해 주는 방식이라, 정확한 인용과 수치는 항상 원문을 기준으로 검증해야 합니다.

윤리 측면에서 두 가지를 강조합니다. 첫째, AI 보조로 작성했다는 사실을 밝힙니다. 둘째, 최종 인용과 수치는 사람 기자가 1차 출처로 다시 확인합니다. AI는 초안과 검증을 돕는 도구일 뿐, 출고 책임은 사람에게 있습니다.

---

## 8. 실습 체크리스트

- [ ] sources/ 에 원본 PDF를 넣었는가
- [ ] 프로젝트 폴더에 .venv를 만들고 활성화했는가
- [ ] (방법 A) .venv에 pymupdf pymupdf4llm을 설치했는가 / (방법 B) pdf-mcp를 등록했는가
- [ ] PDF 변환 후 sources_md/ 에 정상 변환본이 생겼는가 (빈 결과=스캔본 확인)
- [ ] style/my-articles/ 에 과거 기사 3~5편을 넣었는가
- [ ] CLAUDE.md, 서브에이전트 3개, 스킬 3개를 저장했는가
- [ ] 단계 0~6 프롬프트를 순서대로 실행했는가
- [ ] 기사의 모든 핵심 문장에 근거 id가 달려 있는가
- [ ] 표·핵심 수치를 원본 PDF 페이지로 교차 확인했는가
- [ ] factcheck_report에서 통과 못한 항목을 사람이 확인했는가

---

## 9. 자주 막히는 곳

| 증상 | 해결 |
|---|---|
| PDF가 잘 안 읽힘 | pdf-to-md 스킬(pymupdf4llm)로 sources_md/ 에 변환 후 진행 |
| 변환본이 비어 있음(스캔본) | tesseract 한국어 OCR 설치 후 재변환, 또는 docling 모드 사용 |
| 표 수치가 어긋남 | 원본 PDF 페이지로 교차 확인, 복잡한 표는 docling(고정밀)로 변환 |
| 대용량 PDF에서 멈춤·문맥 초과 | pdf-mcp를 붙여 필요한 페이지만 검색·읽기 |
| 한글(.hwp)을 못 읽음 | 한컴오피스에서 PDF로 저장 후 변환 |
| 서브에이전트가 호출되지 않음 | /agents 로 등록 확인, 프롬프트에 에이전트 이름을 직접 지정 |
| 스킬이 적용되지 않음 | 폴더·파일명(.claude/skills/이름/SKILL.md) 확인, description을 상황 중심으로 |
| 검색이 빈약함 | claude mcp add 로 Brave/Tavily MCP 추가 |
| 근거 없는 문장이 보임 | 작성 프롬프트에 "근거 id 없는 문장 금지"를 다시 명시, 팩트체크 재실행 |

> 도구 명령·기본 모델·라이브러리는 2026년 6월 기준이며 자주 바뀝니다. 막히면 Claude Code에서 /help 를 먼저 확인하세요.
