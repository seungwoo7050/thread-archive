===== BEGIN FILE: 01-workspace-package-and-composition-boundaries.md =====
# Workspace·package·composition 경계

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

단일 저장소가 shared, database, API, web package로 분리되고 application bootstrap이 각 dependency의 생성·초기화·종료 책임을 조립하는 실제 순서를 복원합니다.

이 문서는 Phase 1 category audit 후 동결된 scaffold를 기준으로 exact SHA의 구현 발전을 복원합니다.

### 직접 연결되는 불변식

- workspace dependency는 root → package command 위임과 `shared`/DB → API/web의 단방향 소비로 구성됩니다.
- shared package는 transport representation을 소유하지만 persistence driver와 browser/runtime implementation을 포함하지 않습니다.
- PostgreSQL pool과 Kysely client는 repository가 소유하고 composition root는 backend 선택·초기화·종료 호출을 소유합니다.
- web application은 shared TypeScript source를 transpile하되 DB implementation을 browser runtime에 포함하지 않습니다.

## 2. 핵심 질문

- 실제 역사에서 compile-time DTO와 game contract가 DB/API/web package보다 먼저 도입된 이유는 무엇입니까?
- DB package가 dependency scaffold에서 import 가능 migration boundary, repository lifecycle로 발전하는 단계는 무엇입니까?
- `apps/api/src/index.ts`가 생성하는 object와 각 object가 내부적으로 소유하는 resource를 구분할 수 있습니까?
- 초기 bootstrap이 정상 close에서 보장하는 것과 startup 초기화 실패에서 아직 보장하지 않는 것은 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, symbol, caller/callee, 상태 mutation, failure branch, cleanup을 actual historical code로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- S/A/B/C 중요도에 맞춰 설명 깊이를 다르게 유지합니다.
- 실행하지 않은 command 결과를 작성하지 않으며 code inspection과 runtime evidence를 구분합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b625c4f9dfdc` | `chore(workspace): pnpm 모노레포 경계 구성` | B | PERSISTENCE | 루트 명령과 `apps/*`, `packages/*` workspace를 만들고 공통 TypeScript 기준을 둡니다. |
| 2 | `7753ad1fafaf` | `chore(shared): 공유 패키지 경계 구성` | B | PROTOCOL | API와 browser가 함께 소비할 `@pong-pong/shared` workspace를 독립 package로 만듭니다. |
| 3 | `573d11acb75e` | `feat(shared): 사용자와 서비스 DTO 정의` | B | PERSISTENCE, TOURNAMENT, WEB | 사용자·경기·대시보드·친구·채팅·토너먼트의 초기 compile-time DTO와 공개/세션 개인정보 경계를 정의합니다. |
| 4 | `41471c2c2d55` | `feat(shared): 퐁 시뮬레이션 계약 추가` | B | SIMULATION, REALTIME, WEB | 서버 simulation과 browser renderer가 공유할 geometry·timing·snapshot·finished-result 계약을 정의합니다. |
| 5 | `f77297697c66` | `chore(db): PostgreSQL 패키지 경계 구성` | B | PERSISTENCE | PostgreSQL, Kysely, pg 의존성을 API와 분리한 `@pong-pong/db` workspace를 만듭니다. |
| 6 | `1140fb868714` | `feat(db): migration 실행 경계 구성` | B | PERSISTENCE | DB package를 import 가능하게 만들고 초기 schema SQL을 runtime에서 실행할 entry로 노출합니다. |
| 7 | `9277572765e7` | `feat(db): 저장소 lifecycle 구성` | B | PERSISTENCE, OPERATIONS | PostgreSQL과 memory implementation을 공통 `AppRepository` lifecycle 뒤에 두고 생성·초기화·종료 owner를 정의합니다. |
| 8 | `51484e00a1c2` | `chore(api): Fastify 패키지 경계 구성` | B | AUTH, PROTOCOL, REALTIME | Fastify HTTP·cookie·CORS·WebSocket과 shared/DB dependency를 가진 API workspace를 구성합니다. |
| 9 | `4b43a284e637` | `feat(api): 실행 환경과 service bootstrap 구성` | B | PERSISTENCE, OPERATIONS | 환경을 읽고 repository를 선택·초기화한 뒤 Fastify를 시작하고 종료 시 repository를 닫는 composition root를 도입합니다. |
| 10 | `f5c151c7cc7d` | `chore(web): Next.js runtime 경계 구성` | B | PROTOCOL, PERSISTENCE, WEB | Next.js browser application workspace와 shared source transpilation 경계를 구성합니다. |

## 5. Commit별 학습 기록

### 5.1. `chore(workspace): pnpm 모노레포 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `b625c4f9dfdc` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | 루트 명령과 `apps/*`, `packages/*` workspace를 만들고 공통 TypeScript 기준을 둡니다. |

#### 해당 SHA에서 확인할 실제 코드

- 루트 `package.json`, `pnpm-workspace.yaml`, `Makefile`, `tsconfig.base.json`에서 workspace glob과 recursive command를 확인합니다.
- 각 package가 자체 command를 유지하면서 루트 `build`와 `typecheck`가 어떤 순서로 위임되는지 확인합니다.
- ES2022, module resolution, strictness가 이후 package 설정의 공통 기준이 되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 직전 root에는 프로젝트 설명만 있었고 실행 애플리케이션과 재사용 package를 묶는 package manager·compiler 기준이 없었습니다. |
| 해결하려던 문제 | 하나의 저장소에서 여러 실행 단위와 라이브러리를 추가하면 설치·검증 명령과 TypeScript 의미가 package별로 갈라질 수 있었습니다. |
| 핵심 결정 | `apps/*`와 `packages/*`를 pnpm workspace로 선언하고 루트 script와 Make target이 각 workspace 명령을 재귀 호출하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | 개발자가 루트 명령을 실행하면 pnpm이 각 workspace의 `build` 또는 `typecheck`를 호출하고 결과를 루트 종료 상태로 모읍니다. |
| ownership/lifetime/cleanup | 루트가 package 목록과 공통 compiler 기준을 소유하고, 개별 workspace는 자신의 dependency와 명령을 소유합니다. 이 SHA에는 장기 실행 resource가 없습니다. |
| failure/rollback/retry | 한 workspace 명령이 실패하면 recursive command 전체가 실패합니다. 자동 rollback이나 부분 성공 캐시는 정의하지 않습니다. |
| 보장하는 것 | 새 package를 정해진 glob 아래 추가하면 공통 설치·검증 경로에 참여할 수 있습니다. |
| 보장하지 않는 것 | 아직 실제 `shared`, DB, API, web package나 package 간 dependency 방향은 구현하지 않습니다. |
| 후속 연결 | `7753ad1fafaf`부터 각 workspace가 실제 책임을 획득합니다. |

#### 최소 코드 근거

`b625c4f9dfdc`의 `pnpm-workspace.yaml`, 루트 `package.json`, `tsconfig.base.json`이 근거입니다. 별도 코드 발췌 없이 설정 항목 자체로 충분합니다.

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `7753ad1fafaf` — `chore(shared): 공유 패키지 경계 구성`

### 5.2. `chore(shared): 공유 패키지 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `7753ad1fafaf` |
| Importance | B |
| Tags | PROTOCOL |
| Source에서 확정된 역할 | API와 browser가 함께 소비할 `@pong-pong/shared` workspace를 독립 package로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/package.json`의 `name`, `exports`, Zod dependency와 package-local script를 확인합니다.
- `packages/shared/tsconfig.json`이 root compiler policy를 확장하는지 확인합니다.
- 루트 path mapping이 source entry를 직접 가리키며 prebuild 없이 어떤 소비 조건을 요구하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | workspace 골격은 있었지만 둘 이상의 애플리케이션이 공유할 contract owner는 없었습니다. |
| 해결하려던 문제 | API와 web이 같은 타입을 별도로 정의하면 transport shape와 validation rule이 빠르게 달라질 수 있었습니다. |
| 핵심 결정 | `@pong-pong/shared`를 source entry와 package-local typecheck를 가진 독립 workspace로 추가하고 Zod를 runtime dependency로 선언했습니다. |
| 입력 → 상태 전이 → 출력 | consumer import가 root path mapping을 통해 `packages/shared/src/index.ts`로 해석되고 TypeScript가 같은 source를 검사합니다. |
| ownership/lifetime/cleanup | shared package가 공통 contract source를 소유합니다. 소비 애플리케이션은 해당 source를 transpile할 책임을 아직 명시적으로 해결해야 합니다. |
| failure/rollback/retry | package build는 `tsc --noEmit`이므로 배포 JavaScript artifact를 만들지 않습니다. 소비자가 TS source를 처리하지 못하면 runtime import는 실패할 수 있습니다. |
| 보장하는 것 | 공유 계약을 한 package에서 정의하고 두 애플리케이션이 같은 symbol을 import할 기반을 만듭니다. |
| 보장하지 않는 것 | 아직 DTO나 runtime schema 자체는 없으며 순환 dependency를 자동 검출하는 별도 규칙도 없습니다. |
| 후속 연결 | `573d11acb75e`와 `41471c2c2d55`가 실제 공유 표현을 추가합니다. |

#### 최소 코드 근거

`packages/shared/package.json`, `packages/shared/tsconfig.json`, root path mapping을 확인했습니다.

비교 기준:
- 직전 관련 SHA: `b625c4f9dfdc` — `chore(workspace): pnpm 모노레포 경계 구성`
- 다음 관련 SHA: `573d11acb75e` — `feat(shared): 사용자와 서비스 DTO 정의`

### 5.3. `feat(shared): 사용자와 서비스 DTO 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `573d11acb75e` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT, WEB |
| Source에서 확정된 역할 | 사용자·경기·대시보드·친구·채팅·토너먼트의 초기 compile-time DTO와 공개/세션 개인정보 경계를 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`의 `PublicUser`, `SessionUser` 및 서비스 DTO를 비교합니다.
- email, online, 날짜 문자열이 storage row나 JavaScript `Date` 대신 transport 표현으로 정의되는지 확인합니다.
- `packages/shared/src/index.ts`가 해당 계약을 외부에 노출하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared package는 비어 있었고 API·DB·browser가 합의할 사용자 및 서비스 응답 shape가 없었습니다. |
| 해결하려던 문제 | 각 계층이 필드 이름, nullable 여부, 개인정보 노출 범위를 독자적으로 선택할 위험이 있었습니다. |
| 핵심 결정 | 공개 사용자는 email을 제외하고 `SessionUser`만 이를 확장하도록 하며, presence와 시간 값을 transport 친화적 필드로 표현했습니다. |
| 입력 → 상태 전이 → 출력 | producer가 DTO shape를 구성하고 consumer의 TypeScript가 동일 interface를 기준으로 필드 접근을 검사합니다. |
| ownership/lifetime/cleanup | shared package가 외부 표현을 소유하지만 값의 생성과 수명은 API·repository가 소유합니다. `online`은 영속 row가 아니라 runtime context입니다. |
| failure/rollback/retry | compile-time interface이므로 잘못된 JSON이나 untyped repository 결과를 runtime에서 거부하지 않습니다. |
| 보장하는 것 | public/session privacy 차이와 기본 서비스 aggregate의 필드 이름을 한 곳에 고정합니다. |
| 보장하지 않는 것 | 값 범위, unknown field, JSON input은 검증하지 않으며 persistence schema나 상태 전이도 구현하지 않습니다. |
| 후속 연결 | 후속 HTTP runtime contract 커밋들이 같은 표현을 Zod schema로 교체합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 `PublicUser`와 이를 확장한 `SessionUser`가 개인정보 차이를 코드로 드러냅니다.

비교 기준:
- 직전 관련 SHA: `7753ad1fafaf` — `chore(shared): 공유 패키지 경계 구성`
- 다음 관련 SHA: `41471c2c2d55` — `feat(shared): 퐁 시뮬레이션 계약 추가`

### 5.4. `feat(shared): 퐁 시뮬레이션 계약 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `41471c2c2d55` |
| Importance | B |
| Tags | SIMULATION, REALTIME, WEB |
| Source에서 확정된 역할 | 서버 simulation과 browser renderer가 공유할 geometry·timing·snapshot·finished-result 계약을 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/game.ts`의 court, paddle, ball, score, tick 상수를 확인합니다.
- `GameSnapshot`이 phase, tick, score, player, server time을 모두 포함하는지 확인합니다.
- 계약이 physics 실행이나 mutable room state를 포함하지 않고 serialization boundary에 머무는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 서비스 DTO는 있었지만 game server와 browser가 사용할 공통 좌표계·snapshot shape는 없었습니다. |
| 해결하려던 문제 | 서버와 renderer가 서로 다른 크기, tick rate, phase 이름을 사용하면 같은 경기 상태를 다르게 해석할 수 있었습니다. |
| 핵심 결정 | 고정 geometry와 timing 상수, 제한된 paddle direction, 완전한 snapshot 및 finished result 타입을 shared에 정의했습니다. |
| 입력 → 상태 전이 → 출력 | 서버가 shared shape의 snapshot을 만들고 browser가 같은 상수와 필드를 사용해 렌더링하도록 dependency 방향을 정합니다. |
| ownership/lifetime/cleanup | 계약만 shared가 소유하며 실제 simulation state, room lifecycle, socket은 이후 API subsystem이 소유합니다. |
| failure/rollback/retry | 이 SHA에는 runtime parser, physics, ordering, retry 또는 cleanup이 없습니다. |
| 보장하는 것 | server와 browser가 동일한 직렬화 표현과 좌표계를 참조할 수 있습니다. |
| 보장하지 않는 것 | snapshot이 authoritative하다는 실행 보장, 상태 전이의 합법성, 네트워크 순서 보장은 아직 없습니다. |
| 후속 연결 | 후속 realtime category가 이 계약 위에 simulation·room·transport를 구현합니다. |

#### 최소 코드 근거

`packages/shared/src/game.ts`의 상수와 `GameSnapshot`/finished result type이 근거입니다.

비교 기준:
- 직전 관련 SHA: `573d11acb75e` — `feat(shared): 사용자와 서비스 DTO 정의`
- 다음 관련 SHA: `f77297697c66` — `chore(db): PostgreSQL 패키지 경계 구성`

### 5.5. `chore(db): PostgreSQL 패키지 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f77297697c66` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | PostgreSQL, Kysely, pg 의존성을 API와 분리한 `@pong-pong/db` workspace를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/package.json`에서 shared·Kysely·pg dependency와 tooling을 확인합니다.
- `packages/db/tsconfig.json`이 root compiler policy를 상속하는지 확인합니다.
- 이 SHA에 repository implementation이나 export surface가 아직 없는지 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared contract는 있었지만 persistence driver를 둘 위치와 API의 dependency 방향은 정해지지 않았습니다. |
| 해결하려던 문제 | Fastify route가 직접 `pg`나 Kysely를 소유하면 transport와 storage lifecycle이 결합될 수 있었습니다. |
| 핵심 결정 | `@pong-pong/db` workspace를 만들고 database driver·query builder dependency를 그 package에 한정했습니다. |
| 입력 → 상태 전이 → 출력 | 향후 API는 DB package의 exported operation만 import하고 driver-specific code는 packages/db 내부에서 실행하도록 방향을 잡았습니다. |
| ownership/lifetime/cleanup | DB package가 persistence dependency를 소유하지만 아직 pool, client, close owner는 없습니다. |
| failure/rollback/retry | 실행 entry와 export가 없으므로 이 상태만으로 API가 DB 기능을 사용할 수 없습니다. |
| 보장하는 것 | 물리적인 package dependency 경계를 만들고 transport package에서 driver dependency를 제거할 수 있습니다. |
| 보장하지 않는 것 | repository abstraction, migration 실행, transaction 및 실제 row mapping은 보장하지 않습니다. |
| 후속 연결 | `1140fb868714`이 import·migration 실행 경계를, `9277572765e7`이 lifecycle을 추가합니다. |

#### 최소 코드 근거

`packages/db/package.json`과 package-local TypeScript 설정이 근거입니다.

비교 기준:
- 직전 관련 SHA: `41471c2c2d55` — `feat(shared): 퐁 시뮬레이션 계약 추가`
- 다음 관련 SHA: `1140fb868714` — `feat(db): migration 실행 경계 구성`

### 5.6. `feat(db): migration 실행 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `1140fb868714` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | DB package를 import 가능하게 만들고 초기 schema SQL을 runtime에서 실행할 entry로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/package.json`의 `exports`, build/typecheck script를 확인합니다.
- `packages/db/src/migrations.ts`의 `initialMigrationSql`과 기존 SQL file 사이 중복 소유 상태를 확인합니다.
- `tsconfig.base.json`의 `@pong-pong/db` path mapping이 consumer import를 가능하게 하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | DB workspace는 dependency만 있었고 consumer가 import할 package export나 실행 가능한 schema 초기화 값이 없었습니다. |
| 해결하려던 문제 | API bootstrap이 storage package를 조립하려면 stable package entry와 code에서 호출 가능한 migration boundary가 필요했습니다. |
| 핵심 결정 | package export와 root alias를 추가하고 초기 schema를 `initialMigrationSql` 문자열로 export했습니다. |
| 입력 → 상태 전이 → 출력 | consumer가 `@pong-pong/db`를 import하고 이후 repository가 SQL string을 Kysely에 제출할 수 있게 됩니다. |
| ownership/lifetime/cleanup | DB package가 schema bootstrap text를 소유하지만 이 SHA에는 pool이나 migrator lifecycle owner가 없습니다. |
| failure/rollback/retry | 독립 SQL file과 문자열이 중복되어 자동 동기화되지 않으며 versioned migration registry도 없습니다. |
| 보장하는 것 | API가 directory 내부 경로에 의존하지 않고 DB package를 import할 수 있습니다. |
| 보장하지 않는 것 | 실제 schema 실행, seed, rollback, migration version 추적은 보장하지 않습니다. |
| 후속 연결 | `9277572765e7`의 `ensureSeedData`가 이 SQL을 실행합니다. |

#### 최소 코드 근거

`packages/db/package.json`, `packages/db/src/migrations.ts`, `tsconfig.base.json`의 세 변경을 함께 확인했습니다.

비교 기준:
- 직전 관련 SHA: `f77297697c66` — `chore(db): PostgreSQL 패키지 경계 구성`
- 다음 관련 SHA: `9277572765e7` — `feat(db): 저장소 lifecycle 구성`

### 5.7. `feat(db): 저장소 lifecycle 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `9277572765e7` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | PostgreSQL과 memory implementation을 공통 `AppRepository` lifecycle 뒤에 두고 생성·초기화·종료 owner를 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository`, factory, `PostgresRepository`, `MemoryRepository`를 확인합니다.
- PostgreSQL factory가 `Pool`과 `Kysely`를 함께 생성하고 `close()`가 어떤 순서로 종료하는지 확인합니다.
- `ensureSeedData()`가 `initialMigrationSql`을 실행하며 memory backend는 같은 method를 no-op으로 제공하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | DB package는 import 가능했지만 pool/client 생성, 초기화, 종료를 호출자가 일관되게 다룰 공통 contract가 없었습니다. |
| 해결하려던 문제 | API가 backend별 분기를 갖거나 pool을 닫지 않으면 startup/shutdown 경로마다 자원 누수가 달라질 수 있었습니다. |
| 핵심 결정 | `AppRepository`에 `ensureSeedData()`와 `close()`를 두고 PostgreSQL과 memory factory가 같은 interface를 반환하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | factory가 pool과 Kysely를 생성해 repository에 넘기고, 초기화는 SQL 실행, 종료는 `db.destroy()` 후 중복 pool 종료 오류를 흡수합니다. |
| ownership/lifetime/cleanup | PostgresRepository가 pool과 Kysely의 lifetime을 소유합니다. MemoryRepository는 동일 lifecycle 호출을 허용하는 no-op owner입니다. |
| failure/rollback/retry | 초기 SQL 실행이 실패하면 이 method 자체는 예외를 전달하며 rollback이나 자동 close를 수행하지 않습니다. |
| 보장하는 것 | 상위 composition root가 backend 종류와 무관하게 동일한 초기화·종료 sequence를 사용할 수 있습니다. |
| 보장하지 않는 것 | 이 시점의 `AppRepository`에는 실제 도메인 operation이 없고 startup 중 초기화 실패 시 자동 정리는 아직 보장되지 않습니다. |
| 후속 연결 | `4b43a284e637`이 repository 선택과 app shutdown에 lifecycle을 연결합니다. |

#### 최소 코드 근거

`9277572765e7`, `packages/db/src/index.ts`:

```ts
export interface AppRepository {
  close(): Promise<void>;
  ensureSeedData(): Promise<void>;
}
```

factory가 만든 pool과 Kysely는 `PostgresRepository.close()`에서 정리됩니다.

비교 기준:
- 직전 관련 SHA: `1140fb868714` — `feat(db): migration 실행 경계 구성`
- 다음 관련 SHA: `51484e00a1c2` — `chore(api): Fastify 패키지 경계 구성`

### 5.8. `chore(api): Fastify 패키지 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `51484e00a1c2` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | Fastify HTTP·cookie·CORS·WebSocket과 shared/DB dependency를 가진 API workspace를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/package.json`의 Fastify plugin, shared, DB, Zod, ws dependency를 확인합니다.
- API package가 DB driver를 직접 의존하지 않고 `@pong-pong/db`만 소비하는지 확인합니다.
- `apps/api/tsconfig.json`과 script가 아직 실행 entry를 포함하지 않는 초기 경계인지 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared와 DB package는 있었지만 HTTP service를 소유할 executable workspace가 없었습니다. |
| 해결하려던 문제 | HTTP, cookie, CORS, WebSocket을 root나 DB package에 섞지 않고 별도 application owner가 필요했습니다. |
| 핵심 결정 | `apps/api`를 만들고 transport dependency와 내부 workspace dependency를 선언했습니다. |
| 입력 → 상태 전이 → 출력 | 향후 `buildApp`와 process entry가 이 package 안에서 shared contract와 repository를 조립하도록 dependency 방향을 정했습니다. |
| ownership/lifetime/cleanup | API package가 transport runtime을 소유할 예정이지만 이 SHA에는 listener, socket, cleanup handle이 없습니다. |
| failure/rollback/retry | package 설정만 있으므로 HTTP endpoint나 인증·오류 동작은 아직 없습니다. |
| 보장하는 것 | Fastify application code가 들어갈 독립 경계와 필요한 dependency 집합을 고정합니다. |
| 보장하지 않는 것 | 실행 가능 service, configuration, repository selection을 보장하지 않습니다. |
| 후속 연결 | `4b43a284e637`이 process bootstrap을 추가하고 Thread 2의 `1779df300611`이 route를 구현합니다. |

#### 최소 코드 근거

`apps/api/package.json`과 package TypeScript 설정이 근거입니다.

비교 기준:
- 직전 관련 SHA: `9277572765e7` — `feat(db): 저장소 lifecycle 구성`
- 다음 관련 SHA: `4b43a284e637` — `feat(api): 실행 환경과 service bootstrap 구성`

### 5.9. `feat(api): 실행 환경과 service bootstrap 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `4b43a284e637` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | 환경을 읽고 repository를 선택·초기화한 뒤 Fastify를 시작하고 종료 시 repository를 닫는 composition root를 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.ts`의 default port, database URL, origin, session secret 처리 방식을 확인합니다.
- `apps/api/src/index.ts`에서 repository factory 선택 → `ensureSeedData` → `buildApp` → `listen` 순서를 확인합니다.
- Fastify `onClose`와 listen failure catch가 `repo.close()`를 호출하는 경로 및 startup 초기화 실패의 미보장 범위를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API workspace와 repository lifecycle은 있었지만 process가 어떤 backend를 만들고 언제 닫는지 조립하는 owner가 없었습니다. |
| 해결하려던 문제 | repository 선택과 listener startup이 분산되면 memory/PostgreSQL 분기와 cleanup이 여러 entry에서 달라질 수 있었습니다. |
| 핵심 결정 | `index.ts`를 composition root로 두어 환경 읽기, repository 생성, seed/schema 초기화, app 생성, listener 시작을 한 순서로 모았습니다. |
| 입력 → 상태 전이 → 출력 | `DATABASE_URL` 유무로 backend를 선택하고 초기화한 뒤 app을 구성합니다. app close hook과 listen 실패 catch가 repository close를 호출합니다. |
| ownership/lifetime/cleanup | process entry가 repository와 Fastify app의 조립을 소유하고, repository 자체가 pool lifetime을 소유합니다. 정상 shutdown에서는 app close가 repository close로 이어집니다. |
| failure/rollback/retry | `ensureSeedData()`가 `try` 전에 실패하면 catch의 `repo.close()`가 실행되지 않습니다. port 값도 `Number` 변환 후 별도 검증하지 않습니다. |
| 보장하는 것 | 정상 listen 실패와 Fastify close 경로에서 persistence 자원을 닫는 composition rule을 제공합니다. |
| 보장하지 않는 것 | production에서 memory fallback을 막지 않고, 약한 default secret을 허용하며, signal 기반 graceful shutdown은 아직 없습니다. |
| 후속 연결 | Runtime Thread의 `f801ccd09cf0`·`eb675ef74af3`가 환경별 fail-closed rule을 강화합니다. |

#### 최소 코드 근거

`4b43a284e637`, `apps/api/src/index.ts`의 핵심 순서는 다음과 같습니다.

```ts
const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl)
  : createMemoryRepository();
await repo.ensureSeedData();
app.addHook("onClose", async () => repo.close());
```

비교 기준:
- 직전 관련 SHA: `51484e00a1c2` — `chore(api): Fastify 패키지 경계 구성`
- 다음 관련 SHA: `f5c151c7cc7d` — `chore(web): Next.js runtime 경계 구성`

### 5.10. `chore(web): Next.js runtime 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f5c151c7cc7d` |
| Importance | B |
| Tags | PROTOCOL, PERSISTENCE, WEB |
| Source에서 확정된 역할 | Next.js browser application workspace와 shared source transpilation 경계를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/package.json`, `next.config.mjs`, `tsconfig.json`에서 Next.js runtime과 build/typecheck command를 확인합니다.
- `transpilePackages`가 `@pong-pong/shared` source import를 처리하는지 확인합니다.
- web package dependency에 DB implementation이 직접 포함되지 않는지 확인하고 path alias와 실제 package dependency를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API 실행 경계는 있었지만 shared contract를 소비할 browser application workspace가 없었습니다. |
| 해결하려던 문제 | shared가 TypeScript source를 직접 export하므로 Next.js가 이를 외부 precompiled package처럼 처리하면 build가 실패할 수 있었습니다. |
| 핵심 결정 | `apps/web`을 Next.js application으로 만들고 `@pong-pong/shared`를 workspace dependency와 transpile 대상에 추가했습니다. |
| 입력 → 상태 전이 → 출력 | Next build가 web source와 shared source를 함께 transpile하고 browser application은 shared type/constant를 import할 수 있습니다. |
| ownership/lifetime/cleanup | web workspace가 browser runtime/build를 소유합니다. DB implementation은 server-side package에 남고 browser가 pool lifecycle을 소유하지 않습니다. |
| failure/rollback/retry | 이 SHA에는 page, HTTP client, runtime schema parse 또는 network cleanup이 없습니다. |
| 보장하는 것 | browser code와 API/DB package를 물리적으로 분리하면서 shared source를 소비할 build 경로를 제공합니다. |
| 보장하지 않는 것 | package path alias만으로 런타임 접근 제한을 강제하지 않으며 실제 UI·API adapter는 구현하지 않습니다. |
| 후속 연결 | 후속 browser category가 화면·HTTP client·socket owner를 추가합니다. |

#### 최소 코드 근거

`apps/web/next.config.mjs`의 `transpilePackages: ["@pong-pong/shared"]`와 package dependency가 핵심 근거입니다.

비교 기준:
- 직전 관련 SHA: `4b43a284e637` — `feat(api): 실행 환경과 service bootstrap 구성`

## 6. Invariant evolution

| 단계 | 관련 SHA | 기록 |
| --- | --- | --- |
| Workspace 공통 실행 기준 | `b625c4f9dfdc` | apps/packages glob, recursive command, 공통 TypeScript policy가 도입됩니다. |
| 공유 표현 owner | `7753ad1fafaf` → `573d11acb75e` → `41471c2c2d55` | 독립 shared package에 서비스 DTO와 game serialization contract가 순서대로 들어갑니다. |
| Persistence import와 lifecycle | `f77297697c66` → `1140fb868714` → `9277572765e7` | driver package에서 importable migration boundary, 공통 repository lifecycle로 발전합니다. |
| HTTP composition root | `51484e00a1c2` → `4b43a284e637` | API package가 repository를 선택·초기화하고 app close와 storage close를 연결합니다. |
| Browser package 소비 | `f5c151c7cc7d` | Next.js가 shared source를 transpile하며 browser runtime을 별도 owner로 둡니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 불충분한 상태 | Fix/구현 SHA | Test 또는 후속 증거 |
| --- | --- | --- |
| DB package를 만들었지만 consumer import가 없음 | `1140fb868714` | package export/path alias와 executable migration text를 추가합니다. |
| pool/client lifecycle을 상위가 backend별로 알아야 함 | `9277572765e7` | `AppRepository` factory와 공통 `ensureSeedData`/`close`를 도입합니다. |
| 생성·초기화·listen·close가 분리됨 | `4b43a284e637` | composition root가 repository와 Fastify의 lifecycle을 연결합니다. |

## 8. Ownership·state·responsibility 변화

| Owner | 최종 책임 | 명시적 비책임 |
| --- | --- | --- |
| 루트 workspace | workspace 목록, 공통 compiler policy, recursive verification | 각 package의 내부 runtime state를 소유하지 않습니다. |
| `@pong-pong/shared` | 서비스·game transport representation | DB row, Fastify app, browser state를 소유하지 않습니다. |
| `@pong-pong/db` | driver dependency, pool/Kysely, repository lifecycle | HTTP request와 browser rendering을 소유하지 않습니다. |
| `apps/api` composition root | 환경 읽기, backend 선택, 초기화, app/listener 연결, close 호출 | pool 세부 구현은 repository에 위임합니다. |
| `apps/web` | Next.js build/runtime과 shared source transpilation | DB driver나 server resource를 소유하지 않습니다. |

## 9. Thread 최종 상태

- 최종 SHA 기준으로 root는 공통 개발 명령을 제공하고 shared, DB, API, web은 별도 workspace입니다.
- 공통 contract source와 persistence implementation이 분리되며 API는 두 package를 조립합니다.
- repository가 storage resource를 소유하고 API process entry가 생성·초기화·종료 호출 순서를 소유합니다.
- 초기 bootstrap에는 production fail-closed와 startup 초기화 실패 cleanup 같은 후속 보강이 아직 포함되지 않습니다.

### 실행 검증 기록

- Source inspection: GitHub connector로 Commit map의 exact SHA diff와 필요한 historical file을 조회했습니다.
- Branch validation: branch-local `commit/commit-importance.md`가 선언한 433개 선형 이력에서 모든 참조 SHA와 source classification을 대조했고, 가장 이른 참조 `b625c4f9dfdc`는 branch HEAD 비교에서 merge base가 동일 SHA임을 확인했습니다.
- 실제 실행 시도: `git clone --branch web/ft_transcendence --single-branch https://github.com/seungwoo7050/42-archive.git /tmp/ft-transcendence-audit`
- 결과: exit status 128, `Could not resolve host: github.com`. 따라서 repository checkout, package install, test command는 실행하지 않았으며 runtime 결과를 주장하지 않습니다.

## 10. 최종 실행 흐름

1. 루트 command가 각 workspace command를 실행합니다.
2. API process가 `readEnv`로 설정을 읽습니다.
3. `DATABASE_URL` 유무로 memory/PostgreSQL repository를 만듭니다.
4. `repo.ensureSeedData()`로 초기 schema/data를 준비합니다.
5. `buildApp`에 repository와 origin을 주입하고 listener를 엽니다.
6. Fastify close 또는 listen failure에서 `repo.close()`를 호출합니다.
7. web build는 `@pong-pong/shared` source를 transpile해 browser bundle에서 공통 표현을 사용합니다.

## 11. 학습 완료 확인

- [x] 모든 Commit map SHA의 historical code를 확인했습니다.
- [x] parent/직전 관련 상태와 현재 SHA를 구분했습니다.
- [x] fix의 이전 가정·root cause·corrected invariant를 연결했습니다.
- [x] test가 통과하는 production path와 비증명 범위를 구분했습니다.
- [x] ownership/lifetime/cleanup과 failure path를 기록했습니다.
- [x] 실행 evidence와 code inspection을 구분했습니다.
- [x] 마지막 SHA 기준 최종 flow와 non-guarantee를 설명할 수 있습니다.
===== END FILE: 01-workspace-package-and-composition-boundaries.md =====

===== BEGIN FILE: 02-executable-http-contracts-and-resource-api.md =====
# Resource API와 실행 가능한 HTTP contract

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

먼저 assertion과 route-local 오류로 구현된 Fastify resource API가 baseline integration test를 얻고, 이후 shared Zod contract·typed error boundary·route별 input/output validation으로 retrofit되는 과정을 실제 역사 순서대로 복원합니다.

이 문서는 Phase 1 category audit 후 동결된 scaffold를 기준으로 exact SHA의 구현 발전을 복원합니다.

### 직접 연결되는 불변식

- 초기 API route는 repository operation을 transport에 연결하지만 runtime contract는 후속 단계에서 별도로 도입됩니다.
- shared package는 domain/request/response/error schema를 소유하고 API는 request entry와 response exit에서 이를 실행합니다.
- 예상된 client failure와 예상 밖 internal failure는 중앙 boundary에서 machine code, request ID, public message로 분류됩니다.
- 최종 대표 integration tests는 raw token JSON이 아니라 session cookie와 typed response를 기준으로 합니다.

## 2. 핵심 질문

- 초기 assertion/default 기반 route와 최종 strict typed route의 차이를 같은 resource에서 비교할 수 있습니까?
- domain schema, endpoint wrapper schema, request schema, error envelope는 각각 무엇을 검증합니까?
- public read, authenticated mutation, administrator mutation에서 input parse·authorization·repository call·output parse 순서는 어떻게 다릅니까?
- `parseOutput` 실패는 무엇을 막으며 이미 발생한 side effect까지 rollback하지 못하는 이유는 무엇입니까?
- 초기 integration tests와 `50caaf5c7c49`의 credential/response 기대값 변화가 API migration을 어떻게 보여줍니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, symbol, caller/callee, 상태 mutation, failure branch, cleanup을 actual historical code로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- S/A/B/C 중요도에 맞춰 설명 깊이를 다르게 유지합니다.
- 실행하지 않은 command 결과를 작성하지 않으며 code inspection과 runtime evidence를 구분합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1779df300611` | `feat(api): 로그인과 로비 HTTP 경계 구현` | B | AUTH, PERSISTENCE, WEB | 저장소 기반 개발 로그인, 현재 사용자, 로비, 순위 endpoint를 최초 Fastify route로 노출합니다. |
| 2 | `0bcc487d949f` | `feat(api): 프로필과 친구 리소스 라우트 추가` | B | PERSISTENCE | 공개 profile/dashboard와 사용자별 profile update·friendship mutation route를 추가합니다. |
| 3 | `e8bb6a4bf68b` | `feat(api): 토너먼트와 관리자 라우트 추가` | B | AUTH, PERSISTENCE, TOURNAMENT | 토너먼트 생성·참가와 관리자 사용자 조회·상태 변경 route를 추가합니다. |
| 4 | `fb1c287d9e79` | `test(api): 로그인과 로비 조회 검증` | B | PERSISTENCE, TEST | 초기 로그인→현재 사용자와 leaderboard/lobby read 경로를 Fastify injection으로 검증합니다. |
| 5 | `1395d45a3665` | `test(api): 관리자 사용자 상태 변경 검증` | B | AUTH, PERSISTENCE, TEST | 개발 로그인부터 admin ban route와 memory repository 상태 변경까지 초기 관리자 경로를 검증합니다. |
| 6 | `5088099d1e7d` | `test(api): 토너먼트 생성 흐름 검증` | B | AUTH, PERSISTENCE, TOURNAMENT | 인증 사용자 tournament 생성 후 목록에서 다시 읽는 write→read 경로를 검증합니다. |
| 7 | `0c5c27c8c3df` | `feat(shared): 사용자 HTTP runtime contract 정의` | B | PROTOCOL, TOURNAMENT | compile-time user interface를 실행 가능한 Zod schema와 inferred type으로 교체합니다. |
| 8 | `6704f37ca6a3` | `feat(shared): 경기·대시보드 runtime contract 정의` | B | PROTOCOL | match summary, dashboard, leaderboard payload를 실행 가능한 schema로 정의합니다. |
| 9 | `4bace138f188` | `feat(shared): 친구·채팅·로비 runtime contract 정의` | B | PROTOCOL, REALTIME | friendship, chat, lobby stats/aggregate를 runtime schema로 확장합니다. |
| 10 | `7d0793a23f5d` | `feat(shared): 토너먼트·관리 runtime contract 정의` | B | PROTOCOL, TOURNAMENT, OPERATIONS | 토너먼트 aggregate와 관리자 audit response를 runtime schema로 정의합니다. |
| 11 | `282a9d0beb47` | `feat(shared): HTTP 요청·오류 schema 정의` | B | - | route params/body와 공통 API error envelope를 strict schema로 정의합니다. |
| 12 | `e226b68fe235` | `feat(shared): HTTP 응답 runtime contract 정의` | B | AUTH, PROTOCOL, REALTIME | domain validator를 health/auth/user/lobby/tournament/admin endpoint별 response schema로 조합합니다. |
| 13 | `78cf83f29e80` | `test(shared): HTTP contract 검증` | B | AUTH, PROTOCOL, TEST | HTTP schema의 positive/negative shape와 normalization 규칙을 package unit test로 고정합니다. |
| 14 | `ac85316bb0cb` | `feat(api): typed HTTP 오류 boundary 추가` | A | AUTH, PROTOCOL, RISK | Zod 입력 오류, 예상된 API 오류, not-found, 예상 밖 실패를 하나의 Fastify error boundary로 통합합니다. |
| 15 | `c4cba7d3f871` | `feat(api): 인증·사용자 HTTP contract 적용` | B | AUTH, PROTOCOL, WEB | health/auth/user/profile route에 shared input/output schema와 typed error boundary를 적용합니다. |
| 16 | `05e3ecfa2a2d` | `feat(api): 로비·친구 HTTP contract 적용` | B | AUTH, PROTOCOL, PERSISTENCE | lobby/chat/leaderboard/dashboard/friendship route에 runtime input/output validation과 typed failure를 적용합니다. |
| 17 | `24f99345452d` | `feat(api): 토너먼트·관리 HTTP contract 적용` | B | AUTH, TOURNAMENT | 토너먼트와 관리자 route에 request/response schema 및 공통 authorization/error helper를 적용합니다. |
| 18 | `b2a8de5a0027` | `refactor(api): HTTP boundary helper 통합` | B | AUTH | 남은 route-local unauthorized/suspended helper를 제거하고 공통 typed boundary 함수를 직접 사용합니다. |
| 19 | `50caaf5c7c49` | `test(api): typed HTTP boundary 기대값 정렬` | B | AUTH, TOURNAMENT, WEB | 기존 API·관리·토너먼트 통합 test를 HttpOnly cookie와 typed response/error contract에 맞춥니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(api): 로그인과 로비 HTTP 경계 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `1779df300611` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, WEB |
| Source에서 확정된 역할 | 저장소 기반 개발 로그인, 현재 사용자, 로비, 순위 endpoint를 최초 Fastify route로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`에서 CORS/cookie 등록과 `/auth/dev-login`, `/me`, `/lobby`, `/leaderboard` handler를 확인합니다.
- `request.body` type assertion과 fallback, token을 JSON과 cookie로 동시에 반환하는 초기 contract를 기록합니다.
- `getCurrentUser`가 cookie, bearer header, query token을 어떤 우선순위로 읽고 route-local 오류를 만드는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API package와 bootstrap은 있었지만 외부 요청을 repository operation에 연결하는 HTTP resource가 없었습니다. |
| 해결하려던 문제 | 로그인·현재 사용자·로비 read model을 browser가 사용할 transport 경계가 필요했습니다. |
| 핵심 결정 | Fastify app에 개발 로그인, logout, 사용자 조회, 로비, leaderboard route를 추가하고 repository session을 연결했습니다. |
| 입력 → 상태 전이 → 출력 | 로그인 body를 type assertion으로 읽어 user/session을 만들고 cookie와 JSON token을 반환합니다. 보호 route는 여러 token source를 조회한 뒤 repository에서 사용자를 찾습니다. |
| ownership/lifetime/cleanup | Fastify app이 request/response lifecycle을 소유하고 repository가 session과 data를 소유합니다. app 자체 close는 composition root가 처리합니다. |
| failure/rollback/retry | 입력은 runtime schema를 통과하지 않으며 unauthorized/not-found가 route별 body로 반환됩니다. logout은 이 SHA에서 cookie만 지웁니다. |
| 보장하는 것 | 실제 repository-backed identity와 read model을 HTTP로 호출할 수 있습니다. |
| 보장하지 않는 것 | unknown field, 잘못된 값, output shape, cookie-only credential, 중앙 error envelope는 보장하지 않습니다. |
| 후속 연결 | `0bcc487d949f`와 `e8bb6a4bf68b`가 resource를 확장하고 후속 contract retrofit의 비교 기준이 됩니다. |

#### 최소 코드 근거

`1779df300611`, `apps/api/src/app.ts`의 `buildApp`, `/auth/dev-login`, `getCurrentUser`를 확인했습니다.

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `0bcc487d949f` — `feat(api): 프로필과 친구 리소스 라우트 추가`

### 5.2. `feat(api): 프로필과 친구 리소스 라우트 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `0bcc487d949f` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | 공개 profile/dashboard와 사용자별 profile update·friendship mutation route를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 `/users/:id`, `/dashboard`, `/profile/:handle`, `/profile/me`, `/friends` 계열을 확인합니다.
- 공개 read와 현재 사용자 mutation이 어떤 session 검사를 거치는지 비교합니다.
- params/body type assertion과 route-local 404·401 응답이 남아 있는지 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 API는 로그인·로비에 한정되어 profile과 social repository operation을 transport로 노출하지 않았습니다. |
| 해결하려던 문제 | 공개 프로필 조회와 현재 사용자 변경, friend request/accept를 서로 다른 권한 수준으로 연결해야 했습니다. |
| 핵심 결정 | 공개 lookup, authenticated dashboard/update, friendship list/request/accept route를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | handler가 params/body를 assertion으로 읽고 session user를 확인한 뒤 해당 repository method를 호출해 JSON을 반환합니다. |
| ownership/lifetime/cleanup | repository가 profile/friend state를 소유하고 Fastify handler는 인증된 caller identity를 operation에 전달합니다. |
| failure/rollback/retry | 잘못된 UUID나 body shape는 handler 이전에 거부되지 않으며 route별 error body가 다를 수 있습니다. |
| 보장하는 것 | 공개 read와 identity-bound mutation의 resource 구분을 API 표면에 만듭니다. |
| 보장하지 않는 것 | 실행 가능한 input/output contract와 중앙 authorization helper는 아직 없습니다. |
| 후속 연결 | `e8bb6a4bf68b`가 고권한 resource를 추가하고 later Zod commits가 같은 route를 retrofit합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 profile/friends handler와 repository 호출 위치를 확인했습니다.

비교 기준:
- 직전 관련 SHA: `1779df300611` — `feat(api): 로그인과 로비 HTTP 경계 구현`
- 다음 관련 SHA: `e8bb6a4bf68b` — `feat(api): 토너먼트와 관리자 라우트 추가`

### 5.3. `feat(api): 토너먼트와 관리자 라우트 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e8bb6a4bf68b` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TOURNAMENT |
| Source에서 확정된 역할 | 토너먼트 생성·참가와 관리자 사용자 조회·상태 변경 route를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`의 tournament GET/POST/join과 admin list/ban/status handler를 확인합니다.
- 인증과 admin role 검사 순서, malformed name/reason에 적용되는 초기 default를 확인합니다.
- route-local authorization/error response가 다른 resource와 중복되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | profile/social route는 있었지만 tournament와 administrator operation은 HTTP에서 호출할 수 없었습니다. |
| 해결하려던 문제 | 일반 authenticated mutation과 관리자 전용 mutation을 같은 app 안에서 구분해야 했습니다. |
| 핵심 결정 | 토너먼트 목록·생성·참가와 관리자 사용자 목록·상태 변경 route를 추가하고 role check를 배치했습니다. |
| 입력 → 상태 전이 → 출력 | handler가 현재 사용자를 조회하고 role/status를 검사한 뒤 repository에 tournament 또는 admin mutation을 요청합니다. |
| ownership/lifetime/cleanup | repository가 tournament/user state를 소유하며 route가 actor identity와 target ID를 전달합니다. |
| failure/rollback/retry | body/params는 assertion과 default에 의존해 malformed input이 의도치 않은 값으로 바뀔 수 있고 error envelope가 통일되지 않습니다. |
| 보장하는 것 | 고권한 resource가 명시적 route와 repository operation을 통해 접근됩니다. |
| 보장하지 않는 것 | handle 기반 초기 admin seed의 안전성, strict input, audit atomicity는 보장하지 않습니다. |
| 후속 연결 | `1395d45a3665`와 `5088099d1e7d`가 초기 경로를 고정하고 후속 contract commit이 validation을 적용합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 tournament/admin route와 role 검사 코드를 확인했습니다.

비교 기준:
- 직전 관련 SHA: `0bcc487d949f` — `feat(api): 프로필과 친구 리소스 라우트 추가`
- 다음 관련 SHA: `fb1c287d9e79` — `test(api): 로그인과 로비 조회 검증`

### 5.4. `test(api): 로그인과 로비 조회 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `fb1c287d9e79` |
| Importance | B |
| Tags | PERSISTENCE, TEST |
| Source에서 확정된 역할 | 초기 로그인→현재 사용자와 leaderboard/lobby read 경로를 Fastify injection으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.test.ts`의 memory repository setup, `app.ready`, `app.close`, `repo.close` 순서를 확인합니다.
- 로그인 response token을 bearer header로 `/me`에 재사용하는 당시 contract를 확인합니다.
- leaderboard와 lobby 검사가 status와 최소 데이터 존재만 증명하는 범위를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 route는 구현됐지만 request부터 repository까지 연결되는 자동 검증이 없었습니다. |
| 해결하려던 문제 | route helper가 아니라 실제 Fastify registration, session, memory state가 함께 동작하는 기준이 필요했습니다. |
| 핵심 결정 | 각 test마다 memory repository와 app을 만들고 Fastify injection으로 로그인·현재 사용자·read endpoint를 호출했습니다. |
| 입력 → 상태 전이 → 출력 | `POST /auth/dev-login`이 token을 반환하고 `GET /me`가 이를 bearer로 해석해 같은 handle을 반환합니다. read test는 leaderboard/lobby를 호출합니다. |
| ownership/lifetime/cleanup | test fixture가 app/repository lifetime을 소유하고 `afterEach`에서 둘을 닫습니다. |
| failure/rollback/retry | 실패 injection은 없으며 assertion 실패 시 test가 종료됩니다. 실제 network, cookie agent, PostgreSQL은 사용하지 않습니다. |
| 보장하는 것 | 초기 bearer/token 기반 API의 happy path와 read resource registration을 고정합니다. |
| 보장하지 않는 것 | 입력 rejection, 권한 negative path, output schema, real DB consistency는 증명하지 않습니다. |
| 후속 연결 | 후속 `50caaf5c7c49`가 같은 test를 cookie 기반 typed boundary에 맞춥니다. |

#### 최소 코드 근거

`apps/api/src/app.test.ts`의 `app.inject`와 memory repository fixture가 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 로그인 session이 현재 사용자 조회에 연결되고 lobby/leaderboard route가 등록되어야 합니다. |
| 재현하는 failure/boundary | 개발 로그인 후 bearer token으로 `/me`를 호출하고 두 read endpoint가 200과 데이터를 반환하는 경계입니다. |
| test technique | in-process Fastify integration test + memory repository |
| 통과하는 production path | `buildApp` → Fastify route → repository session/read operation → JSON response |
| 증명하는 것 | 당시 초기 API의 대표 happy path와 fixture cleanup을 증명합니다. |
| 증명하지 않는 것 | 실제 socket, browser cookie behavior, PostgreSQL, malformed input을 증명하지 않습니다. |
| 후속 회귀 방지 | 후속 route/contract refactor가 기본 로그인·read 경로를 깨뜨리지 않도록 합니다. |

비교 기준:
- 직전 관련 SHA: `e8bb6a4bf68b` — `feat(api): 토너먼트와 관리자 라우트 추가`
- 다음 관련 SHA: `1395d45a3665` — `test(api): 관리자 사용자 상태 변경 검증`

### 5.5. `test(api): 관리자 사용자 상태 변경 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1395d45a3665` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TEST |
| Source에서 확정된 역할 | 개발 로그인부터 admin ban route와 memory repository 상태 변경까지 초기 관리자 경로를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/admin.test.ts`에서 admin/target login, token 추출, ban request를 확인합니다.
- 관리자 권한이 당시 seed handle에 의존하는지와 target status 결과를 확인합니다.
- 권한 거부·audit·동시성은 이 test가 다루지 않는다는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 관리자 route는 있었지만 실제 로그인과 repository mutation을 통과하는 test가 없었습니다. |
| 해결하려던 문제 | route 등록·actor session·target ID·status update가 연결되지 않으면 고권한 기능이 조용히 깨질 수 있었습니다. |
| 핵심 결정 | admin과 target을 dev-login으로 만든 뒤 bearer token으로 ban route를 호출하고 반환 user status를 검사했습니다. |
| 입력 → 상태 전이 → 출력 | 두 로그인 → admin token/target ID 추출 → `POST /admin/users/:id/ban` → memory repository update → banned response 순입니다. |
| ownership/lifetime/cleanup | fixture가 app/repo를 소유·정리하고 repository가 target 상태를 소유합니다. |
| failure/rollback/retry | 권한이 없는 caller와 실패 rollback은 주입하지 않습니다. 당시 admin handle seed 가정을 그대로 사용합니다. |
| 보장하는 것 | 초기 관리자 상태 변경의 end-to-end-in-process happy path를 증명합니다. |
| 보장하지 않는 것 | 명시적 role assignment, audit atomicity, PostgreSQL row effect는 증명하지 않습니다. |
| 후속 연결 | 후속 auth category가 handle privilege를 제거하며 `50caaf5c7c49`가 test fixture를 명시적 seed/cookie 방식으로 갱신합니다. |

#### 최소 코드 근거

`apps/api/src/admin.test.ts`의 두 로그인과 ban injection을 확인했습니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 인증된 관리자가 대상 계정 상태를 repository를 통해 변경할 수 있어야 합니다. |
| 재현하는 failure/boundary | admin login, target login, bearer authorization, ban mutation, returned status의 연결입니다. |
| test technique | in-process API integration test + memory repository |
| 통과하는 production path | Fastify login routes → session lookup → admin route → repository update → response |
| 증명하는 것 | 당시 admin happy path가 route와 repository를 관통함을 증명합니다. |
| 증명하지 않는 것 | 비관리자 거부, audit record, transaction, real PostgreSQL은 증명하지 않습니다. |
| 후속 회귀 방지 | 관리자 route나 repository signature 변경 시 대표 mutation 회귀를 감지합니다. |

비교 기준:
- 직전 관련 SHA: `fb1c287d9e79` — `test(api): 로그인과 로비 조회 검증`
- 다음 관련 SHA: `5088099d1e7d` — `test(api): 토너먼트 생성 흐름 검증`

### 5.6. `test(api): 토너먼트 생성 흐름 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `5088099d1e7d` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, TOURNAMENT |
| Source에서 확정된 역할 | 인증 사용자 tournament 생성 후 목록에서 다시 읽는 write→read 경로를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/tournament.test.ts`의 login fixture와 token lifetime을 확인합니다.
- 생성 request가 repository state를 바꾸고 바로 뒤 list request에서 같은 이름을 관찰하는지 확인합니다.
- capacity, duplicate join, bracket transition은 이 test 범위 밖임을 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 토너먼트 route는 구현됐지만 생성 결과가 repository list projection에 반영되는 자동 증거가 없었습니다. |
| 해결하려던 문제 | write response만 성공하고 이후 read에서 사라지는 연결 오류를 탐지할 기준이 필요했습니다. |
| 핵심 결정 | fixture 로그인으로 token을 얻고 tournament 생성 후 공개 list를 호출해 첫 항목 이름을 검사했습니다. |
| 입력 → 상태 전이 → 출력 | login → authenticated create → memory repository mutation → public list → 동일 이름 관찰 순입니다. |
| ownership/lifetime/cleanup | memory repository가 tournament lifetime을 소유하고 fixture가 app/repo를 닫습니다. |
| failure/rollback/retry | 동시 join, persistence durability, rollback failure는 주입하지 않습니다. |
| 보장하는 것 | 초기 tournament write-to-read HTTP contract를 고정합니다. |
| 보장하지 않는 것 | PostgreSQL, transaction, bracket creation, authorization negative path는 증명하지 않습니다. |
| 후속 연결 | 후속 tournament schema/API contract 적용과 `50caaf5c7c49`의 cookie fixture 정렬로 이어집니다. |

#### 최소 코드 근거

`apps/api/src/tournament.test.ts`의 create/list 연속 injection이 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 인증된 생성 mutation이 이후 tournament list projection에서 관찰되어야 합니다. |
| 재현하는 failure/boundary | create 성공 후 같은 app/repository에서 list가 생성 이름을 반환하는 경계입니다. |
| test technique | in-process write-to-read integration test + memory repository |
| 통과하는 production path | login → create route → repository create → list route → repository list |
| 증명하는 것 | 초기 route와 memory backend의 observable write/read 연결을 증명합니다. |
| 증명하지 않는 것 | 실제 DB durability, concurrent capacity, bracket 상태 전이는 증명하지 않습니다. |
| 후속 회귀 방지 | API 또는 repository refactor가 생성 데이터를 list에서 잃는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `1395d45a3665` — `test(api): 관리자 사용자 상태 변경 검증`
- 다음 관련 SHA: `0c5c27c8c3df` — `feat(shared): 사용자 HTTP runtime contract 정의`

### 5.7. `feat(shared): 사용자 HTTP runtime contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `0c5c27c8c3df` |
| Importance | B |
| Tags | PROTOCOL, TOURNAMENT |
| Source에서 확정된 역할 | compile-time user interface를 실행 가능한 Zod schema와 inferred type으로 교체합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`에서 user role/status, UUID, integer 통계, 공개/세션 schema를 확인합니다.
- `z.infer`가 기존 interface 대신 type source가 되는지 확인합니다.
- unknown field 정책과 nested object strictness가 이 SHA에서 어느 수준인지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 API는 TypeScript interface와 assertion만 사용해 runtime JSON을 거부하지 못했습니다. |
| 해결하려던 문제 | 외부·repository 값이 선언된 user shape와 달라도 handler가 그대로 전달할 수 있었습니다. |
| 핵심 결정 | user field를 Zod schema로 정의하고 공개/세션 schema 차이를 유지한 채 type을 schema에서 추론했습니다. |
| 입력 → 상태 전이 → 출력 | producer 또는 consumer가 schema `parse`/`safeParse`를 호출하면 UUID·enum·integer·nullable 조건이 runtime에서 검사됩니다. |
| ownership/lifetime/cleanup | shared package가 user contract의 runtime·compile-time source를 함께 소유합니다. |
| failure/rollback/retry | route가 아직 schema를 호출하지 않으면 보장은 적용되지 않습니다. |
| 보장하는 것 | user shape에 실행 가능한 검증 규칙을 부여하고 type/schema drift를 줄입니다. |
| 보장하지 않는 것 | 다른 domain과 endpoint별 request/response, 중앙 error handling은 아직 없습니다. |
| 후속 연결 | `6704f37ca6a3`부터 나머지 domain을 확장하고 `c4cba7d3f871`이 route에 적용합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 Zod user schema와 `z.infer` type alias가 근거입니다.

비교 기준:
- 직전 관련 SHA: `5088099d1e7d` — `test(api): 토너먼트 생성 흐름 검증`
- 다음 관련 SHA: `6704f37ca6a3` — `feat(shared): 경기·대시보드 runtime contract 정의`

### 5.8. `feat(shared): 경기·대시보드 runtime contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `6704f37ca6a3` |
| Importance | B |
| Tags | PROTOCOL |
| Source에서 확정된 역할 | match summary, dashboard, leaderboard payload를 실행 가능한 schema로 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`의 match result/mode/date, dashboard stats, leaderboard rank/rate 범위를 확인합니다.
- zero game과 percentage 범위 같은 derived value가 schema에 어떻게 제한되는지 확인합니다.
- nested user schema 재사용을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | user만 runtime schema였고 match/dashboard/leaderboard는 compile-time shape에 머물렀습니다. |
| 해결하려던 문제 | 복합 read model의 숫자·enum·날짜 값이 잘못돼도 transport 경계에서 탐지되지 않았습니다. |
| 핵심 결정 | 기존 user schema를 조합해 match, dashboard, leaderboard의 runtime constraint를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | aggregate output을 schema에 전달하면 nested user와 통계 범위를 재귀적으로 검사합니다. |
| ownership/lifetime/cleanup | shared가 aggregate shape를 소유하고 repository/API는 값을 생산합니다. |
| failure/rollback/retry | 실제 winRate 계산 정확성이나 repository query는 검사하지 않습니다. |
| 보장하는 것 | 복합 read response의 허용 값 범위를 실행 가능하게 표현합니다. |
| 보장하지 않는 것 | route output parse를 호출하기 전에는 잘못된 producer 결과가 여전히 통과할 수 있습니다. |
| 후속 연결 | `e226b68fe235`가 endpoint response schema를 조합하고 API apply commits가 이를 사용합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 match/dashboard/leaderboard schema를 확인했습니다.

비교 기준:
- 직전 관련 SHA: `0c5c27c8c3df` — `feat(shared): 사용자 HTTP runtime contract 정의`
- 다음 관련 SHA: `4bace138f188` — `feat(shared): 친구·채팅·로비 runtime contract 정의`

### 5.9. `feat(shared): 친구·채팅·로비 runtime contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `4bace138f188` |
| Importance | B |
| Tags | PROTOCOL, REALTIME |
| Source에서 확정된 역할 | friendship, chat, lobby stats/aggregate를 runtime schema로 확장합니다. |

#### 해당 SHA에서 확인할 실제 코드

- friend status와 chat scope/body/date, lobby array와 nullable current user를 확인합니다.
- 중첩 public user schema가 재사용되는지 확인합니다.
- chat body의 transport response shape와 later request body schema를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | user와 일부 read model만 runtime 검증됐고 social/lobby aggregate는 interface에 머물렀습니다. |
| 해결하려던 문제 | 친구 상태·채팅 범위·로비 통계가 잘못된 값이어도 API response에서 탐지할 방법이 없었습니다. |
| 핵심 결정 | 기존 user schema를 중첩해 friend, chat, lobby response 구성요소를 Zod로 정의했습니다. |
| 입력 → 상태 전이 → 출력 | aggregate parse 시 배열 원소와 nested sender/user, scope, stats가 함께 검사됩니다. |
| ownership/lifetime/cleanup | shared가 response shape를 소유하고 repository/API가 social/lobby data를 생산합니다. |
| failure/rollback/retry | 메시지 저장 권한이나 room scope authorization은 이 schema가 보장하지 않습니다. |
| 보장하는 것 | social·lobby output을 runtime에서 검증할 수 있습니다. |
| 보장하지 않는 것 | endpoint schema와 route 적용, business invariant는 아직 보장하지 않습니다. |
| 후속 연결 | `e226b68fe235`와 `05e3ecfa2a2d`가 endpoint response 및 handler에 연결합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 friend/chat/lobby schema를 확인했습니다.

비교 기준:
- 직전 관련 SHA: `6704f37ca6a3` — `feat(shared): 경기·대시보드 runtime contract 정의`
- 다음 관련 SHA: `7d0793a23f5d` — `feat(shared): 토너먼트·관리 runtime contract 정의`

### 5.10. `feat(shared): 토너먼트·관리 runtime contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `7d0793a23f5d` |
| Importance | B |
| Tags | PROTOCOL, TOURNAMENT, OPERATIONS |
| Source에서 확정된 역할 | 토너먼트 aggregate와 관리자 audit response를 runtime schema로 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 토너먼트 status, participant, bracket nullable field, admin action schema를 확인합니다.
- creator/winner/user nested schema 재사용과 날짜 표현을 확인합니다.
- nullable room/match/score가 lifecycle 단계를 어떻게 표현하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 tournament/admin route가 반환하는 복합 shape는 runtime schema가 없었습니다. |
| 해결하려던 문제 | bracket lifecycle의 nullable field나 audit actor/target shape가 drift해도 handler가 탐지하지 못했습니다. |
| 핵심 결정 | 토너먼트 summary/bracket과 admin action을 Zod schema로 만들고 shared user/date schema를 조합했습니다. |
| 입력 → 상태 전이 → 출력 | producer output을 parse하면 lifecycle status와 nested participant/audit record shape가 검사됩니다. |
| ownership/lifetime/cleanup | shared가 공개 aggregate의 표현을 소유하며 repository가 실제 상태 전이를 소유합니다. |
| failure/rollback/retry | 중복 bracket, authorization, transaction atomicity는 schema만으로 보장하지 않습니다. |
| 보장하는 것 | 토너먼트·관리 응답의 직렬화 shape를 실행 가능하게 고정합니다. |
| 보장하지 않는 것 | route가 parseOutput을 쓰기 전에는 적용되지 않으며 상태 전이의 합법성은 다루지 않습니다. |
| 후속 연결 | `24f99345452d`가 tournament/admin route에 적용합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 tournament/admin schema가 근거입니다.

비교 기준:
- 직전 관련 SHA: `4bace138f188` — `feat(shared): 친구·채팅·로비 runtime contract 정의`
- 다음 관련 SHA: `282a9d0beb47` — `feat(shared): HTTP 요청·오류 schema 정의`

### 5.11. `feat(shared): HTTP 요청·오류 schema 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `282a9d0beb47` |
| Importance | B |
| Tags | - |
| Source에서 확정된 역할 | route params/body와 공통 API error envelope를 strict schema로 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `emptyParamsSchema`, UUID/handle params와 각 body schema의 `.strict()`를 확인합니다.
- profile update refine와 문자열 trim/length 조건을 확인합니다.
- `apiErrorBodySchema`의 code, message, requestId, optional fieldErrors shape를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | domain output schema는 있었지만 endpoint input과 error body는 route-local assertion·임의 JSON에 의존했습니다. |
| 해결하려던 문제 | unknown field, 잘못된 path parameter, 빈 update를 일관되게 거부하고 caller가 처리 가능한 오류 shape가 필요했습니다. |
| 핵심 결정 | params/body를 strict Zod object로 만들고 공통 error envelope에 machine code, request ID, field errors를 정의했습니다. |
| 입력 → 상태 전이 → 출력 | handler나 boundary가 schema를 호출하면 normalized input을 반환하거나 Zod issue를 error response 재료로 바꿀 수 있습니다. |
| ownership/lifetime/cleanup | shared가 wire-level request/error shape를 소유합니다. Fastify boundary는 아직 이를 실제 response로 변환해야 합니다. |
| failure/rollback/retry | 각 route의 params/query/body 조합을 하나의 map으로 묶지 않았고 일부 bodyless route가 검증을 호출하지 않을 수 있습니다. |
| 보장하는 것 | input과 error envelope를 runtime에서 검증할 수 있는 공통 정의를 제공합니다. |
| 보장하지 않는 것 | 모든 route가 검증된다는 보장과 내부 error redaction은 아직 없습니다. |
| 후속 연결 | `e226b68fe235`가 response schema를, `ac85316bb0cb`가 typed boundary를 추가합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 strict input schema와 `apiErrorBodySchema`를 확인했습니다.

비교 기준:
- 직전 관련 SHA: `7d0793a23f5d` — `feat(shared): 토너먼트·관리 runtime contract 정의`
- 다음 관련 SHA: `e226b68fe235` — `feat(shared): HTTP 응답 runtime contract 정의`

### 5.12. `feat(shared): HTTP 응답 runtime contract 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `e226b68fe235` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | domain validator를 health/auth/user/lobby/tournament/admin endpoint별 response schema로 조합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- endpoint response schema가 기존 domain schema를 어떤 객체 키로 감싸는지 확인합니다.
- health service literal과 WS ticket format/version/expiry 조건을 확인합니다.
- 각 exported inferred response type이 schema에서 파생되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | domain component schema는 있었지만 handler별 top-level JSON shape는 명시적 executable contract가 아니었습니다. |
| 해결하려던 문제 | 같은 user나 tournament라도 endpoint wrapper key가 달라지면 client와 server가 drift할 수 있었습니다. |
| 핵심 결정 | health, auth, user, social, tournament, admin response를 domain schema 조합으로 정의했습니다. |
| 입력 → 상태 전이 → 출력 | handler result를 endpoint schema에 넣으면 top-level key와 nested aggregate가 함께 검증됩니다. |
| ownership/lifetime/cleanup | shared가 endpoint output contract를 소유하고 API handler가 이를 호출할 책임을 갖습니다. |
| failure/rollback/retry | 실제 HTTP status나 header, repository side effect는 schema가 검사하지 않습니다. |
| 보장하는 것 | endpoint별 response JSON을 runtime에서 fail-closed 검사할 수 있습니다. |
| 보장하지 않는 것 | API가 `parseOutput`을 적용하기 전에는 producer 오류를 막지 못합니다. |
| 후속 연결 | `78cf83f29e80`가 schema 자체를 검증하고 `c4cba7d3f871` 이후 route가 소비합니다. |

#### 최소 코드 근거

`packages/shared/src/http.ts`의 endpoint response schema 묶음이 근거입니다.

비교 기준:
- 직전 관련 SHA: `282a9d0beb47` — `feat(shared): HTTP 요청·오류 schema 정의`
- 다음 관련 SHA: `78cf83f29e80` — `test(shared): HTTP contract 검증`

### 5.13. `test(shared): HTTP contract 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `78cf83f29e80` |
| Importance | B |
| Tags | AUTH, PROTOCOL, TEST |
| Source에서 확정된 역할 | HTTP schema의 positive/negative shape와 normalization 규칙을 package unit test로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.test.ts`의 valid session user, unknown login field, invalid handle, trim, UUID, profile refine, error envelope, WS ticket case를 확인합니다.
- 각 assertion이 schema parse 자체를 검사하며 Fastify route를 통과하지 않는다는 점을 기록합니다.
- accepted value와 rejected value 경계가 어느 commit의 schema를 보호하는지 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 실행 가능한 schema는 추가됐지만 실제 negative example이 지속적으로 거부된다는 자동 증거가 없었습니다. |
| 해결하려던 문제 | refactor 중 `.strict()`, trim, UUID, refine, ticket rule이 완화될 수 있었습니다. |
| 핵심 결정 | 대표 valid value와 unknown/invalid value를 table·focused assertion으로 parse해 성공/실패를 검증했습니다. |
| 입력 → 상태 전이 → 출력 | fixture → shared schema `parse`/`safeParse` → normalized output 또는 Zod failure를 직접 관찰합니다. |
| ownership/lifetime/cleanup | 각 test가 자체 value를 소유하고 별도 app/repository resource는 없습니다. |
| failure/rollback/retry | production route와 network failure는 통과하지 않으며 schema implementation만 검사합니다. |
| 보장하는 것 | 공유 HTTP contract의 주요 behavioral rule을 빠르고 결정적으로 고정합니다. |
| 보장하지 않는 것 | route가 올바른 schema를 호출하는지, status/error handler, real JSON serialization은 증명하지 않습니다. |
| 후속 연결 | `ac85316bb0cb`과 세 적용 commit이 schema를 API boundary에 연결합니다. |

#### 최소 코드 근거

`packages/shared/src/http.test.ts`의 negative case가 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | shared HTTP schema가 허용 shape를 normalize하고 unknown·invalid 값을 거부해야 합니다. |
| 재현하는 failure/boundary | unknown login field, invalid handle/UUID, 빈 profile update, malformed error/ticket 값입니다. |
| test technique | pure schema unit/boundary test |
| 통과하는 production path | fixture → Zod schema parse/safeParse |
| 증명하는 것 | schema 자체의 constraint와 normalization을 결정적으로 증명합니다. |
| 증명하지 않는 것 | Fastify route selection, HTTP status/header, repository behavior는 증명하지 않습니다. |
| 후속 회귀 방지 | contract rule이 느슨해지는 회귀를 조기에 감지합니다. |

비교 기준:
- 직전 관련 SHA: `e226b68fe235` — `feat(shared): HTTP 응답 runtime contract 정의`
- 다음 관련 SHA: `ac85316bb0cb` — `feat(api): typed HTTP 오류 boundary 추가`

### 5.14. `feat(api): typed HTTP 오류 boundary 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `ac85316bb0cb` |
| Importance | A |
| Tags | AUTH, PROTOCOL, RISK |
| Source에서 확정된 역할 | Zod 입력 오류, 예상된 API 오류, not-found, 예상 밖 실패를 하나의 Fastify error boundary로 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/httpBoundary.ts`의 `ApiHttpError`, `parseInput`, `parseOutput`, `sendApiError`, `installHttpErrorBoundary`를 확인합니다.
- Zod issue path가 `fieldErrors`로 바뀌고 `request.id`가 response에 들어가는 경로를 추적합니다.
- expected error와 unexpected error가 log/public message에서 분리되고 output validation failure가 500으로 fail-closed 되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | route마다 status/body를 직접 반환하고 입력 assertion·output 무검증이 섞여 있었습니다. |
| 해결하려던 문제 | validation·authorization·not-found·internal failure가 서로 다른 JSON으로 노출되고 내부 예외가 그대로 전달될 위험이 있었습니다. |
| 핵심 결정 | `ApiHttpError`와 공통 parser/sender를 도입하고 Fastify not-found/error handler를 한 번 설치하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | 입력은 `safeParse` 후 issue path를 모아 `validation_failed`로 던지고, 예상 오류는 typed status/code로 응답합니다. 예상 밖 오류는 server log에 남기고 generic 500을 보냅니다. |
| ownership/lifetime/cleanup | HTTP boundary module이 public error envelope와 response validation을 소유합니다. route는 domain-specific 조건에서 typed error를 던집니다. |
| failure/rollback/retry | 자동 retry/rollback은 없으며 repository mutation 뒤 output parse가 실패하면 이미 수행된 side effect를 되돌리지 않습니다. |
| 보장하는 것 | 모든 boundary가 동일한 code/message/requestId/fieldErrors shape를 만들고 내부 error를 generic 500으로 축소할 수 있습니다. |
| 보장하지 않는 것 | 이 SHA만으로 기존 모든 route가 helper를 사용하지는 않으며 strict query/body 전체 coverage도 아직 없습니다. |
| 후속 연결 | `c4cba7d3f871`, `05e3ecfa2a2d`, `24f99345452d`가 route별로 적용합니다. |

#### 최소 코드 근거

`ac85316bb0cb`, `apps/api/src/httpBoundary.ts`:

```ts
if (error instanceof ApiHttpError) {
  sendApiError(reply, request, error.statusCode, error.code,
    error.message, error.fieldErrors);
  return;
}
request.log.error({ err: error }, "request failed");
sendApiError(reply, request, 500, "internal_error",
  "요청을 처리하지 못했습니다.");
```

비교 기준:
- 직전 관련 SHA: `78cf83f29e80` — `test(shared): HTTP contract 검증`
- 다음 관련 SHA: `c4cba7d3f871` — `feat(api): 인증·사용자 HTTP contract 적용`

### 5.15. `feat(api): 인증·사용자 HTTP contract 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `c4cba7d3f871` |
| Importance | B |
| Tags | AUTH, PROTOCOL, WEB |
| Source에서 확정된 역할 | health/auth/user/profile route에 shared input/output schema와 typed error boundary를 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts`에서 boundary 설치 위치와 health/auth/user/profile의 `parseInput`/`parseOutput` 호출을 확인합니다.
- 개발 로그인 JSON에서 raw token이 제거되고 HttpOnly cookie만 남는 변경을 확인합니다.
- CORS에 `x-request-id`가 추가되고 authentication/not-found가 typed helper로 전환되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공통 boundary는 존재했지만 초기 route는 assertion, token response, route-local error를 계속 사용했습니다. |
| 해결하려던 문제 | 핵심 identity route가 shared schema를 호출하지 않으면 contract와 실제 API가 분리된 상태였습니다. |
| 핵심 결정 | boundary를 app에 설치하고 health/auth/user/profile의 input/output을 shared schema로 parse했습니다. 로그인 token을 JSON에서 제거했습니다. |
| 입력 → 상태 전이 → 출력 | request input parse → repository operation → output parse 순으로 실행하고 예상 failure는 typed helper가 중앙 handler로 전달합니다. |
| ownership/lifetime/cleanup | cookie/session은 repository와 Fastify cookie가 소유하고 raw token은 response body에서 제거됩니다. boundary가 public failure shape를 소유합니다. |
| failure/rollback/retry | repository side effect 후 output parse failure에 rollback은 없으며 아직 모든 route가 적용된 것은 아닙니다. |
| 보장하는 것 | identity/user route에서 malformed input과 malformed output을 fail-closed 처리하고 일관된 error envelope를 사용합니다. |
| 보장하지 않는 것 | lobby/friend/tournament/admin route와 모든 query/body strictness는 아직 완전하지 않습니다. |
| 후속 연결 | `05e3ecfa2a2d`와 `24f99345452d`가 나머지 resource에 적용합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 boundary 설치, login response, profile route parse 호출을 확인했습니다.

비교 기준:
- 직전 관련 SHA: `ac85316bb0cb` — `feat(api): typed HTTP 오류 boundary 추가`
- 다음 관련 SHA: `05e3ecfa2a2d` — `feat(api): 로비·친구 HTTP contract 적용`

### 5.16. `feat(api): 로비·친구 HTTP contract 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `05e3ecfa2a2d` |
| Importance | B |
| Tags | AUTH, PROTOCOL, PERSISTENCE |
| Source에서 확정된 역할 | lobby/chat/leaderboard/dashboard/friendship route에 runtime input/output validation과 typed failure를 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 해당 route의 `parseInput`, `parseOutput`, authentication/suspension helper 호출 순서를 확인합니다.
- 두 friendship request alias가 동일 handler를 공유하는지 확인합니다.
- chat text/handle/UUID가 repository 호출 전에 normalized·validated 되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | identity route는 typed boundary를 사용했지만 social/lobby route는 assertion과 route-local error가 남아 있었습니다. |
| 해결하려던 문제 | 동일 API 안에서 resource마다 입력·오류 보장이 달라지는 상태를 제거해야 했습니다. |
| 핵심 결정 | lobby/social handler에 shared request/response schema와 typed auth/suspension error를 적용하고 request alias를 한 handler로 통합했습니다. |
| 입력 → 상태 전이 → 출력 | 입력 parse → caller status 확인 → repository read/write → aggregate output parse 순으로 동작합니다. |
| ownership/lifetime/cleanup | shared가 wire contract, route가 authorization order, repository가 state를 소유합니다. |
| failure/rollback/retry | 각 repository mutation의 transaction/rollback은 이 commit이 바꾸지 않으며 unknown query를 모든 route에서 검사하지는 않습니다. |
| 보장하는 것 | social/lobby resource가 같은 runtime contract와 error envelope를 사용합니다. |
| 보장하지 않는 것 | 토너먼트/관리 route와 route별 params/query/body 전체 map은 아직 남습니다. |
| 후속 연결 | `24f99345452d`와 later strict-request Thread가 coverage를 완성합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 lobby/chat/friend handler와 shared schema 호출이 근거입니다.

비교 기준:
- 직전 관련 SHA: `c4cba7d3f871` — `feat(api): 인증·사용자 HTTP contract 적용`
- 다음 관련 SHA: `24f99345452d` — `feat(api): 토너먼트·관리 HTTP contract 적용`

### 5.17. `feat(api): 토너먼트·관리 HTTP contract 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `24f99345452d` |
| Importance | B |
| Tags | AUTH, TOURNAMENT |
| Source에서 확정된 역할 | 토너먼트와 관리자 route에 request/response schema 및 공통 authorization/error helper를 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- create/join/admin params·body parse와 response parse 위치를 확인합니다.
- 잘못된 tournament name에 default를 만들던 이전 경로가 제거되는지 확인합니다.
- `requireAdmin`이 unauthenticated와 non-admin을 구분해 typed error를 던지는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 나머지 resource가 typed boundary를 사용해도 고권한 tournament/admin route는 여전히 assertion과 default를 사용했습니다. |
| 해결하려던 문제 | 가장 높은 권한의 mutation이 malformed input을 보정하거나 임의 error payload를 반환하면 전체 API contract가 불완전했습니다. |
| 핵심 결정 | UUID/body schema와 output schema를 적용하고 admin check를 공통 helper로 모았습니다. |
| 입력 → 상태 전이 → 출력 | input parse → authentication → admin/active check → repository mutation → output parse 순으로 실행합니다. |
| ownership/lifetime/cleanup | route가 권한 검사 순서를 소유하고 repository가 mutation을 소유하며 boundary가 public error를 소유합니다. |
| failure/rollback/retry | repository operation의 atomicity나 role source는 이 commit이 변경하지 않습니다. |
| 보장하는 것 | 초기 JSON resource 전체가 shared runtime contract와 typed boundary를 사용할 수 있게 됩니다. |
| 보장하지 않는 것 | bodyless route의 unknown query/body를 모두 검사한다는 보장은 아직 없고 helper 중복이 남습니다. |
| 후속 연결 | `b2a8de5a0027`이 helper를 통합하고 strict-request Thread가 route map을 추가합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 tournament/admin parse와 `requireAdmin`이 근거입니다.

비교 기준:
- 직전 관련 SHA: `05e3ecfa2a2d` — `feat(api): 로비·친구 HTTP contract 적용`
- 다음 관련 SHA: `b2a8de5a0027` — `refactor(api): HTTP boundary helper 통합`

### 5.18. `refactor(api): HTTP boundary helper 통합`

| 항목 | 값 |
| --- | --- |
| SHA | `b2a8de5a0027` |
| Importance | B |
| Tags | AUTH |
| Source에서 확정된 역할 | 남은 route-local unauthorized/suspended helper를 제거하고 공통 typed boundary 함수를 직접 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.ts` import와 helper 삭제 diff를 확인합니다.
- 동일 조건에서 status/code/message가 바뀌지 않는 behavior-preserving refactor인지 확인합니다.
- 모든 caller가 `httpBoundary.ts`의 owner를 직접 사용해 중복 failure path가 사라지는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 주요 route는 typed boundary를 적용했지만 app 내부에 같은 의미의 legacy helper가 남아 있었습니다. |
| 해결하려던 문제 | 중복 helper가 이후 서로 다른 status/code/message로 drift할 수 있었습니다. |
| 핵심 결정 | route-local helper를 제거하고 모든 caller가 공통 unauthorized/suspended 함수를 import하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | condition이 참이면 공통 helper가 `ApiHttpError`를 던지고 설치된 handler가 동일 error envelope를 보냅니다. |
| ownership/lifetime/cleanup | failure classification owner를 `httpBoundary.ts`로 단일화합니다. |
| failure/rollback/retry | behavior를 추가하지 않으며 repository rollback·strict query coverage는 그대로입니다. |
| 보장하는 것 | 인증/정지 failure의 구현 위치가 하나로 줄어듭니다. |
| 보장하지 않는 것 | 중앙 helper가 올바르게 호출되는지에 대한 독립 새 test는 이 commit 자체에 없습니다. |
| 후속 연결 | `50caaf5c7c49`가 기존 API tests를 최종 cookie/typed response 기대값에 맞춥니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 legacy helper 삭제와 import 변경으로 충분히 확인됩니다.

비교 기준:
- 직전 관련 SHA: `24f99345452d` — `feat(api): 토너먼트·관리 HTTP contract 적용`
- 다음 관련 SHA: `50caaf5c7c49` — `test(api): typed HTTP boundary 기대값 정렬`

### 5.19. `test(api): typed HTTP boundary 기대값 정렬`

| 항목 | 값 |
| --- | --- |
| SHA | `50caaf5c7c49` |
| Importance | B |
| Tags | AUTH, TOURNAMENT, WEB |
| Source에서 확정된 역할 | 기존 API·관리·토너먼트 통합 test를 HttpOnly cookie와 typed response/error contract에 맞춥니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/app.test.ts`, `admin.test.ts`, `tournament.test.ts`의 bearer token 제거와 `set-cookie` 추출 helper를 확인합니다.
- 로그인 JSON에 `token`이 없다는 assertion과 cookie를 이용한 보호 route 호출을 확인합니다.
- development seed profile, 명시적 admin fixture, suspension/audit assertion이 어떤 선행 변경을 흡수하는지 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 integration tests는 로그인 JSON token과 bearer header를 전제로 해 typed boundary 이후 production contract와 어긋났습니다. |
| 해결하려던 문제 | 구현은 cookie 기반으로 바뀌었지만 test가 legacy credential transport를 계속 사용하면 잘못된 API를 보호하게 됩니다. |
| 핵심 결정 | `set-cookie`에서 `pp_session`을 추출해 보호 route에 전달하고 로그인 body에 raw token이 없음을 검사하도록 tests를 갱신했습니다. |
| 입력 → 상태 전이 → 출력 | login injection → cookie 추출 → 보호 route injection → typed response/status assertion 순으로 실행합니다. |
| ownership/lifetime/cleanup | 각 fixture가 session cookie value와 app/repo cleanup을 소유합니다. production session state는 repository가 소유합니다. |
| failure/rollback/retry | 실제 browser cookie policy, network CORS, PostgreSQL은 사용하지 않습니다. 이 commit은 여러 선행 auth/seed 변화를 함께 반영합니다. |
| 보장하는 것 | in-process API tests가 현재 cookie/typed boundary와 일치하고 raw token response 회귀를 감지합니다. |
| 보장하지 않는 것 | cookie attributes의 browser 적용, cross-origin credential 전송, real database behavior는 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 최종 API contract verification이며 cookie-only auth의 전체 보안 invariant는 별도 identity category가 다룹니다. |

#### 최소 코드 근거

세 test file의 `sessionCookie` helper와 `not.toHaveProperty("token")` assertion이 핵심 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | typed HTTP boundary 이후 보호 route는 session cookie를 소비하고 login JSON은 raw token을 노출하지 않아야 합니다. |
| 재현하는 failure/boundary | legacy bearer/token fixture를 cookie fixture로 바꾸고 API·admin·tournament 대표 경로를 다시 실행합니다. |
| test technique | in-process Fastify integration regression test + memory repository |
| 통과하는 production path | login → Set-Cookie 추출 → protected route → shared/typed response |
| 증명하는 것 | 대표 resource tests가 최종 credential/response contract와 일치함을 증명합니다. |
| 증명하지 않는 것 | 브라우저 CORS·SameSite enforcement, 실제 network, PostgreSQL은 증명하지 않습니다. |
| 후속 회귀 방지 | 향후 route refactor가 raw token을 다시 노출하거나 cookie 경로를 깨는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `b2a8de5a0027` — `refactor(api): HTTP boundary helper 통합`

## 6. Invariant evolution

| 단계 | 관련 SHA | 기록 |
| --- | --- | --- |
| Resource API 도입 | `1779df300611` → `0bcc487d949f` → `e8bb6a4bf68b` | 로그인/로비에서 profile/friend, tournament/admin으로 route가 확장되지만 assertion과 local error가 남습니다. |
| 초기 행동 기준 | `fb1c287d9e79`, `1395d45a3665`, `5088099d1e7d` | Fastify injection과 memory repository로 초기 happy path를 고정합니다. |
| Executable domain contract | `0c5c27c8c3df` → `7d0793a23f5d` | compile-time DTO가 domain별 Zod schema로 교체됩니다. |
| Wire contract 완성 | `282a9d0beb47` → `e226b68fe235` → `78cf83f29e80` | strict request/error와 endpoint response schema를 만들고 schema rule을 unit test합니다. |
| API boundary 적용 | `ac85316bb0cb` → `c4cba7d3f871` → `05e3ecfa2a2d` → `24f99345452d` → `b2a8de5a0027` | typed failure와 input/output parse를 모든 기존 resource에 적용하고 helper owner를 통합합니다. |
| 최종 통합 기대값 | `50caaf5c7c49` | 대표 tests가 session cookie와 token 비노출을 기준으로 갱신됩니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 불충분한 상태 | Fix/구현 SHA | Test 또는 후속 증거 |
| --- | --- | --- |
| Type assertion/default가 malformed input을 통과·보정함 | `282a9d0beb47` + route 적용 commits | strict schema와 handler parse를 도입합니다. |
| Route-local 오류 body가 서로 다름 | `ac85316bb0cb` | Fastify error handler와 shared error envelope로 통합합니다. |
| 공통 boundary가 있어도 일부 route가 미적용 | `c4cba7d3f871` → `24f99345452d` | resource 군별로 input/output/error 적용을 완료합니다. |
| Legacy helper가 중앙 owner와 중복 | `b2a8de5a0027` | route-local helper를 제거합니다. |
| Tests가 JSON token/bearer contract를 계속 보호함 | `50caaf5c7c49` | Set-Cookie 기반 fixture와 raw-token negative assertion으로 정렬합니다. |

## 8. Ownership·state·responsibility 변화

| Owner | 최종 책임 | 명시적 비책임 |
| --- | --- | --- |
| Shared domain/request/response schema | 허용 JSON shape, normalization, field constraint | HTTP status와 repository side effect를 소유하지 않습니다. |
| Fastify route | endpoint 선택, authorization order, repository method 호출 | wire error formatting은 boundary에 위임합니다. |
| HTTP boundary module | input/output parse, typed error classification, request ID, generic 500 | domain state rollback은 소유하지 않습니다. |
| Repository | session 및 resource state/read-write operation | HTTP schema와 public error envelope를 소유하지 않습니다. |
| Integration tests | app/repo fixture와 대표 request sequence | 실제 browser/network/PostgreSQL을 대신하지 않습니다. |

## 9. Thread 최종 상태

- 최종 SHA 기준으로 기존 JSON resource는 shared runtime contract와 중앙 typed error boundary를 사용합니다.
- handler는 검증된 input을 repository에 전달하고 반환값을 endpoint response schema로 검사합니다.
- 예상 오류는 code/status/requestId/fieldErrors를 가진 공통 envelope로, 예상 밖 오류는 generic 500으로 나갑니다.
- 대표 API tests는 HttpOnly session cookie contract에 맞지만 browser/CORS/real DB 증거는 별도 category의 책임입니다.

### 실행 검증 기록

- Source inspection: GitHub connector로 Commit map의 exact SHA diff와 필요한 historical file을 조회했습니다.
- Branch validation: branch-local `commit/commit-importance.md`가 선언한 433개 선형 이력에서 모든 참조 SHA와 source classification을 대조했고, 가장 이른 참조 `b625c4f9dfdc`는 branch HEAD 비교에서 merge base가 동일 SHA임을 확인했습니다.
- 실제 실행 시도: `git clone --branch web/ft_transcendence --single-branch https://github.com/seungwoo7050/42-archive.git /tmp/ft-transcendence-audit`
- 결과: exit status 128, `Could not resolve host: github.com`. 따라서 repository checkout, package install, test command는 실행하지 않았으며 runtime 결과를 주장하지 않습니다.

## 10. 최종 실행 흐름

1. Fastify가 method/path로 route를 선택합니다.
2. route가 shared request schema로 input을 parse합니다.
3. 인증·계정 상태·role 같은 capability를 검사합니다.
4. 검증된 ID/body를 repository method에 전달합니다.
5. repository 결과를 endpoint response schema로 parse합니다.
6. 성공이면 typed JSON을 반환합니다.
7. 예상 failure는 `ApiHttpError`, 예상 밖 failure는 generic internal error envelope로 변환됩니다.

## 11. 학습 완료 확인

- [x] 모든 Commit map SHA의 historical code를 확인했습니다.
- [x] parent/직전 관련 상태와 현재 SHA를 구분했습니다.
- [x] fix의 이전 가정·root cause·corrected invariant를 연결했습니다.
- [x] test가 통과하는 production path와 비증명 범위를 구분했습니다.
- [x] ownership/lifetime/cleanup과 failure path를 기록했습니다.
- [x] 실행 evidence와 code inspection을 구분했습니다.
- [x] 마지막 SHA 기준 최종 flow와 non-guarantee를 설명할 수 있습니다.
===== END FILE: 02-executable-http-contracts-and-resource-api.md =====

===== BEGIN FILE: 03-strict-json-request-validation.md =====
# Strict JSON request validation

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

bodyless route의 국소 normalization 문제에서 출발해 모든 JSON route의 params/query/body를 strict shared contract로 정의하고 business logic 전에 공통 parser로 집행한 뒤 route-wide regression test로 고정하는 과정을 복원합니다.

이 문서는 Phase 1 category audit 후 동결된 scaffold를 기준으로 exact SHA의 구현 발전을 복원합니다.

### 직접 연결되는 불변식

- 모든 현재 JSON route는 untrusted params/query/body를 business logic 전에 strict schema로 검증합니다.
- bodyless route도 명시적 empty-object contract를 가져 unknown query/body를 허용하지 않습니다.
- validation failure는 repository mutation 전에 공통 typed error envelope로 종료됩니다.
- forwarded client address는 explicit proxy trust가 없을 때 identity/rate-limit key를 바꾸지 못합니다.

## 2. 핵심 질문

- `request.body ?? {}`라는 local fix가 왜 route-wide contract map으로 일반화되어야 했습니까?
- empty params/query/body contract가 bodyless GET/POST의 unknown field를 어떻게 거부합니까?
- `parseHttpRequest`가 반환한 값만 repository call에 쓰인다는 근거는 무엇입니까?
- route matrix test가 증명하는 coverage와 새 route 추가 시 남는 수동 책임은 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, symbol, caller/callee, 상태 mutation, failure branch, cleanup을 actual historical code로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- S/A/B/C 중요도에 맞춰 설명 깊이를 다르게 유지합니다.
- 실행하지 않은 command 결과를 작성하지 않으며 code inspection과 runtime evidence를 구분합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `8ce1199ffd12` | `fix(api): body 없는 로비 채팅 요청 처리` | B | - | 선택적 request body를 `{}`로 정규화해 route-local Zod parse가 `undefined`에서 실패하던 문제를 수정합니다. |
| 2 | `d07056e11871` | `feat(shared): 모든 HTTP request schema를 strict하게 정의` | A | PROTOCOL | 모든 JSON route에 params/query/body 세 schema를 가진 strict request contract map을 정의합니다. |
| 3 | `59d75fddcaa6` | `fix(api): 모든 route input을 runtime 검증` | A | PROTOCOL, RISK | 공통 `parseHttpRequest`를 모든 JSON handler의 business logic 전에 호출해 params/query/body strictness를 실제로 집행합니다. |
| 4 | `1abbf7dcdde4` | `test(api): strict request contract 검증` | B | TEST | 전체 JSON route의 unknown query/body와 invalid path parameter, untrusted forwarded address를 table-driven Fastify injection으로 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `fix(api): body 없는 로비 채팅 요청 처리`

| 항목 | 값 |
| --- | --- |
| SHA | `8ce1199ffd12` |
| Importance | B |
| Tags | - |
| Source에서 확정된 역할 | 선택적 request body를 `{}`로 정규화해 route-local Zod parse가 `undefined`에서 실패하던 문제를 수정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent와 `apps/api/src/app.ts`의 lobby chat handler를 비교해 `request.body`와 `request.body ?? {}` 차이를 확인합니다.
- body가 없을 때 Fastify가 전달하는 값과 empty-object schema가 기대하는 입력을 연결합니다.
- 이 수정이 해당 route에만 적용되고 전체 route contract는 만들지 않는다는 한계를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 로비 채팅 route는 request body가 없을 때 `undefined`를 그대로 empty-object schema에 전달했습니다. |
| 해결하려던 문제 | bodyless 호출도 허용하려는 route에서 parser가 object 대신 `undefined`를 받아 validation failure가 발생했습니다. |
| 핵심 결정 | `request.body ?? {}`로 optional body를 empty object에 정규화한 뒤 기존 schema에 전달했습니다. |
| 입력 → 상태 전이 → 출력 | Fastify request body가 `undefined`이면 `{}`로 대체되고 schema parse 후 handler가 계속 진행합니다. |
| ownership/lifetime/cleanup | route handler가 normalization을 소유하며 새 장기 resource는 없습니다. |
| failure/rollback/retry | 잘못된 non-empty body는 기존 schema에서 거부되지만 다른 route의 params/query/body에는 적용되지 않습니다. |
| 보장하는 것 | 해당 bodyless lobby chat 경로가 missing body를 empty object로 해석합니다. |
| 보장하지 않는 것 | 모든 route에 같은 normalization·strictness가 적용된다는 보장은 없습니다. |
| 후속 연결 | `d07056e11871`과 `59d75fddcaa6`가 이 local rule을 공통 request contract로 일반화합니다. |

#### 최소 코드 근거

`8ce1199ffd12`, `apps/api/src/app.ts`의 `request.body ?? {}` 변경이 근거입니다.

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | empty object schema를 호출하면 request body도 항상 object일 것이라고 가정했습니다. |
| 실제 실패 또는 위험 | body가 생략된 요청에서 Fastify가 `undefined`를 제공해 validation이 실패했습니다. |
| Root cause | optional transport field와 schema input 사이 normalization이 route에 없었습니다. |
| 수정된 invariant | bodyless request는 parsing 전에 `{}`로 정규화해야 합니다. |
| 변경 코드 | `apps/api/src/app.ts` lobby chat handler의 nullish fallback입니다. |
| Regression evidence | 후속 `1abbf7dcdde4`의 body/route matrix가 공통 strict parser 적용 뒤 body contract를 보호합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `d07056e11871` — `feat(shared): 모든 HTTP request schema를 strict하게 정의`

### 5.2. `feat(shared): 모든 HTTP request schema를 strict하게 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `d07056e11871` |
| Importance | A |
| Tags | PROTOCOL |
| Source에서 확정된 역할 | 모든 JSON route에 params/query/body 세 schema를 가진 strict request contract map을 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/src/http.ts`의 `defineHttpRequestContract`, empty/id contract와 `jsonHttpRequestContracts` 전체 key를 확인합니다.
- bodyless route도 empty strict object를 명시해 unknown query/body를 거부할 수 있는지 확인합니다.
- route 목록과 contract map이 일대일로 대응하고 params 종류가 올바른지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 개별 input schema는 있었지만 route마다 params/query/body 중 일부만 직접 parse했고 bodyless route는 아무 검증도 하지 않았습니다. |
| 해결하려던 문제 | unknown query나 body field가 route에 따라 통과하고 새 route가 validation 호출을 빠뜨릴 수 있었습니다. |
| 핵심 결정 | 모든 JSON route key를 params/query/body contract에 매핑하고 empty route도 세 축 모두 strict empty object를 사용하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | API가 contract를 선택해 Fastify request의 세 부분을 각각 parse하면 normalized typed tuple을 얻습니다. |
| ownership/lifetime/cleanup | shared package가 route별 wire contract map을 소유합니다. API는 실제 route와 올바른 key를 연결할 책임을 가집니다. |
| failure/rollback/retry | 이 SHA는 map만 정의하므로 route가 호출하지 않으면 runtime behavior는 바뀌지 않습니다. route registry와 자동 일치 검사도 없습니다. |
| 보장하는 것 | 모든 기존 JSON endpoint에 명시적 strict params/query/body 정의가 존재합니다. |
| 보장하지 않는 것 | Fastify handler 전부가 이를 사용한다는 보장은 다음 commit 전에는 없습니다. |
| 후속 연결 | `59d75fddcaa6`이 map을 모든 handler의 business logic 앞에 적용합니다. |

#### 최소 코드 근거

`d07056e11871`, `packages/shared/src/http.ts`:

```ts
const emptyHttpRequestContract = defineHttpRequestContract(
  emptyParamsSchema, emptyParamsSchema, emptyParamsSchema
);
export const jsonHttpRequestContracts = { /* 모든 JSON route */ } as const;
```

비교 기준:
- 직전 관련 SHA: `8ce1199ffd12` — `fix(api): body 없는 로비 채팅 요청 처리`
- 다음 관련 SHA: `59d75fddcaa6` — `fix(api): 모든 route input을 runtime 검증`

### 5.3. `fix(api): 모든 route input을 runtime 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `59d75fddcaa6` |
| Importance | A |
| Tags | PROTOCOL, RISK |
| Source에서 확정된 역할 | 공통 `parseHttpRequest`를 모든 JSON handler의 business logic 전에 호출해 params/query/body strictness를 실제로 집행합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/httpBoundary.ts`의 `parseHttpRequest`가 `request.params/query/body ?? {}`를 각각 parse하는지 확인합니다.
- `apps/api/src/app.ts`의 health부터 admin까지 모든 JSON route가 올바른 contract key를 호출하는지 확인합니다.
- parsed params/body를 이후 repository call에 사용하며 raw request assertion이 제거되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared contract map은 존재했지만 app handler는 기존 부분 parse를 계속 사용해 unknown input이 route마다 다르게 처리됐습니다. |
| 해결하려던 문제 | 정의만 있고 소비되지 않는 schema는 신뢰 경계를 만들지 못하며 business logic이 untrusted raw input을 먼저 볼 수 있었습니다. |
| 핵심 결정 | `parseHttpRequest`가 세 request component를 공통 방식으로 parse하도록 하고 모든 JSON route 첫 단계에 배치했습니다. |
| 입력 → 상태 전이 → 출력 | route contract 선택 → params/query/body nullish normalization → 세 strict parse → typed 값 추출 → 인증·repository logic → output parse 순입니다. |
| ownership/lifetime/cleanup | HTTP boundary가 request parsing을 소유하고 handler는 parser가 반환한 값만 business operation에 전달합니다. |
| failure/rollback/retry | validation 실패는 `ApiHttpError`로 중단되며 mutation 전이므로 rollback이 필요 없습니다. schema map과 route 목록의 자동 compile-time 완전성은 제한적입니다. |
| 보장하는 것 | 모든 현재 JSON route에서 unknown query/body와 invalid path가 business logic 전에 거부됩니다. |
| 보장하지 않는 것 | multipart, WebSocket payload, future route가 map 적용을 누락하는 경우까지 자동 보장하지 않습니다. |
| 후속 연결 | `1abbf7dcdde4`가 전체 route matrix로 이 invariant를 회귀 검증합니다. |

#### 최소 코드 근거

`59d75fddcaa6`, `apps/api/src/httpBoundary.ts`:

```ts
return {
  params: parseInput(contract.params, request.params ?? {}),
  query: parseInput(contract.query, request.query ?? {}),
  body: parseInput(contract.body, request.body ?? {})
};
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | route별로 필요한 field만 parse하면 전체 input boundary가 충분하다고 가정했습니다. |
| 실제 실패 또는 위험 | 검사하지 않은 query/body field와 bodyless route input이 handler마다 다르게 통과했습니다. |
| Root cause | route 단위 세 축 contract가 실제 handler entry에서 호출되지 않았습니다. |
| 수정된 invariant | 모든 JSON route는 business logic 전에 params/query/body를 동일 parser로 검증해야 합니다. |
| 변경 코드 | `httpBoundary.ts::parseHttpRequest`와 `app.ts` 전 route 호출입니다. |
| Regression evidence | `1abbf7dcdde4`의 table-driven route matrix입니다. |

