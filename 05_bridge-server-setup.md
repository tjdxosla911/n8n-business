# [실행 가이드 05] Claude 브릿지 서버 설치 + SSL + n8n 워크플로 연동

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~04` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01~04번 작업 완료
- [ ] Anthropic API Key 발급 (`sk-ant-...`)
- [ ] 가비아 DNS에 A레코드 추가: `api` → `서버 고정IP`

---

## Claude에게 전달할 정보

```
서버 고정 IP         :
SSH 키 경로          :        (예: C:\Users\...\lightsail_key.pem)
Anthropic API Key    :        (sk-ant-...)
브릿지 도메인        :        (예: api.example.com)
n8n URL             :        (예: https://n8n.example.com)
n8n 이메일          :
n8n 비밀번호         :
n8n Basic Auth PW   :        (docker-compose의 N8N_BASIC_AUTH_PASSWORD)
n8n 워크플로명       : Mattermost 콘텐츠생성
```

---

## Claude 실행 내용 (참고용)

### 1단계: 브릿지 서버 파일 작성

`/opt/bridge/server.js`:

```js
const express = require('express');
const Anthropic = require('@anthropic-ai/sdk');

const app = express();
app.use(express.json());

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

app.post('/generate/sns', async (req, res) => {
  const { product, platform = '인스타그램', tone = '친근한', options = {} } = req.body;
  if (!product) return res.status(400).json({ error: '상품명(product)이 필요합니다.' });

  const prompt = `당신은 SNS 마케팅 전문가입니다.
아래 상품에 대한 ${platform} 홍보 게시물을 작성해주세요.

상품명: ${product}
톤앤매너: ${tone}
${options.keywords ? `키워드: ${options.keywords.join(', ')}` : ''}
${options.target ? `타겟: ${options.target}` : ''}

다음 형식으로 작성해주세요:
1. 메인 카피 (한 줄, 임팩트 있게)
2. 본문 (3~5줄, 상품 특징과 혜택 중심)
3. 해시태그 (10개, #으로 시작)
4. CTA (행동 유도 문구)

JSON 형식으로 반환:
{
  "mainCopy": "...",
  "body": "...",
  "hashtags": ["#...", ...],
  "cta": "..."
}`;

  try {
    const message = await client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }],
    });
    const text = message.content[0].text;
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      return res.json({ success: true, product, platform, content: JSON.parse(jsonMatch[0]), raw: text });
    }
    return res.json({ success: true, product, platform, content: null, raw: text });
  } catch (err) {
    console.error('[ERROR]', err.message);
    return res.status(500).json({ error: err.message });
  }
});

const PORT = process.env.PORT || 3100;
app.listen(PORT, '0.0.0.0', () => console.log(`[Bridge Server] Running on port ${PORT}`));
```

`/opt/bridge/package.json`:

```json
{
  "name": "claude-bridge-server",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2",
    "@anthropic-ai/sdk": "^0.39.0"
  }
}
```

`/opt/bridge/ecosystem.config.js` (PM2):

```js
module.exports = {
  apps: [{
    name: 'bridge-server',
    script: '/opt/bridge/server.js',
    cwd: '/opt/bridge',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '512M',
    env: {
      NODE_ENV: 'production',
      PORT: 3100,
      ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
    },
    error_file: '/opt/bridge/logs/error.log',
    out_file: '/opt/bridge/logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
  }],
};
```

---

### 2단계: 서버에 설치

```bash
# 디렉토리 생성 및 파일 복사
sudo mkdir -p /opt/bridge/logs
sudo chown -R ubuntu:ubuntu /opt/bridge
cd /opt/bridge && npm install --omit=dev

# .env 생성
echo "ANTHROPIC_API_KEY=sk-ant-..." > /opt/bridge/.env
echo "PORT=3100" >> /opt/bridge/.env

# PM2 설치 및 시작
sudo npm install -g pm2
ANTHROPIC_API_KEY=sk-ant-... pm2 start /opt/bridge/ecosystem.config.js
pm2 save

# 시스템 재시작 시 자동 실행 등록
sudo env PATH=$PATH:/usr/bin:/usr/local/bin \
  /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
