# [실행 가이드 09] 대시보드 삭제 기능 + 페이지네이션

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `06` 작업 완료 후 진행하세요.

---

## 개요

대시보드에 2가지 기능을 추가합니다.

**1. 삭제 기능**
- 상세 페이지: 하단 "삭제" 버튼 (개별 삭제)
- 목록 페이지: 체크박스 선택 → 상단 "선택 삭제" 바 (일괄 삭제)

**2. 페이지네이션**
- 목록 하단에 페이지 번호 + 이전/다음 버튼
- 페이지당 표시 개수 선택: 10 / 20 / 30 / 40 / 50 / 100 (기본 20)
- 총 건수 표시

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

### 1단계: app.js — 삭제 라우트 + 페이지네이션 쿼리 추가

**변경 1 — GET / (목록) 라우트에 페이지네이션 추가:**

```js
// GET / - 목록
app.get('/', (req, res) => {
  const status = ['pending','approved','rejected','published','publish_failed'].includes(req.query.status)
    ? req.query.status : 'pending';
  const perPage = [10, 20, 30, 40, 50, 100].includes(Number(req.query.per)) ? Number(req.query.per) : 20;
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const total = db.prepare('SELECT COUNT(*) AS c FROM contents WHERE status = ?').get(status).c;
  const totalPages = Math.max(1, Math.ceil(total / perPage));
  const offset = (page - 1) * perPage;
  const contents = db.prepare(
    'SELECT * FROM contents WHERE status = ? ORDER BY created_at DESC LIMIT ? OFFSET ?'
  ).all(status, perPage, offset);
  res.render('index', { contents, status, counts: counts(), page, totalPages, perPage, total });
});
```

쿼리 파라미터:
- `per` — 페이지당 개수 (10/20/30/40/50/100, 기본 20)
- `page` — 현재 페이지 (기본 1)

**변경 2 — 삭제 라우트 추가 (reject 라우트 뒤에 삽입):**

```js
// POST /contents/:id/delete - 개별 삭제
app.post('/contents/:id/delete', (req, res) => {
  const content = db.prepare('SELECT * FROM contents WHERE id = ?').get(req.params.id);
  if (!content) return res.status(404).send('Not found');
  const status = content.status;
  db.prepare('DELETE FROM contents WHERE id = ?').run(req.params.id);
  console.log('[delete] content ' + req.params.id + ' deleted');
  res.redirect('/?status=' + status);
});

// POST /contents/bulk-delete - 일괄 삭제
app.post('/contents/bulk-delete', (req, res) => {
  const ids = req.body.ids;
  const status = req.body.status || 'pending';
  if (!ids || !ids.length) return res.redirect('/?status=' + status);
  const placeholders = ids.map(() => '?').join(',');
  const result = db.prepare('DELETE FROM contents WHERE id IN (' + placeholders + ')').run(...ids.map(Number));
  console.log('[bulk-delete] deleted ' + result.changes + ' contents');
  res.redirect('/?status=' + status);
});
```

---

### 2단계: index.ejs — 체크박스 + 선택 삭제 바 + 페이지네이션 UI

**변경 1 — 각 목록 항목에 체크박스 추가:**

기존 `<a>` 태그를 `<div>` + 체크박스 + `<a>` 구조로 변경:

```html
<form id="bulk-form" action="/contents/bulk-delete" method="POST">
  <input type="hidden" name="status" value="<%= status %>">

  <!-- 선택 삭제 바 (체크 시 표시) -->
  <div id="bulk-bar" class="hidden mb-4 flex items-center gap-3 bg-red-50 border border-red-200 rounded-xl px-5 py-3">
    <span id="bulk-count" class="text-sm font-medium text-red-700">0개 선택</span>
    <button type="submit" onclick="return confirm('선택한 콘텐츠를 삭제하시겠습니까?')"
            class="ml-auto px-4 py-1.5 bg-red-500 hover:bg-red-600 text-white text-sm font-medium rounded-lg transition">
      선택 삭제
    </button>
    <button type="button" onclick="clearSelection()"
            class="px-4 py-1.5 bg-white hover:bg-gray-50 text-gray-600 text-sm font-medium rounded-lg border transition">
      취소
    </button>
  </div>

  <!-- 목록 항목 -->
  <div class="space-y-3">
    <% contents.forEach(function(c) { %>
      <div class="flex items-center gap-3 bg-white rounded-xl border ...">
        <input type="checkbox" name="ids" value="<%= c.id %>" onchange="updateBulkBar()"
               class="w-4 h-4 text-red-500 border-gray-300 rounded shrink-0 cursor-pointer">
        <a href="/contents/<%= c.id %>" class="flex-1 ...">
          <!-- 기존 목록 항목 내용 -->
        </a>
      </div>
    <% }); %>
  </div>
</form>
```

**변경 2 — 페이지네이션 UI (목록 하단):**

