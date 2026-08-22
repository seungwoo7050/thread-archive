===== BEGIN FILE: 01-startup-liveness-readiness-and-storage-state.md =====
# Startup·liveness·readiness·storage state

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

service bootstrap과 환경 기본값에서 시작해 migration set 및 repository health를 readiness로 노출하고 startup data mutation과 production memory fallback을 제거하는 과정을 복원합니다.

### 직접 연결되는 불변식

- liveness는 process 상태를, readiness는 storage/migration/lifecycle 상태를 반영합니다.
- migration divergence와 database failure는 readiness를 fail-closed 상태로 만듭니다.
- application startup은 암묵적으로 schema/seed data를 변경하지 않습니다.
- production은 durable storage 없이 시작하지 않습니다.

## 2. 핵심 질문

- process가 살아 있는 것과 traffic을 받을 준비가 된 것은 어떤 endpoint/조건으로 분리됩니까?
- repository readiness가 connection failure, pending migration, diverged migration을 어떻게 표현합니까?
- startup이 seed/migration을 암묵 실행하지 않을 때 운영자가 사전에 수행해야 하는 명시적 단계는 무엇입니까?
- production persistent-store requirement와 demo memory mode가 어떤 configuration branch로 분리됩니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4b43a284e637` | `feat(api): 실행 환경과 service bootstrap 구성` | B | PERSISTENCE, OPERATIONS | runtime env, repository, Fastify app, close lifecycle을 composition root에서 조립합니다. |
| 2 | `85ac2a949439` | `test(api): 실행 환경 기본값 검증` | B | PROTOCOL, PERSISTENCE, TEST | configuration precedence와 local startup default를 고정합니다. |
| 3 | `30aac132e14e` | `feat(db): migration set 상태 검사 추가` | A | PERSISTENCE, OPERATIONS, RISK | bundle/applied migration set을 current, pending, diverged로 분류합니다. |
| 4 | `2f05d5d79c64` | `feat(db): repository readiness 경계 추가` | A | PERSISTENCE, OPERATIONS | database availability와 migration status를 repository contract로 노출합니다. |
| 5 | `15002e229acb` | `feat(ops): liveness와 readiness endpoint 추가` | A | PROTOCOL, PERSISTENCE, OPERATIONS | process liveness와 dependency-aware service readiness를 versioned response로 분리합니다. |
| 6 | `6937cf60aeea` | `test(ops): health와 database readiness 검증` | B | AUTH, PERSISTENCE, OPERATIONS | DB down/migration state와 무관하게 liveness가 유지되고 readiness만 실패하는지 검증합니다. |
| 7 | `e1a0316fbe84` | `fix(api): startup seed 생성을 제거` | B | PERSISTENCE | API startup에서 implicit seed mutation을 제거합니다. |
| 8 | `5cac4843fd9b` | `test(api): startup seed 금지 검증` | B | REALTIME, TEST | entrypoint가 seed operation을 호출하지 않는지 검증합니다. |
| 9 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | TOURNAMENT, OPERATIONS, RISK | production에서 database URL이 없으면 environment parsing 단계에서 실패합니다. |
| 10 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | OPERATIONS, TEST | explicit/inferred production 모두 memory fallback을 거부하는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(api): 실행 환경과 service bootstrap 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `4b43a284e637` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | runtime env, repository, Fastify app, close lifecycle을 composition root에서 조립합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.ts`, `apps/api/src/index.ts`, `apps/api/src/app.ts`
- 핵심 symbol: `readEnv`, `buildApp`, API entrypoint의 repository 선택과 `onClose` hook
- `readEnv`가 누락된 환경값을 어떤 로컬 기본값으로 바꾸고 숫자·URL·mode를 어디서 검증하는지 확인합니다.
- `index.ts`가 `DATABASE_URL` 유무로 PostgreSQL/memory repository를 선택하고 `buildApp`에 넘기는 순서를 추적합니다.
- 당시에는 startup에서 `ensureSeedData`를 호출한다는 사실과, `app.close()`가 repository `close()`로 이어지는 소유권을 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:4b43a284e637:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | API 패키지는 있었지만 실행 시 환경값, 저장소 구현, Fastify 인스턴스와 종료 처리를 한 곳에서 조립하는 명시적 entrypoint가 없었습니다. 개발 서버를 실제 process로 띄우려면 환경 해석과 저장소 선택, HTTP 서버 시작, 종료 시 자원 반환 순서를 하나의 composition root가 소유해야 했습니다. |
| 구현 또는 검증 결정 | `readEnv`로 런타임 설정을 구성하고, `DATABASE_URL`이 있으면 PostgreSQL 저장소를, 없으면 memory 저장소를 만든 뒤 `buildApp`에 주입합니다. 이 SHA에서는 저장소 생성 뒤 `ensureSeedData`를 호출하므로 startup이 데이터 변경까지 수행합니다. |
| 실행/검증 경로 | process 시작 → 환경 해석 → repository 생성·seed → Fastify app 생성 → `listen` → 종료 시 `onClose`에서 repository `close` 순서입니다. |
| ownership과 failure 처리 | entrypoint가 repository의 생성과 lifetime을 소유하고 Fastify `onClose` hook에 해제를 연결합니다. app 내부 route는 구현체가 아니라 주입된 `AppRepository`만 사용합니다. 환경 파싱 또는 listen이 실패하면 startup이 끝나지 않습니다. 아직 storage health를 traffic admission과 분리하는 readiness 경계는 없습니다. |
| 보장하는 것 | 로컬에서 실행 가능한 API bootstrap과 한 번의 repository cleanup 경로가 생깁니다. |
| 보장하지 않는 것 | startup seed mutation, production memory fallback, migration 상태 검사는 아직 허용되거나 정의되지 않았습니다. |
| 후속 연결 | `85ac2a949439`가 설정 기본값을 고정하고, `e1a0316fbe84`가 여기의 implicit seed 호출을 나중에 제거합니다. |
<!-- LEARNER-END:4b43a284e637:record -->




#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `85ac2a949439` — `test(api): 실행 환경 기본값 검증`

### 5.2. `test(api): 실행 환경 기본값 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `85ac2a949439` |
| Importance | B |
| Tags | PROTOCOL, PERSISTENCE, TEST |
| Source에서 확정된 역할 | configuration precedence와 local startup default를 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.test.ts`
- 핵심 symbol: `readEnv`의 기본값·환경값 우선순위·유효성 검사
- 빈 환경 객체와 명시적 환경 객체를 각각 `readEnv`에 넣어 어떤 필드가 기본값/override가 되는지 확인합니다.
- port 같은 숫자 값과 mode/url 값의 negative case가 실제 parser에서 거부되는지 테스트 표와 연결합니다.
- 이 테스트는 process를 시작하지 않고 순수 설정 해석만 검증한다는 한계를 구분합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:85ac2a949439:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | bootstrap이 환경값을 읽기 시작했지만 기본값이나 override 규칙이 바뀌어도 이를 탐지할 자동화된 계약이 없었습니다. 로컬 startup과 배포 설정이 같은 parser를 쓰므로, 누락 값과 명시 값의 우선순위가 흔들리면 다른 저장소·origin·port로 실행될 수 있었습니다. |
| 구현 또는 검증 결정 | `readEnv`에 통제된 환경 객체를 주입해 기본값과 명시 값이 예상 필드로 변환되는지 검증합니다. |
| 실행/검증 경로 | test environment 구성 → `readEnv` 호출 → 반환된 configuration의 각 필드 비교입니다. |
| ownership과 failure 처리 | 테스트가 process-global `process.env` 대신 입력 객체를 소유하므로 케이스 간 상태 오염을 피합니다. 잘못된 입력을 거부하는 parser branch는 확인하지만 실제 OS 환경, socket bind, database 연결 실패는 실행하지 않습니다. |
| 보장하는 것 | 설정 기본값과 override 규칙의 회귀를 단위 수준에서 탐지합니다. |
| 보장하지 않는 것 | 설정이 실제 서비스 의존성을 사용할 수 있는지는 증명하지 않습니다. |
| 후속 연결 | 뒤의 `eb675ef74af3`와 `4633dfde208d`가 같은 설정 경계에 production 전용 fail-closed 규칙을 추가합니다. |
<!-- LEARNER-END:85ac2a949439:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:85ac2a949439:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 configuration unit test |
| 주입·재현 방식 | 외부 process나 네트워크 없이 명시적 환경 객체를 입력하고 `readEnv` 반환값과 예외 branch를 직접 비교합니다. |
| 증명하는 것 | parser의 기본값·우선순위·입력 검증이 고정됩니다. |
| 증명하지 않는 것 | 실제 환경 변수 주입, port bind, PostgreSQL 연결 성공까지는 증명하지 않습니다. |
<!-- LEARNER-END:85ac2a949439:test -->



#### 비교 기준

- 직전 관련 SHA: `4b43a284e637` — `feat(api): 실행 환경과 service bootstrap 구성`
- 다음 관련 SHA: `30aac132e14e` — `feat(db): migration set 상태 검사 추가`

### 5.3. `feat(db): migration set 상태 검사 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `30aac132e14e` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS, RISK |
| Source에서 확정된 역할 | bundle/applied migration set을 current, pending, diverged로 분류합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/migrator.ts`
- 핵심 symbol: `findMigrationFiles`, `compareMigrationSets`, `inspectMigrationSet`
- bundle 디렉터리의 SQL 파일명과 `kysely_migration.name` 조회 결과를 어떤 정규화 규칙으로 비교하는지 확인합니다.
- `current`, `pending`, `diverged`를 결정하는 missing/unexpected 집합 계산을 실제 조건문으로 기록합니다.
- PostgreSQL 오류 코드 `42P01`만 migration table 부재로 해석하고 다른 query 오류는 다시 던지는 fail-closed 분기를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:30aac132e14e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | migration runner는 SQL을 적용할 수 있었지만, 이미 실행 중인 서비스가 bundle과 database의 migration 집합이 일치하는지 읽기 전용으로 판정할 수 없었습니다. |
| 해결하려던 문제와 위험 | 단순 연결 성공만으로는 코드가 기대하는 schema인지 알 수 없습니다. 누락 migration과 database에만 존재하는 예상 밖 migration을 구분하지 않으면 호환되지 않는 schema에도 traffic을 받을 수 있습니다. |
| 핵심 구현 결정 | bundle SQL 이름과 적용된 Kysely migration 이름을 집합으로 비교합니다. 누락만 있으면 `pending`, 예상 밖 이름이 하나라도 있으면 `diverged`, 두 집합이 같으면 `current`로 반환합니다. migration table이 아직 없는 `42P01`은 applied set이 빈 상태로 처리합니다. |
| 입력 → 상태 전이 → 출력 | migration 파일 열거 → `select name from kysely_migration` → missing/unexpected 계산 → status와 차이 목록 반환입니다. |
| ownership/lifetime/cleanup | 함수는 schema를 변경하지 않고 파일 목록과 query 결과만 읽습니다. query client/pool의 lifetime은 호출자가 계속 소유합니다. |
| failure/rollback/retry | migration table 부재 이외의 SQL 오류를 정상 상태로 삼지 않고 전파합니다. 따라서 database 장애가 migration pending으로 위장되지 않습니다. |
| 보장하는 것 | bundle과 database migration 집합의 관계를 명시적으로 판정하고 divergence를 fail-closed 신호로 만들 수 있습니다. |
| 보장하지 않는 것 | migration을 자동 실행하거나 SQL 내용의 의미적 호환성을 증명하지 않습니다. 이름 집합만 비교합니다. |
| 후속 연결 | `2f05d5d79c64`가 이 판정을 repository readiness 결과로 노출하고 `6937cf60aeea`가 pending/current/diverged를 테스트합니다. |
<!-- LEARNER-END:30aac132e14e:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:30aac132e14e:snippet -->
- SHA: `30aac132e14e`
- 위치: `packages/db/src/migrator.ts`; `findMigrationFiles`, `compareMigrationSets`, `inspectMigrationSet`

```ts
if (unexpected.length > 0) return { status: "diverged", missing, unexpected };
if (missing.length > 0) return { status: "pending", missing, unexpected };
return { status: "current", missing, unexpected };
```
<!-- LEARNER-END:30aac132e14e:snippet -->

#### 비교 기준

- 직전 관련 SHA: `85ac2a949439` — `test(api): 실행 환경 기본값 검증`
- 다음 관련 SHA: `2f05d5d79c64` — `feat(db): repository readiness 경계 추가`

### 5.4. `feat(db): repository readiness 경계 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `2f05d5d79c64` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | database availability와 migration status를 repository contract로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/index.ts`
- 핵심 symbol: `AppRepository.checkReadiness`, `PostgresRepository.checkReadiness`, memory repository implementation
- `AppRepository`의 readiness 반환형과 PostgreSQL/memory 구현이 같은 계약을 어떻게 다르게 채우는지 확인합니다.
- PostgreSQL 구현에서 `select 1`과 `inspectMigrationSet` 호출 순서, 오류 전파, migration status 매핑을 추적합니다.
- memory 구현의 `migrations: not_applicable`가 `current`를 가장하지 않는 이유를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:2f05d5d79c64:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | migration 상태 판정 함수는 존재했지만 API가 저장소 구현을 열어 pool/query 세부사항을 알아야 health를 계산할 수 있었습니다. |
| 해결하려던 문제와 위험 | readiness가 API의 구현 추측에 의존하면 memory와 PostgreSQL이 다른 의미를 반환하거나 migration 검사 순서가 분산됩니다. |
| 핵심 구현 결정 | 공통 `AppRepository`에 `checkReadiness()`를 추가합니다. PostgreSQL 구현은 연결 query와 migration 집합 검사를 수행하고, memory 구현은 database `up`, migrations `not_applicable`을 반환합니다. |
| 입력 → 상태 전이 → 출력 | API 또는 운영 caller → `repo.checkReadiness()` → 구현별 dependency 검사 → `{database, migrations}` 반환입니다. |
| ownership/lifetime/cleanup | 저장소가 자신의 connection과 schema 상태를 판정하는 책임을 소유합니다. API는 결과만 소비하고 pool을 직접 만지지 않습니다. |
| failure/rollback/retry | PostgreSQL query 또는 migration 검사가 실패하면 Promise가 reject됩니다. 이 SHA 자체는 오류를 HTTP 상태로 변환하지 않습니다. |
| 보장하는 것 | storage 종류에 관계없이 동일한 readiness 호출 지점이 생기고 migration applicability를 명시적으로 표현합니다. |
| 보장하지 않는 것 | service lifecycle이나 traffic admission은 아직 포함하지 않으며 database 오류를 sanitize하는 HTTP 경계도 다음 commit에 남습니다. |
| 후속 연결 | `15002e229acb`가 이 contract를 `/health/ready`에 연결합니다. |
<!-- LEARNER-END:2f05d5d79c64:record -->




#### 비교 기준

- 직전 관련 SHA: `30aac132e14e` — `feat(db): migration set 상태 검사 추가`
- 다음 관련 SHA: `15002e229acb` — `feat(ops): liveness와 readiness endpoint 추가`

### 5.5. `feat(ops): liveness와 readiness endpoint 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `15002e229acb` |
| Importance | A |
| Tags | PROTOCOL, PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | process liveness와 dependency-aware service readiness를 versioned response로 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/shared/src/http.ts`, `apps/api/src/app.ts`
- 핵심 symbol: liveness/readiness response schema, `/health/live`, `/health/ready`, `repo.checkReadiness`
- shared response schema가 `status`, `service`, `checks.lifecycle/database/migrations`를 어떤 bounded enum으로 제한하는지 확인합니다.
- `/health/live`가 dependency query 없이 200을 반환하고 `/health/ready`만 repository를 호출하는 분리를 추적합니다.
- readiness 성공 조건과 catch branch의 503/down/unknown 응답, 원본 database 오류를 body에 넣지 않는 처리를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:15002e229acb:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 기존 `/health`는 process가 응답한다는 사실만 보여 주었고 database/migration 불일치에도 traffic을 받아야 하는지 판단할 수 없었습니다. |
| 해결하려던 문제와 위험 | orchestrator가 process restart와 traffic removal을 같은 신호로 처리하면 일시적 database 장애에 process가 재시작되거나, 반대로 schema가 준비되지 않은 process에 요청이 유입됩니다. |
| 핵심 구현 결정 | liveness와 readiness를 별도 route로 만듭니다. readiness는 database가 `up`이고 migration이 `current` 또는 `not_applicable`일 때만 200/`ready`를 반환하며, 그 외 또는 예외는 503/`not_ready`로 변환합니다. |
| 입력 → 상태 전이 → 출력 | HTTP request → Fastify route → live는 즉시 schema 응답, ready는 `repo.checkReadiness` → 조건 판정 → shared schema로 응답입니다. |
| ownership/lifetime/cleanup | process 생존 신호는 app이, storage 판정은 repository가 소유합니다. HTTP 경계가 내부 예외를 외부의 제한된 상태값으로 변환합니다. |
| failure/rollback/retry | repository reject는 `database: down`, `migrations: unknown`인 503으로 축약됩니다. credential이나 connection string은 응답에 포함되지 않습니다. |
| 보장하는 것 | liveness와 dependency-aware readiness가 서로 독립된 운영 신호가 됩니다. |
| 보장하지 않는 것 | 이 시점의 lifecycle 값은 항상 `accepting`이며 drain 상태는 `44ef3e07e1a5`에서 추가됩니다. |
| 후속 연결 | `6937cf60aeea`가 HTTP와 실제 PostgreSQL 상태 전환을 검증합니다. |
<!-- LEARNER-END:15002e229acb:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:15002e229acb:snippet -->
- SHA: `15002e229acb`
- 위치: `packages/shared/src/http.ts`; liveness/readiness response schema, `/health/live`, `/health/ready`, `repo.checkReadiness`

```ts
const ready = repository.database === "up"
  && (repository.migrations === "current"
    || repository.migrations === "not_applicable");