비교 기준:
- 직전 관련 SHA: `d07056e11871` — `feat(shared): 모든 HTTP request schema를 strict하게 정의`
- 다음 관련 SHA: `1abbf7dcdde4` — `test(api): strict request contract 검증`

### 5.4. `test(api): strict request contract 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1abbf7dcdde4` |
| Importance | B |
| Tags | TEST |
| Source에서 확정된 역할 | 전체 JSON route의 unknown query/body와 invalid path parameter, untrusted forwarded address를 table-driven Fastify injection으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/http-contract.test.ts`의 `jsonRoutes`, `jsonBodyRoutes`, invalid params case를 확인합니다.
- 각 request가 실제 `buildApp` route와 shared error envelope를 통과하는지 확인합니다.
- `guest-demo.test.ts`의 `trustProxy` false에서 forwarded address spoof가 rate-limit identity를 바꾸지 못하는 case를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공통 parser가 적용됐지만 일부 route key 누락·잘못된 schema 연결을 정적 코드만으로 모두 탐지하기 어려웠습니다. |
| 해결하려던 문제 | 특정 route가 unknown query/body나 invalid path를 받아들이는 회귀를 route 수만큼 반복 확인할 필요가 있었습니다. |
| 핵심 결정 | 27개 JSON route와 body-bearing subset을 table로 만들고 unknown field를 주입했으며 invalid UUID/handle과 bodyless contract를 별도로 검사했습니다. |
| 입력 → 상태 전이 → 출력 | Fastify injection → 실제 route → `parseHttpRequest` → typed boundary error envelope를 관찰합니다. proxy case는 두 spoofed header 요청이 같은 direct IP rate key를 공유하는지 확인합니다. |
| ownership/lifetime/cleanup | fixture가 memory repo와 app을 소유·정리합니다. proxy identity는 Fastify 설정과 `request.ip`가 결정합니다. |
| failure/rollback/retry | 실제 reverse proxy/network는 없고 in-process injection입니다. 새 route가 table에 추가되지 않으면 자동 coverage는 생기지 않습니다. |
| 보장하는 것 | 현재 route set의 strict query/body/path와 trustProxy false behavior를 결정적으로 검증합니다. |
| 보장하지 않는 것 | 실제 browser, CORS preflight, PostgreSQL, WebSocket input은 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 공통 strict boundary를 고정하며 WS payload는 별도 Thread가 다룹니다. |

#### 최소 코드 근거

`apps/api/src/http-contract.test.ts`의 route matrix와 `guest-demo.test.ts`의 forwarded-address case가 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 모든 현재 JSON route가 params/query/body strict contract를 business logic 전에 집행해야 합니다. |
| 재현하는 failure/boundary | unknown query/body, invalid UUID/handle, bodyless route의 explicit empty body contract와 untrusted forwarded address입니다. |
| test technique | in-process table-driven API integration/boundary test |
| 통과하는 production path | Fastify inject → route → `parseHttpRequest` → `parseInput` → shared error envelope |
| 증명하는 것 | 현재 table에 열거된 route에서 strict input rejection과 trustProxy false identity behavior를 증명합니다. |
| 증명하지 않는 것 | 실제 proxy hop, browser, real DB, future route 자동 등록은 증명하지 않습니다. |
| 후속 회귀 방지 | route별 parser 누락·잘못된 contract key·strictness 완화 회귀를 방지합니다. |

비교 기준:
- 직전 관련 SHA: `59d75fddcaa6` — `fix(api): 모든 route input을 runtime 검증`

## 6. Invariant evolution

| 단계 | 관련 SHA | 기록 |
| --- | --- | --- |
| Local normalization | `8ce1199ffd12` | 한 bodyless route에서 missing body를 `{}`로 처리합니다. |
| Route-wide contract definition | `d07056e11871` | 모든 JSON route에 strict params/query/body map을 정의합니다. |
| Runtime enforcement | `59d75fddcaa6` | 모든 handler entry가 공통 parser를 실행하고 raw request assertion을 제거합니다. |
| Deterministic verification | `1abbf7dcdde4` | route matrices가 unknown field와 invalid path, untrusted forwarded address를 거부하는지 검증합니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 불충분한 상태 | Fix/구현 SHA | Test 또는 후속 증거 |
| --- | --- | --- |
| Absent body가 `undefined`라 local parse 실패 | `8ce1199ffd12` | body를 `{}`로 정규화합니다. |
| Route마다 일부 input만 검사 | `d07056e11871` + `59d75fddcaa6` | 세 축 contract map과 공통 parser를 도입합니다. |
| 적용 누락을 코드 리뷰만으로 찾기 어려움 | `1abbf7dcdde4` | 모든 현행 route를 table-driven injection으로 순회합니다. |

## 8. Ownership·state·responsibility 변화

| Owner | 최종 책임 | 명시적 비책임 |
| --- | --- | --- |
| Shared request map | route별 params/query/body schema | Fastify route registration을 자동 생성하지 않습니다. |
| HTTP boundary parser | nullish normalization과 세 schema 실행 | business mutation을 소유하지 않습니다. |
| Route handler | parser 반환값으로 authorization/repository logic 수행 | raw request 값을 직접 신뢰하지 않습니다. |
| Regression suite | 현행 route 목록과 negative input matrix | future route 자동 발견은 하지 않습니다. |

## 9. Thread 최종 상태

- 최종 SHA 기준으로 현재 JSON route는 strict params/query/body를 모두 검증합니다.
- unknown field와 invalid path는 repository operation 전에 validation error로 중단됩니다.
- in-process route matrix가 current coverage를 보호하지만 future route를 test table과 contract map에 추가하는 책임은 남습니다.

### 실행 검증 기록

- Source inspection: GitHub connector로 Commit map의 exact SHA diff와 필요한 historical file을 조회했습니다.
- Branch validation: branch-local `commit/commit-importance.md`가 선언한 433개 선형 이력에서 모든 참조 SHA와 source classification을 대조했고, 가장 이른 참조 `b625c4f9dfdc`는 branch HEAD 비교에서 merge base가 동일 SHA임을 확인했습니다.
- 실제 실행 시도: `git clone --branch web/ft_transcendence --single-branch https://github.com/seungwoo7050/42-archive.git /tmp/ft-transcendence-audit`
- 결과: exit status 128, `Could not resolve host: github.com`. 따라서 repository checkout, package install, test command는 실행하지 않았으며 runtime 결과를 주장하지 않습니다.

