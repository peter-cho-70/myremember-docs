# 개발 현황 (STATUS)

> 기준일: 2026-08-15 — 이 문서는 append하지 않고 매번 "현재 기준"으로 덮어써서 갱신한다. 히스토리는 DEVLOG.md에 남긴다.

## ✅ 완료

- vault 폴더 구조 (`areas/`, `topics/`, `daily/`, `scripts/`, `output/`, `backups/`) 및 `.gitignore`
- `scripts/vault.py`: 공통 config 로더 + 로거 + 노트 순회/제목·태그 추출/코드블록 제거 유틸 (모든 자동화 스크립트가 공유)
- `scripts/create-daily-note.py` (DailyScribe): 오늘 daily note 생성, 중복 생성 방지(멱등), `daily/index.md` 월별 인덱스 자동 재생성
- `scripts/validate-links.py` (LinkValidator): `[[위키링크]]` 무결성 검사, 코드블록 예외 처리, 모호한 링크(동명 파일) 감지, `output/link-report.md` 리포트, 깨진 링크 시 exit code 1
- `scripts/build-search-index.py` (SearchIndexer): SQLite FTS5로 제목/본문/태그 전문 인덱싱(`output/search.db`), `--query`로 검색(phrase 쿼리로 안전 처리)
- `scripts/build-backlinks.py` (BacklinkMapper): `[[링크]]` 역참조 그래프(`output/backlinks.json`, nodes/edges/backlinks). 노드 id는 경로 기반이라 `areas/*/README.md` 동명 파일도 안전하게 구분됨
- `scripts/search.sh`: ripgrep(+fzf) 인터랙티브 검색. fzf 없으면 ripgrep 결과만 출력하는 폴백 포함
- `scripts/generate-html.py` + `generate-html.sh` (HtmlPublisher): Pandoc+Jinja2로 `output/html`에 정적 웹뷰어 생성 — 홈/노트/태그/검색(클라이언트 사이드)/다크모드/반응형. Playwright로 실제 브라우저 렌더링·다크모드·검색·모바일 뷰까지 확인함
- `scripts/preview.sh`: 백링크 갱신 + 웹뷰어 생성 + 로컬 서버 실행을 한 번에 하는 편의 스크립트 (`scripts/preview.sh [포트]`)
- 링크 해석 규칙 개선: `areas/{project}/README.md`를 `[[project-name]]`(폴더명)으로도 찾을 수 있도록 `vault.note_lookup_keys` 추가 (validate-links/build-backlinks/generate-html 세 스크립트 모두 반영)
- 루트 `README.md`, `ARCHITECTURE.md` 작성 (Phase 1+2+3 반영)
- GitHub 저장소 생성 및 푸시: https://github.com/peter-cho-70/myremember (private)
- `scripts/git-auto-commit.sh` (VersionKeeper): 변경사항 커밋 + 월간 태그(`v{YYYY-MM}`, idempotent).
  push는 기본 비활성 — `--push`로만 시도하고, 대화형 터미널이면 확인 프롬프트, 비대화형이면
  `--yes` 없이는 건너뜀 (PRD 3.4절 "승인 필요" 반영)
- `scripts/backup.sh` (BackupKeeper): 로컬 압축 백업(`backups/{YYYY-MM}.tar.gz`) 자동 생성 +
  rclone remote가 `config.yaml`에 설정돼 있으면 확인 후 클라우드 업로드(없으면 안내하고 스킵).
  분기 시작(1/4/7/10월 1일)에는 외장 HDD 백업 리마인더 로그
