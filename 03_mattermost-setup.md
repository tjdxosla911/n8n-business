# [실행 가이드 03] Mattermost 설치 + Nginx SSL + 초기 설정

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01_lightsail-n8n-setup.md` 작업 완료된 서버에서 진행하세요.

---

## 사전 조건

- [ ] 01번 작업 완료 (n8n 서버 실행 중, Docker 설치됨)
- [ ] 가비아(또는 사용 중인 DNS)에 A레코드 추가: `chat` → `서버 고정IP`
- [ ] DNS 전파 확인 후 진행

---

## Claude에게 전달할 정보

```
서버 고정 IP      :
서버 SSH 키 경로  :        (예: C:\Users\...\lightsail_key.pem)
Mattermost 도메인 :        (예: chat.example.com)
n8n 도메인        :        (예: n8n.example.com)
관리자 이메일     :
n8n 웹훅 URL      :        (예: https://n8n.example.com/webhook/content-generate)
```

---

## Claude 실행 내용 (참고용)

### 1단계: Mattermost Docker 설치

`/opt/mattermost/docker-compose.yml` 생성 후 실행:

```yaml
services:
  mattermost-db:
    image: postgres:15
    container_name: mattermost-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: mattermost
      POSTGRES_PASSWORD: mm_password_2024    # ← 변경 권장
      POSTGRES_DB: mattermost
    volumes:
      - mattermost_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mattermost"]
      interval: 10s
      timeout: 5s
      retries: 5

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    container_name: mattermost
    restart: unless-stopped
    depends_on:
      mattermost-db:
        condition: service_healthy
    ports:
      - "8065:8065"
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: postgres://mattermost:mm_password_2024@mattermost-db:5432/mattermost?sslmode=disable
      MM_SERVICESETTINGS_SITEURL: https://[MATTERMOST_DOMAIN]
      MM_SERVICESETTINGS_LISTENADDRESS: :8065
      MM_PLUGINSETTINGS_ENABLE: "true"
      MM_LOGSETTINGS_CONSOLELEVEL: INFO
    volumes:
      - mattermost_data:/mattermost/data
      - mattermost_logs:/mattermost/logs
      - mattermost_config:/mattermost/config
      - mattermost_plugins:/mattermost/plugins

volumes:
  mattermost_db_data:
  mattermost_data:
  mattermost_logs:
  mattermost_config:
  mattermost_plugins:
```

```bash
cd /opt/mattermost && sudo docker compose up -d
# healthy 상태까지 대기 (~1분)
```

---

### 2단계: Nginx + SSL

> ⚠ Nginx 설정 파일은 반드시 **로컬 파일 생성 후 SCP 전송** 방식 사용
> (SSH heredoc 내 `$` 변수가 쉘에서 치환되는 문제 방지)

**로컬에 `chat_nginx.conf` 생성:**

```nginx
server {
    listen 80;
    server_name [MATTERMOST_DOMAIN];
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name [MATTERMOST_DOMAIN];

    ssl_certificate     /etc/letsencrypt/live/[MATTERMOST_DOMAIN]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[MATTERMOST_DOMAIN]/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 100m;

    location / {
        proxy_pass         http://localhost:8065;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
    }
}
```

**SSL 발급 및 적용:**

```bash
# SSL 인증서 발급
sudo certbot certonly --nginx -d [MATTERMOST_DOMAIN] \
  --non-interactive --agree-tos -m [ADMIN_EMAIL]

# 설정 SCP 전송 후
sudo cp /tmp/chat_nginx.conf /etc/nginx/sites-available/chat
sudo ln -sf /etc/nginx/sites-available/chat /etc/nginx/sites-enabled/chat
sudo nginx -t && sudo systemctl reload nginx
```

---

### 3단계: 초기 설정 (API)

모두 `http://localhost:8065/api/v4` 엔드포인트로 실행:

```bash
BASE="http://localhost:8065/api/v4"

# 관리자 계정 생성
curl -X POST "$BASE/users" \
  -H "Content-Type: application/json" \
  -d '{"email":"[ADMIN_EMAIL]","username":"admin","password":"[ADMIN_PASSWORD]","first_name":"Admin"}'

# 로그인 & 토큰 취득
LOGIN=$(curl -sfi -X POST "$BASE/users/login" \
  -H "Content-Type: application/json" \
  -d '{"login_id":"[ADMIN_EMAIL]","password":"[ADMIN_PASSWORD]"}')
TOKEN=$(echo "$LOGIN" | grep -i "^Token:" | awk '{print $2}' | tr -d '\r')

# system_admin 권한 부여 (PostgreSQL 직접)
sudo docker exec mattermost-db psql -U mattermost -d mattermost -c \
  "UPDATE users SET roles='system_user system_admin' WHERE username='admin';"

# 봇 계정 생성 활성화 (필수!)
curl -X PUT "$BASE/config/patch" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ServiceSettings":{"EnableBotAccountCreation":true}}'

# 팀 생성
TEAM=$(curl -sf -X POST "$BASE/teams" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"[TEAM_NAME]","display_name":"[TEAM_DISPLAY_NAME]","type":"O"}')
TEAM_ID=$(echo "$TEAM" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# 채널 생성
for CH in "콘텐츠생성:content-create" "알림:notifications" "시스템:system-logs"; do
  DISPLAY="${CH%%:*}"; NAME="${CH##*:}"
  curl -sf -X POST "$BASE/channels" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"team_id\":\"$TEAM_ID\",\"name\":\"$NAME\",\"display_name\":\"$DISPLAY\",\"type\":\"O\"}"
done

# 봇 생성
BOT=$(curl -sf -X POST "$BASE/bots" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username":"n8n-bot","display_name":"n8n Bot","description":"n8n 자동화 봇"}')
BOT_USER_ID=$(echo "$BOT" | python3 -c "import sys,json; print(json.load(sys.stdin)['user_id'])")

# 봇 토큰 생성
BOT_TOKEN=$(curl -sf -X POST "$BASE/users/$BOT_USER_ID/tokens" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description":"n8n integration token"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 봇 팀 추가
curl -sf -X POST "$BASE/teams/$TEAM_ID/members" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"team_id\":\"$TEAM_ID\",\"user_id\":\"$BOT_USER_ID\"}"

# ⚠ 봇을 각 채널에도 추가 (필수! 팀 멤버여도 채널 비멤버면 POST 시 403 Forbidden)
CONTENT_CH_ID=$(curl -sf "$BASE/teams/$TEAM_ID/channels/name/content-create" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
NOTIF_CH_ID=$(curl -sf "$BASE/teams/$TEAM_ID/channels/name/notifications" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

for CH_ID in "$CONTENT_CH_ID" "$NOTIF_CH_ID"; do
  curl -sf -X POST "$BASE/channels/$CH_ID/members" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"user_id\":\"$BOT_USER_ID\"}"
done
echo "✓ 봇 채널 추가 완료"

# Slash Command 생성
curl -sf -X POST "$BASE/commands" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"team_id\": \"$TEAM_ID\",
    \"trigger\": \"콘텐츠생성\",
    \"method\": \"P\",
    \"url\": \"[N8N_WEBHOOK_URL]\",
    \"username\": \"n8n Bot\",
    \"description\": \"n8n으로 콘텐츠 생성\",
    \"display_name\": \"콘텐츠생성\"
  }"
```

---

## 완료 후 출력 정보

```
Mattermost URL  : https://[MATTERMOST_DOMAIN]
관리자 계정     : [ADMIN_EMAIL] / [ADMIN_PASSWORD]
Bot Token       : [BOT_TOKEN]
Team ID         : [TEAM_ID]
콘텐츠생성 채널 ID : [CHANNEL_ID]
알림 채널 ID    : [CHANNEL_ID]
시스템 채널 ID  : [CHANNEL_ID]
```

---

## n8n Credential 등록 정보

n8n → Settings → Credentials → Mattermost API:

| 항목 | 값 |
|------|-----|
| Server URL | `https://[MATTERMOST_DOMAIN]` |
| Access Token | `[BOT_TOKEN]` |

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 봇 생성 API 실패 | `EnableBotAccountCreation=false` (기본값) | config patch API로 활성화 후 재시도 |
| system_admin 권한 부여 실패 | mmctl `role` 서브커맨드 없음 | PostgreSQL UPDATE 직접 실행 |
| Nginx `invalid number of arguments` | SSH heredoc 내 `$` 변수 치환 | 로컬 파일 생성 후 SCP 전송 방식 사용 |
| Mattermost 접속 안 됨 | healthy 상태 아직 아님 | `docker inspect --format='{{.State.Health.Status}}' mattermost` 로 확인 후 대기 |
| n8n Mattermost노드 403 Forbidden "You do not have the appropriate permissions" | 봇이 팀 멤버이지만 채널 멤버가 아님 (팀 추가 ≠ 채널 추가) | Admin 토큰으로 `POST /api/v4/channels/{channelId}/members` 호출해 봇을 채널에 추가 |

---

## 완료 체크리스트

- [ ] `https://[MATTERMOST_DOMAIN]` 접속 성공
- [ ] 관리자 로그인 성공
- [ ] 팀 및 채널 (#콘텐츠생성, #알림, #시스템) 생성 확인
- [ ] n8n에서 Mattermost Credential 등록
- [ ] `/콘텐츠생성` Slash Command 동작 확인
