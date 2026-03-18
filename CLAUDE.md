# CLAUDE.md — DevCMD 프로젝트 설계서

> 스택별 개발 명령어 레퍼런스 · 개인 CLI 지식 베이스  
> 작성일: 2026-03-18 | 버전: 1.0.0

---

## 0. 프로젝트 개요

### 0.1 한 줄 소개

> "자주 쓰는 CLI 명령어를 스택별·태그별로 정리하고, 직접 추가·수정하며 쓰는 개인 명령어 레퍼런스 툴"

### 0.2 배경 및 목적

개발하다 보면 같은 명령어를 반복적으로 검색한다.  
`docker compose down -v` 가 뭐였더라, `alembic downgrade -1` 이 어디 있었지 — 이런 맥락 전환 비용을 줄이기 위한 툴.

단순 치트시트와 다른 점은:

- **언제 쓰는지** 설명이 함께 있다 (명령어 암기가 아닌 맥락 이해)
- **직접 추가·수정** 가능 (내 프로젝트·팀 특화 명령어 관리)
- **즐겨찾기** 로 자주 쓰는 것만 빠르게 접근
- 로컬 `localStorage` 저장 — 서버 불필요

### 0.3 현재 단계

| 단계 | 상태 | 설명 |
|------|------|------|
| v1.0 목업 | ✅ 완료 | 단일 HTML 파일, 80+ 내장 명령어 |
| v1.1 개선 | ✅ 완료 | 내보내기/가져오기 (JSON), 키보드 단축키, 스택 관리, 정렬 |
| v2.0 앱화 | 🔲 예정 | Tauri 데스크톱 앱 또는 Next.js + 로컬 DB |

---

## 1. 기능 명세

### 1.1 핵심 기능

```
F-01  스택 탭 필터     상단 탭으로 스택별 필터. 전체/Git/Docker/Django 등
F-02  태그 사이드바    좌측 사이드바에서 #setup #run #deploy 등 18개 태그 필터
F-03  통합 검색        명령어·설명·언제사용·태그 동시 검색 (실시간)
F-04  명령어 카드      제목 / 언제사용 / 코드 / 태그 4개 구역으로 구성
F-05  즐겨찾기         ★ 버튼 토글, localStorage 영구 저장, 즐겨찾기 전용 뷰
F-06  복사             ⎘ 버튼 클릭 → 클립보드 복사 → 1.5초 후 ✓ 표시
F-07  명령어 추가      모달 팝업 — 스택/카테고리/제목/언제/명령어/태그 입력
F-08  명령어 수정      ✎ 버튼 — 기존 내용 채워진 수정 모달
F-09  명령어 삭제      커스텀 명령어에만 ✕ 버튼 노출
F-10  카테고리 분류    스택 내 카테고리 단위로 소제목 그룹핑
```

### 1.2 비기능 요구사항

```
N-01  단일 파일        HTML/CSS/JS 모두 devcmd.html 한 파일
N-02  서버 불필요      localStorage만 사용, 오프라인 완전 동작
N-03  폰트 외부 의존   Google Fonts (Noto Sans KR, JetBrains Mono, Fraunces)
N-04  반응형           최소 1280px 기준 설계 (모바일 미지원)
N-05  접근성           tabindex, aria-label 미적용 (v1 스코프 외)
```

### 1.3 스택 목록 (11개)

| ID | 레이블 | 색상 | 내장 명령어 수 |
|----|--------|------|----------------|
| `git` | Git | `#d4500a` | 12 |
| `docker` | Docker | `#1d6fa4` | 9 |
| `django` | Django | `#1a7a50` | 9 |
| `fastapi` | FastAPI | `#0d7a78` | 7 |
| `nextjs` | Next.js | `#1c1a17` | 7 |
| `flutter` | Flutter | `#6340c0` | 7 |
| `python` | Python | `#a06a00` | 5 |
| `npm` | npm/yarn | `#c0302a` | 6 |
| `rust` | Rust | `#b8420a` | 5 |
| `linux` | Linux/CLI | `#4a4640` | 7 |
| `custom` | 커스텀 | `#8c877f` | 사용자 입력 |

