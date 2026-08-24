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

`tour` 폴더에서 `claude`를 실행하고 아래를 그대로 붙여넣습니다.

> **프롬프트는 다섯 덩어리로 나눠 씁니다** — `[목표]` 무엇을 만들지 · `[참고 자료]` 무엇을 근거로 삼을지 · `[요구사항]` 구체적으로 어떻게 · `[제약]` 하지 말아야 할 것 · `[확인]` 다 됐는지 어떻게 아는지. 한 줄로 던지면 AI가 빈칸을 상상으로 채웁니다. 이 문서의 프롬프트는 모두 이 양식입니다.

```text
[목표]
web 폴더에 한국관광공사 관광정보를 검색해 보여 주는 단일 HTML 파일(web/index.html)을 만들어 줘.

[참고 자료]
- web 폴더에 있는 「한국관광공사_개방데이터_활용매뉴얼」 문서를 먼저 읽어. 요청 주소(엔드포인트), 오퍼레이션 이름,
  파라미터 이름은 반드시 이 매뉴얼에 적힌 것을 그대로 써. 매뉴얼에 없는 이름은 지어내지 마.
- 인증키는 공공데이터포털에서 받은 Encoding 인증키를 쓴다. 파일 맨 위에 const SERVICE_KEY = "..." 한 줄로 두어
  내가 나중에 찾아 바꾸기 쉽게 해 줘.

[요구사항]
1. 키워드 검색 — 검색창에 단어를 넣으면 매뉴얼의 '키워드 검색 조회' 오퍼레이션으로 관광지를 찾는다.
2. 지역 선택 — 시·도를 고르는 드롭다운을 두고, 매뉴얼의 '지역코드 조회' 오퍼레이션으로 받아 온 값으로 채운다.
   지역만 골랐을 때는 '지역기반 관광정보 조회'로 검색한다.
3. 공통 파라미터(serviceKey, MobileOS, MobileApp, _type=json, numOfRows, pageNo 등) 중 매뉴얼이 필수라고 적은
   항목을 하나도 빠뜨리지 마.
4. 결과는 카드 목록으로 — 관광지명 · 주소 · 대표 이미지 · 전화번호. 이미지가 없으면 회색 상자를 대신 보여 준다.
5. 응답 처리 — response.header.resultCode 가 0000이 아니거나 totalCount가 0이면 그 사실을 화면에 그대로 보여 준다.
   조용히 빈 화면을 두지 마.

[제약]
- 파일은 web/index.html 하나. 외부 라이브러리나 빌드 도구 없이 순수 HTML + CSS + JavaScript로.
- <meta charset="utf-8">를 넣어 한글이 깨지지 않게 한다.
- 초보자가 읽을 수 있게 짧게 쓰고, API를 호출하는 부분에는 한국어 주석을 단다.

[확인]
다 만든 뒤 실제 요청 주소를 한 번 호출해서 정상 응답(resultCode 0000)이 오는지 네가 먼저 확인하고,
호출한 주소와 응답 건수를 나에게 보여 줘. 실패했다면 원인과 고친 내용을 알려 줘.
```

AI가 매뉴얼을 읽고 계획을 보여 줍니다. 동의하면 `tour/web/index.html`이 만들어집니다. 브라우저로 열어 확인하세요.

```bash
# macOS
open web/index.html
# Windows
start web/index.html
```

> ✅ **체크포인트**: 검색창에 `경복궁`을 넣었을 때 주소와 사진이 있는 목록이 나오면 성공입니다.

더 붙이고 싶은 기능도 같은 양식으로 말합니다.

```text
[추가 요청]
결과 카드마다 '상세보기' 버튼을 달아 줘.

[요구사항]
1. 누르면 매뉴얼의 '공통정보 조회' 오퍼레이션을 contentId로 호출해 개요(overview)를 받아 카드 아래에 펼친다.
2. 결과가 많을 때를 위해 목록 아래에 '더 보기' 버튼을 두고, 누르면 pageNo를 1 올려 이어 붙인다.
3. 이미 펼친 상세 정보는 다시 호출하지 말고 재사용한다.

[제약]
지금 동작하는 검색 기능은 건드리지 마. 파일은 계속 web/index.html 하나로 유지해.
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
3. 토큰을 `tour/.env`에 넣습니다.

```bash
# 깃허브 (선택 — 배포할 때만)
GITHUB_TOKEN=여기에_토큰_붙여넣기
```

나머지는 AI에게 맡깁니다.

```text
[목표]
web/index.html 을 GitHub Pages로 배포해서 링크 하나로 공유할 수 있게 해 줘.

