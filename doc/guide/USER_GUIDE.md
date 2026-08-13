# MyRemember 사용설명서

> 기준일: 2026-08-11 — 처음 시작하는 사람 기준으로 처음부터 끝까지 순서대로 따라갈 수 있게 정리했다.
> 전체 기능 목록/설계 배경은 [`README.md`](../../README.md), [`ARCHITECTURE.md`](../../ARCHITECTURE.md) 참고.

## 0. 사전 준비

| 필요한 것 | 용도 | 설치 |
|---|---|---|
| Python 3.11+ | 모든 자동화 스크립트 | macOS는 보통 기본 설치돼 있음(`python3 --version`으로 확인) |
| pandoc | 웹뷰어 생성(`generate-html.sh`) | `brew install pandoc` |
| poppler (선택) | "노트 가져오기"의 PDF 지원(`pdftotext`) | `brew install poppler` — 없어도 `.md`/`.html`/`.txt` 가져오기는 동작 |
| ripgrep | 빠른 텍스트 검색(`search.sh`) | `brew install ripgrep` |
| fzf (선택) | `search.sh`의 인터랙티브 필터링 | `brew install fzf` — 없어도 ripgrep 결과는 그대로 보임 |
| rclone (선택) | `backup.sh`의 클라우드(Google Drive) 백업 | `brew install rclone` — 없으면 로컬 백업만 동작 |

## 1. 최초 1회 설정

```bash
cd /Volumes/data/vibecoding/myremember
pip install -r scripts/requirements.txt
```

`scripts/requirements.txt`에는 `PyYAML`, `Jinja2` 두 개만 들어있다 — 스크립트가 `scripts/config.yaml`을
읽고(PyYAML) 웹뷰어 HTML을 렌더링(Jinja2)하는 데 쓴다.

## 2. 오늘의 daily note로 시작하기

daily note는 하루 단위로 무엇을 했는지 기록하는 타임시리즈 노트다. 아침에 한 번 만들어두고
하루 종일 채워나가는 방식으로 쓴다. 두 가지 방법이 있다.

```bash
# (A) 터미널에서
python3 scripts/create-daily-note.py

# (B) 웹뷰어 대시보드에서 — scripts/preview.sh로 띄운 상태에서 "오늘 노트 열기" 버튼
```

- `daily/2026-08-11.md`처럼 오늘 날짜 파일이 생성된다.
- 이미 오늘 파일이 있으면 아무것도 하지 않는다(멱등) — 하루에 여러 번 실행해도 안전하다.
- `daily/index.md`(월별 인덱스)도 함께 자동 갱신된다.
- 특정 날짜로 만들고 싶으면: `python3 scripts/create-daily-note.py --date 2026-08-10`

생성된 파일은 `scripts/config.yaml`의 `daily_note.template`을 따라 아래 구조로 시작한다.

```markdown
# 2026-08-11

## 오늘의 주제
-

## 진행 상황
-

## 배운 점
-

## 관련 노트
-

## 내일 할 일
- [ ]
```

VS Code로 바로 열어서 채우면 된다: `code daily/$(date +%Y-%m-%d).md`

## 3. 프로젝트/주제 노트 만들기

daily note 말고, 좀 더 오래 유지되는 지식은 두 곳 중 하나에 쓴다.

- **프로젝트(진행 중이거나 끝난 일)**: `areas/{프로젝트명}/README.md` 파일을 새로 만든다.
  필요하면 같은 폴더에 `progress.md`(진행 상황), `decisions.md`(의사결정)도 추가한다.
- **주제(계속 관심을 두는 분야)**: `topics/{주제명}.md` 파일을 새로 만든다. 이미 다른 곳에서
  써둔 `.md`/`.html`/`.pdf` 파일이 있다면 새로 타이핑할 필요 없이 대시보드의 "노트 가져오기"
  버튼으로 바로 등록할 수 있다(8절 참고).

양식 예시는 [`myremember_prd.md`](../../myremember_prd.md)의 "부록: 마크다운 양식 예시"에 있다.
파일명은 폴더 안에서 고유해야 한다 — 같은 이름이 여러 개면 링크가 어느 쪽을 가리키는지
모호해진다(`validate-links.py`가 이런 경우를 "모호한 링크"로 따로 잡아준다).

## 4. 노트끼리 연결하기 — `[[위키링크]]`

