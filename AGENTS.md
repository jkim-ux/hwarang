# AGENTS.md — 숙제 리뷰 데스크 작업 규칙

이 저장소에서 작업하는 코딩 에이전트(Codex, Claude Code 등)가 따를 규칙이다. 프로젝트 설명은 `HANDOFF.md`, 기능 상세는 `README.md` 를 먼저 읽는다.

> 출처: 기존 `CLAUDE.md` 는 없다 (저장소와 사용자 설정 모두 확인함). 기존 `AGENTS.md` 도 없었으므로 새로 만들었다. 아래 규칙은 README, 커밋 기록, 개발 중 교사(사용자)가 준 지시를 모은 것이다.

---

## 1. 프로젝트 성격 (먼저 알아야 할 것)

- `review-desk/index.html` 한 파일이 claude.ai **Artifact** 로 게시된 운영 페이지다.
- 이 페이지는 `window.claude.use('db' | 'mcp' | 'sample')` 에 의존한다. claude.ai 밖에서는 데이터와 채점이 동작하지 않는다.
- 운영 데이터는 claude.ai Artifact DB 에 있다. 저장소에는 데이터가 없고, 있어서도 안 된다.
- 게시와 DB 조작은 Claude Code(Artifact / ArtifactData 도구)로만 할 수 있다. Codex 는 소스 수정과 로컬 테스트까지만 한다.

## 2. 수정 금지 영역

교사가 명시적으로 요청하기 전에는 아래를 건드리지 않는다.

1. **운영 데이터:** DB 의 `reviews`, `config`, `roster`, `assignments`, `feedback`, `answerKeys`, `keyBindings`, `runs`.
   - 삭제, 일괄 수정, 스키마 마이그레이션 금지.
   - 꼭 써야 하면 읽은 버전으로 고정(`if_version`)한다.
   - `config/main` 은 교사가 페이지 설정 패널에서 계속 고친다. 덮어쓰지 말고 필요한 필드만 update 한다.
2. **게시 설정:**
   - Artifact `capabilities` 선언 (`db`, `sample`, `mcp` Google Drive 4개 도구) 변경 금지
   - 런타임 계약 버전(0.2.45) 변경 금지. 올리려면 먼저 별도 테스트 사본으로 검증한다.
   - 운영 URL 에 테스트 게시 금지
3. **Google Drive:** 페이지와 에이전트 모두 **읽기만** 한다. 학생 파일과 폴더의 수정·이동·공유·삭제 금지. 쓰기 도구를 capabilities 에 추가하지 않는다.
4. **채점 기준 텍스트:** `DEFAULT_RUBRIC`, `DEFAULT_CRITERIA`, DB `config/main.criteria` 는 교사가 정한 것이다. 요청이 있을 때만 바꾸고, 바꾼 내용을 보고한다.
5. **비활성 기능 되살리기 금지:**
   - 정기 자동 채점 루틴 (`routine-prompt.md` 는 참고용)
   - 미제출자 보기
   - 9·10학년 과목
6. **개인정보:** 학생 이름, 점수, 답안, Drive 폴더 구조 덤프, DB 내보내기 파일을 저장소에 커밋하지 않는다. 키, 토큰, 비밀번호는 어디에도 적지 않는다.

## 3. 교사가 정한 채점·제품 규칙 (기존 기능 보존 원칙)

새 기능을 만들 때도 아래 동작이 그대로 유지돼야 한다.

**채점 범위**
- 채점 범위는 IB 과목 숙제만이다. 9·10학년 Social Science 는 제외한다.
- IA, Task 3 는 절대 채점하지 않는다. 제목 정규식 기본값은 `\bIA\b|task\s*3|internal assessment|supporting document|feedback` 이다.

**확인 실행**
- 확인은 교사가 버튼을 눌렀을 때만 한다. 토큰 절약 때문이다. 자동 예약 실행을 넣지 않는다.
- 채점 모델은 최상위 티어(`modelTier: 'complex'`)다. 페이지는 모델 이름을 고를 수 없고, 실제 응답 티어를 리뷰에 기록한다.

**점수**
- 점수는 문제별 만점 기준이다. 만점은 학생 문서나 과제 파일의 배점에서 읽고, 없으면 문제 유형으로 추정한다 (`maxScore`, `scoreBasis`).

**Economics 기준**
- 10점 문항: real-life example 을 요구하지 않는다. 결론 문단도, intro 문단도 요구하지 않는다.
- 15점 문항: real-life example 이 필요하다.
- price mechanism 요소(shift 원인, shortage, rationing, signalling, incentive, market-clearing)와 감점 상한은 `criteria` 에 있는 그대로 따른다.

**학생 코멘트**
- 반말로, 짧게 쓴다.
- 과목 용어는 영어로 쓴다.
- 따옴표로 학생 문장을 인용하지 않는다.

**프롬프트 우선순위** (위가 아래보다 우선)
1. 시스템 피드백(`feedback`, active)
2. 이번 확인 지시사항
3. 답안지
4. 과목별 기준
5. 공통 rubric

**그래프·이미지**
- graph 는 이미지로 확인한다. 이미지를 못 보낸 경우 graph 가 없다고 단정하지 않는다. "텍스트로는 확인 불가"로만 적는다.
- PDF·사진 제출물은 페이지 이미지 없이 텍스트만으로 채점하지 않는다. 이미지를 못 보내면 `status: needs_vision` 으로 남긴다. 인쇄된 문제지 글자만 보고 0점을 준 사고가 있었다.

**답안지**
- 답안지를 쓴 채점은 어떤 답안지를 썼는지 남겨야 한다: `answerKeyUsed`, `answerKeyMatch`, `keyChecklist`, `answerKeyStatus`.
- 같은 반·같은 주 답안지 재사용(`keyBindings`)을 깨지 않는다.

