# gitops: 플랫폼이 앱의 배포 상태를 기록하는 저장소

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `a906e96`)

## 이 파트가 하는 일

gitops는 HopCharlie 플랫폼이 앱의 배포 상태를 파일로 기록해 두는 저장소입니다. Argo CD가 이 저장소를 읽어 워크로드 클러스터에 앱을 배포합니다. 파일은 플랫폼이 만들고 지우며, 사람이 직접 고치지 않습니다. 배포를 바꾸려면 콘솔이나 `charlie` CLI를 씁니다. 커밋 하나가 배포 하나이므로, 이전 커밋의 이미지로 다시 커밋하면 빌드 없이 그 버전으로 돌아갑니다.

## 전체 흐름에서의 위치

이전 파이프라인(계획 → 리허설 → 승인 → 이전 → 전환 → 검증)의 바깥에서, 앱 배포 선언을 담는 "배포 기록" 역할입니다. 이전 엔진(migration-engine)이 데이터를 옮기는 동안 이 저장소가 직접 관여한다는 설명은 README에 없습니다(확인 필요).

앱 배포 흐름은 아래와 같습니다.

1. 앱 레포에 push하면 플랫폼이 이미지를 빌드합니다.
2. 플랫폼이 digest로 고정한 이미지와 앱 설정을 `apps/<instance>/<env>.yaml`에 커밋합니다.
3. infra 저장소의 ApplicationSet `hopcharlie-apps`가 이 파일로 Application을 만들고, infra의 `hopcharlie-app` 차트로 배포합니다.
4. 앱을 지우면 플랫폼이 파일을 지우고, Argo CD가 Application과 그 리소스를 지웁니다.

앞뒤로 연결되는 레포는 파일을 쓰는 platform, 파일을 읽어 배포하는 infra(ApplicationSet과 차트)입니다.

## 담당

조직 프로필 README의 RnR 표에는 gitops 저장소를 직접 맡은 사람이 적혀 있지 않습니다. 관련 역할만 옮깁니다. 담당자는 확인 필요입니다.

| 이름 | 표에 적힌 관련 역할 |
| --- | --- |
| 조서현 | Main: 앱 배포·Compose 온보딩·CLI, 플랫폼·콘솔 이미지 게시 자동화 |
| 김동윤 | Main: 앱 이미지 빌드·게시와 CI 러너 운영 |

## 구조

| 경로 | 역할 |
| --- | --- |
| `README.md` | 저장소 설명, 구조와 배포 흐름 |
| `apps/<instance>/<env>.yaml` | 앱 배포 선언 하나. Argo CD Application `<instance>-<env>` 하나가 됨 |
| `apps/.gitkeep` | 빈 `apps/` 디렉터리 유지용 파일 |

- `<instance>`: 앱의 배포 단위이며 배포 네임스페이스 이름과 같습니다. 기본 브랜치는 앱 이름을 씁니다(README의 예: `moimroom`).
- `<env>`: 배포할 환경(클러스터) 이름입니다. 예: `eks`, `gke`, `onprem`.
- 파일 상단에는 "플랫폼이 생성했으니 편집하지 말라"는 주석이 있습니다(파일 확인). 파일에는 이미지 digest, 서비스 포트·헬스체크 경로, DB·스토리지 리소스 선언, 암호화된 앱 설정이 들어갑니다. 값의 상세 형식은 README가 아니라 infra의 `hopcharlie-app` 차트(`values.yaml`, `values.schema.json`)가 정의합니다.

## 실행과 테스트

실행하거나 테스트할 명령은 없습니다. README에도 빌드·테스트 절차가 없습니다. 이 저장소의 파일은 직접 수정하지 말고 콘솔 또는 `charlie` CLI로 배포를 바꿉니다.

되돌리기는 이전 커밋의 이미지로 다시 커밋하는 방식(플랫폼이 수행)이라고 README에 적혀 있습니다. 이 과정에서 필요한 환경 변수나 자격 증명은 README에 없습니다(확인 필요).

## 연결된 파트와 문서

- [gitops README](https://github.com/2026-Softbank-Hackathon-Charlie/gitops/blob/main/README.md)
- [platform ADR 0011: GitHub 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0011-github-app-deploy.md)
- [platform README](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md)
- [infra ADR 0023: ApplicationSet으로 GitHub 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0023-deploy-github-apps-with-applicationset.md)
- [infra ADR 0036: 앱 라우팅과 Argo 앱](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/adr/0036-app-routing-and-argo-apps.md)
- [infra 앱 라우팅 런북](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/docs/runbooks/app-routing.md)
- [infra 앱 차트 values 스키마](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/helm/charts/hopcharlie-app/values.schema.json)
- [infra README](https://github.com/2026-Softbank-Hackathon-Charlie/infra/blob/main/README.md): "GitHub에서 배포하는 앱" 절

## 알려진 한계와 미완

README에 한계나 미완으로 적힌 항목은 없습니다. 아래는 확인 필요 항목입니다.

- 파일에 들어가는 필드의 설명이 README에 없습니다. infra의 앱 차트 문서를 따라야 합니다.
- infra README는 GitOps를 "검토한 이전 commit SHA로 고정해 되돌린다"고 적고(infra README "영향과 복구"), 이 저장소 README는 "이전 커밋의 이미지로 다시 커밋"한다고 적습니다. 두 방식의 관계는 확인 필요입니다.

## 더 보려면

[gitops README 원문](https://github.com/2026-Softbank-Hackathon-Charlie/gitops#readme)
