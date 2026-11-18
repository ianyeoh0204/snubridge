# SNU Bridge

English first. [한국어는 아래에 있습니다.](#korean)

Graduation requirements and department notices in English, for international students. The code builds a static website
from each department's official rule documents and its notice board.

Four departments are on the site: Electrical and Computer Engineering (ECE), Aerospace, Naval Architecture and Ocean
Engineering, and Nuclear Engineering.

## Set up
```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env   # then put your own key in .env
```

## Run
```bash
.venv/bin/python -m bridge notices ECE   # read the department's board and record new and changed notices
.venv/bin/python -m bridge briefs ECE    # translate the notices that are new or changed (calls the model)
.venv/bin/python -m bridge site          # build site/index.html, site/cards.json and the calendar site/ece.ics
```
Replace `ECE` with `AEROSPACE`, `NAVAL` or `NUCLEAR` for the other departments.

The calendar (`bridge/deadlines.py`) holds only the dates that a notice prints with a weekday. Code checks each date
against that weekday and against the English line. No model is asked for a date. The build writes the calendar for a
department whose `config.json` has `site`, the public address of the page. Students subscribe to it from the Notices page.

## How it stays up to date without anyone's laptop
`.github/workflows/update.yml` runs on GitHub once a day. Its schedule says 07:17 in Seoul, but GitHub starts scheduled
jobs late (the run of 12 Nov 2026 began at 12:08), so the site and the poster promise no time of day.

Each run reads every department's board, translates at most 40 new or changed notices per department, saves them to
`data/`, rebuilds the site and publishes it with GitHub Pages. A department whose site is down does not stop the others.
The page is still rebuilt, and the run is marked failed at the end so that GitHub emails the owner.

A push to `main` republishes the site without reading the boards. The "Run workflow" button does a full run on demand.

## Taking over
1. Get access to this repository (or fork it) and turn on Pages under Settings > Pages > Source: GitHub Actions.
2. Add your own model key under Settings > Secrets and variables > Actions > New repository secret. Name it
   `ANTHROPIC_API_KEY` (the workflow also accepts `API_KEY`). To use another provider, change `bridge/llm.py` only.
3. Each semester, download the new official files into `sources/<department>/`, list them in its `config.json` and
   update the cards. Nothing else needs a person.
4. GitHub pauses scheduled workflows in a repository with no activity for 60 days, and emails first. Press
   "Run workflow" once to resume.

## Requirement cards
A person downloads each department's rule documents. An AI assistant writes the cards from them, one script per
department, and code checks every card against the Korean before saving it.
```bash
(cd tools && ../.venv/bin/python build_ece.py)   # ECE: writes data/ECE/cards.json
.venv/bin/python tools/pages_aerospace.py        # Aerospace: read its graduate rule pages (two rule sets per page, by entry year)
.venv/bin/python tools/cards_aerospace.py        # Aerospace: check every card and write data/AEROSPACE/cards.json
.venv/bin/python tools/cards_naval.py            # Naval: the same, from its rule PDFs in sources/NAVAL/
.venv/bin/python tools/cards_nuclear.py          # Nuclear: the same, from its rule PDFs in sources/NUCLEAR/
```
Each of the last three also loads that department's undergraduate cards from `tools/cards_<department>_undergraduate.py`.

## Checks
```bash
.venv/bin/python tools/check_quotes.py     # ECE undergraduate cards: the Korean quote and course codes are in every entry year's file
.venv/bin/python tools/check_updates.py    # "later notices" on ECE cards: each Korean quote is in the notice it cites
.venv/bin/python tools/audit.py prepare    # ECE: a second reader re-reads cards against the page images (see the top of the file)
.venv/bin/python tools/evidence.py         # recount ECE's Korean and English boards for the Background and Method page
.venv/bin/python tools/survey.py count     # count Korean and English notices in ten engineering departments
.venv/bin/python tools/survey.py aggregate # write the totals to data/survey.json (the commands `check` and `sample` verify the count)
.venv/bin/python tools/korean_review.py    # every English and Korean sentence pair, as a sheet for a Korean reader
.venv/bin/python tools/reader_sheet.py     # ten notices beside their Korean originals, as a sheet for a Korean-speaking reader
.venv/bin/python tools/poster.py           # the competition poster as a PDF in out/poster/, drawn from the same data as the site (first: .venv/bin/pip install segno)
```
`sources/context.json` holds the facts and links for the first step of the Background page, which covers what the
university has said and done this year. Re-read each cited page before changing a line there.

When two sources differ, the most recent official one is the rule. Add a line to `sources/<department>/updates.json`
with the notice number and the exact Korean line. The card then shows it under "Added or changed by later notices".

## The department's own files
The official PDFs and spreadsheets are not in this repository, because they are the department's documents, not ours.
`sources/<department>/config.json` lists each file with the page it was downloaded from. Download them into that folder
before running a card script, `tools/check_quotes.py` or `tools/audit.py`. The daily update and the site build do not
need them.

## Add a department
ECE is the main department (`"main": true` in its config). The page opens on it, and the Background page is about its
boards. Aerospace, Naval and Nuclear were each added in about a day, in four steps.

1. **Notices.** Create `sources/<NAME>/config.json` with the board's addresses and a `since` date. Copy
   `sources/NUCLEAR/config.json` as a start. Run `python -m bridge notices <NAME>` and `python -m bridge briefs <NAME>`
   on your own machine first. Never push a new department folder without its translated notices, or the daily job
   translates them all on GitHub. The readers in `bridge/notices.py` and `bridge/briefs.py` fit the board software these
   four departments use. Other software needs its own two readers, and `tools/survey.py` shows how the other engineering
   sites list their posts. For a slow site, set `board.pause` in its config to the seconds to wait between requests
   (Naval uses 5).
2. **Requirements.** A person downloads the department's graduation rules into `sources/<NAME>/`. PDFs there are never
   uploaded. An AI assistant writes `tools/cards_<name>.py` from `prompts/write-cards.md`. The checks in
   `tools/cards_pdf.py` refuse a card whose Korean quote is not in the document or whose numbers are not in a quote.
   For rules published as web pages, follow `tools/pages_aerospace.py`. Undergraduate rules differ by year of admission,
   so each undergraduate card lists its `entry_years`. Where every year has its own file, the card's quotes must be in
   each of them. `tools/cards_nuclear_undergraduate.py` is the shortest example.
3. **Second reading.** A model from another maker reads every card against the Korean with `prompts/read-cards.md`.
   Correct the cards and repeat until the findings no longer change what a student must do. Record the result in
   `data/<NAME>/audit.json`.
4. **Publish.** Add `programs`, `headline_cards` and `files` to the config and run `python -m bridge site`. Look at the
   department in the switch under the header, then push.

A folder with a `board` and `"pilot": true` but no cards appears on the Notices page only, as a pilot.

## The part done in ChatGPT Edu
`navigator/instructions.md` holds the instructions for the assistant that a student uses for what this site does not
read: images, attachments and the student's own questions about a notice. Paste them into a ChatGPT Edu project. They are
plain text, so they can move to a plugin or skill when the workspace offers one.

## Sending the source
On the sending day, pull first. The daily job saves new notices on GitHub, and the poster prints the day it was built.
```bash
git pull
.venv/bin/python tools/poster.py
git archive --format=zip --prefix=snubridge/ -o out/snubridge-source.zip HEAD
```
Look at `out/poster/poster.png` before sending. The last command packs exactly what is in the repository. It contains no key, no downloaded department documents and nothing from `out/`.

The archive sent for the competition also carries `site/` (the built page, its data and the calendar file), added after `python -m bridge site`, so that the site can be opened without building: open `site/index.html`.

---

<a id="korean"></a>

# SNU Bridge (한국어)

외국인 학생을 위한 영문 졸업 요건과 학과 공지입니다. 이 코드는 각 학과의 공식 규정 문서와 공지 게시판으로 정적 웹사이트를 만듭니다.

사이트에는 전기·정보공학부(ECE), 항공우주공학과, 조선해양공학과, 원자핵공학과의 네 개 학과가 올라와 있습니다.

## 설치
```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env   # 이후 .env에 본인의 키를 입력
```

## 실행
```bash
.venv/bin/python -m bridge notices ECE   # 학과 게시판을 읽고 새 공지와 변경된 공지를 기록
.venv/bin/python -m bridge briefs ECE    # 새 공지와 변경된 공지를 번역 (모델을 호출)
.venv/bin/python -m bridge site          # site/index.html, site/cards.json, 캘린더 site/ece.ics를 빌드
```
다른 학과는 `ECE`를 `AEROSPACE`, `NAVAL` 또는 `NUCLEAR`로 바꿉니다.

캘린더(`bridge/deadlines.py`)에는 공지에 요일과 함께 적힌 날짜만 담깁니다. 코드가 각 날짜를 그 요일 및 영문 줄과 대조하여 검증합니다. 날짜는 모델에게 묻지 않습니다. 빌드는 `config.json`에 페이지의 공개 주소인 `site`가 있는 학과의 캘린더를 만듭니다. 학생은 공지사항 페이지에서 캘린더를 구독합니다.

## 누구도 직접 관여하지 않아도 최신 상태로 유지되는 방식
`.github/workflows/update.yml`은 GitHub에서 하루 한 번 실행됩니다. 예약은 서울 기준 07:17로 되어 있지만 GitHub는 예약된 작업을 늦게 시작하므로(2026년 11월 12일의 실행은 12:08에 시작되었습니다), 사이트와 포스터에는 하루 중 특정 시각을 약속하지 않습니다.

매 실행마다 모든 학과의 게시판을 읽고, 학과별로 새 공지 또는 변경된 공지를 최대 40건까지 번역하여 `data/`에 저장하고, 사이트를 다시 빌드한 뒤 GitHub Pages로 게시합니다. 사이트에 접속할 수 없는 학과가 있어도 다른 학과의 작업은 중단되지 않습니다. 페이지는 그래도 다시 빌드되며, 실행은 마지막에 실패로 표시되어 GitHub가 소유자에게 이메일을 보냅니다.

`main`에 푸시하면 게시판을 읽지 않고 사이트만 다시 게시합니다. "Run workflow" 버튼은 필요할 때 전체 실행을 수행합니다.

## 인수인계
1. 이 저장소의 접근 권한을 받고(또는 포크하고) Settings > Pages > Source: GitHub Actions에서 Pages를 켭니다.
2. Settings > Secrets and variables > Actions > New repository secret에서 본인의 모델 키를 추가합니다. 이름은 `ANTHROPIC_API_KEY`로 지정합니다(워크플로는 `API_KEY`도 받아들입니다). 다른 모델 제공사를 사용하려면 `bridge/llm.py`만 변경합니다.
3. 매 학기 새 공식 문서를 `sources/<department>/`에 내려받고, 해당 `config.json`에 기재하고, 요건 카드를 갱신합니다. 그 밖의 작업에는 사람이 필요하지 않습니다.
4. GitHub는 60일 동안 활동이 없는 저장소의 예약된 워크플로를 일시 중지하며, 그 전에 먼저 이메일을 보냅니다. 다시 시작하려면 "Run workflow"를 한 번 누릅니다.

## 요건 카드
사람이 각 학과의 규정 문서를 내려받습니다. AI 어시스턴트가 이를 바탕으로 학과마다 스크립트 하나씩으로 카드를 작성하고, 코드가 저장하기 전에 모든 카드를 한국어 원문과 대조하여 검증합니다.
```bash
(cd tools && ../.venv/bin/python build_ece.py)   # 전기·정보공학부: data/ECE/cards.json 작성
.venv/bin/python tools/pages_aerospace.py        # 항공우주공학과: 대학원 규정 페이지 읽기 (페이지마다 입학 연도별 규정 두 벌)
.venv/bin/python tools/cards_aerospace.py        # 항공우주공학과: 모든 카드를 검증하고 data/AEROSPACE/cards.json 작성
.venv/bin/python tools/cards_naval.py            # 조선해양공학과: 위와 같음, sources/NAVAL/의 규정 PDF 기준
.venv/bin/python tools/cards_nuclear.py          # 원자핵공학과: 위와 같음, sources/NUCLEAR/의 규정 PDF 기준
```
마지막 세 스크립트는 각각 `tools/cards_<department>_undergraduate.py`에서 해당 학과의 학부 카드도 불러옵니다.

## 검증
```bash
.venv/bin/python tools/check_quotes.py     # 전기·정보공학부 학부 카드: 한국어 인용문과 과목 코드가 모든 입학 연도의 문서에 있는지 확인
.venv/bin/python tools/check_updates.py    # 전기·정보공학부 카드의 "이후 공지": 각 한국어 인용문이 인용한 공지에 있는지 확인
.venv/bin/python tools/audit.py prepare    # 전기·정보공학부: 2차 검수자가 카드를 페이지 이미지와 대조하여 다시 검수 (파일 상단 참조)
.venv/bin/python tools/evidence.py         # 배경 및 방법 페이지를 위해 전기·정보공학부의 한국어 게시판과 영문 게시판을 다시 집계
.venv/bin/python tools/survey.py count     # 공과대학 10개 학과의 한국어 공지와 영문 공지를 집계
.venv/bin/python tools/survey.py aggregate # 합계를 data/survey.json에 기록 (`check`와 `sample` 명령이 집계를 검증)
.venv/bin/python tools/korean_review.py    # 모든 영어·한국어 문장 쌍을 한국어 독자용 시트로 정리
.venv/bin/python tools/reader_sheet.py     # 공지 10건을 한국어 원문과 나란히 놓은, 한국어 사용 독자용 시트
.venv/bin/python tools/poster.py           # 경진대회 포스터를 out/poster/에 PDF로 생성, 사이트와 같은 데이터로 제작 (먼저: .venv/bin/pip install segno)
```
`sources/context.json`에는 배경 및 방법 페이지의 첫 단계에 쓰이는 사실과 링크가 담겨 있으며, 이 단계는 올해 대학이 밝히고 시행한 내용을 다룹니다. 그곳의 한 줄을 바꾸기 전에 인용된 각 페이지를 다시 읽습니다.

두 출처의 내용이 다를 때는 가장 최근의 공식 출처가 기준입니다. `sources/<department>/updates.json`에 공지 번호와 정확한 한국어 줄을 한 줄 추가합니다. 그러면 요건 카드의 "이후 공지로 추가되거나 변경된 내용" 아래에 표시됩니다.

## 학과의 자체 문서
공식 PDF와 스프레드시트는 우리 문서가 아니라 학과의 문서이므로 이 저장소에 없습니다. `sources/<department>/config.json`은 각 파일을 내려받은 페이지와 함께 나열합니다. 요건 카드 스크립트, `tools/check_quotes.py` 또는 `tools/audit.py`를 실행하기 전에 해당 폴더에 내려받습니다. 일일 업데이트와 사이트 빌드에는 이 파일이 필요하지 않습니다.

## 학과 추가하기
전기·정보공학부가 주 학과입니다(설정에 `"main": true`로 지정). 사이트는 이 학과로 열리며, 배경 및 방법 페이지는 이 학과의 게시판을 다룹니다. 항공우주공학과, 조선해양공학과, 원자핵공학과는 각각 약 하루 만에 네 단계로 추가되었습니다.

1. **공지.** 게시판 주소와 `since` 날짜를 넣어 `sources/<NAME>/config.json`을 만듭니다. 시작점으로 `sources/NUCLEAR/config.json`을 복사합니다. 먼저 본인의 컴퓨터에서 `python -m bridge notices <NAME>`과 `python -m bridge briefs <NAME>`을 실행합니다. 일일 작업이 GitHub에서 공지를 모두 번역하게 되므로, 번역된 공지가 없는 새 학과 폴더는 절대 푸시하지 않습니다. `bridge/notices.py`와 `bridge/briefs.py`의 읽기 코드는 이 네 학과가 사용하는 게시판 소프트웨어에 맞춰져 있습니다. 다른 소프트웨어에는 별도의 읽기 코드 두 개가 필요하며, `tools/survey.py`가 다른 공과대학 학과 사이트가 게시물을 나열하는 방식을 보여 줍니다. 응답이 느린 사이트는 설정의 `board.pause`를 요청 사이에 기다릴 초 단위 시간으로 지정합니다(조선해양공학과는 5를 사용합니다).
2. **요건.** 사람이 학과의 졸업 규정 문서를 `sources/<NAME>/`에 내려받습니다. 그곳의 PDF는 절대 업로드하지 않습니다. AI 어시스턴트가 `prompts/write-cards.md`를 바탕으로 `tools/cards_<name>.py`를 작성합니다. `tools/cards_pdf.py`의 검증은 한국어 인용문이 문서에 없거나 숫자가 인용문에 없는 카드를 거부합니다. 웹 페이지로 게시된 규정은 `tools/pages_aerospace.py`를 따릅니다. 학부 규정은 입학 연도에 따라 다르므로, 학부 카드마다 `entry_years`를 나열합니다. 연도마다 별도의 문서가 있는 경우, 카드의 인용문은 그 문서 각각에 있어야 합니다. `tools/cards_nuclear_undergraduate.py`가 가장 짧은 예시입니다.
3. **2차 검수.** 다른 개발사의 모델이 `prompts/read-cards.md`를 사용해 모든 카드를 한국어 원문과 대조하여 검수합니다. 카드를 수정하고, 지적 사항이 학생이 해야 할 일을 더 이상 바꾸지 않을 때까지 반복합니다. 결과를 `data/<NAME>/audit.json`에 기록합니다.
4. **게시.** 설정에 `programs`, `headline_cards`, `files`를 추가하고 `python -m bridge site`를 실행합니다. 헤더 아래의 전환 메뉴에서 학과를 확인한 뒤 푸시합니다.

`board`와 `"pilot": true`는 있으나 카드가 없는 폴더는 공지사항 페이지에만 시범으로 표시됩니다.

## ChatGPT Edu에서 하는 부분
`navigator/instructions.md`에는 이 사이트가 읽지 않는 내용, 즉 이미지, 첨부 파일, 그리고 공지에 대한 학생 본인의 질문을 위해 학생이 사용하는 AI 어시스턴트의 지침이 담겨 있습니다. ChatGPT Edu 프로젝트에 붙여 넣습니다. 지침은 일반 텍스트이므로, 워크스페이스가 플러그인이나 스킬을 제공하면 그쪽으로 옮길 수 있습니다.

## 소스 보내기
발송하는 날에는 먼저 풀(pull)합니다. 일일 작업은 새 공지를 GitHub에 저장하며, 포스터에는 빌드된 날짜가 인쇄됩니다.
```bash
git pull
.venv/bin/python tools/poster.py
git archive --format=zip --prefix=snubridge/ -o out/snubridge-source.zip HEAD
```
보내기 전에 `out/poster/poster.png`를 확인합니다. 마지막 명령은 저장소에 있는 내용만을 정확히 압축 파일로 묶습니다. 여기에는 키도, 내려받은 학과 문서도, `out/`의 어떤 파일도 들어 있지 않습니다.

경진대회에 제출하는 압축 파일에는 `python -m bridge site` 실행 후 추가한 `site/`(빌드된 페이지, 그 데이터, 캘린더 파일)도 들어 있어 빌드 없이도 사이트를 열 수 있으며, `site/index.html`을 열면 됩니다.
