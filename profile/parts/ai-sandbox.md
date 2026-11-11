# ai-sandbox: 이전 계획을 AI가 쓰고, 승인 전에 격리 샌드박스에서 먼저 돌려 보는 레포

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `aa8d4f4`)

## 이 파트가 하는 일

HopCharlie의 AI(이전 계획 작성, 리허설 실패 진단, 수정안, 들여온 오픈소스 레포의 배포 준비)와 샌드박스 격리를 맡는 Go 레포입니다. AI는 읽기 전용 도구로 원본·대상 환경을 조사해 플랫폼의 `EnginePlan` 형식 JSON과 대상 매니페스트, 근거를 만듭니다. 플랫폼이 스키마와 고정 규칙으로 이 출력을 검사하고, 통과한 것만 다음 단계로 넘어갑니다.

승인 전에는 대상 클러스터에 만든 격리 샌드박스(임시 PostgreSQL CNPG와 합성 데이터)에서 실제 migration-engine으로 Plan을 끝까지 돌려 봅니다. 대상 환경의 차이(StorageClass, 노드 자원, 네트워크)를 운영자 승인 전에 드러내려는 것입니다. 실패하면 AI가 샌드박스 로그만 보고 원인과 수정안을 내고, 사본에 적용해 새 샌드박스에서 다시 돌립니다.

AI에게는 운영 클러스터를 바꾸는 권한이 없습니다. 운영 변경은 사람이 승인한 Plan을 엔진이 실행할 때만 일어납니다(README 1장).

## 전체 흐름에서의 위치

계획 → 리허설 → 승인 → 이전 → 전환 → 검증 중 **계획과 리허설**(승인 직전까지)이 이 레포의 몫입니다. README 2장 기준으로 `hopai run`이 Plan 초안 요청, AI 계획, 제출, 리허설, 진단·수정, 리포트 첨부를 한 번에 돌리고 승인 전에서 멈춥니다. 승인과 실행은 하지 않습니다.

- 앞: platform이 Plan 초안(`POST /v1/plans/draft`)을 주고, 제출된 Plan을 `pending_approval`로 저장합니다.
- 뒤: 운영자가 콘솔(frontend)에서 첨부와 AI 활동 기록을 보고 승인하면, platform과 migration-engine이 승인된 Plan을 실행합니다(승인 뒤의 9단계는 엔진의 고정 순서).
- 승인 게이트: 리허설 리포트가 리허설한 후보의 SHA-256을 남기고, 플랫폼이 승인 때 이를 확인하는 쪽은 platform 이슈 #24(AI 제출 Plan 승인에 입력과 일치하는 리허설 증거 요구)로 추적했고, 이 이슈는 2026-11-05에 닫혔습니다. README 11장은 11/5 이전 시점이라 아직 "남음"으로 적혀 있고, 구현이 들어간 PR은 확인 필요입니다.
- 이 밖에 `hopai prepare`, `hopai onboard`(앱 레포에 계약 파일 추가), `hopai recommend`(개요 화면 이전 추천)도 같은 레포에 있습니다.

## 담당

조직 프로필 README의 RnR 표에서 이태훈이 "AI Sandbox · Console"입니다. 박준석의 "Migration · DB · Map · AI Recommendations" 중 AI Recommendations는 `hopai recommend`와 관련될 수 있으나, 표에서 확정할 수 없어 확인 필요입니다. 이 레포 README 설계 문서의 작성자 표기는 "태훈"입니다.

## 구조

| 경로 | 역할 |
|---|---|
| `cmd/hopai` | 전체 흐름 CLI: `run`, `prepare`, `onboard`, `recommend`, `telemetry-check` |
| `cmd/sandboxctl` | 샌드박스를 손으로 만들고 검사하고 지우는 도구 |
| `llm` | 도구를 쓰는 AI 에이전트(계획, 진단, 배포 준비), 읽기 전용 조사 도구, JSON 스키마 |
| `plan` | AI 출력과 진단 수정안 검사, 사본 적용, 제출 본문과 후보 digest |
| `mask` | AI·플랫폼에 보내기 전 비밀값 가리기 |
| `agentlog` | 플랫폼 에이전트 세션 기록(`/v1/agent/sessions`) |
| `sandbox` | 샌드박스 생성·삭제, 1시간 토큰, 만료 정리, CNPG 모드, 격리 검사 |
| `rehearsal` | 샌드박스에서 실제 엔진으로 리허설, 실패 시 진단·재실행, 리포트 |
| `telemetry` | Langfuse 기록(ADR 0003) |
| `docs/HISTORY.md` | 머지된 PR별 변경 이력 |
| `docs/adr/` | 결정 기록 4건(0001~0004) |
| `.github/workflows/ci.yml` | PR과 main에서 `go vet`, gofmt, `go test ./...` |