[참고 자료]
- 인증은 tour/.env 의 GITHUB_TOKEN 을 읽어서 쓴다. 토큰 값을 화면에 출력하거나 코드·커밋 메시지에 남기지 마.
- 올릴 파일은 web/index.html 하나뿐이다. tour/.env 와 .gitignore 에 적힌 것은 절대 올리지 마.

[할 일]
1. 내 깃허브 계정에 tour-web 이라는 public 저장소를 만든다. 이미 있으면 그것을 쓴다.
2. web/index.html 을 main 브랜치 최상단에 올린다.
3. 저장소 Settings → Pages 를 'main 브랜치 / 루트(/)'로 설정한다.
4. 완성된 배포 주소를 알려 준다.

[제약]
- 커밋은 한 번으로 정리해 줘.
- 저장소 이름이 이미 쓰이고 있으면 멋대로 다른 이름을 만들지 말고 나에게 먼저 물어봐.

[확인]
올리기 전에, web/index.html 안에 적힌 공공데이터포털 인증키가 공개돼도 되는 무료 키가 맞는지 나에게 한 번 확인해 줘.
올린 뒤에는 배포 주소를 실제로 열어 페이지가 뜨는지 확인하고 결과를 알려 줘.
```

> Pages는 반영에 1~2분쯤 걸립니다. 바로 404가 떠도 잠시 뒤 다시 열어 보세요.

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

가상환경을 켜고 `tour` 폴더에서 `claude`를 실행한 뒤 붙여넣습니다.

```text
[목표]
chatbot/chatbot.py 파일 하나로 도는 터미널 챗봇을 만들어 줘.
이용자가 관광지를 물으면 한국관광공사 API로 실제 정보를 가져오고, 그 정보를 근거로 Gemini가 한국어로 답한다.

[참고 자료]
- web/index.html — 1단계에서 이미 동작을 확인한 관광공사 API 호출 방식이 들어 있다.
  요청 주소, 오퍼레이션 이름, 파라미터 구성을 그대로 옮겨 써. 새로 지어내지 마.
- chatbot/gemini_api.txt — Gemini 모델 이름과 파이썬 호출 형식.
- 키는 tour/.env 에서 python-dotenv 로 읽는다. 관광공사 키는 DATAGOKR_API_KEY, Gemini 키는 GOOGLE_API_KEY.
  코드 안에 키 값을 직접 적지 마.

[요구사항]
1. 이용자 질문에서 지역과 키워드를 뽑아 관광공사 API를 호출한다.
   키워드가 분명하면 '키워드 검색 조회', 지역만 있으면 '지역기반 관광정보 조회'를 쓴다.
2. 공통 파라미터(serviceKey, MobileOS=ETC, MobileApp, _type=json, numOfRows, pageNo)를 빠뜨리지 않는다.
3. 받아 온 결과 상위 5건(관광지명·주소·전화·개요)을 정리해 Gemini 프롬프트에 '참고 자료'로 넣고,
   "이 자료 안에 있는 사실만 쓰고, 없는 것은 모른다고 답하라"고 지시한다.
4. 답변 아래에 참고한 관광지의 이름과 주소를 목록으로 붙인다.
5. API 결과가 0건이면 지어내지 말고 "관광공사 자료에서 찾지 못했습니다"라고 답한다.
6. '종료'를 입력할 때까지 계속 대화한다.

[제약]
- requests 로 호출하고 인증키는 params= 로 넘긴다. 이때는 반드시 Decoding 인증키를 쓴다(이중 인코딩 주의).
- 파일은 chatbot/chatbot.py 하나. 필요한 패키지는 chatbot/requirements.txt 에 적어 준다.
- 파일 입출력은 모두 encoding="utf-8".
- API 호출이 실패하면 상태 코드와 응답 앞부분을 그대로 출력한다. 예외를 조용히 삼키지 마.