## 10. 최종 실행 흐름

1. route가 `jsonHttpRequestContracts`에서 자신의 contract를 선택합니다.
2. `parseHttpRequest`가 missing params/query/body를 `{}`로 정규화합니다.
3. 세 strict schema가 각각 값을 parse합니다.
4. 하나라도 실패하면 typed validation error로 종료합니다.
5. 모두 성공하면 parsed 값만 authorization과 repository operation에 전달합니다.
6. output은 기존 typed HTTP boundary를 통해 검증·반환됩니다.

## 11. 학습 완료 확인

- [x] 모든 Commit map SHA의 historical code를 확인했습니다.
- [x] parent/직전 관련 상태와 현재 SHA를 구분했습니다.
- [x] fix의 이전 가정·root cause·corrected invariant를 연결했습니다.
- [x] test가 통과하는 production path와 비증명 범위를 구분했습니다.
- [x] ownership/lifetime/cleanup과 failure path를 기록했습니다.
- [x] 실행 evidence와 code inspection을 구분했습니다.
- [x] 마지막 SHA 기준 최종 flow와 non-guarantee를 설명할 수 있습니다.
===== END FILE: 03-strict-json-request-validation.md =====

===== BEGIN FILE: 04-websocket-internal-error-containment.md =====
# WebSocket client error boundary와 내부 오류 격리

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