- 공개 문서 사이트 `myremember-docs`(https://peter-cho-70.github.io/myremember-docs/) 발행 —
  PRD/ARCHITECTURE/STATUS/DEVLOG/사용설명서만 공개, vault 콘텐츠가 있는 소스 저장소는 계속 private 유지.
  허브(`peter-cho-70.github.io`)에도 카드 등록 완료
- `docs/guide/USER_GUIDE.md` 사용설명서 작성 + 공개 문서 사이트에 배포
- **웹뷰어 UI 전면 리디자인**: 사용자가 준 참고 대시보드 HTML(Tailwind 기반 사이드바+카드
  스타일)을 순수 CSS로 이식(외부 CDN 없이 오프라인 사용 유지). `index.html`(대시보드)과
  `vault.html`(기존 홈 콘텐츠: areas/topics/daily 목록)로 페이지 역할 분리, 모바일 오프캔버스
  사이드바(`assets/nav.js`) 추가. Playwright로 라이트/다크모드, 데스크톱/모바일(390px), 노트·
  태그·검색 페이지, todo/메모 상호작용까지 확인함
- **대시보드 위젯**: 할 일 체크리스트(추가/완료 토글/수정/삭제) + 빠른 메모(둘 다 브라우저
  localStorage가 원본, vault 마크다운과 무관, `assets/dashboard.js`). 할 일 수정은 항목의 연필
  아이콘 → 인라인 입력창으로 텍스트 변경(Enter 저장/Escape 취소). **(2026-08-12 갱신)
  "매일" 반복 옵션**(추가 시 체크박스, 별도 템플릿 목록 `myremember-todo-recurring-
  templates`로 관리) **+ 미완료 일반 항목의 다음날 자동 이월**(그날 화면을 열 때 전날의
  미완료 항목을 그 날짜 배열에 복제, `carriedFrom` 필드로 추적) 추가 — 반복/이월 항목은
  제목 옆에 각각 "매일"/"이월" 배지(`.todo-badge`)로 구분 표시. 하루씩만 보고 넘기는
  방식이라 여러 날을 한 번도 안 열어보고 건너뛰면 그 사이는 안 이어짐(정적 localStorage
  라 백그라운드 처리가 없는 데서 오는 의도적 단순화). 자세한 내용은 DEVLOG 2026-08-12
  "할 일 위젯: '매일' 반복 옵션 + 미완료 항목 자동 이월" 참고. **(2026-08-13 갱신) 기기 간
  보기 동기화**: localStorage는 브라우저(기기)별로 완전히 갈라져 있어서, 배포된 사이트를
  모바일에서 열면 로컬 PC에서 넣은 할 일이 전혀 안 보이는 문제가 있었다 — 로컬 서버로
  열었을 때는 저장할 때마다 새 API `POST /api/save-todos`로 전체 상태를
  `scripts/webviewer/data/todos.json`에도 미러링해두고, `generate-html.py`가 사이트를
  만들 때마다 그걸 `assets/todos-data.js`(`window.MYREMEMBER_TODOS`, search-data.js와
  같은 패턴)로 구워 넣는다. 이 스냅샷은 "그 기기에 그 날짜 로컬 기록이 아예 없을 때"만
  초기값으로 쓰이고, 그 뒤로는 그 기기의 localStorage가 그대로 이어받는다 — 모바일에서
  로컬 할 일을 볼 수는 있지만, 모바일 쪽에서 체크/수정해도 PC로 다시 동기화되진 않는다
  (뷰어 성격, 양방향 동기화 아님). 체크박스 클릭마다 사이트 전체(pandoc 재변환 포함)를
  다시 만드는 건 낭비라 이 API는 사이트를 재생성하지 않고, 다음 노트 저장/배포 시점에
  자연스럽게 반영된다. 자세한 내용은 DEVLOG 2026-08-13 "'할 일' 위젯 기기 간 보기 동기화"
  참고.
- **"오늘 Daily Note" 버튼 + `scripts/webviewer/server.py`**: 웹뷰어가 처음으로 파일을 쓰는
  서버 쪽 코드를 갖게 됨 — 정적 파일 서버(`http.server`)로는 버튼 클릭(POST) 처리가
  불가능해서, `SimpleHTTPRequestHandler`를 상속한 작은 서버를 새로 만들고 `preview.sh`가
  이걸 쓰도록 변경(127.0.0.1에만 바인딩). 버튼을 누르면 `create-daily-note.py`의 로직을
  호출해 없으면 만들고, `generate-html.py`를 재실행해 사이트를 통째로 재생성한 뒤 새/기존
  노트 페이지로 이동한다. 이 버튼은 `server.py`로 띄웠을 때만 동작(정적 배포·`file://`·
  일반 `http.server`에서는 에러 메시지만 표시, 다른 기능은 그대로 동작). **(2026-08-12
  갱신) 버튼 자체는 더 이상 대시보드 카드가 아니라 사이드바 "오늘 노트" 항목**(모든
  페이지에서 보임, `assets/daily-note-nav.js`) — 자세한 내용은 아래 "대시보드 레이아웃
  재정리" 항목 참고.
- **"노트 가져오기" 버튼**: 같은 서버에 `POST /api/import-note` 추가. 기존 `.md`/`.html`
  파일을 대시보드에서 골라서 `topics/`에 노트로 등록(HTML은 pandoc으로 마크다운 변환,
  제목은 첫 헤딩/`<title>`/파일명 순으로 추정, 파일명 중복 시 `-2`/`-3` 자동 부여). 저장
  후 백링크+사이트 재생성까지 자동으로 이어짐. curl로 HTML 변환/중복 처리/미지원 확장자
  에러 확인 + Playwright로 실제 파일 선택 → 노트 이동 → 검색 반영까지 확인
- **노트 편집 버튼**: 세 번째 쓰기 엔드포인트 `POST /api/save-note`. daily뿐 아니라
  areas/topics/daily 모든 노트 페이지에서 "편집" → 원본 마크다운 수정 → "저장"이 가능.
  경로는 `vault.note_dirs()` 하위인지 + 확장자가 `.md`인지 검증해서 vault 밖 파일은 절대
  못 쓰게 막음. Playwright로 실제 daily note 편집·저장·새로고침 반영까지 확인(테스트 후
  사용자의 실제 daily note는 원상복구함)
- **정적 자산 캐시 비활성화**: 사용자가 "노트 가져오기"가 안 된다고 두 번 신고 → 서버
  로그에 요청 자체가 도달한 기록이 없어 브라우저의 오래된 `dashboard.js` 캐싱을 의심 →
  `Handler.end_headers()`에서 모든 응답에 `Cache-Control: no-store, must-revalidate` 강제.
  이후 사용자가 강력 새로고침 후 실제로 노트 5개를 성공적으로 가져옴(문제 해결 확인됨)
- **노트 삭제 버튼**: 네 번째 쓰기 엔드포인트 `POST /api/delete-note`. 노트 상세 페이지의
  "삭제"(누르면 노트 목록으로 이동)와 노트 목록(노트/태그/Daily 아카이브)의 각 항목별
  삭제 버튼 두 군데에서 다 동작. `window.confirm()`으로 한 번 확인 후 삭제(휴지통 없음,
  git 커밋 상태였다면만 복구 가능). 배포 직후 사용자가 실수로 중복 가져오기한 노트를
  실제로 이 기능으로 직접 삭제하는 것까지 서버 로그로 확인됨. 목록의 삭제 버튼은 항상
  숨겨져 있다가 그 줄에 마우스를 올렸을 때만 나타남(Playwright로 기본 숨김/hover 시
  노출 둘 다 스크린샷으로 확인)
- **HTML 가져오기 정리 + 태그 오탐 수정**: 스타일 많은 HTML을 가져오면 pandoc이
  `` `텍스트`{style="color:#fff;..."} `` 같은 속성 문법을 남기고, 그 안의 CSS 색상값이
  `#태그`로 오인식되는 문제(실제로 `태그` 페이지에 `#FF6B35` 등이 노출됨)를 발견 →
  `html_to_markdown()`을 `--from=html-native_divs-native_spans
  --to=markdown-inline_code_attributes`로 바꿔 근본 해결 + `vault.extract_tags()`에
  3/6자리 순수 16진수 필터 추가(이중 방어)
- **노트 가져오기 시 태그 자동 생성**: "내가 입력하면 알아서 토픽을 나눠주나?" 질문에
  답한 뒤 "가져오기 시 태그 자동 생성"으로 방향 확정. Claude 세션 개입 없이 `server.py`가
  그 자리에서 바로 처리해야 해서 API 호출 없는 로컬 휴리스틱으로 구현 —
  ① vault에 이미 있는 태그 중 새 노트에 실제로 등장하는 것 우선 재사용, ② 부족하면
  제목 앞부분 토큰에서 조사를 뗀 후보로 보완(한국어 제목은 명사가 앞, 서술어가 뒤라는
  경향을 이용해 범위를 좁힘). 실제 예시로 검증(짧은 제목엔 깔끔한 태그, 기존 태그
  재사용도 정상 동작 확인)
- **Gmail/캘린더 스냅샷**: MCP 도구로 실제 데이터를 가져와
  `scripts/webviewer/data/dashboard-snapshot.json`에 채워넣고 대시보드에 렌더링(받은편지함
  3645통·안읽음 2503통). `scripts/build-dashboard-snapshot.py`로 스키마
  검증. cron 자동화는 불가(Gmail/Calendar MCP 접근은 Claude 세션에서만 가능) — 갱신하려면
  매번 "대시보드 스냅샷 갱신해줘"라고 요청해야 함. **(2026-08-12) Gmail 탭에 새로고침
  안내 아이콘 추가**(hover 시 "Claude에게 갱신 요청" 툴팁 — 실시간 재조회는 Gmail MCP가
  Claude 세션 전용이라 로컬 버튼 하나로는 불가능해서 안내로 대체) **+ 메일 행을 클릭하면
  실제 Gmail 웹의 그 메일로 바로 이동**(`gmail.messages[].url`, `#all/{thread_id}`
  링크 — 본문 전체를 우리 사이트에 올리지 않고 실제 Gmail로 위임). 카운트는
  `list_labels`의 INBOX `threadsTotal`/`threadsUnread`를 씀(`search_threads`의
  `resultCountEstimate`는 쿼리와 무관하게 고정값이라 못 쓴다는 걸 실측으로 확인).
  자세한 내용은 DEVLOG 2026-08-12 "Gmail 새로고침 안내 + 메일 클릭해서 열기" 참고.
  **(2026-08-12 갱신) 캘린더는 기본
  캘린더 하나가 아니라 `list_calendars`로 접근 가능한 캘린더 전체(현재 11개)를 순회해서
  합친다** — 각 이벤트에 `calendar` 필드로 출처 캘린더 이름을 남길 수는 있지만
  **(2026-08-12 세 번째 갱신) "캘린더 종류는 표시하지 말아달라"는 요청으로 화면에서
  배지를 뺐다** — 시간+제목만 보임. **범위는 처음엔 "이번 주"였다가(2026-08-12 두 번째 갱신)
  사용자 요청으로 "오늘 기준 3일(오늘+다음 2일)"로 좁혔다** — **(2026-08-13 갱신) "내
  캘린더"는 굵게 표시**: 사용자 본인 캘린더(`ubsoldboy@gmail.com` = "조충남 개인일정")에서
  온 일정에만 이벤트마다 `mine: true`를 추가해 `home.html`에서 `.event-title-mine`
  (font-weight 700)으로 다른 사람/조직 캘린더 일정과 구분되게 표시. 같은 요청으로 스냅샷도
  실제 오늘(08/13) 기준으로 다시 가져와 08/12에 멈춰있던 날짜를 갱신함(스냅샷은 세션 시점
  기준이라 시간이 지나면 자연히 낡는다 — 매번 새로 요청해야 하는 한계는 동일). 현재 스냅샷
  기준 08/13~08/15, 4개 캘린더(조충남 개인일정·보도기술부 일정·KOBETA·대한민국의 휴일)에
  실제 일정 있음.
  다음에 "대시보드 스냅샷 갱신해줘"라고 요청받는 세션도 이 절차와 범위
  (`build-dashboard-snapshot.py`의 안내 주석)를 따르면 된다.
- **관심종목(주식)은 의도적으로 비워둠**: 실시간 시세 API가 없어 WebSearch로 코스피 지수를
  두 번 조회했더니 서로 다른 값(6,258 vs 6,299)이 나와 신뢰할 수 없었음 — 부정확한 금융 수치를
  넣지 않고 이유를 스냅샷에 남김. `stocks.indices`/`stocks.sectors` 스키마는 준비돼 있어
  나중에 실제 시세 API를 붙이거나 값을 채워주면 바로 표시됨
- **pandoc 헤딩+목록 인접 버그 발견 및 수정**: pandoc 3.10 계열에서 `## 헤딩` 바로 다음 줄에
  빈 줄 없이 `-` 목록이 오면(정확히 `daily_note.template`의 패턴) 헤딩 텍스트에 `#`이 남는
  파싱 버그를 발견 → `generate-html.py`에서 pandoc에 넘기기 전에 헤딩-목록 사이에 빈 줄을
  자동 삽입해 우회 (원본 `.md` 파일은 그대로 둠)
- 서비스 기본 포트를 8000 → 4500으로 변경 (`preview.sh`, `generate-html.py` 안내 로그,
  README/사용설명서 반영). 처음엔 5000으로 바꿨다가 이 macOS 환경의 ControlCenter(AirPlay
  수신)가 5000번을 이미 쓰고 있어 바인딩이 실패하는 걸 확인 → 4500으로 재변경(확인 시점
  기준 이 환경에서 비어있음)
- **"노트 가져오기"에 PDF 지원 추가**: `.md`/`.html`뿐이던 지원 확장자에 `.pdf` 추가.
  pandoc은 PDF를 입력 포맷으로 못 받아서 poppler의 `pdftotext -layout`을 서브프로세스로
  호출해 텍스트만 뽑아내는 방식(brew로 이미 설치돼 있던 `pdftotext` 재사용, 새 의존성
  추가 없음). 브라우저는 PDF를 바이너리로 읽어 base64로 감싸 JSON으로 보내고(`file.text()`로는
  못 읽으므로), 서버가 디코딩 후 임시 파일에 써서 `pdftotext`에 넘긴다. 폼피드(페이지 구분
  문자)는 빈 줄로, 3줄 이상 연속 빈 줄은 2줄로 정리. 텍스트 레이어가 없는 스캔 이미지 PDF는
  추출 결과가 비어 명확한 에러로 실패한다. 요청 크기 제한을 5MB→20MB로 상향(base64 오버헤드
  ~1.37배 감안). 임시 vault로 정상 변환/제목·태그 추출/사이트 재생성과, 미지원 확장자·손상된
  base64 두 에러 경로까지 스크립트로 검증 완료.
- **"노트 가져오기"에 PDF 원본 보관(선택) 추가**: PDF는 변환하면 이미지·표·레이아웃이
  사라지는데, 사용자가 "AI로 정리한 자료를 PDF로도 보관하고 싶은데 원본을 잃고 싶지는
  않다"고 해서 체크박스로 선택 가능하게 함(기본은 꺼짐 — md/html은 변환 손실이 없거나
  git 히스토리로 이미 커버돼서 대상에서 제외, PDF만 해당). 체크하고 가져오면 원본 PDF가
  git 추적 밖의 새 최상위 폴더 `attachments/{노트 slug}/`에 저장된다. **가져오는 순간
  클라우드에 바로 올리지 않고**, 이미 있는 `scripts/backup.sh`의 월간 로컬 백업
  (`backups/{YYYY-MM}.tar.gz`)에 `attachments`를 새로 포함시켜서, rclone이 설정돼 있으면
  기존의 "승인 후 클라우드 백업" 흐름을 그대로 타고 구글드라이브까지 올라가게 했다(PRD
  3.4절의 "클라우드 백업은 승인 필요"와 일치 — 가져오기마다 즉시 업로드하는 방식은 이
  원칙과 맞지 않아 배제함). `backup.sh`는 `attachments/`가 없어도 `tar` 명령이 실패하지
  않도록 실행 전에 항상 `mkdir -p`로 만들어둔다. 노트를 "삭제"하면 매칭되는
  `attachments/{slug}/`도 자동으로 같이 지워져서 고아 파일이 안 남는다(topics/ 노트일
  때만 — areas/daily는 이 기능의 대상이 아님). 참고: 이 환경엔 아직 rclone이 설치·인증
  안 돼 있어서(Phase 4부터 이어지는 미해결 항목) 실제 업로드까지는 확인 못 했고, 로컬
  단계(체크박스 on/off 두 경우 모두, 삭제 연동, `backup.sh`가 만든 tar.gz 안에
  `attachments/`가 실제로 포함되는지)까지만 스크립트+실서버로 검증함.
