# [실행 가이드 10] 대시보드 Threads 재발행 기능 + 발행실패 탭

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `06` 작업 완료 후 진행하세요.

---

## 개요

Threads API 발행이 실패한 콘텐츠를 대시보드에서 **재발행**할 수 있도록 합니다.

**기존 문제:**
- 발행 실패 시 `publish_failed` 상태로 남고 복구 방법 없음
- 승인(`approved`) 상태에서 발행이 안 된 채 멈춘 건도 재처리 불가
- 발행실패 콘텐츠를 모아볼 탭이 없음

**개선 후:**
- 상세 페이지에 "재발행 → Threads" 버튼 (발행실패/승인완료 상태)
- 이전 실패 사유 표시
- 목록에 "발행실패" 탭 추가 (주황색)

---

## 사전 조건

- [ ] 06번 작업 완료 (대시보드 가동 중)
- [ ] 서버 SSH 접속 가능

---

## Claude에게 전달할 정보

```
서버 고정 IP         :
SSH 키 경로          :        (예: C:\Users\...\lightsail_key.pem)
```

---

## Claude 실행 내용 (참고용)

### 1단계: app.js — 재발행 라우트 + counts에 publish_failed 추가

**변경 1 — counts 함수에 `publish_failed` 추가:**

```js
function counts() {
  return {
    pending:        db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='pending'").get().c,
    approved:       db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='approved'").get().c,
    rejected:       db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='rejected'").get().c,
    published:      db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='published'").get().c,
    publish_failed: db.prepare("SELECT COUNT(*) AS c FROM contents WHERE status='publish_failed'").get().c,
  };
}
```

**변경 2 — 재발행 라우트 추가 (delete 라우트 앞에 삽입):**

```js
// POST /contents/:id/republish - 재발행 (발행실패/승인완료 → 다시 Threads 발행)
app.post('/contents/:id/republish', async (req, res) => {
  const content = db.prepare('SELECT * FROM contents WHERE id = ?').get(req.params.id);
  if (!content) return res.status(404).send('Not found');

  // 상태를 approved로 리셋 (발행 중 상태)
  db.prepare("UPDATE contents SET status='approved', publish_message=NULL, updated_at=CURRENT_TIMESTAMP WHERE id=?")
    .run(req.params.id);

  try {
    const webhookUrl = process.env.N8N_APPROVE_WEBHOOK ||
      'https://n8n.bestrealinfo.com/webhook/content-approve';
    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        id:        content.id,
        title:     content.title,
        body:      content.body,
        hashtags:  content.hashtags,
        product:   content.product,
        requester: content.requester,
        platform:  content.platform,
        image_url: content.image_url || null,
      }),
    });
    console.log('[republish] content ' + content.id + ' -> n8n webhook OK');
  } catch (err) {
    console.error('[republish] webhook error:', err.message);
  }

  res.redirect('/contents/' + req.params.id);
});
```

> 재발행 동작: 상태를 `approved`로 리셋 → 승인 웹훅을 다시 전송 → n8n "Threads 자동발행" 워크플로가 재실행 → 결과에 따라 `published` 또는 `publish_failed`로 업데이트

---

### 2단계: index.ejs — 발행실패 탭 추가

발행완료 탭과 거절 탭 사이에 추가:

```html
<a href="/?status=publish_failed"
   class="px-5 py-2 rounded-full text-sm font-medium transition
          <%- status === 'publish_failed' ? 'bg-orange-500 text-white shadow' : 'bg-white text-gray-600 border hover:bg-gray-50' %>">
  발행실패 <span class="ml-1 font-bold"><%= counts.publish_failed %></span>
</a>
```

탭 순서: `승인대기 | 승인완료 | 발행완료 | 발행실패 | 거절`

---

### 3단계: detail.ejs — 재발행 버튼

기존 "이미 처리된 콘텐츠입니다" 영역을 상태별로 분기:

```html
<% if (content.status === 'pending') { %>
  <!-- 기존 승인/거절 버튼 -->

<% } else if (content.status === 'publish_failed' || content.status === 'approved') { %>
  <!-- 재발행 버튼 -->
  <div class="flex gap-3 mt-5">
    <form action="/contents/<%= content.id %>/republish" method="POST" class="flex-1"
          onsubmit="return handleSubmit(this, 'approve')">
      <button type="submit"
              class="w-full py-3 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold rounded-xl transition shadow">
        재발행 → Threads<%= content.image_url ? ' (이미지+텍스트)' : '' %>
      </button>
    </form>
  </div>
  <% if (content.publish_message) { %>
    <div class="mt-3 p-3 bg-orange-50 border border-orange-200 rounded-lg text-sm text-orange-700">
      이전 발행 실패: <%= content.publish_message %>
    </div>
  <% } %>

<% } else { %>
  <!-- 발행완료 등 -->
  <div class="mt-5 p-4 bg-gray-50 border border-gray-200 rounded-xl text-center text-sm text-gray-500">
    이미 처리된 콘텐츠입니다.
  </div>
<% } %>
```

> 재발행 버튼 클릭 시 08번 가이드의 딤+스피너 오버레이가 동일하게 동작합니다.
> (`onsubmit="return handleSubmit(this, 'approve')"` → "Threads 발행 중..." 표시)

---

### 4단계: 배포

```bash
scp -i [SSH키] app.js ubuntu@[서버IP]:/opt/dashboard/app.js
scp -i [SSH키] index.ejs ubuntu@[서버IP]:/opt/dashboard/views/index.ejs
scp -i [SSH키] detail.ejs ubuntu@[서버IP]:/opt/dashboard/views/detail.ejs

ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart dashboard"
```

---

## 동작 흐름

```
[최초 승인]
 → Threads 발행 시도
 → 실패 시 status='publish_failed', publish_message='실패 사유'

[재발행]
 → 대시보드 "발행실패" 탭에서 콘텐츠 클릭
 → 상세 페이지에서 실패 사유 확인
 → "재발행 → Threads" 클릭
 → status='approved'로 리셋 + 승인 웹훅 재전송
 → n8n "Threads 자동발행" 워크플로 재실행
 → 성공: status='published' / 실패: status='publish_failed' (재시도 가능)
```

---

## 전체 수정 요약

| 파일 | 수정 내용 |
|------|-----------|
| `/opt/dashboard/app.js` | counts에 `publish_failed` 추가, POST `/contents/:id/republish` 라우트 추가 |
| `/opt/dashboard/views/index.ejs` | "발행실패" 탭 추가 (주황색, 발행완료와 거절 사이) |
| `/opt/dashboard/views/detail.ejs` | `publish_failed`/`approved` 상태에서 "재발행 → Threads" 버튼 + 이전 실패 사유 표시 |

---

## 완료 후 확인

1. 대시보드 목록에 "발행실패" 주황색 탭 표시 + 건수
2. 발행실패 콘텐츠 상세 → "재발행 → Threads" 버튼 + 실패 사유
3. 재발행 클릭 → 딤+스피너 → 상세 페이지로 복귀
4. 성공 시 "발행완료" 탭으로 이동, 실패 시 "발행실패"에 재표시

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 재발행 후 상태가 계속 `approved` | n8n "Threads 자동발행" 워크플로가 상태 업데이트 API를 못 호출 | n8n 워크플로 활성화 상태 확인 + Bridge 서버 로그 확인 |
| 재발행 버튼이 안 보임 | 상태가 `published`이면 표시 안 됨 (정상) | 이미 발행 완료된 콘텐츠는 재발행 불필요 |
| Threads 토큰 만료로 계속 실패 | Threads access_token 유효기간 만료 | Bridge 서버 환경변수 `THREADS_ACCESS_TOKEN` 갱신 후 `pm2 restart bridge-server` |
