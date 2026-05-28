# 온보딩 앱 핸드오프 프롬프트

> 이 파일 전체를 다른 코딩 에이전트(Claude / GPT / Copilot 등)에게 그대로 붙여넣으면 동일한 앱이 만들어집니다.
> 시작 신호: **"아래 사양으로 `onboarding-app` 프로젝트를 새로 만들어줘. 모두 완료되면 헬스체크와 매칭 결과를 콘솔로 보여줘."**

---

## 1. 목표 (Why)

신입사원 온보딩을 위한 웹앱. 메인은 **체크리스트**이고, 각 항목 뒤에 **생활백서(Knowledge Vault)**가 자동으로 연결되어 클릭하면 본문이 보인다.

- **1차 (기본 온보딩)** — 체크리스트 항목 텍스트를 분석해 백서 노트를 키워드로 자동 매칭, 칩으로 표시. 체크 → 진행률 반영.
- **2차 (심화 온보딩, 스캐폴딩만)** — Open Question을 받으면 Vault 전체에서 답변을 찾는 Mock Agent. 나중에 RAG/LLM으로 교체할 수 있게 인터페이스만 잡아둔다.
- **지식 그래프** — Vault 노트들은 Obsidian처럼 `[[wiki-link]]`로 묶여 있고, 그 연결 구조를 시각화한다.

## 2. 기술 스택 (확정사항, 변경 금지)

| 영역 | 결정 |
| --- | --- |
| Frontend | React 18 + Vite + TypeScript |
| Backend | Express + TypeScript, dev 실행은 `tsx` |
| 데이터 저장소 | **로컬 파일** — Obsidian-style `.md` (frontmatter + `[[wiki-link]]`) |
| 상태 영속화 | `data/state/checklist-state.json` (JSON 파일) |
| 마크다운 렌더 | `react-markdown` + `remark-gfm` |
| Frontmatter 파서 | `gray-matter` |
| Dev 동시 실행 | `concurrently` |
| 포트 | client 5173 / api 3001, Vite proxy `/api → 3001` |

**DB·외부 LLM·인증 모두 사용하지 않는다.** Phase 2 Agent는 LLM이 아니라 키워드 검색이다.

## 3. 폴더 구조 (정확히 이대로)

```
onboarding-app/
├── package.json            # type: "module", scripts: dev / dev:server / dev:client / build / start:server
├── tsconfig.json           # ES2022, strict, jsx: react-jsx, include: [src, server, shared]
├── vite.config.ts          # react plugin + proxy
├── index.html
├── .gitignore              # node_modules, dist, data/state/
├── README.md
├── shared/
│   └── types.ts            # HandbookNote, ChecklistItem, LinkedNote, AgentAnswer, GraphPayload
├── server/
│   ├── server.ts           # Express, cache, routes
│   ├── vault.ts            # .md 로딩 + wiki-link 파서 + backlinks
│   ├── linker.ts           # 키워드 매칭 (가중치)
│   └── agent.ts            # Mock RAG (LLM 교체 지점)
├── src/
│   ├── main.tsx
│   ├── App.tsx             # 3-tab 셸 + 글로벌 HandbookPanel
│   ├── api.ts              # fetch 래퍼
│   ├── styles.css          # 단일 CSS (CSS 변수)
│   └── components/
│       ├── ChecklistView.tsx
│       ├── HandbookPanel.tsx
│       ├── Markdown.tsx     # [[wiki-link]] 인터셉터
│       ├── AgentChat.tsx
│       └── GraphView.tsx
└── data/
    ├── checklist.json
    └── vault/              # *.md 노트들
```

## 4. 공유 타입 (`shared/types.ts`)