- **대시보드에 "노트 그래프" 위젯 추가(태그 기준 미리보기)**: 사용자가 "노트개수는 보이는데
  분야별로 뭐가 들어있는지 옵시디언처럼 체계적으로 보고 싶다"고 요청 → 배치(전용 페이지 vs
  대시보드 위젯)와 그룹 기준(노트 타입 vs 태그)을 먼저 확인 질문으로 정하고 "대시보드 안
  작은 미리보기 위젯" + "태그 기준"으로 확정. `topics/`가 물리적으로는 계속 flat 폴더라는
  점, 태그는 직접 쓰는 노트는 수동으로 붙여야 하고 "노트 가져오기"만 로컬 휴리스틱으로
  자동으로 붙는다는 점을 먼저 설명한 뒤 진행.
  - 데이터는 새로 만들지 않고 기존 `NoteMeta`(제목/태그/타입)와 `output/backlinks.json`의
    엣지를 그대로 재사용 — `generate-html.py`가 `assets/graph-data.js`로
    `window.MYREMEMBER_GRAPH = {nodes, edges}`를 굽는다(검색 페이지의 `search-data.js`와
    같은 패턴, fetch 없이 `<script>`로 바로 로드).
  - `graph-widget.js`: 외부 라이브러리(D3 등) 없이 순수 JS로 아주 단순한 힘-기반 레이아웃을
    직접 구현 — 노트끼리 서로 밀어내고, `[[링크]]`로 연결된 노트는 강하게, **대표 태그가
    같은 노트는 약하게** 서로 당긴다(지금 실제 vault엔 `[[링크]]`가 0개라 태그 인력이 없으면
    그냥 무작위로 흩어져 보였을 것). 지속 애니메이션이 아니라 고정 횟수만 미리 계산해서
    한 번에 그리는 방식(미리보기 위젯이라 CPU를 계속 쓸 필요 없음).
  - 색상은 `dataviz` 스킬의 검증된 8색 카테고리 팔레트를 이 프로젝트의 기존
    `rgb(var(--x))` 라이트/다크 토큰 패턴 그대로 이식(`--series-1~8`, `validate_palette.js`로
    인접-쌍 기준 라이트/다크 둘 다 PASS 확인). 노트 집합 안에서 등장 빈도 상위 8개 태그만
    고정 순서로 색을 받고, 9번째 이후 태그·태그 없는 노트는 전부 회색 "기타"로 폴백 —
    범례와 hover 툴팁에 항상 태그 이름 텍스트가 같이 있어서 색만으로 구분하지 않는다
    (스킬이 scatter류 전체-쌍 조합에서는 3개까지만 전 구간 안전을 보장한다고 명시한
    것에 대한 의도적 실용적 절충 — 개인 단일 사용자 도구라 텍스트 백업이 항상 있다는
    전제로 8색 전부 사용).
  - 점을 클릭하면 해당 노트로 바로 이동(SVG `<a>`), hover하면 제목+태그 툴팁(`<title>`).
    범례/툴팁 텍스트는 전부 `textContent`로 채워서 innerHTML 문자열 조립을 피함.
  - Playwright로 라이트/다크모드, 데스크톱/모바일(390px) 렌더링, 실제 색-태그 매핑이
    설계대로 나오는지(같은 태그를 공유하는 두 노트가 같은 색으로 묶이는지) JS로 직접
    검증, 클릭 시 실제로 해당 노트 페이지로 이동하는지까지 확인.