WebSocket message parse failure와 command/repository processing failure가 한 catch에 결합되어 있던 상태를 분리하고, 내부 예외 원문을 고정된 public message로 redaction한 뒤 deterministic repository failure injection으로 검증하는 과정을 복원합니다.

이 문서는 Phase 1 category audit 후 동결된 scaffold를 기준으로 exact SHA의 구현 발전을 복원합니다.

### 직접 연결되는 불변식

- 잘못된 client payload는 `invalid_event`, server-side processing failure는 `internal_error`로 구분됩니다.
- repository·SQL·internal host·raw exception message는 client-facing WebSocket event에 포함되지 않습니다.
- parse 실패는 command dispatch 전에 즉시 반환하고 processing failure는 동일 connection의 public error event로 격리됩니다.

## 2. 핵심 질문

- 기존 단일 catch가 왜 client fault와 server fault를 동시에 오분류했습니까?
- parse 단계와 async repository command 단계를 분리하면 public code와 failure owner가 어떻게 달라집니까?
- failure injection test는 어느 repository call을 실패시키고 어떤 문자열 부재를 검사합니까?
- 이 redaction이 보장하지 않는 rollback·logging·다른 async callback 범위는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, symbol, caller/callee, 상태 mutation, failure branch, cleanup을 actual historical code로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- S/A/B/C 중요도에 맞춰 설명 깊이를 다르게 유지합니다.
- 실행하지 않은 command 결과를 작성하지 않으며 code inspection과 runtime evidence를 구분합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fe62962d65d9` | `fix(api): 내부 WebSocket 오류 숨김` | A | PROTOCOL, REALTIME, PERSISTENCE | client event parse failure와 command/repository processing failure를 분리하고 raw exception을 WebSocket response에서 제거합니다. |
| 2 | `20933b1393f3` | `test(api): WebSocket repository error redaction 검증` | B | PROTOCOL, REALTIME, PERSISTENCE | chat repository method에 내부 SQL·host 문자열이 든 예외를 주입하고 client event가 generic error만 포함하는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `fix(api): 내부 WebSocket 오류 숨김`

| 항목 | 값 |
| --- | --- |
| SHA | `fe62962d65d9` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | client event parse failure와 command/repository processing failure를 분리하고 raw exception을 WebSocket response에서 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent와 `apps/api/src/gameHub.ts::receive`를 비교해 단일 catch가 parse와 processing을 함께 처리하던 상태를 확인합니다.
- `INVALID_EVENT_MESSAGE`, `INTERNAL_ERROR_MESSAGE`와 `invalid_event`/`internal_error` code가 어떤 catch에서 사용되는지 확인합니다.
- AI fallback promise catch도 raw error message 대신 같은 internal message를 사용하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `GameHub.receive`의 한 catch가 JSON/schema parse와 repository/command 실패를 모두 `invalid_event`로 처리하며 `error.message`를 client에게 보냈습니다. |
| 해결하려던 문제 | SQL text, host, stack 단서 같은 내부 정보가 wire response에 노출되고 client 입력 오류와 server failure가 같은 code로 오분류될 수 있었습니다. |
| 핵심 결정 | event parse를 첫 try/catch로 분리하고 processing을 두 번째 try/catch로 감쌌으며 두 public message를 상수화했습니다. |
| 입력 → 상태 전이 → 출력 | payload parse 실패는 generic `invalid_event`를 보내고 즉시 반환합니다. parse 성공 후 command/repository 실패는 generic `internal_error`를 보냅니다. |
| ownership/lifetime/cleanup | GameHub가 WebSocket public failure classification을 소유합니다. repository exception 내용은 server 내부에 남고 socket event에는 복사되지 않습니다. |
| failure/rollback/retry | processing failure의 rollback이나 retry는 추가하지 않습니다. chat write가 일부 수행된 뒤 후속 broadcast가 실패하는 경우의 atomicity도 보장하지 않습니다. |
| 보장하는 것 | 잘못된 client event와 내부 처리 실패가 다른 code로 분리되고 raw exception message가 client event에 포함되지 않습니다. |
| 보장하지 않는 것 | 서버 log에 오류가 기록된다는 보장, 모든 async callback의 redaction, repository rollback은 이 commit 범위 밖입니다. |
| 후속 연결 | `20933b1393f3`가 repository error를 결정적으로 주입해 redaction을 검증합니다. |

#### 최소 코드 근거

`fe62962d65d9`, `apps/api/src/gameHub.ts::receive`:

```ts
try { event = parseClientEvent(payload); }
catch { sendInvalidEvent(); return; }
try { /* command와 repository 호출 */ }
catch { sendInternalError(); }
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | 모든 receive failure를 잘못된 client event로 취급하고 예외 message를 진단 정보로 보내도 된다고 가정했습니다. |
| 실제 실패 또는 위험 | repository exception의 SQL·host·민감한 column 정보가 WebSocket client에게 노출될 수 있었습니다. |
| Root cause | parse failure와 내부 processing failure가 한 catch에 결합되어 public classification과 redaction owner가 없었습니다. |
| 수정된 invariant | client parse failure와 server internal failure는 분리하고 public message는 고정된 generic 값만 사용해야 합니다. |
| 변경 코드 | `apps/api/src/gameHub.ts::receive`, AI fallback catch입니다. |
| Regression evidence | `20933b1393f3`의 injected repository rejection과 secret-string negative assertion입니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `20933b1393f3` — `test(api): WebSocket repository error redaction 검증`