어떤 노트 안에서든 다른 노트 이름을 `[[대괄호 두 개]]`로 감싸면 자동으로 링크가 된다.
확장자(`.md`)는 빼고 파일명만 쓴다.

```markdown
오늘 [[3d-printer-research]] 관련해서 조사했다. 진행 중인 [[audio-article-highlight]]
프로젝트에도 참고할 수 있을 듯.
```

- `areas/{프로젝트}/README.md`처럼 "폴더 자체가 노트"인 경우엔 폴더명으로 링크할 수 있다
  (`[[audio-article-highlight]]`가 `areas/audio-article-highlight/README.md`를 가리킨다).
- 헤딩까지 지정하거나(`[[note#헤딩]]`) 표시 텍스트를 바꾸는 것(`[[note|표시할 글자]]`)도 된다.

## 5. 링크가 안 깨졌는지 확인하기

```bash
python3 scripts/validate-links.py
```

- 문제없으면 `링크 N개 검사 완료 - 깨진 링크 0개`가 뜨고 종료(exit code 0).
- 깨진 링크가 있으면 `output/link-report.md`에 어떤 파일 몇 번째 줄에서 어떤 링크가 깨졌는지
  나오고, 종료 코드가 1이 된다(나중에 cron/CI에서 실패로 감지할 수 있게).
- 노트를 쓰다가 오탈자로 링크가 깨지는 걸 잡아내는 용도라, 주 1회 정도 습관적으로 돌리면 좋다.

## 6. 검색하기

두 가지 방법이 있다.

```bash
# (A) 그때그때 바로 검색 — 인덱스 필요 없음, 항상 최신 내용
scripts/search.sh "찾고 싶은 단어"

# (B) SQLite 인덱스를 미리 만들어두고 검색 — 노트가 많아지면 (A)보다 빠름
python3 scripts/build-search-index.py                          # 인덱스 생성/갱신
python3 scripts/build-search-index.py --query "찾고 싶은 단어"   # 검색만 (재생성 없음)
python3 scripts/build-search-index.py --query "찾고 싶은 단어" --limit 20  # 결과 개수 조정(기본 10)
```

(A)는 ripgrep이 vault 파일을 직접 훑기 때문에 방금 저장한 내용도 바로 검색된다. fzf가 설치돼
있으면 결과를 다시 인터랙티브하게 필터링할 수 있다. (B)는 제목/본문/태그를 SQLite FTS5로
색인해두는 방식이라 노트 수가 많아졌을 때 유리하다.

## 7. 노트 간 연결 관계(백링크) 보기

```bash
python3 scripts/build-backlinks.py
cat output/backlinks.json   # 또는 VS Code로 열어서 확인
```

`output/backlinks.json`에는 각 노트가 어떤 노트를 링크했는지(`edges`)와, 반대로 어떤 노트에게
링크받았는지(`backlinks`)가 들어있다. 이 파일은 웹뷰어의 "이 노트를 링크한 노트" 패널에도
그대로 쓰인다.

## 8. 웹뷰어(개인 위키 사이트)로 둘러보기

```bash
scripts/preview.sh
```

한 번에 백링크 갱신 → 웹뷰어(`output/html`) 생성 → 로컬 서버 실행까지 끝난다.
`http://localhost:4500`을 브라우저로 열면 사이드바(대시보드/노트/태그/검색/Daily 아카이브)가
있는 개인 위키가 뜬다. 종료는 `Ctrl+C`, 포트를 바꾸려면 `scripts/preview.sh 8080`처럼 인자를
넘긴다.

지금은 로컬 전용이다 — GitHub Pages 같은 공개 배포는 하지 않는다. vault에 개인적인 프로젝트/
일상 메모가 쌓이는데, GitHub Free 플랜은 저장소가 private이어도 Pages 사이트 자체는 URL을
아는 누구나 볼 수 있는 public이 되기 때문이다(자세한 이유는 `ARCHITECTURE.md` 참고). 이
사용설명서를 포함한 PRD/개발기록만 별도의 공개 문서 사이트(`myremember-docs`)로 분리해 공개한다.

### 사이드바의 두 액션 버튼

탐색 링크(대시보드/노트/태그/검색/Daily 아카이브) 아래 구분선 밑에 자주 쓰지 않는 액션
두 개가 있다 — 모든 페이지에서 보인다.