### 1.4 태그 목록 (18개)

```
setup  run  build  deploy  db  migrate  test  debug
git  package  docker  env  ci  log  network  security  perf  custom
```

---

## 2. 데이터 구조

### 2.1 Command 객체

```typescript
interface Command {
  id:       number;       // 고유 ID (내장: 1~999, 커스텀: 1000+)
  stack:    StackId;      // 'git' | 'docker' | 'django' | ...
  category: string;       // 스택 내 카테고리 (예: '마이그레이션', '빌드')
  desc:     string;       // 명령어 제목 (카드 헤더)
  when:     string;       // 언제 사용하는지 설명
  cmd:      string;       // 실제 명령어 (멀티라인 허용)
  tags:     TagId[];      // 1개 이상의 태그
}
```

**예시**

```json
{
  "id": 62,
  "stack": "fastapi",
  "category": "Alembic (DB)",
  "desc": "마이그레이션 생성 → 적용 → 롤백",
  "when": "SQLAlchemy 모델 변경을 DB에 반영할 때. autogenerate가 모델과 DB 스키마 차이를 자동 감지.",
  "cmd": "alembic init alembic\nalembic revision --autogenerate -m \"add users\"\nalembic upgrade head\nalembic downgrade -1",
  "tags": ["db", "migrate"]
}
```

### 2.2 localStorage 키

| 키 | 타입 | 설명 |
|----|------|------|
| `dcmd_custom` | `Command[]` (JSON) | 사용자 추가 명령어 배열 |
| `dcmd_favs` | `number[]` (JSON) | 즐겨찾기 ID 배열 |
| `dcmd_nextid` | `string` | 다음 커스텀 ID 카운터 (초기값 2000) |

### 2.3 상태 변수 (런타임)

```javascript
let activeStack = 'all';   // 현재 선택된 스택 탭
let activeTag   = null;    // 현재 선택된 태그 (null = 전체)
let favOnly     = false;   // 즐겨찾기 전용 뷰 여부
let editId      = null;    // 수정 모달의 대상 ID (null = 신규)
let selTags     = [];      // 모달에서 선택 중인 태그 배열
```

---

## 3. UI 구조 / 목업

### 3.1 전체 레이아웃

```
┌────────────────────────────────────────────────────────────┐
│  HEADER                                                     │
│  ┌──────────┐ ┌──────────────────────┐ ┌────┐ ┌────────┐  │
│  │ DevCMD   │ │ ⌕  명령어, 설명 검색  │ │ ★  │ │ ＋추가 │  │
│  └──────────┘ └──────────────────────┘ └────┘ └────────┘  │
│  [전체] [Git] [Docker] [Django] [FastAPI] [Next.js] ...    │
├──────────┬─────────────────────────────────────────────────┤
│ SIDEBAR  │  CONTENT AREA                                    │
│          │                                                  │
│ 태그 필터 │  Git  ──────────────────────  12개             │
│          │                                                  │
│ ● 전체   │    기초                                          │
│ ─────    │    ┌──────────────┐ ┌──────────────┐            │
│ #setup   │    │ 저장소 초기화 │ │ 원격 저장소   │            │
│ #run     │    │ 언제: 새 프...│ │ 클론          │            │
│ #build   │    │ git init     │ │ 언제: GitHub  │            │
│ #deploy  │    │ #setup #git  │ │ git clone ... │            │
│ #db      │    └──────────────┘ └──────────────┘            │
│ #migrate │                                                  │
│ #test    │    커밋                                          │
│ #debug   │    ┌──────────────┐ ┌──────────────┐            │
│ #git     │    │ 변경 파일    │ │ 커밋          │            │
│ #package │    │ 스테이징     │ │               │            │
│ #docker  │    └──────────────┘ └──────────────┘            │
│ ...      │                                                  │
└──────────┴─────────────────────────────────────────────────┘
```