## 실행과 테스트

비밀값은 환경 변수로만 받습니다(README 12장). 필요한 변수: `ANTHROPIC_API_KEY`, `PLATFORM_URL`, `PLATFORM_TOKEN`. 선택: `LANGFUSE_BASE_URL`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`. 클러스터 접속은 kubeconfig 파일(`-env`, `-sandbox-kubeconfig`)로 줍니다.

```bash
go run ./cmd/hopai run -app moimroom -from gke -to eks \
  -env gke=gke-readonly.kubeconfig -env eks=eks-readonly.kubeconfig \
  -sandbox-kubeconfig eks-sandbox-admin.kubeconfig \
  -target db=moimroom-db-eks-rw.moimroom.svc.cluster.local:5432

# 비공개 모듈(platform, migration-engine)을 받으려면
go env -w GOPRIVATE=github.com/2026-Softbank-Hackathon-Charlie

# 들여온 오픈소스 레포의 배포 준비
go run ./cmd/hopai prepare -repo 2026-Softbank-Hackathon-Charlie/<레포> [-deployment <배포 ID>]

# 테스트
go test ./...

# 실제 클러스터(CNPG 설치) 리허설 테스트. AI 없이 예시 계획으로 엔진만 돌림
REHEARSAL_KUBECONFIG=<kubeconfig> REHEARSAL_SC=standard go test -tags cluster -run Cluster -v -timeout 40m ./rehearsal/
```

데모 실패 시나리오는 `hopai run`에 `-demo-copy-source-storage`를 붙입니다(README 9장). `hopai run`은 리허설을 통과하지 못하면 종료 코드 1입니다.

## 연결된 파트와 문서

- [README 원문](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/README.md): 1~3장 원칙·흐름·Plan JSON, 6장 샌드박스, 7장 리허설, 12장 구현 현황
- [docs/HISTORY.md](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/docs/HISTORY.md): 버전별 변경 이력
- [ADR 0001 서버 토폴로지 권한 보존](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/docs/adr/0001-preserve-server-topology-permission.md)
- [ADR 0003 실제 에이전트 호출을 Langfuse에 기록](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/docs/adr/0003-record-real-agent-calls-in-langfuse.md)
- [ADR 0004 읽기 전용 이전 추천의 범위](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/docs/adr/0004-bound-read-only-migration-recommendations.md)
- [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform): Plan 제출·승인 API, OpenAPI의 `EnginePlan` 계약
- [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine): 리허설과 실제 이전에서 쓰는 엔진
- [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend): 승인 화면과 AI 활동 표시
- [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra): 클러스터와 NetworkPolicy 기반

## 알려진 한계와 미완

모두 README 기준이며 작성 시점(대부분 11/5 전후)의 서술입니다.

- 새 Pod는 NetworkPolicy가 붙기 전 몇 초간 통신이 열립니다. 격리 검사 Pod가 15초 기다린 뒤 연결하도록 우회했습니다(12장 알려진 한계).
- 수정 뒤의 전체 `hopai run`(리허설 통과, 실제 AI 진단, 수정안 첨부)과 `hopai prepare`는 실제 Claude·플랫폼·GitHub로 돌려 보지 못했다고 적혀 있습니다(12장).
- Kubernetes 이벤트나 Job 실패로 드러나지 않는 실패는 `catchupTimeout`까지 기다립니다(12장).
- 파일(rclone), 게이트웨이와 카나리, 부하(k6), 클러스터 사이 복제는 리허설하지 않습니다(7장, 11장).
- 만료 샌드박스를 지우는 CronJob 매니페스트는 아직 없습니다(6장).
- 쓰기 보류 시간은 측정마다 편차가 컸고, 상한과 타임아웃·재요청 값은 제안값입니다. 실제 EKS 재측정이 남았습니다(10장, 11장).
- 샌드박스 관리 자격 증명을 어느 계정으로 할지, strict 모드 적용 시점 등은 팀 결정 대기였습니다(11장).
- `hopai prepare`는 push 전에 사람이 diff를 보는 단계가 없고, 이미 등록된 앱이면 push가 바로 재배포합니다(12장).

## 더 보려면

[ai-sandbox README 전문](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox/blob/main/README.md)
