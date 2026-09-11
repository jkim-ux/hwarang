# 숙제 자동 리뷰 루틴 프롬프트

아래 내용이 매일 08:00 / 17:00 (KST) 에 Claude 세션으로 전달됩니다.
루틴은 claude.ai/code 의 Routines 목록에서 수정·중지할 수 있습니다.

> 현재 루틴은 이 페이지를 만든 세션(구글 드라이브 커넥터가 연결된 세션)으로 되돌아와 실행되도록 묶여 있습니다.
> 조직 설정상 API로 만든 루틴에는 커넥터를 붙일 수 없어서, 새 세션으로 띄우는 방식을 쓰려면
> claude.ai 의 Routines 화면에서 Google Drive 커넥터를 켜고 이 프롬프트를 붙여 넣어 직접 만들어야 합니다.

---

[정기 실행] 숙제 자동 리뷰 시각이다. 이번 실행의 출처(source)는 "scheduled" 이다. 사람이 지켜보고 있지 않으므로 질문하지 말고 아래 절차를 끝까지 실행한 뒤 한 줄로만 보고하라 (검사 n건, 리뷰 n건, 오류 n건). 이전 대화의 세부 내용이 요약돼 있어도 아래 지시만으로 충분히 실행할 수 있다.

## 데이터 위치
- 아티팩트: https://claude.ai/code/artifact/b3743ca6-bd7f-4fd2-8b24-fcf536678b5b (숙제 리뷰 데스크)
- 모든 설정·리뷰·명단·과제·실행 기록은 이 아티팩트의 DB에 있다. `Artifact` 도구의 `read_db` / `write_db` 액션에 위 URL을 `url` 로 넘겨 읽고 쓴다.
- 페이지 HTML은 절대 다시 게시(publish)하지 마라. DB만 쓴다.
- 학생 문서는 읽기만 한다. 수정·이동·공유·삭제 금지.
- Google Drive 도구(mcp__Google_Drive__search_files, mcp__Google_Drive__read_file_content)는 ToolSearch 로 "select:" 하여 불러온다. 없으면 runs 에 errors [{"title":"실행","message":"Google Drive 커넥터 없음"}] 로 기록하고 종료.

## 절차
1. `read_db` get, collection `config`, doc_id `main`. 필드: `folders[{id,name,label}]` (과목 폴더 목록), `exclude[]` (학생 폴더로 보지 않을 하위 폴더 이름 조각), `skipTitles` (채점하지 않을 문서 제목 정규식, 대소문자 무시. 기본 `\bIA\b|task\s*3|internal assessment|supporting document`. IA 와 Task 3 는 교사가 채점하지 않기로 했으므로 이 패턴에 걸리는 문서는 후보에서 빼고 리뷰를 만들지 않는다), `materialsPattern` (자료실 폴더 이름 조각, 기본 "자료실"), `homeworkPattern` (숙제 폴더 이름 조각, 기본 "Homework"), `sinceDays`, `minChars`, `rubric`. 문서가 없거나 `folders` 가 비어 있으면 아무것도 쓰지 말고 종료.
2. `read_db` list, collection `reviews` (`query.limit` 1000, `next_cursor` 가 있으면 계속). `fileId → {modifiedTime, revision, assignmentId}` 맵을 만든다.
3. 각 folder 에 대해 Google Drive `search_files` 를 `query: "parentId = '<folder.id>'"`, `excludeContentSnippets: true`, `pageSize: 100` 으로 호출하고 `nextPageToken` 이 있으면 `pageToken` 으로 이어서 호출한다. 항목을 셋으로 나눈다:
   - **자료실**: 제목에 `materialsPattern` 이 포함된 폴더 (첫 번째 것). 그 안에서 제목에 `homeworkPattern` 이 포함된 폴더를 찾고, 그 하위 폴더 각각이 **과제** 다. 과제 = `{id: 폴더 id, name: 폴더 제목, due: 제목의 "M.DD" 를 YYYY-MM-DD 로 (올해 기준), url: 폴더 viewUrl, docs: [{id,title}] (안의 구글 문서), prompt: 마감이 최근 60일 이내인 과제는 첫 문서 본문 앞 2000자, 아니면 빈 문자열}`.
   - **학생 폴더**: 그 외 폴더 중 제목에 `exclude` 의 문자열이 포함되지 않은 것. 학생 이름 = 폴더 제목에서 `folder.name` 을 지우고 공백을 정리한 것 (예: `박성하 IB Economics YR 1` → `박성하`). 각 학생 폴더 안의 구글 문서(`mimeType` = `application/vnd.google-apps.document`)를 모은다 (query 에 `and modifiedTime > '<지금 - sinceDays 일, RFC3339>'` 를 붙여도 된다).
   - **폴더 바로 아래 문서**: 학생 이름 = `owner` 이메일의 `@` 앞부분 (없으면 빈 문자열).
   - 두 단계보다 깊이 들어가지 않는다.
   - 과목마다 `write_db` set: collection `roster`, doc_id = folder.id, data `{folderId, folderName, students: [{name, folderId, folderName: 폴더 제목, url}], updatedAt}` 과 collection `assignments`, doc_id = folder.id, data `{folderId, folderName, items: [과제...], updatedAt}`.
