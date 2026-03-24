# [실행 가이드 08] 이미지 콘텐츠 생성 + 발행 연동

> 01~07 완료 후 진행. 이미지 포함 SNS 콘텐츠를 Mattermost에서 생성하고 Threads에 이미지+텍스트로 발행한다.

---

## 사전 조건

- [ ] 01~07번 작업 완료 (n8n + 브릿지 서버 + 대시보드 + Threads 텍스트 발행 정상)
- [ ] 사용자가 이미지를 직접 준비 (젠스파크 나노바나나, Canva 등)

---

## 아키텍처

```
[Mattermost 채널]
  이미지 첨부 + "콘텐츠생성 상품명" 메시지 (슬래시 없이)
    ↓ Outgoing Webhook
[n8n - 이미지 콘텐츠생성 워크플로우]
  1. 웹훅 수신 → file_ids + 상품명 파싱
  2. 브릿지 /download-mattermost-image → Mattermost에서 이미지 다운로드
  3. 이미지를 대시보드 public/uploads/ 에 저장 → public URL 확보
  4. 브릿지 /generate/sns → Claude 텍스트 생성
  5. 대시보드 /api/contents 저장 (image_url 포함)
  6. Mattermost 채널에 완료 알림
    ↓
[대시보드]
  이미지 미리보기 + 텍스트 확인/수정 → 승인 클릭
    ↓ n8n 웹훅
[브릿지 서버]
  Threads API IMAGE 타입으로 발행
    ├→ POST /v1.0/{user_id}/threads (media_type: IMAGE, image_url, text)
    ├→ 5초 대기 (이미지 처리)
    └→ POST /v1.0/{user_id}/threads_publish
```

### 기존 텍스트 전용 플로우와의 차이

| 항목 | 텍스트 전용 (기존) | 이미지+텍스트 (신규) |
|------|-------------------|---------------------|
| 트리거 | 슬래시커맨드 `/콘텐츠생성` | Outgoing Webhook `콘텐츠생성` (슬래시 없이) |
| 이미지 | 없음 | Mattermost 첨부파일 다운로드 |
| Threads 발행 | media_type: TEXT | media_type: IMAGE + image_url |
| 대기 시간 | 2초 | 5초 (이미지 처리) |

---

## Claude 실행 내용 (참고용)

### 1단계: 대시보드 수정

`/opt/dashboard/app.js` 변경:

- 의존성 추가: `npm install multer`
- `public/uploads/` 디렉토리 생성 (이미지 저장용, Nginx 통해 public 서빙)
- DB 컬럼 추가: `image_url TEXT DEFAULT NULL`
- `POST /api/upload` 엔드포인트 추가 (multer로 이미지 수신 → uploads/ 저장 → URL 반환)
- `POST /api/contents`: `image_url` 필드 수신/저장
- `POST /contents/:id/approve`: n8n 웹훅 호출 시 `image_url` 포함

뷰 템플릿 변경:

- `views/index.ejs`: 목록 카드에 이미지 썸네일 (16x16) 표시
- `views/detail.ejs`: 상세 페이지 상단에 이미지 미리보기, 승인 버튼에 "(이미지+텍스트)" 표시

```bash
cd /opt/dashboard && npm install multer && mkdir -p public/uploads
pm2 restart dashboard
```

### 2단계: 브릿지 서버 수정

`/opt/bridge/server.js` 변경:

**`POST /publish/threads` 수정:**
- `imageUrl` 파라미터 추가
- imageUrl 있으면 `media_type: 'IMAGE'` + `image_url` 사용
- 이미지 대기 시간 5초 (텍스트는 2초)

**`POST /download-mattermost-image` 신규 엔드포인트:**
- Mattermost API (`localhost:8065/api/v4/files/{file_id}`)에서 봇 토큰으로 이미지 다운로드
- `/opt/dashboard/public/uploads/` 에 저장
- `https://dash.{DOMAIN}/uploads/{filename}` URL 반환

```bash
pm2 restart bridge-server
```

### 3단계: Mattermost Outgoing Webhook 생성

관리자 로그인 후 API로 생성:

```bash
# 관리자 토큰 획득
ADMIN_TOKEN=$(curl -s -i -X POST http://localhost:8065/api/v4/users/login \
  -H 'Content-Type: application/json' \
  -d '{"login_id":"admin","password":"[ADMIN_PASSWORD]"}' | grep -i '^token:' | awk '{print $2}' | tr -d '\r')

# Outgoing Webhook 활성화
curl -s -X PUT -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  'http://localhost:8065/api/v4/config/patch' \
  -d '{"ServiceSettings":{"EnableOutgoingWebhooks":true}}'

# Outgoing Webhook 생성
curl -s -X POST -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  'http://localhost:8065/api/v4/hooks/outgoing' \
  -d '{
    "team_id": "[TEAM_ID]",
    "channel_id": "[콘텐츠생성 채널 ID]",
    "display_name": "이미지 콘텐츠생성",
    "description": "이미지+텍스트 콘텐츠 생성 트리거",
    "trigger_words": ["콘텐츠생성"],
    "trigger_when": 0,
    "callback_urls": ["https://n8n.{DOMAIN}/webhook/content-generate-image"],
    "content_type": "application/json"
  }'
```

