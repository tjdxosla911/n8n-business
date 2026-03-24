# AWS Lightsail + n8n 서버 세팅 가이드

> 최종 확인: 2026-03-13 | 검증 완료된 설치 기준

---

## 개요

이 가이드대로 진행하면 AWS Lightsail에 아래 환경이 자동 구성됩니다.

| 항목 | 내용 |
|------|------|
| 서버 | AWS Lightsail (Ubuntu 22.04) |
| 스펙 | 2 vCPU / 2GB RAM (medium_3_0) |
| 리전 | ap-northeast-2 (서울) |
| n8n | Docker 컨테이너 (포트 5678) |
| DB | PostgreSQL 15 (Docker 컨테이너) |
| 설치 | Docker, Docker Compose, Node.js 18, Claude Code CLI |

---

## 사전 준비

- [ ] **Node.js v18 이상** 설치 ([nodejs.org](https://nodejs.org))
- [ ] **AWS IAM 계정** — Lightsail 전체 권한 필요
  - `AmazonLightsailFullAccess` 정책 부여
- [ ] **AWS Access Key ID / Secret Access Key** 발급

> **보안 주의:** Access Key는 코드에 직접 넣지 말고 사용 후 환경변수나 `~/.aws/credentials`로 이동하세요.

---

## 1단계: 설치 스크립트 준비

### 1-1. 작업 폴더 생성 및 패키지 설치

```bash
mkdir lightsail-setup && cd lightsail-setup
```

`package.json` 생성:

```json
{
  "name": "lightsail-setup",
  "version": "1.0.0",
  "dependencies": {
    "@aws-sdk/client-lightsail": "^3.0.0"
  }
}
```

```bash
npm install
```

---

### 1-2. 세팅 스크립트 작성

`setup.mjs` 파일 생성 — 아래 내용에서 **대문자로 표시된 값만 수정**하세요:

```js
import {
  LightsailClient,
  CreateInstancesCommand,
  GetInstanceCommand,
  OpenInstancePublicPortsCommand,
  AllocateStaticIpCommand,
  AttachStaticIpCommand,
  GetStaticIpCommand,
} from "@aws-sdk/client-lightsail";

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
//  수정 필요한 값
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
const CONFIG = {
  accessKeyId:     "YOUR_ACCESS_KEY_ID",
  secretAccessKey: "YOUR_SECRET_ACCESS_KEY",
  region:          "ap-northeast-2",
  instanceName:    "n8n-server",
  staticIpName:    "n8n-server-ip",
  availabilityZone:"ap-northeast-2a",
  n8nUser:         "admin",
  n8nPassword:     "YOUR_N8N_PASSWORD",  // 반드시 변경!
  dbPassword:      "YOUR_DB_PASSWORD",   // 반드시 변경!
};
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

const client = new LightsailClient({
  region: CONFIG.region,
  credentials: {
    accessKeyId: CONFIG.accessKeyId,
    secretAccessKey: CONFIG.secretAccessKey,
  },
});

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function waitForInstance(name, targetState = "running") {
  console.log(`  Waiting for '${name}' → ${targetState}...`);
  for (let i = 0; i < 60; i++) {
    try {
      const res = await client.send(new GetInstanceCommand({ instanceName: name }));
      const state = res.instance?.state?.name;
      process.stdout.write(`\r  state: ${state}          `);
      if (state === targetState) { console.log(); return res.instance; }
    } catch (e) {}
    await sleep(5000);
  }
  throw new Error("Timeout");
}

function notFound(e) {
  return e.message?.includes("does not exist") || e.message?.includes("not found");
}

const dockerComposeYml = `
services:
  postgres:
    image: postgres:15
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${CONFIG.dbPassword}
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${CONFIG.dbPassword}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${CONFIG.n8nUser}
      - N8N_BASIC_AUTH_PASSWORD=${CONFIG.n8nPassword}
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_SECURE_COOKIE=false
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
  n8n_data:
`.trim();

const userData = `#!/bin/bash
set -e
apt-get update -y
apt-get install -y ca-certificates curl gnupg lsb-release

# Docker
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable docker && systemctl start docker
usermod -aG docker ubuntu

# Docker Compose standalone
curl -SL "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs

# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# n8n 실행
mkdir -p /opt/n8n
cat > /opt/n8n/docker-compose.yml << 'DCEOF'
${dockerComposeYml}
DCEOF

cd /opt/n8n && docker compose up -d
`;

async function main() {
  // 1. 인스턴스 확인 / 생성
  console.log("\n[1/4] 인스턴스 확인...");
  let instance = null;
  try {
    instance = (await client.send(new GetInstanceCommand({ instanceName: CONFIG.instanceName }))).instance;
    console.log(`  이미 존재함 (state: ${instance.state?.name})`);
  } catch (e) {
    if (!notFound(e)) throw e;
    console.log("  없음 → 생성 중...");
    await client.send(new CreateInstancesCommand({
      instanceNames: [CONFIG.instanceName],
      availabilityZone: CONFIG.availabilityZone,
      blueprintId: "ubuntu_22_04",
      bundleId: "medium_3_0",
      userData,
    }));
    await sleep(20000);
    instance = await waitForInstance(CONFIG.instanceName, "running");
    console.log(`  생성 완료: ${instance.publicIpAddress}`);
  }

  // 2. 방화벽 포트 오픈
  console.log("\n[2/4] 방화벽 포트 오픈...");
  const ports = [22, 80, 443, 5678, 3100];
  for (const port of ports) {
    try {
      await client.send(new OpenInstancePublicPortsCommand({
        instanceName: CONFIG.instanceName,
        portInfo: { fromPort: port, toPort: port, protocol: "tcp", cidrs: ["0.0.0.0/0"] },
      }));
      console.log(`  ✓ ${port}/tcp`);
    } catch (e) {
      console.log(`  ! ${port}: ${e.message}`);
    }
  }

  // 3. 고정 IP 할당 및 연결
  console.log("\n[3/4] 고정 IP...");
  let staticIp = null;
  try {
    staticIp = (await client.send(new GetStaticIpCommand({ staticIpName: CONFIG.staticIpName }))).staticIp;
    console.log(`  이미 존재함: ${staticIp.ipAddress}`);
  } catch (e) {
    if (!notFound(e)) throw e;
    await client.send(new AllocateStaticIpCommand({ staticIpName: CONFIG.staticIpName }));
    await sleep(5000);
    staticIp = (await client.send(new GetStaticIpCommand({ staticIpName: CONFIG.staticIpName }))).staticIp;
    console.log(`  할당됨: ${staticIp.ipAddress}`);
  }
  if (!staticIp.attachedTo) {
    await client.send(new AttachStaticIpCommand({ staticIpName: CONFIG.staticIpName, instanceName: CONFIG.instanceName }));
    await sleep(5000);
    staticIp = (await client.send(new GetStaticIpCommand({ staticIpName: CONFIG.staticIpName }))).staticIp;
    console.log(`  연결됨: ${staticIp.attachedTo}`);
  }

  // 4. 완료
  console.log("\n" + "=".repeat(55));
  console.log("설치 완료!");
  console.log(`고정 IP   : ${staticIp.ipAddress}`);
  console.log(`n8n 접속  : http://${staticIp.ipAddress}:5678`);
  console.log(`로그인    : ${CONFIG.n8nUser} / ${CONFIG.n8nPassword}`);
  console.log("=".repeat(55));
  console.log("\n⚠ 서버 초기화 완료까지 약 5분 소요됩니다.");
  console.log("  (Docker 설치 + 컨테이너 시작 시간)");
}

main().catch((e) => { console.error("ERROR:", e.message); process.exit(1); });
```

---

## 2단계: 실행

```bash
node setup.mjs
```

완료 출력 예시:
```
=======================================================
설치 완료!
고정 IP   : xxx.xxx.xxx.xxx
n8n 접속  : http://xxx.xxx.xxx.xxx:5678
로그인    : admin / YOUR_N8N_PASSWORD
=======================================================
⚠ 서버 초기화 완료까지 약 5분 소요됩니다.
```

---

## 3단계: 접속 확인

5분 후 브라우저에서 접속:

```
http://[고정IP]:5678
```

- 반드시 `http://` 로 접속 (https 아님)
- n8n 로그인 화면이 나오면 성공

---

## 서버 관리 명령어

SSH 접속:
```bash
# Lightsail 기본 키 다운로드 후
ssh -i lightsail_key.pem ubuntu@[고정IP]
```

컨테이너 관리:
```bash
cd /opt/n8n

docker compose ps                  # 상태 확인
docker compose logs -f n8n         # n8n 로그
docker compose logs -f postgres    # DB 로그
docker compose restart n8n         # 재시작
docker compose down                # 중지
docker compose up -d               # 시작
```

docker-compose.yml 수정 후 적용:
```bash
cd /opt/n8n
sudo nano docker-compose.yml
sudo docker compose up -d --force-recreate n8n
```

---

## 오픈 포트 목록

| 포트 | 용도 |
|------|------|
| 22 | SSH |
| 80 | HTTP (도메인 연결 후 사용) |
| 443 | HTTPS (SSL 설정 후 사용) |
| 5678 | n8n |
| 3100 | 브릿지 서버 |

---

## 알려진 이슈 및 해결법

### n8n 로그인 시 "Secure Cookie" 경고
HTTP로 운영 시 발생. `docker-compose.yml`에 아래 환경변수 추가 후 재시작:
```yaml
- N8N_SECURE_COOKIE=false
```
> 도메인 + HTTPS 설정 시 이 값을 제거하고 `N8N_PROTOCOL=https`로 변경

### 접속이 안 될 때 확인 순서
1. `http://` 로 접속하고 있는지 확인 (https 아님)
2. 인스턴스 생성 후 5분 기다렸는지 확인
3. SSH 접속 후 `sudo docker ps` 로 컨테이너 실행 여부 확인
4. `sudo docker logs n8n` 으로 에러 메시지 확인

---

## 다음 단계 (도메인 + SSL)

1. 도메인 DNS A레코드 → 고정 IP 연결
2. 서버에 Nginx 설치
3. Let's Encrypt로 SSL 인증서 발급 (`certbot`)
4. Nginx 리버스 프록시 설정 (80/443 → 5678)
5. `docker-compose.yml` 수정:
   - `N8N_SECURE_COOKIE=false` 제거
   - `N8N_PROTOCOL=https`
   - `N8N_HOST=도메인주소`
6. 방화벽에서 SSH(22), n8n(5678) IP 제한

---

## 설치된 소프트웨어 버전 기준

| 소프트웨어 | 버전 |
|-----------|------|
| Ubuntu | 22.04 LTS |
| Docker CE | latest |
| Docker Compose | v2.24.0 |
| Node.js | 18.x |
| Claude Code CLI | latest |
| n8n | latest |
| PostgreSQL | 15 |
