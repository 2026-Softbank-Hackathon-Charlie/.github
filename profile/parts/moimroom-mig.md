# moimroom-mig: 이전 시험과 시연에 쓰는 모임룸 사본 (관측 기능 추가)

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `e4172ae`)

## 이 파트가 하는 일

공간 예약 서비스 "모임룸"을 `moimroom` 레포에서 가져온 사본(레포 설명: "Imported from …/moimroom.git", 운영 가이드는 "fork"라고 부름)입니다. README의 앱 설명, 기술 스택, API, 실행 방법은 `moimroom`과 같고, 이 레포는 이전·전환 시험에 필요한 기능을 더합니다. README에는 HTTP 관측(OpenTelemetry server trace와 로그의 trace_id/span_id 연결)과 AI 안내 호출의 generation span 설명이 추가되어 있습니다. 운영 가이드와 `docs/`에는 DB·외부 호출 계측, 예약 커밋 쓰기 기록(outbox) 수집, 게이트웨이 hold/drain 측정 기록, 원본 기능 동기화 검증 기록이 있습니다. 조직 프로필 README는 이 레포를 **시연 앱**("시연 앱은 예약 서비스 `moimroom-mig`")으로 지정합니다.

### 세 moimroom 레포의 구분

| 레포 | 용도 | 근거 |
|---|---|---|
| `moimroom` | 기본 샘플 앱(원본) | 이 레포의 `docs/operations.md`와 `mig-sync` 기록이 `moimroom`을 "원본"이라 부름 |
| `moimroom-mig` (이 레포) | 원본을 가져와 관측과 이전 시험 기능을 더한 사본이자 시연 앱 | 레포 설명, 조직 README, `docs/test-records/` |
| `moimroom-legacy` | Docker Compose와 Caddy로 서버 한 대에서 운영하는 구성. 용도는 README에 명시되지 않음(확인 필요) | `moimroom-legacy` README 배포 절 |

## 전체 흐름에서의 위치

- 계획 → 리허설 → 승인 → 이전 → 전환 → 검증 흐름에서 **이전·전환되는 실제 앱**입니다. 이전 엔진과 트래픽 게이트웨이가 다루는 대상이고, 이 레포는 그 앱이 이전 중에도 정합성 근거와 관측 신호를 내도록 합니다.
- 검증 단계에 쓰이는 앱 쪽 기능: 예약 생성 멱등 키(이전·복원 후 같은 키 재시도), 커밋된 예약 INSERT의 기본키를 플랫폼에 보고하는 쓰기 기록 수집(선택), 읽기 전용 자격증명 사용 프로브.
- 연결되는 레포: `moimroom`(원본, 기능을 순서대로 이식함), `platform`(게이트웨이, 쓰기 기록 API), `migration-engine`, `frontend`(데모와 이전 화면의 이전 대상). 이 레포 문서는 앱이 OTLP로 trace를 내보내는 부분까지만 다루고(로그는 기존 Alloy, 메트릭은 기존 Prometheus가 수집한다고 적음), 중앙 수집 구성은 이 레포 밖입니다.

## 담당

조직 프로필 README의 RnR 표에는 이 레포가 이름으로 나오지 않습니다. "샘플 앱 개발"은 조서현의 Main 항목이고 RnR에 Sample App이 적혀 있습니다. 관측 구축은 강종욱의 Main("ClickStack·AWS/GKE 메트릭·Langfuse 관측 구축")에 있으나, 이 레포의 어느 부분을 누가 맡았는지는 확인 필요입니다.

## 구조

README의 디렉터리 구조 절은 `moimroom`과 같고, 아래 표의 `internal/telemetry/`, `ops/holdbench/`, `docs/ai/`는 README 구조 절에 없고 레포 파일 목록에서 확인한 것입니다.