[확인]
다 만든 뒤 "부산 해수욕장 추천해 줘"로 한 번 실행해서,
① 관광공사 API가 돌려준 건수와 ② 최종 답변을 함께 보여 줘.
```

AI는 1단계 HTML에서 **이미 검증된 API 호출 방식**을 그대로 가져다 쓰고, `gemini_api.txt`에서 모델 이름과 호출 형식을 읽습니다. 그래서 처음부터 만들 때보다 훨씬 정확합니다.

```bash
python chatbot/chatbot.py
```

> ✅ **체크포인트**: "부산에 갈 만한 바닷가 알려 줘"라고 물었을 때, 실제 관광지 이름과 주소가 섞인 답이 나오면 성공입니다. 지어낸 이름만 나온다면 API 호출이 실패한 것이니 4-5절을 확인하세요.

이어서 다듬습니다.

```text
[추가 요청]
대화가 이어지게 만들어 줘.

[요구사항]
1. 직전 3턴을 기억해서 "거기 근처에 먹을 데는?" 같은 이어지는 질문도 알아듣게 한다.
   이때도 답은 새로 호출한 관광공사 API 결과에 근거해야 한다.
2. 대화를 마치면 오간 내용과 그때 호출한 API 결과 건수를 chatbot/log.md 로 저장한다.

[제약]
기억은 3턴까지만. 그보다 오래된 대화는 버려서 프롬프트가 길어지지 않게 해 줘.
```

### 5-4. (선택) Hugging Face Spaces에 배포하기

챗봇을 웹에서 쓰게 하고 싶을 때만 하면 됩니다.

1. [huggingface.co](https://huggingface.co)에 가입합니다.
2. **Settings → Access Tokens**에서 **읽기·쓰기(read and write)** 토큰을 발급받습니다.
3. **New Space**에서 `tour` 스페이스를 만듭니다. SDK는 **Gradio**를 고릅니다.
4. 스페이스의 **Settings → Variables and secrets**에 `GOOGLE_API_KEY`와 `DATAGOKR_API_KEY`를 **Secret**으로 등록합니다.

> **`.env` 파일은 절대 업로드하지 마세요.** 스페이스에서는 Secret 기능이 `.env` 역할을 대신합니다. 코드의 `os.getenv("GOOGLE_API_KEY")`는 그대로 동작합니다.

올릴 파일 준비는 AI에게 맡깁니다.

```text
[목표]
chatbot.py 를 Hugging Face Spaces(Gradio)에서 그대로 돌아가게 바꿔 줘.

[참고 자료]
- chatbot/chatbot.py — 여기 있는 관광공사 API 호출과 Gemini 호출 로직을 그대로 재사용한다.
  화면만 Gradio로 감싸는 것이지, 동작을 다시 짜는 것이 아니다.
- 키는 Spaces의 Secret에서 온다. 코드는 지금처럼 os.getenv("DATAGOKR_API_KEY"),
  os.getenv("GOOGLE_API_KEY") 로 읽고, load_dotenv() 는 로컬에서만 쓰이도록 둔다.

[할 일]
1. chatbot/app.py 를 만들고 gr.ChatInterface 로 대화 화면을 붙인다.
2. 답변 아래에 참고한 관광지 이름·주소 목록이 그대로 나오게 한다.
3. chatbot/requirements.txt 에 gradio, requests, google-genai, python-dotenv 를 적는다.
4. chatbot/README.md 를 만들고 맨 위에 Spaces용 머리말(title, sdk: gradio, app_file: app.py)을 넣는다.
5. 스페이스에 올려야 할 파일 목록을 정리해서 알려 준다.

[제약]
- .env 파일과 인증키 값은 업로드 목록에 절대 넣지 마. 목록을 알려 줄 때 이 점을 한 번 더 확인해 줘.
- 키가 없을 때는 화면에 "Secret에 DATAGOKR_API_KEY / GOOGLE_API_KEY 를 등록하세요"라고 안내한다.

[확인]
로컬에서 python chatbot/app.py 로 한 번 띄운 뒤,
"제주 오름 알려 줘"를 넣었을 때 관광공사 API 결과가 화면에 나오는지 확인하고 결과를 알려 줘.
```

---

## 6. 3단계 — 글쓰기 에이전트

여기부터가 심화 가이드에서 배운 **서브에이전트**와 **스킬**을 실제로 쓰는 단계입니다.

### 6-1. `writing_tips.txt` 만들기

먼저 **재미있게 쓰는 법**에 대한 기준을 파일로 만들어 둡니다. 참고 예시는 [How to make your blog more entertaining (Indie Essentials)](https://indieessentials.co.uk/make-your-blog-more-entertaining/) — 비유로 풀기, 이야기로 말하기, 자기 목소리 넣기, 관점 뒤집기 같은 요령을 짧게 정리해 둔 글입니다.

이것도 AI에게 맡기면 편합니다.

```text
[목표]
writing_agent/writing_tips.txt 를 만들어 줘. 재미있는 글을 쓰는 기준을 파일 하나로 고정하는 것이 목적이다.

