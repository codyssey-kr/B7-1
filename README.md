# B7-1 AI 챗봇

로그인한 사용자가 웹 페이지에서 질문하면 FastAPI 서버가 네이토의 OpenAI 호환 Chat Completions API를 호출해 답변을 보여 주고, 질문과 답변을 SQLite에 저장하는 서비스입니다. 화면은 React로 만들고 같은 FastAPI 서버가 제공합니다.

- 대상: 개념 설명과 질의응답이 필요한 학습자
- 핵심 흐름: 회원가입 → 로그인 → 질문 → 답변 → 후속 질문(최근 대화 문맥 유지) → 재로그인 후 기록 조회
- 설계·ERD·API 예시: [설계 문서](docs/design.md)
- 과제 기준: [B7-1](docs/B7-1.md)
- 팀 역할·기여 기록: [팀 문서](docs/team.md)
- 외부 서비스 URL: http://3.38.152.93
- 제출용 GitHub URL: https://github.com/codyssey-kr/B7-1

## 로컬 실행

Python 3.12, [uv](https://docs.astral.sh/uv/), Node.js 24, pnpm 12.3.4를 사용합니다.

```bash
npm install --global pnpm@12.3.4
pnpm --dir frontend install --frozen-lockfile
pnpm --dir frontend build
uv sync --frozen --extra dev
cp .env.example .env   # .env 값을 설정합니다.
mkdir -p data
uv run --frozen uvicorn app.main:create_app --factory --reload
```

`http://localhost:8000`에서 가입하고 로그인합니다. DB 테이블은 서버 시작 시 자동으로 생성됩니다. 이전 버전에서 만든 `data/app.db`가 있다면 스키마가 다르므로 삭제한 뒤 실행하세요. 프런트를 수정했다면 `pnpm --dir frontend build`를 다시 실행합니다. 자동 반영 개발 서버는 위 서버를 띄운 상태에서 `pnpm --dir frontend dev`로 실행하고 `http://localhost:5173`에 접속합니다. Vite가 `/api`를 8000으로 프록시합니다.

## 환경 변수

| 이름 | 기본값 / 용도 |
| --- | --- |
| `OPENAI_API_KEY` | 필수. 네이토에서 발급한 API 키. `.env`에만 설정 |
| `OPENAI_BASE_URL` | `https://copa.codyssey.kr/v1`; 네이토 API 기본 URL. `/chat/completions`는 붙이지 않음 |
| `AI_MODEL` | `gpt-5-mini`; 네이토에서 사용할 모델 ID |
| `AI_TIMEOUT_SECONDS` | `30`; AI API 호출 시간 제한(초), 자동 재시도 없음 |
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/app.db`; Compose는 `/data/app.db` |

`.env.example`에는 키 값을 넣지 않았습니다. `.env`를 만든 뒤 `OPENAI_API_KEY`에 네이토 키를 설정하세요. 변수명이 `OPENAI_*`인 것은 OpenAI 호환 API 클라이언트를 사용하기 때문이며, 이 프로젝트는 `OPENAI_BASE_URL`로 네이토 서버를 지정합니다.

## 민감정보 관리

- API 키는 `.env`에만 두고 코드·문서·이미지에 넣지 않습니다. `.env.example`에는 변수 이름과 비밀이 아닌 기본값만 있습니다.
- `.gitignore`와 `.dockerignore`가 `.env`, DB 파일, 로그를 제외합니다.
- AI API 호출은 서버에서만 하며 브라우저에는 답변만 전달합니다.
- 로그에는 질문·답변·비밀번호를 남기지 않습니다.
- **비밀번호는 DB에 평문으로 저장되고 HTTP로 전송됩니다. 실제로 쓰는 비밀번호를 사용하지 마세요.**

## 도커 실행

Docker Engine과 Compose를 설치한 뒤 **저장소 루트**에서 실행합니다.

```bash
cp .env.example .env   # .env 값을 설정합니다.
docker compose up --build -d
docker compose logs --tail=100 app
```

`app` 컨테이너가 80 포트로 FastAPI를 제공하고 SQLite를 `app_data` 볼륨의 `/data/app.db`에 저장합니다. 서비스 URL은 `http://<인스턴스 공인 IP 또는 도메인>`입니다.

운영 서버 준비·업데이트·DB 초기화·배포 확인은 [배포 및 운영 문서](infra/README.md)를 참고하세요.

## API

전체 스키마는 서버의 `/docs`, 요청·응답 예시는 [설계의 API 명세](docs/design.md#6-api-명세)를 참고하세요. 로그인하지 않은 채팅 요청은 `401`을 반환합니다.

| 메서드 | 경로 | 동작 |
| --- | --- | --- |
| POST | `/api/auth/signup` | `{username, password}`로 가입 |
| POST | `/api/auth/login` | 로그인, `session` 쿠키 발급 |
| POST | `/api/auth/logout` | 세션 폐기 |
| POST | `/api/chat` | `{question}`으로 질문, 저장된 문답 반환 |
| GET | `/api/me/chats` | 내 대화 로그 조회 |

```bash
curl -sS http://localhost:8000/api/auth/signup -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"local-review-only"}'
curl -sS -c cookies.txt http://localhost:8000/api/auth/login -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"local-review-only"}'
curl -sS -b cookies.txt http://localhost:8000/api/chat -H 'Content-Type: application/json' \
  -d '{"question":"FastAPI에서 라우터가 뭐야?"}'
curl -sS -b cookies.txt http://localhost:8000/api/me/chats
```

## DB 확인

`chats` 테이블에 사용자 ID, 생성 시각, 질문, 답변이 저장됩니다. `scripts/check_logs.sql`은 특정 사용자의 최근 대화 20건을 조회합니다.

로컬에서는 저장소 루트에서 실행합니다.

```bash
sqlite3 data/app.db '.parameter init' '.parameter set :user_id 1' '.read scripts/check_logs.sql'
```

운영 서버의 조회 방법은 [운영 DB 조회](infra/README.md#운영-db-조회)를 참고하세요.

## 검증

```bash
pnpm --dir frontend check
pnpm --dir frontend build
uv run --frozen ruff check app
```

자동 테스트는 두지 않습니다. B7-1 요구사항별 확인 절차는 [점검 시나리오](docs/check-scenario.md)에 있으며, AI 에이전트나 사람이 그대로 따라 실행할 수 있습니다.
