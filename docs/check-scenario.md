# B7-1 요구사항 점검 시나리오 (AI 에이전트용)

[B7-1 과제](B7-1.md)의 요구사항을 실제 서버로 확인하는 절차다. AI 에이전트(또는 사람)가 위에서부터 그대로 실행하고, 마지막의 보고 형식으로 결과를 정리한다. 자동 테스트 코드는 두지 않는다.

## 0. 규칙

- 사용자의 개발 서버(`:8000`, `:5173`)와 `data/app.db`는 건드리지 않는다. 점검용 서버는 임시 디렉터리 DB와 `:8010`~`:8013` 포트를 쓴다. 포트가 사용 중이면 다른 빈 포트를 쓴다.
- `.env`의 API 키 값은 출력하지 않는다. 설정 여부만 확인한다.
- 실제 OpenAI 호출이 8회(8단계 브라우저 확인 포함 시 9회) 발생한다(소량 비용). 그 외 실패 경로는 키·시간 제한을 바꿔 비용 없이 확인한다.
- 테스트 계정에는 임시 비밀번호만 쓴다. 비밀번호는 평문 저장된다.
- zsh에서 반복 변수 이름으로 `path`를 쓰지 않는다(`PATH`가 덮여 `curl`을 찾지 못한다).
- 끝나면 자신이 띄운 서버만 종료한다.

## 1. 준비

```bash
cd <저장소 루트>
for k in OPENAI_API_KEY AI_MODEL; do v=$(grep -E "^$k=" .env | cut -d= -f2-); [ -n "$v" ] && echo "$k: set" || echo "$k: MISSING"; done
pnpm --dir frontend install --frozen-lockfile
pnpm --dir frontend check && pnpm --dir frontend build
uv sync --frozen --extra dev && uv run --frozen ruff check app

R=$(mktemp -d)   # 점검용 DB·로그·쿠키 위치
: > "$R/pids"
start() { # start <port> <db name> [ENV=VALUE ...]
  local port=$1 db=$2; shift 2
  if lsof -tiTCP:"$port" -sTCP:LISTEN >/dev/null; then
    echo "port :$port is already in use; choose another port"; return 1
  fi
  env "$@" DATABASE_URL="sqlite+aiosqlite:///$R/$db.db" \
    uv run --frozen uvicorn app.main:create_app --factory --port "$port" > "$R/$db.log" 2>&1 &
  local server_pid=$!
  echo "$server_pid" >> "$R/pids"
  for i in $(seq 1 50); do
    kill -0 "$server_pid" 2>/dev/null || break
    curl -fsS -o /dev/null "http://127.0.0.1:$port/" 2>/dev/null && return
    sleep 0.2
  done
  echo "server :$port did not start"; return 1
}
start 8010 main                                   # 정상 서버
start 8011 timeout AI_TIMEOUT_SECONDS=0.01        # AI 시간 초과
start 8012 badkey OPENAI_API_KEY=sk-invalid-test  # AI 호출 실패
start 8013 dbfail                                 # DB 저장 실패

B=http://127.0.0.1:8010; J='Content-Type: application/json'
chk() { printf '%-40s %-8s %s\n' "$1" "$2" "$([ "$2" = "$3" ] && echo PASS || echo "FAIL (want $3)")"; }
code() { curl -s -o /dev/null -w '%{http_code}' "$@"; }
```

통과 기준: 키·모델이 `set`, 프런트 검사·빌드와 ruff 통과, 네 서버 모두 시작.

## 2. 인증·접근 제어 (4.2)

```bash
chk "chat without login"     "$(code $B/api/chat -H "$J" -d '{"question":"hi"}')" 401
chk "my chats without login" "$(code $B/api/me/chats)" 401
chk "signup A"               "$(code $B/api/auth/signup -H "$J" -d '{"username":"userA","password":"test-a"}')" 204
chk "duplicate signup"       "$(code $B/api/auth/signup -H "$J" -d '{"username":"userA","password":"x"}')" 409
chk "wrong password"         "$(code $B/api/auth/login -H "$J" -d '{"username":"userA","password":"bad"}')" 401
chk "login A"                "$(code -c $R/a.txt $B/api/auth/login -H "$J" -d '{"username":"userA","password":"test-a"}')" 204
```

