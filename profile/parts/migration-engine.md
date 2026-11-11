# migration-engine: DB(CNPG)와 파일(S3)을 다른 클러스터로 옮기는 이전 엔진

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `ba2322b`)

## 이 파트가 하는 일

가동 중인 서비스의 DB(CNPG)와 파일(S3 계열 스토리지)을 다른 클러스터로 옮기는 HopCharlie의 이전 엔진입니다. AI가 만든 Plan을 사람이 승인하면, 엔진이 정해진 순서대로만 실행하고 누락 0건을 증거로 남깁니다. AI는 Plan만 만들고, 클러스터를 바꾸는 작업은 이 엔진만 합니다. 실행 상태를 단계마다 파일에 저장해 재시작 후 이어서 실행할 수 있고, 실패 시 어디까지 자동 원복하고 어디서부터 사람이 판단하는지 경계가 정해져 있습니다. 이전이 끝난 뒤에는 옛 앱을 멈추고 의존성이 남았는지 확인하는 정리(decommission) 단계도 있습니다.

## 전체 흐름에서의 위치

계획 → 리허설 → 승인 → 이전 → 전환 → 검증 중 **승인된 Plan을 받아 이전, 전환, 검증을 실행**하는 부분입니다. 엔진은 CLI(`hopmigrate`)로 쓰거나, 플랫폼 API가 라이브러리로 가져다 씁니다.

- 앞: Plan을 만드는 AI 쪽은 ai-sandbox(조직 프로필 README의 설명)입니다. 승인과 실행 요청은 platform과 콘솔(frontend)을 거칩니다.
- 옆: 쓰기 보류와 트래픽 전환은 게이트웨이를 통해 합니다(platform의 트래픽 게이트웨이, 프로필 README 근거). 대상 환경의 CNPG 설정은 infra 쪽 약속입니다(`docs/details.md`의 "A. 인프라").
- 구간: 점검(precheck, prepare) → 동기화(presync, delta) → 전환(fence, switchover, cutover) → 증명(verify, finalize). 상세 문서에는 cold 앱용 `start-target` 단계가 switchover와 cutover 사이에 있습니다.

## 담당

조직 프로필 README 기준입니다.

| 이름 | 맡은 역할 |
| --- | --- |
| 박준석 | Main: 마이그레이션 엔진·롤백·데이터 검증, 3개 환경 DB·파일 이전 |
| 김동윤 | Assist: 이전 엔진 안전성 검증 |
| 조서현 | Assist: 이전 엔진 연동 |
| 강종욱 | Assist: 이전 환경의 CNPG 호환성 |

## 구조

| 경로 | 역할 |
| --- | --- |
| `cmd/hopmigrate` | CLI |
| `pkg/plan` | Plan 스키마, 기본값, 검증 |
| `pkg/engine` | 단계 실행기, 상태 저장·재개, 원복 경계, 검증 리포트 |
| `pkg/cnpg`, `pkg/pg` | CNPG Cluster 조회·apply·demote/promote, LSN·복제 지연 |
| `pkg/files` | rclone sync/check |
| `pkg/gateway` | 게이트웨이 클라이언트(HTTP / none) |
| `pkg/verify` | 테이블 해시 비교, 누락 행 검사, k6 기록 대조 |
| `pkg/decommission`, `pkg/envcheck`, `pkg/loadguard` | 원본 정리(ADR 0010), 환경 준비 점검(ADR 0014), 운영 DB 부하 예산 가드(ADR 0016). `pkg/kube`의 역할은 확인 필요 |
| `examples/` | Plan 예시와 CNPG 예시 |
| `test/integration`, `test/rehearsal` | PostgreSQL 통합 테스트, kind 기반 왕복 이전·원복 리허설 |
| `docs/details.md` | 단계별 동작, 다른 팀과의 약속, 알려진 한계 |
| `docs/adr/` | 설계 결정 기록(0001~0023) |
| `docs/onprem.md`, `docs/db-load-budget.md` | 온프렘 대상 준비 순서, 운영 DB 부하 예산 |
| `.github/branch-protection/main.json` | main 머지 조건 설정 요청 |

## 실행과 테스트

```bash
go build -o bin/hopmigrate ./cmd/hopmigrate

bin/hopmigrate validate -plan examples/plan.gke-to-eks.json   # Plan 확인
bin/hopmigrate run      -plan examples/plan.gke-to-eks.json   # 전체 실행
bin/hopmigrate run      -plan ... -run <RUN_ID>               # 이어서 실행
bin/hopmigrate step     -plan ... -run <RUN_ID> -step fence   # 한 단계씩 리허설
```

