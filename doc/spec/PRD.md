# MyRemember - 개인용 지식 관리 시스템 PRD

**Version**: 1.0  
**Last Updated**: 2026-08-11  
**Status**: 기획 단계

---

## 1. 제품 개요

### 1.1 비전
로컬 우선(Local-First) 방식으로 개인의 모든 관심사와 프로젝트를 체계적으로 관리하면서, 자동화를 통해 반복 작업을 최소화하고 지식이 자동으로 연결되는 개인용 AI 지식 관리 시스템.

### 1.2 핵심 가치
- **소유권**: 모든 데이터는 로컬에 저장, 클라우드 벤더 종속 없음
- **자동 연결**: Backlink와 Full-text 인덱싱으로 지식 자동 연결
- **자동화 중심**: Python/Bash 스크립트로 반복 작업 제거
- **확장성**: Git + 클라우드 백업으로 어디서든 접근
- **개발자 친화적**: CLI, API, 마크다운 기반

### 1.3 Argo에서 영감 받은 개념

| Argo 개념 | MyRemember 적용 |
|---------|----------------|
| **Company** (워크스페이스) | 개인 지식 저장소 (~/my-knowledge/) |
| **Crew** (AI 에이전트) | 자동화 에이전트 (검색, 인덱싱, 변환) |
| **Vault** (폴더 규모 메모리) | /areas, /topics, /daily 계층 구조 |
| **Delegation** (위임) | 자동화 스크립트의 역할 분담 |
| **Approval** (승인) | 중요 작업 전 검증 (링크 검증, 충돌 확인) |
| **Memory Accumulation** | Git 커밋 히스토리 + 마크다운 Backlink |
| **System Prompt** | 각 자동화 스크립트의 규칙 정의 |

---

## 2. 요구사항 분석

### 2.1 주요 사용자 시나리오

**시나리오 1: 새로운 관심사 학습**
```
1. 주제 폴더 생성: /topics/new-topic.md
2. 기존 노트와 자동 Backlink 생성
3. 주간 요약 자동 생성
4. GitHub에 자동 커밋
→ 최소한의 수작업으로 체계적인 기록 유지
```

**시나리오 2: 프로젝트 진행**
```
1. 프로젝트 폴더: /areas/project-name/
   - README.md (개요)
   - tasks.md (할 일)
   - decisions.md (의사결정)
2. 일일 노트에서 진행 상황 기록
3. 자동으로 프로젝트 현황 요약 생성
4. 관련 주제와 자동 연결
→ 프로젝트 진행 과정의 완전한 추적
```

**시나리오 3: 어디서든 접근**
```
1. 데스크톱: VS Code에서 편집
2. 웹: GitHub 웹 뷰어 또는 생성된 HTML 사이트
3. 모바일: Git 클라이언트 또는 웹 뷰어
→ 로컬 소유권 + 어디서든 접근성
```

### 2.2 핵심 기능 요구사항

| 계층 | 기능 | 우선순위 | 자동화 여부 |
|------|------|---------|----------|
| **저장소** | 마크다운 폴더 구조 | P0 | 수동 |
| **저장소** | 폴더별 INDEX.md 자동 생성 | P1 | 자동 |
| **연결** | [[노트]] 링크 검증 | P1 | 자동 |
| **연결** | 역참조(Backlink) 추출 | P2 | 자동 |
| **연결** | 관계도 시각화 | P3 | 자동 |
| **검색** | Full-text 인덱싱 (SQLite) | P1 | 자동 |
| **검색** | CLI 검색 도구 | P1 | 수동 |
| **변환** | Pandoc: MD → HTML | P2 | 자동 |
| **변환** | Pandoc: MD → PDF | P2 | 자동 |
| **버전** | Git 자동 커밋 | P1 | 자동 |
| **버전** | 자동 태그 생성 | P2 | 자동 |
| **백업** | Rclone → Google Drive | P1 | 자동 |
| **백업** | 로컬 압축 백업 | P1 | 자동 |
| **접근** | VS Code 마크다운 플러그인 | P0 | 수동 |
| **접근** | 웹 뷰어 (정적 HTML) | P2 | 자동 |
| **일일** | 일일 노트 자동 생성 | P1 | 자동 |
| **요약** | 주간/월간 요약 생성 | P2 | 자동 |

