# 배포 및 운영

앱 실행·업데이트·DB 조회·백업 명령은 **저장소 루트**에서 실행합니다. SSH 접속과 백업 다운로드는 각 절차에 표시한 컴퓨터에서 실행합니다. 환경 변수는 [루트 README](../README.md#환경-변수), 공통 컨테이너 실행 명령은 [도커 실행](../README.md#도커-실행)을 참고합니다.

## EC2 스택 생성

[ec2.yaml](ec2.yaml)은 기본 VPC에 EC2 1대와 보안 그룹을 만드는 CloudFormation 템플릿입니다. 서울 리전(`ap-northeast-2`)에서 사용합니다. 기본값은 이미지 빌드 여유를 위한 `t3.small`이며, AWS 사용량은 계정 크레딧을 차감하거나 요금이 발생할 수 있으니 생성 전에 확인하고 평가가 끝나면 스택을 삭제합니다.

먼저 서울 리전에 EC2 키 페어가 있는지 확인합니다. 없다면 EC2 콘솔의 **키 페어**에서 새 키 페어를 만들고 내려받은 `.pem` 파일을 저장소 밖의 안전한 곳에 보관합니다. 개인 키는 다시 내려받을 수 없으므로 저장소나 채팅에 올리지 않습니다. 현재 공인 IPv4 주소도 확인합니다.

AWS 콘솔에서 **CloudFormation → 스택 생성 → 새 리소스 사용(표준)**을 선택하고 `infra/ec2.yaml`을 업로드합니다. 다음 값을 설정합니다.

| 파라미터 | 값 |
| --- | --- |
| `KeyName` | 앞에서 확인한 EC2 키 페어 |
| `SshCidr` | 현재 공인 IPv4 주소에 `/32`를 붙인 값, 예: `203.0.113.10/32` |
| `InstanceType` | 기본 `t3.small` 권장 |
| `AmazonLinux2023Ami` | 기본값 유지 |

스택을 생성한 뒤 **Outputs**의 `PublicIp`와 `ServiceUrl`을 확인합니다. HTTP 80 포트는 외부 공개이고 SSH 22 포트는 지정한 주소에서만 접속됩니다. 템플릿은 기본 VPC와 기본 서브넷을 사용합니다. 서울 리전에 두 리소스가 있고, 기본 서브넷의 공인 IPv4 자동 할당과 인터넷 게이트웨이 경로가 유지되어 있어야 합니다.

## 서버 접속과 Docker 준비

내 컴퓨터 터미널에서 개인 키 권한을 제한하고 인스턴스에 접속합니다.

```bash
chmod 400 /path/to/key.pem
PUBLIC_IP="REPLACE_WITH_PUBLIC_IP"  # CloudFormation Outputs의 PublicIp 값으로 교체
ssh -i /path/to/key.pem "ec2-user@$PUBLIC_IP"
```

인스턴스 터미널에서 Docker를 설치하고 시작합니다.

```bash
sudo dnf install -y docker git nano
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
exit
```

SSH로 다시 접속한 뒤 Docker Compose와 Buildx 플러그인을 설치합니다. [Compose 빌드는 Buildx 0.17.0 이상이 필요](https://github.com/docker/compose/blob/v5.6.0/pkg/compose/api_versions.go)하므로 Amazon Linux의 기본 플러그인 대신 아래 버전을 함께 설치합니다. 두 플러그인은 Docker CLI에 사용자별로 설치됩니다.

```bash
mkdir -p ~/.docker/cli-plugins
COMPOSE_VERSION=v5.6.0
BUILDX_VERSION=v0.37.2
curl -fSL "https://github.com/docker/compose/releases/download/$COMPOSE_VERSION/docker-compose-linux-x86_64" \
  -o ~/.docker/cli-plugins/docker-compose
curl -fSL "https://github.com/docker/buildx/releases/download/$BUILDX_VERSION/buildx-$BUILDX_VERSION.linux-amd64" \
  -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-compose ~/.docker/cli-plugins/docker-buildx
docker compose version
docker buildx version
```

## 앱 실행

PR이 공개 저장소의 기본 브랜치에 머지된 뒤 저장소를 내려받고 서비스를 시작합니다.

```bash
git clone https://github.com/codyssey-kr/B7-1.git
cd B7-1
cp .env.example .env
nano .env   # .env 값을 설정합니다.
docker compose up --build -d
docker compose ps
docker compose logs --tail=100 app
```

서비스 주소는 `http://<PublicIp>`입니다. 실행 후 아래 [배포 확인](#배포-확인) 절차를 진행합니다.

## 업데이트와 DB 초기화

코드를 업데이트할 때는 저장소에서 `git pull`로 최신 코드를 받습니다. 이전 배포에서 `.env`에 `AI_API_KEY`·`AI_BASE_URL`을 사용했다면 같은 값을 각각 `OPENAI_API_KEY`·`OPENAI_BASE_URL`로 옮깁니다. `AI_MODEL`은 그대로 사용합니다. 설정을 확인한 뒤 `docker compose up --build -d`를 실행합니다.

`docker compose down -v`는 DB 볼륨까지 삭제하므로 평소에는 사용하지 않습니다. 마이그레이션 도구가 없으므로 DB 스키마(`app/models.py`)를 바꾸면 DB를 초기화해야 하며 저장된 계정·대화가 모두 삭제됩니다. 로컬은 `data/app.db`를 지우고, 서버는 `docker compose down -v && docker compose up --build -d`를 실행합니다.

## 운영 DB 조회

운영 서버에서는 저장소 루트에서 아래 명령을 실행합니다. 컨테이너의 Python으로 DB를 읽기 전용으로 열어 조회하므로 실행 중인 DB 파일을 복사하거나 서버에 `sqlite3` CLI를 추가 설치할 필요가 없습니다. 끝의 `1`은 조회할 계정의 `user_id`로 바꾸며, 첫 출력의 `users` 목록에서 ID를 확인할 수 있습니다.

```bash
docker compose exec -T app python -c '
import sqlite3
import sys

with sqlite3.connect("file:/data/app.db?mode=ro", uri=True) as db:
    print("users:", db.execute("SELECT id, username FROM users").fetchall())
    rows = db.execute(sys.stdin.read(), {"user_id": int(sys.argv[1])})
    print(*(column[0] for column in rows.description), sep="\t")
    for row in rows:
        print(*row, sep="\t")
' 1 < scripts/check_logs.sql
```

조회 결과에는 사용자와 대화 내용이 포함되므로 외부에 공유하지 않습니다.

## DB 백업과 스택 정리

대화 DB는 EC2 내부의 Docker 볼륨 `app_data`에 있습니다. 실행 중인 DB 파일을 직접 복사하지 않고 [SQLite 백업 API](https://www.sqlite.org/backup.html)로 일관된 복사본을 만듭니다. 인스턴스의 저장소 디렉터리에서 실행하고, 오류 없이 끝났을 때만 다음 단계로 진행합니다.

```bash
docker compose exec -T app python - <<'PYBACKUP'
import sqlite3

source = sqlite3.connect("file:/data/app.db?mode=ro", uri=True)
with sqlite3.connect("/data/app-backup.db") as backup:
    source.backup(backup)
source.close()
PYBACKUP
docker compose cp app:/data/app-backup.db ./app-backup.db
chmod 600 app-backup.db
```

백업도 인스턴스 안에 있으면 스택 삭제 시 함께 사라집니다. **내 컴퓨터의 새 터미널**에서 저장소 밖으로 내려받고, 파일이 정상인지 확인한 뒤 스택을 삭제합니다. `PUBLIC_IP`는 Outputs의 값으로 바꿉니다.

```bash
PUBLIC_IP="REPLACE_WITH_PUBLIC_IP"
mkdir -p ~/b7-backups
chmod 700 ~/b7-backups
BACKUP_FILE="$HOME/b7-backups/app-$(date -u +%Y%m%dT%H%M%SZ).db"
scp -i /path/to/key.pem "ec2-user@$PUBLIC_IP:~/B7-1/app-backup.db" "$BACKUP_FILE"
chmod 600 "$BACKUP_FILE"
python3 - "$BACKUP_FILE" <<'PYCHECK'
import sqlite3
import sys
from pathlib import Path

with sqlite3.connect(Path(sys.argv[1]).resolve().as_uri() + "?mode=ro", uri=True) as db:
    result = db.execute("PRAGMA integrity_check").fetchall()
    assert result == [("ok",)], result
    print("backup OK; chats:", db.execute("SELECT COUNT(*) FROM chats").fetchone()[0])
PYCHECK
```

백업에는 계정·세션·대화가 들어 있으므로 공개하거나 저장소에 추가하지 않습니다. 작업을 마치면 CloudFormation에서 해당 스택을 삭제합니다. EC2와 루트 디스크도 삭제되므로 외부에 백업하지 않은 DB는 복구할 수 없습니다. 업데이트와 DB 초기화 시 주의점은 [업데이트와 DB 초기화](#업데이트와-db-초기화) 절차를 따릅니다.

## 배포 확인

배포 후 외부 네트워크에서 가입·로그인·실제 AI 질문·재시작 후 기록 조회를 확인합니다.

서비스 주소는 [루트 README](../README.md)에 기록합니다.

## 배포 기록

- **배포 검증(2026-10-08, @1st0Groom 기록):** 외부 접속, 로그인 후 네이토 AI 응답, 앱 재시작 후 대화 기록 유지 확인