### 4단계: n8n 워크플로우

**신규 워크플로우: "이미지 콘텐츠생성"**

```
Webhook (POST /webhook/content-generate-image)
  → 메시지파싱 (Code): text에서 상품명 추출, file_ids 파싱
  → 이미지다운로드 (HTTP POST → https://api.{DOMAIN}/download-mattermost-image)
  → 데이터병합 (Code): imageUrl + product + userName 조합
  → 텍스트생성 (HTTP POST → https://api.{DOMAIN}/generate/sns)
  → 대시보드저장 (HTTP POST → https://dash.{DOMAIN}/api/contents) (image_url 포함)
  → Mattermost응답 (Respond to Webhook)
```

**기존 워크플로우 수정: "Threads 자동발행"**

- Code 노드: `data.image_url` 추출하여 `imageUrl`로 전달
- 브릿지 HTTP Request: `imageUrl` 파라미터 추가

⚠ n8n v2 활성화 버그 대응 (07과 동일):

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET \"activeVersionId\" = \"versionId\" WHERE active = true;"
cd /opt/n8n && docker compose restart n8n
```

---

## API 명세 (신규)

### POST /download-mattermost-image (브릿지 서버)

```
POST https://api.{DOMAIN}/download-mattermost-image
Content-Type: application/json

{ "file_id": "Mattermost 파일 ID" }

응답:
{ "success": true, "imageUrl": "https://dash.{DOMAIN}/uploads/1773931012434-nj5sty.png", "filename": "1773931012434-nj5sty.png" }
```

### POST /api/upload (대시보드)

```
POST https://dash.{DOMAIN}/api/upload
Content-Type: multipart/form-data
Field: image (파일)

응답:
{ "success": true, "imageUrl": "https://dash.{DOMAIN}/uploads/filename.jpg", "filename": "filename.jpg" }
```

### POST /publish/threads 변경점 (브릿지 서버)

```
POST https://api.{DOMAIN}/publish/threads
Content-Type: application/json

{
  "text": "게시물 텍스트",
  "contentId": "대시보드 콘텐츠 ID",
  "imageUrl": "https://dash.{DOMAIN}/uploads/image.png"  ← 신규 (없으면 TEXT 발행)
}
```

---

## 사용 방법

### 이미지 포함 콘텐츠 (신규)
1. 젠스파크/Canva 등에서 이미지 생성
2. Mattermost `콘텐츠생성` 채널에서 이미지 첨부 + `콘텐츠생성 상품명` 입력 (**슬래시 없이**)
3. 대시보드에서 이미지+텍스트 미리보기 확인/수정
4. 승인 → Threads에 이미지+텍스트 자동 발행

### 텍스트 전용 콘텐츠 (기존 그대로)
1. Mattermost에서 `/콘텐츠생성 상품명` 슬래시커맨드
2. 대시보드에서 텍스트 확인/수정 → 승인 → Threads TEXT 발행

---

## Threads 해시태그 참고

- Threads에서 해시태그는 아이디 옆에 **토픽 태그**로 표시됨 (예: `freefree87st > 발리여행`)
- 본문 하단에 `#태그` 를 많이 나열해도 텍스트로만 보임 (클릭 불가)
- 토픽 태그는 Threads 앱에서 게시 시 수동 설정하거나, API의 `text` 첫 부분에 포함
- 해시태그 과다 사용보다 핵심 키워드 1~2개를 토픽으로 설정하는 것이 효과적

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 이미지 첨부했는데 텍스트만 생성됨 | `/콘텐츠생성` (슬래시커맨드) 사용 | 슬래시 없이 `콘텐츠생성 상품명` 으로 입력 |
| `API access blocked` | Threads 액세스 토큰 만료/차단 | Meta 개발자 콘솔에서 새 토큰 발급 후 PM2 환경변수 교체 |
| 이미지 발행 후 빈 게시물 | image_url이 외부에서 접근 불가 | dash.{DOMAIN}/uploads/ 경로가 Nginx에서 정상 서빙되는지 확인 |
| Outgoing Webhook 미동작 | Mattermost 설정에서 Outgoing Webhook 비활성화 | 관리자 콘솔 → 통합 → Outgoing Webhook 활성화 |

---

## Threads 토큰 갱신 방법

```bash
# 현재 토큰 확인
pm2 env [bridge-server ID] | grep THREADS_ACCESS_TOKEN

# 토큰 교체
ANTHROPIC_KEY=$(pm2 env [bridge-server ID] | grep ANTHROPIC_API_KEY | head -1 | sed "s/.*: //")
THREADS_UID=$(pm2 env [bridge-server ID] | grep THREADS_USER_ID | head -1 | sed "s/.*: //")

pm2 delete bridge-server

ANTHROPIC_API_KEY="$ANTHROPIC_KEY" \
THREADS_USER_ID="$THREADS_UID" \
THREADS_ACCESS_TOKEN="[새 토큰]" \
pm2 start /opt/bridge/server.js --name bridge-server

pm2 save
```

---

## client.env 추가 변수

```
# ── 7. Mattermost 관리자 (08단계에서 사용) ─────────────────────
MATTERMOST_ADMIN_PASSWORD=       (예: Admin1234!)
```