```
<!-- LEARNER-END:15002e229acb:snippet -->

#### 비교 기준

- 직전 관련 SHA: `2f05d5d79c64` — `feat(db): repository readiness 경계 추가`
- 다음 관련 SHA: `6937cf60aeea` — `test(ops): health와 database readiness 검증`

### 5.6. `test(ops): health와 database readiness 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `6937cf60aeea` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | DB down/migration state와 무관하게 liveness가 유지되고 readiness만 실패하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/health.test.ts`, `packages/db/src/postgres.integration.test.ts`, `packages/db/src/readiness.test.ts`
- 핵심 symbol: Fastify `inject`, memory readiness stub, isolated PostgreSQL database, `compareMigrationSets`
- legacy `/health`, `/health/live`, `/health/ready` 응답을 한 app instance에서 비교하는 route test를 확인합니다.
- `checkReadiness` rejection에 credential이 든 오류를 주입하고 503 body에 secret/URL이 없는지 확인합니다.
- migration 미적용 isolated database가 `pending`, migrate 후 `current`가 되는 실제 PostgreSQL integration path를 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:6937cf60aeea:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | health route와 repository 판정이 구현됐지만 process 신호 분리, secret 비노출, 실제 migration 전후 상태를 함께 고정하는 증거가 없었습니다. 단위 stub만으로는 실제 migration table과 bundle 비교가 맞는지, 통합 테스트만으로는 HTTP 오류 변환이 안전한지 각각 부족했습니다. |
| 구현 또는 검증 결정 | Fastify injection으로 route contract를, memory/stub으로 실패 응답을, isolated PostgreSQL schema로 pending→current 전환을 각각 검증합니다. |
| 실행/검증 경로 | test setup → app/repository 생성 → health request 또는 migration 실행 → status/body 비교 → reverse-order close입니다. |
| ownership과 failure 처리 | 테스트가 생성한 app, repository, isolated database를 teardown에서 회수합니다. 의도적으로 `checkReadiness`를 reject시켜 503 변환과 secret 비노출을 확인합니다. 실제 네트워크 단절을 주입하는 테스트는 아닙니다. |
| 보장하는 것 | liveness/readiness 분리, memory `not_applicable`, migration set 판정, HTTP 오류 축약이 회귀 테스트로 고정됩니다. |
| 보장하지 않는 것 | orchestrator probe 설정이나 장시간 장애 복구는 검증하지 않습니다. |
| 후속 연결 | 후속 drain 테스트가 readiness lifecycle에 `draining`을 추가하면서 이 계약을 확장합니다. |
<!-- LEARNER-END:6937cf60aeea:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:6937cf60aeea:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | route unit/integration + PostgreSQL integration |
| 주입·재현 방식 | Fastify `inject`, repository rejection stub, migration 전후 isolated PostgreSQL schema를 조합합니다. |
| 증명하는 것 | HTTP contract와 실제 migration state가 같은 readiness 의미로 연결됩니다. |
| 증명하지 않는 것 | 실제 process health probe, container restart, 외부 network partition은 증명하지 않습니다. |
<!-- LEARNER-END:6937cf60aeea:test -->



#### 비교 기준

- 직전 관련 SHA: `15002e229acb` — `feat(ops): liveness와 readiness endpoint 추가`
- 다음 관련 SHA: `e1a0316fbe84` — `fix(api): startup seed 생성을 제거`

### 5.7. `fix(api): startup seed 생성을 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `e1a0316fbe84` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | API startup에서 implicit seed mutation을 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/index.ts`
- 핵심 symbol: entrypoint의 `ensureSeedData` 호출 제거
- parent의 repository 생성 뒤 `ensureSeedData` 호출과 이 SHA의 제거 diff를 직접 비교합니다.
- migration/seed CLI가 남아 있는지와 API startup이 이제 read/configure/listen만 수행하는지 구분합니다.
- memory와 PostgreSQL 모두에서 같은 implicit mutation이 사라지는 범위를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e1a0316fbe84:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 초기 bootstrap은 repository를 만든 직후 `ensureSeedData`를 호출해 process 재시작이 데이터 생성 side effect를 가졌습니다. 서비스 startup과 데이터 준비가 결합되면 production에서도 운영자가 승인하지 않은 seed mutation이 일어날 수 있고 readiness가 실제 사전 준비 상태를 가립니다. |
| 구현 또는 검증 결정 | API entrypoint에서 `ensureSeedData` 호출을 삭제합니다. 데이터 준비는 별도 명령/운영 단계로 남기고 process는 저장소를 선택해 app을 시작하는 일만 합니다. |
| 실행/검증 경로 | 환경 해석 → repository 생성 → app 생성 → listen으로 단순화됩니다. |
| ownership과 failure 처리 | seed 실행 책임이 API process lifetime에서 외부의 명시적 관리 작업으로 이동합니다. 필요한 데이터가 없으면 startup이 자동 복구하지 않습니다. 이는 의도된 non-guarantee이며 운영자가 명시적으로 준비해야 합니다. |
| 보장하는 것 | API startup이 seed 데이터를 암묵적으로 만들지 않습니다. |
| 보장하지 않는 것 | source-level 호출 하나를 제거했을 뿐 다른 경로의 data mutation을 전역적으로 금지하는 것은 아닙니다. |
| 후속 연결 | `5cac4843fd9b`가 entrypoint source에서 호출 재도입을 막습니다. |
<!-- LEARNER-END:e1a0316fbe84:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:e1a0316fbe84:fix -->
startup이 개발 편의를 위해 seed를 생성한다는 초기 가정 → production에서 암묵적 데이터 변경 위험 → entrypoint 호출 제거 → source-level regression guard
<!-- LEARNER-END:e1a0316fbe84:fix -->


#### 비교 기준

- 직전 관련 SHA: `6937cf60aeea` — `test(ops): health와 database readiness 검증`
- 다음 관련 SHA: `5cac4843fd9b` — `test(api): startup seed 금지 검증`

### 5.8. `test(api): startup seed 금지 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `5cac4843fd9b` |
| Importance | B |
| Tags | REALTIME, TEST |
| Source에서 확정된 역할 | entrypoint가 seed operation을 호출하지 않는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/startup.test.ts` (같은 commit의 `tests/smoke-ws.mjs` AI fixture 정렬은 이 Thread의 부수 변경)
- 핵심 symbol: `index.ts` source read, `/\.ensureSeedData\s*\(/` negative assertion
- `startup.test.ts`가 build artifact가 아니라 exact source인 `index.ts`를 읽는 방식을 확인합니다.
- 정규식이 method-call 형태의 `ensureSeedData` 재도입만 막는다는 범위와 우회 가능성을 기록합니다.
- 같은 commit의 WebSocket smoke fixture 변경은 startup seed invariant와 분리해 다룹니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:5cac4843fd9b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | seed 호출은 제거됐지만 이후 refactor가 entrypoint에 다시 추가되어도 이를 직접 탐지하는 테스트가 없었습니다. startup mutation 금지는 코드 review 규칙만으로는 쉽게 회귀할 수 있습니다. |
| 구현 또는 검증 결정 | entrypoint source를 문자열로 읽어 `.ensureSeedData(` 호출 패턴이 없음을 확인합니다. |
| 실행/검증 경로 | test가 `index.ts` 경로 해석 → source read → negative regex assertion을 수행합니다. |
| ownership과 failure 처리 | 테스트는 파일 descriptor를 장기 보유하지 않는 동기 read만 수행합니다. 정규식이 일치하면 즉시 실패합니다. 간접 호출이나 다른 이름의 mutation은 탐지하지 못합니다. |
| 보장하는 것 | entrypoint에 직접 `ensureSeedData` method call이 돌아오는 회귀를 고정적으로 막습니다. |
| 보장하지 않는 것 | process를 실행하거나 실제 database가 변경되지 않는지 관찰하는 테스트는 아닙니다. |
| 후속 연결 | `e1a0316fbe84`의 fix를 좁은 source contract로 보호합니다. |
<!-- LEARNER-END:5cac4843fd9b:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:5cac4843fd9b:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 정적 source regression test |
| 주입·재현 방식 | Node `readFileSync`로 동일 디렉터리의 `index.ts`를 읽고 direct method-call pattern의 부재를 검사합니다. |
| 증명하는 것 | 정확히 그 호출 형태의 재도입을 탐지합니다. |
| 증명하지 않는 것 | 간접 seed 호출, 다른 startup module, runtime database write는 증명하지 않습니다. |
<!-- LEARNER-END:5cac4843fd9b:test -->



#### 비교 기준

- 직전 관련 SHA: `e1a0316fbe84` — `fix(api): startup seed 생성을 제거`
- 다음 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

### 5.9. `fix(config): production에서 영속 저장소 요구`

| 항목 | 값 |
| --- | --- |
| SHA | `eb675ef74af3` |
| Importance | A |
| Tags | TOURNAMENT, OPERATIONS, RISK |
| Source에서 확정된 역할 | production에서 database URL이 없으면 environment parsing 단계에서 실패합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.ts`
- 핵심 symbol: `readEnv`의 production mode 판정과 `DATABASE_URL` 요구 조건
- 명시적 `APP_MODE=production`과 `NODE_ENV=production`에서 mode가 어떻게 결정되는지 확인합니다.
- `DATABASE_URL` 부재를 memory repository 선택까지 미루지 않고 parser에서 거부하는 exact 조건을 기록합니다.
- demo/local/test mode의 memory fallback이 계속 허용되는 범위를 구분합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:eb675ef74af3:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | bootstrap은 `DATABASE_URL`이 없으면 mode와 관계없이 memory repository를 선택할 수 있었습니다. |
| 해결하려던 문제와 위험 | production process가 잘못된 환경 설정으로 시작해 일시적 memory state를 정상 서비스처럼 제공하면 경기·사용자·토너먼트 데이터가 재시작과 함께 사라집니다. |
| 핵심 구현 결정 | 설정 parser가 production mode를 확정한 뒤 database URL이 없으면 예외를 던지도록 변경합니다. 실패가 repository factory보다 앞서므로 서버는 listen하지 않습니다. |
| 입력 → 상태 전이 → 출력 | 환경값 읽기 → app mode 결정 → production이면 `DATABASE_URL` 존재 확인 → 실패 시 throw, 성공 시 bootstrap 진행입니다. |
| ownership/lifetime/cleanup | durable-storage 선택 정책이 composition root의 임시 조건이 아니라 configuration boundary의 검증 규칙이 됩니다. |
| failure/rollback/retry | 오설정 production은 startup fail-fast입니다. memory로 조용히 fallback하지 않습니다. |
| 보장하는 것 | production API는 영속 저장소 위치가 명시되지 않으면 시작하지 않습니다. |
| 보장하지 않는 것 | URL이 실제 연결 가능하거나 올바른 database를 가리키는지는 readiness가 별도로 판정합니다. |
| 후속 연결 | `4633dfde208d`가 명시적/추론된 production 두 경로를 모두 검증합니다. |
<!-- LEARNER-END:eb675ef74af3:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:eb675ef74af3:fix -->
URL 부재 시 모든 mode에서 memory fallback 가능 → production data loss 위험 → configuration parser에서 fail-fast → mode별 regression tests
<!-- LEARNER-END:eb675ef74af3:fix -->


#### 비교 기준

- 직전 관련 SHA: `5cac4843fd9b` — `test(api): startup seed 금지 검증`
- 다음 관련 SHA: `4633dfde208d` — `test(config): production memory fallback 거부 검증`

### 5.10. `test(config): production memory fallback 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4633dfde208d` |
| Importance | A |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | explicit/inferred production 모두 memory fallback을 거부하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.test.ts`
- 핵심 symbol: `readEnv` production cases, demo/local memory allowance
- `APP_MODE=production`과 `NODE_ENV=production` 각각에서 `DATABASE_URL` 누락이 같은 오류로 끝나는지 확인합니다.
- production에 URL을 제공한 positive case와 demo mode의 memory 허용 case를 대조합니다.
- 테스트가 repository 생성 전 parser 단계만 검증한다는 범위를 명시합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:4633dfde208d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production fail-fast 구현은 있었지만 한 mode 판정 경로만 테스트하면 다른 추론 경로가 memory fallback을 다시 허용할 수 있었습니다. |
| 해결하려던 문제와 위험 | explicit app mode와 deployment의 `NODE_ENV` 추론이 서로 다른 branch를 타면 production invariant가 설정 방식에 따라 달라집니다. |
| 핵심 구현 결정 | 두 production 판정 방식 모두 URL 부재에서 reject되는지, URL 제공 시 통과하는지, demo는 계속 memory를 허용하는지 표로 고정합니다. |
| 입력 → 상태 전이 → 출력 | 환경 case 구성 → `readEnv` 호출 → throw 또는 반환 config assertion입니다. |
| ownership/lifetime/cleanup | 테스트가 각 환경 객체를 독립적으로 만들어 process-global 상태에 의존하지 않습니다. |
| failure/rollback/retry | 오류는 configuration 단계에서 발생하므로 repository나 network를 만들지 않습니다. |
| 보장하는 것 | production memory fallback 거부가 mode 표기 방식과 무관하게 유지됩니다. |
| 보장하지 않는 것 | 실제 PostgreSQL 연결·migration current 여부는 readiness 테스트의 책임입니다. |
| 후속 연결 | Thread의 최종 startup invariant를 완성합니다. |
<!-- LEARNER-END:4633dfde208d:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:4633dfde208d:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | negative configuration boundary test |
| 주입·재현 방식 | 명시적 production, `NODE_ENV` 기반 production, demo mode를 독립 입력으로 비교합니다. |
| 증명하는 것 | 모든 production 해석 경로에서 durable-store 설정이 필수입니다. |
| 증명하지 않는 것 | 제공된 URL의 접근 가능성이나 database durability 자체는 증명하지 않습니다. |
<!-- LEARNER-END:4633dfde208d:test -->



#### 비교 기준

- 직전 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:evolution -->
`4b43a284e637`은 process가 실행될 composition root와 repository lifetime을 만들었지만 startup seed와 memory fallback을 함께 허용했습니다. `30aac132e14e`와 `2f05d5d79c64`는 schema 상태 판정과 저장소 소유의 readiness를 도입했고, `15002e229acb`는 이를 process liveness와 분리된 HTTP 신호로 내보냈습니다. `e1a0316fbe84`/`5cac4843fd9b`가 startup mutation을 제거·고정하고, `eb675ef74af3`/`4633dfde208d`가 production의 durable-storage fail-fast를 완성합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:failure-links -->
- implicit seed mutation → `e1a0316fbe84` 제거 → `5cac4843fd9b` source regression guard
- production memory fallback → `eb675ef74af3` parser fail-fast → `4633dfde208d` explicit/inferred production tests
- database/migration 불확실성 → `30aac132e14e`/`2f05d5d79c64` 판정 → `6937cf60aeea` route·PostgreSQL evidence
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:ownership -->
entrypoint는 환경·repository 생성과 close hook만 소유합니다. repository는 connection 및 migration 상태 판정을 소유하고, Fastify health route는 그 결과를 제한된 외부 상태로 변환합니다. seed/migration 실행은 API process 밖의 명시적 운영 작업으로 이동합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:final-state -->
process가 응답하면 liveness는 유지되지만, database down·pending/diverged migration이면 readiness는 503입니다. production은 database URL이 없으면 listen 이전에 실패하고, startup은 seed를 만들지 않습니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:final-flow -->
- 환경 파싱에서 mode와 durable-store 요구를 검증합니다.
- entrypoint가 repository를 만들고 Fastify app에 주입합니다.
- `/health/live`는 dependency와 무관하게 process 생존을 응답합니다.
- `/health/ready`는 repository의 connection 및 migration 집합 결과를 읽어 200/503을 결정합니다.
- 종료 시 Fastify `onClose`가 repository `close`를 호출합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:checks -->
- [x] migration `current/pending/diverged/not_applicable`의 의미와 계산 근거를 설명할 수 있습니다.
- [x] liveness와 readiness의 호출 경로·실패 branch를 exact SHA 기준으로 구분할 수 있습니다.
- [x] startup seed 제거와 production memory fallback 거부의 fix→test 관계를 설명할 수 있습니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:checks -->
===== END FILE: 01-startup-liveness-readiness-and-storage-state.md =====

===== BEGIN FILE: 02-metrics-observer-boundaries-and-cardinality.md =====
# Metrics observer boundary와 cardinality

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

runtime, HTTP, repository, room/reconnect, finalization, snapshot delivery, event-loop lag를 Prometheus에 노출하되 domain code와 high-cardinality identity를 metric label에 결합하지 않는 구조를 복원합니다.

범위 메모: 민감 로그 redaction(`4c7e884bc9b0`)은 observability tag를 갖지만 주된 invariant가 인증 credential 비노출이므로 identity/security 카테고리에 남기고, 여기서는 metric cardinality와 observer ownership만 다룹니다.

### 직접 연결되는 불변식

- 관측 코드는 domain state machine 내부 규칙을 소유하거나 변경하지 않습니다.
- metric label은 bounded vocabulary를 사용해 cardinality가 user/room 수에 비례하지 않습니다.
- delivery/finalization/readiness metric은 해당 결과가 실제 결정되는 경계에서 기록됩니다.
- collector와 event-loop histogram의 lifetime은 Fastify app lifetime과 함께 종료됩니다.

## 2. 핵심 질문

- metric registry와 collector lifecycle은 app startup/close에서 누가 소유합니까?
- HTTP raw URL 대신 route template을 label로 사용하는 이유는 무엇입니까?
- repository proxy가 method return type, `this`, throw/rejection semantics를 보존합니까?
- room/user/request ID를 metric label로 쓰지 않으면서 correlation은 log/observer에서 어떻게 유지합니까?
- snapshot drop과 send callback delay를 측정하는 위치가 실제 semantics owner와 일치합니까?
- event-loop p95가 load harness에 전달될 때 missing sample을 어떻게 fail-closed 처리합니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `6bf29a5acf35` | `build(api): metrics 수집 의존성 추가` | C | - | `prom-client`를 API의 명시적 runtime dependency로 추가합니다. |
| 2 | `69278d8fc456` | `feat(metrics): runtime gauge registry 추가` | B | REALTIME, OPERATIONS, OBSERVABILITY | Node collector와 live GameHub gauge를 위한 전용 registry를 만듭니다. |
| 3 | `02b3b3a32f14` | `feat(metrics): HTTP와 readiness 측정 추가` | B | OPERATIONS, OBSERVABILITY, PERF | normalized route/method/status로 request duration과 readiness를 측정합니다. |
| 4 | `843d355afc69` | `feat(metrics): repository operation 측정 추가` | B | PERSISTENCE, OBSERVABILITY | transparent proxy로 sync/async repository method duration/result를 측정합니다. |
| 5 | `e08367a1be5e` | `feat(metrics): game room과 reconnect 관측 추가` | B | AUTH, PROTOCOL, REALTIME | GameHub lifecycle 주변 observer로 room/reconnect event를 관측합니다. |
| 6 | `e850b3356b9b` | `feat(metrics): match finalization 결과 관측 추가` | B | REALTIME, PERSISTENCE, OBSERVABILITY | guest memory result와 DB-backed finalization success/failure를 구분해 측정합니다. |
| 7 | `c0d184bcc928` | `feat(metrics): snapshot delivery와 drop 관측 추가` | B | REALTIME, OBSERVABILITY, PERF | latest-buffer가 실제 delivery/drop을 결정하는 지점에서 측정합니다. |
| 8 | `685d85c863a4` | `test(metrics): database와 snapshot 지표 검증` | B | AUTH, REALTIME, PERSISTENCE | user/room ID를 label로 만들지 않고 DB/realtime behavior를 관측하는지 검증합니다. |
| 9 | `1baf4c5a57ba` | `feat(metrics): event-loop lag 측정 추가` | B | OBSERVABILITY | Node event-loop delay histogram의 p95를 gauge로 노출합니다. |
| 10 | `66b8f07c2387` | `test(load): event-loop lag를 부하 profile에 노출` | B | OPERATIONS, OBSERVABILITY, PERF | load overlay에서 metrics endpoint를 loopback에 노출하고 k6 teardown이 server p95를 수집합니다. |
| 11 | `697a63ebb8c8` | `test(load): event-loop lag 임계값 검증` | B | OPERATIONS, OBSERVABILITY, PERF | 50ms p95 threshold, required metric, loopback metrics exposure를 contract test로 고정합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(api): metrics 수집 의존성 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `6bf29a5acf35` |
| Importance | C |
| Tags | - |
| Source에서 확정된 역할 | `prom-client`를 API의 명시적 runtime dependency로 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/package.json`, `pnpm-lock.yaml`
- 핵심 symbol: API dependency `prom-client@^15.1.3`와 lockfile resolution
- API package의 dependency 구역과 lockfile importer가 같은 버전을 가리키는지 확인합니다.
- 이 SHA에는 collector나 route가 없고 다음 commit을 가능하게 하는 기계적 준비라는 범위를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:6bf29a5acf35:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 맥락 | API package에는 Prometheus registry·metric type을 제공하는 runtime dependency가 없었습니다. |
| 구체적 역할 | API package에 `prom-client`를 추가하고 lockfile을 갱신합니다. |
| 보장/제한 | API가 `prom-client`를 직접 import할 수 있습니다. metric 생성·노출·cleanup 동작은 아직 없습니다. |
| 후속 연결 | `69278d8fc456`가 이 의존성으로 전용 registry를 만듭니다. |
<!-- LEARNER-END:6bf29a5acf35:record -->




#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `69278d8fc456` — `feat(metrics): runtime gauge registry 추가`

### 5.2. `feat(metrics): runtime gauge registry 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `69278d8fc456` |
| Importance | B |
| Tags | REALTIME, OPERATIONS, OBSERVABILITY |
| Source에서 확정된 역할 | Node collector와 live GameHub gauge를 위한 전용 registry를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`
- 핵심 symbol: `ApiMetrics`, private `Registry`, `collectDefaultMetrics`, `scrape`, `close`
- `ApiMetrics`가 global registry가 아니라 private `Registry`를 만들고 모든 gauge를 그 registry에 등록하는지 확인합니다.
- `scrape()`가 `readGameStats` callback을 호출한 시점의 online/queued/room 값을 gauge에 set한 뒤 serialization하는 순서를 추적합니다.
- `close()`가 registry를 clear하고 app lifetime과 collector lifetime을 맞추는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:69278d8fc456:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | GameHub의 live state와 Node runtime 상태를 scrape 가능한 형태로 보유하는 객체가 없었습니다. global singleton metric을 쓰면 test/app instance 사이 collector 중복과 lifetime 누수가 생기며, game state를 metric object가 직접 소유하면 domain state와 관측 state가 결합됩니다. |
| 구현 또는 검증 결정 | `ApiMetrics`가 전용 `Registry`를 소유하고 Node default metrics, connection·queue·room gauge를 등록합니다. GameHub 상태는 생성자에 주입된 `readGameStats` callback으로 scrape 시점에 읽습니다. |
| 실행/검증 경로 | scrape 요청 → callback으로 live stats 읽기 → gauge set → private registry serialize입니다. |
| ownership과 failure 처리 | app instance마다 `ApiMetrics`와 registry가 하나씩 존재합니다. GameHub는 상태를 소유하고 metrics는 read callback만 보유하며 `close()`가 registry를 정리합니다. callback이나 registry serialization 오류는 `scrape()` reject로 남습니다. metric은 domain state를 변경하지 않습니다. |
| 보장하는 것 | Node와 live GameHub 상태를 app-local registry에서 수집할 기반이 생깁니다. |
| 보장하지 않는 것 | HTTP endpoint와 request/repository/realtime outcome 측정은 아직 연결되지 않습니다. |
| 후속 연결 | `02b3b3a32f14`가 app lifecycle과 `/metrics` route에 연결합니다. |
<!-- LEARNER-END:69278d8fc456:record -->




#### 비교 기준

- 직전 관련 SHA: `6bf29a5acf35` — `build(api): metrics 수집 의존성 추가`
- 다음 관련 SHA: `02b3b3a32f14` — `feat(metrics): HTTP와 readiness 측정 추가`

### 5.3. `feat(metrics): HTTP와 readiness 측정 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `02b3b3a32f14` |
| Importance | B |
| Tags | OPERATIONS, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | normalized route/method/status로 request duration과 readiness를 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`, `apps/api/src/app.ts`
- 핵심 symbol: HTTP duration histogram, readiness histogram/counter, `/metrics`, `onResponse` hook
- request label이 raw URL이 아니라 `request.routeOptions.url` 또는 bounded fallback을 쓰는지 확인합니다.
- duration을 high-resolution elapsed time에서 seconds로 바꾸고 음수를 clamp하는 helper와 status/method label을 확인합니다.
- `buildApp`이 metrics를 만들고 `onResponse`, readiness route, `/metrics`, `onClose`에 연결하는 lifetime을 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:02b3b3a32f14:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | registry는 있었지만 scrape route가 없고 HTTP·readiness 결과가 운영 지표에 남지 않았습니다. raw URL이나 request ID를 label로 쓰면 요청 수에 비례해 시계열이 증가합니다. 또한 health route의 결과와 요청 duration을 실제 응답 경계에서 기록해야 합니다. |
| 구현 또는 검증 결정 | Fastify `onResponse`에서 normalized route template, method, status code로 request duration을 기록합니다. readiness 실행 시간·결과를 별도로 기록하고 `/metrics`가 registry content type과 body를 반환하도록 연결합니다. |
| 실행/검증 경로 | request 시작 시각 보관 → route 실행 → `onResponse`에서 elapsed seconds 기록; `/health/ready`는 repository 결과를 응답으로 바꾸며 readiness metric도 기록; `/metrics`는 scrape합니다. |
| ownership과 failure 처리 | Fastify app이 `ApiMetrics`를 생성·close하고 hook/route가 같은 인스턴스를 공유합니다. unmatched route는 제한된 fallback label을 사용하며 negative elapsed는 0으로 clamp됩니다. scrape 실패를 masking하지는 않습니다. |
| 보장하는 것 | HTTP와 readiness의 latency/outcome이 bounded labels로 노출됩니다. |
| 보장하지 않는 것 | repository method와 GameHub 내부 outcome은 아직 HTTP metric으로만 간접 관찰됩니다. |
| 후속 연결 | `843d355afc69`부터 결과가 결정되는 내부 경계에 observer를 추가합니다. |
<!-- LEARNER-END:02b3b3a32f14:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:02b3b3a32f14:snippet -->
- SHA: `02b3b3a32f14`
- 위치: `apps/api/src/observability.ts`; HTTP duration histogram, readiness histogram/counter, `/metrics`, `onResponse` hook

```ts
const route = request.routeOptions.url ?? "unmatched";
metrics.observeRequest(route, request.method, reply.statusCode, elapsedSeconds);
```
<!-- LEARNER-END:02b3b3a32f14:snippet -->

#### 비교 기준

- 직전 관련 SHA: `69278d8fc456` — `feat(metrics): runtime gauge registry 추가`
- 다음 관련 SHA: `843d355afc69` — `feat(metrics): repository operation 측정 추가`

### 5.4. `feat(metrics): repository operation 측정 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `843d355afc69` |
| Importance | B |
| Tags | PERSISTENCE, OBSERVABILITY |
| Source에서 확정된 역할 | transparent proxy로 sync/async repository method duration/result를 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`, `apps/api/src/app.ts`
- 핵심 symbol: `instrumentRepository`, `Proxy.get`, bounded `REPOSITORY_OPERATIONS`, success/failure observer
- proxy가 known repository method만 bounded operation label로 쓰고 나머지는 `other`로 축약하는지 확인합니다.
- method 호출 시 receiver/`this`를 원본 repository로 보존하는 `Reflect`/`apply` 경로를 확인합니다.
- 동기 throw와 Promise resolve/reject가 각각 한 번만 duration/outcome으로 기록되고 원래 반환·예외 semantics를 보존하는지 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:843d355afc69:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | HTTP route duration은 알 수 있었지만 database/repository 작업별 latency와 실패를 실제 repository 호출 경계에서 구분할 수 없었습니다. caller마다 측정 코드를 넣으면 누락과 중복이 생기고, wrapper가 sync/async 반환이나 `this` binding을 바꾸면 production behavior가 달라집니다. |
| 구현 또는 검증 결정 | repository를 `Proxy`로 감싸 함수 호출을 측정하되 원본 receiver와 반환값을 보존합니다. 동기 throw는 catch에서, Promise는 resolve/reject handler에서 outcome을 기록합니다. |
| 실행/검증 경로 | caller → proxy property get → 원본 method apply → sync 결과/throw 또는 Promise settle → metric 기록 → 원래 결과 전달입니다. |
| ownership과 failure 처리 | source repository가 connection과 data state를 계속 소유합니다. proxy는 측정 wrapper일 뿐 `close`를 포함한 모든 method를 원본에 위임합니다. 동기 예외와 비동기 rejection 모두 `failure`로 기록한 뒤 그대로 다시 전달합니다. operation label은 bounded vocabulary 밖이면 `other`입니다. |
| 보장하는 것 | repository behavior를 바꾸지 않으면서 operation별 success/failure duration을 관찰합니다. |
| 보장하지 않는 것 | transaction 내부 SQL statement별 latency나 query cardinality까지는 보여 주지 않습니다. |
| 후속 연결 | `685d85c863a4`가 실제 database metric과 label 비노출을 검증합니다. |
<!-- LEARNER-END:843d355afc69:record -->




#### 비교 기준

- 직전 관련 SHA: `02b3b3a32f14` — `feat(metrics): HTTP와 readiness 측정 추가`
- 다음 관련 SHA: `e08367a1be5e` — `feat(metrics): game room과 reconnect 관측 추가`

### 5.5. `feat(metrics): game room과 reconnect 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e08367a1be5e` |
| Importance | B |
| Tags | AUTH, PROTOCOL, REALTIME |
| Source에서 확정된 역할 | GameHub lifecycle 주변 observer로 room/reconnect event를 관측합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`
- 핵심 symbol: `GameHubObserver.roomCreated`, `GameHubObserver.reconnect`, client `requestId`, app logging callback
- GameHub가 concrete logger/metric class가 아니라 optional observer callbacks만 받는 constructor boundary를 확인합니다.
- room 생성 시 room/user/request ID는 structured log context로 전달되지만 metric label에는 들어가지 않는 분리를 확인합니다.
- reconnect `success|expired` outcome이 결정된 정확한 state transition 뒤 observer가 호출되는지 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e08367a1be5e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | GameHub room/reconnect lifecycle은 동작했지만 어떤 room이 생성되고 복구가 성공/만료됐는지 외부에서 관찰할 hook이 없었습니다. domain state machine 내부에 logger와 metric registry를 직접 넣으면 lifecycle 규칙과 관측 구현이 결합되고, identity를 metric label로 쓰면 cardinality가 무한히 증가합니다. |
| 구현 또는 검증 결정 | GameHub는 선택적 `GameHubObserver`를 받고 room 생성과 reconnect 결과가 확정된 지점에서 bounded outcome 및 correlation context를 전달합니다. app은 outcome만 metric에 넣고 ID는 structured log context로 남깁니다. |
| 실행/검증 경로 | HTTP/WebSocket 인증에서 request/user context 확보 → `hub.connect` → room/reconnect transition → observer callback → app logger/metric 호출입니다. |
| ownership과 failure 처리 | GameHub가 room state와 callback 호출 시점을 소유하고 app이 logging/metrics 구현을 소유합니다. observer는 state를 변경할 권한이 없습니다. observer는 optional이라 미설정 시 domain 동작은 계속됩니다. 이 SHA에서 callback 자체가 throw할 때 containment를 별도로 제공하는지는 확인되지 않습니다. |
| 보장하는 것 | room/reconnect 관측이 state machine 밖의 adapter에 연결되고 metric label은 bounded outcome으로 제한됩니다. |
| 보장하지 않는 것 | structured log의 redaction 정책은 이 Thread 밖의 logging/auth commit이 소유합니다. |
| 후속 연결 | `e850b3356b9b`가 같은 pattern을 finalization outcome에 확장합니다. |
<!-- LEARNER-END:e08367a1be5e:record -->




#### 비교 기준

- 직전 관련 SHA: `843d355afc69` — `feat(metrics): repository operation 측정 추가`
- 다음 관련 SHA: `e850b3356b9b` — `feat(metrics): match finalization 결과 관측 추가`

### 5.6. `feat(metrics): match finalization 결과 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e850b3356b9b` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, OBSERVABILITY |
| Source에서 확정된 역할 | guest memory result와 DB-backed finalization success/failure를 구분해 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`, `apps/api/src/observability.ts`
- 핵심 symbol: `GameHubObserver.matchFinalized`, finalization success/failure branches, persistence/outcome labels
- memory guest result와 database-backed `repo.finalizeMatch` 경로에서 observer context가 어떻게 달라지는지 확인합니다.
- database finalization Promise의 성공과 catch branch가 각각 `success|failure` outcome을 한 번 기록하는지 추적합니다.
- persistence label이 `memory|database`, outcome이 bounded vocabulary이며 match/room ID가 metric label에 없는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e850b3356b9b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | repository proxy로 method failure는 볼 수 있었지만 하나의 match finalization이라는 domain 결과가 memory/database에서 어떻게 끝났는지 직접 나타내지 못했습니다. repository operation metric만으로는 retry·duplicate·guest completion과 같은 domain 의미를 복원하기 어렵습니다. |
| 구현 또는 검증 결정 | GameHub finalization 경계에 observer를 추가해 persistence 종류와 success/failure를 기록합니다. memory 결과는 저장소 호출 없이 success로, database 결과는 `finalizeMatch` settle 결과에 따라 기록합니다. |
| 실행/검증 경로 | room terminal state → memory result 생성 또는 repository finalization → 결과 확정 → observer → metric/log adapter입니다. |
| ownership과 failure 처리 | GameHub가 finalization lifecycle과 관찰 시점을 소유하고 repository가 durable write를 소유합니다. metrics는 결과를 복제해 기록할 뿐 성공 여부를 결정하지 않습니다. database reject는 failure metric을 남긴 뒤 기존 retry/cleanup path로 전달됩니다. 관측 기록이 persistence 결과를 success로 바꾸지 않습니다. |
| 보장하는 것 | domain finalization outcome과 persistence 종류를 bounded metric으로 구분합니다. |
| 보장하지 않는 것 | duplicate finalization counter는 `ad482c200cea`의 후속 cadence/finalization 보정에서 추가됩니다. |
| 후속 연결 | `547d9943d30a`가 load harness의 source of truth를 client event에서 이 server metric으로 옮깁니다. |
<!-- LEARNER-END:e850b3356b9b:record -->




#### 비교 기준

- 직전 관련 SHA: `e08367a1be5e` — `feat(metrics): game room과 reconnect 관측 추가`
- 다음 관련 SHA: `c0d184bcc928` — `feat(metrics): snapshot delivery와 drop 관측 추가`

### 5.7. `feat(metrics): snapshot delivery와 drop 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `c0d184bcc928` |
| Importance | B |
| Tags | REALTIME, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | latest-buffer가 실제 delivery/drop을 결정하는 지점에서 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`, `apps/api/src/observability.ts`, GameHub wiring
- 핵심 symbol: `PendingSnapshot.enqueuedAt`, delivered observer, drop reasons `replaced|connection_closed|congestion`
- snapshot을 enqueue할 때 monotonic timestamp를 저장하고 실제 send callback/queue 처리에서 delay를 계산하는지 확인합니다.
- latest-value replacement, connection close, congestion termination 각각이 bounded drop reason으로 기록되는 branch를 추적합니다.
- observer가 buffer의 delivery semantics owner 안에 있고 GameHub가 추측해서 drop을 세지 않는 이유를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:c0d184bcc928:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | snapshot buffer는 오래된 값을 교체하거나 congestion에서 버릴 수 있었지만 외부에서는 정상 latest-value loss와 transport failure를 구분할 수 없었습니다. 호출자에서 send 횟수만 세면 실제로 대체된 snapshot, 연결 종료로 폐기된 snapshot, 전달 완료 지연을 정확히 알 수 없습니다. |
| 구현 또는 검증 결정 | pending snapshot에 enqueue 시각을 넣고 buffer가 delivered/drop을 결정하는 branch에서 observer를 호출합니다. drop reason은 세 개의 bounded 값으로 제한합니다. |
| 실행/검증 경로 | GameHub snapshot → buffer enqueue/replacement → socket 상태·buffered amount 판단 → send 또는 drop → delay/reason observer입니다. |
| ownership과 failure 처리 | 각 client의 `LatestSnapshotBuffer`가 pending payload와 측정 시점을 소유합니다. metrics adapter는 숫자와 bounded reason만 받습니다. connection close와 congestion은 drop으로 기록되며 source payload나 room ID는 metric label에 포함되지 않습니다. |
| 보장하는 것 | delivery delay와 실제 drop 결정이 동일한 owner에서 관찰됩니다. |
| 보장하지 않는 것 | 이 시점의 `sending` flag가 congestion으로 해석되는 가정은 `d90f17fa765d`에서 수정됩니다. |
| 후속 연결 | `685d85c863a4`가 measurement와 cardinality를 검증하고 `d90f17fa765d`가 callback 지연 오판을 고칩니다. |
<!-- LEARNER-END:c0d184bcc928:record -->




#### 비교 기준

- 직전 관련 SHA: `e850b3356b9b` — `feat(metrics): match finalization 결과 관측 추가`
- 다음 관련 SHA: `685d85c863a4` — `test(metrics): database와 snapshot 지표 검증`

### 5.8. `test(metrics): database와 snapshot 지표 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `685d85c863a4` |
| Importance | B |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | user/room ID를 label로 만들지 않고 DB/realtime behavior를 관측하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: observability route/test files와 `LatestSnapshotBuffer` fake-socket tests
- 핵심 symbol: Fastify `/metrics` scrape, fake timers/socket, label absence assertions
- repository success/failure와 snapshot replacement/delivery를 만들고 scrape body의 metric 이름·label을 확인하는 test setup을 추적합니다.
- 50ms delivery와 replacement/drop을 fake timer·fake socket으로 결정적으로 재현하는지 확인합니다.
- `requestId`, `userId`, `roomId`, `matchId` 문자열이 Prometheus output에 없다는 negative assertion을 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:685d85c863a4:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | metric 구현은 있었지만 결과가 실제 scrape에 나타나는지, identity가 label로 새어 시계열을 폭증시키지 않는지 보호되지 않았습니다. metric 이름 존재만 확인하면 observer가 잘못된 branch에서 호출되거나 high-cardinality label이 추가돼도 놓칠 수 있습니다. |
| 구현 또는 검증 결정 | repository와 snapshot buffer의 실제 production path를 통과시킨 뒤 `/metrics` output을 검사합니다. fake socket과 timer로 replacement·delivery를 재현하고 ID label 부재를 확인합니다. |
| 실행/검증 경로 | test action → observer metric update → app scrape → text exposition assertion입니다. |
| ownership과 failure 처리 | 테스트가 app, repository, fake socket, fake timer를 생성하고 teardown에서 정리합니다. repository rejection과 snapshot drop을 의도적으로 만들지만 실제 PostgreSQL/network congestion은 사용하지 않습니다. |
| 보장하는 것 | 중요 metric이 scrape되고 bounded label 정책이 regression으로 고정됩니다. |
| 보장하지 않는 것 | Prometheus server ingestion, retention, alert rule, 실제 부하 분포는 증명하지 않습니다. |
| 후속 연결 | 후속 `66b8f07c2387`/`697a63ebb8c8`가 실제 load profile에서 event-loop metric을 읽고 threshold를 고정합니다. |
<!-- LEARNER-END:685d85c863a4:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:685d85c863a4:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 metric integration test |
| 주입·재현 방식 | Fastify scrape와 fake timer/socket을 결합해 production observer path를 실행합니다. |
| 증명하는 것 | metric emission, bounded labels, snapshot delivery/drop 측정 위치를 검증합니다. |
| 증명하지 않는 것 | 실제 collector backend나 장시간 cardinality 증가량은 측정하지 않습니다. |
<!-- LEARNER-END:685d85c863a4:test -->



#### 비교 기준

- 직전 관련 SHA: `c0d184bcc928` — `feat(metrics): snapshot delivery와 drop 관측 추가`
- 다음 관련 SHA: `1baf4c5a57ba` — `feat(metrics): event-loop lag 측정 추가`

### 5.9. `feat(metrics): event-loop lag 측정 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1baf4c5a57ba` |
| Importance | B |
| Tags | OBSERVABILITY |
| Source에서 확정된 역할 | Node event-loop delay histogram의 p95를 gauge로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`
- 핵심 symbol: `monitorEventLoopDelay({resolution: 20})`, p95 gauge collect callback, `enable`, `disable`
- Node histogram을 constructor에서 enable하고 `close()`에서 disable하는 lifetime을 확인합니다.
- nanoseconds percentile 값을 seconds로 변환하는 계산과 p95 gauge collect 시점을 확인합니다.
- default metrics의 event-loop 지표와 별도로 service-level p95 gauge를 만든 이유와 reset 여부를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1baf4c5a57ba:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | Node default metrics는 있었지만 load threshold가 직접 소비할 하나의 p95 event-loop lag sample이 없었습니다. runtime scheduling pressure를 client snapshot delay만으로 추정하면 network와 browser 영향을 분리할 수 없습니다. |
| 구현 또는 검증 결정 | `monitorEventLoopDelay` histogram을 20ms resolution으로 enable하고 collect 시 95번째 percentile을 seconds gauge로 설정합니다. |
| 실행/검증 경로 | app metrics 생성 → histogram enable → runtime delay samples 누적 → scrape collect에서 p95/1e9 → app close에서 disable입니다. |
| ownership과 failure 처리 | `ApiMetrics`가 histogram handle의 enable/disable lifetime을 소유합니다. sample이 비정상일 때의 세부 fallback은 metric 구현 범위에 따르며, 이 commit은 alert나 admission을 바꾸지 않습니다. |
| 보장하는 것 | server event-loop p95를 scrape 가능한 bounded 단일 gauge로 제공합니다. |
| 보장하지 않는 것 | 이 metric만으로 원인이나 per-request 지연을 식별하지 못합니다. |
| 후속 연결 | `66b8f07c2387`가 load overlay와 k6 teardown에서 이 gauge를 읽습니다. |
<!-- LEARNER-END:1baf4c5a57ba:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:1baf4c5a57ba:snippet -->
- SHA: `1baf4c5a57ba`
- 위치: `apps/api/src/observability.ts`; `monitorEventLoopDelay({resolution: 20})`, p95 gauge collect callback, `enable`, `disable`

```ts
const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();
// collect: histogram.percentile(95) / 1_000_000_000
```
<!-- LEARNER-END:1baf4c5a57ba:snippet -->

#### 비교 기준

- 직전 관련 SHA: `685d85c863a4` — `test(metrics): database와 snapshot 지표 검증`
- 다음 관련 SHA: `66b8f07c2387` — `test(load): event-loop lag를 부하 profile에 노출`

### 5.10. `test(load): event-loop lag를 부하 profile에 노출`

| 항목 | 값 |
| --- | --- |
| SHA | `66b8f07c2387` |
| Importance | B |
| Tags | OPERATIONS, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | load overlay에서 metrics endpoint를 loopback에 노출하고 k6 teardown이 server p95를 수집합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.load.yml`, `tests/load/pong-load.js`, load profile configuration
- 핵심 symbol: loopback-only API metrics port, k6 `teardown`, `event_loop_lag_p95_ms` trend
- load overlay가 API metrics port를 `127.0.0.1`에만 publish하는지 확인합니다.
- k6 teardown이 `/metrics`를 GET하고 `pong_pong_api_event_loop_lag_p95_seconds` sample을 파싱해 milliseconds trend에 넣는지 추적합니다.
- threshold `p(95)<=50`과 scrape 실패/missing metric에서 `fail`하는 branch를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:66b8f07c2387:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | event-loop gauge는 API 내부에 있었지만 load harness가 이를 읽지 않아 client-visible SLI와 server scheduler pressure를 같은 run에서 비교할 수 없었습니다. metrics endpoint가 외부에 무제한 노출되거나 k6가 metric 부재를 0으로 처리하면 부하 결과가 잘못 통과할 수 있습니다. |
| 구현 또는 검증 결정 | load overlay에서 metrics port를 loopback으로 제한하고, k6 teardown이 scrape 결과를 읽어 p95 seconds를 milliseconds trend로 기록합니다. 누락·비정상 sample은 run failure입니다. |
| 실행/검증 경로 | load scenario 실행 → teardown HTTP scrape → Prometheus line parse → k6 custom trend 추가 → threshold 평가입니다. |
| ownership과 failure 처리 | API가 metric을 소유하고 load harness는 run 종료 시 읽기만 합니다. port 공개 범위는 overlay가 소유합니다. scrape status/body 또는 sample이 유효하지 않으면 `fail`합니다. 0으로 대체해 false pass하지 않습니다. |
| 보장하는 것 | load run이 server event-loop p95를 결과에 포함하고 loopback에서만 접근합니다. |
| 보장하지 않는 것 | 실제 load run을 이 commit의 unit test가 실행하지는 않습니다. |
| 후속 연결 | `697a63ebb8c8`가 threshold와 overlay/text contract를 정적으로 검증합니다. |
<!-- LEARNER-END:66b8f07c2387:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:66b8f07c2387:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | load-harness operational integration |
| 주입·재현 방식 | k6 teardown이 실제 HTTP scrape를 수행하도록 구현되며 threshold는 k6 options에 포함됩니다. |
| 증명하는 것 | 실행된 load run에서는 metric 부재가 실패로 처리되도록 경로가 존재합니다. |
| 증명하지 않는 것 | 이 workbook 환경에서는 k6 run이 실행되지 않았으므로 수치 결과는 제공하지 않습니다. |
<!-- LEARNER-END:66b8f07c2387:test -->



#### 비교 기준

- 직전 관련 SHA: `1baf4c5a57ba` — `feat(metrics): event-loop lag 측정 추가`
- 다음 관련 SHA: `697a63ebb8c8` — `test(load): event-loop lag 임계값 검증`

### 5.11. `test(load): event-loop lag 임계값 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `697a63ebb8c8` |
| Importance | B |
| Tags | OPERATIONS, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | 50ms p95 threshold, required metric, loopback metrics exposure를 contract test로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-harness.test.mjs`, load profile/overlay source
- 핵심 symbol: source/compose assertions, `event_loop_lag_p95_ms` threshold, required metric name
- load profile options가 event-loop trend에 `p(95)<=50`을 정확히 요구하는지 확인합니다.
- harness source가 expected Prometheus metric을 참조하고 teardown에서 읽는다는 정적 assertion을 확인합니다.
- Compose publish address가 loopback인지와 public edge port와 섞이지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:697a63ebb8c8:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | load harness가 threshold를 갖게 됐지만 설정·metric 이름·port binding이 수정되면 실제 부하 전까지 회귀를 발견하기 어려웠습니다. 운영 측정의 wiring은 문자열/config 변경으로 쉽게 무력화되며 실행 비용이 큰 k6만으로 매 commit 검증하기 어렵습니다. |
| 구현 또는 검증 결정 | Node contract test가 load profile, k6 source, Compose overlay를 읽어 50ms threshold와 required metric, loopback binding을 고정합니다. |
| 실행/검증 경로 | source/config read → pattern 및 object assertion → 실패 시 test error입니다. |
| ownership과 failure 처리 | 정적 contract test는 runtime service를 시작하지 않고 파일 내용만 소유합니다. threshold 제거, metric rename, non-loopback port publish를 탐지합니다. |
| 보장하는 것 | event-loop load contract의 핵심 wiring이 빠른 정적 테스트로 보호됩니다. |
| 보장하지 않는 것 | 실제 50ms 이하 성능이나 Prometheus parser의 모든 exposition format을 증명하지 않습니다. |
| 후속 연결 | Thread의 측정 체인을 registry → scrape → load collection → static contract로 닫습니다. |
<!-- LEARNER-END:697a63ebb8c8:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:697a63ebb8c8:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 정적 operational contract test |
| 주입·재현 방식 | load profile object와 source/Compose text를 읽어 threshold·metric·binding을 검사합니다. |
| 증명하는 것 | 측정 wiring과 임계값 설정의 회귀를 탐지합니다. |
| 증명하지 않는 것 | 부하 중 event-loop p95가 실제 기준을 만족한다는 실행 증거는 아닙니다. |
<!-- LEARNER-END:697a63ebb8c8:test -->



#### 비교 기준

- 직전 관련 SHA: `66b8f07c2387` — `test(load): event-loop lag를 부하 profile에 노출`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:evolution -->
`6bf29a5acf35`/`69278d8fc456`은 app-local registry와 collector lifetime을 만들고, `02b3b3a32f14`는 bounded HTTP/readiness labels로 외부 scrape 경계를 엽니다. `843d355afc69`부터 repository, GameHub lifecycle, finalization, snapshot buffer처럼 결과가 확정되는 owner에 observer를 배치합니다. `685d85c863a4`는 identity label 부재를 고정하고, `1baf4c5a57ba`부터 event-loop p95를 load run까지 전달해 50ms contract로 보호합니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:failure-links -->
- raw URL/identity label cardinality 위험 → normalized route와 bounded outcome/reason → `685d85c863a4` negative label assertions
- repository sync throw/async reject 관측 누락 위험 → transparent proxy settle handling → database metric tests
- event-loop metric 누락을 0으로 오판할 위험 → teardown fail-closed parser → `697a63ebb8c8` static contract
- callback 지연을 drop으로 오판하는 측정 의미 문제는 Thread 04의 `d90f17fa765d`/`5cd54767858f`에서 수정됩니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:ownership -->
Fastify app이 `ApiMetrics`와 registry/histogram lifetime을 소유합니다. repository와 GameHub는 production state를 계속 소유하고 proxy/observer는 결과를 읽어 전달합니다. `LatestSnapshotBuffer`는 delivery/drop 의미를 소유하므로 해당 지표도 buffer 내부에서 발생합니다. load harness는 scrape consumer일 뿐 server metric을 재정의하지 않습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:final-state -->
HTTP, readiness, repository, room/reconnect, finalization, snapshot delivery/drop, event-loop p95가 bounded labels로 scrape됩니다. user/request/room/match ID는 correlation용 structured log context에만 남고 metric 시계열을 생성하지 않습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:final-flow -->
- app 생성 시 private registry와 default/event-loop collector를 시작합니다.
- HTTP·repository·GameHub·snapshot buffer가 각 결과 확정 지점에서 bounded observer를 호출합니다.
- `/metrics`가 app-local registry를 Prometheus text로 직렬화합니다.
- load harness teardown이 loopback scrape에서 event-loop p95와 server finalization counters를 읽습니다.
- app close가 registry를 clear하고 event-loop histogram을 disable합니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:checks -->
- [x] 각 metric의 semantic owner와 label vocabulary를 파일·함수 기준으로 설명할 수 있습니다.
- [x] repository proxy의 sync/async semantics 보존과 failure 기록 순서를 설명할 수 있습니다.
- [x] event-loop metric의 생성·scrape·load threshold·정적 contract 연결을 구분할 수 있습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:checks -->
===== END FILE: 02-metrics-observer-boundaries-and-cardinality.md =====

===== BEGIN FILE: 03-runtime-limiter-primitives-and-bounded-work.md =====
# Runtime limiter primitive와 bounded work

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

fixed-step 시간 보정, connection heartbeat, input ordering/rate limit, latest-value snapshot buffer를 GameHub와 독립된 primitive로 복원하고 각 primitive가 소유하는 시간·상태·cleanup 상한을 확인합니다.

범위 메모: 이 Thread는 reusable limiter의 탄생과 deterministic 경계까지만 다룹니다. GameHub lifecycle 통합, shared scheduler ownership, 실제 congestion fix는 다음 Thread로 분리합니다.

### 직접 연결되는 불변식

- 한 timer callback이 수행하는 simulation catch-up work는 고정된 상한을 넘지 않습니다.
- 응답 없는 connection은 단일 heartbeat owner가 bounded deadline 뒤 제거합니다.
- accepted input sequence는 user-room별로 증가하고 accepted rate는 user별 token budget을 넘지 않습니다.
- snapshot backlog는 client별 latest one value로 제한되고 transport congestion은 bounded termination으로 수렴합니다.

## 2. 핵심 질문

- elapsed wall-clock과 fixed simulation timestep은 어떤 accumulator·clamp 산술로 분리됩니까?
- heartbeat interval과 timeout handle은 누가 만들고 acknowledge/stop 때 어떻게 교체·해제합니까?
- stale input과 rate-limited input 중 어느 검사가 먼저이며 그 순서가 token budget에 어떤 영향을 줍니까?
- latest snapshot replacement, soft retry, hard termination의 조건과 최초 callback 지연 가정은 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3a2943ff385d` | `feat(game): fixed-step scheduler 추가` | A | SIMULATION, REALTIME, OBSERVABILITY | monotonic elapsed time을 bounded 50 ms simulation work로 변환하는 fixed-step accumulator를 도입합니다. |
| 2 | `0888e119036d` | `test(game): fixed-step 보정 범위 검증` | B | SIMULATION, REALTIME, TEST | elapsed monotonic time을 fixed 50 ms work로 바꾸는 경계와 catch-up 상한을 deterministic clock으로 검증합니다. |
| 3 | `10a656e59864` | `feat(game): WebSocket heartbeat 추가` | A | REALTIME, RISK | WebSocket 연결의 ping 주기, 응답 deadline, 종료 cleanup을 단일 heartbeat lifecycle로 만듭니다. |
| 4 | `81031dcd2c1c` | `test(game): heartbeat timeout 검증` | B | REALTIME, TEST | ping cadence, 45초 timeout, acknowledge deadline reset을 fake timer로 검증합니다. |
| 5 | `207df3f47935` | `feat(game): 입력 순서와 rate limit 보호` | A | SIMULATION, REALTIME, RISK | room별 monotonic input sequence와 user별 token-bucket budget을 하나의 input admission gate로 결합합니다. |
| 6 | `1353e3eb99cc` | `test(game): input gate 제한 검증` | B | REALTIME, TEST | monotonic ordering과 user별 token bucket의 경계·격리를 deterministic clock으로 검증합니다. |
| 7 | `8589ff3c4821` | `feat(game): latest snapshot buffer 추가` | A | SIMULATION, REALTIME, OPERATIONS | 느린 transport에서 snapshot backlog를 누적하지 않고 latest value만 보존하며 congestion을 bounded termination으로 바꿉니다. |
| 8 | `125aa113a01c` | `test(game): snapshot replacement와 congestion 검증` | A | REALTIME, PERF, RISK | latest-value replacement, soft retry, hard termination, congestion timeout을 controlled socket state와 fake time으로 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): fixed-step scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `3a2943ff385d` |
| Importance | A |
| Tags | SIMULATION, REALTIME, OBSERVABILITY |
| Source에서 확정된 역할 | monotonic elapsed time을 bounded 50 ms simulation work로 변환하는 fixed-step accumulator를 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 symbol: `FixedStepScheduler`, `DEFAULT_TIMESTEP_MS`, `MAX_STEPS_PER_TICK`
- `start`, `stop`, 내부 timer callback이 `now()`와 `lastTimeMs`, `accumulatedMs`를 갱신하는 순서를 확인합니다.
- elapsed가 음수일 때 0으로 취급하고, 긴 지연 뒤 누적량을 `timestep * maxSteps`로 제한하는 산술을 확인합니다.
- 한 callback에서 실행할 step 수를 최대 5회로 제한한 뒤 남은 누적량을 어떻게 처리하는지 확인합니다.
- 주입 가능한 monotonic clock과 timer API가 production 시간과 deterministic test를 어떻게 분리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:3a2943ff385d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub의 각 room은 wall-clock interval callback이 호출될 때마다 곧바로 한 번 simulation을 진행했습니다. callback이 늦거나 몰릴 때 실제 경과 시간과 simulation step 수의 관계를 설명할 별도 소유자가 없었습니다. |
| 해결하려던 문제와 위험 | event loop 지연을 그대로 따라잡으려 하면 한 callback에서 무제한 작업이 발생하고, 반대로 항상 한 번만 진행하면 simulation 시간이 실제 경과와 크게 어긋납니다. 또한 시스템 시계 역행이 음수 elapsed를 만들 수 있었습니다. |
| 핵심 구현 결정 | `FixedStepScheduler`가 monotonic `now()` 차이를 accumulator에 더하고 50 ms 단위로 step callback을 실행합니다. 누적 지연은 최대 250 ms, 한 callback의 step은 최대 5회로 제한하며 음수 elapsed는 0으로 보정합니다. timer와 clock을 주입할 수 있어 계산 규칙을 transport와 분리합니다. |
| 입력 → 상태 전이 → 출력 | `start` → 현재 monotonic 시각 저장 → interval callback → `max(0, now-last)` 계산 → 누적량을 250 ms 이하로 clamp → 누적량이 50 ms 이상인 동안 최대 5회 `step(50)` → 사용한 시간을 accumulator에서 차감합니다. |
| ownership/lifetime/cleanup | scheduler 인스턴스가 interval handle, 마지막 관측 시각, 누적 elapsed를 단독 소유합니다. `start`와 `stop`은 중복 호출에 안전하며 `stop`이 handle과 누적 진행 상태를 정리합니다. |
| failure/rollback/retry | clock이 뒤로 가면 elapsed를 0으로 만들고, event loop가 장시간 멈춰도 과거의 모든 tick을 재생하지 않습니다. step callback 자체의 예외를 복구하는 장치는 이 primitive에 없으므로 caller가 실행 실패 의미를 소유합니다. |
| 보장하는 것 | simulation work는 고정된 50 ms 단위이며 한 timer callback당 최대 5회로 제한됩니다. wall-clock 지연이 즉시 무제한 CPU burst로 변환되지 않습니다. |
| 보장하지 않는 것 | 이 SHA는 scheduler primitive만 추가합니다. GameHub room이 아직 이를 사용하지 않으므로 실제 realtime 경로의 timer topology는 바뀌지 않습니다. |
| 후속 연결 | `0888e119036d`가 보정 산술을 고정하고, `a6a1f4fba60e`에서 각 room의 simulation owner로 처음 통합됩니다. |
<!-- LEARNER-END:3a2943ff385d:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:3a2943ff385d:snippet -->
- SHA: `3a2943ff385d`
- 위치: `apps/api/src/game/fixedStepScheduler.ts`; `FixedStepScheduler`, `DEFAULT_TIMESTEP_MS`, `MAX_STEPS_PER_TICK`

```ts
const elapsedMs = Math.max(0, currentTimeMs - this.lastTimeMs);
this.accumulatedMs = Math.min(this.accumulatedMs + elapsedMs, this.timestepMs * this.maxStepsPerTick);
```
<!-- LEARNER-END:3a2943ff385d:snippet -->

#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `0888e119036d` — `test(game): fixed-step 보정 범위 검증`

### 5.2. `test(game): fixed-step 보정 범위 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `0888e119036d` |
| Importance | B |
| Tags | SIMULATION, REALTIME, TEST |
| Source에서 확정된 역할 | elapsed monotonic time을 fixed 50 ms work로 바꾸는 경계와 catch-up 상한을 deterministic clock으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/fixedStepScheduler.test.ts`
- 핵심 symbol: fake clock/timer를 사용하는 `FixedStepScheduler` test cases
- 49 ms와 50 ms에서 step 호출 횟수가 달라지는 경계 테스트를 확인합니다.
- 10초 지연을 주입했을 때 5회만 실행되고 오래된 lag를 다음 callback으로 무한 이월하지 않는지 확인합니다.
- clock 역행과 `stop` 뒤 callback 중단을 어떤 fake timer 상태로 증명하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:0888e119036d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | fixed-step 산술은 구현됐지만 50 ms 경계, 긴 pause, 시계 역행, stop cleanup을 재현 가능한 방식으로 고정한 증거가 없었습니다. 실제 시간을 기다리는 테스트는 느리고 비결정적이며, accumulator의 off-by-one이나 stale lag 이월을 놓치기 쉽습니다. |
| 구현 또는 검증 결정 | 가짜 monotonic 시각과 timer를 주입해 49/50 ms 경계, 10초 pause, 역행한 시각, stop 이후 상태를 직접 구동합니다. |
| 실행/검증 경로 | scheduler 시작 → fake clock 변경 → timer callback 실행 → 기록된 step duration·호출 횟수 확인 → stop 뒤 추가 callback이 work를 만들지 않는지 확인합니다. |
| ownership과 failure 처리 | 테스트가 clock과 timer 진행을 소유하고 production scheduler는 주입된 API만 사용합니다. 각 case는 scheduler를 중지해 handle을 남기지 않습니다. 10초 경과를 한 번에 주입해 catch-up이 5회로 제한되는지, 다음 정상 tick이 과거 lag를 반복하지 않는지 확인합니다. |
| 보장하는 것 | fixed-step의 산술 경계와 cleanup이 wall-clock timing에 의존하지 않고 회귀 테스트로 고정됩니다. |
| 보장하지 않는 것 | 실제 Node event-loop 지연이나 여러 room의 작업 비용은 측정하지 않습니다. |
| 후속 연결 | `3a2943ff385d`의 primitive를 검증하며, 실제 GameHub 통합은 `a6a1f4fba60e`에서 별도로 검증해야 합니다. |
<!-- LEARNER-END:0888e119036d:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:0888e119036d:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic boundary test |
| 주입·재현 방식 | 주입된 fake clock과 timer callback으로 49/50 ms, 10초 delay, backward time, stop 상태를 직접 재현합니다. |
| 증명하는 것 | 한 callback당 최대 5 step, 음수 elapsed 무시, stop 이후 zero progress를 증명합니다. |
| 증명하지 않는 것 | OS scheduler·실제 event-loop·simulation body의 처리량은 증명하지 않습니다. |
<!-- LEARNER-END:0888e119036d:test -->



#### 비교 기준

- 직전 관련 SHA: `3a2943ff385d` — `feat(game): fixed-step scheduler 추가`
- 다음 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`

### 5.3. `feat(game): WebSocket heartbeat 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `10a656e59864` |
| Importance | A |
| Tags | REALTIME, RISK |
| Source에서 확정된 역할 | WebSocket 연결의 ping 주기, 응답 deadline, 종료 cleanup을 단일 heartbeat lifecycle로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/connectionHeartbeat.ts`
- 핵심 symbol: `ConnectionHeartbeat.start`, `acknowledge`, `stop`, ping/timeout handles
- 15초 ping interval과 마지막 응답 기준 45초 timeout이 어떤 timer들로 구성되는지 확인합니다.
- `acknowledge`가 timeout ownership을 갱신하는 순서와 `start`/`stop`의 멱등성을 확인합니다.
- `socket.ping()`이 throw할 때 즉시 terminate하는 failure branch와 모든 handle cleanup을 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:10a656e59864:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | socket close 이벤트만으로 연결 상실을 알 수 있었고, half-open 연결이나 응답 없는 peer를 bounded time 안에 제거하는 owner가 없었습니다. |
| 해결하려던 문제와 위험 | transport가 열림 상태로 남아도 상대가 사라질 수 있습니다. ping과 deadline timer가 여러 위치에 흩어지면 reconnect·replacement·close 때 timer가 남거나 중복 종료할 위험이 있습니다. |
| 핵심 구현 결정 | `ConnectionHeartbeat`가 15초마다 ping을 보내고 45초 동안 acknowledge가 없으면 socket을 terminate합니다. acknowledge는 timeout deadline을 다시 예약하고, start/stop이 모든 timer handle을 한 객체에서 관리합니다. |
| 입력 → 상태 전이 → 출력 | `start` → ping interval·initial timeout 예약 → ping callback에서 `socket.ping()` → pong/활동 시 `acknowledge`가 timeout 재설정 → deadline 도달 또는 ping 예외 시 terminate → `stop`에서 interval/timeout 제거입니다. |
| ownership/lifetime/cleanup | 연결마다 하나의 heartbeat 인스턴스가 ping interval과 timeout을 소유합니다. transport owner는 연결 생성 시 start하고 disconnect/replacement/close 시 stop해야 합니다. |
| failure/rollback/retry | ping 호출이 동기 예외를 던지면 더 이상 heartbeat를 반복하지 않고 socket을 terminate합니다. 응답 없음도 45초 상한 뒤 같은 terminal path로 수렴합니다. |
| 보장하는 것 | 응답 없는 연결은 bounded deadline 안에 제거되고, timer cleanup을 단일 객체가 멱등적으로 수행할 수 있습니다. |
| 보장하지 않는 것 | primitive는 누가 pong을 acknowledge로 전달하고 언제 stop할지 알지 못합니다. 실제 GameHub client lifecycle 연결은 아직 없습니다. |
| 후속 연결 | `81031dcd2c1c`가 시간 계약을 고정하고, `fc2a4451eed1`이 GameHub client 생성·close에 연결합니다. |
<!-- LEARNER-END:10a656e59864:record -->




#### 비교 기준

- 직전 관련 SHA: `0888e119036d` — `test(game): fixed-step 보정 범위 검증`
- 다음 관련 SHA: `81031dcd2c1c` — `test(game): heartbeat timeout 검증`

### 5.4. `test(game): heartbeat timeout 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `81031dcd2c1c` |
| Importance | B |
| Tags | REALTIME, TEST |
| Source에서 확정된 역할 | ping cadence, 45초 timeout, acknowledge deadline reset을 fake timer로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/connectionHeartbeat.test.ts`
- 핵심 symbol: fake socket의 `ping`/`terminate`, fake timers
- 30초 진행 뒤 ping 두 번, 45초 경계에서 terminate되는 expectation을 확인합니다.
- 중간 `acknowledge`가 기존 timeout을 취소하고 새 45초 deadline을 만드는지 확인합니다.
- stop 또는 ping throw case가 남은 timer를 어떻게 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:81031dcd2c1c:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | heartbeat 구현의 15초/45초 관계와 deadline reset이 실제 시간을 기다리지 않고 검증되지 않았습니다. timer 기반 liveness는 경계 시점과 중복 handle 오류가 흔하며 실제 시간 테스트는 flaky합니다. |
| 구현 또는 검증 결정 | Vitest fake timers와 호출 기록 socket으로 ping 횟수, terminate 시각, acknowledge 후 deadline 이동을 결정적으로 검사합니다. |
| 실행/검증 경로 | heartbeat 시작 → fake time 30초 진행 → ping 호출 확인 → deadline 직전/도달 상태 비교 → acknowledge case에서 deadline 재계산을 확인합니다. |
| ownership과 failure 처리 | 테스트가 fake timer queue를 소유하고 종료 시 timer를 비웁니다. heartbeat의 `stop`이 production handle cleanup을 수행하는지 호출 수로 확인합니다. 응답 없음과 ping 예외를 각각 terminate로 수렴시키는 경계를 재현합니다. |
| 보장하는 것 | heartbeat의 시간 상수와 reset/cleanup 규칙이 deterministic regression으로 고정됩니다. |
| 보장하지 않는 것 | 실제 WebSocket pong event wiring이나 GameHub reconnect semantics는 검증하지 않습니다. |
| 후속 연결 | `10a656e59864`의 primitive 증거이며, `fc2a4451eed1` 이후 통합 경로 테스트가 필요합니다. |
<!-- LEARNER-END:81031dcd2c1c:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:81031dcd2c1c:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic timer test |
| 주입·재현 방식 | Vitest fake timers와 fake socket call spies를 사용합니다. |
| 증명하는 것 | ping 15초 cadence, 45초 timeout, acknowledge deadline reset을 증명합니다. |
| 증명하지 않는 것 | 실제 network half-open 탐지 시간이나 OS TCP behavior는 증명하지 않습니다. |
<!-- LEARNER-END:81031dcd2c1c:test -->



#### 비교 기준

- 직전 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`
- 다음 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`

### 5.5. `feat(game): 입력 순서와 rate limit 보호`

| 항목 | 값 |
| --- | --- |
| SHA | `207df3f47935` |
| Importance | A |
| Tags | SIMULATION, REALTIME, RISK |
| Source에서 확정된 역할 | room별 monotonic input sequence와 user별 token-bucket budget을 하나의 input admission gate로 결합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/inputGate.ts`
- 핵심 symbol: `InputGate.accept`, `releaseUser`, token bucket state와 sequence key
- sequence key가 user와 room을 NUL delimiter로 결합해 room별 monotonic ordering을 유지하는지 확인합니다.
- stale/duplicate sequence를 token 차감 전에 거부하는 순서를 확인합니다.
- 초당 30 token, burst 8의 refill 산술과 같은 user가 여러 room에서 budget을 공유하는지 확인합니다.
- `releaseUser`가 user bucket과 관련 sequence entries를 모두 제거하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:207df3f47935:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub는 도착한 paddle input을 transport 순서대로 처리했으며 duplicate/stale input과 한 사용자의 과도한 명령을 독립적으로 제한하지 않았습니다. |
| 해결하려던 문제와 위험 | 재전송·역순 패킷은 state를 뒤로 움직일 수 있고, 무제한 input은 event loop와 simulation work를 압박합니다. stale 입력이 rate budget까지 소비하면 정상 최신 입력을 부당하게 막을 수 있습니다. |
| 핵심 구현 결정 | `InputGate`가 `(user, room)`별 마지막 `inputSeq`를 먼저 검사하고, 증가한 입력에만 user별 token bucket을 적용합니다. bucket은 초당 30개까지 refill되고 최대 8개 burst를 허용합니다. |
| 입력 → 상태 전이 → 출력 | input 도착 → user/room last sequence 조회 → `inputSeq <= last`면 stale 거부 → 현재 시각으로 user bucket refill → token 없으면 rate 거부 → token 1 차감·last sequence 갱신 → accept입니다. |
| ownership/lifetime/cleanup | InputGate가 user bucket과 user/room sequence map을 소유합니다. caller는 사용자의 마지막 authoritative connection이 제거될 때 `releaseUser`를 호출해야 memory와 budget state가 해제됩니다. |
| failure/rollback/retry | stale 입력은 token을 소비하지 않고, rate 초과 입력은 sequence를 advance하지 않아 나중의 동일보다 큰 sequence가 refill 후 수용될 수 있습니다. unsafe/non-monotonic clock은 주입 clock과 nonnegative elapsed 처리로 제한됩니다. |
| 보장하는 것 | 같은 user-room의 accepted input은 strictly increasing이며, 한 user의 accepted rate는 burst 8과 초당 30 refill로 제한됩니다. |
| 보장하지 않는 것 | 이 primitive 자체는 payload schema, room membership, player side authorization을 검증하지 않습니다. GameHub가 그 선행 조건을 확인해야 합니다. |
| 후속 연결 | `1353e3eb99cc`가 ordering과 budget 상호작용을 검증하고, `fc2a4451eed1`에서 실제 message path에 통합됩니다. |
<!-- LEARNER-END:207df3f47935:record -->




#### 비교 기준

- 직전 관련 SHA: `81031dcd2c1c` — `test(game): heartbeat timeout 검증`
- 다음 관련 SHA: `1353e3eb99cc` — `test(game): input gate 제한 검증`

### 5.6. `test(game): input gate 제한 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1353e3eb99cc` |
| Importance | B |
| Tags | REALTIME, TEST |
| Source에서 확정된 역할 | monotonic ordering과 user별 token bucket의 경계·격리를 deterministic clock으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/inputGate.test.ts`
- 핵심 symbol: 가짜 시각을 사용하는 `InputGate.accept` cases
- burst 8 이후 거부와 시간 진행에 따른 refill을 확인합니다.
- stale sequence를 반복해도 token이 줄지 않는지 확인합니다.
- 한 user의 여러 room이 budget을 공유하고 다른 user는 격리되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1353e3eb99cc:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 입력 gate 구현은 sequence와 rate 두 상태를 결합했지만 검사 순서와 key scope가 regression으로 고정되지 않았습니다. stale input이 token을 소모하거나 room마다 독립 bucket을 만들면 공격자가 제한을 우회하거나 정상 input을 차단할 수 있습니다. |
| 구현 또는 검증 결정 | 주입 clock을 전진시키며 burst/refill을 검사하고, stale sequence·동일 user 다중 room·다른 user cases를 조합합니다. |
| 실행/검증 경로 | 초기 burst consume → 9번째 거부 → fake time 진행 → refill acceptance; 별도 case에서 stale 입력 후 최신 입력 acceptance; room/user key scope 비교입니다. |
| ownership과 failure 처리 | 테스트가 시각과 input sequence를 명시하며 gate state는 case별 새 인스턴스에 격리됩니다. rate exhaustion과 stale ordering을 서로 다른 rejection reason으로 재현하고 stale 경로가 budget에 side effect를 만들지 않는지 확인합니다. |
| 보장하는 것 | ordering 검사가 rate charge보다 먼저이고, budget scope가 user 단위라는 구현 규칙을 고정합니다. |
| 보장하지 않는 것 | WebSocket error event나 GameHub client cleanup은 이 unit test 범위 밖입니다. |
| 후속 연결 | `207df3f47935`의 admission invariant를 보호하며 `400ea1589260`이 실제 GameHub boundary를 추가 검증합니다. |
<!-- LEARNER-END:1353e3eb99cc:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:1353e3eb99cc:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic boundary/isolation test |
| 주입·재현 방식 | 가짜 clock, 여러 user/room key, 연속 sequence를 조합합니다. |
| 증명하는 것 | burst/refill 산술, stale non-consumption, user-scoped budget을 증명합니다. |
| 증명하지 않는 것 | network message ordering·schema validation·authorization은 증명하지 않습니다. |
<!-- LEARNER-END:1353e3eb99cc:test -->



#### 비교 기준

- 직전 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`
- 다음 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`

### 5.7. `feat(game): latest snapshot buffer 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `8589ff3c4821` |
| Importance | A |
| Tags | SIMULATION, REALTIME, OPERATIONS |
| Source에서 확정된 역할 | 느린 transport에서 snapshot backlog를 누적하지 않고 latest value만 보존하며 congestion을 bounded termination으로 바꿉니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 symbol: `LatestSnapshotBuffer.enqueue`, `flush`, `close`, soft/hard buffer limits와 congestion timer
- pending snapshot 하나만 유지하고 새 snapshot이 기존 pending 값을 교체하는지 확인합니다.
- `bufferedAmount`의 soft 256 KiB와 hard 1 MiB 경계, 50 ms retry, 5초 congestion deadline을 확인합니다.
- 이 최초 버전에서 outstanding send callback을 나타내는 `sending` 상태가 congestion 판단에 포함되는지 확인합니다.
- replace/deliver/congestion/connection-closed가 어떤 callback 또는 cleanup path로 관측 가능한지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8589ff3c4821:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 모든 simulation snapshot을 즉시 `socket.send`하면 느린 client마다 outbound queue가 무한히 늘고 오래된 state가 최신 state보다 늦게 전달될 수 있었습니다. |
| 해결하려던 문제와 위험 | snapshot은 중간 값을 모두 보존할 필요가 없지만 control event와 달리 높은 빈도로 생성됩니다. transport pressure가 계속되면 process memory와 latency가 무한히 증가하므로 drop과 terminal threshold가 필요했습니다. |
| 핵심 구현 결정 | buffer는 pending snapshot을 하나만 소유하고 다음 snapshot이 오면 교체합니다. transport `bufferedAmount`가 soft limit 이상이면 retry하며, hard limit 초과 또는 5초 지속 congestion이면 socket을 terminate합니다. 이 SHA에서는 send callback 미완료를 나타내는 `sending`도 새 전송을 막는 조건입니다. |
| 입력 → 상태 전이 → 출력 | `enqueue(payload)` → 기존 pending이 있으면 replace 기록 → pending 저장 → `flush` → socket closed면 drop/cleanup → hard pressure면 terminate → soft pressure 또는 sending이면 retry timer → 전송 가능하면 pending을 꺼내 send callback을 기다리고 이후 다시 flush합니다. |
| ownership/lifetime/cleanup | client별 buffer가 latest payload, enqueue 시각, retry timer, congestion 시작 시각, sending 상태를 단독 소유합니다. `close`는 timer와 pending 값을 버리고 이후 enqueue를 받지 않습니다. |
| failure/rollback/retry | send callback 오류, closed socket, hard buffer, 5초 soft congestion을 각각 terminal 또는 drop path로 처리합니다. 다만 callback 지연 자체를 congestion으로 간주하는 가정은 후속 `d90f17fa765d`에서 잘못된 것으로 판명됩니다. |
| 보장하는 것 | outbound snapshot backlog는 client당 latest one value로 제한되고 실제 buffered pressure는 hard/시간 상한을 갖습니다. |
| 보장하지 않는 것 | 최초 구현은 `sending` 상태와 transport `bufferedAmount`를 동일한 pressure 신호처럼 취급해 callback이 느린 정상 socket도 drop/terminate할 수 있습니다. |
| 후속 연결 | `125aa113a01c`가 최초 semantics를 검증하고 `49ca3e778801`에서 GameHub에 연결됩니다. `d90f17fa765d`/`5cd54767858f`가 callback 지연 오판을 나중에 교정합니다. |
<!-- LEARNER-END:8589ff3c4821:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:8589ff3c4821:fix -->
초기 가정: send callback 미완료도 congestion이다 → 실제 위험: callback 지연과 kernel/userland buffer pressure는 다르다 → `d90f17fa765d`에서 `bufferedAmount`만 pressure source로 남깁니다.
<!-- LEARNER-END:8589ff3c4821:fix -->


#### 비교 기준

- 직전 관련 SHA: `1353e3eb99cc` — `test(game): input gate 제한 검증`
- 다음 관련 SHA: `125aa113a01c` — `test(game): snapshot replacement와 congestion 검증`

### 5.8. `test(game): snapshot replacement와 congestion 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `125aa113a01c` |
| Importance | A |
| Tags | REALTIME, PERF, RISK |
| Source에서 확정된 역할 | latest-value replacement, soft retry, hard termination, congestion timeout을 controlled socket state와 fake time으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.test.ts`
- 핵심 symbol: fake socket의 `bufferedAmount`, delayed send callback, fake timers, observer spies
- 전송 중 여러 snapshot을 enqueue할 때 intermediate 값이 교체되고 latest만 남는지 확인합니다.
- soft pressure에서 retry timer가 진행되고 pressure 해제 뒤 latest snapshot이 전송되는지 확인합니다.
- hard 1 MiB 초과와 5초 지속 congestion이 각각 terminate를 유발하는지 확인합니다.
- 테스트가 당시 `sending`을 congestion 조건으로 기대한다는 점을 후속 fix와 비교합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:125aa113a01c:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | latest buffer는 중요한 resource-boundary였지만 replacement/drop/termination 기준이 socket 구현에 따라 달라질 위험이 있었습니다. |
| 해결하려던 문제와 위험 | 실제 network를 사용하면 bufferedAmount와 callback 시점을 정밀하게 제어하기 어려워 soft/hard 경계 및 5초 deadline regression을 재현하기 어렵습니다. |
| 핵심 구현 결정 | fake socket에서 `bufferedAmount`와 send callback 완료를 직접 제어하고 fake timers로 50 ms retry와 5초 deadline을 진행합니다. observer 호출과 실제 sent payload를 함께 검사합니다. |
| 입력 → 상태 전이 → 출력 | 첫 snapshot enqueue/전송 → callback 또는 buffer pressure를 유지 → 후속 snapshots enqueue → latest replacement 확인 → pressure 해제 또는 deadline 진행 → send/terminate/drop 결과 확인입니다. |
| ownership/lifetime/cleanup | 테스트 fake socket이 transport pressure와 callback completion을 소유하고, buffer가 retry timer와 pending payload를 정리하는지 확인합니다. |
| failure/rollback/retry | soft pressure recovery, hard immediate termination, prolonged pressure termination을 분리해 재현합니다. 당시에는 delayed callback 역시 pressure 경로에 포함됩니다. |
| 보장하는 것 | 최초 latest-buffer의 bounded-memory와 termination 규칙이 결정적으로 검증됩니다. |
| 보장하지 않는 것 | 테스트가 구현 당시의 잘못된 `sending == congestion` 가정도 고정합니다. 따라서 후속 fix에서는 기대값을 의도적으로 변경해야 합니다. |
| 후속 연결 | `8589ff3c4821`의 high-risk semantics 증거이며, `5cd54767858f`가 callback 지연과 실제 buffered congestion을 분리하는 새 regression으로 대체·확장합니다. |
<!-- LEARNER-END:125aa113a01c:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:125aa113a01c:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic backpressure/failure test |
| 주입·재현 방식 | fake `bufferedAmount`, 수동 send callback, fake timers로 replacement·soft/hard congestion을 재현합니다. |
| 증명하는 것 | latest-only retention, retry/timeout, hard termination, cleanup을 증명합니다. |
| 증명하지 않는 것 | 실제 OS socket buffer 크기나 network throughput은 증명하지 않으며 당시 callback 가정을 그대로 반영합니다. |
<!-- LEARNER-END:125aa113a01c:test -->

#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:125aa113a01c:fix -->
이 테스트가 고정한 callback-pressure 가정은 `d90f17fa765d`에서 root cause가 수정되고 `5cd54767858f`에서 새 기대값으로 회귀 보호됩니다.
<!-- LEARNER-END:125aa113a01c:fix -->


#### 비교 기준

- 직전 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:evolution -->
`3a2943ff385d`/`0888e119036d`는 elapsed time을 최대 5개의 50 ms step으로 제한합니다. `10a656e59864`/`81031dcd2c1c`는 connection liveness timer를 한 객체로 묶고, `207df3f47935`/`1353e3eb99cc`는 stale ordering을 token charge보다 먼저 판정합니다. `8589ff3c4821`/`125aa113a01c`는 outbound snapshot을 latest one value로 제한하지만 send callback 미완료를 congestion으로 보는 가정은 후속 integration Thread에서 수정됩니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:failure-links -->
- event-loop 장기 정지 → accumulator/catch-up clamp → 49/50 ms·10초 pause regression
- half-open connection → ping/deadline lifecycle → fake-time timeout·acknowledge reset
- stale flood/rate burst → sequence-first token gate → user/room isolation tests
- outbound backlog → latest replacement·soft/hard pressure → controlled socket failure tests → callback 오판은 `d90f17fa765d`에서 교정
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:ownership -->
`FixedStepScheduler`는 interval과 accumulator를, `ConnectionHeartbeat`는 ping/timeout handles를, `InputGate`는 user bucket과 user-room sequence를, `LatestSnapshotBuffer`는 pending payload·retry/congestion 상태를 소유합니다. 이 단계에서는 GameHub가 아직 이 lifetime들을 실제 client/room lifecycle에 연결하지 않습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:final-state -->
네 limiter primitive와 deterministic unit evidence가 존재합니다. 각 primitive의 work·time·memory 상한은 정의됐지만 실제 room/client 생성·교체·pause·finalization에서 start/stop/release가 정확히 호출되는지는 다음 Thread의 통합 책임입니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:final-flow -->
elapsed time은 fixed-step accumulator로, socket liveness는 heartbeat deadline으로, client input은 sequence+token admission으로, outbound snapshot은 latest buffer+pressure limit으로 각각 독립 처리됩니다. primitive는 상태 결정만 소유하고 domain room 전이와 transport membership은 소유하지 않습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:checks -->
- [x] 50 ms timestep과 최대 5회 catch-up 산술을 설명할 수 있습니다.
- [x] heartbeat의 interval/timeout handle과 acknowledge reset을 추적할 수 있습니다.
- [x] stale input이 token을 소비하지 않는 이유와 user budget scope를 설명할 수 있습니다.
- [x] latest buffer의 soft/hard/time limits와 최초 callback 지연 가정의 한계를 구분할 수 있습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:checks -->
===== END FILE: 03-runtime-limiter-primitives-and-bounded-work.md =====

===== BEGIN FILE: 04-gamehub-runtime-integration-shared-scheduling-and-congestion.md =====
# GameHub runtime 통합, shared scheduling과 congestion

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

독립 limiter를 GameHub의 room/client lifecycle에 연결하고, room별 timer를 shared scheduler로 이전하며, simulation·snapshot cadence와 실제 transport congestion을 분리하는 과정을 복원합니다.

범위 메모: 이 Thread는 GameHub 내부 runtime work와 transport limits를 다룹니다. process drain과 signal shutdown은 다음 Thread, external k6/Toxiproxy와 DB pool error containment는 마지막 Thread로 분리합니다.

### 직접 연결되는 불변식

- room이 runnable일 때만 정확히 한 scheduler membership을 가지며 모든 active room은 하나의 underlying fixed-step clock을 공유합니다.
- client heartbeat·input budget·snapshot buffer는 connection/user scope에 맞춰 생성·이전·해제됩니다.
- authoritative simulation은 20 Hz를 유지하고 normal snapshot delivery는 10 Hz로 분산됩니다.
- send callback 지연은 congestion이 아니며 실제 queued bytes와 frame size만 bounded transport failure를 결정합니다.

## 2. 핵심 질문

- primitive start/stop/release가 ready, pause, reconnect, replacement, finish, close 경로에 어떻게 연결됩니까?
- room별 timer에서 shared timer로 ownership이 이동할 때 runnable membership을 누가 결정합니까?
- 20 Hz simulation과 10 Hz snapshot delivery를 분리하면서 terminal event는 어떻게 보존합니까?
- callback completion, `bufferedAmount`, frame `maxPayload`는 각각 어느 layer가 소유하는 신호입니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a6a1f4fba60e` | `feat(game): fixed-step scheduler를 GameHub에 연결` | A | SIMULATION, REALTIME, OBSERVABILITY | 각 room의 direct interval을 `FixedStepScheduler`로 교체해 simulation time owner를 명시합니다. |
| 2 | `fc2a4451eed1` | `feat(game): heartbeat와 input gate를 GameHub에 연결` | A | PROTOCOL, REALTIME, RISK | connection별 heartbeat와 shared input gate를 GameHub client lifecycle·message path에 통합합니다. |
| 3 | `49ca3e778801` | `feat(game): latest snapshot buffer를 GameHub에 연결` | A | PROTOCOL, REALTIME, RISK | 고빈도 snapshot만 client별 latest buffer로 보내고 control/lifecycle event는 ordinary send path에 유지합니다. |
| 4 | `400ea1589260` | `test(game): GameHub runtime 제한 검증` | B | PROTOCOL, REALTIME, TEST | input gate가 실제 authenticated GameHub message path에서 rate-limited protocol 결과를 만드는지 검증합니다. |
| 5 | `aed88c8a93e0` | `perf(game): scheduler benchmark 실행 경계 추가` | B | REALTIME, OBSERVABILITY, PERF | room별 timer와 shared timer topology를 같은 50 ms cadence·동일 synthetic step work로 비교하는 standalone benchmark를 만듭니다. |
| 6 | `8d24b5e70837` | `perf(game): scheduler benchmark 측정 결과 출력` | B | REALTIME, OBSERVABILITY, PERF | scheduler benchmark를 runtime metadata·measurements·명시적 선택 규칙을 가진 JSON report로 완성합니다. |
| 7 | `d21a47ee92d2` | `refactor(game): shared room scheduler 추가` | A | SIMULATION, REALTIME, REFACTOR | 모든 active room을 하나의 fixed-step clock으로 구동할 수 있는 `SharedRoomScheduler` abstraction을 도입합니다. |
| 8 | `518a8368e28f` | `test(game): shared room scheduler 검증` | B | REALTIME, TEST | 하나의 timer ownership, room 등록·해제, tick 중 membership 변경을 deterministic time으로 검증합니다. |
| 9 | `fb5b1abc97f5` | `refactor(game): GameHub가 shared room scheduler 사용` | A | SIMULATION, REALTIME, REFACTOR | simulation timing ownership을 room별 scheduler에서 GameHub가 소유한 단일 shared scheduler로 이전합니다. |
| 10 | `69fb44d2f0ca` | `test(game): shared scheduler lifecycle 검증` | A | AUTH, SIMULATION, REALTIME | connection loss·reconnect·finish 동안 GameHub의 shared scheduler membership과 cleanup 전이를 검증합니다. |
| 11 | `ad482c200cea` | `fix(game): 부하 중 snapshot cadence 안정화` | A | SIMULATION, REALTIME, PERF | authoritative simulation은 20 Hz로 유지하면서 client snapshot delivery를 10 Hz로 낮추고 room별 slot을 분산합니다. |
| 12 | `db1ae3d47b96` | `test(load): 기본 부하 병목 구간 검증` | B | SIMULATION, REALTIME, OPERATIONS | 여러 room에서 20 Hz simulation과 staggered 10 Hz snapshot delivery가 분리되는지 deterministic GameHub tests로 검증합니다. |
| 13 | `d90f17fa765d` | `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음` | A | REALTIME, PERF, RISK | outstanding WebSocket send callback을 congestion 신호에서 제거하고 실제 `bufferedAmount`만 transport pressure로 사용합니다. |
| 14 | `5cd54767858f` | `test(game): callback 지연과 실제 congestion 구분` | A | REALTIME, PERF, TEST | delayed send callback은 정상 delivery로, 높은 `bufferedAmount`는 실제 congestion으로 처리되는지 분리해 검증합니다. |
| 15 | `8ea18a1b92db` | `fix(realtime): WebSocket transport payload 상한 설정` | A | AUTH, REALTIME, RISK | application pre-auth limit과 동일한 8 KiB를 underlying `ws` server의 `maxPayload`에 설정합니다. |
| 16 | `1afec49052b6` | `test(realtime): oversized WebSocket frame 거부 검증` | B | AUTH, REALTIME, TEST | 실제 server/socket 경로에서 8,193-byte frame이 close code 1009로 거부되는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): fixed-step scheduler를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `a6a1f4fba60e` |
| Importance | A |
| Tags | SIMULATION, REALTIME, OBSERVABILITY |
| Source에서 확정된 역할 | 각 room의 direct interval을 `FixedStepScheduler`로 교체해 simulation time owner를 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 symbol: room의 scheduler field, `startGame`, pause/resume/finalize cleanup 경로
- room 생성 시 scheduler가 어떤 step callback을 캡처하고 simulation state를 갱신하는지 확인합니다.
- waiting→playing에서 `start`, disconnect/pause에서 `stop`, reconnect/resume에서 재시작하는 호출 위치를 추적합니다.
- score terminal, persistence failure, room cleanup 경로가 scheduler를 중지하는지 확인합니다.
- 한 room당 여전히 하나의 underlying timer를 소유한다는 topology 한계를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:a6a1f4fba60e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | fixed-step primitive는 존재했지만 GameHub room은 direct `setInterval` 기반 timing을 계속 사용해 elapsed clamp와 deterministic lifecycle cleanup이 production path에 적용되지 않았습니다. |
| 해결하려던 문제와 위험 | room state transition과 timer start/stop이 분리되면 waiting·paused·finished room이 계속 simulation work를 만들거나 reconnect 후 중복 timer가 생길 수 있습니다. |
| 핵심 구현 결정 | 각 room에 `FixedStepScheduler`를 넣고 50 ms step callback이 authoritative simulation을 진행하도록 바꿉니다. room이 playing 상태에 들어갈 때 start하고 pause/finalization/removal에서 stop합니다. |
| 입력 → 상태 전이 → 출력 | room 생성 → waiting 상태로 scheduler 보유 → 양측 ready로 playing 전이 → scheduler start → fixed step마다 simulation·snapshot/terminal 판단 → disconnect면 pause/stop → reconnect면 resume/start → finish/cleanup이면 stop입니다. |
| ownership/lifetime/cleanup | 이 단계의 timer owner는 여전히 각 room입니다. GameHub는 room lifecycle 전이 시 scheduler start/stop을 호출하고 room map에서 제거하기 전에 handle이 정리되도록 책임집니다. |
| failure/rollback/retry | simulation callback 또는 finish 비동기 작업의 오류가 room timer를 남기지 않도록 terminal/cleanup 경로에서 stop합니다. 다만 room 수만큼 timer가 늘어나는 topology는 해결하지 않습니다. |
| 보장하는 것 | 실제 GameHub simulation도 50 ms fixed-step clamp를 사용하고 room lifecycle과 timer 상태가 연결됩니다. |
| 보장하지 않는 것 | active room N개가 N개의 scheduler/timer를 소유합니다. 전역 event-loop wakeup과 room iteration 비용은 아직 중앙화되지 않습니다. |
| 후속 연결 | `400ea1589260`이 전체 runtime boundary를 검증하고, `d21a47ee92d2`/`fb5b1abc97f5`가 timer ownership을 shared scheduler로 이전합니다. |
<!-- LEARNER-END:a6a1f4fba60e:record -->




