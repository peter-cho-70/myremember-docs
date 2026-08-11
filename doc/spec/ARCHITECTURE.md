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
