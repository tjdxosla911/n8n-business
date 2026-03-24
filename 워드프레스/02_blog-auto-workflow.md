# [실행 가이드] 블로그 자동생성 워크플로 (/블로그생성 → WordPress 발행)

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 WordPress Docker 설치(01) 완료 후 진행하세요.

---

## 개요

Mattermost에서 `/블로그생성 [주제]` 입력 → Claude API로 SEO 최적화 블로그 포스팅 생성 → WordPress REST API로 자동 발행 → Mattermost 채널에 완료 알림

기존 `/콘텐츠생성` (Threads SNS용)과 완전히 별개 워크플로입니다.

---

## 사전 조건

- [ ] WordPress Docker 설치 + 초기 설정 완료 (01번 가이드)
- [ ] WordPress 관리자 계정에서 **애플리케이션 비밀번호** 발급
  - wp-admin → 사용자 → 프로필 → 애플리케이션 비밀번호 → 새 비밀번호 추가
- [ ] 서버 SSH 접속 가능

---

## Claude에게 전달할 정보

```
서버 고정 IP              :
SSH 키 경로               :        (예: C:\Users\...\lightsail_key.pem)
WordPress URL             :        (예: https://blog.example.com)
WordPress 사용자명        :        (wp-admin 로그인 ID)
WordPress 애플리케이션 PW :        (예: LpnS GXrO Dp5h 2To3 Zwi5 5Bxw)
n8n URL                   :        (예: https://n8n.example.com)
n8n 이메일                :
n8n 비밀번호              :
n8n Basic Auth PW         :
Mattermost 관리자 이메일  :
Mattermost 관리자 비밀번호 :
```

---

## Claude 실행 내용 (참고용)

### 1단계: Bridge 서버 — `/generate/blog` 엔드포인트 추가

`/opt/bridge/server.js`에 추가 (기존 `/generate/sns` 뒤에):

```js
app.post('/generate/blog', async (req, res) => {
  const { topic } = req.body;

  if (!topic) {
    return res.status(400).json({ error: '주제(topic)가 필요합니다.' });
  }

  var systemPrompt = '당신은 SEO 전문 블로그 작가입니다.\n' +
    '중소기업 대표들이 검색할 만한 키워드를 자연스럽게 녹여서 글을 작성합니다.\n' +
    '아래 주제로 블로그 포스팅을 작성하세요.\n\n' +
    '주제: ' + topic + '\n\n' +
    '규칙:\n' +
    '- 제목: SEO 최적화. 검색 유입을 고려한 제목 (50자 이내)\n' +
    '- 본문: 1000자 이상. H2/H3 소제목을 활용한 구조화된 글\n' +
    '- 도입부: 독자의 문제/궁금증을 건드려서 스크롤하게 만들어라\n' +
    '- 본론: 구체적 수치, 사례, 비교를 포함. 뻔한 일반론 금지\n' +
    '- 결론: 핵심 요약 + 행동 유도 (CTA)\n' +
    '- 톤: 전문적이되 읽기 쉽게. 대표님들이 직원한테 공유할 수준\n' +
    '- HTML 태그 사용: <h2>, <h3>, <p>, <strong>, <ul>, <li> 등\n' +
    '- 메타 디스크립션: 검색 결과에 표시될 요약 (160자 이내)\n\n' +
    'JSON 형식으로 반환:\n' +
    '{\n' +
    '  "title": "블로그 제목",\n' +
    '  "content": "<h2>...</h2><p>...</p>...",\n' +
    '  "excerpt": "메타 디스크립션",\n' +
    '  "tags": ["태그1", "태그2", ...]\n' +
    '}';

  try {
    var message = await client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4000,
      system: systemPrompt,
      messages: [{ role: 'user', content: '블로그 포스팅을 작성해주세요.' }],
    });

    var text = message.content[0].text;

    // JSON 파싱 (여러 방법 시도)
    var parsed = null;
    try {
      var codeBlockMatch = text.match(/```(?:json)?\s*\n?([\s\S]*?)\n?```/);
      if (codeBlockMatch) {
        parsed = JSON.parse(codeBlockMatch[1].trim());
      }
    } catch (e) {}

    if (!parsed) {
      try {
        var firstBrace = text.indexOf('{');
        var lastBrace = text.lastIndexOf('}');
        if (firstBrace !== -1 && lastBrace > firstBrace) {
          parsed = JSON.parse(text.substring(firstBrace, lastBrace + 1));
        }
      } catch (e) {}
    }

    if (parsed) {
      console.log('[BLOG] 생성 완료: ' + (parsed.title || '').substring(0, 50));
      return res.json({ success: true, topic: topic, blog: parsed, raw: text });
    }
    console.log('[BLOG] JSON 파싱 실패 — raw 텍스트로 반환');
    return res.json({ success: true, topic: topic, blog: null, raw: text });
  } catch (err) {
    console.error('[BLOG ERROR]', err.message);
    return res.status(500).json({ error: err.message });
  }
});
```

