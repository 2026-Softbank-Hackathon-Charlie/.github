# platform: 이전 계획의 승인과 실행, 트래픽 전환을 맡는 HopCharlie 서버

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `5ac0195`)

## 이 파트가 하는 일

platform은 HopCharlie의 서버 쪽 중심입니다. 플랫폼 API(`cmd/api`)와 트래픽 게이트웨이(`cmd/gateway`) 두 바이너리로 구성된다고 README가 설명합니다. 플랫폼 API는 AI 플래너가 낸 이전 Plan을 접수해 보안 정책을 검사하고, 운영자의 승인을 받은 뒤 이전 실행(run)의 단계와 상태를 관리하며, 진행 이벤트(SSE)와 감사 로그를 남깁니다. 게이트웨이는 사용자 요청을 받아 쓰기는 한 환경으로만 보내고, 이전 중에는 쓰기를 보류하고 환경을 바꾸며 읽기 비율과 요청 통계를 다룹니다. 이 밖에 GitHub 앱 등록·배포 실행기(이미지 빌드, gitops 커밋, Argo CD 대기, 게이트웨이 등록), 앱 시크릿, 보안 점검 뷰, 에이전트 활동 기록 API도 이 레포에 있습니다. API 계약(`api/openapi.yaml`)은 서버, CLI, 콘솔이 함께 따르는 단일 기준입니다.

## 전체 흐름에서의 위치

- **계획**: AI 플래너가 agent 토큰으로 `POST /v1/plans`에 Plan을 제출합니다. 플랫폼은 보안 정책을 검사하고 `pending_approval`로 둡니다. 플래너 쪽 구현은 `ai-sandbox` 레포입니다. 이 README는 리허설 보고서와 Plan 승인의 관계를 직접 다루지 않습니다. 이 연결은 platform 이슈 #24(AI 제출 Plan 승인에 입력과 일치하는 리허설 증거 요구)로 추적했고 2026-11-05에 닫혔지만, 구현의 세부는 README가 아니라 코드와 ADR로 확인해야 합니다.
- **승인**: operator만 Plan을 승인·거절합니다. agent 토큰으로는 승인할 수 없습니다. 첨부가 있는 Plan은 검토한 `revision`을 보내야 승인됩니다.
- **이전**: 승인된 Plan으로 `POST /v1/migrations`를 실행하면 run이 엔진(`migration-engine`)을 호출해 precheck, prepare, presync, delta를 진행합니다. 카나리를 켜면 게이트웨이가 읽기 일부를 대상으로 보내고 단계마다 오류율과 p95로 판정합니다.
- **전환**: 데이터가 따라잡히면 `awaiting_switch`에서 운영자의 `/switch`를 기다립니다(`autoSwitch`면 생략). 엔진이 fence부터 finalize까지 실행하고 게이트웨이가 hold, route, release를 수행합니다.
- **검증**: 엔진의 verify 결과와 쓰기 기록(`written-records`)을 run에 반영하고, `observe`를 켜면 전환 뒤에도 대상을 관찰합니다. 위반 시 `autoRollback`이면 1클릭 롤백과 같은 경로로 되돌립니다.
- **앞뒤 레포**: `cli`(이 API의 클라이언트, `pkg/client`와 `pkg/apiv1`을 import), `frontend`(콘솔), `migration-engine`(단계 실행), `infra`(관리 호스트와 환경의 Terraform, 운영 설정), `gitops`(배포 선언 저장소).

## 담당

조직 프로필 README의 RnR 표 기준입니다.

| 이름 | 표에 적힌 역할 |
|---|---|
| 조서현 | Main: 플랫폼 API·트래픽 게이트웨이, 앱 배포, 운영 콘솔 실제 API 연동, 플랫폼·콘솔 이미지 게시 자동화 |
| 김동윤 | 표에 "보안 API·콘솔"이 있으나 이 레포의 `/v1/security`를 가리키는지는 확인 필요 |
| 박준석 | 표에 "근거 기반 AI 이전 추천"이 있으나 이 레포의 추천 기능(`internal/recommend`)과의 관계는 확인 필요 |

