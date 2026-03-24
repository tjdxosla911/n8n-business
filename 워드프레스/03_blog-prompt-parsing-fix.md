# [실행 가이드] 블로그 프롬프트 교체 + JSON 파싱 강화 + \n 제거

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 WordPress 블로그 워크플로(02) 완료 후 진행하세요.

---

## 개요

`/generate/blog` 엔드포인트에 대한 3가지 수정:

1. **프롬프트 교체** — SEO 작가 → "비즈니스 이면을 파헤치는 블로그 작가" 페르소나. 시리즈/편수 지원
2. **JSON 파싱 강화** — Claude 응답에서 JSON만 정확히 추출 (3단계 fallback + 수동 필드 추출)
3. **\n 줄바꿈 제거** — WordPress REST API에 전달되는 content에서 `\n` 제거, HTML 태그로만 구조화

---

## 사전 조건

- [ ] 02번 가이드 완료 (블로그 워크플로 가동 중)
- [ ] 서버 SSH 접속 가능

---

## Claude에게 전달할 정보

```
서버 고정 IP         :
SSH 키 경로          :        (예: C:\Users\...\lightsail_key.pem)
```

---

## Claude 실행 내용 (참고용)

### 1단계: `/generate/blog` 프롬프트 교체

**변경 전**: SEO 전문 블로그 작가, JSON 4필드 (title/content/excerpt/tags)
**변경 후**: 비즈니스 이면을 파헤치는 블로그 작가, JSON 7필드 (+ series/episode/next_episode_preview)

요청 파라미터 추가:
- `series` — 시리즈명 (선택, 기본 null)
- `episode` — 편수 (선택, 기본 1)

새 시스템 프롬프트:

```
당신은 기업의 수익구조와 비즈니스 이면을 파헤치는 블로그 작가입니다.
독자가 읽다가 멈출 수 없게 만드는 게 당신의 목표입니다.
수십 개의 중소기업 내부를 들여다보면서 돈이 어디서 새고 어디서 버는지 직접 목격한 사람의 시선으로 씁니다.

주제: ${topic}
시리즈명: ${series || null}
편수: ${episode || 1}

글쓰기 규칙:
1. 제목: 클릭 안 하면 손해볼 것 같은 제목. "진짜", "숨겨진", "아무도 안 알려주는", "이면" 같은 단어 활용. 시리즈물이면 [시리즈명] n편: 형태로 앞에 붙여라
2. 첫 문단: "이거 모르면 당신만 손해" 느낌으로 시작. 독자가 "나 몰랐는데?" 하고 찔리는 사실 하나로 시작. 질문형 시작 금지
3. 본문 구조:
   - 표면적으로 알려진 사실 제시
   - "근데 진짜는 이거야" 반전 포인트
   - 구체적 수치나 사례 포함 (없으면 "업계 관계자에 따르면" 활용)
   - 독자 입장에서 "나한테 왜 중요한가" 연결
4. 심리 트리거: 손실 회피, 호기심 갭, 사회적 증거 중 최소 2개 사용
5. 분량: 1500자 이상
6. 마지막 문단: 반드시 다음 편 예고. "다음 편에서는 [더 충격적인 내용] 공개합니다" 형태로 끝내라. 독자가 북마크 하게 만들어라
7. 전체 톤: 차분한 정보 전달 금지. 불편한 진실을 던지는 사람의 말투
8. content 필드는 반드시 HTML 태그(<p>, <h2>, <h3>, <br>, <ul>, <li>, <strong> 등)로만 줄바꿈/구조화하라. \n 줄바꿈 문자 절대 사용 금지. 모든 문단은 <p>태그로 감싸라

출력 형식 (JSON만 출력, 다른 텍스트 절대 금지):
{
  "title": "제목",
  "content": "<h2>소제목</h2><p>본문...</p><p>본문...</p> (HTML만 사용, \n 금지, 1500자 이상)",
  "excerpt": "요약 2줄 (클릭 유도형)",
  "tags": ["태그1", "태그2", "태그3"],
  "series": "시리즈명 또는 null",
  "episode": 편수,
  "next_episode_preview": "다음 편 예고 문구"
}
```

---

### 2단계: JSON 파싱 3단계 fallback

Claude 응답에서 JSON을 추출하는 로직:

```js
// 코드블록 제거
var cleanText = text;
var codeBlockMatch = text.match(/```(?:json)?\s*\n([\s\S]*?)\n```/);
if (codeBlockMatch) {
  cleanText = codeBlockMatch[1].trim();
}

// 1단계: 전체 텍스트 직접 파싱
try { parsed = JSON.parse(cleanText); } catch (e) {}

// 2단계: 첫 { ~ 마지막 } 추출
if (!parsed) {
  try {
    var firstBrace = cleanText.indexOf('{');
    var lastBrace = cleanText.lastIndexOf('}');
    if (firstBrace !== -1 && lastBrace > firstBrace) {
      parsed = JSON.parse(cleanText.substring(firstBrace, lastBrace + 1));
    }
  } catch (e) {}
}

// 3단계: 수동 필드 추출 (HTML content가 JSON을 깨뜨리는 경우)
if (!parsed) {
  // title, excerpt, tags 등은 정규식으로 추출
  // content는 "content": " ~ ", "excerpt" 사이 범위로 추출
  // → 상세 코드는 server.js 참고
}
```

> **시행착오**: 1~2단계만으로는 부족. Claude가 content에 HTML을 넣으면 JSON 내부의 `"`, `{`, `}` 등이 이스케이프 처리되지 않아 `JSON.parse` 실패.
> 3단계 수동 파싱은 각 필드를 개별적으로 추출하여 content의 HTML이 깨뜨려도 나머지 필드는 정상 파싱.

---

### 3단계: content \n 제거

파싱 성공 후, WordPress에 전달하기 전 `\n` 제거:

```js
if (parsed) {
  if (parsed.content) {
    parsed.content = parsed.content.replace(/\\n/g, '').replace(/\n/g, '');
  }
  return res.json({ success: true, topic: topic, blog: parsed, raw: text });
}
```

이중 안전장치:
1. **프롬프트에서 금지**: 규칙 8번 "\\n 줄바꿈 문자 절대 사용 금지"
2. **코드에서 제거**: 프롬프트를 무시하고 `\n`을 넣어도 서버에서 제거

---

### 4단계: 배포

```bash
scp -i [SSH키] server.js ubuntu@[서버IP]:/opt/bridge/server.js
ssh -i [SSH키] ubuntu@[서버IP] "pm2 restart bridge-server"
```

---

## API 변경 사항

### POST /generate/blog

**요청:**

```json
{
  "topic": "중소기업 4대보험 절감",
  "series": "사장님이 모르는 돈 이야기",   // 선택, 기본 null
  "episode": 1                            // 선택, 기본 1
}
```

**응답 (blog 필드):**

```json
{
  "title": "제목",
  "content": "<h2>...</h2><p>...</p>...",
  "excerpt": "요약 2줄",
  "tags": ["태그1", "태그2"],
  "series": "시리즈명 또는 null",
  "episode": 1,
  "next_episode_preview": "다음 편 예고 문구"
}
```

---

## 완료 후 확인

```bash
# API 직접 테스트
curl -s -X POST https://[API_DOMAIN]/generate/blog \
  -H "Content-Type: application/json" \
  -d '{"topic":"중소기업 전기세 환급 제도"}' \
  --max-time 120 | python3 -m json.tool | head -20

# 확인 포인트:
# - blog.content에 \n이 없는지
# - blog.content가 <p>, <h2> 등 HTML 태그로만 구성되는지
# - blog.content 길이가 1500자 이상인지
# - blog.next_episode_preview가 있는지
```

Mattermost 테스트:

```
/블로그생성 중소기업 전기세 환급 제도
→ WordPress에 발행된 글에서 \n 문자가 보이지 않아야 함
→ HTML 태그로 구조화된 깔끔한 레이아웃
```

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| content에 `\n`이 리터럴 문자로 표시 | Claude가 프롬프트 규칙을 무시하고 `\n` 삽입 | 서버에서 `replace(/\\n/g, '').replace(/\n/g, '')` 로 제거 (안전장치) |
| JSON 파싱 실패 → blog: null | content 안 HTML의 `"`가 JSON 이스케이프 안 됨 | 3단계 수동 필드 추출로 fallback |
| 응답 앞에 "Blog" 같은 텍스트 | Claude가 JSON 외 텍스트 출력 | 프롬프트에 "JSON만 출력, 다른 텍스트 절대 금지" 명시 + 코드블록/firstBrace 파싱 |
| 시리즈 파라미터가 무시됨 | n8n 주제파싱 노드에서 series/episode 미전달 | 현재 Mattermost 커맨드로는 topic만 전달. 시리즈 기능은 API 직접 호출 시 사용 가능 |
