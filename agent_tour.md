# 관광지 검색 · 챗봇 · 글쓰기 실습

### 공공데이터 API 하나로 웹페이지 → 챗봇 → 글쓰기 에이전트까지

> **대상**: 입문 과정을 마치고 실제로 쓸 만한 것을 만들어 보고 싶은 사람<br>
> **도구**: Claude Code (Codex·Antigravity CLI도 프롬프트는 동일)<br>
> **목표**: 무료 API 키 두 개로 **웹페이지 → 챗봇 → 글쓰기 에이전트** 세 가지를 차례로 만들기<br>
> **핵심**: 코드는 한 줄도 직접 쓰지 않습니다. **키를 받아 `.env`에 넣고, 프롬프트를 붙여넣는 것**이 전부입니다.

---

## 1. 무엇을 만드나

폴더 하나에서 세 가지를 순서대로 만듭니다. 뒤 단계는 앞 단계 결과물을 재료로 씁니다.

```text
공공데이터포털 관광정보 API
   │
   ▼ 1단계  검색 웹페이지     → tour/web/index.html        (HTML 한 파일, 약 30분)
   │           지역·키워드로 관광지를 찾아 목록으로 보여 줍니다
   │
   ▼ 2단계  챗봇             → tour/chatbot/chatbot.py    (파이썬, 약 30분)
   │           말로 물으면 관광공사 정보를 가져와 Gemini가 답합니다
   │
   ▼ 3단계  글쓰기 에이전트   → tour/writing_agent/result.md  (약 40분)
               키워드 하나로 검색→요약→재미있는 글까지 자동 작성
```

| 단계 | 만드는 것 | 배우는 것 |
|---|---|---|
| 1단계 | 관광지 검색 웹페이지 | 공공 API 키 발급, 문서를 근거로 코드 만들기 |
| 2단계 | 관광 안내 챗봇 | 외부 API + LLM API를 엮기 |
| 3단계 | 글쓰기 에이전트 | 서브에이전트·스킬로 작업을 나누기 |

> 1·2단계 끝에는 **(선택) 배포**가 붙어 있습니다. 남에게 링크로 보여 주고 싶을 때만 하면 되고, 건너뛰어도 실습에는 지장이 없습니다.

---

## 2. 폴더 만들기

원하는 위치에 `tour` 폴더를 만들고, 그 안에 하위 폴더 셋을 만듭니다.

```bash
mkdir tour
cd tour
mkdir web chatbot writing_agent
```

완성되면 이런 모양입니다.

```text
tour/
├── .env                 # 열쇠 보관함 (3장에서 만듭니다)
├── .gitignore
├── web/                 # 1단계 — 검색 웹페이지
├── chatbot/             # 2단계 — 챗봇
└── writing_agent/       # 3단계 — 글쓰기 에이전트
```

VS Code에서 **File → Open Folder**로 `tour` 폴더를 열고, 터미널에서 `claude`를 실행합니다.

> ✅ **체크포인트**: 터미널 경로 끝이 `tour`이고, 왼쪽 탐색기에 `web`·`chatbot`·`writing_agent` 세 폴더가 보이면 준비 완료입니다.

---

## 3. `.env` — 열쇠 보관함

이 실습에서 가장 먼저, 그리고 가장 정확히 해야 하는 일입니다. 여기서 어긋나면 뒤 단계가 전부 막힙니다.

### 3-1. `.env`가 뭔가요?

**API 키(열쇠)를 코드와 분리해 따로 보관하는 파일**입니다. `tour` 폴더 **바로 아래**(최상단)에 둡니다.

- 키를 코드 안에 직접 쓰면 파일을 공유하거나 깃허브에 올리는 순간 **남에게 그대로 넘어갑니다.**
- `.env`에 모아 두면 **이 파일 하나만 공유하지 않으면** 됩니다.
- 키가 바뀌어도 `.env` 한 줄만 고치면 됩니다.

### 3-2. 이 실습에 필요한 키 두 개