```ts
export interface HandbookNote {
  id: string;            // path slug (e.g. "재택근무-정책과-신청")
  title: string;         // frontmatter.title || 첫 H1 || 파일명
  path: string;          // vault-relative (slash-separated)
  tags: string[];
  aliases: string[];
  summary: string;       // frontmatter.summary || 첫 단락(200자)
  body: string;          // frontmatter 제거된 본문
  outgoingLinks: string[]; // 해소된 id, 미해소 시 raw 레이블
  backlinks: string[];   // 이 노트를 가리키는 id들
}

export interface ChecklistItem {
  id: string;
  title: string;
  description?: string;
  handbookIds?: string[];     // 관리자 핀(선택)
  children?: ChecklistItem[];
}
export interface ChecklistSection { id: string; title: string; phase?: string; items: ChecklistItem[]; }
export interface Checklist { title: string; sections: ChecklistSection[]; }

export interface LinkedNote { noteId: string; title: string; score: number; reasons: string[]; }
export interface AgentSource { noteId: string; title: string; snippet: string; score: number; }
export interface AgentAnswer { question: string; answer: string; sources: AgentSource[]; }
export interface GraphPayload {
  nodes: { id: string; title: string; tags: string[] }[];
  edges: { source: string; target: string }[];
}
```

## 5. 백엔드 사양

### 5-1. `vault.ts` — 로더
- 재귀로 `data/vault/**/*.md` 스캔.
- `gray-matter`로 frontmatter 분리. body는 frontmatter 제거 결과.
- `id = path.replace(/\\/g,'/').replace(/\.md$/i,'')`.
- title 우선순위: `fm.title` → 첫 `# ...` → 파일명.
- `tags`, `aliases`는 배열 또는 콤마 문자열 둘 다 받게 정규화.
- summary: `fm.summary` → 첫 단락(헤딩·코드블록 제외, 200자 컷).
- **wiki-link 파서**: 정규식 `/\[\[([^\]\n]+?)\]\]/g`. `target|alias` 분리해 target만 outgoing으로.
- 1패스로 raw 수집 → 2패스로 outgoing 해소(소문자 비교: title/alias/파일명) + backlinks 채움. 해소 실패 시 raw 레이블을 그대로 둔다.

### 5-2. `linker.ts` — 매칭
- **토크나이저**: 소문자화 → `/[^\p{L}\p{N}\s]/gu`를 공백으로 → split. 2자 미만·stopwords 제거. 한국어(`/[\uAC00-\uD7AF]/`)이고 길이 ≥3이면 추가로 `prefix(len-1)`, `prefix(len-2)`도 토큰에 포함(조사 제거 효과).
- **stopwords**: 한국어 조사·일반어 + 영어 관사/전치사 (`및, 등, 의, 를, 을, 이, 가, 은, 는, 에, 에서, 으로, 와, 과, 도, 하기, 받기, 대한, 관련, 방법, 안내, 가이드, and, or, the, a, an, to, for, of, in, on`).
- **가중치**: title=5, alias=4, tag=3, heading=2, body=1. 각 신호는 토큰당 한 번만 카운트(첫 매칭으로 break).
- `handbookIds`로 핀된 항목은 +1000 (항상 상위), reason에 "관리자 지정" 라벨.
- `linkChecklistItem(title, description, pinned, notes, topN=3)`이 시그니처. `reasons`는 사람이 읽을 수 있는 한국어("제목: 휴가", "태그: 재택" 등).

### 5-3. `agent.ts` — Mock Agent
- `answerQuestion(question, notes, topN=3): AgentAnswer`.
- 빈 질문 처리: 예시 문구로 안내.
- 점수: 본문에 토큰 포함 +1, 제목에 포함 +3, 태그에 포함 +2.
- 스니펫: `body.split(/\n\s*\n/)` → 각 단락 trim → **헤딩(`/^#+\s/`)이 아닌 단락**만 후보 → 쿼리 토큰 포함하는 첫 단락 → 없으면 `summary` → 없으면 첫 단락. 240자 컷.
- 답변 포맷: lead 문서 1개를 강조 + 나머지를 "함께 보면 좋은 항목"으로 나열. 0건이면 "관련 백서를 찾지 못했어요…" 메시지.

