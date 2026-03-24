# [실행 가이드 07] Threads 자동 발행 연동

> `client.env`의 값을 사용합니다. 개별 전달 불필요.

---

## 사전 조건

- [ ] 01~06번 작업 완료 (n8n + 브릿지 서버 + 대시보드 실행 중)
- [ ] Threads 앱 등록 및 액세스 토큰 발급 완료
- [ ] Threads user_id 확인 (`https://graph.threads.net/v1.0/me?access_token=...`)

---

## client.env에서 사용하는 변수

```
DOMAIN                  → n8n: https://n8n.{DOMAIN}, 브릿지: https://api.{DOMAIN}, 대시보드: https://dash.{DOMAIN}
SSH_KEY_PATH            → 서버 SSH 접속용
SERVER_IP               → 01단계에서 자동 할당된 고정 IP
ADMIN_EMAIL             → n8n 로그인 이메일
N8N_PASSWORD            → n8n 로그인 비밀번호
N8N_BASIC_AUTH_PASSWORD → n8n Basic Auth (API 접근용)
THREADS_APP_ID          → Meta Threads 앱 ID
THREADS_APP_SECRET      → Meta Threads 앱 시크릿
THREADS_ACCESS_TOKEN    → Threads 액세스 토큰
THREADS_USER_ID         → Threads user_id (비워두면 설치 시 자동 조회)
```

---

## 아키텍처

```
대시보드 승인 버튼 클릭
 → n8n /webhook/content-approve 호출
 → n8n Code 노드: 텍스트 조립 (title + body + hashtags)
 → n8n HTTP Request: 브릿지 서버 /publish/threads 호출
 → 브릿지 서버: Threads API 직접 호출 (UTF-8 보장)
     ├→ POST /v1.0/{user_id}/threads (컨테이너 생성)
     ├→ 2초 대기
     └→ POST /v1.0/{user_id}/threads_publish (발행)
 → n8n HTTP Request: 대시보드 /api/contents/status 호출 (상태 업데이트)
```

### 왜 브릿지 서버를 경유하는가?

- n8n Docker 컨테이너의 HTTP Request 노드에서 한글이 깨지는 인코딩 문제 발생
- n8n Code 노드는 샌드박스 환경이라 `fetch` 사용 불가
- 브릿지 서버(Node.js)에서 직접 `fetch`로 Threads API 호출 시 UTF-8 정상 처리
- n8n → `https://api.도메인/publish/threads` (Nginx 경유, 같은 서버)

---

## Claude 실행 내용 (참고용)

### 1단계: 브릿지 서버에 Threads 발행 엔드포인트 추가

`/opt/bridge/server.js`에 추가:

```js
// Threads 설정
const THREADS_USER_ID = process.env.THREADS_USER_ID || '[USER_ID]';
const THREADS_TOKEN   = process.env.THREADS_ACCESS_TOKEN || '';

// POST /publish/threads
app.post('/publish/threads', async (req, res) => {
  const { text, contentId } = req.body;
  if (!text) return res.status(400).json({ success: false, error: 'text 필요' });

  try {
    // 컨테이너 생성
    const containerRes = await fetch(
      `https://graph.threads.net/v1.0/${THREADS_USER_ID}/threads`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json; charset=utf-8' },
        body: JSON.stringify({
          media_type: 'TEXT',
          text: text,
          access_token: THREADS_TOKEN,
        }),
      }
    );
    const containerData = await containerRes.json();
    if (!containerData.id) {
      return res.json({ success: false, contentId, threadId: null,
        message: '컨테이너 생성 실패: ' + JSON.stringify(containerData) });
    }

    // 2초 대기 (Threads API 권장)
    await new Promise(r => setTimeout(r, 2000));

    // 발행
    const publishRes = await fetch(
      `https://graph.threads.net/v1.0/${THREADS_USER_ID}/threads_publish`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json; charset=utf-8' },
        body: JSON.stringify({
          creation_id: containerData.id,
          access_token: THREADS_TOKEN,
        }),
      }
    );
    const publishData = await publishRes.json();
    const success = !!publishData.id;

    return res.json({
      success, contentId,
      threadId: publishData.id ? String(publishData.id) : null,
      message: success
        ? 'Threads 발행 완료 (ID: ' + publishData.id + ')'
        : 'Threads 발행 실패: ' + JSON.stringify(publishData),
    });
  } catch (err) {
    return res.status(500).json({
      success: false, contentId, threadId: null,
      message: 'Threads 발행 에러: ' + err.message,
    });
  }
});
```

### 2단계: PM2 재시작 (환경변수 포함)

```bash
# 기존 Anthropic API Key 확보
ANTHROPIC_KEY=$(pm2 env 0 2>/dev/null | grep ANTHROPIC_API_KEY | head -1 | sed "s/.*: //")

pm2 delete bridge-server