| 이름 | 어디서 받나 | 어디에 쓰나 | 비용 |
|---|---|---|---|
| `DATAGOKR_API_KEY` | [공공데이터포털](https://www.data.go.kr/data/15101578/openapi.do) — 한국관광공사 관광정보 | 1·2·3단계 전부 (관광지 정보 조회) | 무료 |
| `GOOGLE_API_KEY` | [Google AI Studio](https://aistudio.google.com/api-key) | 2단계 챗봇 (Gemini 답변 생성) | 무료 등급 있음 |

### 3-3. 파일 양식

`tour/.env` 파일을 만들고 아래 형식으로 씁니다. **`이름=값`을 한 줄에 하나씩**입니다.

```bash
# 공공데이터포털 — 한국관광공사 관광정보 (4장에서 발급)
DATAGOKR_API_KEY=여기에_발급받은_인증키_붙여넣기

# Google AI Studio — Gemini API (5장에서 발급)
GOOGLE_API_KEY=여기에_발급받은_API_키_붙여넣기
```

지켜야 할 규칙은 다섯 가지뿐입니다.

| 규칙 | 맞는 예 | 틀린 예 |
|---|---|---|
| `=` 앞뒤에 **공백을 넣지 않습니다** | `GOOGLE_API_KEY=AIza...` | `GOOGLE_API_KEY = AIza...` |
| **따옴표를 붙이지 않습니다** | `GOOGLE_API_KEY=AIza...` | `GOOGLE_API_KEY="AIza..."` |
| 한 줄에 **하나씩** 씁니다 | 위 예시대로 | 한 줄에 두 개 |
| 이름은 **영문 대문자와 밑줄**만 씁니다 | `DATAGOKR_API_KEY` | `datagokr-api-key` |
| 설명은 `#`로 시작하는 줄에 씁니다 | `# 공공데이터포털 키` | — |

> **파일 이름이 `.env`가 맞는지 확인하세요.** 점(.)으로 시작하는 이름이라 Windows 탐색기에서는 `.env.txt`로 저장되기 쉽습니다. VS Code에서 새 파일을 만들어 `.env`로 저장하는 편이 안전합니다.

### 3-4. `.gitignore` — 실수로 올리지 않게

`tour` 폴더에 `.gitignore` 파일을 하나 더 만듭니다. 여기 적힌 파일은 깃허브에 올라가지 않습니다.

```text
.env
.venv/
__pycache__/
```

> **키가 이미 공개됐다면** 파일을 지우는 것만으로는 부족합니다. 발급처에서 **키를 폐기하고 새로 받으세요.** 공공데이터포털·AI Studio 모두 재발급이 무료입니다.

### 3-5. `.env`를 읽는 법

**파이썬**은 `python-dotenv` 패키지를 씁니다. 가상환경을 켠 상태에서 설치하세요.

```bash
pip install python-dotenv google-genai requests
```

```python
import os
from dotenv import load_dotenv

load_dotenv()                              # tour/.env 를 읽어 온다
KEY = os.getenv("DATAGOKR_API_KEY")        # 이름으로 꺼내 쓴다
```

**HTML 파일**은 `.env`를 읽지 못합니다(브라우저에는 파일 시스템 권한이 없습니다). 그래서 1단계 웹페이지는 키를 HTML 안에 직접 적습니다. 이 점은 [4-5절](#key)에서 다시 다룹니다.

---

## 4. 1단계 — 관광지 검색 웹페이지

### 4-1. 공공데이터포털에서 인증키 받기

1. [공공데이터포털](https://www.data.go.kr)에 접속해 회원가입·로그인합니다.
2. 검색창에 **`한국관광공사 관광정보`**를 입력하고, **한국관광공사_국문 관광정보 서비스_GW**를 엽니다. ([바로가기](https://www.data.go.kr/data/15101578/openapi.do))
3. 오른쪽 **활용신청** 버튼을 누릅니다. 활용목적은 "앱개발(교육용)" 정도로 적으면 됩니다.
4. 승인되면 **마이페이지 → 데이터활용 → 개발계정**에서 **일반 인증키(Encoding / Decoding)** 두 가지를 확인할 수 있습니다.

> **자동 승인이지만 반영에 시간이 걸립니다.** 신청 직후에는 `SERVICE_KEY_IS_NOT_REGISTERED_ERROR`가 뜰 수 있습니다. 보통 수 분에서 한 시간이면 풀립니다. **다른 API에 신청한 키는 이 API에서 쓸 수 없습니다** — 이 API에 별도로 신청해야 합니다.

### 4-2. 활용매뉴얼 내려받기

같은 페이지 아래쪽 **참고문서**에서 **`개방데이터_활용매뉴얼(국문).zip`**을 내려받습니다. 압축을 풀어 나온 문서를 **`tour/web` 폴더에 그대로 저장**합니다.

이 매뉴얼이 있어야 AI가 **어떤 주소로, 어떤 항목을 요청해야 하는지** 정확히 압니다. 없으면 AI가 파라미터를 지어내 엉뚱한 코드를 만듭니다.

```text
tour/web/
└── 한국관광공사_개방데이터_활용매뉴얼.docx   (또는 pdf/hwp)
```

### 4-3. `.env`에 키 저장

3-3절 양식대로 `tour/.env`의 `DATAGOKR_API_KEY` 줄에 인증키를 붙여넣습니다.

### 4-4. 프롬프트

`tour` 폴더에서 `claude`를 실행하고 아래를 그대로 입력합니다.

```text
web 폴더에 한국관광공사_개방데이터_활용매뉴얼 바탕으로 관광지 간단히 검색하는 단일 html 파일 간결하게 생성해
```

AI가 매뉴얼을 읽고 계획을 보여 줍니다. 동의하면 `tour/web/index.html`이 만들어집니다. 브라우저로 열어 확인하세요.

```bash
# macOS
open web/index.html
# Windows
start web/index.html
```

> ✅ **체크포인트**: 검색창에 `경복궁`을 넣었을 때 주소와 사진이 있는 목록이 나오면 성공입니다.

마음에 안 드는 부분은 말로 고칩니다.

```text
지역(시·도)을 고르는 드롭다운도 추가하고, 결과 카드에 사진이 없으면 회색 상자를 대신 보여 줘.
```

### 4-5. 꼭 알아야 할 것 — 인증키 두 종류 {#key}

공공데이터포털은 인증키를 **Encoding**과 **Decoding** 두 가지로 줍니다. 겉보기엔 비슷한데, **쓰는 자리가 다릅니다.** 이걸 바꿔 쓰면 키가 멀쩡해도 **403 `SERVICE_KEY_IS_NOT_REGISTERED_ERROR`**가 뜹니다. 이 실습에서 가장 흔한 사고입니다.

| 어디서 쓰나 | 어떤 키 | 이유 |
|---|---|---|
| **HTML/자바스크립트**에서 주소 문자열에 직접 붙일 때 | **Encoding 키** (`%2B`, `%3D` 같은 기호가 들어 있음) | 이미 인코딩된 상태라 그대로 붙이면 됩니다 |
| **파이썬** `requests`의 `params=`로 넘길 때 | **Decoding 키** (`+`, `=` 기호가 보임) | 라이브러리가 한 번 더 인코딩하므로, 인코딩된 키를 주면 이중 인코딩됩니다 |

> **막혔을 때 한 문장으로 물어보세요** — "403 SERVICE_KEY_IS_NOT_REGISTERED_ERROR가 나. 인코딩 키와 디코딩 키를 바꿔서 시험해 보고 되는 쪽으로 고쳐 줘." AI가 두 경우를 직접 호출해 보고 맞는 쪽으로 고쳐 줍니다.

> **웹페이지에 키가 노출됩니다.** 단일 HTML 파일은 키를 코드 안에 적을 수밖에 없고, 배포하면 누구나 볼 수 있습니다. 이 실습은 **무료·공개 데이터 키**라 교육용으로는 괜찮지만, 유료 API 키는 절대 이렇게 쓰지 마세요. 제대로 하려면 키를 감추는 서버(백엔드)를 따로 둡니다.

### 4-6. (선택) GitHub Pages로 배포하기

만든 페이지를 링크로 공유하고 싶을 때만 하면 됩니다.

1. [github.com](https://github.com)에 가입합니다.
2. **Settings → Developer settings → Personal access tokens**에서 **읽기·쓰기(read and write)** 권한 토큰을 발급받습니다.
3. `tour` 저장소를 만들고 `web` 폴더 내용을 올립니다.
4. 저장소 **Settings → Pages**에서 배포 브랜치를 지정하면 주소가 생깁니다.

토큰도 `.env`에 넣어 두면 편합니다.

```bash
# 깃허브 (선택 — 배포할 때만)
GITHUB_TOKEN=여기에_토큰_붙여넣기
```

> 업로드도 말로 시킬 수 있습니다. "`.env`의 GITHUB_TOKEN을 써서 내 계정에 tour 저장소를 만들고 web 폴더를 올린 뒤 Pages 설정까지 해 줘."

---

## 5. 2단계 — 관광 안내 챗봇

### 5-1. Gemini API 키 받기

1. [aistudio.google.com/api-key](https://aistudio.google.com/api-key)에 구글 계정으로 접속합니다.
2. **Create API key**를 누르면 키가 바로 나옵니다. 신용카드가 필요 없습니다.
3. `tour/.env`의 `GOOGLE_API_KEY` 줄에 붙여넣습니다.

> **키는 만들어진 순간 한 번만 온전히 보입니다.** 창을 닫기 전에 `.env`에 저장하세요.

### 5-2. `gemini_api.txt` 만들기

[aistudio.google.com/docs](https://aistudio.google.com/docs)에서 **Gemini API 파이썬 사용법**을 찾아 복사하고, `tour/chatbot/gemini_api.txt`로 저장합니다. 4-2절의 활용매뉴얼과 같은 역할입니다 — **AI에게 정답 문서를 쥐여 주는 것**입니다.

파일 내용은 이 정도면 충분합니다.

```text
# Gemini API — 파이썬 사용법 (이 실습 기준)

설치:
  pip install google-genai python-dotenv

모델: gemini-3.1-flash-lite
무료 등급 한도:
  RPM(분당 요청) : 약 15회/분
  RPD(하루 요청) : 약 500~1,000회/일
  ※ 한도를 넘기면 429 오류가 납니다. 잠시 기다렸다 다시 시도하세요.

기본 코드:
  import os
  from dotenv import load_dotenv
  from google import genai

  load_dotenv()
  client = genai.Client(api_key=os.getenv("GOOGLE_API_KEY"))

  response = client.models.generate_content(
      model="gemini-3.1-flash-lite",
      contents="경복궁을 한 문장으로 소개해 줘",
  )
  print(response.text)
```

### 5-3. 프롬프트

가상환경을 켜고 `tour` 폴더에서 `claude`를 실행한 뒤 입력합니다.

```text
tour/web에 생성된 html 파일과 tour/chatbot 폴더의 gemini_api.txt 참조해서, 이용자가 관광지 물어보면 관광공사 정보를 API로 가져와 이를 참조해서 답하는 챗봇 파일을 간결하게 파이선(py) 파일로 생성해
```

AI는 1단계 HTML에서 **이미 검증된 API 호출 방식**을 그대로 가져다 쓰고, `gemini_api.txt`에서 모델 이름과 호출 형식을 읽습니다. 그래서 처음부터 만들 때보다 훨씬 정확합니다.

```bash
python chatbot/chatbot.py
```

> ✅ **체크포인트**: "부산에 갈 만한 바닷가 알려 줘"라고 물었을 때, 실제 관광지 이름과 주소가 섞인 답이 나오면 성공입니다. 지어낸 이름만 나온다면 API 호출이 실패한 것이니 4-5절을 확인하세요.

이어서 다듬습니다.

```text
답변 아래에 참고한 관광지의 이름과 주소를 목록으로 붙여 줘. API에서 못 찾으면 '정보 없음'이라고 솔직히 답하게 해 줘.
```

### 5-4. (선택) Hugging Face Spaces에 배포하기

챗봇을 웹에서 쓰게 하고 싶을 때만 하면 됩니다.

1. [huggingface.co](https://huggingface.co)에 가입합니다.
2. **Settings → Access Tokens**에서 **읽기·쓰기(read and write)** 토큰을 발급받습니다.
3. **New Space**에서 `tour` 스페이스를 만듭니다. SDK는 **Gradio**를 고릅니다.
4. 스페이스의 **Settings → Variables and secrets**에 `GOOGLE_API_KEY`와 `DATAGOKR_API_KEY`를 **Secret**으로 등록합니다.

> **`.env` 파일은 절대 업로드하지 마세요.** 스페이스에서는 Secret 기능이 `.env` 역할을 대신합니다. 코드의 `os.getenv("GOOGLE_API_KEY")`는 그대로 동작합니다.

AI에게 화면 만들기까지 맡길 수 있습니다.

```text
chatbot.py를 Hugging Face Spaces에 올릴 수 있게 Gradio 화면을 붙이고, requirements.txt도 만들어 줘.
```

---

## 6. 3단계 — 글쓰기 에이전트

여기부터가 심화 가이드에서 배운 **서브에이전트**와 **스킬**을 실제로 쓰는 단계입니다.

### 6-1. `writing_tips.txt` 만들기

먼저 **재미있게 쓰는 법**에 대한 자료를 모읍니다. 구글에서 "재미있는 글 쓰는 법", "글 잘 쓰는 방법" 등으로 검색해 쓸 만한 내용을 골라 `tour/writing_agent/writing_tips.txt`로 저장합니다. ([참고 예시](https://blog.naver.com/sunyoool/223358899915))

이것도 AI에게 맡기면 편합니다.

```text
재미있는 글 쓰는 법을 웹에서 검색해서 핵심 요령만 골라 writing_agent/writing_tips.txt로 정리해 줘. 출처 링크도 함께 남겨 줘.
```

> **왜 이 파일이 필요한가요?** 그냥 "재미있게 써 줘"라고 하면 AI마다, 그날그날 결과가 들쭉날쭉합니다. **기준을 파일로 고정**해 두면 언제 실행해도 같은 톤이 나옵니다. 심화 가이드의 메모리 파일과 같은 원리입니다.

### 6-2. 프롬프트

```text
tour/writing_agent에 관광 관련 키워드를 입력하면 관련 정보 검색해 수집하고 요약정리한 뒤 재미있게 작성하는 sub-agent와 필요한 skill 생성해. sub-agent는 검색수집, 요약정리, 글 작성의 3개로 구성. skill은 tour/web의 html 파일(API key 포함)을 참조해 검색수집, tour/writing_agent 폴더의 writing_tips.txt를 참조해 글 작성 등 필요한대로 구성해. 결과를 tour/writing_agent에 result.md로 생성해
```

### 6-3. 만들어질 구조

AI가 대략 이런 구성을 만듭니다. 파일 이름은 조금씩 다를 수 있습니다.

```text
tour/
├── writing_agent/
│   ├── writing_tips.txt      # 글쓰기 기준 (내가 만든 것)
│   └── result.md             # 최종 결과물
└── .claude/
    ├── agents/
    │   ├── collector.md      # ① 검색·수집  — 관광공사 API + 웹 검색
    │   ├── summarizer.md     # ② 요약·정리  — 사실만 추려 정돈
    │   └── writer.md         # ③ 글 작성    — writing_tips 기준으로 재미있게
    └── skills/
        ├── tour-search/SKILL.md    # API 호출 절차 (web/index.html 참조)
        └── fun-writing/SKILL.md    # 글쓰기 절차 (writing_tips.txt 참조)
```

세 일꾼이 순서대로 일합니다. **각자 자기 책상(독립 문맥)을 쓰기 때문에**, 수집한 원자료가 글쓰기 단계까지 그대로 흘러들어 대화가 지저분해지는 일이 없습니다.

```text
키워드 "제주 오름"
   │
   ▼ ① 검색·수집   → 관광공사 API + 웹에서 자료 모으기
   ▼ ② 요약·정리   → 사실만 추려 정돈, 출처 표시
   ▼ ③ 글 작성     → writing_tips.txt 기준으로 재미있게
   │
   ▼ writing_agent/result.md
```

### 6-4. 실행

```text
"제주 오름"으로 글 써 줘.
```

> ✅ **체크포인트**: `writing_agent/result.md`가 생기고, 그 안에 **실제 존재하는 오름 이름과 주소**가 들어 있으면 성공입니다.

단계를 끊어서 시키면 어느 일꾼이 무슨 일을 하는지 눈으로 볼 수 있어 실습용으로 좋습니다.

```text
검색수집 서브에이전트만 먼저 돌려서 "제주 오름" 자료를 모아 줘.
```

> **큰 작업 전에는 `Shift+Tab`으로 플랜 모드**에 들어가세요. 어떤 서브에이전트와 스킬이 언제 작동하는지 계획으로 먼저 보여 줍니다.

> **결과물은 반드시 사람이 확인하세요.** AI는 그럴듯하게 지어내는 일이 있습니다. 관광지 이름·주소·영업시간은 [대한민국 구석구석](https://korean.visitkorea.or.kr) 같은 원출처에서 다시 확인한 뒤 쓰세요.

---

## 7. 자주 막히는 곳

| 증상 | 해결 |
|---|---|
| `SERVICE_KEY_IS_NOT_REGISTERED_ERROR` (403) | ① 이 API에 활용신청을 했는지 ② 승인 후 시간이 지났는지 ③ **인코딩/디코딩 키를 바꿔 썼는지**([4-5절](#key)) 순서로 확인 |
| `.env`를 못 읽음 | 파일 위치가 `tour/` 최상단인지, 이름이 `.env.txt`가 아닌지 확인. 파이썬은 `load_dotenv()`를 호출했는지 확인 |
| `ModuleNotFoundError: No module named 'dotenv'` | 가상환경이 꺼져 있습니다. `(.venv)` 표시를 확인하고 `pip install python-dotenv` 재실행 |
| Gemini `429 RESOURCE_EXHAUSTED` | 무료 한도(분당 15회) 초과입니다. 1분 기다렸다 다시 시도하세요 |
| 챗봇이 관광지를 지어냄 | API 호출이 실패해 AI가 자기 지식으로 답한 것입니다. "API 응답을 그대로 출력해 봐"라고 시켜 확인하세요 |
| 서브에이전트가 안 불림 | `.claude/agents/`의 `description` 줄이 모호하면 AI가 언제 쓸지 판단하지 못합니다. "언제 쓰는지"를 구체적으로 고쳐 주세요 |
| 한글이 깨짐 | 파일을 **UTF-8**로 저장. 파이썬은 `open(..., encoding="utf-8")` |

> **모든 오류의 1번 해결책** — 오류 메시지를 **그대로 복사해서** AI에게 붙여 넣고 "원인과 해결 방법을 알려줘"라고 물어보세요.

---

## 8. 참고 링크

1. 공공데이터포털 — 한국관광공사_국문 관광정보 서비스_GW. https://www.data.go.kr/data/15101578/openapi.do
2. 한국관광공사 TourAPI (콘텐츠랩). https://api.visitkorea.or.kr
3. Google AI Studio — API 키 발급. https://aistudio.google.com/api-key
4. Google AI Studio — 개발 문서. https://aistudio.google.com/docs
5. Gemini API 파이썬 SDK. https://googleapis.github.io/python-genai/
6. python-dotenv 문서. https://pypi.org/project/python-dotenv/
7. GitHub Pages 시작하기. https://docs.github.com/pages
8. Hugging Face Spaces 문서. https://huggingface.co/docs/hub/spaces
9. 재미있는 글쓰기 참고 예시. https://blog.naver.com/sunyoool/223358899915

> API 주소·모델명·무료 한도는 2026년 8월 기준이며 자주 바뀝니다. 막히면 각 도구에서 `/help`를 먼저 확인하고 위 문서를 참고하세요.
