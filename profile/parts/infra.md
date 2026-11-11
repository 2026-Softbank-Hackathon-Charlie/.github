# infra: HopCharlie를 운영하는 Terraform·Helm·Argo CD 인프라 저장소

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `32cd091`)

## 이 파트가 하는 일

SoftBank Hackathon 2026 본선에서 HopCharlie를 운영하기 위해 쓴 인프라 저장소입니다. Terraform으로 AWS·GCP 기반(state, VPC, 권한, EKS/GKE)을 만들고, Helm과 Argo CD로 관리 플랫폼과 클러스터 공통 설정(애드온, 관측, DB 등)을 배포합니다. 이전 대상 환경 네 곳(AWS 관리 EKS, AWS 워크로드 EKS, GCP GKE, 온프레미스 k3s)의 기반이 여기서 정의됩니다. 실제 계정 값, tfvars, state, plan, kubeconfig, 비밀값은 Git에 두지 않습니다.

README에 따르면 시연에 쓴 AWS·GCP 환경은 2026-11-09에 정리했고, 지금 README는 저장소에 남은 구성과 재현 절차입니다.

## 전체 흐름에서의 위치

이전 흐름(계획 → 리허설 → 승인 → 이전 → 전환 → 검증)이 돌아가는 "바닥"을 제공합니다.

- 관리 EKS: 플랫폼 API, 트래픽 게이트웨이, 콘솔, 상태 DB, Argo CD가 올라가는 곳입니다(platform, frontend 레포의 이미지를 배포).
- 워크로드 환경(AWS EKS, GKE, 온프레미스 k3s): 앱과 CNPG, 객체 스토리지가 있는 이전의 출발지와 목적지입니다. migration-engine이 이 환경들의 CNPG를 대상으로 동작하며, 필요한 CNPG 설정은 migration-engine 문서의 "다른 영역과의 약속"에 있습니다.
- 앱 배포: platform이 gitops 레포에 쓰는 파일을 이 레포의 ApplicationSet과 `hopcharlie-app` 차트가 읽어 워크로드 클러스터에 배포합니다.
- AI 실행기(hopai) Job 템플릿, 관측(ClickStack, Prometheus 계열)도 이 레포의 차트와 values로 배포합니다.

## 담당

조직 프로필 README의 RnR 표 기준입니다. AWS EKS 구성의 Main 담당은 표에서 확정할 수 없습니다(확인 필요).

| 이름 | 맡은 역할 |
| --- | --- |
| 김동윤 | Main: GKE·온프레 인프라, 보안 설계·구축, CI 러너 운영. Assist: AWS EKS 구성 |
| 강종욱 | Main: AWS 워크로드 DB 기반(CNPG), ClickStack·AWS/GKE 메트릭·Langfuse 관측. Assist: Terraform 배포 자동화 |
| 조서현 | Assist: 인프라 구성. Main: 플랫폼 이미지 게시 자동화 |

## 구조

| 경로 | 역할 |
| --- | --- |
| `terraform/` | AWS·GCP 기반과 state, 권한, 클러스터, 서비스별 통합. 하위에 `bootstrap`, `account`, `environments/{aws,gcp}`, `integrations`, `modules/eks`, `platform/aws` |
| `helm/` | bootstrap(Argo CD), 차트(`hopcharlie-platform`, `hopcharlie-app`, `hopcharlie-apps` 등), GitOps 기본 values, 이미지 digest 고정 파일 |
| `clusters/` | 환경별 values와 매니페스트(`aws-dev-management`, `aws-dev-workload`, `gcp-dev-workload`, `onprem-dev-workload`) |
| `config/` | 공유 입력 `project.json`, 계정별 입력 예시 `account.local.json.example` |
| `scripts/` | 입력 검증, Terraform 실행 wrapper, 접근·관측·검사 보조 스크립트 |
| `docs/runbooks/` | 실행 절차(앱 라우팅, CNPG 접속, DB 이전 등) |
| `docs/operations/`, `docs/plans/` | 관측 준비 문서, 승인 경계가 있는 작업 계획 |
| `docs/adr/` | 설계 결정 기록 |
| `.github/workflows/terraform-check.yml` | PR·main 검증과 AWS 입력 변경 시 자동 apply, 수동 plan/apply |
| `Makefile` | 검증, 로컬 Terraform, SSM 터널, Argo CD 접속 진입점 |
| `AGENTS.md` | 주요 의사결정을 ADR로 기록하는 Agent 지침 |

## 실행과 테스트

로컬 검증은 읽기 전용이며 실제 클라우드를 바꾸지 않습니다. 필요한 도구는 Terraform(`.terraform-version`), Helm, Python 3와 PyYAML, Bash, make입니다.

```bash
python3 -m venv /tmp/charlie-check-venv
/tmp/charlie-check-venv/bin/python -m pip install PyYAML==6.0.3
make check PYTHON=/tmp/charlie-check-venv/bin/python
make inventory-check   # README의 추적 파일 링크와 Terraform 상대 참조
make config-check      # 프로젝트 설정과 네트워크 입력
```

이 검증은 실제 IAM 충분성, 네트워크 연결, 이미지 pull, DNS/TLS, 배포 성공을 보장하지 않습니다(README "검증").

