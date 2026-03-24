# [실행 가이드 12] Lightsail 미디어 파일 서버 구축

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01~02` 작업 완료 후 진행하세요.

---

## 사전 조건

- [ ] 01번: Lightsail + n8n 설치 완료
- [ ] 02번: SSL + Nginx 설정 완료
- [ ] 가비아 DNS에 `media` A레코드 추가 → 서버 고정 IP
- [ ] DNS 전파 확인 (`nslookup media.도메인 → 서버 IP`)

---

## Claude에게 전달할 정보

```
서버 SSH 키 경로     :        (예: C:\Users\...\lightsail_key.pem)
서버 호스트          :        (예: n8n.example.com 또는 서버 IP)
미디어 도메인        :        (예: media.example.com)
관리자 이메일        :        (SSL 인증서 발급용)
```

---

## 기능 설명

Lightsail 서버에 중앙 파일 저장소를 구축합니다. S3 없이 서버 디스크에 직접 저장하며, Nginx가 정적 파일을 직접 서빙합니다.

### 용도

```
Mattermost 파일 업로드
 └→ n8n이 감지 → 미디어 서버에 저장
     └→ https://media.도메인/files/파일명.jpg URL 생성
         ├→ Trello 카드에 이미지 첨부
         ├→ Threads 발행에 이미지 삽입
         └→ 블로그 포스트에 이미지 삽입
```

### API 엔드포인트

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/upload` | POST | 파일 직접 업로드 (multipart, 필드명: `file`, 최대 50MB) |
| `/upload-from-url` | POST | URL로 파일 다운로드 후 저장 (JSON: `{ "url": "...", "filename": "..." }`) |
| `/files/파일명` | GET | 저장된 파일 조회 (Nginx 직접 서빙, 30일 캐시) |
| `/list` | GET | 전체 파일 목록 |
| `/health` | GET | 헬스체크 |

### 허용 파일 형식

jpg, jpeg, png, gif, webp, mp4, pdf, svg

---

## Claude 실행 내용 (참고용)

### 1. Express 파일 서버 생성

```bash
sudo mkdir -p /opt/media/uploads
sudo chown -R ubuntu:ubuntu /opt/media
cd /opt/media && npm init -y && npm install express multer
```

```js
// /opt/media/server.js
const express = require('express');
const multer = require('multer');
const path = require('path');
const crypto = require('crypto');
const fs = require('fs');

const app = express();
const PORT = 3200;
const UPLOAD_DIR = path.join(__dirname, 'uploads');
const BASE_URL = 'https://MEDIA_DOMAIN';

// 파일명 중복 방지: timestamp + random hash
const storage = multer.diskStorage({
  destination: UPLOAD_DIR,
  filename: (req, file, cb) => {
    const ext = path.extname(file.originalname);
    const hash = crypto.randomBytes(6).toString('hex');
    const ts = Date.now();
    cb(null, ts + '-' + hash + ext);
  },
});

const upload = multer({
  storage,
  limits: { fileSize: 50 * 1024 * 1024 }, // 50MB
  fileFilter: (req, file, cb) => {
    const allowed = /\.(jpg|jpeg|png|gif|webp|mp4|pdf|svg)$/i;
    if (allowed.test(path.extname(file.originalname))) {
      cb(null, true);
    } else {
      cb(new Error('허용되지 않는 파일 형식'), false);
    }
  },
});

// 파일 직접 업로드
app.post('/upload', upload.single('file'), (req, res) => {
  if (!req.file) return res.status(400).json({ error: '파일이 없습니다' });
  const url = BASE_URL + '/files/' + req.file.filename;
  res.json({
    success: true,
    url,
    filename: req.file.filename,
    originalName: req.file.originalname,
    size: req.file.size,
  });
});

// URL로 다운로드 후 저장 (n8n에서 Mattermost 파일 URL 전달용)
app.post('/upload-from-url', express.json(), async (req, res) => {
  try {
    const { url: fileUrl, filename: origName } = req.body;
    if (!fileUrl) return res.status(400).json({ error: 'url 필수' });

    const response = await fetch(fileUrl);
    if (!response.ok) return res.status(400).json({ error: '파일 다운로드 실패' });

    const ext = path.extname(origName || '') || '.jpg';
    const hash = crypto.randomBytes(6).toString('hex');
    const newFilename = Date.now() + '-' + hash + ext;
    const filePath = path.join(UPLOAD_DIR, newFilename);

    const buffer = Buffer.from(await response.arrayBuffer());
    fs.writeFileSync(filePath, buffer);

    const publicUrl = BASE_URL + '/files/' + newFilename;
    res.json({ success: true, url: publicUrl, filename: newFilename, size: buffer.length });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// 파일 목록
app.get('/list', (req, res) => {
  const files = fs.readdirSync(UPLOAD_DIR).map(f => ({
    filename: f,
    url: BASE_URL + '/files/' + f,
    size: fs.statSync(path.join(UPLOAD_DIR, f)).size,
  }));
  res.json({ count: files.length, files });
});

// 헬스체크
app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.listen(PORT, () => console.log('Media server running on port ' + PORT));
```