4. 후보 = `modifiedTime` 이 지금부터 `sinceDays` 일 이내이고 제목이 `skipTitles` 에 걸리지 않는 문서. 리뷰 대상 = `reviews` 에 없거나, 저장된 `modifiedTime` 이 문서의 `modifiedTime` 보다 이전인 것.
5. 각 대상 문서에 대해 `read_file_content` 로 본문을 읽는다.
   - 본문(앞뒤 공백 제거 후) 길이가 `minChars` 미만이면 `status: "empty"` 리뷰를 만든다.
   - 아니면 `rubric` 의 지시에 따라 리뷰를 직접 작성한다. 과목은 `folder.name` 으로 판단한다. 본문이 60,000자를 넘으면 앞 60,000자만 읽는다.
   - **과제 매칭**: 그 과목의 과제 목록과 비교해 이 문서가 어느 과제의 답인지 정한다. 문서 제목·내용을 우선하고, 제출 시각과 마감일은 보조 근거로 쓴다 (학생이 결석 등으로 한 주 늦게 낼 수 있다). 확신이 없으면 마감일이 `modifiedTime` 이후 10일 이내에서 가장 가까운 과제로 정한다. 그래도 없으면 빈 문자열.
6. 리뷰를 `write_db` batch (최대 50개씩), op `set`, collection `reviews`, doc_id = 문서의 fileId 로 저장한다.
7. 기존 리뷰 중 `assignmentId` 가 비어 있는 것은 5의 날짜 규칙으로 채워 `write_db` update 한다 (`assignmentId`, `assignmentName`, `assignmentDue`).
8. 마지막에 실행 기록을 `write_db` set, collection `runs`, doc_id `run_<시작시각 ISO에서 : 과 . 을 - 로 바꾼 것>` 으로 저장한다. 리뷰 대상이 0건이어도 실행 기록은 남긴다.

## 리뷰 문서 스키마 (`reviews/<fileId>`)
{"fileId": "<Drive file id>", "title": "<문서 제목>", "student": "<학생 이름>", "folderId": "<config.folders[].id>", "folderName": "<config.folders[].name>", "subfolder": "<학생 폴더 제목 또는 빈 문자열>", "subfolderId": "<학생 폴더 id 또는 빈 문자열>", "subfolderUrl": "<학생 폴더 viewUrl 또는 빈 문자열>", "assignmentId": "<과제 폴더 id 또는 빈 문자열>", "assignmentName": "<과제 이름>", "assignmentDue": "<YYYY-MM-DD>", "viewUrl": "<문서 viewUrl>", "modifiedTime": "<문서 modifiedTime 그대로>", "reviewedAt": "<지금 ISO 시각>", "source": "scheduled", "revision": 1, "seen": false, "status": "reviewed", "score": 7, "completeness": "complete", "summary": "교사가 한눈에 볼 한 문장 (60자 이내)", "strengths": ["구체적인 잘한 점"], "improvements": ["구체적인 고칠 점과 고치는 방법"], "nextSteps": ["학생에게 줄 다음 과제·행동 1~3개"], "flags": [], "comment": "학생에게 그대로 전달해도 되는 3~5문장 코멘트 (한국어)", "wordCount": 123, "docText": "<본문 앞 30000자 그대로>", "annotations": [{"quote": "<본문에서 그대로 복사한 구절, 10~200자, 줄바꿈 없이>", "note": "<그 부분에 달 코멘트, comment와 같은 말투, 1~2문장>"}]}
- `revision`: 기존 리뷰가 있으면 기존 `revision` + 1, 없으면 1.
- `score`: 0~10 정수. `status` 가 "empty" 이면 null.
- `completeness`: "complete" | "partial" | "empty".
- `flags`: "미완성", "과제와 무관", "AI 작성 의심", "표절 의심" 중 해당하는 것만. 없으면 [].
- `annotations`: 고칠 부분 위주로 최대 8개. quote는 반드시 본문에 있는 문자열 그대로 (페이지가 이 문자열로 본문을 찾아 하이라이트한다). `docText`: read_file_content 로 읽은 본문 앞 30000자.
- empty 리뷰: status "empty", score null, completeness "empty", summary "문서에 내용이 거의 없습니다 (N자).", strengths [], improvements ["문서를 열어 과제를 작성해야 합니다."], nextSteps [], flags ["미완성"], comment "".

## 실행 기록 스키마 (`runs/<id>`)
{"id": "run_2026-09-10T23-00-00-000Z", "source": "scheduled", "startedAt": "<ISO>", "finishedAt": "<ISO>", "status": "done", "checked": 12, "reviewed": 3, "skipped": 9, "errors": [{"title": "학생 · 문서", "message": "무엇이 실패했는지"}]}