### 5.2. `test(api): WebSocket repository error redaction 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `20933b1393f3` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | chat repository method에 내부 SQL·host 문자열이 든 예외를 주입하고 client event가 generic error만 포함하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/gameHub.runtime.test.ts`에서 memory repository method spy와 rejected error 문자열을 확인합니다.
- `FakeSocket.receive`가 실제 `GameHub.receive` message listener를 통과하는지 확인합니다.
- error event code/message와 serialized event 전체의 secret-string 부재를 함께 검사하는 이유를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | redaction code는 구현됐지만 repository가 실제로 민감한 message를 던질 때 wire event가 안전하다는 결정적 증거가 없었습니다. |
| 해결하려던 문제 | 단순 expected message assertion만으로는 다른 event field에 원문이 남는 회귀를 놓칠 수 있었습니다. |
| 핵심 결정 | `createChatMessage`를 SQL·host 문자열을 포함한 rejected promise로 바꾸고 FakeSocket으로 `chat.send`를 전달했습니다. |
| 입력 → 상태 전이 → 출력 | fake socket message → GameHub parse → repository spy rejection → processing catch → generic error event → poll/assert 순입니다. |
| ownership/lifetime/cleanup | fixture가 memory repo, GameHub, FakeSocket을 소유하고 test 끝에 socket을 terminate합니다. |
| failure/rollback/retry | 실제 PostgreSQL이나 network socket은 사용하지 않습니다. 내부 error가 server log에 남는지는 검사하지 않습니다. |
| 보장하는 것 | 이 production path에서 client는 `internal_error`와 고정 message만 받고 `password_hash`·internal host를 관찰하지 못합니다. |
| 보장하지 않는 것 | 다른 repository method·async callback·HTTP error path 전체의 redaction은 증명하지 않습니다. |
| 후속 연결 | `fe62962d65d9`의 보안 invariant를 deterministic failure injection으로 고정합니다. |

