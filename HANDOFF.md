# HANDOFF — 숙제 리뷰 데스크 (hwarang)

이 문서는 지금까지 Claude Code 세션에서 개발한 내용을 다른 에이전트(ChatGPT Codex 등)가 이어받을 수 있도록 정리한 인수인계 문서다.
기준 시점: 2026-11-18, 커밋 `c4817ad` (게시된 페이지는 Artifact 버전 22, 페이지 안 표시는 `v21`).

> 이 문서에는 API 키, 토큰, 비밀번호, 학생 개인정보(이름, 점수, 답안 내용)를 적지 않았다. DB 수치는 건수만 적었다.

---

## 1. 프로젝트 목적

Verus Academy 의 IB 과목 교사가 쓰는 **숙제 채점·리뷰 도구**.

- 학생들은 구글 드라이브의 과목 폴더 › 학생 폴더에 숙제(구글 문서, PDF, 사진)를 올린다.
- 교사가 페이지에서 "지금 확인하기"를 누르면 새로 올라오거나 수정된 숙제를 찾는다. 이어서 Claude가 과목별 채점 기준과 답안지로 채점한다.
- 채점 결과는 점수(문제별 만점 기준), 요약, 잘한 점, 고칠 점, 다음 단계, 학생에게 보낼 코멘트, 문장별 코멘트다.
- 교사는 이 결과를 받은편지함 형태로 확인한다.
- 대상 과목: IB Economics Y1·Y2, Business Management Y1·Y2, Psychology Y1, Global Politics Y1, 그리고 개인 수업 폴더 1개. 9·10학년 Social Science 는 제외.

---

## 2. 전체 아키텍처

빌드 도구도 서버도 없는 **단일 HTML 파일**이다. claude.ai **Artifact** 로 게시되어 claude.ai 안에서만 동작한다.

```
[교사 브라우저: claude.ai 안의 Artifact 페이지 (review-desk/index.html)]
   │  window.claude.use('db')     → Artifact 전용 공유 DB (실시간 JSON 문서 저장소)
   │  window.claude.use('mcp')    → 교사 본인의 Google Drive 커넥터 (읽기 전용 4개 도구)
   │  window.claude.use('sample') → Claude 호출 (모델 이름이 아니라 티어만 선택: quick/default/complex)
   │  cdnjs pdf.js 3.11.174       → PDF 렌더링 (비교 화면, 그래프 crop, 스캔 PDF 이미지화, 답안지 PDF 텍스트 추출)
   ▼
 Google Drive (학생 제출물, 과제 폴더, 답안지)  ──읽기만──▶  Claude 채점  ──▶  DB 저장  ──▶  화면 실시간 갱신(onSnapshot)
```

- **인증:** 페이지는 키를 들고 있지 않다. Drive 와 Claude 호출은 모두 페이지를 연 사람(교사)의 claude.ai 계정·커넥터 권한으로 실행된다. 처음 쓸 때 claude.ai 가 동의를 받는다.
- **모델:** 채점은 `modelTier: 'complex'`(최상위 티어). 지시사항으로 문서를 고르는 필터만 `quick` 이다. 실제로 응답한 티어는 리뷰의 `modelTier` 에 기록된다.
- **런타임 계약 버전:** Artifact runtime contract **0.2.45** 에 고정돼 있다. 최신은 0.2.92 지만 일부러 올리지 않았다.
- **선언된 capabilities** (게시 시 저장되어 있음, 다시 게시할 때 생략하면 유지):
  - `db`
  - `sample`
  - `mcp`: Google Drive 커넥터의 `search_files`, `read_file_content`, `get_file_metadata`, `download_file_content`

### ⚠️ 이전(migration) 시 가장 중요한 제약