- **오늘 노트**: 눌러서 없으면 오늘 Daily Note를 만들고, 이미 있으면 바로 그 노트로
  이동한다. `scripts/create-daily-note.py`를 터미널에서 직접 안 돌려도 이 버튼 하나로
  끝난다. **단, `scripts/preview.sh`(=`scripts/webviewer/server.py`)로 띄웠을 때만
  동작한다** — 실제로 파일을 쓰는 요청이라 정적 파일 서버로는 처리할 수 없다.
- **새 노트**: 전용 페이지로 이동해서 제목과 내용을 직접 입력해 바로 노트를 만들 수
  있다. 제목 칸에 제목을, 내용 칸에 본문을 쓰고 "저장"을 누르면 `topics/`에 등록되고
  바로 그 페이지로 이동한다 — "노트 가져오기"와 저장 로직(자동 태그 포함)을 그대로
  공유한다. 이 페이지도 `scripts/preview.sh`로 띄웠을 때만 동작한다.

### 대시보드 (첫 화면)

`index.html`(사이드바의 "대시보드")이 기본 첫 화면이다. vault 개요(areas/topics/daily
목록)는 사이드바의 "노트"로 옮겨졌다. 위→아래 순서대로:

- **통계 카드**: 오늘 할 일 건수·vault 노트 개수·태그 개수를 한눈에.
- **할 일**: 날짜별 체크리스트. 직접 추가/완료 토글/수정/삭제가 가능하고 캘린더 연동은 없다.
  항목에 마우스를 올리면 나오는 연필 아이콘을 누르면 바로 텍스트를 고칠 수 있다(Enter로 저장,
  Escape로 취소). 원본은 이 브라우저의 `localStorage`이고 vault 마크다운 파일과는 무관하다 —
  다만 `scripts/preview.sh`(로컬 서버)로 열었을 때는 저장할 때마다 서버 파일
  (`scripts/webviewer/data/todos.json`)로도 미러링돼서, 사이트를 다시 만들고 배포하면 그
  시점 스냅샷을 모바일 등 다른 기기에서도 볼 수 있다. 다른 기기에서의 체크/수정은 그
  기기에만 남고 이 컴퓨터로 되돌아오진 않는다(뷰어 성격, 양방향 동기화 아님).
- **노트 그래프**: 작은 네트워크 미리보기가 있다. 같은 태그를 가진 노트끼리, `[[링크]]`로
  연결된 노트끼리 가까이 모이도록 자동 배치되고, 태그별로 색이 다르다(등장 빈도 상위
  8개 태그만 고유 색, 그 밖과 태그 없는 노트는 회색 "기타"). 점 위에 마우스를 올리면
  제목/태그가 뜨고, 클릭하면 그 노트로 바로 이동한다. 물리적으로 폴더를 나누지 않아도
  되고, 태그를 붙이는 것만으로 이 그래프에 반영된다(직접 쓰는 노트는 본문 끝에 `#태그`를
  손으로 적어야 한다 — 아래 "노트 가져오기" 참고). `[[링크]]`를 아직 하나도 안 썼다면
  점들이 태그로만 묶이고 서로 이어진 선은 안 보이는 게 정상이다.
