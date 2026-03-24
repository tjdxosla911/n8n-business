# [실행 가이드 11] Mattermost → Trello 이슈 자동 등록 (이미지 첨부 지원)

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~03`, `12~13` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01번: Lightsail + n8n 설치 완료
- [ ] 02번: n8n SSL 완료 (https://n8n.도메인)
- [ ] 03번: Mattermost 설치 완료
- [ ] 12번: 미디어 파일 서버 구축 완료 (https://media.도메인)
- [ ] 13번: 미디어 워크플로우 완료 (Personal Access Token + HTTP Header Auth Credential 등록됨)
- [ ] Trello 계정 생성 + 보드 생성 완료
- [ ] Trello API Key + Token 발급 완료

### Trello API Key / Token 발급 방법

1. https://trello.com/power-ups/admin 접속 → "New" 클릭 → Power-Up 생성
2. 생성된 Power-Up 클릭 → "API Key" 탭 → API Key 확인
3. 같은 페이지에서 "Token" 링크 클릭 → 권한 승인 → Token 발급
4. 발급된 API Key와 Token을 아래 시트에 기입

### Trello Board ID 확인 방법

브라우저에서 Trello 보드를 열면 URL이 `https://trello.com/b/hixnVDR4/보드이름` 형태입니다.
`hixnVDR4` 부분이 Board ID입니다.

### Trello List ID 확인 방법

아래 URL을 브라우저에서 열면 리스트 목록이 JSON으로 표시됩니다:
```
https://api.trello.com/1/boards/{BOARD_ID}/lists?key={API_KEY}&token={TOKEN}
```
이슈를 등록할 리스트(예: BackLog)의 `id` 값을 아래 시트에 기입하세요.

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

Trello API Key       :
Trello API Token     :
Trello Board ID      :        (예: hixnVDR4)
Trello List ID       :        (예: 69c0c0a533e69969460ac2c7, BackLog 리스트)
```

---

## 기능 설명

### 사용법

Mattermost 아무 채널에서 (`!이슈` — 슬래시 커맨드가 아닌 Outgoing Webhook 방식):

```
!이슈 제목                              ← 텍스트만
!이슈 제목 | 설명                       ← 텍스트 + 설명
!이슈 제목 | 설명 + [이미지 첨부]       ← 텍스트 + 설명 + 이미지
```

- 이미지 여러 장 첨부 가능
- 이미지는 미디어 서버에 저장 → Trello 카드에 인라인 표시

> ⚠ Mattermost 슬래시 커맨드(`/이슈`)는 파일 첨부를 지원하지 않음
> → Outgoing Webhook 방식(`!이슈`)으로 구현하여 이미지 첨부 가능

### 예시

| 입력 | Trello 카드 제목 | Description |
|------|-----------------|-------------|
| `!이슈 로그인 버그` | 로그인 버그 | 등록일시 + 등록자 |
| `!이슈 로그인 버그 \| 모바일 CSS 깨짐` | 로그인 버그 | 설명 + 등록일시 + 등록자 |
| `!이슈 로그인 버그 \| CSS 깨짐 + 📎이미지3장` | 로그인 버그 | 설명 + 등록정보 + 이미지 인라인 표시 |

### Trello 카드 Description (마크다운)

```markdown
모바일 CSS 깨짐

---

**등록일시:** 2026. 3. 23. 오후 4:11
**등록자:** @admin

---