- 이 페이지의 `db` / `mcp` / `sample` API 는 **claude.ai Artifact 런타임 전용**이다. 일반 웹 서버나 로컬에서 열면 데이터 로드와 채점이 모두 동작하지 않는다. 화면 껍데기만 보인다.
- 운영 DB 도 claude.ai Artifact 저장소 안에 있다. 그래서 Codex 가 직접 읽고 쓸 수 없다.
- 선택지는 두 가지다.
  1. **소스만 Codex 에서 고친다.** 게시(publish)와 DB 조작은 claude.ai/Claude Code 로 한다. 코드 변경 없이 가능하다.
  2. **다른 플랫폼으로 옮긴다.** 아래를 갈아끼우는 큰 재작성이 필요하다.
     - `db` → Firebase, Supabase 등
     - `mcp` Drive → Google Drive API + OAuth
     - `sample` → Anthropic API 또는 OpenAI API + 서버 측 키 보관
     - 기존 데이터는 Claude Code 의 ArtifactData 도구 등으로 JSON 으로 내보낸 뒤 옮겨야 한다.

---

## 3. 폴더 구조

```
hwarang/
├── README.md                    # 기능 설명 (한국어, 교사용). 동작 방식·제한 사항이 자세히 있음
├── HANDOFF.md                   # 이 문서
├── AGENTS.md                    # 에이전트 작업 규칙
└── review-desk/
    ├── index.html               # 페이지 전체 소스 (~200KB, CSS + HTML + JS 한 파일)
    └── routine-prompt.md        # (미사용) 예전 정기 자동 채점 루틴 프롬프트. 루틴은 2026-09-12 에 껐음
```

`index.html` 내부 구성:

| 구간 | 줄 | 내용 |
| --- | --- | --- |
| `<head>` | 1–3 | Google Fonts (Archivo, Noto Sans KR) |
| `<style>` | 4–389 | `:root` 색 토큰, 다크 모드, 3단 레이아웃, 대화상자, 비교 화면 |
| 마크업 | 390–460 | 서랍 / 목록 / 상세 / 설정 패널 / 대화상자 컨테이너 |
| pdf.js | 461 | cdnjs `pdf.js/3.11.174/pdf.min.js` |
| `<script>` | 462–1998 | ES5 스타일 단일 IIFE |

주요 함수 위치 (커밋 `c4817ad` 기준):

| 함수 | 줄 | 역할 |
| --- | --- | --- |
| `submitKind` | 468 | mimeType 에서 제출 형태 판별: doc / pdf / image |
| `cfg` | 507 | `config/main` 에 기본값을 채워 반환 |
| `criteriaFor` | 522 | 폴더 이름으로 과목별 채점 기준 매칭 |
| `guessAssignment` | 543 | 제출 시각과 Due 날짜로 과제 추정 |
| `subscribe` | 568 | DB 컬렉션 실시간 구독 |
| `render` | 615 | 서랍·목록·상세 렌더 |
| `keyBlock` | 768 | 상세 화면의 "채점에 쓴 답안지" 칸 |
| `bindUI` | 795 | 메인 화면 이벤트 |
| `openSettings` / `saveSettings` | 870 / 905 | 설정 패널 (과목 폴더, 채점 기준, 답안지 폴더 등) |
| `scanFolder` | 939 | 과목 폴더 › 학생 폴더 › 제출물 수집, 자료실 › Homework › Due 폴더에서 과제 수집 |
| `buildPrompt` | 986 | 채점 프롬프트 조립 |
| `keysHtml` / `bindHtml` | 1050 / 1248 | 확인 창의 답안지 선택 UI, 반·주 연결 UI |
| `docxText` / `pdfKey` / `prepareKeys` | 1122 / 1128 / 1146 | 답안지 텍스트·이미지 준비 |
| `keysPromptText` / `keyStatus` | 1189 / 1201 | 답안지 프롬프트, 사용한 답안지 검증 |
| `docBoundRefs` / `saveBindings` | 1220 / 1221 | 같은 반·같은 주 답안지 재사용 |
| `imageCapability` | 1281 | `sample.limits().images` 확인 |
| `openRunDlg` / `renderRunDlg` | 1285 / 1311 | 확인 창 |
| `scanTargets` / `findDocs` / `filterPrompt` | 1379 / 1397 / 1427 | 폴더 스캔, 후보 문서, 지시사항 필터 |
| `startPicked` / `reviewList` / `runReview` | 1439 / 1455 / 1491 | 채점 실행, 다시 채점 |
| `openFb` / `saveFb` | 1516 / 1559 | 시스템 피드백 모달 |
| `imageRects` / `pageImages` | 1573 / 1595 | PDF 그림 영역 crop, 페이지 이미지 생성 |
| `parseJsonText` / `reviewOne` | 1655 / 1661 | 응답 파싱, 문서 한 건 채점·저장 |
| `anchorAll` / `fetchRaw` / `fetchPdf` | 1724 / 1748 / 1759 | 비교 화면의 문장 위치 매칭, Drive 바이트 받기 |
| `openCompare` / `loadPages` | 1771 / 1810 | "문서 보며 코멘트 달기" 비교 화면 |

