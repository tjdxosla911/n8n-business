# 15. AWS 서버 Git 설정

## 개요
AWS Lightsail 서버의 3개 Node.js 앱을 GitHub에 연동

---

## 사전 준비

| 항목 | 내용 |
|---|---|
| 서버 IP | [SERVER_IP] |
| GitHub 계정 | [GITHUB_USERNAME] |
| SSH 키 경로 | `~/.ssh/id_ed25519` |

---

## 앱별 레포 구성

| 앱 | 서버 경로 | GitHub 레포 |
|---|---|---|
| 대시보드 | `/opt/dashboard` | `n8n-content-dashboard` |
| 브릿지 서버 | `/opt/bridge` | `n8n-bridge-server` |
| 미디어 서버 | `/opt/media` | `n8n-media-server` |

---

## 설치 절차

### 1. 서버 SSH 접속

```bash
ssh -i [SSH_KEY_PATH] ubuntu@[SERVER_IP]
```

### 2. GitHub SSH 키 생성

```bash
ssh-keygen -t ed25519 -C 'aws-lightsail' -f ~/.ssh/id_ed25519 -N ''
cat ~/.ssh/id_ed25519.pub
```

### 3. GitHub에 공개키 등록

**GitHub → Settings → SSH and GPG keys → New SSH key**

- Title: `aws-lightsail`
- Key: 위에서 출력된 공개키 붙여넣기

### 4. 연결 테스트

```bash
ssh -T git@github.com
# Hi [USERNAME]! You've successfully authenticated... 확인
```

### 5. 각 앱 Git 초기화 및 push

```bash
# .gitignore 공통 내용
GITIGNORE='node_modules/
logs/
*.log
.env
uploads/
data/
*.bak
*.bak2'

for DIR in /opt/dashboard /opt/bridge /opt/media; do
  cd $DIR
  echo "$GITIGNORE" > .gitignore
  git init
  git add .
  git commit -m "initial commit"
done
```

### 6. Remote URL 등록 (SSH 방식)

```bash
git -C /opt/dashboard remote add origin git@github.com:[USERNAME]/n8n-content-dashboard.git
git -C /opt/bridge   remote add origin git@github.com:[USERNAME]/n8n-bridge-server.git
git -C /opt/media    remote add origin git@github.com:[USERNAME]/n8n-media-server.git
```

### 7. Push

```bash
git -C /opt/dashboard push -u origin main
git -C /opt/bridge    push -u origin main
git -C /opt/media     push -u origin main
```

---

## 이후 코드 수정 시 작업 흐름

```bash
cd /opt/dashboard  # or /opt/bridge, /opt/media

# 코드 수정 후
git add .
git commit -m "수정 내용"
git push
```

---

## 주의사항

- `.env` 파일은 `.gitignore`에 포함되어 있어 push되지 않음
- `uploads/`, `data/` 디렉토리도 제외됨 (용량 큰 파일)
- SSH 키 방식이므로 토큰 만료 걱정 없음
