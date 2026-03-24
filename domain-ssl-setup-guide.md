# 도메인 연결 + SSL(HTTPS) 설정 가이드

> 작성일: 2026-03-13
> 전제 조건: AWS Lightsail + n8n 서버 세팅 완료 (setup-guide 참고)
> 목적: 도메인 구매 → DNS 연결 → SSL 인증서 → HTTPS 접속까지

---

## 전체 구조

```
[브라우저] → https://n8n.example.com
    ↓
[Nginx (443/SSL)] → 리버스 프록시 → [n8n (localhost:5678)]
```

---

## 1단계: 도메인 구매

### 추천 도메인 등록 업체

| 업체 | 특징 | .com 가격 (연) |
|------|------|---------------|
| 가비아 (gabia.com) | 국내 1위, 한국어 지원 | 약 11,000원 |
| 호스팅케이알 (hosting.kr) | 저렴, 한국어 지원 | 약 9,900원 |
| Namecheap | 해외, 저렴 | 약 $8.88 |
| Cloudflare Registrar | 원가 판매, DNS 무료 | 약 $9.15 |

### 도메인 선택 팁
- 사업용이면 .com 추천
- 짧고 기억하기 쉬운 이름
- 이미 보유한 도메인에 서브도메인 추가도 가능 (추가 비용 없음)

---

## 2단계: DNS 설정 (서브도메인 연결)

### 가비아 기준

1. https://dns.gabia.com 접속
2. 해당 도메인 선택 → **DNS 설정**
3. **레코드 수정** 클릭
4. 새 레코드 추가:

| 타입 | 호스트 | 값/위치 | TTL |
|------|--------|---------|-----|
| A | n8n | [서버 고정 IP] | 3600 |

5. 저장

### 호스팅케이알 기준

1. 로그인 → 도메인 관리 → DNS 관리
2. 레코드 추가:
   - 타입: A
   - 호스트: n8n
   - IP: [서버 고정 IP]
3. 저장

### Cloudflare 기준

1. 대시보드 → 해당 도메인 → DNS
2. Add record:
   - Type: A
   - Name: n8n
   - IPv4: [서버 고정 IP]
   - Proxy status: DNS only (회색 구름)
3. Save

### 확인 방법

터미널에서:
```bash
nslookup n8n.example.com
```
서버 IP가 나오면 DNS 연결 성공. 반영까지 최대 10분 소요.

---

## 3단계: 서브도메인 계획

하나의 서버에 여러 서비스를 올릴 때:

| 서브도메인 | 용도 | 내부 포트 |
|-----------|------|----------|
| n8n.example.com | n8n 자동화 엔진 | 5678 |
| api.example.com | 브릿지 서버 (Claude) | 3100 |
| dash.example.com | 승인 대시보드 | 3000 |
| example.com | 메인 사이트/블로그 | 80 |

전부 같은 서버 IP의 A 레코드로 추가. Nginx에서 서브도메인별로 다른 포트로 라우팅.

---

## 4단계: SSL 인증서 발급 (Let's Encrypt)

### SSH 접속

```bash
ssh -i lightsail_key.pem ubuntu@[서버IP]
```

### Nginx 설치

```bash
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### Certbot 설치

```bash
sudo apt-get install -y certbot python3-certbot-nginx
```

### SSL 인증서 발급

```bash
sudo certbot --nginx -d n8n.example.com
```

프롬프트 입력:
- 이메일: 본인 이메일 입력
- 약관 동의: Y
- 뉴스레터: N
- HTTP → HTTPS 리다이렉트: 2 (Redirect)

성공 시 출력:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/n8n.example.com/fullchain.pem
Key is saved at: /etc/letsencrypt/live/n8n.example.com/privkey.pem
```

### 인증서 자동 갱신 확인

```bash
sudo certbot renew --dry-run
```

Let's Encrypt 인증서는 90일마다 갱신 필요. certbot이 자동으로 cron job 등록해줌.

---

## 5단계: Nginx 리버스 프록시 설정

### 설정 파일 생성

```bash
sudo nano /etc/nginx/sites-available/n8n
```

아래 내용 입력 (도메인명 수정):

```nginx
server {
    listen 80;
    server_name n8n.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding on;
    }
}
```

### 설정 활성화

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` 에서 `syntax is ok` 나오면 성공.

---

## 6단계: n8n 설정 업데이트

### docker-compose.yml 수정

```bash
cd /opt/n8n
sudo nano docker-compose.yml
```

n8n 환경변수 변경:

```yaml
# 변경 전
- N8N_SECURE_COOKIE=false
- N8N_PROTOCOL=http
- N8N_HOST=0.0.0.0

