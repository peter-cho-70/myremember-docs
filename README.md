# MyRemember — 문서

로컬 우선(Local-First) 개인용 지식 관리 시스템. Markdown 파일(`areas/` 프로젝트, `topics/` 주제,
`daily/` 일일 노트)로 이루어진 vault를 Python/Bash 자동화 스크립트가 관리한다 —
daily note 자동 생성, `[[위키링크]]` 무결성 검사, SQLite 전문검색 인덱스, 역참조(backlink)
그래프, Pandoc+Jinja2 기반 정적 웹뷰어까지 전부 로컬에서 돈다.

## 핵심 기능

- **DailyScribe**: 매일 daily note 자동 생성 + 월별 인덱스 재생성 (멱등)
- **LinkValidator**: `[[노트]]` 링크 무결성 검사, 깨진/모호한 링크 리포트
- **SearchIndexer / BacklinkMapper**: SQLite FTS5 전문검색 + `[[링크]]` 역참조 그래프(JSON)
- **HtmlPublisher**: Pandoc+Jinja2로 홈/노트/태그/검색/다크모드를 갖춘 정적 웹뷰어 생성
- (진행 중) **VersionKeeper / BackupKeeper**: git 자동 커밋·태그·push, 로컬 압축 백업 + rclone 클라우드 백업

Argo(AI 에이전트 플랫폼)의 "Crew" 개념에서 영감을 받아, 각 자동화 스크립트를 독립된 역할을
가진 에이전트로 설계했다. 자세한 배경은 [PRD](doc/spec/PRD.md) 참고.

## 문서

- [개발 현황 (STATUS)](doc/STATUS.md) — 완료된 기능, 진행 중인 작업, 알려진 이슈
- [개발 기록 (DEVLOG)](doc/DEVLOG.md) — Phase별 타임라인: 무엇을 만들었고, 어떤 버그를 어떻게 고쳤는지
- [PRD](doc/spec/PRD.md) — 제품 기획: 비전, 요구사항, Phase별 로드맵
- [ARCHITECTURE](doc/spec/ARCHITECTURE.md) — 실제 구현 현황: 공통 모듈, 각 스크립트 상세 동작, 설계 원칙

소스 코드는 별도의 비공개 저장소(`peter-cho-70/myremember`)에 있습니다 — vault에 개인적인
프로젝트/일상 메모가 쌓이기 때문에 소스는 private으로 유지하고, 이 문서 사이트만 공개합니다.
