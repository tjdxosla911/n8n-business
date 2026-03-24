# [실행 가이드 07] 콘텐츠 유형별 프롬프트 시스템 (일상/지식/홍보)

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~06` 작업 완료 후 진행하세요.

---

## 개요

기존에는 `/콘텐츠생성 [상품명]` 으로 홍보 게시물만 생성 가능했습니다.
이 가이드를 적용하면 **유형 파라미터**를 추가하여 3가지 콘텐츠를 생성할 수 있습니다.

| 유형 | 목적 | 톤 | 해시태그 |
|------|------|----|----------|
| 일상 | 공감형 일상 게시물 — 직접적 홍보 없음 | 담백한 사업가 말투 | 5개 |
| 지식 | "나 손해보고 있었나?" 유도 — 정보성 | 전문가 분석 | 7개 |
| 홍보 | 직접 상품 홍보 — 부정의문문+CTA | SNS 마케터 | 10개 |

### 커맨드 사용법

```
/콘텐츠생성 일상 세금계산서 실수
/콘텐츠생성 지식 4대보험 절감
/콘텐츠생성 홍보 전기세환급
/콘텐츠생성 무선이어폰            ← 유형 생략 시 기본값 '홍보'
```

---

## 사전 조건

- [ ] 01~06번 작업 완료 (브릿지 서버 + n8n 워크플로 + 대시보드 가동 중)
- [ ] 서버 SSH 접속 가능

---

## Claude에게 전달할 정보

```
서버 고정 IP         :
SSH 키 경로          :        (예: C:\Users\...\lightsail_key.pem)
n8n URL             :        (예: https://n8n.example.com)
n8n 이메일          :
n8n 비밀번호         :
n8n Basic Auth PW   :        (docker-compose의 N8N_BASIC_AUTH_PASSWORD)
```

---

## Claude 실행 내용 (참고용)

### 1단계: 브릿지 서버 — `/generate/sns` 엔드포인트 수정

변경 범위: `/opt/bridge/server.js` 의 `app.post('/generate/sns', ...)` 핸들러만 수정.
다른 엔드포인트(`/health`, `/publish/threads`, `/download-mattermost-image`)는 변경 없음.

**변경 전 (기존):**
- 단일 프롬프트 (홍보용 SNS 마케팅 전문가)
- `messages` 배열에 프롬프트+상품명을 user 메시지로 전달
- JSON 출력 형식 요청 → 파싱

**변경 후:**
- `type` 파라미터 수신 (기본값: `'홍보'`)
- 유형별 프롬프트 3종을 `system` 메시지로 전달
- `messages`는 `"게시물을 작성해주세요."` 한 줄만
- 응답 포맷(JSON 파싱 + `success/content/raw` 구조)은 기존 그대로 유지

수정된 `/opt/bridge/server.js` — `/generate/sns` 핸들러:

```js
app.post('/generate/sns', async (req, res) => {
  const { product, type = '홍보', platform = '인스타그램', tone = '친근한', target, keywords, options = {} } = req.body;

  if (!product) {
    return res.status(400).json({ error: '상품명(product)이 필요합니다.' });
  }

  var effectiveTone = tone || (options.tone ? options.tone : '친근한');
  var effectiveTarget = target || (options.target ? options.target : '');
  var effectiveKeywords = keywords || (options.keywords ? options.keywords.join(', ') : '');

  const prompts = {
    '일상': '당신은 AI 자동화로 먹고사는 1인 사업가입니다.\n' +
      '수십 개의 중소기업 내부 시스템을 직접 만지면서, 대표들이 모르고 새는 돈의 구조를 누구보다 가까이서 봐온 사람입니다.\n' +
      '아래 주제로 Threads 게시물을 1인칭으로 작성하세요.\n\n' +
      '주제: ' + product + '\n\n' +
      '규칙:\n' +
      '- "나는", "내가", "오늘" 등 1인칭 일상 어투로 시작\n' +
      '- 대표님들이 "맞아, 나도 저랬는데" 하고 찔리는 장면을 그려라\n' +
      '- 광고처럼 보이면 실패. 그냥 오늘 있었던 일처럼 써라\n' +
      '- 마지막 줄은 불편한 질문 하나로 끝내라 (답 주지 마라)\n' +
      '- 분량: 3~5줄\n' +
      '- 해시태그: 핵심 1개만. 본문 내용을 가장 잘 대표하는 단어로.\n' +
      '- CTA 없음',

    '지식': '당신은 AI 자동화로 먹고사는 1인 사업가입니다.\n' +
      '수십 개의 중소기업 내부 시스템을 직접 만지면서, 대표들이 모르고 새는 돈의 구조를 누구보다 가까이서 봐온 사람입니다.\n' +
      '아래 주제로 Threads 지식 게시물을 작성하세요.\n\n' +
      '주제: ' + product + '\n\n' +
      '규칙:\n' +
      '- 첫 줄은 대표들이 불편해질 만한 사실 하나로 시작 (부정의문문 또는 단언)\n' +
      '- "대부분의 사장님들은 이걸 모른다" 류의 프레이밍 사용\n' +
      '- 수치나 구체적 상황 포함 (없으면 "제조업 대표 10명 중 8명" 같은 경험적 표현)\n' +
      '- 해결책은 절반만 줘라. 나머지는 독자가 궁금해하게 남겨라\n' +
      '- 마지막 줄: "이거 모르면 그냥 기부하는 거임" 류의 도발적 마무리\n' +
      '- 분량: 4~6줄\n' +
      '- 해시태그: 핵심 1개만. 본문 내용을 가장 잘 대표하는 단어로.\n' +
      '- CTA 없음',

    '홍보': '당신은 AI 자동화로 먹고사는 1인 사업가입니다.\n' +
      '수십 개의 중소기업 내부 시스템을 직접 만지면서, 대표들이 모르고 새는 돈의 구조를 누구보다 가까이서 봐온 사람입니다.\n' +
      '아래 상품으로 Threads 게시물을 1인칭으로 작성하세요.\n\n' +
      '상품명: ' + product + '\n' +
      (effectiveTone ? '톤: ' + effectiveTone + '\n' : '') +
      (effectiveTarget ? '타겟: ' + effectiveTarget + '\n' : '타겟: 제조업 중소기업 대표\n') +
      (effectiveKeywords ? '키워드: ' + effectiveKeywords + '\n' : '') +
      '\n카피 구조 (반드시 이 순서):\n' +
      '1. 첫 줄: "아직도 [손실 상황] 모르세요?" 또는 "[손실 상황], 당신 회사 얘기 아닌가요?" 형태\n' +
      '2. 본문: 내가 직접 발견한 것처럼 써라. "시스템 들여다보다 보니~", "대표님한테 확인해봤더니~" 식의 1인칭 경험담 구조. 3~4줄\n' +
      '3. 해시태그: 핵심 1개만. 본문 내용을 가장 잘 대표하는 단어로.\n' +
      '4. CTA: "손해 볼 거 없으니까 한 번만 확인해보세요" 류. 저항 없이 클릭하게.'
  };

  var systemPrompt = prompts[type] || prompts['홍보'];

  try {
    var message = await client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 1000,
      system: systemPrompt,
      messages: [{ role: 'user', content: '게시물을 작성해주세요.' }],
    });

    var text = message.content[0].text;
    var jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      var parsed = JSON.parse(jsonMatch[0]);
      return res.json({ success: true, product: product, platform: platform, content: parsed, raw: text });
    }
    return res.json({ success: true, product: product, platform: platform, content: null, raw: text });
  } catch (err) {
    console.error('[ERROR]', err.message);
    return res.status(500).json({ error: err.message });
  }
});
```

배포:

```bash
# 로컬에서 SCP 업로드
scp -i [SSH키] server.js ubuntu@[서버IP]:/opt/bridge/server.js

