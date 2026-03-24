# [실행 가이드 06] 콘텐츠 승인 대시보드 설치

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~05` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01~05번 작업 완료 (n8n + 브릿지 서버 실행 중)
- [ ] 가비아 DNS에 A레코드 추가: `dash` → `서버 고정IP`
- [ ] DNS 전파 확인

---

## Claude에게 전달할 정보

```
서버 고정 IP      :
SSH 키 경로       :        (예: C:\Users\...\lightsail_key.pem)
대시보드 도메인   :        (예: dash.example.com)
관리자 이메일     :
n8n 승인 웹훅 URL :        (예: https://n8n.example.com/webhook/content-approve)
```

---

## Claude 실행 내용 (참고용)

### 1단계: 앱 파일 작성

`/opt/dashboard/package.json`:

```json
{
  "name": "content-dashboard",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": { "start": "node app.js" },
  "dependencies": {
    "better-sqlite3": "^9.4.3",
    "ejs": "^3.1.10",
    "express": "^4.18.2"
  }
}
```

`/opt/dashboard/app.js`:

```js
const express = require('express');
const Database = require('better-sqlite3');
const path = require('path');
const fs = require('fs');

const app = express();
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

const dataDir = path.join(__dirname, 'data');
if (!fs.existsSync(dataDir)) fs.mkdirSync(dataDir);

const db = new Database(path.join(dataDir, 'dashboard.db'));
db.exec(`
  CREATE TABLE IF NOT EXISTS contents (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    title       TEXT    NOT NULL,
    body        TEXT    DEFAULT '',
    hashtags    TEXT    DEFAULT '',
    product     TEXT    DEFAULT '',
    requester   TEXT    DEFAULT '',
    platform    TEXT    DEFAULT '인스타그램',
    status      TEXT    DEFAULT 'pending',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);

function counts() {
  return {
    pending:  db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='pending'").get().c,
    approved: db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='approved'").get().c,
    rejected: db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='rejected'").get().c,
  };
}

function formatDt(dt) {
  if (!dt) return '';
  return new Date(dt).toLocaleString('ko-KR', { timeZone: 'Asia/Seoul' });
}
app.locals.formatDt = formatDt;

// 목록
app.get('/', (req, res) => {
  const status = ['pending','approved','rejected'].includes(req.query.status)
    ? req.query.status : 'pending';
  const contents = db.prepare(
    'SELECT * FROM contents WHERE status = ? ORDER BY created_at DESC'
  ).all(status);
  res.render('index', { contents, status, counts: counts() });
});

// 상세
app.get('/contents/:id', (req, res) => {
  const content = db.prepare('SELECT * FROM contents WHERE id = ?').get(req.params.id);
  if (!content) return res.status(404).send('콘텐츠를 찾을 수 없습니다.');
  res.render('detail', { content, counts: counts() });
});

// 본문 수정
app.post('/contents/:id/save', (req, res) => {
  const { title, body, hashtags } = req.body;
  db.prepare(`
    UPDATE contents SET title=?, body=?, hashtags=?, updated_at=CURRENT_TIMESTAMP WHERE id=?
  `).run(title, body, hashtags, req.params.id);
  res.redirect(`/contents/${req.params.id}?saved=1`);
});

// 승인 → n8n 웹훅 호출
app.post('/contents/:id/approve', async (req, res) => {
  const content = db.prepare('SELECT * FROM contents WHERE id = ?').get(req.params.id);
  if (!content) return res.status(404).send('Not found');
  try {
    const webhookUrl = process.env.N8N_APPROVE_WEBHOOK;
    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        id: content.id, title: content.title, body: content.body,
        hashtags: content.hashtags, product: content.product,
        requester: content.requester, platform: content.platform,
      }),
    });
    console.log(`[approve] id=${content.id} → webhook OK`);
  } catch (err) {
    console.error(`[approve] webhook error:`, err.message);
  }
  db.prepare(`UPDATE contents SET status='approved', updated_at=CURRENT_TIMESTAMP WHERE id=?`)
    .run(req.params.id);
  res.redirect('/?status=approved');
});

// 거절
app.post('/contents/:id/reject', (req, res) => {
  db.prepare(`UPDATE contents SET status='rejected', updated_at=CURRENT_TIMESTAMP WHERE id=?`)
    .run(req.params.id);
  res.redirect('/?status=rejected');
});

// API: n8n에서 콘텐츠 저장
app.post('/api/contents', (req, res) => {
  const { title, body, hashtags, product, requester, platform } = req.body;
  const hashtagStr = Array.isArray(hashtags) ? hashtags.join(' ') : (hashtags || '');
  const result = db.prepare(`
    INSERT INTO contents (title, body, hashtags, product, requester, platform)
    VALUES (?, ?, ?, ?, ?, ?)
  `).run(title || product || '제목 없음', body || '', hashtagStr,
         product || '', requester || '', platform || '인스타그램');
  console.log(`[api] saved id=${result.lastInsertRowid} product="${product}"`);
  res.json({ success: true, id: result.lastInsertRowid });
});

// API: 목록 조회 (JSON)
app.get('/api/contents', (req, res) => {
  const status = req.query.status || 'pending';
  res.json({ success: true, data: db.prepare(
    'SELECT * FROM contents WHERE status=? ORDER BY created_at DESC'
  ).all(status) });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, '0.0.0.0', () => console.log(`[Dashboard] Running on port ${PORT}`));
```

`/opt/dashboard/ecosystem.config.js` (PM2):

```js
module.exports = {
  apps: [{
    name: 'dashboard',
    script: '/opt/dashboard/app.js',
    cwd: '/opt/dashboard',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '256M',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
      N8N_APPROVE_WEBHOOK: 'https://[N8N_DOMAIN]/webhook/content-approve',
    },
    error_file: '/opt/dashboard/logs/error.log',
    out_file:   '/opt/dashboard/logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
  }],
};
```

---

### 2단계: 서버 설치 + PM2 시작

```bash
# 디렉토리 생성
sudo mkdir -p /opt/dashboard/logs /opt/dashboard/data
sudo chown -R ubuntu:ubuntu /opt/dashboard

# 파일 복사 후
cd /opt/dashboard && npm install --omit=dev

# PM2 시작
pm2 start /opt/dashboard/ecosystem.config.js
pm2 save

# 헬스체크
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/
# 200 응답 확인
```

---

### 3단계: Nginx + SSL

> ⚠ Nginx 설정은 로컬 파일 생성 후 SCP 전송 방식 사용

`dash_nginx.conf` 로컬 생성:

```nginx
server {
    listen 80;
    server_name [DASH_DOMAIN];
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name [DASH_DOMAIN];

    ssl_certificate     /etc/letsencrypt/live/[DASH_DOMAIN]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[DASH_DOMAIN]/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 10m;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

```bash
# SSL 인증서 발급
sudo certbot --nginx -d [DASH_DOMAIN] \
  --non-interactive --agree-tos -m [ADMIN_EMAIL]

# Nginx 설정 적용 (SCP 전송 후)
sudo cp /tmp/dash_nginx.conf /etc/nginx/sites-available/dash
sudo ln -sf /etc/nginx/sites-available/dash /etc/nginx/sites-enabled/dash
sudo nginx -t && sudo systemctl reload nginx

# HTTPS 헬스체크
curl -s -o /dev/null -w "%{http_code}" https://[DASH_DOMAIN]/
# 200 응답 확인
```

---

### 4단계: Lightsail 방화벽 포트 오픈

```js
// AWS SDK로 포트 3000 오픈
import { LightsailClient, OpenInstancePublicPortsCommand } from "@aws-sdk/client-lightsail";
const client = new LightsailClient({ region: "ap-northeast-2", credentials: { ... } });
await client.send(new OpenInstancePublicPortsCommand({
  instanceName: "n8n-server",
  portInfo: { fromPort: 3000, toPort: 3000, protocol: "tcp" },
}));
```

---

## API 명세

### POST /api/contents — 콘텐츠 저장 (n8n → 대시보드)

```
POST https://[DASH_DOMAIN]/api/contents
Content-Type: application/json

{
  "product":   "상품명",
  "title":     "메인 카피",
  "body":      "본문 텍스트",
  "hashtags":  "#태그1 #태그2 ...",   // 문자열 또는 배열 모두 허용
  "requester": "요청자 아이디",
  "platform":  "인스타그램"
}

응답: { "success": true, "id": 1 }
```

### GET /api/contents?status=pending — 목록 조회

```
status: pending | approved | rejected
```

---

## n8n 워크플로 연동 (04번 문서 참고)

콘텐츠 생성 워크플로의 `Claude브릿지호출` 노드 이후에 병렬로 추가:

```
Claude브릿지호출
 ├→ 포맷팅 → Mattermost채널전송
 └→ 대시보드페이로드 (Code) → 대시보드저장 (HTTP POST /api/contents)
```

**대시보드페이로드 Code 노드** — bridge 응답에서 API 필드 조립:

```js
const bridge  = $input.first().json;
const content = bridge.content || {};
const parse   = $('상품명파싱').first().json;

return [{
  json: {
    product:   bridge.product   || parse.productName || '',
    title:     content.mainCopy || '',
    body:      content.body     || '',
    hashtags:  Array.isArray(content.hashtags) ? content.hashtags.join(' ') : '',
    requester: parse.userId     || '',
    platform:  bridge.platform  || '인스타그램',
  }
}];
```

**대시보드저장 HTTP Request 노드**:
- Method: POST
- URL: `https://[DASH_DOMAIN]/api/contents`
- Body: `specifyBody: keypair` (각 필드를 `={{ $json.필드명 }}` 으로 지정)

> ⚠ `specifyBody: 'json'` + `jsonBody` 방식은 n8n v2에서 `={{ }}` 표현식이 평가되지 않음.
> 반드시 `specifyBody: 'keypair'` 또는 Code 노드로 HTTP 요청 직접 처리.

---

## 완료 후 확인

```
https://[DASH_DOMAIN]/            → 대시보드 목록 (승인대기/완료/거절 탭)
https://[DASH_DOMAIN]/api/contents?status=pending → API JSON 응답
```

1. Mattermost에서 `/콘텐츠생성 [상품명]` 입력
2. n8n 워크플로 실행 → Mattermost 채널에 결과 게시
3. 대시보드에 `pending` 상태로 자동 저장
4. 대시보드에서 내용 확인/수정 후 **승인** → n8n `/webhook/content-approve` 트리거

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| certbot `Could not install certificate` | nginx server_name 블록 미존재 상태에서 발급 | 인증서는 정상 발급됨 → SCP로 nginx 설정 적용 후 reload |
| `jsonBody` 내 `={{ }}` 리터럴로 저장됨 | n8n v2 httpRequest `specifyBody: json` 버그 | `specifyBody: keypair` 방식으로 변경 |
| PM2 재시작 후 웹훅 URL 환경변수 없음 | ecosystem.config.js env 미반영 | `pm2 delete dashboard && pm2 start ecosystem.config.js` 재실행 |

---

## 서버 관리 명령어

```bash
pm2 status                           # 전체 프로세스 상태
pm2 logs dashboard --lines 30        # 대시보드 로그
pm2 restart dashboard                # 재시작
node -e "const D=require('better-sqlite3'); const db=new D('/opt/dashboard/data/dashboard.db'); console.log(db.prepare('SELECT id,title,status FROM contents ORDER BY id DESC LIMIT 10').all())" # DB 직접 조회 (반드시 /opt/dashboard에서 실행)
```
