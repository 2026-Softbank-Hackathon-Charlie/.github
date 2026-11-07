# HopCharlie

**Beyond Lock-in · 가동 중인 앱의 환경 이전을 계획하고, 검증하고, 승인한 뒤 실행하는 멀티클라우드 관리 플랫폼**

사용자는 하나의 콘솔에서 AWS·GCP·온프레미스 환경을 다룹니다. 플랫폼은 앱 배포, DB·파일 동기화, 트래픽 전환, 보안 정책과 관측을 연결하고, AI는 격리된 샌드박스에서 계획을 리허설합니다.

## 전체 아키텍처

```mermaid
flowchart TB
    OP["운영자"] --> UI["관리 콘솔 · CLI"]
    USER["앱 사용자"] --> GW["트래픽 게이트웨이<br/>읽기 가중치 · 단일 쓰기 대상<br/>hold / drain / route / release"]
    REPO["앱 저장소<br/>moimroom-mig · platform.yaml"]
    GITOPS["GitOps 저장소 · GitHub"]

    subgraph CONTROL["관리 영역 · AWS EKS"]
        API["Platform API<br/>인증 · Plan · 승인 · 실행 · SSE"]
        STORE[("PostgreSQL<br/>Plan · 실행 상태 · 이벤트 · 감사")]
        ENGINE["Migration Engine 라이브러리<br/>단계 실행 · 복구 판단 · 검증"]
        POLICY["정책·환경 검사<br/>레지던시 · 동등성 · 버전·용량"]
        AI["AI Runner Job<br/>계획 · 격리 리허설 · 진단"]
        DEPLOY["배포 실행기<br/>빌드 · 이미지 digest · GitOps 갱신"]
        ARGO["Argo CD"]
        API --> STORE
        API --> ENGINE
        API --> POLICY
        API --> AI
        API --> DEPLOY
    end

    UI --> API
    REPO --> DEPLOY
    DEPLOY --> REG["ECR · Artifact Registry"]
    DEPLOY --> GITOPS
    GITOPS --> ARGO
    ENGINE --> GW

    subgraph AWS["AWS · EKS"]
        APP_A["앱"] --> DB_A[("CNPG PostgreSQL")]
        APP_A --> OBJ_A["S3"]
    end
    subgraph GCP["GCP · GKE"]
        APP_B["앱"] --> DB_B[("CNPG PostgreSQL")]
        APP_B --> OBJ_B["Cloud Storage"]
    end
    ONPREM["온프레미스 대상<br/>Argo CD 연결 확인<br/>서비스·스토리지 동등성 준비 중"]

    ARGO --> AWS
    ARGO --> GCP
    ARGO -. "등록된 대상" .-> ONPREM
    REG -. "고정 digest" .-> APP_A
    REG -. "고정 digest" .-> APP_B
    GW --> APP_A
    GW --> APP_B
    GW -. "준비 후 연결" .-> ONPREM
    ENGINE --> DB_A
    ENGINE --> DB_B
    DB_A -. "WAL 물리 복제 · 이전 방향에 따라 전환" .-> DB_B
    OBJ_A -. "rclone 동기화·비교" .-> OBJ_B

    subgraph OBS["관측 영역"]
        COLLECT["OTel SDK · Alloy<br/>인증된 수집 경로"] --> CLICK["ClickStack<br/>OTel Collector · ClickHouse · HyperDX"]
        PROM["Prometheus · Alertmanager"] -. "조회 연계 준비" .-> CLICK
    end
    API -. "구조화 로그" .-> COLLECT
    AI -. "작업 로그" .-> COLLECT
    APP_A -. "앱 계측" .-> COLLECT
    APP_B -. "앱 계측" .-> COLLECT
    GW -. "요청·보류 메트릭" .-> PROM

    classDef preparing stroke-dasharray: 5 5;
    class ONPREM preparing;
```

게이트웨이는 관리 EKS에서 운영하며, 그림에서는 사용자 트래픽 경로로 분리했습니다. 한 환경만 쓰기 대상으로 선택하고, DB 복제는 현재 primary에서 replica로 흐르다가 이전 시 방향을 바꿉니다. 온프레미스의 점선은 연결된 대상과 확장 경로이며, CNPG·MinIO 등 대체 서비스의 설치·이전 완료를 뜻하지 않습니다.

## 가동 중 이전과 복구 경계

