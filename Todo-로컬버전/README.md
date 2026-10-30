# Todo 앱 (로컬 버전) - 코드 리뷰용

퀘스트 C-2 코드 페어리뷰를 위해 **로컬(SQLite)로 실행되는 부분만** 따로 모아 둔 폴더예요. 이 폴더만 받아서 실행하면 돼요.
계정, 키, `.env`, 인터넷 서비스가 **필요 없어요.**

> 이 폴더의 코드 파일(`server.js`, `db.js`, `store-sqlite.js`, `public/index.html`, `PRD.md`)은 전체 앱 저장소([`Gkerala/todo-app`](https://github.com/Gkerala/todo-app))에서 **그대로 복사**한 것이에요. 바뀐 건 `package.json`(의존성에서 클라우드용 `@supabase/supabase-js` 제거)과 `.gitignore`, 이 README뿐이에요.

## 1. 실행 방법
Node.js 24 계열(LTS)이 필요해요. 이 폴더에서 터미널을 열고 아래 두 줄을 실행하세요.
```bash
npm install
npm start
```
브라우저에서 **http://localhost:3000** 을 열면 돼요. 종료는 터미널에서 `Ctrl+C`.
- 데이터는 이 폴더 안 **`todo.db`** 파일에 저장돼요. 처음 실행할 때 자동으로 만들어져요(GitHub에는 없는 파일).
- 포트 3000이 이미 쓰이고 있으면(`EADDRINUSE`) 다른 서버를 끄거나 `PORT=3001`처럼 바꿔 실행하세요.
- `npm install`에서 `better-sqlite3` 설치 오류가 나면 에러 문구를 그대로 에이전트에게 붙여 넣고 OS를 알려 주세요(Windows는 빌드 도구가 필요할 수 있어요).

## 2. 이 폴더의 파일
| 파일 | 역할 |
|---|---|
| `server.js` | 서버(Express). API 4개(목록·추가·완료·삭제)와 입력 검증 |
| `db.js` | SQLite 파일(`todo.db`)을 열고 표 3개(`todos`, `tags`, `todo_tags`)를 만듦 |
| `store-sqlite.js` | 실제 DB 읽기·쓰기. 모든 SQL이 `?` 자리표시자를 씀 |
| `public/index.html` | 화면(HTML+CSS+JS 한 파일). 서버 API에만 요청함 |
| `PRD.md` | 무엇을 만들려고 했는지 (이 앱의 요구 기능) |
| `package.json`, `package-lock.json` | 의존성: `express`, `better-sqlite3` |

## 3. 이 폴더에 **없는 것** (클라우드 전용, 이번 리뷰 대상 아님)
- Supabase 저장소 코드(`store-supabase.js`), DB 설계·RLS 정책(`supabase/schema.sql`), RLS 확인 스크립트(`scripts/rls-check.js`)
- Render 배포 설정, 시연 영상(`demo/`), `.env.example`
- `server.js`와 `index.html` 안에는 **로그인·회원가입 코드가 남아 있지만 로컬 모드에서는 실행되지 않아요.** (`/api/config`가 `login:false`를 돌려주고, 서버는 `sqlite` 모드에서 로그인 경로를 만들지 않음)
- 전체 앱(클라우드 포함)은 [`Gkerala/todo-app`](https://github.com/Gkerala/todo-app) 저장소에 있어요.

## 4. 확인해 볼 기능 (ART 1번: 실행 화면을 근거로)
1. 할 일 **추가**: 제목 + 마감일 + 태그(쉼표로 구분)
2. 목록에 마감일(오늘은 초록, 기한 지남은 주황)과 태그가 보이는지
3. **완료** 체크(취소선), **삭제**
4. **오늘 할 일만**, **검색**, **태그 클릭으로 거르기**와 해제
5. **새로고침해도** 남는지
6. 터미널에서 `Ctrl+C`로 서버를 끈 뒤 **다시 `npm start`** 했을 때 남는지

## 5. 보안 확인 포인트 (ART 3번)
| 볼 것 | 어디서 |
|---|---|
| 키·비밀번호가 코드에 하드코딩됐나 | 이 폴더에는 `.env`·키 자체가 없음. `.gitignore`가 `.env`, `todo.db`를 막음 |
| 입력값 검증 | `server.js` 49~53줄: 제목 공백 제거·빈 값 거절, 마감일 `YYYY-MM-DD` 형식, 태그 정리·중복 제거 |
| SQL 주입 | `store-sqlite.js` 5, 7, 8, 9, 19, 39, 43줄: 값은 모두 `?`로 전달 (문자열을 이어 붙이지 않음) |
| XSS(화면 코드) | `public/index.html`: 사용자 입력을 `textContent`로만 표시 (`innerHTML` 사용 없음) |
| 오류 내용 노출 | `server.js` 67줄: 내부 오류는 로그에만 남기고 화면에는 `서버 오류`만 보냄 |
| 외부 의존성 | `express`, `better-sqlite3` 두 개. `npm audit` 결과 취약점 0 (2026-10-30) |
| 한계(알아 둘 점) | 로컬 모드에는 **로그인이 없음**. 내 컴퓨터에서만 쓰는 용도 |

## 6. 문제 해결 기록 (ART 4번)
에이전트(Claude Code)와 함께 이 앱을 만들고 시험하면서 **실제로 발견하고 고친 문제**예요. 추측이 아니라 겪은 순서대로 적었어요.

| # | 증상(어떻게 발견했나) | 원인 | 해결과 확인 |
|---|---|---|---|
| 1 | 시연 영상을 다시 보다가 **로그인한 뒤에도 로그인 칸이 화면에 계속 남아** 있었다 | CSS의 `form { display: flex }`가 HTML `hidden` 속성을 **덮어써서** 숨겨지지 않았다 | `public/index.html`에 `[hidden] { display: none !important; }` 한 줄 추가. 브라우저에서 `hidden`일 때 `display:none`, 아닐 때 `flex`인 것을 확인했다 |
| 2 | 한글로 검색·태그 필터를 시험했더니 **결과가 비어 있고** 목록의 한글이 깨져 보였다. 서버 오류처럼 보였다 | 서버가 아니라 **`curl`이 한글을 UTF-8로 보내지 않은 것**(터미널 인코딩) | Node의 `fetch`로 같은 시험을 다시 해서 통과했다. 서버 코드는 고치지 않았다 |
| 3 | API 시험이 `6 !== 3`(할 일이 3개여야 하는데 6개)으로 **실패**했다 | 이전 시험에서 띄운 서버가 같은 포트(3112)를 **계속 쥐고** 있어서, 새 서버가 아니라 옛 서버(옛 데이터)에 요청이 갔다 | 옛 시험 서버를 종료하고 **다른 포트(3114)**에서 다시 실행해 통과했다. 코드 문제가 아니라 시험 환경 문제였다 |

**추가로 해 본 시험(새로운 시도)**
- **SQL 주입 시도:** 검색어에 `' OR 1=1 --`를 넣었더니 **0건**이었다. 입력이 명령이 아니라 값으로만 처리된다(`?` 파라미터).
- **같은 태그를 두 번 입력:** 한 번만 붙는다(`INSERT OR IGNORE`).
- **할 일을 삭제:** 그 할 일에 붙은 태그 연결 줄도 같이 지워진다(`ON DELETE CASCADE`, `db.js`의 `foreign_keys` 켜기).
- **서버를 껐다 켠 뒤:** 데이터가 그대로 남는다.
- 이 시험들은 한글이 포함된 API 시험 한 파일로 묶어 한 번에 통과를 확인했다.

**클라우드 쪽에서 겪은 문제(이 폴더에는 없는 부분)** 는 [`Gkerala/todo-app`](https://github.com/Gkerala/todo-app)의 README에 있어요: Render의 Start Command를 비워 두면 `node index.js`로 실행되어 실패한 것, 폴더 이름에 한글이 들어가자 Render의 Root Directory가 거부한 것.

## 7. 리뷰 제출 위치 (개발자 메모)
`Review.md`는 이 저장소의 **맨 위**(`README.md`와 같은 위치)에 두고 PR을 보내는 것을 제안해요. **(보충: 퀘스트 안내에 위치는 없어요)**
