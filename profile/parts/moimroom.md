# moimroom: 기본 샘플 앱 (공간 예약 서비스 원본)

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `d981921`)

## 이 파트가 하는 일

스터디룸, 회의실, 파티룸 같은 작은 공간을 시간 단위로 예약하는 서비스 "모임룸"의 **기준(원본) 레포**입니다. Go 서버 한 개가 빌드한 React SPA를 내장해 제공하고, PostgreSQL 17에 예약 데이터를, S3 호환 오브젝트 스토리지(AWS S3, Google Cloud Storage, MinIO)에 사진을 저장합니다. DB와 파일 저장소를 함께 쓰는 앱이고, README 첫머리에 이전·복원 때 예약 재시도 기록(`app.reservation_retries`)을 함께 보존해야 한다는 계약(멱등 예약)을 둡니다. 이 앱이 왜 만들어졌는지를 밝히는 문장은 README에 없고, 이전 대상이라는 위치는 다른 레포의 문서에서 확인됩니다. 세 moimroom 레포 중 Helm·CloudNativePG 배포와 AWS·GCP 이미지 동시 게시를 갖춘 기본 버전입니다.

### 세 moimroom 레포의 구분

| 레포 | 레포 설명(description) | 용도 | 근거 |
|---|---|---|---|
| `moimroom` (이 레포) | "모임룸: 스터디룸, 회의실, 파티룸을 시간 단위로 예약하는 서비스" | 기본 샘플 앱. 다른 두 레포가 기준으로 삼는 원본 | `moimroom-mig` 문서가 이 레포를 "원본"이라 부름. 프런트엔드 README가 이전 대상 샘플 앱을 `moimroom`으로 적음 |
| `moimroom-mig` | "Imported from …/moimroom.git" | 이 레포를 가져와 관측(OpenTelemetry)과 이전 시험 기록을 더한 사본. 조직 README가 시연 앱으로 지정 | 설명, 조직 README "시연 앱은 … `moimroom-mig`" |
| `moimroom-legacy` | "모임룸 공간 예약 서비스" | 서버 한 대에서 Docker Compose·Caddy로 운영하는 구성. 용도는 README에 명시되지 않음(기존 앱 가져오기 시험용으로 보이나 확인 필요) | README 배포 절, 커밋 이력 |

## 전체 흐름에서의 위치

- 계획 → 리허설 → 승인 → 이전 → 전환 → 검증 흐름에서 **이전되는 앱(워크로드)** 쪽입니다. 플랫폼 코드가 아니라 이전 대상입니다.
- `platform.yaml`(서비스, 포트 8080, 헬스 체크, DB와 버킷 요구)로 플랫폼이 앱을 빌드·배포할 수 있게 합니다. 실제 환경별 주입 변수는 README의 platform.yaml 절을 따릅니다.
- 앞뒤 연결: `platform`(배포·게이트웨이), `migration-engine`(DB·파일 이전), `frontend`(데모 모드의 이전 대상이 이 앱의 구성을 따름). `infra`와의 연결은 확인 필요입니다. 이 레포의 `docs/test-records/2026-11-06-platform-91`에 "platform #91 로컬 검증" 기록(백업·복원 뒤 같은 키 재시도 확인)이 있습니다.

## 담당

조직 프로필 README의 RnR 표에는 `moimroom` 레포가 이름으로 나오지 않습니다. "샘플 앱 개발"은 조서현의 Main 항목이고 RnR에 Sample App이 적혀 있으나, 이 레포 단위 담당은 확인 필요입니다.

## 구조

| 경로 | 역할 |
|---|---|
| `cmd/moimroom/` | 서버 진입점 |
| `internal/api/` | JSON API 핸들러 |
| `internal/store/` | PostgreSQL 쿼리, 스키마(`schema.sql`), 초기 데이터(`seed.sql`) |
| `internal/spa/` | 내장 SPA 제공 |
| `internal/blob/` | S3 호환 저장소 클라이언트 |
| `internal/httpx/` | 요청 ID, 로그, 헬스 체크, 종료 처리 |
| `internal/fault/` | 운영용 장애 주입 |
| `web/` | React 프런트엔드 |
| `deploy/helm/`, `deploy/cnpg/` | Helm 차트, CloudNativePG 클러스터 예시 |
| `ops/` | 부하 테스트(k6)와 장애 주입 스크립트 |
| `docs/` | 운영 가이드, ADR, 쓰기 기록 수집 문서, 테스트 기록 |
| `platform.yaml` | 배포 명세 (서비스, 포트, 헬스 체크, DB와 버킷) |

## 실행과 테스트

README의 명령입니다. Docker와 Docker Compose가 필요하고, 실행하면 http://localhost:8080 에서 사이트가 열립니다(PostgreSQL, MinIO, 앱이 함께 뜨고 공간 16곳이 등록됨).

```bash
docker compose up --build
```

프런트엔드 개발(Node.js 24, compose를 띄워 둔 상태):

```bash
cd web
npm ci
npm run dev
```

테스트(DB 테스트는 `app` 스키마를 지우고 다시 만들므로 전용 데이터베이스를 씁니다):

```bash
go test ./...
MOIMROOM_TEST_DATABASE_URL=<전용 테스트 DB URI> go test ./...
cd web && npm test
```

이미지 빌드와 Helm 설치는 README의 배포 절을 따릅니다. 환경 변수(이름만): `PORT`, `DATABASE_URL`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_PATH_STYLE`, `HOST_CODE`, `ADMIN_TOKEN`, `SITE_REGION_LABEL`, `SEED_SPACES`. 이미지 게시 workflow는 Actions 변수 `AWS_IMAGE_PUBLISH_ROLE_ARN`, `AWS_ECR_REPOSITORY`, `GCP_WORKLOAD_IDENTITY_PROVIDER`, `GCP_IMAGE_PUBLISH_SERVICE_ACCOUNT`, `GCP_IMAGE_REPOSITORY`를 읽습니다.

## 연결된 파트와 문서

- [README 설정 절](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/README.md#설정): 환경 변수 표
- [README 배포 절](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/README.md#배포): 이미지 빌드, 자동 게시, platform.yaml, Helm
- [운영 가이드](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/docs/operations.md): 헬스 체크, 부하 테스트, 장애 주입, 데이터 정합성 확인
- [ADR 0001 멱등 예약](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/docs/adr/0001-reservation-idempotency.md)
- [쓰기 기록 수집](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/docs/written-record-collection.md)
- [platform #91 로컬 검증 기록](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/docs/test-records/2026-11-06-platform-91/README.md)
- [platform.yaml](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/platform.yaml)
- 같은 앱의 다른 버전: [moimroom-mig](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig), [moimroom-legacy](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy)
- 다른 레포: [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform), [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine)

## 알려진 한계와 미완

- `HOST_CODE`와 `ADMIN_TOKEN`은 비밀값이라 `platform.yaml`에 넣지 않으므로, 이 명세로 배포하면 호스트 API와 운영 API는 꺼져 있습니다. (배포 > platform.yaml)
- 멱등 예약은 `Idempotency-Key`를 보낸 예약 생성에만 적용되고 다른 쓰기 endpoint에는 적용되지 않으며, 이전 양쪽 앱 버전이 이 계약을 지원해야 합니다. (README 첫머리)
- 앱 자동 배포는 이미지 게시와 별도 단계이며 이 README에는 적혀 있지 않습니다. (배포 > 이미지 자동 게시)
- 그 밖에 README에 "미완"으로 적힌 항목은 없습니다.

## 더 보려면

원문: [moimroom README](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom/blob/main/README.md)