## 3. 입력 검증 (4.5)

```bash
chk "blank question"  "$(code -b $R/a.txt $B/api/chat -H "$J" -d '{"question":"   "}')" 422
chk "2001-char question" "$(code -b $R/a.txt $B/api/chat -H "$J" -d "{\"question\":\"$(python3 -c 'print("x"*2001)')\"}")" 422
```

## 4. AI 응답과 문맥 유지 (4.3) — 실제 OpenAI 호출 2회

```bash
curl -s -b $R/a.txt $B/api/chat -H "$J" -d '{"question":"배포 방법을 한 문장으로 알려줘."}'; echo
curl -s -b $R/a.txt $B/api/chat -H "$J" -d '{"question":"내가 방금 뭘 물어봤지?"}'; echo
```

통과 기준: 둘 다 HTTP 200, `answer`가 비어 있지 않다. 두 번째 답변이 첫 질문(배포 방법)을 언급한다.

## 5. 대화 로그 저장·조회·사용자 분리 (4.4)

```bash
curl -s -b $R/a.txt $B/api/me/chats; echo     # 2건, 각 항목에 question·answer·created_at(Z)
curl -s -o /dev/null $B/api/auth/signup -H "$J" -d '{"username":"userB","password":"test-b"}'
curl -s -o /dev/null -c $R/b.txt $B/api/auth/login -H "$J" -d '{"username":"userB","password":"test-b"}'
chk "B cannot see A's log" "$(curl -s -b $R/b.txt $B/api/me/chats)" "[]"
curl -s -b $R/b.txt $B/api/chat -H "$J" -d '{"question":"내가 방금 뭘 물어봤지?"}'; echo   # A의 질문을 모른다
chk "logout A"             "$(code -b $R/a.txt -X POST $B/api/auth/logout)" 204
chk "old cookie after logout" "$(code -b $R/a.txt $B/api/me/chats)" 401
curl -s -o /dev/null -c $R/a.txt $B/api/auth/login -H "$J" -d '{"username":"userA","password":"test-a"}'
chk "history after re-login" "$(curl -s -b $R/a.txt $B/api/me/chats | python3 -c 'import json,sys; print(len(json.load(sys.stdin)))')" 2
sqlite3 -header -column $R/main.db '.parameter init' '.parameter set :user_id 1' '.read scripts/check_logs.sql'
```

통과 기준: 모든 `chk` PASS. B의 답변에 A의 질문 내용이 없다. SQL 결과에 userA의 2건(사용자·시각·질문·답변)이 나온다.

## 5-1. 다중 사용자 동시 대화 (4.3, 4.4) — 실제 OpenAI 호출 4회

두 사용자가 동시에 서로 다른 주제로 질문하고, 이어서 동시에 후속 질문한다.

```bash
for u in userC userD; do
  curl -s -o /dev/null $B/api/auth/signup -H "$J" -d "{\"username\":\"$u\",\"password\":\"test-$u\"}"
  curl -s -o /dev/null -c $R/$u.txt $B/api/auth/login -H "$J" -d "{\"username\":\"$u\",\"password\":\"test-$u\"}"
done
ask() { curl -s -b $R/$1.txt $B/api/chat -H "$J" -d "{\"question\":\"$2\"}" > $R/$1-$3.json; }
ask userC "파이썬 리스트를 한 문장으로 설명해 줘." 1 & ask userD "SQL JOIN을 한 문장으로 설명해 줘." 1 & wait
ask userC "내가 방금 뭘 물어봤지?" 2 & ask userD "내가 방금 뭘 물어봤지?" 2 & wait
for u in userC userD; do echo "$u: $(python3 -c "import json; print(json.load(open('$R/$u-2.json'))['answer'])")"; done
first() { curl -s -b $R/$1.txt $B/api/me/chats | python3 -c 'import json,sys; d=json.load(sys.stdin); print(len(d), d[0]["question"].split()[0])'; }
chk "userC log" "$(first userC)" "2 파이썬"
chk "userD log" "$(first userD)" "2 SQL"
```