클러스터에 접속하는 명령은 로그인된 AWS 프로필(`AWS_PROFILE`, 기본값은 README 참조)과 AWS CLI v2, kubectl, session-manager-plugin이 필요합니다. 각각 별도 터미널에서 실행하며 foreground로 계속 켜 둡니다.

```bash
make aws-tunnel          # 관리 EKS API 터널
make argo-port-forward   # Argo CD, 브라우저에서 https://localhost:8080
```

아래는 **운영자가 별도 승인 후 수동 실행**하는 배포 절차입니다. 예제 값을 그대로 배포하지 않습니다. `apply`, migration, Helm/kubectl 적용은 실제 외부 변경입니다. 계정 입력은 `config/account.local.json.example`을 복사해 로컬에서만 채우고(Git에 올리지 않음) `AWS_ACCOUNT_ID` 환경 변수를 설정합니다.

```bash
make tf-init STACK=account/aws     # 이후 tf-plan, tf-apply. 별도 승인 후 수동 실행
# STACK=aws/dev/foundation, aws/dev/management, aws/dev/workload 순서로 반복
```

- CI: `main`에서 AWS 배포 입력 경로가 바뀌고 `check`가 성공한 경우에만 plan 후 변경이 있는 stack을 자동 apply합니다. 수동 실행의 기본값은 `plan` + `bootstrap`입니다. PR과 main 이외 브랜치에서는 AWS 배포를 하지 않습니다.
- GCP 배포는 `main`에서 수동으로 시작합니다.
- 권한이 필요한 감사·변경 작업은 사전 승인이 필요합니다(ADR 0002, `AGENTS.md`).

## 연결된 파트와 문서

- [infra README](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/README.md): "환경별 구성", "처음 사용할 때: 입력과 state", "영향과 복구", "GitHub Actions 파이프라인", "HopCharlie 플랫폼", "검증" 절
- [AGENTS.md](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/AGENTS.md)
- [ADR 0002: 권한 작업 사전 승인](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0002-require-approval-for-privileged-operations.md)
- [ADR 0016: 플랫폼을 관리 클러스터에서 운영](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0016-run-hopcharlie-platform-on-management-cluster.md)
- [ADR 0023: ApplicationSet으로 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0023-deploy-github-apps-with-applicationset.md)
- [ADR 0025: infra main에 이미지 고정](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0025-pin-images-on-infra-main.md)
- [ADR 0036: 앱 라우팅과 Argo 앱](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0036-app-routing-and-argo-apps.md)
- [ADR 0037: AWS 게이트웨이와 GKE 앱의 Tailscale 연결](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0037-connect-aws-gateway-to-gke-apps-with-tailscale-routers.md)
- [ADR 0072: 온프레미스를 쌍 DB 환경으로 추가](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0072-add-onprem-as-paired-database-environment.md)
- [앱 라우팅 런북](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/runbooks/app-routing.md) · [CNPG 접속](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/runbooks/cnpg-access.md) · [AWS-GCP DB 이전](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/runbooks/aws-gcp-db-migration.md)
- [GCP 구축 절차](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/gcp-deployment.md) · [ClickStack 구성](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/clickstack-architecture.md) · [온프레미스 관측](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/operations/onprem-observability.md)
- [플랫폼 Terraform](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/terraform/platform/aws/README.md)
- 다른 레포: [gitops](https://github.com/2026-Softbank-Hackathon-Charlie/gitops/blob/main/README.md), [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md), [migration-engine](https://github.com/2026-Softbank-Hackathon-Charlie/migration-engine/blob/main/README.md)

## 알려진 한계와 미완

- 시연 환경은 2026-11-09에 정리되어 접속 주소나 자원 수는 작성 시점의 기록 또는 코드 기본값입니다(README 서두, "상세 절차와 인벤토리").
- Argo와 지정 운영자의 EKS 권한은 cluster-admin이고, Prometheus/Grafana/Alertmanager/Loki와 Argo는 단일 replica·일부 AZ 종속 EBS라 전체 서비스 HA를 보장하지 않습니다(README "영향과 복구").
- 외부 ALB는 기본 비활성이며 DNS/ACM 자동 생성은 포함하지 않습니다. 애플리케이션 코드, 서비스 ECR, RDS는 별도 구성입니다(README "구성 범위").
- 기본 GitOps values의 대상은 비어 있고 Headlamp, 관측, 온프레미스 자동 등록은 별도 활성화가 필요합니다(README "배포와 운영").
- AWS CNPG PoC: 2026-11-04 확인 시 DB는 Ready였으나 SQL 접속, migration/grant, 실제 복원, RPO/RTO는 미검증이었습니다(README "AWS CNPG PoC").
- 코드 merge만으로 실제 IAM trust는 바뀌지 않고, 실제 IAM 변경은 별도 승인이 필요합니다(README "처음 사용할 때"). Sealed Secrets private key의 백업·복구 절차는 운영 사용 전에 따로 준비해야 합니다(README "영향과 복구").
- 단일 상태 소유자 기반 게이트웨이 파드 복구는 제안 단계입니다(ADR 0091 언급).

## 더 보려면

[infra README 원문](https://github.com/2026-Softbank-Hackathon-Charlie/infra#readme)