# PM2 재시작
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart bridge-server"
```

---

### 2단계: n8n 워크플로 — 상품명파싱 노드 수정

Node.js 스크립트로 n8n API 호출하여 수정.

**상품명파싱** 노드 — 첫 단어가 유형(일상/지식/홍보)이면 분리, 아니면 기본값 `홍보`:

```js
const body = $input.first().json.body || {};
const text = (body.text || '').trim();
const userId = body.user_name || 'unknown';

const validTypes = ['일상', '지식', '홍보'];
const parts = text.split(/\s+/);
let contentType = '홍보';
let productName = text;

if (parts.length >= 2 && validTypes.includes(parts[0])) {
  contentType = parts[0];
  productName = parts.slice(1).join(' ');
}

const randomProducts = ['스마트폰 케이스', '무선 이어폰', '보조배터리', '노트북 거치대', '블루투스 스피커'];
if (!productName) {
  productName = randomProducts[Math.floor(Math.random() * randomProducts.length)];
}

return [{ json: { productName, contentType, userId, originalText: text } }];
```

출력 필드 변경:
- 기존: `{ productName, userId, originalText }`
- 변경: `{ productName, contentType, userId, originalText }` ← `contentType` 추가

---

### 3단계: n8n 워크플로 — Claude브릿지호출 노드 수정

`jsonBody`에 `type` 필드 추가:

```
기존: ={"product": "{{ $json.productName }}"}
변경: ={"product": "{{ $json.productName }}", "type": "{{ $json.contentType }}"}
```

---

### 4단계: n8n 워크플로 — 포맷팅 노드 수정

새 프롬프트는 JSON 출력이 아닌 자유 형식 텍스트를 반환하므로, `content`(JSON 파싱 결과)가 null일 때 `raw`(원본 텍스트)를 사용하도록 fallback 추가:

```js
const bridge = $input.first().json;
const content = bridge.content || {};
const parse = $('상품명파싱').first().json;

