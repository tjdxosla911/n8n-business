# [실행 가이드] WordPress Docker 설치 + Nginx SSL 연동

> Claude에게 이 문서를 첨부하고 아래 정보를 함께 전달하면 자동으로 실행합니다.
> 기존 Lightsail 서버(n8n/Mattermost/Bridge/Dashboard)에 WordPress를 추가 설치합니다.

---

## 사전 조건

- [ ] Lightsail 서버 가동 중 (Nginx 설치 완료 상태)
- [ ] 가비아 DNS에 A레코드 추가: `blog` → `서버 고정IP`
- [ ] DNS 전파 확인

---

## Claude에게 전달할 정보

```
서버 고정 IP         :
SSH 키 경로          :        (예: C:\Users\...\lightsail_key.pem)
블로그 도메인        :        (예: blog.example.com)
관리자 이메일        :        (SSL 인증서 발급용)
```

---

## Claude 실행 내용 (참고용)

### 1단계: Docker Compose 파일 작성

`/opt/wordpress/docker-compose.yml`:

```yaml
version: "3.8"

services:
  wordpress-db:
    image: mysql:8.0
    container_name: wordpress-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: [랜덤 생성]
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: [랜덤 생성]
    volumes:
      - wordpress_db:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    depends_on:
      - wordpress-db
    ports:
      - "8081:80"
    environment:
      WORDPRESS_DB_HOST: wordpress-db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: [위와 동일]
    volumes:
      - wordpress_data:/var/www/html

volumes:
  wordpress_db:
  wordpress_data:
```

구성:
- **WordPress**: 최신 버전, 포트 8081 (호스트) → 80 (컨테이너)
- **MySQL 8.0**: WordPress 전용 DB, 외부 포트 노출 없음
- **볼륨**: `wordpress_data` (파일), `wordpress_db` (DB) — 컨테이너 재시작해도 데이터 유지

---

### 2단계: 컨테이너 실행

```bash
sudo mkdir -p /opt/wordpress
sudo chown ubuntu:ubuntu /opt/wordpress

# docker-compose.yml 작성 후
cd /opt/wordpress && docker compose up -d

# 상태 확인
docker ps --filter name=wordpress

# WordPress 응답 확인 (302 = 설치 페이지 리다이렉트 → 정상)
curl -s -o /dev/null -w '%{http_code}' http://localhost:8081
```

> MySQL 초기화에 10~15초 소요. 처음에 500 에러가 나오면 잠시 대기 후 재확인.

---

### 3단계: SSL 인증서 발급

```bash
sudo certbot --nginx -d [BLOG_DOMAIN] \
  --non-interactive --agree-tos -m [ADMIN_EMAIL]
```

> certbot이 Nginx server_name 블록을 찾지 못해 자동 설치에 실패할 수 있음.
> 인증서는 정상 발급되므로, 4단계에서 Nginx 설정을 직접 작성.

인증서 경로:
- `/etc/letsencrypt/live/[BLOG_DOMAIN]/fullchain.pem`
- `/etc/letsencrypt/live/[BLOG_DOMAIN]/privkey.pem`

---

### 4단계: Nginx 리버스 프록시 설정

`/etc/nginx/sites-available/blog`:

```nginx
server {
    listen 80;
    server_name [BLOG_DOMAIN];
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name [BLOG_DOMAIN];

    ssl_certificate     /etc/letsencrypt/live/[BLOG_DOMAIN]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[BLOG_DOMAIN]/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 64m;

    location / {
        proxy_pass         http://localhost:8081;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

> `client_max_body_size 64m` — WordPress 미디어 업로드 용량 제한. 기본 1m이면 이미지 업로드 실패.

```bash
sudo ln -sf /etc/nginx/sites-available/blog /etc/nginx/sites-enabled/blog
sudo nginx -t && sudo systemctl reload nginx
```

---

### 5단계: WordPress 초기 설정

`https://[BLOG_DOMAIN]` 접속 후 웹 UI에서 진행 (Claude가 대신할 수 없음):

1. 언어 선택 (한국어)
2. 사이트 제목 입력
3. 관리자 계정 생성 (사용자명/비밀번호/이메일)
4. 설치 완료

---

## 서버 포트 구성 (WordPress 추가 후)

| 서비스 | 포트 | 도메인 |
|--------|------|--------|
| Nginx | 80/443 | - |
| n8n | 5678 | n8n.도메인 |
| Mattermost | 8065 | chat.도메인 |
| Bridge Server | 3100 | api.도메인 |
| Dashboard | 3000 | dash.도메인 |
| **WordPress** | **8081** | **blog.도메인** |
| WordPress DB (MySQL) | 3306 (내부) | - |

---

## 완료 후 확인

```bash
# Docker 상태
docker ps --filter name=wordpress

# HTTPS 접속
curl -s -o /dev/null -w '%{http_code}' https://[BLOG_DOMAIN]
# 200 또는 302 → 정상

# Nginx 설정
ls /etc/nginx/sites-enabled/
# api  blog  chat  dash  n8n
```

---

## 서버 관리 명령어

```bash
# WordPress 컨테이너 상태
docker ps --filter name=wordpress

# 재시작
cd /opt/wordpress && docker compose restart

# 중지
cd /opt/wordpress && docker compose down

# 로그
docker logs wordpress --tail 20
docker logs wordpress-db --tail 20

# WordPress 파일 위치 (볼륨)
docker volume inspect wordpress_wordpress_data

# DB 백업
docker exec wordpress-db mysqldump -u wpuser -p'[DB_PASSWORD]' wordpress > wp_backup.sql
```

---

## 알려진 이슈

| 증상 | 원인 | 해결 |
|------|------|------|
| 최초 접속 시 500 에러 | MySQL 초기화 미완료 | 10~15초 대기 후 재접속 |
| certbot 자동 설치 실패 | Nginx에 server_name 블록 없음 | 인증서는 발급됨 → Nginx 설정 직접 작성 |
| 미디어 업로드 실패 "파일이 너무 큼" | Nginx `client_max_body_size` 기본값 1m | `client_max_body_size 64m` 설정 확인 |
| WordPress 주소 설정 오류로 접속 불가 | wp-admin에서 사이트 URL 잘못 변경 | DB에서 직접 수정: `docker exec wordpress-db mysql -u wpuser -p wordpress -e "UPDATE wp_options SET option_value='https://[DOMAIN]' WHERE option_name IN ('siteurl','home');"` |
| HTTPS 리다이렉트 무한 루프 | WordPress가 프록시 뒤에 있는데 HTTPS 감지 못함 | `wp-config.php`에 추가: `$_SERVER['HTTPS'] = 'on';` |