- **노트 가져오기**: 이미 갖고 있는 `.md`/`.markdown`/`.html`/`.htm`/`.txt`/`.pdf` 파일을
  제목 옆 "파일 선택"으로 고르면 `topics/`에 노트로 등록되고 바로 그 페이지로 이동한다.
  HTML은 자동으로 마크다운으로 변환되고(pandoc), PDF는 텍스트만 추출된다(`pdftotext`,
  poppler 필요 — 위 1절 참고). 스캔한 이미지로만 된 PDF(텍스트 레이어 없음)는 추출할
  내용이 없어 가져오기가 실패한다. 제목은 파일의 첫 헤딩(HTML은 `<title>` 태그)에서
  가져오고, 파일명은 그 제목을 슬러그화해서 만든다 — 같은 이름이 이미 있으면 `-2`,
  `-3`처럼 번호가 붙어 기존 노트를 덮어쓰지 않는다.
  **태그도 자동으로 붙는다**: 이미 vault에서 쓰이고 있는 태그 중 이 노트 내용에 실제로
  등장하는 것을 먼저 재사용하고, 부족하면 제목에서 뽑은 후보로 채운다. 완벽한 주제 분류는
  아니고(형태소 분석기 없이 조사만 대충 떼는 수준) 가끔 어색한 태그가 붙을 수 있다 —
  그럴 땐 노트를 열어 "편집"으로 직접 고치거나, 태그 자체는 그냥 본문 맨 아래 한 줄
  (`#태그1 #태그2`)이라 지우고 새로 써도 된다.
  **"원본 보관" 아이콘(제목 줄 오른쪽 끝, PDF 전용)**: 마우스를 올리면 설명이 뜨는 작은
  상자 아이콘 — 클릭해서 켜면(색이 바뀜) PDF를 가져올 때 텍스트만 md로 등록되는 건
  똑같지만, 원본 PDF 파일도 `attachments/{노트 제목}/`에 같이 저장된다(`.md`/`.html`은
  이 옵션이 적용되지 않음 — 가져온 결과가 원본과 거의 같아서 원본을 따로 보관할 실익이
  적다). 이 폴더는 git에는 안 올라가고, 다음에 `scripts/backup.sh`를 실행할 때 로컬
  백업(`backups/{YYYY-MM}.tar.gz`)에 포함되고, rclone으로 Google Drive가 설정돼 있으면
  그때 클라우드에도 같이 올라간다 — **켠다고 그 자리에서 바로 업로드되는 건 아니고**,
  다음 백업 실행을 기다린다(11절 참고). 노트를 "삭제"하면 매칭되는 `attachments/`
  폴더도 같이 지워진다. 이 카드는 `scripts/preview.sh`로 띄웠을 때만 동작한다.
- **빠른 메모**: 나중에 daily note나 topic으로 옮길 메모를 잠깐 적어두는 스크래치 공간.
  역시 `localStorage`에만 저장된다 — 정식 기록은 daily note/topic 노트에 남겨야 한다.
- **Gmail / 캘린더 / 관심종목 스냅샷**: `scripts/webviewer/data/dashboard-snapshot.json`이
  있으면 표시된다. 이 데이터는 Gmail/Google Calendar MCP 도구가 있어야 가져올 수 있어서
  `generate-html.sh`가 자동으로 채우지 못한다 — **Claude Code 세션에서 "대시보드 스냅샷
  갱신해줘"라고 요청**하면 Claude가 실제 데이터를 가져와 이 파일을 다시 쓰고, 그 다음
  `scripts/generate-html.sh`(또는 `preview.sh`)를 돌리면 반영된다. 스냅샷이 없으면 빈
  상태로 표시될 뿐 빌드가 실패하지는 않는다.
  **캘린더는 접근 가능한 캘린더 전체를 합쳐서 보여준다**(기본 캘린더 하나만이 아님),
  범위는 **오늘 기준 3일(오늘 + 다음 2일)**. 어느 캘린더에서 왔는지 배지는 화면에 안
  뜨지만, **본인 캘린더(`조충남 개인일정`)에서 온 일정만 제목이 굵게** 표시돼서 다른
  사람/조직 캘린더 일정과 한눈에 구분된다. 보고 싶지 않은 캘린더가 있으면 "다음번
  스냅샷 갱신할 때 [캘린더 이름]은 빼줘"라고 요청하면 된다.

### 노트 편집 · 삭제 (브라우저에서 바로)

daily/areas/topics, 어떤 노트를 열어도 우측 상단에 "편집"/"삭제" 버튼이 있다.

- **편집**: 누르면 원본 마크다운이 그대로 담긴 편집창이 뜨고, 고친 뒤 "저장"을 누르면 그
  파일이 바로 바뀌고 사이트가 다시 생성돼 새 내용이 즉시 보인다("취소"를 누르면 저장 없이
  원래 화면으로 돌아간다). 텍스트 에디터로 파일을 열 필요 없이, 웹뷰어를 보다가 바로 고칠
  수 있다는 뜻이다.
- **삭제**: 누르면 정말 지울지 한 번 더 물어보고(**되돌릴 수 없음** — git으로 커밋해둔
  상태였다면 거기서 복구 가능), 확인하면 파일이 지워지고 노트 목록으로 이동한다. 노트를
  열지 않고도 지우고 싶으면, "노트"/"태그"/"Daily 아카이브" 페이지의 목록에서 각 항목에
  마우스를 올리면 나오는 휴지통 아이콘으로 바로 지울 수 있다(가져오기 하다가 같은 파일을
  실수로 두 번 등록했을 때 특히 유용).

이 버튼들 전부 `scripts/preview.sh`로 띄웠을 때만 동작한다(다른 방식으로 서빙하면 버튼은
뜨지만 클릭 시 에러가 뜬다).