---

## 3. 시스템 아키텍처

### 3.1 폴더 구조 (Vault)
```
~/my-knowledge/
├── /areas/                  # 프로젝트별 (진행 중 또는 완료된 것)
│   ├── audio-article-highlight/
│   │   ├── README.md
│   │   ├── progress.md
│   │   └── decisions.md
│   ├── vmix-auto-cam-switch/
│   └── tokyo-trip/
├── /topics/                 # 주제별 (지속 학습 분야)
│   ├── 3d-printer-research.md
│   ├── behavioral-economics.md
│   ├── real-estate.md
│   └── ai-coding-workflow.md
├── /daily/                  # 일일 노트 (타임시리즈)
│   ├── 2026-08-11.md
│   ├── 2026-08-10.md
│   └── index.md             # 월별 인덱스
├── /scripts/                # 자동화 스크립트
│   ├── create-daily-note.py
│   ├── validate-links.py
│   ├── build-search-index.py
│   ├── build-backlinks.py
│   ├── generate-html.sh
│   ├── backup.sh
│   └── config.yaml          # 스크립트 설정
├── /output/                 # 생성 결과 (git ignore)
│   ├── search.db
│   ├── backlinks.json
│   └── /html/
├── /backups/                # 로컬 백업 (git ignore)
├── README.md                # 전체 인덱스 및 가이드
├── .gitignore
├── .github/
│   └── workflows/
│       └── auto-commit.yml  # GitHub Actions (선택)
└── ARCHITECTURE.md          # 이 문서
```

### 3.2 자동화 에이전트 (Crew 패턴)

Argo의 "Crew"처럼, 각 자동화 스크립트가 특정 역할을 담당:

```yaml
crew:
  - name: "DailyScribe"
    role: "매일 아침 새로운 노트 생성"
    script: create-daily-note.py
    schedule: "0 6 * * *"  # 매일 6시
    rules:
      - "오늘 날짜로 /daily/{YYYY-MM-DD}.md 생성"
      - "템플릿 사용: 주제, 진행상황, 배운점, 내일 할 일"
      - "중복 생성 방지"
    
  - name: "LinkValidator"
    role: "마크다운 참조 무결성 검사"
    script: validate-links.py
    schedule: "0 12 * * 0"  # 주 1회 (일요일)
    rules:
      - "[[노트]] 형식의 모든 링크 검증"
      - "없는 파일 목록 리포트 생성"
      - "깨진 링크 발견 시 GitHub Issue 자동 생성 (옵션)"
    approval: true  # 중요 작업이므로 승인 필요
    
  - name: "SearchIndexer"
    role: "전문검색 인덱스 생성"
    script: build-search-index.py
    schedule: "0 2 1 * *"   # 월 1회 (1일 새벽)
    rules:
      - "SQLite 데이터베이스에 마크다운 전문 인덱싱"
      - "제목, 본문, 태그별 인덱싱"
      - "/output/search.db 생성"
    memory_usage: "효율적 - 키워드 검색만 로드"
    
  - name: "BacklinkMapper"
    role: "노트 간 역참조 맵 생성"
    script: build-backlinks.py
    schedule: "0 3 1 * *"   # 월 1회
    rules:
      - "[[노트]] 참조 역방향 추출"
      - "JSON 형식 그래프 생성: /output/backlinks.json"
      - "노드 간 관계도 시각화 데이터 제공"
    
  - name: "HtmlPublisher"
    role: "정적 웹사이트 생성 및 배포"
    script: generate-html.sh
    schedule: "0 4 * * *"   # 매일 (변경이 있을 때만 실행)
    rules:
      - "Pandoc으로 마크다운 → HTML 변환"
      - "CSS 템플릿 적용 (다크모드 지원)"
      - "검색 기능, 태그 페이지, 사이트맵 생성"
      - "/output/html/ 에 배포"
    
  - name: "VersionKeeper"
    role: "Git 자동 커밋 및 태그"
    script: git-auto-commit.sh
    schedule: "0 20 * * *"  # 매일 저녁
    rules:
      - "변경사항 감지 및 자동 커밋"
      - "커밋 메시지: Daily update: {DATE}"
      - "월간 태그: v{YYYY-MM} (1일)"
      - "GitHub에 자동 푸시"
    approval: true  # 버전 작업이므로 검증 필요
    
  - name: "BackupKeeper"
    role: "클라우드 및 로컬 백업"
    script: backup.sh
    schedule: "0 5 1 * *"   # 월 1회 (1일 새벽)
    rules:
      - "Rclone: ~/my-knowledge → Google Drive"
      - "로컬 압축: ~/backups/{YYYY-MM}.tar.gz"
      - "GitHub Releases에 월간 스냅샷 업로드 (선택)"
      - "외장 HDD 백업 리마인더 (분기별)"
    approval: true  # 데이터 백업이므로 검증 필요
```