---

## 4. 핵심 기능

1. **받은편지함형 3단 화면**
   - 왼쪽 서랍: 확인 안 함 / 확인함, 과목 필터, 마지막 확인 시각, "지금 확인하기"
   - 가운데: 제출일별 목록
   - 오른쪽: 리뷰 상세
   - 항목을 열면 그 순간 "확인함"으로 옮겨진다.
2. **확인 창 ("지금 확인하기")**
   - 과목·학년 선택 (Y1 전체 같은 묶음 버튼 포함)
   - 이번 확인 지시사항 (선택)
   - 답안지 선택
   - 최근 N일, 이미 리뷰한 문서도 다시 채점, 제외 제목 빼기
   - "문서 찾기" → 찾은 문서 체크 → 채점 시작
   - 지시사항이 있으면 빠른 모델이 대상 문서를 미리 골라 둔다.
3. **채점 프롬프트 우선순위** (위가 아래보다 우선):
   1. 시스템 피드백
   2. 이번 지시사항
   3. 답안지
   4. 과목별 채점 기준
   5. 공통 rubric
   - 과제 목록(Due 폴더)을 주고 어느 과제의 답인지도 정하게 한다.
4. **문제별 만점:** 학생 문서나 과제 파일의 배점([2], [6] 등)에서 읽는다. 없으면 문제 유형으로 추정한다. `maxScore` 와 `scoreBasis` 로 저장된다.
5. **그래프 보기:** 구글 문서를 PDF 로 내보내 pdf.js 로 그린다. 그 위에서 그림 영역은 고해상도로 잘라내고, 페이지 전체 이미지와 함께 Claude 에게 보낸다 (문서당 최대 10장). 단 이미지를 보낼 수 있는 화면에서만 가능하다 (7절 참고).
6. **PDF·사진 제출물 채점:**
   - 스캔·손글씨 PDF 는 페이지 전체를 이미지로 보낸다.
   - 사진은 원본을 그대로 보낸다.
   - 이미지를 못 보내면 텍스트만으로 채점하지 않는다. 대신 `status: needs_vision` 인 "채점 대기 · 세션 채점 필요" 항목으로 남긴다.
7. **답안지 (모범답안·markscheme)**
   - 드라이브 답안지 폴더에서 고르거나 파일(PDF, 이미지, Word, 텍스트)을 올린다.
   - 올린 파일의 텍스트는 `answerKeys` 에 저장된다.
   - Claude 는 실제로 쓴 답안지 제목, 문제 매칭 근거, 채점 요소별 충족·부분·빠짐 체크리스트를 돌려준다.
   - 페이지가 그 제목을 보낸 목록과 대조해 `answerKeyStatus` 로 저장한다.
8. **같은 반·같은 주 답안지 재사용:** `keyBindings/<과목폴더ID>_<주 월요일>` 에 저장된다. 이후 그 반 그 주 숙제를 채점하거나 다시 채점할 때 자동으로 붙는다. 주는 Due 날짜 기준이고, Due 가 없으면 제출일 기준(KST 월~일)이다.
9. **시스템 피드백 모달 (좌측 하단):**
   - 교사가 채점 방식 지적을 적으면 다음 채점부터 프롬프트 맨 앞에 들어간다.
   - 과목 범위를 지정하거나 보던 리뷰를 첨부할 수 있다.
   - 켜기/끄기, 삭제가 가능하다.