- **STATUS/DEVLOG/README/ARCHITECTURE/USER_GUIDE 전부 갱신** — 이번 세션에서 추가된 PDF
  가져오기, PDF 원본 보관, 그래프 위젯 세 기능을 모두 반영. 공개 문서 사이트
  (`myremember-docs`) 재배포는 사용자가 "문서 정리해줘"라고 요청할 때까지 보류(기존 원칙).
- **태그 관리 — 상위/하위 관계 + 병합 기능 추가**: 사용자가 "태그끼리 연관연결 + 연결 시
  주종(상위/하위)을 결정하는 기능", 이어서 "흩어진 태그를 합쳐서 같은 태그로 만드는 기능"도
  요청. 관리 방식(웹 UI vs config 파일), 구조(단일 상위 트리 vs 여러 상위 그래프), 활용처
  (태그 페이지 표시 vs 그래프 위젯 반영)를 먼저 확인 질문으로 정한 뒤 구현 — 웹 UI, 여러
  상위 허용(그래프 구조), 태그 페이지+그래프 위젯 둘 다 반영으로 확정.
  - **관계 저장**: 새 파일 `scripts/webviewer/data/tag-relations.json`
    (`{"parents": {자식태그: [상위태그, ...]}}`, `config.yaml`의 `tags.relations_path`로
    설정) — `dashboard-snapshot.json`과 같은 성격(git 추적되는 입력 데이터, output/처럼
    재생성 파생물이 아님)이라 같은 패턴을 그대로 따름. `vault.py`에
    `load_tag_relations()`/`save_tag_relations()` 공용 헬퍼 추가(server.py는 쓰기,
    generate-html.py는 읽기 양쪽에서 재사용).
  - **`POST /api/set-tag-relations`**: 태그 하나의 상위 목록을 통째로 교체. 여러 상위를
    허용하는 그래프 구조라 트리와 달리 순환(A의 상위가 B, B의 상위가 다시 A)이 저절로
    막히지 않아서, 저장 전 DFS로 순환 여부를 확인해 순환이 생기면 저장을 거부(에러 메시지
    반환, 파일 변경 없음).
  - **`POST /api/merge-tags`**: 흩어진 태그 여러 개(`sources`)를 하나(`target`)로 합친다 —
    vault 전체 노트 파일에서 `#소스태그`를 정확한 토큰 매칭(`#3D`가 `#3D프린터`의 일부로
    오매칭되지 않게 경계 처리)으로 `#타겟태그`로 치환하고, 실제로 파일이 수정된 노트 개수를
    반환. 이 프로젝트의 태그 표기 관례(마지막 줄이 `#태그 #태그 ...` 형식)를 따르는 노트는
    치환 후 그 줄에 중복된 태그가 남으면 정리도 같이 함(본문 중간 언급은 안 건드림).
    `tag-relations.json`에 남아있던 소스 태그명도 target으로 합쳐서(상위 목록 union, 자기
    참조 자동 제거) 관계가 끊기지 않게 함.
  - **"태그 관리" 페이지 신설**(`tags/manage.html`, 태그 목록 페이지 상단 버튼으로 연결):
    병합 폼(소스 다중선택 + 타겟 이름 입력, 기존 태그 자동완성 지원, 확인 대화상자 후 실행)
    + 태그별 상위 태그 편집. 관계 편집 UI는 "너무 어렵다"는 피드백을 받아 이후
    (2026-08-12 두 번째 세션) 태그 하나를 고르면 현재 상위 태그가 칩으로 보이고
    드롭다운+"추가"/칩의 `×`로 즉시 저장되는 방식으로 다시 만들었다(자세한 내용은 바로
    아래 항목 참고). 새 JS `tag-manage.js`.
  - **태그 페이지(`tag.html`)에 계층 표시**: 상위 태그는 브레드크럼에("대시보드 / 태그 /
    #상위태그1, #상위태그2"), 하위 태그는 카드 위 별도 줄에 링크 목록으로.
  - **그래프 위젯 재설계**: 기존엔 "같은 대표 태그의 노트끼리 서로 당기는" 노트-노트 인력
    구조였는데, 이번에 **태그 허브 노드**(색 배정된 태그 + 관계가 설정된 태그는 전부 허브
    생성, 라벨이 항상 보이는 좀 더 큰 원)를 새로 그려서 "노트→자기 대표 태그 허브" 인력 +
    "하위 태그 허브→상위 태그 허브" 인력(점선으로 연결선도 그림)으로 바꿨다 — 허브-스포크
    구조라 관계가 그래프에 실제로 드러난다. 허브를 클릭하면 그 태그 페이지로 이동. 데이터는
    `graph-data.js`에 `tagParents` 필드 하나만 추가해서 재사용(새 파이프라인 없음).
  - 검증: 임시 vault로 병합(정확 토큰 매칭·중복 정리)/관계 저장/순환 거부 세 가지를
    직접 함수 호출로 확인. 실제 vault·실서버로 관계 저장(태그 페이지 상위/하위 표시,
    그래프 위젯 허브+점선 반영까지 Playwright 라이트/다크/모바일 스크린샷)과 병합(디스포저블
    테스트 노트로 실제 파일이 정확히 바뀌는지) 둘 다 curl+UI로 재확인 후 테스트 흔적은 전부
    삭제해 원상 복구.
  - **참고**: 이 세션 도중 사용자가 실제로 "노트 가져오기" 기능을 여러 번 써서 vault에
    새 노트 4개(ESP32/라즈베리파이/마스터 인터페이스 정의서 등)가 실제로 쌓였다 — 이 기능
    작업과 무관하게 사용자가 직접 만든 정상적인 콘텐츠임(session 로그에서 확인, 건드리지 않음).
- **"새 노트 작성"**: 파일 업로드("노트 가져오기") 없이 제목+본문을 브라우저에서 바로
  입력해 `topics/`에 노트를 만드는 기능. `POST /api/create-note` 추가 —
  `import_note()`의 마무리 로직(자동 태그·슬러그화·중복 처리·저장)을
  `_save_new_topic_note()`로 뽑아내 새 `create_note()`와 공유하도록 리팩터(파일 변환
  단계만 없을 뿐 나머지는 동일 동작). 저장 후 사이트 재생성 + 새 노트 페이지로 이동.
  **(2026-08-12 갱신) 대시보드 카드가 아니라 사이드바 "새 노트" 링크 → 전용
  페이지(`new-note.html`)**로 옮겨졌다 — 자세한 내용은 아래 "대시보드 레이아웃 재정리"
  항목 참고.
- **태그 관계 UI를 칩 방식으로 단순화**: "인터페이스가 너무 어렵다"는 피드백을 받아
  `tags/manage.html`의 관계 편집을 다시 만들었다. 지금은: 드롭다운에서 태그를 하나 고르면
  현재 상위 태그가 알약 모양 칩으로 보이고, 다른 드롭다운+"+ 추가"로 상위를 더하거나
  칩의 `×`로 뺀다 — 둘 다 클릭 즉시 `/api/set-tag-relations`로 저장되고 별도 "저장"
  버튼은 없다(예전엔 태그마다 한 행 + `<select multiple>`로 Ctrl/Cmd+클릭해야 했음).
  전체 관계 맵은 `assets/tag-relations-data.js`로 클라이언트에 미리 전달돼 태그를
  고르자마자 서버 왕복 없이 바로 보여준다. 병합 폼(다중 선택이 개념상 맞는 동작)은
  그대로 뒀다. 실제 서버+Playwright로 선택→추가→칩 표시→파일 반영, 제거→칩 사라짐→파일
  반영, 라이트/다크/모바일까지 확인.
- **대시보드/사이드바 레이아웃 재정리(현재 기준)**: "자주 안 쓰는 액션은 대시보드에서
  빼고 자주 보는 정보를 위로"라는 요청, 이어서(2026-08-12) "관심종목/유튜브/카카오톡은
  좌측 메뉴로 빼달라"는 요청에 따라 최종적으로 재배치했다.
  - **사이드바**(모든 페이지 공통, `base.html`): 대시보드 · 노트 · 태그 · 검색 · Daily
    아카이브 (탐색) — 구분선 — **관심종목 · 유튜브 · 카카오톡**(각자 전용 페이지,
    `stocks.html`/`youtube.html`/`kakao.html` — readonly 여부와 무관하게 항상 생성되는
    순수 조회 페이지) — `{% if not readonly %}`일 때만: 구분선 — **오늘 노트**(버튼) ·
    **새 노트**(링크, `new-note.html`).
  - **대시보드는 2단 레이아웃**(`min-width:860px`부터, 그 아래는 세로 1단): 왼쪽
    `.dashboard-main`(위→아래: 통계 카드 → 할 일 위젯 → 노트 그래프 → 노트 가져오기 →
    빠른 메모) + 오른쪽 고정폭(300px, sticky) `.dashboard-side`에 **Gmail·캘린더만**
    탭으로 표시(관심종목/유튜브/카카오톡은 더 이상 이 칸에 없음 — 위 사이드바 참고).
  - **"노트 가져오기" 카드**: 제목·"파일 선택"·"원본 보관" 토글이 한 줄에. "원본 보관"은
    체크박스+긴 설명 문구 대신, hover 시 native `title` 툴팁으로 설명이 뜨는 작은 아이콘
    토글(`.icon-toggle`)로 바뀜 — 체크되면 배경이 강조색으로 바뀜.
  - 새 페이지 `new-note.html`: 사이드바 "새 노트"로 이동하면 나오는 제목+내용 전용 폼
    (대시보드보다 넓은 textarea).
  - `.page-narrow`(max-width:640px): 관심종목/유튜브/카카오톡 페이지처럼 원래 300px
    사이드 탭 전용으로 만들어졌던 콘텐츠(막대그래프·태그 칩)가 `.content`의 최대
    980px를 그대로 쓰면 카드 오른쪽에 빈 공간이 남아 보였던 문제를 좁은 폭으로
    감싸 해결(모바일은 어차피 640px보다 좁아 영향 없음). 태블릿 가로/데스크톱에서
    효과가 드러난다.
  - **(2026-08-12) 넓은 PC 화면 대응**: `.content`가 항상 980px로 고정돼 있어서
    넓은 모니터에서 창을 넓혀도 정보량이 그대로였던 걸, `min-width:1280px`(1180px·
    사이드 340px)/`min-width:1600px`(1400px·사이드 380px) 두 단계로 넓어지게
    함(`.page-narrow`는 예외 — 계속 640px 고정). **태블릿 화면 비율 재점검**:
    1024px(아이패드 가로 등)에서 사이드바+Gmail 열을 빼면 본문이 ~420px라 상단
    통계 3칸이 두 줄로 밀리던 걸 발견해 `.stats-row`의 `minmax` 기준을 140px→110px로
    낮춰 해결. 자세한 내용은 DEVLOG 2026-08-12 "PC 넓은 화면 대응 + 태블릿 화면
    비율 재점검" 참고. **(2026-08-13 갱신) 메일·캘린더 사이드 패널 폭 2배 + 흐르는
    반응형**: "폭을 2배로 늘리고 화면 비율에 따라 지금 비율까지 줄어들게" 요청 —
    `.dashboard-side` 폭을 고정 px(300/340/380) 대신 `clamp(현재값, ~vw, 2배값)`으로
    바꿔서, 초광폭 화면(대략 2700px+)에서는 최대 2배(600/680/760px)까지 흐르듯
    넓어지고 창을 좁히면 원래 폭까지 매끄럽게 줄어든다. 이 작업 중 **1280px/1600px
    브레이크포인트가 실제로는 한 번도 적용된 적이 없던 버그를 발견**했다 — 소스
    파일에서 이 두 `@media` 블록이 860px 블록보다 앞쪽에 있어서, CSS 캐스케이드는
    "더 큰 min-width"가 아니라 "소스 순서상 나중"을 우선하기 때문에 항상 860px 블록의
    값(300px)에 덮어써지고 있었다 — 세 블록을 860→1280→1600 오름차순으로
    재배치해서 같이 고침(과거 DEVLOG에 적힌 "1920px/1366px에서 확인" 기록은 육안상
    큰 차이가 안 보여 못 걸러졌던 것으로 보임). Playwright로 820~3000px 여러 폭에서
    실측 픽셀값을 찍어 확인.
- **유튜브 시청기록 페이지(Google Takeout 기반, JSON/HTML 둘 다 지원)**: YouTube Data
  API는 실제 시청기록을 못 준다는 제약이 있어, 사용자가 Google Takeout에서
  watch-history를 직접 받아 `scripts/webviewer/data/youtube-watch-history.json`에
  넣으면 `python3 scripts/analyze-youtube-history.py`가 채널별 시청 횟수(상위 15개)·
  전체 개수·날짜 범위·최근 30일 개수를 집계해 `youtube-summary.json`에 저장하는 구조.
  **형식은 파일명이 아니라 내용으로 자동 판별**(Takeout 기본값이 HTML이라 처음
  JSON만 가정했던 걸 나중에 HTML도 지원하도록 고침 — 자세한 내용은 DEVLOG 2026-08-12
  "유튜브 시청기록 HTML 형식 지원" 참고. HTML 쪽은 광고 여부를 구분하는 필드가 없어
  광고 시청이 총 개수에 섞일 수 있다는 한계가 있음). Gmail/캘린더와 달리 **API 키도
  Claude 세션도 필요 없이 스크립트만 다시 돌리면 갱신됨**(그래서 데이터 저장을
  `dashboard-snapshot.json`과 분리해뒀다). 사이드바 "유튜브" 전용 페이지
  (`youtube.html`)에 채널별 막대 목록으로 표시, 웹 화면에서 파일 선택만으로 바로
  분석까지 되는 업로드 카드도 있음(아래 "웹 화면에서 유튜브/카카오톡 업로드" 항목
  참고). `scripts/webviewer/data/` 전체(Gmail/캘린더/유튜브 등 개인 데이터가 모이는
  폴더)를 `.gitignore`에 추가해 실수로 커밋되는
  일을 막아둠.
- **웹뷰어 대시보드를 클라우드에 읽기 전용 + 비밀번호 게이트로 배포함**:
  `scripts/build-cloud-site.py`가 쓰기 UI를 전부 뺀 정적 사이트 + SHA-256 해시 기반
  비밀번호 게이트를 만들고(자세한 구현은 DEVLOG 2026-08-12 "클라우드 배포용 읽기 전용
  빌드" 참고), 실제로 새 저장소 `peter-cho-70/myremember-vault`를 만들어 배포했다.
  **저장소는 public이다** — 처음엔 private로 만들었지만 이 계정의 GitHub 플랜은
  private 저장소에서 Pages 자체를 지원하지 않아(API가 "current plan does not support
  GitHub Pages for this repository"로 명시적으로 거부) public으로 전환했다. 어차피
  "비밀번호로 보호된 공개 사이트"가 이미 확정된 설계였어서 목표 자체는 그대로다 — 다만
  저장소 소스 파일(빌드된 정적 HTML)도 GitHub에서 볼 수 있게 됐다는 점은 "private
  저장소" 표현과 달라진 부분이라 명시해 둔다.
  - **실제 URL**: https://peter-cho-70.github.io/myremember-vault/ — 비밀번호는 이
    문서에는 안 남기고 사용자에게 대화로 별도 전달함(무작위 16자 영숫자, 매번 다시
    빌드하면 새로 생성 가능).
  - Playwright로 실제 배포된 URL에 직접 접속해 ① 첫 진입 시 게이트가 콘텐츠를 가리는지,
    ② 올바른 비밀번호로 잠금 해제되는지, ③ `assets/gate-hash.js`에 담긴 해시가 로컬
    빌드와 일치하는지, ④ 쓰기 UI(가져오기 입력창 등)가 전혀 없는지까지 실제 라이브
    사이트 기준으로 확인.
  - **허브(`peter-cho-70.github.io`)에는 등록하지 않음** — 이 저장소용 CLAUDE.md의
    "문서화 표준" 사전 승인 워크플로우는 문서 사이트 발행에 한정되고, 이건 개인 vault
    콘텐츠(비밀번호로 막아둔 이유가 있는 것)라 공개 허브에 링크를 노출하면 게이트를
    두는 의미가 없어짐.
  - **갱신 방법(2026-08-12 재설계됨)**: `MYREMEMBER_CLOUD_PASSWORD='...' python3
    scripts/build-cloud-site.py --push` 한 줄이면 빌드+동기화+커밋+푸시까지 전부
    끝난다. 빌드 결과물(`output/html-cloud`, 소스 저장소 안, 매번 통째로 재생성)과
    실제 git 체크아웃(`../myremember-vault-deploy`, 소스 저장소 밖의 형제 폴더 — 처음
    실행 시 `myremember-vault`를 자동 클론)을 분리해 `rsync`로 동기화하는 구조 —
    자세한 배경은 DEVLOG 2026-08-12 "[긴급] 배포 사이트 노트 링크 깨짐 수정 +
    배포 스크립트 사고 후 재설계" 참고(과거엔 빌드 폴더 안에서 직접 git을 써서,
    재생성 때마다 `.git`이 같이 지워지는 사고가 있었음). `--push` 없이 실행하면
    빌드+동기화만 하고 커밋은 하지 않는다.
  - **NFC/NFD 한글 파일명 링크 깨짐 버그 수정(2026-08-12)**: 배포된 사이트에서 한글
    파일명 노트/태그 링크가 전부 404였던 치명적 버그 — macOS가 결합 문자를 NFD로
    저장하는데 GitHub Pages는 NFC 기준으로 경로를 찾아서 발생. `vault.py`의
    `note_id()`/`extract_tags()`를 NFC 정규화하도록 중앙화해서 해결, 라이브 사이트
    기준으로 노트 14개 링크 전부 200 확인 완료. 자세한 내용은 위 DEVLOG 항목 참고.

- **카카오톡 대화 분석 페이지(대화 내보내기 파일 기반, CSV/텍스트 둘 다 지원)**:
  유튜브와 같은 패턴 — API가 없어 카카오톡 앱/PC판의 "대화 내용 내보내기"로
  받은 파일을 `scripts/webviewer/data/kakao-chat.txt`에 넣고
  `python3 scripts/analyze-kakao-chat.py`를 돌리면 공유 링크(도메인별 집계)와
  메시지 키워드(조사 제거 + 불용어 필터)를 `kakao-summary.json`에 저장한다. 원문은
  저장하지 않고 집계만 남김(그룹 채팅 상대방 메시지 노출 최소화). 사이드바 "카카오톡"
  전용 페이지(`kakao.html`)에 링크 막대그래프 + 키워드 태그로 표시. **형식은 내용
  첫 줄로 자동 판별**(처음엔 안드로이드 앱 텍스트 내보내기만 가정했는데, 실제
  사용자 파일은 PC판이 만드는 CSV(`Date,User,Message` 헤더)였다 — 첫 줄이 CSV
  헤더로 보이면 표준 라이브러리 `csv.DictReader`로, 아니면 기존 텍스트 파서로
  처리. 실제 파일(1,350개 메시지)로 검증 완료, 자세한 내용은 DEVLOG 2026-08-12
  "카카오톡 CSV 형식 지원" 참고).
- **웹 화면에서 유튜브/카카오톡 파일 선택만으로 바로 분석**: 예전엔 터미널에서
  `watch-history`/카카오톡 내보내기 파일을 정해진 경로에 직접 옮기고 분석
  스크립트를 수동으로 실행해야 했는데, `youtube.html`/`kakao.html`(로컬
  `server.py`로 열었을 때만, `{% if not readonly %}`)에 "노트 가져오기"와 같은
  업로드 카드를 추가해 파일 선택만으로 서버가 그 자리에서 분석 스크립트를 실행하고
  페이지가 새로고침되며 결과가 바로 반영되게 했다. `server.py`에
  `/api/import-youtube`/`/api/import-kakao` 두 엔드포인트, 새 JS
  `assets/data-import.js`(업로드→분석→새로고침 패턴, 실패 시 이전 데이터 보존한
  채 에러 문구만 표시). 클라우드 배포 사이트는 읽기 전용이라 이 업로드 카드
  자체가 안 보임(스크립트를 실행할 서버가 없어서 당연한 제약, 기존 "노트
  가져오기"와 같은 원칙). **(2026-08-12 실제 Takeout 파일로 검증 후 버그 2개
  수정)** `import_youtube_history()`가 여전히 JSON 전용 사전 검증을 하고 있어서
  HTML 업로드가 전부 500 에러였던 것과, 요청 크기 제한(20MB)이 실제 시청기록
  파일(수십MB 흔함) 대비 너무 작았던 것 — 사전 검증은 `analyze-youtube-history.py`
  한 곳으로 정리하고 제한은 100MB로 상향. 실제 36MB/3만5천개 항목 파일로 업로드
  →분석→반영까지 전체 과정 검증 완료(자세한 내용은 DEVLOG 2026-08-12 "유튜브 웹
  업로드 버그 두 개 수정" 참고).
- **(2026-08-13) "노트" 목록(Areas/Topics) 정렬을 최신순으로 변경**: "노트에 파일을
  입력하면 최신이 맨 위로 가게 해달라"는 요청 — `NoteMeta`에 파일 수정시각(`mtime`)을
  추가하고, Areas/Topics 정렬 기준을 각각 경로/제목 알파벳순에서 `mtime` 내림차순으로
  바꿨다(Daily는 이미 날짜 기준 최신순이라 그대로 둠). areas/topics 파일명에는 daily처럼
  날짜가 안 박혀 있어서 mtime이 "최신"의 유일하게 쓸 수 있는 기준이다.
  - **재발견한 운영 버그**: 수정 직후 사용자가 "지금도 정렬이 안 된다"고 신고 — 원인은
    코드가 아니라, 이미 새벽부터 떠 있던 로컬 미리보기 서버(`server.py`)가
    `generate-html.py`를 프로세스 시작 시점에 메모리로 한 번만 읽어들이고 있어서 그 뒤의
    소스 수정을 반영하지 못한 것이었다(전에도 "정적 자산 캐시" 문제로 비슷한 유형을 겪은
    적 있음 — 이번엔 브라우저 캐시가 아니라 서버 프로세스 자체의 캐시). 서버를 재시작해
    해결 — **`server.py`가 이미 떠 있는 상태에서 `generate-html.py`(또는 서버가 import하는
    다른 스크립트)를 고치면, 반영하려면 서버 재시작이 필요하다**는 걸 사용자에게 안내함.
  - **Areas 사용법 질문에 답변**(코드 변경 아님): PRD 기준 Areas=시작·끝이 있는 프로젝트
    (`areas/{프로젝트}/README.md`+`progress.md`+`decisions.md`), Topics=끝이 없는 지속
    관심사. 지금 vault의 `areas/`는 비어 있고 콘텐츠가 전부 `topics/`에 있음 — 강제 사항은
    아니라고 안내.
- **(2026-08-14) 캘린더 오늘 기준 재갱신 + 캘린더 탭에도 새로고침 안내 아이콘**: Google
  Calendar MCP로 접근 가능한 캘린더 10개 전체를 다시 조회해 `dashboard-snapshot.json`을
  오늘(08/14) 기준으로 갱신. Gmail 탭에만 있던 "새로고침 안내" 아이콘(hover 시 "Claude에게
  갱신 요청" 툴팁)을 캘린더 탭에도 동일 패턴(`.side-tab-summary-row`+`.btn-icon`)으로
  추가해 짝을 맞췄다. **Gmail은 이번엔 갱신 못 함** — Gmail MCP 커넥터 토큰이 만료되어
  재인증 필요(claude.ai에서 Gmail 커넥터 재연결 후 다음 세션에서 갱신 가능). 자세한 내용은
  DEVLOG 2026-08-14 "캘린더 오늘 기준 재갱신 + 캘린더 탭에도 새로고침 안내 아이콘" 참고.
- **(2026-08-14) CADENCE 컨셉 검토 + daily note에 Phase 0 회고 4문항 추가**: 사용자가
  만든 개인 프로젝트 컨셉 문서(`docs/cadence-concept.html`, 행동 축적형 개인 업무 OS)를
  검토하고 개발 방향 의견 제공 — Phase 0(습관 검증)을 새 도구·저장소 없이 이 vault
  안에서 시작하도록 권했고 사용자가 동의. `config.yaml`의 `daily_note.template`에
  "CADENCE 회고 (Phase 0 검증)" 섹션(Q1~Q4)을 추가해 이후 모든 daily note에 자동
  포함되게 함. 자세한 내용은 DEVLOG 2026-08-14 "CADENCE 컨셉 검토 + daily note에 Phase 0
  회고 4문항 추가" 참고.
- **(2026-08-14) "할 일을 스마트폰에 보내기" — 경량 클라우드 동기화**: 노트 하나 안 바뀌어도
  전체 사이트를 재빌드+커밋+푸시해야 했던 기존 `build-cloud-site.py`와 달리, 새 스크립트
  `scripts/sync-cloud-todos.py`는 `assets/todos-data.js` 파일 하나만 갱신해서 커밋+푸시한다
  — 비밀번호 게이트(`gate-hash.js`)를 안 건드리므로 **`MYREMEMBER_CLOUD_PASSWORD`가 필요
  없음**. `server.py`에 `POST /api/sync-cloud-todos`, 대시보드에 "할 일을 스마트폰에
  보내기" 버튼 + 상태 텍스트 추가. **버그 2개 발견/수정**: ① 이 버튼이 readonly(배포된)
  사이트에도 잘못 노출되던 것을 `{% if not readonly %}`로 숨김. ② 더 근본적으로,
  `window.MYREMEMBER_TODOS` 스냅샷이 "그 기기가 그 날짜를 localStorage에 한 번도 연 적
  없을 때만" 초기값으로 쓰이는 기존 설계 때문에, 폰이 이미 그 날짜를 한 번 열어본 뒤로는
  PC가 아무리 새로 보내도 절대 반영되지 않는 문제를 발견 — `save_todos()`가 저장할 때마다
  `synced_at` 타임스탬프를 `todos.json`에 같이 남기고, `dashboard.js`가 페이지 로드 시 이
  값을 그 기기가 마지막으로 반영한 시각과 비교해서 더 새로우면 SNAPSHOT의 모든 날짜를
  강제로 덮어쓰도록 수정(**"보내기"를 누른 뒤에는 PC 쪽이 우선** — 그 사이 폰에서 직접
  체크/수정한 내용은 다음 새로고침 때 덮어써질 수 있음, 기존 "뷰어일 뿐 양방향 동기화
  아님" 설계에서 한 단계 더 나아간 것). 자세한 내용은 DEVLOG 2026-08-14 "'할 일을
  스마트폰에 보내기': 경량 클라우드 동기화 + 스냅샷 최초 1회 시딩 버그 발견/수정" 참고.
- **(2026-08-14) 잘못 가져온 CADENCE 노트 정리**: 배포 전 점검 중 "노트 가져오기"로 만든
  `topics/cadence-concept.md`가 실제로는 반도체/EDA 회사 "Cadence Design Systems"에 대한
  브라우저 AI 탭 검색 결과(사용자가 실수로 붙여넣은 것)였음을 발견해 삭제, 나머지
  `cadence-concept-v02.md`의 잘못된 자동 태그(`#LED #WS2812` 등 — 다른 노트에서 잘못
  매칭된 것)를 `#CADENCE #업무관리 #컨셉설계`로 수정. 자세한 내용은 DEVLOG 2026-08-14
  "잘못 가져온 CADENCE 노트 정리" 참고.
- **(2026-08-14) 사이드바 왼쪽 상단 버전 표시 + 최신 아이템 등록 시간**: "프로그램이
  업그레이드되거나 새 내용이 첨가되면 버전관리를 해줘. 왼쪽 상단에 날짜와 버전을 넣고,
  최신 아이템 등록 시간도 표시해줘" 요청 구현. `config.yaml`에 `app.version_path`
  (`scripts/webviewer/data/version.json`, `{version, date, summary}`) 추가 — 기능을
  추가/수정할 때마다 Claude가 이 파일을 갱신한다(이 vault는 작업이 미커밋 상태로 오래
  쌓이는 경우가 많아 git 커밋 수 대신 별도 버전을 관리). `generate-html.py`에
  `load_app_version()`/`latest_registered_item()`(vault 전체 노트 중 `mtime` 최신 1개,
  Areas/Topics 정렬에 이미 쓰는 기준 재사용) 추가해 `base_ctx`로 모든 페이지에 전달.
  `base.html`의 `sidebar-head`(48px 고정 높이 → 가변 높이로 변경)에 브랜드명 아래
  "v1.0 · 2026-08-14"와 "최근 등록 {제목} · {시각}"(제목 클릭 시 그 노트로 이동, 넘치면
  말줄임+hover 툴팁) 두 줄 추가. readonly(클라우드 배포본)에도 그대로 표시(정보 노출
  성격상 굳이 숨길 이유 없음).
- **(2026-08-14) 할 일 갱신 안 된다는 신고 조사 — 실제로는 정상**: "대시보드에 할일도
  갱신되지 않았어"라는 신고를 받아 로컬 `todos.json`·배포된 `myremember-vault`의
  `todos-data.js`·`dashboard.js`(스냅샷 재적용 로직 포함 여부)를 전부 대조 확인 —
  `synced_at`·항목 14개 전부 로컬/배포 완전 일치, 로컬 화면도 정확히 반영됨을 확인. 코드
  결함은 못 찾음 — GitHub Pages CDN 캐시(`max-age=600`)나 브라우저 캐시로 인한 일시적
  표시 지연으로 추정, 사용자에게 새로고침 확인을 요청하고 원인 특정 대기 중(재현되면 다시
  조사 필요).
- **(2026-08-15) Gmail/캘린더 새로고침이 실제로 동작하도록 전환**: 클릭하면 "Claude에게
  보낼 안내 문구를 클립보드에 복사"만 하던 방식(2026-08-14, MCP 도구는 Claude 세션
  전용이라 로컬 서버 혼자서는 못 불러온다는 제약 때문)을 버리고, `server.py`가 `claude`
  CLI를 `-p`(비대화형) subprocess로 직접 호출해 Gmail/Calendar MCP 도구로 실데이터를
  가져와 `dashboard-snapshot.json`을 갱신하고 사이트를 재생성하는 방식으로 바꿨다.
  `--allowedTools`로 필요한 MCP 도구 딱 두 개(Gmail: `list_labels`/`search_threads`,
  캘린더: `list_calendars`/`list_events`)만 허용해서 다른 MCP 서버(Vercel/Drive 등)
  스키마가 컨텍스트에 실리는 비용을 줄임(실측: 전체 로드 시 캐시 생성 42k 토큰 → 이
  방식 13k). 새 엔드포인트 `POST /api/refresh-gmail`/`POST /api/refresh-calendar`,
  버튼 클릭 시 잠금+아이콘 회전(`.btn-icon svg.spin`)으로 진행 중임을 보여주고 완료되면
  자동 새로고침, 실패하면 버튼 아래에 이유 표시(claude CLI 미설치/미로그인 등). 도구
  호출이 여러 번 필요해 1~3분 걸릴 수 있음. 두 버튼 모두 노트 편집/삭제 버튼과 같은
  패턴으로 항상 보이되(`readonly` 여부로 안 숨김) 로컬 서버가 있을 때만 실제로 동작한다.
  **Gmail MCP 재인증도 이번에 완료됨** — 2026-08-14 세션에 토큰 만료로 갱신이 막혀 있던
  것이 해결되어, 이 버튼으로 직접 갱신한 오늘(08/15) 기준 스냅샷(받은편지함 3669통·
  안읽음 2521통, 캘린더 3개·08/15~08/17)을 확인함(아래 "진행 중"의 재인증 항목은 이제
  해소됨).
- **(2026-08-15) DailyLogSync — daily note 자동 기록** (`scripts/sync-daily-log.py`
  신설): daily note의 "진행 상황"/"관련 노트" 섹션에 그날 완료한 할 일과 그날
  추가/수정된 노트(제목+태그)를 손으로 옮겨 적지 않아도 자동으로 채워 넣는다. 노트를
  쓰거나 지우는 모든 경로(`rebuild_site`)와 할 일 저장 경로(`save_todos` — 무거운 사이트
  전체 재생성 없이 이 파일만 가볍게 갱신) 양쪽에서 호출된다. HTML 주석 마커
  (`<!-- auto:done-todos:start -->`/`<!-- auto:new-notes:start -->`)로 그 구간만
  갈아끼우고, 마커 밖에 사용자가 직접 쓴 내용은 절대 건드리지 않는다. 대상 날짜의 daily
  note가 없으면 `create-daily-note.py` 로직으로 먼저 만든다. `--date YYYY-MM-DD` 인자로
  특정 날짜 백필도 가능.
- **(2026-08-15) Tailscale로 폰에서 로컬 서버 바로 접속**: `server.py`가 기존
  `127.0.0.1` 바인딩은 그대로 유지한 채, 이 기기에 Tailscale이 로그인돼 있으면 tailnet
  IP(`100.x.y.z`)에도 추가로 바인딩(별도 스레드의 두 번째 `ThreadingHTTPServer`)해서
  터미널에 `Tailscale로 폰 등 다른 기기에서: http://100.x.y.z:4500` 안내를 띄운다. 별도
  배포(재빌드+커밋+푸시) 없이 저장하자마자 폰에도 바로 보이는 게 장점이고, 이 컴퓨터가
  켜져 있고 `preview.sh`가 떠 있을 때만 동작한다(상시 서비스 아님). 일반 Wi-Fi의 다른
  기기나 인터넷 전체에는 여전히 안 열려서, "127.0.0.1 전용"으로 지키려던 보안 의도(다른
  기기가 vault에 못 쓰게)는 그대로 유지된다 — tailnet에 로그인된 기기만 닿을 수 있다.
  사용법을 정리한 `topics/tailscale-사용자-매뉴얼.md` 노트도 같이 작성함.
- **(2026-08-15) 신규 "사용법" 페이지** (`guide.html`, 사이드바 새 탭): 대시보드/노트/
  검색/Daily 아카이브/관심종목·유튜브·카카오톡/Tailscale 폰 접속/기타 기능을 한 화면에
  정리한 인앱 도움말. `readonly`(클라우드 배포본) 여부에 따라 안내 문구가 달라진다 —
  배포본은 "조회 전용, 쓰기 기능 버튼 자체가 없음"을, 로컬 서버는 "이 화면의 모든 기능이
  동작함"을 설명. `USER_GUIDE.md`와 내용은 겹치지만, 터미널 문서를 열지 않고 화면 안에서
  바로 참고할 수 있게 하려는 목적.

## 🚧 진행 중

- Phase 4 스크립트(`git-auto-commit.sh`/`backup.sh`)는 작성 완료했지만 **실제 실행은 아직
  안 함** — `--push`와 클라우드 업로드는 사용자 승인 후 다음 세션에서. `rclone`도 아직 미설치
  (`rclone config`의 Google 계정 OAuth는 브라우저에서 사용자가 직접 완료해야 함)
- **클라우드 배포 사이트(`myremember-vault`) 재배포 필요**: 2026-08-13분까지는 사용자가
  08-14 세션 중 `build-cloud-site.py --push`로 실제 배포 확인함(로그에 16:09 커밋 확인).
  그 이후 만든 "할 일을 스마트폰에 보내기" 스냅샷 재적용 버그 수정(`synced_at`)·CADENCE
  노트 정리·버전 표시(08-14)와, 이번(08-15) DailyLogSync·사용법 페이지(`guide.html`)·
  Gmail/캘린더 실시간 새로고침·Tailscale 안내(단, Tailscale 카드 자체는
  `{% if not readonly %}`라 클라우드본에는 안 보임)가 전부 아직 반영 안 됨. 다음에
  `MYREMEMBER_CLOUD_PASSWORD='...' python3 scripts/build-cloud-site.py --push`를 실행해야
  최신 상태가 반영된다.

## 🐞 알려진 이슈

- `generate-html.py`의 헤딩 링크(`[[note#heading]]`) 앵커는 자체 slugify라 Pandoc의
  `auto_identifiers`와 100% 동일한 알고리즘은 아니다. 테스트한 케이스(한글 헤딩)는 일치했지만
  엣지 케이스(중복 헤딩, 특수문자 등)는 다를 수 있음.
- `search.sh`의 fzf 인터랙티브 경로(프리뷰 창 등)는 개발 환경에 fzf가 없어 직접 확인 못 함 —
  ripgrep 폴백 경로는 정상 동작 확인.
- ~~"노트 가져오기"의 제목 추출이 스타일이 많이 들어간 HTML에서 지저분해짐~~ →
  pandoc 옵션(`-native_divs-native_spans`, `-inline_code_attributes`)으로 해결됨(위
  "완료" 참고). 단, **이미 그 문제로 만들어진 기존 노트 1개**(`topics/내가-만든-명령어를-
  명령어stylebackgroundrgba...법.md`)는 파일명이 여전히 지저분한 채로 남아있다 — 사용자가
  이미 "편집"으로 본문 제목은 정리했지만 파일명(URL)까지 바꾸려면 삭제 후 재가져오기하거나
  수동으로 리네임 필요.
- **노트 가져오기 자동 태그의 정확도 한계**: 형태소 분석기 없는 조사 제거 규칙 기반이라,
  "는/은"처럼 조사와 동사 활용 어미가 겹치는 경우 가끔 어색한 태그(동사 어간 등)가 붙을
  수 있다. 제목 앞부분만 후보로 봐서 최소화했지만 완전하지는 않음 — 필요하면 Claude
  세션에서 태그를 다듬어달라고 요청.

## 🗺️ 다음 우선순위

- **vault 웹뷰어(`output/html`)의 GitHub Pages 배포는 계속 보류 중** — Free 플랜은 private
  저장소여도 Pages 사이트 자체는 public이 되기 때문에, vault에 개인적인 내용이 쌓이는 걸
  감안해 미룸(공개된 건 `myremember-docs`뿐 — PRD/개발 기록이지 vault 콘텐츠가 아님).
  지금은 `python3 -m http.server --directory output/html`로 로컬에서만 사용.
- 사용자가 Phase 1~3 전체를 직접 테스트 (본인이 예고함)
- PRD Phase 1 남은 항목: cron 잡 등록 (사용자가 아직 원하지 않아 보류)
- Phase 4 마무리: `rclone` 설치·`rclone config`(사용자가 직접), `config.yaml`에
  `backup.rclone_remote` 채우기, `git-auto-commit.sh --push`/`backup.sh` 실제 실행 승인
- 실제 콘텐츠가 쌓이면 `build-search-index.py`/`build-backlinks.py`/`generate-html.sh`/`validate-links.py`를 주기적으로 돌려 최신 상태 유지
- 로컬 저장소에 미커밋 변경사항 다수(UI 리디자인, 대시보드, Phase 4 스크립트, 이번(08-15)
  DailyLogSync·Tailscale·Gmail/캘린더 새로고침·사용법 페이지 등) — 사용자 확인 후 커밋 필요
- 관심종목 실데이터가 필요하면 실제 시세 API 연동(한국투자증권/키움 OpenAPI 등) 여부를
  사용자와 상의
- **클라우드 배포 사이트(`myremember-vault`) 재배포**: 위 "진행 중" 참고 — 08-14~08-15분
  변경사항까지 반영하는 재배포 필요(`MYREMEMBER_CLOUD_PASSWORD` 있어야 함, Claude는 실행
  불가)
- 기능 추가/수정할 때마다 `scripts/webviewer/data/version.json` 갱신하는 걸 습관화(이번
  세션에도 아직 반영 안 됨 — 다음에 이어서 갱신)
