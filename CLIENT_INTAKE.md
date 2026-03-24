# 고객 온보딩 입력 시트

> 신규 고객 세팅 전 아래 정보를 모두 채운 뒤, 이 파일 + 가이드 문서를 Claude에게 전달하면 됩니다.

---

## A. 사전 준비 (고객 또는 운영자가 직접 완료)

아래 항목은 Claude가 대신할 수 없습니다. 세팅 시작 전 반드시 완료해주세요.

| # | 항목 | 완료 | 비고 |
|---|------|------|------|
| 1 | AWS 계정 생성 + 결제 수단 등록 | [ ] | https://aws.amazon.com |
| 2 | AWS IAM 사용자 생성 + Access Key 발급 | [ ] | Lightsail 권한 필요 |
| 3 | Anthropic API Key 발급 | [ ] | https://console.anthropic.com |
| 4 | 도메인 구매 (가비아 등) | [ ] | |
| 5 | 가비아 DNS A레코드 4개 추가 | [ ] | 아래 DNS 설정 참고 |
| 6 | Meta 개발자 앱 등록 + Threads API 토큰 발급 | [ ] | 아래 Threads 토큰 가이드 참고 |

### DNS A레코드 설정 (가비아 → DNS 관리)

모두 동일한 서버 IP를 가리킵니다. 서버 생성 후 고정 IP가 나오면 설정하세요.

| 호스트 | 타입 | 값 | 용도 |
|--------|------|----|------|
| `n8n` | A | [서버 고정 IP] | n8n 워크플로 |
| `chat` | A | [서버 고정 IP] | Mattermost 채팅 |
| `api` | A | [서버 고정 IP] | Claude 브릿지 서버 |
| `dash` | A | [서버 고정 IP] | 콘텐츠 승인 대시보드 |

### Threads API 토큰 발급 방법

1. https://developers.facebook.com 에서 앱 생성 (타입: Business)
2. 제품 추가 → "Threads API" 선택
3. 사용 사례 → "threads_basic", "threads_content_publish" 권한 추가
4. 설정 → 기본 설정에서 Threads 사용자 ID 확인
5. Graph API Explorer에서 장기 토큰 발급:
   - 단기 토큰 발급 후 → 장기 토큰으로 교환
   - `GET /oauth/access_token?grant_type=th_exchange_token&client_secret={앱시크릿}&access_token={단기토큰}`
6. 발급된 토큰과 사용자 ID를 아래 시트에 기입

---

## B. 고객 정보

```
고객사명             :
담당자 이름          :
담당자 연락처        :
```

---

## C. AWS 자격증명

```
AWS Access Key ID    :
AWS Secret Key       :
AWS 리전             : ap-northeast-2        (기본값: 서울)
```

---

## D. 도메인 정보

```
기본 도메인          :        (예: example.com)
n8n 도메인           :        (예: n8n.example.com)
Mattermost 도메인    :        (예: chat.example.com)
브릿지 API 도메인    :        (예: api.example.com)
대시보드 도메인      :        (예: dash.example.com)
관리자 이메일        :        (SSL 인증서 발급 + 알림용)
```

---

## E. Anthropic API

```
Anthropic API Key    :        (sk-ant-...)
Claude 모델          : claude-sonnet-4-6     (기본값)
```

---

## F. Mattermost 설정

```
관리자 이메일        :        (첫 가입 시 사용할 이메일)
관리자 비밀번호      :        (최소 8자, 영+숫자+특수문자)
팀 이름              :        (예: mycompany)
콘텐츠 채널명        : 콘텐츠생성            (기본값)
```

---

## G. n8n 설정

```
n8n 관리자 이메일    :
n8n 관리자 비밀번호  :
n8n Basic Auth User  : admin                 (기본값)
n8n Basic Auth PW    :        (docker-compose에 설정할 값)
```

---

## H. Threads 발행 설정

```
Threads 사용자 ID    :        (Meta 개발자 콘솔에서 확인)
Threads Access Token :        (장기 토큰)
```

