# 개발 현황 (STATUS)

> 기준일: 2026-08-11 — 이 문서는 append하지 않고 매번 "현재 기준"으로 덮어써서 갱신한다. 히스토리는 DEVLOG.md에 남긴다.

## ✅ 완료 (Phase 1~3)

- vault 폴더 구조(`areas/`, `topics/`, `daily/`, `scripts/`, `output/`, `backups/`) 및 `.gitignore`
- `scripts/vault.py`: 공통 config 로더 + 로거 + 노트 순회/제목·태그 추출/코드블록 제거 유틸 (모든 자동화 스크립트가 공유)
- `scripts/create-daily-note.py` (DailyScribe): 오늘 daily note 생성, 중복 생성 방지(멱등), `daily/index.md` 월별 인덱스 자동 재생성
- `scripts/validate-links.py` (LinkValidator): `[[위키링크]]` 무결성 검사, 코드블록 예외 처리, 모호한 링크(동명 파일) 감지, `output/link-report.md` 리포트, 깨진 링크 시 exit code 1
- `scripts/build-search-index.py` (SearchIndexer): SQLite FTS5로 제목/본문/태그 전문 인덱싱(`output/search.db`), `--query`로 검색(phrase 쿼리로 안전 처리)
- `scripts/build-backlinks.py` (BacklinkMapper): `[[링크]]` 역참조 그래프(`output/backlinks.json`, nodes/edges/backlinks). 노드 id는 경로 기반이라 `areas/*/README.md` 동명 파일도 안전하게 구분됨
- `scripts/search.sh`: ripgrep(+fzf) 인터랙티브 검색. fzf 없으면 ripgrep 결과만 출력하는 폴백 포함
- `scripts/generate-html.py` + `generate-html.sh` (HtmlPublisher): Pandoc+Jinja2로 `output/html`에 정적 웹뷰어 생성 — 홈/노트/태그/검색(클라이언트 사이드)/다크모드/반응형
- `scripts/preview.sh`: 백링크 갱신 + 웹뷰어 생성 + 로컬 서버 실행을 한 번에 하는 편의 스크립트
- 링크 해석 규칙 개선: `areas/{project}/README.md`를 `[[project-name]]`(폴더명)으로도 찾을 수 있도록 `vault.note_lookup_keys` 추가
- 소스 저장소(`peter-cho-70/myremember`, private) 생성 및 GitHub 푸시

## 🚧 진행 중

- **Phase 4 (버전 & 백업)**: `git-auto-commit.sh`(커밋+월간 태그+push), `backup.sh`(로컬 압축 백업 + rclone 연동)를
  스크립트 형태로 작성 중. PRD상 이 둘은 "승인 필요" 고위험 작업으로 분류되어, 실제 push/클라우드 업로드
  실행과 rclone 설정(Google 계정 OAuth, 사용자가 직접 브라우저에서 완료해야 함)은 이후 세션에서 진행.

## 🐞 알려진 이슈

- `generate-html.py`의 헤딩 링크(`[[note#heading]]`) 앵커는 자체 slugify라 Pandoc의
  `auto_identifiers`와 100% 동일한 알고리즘은 아니다. 테스트한 케이스(한글 헤딩)는 일치했지만
  엣지 케이스(중복 헤딩, 특수문자 등)는 다를 수 있음.
- `search.sh`의 fzf 인터랙티브 경로(프리뷰 창 등)는 개발 환경에 fzf가 없어 직접 확인 못 함 —
  ripgrep 폴백 경로는 정상 동작 확인.

## 🗺️ 다음 우선순위

- **GitHub Pages 배포는 보류 중** — Free 플랜에서는 private 저장소여도 Pages 사이트 자체는
  public이 되기 때문에, vault에 개인적인 내용이 쌓이는 걸 감안해 사용자와 상의 후 미룸.
  지금은 `python3 -m http.server --directory output/html`로 로컬에서만 사용. (이 문서 사이트
  자체는 vault 콘텐츠가 아니라 PRD/개발 기록만 담고 있어 별도로 공개함.)
- PRD Phase 1 남은 항목: cron 잡 등록 (사용자가 아직 원하지 않아 보류)
- PRD Phase 4 완료 후: `git-auto-commit.sh`/`backup.sh`의 실제 실행·rclone 설정
- 실제 콘텐츠가 쌓이면 `build-search-index.py`/`build-backlinks.py`/`generate-html.sh`/`validate-links.py`를 주기적으로 돌려 최신 상태 유지