### 3.2 명령어 카드 구조

```
┌──────────────────────────────────────────────────┐
│  마이그레이션 생성 → 적용 → 롤백      [★][⎘][✎]  │  ← 헤더
│  언제: SQLAlchemy 모델 변경을 DB에 반영할 때.     │  ← when 설명
│        autogenerate가 모델과 DB 스키마 차이를...  │
│                                                  │
│  ║ alembic init alembic                          │  ← 코드 블록
│  ║ alembic revision --autogenerate -m "add"      │    (왼쪽 주황 border)
│  ║ alembic upgrade head                          │
│  ║ alembic downgrade -1                          │
│                                                  │
│  [#db] [#migrate]                                │  ← 태그
└──────────────────────────────────────────────────┘
```

**카드 아이콘 액션**

| 아이콘 | 동작 | 조건 |
|--------|------|------|
| ★ | 즐겨찾기 토글 | 전체 카드 |
| ⎘ | 클립보드 복사 → 1.5s 후 ✓ | 전체 카드 |
| ✎ | 수정 모달 오픈 | 전체 카드 |
| ✕ | 삭제 확인 후 제거 | 커스텀 명령어만 |

### 3.3 추가/수정 모달

```
┌──────────────────────────────────────────┐
│  명령어 추가                              │
│                                          │
│  스택 *           카테고리               │
│  [Django      ▾]  [데이터베이스        ]  │
│                                          │
│  명령어 제목 *                            │
│  [모델 필드 추가 마이그레이션           ]  │
│                                          │
│  언제 사용하나요?                         │
│  [모델에 새 필드를 추가한 후 DB에...    ]  │
│                                          │
│  명령어 *                                │
│  ┌──────────────────────────────────┐   │
│  │ python manage.py makemigrations  │   │
│  │ python manage.py migrate         │   │
│  └──────────────────────────────────┘   │
│                                          │
│  태그 선택                               │
│  [#setup] [#run] [#build] [#deploy] ... │
│  ┌──────────────────────────────────┐   │
│  │ [db ×] [migrate ×]               │   │
│  └──────────────────────────────────┘   │
│                                          │
│                       [취소] [저장]       │
└──────────────────────────────────────────┘
```

---

## 4. 화면 흐름 (User Flow)

```
앱 로드
  │
  ├─ localStorage 읽기
  │    ├─ dcmd_custom  → customCmds[]
  │    ├─ dcmd_favs    → favorites Set
  │    └─ dcmd_nextid  → nextId
  │
  ├─ renderTabs()      — 스택 탭 렌더
  ├─ renderTagBar()    — 태그 사이드바 렌더
  └─ renderAll()       — 카드 그리드 렌더
         │
         └─ filter 적용 (stack + tag + search + favOnly)
              └─ group by stack → group by category → 카드 렌더

[사용자 액션]

검색 입력
  └─ oninput → renderAll()

스택 탭 클릭
  └─ setStack(id) → renderTabs + renderTagBar + renderAll

태그 클릭
  └─ setTag(t) → renderTagBar + renderAll

즐겨찾기 토글 (카드)
  └─ tFav(id) → Set 업데이트 → persist → 뱃지 갱신

복사 버튼
  └─ tCopy(id, text) → clipboard API → 버튼 상태 변경

명령어 추가 버튼
  └─ openModal(null) → 빈 모달 오픈

수정 버튼 (카드)
  └─ doEdit(id) → openModal(cmd) → 채워진 모달 오픈

저장 버튼 (모달)
  └─ saveCmd() → validation → customCmds push/update → persist
       └─ renderTabs + renderTagBar + renderAll

삭제 버튼
  └─ doDel(id) → confirm → customCmds filter → persist → render
```

---

## 5. 파일 구조

### 5.1 v1 (현재 — 단일 파일)

```
devcmd.html          # 전체 앱 (HTML + CSS + JS 인라인)
CLAUDE.md            # 이 설계서
```

### 5.2 v2 (예정 — Next.js 앱)