#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `fc2a4451eed1` — `feat(game): heartbeat와 input gate를 GameHub에 연결`

### 5.2. `feat(game): heartbeat와 input gate를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `fc2a4451eed1` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, RISK |
| Source에서 확정된 역할 | connection별 heartbeat와 shared input gate를 GameHub client lifecycle·message path에 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/connectionHeartbeat.ts`, `apps/api/src/game/inputGate.ts`
- 핵심 symbol: `GameHub.connect`, client record의 `heartbeat`, input event handler, disconnect/replacement cleanup
- 새 client 생성 시 heartbeat start와 pong/activity acknowledge가 어디서 연결되는지 확인합니다.
- validated `game.input`이 membership/room/side 확인 뒤 `InputGate.accept`를 통과하는 순서를 확인합니다.
- rate 거부가 stable `rate_limited` protocol error로 바뀌고 stale input은 어떤 방식으로 무시되는지 확인합니다.
- 마지막 user connection이 사라질 때만 `releaseUser`를 호출해 replacement가 budget state를 잘못 지우지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:fc2a4451eed1:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | heartbeat와 input gate는 unit 수준으로만 존재해 실제 socket은 timeout되지 않았고 GameHub는 input ordering/rate state를 사용하지 않았습니다. |
| 해결하려던 문제와 위험 | primitive를 잘못 배치하면 unauthenticated/unauthorized input이 budget을 소모하거나, connection replacement 때 old socket cleanup이 new socket의 heartbeat/input state까지 제거할 수 있습니다. |
| 핵심 구현 결정 | client record에 heartbeat를 넣어 connect 시 start하고 close/replacement에서 stop합니다. input handler는 protocol validation과 room ownership을 확인한 뒤 InputGate를 호출하며 rate exhaustion만 client-visible error로 반환합니다. |
| 입력 → 상태 전이 → 출력 | authenticated connect → client/heartbeat 생성·start → message parse·membership 확인 → input gate stale/rate/accept 판정 → accept된 direction만 simulation state에 반영 → pong은 acknowledge → close에서 heartbeat stop, 마지막 connection이면 user gate release입니다. |
| ownership/lifetime/cleanup | heartbeat는 client별, InputGate의 bucket은 user별입니다. GameHub가 둘의 lifecycle adapter이며 connection replacement 시 old client만 stop하고 user-level state는 authoritative connection 존재 여부에 따라 유지합니다. |
| failure/rollback/retry | ping/timeout은 socket terminate로, rate 초과는 protocol `rate_limited`로, stale input은 state 변경 없이 종료됩니다. malformed/forbidden message는 input gate 이전 경계에서 거부됩니다. |
| 보장하는 것 | production WebSocket 경로에서 liveness와 bounded input admission이 실제로 적용되고 cleanup scope가 client/user로 구분됩니다. |
| 보장하지 않는 것 | snapshot delivery는 아직 ordinary send path이고 slow-client backlog는 해결되지 않습니다. |
| 후속 연결 | `49ca3e778801`이 snapshot buffer를 같은 client lifecycle에 추가하고 `400ea1589260`이 input throttling을 end-to-end 검증합니다. |
<!-- LEARNER-END:fc2a4451eed1:record -->




#### 비교 기준

- 직전 관련 SHA: `a6a1f4fba60e` — `feat(game): fixed-step scheduler를 GameHub에 연결`
- 다음 관련 SHA: `49ca3e778801` — `feat(game): latest snapshot buffer를 GameHub에 연결`

### 5.3. `feat(game): latest snapshot buffer를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `49ca3e778801` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, RISK |
| Source에서 확정된 역할 | 고빈도 snapshot만 client별 latest buffer로 보내고 control/lifecycle event는 ordinary send path에 유지합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 symbol: client record의 `snapshots`, snapshot broadcast path, ordinary `send`, client cleanup
- `game.snapshot`만 `LatestSnapshotBuffer.enqueue`로 라우팅되고 queue/error/finished 같은 control event는 직접 send되는지 확인합니다.
- client 생성 시 buffer observer와 socket을 결합하고 close/replacement에서 `snapshots.close()`를 호출하는지 확인합니다.
- ordinary event가 hard buffered threshold를 넘으면 terminate하고 send callback error가 열린 socket을 종료하는지 확인합니다.
- snapshot replacement가 room simulation이나 other-client delivery를 막지 않는지 caller loop를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:49ca3e778801:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 모든 server event가 같은 direct send path를 사용해 snapshot burst가 느린 client의 queue를 누적시킬 수 있었습니다. |
| 해결하려던 문제와 위험 | snapshot은 latest state만 중요하지만 queue match, error, game finished 같은 control event는 손실시키면 lifecycle이 깨집니다. 두 종류를 같은 drop 정책으로 처리할 수 없습니다. |
| 핵심 구현 결정 | client마다 `LatestSnapshotBuffer`를 만들고 snapshot event만 enqueue합니다. control event는 direct send를 유지하되 transport hard pressure와 callback error에서 connection을 종료합니다. |
| 입력 → 상태 전이 → 출력 | simulation tick → snapshot event 생성 → 각 client의 latest buffer enqueue/replace/retry; control event → ordinary send → callback 완료/오류 처리; client close → heartbeat와 snapshot buffer 모두 close입니다. |
| ownership/lifetime/cleanup | GameHub client record가 heartbeat와 snapshots 두 transport resource를 함께 소유합니다. snapshot buffer는 pending/retry를, ordinary send는 해당 호출의 callback을 소유합니다. |
| failure/rollback/retry | snapshot은 soft/hard congestion 규칙을 적용하고, control event는 hard pressure나 send 오류에서 socket을 종료해 silent lifecycle loss를 피합니다. |
| 보장하는 것 | 고빈도 state update backlog는 latest-value로 제한되면서 필수 control event는 임의 drop되지 않습니다. |
| 보장하지 않는 것 | 최초 buffer의 `sending` 오판이 그대로 통합되며, room별 timer topology도 유지됩니다. |
| 후속 연결 | `d90f17fa765d`가 callback pressure 가정을 수정하고 `8ea18a1b92db`가 frame 자체의 transport size 상한을 추가합니다. |
<!-- LEARNER-END:49ca3e778801:record -->