통과 기준: 두 `chk` PASS(각자 자기 질문 2건만 저장). userC의 후속 답변은 파이썬 리스트를 언급하고 SQL은 언급하지 않는다. userD의 후속 답변은 SQL JOIN을 언급하고 파이썬은 언급하지 않는다.

## 6. 실패 처리 (4.5, 6. 운영 안정성)

```bash
for p in 8011 8012 8013; do
  curl -s -o /dev/null http://127.0.0.1:$p/api/auth/signup -H "$J" -d '{"username":"u","password":"p"}'
  curl -s -o /dev/null -c $R/c$p.txt http://127.0.0.1:$p/api/auth/login -H "$J" -d '{"username":"u","password":"p"}'
done
# DB 저장 실패: chats 삽입을 막는 트리거 (AI 호출은 성공 — 실제 OpenAI 호출 1회)
sqlite3 $R/dbfail.db "CREATE TRIGGER block_insert BEFORE INSERT ON chats BEGIN SELECT RAISE(ABORT, 'forced'); END;"

curl -s -b $R/c8011.txt -w ' HTTP %{http_code}\n' http://127.0.0.1:8011/api/chat -H "$J" -d '{"question":"긴 글 요약해줘"}'
curl -s -b $R/c8012.txt -w ' HTTP %{http_code}\n' http://127.0.0.1:8012/api/chat -H "$J" -d '{"question":"hi"}'
curl -s -b $R/c8013.txt -w ' HTTP %{http_code}\n' http://127.0.0.1:8013/api/chat -H "$J" -d '{"question":"한 단어로 인사해 줘."}'
for p in 8011 8012 8013; do echo ":$p alive $(code http://127.0.0.1:$p/) saved $(curl -s -b $R/c$p.txt http://127.0.0.1:$p/api/me/chats)"; done
```

| 서버 | 기대 응답 | 기대 로그 |
| --- | --- | --- |
| `:8011` 시간 초과 | `504 AI_TIMEOUT`, “현재 응답이 지연되고 있어요…” | `ai_provider_error` (`APITimeoutError`), `ai_call_failed` (`code: AI_TIMEOUT`, `latency_ms`) |
| `:8012` 잘못된 키 | `502 AI_UNAVAILABLE` | `ai_provider_error` (`AuthenticationError`, 401), `ai_call_failed` |
| `:8013` DB 실패 | `503 DB_UNAVAILABLE` | `ai_call_success` 다음 `db_save_failed` |

통과 기준: 위 응답과 일치, 세 서버 모두 `alive 200`, 저장된 대화 `[]`.

## 7. 운영 로그 (4.5, 6)

```bash
rid=$(grep '"event": "ai_call_success"' $R/main.log | head -1 | grep -o '"request_id": "[a-f0-9]*"')
grep "$rid" $R/main.log | grep -oE '"event": "[a-z_]+"'
grep -hoE '"event": "(ai_call_failed|ai_provider_error|db_save_failed)"[^}]*' $R/timeout.log $R/badkey.log $R/dbfail.log
key_tail=$(grep -E '^OPENAI_API_KEY=' .env | cut -d= -f2- | tail -c 12)
for s in "$key_tail" test-a test-b "배포 방법"; do grep -c -- "$s" $R/*.log | awk -F: '{n+=$2} END {print n}'; done
```

통과 기준: 성공 요청 하나에 `request_received` → `ai_call_start` → `ai_call_success`(`latency_ms` 포함) → `db_save_success`가 같은 `request_id`로 남는다. 실패 이벤트가 6단계 표와 같다. 마지막 반복의 출력이 모두 `0`이다(API 키·비밀번호·질문 내용이 로그에 없음).

