# 팀 역할 및 기여 기록

팀원별 담당 역할과 실제 작업 요약을 기록한다.

| 팀원 | 담당 기능 | 개인 작업 요약 |
| --- | --- | --- |
| @parkhojeong | 인증·세션·접근 제어, 기능 통합·리뷰 | 회원가입·로그인·로그아웃과 서버 세션 인증 구현, 사용자별 접근 제어, AI 오류·계정 전환 처리 보완 ([#7](https://github.com/codyssey-kr/B7-1/pull/7), [#13](https://github.com/codyssey-kr/B7-1/pull/13)) |
| @SeouliteParker | AI 채팅 (질문 처리·문맥 구성·OpenAI 호출·대화 로그) | AI 호출 실패 원인·소요 시간 로그 추가, 문맥 글자 수 상한(4,000자) 적용, 시스템 프롬프트를 `prompts.py`로 분리, 개발 초보자 기준 시스템 프롬프트 개선 ([#1](https://github.com/codyssey-kr/B7-1/pull/1), [#2](https://github.com/codyssey-kr/B7-1/pull/2), [#3](https://github.com/codyssey-kr/B7-1/pull/3), [#6](https://github.com/codyssey-kr/B7-1/pull/6)) |
| solbao-dev | 프론트엔드 화면·UX 개선, 팀 Git/PR 협업 규칙 문서화 | 초보자용 화면·예시 질문, 인증·질문 입력·응답 대기·학습 기록 UX 개선; Git/PR 협업 규칙 작성 ([#4](https://github.com/codyssey-kr/B7-1/pull/4), [#5](https://github.com/codyssey-kr/B7-1/pull/5), [#13](https://github.com/codyssey-kr/B7-1/pull/13)) |
| @1st0Groom | 배포·로그·통합 검증 | EC2 배포 구성·Docker Compose 운영, Buildx 설치·SQLite 조회 절차 문서화, 배포 검증 기록 작성 ([#16](https://github.com/codyssey-kr/B7-1/pull/16), [#17](https://github.com/codyssey-kr/B7-1/pull/17)) |

팀 인원에 맞게 역할을 조정하고 각 기능 담당자가 테스트와 문서를 함께 작성한다. 기능 브랜치 → PR → 다른 팀원 검토 → merge commit 순서로 통합한다. 팀원별 유의미한 커밋 10회 이상과 PR 기반 머지 기록은 실제 Git 이력으로 증빙한다. 자동 생성된 초기 구현이나 이 문서의 역할 예시만으로 개인 기여 요건이 충족되지는 않는다.

## Git / PR 협업 규칙

아래는 위 통합 흐름을 일관되게 따르기 위한 팀 협업 규칙이며, 과제의 공식 필수 규칙을 새로 정하는 내용은 아니다.

### 브랜치

- `main`에서 직접 작업하지 않고, 작업 목적별로 별도 브랜치를 만든다.
- 하나의 브랜치는 하나의 명확한 작업 목적만 다룬다. 예: `feat/learning-chat-ui`, `docs/team-pr-guidelines`.

### 커밋

- 기능이나 작업 목적에 따라 의미 있는 단위로 커밋한다. 커밋 수를 늘리기 위한 불필요한 분할은 하지 않는다.
- 커밋 메시지만 보고도 변경 목적을 알 수 있게 작성한다.

### PR (Pull Request)

- 작업 브랜치는 PR을 통해 `main`에 병합하고, 병합 전 다른 팀원의 검토를 받는다.
- PR에는 가능하면 작업 목적, 주요 변경 사항, 작업 커밋, 테스트 및 확인 결과, 변경하지 않은 범위 또는 영향 범위를 기록한다.
- 관련 없는 작업을 하나의 PR에 섞지 않는다. PR 생성 전 변경 파일과 커밋을 확인한다.

### 기본 작업 흐름

`main` 최신화 → 작업 브랜치 생성 → 구현 및 확인 → 의미 있는 단위로 커밋 → fork의 작업 브랜치에 push → PR 생성 → 다른 팀원 검토 → `main` 병합