#### 최소 코드 근거

`apps/api/src/gameHub.runtime.test.ts`의 `vi.spyOn(...).mockRejectedValue`와 serialized event negative assertions가 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | WebSocket processing failure는 repository exception 원문을 client에게 노출하지 않아야 합니다. |
| 재현하는 failure/boundary | chat write가 SQL text와 internal host를 포함한 예외로 실패하는 경계입니다. |
| test technique | deterministic failure injection + in-memory GameHub/FakeSocket integration test |
| 통과하는 production path | FakeSocket message → `GameHub.receive` → `repo.createChatMessage` rejection → internal catch → socket event |
| 증명하는 것 | 주입한 민감 문자열이 모든 emitted event serialization에 없고 generic code/message만 존재함을 증명합니다. |
| 증명하지 않는 것 | 실제 DB driver, real WebSocket transport, 다른 command path, server logging은 증명하지 않습니다. |
| 후속 회귀 방지 | raw exception을 다시 response에 넣거나 parse/internal code를 합치는 회귀를 방지합니다. |

비교 기준:
- 직전 관련 SHA: `fe62962d65d9` — `fix(api): 내부 WebSocket 오류 숨김`

## 6. Invariant evolution

| 단계 | 관련 SHA | 기록 |
| --- | --- | --- |
| 불충분한 상태 | `fe62962d65d9` parent | parse와 processing failure가 모두 `invalid_event`이며 raw exception message가 전송될 수 있습니다. |
| Corrected boundary | `fe62962d65d9` | 두 catch와 두 고정 public message로 client/internal failure를 분리합니다. |
| Deterministic verification | `20933b1393f3` | repository rejection에 민감 문자열을 넣고 emitted event 전체에서 부재를 검사합니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 불충분한 상태 | Fix/구현 SHA | Test 또는 후속 증거 |
| --- | --- | --- |
| 한 catch가 parse와 repository error를 합침 | `fe62962d65d9` | parse catch와 processing catch를 분리하고 raw message 사용을 제거합니다. |
| 정적 코드만으로 redaction 회귀를 놓칠 수 있음 | `20933b1393f3` | 민감 문자열이 든 repository failure를 주입하고 serialized event를 검사합니다. |