- `run`과 `step`은 Plan에 적힌 실제 클러스터를 바꾸므로, 운영 대상에는 승인된 Plan으로만 실행합니다. 필요한 환경 변수 이름은 `SRC_PGPASSWORD`, `DST_PGPASSWORD`, `GATEWAY_TOKEN`입니다(`docs/details.md`). rclone 리모트는 `RCLONE_CONFIG_*` 환경 변수로 설정합니다. 값은 적지 않습니다.
- 원본 정리: 이전이 성공한 뒤 `hopmigrate decommission`을 실행합니다(ADR 0010).
- 상태 확인과 출력은 `status`, `show`, `verify` 하위 명령이 `docs/details.md`에 있습니다. 실행 상태는 `.hopcharlie/runs/<RUN_ID>.json`에 저장됩니다.
- 검증 스크립트(Docker 필요):
  - `test/integration/run.sh`: PostgreSQL primary와 standby를 띄워 `pg`, `verify` 통합 테스트
  - `test/rehearsal/run.sh`: kind에 CNPG 클러스터 두 개를 띄워 왕복 이전과 원복 리허설(kind, kubectl 필요)
- CI: 모든 PR은 `go`(race·vet·build·PostgreSQL 통합)와 `cnpg-write-fence`(실제 kind CNPG fence·admission) 검사를 통과해야 합니다.

## 연결된 파트와 문서

- [README](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/README.md)
- [상세 문서 docs/details.md](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/details.md)
- [ADR 0001: CNPG distributed topology](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0001-cnpg-distributed-topology.md)
- [ADR 0003: 롤백 경계](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0003-rollback-boundary.md)
- [ADR 0004: fence gate와 verify](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0004-fence-gate-and-verify.md)
- [ADR 0005: 게이트웨이 계약](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0005-gateway-contract.md)
- [ADR 0010: 이전 엔진 안의 원본 정리](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0010-decommission-in-engine.md)
- [ADR 0015: 롤백 시 승인된 쓰기 보존 범위](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0015-rollback-acknowledged-writes.md)
- [ADR 0023: main 머지 전 기존 CI 필수](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/adr/0023-require-existing-ci-before-main-merge.md)
- [온프렘을 이전 대상으로 붙이는 순서](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/onprem.md)
- [운영 DB 부하 예산](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/docs/db-load-budget.md)
- 다른 레포: [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md), [ai-sandbox](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/README.md), [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/README.md), [platform ADR 0041: 쓰기 증거 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0041-committed-write-receipts.md)

## 알려진 한계와 미완

- 대상 승격 요청 이후 실패하면 자동으로 되돌리지 않고 `halted`로 멈춰 사람이 판단합니다. split-brain을 막기 위한 의도된 설계입니다(README "실패하면").
- 교차 환경 복제는 비동기라, primary 환경이나 저장소가 사라지는 경우의 RPO=0은 지원하지 않습니다. 게이트웨이 밖 writer는 중지·차단하고 진행 중 트랜잭션 종료를 확인해야 합니다(README "롤백의 쓰기 보존 범위", ADR 0015).
- PK는 행 중복을 막지만 외부 부작용의 exactly-once까지 증명하지 않습니다. 안전 단계 실행 중 재시작은 자동 재실행하지 않고 halt하며, 승격 중 중단만 `resume`으로 재개할 수 있습니다(README).
- 게이트웨이를 거치지 않는 쓰기(클러스터 안 배치 작업 등)는 fence가 막지 못합니다(`docs/details.md` "알려진 한계").
- 검증은 PK 집합을 메모리에 올려, 큰 테이블에는 구간 해시 방식이 필요합니다. PK 없는 테이블은 누락 검사에서 빠지고 fence gate 해시 비교에만 포함됩니다(같은 절).
- 정리 단계는 관찰 시간 밖에서만 도는 의존성을 잡지 못하고, NetworkPolicy를 적용하는 네트워크 플러그인이 필요합니다(`docs/details.md` "원본 정리").
- 이미 열린 WebSocket 세션은 쓰기 정지에 붙잡히지 않습니다(`docs/details.md`, ADR 0020 언급).
- main 보호 설정 파일을 바꾼다고 GitHub 설정이 자동 변경되지는 않습니다(README "CI와 main 머지 조건").

## 더 보려면

[migration-engine README 원문](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine#readme)
