# frontend: HopCharlie 운영 콘솔과 로컬 데모

> 작성 기준: 2026-11-11 · 근거: 이 레포의 README와 문서(main `1e61460`)

## 이 파트가 하는 일

HopCharlie 운영 콘솔입니다. 운영자가 앱과 이전 방향을 고르고, AI 계획과 리허설 결과를 검토해 승인하고, 동기화부터 전환·검증까지의 진행과 결과를 보는 웹 화면입니다. 같은 화면 컴포넌트를 두 가지 데이터 공급 경로로 씁니다. **일반 모드**는 관리자 로그인 후 실제 플랫폼 API를 읽고, **데모 모드**는 로그인 없이 시연용 시뮬레이션으로 같은 화면을 보여 줍니다. 데모는 플랫폼 계약과 같은 데이터 모양을 만들기 때문에, 실제 환경이 없는 곳(로컬 데모)에서도 화면과 흐름을 확인할 수 있습니다. 이 외에 GitHub 레포로 앱을 등록·배포하는 화면, 앱 시크릿, 보안, 환경, AI 활동, CLI 로그인 승인 화면도 이 레포에 있습니다.

## 전체 흐름에서의 위치

- **계획 → 리허설 → 승인**: 콘솔이 `ai-sandbox`의 AI 실행기(`hopai run`)가 만든 계획 세션과 리허설·진단 세션을 실시간으로 보여 주고, 사람이 보고 있는 revision으로 승인합니다.
- **이전 → 전환 → 검증**: 승인한 Plan으로 이전을 시작하고, 동기화·읽기 카나리·전환·검증 4묶음(엔진 9단계)의 진행, 전환 승인 대기(`awaiting_switch`)에서의 전환 버튼, 이전 리포트, 1-click 롤백을 제공합니다.
- 앞뒤로 연결되는 레포: `platform`(콘솔이 읽는 API 계약 `api/openapi.yaml`, 인증, 승인·실행 상태), `ai-sandbox`(AI 계획·리허설), `cli`(`charlie login`의 코드를 콘솔 CLI 연결 화면에서 승인), `infra`(콘솔 이미지를 GitOps로 배포), 이전 엔진은 플랫폼을 거쳐 간접 연결됩니다.
- 콘솔 자체의 배포: main push마다 이미지를 ECR에 게시하고 infra main에 커밋하는 GitHub Actions를 이 레포가 가집니다.

## 담당

조직 프로필 README의 RnR 표에서 이 레포를 이름으로 지정한 항목은 없습니다. 콘솔과 관련된 항목만 옮깁니다. 레포 단위 담당은 확인 필요입니다.

| 사람 | 표에 적힌 관련 역할 |
|---|---|
| 조서현 | RnR에 Console. Main에 "운영 콘솔 실제 API 연동", "플랫폼·콘솔 이미지 게시 자동화" |
| 이태훈 | RnR에 Console. Main에 "운영 콘솔의 이전·AI 실행 흐름" |
| 박준석 | Main에 "프런트엔드 이주 지도" |
| 김동윤 | Assist에 "이주 지도·복구 화면" |

## 구조

| 경로 | 역할 |
|---|---|
| `src/api/` | 플랫폼 API 데이터 계층(계약 타입, 클라이언트, SSE, 훅). 자세한 설명은 `src/api/README.md` |
| `src/data/live.ts` | 모드와 로그인 세션(localStorage, 탭 간 동기화, 만료, 401 처리) |
| `src/data/migration.ts` | 이전 화면용 보기 모델. 시뮬레이션과 실제 API를 같은 모양으로 맞춤 |
| `src/data/deploy.ts`, `secrets.ts` | 새 앱 배포·앱 목록·배포 진행, 앱 시크릿 보기 모델 |
| `src/sim/` | 계약 모양으로 만드는 시뮬레이션(이전, 앱 배포, 시크릿)과 상태 머신 |
| `src/pages/` | 화면. 모드별로 감싸지만 그리는 코드는 하나. `Entry.tsx`는 모드 선택·로그인, `Device.tsx`는 CLI 연결 |
| `src/components/` | 공통 UI(Chip, Tabs, LogBox, Table, AI 세션 표, 배포·시크릿 조각 등) |
| `src/router.ts` | 해시 라우팅(`#overview`, `#apps`, `#migrate`, `#security`, `#device` 등) |
| `src/styles/` | 디자인 토큰(`tokens.css`)과 화면별 CSS |
| `docs/HISTORY.md`, `docs/screenshots/` | PR별 화면 히스토리와 README용 스크린샷 |
| `docs/adr/` | 콘솔 ADR(이주 지도, 쓰기 기록 증거, AI 이전 추천 등) |
| `.github/workflows/` | `ci.yml`(PR 검증), `deploy.yml`(main push 시 콘솔 이미지 게시) |

## 실행과 테스트

README의 명령입니다.

```bash
npm install
npm run dev        # 개발 서버
npm test           # 시뮬레이션·화면 테스트
npm run build      # 타입 검사 + 프로덕션 빌드
```

