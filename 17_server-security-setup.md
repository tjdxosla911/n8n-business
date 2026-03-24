# 17. 서버 보안 설정

## 개요
AWS Lightsail 서버의 방화벽(ufw) 설정으로 불필요한 포트 외부 노출 차단

---

## 보안 구성 원칙

- 외부에서는 **22(SSH), 80(HTTP), 443(HTTPS)** 만 접근 허용
- 앱 포트(3000, 3100, 3200, 5678, 8065, 8081)는 내부(localhost)에서만 접근
- 모든 앱은 Nginx 리버스 프록시를 통해 외부에 노출
- SSH는 키 인증 방식으로만 접근 (비밀번호 인증 비활성화)

---

## 포트 구성

| 포트 | 앱 | 외부 접근 | 접근 방법 |
|---|---|---|---|
| 22 | SSH | 허용 | 키 인증 |
| 80 | HTTP | 허용 | Nginx |
| 443 | HTTPS | 허용 | Nginx |
| 3000 | 대시보드 | 차단 | Nginx 통해서만 |
| 3100 | 브릿지 서버 | 차단 | Nginx 통해서만 |
| 3200 | 미디어 서버 | 차단 | Nginx 통해서만 |
| 5678 | n8n | 차단 | Nginx 통해서만 |
| 8065 | Mattermost | 차단 | Nginx 통해서만 |
| 8081 | WordPress | 차단 | Nginx 통해서만 |

---

## 설치 절차

### 1. 서버 SSH 접속

```bash
ssh -i [SSH_KEY_PATH] ubuntu@[SERVER_IP]
```

### 2. ufw 방화벽 설정

```bash
# SSH 먼저 허용 (순서 중요 — 이 전에 enable 하면 접속 차단됨)
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 방화벽 활성화
sudo ufw --force enable

# 상태 확인
sudo ufw status verbose
```

### 3. 정상 출력 확인

```
Status: active
Default: deny (incoming), allow (outgoing)

To                         Action      From
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

---

## 이후 포트 추가 시

```bash
# 특정 포트 허용
sudo ufw allow [PORT]/tcp

# 특정 포트 차단
sudo ufw deny [PORT]/tcp

# 규칙 삭제
sudo ufw delete allow [PORT]/tcp
```

---

## 주의사항

- ufw 활성화 전 반드시 22번 포트 허용할 것 (안 하면 SSH 접속 차단)
- 서버 재시작 시 ufw 자동 활성화됨
- 고정 IP 확보 후 SSH를 특정 IP로 제한하면 보안 강화 가능
  ```bash
  sudo ufw delete allow 22/tcp
  sudo ufw allow from [YOUR_IP] to any port 22
  ```