[참고 자료]
- https://indieessentials.co.uk/make-your-blog-more-entertaining/ 를 먼저 읽고, 여기서 말하는 요령을 뼈대로 삼아.
- 여기에 더해 한국어 블로그·에세이 글쓰기 요령을 두세 곳 더 검색해 보완해.

[요구사항]
1. 요령은 실행할 수 있는 지시문으로 쓴다.
   좋은 예 — "첫 문장은 질문이나 장면으로 시작한다"
   나쁜 예 — "재미있게 쓴다" (무엇을 하라는 건지 알 수 없다)
2. 8~12개로 추리고, 항목마다 한 줄 설명과 짧은 예시를 붙인다.
3. '지키지 말아야 할 것'을 별도 목록으로 넣는다. 문단·문장 길이 기준, 과장 광고 문구,
   확인되지 않은 사실을 단정하는 표현 등.
4. 맨 아래에 참고한 링크를 출처로 남긴다.

[제약]
- 일반 텍스트(UTF-8)로 저장한다. 마크다운 문법은 쓰지 않는다.
- 이 파일 하나만 읽어도 기준을 알 수 있게, 다른 문서를 찾아보라는 문장은 넣지 마.

[산출물]
tour/writing_agent/writing_tips.txt
```

> **왜 이 파일이 필요한가요?** 그냥 "재미있게 써 줘"라고 하면 AI마다, 그날그날 결과가 들쭉날쭉합니다. **기준을 파일로 고정**해 두면 언제 실행해도 같은 톤이 나옵니다. 심화 가이드의 메모리 파일과 같은 원리입니다.

### 6-2. 프롬프트

```text
[목표]
tour/writing_agent 에서 도는 글쓰기 파이프라인을 만들어 줘.
관광 키워드 하나를 주면 → 자료를 모으고 → 사실만 추려 정리하고 → 재미있는 글로 써서 result.md 로 저장한다.

[만들 것]
1. 서브에이전트 3개 — .claude/agents/ 에 만든다.
   - tour-collector   검색·수집
   - tour-summarizer  요약·정리
   - tour-writer      글 작성
2. 스킬 2개 — .claude/skills/ 에 만든다.
   - tour-search   한국관광공사 API 호출 절차
   - fun-writing   writing_tips.txt 기준의 글쓰기 절차

[참고 자료 — 정확히 이대로 연결해 줘]
- tour-search 스킬에는 web/index.html 안에서 이미 동작이 확인된 관광공사 API 호출 방식
  (요청 주소, 오퍼레이션 이름, 파라미터 목록, 인증키를 넣는 자리)을 그대로 옮겨 절차로 적는다.
  주소나 파라미터를 새로 지어내지 마.
- 명령줄이나 파이썬에서 호출할 때는 tour/.env 의 DATAGOKR_API_KEY(Decoding 키)를 쓰고,
  공통 파라미터(MobileOS=ETC, MobileApp, _type=json, numOfRows, pageNo)를 빠뜨리지 않는다.
- fun-writing 스킬은 writing_agent/writing_tips.txt 를 매번 다시 읽어 그 기준을 적용한다.
  스킬 안에 요령을 베껴 적지 마. 기준은 그 파일 하나가 갖는다.

[각 에이전트가 할 일]
- tour-collector  : tour-search 스킬로 관광공사 자료를 모은다(관광지명·주소·전화·개요·이미지 주소·contentid).
                    API로 채우지 못한 것만 웹 검색으로 보완하고 출처를 남긴다. → writing_agent/collected.md
- tour-summarizer : collected.md 에서 확인된 사실만 추려 정리하고 근거를 표시한다.
                    어느 출처에도 없는 내용은 '출처 미확인'으로 남긴다. → writing_agent/summary.md
- tour-writer     : summary.md 와 writing_tips.txt 만 근거로 글을 쓴다. 자료를 더 찾지 않는다.
                    → writing_agent/result.md