```
devcmd/
├── CLAUDE.md
├── package.json
├── next.config.ts
├── tsconfig.json
│
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx            # 메인 뷰
│   │   └── globals.css
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.tsx      # 로고 + 검색 + 액션버튼
│   │   │   ├── StackTabs.tsx   # 상단 스택 탭
│   │   │   └── TagSidebar.tsx  # 좌측 태그 필터
│   │   │
│   │   ├── command/
│   │   │   ├── CommandCard.tsx # 카드 단위
│   │   │   ├── CommandGrid.tsx # 그리드 래퍼
│   │   │   └── GroupHeader.tsx # 스택 그룹 헤더
│   │   │
│   │   └── modal/
│   │       └── CommandModal.tsx # 추가/수정 모달
│   │
│   ├── data/
│   │   └── commands.ts         # 내장 명령어 배열 (CMDS)
│   │
│   ├── store/
│   │   └── useCommandStore.ts  # Zustand 상태 (필터 + 즐겨찾기 + 커스텀)
│   │
│   └── types/
│       └── index.ts            # Command, Stack, Tag 타입 정의
│
└── public/
    └── favicon.ico
```

---

## 6. 기술 스택

### 6.1 v1 (현재)

| 레이어 | 기술 |
|--------|------|
| 마크업 | HTML5 |
| 스타일 | CSS3 (Variables, Grid, Flexbox) |
| 로직 | Vanilla JS (ES2020) |
| 폰트 | Google Fonts (Noto Sans KR, JetBrains Mono, Fraunces) |
| 저장 | localStorage |
| 빌드 | 없음 (단일 HTML) |

### 6.2 v2 (예정)

| 레이어 | 기술 | 선택 이유 |
|--------|------|----------|
| 프레임워크 | Next.js 15 (App Router) | 기존 스택 일관성 |
| 언어 | TypeScript | 타입 안전성 |
| 스타일 | Tailwind CSS v4 | 유틸리티 클래스 속도 |
| 상태 관리 | Zustand | 경량, localStorage persist |
| DB (옵션) | SQLite + better-sqlite3 | 로컬 파일 기반 |
| 배포 | Tauri (데스크톱) 또는 Vercel | 인터넷 없이 사용 가능 |

---

## 7. 디자인 시스템

### 7.1 컬러 팔레트

```css
/* 배경 */
--bg:        #f7f6f2   /* 전체 배경 (따뜻한 크림) */
--surface:   #ffffff   /* 카드, 헤더, 사이드바 */
--surface2:  #f2f0eb   /* hover 배경, 뱃지 */

/* 경계선 */
--border:    #e2ddd5
--border2:   #cdc8be   /* hover border */

/* 텍스트 */
--text:      #1c1a17   /* 제목, 주요 텍스트 */
--text2:     #4a4640   /* 보조 텍스트 */
--text3:     #8c877f   /* 힌트, 레이블 */

/* 액센트 */
--accent:    #d4500a   /* 주황 — 버튼, 코드 border, 로고 */
--accent-bg: #fef3ec
--accent-bd: #f5c9a8
```

### 7.2 타이포그래피

| 역할 | 폰트 | 굵기 | 크기 |
|------|------|------|------|
| 로고 | Fraunces (이탤릭) | 600 | 18px |
| 그룹 제목 | Fraunces | 600 | 16px |
| 본문 | Noto Sans KR | 400/500/600 | 13–14px |
| 명령어 코드 | JetBrains Mono | 400/500 | 12px |
| 태그 | JetBrains Mono | 500 | 10–11px |
| 검색 입력 | JetBrains Mono | 400 | 12px |

### 7.3 태그 컬러 맵