### 3.3 메모리 누적 (Argo의 Vault 패턴)

```
기본 규칙:
1. 반복되는 결정/선호도 → /topics/ 또는 /areas/ 에 note로 저장
2. 검색할 때만 필요한 메모리 로드 (전체 재읽음 없음)
3. Backlink로 자동 연결 → 지식이 유기적으로 증가
4. 월간 요약 생성 → 패턴 인식 (예: 3D 프린터 구매 패턴)

예시:
- "3D 프린터는 Bambu Lab 선호" → /topics/3d-printer-research.md에 기록
- 다음 번 프린터 관련 노트에서 자동 링크
- 월간 "3D 프린팅 진행" 요약 생성
```

### 3.4 권한 및 승인 (Argo의 Approval 패턴)

```
저위험 작업 (자동 실행):
- 일일 노트 생성
- 검색 인덱스 생성
- HTML 변환

중위험 작업 (실행 전 검증):
- Git 커밋/푸시
- 파일 삭제 (없음, 하지만 규칙 위반 감지)
- 백업 시작

고위험 작업 (사용자 승인 필수):
- 클라우드 백업
- GitHub Releases 배포
- 중요 파일 삭제 (향후 기능)
```

---

## 4. 구현 계획

### Phase 1: MVP (Week 1-2)
**목표**: 기본 저장소 + 일일 자동화

- [ ] GitHub 저장소 생성
- [ ] 폴더 구조 설정
- [ ] `create-daily-note.py` 구현
- [ ] `validate-links.py` 구현
- [ ] `.gitignore` 설정
- [ ] cron 잡 설정 (일일 노트)

**결과**: 매일 자동으로 새로운 노트가 생성되고, 링크가 검증됨

### Phase 2: 연결 & 검색 (Week 3-4)
**목표**: 지식의 자동 연결 및 검색 기능

- [ ] `build-search-index.py` 구현 (SQLite)
- [ ] `build-backlinks.py` 구현 (JSON 그래프)
- [ ] CLI 검색 도구 (fzf + ripgrep 통합)
- [ ] Backlink 시각화 (D3.js, 선택)
- [ ] cron 잡 설정 (월 1회)

**결과**: 어떤 노트를 검색해도 관련 노트가 자동으로 나타남

### Phase 3: 퍼블리싱 (Week 5-6)
**목표**: 어디서든 접근 가능한 웹 뷰어