### 5-4. `server.ts` — API
- `cors`, `express.json()`.
- 인메모리 캐시: `loadVault` + `checklist.json`을 1회 로드 후 재사용. `POST /api/dev/reload`로 비울 수 있다.
- 체크 상태는 `data/state/checklist-state.json`에 `{ [itemId]: true }`로만 저장(false면 키 삭제).

| Method | Path | 동작 |
| --- | --- | --- |
| GET | `/api/health` | `{ ok, notes, sections }` |
| GET | `/api/checklist` | checklist에 각 item별 `linked`(linker 결과)와 `done`(state) 합쳐 반환 |
| POST | `/api/checklist/state` | `{ id, done }`로 토글, state 파일에 영속화 |
| GET | `/api/notes` | 모든 노트 메타 (body 제외) |
| GET | `/api/notes/:id(*)` | 단일 노트 전체 (slash 포함 id 허용) |
| POST | `/api/agent/ask` | `{ question }` → `AgentAnswer` |
| GET | `/api/graph` | `GraphPayload` (해소된 엣지만) |
| POST | `/api/dev/reload` | 캐시 무효화 |

## 6. 프론트엔드 사양

### 6-1. 셸 (`App.tsx`)
- 좌측 브랜드(`온보딩 가이드`), 우측 3-탭(`1차 · 기본 온보딩`, `2차 · 심화 (에이전트)`, `지식 그래프`).
- 글로벌 `HandbookPanel`을 보유, 어느 탭에서든 `openNoteId`를 set하면 우측 슬라이드인 패널이 열린다.

### 6-2. `ChecklistView.tsx`
- 헤더에 진행률 텍스트(`X / Y 완료 · %`) + 그라데이션 진행률 바.
- 섹션별 카드(섹션 phase 배지 + 제목 + 아이템 목록).
- 항목: 좌측 커스텀 체크박스(완료 시 초록 + 흰 체크), 제목, 설명, 그 아래 "관련 생활백서" 라벨 + 칩들.
- 칩 hover 시 `reasons.join(' · ')`을 tooltip으로 노출.
- 체크 토글은 **optimistic UI**(즉시 반영, 실패 시 재조회).

### 6-3. `HandbookPanel.tsx`
- `position: fixed; right: 0`. 슬라이드인 애니메이션(0.18s).
- 헤더: eyebrow "생활백서" + 제목 + 태그 칩들 + 요약(좌측 보더 강조).
- 본문은 `Markdown.tsx`로 렌더.
- 푸터에 `outgoingLinks` / `backlinks` 두 섹션(각각 노트 버튼 목록, 미등록은 회색 처리).

### 6-4. `Markdown.tsx`
- 본문 문자열을 사전처리: `[[target]]` → `[target](wiki:${encodeURIComponent(target)})`, `[[target|alias]]` → `[alias](wiki:${encodeURIComponent(target)})`.
- `ReactMarkdown`의 `components.a` 커스터마이즈: `href`가 `wiki:`로 시작하면 `preventDefault` + `onWikiLink(decode)` 호출 + `.wiki-link` 클래스. 그 외는 새 탭으로.
- CSS로 `.wiki-link::before`/`::after`에 `[[ ]]` 표시.

### 6-5. `AgentChat.tsx`
- 안내 문구로 "Mock Agent / RAG 교체 지점" 명시.
- 입력 + 버튼, 샘플 질문 칩 4개(`재택근무 어떻게 신청해?`, `법인카드 한도가 얼마야?`, `회의실 노쇼 정책이 뭐야?`, `병가 처리 절차는?`).
- 답변 카드: 질문 → 답변(줄바꿈마다 `<p>`) → 출처 칩(클릭 시 패널 열림).
- 최근 답변이 위에 쌓이는 히스토리.

### 6-6. `GraphView.tsx`
- 노드를 원형으로 배치(`size=720`, `r=size/2-80`)한 SVG.
- 엣지는 단순 라인, hover 시 노드와 연결된 엣지/노드 강조.
- 상단에 전체 태그 칩 필터(단일 선택 + "전체").
- 노드 클릭 시 패널 열림.

