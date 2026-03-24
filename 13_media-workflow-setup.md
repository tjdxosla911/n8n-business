# [실행 가이드 13] Mattermost 파일 업로드 → 미디어서버 자동 저장 워크플로우

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `12번 (미디어 파일 서버)` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01번: Lightsail + n8n 설치 완료
- [ ] 03번: Mattermost 설치 + 봇 토큰 발급 완료
- [ ] 04번: n8n Mattermost Credential 등록 완료
- [ ] 12번: 미디어 파일 서버 구축 완료 (https://media.도메인)

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

서버 SSH 키 경로     :        (예: C:\Users\...\lightsail_key.pem)
서버 호스트          :        (예: n8n.example.com 또는 서버 IP)
```

---

## 기능 설명

### 사용법

Mattermost 아무 채널에서 파일을 첨부하고:

```
!저장                      ← 파일 저장만 (URL 반환)
!저장 트렐로               ← 저장 + Trello BackLog에 카드 생성
!저장 블로그 쓰레드        ← 저장 + 2개 플랫폼에 전달
!저장 전체                 ← 저장 + 트렐로/블로그/쓰레드 전부
```

- 여러 장 첨부 가능 (테스트 완료: 7장 동시 처리)
- 플랫폼 키워드는 조합 자유

### 결과 예시 (7장 + 트렐로)

```
유저: [이미지 7장 첨부] !저장 트렐로
봇:   ⏳ 파일 저장 중...
봇:   ✅ 파일 저장 완료 (7개)

      1. 📎 스크린샷 2026-03-23 153746.png (48.3KB)
         🔗 https://media.도메인/files/1774249348706-...png
      2. 📎 스크린샷 2026-03-23 155115.png (181.0KB)
         🔗 https://media.도메인/files/1774249348706-...png
      ...
      7. 📎 스크린샷 2026-03-23 155058.png (171.5KB)
         🔗 https://media.도메인/files/1774249348939-...png

      📌 전달 대상: 트렐로
```

- 반환된 URL은 영구적이며, Trello/Threads/블로그 어디서든 사용 가능
- 파일 없이 `!저장`만 입력하면 안내 메시지 표시

---

## Claude 실행 내용 (참고용)

### 실행 순서

1. Mattermost config에서 Personal Access Token 기능 활성화
2. Mattermost Personal Access Token 생성 (파일 다운로드 인증용)
3. n8n에 HTTP Header Auth Credential 등록 (토큰 저장)
4. n8n 워크플로우 생성 + 활성화
5. Mattermost Outgoing Webhook 생성

### 1. Mattermost Personal Access Token 활성화

Mattermost config.json에서 활성화 필요:

```bash
ssh -i [SSH키] ubuntu@[서버호스트] 'sudo python3 << PYEOF
import json
path = "/var/lib/docker/volumes/mattermost_mattermost_config/_data/config.json"
with open(path) as f:
    cfg = json.load(f)
cfg["ServiceSettings"]["EnableUserAccessTokens"] = True
with open(path, "w") as f:
    json.dump(cfg, f, indent=2)
print("done")
PYEOF
docker restart mattermost'
```

> ⚠ Mattermost API `PATCH /config/patch`로는 이 설정이 변경되지 않음 → config.json 직접 수정 필요

### 2. 토큰 생성 + n8n Credential 등록

```js
// Mattermost Personal Access Token 생성
const mmLoginRes = await fetch(`${MM_URL}/api/v4/users/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ login_id: MM_USER, password: MM_PASS }),
});
const mmToken = mmLoginRes.headers.get('token');

const me = await mmReq('GET', '/users/me');
const newToken = await mmReq('POST', `/users/${me.data.id}/tokens`, {
  description: 'n8n-media-workflow',
});
const patToken = newToken.data.token;

// n8n HTTP Header Auth Credential 생성
await n8nReq('POST', '/rest/credentials', {
  name: 'Mattermost File Access',
  type: 'httpHeaderAuth',
  data: { name: 'Authorization', value: `Bearer ${patToken}` },
});
```

### 3. 워크플로우 구조

```
Mattermost !저장 [트렐로] [블로그] [쓰레드] (파일 N개 첨부)
 └→ Outgoing Webhook → n8n Webhook (POST /webhook/media-upload)
     ├→ 즉시응답: "⏳ 파일 저장 중..."
     └→ 포스트조회 (GET /api/v4/posts/{post_id}, 인증: Personal Access Token)
         └→ 전체파일처리 (Code: this.helpers.httpRequest로 N개 파일 순차 처리)
             │  - file_ids 추출
             │  - 플랫폼 키워드 파싱 (트렐로/블로그/쓰레드/전체)
             │  - 각 파일: 정보 조회 → 미디어 서버 업로드 → URL 수집
             │  - 결과 메시지 조립
             └→ Mattermost전송 (결과 + URL 목록)
                 └→ 플랫폼분기 (Code: 지정된 플랫폼별 아이템 생성)
                     └→ 트렐로분기 (IF)
                         ├→ [트렐로] → Trello카드생성 (이미지 URL 포함)
                         └→ [기타] → 기타플랫폼 (향후 연동)
```

> ⚠ **n8n Code 노드에서 HTTP 호출 시 `fetch`는 사용 불가** — 반드시 `this.helpers.httpRequest` 사용
> ⚠ n8n의 멀티 아이템 분기 처리는 Aggregate 등으로 합산이 까다로움 — Code 노드 하나에서 루프로 처리하는 것이 안정적

### 4. 워크플로우 핵심 노드 (전체파일처리 Code)

```js
// n8n Code 노드: 전체파일처리
// ⚠ fetch 사용 불가 → this.helpers.httpRequest 사용
// ⚠ 여러 파일을 노드 분기로 처리하면 합산이 안정적이지 않음 → 단일 Code에서 루프 처리

const post = $input.first().json;
const fileIds = post.file_ids || [];
const channelId = post.channel_id || '';
const webhookData = $('Webhook').first().json.body || {};
const userName = webhookData.user_name || 'unknown';
const authHeader = 'Bearer PERSONAL_ACCESS_TOKEN';

// 플랫폼 키워드 파싱
const rawText = (webhookData.text || '').trim();
const words = rawText.split(/\s+/).slice(1).map(w => w.toLowerCase());
const platformMap = {
  '트렐로': 'trello', '블로그': 'blog', '쓰레드': 'threads', '전체': 'all',
};
let platforms = [];
for (const w of words) {
  if (platformMap[w]) {
    if (platformMap[w] === 'all') { platforms = ['trello', 'blog', 'threads']; break; }
    platforms.push(platformMap[w]);
  }
}

// 모든 파일을 순차적으로 처리
const results = [];
for (const fid of fileIds) {
  // 1. 파일 정보 조회
  const info = await this.helpers.httpRequest({
    method: 'GET',
    url: 'MATTERMOST_URL/api/v4/files/' + fid + '/info',
    headers: { Authorization: authHeader },
    json: true,
  });

  // 2. 미디어 서버에 업로드
  const upload = await this.helpers.httpRequest({
    method: 'POST',
    url: 'MEDIA_URL/upload-from-url',
    headers: { 'Content-Type': 'application/json' },
    body: {
      url: 'MATTERMOST_URL/api/v4/files/' + fid,
      filename: info.name,
      authorization: authHeader,
    },
    json: true,
  });

  results.push({ filename: info.name, url: upload.url, size: upload.size });
}

// 결과 메시지 조립 후 반환
return [{ json: { channelId, message, platforms, fileUrls, userName } }];
```

### 5. 세팅 스크립트 (설정 자동화)

세팅 스크립트는 이전 섹션(1~2)의 토큰 생성 + Credential 등록에 이어서:

1. 워크플로우 생성 (위 구조의 전체 노드 + connections)
2. 워크플로우 활성화 (API + DB 직접 수정)
3. Mattermost Outgoing Webhook 생성 (trigger_words: `['!저장']`)

> ⚠ `PERSONAL_ACCESS_TOKEN`, `MATTERMOST_URL`, `MEDIA_URL`, `MM_CRED_ID`는 실행 시 동적으로 대체됩니다.

---

## 워크플로우 구조 상세

```
[Mattermost]                    [n8n]                           [미디어 서버]
 유저가 파일 N개 첨부 +          Webhook 수신
 "!저장 트렐로" 입력              │
     │                           ├→ 즉시응답 (⏳ 파일 저장 중...)
     │                           │
     │  Outgoing Webhook ──────→ └→ 포스트조회 (MM API: GET /posts/{id})
     │                               │
     │                               └→ 전체파일처리 (Code 노드 1개)
     │                                   │  this.helpers.httpRequest 루프:
     │                                   │  파일1: 정보조회 → 업로드 → URL
     │                                   │  파일2: 정보조회 → 업로드 → URL
     │                                   │  파일N: 정보조회 → 업로드 → URL ──→ POST /upload-from-url
     │                                   │                                      (각 파일 다운로드+저장)
     │                                   │
     ←── 결과 메시지 ←── Mattermost전송 ←┘
     "✅ 파일 저장 완료 (7개)             └→ 플랫폼분기
      1. 📎 파일1.png (48.3KB)                ├→ [트렐로] → Trello카드생성
         🔗 https://media.도메인/...          ├→ [블로그] → (향후 연동)
      ...                                    └→ [쓰레드] → (향후 연동)
      📌 전달 대상: 트렐로"
```

### 미디어 서버 /upload-from-url 동작

```
n8n → POST /upload-from-url
       { url: "MM파일URL", filename: "원본명.jpg", authorization: "Bearer TOKEN" }
             │
             ├→ Mattermost API에서 파일 다운로드 (authorization 헤더 포함)
             ├→ /opt/media/uploads/에 고유 파일명으로 저장
             └→ 공개 URL 반환: https://media.도메인/files/1774246538547-abc123.jpg
```

---

## 완료 후 확인

- [ ] n8n 워크플로 화면에서 "Mattermost 파일 → 미디어서버" Active 상태 확인
- [ ] 이미지 1장 + `!저장` → 1개 파일 URL 응답 확인
- [ ] 이미지 여러 장 + `!저장 트렐로` → N개 전부 처리 + Trello 카드 생성 확인
- [ ] `!저장 블로그 쓰레드` → 복수 플랫폼 표시 확인
- [ ] `!저장 전체` → 트렐로/블로그/쓰레드 전부 표시 확인
- [ ] 반환된 URL을 브라우저에서 열어 이미지 표시 확인
- [ ] 파일 없이 `!저장`만 입력 → 에러 안내 메시지 확인

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| Personal Access Token 생성 시 501 에러 | Mattermost config에서 `EnableUserAccessTokens: false` | config.json 직접 수정 후 Mattermost 재시작 (API PATCH로는 변경 안 됨) |
| 파일 다운로드 401 에러 | Personal Access Token 만료 또는 잘못된 토큰 | 토큰 재생성 후 n8n Credential 업데이트 |
| Outgoing Webhook 미작동 | Mattermost 시스템 콘솔에서 Outgoing Webhook 비활성화 상태 | 시스템 콘솔 → 통합 → Outgoing Webhook 활성화 |
| Code 노드에서 `fetch is not defined` | n8n Code 노드에서 `fetch` API 미지원 | `this.helpers.httpRequest` 사용 (n8n 공식 방법) |
| 여러 파일이 1개만 처리됨 | n8n 멀티 아이템 분기 후 Aggregate 합산 불안정 | 단일 Code 노드에서 `this.helpers.httpRequest` 루프로 전체 처리 |
| 워크플로 활성화 실패 | n8n v2 활성화 버그 (activeVersionId = null) | DB 직접 수정 후 n8n 재시작 (11번 가이드 참고) |

---

## n8n v2 활성화 버그 해결

```bash
ssh -i [SSH키] ubuntu@[서버호스트] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"UPDATE workflow_entity SET active=true, \\\"activeVersionId\\\"=\\\"versionId\\\" WHERE name = 'Mattermost 파일 → 미디어서버';\"
  cd /opt/n8n && docker compose restart n8n
"
```

---

## 다른 워크플로우에서 미디어 서버 활용

이 워크플로우로 생성된 URL은 다른 워크플로우에서 바로 사용할 수 있습니다:

| 워크플로우 | 활용 방식 |
|-----------|----------|
| Trello 이슈 등록 (11번) | 카드 description에 이미지 URL 삽입 |
| 콘텐츠 생성 (04번) | SNS 콘텐츠에 이미지 URL 포함 |
| Threads 자동발행 | 이미지 URL로 미디어 첨부 발행 |
| 블로그 자동생성 | 본문에 이미지 URL 삽입 |