나머지 팀원이 이 레포에서 맡은 부분은 표에서 확정할 수 없어 확인 필요입니다.

## 구조

README의 트리와 레포 디렉터리 목록을 함께 정리했습니다.

| 경로 | 역할 |
|---|---|
| `api/openapi.yaml` | API 계약. 서버가 `GET /openapi.yaml`로 같은 파일을 제공 |
| `cmd/api/` | 플랫폼 API: Plan 접수, 승인, 이전 실행, 이벤트 스트림, 환경, 감사 로그 |
| `cmd/gateway/` | 트래픽 게이트웨이: 쓰기 보류, 환경 전환, 읽기 가중치, 요청 통계 |
| `cmd/onprem-agent/` | README에 설명 없음. 확인 필요 |
| `pkg/apiv1/`, `pkg/client/`, `pkg/appspec/` | OpenAPI에서 생성한 Go 타입, Go 클라이언트, `platform.yaml` 해석·검사. CLI가 import |
| `internal/api/`, `auth/`, `plans/`, `runs/` | HTTP 핸들러·역할·감사·SSE, 토큰과 역할, Plan 검증·승인, 이전 실행 단계 |
| `internal/envs/`, `gateway/`, `store/` | 환경 등록과 클러스터 상태, 게이트웨이 본체, PostgreSQL 저장소 |
| `internal/deploy/` | GitHub 앱 배포 실행기(빌드, gitops 커밋, Argo CD 대기, 게이트웨이 등록) |
| `internal/preflight/` | 이전 전 동등성·레지던시 검사(자체 README 있음) |
| `deploy/`, `examples/` | 앱 차트·Argo CD·AWS 배포 도우미, 로컬 개발용 환경·앱 설정 예시 |
| `test/e2e/`, `docs/adr/`, `docs/e2e/` | kind 기반 E2E, 설계 결정 기록, 종합 E2E 검증 기록 |

README 트리에 없는 디렉터리(`internal/recommend`, `secrets`, `security` 등)가 레포에 더 있습니다. 상세는 README가 아니라 코드와 ADR로 확인해야 합니다.

## 실행과 테스트

엔진 레포가 private이라 Go 모듈 설정이 한 번 필요합니다. 로컬에서는 PostgreSQL을 띄우고 API와 게이트웨이를 실행합니다.

```bash
go env -w GOPRIVATE=github.com/2026-Softbank-Hackathon-Charlie
docker run -d --name charlie-pg -e POSTGRES_USER=charlie -e POSTGRES_PASSWORD=<로컬 비밀번호> -p 55432:5432 postgres:17-alpine

export PLATFORM_DATABASE_URL='<로컬 PostgreSQL 접속 URL>'
export PLATFORM_TOKENS='<이름:역할:토큰, 쉼표 구분>'
export PLATFORM_ENVS_FILE=examples/envs.yaml
export PLATFORM_GATEWAY_URL=http://127.0.0.1:9901 GATEWAY_TOKEN=<게이트웨이 관리 토큰>
export SRC_PGPASSWORD=... DST_PGPASSWORD=...
go run ./cmd/api

export GATEWAY_ADMIN_TOKEN=$GATEWAY_TOKEN
export GATEWAY_BACKENDS='<이름=URL, 쉼표 구분>'
go run ./cmd/gateway
```

- 필수 환경 변수: `PLATFORM_DATABASE_URL`, `PLATFORM_ENVS_FILE`. 인증은 `PLATFORM_TOKENS` 또는 `PLATFORM_ADMINS`가 필요하고 둘 다 비면 `-insecure-no-auth` 없이는 시작하지 않습니다. 전체 목록은 README의 "플랫폼 API 환경 변수" 표를 봅니다.
- 관리자 비밀번호 해시는 `go run ./cmd/api -hash-password`로 만듭니다.

