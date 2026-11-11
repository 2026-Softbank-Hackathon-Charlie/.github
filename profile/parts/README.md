# 레포별 파트 문서

HopCharlie는 여러 레포로 나뉘어 있습니다. 이 폴더는 **레포마다 하는 일, 전체 이전 흐름에서의 위치, 담당, 구조, 실행 방법, 한계**를 한 곳에서 빠르게 훑어볼 수 있게 정리한 입구입니다. 자세한 내용은 각 레포의 README와 ADR이 원본이고, 이 문서는 그 요약과 링크입니다.

> 작성 기준: 2026-11-11. 각 문서 첫머리에 근거로 삼은 레포의 main 커밋을 적었습니다. 이 문서는 그 시점 README와 문서에서 확인되는 내용만 담았고, 확인하지 못한 것은 "확인 필요"로 남겼습니다. 레포가 바뀌면 문서와 어긋날 수 있으니 원본 README를 함께 보세요.

## 어디부터 읽을까

이전 한 번의 흐름(계획 → 리허설 → 승인 → 이전 → 전환 → 검증) 순서대로 읽으면 전체가 이어집니다.

| 순서 | 레포 | 한 줄 | 문서 |
|---|---|---|---|
| 1 | ai-sandbox | 이전 계획을 AI가 쓰고, 승인 전에 격리 샌드박스에서 먼저 돌려 본다 | [ai-sandbox.md](ai-sandbox.md) |
| 2 | platform | 계획의 승인과 이전 실행, 트래픽 전환을 맡는 서버와 게이트웨이 | [platform.md](platform.md) |
| 3 | migration-engine | DB(CNPG)와 파일(S3)을 다른 클러스터로 옮기는 이전 엔진 | [migration-engine.md](migration-engine.md) |
| 4 | frontend | 운영 콘솔과 로컬 데모(시뮬레이션) | [frontend.md](frontend.md) |
| 5 | cli | 플랫폼 API를 터미널에서 쓰는 운영 CLI `charlie` | [cli.md](cli.md) |
| 6 | infra | HopCharlie를 운영하는 Terraform·Helm·Argo CD 인프라 저장소 | [infra.md](infra.md) |
| 7 | gitops | 플랫폼이 앱의 배포 상태를 기록하는 저장소(플랫폼이 자동 생성) | [gitops.md](gitops.md) |

## 샘플 앱과 부속 레포

| 레포 | 용도 | 문서 |
|---|---|---|
| moimroom | 기본 샘플 앱(공간 예약 서비스 원본) | [moimroom.md](moimroom.md) |
| moimroom-mig | 원본 사본에 관측과 이전 시험 기록을 더한 시연 앱 | [moimroom-mig.md](moimroom-mig.md) |
| moimroom-legacy | 서버 한 대에서 Compose로 운영하는 구성(기존 앱 가져오기 시험의 입력으로 보이나 확인 필요) | [moimroom-legacy.md](moimroom-legacy.md) |
| rin | 팀 Slack에서 부르는 Claude Code 에이전트 봇 | [rin.md](rin.md) |

## 이 문서에 없는 레포

`.github`(조직 프로필), `docker-doom`, `pxls`, `slack-backup`은 이번에 문서를 만들지 않았습니다. 이전 흐름을 이루는 레포가 아니거나, 해커톤 시스템과 직접 관련이 없어 보이는 레포입니다(내용은 확인하지 않았습니다). 필요하면 추가하세요.

## 각 문서의 구성

모든 문서가 같은 순서입니다: 하는 일 → 전체 흐름에서의 위치 → 담당 → 구조 → 실행과 테스트 → 연결된 문서 → 알려진 한계 → 더 보려면.

- **담당**은 조직 프로필 README의 팀 구성원 RnR 표를 근거로 했습니다. 표에서 레포 단위로 확정할 수 없는 곳은 "확인 필요"입니다.
- **실행과 테스트**의 명령은 README에 있는 것을 옮겼고, 비밀값은 `<...>` 자리표시자나 변수 이름만 적었습니다. 운영 클러스터에 영향을 주는 명령은 README의 주의 문구를 그대로 옮겼습니다.
- **알려진 한계**는 README와 ADR에 적힌 것만 모았습니다. 작성 시점 이후 해결됐을 수 있습니다.

전체 소개는 [조직 프로필 README](../README.md)를 보세요.