# 변경 후
- N8N_PROTOCOL=https
- N8N_HOST=n8n.example.com
- WEBHOOK_URL=https://n8n.example.com/
```

`N8N_SECURE_COOKIE=false` 줄은 삭제.

### n8n 재시작

```bash
cd /opt/n8n
sudo docker compose up -d --force-recreate n8n
```

---

## 7단계: 접속 확인

브라우저에서:

```
https://n8n.example.com
```

확인 사항:
- [x] 자물쇠 아이콘 표시됨 (SSL 정상)
- [x] 포트 번호 없이 접속 가능
- [x] HTTP 접속 시 HTTPS로 자동 리다이렉트
- [x] n8n 로그인 화면 정상 표시

---

## 8단계: 슬랙 웹훅 URL 업데이트

SSL 설정 완료 후 슬랙 앱의 URL을 업데이트:

1. https://api.slack.com/apps → 앱 선택
2. **Slash Commands** → 기존 명령어 수정:
   ```
   기존: http://[IP]:5678/webhook/...
   변경: https://n8n.example.com/webhook/...
   ```
3. **OAuth & Permissions** → Redirect URLs 업데이트:
   ```
   https://n8n.example.com/rest/oauth2-credential/callback
   ```
4. **Event Subscriptions** (사용 시):
   ```
   https://n8n.example.com/webhook/...
   ```

---

## 9단계: 보안 강화 (선택)

### 방화벽에서 직접 포트 접근 차단

SSL + Nginx 설정 완료 후, n8n 직접 접근 포트를 차단:

```bash
# Lightsail 방화벽에서
# 5678 포트: 제거 (Nginx가 프록시하므로 불필요)
# 3100 포트: 내 IP에서만 접근 가능하게 제한
# 22 포트: 내 IP에서만 접근 가능하게 제한
```

### IP 제한 (Nginx)

특정 IP에서만 n8n 접근 가능하게:

```nginx
# /etc/nginx/sites-available/n8n 의 server 블록 안에 추가
allow YOUR_IP_ADDRESS;
deny all;
```

수정 후:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 서브도메인 추가 시 (브릿지 서버, 대시보드 등)

### 1. DNS에 A 레코드 추가
```
A  api   [서버IP]
A  dash  [서버IP]
```

### 2. SSL 인증서 발급
```bash
sudo certbot --nginx -d api.example.com
sudo certbot --nginx -d dash.example.com
```

### 3. Nginx 설정 파일 추가
```bash
sudo nano /etc/nginx/sites-available/api
# 위 n8n 설정과 동일 구조, proxy_pass 포트만 변경 (3100 등)
sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 트러블슈팅

### certbot 실행 시 "Could not automatically find a matching server block"
- Nginx 기본 설정에 server_name이 없어서 발생
- 위 5단계의 설정 파일을 먼저 만들고 certbot 실행

### "Too many certificates already issued"
- Let's Encrypt는 주당 5개까지 발급 제한
- 테스트 시 `--staging` 옵션 사용

### HTTPS 접속은 되는데 웹소켓 에러
- Nginx 설정에서 `proxy_set_header Upgrade`와 `Connection "upgrade"` 확인
- n8n은 웹소켓을 사용하므로 이 설정이 필수

### 인증서 갱신 실패
```bash
sudo certbot renew --force-renewal
sudo systemctl reload nginx
```

### HTTP로 접속 시 리다이렉트 안 됨
- Nginx 80번 서버 블록에 `return 301 https://...` 확인
- `sudo systemctl reload nginx` 실행

---

## 전체 체크리스트

- [ ] 도메인 구매 또는 기존 도메인 확인
- [ ] DNS A 레코드 추가 (서브도메인 → 서버 IP)
- [ ] DNS 반영 확인 (nslookup)
- [ ] Nginx 설치
- [ ] Certbot 설치
- [ ] SSL 인증서 발급
- [ ] Nginx 리버스 프록시 설정
- [ ] n8n docker-compose.yml 업데이트
- [ ] n8n 컨테이너 재시작
- [ ] HTTPS 접속 확인
- [ ] 슬랙 웹훅 URL 업데이트
- [ ] 보안 강화 (포트 차단, IP 제한)
- [ ] 인증서 자동 갱신 확인

---

> 이 가이드는 n8n 서버 세팅 가이드(setup-guide)와 함께 사용됩니다.
> 두 문서를 합치면 서버 생성부터 HTTPS 접속까지 전체 프로세스가 커버됩니다.