ANTHROPIC_API_KEY="$ANTHROPIC_KEY" \
THREADS_USER_ID="[USER_ID]" \
THREADS_ACCESS_TOKEN="[ACCESS_TOKEN]" \
pm2 start /opt/bridge/server.js --name bridge-server

pm2 save

# 확인
curl -s http://localhost:3100/health
```

### 3단계: 대시보드 수정

`/opt/dashboard/app.js`에 추가:

1. DB 컬럼: `thread_id TEXT`, `publish_message TEXT`
2. API: `POST /api/contents/status` (발행 상태 업데이트)
3. counts에 `published` 추가
4. 목록/상세 뷰에 `published`, `publish_failed` 상태 배지 추가
5. **approve 라우트**: n8n이 `published`로 먼저 업데이트하면 덮어쓰지 않도록 조건부 처리

```js
// approve 라우트 내 — 웹훅 호출 후 상태 업데이트
const current = db.prepare('SELECT status FROM contents WHERE id = ?').get(req.params.id);
if (current && current.status !== 'published') {
  db.prepare('UPDATE contents SET status=\'approved\', updated_at=CURRENT_TIMESTAMP WHERE id=?')
    .run(req.params.id);
}
```

```bash
pm2 restart dashboard
```

### 4단계: n8n 워크플로우 생성

Node.js 스크립트로 n8n API를 통해 워크플로우 자동 등록:

```
워크플로우명: Threads 자동발행
Webhook: POST /webhook/content-approve

노드 구성:
  Webhook
   → Threads텍스트조립 (Code)
   → 브릿지서버Threads발행 (HTTP POST → https://api.도메인/publish/threads)
   → 대시보드상태업데이트 (HTTP POST → https://dash.도메인/api/contents/status)
```

⚠ n8n v2 활성화 버그 대응 필요 (DB 직접 수정):

```bash
VERSION_ID=$(docker exec n8n-postgres psql -U n8n -d n8n -t -c \
  "SELECT \"versionId\" FROM workflow_entity WHERE name = 'Threads 자동발행';" | tr -d ' \n')

docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET active=true, \"activeVersionId\"='$VERSION_ID' WHERE name = 'Threads 자동발행';"

cd /opt/n8n && docker compose restart n8n
```

---

## API 명세

### POST /publish/threads (브릿지 서버)

```
POST https://api.도메인/publish/threads
Content-Type: application/json; charset=utf-8

{
  "text": "게시물 텍스트",
  "contentId": "대시보드 콘텐츠 ID"
}

응답:
{
  "success": true,
  "contentId": "4",
  "threadId": "18224084113309654",
  "message": "Threads 발행 완료 (ID: 18224084113309654)"
}
```

### POST /api/contents/status (대시보드)

```
POST https://dash.도메인/api/contents/status
Content-Type: application/json

{
  "id": 4,
  "status": "published",        // published | publish_failed
  "threadId": "18224084113309654",
  "message": "Threads 발행 완료 (ID: 18224084113309654)"
}
```

---

## 완료 후 확인

1. 대시보드에서 `pending` 콘텐츠 선택 → **승인** 클릭
2. n8n 워크플로우 실행 확인
3. Threads 앱에서 게시물 발행 확인 (한글 정상 출력)
4. 대시보드에서 `published` 탭으로 상태 변경 확인
5. Thread ID 표시 확인

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| n8n HTTP Request로 직접 Threads API 호출 시 한글 깨짐 | n8n Docker 컨테이너 인코딩 문제 | 브릿지 서버 경유 방식으로 우회 |
| n8n Code 노드에서 `fetch is not defined` | Task Runner 샌드박스 환경 제한 | HTTP Request 노드 또는 브릿지 서버 사용 |
| n8n `jsonBody` 내 `={{ }}` 표현식 미평가 | n8n v2 버그 | `specifyBody: 'keypair'` 방식 사용 |
| `creation_id must be a number` | jsonBody에서 표현식이 문자열로 전달 | keypair 방식으로 변경 |
| n8n에서 `localhost:3100` 접근 불가 | n8n은 Docker 컨테이너, 브릿지는 호스트 | Nginx 도메인 경유 (`https://api.도메인`) |
| 워크플로우 PATCH 후 웹훅 미등록 | n8n v2 activeVersionId 버그 | DB 직접 수정 후 n8n 재시작 |
| 승인 후 상태가 `approved`로 남음 | approve 라우트가 n8n보다 늦게 실행되어 `published`를 덮어씀 | approve 시 현재 상태가 `published`면 덮어쓰지 않도록 조건 추가 |

---

## 서버 관리 명령어

```bash
# 브릿지 서버 로그 (Threads 발행 로그 포함)
pm2 logs bridge-server --lines 30

# 대시보드 로그
pm2 logs dashboard --lines 30

# n8n 워크플로우 상태
# https://n8n.bestrealinfo.com/workflow/[ID]

# Threads 게시물 확인
curl -s "https://graph.threads.net/v1.0/[THREAD_ID]?fields=id,text&access_token=[TOKEN]"
```