[규칙]
- 세 에이전트는 순서대로 실행되고, 앞 단계가 남긴 파일만 다음 단계에 넘긴다.
- 관광지 이름과 주소는 반드시 관광공사 API 응답에서 온 값을 쓴다. 다듬거나 바꾸지 마.
- 각 에이전트의 description 에는 '언제 쓰는 에이전트인지'를 한 문장으로 분명히 적는다.
- 모든 파일은 UTF-8.

[산출물]
tour/writing_agent/result.md — 제목 한 줄, 본문 1,200자 내외, 맨 아래 '참고한 관광지' 표(이름·주소·출처)

[확인]
다 만든 뒤 생성한 파일 목록과 각 에이전트·스킬의 description 줄을 보여 줘. 아직 실행은 하지 마.
```

### 6-3. 만들어질 구조

AI가 대략 이런 구성을 만듭니다. 파일 이름은 조금씩 다를 수 있습니다.

```text
tour/
├── writing_agent/
│   ├── writing_tips.txt      # 글쓰기 기준 (내가 만든 것)
│   ├── collected.md          # ① 수집한 원자료
│   ├── summary.md            # ② 사실만 추린 정리본
│   └── result.md             # ③ 최종 결과물
└── .claude/
    ├── agents/
    │   ├── tour-collector.md    # ① 검색·수집  — 관광공사 API + 웹 검색
    │   ├── tour-summarizer.md   # ② 요약·정리  — 사실만 추려 정돈
    │   └── tour-writer.md       # ③ 글 작성    — writing_tips 기준으로 재미있게
    └── skills/
        ├── tour-search/SKILL.md   # API 호출 절차 (web/index.html 참조)
        └── fun-writing/SKILL.md   # 글쓰기 절차 (writing_tips.txt 참조)
```

세 일꾼이 순서대로 일합니다. **각자 자기 책상(독립 문맥)을 쓰기 때문에**, 수집한 원자료가 글쓰기 단계까지 그대로 흘러들어 대화가 지저분해지는 일이 없습니다.

```text
키워드 "제주 오름"
   │
   ▼ ① 검색·수집   → 관광공사 API + 웹에서 자료 모으기      → collected.md
   ▼ ② 요약·정리   → 사실만 추려 정돈, 출처 표시            → summary.md
   ▼ ③ 글 작성     → writing_tips.txt 기준으로 재미있게     → result.md
```

### 6-4. 만들어진 파일은 이렇게 생겼습니다

AI가 만들어 준 파일을 열어 보면 아래와 비슷합니다. **똑같이 나오지는 않습니다** — 내용을 읽어 보고 어색한 곳은 말로 고치라는 뜻으로 실어 둡니다.

파일 맨 위 `---` 사이가 **머리말(front matter)**입니다. 그중 `description` 줄이 특히 중요합니다. AI는 이 한 줄만 보고 "지금 이 일꾼을 부를 때인가"를 판단하기 때문에, 여기가 모호하면 에이전트가 아예 불리지 않습니다.

**① `.claude/agents/tour-collector.md`**

```markdown
---
name: tour-collector
description: 관광 키워드로 자료를 모을 때 사용한다. 한국관광공사 API에서 관광지 정보를 받아 오고, API로 채우지 못한 부분만 웹 검색으로 보완한다. 글쓰기 파이프라인 1단계.
tools: Read, Write, Bash, WebSearch, WebFetch
---

너는 자료 수집 담당이다. 판단하거나 글을 쓰지 않는다. 사실을 모아 파일로 남기는 것까지가 네 일이다.

## 절차
1. `tour-search` 스킬을 읽고, 거기 적힌 절차대로 한국관광공사 API를 호출한다.
2. 키워드로 '키워드 검색 조회'를 먼저 호출한다. 결과가 5건 미만이면 지역코드를 찾아
   '지역기반 관광정보 조회'로 보완한다.
3. 관광지마다 contentid 로 '공통정보 조회'를 호출해 개요(overview)를 받아 온다.
4. API로 채우지 못한 항목(계절 정보, 최근 소식 등)만 웹 검색으로 보완하고, 출처 URL을 함께 적는다.

## 산출물 — writing_agent/collected.md
관광지 하나당 아래 항목을 적는다. 빈 항목은 비워 두고 추측으로 채우지 않는다.
- 관광지명 / 주소 / 전화 / 개요 / 대표 이미지 주소 / contentid
- 출처: `관광공사 API(오퍼레이션 이름)` 또는 웹 링크