### 태그 관리 (병합 + 상위/하위 관계)

사이드바 "태그" 페이지 상단의 "태그 관리" 버튼을 누르면 두 가지를 할 수 있다.

- **태그 병합**: "노트 가져오기"의 자동 태그가 가끔 하나의 개념을 여러 태그로 쪼개는 경우가
  있다(예: "3D 프린터 완벽 가이드"가 `#3D #가이드 #완벽 #프린터`로 나뉘는 식). 목록에서
  합칠 태그를 여러 개 선택하고, 합칠 이름을 입력한다(이미 있는 태그를 골라도 되고 완전히
  새 이름이어도 된다). "병합"을 누르면 확인창이 뜨고, 확인하면 **선택한 태그가 쓰인 모든
  노트 파일이 실제로 수정된다**(되돌릴 수 없음 — git으로 커밋해뒀다면 거기서 복구 가능).
- **태그 관계(상위/하위)**: 태그마다 상위 태그를 여러 개 지정할 수 있다(예: "3D프린터"의
  상위로 "하드웨어"와 "취미"를 둘 다 지정 가능). 위쪽 드롭다운에서 태그를 하나 고르면
  그 아래 현재 상위 태그가 알약 모양 칩으로 나타난다. 새 상위를 추가하려면 드롭다운에서
  고르고 "+ 추가"를 누르면 되고, 뺄 때는 칩 옆 `×`를 누르면 된다 — 둘 다 누르는 즉시
  저장되고 별도 "저장" 버튼은 없다. 순환 참조(A의 상위가 B, B의 상위가 다시 A)를
  만들려고 하면 저장이 거부되고 에러 메시지가 뜬다. 설정한 관계는 그 태그의 페이지(상위는
  브레드크럼, 하위는 목록)와 대시보드 "노트 그래프" 위젯(하위 태그 허브가 상위 태그 허브
  쪽으로 끌리고 점선으로 연결됨)에 바로 반영된다.

이 페이지도 `scripts/preview.sh`로 띄웠을 때만 동작한다.

## 9. 하루/주 단위로 반복하는 흐름

1. 아침에 `python3 scripts/create-daily-note.py` 실행하고 그 파일에 하루를 기록
2. 노트를 쓰다가 관련 있는 다른 노트가 있으면 `[[노트이름]]`으로 링크
3. 가끔(주 1회 정도) `python3 scripts/validate-links.py`로 깨진 링크 없는지 점검
4. 뭔가 찾고 싶을 때 `scripts/search.sh "검색어"`
5. 노트가 꽤 쌓였으면 `build-search-index.py`, `build-backlinks.py`를 다시 돌려서
   검색/그래프를 최신 상태로 갱신하고, 웹뷰어로 훑어보고 싶으면 `scripts/preview.sh`

## 10. 버전 관리 — `git-auto-commit.sh`

vault 변경사항을 커밋하고, 매달 스냅샷 태그를 남기는 스크립트다.

```bash
scripts/git-auto-commit.sh              # 변경사항 커밋 + 이번 달 태그(v2026-08) — push는 안 함
scripts/git-auto-commit.sh --push       # 위 작업 후 origin에 push 시도 (터미널에서 y/N 확인)
scripts/git-auto-commit.sh --push --yes # 확인 없이 바로 push (cron 등 무인 실행 전용)
```

**push는 기본적으로 하지 않는다.** git push는 원격 저장소(GitHub)에 실제로 영향을 주는
작업이라 `--push`를 명시해야 시도하고, 사람이 터미널 앞에 있을 때는 한 번 더 `y/N` 확인을
받는다. 터미널이 없는 실행(예: cron)에서는 `--yes`를 주지 않는 한 push를 건너뛰고 로컬
커밋/태그만 남긴다 — 실수로 원격을 건드리는 일이 없도록 하는 안전장치다.

## 11. 백업 — `backup.sh`

```bash
scripts/backup.sh          # 로컬 압축 백업 생성 + (설정돼 있으면) 클라우드 업로드 확인
scripts/backup.sh --yes    # 클라우드 업로드 확인 없이 바로 진행 (무인 실행 전용)
```

- **로컬 백업**(`backups/{YYYY-MM}.tar.gz`)은 매번 자동으로 만들어진다 — vault 밖으로 나가지
  않는 작업이라 확인 없이 실행된다. "노트 가져오기"에서 "원본 보관"을 체크해 둔 PDF
  원본(`attachments/`)도 이 안에 같이 포함된다.
