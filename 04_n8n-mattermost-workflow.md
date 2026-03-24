# [실행 가이드 04] n8n Mattermost Credential + 워크플로 자동 등록

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~03` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01번: Lightsail + n8n 설치 완료
- [ ] 02번: n8n SSL 완료 (https://n8n.도메인)
- [ ] 03번: Mattermost 설치 + 봇 토큰 발급 완료

---

## Claude에게 전달할 정보

```
n8n URL          :        (예: https://n8n.example.com)
n8n 이메일       :
n8n 비밀번호     :
n8n Basic Auth   : admin / [docker-compose의 N8N_BASIC_AUTH_PASSWORD]

Mattermost URL   :        (예: https://chat.example.com)
Bot Token        :        (03번 작업 완료 시 발급된 값)
채널 ID (#콘텐츠생성) :   (03번 작업 완료 시 출력된 값)

브릿지 서버 URL  :        (예: http://[서버IP]:3100)
```

---

## Claude 실행 내용 (참고용)

Node.js 스크립트로 실행 (`node n8n-setup.mjs`):

```js
// n8n Mattermost Credential + 워크플로 자동 설정
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'; // Self-signed cert 대응

const N8N_BASE     = 'N8N_URL';
const N8N_EMAIL    = 'N8N_EMAIL';
const N8N_PASSWORD = 'N8N_PASSWORD';
const BASIC_AUTH   = 'Basic ' + Buffer.from('admin:N8N_BASIC_AUTH_PW').toString('base64');

const MM_SERVER  = 'MATTERMOST_URL';
const MM_TOKEN   = 'BOT_TOKEN';
const MM_CH_ID   = 'CONTENT_CHANNEL_ID';
const BRIDGE_URL = 'BRIDGE_SERVER_URL';

// ── 공통 요청 함수 ──────────────────────────────────────────────
let sessionCookie = '';

async function req(method, path, body, extraHeaders = {}) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': BASIC_AUTH,
    ...(sessionCookie ? { Cookie: sessionCookie } : {}),
    ...extraHeaders,
  };
  const res = await fetch(`${N8N_BASE}${path}`, {
    method,
    headers,
    body: body ? JSON.stringify(body) : undefined,
  });
  const setCookie = res.headers.getSetCookie?.() || [];
  if (setCookie.length) sessionCookie = setCookie.map(c => c.split(';')[0]).join('; ');
  const text = await res.text();
  try { return { status: res.status, data: JSON.parse(text) }; }
  catch { return { status: res.status, data: text }; }
}

// ── 1. 로그인 ────────────────────────────────────────────────────
// ⚠ n8n 로그인 필드명: emailOrLdapLoginId (email 아님)
const login = await req('POST', '/rest/login', {
  emailOrLdapLoginId: N8N_EMAIL,
  password: N8N_PASSWORD,
});
if (login.status !== 200) { console.error('로그인 실패:', login.data); process.exit(1); }
console.log(`✓ 로그인 성공`);

// ── 2. Mattermost Credential 생성 ────────────────────────────────
const credsRes = await req('GET', '/rest/credentials?includeData=false');
const existing = (credsRes.data?.data || []).find(c => c.name === 'Mattermost Bot');
let credId;

if (existing) {
  credId = existing.id;
  console.log(`✓ Credential 이미 존재 (id: ${credId})`);
} else {
  const cred = await req('POST', '/rest/credentials', {
    name: 'Mattermost Bot',
    type: 'mattermostApi',
    data: { baseUrl: MM_SERVER, accessToken: MM_TOKEN },
  });
  credId = cred.data.data?.id || cred.data.id;
  console.log(`✓ Credential 생성 완료 (id: ${credId})`);
}

// ── 3. 워크플로 JSON ────────────────────────────────────────────
const workflow = {
  name: 'Mattermost 콘텐츠생성',
  active: false,
  nodes: [
    {
      id: 'node-webhook',
      name: 'Webhook',
      type: 'n8n-nodes-base.webhook',
      typeVersion: 2,
      position: [200, 300],
      webhookId: 'content-generate',
      parameters: {
        httpMethod: 'POST',
        path: 'content-generate',
        responseMode: 'responseNode',
        options: {},
      },
    },
    {
      id: 'node-respond',
      name: '즉시응답',
      type: 'n8n-nodes-base.respondToWebhook',
      typeVersion: 1,
      position: [420, 160],
      parameters: {
        respondWith: 'text',
        responseBody: '⏳ 콘텐츠 생성 중... 잠시 후 채널에 결과가 올라옵니다.',
        options: {},
      },
    },
    {
      id: 'node-parse',
      name: '상품명파싱',
      type: 'n8n-nodes-base.code',
      typeVersion: 2,
      position: [420, 420],
      parameters: {
        jsCode: `
const body = $input.first().json.body || {};
const text = (body.text || '').trim();
const userId = body.user_name || 'unknown';
const randomProducts = ['스마트폰 케이스', '무선 이어폰', '보조배터리', '노트북 거치대', '블루투스 스피커'];
const productName = text || randomProducts[Math.floor(Math.random() * randomProducts.length)];
return [{ json: { productName, userId, originalText: text } }];
`.trim(),
      },
    },
    {
      id: 'node-bridge',
      name: 'Claude브릿지호출',
      type: 'n8n-nodes-base.httpRequest',
      typeVersion: 4,
      position: [640, 420],
      parameters: {
        method: 'POST',
        url: `${BRIDGE_URL}/generate/sns`,  // ⚠ 반드시 /generate/sns (브릿지 서버 엔드포인트)
        sendBody: true,
        contentType: 'json',
        specifyBody: 'json',                // ⚠ body.rawParameters 방식은 동작 안 함
        jsonBody: '{"product": "={{ $json.productName }}"}',
        options: { timeout: 60000 },
      },
    },
    {
      // 브릿지 응답 구조: { success, product, platform, content: { mainCopy, body, hashtags[], cta }, raw }
      // content 전체를 그대로 message에 넣으면 [object Object]가 됨 → 각 필드를 꺼내서 조립
      id: 'node-format',
      name: '포맷팅',
      type: 'n8n-nodes-base.code',
      typeVersion: 2,
      position: [860, 420],
      parameters: {
        jsCode: `
const bridge = $input.first().json;
const content = bridge.content || {};
const parse = $('상품명파싱').first().json;

const mainCopy = content.mainCopy || '';
const body = content.body || '';
const hashtags = Array.isArray(content.hashtags) ? content.hashtags.join(' ') : '';
const cta = content.cta || '';

const message = [
  '📝 콘텐츠 생성 완료',
  '',
  '요청자: ' + parse.userId,
  '상품명: ' + parse.productName,
  '',
  '---',
  mainCopy,
  '',
  body,
  '',
  cta,
  '---',
  '',
  hashtags,
].join('\\n');

return [{ json: { message } }];
`.trim(),
      },
    },
    {
      id: 'node-mm',
      name: 'Mattermost채널전송',
      type: 'n8n-nodes-base.mattermost',
      typeVersion: 1,
      position: [1080, 420],
      credentials: { mattermostApi: { id: credId, name: 'Mattermost Bot' } },
      parameters: {
        resource: 'message',
        operation: 'post',
        channelId: MM_CH_ID,
        message: '={{ $json.message }}',   // 포맷팅 노드에서 조립된 문자열 사용
      },
    },
    {
      // 브릿지 응답 구조에서 대시보드 API용 페이로드 준비
      // ⚠ jsonBody 내 ={{ }} 표현식은 n8n v2에서 평가되지 않음 → Code 노드로 먼저 조립
      id: 'node-dash-payload',
      name: '대시보드페이로드',
      type: 'n8n-nodes-base.code',
      typeVersion: 2,
      position: [860, 620],
      parameters: {
        jsCode: `
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
`.trim(),
      },
    },
    {
      // ⚠ specifyBody: 'keypair' 방식 사용 (jsonBody 방식은 표현식 평가 안 됨)
      id: 'node-dash-save',
      name: '대시보드저장',
      type: 'n8n-nodes-base.httpRequest',
      typeVersion: 4,
      position: [1080, 620],
      parameters: {
        method: 'POST',
        url: 'DASHBOARD_URL/api/contents',  // 예: https://dash.example.com/api/contents
        sendBody: true,
        contentType: 'json',
        specifyBody: 'keypair',
        bodyParameters: {
          parameters: [
            { name: 'product',   value: '={{ $json.product }}'   },
            { name: 'title',     value: '={{ $json.title }}'     },
            { name: 'body',      value: '={{ $json.body }}'      },
            { name: 'hashtags',  value: '={{ $json.hashtags }}'  },
            { name: 'requester', value: '={{ $json.requester }}' },
            { name: 'platform',  value: '={{ $json.platform }}'  },
          ],
        },
        options: { timeout: 10000 },
      },
    },
  ],
  connections: {
    Webhook: {
      main: [[
        { node: '즉시응답',   type: 'main', index: 0 },
        { node: '상품명파싱', type: 'main', index: 0 },
      ]],
    },
    상품명파싱:        { main: [[{ node: 'Claude브릿지호출',  type: 'main', index: 0 }]] },
    Claude브릿지호출:  { main: [[
      { node: '포맷팅',          type: 'main', index: 0 },  // Mattermost 전송용
      { node: '대시보드페이로드', type: 'main', index: 0 },  // 대시보드 저장용 (병렬)
    ]] },
    포맷팅:            { main: [[{ node: 'Mattermost채널전송', type: 'main', index: 0 }]] },
    대시보드페이로드:  { main: [[{ node: '대시보드저장',       type: 'main', index: 0 }]] },
  },
  settings: { executionOrder: 'v1', saveManualExecutions: true },
};

// 기존 워크플로 확인 후 생성/업데이트
const wfRes = await req('GET', '/rest/workflows');
const existingWf = (wfRes.data?.data || []).find(w => w.name === workflow.name);
let wfId;

if (existingWf) {
  const upd = await req('PATCH', `/rest/workflows/${existingWf.id}`, workflow);
  wfId = existingWf.id;
  console.log(`✓ 워크플로 업데이트 완료 (id: ${wfId})`);
} else {
  const created = await req('POST', '/rest/workflows', workflow);
  wfId = created.data.data?.id || created.data.id;
  console.log(`✓ 워크플로 생성 완료 (id: ${wfId})`);
}

// ── 4. 워크플로 활성화 ───────────────────────────────────────────
// ⚠ 활성화 시 versionId 필수 (PATCH /rest/workflows/{id} with active:true)
// ⚠ n8n v2에서 PATCH active:true가 200을 반환해도 실제로 inactive 상태일 수 있음
//    → DB의 activeVersionId=null이 원인. 아래 단계 실행 후 n8n 재시작 필요할 수 있음
const wfDetail = await req('GET', `/rest/workflows/${wfId}`);
const versionId = wfDetail.data?.data?.versionId || wfDetail.data?.versionId;
const activate = await req('PATCH', `/rest/workflows/${wfId}`, {
  ...workflow, id: wfId, versionId, active: true,
});
console.log(`✓ 활성화 완료 (status: ${activate.status})`);

// 활성화 상태 재확인
await new Promise(r => setTimeout(r, 1000));
const checkRes = await req('GET', `/rest/workflows/${wfId}`);
const isActive = checkRes.data?.data?.active ?? checkRes.data?.active;
if (!isActive) {
  console.warn('⚠ API 활성화 실패 — DB 직접 수정 필요 (아래 "n8n v2 활성화 버그" 참고)');
} else {
  console.log('✓ active 상태 확인 완료');
}

console.log('\n=== 완료 ===');
console.log(`Webhook URL : ${N8N_BASE}/webhook/content-generate`);
console.log(`워크플로    : ${N8N_BASE}/workflow/${wfId}`);
```

---

## 워크플로 구조

```
Mattermost /콘텐츠생성
 └→ Webhook (POST /webhook/content-generate)
     ├→ 즉시응답: "⏳ 콘텐츠 생성 중..."
     └→ 상품명파싱 (Code)
          └→ Claude브릿지호출 (HTTP POST → /generate/sns)
               ├→ 포맷팅 (Code)
               │    └→ Mattermost채널전송 (#콘텐츠생성)
               └→ 대시보드페이로드 (Code)        ← 병렬 실행
                    └→ 대시보드저장 (HTTP POST → /api/contents)
```

### 브릿지 응답 구조

```json
{
  "success": true,
  "product": "상품명",
  "platform": "인스타그램",
  "content": {
    "mainCopy": "메인 카피 한 줄",
    "body": "본문 3~5줄",
    "hashtags": ["#태그1", "#태그2", ...],
    "cta": "행동 유도 문구"
  },
  "raw": "Claude 원본 응답 텍스트"
}
```

> `$json.content`를 그대로 message에 넣으면 `[object Object]`가 됩니다.
> 반드시 포맷팅 노드에서 각 필드를 꺼내 문자열로 조립해야 합니다.

---

## 완료 후 확인

- n8n 워크플로 화면에서 Active 상태 확인
- Mattermost에서 `/콘텐츠생성 [상품명]` 입력 → 즉시 "처리 중" 응답 → 채널에 결과 게시

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| `fetch failed: SELF_SIGNED_CERT` | Node.js가 Let's Encrypt 중간 CA 미신뢰 | 스크립트 최상단에 `process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'` 추가 |
| 로그인 400 `emailOrLdapLoginId Required` | n8n 로그인 필드명이 `email`이 아님 | `emailOrLdapLoginId` 키 사용 |
| 활성화 400 `versionId Required` | PATCH 활성화 시 versionId 필수 | GET으로 워크플로 조회 후 versionId 추출해서 포함 |
| Claude브릿지호출 노드 "Bad request - please check your parameters" | `body.rawParameters` 형식은 n8n httpRequest v4에서 동작 안 함 → body 없이 요청 전송 → 브릿지 서버 400 | `specifyBody: 'json'` + `jsonBody` 형식으로 변경 |
| Mattermost 채널에 `[object Object]`로 게시됨 | 브릿지 응답의 `content` 필드가 객체인데 문자열로 직접 변환됨 | 포맷팅 Code 노드에서 `content.mainCopy`, `content.body`, `content.hashtags.join(' ')` 등 개별 필드를 꺼내 조립 |
| `jsonBody` 내 `={{ }}` 표현식이 리터럴 문자열로 저장됨 | n8n v2 `specifyBody: 'json'` + `jsonBody`에서 `={{ }}` 표현식이 평가되지 않음 | Code 노드로 먼저 데이터를 조립한 뒤 `specifyBody: 'keypair'` 방식으로 HTTP 요청 |
| 브릿지 호출 실패 (연결 자체가 안 됨) | 브릿지 서버 미실행 또는 URL 오탈자 | `:3100` 서버 실행 상태 확인, URL 끝에 `/generate/sns` 포함 여부 확인 |
| PATCH active:true → 200 반환하나 실제로 inactive | n8n v2 버그: DB `activeVersionId=null`이면 웹훅 미등록 | 아래 "n8n v2 활성화 버그 해결" 섹션 참고 |
| 워크플로 수정(PATCH) 후 웹훅이 다시 미등록됨 | PATCH로 노드 수정 시 `versionId`가 바뀌지만 `activeVersionId`는 구버전 유지 | DB에서 `activeVersionId = 현재 versionId`로 업데이트 후 n8n 재시작 |
| POST `/activate`, POST `/publish` → 404 | n8n v2에서 해당 엔드포인트 미존재 | DB 직접 수정 방식으로 대체 |
| Mattermost `/콘텐츠생성` → 404 Not Found | 워크플로 inactive로 웹훅 미등록 상태 | 워크플로 active 상태 확인 → 아래 해결책 적용 |

---

## n8n v2 활성화 버그 해결

> **발생 패턴 2가지**
> 1. 최초 활성화 시: `activeVersionId=null` → 웹훅 미등록
> 2. 워크플로 수정(PATCH) 후: `activeVersionId`가 구버전을 가리킴 → 웹훅 미등록
>
> 두 경우 모두 DB에서 `activeVersionId = versionId`로 맞춰주고 n8n 재시작하면 해결됩니다.

### 증상 확인

```bash
# versions_match가 f(false)이거나 activeVersionId가 NULL이면 조치 필요
ssh -i [SSH키] ubuntu@[서버IP] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"SELECT name, active,
      \\\"activeVersionId\\\",
      \\\"versionId\\\",
      (\\\"activeVersionId\\\" = \\\"versionId\\\") AS versions_match
    FROM workflow_entity WHERE name = 'Mattermost 콘텐츠생성';\"
"
```

### 해결 방법 (스크립트로 서버에 전송 후 실행)

```bash
cat > /tmp/fix-n8n-active.sh << 'EOF'
#!/bin/bash
WF_NAME="Mattermost 콘텐츠생성"

VERSION_ID=$(docker exec n8n-postgres psql -U n8n -d n8n -t -c \
  "SELECT \"versionId\" FROM workflow_entity WHERE name = '$WF_NAME';" | tr -d ' \n')
echo "현재 versionId: $VERSION_ID"

docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET active=true, \"activeVersionId\"='$VERSION_ID' WHERE name = '$WF_NAME';"

cd /opt/n8n && docker compose restart n8n
sleep 12

docker compose logs n8n --tail=5 2>&1 | grep -E "Activated|Error"
EOF

scp -i [SSH키] /tmp/fix-n8n-active.sh ubuntu@[서버IP]:/tmp/fix-n8n-active.sh
ssh -i [SSH키] ubuntu@[서버IP] "bash /tmp/fix-n8n-active.sh"
# "Activated workflow" 로그 확인
```

### 재시작 후 동작 확인

```bash
curl -s -X POST https://[N8N_URL]/webhook/content-generate \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "text=테스트상품&user_name=testuser"
# "⏳ 콘텐츠 생성 중..." 응답이 오면 정상
```

> **주의**: 워크플로를 API로 수정(PATCH)할 때마다 이 문제가 재발할 수 있습니다.
> 수정 후 항상 위 스크립트를 실행하거나, n8n UI에서 직접 수정하면 이 문제가 발생하지 않습니다.