## 하지 말 것
- API 응답에 없는 이름·주소·전화번호를 만들어 내지 않는다.
- 요약하거나 문장을 다듬지 않는다. 받은 값을 그대로 옮긴다.
- 호출이 실패하면 그냥 넘어가지 말고, 요청 주소와 오류 코드를 그대로 파일에 적어 남긴다.
```

**② `.claude/agents/tour-summarizer.md`**

```markdown
---
name: tour-summarizer
description: collected.md 의 원자료를 사실만 남겨 정리할 때 사용한다. 출처가 없는 내용을 걸러 내는 것이 목적이다. 글쓰기 파이프라인 2단계.
tools: Read, Write
---

너는 정리 담당이다. 새 정보를 찾지 않는다. `writing_agent/collected.md` 에 있는 것만 다룬다.

## 절차
1. `collected.md` 를 읽는다. 파일이 없으면 멈추고 tour-collector 를 먼저 돌리라고 알린다.
2. 같은 contentid 를 가진 항목은 한 곳으로 합친다.
3. 관광지마다 한 문단으로 줄인다. 이름과 주소는 원문 그대로 옮긴다.
4. 근거가 관광공사 API 응답이면 문장 끝에 [관광공사], 웹이면 [웹: 링크] 를 붙인다.
5. 어느 쪽에도 없는 내용은 버리거나 [출처 미확인] 으로 표시한다.

## 산출물 — writing_agent/summary.md
- 맨 위: 키워드와 수집 건수
- 관광지별로: 이름 / 주소 / 한 줄 요약 / 특징 2~3문장(근거 표시 포함)
- 맨 아래: '확인하지 못한 것' 목록

## 하지 말 것
- 형용사를 덧붙여 인상을 바꾸지 않는다. 재미있게 만드는 일은 다음 단계의 몫이다.
- 건수를 맞추려고 없는 관광지를 채우지 않는다. 적으면 적은 대로 넘긴다.
```

**③ `.claude/agents/tour-writer.md`**

```markdown
---
name: tour-writer
description: summary.md 를 재미있는 글로 바꿀 때 사용한다. writing_tips.txt 의 기준을 그대로 적용한다. 글쓰기 파이프라인 3단계.
tools: Read, Write
---

너는 글쓴이다. 자료를 더 찾지 않는다. `writing_agent/summary.md` 안에 있는 사실만 쓴다.

## 절차
1. `fun-writing` 스킬을 읽고, 거기 지시대로 `writing_agent/writing_tips.txt` 를 읽는다.
2. summary.md 를 읽고 글의 흐름을 먼저 세 줄로 잡는다 — 무엇으로 시작해 무엇으로 끝낼지.
3. writing_tips.txt 의 요령을 항목별로 적용해 초고를 쓴다.
4. 다 쓴 뒤 writing_tips.txt 의 '지키지 말아야 할 것' 목록을 하나씩 대조해 고친다.

## 산출물 — writing_agent/result.md
- 제목 한 줄
- 본문 1,200자 내외
- 맨 아래 '참고한 관광지' 표: 관광지명 · 주소 · 출처

## 하지 말 것
- summary.md 에 없는 관광지·주소·영업시간·요금을 쓰지 않는다.
- [출처 미확인] 이 붙은 내용을 단정해서 쓰지 않는다. 쓰려면 "~라고 한다"로 표시한다.
```

**④ `.claude/skills/tour-search/SKILL.md`**

```markdown
---
name: tour-search
description: 한국관광공사 관광정보 API로 관광지를 검색할 때 사용한다. 요청 주소·오퍼레이션·필수 파라미터·인증키 규칙을 담고 있다.
---

# 한국관광공사 관광정보 API 호출 절차

## 0. 근거 파일
요청 주소와 오퍼레이션 이름은 `web/index.html` 안에 이미 동작이 확인된 값이 들어 있다.
새로 짓지 말고 항상 그 파일에서 읽어 온다.

## 1. 인증키
- 파이썬 requests 의 params= 로 넘길 때 → `.env` 의 DATAGOKR_API_KEY (**Decoding 키**)
- 주소 문자열에 직접 이어 붙일 때 → **Encoding 키**
- 둘을 바꿔 쓰면 키가 멀쩡해도 403 SERVICE_KEY_IS_NOT_REGISTERED_ERROR 가 난다.