## 8. 웹 화면 (4.1, 4.2) — 브라우저, 실제 OpenAI 호출 1회

브라우저 자동화 도구(Claude in Chrome, Playwright 등)로 `http://localhost:8010/`을 연다.

| 단계 | 기대 결과 |
| --- | --- |
| 첫 접속 | 로그인 폼(아이디·비밀번호·“로그인”·“회원가입으로”) |
| 없는 계정으로 로그인 | 폼 아래 “아이디 또는 비밀번호를 확인해 주세요.” |
| “회원가입으로” → 새 아이디로 “회원가입” | 채팅 화면(질문 입력창, “질문 보내기”, “로그아웃”) |
| 질문 입력 후 “질문 보내기” | 대기 중 질문과 “답변을 만들고 있어요...” 표시, 입력창·전송 버튼 비활성화 → 질문·답변·시각이 한 번만 기록에 표시 |
| A 계정의 질문 대기 중 로그아웃 → B 계정 로그인 (네트워크 대기·빠른 응답의 표시 대기 각각 확인) | B의 기록만 보이며, A의 늦은 응답·오류가 B의 화면이나 로그인 상태를 바꾸지 않음 |
| “동작 줄이기” 설정을 켜고 로그인·질문 전송 | 로딩·답변 대기 스피너가 회전하지 않고 자동 스크롤이 애니메이션 없이 이동 |
| 새로고침 | 같은 대화가 그대로 보인다 |
| “로그아웃” 후 새로고침 | 로그인 폼 |
| 콘솔 | 오류 없음 |

주의: Claude in Chrome에서 요소 참조(`ref`)로 입력란을 클릭하면 포커스가 잡히지 않아 입력이 비고, 브라우저의 `required` 검사로 제출이 막힐 수 있다. 스크린샷 좌표로 클릭하고 0.5초 기다린 뒤 입력하며, 제출 전에 확대 스크린샷으로 값이 들어갔는지 확인한다. 화면이 바뀌면 좌표·참조를 다시 확인한다.

## 9. 로컬에서 확인할 수 없는 항목

| 항목 | 확인 방법 |
| --- | --- |
| 외부 접속 (1, 4.6) | README의 서비스 URL에 다른 네트워크에서 접속해 2·4·8단계를 반복한다. URL이 “배포 후 기록 필요”이면 미충족 |
| 협업 (4.7) | `git branch -a`, `git log --merges --oneline`, `git shortlog -sn`, `gh pr list --state merged` — 기능 브랜치·PR 머지 기록, 팀원별 유의미한 커밋 10회 이상 |
| 팀 문서 (2, 4.7) | `docs/team.md`에 “미정/미작성”이 남아 있지 않고 Git 이력과 맞는다 |
| Docker 이미지 | Docker 데몬이 켜져 있으면 `docker build -t b7-check .` 성공 후 `docker image rm b7-check` |

## 10. 정리

```bash
while IFS= read -r server_pid; do
  kill "$server_pid" 2>/dev/null || true
  wait "$server_pid" 2>/dev/null || true
done < "$R/pids"
rm -rf "$R"
```

브라우저 탭을 열었다면 닫는다.

## 11. 보고 형식

| B7-1 항목 | 확인 내용 | 결과 |
| --- | --- | --- |
| 4.1 웹 UI | … | ✅ / ❌ |
| 4.2 인증·접근 제어 | … | |
| 4.3 AI 처리·문맥 | … | |
| 4.4 로그 저장·조회 | … | |
| 4.5 입력 검증·예외·로그 | … | |
| 4.6 배포 | … | |
| 4.7 협업 | … | |

실패 항목에는 실제 응답·로그 한 줄을 함께 적는다. 실행하지 못한 단계는 이유와 함께 “미확인”으로 적는다.
