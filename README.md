# HyperCube 배포 번들

[HyperCube](https://github.com/qkr7287/HyperCube) 애플리케이션을 **Docker만 설치된 호스트 어디서든** 한 번에 올리기 위한 배포 전용 저장소입니다.

- 앱 소스 코드와 빌드 파이프라인은 (private한) 본 레포에 있음
- 이 레포에는 **`docker-compose.yml` + `.env.example` + 가이드**만 존재
- 실제 실행 이미지는 GHCR의 public 패키지

## 필요한 이미지 (자동으로 당겨짐)

| 이미지 | 역할 | 출처 |
|---|---|---|
| `ghcr.io/qkr7287/hypercube-backend` | Django + Celery 베이스 | GHCR public |
| `ghcr.io/qkr7287/hypercube-nginx`   | SvelteKit SPA + nginx  | GHCR public |
| `pgvector/pgvector:pg16`            | PostgreSQL + pgvector  | Docker Hub |
| `redis:7-alpine`                    | Redis                  | Docker Hub |

모두 public이라 **`docker login` 필요 없음**.

---

## 타겟 호스트 요구사항

- Docker Engine 24 이상
- Docker Compose v2 (`docker compose version` 확인)
- `ghcr.io`, `docker.io` 아웃바운드 허용

그게 전부. git, Node, Python 전혀 필요 없음.

---

## 신규 설치 (완전히 빈 호스트에)

```bash
# 1) 배포 폴더 만들기
sudo mkdir -p /opt/hypercube
sudo chown "$USER":"$USER" /opt/hypercube
cd /opt/hypercube

# 2) 이 레포에서 파일 2개만 다운로드
curl -fLO https://raw.githubusercontent.com/qkr7287/hypercube-deploy/main/docker-compose.yml
curl -fL  https://raw.githubusercontent.com/qkr7287/hypercube-deploy/main/.env.example -o .env

# 3) .env 시크릿 채우기 (아래 "필수 .env 값" 참고)
nano .env

# 4) 이미지 당기고 기동
docker compose pull
docker compose up -d

# 5) 첫 부팅 로그 확인 (마이그레이션 자동 실행됨)
docker compose logs -f backend
# "Uvicorn running on ..." 나오면 Ctrl+C로 빠져나오기

# 6) 첫 관리자 계정
docker compose exec backend python manage.py createsuperuser
```

접속: `http://<호스트>:${HC_PORT:-7003}/hypercube`

---

## 필수 `.env` 값

타겟 호스트에서 랜덤 키 생성:

```bash
echo "DJANGO_SECRET_KEY=$(openssl rand -base64 50 | tr -d '=\n/' | cut -c1-50)"
echo "DB_PASSWORD=$(openssl rand -base64 32 | tr -d '=\n/' | cut -c1-32)"
```

출력된 값을 `.env`에 붙여넣으세요.

| 변수 | 채워 넣을 값 |
|---|---|
| `DJANGO_SECRET_KEY` | 위 명령 결과물 (랜덤 50자) |
| `DB_PASSWORD` | 위 명령 결과물 (랜덤 32자) |
| `DJANGO_ALLOWED_HOSTS` | 접속할 호스트/IP 콤마 구분, 예: `hypercube.example.com,192.168.0.16` |

선택 설정:

| 변수 | 기본값 | 설명 |
|---|---|---|
| `IMAGE_TAG` | `latest` | 재현 가능한 배포용 고정 태그 (예: `sha-daa5dc7`) |
| `HC_PORT`   | `7003`   | 외부에서 접속할 포트 |
| `CORS_ALLOWED_ORIGINS`  | 빈값 | 예: `https://hypercube.example.com` |
| `CORS_ALLOW_ALL_ORIGINS` | `false` | 초기 테스트 시 잠깐 `true` |
| `CSRF_TRUSTED_ORIGINS`  | 빈값 | UI를 서빙하는 스키마+호스트 |

---

## 업데이트 (같은 호스트에서 새 버전으로)

```bash
cd /opt/hypercube
docker compose pull
docker compose up -d
```

Django 마이그레이션은 backend 컨테이너 시작 시 자동 실행. 스키마 변경이 큰 릴리스 전에는 백업을 권장:

```bash
docker compose exec postgres pg_dump -U hypercube hypercube \
  > backup-$(date +%F).sql
```

---

## 롤백 (특정 버전으로 되돌리기)

main에 push되는 모든 이미지는 커밋 SHA 태그가 함께 붙습니다. `.env` 수정으로 고정:

```
IMAGE_TAG=sha-daa5dc7
```

적용:

```bash
docker compose pull
docker compose up -d
```

---

## 올라오는 서비스 (컨테이너 6개)

| 컨테이너 | 이미지 | 역할 |
|---|---|---|
| `hc-nginx`         | `ghcr.io/qkr7287/hypercube-nginx:<tag>`   | SPA 정적 서빙 + 리버스 프록시 |
| `hc-backend`       | `ghcr.io/qkr7287/hypercube-backend:<tag>` | Django (uvicorn, ASGI) |
| `hc-celery-worker` | 위 backend 이미지 동일, command 다름      | 백그라운드 작업 워커 |
| `hc-celery-beat`   | 위 backend 이미지 동일, command 다름      | 정기 작업 스케줄러 |
| `hc-postgres`      | `pgvector/pgvector:pg16` (Docker Hub)     | Postgres 16 + pgvector |
| `hc-redis`         | `redis:7-alpine` (Docker Hub)             | 채널 레이어 + 캐시 + Celery 브로커 |

> backend 이미지 하나가 3개 역할(uvicorn / celery worker / celery beat)을 같은 코드베이스로 수행합니다. 공간 절약 + 버전 일치 보장.

---

## 제거

```bash
docker compose down        # 컨테이너 중단 + 삭제, DB 볼륨 유지
docker compose down -v     # 위 + Postgres 볼륨까지 삭제 (데이터 날아감)
```

---

## 자주 나는 문제

| 증상 | 원인 / 해결 |
|---|---|
| `docker compose pull` → `denied` / `403` | GHCR 이미지가 private으로 바뀌어 있음. Profile → Packages → `hypercube-*` → Package settings → **visibility = Public** 확인 |
| backend가 `up -d` 직후 재시작 반복 | `.env`의 `DJANGO_SECRET_KEY` 또는 `DB_PASSWORD` 빠짐. `docker compose logs backend`에 원인 뜸 |
| 브라우저에서 `/hypercube` 404 | SPA 마운트 경로가 `/hypercube`라서 `/`는 의도적으로 404. 꼭 `http://host:7003/hypercube`로 접속 |
| 해당 포트 응답 없음 | 다른 프로세스가 포트 점유 중 (`ss -tlnp \| grep 7003`로 확인). `.env`에서 `HC_PORT` 바꾸고 `docker compose up -d` |
| 완전 초기화하고 싶음 | `docker compose down -v` 후 `.env` 수정 → `docker compose up -d` (DB 새로 초기화됨) |

---

## 이슈 리포팅

앱 관련 버그는 소스 레포에 올려주세요:
<https://github.com/qkr7287/HyperCube/issues> (private, 접근 권한 필요)

배포 번들(이 레포의 compose/env/README)만의 문제는:
<https://github.com/qkr7287/hypercube-deploy/issues>
