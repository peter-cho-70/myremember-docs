# MyRemember 사용설명서

> 기준일: 2026-08-11 — 처음 시작하는 사람 기준으로 처음부터 끝까지 순서대로 따라갈 수 있게 정리했다.
> 전체 기능 목록/설계 배경은 [`README.md`](../README.md), [`ARCHITECTURE.md`](spec/ARCHITECTURE.md) 참고.

## 0. 사전 준비

| 필요한 것 | 용도 | 설치 |
|---|---|---|
| Python 3.11+ | 모든 자동화 스크립트 | macOS는 보통 기본 설치돼 있음(`python3 --version`으로 확인) |
| pandoc | 웹뷰어 생성(`generate-html.sh`) | `brew install pandoc` |
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
하루 종일 채워나가는 방식으로 쓴다.

```bash
python3 scripts/create-daily-note.py
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
- **주제(계속 관심을 두는 분야)**: `topics/{주제명}.md` 파일을 새로 만든다.

양식 예시는 [PRD 문서](spec/PRD.md)의 "부록: 마크다운 양식 예시"에 있다.
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
`http://localhost:8000`을 브라우저로 열면 홈/노트/태그/검색 페이지와 다크모드 토글이 있는
개인 위키가 뜬다. 종료는 `Ctrl+C`, 포트를 바꾸려면 `scripts/preview.sh 8080`처럼 인자를 넘긴다.

지금은 로컬 전용이다 — GitHub Pages 같은 공개 배포는 하지 않는다. vault에 개인적인 프로젝트/
일상 메모가 쌓이는데, GitHub Free 플랜은 저장소가 private이어도 Pages 사이트 자체는 URL을
아는 누구나 볼 수 있는 public이 되기 때문이다(자세한 이유는 `ARCHITECTURE.md` 참고). 이
사용설명서를 포함한 PRD/개발기록만 별도의 공개 문서 사이트(`myremember-docs`)로 분리해 공개한다.

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
  않는 작업이라 확인 없이 실행된다.
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
| `scripts/preview.sh [포트]` | 백링크 갱신 + 웹뷰어 생성 + 로컬 서버 (기본 8000) |
| `scripts/git-auto-commit.sh [--push] [--yes]` | 커밋 + 월간 태그 (+선택적 push) |
| `scripts/backup.sh [--yes]` | 로컬 백업 (+설정된 경우 클라우드 업로드) |

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
