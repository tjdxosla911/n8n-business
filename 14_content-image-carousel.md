# [실행 가이드 14] 콘텐츠생성 이미지 첨부 + Threads 캐러셀 발행

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `04`, `12~13` 작업 완료 후 진행하세요.
> 이 가이드는 04번의 콘텐츠생성 워크플로우를 **이미지 첨부 지원 버전으로 업그레이드**합니다.

---

## 사전 조건

- [ ] 04번: n8n Mattermost Credential + 콘텐츠생성 워크플로우 완료
- [ ] 12번: 미디어 파일 서버 구축 완료 (https://media.도메인)
- [ ] 13번: 미디어 워크플로우 완료 (Personal Access Token + HTTP Header Auth Credential 등록됨)

---

## Claude에게 전달할 정보

```
n8n URL              :        (예: https://n8n.example.com)
n8n 이메일           :
n8n 비밀번호         :

Mattermost URL       :        (예: https://chat.example.com)
Mattermost 관리자 ID :
Mattermost 관리자 PW :

미디어 서버 URL      :        (예: https://media.example.com)
브릿지 서버 URL      :        (예: https://api.example.com)
대시보드 URL         :        (예: https://dash.example.com)
콘텐츠 채널 ID       :        (04번 작업 시 생성된 Mattermost 채널 ID)

서버 SSH 키 경로     :        (예: C:\Users\...\lightsail_key.pem)
서버 호스트          :        (예: n8n.example.com 또는 서버 IP)
```

---

## 기능 설명

### 변경 요약

| 항목 | Before (04번) | After (14번) |
|------|--------------|-------------|
| 트리거 | `/콘텐츠생성` (슬래시 커맨드) | `!콘텐츠생성` (Outgoing Webhook) |
| 이미지 첨부 | 불가 | 여러 장 가능 |
| 대시보드 목록 | 텍스트만 | 첫 번째 이미지 썸네일 표시 |
| 대시보드 상세 | 텍스트만 | 여러 이미지 미리보기 (가로 배치) |
| Threads 발행 | 텍스트만 또는 이미지 1장 | **캐러셀 지원** (이미지 2~20장) |

> ⚠ Mattermost 슬래시 커맨드(`/`)는 파일 첨부를 지원하지 않음
> → Outgoing Webhook 방식(`!`)으로 변경하여 이미지 첨부 가능

> ⚠ Threads는 텍스트 중간에 이미지를 삽입할 수 없음 (텍스트와 이미지 분리)
> → 이미지는 캐러셀(스와이프)로 별도 첨부, 텍스트는 Claude가 이미지 없이 생성

### 사용법

```
!콘텐츠생성 상품명                         ← 텍스트만
!콘텐츠생성 일상 오늘의 커피               ← 유형 지정 (일상/지식/홍보)
!콘텐츠생성 상품명 + [이미지 3장 첨부]     ← 이미지 포함
```

### 전체 흐름

```
Mattermost !콘텐츠생성 상품명 + [이미지 첨부]
 │
 └→ n8n 콘텐츠생성 워크플로우
     ├→ 즉시응답: "⏳ 콘텐츠 생성 중..."
     ├→ 이미지 → 미디어 서버 저장 → URL 생성
     ├→ Claude 브릿지 → SNS 콘텐츠 생성 (텍스트만, 이미지 분석 없음)
     ├→ Mattermost 채널에 결과 게시
     └→ 대시보드에 저장 (image_url: 쉼표 구분 URL)
         │
         └→ 대시보드에서 승인 버튼 클릭
             │
             └→ n8n Threads 자동발행 워크플로우
                 ├→ 이미지 0장 → TEXT 발행
                 ├→ 이미지 1장 → IMAGE 발행
                 └→ 이미지 2장+ → CAROUSEL 발행 (캐러셀)
```

### Threads 발행 타입

| 첨부 이미지 | Threads 발행 타입 | 결과 |
|------------|-----------------|------|
| 0장 | TEXT | 텍스트만 발행 |
| 1장 | IMAGE | 이미지 1장 + 텍스트 |
| 2~20장 | CAROUSEL | 캐러셀 (스와이프) + 텍스트 |

---

## Claude 실행 내용 (참고용)

### 실행 순서

1. 브릿지 서버 업데이트: `/publish/threads` 캐러셀 지원 추가
2. n8n 콘텐츠생성 워크플로우 업데이트: `!콘텐츠생성` + 이미지 처리
3. n8n Threads 자동발행 워크플로우 업데이트: `imageUrls` 배열 전달
4. 대시보드 UI 업데이트: 여러 이미지 표시 + 리스트 썸네일
5. Mattermost: `/콘텐츠생성` 슬래시 커맨드 삭제 + `!콘텐츠생성` Outgoing Webhook 생성

### 1. 브릿지 서버: 캐러셀 지원 추가

`/opt/bridge/server.js`의 `POST /publish/threads` 엔드포인트를 교체합니다.

**변경 전**: `imageUrl` (단일 문자열) → TEXT 또는 IMAGE 발행만 지원
**변경 후**: `imageUrls` (배열) 추가 → TEXT / IMAGE / CAROUSEL 3가지 지원 (하위호환 유지)

```js
// POST /publish/threads 핵심 로직
app.post('/publish/threads', async (req, res) => {
  const { text, contentId, imageUrl, imageUrls } = req.body;

  // imageUrls (배열) 또는 imageUrl (단일) 지원 - 하위호환
  var urls = [];
  if (imageUrls && Array.isArray(imageUrls) && imageUrls.length > 0) {
    urls = imageUrls;
  } else if (imageUrl) {
    urls = [imageUrl];
  }

  var mediaType = urls.length === 0 ? 'TEXT' : (urls.length === 1 ? 'IMAGE' : 'CAROUSEL');

  if (mediaType === 'CAROUSEL') {
    // 1단계: 각 이미지 개별 컨테이너 생성 (is_carousel_item: true)
    var childIds = [];
    for (var i = 0; i < urls.length; i++) {
      var cr = await fetch('https://graph.threads.net/v1.0/' + THREADS_USER_ID + '/threads', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json; charset=utf-8' },
        body: JSON.stringify({
          media_type: 'IMAGE',
          image_url: urls[i],
          is_carousel_item: true,
          access_token: token,
        }),
      });
      var cd = await cr.json();
      childIds.push(cd.id);
      await new Promise(resolve => setTimeout(resolve, 2000)); // 이미지 처리 대기
    }

    // 2단계: 캐러셀 컨테이너 생성
    var carouselRes = await fetch('https://graph.threads.net/v1.0/' + THREADS_USER_ID + '/threads', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json; charset=utf-8' },
      body: JSON.stringify({
        media_type: 'CAROUSEL',
        children: childIds.join(','),
        text: text,
        access_token: token,
      }),
    });

    // 3단계: 발행 (threads_publish)
  }
});
```

> ⚠ 캐러셀 이미지 처리에는 시간이 소요됨 (이미지당 2초 대기 + 발행 전 8초 대기)
> ⚠ Threads API 캐러셀은 최대 20장까지 지원

**적용 방법**: Python 패치 스크립트를 SCP로 전송 후 실행

```bash
scp -i [SSH키] patch-threads.py ubuntu@[서버호스트]:/tmp/
ssh -i [SSH키] ubuntu@[서버호스트] "
  cp /opt/bridge/server.js /opt/bridge/server.js.bak
  python3 /tmp/patch-threads.py
  pm2 restart bridge-server
"
```

### 2. n8n 콘텐츠생성 워크플로우 업데이트

```
Webhook (POST /webhook/content-generate, Outgoing Webhook 방식)
 ├→ 즉시응답 ("⏳ 콘텐츠 생성 중...")
 └→ 포스트조회 (GET /api/v4/posts/{post_id}, 인증: HTTP Header Auth)
     └→ 입력처리 (Code: this.helpers.httpRequest)
         │  - "!콘텐츠생성 [유형] 상품명" 파싱
         │  - file_ids 추출 → 각 파일 미디어 서버 업로드
         │  - imageUrls 배열 생성
         └→ Claude브릿지호출 (HTTP Request: POST /generate/sns)
             │  ⚠ 이미지는 Claude에 전달하지 않음 (Threads는 텍스트/이미지 분리)
             ├→ 포맷팅 → Mattermost채널전송
             └→ 대시보드페이로드 (image_url: 쉼표 구분 URL) → 대시보드저장
```

핵심: `입력처리` Code 노드에서 `this.helpers.httpRequest`로 이미지를 미디어 서버에 업로드하고, `imageUrls` 배열을 생성합니다. 대시보드 저장 시 `image_url` 필드에 쉼표 구분으로 저장합니다. Claude 브릿지에는 텍스트 생성 요청만 보냅니다.

### 3. n8n Threads 자동발행 워크플로우 업데이트

```
Webhook (POST /webhook/content-approve)
 └→ Threads텍스트조립+발행 (Code: this.helpers.httpRequest)
     │  - image_url 쉼표 구분 → imageUrls 배열 변환
     │  - 브릿지 서버 /publish/threads 직접 호출
     │  - 이미지 1장: imageUrl 전달 / 2장+: imageUrls 전달
     └→ 대시보드상태업데이트 (published / publish_failed)
```

> ⚠ n8n `jsonBody`에서 `JSON.stringify()` 표현식이 평가되지 않는 버그 있음
> → HTTP Request 노드 대신 Code 노드에서 `this.helpers.httpRequest`로 직접 호출하여 해결

### 4. 대시보드 UI 업데이트

#### 4-1. 상세 페이지: 여러 이미지 표시 (`detail.ejs`)

```html
<!-- 변경 전: 단일 이미지 -->
<img src="<%= content.image_url %>" alt="첨부 이미지">

<!-- 변경 후: 쉼표 구분 → 여러 이미지 가로 배치 -->
<label>첨부 이미지 (<%= content.image_url.split(",").filter(u => u.trim()).length %>개)</label>
<div class="flex flex-wrap gap-3 justify-center">
  <% content.image_url.split(",").filter(u => u.trim()).forEach(function(url, i) { %>
    <img src="<%= url.trim() %>" alt="첨부 이미지 <%= i+1 %>"
         class="max-h-64 rounded-lg border border-gray-200 object-cover">
  <% }); %>
</div>
```

#### 4-2. 리스트 페이지: 썸네일 엑박 수정 (`index.ejs`)

```html
<!-- 변경 전: 쉼표 구분 URL 그대로 → 엑박 -->
<img src="<%= c.image_url %>" alt="">

<!-- 변경 후: 첫 번째 이미지만 썸네일로 표시 -->
<img src="<%= c.image_url.split(',')[0].trim() %>" alt="">
```

적용 후 `pm2 restart dashboard`

### 5. Mattermost 설정 변경

```js
// 기존 /콘텐츠생성 슬래시 커맨드 삭제
const cmds = await mmReq('GET', `/commands?team_id=${teamId}`);
const contentCmd = cmds.data.find(c => c.trigger === '콘텐츠생성');
if (contentCmd) await mmReq('DELETE', `/commands/${contentCmd.id}`);

// 기존 /이미지콘텐츠생성 슬래시 커맨드도 삭제 (통합됨)
const imgCmd = cmds.data.find(c => c.trigger === '이미지콘텐츠생성');
if (imgCmd) await mmReq('DELETE', `/commands/${imgCmd.id}`);

// Outgoing Webhook 생성
await mmReq('POST', '/hooks/outgoing', {
  team_id: teamId,
  display_name: '콘텐츠 생성',
  description: 'Claude AI로 SNS 콘텐츠 생성 (이미지 첨부 지원)',
  trigger_words: ['!콘텐츠생성'],
  trigger_when: 0,
  callback_urls: ['https://n8n.도메인/webhook/content-generate'],
  content_type: 'application/json',
});
```

---

## 완료 후 확인

- [ ] 브릿지 서버: `pm2 logs bridge-server` → "POST /publish/threads (TEXT + IMAGE + CAROUSEL)" 확인
- [ ] `!콘텐츠생성 테스트상품` → 텍스트 콘텐츠 생성 확인
- [ ] `!콘텐츠생성 테스트상품` + 이미지 2장 첨부 → 생성 + 이미지 포함 확인
- [ ] 대시보드 목록에서 첫 번째 이미지 썸네일 표시 확인 (엑박 없음)
- [ ] 대시보드 상세에서 "첨부 이미지 (2개)" 가로 배치 표시 확인
- [ ] 대시보드에서 승인 → Threads에 캐러셀로 발행 확인
- [ ] 이미지 없는 콘텐츠 승인 → Threads에 텍스트만 발행 확인

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| `jsonBody`에서 `JSON.stringify()` 평가 안 됨 | n8n v2 `specifyBody: 'json'` + `jsonBody`에서 복잡한 표현식 미평가 | Code 노드에서 `this.helpers.httpRequest`로 직접 호출 |
| Code 노드에서 `fetch is not defined` | n8n Code 노드에서 `fetch` API 미지원 | `this.helpers.httpRequest` 사용 (n8n 공식 방법) |
| 대시보드 리스트 이미지 엑박 | `image_url`에 쉼표 구분 URL이 저장되는데 `<img src>`에 그대로 전달 | `split(",")[0].trim()`으로 첫 번째 이미지만 썸네일 표시 |
| 대시보드 상세 이미지 깨짐 | `image_url`에 쉼표 구분 URL이 저장되는데 `<img src>`에 그대로 전달 | `split(",")`으로 분리하여 각각 `<img>` 렌더링 |
| 슬래시 커맨드에 이미지 첨부 불가 | Mattermost 슬래시 커맨드는 파일 첨부 미지원 | Outgoing Webhook 방식(`!콘텐츠생성`)으로 변경 |
| 캐러셀 발행 느림 (30초+) | Threads API가 각 이미지 처리에 시간 소요 (이미지당 2초 + 발행 전 8초) | 정상 동작, 타임아웃을 120초로 설정 |
| 승인 후 "승인완료"에서 멈춤 | Threads 발행 워크플로우 에러 (jsonBody 표현식 문제) | Code 노드에서 this.helpers.httpRequest로 직접 호출 |
| 워크플로 활성화 실패 | n8n v2 활성화 버그 | DB 직접 수정 후 n8n 재시작 |

---

## n8n v2 활성화 버그 해결

```bash
ssh -i [SSH키] ubuntu@[서버호스트] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"UPDATE workflow_entity SET active=true, \\\"activeVersionId\\\"=\\\"versionId\\\"
     WHERE name IN ('Mattermost 콘텐츠생성', 'Threads 자동발행');\"
  cd /opt/n8n && docker compose restart n8n
"
```

---

## 이미지 저장 및 발행 구조

```
Mattermost 파일 첨부 + !콘텐츠생성
 │
 └→ n8n 입력처리 (Code: this.helpers.httpRequest)
     │  파일 다운로드 (Mattermost API) → 미디어 서버 업로드
     │
     └→ /opt/media/uploads/ 에 저장
         │
         └→ 공개 URL: https://media.도메인/files/timestamp-hash.jpg
              │
              ├→ 대시보드 DB: image_url 컬럼 (쉼표 구분)
              │   "https://media.도메인/files/a.jpg,https://media.도메인/files/b.jpg"
              │
              ├→ 대시보드 목록: split(",")[0] → 첫 번째 이미지 썸네일
              ├→ 대시보드 상세: split(",") → 모든 이미지 가로 배치
              │
              └→ Threads 발행 (승인 시)
                  │  image_url 쉼표 구분 → imageUrls 배열 변환
                  │
                  ├→ 1장: IMAGE 타입
                  │   POST /threads { media_type: "IMAGE", image_url: url, text: text }
                  │   → POST /threads_publish
                  │
                  └→ 2장+: CAROUSEL 타입
                      ① 각 이미지 개별 컨테이너 생성
                         POST /threads { media_type: "IMAGE", image_url: url, is_carousel_item: true }
                         → childId 수집 (이미지당 2초 대기)
                      ② 캐러셀 컨테이너 생성
                         POST /threads { media_type: "CAROUSEL", children: "id1,id2,...", text: text }
                         → 8초 대기
                      ③ 발행
                         POST /threads_publish { creation_id: carouselId }
```

---

## 설계 결정 기록

| 결정 | 이유 |
|------|------|
| Threads 콘텐츠에 Claude Vision 미사용 | Threads는 텍스트와 이미지가 분리됨. 텍스트 중간에 이미지 삽입 불가 → 이미지 분석이 무의미 |
| `image_url` DB 컬럼에 쉼표 구분 저장 | DB 스키마 변경 없이 여러 이미지 지원. 기존 단일 이미지 하위호환 유지 |
| 브릿지 서버에서 `imageUrl` + `imageUrls` 동시 지원 | 기존 단일 이미지 워크플로우(대시보드 재발행 등)와 하위호환 |
| n8n Threads 발행에서 HTTP Request 대신 Code 사용 | `jsonBody` 표현식 평가 버그 우회 |
