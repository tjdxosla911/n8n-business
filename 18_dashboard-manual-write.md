# 18. 대시보드 수동 글 작성 및 이미지 등록

## 개요
n8n 워크플로우 없이 대시보드에서 직접 콘텐츠 작성 및 이미지 등록 기능 추가

---

## 추가된 기능

| 기능 | 위치 | 설명 |
|---|---|---|
| 새 글 작성 | 목록 페이지 헤더 | 모달로 콘텐츠 직접 작성 + 이미지 첨부 + 썸네일 미리보기 |
| 이미지 수동 등록 | 콘텐츠 상세 페이지 | 기존 콘텐츠에 이미지 추가 업로드 |
| 이미지 삭제 | 콘텐츠 상세 페이지 | 이미지 hover 시 X 버튼 → DB + 실제 파일 동시 삭제 |

---

## 변경 파일

### app.js - 추가된 라우트

**POST /contents/create** — 대시보드에서 직접 글 작성

```js
app.post('/contents/create', upload.array('images', 10), (req, res) => {
  const { title, body, hashtags, product, requester, platform } = req.body;
  if (!title || !title.trim()) return res.redirect('/?error=title');

  const baseUrl = process.env.DASHBOARD_URL || 'https://dash.[DOMAIN]';
  const imageUrls = req.files && req.files.length
    ? req.files.map(f => baseUrl + '/uploads/' + f.filename).join(', ')
    : null;

  const result = db.prepare(`
    INSERT INTO contents (title, body, hashtags, product, requester, platform, image_url)
    VALUES (?, ?, ?, ?, ?, ?, ?)
  `).run(title.trim(), body || '', hashtags || '', product || '', requester || '', platform || '인스타그램', imageUrls);

  res.redirect('/contents/' + result.lastInsertRowid);
});
```

**POST /contents/:id/image** — 기존 콘텐츠에 이미지 추가

```js
app.post('/contents/:id/image', upload.array('images', 10), (req, res) => {
  const content = db.prepare('SELECT * FROM contents WHERE id = ?').get(req.params.id);
  if (!content) return res.status(404).send('Not found');

  const baseUrl = process.env.DASHBOARD_URL || 'https://dash.[DOMAIN]';
  const newUrls = req.files.map(f => baseUrl + '/uploads/' + f.filename);

  const existing = content.image_url
    ? content.image_url.split(',').map(u => u.trim()).filter(Boolean)
    : [];
  const merged = [...existing, ...newUrls].join(', ');

  db.prepare('UPDATE contents SET image_url = ?, updated_at = CURRENT_TIMESTAMP WHERE id = ?')
    .run(merged, req.params.id);

  res.redirect('/contents/' + req.params.id);
});
```

---

### views/index.ejs - 새 글 작성 버튼 + 모달

- 헤더 우측에 `+ 새 글 작성` 버튼 추가
- 클릭 시 모달 팝업
- 입력 필드: 제목(필수), 본문, 해시태그, 상품명, 요청자, 플랫폼, 이미지 첨부(다중)
- `enctype="multipart/form-data"` 설정

### views/index.ejs - 모달 이미지 미리보기

- 이미지 선택 시 JS로 썸네일 즉시 표시
- "N개 파일 선택됨" 텍스트로 선택 수 확인 가능
- 모달 닫으면 미리보기 초기화

### views/detail.ejs - 이미지 업로드 + 삭제

- 편집 폼 위에 이미지 업로드 섹션 추가
- 다중 이미지 선택 가능 (최대 10장)
- 기존 이미지에 누적 추가됨
- 이미지 hover 시 빨간 X 버튼 표시 → 클릭 시 파일 + DB 동시 삭제

**POST /contents/:id/image/delete** — 이미지 삭제

```js
app.post('/contents/:id/image/delete', (req, res) => {
  // 실제 파일 삭제
  const filename = targetUrl.split('/uploads/')[1];
  const filePath = path.join(__dirname, 'public', 'uploads', filename);
  if (fs.existsSync(filePath)) fs.unlinkSync(filePath);

  // DB에서 해당 URL 제거
  const remaining = content.image_url
    .split(',').map(u => u.trim()).filter(u => u && u !== targetUrl);
  db.prepare('UPDATE contents SET image_url = ? WHERE id = ?')
    .run(remaining.length ? remaining.join(', ') : null, req.params.id);
});
```

---

## 배포 절차

```bash
ssh -i [SSH_KEY_PATH] ubuntu@[SERVER_IP]

cd /opt/dashboard

# app.js, views/index.ejs, views/detail.ejs 수정 후
pm2 restart dashboard
```

---

## 주의사항

- 이미지는 `/opt/dashboard/public/uploads/`에 저장됨
- 파일 크기 제한: 10MB/장
- 지원 형식: jpg, jpeg, png, gif, webp
- `uploads/` 디렉토리는 `.gitignore`에 포함되어 git에 올라가지 않음
