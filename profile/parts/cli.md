# cli: 플랫폼 API를 터미널에서 쓰는 운영 CLI `charlie`

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `9aad1a2`)

## 이 파트가 하는 일

cli는 HopCharlie 플랫폼 API를 터미널에서 호출하는 Go 기반 명령행 도구 `charlie`입니다. 웹 콘솔과 같은 API를 부르며, 응답 타입은 platform 레포의 `pkg/apiv1`(OpenAPI 명세에서 생성)을 그대로 씁니다. 이전 Plan의 제출·검토·승인, 이전 실행의 시작·전환 승인·중단·재개·롤백과 실시간 관찰, 게이트웨이 트래픽과 감사 로그 조회를 맡습니다. 또 GitHub 레포를 앱으로 등록하고 배포하며, 앱 시크릿 관리, 기존 compose 앱 가져오기, 보안 뷰와 AI 에이전트 활동 조회도 제공합니다. 콘솔을 열 수 없는 스크립트, CI, 봇 환경에서 같은 작업을 하게 하는 것이 필요한 이유입니다(`--json` 출력과 종료 코드 제공).

## 전체 흐름에서의 위치

- **계획**: `plan new`로 Plan 초안을 만들고 `plan submit`으로 제출합니다. `plan attach`로 대상 CNPG 매니페스트 등 첨부를 붙입니다.
- **리허설**: AI 리허설과 샌드박스는 `ai-sandbox` 레포 몫입니다. CLI에서는 `agent list/show/watch`로 AI 에이전트 세션 기록을 볼 수 있습니다.
- **승인**: `plan show`로 revision과 첨부(이름, 크기, SHA-256)를 확인하고 `plan approve`로 승인합니다. 승인과 실행은 operator 역할만 할 수 있습니다.
- **이전·전환**: `migrate start`(카나리, observe, 자동 롤백 옵션), `migrate switch`로 전환을 승인합니다.
- **검증**: `migrate watch`가 단계, 다운타임, 복제 지연, 실패 요청 수, 누락 수를 실시간으로 보여 주고, `migrate rollback`으로 되돌립니다.
- **앞뒤 레포**: 호출 대상은 `platform`(API 서버, `pkg/client`·`pkg/apiv1`·`pkg/appspec` import)입니다. 로그인 승인은 `frontend`의 콘솔 `#device` 화면에서 합니다. 앱 배포는 platform이 `gitops` 레포에 커밋하는 흐름을 탑니다.

## 담당

조직 프로필 README의 RnR 표 기준입니다.

| 이름 | 표에 적힌 역할 |
|---|---|
| 조서현 | Main: 앱 배포·Compose 온보딩·CLI (플랫폼 API·트래픽 게이트웨이, 샘플 앱 개발도 함께 맡음) |

다른 팀원이 이 레포에서 맡은 부분은 표에서 확정할 수 없어 확인 필요입니다.

## 구조

README에 디렉터리 설명은 없어 레포 파일 목록으로 정리했습니다.

| 경로 | 역할 |
|---|---|
| `cmd/charlie/main.go` | 진입점. 버전은 빌드 시 `-X main.version`으로 주입 |
| `internal/cli/auth.go` | `login`, `whoami`, `logout`(device code 로그인) |
| `internal/cli/config.go` | 저장된 로그인과 설정 디렉터리 |
| `internal/cli/envs.go`, `traffic.go`, `misc.go` | 환경, 트래픽, 그 밖의 조회 명령. 이 파일이 어느 명령을 맡는지는 확인 필요 |
| `internal/cli/plans.go`, `plantools.go` | Plan 제출·승인, `plan new`·`attach`·`written-records` |
| `internal/cli/migrate.go`, `watch.go` | 이전 실행과 실시간 관찰 |
| `internal/cli/app.go`, `appenv.go`, `branch.go` | 앱 등록, 환경 추가·제거, 브랜치 인스턴스 |
| `internal/cli/deploy.go`, `deploywatch.go` | 배포와 배포 로그·관찰 |
| `internal/cli/secret.go`, `onboard.go`, `security.go`, `agent.go` | 시크릿, compose 앱 가져오기, 보안 뷰, 에이전트 기록 |
| `internal/update/` | 자동 업데이트 |
| `.github/workflows/ci.yml` | CI |

## 실행과 테스트

platform 레포가 private이라 한 번 설정이 필요합니다.

```bash
go env -w GOPRIVATE=github.com/2026-Softbank-Hackathon-Charlie
go install github.com/2026-Softbank-Hackathon-Charlie/cli/cmd/charlie@latest
```

