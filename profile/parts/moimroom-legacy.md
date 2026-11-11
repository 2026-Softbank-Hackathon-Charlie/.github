# moimroom-legacy: 서버 한 대에서 Docker Compose로 운영하는 모임룸 구성

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `0c649ed`)

## 이 파트가 하는 일

스터디룸, 회의실, 파티룸을 시간 단위로 예약하는 "모임룸" 앱을 **운영 서버 한 대**에서 Docker Compose로 띄우는 구성입니다. 앱(Go 서버 + 내장 React SPA), PostgreSQL, Caddy(HTTPS 리버스 프록시)가 한 compose 파일에 있고, 사진은 AWS S3 버킷에 저장합니다. `main`에 push하면 GitHub Actions가 서버에 ssh로 접속해 `docker compose up -d --build`로 배포합니다. 앱 코드와 기능 설명은 `moimroom`과 같고, 다른 점은 배포 방식입니다(Helm·CloudNativePG·MinIO·다중 레플리카가 없고 Compose·Caddy·S3 단일 구성). 이 레포를 만든 목적은 README에 적혀 있지 않습니다.

### 세 moimroom 레포의 구분

| 레포 | 용도 | 근거 |
|---|---|---|
| `moimroom` | 기본 샘플 앱(원본). Helm·CNPG 배포 | 레포 설명, README 비교 |
| `moimroom-mig` | 원본 사본에 관측과 이전 시험 기록을 더한 시연 앱 | 레포 설명("Imported from …"), 조직 README |
| `moimroom-legacy` (이 레포) | 한 서버에서 Compose로 운영하는 구성. "기존 앱 가져오기" 시험의 입력으로 보이나 README에는 없어 확인 필요 | README 배포 절, `platform.yaml`, `hopcharlie.Dockerfile`, 커밋 메시지 |

"기존 앱 가져오기" 시험용이라는 추정의 근거는 세 가지입니다. 이 레포의 두 번째 커밋 메시지가 "hopcharlie 계약 파일 추가 (onb-…)"이고(`platform.yaml`, `hopcharlie.Dockerfile` 추가), `platform.yaml`의 이름이 `moimroom-legacy`이며, `frontend` README가 가져오기 흐름을 "플랫폼 밖(EC2, RDS, S3)에서 돌던 앱"을 점검해 클러스터에 첫 배포하는 것으로 설명하기 때문입니다. 이 레포의 README 자체는 이 용도를 언급하지 않으므로 확정하지 않습니다. infra 레포의 시험용 VM 초안 PR(#143, 아직 머지되지 않음)이 이 앱을 platform ADR 0042(compose로 돌던 앱 가져오기)를 시험하기 위한 고객 앱 대역으로 설명하고 있어 위 추정과 일치합니다.

## 전체 흐름에서의 위치

- README 기준으로는 이전 흐름과 직접 연결된 설명이 없습니다. 플랫폼 밖에서 운영되는 앱의 한 예(Compose + Caddy + S3)라는 점만 읽힙니다.
- 추가로 들어 있는 `platform.yaml`(서비스 `app`, 포트 8080, 헬스 체크 `/healthz`, DB `db`와 버킷 `photos` 요구)과 `hopcharlie.Dockerfile`은 HopCharlie 플랫폼이 이 앱을 빌드·배포하기 위한 파일로 보이지만, README 설명이 없어 확인 필요입니다.
- 연결 가능성이 있는 레포: `platform`(배포 명세 소비), `frontend`(기존 앱 가져오기 화면). 둘 다 이 레포의 문서에서 직접 확인되지는 않았습니다.

## 담당

조직 프로필 README의 RnR 표에는 이 레포가 이름으로 나오지 않습니다. "샘플 앱 개발"은 조서현의 Main 항목이고 RnR에 Sample App이 적혀 있으나, 이 레포 단위 담당은 확인 필요입니다.

## 구조

README의 디렉터리 구조 절에 `docs/`, `deploy/`, `ops/`는 없고 레포에도 없습니다.

| 경로 | 역할 |
|---|---|
| `cmd/moimroom/` | 서버 진입점 |
| `internal/api/` | JSON API 핸들러 |
| `internal/store/` | PostgreSQL 쿼리, 스키마(`schema.sql`), 초기 데이터(`seed.sql`) |
| `internal/spa/` | 내장 SPA 제공 |
| `internal/blob/` | S3 클라이언트 |
| `internal/httpx/`, `internal/fault/` | 요청 ID·로그·헬스 체크·종료 처리, 운영용 장애 주입 |
| `web/` | React 프런트엔드 |
| `compose.yaml` | 운영 서버 구성 (app, db, caddy) |
| `Caddyfile` | HTTPS와 리버스 프록시 설정 |
| `.env.example` | 환경 변수 예시(값 없이 형식만) |
| `.github/workflows/` | main 브랜치 자동 배포(`deploy.yml`) |
| `platform.yaml`, `hopcharlie.Dockerfile` | 플랫폼 배포 명세와 이미지 빌드 파일. README에는 설명 없음(확인 필요) |

## 실행과 테스트

README의 명령입니다. Docker와 Docker Compose가 필요하고, 사진을 S3 버킷에 저장하므로 개발용 버킷과 접근 키를 준비합니다.

```bash
cp .env.example .env
# .env에서 SITE_DOMAIN=localhost 로 두고, S3_BUCKET과 접근 키를 개발용 값으로 채웁니다.
docker compose up --build
```

https://localhost 에서 열리며(`SITE_DOMAIN`이 `localhost`면 Caddy 자체 인증서라 브라우저 경고가 한 번 뜹니다), 처음 시작할 때 공간 16곳이 등록됩니다. 프런트엔드 개발(Node.js 24, Go 서버를 따로 띄운 상태)과 테스트:

```bash
go run ./cmd/moimroom
cd web
npm ci
npm run dev
```

```bash
go test ./...
MOIMROOM_TEST_DATABASE_URL=<전용 테스트 DB URI> go test ./...
cd web && npm test
```

환경 변수(이름만): `SITE_DOMAIN`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `PORT`, `DATABASE_URL`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_PATH_STYLE`, `HOST_CODE`, `ADMIN_TOKEN`, `SITE_REGION_LABEL`, `SEED_SPACES`. 자동 배포용 Actions secret 이름은 `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`(선택), `APP_ENV`입니다.

## 연결된 파트와 문서

- [README 배포 절](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/README.md#배포): 자동 배포 절차와 필요한 secret, 서버 요건(80·443 포트, 도메인 A 레코드)
- [compose.yaml](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/compose.yaml)
- [Caddyfile](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/Caddyfile)
- [배포 workflow](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/.github/workflows/deploy.yml)
- [.env.example](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/.env.example)
- [platform.yaml](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/platform.yaml)
- 같은 앱의 다른 버전: [moimroom](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom), [moimroom-mig](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig)
- 다른 레포: [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend)

## 알려진 한계와 미완

- 배포 때 서버의 `.env`는 `APP_ENV` secret 내용으로 새로 쓰며, DB 볼륨은 그대로 남습니다. `DEPLOY_KNOWN_HOSTS`가 없으면 처음 접속한 호스트 키를 받아들입니다. (배포)
- 그 밖에 README에 한계나 미완으로 적힌 항목은 없습니다.

## 더 보려면

원문: [moimroom-legacy README](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy/blob/main/README.md)
