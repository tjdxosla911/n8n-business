# n8n 보안 취약점 긴급 패치

> n8n 공식 보안 권고에 따른 긴급 업데이트 (2.11.3 → 2.13.3)

---

## 배경

- n8n에서 보안 취약점 긴급 패치 메일 수신
- 대상 버전: 2.13.3 미만
- 조치: 최신 이미지로 업데이트 필요

---

## 사전 조건

- AWS Lightsail에 Docker Compose로 n8n이 설치되어 있을 것
- SSH 접속 가능할 것

---

## 작업 절차

### 1. 현재 버전 확인

```bash
docker exec n8n n8n --version
```

출력 예시: `2.11.3`

### 2. 컨테이너 및 실행 방식 확인

```bash
docker ps --format '{{.Names}} {{.Image}} {{.Status}}' | grep n8n
cat /opt/n8n/docker-compose.yml
```

- `n8nio/n8n:latest` 이미지 사용 중인지 확인
- named volume (`n8n_data`, `postgres_data`) 사용 중인지 확인

### 3. 최신 이미지 pull

```bash
cd /opt/n8n
docker compose pull n8n
```

- n8n 이미지만 pull (postgres는 변경 없음)

### 4. 컨테이너 재생성

```bash
cd /opt/n8n
docker compose up -d
```

- n8n 컨테이너만 Recreate됨
- postgres 컨테이너: Running 유지
- **데이터 볼륨은 삭제되지 않음** (named volume 방식)

### 5. 업데이트 후 버전 확인

```bash
docker exec n8n n8n --version
```

출력: `2.13.3` 이상이면 패치 완료

### 6. 서비스 정상 동작 확인

```bash
docker ps --format '{{.Names}} {{.Image}} {{.Status}}' | grep n8n
```

- n8n: Up 상태
- n8n-postgres: Up + healthy 상태

웹 브라우저에서 `https://n8n.도메인` 접속하여 로그인 및 워크플로우 정상 동작 확인

---

## 실제 작업 기록

| 항목 | 내용 |
|------|------|
| 작업일 | 2026-03-26 |
| 대상 서버 | AWS Lightsail (3.37.79.43) |
| 패치 전 버전 | 2.11.3 |
| 패치 후 버전 | 2.13.3 |
| 데이터 볼륨 | 정상 유지 (`n8n_data`, `postgres_data`) |
| postgres | 영향 없음 (Up 13 days 유지) |
| 서비스 중단 시간 | ~10초 (컨테이너 재생성 소요) |

---

## 주의사항

- `docker compose down -v` 는 **절대 사용 금지** (볼륨 삭제됨)
- `docker compose up -d` 만 사용하면 볼륨은 안전하게 유지됨
- 업데이트 후 n8n 워크플로우 활성화 상태가 풀릴 수 있으므로 확인 필요:

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "SELECT name, active, \"activeVersionId\" = \"versionId\" AS ok FROM workflow_entity;"
```

- `ok` 가 `f` 인 워크플로우가 있으면 아래 명령으로 복구:

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "UPDATE workflow_entity SET \"activeVersionId\" = \"versionId\" WHERE active = true;"
cd /opt/n8n && docker compose restart n8n
```