10. **문서 보며 코멘트 달기 (비교 화면):**
    - Drive PDF 내보내기를 페이지 그대로 렌더한다.
    - 문장별 코멘트를 구글 문서 댓글처럼 여백에 배치하고, 본문을 하이라이트한다.
    - 코멘트 수정·삭제·복사, 드래그로 새 코멘트 추가가 된다.
11. **설정 패널:** 과목 폴더, 건너뛸 하위 폴더, 채점 제외 제목 정규식, 최근 N일, 최소 글자 수, rubric, 과목별 채점 기준, 답안지 폴더, 자료실·Homework 폴더 이름 패턴을 편집한다.
12. **다시 채점:** 상세 화면 버튼. 그 리뷰에 썼던 답안지와 반·주 연결 답안지를 다시 쓴다.

---

## 5. 구현 완료 / 미완료

### 완료
- 1~12의 모든 기능
- 버튼으로만 확인 (정기 자동 실행 꺼짐)
- IA / Task 3 / Internal Assessment / Supporting Document / Feedback 제목은 채점 제외
- 9·10학년 제외
- 미제출자 보기 제거 (교사 요청)
- Economics 채점 기준에 학교 교사 피드백 반영: price mechanism 의 rationing, signalling, incentive, market-clearing 요소와 감점 상한
- 답안지 사용 여부 투명화, 반·주 단위 재사용 (2026-10-28, v21)

### 미완료 / 보류
- **페이지 안 이미지 전송:** 교사 화면에서 `sample.limits().images` 가 없다. 그래서 그래프와 PDF·사진을 페이지 안에서 볼 수 없다. 런타임 계약을 0.2.92 로 올리면 될 수도 있지만 검증하지 않았다.
- **`needs_vision` 항목 처리:** 지금은 사람이 Claude Code 세션에 "채점해 줘"라고 해야 한다. 세션이 Drive 에서 PDF 를 받아 이미지로 보고, 같은 리뷰 문서에 `source: "session"` 으로 저장하는 수동 절차다.
- **자동 테스트가 저장소에 없다.** Playwright 하네스는 세션 임시 폴더에만 있었다 (8절).
- 여러 장 사진 제출을 한 숙제로 묶어 채점하는 기능
- Word(.docx) 학생 제출물, iPhone HEIC 사진 지원
- 오래된 리뷰 정리 / 아카이브 기능

---

## 6. 알려진 버그·제약