#### 비교 기준

- 직전 관련 SHA: `fc2a4451eed1` — `feat(game): heartbeat와 input gate를 GameHub에 연결`
- 다음 관련 SHA: `400ea1589260` — `test(game): GameHub runtime 제한 검증`

### 5.4. `test(game): GameHub runtime 제한 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `400ea1589260` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, TEST |
| Source에서 확정된 역할 | input gate가 실제 authenticated GameHub message path에서 rate-limited protocol 결과를 만드는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.runtime.test.ts`
- 핵심 symbol: fake socket, AI room setup, `game.input` burst, parsed server errors
- fake socket을 GameHub에 연결하고 AI room을 ready/playing으로 만드는 setup을 확인합니다.
- 연속 9개 input 중 burst 8 이후 stable `rate_limited` error가 한 번 발생하는지 확인합니다.
- unit `InputGate`가 아니라 message parsing·room membership·send path까지 통과하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:400ea1589260:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | InputGate unit tests는 산술을 검증했지만 GameHub가 올바른 user/room key와 protocol error로 연결했는지는 증명하지 못했습니다. 통합 중 gate 호출 위치나 error mapping이 잘못되면 primitive가 있어도 실제 socket 경로가 제한을 우회할 수 있습니다. |
| 구현 또는 검증 결정 | memory repository와 fake socket으로 AI match를 만들고 valid input event를 burst로 주입해 outbound server event를 파싱합니다. |
| 실행/검증 경로 | connect → queue AI → ready → playing → sequence가 증가하는 input 9개 전송 → 8개 accept 후 rate-limited error 확인입니다. |
| ownership과 failure 처리 | 테스트가 socket event delivery와 repository cleanup을 소유하고, GameHub가 client heartbeat/buffer/gate cleanup을 수행합니다. 실제 full message path에서 token exhaustion을 재현하며 stale/unauthorized cases는 이 commit의 중심이 아닙니다. |
| 보장하는 것 | GameHub가 rate limit primitive를 우회하지 않고 stable protocol error로 노출함을 증명합니다. |
| 보장하지 않는 것 | real WebSocket transport, heartbeat timeout, outbound congestion은 검증하지 않습니다. |
| 후속 연결 | `fc2a4451eed1`의 integration evidence이며 이후 shared scheduler 변경과 독립적으로 input admission을 보호합니다. |
<!-- LEARNER-END:400ea1589260:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:400ea1589260:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | GameHub integration test |
| 주입·재현 방식 | fake socket과 memory repository로 실제 event parser·room setup·input handler·server error path를 실행합니다. |
| 증명하는 것 | burst 8 이후 9번째 valid input이 rate-limited로 관측됨을 증명합니다. |
| 증명하지 않는 것 | network throughput·multi-process rate limiting·persistent state는 증명하지 않습니다. |
<!-- LEARNER-END:400ea1589260:test -->



#### 비교 기준

- 직전 관련 SHA: `49ca3e778801` — `feat(game): latest snapshot buffer를 GameHub에 연결`
- 다음 관련 SHA: `aed88c8a93e0` — `perf(game): scheduler benchmark 실행 경계 추가`

### 5.5. `perf(game): scheduler benchmark 실행 경계 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `aed88c8a93e0` |
| Importance | B |
| Tags | REALTIME, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | room별 timer와 shared timer topology를 같은 50 ms cadence·동일 synthetic step work로 비교하는 standalone benchmark를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/scheduler-benchmark.mjs`
- 핵심 symbol: `measure(strategy, roomCount)`, `simulateRoomStep`, percentile helpers
- room counts 1/20/50/100, 50 ms timestep, 250 ms warmup, 기본 1.5초 duration과 3 repeats 설정을 확인합니다.
- `room` 전략은 room마다 interval을 만들고 `shared` 전략은 하나의 interval에서 모든 room을 순회하는지 확인합니다.
- 두 전략이 같은 `simulateRoomStep` work를 수행하며 p95/p99 lag sample을 어떻게 수집하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:aed88c8a93e0:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | room별 scheduler가 기능적으로 동작했지만 timer 수를 중앙화할 근거와 비교 가능한 측정 경계가 없었습니다. shared scheduler로의 refactor가 단순 취향이 되지 않으려면 work와 cadence를 통제한 topology 비교가 필요했습니다. |
| 구현 또는 검증 결정 | standalone Node script가 room별 interval과 shared interval을 동일한 room count·step workload로 실행하고 callback lag samples를 수집합니다. |
| 실행/검증 경로 | strategy/room count 선택 → warmup → fixed 50 ms callback에서 synthetic room work → p95/p99 sample 산출입니다. |
| ownership과 failure 처리 | benchmark가 생성한 모든 intervals를 배열로 소유하고 duration 종료 뒤 전부 clear합니다. production GameHub state는 사용하지 않습니다. 환경·CPU noise는 repeats와 warmup으로 완화하지만 제거하지 못합니다. 입력값 검증과 결과 decision 출력은 다음 commit에서 완성됩니다. |
| 보장하는 것 | 두 timer topology를 동일 조건으로 비교할 재현 가능한 실행 경계가 생깁니다. |
| 보장하지 않는 것 | 실제 simulation, socket, persistence를 실행하지 않으며 commit 자체는 측정 결과나 선택 결정을 출력하지 않습니다. |
| 후속 연결 | `8d24b5e70837`가 결과 schema와 50-room decision rule을 추가하고 `d21a47ee92d2`가 선택된 shared abstraction을 구현합니다. |
<!-- LEARNER-END:aed88c8a93e0:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:aed88c8a93e0:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | standalone microbenchmark harness |
| 주입·재현 방식 | 동일 synthetic CPU work와 50 ms cadence에서 room-per-timer와 one-shared-timer를 비교합니다. |
| 증명하는 것 | 실행 시 topology별 callback lag를 같은 harness로 측정할 수 있음을 증명합니다. |
| 증명하지 않는 것 | 이 workbook에서는 benchmark를 실행하지 않았으므로 구체 p95 수치나 우열은 runtime evidence로 주장하지 않습니다. |
<!-- LEARNER-END:aed88c8a93e0:test -->



#### 비교 기준

- 직전 관련 SHA: `400ea1589260` — `test(game): GameHub runtime 제한 검증`
- 다음 관련 SHA: `8d24b5e70837` — `perf(game): scheduler benchmark 측정 결과 출력`

### 5.6. `perf(game): scheduler benchmark 측정 결과 출력`

| 항목 | 값 |
| --- | --- |
| SHA | `8d24b5e70837` |
| Importance | B |
| Tags | REALTIME, OBSERVABILITY, PERF |
| Source에서 확정된 역할 | scheduler benchmark를 runtime metadata·measurements·명시적 선택 규칙을 가진 JSON report로 완성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/scheduler-benchmark.mjs`
- 핵심 symbol: top-level measurement loop, median aggregation, `decision.selectedStrategy`
- 각 room count와 두 strategy를 기본 3회 반복하고 p95/p99의 median을 계산하는지 확인합니다.
- Node/platform/CPU/memory와 benchmark settings를 report에 포함해 결과의 실행 환경을 보존하는지 확인합니다.
- 50-room shared p95가 room p95의 105% 이하일 때 shared를 선택하는 decision rule을 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8d24b5e70837:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | benchmark는 samples를 반환했지만 실행 환경, 반복 집계, 선택 기준을 한 결과물로 남기지 않았습니다. 숫자만 출력하면 어떤 runtime에서 어떤 설정으로 측정했는지 재현하기 어렵고, refactor 결정 기준을 사후에 바꿀 수 있습니다. |
| 구현 또는 검증 결정 | 모든 조합을 반복 실행해 median p95/p99를 모으고 runtime/settings/measurements/decision을 JSON으로 출력합니다. 50-room 비교에서 shared p95가 room p95보다 5% 이상 악화되지 않으면 shared를 선택합니다. |
| 실행/검증 경로 | room count×strategy×repeat 실행 → run samples 집계 → median 산출 → 50-room pair 조회 → 1.05 threshold 계산 → JSON report 출력입니다. |
| ownership과 failure 처리 | script가 측정 배열과 runtime metadata를 한 report object에 모으며 production artifact나 repository state를 변경하지 않습니다. 50-room 결과가 없으면 명시적으로 throw합니다. 환경 noise와 short duration은 여전히 결과 해석의 제한입니다. |
| 보장하는 것 | scheduler 선택이 명시된 비교 규칙과 실행 metadata를 가진 재현 가능한 report 형태로 남습니다. |
| 보장하지 않는 것 | 코드가 결과 schema를 정의할 뿐 특정 machine의 수치는 invariant가 아닙니다. 이번 작업에서도 benchmark 명령을 실행하지 않았습니다. |
| 후속 연결 | `d21a47ee92d2`의 shared scheduler abstraction을 선택하는 사전 operational evidence입니다. |
<!-- LEARNER-END:8d24b5e70837:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:8d24b5e70837:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | benchmark/report contract |
| 주입·재현 방식 | 반복 median과 50-room 5% decision threshold를 source에서 확인했습니다. |
| 증명하는 것 | 실행 결과가 strategy selection을 포함한 structured JSON으로 재현될 수 있음을 증명합니다. |
| 증명하지 않는 것 | shared topology가 모든 production load에서 더 빠르다는 보편적 성능 보장은 하지 않습니다. |
<!-- LEARNER-END:8d24b5e70837:test -->