### 6-7. CSS (`styles.css`)
- 단일 파일. CSS 변수로 `--bg, --surface, --border, --text, --muted, --brand(#6366f1), --brand-soft, --success, --shadow` 정의.
- 폰트: `system-ui` 스택 + 한글 폰트(`Apple SD Gothic Neo`, `Malgun Gothic`).
- 모던하고 부드럽게 — 그림자/라운드/그라데이션 진행률 바.

## 7. 시드 데이터

### 7-1. `data/checklist.json` — 3 섹션 / 12 항목

- **Day 1 (`day1`)**: `day1-badge`(출입증 수령 및 등록) · `day1-laptop`(노트북 셋업 및 사내 계정 로그인) · `day1-teams`(Teams 가입 및 팀 채널 입장) · `day1-intro`(팀원에게 자기소개)
- **Week 1 (`week1`)**: `week1-attendance`(근태 시스템 등록 및 출근 체크) · `week1-leave`(휴가 신청 방법 숙지) · `week1-meeting`(회의실 예약 연습) · `week1-expense`(경비처리·법인카드 가이드 읽기)
- **Month 1 (`month1`)**: `month1-mentor`(멘토와 1:1 미팅) · `month1-remote`(재택근무 신청 절차 확인) · `month1-security`(정보보안 교육 이수) · `month1-welfare`(복지 포인트 및 식대 사용 안내)

각 항목은 `description`도 한 줄 포함.

### 7-2. `data/vault/*.md` — 13개 노트 (모두 한국어, frontmatter 필수)

파일명은 한글-하이픈 형식. 모든 노트에 `tags`, `aliases`, `summary`를 frontmatter로 넣고 본문에 **최소 1개 이상의 `[[wiki-link]]`**를 포함해서 그래프가 비지 않게.

1. `출입증과-사옥-출입.md` — tags: 보안/출입증/첫출근/출근
2. `노트북-셋업과-사내-계정.md` — tags: IT/노트북/계정/첫출근/셋업
3. `사내-메신저-Teams.md` — tags: Teams/메신저/커뮤니케이션/첫출근
4. `근태-관리와-출근-시간.md` — tags: 근태/출근/퇴근/HR
5. `휴가-신청과-연차-관리.md` — tags: 휴가/연차/반차/병가/HR
6. `회의실-예약-시스템.md` — tags: 회의실/예약/미팅
7. `경비처리와-법인카드.md` — tags: 경비/법인카드/영수증/재무
8. `재택근무-정책과-신청.md` — tags: 재택/원격근무/유연근무/HR
9. `정보보안-교육과-서약.md` — tags: 보안/교육/서약/컴플라이언스
10. `사내-식당과-복지-포인트.md` — tags: 복지/식당/카페/식대
11. `멘토링-프로그램.md` — tags: 멘토/멘토링/온보딩/HR
12. `IT-장비-신청과-변경.md` — tags: IT/장비/신청/모니터/키보드
13. `방문자-응대-가이드.md` — tags: 방문자/게스트/보안/미팅

각 노트는 `## 발급/신청/정책 ...` 등 소제목 2~4개 + 다른 노트로의 `[[...]]` 2~4개. 내용은 가상 회사 규정 톤(시간·금액·연락처 포함)으로 그럴듯하게.

## 8. 의존성 (`package.json`)

```json
{
  "type": "module",
  "scripts": {
    "dev": "concurrently -k -n server,client -c blue,green \"npm:dev:server\" \"npm:dev:client\"",
    "dev:server": "tsx watch server/server.ts",
    "dev:client": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "start:server": "tsx server/server.ts"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^4.19.2",
    "gray-matter": "^4.0.3",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-markdown": "^9.0.1",
    "remark-gfm": "^4.0.0"
  },
  "devDependencies": {
    "@types/cors": "^2.8.17",
    "@types/express": "^4.17.21",
    "@types/node": "^20.14.10",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "concurrently": "^8.2.2",
    "tsx": "^4.16.2",
    "typescript": "^5.5.3",
    "vite": "^5.3.3"
  }
}
```