```bash
go test ./...
PLATFORM_TEST_DATABASE_URL='<PostgreSQL 접속 URL>' go test ./internal/store/...
go generate ./pkg/apiv1      # api/openapi.yaml을 고친 뒤
test/e2e/run.sh              # 부하 중 이전 E2E (docker, kubectl, go 필요)
test/e2e/deploy/run.sh       # 배포 E2E (kind의 Argo CD)
```

CI는 PR과 main push마다 vet, gofmt, 생성 코드 확인, `go test -race`를 돌립니다. private 모듈을 받기 위해 레포 secret `MODULE_READ_TOKEN`을 씁니다(README "CI").

## 연결된 파트와 문서

- 원문 README 절: [이전 한 번의 흐름](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md#이전-한-번의-흐름) · [API](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md#api) · [게이트웨이](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md#게이트웨이) · [GitHub 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md#github-앱-배포)
- API 계약: [api/openapi.yaml](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/api/openapi.yaml) · [ADR 0001 단일 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0001-openapi-single-contract.md)
- 핵심 ADR: [0003 역할과 감사](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0003-roles-and-audit.md) · [0004 트래픽 게이트웨이](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0004-traffic-gateway.md) · [0005 run 단계](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0005-run-phases.md) · [0009 전환 후 관찰](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0009-post-cutover-observe.md) · [0011 GitHub 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0011-github-app-deploy.md) · [0041 쓰기 증거 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0041-committed-write-receipts.md)
- 검사와 검증 기록: [internal/preflight README](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/internal/preflight/README.md) · [부하 중 이전 E2E](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/test/e2e/README.md) · [2026-11-08 종합 E2E 기록](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/e2e/2026-11-08/README.md)
- AWS 배포 도우미: [deploy/aws/README.md](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/deploy/aws/README.md)
- 다른 레포: [cli](https://github.com/2026-Softbank-Hackathon-Charlie/cli) · [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine) · [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend) · [ai-sandbox](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox) · [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra) · [gitops](https://github.com/2026-Softbank-Hackathon-Charlie/gitops)

## 알려진 한계와 미완

- 게이트웨이는 한 대뿐이라 단일 장애점입니다(README "알려진 한계", [ADR 0026](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0026-gateway-single-replica.md)).
- 관리자 변경에는 서버 재시작이 필요하고, 비밀번호만 바꾸면 이미 나간 세션과 CLI 토큰은 만료까지 유효합니다. 운영 환경에는 OIDC 같은 IdP 연동이 필요하다고 적혀 있습니다(README "알려진 한계").
- CLI 토큰 하나만 끊는 API가 없어 플랫폼 DB의 행을 직접 지워야 합니다(README "알려진 한계").
- 게이트웨이를 거치지 않고 DB에 쓰는 클러스터 내부 작업은 전환 중에 막을 수 없어, 이전하는 동안 멈춰야 합니다(README "알려진 한계").
- 온프렘 연결 코드는 등록 worker가 없어 인증 후 claimed/검증 미관측 상태로 남고, 자동 Argo·SSM 등록을 뜻하지 않습니다(README "온프렘 연결 코드", [ADR 0049](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0049-one-use-onprem-bootstrap-enrollments.md)).
- 동등성 검사는 기록만 하고 막지 않으며, 부분 검사 통과가 시크릿 사용·감사 전달·복구 성공을 뜻하지 않습니다(README 상단, [ADR 0029](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0029-parity-resource-evidence.md)).
- 종합 E2E 기록은 DB와 파일 시험을 독립 실행으로 구분하며, DB·파일 동시 이전과 AWS↔GCP↔실제 노트북 이전 성공으로 합산하지 않는다고 밝힙니다(2026-11-08 종합 E2E 기록, 범위와 기준).
- 임시 DB 전체 이전 시험은 클라우드 네트워크·운영 이전·파일 동시 이전을 미검증으로 표시합니다(같은 문서, 실행 결과 표).
- README "알려진 한계"에는 "실제 클러스터에서 끝까지 이전해 본 것은 아직 없다"는 문장이 있습니다. 이후 작성된 E2E 기록과 함께 읽어야 하며, 현재도 유효한지는 확인 필요입니다.

## 더 보려면

원문 README: https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md