## 8. Ownership·state·responsibility 변화

| Owner | 최종 책임 | 명시적 비책임 |
| --- | --- | --- |
| Shared WS parser | client payload JSON/schema validation | repository command 결과를 소유하지 않습니다. |
| GameHub receive | public failure classification과 socket error event | repository rollback을 소유하지 않습니다. |
| Repository | chat write 등 command side effect | exception 원문을 public contract로 정의하지 않습니다. |
| Failure-injection test | rejected repository call과 FakeSocket event 관찰 | 실제 DB/network를 대신하지 않습니다. |

## 9. Thread 최종 상태

- 최종 SHA 기준으로 malformed payload와 내부 processing failure는 서로 다른 public code를 사용합니다.
- repository exception의 SQL/host text는 client event에 복사되지 않습니다.
- 이 Thread는 response redaction을 다루며 command rollback, retry, server logging completeness는 보장하지 않습니다.

### 실행 검증 기록

- Source inspection: GitHub connector로 Commit map의 exact SHA diff와 필요한 historical file을 조회했습니다.
- Branch validation: branch-local `commit/commit-importance.md`가 선언한 433개 선형 이력에서 모든 참조 SHA와 source classification을 대조했고, 가장 이른 참조 `b625c4f9dfdc`는 branch HEAD 비교에서 merge base가 동일 SHA임을 확인했습니다.
- 실제 실행 시도: `git clone --branch web/ft_transcendence --single-branch https://github.com/seungwoo7050/42-archive.git /tmp/ft-transcendence-audit`
- 결과: exit status 128, `Could not resolve host: github.com`. 따라서 repository checkout, package install, test command는 실행하지 않았으며 runtime 결과를 주장하지 않습니다.

## 10. 최종 실행 흐름

1. 소켓 message가 `GameHub.receive`에 들어옵니다.
2. `parseClientEvent`가 JSON과 schema를 검증합니다.
3. parse 실패면 generic `invalid_event`를 보내고 종료합니다.
4. parse 성공이면 event type에 맞는 command/repository operation을 실행합니다.
5. processing 실패면 exception 원문을 버리고 generic `internal_error`를 보냅니다.
6. 성공한 command만 정상 broadcast/state transition으로 이어집니다.

## 11. 학습 완료 확인

- [x] 모든 Commit map SHA의 historical code를 확인했습니다.
- [x] parent/직전 관련 상태와 현재 SHA를 구분했습니다.
- [x] fix의 이전 가정·root cause·corrected invariant를 연결했습니다.
- [x] test가 통과하는 production path와 비증명 범위를 구분했습니다.
- [x] ownership/lifetime/cleanup과 failure path를 기록했습니다.
- [x] 실행 evidence와 code inspection을 구분했습니다.
- [x] 마지막 SHA 기준 최종 flow와 non-guarantee를 설명할 수 있습니다.
===== END FILE: 04-websocket-internal-error-containment.md =====

===== BEGIN FILE: 05-runtime-mode-cors-and-network-trust.md =====
# Runtime mode·CORS·network trust

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

초기 local environment default와 browser CORS 설정에서 출발해 development/test/production/demo mode, 강한 secret, explicit proxy trust, invalid mode fail-closed, production durable-storage requirement를 하나의 검증된 환경 parser로 수렴시키는 과정을 복원합니다.

이 문서는 Phase 1 category audit 후 동결된 scaffold를 기준으로 exact SHA의 구현 발전을 복원합니다.

### 직접 연결되는 불변식

- 환경별 capability는 `readAppMode`가 반환한 단일 validated mode에서 파생됩니다.
- explicit mode 오타는 development로 downgrade되지 않고 startup error가 됩니다.
- production은 강한 session secret과 `DATABASE_URL` 없이는 환경 parsing을 통과하지 못합니다.
- demo는 의도적으로 memory storage를 허용하며 proxy address parsing은 `TRUST_PROXY=1`일 때만 활성화됩니다.
- credentialed browser mutation에 필요한 origin, method, header는 CORS configuration에 명시됩니다.

## 2. 핵심 질문

- 초기 `readEnv` default와 mode-aware fail-closed parser 사이에 어떤 rule이 추가됩니까?
- explicit APP_MODE와 NODE_ENV fallback의 우선순위는 무엇이며 invalid explicit value를 왜 거부합니까?
- production과 demo의 persistence capability가 어떤 조건에서 갈라집니까?
- CORS method/header, `credentials: true`, proxy trust는 각각 다른 신뢰 결정을 어떻게 표현합니까?
- 이 Thread의 unit tests가 process startup·DB readiness·real browser를 증명하지 않는 이유는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 지정 브랜치 ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 변경 전후 상태를 구분합니다.
- 파일, symbol, caller/callee, 상태 mutation, failure branch, cleanup을 actual historical code로 기록합니다.
- Fix는 이전 가정과 root cause를, test는 production path와 증명/비증명 범위를 연결합니다.
- S/A/B/C 중요도에 맞춰 설명 깊이를 다르게 유지합니다.
- 실행하지 않은 command 결과를 작성하지 않으며 code inspection과 runtime evidence를 구분합니다.
- 마지막 SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85ac2a949439` | `test(api): 실행 환경 기본값 검증` | B | PROTOCOL, PERSISTENCE, TEST | 명시적 환경 값과 local prototype default를 `readEnv` unit test로 고정합니다. |
| 2 | `66155cf8a27d` | `fix(api): 변경 요청용 CORS method와 header 허용` | B | AUTH, WEB | credentialed browser mutation에 필요한 method와 request header를 Fastify CORS 설정에 명시합니다. |
| 3 | `f801ccd09cf0` | `feat(guest): guest runtime 환경 경계 구성` | A | AUTH, WEB | `APP_MODE`, explicit proxy trust, 강한 session secret, browser-facing runtime URL을 환경 contract에 추가합니다. |
| 4 | `2b274686e6d4` | `fix(guest): 체험 환경의 runtime 복구 제한` | A | AUTH, REALTIME, RISK | 중복 runtime-mode parser를 제거하고 explicit `APP_MODE` 네 값을 허용하며 잘못된 값을 startup error로 처리합니다. |
| 5 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | TOURNAMENT, OPERATIONS, RISK | production mode에서 `DATABASE_URL`이 없으면 memory repository로 fallback하지 않고 startup 전에 실패시킵니다. |
| 6 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | OPERATIONS, TEST | explicit `APP_MODE=production`과 inferred `NODE_ENV=production` 모두 DB URL 없이는 실패하고 demo만 null DB를 허용하는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `test(api): 실행 환경 기본값 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `85ac2a949439` |
| Importance | B |
| Tags | PROTOCOL, PERSISTENCE, TEST |
| Source에서 확정된 역할 | 명시적 환경 값과 local prototype default를 `readEnv` unit test로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.test.ts`에서 explicit port/DB/origin과 empty input default assertion을 확인합니다.
- input object를 직접 주입해 process-global environment mutation 없이 parser를 검사하는지 확인합니다.
- 이 시점에 mode/trust/strong-secret/production DB rule이 아직 없다는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | bootstrap이 환경 값을 읽었지만 default와 explicit override가 자동 검증되지 않았습니다. |
| 해결하려던 문제 | 환경 parser refactor가 local port, memory fallback, web origin을 조용히 바꿀 수 있었습니다. |
| 핵심 결정 | `readEnv`에 object를 직접 넘겨 명시 값과 빈 환경의 기본값을 unit test로 고정했습니다. |
| 입력 → 상태 전이 → 출력 | fixture object → `readEnv` → parsed port/databaseUrl/webOrigin assertion 순입니다. |
| ownership/lifetime/cleanup | 함수 호출이 반환 object만 만들며 외부 resource나 cleanup은 없습니다. |
| failure/rollback/retry | 잘못된 numeric port나 secret은 이 시점 test하지 않습니다. |
| 보장하는 것 | local default 4000, null DB, localhost origin과 explicit override를 증명합니다. |
| 보장하지 않는 것 | production safety, runtime mode, proxy trust는 증명하지 않습니다. |
| 후속 연결 | `f801ccd09cf0` 이후 test가 mode·secret·trust rule로 확장됩니다. |

#### 최소 코드 근거

`apps/api/src/env.test.ts`의 두 초기 test가 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 환경 parser가 explicit value를 우선하고 정해진 local default를 사용해야 합니다. |
| 재현하는 failure/boundary | 명시된 port/DB/origin과 빈 environment object입니다. |
| test technique | pure unit test with injected environment object |
| 통과하는 production path | input object → `readEnv` → `ApiEnv` |
| 증명하는 것 | 초기 precedence와 local defaults를 증명합니다. |
| 증명하지 않는 것 | process startup, invalid values, production resource availability는 증명하지 않습니다. |
| 후속 회귀 방지 | 환경 parser refactor가 기본 실행 계약을 바꾸는 회귀를 방지합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `66155cf8a27d` — `fix(api): 변경 요청용 CORS method와 header 허용`

### 5.2. `fix(api): 변경 요청용 CORS method와 header 허용`

| 항목 | 값 |
| --- | --- |
| SHA | `66155cf8a27d` |
| Importance | B |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | credentialed browser mutation에 필요한 method와 request header를 Fastify CORS 설정에 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent와 `apps/api/src/app.ts`의 CORS option을 비교합니다.
- origin allowlist, `credentials: true`, methods, allowedHeaders가 함께 설정되는지 확인합니다.
- PATCH/DELETE preflight와 content-type/authorization header가 이전 default에서 막힐 수 있던 조건을 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | CORS는 origin과 credentials만 설정해 mutation method/header 허용을 plugin default에 맡겼습니다. |
| 해결하려던 문제 | browser preflight가 PATCH/DELETE 또는 JSON/authorization header를 허용받지 못해 route가 존재해도 호출 전에 차단될 수 있었습니다. |
| 핵심 결정 | GET, POST, PATCH, DELETE, OPTIONS와 content-type, authorization header를 명시했습니다. |
| 입력 → 상태 전이 → 출력 | browser preflight가 app CORS plugin에 도착하면 allowlist origin과 method/header 조건으로 response header가 생성됩니다. |
| ownership/lifetime/cleanup | Fastify CORS plugin 설정이 browser cross-origin admission rule을 소유합니다. |
| failure/rollback/retry | origin 배열은 고정 값과 configured origin을 포함하며 wildcard를 쓰지 않습니다. 실제 browser test는 이 commit에 없습니다. |
| 보장하는 것 | 정의된 origin에서 credentialed mutation에 필요한 method/header를 허용합니다. |
| 보장하지 않는 것 | CSRF 방어, cookie SameSite, arbitrary custom header, proxy trust는 보장하지 않습니다. |
| 후속 연결 | `f801ccd09cf0`이 mode/trust를 환경 contract에 추가하고 later HTTP boundary가 request-id header를 확장합니다. |

#### 최소 코드 근거

`apps/api/src/app.ts`의 CORS `methods`와 `allowedHeaders` diff가 근거입니다.

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | origin과 credentials만 지정해도 필요한 mutation preflight가 모두 허용될 것이라 가정했습니다. |
| 실제 실패 또는 위험 | PATCH/DELETE와 JSON/authorization header를 쓰는 browser request가 preflight에서 거부될 수 있었습니다. |
| Root cause | 허용 method/header가 application contract에 명시되지 않았습니다. |
| 수정된 invariant | browser가 실제 사용하는 method/header를 CORS allowlist에 명시해야 합니다. |
| 변경 코드 | `apps/api/src/app.ts` CORS registration options입니다. |
| Regression evidence | 이 commit에는 독립 browser regression test가 없으며 later API/browser suites가 통합 경로를 간접 검증합니다. |

비교 기준:
- 직전 관련 SHA: `85ac2a949439` — `test(api): 실행 환경 기본값 검증`
- 다음 관련 SHA: `f801ccd09cf0` — `feat(guest): guest runtime 환경 경계 구성`

### 5.3. `feat(guest): guest runtime 환경 경계 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f801ccd09cf0` |
| Importance | A |
| Tags | AUTH, WEB |
| Source에서 확정된 역할 | `APP_MODE`, explicit proxy trust, 강한 session secret, browser-facing runtime URL을 환경 contract에 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.env.example`과 `apps/api/src/env.ts`에서 APP_MODE, TRUST_PROXY, API/web URL을 확인합니다.
- demo/production에서 UTF-8 32-byte 이상 secret을 요구하는 분기를 확인합니다.
- 초기 `readAppMode`가 explicit demo만 특별 취급하고 NODE_ENV production/test에 fallback하는 제한을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 환경 parser는 port/DB/origin/secret만 반환하고 development/test/production/demo capability와 proxy trust를 구분하지 않았습니다. |
| 해결하려던 문제 | guest/demo와 production이 약한 default secret이나 forwarded header 신뢰를 암묵적으로 사용할 위험이 있었습니다. |
| 핵심 결정 | `ApiEnv`에 appMode와 trustProxy를 추가하고 demo/production에서 명시적 32-byte secret을 요구했으며 `.env.example`에 server/browser 값을 문서화했습니다. |
| 입력 → 상태 전이 → 출력 | `readEnv`가 mode를 먼저 결정하고 secret guard를 통과한 뒤 URL·trust 값을 반환합니다. `TRUST_PROXY`는 정확히 `"1"`일 때만 true입니다. |
| ownership/lifetime/cleanup | 환경 parser가 capability source를 소유하고 composition/app builder가 반환 값을 소비합니다. 생성되는 장기 resource는 없습니다. |
| failure/rollback/retry | unknown explicit `APP_MODE`는 이 SHA에서 development로 떨어질 수 있고 production DB 부재도 허용합니다. |
| 보장하는 것 | demo/production의 약한 secret을 startup 전에 거부하고 proxy trust를 opt-in으로 만듭니다. |
| 보장하지 않는 것 | 모든 explicit mode validation, single parser ownership, production persistence requirement는 아직 없습니다. |
| 후속 연결 | `2b274686e6d4`가 mode parser를 중앙화·fail-closed하고 `eb675ef74af3`가 DB requirement를 추가합니다. |