#### 비교 기준

- 직전 관련 SHA: `aed88c8a93e0` — `perf(game): scheduler benchmark 실행 경계 추가`
- 다음 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`

### 5.7. `refactor(game): shared room scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `d21a47ee92d2` |
| Importance | A |
| Tags | SIMULATION, REALTIME, REFACTOR |
| Source에서 확정된 역할 | 모든 active room을 하나의 fixed-step clock으로 구동할 수 있는 `SharedRoomScheduler` abstraction을 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/sharedRoomScheduler.ts`, `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 symbol: `SharedRoomScheduler.register`, `unregister`, `stop`, room-step map
- 첫 room 등록에서 underlying scheduler가 시작되고 마지막 unregister에서 중지되는지 확인합니다.
- room ID→step callback map이 duplicate register와 unregister를 어떻게 처리하는지 확인합니다.
- tick 시작 시 callback values를 복사해 step 도중 register/unregister mutation이 현재 iteration을 깨지 않는지 확인합니다.
- `stop`이 timer와 모든 room membership을 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:d21a47ee92d2:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub integration은 room마다 하나의 FixedStepScheduler를 만들어 active room 수만큼 interval handles를 생성했습니다. |
| 해결하려던 문제와 위험 | room별 timer는 같은 50 ms cadence인데도 callback wakeup과 accumulator state가 room 수에 비례합니다. 한 room step이 lifecycle 중 map을 변경할 때 shared iteration 안정성도 필요합니다. |
| 핵심 구현 결정 | `SharedRoomScheduler`가 하나의 `FixedStepScheduler`와 room callback map을 소유합니다. 첫 등록 시 clock을 시작하고 매 fixed step마다 등록 callback snapshot을 순회하며 마지막 해제에서 clock을 멈춥니다. |
| 입력 → 상태 전이 → 출력 | `register(roomId, step)` → map 추가 → 첫 room이면 underlying start → tick에서 current callbacks 복사·각 step 호출 → `unregister` → map empty면 underlying stop → global `stop`은 timer와 map을 정리합니다. |
| ownership/lifetime/cleanup | timer와 accumulator는 shared scheduler 하나가 소유하고 room은 callback membership만 등록합니다. room lifecycle owner는 등록/해제를 정확히 호출해야 합니다. |
| failure/rollback/retry | step callback 중 room이 자신이나 다른 room을 해제해도 copied callback list가 iteration skip/iterator corruption을 막습니다. callback 예외 containment는 caller 설계에 의존합니다. |
| 보장하는 것 | 동일 cadence의 active rooms가 하나의 timer owner를 공유하고 등록된 callback 집합으로만 simulation work가 발생합니다. |
| 보장하지 않는 것 | 이 SHA는 abstraction만 추가하며 기존 GameHub room별 schedulers를 아직 교체하지 않습니다. |
| 후속 연결 | `518a8368e28f`가 central ownership을 검증하고 `fb5b1abc97f5`가 GameHub에 실제 적용합니다. |
<!-- LEARNER-END:d21a47ee92d2:record -->




#### 비교 기준

- 직전 관련 SHA: `8d24b5e70837` — `perf(game): scheduler benchmark 측정 결과 출력`
- 다음 관련 SHA: `518a8368e28f` — `test(game): shared room scheduler 검증`

### 5.8. `test(game): shared room scheduler 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `518a8368e28f` |
| Importance | B |
| Tags | REALTIME, TEST |
| Source에서 확정된 역할 | 하나의 timer ownership, room 등록·해제, tick 중 membership 변경을 deterministic time으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/sharedRoomScheduler.test.ts`
- 핵심 symbol: fake timers, registered room callbacks, timer count/lifecycle assertions
- 두 room을 등록해도 underlying interval 하나만 존재하는지 확인합니다.
- 한 room 해제 뒤 다른 room은 계속 step하고 마지막 room 해제에서 timer가 중지되는지 확인합니다.
- 첫 callback이 tick 중 unregister를 수행해도 뒤 callback이 skip되지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:518a8368e28f:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | shared scheduler abstraction은 map iteration과 first/last membership transition에 subtle lifecycle bug가 생길 수 있었습니다. tick 중 map mutation은 JavaScript iterator behavior에 기대면 room skip/duplicate step을 만들 수 있고, 마지막 room 뒤 timer가 남으면 zero-use work가 지속됩니다. |
| 구현 또는 검증 결정 | fake timer로 동일 tick의 room callbacks와 timer count를 관찰하고 callback 내부 unregister case를 재현합니다. |
| 실행/검증 경로 | room A/B 등록 → one timer assertion → tick → A callback에서 membership 변경 → B 호출 확인 → 해제 순서에 따른 timer stop 확인입니다. |
| ownership과 failure 처리 | 테스트가 timer queue를 소유하며 scheduler의 map·underlying handle cleanup을 외부 호출 수로 검증합니다. mutation-during-iteration과 last-unregister leak을 결정적으로 재현합니다. |
| 보장하는 것 | central timer와 membership snapshot iteration의 핵심 ownership 규칙을 고정합니다. |
| 보장하지 않는 것 | GameHub가 모든 room state transition에서 register/unregister를 호출하는지는 검증하지 않습니다. |
| 후속 연결 | `d21a47ee92d2`의 abstraction evidence이고 `69fb44d2f0ca`가 GameHub recovery lifecycle까지 확장합니다. |
<!-- LEARNER-END:518a8368e28f:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:518a8368e28f:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic scheduler lifecycle test |
| 주입·재현 방식 | fake timers와 callback 내 unregister로 one-timer·mutation safety를 검사합니다. |
| 증명하는 것 | first register starts, last unregister stops, tick mutation이 later room을 skip하지 않음을 증명합니다. |
| 증명하지 않는 것 | 실제 room state machine·simulation cost·socket delivery는 증명하지 않습니다. |
<!-- LEARNER-END:518a8368e28f:test -->



#### 비교 기준

- 직전 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`
- 다음 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`

### 5.9. `refactor(game): GameHub가 shared room scheduler 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `fb5b1abc97f5` |
| Importance | A |
| Tags | SIMULATION, REALTIME, REFACTOR |
| Source에서 확정된 역할 | simulation timing ownership을 room별 scheduler에서 GameHub가 소유한 단일 shared scheduler로 이전합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/sharedRoomScheduler.ts`
- 핵심 symbol: GameHub의 `roomScheduler`, room schedule/unschedule helpers, `liveStats().scheduledRooms` 관련 경로
- room object에서 per-room scheduler state가 제거되고 GameHub constructor가 하나의 `SharedRoomScheduler`를 소유하는지 확인합니다.
- ready/play, disconnect/pause, reconnect/resume, abandon, finalization, room removal의 모든 경로에서 register/unregister가 대칭인지 추적합니다.
- 동일 room의 중복 등록과 stale cleanup이 scheduler membership을 어떻게 보존하는지 확인합니다.
- `GameHub.close` 또는 최종 cleanup이 shared scheduler를 중지하는지 당시 구현 범위를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:fb5b1abc97f5:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared scheduler abstraction은 존재했지만 GameHub room은 계속 각자의 FixedStepScheduler를 소유했습니다. production timer topology는 아직 room 수에 비례했습니다. |
| 해결하려던 문제와 위험 | ownership 이전에서 한 state transition이라도 unregister를 놓치면 paused/finished room이 계속 step하고, 중복 register는 같은 room을 두 번 진행할 수 있습니다. 반대로 disconnect에서 너무 일찍 제거하면 reconnect 가능한 room이 재개되지 않습니다. |
| 핵심 구현 결정 | GameHub에 하나의 `SharedRoomScheduler`를 두고 runnable room ID와 step callback만 등록합니다. room object의 scheduler를 제거하고 lifecycle transition마다 schedule/unschedule helper를 호출합니다. |
| 입력 → 상태 전이 → 출력 | GameHub 생성 → shared scheduler 하나 생성 → room playing 시 room ID 등록 → shared tick에서 simulation step → pause/disconnect 시 해제 → valid reconnect 시 재등록 → finish/abandon/remove 시 최종 해제입니다. |
| ownership/lifetime/cleanup | GameHub가 timer topology의 유일한 owner가 됩니다. RoomSession은 legal state를 결정하고, GameHub가 그 상태에 맞춰 scheduler membership을 반영하며, shared scheduler는 callback map과 timer handle만 소유합니다. |
| failure/rollback/retry | partial room creation, disconnect, reconnect expiry, finalization failure 같은 여러 exit에서 unregister가 누락되지 않도록 removal helper에 정리 책임을 집중합니다. persistence retry가 진행 중이어도 terminal room은 simulation membership에서 분리됩니다. |
| 보장하는 것 | active runnable room 수와 무관하게 underlying fixed-step timer는 하나이며, room membership이 simulation work의 유일한 진입 조건이 됩니다. |
| 보장하지 않는 것 | snapshot cadence는 아직 모든 simulation tick과 결합돼 있고, slow send callback을 congestion으로 보는 buffer 가정도 남아 있습니다. |
| 후속 연결 | `69fb44d2f0ca`가 reconnect/finish lifecycle의 membership을 검증하고 `ad482c200cea`가 shared tick 위에서 snapshot delivery cadence를 분리합니다. |
<!-- LEARNER-END:fb5b1abc97f5:record -->




#### 비교 기준

- 직전 관련 SHA: `518a8368e28f` — `test(game): shared room scheduler 검증`
- 다음 관련 SHA: `69fb44d2f0ca` — `test(game): shared scheduler lifecycle 검증`

### 5.10. `test(game): shared scheduler lifecycle 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `69fb44d2f0ca` |
| Importance | A |
| Tags | AUTH, SIMULATION, REALTIME |
| Source에서 확정된 역할 | connection loss·reconnect·finish 동안 GameHub의 shared scheduler membership과 cleanup 전이를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.reconnect.test.ts`
- 핵심 symbol: fake socket/repository, scheduled room count, reconnect/finish transition assertions
- 두 player를 room에 연결하고 ready 이후 scheduled room count가 1이 되는 setup을 확인합니다.
- 한 connection이 끊겨 room이 paused/reserved 상태가 될 때 membership이 0으로 내려가는지 확인합니다.
- 같은 user의 reconnect가 기존 side를 회복한 뒤 membership이 다시 1이 되고 새 room을 만들지 않는지 확인합니다.
- match finish/finalization 뒤 membership이 0이고 persistence가 한 번만 실행되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:69fb44d2f0ca:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared scheduler unit test는 map과 timer만 검증했으며 GameHub의 room/reconnect state machine이 membership을 올바르게 반영하는지는 열려 있었습니다. |
| 해결하려던 문제와 위험 | reconnect는 transport ownership을 교체하면서 room은 보존하는 복합 전이입니다. pause에서 계속 step하거나 reconnect가 두 번째 membership을 만들면 simulation과 결과가 중복됩니다. |
| 핵심 구현 결정 | fake timers와 sockets로 실제 GameHub match를 만든 뒤 disconnect, reconnect, terminal finish를 순서대로 진행하고 scheduled room count와 finalization 호출을 확인합니다. |
| 입력 → 상태 전이 → 출력 | room 생성·ready → scheduled=1 → player disconnect → paused·scheduled=0 → replacement/reconnect → same room resume·scheduled=1 → terminal finish → scheduled=0·finalize once입니다. |
| ownership/lifetime/cleanup | test는 transport events와 fake time을 구동하고 GameHub가 room membership, shared timer, reconnect reservation, repository finalization을 조정합니다. |
| failure/rollback/retry | stale socket이나 duplicate reconnect가 추가 room/timer를 만들지 않는지, finish 뒤 scheduler work가 남지 않는지 확인합니다. |
| 보장하는 것 | shared scheduler ownership 이전이 실제 reconnect lifecycle과 결합돼 한 room을 한 번만 scheduling한다는 high-risk invariant를 증명합니다. |
| 보장하지 않는 것 | 다수 room의 snapshot cadence와 transport congestion behavior는 이 테스트 범위가 아닙니다. |
| 후속 연결 | `fb5b1abc97f5`의 ownership transfer를 보호하고 `db1ae3d47b96`이 multi-room cadence를 추가 검증합니다. |
<!-- LEARNER-END:69fb44d2f0ca:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:69fb44d2f0ca:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic GameHub lifecycle integration test |
| 주입·재현 방식 | fake sockets/timers/repository로 ready→disconnect→reconnect→finish 상태를 구동하고 scheduler membership을 관찰합니다. |
| 증명하는 것 | room scheduler membership 1→0→1→0, same-room recovery, finalize-once를 증명합니다. |
| 증명하지 않는 것 | 실제 network latency, process restart, PostgreSQL transaction은 증명하지 않습니다. |
<!-- LEARNER-END:69fb44d2f0ca:test -->



#### 비교 기준

- 직전 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`
- 다음 관련 SHA: `ad482c200cea` — `fix(game): 부하 중 snapshot cadence 안정화`

### 5.11. `fix(game): 부하 중 snapshot cadence 안정화`

