# [실행 가이드 02] SSL 인증서 + Nginx 리버스 프록시 설정

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 반드시 `01_lightsail-n8n-setup.md` 작업 완료 후 진행하세요.

---

## 사전 조건 체크

- [ ] 01번 작업 완료 (n8n 서버 실행 중)
- [ ] 도메인 DNS A레코드 설정 완료 (`도메인` → `서버 고정IP`)
- [ ] DNS 전파 확인 (보통 수분~1시간 소요)

---

## Claude에게 전달할 정보

```
서버 고정 IP  :
도메인        :           (예: n8n.example.com)
이메일        :           (Let's Encrypt 인증서 발급용)
```

---

## Claude 실행 내용 (참고용)

### 작업 순서

**1단계 - Nginx + Certbot 설치, SSL 인증서 발급**

SSH로 서버 접속 후 실행:
```bash
# Nginx, Certbot 설치
sudo apt-get update -y -q
sudo apt-get install -y -q nginx certbot python3-certbot-nginx

# 임시 Nginx 설정 (도메인 확인용)
sudo tee /etc/nginx/sites-available/n8n > /dev/null << 'EOF'
server {
    listen 80;
    server_name [도메인];
    location / { return 200 'ok'; }
}
EOF

sudo ln -sf /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/n8n
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx

# SSL 인증서 발급
sudo certbot --nginx -d [도메인] --non-interactive --agree-tos \
  -m [이메일] --redirect
```

**2단계 - Nginx 리버스 프록시 설정**

⚠ Nginx 설정 파일은 반드시 **로컬에서 파일 생성 후 SCP 전송** 방식 사용
(SSH heredoc 안에서 `$` 변수가 쉘에 의해 치환되는 문제 방지)

로컬에 `n8n_nginx.conf` 파일 생성:
```nginx
server {
    listen 80;
    server_name [도메인];
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name [도메인];

    ssl_certificate     /etc/letsencrypt/live/[도메인]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[도메인]/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 100m;

    location / {
        proxy_pass         http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 300s;
    }
}
```

SCP로 전송 후 적용:
```bash
scp -i [키파일] n8n_nginx.conf ubuntu@[서버IP]:/tmp/n8n_nginx.conf
ssh -i [키파일] ubuntu@[서버IP] "
  sudo cp /tmp/n8n_nginx.conf /etc/nginx/sites-available/n8n
  sudo nginx -t && sudo systemctl reload nginx
"
```

**3단계 - docker-compose.yml 수정 및 n8n 재시작**

```bash
# N8N_SECURE_COOKIE=false 제거
sudo sed -i '/N8N_SECURE_COOKIE=false/d' /opt/n8n/docker-compose.yml

# HTTP → HTTPS, HOST 변경
sudo sed -i 's/N8N_PROTOCOL=http/N8N_PROTOCOL=https/' /opt/n8n/docker-compose.yml
sudo sed -i 's/N8N_HOST=0.0.0.0/N8N_HOST=[도메인]/' /opt/n8n/docker-compose.yml

# n8n 재시작
cd /opt/n8n && sudo docker compose up -d --force-recreate n8n
```

---

## 완료 후 확인

- `https://[도메인]` 접속 → n8n 로그인 화면 (자물쇠 아이콘 확인)
- `http://[도메인]` 접속 시 자동으로 https로 리다이렉트되는지 확인

---

## SSL 인증서 자동 갱신

Certbot 설치 시 자동으로 cron/systemd 타이머 등록됨. 별도 설정 불필요.

만료일 확인:
```bash
sudo certbot certificates
```

수동 갱신 테스트:
```bash
sudo certbot renew --dry-run
```

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 인증서 발급 실패 | DNS 전파 안 됨 | DNS 설정 확인 후 재시도 |
| Nginx 설정 오류 `invalid number of arguments` | `$` 변수가 shell에서 치환됨 | SCP 파일 전송 방식 사용 (위 참고) |
| n8n 로그인 후 redirect 오류 | PROTOCOL/HOST 설정 안 맞음 | docker-compose 환경변수 확인 |

---

## 작업 완료 기준

- [ ] `https://[도메인]` 접속 성공
- [ ] 브라우저 주소창에 자물쇠(🔒) 표시
- [ ] http 접속 시 https로 자동 리다이렉트
- [ ] n8n 로그인 및 정상 작동 확인