| 구분 | 내용 |
| --- | --- |
| 이미지 | 교사 화면은 Claude 에게 이미지를 못 보낸다. 운영 리뷰 51건 중 그래프 이미지를 실제로 본 리뷰는 1건(세션 채점)뿐이다. 나머지는 graph 를 "텍스트로는 확인 불가"로 처리했다. |
| 과거 버그 (수정됨) | 인쇄된 문제지 텍스트가 300자 넘게 들어 있는 PDF 가 스캔본으로 판정되지 않았다. 그래서 문제지 글자만 보고 0점을 줬다. v21 에서 PDF·사진은 이미지 없이 절대 채점하지 않도록 고쳤고, 해당 리뷰는 세션에서 다시 채점했다. |
| 실행 기록 | 채점 중 탭을 닫으면 `runs` 문서가 `status: "running"` 으로 남는다 (현재 3건). 정리 로직이 없다. |
| 일시 오류 | `no reply from shell`, `the call was held ...` 가 가끔 난다. 한 번 재시도하고, 그래도 실패하면 run 의 errors 에 남는다. |
| 답안지 | 올린 이미지 답안지는 이미지가 저장되지 않는다 (텍스트만 `answerKeys` 에 저장). 다시 채점하거나 반·주 재사용할 때 이미지 전용 답안지는 빠진다. 같은 파일을 두 번 올리면 `answerKeys` 문서가 중복으로 생긴다. |
| 반·주 연결 | 확인 창에서 체크해 둔 모든 반에 연결된다. 과목을 여러 개 고른 채 답안지를 연결하면 다른 과목에도 붙는다. Claude 가 문제가 안 맞으면 `nomatch` 로 처리하긴 한다. |
| 비교 화면 | PDF·사진 제출물은 `docText` 가 추출 텍스트(또는 비어 있음)라 문장 위치 하이라이트가 안 된다. |
| 제목 제외 | 기본 제외 정규식에 `feedback` 이 있어서, 제목에 feedback 이 들어간 정상 숙제도 건너뛴다. |
| 동시성 | DB 는 last-writer-wins 다. 두 탭에서 동시에 채점하면 덮어쓸 수 있다. 설정도 페이지 저장이 마지막 값으로 덮는다. |
| 학생 이름 | 학생 폴더 이름에서 수업 이름을 빼서 만든다. 폴더 이름 규칙이 다르면 이름이 이상하게 나온다. |
| 버전 표시 | 페이지 헤더는 `v21`, Artifact 버전은 22 다. 22 는 21 에 "Claude Code 세션 채점" 라벨 한 줄만 더한 것이다. |
| README | README 첫 줄의 게시 URL 은 예전 형식이다. 현재 주소는 9절 참고. |

---

## 7. 최근 변경사항 (최신순)

| 날짜 | 커밋 | 내용 |
| --- | --- | --- |
| 2026-10-28 | `c4817ad` | 답안지 사용 표시. Claude 응답에 `answerKeyUsed`, `answerKeyMatch`, `keyChecklist` 추가, 페이지가 검증해 `answerKeyStatus` 저장. `keyBindings` 로 같은 반·같은 주 재사용. PDF·사진은 이미지 없이 채점 금지(`needs_vision`). |
| 2026-10-28 | `338f75d` | 학생 폴더 안 PDF·사진 채점. 스캔 PDF 는 페이지 이미지, 사진은 원본을 보냄. |
| 2026-10-26 | `3817108` | 확인 창 답안지: Drive 답안지 폴더 선택 + 파일 업로드 (docx 는 브라우저 내장 DecompressionStream 으로 해석). |
| 2026-09-28 | `af17f90` | Economics 기준에 학교 교사 피드백 반영 (price mechanism 요소·상한), Feedback 제목 제외. |
| 2026-09-25 | `90a663f` | 확인 창(과목·학년, 지시사항), 시스템 피드백 모달, 최상위 티어 채점, 그래프 crop. |
| 2026-09-19 | `3ff4cb1` | 페이지 이미지로 graph 보기 시도, 10점 문항 결론 문단 요구 삭제. |
| 2026-09-12 | `dea3a75` | 정기 자동 루틴 끔. 버튼으로만 확인. |
| 2026-09-11 | 여러 커밋 | 미제출자 보기 제거, 과목별 기준, 문제별 만점, IA/Task 3 제외, 비교 화면, 코멘트 말투 규칙, 받은편지함 재구성 |
| 2026-09-10 | `b128361` | 최초 페이지 + 정기 루틴 |

DB 에서 v21 이후 실제 사용 흔적도 있다. 2026-10-29 과 2026-11-04 에 수동 확인을 했고, `keyBindings` 1건이 생성됐다. 설정 문서는 버전 14 로, 교사가 페이지에서 계속 편집하고 있다.

---

## 8. 실행 / 테스트 / 빌드

- **빌드:** 없음. `review-desk/index.html` 하나가 전부다. npm 패키지, 번들러, 환경 파일도 없다.
- **실행:** claude.ai 에 로그인해 게시된 Artifact 를 연다 (9절). 로컬에서 파일을 열면 "claude.ai 안에서 열어야 리뷰 데이터를 불러올 수 있습니다" 안내만 나온다.
- **문법 확인:**

  ```bash
  node -e "const s=require('fs').readFileSync('review-desk/index.html','utf8');(s.match(/<script>([\s\S]*?)<\/script>/g)||[]).forEach(x=>new Function(x.replace(/<\/?script>/g,'')));console.log('ok')"
  ```