> **시행착오**: 최초 버전에서 `text.match(/\{[\s\S]*\}/)` 단일 정규식으로 JSON을 추출했으나,
> Claude 응답에 HTML 중괄호나 마크다운 코드블록이 섞이면 파싱 실패 (500 에러).
> → 코드블록 추출 → firstBrace~lastBrace fallback 2단계 파싱으로 해결.

배포:

```bash
scp -i [SSH키] server.js ubuntu@[서버IP]:/opt/bridge/server.js
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart bridge-server"
```

---

### 2단계: n8n 워크플로 생성

Node.js 스크립트로 n8n API를 호출하여 워크플로 생성.

**워크플로 구조:**

```
/블로그생성 [주제]
 → Webhook (POST /webhook/blog-generate)
 → 즉시응답 "⏳ 블로그 포스팅 생성 중..."
 → 주제파싱 (Code)
 → Claude블로그생성 (HTTP POST → /generate/blog, timeout 120s)
 → 결과확인 (Code — blog JSON 유효성 검사)
 → WordPress발행 (HTTP POST → /wp-json/wp/v2/posts, Basic Auth)
 → Mattermost알림 (Mattermost 크레덴셜 노드)
```

**주요 노드 설정:**

WordPress발행 노드:
- Method: POST
- URL: `https://[WP_DOMAIN]/wp-json/wp/v2/posts`
- Body: keypair 방식 (`title`, `content`, `excerpt`, `status: publish`)
- Header: `Authorization: Basic [base64(사용자명:애플리케이션비밀번호)]`

Mattermost알림 노드:
- **반드시 Mattermost 크레덴셜 노드 사용** (HTTP Request 아님)
- 기존 `Mattermost Bot` 크레덴셜 + 동일 채널

> **시행착오**: 최초 Mattermost알림을 HTTP Request 노드로 구현하고 URL을 `http://localhost:8065`로 설정했으나,
> n8n이 Docker 컨테이너 안에서 실행되므로 `localhost`가 n8n 컨테이너 자체를 가리킴 → 연결 거부 에러.
> → 기존 콘텐츠생성 워크플로와 동일하게 **Mattermost 크레덴셜 노드** 방식으로 교체하여 해결.

활성화 후 반드시 DB 동기화:

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET \"activeVersionId\" = \"versionId\" WHERE name = '블로그 자동생성';"
cd /opt/n8n && docker compose restart n8n
```

---

### 3단계: Mattermost 슬래시 커맨드 등록

> **주의**: 봇 토큰으로는 슬래시 커맨드 생성 권한 없음 (403). **관리자 세션 토큰**으로 등록해야 함.

서버에서 SSH로 실행:

```bash
# 관리자 로그인 → 세션 토큰 획득
ADMIN_TOKEN=$(curl -si -X POST http://localhost:8065/api/v4/users/login \
  -H "Content-Type: application/json" \
  -d '{"login_id":"[ADMIN_EMAIL]","password":"[ADMIN_PASSWORD]"}' \
  | grep -i "^token:" | sed "s/token: //i" | tr -d "\r\n")

# 슬래시 커맨드 등록
curl -s -X POST http://localhost:8065/api/v4/commands \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "team_id": "[TEAM_ID]",
    "trigger": "블로그생성",
    "method": "P",
    "url": "https://[N8N_DOMAIN]/webhook/blog-generate",
    "display_name": "블로그 자동생성",
    "description": "SEO 블로그 포스팅 생성 후 WordPress 자동 발행",
    "auto_complete": true,
    "auto_complete_hint": "[주제]",
    "auto_complete_desc": "SEO 블로그 포스팅 생성 → WordPress 자동 발행",
    "username": "blog-bot"
  }'