실제 API에 연결하려면 프로젝트 루트의 `.env.local`에 `VITE_PLATFORM_URL`(플랫폼 주소)을 적고 `npm run dev`를 다시 시작합니다. 빌드 변수로 `VITE_PLATFORM_URL`과 `VITE_HYPERDX_URL`(비우면 "HyperDX에서 보기" 버튼이 나오지 않음)이 있습니다. API 타입을 다시 만들 때는 `src/api/README.md`의 명령을 씁니다.

```bash
npm run gen:api    # ../platform/api/openapi.yaml 기준으로 src/api/schema.d.ts 재생성
```

### 데모 모드와 일반 모드

처음 열면 모드를 고르며, 위쪽 막대 버튼으로 언제든 바꿀 수 있고 바꿔도 로그아웃되지 않습니다.

| 구분 | 일반 모드 | 데모 모드 (시뮬레이션) |
|---|---|---|
| 데이터 | 실제 플랫폼 API (`src/api`) | 계약과 같은 모양으로 만든 가상 데이터 (`src/sim`) |
| 로그인 | 관리자 아이디·비밀번호(플랫폼 서버의 `PLATFORM_ADMINS`). 세션 12시간, 새로고침·새 탭에서도 유지 | 로그인 없음. "처음 상태로 되돌리기"로 초기화 |
| 이전 | AI 계획·리허설, 수동 Plan, 읽기 카나리와 전환 후 관찰·자동 롤백 선택 | AI 계획·리허설과 빠른 카나리 시나리오. 수동 Plan·관찰·자동 롤백은 비활성화 |
| 보안 | 발견 항목·수집 상태·저장소 키는 실제 API. 통제 평가·CVE는 미연동 | 같은 5개 탭. 통제·CVE·자산 관계는 명시한 가상 데이터 |
| 기존 앱 가져오기 | 실제 API 미연동 안내. 자격증명 입력·시연 실행 없음 | 기존 가상 가져오기 흐름. 실제 자격증명을 입력하지 않음 |
| 메뉴 배지 | 없음(API 호출을 늘리지 않으려고) | 진행 중 상태와 건수 표시 |

**MOCK 표시**: 시뮬레이션의 수치(파일 수, DB 크기, 보안 시연 데이터)는 자리표시이고, 코드에서 `MOCK:` 주석을 검색하면 나옵니다. 시뮬레이션의 이전 대상은 샘플 앱 moimroom의 구성(서비스 `web` 레플리카 2개, CNPG `moimroom-db`, 스키마 `app` 테이블 5개, 사진 버킷)을 따릅니다.

## 연결된 파트와 문서

- [README 화면 절](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/README.md#화면): 모드 선택, 개요, 이전, 앱, 보안 화면 설명
- [README 이전 흐름 절](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/README.md#이전-흐름): 단계별 시뮬레이션과 실제 API 대응 표
- [README 배포 절](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/README.md#배포-github-actions): 콘솔 이미지 게시와 GitOps 흐름
- [버전 히스토리와 화면](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/docs/HISTORY.md)
- [플랫폼 API 데이터 계층](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/src/api/README.md)
- [이주 지도 ADR 0001](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/docs/adr/0001-state-driven-migration-map-iframe.md)
- [AI 이전 추천 ADR 0014](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/docs/adr/0014-on-demand-ai-migration-recommendations.md)
- [콘솔 배포 workflow](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/.github/workflows/deploy.yml)
- [platform API 계약](https://github.com/2026-Softbank-Hackathon-Charlie/platform/blob/main/api/openapi.yaml)
- 다른 레포: [platform](https://github.com/2026-Softbank-Hackathon-Charlie/platform), [ai-sandbox](https://github.com/2026-Softbank-Hackathon-Charlie/ai-sandbox), [cli](https://github.com/2026-Softbank-Hackathon-Charlie/cli), [infra](https://github.com/2026-Softbank-Hackathon-Charlie/infra)

## 알려진 한계와 미완

- 기존 앱 가져오기(AI 점검, 샌드박스 테스트, PR 승인, 첫 배포)는 AI 범위 결정 전 시안이며 일반 모드에서는 실제 API에 연결되지 않았습니다. (실제 API에 연결하기, 이전 흐름)
- 보안 화면의 개별 통제 평가, 구조화된 CVE, 발견 항목과 앱의 관계는 현재 API에 없어 일반 모드에서 미연동이고 앱 필터가 비활성화됩니다. 이 화면은 스캐너·정책 변경·예외 승인을 실행하지 않습니다. (화면, AI 활동·보안)
- 데모 모드의 이전은 정해진 시나리오 범위입니다. 수동 Plan, 전환 후 관찰, 자동 롤백은 데모에서 쓸 수 없습니다. (README 개요 표)
- 읽기 카나리는 플랫폼 게이트웨이(http)를 거치는 Plan에서만 동작합니다. (화면 > 이전)
- 배포 workflow의 성공은 ECR 게시와 infra main 커밋까지이고, 실제 콘솔 반영은 관리 EKS의 Argo CD 동기화가 끝나야 완료됩니다. (배포 GitHub Actions)
- "HyperDX에서 보기"의 주소는 기본으로 공개되어 있지 않아 시연 PC에서는 포트 포워딩한 주소를 써야 합니다. (배포 > 최초 준비)

## 더 보려면

원문: [frontend README](https://github.com/2026-Softbank-Hackathon-Charlie/frontend/blob/main/README.md)
