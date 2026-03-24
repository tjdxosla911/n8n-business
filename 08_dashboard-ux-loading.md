# [실행 가이드 08] 대시보드 UX 개선 — 승인/거절 로딩 오버레이

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `06` 작업 완료 후 진행하세요.

---

## 개요

대시보드에서 **승인** 또는 **거절** 버튼을 클릭하면 Threads API 발행까지 수 초가 소요됩니다.
기존에는 버튼 클릭 후 아무런 피드백 없이 화면이 멈춰 있어 진행 상태를 알 수 없었습니다.

이 가이드를 적용하면:
- 버튼 클릭 → **화면 전체 딤(반투명 검정)** + **중앙 스피너** + **상태 텍스트** 표시
- 승인: "Threads 발행 중..."
- 거절: "처리 중..."

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

### 1단계: `/opt/dashboard/views/detail.ejs` 수정

변경 범위: 승인/거절 버튼 영역 + `<script>` 블록만 수정. 나머지(메타 정보, 편집 폼 등)는 변경 없음.

**추가 1 — 전체 화면 딤 + 스피너 오버레이 HTML** (`</main>` 바로 아래):

```html
<!-- 전체 화면 딤 + 스피너 오버레이 -->
<div id="loading-overlay" class="hidden fixed inset-0 z-50 flex flex-col items-center justify-center" style="background:rgba(0,0,0,0.5)">
  <svg class="animate-spin h-12 w-12 text-white mb-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"></path>
  </svg>
  <p id="loading-text" class="text-white text-lg font-medium"></p>
</div>
```

**추가 2 — 승인/거절 form에 `onsubmit` 핸들러 연결**:

```html
<!-- 승인 -->
<form action="/contents/<%= content.id %>/approve" method="POST" class="flex-1"
      onsubmit="return handleSubmit(this, 'approve')">
  <button type="submit"
          class="w-full py-3 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold rounded-xl transition shadow">
    승인 → Threads 발행<%= content.image_url ? ' (이미지+텍스트)' : '' %>
  </button>
</form>

<!-- 거절 -->
<form action="/contents/<%= content.id %>/reject" method="POST" class="flex-1"
      onsubmit="return handleSubmit(this, 'reject')">
  <button type="submit"
          class="w-full py-3 bg-red-50 hover:bg-red-100 text-red-600 font-semibold rounded-xl border border-red-200 transition">
    거절
  </button>
</form>
```

**추가 3 — JavaScript** (`<script>` 블록):

```js
function handleSubmit(form, action) {
  var msg = action === 'approve'
    ? '승인하시겠습니까? Threads에 자동 발행됩니다.'
    : '거절하시겠습니까?';
  if (!confirm(msg)) return false;

  var overlay = document.getElementById('loading-overlay');
  var text = document.getElementById('loading-text');
  text.textContent = action === 'approve' ? 'Threads 발행 중...' : '처리 중...';
  overlay.classList.remove('hidden');

  return true;
}
```

> 기존 `onclick="return confirm(...)"` 인라인 핸들러를 제거하고 `onsubmit` 핸들러로 통합.
> `confirm()` → 확인 클릭 → 딤 오버레이 표시 → form 제출(POST) 순서로 동작.

---

### 2단계: 배포

```bash
# 로컬에서 SCP 업로드
scp -i [SSH키] detail.ejs ubuntu@[서버IP]:/opt/dashboard/views/detail.ejs

# PM2 재시작 (EJS는 매 요청마다 렌더링되므로 재시작 필요 없을 수 있으나, 확실하게)
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart dashboard"
```

---

## 동작 흐름

```
[승인 버튼 클릭]
 → confirm 다이얼로그 "승인하시겠습니까?"
 → 확인 클릭
 → 화면 전체 딤(rgba 0,0,0,0.5)) + 스피너 + "Threads 발행 중..."
 → form POST /contents/:id/approve
 → 서버 처리 (Threads API 호출 ~3~8초)
 → 리다이렉트 /?status=approved
```

---

## 스타일 상세

| 요소 | 스펙 |
|------|------|
| 오버레이 배경 | `rgba(0,0,0,0.5)` — 반투명 검정 |
| 오버레이 z-index | `z-50` (Tailwind) |
| 스피너 | SVG `animate-spin`, 48x48px, 흰색 |
| 텍스트 | `text-lg font-medium text-white` |
| 위치 | `fixed inset-0`, flex 중앙 정렬 |

---

## 완료 후 확인

1. 대시보드에서 승인대기 콘텐츠 진입
2. **승인** 클릭 → confirm → 화면 딤 + 스피너 + "Threads 발행 중..." 확인
3. 발행 완료 후 자동으로 승인 목록 페이지로 이동
4. **거절** 클릭 → confirm → 화면 딤 + "처리 중..." → 거절 목록으로 이동

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 딤 오버레이가 안 나타남 | Tailwind CDN 로드 실패 | 네트워크 확인, `animate-spin` 클래스가 동작하는지 확인 |
| 승인 후 오버레이가 계속 떠 있음 | 서버 응답 지연 또는 Threads API 타임아웃 | 정상 동작 — 서버 응답 후 리다이렉트되면 자동 해제 |
| confirm 취소했는데 오버레이 뜸 | `handleSubmit` 로직 오류 | confirm `false` 시 `return false`로 form 제출 차단 — 오버레이는 confirm 후에만 표시 |