```css
.t-setup    { bg: #edf5fc; text: #1d6fa4 }  /* 파랑 */
.t-run      { bg: #eaf5ef; text: #1a7a50 }  /* 초록 */
.t-db       { bg: #f0ecfc; text: #6340c0 }  /* 보라 */
.t-build    { bg: #fef8e6; text: #a06a00 }  /* 노랑 */
.t-deploy   { bg: #f0ecfc; text: #6340c0 }  /* 보라 */
.t-debug    { bg: #fcecea; text: #c0302a }  /* 빨강 */
.t-test     { bg: #e9f6f5; text: #0d7a78 }  /* 청록 */
.t-git      { bg: #fef3ec; text: #d4500a }  /* 주황 */
.t-package  { bg: #f4edfb; text: #7c3aed }  /* 인디고 */
.t-docker   { bg: #edf5fc; text: #1d6fa4 }  /* 파랑 */
.t-env      { bg: #eaf5ef; text: #1a7a50 }  /* 초록 */
.t-migrate  { bg: #f0ecfc; text: #6340c0 }  /* 보라 */
.t-ci       { bg: #fef3ec; text: #d4500a }  /* 주황 */
.t-log      { bg: #fef8e6; text: #a06a00 }  /* 노랑 */
.t-network  { bg: #e9f6f5; text: #0d7a78 }  /* 청록 */
.t-security { bg: #fcecea; text: #c0302a }  /* 빨강 */
.t-perf     { bg: #f4edfb; text: #7c3aed }  /* 인디고 */
.t-custom   { bg: #f2f0eb; text: #4a4640 }  /* 회색 */
```

---

## 8. 주요 로직

### 8.1 필터 파이프라인

```javascript
function renderAll() {
  const q = searchInput.value.toLowerCase().trim();

  let list = allCmds()           // CMDS (내장) + customCmds (사용자)
    .filter(c => {
      if (favOnly && !favorites.has(c.id))      return false;
      if (activeStack !== 'all' && c.stack !== activeStack) return false;
      if (activeTag && !c.tags.includes(activeTag))         return false;
      if (q && !matchesQuery(c, q))             return false;
      return true;
    });

  // 스택 순서 고정 → 카테고리 소그룹 → 카드 렌더
}

function matchesQuery(c, q) {
  return c.desc.toLowerCase().includes(q)
      || c.cmd.toLowerCase().includes(q)
      || (c.when || '').toLowerCase().includes(q)
      || c.tags.some(t => t.includes(q));
}
```

### 8.2 저장 / 불러오기

```javascript
// 저장
function persist() {
  localStorage.setItem('dcmd_custom', JSON.stringify(customCmds));
  localStorage.setItem('dcmd_favs',   JSON.stringify([...favorites]));
  localStorage.setItem('dcmd_nextid', String(nextId));
}

// 불러오기 (앱 초기화 시)
let customCmds = JSON.parse(localStorage.getItem('dcmd_custom') || '[]');
let favorites  = new Set(JSON.parse(localStorage.getItem('dcmd_favs') || '[]'));
let nextId     = parseInt(localStorage.getItem('dcmd_nextid') || '2000');
```

### 8.3 커스텀 명령어 저장 로직

```javascript
function saveCmd() {
  const payload = { stack, category, desc, when, cmd, tags: [...selTags] };

  if (editId !== null) {
    const idx = customCmds.findIndex(c => c.id === editId);
    if (idx >= 0) {
      // 커스텀 명령어 수정
      customCmds[idx] = { ...customCmds[idx], ...payload };
    } else {
      // 내장 명령어 수정 → 새 커스텀으로 추가 (원본 유지)
      customCmds.push({ id: nextId++, ...payload });
    }
  } else {
    // 신규 추가
    customCmds.push({ id: nextId++, ...payload });
  }

  persist();
}
```

> **참고:** 내장 명령어(CMDS)는 수정 불가. 수정 시 새 커스텀 항목으로 복사 추가됨.  
> 원본 내장 명령어는 항상 보존.

---

## 9. 에지 케이스 & 규칙

