# ARCHITECTURE

> 설계 배경(Argo에서 가져온 개념, 전체 로드맵)은 [`myremember_prd.md`](./myremember_prd.md) 참고.
> 이 문서는 "지금 실제로 어떻게 동작하는지"를 유지한다 — PRD는 계획, 이 문서는 구현 현황.

## 원칙

1. **로컬 우선**: vault(`areas/`, `topics/`, `daily/`)는 순수 마크다운 파일이며, 어떤
   스크립트도 없이 그 자체로 완결된다. 스크립트는 전부 부가 기능(검증/생성/변환)이다.
2. **경로는 항상 스크립트 파일 위치 기준**: `scripts/vault.py`가 `Path(__file__)` 기준으로
   vault 루트를 계산한다. vault 전체를 다른 위치로 옮겨도 스크립트가 그대로 동작한다.
3. **생성물과 원본의 분리**: `output/`, `backups/`는 스크립트로 언제든 재생성 가능한
   파생 데이터라서 git에 커밋하지 않는다(`.gitignore`). 원본은 `areas/`, `topics/`, `daily/`뿐.
4. **멱등성(idempotent)**: 자동화 스크립트는 몇 번을 실행해도 같은 결과여야 한다
   (cron으로 매일 돌리는 것을 전제). 예: `create-daily-note.py`는 이미 있는 노트를 건드리지 않는다.

## 공통 모듈: `scripts/vault.py`