- **클라우드 백업**(Google Drive 등)은 선택 사항이다. 아래 순서로 설정해야 동작한다.

  ```bash
  brew install rclone
  rclone config        # 대화형 설정 — Google Drive 등을 선택하고 브라우저에서 로그인
  ```

  설정한 remote 이름을 `scripts/config.yaml`의 `backup.rclone_remote`에 적는다:

  ```yaml
  backup:
    rclone_remote: "gdrive"          # rclone config에서 만든 이름
    remote_path: "MyRemember-backups"
  ```

  이렇게 설정해두면 `backup.sh` 실행 시 클라우드 업로드 여부를 물어본다(`--yes`면 바로 진행).
  `rclone`이 없거나 remote가 설정 안 돼 있으면 클라우드 단계는 조용히 건너뛰고 로컬 백업만
  남긴다 — 아직 설정 전이라도 스크립트를 실행하는 데 문제는 없다.

## 12. 전체 명령어 치트시트

| 명령 | 하는 일 |
|---|---|
| `python3 scripts/create-daily-note.py [--date YYYY-MM-DD]` | 오늘(또는 지정 날짜) daily note 생성 |
| `python3 scripts/validate-links.py` | `[[링크]]` 무결성 검사 |
| `python3 scripts/build-search-index.py [--query "..." [--limit N]]` | 검색 인덱스 생성 또는 검색 |
| `python3 scripts/build-backlinks.py` | 역참조 그래프 생성 |
| `scripts/search.sh "검색어"` | ripgrep(+fzf) 즉시 검색 |
| `scripts/generate-html.sh` | 정적 웹뷰어 생성 |
| `scripts/preview.sh [포트]` | 백링크 갱신 + 웹뷰어 생성 + 로컬 서버 (기본 4500, "오늘 Daily Note"·"노트 가져오기" 버튼 포함) |
| `scripts/git-auto-commit.sh [--push] [--yes]` | 커밋 + 월간 태그 (+선택적 push) |
| `scripts/backup.sh [--yes]` | 로컬 백업 (+설정된 경우 클라우드 업로드) |
| `python3 scripts/build-dashboard-snapshot.py [--init]` | 대시보드 스냅샷 JSON 검증(또는 빈 틀 생성). 데이터 자체는 Claude 세션에서만 채울 수 있음 |

## 13. 막혔을 때

- **"깨진 링크"가 자꾸 뜬다**: `output/link-report.md`를 열어서 정확히 몇 번째 줄, 어떤
  링크인지 확인한다. 대상 파일명 오타이거나, `areas/{project}/README.md`를 폴더명이 아니라
  `readme`로 링크한 경우가 흔하다.
- **"모호한 링크"가 뜬다**: 같은 파일명이 여러 폴더에 있다는 뜻이다(예: `areas/a/README.md`,
  `areas/b/README.md`를 둘 다 `[[README]]`로 링크). 폴더명으로 링크하거나 파일명을 다르게
  바꾼다.
- **검색 결과가 이상하게 안 나온다**: `build-search-index.py`는 재생성할 때마다 인덱스를
  통째로 새로 만든다 — 노트를 옮기거나 지운 뒤에는 한 번 다시 돌려야 인덱스가 최신 상태가
  된다.
- **웹뷰어가 안 열린다**: `pandoc`이 설치돼 있는지(`pandoc --version`) 먼저 확인한다.
- **PDF 가져오기가 실패한다**: `pdftotext -v`로 poppler가 설치돼 있는지 먼저 확인한다
  (`brew install poppler`). 설치돼 있는데도 실패하면 스캔한 이미지로만 된 PDF(텍스트
  레이어 없음)일 가능성이 크다 — 이 경우 OCR이 필요한데 현재 지원 범위 밖이다.
- **"가져오기"/"편집"/"삭제" 버튼을 눌러도 아무 반응이 없다**: 브라우저가 예전 페이지를
  캐시하고 있을 가능성이 크다 — 강력 새로고침(Mac 크롬/사파리 `Cmd+Shift+R`)을 한 번
  해보고 다시 시도한다. 그래도 안 되면 `scripts/preview.sh`로 띄운 주소(기본
  `http://localhost:4500`)가 맞는지, 다른 포트나 `file://`로 열지는 않았는지 확인한다.
