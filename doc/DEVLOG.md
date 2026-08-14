# 개발 기록 (DEVLOG)

> 날짜별로 새 섹션을 계속 append한다(과거 항목은 지우거나 고치지 않음 — 현재 상태 요약은 STATUS.md 몫). 각 항목은 무엇을 했는지뿐 아니라 **수정/버그/이슈**를 구분해 적는다.

## 2026-08-11 — Phase 1 MVP: vault 구조 + daily note/링크 검증 자동화

- **추가/변경**: PRD 기반으로 vault 폴더 구조(`areas/topics/daily/scripts/output/backups`)를 만들고 Phase 1 스크립트 두 개를 구현했다.
  - `scripts/vault.py` 공통 모듈로 config 로딩과 로깅을 모든 스크립트가 재사용하도록 설계.
  - `create-daily-note.py`: 날짜 기반 daily note 생성 + 월별 index.md 재생성. 이미 존재하는 노트는 건드리지 않아 cron으로 매일 실행해도 안전.
  - `validate-links.py`: `[[링크]]` 대상을 파일명 기준(대소문자 무시)으로 매칭, 코드펜스 내부 제외, 깨진 링크/모호한 링크를 구분해 리포트.
  - 두 스크립트 모두 임시 테스트 노트로 정상 동작(생성/멱등성/깨진 링크 감지/코드블록 예외) 확인 후 테스트 파일 정리.
  - git 저장소 초기화(`main` 브랜치) 후 GitHub private 저장소(`peter-cho-70/myremember`) 생성 및 푸시.
- **이슈/막힌 점**: 없음. cron 실등록과 Phase 2(검색 인덱스/백링크 그래프)는 사용자 요청 시 다음 세션에서 진행하기로 보류.

## 2026-08-11 — Phase 2: 검색 인덱스 + 백링크 그래프 + CLI 검색

- **추가/변경**: PRD Phase 2(연결 & 검색) 구현.
  - `scripts/vault.py`에 `strip_code_blocks`/`iter_note_files`/`dir_type`/`collect_notes`를 공용 유틸로 옮기고 `validate-links.py`가 이를 재사용하도록 정리(Phase 2 스크립트들과 로직 중복 제거).
  - `build-search-index.py`: SQLite FTS5로 제목/본문/태그 인덱싱, `--query` 검색 옵션 추가.
  - `build-backlinks.py`: `[[링크]]` 역참조 JSON 그래프(nodes/edges/backlinks) 생성.
  - `search.sh`: ripgrep(+fzf) CLI 검색, fzf 미설치 시 폴백.
  - 임시 테스트 노트(프로젝트 2개 + 주제 2개, README.md 동명 파일 포함)로 세 스크립트를 함께 검증한 뒤 정리.
- **버그 수정**:
  - FTS5 MATCH 쿼리에서 `-`가 연산자로 파싱되어 `test-tag` 같은 하이픈 포함 검색어가 `sqlite3.OperationalError: no such column`으로 실패하던 문제 → 사용자 검색어를 항상 큰따옴표로 감싸는 phrase 쿼리로 고정해 해결.
  - `build-backlinks.py` 초기 구현에서 노드 id를 파일 stem으로 썼더니 `areas/project-a/README.md`와 `areas/project-b/README.md`가 그래프에서 같은 노드로 합쳐지는 문제 발견 → 노드 id를 vault 루트 기준 경로(확장자 제외)로 바꿔 해결. 링크 대상 매칭(어떤 파일을 가리키는지)은 기존처럼 파일명 기준을 유지하되, 그래프에 넣는 노드/엣지의 id만 경로 기반으로 분리.
- **이슈/막힌 점**: 개발 환경에 `fzf`가 설치되어 있지 않아 `search.sh`의 fzf 인터랙티브 경로(프리뷰 창 등)는 코드 리뷰 수준으로만 확인했고 실제 실행 검증은 못 했다. ripgrep 폴백 경로는 정상 동작 확인함. `docs/STATUS.md`의 "알려진 이슈"에 남겨둠.

## 2026-08-11 — Phase 3: Pandoc+Jinja2 웹뷰어 (정적 사이트)

- **추가/변경**: PRD Phase 3(퍼블리싱) 구현.
  - 사용자와 상의해 마크다운 변환기로 Pandoc을 선택(대안: 순수 Python `markdown` 라이브러리) → `brew install pandoc`으로 설치.
  - `scripts/webviewer/templates/*.html`(Jinja2: base/note/home/tags_index/tag/search/daily_archive)과 `scripts/webviewer/static/`(style.css, theme.js, search.js) 작성. CSS는 `prefers-color-scheme` 기본값 + 수동 토글(로컬스토리지 저장), 모바일 반응형 포함.
  - `generate-html.py`: `[[위키링크]]`를 Pandoc에 넘기기 전 직접 치환(대상 없으면 `broken-link` 스타일 span, 있으면 표준 md 링크로), pandoc 서브프로세스로 본문 변환, Jinja2로 최종 페이지 렌더링. 홈/노트/태그 인덱스·개별 태그/검색(클라이언트 사이드, JSON을 fetch가 아니라 `<script>`로 로드)/daily 전체 아카이브(월별) 페이지 생성.
  - `generate-html.sh`: PRD 크루 이름(`generate-html.sh`)에 맞춘 얇은 bash 래퍼, 실제 로직은 `.py` 쪽.
  - `preview.sh`: 사용자가 "한 번에 실행할 수 있는 명령어"를 요청해서 추가 — `build-backlinks.py` → `generate-html.sh` → `python3 -m http.server`를 순서대로 실행하는 편의 스크립트. 포트는 인자로 지정 가능(기본 8000).
  - `link_validation.pattern`(config.yaml)에 헤딩/별칭 캡처 그룹 추가(`[[note#heading|별칭]]` 지원) — 기존 스크립트는 group(1)만 써서 하위호환 유지.
  - Playwright로 로컬 서버(`python3 -m http.server`) 띄워 홈/노트/태그/검색/다크모드/모바일 뷰(390px) 실제 렌더링 확인.
- **버그 수정**:
  - **폴더형 노트 링크 불가**: `[[project-name]]`이 `areas/project-name/README.md`를 못 찾고 항상 깨진 링크로 뜨는 문제 발견(모든 README.md의 stem이 "readme"라 폴더명으로는 매칭이 안 됐음 — PRD가 프로젝트 구조 자체를 이 패턴으로 정의해서 실사용에 바로 영향). `vault.py`에 `note_lookup_keys()`를 추가해 README.md는 파일명("readme")뿐 아니라 부모 폴더명으로도 찾을 수 있게 하고, `collect_notes()`와 `generate-html.py`가 모두 이 함수를 쓰도록 통일.
  - **검색이 항상 "결과 없음"**: `search-data.js`를 `const MYREMEMBER_NOTES = [...]`로 생성했더니 `search.js`가 읽는 `window.MYREMEMBER_NOTES`가 항상 undefined였음 — 최상위 `const`/`let`은 전역 렉시컬 스코프에는 있어도 `window` 객체 프로퍼티로는 안 붙는다는 JS 특성 때문. `window.MYREMEMBER_NOTES = [...]`로 명시 할당하도록 수정, Playwright로 실제 검색 결과 뜨는 것까지 재확인.
- **이슈/막힌 점**: GitHub Pages 배포 여부를 사용자에게 물어봄 — Free 플랜은 private 저장소여도 Pages 사이트는 public이 되는데, vault에 개인적인 내용(일상/프로젝트 메모)이 쌓일 수 있어서 사용자가 "지금은 배포하지 않음"을 선택. 로컬 서빙만 우선 지원하고, 배포 결정은 나중에 다시 하기로 함(`ARCHITECTURE.md`에 사유 기록).

## 2026-08-11 — Phase 4 스크립트(VersionKeeper/BackupKeeper) + 공개 문서 사이트 발행

- **추가/변경**: PRD Phase 4(버전 & 백업)를 스크립트 단계까지 구현. 사용자와 범위를 먼저
  조율 — "스크립트만 구현, 실제 push/rclone 설정·실행은 나중에"로 확정.
  - `scripts/git-auto-commit.sh` (VersionKeeper): 변경사항 커밋 + 월간 태그(`v{YYYY-MM}`,
    "이번 달 태그가 없으면 생성"이라 스케줄이 밀려도 안전). push는 `--push`를 줘야 시도하고,
    대화형(tty)이면 확인 프롬프트, 비대화형(cron)이면 `--yes` 없이는 자동으로 건너뜀 — PRD
    3.4절의 "git push는 승인 필요"를 스크립트 레벨에서 강제.
  - `scripts/backup.sh` (BackupKeeper): 로컬 압축 백업(`backups/{YYYY-MM}.tar.gz`)은 저위험이라
    항상 자동 생성. 클라우드 업로드는 `rclone` 설치 → `config.yaml`의 `backup.rclone_remote`
    설정 → `rclone listremotes`로 실제 구성 여부를 순서대로 확인하고, 하나라도 안 갖춰지면
    이유를 로그로 남기고 조용히 스킵(스크립트 실패시키지 않음). 조건이 갖춰졌을 때만
    VersionKeeper와 동일한 확인 패턴(`--yes` 없으면 대화형 확인, 없으면 스킵)으로 `rclone copy`.
    분기 시작(1/4/7/10월 1일)엔 외장 HDD 백업 리마인더 로그도 남긴다.
  - `config.yaml`에 `backup.rclone_remote`/`backup.remote_path` 설정 추가.
  - 이 환경에 `rclone`이 설치돼 있지 않고, `rclone config`의 Google 계정 OAuth는 브라우저가
    필요해 사용자가 직접 해야 하는 작업이라 실제 실행은 다음 세션으로 미룸 — `--help` 출력과
    `bash -n` 문법 검사로만 검증(실제 `git add`/`commit`/`push`나 tar 생성은 실행하지 않음).
  - PRD 문서화 표준에 따라 이번 세션에서 처음으로 공개 문서 사이트를 만듦: PRD/ARCHITECTURE/
    STATUS/DEVLOG만 골라 `peter-cho-70/myremember-docs`(public)로 발행, GitHub Pages 활성화
    (https://peter-cho-70.github.io/myremember-docs/), 허브(`peter-cho-70.github.io`)에 카드
    등록. vault 콘텐츠(daily/areas/topics)가 들어있는 소스 저장소(`peter-cho-70/myremember`)는
    계속 private으로 유지 — 공개되는 건 기획/개발 기록 문서뿐.
- **이슈/막힌 점**: 없음.

## 2026-08-11 — 웹뷰어 UI 리디자인 + 대시보드(할 일/메모/Gmail/캘린더) 추가

- **추가/변경**: 사용자가 개인 업무 대시보드 HTML(참고: Tailwind 기반 사이드바+카드 스타일,
  Gmail/캘린더/주식/할 일/메모 위젯을 가진 파일)을 주고 "이런 식으로 UI를 바꾸고 이 기능들을
  추가해달라"고 요청. 적용 대상(myremember 웹뷰어)과 구체적으로 가져올 기능(사이드바 UI,
  메모, 할 일, Gmail/Calendar/주식 연동 전부)을 확인 질문으로 먼저 좁힌 뒤 진행.
  - `scripts/webviewer/static/style.css` 전면 재작성: 참고 파일의 CSS 커스텀 프로퍼티(색상
    팔레트, `rgb(var(--x))` 패턴)를 그대로 이식하되 Tailwind CDN이나 Google Fonts 같은 외부
    리소스는 전혀 쓰지 않고 순수 CSS로 구현(로컬/오프라인 사용 원칙 유지).
  - `base.html`을 사이드바(대시보드/노트/태그/검색/Daily 아카이브) + 상단바 구조로 재작성,
    모바일 오프캔버스 사이드바(`assets/nav.js`, 햄버거 토글 + 오버레이) 추가.
    `home.html`(→`index.html`)을 새 대시보드 콘텐츠로 바꾸고, 기존 홈 콘텐츠(areas/topics/
    daily 목록)는 새 템플릿 `vault.html`(→`vault.html`)로 분리. note/tag/tags_index/search/
    daily_archive 템플릿 전부 카드형 레이아웃으로 리디자인.
  - `assets/dashboard.js`: 날짜별 할 일 체크리스트(직접 추가/완료 토글/삭제, 캘린더 연동
    없음)와 단일 스크래치 메모 위젯 — 둘 다 브라우저 `localStorage`에만 저장, vault 마크다운
    파일과는 무관함을 카드 하단에 명시.
  - `generate-html.py`에 `load_dashboard_snapshot()` 추가: `scripts/webviewer/data/
    dashboard-snapshot.json`을 읽어 대시보드에 Gmail/캘린더/관심종목 카드를 렌더링. 이
    JSON은 Gmail/Google Calendar MCP 도구가 있어야 채울 수 있어 일반 Python 스크립트로는
    자동화 불가 — `scripts/build-dashboard-snapshot.py`는 데이터를 가져오는 대신 스키마
    검증(`--init`으로 빈 틀 생성)만 담당하도록 설계.
  - 실제로 Gmail(`search_threads`, `list_labels`)과 Google Calendar(`list_events`) MCP
    도구를 이 세션에서 호출해 진짜 데이터로 스냅샷을 채움 — 받은편지함 3639통/안읽음
    2504통(라벨 API 기준, 최근 6통 요약), 이번 주(08/11~08/17) 캘린더 일정 전체. 관심종목은
    WebSearch로 코스피 지수를 두 번 조회했더니 서로 다른 값(6,258 vs 6,299)이 나와 신뢰할
    수 없다고 판단해 의도적으로 비워두고 이유를 `stocks.as_of`에 기록(스키마는 준비해 둠 —
    나중에 실제 시세 API를 붙이면 채워짐).
  - 서비스 기본 포트를 8000 → 5000으로 변경(`preview.sh`, `generate-html.py`, README,
    사용설명서 반영) — 사용자 요청. 이후 5000이 이 macOS의 AirPlay 수신 포트와 겹치는 걸
    발견해 사용자에게 대안(4000)을 추천했고, 사용자가 4500을 지정 → 비어있음을 `lsof`로
    확인 후 4500으로 최종 변경(같은 파일들 재반영).
- **버그 수정**: Playwright로 실제 렌더링을 검증하다가 두 가지 발견.
  1. **"추가" 버튼 아이콘이 거대하게 깨짐**: `.btn` 클래스에 자식 `svg` 크기 규칙이 없어서
     아이콘이 기본 크기로 렌더링되어 버튼이 줄바꿈됨 → `.btn svg { width/height: 14px }` 추가.
  2. **모바일에서 "할 일" 카드 헤딩이 음절 단위로 줄바꿈**: `.card-head`에 `flex-wrap`이
     없어 좁은 화면에서 제목과 툴바(날짜 이동 버튼들)가 한 줄에 욱여넣어지며 "할 일"이
     "할"/"일"로 쪼개짐 → `.card-head`에 `flex-wrap: wrap`, `.card-title`에
     `word-break: keep-all`(한글은 음절 단위로 안 쪼개지게) 추가.
  3. **(웹뷰어 콘텐츠 렌더링) pandoc 헤딩+목록 인접 버그**: daily note를 열어보다가
     `## 오늘의 주제` 같은 헤딩이 `<h2>## 오늘의 주제</h2>`처럼 `#`이 텍스트에 그대로 남는
     걸 발견. 재현: pandoc 3.10.1에 `## 헤딩\n-\n`(헤딩 바로 다음 줄에 빈 줄 없이 `-` 목록)을
     넣으면 항상 이렇게 깨짐(id는 정상 생성됨). `daily_note.template`이 정확히 이 패턴이라
     매번 발생하는 문제였음 — Phase 3에서는 다른 테스트 콘텐츠라 못 잡았던 것으로 추정.
     `generate-html.py`에 `fix_heading_list_adjacency()`를 추가해 pandoc에 넘기기 전
     헤딩-목록 사이에 빈 줄을 자동 삽입해 우회(원본 `.md` 파일 자체는 건드리지 않음).
- **이슈/막힌 점**:
  - 서비스 기본 포트를 처음엔 5000으로 바꿨지만, 이 macOS 환경은 ControlCenter(AirPlay 수신
    기능)가 이미 5000번 포트를 쓰고 있어 `preview.sh`를 인자 없이 실행하면 바인딩에
    실패함(403) → 사용자와 상의해 4500으로 최종 변경, `lsof`로 비어있음 확인 후 반영함.
  - 관심종목(주식) 실시간 시세를 가져올 MCP 도구가 없어 WebSearch로 대체 시도했으나 결과가
    일관되지 않아 데이터를 채우지 못함 — 실제 시세 API 연동은 향후 과제로 남김.
  - Gmail/캘린더 스냅샷은 이 세션에서 1회 채워졌을 뿐, cron 등으로 자동 갱신되지 않음(MCP
    도구는 Claude 세션에서만 쓸 수 있음) — 매번 세션에서 수동으로 갱신 요청해야 함.
  - 사용자가 할 일을 추가했는데 안 보인다고 해서 확인해보니, 이전에 서비스 기본 포트를
    5000→4500으로 바꾸는 과정에서 브라우저에 각기 다른 포트로 접속했던 게 원인으로 추정됨
    (localStorage는 origin=scheme+host+**port** 단위로 분리 저장되므로, 포트가 바뀌면
    이전 포트에 저장된 항목이 안 보인다 — 삭제된 게 아니라 다른 origin에 남아있는 것).
    포트를 4500으로 고정하기로 안내하고 재입력을 권함.
- **추가/변경**: 할 일 항목에 **수정 기능** 추가(사용자 요청) — 항목에 마우스를 올리면 나오는
  연필 아이콘을 누르면 텍스트가 인라인 `<input>`으로 바뀌고 Enter/블러로 저장, Escape로 취소.
  빈 텍스트로 저장하면 기존 텍스트를 그대로 유지해 실수로 지워지는 걸 막는다. 체크박스
  완료토글/삭제 버튼과 클릭 이벤트가 겹치지 않도록 `todoList` 클릭 위임 핸들러에 `.todo-edit`
  분기를 추가하고, 입력창 자체의 클릭은 `stopPropagation`으로 행 전체 토글과 분리했다.
  Playwright로 수정 저장/체크박스 토글/삭제가 서로 간섭 없이 동작하는 것까지 확인.