```mermaid
flowchart LR
    P["승인된 Plan"] --> PRE["precheck · prepare<br/>환경·버전·대상 준비"]
    PRE --> SYNC["presync · delta<br/>DB 따라잡기 · 파일 복사<br/>원본 서비스 계속 처리"]
    SYNC --> HOLD["fence<br/>쓰기 보류 · 기존 쓰기 drain<br/>최종 LSN · 마지막 파일 동기화"]
    HOLD --> GATE["switchover<br/>fence LSN replay · 데이터 비교"]
    GATE --> DEMOTE["원본 강등<br/>demotion token 확인"]
    DEMOTE --> PROMOTE["대상 승격 요청<br/>쓰기 가능·안정 상태 확인"]
    PROMOTE --> APP["필요 시 대상 앱 시작·준비 검사"]
    APP --> CUT["cutover<br/>route → release"]
    CUT --> CHECK["verify · finalize<br/>행·기록된 쓰기·파일 검증"]
    CHECK --> REPORT["실행별 리포트<br/>지연 · 누락 · 검증 범위"]

    HOLD -. "승격 요청 전 실패" .-> REVERT["원본 복구 · 쓰기 해제"]
    PROMOTE -. "승격 결과 불명확·전환 실패" .-> HALT["halted<br/>보류 유지 · 운영자 확인"]
```

쓰기를 해제하기 전에 복제 위치와 DB 상태를 확인합니다. 승격 결과가 불명확하면 양쪽을 동시에 primary로 만들지 않도록 멈춥니다. cold 대상 앱이 시작하지 못한 경우에는 **트래픽 전환 전·모든 쓰기 보류 상태**라는 제한된 조건에서만 별도 복구 절차를 허용합니다. 정상 복귀는 살아 있는 새 primary의 WAL을 전달하는 역방향 이전으로 수행합니다.

비동기 복제의 환경 유실에 대한 RPO=0을 보장하지 않습니다. 누락 결과는 검사한 데이터와 수집된 쓰기 기록의 범위이며, 미측정·오류는 성공한 0건으로 바꾸지 않습니다. [엔진 계약과 검증 기록](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/tree/main/docs)

## AI는 리허설하고, 사람은 승인한다

```mermaid
flowchart LR
    REQUEST["콘솔에서 앱·이전 방향 선택"] --> JOB["AI Runner Job"]
    JOB --> TOOLS["허용된 도구로 환경 조사<br/>원본 primary·버전·자원 확인"]
    TOOLS --> PLAN["Plan · 대상 매니페스트 제안"]
    PLAN --> VALID["출력·정책·실행 대상 검사"]
    VALID --> SB["격리 namespace<br/>제한된 권한 · 네트워크·자원 경계"]
    SB --> RUN["임시 CNPG 원본 + 합성 데이터<br/>실제 엔진 리허설"]
    RUN --> RESULT{"리허설 결과"}
    RESULT -->|"실패"| DIAG["근거 수집 · AI 진단·수정안"]
    DIAG -->|"maxRuns 상한 내 재리허설"| PLAN
    RESULT -->|"통과"| REVIEW["Plan revision · 첨부 · 증거<br/>사람 검토·승인"]
    REVIEW --> EXEC["승인된 Plan으로 실행 요청"]
    JOB -. "도구·판단·리허설 기록" .-> EVENTS["플랫폼 세션·이벤트 → 콘솔"]
```

실패 시 유효한 수정안이 있으면 `maxRuns` 상한 안에서 새 샌드박스로 다시 리허설합니다. AI가 운영 이전을 스스로 승인하지 않습니다. 리허설은 실제 CNPG와 엔진을 사용하지만, 운영 앱의 전체 데이터·트래픽·세션 정합성을 그대로 재현했다는 의미는 아닙니다. [AI·샌드박스 구현](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox)

## 평가항목을 충족하는 구조와 시연