| 항목 | 값 |
| --- | --- |
| SHA | `ad482c200cea` |
| Importance | A |
| Tags | SIMULATION, REALTIME, PERF |
| Source에서 확정된 역할 | authoritative simulation은 20 Hz로 유지하면서 client snapshot delivery를 10 Hz로 낮추고 room별 slot을 분산합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/observability.ts`
- 핵심 symbol: shared tick sequence, snapshot divisor/room delivery slot, finalization observer counters
- 50 ms simulation step은 그대로 실행하면서 snapshot 생성/전송은 매 두 번째 tick에만 수행하는지 확인합니다.
- room ID 또는 등록 순서에서 alternating delivery slot을 정해 같은 tick의 snapshot burst를 분산하는지 확인합니다.
- terminal/finished event가 reduced snapshot cadence 때문에 지연되거나 누락되지 않는지 별도 control path를 확인합니다.
- finalization metrics가 created/duplicate 의미를 구분하도록 함께 조정된 부분을 실제 diff에서 분리해 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:ad482c200cea:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared scheduler가 모든 room을 같은 20 Hz tick에서 순회하고 매 tick마다 snapshot을 보내므로, 50 room에서는 동일 callback에 delivery burst가 집중됐습니다. |
| 해결하려던 문제와 위험 | simulation frequency를 낮추면 authoritative physics가 바뀌지만, snapshot을 그대로 20 Hz로 유지하면 serialization·WebSocket delivery가 event loop를 압박합니다. 모든 room을 같은 delivery tick에 보내는 것도 burst를 만듭니다. |
| 핵심 구현 결정 | simulation step은 50 ms마다 계속 수행하고 snapshot cadence만 divisor 2로 분리해 10 Hz로 만듭니다. room마다 alternating slot을 배정해 절반씩 다른 shared tick에 snapshot을 전달합니다. |
| 입력 → 상태 전이 → 출력 | shared scheduler 20 Hz tick → 모든 runnable room simulation step → room delivery slot과 tick parity 비교 → 해당 slot room만 snapshot enqueue → terminal state면 cadence와 무관하게 finish/finalization path 실행입니다. |
| ownership/lifetime/cleanup | GameHub shared tick이 simulation sequence와 delivery slot 결정을 소유합니다. `LatestSnapshotBuffer`는 선택된 snapshot의 client별 delivery만 소유하며 simulation state를 소유하지 않습니다. |
| failure/rollback/retry | 이전에는 synchronized burst가 callback lag와 snapshot drop을 늘릴 수 있었습니다. 수정은 work를 시간축에 분산하지만 transport pressure가 실제로 높은 client는 기존 buffer termination 규칙을 계속 적용합니다. |
| 보장하는 것 | physics는 20 Hz로 유지되고 normal snapshot delivery는 room당 10 Hz이며 room 간 burst가 두 slot으로 분산됩니다. |
| 보장하지 않는 것 | 10 Hz/두 slot은 현재 부하 목표에 맞춘 고정 정책입니다. send callback 지연 오판은 아직 남아 있으며 `d90f17fa765d`에서 별도로 수정됩니다. |
| 후속 연결 | `db1ae3d47b96`가 multi-room 20 Hz/10 Hz 분리를 검증하고 `547d9943d30a`가 load harness 측정 자체를 안정화합니다. |
<!-- LEARNER-END:ad482c200cea:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:ad482c200cea:fix -->
이전 가정: simulation tick마다 모든 room snapshot을 보내도 된다 → 실제 위험: shared tick에 delivery burst 집중 → 수정: 20 Hz simulation과 10 Hz staggered snapshot cadence 분리 → `db1ae3d47b96` 회귀 테스트.
<!-- LEARNER-END:ad482c200cea:fix -->


#### 비교 기준

- 직전 관련 SHA: `69fb44d2f0ca` — `test(game): shared scheduler lifecycle 검증`
- 다음 관련 SHA: `db1ae3d47b96` — `test(load): 기본 부하 병목 구간 검증`

### 5.12. `test(load): 기본 부하 병목 구간 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `db1ae3d47b96` |
| Importance | B |
| Tags | SIMULATION, REALTIME, OPERATIONS |
| Source에서 확정된 역할 | 여러 room에서 20 Hz simulation과 staggered 10 Hz snapshot delivery가 분리되는지 deterministic GameHub tests로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.snapshotCadence.test.ts`, `apps/api/src/gameHub.reconnect.test.ts`, `apps/api/src/observability.finalization.test.ts`, `tests/load/*` 관련 조정
- 핵심 symbol: 두 AI room fake-timer scenario, snapshot counts, finalization observer assertions
- 두 active room을 만들고 네 simulation ticks 동안 각 room이 두 snapshot만 받는지 확인합니다.
- 두 room의 delivery slot이 번갈아 실행돼 한 shared tick에 한 room delivery만 발생하는지 확인합니다.
- simulation state는 네 번 advance하면서 snapshot count만 두 번인지 구분합니다.
- finalization observer가 created/duplicate 결과를 실제 production callback에서 어떻게 기록하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:db1ae3d47b96:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | cadence fix는 구현됐지만 simulation 횟수와 delivery 횟수를 분리해 검증하는 multi-room regression이 없었습니다. 단순 snapshot count만 보면 simulation까지 10 Hz로 낮아진 버그를 놓칠 수 있고, 한 room test만으로는 slot staggering을 증명할 수 없습니다. |
| 구현 또는 검증 결정 | fake timers로 두 AI room을 동시에 구동해 simulation tick과 socket snapshot payload를 별도로 기록합니다. four ticks/two snapshots per room과 alternating aggregate delivery를 검사합니다. |
| 실행/검증 경로 | 두 room ready → fake time으로 4×50 ms advance → simulation sequence/state progress 확인 → room별 snapshot 2개와 tick별 분산 확인 → cleanup/finalization observer 확인입니다. |
| ownership과 failure 처리 | 테스트가 fake time과 fake sockets를 소유하고 GameHub shared scheduler·room slots·buffers가 실제 production path를 실행합니다. delivery burst regression, simulation cadence 저하, duplicate finalization observation을 서로 다른 assertions로 감지합니다. |
| 보장하는 것 | 20 Hz authoritative update와 10 Hz staggered delivery의 분리가 multi-room 상황에서 고정됩니다. |
| 보장하지 않는 것 | 실제 500-connection k6 throughput이나 OS event-loop p95는 실행하지 않습니다. |
| 후속 연결 | `ad482c200cea`의 fix regression이며 `547d9943d30a`는 외부 load scenario의 reconnect/finalization 측정을 추가 안정화합니다. |
<!-- LEARNER-END:db1ae3d47b96:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:db1ae3d47b96:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic multi-room load-boundary test |
| 주입·재현 방식 | fake timers와 두 AI room/fake sockets로 simulation tick과 emitted snapshot을 별도 집계합니다. |
| 증명하는 것 | 4 simulation ticks 동안 room별 2 snapshots와 alternating delivery slot을 증명합니다. |
| 증명하지 않는 것 | 실제 production load 수치, network backpressure, PostgreSQL 성능은 증명하지 않습니다. |
<!-- LEARNER-END:db1ae3d47b96:test -->



#### 비교 기준

- 직전 관련 SHA: `ad482c200cea` — `fix(game): 부하 중 snapshot cadence 안정화`
- 다음 관련 SHA: `d90f17fa765d` — `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`

### 5.13. `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`

| 항목 | 값 |
| --- | --- |
| SHA | `d90f17fa765d` |
| Importance | A |
| Tags | REALTIME, PERF, RISK |
| Source에서 확정된 역할 | outstanding WebSocket send callback을 congestion 신호에서 제거하고 실제 `bufferedAmount`만 transport pressure로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 symbol: `LatestSnapshotBuffer.flush`, 제거된 `sending` gate, send callback continuation
- parent에서 `sending`이 flush/retry/congestion elapsed에 어떤 영향을 주는지 비교합니다.
- 수정 후 outstanding callback이 있어도 `bufferedAmount`가 낮으면 후속 latest snapshot을 보낼 수 있는지 확인합니다.
- soft/hard thresholds와 congestion deadline은 그대로 `bufferedAmount`에만 적용되는지 확인합니다.
- 여러 callback 완료 순서가 pending snapshot cleanup과 close state를 깨지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:d90f17fa765d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 최초 buffer는 send callback이 아직 호출되지 않은 상태를 socket buffer pressure와 동일하게 취급해 retry/congestion timer를 시작했습니다. |
| 해결하려던 문제와 위험 | `ws.send` callback 지연은 event-loop callback scheduling이나 library completion 시점일 수 있으며 `bufferedAmount`가 낮은 정상 transport에서도 발생합니다. 이를 congestion으로 보면 snapshot을 불필요하게 drop하고 5초 뒤 healthy connection을 terminate할 수 있습니다. |
| 핵심 구현 결정 | `sending`을 pressure gate에서 제거하고 flush 판단을 `socket.bufferedAmount`와 connection state에만 의존하도록 바꿉니다. callback은 error reporting과 후속 flush trigger 역할만 유지합니다. |
| 입력 → 상태 전이 → 출력 | snapshot enqueue → `bufferedAmount` hard/soft 검사 → 낮으면 outstanding callback 여부와 무관하게 send → callback 완료 시 error 처리·pending flush 재시도; pressure timer는 실제 buffered bytes가 soft limit 이상일 때만 시작합니다. |
| ownership/lifetime/cleanup | buffer는 pending latest value와 retry/congestion state를 계속 소유하지만 send callback 개수 자체를 exclusive send lock으로 소유하지 않습니다. transport가 callback completion을 비동기로 통지합니다. |
| failure/rollback/retry | 실제 buffered pressure는 기존 soft retry/hard terminate/5초 deadline으로 계속 제한됩니다. delayed callback만으로는 drop reason이나 termination이 발생하지 않습니다. |
| 보장하는 것 | callback latency와 queued-byte congestion을 구분해 정상 connection을 오판하지 않으면서 실제 outbound memory 상한은 유지합니다. |
| 보장하지 않는 것 | transport library가 `bufferedAmount`를 정확히 반영한다는 전제는 남습니다. frame 하나의 크기 상한은 아직 plugin layer에 강제되지 않습니다. |
| 후속 연결 | `5cd54767858f`가 delayed callbacks와 true pressure를 분리해 검증하고 `8ea18a1b92db`가 frame-size bound를 추가합니다. |
<!-- LEARNER-END:d90f17fa765d:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:d90f17fa765d:fix -->
이전 가정: callback 미완료 == congestion → 실제 failure: 낮은 bufferedAmount에서도 callback delay로 drop/terminate → root cause: completion signal과 queued bytes 혼동 → 수정: `bufferedAmount`만 pressure source → regression `5cd54767858f`.
<!-- LEARNER-END:d90f17fa765d:fix -->


#### 비교 기준

- 직전 관련 SHA: `db1ae3d47b96` — `test(load): 기본 부하 병목 구간 검증`
- 다음 관련 SHA: `5cd54767858f` — `test(game): callback 지연과 실제 congestion 구분`

### 5.14. `test(game): callback 지연과 실제 congestion 구분`

| 항목 | 값 |
| --- | --- |
| SHA | `5cd54767858f` |
| Importance | A |
| Tags | REALTIME, PERF, TEST |
| Source에서 확정된 역할 | delayed send callback은 정상 delivery로, 높은 `bufferedAmount`는 실제 congestion으로 처리되는지 분리해 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.test.ts`
- 핵심 symbol: 수동 callback completion fake socket, bufferedAmount pressure cases, delivered/drop/terminate spies
- `bufferedAmount=0`인 채 여러 send callbacks를 지연하고 snapshots가 drop/terminate 없이 전송되는지 확인합니다.
- callbacks를 순서와 다르게 완료해도 buffer state가 깨지지 않는지 확인합니다.
- soft pressure에서는 latest one value만 유지되고 pressure 해제 뒤 전달되는 기존 semantics가 남는지 확인합니다.
- hard/time congestion tests가 callback delay case와 명확히 분리되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:5cd54767858f:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 기존 `125aa113a01c` 테스트는 delayed callback을 pressure로 취급하는 구현 기대값을 포함했습니다. |
| 해결하려던 문제와 위험 | fix가 callback gate만 제거하고 true backpressure protections까지 약화시키거나 concurrent callback completion에서 stale pending을 전송할 위험이 있었습니다. |
| 핵심 구현 결정 | fake socket에서 callbacks를 보류하되 buffered bytes는 0으로 유지하는 case와, buffered bytes를 soft/hard limit로 올리는 case를 분리합니다. observer와 termination 호출을 각각 확인합니다. |
| 입력 → 상태 전이 → 출력 | low-buffer delayed callbacks + multiple enqueue → sends continue/no congestion; callbacks complete → state clean; 별도 high-buffer case → latest replacement/retry → pressure 해제 또는 terminal threshold 결과 확인입니다. |
| ownership/lifetime/cleanup | 테스트가 callback completion과 `bufferedAmount`를 독립 제어해 두 신호의 ownership을 분리합니다. buffer cleanup은 close/pressure path에서 검증됩니다. |
| failure/rollback/retry | 과거 false-positive congestion을 직접 재현하고, 실제 soft/hard pressure regression도 동시에 보호합니다. |
| 보장하는 것 | callback delay는 congestion이 아니며 queued-byte pressure만 drop/retry/terminate를 유발한다는 수정된 invariant를 증명합니다. |
| 보장하지 않는 것 | 실제 network stack의 callback와 bufferedAmount 상관관계는 fake socket 밖에서 측정하지 않습니다. |
| 후속 연결 | `d90f17fa765d`의 root-cause fix를 보호하고 최초 `125aa113a01c`의 잘못된 가정을 의도적으로 교정합니다. |
<!-- LEARNER-END:5cd54767858f:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:5cd54767858f:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic regression/failure discrimination test |
| 주입·재현 방식 | send callback completion과 `bufferedAmount`를 독립적으로 제어하는 fake socket을 사용합니다. |
| 증명하는 것 | delayed callback no-drop/no-terminate와 true congestion latest-only/termination을 구분해 증명합니다. |
| 증명하지 않는 것 | 실제 kernel buffer behavior나 WAN latency에서의 throughput은 증명하지 않습니다. |
<!-- LEARNER-END:5cd54767858f:test -->

#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:5cd54767858f:fix -->
false positive를 재현하는 regression test이며, 수정 전 test expectation과 달리 low-buffer callback delay를 정상으로 고정합니다.
<!-- LEARNER-END:5cd54767858f:fix -->


#### 비교 기준

- 직전 관련 SHA: `d90f17fa765d` — `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`
- 다음 관련 SHA: `8ea18a1b92db` — `fix(realtime): WebSocket transport payload 상한 설정`

### 5.15. `fix(realtime): WebSocket transport payload 상한 설정`

| 항목 | 값 |
| --- | --- |
| SHA | `8ea18a1b92db` |
| Importance | A |
| Tags | AUTH, REALTIME, RISK |
| Source에서 확정된 역할 | application pre-auth limit과 동일한 8 KiB를 underlying `ws` server의 `maxPayload`에 설정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/ws-ticket.test.ts`
- 핵심 symbol: WebSocket plugin options의 `maxPayload`, 기존 pre-authentication payload constant
- parent에서 application message buffer가 8 KiB를 검사해도 transport가 frame 전체를 먼저 수신하는지 확인합니다.
- plugin registration이 기존 pre-auth 상수와 동일한 `maxPayload`를 사용하는지 확인합니다.
- limit 초과 frame이 application JSON parser나 auth buffer에 도달하기 전에 `ws` close 1009로 종료되는지 확인합니다.
- text/binary frame 모두 transport limit 적용 범위에 포함되는지 library boundary를 실제 설정으로 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8ea18a1b92db:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | pre-auth message handler는 누적 payload 8 KiB를 제한했지만 underlying WebSocket server는 더 큰 frame을 먼저 메모리에 받아 application callback에 전달할 수 있었습니다. |
| 해결하려던 문제와 위험 | application-level length check만으로는 frame allocation과 parser 진입 전 resource consumption을 막지 못합니다. 인증 전 공격자가 oversized frame을 반복할 수 있었습니다. |
| 핵심 구현 결정 | Fastify WebSocket/`ws` server registration에 `maxPayload`를 기존 8 KiB 상수로 설정해 transport parser 자체가 oversized frame을 거부하도록 합니다. |
| 입력 → 상태 전이 → 출력 | client frame 수신 → `ws` parser가 frame size와 8 KiB maxPayload 비교 → 초과면 application handler 호출 없이 protocol close 1009 → 허용 크기만 pre-auth/authenticated message path로 전달됩니다. |
| ownership/lifetime/cleanup | frame-size enforcement는 application buffer가 아니라 transport server가 소유합니다. application은 여전히 인증 전 메시지 수·누적 상태와 schema validation을 소유합니다. |
| failure/rollback/retry | 8,193-byte frame은 transport close로 종료돼 JSON parsing, ticket lookup, GameHub connect에 도달하지 않습니다. |
| 보장하는 것 | single WebSocket frame이 8 KiB를 넘으면 allocation 이후 application processing으로 확대되지 않고 protocol-level rejection을 받습니다. |
| 보장하지 않는 것 | 허용 크기 이하의 다수 frame, decompression 설정, connection 수 자체는 다른 limit가 소유합니다. |
| 후속 연결 | `1afec49052b6`가 실제 authenticated WebSocket으로 8,193-byte close 1009를 검증합니다. |
<!-- LEARNER-END:8ea18a1b92db:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:8ea18a1b92db:fix -->
이전 가정: application payload check면 충분 → 실제 risk: transport가 oversized frame을 먼저 수신 → 수정: `ws.maxPayload=8 KiB` → real-socket regression `1afec49052b6`.
<!-- LEARNER-END:8ea18a1b92db:fix -->


#### 비교 기준

- 직전 관련 SHA: `5cd54767858f` — `test(game): callback 지연과 실제 congestion 구분`
- 다음 관련 SHA: `1afec49052b6` — `test(realtime): oversized WebSocket frame 거부 검증`

### 5.16. `test(realtime): oversized WebSocket frame 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1afec49052b6` |
| Importance | B |
| Tags | AUTH, REALTIME, TEST |
| Source에서 확정된 역할 | 실제 server/socket 경로에서 8,193-byte frame이 close code 1009로 거부되는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/ws-ticket.test.ts`
- 핵심 symbol: server startup, authenticated WebSocket ticket flow, oversized frame send, close event assertion
- memory repository와 real listening server를 띄우고 인증/ticket을 거쳐 WebSocket을 여는 setup을 확인합니다.
- 정확히 8,193-byte payload를 전송하고 close code 1009를 기다리는지 확인합니다.
- application error event가 아니라 transport close를 관측하며 cleanup에서 socket/app/repository를 닫는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1afec49052b6:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | `maxPayload` 설정은 있었지만 Fastify plugin과 실제 `ws` server까지 전달되는지 unit inspection만으로 확정하기 어려웠습니다. framework option wiring이 잘못되면 설정이 존재해도 real transport는 default limit를 사용할 수 있습니다. |
| 구현 또는 검증 결정 | 실제 local WebSocket connection을 인증한 뒤 limit보다 1 byte 큰 frame을 보내 protocol close event를 검사합니다. |
| 실행/검증 경로 | app listen → HTTP auth/ticket → WebSocket upgrade → 8,193-byte send → `ws` parser rejection → client close code 1009 → resources close입니다. |
| ownership과 failure 처리 | 테스트가 실제 process-local server/socket lifetime을 소유하고 after cleanup에서 열린 handles를 닫습니다. transport parser의 message-too-big path를 실제 frame으로 재현합니다. |
| 보장하는 것 | 8 KiB 설정이 framework에서 real WebSocket transport까지 적용됨을 증명합니다. |
| 보장하지 않는 것 | 8,192-byte boundary acceptance, compressed frame amplification, distributed ingress limits은 이 테스트가 증명하지 않습니다. |
| 후속 연결 | `8ea18a1b92db`의 transport-layer fix를 직접 보호합니다. |
<!-- LEARNER-END:1afec49052b6:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:1afec49052b6:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | real-socket integration regression test |
| 주입·재현 방식 | 인증된 실제 WebSocket에 8,193-byte frame을 보내 close code 1009를 관찰합니다. |
| 증명하는 것 | application handler 이전 transport size rejection이 실제 wiring에서 동작함을 증명합니다. |
| 증명하지 않는 것 | 모든 frame fragmentation/compression 조합이나 connection-rate defense는 증명하지 않습니다. |
<!-- LEARNER-END:1afec49052b6:test -->



#### 비교 기준

- 직전 관련 SHA: `8ea18a1b92db` — `fix(realtime): WebSocket transport payload 상한 설정`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:evolution -->
`a6a1f4fba60e`~`49ca3e778801`은 room/client에 fixed-step, heartbeat/input gate, snapshot buffer를 실제 연결합니다. `aed88c8a93e0`/`8d24b5e70837`이 timer topology 비교 경계를 만든 뒤 `d21a47ee92d2`/`fb5b1abc97f5`가 one-clock ownership을 도입·이전하고 `69fb44d2f0ca`가 recovery 전이를 고정합니다. `ad482c200cea`/`db1ae3d47b96`은 20 Hz simulation과 staggered 10 Hz delivery를 분리합니다. `d90f17fa765d`/`5cd54767858f`는 callback 지연 오판을 교정하고, `8ea18a1b92db`/`1afec49052b6`는 8 KiB frame bound를 transport layer에서 강제합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:failure-links -->
- primitive 미통합/cleanup 누락 → GameHub lifecycle wiring → runtime integration tests
- room별 timer 증식 → controlled benchmark boundary → shared scheduler abstraction/ownership transfer → reconnect lifecycle test
- synchronized snapshot burst → 20 Hz simulation/10 Hz staggered delivery → multi-room cadence regression
- send callback false congestion → `bufferedAmount`-only fix → delayed callback vs real pressure regression
- application-only size check → transport `maxPayload` → real-socket 1009 regression
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:ownership -->
초기에는 room이 timer를 소유했지만 최종적으로 GameHub의 `SharedRoomScheduler`가 단 하나의 clock을 소유하고 room은 runnable callback membership만 가집니다. client record는 heartbeat와 latest buffer를 소유하고, InputGate는 user-level budget을 소유합니다. `ws` server는 frame-size enforcement를, buffer는 queued-byte pressure와 latest payload를 소유합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-state -->
GameHub는 한 shared fixed-step clock으로 runnable rooms를 20 Hz 진행하고 snapshot만 10 Hz two-slot cadence로 전달합니다. client별 heartbeat/input/snapshot limits가 lifecycle에 연결되며 callback 지연은 congestion으로 취급되지 않습니다. 8 KiB 초과 frame은 application handler 전에 transport가 거부합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-flow -->
authenticated connect가 heartbeat와 snapshot buffer를 만들고 user-level input gate를 공유합니다. room이 playing이 되면 shared scheduler에 등록되고 매 50 ms simulation이 진행됩니다. tick parity에 맞는 room만 snapshot을 latest buffer에 enqueue합니다. queued bytes가 실제 threshold를 넘을 때만 retry/terminate하며 oversized frame은 `ws` parser에서 1009로 종료됩니다. pause/finish/close는 scheduler membership과 client resources를 정리합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:checks -->
- [x] room별 timer에서 shared timer로 ownership이 이동한 전후를 설명할 수 있습니다.
- [x] ready/pause/reconnect/finish마다 scheduler membership이 어떻게 변하는지 추적할 수 있습니다.
- [x] simulation frequency와 snapshot frequency가 왜 분리됐는지 설명할 수 있습니다.
- [x] callback delay, buffered bytes, frame size를 서로 다른 pressure signal로 구분할 수 있습니다.
- [x] 각 fix를 해당 regression test와 연결할 수 있습니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:checks -->
===== END FILE: 04-gamehub-runtime-integration-shared-scheduling-and-congestion.md =====

===== BEGIN FILE: 05-draining-readiness-and-graceful-shutdown.md =====
# Draining readiness와 graceful shutdown

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

새 work admission을 즉시 닫고 active rooms를 bounded time 동안 drain한 뒤 process resources를 한 번만 정리하며, external container kill budget까지 application drain과 정렬하는 종료 lifecycle을 복원합니다.

범위 메모: Compose 설정을 참조하지만 일반적인 image/release delivery가 아니라 application drain 보장의 외부 상한이므로 이 카테고리에 포함합니다. production artifact 구성 자체는 카테고리 09에 남습니다.

### 직접 연결되는 불변식

- drain 시작 즉시 readiness가 not-ready가 되고 새 queue/tournament/AI work는 거부됩니다.
- 기존 active rooms는 최대 60초 동안 완료할 수 있지만 shutdown은 무기한 기다리지 않습니다.
- 반복 SIGTERM/SIGINT는 하나의 drain-and-close sequence만 시작합니다.
- container termination grace는 application의 60초 drain budget보다 짧지 않습니다.

## 2. 핵심 질문

- readiness lifecycle과 GameHub admission state는 어떤 호출에서 동시에 draining으로 전이됩니까?
- waiting work와 active rooms는 drain에서 왜 다르게 처리됩니까?
- signal 중복·drain timeout·close failure는 어떤 result/exit state로 수렴합니까?
- application timeout과 Compose stop grace의 ownership 관계는 어떻게 검증됩니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `44ef3e07e1a5` | `feat(game): 새 작업 차단과 active room drain 추가` | A | PROTOCOL, REALTIME, TOURNAMENT | GameHub와 readiness가 공유하는 draining state를 도입해 새 matchmaking을 즉시 거부하고 active rooms를 bounded time 동안 기다립니다. |
| 2 | `1c9981393973` | `feat(ops): graceful shutdown 절차 추가` | A | REALTIME, PERSISTENCE, OPERATIONS | SIGTERM/SIGINT를 단일-entry drain→app close 절차로 연결하고 shutdown 오류를 process exit 상태에 반영합니다. |
| 3 | `9d05f47e7f4b` | `test(ops): GameHub drain과 graceful shutdown 검증` | A | REALTIME, OPERATIONS, RISK | drain admission, active-room timeout, readiness transition, repeated signal single-entry를 fake time과 injected signal source로 검증합니다. |
| 4 | `312ddbc6fbe2` | `fix(runtime): container 종료 유예를 room drain과 정렬` | A | REALTIME, OPERATIONS, RISK | API container의 `stop_grace_period`를 70초로 설정해 application의 60초 room drain budget보다 길게 만듭니다. |
| 5 | `73ba979841cd` | `test(docker): API 종료 유예 계약 검증` | B | OPERATIONS, TEST | production Compose duration을 파싱해 API stop grace가 application 60초 drain budget 이상인지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): 새 작업 차단과 active room drain 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `44ef3e07e1a5` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, TOURNAMENT |
| Source에서 확정된 역할 | GameHub와 readiness가 공유하는 draining state를 도입해 새 matchmaking을 즉시 거부하고 active rooms를 bounded time 동안 기다립니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`, `packages/shared/src/ws.ts`
- 핵심 symbol: `app.beginDrain`, `GameHub.beginDrain`, `acceptingMatches`, drain waiters/timer, `server_draining` error
- `app.beginDrain`이 lifecycle flag를 먼저 `draining`으로 바꿔 `/health/ready`를 즉시 503으로 만드는지 확인합니다.
- `GameHub.beginDrain(timeoutMs)`가 queue, AI fallback timers, tournament waiting state를 비우고 새 queue command를 `server_draining`으로 거부하는지 추적합니다.
- 이미 active인 rooms는 즉시 제거하지 않고 room count가 0이 되거나 timeout이 만료될 때 Promise를 resolve하는지 확인합니다.
- 반복 `beginDrain` 호출이 같은 Promise/transition을 재사용하고 timeout handle이 `unref`/cleanup되는지 확인합니다.
- `GameHub.close`가 shared scheduler, reconnect timers, guest result retention, heartbeat, snapshot buffer, sockets를 어떤 순서로 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:44ef3e07e1a5:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | service는 process signal이나 deployment 종료가 시작돼도 readiness가 계속 ready였고, 새 queue/tournament/AI work를 받으면서 기존 rooms와 함께 종료해야 했습니다. |
| 해결하려던 문제와 위험 | 새 작업 admission을 멈추지 않으면 active room 수가 drain 중에도 늘어나 종료가 수렴하지 않습니다. 반대로 모든 socket을 즉시 닫으면 진행 중 match의 authoritative result와 persistence 기회를 잃습니다. |
| 핵심 구현 결정 | app과 GameHub에 explicit draining state를 추가합니다. app은 drain 시작 즉시 readiness lifecycle을 `draining`으로 노출하고, GameHub는 matchmaking admission을 닫아 waiting work를 정리하되 active room은 timeout budget 안에서 finish하도록 둡니다. shared protocol에 stable `server_draining` error가 추가됩니다. |
| 입력 → 상태 전이 → 출력 | `app.beginDrain(60_000)` → lifecycle=`draining` → `hub.beginDrain` → queue/timers/waiters clear·new work reject → active room removal 때 drain waiter 재평가 → rooms=0이면 `{drained:true}` 또는 timeout이면 `{drained:false, activeRooms:n}` resolve → 이후 app close입니다. |
| ownership/lifetime/cleanup | app은 public readiness lifecycle을, GameHub는 queue/room admission과 drain Promise/timer를 소유합니다. 기존 room은 정상 RoomSession/GameHub owner를 유지하며 terminal cleanup이 drain waiter를 알립니다. |
| failure/rollback/retry | active room이 finish하지 않아도 timeout이 상한을 보장합니다. repeated drain은 새 timer를 만들지 않고, new queue attempt는 explicit protocol error를 받습니다. timeout 뒤 remaining room sockets의 최종 처리는 후속 app close가 담당합니다. |
| 보장하는 것 | drain 시작과 동시에 새 work가 차단되고 readiness가 내려가며, 기존 active rooms는 bounded budget 안에서 정상 완료할 기회를 갖습니다. |
| 보장하지 않는 것 | 이 SHA만으로 SIGTERM/SIGINT가 drain을 호출하지 않으며 container가 60초보다 일찍 kill하면 budget을 보장하지 못합니다. |
| 후속 연결 | `1c9981393973`이 process signals를 drain→close에 연결하고 `9d05f47e7f4b`가 admission/timeout/readiness 전이를 검증합니다. `312ddbc6fbe2`가 container budget을 정렬합니다. |
<!-- LEARNER-END:44ef3e07e1a5:record -->




#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`

### 5.2. `feat(ops): graceful shutdown 절차 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1c9981393973` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | SIGTERM/SIGINT를 단일-entry drain→app close 절차로 연결하고 shutdown 오류를 process exit 상태에 반영합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gracefulShutdown.ts`, `apps/api/src/index.ts`
- 핵심 symbol: `installGracefulShutdown`, guarded shutdown callback, signal listener dispose, `app.beginDrain(60_000)`
- `installGracefulShutdown`이 SIGTERM/SIGINT listener를 설치하고 첫 signal의 shutdown Promise만 실행하는 guard를 확인합니다.
- entrypoint shutdown callback이 60초 drain을 기다린 뒤 `app.close()`를 호출하는 순서를 확인합니다.
- drain 또는 close 실패 시 logging/`process.exitCode=1`과 best-effort close가 어떻게 수행되는지 확인합니다.
- app close hook 또는 disposer가 signal listeners를 제거해 테스트·embedded lifecycle에서 중복 listener를 남기지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1c9981393973:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | drain API는 수동 호출할 수 있었지만 process signals는 default Node 종료 동작에 맡겨져 deployment termination과 application lifecycle이 연결되지 않았습니다. |
| 해결하려던 문제와 위험 | signal마다 별도 shutdown을 시작하면 app.close, repository.close, socket cleanup이 중복 실행될 수 있습니다. 오류에서 즉시 `process.exit()`하면 cleanup이 중단되고, 오류를 무시하면 성공 종료로 오인됩니다. |
| 핵심 구현 결정 | signal source와 shutdown callback을 주입받는 `installGracefulShutdown`을 추가합니다. 첫 signal만 async shutdown을 시작하고 후속 signals는 무시합니다. entrypoint는 GameHub drain budget을 기다린 뒤 Fastify를 닫고, 실패를 log·exitCode 1로 기록합니다. |
| 입력 → 상태 전이 → 출력 | SIGTERM 또는 SIGINT → single-entry guard set → `app.beginDrain(60_000)` → drain result log → `app.close()` → Fastify onClose가 GameHub/repository resources 정리 → Promise failure면 error report·exitCode 1·best-effort close입니다. |
| ownership/lifetime/cleanup | signal listeners와 one-shot guard는 installer가 소유하며 disposer가 해제합니다. entrypoint shutdown callback은 app/drain 순서를, Fastify hooks는 하위 repository/GameHub cleanup을 소유합니다. |
| failure/rollback/retry | drain timeout은 정상 결과로 보고 close를 계속합니다. drain/close rejection은 uncaught signal handler exception으로 퍼뜨리지 않고 error callback 및 nonzero exitCode에 기록합니다. repeated signals는 두 번째 cleanup을 시작하지 않습니다. |
| 보장하는 것 | deployment signal이 하나의 bounded drain-and-close sequence로 수렴하고 failure는 성공 exit로 위장되지 않습니다. |
| 보장하지 않는 것 | external orchestrator가 application의 60초 budget보다 긴 grace를 주는지는 이 code가 통제하지 못합니다. |
| 후속 연결 | `9d05f47e7f4b`가 repeated signals와 failure callback을 결정적으로 검증하고, `312ddbc6fbe2`가 Compose grace를 70초로 늘립니다. |
<!-- LEARNER-END:1c9981393973:record -->




#### 비교 기준

- 직전 관련 SHA: `44ef3e07e1a5` — `feat(game): 새 작업 차단과 active room drain 추가`
- 다음 관련 SHA: `9d05f47e7f4b` — `test(ops): GameHub drain과 graceful shutdown 검증`

### 5.3. `test(ops): GameHub drain과 graceful shutdown 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `9d05f47e7f4b` |
| Importance | A |
| Tags | REALTIME, OPERATIONS, RISK |
| Source에서 확정된 역할 | drain admission, active-room timeout, readiness transition, repeated signal single-entry를 fake time과 injected signal source로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.drain.test.ts`, `apps/api/src/gracefulShutdown.test.ts`, `apps/api/src/health.test.ts`
- 핵심 symbol: `beginDrain` cases, `EventEmitter` signal source, fake timers, readiness inject
- waiting player를 넣은 뒤 drain 시작 즉시 queued count가 0이 되고 새 AI/queue command가 `server_draining`을 받는지 확인합니다.
- room이 없을 때 `{drained:true, activeRooms:0}`으로 즉시 resolve하는지 확인합니다.
- active AI room에서 59,999 ms에는 unresolved, 60,000 ms에 `{drained:false, activeRooms:1}`이 되는 fake-timer boundary를 확인합니다.
- SIGTERM→SIGINT→SIGTERM을 연속 emit해 shutdown callback이 한 번, 첫 signal 값으로만 호출되는지 확인합니다.
- drain 직후 `/health/ready`가 503과 `checks.lifecycle='draining'`을 반환하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:9d05f47e7f4b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | drain과 signal code는 여러 interacting state를 다뤘지만 timeout boundary, new admission rejection, repeated signal behavior가 통합 증거로 고정되지 않았습니다. |
| 해결하려던 문제와 위험 | 종료 code는 happy path만 보면 맞아 보이지만 active room이 남거나 signal이 반복될 때 deadlock·duplicate close·new work admission이 발생할 수 있습니다. |
| 핵심 구현 결정 | GameHub에는 fake sockets/repository와 fake timers를, graceful shutdown에는 injected `EventEmitter`와 controlled Promise를 사용합니다. health route는 Fastify inject로 drain 직후 응답을 확인합니다. |
| 입력 → 상태 전이 → 출력 | queue/room setup → beginDrain → immediate state assertions → fake deadline advance → result assertion; 별도 signal test에서 repeated emit → one shutdown → controlled resolve/reject → onError/dispose 확인입니다. |
| ownership/lifetime/cleanup | 테스트가 time, signal emission, socket lifetime, repository close를 명시적으로 소유합니다. production GameHub/app/installer cleanup 호출이 afterEach에 남는 자원 없이 종료되는지 확인합니다. |
| failure/rollback/retry | active room non-progress, repeated signals, shutdown rejection을 각각 결정적으로 재현합니다. raw process signal이나 external SIGKILL은 사용하지 않습니다. |
| 보장하는 것 | drain이 admission을 즉시 닫고 60초 상한을 지키며 readiness를 내리고, signals가 single-entry shutdown으로 수렴함을 증명합니다. |
| 보장하지 않는 것 | 실제 container stop grace, OS signal delivery, PostgreSQL close latency는 이 tests가 증명하지 않습니다. |
| 후속 연결 | `44ef3e07e1a5`/`1c9981393973`의 lifecycle invariant를 보호하며 `73ba979841cd`가 external Compose budget을 정적 계약으로 추가합니다. |
<!-- LEARNER-END:9d05f47e7f4b:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:9d05f47e7f4b:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic lifecycle integration/regression test |
| 주입·재현 방식 | fake timers, fake sockets, memory repository, injected `EventEmitter`, Fastify inject를 조합합니다. |
| 증명하는 것 | queue clearing/new rejection, no-room success, 60초 timeout, readiness 503, repeated-signal once, error reporting을 증명합니다. |
| 증명하지 않는 것 | 실제 Docker daemon의 kill timing이나 process-level signal integration은 증명하지 않습니다. |
<!-- LEARNER-END:9d05f47e7f4b:test -->



#### 비교 기준

- 직전 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`
- 다음 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`

### 5.4. `fix(runtime): container 종료 유예를 room drain과 정렬`

| 항목 | 값 |
| --- | --- |
| SHA | `312ddbc6fbe2` |
| Importance | A |
| Tags | REALTIME, OPERATIONS, RISK |
| Source에서 확정된 역할 | API container의 `stop_grace_period`를 70초로 설정해 application의 60초 room drain budget보다 길게 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.yml`
- 핵심 symbol: production Compose `services.api.stop_grace_period`
- application `beginDrain(60_000)`과 Compose API service의 이전 default/누락된 stop grace를 비교합니다.
- `stop_grace_period: 70s`가 API service에만 적용되고 migration/web/db lifecycle과 혼동되지 않는지 확인합니다.
- orchestrator가 SIGTERM 뒤 강제 SIGKILL하기 전 application drain+close에 남기는 10초 margin의 의미를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:312ddbc6fbe2:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | application은 최대 60초 동안 active rooms를 기다렸지만 Compose에는 그보다 긴 명시적 stop grace가 없어 runtime이 drain 완료 전 강제 종료될 수 있었습니다. |
| 해결하려던 문제와 위험 | 코드 내부 timeout만 길게 설정해도 external orchestrator kill budget이 짧으면 보장이 무효입니다. 이는 application과 deployment configuration 사이의 cross-layer invariant였습니다. |
| 핵심 구현 결정 | production Compose의 API service에 `stop_grace_period: 70s`를 추가해 60초 drain budget보다 10초 긴 termination grace를 제공합니다. |
| 입력 → 상태 전이 → 출력 | Compose stop → API에 SIGTERM → application 최대 60초 drain → Fastify/GameHub/repository close → 70초 grace 만료 전 정상 exit; 그때까지 종료하지 못하면 orchestrator 강제 종료입니다. |
| ownership/lifetime/cleanup | application은 60초 drain과 cleanup을, Compose runtime은 70초 kill deadline을 소유합니다. 10초 차이는 close/logging overhead를 위한 외부 여유입니다. |
| failure/rollback/retry | 이전 mismatch는 active room 결과나 repository cleanup이 SIGKILL로 중단될 위험이었습니다. 설정 수정은 kill deadline을 application budget보다 뒤로 이동시킵니다. |
| 보장하는 것 | 지정 Compose 환경에서 API가 application drain budget 전체를 사용할 최소 termination window가 생깁니다. |
| 보장하지 않는 것 | 호스트 강제 종료, Kubernetes 등 다른 orchestrator, 70초를 넘는 repository close는 보장하지 않습니다. |
| 후속 연결 | `73ba979841cd`가 Compose duration을 파싱해 최소 60초 계약을 회귀 테스트로 고정합니다. |
<!-- LEARNER-END:312ddbc6fbe2:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:312ddbc6fbe2:fix -->
이전 가정: application 60초 timeout이면 충분 → 실제 risk: container grace가 더 짧으면 SIGKILL → 수정: Compose 70초 → static contract `73ba979841cd`.
<!-- LEARNER-END:312ddbc6fbe2:fix -->

#### 최소 코드 근거

<!-- LEARNER-BEGIN:312ddbc6fbe2:snippet -->
- SHA: `312ddbc6fbe2`
- 위치: `docker-compose.yml`; production Compose `services.api.stop_grace_period`

```yaml
stop_grace_period: 70s
```
<!-- LEARNER-END:312ddbc6fbe2:snippet -->

#### 비교 기준

- 직전 관련 SHA: `9d05f47e7f4b` — `test(ops): GameHub drain과 graceful shutdown 검증`
- 다음 관련 SHA: `73ba979841cd` — `test(docker): API 종료 유예 계약 검증`

### 5.5. `test(docker): API 종료 유예 계약 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `73ba979841cd` |
| Importance | B |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | production Compose duration을 파싱해 API stop grace가 application 60초 drain budget 이상인지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/docker-production.test.mjs`
- 핵심 symbol: `parseDurationSeconds`, `services.api.stop_grace_period` assertion
- Compose를 파싱한 기존 production contract test에 `stop_grace_period` assertion이 추가되는 위치를 확인합니다.
- `parseDurationSeconds`가 `XmYs` 형식을 분·초로 변환하고 unsupported value를 fail-closed로 거부하는지 확인합니다.
- assertion이 특정 70초 문자열이 아니라 최소 60초 관계를 고정하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:73ba979841cd:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | Compose에 70초가 설정됐지만 후속 변경이 이를 제거하거나 60초 미만으로 낮춰도 application tests는 감지하지 못했습니다. application timeout과 deployment grace의 관계는 서로 다른 파일에 있어 한쪽만 변경될 때 쉽게 깨집니다. |
| 구현 또는 검증 결정 | production Docker contract test가 parsed Compose API service의 duration을 초로 변환하고 `>= 60`을 요구합니다. |
| 실행/검증 경로 | Compose file read/parse → API service 조회 → duration string validate/seconds convert → application drain minimum과 비교 → failure면 contract test 종료입니다. |
| ownership과 failure 처리 | 정적 contract test가 cross-file duration 관계를 소유하며 실제 container를 시작하지 않습니다. 누락·비문자열·지원하지 않는 duration 또는 60초 미만 값을 assertion failure로 만듭니다. |
| 보장하는 것 | repository configuration에서 API stop grace가 application drain budget 아래로 회귀하지 않습니다. |
| 보장하지 않는 것 | Docker daemon이 duration을 실제로 적용하는지, process가 60초 안에 종료되는지는 실행하지 않습니다. |
| 후속 연결 | `312ddbc6fbe2`의 cross-layer fix를 보호하는 최종 regression입니다. |
<!-- LEARNER-END:73ba979841cd:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:73ba979841cd:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | static deployment contract test |
| 주입·재현 방식 | Compose YAML을 파싱하고 duration string을 초로 환산해 `>=60`을 검사합니다. |
| 증명하는 것 | checked-in production Compose와 application drain budget의 최소 관계를 증명합니다. |
| 증명하지 않는 것 | 실제 container stop·SIGTERM·SIGKILL timing은 증명하지 않습니다. |
<!-- LEARNER-END:73ba979841cd:test -->



#### 비교 기준

- 직전 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:evolution -->
`44ef3e07e1a5`는 readiness와 GameHub admission을 공유하는 drain state를 만들고 active rooms만 bounded wait 대상으로 남깁니다. `1c9981393973`은 SIGTERM/SIGINT를 single-entry 60초 drain→close sequence로 연결하며, `9d05f47e7f4b`가 queue·timeout·readiness·signal edge를 고정합니다. `312ddbc6fbe2`는 application 바깥의 Compose kill budget을 70초로 정렬하고 `73ba979841cd`가 그 관계를 정적 계약으로 보호합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:failure-links -->
- drain 중 new admission → accepting flag/queue cleanup/server_draining → fake-socket regression
- active room non-progress → 60초 timeout result → 59,999/60,000 ms boundary test
- repeated signals/close rejection → single-entry guard·exitCode/onError → injected signal tests
- external grace < app timeout → Compose 70초 fix → duration contract test
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:ownership -->
app은 readiness lifecycle을, GameHub는 matchmaking admission·active room drain timer/Promise를, signal installer는 listeners와 one-shot guard를, Fastify close hooks는 GameHub/repository cleanup을 소유합니다. Compose runtime은 application 밖에서 70초 강제 종료 deadline을 소유합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:final-state -->
SIGTERM/SIGINT가 오면 readiness와 admission이 즉시 닫히고 waiting work가 제거됩니다. active rooms는 최대 60초 완료 기회를 가지며 이후 app close가 모든 runtime/storage resources를 정리합니다. 중복 signals는 무시되고 Compose는 70초를 제공하며 static test가 최소 관계를 보호합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:final-flow -->
signal → `installGracefulShutdown` one-shot → `app.beginDrain(60_000)` → readiness 503 + GameHub new work reject/queue clear → active room count가 0 또는 timeout → `app.close()` → GameHub scheduler/timers/sockets와 repository close → nonzero exitCode on failure. Compose는 최대 70초 뒤 강제 종료합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:checks -->
- [x] drain에서 waiting work와 active room의 처리 차이를 설명할 수 있습니다.
- [x] readiness 503가 drain 결과를 기다리지 않고 즉시 발생하는 이유를 설명할 수 있습니다.
- [x] repeated signals가 duplicate cleanup을 만들지 않는 구조를 추적할 수 있습니다.
- [x] 60초 application budget과 70초 container grace의 cross-layer invariant를 설명할 수 있습니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:checks -->
===== END FILE: 05-draining-readiness-and-graceful-shutdown.md =====

===== BEGIN FILE: 06-load-fault-recovery-and-pool-error-containment.md =====
# Load·fault recovery와 pool error containment

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Phase 1 상태: frozen authoritative scaffold

## 1. Thread 목표

realtime load acceptance criteria를 test-first로 고정하고 k6/Toxiproxy로 connection·room·reconnect·dependency fault를 재현하며, fault가 PostgreSQL idle pool event로 나타날 때 process crash와 credential leakage를 차단하는 과정을 복원합니다.

범위 메모: 부하와 fault harness는 검증 architecture와도 교차하지만, 여기서는 runtime health signal·measurement source·failure containment의 engineering story만 다룹니다. CI job 구성이나 release artifact는 포함하지 않습니다.

### 직접 연결되는 불변식

- load harness는 connection/reconnect/snapshot/finalization을 bounded thresholds와 명시된 measurement source로 평가합니다.
- fault control은 loopback target만 조작하고 각 run의 성공·실패와 무관하게 proxy reset을 시도합니다.
- readiness는 database down과 recovery를 503/down→200/up으로 관측하며 polling은 bounded deadline을 가집니다.
- PostgreSQL idle client error와 reporter failure는 process exception으로 탈출하지 않고 sanitized bounded metadata만 남깁니다.

## 2. 핵심 질문

- 500 connections·50 rooms profile이 실제 auth/ticket/room/reconnect path를 어떻게 구성합니까?
- client event와 server-side persistence metric 중 finalization evidence owner는 누구입니까?
- Toxiproxy fault sequence와 readiness expected state, timeout, always-reset cleanup은 어떻게 연결됩니까?
- Pool `error` EventEmitter, sanitizer, early logging buffer는 어떤 failure를 각각 containment합니까?

## 3. 완료 기준

- Commit map의 모든 SHA를 `web/ft_transcendence` ancestry에서 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 당시 상태만 설명합니다.
- 파일, symbol, caller/callee, 상태 mutation, ownership, cleanup, failure branch를 실제 코드로 기록합니다.
- Fix는 이전 가정과 root cause를, test/benchmark는 production path와 증명·비증명 범위를 연결합니다.
- 실행하지 않은 command나 benchmark 수치를 runtime evidence로 기록하지 않습니다.
- 마지막 selected SHA까지만 사용해 Thread 최종 owner, invariant, execution flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ff1bffcd5296` | `test(load): 실시간 부하 임계값 정의` | B | REALTIME, PERSISTENCE, OPERATIONS | 500 connections·50 rooms를 기본으로 하는 k6/Toxiproxy harness의 service-level thresholds와 구성 계약을 test-first로 정의합니다. |
| 2 | `7b0b5f086b41` | `test(load): 실시간 fault injection 도구 추가` | A | AUTH, REALTIME, PERSISTENCE | k6 realtime load scenario와 PostgreSQL/edge Toxiproxy control plane을 구현해 connection, room, reconnect, snapshot, finalization, dependency failure를 재현합니다. |
| 3 | `547d9943d30a` | `fix(load): 기본 부하 profile 측정 안정화` | A | REALTIME, OPERATIONS, OBSERVABILITY | reconnect burst와 client-side finalization 오판을 제거해 기본 load profile의 측정 source와 timing을 안정화합니다. |
| 4 | `84bec3bf57ae` | `test(load): fault recovery 검사 자동화` | A | PERSISTENCE, OPERATIONS, PERF | Toxiproxy command와 readiness polling을 순서화해 database/edge failure와 recovery를 versioned JSON report로 자동화합니다. |
| 5 | `335565908920` | `test(load): fault scenario 설정과 report 검증` | B | PERSISTENCE, OPERATIONS, PERF | fault runner의 loopback guard, command ordering, bounded polling, report schema, failure cleanup을 deterministic dependencies로 검증합니다. |
| 6 | `eca21f115c1b` | `fix(db): idle connection pool 오류에서 복구` | A | PERSISTENCE, RISK | PostgreSQL Pool의 idle-client `error` event를 sanitized report로 변환하고 malformed error·reporter failure가 process crash로 번지지 않게 containment합니다. |
| 7 | `493babe1cf30` | `test(db): 안전한 connection pool 오류 처리 검증` | B | PERSISTENCE, TEST | real `pg.Pool` EventEmitter에서 idle error가 sanitized metadata로 관측되고 no/throwing reporter에서도 밖으로 throw되지 않는지 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `test(load): 실시간 부하 임계값 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `ff1bffcd5296` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | 500 connections·50 rooms를 기본으로 하는 k6/Toxiproxy harness의 service-level thresholds와 구성 계약을 test-first로 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-harness.test.mjs`
- 핵심 symbol: `createLoadProfile` expectations, required k6 metrics/source assertions, proxy definition assertions
- 기본 500 connections, 50 rooms, 100 player connections, 495 minimum success와 4분 scenario를 확인합니다.
- connection/reconnect 99%, snapshot p95≤150 ms·p99≤250 ms, normal drop<1%, finalization zero failures/duplicates 등의 thresholds를 확인합니다.
- k6 source에 login, one-time ticket, queue join, ready, sequenced input, serverTime 측정이 필요하다고 source-level로 고정하는지 확인합니다.
- PostgreSQL과 edge proxy가 분리되고 load overlay가 API DB traffic을 proxy로 라우팅하도록 요구하는지 확인합니다.
- 이 SHA에서 imported harness modules가 아직 다음 commit 구현 전이라는 test-first 상태를 명시합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:ff1bffcd5296:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | runtime metrics와 limiter는 있었지만 몇 connection/room에서 어떤 성공률·latency·drop/finalization 조건을 통과해야 하는지 executable load contract가 없었습니다. 부하 도구를 먼저 만들면 측정 가능한 값에 맞춰 목표를 낮출 수 있고, reconnect·finalization·fault path 중 일부를 누락하기 쉽습니다. |
| 구현 또는 검증 결정 | Node contract test가 아직 구현될 `createLoadProfile`, k6 source, Toxiproxy definitions, Compose overlay에 요구하는 숫자·metric names·routes·failure paths를 먼저 고정합니다. |
| 실행/검증 경로 | load contract test import → profile 생성 → exact default/threshold assertions → k6 source scan → proxy command/Compose route assertions입니다. 이 commit alone에서는 implementation modules가 완성되지 않아 suite가 green이라고 추정할 수 없습니다. |
| ownership과 failure 처리 | 테스트 파일이 operational acceptance vocabulary를 소유합니다. 실제 connection/socket/proxy/process lifetime은 후속 harness가 소유합니다. 잘못된 connection-room ratio, 누락된 SLI, 단일 proxy로 DB/edge failure를 혼합하는 구성을 assertion failure로 만듭니다. |
| 보장하는 것 | 후속 harness가 충족해야 할 load/fault acceptance contract가 숫자와 metric name 수준으로 명시됩니다. |
| 보장하지 않는 것 | test-first commit은 실제 500 connections를 실행하지 않고, 다음 commit 전에는 참조 구현이 없을 수 있으므로 runtime success evidence가 아닙니다. |
| 후속 연결 | `7b0b5f086b41`이 k6/Toxiproxy implementation을 추가하고 `547d9943d30a`가 측정 자체의 reconnect/finalization bias를 교정합니다. |
<!-- LEARNER-END:ff1bffcd5296:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:ff1bffcd5296:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | test-first static/configuration contract |
| 주입·재현 방식 | profile object, source file patterns, proxy definitions, Compose routing을 Node assertions로 고정합니다. |
| 증명하는 것 | 요구되는 부하 규모·SLI·fault topology가 source contract로 명시됐음을 증명합니다. |
| 증명하지 않는 것 | 실제 k6 실행 결과, threshold 통과, network/database recovery를 증명하지 않습니다. |
<!-- LEARNER-END:ff1bffcd5296:test -->



#### 비교 기준

- 이 commit의 parent 상태와 비교합니다.
- 다음 관련 SHA: `7b0b5f086b41` — `test(load): 실시간 fault injection 도구 추가`

### 5.2. `test(load): 실시간 fault injection 도구 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `7b0b5f086b41` |
| Importance | A |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source에서 확정된 역할 | k6 realtime load scenario와 PostgreSQL/edge Toxiproxy control plane을 구현해 connection, room, reconnect, snapshot, finalization, dependency failure를 재현합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.load.yml`, `tests/load/load-profile.mjs`, `tests/load/pong-load.js`, `tests/load/toxiproxy-control.mjs`
- 핵심 symbol: `createLoadProfile`, k6 `setup/default`, `connectSession`, `buildProxyDefinitions`, `toxicForCommand`, `runCommand`
- load overlay가 Toxiproxy control/edge ports를 loopback에만 publish하고 API `DATABASE_URL`을 PostgreSQL proxy로 바꾸는지 확인합니다.
- `createLoadProfile`이 positive safe integer를 요구하고 connections≥2×rooms, default/extended profile, 99% success count를 계산하는지 확인합니다.
- k6 VU가 dev login→WS ticket→versioned WebSocket→queue/ready/input→disconnect/reconnect→snapshot/finalization을 어떤 순서로 수행하는지 확인합니다.
- snapshot delay가 `Date.now()-serverTimeMs`, delivery gaps가 sequence 차이, reconnect recovery가 same room ID로 측정되는지 확인합니다.
- Toxiproxy commands가 DB latency/down/up와 edge latency/reset/down/up를 서로 다른 proxy/toxic으로 만들고 reset/ensure를 어떻게 수행하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:7b0b5f086b41:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | thresholds는 test-first로 정의됐지만 실제 load processes, connection choreography, dependency fault control은 없었습니다. |
| 해결하려던 문제와 위험 | 단순 WebSocket 연결 수만 세면 auth ticket, matchmaking, active room, sequenced input, reconnect ownership, snapshot freshness, idempotent finalization을 검증하지 못합니다. DB와 edge 장애도 분리 주입해야 원인을 구분할 수 있습니다. |
| 핵심 구현 결정 | k6 profile과 scenario를 추가해 기본 500 VUs 중 100 players가 50 rooms를 만들고 나머지는 online load를 구성합니다. login과 one-time ticket을 거쳐 inputs를 보내고 selected players를 disconnect/reconnect합니다. Toxiproxy overlay와 control script가 PostgreSQL과 public edge를 별도 proxy로 구성합니다. |
| 입력 → 상태 전이 → 출력 | Compose load overlay 기동 → Toxiproxy ensure → k6 setup readiness → VU별 login/ticket/connect → presence가 99% 이상일 때 players queue → matched/ready/input → initial connection close·new ticket reconnect → snapshot/finalization metric 기록; 별도 control command로 DB/edge toxic 적용·reset입니다. |
| ownership/lifetime/cleanup | k6 VU가 자신의 socket, inputSeq, observed room/match sets와 timers를 소유합니다. load profile은 thresholds를, Toxiproxy controller는 proxy definitions/toxics를, Compose overlay는 process routing을 소유합니다. |
| failure/rollback/retry | login/ticket/upgrade/reconnect 실패는 rate metrics로, snapshot gap/delay는 Trend/Rate로, duplicate/failure finalization은 counters로 기록합니다. proxy command validation은 nonpositive/unknown args를 fail-fast합니다. |
| 보장하는 것 | realtime product flow와 DB/edge fault injection을 같은 operational harness에서 재현하고 machine-readable thresholds로 평가할 수 있습니다. |
| 보장하지 않는 것 | 초기 reconnect가 모든 players에게 같은 시점에 몰리고 finalization success를 client `game.finished` event에 의존해 server persistence 결과와 혼동할 수 있습니다. `547d9943d30a`가 이를 교정합니다. |
| 후속 연결 | `547d9943d30a`가 reconnect staggering과 server Prometheus finalization source를 추가하고 `84bec3bf57ae`가 fault sequence 자체를 자동화합니다. |
<!-- LEARNER-END:7b0b5f086b41:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:7b0b5f086b41:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | k6 load harness와 Toxiproxy fault-injection implementation |
| 주입·재현 방식 | k6 VU가 HTTP login·one-time ticket·versioned WebSocket·queue/ready/input·reconnect를 실행하고, Toxiproxy control script가 PostgreSQL/edge latency·down·reset을 적용하도록 구성합니다. |
| 증명하는 것 | source inspection상 요구된 product path와 SLI/fault control을 실행할 harness가 구현됐음을 증명합니다. |
| 증명하지 않는 것 | 이번 작업에서는 k6·Compose·Toxiproxy를 실행하지 않았으므로 threshold 통과나 실제 recovery 결과를 증명하지 않습니다. |
<!-- LEARNER-END:7b0b5f086b41:test -->



#### 비교 기준

- 직전 관련 SHA: `ff1bffcd5296` — `test(load): 실시간 부하 임계값 정의`
- 다음 관련 SHA: `547d9943d30a` — `fix(load): 기본 부하 profile 측정 안정화`

### 5.3. `fix(load): 기본 부하 profile 측정 안정화`

| 항목 | 값 |
| --- | --- |
| SHA | `547d9943d30a` |
| Importance | A |
| Tags | REALTIME, OPERATIONS, OBSERVABILITY |
| Source에서 확정된 역할 | reconnect burst와 client-side finalization 오판을 제거해 기본 load profile의 측정 source와 timing을 안정화합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-profile.mjs`, `tests/load/pong-load.js`, `tests/load/load-harness.test.mjs`
- 핵심 symbol: `PLAYER_RECONNECT_STAGGER_MS`, `reconnectDelayFor`, k6 `teardown`, `readPrometheusSample`
- 기본 reconnect delay가 2초이고 player population 전체에 5초 stagger를 분배하는 산술을 확인합니다.
- connection close timer가 `queue.matched` 직후가 아니라 실제 `playing` snapshot을 본 뒤에만 arm되는지 확인합니다.
- VU의 `game.finished` event 대신 teardown에서 API Prometheus의 database finalization success/failure/duplicate samples를 읽는지 확인합니다.
- `readPrometheusSample`이 metric name과 expected bounded labels를 정확히 matching하고 missing/invalid event-loop metric을 fail-closed 처리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:547d9943d30a:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 harness는 100 player connections가 거의 같은 시점에 끊겨 reconnect stampede를 만들었고, client가 `game.finished`를 받았다는 사실을 durable finalization success처럼 세었습니다. |
| 해결하려던 문제와 위험 | test 자체가 synchronized fault를 만들어 production bottleneck을 과장할 수 있고, transport event 수신은 database commit 성공·duplicate absence를 증명하지 않습니다. |
| 핵심 구현 결정 | player index에 따라 2–7초 범위로 reconnect close를 분산하고, room이 실제 playing snapshot을 보낸 뒤에만 close timer를 설정합니다. finalization 결과는 test 종료 시 server Prometheus counters에서 persistence/outcome labels로 읽습니다. |
| 입력 → 상태 전이 → 출력 | player match/ready → first playing snapshot → `reconnectDelayFor(vuId)` timer → staggered close/reconnect; teardown → `/metrics` scrape → event-loop p95와 finalization success/failure/duplicates parse → k6 metrics에 기록입니다. |
| ownership/lifetime/cleanup | load VU는 reconnect trigger만 소유하고 authoritative finalization outcome은 server metrics owner에게서 읽습니다. teardown parser는 scrape validation과 label selection을 소유합니다. |
| failure/rollback/retry | playing 전에 close해 reconnect window를 잘못 측정하는 case와 client event를 persistence success로 오인하는 case를 제거합니다. required event-loop metric이 없으면 0으로 대체하지 않고 test를 실패시킵니다. |
| 보장하는 것 | 기본 load의 reconnect가 시간축에 분산되고 finalization SLI가 실제 server-side persistence observation을 반영합니다. |
| 보장하지 않는 것 | Prometheus scrape 자체가 성공해야 하며 scrape 시점 이후의 늦은 finalization은 결과에 포함되지 않을 수 있습니다. 실제 run evidence는 별도 실행이 필요합니다. |
| 후속 연결 | `db1ae3d47b96`의 server cadence fix 뒤 measurement harness를 정렬하며 `84bec3bf57ae`의 automated fault recovery와 함께 operational evidence를 강화합니다. |
<!-- LEARNER-END:547d9943d30a:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:547d9943d30a:fix -->
이전 가정: uniform reconnect와 client finished event가 적절한 load/finalization evidence → 실제 failure: reconnect stampede·persistence success 오판 → 수정: playing-gated stagger + server metrics → load contract tests 갱신.
<!-- LEARNER-END:547d9943d30a:fix -->


#### 비교 기준

- 직전 관련 SHA: `7b0b5f086b41` — `test(load): 실시간 fault injection 도구 추가`
- 다음 관련 SHA: `84bec3bf57ae` — `test(load): fault recovery 검사 자동화`

### 5.4. `test(load): fault recovery 검사 자동화`

| 항목 | 값 |
| --- | --- |
| SHA | `84bec3bf57ae` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS, PERF |
| Source에서 확정된 역할 | Toxiproxy command와 readiness polling을 순서화해 database/edge failure와 recovery를 versioned JSON report로 자동화합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/fault-scenario.mjs`, `package.json`
- 핵심 symbol: `createFaultScenarioConfig`, `runFaultScenario`, `observeStep`, `probeReadiness`, `formatFaultReport`, `load:faults`
- loopback-only Toxiproxy/API/edge URLs와 DB 300 ms, edge 150 ms, request 5초, recovery 15초, poll 250 ms defaults를 확인합니다.
- `pollIntervalMs <= recoveryTimeoutMs`와 positive integer/boolean parsing이 fail-fast하는지 확인합니다.
- reset→baseline→DB latency→DB down→DB up/recovery→edge latency→edge reset→edge up/recovery 순서를 확인합니다.
- DB down expected state가 HTTP 503, `status=not_ready`, `checks.database=down`인지 확인합니다.
- scenario error가 나도 final reset을 시도하고 cleanup error를 원래 error와 어떻게 결합하는지 확인합니다.
- report가 schemaVersion, timestamps, settings, targets, ordered steps, passed를 보존하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:84bec3bf57ae:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Toxiproxy commands는 수동으로 호출할 수 있었지만 어떤 순서로 fault를 넣고 어떤 readiness state를 기다린 뒤 recovery를 판단할지 자동화되지 않았습니다. |
| 해결하려던 문제와 위험 | 수동 장애 실험은 cleanup 누락으로 다음 run을 오염시키고, 한 번의 probe로 recovery를 판정하면 timing race가 생깁니다. 실패 관찰과 복구 관찰을 versioned evidence로 남겨야 했습니다. |
| 핵심 구현 결정 | config parser와 scenario runner를 만들고 각 단계에서 bounded polling으로 expected readiness를 기다립니다. 모든 path에서 proxy reset을 시도하며 성공한 observations를 ordered JSON report에 기록합니다. |
| 입력 → 상태 전이 → 출력 | config validate → proxy reset → baseline ready poll → DB latency ready → DB down 503/down → DB up ready → optional edge latency ready → edge reset network/5xx → edge up ready → final reset → all steps passed면 report finalize입니다. |
| ownership/lifetime/cleanup | runner가 fault command ordering, polling deadline, observations, report lifecycle를 소유합니다. Toxiproxy controller가 toxic mutation을, API readiness가 dependency state interpretation을 소유합니다. |
| failure/rollback/retry | command/probe/deadline 실패는 scenario error로 수렴하되 final reset을 별도로 시도합니다. cleanup도 실패하면 원래 error를 잃지 않고 cause로 연결합니다. target URL은 loopback 외부를 거부해 잘못된 환경 조작을 막습니다. |
| 보장하는 것 | database와 edge failure/recovery가 명시된 순서와 bounded deadlines로 자동 실행되고 versioned JSON evidence로 표현됩니다. |
| 보장하지 않는 것 | runner 자체가 production database correctness를 증명하지 않으며, 실제 Toxiproxy/Compose stack이 떠 있어야 end-to-end 실행됩니다. 이 작업에서는 해당 명령을 실행하지 않았습니다. |
| 후속 연결 | `335565908920`이 config/order/report/cleanup을 injected dependencies로 검증하고, 실제 DB idle connection failure containment는 `eca21f115c1b`에서 application code로 수정됩니다. |
<!-- LEARNER-END:84bec3bf57ae:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:84bec3bf57ae:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | automated fault-recovery scenario |
| 주입·재현 방식 | Toxiproxy command와 readiness probe를 reset→baseline→DB latency/down/up→edge latency/reset/up 순서로 실행하고 bounded polling과 final reset, versioned JSON report를 적용합니다. |
| 증명하는 것 | 실행 시 fault/recovery state를 ordered observations와 deadline으로 판정할 orchestration path가 존재함을 증명합니다. |
| 증명하지 않는 것 | 실제 infrastructure를 실행하지 않았으므로 database/edge가 해당 시간 안에 복구됐다는 runtime evidence는 아닙니다. |
<!-- LEARNER-END:84bec3bf57ae:test -->



#### 비교 기준

- 직전 관련 SHA: `547d9943d30a` — `fix(load): 기본 부하 profile 측정 안정화`
- 다음 관련 SHA: `335565908920` — `test(load): fault scenario 설정과 report 검증`

### 5.5. `test(load): fault scenario 설정과 report 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `335565908920` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS, PERF |
| Source에서 확정된 역할 | fault runner의 loopback guard, command ordering, bounded polling, report schema, failure cleanup을 deterministic dependencies로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/fault-scenario.test.mjs`
- 핵심 symbol: injected `applyToxiproxyCommand`, `probeReadiness`, `sleep`, `now`, fake observation queues
- default config와 `FAULT_INCLUDE_EDGE=0`, invalid external URLs를 검사하는 cases를 확인합니다.
- prepared readiness observations가 DB/edge step 순서와 두 번의 250 ms sleep을 만들고 ordered commands를 고정하는지 확인합니다.
- report timestamps, target URLs, step names/status/duration/body/error와 JSON round-trip을 확인합니다.
- `db-down` command가 throw해도 마지막 `reset` command가 호출되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:335565908920:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | fault runner는 여러 injected boundaries와 cleanup path를 가졌지만 실제 proxy/network 없이 그 순서를 결정적으로 검증할 evidence가 없었습니다. end-to-end fault run만으로는 실패 시 cleanup이 실행됐는지, polling이 어떤 observation을 받아 통과했는지 재현하기 어렵습니다. |
| 구현 또는 검증 결정 | command/probe/sleep/clock를 fake로 주입해 fixed observation queue를 소비하고 exact command list, sleep list, report fields를 검사합니다. |
| 실행/검증 경로 | config build → fake commands/probes로 full scenario → expected step list/report 확인; 별도 failure case에서 `db-down` throw → catch → final reset 호출 확인입니다. |
| ownership과 failure 처리 | 테스트가 모든 external side effect를 fake dependency로 소유하고 runner의 orchestration logic만 실행합니다. non-loopback target, invalid config, mid-command failure, cleanup reset을 deterministic assertion으로 재현합니다. |
| 보장하는 것 | fault scenario control logic과 report contract가 실제 infrastructure availability와 무관하게 회귀 보호됩니다. |
| 보장하지 않는 것 | Toxiproxy API, network reset, PostgreSQL readiness가 실제로 동작하는지는 증명하지 않습니다. |
| 후속 연결 | `84bec3bf57ae`의 operational harness evidence이며 `eca21f115c1b`/`493babe1cf30`은 fault가 application pool event로 나타날 때의 containment를 다룹니다. |
<!-- LEARNER-END:335565908920:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:335565908920:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | deterministic fault-orchestration test |
| 주입·재현 방식 | command/probe/sleep/clock dependency injection과 queued observations를 사용합니다. |
| 증명하는 것 | ordered fault/recovery steps, loopback guard, report schema, always-reset cleanup을 증명합니다. |
| 증명하지 않는 것 | 실제 DB/edge outage와 recovery latency는 증명하지 않습니다. |
<!-- LEARNER-END:335565908920:test -->



#### 비교 기준

- 직전 관련 SHA: `84bec3bf57ae` — `test(load): fault recovery 검사 자동화`
- 다음 관련 SHA: `eca21f115c1b` — `fix(db): idle connection pool 오류에서 복구`

### 5.6. `fix(db): idle connection pool 오류에서 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `eca21f115c1b` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| Source에서 확정된 역할 | PostgreSQL Pool의 idle-client `error` event를 sanitized report로 변환하고 malformed error·reporter failure가 process crash로 번지지 않게 containment합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/poolError.ts`, `packages/db/src/index.ts`, `apps/api/src/index.ts`
- 핵심 symbol: `installPostgresPoolErrorHandler`, `toSafePoolErrorEvent`, `safeLabel`, `PostgresRepositoryOptions.onPoolError`, early error buffer
- `Pool.on('error')` listener가 설치되지 않았던 parent와 Node EventEmitter의 unhandled error behavior를 비교합니다.
- raw Error message/connection string 대신 `{kind,errorName,errorCode}`만 만들고 label을 `[A-Za-z0-9_]{1,64}`로 제한하는지 확인합니다.
- malformed error object 변환과 user-supplied reporter callback을 각각 try/catch로 containment하는지 확인합니다.
- `createPostgresRepository`가 `onPoolError` option을 받고 pool 생성 직후 listener를 설치하는 순서를 확인합니다.
- API entrypoint가 Fastify logger 준비 전 발생한 events를 배열에 임시 보관한 뒤 logger owner가 생기면 flush하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:eca21f115c1b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | PostgreSQL pool은 query 중이 아닌 idle client에서 connection failure를 `error` event로 방출할 수 있었고, listener가 없으면 EventEmitter semantics상 process-level uncaught error가 될 수 있었습니다. |
| 해결하려던 문제와 위험 | database outage 실험 중 idle connection error가 readiness 503로만 표현되지 않고 API process를 종료시킬 수 있었습니다. raw error를 그대로 log하면 credentials/host details가 노출되고, reporter 자체가 throw해도 crash가 재발합니다. |
| 핵심 구현 결정 | pool 생성 즉시 error listener를 설치해 safe bounded metadata로 변환합니다. conversion과 optional reporter를 별도 try/catch로 감싸 event boundary 밖으로 exception이 나오지 않게 합니다. API는 app logger가 준비되기 전 events를 buffer하고 준비 후 structured error log로 flush합니다. |
| 입력 → 상태 전이 → 출력 | idle PG client error emit → pool listener → safe event conversion 또는 fallback → optional reporter 호출(throw contained) → startup 전이면 `earlyPoolErrors.push` → app logger 준비 뒤 reporter 교체·buffer flush → 이후 events 직접 structured log입니다. |
| ownership/lifetime/cleanup | DB package가 Pool listener와 sanitization boundary를 소유합니다. API composition root는 logger readiness 전 임시 event buffer와 이후 reporting destination을 소유합니다. repository close는 기존 pool lifetime owner를 유지합니다. |
| failure/rollback/retry | malformed name/code는 fallback `UnknownError`/null로 축소되고 raw message는 전달되지 않습니다. reporter가 throw해도 listener가 삼켜 idle-client failure를 process crash로 바꾸지 않습니다. early buffer는 logger 설치 후 `splice`로 비웁니다. |
| 보장하는 것 | idle pool error가 uncaught EventEmitter error로 process를 종료하지 않고, credential-bearing message 없이 bounded metadata로 관측됩니다. |
| 보장하지 않는 것 | active query failure를 성공으로 바꾸거나 database를 자동 reconnect한다고 보장하지 않습니다. readiness와 client retry는 별도 `pg`/repository behavior입니다. |
| 후속 연결 | `493babe1cf30`이 real `pg.Pool` event와 malicious reporter를 검증합니다. `84bec3bf57ae`의 DB outage/recovery scenario와 함께 process-level containment story를 완성합니다. |
<!-- LEARNER-END:eca21f115c1b:record -->


#### Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:eca21f115c1b:fix -->
이전 가정: query rejection/readiness 처리만으로 DB failure가 containment됨 → 실제 failure: idle client `error` EventEmitter가 uncaught crash 가능 → root cause: pool listener 부재 → 수정: safe listener+reporter containment+early logging buffer → regression `493babe1cf30`.
<!-- LEARNER-END:eca21f115c1b:fix -->


#### 비교 기준

- 직전 관련 SHA: `335565908920` — `test(load): fault scenario 설정과 report 검증`
- 다음 관련 SHA: `493babe1cf30` — `test(db): 안전한 connection pool 오류 처리 검증`

### 5.7. `test(db): 안전한 connection pool 오류 처리 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `493babe1cf30` |
| Importance | B |
| Tags | PERSISTENCE, TEST |
| Source에서 확정된 역할 | real `pg.Pool` EventEmitter에서 idle error가 sanitized metadata로 관측되고 no/throwing reporter에서도 밖으로 throw되지 않는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/poolError.test.ts`
- 핵심 symbol: real `Pool`, `pool.emit('error', error)`, reporter spy/throw cases
- real `new Pool()`에 listener가 정확히 하나 설치되는지 확인합니다.
- secret-bearing message·connectionString과 code `57P01`를 가진 Error를 emit하고 callback payload만 `{kind:'idle_client_error', errorName:'Error', errorCode:'57P01'}`인지 확인합니다.
- serialized mock calls에 `secret`과 `Connection terminated`가 없는 negative assertions를 확인합니다.
- reporter 미설정과 reporter 자체 throw 두 cases 모두 `pool.emit`이 throw하지 않는지 확인합니다.
- 각 case가 `pool.end()`로 pool lifetime을 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:493babe1cf30:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | safe listener implementation은 있었지만 EventEmitter의 실제 `error` semantics, raw credential redaction, reporter failure containment을 함께 고정하지 않았습니다. mock emitter는 Node의 특별한 unhandled `error` behavior를 놓칠 수 있고, sanitizer가 raw Error reference를 payload에 남기면 JSON/log에서 secret이 노출될 수 있습니다. |
| 구현 또는 검증 결정 | 실제 `pg.Pool` 인스턴스에 handler를 설치하고 crafted error를 직접 emit합니다. reporter call shape와 serialized negative substrings, no-reporter/throwing-reporter non-throw behavior를 검사합니다. |
| 실행/검증 경로 | Pool 생성 → handler install/listener count → secret-bearing Error emit → no throw → safe callback inspect → pool.end; 별도 cases에서 reporter 없음/throw → no throw → end입니다. |
| ownership과 failure 처리 | 테스트가 Pool 생성·종료를 소유하고 production handler가 event conversion/reporting containment을 실행합니다. unhandled EventEmitter error, credential leakage, reporter-induced crash를 각각 직접 재현할 입력으로 검사합니다. |
| 보장하는 것 | idle pool error boundary가 실제 `pg.Pool`에서 process exception을 방지하고 bounded sanitized metadata만 보고함을 증명합니다. |
| 보장하지 않는 것 | 실제 PostgreSQL server disconnect/reconnect나 readiness recovery timing은 실행하지 않습니다. |
| 후속 연결 | `eca21f115c1b`의 root-cause fix를 보호하는 최종 database failure containment regression입니다. |
<!-- LEARNER-END:493babe1cf30:record -->

#### 검증·측정 기록

<!-- LEARNER-BEGIN:493babe1cf30:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | real-library EventEmitter regression test |
| 주입·재현 방식 | 실제 `pg.Pool`에 crafted idle error를 emit하고 callback shape·negative secret strings·throw containment을 검사합니다. |
| 증명하는 것 | listener 존재, safe labels, no raw message/secret, no/throwing reporter non-crash를 증명합니다. |
| 증명하지 않는 것 | 실제 network outage와 pool reconnection 성공은 증명하지 않습니다. |
<!-- LEARNER-END:493babe1cf30:test -->



#### 비교 기준

- 직전 관련 SHA: `eca21f115c1b` — `fix(db): idle connection pool 오류에서 복구`
- 이 Thread의 마지막 selected SHA입니다.

## 6. 불변식의 변화

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:evolution -->
`ff1bffcd5296`은 implementation 전에 load scale와 SLI thresholds를 고정하고 `7b0b5f086b41`이 k6/Toxiproxy harness를 구현합니다. `547d9943d30a`는 synchronized reconnect와 client-side finalization 오판을 playing-gated stagger/server Prometheus source로 수정합니다. `84bec3bf57ae`/`335565908920`은 DB/edge failure→recovery를 bounded polling·always-reset·JSON report로 자동화/검증합니다. `eca21f115c1b`/`493babe1cf30`은 DB fault가 idle pool `error`로 나타나도 safe metadata 관측으로 containment합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:evolution -->

## 7. Failure → Fix → Test 관계

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:failure-links -->
- load 목표/SLI 누락 → test-first threshold contract → k6/Toxiproxy implementation
- reconnect stampede·client finalization 오판 → playing-gated stagger/server metrics → harness contract update
- manual fault experiment 오염·race → ordered bounded runner+always reset → dependency-injected report/cleanup test
- idle pool EventEmitter crash·secret log·reporter throw → safe listener/early buffer → real `pg.Pool` regression
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:failure-links -->

## 8. Ownership·state·cleanup 변화

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:ownership -->
k6 profile은 acceptance thresholds를, 각 VU는 connection/input/reconnect observation을, teardown은 server metric scrape를 소유합니다. Toxiproxy controller는 DB/edge toxic mutation을, fault runner는 ordered polling/report/cleanup을 소유합니다. DB package는 Pool event sanitization/containment을, API composition root는 logger 준비 전 event buffer를 소유합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:ownership -->

## 9. Thread 최종 상태

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:final-state -->
기본 load contract는 500 connections·50 rooms와 connection/reconnect/snapshot/finalization thresholds를 정의합니다. reconnect는 playing 이후 분산되고 durable finalization은 server metrics에서 읽습니다. DB/edge faults는 loopback-only Toxiproxy로 순서화돼 readiness recovery JSON report를 만들며, idle PG pool errors는 process를 죽이지 않고 safe name/code만 log됩니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:final-flow -->
load overlay → Toxiproxy ensure → k6 readiness/login/ticket/ws/rooms/input/reconnect → teardown Prometheus evaluation. fault run은 baseline→DB latency/down/up→edge latency/reset/up→always reset을 polling/report합니다. DB outage 중 idle Pool error는 listener→sanitizer→reporter containment→early buffer/logger로 흐르며 readiness는 별도로 dependency state를 반환합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:execution -->
- 저장소 runtime/test command는 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: exit status 128, `Could not resolve host: github.com`.
- 따라서 test pass, benchmark 수치, k6/Toxiproxy recovery 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 exact selected commit의 diff와 당시 파일을 확인한 정적 historical inspection 결과입니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:checks -->
- [x] test-first load contract와 실제 harness implementation을 구분할 수 있습니다.
- [x] reconnect staggering과 server-side finalization metric으로 바뀐 root cause를 설명할 수 있습니다.
- [x] fault runner의 ordered steps, expected readiness, timeout, always-reset cleanup을 추적할 수 있습니다.
- [x] idle Pool error가 readiness failure와 별개로 process crash가 될 수 있었던 이유를 설명할 수 있습니다.
- [x] sanitizer와 reporter containment이 보장하는 것과 DB reconnect를 보장하지 않는 것을 구분할 수 있습니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:checks -->
===== END FILE: 06-load-fault-recovery-and-pool-error-containment.md =====

===== BEGIN FILE: README.md =====
# 런타임 관측성과 서비스 상태

이 카테고리는 application startup/readiness, migration health, Prometheus observer boundary, event-loop 및 realtime delivery 측정, bounded runtime work, GameHub scheduler ownership, graceful drain, load/fault recovery와 database pool error containment를 다룹니다.

## 범위

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Category: `07-runtime-observability-and-service-health`
- 상태: Phase 1 audit 완료, frozen scaffold
- 제외: 일반적인 Docker/Caddy image 구성, CI 배포 job, release artifact, dependency patch, media asset 생성은 `09-production-delivery-and-release-engineering`의 범위입니다.
- 교차 참조 허용: application drain guarantee를 직접 성립시키는 container grace, runtime health를 직접 검증하는 load/fault harness는 이 카테고리에 포함합니다.

## Phase 1 감사 결과

초기 draft는 3개 Thread, 31개 selected commit으로 구성돼 있었습니다. 실제 `web/ft_transcendence`의 linear history와 `commit/commit-importance.md` 분류를 대조한 뒤 6개 Thread, 57개 unique selected commit으로 보정했습니다.

- Startup/readiness Thread는 기존 10개 commit을 유지했습니다.
- Metrics Thread는 `prom-client` dependency와 event-loop load exposure/threshold evidence를 추가했습니다.
- 기존 runtime-limit Thread는 primitive, GameHub 통합/shared scheduling, drain, load/fault/pool containment의 독립된 이야기로 분리했습니다.
- room별 timer에서 shared scheduler로 이동한 근거인 scheduler benchmark와 deterministic lifecycle tests를 추가했습니다.
- application의 60초 drain과 container의 70초 grace를 하나의 cross-layer invariant로 연결했습니다.
- 기존에 잘못 배열된 late history를 실제 branch 순서에 맞게 이동했습니다. 특히 cadence/load/fault/pool fix와 callback-congestion fix의 순서를 분리했습니다.
- draft에 있던 commit은 제거하지 않았으며, DB pool 관련 commit은 올바른 fault-containment Thread로 이동했습니다.
- 일반 logging redaction, build image, CI delivery와 dependency patch는 이 카테고리의 독립 engineering story가 아니므로 추가하지 않았습니다.

Phase 1 종료 뒤 이 `scaffold/` 파일 집합을 동결했습니다. `completed/`는 동일한 파일명·구조·fixed text를 보존하고 learner-facing block만 채운 사본입니다.

## Thread

1. [Startup·liveness·readiness·storage state](01-startup-liveness-readiness-and-storage-state.md)
2. [Metrics observer boundary와 cardinality](02-metrics-observer-boundaries-and-cardinality.md)
3. [Runtime limiter primitive와 bounded work](03-runtime-limiter-primitives-and-bounded-work.md)
4. [GameHub runtime 통합, shared scheduling과 congestion](04-gamehub-runtime-integration-shared-scheduling-and-congestion.md)
5. [Draining readiness와 graceful shutdown](05-draining-readiness-and-graceful-shutdown.md)
6. [Load·fault recovery와 pool error containment](06-load-fault-recovery-and-pool-error-containment.md)

## 사용 원칙

- 각 문서의 Commit map 순서를 유지합니다.
- exact SHA의 코드와 parent 또는 직전 관련 SHA를 확인합니다.
- 다른 카테고리에서 같은 SHA를 교차 참조하더라도 이 문서의 runtime health/ownership/failure 질문에 맞는 근거만 기록합니다.
- 실행하지 않은 test·benchmark·fault scenario의 결과는 기록하지 않습니다.
- `scaffold/`의 learner block 외 fixed text는 Phase 2에서 변경하지 않습니다.
===== END FILE: README.md =====

