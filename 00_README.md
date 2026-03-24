# SNS 콘텐츠 자동화 사업 템플릿

> 고객에게 "메신저에 상품명만 치면 SNS 게시물이 자동으로 올라간다"를 제공하는 사업 인프라

---

## 서비스 개요

```
고객사 직원이 Mattermost에서 /콘텐츠생성 [상품명] 입력
  → Claude AI가 SNS 마케팅 콘텐츠 자동 생성
  → 승인 대시보드에 등록 (pending)
  → 담당자가 내용 확인/수정 후 승인 버튼 클릭
  → Threads에 게시물 자동 발행 (한글 정상)
  → 대시보드에 발행 완료 상태 + Thread ID 기록
```

### 핵심 가치

- 상품명 하나로 SNS 콘텐츠 생성 → 발행까지 **원클릭 자동화**
- 승인 과정이 있어 **품질 관리** 가능
- 고객사별 독립 서버 → **멀티테넌트 확장** 가능

---

## 전체 아키텍처

```
[Mattermost]                    [n8n]                        [브릿지 서버]
/콘텐츠생성 상품명  ──────→  Webhook 수신              Claude API 호출
                              상품명 파싱               콘텐츠 JSON 생성
                              브릿지 호출 ──────────→   /generate/sns
                              ←──────────────────────   { mainCopy, body, hashtags }
                              Mattermost 채널 전송
                              대시보드 저장 ──────────→ [대시보드] POST /api/contents

[대시보드]                      [n8n]                        [브릿지 서버]
승인 버튼 클릭 ──────────→   Webhook 수신              Threads API 호출
                              텍스트 조립                /publish/threads
                              브릿지 호출 ──────────→   컨테이너 생성 + 발행
                              ←──────────────────────   { success, threadId }
                              대시보드 상태 업데이트 ──→ POST /api/contents/status
```

### 서버 구성 (단일 Lightsail 인스턴스)

| 컴포넌트 | 포트 | 도메인 | 실행 방식 |
|----------|------|--------|----------|
| n8n | 5678 | n8n.도메인.com | Docker |
| PostgreSQL (n8n) | - | - | Docker |
| Mattermost | 8065 | chat.도메인.com | Docker |
| PostgreSQL (MM) | - | - | Docker |
| 브릿지 서버 | 3100 | api.도메인.com | PM2 |
| 대시보드 | 3000 | dash.도메인.com | PM2 |
| Nginx | 80/443 | 전체 리버스 프록시 | systemd |

### n8n 워크플로우

| 워크플로우 | 웹훅 경로 | 기능 |
|-----------|----------|------|
| Mattermost 콘텐츠생성 | `/webhook/content-generate` | 슬래시커맨드 → Claude 생성 → Mattermost 전송 + 대시보드 등록 |
| Threads 자동발행 | `/webhook/content-approve` | 대시보드 승인 → 브릿지 서버 → Threads 발행 → 상태 업데이트 |

---

## 실행 가이드 목록

### 기본 인프라 설치 (필수, 순서대로)

| 순서 | 파일 | 내용 | 소요시간 |
|------|------|------|---------|
| 1 | `01_lightsail-n8n-setup.md` | AWS Lightsail 인스턴스 생성 + Docker + n8n 설치 | ~5분 |
| 2 | `02_ssl-nginx-setup.md` | 도메인 SSL 인증서 + Nginx 리버스 프록시 | ~3분 |
| 3 | `03_mattermost-setup.md` | Mattermost 설치 + SSL + 봇/채널/슬래시커맨드 | ~10분 |
| 4 | `04_n8n-mattermost-workflow.md` | n8n Mattermost Credential + 콘텐츠생성 워크플로우 | ~3분 |
| 5 | `05_bridge-server-setup.md` | Claude 브릿지 서버 설치 + SSL + n8n 연동 | ~10분 |
| 6 | `06_dashboard-setup.md` | 콘텐츠 승인 대시보드 설치 + SSL + n8n 연동 | ~5분 |

**기본 인프라 총 소요시간**: ~36분

### 기능 확장 (선택/추가)

| 파일 | 내용 | 소요시간 |
|------|------|---------|
| `07_threads-auto-publish.md` | 대시보드 승인 → Threads 자동 발행 연동 | ~5분 |
| `08_image-content-setup.md` | 이미지 첨부 콘텐츠 생성 + Threads 이미지 발행 | ~10분 |
| `07_prompt-type-system.md` | 콘텐츠 유형별 프롬프트 시스템 (일상/지식/홍보) | ~5분 |
| `08_dashboard-ux-loading.md` | 대시보드 승인/거절 로딩 오버레이 UX 개선 | ~2분 |
| `09_dashboard-delete-pagination.md` | 대시보드 삭제 기능 + 페이지네이션 | ~3분 |
| `10_dashboard-republish.md` | 대시보드 Threads 재발행 + 발행실패 탭 | ~3분 |
| `11_trello-issue-workflow.md` | Trello 이슈 관리 워크플로우 연동 | - |
| `12_media-server-setup.md` | 미디어 서버 설치 및 설정 | - |
| `13_media-workflow-setup.md` | 미디어 처리 워크플로우 설정 | - |
| `14_content-image-carousel.md` | 콘텐츠 이미지 캐러셀 기능 | - |

### 워드프레스 연동 (선택)

| 파일 | 내용 |
|------|------|
| `워드프레스/01_wordpress-docker-setup.md` | WordPress Docker 설치 |
| `워드프레스/02_blog-auto-workflow.md` | 블로그 자동화 워크플로우 |
| `워드프레스/03_blog-prompt-parsing-fix.md` | 블로그 프롬프트 파싱 수정 |
| `워드프레스/04_blog-image-upload.md` | 블로그 이미지 업로드 |