| 케이스 | 처리 방식 |
|--------|----------|
| 내장 명령어 수정 | 원본 유지, 별도 커스텀으로 추가 |
| 내장 명령어 삭제 버튼 | 노출 안 함 (✕ 버튼 커스텀 전용) |
| 검색 결과 없음 | "일치하는 명령어가 없습니다" 빈 상태 표시 |
| 태그 미선택으로 저장 | 허용 (tags: []) |
| 모달 외부 클릭 | overlay 클릭 시 모달 닫힘 |
| 커스텀 ID 충돌 | nextId는 2000부터 단조 증가, 삭제해도 재사용 안 함 |
| 멀티라인 cmd 복사 | 전체 줄바꿈 포함 복사 |
| cmd 안의 backtick | JS 템플릿 리터럴 이스케이프 `\\`` 처리 |

---

## 10. v1.1 개선 계획 (다음 스프린트)

### 10.1 추가 기능

```
F-11  JSON 내보내기    커스텀 명령어 + 즐겨찾기 → .json 다운로드
F-12  JSON 가져오기    파일 업로드 → 기존 데이터에 병합
F-13  키보드 단축키    / : 검색 포커스, Esc : 모달 닫기
F-14  스택 커스텀      사용자가 스택 추가 (이름 + 색상)
F-15  정렬 옵션        최근 추가순 / 이름순 / 즐겨찾기 우선
```

### 10.2 JSON 내보내기 스키마

```json
{
  "version": "1.1",
  "exported_at": "2026-03-18T00:00:00Z",
  "favorites": [1, 5, 62, 1002],
  "custom_commands": [
    {
      "id": 2000,
      "stack": "django",
      "category": "배포",
      "desc": "Gunicorn 실행",
      "when": "프로덕션 배포 시 Nginx 뒤에 Gunicorn을 붙일 때",
      "cmd": "gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3",
      "tags": ["run", "deploy"]
    }
  ]
}
```

---

## 11. v2 아키텍처 설계 (Next.js)

### 11.1 상태 설계 (Zustand)

```typescript
interface CommandStore {
  // 데이터
  customCmds: Command[];
  favorites:  Set<number>;
  nextId:     number;

  // UI 상태
  activeStack: string;
  activeTag:   string | null;
  favOnly:     boolean;
  searchQuery: string;

  // 파생 상태 (computed)
  filteredCmds: () => Command[];

  // 액션
  addCmd:      (cmd: Omit<Command, 'id'>) => void;
  updateCmd:   (id: number, cmd: Partial<Command>) => void;
  deleteCmd:   (id: number) => void;
  toggleFav:   (id: number) => void;
  setStack:    (stack: string) => void;
  setTag:      (tag: string | null) => void;
  setFavOnly:  (v: boolean) => void;
  setSearch:   (q: string) => void;
  exportJSON:  () => void;
  importJSON:  (file: File) => Promise<void>;
}
```

### 11.2 컴포넌트 계층

```
page.tsx
└── Layout
    ├── Header
    │   ├── Logo
    │   ├── SearchInput
    │   ├── FavButton
    │   └── AddButton
    ├── StackTabs
    ├── ContentLayout
    │   ├── TagSidebar
    │   └── CommandArea
    │       └── StackGroup[]
    │           └── CategoryGroup[]
    │               └── CommandGrid
    │                   └── CommandCard[]
    └── CommandModal (portal)
```

---

## 12. 개발 가이드라인

### 12.1 명령어 추가 시 규칙

```markdown
- `when` 필드는 반드시 작성 (없으면 카드 정보 부족)
- `when` 은 1~3문장. 언제 실행하는지 + 주의사항 포함
- `cmd` 멀티라인 허용 (관련 명령어 묶음)
- 주석은 `#` 으로 인라인 작성 가능
- 내장 ID: 스택별 20씩 증가 (git 1~19, docker 20~39, ...)
- 태그는 1개 이상 필수
```

### 12.2 CSS 변수 변경 시

```css
/* 다크 모드 추가 시 이 블록만 추가 */
@media (prefers-color-scheme: dark) {
  :root {
    --bg:      #0e0f11;
    --surface: #141517;
    /* ... */
  }
}
```

### 12.3 새 태그 추가 시

1. `ALL_TAGS` 배열에 ID 추가
2. `TAG_CLR` 객체에 색상 추가
3. CSS `.t-{tagId}` 클래스 추가
4. 기존 명령어에 필요한 경우 태그 반영

### 12.4 새 스택 추가 시

1. `STACKS` 배열에 `{ id, label, color }` 추가
2. 내장 명령어 `CMDS` 에 해당 스택 항목 추가
3. ID 범위 규칙 참고 (현재 linux = 180~, 다음 스택 = 200~)

---

## 13. 알려진 한계 및 TODO

```
TODO  내장 명령어 수정 시 원본과 중복 표시 문제 — 수정본이 추가되면서
      원본도 그대로 남음. 필터링 또는 override 맵 방식으로 개선 필요.