- [ ] `generate-html.sh` 구현 (Pandoc)
- [ ] CSS 템플릿 작성 (다크모드, 반응형)
- [ ] 검색 페이지 생성
- [ ] 태그 페이지 생성
- [ ] GitHub Pages 배포 설정

**결과**: 모바일에서도 접근 가능한 개인 위키

### Phase 4: 버전 & 백업 (Week 7-8)
**목표**: 신뢰할 수 있는 버전 관리 및 백업

- [ ] `git-auto-commit.sh` 구현
- [ ] 자동 태그 생성 스크립트
- [ ] Rclone 설정 (Google Drive)
- [ ] 로컬 백업 스크립트
- [ ] 백업 모니터링 대시보드 (선택)

**결과**: 매월 버전이 태그되고, 어디서나 복구 가능

### Phase 5: 고도화 (Week 9+)
**목표**: AI 기반 기능 추가

- [ ] Claude API 통합: 자동 요약 생성
- [ ] Prompt: "이 노트와 관련된 주제들을 추천해줘"
- [ ] 월간 리뷰 자동 생성
- [ ] 아이디어 클러스터링 (유사 노트 자동 그룹화)

---

## 5. 기술 스택

### Core
- **언어**: Python 3.11+, Bash
- **저장소**: Git (로컬), GitHub (원격)
- **마크다운**: 표준 마크다운 + [[위키링크]] 확장

### 자동화
- **스케줄링**: cron (Linux/macOS), Task Scheduler (Windows)
- **데이터베이스**: SQLite (검색 인덱스)
- **검색**: ripgrep, fzf

### 퍼블리싱
- **변환**: Pandoc (MD → HTML/PDF)
- **템플릿**: Jinja2 (HTML 생성)
- **배포**: GitHub Pages (정적 호스팅)

### 선택 (고도화)
- **그래프 시각화**: D3.js, Vis.js
- **AI 요약**: Claude API (anthropic-sdk)
- **웹 프레임워크**: Flask (웹 인터페이스, 선택)

### 백업
- **클라우드 동기화**: Rclone (Google Drive)
- **압축**: tar, gzip
- **스케줄링**: cron

---

## 6. 성공 지표

### 정량적
- ✅ 일일 노트 자동 생성 성공률: 100%
- ✅ 링크 검증 정확도: 100%
- ✅ Git 커밋 빠뜨림: 0%
- ✅ 월간 백업 성공: 100%
- ✅ 검색 응답 시간: < 1초

### 정성적
- ✅ 노트 작성 시 "어디에 저장할지" 고민 최소화
- ✅ 관련 노트가 자동으로 나타나는 경험 (serendipity)
- ✅ 지식이 "자동으로 연결되는 느낌"
- ✅ 어디서든 노트 접근 가능 (안심감)
- ✅ 자동화로 인한 수작업 90% 이상 감소

---

## 7. Argo에서 배운 핵심 설계 원칙

### 원칙 1: 로컬 우선, 클라우드는 선택
```
"로컬에서 완전히 작동하며, 클라우드는 백업일 뿐"
→ 종속성 없음, 완전한 소유권
```

### 원칙 2: 폴더 규모의 메모리
```
"한 번에 한 파일이 아니라 폴더 전체를 기억"
→ 컨텍스트 풍부, 관계도 파악
```

### 원칙 3: 자동 위임 (Delegation)
```
"사용자가 지시하면, 에이전트가 적절한 도구를 선택"
→ 매번 "어떤 스크립트를 쓸까?" 고민 없음
```

### 원칙 4: 승인 기반 실행
```
"위험한 작업은 실행 전에 사용자 동의 필요"
→ 실수로 인한 손실 방지
```

### 원칙 5: 지식의 자동 연결
```
"반복되는 결정이 자동으로 기억되고, 링크됨"
→ 지식이 "배합된다" (compounds)
```

---

## 8. 확장 가능성 (Future Features)