- **동작 테스트 (지금까지 쓴 방식):**
  - Playwright + Chromium 으로 페이지를 띄운다.
  - `addInitScript` 로 가짜 `window.claude` 를 주입한다. `use('db'|'mcp'|'sample')` 이 메모리 DB, 가짜 Drive 트리, 고정 JSON 을 돌려주는 sample 을 반환하게 만든다.
  - `page.route('https://cdnjs.cloudflare.com/**')` 로 pdf.js 를 로컬 `pdfjs-dist@3.11.174` 파일로 바꿔치기한다.
  - 시나리오별로 확인 창 조작 → 채점 → DB·프롬프트·화면 텍스트를 검증한다.
  - 이 하네스는 세션 임시 폴더에만 있었고 저장소에는 없다. 다시 만들 때 확인했던 시나리오는 이렇다.
    - 답안지 폴더 추가와 선택, 파일 업로드(txt, docx, pdf), 프롬프트에 답안지 포함
    - PDF 그래프 crop 이미지 수, 스캔 PDF 페이지 이미지, 사진 원본 전송
    - 이미지를 못 보낼 때 PDF 는 `needs_vision`, 문서는 graph 확인 불가 문구
    - 답안지 used / nomatch 표시, 체크리스트, 다른 답안지 목록
    - 반·주 연결 저장, 다음 확인에서 자동 첨부, 다시 채점 시 중복 없음, × 로 해제
    - 시스템 피드백 저장·켜기/끄기·프롬프트 반영
    - 지시사항 필터 (`sample.json`, quick 티어)
- **게시 후 확인:** claude.ai 에서 페이지를 열어 확인 창을 띄운다. 새 컬렉션이 생겼다면 DB 에서 한 번 읽어 확인한다.

---

## 9. 배포 구조

- **형태:** claude.ai Artifact (비공개, 소유자만 열람). 별도 서버·호스팅·CI 없음.
- **현재 주소:** https://claude.ai/artifact/PAGQNsjVppv8rPS2NMX5m4 (예전 형식 주소 `https://claude.ai/code/artifact/b3743ca6-bd7f-4fd2-8b24-fcf536678b5b` 와 같은 페이지)
- **현재 버전:** Artifact 버전 22, 2026-10-28 게시. 그 뒤로 다시 게시하지 않았다.
- **게시 방법:**
  - Claude Code 의 Artifact 도구로 `review-desk/index.html` 을 같은 URL 에 publish 한다.
  - 이때 `capabilities` 를 생략하면 기존 선언이 유지된다. 빈 객체를 넘기면 전부 해제되니 금지다.
  - 계약 버전도 생략하면 0.2.45 가 유지된다.
  - 다시 게시해도 DB 데이터는 그대로다.
- **예약 작업:** 숙제 리뷰 관련 예약 작업은 없다. 계정에 다른 예약 작업("IA 기사 스크랩", 매일)이 하나 있지만 이 프로젝트와 무관하다.

---

## 10. 데이터베이스 구조 (Artifact `db`, 운영 중)

문서 ID 와 필드. 건수는 2026-11-18 기준이다.