```html
<div class="mt-6 flex items-center justify-between">
  <!-- 왼쪽: 페이지당 개수 선택 + 총 건수 -->
  <div class="flex items-center gap-2 text-sm text-gray-500">
    <span>페이지당</span>
    <select onchange="location.href='/?status=<%= status %>&per=' + this.value"
            class="border border-gray-300 rounded-lg px-2 py-1 text-sm ...">
      <% [10, 20, 30, 40, 50, 100].forEach(function(n) { %>
        <option value="<%= n %>" <%- n === perPage ? 'selected' : '' %>><%= n %>개</option>
      <% }); %>
    </select>
    <span class="ml-2 text-gray-400">총 <%= total %>건</span>
  </div>

  <!-- 오른쪽: 페이지 번호 -->
  <div class="flex items-center gap-1">
    <% if (page > 1) { %>
      <a href="/?status=<%= status %>&per=<%= perPage %>&page=<%= page - 1 %>"
         class="px-3 py-1.5 text-sm border ...">« 이전</a>
    <% } %>

    <% var startPage = Math.max(1, page - 2); var endPage = Math.min(totalPages, page + 2); %>
    <% for (var p = startPage; p <= endPage; p++) { %>
      <a href="/?status=<%= status %>&per=<%= perPage %>&page=<%= p %>"
         class="px-3 py-1.5 text-sm rounded-lg <%- p === page ? 'bg-indigo-600 text-white' : 'border ...' %>"><%= p %></a>
    <% } %>

    <% if (page < totalPages) { %>
      <a href="/?status=<%= status %>&per=<%= perPage %>&page=<%= page + 1 %>"
         class="px-3 py-1.5 text-sm border ...">다음 »</a>
    <% } %>
  </div>
</div>
```

**변경 3 — JavaScript (체크박스 제어):**

```js
function updateBulkBar() {
  var checked = document.querySelectorAll('input[name="ids"]:checked');
  var bar = document.getElementById('bulk-bar');
  var count = document.getElementById('bulk-count');
  if (checked.length > 0) {
    bar.classList.remove('hidden');
    count.textContent = checked.length + '개 선택';
  } else {
    bar.classList.add('hidden');
  }
}

function clearSelection() {
  document.querySelectorAll('input[name="ids"]:checked').forEach(function(cb) {
    cb.checked = false;
  });
  updateBulkBar();
}
```

---

### 3단계: detail.ejs — 개별 삭제 버튼

승인/거절 버튼 영역 아래에 추가:

```html
<!-- 삭제 버튼 -->
<div class="mt-3 flex justify-end">
  <form action="/contents/<%= content.id %>/delete" method="POST"
        onsubmit="return confirm('이 콘텐츠를 삭제하시겠습니까? 복구할 수 없습니다.')">
    <button type="submit"
            class="px-4 py-2 text-sm text-gray-400 hover:text-red-500 hover:bg-red-50 rounded-lg transition">
      삭제
    </button>
  </form>
</div>
```

> 삭제 버튼은 모든 상태(대기/승인/발행/거절)에서 표시됩니다.
> 눌렀을 때 confirm 확인 후 삭제 → 해당 상태의 목록 페이지로 이동.

---

### 4단계: 배포

```bash
# 파일 업로드 (3개)
scp -i [SSH키] app.js ubuntu@[서버IP]:/opt/dashboard/app.js
scp -i [SSH키] index.ejs ubuntu@[서버IP]:/opt/dashboard/views/index.ejs
scp -i [SSH키] detail.ejs ubuntu@[서버IP]:/opt/dashboard/views/detail.ejs

# PM2 재시작
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart dashboard"
```

---

## 전체 수정 요약

| 파일 | 수정 내용 |
|------|-----------|
| `/opt/dashboard/app.js` | GET / 에 페이지네이션(per, page) 추가, POST /contents/:id/delete + POST /contents/bulk-delete 라우트 추가 |
| `/opt/dashboard/views/index.ejs` | 체크박스 + 선택 삭제 바 + 페이지네이션 UI (per page select + 페이지 번호) |
| `/opt/dashboard/views/detail.ejs` | 하단 "삭제" 버튼 추가 |

---

## 완료 후 확인

1. **목록 페이지**
   - 각 항목 왼쪽에 체크박스 표시
   - 체크하면 상단에 빨간 "N개 선택 / 선택 삭제 / 취소" 바 표시
   - 하단에 "페이지당 [select] 총 N건" + 페이지 번호 표시
   - select 변경 시 해당 개수로 페이지 갱신

2. **상세 페이지**
   - 하단에 회색 "삭제" 텍스트 버튼
   - 클릭 → confirm → 삭제 후 목록으로 이동

3. **페이지네이션**
   - `/?status=pending&per=10&page=2` 형태의 URL로 동작
   - per 기본값 20, page 기본값 1

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 선택 삭제 후 빈 페이지 | 마지막 페이지의 항목을 모두 삭제 | 삭제 후 해당 상태의 1페이지로 리다이렉트됨 — 정상 |
| per page 변경 시 page가 1로 초기화 | select의 onchange에 page 파라미터 미포함 | 의도된 동작 — per 변경 시 1페이지부터 보는 것이 자연스러움 |
| 체크박스가 페이지 이동 후 초기화 | form은 현재 페이지 항목만 포함 | 정상 — 다른 페이지의 항목은 별도로 선택 |
