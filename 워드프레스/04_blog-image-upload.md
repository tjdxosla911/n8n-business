# [실행 가이드] 블로그 이미지 첨부 기능 (Mattermost → WordPress 썸네일)

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 블로그 워크플로(02) 완료 후 진행하세요.

---

## 개요

Mattermost에서 이미지를 첨부하여 블로그를 생성하면, 해당 이미지가 WordPress에 업로드되어 글의 대표 이미지(썸네일)로 설정됩니다.

### 두 가지 블로그 생성 방식

| 방식 | 사용법 | 이미지 | 트리거 |
|------|--------|--------|--------|
| 슬래시 커맨드 | `/블로그생성 키워드` | 텍스트만 | 슬래시 커맨드 |
| 일반 메시지 | `블로그생성 키워드` + 이미지 첨부 | 썸네일 업로드 | Outgoing Webhook |

> **시행착오**: 최초에 슬래시 커맨드(`/블로그생성`)에 이미지 첨부를 시도했으나, **Mattermost 슬래시 커맨드는 파일 첨부를 지원하지 않습니다.** Webhook body에 `file_ids`가 전달되지 않음.
> → 기존 "이미지 콘텐츠생성" 워크플로와 동일하게 **Outgoing Webhook + 트리거 단어** 방식으로 해결.

---

## 사전 조건

- [ ] 02번 가이드 완료 (블로그 워크플로 + WordPress 발행 동작)
- [ ] Bridge 서버에 `/upload/wordpress-media` 엔드포인트 존재
- [ ] 서버 SSH 접속 가능

---

## Claude에게 전달할 정보

```
서버 고정 IP              :
SSH 키 경로               :
WordPress URL             :        (예: https://blog.example.com)
WordPress 사용자명        :
WordPress 애플리케이션 PW :
n8n URL                   :
n8n 이메일                :
n8n 비밀번호              :
n8n Basic Auth PW         :
Mattermost 관리자 이메일  :
Mattermost 관리자 비밀번호 :
콘텐츠 채널 ID            :        (기존 콘텐츠생성 채널)
```

---

## Claude 실행 내용 (참고용)

### 1단계: Bridge 서버 — `/upload/wordpress-media` 엔드포인트 추가

Mattermost에서 이미지 다운로드 → WordPress REST API로 업로드 → media ID 반환.

`/opt/bridge/server.js`에 추가:

```js
app.post('/upload/wordpress-media', async (req, res) => {
  var { file_id, wp_url, wp_auth } = req.body;
  if (!file_id || !wp_url || !wp_auth) {
    return res.status(400).json({ success: false, error: 'file_id, wp_url, wp_auth 필요' });
  }

  try {
    // 1. Mattermost에서 이미지 다운로드
    var mmRes = await fetch('http://localhost:8065/api/v4/files/' + file_id, {
      headers: { 'Authorization': 'Bearer ' + MM_BOT_TOKEN }
    });
    if (!mmRes.ok) {
      return res.json({ success: false, error: 'Mattermost 다운로드 실패: ' + mmRes.status });
    }

    // 파일 정보
    var infoRes = await fetch('http://localhost:8065/api/v4/files/' + file_id + '/info', {
      headers: { 'Authorization': 'Bearer ' + MM_BOT_TOKEN }
    });
    var fileInfo = await infoRes.json();
    var filename = fileInfo.name || ('image_' + Date.now() + '.jpg');
    var mimeType = fileInfo.mime_type || 'image/jpeg';
    var buffer = Buffer.from(await mmRes.arrayBuffer());

    // 2. WordPress REST API로 업로드
    var wpRes = await fetch(wp_url.replace(/\/$/, '') + '/wp-json/wp/v2/media', {
      method: 'POST',
      headers: {
        'Authorization': 'Basic ' + wp_auth,
        'Content-Disposition': 'attachment; filename="' + filename + '"',
        'Content-Type': mimeType,
      },
      body: buffer,
    });
    var wpData = await wpRes.json();

    if (!wpRes.ok) {
      return res.json({ success: false, error: 'WordPress 업로드 실패: ' + (wpData.message || wpRes.status) });
    }

    return res.json({
      success: true,
      mediaId: wpData.id,
      mediaUrl: wpData.source_url || '',
      filename: filename,
    });
  } catch (err) {
    return res.status(500).json({ success: false, error: err.message });
  }
});
```

요청/응답:

```
POST /upload/wordpress-media
{
  "file_id": "mattermost_file_id",
  "wp_url": "https://blog.example.com",
  "wp_auth": "base64(사용자명:애플리케이션비밀번호)"
}

→ { "success": true, "mediaId": 123, "mediaUrl": "https://...", "filename": "image.jpg" }
```

---

### 2단계: n8n 워크플로 생성 — "이미지 블로그생성"

기존 "블로그 자동생성" (슬래시 커맨드)과 별개 워크플로.

```
채널에 "블로그생성 키워드" + 이미지 첨부
 → Outgoing Webhook → n8n Webhook
 → 메시지파싱 (topic + file_ids 추출)
 → 즉시응답 "⏳ 블로그 포스팅 생성 중..."
 → 이미지분기 (hasImage?)
    ├→ [이미지O] WP미디어업로드 → Claude블로그생성 → WP발행(featured_media 포함) → Mattermost알림
    └→ [이미지X] Claude블로그생성 → WP발행(텍스트만) → Mattermost알림
```

**메시지파싱** 노드 — Outgoing Webhook의 body에서 데이터 추출:

```js
const data = $input.first().json.body || $input.first().json;
const rawText = String(data.text || '');
const topic = rawText.replace(/^블로그생성\s*/, '').trim() || 'AI 자동화로 중소기업 비용 절감';
// Outgoing Webhook은 file_ids를 쉼표 구분 문자열로 전달
const fileIds = data.file_ids ? String(data.file_ids).split(',').map(s => s.trim()).filter(Boolean) : [];
const userName = String(data.user_name || 'unknown');
const channelId = String(data.channel_id || '');
return [{ json: { topic, fileIds, userName, channelId, hasImage: fileIds.length > 0 } }];
```

> **핵심**: Outgoing Webhook의 `file_ids`는 **쉼표 구분 문자열**이지 배열이 아님.
> `String(data.file_ids).split(',')` 로 파싱해야 함.

**WP발행(이미지)** 노드 — `featured_media`에 media ID 삽입:

```js
// jsonBody
JSON.stringify({
  title: $json.blog.title,
  content: $json.blog.content,
  excerpt: $json.blog.excerpt,
  status: "publish",
  featured_media: $("WP미디어업로드").first().json.mediaId || 0
})
```

---

### 3단계: Mattermost Outgoing Webhook 등록

서버에서 관리자 세션으로 실행:

```bash
ADMIN_TOKEN=$(curl -si -X POST http://localhost:8065/api/v4/users/login \
  -H "Content-Type: application/json" \
  -d '{"login_id":"[ADMIN_EMAIL]","password":"[ADMIN_PW]"}' \
  | grep -i "^token:" | sed "s/token: //i" | tr -d "\r\n")

curl -s -X POST http://localhost:8065/api/v4/hooks/outgoing \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "team_id": "[TEAM_ID]",
    "channel_id": "[CHANNEL_ID]",
    "display_name": "이미지 블로그생성",
    "description": "이미지+키워드로 블로그 포스팅 생성 후 WordPress 발행",
    "trigger_words": ["블로그생성"],
    "callback_urls": ["https://[N8N_DOMAIN]/webhook/blog-generate-image"],
    "content_type": "application/x-www-form-urlencoded"
  }'
```

> Outgoing Webhook은 특정 채널 + 트리거 단어로 동작. 슬래시 커맨드와 충돌하지 않음.
> - 슬래시: `/블로그생성 키워드` → 슬래시 커맨드 워크플로 (이미지 없음)
> - 일반 메시지: `블로그생성 키워드` + 이미지 → Outgoing Webhook 워크플로 (이미지 있음)

---

### 4단계: n8n 활성화 + DB 동기화

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET \"activeVersionId\" = \"versionId\" WHERE name = '이미지 블로그생성';"
cd /opt/n8n && docker compose restart n8n
```

---

## 배포

```bash
# Bridge 서버 (1단계)
scp -i [SSH키] server.js ubuntu@[서버IP]:/opt/bridge/server.js
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart bridge-server"

# n8n 워크플로 (2단계) — Node.js 스크립트로 생성
node create-blog-image-workflow.mjs

# Mattermost Webhook (3단계) — 서버에서 실행
ssh -i [SSH키] ubuntu@[서버IP] "..."

# DB 동기화 (4단계)
ssh -i [SSH키] ubuntu@[서버IP] "docker exec n8n-postgres psql ..."
```

---

## 완료 후 확인

1. 콘텐츠생성 채널에서 이미지를 첨부하고 `블로그생성 중소기업 전기세 절감` 메시지 전송
2. "⏳ 블로그 포스팅 생성 중..." 즉시 응답 확인
3. 30~60초 후 "📝 블로그 발행 완료!" 알림 확인
4. WordPress에서 글 확인 → 대표 이미지(썸네일)에 첨부한 이미지가 설정되어 있는지

이미지 없이 `블로그생성 키워드` 만 보내면 이미지 없이 발행 (기존과 동일).

---

## n8n 워크플로 전체 목록 (최종)

| 워크플로 | 트리거 | 용도 |
|----------|--------|------|
| Mattermost 콘텐츠생성 | `/콘텐츠생성 [유형] [주제]` | Threads SNS 콘텐츠 |
| 이미지 콘텐츠생성 | `콘텐츠생성 [주제]` + 이미지 | Threads SNS (이미지 포함) |
| Threads 자동발행 | 대시보드 승인 | Threads 발행 |
| 블로그 자동생성 | `/블로그생성 [주제]` | WordPress 블로그 (텍스트) |
| **이미지 블로그생성** | **`블로그생성 [주제]` + 이미지** | **WordPress 블로그 (썸네일 포함)** |

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 슬래시 커맨드로 이미지 첨부 안 됨 | Mattermost 슬래시 커맨드는 file_ids를 전달하지 않음 | Outgoing Webhook + 일반 메시지 방식 사용 |
| Outgoing Webhook에서 file_ids가 빈 배열 | `file_ids`가 쉼표 구분 문자열인데 배열로 처리 | `String(data.file_ids).split(',')` 로 파싱 |
| WordPress 미디어 업로드 401 | 애플리케이션 비밀번호 오류 | wp-admin에서 재발급 |
| 이미지 업로드 성공했으나 썸네일 안 보임 | WordPress 테마에서 featured_media 미지원 | 테마 설정에서 "글 대표 이미지" 활성화 |
| 같은 채널에서 "콘텐츠생성"과 "블로그생성" 충돌 | 두 Outgoing Webhook이 같은 채널에서 각각 다른 트리거 단어로 동작 | 충돌 없음 — 트리거 단어가 다름 |