```

> **시행착오**: Mattermost 관리자 로그인 시 이메일이 n8n 이메일과 다를 수 있음.
> DB에서 확인: `docker exec [MM_DB] psql -U [USER] -d mattermost -c "SELECT username, email FROM users WHERE roles LIKE '%system_admin%';"`

---

### 4단계: WordPress 루트 도메인 연결 (선택)

`bestrealinfo.com`, `www.bestrealinfo.com`도 같은 WordPress에 연결하는 경우:

```bash
# SSL 인증서 발급 (루트 + www 한번에)
sudo certbot certonly --nginx -d bestrealinfo.com -d www.bestrealinfo.com \
  --non-interactive --agree-tos -m [ADMIN_EMAIL]

# Nginx 설정
sudo tee /etc/nginx/sites-available/root > /dev/null << 'EOF'
server {
    listen 80;
    server_name bestrealinfo.com www.bestrealinfo.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name bestrealinfo.com www.bestrealinfo.com;

    ssl_certificate     /etc/letsencrypt/live/bestrealinfo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bestrealinfo.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 64m;

    location / {
        proxy_pass         http://localhost:8081;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/root /etc/nginx/sites-enabled/root
sudo nginx -t && sudo systemctl reload nginx
```

DNS A레코드 필요:
| 호스트 | 타입 | 값 |
|--------|------|----|
| `@` (루트) | A | [서버IP] |
| `www` | A | [서버IP] |

---

## 완료 후 Nginx 전체 구성

```bash
ls /etc/nginx/sites-enabled/
# api   blog   chat   dash   n8n   root
```

| 파일 | 도메인 | 포트 |
|------|--------|------|
| api | api.도메인 | 3100 (Bridge) |
| blog | blog.도메인 | 8081 (WordPress) |
| chat | chat.도메인 | 8065 (Mattermost) |
| dash | dash.도메인 | 3000 (Dashboard) |
| n8n | n8n.도메인 | 5678 (n8n) |
| root | 도메인, www.도메인 | 8081 (WordPress) |

---

## 완료 후 확인

```
/블로그생성 중소기업 전기세 절감 방법
→ "⏳ 블로그 포스팅 생성 중..." 즉시 응답
→ 30~60초 후 채널에 "📝 블로그 발행 완료! 제목: ... 👉 URL" 알림
→ WordPress에서 게시물 확인
```

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| Claude블로그생성 노드 500 에러 "service was not able to process" | JSON 파싱 실패 — Claude 응답에 HTML 중괄호/코드블록 혼재 | 코드블록 추출 + firstBrace~lastBrace fallback 2단계 파싱 적용 |
| Mattermost알림 노드 "service refused the connection" | n8n Docker에서 `localhost:8065` → n8n 컨테이너 자체를 가리킴 | HTTP Request → Mattermost 크레덴셜 노드로 교체 |
| 슬래시 커맨드 등록 403 Forbidden | 봇 토큰에 커맨드 생성 권한 없음 | 관리자 세션 토큰으로 등록 |
| WordPress 발행 401 Unauthorized | 애플리케이션 비밀번호 오류 또는 REST API 비활성화 | wp-admin에서 애플리케이션 비밀번호 재발급, `.htaccess`에서 REST API 차단 여부 확인 |
| WordPress 발행 후 HTML 태그가 그대로 노출 | WordPress가 HTML을 이스케이프 처리 | `content` 필드가 HTML raw로 전달되는지 확인, 필요시 `wp_kses` 필터 확인 |

---

## 서버 관리 명령어

```bash
# Bridge 서버 블로그 관련 로그
pm2 logs bridge-server --lines 20 | grep -i blog

# n8n 워크플로 상태 확인
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "SELECT name, active, (\"activeVersionId\" = \"versionId\") AS synced FROM workflow_entity;"

# WordPress REST API 테스트
curl -s -u '사용자명:애플리케이션비밀번호' \
  https://[WP_DOMAIN]/wp-json/wp/v2/posts?per_page=1 | python3 -m json.tool | head -10
```