### 2. PM2 등록

```bash
pm2 start /opt/media/server.js --name media-server
pm2 save
```

### 3. Nginx 설정

```nginx
# /etc/nginx/sites-available/media.도메인
server {
    listen 80;
    server_name media.도메인;

    # 업로드 API → Express
    location /upload {
        proxy_pass http://127.0.0.1:3200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 50M;
    }

    location /upload-from-url {
        proxy_pass http://127.0.0.1:3200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 50M;
    }

    location /list {
        proxy_pass http://127.0.0.1:3200;
        proxy_set_header Host $host;
    }

    location /health {
        proxy_pass http://127.0.0.1:3200;
    }

    # 정적 파일 서빙 → Nginx 직접 (빠름)
    location /files/ {
        alias /opt/media/uploads/;
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/media.도메인 /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 4. SSL 인증서 발급

```bash
sudo certbot --nginx -d media.도메인 --non-interactive --agree-tos -m 관리자이메일
```

---

## 완료 후 확인

- [ ] `curl https://media.도메인/health` → `{"status":"ok"}`
- [ ] 파일 업로드 테스트:
  ```bash
  curl -X POST https://media.도메인/upload -F 'file=@테스트이미지.jpg'
  # → {"success":true,"url":"https://media.도메인/files/...jpg",...}
  ```
- [ ] 반환된 URL을 브라우저에서 열어 파일 표시 확인
- [ ] URL 다운로드 저장 테스트:
  ```bash
  curl -X POST https://media.도메인/upload-from-url \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://이미지URL","filename":"test.jpg"}'
  ```
- [ ] PM2 상태 확인: `pm2 list` → media-server online

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 업로드 시 413 Request Entity Too Large | Nginx 기본 body 크기 제한 (1MB) | Nginx location에 `client_max_body_size 50M;` 추가 |
| 파일 URL 접근 시 403 Forbidden | uploads 디렉토리 권한 문제 | `sudo chown -R ubuntu:ubuntu /opt/media/uploads` |
| SSL 인증서 발급 실패 | DNS 미전파 또는 80 포트 차단 | `nslookup media.도메인`으로 IP 확인, Lightsail 방화벽에서 80/443 오픈 |

---

## 운영 참고

### 디스크 용량 관리

```bash
# 현재 미디어 용량 확인
du -sh /opt/media/uploads/

# 전체 디스크 확인
df -h
```

Lightsail $5 플랜 기준 디스크 40GB입니다.
이미지 평균 2MB 기준 약 15,000장 저장 가능합니다.
월 1회 용량 확인을 권장합니다.

### 서버 재시작

```bash
pm2 restart media-server
```