sudo systemctl enable pm2-ubuntu
```

---

### 3단계: Nginx + SSL

> ⚠ Nginx 설정 파일은 로컬 생성 후 SCP 전송 방식 사용

`api_nginx.conf` 로컬 생성:

```nginx
server {
    listen 80;
    server_name [API_DOMAIN];
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name [API_DOMAIN];

    ssl_certificate     /etc/letsencrypt/live/[API_DOMAIN]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[API_DOMAIN]/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 10m;

    location / {
        proxy_pass         http://localhost:3100;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

```bash
# 임시 HTTP 설정으로 SSL 발급 (certonly 말고 --nginx 사용)
sudo certbot --nginx -d [API_DOMAIN] \
  --non-interactive --agree-tos -m [ADMIN_EMAIL] --redirect

# 리버스 프록시 설정 적용
scp api_nginx.conf ubuntu@[서버IP]:/tmp/api_nginx.conf
sudo cp /tmp/api_nginx.conf /etc/nginx/sites-available/api
sudo nginx -t && sudo systemctl reload nginx
```

---

### 4단계: n8n 워크플로 브릿지 URL 수정

Node.js 스크립트로 실행:

```js
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';

const N8N_BASE   = 'N8N_URL';
const BASIC_AUTH = 'Basic ' + Buffer.from('admin:N8N_BASIC_AUTH_PW').toString('base64');
const NEW_BRIDGE = 'https://[API_DOMAIN]/generate/sns';

let sessionCookie = '';
async function req(method, path, body) {
  const res = await fetch(`${N8N_BASE}${path}`, {
    method,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': BASIC_AUTH,
      ...(sessionCookie ? { Cookie: sessionCookie } : {}),
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  const setCookie = res.headers.getSetCookie?.() || [];
  if (setCookie.length) sessionCookie = setCookie.map(c => c.split(';')[0]).join('; ');
  return { status: res.status, data: await res.json() };
}

await req('POST', '/rest/login', { emailOrLdapLoginId: 'N8N_EMAIL', password: 'N8N_PASSWORD' });

const wfRes = await req('GET', '/rest/workflows');
const wf = (wfRes.data?.data || []).find(w => w.name === 'Mattermost 콘텐츠생성');
const detail = await req('GET', `/rest/workflows/${wf.id}`);
const workflow = detail.data?.data || detail.data;

for (const node of workflow.nodes) {
  if (node.name === 'Claude브릿지호출') node.parameters.url = NEW_BRIDGE;
}

await req('PATCH', `/rest/workflows/${wf.id}`, workflow);

const wfDetail2 = await req('GET', `/rest/workflows/${wf.id}`);
const versionId = wfDetail2.data?.data?.versionId || wfDetail2.data?.versionId;
await req('PATCH', `/rest/workflows/${wf.id}`, { ...workflow, id: wf.id, versionId, active: true });

console.log(`✓ 브릿지 URL 업데이트 완료: ${NEW_BRIDGE}`);
```

---

## 완료 후 확인

```bash
# 헬스체크
curl https://[API_DOMAIN]/health

# 콘텐츠 생성 테스트
curl -X POST https://[API_DOMAIN]/generate/sns \
  -H "Content-Type: application/json" \
  -d '{"product":"무선 이어폰","platform":"인스타그램","tone":"친근한"}'

# PM2 상태
pm2 status
pm2 logs bridge-server --lines 20
```

---

## 전체 파이프라인

```
Mattermost /콘텐츠생성 [상품명]
 → n8n Webhook
 → https://[API_DOMAIN]/generate/sns  ← 브릿지 서버 (Claude API)
 → Mattermost #콘텐츠생성 채널에 결과 게시
```

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| `certbot certonly` 실패 "No such authorization" | 이전 실패한 인증 캐시 | `certbot --nginx` (certonly 없이) 사용 |
| Claude CLI auth hanging | Claude Code CLI는 인터랙티브 OAuth 방식 | Anthropic SDK + API Key 방식으로 대체 |
| PM2 재시작 후 API Key 없음 | ecosystem.config.js에서 env 변수 미전달 | 시작 시 `ANTHROPIC_API_KEY=... pm2 start` 명령으로 주입 |

---

## 서버 관리 명령어

```bash
pm2 status                        # 상태 확인
pm2 restart bridge-server         # 재시작
pm2 logs bridge-server --lines 50 # 로그 확인
pm2 stop bridge-server            # 중지
```