| 경로 | 역할 |
|---|---|
| `cmd/moimroom/` | 서버 진입점. `holdbench_profile.go`가 추가로 있음 |
| `internal/api/`, `internal/store/` | JSON API 핸들러, PostgreSQL 쿼리와 스키마·초기 데이터 |
| `internal/spa/`, `web/` | 내장 SPA 제공, React 프런트엔드 |
| `internal/blob/` | S3 호환 저장소 클라이언트 |
| `internal/telemetry/` | OpenTelemetry HTTP·DB·AI·generation 계측 (파일 목록 기준) |
| `internal/writtenrecords/` | 커밋된 예약 쓰기 기록 수집 (파일 목록 기준) |
| `deploy/helm/`, `deploy/cnpg/` | Helm 차트, CloudNativePG 클러스터 예시 |
| `ops/` | k6 부하 테스트, 장애 주입, `holdbench/`(hold 측정 스크립트) |
| `docs/adr/` | ADR 0001~0010 (OTel 추적, 러너, 멱등 예약, outbox, 계측 등) |
| `docs/test-records/` | `2026-11-06-platform-91`, `2026-11-07-mig-sync`, `2026-11-07-gateway-hold` |
| `docs/ai/` | AI 안내 API 계약 JSON(`contract-interface.json` 등). 설명 문서는 README에 없음 |
| `platform.yaml` | 배포 명세 |

## 실행과 테스트

README의 명령은 `moimroom`과 같습니다. Docker와 Docker Compose가 필요하고 http://localhost:8080 에서 열립니다.

```bash
docker compose up --build
```

```bash
cd web
npm ci
npm run dev
```

```bash
go test ./...
MOIMROOM_TEST_DATABASE_URL=<전용 테스트 DB URI> go test ./...
cd web && npm test
```

환경 변수(이름만): `PORT`, `DATABASE_URL`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_PATH_STYLE`, `HOST_CODE`, `ADMIN_TOKEN`, `SITE_REGION_LABEL`, `SEED_SPACES`. 관측용으로 운영 가이드와 문서에 `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_CERTIFICATE`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `AI_GATEWAY_BASE_URL`, 쓰기 기록 수집용으로 `WRITTEN_RECORDS_URL`, `WRITTEN_RECORDS_STREAM`, `WRITTEN_RECORDS_TOKEN`이 적혀 있습니다(쓰기 기록은 세 변수를 모두 설정하거나 모두 비웁니다).

## 연결된 파트와 문서

- [README HTTP 관측 절](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/README.md#http-관측)
- [운영 가이드: OpenTelemetry HTTP trace와 로그 연결, 예약 재시도·쓰기 기록 수집](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/operations.md)
- [쓰기 기록 수집 설정과 적용 순서](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/written-record-collection.md)
- [ADR 0001 opt-in HTTP OpenTelemetry](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/adr/0001-opt-in-http-opentelemetry-tracing.md)
- [ADR 0004 멱등 예약](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/adr/0004-reservation-idempotency.md)
- [ADR 0010 원문 없는 AI generation 관측](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/adr/0010-observe-app-generations-without-content.md)
- [게이트웨이 hold/drain 측정 기록](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/test-records/2026-11-07-gateway-hold/README.md)
- [원본 기능 동기화 검증](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/test-records/2026-11-07-mig-sync/README.md)
- [platform #91 로컬 검증 기록](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/docs/test-records/2026-11-06-platform-91/README.md)
- 같은 앱의 다른 버전: [moimroom](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom), [moimroom-legacy](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-legacy)
- 다른 레포: [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform), [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine), [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend)

## 알려진 한계와 미완

- 코드와 테스트 통과만으로 운영 관측이 연결되었다고 보지 않으며, 실제 중앙 수집을 확인하기 전에는 trace 배포가 완료된 것으로 간주하지 않습니다. Langfuse 수신에는 앱 이미지 배포와 별도 중앙 collector의 앱 전용 exporter 연결이 필요합니다. (README, HTTP 관측)
- OTel이 제공하는 것은 server span과 로그 상관관계, DB·외부 호출 span이며, 브라우저 계측은 자동으로 제공하지 않습니다. (운영 가이드)
- 쓰기 기록 수집은 선택 기능이며 커밋된 예약 INSERT 기본키만 보고합니다. 응답 전달 성공, 모든 CRUD, 행 내용, 전환 시점의 모든 커밋 수집 완료를 보증하지 않습니다. (`docs/written-record-collection.md`)
- 게이트웨이 hold 측정은 쓰기 기록 수집 환경 변수를 설정하지 않은 구성에서 한 것이라 수집이 켜진 구성의 성능은 증명하지 않습니다. (`2026-11-07-gateway-hold`)
- 동기화 검증은 일회용 로컬 인스턴스 시험이어서 운영 RPO/RTO나 환경 간 이전을 증명하지 않습니다. (`2026-11-07-mig-sync`)

## 더 보려면

원문: [moimroom-mig README](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig/blob/main/README.md)
