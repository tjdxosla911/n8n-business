# [실행 가이드 01] AWS Lightsail + n8n 서버 구축

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.

---

## Claude에게 전달할 정보

```
AWS Access Key ID     :
AWS Secret Access Key :
인스턴스명            : n8n-server          ← 원하면 변경
리전                  : ap-northeast-2      ← 원하면 변경
가용 영역             : ap-northeast-2a     ← 원하면 변경
n8n 로그인 ID         : admin               ← 원하면 변경
n8n 비밀번호          :                     ← 반드시 입력
DB 비밀번호           :                     ← 반드시 입력
```

---

## Claude 실행 내용 (참고용)

### 환경
- Node.js + `@aws-sdk/client-lightsail` 사용 (AWS CLI 불필요)
- Windows 환경에서 PowerShell 스크립트로 실행

### 작업 순서

**1. AWS SDK 설치**
```bash
mkdir lightsail-setup && cd lightsail-setup
npm install @aws-sdk/client-lightsail
```

**2. 세팅 스크립트 실행**

아래 스크립트에서 CONFIG 값을 채운 뒤 `node setup.mjs`로 실행:

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

const CONFIG = {
  accessKeyId:      "ACCESS_KEY_ID",
  secretAccessKey:  "SECRET_ACCESS_KEY",
  region:           "ap-northeast-2",
  instanceName:     "n8n-server",
  staticIpName:     "n8n-server-ip",
  availabilityZone: "ap-northeast-2a",
  n8nUser:          "admin",
  n8nPassword:      "N8N_PASSWORD",
  dbPassword:       "DB_PASSWORD",
};

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

const dockerComposeYml = `services:
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
  n8n_data:`;

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
cd /opt/n8n && docker compose up -d`;

async function main() {
  // 1. 인스턴스 확인/생성
  console.log("\n[1/4] 인스턴스 확인...");
  let instance = null;
  try {
    instance = (await client.send(new GetInstanceCommand({ instanceName: CONFIG.instanceName }))).instance;
    console.log(`  이미 존재 (state: ${instance.state?.name})`);
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
  for (const port of [22, 80, 443, 5678, 3100]) {
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

  // 3. 고정 IP
  console.log("\n[3/4] 고정 IP...");
  let staticIp = null;
  try {
    staticIp = (await client.send(new GetStaticIpCommand({ staticIpName: CONFIG.staticIpName }))).staticIp;
    console.log(`  이미 존재: ${staticIp.ipAddress}`);
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
    console.log(`  연결됨`);
  }

  // 4. 완료
  console.log("\n" + "=".repeat(55));
  console.log("✅ 설치 완료!");
  console.log(`고정 IP  : ${staticIp.ipAddress}`);
  console.log(`n8n URL  : http://${staticIp.ipAddress}:5678`);
  console.log(`로그인   : ${CONFIG.n8nUser} / ${CONFIG.n8nPassword}`);
  console.log("=".repeat(55));
  console.log("⚠ 서버 초기화까지 약 5분 소요 (Docker 설치 + 컨테이너 시작)");
  console.log(`\n다음 단계: 도메인 DNS A레코드를 ${staticIp.ipAddress} 로 설정 후`);
  console.log("02_ssl-nginx-setup.md 를 Claude에게 전달하세요.");
}

main().catch((e) => { console.error("ERROR:", e.message); process.exit(1); });
```

---

## 완료 후 확인

- `http://[고정IP]:5678` 접속 → n8n 로그인 화면 확인
- 이후 SSL 설정: `02_ssl-nginx-setup.md` 참고

---

## 주의사항

- 인스턴스 생성 후 **5분 대기** 필요 (cloud-init 실행 시간)
- n8n 기본 비밀번호 반드시 변경할 것
- AWS Access Key는 작업 후 환경변수로 이동 권장

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| n8n 접속 안 됨 | cloud-init 아직 실행 중 | 5분 대기 후 재시도 |
| Secure Cookie 경고 | HTTP 환경 | `N8N_SECURE_COOKIE=false` 추가 (SSL 전 임시) |
| AWS CLI 설치 실패 | Windows 권한 문제(1603) | Node.js SDK 방식으로 우회 (이 가이드 방식) |