// content가 있으면 기존 JSON 포맷, 없으면 raw 텍스트 사용
if (bridge.content && typeof bridge.content === 'object') {
  const mainCopy = content.mainCopy || '';
  const body = content.body || '';
  const hashtags = Array.isArray(content.hashtags) ? content.hashtags.join(' ') : '';
  const cta = content.cta || '';

  const message = [
    '📝 콘텐츠 생성 완료',
    '',
    '요청자: ' + parse.userId,
    '상품명: ' + parse.productName,
    '',
    '---',
    mainCopy,
    '',
    body,
    '',
    cta,
    '---',
    '',
    hashtags,
  ].join('\n');
  return [{ json: { message } }];
}

const message = [
  '📝 콘텐츠 생성 완료',
  '',
  '요청자: ' + parse.userId,
  '상품/주제: ' + parse.productName,
  '유형: ' + (parse.contentType || '홍보'),
  '',
  '---',
  bridge.raw || '(내용 없음)',
  '---',
].join('\n');

return [{ json: { message } }];
```

---

### 5단계: n8n 워크플로 — 대시보드페이로드 노드 수정

대시보드 저장 시에도 동일한 fallback 적용 + `platform`을 `'Threads'`로 고정:

```js
const bridge  = $input.first().json;
const content = bridge.content || {};
const parse   = $('상품명파싱').first().json;

if (bridge.content && typeof bridge.content === 'object') {
  return [{
    json: {
      product:   bridge.product   || parse.productName || '',
      title:     content.mainCopy || '',
      body:      content.body     || '',
      hashtags:  Array.isArray(content.hashtags) ? content.hashtags.join(' ') : '',
      requester: parse.userId     || '',
      platform:  bridge.platform  || 'Threads',
    }
  }];
}

return [{
  json: {
    product:   bridge.product   || parse.productName || '',
    title:     parse.productName || '',
    body:      bridge.raw       || '',
    hashtags:  '',
    requester: parse.userId     || '',
    platform:  'Threads',
  }
}];
```

---

### 6단계: n8n 워크플로 활성화 (DB 동기화)

> ⚠ n8n v2 활성화 버그: API로 PATCH 후 `activeVersionId`가 갱신되지 않아 이전 버전 코드로 실행됨.
> **n8n API로 워크플로를 수정할 때마다 반드시 실행해야 합니다.**

```bash
ssh -i [SSH키] ubuntu@[서버IP] "
  docker exec n8n-postgres psql -U n8n -d n8n -c \
    \"UPDATE workflow_entity SET \\\"activeVersionId\\\" = \\\"versionId\\\" WHERE name = 'Mattermost 콘텐츠생성';\"
  cd /opt/n8n && docker compose restart n8n