![스크린샷 2026-03-23.png](https://media.도메인/files/...png)

![스크린샷 2026-03-23.png](https://media.도메인/files/...png)
```

- Trello에서 `![](url)` 마크다운이 렌더링되어 이미지가 카드 내에 직접 표시됨
- 설명이 없으면 등록정보 + 이미지만 표시
- 이미지가 없으면 텍스트만 표시

---

## Claude 실행 내용 (참고용)

### 실행 순서

1. n8n에 Trello Credential 등록 (없으면 생성)
2. 13번에서 등록한 Mattermost File Access Credential + Personal Access Token 재사용
3. n8n 워크플로우 생성 (Outgoing Webhook → 포스트조회 → 파일처리 → Trello 카드 생성)
4. 워크플로우 활성화 (API + DB 직접 수정)
5. Mattermost Outgoing Webhook 생성 (trigger: `!이슈`)

### 워크플로우 구조

```
Mattermost !이슈 [제목 | 설명] + [이미지 첨부]
 └→ Outgoing Webhook → n8n Webhook (POST /webhook/mattermost-issue)
     ├→ 즉시응답: "⏳ 이슈 등록 중..."
     └→ 포스트조회 (GET /api/v4/posts/{post_id}, 인증: HTTP Header Auth)
         └→ 이슈처리 (Code: this.helpers.httpRequest)
             │  - "!이슈 제목 | 설명" 파싱
             │  - file_ids 추출 → 각 파일 미디어 서버 업로드
             │  - Description 조립 (마크다운: 설명 + 등록정보 + 이미지 인라인)
             │
             └→ 에러체크 (IF: error === true)
                 ├→ [에러] → 에러알림 (Mattermost: 사용법 안내)
                 └→ [정상] → Trello카드생성 (HTTP Request → Trello API)
                              └→ 결과전송 (Mattermost: "✅ 이슈가 등록됐습니다")
```

### 핵심 노드: 이슈처리 (Code)

```js
// ⚠ fetch 사용 불가 → this.helpers.httpRequest 사용
// ⚠ 여러 파일은 단일 Code 노드에서 루프로 처리 (멀티 아이템 분기 불안정)

const post = $input.first().json;
const fileIds = post.file_ids || [];
const webhookData = $('Webhook').first().json.body || {};
const authHeader = 'Bearer PERSONAL_ACCESS_TOKEN';

// "!이슈 제목 | 설명" 파싱
const rawText = (webhookData.text || '').trim();
const withoutTrigger = rawText.replace(/^!이슈\s*/, '').trim();
const parts = withoutTrigger.split('|').map(s => s.trim());
const title = parts[0];
const userDesc = parts.length > 1 ? parts.slice(1).join('|').trim() : '';

// 파일 처리 (루프)
const fileUrls = [];
for (const fid of fileIds) {
  const info = await this.helpers.httpRequest({
    method: 'GET',
    url: 'MATTERMOST_URL/api/v4/files/' + fid + '/info',
    headers: { Authorization: authHeader },
    json: true,
  });
  const upload = await this.helpers.httpRequest({
    method: 'POST',
    url: 'MEDIA_URL/upload-from-url',
    headers: { 'Content-Type': 'application/json' },
    body: { url: 'MATTERMOST_URL/api/v4/files/' + fid, filename: info.name, authorization: authHeader },
    json: true,
  });
  fileUrls.push({ name: info.name, url: upload.url });
}

// Trello Description (마크다운) 조립
const descLines = [];
if (userDesc) descLines.push(userDesc);
descLines.push('', '---', '', '**등록일시:** ' + now, '**등록자:** @' + userName);
if (fileUrls.length > 0) {
  descLines.push('', '---', '');
  fileUrls.forEach(f => { descLines.push('![' + f.name + '](' + f.url + ')', ''); });
}

return [{ json: { title, desc: descLines.join('\n'), channelId, message } }];
```

> ⚠ `PERSONAL_ACCESS_TOKEN`, `MATTERMOST_URL`, `MEDIA_URL`은 실행 시 동적으로 대체됩니다.

---

## 완료 후 확인

- [ ] n8n 워크플로 화면에서 "Mattermost 이슈 → Trello" Active 상태 확인
- [ ] `!이슈 테스트` → "✅ 이슈가 등록됐습니다: 테스트" 응답 + Trello 카드 생성
- [ ] `!이슈 제목 | 설명` → Description에 설명 + 등록일시 + 등록자 확인
- [ ] `!이슈 제목 | 설명` + 이미지 3장 첨부 → Trello 카드에 이미지 인라인 표시 확인
- [ ] 빈 입력 (`!이슈`만) → 에러 안내 메시지 확인

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| Trello 노드 `additionalFields.desc`가 빈값 | n8n Trello 노드 버그: description 필드가 API에 전달되지 않음 | HTTP Request 노드 + `predefinedCredentialType: trelloApi`로 Trello API 직접 호출 |
| Code 노드에서 `fetch is not defined` | n8n Code 노드에서 `fetch` API 미지원 | `this.helpers.httpRequest` 사용 (n8n 공식 방법) |
| 여러 파일이 1개만 처리됨 | n8n 멀티 아이템 분기 후 합산 불안정 | 단일 Code 노드에서 `this.helpers.httpRequest` 루프로 전체 처리 |
| Mattermost 슬래시 커맨드에 이미지 첨부 불가 | Mattermost 슬래시 커맨드는 파일 첨부 미지원 | Outgoing Webhook 방식(`!이슈`)으로 변경 |
| PATCH active:true → 200이지만 실제 inactive | n8n v2 버그: DB `activeVersionId=null` | 아래 "n8n v2 활성화 버그 해결" 참고 |
| 워크플로 수정(PATCH) 후 웹훅 미등록 | PATCH 시 versionId 변경되나 activeVersionId 미갱신 | DB에서 `activeVersionId = versionId` 업데이트 후 n8n 재시작 |

---

## n8n v2 활성화 버그 해결

### 증상 확인

```bash
ssh -i [SSH키] ubuntu@[서버호스트] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"SELECT name, active,
      \\\"activeVersionId\\\",
      \\\"versionId\\\",
      (\\\"activeVersionId\\\" = \\\"versionId\\\") AS versions_match
    FROM workflow_entity WHERE name = 'Mattermost 이슈 → Trello';\"
"
```

### 해결 방법

```bash
ssh -i [SSH키] ubuntu@[서버호스트] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"UPDATE workflow_entity SET active=true, \\\"activeVersionId\\\"=\\\"versionId\\\" WHERE name = 'Mattermost 이슈 → Trello';\"
  cd /opt/n8n && docker compose restart n8n
"
# 재시작 후 12초 대기 → 로그에서 "Activated workflow" 확인
```

> **주의**: 워크플로를 API로 수정(PATCH)할 때마다 이 문제가 재발할 수 있습니다.
> 수정 후 항상 위 스크립트를 실행하거나, n8n UI에서 직접 수정하면 이 문제가 발생하지 않습니다.