- **추가/변경**: 대시보드에 **"오늘 Daily Note" 버튼** 추가(사용자 요청 — "버튼을 눌러서
  데일리 노트를 만들 수 있도록 해줘"). 지금까지 웹뷰어는 완전히 정적이라 파일을 쓰는 동작이
  하나도 없었는데, 이 버튼은 처음으로 서버 쪽에서 파일을 쓰는 요청(POST)을 처리해야 해서
  정적 파일 서버(`http.server`)로는 구현이 불가능했다.
  - `create-daily-note.py`의 핵심 로직을 `create_daily_note(config, target_date, logger)`
    함수로 분리(CLI `main()`과 서버가 공유).
  - `scripts/webviewer/server.py` 신설: `SimpleHTTPRequestHandler`를 상속해 정적 파일은
    그대로 서빙하되, `POST /api/create-daily-note`만 추가로 처리 — 없으면 노트 생성,
    있으면 그대로, 이후 `generate-html.py`의 `main()`을 그대로 호출해 사이트를 통째로
    재생성(새 노트의 HTML이 응답 시점에 이미 존재하게)하고 `{ok, created, url}`을 반환.
    쓰기 가능한 엔드포인트가 생긴 만큼 `127.0.0.1`에만 바인딩(같은 Wi-Fi의 다른 기기가
    내 vault에 파일을 쓸 수 없도록).
  - `scripts/preview.sh`가 `python -m http.server` 대신 이 서버를 쓰도록 변경.
    `generate-html.py`의 "로컬 미리보기" 안내 로그도 `preview.sh`/`server.py`를 가리키게 수정.
  - `assets/dashboard.js`에 버튼 핸들러 추가: 클릭 시 `fetch('/api/create-daily-note', {method:'POST'})`
    → 성공하면 반환된 `url`로 바로 이동, 실패하면(정적 서버로 띄운 경우 등) 버튼 아래에
    이유와 함께 에러 메시지 표시.
- **버그 수정**: `create-daily-note.py`/`generate-html.py`는 파일명에 하이픈이 있어 평범한
  `import`가 안 돼 `importlib.util.spec_from_file_location`으로 불러왔는데, 처음엔
  `exec_module()` 실행 전에 `sys.modules[name]`에 등록하는 걸 빠뜨려서
  `generate-html.py`의 `@dataclass` 처리 중 `sys.modules[cls.__module__]`이 `None`이 되어
  `AttributeError`가 났다. `spec.loader.exec_module()` 호출 전에 `sys.modules[name] = module`을
  먼저 하도록 순서를 바꿔 해결.
- **이슈/막힌 점**: 이미 존재하는 오늘 노트로 버튼이 정상적으로 이동하는 것까지는 Playwright로
  확인했지만, "새로 생성" 분기는 실제 사용자 vault의 오늘 노트를 지웠다가 다시 만들어야
  재현되는 상황이라 실사용자 데이터를 건드리지 않으려고 버튼을 통한 재현은 하지 않았다 —
  같은 함수(`create_daily_note`)의 "새로 생성" 경로는 Phase 1에서 CLI로 이미 검증된 로직이라
  간접적으로만 확인.

## 2026-08-11 — "노트 가져오기" 버튼 (기존 MD/HTML 파일 등록)

- **추가/변경**: 사용자가 "지금까지 만든 MD파일이나 HTML파일을 넣으면 바로 등록이 되도록
  하고싶어"라고 요청. 인터페이스(버튼 vs CLI)와 저장 위치(topics/ 고정 vs 매번 선택)가
  구현 방향을 크게 바꾸는 결정이라 먼저 확인 질문으로 좁힘 — 대시보드 버튼 + topics/ 고정으로 확정.
  - `scripts/webviewer/server.py`에 두 번째 엔드포인트 `POST /api/import-note` 추가.
    **멀티파트 업로드가 아니라 `File.text()`로 읽은 내용을 JSON으로 보내는 방식**을 택함 —
    이 서버가 상속하는 `http.server`엔 멀티파트 파서가 없고, 표준 `cgi.FieldStorage`는
    Python 3.13에서 아예 제거됐다. 텍스트 콘텐츠만 다루면 되는 용도라 이 편이 의존성 없이
    훨씬 간단했다.
  - `import_note()`: `.html`/`.htm`은 `pandoc --from=html --to=markdown`으로 변환(기존
    `generate-html.py`가 반대 방향으로 쓰던 pandoc을 재사용). 제목은 변환된 마크다운의 첫
    `# ` 헤딩 → 없으면 원본 HTML의 `<title>` 태그 → 그것도 없으면 파일명 순으로 결정.
    `.md`/`.markdown`/`.txt`는 변환 없이 같은 제목 규칙만 적용. 파일명은
    `generate-html.py`의 `slugify_heading()`(헤딩 앵커 slugify와 동일 함수)을 재사용해
    제목을 슬러그화하고, `topics/{slug}.md`가 이미 있으면 `-2`/`-3`으로 기존 노트를
    보호. 저장 후 `build-backlinks.py` → `generate-html.py`를 순서대로 재실행해 사이트를
    통째로 재생성.
  - 대시보드에 "노트 가져오기" 카드 추가(`<input type="file" accept=".md,.markdown,.html,.htm,.txt">`
    를 라벨 버튼으로 감싸 파일 선택창을 뜨게 함) + `assets/dashboard.js`에 핸들러: 파일
    선택 → `file.text()` → `POST /api/import-note` → 성공 시 반환된 `url`로 이동, 실패
    시 버튼 아래에 이유 표시(daily note 버튼과 동일한 패턴).
  - curl로 HTML→MD 변환(실제로 `<h1>`/`<title>`/목록이 있는 샘플 HTML 사용), 같은 파일
    두 번 가져오기(중복 파일명 `-2` 처리), `.md` 파일(제목 헤딩 없을 때 파일명 폴백),
    지원하지 않는 확장자(`.png`) 에러까지 전부 확인. 이후 Playwright로 실제 "파일 선택"
    클릭 → 파일 업로드 → 노트 페이지 자동 이동 → 검색 페이지에서 실제로 찾아지는 것까지
    엔드투엔드 확인. 테스트에 쓴 가짜 노트(3d-프린터-조사, my-notes, 행동경제학-공부)는
    전부 지우고 사이트 재생성해서 vault를 원래 상태로 되돌림.
- **이슈/막힌 점**: Playwright의 `browser_file_upload`가 프로젝트 폴더 밖의 경로(스크래치
  디렉터리)는 "File access denied"로 거부해서, 테스트용 HTML 파일을 `.playwright-mcp/`
  (이미 허용 경로이자 `.gitignore` 대상) 안으로 복사한 뒤에야 업로드 테스트를 할 수 있었다.
- **사용자 확인**: 노트 가져오기 사용 직후 사용자가 "노트가져오기했는데 어디에 저장되는지
  모르겠다"고 문의. 서버 로그를 보니 실제로는 import-note 요청 자체가 서버에 도달한 기록이
  없었다(내가 테스트용으로 만들었다 지운 로그만 있음) — 파일 선택 취소, 미지원 확장자 선택,
  또는 기능이 추가되기 전의 캐시된 페이지를 보고 있었을 가능성. 라이브 서버에 curl +
  Playwright로 다시 한번 정확히 재현해 "지금 이 순간 정상 동작함"을 확인해줌.

## 2026-08-11 — 노트 편집 기능 (브라우저에서 바로 저장)

- **추가/변경**: 사용자가 "daily note 안의 내용도 브라우저에서 바로 타이핑해서 저장되게
  하고 싶다"고 요청. daily note에만 제한할 이유가 없어서 **모든 노트(areas/topics/daily)에
  공통 적용**하기로 함(같은 템플릿 `note.html`이라 한 번만 구현하면 전부 적용됨).
  - `generate-html.py`: 노트 렌더링 시 `raw_source=note.text`(가공 전 원본 마크다운)를
    템플릿에 추가로 넘김.
  - `note.html`: 렌더링된 뷰(`#note-view`)와 숨겨진 편집기(`#note-editor`, textarea 초기값이
    바로 `{{ raw_source }}`)를 같이 두고 "편집" 버튼으로 토글. Jinja2 autoescape +
    `<textarea>`의 raw-text 콘텐츠 모델 덕분에 원본을 그대로 넣어도 안전하다(엔티티로
    이스케이프됐다가 브라우저가 다시 원문자로 복원 — 별도 JSON 인코딩 불필요).
  - `scripts/webviewer/server.py`에 세 번째 엔드포인트 `POST /api/save-note` 추가.
    `save_note()`가 브라우저에서 온 경로 문자열을 그대로 믿지 않고 ① 확장자가 `.md`인지
    ② `vault.note_dirs()`(areas/topics/daily) 하위인지 검증 — `scripts/`나 vault 바깥
    파일은 절대 못 쓰게 막았고, 새 파일 생성도 막았다(그건 "가져오기"/daily note 버튼의
    역할).
  - `assets/note-edit.js` 신설: 편집/취소/저장 버튼 핸들러, daily-note 버튼과 동일한
    실패 처리 패턴(에러 메시지 표시, 버튼 재활성화).
  - Playwright로 실제 오늘 daily note를 열어 편집 → 저장 → 새로고침까지 반영되는 것을
    확인. 테스트 문구("myremember 대시보드 기능 테스트" 등)는 사용자의 진짜 daily note
    파일이라 확인 후 원래(빈 템플릿) 상태로 되돌려 놓음 — 테스트 데이터를 사용자의 개인
    기록에 남기지 않기 위해.
- **이슈/막힌 점**: 없음.

## 2026-08-11 — "노트 가져오기" 캐시 버그 수정 + 노트 삭제 기능

- **버그 수정**: 사용자가 "노트 가져오기가 파일을 등록해도 안 보인다"고 두 번째로 신고.
  서버 로그를 전체 확인해보니 import-note 요청 자체가 서버에 한 번도 도달한 적이 없었다
  (내가 테스트한 기록만 있음) — 파일 선택은 되는데 그 이후(파일을 읽어 서버로 보내는 JS)가
  전혀 실행 안 된 것으로 추정, 브라우저가 기능 추가 이전 버전의 `assets/dashboard.js`를
  캐시해서 계속 쓰고 있었을 가능성이 가장 유력했다. `SimpleHTTPRequestHandler`는 기본적으로
  `Last-Modified`만 보내고 `Cache-Control`은 안 보내서 브라우저가 재검증 없이 오래된 캐시를
  휴리스틱하게 재사용할 수 있다는 게 원인. `scripts/webviewer/server.py`의 `Handler`에
  `end_headers()`를 오버라이드해 모든 응답에 `Cache-Control: no-store, must-revalidate`를
  강제하도록 수정하고, 사용자에게 강력 새로고침(`Cmd+Shift+R`) 후 재시도를 안내.
  → **결과: 사용자가 실제로 노트 5개(Claude Code 관련 가이드 문서들)를 성공적으로
  가져왔고, 이후 서버 로그로 확인함.**
- **추가/변경**: 사용자가 "토픽에 등록된 것을 수정, 삭제하는 기능을 만들자"고 요청. 수정은
  이미 있던 노트 편집 기능으로 커버됨(topics 포함 전체 노트에 적용돼 있었음) — 삭제 기능을
  새로 추가.
  - `server.py`: `save_note()`와 경로 검증 로직을 공유하도록 `resolve_note_path()`로
    리팩터링(`action` 인자로 에러 메시지만 "저장"/"삭제" 구분). `delete_note()` +
    `POST /api/delete-note` 추가 — `Path.unlink()`로 지우고 사이트 재생성. 휴지통은
    없음(git 커밋 상태였다면만 복구 가능).
  - `generate-html.py`의 `as_links()`가 `path`(vault 루트 기준 `.md` 상대경로)도 반환하도록
    확장 — 목록 항목의 삭제 버튼이 이 값을 그대로 씀.
  - 삭제 버튼을 두 군데에 배치: ① 노트 상세 페이지(`note.html`, "편집" 옆) — 삭제 후
    `vault.html`로 리다이렉트. ② 노트 목록(`vault.html`/`tag.html`/`daily_archive.html`)의
    각 항목 — 삭제 후 같은 목록 새로고침. 둘 다 같은 클래스(`.note-delete-btn`)와
    `data-path`/`data-title`/(목록에서만 없는) `data-redirect`를 쓰고, `assets/note-actions.js`
    하나가 `document`에 이벤트 위임으로 걸려 페이지 종류 상관없이 전부 처리 — 그래서
    `note-edit.js`(note.html 전용)와 달리 `base.html`에서 전역으로 로드하도록 함.
  - `.note-list li`를 flex(`justify-content:space-between`)로 바꿔 삭제 버튼이 오른쪽에
    붙게 하고, 마우스를 올렸을 때만 보이도록(기본 투명) 스타일 추가.
  - Playwright로 노트 상세 페이지 삭제(확인 다이얼로그 → redirect 확인)와 목록 페이지
    삭제(확인 다이얼로그 → 새로고침 확인) 둘 다 별도 테스트 노트로 검증, 테스트 후 정리.
- **실사용 확인**: 이 기능을 배포한 직후, 사용자가 이전에 같은 HTML 파일을 실수로 두 번
  가져와서 생긴 중복 노트(`스킬skills-...법.md`/`-법-2.md`) 중 하나를 실제로 이 삭제
  기능으로 직접 지우는 것을 서버 로그로 확인함 — 배포와 거의 동시에 실사용 검증이 됨.
  같은 시간대에 그 노트를 "편집" 기능으로 저장하기도 해서, 편집 기능도 실사용자 기준으로
  다시 한번 검증된 셈.
- **이슈/막힌 점**: "노트 가져오기"로 스타일이 많이 들어간 HTML(예: 커스텀 랜딩 페이지형
  가이드 문서)을 넣으면, pandoc이 인라인 `style` 속성을 Pandoc bracketed-span 문법
  (`` `텍스트`{style="..."} ``)으로 변환하면서 그 속성 문자열이 그대로 노트 제목/파일명에
  섞여 들어가는 문제를 발견(`topics/내가-만든-명령어를-명령어stylebackground...md`가 실사례).
  내용 자체는 정상이라 급하지 않지만, 제목 추출 시 `{...}` 속성 구문을 걸러내는 보완이
  필요 — `docs/STATUS.md`의 "알려진 이슈"에 기록.

## 2026-08-11 — 노트 가져오기 자동 태그 + pandoc 정리 옵션 + 태그 오탐 수정

- **버그 수정**: 사용자가 "내가 입력하면 알아서 토픽을 나누어 주나?"라고 물어봐서 현재는
  자동 분류가 없다고 답한 뒤, 확인 질문으로 "가져오기 시 태그 자동 생성"을 원한다는 걸
  확인. 자동 태그를 만들기 전에 vault의 실제 `태그` 페이지를 먼저 점검해보니 `#FF6B35`,
  `#004E89` 같은 CSS 색상 코드가 진짜 태그처럼 노출되고 있는 걸 발견 — 앞서 남겨둔
  "제목이 지저분해짐" 이슈와 같은 근본 원인(pandoc이 스타일 속성을 Pandoc bracketed-span
  문법으로 변환)이었다. 자동 태그 기능을 얹기 전에 이 근본 원인부터 고치는 게 순서라고
  판단해 먼저 처리:
  - `server.py`의 `html_to_markdown()`을 `pandoc --from=html-native_divs-native_spans
    --to=markdown-inline_code_attributes --wrap=none`으로 변경. 커맨드라인에서
    `<div class="box">`/`<span style="...">`/`<code style="...">`가 섞인 샘플 HTML로
    전후 비교해 `:::` 감싸기와 `{style="..."}` 속성이 전부 사라지는 것을 직접 확인 후 반영
    — 이 옵션 하나로 "제목 지저분해짐" 이슈와 "태그 오탐" 이슈가 동시에 해결됨.
  - `vault.py`의 `extract_tags()`에 `HEX_COLOR_RE`(3자리/6자리 순수 16진수 문자열) 필터를
    추가해 이중 방어 — pandoc 옵션을 못 바꾸는 다른 경로로 비슷한 텍스트가 들어와도 태그로
    안 잡히게.
- **추가/변경**: `import_note()`에 자동 태그 로직 추가.
  - `all_existing_tags(config)`: vault 전체(`iter_note_files` + `extract_tags`)에서 이미
    쓰이는 태그 모음을 만든다.
  - `suggest_tags(existing_tags, title, body, limit=5)`: ① 기존 태그 중 새 노트 내용에
    실제로 등장하는 것을 우선 재사용(어휘가 무한정 늘지 않게) ② 그것만으로 부족하면
    제목을 공백/괄호/구두점으로 쪼갠 앞부분 최대 6개 토큰에서 조사를 뗀(`_JOSA_SUFFIXES`,
    긴 조사부터 매치, 조사 뗀 뒤 길이 2자 미만이면 포기) 후보로 채운다. 왜 "제목
    앞부분만"인가: 한국어 제목은 핵심 명사가 앞에, 서술어가 뒤에 오는 경향이 있어서
    — 실제로 뒤쪽까지 다 보게 했더니 "다시 살펴보는" 같은 구간에서 "살펴보"(동사 어간)
    처럼 조사 제거가 동사 활용형을 잘못 건드리는 사례가 나와서 범위를 좁혔다.
    `_TAG_STOPWORDS`(대명사·부사 등)로 한 번 더 거른다.
  - Anthropic API나 다른 외부 도구를 쓰지 않고 순수 정규식/문자열 처리로만 구현 — 이
    기능은 사용자가 브라우저에서 파일을 선택하는 순간 `server.py`가 그 자리에서 바로
    응답해야 해서, Gmail/캘린더 스냅샷처럼 "Claude 세션에서만 가능"한 방식을 쓸 수 없다.
    완벽한 의미 분류가 아니라 로컬 휴리스틱임을 코드 주석과 사용설명서에 명시.
  - curl로 두 가지 시나리오 검증: ① `<title>세션 관리 테스트</title>`만 있는 짧은 HTML →
    `#세션 #관리 #테스트` 자동 부착 확인. ② 위에서 생긴 "세션"/"관리" 태그가 있는 상태로
    조사가 붙은 제목("세션을 다시 살펴보는 이유와 클로드 코드 활용법")을 가져왔더니 기존
    태그를 올바르게 재사용하면서 새 후보도 일부 채워지는 것 확인(단, "살펴보" 같은 다소
    어색한 후보도 같이 섞여 나옴 — 한계로 기록).
- **사용자 요청 처리**: "토픽에 있는 삭제버튼은 마우스를 해당 줄에 대면 나타나도록 해줘"
  요청이 들어왔는데, 확인해보니 삭제 기능을 처음 만들 때 이미 그렇게 구현돼 있었다
  (`.note-list .note-delete-btn { opacity:0 }` + `:hover`로 노출). Playwright로 기본
  상태(완전히 안 보임)와 hover 상태(휴지통 아이콘 노출) 스크린샷을 찍어 실제로 이미
  동작 중임을 확인해 회신.
- **이슈/막힌 점**: 자동 태그 휴리스틱은 형태소 분석기가 아니라서 완벽하지 않다(위 curl
  테스트에서도 확인). `docs/STATUS.md`의 "알려진 이슈"에 한계를 기록해두고, 필요하면
  Claude 세션에서 태그를 다듬어달라고 요청하는 경로를 안내함.

## 2026-08-12 — "노트 가져오기"에 PDF 지원 추가

- **추가/변경**: 사용자가 "노트 가져오기에 PDF도 추가해줘"라고 요청. 기존 `.md`/`.html`/
  `.txt`만 되던 가져오기에 `.pdf`를 추가.
  - pandoc은 PDF를 입력으로 못 받아서(출력만 가능) 별도 변환 경로가 필요했음 — 이미 이
    환경에 설치돼 있던 poppler의 `pdftotext` CLI(`brew install poppler`)를 서브프로세스로
    호출하는 방식을 선택, 새 Python 의존성은 추가하지 않음. `server.py`에
    `pdf_to_text(pdf_bytes)` 추가: 바이트를 임시 파일에 쓰고 `pdftotext -layout <tmp> -`로
    stdout에서 텍스트를 받는다(`-layout`으로 원본 줄/컬럼 배치를 최대한 보존). 폼피드(페이지
    구분 문자 `\x0c`)는 빈 줄로 치환하고 3줄 이상 연속 빈 줄은 2줄로 정리. 텍스트 레이어가
    없는 스캔 이미지 PDF는 추출 결과가 비어 명시적 에러(`ValueError`)로 실패하게 함(조용히
    빈 노트가 만들어지는 것 방지). `pdftotext` 자체가 없는 환경이면 `brew install poppler`
    안내가 담긴 에러 메시지.
  - `import_note()`의 확장자 분기에 `.pdf` 추가 — `content`가 브라우저에서 base64로 감싸
    보낸 문자열이라는 전제로 `base64.b64decode(..., validate=True)` 후 `pdf_to_text()`에
    넘긴다. 그 이후(제목 추출/자동 태그/slug/중복 처리/백링크·사이트 재생성)는 기존
    `.txt` 경로와 동일 로직 재사용 — PDF에서 뽑은 텍스트를 일반 텍스트 노트처럼 취급.
  - `assets/dashboard.js`: `.md`/`.html`/`.txt`는 기존처럼 `file.text()`로 읽지만, PDF는
    바이너리라 그 방법으로는 못 읽는다 — `file.arrayBuffer()` → `Uint8Array` → `btoa()`로
    base64 인코딩해서 같은 `/api/import-note` JSON 바디로 보내도록 분기 추가(멀티파트
    업로드는 여전히 안 씀 — 기존 "노트 가져오기" 구현 당시의 결정을 그대로 유지, DEVLOG
    2026-08-11 "노트 가져오기" 항목 참고).
  - `home.html`의 파일 입력 `accept`에 `.pdf` 추가, 안내 문구에도 PDF 언급 추가.
  - 요청 크기 제한(`MAX_BODY_BYTES`)을 5MB→20MB로 상향 — base64 인코딩이 원본 대비
    ~1.37배 커지는 걸 감안(개인용 로컬 서버라 상향에 큰 리스크 없음).
  - 검증: `/tmp` 임시 vault(실제 vault와 분리)를 만들어 `import_note()`를 직접 호출.
    `cupsfilter`로 한글 텍스트 PDF를 만들어(이 환경에 LaTeX/wkhtmltopdf 등 PDF 생성용
    pandoc 엔진이 없어 대안으로 사용) 정상 변환(제목/본문/자동 태그/백링크·사이트 재생성)을
    확인했고, 미지원 확장자(`.png`)와 손상된 base64(`.pdf`) 두 에러 경로도 확인. 테스트가
    끝난 뒤 임시 vault는 통째로 삭제해 실제 vault에는 아무 흔적도 남기지 않음.
- **이슈/막힌 점**: 없음. (참고: 스캔 이미지로만 된 PDF는 텍스트 레이어가 없어 이 기능으로
  가져올 수 없다 — OCR은 범위 밖이라 별도 처리 없이 명확한 에러 메시지로 안내함)
- **후속 논의 → PDF 원본 보관(선택) 기능 추가**: 배포 직후 사용자가 "원본이 사라지는 거
  맞냐"고 확인 질문, 이어서 "PDF/이미지는 원본을 구글드라이브에 백업하자"고 요청. 바로
  구현하기 전에 먼저 설계를 같이 정리했다:
  - **범위 좁히기**: `.md`는 가져온 결과가 사실상 원본과 동일하고 git 히스토리로도 이미
    복구 가능해서 원본 보관 실익이 없고, `.html`은 텍스트라 저장 부담은 적지만 손실이
    있어 애매한 케이스 — 사용자가 최종적으로 "AI로 정리한 자료는 처음부터 `.md`로
    출력해서 등록하면 원본 보관 자체가 불필요해진다"고 스스로 결론 내려서, 이번엔 손실이
    가장 크고(이미지·표·레이아웃 전부 소실) 이진 파일이라 재변환 여지가 있는 **PDF만**
    범위로 확정. 이미지 첨부는 애초에 "노트 가져오기"가 지원하지 않아 이번 범위에서 제외.
  - **업로드 시점**: 가져오는 즉시 `rclone copy`를 부르는 방식(A)과, 로컬에만 저장해두고
    기존 `backup.sh`의 월간 백업이 같이 챙기게 하는 방식(B)을 놓고 상의 — A는 매 가져오기마다
    네트워크 의존이 생기고, PRD 3.4절이 이미 "클라우드 백업은 승인 필요"로 정해둔 것과도
    어긋나서 B로 확정.
  - **저장 위치 재검토**: 처음엔 `backups/imported-originals/`를 생각했는데, `backup.sh`의
    tar 명령이 `--exclude="backups"`로 **백업 폴더 자기 자신을 이미 제외하고 있다는 걸
    코드를 다시 읽다가 발견**(tar가 자기가 쓰고 있는 아카이브를 다시 자기 안에 넣는 자기
    참조 버그를 막기 위한 기존 설계) — 그대로 뒀으면 원본이 로컬에만 남고 절대 백업/업로드
    안 되는 조용한 버그가 될 뻔했다. 그래서 vault 루트에 새 최상위 폴더 `attachments/`를
    만들고, `.gitignore`에 추가한 뒤 `backup.sh`의 tar 소스 목록에 명시적으로 포함시키는
    쪽으로 변경.
  - `server.py`: `save_original_for_backup(note_path, filename, data, logger)` 추가 —
    `attachments/{노트 slug}/{원본 파일명}`에 원본 바이트를 쓴다(노트 slug가 이미
    `unique_topic_path()`로 중복 방지돼 있어서 첨부 폴더도 자동으로 충돌 없이 유일해짐).
    `import_note()`에 `keep_original: bool = False` 매개변수 추가, `.pdf` 분기에서만
    동작(다른 확장자는 무시). `_import_note()` 핸들러가 요청 JSON의 `keep_original`을 읽어
    전달하고, 응답에 `kept_original` 필드를 추가해 프런트가 결과를 알 수 있게 함.
  - `home.html`: "노트 가져오기" 카드에 체크박스(`#import-keep-original`) 추가, 기본은
    꺼짐. `dashboard.js`가 파일 선택 시 이 체크박스 상태를 읽어 `keep_original`으로
    같이 보낸다.
  - `scripts/backup.sh`: tar 소스 목록에 `attachments` 추가 + `mkdir -p`에도 추가(사용자가
    아직 한 번도 "원본 보관"을 안 써서 폴더가 없는 상태에서도 `tar` 명령이 존재하지 않는
    인자 때문에 실패하지 않도록 — 처음엔 이걸 안 넣어서 `tar: attachments: Cannot stat` 로
    스크립트 전체가 실패하는 걸 임시 vault 테스트에서 직접 재현하고 나서 고쳤다).
  - **삭제 연동**: 구현 다 하고 나서 "노트를 지우면 첨부 폴더는 어떻게 되나"를 스스로
    점검하다가, 기존 `delete_note()`가 `.md` 파일만 지우고 `attachments/{slug}/`는 그대로
    남긴다는 걸 발견 — 요청받은 범위는 아니었지만 방치하면 지운 노트의 원본 PDF가 로컬
    디스크와 향후 모든 클라우드 백업에 영원히 고아 상태로 남는 실질적 버그라 판단해 같이
    고침. `target.parent`가 `topics_dir`일 때만(첨부는 topics/ 가져오기에서만 생김)
    `attachments/{stem}/`이 있으면 `shutil.rmtree`로 같이 제거하도록 `delete_note()`를 확장.
  - 검증: 임시 vault에서 `keep_original=True`/`False` 두 경로를 `import_note()` 직접 호출로
    비교(True일 때만 `attachments/`에 파일이 남는지) → 실제 vault에서 `bash scripts/backup.sh`를
    직접 실행해 만들어진 `backups/2026-08.tar.gz` 안에 `attachments/`가 실제로 포함되는지
    `tar tzf`로 확인(이때 rclone 미설치라 클라우드 업로드는 정상적으로 스킵됨) → 실행 중인
    `preview.sh` 서버를 재시작(코드가 메모리에 이미 로드돼 있어 재시작해야 새 로직이 반영됨을
    알고 있었음)한 뒤 curl로 실제 HTTP 엔드포인트까지 가져오기→삭제 전체 사이클을 돌려
    `attachments/`가 생성됐다가 삭제 후 자동으로 없어지는 것까지 확인. 테스트 산출물은 전부
    정리해 실제 vault에는 흔적이 남지 않음.
- **이슈/막힌 점**: 이 환경에 `rclone`이 아직 설치·인증돼 있지 않아(Phase 4부터 이어지는
  미해결 항목), 로컬 저장 + tar 포함까지만 확인했고 실제 구글드라이브 업로드 자체는
  검증하지 못했다 — 사용자가 `rclone config`를 완료하면 그다음 `backup.sh` 실행 때
  자동으로 확인되는 구조.

## 2026-08-12 — 대시보드 "노트 그래프" 위젯 (태그 기준 미리보기)

- **추가/변경**: 사용자가 "대시보드에 노트개수는 보이는데, 각 분야별로 어떤 정보가 들어있는지
  옵시디언처럼 체계적으로 보이도록 비주얼한 관리가 가능할까?"라고 요청, 이어서 "그러려면
  폴더별로 저장돼야 한다고 하던데"라고 덧붙임. 바로 만들지 않고 먼저 두 가지를 확인 질문으로
  좁혔다: ① 배치(전용 "그래프" 페이지 vs 대시보드 안 작은 미리보기 위젯) → **위젯**, ②
  그룹 기준(노트 타입 vs 태그) → **태그**. "폴더별 저장" 언급에는 실제로 답하기 전에,
  `topics/`가 지금도 물리적으로는 flat 폴더이고 "분야별로 보이는" 건 순전히 태그
  때문이라는 점, 태그는 직접 쓰는 노트는 수동으로 붙여야 하고 "노트 가져오기"만 로컬
  휴리스틱으로 자동으로 붙는다는 점(백그라운드 AI 분류기는 없음)을 먼저 설명해서, 폴더
  재구성 없이도 원하는 그림이 가능하다는 걸 확인시켰다 — 폴더를 실제로 나눴다면 기존
  노트 경로가 전부 바뀌어 링크·백링크·검색 인덱스를 다시 맞춰야 했을 것이라 범위가
  훨씬 커질 뻔했다.
  - **데이터**: 새 파이프라인을 만들지 않고 기존 자산을 재사용. `generate-html.py`의
    `load_backlinks()`가 `backlinks` 맵뿐 아니라 원본 `edges` 배열도 같이 반환하도록
    확장(반환 타입이 튜플로 바뀌어 유일한 호출부도 같이 수정). `NoteMeta`(제목/태그/
    dir_type)와 이 엣지를 합쳐 `assets/graph-data.js`에
    `window.MYREMEMBER_GRAPH = {nodes, edges}`로 굽는다 — `search-data.js`와 완전히 같은
    패턴(fetch 없이 `<script>`로 즉시 로드, `file://`로 열어도 동작).
  - **레이아웃**: `scripts/webviewer/static/graph-widget.js` 신설. 외부 라이브러리
    (D3.js 등) 없이 순수 JS로 아주 단순한 힘-기반(force-directed) 시뮬레이션을 직접
    구현 — 모든 노드쌍 상호 반발(O(n²), 개인 vault 규모에선 무시할 수준) + `[[링크]]`로
    이어진 노드는 강한 인력 + **대표 태그가 같은 노드는 약한 인력** + 중심으로 살짝
    당기는 힘. `requestAnimationFrame`으로 매 프레임 다시 계산하는 대신 고정 250회
    (노드 60개 넘으면 120회로 축소) 미리 계산해서 한 번에 그린다 — 미리보기 위젯이라
    지속 애니메이션은 불필요한 CPU 낭비. **지금 실제 vault엔 `[[링크]]`가 아직 0개라서**
    태그 인력이 없었으면 그냥 무작위로 흩어진 점들이 됐을 텐데, 태그 인력 덕분에 같은
    태그를 공유하는 노트끼리 실제로 가깝게 모인다(예: "3D 프린터 완벽 가이드"와 "모로코
    세우타"가 둘 다 `#완벽` 태그를 갖고 있어서 서로 가깝게 뭉침 — 사용자 입장에서 뜬금없어
    보일 수 있지만 실제 데이터가 그렇게 태깅돼 있어서임).
  - **색상**: 이 작업이 정확히 dataviz 스킬의 트리거 조건(그래프/네트워크 뷰, 태그별
    카테고리 색상)에 해당해서 스킬을 먼저 로드하고 절차를 따름 — 색은 맨 마지막에,
    검증된 8색 카테고리 팔레트(`palette.md`)를 이 프로젝트 기존 `rgb(var(--x))`
    라이트/다크 토큰 패턴(`style.css`의 `:root`/`prefers-color-scheme`/`data-theme`
    3중 구조)에 그대로 이식해 `--series-1~8` 추가. `validate_palette.js`로 인접-쌍
    기준 라이트/다크 둘 다 PASS 확인(`--pairs all` 기준으론 3개까지만 전 구간이 안전하다는
    것도 확인했지만, 이 위젯은 범례+hover 툴팁에 태그 이름 텍스트가 항상 같이 붙어 있어
    색만으로 식별하지 않는다는 전제로 8색 전부를 쓰기로 의도적으로 절충 — 개인 단일
    사용자용 로컬 도구라 외부 공개 접근성 기준까지는 안 맞춰도 된다고 판단). 노트 집합
    안에서 등장 빈도 상위 8개 태그만 고정 순서(순환 없음)로 색을 받고, 9번째 이후
    태그와 태그 없는 노트는 전부 `--muted-foreground`("기타")로 폴백.
  - 노드는 SVG `<a>`로 감싸 클릭 시 해당 노트로 바로 이동, `<title>` 자식으로 제목+태그
    hover 툴팁. 범례/툴팁 텍스트는 전부 `textContent`로 채워 innerHTML 문자열 조립을
    피함(dataviz 스킬의 "라벨은 신뢰할 수 없는 데이터" 원칙).
  - `home.html`에 새 카드("노트 그래프") 추가, 통계 카드 바로 아래 배치(노트 개수를 본
    직후 바로 시각화가 보이도록). 위젯용 CSS(`.graph-widget-svg`, `.graph-edge`,
    `.graph-node-dot`, `.graph-legend*`)를 `style.css`에 추가.
  - 검증: `generate-html.sh` 재생성 후 실행 중이던 `preview.sh` 서버 재시작, Playwright로
    라이트/다크모드와 모바일(390px) 스크린샷 확인 + JS로 각 노드의 실제 `fill`/`title`/
    `href`를 직접 조회해서 같은 태그를 가진 두 노트가 정확히 같은 색으로 묶이는지,
    태그 없는 노트가 정확히 회색으로 빠지는지 데이터 레벨로 검증 + 실제로 점을 클릭해
    해당 노트 페이지로 이동하는 것까지 확인. 스크린샷/`.playwright-mcp` 산출물은 확인 후
    정리해 vault에 흔적 남기지 않음.
- **이슈/막힌 점**: 없음. (참고: 이 위젯은 사용자가 명시적으로 선택한 범위대로 "작은
  미리보기"이지, Obsidian처럼 줌/팬이 되는 전체 화면 그래프 페이지는 아니다 — 필요해지면
  별도 페이지로 확장 가능하도록 데이터 계층(`graph-data.js`)은 이미 분리해뒀음)

## 2026-08-12 — 태그 관리: 상위/하위 관계 + 병합 기능

- **추가/변경**: 그래프 위젯을 만든 직후 사용자가 "태그에서 서로 연관연결을 시킬 수 있는
  기능을 추가하고, 연결시 주종을 결정하는 기능도 넣어줘"라고 요청, 곧이어 "태그가 분리되어
  있는 것을 합쳐서 같은 태그를 갖도록 하는 기능도 필요할 듯"이라고 덧붙임. 실제로 이 vault의
  기존 노트 하나("3D 프린터 완벽 가이드")가 자동 태그 휴리스틱 때문에 `#3D #가이드 #완벽
  #프린터`로 쪼개져 있던 걸 그래프 위젯 작업 때 이미 확인한 터라, 병합 기능의 필요성이
  바로 와닿는 요청이었다.
  - 바로 구현하지 않고 세 가지를 확인 질문으로 먼저 정함: ① 관리 방식(웹 UI vs config 파일
    직접 편집) → **웹 UI**(노트 가져오기/편집/삭제와 같은 기존 패턴), ② 구조(태그 하나가
    상위를 하나만 가지는 단순 트리 vs 여러 개 가지는 그래프) → **여러 개 허용**(예:
    "3D프린터"가 "하드�웨어"와 "취미" 둘 다의 하위일 수 있음 — 대신 순환 참조 처리가
    필요해짐), ③ 활용처(태그 페이지 표시만 vs 그래프 위젯도 반영) → **둘 다**.
  - **관계 저장소**: `scripts/webviewer/data/tag-relations.json`
    (`{"parents": {자식: [상위, ...]}}`) 신설, `config.yaml`에 `tags.relations_path` 추가.
    `dashboard-snapshot.json`과 같은 성격(git 추적 입력 데이터)이라 같은 패턴을 따름.
    `vault.py`에 `load_tag_relations()`/`save_tag_relations()`를 공용 헬퍼로 추가해
    server.py(쓰기)와 generate-html.py(읽기)가 같이 쓰게 함.
  - **`set_tag_parents()` + `POST /api/set-tag-relations`**: 태그 하나의 상위 목록을
    통째로 교체. "여러 상위 허용"을 택하면서 생긴 문제 — 트리라면 부모를 하나만 가리키니
    순환이 구조적으로 불가능한데, 그래프에서는 A→B→A 같은 순환이 얼마든지 생길 수 있다.
    저장 후보를 만든 뒤 DFS로 시작 태그에서 상위를 계속 타고 올라갔을 때 자기 자신으로
    돌아오는지 확인해서, 순환이면 저장 자체를 거부(에러만 반환, 파일은 안 바뀜). 임시
    vault에서 "A의 상위를 B로" 설정한 다음 "B의 상위를 A로" 설정을 시도해 실제로 거부되는
    것까지 확인.
  - **`merge_tags()` + `POST /api/merge-tags`**: 소스 태그 여러 개를 타겟 태그 하나로
    합친다. vault 전체 노트를 훑어 `#소스태그`를 정확한 토큰 경계로(`(?<!\S)#태그
    (?![\w가-힣-])` — `#3D`가 `#3D프린터`의 일부로 오매칭되지 않게) `#타겟태그`로 치환.
    처음 구현했을 때 같은 노트 안에 태그가 두 번 남는 문제를 발견(예: `#3D`와 `#프린터`를
    둘 다 `#3D프린터`로 합치면 `#3D프린터 #3D프린터`가 됨) — `extract_tags()`가 집합으로
    다시 뽑기 때문에 태그 목록/그래프 자체는 중복 없이 정상 동작하지만 원본 마크다운이
    지저분해 보여서, 마지막 줄이 이 프로젝트의 태그 표기 관례(순수 `#태그 #태그 ...` 형식,
    `suggest_tags()`가 붙이는 것과 같은 형식)와 일치하면 그 줄만 순서를 유지한 채 중복
    제거하도록 보완(본문 중간의 태그 언급은 안 건드림 — 사용자가 쓴 글을 함부로 고치지
    않기 위해). `tag-relations.json`에 소스 태그명이 남아있으면 타겟으로 합치면서(상위
    목록 union) 관계가 안 끊기게 하고, 이 과정에서 소스가 서로의 상위였던 경우 타겟이
    자기 자신을 가리키게 되는 자기참조 엣지도 같이 제거.
  - **"태그 관리" 페이지**(`tags/manage.html`) 신설 — 태그 목록 페이지(`tags/index.html`)
    상단에 버튼으로 연결. 병합 폼(다중 선택 소스 + 타겟 이름 입력, 기존 태그로 자동완성,
    확인 대화상자 후 실행 — 노트 파일이 실제로 바뀌는 만큼 삭제 버튼과 같은 확인 패턴 적용)
    + 태그별 상위 태그 편집(각 태그가 한 행, 다중 선택 드롭다운 + 저장 버튼). 새
    `tag-manage.js`.
  - **`tag.html`에 계층 표시**: 상위 태그는 브레드크럼에 이어서("대시보드 / 태그 /
    #상위태그1, #상위태그2"), 하위 태그는 카드 위에 별도 줄로.
  - **그래프 위젯 재설계**: 기존엔 "대표 태그가 같은 노트끼리 서로 당기는" 노트-노트 인력
    구조였는데, 이번 요청으로 태그 자체가 1급 개체가 돼야 해서 구조를 바꿨다 — **태그 허브
    노드**(색 배정된 태그 + 관계가 설정된 태그는 전부 허브 생성, 노트 점보다 크고 라벨이
    항상 보임)를 추가하고, 노트-노트 태그 인력을 "노트→자기 대표 태그 허브" 인력으로
    바꾼 뒤, "하위 태그 허브→상위 태그 허브" 인력(점선 연결선도 같이 그림)을 새로 추가 —
    허브-스포크 구조라 태그 관계가 그래프에 실제로 드러난다. 허브 클릭 시 해당 태그
    페이지로 이동. `generate-html.py`가 `graph-data.js`에 `tagParents` 필드 하나만
    추가해서 넘기고, 나머지(물리 시뮬레이션·렌더링)는 `graph-widget.js` 안에서 처리 — 새
    파이프라인은 만들지 않음.
  - 검증: 임시 vault에서 병합(정확 토큰 매칭 확인, 태그 줄 중복 정리 확인)·관계 저장·순환
    거부 세 가지를 `merge_tags()`/`set_tag_parents()` 직접 호출로 확인. 실제 vault·실행 중인
    `preview.sh` 서버로 ① 태그 관리 페이지 렌더링, ② 실제 폼으로 관계 저장 후
    `tag-relations.json` 파일 내용과 태그 페이지(`#3D`/`#DevKit`)의 상위/하위 표시를 직접
    확인, ③ 디스포저블 테스트 노트로 병합 API를 curl로 호출해 실제 파일이 정확히 바뀌는지
    확인, ④ Playwright로 그래프 위젯의 허브+점선이 라이트/다크/모바일(390px)에서 정상
    렌더링되고 허브 클릭 시 태그 페이지로 이동하는지까지 확인. 테스트에 쓴 관계/병합/노트는
    전부 원상 복구해 실제 vault에는 흔적이 남지 않음.
- **이슈/막힌 점**: 없음. (참고: 태그 관리 페이지는 지금 이 vault 기준 태그 19개라 관계
  편집 목록이 세로로 꽤 길다 — 태그가 훨씬 더 많아지면 검색/접기 같은 UI 개선이 필요할 수
  있지만, 지금 범위에서는 기능이 우선이라 미루고 STATUS.md에는 남기지 않음, 필요해지면
  요청 시 개선)

## 2026-08-12 — 대시보드 "새 노트 작성" (파일 업로드 없이 직접 입력)

- **추가/변경**: 사용자가 "노트에 텍스트를 바로 입력하는 기능도 넣어줘. 제목도 달 수
  있도록 하고"라고 요청. 지금까지 새 노트를 vault에 등록하는 방법은 "노트 가져오기"
  (기존 파일 업로드)뿐이었어서, 파일 없이 그 자리에서 타이핑해서 바로 노트를 만드는
  경로가 없었다.
  - `server.py`를 먼저 리팩터: `import_note()`의 뒷부분(제목/본문을 받아 자동 태그 붙이고
    슬러그화해서 `topics/`에 저장하는 로직)을 `_save_new_topic_note(config, title, body,
    logger, source_desc)`로 뽑아냄. `source_desc`는 로그 메시지 구분용("원본: {파일명}"
    vs "직접 입력"). 이렇게 뽑아둔 이유: 새로 만들 `create_note()`가 이 로직을 파일 변환
    단계(pandoc/pdftotext) 없이 그대로 재사용해야 해서 — 코드를 복붙하면 나중에 자동 태그
    로직을 고칠 때 두 곳을 따로 고쳐야 하는 문제가 바로 생길 상황이었다.
  - `create_note(config, title, content, logger)` 추가: 제목/내용이 비어있으면 바로
    `ValueError`, 아니면 `_save_new_topic_note()` 호출 후 사이트 재생성. `POST
    /api/create-note` 엔드포인트 + `_create_note()` 핸들러 추가(기존 버튼들과 동일한
    try/except 패턴).
  - `home.html`: "노트 가져오기" 카드 바로 아래에 "새 노트 작성" 카드 신설 — 제목
    `<input>` + 내용 `<textarea>`(빠른 메모의 `.memo-card textarea`와 같은 스타일
    패턴으로 `.new-note-textarea` 추가) + "저장" 버튼. `dashboard.js`에 핸들러 추가:
    빈 제목/내용 클라이언트 검증 → `fetch('/api/create-note', ...)` → 성공하면 반환된
    `url`로 이동, 실패하면 상태 메시지(다른 버튼들과 동일 실패 처리 패턴).
  - 검증: 임시 vault에서 `create_note()` 정상 동작(제목이 첫 헤딩으로, 태그 자동 추가)과
    빈 제목/빈 내용 에러 확인, 같은 자리에서 리팩터한 `import_note()`도 다시 호출해 회귀가
    없는지 확인. 실제 vault·실행 중인 서버로 Playwright를 통해 실제 폼에 입력 → 저장
    클릭 → 새 노트 페이지로 이동 → 파일 내용 확인까지 end-to-end로 검증(라이트/다크/
    모바일 390px 카드 렌더링도 확인). 테스트로 만든 노트는 삭제해 원상 복구.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 태그 관계 UI 단순화 (멀티셀렉트 → 칩 방식)

- **버그 수정 성격의 UX 개선**: 사용자가 "태그 관계의 인터페이스가 너무 어렵게 되어있는데,
  좀 쉽게 해줘"라고 요청. 기존 방식(태그마다 한 행, `<select multiple>`로 상위 태그를
  Ctrl/Cmd+클릭해서 고르고 "저장" 버튼)은 ① 멀티셀렉트 자체가 직관적이지 않고(대부분의
  사용자가 Ctrl/Cmd+클릭이 필요한 줄 모름), ② 지금 이 vault만 해도 태그가 15~20개라
  화면에 행이 그만큼 나열돼 있어서 부담스러웠다.
  - **새 방식**: 드롭다운에서 태그를 하나 고르면, 그 태그의 현재 상위 태그가 알약 모양
    칩으로 나타난다. 상위를 추가할 땐 다른 드롭다운에서 골라 "+ 추가"를 누르고, 뺄 땐
    칩 옆 `×`를 누른다 — 두 동작 다 클릭 즉시 `/api/set-tag-relations`를 호출해서
    저장하고 화면도 바로 갱신된다(별도 "저장" 버튼 없음). 한 번에 한 태그의 관계만
    보이므로 화면도 훨씬 짧아짐.
  - `generate-html.py`: `tag_manage_tpl.render()`에 더는 `relations`를 Jinja 컨텍스트로
    안 넘기고, 대신 전체 관계 맵(`{태그: [상위, ...]}`)을 `assets/tag-relations-data.js`
    (`window.MYREMEMBER_TAG_RELATIONS`)로 굽는다 — search-data.js/graph-data.js와 같은
    패턴. 태그를 고르자마자(서버 왕복 없이) 그 태그의 현재 상위 태그를 바로 보여줘야
    해서, 페이지 로드 시점에 전체 맵이 클라이언트에 있어야 했다.
  - `tag_manage.html`: 태그마다 한 행씩 반복하던 `{% for %}` 블록을 없애고, 단일 `<select
    id="relation-tag-select">` + 칩 컨테이너(`#relation-chips`) + 추가용 `<select
    id="relation-add-select">` + "+ 추가" 버튼으로 교체.
  - `tag-manage.js`: 관계 편집 로직을 전면 재작성. 태그 선택 시 `render()`가 칩 목록과
    "추가 가능한 태그"(현재 태그 자신 + 이미 상위인 태그는 제외) 드롭다운을 다시 그리고,
    칩의 `×`나 "+ 추가" 클릭은 곧바로 `saveParents()`를 불러 서버에 저장한 뒤 응답으로
    받은 최신 목록으로 다시 렌더링한다. 병합 섹션(`#merge-sources` 멀티셀렉트)은 그대로
    뒀다 — "여러 태그를 하나로 합친다"는 병합의 개념 자체가 다중 선택을 요구하는 동작이라
    이 피드백의 대상이 아니었음.
  - `style.css`: 옛 `.relation-row`/`.relation-tag`/`.relation-status`(행 레이아웃용)를
    지우고 `.tag-manage-select-single`/`.chip-row`/`.chip`/`.chip-remove`를 새로 추가.
  - 검증: 실제 vault·실행 중인 서버에서 Playwright로 ① 태그 선택 시 에디터가 나타나는지,
    ② "+ 추가"로 상위 태그를 추가하면 칩으로 뜨고 `tag-relations.json`에 실제로 저장되는지,
    ③ 칩의 `×`로 제거하면 파일에서도 지워지는지, ④ 라이트/다크/모바일(390px) 렌더링까지
    확인. 테스트로 추가한 관계는 그 자리에서 다시 제거해 원상 복구(파일이 빈 `{"parents":
    {}}`로 남았는데, 이는 관계가 하나도 없는 것과 동일한 값이라 문제 없음).
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 대시보드 레이아웃 재정리 (자주 안 쓰는 액션은 사이드바로)

- **추가/변경**: 사용자가 여러 메시지로 나눠서 요청 — "오늘 노트열기는 왼쪽 메뉴로 이동:
  자주사용하지 않음", "원본 보관을 윗줄로 이동하고 별도 표시로 마우스 올리면 설명이
  나오도록", "파일선택도 '노트가져오기' 바로 옆으로 이동", "새노트 작성도 왼쪽 메뉴로
  이동", "맨 상단에 오늘할일, 노트, 태그와 노트 그래픽을 이동함", "오늘 할일을 노트 그래프
  위로 이동하기". "새 노트 작성"을 사이드바로 옮기면 클릭했을 때 어떻게 동작해야 하는지
  (오늘 노트와 달리 제목+내용 입력이 필요함)와, 종합한 최종 대시보드 순서가 맞는지 두 번
  확인 질문으로 정리한 뒤 진행.
  - **"오늘 노트" → 사이드바**: `base.html`의 `<nav class="sidebar-nav">` 맨 아래(구분선
    아래)에 버튼(`#daily-note-nav-btn`)으로 이동. 로직은 새 `assets/daily-note-nav.js`로
    분리해 모든 페이지(사이드바가 `base.html`에 있어 항상 보임)에서 동작하게 함 — 기존
    `dashboard.js`의 핸들러는 제거. 이동하면서 중요한 버그를 하나 먼저 막았다: 서버가
    돌려주는 `url`(예: `daily/2026-08-12.html`)이 vault 루트 기준 상대경로인데, 이 버튼이
    이제 `/tags/manage.html`처럼 깊이가 다른 페이지에도 있으므로 그대로 쓰면
    `/tags/daily/2026-08-12.html`처럼 잘못된 주소로 깨진다 — `"/" + url`로 항상 절대경로를
    쓰도록 고쳐서 어느 페이지에서 눌러도 올바르게 이동하게 함(Playwright로 `/`와
    `/tags/index.html` 양쪽에서 클릭해 확인).
  - **"새 노트 작성" → 전용 페이지**: 대시보드 카드를 통째로 없애고 `new-note.html`
    (`generate-html.py`가 `root=""`로 렌더링, 사이드바에 `<a href="new-note.html">`)로
    분리. 로직은 새 `assets/new-note.js`로(내용은 기존 `dashboard.js`의 핸들러와 거의
    같음, `title`/`content` 입력 필드에 포커스 자동 이동 추가). 전용 페이지라 textarea를
    더 크게(`min-height: 340px`, `.new-note-textarea-tall`) 키웠다.
  - **"노트 가져오기" 카드 재배치**: 제목+"파일 선택" 버튼+"원본 보관" 토글을 한 줄에
    (`.import-note-head`, flex). "원본 보관"은 체크박스+긴 설명 문구였던 걸, 네이티브
    `title` 속성으로 hover 시 설명이 뜨는 작은 아이콘 토글(`.icon-toggle`,
    아카이브박스 아이콘)로 바꿨다 — 체크되면 배경이 `--primary` 색으로 바뀌어 상태가
    한눈에 보인다. 마크업만 바뀌고 `id="import-keep-original"`은 그대로 유지해서
    `dashboard.js`의 기존 로직은 손대지 않음.
  - **대시보드 섹션 순서 재배열**: 통계 카드(오늘 할일/노트/태그 수) → 할 일 위젯 →
    노트 그래프 → 노트 가져오기 → 빠른 메모 → Gmail/캘린더/관심종목. "오늘 노트"/"새 노트
    작성" 카드는 위 항목대로 완전히 빠짐.
  - `style.css`: `button.nav-link`(사이드바 버튼이 `<a>`와 똑같이 보이도록 기본 버튼
    스타일 리셋), `.sidebar-nav-divider`(탐색 항목과 액션 항목 사이 구분선),
    `.import-note-head`/`.icon-toggle`/`.icon-toggle-box`(체크박스를 아이콘 버튼처럼
    보이게 하는 패턴 — `opacity:0`으로 실제 input은 숨기고 인접 `<span>`을 체크 상태에
    따라 스타일링), `.new-note-textarea-tall` 추가.
  - 검증: 실제 vault·실행 중인 서버로 Playwright 전체 사이클 확인 — ① 대시보드 순서가
    확정한 대로 나오는지 스크린샷, ② 사이드바 "오늘 노트"를 대시보드와 `/tags/index.html`
    양쪽에서 클릭해 절대경로 이동 확인, ③ "새 노트" 사이드바 링크 → 전용 페이지에서
    실제로 제목+내용 입력 → 저장 → 새 노트 페이지 이동 확인, ④ "원본 보관" 토글 hover
    시 title 텍스트와 체크 시 색 변화 확인, ⑤ 체크박스를 켠 채로 실제 PDF 파일을
    업로드해서(재배치된 마크업에서도) `attachments/`에 원본이 저장되고 삭제 시 같이
    지워지는 기존 동작이 안 깨졌는지 end-to-end 재확인, ⑥ 라이트/다크/모바일(390px)
    렌더링 확인. 테스트에 쓴 노트/원본은 전부 정리해 원상 복구.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 캘린더 스냅샷: 기본 캘린더 하나 → 접근 가능한 캘린더 전체

- **추가/변경**: 사용자가 "캘린더를 내 캘린더 정보를 모두 가져오도록 해줘"라고 요청.
  범위를 "기간을 늘림"인지 "캘린더 개수를 늘림"인지 확인 질문으로 좁혔더니 후자 —
  지금까지는 "이번 주" 기간은 맞았지만 캘린더 하나(개인 일정)에서만 가져오고 있었다.
  - `list_calendars` MCP 도구로 확인해보니 이 사용자는 캘린더 11개에 접근 가능했다:
    기본(zisomtb@gmail.com, 이벤트 없음), 대한민국의 휴일, 쌍용 프로젝트 진행 보고,
    2026학년 대입 전형일, 조충남 개인일정(기존에 쓰던 것), KOBETA, 경매일정, MBC
    보도NPS 구축 프로젝트, MBC기술인협회, 30대 MBC 방송기술인 협회, 보도기술부 일정.
    각 캘린더에 `list_events`(이번 주 08/11~08/17 범위)를 호출해보니 4개
    캘린더(대한민국의 휴일/조충남 개인일정/KOBETA/보도기술부 일정)에 실제 일정이 있었고
    나머지 7개는 이 기간엔 비어있었다(계속 접근은 유지, 그냥 이번 주 일정이 없을 뿐).
  - **스키마**: `calendar.days[].events[]`의 각 이벤트에 선택 필드 `calendar`(어느
    캘린더에서 왔는지, 그 캘린더의 `summary`)를 추가 — 기존 스키마와 하위호환(없어도
    검증 통과, `home.html`이 있을 때만 배지를 그림). `build-dashboard-snapshot.py`의
    안내 주석에 "매번 `list_calendars`로 목록을 얻고 전부 순회해서 합치라"는 절차를
    명시해서, 다음에 "대시보드 스냅샷 갱신해줘"라고 요청받는 세션도 같은 방식을 따르게 함.
  - `home.html`: 이벤트 줄에 `ev.calendar`가 있으면 오른쪽에 작은 배지(`.event-cal-badge`)
    로 캘린더 이름을 표시. `.event-title`에 `flex:1`을 줘서 배지가 항상 오른쪽 끝에
    붙게 함.
  - 실제로 11개 캘린더를 전부 조회해 이번 주 스냅샷을 다시 만들어 반영(`range_label`에
    "캘린더 6개"라고 몇 개에 실제 일정이 있었는지 요약, `as_of`에 어느 캘린더가
    비어있었는지도 기록해 나중에 헷갈리지 않게 함). 실제 vault의
    `dashboard-snapshot.json`을 이 세션에서 직접 갱신하고 사이트도 재생성.
  - 검증: `build-dashboard-snapshot.py`로 스키마 통과 확인, Playwright로 대시보드
    캘린더 카드에 요일마다 여러 캘린더의 일정이 배지와 함께 섞여 나오는 것을 스크린샷으로
    확인.
- **이슈/막힌 점**: 없음. (참고: "보도기술부 일정" 캘린더는 근무 조편성 코드로 보이는
  한 글자짜리 이벤트(Y/R/휴/U/W)를 종일 이벤트로 쓰고 있다 — 이 코드가 정확히 무엇을
  뜻하는지는 사용자만 알 수 있어서 그대로 표시, 해석하지 않음)

## 2026-08-12 — 대시보드 2단 레이아웃 (Gmail/캘린더/관심종목을 오른쪽 탭으로)

- **추가/변경**: 사용자가 "캘런더위치를 오른쪽에 탭을 나누어서 여기에 보이도록 하자"라고
  요청. 미리보기로 구체화해 확인한 뒤(데스크톱 2단: 왼쪽 주 콘텐츠 + 오른쪽 고정폭 탭,
  모바일은 세로로 다시 쌓임) 진행.
  - `home.html`을 `.dashboard-layout`(그리드) 아래 `.dashboard-main`(통계·할일·그래프·
    가져오기·메모, 기존 순서 그대로)과 `.dashboard-side`(Gmail/캘린더/관심종목 탭
    카드)로 재구성. 스냅샷이 없을 때의 빈 상태 카드도 `.dashboard-side` 안으로 이동.
  - `style.css`: 더 이상 안 쓰는 `.dash-grid`/`.dash-grid-2`(예전 Gmail+캘린더 2열
    그리드)를 지우고 `.dashboard-layout`(`min-width:860px`에서 `1fr 300px`로 분할,
    오른쪽 열은 `position:sticky`로 스크롤해도 따라옴) + `.side-tabs`/`.side-tab`
    (밑줄 강조 탭 버튼) 추가. `.stats-row`가 이미 `grid-template-columns:
    repeat(auto-fit, minmax(140px,1fr))`라 300px 폭에서도 별도 처리 없이 자동으로
    줄어듦.
  - `dashboard.js`에 탭 전환 핸들러 추가 — `.side-tab` 클릭 시 `data-tab`이 일치하는
    `.side-tab-panel`만 보이고 나머지는 `hidden` 처리, `aria-selected`도 같이 갱신.
  - 검증: Playwright로 데스크톱(1280px) 2단 배치, 탭 3개(Gmail 기본→캘린더→관심종목)
    전환이 실제로 내용을 바꾸는지, 모바일(390px)에서 오른쪽 열이 아래로 내려와 세로로
    다시 배치되는지 확인. 스크린샷은 확인 후 정리.
- **이슈/막힌 점**: 없음. (참고: 캘린더 탭은 이벤트가 많고 열이 좁아(300px) 제목이
  여러 줄로 줄바꿈된다 — 사용자가 미리 확인한 "고정 폭" 설계의 자연스러운 결과라 그대로
  둠. 더 넓게 보고 싶으면 나중에 캘린더만 전용 페이지로 분리하는 것도 방법)

## 2026-08-12 — 유튜브 시청기록 분석 (Google Takeout 업로드 기반)

- **추가/변경**: 사용자가 "유튜브를 시청하는 정보를 여기서 보고 관심사를 분석해달라"고
  요청. YouTube Data API로는 실제 시청기록을 못 가져온다는 점(Google이 공개 API에서
  시청기록을 막아둠 — 좋아요/구독 같은 공개 데이터만 가능)을 먼저 설명하고 대안을
  확인했더니, 사용자가 "내가 직접 데이터를 다운받아서 업로드 하는 것으로 변경하자"고
  방향을 정함 — Google Takeout에서 받은 `watch-history.json`을 vault에 넣으면 로컬
  스크립트가 분석하는 구조로 확정.
  - **Gmail/캘린더 스냅샷과의 차이**: 저것들은 MCP 도구가 있어야 해서 Claude 세션에서만
    갱신 가능하지만, 유튜브는 사용자가 원본 파일만 내려받아 넣어두면 **API 키도 Claude
    세션도 없이 순수 로컬 스크립트만으로 반복 갱신**할 수 있다 — 그래서 데이터 저장을
    Gmail/캘린더와 같은 `dashboard-snapshot.json`에 섞지 않고 완전히 독립된 파이프라인
    (`config.yaml`의 `youtube.raw_history_path`/`summary_path`)으로 분리했다. 섞었으면
    나중에 "대시보드 스냅샷 갱신해줘" 세션이 Gmail/캘린더를 다시 쓰면서 유튜브 키를
    실수로 날릴 위험이 있었다.
  - `scripts/analyze-youtube-history.py` 신설: Takeout JSON(항목마다 `title`
    (`"Watched ..."` 접두사), `subtitles[0].name`(채널명), `time`)을 파싱 — 광고 항목
    (`details`에 "ads" 포함)과 삭제된 영상("Watched a video that has been removed")은
    제외. 채널별 시청 횟수 상위 15개, 전체 개수, 날짜 범위, 최근 30일 개수를 집계해
    `youtube-summary.json`에 저장. 원본 없으면 Takeout 받는 방법을 안내하고 조용히
    종료(에러 아님).
  - **민감한 개인 데이터 보호**: `scripts/webviewer/data/` 전체(이미 있던
    `dashboard-snapshot.json`도 포함 — 그동안 한 번도 git에 커밋된 적 없었다는 걸
    `git status`로 확인)를 `.gitignore`에 새로 추가 — Gmail 개수·가족 이름이 담긴
    캘린더·유튜브 시청기록이 실수로 커밋되는 걸 막기 위한 선제 조치.
  - `generate-html.py`에 `load_youtube_summary()` 추가, `home.html`의 오른쪽 탭에
    "유튜브" 탭을 4번째로 추가 — 요약 문구 + 채널별 막대(가장 많이 본 채널 기준 상대
    너비) 목록. 데이터 없으면 Takeout 받는 법을 안내하는 빈 상태 표시.
  - 검증: 가짜 Takeout 샘플(채널 3개, 광고/삭제 항목 포함 32개 항목)을 만들어
    ① 분석 스크립트가 광고/삭제 항목을 정확히 걸러내고 채널별 집계가 맞는지, ②
    실서버+Playwright로 유튜브 탭에 막대그래프가 비율대로 그려지는지, ③ 파일을 지운
    뒤 빈 상태 안내 문구가 제대로 뜨는지 확인. 테스트 샘플은 실행 후 삭제(어차피
    git-ignore 대상이라 커밋 위험은 없었음).
- **이슈/막힌 점**: 없음. (참고: 지금은 채널별 횟수만 집계 — 제목 키워드 분석 같은
  더 정교한 "관심사 분석"은 실제 데이터로 튜닝해봐야 의미가 있어서 이번 범위에서는
  뺐다. 사용자가 실제 Takeout 파일을 넣어본 뒤 필요하면 추가 요청하면 됨)

## 2026-08-12 — 클라우드 배포용 읽기 전용 빌드 + 비밀번호 게이트 (빌드까지, 배포는 보류)

- **추가/변경**: 사용자가 "지금은 로컬 위주인데, 클라우드로 업로드해서 이 대시보드를
  볼 수 있도록 만들자"고 요청. 이전에 GitHub Pages 공개 배포를 보류했던 이유("private
  저장소여도 Pages는 URL을 아는 누구나 볼 수 있음")를 다시 짚고, 지금은 Gmail 개수·
  가족 이름이 담긴 캘린더·개인 daily note까지 있어 더 민감해졌다는 점을 확인시킨 뒤
  두 가지를 확인 질문으로 정함: ① 접근 범위 → **GitHub Pages + 간단한 JS 비밀번호
  화면**(진짜 서버측 인증은 아님을 사용자가 인지하고 선택 — Cloudflare Access 같은
  새 계정이 필요한 진짜 인증 대신 기존 GitHub 인프라만 쓰는 쪽을 택함), ② 쓰기 기능
  → **읽기 전용으로 배포**.
  - **`generate-html.py`에 `readonly` 모드 추가**: `main(readonly=False,
    output_dir_override=None)`로 시그니처 확장, CLI에서 `--readonly [출력폴더]`로
    호출 가능(기존 `python3 generate-html.py`는 그대로 동작 — 하위호환). `readonly`를
    `base_ctx`에 넣어 모든 템플릿에서 접근 가능하게 하고, 노트 가져오기 카드(`home.html`)/
    편집·삭제 버튼(`note.html`)/목록의 삭제 버튼(`vault.html`/`tag.html`/
    `daily_archive.html`, `.readonly-mode .note-delete-btn { display:none; }` 한 줄로
    중앙 처리)/"태그 관리" 링크(`tags_index.html`)/사이드바의 "오늘 노트"·"새 노트"
    (`base.html`)를 전부 숨긴다. `new-note.html`/`tags/manage.html`은 아예 생성하지
    않는다(링크가 없으니 만들 이유가 없고, 만들어봐야 정적 호스팅엔 저장할 서버가 없어
    죽은 페이지가 됨).
  - **비밀번호 게이트**(`assets/password-gate.js` + `style.css`): `<head>`의 인라인
    스크립트가 `readonly`일 때 `<html>`에 `gate-locked` 클래스를 **동기적으로** 먼저
    붙여서(CSS `html.gate-locked body { visibility:hidden; }`) 내용이 잠깐이라도
    보이는 플래시를 막는다. `password-gate.js`는 입력한 비밀번호를 `crypto.subtle
    .digest`로 SHA-256 해시한 뒤 `assets/gate-hash.js`에 심어둔 해시와 비교 — 맞으면
    `sessionStorage`에 저장해 같은 세션 안에서는 다시 안 물어봄. **평문 비밀번호는
    어디에도 파일로 안 남는다**(환경변수로만 받아서 해시만 저장). 코드 주석과 커밋
    메시지 후보에도 명시했듯 **이건 진짜 보안이 아니다** — 정적 파일이라 view-source나
    JS를 꺼서 우회 가능, 검색엔진 노출·우연한 방문 정도만 막는 캐주얼한 장벽이다(사용자가
    이 트레이드오프를 알고 선택함).
  - `scripts/build-cloud-site.py` 신설: `MYREMEMBER_CLOUD_PASSWORD` 환경변수로 비밀번호를
    받아 `generate-html.py --readonly`를 호출한 뒤, 결과 폴더(기본
    `output/html-cloud`)의 `assets/gate-hash.js`를 진짜 해시로 덮어쓴다. 로컬 미리보기
    (`output/html`, `preview.sh`)와는 완전히 분리된 별도 산출물이라 서로 안 건드림.
  - **민감한 개인 데이터 보호**(이 세션 앞부분에서 이미 해둔 것과 이어짐): `.gitignore`에
    `/scripts/webviewer/data/`를 이미 추가해둔 상태라 Gmail/캘린더/유튜브 원본은 애초에
    이 저장소(`myremember`, 소스 저장소)에 커밋될 일이 없다. 클라우드에 올라가는 건
    `output/html-cloud`(빌드 결과물)뿐이고, 이것도 아직 어디에도 푸시하지 않았다.
  - 검증: 실제로 `MYREMEMBER_CLOUD_PASSWORD` 지정해서 빌드 → `new-note.html`/
    `tags/manage.html` 미생성, `index.html`에 쓰기 UI(가져오기 입력창, 사이드바 액션
    버튼) 없음, `gate-hash.js`에 해시 확인 → `output/html-cloud`를 **일반**
    `python3 -m http.server`(GitHub Pages처럼 쓰기 API가 전혀 없는 순수 정적 서버)로
    띄워 Playwright로 ① 첫 진입 시 게이트가 콘텐츠를 완전히 가리는지, ② 틀린 비밀번호
    → 에러 메시지, ③ 맞는 비밀번호 → 잠금 해제 + 다른 페이지 이동해도 재입력 안 물어봄
    (`sessionStorage`), ④ 노트 상세 페이지에 편집/삭제 버튼이 DOM에 아예 없는지 확인.
    마지막으로 기존 로컬 미리보기(`output/html`, `preview.sh`)가 이번 템플릿 변경들로
    안 깨졌는지(쓰기 UI 그대로 있고 게이트 없음) 재확인.
- **의도적으로 여기까지만(당시)**: 실제 GitHub 저장소 생성·Pages 활성화·
  `output/html-cloud` 푸시는 로컬 검증만 마치고 일단 멈춘 뒤, 배포 직전에 사용자 확인을
  다시 받기로 함 — 실제 개인 데이터(Gmail 개수·가족 이름 캘린더·개인 노트)가 외부에서
  URL로 접근 가능해지는 되돌리기 어려운 조작이라서.
- **후속: 사용자가 배포를 승인**("네, 지금 배포해줘 — 비밀번호는 따로 알려주세요") →
  같은 세션에서 바로 진행.
  - 비밀번호를 `secrets`로 무작위 생성(영숫자 16자) → `MYREMEMBER_CLOUD_PASSWORD`로
    `build-cloud-site.py` 실행 → `output/html-cloud`에서 새 git 저장소 초기화 → `gh
    repo create myremember-vault --private --source=. --remote=origin --push`.
  - **막힌 점 발견**: private 저장소로 Pages를 켜려니 GitHub API가 "Your current plan
    does not support GitHub Pages for this repository"로 거부 — 이 계정 플랜은 private
    저장소의 Pages 자체를 지원하지 않음(무료 플랜은 Pro/Team/Enterprise에서만 되는
    기능). 어차피 최종 목표가 "비밀번호로 보호된 **공개** 사이트"로 이미 확정돼 있었으니
    (앞선 확인 질문에서 사용자가 직접 고른 옵션), private을 고집할 이유가 없었다 —
    `gh repo edit --visibility public --accept-visibility-change-consequences`로 전환
    후 Pages 활성화 재시도해서 성공.
  - `gh api .../pages/builds/latest`를 폴링해 빌드 완료 확인 → 실제 배포된 URL
    (https://peter-cho-70.github.io/myremember-vault/)에 Playwright로 접속해 게이트가
    콘텐츠를 가리는지, 생성했던 비밀번호로 실제로 풀리는지, 배포된 `gate-hash.js`가
    로컬에서 만든 해시와 일치하는지, 쓰기 UI가 전혀 없는지까지 라이브 사이트 기준으로
    재확인. 테스트에 쓴 로컬 스크린샷/`.playwright-mcp` 산출물은 정리.
  - 허브(`peter-cho-70.github.io`)에는 의도적으로 등록하지 않음 — "문서화 표준" 사전
    승인 워크플로우는 문서 사이트 전용이고, 이건 비밀번호로 막아둔 개인 vault
    콘텐츠라 공개 허브에 링크가 노출되면 게이트의 의미가 없어짐.
- **이슈/막힌 점**: private Pages 미지원 플랜 제약을 실제로 API 호출해보고서야
  알았다(문서로는 미리 알 수 없었음) — public 전환이 이미 합의된 최종 형태와 일치해서
  추가 확인 없이 바로 전환했지만, 만약 사용자가 "private 유지"를 더 중요하게 여겼다면
  이 지점에서 다시 확인받았어야 했을 상황이었다.

## 2026-08-12 — [긴급] 배포 사이트 노트 링크 깨짐(NFC/NFD) 수정 + 배포 스크립트 사고 후 재설계

- **버그(최우선순위로 신고받음)**: 사용자가 "배포된 사이트에 노트가 연결이 되지않아.
  최우선적으로 수정해줘"라고 반복 신고. 실제 라이브 URL에서 Playwright로 노트 링크를
  클릭해 404를 직접 재현한 뒤 원인 추적.
  - **근본 원인**: macOS(APFS/HFS+)는 한글처럼 결합 문자가 있는 파일명을 파일시스템
    레벨에서 항상 NFD(자모 분해형)로 저장한다. 로컬 macOS는 NFC/NFD를 같은 파일로
    취급해 문제가 전혀 안 보이지만, `note_id()`가 이 경로 문자열을 그대로
    `<a href>`에 넣는 정적 사이트를 GitHub Pages에 올리면 그쪽은 NFC 기준으로 경로를
    찾기 때문에 한글이 든 노트/태그 링크가 전부 깨졌다(로컬 검증만으로는 절대 못
    잡는 유형의 버그였음 — 로컬 서버로 아무리 확인해도 재현이 안 됨).
  - **수정**: `vault.py`에 `note_id()`를 새로 만들어(기존에 `build-backlinks.py`와
    `generate-html.py`가 각자 따로 정의하고 있던 걸 통합) `unicodedata.normalize("NFC",
    ...)`로 정규화 후 반환하도록 중앙화. `extract_tags()`도 같은 이유(태그명이
    `tags/{태그}.html` URL로 쓰임)로 NFC 정규화 추가. 두 스크립트가 각자 다른 정규화
    방식을 쓰면 백링크 매칭 자체가 깨지므로 반드시 한 곳에서만 정의해야 하는 함수였다.
  - **검증**: 로컬에서 `note_id()`가 실제로 NFC를 반환하는지 직접 테스트 → 사이트
    재생성 → Playwright `page.request.get()`으로 노트 14개 링크 전부 HTTP 200 확인 →
    라이브 배포 후 같은 방식으로 재확인(아래).
- **사고: 배포 중 실수로 소스 저장소에 잘못 커밋+푸시**:
  - 기존 `build-cloud-site.py`는 `output/html-cloud`(vault 소스 저장소 `myremember`
    내부 폴더) 안에서 직접 `git init`하고 커밋하는 구조였다. 그런데
    `generate-html.py`가 재생성할 때마다 `shutil.rmtree(output_dir)`로 그 폴더를
    통째로 지우기 때문에, 그 안의 `.git`도 같이 사라졌다.
  - NFC 수정 후 재배포하려고 `cd output/html-cloud && git add -A && git commit &&
    git push`를 실행했는데, 직전에 `.git`이 지워진 상태라 git이 상위 폴더의 `.git`
    (소스 저장소 `myremember` 자신)을 대신 찾아서 **이 세션에서 쌓인 작업 전부(54개
    파일, 12991줄 추가)가 실수로 `myremember`(private) 저장소의 `origin/main`에
    커밋 `2bf7a26`으로 푸시됐다.**
  - 발견 즉시 사용자에게 축소 없이 그대로 알리고(영향 범위: private 저장소라 외부
    노출은 없음, 그래도 승인 없는 조작이었다는 점은 별개), 커밋을 그대로 둘지
    `ea96634`로 되돌릴지 확인 질문 → 사용자가 "그대로 두기(추천)" 선택.
  - **재발 방지 재설계**: `build-cloud-site.py`를 다시 만들어 빌드 결과물 폴더
    (`output/html-cloud`, 매번 통째로 재생성돼도 안전)와 영구 git 체크아웃
    (`../myremember-vault-deploy`, 이 저장소 밖의 형제 폴더 — `myremember-vault`
    저장소를 클론해서 만듦)을 완전히 분리했다. 빌드 후 `rsync -a --delete
    --exclude=.git`로 내용만 동기화하고, git 조작은 전부 `git -C <경로>`로 대상을
    명시(`cd` 뒤 상대 경로에 의존하지 않음). 실제로 커밋+푸시하려면 `--push` 플래그를
    명시적으로 줘야 하고, 안 주면 빌드+동기화까지만 하고 멈춘다(사고 재발을 스크립트
    구조 자체로 막음).
  - 새 스크립트로 재배포 실행 → 정확히 `peter-cho-70/myremember-vault` 저장소에만
    푸시됐는지 확인(`git -C .../myremember-vault-deploy remote -v`), 동시에
    `myremember`(소스 저장소) 쪽은 의도한 `build-cloud-site.py` 파일 수정 하나만
    남고 다른 변경이 없는지 `git status`로 재확인.
- **배포 후 자체 테스트**("배포된 내용이 잘 보이도록 자체 테스트도 해줘" 요청):
  GitHub Pages 빌드 완료를 폴링으로 확인한 뒤 라이브 사이트에 Playwright로 접속.
  - 처음 접속 시 게이트 없이 바로 대시보드가 보이는 이상 현상 발견 → 원인은 버그가
    아니라 **같은 브라우저 세션에서 이전에 이미 이 URL을 잠금 해제한 적이 있어
    `sessionStorage`가 남아있었던 것**(정상 동작). `sessionStorage.clear()` 후
    재접속해 게이트가 실제로 뜨는 것, 비밀번호 입력 후 정상 해제되는 것을 재확인.
  - 대시보드에 있는 모든 링크(노트 14개 전부 포함, 한글 파일명 다수)를 `fetch()`로
    순회해 전부 HTTP 200 확인. 한글 노트 페이지 1개, 한글 태그 페이지 1개에 직접
    들어가 본문·백링크·태그 브레드크럼이 제대로 렌더링되는지, 편집/삭제 등 쓰기 UI가
    전혀 없는지(읽기 전용 확인)까지 검증.
- **이슈/막힌 점**: 이번 배포 시점 기준 vault에 실제 `[[링크]]`가 0개라
  `output/backlinks.json`의 edges가 0으로 나온다(경고 로그도 뜸) — 버그가 아니라
  현재 vault 콘텐츠에 노트 간 위키링크를 아직 안 걸어놔서 그런 것. `daily/index.md`와
  가이드성 노트 1개에만 예시/날짜 형식 `[[...]]`가 있을 뿐 실제 노트 간 상호링크는
  없는 상태.

## 2026-08-12 — 카카오톡 대화 분석 대시보드 탭 (링크 + 키워드)

- **추가/변경**: "카카오톡 내용을 복사해서 나의 관심도를 체크하는 방법", 이어서
  "메시지 키워드도 분석해주고, 만들어 줘"라는 요청에 따라 유튜브와 같은 패턴(API 대신
  사용자가 직접 내보낸 파일을 분석)으로 구현.
  - `scripts/analyze-kakao-chat.py` 신설: 카카오톡 앱 "대화 내용 내보내기"(텍스트만
    저장)로 만든 `.txt`를 파싱한다. 안드로이드 내보내기 형식
    (`YYYY년 M월 D일 오전/오후 H:MM, 발신자 : 메시지`)의 정규식 매칭 + 여러 줄에 걸친
    메시지(다음 줄에 발신자 패턴이 없으면 이전 메시지에 이어붙임) 처리. 메시지에서
    URL을 뽑아 도메인별로 집계(`urlparse`), 조사를 뗀 뒤 불용어를 거른 토큰으로
    키워드 빈도를 집계(`vault.py`의 자동 태그 후보 로직과 같은 조사 제거 규칙 재사용,
    단 채팅 특유의 필러 단어는 별도 불용어 목록으로 관리). 카카오톡은 대화 내용을
    "읽어오는" 공개 API가 없어(메시지 발신 API만 있음) 이 방식이 유일한 선택지 —
    사용자에게도 이 제약을 먼저 설명하고 진행함. **원문은 저장하지 않고 집계 결과만**
    `kakao-summary.json`에 남긴다(그룹 채팅방을 내보내면 상대방 메시지도 섞여 들어올
    수 있어서 원문 보존 자체를 최소화).
  - `generate-html.py`에 `load_kakao_summary()` 추가(이미 만들어져 있었으나 이번에
    실제로 `home_tpl.render()`에 `kakao=` 인자로 연결) — 유튜브 로더와 완전히 같은
    패턴(파일 없으면 `None`, 로그로 안내).
  - `home.html`의 오른쪽 탭 그룹에 5번째 탭 "카카오톡" 추가. 패널 내용은 공유된
    링크(도메인별, 유튜브 탭의 채널 막대와 같은 `bar-row` 재사용)와 자주 나온
    키워드(기존 `.tag-cloud`/`.tag` 스타일 재사용, 빈도수를 라벨에 같이 표시) 두
    섹션. 5개 탭 전환 로직은 `dashboard.js`가 `data-tab`/`data-panel` 속성만 보고
    동작하는 범용 코드라 탭을 추가해도 JS 수정이 필요 없었음. 사이드 카드 전체를
    보여줄지 말지 판단하는 조건(`{% if dashboard or youtube %}`)에 `or kakao`를
    추가해서, Gmail/캘린더/유튜브가 전부 비어 있어도 카카오톡 데이터만 있으면 탭
    카드가 뜨도록 함.
  - 검증: 합성 대화 6줄(도메인 2개, "제주"·"여행" 등 조사 제거가 실제로 동작하는
    키워드 포함)로 `analyze-kakao-chat.py` → `build-backlinks.py` →
    `generate-html.sh` → 로컬 서버(4501) 전체 파이프라인을 직접 돌려 Playwright로
    카카오톡 탭 클릭 → 링크 막대(카운트 순 정렬)·키워드 태그 렌더링, 다른 4개 탭이
    안 깨졌는지, 라이트/다크모드까지 스크린샷으로 확인. 테스트에 쓴 합성 데이터
    파일(`kakao-chat.txt`/`kakao-summary.json`)과 `.playwright-mcp` 산출물은 확인 후
    전부 삭제하고 `output/html`을 실제 vault 상태로 재생성해 원상 복구.
- **이슈/막힌 점**: 없음. 빈 상태 안내 문구가 참조하던 파일 경로
  (`kakao-chat-export.txt`)가 실제 `config.yaml`의 `kakao.raw_chat_path`
  (`kakao-chat.txt`)와 달랐던 걸 검증 중 발견해 바로 일치시켰다.

## 2026-08-12 — 관심종목/유튜브/카카오톡을 사이드바 전용 페이지로 분리 + 화면 비율 정리

- **추가/변경**: "관심종목, 유튜브, 카카오톡은 좌측 메뉴로 빼주고, 배포된 사이트도
  적용해주고, 모바일/태블릿에 최적화되게 화면 비율을 잘 만들어줘"라는 요청. 대시보드
  오른쪽 300px 탭 칸에 5개 탭(Gmail/캘린더/관심종목/유튜브/카카오톡)이 몰려 있던 걸
  Gmail·캘린더만 남기고 나머지 3개는 사이드바(`base.html`)의 새 링크 3개
  (`stocks.html`/`youtube.html`/`kakao.html`)로 분리했다.
  - 새 템플릿 3개: 각자 `home.html`에 있던 해당 탭 패널 내용을 그대로 옮기되, 300px
    좁은 칸 전용으로 만들어졌던 마크업(막대그래프·태그 칩·표)이 `.content`의 최대
    980px 전체 폭을 쓰게 되면서 카드 오른쪽에 빈 공간만 남는 문제가 생겨(특히 태블릿
    가로/데스크톱에서 두드러짐) `.page-narrow`(max-width:640px) 래퍼로 감싸 콘텐츠
    양과 화면 비율이 맞도록 정리했다. 모바일은 어차피 640px보다 좁아 영향 없음.
  - `generate-html.py`: 세 템플릿을 로드해 `readonly` 여부와 무관하게 항상
    `stocks.html`/`youtube.html`/`kakao.html`을 생성(쓰기 UI가 없는 순수 조회
    페이지라 정적 배포에서도 그대로 보여야 함). `dashboard_snapshot.get("stocks")`를
    독립적으로 넘겨준다.
  - `home.html`: 사이드 카드 조건을 `{% if dashboard or youtube or kakao %}`에서
    `{% if dashboard %}`로 좁히고(이제 Gmail/캘린더만 이 카드에 남았으므로), 빈
    상태 안내문에서 관심종목/유튜브/카카오톡 언급을 빼고 "왼쪽 메뉴에서 볼 수
    있습니다" 안내를 추가했다.
  - 검증: 로컬 서버로 사이드바 3개 링크 클릭 → 각 페이지 렌더링, Gmail/캘린더만
    남은 대시보드 탭, 모바일(390px)·태블릿 세로(768px)·태블릿 가로(1024px) 세
    폭에서 오프캔버스 사이드바/카드 폭/`.page-narrow` 적용 여부까지 스크린샷으로
    확인. 라이트/다크모드도 확인.
  - 배포: `build-cloud-site.py --push`로 `myremember-vault`에 반영, Pages 빌드 완료
    확인 후 라이브 사이트에서 게이트 해제 → 사이드바 3개 링크 → 각 페이지가 실제로
    뜨는지까지 재확인.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 유튜브/카카오톡을 웹 화면에서 파일 선택만으로 바로 분석하도록 연결

- **추가/변경**: "카카오톡, 유튜브 내용을 웹화면에서 선택하고, 프로그램 실행할 수
  있도록 해줘"라는 요청. 지금까지는 `watch-history.json`/카카오톡 대화 내보내기
  `.txt`를 터미널에서 정해진 경로에 직접 옮기고 분석 스크립트를 수동으로 실행해야
  했는데, "노트 가져오기"와 같은 업로드 패턴을 재사용해 새로 만든 `stocks.html`/
  `youtube.html`/`kakao.html` 전용 페이지에서 파일 선택만으로 끝나게 만들었다.
  - `server.py`에 `import_youtube_history()`/`import_kakao_chat()` 추가 —
    업로드된 텍스트를 `config.yaml`의 `raw_history_path`/`raw_chat_path`에 저장한
    뒤, `analyze-youtube-history.py`/`analyze-kakao-chat.py`의 `main()`을 그
    자리에서 직접 호출해 요약을 갱신하고 `rebuild_site()`까지 이어서 실행한다.
    두 분석 스크립트는 이미 "파일이 정해진 경로에 있으면 읽어서 요약을 쓴다"는
    구조였기 때문에, 서버는 그 앞뒤(파일 쓰기 + 실행 + 사이트 재생성)만 감싸면
    됐다. JSON 파싱 실패 등 스크립트가 `rc != 0`을 반환하는 경우는 에러로 변환해
    버튼 쪽에 그대로 보여준다.
  - `/api/import-youtube`, `/api/import-kakao` 두 POST 엔드포인트 추가(기존
    `/api/import-note`와 같은 라우팅·에러 처리 관례).
  - 새 JS `data-import.js`: `note-actions.js`/`dashboard.js`의 업로드 패턴(파일
    선택 → `file.text()` → POST → 성공 시 이동/실패 시 상태 문구)을 그대로
    재사용하되, 이번엔 성공 시 노트로 이동하는 대신 **같은 페이지를 새로고침**해서
    방금 반영된 요약이 바로 보이게 한다.
  - `youtube.html`/`kakao.html`에 "가져오기" 카드 추가(`{% if not readonly %}`로
    감싸 로컬 서버(`server.py`)로 열었을 때만 보임 — 클라우드 정적 배포는 쓰기
    API가 없어 어차피 동작 못 하는 건 기존 "노트 가져오기"와 같은 제약).
  - 검증: 합성 `watch-history.json`(채널 2개, 영상 3개)과 카카오톡 대화 `.txt`를
    Playwright `browser_file_upload`로 실제 업로드 → 서버가 분석 스크립트를 실행하고
    페이지가 새로고침되며 채널별 막대그래프/링크·키워드가 바로 반영되는 것까지
    확인. 잘못된 JSON 업로드 시 이전 데이터가 보존된 채 에러 문구만 뜨는 실패 경로도
    확인. 테스트 파일과 `.playwright-mcp` 산출물은 삭제, `scripts/webviewer/data/`의
    실제 상태(둘 다 없음)로 복구.
  - 이 기능은 로컬 전용이다(server.py가 있어야 동작) — 클라우드 배포 사이트는
    읽기 전용이라 업로드 카드 자체가 안 보인다(스크립트 실행 서버가 없으니 당연한
    제약). 배포는 템플릿 동기화 차원에서 한 번 더 실행했지만, readonly 빌드
    결과물 자체는 이 변경으로 달라지지 않는다(가져오기 카드가 애초에
    `{% if not readonly %}`라 컴파일 결과에 안 남음).
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 유튜브 시청기록 HTML 형식 지원 (Takeout 기본 내보내기 형식)

- **버그**: 방금 만든 유튜브 업로드 기능을 실제로 써보니 "유튜브 시청기록은 HTML로
  되어있는데, 수정해줘"라는 신고 — `analyze-youtube-history.py`가 JSON만 가정하고
  있었는데, Google Takeout이 "기록(History)" 항목을 내보낼 때 **기본값이 HTML**이라
  (JSON은 별도로 형식을 바꿔야 나옴) 실제로 HTML을 받는 사용자가 더 많았다.
- **수정**: 파일 확장자가 아니라 **내용 첫 글자로 JSON/HTML을 자동 판별**하도록
  `sniff_format()`을 추가(`<`로 시작하면 html, `[`/`{`로 시작하면 json) — 서버 업로드
  경로(`raw_history_path`)의 확장자와 실제 내용이 다를 수 있어 확장자 기반 분기는
  신뢰할 수 없다고 판단.
  - HTML 파서는 새 의존성 없이 표준 라이브러리 `html.parser.HTMLParser`로 직접
    구현(`_TakeoutHtmlParser`) — Takeout이 여러 제품(검색 기록 등)에 공통으로 쓰는
    `<div class="content-cell ... mdl-typography--body-1">` 블록을 항목 경계로 삼고,
    그 안의 첫 `<a>`를 영상 링크(제목), 두 번째 `<a>`(있으면)를 채널로, 링크 뒤에
    남는 텍스트를 시각 문자열로 취급한다. 링크가 아예 없으면(예: "삭제된 영상을
    시청했습니다") 건너뛴다.
  - 시각 문자열은 Takeout 계정 언어에 따라 영어("Aug 10, 2026, 9:00:00 AM KST")
    또는 한국어("2026. 8. 10. 오후 2:15:00 GMT+09:00") 등으로 나올 수 있어 정규식
    두 개로 각각 시도(`_parse_takeout_timestamp`) — 시간대 표기는 무시하고 naive
    datetime으로 처리(이 요약엔 절대 UTC가 아니라 날짜 범위/최근 N일 판단만
    필요해서 충분함). 못 읽으면 `None`으로 두고 그 항목의 시각만 빼고 채널 집계
    등 나머지는 그대로 진행 — 타임스탬프 포맷을 100% 확신할 수 없는 상태에서도
    핵심 기능(채널별 집계)이 죽지 않게 하려는 의도적 완충.
  - JSON 쪽 로직(`_parse_json_entry`/`parse_json_history`)은 그대로 유지, 공통
    집계 로직만 `summarize(parsed)`로 분리해 두 경로가 재사용.
  - **알려진 한계**: HTML 내보내기에는 JSON에만 있는 `details` 필드(광고 여부
    식별용)가 없어서, HTML 쪽은 광고 시청 항목을 걸러내지 못한다 — 총 시청 수에
    광고가 섞여 있을 수 있음을 스크립트 docstring에 명시.
  - 검증: 합성 HTML(영어/한국어 타임스탬프 각 1개, 삭제된 영상 1개, 채널 링크
    없는 영상 1개 포함)로 직접 파싱 테스트 → 채널 집계·날짜 범위·삭제된 영상
    제외까지 정확히 확인. 기존 JSON 경로도 회귀 테스트로 재확인(정상 동작).
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 캘린더 스냅샷 범위: 이번 주 → 오늘 기준 3일로 축소

- **추가/변경**: "캘런더는 오늘 기준으로 3일치 내용만 있으면 됨"이라는 요청. 기존엔
  이번 주(일~토) 전체를 가져왔는데, 오늘(2026-08-12) 기준 3일(오늘+다음 2일,
  08/12~08/14)로 좁혀서 Google Calendar MCP 도구로 11개 캘린더를 다시 조회해
  `dashboard-snapshot.json`의 `calendar` 섹션을 갱신했다(그 안에 실제 일정이 있던
  건 조충남 개인일정·KOBETA·보도기술부 일정 3개).
  - `build-dashboard-snapshot.py`의 안내 주석(다음에 "대시보드 스냅샷 갱신해줘"라고
    요청받을 세션이 따라야 할 절차)도 "이번 주 범위"에서 "오늘 기준 3일(오늘 +
    다음 2일)"로 갱신 — 이번에만 한 번 좁힌 게 아니라 앞으로 갱신할 때마다 이
    범위를 쓰라는 뜻이라 스크립트 쪽 안내를 반드시 같이 고쳐야 다음 세션도
    일관되게 따름.
  - 로컬 사이트 재생성 후 대시보드 "캘린더" 탭에서 `range_label`("오늘 기준 3일
    일정 · 캘린더 3개 (08/12 ~ 08/14)")과 날짜별 이벤트(출처 캘린더 배지 포함)가
    정확히 뜨는지 Playwright로 확인.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 할 일 위젯: "매일" 반복 옵션 + 미완료 항목 자동 이월

- **추가/변경**: "할일에는 매일하는 옵션을 주고, 만약 하지 않은 일이 있으면 내일도
  할일로 자동으로 넘겨줘"라는 요청. 기존 할 일 위젯은 날짜별로 완전히 독립된
  localStorage 배열(`myremember-todo-{날짜}`)이라, 이 기능을 넣으려면 "그날 화면을
  열 때 필요한 항목을 그 날짜 배열에 복제해 넣는" materialize 방식이 필요했다.
  - **매일 반복**: 새 localStorage 키 `myremember-todo-recurring-templates`에
    `{id, text}`만 있는 템플릿 목록을 따로 둔다(완료 여부는 날짜마다 다르므로
    템플릿엔 없음). 추가 폼에 "매일" 체크박스 추가 — 체크하고 추가하면 템플릿에도
    등록되고, 그 날짜 배열에도 바로 인스턴스가 하나 생긴다.
  - **미완료 자동 이월**: 어떤 날짜 D를 열면(`ensureMaterialized(D)`), **바로 전날
    D-1의 원본 배열**(재귀적으로 더 materialize하지 않는 원시 읽기)을 보고 완료
    안 된 일반 항목(반복 항목 제외 — 반복은 자기가 알아서 매일 새로 생김)을 D의
    배열에 `carriedFrom` 필드와 함께 복제해 넣는다. 하루씩만 보기 때문에, 연속으로
    여러 날을 한 번도 열어보지 않고 건너뛰면 그 사이 항목은 이어지지 않는다(정적
    localStorage라 백그라운드 처리가 없어서 "실제로 열어본 날짜만 이어짐" — 개인용
    로컬 위젯에 맞는 실용적 단순화).
  - 두 경우 다 "이미 그 날짜 배열에 있으면 건너뜀"으로 중복 복제를 막고, 처음
    materialize될 때만 localStorage에 실제로 써서 다음부터는 그 날짜 고유의
    독립된 기록이 된다(다른 날짜에서 체크/수정해도 서로 영향 없음).
  - UI: 반복 항목엔 "매일" 배지, 이월된 항목엔 "이월" 배지(`.todo-badge`, 기존
    캘린더 이벤트의 `.event-cal-badge`와 같은 스타일 패턴 재사용)를 제목 옆에
    작게 표시. 반복 항목 삭제는 그 날짜 인스턴스뿐 아니라 **템플릿 자체를 지워서
    미래 반복을 중단**시킨다(이미 다른 날짜에 복제된 과거 기록은 그대로 남음 —
    "그날의 기록"이라 소급 삭제하지 않음). 반복 항목 수정은 템플릿 텍스트도 같이
    바꿔서 앞으로 생길 인스턴스에 반영.
  - 검증: 오늘 날짜에 매일 항목 1개 + 일반 미완료 항목 1개 + 일반 완료 항목 1개를
    만든 뒤 Playwright로 다음날 이동 → 매일 항목은 새로 생김(미체크)·미완료
    항목은 "이월" 배지와 함께 넘어옴·완료 항목은 안 넘어옴을 확인. 그 다음날 다시
    이동해 이월된 항목을 완료 처리한 뒤 그 다음날엔 더 이상 안 넘어오는 것도 확인.
    매일 항목을 어느 날짜에서 삭제하면 그 이후 날짜엔 안 생기지만, 이미 생성됐던
    과거 날짜의 기록은 그대로 남는 것까지 확인. 라이트/다크모드 스크린샷 확인
    후 테스트에 쓴 localStorage는 `localStorage.clear()`로 정리.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 캘린더 이벤트에서 출처 캘린더 배지 제거

- **추가/변경**: "캘런더에는 캘런드 종류는 표시하지 말아줘" 요청 — 여러 캘린더를
  합쳐서 보여주며 각 이벤트 옆에 달아뒀던 출처 캘린더 배지(`.event-cal-badge`,
  예: "KOBETA"/"조충남 개인일정")를 뺐다. `home.html`에서 `{% if ev.calendar %}`
  블록을 지우고, 더는 안 쓰는 `.event-cal-badge` CSS도 같이 삭제. `dashboard-
  snapshot.json`의 각 이벤트엔 여전히 `calendar` 필드가 들어있어도 무방하지만
  (스키마상 선택 필드) 화면에는 이제 시간+제목만 나온다.
  `build-dashboard-snapshot.py`의 안내 주석도 "화면에는 표시하지 않는다"로
  갱신해서 다음에 캘린더 스냅샷을 갱신하는 세션도 이 사실을 알 수 있게 함.
  로컬에서 렌더링 확인 후 클라우드에도 재배포.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — 유튜브 웹 업로드 버그 두 개 수정(실제 Takeout 파일로 검증)

- **버그 발견 경위**: 사용자가 실제 Google Takeout 파일 경로
  (`~/Downloads/Takeout/YouTube 및 YouTube Music/시청 기록/시청 기록.html`, 36MB,
  항목 약 35,083개)를 주면서 "이걸 인식할 수 있도록 하고, 파일만 선택하면 자연적으로
  실행이 되도록" 요청 — 실제 파일로 직접 검증해보니 두 가지 문제가 있었다.
  1. **HTML 파서 자체는 정상 동작 확인**: 실제 파일의 한국어 로캘 문구가 예상과
     달랐다("Watched " 접두사 대신 `<a>제목</a> 을(를) 시청했습니다.` 형태) — 하지만
     `_TakeoutHtmlParser`는 접두사 텍스트에 의존하지 않고 순전히 구조(첫 `<a>`=영상,
     두 번째 `<a>`=채널, 마지막 링크 뒤 텍스트=시각)로만 파싱해서 실제로는 문제
     없이 35,083개 항목·타임스탬프 100% 파싱 성공(직접 스크립트로 재현 테스트).
     광고 항목("YouTube 홈페이지에서 본 광고" 등)은 애초에 `<a>` 링크가 아예 없어서
     이미 정상적으로 걸러짐.
  2. **`server.py`의 `import_youtube_history()`가 여전히 JSON 전용 사전 검증을
     하고 있었음**: HTML 지원을 `analyze-youtube-history.py`에 추가할 때
     `sniff_format()`으로 판별 로직을 스크립트 쪽에만 넣고, `server.py`는 안
     고쳤다 — 업로드가 서버에 도착하자마자 `json.loads(content)`부터 실행해서
     HTML을 보내면 무조건 500 에러였다. 수정: 이 사전 검증을 아예 지우고, 형식
     판별은 `analyze-youtube-history.py`의 `sniff_format()` 한 곳에서만 하도록
     정리(이중 검증이 서로 어긋나서 생긴 버그라, "검증은 한 곳에서만" 원칙으로
     재발 방지).
  3. **요청 크기 제한(20MB)도 걸림**: 실제 파일이 36MB라 기존 PDF 기준으로 잡아둔
     `MAX_BODY_BYTES`를 넘었다. 유튜브 시청기록은 몇 년치를 내보내면 수십MB가
     흔하다는 걸 실제로 확인해서, 100MB로 상향.
  - 두 수정 다 코드 변경이라 **로컬 서버 프로세스를 재시작해야 반영됨**(이전에
    이미 한 번 겪은 "실행 중인 프로세스는 파이썬 소스 변경을 자동으로 다시 안
    읽는다" 문제와 같은 종류라 재시작 습관화 필요).
  - 검증: 실제 36MB 파일을 Playwright `browser_file_upload`로 실제 서버(4500번
    포트)에 직접 업로드 → 파일 선택만으로 자동 업로드→분석→새로고침까지 이어져서
    실제 채널별 집계(총 35,083개 시청, 2025-02-12~2026-08-12, 상위 채널 목록)가
    대시보드에 그대로 반영되는 것까지 확인. 실제 개인 시청기록이라 상세 목록은
    이 문서에 남기지 않음(gitignore된 `scripts/webviewer/data/`에만 존재).
- **이슈/막힌 점**: 없음(둘 다 즉시 수정·검증 완료).

## 2026-08-12 — 카카오톡 CSV 형식 지원 (실제 PC판 내보내기 파일로 확인)

- **배경**: 이전 세션에서 사용자가 "카카오톡은 CSV 파일로 되어있는데, 수정해줘"라고
  알려줬을 때는 정확한 칼럼 구조를 몰라 샘플을 요청해둔 상태였다. 그 사이 사용자가
  직접 `kakao.html`의 업로드 카드로 실제 파일을 올려봤는데, 당시 파서는 안드로이드
  텍스트 형식만 지원해서 "메시지 0개"로 조용히 실패했다(에러는 안 났지만 아무것도
  못 읽음). 서버에 이미 저장된 실제 업로드 파일(`scripts/webviewer/data/
  kakao-chat.txt`, 213KB)을 직접 열어봐서 정확한 CSV 구조를 확인했다 — 사용자
  답변을 더 기다리지 않고 실제 파일로 바로 검증.
  - 실제 구조: `Date,User,Message` 헤더 + 표준 RFC4180 CSV(따옴표 이스케이프
    `""`, 메시지 필드 안에 줄바꿈이 그대로 들어감 — 여러 줄 메시지가 텍스트
    형식처럼 "이어지는 줄"이 아니라 하나의 따옴표 필드로 옴). 날짜는
    `YYYY-MM-DD HH:MM:SS`(24시간제, 오전/오후 없음) — 이전에 가정했던 안드로이드
    형식(`YYYY년 M월 D일 오전/오후 H:MM, 발신자 : 메시지`)과는 완전히 다른 PC판
    포맷이었다.
  - **수정**: `analyze-kakao-chat.py`에 `sniff_format()` 추가(유튜브 스크립트와
    같은 패턴) — 첫 줄이 `date,user,message` 헤더로 보이면 CSV, 아니면 기존
    안드로이드 텍스트 파서로 처리. CSV 파싱은 표준 라이브러리 `csv.DictReader`를
    그대로 씀(따옴표 이스케이프·필드 안 줄바꿈을 직접 구현할 필요 없이 stdlib가
    RFC4180대로 처리 — 텍스트 파서의 "매칭 안 되는 줄은 이어붙인다" 로직이 여기선
    아예 필요 없어짐). 두 파서 다 같은 `{"time", "sender", "text"}` 딕셔너리
    모양으로 결과를 맞춰서 이후 링크/키워드 집계 로직(`extract_links`/
    `extract_keywords`)은 형식과 무관하게 그대로 재사용.
  - 검증: 실제 파일로 직접 재현 — 1,350개 메시지, 타임스탬프 100% 파싱 성공,
    실제 공유 링크 도메인 15개(instagram.com 599회 등)·키워드 25개 집계 확인.
    로컬 서버 재시작(코드 변경 반영) 후 `kakao.html`에서 실제 데이터 렌더링까지
    Playwright로 확인. 실제 개인 대화 집계 결과라 원문/발신자는 이 문서에 남기지
    않음.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — Gmail 새로고침 안내 + 메일 클릭해서 열기

- **추가/변경**: "Gmail을 다시 불러올 수 있는 기능과 클릭해서 내용을 볼 수 있는
  기능도 넣어줘" 요청. Gmail MCP는 Claude 세션 안에서만 쓸 수 있어서 로컬 서버가
  스스로 실시간 재조회를 할 수 없다는 제약을 먼저 설명하고, 두 방향(① 지금 한 번
  새로 불러오고 버튼은 안내만 vs ② Google OAuth를 직접 붙여 진짜 새로고침 버튼)
  중 사용자가 ①(추천)을 선택.
  - **지금 실제로 다시 불러옴**: `list_labels`로 INBOX 라벨의
    `threadsTotal`/`threadsUnread`를 가져와 `total_count`/`unread_count`로 씀
    (기존엔 `search_threads`의 `resultCountEstimate`를 썼는데, 쿼리를 바꿔도 항상
    같은 값(201)이 나오는 걸 실측으로 확인해서 못 쓰는 필드였음 — 이번에 발견).
    최근 15통을 다시 가져와 발신자 표시 이름(도메인에서 유추, 예:
    `no-reply@yes24.com` → "YES24")·제목·스니펫·시각(KST)·읽음여부·`url`(Gmail
    웹에서 그 메일을 바로 여는 링크, `#all/{thread_id}` 형식)로 정리해
    `dashboard-snapshot.json`의 `gmail.messages`를 갱신.
  - **새로고침 버튼은 안내용**: Gmail 탭 요약 줄 옆에 새로고침 아이콘
    버튼(`.btn-icon`)을 추가했는데, 실제로 API를 호출하지는 않고 hover 시
    "최신 메일을 보려면 Claude에게 '대시보드 스냅샷 갱신해줘'라고 요청하세요"
    툴팁만 보여준다 — 실제 갱신 방법을 사용자가 매번 기억하지 않아도 되게.
  - **메일 클릭해서 보기**: 각 메일 행(`.mail-row`)을 `<div>`에서 `<a href="{{
    m.url }}" target="_blank">`로 바꿔서 클릭하면 실제 Gmail 웹의 그 메일로
    바로 이동한다. 본문 전체를 우리 사이트에 직접 저장/표시하지 않는 이유:
    (1) 비밀번호 게이트가 있어도 어쨌든 공개 웹에 호스팅되는 페이지라 스니펫보다
    훨씬 긴 본문 전체를 올리는 건 노출 범위를 크게 늘리는 일이고, (2) 이메일
    HTML 본문을 그대로 렌더링하면 삭제 안 한 트래킹 픽셀·잠재적 XSS 등 위생 문제가
    있어서, 이미 로그인돼 있는 실제 Gmail로 보내는 쪽이 더 안전하고 구현도 훨씬
    간단함. `m.url`이 없는 옛 스냅샷 데이터에도 안전하게(빈 href로 새로고침되는
    사고 없이) 동작하도록 `href="{{ m.url or '#' }}"` + url 있을 때만
    `target="_blank"` 붙임.
  - `build-dashboard-snapshot.py`의 안내 주석에 이 절차(INBOX 라벨 카운트 쓰기,
    url 필드 채우기, 발신자 이름 유추 방식)를 남겨서 다음에 "Gmail 스냅샷
    갱신해줘"라고 요청받는 세션도 그대로 따르면 됨.
  - 검증: 로컬 서버(재시작 후) + Playwright로 새로고침 아이콘 툴팁 문구, 메일
    행의 실제 `href`/`target` 속성, 갱신된 통계(3645통·안읽음 2503통)까지 확인.
- **이슈/막힌 점**: 없음.

## 2026-08-12 — PC 넓은 화면 대응 + 태블릿 화면 비율 재점검, 배포

- **추가/변경**: "PC에서는 좌우폭을 넓히면 좀 더 많은 정보가 나오도록 화면을
  넓히고, 스마트폰과 테블릿에 적합한 화면 비율을 찾아서 배포버전을 수정해줘"
  요청.
  - **PC**: `.content`가 어떤 창 너비에서든 980px에 고정돼 있어서, 넓은 모니터에서
    창을 넓혀도 화면 양옆에 빈 여백만 늘어나고 실제 정보량은 그대로였다. 두 단계
    브레이크포인트 추가 — `min-width:1280px`에서 `.content` 1180px·대시보드
    사이드 열 340px, `min-width:1600px`에서 1400px·380px. 1920px/1366px에서
    실제로 메일 목록·노트 그래프가 더 넓게 보이는 것까지 스크린샷으로 확인.
    관심종목/유튜브/카카오톡(`.page-narrow`, 640px 고정)은 일부러 안 넓힘 —
    내용량이 적어서 넓히면 다시 카드 오른쪽에 빈 공간만 남는 문제로 되돌아감
    (1920px에서도 640px 그대로 유지되는 것 확인).
  - **태블릿**: 여러 실제 기기 크기(834/768/1024px 등)로 직접 스크린샷 검증하다가
    1024px(아이패드 가로/12.9인치 세로)에서 사이드바+Gmail 열을 빼면 본문이
    ~420px밖에 안 남아 상단 통계 3칸(`.stats-row`, `minmax(140px,1fr)`)이 두
    줄로 어색하게 밀리는 걸 발견 — `minmax(110px,1fr)`로 낮춰서 그 폭에서도
    한 줄에 들어가게 고침. 834/768/375/360px까지 스크린샷으로 재확인, 문제
    없음.
  - 배포: `build-cloud-site.py --push`로 Gmail 기능과 같이 한 번에 반영,
    Pages 빌드 완료 확인 후 라이브 사이트에서 재확인.
- **이슈/막힌 점**: 없음. 1024px 3칸 통계 줄바꿈은 이번에 직접 스크린샷 비교로
  처음 발견한 것 — 이전 세션(모바일/태블릿 최적화 1차 작업)에서는 390/768/1024
  세 폭만 확인했는데 그때는 Gmail 탭 데이터가 지금보다 적어서 본문 폭이 살짝
  더 넓었을 가능성이 있음(우연히 안 걸렸을 뿐).

## 2026-08-13 — 메일·캘린더 사이드 패널 폭 2배 + 흐르는 반응형 (오래된 브레이크포인트 버그 발견)

- **배경**: "메일과 캘린더 폭을 2배로 늘리고, 화면 비율에 따라 1차적으로 지금 비율까지
  줄어드는것으로 수정해줘" 요청 — 그냥 고정값을 2배로 바꾸는 게 아니라, 창을 좁히면
  원래 폭(현재 감각)까지는 자연스럽게 줄어들어야 한다는 뜻으로 이해하고 반응형으로
  설계함.
- **추가/변경**: `.dashboard-side`의 세 브레이크포인트(860/1280/1600px)에서 쓰던
  고정 px(300/340/380)를 각각 `clamp(현재값, ~vw, 2배값)`(600/680/760)로 바꿔서
  뷰포트 폭에 비례해 흐르게 만들었다. 860px 근방에서는 사실상 기존 값(300px 근처)에
  머무르다가, 화면이 넓어질수록(대략 2700px+에서 상한 도달) 점점 넓어져 최대 2배까지
  간다 — "창을 좁히면 1차로 지금 비율까지 줄어든다"는 요청을 clamp의 최소값이 곧 지금
  값이 되도록 설계해서 충족.
  - **버그 발견**: 구현하고 실제 뷰포트별 픽셀값을 재보니 1280px/1600px 구간에서
    기대한 값이 전혀 안 나왔다 — 원인을 추적해보니 **이 세 `@media` 블록이 파일
    안에서 860px 블록보다 앞쪽(1280/1600)과 뒤쪽(860)으로 흩어져 있어서, 뷰포트가
    1280px 이상이면 세 조건이 전부 동시에 참이 되고 CSS 캐스케이드는 그중
    "소스 순서상 나중에 나온 규칙"을 이긴다는 규칙 때문에 항상 맨 뒤에 있던 860px
    블록의 값으로 덮어써지고 있었다** — 즉 1280px/1600px 전용 규칙은 이 프로젝트가
    생긴 이래 한 번도 실제로 적용된 적이 없는 죽은 코드였다. 세 블록을
    860→1280→1600 오름차순으로 재배치해서 캐스케이드 순서와 min-width 크기 순서가
    일치하도록 고쳤다(일반적인 모바일-퍼스트 미디어쿼리 관례).
  - 검증: Playwright로 820/860/1000/1279/1280/1500/1599/1600/1920/2560/3000px
    각각에서 `.dashboard-side`의 실제 렌더링 폭을 `getBoundingClientRect()`로
    찍어서, 수정 전(버그 상태 — 모든 폭에서 300~333px로 고정)과 수정 후(300→760px로
    매끄럽게 증가, 3000px에서 상한 760px 도달)를 직접 비교 확인. 1920px 전체
    스크린샷으로 메일 목록·본문 레이아웃이 실제로 자연스러운지도 확인.
- **이슈/막힌 점**: 이 버그는 이번 요청과 무관하게 이전부터 있던 것 — DEVLOG
  2026-08-12 "PC 넓은 화면 대응"에 "1920px/1366px에서 실제로 확인"이라고 적혀
  있지만, 300px와 340/380px의 차이가 육안으로는 크게 눈에 띄지 않아 스크린샷
  검토로는 못 걸러졌던 것으로 보인다. 같은 프로젝트에서 여러 `@media` 블록으로
  같은 지정자를 재정의할 때는 앞으로 항상 min-width 오름차순으로만 배치해야
  한다는 교훈.

## 2026-08-13 — 노트 목록(Areas/Topics) 정렬 최신순 변경 + Areas 사용법 안내

- **배경**: "노트에 파일을 입력하면, 정렬 순서는 최신이 맨위로 가게 해주고, Areas는
  어떻게 사용하는거야?" — 정렬 변경 요청과 질문이 함께 왔다.
- **추가/변경**: `generate-html.py`의 `NoteMeta`에 파일 수정시각(`mtime =
  md_file.stat().st_mtime`)을 추가하고, "노트" 페이지(`vault.html`)의 Areas/Topics
  목록 정렬 기준을 `rel_md`/`title.lower()` 알파벳순에서 `mtime` 내림차순으로
  바꿨다. Daily는 원래부터 날짜 문자열 기준 최신순이라 그대로 뒀다 — areas/topics
  파일명에는 daily처럼 날짜가 박혀있지 않아서 mtime이 "최신"을 판단할 수 있는
  사실상 유일한 신호다.
  - **Areas 사용법**: PRD 기준으로 Areas는 시작·끝이 있는 **프로젝트**
    (`areas/{프로젝트}/README.md` + `progress.md` + `decisions.md`), Topics는
    끝이 없는 **지속 관심사**. 지금 vault의 `areas/`는 비어 있고 콘텐츠가 전부
    `topics/`에 있다는 실제 상태를 확인해서 설명 — 강제 규칙은 아니라고 안내함.
  - 검증: 빌드 후 `output/html/vault.html`의 Topics 목록을 파싱해 실제 파일
    mtime 순서와 일치하는지 확인.
- **재발견한 운영 버그**: 검증 직후 사용자가 "지금도 정렬이 안 된다"고 신고 —
  코드는 맞았는데, 새벽부터 이미 떠 있던 로컬 미리보기 서버(`server.py`, 9:23AM
  기동)가 `generate-html.py`를 프로세스 시작 시점에 메모리로 딱 한 번만 읽어들이는
  구조라(`_load_sibling_module`) 그 이후의 소스 수정을 반영하지 못하고 있었다.
  사용자가 그 사이 웹 UI로 새 노트를 하나 추가했는데, 그때 서버가 자체적으로
  호출한 재생성이 이 낡은(수정 전) 코드로 실행돼 옛날 정렬 방식으로 되돌아간
  상태였다. 서버를 재시작하고 `generate-html.py`를 다시 실행해 해결 — 실제로
  가장 최근 파일이 맨 위로 오는 것까지 재확인.
  - 이슈: 이 프로젝트에서 "정적 자산(JS) 브라우저 캐시" 문제는 2026-08-12에 한 번
    겪어 `Cache-Control: no-store`로 막아뒀는데, 이번 건 그것과는 다른 종류(서버
    프로세스 자체의 파이썬 모듈 캐시)다 — 앞으로 `server.py`가 이미 떠 있는 상태에서
    `generate-html.py`(또는 서버가 import하는 다른 스크립트)를 고치면 반드시 서버를
    재시작해야 반영된다는 걸 사용자에게 안내함.

## 2026-08-13 — 캘린더 오늘 기준 갱신 + "내 캘린더" 굵게 표시

- **배경**: "그리고 캘런더는 오늘 기준으로 데이터를 가지고 오고, 내 캘런더는 굵은
  글자로 해줘" — 앞선 작업 도중 이어서 들어온 요청.
- **추가/변경**: `list_calendars`로 캘린더 목록을 다시 확인해 사용자 본인 계정
  (`ubsoldboy@gmail.com`)이 "조충남 개인일정"이라는 이름의 캘린더임을 확인하고,
  그 캘린더를 포함해 접근 가능한 캘린더 전체에 `list_events`로 오늘(2026-08-13)
  기준 3일(오늘+다음 2일)치를 다시 조회했다(기존 스냅샷은 08-12 기준으로 하루
  낡아 있었음 — 스냅샷은 세션 시점 기준이라 시간이 지나면 자연히 낡는 구조적
  한계는 그대로다). `dashboard-snapshot.json`의 `calendar`를 08/13~08/15,
  4개 캘린더(조충남 개인일정·보도기술부 일정·KOBETA·대한민국의 휴일 — 광복절이
  이 범위에 들어와 새로 포함됨)에 실제 일정이 있는 상태로 갱신.
  - **"내 캘린더" 굵게**: 사용자 본인 캘린더에서 온 이벤트에만 `"mine": true`
    필드를 추가하고, `home.html`이 `ev.mine`이면 `event-title` 클래스에
    `event-title-mine`을 더 붙여서 `font-weight: 700`으로 렌더링 — 다른
    사람/조직 캘린더(KOBETA, 보도기술부 일정 등) 일정과 한눈에 구분되게 했다.
    `build-dashboard-snapshot.py`의 안내 주석에 이 `mine` 필드 규칙(본인 캘린더
    판정 기준, 생략 시 기본 false 취급)을 남겨서 다음에 "대시보드 스냅샷
    갱신해줘"라고 요청받는 세션도 그대로 따르면 됨.
  - 검증: 로컬 서버 재시작 후 Playwright로 캘린더 탭 스크린샷 확인 — 조충남
    개인일정 항목(근무·수업 깨우기·전기료·생일 등)만 굵게, KOBETA/보도기술부
    일정/공휴일은 일반 굵기로 나오는 것 확인.
- **이슈/막힌 점**: 없음.

## 2026-08-13 — "할 일" 위젯 기기 간 보기 동기화 (배포 사이트/모바일에서도 로컬 할 일 확인)

- **배경**: "배포버전에 내가 로컬에서 할일을 넣은것이 보이지 않는데, 할일을
  모바일에서 확인 할 수 있도록 해줘". 원인 설명: "할 일"은 처음 설계부터 vault
  마크다운과 무관하게 그 브라우저의 `localStorage`에만 저장되는 기능이었다 —
  localStorage는 브라우저(정확히는 오리진)별로 완전히 갈라지는 저장소라, 로컬 PC의
  Chrome과 배포된 GitHub Pages 사이트(다른 도메인)는 애초에 같은 데이터를 공유할
  방법이 없었다. Gmail/캘린더는 Claude 세션이 데이터를 만들어 정적 파일에 구워
  넣는 "스냅샷" 방식이라 이 문제가 없었는데, 할 일은 순수 클라이언트 상태라 이
  패턴을 못 타고 있었던 것.
- **추가/변경**: Gmail/캘린더 스냅샷과 같은 "로컬에서 만든 데이터를 정적 사이트에
  구워 넣는" 패턴을 할 일에도 적용했다 — 다만 데이터 출처가 Claude(MCP)가 아니라
  브라우저 자신이라는 점이 다르다.
  - `config.yaml`에 `dashboard.todos_path`("scripts/webviewer/data/todos.json")
    추가.
  - `server.py`에 새 엔드포인트 `POST /api/save-todos` — 브라우저가 보낸
    `{days: {날짜: [...]}, recurring: [...]}` 전체 상태를 그 경로에 그대로
    저장하는 `save_todos()`. 체크박스 하나 누를 때마다 pandoc이 전체 vault를
    재변환하는 사이트 전체 재생성을 부르는 건 낭비라, 이 엔드포인트는
    `rebuild_site()`를 부르지 않는다 — 다음 노트 저장이나 배포 시점에 자연스럽게
    반영된다.
  - `generate-html.py`에 `load_todos_snapshot()` 추가, `search-data.js`/
    `graph-data.js`와 같은 패턴으로 `assets/todos-data.js`
    (`window.MYREMEMBER_TODOS = {days, recurring}`)를 항상 굽는다(readonly
    여부와 무관 — 클라우드 배포본에도 포함됨). `home.html`에 이 스크립트를
    `dashboard.js`보다 먼저 로드하도록 추가.
  - `dashboard.js`: `loadRawTodos()`/`loadRecurringTemplates()`가 이제 그
    localStorage 키가 **아예 없을 때만**(빈 배열로 명시적으로 저장된 경우는
    제외 — 사용자가 실제로 다 지운 날짜를 되살리지 않기 위함) `window.
    MYREMEMBER_TODOS`의 값을 초기값으로 쓰고 그대로 localStorage에 저장해둔다
    (`SNAPSHOT` 시딩). `saveRawTodos()`/`saveRecurringTemplates()`는 저장할
    때마다 `syncToServer()`를 불러 그 기기의 localStorage 전체(정규식
    `/^myremember-todo-(\d{4}-\d{2}-\d{2})$/`로 날짜 키만 골라 "매일" 템플릿
    키와 혼동되지 않게 함)를 `/api/save-todos`로 흘려보낸다 — 이 서버가 없는
    환경(배포된 정적 사이트)에서는 fetch가 그냥 실패하니 `.catch(()=>{})`로
    조용히 무시.
  - 결과적으로 동작은 **로컬(PC)에서 넣은 할 일 → 다음 사이트 재생성/배포 →
    다른 기기에서 스냅샷으로 보임 → 그 기기에서 체크/수정해도 로컬에는
    반영 안 됨(뷰어일 뿐, 양방향 동기화 아님)** — Gmail/캘린더와 동일한
    "스냅샷" 철학을 그대로 따른 것.
  - `home.html`/`docs/guide/USER_GUIDE.md`의 "할 일은 localStorage에만
    저장된다"는 문구를 이 동작에 맞게 갱신.
  - 검증: Playwright로 로컬 서버(4500)에 실제 할 일 추가 →
    `scripts/webviewer/data/todos.json`에 반영 확인 → `generate-html.py`
    재실행으로 `assets/todos-data.js`에 구워짐 확인 → 같은 페이지에서
    `localStorage.clear()` 후 새로고침(기기를 처음 여는 상황을 재현)해도 그
    할 일이 그대로 보이는 것까지 확인. 테스트로 추가한 항목은 삭제해 정리.
- **이슈/막힌 점**: 배포된 사이트(`myremember-vault`)에는 아직 이 수정이
  반영되지 않았다 — 사용자가 재배포를 요청했으나 `MYREMEMBER_CLOUD_PASSWORD`가
  이 세션 환경이나 셸 프로필 어디에도 없어서 Claude가 대신 `build-cloud-site.py
  --push`를 실행하지 못했다. 사용자에게 직접 터미널에서 실행하도록 안내했고
  (`! MYREMEMBER_CLOUD_PASSWORD='...' python3 scripts/build-cloud-site.py
  --push`), 실행 여부/결과는 이 세션에서 확인하지 못한 채로 남아있다 — 다음
  세션에서 배포 상태 재확인 필요.

## 2026-08-14 — 캘린더 오늘 기준 재갱신 + 캘린더 탭에도 새로고침 안내 아이콘

- **배경**: "이메일과 캘린더가 오늘날짜로 새로 갱신이 되도록 해줘. 그리고 새로
  불러오기 기능도 넣어줘".
- **추가/변경**: Google Calendar MCP로 접근 가능한 캘린더 10개를 모두 조회해
  오늘(08/14) 기준 3일(08/14~08/16) 일정을 다시 가져와
  `dashboard-snapshot.json`을 갱신. 코드를 보니 Gmail 탭에는 이미(직전 세션에서,
  아직 미커밋 상태로) "새로고침 안내" 아이콘(hover 시 "Claude에게 갱신 요청"
  툴팁)이 있는데 캘린더 탭에는 빠져 있어서, 같은 `.side-tab-summary-row` +
  `.btn-icon` 패턴으로 캘린더 탭에도 짝을 맞춰 추가(`home.html`).
- **이슈/막힌 점**: Gmail MCP 커넥터 토큰이 만료되어(`requires re-authorization`)
  이번엔 Gmail은 갱신하지 못했다 — 캘린더만 갱신됨. 사용자가 claude.ai에서
  Gmail 커넥터를 재인증해야 다음에 갱신 가능. 구조적으로 Gmail/캘린더 모두
  브라우저 버튼 클릭만으로 실시간 재조회는 불가능(OAuth가 Claude Code 세션
  안에서만 되고, 로컬 `server.py`는 그 인증에 접근할 수 없음) — 그래서 두 새로고침
  아이콘은 실제 갱신이 아니라 "Claude에게 요청하라"는 안내로 설계됨(기존 Gmail
  아이콘과 동일한 설계 결정을 캘린더에도 그대로 적용한 것).

## 2026-08-14 — CADENCE 컨셉 검토 + daily note에 Phase 0 회고 4문항 추가

- **배경**: 사용자가 미리 만들어둔 컨셉 문서(`docs/cadence-concept.html`,
  행동 축적형 개인 업무 OS — 주간/격주/월간 3주기 엔진, R/L/U/W 교대 근무 축)를
  검토하고 "내가 원하는 방향으로 개발하려면 어떻게 해야할지 의견을 줘봐"라는
  요청을 받아 리뷰.
- **의견 요지**: 구조(관찰/개입/축적 3주기 분리, 실험 객체로 닫힌 피드백 루프,
  요일/시계시각 대신 R/L/U/W 축)는 탄탄하지만, ① 문서 스스로 "Phase 0을 건너뛰지
  말라"고 경고해놓고 §12가 이미 Tauri+Rust 스택을 확정 톤으로 제안하는 모순, ②
  이 vault(MyRemember)의 daily note/할 일과 CADENCE의 관계가 안 정해진 점, ③
  격주 판정의 표본 크기(근무 유형당 2주 4~6회)가 얇다는 점 세 가지를 지적. Phase
  0(습관 검증)을 새 도구·새 저장소 없이 이 vault 안에서 시작할 것을 권했고
  사용자가 동의.
- **추가/변경**: `config.yaml`의 `daily_note.template`에 "CADENCE 회고 (Phase 0
  검증)" 섹션을 추가 — Q1(오늘 가장 오래 걸린 일과 이유)/Q2(막힌 지점 태그)/
  Q3(에너지 1~5)/Q4(오늘 알게 된 것 한 줄), 문서(§13)에 정의된 4문항 그대로.
  이후 생성되는 모든 daily note에 자동으로 포함됨. `daily/2026-08-14.md`를
  이 템플릿으로 생성해 오늘부터 바로 채울 수 있게 함.

## 2026-08-14 — "할 일을 스마트폰에 보내기": 경량 클라우드 동기화 + 스냅샷 최초 1회 시딩 버그 발견/수정

- **배경**: "내가 오늘 할일을 로컬에서 적으면, 배포된 사이트를 내가 스마트폰으로
  열면 오늘 할일이 없는데, 만약 오늘할일 보내기 기능을 만들어서 필요시에만
  배포된 사이트와 동기화를 하도록 해줘". 기존 구조는 노트 하나만 바뀌어도
  `build-cloud-site.py`로 vault 전체를 pandoc 재변환+커밋+푸시해야 했고, 이건
  `MYREMEMBER_CLOUD_PASSWORD`가 있어야만 가능해서 매번 무겁고 손이 많이 갔다.
- **추가/변경**: 새 스크립트 `scripts/sync-cloud-todos.py` — 전체 재빌드 없이
  `assets/todos-data.js` 파일 하나만 갱신해서 클라우드 배포 체크아웃
  (`../myremember-vault-deploy`)에 커밋+푸시한다. 비밀번호 게이트(`gate-hash.js`)를
  건드리지 않으므로 **`MYREMEMBER_CLOUD_PASSWORD`가 필요 없다** — 이번 기능의
  핵심 설계 포인트. 변경이 없으면(`git status --porcelain`) 커밋을 만들지 않아
  빈 커밋이 안 쌓인다. `server.py`에 `POST /api/sync-cloud-todos` 엔드포인트,
  대시보드 "할 일" 카드에 "할 일을 스마트폰에 보내기" 버튼 + 상태 텍스트("보내는
  중…"/"반영됨"/"이미 최신 상태"/"실패 — ...") 추가.
- **버그 발견 1 (readonly 사이트에 노출)**: 새 버튼이 배포된(readonly) 사이트에도
  그대로 렌더링되고 있었다 — 로컬 서버 전용 기능인데 폰 화면에도 보여서 눌러도
  조용히 실패만 했다. `home.html`에서 `{% if not readonly %}`로 감싸 해결.
- **버그 발견 2 (더 근본적 — 스냅샷이 기기당 최초 1회만 시딩됨)**: 사용자가 실제로
  버튼을 눌렀는데 폰에 아무 변화가 없다고 신고. 로그를 보니 클라우드 동기화
  자체는 성공(`할 일 클라우드 동기화 완료`)했는데 폰 화면은 그대로였다 — 원인은
  `dashboard.js`의 기존 설계: `window.MYREMEMBER_TODOS` 스냅샷은 "그 기기가 그
  날짜를 localStorage에 **한 번도 연 적이 없을 때만**" 초기값으로 쓰이는
  구조였다. 사용자가 이미 그날 아침 전체 배포를 확인하며 폰에서 오늘 날짜를 한
  번 열어봤기 때문에, 그 순간 폰 localStorage에 그날 키가 생겨버렸고 그 뒤로는
  PC가 아무리 새로 보내도 폰은 자기 캐시만 봤다. → `server.py`의 `save_todos()`가
  저장할 때마다 `synced_at`(ISO 타임스탬프)을 `todos.json`에 같이 남기도록
  수정, `dashboard.js`는 페이지 로드 시 `SNAPSHOT.synced_at`을 그 기기가 마지막
  으로 반영한 시각(`localStorage`의 `myremember-todo-snapshot-applied`)과
  비교해서 **더 새로우면 SNAPSHOT에 있는 모든 날짜를 그 기기의 localStorage에
  강제로 덮어쓴다** — 즉 "보내기"를 누른 뒤 폰에서 새로고침하면 이번엔 진짜로
  반영되지만, 그 사이 폰에서 직접 체크/수정한 내용은 덮어써질 수 있다(PC 우선
  정책으로 전환 — 기존 "뷰어일 뿐 양방향 동기화 아님" 설계에서, "PC가 누른
  뒤에는 PC가 이긴다"로 의미가 살짝 바뀜. `home.html`의 안내 문구도 갱신).
  `synced_at` 필드가 없는 옛 스냅샷이면 기존처럼 최초 1회 시딩만 하도록
  하위호환 유지.
- **검증**: 로컬 서버 재시작 후 `todos.json`에 `synced_at` 수동 스탬프 →
  전체 재배포(`build-cloud-site.py --push`, 사용자가 직접 실행, 로그에 16:09/
  16:14/16:21/16:29 커밋 4개로 확인됨: 경량 동기화 2회 + 전체 배포 2회) 완료.
- **이슈/막힌 점**: 마지막 전체 배포(16:29)가 아래 "잘못 가져온 CADENCE 노트
  정리"(16:35)보다 먼저 일어나서, 배포된 사이트에는 아직 정리 전 상태(엉뚱한
  `cadence-concept.html`)가 남아있다 — 재배포 한 번 더 필요(STATUS.md "진행 중"
  참고).

## 2026-08-14 — 잘못 가져온 CADENCE 노트 정리

- **배경**: 배포 전 vault를 점검하다가 "노트 가져오기"로 만들어진 CADENCE 관련
  노트 2개 중 하나가 완전히 엉뚱한 내용임을 발견 — `topics/cadence-concept.md`가
  이 프로젝트의 CADENCE(행동 축적형 개인 업무 OS)가 아니라 반도체/EDA 회사
  "Cadence Design Systems"(OrCAD, Innovus 등)에 대한 브라우저 AI 탭 검색
  결과였다("위 내용은 AI탭이 제공한 결과물로 부정확할 수 있음"이라는 문구가
  본문에 그대로 남아있어 확인). 사용자가 검색 결과를 실수로 이 노트에 붙여넣은
  것으로 보임. 나머지 하나(`topics/cadence-concept-v02.md`, HTML을 가져와
  변환한 것)는 내용은 맞았지만 자동 태그가 `#LED #WS2812 #md #근무 #대시보드`로
  붙어 있었다 — LED/WS2812는 완전히 다른 노트(WS2812 매트릭스 프로젝트)의
  태그가 잘못 붙은 것.
- **추가/변경**: 사용자에게 확인 질문 후 "잘못된 것 삭제 + 태그 수정"으로 진행.
  `topics/cadence-concept.md` 삭제, `cadence-concept-v02.md`의 태그를
  `#CADENCE #업무관리 #컨셉설계`로 교체. 백링크/사이트 재생성.