**신규 고객 기본 세팅**: `01` → `02` → `03` → `04` → `05` → `06` 순서로 진행
**Threads 발행 추가**: → `07_threads-auto-publish` → `08_image-content-setup`

---

## 사용 방법

1. `CLIENT_INTAKE.md` (고객 온보딩 입력 시트) 를 먼저 작성
2. 해당 가이드 문서를 Claude 대화에 첨부
3. 입력 시트의 값을 함께 전달
4. Claude가 나머지 실행

---

## 고객별 필요 정보

```
AWS Access Key / Secret Key
도메인                        (예: clientname.com)
관리자 이메일
n8n 비밀번호
DB 비밀번호
Anthropic API Key             (sk-ant-...)
Threads 앱 ID / 시크릿 / 액세스 토큰
```

> 상세 입력 양식: `CLIENT_INTAKE.md` 참조
> 환경변수 템플릿 (배치 자동화용): `client.env.template` 참조

---

## 사업 확장 포인트

### 현재 지원 플랫폼
- [x] Threads (자동 발행 — 텍스트 + 이미지)
- [ ] Instagram (API 제한으로 수동 발행 가이드 제공)
- [ ] Facebook
- [ ] Twitter/X

### 현재 지원 콘텐츠 타입
- [x] 텍스트 전용 (슬래시커맨드 `/콘텐츠생성`)
- [x] 이미지+텍스트 (Outgoing Webhook, 직접 이미지 첨부)
- [ ] BGM + 숏폼 영상
- [ ] 광고 소재 자동 수집

### 추가 가능 기능
- [ ] 예약 발행 (n8n Cron 트리거)
- [ ] 성과 분석 대시보드 (Threads Insights API)
- [ ] 슬랙 연동 (Mattermost 대체)
- [ ] 멀티 계정 관리

---

## 비용 구조 (월간, 고객당)

| 항목 | 비용 |
|------|------|
| AWS Lightsail (2vCPU/2GB) | $20/월 |
| 도메인 (.com) | ~$1/월 (연 $12) |
| Anthropic API (Claude) | 사용량 비례 (~$5~30/월) |
| Threads API | 무료 |
| Mattermost | 무료 (Team Edition) |
| n8n | 무료 (Self-hosted) |
| **합계** | **~$26~51/월** |

---

## 알려진 이슈 & 해결 (전체)

| 증상 | 원인 | 해결 |
|------|------|------|
| n8n HTTP Request에서 한글 깨짐 | n8n Docker 인코딩 문제 | 브릿지 서버 경유 (`/publish/threads`) |
| n8n Code 노드 `fetch is not defined` | Task Runner 샌드박스 | HTTP Request 노드 또는 브릿지 서버 사용 |
| n8n `jsonBody` `={{ }}` 미평가 | n8n v2 버그 | `specifyBody: 'keypair'` 방식 |
| n8n에서 `localhost:3100` 불가 | Docker ≠ 호스트 네트워크 | Nginx 도메인 경유 (`https://api.도메인`) |
| n8n 워크플로우 활성화 안 됨 | activeVersionId 버그 | DB 직접 수정 후 재시작 |
| Nginx 설정 `$` 변수 치환 | SSH heredoc 문제 | 로컬 파일 생성 → SCP 전송 |
| 승인 후 상태가 `approved`로 남음 | approve 라우트가 published 덮어쓰기 | published 상태 체크 후 조건부 업데이트 |
| Mattermost 봇 403 Forbidden | 봇이 채널 미가입 | 채널별 멤버 추가 API 호출 |
| PM2 재시작 후 환경변수 없음 | ecosystem.config.js env 미반영 | `pm2 delete` → 환경변수 포함 재시작 |

---

## 서버 관리 명령어 모음

```bash
# SSH 접속
ssh -i [키파일] ubuntu@[서버IP]

# 전체 상태
pm2 status
docker ps

# 로그
pm2 logs bridge-server --lines 30
pm2 logs dashboard --lines 30
cd /opt/n8n && docker compose logs n8n --tail=30
cd /opt/mattermost && docker compose logs mattermost --tail=30

# 재시작
pm2 restart bridge-server
pm2 restart dashboard
cd /opt/n8n && docker compose restart n8n
cd /opt/mattermost && docker compose restart mattermost

# n8n 워크플로우 활성화 상태 확인
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "SELECT name, active, \"activeVersionId\" = \"versionId\" AS ok FROM workflow_entity;"

# Nginx
sudo nginx -t && sudo systemctl reload nginx

# SSL 인증서 갱신 확인
sudo certbot certificates
sudo certbot renew --dry-run

# 대시보드 DB 직접 조회
cd /opt/dashboard && node -e "
  const D=require('better-sqlite3');
  const db=new D('data/dashboard.db');
  console.log(db.prepare('SELECT id,title,status,thread_id FROM contents ORDER BY id DESC LIMIT 10').all());
"
```

---

## 레퍼런스 문서

| 파일 | 내용 |
|------|------|
| `aws-lightsail-n8n-setup-guide.md` | AWS Lightsail + n8n 상세 설치 가이드 (수동) |
| `domain-ssl-setup-guide.md` | 도메인 구매 + DNS + SSL 설정 상세 가이드 (수동) |
| `CLIENT_INTAKE.md` | 신규 고객 온보딩 입력 시트 |
| `client.env.template` | 환경변수 템플릿 (배치 자동화용) |