| 컬렉션 | 건수 | 문서 ID | 주요 필드 |
| --- | --- | --- | --- |
| `config` | 1 (`main`) | `main` | 아래 config 표 참고 |
| `reviews` | 51 | Drive fileId | 아래 reviews 표 참고 |
| `runs` | 23 | `run_<ISO 시각>` | `id, source, startedAt, finishedAt, status(running/done), checked, reviewed, skipped, errors[{title,message}], note, answerKeys[제목]` |
| `roster` | 7 | 과목 폴더 ID | `folderId, folderName, students[{name, folderId, folderName, url}], updatedAt` |
| `assignments` | 7 | 과목 폴더 ID | `folderId, folderName, items[{id, name, due(YYYY-MM-DD), url, docs[{id,title}], prompt}], updatedAt` |
| `feedback` | 1 | `fb_<ISO>` | `id, text, scope(과목 폴더 ID 또는 빈 값), status(active/off), createdAt, context{fileId, student, title, folderId, summary, score, maxScore}` |
| `answerKeys` | 3 | `key_<ISO>_<rand>` | `id, title, text(최대 60,000자), hasImages, createdAt, source:'upload'` |
| `keyBindings` | 1 | `<과목폴더ID>_<주 월요일 YYYY-MM-DD>` | `id, folderId, folderName, week, keys[참조], createdAt, updatedAt` |

**`config/main` 필드:**
- 과목·폴더: `folders[{id,name,label}]`, `exclude[]` (학생 폴더로 보지 않을 하위 폴더 이름 조각; 기본 자료실, 비공개)
- 제외·기간: `skipTitles` (정규식), `sinceDays`(14), `minChars`(40)
- 채점 기준: `rubric`, `criteria[{match, text}]` (Economics, Business, Psychology, Global Politics)
- 답안지·과제 폴더: `answerFolders[{id,name}]` (현재 비어 있음), `materialsPattern`(자료실), `homeworkPattern`(Homework)
- `updatedAt`

**`reviews/<fileId>` 필드:**
- 식별: `fileId, title, student, folderId, folderName, subfolder, subfolderId, subfolderUrl, viewUrl, kind(doc/pdf/image), mimeType`
- 과제: `assignmentId, assignmentName, assignmentDue`
- 실행 정보:
  - `modifiedTime, reviewedAt`
  - `source`: manual / scheduled / session
  - `revision`, `seen`, `seenAt`
  - `status`: reviewed / empty / needs_vision
- 채점 결과: `maxScore, scoreBasis, score, completeness(complete/partial/empty), summary, strengths[], improvements[], nextSteps[], flags[], comment, wordCount, docText(최대 30,000자), annotations[{quote, note}]`
- 모델·이미지: `modelTier, graphImages(보낸 이미지 수), graphNote(못 보낸 이유)`
- 답안지:
  - `answerKeys[참조]`, `answerKeyUsed`, `answerKeyMatch`, `keyChecklist[{point, met: yes/partial/no}]`
  - `answerKeyStatus`: used / nomatch / unknown / unread / none

**답안지 참조 형식:**
- Drive 파일: `{source:'drive', fileId, title, kind, mime}`
- 업로드: `{source:'saved', keyId, title}`
- 반·주 연결로 붙은 것은 `via:'bound'` 가 추가된다.

**호환성:** 오래된 리뷰에는 `kind`, `modelTier`, `graphImages`, `answerKeyStatus` 등이 없다 (51건 중 48건 `kind` 없음 = 구글 문서). 화면은 필드가 없어도 동작해야 한다.

**읽기·쓰기 경로:**
- 페이지: `db.doc(...).set/update/delete`, `db.collection(...).onSnapshot`
- Claude Code 세션: ArtifactData 도구 (get/list/query/set/update/batch, `if_version` 으로 버전 고정)

---

## 11. 외부 서비스 의존성

| 서비스 | 용도 | 비고 |
| --- | --- | --- |
| claude.ai Artifact 런타임 (contract 0.2.45) | 페이지 호스팅, `db`, `mcp`, `sample` | 필수. 이 밖에서는 동작 안 함 |
| Claude (`sample`, 티어 complex / quick) | 채점, 지시사항 필터 | 비용은 페이지를 연 사람 계정에 청구 |
| Google Drive 커넥터 (claude.ai 계정에 연결) | 폴더 목록, 문서 텍스트, 메타데이터, 파일·PDF 내보내기 다운로드 | 읽기만 한다. `needs_reauth` 오류가 나면 claude.ai 설정 › 커넥터에서 재연결 |
| cdnjs `pdf.js` 3.11.174 | PDF 렌더·텍스트 추출 | worker 도 cdnjs |
| Google Fonts | Archivo, Noto Sans KR | |
| 브라우저 API | `DecompressionStream`(docx), canvas, `localStorage`(보기·과목 필터 기억) | |
| Claude Code 세션 (수동) | 게시, DB 직접 조작, `needs_vision` 채점 | Codex 로는 대체 불가 |

