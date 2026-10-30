# 코드 리뷰 (ART, 간단 버전)

> `Review_양식.md`를 따라 썼고, 2번(프롬프트)·5번(회고)은 선택이라 생략했어요.
> 대상: 이 저장소의 `Todo-로컬버전/` (커밋 `c822ee0`)과, 같은 코드의 전체 앱 [`Gkerala/todo-app`](https://github.com/Gkerala/todo-app) (커밋 `ae00ee5`).
> 맞다/틀리다를 가리려는 게 아니라 같이 보면 좋을 점을 나누려는 리뷰예요. 줄 번호는 위 커밋 기준이에요.

- 개발자 : Gkerala
- 리뷰어 : jhwksh

## 1. 원하는 기능이 의도한 대로 동작하나요?  [O]
- PRD의 기능(추가·목록·완료·삭제, 마감일, N:M 태그, 오늘 할 일만, 검색, 껐다 켜도 유지)이 모두 코드에 있어요.
  - 추가·태그: `server.js:48-55`, `store-sqlite.js:4-12` (트랜잭션으로 묶음)
  - 오늘·검색·태그 거르기: `store-sqlite.js:15-31`
  - 완료·삭제: `server.js:58-64`
  - 삭제 시 연결 줄 정리: `db.js:5`(`foreign_keys = ON`) + `db.js:20-21`(`ON DELETE CASCADE`)
- 근거: 코드를 읽어 확인했고, 실행 화면은 `Gkerala/todo-app`의 시연 영상(`demo/demo.mp4`, 0:00~0:47 로컬 모드)으로 확인했어요. 리뷰어 컴퓨터에서 찍은 실행 캡처는 아직 붙이지 못했어요.
- 같이 보면 좋을 점 (기능):
  - **클라우드 모드의 "오늘 할 일만"이 서버 시간 기준이에요** (`todo-app/store-supabase.js:36`). Render 서버는 보통 UTC라서 한국 시간 00:00~08:59에는 "어제" 마감 할 일이 걸러질 수 있어요(Render 시간대는 추측). 화면 배지(`index.html:248`)는 브라우저 날짜를 써서 "오늘"로 보여 주니 서로 어긋나요. → 화면이 `today=YYYY-MM-DD`로 날짜를 보내고, 서버는 그 값으로 거르면 두 모드가 같아져요.
  - 마감일은 모양만 검사해서 `2026-13-45`도 들어가요 (`server.js:52`). → `new Date(d + 'T00:00:00Z').toISOString().slice(0,10) === d`로 실제 날짜인지 확인하면 돼요.
  - 태그 이름에 쉼표가 있으면(API로만 가능) 목록에서 두 개로 쪼개져 보여요 (`store-sqlite.js:26, 31`의 `group_concat`/`split(',')`).
  - 검색어의 `%`, `_`가 와일드카드로 동작해요. `%`만 쳐도 전부 나와요 (`store-sqlite.js:19`). SQL 주입은 아니에요.
  - 상위 폴더에서 `node Todo-로컬버전/server.js sqlite`로 실행하면 화면이 404가 되고 `todo.db`가 상위 폴더에 새로 생겨요 (`server.js:11`, `db.js:4`가 현재 폴더 기준). → `path.join(__dirname, ...)`. (`npm start`로 실행하면 문제없어요.)

## 3. 보안 검토를 수행했나요?  [O]
- README 5절에 보안 확인 표가 있고, 대부분 코드와 맞았어요.
  - 하드코딩된 비밀값 없음. `.gitignore`가 `.env`와 `todo.db`를 막아요. `todo-app`의 git 기록 전체에서 키 형태(`sb_secret`, `sb_publishable_`, `eyJ…`)를 검색했는데 0건이었어요.
  - SQL은 모두 `?` 자리표시자를 써요. README 표에 없는 `store-sqlite.js:21-22`(태그 거르기)도 `?`를 써요.
  - 화면은 `textContent`만 써요 (`index.html:278, 287`).
  - 내부 오류는 숨겨요 (`server.js:67-70`).
  - 리뷰어도 `npm audit --package-lock-only`를 돌려 봤는데 취약점 0개였어요(2026-10-30).
  - 클라우드 쪽 잘한 점: service_role 키 없이 **사용자 토큰 + publishable 키**로만 DB에 접근해요. 세 표 모두 RLS를 켰어요. 연결표는 "할 일과 태그가 둘 다 내 것"일 때만 허용해요 (`supabase/schema.sql:43-52`).
- 같이 보면 좋을 점 (보안):
  1. **로컬 서버가 같은 와이파이 전체에 열려 있어요** (`server.js:73`). `app.listen(port)`에 주소를 안 줘서 모든 네트워크에서 접속을 받는데, 로컬 모드는 로그인이 없어요. 같은 네트워크의 다른 사람이 할 일을 보고 지울 수 있어요(Windows 방화벽에서 "허용"을 누른 경우). → 로컬 모드만 `app.listen(port, '127.0.0.1')`로 바꾸세요. Render는 지금처럼 두면 돼요.
  2. **입력 검사가 README 설명보다 약해요.**
     - 본문이 없거나 `tags`가 배열이 아니면(`"tags":"a"`) 400이 아니라 500 "서버 오류"가 나요 (`server.js:49, 53, 59`). Express 5에서는 본문이 없으면 `req.body`가 `undefined`라서 그래요.
     - `done:"false"`(문자열)를 보내면 완료로 바뀌어요 (`store-sqlite.js:39`).
     - → `const body = req.body || {}`로 시작하고, `Array.isArray(tags)`, `typeof done === 'boolean'`을 확인하세요.
  3. 제목·태그 길이에 제한이 없어요 (`server.js:49, 53`). 클라우드는 **공개 회원가입 + 요청 횟수 제한 없음**이라 무료 DB가 가득 찰 수 있어요. → 길이 상한을 두고, `/api/login`·`/api/signup`에 `express-rate-limit`을 거세요.
  4. 가입 실패 시 Supabase 원문을 그대로 보여 줘요 (`server.js:29`). "User already registered"처럼 이미 가입된 이메일인지 알 수 있어요. 로그인(`:19`)처럼 일반 메시지로 바꾸면 좋아요.
  5. `todo-app/.env.example:3`의 `SUPABASE_DB_PASSWORD`는 앱이 쓰지 않아요. 지우면 강한 비밀번호를 파일에 적을 일이 줄어요.

## 4. 에러를 디버깅하고 "문제를 해결한 기록"을 남겼나요?  [O]
- `Todo-로컬버전/README.md` 6절에 3건이 **증상 → 원인 → 해결과 확인** 순서로 적혀 있어요.
  - `[hidden]`이 `display:flex`에 덮인 문제
  - curl 한글 인코딩 문제
  - 옛 서버가 포트를 쥐고 있던 문제
- 추가 실험(SQL 주입 시도, 중복 태그, CASCADE 삭제, 재시작 유지)도 있어요. "코드 문제가 아니라 시험 환경 문제였다"고 구분한 점이 특히 좋았어요.
- 클라우드 쪽 기록(`todo-app/README.md` 93-95줄, Render `index.js` 실패·한글 경로 거부)과 RLS 확인 스크립트(`scripts/rls-check.js`, 12개 항목)도 재현할 수 있게 적혀 있어요.
- 보충 제안: `rls-check.js`에서 할 일 만들기가 실패하면 `id`가 `undefined`가 돼요. 그러면 "B에게 안 보인다" 같은 항목이 저절로 통과해요. `id`가 없으면 바로 멈추고, 정리(`:70`)를 `finally`로 옮기면 더 단단해져요. B가 A의 할 일에 `todo_tags`를 연결하는 시도도 시험에 추가하면 좋겠어요.

## 참고 링크 및 코드 개선
1. 리뷰하며 참고한 링크와 간단한 설명:
   - Express 5 이전 안내 (`req.body`가 기본으로 `undefined`): https://expressjs.com/en/guide/migrating-5.html
   - Node `server.listen`의 host (생략하면 모든 주소): https://nodejs.org/api/net.html#serverlistenport-host-backlog-callback
   - Supabase RLS 성능 권장(`(select auth.uid())`, 정책 열 인덱스): https://supabase.com/docs/guides/database/postgres/row-level-security
2. 개선을 제안하고 싶은 부분과 간단한 설명:
   - (문서) `Todo-로컬버전/README.md:16`의 `PORT=3001`은 bash 문법이에요. Windows PowerShell은 `$env:PORT=3001; npm start`예요.
   - (문서) `server.js:3` 주석의 `npm run start:cloud`는 로컬 폴더 `package.json`에 없어요. README 3절에 "전체 앱 기준"이라고 한 줄 있으면 덜 헷갈려요.
   - (문서) `todo-app` README 실행 방법에 Supabase 대시보드에서 **Confirm email 끄기** 단계가 있으면 좋아요. 안 끄면 가입 뒤 로그인이 안 돼요.
   - (설정) `package.json`에 `"engines": { "node": ">=22" }`를 추가하면 좋아요(better-sqlite3 13 요구사항). `todo-app/package.json`의 `"main": "index.js"`는 없는 파일이에요.
   - (DB) `create index on public.todos(user_id); create index on public.todo_tags(tag_id);`
   - (품질) `store-sqlite.js:5-9`의 `db.prepare`를 반복문 밖에서 한 번만 만들면 더 짧고 빨라요.
   - (화면) 완료·삭제 응답의 `res.ok`를 확인해서 실패하면 알려 주세요 (`index.html:274, 297`).

## 총평
작은 앱인데도 **SQL 자리표시자, textContent, RLS 3개 정책, service_role 미사용**처럼 보안의 기본이 꼼꼼하게 들어가 있고, README가 실행 방법·보안 포인트·문제 해결 기록까지 리뷰어가 따라가기 쉽게 정리되어 있어서 많이 배웠어요. 제안 중에서 하나만 고른다면 **로컬 서버를 `127.0.0.1`로만 여는 것**과 **클라우드의 "오늘" 날짜 기준 맞추기**예요. 나머지는 입력 검사를 조금 더 단단하게 하는 정도의 작은 제안이에요.