"
```

확인:

```bash
ssh -i [SSH키] ubuntu@[서버IP] "
  cd /opt/n8n && docker compose logs n8n --tail=5 2>&1 | grep 'Activated'
"
# "Activated workflow "Mattermost 콘텐츠생성"" 출력 확인
```

---

## 전체 수정 요약

| 위치 | 파일/노드 | 수정 내용 |
|------|-----------|-----------|
| 브릿지 서버 | `/opt/bridge/server.js` — `/generate/sns` | type 파라미터 + 프롬프트 3종 + system 메시지 방식 |
| n8n | 상품명파싱 (Code) | 첫 단어 유형 파싱 → `contentType` 출력 추가 |
| n8n | Claude브릿지호출 (HTTP) | jsonBody에 `type` 필드 추가 |
| n8n | 포맷팅 (Code) | `content` null일 때 `raw` fallback |
| n8n | 대시보드페이로드 (Code) | `raw` fallback + platform `'Threads'` 고정 |
| n8n | DB | `activeVersionId = versionId` 동기화 + 재시작 |

---

## 완료 후 확인

```bash
# 1. 브릿지 서버 직접 테스트 (3가지 유형)
curl -X POST https://[API_DOMAIN]/generate/sns \
  -H "Content-Type: application/json" \
  -d '{"product":"세금계산서 실수","type":"일상"}'

curl -X POST https://[API_DOMAIN]/generate/sns \
  -H "Content-Type: application/json" \
  -d '{"product":"4대보험 절감","type":"지식"}'

curl -X POST https://[API_DOMAIN]/generate/sns \
  -H "Content-Type: application/json" \
  -d '{"product":"전기세환급","type":"홍보"}'

# 2. Mattermost 커맨드 테스트
/콘텐츠생성 일상 직원고용 어려움
/콘텐츠생성 지식 4대보험 절감
/콘텐츠생성 홍보 전기세환급
/콘텐츠생성 무선이어폰          ← 유형 생략 시 홍보

# 3. 대시보드 확인
# → 상품명에 유형이 포함되지 않고 분리되어 있는지
# → 플랫폼이 "Threads"로 표시되는지
# → 본문에 Claude 생성 텍스트가 있는지
```

---

## 프롬프트 커스터마이징

새로운 유형을 추가하려면:

1. **server.js** — `prompts` 객체에 새 키 추가 (예: `'뉴스'`)
2. **n8n 상품명파싱** — `validTypes` 배열에 추가 (예: `['일상', '지식', '홍보', '뉴스']`)
3. DB 활성화 동기화 실행

프롬프트 텍스트만 수정하려면 server.js의 `prompts` 객체 내용만 변경하고 `pm2 restart bridge-server` 실행.

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 유형을 넣었는데 상품명에 유형이 포함됨 ("일상 직원고용") | n8n 워크플로가 구버전으로 실행됨 | DB `activeVersionId` 동기화 후 n8n 재시작 |
| 대시보드 본문이 비어있음 | 대시보드페이로드 노드가 `content.body`만 읽고 `raw` fallback 없음 | 대시보드페이로드 노드 업데이트 |
| 플랫폼이 "일상"/"지식" 등으로 표시됨 | `platform` 필드에 `contentType`이 들어감 | `platform: 'Threads'`로 고정 |
| n8n API 수정 후 웹훅 404 | `activeVersionId` 불일치로 웹훅 미등록 | DB 동기화 + `docker compose restart n8n` |
| 기존 `/콘텐츠생성 상품명` (유형 없이) 이 안 됨 | 파싱 로직 오류 | 기본값 `'홍보'`로 동작하도록 구현됨 — 정상 작동 |

---

## 서버 관리 명령어

```bash
# 브릿지 서버
pm2 restart bridge-server                # 프롬프트 수정 후 재시작
pm2 logs bridge-server --lines 20        # API 호출 로그 확인

# n8n 워크플로 버전 상태 확인
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "SELECT name, active, \"activeVersionId\", \"versionId\", (\"activeVersionId\" = \"versionId\") AS versions_match FROM workflow_entity WHERE name = 'Mattermost 콘텐츠생성';"
```