| 평가항목 | 배점 | 아키텍처에 넣은 솔루션 | 보여줄 근거 |
| --- | ---: | --- | --- |
| 완성도·데모 | 30 | DB·파일·트래픽을 묶은 이전 상태 머신, 쓰기 보류·복구 경계, 실행 리포트 | **(a)** 실제 앱에 트래픽을 보내며 이전하고 요청 지연·실패·검증 결과 확인 |
| 클라우드 활용 수준 | 30 | EKS·GKE의 공통 CNPG 계약, 저장소·게이트웨이 어댑터, 정책·환경 동등성 검사 | **(b)** 환경별 보안·거버넌스 결과와 근거를 같은 모델로 조회, 온프레미스 준비 상태 구분 |
| 팀 개발 | 20 | API·엔진·콘솔·인프라의 책임 분리, OpenAPI 계약, ADR·PR·CI | 저장소 간 계약과 설계 선택, 실제 테스트·리허설 기록 추적 |
| 재미·독창성 | 10 | 배포 이후의 **환경 이동**을 운영 대상으로 삼고, 자원 존재와 실제 사용 증거를 분리 | 같은 앱의 이전 방향·목적지를 바꾸고 `pass / fail / unknown`의 이유 확인 |
| AI 활용 | 10 | 도구를 쓰는 AI, 격리된 실제 엔진 리허설, 실패 진단·수정, 사람 승인 | **(c)** 도구 호출 → 리허설 실패·수정 → 증거 검토 → 승인 흐름 확인 |

## 현재 검증 범위

| 범위 | 현재 상태와 남은 경계 |
| --- | --- |
| 클라우드 이전 | EKS↔GKE 및 격리 CNPG 이전·복구 실행 기록이 있습니다. 실제 환경·데이터·부하별 결과로 판단합니다. [실행 기록](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/tree/main/docs/test-records) |
| 온프레미스 | Argo CD 대상 연결을 확인했습니다. CNPG·S3 호환 저장소·비밀 관리·백업·관측의 동등성과 실제 이전은 추가 준비·검증 범위입니다. |
| 보안·동등성 | 레지던시는 설정된 정책으로 시작을 제어합니다. 동등성은 현재 감사 모드이며, 지정된 AWS·GCP 권한 요구와 지원하는 앱의 자격증명 사용을 부분 검사합니다. 미구현·설정 없음·검사 불가는 `unknown`입니다. [검사 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/tree/main/internal/preflight) |
| 쓰기 증거 | `moimroom-mig`는 예약 INSERT와 같은 트랜잭션의 outbox를 지원합니다. 운영 수집에는 별도 설정·전용 토큰이 필요하며, 수집된 커밋 기록은 클라이언트 응답 수신이나 전체 CRUD 수집을 증명하지 않습니다. [수집 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0041-committed-write-receipts.md) |
| 관측·최적화 | 로그·트레이스는 HyperDX/ClickStack, 메트릭·알림은 Prometheus/Alertmanager로 구분합니다. HyperDX에서 Prometheus를 조회하는 인증 경로의 검증은 후속 범위입니다. SDK·수집 설정이 있는 경로를 관측하며, 쓰기 보류 단축은 별도 실측·개선 중입니다. [관측 설계](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/clickstack-architecture.md) |

비용 기반 자동 재배치, 세션 정합성 보장, AI의 GPU 실사용 자동 판별은 현재 완료 기능으로 표시하지 않습니다.

## 저장소와 책임

| 저장소 | 맡는 경계 |
| --- | --- |
| [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform) | API·승인·지속 상태·감사, 배포 실행기와 트래픽 게이트웨이 |
| [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine) | CNPG·PostgreSQL·rclone 어댑터, 이전·복구 상태 머신과 검증 |
| [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend) | 운영 콘솔, 실행·보안·AI 활동·검증 결과 표시 |
| [ai-sandbox](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox) | AI 도구·Plan 검사·격리 샌드박스·리허설·진단 |
| [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra) | Terraform·Helm, 클러스터 공통 구성·권한·관측 기반 |
| [gitops](https://github.com/2026-Softbank-Hackathon-Charlie/gitops) | Argo CD가 적용하는 앱별 배포 선언 |
| [cli](https://github.com/2026-Softbank-Hackathon-Charlie/cli) | 플랫폼 API를 사용하는 운영 CLI |
| [moimroom-mig](https://github.com/2026-Softbank-Hackathon-Charlie/moimroom-mig) | 실제 이전 대상 예약 앱, DB·파일·멱등 예약·쓰기 기록 |

설계 선택과 한계는 각 저장소의 ADR·실행 기록에 남깁니다. 평가 기준의 원문은 [팀 본선 제약사항](https://github.com/2026-Softbank-Hackathon-Charlie/.github/blob/main/agent-rules/.claude/skills/finals-constraints/SKILL.md)을 따릅니다.