---

## I. 프롬프트 커스터마이징 (선택)

기본 프롬프트는 "AI 자동화 1인 사업가 → 중소기업 대표 대상" 페르소나입니다.
고객에 맞게 변경이 필요하면 아래에 기입하세요.

```
페르소나/역할        :        (기본: AI 자동화로 먹고사는 1인 사업가)
타겟 독자            :        (기본: 제조업 중소기업 대표)
톤앤매너             :        (기본: 1인칭, 담백한 사업가 말투)
주요 키워드          :        (쉼표 구분)
```

---

## 실행 방법

### 전체 신규 세팅 (약 40분)

```
[시트 작성 완료]
     │
     ▼
① Claude에게 시트 + 01 첨부 → 서버 생성 + n8n 설치
     │
     ▼ 서버 고정 IP 출력됨
     │
     ╔══════════════════════════════════════════════════════════╗
     ║  ⚠ 여기서 사람이 직접 작업 (5~10분)                      ║
     ║                                                          ║
     ║  1. 가비아 DNS 관리 접속                                  ║
     ║  2. A레코드 4개 추가 (n8n / chat / api / dash → 서버IP)  ║
     ║  3. DNS 전파 대기 (5~10분)                                ║
     ║  4. 확인: nslookup n8n.도메인 → 서버 IP 응답 확인         ║
     ╚══════════════════════════════════════════════════════════╝
     │
     ▼ DNS 전파 확인 후
② Claude에게 02~10 순서대로 첨부 → 전부 자동 실행
     │
     ▼
[세팅 완료 → 체크리스트 확인]
```

> ① → ② 사이의 DNS 작업만 사람이 하면, 나머지는 전부 Claude가 실행합니다.

### 단계별 개별 실행

특정 기능만 추가할 때는 해당 문서만 첨부하면 됩니다.
예: 프롬프트만 변경 → `07` 첨부, 재발행 기능 추가 → `10` 첨부

---

## 세팅 완료 후 체크리스트

| # | 확인 항목 | 상태 |
|---|----------|------|
| 1 | `https://n8n.도메인` 접속 → 로그인 가능 | [ ] |
| 2 | `https://chat.도메인` 접속 → Mattermost 로그인 가능 | [ ] |
| 3 | Mattermost에서 `/콘텐츠생성 홍보 테스트상품` → 채널에 결과 게시 | [ ] |
| 4 | `https://dash.도메인` → 대시보드에 콘텐츠 표시 | [ ] |
| 5 | 대시보드에서 승인 → Threads 발행 완료 | [ ] |
| 6 | `/콘텐츠생성 일상 테스트주제` → 유형별 프롬프트 동작 | [ ] |
| 7 | `/콘텐츠생성 지식 테스트주제` → 유형별 프롬프트 동작 | [ ] |
| 8 | 대시보드 삭제/페이지네이션 동작 | [ ] |
| 9 | 발행실패 탭 + 재발행 버튼 동작 | [ ] |

---

## 운영 참고

### 정기 확인 필요 항목

| 항목 | 주기 | 확인 방법 |
|------|------|-----------|
| Threads 토큰 만료 | 60일 | 발행 실패 증가 시 토큰 갱신 |
| Let's Encrypt SSL 갱신 | 자동 (90일) | `sudo certbot renew --dry-run` |
| 서버 디스크 용량 | 월 1회 | `df -h` (이미지 업로드 누적) |
| PM2 로그 용량 | 월 1회 | `pm2 flush` |

### 긴급 복구

```bash
# 전체 서비스 상태 확인
pm2 list && docker ps

# Bridge 서버 재시작
pm2 restart bridge-server

# Dashboard 재시작
pm2 restart dashboard

# n8n 재시작
cd /opt/n8n && docker compose restart n8n

# n8n 워크플로 활성화 버그 수정
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET \"activeVersionId\" = \"versionId\" WHERE active = true;"
cd /opt/n8n && docker compose restart n8n
```
