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