**데이터 호환성**
- 기존 리뷰 문서는 필드가 빠져 있을 수 있다. 렌더 코드는 필드가 없어도 동작해야 한다.
- 새 필드는 추가만 한다. 기존 필드의 의미나 형식을 바꾸지 않는다.

## 4. 코딩 규칙

**구조**
- 단일 파일 구조를 유지한다. CSS, 마크업, 스크립트가 `review-desk/index.html` 하나에 있다.
- 빌드 도구, 프레임워크, npm 의존성을 페이지에 넣지 않는다.

**스크립트 스타일**
- 기존 스타일을 따른다. 하나의 IIFE 안에서 `var`, `function` 선언, 콜백과 Promise 체인을 쓴다.
- HTML 은 문자열을 이어 붙여 만들고, 모든 사용자·DB 값은 `esc()` 로 이스케이프한다.
- 이벤트는 컨테이너에 위임하고 `data-*` 속성으로 구분한다.
- 상태는 `state` 객체 하나에 두고 `render()` / `renderRunDlg()` 로 다시 그린다.
- DB 스냅샷 데이터는 읽기 전용이다. `clone()` 한 뒤 쓴다. 예전에 스냅샷을 직접 수정해 클릭이 고장 난 적이 있다.
- 주석은 지금처럼 적게, 구획 제목 위주로 단다. 이름은 짧은 camelCase 를 쓴다.

**외부 스크립트·스타일**
- 외부 스크립트는 cdnjs 등 허용된 CDN 에서만, 정확한 버전으로 불러온다 (현재 pdf.js 3.11.174).
- 색은 `:root` 토큰과 다크 모드 규칙을 쓴다. 하드코딩 색을 추가하지 않는다.

**문구**
- UI 문구는 한국어(교사용), 짧은 반말·평서체다.
- 페이지 헤더의 버전 표시(`' · v21'`)는 의미 있는 변경 때 올린다.

**변경 범위**
- 요청받은 것만 최소로 바꾼다. 요청 없는 리팩터링, 포맷 정리, 파일 분리는 하지 않는다.

## 5. 테스트 절차

변경 후 아래를 순서대로 한다. 실패하면 커밋하지 않는다.

1. **문법 확인:**

   ```bash
   node -e "const s=require('fs').readFileSync('review-desk/index.html','utf8');(s.match(/<script>([\s\S]*?)<\/script>/g)||[]).forEach(x=>new Function(x.replace(/<\/?script>/g,'')));console.log('ok')"
   ```

2. **브라우저 시나리오 테스트 (Playwright + Chromium):**
   - 페이지를 `<!doctype html><html><head><meta charset="utf-8"></head><body>` + index.html + `</body></html>` 로 감싸 로컬에서 연다.
   - `page.addInitScript` 로 가짜 `window.claude = { use: async name => ... }` 를 넣는다.
     - `db`: 메모리 저장소. `doc().get/set/update/delete/onSnapshot`, `collection().orderBy/limit/where/get/onSnapshot` 를 구현하고, 쓰기마다 해당 컬렉션 리스너에 스냅샷을 다시 보낸다.
     - `mcp`: `server()` 가 가짜 Drive 를 돌려준다. `search_files`(쿼리의 `parentId = '...'` 로 트리 조회), `read_file_content`, `download_file_content`(base64), `get_file_metadata`.
     - `sample`: 함수이면서 `.json()` 과 `.limits()` 를 가진다. 호출 인자(프롬프트, 이미지 수, 티어)를 기록하고 고정 JSON 을 돌려준다. `limits()` 에 `images` 를 넣거나 빼서 이미지 가능·불가 두 경우를 모두 확인한다.
   - `page.route('https://cdnjs.cloudflare.com/**')` 로 pdf.js 를 로컬 `pdfjs-dist@3.11.174` 의 `build/pdf.min.js`, `build/pdf.worker.min.js` 로 바꾼다. Google Fonts 요청은 abort 한다.
   - `pageerror` 이벤트가 없어야 한다.
   - 바꾼 기능의 시나리오와 함께 아래 회귀 항목을 확인한다.
     - 확인 창 → 문서 찾기 → 채점 → `reviews` 저장 → 목록·상세 표시
     - 이미지 불가 환경에서 PDF 는 `needs_vision`, 문서는 graph 확인 불가 문구
     - 답안지 선택·업로드 → 프롬프트 포함 → `answerKeyStatus` 표시
     - `keyBindings` 저장, 자동 첨부, 다시 채점 시 중복 없음
     - 시스템 피드백이 프롬프트에 들어감
     - 비교 화면이 열리고 코멘트 위치가 표시됨
3. **게시 전 diff 재검토:** 2절 금지 영역을 건드리지 않았는지, 개인정보가 들어가지 않았는지 본다.
4. **게시는 사람이 결정한다.** 게시가 필요하면 교사에게 알리고, Claude Code 에서 같은 URL 로 capabilities 를 생략한 채 publish 한다. 게시 후 claude.ai 에서 한 번 열어 본다.

## 6. Git

- 기본(유일한) 브랜치는 `claude/cool-sagan-ogc2zd`, 원격은 `https://github.com/jkim-ux/hwarang` 이다.
- 커밋 메시지는 영어, 명령형 한 줄 요약에 필요하면 본문을 단다 (기존 기록과 같은 형식).
- 히스토리 재작성(force push, rebase, amend) 금지.

## 7. 확인이 필요하면

- 채점 기준, 말투, 대상 과목, 비용(토큰)에 영향을 주는 변경은 먼저 교사에게 묻는다.
- 운영 DB 나 게시 페이지에 손대야 하는 작업은 직접 하지 말고, 해야 할 일을 정리해 보고한다.