## 2. 공통 파라미터 (하나라도 빠지면 오류)
- serviceKey : 인증키
- MobileOS   : ETC
- MobileApp  : 아무 이름 (예: tour)
- _type      : json
- numOfRows  : 10 / pageNo : 1

## 3. 쓰는 오퍼레이션
- 키워드로 찾기   — 키워드 검색 조회 (keyword)
- 지역으로 찾기   — 지역기반 관광정보 조회 (areaCode, contentTypeId)
- 지역 코드 목록  — 지역코드 조회
- 개요·상세      — 공통정보 조회 (contentId)
정확한 오퍼레이션 이름은 `web/index.html` 과 활용매뉴얼에 적힌 것을 그대로 쓴다.

## 4. 응답 확인
- response.header.resultCode 가 0000 이어야 정상이다.
- totalCount 가 0이면 결과가 없는 것이다. 다른 키워드로 다시 시도하되, 없는 결과를 만들어 내지 않는다.
- XML이 돌아오면 _type=json 이 빠진 것이다.

## 5. 호출 예의
- 연속 호출 사이에 0.3초쯤 쉰다.
- 같은 요청을 반복하지 않는다. 개발계정은 하루 호출 한도가 있다.
```

**⑤ `.claude/skills/fun-writing/SKILL.md`**

```markdown
---
name: fun-writing
description: 정리된 자료를 재미있는 글로 쓸 때 사용한다. writing_tips.txt 의 기준을 적용하고, 다 쓴 뒤 점검하는 절차를 담고 있다.
---

# 재미있게 쓰기 절차

## 0. 기준 파일
글을 쓰기 전에 `writing_agent/writing_tips.txt` 를 **매번 다시 읽는다.**
이 스킬에는 요령을 베껴 적지 않는다. 기준은 그 파일 하나가 갖는다. 파일이 바뀌면 결과도 바뀌어야 한다.

## 1. 쓰기 전
- summary.md 에서 가장 의외인 사실 하나를 고른다. 그것이 첫 문단이 된다.
- 독자를 한 명으로 정한다. 예: 주말에 갈 곳을 찾는 직장인.

## 2. 쓰는 중
- writing_tips.txt 의 항목을 위에서부터 적용한다.
- 사실 문장과 감상 문장을 섞는다. 사실만 이어지면 목록이 되고, 감상만 이어지면 광고가 된다.
- 관광지 이름이 처음 나올 때 주소를 괄호로 붙인다.

## 3. 쓴 뒤 점검 — 하나씩 확인하고 결과를 보고한다
- writing_tips.txt 의 '지키지 말아야 할 것' 목록에 걸리는 문장이 있는가
- summary.md 에 없는 사실이 들어가지 않았는가
- 한 문단이 다섯 줄을 넘지 않는가
- 제목만 읽고도 무슨 글인지 알 수 있는가

## 4. 산출물
`writing_agent/result.md` 로 저장하고, 3번 점검에서 고친 내용을 한 줄로 함께 남긴다.
```

> **읽어 보고 고치세요.** 특히 `description` 줄이 "관광 관련 작업" 처럼 두루뭉술하면 에이전트가 불리지 않습니다. "언제 쓰는지"가 분명한 문장으로 바꿔 달라고 말하면 됩니다.

### 6-5. 실행

```text
"제주 오름"으로 글 써 줘.
```

> ✅ **체크포인트**: `writing_agent/result.md`가 생기고, 그 안에 **실제 존재하는 오름 이름과 주소**가 들어 있으면 성공입니다.

단계를 끊어서 시키면 어느 일꾼이 무슨 일을 하는지 눈으로 볼 수 있어 실습용으로 좋습니다.

```text
tour-collector 서브에이전트만 먼저 돌려서 "제주 오름" 자료를 모아 줘. collected.md 까지만 만들고 멈춰.
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
7. GitHub Pages 시작하기. https://docs.github.com/en/pages
8. Hugging Face Spaces 문서. https://huggingface.co/docs/hub/spaces
9. Indie Essentials — How to make your blog more entertaining. https://indieessentials.co.uk/make-your-blog-more-entertaining/

> API 주소·모델명·무료 한도는 2026년 8월 기준이며 자주 바뀝니다. 막히면 각 도구에서 `/help`를 먼저 확인하고 위 문서를 참고하세요.