### 환경변수
**없음.** 페이지와 저장소 모두 환경변수나 `.env` 를 쓰지 않는다. 인증은 claude.ai 계정과 커넥터가 처리하고, 키는 페이지에 들어가지 않는다.

다른 플랫폼으로 옮긴다면 새로 필요할 이름은 예를 들면 이렇다 (값은 문서화하지 말 것).
- Google OAuth 클라이언트 ID·시크릿: Drive 읽기
- LLM API 키: 채점
- DB 접속 정보: 리뷰 저장

---

## 12. 현재 상태 보고 (2026-11-18)

- **Git:**
  - 브랜치 `claude/cool-sagan-ogc2zd` 하나뿐이다. 원격 기본 브랜치(HEAD)도 이 브랜치다.
  - 작업 트리는 깨끗했고(이 문서 추가 전), `origin` 과 같은 커밋(`c4817ad`)이다.
  - 커밋 21개, 태그 없음, PR 없음.
- **원격 저장소:** 있음. `https://github.com/jkim-ux/hwarang`
- **배포 환경:** claude.ai Artifact 버전 22 가 운영 중이다 (비공개).
- **운영 DB 연결:**
  - 저장소 코드에는 DB 접속 설정이 없다. 페이지가 claude.ai 런타임을 통해 연결된다.
  - 이 문서를 쓴 Claude Code 세션은 소유자 권한으로 운영 DB 를 읽을 수 있었다. 문서 작성을 위해 건수 집계용 **읽기만** 했고, 아무것도 쓰지 않았다.
  - Codex 환경에서는 운영 DB 에 접근할 수 없다.

---

## 13. 다음 개발 우선순위 (제안)

1. **이미지 전송 해결:** 런타임 계약을 0.2.92 로 올린 테스트 사본 Artifact 를 따로 게시한다. 거기서 `sample.limits().images` 가 생기는지 확인한다. 운영 페이지는 확인 뒤에만 올린다.
2. **`needs_vision` 처리 흐름:** 이미지 전송이 안 된다면, 세션 채점 대기 목록을 한 번에 넘기는 버튼이나 안내를 만든다. 다른 방법은 이미지 전송이 되는 다른 경로를 쓰는 것이다.
3. **테스트 하네스를 저장소에 추가:** `tests/` 아래에 가짜 `window.claude` + Playwright 시나리오를 둔다 (8절).
4. **`runs` 의 running 고착 정리:** 시작 시 오래된 running 을 aborted 로 바꾼다.
5. **답안지 개선:**
   - 업로드 이미지 답안지 보존 (`assets` capability 검토)
   - 같은 파일 중복 업로드 방지
   - 반·주 연결 대상 반을 따로 선택
6. **여러 장 사진 제출 묶음 채점**, HEIC·docx 학생 제출물 지원
7. 다른 플랫폼으로 옮기기로 결정하면 2절의 선택지 2 를 따른다. 우선 DB 를 JSON 으로 내보내는 것부터 한다.

---

## 14. 관련 자료 (저장소 밖)

- claude.ai 계정의 다른 Artifact:
  - "IB Economics YR 1 채점 기준서 v1"
  - "김진환 선생님 수업 스타일 프로필 — IB Economics YR 1 (파일럿 초안)"
  - "숙제 리뷰 데스크 디자인" (초기 디자인 시안)
- claude.ai 계정 스킬 `ib-econ-homework-marking`: IB Economics YR 1 숙제 채점 방식 정리
- 이 저장소 코드는 위 자료를 직접 읽지 않는다. 채점 기준은 DB `config/main.criteria` 가 기준이다.