TODO  모바일 레이아웃 — 현재 1280px 기준, 모바일에서 사이드바 겹침

TODO  태그 커스텀 — 현재 18개 고정. 사용자 정의 태그 미지원

TODO  명령어 순서 변경 — drag & drop 지원 없음

TODO  다크모드 — CSS 변수 구조는 준비됐으나 토글 미구현

KNOWN  내장 명령어 ✎ 수정 후 저장 시 원본이 중복 남아있음
       → v1.1에서 editId가 내장 ID일 경우 customCmds에 override 마킹 추가 예정
```

---

## 14. 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|----------|
| 1.0.0 | 2026-03-18 | 최초 작성. 단일 HTML 목업, 80+ 내장 명령어, localStorage CRUD |

---


# CLAUDE.md — Project AI Instructions

> Claude가 이 프로젝트에서 코드 작업, 커밋 메시지 생성, PR 작성 시 참고하는 지침 문서입니다.

---

## 🧵 Git Commit Convention

### 형식

```
<이모지> <타입>(<스코프>): <한국어 요약>

<본문>

<푸터>
```

### 타입 & 이모지 매핑

| 이모지 | 타입       | 설명                            |
|--------|------------|---------------------------------|
| ✨     | feat       | 새로운 기능 추가                |
| 🐛     | fix        | 버그 수정                       |
| 💡     | chore      | 주석, 포맷 등 자잘한 수정       |
| 📝     | docs       | 문서 수정                       |
| 🚚     | build      | 빌드/패키지 관련 수정           |
| ✅     | test       | 테스트 코드 추가/수정           |
| ♻️     | refactor   | 기능 변화 없는 리팩터링         |
| 🚑     | hotfix     | 긴급 수정                       |
| ⚙️     | ci         | CI/CD 변경                      |
| 🔧     | config     | 설정 파일 수정                  |
| 🗑️     | remove     | 불필요 파일/코드 삭제           |
| 🔒     | security   | 보안 관련 수정                  |
| 🚀     | deploy     | 배포 관련 커밋                  |
| 🧩     | style      | 코드 스타일 변경                |
| 🎨     | ui         | UI/템플릿/CSS 변경              |
| 🔄     | sync       | 코드/데이터 동기화              |
| 🔥     | clean      | 코드/로그 정리                  |
| 🧠     | perf       | 성능 개선                       |

### 규칙

- 제목은 **한국어**, 50자 이내, 마침표 없음
- 본문 각 줄 72자 이내, 변경 이유 서술
- 하나의 커밋 = 하나의 타입
- 이모지 **필수** (생략 금지)
- Breaking Change → 푸터에 `BREAKING CHANGE:` 명시
- 이슈 연결 → `Fixes #N` 또는 `Refs #N`

### 예시

```
✨ feat(products): 상품 상세페이지 추가

- Bootstrap 기반 product_detail.html 추가
- views.py에서 Product 더미데이터 연결

Fixes #12
```

---

## 📌 Claude에게 요청할 때

커밋 메시지 작성을 요청할 경우, Claude는 위 컨벤션에 따라 메시지를 생성합니다.

프롬프트 예시:
> "변경된 내용을 보고 CLAUDE.md 커밋 컨벤션에 맞게 커밋 메시지 작성해줘."

*CLAUDE.md — DevCMD v1.0.0 | 한겨울*