직접 빌드할 때는 다음 명령을 씁니다.

```bash
go build -o bin/charlie -ldflags "-X main.version=v0.1.0" ./cmd/charlie
```

기본 사용 흐름은 README의 예시를 따릅니다.

```bash
charlie login     # 코드를 보여 주고, 관리자가 콘솔에서 승인하면 토큰을 저장한다
charlie whoami
charlie plan submit -f plan.json                  # agent 또는 operator
charlie plan show pln_xxx                         # revision과 첨부 확인
charlie plan approve pln_xxx --revision 2 --comment "확인함"    # operator
charlie migrate start pln_xxx --canary 1,10,25,50,75,90,100 --watch
# 다른 터미널에서, awaiting_switch가 되면
charlie migrate switch <run id>
```

- 환경 변수(이름만): `CHARLIE_SERVER`, `CHARLIE_TOKEN`(저장된 값보다 우선, `--server` 플래그가 최우선), `CHARLIE_CONFIG_DIR`, `CHARLIE_UPDATE_INTERVAL`, `CHARLIE_NO_UPDATE`, `NO_COLOR`.
- 서버를 지정하지 않으면 팀 배포 주소를 기본으로 씁니다.
- 봇이나 원격 호스트에서는 `charlie login --no-wait` 후 승인되면 `charlie login`을 다시 실행합니다.
- 종료 코드: 0 성공, 1 오류, 2 사용법 오류, 3 지켜보던 run·배포가 성공 상태로 끝나지 않음, 130 Ctrl-C.
- 테스트: README에 테스트 명령은 없습니다. 레포에는 `internal/cli`의 `*_test.go`가 있고, CI 워크플로가 `go vet -race ./...`, gofmt 검사, `go test -race ./...`를 실행합니다.

## 연결된 파트와 문서

- 원문 README 절: [로그인](https://github.com/2026-Softbank-Hackathon-Charlie/cli/blob/main/README.md#로그인) · [명령](https://github.com/2026-Softbank-Hackathon-Charlie/cli/blob/main/README.md#명령) · [GitHub 레포로 앱 등록과 배포](https://github.com/2026-Softbank-Hackathon-Charlie/cli/blob/main/README.md#github-레포로-앱-등록과-배포) · [종료 코드](https://github.com/2026-Softbank-Hackathon-Charlie/cli/blob/main/README.md#종료-코드)
- API 계약: [platform api/openapi.yaml](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/api/openapi.yaml) · [platform README](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/README.md)
- README가 가리키는 platform ADR: [0011 GitHub 앱 배포](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0011-github-app-deploy.md) · [0015 브랜치 인스턴스](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0015-branch-instances.md) · [0019 앱 시크릿과 재배포](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0019-app-secrets-and-redeploy.md) · [0042 compose 앱 가져오기](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0042-onboard-existing-compose-apps.md) · [0043 앱에 환경 추가](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/docs/adr/0043-add-environment-to-registered-app.md)
- 다른 레포: [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform) · [frontend](https://github.com/2026-Softbank-Hackathon-Charlie/frontend) · [ai-sandbox](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox) · [gitops](https://github.com/2026-Softbank-Hackathon-Charlie/gitops)

## 알려진 한계와 미완

- 정적 토큰(`CHARLIE_TOKEN`)은 만료되지 않고 `logout`으로 끝낼 수 없습니다(README "설정 디렉터리와 환경 변수"). 서버에 닿지 못한 채 `logout`하면 파일만 지워지고 토큰은 만료까지 서버에서 유효합니다(README "로그인").
- 추가한 브랜치 인스턴스는 지금 eks 환경에서만 돌 수 있습니다. GKE의 Argo CD가 `charlie-apps` 네임스페이스만 관리하기 때문입니다(README "GitHub 레포로 앱 등록과 배포").
- `onboard` 명령은 서버가 `PLATFORM_ONBOARDING=1`일 때만 경로가 있고, 구현 중에는 `not_implemented`(501)로 답합니다(README "기존 compose 앱 가져오기").
- Git Bash(mintty)에서는 숨김 입력 프롬프트를 쓸 수 없어 `winpty`, PowerShell, `--stdin`이 필요합니다(README "시크릿").
- `app rm`, `app branch rm`, `app env remove`는 데이터를 되돌릴 수 없게 지웁니다. 그래서 `--yes`나 이름 입력 확인을 요구합니다(README 해당 명령 표).

## 더 보려면

원문 README: https://github.com/2026-Softbank-Hackathon-Charlie/cli/blob/main/README.md