#### 최소 코드 근거

`f801ccd09cf0`, `apps/api/src/env.ts`의 핵심 guard:

```ts
if ((appMode === "demo" || appMode === "production") &&
    (!configuredSecret || Buffer.byteLength(configuredSecret, "utf8") < 32)) {
  throw new Error("SESSION_SECRET must be at least 32 bytes...");
}
```

비교 기준:
- 직전 관련 SHA: `66155cf8a27d` — `fix(api): 변경 요청용 CORS method와 header 허용`
- 다음 관련 SHA: `2b274686e6d4` — `fix(guest): 체험 환경의 runtime 복구 제한`

### 5.4. `fix(guest): 체험 환경의 runtime 복구 제한`

| 항목 | 값 |
| --- | --- |
| SHA | `2b274686e6d4` |
| Importance | A |
| Tags | AUTH, REALTIME, RISK |
| Source에서 확정된 역할 | 중복 runtime-mode parser를 제거하고 explicit `APP_MODE` 네 값을 허용하며 잘못된 값을 startup error로 처리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.ts::readAppMode`와 `app.ts`에서 local parser가 제거된 diff를 확인합니다.
- APP_MODE가 있으면 네 값만 통과하고, 없을 때만 NODE_ENV production/test에 fallback하는 우선순위를 확인합니다.
- 같은 commit의 guest rolling-window/ticket 제한은 이 Thread의 주 역할이 아니며 별도 guest category로 교차 참조함을 명시합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | env와 app에 mode 결정 로직이 중복되고 explicit `APP_MODE`는 demo 외 값을 제대로 표현하지 못했습니다. |
| 해결하려던 문제 | 오타나 unsupported mode가 development로 조용히 복구되면 secure cookie, route exposure, guest capability가 잘못 파생될 수 있었습니다. |
| 핵심 결정 | `readAppMode`를 export한 단일 함수로 만들고 explicit development/test/production/demo만 허용하며 그 외 값은 throw했습니다. |
| 입력 → 상태 전이 → 출력 | `APP_MODE` 존재 → allowlist 검사 → 반환/throw, 부재 → NODE_ENV production/test → development default 순입니다. app은 같은 함수를 import합니다. |
| ownership/lifetime/cleanup | env module이 runtime mode 해석을 단독 소유합니다. app은 결과로 capability를 파생합니다. |
| failure/rollback/retry | 잘못된 값은 startup 전에 예외로 중단됩니다. fallback recovery는 의도적으로 하지 않습니다. |
| 보장하는 것 | explicit mode가 오타로 약한 환경으로 downgrade되지 않고 모든 소비자가 같은 parser를 사용할 수 있습니다. |
| 보장하지 않는 것 | 이 commit에 함께 포함된 guest rate/ticket resource limit의 전체 correctness는 이 category가 보장하지 않습니다. |
| 후속 연결 | `eb675ef74af3`이 production mode의 storage capability를 fail-closed로 연결합니다. |

#### 최소 코드 근거

`2b274686e6d4`, `apps/api/src/env.ts`:

```ts
if (input.APP_MODE !== undefined) {
  if (["development", "test", "production", "demo"]
      .includes(input.APP_MODE)) return input.APP_MODE as ApiEnv["appMode"];
  throw new Error(`APP_MODE must be ...`);
}
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | unknown explicit mode도 development default로 복구해도 된다고 가정했고 env/app가 각자 mode를 해석했습니다. |
| 실제 실패 또는 위험 | 오타가 security capability를 낮추고 두 parser가 서로 다른 mode를 선택할 위험이 있었습니다. |
| Root cause | explicit-value allowlist와 single owner가 없었습니다. |
| 수정된 invariant | 명시된 mode는 유효할 때만 사용하고 잘못되면 startup을 중단하며 mode parsing은 한 함수가 소유해야 합니다. |
| 변경 코드 | `apps/api/src/env.ts::readAppMode`, `apps/api/src/app.ts` local parser 삭제입니다. |
| Regression evidence | 같은 SHA의 `env.test.ts`가 strong secret과 trust switch를 검사하고 later startup/config tests가 mode failure를 보호합니다. |

비교 기준:
- 직전 관련 SHA: `f801ccd09cf0` — `feat(guest): guest runtime 환경 경계 구성`
- 다음 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

### 5.5. `fix(config): production에서 영속 저장소 요구`

| 항목 | 값 |
| --- | --- |
| SHA | `eb675ef74af3` |
| Importance | A |
| Tags | TOURNAMENT, OPERATIONS, RISK |
| Source에서 확정된 역할 | production mode에서 `DATABASE_URL`이 없으면 memory repository로 fallback하지 않고 startup 전에 실패시킵니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.ts`에서 database URL을 먼저 local 변수로 만들고 production guard를 적용하는 위치를 확인합니다.
- demo mode는 null DB를 계속 허용하는지 확인합니다.
- composition root가 이 parser 결과를 사용하므로 guard가 repository factory 선택 전에 실행되는지 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | composition root는 DB URL이 없으면 모든 mode에서 memory repository를 선택했고 strong secret만 production을 구분했습니다. |
| 해결하려던 문제 | production이 정상 시작된 것처럼 보이면서 모든 사용자·경기 상태가 process 종료와 함께 사라질 수 있었습니다. |
| 핵심 결정 | `readEnv`가 production mode와 null `DATABASE_URL` 조합을 즉시 거부하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | mode/secret parse → database URL 읽기 → production/null guard → 유효 `ApiEnv` 반환 → composition root factory 선택 순입니다. |
| ownership/lifetime/cleanup | env parser가 production storage capability 검증을 소유하고 repository factory는 이미 검증된 URL/null을 소비합니다. |
| failure/rollback/retry | guard failure는 startup 전에 throw합니다. DB 연결 가능성·migration readiness는 이 guard가 확인하지 않습니다. |
| 보장하는 것 | production은 memory fallback으로 시작할 수 없고 demo는 의도적으로 transient storage를 유지할 수 있습니다. |
| 보장하지 않는 것 | URL이 실제 DB에 연결되는지, schema가 current인지, data durability가 보장되는지는 별도 startup/readiness 책임입니다. |
| 후속 연결 | `4633dfde208d`가 explicit/inferred production과 demo 예외를 unit test로 고정합니다. |

#### 최소 코드 근거

`eb675ef74af3`, `apps/api/src/env.ts`:

```ts
const databaseUrl = input.DATABASE_URL ?? null;
if (appMode === "production" && !databaseUrl) {
  throw new Error("DATABASE_URL is required in production mode");
}
```

#### Fix 연결

| 단계 | 기록 |
| --- | --- |
| 이전 가정 | DB URL이 없으면 mode와 무관하게 memory repository로 실행해도 된다고 가정했습니다. |
| 실제 실패 또는 위험 | production state가 비영속 memory에 저장되어 process restart에서 사라질 수 있었습니다. |
| Root cause | runtime mode와 repository capability 사이 startup validation이 없었습니다. |
| 수정된 invariant | production은 durable repository URL 없이는 시작하지 않아야 하며 demo만 transient fallback을 허용할 수 있습니다. |
| 변경 코드 | `apps/api/src/env.ts::readEnv` production database guard입니다. |
| Regression evidence | `4633dfde208d`의 explicit/inferred production test입니다. |

비교 기준:
- 직전 관련 SHA: `2b274686e6d4` — `fix(guest): 체험 환경의 runtime 복구 제한`
- 다음 관련 SHA: `4633dfde208d` — `test(config): production memory fallback 거부 검증`

### 5.6. `test(config): production memory fallback 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4633dfde208d` |
| Importance | A |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | explicit `APP_MODE=production`과 inferred `NODE_ENV=production` 모두 DB URL 없이는 실패하고 demo만 null DB를 허용하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.test.ts`에서 strong secret을 제공해 secret guard가 아닌 DB guard만 격리하는지 확인합니다.
- explicit production과 NODE_ENV fallback production 두 경로를 각각 검사합니다.
- demo case가 null databaseUrl을 반환해 의도적 차이를 보호하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production DB guard가 추가됐지만 mode 결정 두 경로와 demo 예외가 함께 유지되는지 자동 증거가 없었습니다. |
| 해결하려던 문제 | 한 경로만 test하면 inferred production이 memory로 빠지거나 demo까지 잘못 금지하는 회귀를 놓칠 수 있었습니다. |
| 핵심 결정 | 동일한 strong secret으로 explicit/inferred production을 각각 호출해 `DATABASE_URL` error를 기대하고 demo의 null DB를 확인했습니다. |
| 입력 → 상태 전이 → 출력 | environment fixture → `readAppMode` → secret guard 통과 → DB guard throw/return 순을 직접 실행합니다. |
| ownership/lifetime/cleanup | pure function test로 외부 resource와 cleanup이 없습니다. |
| failure/rollback/retry | 실제 DB 연결·process exit code·repository factory는 실행하지 않습니다. |
| 보장하는 것 | 두 production mode source가 모두 memory fallback을 거부하고 demo는 허용함을 결정적으로 증명합니다. |
| 보장하지 않는 것 | DATABASE_URL 유효성, connection readiness, migration 상태는 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 production/demo storage capability 차이를 회귀 방지합니다. |

#### 최소 코드 근거

`apps/api/src/env.test.ts`의 세 environment object assertion이 근거입니다.

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | production은 mode 결정 방식과 무관하게 durable DB URL 없이는 환경 parsing에 실패해야 하고 demo만 null DB를 허용해야 합니다. |
| 재현하는 failure/boundary | explicit APP_MODE production, inferred NODE_ENV production, APP_MODE demo의 세 경계입니다. |
| test technique | pure configuration unit/boundary test |
| 통과하는 production path | environment object → `readEnv` → mode/secret/database guards |
| 증명하는 것 | production memory fallback 거부와 demo 예외를 정확히 증명합니다. |
| 증명하지 않는 것 | 실제 process startup, DB connection, migration/readiness는 증명하지 않습니다. |
| 후속 회귀 방지 | 환경 parser refactor가 production을 transient storage로 downgrade하는 회귀를 막습니다. |

비교 기준:
- 직전 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

## 6. Invariant evolution

| 단계 | 관련 SHA | 기록 |
| --- | --- | --- |
| Local defaults | `85ac2a949439` | port, DB null, web origin과 explicit override를 unit test로 고정합니다. |
| Browser mutation allowlist | `66155cf8a27d` | CORS methods와 headers를 명시합니다. |
| Mode/secret/trust introduction | `f801ccd09cf0` | appMode, strong secret, explicit proxy trust를 환경 contract에 추가합니다. |
| Single fail-closed mode owner | `2b274686e6d4` | 중복 parser를 제거하고 invalid explicit mode를 거부합니다. |
| Production storage capability | `eb675ef74af3` | production의 memory fallback을 startup 전에 금지합니다. |
| Boundary verification | `4633dfde208d` | explicit/inferred production과 demo 차이를 unit test로 고정합니다. |

## 7. Failure → Fix → Test 관계

| Failure 또는 불충분한 상태 | Fix/구현 SHA | Test 또는 후속 증거 |
| --- | --- | --- |
| CORS default가 mutation preflight와 불일치 | `66155cf8a27d` | 실제 method/header allowlist를 추가합니다. |
| Mode parser 중복과 unknown value fallback | `2b274686e6d4` | single exported parser와 explicit allowlist/throw를 도입합니다. |
| Production이 memory로 정상 시작 가능 | `eb675ef74af3` → `4633dfde208d` | DB URL guard와 두 production source regression test를 추가합니다. |

## 8. Ownership·state·responsibility 변화

| Owner | 최종 책임 | 명시적 비책임 |
| --- | --- | --- |
| `readEnv`/`readAppMode` | 환경 값 parsing, mode/secret/storage/trust validation | DB connection readiness를 소유하지 않습니다. |
| Fastify CORS plugin | origin/method/header/credentialed preflight response | CSRF·cookie browser enforcement를 소유하지 않습니다. |
| Composition root | 검증된 env로 repository/app option 선택 | mode 문자열 해석을 중복하지 않습니다. |
| Configuration tests | pure environment fixture와 guard 결과 | 실제 process/browser/DB를 실행하지 않습니다. |

## 9. Thread 최종 상태

- 최종 SHA 기준으로 mode는 한 parser에서 결정되고 explicit 오타는 거부됩니다.
- demo/production은 강한 secret을 요구하며 production만 durable DB URL을 반드시 요구합니다.
- proxy trust는 명시적으로 opt-in이고 browser mutation용 CORS method/header가 선언됩니다.
- 실제 DB 연결·migration readiness·browser preflight 실행은 이 Thread의 code inspection과 unit tests만으로 증명되지 않습니다.

### 실행 검증 기록

- Source inspection: GitHub connector로 Commit map의 exact SHA diff와 필요한 historical file을 조회했습니다.
- Branch validation: branch-local `commit/commit-importance.md`가 선언한 433개 선형 이력에서 모든 참조 SHA와 source classification을 대조했고, 가장 이른 참조 `b625c4f9dfdc`는 branch HEAD 비교에서 merge base가 동일 SHA임을 확인했습니다.
- 실제 실행 시도: `git clone --branch web/ft_transcendence --single-branch https://github.com/seungwoo7050/42-archive.git /tmp/ft-transcendence-audit`
- 결과: exit status 128, `Could not resolve host: github.com`. 따라서 repository checkout, package install, test command는 실행하지 않았으며 runtime 결과를 주장하지 않습니다.

## 10. 최종 실행 흐름

1. environment object에서 explicit APP_MODE를 우선 검사합니다.
2. 없으면 NODE_ENV production/test를 해석하고 그 외는 development를 사용합니다.
3. demo/production이면 session secret byte length를 검사합니다.
4. production이면 DATABASE_URL 존재를 검사합니다.
5. TRUST_PROXY가 정확히 `1`인지 읽어 env object를 반환합니다.
6. composition root가 검증된 mode/DB/trust 값을 repository와 Fastify option에 전달합니다.
7. browser request는 configured CORS origin/method/header 규칙을 통과한 뒤 route에 도달합니다.

## 11. 학습 완료 확인

- [x] 모든 Commit map SHA의 historical code를 확인했습니다.
- [x] parent/직전 관련 상태와 현재 SHA를 구분했습니다.
- [x] fix의 이전 가정·root cause·corrected invariant를 연결했습니다.
- [x] test가 통과하는 production path와 비증명 범위를 구분했습니다.
- [x] ownership/lifetime/cleanup과 failure path를 기록했습니다.
- [x] 실행 evidence와 code inspection을 구분했습니다.
- [x] 마지막 SHA 기준 최종 flow와 non-guarantee를 설명할 수 있습니다.
===== END FILE: 05-runtime-mode-cors-and-network-trust.md =====

===== BEGIN FILE: README.md =====
# 애플리케이션 기반과 API 경계

모노레포 package ownership, shared executable contract, Fastify resource API, typed failure, strict JSON request validation, WebSocket client-facing error containment, runtime mode와 CORS 같은 애플리케이션 기반을 다룹니다.

## 범위

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- 상태: Phase 1 category audit 완료·동결
- 포함: workspace/package ownership, shared contract, DB import/lifecycle, API composition, resource HTTP boundary, strict request parsing, client-facing error redaction, mode/CORS/proxy trust
- 제외: repository domain integrity·migration evolution, session/logout·role·account lifecycle, core WebSocket admission/room/simulation, browser state architecture, readiness/metrics/drain, production delivery artifact

## Phase 1 audit 결과

- 기존 4개 Thread를 실제 역사와 category boundary에 맞춰 5개로 재구성했습니다.
- workspace Thread는 shared DTO/game contract가 DB/API/web package보다 먼저 도입된 실제 순서로 정렬했습니다.
- DB package의 import 가능 경계 `1140fb868714`과 repository lifecycle `9277572765e7`을 composition story에 추가했습니다.
- Resource API Thread는 초기 route 구현과 baseline integration tests가 runtime schema보다 먼저 존재한 실제 순서로 정렬했습니다.
- 초기 API 통합 증거 `fb1c287d9e79`, `1395d45a3665`, `5088099d1e7d`와 typed boundary 후속 test `50caaf5c7c49`을 추가했습니다.
- HTTP strict request validation과 WebSocket internal-error redaction은 변경 파일·실패 원인·검증 방식이 독립적이므로 별도 Thread로 분리했습니다.
- `2b274686e6d4`는 이 카테고리에서 runtime-mode single owner와 invalid-value fail-closed만 다루며 guest resource-limit 변경은 별도 category 책임으로 남깁니다.

## Frozen Thread

1. [Workspace·package·composition 경계](01-workspace-package-and-composition-boundaries.md)
2. [Resource API와 실행 가능한 HTTP contract](02-executable-http-contracts-and-resource-api.md)
3. [Strict JSON request validation](03-strict-json-request-validation.md)
4. [WebSocket client error boundary와 내부 오류 격리](04-websocket-internal-error-containment.md)
5. [Runtime mode·CORS·network trust](05-runtime-mode-cors-and-network-trust.md)

## 사용 원칙

- 각 문서의 Commit map은 해당 Thread 안에서 실제 commit 시간 순서를 따릅니다.
- exact SHA의 diff와 해당 SHA 파일을 기준으로 설명하며 final HEAD를 과거 상태에 투영하지 않습니다.
- 같은 SHA가 다른 category에 있어도 이 문서는 위 category boundary에 해당하는 역할만 다룹니다.
- Phase 2 completed 문서는 이 frozen scaffold의 filename, structure, SHA, subject, importance, tags, role을 그대로 보존합니다.
===== END FILE: README.md =====