모든 스크립트가 공유하는 것:
- `VAULT_ROOT`: vault 루트 절대경로
- `load_config()`: `scripts/config.yaml` 로드
- `vault_path(*parts)`: vault 루트 기준 상대경로 → 절대경로
- `note_dirs(config)`: `config["note_dirs"]`(areas/topics/daily)를 절대경로 리스트로
- `get_logger(name, config)`: `scripts/logs/{name}.log` 파일 + stdout 동시 출력하는 로거
- `strip_code_blocks(text)`: 코드 펜스(` ``` `) 안의 텍스트 제거 (링크/태그 파싱 오탐 방지)
- `iter_note_files(dirs)`: 폴더 목록 아래 모든 노트(`*.md`, `index.md` 제외) 순회
- `dir_type(path)`: vault 루트 기준 최상위 폴더명(`areas`/`topics`/`daily`) 반환
- `collect_notes(dirs)`: `검색 키(소문자) -> 파일 경로 목록`. 동일 키가 여러 개면 모호 링크 후보
- `note_lookup_keys(path)`: 이 노트가 `[[링크]]`에서 검색될 수 있는 이름들 — 기본은 파일명
  (stem)이지만, `areas/{project}/README.md`처럼 "폴더가 곧 노트"인 경우 폴더명도 별칭으로 추가.
  `collect_notes`가 내부적으로 쓰고, `generate-html.py`도 같은 규칙을 쓰려고 직접 import한다.
- `extract_title(text, fallback)` / `extract_tags(text)`: 첫 `# ` 헤딩을 제목으로, 본문의
  `#해시태그`를 태그로 추출 (코드블록 제외). `build-search-index.py`와 `generate-html.py`가 공유.

새 스크립트를 추가할 때는 이 모듈을 그대로 재사용한다 (설정 파싱/로깅/노트 순회를 각자 새로 만들지 않는다).

## 자동화 에이전트(Crew) 현황

PRD 3.2절의 crew 패턴 기준. `상태` 컬럼이 이 저장소의 실제 구현 여부.

| 이름 | 역할 | 스크립트 | 상태 |
|------|------|---------|------|
| DailyScribe | 오늘 daily note 생성 + 월별 인덱스 갱신 | `create-daily-note.py` | ✅ 구현 완료 |
| LinkValidator | `[[링크]]` 무결성 검사, 리포트 생성 | `validate-links.py` | ✅ 구현 완료 |
| SearchIndexer | SQLite 전문검색 인덱스 | `build-search-index.py` | ✅ 구현 완료 |
| BacklinkMapper | 역참조 그래프(JSON) 생성 | `build-backlinks.py` | ✅ 구현 완료 |
| (CLI 검색 도구) | ripgrep+fzf 인터랙티브 검색 | `search.sh` | ✅ 구현 완료 (PRD 표에는 크루가 아니지만 Phase 2 항목) |
| HtmlPublisher | Pandoc+Jinja2로 정적 사이트 생성 | `generate-html.sh` → `generate-html.py` | ✅ 구현 완료 (GitHub Pages 배포는 보류 — 아래 참고) |
| VersionKeeper | git 자동 커밋/태그/푸시 | `git-auto-commit.sh` | ✅ 구현 완료 (push 실제 실행은 사용자 승인 대기) |
| BackupKeeper | Rclone + 로컬 압축 백업 | `backup.sh` | ✅ 구현 완료 (rclone 설정·실제 클라우드 업로드는 대기) |
| (대시보드 스냅샷 검증) | Gmail/캘린더/관심종목 JSON 스키마 검증 + 빈 틀 생성 | `build-dashboard-snapshot.py` | ✅ 구현 완료 (PRD에는 없는 항목 — 실제 데이터 수집은 Claude 세션에서만 가능, 아래 "대시보드 위젯" 참고) |

### LinkValidator 상세 동작

- 대상: `config.yaml`의 `note_dirs`(`areas`, `topics`, `daily`) 아래 모든 `*.md`
  (단, 각 폴더의 `index.md`는 링크 대상 후보에서는 제외 — 생성물이라 서로 다른 폴더에 중복될 수 있음)
- 코드 펜스(` ``` `) 안의 텍스트는 링크로 인식하지 않는다.
- 링크 대상 매칭은 **파일 경로가 아니라 파일명(확장자 제외, 대소문자 무시)** 기준.
  같은 파일명이 여러 폴더에 있으면 "모호한 링크"로 별도 리포트.
- 결과: `output/link-report.md` + 로그. 깨진 링크가 하나라도 있으면 exit code 1
  (나중에 cron/GitHub Actions에서 실패로 감지 가능).

### SearchIndexer 상세 동작

- `output/search.db`에 SQLite **FTS5** 가상 테이블(`notes`)을 만든다. 증분 인덱싱이 아니라
  실행할 때마다 통째로 재생성 — 개인 vault 규모에서는 전체 재빌드가 충분히 빠르고,
  삭제/이름변경된 노트가 인덱스에 유령으로 남는 문제를 원천적으로 피한다.
- 컬럼: `title`(첫 `# ` 헤딩, 없으면 파일명), `body`(전체 본문), `tags`(본문의 `#해시태그`,
  코드블록 제외), `path`/`dir_type`(UNINDEXED, 결과 표시용).
- `--query "검색어"`로 기존 인덱스를 검색(재빌드 없음). 검색어는 항상 큰따옴표로 감싼
  **phrase 검색**으로 처리한다 — FTS5 쿼리 문법에서 `-`, `:` 등이 연산자로 해석되어
  `test-tag` 같은 태그 검색이 깨지는 문제가 있어, 사용자 입력을 리터럴로 고정했다.

### BacklinkMapper 상세 동작

- 링크 파싱은 LinkValidator와 같은 `config.yaml`의 `link_validation.pattern`을 공유한다.
- **노드 id는 파일명이 아니라 vault 루트 기준 경로**(확장자 제외)다. `areas/*/README.md`처럼
  서로 다른 노트가 같은 파일명을 쓰는 경우가 흔해서, stem만으로 노드를 구분하면 서로
  다른 프로젝트의 README가 그래프에서 하나로 뭉개진다 — 초기 구현에서 이 문제를 발견해
  경로 기반 id로 수정했다.
- 깨진 링크(대상 노트 없음)는 엣지에 포함하지 않는다 — 그건 LinkValidator의 책임.
- 모호한 링크(동명 파일 여럿)는 후보 중 첫 번째로 연결한다. 근본 해결은 파일명을 고유하게
  바꾸는 것 (LinkValidator 리포트로 확인 가능).
- 출력(`output/backlinks.json`): `nodes`(id/path/dir_type), `edges`(source/target),
  `backlinks`(타깃 id → 그 노트를 링크한 소스 id 목록). D3.js/Vis.js 등 그래프 시각화에
  바로 쓸 수 있는 형태.

### search.sh (CLI 검색 도구)

- `ripgrep`으로 vault(`areas`/`topics`/`daily`)를 라이브 검색하고, `fzf`가 설치되어 있으면
  그 결과를 인터랙티브하게 다시 필터링한다. SQLite 인덱스와 무관하게 항상 최신 파일
  내용을 대상으로 하므로 `build-search-index.py`를 먼저 돌릴 필요가 없다.
- `fzf` 미설치 시에도 동작은 하되(ripgrep 결과만 출력) 인터랙티브 필터링은 빠진다 —
  이 저장소를 개발한 환경에는 `fzf`가 없어서 그 폴백 경로로 검증했다. `fzf` 설치 후
  인터랙티브 프리뷰(`cat -n`으로 파일 전체 표시) 동작은 사용자가 직접 확인 필요.

### HtmlPublisher 상세 동작 (`generate-html.py`, `generate-html.sh`는 얇은 래퍼)

- `output/html`을 통째로 재생성한다. 다른 생성 스크립트와 같은 철학(증분 아님, 매번 새로).
- **`[[위키링크]]는 Pandoc에 넘기기 전에 직접 치환한다** (Pandoc의 wikilinks 확장에 맡기지
  않음). 이유: LinkValidator/BacklinkMapper와 정확히 같은 규칙(`vault.note_lookup_keys`
  기준 파일명/폴더명 매칭)으로 링크를 풀어야 세 스크립트의 판단이 일치한다. 깨진 링크는
  빌드를 실패시키지 않고 `<span class="broken-link">`로 표시만 한다.
- 헤딩 링크(`[[note#heading]]`)의 앵커는 자체 slugify 함수로 만든다 — Pandoc의
  `auto_identifiers`가 만드는 id와 100% 동일한 알고리즘은 아니지만(공백→하이픈, 영숫자/한글
  유지, 나머지 제거), 테스트한 한글 헤딩 케이스에서는 일치했다. 완벽한 재현이 필요하면
  Pandoc의 실제 id 생성 로직을 참고해 보완할 것.
- 백링크 패널은 `output/backlinks.json`이 있으면 읽어서 채운다. 없으면 경고 로그만 남기고
  생략 — `build-backlinks.py`를 먼저 돌리라고 안내한다 (스크립트 간 강한 의존을 피하려고
  import가 아니라 `output/`의 생성물을 통해서만 연결한다).
- 검색 페이지는 서버 없이(로컬에서 `index.html`을 그냥 열어도) 동작해야 해서 `fetch()`로
  JSON을 읽지 않고 `assets/search-data.js`에 `window.MYREMEMBER_NOTES = [...]`를 박아
  `<script>` 태그로 로드한다. (참고: 이 파일을 만들 때 처음엔 `const MYREMEMBER_NOTES = ...`로
  썼다가 검색이 항상 "결과 없음"만 뜨는 버그가 났다 — 최상위 `const`/`let`은 전역 렉시컬
  스코프에만 생기고 `window` 객체 프로퍼티로는 붙지 않는다는 JS 특성 때문. `window.` 명시
  할당으로 고쳤다.)
- 다크모드는 `prefers-color-scheme` 기본값 + 우측 상단 토글(로컬스토리지에 사용자 선택 저장).
- 템플릿(`scripts/webviewer/templates/*.html`, Jinja2)과 정적 자산(`scripts/webviewer/static/`)은
  스크립트와 분리되어 있어 디자인만 따로 수정 가능하다.
- "사이트맵"(PRD 3.2절 HtmlPublisher 규칙)은 별도 페이지 대신 홈페이지가 겸한다 —
  areas/topics/daily 전체 목록이 이미 홈에 있어서 중복 페이지를 만들지 않았다.
- **UI**: 사이드바(대시보드/노트/태그/검색/Daily 아카이브) + 카드 레이아웃. 모바일에서는
  사이드바가 오프캔버스로 접히고 햄버거 버튼(`assets/nav.js`)으로 연다. 디자인 토큰(색상
  변수, `rgb(var(--x))` 패턴)은 사용자가 준 참고 대시보드 HTML(Tailwind CDN 기반)의 팔레트를
  순수 CSS로 그대로 옮긴 것 — 이 프로젝트는 로컬/오프라인 사용을 전제로 하므로 Tailwind CDN이나
  Google Fonts 같은 외부 리소스는 쓰지 않고 `scripts/webviewer/static/style.css`에 직접 구현했다.
  `index.html`(대시보드)과 `vault.html`(기존 홈 콘텐츠: areas/topics/daily 목록)로 역할을
  분리했다 — `home.html` 템플릿이 대시보드, `vault.html` 템플릿이 옛 홈 콘텐츠를 렌더링한다.
- **pandoc 헤딩+목록 인접 버그 우회**: pandoc 3.10 계열에서 `## 헤딩\n-\n`처럼 헤딩 바로 다음 줄에
  빈 줄 없이 목록이 오면 `<h2 id="...">## 헤딩</h2>`처럼 텍스트에 `#`이 그대로 남는 파싱 버그가
  있다(id는 정상 생성됨 — 텍스트만 깨짐). `daily_note.template`이 정확히 이 패턴이라 daily note를
  볼 때마다 발생했다. `render_note_bodies()`에서 pandoc에 넘기기 전에 `fix_heading_list_adjacency()`로
  헤딩과 목록 사이에 빈 줄을 자동 삽입해 우회한다(원본 `.md` 파일은 건드리지 않음).

### "오늘 Daily Note" 버튼 + `scripts/webviewer/server.py`

- 지금까지 웹뷰어는 완전히 정적이었다(파일을 쓰는 동작이 하나도 없었음). 대시보드에 "오늘
  Daily Note" 버튼을 추가하면서 처음으로 **파일을 쓰는 서버 쪽 코드**가 필요해졌다 — 정적
  파일 서버(`python3 -m http.server`)는 GET으로 파일을 내려줄 뿐 POST 요청을 처리할 수
  없기 때문이다.
- `scripts/webviewer/server.py`는 `http.server.SimpleHTTPRequestHandler`를 상속한 아주 작은
  서버로, `POST /api/create-daily-note`(이 절)와 `POST /api/import-note`(다음 절, 기존 노트
  가져오기)만 추가로 처리한다. 나머지는 표준 정적 파일 서빙 그대로다. `/api/create-daily-note`
  처리 순서: `create-daily-note.py`의 `create_daily_note()`
  호출(이미 있으면 건드리지 않음) → `generate-html.py`의 `main()`을 그대로 호출해 사이트
  전체 재생성(새 노트의 HTML 페이지가 즉시 존재하게) → `{ok, created, url}` JSON 응답.
  버튼 클릭 시 프런트(`assets/dashboard.js`)는 이 응답의 `url`로 바로 이동한다.
- 두 스크립트 모두 파일명에 하이픈이 있어(`create-daily-note.py`, `generate-html.py`)
  평범한 `import`가 안 된다 — `importlib.util.spec_from_file_location`으로 불러온다. 이때
  `sys.modules[name]`에 먼저 등록한 뒤 `exec_module`을 호출해야 하는데(등록 전에 실행하면
  `generate-html.py`의 `@dataclass`가 내부적으로 `sys.modules[cls.__module__]`을 찾다가
  `None`을 만나 터진다), 실제로 이 순서를 빠뜨렸다가 겪은 문제라 주석으로 남겨뒀다.
- **보안**: 이 서버는 쓰기 가능한 엔드포인트가 있어서 정적 파일 서빙과 달리 `127.0.0.1`에만
  바인딩한다 — 같은 Wi-Fi의 다른 기기가 내 vault에 파일을 만들게 할 수는 없어야 하므로.
- **이 버튼은 `server.py`로 띄웠을 때만 동작한다.** `scripts/preview.sh`가 내부적으로
  `python -m http.server` 대신 이 서버를 쓰도록 이미 바뀌었다. 하지만 `output/html/index.html`을
  `file://`로 직접 열거나, 표준 `http.server`로 서빙하거나(원래 안내 문구), GitHub Pages 같은
  완전 정적 배포 환경에서는 이 엔드포인트가 없어 버튼을 누르면 에러 메시지가 뜬다(다른
  기능 — 검색, 노트 열람 등 — 은 그런 환경에서도 그대로 동작한다, 이 버튼만 예외).

### "노트 가져오기" 버튼 (`POST /api/import-note`)

- 같은 서버에 두 번째 엔드포인트로 얹었다. 기존에 만들어둔 `.md`/`.markdown`/`.html`/`.htm`/
  `.txt`/`.pdf` 파일을 대시보드의 "파일 선택" 버튼으로 고르면(`<input type="file">`), 브라우저가
  `File.text()`로 내용을 읽어 `{filename, content}` JSON을 POST한다 — **멀티파트 업로드가
  아니라 파일 내용을 텍스트로 읽어 보내는 방식**을 택했다. 이유: 이 서버가 상속하는
  `http.server`에는 멀티파트 파서가 없고, 표준 `cgi.FieldStorage`는 Python 3.13에서
  제거됐다 — 텍스트만 다루면 되는 이 용도엔 `fetch` + `File.text()` + JSON이 훨씬 간단하고
  의존성도 없다. PDF는 바이너리라 `File.text()`로는 못 읽으므로 그 파일만
  `file.arrayBuffer()` → base64로 감싸 같은 JSON 바디로 보낸다(여전히 멀티파트는 안 씀).
- 서버 쪽 처리(`import_note()`): 확장자가 `.html`/`.htm`이면 `pandoc --from=html --to=markdown`으로
  변환(생성 방향의 반대 — `generate-html.py`가 이미 pandoc을 md→html에 쓰고 있어 같은
  바이너리를 재사용). 확장자가 `.pdf`면 base64를 디코딩한 뒤 poppler의
  `pdftotext -layout <tmp> -`를 서브프로세스로 호출해 텍스트를 뽑는다(pandoc은 PDF를 입력
  포맷으로 못 받아서 별도 도구가 필요했다). 제목은 변환된 마크다운/텍스트의 첫 `# ` 헤딩
  (`vault.extract_title`과 동일 로직)을 우선 쓰고, 없으면 원본 HTML의 `<title>` 태그
  (HTML만), 그것도 없으면 파일명을 쓴다. `.md`/`.markdown`/`.txt`/`.pdf`(변환 후)는 그
  이후로 같은 제목 규칙을 적용. 파일명은 `generate-html.py`의 `slugify_heading()`(헤딩
  앵커 slugify와 동일 함수 재사용)으로 제목을 슬러그화해서 만들고, `topics/{slug}.md`가
  이미 있으면 `-2`, `-3`처럼 번호를 붙여 기존 노트를 덮어쓰지 않는다. 저장 위치는 항상
  `topics/`로 고정(사용자 확인 완료 — area인지 topic인지 매번 고르게 하는 것보다 간단함을
  우선). 저장 후 `POST /api/create-daily-note`와 같은 방식으로 백링크+사이트를 통째로
  재생성하고 새 노트의 URL을 응답, 프런트는 그 URL로 바로 이동한다.
- CLI로 curl 테스트(HTML 변환, 중복 파일명 처리, 지원하지 않는 확장자 에러) + Playwright로
  실제 파일 선택 → 노트 페이지 이동 → 검색 결과에 잡히는 것까지 확인. PDF 경로는 임시
  vault + `import_note()` 직접 호출로 정상 변환·에러 처리(미지원 확장자, 손상된 base64)를
  검증(자세한 내용은 DEVLOG 2026-08-12 참고).

### PDF 원본 보관 (선택, `keep_original`)

- `.pdf`는 변환하면서 이미지·표·레이아웃이 사라지는데, 그 원본을 잃고 싶지 않을 때를 위한
  선택 기능. 범위는 `.pdf`만 — `.md`는 가져온 결과가 원본과 사실상 같고, `.html`은 git
  히스토리로 어차피 커버되거나 손실이 상대적으로 적어서 대상에서 제외했다(자세한 논의는
  DEVLOG 2026-08-12 "후속 논의" 참고).
- 저장 위치는 git 추적 밖의 새 최상위 폴더 `attachments/{노트 slug}/{원본 파일명}`
  (`.gitignore`에 등록). **처음엔 `backups/imported-originals/`를 고려했지만, `backup.sh`의
  tar 명령이 자기 자신을 참조하는 버그를 막으려고 이미 `--exclude="backups"`를 걸어두고
  있다는 걸 뒤늦게 발견** — 그대로 뒀으면 원본이 로컬에만 남고 백업/클라우드 어디에도
  절대 안 올라가는 조용한 버그가 될 뻔했다. 그래서 `backups/`와는 완전히 별개인
  `attachments/`를 새로 만들고, `scripts/backup.sh`의 tar 소스 목록에 명시적으로 추가했다.
- 업로드 시점은 **가져오는 즉시가 아니라 `backup.sh`의 기존 월간 로컬 백업(+ rclone
  설정 시 클라우드 업로드)에 편승**하는 방식을 택했다 — PRD 3.4절이 "클라우드 백업은
  승인 필요"로 이미 정해둔 원칙과, 매 가져오기마다 네트워크 업로드를 기다리게 하고 싶지
  않다는 두 가지 이유. `backup.sh`는 `attachments/`가 아직 한 번도 안 생겼어도 `tar`가
  실패하지 않도록 실행 전에 항상 `mkdir -p`로 만들어둔다.
- `import_note()`에 `keep_original: bool = False` 매개변수를 추가하고 `.pdf` 분기에서만
  `save_original_for_backup()`을 호출한다. 노트 slug가 이미 `unique_topic_path()`로
  중복 방지돼 있어서, 그 slug를 그대로 첨부 폴더 이름으로 써도 자동으로 충돌이 없다.
  `home.html`의 "노트 가져오기" 카드에 체크박스(`#import-keep-original`, 기본 꺼짐)를
  추가하고 `dashboard.js`가 그 상태를 `keep_original`로 같이 전송한다.
- **삭제와의 연동**: `delete_note()`가 `.md`만 지우고 `attachments/{slug}/`는 그대로 두면
  지운 노트의 원본이 로컬 디스크와 이후 모든 백업에 영원히 고아로 남는 문제가 있어서,
  삭제 대상이 `topics_dir` 안일 때만 매칭되는 `attachments/{stem}/`도 `shutil.rmtree`로
  같이 지우도록 함께 고쳤다.
- 검증: 임시 vault에서 `keep_original` True/False 비교, 실제 vault에서 `backup.sh` 직접
  실행 후 `tar tzf`로 `attachments/`가 아카이브에 포함되는지 확인, 실행 중이던
  `server.py`를 재시작(코드 변경은 프로세스 재시작 전까진 반영 안 됨)한 뒤 curl로
  가져오기(`keep_original: true`) → `attachments/`에 파일 생성 확인 → 삭제 →
  `attachments/`가 자동으로 없어지는 것까지 실서버로 확인. rclone은 이 환경에 아직
  설치·인증되지 않아 실제 구글드라이브 업로드 자체는 검증 못 함.

### "새 노트 작성" (`POST /api/create-note`)

- 파일 업로드(`import_note()`) 없이 제목+본문을 브라우저에서 바로 입력해 새 노트를 만드는
  경로. `import_note()`의 뒷부분(자동 태그 → 슬러그화 → 중복 처리 → 저장)을
  `_save_new_topic_note(config, title, body, logger, source_desc)`로 뽑아내 두 함수가
  공유하게 리팩터했다 — `create_note()`는 이 로직을 파일 변환 단계(pandoc/pdftotext) 없이
  그대로 재사용한다. `source_desc`는 로그 메시지 구분용("원본: {파일명}" vs "직접 입력")
  이고, `rebuild_site()`는 공유 함수 밖에서 각 호출부가 책임진다(`import_note()`는
  `keep_original` 원본 저장까지 끝난 뒤 호출해야 해서).
- UI는 처음엔 대시보드 카드(제목 `<input>` + `<textarea>` + "저장")였다가, 2026-08-12
  레이아웃 재정리 때 전용 페이지 `new-note.html`로 옮겨졌다 — 아래 "대시보드 레이아웃
  재정리" 항목 참고. `POST /api/create-note`로 `{title, content}`를 보내는 클라이언트
  로직 자체는 그대로다.
- 검증: 임시 vault에서 `create_note()` 정상 동작·빈 제목/내용 에러, 리팩터 후
  `import_note()` 회귀 여부까지 함수 직접 호출로 확인. 실제 vault·실행 중인 서버로
  Playwright로 폼 입력 → 저장 → 새 노트 페이지 이동 → 파일 내용까지 end-to-end 확인.

### 대시보드 레이아웃 재정리 (자주 안 쓰는 액션은 사이드바로)

- 사용자가 "오늘 노트 열기는 자주 안 쓴다", "새 노트 작성도 사이드바로", "원본 보관을
  윗줄로 + hover 툴팁으로", "파일 선택을 노트 가져오기 옆으로", "통계/할 일/그래프를 맨
  위로" 등 여러 메시지로 나눠 요청 — 종합해서 확인 질문 두 번(새 노트의 목적지, 최종
  순서)으로 정리한 뒤 진행.
- **"오늘 노트"**: `base.html`의 `<nav class="sidebar-nav">` 맨 아래(구분선 아래)에 버튼
  (`#daily-note-nav-btn`)으로 이동. 사이드바는 모든 페이지에 있으므로 로직도
  `dashboard.js`(홈 전용)가 아니라 새 `assets/daily-note-nav.js`로 분리해 `base.html`
  에서 전역 로드. **주의할 버그**: 서버가 돌려주는 `url`(예: `daily/2026-08-12.html`)은
  vault 루트 기준 상대경로인데, 버튼이 이제 `/tags/manage.html`처럼 깊이가 다른 페이지
  에도 있어서 그대로 쓰면 `/tags/daily/2026-08-12.html`처럼 깨진다 — `"/" + url`로 항상
  절대경로를 쓰도록 해서 해결(Playwright로 `/`와 `/tags/index.html` 양쪽에서 클릭해 확인).
- **"새 노트"**: 사이드바 `<a href="new-note.html">` → 전용 페이지. `generate-html.py`가
  `new_note_tpl.render(root="", active_nav="new-note")`로 `output_dir/new-note.html`을
  만든다. 로직은 `assets/new-note.js`(대시보드 버전과 거의 동일, 포커스 자동 이동 추가).
  전용 페이지라 textarea를 더 크게(`.new-note-textarea-tall`, `min-height: 340px`).
- **"노트 가져오기" 카드**: `.import-note-head`(flex row)로 제목+"파일 선택"+"원본 보관"
  토글을 한 줄에 배치. "원본 보관"은 체크박스+긴 설명 문구 대신, 네이티브 `title` 속성
  으로 hover 시 설명이 뜨는 아이콘 토글(`.icon-toggle` — 실제 `<input type=checkbox>`는
  `opacity:0`으로 숨기고 인접 `<span class="icon-toggle-box">`를 `:checked` 형제
  선택자로 스타일링, 체크되면 배경이 `--primary`로 바뀜)로 교체. `id="import-keep-original"`
  은 그대로 유지해서 `dashboard.js`의 기존 로직은 안 건드림.
- **순서**: 통계 카드 → 할 일 위젯 → 노트 그래프 → 노트 가져오기 → 빠른 메모 →
  Gmail/캘린더/관심종목.
- `style.css`: `button.nav-link`(버튼이 `<a>`와 동일하게 보이도록 기본 스타일 리셋),
  `.sidebar-nav-divider`, `.import-note-head`, `.icon-toggle`/`.icon-toggle-box`,
  `.new-note-textarea-tall` 추가.
- 검증: Playwright로 ① 대시보드 순서 스크린샷, ② "오늘 노트"를 대시보드·`/tags/index.html`
  양쪽에서 클릭해 절대경로 이동 확인, ③ "새 노트" → 전용 페이지에서 실제 저장 → 새 노트
  이동 확인, ④ "원본 보관" hover 툴팁·체크 시 색 변화 확인, ⑤ 체크박스를 켠 채 재배치된
  마크업에서도 실제 PDF 업로드 → `attachments/` 저장 → 삭제 시 같이 지워지는 기존 동작이
  안 깨졌는지 end-to-end 재확인, ⑥ 라이트/다크/모바일(390px) 렌더링 확인.

### 노트 편집 (`POST /api/save-note`)

- 세 번째 쓰기 엔드포인트. **daily note에만 국한하지 않고 노트 페이지(`note.html`) 전체에
  적용했다** — areas/topics/daily 어떤 노트를 열어도 우측 상단 "편집" 버튼으로 원본
  마크다운을 그 자리에서 고치고 저장할 수 있다. 별도 페이지로 안 만든 이유: note.html이
  모든 노트 타입에 공용 템플릿이라 한 군데만 고치면 전부 적용된다.
- `generate-html.py`가 노트 페이지를 렌더링할 때 `raw_source=note.text`(가공 전 원본
  마크다운)를 템플릿에 같이 넘긴다. `note.html`은 렌더링된 `.note-body`(카드 `#note-view`)와
  숨겨진 `<textarea>`(카드 `#note-editor`, 초기값이 바로 `{{ raw_source }}`) 두 개를 같이
  갖고 있다가, "편집" 버튼으로 뷰↔에디터를 토글한다. `{{ raw_source }}`를 textarea 안에
  그대로 넣어도 안전한 이유: Jinja2 autoescape가 `&`/`<`/`>`를 HTML 엔티티로 이스케이프하고,
  `<textarea>`의 콘텐츠 모델은 원래 raw text라 브라우저가 엔티티를 다시 원문자로 복원해서
  `textarea.value`에 그대로 들어간다 — 별도 JSON 인코딩이나 이스케이프 처리가 필요 없다.
- **경로 검증(`save_note()`)**: 브라우저에서 오는 `path` 문자열을 그대로 믿지 않는다.
  `vault_root / rel_path`로 절대경로를 만든 뒤, ① 확장자가 `.md`인지, ② `vault.note_dirs()`
  (areas/topics/daily)의 하위 경로인지(`Path.relative_to`로 검사)를 확인하고, 아니면 저장을
  거부한다 — `scripts/`, `config.yaml`, vault 바깥 등 어디에도 쓸 수 없다. 새 파일 생성도
  막는다(대상이 이미 존재해야 함 — 새 노트 생성은 "노트 가져오기"/daily note 버튼의 역할이지
  편집 화면의 역할이 아니다).
- Playwright로 실제 daily note를 열어 "편집" → 내용 수정 → "저장" → 새로고침 후 수정된
  내용이 그대로 반영되는 것까지 확인. 테스트 후에는 실사용자의 진짜 daily note 파일이라
  원래(빈 템플릿) 상태로 되돌려 놓았다 — 테스트 문구를 사용자의 개인 기록에 남기지 않기 위해.

### 정적 자산 캐시 비활성화 (`Handler.end_headers`)

- 사용자가 "노트 가져오기"가 안 된다고 두 번 보고했는데, 서버 로그를 보니 요청 자체가
  한 번도 도달하지 않았다 — 원인은 브라우저가 `assets/dashboard.js`를 예전 버전(가져오기
  기능이 추가되기 전)으로 캐시해두고 계속 재사용한 것으로 추정된다(`SimpleHTTPRequestHandler`는
  기본적으로 `Last-Modified`만 보내고 `Cache-Control`은 안 보내서, 브라우저가 휴리스틱
  캐싱으로 재검증 없이 오래된 버전을 계속 쓸 수 있다). `Handler.end_headers()`를 오버라이드해
  모든 응답(정적 파일 + API 응답)에 `Cache-Control: no-store, must-revalidate`를 강제한다 —
  이 vault는 스크립트를 자주 고치는 로컬 미리보기 용도라 캐시 이득보다 최신 버전 보장이
  훨씬 중요하다.

### 노트 삭제 (`POST /api/delete-note`)

- 네 번째이자 마지막 쓰기 엔드포인트. `save_note()`와 경로 검증 로직을 공유하도록
  `resolve_note_path(config, rel_path, action)`로 리팩터링했다(`action`은 에러 메시지에만
  쓰임 — "저장"/"삭제"). 검증 통과하면 `Path.unlink()`로 파일을 지우고 사이트를 재생성한다.
  **휴지통 없음** — git으로 커밋해둔 상태였다면 거기서 복구 가능하지만, 그 전이면 되돌릴
  수 없다. 그래서 프런트에서 `window.confirm()`으로 한 번 더 확인받는다.
- 삭제 버튼은 두 군데에 나온다: ① 노트 상세 페이지(`note.html`, "편집" 옆) — 삭제하면
  그 페이지 자체가 없어지므로 `data-redirect="{{ root }}vault.html"`로 노트 목록으로 이동.
  ② 노트 목록(`vault.html`/`tag.html`/`daily_archive.html`)의 각 항목 — 삭제하면 같은
  목록 페이지를 새로고침. 두 경우 모두 같은 클래스(`.note-delete-btn`, `data-path`/
  `data-title`/선택적 `data-redirect`)를 쓰고, `assets/note-actions.js` 하나가
  `document`에 이벤트 위임으로 걸려서 페이지 종류를 가리지 않고 전부 처리한다(그래서
  `note-actions.js`는 note-edit.js와 달리 `base.html`에서 전역으로 로드한다).
- `as_links()`(generate-html.py)가 이제 `path`(vault 루트 기준 `.md` 상대경로)도 같이
  반환하도록 확장 — 목록 항목의 삭제 버튼이 이 값을 `data-path`로 그대로 쓴다.
- `.note-list li`를 flex(`justify-content:space-between`)로 바꿔 제목은 왼쪽, 삭제
  버튼은 오른쪽에 붙게 했다(마우스를 올렸을 때만 보이도록 기본은 투명). 검색 결과
  목록(`search.js`가 만드는 `<li>`)도 같은 클래스를 쓰지만 삭제 버튼이 없어 태그
  span이 오른쪽 정렬되는 정도의 자연스러운 부수 효과만 있다.
- 실제로 사용자가 가져온 노트 중 같은 파일을 두 번 실수로 가져와 생긴 중복본
  (`스킬skills-...법.md`/`-법-2.md`)이 있었는데, 이 기능을 배포하자마자 사용자가 직접
  중복본 하나를 삭제하는 것을 서버 로그로 확인했다 — 실사용 검증까지 자연스럽게 됨.

### HTML 가져오기 정리(pandoc 옵션) + 태그 오탐 방지

- 실사용 중 발견: 스타일이 많이 들어간 HTML(커스텀 랜딩 페이지형 가이드 문서)을 가져오면
  ① 제목/파일명에 `` `텍스트`{style="color:#fff;..."} `` 같은 Pandoc 속성 문법이 섞이고,
  ② 그 안의 CSS 색상 값(`#FF6B35` 등)이 본문에 남아서 태그 정규식(`#([\w가-힣-]+)`)이
  진짜 해시태그로 오인하는 두 가지 문제가 실제로 발생했다(`태그` 페이지에 `#FF6B35`,
  `#004E89` 같은 게 그대로 노출됨).
  - `html_to_markdown()`의 pandoc 호출을 `--from=html-native_divs-native_spans
    --to=markdown-inline_code_attributes`로 바꿔서 해결. `-native_divs-native_spans`는
    `<div>`/`<span>`을 Pandoc의 `:::`/`{...}` 감싸기 문법으로 만들지 않고 내용만 뽑게
    하고, 마크다운 writer 쪽 `-inline_code_attributes`는 인라인 코드에 속성이 안 남게
    한다. 두 옵션을 합치면 스타일 정보가 거의 다 사라지고 순수 텍스트/서식만 남는다
    (`printf ... | pandoc --from=html-native_divs-native_spans --to=markdown-inline_code_attributes`로
    직접 재현/검증함).
  - `vault.py`의 `extract_tags()`에 `HEX_COLOR_RE`(3자리/6자리 순수 16진수) 필터를
    추가해서, pandoc 옵션을 바꾸기 전에 이미 만들어진 노트나 앞으로도 있을 수 있는
    비슷한 케이스를 이중으로 방어한다.

### 노트 가져오기 시 자동 태그 (`suggest_tags()`)

- 사용자 질문("내가 입력하면 알아서 토픽을 나누어 주나?") → 현재는 그런 기능이 없다고
  답한 뒤, 원하는 방식을 물어봐서 "가져오기 시 태그 자동 생성"으로 확정.
- **왜 로컬 휴리스틱인가**: Gmail/캘린더 스냅샷과 달리 이 기능은 사용자가 브라우저에서
  "파일 선택"을 누르는 순간 `server.py`가 그 자리에서 바로 처리해야 한다 — Claude 세션이
  옆에서 지켜보다가 개입할 수 있는 타이밍이 아니다. 그래서 Anthropic API 호출 같은 건
  쓰지 않고(이 프로젝트의 "로컬 우선" 원칙과도 맞지 않음) 순수 Python 정규식/문자열
  처리만으로 태그를 추정한다 — 완벽한 주제 분류가 아니라 "없는 것보다 낫다" 수준의
  휴리스틱임을 코드 주석에도 명시.
- 동작 순서: ① vault 전체(`iter_note_files` + `extract_tags`)에서 이미 쓰이고 있는 태그
  모음을 만들고, 새 노트 내용에 실제로 등장하는 것부터 재사용(어휘가 무한정 늘어나지
  않게). ② 그것만으로 부족하면 제목 앞부분 토큰(공백/괄호/구두점으로 분리, 최대 6개)에서
  조사를 떼어(`_JOSA_SUFFIXES`, 긴 것부터 매치) 후보를 채운다. 한국어 제목은 핵심 명사가
  앞에, 서술어가 뒤에 오는 경향이 있어서 "제목 앞부분만" 본다 — 뒤쪽까지 다 보면 동사
  활용형("살펴보는" → 조사 제거 시 "살펴보"처럼) 오염이 심해짐을 실제로 확인하고 범위를
  좁혔다. 불용어 목록(`_TAG_STOPWORDS`)으로 대명사/부사 등도 걸러낸다.
- **한계(문서화해둠, `docs/STATUS.md` "알려진 이슈")**: 조사 제거가 형태소 분석이 아니라
  규칙 기반이라 "는/은"처럼 조사와 동사 활용 어미가 겹치는 경우 여전히 어색한 후보가 나올
  수 있다(예: "다시 살펴보는" → "살펴보" 같은 동사 어간). 완벽한 태깅이 필요하면 Claude
  세션에서 노트를 보여주고 태그를 다듬어달라고 요청하는 게 낫다 — 대시보드 스냅샷과 같은
  "필요하면 Claude에게 요청" 패턴.
- 태그는 `import_note()`가 본문 저장 직전에 `#태그1 #태그2 ...` 한 줄로 파일 끝에
  추가한다(이미 본문에 있는 태그와 중복되면 제외). 실제 예시로 검증: `<title>세션 관리
  테스트</title>`만 있는 짧은 HTML을 가져왔더니 `#세션 #관리 #테스트`가 붙었고, 이미
  "세션"/"관리" 태그가 있는 상태에서 조사가 붙은 제목("세션을 다시 살펴보는 이유와...")을
  가져왔더니 기존 태그를 올바르게 재사용하는 것도 확인함.

### 대시보드 위젯 (`home.html` = `index.html`)

- **할 일 / 빠른 메모**: vault 마크다운과 완전히 분리된, 브라우저 `localStorage`에만 저장되는
  스크래치 위젯(`assets/dashboard.js`). 할 일은 날짜별(`myremember-todo-{date}`)로 따로 저장되고
  캘린더 연동은 없다(직접 추가/완료 토글/삭제/**수정**). 수정은 항목에 마우스를 올리면 나오는
  연필 아이콘을 누르면 텍스트가 인라인 `<input>`으로 바뀌고, Enter/포커스 아웃으로 저장,
  Escape로 취소한다(빈 텍스트로 저장 시도하면 기존 텍스트를 유지 — 실수로 지워지는 것 방지).
  메모는 날짜 구분 없는 단일 스크래치 메모(`myremember-quick-memo`) — "나중에 daily note/topic으로
  옮길 메모를 잠깐 적어두는" 용도라 날짜별로 쪼개지 않았다. 정식 기록은 여전히 daily note/topic
  마크다운 파일의 몫이다.
- **Gmail/캘린더/관심종목 스냅샷**: `scripts/webviewer/data/dashboard-snapshot.json`(git 추적됨,
  `output/`처럼 재생성 파생물이 아니라 입력 데이터로 취급)을 `generate-html.py`가 읽어
  `dashboard` 컨텍스트로 렌더링한다. 파일이 없으면 대시보드에 "스냅샷 없음" 빈 상태로 표시되고
  빌드는 실패하지 않는다.
  - **왜 cron으로 자동 갱신이 안 되는가**: Gmail/Google Calendar 데이터는 이 세션에 연결된 MCP
    도구로만 가져올 수 있다 — `scripts/`의 평범한 Python 스크립트에는 그 인증/도구 접근 권한이
    없다. 그래서 DailyScribe 같은 다른 크루와 달리 무인 스케줄링이 불가능하고, 갱신하려면 매번
    Claude Code 세션에서 "대시보드 스냅샷 갱신해줘"라고 요청해야 한다.
  - `scripts/build-dashboard-snapshot.py`는 데이터를 가져오는 스크립트가 아니라, 그 JSON의
    스키마가 맞는지 검증하고(`gmail.messages`/`calendar.days`/`stocks.indices`/`stocks.sectors`의
    필수 필드 확인) 처음 시작할 때 빈 틀을 만들어주는(`--init`) 보조 도구다.
  - **관심종목 데이터는 의도적으로 비워뒀다**: 실시간 시세를 가져올 MCP 도구가 없어 WebSearch로
    코스피 지수를 두 번 조회했더니 서로 모순된 값(6,258 vs 6,299 — 캐시/스냅샷 시점이 다른
    결과로 추정)이 나왔다. 개인 vault에 부정확한 금융 수치를 사실처럼 박아넣는 것보다, 비워두고
    이유를 `stocks.as_of`에 남기는 쪽을 택했다. 나중에 실제 시세 API(한국투자증권, 키움 OpenAPI
    등)를 붙이거나 사용자가 직접 수치를 불러주면 채울 수 있는 구조(`stocks.indices`/
    `stocks.sectors`)는 만들어 뒀다.

### 노트 그래프 위젯 (태그 기준 미니 네트워크 뷰)

- 물리적 폴더 재구성 없이 "분야별로 어떻게 정리돼 있는지" 보여주는 게 목표라, 이미 있는
  태그 시스템 위에 시각화 레이어만 얹었다 — `topics/`는 계속 flat 폴더 그대로다.
- **데이터**: 새 파이프라인 없이 기존 자산 재사용. `load_backlinks(config)`가 `backlinks`
  맵과 `edges` 배열을 같이 반환하도록 확장했고(유일한 호출부인 `main()`도 같이 수정),
  `NoteMeta`(제목/태그/dir_type)와 합쳐 `assets/graph-data.js`에
  `window.MYREMEMBER_GRAPH = {nodes, edges}`를 굽는다 — `search-data.js`와 동일한 패턴
  (fetch 없이 `<script>`로 즉시 로드, `file://`로 열어도 동작).
- **레이아웃**: `assets/graph-widget.js`가 외부 라이브러리 없이 순수 JS로 단순한 힘-기반
  시뮬레이션을 구현한다 — 모든 노드쌍 상호 반발 + `[[링크]]` 엣지를 따라 강한 인력 + **대표
  태그가 같은 노드끼리 약한 인력** + 중심으로의 약한 인력. 대표 태그는 그 노드의 태그들 중
  이 vault 안에서 등장 빈도가 가장 높은 것(단순히 `tags[0]`이 아님 — 노트가 흔한 태그와
  드문 태그를 같이 갖고 있을 때 흔한 태그 쪽으로 묶여야 시각적으로 의미가 있다). 매 프레임
  다시 계산하지 않고 고정 250회(노드 60개 초과 시 120회)만 미리 계산해 한 번에 렌더링 —
  "미리보기" 위젯이라 지속 애니메이션은 불필요.
- **색상**: `dataviz` 스킬의 검증된 8색 카테고리 팔레트를 `style.css`의 기존
  `rgb(var(--x))` 라이트/다크 토큰 구조에 그대로 이식(`--series-1~8`). vault 안에서 빈도
  상위 8개 태그만 고정 순서(순환 없음)로 색을 받고, 9번째 이후 태그·태그 없는 노트는
  `--muted-foreground`("기타")로 폴백. `validate_palette.js`는 인접-쌍 기준으로는 라이트/
  다크 둘 다 PASS, `--pairs all`(이 위젯처럼 임의의 두 색이 동시에 화면에 나올 수 있는
  scatter류) 기준으로는 3개 슬롯까지만 전 구간이 안전하다고 나온다 — **의도적으로 8색
  전부를 쓰기로 절충**했다. 근거: 이 위젯은 범례와 노드 hover 툴팁에 태그 이름 텍스트가
  항상 같이 있어서 색만으로 태그를 구분할 일이 없고(dataviz 스킬이 요구하는 "색-단독
  식별 금지" 원칙 자체는 지킴), 외부에 공개하는 제품이 아니라 개인 단일 사용자용 로컬
  도구라 엄격한 공개 접근성 기준까지는 맞추지 않아도 된다고 판단했다.
- 노드는 SVG `<a href="{url}">`로 감싸 클릭 시 해당 노트로 바로 이동, `<title>` 자식
  요소로 제목+태그를 네이티브 hover 툴팁으로 보여준다(별도 커스텀 툴팁 레이어 없음 —
  "미리보기" 범위에 맞춘 최소 구현). 범례/툴팁 텍스트는 `textContent`로만 채워 innerHTML
  문자열 조립을 피한다.
- `home.html`에서 통계 카드(노트/태그 개수) 바로 아래 배치 — 숫자를 본 직후 바로
  시각화가 이어지도록. 전용 "그래프" 페이지는 이번 범위 밖(사용자가 명시적으로 "대시보드
  안 작은 위젯"을 선택함)이지만, 데이터 계층(`graph-data.js`)이 이미 분리돼 있어 나중에
  전체 화면 페이지로 확장하기는 쉽다.
- 검증: `generate-html.sh` 재생성 → 서버 재시작 → Playwright로 라이트/다크모드·모바일
  (390px) 스크린샷 확인 + JS로 각 노드의 실제 `fill`/`title`/`href`를 조회해 같은 태그를
  공유하는 두 노트가 정확히 같은 색으로 묶이는지, 태그 없는 노트가 정확히 회색인지 데이터
  레벨로 검증 + 실제 클릭 시 해당 노트 페이지로 이동하는 것까지 확인.

### 태그 관리 — 관계(상위/하위) + 병합 (`POST /api/set-tag-relations`, `POST /api/merge-tags`)

- **관계 저장소**: `scripts/webviewer/data/tag-relations.json`
  (`{"parents": {자식태그: [상위태그, ...]}}`, 경로는 `config.yaml`의
  `tags.relations_path`) — `dashboard-snapshot.json`과 같은 성격(git 추적되는 입력 데이터,
  `output/`처럼 스크립트가 재생성하는 파생물이 아님). `vault.py`에
  `load_tag_relations()`/`save_tag_relations()`를 공용 헬퍼로 추가해 `server.py`(쓰기)와
  `generate-html.py`(읽기) 양쪽이 재사용한다. 태그 하나가 상위를 **여러 개** 가질 수 있는
  그래프 구조를 택했다(트리보다 유연하지만, 순환 참조를 직접 막아야 하는 대가가 있음).
- **`set_tag_parents(config, tag, parents, logger)`**: 태그 하나의 상위 목록을 통째로
  교체한다. 저장을 시도하기 전에 "만약 이 목록으로 바뀐다면"이라는 후보 상태를 만들고,
  DFS로 `tag`에서 시작해 상위를 계속 타고 올라갔을 때 다시 `tag` 자신으로 돌아오는지
  확인한다 — 순환이면 `ValueError`로 저장을 거부(파일은 안 바뀜). 트리 구조였다면 이
  검사가 필요 없었을 것(부모가 하나뿐이면 순환이 구조적으로 불가능).
- **`merge_tags(config, sources, target, logger)`**: `sources`에 속한 태그를 vault 전체
  노트 파일에서 `target`으로 치환한다. 태그 경계 정규식(`(?<!\S)#태그(?![\w가-힣-])`)으로
  `#3D`가 `#3D프린터`의 일부로 오매칭되지 않게 한다. 같은 노트 안에 치환으로 태그가 두 번
  남는 경우(`#3D`와 `#프린터`를 둘 다 `#3D프린터`로 합치면 `#3D프린터 #3D프린터`)를
  대비해, 마지막 줄이 이 프로젝트의 태그 표기 관례(순수 `#태그 #태그 ...` 형식)와
  일치하면 그 줄만 순서를 유지한 채 중복 제거한다(본문 중간의 태그 언급은 그대로 둠 —
  사용자가 쓴 글을 함부로 다시 쓰지 않기 위해). `tag-relations.json`에 소스 태그명이
  남아있으면 타겟으로 합치면서(상위 목록 union) 관계가 끊기지 않게 하고, 이 과정에서
  소스끼리 서로의 상위였던 경우 타겟이 자기 자신을 상위로 갖게 되는 자기참조도 같이
  제거한다.
- **"태그 관리" 페이지**(`tags/manage.html`, `tags_index.html` 상단 버튼으로 연결): 병합
  폼(다중 선택 소스 + 타겟 이름 입력, `<datalist>`로 기존 태그 자동완성, `window.confirm()`
  확인 후 실행 — 노트 파일이 실제로 바뀌는 만큼 기존 "삭제" 버튼과 같은 확인 패턴 적용)과
  태그별 관계 편집을 한 페이지에 담았다. 관계 편집은 **처음엔 태그마다 한 행씩,
  `<select multiple>`로 상위를 Ctrl/Cmd+클릭해서 고르는 방식**이었는데, 태그가 십수 개만
  돼도 화면이 너무 길고 멀티셀렉트 자체가 직관적이지 않다는 피드백을 받아 **태그 하나를
  고르면 그 태그의 상위 태그가 칩(pill)으로 보이고, 드롭다운+"추가" 버튼으로 하나씩
  더하거나 칩의 `×`로 하나씩 빼는 방식**으로 다시 만들었다 — 두 동작 모두 클릭 즉시
  `/api/set-tag-relations`를 호출해서 저장하고, 별도 "저장" 버튼이 없다. 이 매핑
  전체(`{태그: [상위, ...]}`)는 태그를 고르자마자 서버 왕복 없이 바로 보여줘야 해서
  `generate-html.py`가 `assets/tag-relations-data.js`(`window.MYREMEMBER_TAG_RELATIONS`)로
  통째로 구워서 넘긴다 — search-data.js/graph-data.js와 같은 패턴. 병합 폼은 그대로 뒀다
  (다중 선택이 필요한 게 맞는 동작이라 — "여러 태그를 하나로 합친다"는 개념 자체가
  다중 선택을 요구함).
- **`tag.html`에 계층 표시**: `generate-html.py`가 각 태그의 `parent_tags`/`child_tags`를
  계산해 넘긴다(`tag_relations`를 뒤집어 `tag_children` 맵도 같이 만듦). 상위 태그는
  브레드크럼에 이어서, 하위 태그는 카드 위 별도 줄에 링크 목록으로 표시.
- **그래프 위젯 재설계**: 기존엔 "대표 태그가 같은 노트끼리 서로 당기는" 노트-노트 인력
  구조였는데, 태그 자체가 1급 개체가 되어야 해서 **태그 허브 노드**를 추가했다 — 색이
  배정된 태그(상위 8개)와 관계가 설정된 태그는 전부 허브가 되고(라벨이 항상 보이는, 노트
  점보다 큰 원), 노트는 자기 대표 태그의 허브 쪽으로 끌리며(노트-노트 인력을 대체),
  하위 태그 허브는 상위 태그 허브 쪽으로 끌리고 점선으로 연결선도 그린다. 허브를 클릭하면
  해당 태그 페이지로 이동. `generate-html.py`는 `graph-data.js`에 `tagParents` 필드
  하나만 추가하고, 물리 시뮬레이션·렌더링은 전부 `graph-widget.js` 안에서 처리한다(새
  데이터 파이프라인 없음).
- 검증: 임시 vault에서 `merge_tags()`(정확 토큰 매칭, 태그 줄 중복 정리)·`set_tag_parents()`
  (정상 저장 + 순환 거부) 직접 호출로 확인. 실제 vault·실행 중인 서버로 ① 태그 관리
  페이지 렌더링과 실제 폼 제출(관계 저장 → `tag-relations.json` 파일과 태그 페이지 상위/
  하위 표시 확인), ② 디스포저블 테스트 노트로 병합 API를 curl로 호출해 파일이 정확히
  바뀌는지, ③ Playwright로 그래프 위젯의 허브+점선이 라이트/다크/모바일에서 정상
  렌더링되고 허브 클릭 시 태그 페이지로 이동하는지까지 확인. 테스트에 쓴 관계/병합/노트는
  전부 원상 복구.

### VersionKeeper 상세 동작 (`git-auto-commit.sh`)

- 변경사항이 있으면(`git status --porcelain`) `git add -A` + `Daily update: {YYYY-MM-DD}` 커밋. 변경이
  없으면 커밋을 건너뛴다(멱등, cron으로 매일 돌려도 안전).
- 월간 태그(`v{YYYY-MM}`)는 "매달 1일에만 실행"이 아니라 "이번 달 태그가 로컬에 아직 없으면 생성"
  조건으로 만든다 — 스케줄이 하루 밀려도 그 달의 태그가 누락되지 않는다.
- **push는 기본적으로 하지 않는다.** `--push`를 줘야 시도하고, 그마저도 대화형 터미널(tty)에서는
  `y/N` 확인을 한 번 더 받는다. tty가 없는 실행(cron 등)에서는 `--yes`를 명시하지 않는 한 push를
  건너뛴다 — PRD 3.4절이 git push를 "승인 필요" 고위험 작업으로 분류하기 때문에, 무인 실행이
  실수로 원격 저장소를 건드리지 않도록 막는 안전장치다.
- 태그 push가 실패해도(이미 원격에 같은 태그가 있는 경우 등) 커밋 push 자체는 이미 끝난
  상태이므로 스크립트를 실패시키지 않고 로그만 남긴다.

### BackupKeeper 상세 동작 (`backup.sh`)

- 로컬 압축 백업(`backups/{YYYY-MM}.tar.gz`)은 vault 밖으로 나가지 않는 저위험 작업이라 매번
  자동으로 만든다. 대상은 원본(`areas/topics/daily`)과 그걸 만드는 `scripts/`, 루트 문서들이며,
  재생성 가능한 `output/`과 `backups/` 자신, `.git`은 제외한다.
- 분기 시작(1·4·7·10월 1일)에 실행되면 PRD 3.2절의 "외장 HDD 백업" 규칙을 로그로 상기시킨다
  (자동화하지 않고 사람이 챙겨야 하는 일이라 리마인더만).
- 클라우드 업로드는 `rclone` 설치 여부 → `config.yaml`의 `backup.rclone_remote` 설정 여부 →
  `rclone listremotes`로 그 remote가 실제 구성돼 있는지를 순서대로 확인하고, 하나라도 없으면
  이유를 로그로 남기고 조용히 스킵한다(스크립트를 실패시키지 않음 — 클라우드 백업은 선택 사항).
- 모든 조건이 갖춰졌을 때만 `rclone copy`를 실행하며, VersionKeeper의 push와 동일한 패턴으로
  대화형 확인(또는 `--yes`)을 거친다 — 클라우드 업로드도 PRD 3.4절의 승인 필요 작업이다.
- `rclone`은 아직 이 환경에 설치되지 않았고, `rclone config`의 Google 계정 OAuth 인증은 브라우저가
  필요해 사용자가 직접 완료해야 한다 — 그래서 이 스크립트는 "설정돼 있으면 쓰고, 아니면 안내만
  하고 넘어간다"는 방어적인 구조로 짰다.

### GitHub Pages 배포는 보류함

PRD는 GitHub Pages 배포를 요구하지만, 사용자와 상의 후 지금은 배포하지 않기로 했다.
이유: GitHub Free 플랜에서는 저장소가 private이어도 Pages 사이트 자체는 URL을 아는 누구나
접근 가능한 public이 된다 — 이 vault에는 일상/프로젝트 메모 같은 개인적인 내용이 쌓일
예정이라 무심코 공개되면 곤란하다. 지금은 `output/html`을 로컬에서 열거나
`python3 -m http.server --directory output/html`로 로컬 서빙해서 쓰고, 나중에 실제로
"모바일에서도 보고 싶다"는 필요가 생기면 그때 공개 배포 여부(또는 GitHub Pro의 private
Pages, 혹은 Tailscale 같은 사설 접근 방식)를 다시 결정한다.

## 승인이 필요한 작업 (PRD 3.4절)

git 커밋/푸시(`VersionKeeper`)와 클라우드 백업(`BackupKeeper`)은 스크립트 차원에서 실행 전
확인을 거치도록 구현했다(위 두 절 참고 — 기본은 로컬 동작만 하고, 원격에 손대는 부분은
`--push`/클라우드 업로드 확인 프롬프트, 비대화형 실행은 `--yes` 없이는 건너뜀). 저위험 작업
(`DailyScribe`, `LinkValidator`, `SearchIndexer`, `HtmlPublisher`, 로컬 백업)은 cron으로 승인
없이 자동 실행 가능하도록 설계한다. `HtmlPublisher`의 결과물을 인터넷에 공개하는 것(GitHub
Pages 등 배포)도 위에서 보듯 별도의 승인이 필요한 고위험 작업으로 취급한다.

cron 등록, 실제 `--push`/클라우드 업로드 실행, `rclone config`(Google 계정 OAuth, 브라우저
필요)는 아직 사용자가 진행하지 않았다 — 스크립트는 준비돼 있고 실행만 남은 상태.

## 외국어 학습 모듈 (2026-11-01 기준)

웹뷰어(`scripts/webviewer/server.py`, 127.0.0.1)에 붙은 로컬 전용 기능이라 클라우드(readonly) 빌드에는
공부법 페이지·메뉴·`study.js`를 넣지 않는다.

| 모듈 | 역할 |
|---|---|
| `scripts/language_tts.py` | 학습자료(`areas/language/YYYY-MM-DD.md`) 파싱, 음성 생성(Qwen/Mac say), 오디오·manifest 관리(`attachments/language-audio/`, git 제외), 로컬 TTS 서버 기동·`model_dir` 불일치 시 재시작 |
| `scripts/tts-server/qwen_server.py` | Qwen3-TTS 0.6B 로컬 서버(127.0.0.1:8091). 외장 하드 `SST/model`의 스냅샷을 경로로 직접 로드(오프라인), `/health`(`busy`/`queued`/`model_dir`), `[qwen]` 로그 |
| `scripts/language_study.py` | 공부법 프로필·복습 카드·간격 계산·기록. 저장은 `private/language-study/`(git 제외). 서버는 `POST /api/study`(action 디스패치)로만 접근 |
| `static/voice-ui.js` | 언어별 모델·화자·말투 UI, 기본값(영어=say, 중·일=Qwen 모국어 화자, 사용자 지정 기본), "＋복습" 버튼, 화자 설명 |
| `static/language.js` · `study.js` · `qwen-monitor.js` | 외국어 듣기 화면(언어별 날짜 복원, 다중 파일 등록) · 공부법 화면 · Qwen 생성 모니터 팝업 |

데이터 흐름: 학습자료 등록 → 음성 생성(`/api/language-save`) → 화면에서 "＋복습"/날짜별 담기(`/api/study`
add-cards·add-day) → 오늘 복습(`queue`/`rate`)이 카드 상태(`interval`·`due`·`step`)와 로그를 갱신.
설계 근거와 한계는 `docs/PRD-language-study-methods.md`.