### 단기 (1-2개월)
- 웹 UI 대시보드 (Flask)
- 월간 리뷰 자동 생성
- Telegram/Slack 통합 (알림)

### 중기 (3-6개월)
- Claude API 통합 (자동 요약, 분류)
- 모바일 앱 (iOS, Android)
- 팀 협업 모드 (선택)

### 장기 (6개월+)
- 음성 노트 (Whisper 통합)
- 이미지 분석 (이미지를 노트로 변환)
- 관계도 2D/3D 시각화 (WebGL)
- 개인 LLM 파인튜닝 (로컬 모델)

---

## 9. 리스크 및 완화

| 리스크 | 영향 | 확률 | 완화 방안 |
|--------|------|------|---------|
| cron 잡 실패 | 자동화 중단 | 중 | 로그 모니터링, 주간 상태 리포트 |
| Git 푸시 실패 | 데이터 손실 | 낮 | 로컬 백업 (일일) + 클라우드 (월 1회) |
| 스크립트 의존성 변화 | 호환성 문제 | 중 | requirements.txt 관리, 버전 핀 |
| 대용량 저장소 성능 저하 | 검색 느림 | 중 | 아카이빙 (연도별 분리) |
| 클라우드 API 변경 | 백업 중단 | 낮 | Rclone 추상화, 다중 백업 (Google + HDD) |

---

## 10. 성공 사례 참고

### 유사 시스템들
- **Obsidian**: 로컬 마크다운 + 플러그인 → MyRemember가 더 자동화 중심
- **Logseq**: 일일 노트 + 그래프 → MyRemember가 더 스크립트 기반 커스터마이징
- **Argo**: 폴더 규모 메모리 + AI 에이전트 → MyRemember의 자동화 패턴 아이디어

### MyRemember의 차별성
✅ **완전히 로컬**, 오픈소스  
✅ **개발자 중심** (Python/Bash 자유도)  
✅ **자동화 극대화** (Argo의 crew 패턴 적용)  
✅ **지식의 자동 연결** (Backlink + 검색)  
✅ **어디서든 접근** (Git + 정적 웹사이트)

---

## 11. 다음 단계

1. **이번 주**: GitHub 저장소 생성, 폴더 구조 확정
2. **다음 주**: Phase 1 구현 시작 (일일 노트, 링크 검증)
3. **결과 검증**: 4주 후 실제 사용하며 피드백 수집
4. **고도화**: 사용자 피드백 기반 Phase 2-4 진행

---

## 부록: 마크다운 양식 예시

### /areas/project-name/README.md
```markdown
# 프로젝트명

**상태**: 진행 중 | 완료 | 보류  
**시작**: 2026-08-11  
**목표**: ...

## 개요
프로젝트 설명

## 진행 상황
- [ ] 태스크 1
- [x] 태스크 2

## 관련 노트
- [[topic-1]]
- [[topic-2]]

## 의사결정
- Decision 1: 이유
```

### /topics/topic-name.md
```markdown
# 주제명

**카테고리**: 관심사 | 학습 | 연구  
**마지막 업데이트**: 2026-08-11

## 개요
주제 설명

## 핵심 개념
- 개념 1
- 개념 2

## 학습 자료
- [링크 1]
- [[관련 노트]]

## 진행 중인 프로젝트
- [[프로젝트1]]
- [[프로젝트2]]
```

### /daily/2026-08-11.md
```markdown
# 2026-08-11

## 오늘의 주제
- 지식 관리 시스템 설계

## 진행 상황
- MyRemember PRD 작성 완료
- Argo 문서 학습

## 배운 점
- Crew 패턴을 자동화에 적용 가능
- 폴더 규모 메모리의 중요성

## 관련 노트
- [[knowledge-management-system]]
- [[ai-coding-workflow]]

## 내일 할 일
- [ ] GitHub 저장소 생성
- [ ] 첫 번째 스크립트 구현
```

---

**Document End**
