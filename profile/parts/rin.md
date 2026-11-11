# rin: 팀 Slack에서 부르는 Claude Code 에이전트 봇

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `5ac4472`)

## 이 파트가 하는 일

Rin은 팀 채널 `#final_team_charlie`와 DM에서 부르는 Claude Code 에이전트입니다. 팀원이 `@Rin 할 일`처럼 멘션하거나 DM을 보내면 작업을 큐에 넣고, Claude Agent SDK로 실행해 결과를 Slack 스레드에 올립니다. 팀원 5명만 호출할 수 있고, 실행 중인 작업은 상태 메시지의 취소 버튼으로 멈춥니다. 예약 작업(cron은 KST, 또는 한 번만 실행할 ISO 시각)도 관리합니다.

README 기준으로 Rin은 팀의 작업 도구입니다. 이 레포가 HopCharlie 플랫폼의 이전 기능(Plan, 리허설, 전환)에 쓰이는지는 README에 나오지 않습니다. 계획 문서는 Notion의 "Rin: 팀 슬랙 에이전트 봇 계획"에 있다고만 적혀 있습니다.

## 전체 흐름에서의 위치

README와 문서에서 계획 → 리허설 → 승인 → 이전 → 전환 → 검증 흐름과 연결된다는 내용은 확인되지 않습니다. 조직 프로필 README의 소스 저장소 표에도 rin은 없습니다. 따라서 플랫폼 본체와의 관계는 **확인 필요**이고, 지금까지 확인되는 것은 아래뿐입니다.

- 조직 레포의 clone과 worktree를 Claude가 샌드박스에서 직접 다루며, 코드 변경은 `rin/` 브랜치와 PR로 올립니다(DEPLOY.md).
- 배포 인프라(GCP 호스트, 권한)는 infra 레포의 Terraform이 관리합니다(DEPLOY.md).
- 팀 도구이므로 앞뒤로 연결되는 HopCharlie 레포는 없는 것으로 보이나, 단정하지 않습니다.

## 담당

조직 프로필 README의 RnR 표(조서현, 김동윤, 강종욱, 박준석, 이태훈)에 Rin이나 Slack 봇 담당은 적혀 있지 않습니다. 담당자는 **확인 필요**입니다.

## 구조

| 경로 | 역할 |
|---|---|
| `src/gateway` | Slack Socket Mode로 멘션, DM, 버튼 클릭을 받아 큐에 넣음 |
| `src/worker` | 큐에서 작업을 꺼내 Claude Agent SDK로 실행, 상태 메시지 갱신, 예약 작업 투입 |
| `src/agent` | Claude 실행 옵션과 가드, Rin 도구(`tools.ts`의 `mcp__rin__*`) |
| `src/slack`, `src/github`, `src/cloudflare` | Slack, GitHub, Cloudflare 연동 |
| `src/db` | Postgres 접근(`jobs`, `threads`, `worktrees`, `schedules`, `settings`) |
| `prompts` | 프롬프트 |
| `workspace-template` | 작업 공간 템플릿(CLAUDE.md, 스킬) |
| `deploy/aws`, `deploy/gcp`, `deploy/ec2` | 배포 설정. `ec2`는 GCP에서도 쓰는 공통 배포 스크립트 이름 |
| `compose.dev.yaml`, `Dockerfile` | 로컬 개발 구성과 이미지 |
| `DEPLOY.md` | 현재 배포 방식(교대 배포, 샌드박스, worker 규칙) |
| `ACCOUNTS.md` | Rin이 외부 서비스에 접근할 때 쓰는 신원(계정) 목록 |
| `test` | 테스트 |

Rin 도구는 `slack_read_thread`, `slack_list_messages`, `slack_search_messages`, `slack_send_dm`(팀원에게만), `schedule_create/list/update/delete`입니다. gateway와 worker는 Postgres로만 이어지고, 동시 실행은 최대 4개입니다.

## 실행과 테스트

필요한 환경 변수 이름은 `.env.example` 기준으로 `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `RIN_CHANNEL_ID`, `RIN_ALLOWED_USERS`, `ANTHROPIC_API_KEY`, `RIN_MODEL`, `GITHUB_APP_*`, `DATABASE_URL`, `RIN_WORKSPACE` 등입니다. 값은 `.env`에만 두며 Git에 올리지 않습니다.

```sh
npm ci
cp .env.example .env                                   # 토큰을 채운다
docker build -t rin:local .
docker compose -f compose.dev.yaml --profile app up -d  # postgres, gateway, worker를 컨테이너로 띄운다
```

```sh
npm run typecheck
npm test
docker compose -f compose.dev.yaml up -d postgres
docker compose -f compose.dev.yaml exec postgres createdb -U rin rin_test   # 처음 한 번
# 테이블을 비우는 검사라서 이름이 _test로 끝나는 DB에서만 돈다
DATABASE_URL=postgres://rin:rin@localhost:5433/rin_test node --import tsx test/db.smoke.ts
```

Notion MCP 로그인은 README의 "로컬 실행" 절 명령을 따릅니다. 같은 Slack 앱으로 로컬과 운영을 동시에 띄우지 않습니다(DEPLOY.md).

## 연결된 파트와 문서

- [README 원문](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/README.md): 부르는 방법, 구성, Slack 앱 manifest
- [DEPLOY.md](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/DEPLOY.md): 현재 배포 방식과 worker 사이의 약속
- [GCP runbook](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/deploy/gcp/README.md): 복구와 재부팅 절차
- [ACCOUNTS.md](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/ACCOUNTS.md): Rin이 쓰는 외부 계정 목록
- [src/agent/tools.ts](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/src/agent/tools.ts): Rin 도구 정의
- [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra): GCP 호스트와 배포 권한을 관리하는 Terraform(DEPLOY.md가 언급)

## 알려진 한계와 미완

README 자체에는 한계나 미완 항목이 없습니다. DEPLOY.md에 적힌 운영상 제약만 옮깁니다.

- 로컬 개발(`compose.dev.yaml`)에는 샌드박스가 없고, `RIN_SANDBOX_IMAGE`가 없으면 Claude가 worker 컨테이너 안에서 돕니다(DEPLOY.md "샌드박스").
- 배포 결과는 Slack에 알리지 않으며 GitHub Actions 실행 결과로 확인합니다(DEPLOY.md "worker 사이의 약속").
- 작업은 2시간을 넘으면 큐에 다시 들어가고, 자동 재개는 작업마다 한 번만 합니다(DEPLOY.md).
- 같은 Slack 앱으로 로컬과 운영을 동시에 띄울 수 없습니다(DEPLOY.md "현재 상태").

## 더 보려면

[rin README 전문](https://github.com/2026-Softbank-Hackathon-Charlie/rin/blob/main/README.md)