## 9. 수용 기준 (이게 다 OK여야 완료)

1. `npm install && npm run dev` 한 번에 실행되고 두 포트(5173, 3001) 모두 응답.
2. `GET /api/health` → `{ ok: true, notes: 13, sections: 3 }`.
3. `GET /api/checklist`에서 다음 매칭이 #1로 나와야 함:
   - "출입증 수령 및 등록" → **출입증과 사옥 출입**
   - "노트북 셋업 및 사내 계정 로그인" → **노트북 셋업과 사내 계정**
   - "Teams 가입 및 팀 채널 입장" → **사내 메신저 Teams**
   - "근태 시스템 등록 및 출근 체크" → **근태 관리와 출근 시간**
   - "휴가 신청 방법 숙지" → **휴가 신청과 연차 관리**
   - "회의실 예약 연습" → **회의실 예약 시스템**
   - "경비처리·법인카드 가이드 읽기" → **경비처리와 법인카드**
   - "멘토와 1:1 미팅" → **멘토링 프로그램**
   - "재택근무 신청 절차 확인" → **재택근무 정책과 신청**
   - "정보보안 교육 이수" → **정보보안 교육과 서약**
   - "복지 포인트 및 식대 사용 안내" → **사내 식당과 복지 포인트**
4. `POST /api/agent/ask` with `{ "question": "법인카드 한도는?" }` → answer에 "경비처리와 법인카드" 포함, sources[0].title === "경비처리와 법인카드". 스니펫이 `#`로 시작하지 않음(헤딩 제외).
5. `GET /api/graph` → `nodes.length === 13` 그리고 `edges.length >= 20` (해소된 wiki-link만 카운트).
6. UI에서 체크박스 토글 후 새로고침해도 상태 유지(`data/state/checklist-state.json`).
7. 칩 클릭 → 우측 패널 슬라이드인, 패널 안의 `[[다른 노트 제목]]` 클릭 → 같은 패널에서 해당 노트로 전환.
8. "지식 그래프" 탭에서 노드 13개가 원형으로 보이고, 노드 클릭 시 패널이 열림.

## 10. 하지 말아야 할 것

- ❌ 데이터베이스, ORM, Prisma 도입
- ❌ OpenAI / Azure OpenAI / 외부 LLM 호출 (Phase 2는 키워드 검색만)
- ❌ 인증/로그인
- ❌ Tailwind / styled-components / CSS-in-JS — 단일 `styles.css`만
- ❌ 상태관리 라이브러리(Redux/Zustand) — `useState`로 충분
- ❌ 라우팅 라이브러리 — 탭은 `useState`로
- ❌ 추가 컴포넌트 라이브러리(MUI/Chakra) — 디자인은 plain CSS

## 11. 확장 지점 (코멘트로 명시할 것)

- `server/agent.ts`의 `answerQuestion` 위에 "LLM/RAG 교체 지점" 주석.
- `server/linker.ts`의 토크나이저 위에 "형태소 분석기로 교체 가능(mecab-ko 등)" 주석.
- `src/components/GraphView.tsx` 상단에 "react-force-graph-2d로 교체 권장" 주석.

---

## 실행해서 보여줄 것

```pwsh
cd onboarding-app
npm install
npm run start:server   # 백그라운드
Invoke-RestMethod http://localhost:3001/api/health
$c = Invoke-RestMethod http://localhost:3001/api/checklist
$c.sections | ForEach-Object {
  Write-Host ('=== ' + $_.title + ' ===')
  $_.items | ForEach-Object {
    Write-Host (' • ' + $_.title + ' -> ' + (($_.linked | ForEach-Object { $_.title }) -join ', '))
  }
}
Invoke-RestMethod -Method Post -ContentType 'application/json' `
  -Body '{"question":"법인카드 한도는?"}' http://localhost:3001/api/agent/ask
```

이 출력이 "9. 수용 기준"을 모두 만족하면 완료.
