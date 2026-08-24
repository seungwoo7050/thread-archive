===== BEGIN FILE: 01-startup-liveness-readiness-and-storage-state.md =====
# 시작·생존 상태·준비 상태·저장소 상태

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

서비스 초기화와 환경 기본값에서 시작해 마이그레이션 집합과 저장소 상태를 준비 상태로 노출하고, 시작 중 데이터 변경과 운영 환경의 메모리 저장소 대체 실행을 제거하는 과정을 복원합니다.

### 직접 관련된 불변 조건

- 생존 상태는 프로세스 상태를, 준비 상태는 저장소/마이그레이션/수명주기 상태를 반영합니다.
- 마이그레이션이 일치하지 않거나 데이터베이스에 연결할 수 없으면 준비 상태를 실패로 처리해 트래픽 수신을 차단합니다.
- 애플리케이션 시작은 암묵적으로 스키마/초기 데이터를 변경하지 않습니다.
- 운영은 영속 저장소 없이 시작하지 않습니다.

## 2. 핵심 질문

- 프로세스가 살아 있는 것과 트래픽을 받을 준비가 된 것은 어떤 엔드포인트/조건으로 분리됩니까?
- 저장소 준비 상태가 연결 실패, 대기 중 마이그레이션, 불일치 마이그레이션을 어떻게 표현합니까?
- 시작이 시드/마이그레이션을 암묵 실행하지 않을 때 운영자가 사전에 수행해야 하는 명시적 단계는 무엇입니까?
- 운영 영속 저장소 요구 조건과 체험 메모리 모드가 어떤 설정 브랜치로 분리됩니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4b43a284e637` | `feat(api): 실행 환경과 service bootstrap 구성` | B | PERSISTENCE, OPERATIONS | 실행 환경 설정, 저장소, Fastify 애플리케이션, 종료 수명주기를 구성 진입점에서 조립합니다. |
| 2 | `85ac2a949439` | `test(api): 실행 환경 기본값 검증` | B | PROTOCOL, PERSISTENCE, TEST | 설정 우선순위와 로컬 시작 기본값을 고정합니다. |
| 3 | `30aac132e14e` | `feat(db): migration set 상태 검사 추가` | A | PERSISTENCE, OPERATIONS, RISK | 번들/적용된 마이그레이션 집합을 현재, 대기 중, 불일치로 분류합니다. |
| 4 | `2f05d5d79c64` | `feat(db): repository readiness 경계 추가` | A | PERSISTENCE, OPERATIONS | 데이터베이스 사용 가능 상태와 마이그레이션 상태를 저장소 계약으로 노출합니다. |
| 5 | `15002e229acb` | `feat(ops): liveness와 readiness endpoint 추가` | A | PROTOCOL, PERSISTENCE, OPERATIONS | 프로세스 생존 상태와 의존성 상태를 반영하는 서비스 준비 상태를 버전이 명시된 응답으로 분리합니다. |
| 6 | `6937cf60aeea` | `test(ops): health와 database readiness 검증` | B | AUTH, PERSISTENCE, OPERATIONS | DB 중단/마이그레이션 상태와 무관하게 생존 상태가 유지되고 준비 상태만 실패하는지 검증합니다. |
| 7 | `e1a0316fbe84` | `fix(api): startup seed 생성을 제거` | B | PERSISTENCE | API 시작에서 암묵적 초기 데이터 생성 변경을 제거합니다. |
| 8 | `5cac4843fd9b` | `test(api): startup seed 금지 검증` | B | REALTIME, TEST | 진입점이 시드 연산을 호출하지 않는지 검증합니다. |
| 9 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | TOURNAMENT, OPERATIONS, RISK | 운영에서 데이터베이스 URL이 없으면 실행 환경 파싱 단계에서 실패합니다. |
| 10 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | OPERATIONS, TEST | 명시적/추론된 운영 모두 메모리 저장소 대체 실행을 거부하는지 검증합니다. |

## 5. 커밋별 학습 기록

### 5.1. `feat(api): 실행 환경과 service bootstrap 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `4b43a284e637` |
| 중요도 | B |
| 태그 | PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | 실행 환경 설정, 저장소, Fastify 애플리케이션, 종료 수명주기를 구성 진입점에서 조립합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.ts`, `apps/api/src/index.ts`, `apps/api/src/app.ts`
- 핵심 심벌: `readEnv`, `buildApp`, API 진입점의 저장소 선택과 `onClose` 훅
- `readEnv`가 누락된 환경값을 어떤 로컬 기본값으로 바꾸고 숫자·URL·모드를 어디서 검증하는지 확인합니다.
- `index.ts`가 `DATABASE_URL` 유무로 PostgreSQL/메모리 저장소를 선택하고 `buildApp`에 넘기는 순서를 추적합니다.
- 당시에는 시작에서 `ensureSeedData`를 호출한다는 사실과, `app.close()`가 저장소 `close()`로 이어지는 소유권을 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:4b43a284e637:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | API 패키지는 있었지만 실행 시 환경값, 저장소 구현, Fastify 인스턴스와 종료 처리를 한 곳에서 조립하는 명시적 진입점이 없었습니다. 개발 서버를 실제 프로세스로 띄우려면 환경 해석과 저장소 선택, HTTP 서버 시작, 종료 시 자원 반환 순서를 하나의 구성 진입점이 소유해야 했습니다. |
| 구현 또는 검증 결정 | `readEnv`로 런타임 설정을 구성하고, `DATABASE_URL`이 있으면 PostgreSQL 저장소를, 없으면 메모리 저장소를 만든 뒤 `buildApp`에 주입합니다. 이 SHA에서는 저장소 생성 뒤 `ensureSeedData`를 호출하므로 시작이 데이터 변경까지 수행합니다. |
| 실행/검증 경로 | 프로세스 시작 → 환경 해석 → 저장소 생성·시드 → Fastify 애플리케이션 생성 → `listen` → 종료 시 `onClose`에서 저장소 `close` 순서입니다. |
| 소유권과 실패 처리 | 진입점이 저장소의 생성과 수명을 소유하고 Fastify `onClose` 훅에 해제를 연결합니다. 애플리케이션 내부 라우트는 구현체가 아니라 주입된 `AppRepository`만 사용합니다. 환경 파싱 또는 포트 열기가 실패하면 시작이 끝나지 않습니다. 아직 저장소 상태를 트래픽 수용 여부와 분리하는 준비 상태 경계는 없습니다. |
| 보장하는 것 | 로컬에서 실행 가능한 API 초기화와 한 번의 저장소 정리 경로가 생깁니다. |
| 보장하지 않는 것 | 시작 시 초기 데이터 생성, 운영 환경에서 메모리 저장소로 대체하는 동작, 마이그레이션 상태 검사는 아직 허용되거나 정의되지 않았습니다. |
| 후속 연결 | `85ac2a949439`가 설정 기본값을 고정하고, `e1a0316fbe84`가 여기의 암묵적 초기 데이터 생성 호출을 나중에 제거합니다. |
<!-- LEARNER-END:4b43a284e637:record -->




#### 비교 기준

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `85ac2a949439` — `test(api): 실행 환경 기본값 검증`

### 5.2. `test(api): 실행 환경 기본값 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `85ac2a949439` |
| 중요도 | B |
| 태그 | PROTOCOL, PERSISTENCE, TEST |
| 원문에서 확인한 역할 | 설정 우선순위와 로컬 시작 기본값을 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.test.ts`
- 핵심 심벌: `readEnv`의 기본값·환경값 우선순위·유효성 검사
- 빈 환경 객체와 명시적 환경 객체를 각각 `readEnv`에 넣어 어떤 필드가 기본값/강제 버전 지정이 되는지 확인합니다.
- 포트 같은 숫자 값과 모드/url 값의 실패 사례가 실제 파서에서 거부되는지 테스트 표와 연결합니다.
- 이 테스트는 프로세스를 시작하지 않고 순수 설정 해석만 검증한다는 한계를 구분합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:85ac2a949439:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 초기화가 환경값을 읽기 시작했지만 기본값이나 강제 버전 지정 규칙이 바뀌어도 이를 탐지할 자동화된 계약이 없었습니다. 로컬 시작과 배포 설정이 같은 파서를 쓰므로, 누락 값과 명시 값의 우선순위가 흔들리면 다른 저장소·출처·포트로 실행될 수 있었습니다. |
| 구현 또는 검증 결정 | `readEnv`에 통제된 환경 객체를 주입해 기본값과 명시 값이 예상 필드로 변환되는지 검증합니다. |
| 실행/검증 경로 | 테스트 실행 환경 구성 → `readEnv` 호출 → 반환된 설정의 각 필드 비교입니다. |
| 소유권과 실패 처리 | 테스트가 프로세스 전역 `process.env` 대신 입력 객체를 소유하므로 케이스 간 상태 오염을 피합니다. 잘못된 입력을 거부하는 파서 브랜치는 확인하지만 실제 OS 환경, 소켓 바인딩, 데이터베이스 연결 실패는 실행하지 않습니다. |
| 보장하는 것 | 설정 기본값과 강제 버전 지정 규칙의 회귀를 단위 수준에서 탐지합니다. |
| 보장하지 않는 것 | 설정이 실제 서비스 의존성을 사용할 수 있는지는 검증하지 않습니다. |
| 후속 연결 | 뒤의 `eb675ef74af3`와 `4633dfde208d`가 같은 설정 경계에 운영 전용 실패 시 차단 규칙을 추가합니다. |
<!-- LEARNER-END:85ac2a949439:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:85ac2a949439:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 설정 단위 테스트 |
| 주입·재현 방식 | 외부 프로세스나 네트워크 없이 명시적 환경 객체를 입력하고 `readEnv` 반환값과 예외 브랜치를 직접 비교합니다. |
| 검증하는 것 | 파서의 기본값·우선순위·입력 검증이 고정됩니다. |
| 검증하지 않는 것 | 실제 환경 변수 주입, 포트 바인딩, PostgreSQL 연결 성공까지는 검증하지 않습니다. |
<!-- LEARNER-END:85ac2a949439:test -->



#### 비교 기준

- 직전 관련 SHA: `4b43a284e637` — `feat(api): 실행 환경과 service bootstrap 구성`
- 다음 관련 SHA: `30aac132e14e` — `feat(db): migration set 상태 검사 추가`

### 5.3. `feat(db): migration set 상태 검사 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `30aac132e14e` |
| 중요도 | A |
| 태그 | PERSISTENCE, OPERATIONS, RISK |
| 원문에서 확인한 역할 | 번들/적용된 마이그레이션 집합을 현재, 대기 중, 불일치로 분류합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/migrator.ts`
- 핵심 심벌: `findMigrationFiles`, `compareMigrationSets`, `inspectMigrationSet`
- 번들 디렉터리의 SQL 파일명과 `kysely_migration.name` 조회 결과를 어떤 정규화 규칙으로 비교하는지 확인합니다.
- `current`, `pending`, `diverged`를 결정하는 누락된/예상 밖의 집합 계산을 실제 조건문으로 기록합니다.
- PostgreSQL 오류 코드 `42P01`만 마이그레이션 테이블 부재로 해석하고 다른 쿼리 오류는 다시 던지는 실패 시 차단 분기를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:30aac132e14e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 마이그레이션 실행기는 SQL을 적용할 수 있었지만, 이미 실행 중인 서비스가 번들과 데이터베이스의 마이그레이션 집합이 일치하는지 읽기 전용으로 판정할 수 없었습니다. |
| 해결하려던 문제와 위험 | 단순 연결 성공만으로는 코드가 기대하는 스키마인지 알 수 없습니다. 누락 마이그레이션과 데이터베이스에만 존재하는 예상 밖 마이그레이션을 구분하지 않으면 호환되지 않는 스키마에도 트래픽을 받을 수 있습니다. |
| 핵심 구현 결정 | 번들 SQL 이름과 적용된 Kysely 마이그레이션 이름을 집합으로 비교합니다. 누락만 있으면 `pending`, 예상 밖 이름이 하나라도 있으면 `diverged`, 두 집합이 같으면 `current`로 반환합니다. 마이그레이션 테이블이 아직 없는 `42P01`은 적용된 집합이 빈 상태로 처리합니다. |
| 입력 → 처리 → 결과 | 마이그레이션 파일 열거 → `select name from kysely_migration` → 누락된/예상 밖의 계산 → 상태와 차이 목록 반환입니다. |
| 소유 주체·수명·정리 | 함수는 스키마를 변경하지 않고 파일 목록과 쿼리 결과만 읽습니다. 쿼리 클라이언트/풀의 수명은 호출자가 계속 소유합니다. |
| 실패·되돌리기·재시도 | 마이그레이션 테이블 부재 이외의 SQL 오류를 정상 상태로 삼지 않고 전파합니다. 따라서 데이터베이스 장애가 마이그레이션 대기 중으로 위장되지 않습니다. |
| 보장하는 것 | 번들과 데이터베이스 마이그레이션 집합의 관계를 명시적으로 판정하고 divergence를 실패 시 차단 신호로 만들 수 있습니다. |
| 보장하지 않는 것 | 마이그레이션을 자동 실행하거나 SQL 내용의 의미적 호환성을 검증하지 않습니다. 이름 집합만 비교합니다. |
| 후속 연결 | `2f05d5d79c64`가 이 판정을 저장소 준비 상태 결과로 노출하고 `6937cf60aeea`가 대기 중/현재/불일치를 테스트합니다. |
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
| 중요도 | A |
| 태그 | PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | 데이터베이스 사용 가능 상태와 마이그레이션 상태를 저장소 계약으로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/index.ts`
- 핵심 심벌: `AppRepository.checkReadiness`, `PostgresRepository.checkReadiness`, 메모리 저장소 구현
- `AppRepository`의 준비 상태 반환형과 PostgreSQL/메모리 구현이 같은 계약을 어떻게 다르게 채우는지 확인합니다.
- PostgreSQL 구현에서 `select 1`과 `inspectMigrationSet` 호출 순서, 오류 전파, 마이그레이션 상태 매핑을 추적합니다.
- 메모리 구현의 `migrations: not_applicable`가 `current`를 가장하지 않는 이유를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:2f05d5d79c64:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 마이그레이션 상태 판정 함수는 존재했지만 API가 저장소 구현을 열어 풀/쿼리 세부사항을 알아야 상태 확인을 계산할 수 있었습니다. |
| 해결하려던 문제와 위험 | 준비 상태가 API의 구현 추측에 의존하면 메모리와 PostgreSQL이 다른 의미를 반환하거나 마이그레이션 검사 순서가 분산됩니다. |
| 핵심 구현 결정 | 공통 `AppRepository`에 `checkReadiness()`를 추가합니다. PostgreSQL 구현은 연결 쿼리와 마이그레이션 집합 검사를 수행하고, 메모리 구현은 데이터베이스 `up`, 마이그레이션 `not_applicable`을 반환합니다. |
| 입력 → 처리 → 결과 | API 또는 운영 호출자 → `repo.checkReadiness()` → 구현별 의존성 검사 → `{database, migrations}` 반환입니다. |
| 소유 주체·수명·정리 | 저장소가 자신의 연결과 스키마 상태를 판정하는 책임을 소유합니다. API는 결과만 소비하고 풀을 직접 만지지 않습니다. |
| 실패·되돌리기·재시도 | PostgreSQL 쿼리 또는 마이그레이션 검사가 실패하면 Promise가 거부됩니다. 이 SHA 자체는 오류를 HTTP 상태로 변환하지 않습니다. |
| 보장하는 것 | 저장소 종류에 관계없이 동일한 준비 상태 호출 지점이 생기고 마이그레이션 applicability를 명시적으로 표현합니다. |
| 보장하지 않는 것 | 서비스 수명주기나 트래픽 수용 여부는 아직 포함하지 않으며 데이터베이스 오류를 sanitize하는 HTTP 경계도 다음 커밋에 남습니다. |
| 후속 연결 | `15002e229acb`가 이 계약을 `/health/ready`에 연결합니다. |
<!-- LEARNER-END:2f05d5d79c64:record -->




#### 비교 기준

- 직전 관련 SHA: `30aac132e14e` — `feat(db): migration set 상태 검사 추가`
- 다음 관련 SHA: `15002e229acb` — `feat(ops): liveness와 readiness endpoint 추가`

### 5.5. `feat(ops): liveness와 readiness endpoint 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `15002e229acb` |
| 중요도 | A |
| 태그 | PROTOCOL, PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | 프로세스 생존 상태와 의존성 상태를 반영하는 서비스 준비 상태를 버전이 명시된 응답으로 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/shared/src/http.ts`, `apps/api/src/app.ts`
- 핵심 심벌: 생존 상태/준비 상태 응답 스키마, `/health/live`, `/health/ready`, `repo.checkReadiness`
- 공유 응답 스키마가 `status`, `service`, `checks.lifecycle/database/migrations`를 어떤 정해진 열거형 값으로 제한하는지 확인합니다.
- `/health/live`가 의존성 쿼리 없이 200을 반환하고 `/health/ready`만 저장소를 호출하는 분리를 추적합니다.
- 준비 상태 성공 조건과 오류 처리 브랜치의 503/중단/알 수 없는 응답, 원본 데이터베이스 오류를 본문에 넣지 않는 처리를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:15002e229acb:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 기존 `/health`는 프로세스가 응답한다는 사실만 보여 주었고 데이터베이스/마이그레이션 불일치에도 트래픽을 받아야 하는지 판단할 수 없었습니다. |
| 해결하려던 문제와 위험 | 실행 조정기가 프로세스 재시작과 트래픽 제거를 같은 신호로 처리하면 일시적 데이터베이스 장애에 프로세스가 재시작되거나, 반대로 스키마가 준비되지 않은 프로세스에 요청이 유입됩니다. |
| 핵심 구현 결정 | 생존 상태와 준비 상태를 별도 라우트로 만듭니다. 준비 상태는 데이터베이스가 `up`이고 마이그레이션이 `current` 또는 `not_applicable`일 때만 200/`ready`를 반환하며, 그 외 또는 예외는 503/`not_ready`로 변환합니다. |
| 입력 → 처리 → 결과 | HTTP 요청 → Fastify 라우트 → 실시간은 즉시 스키마 응답, 준비 완료는 `repo.checkReadiness` → 조건 판정 → 공유 스키마로 응답입니다. |
| 소유 주체·수명·정리 | 프로세스 생존 신호는 애플리케이션이, 저장소 판정은 저장소가 소유합니다. HTTP 경계가 내부 예외를 외부의 제한된 상태값으로 변환합니다. |
| 실패·되돌리기·재시도 | 저장소 거부는 `database: down`, `migrations: unknown`인 503으로 축약됩니다. 인증 정보나 연결 문자열은 응답에 포함되지 않습니다. |
| 보장하는 것 | 생존 상태와 의존성 상태를 반영하는 준비 상태가 서로 독립된 운영 신호가 됩니다. |
| 보장하지 않는 것 | 이 시점의 수명주기 값은 항상 `accepting`이며 종료 준비 상태는 `44ef3e07e1a5`에서 추가됩니다. |
| 후속 연결 | `6937cf60aeea`가 HTTP와 실제 PostgreSQL 상태 전환을 검증합니다. |
<!-- LEARNER-END:15002e229acb:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:15002e229acb:snippet -->
- SHA: `15002e229acb`
- 위치: `packages/shared/src/http.ts`; 생존 상태/준비 상태 응답 스키마, `/health/live`, `/health/ready`, `repo.checkReadiness`

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
| 중요도 | B |
| 태그 | AUTH, PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | DB 중단/마이그레이션 상태와 무관하게 생존 상태가 유지되고 준비 상태만 실패하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/health.test.ts`, `packages/db/src/postgres.integration.test.ts`, `packages/db/src/readiness.test.ts`
- 핵심 심벌: Fastify `inject`, 메모리 준비 상태 스텁, 격리된 PostgreSQL 데이터베이스, `compareMigrationSets`
- 기존 `/health`, `/health/live`, `/health/ready` 응답을 한 애플리케이션 인스턴스에서 비교하는 라우트 테스트를 확인합니다.
- `checkReadiness` 실패에 인증 정보가 든 오류를 주입하고 503 본문에 비밀값/URL이 없는지 확인합니다.
- 마이그레이션 미적용 격리된 데이터베이스가 `pending`, 마이그레이션 실행 후 `current`가 되는 실제 PostgreSQL 통합 경로를 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:6937cf60aeea:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 상태 확인 라우트와 저장소 판정이 구현됐지만 프로세스 신호 분리, 비밀값 비노출, 실제 마이그레이션 전후 상태를 함께 고정하는 증거가 없었습니다. 단위 스텁만으로는 실제 마이그레이션 테이블과 번들 비교가 맞는지, 통합 테스트만으로는 HTTP 오류 변환이 안전한지 각각 부족했습니다. |
| 구현 또는 검증 결정 | Fastify 주입으로 라우트 계약을, 메모리/스텁으로 실패 응답을, 격리된 PostgreSQL 스키마로 대기 중→현재 전환을 각각 검증합니다. |
| 실행/검증 경로 | 테스트 설정 → 애플리케이션/저장소 생성 → 상태 확인 요청 또는 마이그레이션 실행 → 상태/본문 비교 → 등록 역순 종료입니다. |
| 소유권과 실패 처리 | 테스트가 생성한 애플리케이션, 저장소, 격리된 데이터베이스를 종료 정리에서 회수합니다. 의도적으로 `checkReadiness`를 거부시켜 503 변환과 비밀값 비노출을 확인합니다. 실제 네트워크 단절을 주입하는 테스트는 아닙니다. |
| 보장하는 것 | 생존 상태/준비 상태 분리, 메모리 `not_applicable`, 마이그레이션 집합 판정, HTTP 오류 축약이 회귀 테스트로 고정됩니다. |
| 보장하지 않는 것 | 실행 조정기 확인 요청 설정이나 장시간 장애 복구는 검증하지 않습니다. |
| 후속 연결 | 후속 종료 준비 테스트가 준비 상태 수명주기에 `draining`을 추가하면서 이 계약을 확장합니다. |
<!-- LEARNER-END:6937cf60aeea:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:6937cf60aeea:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 라우트 단위/통합 + PostgreSQL 통합 |
| 주입·재현 방식 | Fastify `inject`, 저장소 실패 스텁, 마이그레이션 전후 격리된 PostgreSQL 스키마를 조합합니다. |
| 검증하는 것 | HTTP 계약과 실제 마이그레이션 상태가 같은 준비 상태 의미로 연결됩니다. |
| 검증하지 않는 것 | 실제 프로세스 상태 확인 요청, 컨테이너 재시작, 외부 네트워크 분할은 검증하지 않습니다. |
<!-- LEARNER-END:6937cf60aeea:test -->



#### 비교 기준

- 직전 관련 SHA: `15002e229acb` — `feat(ops): liveness와 readiness endpoint 추가`
- 다음 관련 SHA: `e1a0316fbe84` — `fix(api): startup seed 생성을 제거`

### 5.7. `fix(api): startup seed 생성을 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `e1a0316fbe84` |
| 중요도 | B |
| 태그 | PERSISTENCE |
| 원문에서 확인한 역할 | API 시작에서 암묵적 초기 데이터 생성 변경을 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/index.ts`
- 핵심 심벌: 진입점의 `ensureSeedData` 호출 제거
- 부모 커밋의 저장소 생성 뒤 `ensureSeedData` 호출과 이 SHA의 제거 변경 내용을 직접 비교합니다.
- 마이그레이션/시드 CLI가 남아 있는지와 API 시작이 이제 읽기/configure/포트 열기만 수행하는지 구분합니다.
- 메모리와 PostgreSQL 모두에서 같은 암묵적 변경이 사라지는 범위를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e1a0316fbe84:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 초기화는 저장소를 만든 직후 `ensureSeedData`를 호출해 프로세스 재시작이 데이터 생성 부수 효과를 가졌습니다. 서비스 시작과 데이터 준비가 결합되면 운영에서도 운영자가 승인하지 않은 시드 변경이 일어날 수 있고 준비 상태가 실제 사전 준비 상태를 가립니다. |
| 구현 또는 검증 결정 | API 진입점에서 `ensureSeedData` 호출을 삭제합니다. 데이터 준비는 별도 명령/운영 단계로 남기고 프로세스는 저장소를 선택해 애플리케이션을 시작하는 일만 합니다. |
| 실행/검증 경로 | 환경 해석 → 저장소 생성 → 애플리케이션 생성 → 포트 열기로 단순화됩니다. |
| 소유권과 실패 처리 | 시드 실행 책임이 API 프로세스 수명에서 외부의 명시적 관리 작업으로 이동합니다. 필요한 데이터가 없으면 시작이 자동 복구하지 않습니다. 이는 의도된 보장하지 않는 범위이며 운영자가 명시적으로 준비해야 합니다. |
| 보장하는 것 | API 시작이 시드 데이터를 암묵적으로 만들지 않습니다. |
| 보장하지 않는 것 | 소스 수준 호출 하나를 제거했을 뿐 다른 경로의 데이터 변경을 전역적으로 금지하는 것은 아닙니다. |
| 후속 연결 | `5cac4843fd9b`가 진입점 소스에서 호출 재도입을 막습니다. |
<!-- LEARNER-END:e1a0316fbe84:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:e1a0316fbe84:fix -->
시작이 개발 편의를 위해 시드를 생성한다는 초기 가정 → 운영에서 암묵적 데이터 변경 위험 → 진입점 호출 제거 → 소스 수준 회귀 보호 조건
<!-- LEARNER-END:e1a0316fbe84:fix -->


#### 비교 기준

- 직전 관련 SHA: `6937cf60aeea` — `test(ops): health와 database readiness 검증`
- 다음 관련 SHA: `5cac4843fd9b` — `test(api): startup seed 금지 검증`

### 5.8. `test(api): startup seed 금지 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `5cac4843fd9b` |
| 중요도 | B |
| 태그 | REALTIME, TEST |
| 원문에서 확인한 역할 | 진입점이 시드 연산을 호출하지 않는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/startup.test.ts` (같은 커밋의 `tests/smoke-ws.mjs` AI 픽스처 정렬은 이 개발 흐름의 부수 변경)
- 핵심 심벌: `index.ts` 소스 읽기, `/\.ensureSeedData\s*\(/` 실패 검증
- `startup.test.ts`가 빌드 산출물이 아니라 정확한 소스인 `index.ts`를 읽는 방식을 확인합니다.
- 정규식이 메서드 호출 형태의 `ensureSeedData` 재도입만 막는다는 범위와 우회 가능성을 기록합니다.
- 같은 커밋의 WebSocket 실행 확인 픽스처 변경은 시작 시 초기 데이터 생성 불변 조건과 분리해 다룹니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:5cac4843fd9b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 시드 호출은 제거됐지만 이후 리팩터링이 진입점에 다시 추가되어도 이를 직접 탐지하는 테스트가 없었습니다. 시작 변경 금지는 코드 검토 규칙만으로는 쉽게 회귀할 수 있습니다. |
| 구현 또는 검증 결정 | 진입점 소스를 문자열로 읽어 `.ensureSeedData(` 호출 패턴이 없음을 확인합니다. |
| 실행/검증 경로 | 테스트가 `index.ts` 경로 해석 → 소스 읽기 → 실패 정규식 검증을 수행합니다. |
| 소유권과 실패 처리 | 테스트는 파일 descriptor를 장기 보유하지 않는 동기 읽기만 수행합니다. 정규식이 일치하면 즉시 실패합니다. 간접 호출이나 다른 이름의 변경은 탐지하지 못합니다. |
| 보장하는 것 | 진입점에 직접 `ensureSeedData` 메서드 호출이 돌아오는 회귀를 고정적으로 막습니다. |
| 보장하지 않는 것 | 프로세스를 실행하거나 실제 데이터베이스가 변경되지 않는지 관찰하는 테스트는 아닙니다. |
| 후속 연결 | `e1a0316fbe84`의 수정을 좁은 소스 계약으로 보호합니다. |
<!-- LEARNER-END:5cac4843fd9b:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:5cac4843fd9b:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 정적 소스 회귀 테스트 |
| 주입·재현 방식 | Node `readFileSync`로 동일 디렉터리의 `index.ts`를 읽고 직접 메서드 호출 방식의 부재를 검사합니다. |
| 검증하는 것 | 정확히 그 호출 형태의 재도입을 탐지합니다. |
| 검증하지 않는 것 | 간접 시드 호출, 다른 시작 모듈, 실행 시점 데이터베이스 쓰기는 검증하지 않습니다. |
<!-- LEARNER-END:5cac4843fd9b:test -->



#### 비교 기준

- 직전 관련 SHA: `e1a0316fbe84` — `fix(api): startup seed 생성을 제거`
- 다음 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

### 5.9. `fix(config): production에서 영속 저장소 요구`

| 항목 | 값 |
| --- | --- |
| SHA | `eb675ef74af3` |
| 중요도 | A |
| 태그 | TOURNAMENT, OPERATIONS, RISK |
| 원문에서 확인한 역할 | 운영에서 데이터베이스 URL이 없으면 실행 환경 파싱 단계에서 실패합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.ts`
- 핵심 심벌: `readEnv`의 운영 모드 판정과 `DATABASE_URL` 요구 조건
- 명시적 `APP_MODE=production`과 `NODE_ENV=production`에서 모드가 어떻게 결정되는지 확인합니다.
- `DATABASE_URL` 부재를 메모리 저장소 선택까지 미루지 않고 파서에서 거부하는 정확한 조건을 기록합니다.
- 체험/로컬/테스트 모드의 메모리 저장소 대체 실행이 계속 허용되는 범위를 구분합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:eb675ef74af3:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기화는 `DATABASE_URL`이 없으면 모드와 관계없이 메모리 저장소를 선택할 수 있었습니다. |
| 해결하려던 문제와 위험 | 운영 프로세스가 잘못된 환경 설정으로 시작해 일시적 메모리 상태를 정상 서비스처럼 제공하면 경기·사용자·토너먼트 데이터가 재시작과 함께 사라집니다. |
| 핵심 구현 결정 | 설정 파서가 운영 모드를 확정한 뒤 데이터베이스 URL이 없으면 예외를 던지도록 변경합니다. 실패가 저장소 생성 함수보다 앞서므로 서버는 포트 열기하지 않습니다. |
| 입력 → 처리 → 결과 | 환경값 읽기 → 애플리케이션 모드 결정 → 운영이면 `DATABASE_URL` 존재 확인 → 실패 시 예외 발생, 성공 시 초기화 진행입니다. |
| 소유 주체·수명·정리 | 영속 저장소 선택 정책이 구성 진입점의 임시 조건이 아니라 설정 경계의 검증 규칙이 됩니다. |
| 실패·되돌리기·재시도 | 오설정 운영은 시작 즉시 실패입니다. 메모리로 조용히 대체 처리하지 않습니다. |
| 보장하는 것 | 운영 API는 영속 저장소 위치가 명시되지 않으면 시작하지 않습니다. |
| 보장하지 않는 것 | URL이 실제 연결 가능하거나 올바른 데이터베이스를 가리키는지는 준비 상태가 별도로 판정합니다. |
| 후속 연결 | `4633dfde208d`가 명시적/추론된 운영 두 경로를 모두 검증합니다. |
<!-- LEARNER-END:eb675ef74af3:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:eb675ef74af3:fix -->
URL 부재 시 모든 모드에서 메모리 저장소로 대체 가능 → 운영 데이터 손실 위험 → 설정 파서에서 즉시 실패 → 모드별 회귀 테스트
<!-- LEARNER-END:eb675ef74af3:fix -->


#### 비교 기준

- 직전 관련 SHA: `5cac4843fd9b` — `test(api): startup seed 금지 검증`
- 다음 관련 SHA: `4633dfde208d` — `test(config): production memory fallback 거부 검증`

### 5.10. `test(config): production memory fallback 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4633dfde208d` |
| 중요도 | A |
| 태그 | OPERATIONS, TEST |
| 원문에서 확인한 역할 | 명시적/추론된 운영 모두 메모리 저장소 대체 실행을 거부하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/env.test.ts`
- 핵심 심벌: `readEnv` 운영 사례, 체험/로컬 메모리 allowance
- `APP_MODE=production`과 `NODE_ENV=production` 각각에서 `DATABASE_URL` 누락이 같은 오류로 끝나는지 확인합니다.
- 운영에 URL을 제공한 성공 사례와 체험 모드의 메모리 허용 사례를 대조합니다.
- 테스트가 저장소 생성 전 파서 단계만 검증한다는 범위를 명시합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:4633dfde208d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 운영 즉시 실패 구현은 있었지만 한 모드 판정 경로만 테스트하면 다른 추론 경로가 메모리 저장소 대체 실행을 다시 허용할 수 있었습니다. |
| 해결하려던 문제와 위험 | 명시적 애플리케이션 모드와 배포의 `NODE_ENV` 추론이 서로 다른 브랜치를 타면 운영 불변 조건이 설정 방식에 따라 달라집니다. |
| 핵심 구현 결정 | 두 운영 판정 방식 모두 URL 부재에서 거부되는지, URL 제공 시 통과하는지, 체험은 계속 메모리를 허용하는지 표로 고정합니다. |
| 입력 → 처리 → 결과 | 환경 사례 구성 → `readEnv` 호출 → 예외 발생 또는 반환 설정 검증입니다. |
| 소유 주체·수명·정리 | 테스트가 각 환경 객체를 독립적으로 만들어 프로세스 전역 상태에 의존하지 않습니다. |
| 실패·되돌리기·재시도 | 오류는 설정 단계에서 발생하므로 저장소나 네트워크를 만들지 않습니다. |
| 보장하는 것 | 운영 환경에서 메모리 저장소로 대체하는 동작 거부가 모드 표기 방식과 무관하게 유지됩니다. |
| 보장하지 않는 것 | 실제 PostgreSQL 연결·마이그레이션 현재 여부는 준비 상태 테스트의 책임입니다. |
| 후속 연결 | 개발 흐름의 최종 시작 불변 조건을 완성합니다. |
<!-- LEARNER-END:4633dfde208d:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:4633dfde208d:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 실패 설정 경계 테스트 |
| 주입·재현 방식 | 명시적 운영, `NODE_ENV` 기반 운영, 체험 모드를 독립 입력으로 비교합니다. |
| 검증하는 것 | 모든 운영 해석 경로에서 영속 저장소 설정이 필수입니다. |
| 검증하지 않는 것 | 제공된 URL의 접근 가능성이나 데이터베이스 영속성 자체는 검증하지 않습니다. |
<!-- LEARNER-END:4633dfde208d:test -->



#### 비교 기준

- 직전 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:evolution -->
`4b43a284e637`은 프로세스가 실행될 구성 진입점과 저장소 수명을 만들었지만 시작 시 초기 데이터 생성과 메모리 저장소 대체 실행을 함께 허용했습니다. `30aac132e14e`와 `2f05d5d79c64`는 스키마 상태 판정과 저장소 소유의 준비 상태를 도입했고, `15002e229acb`는 이를 프로세스 생존 상태와 분리된 HTTP 신호로 내보냈습니다. `e1a0316fbe84`/`5cac4843fd9b`가 시작 변경을 제거·고정하고, `eb675ef74af3`/`4633dfde208d`가 운영의 영속 저장소 즉시 실패를 완성합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:failure-links -->
- 암묵적 초기 데이터 생성 변경 → `e1a0316fbe84` 제거 → `5cac4843fd9b` 소스 회귀 보호 조건
- 운영 환경에서 메모리 저장소로 대체하는 동작 → `eb675ef74af3` 파서 즉시 실패 → `4633dfde208d` 명시적/추론된 운영 테스트
- 데이터베이스/마이그레이션 불확실성 → `30aac132e14e`/`2f05d5d79c64` 판정 → `6937cf60aeea` 라우트·PostgreSQL 근거
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:ownership -->
진입점은 환경·저장소 생성과 종료 훅만 소유합니다. 저장소는 연결 및 마이그레이션 상태 판정을 소유하고, Fastify 상태 확인 라우트는 그 결과를 제한된 외부 상태로 변환합니다. 시드/마이그레이션 실행은 API 프로세스 밖의 명시적 운영 작업으로 이동합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:final-state -->
프로세스가 응답하면 생존 상태는 유지되지만, 데이터베이스 중단·대기 중/불일치 마이그레이션이면 준비 상태는 503입니다. 운영은 데이터베이스 URL이 없으면 포트 열기 이전에 실패하고, 시작은 시드를 만들지 않습니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:final-flow -->
- 환경 파싱에서 모드와 영속 저장소 요구를 검증합니다.
- 진입점이 저장소를 만들고 Fastify 애플리케이션에 주입합니다.
- `/health/live`는 의존성과 무관하게 프로세스 생존을 응답합니다.
- `/health/ready`는 저장소의 연결 및 마이그레이션 집합 결과를 읽어 200/503을 결정합니다.
- 종료 시 Fastify `onClose`가 저장소 `close`를 호출합니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:01-startup-liveness-readiness-and-storage-state.md:checks -->
- [x] 마이그레이션 `current/pending/diverged/not_applicable`의 의미와 계산 근거를 설명할 수 있습니다.
- [x] 생존 상태와 준비 상태의 호출 경로·실패 브랜치를 정확한 SHA 기준으로 구분할 수 있습니다.
- [x] 시작 시 초기 데이터 생성 제거와 운영 환경에서 메모리 저장소로 대체하는 동작 거부의 수정→테스트 관계를 설명할 수 있습니다.
<!-- LEARNER-END:01-startup-liveness-readiness-and-storage-state.md:checks -->
===== END FILE: 01-startup-liveness-readiness-and-storage-state.md =====

===== BEGIN FILE: 02-metrics-observer-boundaries-and-cardinality.md =====
# 지표 관측 지점과 라벨 조합 수

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

실행 시점, HTTP, 저장소, 경기방/재연결, 결과 확정, 스냅샷 전달, 이벤트 루프 지연을 Prometheus에 노출하되 도메인 코드와 값 종류가 지나치게 많은 신원을 지표 라벨에 결합하지 않는 구조를 복원합니다.

범위 메모: 민감 로그 비밀값 제거(`4c7e884bc9b0`)은 관측 태그를 갖지만 주된 불변 조건이 인증 정보 비노출이므로 신원/보안 카테고리에 남기고, 여기서는 지표 라벨 조합 수와 관측 코드 소유권만 다룹니다.

### 직접 관련된 불변 조건

- 관측 코드는 도메인 상태 기계 내부 규칙을 소유하거나 변경하지 않습니다.
- 지표 라벨은 정해진 이벤트 종류만 사용해 라벨 조합 수가 사용자/경기방 수에 비례하지 않습니다.
- 전달/결과 확정/준비 상태 지표는 해당 결과가 실제 결정되는 경계에서 기록됩니다.
- 수집기와 이벤트 루프 히스토그램의 수명은 Fastify 애플리케이션 수명과 함께 종료됩니다.

## 2. 핵심 질문

- 지표 레지스트리와 수집기 수명주기는 애플리케이션 시작/종료에서 누가 소유합니까?
- HTTP 정규화하지 않은 URL 대신 라우트 템플릿을 라벨로 사용하는 이유는 무엇입니까?
- 저장소 프록시가 메서드 반환 타입, `this`, 예외 발생/실패 동작 의미를 보존합니까?
- 경기방/사용자/요청 ID를 지표 라벨로 쓰지 않으면서 correlation은 로그/관측기에서 어떻게 유지합니까?
- 스냅샷 폐기와 전송 콜백 지연을 측정하는 위치가 실제 동작 의미 소유 주체와 일치합니까?
- 이벤트 루프 p95가 부하 테스트 도구에 전달될 때 누락된 예시를 어떻게 실패 시 차단 처리합니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `6bf29a5acf35` | `build(api): metrics 수집 의존성 추가` | C | - | `prom-client`를 API의 명시적 실행 의존성으로 추가합니다. |
| 2 | `69278d8fc456` | `feat(metrics): runtime gauge registry 추가` | B | REALTIME, OPERATIONS, OBSERVABILITY | Node 수집기와 실시간 GameHub 게이지를 위한 전용 레지스트리를 만듭니다. |
| 3 | `02b3b3a32f14` | `feat(metrics): HTTP와 readiness 측정 추가` | B | OPERATIONS, OBSERVABILITY, PERF | 정규화된 라우트/메서드/상태로 요청 실행 시간과 준비 상태를 측정합니다. |
| 4 | `843d355afc69` | `feat(metrics): repository operation 측정 추가` | B | PERSISTENCE, OBSERVABILITY | 원본 동작을 보존하는 프록시로 동기·비동기 저장소 메서드의 실행 시간과 결과를 측정합니다. |
| 5 | `e08367a1be5e` | `feat(metrics): game room과 reconnect 관측 추가` | B | AUTH, PROTOCOL, REALTIME | GameHub 수명주기 주변 관측기로 경기방/재연결 이벤트를 관측합니다. |
| 6 | `e850b3356b9b` | `feat(metrics): match finalization 결과 관측 추가` | B | REALTIME, PERSISTENCE, OBSERVABILITY | 비회원 메모리 결과와 데이터베이스에 저장하는 결과 확정 성공/실패를 구분해 측정합니다. |
| 7 | `c0d184bcc928` | `feat(metrics): snapshot delivery와 drop 관측 추가` | B | REALTIME, OBSERVABILITY, PERF | 최신 버퍼가 실제 전달/폐기를 결정하는 지점에서 측정합니다. |
| 8 | `685d85c863a4` | `test(metrics): database와 snapshot 지표 검증` | B | AUTH, REALTIME, PERSISTENCE | 사용자/경기방 ID를 라벨로 만들지 않고 DB/실시간 동작을 관측하는지 검증합니다. |
| 9 | `1baf4c5a57ba` | `feat(metrics): event-loop lag 측정 추가` | B | OBSERVABILITY | Node 이벤트 루프 지연 히스토그램의 p95를 게이지로 노출합니다. |
| 10 | `66b8f07c2387` | `test(load): event-loop lag를 부하 profile에 노출` | B | OPERATIONS, OBSERVABILITY, PERF | 부하용 추가 Compose 설정에서 지표 엔드포인트를 루프백에 노출하고 k6 종료 정리가 서버 p95를 수집합니다. |
| 11 | `697a63ebb8c8` | `test(load): event-loop lag 임계값 검증` | B | OPERATIONS, OBSERVABILITY, PERF | 이벤트 루프 지연 p95 50ms 임계값, 필수 지표, 루프백 지표 노출을 계약 테스트로 고정합니다. |

## 5. 커밋별 학습 기록

### 5.1. `build(api): metrics 수집 의존성 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `6bf29a5acf35` |
| 중요도 | C |
| 태그 | - |
| 원문에서 확인한 역할 | `prom-client`를 API의 명시적 실행 의존성으로 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/package.json`, `pnpm-lock.yaml`
- 핵심 심벌: API 의존성 `prom-client@^15.1.3`와 잠금 파일 해석
- API 패키지의 의존성 구역과 잠금 파일의 importer가 같은 버전을 가리키는지 확인합니다.
- 이 SHA에는 수집기나 라우트가 없고 다음 커밋을 가능하게 하는 기계적 준비라는 범위를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:6bf29a5acf35:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 맥락 | API 패키지에는 Prometheus 레지스트리·지표 타입을 제공하는 실행 의존성이 없었습니다. |
| 구체적 역할 | API 패키지에 `prom-client`를 추가하고 잠금 파일을 갱신합니다. |
| 보장 범위/제한 | API가 `prom-client`를 직접 가져올 수 있습니다. 지표 생성·노출·정리 동작은 아직 없습니다. |
| 후속 연결 | `69278d8fc456`가 이 의존성으로 전용 레지스트리를 만듭니다. |
<!-- LEARNER-END:6bf29a5acf35:record -->




#### 비교 기준

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `69278d8fc456` — `feat(metrics): runtime gauge registry 추가`

### 5.2. `feat(metrics): runtime gauge registry 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `69278d8fc456` |
| 중요도 | B |
| 태그 | REALTIME, OPERATIONS, OBSERVABILITY |
| 원문에서 확인한 역할 | Node 수집기와 실시간 GameHub 게이지를 위한 전용 레지스트리를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`
- 핵심 심벌: `ApiMetrics`, 비공개 `Registry`, `collectDefaultMetrics`, `scrape`, `close`
- `ApiMetrics`가 전역 레지스트리가 아니라 비공개 `Registry`를 만들고 모든 게이지를 그 레지스트리에 등록하는지 확인합니다.
- `scrape()`가 `readGameStats` 콜백을 호출한 시점의 온라인/대기 중인/경기방 값을 게이지에 집합한 뒤 직렬화하는 순서를 추적합니다.
- `close()`가 레지스트리를 해제하고 애플리케이션 수명과 수집기 수명을 맞추는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:69278d8fc456:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | GameHub의 실시간 상태와 Node 실행 상태를 수집할 수 있는 형태로 보유하는 객체가 없었습니다. 전역 singleton 지표를 쓰면 테스트/애플리케이션 인스턴스 사이 수집기 중복과 수명 누수가 생기며, 게임 상태를 지표 객체가 직접 소유하면 도메인 상태와 관측 상태가 결합됩니다. |
| 구현 또는 검증 결정 | `ApiMetrics`가 전용 `Registry`를 소유하고 Node 기본값 지표, 연결·대기열·경기방 게이지를 등록합니다. GameHub 상태는 생성자에 주입된 `readGameStats` 콜백으로 지표 수집 시점에 읽습니다. |
| 실행/검증 경로 | 지표 수집 요청 → 콜백으로 실시간 통계 읽기 → 게이지 집합 → 비공개 레지스트리 serialize입니다. |
| 소유권과 실패 처리 | 애플리케이션 인스턴스마다 `ApiMetrics`와 레지스트리가 하나씩 존재합니다. GameHub는 상태를 소유하고 지표는 읽기 콜백만 보유하며 `close()`가 레지스트리를 정리합니다. 콜백이나 레지스트리 직렬화 오류는 `scrape()` 거부로 남습니다. 지표는 도메인 상태를 변경하지 않습니다. |
| 보장하는 것 | Node와 실시간 GameHub 상태를 애플리케이션 로컬 레지스트리에서 수집할 기반이 생깁니다. |
| 보장하지 않는 것 | HTTP 엔드포인트와 요청/저장소/실시간 결과 측정은 아직 연결되지 않습니다. |
| 후속 연결 | `02b3b3a32f14`가 애플리케이션 수명주기와 `/metrics` 라우트에 연결합니다. |
<!-- LEARNER-END:69278d8fc456:record -->




#### 비교 기준

- 직전 관련 SHA: `6bf29a5acf35` — `build(api): metrics 수집 의존성 추가`
- 다음 관련 SHA: `02b3b3a32f14` — `feat(metrics): HTTP와 readiness 측정 추가`

### 5.3. `feat(metrics): HTTP와 readiness 측정 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `02b3b3a32f14` |
| 중요도 | B |
| 태그 | OPERATIONS, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 정규화된 라우트/메서드/상태로 요청 실행 시간과 준비 상태를 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`, `apps/api/src/app.ts`
- 핵심 심벌: HTTP 실행 시간 히스토그램, 준비 상태 히스토그램과 카운터, `/metrics`, `onResponse` 훅
- 요청 라벨이 정규화하지 않은 URL이 아니라 `request.routeOptions.url` 또는 미리 정한 대체 라벨을 쓰는지 확인합니다.
- 실행 시간을 고해상도 경과 시간에서 초 단위로 바꾸고 음수를 범위 제한하는 도우미 함수와 상태/메서드 라벨을 확인합니다.
- `buildApp`이 지표를 만들고 `onResponse`, 준비 상태 라우트, `/metrics`, `onClose`에 연결하는 수명을 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:02b3b3a32f14:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 레지스트리는 있었지만 지표 수집 라우트가 없고 HTTP·준비 상태 결과가 운영 지표에 남지 않았습니다. 정규화하지 않은 URL이나 요청 ID를 라벨로 쓰면 요청 수에 비례해 시계열이 증가합니다. 또한 상태 확인 라우트의 결과와 요청 실행 시간을 실제 응답 경계에서 기록해야 합니다. |
| 구현 또는 검증 결정 | Fastify `onResponse`에서 정규화된 라우트 템플릿, 메서드, 상태 코드로 요청 실행 시간을 기록합니다. 준비 상태 실행 시간·결과를 별도로 기록하고 `/metrics`가 레지스트리 콘텐츠 타입과 본문을 반환하도록 연결합니다. |
| 실행/검증 경로 | 요청 시작 시각 보관 → 라우트 실행 → `onResponse`에서 경과 시간을 초 단위로 변환한 값 기록; `/health/ready`는 저장소 결과를 응답으로 바꾸며 준비 상태 지표도 기록; `/metrics`는 지표 수집합니다. |
| 소유권과 실패 처리 | Fastify 애플리케이션이 `ApiMetrics`를 생성·종료하고 훅/라우트가 같은 인스턴스를 공유합니다. unmatched 라우트는 미리 정한 대체 라벨을 사용하며 음수 경과 시간은 0으로 범위 제한됩니다. 지표 수집 실패를 숨기지는 않습니다. |
| 보장하는 것 | HTTP와 준비 상태의 지연 시간/결과가 정해진 값만 사용하는 라벨로 노출됩니다. |
| 보장하지 않는 것 | 저장소 메서드와 GameHub 내부 결과는 아직 HTTP 지표로만 간접 관찰됩니다. |
| 후속 연결 | `843d355afc69`부터 결과가 결정되는 내부 경계에 관측기를 추가합니다. |
<!-- LEARNER-END:02b3b3a32f14:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:02b3b3a32f14:snippet -->
- SHA: `02b3b3a32f14`
- 위치: `apps/api/src/observability.ts`; HTTP 실행 시간 히스토그램, 준비 상태 히스토그램과 카운터, `/metrics`, `onResponse` 훅

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
| 중요도 | B |
| 태그 | PERSISTENCE, OBSERVABILITY |
| 원문에서 확인한 역할 | 원본 동작을 보존하는 프록시로 동기·비동기 저장소 메서드의 실행 시간과 결과를 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`, `apps/api/src/app.ts`
- 핵심 심벌: `instrumentRepository`, `Proxy.get`, 고정된 `REPOSITORY_OPERATIONS` 목록, 성공/실패 관측기
- 프록시가 알려진 저장소 메서드만 정해진 연산 라벨로 쓰고 나머지는 `other`로 축약하는지 확인합니다.
- 메서드 호출 시 receiver/`this`를 원본 저장소로 보존하는 `Reflect`/`apply` 경로를 확인합니다.
- 동기 예외 발생과 Promise 판별/거부가 각각 한 번만 실행 시간/결과로 기록되고 원래 반환·예외 동작 의미를 보존하는지 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:843d355afc69:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | HTTP 라우트 실행 시간은 알 수 있었지만 데이터베이스/저장소 작업별 지연 시간과 실패를 실제 저장소 호출 경계에서 구분할 수 없었습니다. 호출자마다 측정 코드를 넣으면 누락과 중복이 생기고, 래퍼가 동기/비동기 반환이나 `this` 결합을 바꾸면 운영 동작이 달라집니다. |
| 구현 또는 검증 결정 | 저장소를 `Proxy`로 감싸 함수 호출을 측정하되 원본 receiver와 반환값을 보존합니다. 동기 예외 발생은 오류 처리에서, Promise는 판별/거부 처리 함수에서 결과를 기록합니다. |
| 실행/검증 경로 | 호출자 → 프록시 속성 조회 → 원본 메서드 호출 → 동기 결과/예외 또는 Promise 완료 → 지표 기록 → 원래 결과 전달입니다. |
| 소유권과 실패 처리 | 소스 저장소가 연결과 데이터 상태를 계속 소유합니다. 프록시는 측정 래퍼일 뿐 `close`를 포함한 모든 메서드를 원본에 위임합니다. 동기 예외와 비동기 실패 모두 `failure`로 기록한 뒤 그대로 다시 전달합니다. 연산 이름이 허용 목록 밖이면 `other`입니다. |
| 보장하는 것 | 저장소 동작을 바꾸지 않으면서 연산별 성공/실패 실행 시간을 관찰합니다. |
| 보장하지 않는 것 | 트랜잭션 내부 SQL 문별 지연 시간이나 쿼리 라벨 조합 수까지는 보여 주지 않습니다. |
| 후속 연결 | `685d85c863a4`가 실제 데이터베이스 지표와 라벨 비노출을 검증합니다. |
<!-- LEARNER-END:843d355afc69:record -->




#### 비교 기준

- 직전 관련 SHA: `02b3b3a32f14` — `feat(metrics): HTTP와 readiness 측정 추가`
- 다음 관련 SHA: `e08367a1be5e` — `feat(metrics): game room과 reconnect 관측 추가`

### 5.5. `feat(metrics): game room과 reconnect 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e08367a1be5e` |
| 중요도 | B |
| 태그 | AUTH, PROTOCOL, REALTIME |
| 원문에서 확인한 역할 | GameHub 수명주기 주변 관측기로 경기방/재연결 이벤트를 관측합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`
- 핵심 심벌: `GameHubObserver.roomCreated`, `GameHubObserver.reconnect`, 클라이언트 `requestId`, 애플리케이션 로깅 콜백
- GameHub가 구체적인 로거/지표 클래스가 아니라 선택적 관측기 콜백만 받는 생성자 경계를 확인합니다.
- 경기방 생성 시 경기방/사용자/요청 ID는 구조화된 로그 컨텍스트로 전달되지만 지표 라벨에는 들어가지 않는 분리를 확인합니다.
- 재연결 `success|expired` 결과가 결정된 정확한 상태 전이 뒤 관측기가 호출되는지 추적합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e08367a1be5e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | GameHub 경기방/재연결 수명주기는 동작했지만 어떤 경기방이 생성되고 복구가 성공/만료됐는지 외부에서 관찰할 훅이 없었습니다. 도메인 상태 기계 내부에 로거와 지표 레지스트리를 직접 넣으면 수명주기 규칙과 관측 구현이 결합되고, 신원을 지표 라벨로 쓰면 라벨 조합 수가 무한히 증가합니다. |
| 구현 또는 검증 결정 | GameHub는 선택적 `GameHubObserver`를 받고 경기방 생성과 재연결 결과가 확정된 지점에서 정해진 결과 값과 상관관계 확인용 컨텍스트를 전달합니다. 애플리케이션은 결과만 지표에 넣고 ID는 구조화된 로그 컨텍스트로 남깁니다. |
| 실행/검증 경로 | HTTP/WebSocket 인증에서 요청/사용자 컨텍스트 확보 → `hub.connect` → 경기방/재연결 상태 전이 → 관측기 콜백 → 애플리케이션 로거/지표 호출입니다. |
| 소유권과 실패 처리 | GameHub가 경기방 상태와 콜백 호출 시점을 소유하고 애플리케이션이 로깅/지표 구현을 소유합니다. 관측기는 상태를 변경할 권한이 없습니다. 관측기는 선택적이라 미설정 시 도메인 동작은 계속됩니다. 이 SHA에서 콜백 자체가 예외 발생할 때 격리를 별도로 제공하는지는 확인되지 않습니다. |
| 보장하는 것 | 경기방/재연결 관측이 상태 기계 밖의 어댑터에 연결되고 지표 라벨에는 정해진 결과 값만 사용합니다. |
| 보장하지 않는 것 | 구조화된 로그의 비밀값 제거 정책은 이 개발 흐름 밖의 로깅/인증 커밋이 소유합니다. |
| 후속 연결 | `e850b3356b9b`가 같은 측정 방식을 경기 결과 확정에 확장합니다. |
<!-- LEARNER-END:e08367a1be5e:record -->




#### 비교 기준

- 직전 관련 SHA: `843d355afc69` — `feat(metrics): repository operation 측정 추가`
- 다음 관련 SHA: `e850b3356b9b` — `feat(metrics): match finalization 결과 관측 추가`

### 5.6. `feat(metrics): match finalization 결과 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `e850b3356b9b` |
| 중요도 | B |
| 태그 | REALTIME, PERSISTENCE, OBSERVABILITY |
| 원문에서 확인한 역할 | 비회원 메모리 결과와 데이터베이스에 저장하는 결과 확정 성공/실패를 구분해 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`, `apps/api/src/observability.ts`
- 핵심 심벌: `GameHubObserver.matchFinalized`, 결과 확정 성공·실패 분기, 영속 저장/결과 라벨
- 메모리 비회원 경기 결과와 데이터베이스 기반 `repo.finalizeMatch` 경로에서 관측기 컨텍스트가 어떻게 달라지는지 확인합니다.
- 데이터베이스 결과 확정 Promise의 성공과 오류 처리 브랜치가 각각 `success|failure` 결과를 한 번 기록하는지 추적합니다.
- 영속 저장 라벨이 `memory|database`, 결과가 정해진 이벤트 종류 중 하나이며 경기/경기방 ID가 지표 라벨에 없는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:e850b3356b9b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 저장소 프록시로 메서드 실패는 볼 수 있었지만 하나의 경기 결과 확정이라는 도메인 결과가 메모리/데이터베이스에서 어떻게 끝났는지 직접 나타내지 못했습니다. 저장소 연산 지표만으로는 재시도·중복·비회원 완료와 같은 도메인 의미를 복원하기 어렵습니다. |
| 구현 또는 검증 결정 | GameHub 결과 확정 경계에 관측기를 추가해 영속 저장 종류와 성공/실패를 기록합니다. 메모리 결과는 저장소 호출 없이 성공으로, 데이터베이스 결과는 `finalizeMatch` 완료 결과에 따라 기록합니다. |
| 실행/검증 경로 | 경기방 종료 상태 → 메모리 결과 생성 또는 저장소 결과 확정 → 결과 확정 → 관측기 → 지표/로그 어댑터입니다. |
| 소유권과 실패 처리 | GameHub가 결과 확정 수명주기와 관찰 시점을 소유하고 저장소가 영속 쓰기를 소유합니다. 지표는 결과를 복제해 기록할 뿐 성공 여부를 결정하지 않습니다. 데이터베이스 거부는 실패 지표를 남긴 뒤 기존 재시도/정리 경로로 전달됩니다. 관측 기록이 영속 저장 결과를 성공으로 바꾸지 않습니다. |
| 보장하는 것 | 경기 결과 확정 여부와 영속 저장 방식을 제한된 라벨 값의 지표로 구분합니다. |
| 보장하지 않는 것 | 중복 결과 확정 counter는 `ad482c200cea`의 후속 주기/결과 확정 보정에서 추가됩니다. |
| 후속 연결 | `547d9943d30a`가 부하 테스트 도구의 기준 데이터 소스를 클라이언트 이벤트에서 이 서버 지표로 옮깁니다. |
<!-- LEARNER-END:e850b3356b9b:record -->




#### 비교 기준

- 직전 관련 SHA: `e08367a1be5e` — `feat(metrics): game room과 reconnect 관측 추가`
- 다음 관련 SHA: `c0d184bcc928` — `feat(metrics): snapshot delivery와 drop 관측 추가`

### 5.7. `feat(metrics): snapshot delivery와 drop 관측 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `c0d184bcc928` |
| 중요도 | B |
| 태그 | REALTIME, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 최신 버퍼가 실제 전달/폐기를 결정하는 지점에서 측정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`, `apps/api/src/observability.ts`, GameHub 연결
- 핵심 심벌: `PendingSnapshot.enqueuedAt`, `delivered` 관측기와 폐기 사유 `replaced|connection_closed|congestion`
- 스냅샷을 추가할 때 단조 증가 타임스탬프를 저장하고 실제 전송 콜백/대기열 처리에서 지연을 계산하는지 확인합니다.
- 최신 값만 유지하는 교체, 연결 종료, 혼잡 프로세스 종료 각각이 제한된 폐기 사유로 기록되는 브랜치를 추적합니다.
- 관측기가 버퍼의 전달 동작 의미 소유 주체 안에 있고 GameHub가 추측해서 폐기를 세지 않는 이유를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:c0d184bcc928:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 스냅샷 버퍼는 오래된 값을 새 값으로 교체하거나 혼잡 시 폐기할 수 있었지만, 외부에서는 의도적인 최신값 유지와 전송 실패를 구분할 수 없었습니다. 호출자가 전송 횟수만 세면 교체된 스냅샷, 연결 종료로 폐기된 스냅샷, 실제 전달 완료까지의 지연을 정확히 알 수 없습니다. |
| 구현 또는 검증 결정 | 대기 중 스냅샷에 추가 시각을 넣고 버퍼가 전달 완료·폐기를 결정하는 브랜치에서 관측기를 호출합니다. 폐기 사유는 세 가지 정해진 값 중 하나로 제한합니다. |
| 실행/검증 경로 | GameHub 스냅샷 → 버퍼 추가/교체 → 소켓 상태·버퍼에 쌓인 바이트 수 판단 → 전송 또는 폐기 → 지연/사유 관측기입니다. |
| 소유권과 실패 처리 | 각 클라이언트의 `LatestSnapshotBuffer`가 대기 중 메시지 본문과 측정 시점을 소유합니다. 지표 어댑터는 숫자와 정해진 폐기 사유만 받습니다. 연결 종료와 혼잡은 폐기로 기록되며 소스 메시지 본문이나 경기방 ID는 지표 라벨에 포함되지 않습니다. |
| 보장하는 것 | 전달 지연과 실제 폐기 결정이 동일한 소유 주체에서 관찰됩니다. |
| 보장하지 않는 것 | 이 시점의 `sending` 상태 값이 혼잡으로 해석되는 가정은 `d90f17fa765d`에서 수정됩니다. |
| 후속 연결 | `685d85c863a4`가 측정과 라벨 조합 수를 검증하고 `d90f17fa765d`가 콜백 지연 오판을 고칩니다. |
<!-- LEARNER-END:c0d184bcc928:record -->




#### 비교 기준

- 직전 관련 SHA: `e850b3356b9b` — `feat(metrics): match finalization 결과 관측 추가`
- 다음 관련 SHA: `685d85c863a4` — `test(metrics): database와 snapshot 지표 검증`

### 5.8. `test(metrics): database와 snapshot 지표 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `685d85c863a4` |
| 중요도 | B |
| 태그 | AUTH, REALTIME, PERSISTENCE |
| 원문에서 확인한 역할 | 사용자/경기방 ID를 라벨로 만들지 않고 DB/실시간 동작을 관측하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: 관측 라우트와 테스트 파일, `LatestSnapshotBuffer`의 테스트용 소켓 테스트
- 핵심 심벌: Fastify `/metrics` 지표 수집, 가상 타이머/소켓, 라벨 부재 검증
- 저장소 성공/실패와 스냅샷 교체/전달을 만들고 지표 수집 본문의 지표 이름·라벨을 확인하는 테스트 설정을 추적합니다.
- 50ms 전달과 교체/폐기를 가상 타이머·테스트용 소켓으로 결정적으로 재현하는지 확인합니다.
- `requestId`, `userId`, `roomId`, `matchId` 문자열이 Prometheus 출력에 없다는 실패 검증을 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:685d85c863a4:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 지표 구현은 있었지만 결과가 실제 지표 수집에 나타나는지, 신원이 라벨로 새어 시계열을 폭증시키지 않는지 보호되지 않았습니다. 지표 이름 존재만 확인하면 관측기가 잘못된 브랜치에서 호출되거나 값 종류가 지나치게 많은 라벨이 추가돼도 놓칠 수 있습니다. |
| 구현 또는 검증 결정 | 저장소와 스냅샷 버퍼의 실제 코드 경로를 통과시킨 뒤 `/metrics` 출력을 검사합니다. 테스트용 소켓과 타이머로 교체·전달을 재현하고 ID 라벨 부재를 확인합니다. |
| 실행/검증 경로 | 테스트 동작 → 관측기 지표 갱신 → 애플리케이션 지표 수집 → 본문 노출 형식 검증입니다. |
| 소유권과 실패 처리 | 테스트가 애플리케이션, 저장소, 테스트용 소켓, 가상 타이머를 생성하고 종료 정리에서 정리합니다. 저장소 실패와 스냅샷 폐기를 의도적으로 만들지만 실제 PostgreSQL/네트워크 혼잡은 사용하지 않습니다. |
| 보장하는 것 | 중요 지표가 수집되고 정해진 라벨 값만 사용하는 규칙이 회귀로 고정됩니다. |
| 보장하지 않는 것 | Prometheus 서버 수집, 보존 기간, 경보 기준, 실제 부하 분포는 검증하지 않습니다. |
| 후속 연결 | 후속 `66b8f07c2387`/`697a63ebb8c8`가 실제 부하 프로필에서 이벤트 루프 지표를 읽고 임계값을 고정합니다. |
<!-- LEARNER-END:685d85c863a4:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:685d85c863a4:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 지표 통합 테스트 |
| 주입·재현 방식 | Fastify 지표 수집과 가상 타이머/소켓을 결합해 운영 관측기 경로를 실행합니다. |
| 검증하는 것 | 지표 기록, 정해진 값만 사용하는 라벨, 스냅샷 전달/폐기 측정 위치를 검증합니다. |
| 검증하지 않는 것 | 실제 수집기 백엔드나 장시간 라벨 조합 수 증가량은 측정하지 않습니다. |
<!-- LEARNER-END:685d85c863a4:test -->



#### 비교 기준

- 직전 관련 SHA: `c0d184bcc928` — `feat(metrics): snapshot delivery와 drop 관측 추가`
- 다음 관련 SHA: `1baf4c5a57ba` — `feat(metrics): event-loop lag 측정 추가`

### 5.9. `feat(metrics): event-loop lag 측정 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1baf4c5a57ba` |
| 중요도 | B |
| 태그 | OBSERVABILITY |
| 원문에서 확인한 역할 | Node 이벤트 루프 지연 히스토그램의 p95를 게이지로 노출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/observability.ts`
- 핵심 심벌: `monitorEventLoopDelay({resolution: 20})`, p95 게이지 수집 콜백, `enable`, `disable`
- Node 히스토그램을 생성자에서 활성화하고 `close()`에서 비활성화하는 수명을 확인합니다.
- 나노초 단위 백분위 값 값을 초 단위로 변환하는 계산과 p95 게이지 수집 시점을 확인합니다.
- 기본값 지표의 이벤트 루프 지표와 별도로 서비스 수준 p95 게이지를 만든 이유와 초기화 여부를 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1baf4c5a57ba:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | Node 기본 지표는 있었지만 부하 임계값이 직접 사용할 p95 이벤트 루프 지연 표본은 없었습니다. 런타임 스케줄링 부하를 클라이언트 스냅샷 지연만으로 추정하면 네트워크와 브라우저 영향을 분리할 수 없습니다. |
| 구현 또는 검증 결정 | `monitorEventLoopDelay` 히스토그램을 20ms 해상도로 활성화하고 수집 시 95번째 백분위수를 초 단위 게이지로 설정합니다. |
| 실행/검증 경로 | 애플리케이션 지표 생성 → 히스토그램 활성화 → 이벤트 루프 지연 표본 누적 → 지표 수집 시 p95를 10억으로 나누어 초 단위로 변환 → 애플리케이션 종료 시 비활성화합니다. |
| 소유권과 실패 처리 | `ApiMetrics`가 히스토그램 핸들의 활성화·비활성화 수명을 소유합니다. 표본이 비정상일 때의 기본값 처리는 지표 구현에 따르며, 이 커밋은 경보나 트래픽 수용 여부를 바꾸지 않습니다. |
| 보장하는 것 | 서버 이벤트 루프 p95를 라벨이 없는 단일 게이지로 제공합니다. |
| 보장하지 않는 것 | 이 지표만으로 원인이나 각 요청 지연을 식별하지 못합니다. |
| 후속 연결 | `66b8f07c2387`가 부하용 추가 Compose 설정과 k6 종료 정리에서 이 게이지를 읽습니다. |
<!-- LEARNER-END:1baf4c5a57ba:record -->



#### 최소 코드 근거

<!-- LEARNER-BEGIN:1baf4c5a57ba:snippet -->
- SHA: `1baf4c5a57ba`
- 위치: `apps/api/src/observability.ts`; `monitorEventLoopDelay({resolution: 20})`, p95 게이지 수집 콜백, `enable`, `disable`

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
| 중요도 | B |
| 태그 | OPERATIONS, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 부하용 추가 Compose 설정에서 지표 엔드포인트를 루프백에 노출하고 k6 종료 정리가 서버 p95를 수집합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.load.yml`, `tests/load/pong-load.js`, 부하 프로필 설정
- 핵심 심벌: 루프백 전용 API 지표 포트, k6 `teardown`, `event_loop_lag_p95_ms` 추세 지표
- 부하용 추가 Compose 설정이 API 지표 포트를 `127.0.0.1`에만 공개하는지 확인합니다.
- k6 종료 정리가 `/metrics`를 GET하고 `pong_pong_api_event_loop_lag_p95_seconds` 예시를 파싱해 milliseconds 추세 지표에 넣는지 추적합니다.
- 임계값 `p(95)<=50`과 지표 수집 실패/누락된 지표에서 `fail`하는 브랜치를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:66b8f07c2387:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 이벤트 루프 게이지는 API 내부에 있었지만 부하 테스트 도구가 이를 읽지 않아 클라이언트 표시되는 SLI와 서버 스케줄러 부하를 같은 run에서 비교할 수 없었습니다. 지표 엔드포인트가 외부에 무제한 노출되거나 k6가 지표 부재를 0으로 처리하면 부하 결과가 잘못 통과할 수 있습니다. |
| 구현 또는 검증 결정 | 부하용 추가 Compose 설정에서 지표 포트를 루프백으로 제한하고, k6 종료 정리가 지표 수집 결과를 읽어 p95 초 단위를 밀리초 추세 지표로 기록합니다. 누락되거나 비정상인 표본은 실행 실패로 처리합니다. |
| 실행/검증 경로 | 부하 시나리오 실행 → 종료 정리 HTTP 지표 수집 → Prometheus 텍스트 행 파싱 → k6 사용자 정의 추세 지표 추가 → 임계값 평가입니다. |
| 소유권과 실패 처리 | API가 지표를 소유하고 부하 테스트 도구는 실행 종료 시 읽기만 합니다. 포트 공개 범위는 추가 Compose 설정이 소유합니다. 지표 수집 상태/본문 또는 예시가 유효하지 않으면 `fail`합니다. 0으로 대체해 false 통과하지 않습니다. |
| 보장하는 것 | 부하 실행 결과에 서버 이벤트 루프 p95가 포함되며 지표는 루프백에서만 접근할 수 있습니다. |
| 보장하지 않는 것 | 이 커밋의 단위 테스트가 실제 부하 실행을 수행하지는 않습니다. |
| 후속 연결 | `697a63ebb8c8`가 임계값과 추가 Compose 설정/본문 계약을 정적으로 검증합니다. |
<!-- LEARNER-END:66b8f07c2387:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:66b8f07c2387:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 부하 테스트 실행 틀 운영 통합 |
| 주입·재현 방식 | k6 종료 정리가 실제 HTTP 지표 수집을 수행하도록 구현되며 임계값은 k6 옵션에 포함됩니다. |
| 검증하는 것 | 실제 부하 실행에서는 지표가 없으면 실패하도록 경로가 구성되어 있음을 검증합니다. |
| 검증하지 않는 것 | 이 워크북 환경에서는 k6 run이 실행되지 않았으므로 수치 결과는 제공하지 않습니다. |
<!-- LEARNER-END:66b8f07c2387:test -->



#### 비교 기준

- 직전 관련 SHA: `1baf4c5a57ba` — `feat(metrics): event-loop lag 측정 추가`
- 다음 관련 SHA: `697a63ebb8c8` — `test(load): event-loop lag 임계값 검증`

### 5.11. `test(load): event-loop lag 임계값 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `697a63ebb8c8` |
| 중요도 | B |
| 태그 | OPERATIONS, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 이벤트 루프 지연 p95 50ms 임계값, 필수 지표, 루프백 지표 노출을 계약 테스트로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-harness.test.mjs`, 부하 프로필/추가 Compose 설정 소스
- 핵심 심벌: 소스/compose 검증, `event_loop_lag_p95_ms` 임계값, 필수 지표 이름
- 부하 프로필 옵션이 이벤트 루프 추세 지표에 `p(95)<=50`을 정확히 요구하는지 확인합니다.
- 테스트 실행 틀 소스가 예상 Prometheus 지표를 참조하고 종료 정리에서 읽는다는 정적 검증을 확인합니다.
- Compose가 공개하는 주소가 루프백인지, 외부 공개 포트와 섞이지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:697a63ebb8c8:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 부하 테스트 도구가 임계값을 갖게 됐지만 설정·지표 이름·포트 결합이 수정되면 실제 부하 전까지 회귀를 발견하기 어려웠습니다. 운영 측정의 연결은 문자열/설정 변경으로 쉽게 무력화되며 실행 비용이 큰 k6만으로 매 커밋 검증하기 어렵습니다. |
| 구현 또는 검증 결정 | Node 계약 테스트가 부하 프로필, k6 소스, Compose 추가 Compose 설정을 읽어 50ms 임계값과 필수 지표, 루프백 결합을 고정합니다. |
| 실행/검증 경로 | 소스/설정 읽기 → 방식 및 객체 검증 → 실패 시 테스트 오류입니다. |
| 소유권과 실패 처리 | 정적 계약 테스트는 실행 시점 서비스를 시작하지 않고 파일 내용만 소유합니다. 임계값 제거, 지표 rename, 루프백이 아닌 포트 공개를 탐지합니다. |
| 보장하는 것 | 이벤트 루프 부하 계약의 핵심 연결이 빠른 정적 테스트로 보호됩니다. |
| 보장하지 않는 것 | 실제로 이벤트 루프 지연이 50ms 이하인지와 Prometheus 파서가 지원하는 모든 노출 형식은 검증하지 않습니다. |
| 후속 연결 | 개발 흐름의 측정 체인을 레지스트리 → 지표 수집 → 부하 컬렉션 → 정적 계약으로 닫습니다. |
<!-- LEARNER-END:697a63ebb8c8:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:697a63ebb8c8:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 정적 운영 계약 테스트 |
| 주입·재현 방식 | 부하 프로필 객체와 소스/Compose 본문을 읽어 임계값·지표·결합을 검사합니다. |
| 검증하는 것 | 측정 연결과 임계값 설정의 회귀를 탐지합니다. |
| 검증하지 않는 것 | 부하 중 이벤트 루프 p95가 실제 기준을 만족한다는 실행 증거는 아닙니다. |
<!-- LEARNER-END:697a63ebb8c8:test -->



#### 비교 기준

- 직전 관련 SHA: `66b8f07c2387` — `test(load): event-loop lag를 부하 profile에 노출`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:evolution -->
`6bf29a5acf35`와 `69278d8fc456`은 애플리케이션별 레지스트리와 수집기 수명을 만들고, `02b3b3a32f14`는 값 종류가 제한된 HTTP·준비 상태 라벨로 외부 지표 수집 경로를 엽니다. `843d355afc69`부터 저장소, GameHub 수명주기, 결과 확정, 스냅샷 버퍼처럼 결과를 실제로 결정하는 위치에 관측기를 둡니다. `685d85c863a4`는 신원 라벨 부재를 고정하고, `1baf4c5a57ba`부터 이벤트 루프 p95를 부하 실행까지 전달해 50ms 계약으로 보호합니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:failure-links -->
- 정규화하지 않은 URL/신원 라벨 조합 수 위험 → 정규화된 라우트와 값의 종류를 제한한 결과와 사유 → `685d85c863a4` 실패 라벨 검증
- 저장소 동기 예외와 비동기 거부를 관측하지 못할 위험 → 원본 동작을 보존하는 프록시에서 완료 결과 기록 → 데이터베이스 지표 테스트
- 이벤트 루프 지표 누락을 0으로 오판할 위험 → 종료 단계에서 지표 누락을 실패로 처리하는 파서 → `697a63ebb8c8` 정적 계약
- 콜백 지연을 폐기로 오판하는 측정 의미 문제는 개발 흐름 04의 `d90f17fa765d`/`5cd54767858f`에서 수정됩니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:ownership -->
Fastify 애플리케이션이 `ApiMetrics`와 레지스트리와 히스토그램 수명을 소유합니다. 저장소와 GameHub는 운영 상태를 계속 소유하고 프록시/관측기는 결과를 읽어 전달합니다. `LatestSnapshotBuffer`는 전달/폐기 의미를 소유하므로 해당 지표도 버퍼 내부에서 발생합니다. 부하 테스트 도구는 지표 수집 소비 측일 뿐 서버 지표를 재정의하지 않습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:final-state -->
HTTP, 준비 상태, 저장소, 경기방/재연결, 결과 확정, 스냅샷 전달/폐기, 이벤트 루프 p95가 정해진 값만 사용하는 라벨로 지표 수집됩니다. 사용자/요청/경기방/경기 ID는 correlation용 구조화된 로그 컨텍스트에만 남고 지표 시계열을 생성하지 않습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:final-flow -->
- 애플리케이션 생성 시 비공개 레지스트리와 기본값/이벤트 루프 수집기를 시작합니다.
- HTTP·저장소·GameHub·스냅샷 버퍼가 각 결과 확정 지점에서 정해진 값만 받는 관측기를 호출합니다.
- `/metrics`가 애플리케이션 로컬 레지스트리를 Prometheus 본문으로 직렬화합니다.
- 부하 테스트 도구 종료 정리가 루프백 지표 수집에서 이벤트 루프 p95와 서버 결과 확정 개수를 읽습니다.
- 애플리케이션 종료가 레지스트리를 해제하고 이벤트 루프 히스토그램을 비활성화합니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:02-metrics-observer-boundaries-and-cardinality.md:checks -->
- [x] 각 지표 값이 확정되는 지점와 라벨 이벤트 종류를 파일·함수 기준으로 설명할 수 있습니다.
- [x] 저장소 프록시의 동기/비동기 동작 의미 보존과 실패 기록 순서를 설명할 수 있습니다.
- [x] 이벤트 루프 지표의 생성·지표 수집·부하 임계값·정적 계약 연결을 구분할 수 있습니다.
<!-- LEARNER-END:02-metrics-observer-boundaries-and-cardinality.md:checks -->
===== END FILE: 02-metrics-observer-boundaries-and-cardinality.md =====

===== BEGIN FILE: 03-runtime-limiter-primitives-and-bounded-work.md =====
# 실행 제한 기본 요소와 작업량 상한

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

고정 간격 단계 시간 보정, 연결 확인 신호, 입력 순서/호출 빈도 제한, 최신 값만 유지하는 스냅샷 버퍼를 GameHub와 독립된 기본 요소로 복원하고 각 기본 요소가 소유하는 시간·상태·정리 상한을 확인합니다.

범위 메모: 이 개발 흐름은 재사용 가능한 호출 제한기의 탄생과 결정적 경계까지만 다룹니다. GameHub 수명주기 통합, 공유 스케줄러 소유권, 실제 혼잡 수정은 다음 개발 흐름으로 분리합니다.

### 직접 관련된 불변 조건

- 한 타이머 콜백이 수행하는 시뮬레이션 누적 시간 보정 작업은 고정된 상한을 넘지 않습니다.
- 응답 없는 연결은 단일 연결 확인 신호 소유 주체가 정해진 응답 기한 뒤 제거합니다.
- 허용된 입력 순번은 사용자·경기방별별로 증가하고 허용된 빈도는 사용자별 토큰 허용 시간을 넘지 않습니다.
- 스냅샷 대기열은 클라이언트별 최신 하나 값으로 제한되고 전송 계층 혼잡은 최대 대기 시간이 정해진 연결 종료로 수렴합니다.

## 2. 핵심 질문

- 경과 시간 벽시계와 고정된 시뮬레이션 시간 간격은 어떤 누적 시간·범위 제한 산술로 분리됩니까?
- 연결 확인 신호 주기와 시간 초과 핸들은 누가 만들고 응답/중지 때 어떻게 교체·해제합니까?
- 오래된 입력과 빈도 제한된 입력 중 어느 검사가 먼저이며 그 순서가 토큰 허용 시간에 어떤 영향을 줍니까?
- 최신 스냅샷 교체, 완화된 재시도, 강제 프로세스 종료의 조건과 최초 콜백 지연 가정은 무엇입니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3a2943ff385d` | `feat(game): fixed-step scheduler 추가` | A | SIMULATION, REALTIME, OBSERVABILITY | 단조 증가 경과 시간을 한 번에 최대 5개의 50ms 시뮬레이션 단계로 변환하는 고정 간격 단계 누적 시간을 도입합니다. |
| 2 | `0888e119036d` | `test(game): fixed-step 보정 범위 검증` | B | SIMULATION, REALTIME, TEST | 경과 시간 단조 증가 시간을 고정된 50 ms 작업으로 바꾸는 경계와 누적 시간 보정 상한을 결정적 시계로 검증합니다. |
| 3 | `10a656e59864` | `feat(game): WebSocket heartbeat 추가` | A | REALTIME, RISK | WebSocket 연결의 ping 주기, 응답 기한, 종료 정리를 단일 연결 확인 신호 수명주기로 만듭니다. |
| 4 | `81031dcd2c1c` | `test(game): heartbeat timeout 검증` | B | REALTIME, TEST | ping 주기, 45초 시간 초과, 응답 기한 초기화를 가상 타이머로 검증합니다. |
| 5 | `207df3f47935` | `feat(game): 입력 순서와 rate limit 보호` | A | SIMULATION, REALTIME, RISK | 경기방별 단조 증가 입력 순번과 사용자별 토큰 버킷 허용 시간을 하나의 입력 수락 검사로 결합합니다. |
| 6 | `1353e3eb99cc` | `test(game): input gate 제한 검증` | B | REALTIME, TEST | 단조 증가 순서와 사용자별 토큰 버킷의 경계·격리를 결정적 시계로 검증합니다. |
| 7 | `8589ff3c4821` | `feat(game): latest snapshot buffer 추가` | A | SIMULATION, REALTIME, OPERATIONS | 느린 전송 계층에서는 스냅샷 대기열을 쌓지 않고 최신 값만 보존하며, 혼잡이 계속되면 제한 시간 안에 연결을 종료합니다. |
| 8 | `125aa113a01c` | `test(game): snapshot replacement와 congestion 검증` | A | REALTIME, PERF, RISK | 최신 값만 유지하는 교체, 완화된 재시도, 강제 프로세스 종료, 혼잡 시간 초과를 제어되는 소켓 상태와 가상 시간으로 검증합니다. |

## 5. 커밋별 학습 기록

### 5.1. `feat(game): fixed-step scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `3a2943ff385d` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, OBSERVABILITY |
| 원문에서 확인한 역할 | 단조 증가 경과 시간을 한 번에 최대 5개의 50ms 시뮬레이션 단계로 변환하는 고정 간격 단계 누적 시간을 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 심벌: `FixedStepScheduler`, `DEFAULT_TIMESTEP_MS`, `MAX_STEPS_PER_TICK`
- `start`, `stop`, 내부 타이머 콜백이 `now()`와 `lastTimeMs`, `accumulatedMs`를 갱신하는 순서를 확인합니다.
- 경과 시간이 음수일 때 0으로 취급하고, 긴 지연 뒤 누적량을 `timestep * maxSteps`로 제한하는 산술을 확인합니다.
- 한 콜백에서 실행할 단계 수를 최대 5회로 제한한 뒤 남은 누적량을 어떻게 처리하는지 확인합니다.
- 주입 가능한 단조 증가 시계와 타이머 API가 운영 시간과 결정적 테스트를 어떻게 분리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:3a2943ff385d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub의 각 경기방은 벽시계 주기 콜백이 호출될 때마다 곧바로 한 번 시뮬레이션을 진행했습니다. 콜백이 늦거나 몰릴 때 실제 경과 시간과 시뮬레이션 단계 수의 관계를 설명할 별도 소유자가 없었습니다. |
| 해결하려던 문제와 위험 | 이벤트 루프 지연을 그대로 따라잡으려 하면 한 콜백에서 무제한 작업이 발생하고, 반대로 항상 한 번만 진행하면 시뮬레이션 시간이 실제 경과와 크게 어긋납니다. 또한 시스템 시계 역행이 음수 경과 시간을 만들 수 있었습니다. |
| 핵심 구현 결정 | `FixedStepScheduler`가 단조 증가 `now()` 차이를 누적 시간에 더하고 50 ms 단위로 단계 콜백을 실행합니다. 누적 지연은 최대 250 ms, 한 콜백의 단계는 최대 5회로 제한하며 음수 경과 시간은 0으로 보정합니다. 타이머와 시계를 주입할 수 있어 계산 규칙을 전송 계층과 분리합니다. |
| 입력 → 처리 → 결과 | `start` → 현재 단조 증가 시각 저장 → 주기 콜백 → `max(0, now-last)` 계산 → 누적량을 250 ms 이하로 범위 제한 → 누적량이 50 ms 이상인 동안 최대 5회 `step(50)` → 사용한 시간을 누적 시간에서 차감합니다. |
| 소유 주체·수명·정리 | 스케줄러 인스턴스가 주기 핸들, 마지막 관측 시각, 누적 경과 시간을 단독 소유합니다. `start`와 `stop`은 중복 호출에 안전하며 `stop`이 핸들과 누적 진행 상태를 정리합니다. |
| 실패·되돌리기·재시도 | 시계가 뒤로 가면 경과 시간을 0으로 만들고, 이벤트 루프가 장시간 멈춰도 과거의 모든 틱을 재생하지 않습니다. 단계 콜백 자체의 예외를 복구하는 장치는 이 기본 요소에 없으므로 호출자가 실행 실패 의미를 소유합니다. |
| 보장하는 것 | 시뮬레이션 작업은 고정된 50 ms 단위이며 한 타이머 콜백당 최대 5회로 제한됩니다. 벽시계 지연이 즉시 무제한 CPU 폭주로 변환되지 않습니다. |
| 보장하지 않는 것 | 이 SHA는 스케줄러 기본 요소만 추가합니다. GameHub 경기방이 아직 이를 사용하지 않으므로 실제 실시간 경로의 타이머 구성은 바뀌지 않습니다. |
| 후속 연결 | `0888e119036d`가 보정 산술을 고정하고, `a6a1f4fba60e`에서 각 경기방의 시뮬레이션 소유 주체로 처음 통합됩니다. |
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

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `0888e119036d` — `test(game): fixed-step 보정 범위 검증`

### 5.2. `test(game): fixed-step 보정 범위 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `0888e119036d` |
| 중요도 | B |
| 태그 | SIMULATION, REALTIME, TEST |
| 원문에서 확인한 역할 | 경과 시간 단조 증가 시간을 고정된 50 ms 작업으로 바꾸는 경계와 누적 시간 보정 상한을 결정적 시계로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/fixedStepScheduler.test.ts`
- 핵심 심벌: 제어 가능한 가상 시계/타이머를 사용하는 `FixedStepScheduler` 테스트 사례
- 49 ms와 50 ms에서 단계 호출 횟수가 달라지는 경계 테스트를 확인합니다.
- 10초 지연을 주입했을 때 5회만 실행되고 오래된 lag를 다음 콜백으로 무한 이월하지 않는지 확인합니다.
- 시계 역행과 `stop` 뒤 콜백 중단을 어떤 가상 타이머 상태로 검증하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:0888e119036d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 고정 간격 단계 계산은 구현됐지만 50ms 경계, 긴 일시 정지, 시계 역행, 중지 정리를 재현 가능한 방식으로 고정한 근거가 없었습니다. 실제 시간을 기다리는 테스트는 느리고 비결정적이며, 누적 시간의 경계값 하나 차이나 오래된 지연 값 이월을 놓치기 쉽습니다. |
| 구현 또는 검증 결정 | 가짜 단조 증가 시각과 타이머를 주입해 49/50 ms 경계, 10초 일시정지, 역행한 시각, 중지 이후 상태를 직접 구동합니다. |
| 실행/검증 경로 | 스케줄러 시작 → 제어 가능한 가상 시계 변경 → 타이머 콜백 실행 → 기록된 단계 실행 시간·호출 횟수 확인 → 중지한 뒤 추가 콜백이 작업을 만들지 않는지 확인합니다. |
| 소유권과 실패 처리 | 테스트가 시계와 타이머 진행을 소유하고 운영 스케줄러는 주입된 API만 사용합니다. 각 사례는 스케줄러를 중지해 핸들을 남기지 않습니다. 10초 경과를 한 번에 주입해 누적 시간 보정이 5회로 제한되는지, 다음 정상 틱이 과거 lag를 반복하지 않는지 확인합니다. |
| 보장하는 것 | 고정 간격 단계의 산술 경계와 정리가 벽시계 시간 제어에 의존하지 않고 회귀 테스트로 고정됩니다. |
| 보장하지 않는 것 | 실제 Node 이벤트 루프 지연이나 여러 경기방의 작업 비용은 측정하지 않습니다. |
| 후속 연결 | `3a2943ff385d`의 기본 요소를 검증하며, 실제 GameHub 통합은 `a6a1f4fba60e`에서 별도로 검증해야 합니다. |
<!-- LEARNER-END:0888e119036d:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:0888e119036d:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 경계 테스트 |
| 주입·재현 방식 | 주입된 제어 가능한 가상 시계와 타이머 콜백으로 49/50 ms, 10초 지연, backward 시간, 중지 상태를 직접 재현합니다. |
| 검증하는 것 | 한 콜백당 최대 5 단계, 음수 경과 시간 무시, 중지 이후 0 진행 상태를 검증합니다. |
| 검증하지 않는 것 | OS 스케줄러·실제 이벤트 루프·시뮬레이션 본문의 처리량은 검증하지 않습니다. |
<!-- LEARNER-END:0888e119036d:test -->



#### 비교 기준

- 직전 관련 SHA: `3a2943ff385d` — `feat(game): fixed-step scheduler 추가`
- 다음 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`

### 5.3. `feat(game): WebSocket heartbeat 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `10a656e59864` |
| 중요도 | A |
| 태그 | REALTIME, RISK |
| 원문에서 확인한 역할 | WebSocket 연결의 ping 주기, 응답 기한, 종료 정리를 단일 연결 확인 신호 수명주기로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/connectionHeartbeat.ts`
- 핵심 심벌: `ConnectionHeartbeat.start`, `acknowledge`, `stop`, ping/시간 초과 핸들
- 15초 ping 주기와 마지막 응답 기준 45초 시간 초과가 어떤 타이머들로 구성되는지 확인합니다.
- `acknowledge`가 시간 초과 소유권을 갱신하는 순서와 `start`/`stop`의 멱등성을 확인합니다.
- `socket.ping()`이 예외 발생할 때 즉시 강제 종료하는 실패 분기와 모든 핸들 정리를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:10a656e59864:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 소켓 종료 이벤트만으로 연결 상실을 알 수 있었고, 반개방 연결이나 응답하지 않는 상대를 제한된 시간 안에 제거하는 소유 주체가 없었습니다. |
| 해결하려던 문제와 위험 | 전송 계층이 열림 상태로 남아도 상대가 사라질 수 있습니다. ping과 만료 타이머가 여러 위치에 흩어지면 재연결·교체·종료 때 타이머가 남거나 중복 종료할 위험이 있습니다. |
| 핵심 구현 결정 | `ConnectionHeartbeat`가 15초마다 ping을 보내고 45초 동안 응답이 없으면 소켓을 강제 종료합니다. 응답은 시간 초과 기한을 다시 예약하고, 시작/중지가 모든 타이머 핸들을 한 객체에서 관리합니다. |
| 입력 → 처리 → 결과 | `start` → ping 주기·초기 시간 초과 예약 → ping 콜백에서 `socket.ping()` → pong/활동 시 `acknowledge`가 시간 초과 재설정 → 기한 도달 또는 ping 예외 시 강제 종료 → `stop`에서 주기/시간 초과 제거입니다. |
| 소유 주체·수명·정리 | 연결마다 하나의 연결 확인 신호 인스턴스가 ping 주기와 시간 초과를 소유합니다. 전송 계층 소유 주체는 연결 생성 시 시작하고 연결 해제/교체/종료 시 중지해야 합니다. |
| 실패·되돌리기·재시도 | ping 호출이 동기 예외를 던지면 더 이상 연결 확인 신호를 반복하지 않고 소켓을 강제 종료합니다. 응답 없음도 45초 상한 뒤 같은 종료 경로로 수렴합니다. |
| 보장하는 것 | 응답 없는 연결은 정해진 응답 기한 안에 제거되고, 타이머 정리를 단일 객체가 멱등적으로 수행할 수 있습니다. |
| 보장하지 않는 것 | 기본 요소는 누가 pong을 응답으로 전달하고 언제 중지할지 알지 못합니다. 실제 GameHub 클라이언트 수명주기 연결은 아직 없습니다. |
| 후속 연결 | `81031dcd2c1c`가 시간 계약을 고정하고, `fc2a4451eed1`이 GameHub 클라이언트 생성·종료에 연결합니다. |
<!-- LEARNER-END:10a656e59864:record -->




#### 비교 기준

- 직전 관련 SHA: `0888e119036d` — `test(game): fixed-step 보정 범위 검증`
- 다음 관련 SHA: `81031dcd2c1c` — `test(game): heartbeat timeout 검증`

### 5.4. `test(game): heartbeat timeout 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `81031dcd2c1c` |
| 중요도 | B |
| 태그 | REALTIME, TEST |
| 원문에서 확인한 역할 | ping 주기, 45초 시간 초과, 응답 기한 초기화를 가상 타이머로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/connectionHeartbeat.test.ts`
- 핵심 심벌: 테스트용 소켓의 `ping`/`terminate`, 가상 타이머
- 30초 진행 뒤 ping 두 번, 45초 경계에서 강제 종료되는 기대 조건을 확인합니다.
- 중간 `acknowledge`가 기존 시간 초과를 취소하고 새 45초 기한을 만드는지 확인합니다.
- 중지 또는 ping 예외 발생 사례가 남은 타이머를 어떻게 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:81031dcd2c1c:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 연결 확인 신호 구현의 15초/45초 관계와 기한 초기화가 실제 시간을 기다리지 않고 검증되지 않았습니다. 타이머 기반 생존 상태는 경계 시점과 중복 핸들 오류가 흔하며 실제 시간 테스트는 flaky합니다. |
| 구현 또는 검증 결정 | Vitest 가상 타이머와 호출 기록 소켓으로 ping 횟수, 강제 종료 시각, 응답 후 기한 이동을 결정적으로 검사합니다. |
| 실행/검증 경로 | 연결 확인 신호 시작 → 가상 시간 30초 진행 → ping 호출 확인 → 기한 직전/도달 상태 비교 → acknowl예외 조건에서 기한 재계산을 확인합니다. |
| 소유권과 실패 처리 | 테스트가 가상 타이머 대기열을 소유하고 종료 시 타이머를 비웁니다. 연결 확인 신호의 `stop`이 운영 핸들 정리를 수행하는지 호출 수로 확인합니다. 응답 없음과 ping 예외를 각각 강제 종료로 수렴시키는 경계를 재현합니다. |
| 보장하는 것 | 연결 확인 신호의 시간 상수와 초기화/정리 규칙이 결정적 회귀로 고정됩니다. |
| 보장하지 않는 것 | 실제 WebSocket pong 이벤트 연결이나 GameHub 재연결 동작 의미는 검증하지 않습니다. |
| 후속 연결 | `10a656e59864`의 기본 요소 증거이며, `fc2a4451eed1` 이후 통합 경로 테스트가 필요합니다. |
<!-- LEARNER-END:81031dcd2c1c:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:81031dcd2c1c:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 타이머 테스트 |
| 주입·재현 방식 | Vitest 가상 타이머와 테스트용 소켓 호출 감시 객체를 사용합니다. |
| 검증하는 것 | ping 15초 주기, 45초 시간 초과, 응답 기한 초기화를 검증합니다. |
| 검증하지 않는 것 | 실제 네트워크 반개방 탐지 시간이나 OS TCP 동작은 검증하지 않습니다. |
<!-- LEARNER-END:81031dcd2c1c:test -->



#### 비교 기준

- 직전 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`
- 다음 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`

### 5.5. `feat(game): 입력 순서와 rate limit 보호`

| 항목 | 값 |
| --- | --- |
| SHA | `207df3f47935` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, RISK |
| 원문에서 확인한 역할 | 경기방별 단조 증가 입력 순번과 사용자별 토큰 버킷 허용 시간을 하나의 입력 수락 검사로 결합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/inputGate.ts`
- 핵심 심벌: `InputGate.accept`, `releaseUser`, 토큰 버킷 상태와 순번 키
- 순번 키가 사용자와 경기방을 NUL delimiter로 결합해 경기방별 단조 증가 순서를 유지하는지 확인합니다.
- 오래된/중복 순번을 토큰 차감 전에 거부하는 순서를 확인합니다.
- 초당 30 토큰, 폭주 8의 초당 보충량 산술과 같은 사용자가 여러 경기방에서 허용 시간을 공유하는지 확인합니다.
- `releaseUser`가 사용자 버킷과 관련 순번 참가 기록을 모두 제거하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:207df3f47935:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub는 도착한 패들 입력을 전송 계층 순서대로 처리했으며 중복/오래된 입력과 한 사용자의 과도한 명령을 독립적으로 제한하지 않았습니다. |
| 해결하려던 문제와 위험 | 재전송·역순 패킷은 상태를 뒤로 움직일 수 있고, 무제한 입력은 이벤트 루프와 시뮬레이션 작업을 압박합니다. 오래된 입력이 빈도 허용 시간까지 소비하면 정상 최신 입력을 부당하게 막을 수 있습니다. |
| 핵심 구현 결정 | `InputGate`가 `(user, room)`별 마지막 `inputSeq`를 먼저 검사하고, 증가한 입력에만 사용자별 토큰 버킷을 적용합니다. 버킷은 초당 30개까지 초당 보충량되고 최대 8개 폭주를 허용합니다. |
| 입력 → 처리 → 결과 | 입력 도착 → 사용자/경기방 마지막 순번 조회 → `inputSeq <= last`면 오래된 거부 → 현재 시각으로 사용자 버킷 초당 보충량 → 토큰 없으면 빈도 거부 → 토큰 1 차감·마지막 순번 갱신 → 수락입니다. |
| 소유 주체·수명·정리 | InputGate가 사용자 버킷과 사용자/경기방 순번 목록을 소유합니다. 호출자는 사용자의 마지막 현재 유효한 연결이 제거될 때 `releaseUser`를 호출해야 메모리와 허용 시간 상태가 해제됩니다. |
| 실패·되돌리기·재시도 | 오래된 입력은 토큰을 소비하지 않고, 빈도 제한을 넘은 입력은 순번을 증가시키지 않습니다. 따라서 초당 허용량이 보충된 뒤 더 큰 순번을 다시 보낼 수 있습니다. 안전하지 않거나 단조 증가하지 않는 시계는 주입한 시계와 0 이상으로 보정한 경과 시간 처리로 제한합니다. |
| 보장하는 것 | 같은 사용자·경기방별의 허용된 입력은 strictly increasing이며, 한 사용자의 허용된 빈도는 폭주 8과 초당 30 초당 보충량으로 제한됩니다. |
| 보장하지 않는 것 | 이 기본 요소 자체는 메시지 본문 스키마, 경기방 참가 상태, 플레이어 측 권한 검사를 검증하지 않습니다. GameHub가 그 선행 조건을 확인해야 합니다. |
| 후속 연결 | `1353e3eb99cc`가 순서와 허용 시간 상호작용을 검증하고, `fc2a4451eed1`에서 실제 메시지 경로에 통합됩니다. |
<!-- LEARNER-END:207df3f47935:record -->




#### 비교 기준

- 직전 관련 SHA: `81031dcd2c1c` — `test(game): heartbeat timeout 검증`
- 다음 관련 SHA: `1353e3eb99cc` — `test(game): input gate 제한 검증`

### 5.6. `test(game): input gate 제한 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1353e3eb99cc` |
| 중요도 | B |
| 태그 | REALTIME, TEST |
| 원문에서 확인한 역할 | 단조 증가 순서와 사용자별 토큰 버킷의 경계·격리를 결정적 시계로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/inputGate.test.ts`
- 핵심 심벌: 가짜 시각을 사용하는 `InputGate.accept` 사례
- 폭주 8 이후 거부와 시간 진행에 따른 초당 보충량을 확인합니다.
- 오래된 순번을 반복해도 토큰이 줄지 않는지 확인합니다.
- 한 사용자의 여러 경기방이 허용 시간을 공유하고 다른 사용자는 격리되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1353e3eb99cc:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 입력 검사 구현은 순번과 빈도 두 상태를 결합했지만 검사 순서와 키 범위가 회귀로 고정되지 않았습니다. 오래된 입력이 토큰을 소모하거나 경기방마다 독립 버킷을 만들면 공격자가 제한을 우회하거나 정상 입력을 차단할 수 있습니다. |
| 구현 또는 검증 결정 | 주입 시계를 전진시키며 폭주/초당 보충량을 검사하고, 오래된 순번·동일 사용자 다중 경기방·다른 사용자 사례를 조합합니다. |
| 실행/검증 경로 | 초기 폭주 소비 → 9번째 거부 → 가상 시간 진행 → 초당 보충량 허용; 별도 사례에서 오래된 입력 후 최신 입력 허용; 경기방/사용자 키 범위 비교입니다. |
| 소유권과 실패 처리 | 테스트가 시각과 입력 순번을 명시하며 검사 상태는 사례별 새 인스턴스에 격리됩니다. 빈도 exhaustion과 오래된 순서를 서로 다른 거부 사유로 재현하고 오래된 경로가 허용 시간에 부수 효과를 만들지 않는지 확인합니다. |
| 보장하는 것 | 순서 검사가 빈도 charge보다 먼저이고, 허용 시간 범위가 사용자 단위라는 구현 규칙을 고정합니다. |
| 보장하지 않는 것 | WebSocket 오류 이벤트나 GameHub 클라이언트 정리는 이 단위 테스트 범위 밖입니다. |
| 후속 연결 | `207df3f47935`의 참가 불변 조건을 보호하며 `400ea1589260`이 실제 GameHub 경계를 추가 검증합니다. |
<!-- LEARNER-END:1353e3eb99cc:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:1353e3eb99cc:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 경계/격리 테스트 |
| 주입·재현 방식 | 제어 가능한 가상 시계, 여러 사용자/경기방 키, 연속 순번을 조합합니다. |
| 검증하는 것 | 순간 입력 폭주와 초당 토큰 보충 계산, 오래된 입력을 거부할 때 토큰을 소비하지 않는 동작, 사용자별 허용량을 검증합니다. |
| 검증하지 않는 것 | 네트워크 메시지 순서·스키마 검증·권한 검사는 검증하지 않습니다. |
<!-- LEARNER-END:1353e3eb99cc:test -->



#### 비교 기준

- 직전 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`
- 다음 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`

### 5.7. `feat(game): latest snapshot buffer 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `8589ff3c4821` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, OPERATIONS |
| 원문에서 확인한 역할 | 느린 전송 계층에서는 스냅샷 대기열을 쌓지 않고 최신 값만 보존하며, 혼잡이 계속되면 제한 시간 안에 연결을 종료합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 심벌: `LatestSnapshotBuffer.enqueue`, `flush`, `close`, 경미한/강한 버퍼 상한과 혼잡 타이머
- 대기 중 스냅샷 하나만 유지하고 새 스냅샷이 기존 대기 중 값을 교체하는지 확인합니다.
- `bufferedAmount`의 완화된 256 KiB와 강제 1 MiB 경계, 50 ms 재시도, 5초 혼잡 기한을 확인합니다.
- 이 최초 버전에서 완료되지 않은 전송 콜백을 나타내는 `sending` 상태가 혼잡 판단에 포함되는지 확인합니다.
- 교체/전달/혼잡/연결 종료가 어떤 콜백 또는 정리 경로로 관측 가능한지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8589ff3c4821:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 모든 시뮬레이션 스냅샷을 즉시 `socket.send`하면 느린 클라이언트마다 외부 전송 대기열이 무한히 늘고 오래된 상태가 최신 상태보다 늦게 전달될 수 있었습니다. |
| 해결하려던 문제와 위험 | 스냅샷은 중간 값을 모두 보존할 필요가 없지만 제어 이벤트와 달리 높은 빈도로 생성됩니다. 전송 계층 부하가 계속되면 프로세스 메모리와 지연 시간이 무한히 증가하므로 폐기와 종료 임계값이 필요했습니다. |
| 핵심 구현 결정 | 버퍼는 대기 중 스냅샷을 하나만 소유하고 다음 스냅샷이 오면 교체합니다. 전송 계층 `bufferedAmount`가 완화된 상한 이상이면 재시도하며, 강제 상한 초과 또는 5초 지속 혼잡이면 소켓을 강제 종료합니다. 이 SHA에서는 전송 콜백 미완료를 나타내는 `sending`도 새 전송을 막는 조건입니다. |
| 입력 → 처리 → 결과 | `enqueue(payload)` → 기존 대기 중이 있으면 교체 기록 → 대기 중 저장 → `flush` → 소켓이 닫혀 있으면 폐기/정리 → 강한 혼잡 상태이면 강제 종료 → 경미한 혼잡 상태이거나 전송 중이면 재시도 타이머 → 전송 가능하면 대기 중을 꺼내 전송 콜백을 기다리고 이후 다시 다음 전송을 시도합니다. |
| 소유 주체·수명·정리 | 클라이언트별 버퍼가 최신 메시지 본문, 추가 시각, 재시도 타이머, 혼잡 시작 시각, sending 상태를 단독 소유합니다. `close`는 타이머와 대기 중 값을 버리고 이후 추가를 받지 않습니다. |
| 실패·되돌리기·재시도 | 전송 콜백 오류, 이미 닫힌 소켓, 강제 버퍼, 5초 완화된 혼잡을 각각 종료 또는 폐기 경로로 처리합니다. 다만 콜백 지연 자체를 혼잡으로 간주하는 가정은 후속 `d90f17fa765d`에서 잘못된 것으로 판명됩니다. |
| 보장하는 것 | 외부 전송 스냅샷 대기열은 클라이언트당 최신 하나 값으로 제한되고 실제 버퍼에 쌓인 부하는 강제/시간 상한을 갖습니다. |
| 보장하지 않는 것 | 최초 구현은 `sending` 상태와 전송 계층 `bufferedAmount`를 동일한 부하 신호처럼 취급해 콜백이 느린 정상 소켓도 폐기/강제 종료할 수 있습니다. |
| 후속 연결 | `125aa113a01c`가 최초 동작 의미를 검증하고 `49ca3e778801`에서 GameHub에 연결됩니다. `d90f17fa765d`/`5cd54767858f`가 콜백 지연 오판을 나중에 교정합니다. |
<!-- LEARNER-END:8589ff3c4821:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:8589ff3c4821:fix -->
초기 가정: 전송 콜백 미완료도 혼잡이다 → 실제 위험: 콜백 지연과 커널/사용자 영역 버퍼 부하는 다르다 → `d90f17fa765d`에서 `bufferedAmount`만 부하 소스로 남깁니다.
<!-- LEARNER-END:8589ff3c4821:fix -->


#### 비교 기준

- 직전 관련 SHA: `1353e3eb99cc` — `test(game): input gate 제한 검증`
- 다음 관련 SHA: `125aa113a01c` — `test(game): snapshot replacement와 congestion 검증`

### 5.8. `test(game): snapshot replacement와 congestion 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `125aa113a01c` |
| 중요도 | A |
| 태그 | REALTIME, PERF, RISK |
| 원문에서 확인한 역할 | 최신 값만 유지하는 교체, 완화된 재시도, 강제 프로세스 종료, 혼잡 시간 초과를 제어되는 소켓 상태와 가상 시간으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.test.ts`
- 핵심 심벌: 테스트용 소켓의 `bufferedAmount`, 지연된 전송 콜백, 가상 타이머, 관측기 감시 객체
- 전송 중 여러 스냅샷을 추가할 때 중간 값은 교체되고 최신 값만 남는지 확인합니다.
- 완화된 부하에서 재시도 타이머가 진행되고 부하 해제 뒤 최신 스냅샷이 전송되는지 확인합니다.
- 강제 1 MiB 초과와 5초 지속 혼잡이 각각 강제 종료를 유발하는지 확인합니다.
- 테스트가 당시 `sending`을 혼잡 조건으로 기대한다는 점을 후속 수정과 비교합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:125aa113a01c:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 최신 버퍼는 중요한 자원 경계였지만 교체/폐기/프로세스 종료 기준이 소켓 구현에 따라 달라질 위험이 있었습니다. |
| 해결하려던 문제와 위험 | 실제 네트워크를 사용하면 bufferedAmount와 콜백 시점을 정밀하게 제어하기 어려워 경미한/강한 경계 및 5초 기한 회귀를 재현하기 어렵습니다. |
| 핵심 구현 결정 | 테스트용 소켓에서 `bufferedAmount`와 전송 콜백 완료를 직접 제어하고 가상 타이머로 50 ms 재시도와 5초 기한을 진행합니다. 관측기 호출과 실제 sent 메시지 본문을 함께 검사합니다. |
| 입력 → 처리 → 결과 | 첫 스냅샷 추가/전송 → 콜백 또는 버퍼 부하를 유지 → 후속 스냅샷 추가 → 최신 교체 확인 → 부하 해제 또는 기한 진행 → 전송/강제 종료/폐기 결과 확인입니다. |
| 소유 주체·수명·정리 | 테스트용 소켓이 전송 계층 혼잡도와 콜백 완료 시점을 제어하고, 버퍼가 재시도 타이머와 대기 중 메시지 본문을 정리하는지 확인합니다. |
| 실패·되돌리기·재시도 | 완화 가능한 부하의 복구, 즉시 강제 종료해야 하는 부하, 장시간 지속되어 종료하는 부하를 나누어 재현합니다. 당시에는 지연된 콜백도 혼잡 상태로 처리합니다. |
| 보장하는 것 | 최신 스냅샷 버퍼의 메모리 사용 상한와 프로세스 종료 규칙이 결정적으로 검증됩니다. |
| 보장하지 않는 것 | 테스트가 구현 당시의 잘못된 `sending == congestion` 가정도 고정합니다. 따라서 후속 수정에서는 기대값을 의도적으로 변경해야 합니다. |
| 후속 연결 | `8589ff3c4821`의 고위험 동작 의미 증거이며, `5cd54767858f`가 콜백 지연과 실제 버퍼에 쌓인 혼잡을 분리하는 새 회귀로 대체·확장합니다. |
<!-- LEARNER-END:125aa113a01c:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:125aa113a01c:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 역압/실패 테스트 |
| 주입·재현 방식 | 가짜 `bufferedAmount`, 수동 전송 콜백, 가상 타이머로 교체·경미한/강한 혼잡을 재현합니다. |
| 검증하는 것 | 최신 값만 유지하는 처리 보존 기간, 재시도/시간 초과, 강제 프로세스 종료, 정리를 검증합니다. |
| 검증하지 않는 것 | 실제 OS 소켓 버퍼 크기나 네트워크 처리량은 검증하지 않으며 당시 콜백 가정을 그대로 반영합니다. |
<!-- LEARNER-END:125aa113a01c:test -->

#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:125aa113a01c:fix -->
이 테스트가 고정한 콜백 부하 가정은 `d90f17fa765d`에서 근본 원인이 수정되고 `5cd54767858f`에서 새 기대값으로 회귀 보호됩니다.
<!-- LEARNER-END:125aa113a01c:fix -->


#### 비교 기준

- 직전 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:evolution -->
`3a2943ff385d`/`0888e119036d`는 경과 시간을 최대 5개의 50 ms 단계로 제한합니다. `10a656e59864`/`81031dcd2c1c`는 연결 생존 상태 타이머를 한 객체로 묶고, `207df3f47935`/`1353e3eb99cc`는 오래된 순서를 토큰 charge보다 먼저 판정합니다. `8589ff3c4821`/`125aa113a01c`는 외부 전송 스냅샷을 최신 하나 값으로 제한하지만 전송 콜백 미완료를 혼잡으로 보는 가정은 후속 통합 개발 흐름에서 수정됩니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:failure-links -->
- 이벤트 루프 장기 정지 → 누적 시간/누적 시간 보정 범위 제한 → 49/50 ms·10초 일시정지 회귀
- 반개방 연결 → ping/기한 수명주기 → 가상 시간 초과·응답 초기화
- 오래된 flood/빈도 폭주 → 순번 첫 토큰 검사 → 사용자/경기방 격리 테스트
- 외부 전송 대기열 → 최신 교체·경미한/강한 부하 → 제어되는 소켓 실패 테스트 → 콜백 오판은 `d90f17fa765d`에서 교정
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:ownership -->
`FixedStepScheduler`는 주기와 누적 시간을, `ConnectionHeartbeat`는 ping·시간 초과 핸들을, `InputGate`는 사용자 버킷과 사용자·경기방별 순번을, `LatestSnapshotBuffer`는 대기 중 메시지 본문과 재시도·혼잡 상태를 소유합니다. 이 단계에서는 GameHub가 아직 이 수명들을 실제 클라이언트와 경기방 수명주기에 연결하지 않습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:final-state -->
네 호출 제한기 기본 요소와 결정적 단위 근거가 존재합니다. 각 기본 요소의 작업·시간·메모리 상한은 정의됐지만 실제 경기방/클라이언트 생성·교체·일시정지·결과 확정에서 시작/중지/릴리스가 정확히 호출되는지는 다음 개발 흐름의 통합 책임입니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:final-flow -->
경과 시간은 고정 간격 단계 누산기로, 소켓 생존 상태는 연결 확인 신호 기한으로, 클라이언트 입력은 순번과 토큰 버킷으로, 외부 전송 스냅샷은 최신값 버퍼와 혼잡 상한으로 각각 독립 처리됩니다. 각 기본 요소는 자신의 상태 판정만 담당하고 경기방 상태 전이나 전송 연결 소속 정보는 관리하지 않습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:03-runtime-limiter-primitives-and-bounded-work.md:checks -->
- [x] 50 ms 시간 간격과 최대 5회 누적 시간 보정 산술을 설명할 수 있습니다.
- [x] 연결 확인 신호의 주기/시간 초과 핸들과 응답 초기화를 추적할 수 있습니다.
- [x] 오래된 입력이 토큰을 소비하지 않는 이유와 사용자 허용 시간 범위를 설명할 수 있습니다.
- [x] 최신 버퍼의 경미한/강한/시간 상한과 최초 콜백 지연 가정의 한계를 구분할 수 있습니다.
<!-- LEARNER-END:03-runtime-limiter-primitives-and-bounded-work.md:checks -->
===== END FILE: 03-runtime-limiter-primitives-and-bounded-work.md =====

===== BEGIN FILE: 04-gamehub-runtime-integration-shared-scheduling-and-congestion.md =====
# GameHub 실행 통합, 공유 스케줄링과 혼잡 제어

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

독립 호출 제한기를 GameHub의 경기방/클라이언트 수명주기에 연결하고, 경기방별 타이머를 공유 스케줄러로 이전하며, 시뮬레이션·스냅샷 전송 주기와 실제 전송 계층 혼잡을 분리하는 과정을 복원합니다.

범위 메모: 이 개발 흐름은 GameHub 내부 런타임 작업과 전송 상한을 다룹니다. 프로세스 종료 준비와 신호 종료는 다음 개발 흐름, 외부 k6·Toxiproxy와 DB 연결 풀 오류 격리는 마지막 개발 흐름으로 분리합니다.

### 직접 관련된 불변 조건

- 경기방이 실행 가능한일 때만 정확히 한 스케줄러 소속 정보를 가지며 모든 진행 중인 경기방은 하나의 내부 고정 간격 단계 시계를 공유합니다.
- 클라이언트 연결 확인 신호·입력 허용 시간·스냅샷 버퍼는 연결/사용자 범위에 맞춰 생성·이전·해제됩니다.
- 서버 기준 시뮬레이션은 20 Hz를 유지하고 정상 스냅샷 전달은 10 Hz로 분산됩니다.
- 전송 콜백 지연은 혼잡이 아니며 실제 대기 중인 바이트와 프레임 크기만 제한된 전송 계층 실패를 결정합니다.

## 2. 핵심 질문

- 기본 요소 시작/중지/릴리스가 준비 완료, 일시정지, 재연결, 교체, 종료, 종료 경로에 어떻게 연결됩니까?
- 경기방별 타이머에서 공유 타이머로 소유권이 이동할 때 실행 가능한 소속 정보를 누가 결정합니까?
- 20 Hz 시뮬레이션과 10 Hz 스냅샷 전달을 분리하면서 종료 이벤트는 어떻게 보존합니까?
- 콜백 완료, `bufferedAmount`, 프레임 `maxPayload`는 각각 어느 계층이 소유하는 신호입니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a6a1f4fba60e` | `feat(game): fixed-step scheduler를 GameHub에 연결` | A | SIMULATION, REALTIME, OBSERVABILITY | 각 경기방의 직접 주기를 `FixedStepScheduler`로 교체해 시뮬레이션 시간 소유 주체를 명시합니다. |
| 2 | `fc2a4451eed1` | `feat(game): heartbeat와 input gate를 GameHub에 연결` | A | PROTOCOL, REALTIME, RISK | 연결별 연결 확인 신호와 공유 입력 제한기를 GameHub 클라이언트 수명주기·메시지 경로에 통합합니다. |
| 3 | `49ca3e778801` | `feat(game): latest snapshot buffer를 GameHub에 연결` | A | PROTOCOL, REALTIME, RISK | 고빈도 스냅샷만 클라이언트별 최신 버퍼로 보내고 제어/수명주기 이벤트는 일반 전송 경로에 유지합니다. |
| 4 | `400ea1589260` | `test(game): GameHub runtime 제한 검증` | B | PROTOCOL, REALTIME, TEST | 입력 제한기가 실제 인증된 GameHub 메시지 경로에서 빈도 제한된 프로토콜 결과를 만드는지 검증합니다. |
| 5 | `aed88c8a93e0` | `perf(game): scheduler benchmark 실행 경계 추가` | B | REALTIME, OBSERVABILITY, PERF | 경기방별 타이머와 공유 타이머 구성을 같은 50 ms 주기·동일한 모의 단계 작업으로 비교하는 독립 실행 벤치마크를 만듭니다. |
| 6 | `8d24b5e70837` | `perf(game): scheduler benchmark 측정 결과 출력` | B | REALTIME, OBSERVABILITY, PERF | 스케줄러 벤치마크를 실행 환경 메타데이터, 측정값, 명시적 선택 규칙을 가진 JSON 보고서로 완성합니다. |
| 7 | `d21a47ee92d2` | `refactor(game): shared room scheduler 추가` | A | SIMULATION, REALTIME, REFACTOR | 모든 진행 중인 경기방을 하나의 고정 간격 단계 시계로 구동할 수 있는 `SharedRoomScheduler` 추상화를 도입합니다. |
| 8 | `518a8368e28f` | `test(game): shared room scheduler 검증` | B | REALTIME, TEST | 하나의 타이머 소유권, 경기방 등록·해제, 틱 중 소속 정보 변경을 결정적 시간으로 검증합니다. |
| 9 | `fb5b1abc97f5` | `refactor(game): GameHub가 shared room scheduler 사용` | A | SIMULATION, REALTIME, REFACTOR | 시뮬레이션 시간 제어 소유권을 경기방별 스케줄러에서 GameHub가 소유한 단일 공유 스케줄러로 이전합니다. |
| 10 | `69fb44d2f0ca` | `test(game): shared scheduler lifecycle 검증` | A | AUTH, SIMULATION, REALTIME | 연결 끊김·재연결·종료 동안 GameHub의 공유 스케줄러 등록 상태와 정리 전이를 검증합니다. |
| 11 | `ad482c200cea` | `fix(game): 부하 중 snapshot cadence 안정화` | A | SIMULATION, REALTIME, PERF | 서버 기준 시뮬레이션은 20 Hz로 유지하면서 클라이언트 스냅샷 전달을 10 Hz로 낮추고 경기방별 슬롯을 분산합니다. |
| 12 | `db1ae3d47b96` | `test(load): 기본 부하 병목 구간 검증` | B | SIMULATION, REALTIME, OPERATIONS | 여러 경기방에서 20 Hz 시뮬레이션과 시간을 분산한 10 Hz 스냅샷 전달이 분리되는지 결정적 GameHub 테스트로 검증합니다. |
| 13 | `d90f17fa765d` | `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음` | A | REALTIME, PERF, RISK | 완료되지 않은 WebSocket 전송 콜백을 혼잡 신호에서 제거하고 실제 `bufferedAmount`만 전송 계층 부하로 사용합니다. |
| 14 | `5cd54767858f` | `test(game): callback 지연과 실제 congestion 구분` | A | REALTIME, PERF, TEST | 지연된 전송 콜백은 정상 전달로, 높은 `bufferedAmount`는 실제 혼잡으로 처리되는지 분리해 검증합니다. |
| 15 | `8ea18a1b92db` | `fix(realtime): WebSocket transport payload 상한 설정` | A | AUTH, REALTIME, RISK | 애플리케이션 인증 전 상한과 동일한 8 KiB를 내부 `ws` 서버의 `maxPayload`에 설정합니다. |
| 16 | `1afec49052b6` | `test(realtime): oversized WebSocket frame 거부 검증` | B | AUTH, REALTIME, TEST | 실제 서버/소켓 경로에서 8,193-바이트 프레임이 종료 코드 1009로 거부되는지 검증합니다. |

## 5. 커밋별 학습 기록

### 5.1. `feat(game): fixed-step scheduler를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `a6a1f4fba60e` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, OBSERVABILITY |
| 원문에서 확인한 역할 | 각 경기방의 직접 주기를 `FixedStepScheduler`로 교체해 시뮬레이션 시간 소유 주체를 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 심벌: 경기방의 스케줄러 필드, `startGame`, 일시정지/재개/결과 확정 정리 경로
- 경기방 생성 시 스케줄러가 어떤 단계 콜백을 캡처하고 시뮬레이션 상태를 갱신하는지 확인합니다.
- 대기 중→경기 중에서 `start`, 연결 해제/일시정지에서 `stop`, 재연결/재개에서 재시작하는 호출 위치를 추적합니다.
- 점수 종료, 영속 저장 실패, 경기방 정리 경로가 스케줄러를 중지하는지 확인합니다.
- 한 경기방당 여전히 하나의 내부 타이머를 소유한다는 구성 한계를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:a6a1f4fba60e:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 고정 간격 단계 기본 요소는 존재했지만 GameHub 경기방은 직접 `setInterval` 기반 시간 제어를 계속 사용해 경과 시간 범위 제한과 결정적 수명주기 정리가 실제 코드 경로에 적용되지 않았습니다. |
| 해결하려던 문제와 위험 | 경기방 상태 전이와 타이머 시작/중지가 분리되면 대기 중·일시정지·종료된 경기방이 계속 시뮬레이션 작업을 만들거나 재연결 후 중복 타이머가 생길 수 있습니다. |
| 핵심 구현 결정 | 각 경기방에 `FixedStepScheduler`를 넣고 50 ms 단계 콜백이 서버 기준 시뮬레이션을 진행하도록 바꿉니다. 경기방이 경기 중 상태에 들어갈 때 시작하고 일시정지/결과 확정/제거에서 중지합니다. |
| 입력 → 처리 → 결과 | 경기방 생성 → 대기 중 상태로 스케줄러 보유 → 양측 준비 완료로 경기 중 전이 → 스케줄러 시작 → 고정 간격 단계마다 시뮬레이션·스냅샷/종료 판단 → 연결 해제면 일시정지/중지 → 재연결면 재개/시작 → 종료/정리이면 중지입니다. |
| 소유 주체·수명·정리 | 이 단계의 타이머 소유 주체는 여전히 각 경기방입니다. GameHub는 경기방 수명주기 전이 시 스케줄러 시작/중지를 호출하고 경기방 목록에서 제거하기 전에 핸들이 정리되도록 책임집니다. |
| 실패·되돌리기·재시도 | 시뮬레이션 콜백 또는 종료 비동기 작업의 오류가 경기방 타이머를 남기지 않도록 종료/정리 경로에서 중지합니다. 다만 경기방 수만큼 타이머가 늘어나는 구성은 해결하지 않습니다. |
| 보장하는 것 | 실제 GameHub 시뮬레이션도 50 ms 고정 간격 단계 범위 제한을 사용하고 경기방 수명주기와 타이머 상태가 연결됩니다. |
| 보장하지 않는 것 | 진행 중인 경기방 N개가 N개의 스케줄러와 타이머를 소유합니다. 전역 이벤트 루프를 깨우는 횟수와 경기방 순회 비용은 아직 한 곳에서 관리하지 않습니다. |
| 후속 연결 | `400ea1589260`이 전체 실행 경계를 검증하고, `d21a47ee92d2`/`fb5b1abc97f5`가 타이머 소유권을 공유 스케줄러로 이전합니다. |
<!-- LEARNER-END:a6a1f4fba60e:record -->




#### 비교 기준

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `fc2a4451eed1` — `feat(game): heartbeat와 input gate를 GameHub에 연결`

### 5.2. `feat(game): heartbeat와 input gate를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `fc2a4451eed1` |
| 중요도 | A |
| 태그 | PROTOCOL, REALTIME, RISK |
| 원문에서 확인한 역할 | 연결별 연결 확인 신호와 공유 입력 제한기를 GameHub 클라이언트 수명주기·메시지 경로에 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/connectionHeartbeat.ts`, `apps/api/src/game/inputGate.ts`
- 핵심 심벌: `GameHub.connect`, 클라이언트 레코드의 `heartbeat`, 입력 이벤트 처리 함수, 연결 해제/교체 정리
- 새 클라이언트 생성 시 연결 확인 신호 시작과 pong/activity 응답이 어디서 연결되는지 확인합니다.
- 검증된 `game.input`이 소속 정보/경기방/측 확인 뒤 `InputGate.accept`를 통과하는 순서를 확인합니다.
- 빈도 거부가 안정적인 `rate_limited` 프로토콜 오류로 바뀌고 오래된 입력은 어떤 방식으로 무시되는지 확인합니다.
- 마지막 사용자 연결이 사라질 때만 `releaseUser`를 호출해 교체가 허용 시간 상태를 잘못 지우지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:fc2a4451eed1:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 연결 확인 신호와 입력 제한기는 단위 수준으로만 존재해 실제 소켓은 시간 초과되지 않았고 GameHub는 입력 순서/빈도 상태를 사용하지 않았습니다. |
| 해결하려던 문제와 위험 | 기본 요소를 잘못 배치하면 인증되지 않은 사용자/인증되지 않은 상태 입력이 허용 시간을 소모하거나, 연결 교체 때 기존 소켓 정리가 새 소켓의 연결 확인 신호/입력 상태까지 제거할 수 있습니다. |
| 핵심 구현 결정 | 클라이언트 레코드에 연결 확인 신호를 넣어 연결 시 시작하고 종료/교체에서 중지합니다. 입력 처리 함수는 프로토콜 검증과 경기방 소유권을 확인한 뒤 InputGate를 호출하며 빈도 exhaustion만 클라이언트 표시되는 오류로 반환합니다. |
| 입력 → 처리 → 결과 | 인증된 연결 → 클라이언트/연결 확인 신호 생성·시작 → 메시지 파싱·소속 정보 확인 → 입력 제한기 오래된/빈도/수락 판정 → 수락된 방향만 시뮬레이션 상태에 반영 → pong은 응답 → 종료에서 연결 확인 신호 중지, 마지막 연결이면 사용자 검사 릴리스입니다. |
| 소유 주체·수명·정리 | 연결 확인 신호는 클라이언트별, InputGate의 버킷은 사용자별입니다. GameHub가 둘의 수명주기 어댑터이며 연결 교체 시 이전 클라이언트만 중지하고 사용자 수준 상태는 현재 유효한 연결 존재 여부에 따라 유지합니다. |
| 실패·되돌리기·재시도 | ping/시간 초과는 소켓 강제 종료로, 빈도 초과는 프로토콜 `rate_limited`로, 오래된 입력은 상태 변경 없이 종료됩니다. 잘못된/forbidden 메시지는 입력 제한기 이전 경계에서 거부됩니다. |
| 보장하는 것 | 운영 WebSocket 경로에서 연결 생존 확인과 순번·빈도 제한을 통과한 입력 수락이 실제로 적용되고 정리 범위가 클라이언트/사용자로 구분됩니다. |
| 보장하지 않는 것 | 스냅샷 전달은 아직 일반 전송 경로이고 느린 클라이언트의 전송 대기열은 해결되지 않습니다. |
| 후속 연결 | `49ca3e778801`이 스냅샷 버퍼를 같은 클라이언트 수명주기에 추가하고 `400ea1589260`이 입력 throttling을 종단 간 검증합니다. |
<!-- LEARNER-END:fc2a4451eed1:record -->




#### 비교 기준

- 직전 관련 SHA: `a6a1f4fba60e` — `feat(game): fixed-step scheduler를 GameHub에 연결`
- 다음 관련 SHA: `49ca3e778801` — `feat(game): latest snapshot buffer를 GameHub에 연결`

### 5.3. `feat(game): latest snapshot buffer를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `49ca3e778801` |
| 중요도 | A |
| 태그 | PROTOCOL, REALTIME, RISK |
| 원문에서 확인한 역할 | 고빈도 스냅샷만 클라이언트별 최신 버퍼로 보내고 제어/수명주기 이벤트는 일반 전송 경로에 유지합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 심벌: 클라이언트 레코드의 `snapshots`, 스냅샷 전파 경로, 일반 `send`, 클라이언트 정리
- `game.snapshot`만 `LatestSnapshotBuffer.enqueue`로 라우팅되고 대기열/오류/종료된 같은 제어 이벤트는 직접 전송되는지 확인합니다.
- 클라이언트 생성 시 버퍼 관측기와 소켓을 결합하고 종료/교체에서 `snapshots.close()`를 호출하는지 확인합니다.
- 일반 이벤트가 강제 버퍼에 쌓인 임계값을 넘으면 강제 종료하고 전송 콜백 오류가 열린 소켓을 종료하는지 확인합니다.
- 스냅샷 교체가 경기방 시뮬레이션이나 다른 클라이언트 전달을 막지 않는지 호출자 루프를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:49ca3e778801:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 모든 서버 이벤트가 같은 직접 전송 경로를 사용해 스냅샷 폭주가 느린 클라이언트의 대기열을 누적시킬 수 있었습니다. |
| 해결하려던 문제와 위험 | 스냅샷은 최신 상태만 중요하지만 대기열 경기, 오류, 게임 종료된 같은 제어 이벤트는 손실시키면 수명주기가 깨집니다. 두 종류를 같은 폐기 정책으로 처리할 수 없습니다. |
| 핵심 구현 결정 | 클라이언트마다 `LatestSnapshotBuffer`를 만들고 스냅샷 이벤트만 추가합니다. 제어 이벤트는 직접 전송을 유지하되 전송 계층 강제 부하와 콜백 오류에서 연결을 종료합니다. |
| 입력 → 처리 → 결과 | 시뮬레이션 틱 → 스냅샷 이벤트 생성 → 각 클라이언트의 최신값 버퍼에 추가·교체·재시도; 제어 이벤트 → 일반 전송 → 콜백 완료 또는 오류 처리; 클라이언트 종료 → 연결 확인 신호와 스냅샷 버퍼 종료 순서입니다. |
| 소유 주체·수명·정리 | GameHub 클라이언트 레코드가 연결 확인 신호와 스냅샷 두 전송 계층 자원을 함께 소유합니다. 스냅샷 버퍼는 대기 중/재시도를, 일반 전송은 해당 호출의 콜백을 소유합니다. |
| 실패·되돌리기·재시도 | 스냅샷에는 경미한 혼잡과 강한 혼잡에 서로 다른 처리 규칙을 적용합니다. 제어 이벤트는 강한 혼잡이나 전송 오류가 발생하면 소켓을 닫아, 필수 상태 변경 알림이 조용히 유실되는 상황을 피합니다. |
| 보장하는 것 | 고빈도 상태 갱신 대기열은 최신 값 하나만 유지하도록 제한하면서 필수 제어 이벤트는 임의 폐기되지 않습니다. |
| 보장하지 않는 것 | 최초 버퍼의 `sending` 오판이 그대로 통합되며, 경기방별 타이머 구성도 유지됩니다. |
| 후속 연결 | `d90f17fa765d`가 콜백 부하 가정을 수정하고 `8ea18a1b92db`가 프레임 자체의 전송 계층 크기 상한을 추가합니다. |
<!-- LEARNER-END:49ca3e778801:record -->




#### 비교 기준

- 직전 관련 SHA: `fc2a4451eed1` — `feat(game): heartbeat와 input gate를 GameHub에 연결`
- 다음 관련 SHA: `400ea1589260` — `test(game): GameHub runtime 제한 검증`

### 5.4. `test(game): GameHub runtime 제한 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `400ea1589260` |
| 중요도 | B |
| 태그 | PROTOCOL, REALTIME, TEST |
| 원문에서 확인한 역할 | 입력 제한기가 실제 인증된 GameHub 메시지 경로에서 빈도 제한된 프로토콜 결과를 만드는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.runtime.test.ts`
- 핵심 심벌: 테스트용 소켓, AI 경기방 설정, `game.input` 폭주, 파싱한 서버 errors
- 테스트용 소켓을 GameHub에 연결하고 AI 경기방을 준비 완료/경기 중으로 만드는 설정을 확인합니다.
- 연속 9개 입력 중 폭주 8 이후 안정적인 `rate_limited` 오류가 한 번 발생하는지 확인합니다.
- 단위 `InputGate`가 아니라 메시지 파싱·경기방 참가 상태·전송 경로까지 통과하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:400ea1589260:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | InputGate 단위 테스트는 산술을 검증했지만 GameHub가 올바른 사용자/경기방 키와 프로토콜 오류로 연결했는지는 검증하지 못했습니다. 통합 중 검사 호출 위치나 오류 매핑이 잘못되면 기본 요소가 있어도 실제 소켓 경로가 제한을 우회할 수 있습니다. |
| 구현 또는 검증 결정 | 메모리 저장소와 테스트용 소켓으로 AI 경기를 만들고 유효한 입력 이벤트를 폭주로 주입해 외부 전송 서버 이벤트를 파싱합니다. |
| 실행/검증 경로 | 연결 → 대기열 AI → 준비 완료 → 경기 중 → 순번이 증가하는 입력 9개 전송 → 8개 수락 후 빈도 제한된 오류 확인입니다. |
| 소유권과 실패 처리 | 테스트가 소켓 이벤트 전달과 저장소 정리를 담당하고, GameHub가 클라이언트 연결 확인 신호·버퍼·입력 제한 상태를 정리합니다. 실제 전체 메시지 처리 경로에서 토큰 버킷의 허용량 소진을 재현하며, 오래된 연결이나 인증되지 않은 연결은 이 커밋의 중심 사례가 아닙니다. |
| 보장하는 것 | GameHub가 호출 빈도 제한 기본 요소를 우회하지 않고 안정적인 프로토콜 오류로 노출함을 검증합니다. |
| 보장하지 않는 것 | 실제 WebSocket 전송 계층, 연결 확인 신호 시간 초과, 외부 전송 혼잡은 검증하지 않습니다. |
| 후속 연결 | `fc2a4451eed1`의 통합 근거이며 이후 공유 스케줄러 변경과 독립적으로 입력 수락을 보호합니다. |
<!-- LEARNER-END:400ea1589260:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:400ea1589260:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | GameHub 통합 테스트 |
| 주입·재현 방식 | 테스트용 소켓과 메모리 저장소로 실제 이벤트 파서·경기방 설정·입력 처리 함수·서버 오류 경로를 실행합니다. |
| 검증하는 것 | 폭주 8 이후 9번째 유효한 입력이 빈도 제한 오류로 관측됨을 검증합니다. |
| 검증하지 않는 것 | 네트워크 처리량·여러 프로세스 호출 빈도 제한·저장된 상태는 검증하지 않습니다. |
<!-- LEARNER-END:400ea1589260:test -->



#### 비교 기준

- 직전 관련 SHA: `49ca3e778801` — `feat(game): latest snapshot buffer를 GameHub에 연결`
- 다음 관련 SHA: `aed88c8a93e0` — `perf(game): scheduler benchmark 실행 경계 추가`

### 5.5. `perf(game): scheduler benchmark 실행 경계 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `aed88c8a93e0` |
| 중요도 | B |
| 태그 | REALTIME, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 경기방별 타이머와 공유 타이머 구성을 같은 50 ms 주기·동일한 모의 단계 작업으로 비교하는 독립 실행 벤치마크를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/scheduler-benchmark.mjs`
- 핵심 심벌: `measure(strategy, roomCount)`, `simulateRoomStep`, percentile helpers
- 경기방 수 1·20·50·100, 50ms 시간 간격, 250ms 준비 구간, 기본 1.5초 실행 시간, 3회 반복 설정을 확인합니다.
- `room` 전략은 경기방마다 주기를 만들고 `shared` 전략은 하나의 주기에서 모든 경기방을 순회하는지 확인합니다.
- 두 전략이 같은 `simulateRoomStep` 작업을 수행하며 p95/p99 지연 시간 예시를 어떻게 수집하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:aed88c8a93e0:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 경기방별 스케줄러가 기능적으로 동작했지만 타이머 수를 중앙화할 근거와 비교 가능한 측정 경계가 없었습니다. 공유 스케줄러로의 리팩터링이 단순 취향이 되지 않으려면 작업과 주기를 통제한 구성 비교가 필요했습니다. |
| 구현 또는 검증 결정 | 독립 실행 Node 스크립트가 경기방별 주기와 공유 주기를 같은 경기방 수와 단계 작업량으로 실행하고 콜백 지연 표본을 수집합니다. |
| 실행/검증 경로 | 전략과 경기방 수 선택 → 준비 구간 → 고정된 50 ms 콜백에서 합성 경기방 작업 → p95/p99 예시 산출입니다. |
| 소유권과 실패 처리 | 벤치마크가 만든 모든 타이머를 배열로 소유하고 실행 시간이 끝나면 전부 해제합니다. 운영 GameHub 상태는 사용하지 않습니다. 환경과 CPU 잡음은 반복 횟수와 준비 구간으로 줄이지만 완전히 제거하지 못합니다. 입력값 검증과 결과 판단 출력은 다음 커밋에서 완성됩니다. |
| 보장하는 것 | 두 타이머 구성을 동일 조건으로 비교할 재현 가능한 실행 경계가 생깁니다. |
| 보장하지 않는 것 | 실제 시뮬레이션, 소켓, 영속 저장을 실행하지 않으며 커밋 자체는 측정 결과나 선택 결정을 출력하지 않습니다. |
| 후속 연결 | `8d24b5e70837`가 결과 스키마와 경기방 50개 기준의 판단 규칙을 추가하고 `d21a47ee92d2`가 선택된 공유 스케줄러를 구현합니다. |
<!-- LEARNER-END:aed88c8a93e0:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:aed88c8a93e0:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 독립 실행 마이크로벤치마크 테스트 실행 틀 |
| 주입·재현 방식 | 동일 synthetic CPU 작업과 50 ms 주기에서 경기방 각 타이머와 하나 공유 타이머를 비교합니다. |
| 검증하는 것 | 실행 시 구성별 콜백 lag를 같은 테스트 실행 틀로 측정할 수 있음을 검증합니다. |
| 검증하지 않는 것 | 이 워크북에서는 벤치마크를 실행하지 않았으므로 구체 p95 수치나 우열은 실행 근거로 주장하지 않습니다. |
<!-- LEARNER-END:aed88c8a93e0:test -->



#### 비교 기준

- 직전 관련 SHA: `400ea1589260` — `test(game): GameHub runtime 제한 검증`
- 다음 관련 SHA: `8d24b5e70837` — `perf(game): scheduler benchmark 측정 결과 출력`

### 5.6. `perf(game): scheduler benchmark 측정 결과 출력`

| 항목 | 값 |
| --- | --- |
| SHA | `8d24b5e70837` |
| 중요도 | B |
| 태그 | REALTIME, OBSERVABILITY, PERF |
| 원문에서 확인한 역할 | 스케줄러 벤치마크를 실행 환경 메타데이터, 측정값, 명시적 선택 규칙을 가진 JSON 보고서로 완성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/scheduler-benchmark.mjs`
- 핵심 심벌: 최상위 측정 반복문, 중앙값 집계, `decision.selectedStrategy`
- 각 경기방 개수와 두 전략를 기본 3회 반복하고 p95/p99의 중앙값을 계산하는지 확인합니다.
- Node/플랫폼/CPU/메모리와 벤치마크 설정를 보고서에 포함해 결과의 실행 환경을 보존하는지 확인합니다.
- 경기방 50개에서 공유 스케줄러의 p95가 경기방별 타이머 p95의 105% 이하일 때 공유 방식을 선택하는 판단 기준을 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8d24b5e70837:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 벤치마크는 표본을 반환했지만 실행 환경, 반복 집계, 선택 기준을 하나의 결과물로 남기지 않았습니다. 숫자만 출력하면 언제 어떤 설정으로 측정했는지 재현하기 어렵고 리팩터링 판단 기준을 사후에 바꿀 수 있습니다. |
| 구현 또는 검증 결정 | 모든 조합을 반복 실행해 p95와 p99의 중앙값을 모으고 실행 시점, 설정, 측정값, 판단을 JSON으로 출력합니다. 경기방 50개 비교에서 공유 스케줄러의 p95가 경기방별 스케줄러보다 5% 이상 나쁘지 않으면 공유 방식을 선택합니다. |
| 실행/검증 경로 | 경기방 개수×전략×반복 실행 → 실행 표본 집계 → 중앙값 산출 → 경기방 50개 결과 비교 → 1.05 임계값 계산 → JSON 보고서 출력입니다. |
| 소유권과 실패 처리 | 스크립트가 측정 배열과 실행 환경 메타데이터를 한 보고서 객체에 모으며 운영 산출물이나 저장소 상태를 변경하지 않습니다. 경기방 50개 결과가 없으면 명시적으로 예외를 발생시킵니다. 환경 잡음과 짧은 실행 시간은 여전히 결과 해석의 제한입니다. |
| 보장하는 것 | 스케줄러 선택이 명시된 비교 규칙과 실행 메타데이터를 가진 재현 가능한 보고서 형태로 남습니다. |
| 보장하지 않는 것 | 코드는 결과 스키마를 정의할 뿐 특정 실행 환경의 수치는 불변 조건이 아닙니다. 이번 작업에서도 벤치마크 명령을 실행하지 않았습니다. |
| 후속 연결 | `d21a47ee92d2`의 공유 스케줄러 추상화를 선택하는 사전 운영 근거입니다. |
<!-- LEARNER-END:8d24b5e70837:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:8d24b5e70837:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 벤치마크/보고서 계약 |
| 주입·재현 방식 | 반복 중앙값과 경기방 50개에서 사용하는 5% 판단 임계값을 소스에서 확인했습니다. |
| 검증하는 것 | 실행 결과가 전략 선택을 포함한 구조화된 JSON으로 재현될 수 있음을 검증합니다. |
| 검증하지 않는 것 | 공유 구성이 모든 운영 부하에서 더 빠르다는 보편적 성능 보장은 하지 않습니다. |
<!-- LEARNER-END:8d24b5e70837:test -->



#### 비교 기준

- 직전 관련 SHA: `aed88c8a93e0` — `perf(game): scheduler benchmark 실행 경계 추가`
- 다음 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`

### 5.7. `refactor(game): shared room scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `d21a47ee92d2` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, REFACTOR |
| 원문에서 확인한 역할 | 모든 진행 중인 경기방을 하나의 고정 간격 단계 시계로 구동할 수 있는 `SharedRoomScheduler` 추상화를 도입합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/sharedRoomScheduler.ts`, `apps/api/src/game/fixedStepScheduler.ts`
- 핵심 심벌: `SharedRoomScheduler.register`, `unregister`, `stop`, 경기방 단계 목록
- 첫 경기방 등록에서 내부 스케줄러가 시작되고 마지막 등록 해제에서 중지되는지 확인합니다.
- 경기방 ID→단계 콜백 목록이 중복 등록과 등록 해제를 어떻게 처리하는지 확인합니다.
- 틱 시작 시 콜백 값을 복사해 단계 도중 등록/해제 변경이 현재 순회를 깨지 않는지 확인합니다.
- `stop`이 타이머와 모든 경기방 참가 상태를 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:d21a47ee92d2:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub 통합은 경기방마다 하나의 `FixedStepScheduler`를 만들어 진행 중인 경기방 수만큼 주기 타이머 핸들을 생성했습니다. |
| 해결하려던 문제와 위험 | 경기방별 타이머는 같은 50 ms 주기인데도 콜백 깨우기과 누적 시간 상태가 경기방 수에 비례합니다. 한 경기방 단계가 수명주기 중 목록을 변경할 때 공유 순회 안정성도 필요합니다. |
| 핵심 구현 결정 | `SharedRoomScheduler`가 하나의 `FixedStepScheduler`와 경기방 콜백 목록을 소유합니다. 첫 등록 시 시계를 시작하고 매 고정 간격 단계마다 등록 콜백 스냅샷을 순회하며 마지막 해제에서 시계를 멈춥니다. |
| 입력 → 처리 → 결과 | `register(roomId, step)` → 목록 추가 → 첫 경기방이면 내부 시작 → 틱에서 현재 콜백 복사·각 단계 호출 → `unregister` → 목록 빈면 내부 중지 → 전역 `stop`은 타이머와 목록을 정리합니다. |
| 소유 주체·수명·정리 | 타이머와 누적 시간은 공유 스케줄러 하나가 소유하고 경기방은 콜백 소속 정보만 등록합니다. 경기방 수명주기 소유 주체는 등록/해제를 정확히 호출해야 합니다. |
| 실패·되돌리기·재시도 | 단계 콜백 실행 중 경기방이 자신이나 다른 경기방을 등록 해제해도, 복사한 콜백 목록을 순회하므로 항목 누락이나 반복자 손상이 발생하지 않습니다. 콜백 예외를 격리할지는 호출자가 결정합니다. |
| 보장하는 것 | 동일 주기의 진행 중인 경기방이 하나의 타이머 소유 주체를 공유하고 등록된 콜백 집합으로만 시뮬레이션 작업이 발생합니다. |
| 보장하지 않는 것 | 이 SHA는 구성 요소만 추가하며 기존 GameHub의 경기방별 스케줄러를 아직 교체하지 않습니다. |
| 후속 연결 | `518a8368e28f`가 중앙에서 관리하는 소유권을 검증하고 `fb5b1abc97f5`가 GameHub에 실제 적용합니다. |
<!-- LEARNER-END:d21a47ee92d2:record -->




#### 비교 기준

- 직전 관련 SHA: `8d24b5e70837` — `perf(game): scheduler benchmark 측정 결과 출력`
- 다음 관련 SHA: `518a8368e28f` — `test(game): shared room scheduler 검증`

### 5.8. `test(game): shared room scheduler 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `518a8368e28f` |
| 중요도 | B |
| 태그 | REALTIME, TEST |
| 원문에서 확인한 역할 | 하나의 타이머 소유권, 경기방 등록·해제, 틱 중 소속 정보 변경을 결정적 시간으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/sharedRoomScheduler.test.ts`
- 핵심 심벌: 가상 타이머, 등록 사용자 경기방 콜백, 타이머 개수/수명주기 검증
- 두 경기방을 등록해도 내부 주기 하나만 존재하는지 확인합니다.
- 한 경기방 해제 뒤 다른 경기방은 계속 진행되고 마지막 경기방 해제에서 타이머가 중지되는지 확인합니다.
- 첫 콜백이 틱 도중 등록을 해제해도 뒤쪽 콜백을 건너뛰지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:518a8368e28f:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 공유 스케줄러 추상화는 목록 순회와 첫/마지막 소속 정보 상태 전이에 놓치기 쉬운 수명주기 오류가 생길 수 있었습니다. 틱 중 목록 변경은 JavaScript 반복자 동작에 의존하면 일부 경기방을 건너뛰거나 중복 처리하는 결과를 만들 수 있고, 마지막 경기방 뒤 타이머가 남으면 쓸모없는 작업이 지속됩니다. |
| 구현 또는 검증 결정 | 가상 타이머로 동일 틱의 경기방 콜백과 타이머 개수를 관찰하고 콜백 내부 등록 해제 사례를 재현합니다. |
| 실행/검증 경로 | 경기방 A/B 등록 → 하나 타이머 검증 → 틱 → A 콜백에서 소속 정보 변경 → B 호출 확인 → 해제 순서에 따른 타이머 중지 확인입니다. |
| 소유권과 실패 처리 | 테스트가 타이머 대기열을 소유하며 스케줄러의 목록과 내부 핸들 정리를 외부 호출 횟수로 검증합니다. 순회 중 목록 변경과 마지막 등록 해제 시 발생할 수 있는 누수를 결정적으로 재현합니다. |
| 보장하는 것 | central 타이머와 소속 정보 스냅샷 순회의 핵심 소유권 규칙을 고정합니다. |
| 보장하지 않는 것 | GameHub가 모든 경기방 상태 전이에서 등록/해제를 호출하는지는 검증하지 않습니다. |
| 후속 연결 | `d21a47ee92d2`의 추상화 근거이고 `69fb44d2f0ca`가 GameHub 복구 수명주기까지 확장합니다. |
<!-- LEARNER-END:518a8368e28f:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:518a8368e28f:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 스케줄러 수명주기 테스트 |
| 주입·재현 방식 | 가상 타이머와 콜백 내 등록 해제로 단일 타이머와 변경 안전성을 검사합니다. |
| 검증하는 것 | 첫 등록에서 타이머가 시작되고 마지막 등록 해제에서 멈추며, 틱 도중 등록 목록이 바뀌어도 뒤쪽 경기방을 건너뛰지 않음을 검증합니다. |
| 검증하지 않는 것 | 실제 경기방 상태 기계·시뮬레이션 cost·소켓 전달은 검증하지 않습니다. |
<!-- LEARNER-END:518a8368e28f:test -->



#### 비교 기준

- 직전 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`
- 다음 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`

### 5.9. `refactor(game): GameHub가 shared room scheduler 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `fb5b1abc97f5` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, REFACTOR |
| 원문에서 확인한 역할 | 시뮬레이션 시간 제어 소유권을 경기방별 스케줄러에서 GameHub가 소유한 단일 공유 스케줄러로 이전합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/game/sharedRoomScheduler.ts`
- 핵심 심벌: GameHub의 `roomScheduler`, 경기방 등록/해제 도우미 함수, `liveStats().scheduledRooms` 관련 경로
- 경기방 객체에서 경기방별 스케줄러 상태가 제거되고 GameHub 생성자가 하나의 `SharedRoomScheduler`를 소유하는지 확인합니다.
- 준비 완료·경기 시작, 연결 해제·일시정지, 재연결·재개, 포기 처리, 결과 확정, 경기방 제거의 모든 경로에서 등록/해제가 대칭인지 추적합니다.
- 동일 경기방의 중복 등록과 오래된 정리가 스케줄러 소속 정보를 어떻게 보존하는지 확인합니다.
- `GameHub.close` 또는 최종 정리가 공유 스케줄러를 중지하는지 당시 구현 범위를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:fb5b1abc97f5:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공유 스케줄러 추상화는 존재했지만 GameHub 경기방은 계속 각자의 FixedStepScheduler를 소유했습니다. 운영 타이머 구성은 아직 경기방 수에 비례했습니다. |
| 해결하려던 문제와 위험 | 소유권 이전에서 상태 전이 하나라도 등록 해제를 놓치면 일시정지/종료된 경기방이 계속 진행되고, 중복 등록은 같은 경기방을 두 번 진행할 수 있습니다. 반대로 연결 해제에서 너무 일찍 제거하면 재연결 가능한 경기방이 재개되지 않습니다. |
| 핵심 구현 결정 | GameHub에 하나의 `SharedRoomScheduler`를 두고 실행 가능한 경기방 ID와 단계 콜백만 등록합니다. 경기방 객체의 스케줄러를 제거하고 수명주기 상태 전이마다 schedule/unschedule 도우미 함수를 호출합니다. |
| 입력 → 처리 → 결과 | GameHub 생성 → 공유 스케줄러 하나 생성 → 경기방 경기 중 시 경기방 ID 등록 → 공유 틱에서 시뮬레이션 단계 → 일시정지/연결 해제 시 해제 → 유효한 재연결 시 재등록 → 종료·포기 처리·제거 시 최종 해제입니다. |
| 소유 주체·수명·정리 | GameHub가 타이머 구성의 유일한 소유 주체가 됩니다. RoomSession은 허용된 상태를 결정하고, GameHub가 그 상태에 맞춰 스케줄러 소속 정보를 반영하며, 공유 스케줄러는 콜백 목록과 타이머 핸들만 소유합니다. |
| 실패·되돌리기·재시도 | 부분 반영 경기방 생성, 연결 해제, 재연결 만료, 결과 확정 실패 같은 여러 종료에서 등록 해제가 누락되지 않도록 제거 도우미 함수에 정리 책임을 집중합니다. 영속 저장 재시도가 진행 중이어도 종료 경기방은 시뮬레이션 소속 정보에서 분리됩니다. |
| 보장하는 것 | 활성 실행 가능한 경기방 수와 무관하게 내부 고정 간격 단계 타이머는 하나이며, 경기방 참가 상태가 시뮬레이션 작업의 유일한 진입 조건이 됩니다. |
| 보장하지 않는 것 | 스냅샷 전송 주기는 아직 모든 시뮬레이션 틱과 결합돼 있고, 느린 전송 콜백을 혼잡으로 보는 버퍼 가정도 남아 있습니다. |
| 후속 연결 | `69fb44d2f0ca`가 재연결/종료 수명주기의 소속 정보를 검증하고 `ad482c200cea`가 공유 틱 위에서 스냅샷 전달 주기를 분리합니다. |
<!-- LEARNER-END:fb5b1abc97f5:record -->




#### 비교 기준

- 직전 관련 SHA: `518a8368e28f` — `test(game): shared room scheduler 검증`
- 다음 관련 SHA: `69fb44d2f0ca` — `test(game): shared scheduler lifecycle 검증`

### 5.10. `test(game): shared scheduler lifecycle 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `69fb44d2f0ca` |
| 중요도 | A |
| 태그 | AUTH, SIMULATION, REALTIME |
| 원문에서 확인한 역할 | 연결 끊김·재연결·종료 동안 GameHub의 공유 스케줄러 등록 상태와 정리 전이를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.reconnect.test.ts`
- 핵심 심벌: 테스트용 소켓/저장소, 스케줄러에 등록된 경기방 수, 재연결/종료 상태 전이 검증
- 두 플레이어를 경기방에 연결하고 준비 완료 이후 스케줄러에 등록된 경기방 수가 1이 되는 설정을 확인합니다.
- 한 연결이 끊겨 경기방이 일시정지/예약된 상태가 될 때 소속 정보가 0으로 내려가는지 확인합니다.
- 같은 사용자의 재연결이 기존 측을 회복한 뒤 소속 정보가 다시 1이 되고 새 경기방을 만들지 않는지 확인합니다.
- 경기 종료/결과 확정 뒤 소속 정보가 0이고 영속 저장이 한 번만 실행되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:69fb44d2f0ca:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공유 스케줄러 단위 테스트는 목록과 타이머만 검증했으며 GameHub의 경기방/재연결 상태 기계가 소속 정보를 올바르게 반영하는지는 열려 있었습니다. |
| 해결하려던 문제와 위험 | 재연결은 전송 계층 소유권을 교체하면서 경기방은 보존하는 복합 전이입니다. 일시정지에서 계속 단계하거나 재연결이 두 번째 소속 정보를 만들면 시뮬레이션과 결과가 중복됩니다. |
| 핵심 구현 결정 | 가상 타이머와 소켓으로 실제 GameHub 경기를 만든 뒤 연결 해제, 재연결, 종료를 순서대로 진행하고 스케줄러에 등록된 경기방 수와 결과 확정 호출을 확인합니다. |
| 입력 → 처리 → 결과 | 경기방 생성·준비 완료 → 등록된 경기방 수 1 → 플레이어 연결 해제 → 일시정지·등록 수 0 → 소켓 교체 또는 재연결 → 같은 경기방 재개·등록 수 1 → 경기 종료 → 등록 수 0·결과 확정 한 번 순서입니다. |
| 소유 주체·수명·정리 | 테스트는 전송 계층 이벤트와 가상 시간을 구동하고 GameHub가 경기방 참가 상태, 공유 타이머, 재연결 예약, 저장소 결과 확정을 조정합니다. |
| 실패·되돌리기·재시도 | 오래된 소켓이나 중복 재연결이 추가 경기방/타이머를 만들지 않는지, 종료 뒤 스케줄러 작업이 남지 않는지 확인합니다. |
| 보장하는 것 | 공유 스케줄러 소유권 이전이 실제 재연결 수명주기와 결합돼 한 경기방을 한 번만 스케줄링한다는 고위험 불변 조건을 검증합니다. |
| 보장하지 않는 것 | 다수 경기방의 스냅샷 전송 주기와 전송 계층 혼잡 동작은 이 테스트 범위가 아닙니다. |
| 후속 연결 | `fb5b1abc97f5`의 소유권 이전을 보호하고 `db1ae3d47b96`이 여러 경기방 주기를 추가 검증합니다. |
<!-- LEARNER-END:69fb44d2f0ca:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:69fb44d2f0ca:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 GameHub 수명주기 통합 테스트 |
| 주입·재현 방식 | 테스트용 소켓, 타이머, 저장소로 준비 완료 → 연결 해제 → 재연결 → 종료 상태를 구동하고 스케줄러 소속을 관찰합니다. |
| 검증하는 것 | 경기방의 스케줄러 등록 수가 1→0→1→0으로 변하고, 같은 경기방으로 복구되며, 결과가 한 번만 확정되는지 검증합니다. |
| 검증하지 않는 것 | 실제 네트워크 지연 시간, 프로세스 재시작, PostgreSQL 트랜잭션은 검증하지 않습니다. |
<!-- LEARNER-END:69fb44d2f0ca:test -->



#### 비교 기준

- 직전 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`
- 다음 관련 SHA: `ad482c200cea` — `fix(game): 부하 중 snapshot cadence 안정화`

### 5.11. `fix(game): 부하 중 snapshot cadence 안정화`

| 항목 | 값 |
| --- | --- |
| SHA | `ad482c200cea` |
| 중요도 | A |
| 태그 | SIMULATION, REALTIME, PERF |
| 원문에서 확인한 역할 | 서버 기준 시뮬레이션은 20 Hz로 유지하면서 클라이언트 스냅샷 전달을 10 Hz로 낮추고 경기방별 슬롯을 분산합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.ts`, `apps/api/src/observability.ts`
- 핵심 심벌: 공유 틱 순번, 스냅샷 divisor/경기방 전달 슬롯, 결과 확정 관측기 개수
- 50 ms 시뮬레이션 단계는 그대로 실행하면서 스냅샷 생성/전송은 매 두 번째 틱에만 수행하는지 확인합니다.
- 경기방 ID 또는 등록 순서에서 번갈아 적용하는 전달 슬롯을 정해 같은 틱의 스냅샷 폭주를 분산하는지 확인합니다.
- 경기 종료 이벤트가 낮아진 스냅샷 전송 주기 때문에 지연되거나 누락되지 않는지 별도 제어 경로를 확인합니다.
- 결과 확정 지표가 `created` 여부와 중복 처리 의미를 구분하도록 함께 조정된 부분을 실제 변경 내용에서 분리해 기록합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:ad482c200cea:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 공유 스케줄러가 모든 경기방을 같은 20 Hz 틱에서 순회하고 매 틱마다 스냅샷을 보내므로, 50 경기방에서는 동일 콜백에 전달 폭주가 집중됐습니다. |
| 해결하려던 문제와 위험 | 시뮬레이션 주기를 낮추면 서버 기준 물리 계산이 바뀌지만, 스냅샷을 그대로 20 Hz로 유지하면 직렬화·WebSocket 전달이 이벤트 루프를 압박합니다. 모든 경기방을 같은 전달 틱에 보내는 것도 폭주를 만듭니다. |
| 핵심 구현 결정 | 시뮬레이션 단계는 50 ms마다 계속 수행하고 스냅샷 전송 주기만 divisor 2로 분리해 10 Hz로 만듭니다. 경기방마다 번갈아 적용하는 슬롯을 배정해 절반씩 다른 공유 틱에 스냅샷을 전달합니다. |
| 입력 → 처리 → 결과 | 공유 스케줄러 20 Hz 틱 → 모든 실행 가능한 경기방 시뮬레이션 단계 → 경기방 전달 슬롯과 틱 동작 일치 비교 → 해당 슬롯 경기방만 스냅샷 추가 → 종료 상태면 주기와 무관하게 종료/결과 확정 경로 실행입니다. |
| 소유 주체·수명·정리 | GameHub 공유 틱이 시뮬레이션 순번과 전달 슬롯 결정을 소유합니다. `LatestSnapshotBuffer`는 선택된 스냅샷의 클라이언트별 전달만 소유하며 시뮬레이션 상태를 소유하지 않습니다. |
| 실패·되돌리기·재시도 | 이전에는 synchronized 폭주가 콜백 lag와 스냅샷 폐기를 늘릴 수 있었습니다. 수정은 작업을 시간축에 분산하지만 전송 계층 부하가 실제로 높은 클라이언트는 기존 버퍼 프로세스 종료 규칙을 계속 적용합니다. |
| 보장하는 것 | 물리 계산은 20 Hz로 유지되고 정상 스냅샷 전달은 경기방당 10 Hz이며 경기방 간 폭주가 두 슬롯으로 분산됩니다. |
| 보장하지 않는 것 | 10 Hz/두 슬롯은 현재 부하 목표에 맞춘 고정 정책입니다. 전송 콜백 지연 오판은 아직 남아 있으며 `d90f17fa765d`에서 별도로 수정됩니다. |
| 후속 연결 | `db1ae3d47b96`가 여러 경기방 20 Hz/10 Hz 분리를 검증하고 `547d9943d30a`가 부하 테스트 도구 측정 자체를 안정화합니다. |
<!-- LEARNER-END:ad482c200cea:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:ad482c200cea:fix -->
이전 가정: 시뮬레이션 틱마다 모든 경기방 스냅샷을 보내도 된다 → 실제 위험: 공유 틱에 전달 폭주 집중 → 수정: 20 Hz 시뮬레이션과 10 Hz 시간을 분산한 스냅샷 전송 주기 분리 → `db1ae3d47b96` 회귀 테스트.
<!-- LEARNER-END:ad482c200cea:fix -->


#### 비교 기준

- 직전 관련 SHA: `69fb44d2f0ca` — `test(game): shared scheduler lifecycle 검증`
- 다음 관련 SHA: `db1ae3d47b96` — `test(load): 기본 부하 병목 구간 검증`

### 5.12. `test(load): 기본 부하 병목 구간 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `db1ae3d47b96` |
| 중요도 | B |
| 태그 | SIMULATION, REALTIME, OPERATIONS |
| 원문에서 확인한 역할 | 여러 경기방에서 20 Hz 시뮬레이션과 시간을 분산한 10 Hz 스냅샷 전달이 분리되는지 결정적 GameHub 테스트로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.snapshotCadence.test.ts`, `apps/api/src/gameHub.reconnect.test.ts`, `apps/api/src/observability.finalization.test.ts`, `tests/load/*` 관련 조정
- 핵심 심벌: AI 경기방 2개의 가상 타이머 시나리오, 스냅샷 개수, 결과 확정 관측기 검증
- 두 진행 중인 경기방을 만들고 네 시뮬레이션 틱 동안 각 경기방이 두 스냅샷만 받는지 확인합니다.
- 두 경기방의 전달 슬롯이 번갈아 실행돼 한 공유 틱에 한 경기방 전달만 발생하는지 확인합니다.
- 시뮬레이션 상태는 네 번 advance하면서 스냅샷 개수만 두 번인지 구분합니다.
- 결과 확정 관측기가 `created` 여부와 중복 처리 결과를 실제 운영 콜백에서 어떻게 기록하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:db1ae3d47b96:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 주기 수정은 구현됐지만 시뮬레이션 횟수와 전달 횟수를 분리해 검증하는 여러 경기방 회귀가 없었습니다. 단순 스냅샷 개수만 보면 시뮬레이션까지 10 Hz로 낮아진 버그를 놓칠 수 있고, 한 경기방 테스트만으로는 슬롯 시점을 분산하는 처리를 검증할 수 없습니다. |
| 구현 또는 검증 결정 | 가상 타이머로 두 AI 경기방을 동시에 구동해 시뮬레이션 틱과 소켓 스냅샷 메시지 본문을 별도로 기록합니다. 경기방마다 4틱과 스냅샷 2개와 번갈아 적용하는 집계 전달을 검사합니다. |
| 실행/검증 경로 | 두 경기방 준비 완료 → 가상 시간으로 4×50ms 진행 → 시뮬레이션 순번/상태 진행 상태 확인 → 경기방별 스냅샷 2개와 틱별 분산 확인 → 정리/결과 확정 관측기 확인입니다. |
| 소유권과 실패 처리 | 테스트가 가상 시간과 테스트용 소켓을 소유하고 GameHub 공유 스케줄러, 경기방 슬롯, 버퍼가 실제 코드 경로를 실행합니다. 전달 폭주 회귀, 시뮬레이션 주기 저하, 중복 결과 확정 관찰을 서로 다른 검증으로 감지합니다. |
| 보장하는 것 | 20Hz 서버 기준 상태 갱신과 10Hz로 시간차를 둔 스냅샷 전송의 분리가 여러 경기방 상황에서 고정됩니다. |
| 보장하지 않는 것 | 실제 500개 연결의 k6 처리량이나 OS 이벤트 루프 p95는 실행하지 않습니다. |
| 후속 연결 | `ad482c200cea`의 수정 회귀이며 `547d9943d30a`는 외부 부하 시나리오의 재연결/결과 확정 측정을 추가 안정화합니다. |
<!-- LEARNER-END:db1ae3d47b96:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:db1ae3d47b96:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 여러 경기방 부하 경계 테스트 |
| 주입·재현 방식 | 가상 타이머와 두 AI 경기방, 테스트용 소켓으로 시뮬레이션 틱과 생성된 스냅샷을 따로 집계합니다. |
| 검증하는 것 | 4 시뮬레이션 틱 동안 경기방별 2 스냅샷과 번갈아 적용하는 전달 슬롯을 검증합니다. |
| 검증하지 않는 것 | 실제 운영 부하 수치, 네트워크 역압, PostgreSQL 성능은 검증하지 않습니다. |
<!-- LEARNER-END:db1ae3d47b96:test -->



#### 비교 기준

- 직전 관련 SHA: `ad482c200cea` — `fix(game): 부하 중 snapshot cadence 안정화`
- 다음 관련 SHA: `d90f17fa765d` — `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`

### 5.13. `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`

| 항목 | 값 |
| --- | --- |
| SHA | `d90f17fa765d` |
| 중요도 | A |
| 태그 | REALTIME, PERF, RISK |
| 원문에서 확인한 역할 | 완료되지 않은 WebSocket 전송 콜백을 혼잡 신호에서 제거하고 실제 `bufferedAmount`만 전송 계층 부하로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.ts`
- 핵심 심벌: `LatestSnapshotBuffer.flush`, 제거된 `sending` 검사, 전송 콜백의 후속 처리
- 부모 커밋에서 `sending`이 전달 완료/재시도/혼잡 경과 시간에 어떤 영향을 주는지 비교합니다.
- 수정 후 완료되지 않은 콜백이 있어도 `bufferedAmount`가 낮으면 후속 최신 스냅샷을 보낼 수 있는지 확인합니다.
- 경미한/강한 임계값과 혼잡 기한은 그대로 `bufferedAmount`에만 적용되는지 확인합니다.
- 여러 콜백 완료 순서가 대기 중 스냅샷 정리와 종료 상태를 깨지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:d90f17fa765d:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 최초 버퍼는 전송 콜백이 아직 호출되지 않은 상태를 소켓 버퍼 부하와 동일하게 취급해 재시도/혼잡 타이머를 시작했습니다. |
| 해결하려던 문제와 위험 | `ws.send` 콜백 지연은 이벤트 루프 콜백 스케줄링이나 library 완료 시점일 수 있으며 `bufferedAmount`가 낮은 정상 전송 계층에서도 발생합니다. 이를 혼잡으로 보면 스냅샷을 불필요하게 폐기하고 5초 뒤 정상 상태의 연결을 강제 종료할 수 있습니다. |
| 핵심 구현 결정 | `sending`을 부하 검사에서 제거하고 추가 전송 판단을 `socket.bufferedAmount`와 연결 상태에만 의존하도록 바꿉니다. 콜백은 오류 보고와 후속 추가 전송 조건 역할만 유지합니다. |
| 입력 → 처리 → 결과 | 스냅샷 추가 → `bufferedAmount` 강제/완화된 검사 → 낮으면 완료되지 않은 콜백 여부와 무관하게 전송 → 콜백 완료 시 오류 처리·대기 중 추가 전송 재시도; 부하 타이머는 실제 버퍼에 쌓인 바이트가 완화된 상한 이상일 때만 시작합니다. |
| 소유 주체·수명·정리 | 버퍼는 대기 중 최신 값과 재시도/혼잡 상태를 계속 소유하지만 전송 콜백 개수 자체를 exclusive 전송 잠금으로 소유하지 않습니다. 전송 계층이 콜백 완료를 비동기로 통지합니다. |
| 실패·되돌리기·재시도 | 실제 버퍼에 쌓인 부하는 기존 완화된 재시도/강제 종료/5초 기한으로 계속 제한됩니다. 지연된 콜백만으로는 폐기 사유나 프로세스 종료가 발생하지 않습니다. |
| 보장하는 것 | 콜백 지연 시간과 대기 중인 바이트 혼잡을 구분해 정상 연결을 오판하지 않으면서 실제 외부 전송 메모리 상한은 유지합니다. |
| 보장하지 않는 것 | 전송 계층 library가 `bufferedAmount`를 정확히 반영한다는 전제는 남습니다. 프레임 하나의 크기 상한은 아직 플러그인 계층에 강제되지 않습니다. |
| 후속 연결 | `5cd54767858f`가 지연된 콜백과 실제 부하를 분리해 검증하고 `8ea18a1b92db`가 프레임 크기 종속 상한을 추가합니다. |
<!-- LEARNER-END:d90f17fa765d:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:d90f17fa765d:fix -->
이전 가정: 콜백 미완료 == 혼잡 → 실제 실패: 낮은 bufferedAmount에서도 콜백 지연으로 폐기/강제 종료 → 근본 원인: 완료 신호와 대기 중인 바이트 혼동 → 수정: `bufferedAmount`만 부하 소스 → 회귀 `5cd54767858f`.
<!-- LEARNER-END:d90f17fa765d:fix -->


#### 비교 기준

- 직전 관련 SHA: `db1ae3d47b96` — `test(load): 기본 부하 병목 구간 검증`
- 다음 관련 SHA: `5cd54767858f` — `test(game): callback 지연과 실제 congestion 구분`

### 5.14. `test(game): callback 지연과 실제 congestion 구분`

| 항목 | 값 |
| --- | --- |
| SHA | `5cd54767858f` |
| 중요도 | A |
| 태그 | REALTIME, PERF, TEST |
| 원문에서 확인한 역할 | 지연된 전송 콜백은 정상 전달로, 높은 `bufferedAmount`는 실제 혼잡으로 처리되는지 분리해 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/game/latestSnapshotBuffer.test.ts`
- 핵심 심벌: 콜백 완료 시점을 수동으로 제어하는 테스트용 소켓, `bufferedAmount` 부하 사례, 전달·폐기·강제 종료 감시 객체
- `bufferedAmount=0`인 채 여러 전송 콜백을 지연하고 스냅샷이 폐기/강제 종료 없이 전송되는지 확인합니다.
- 콜백을 순서와 다르게 완료해도 버퍼 상태가 깨지지 않는지 확인합니다.
- 완화된 부하에서는 최신 하나 값만 유지되고 부하 해제 뒤 전달되는 기존 동작 의미가 남는지 확인합니다.
- 강제/시간 혼잡 테스트가 콜백 지연 사례와 명확히 분리되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:5cd54767858f:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 기존 `125aa113a01c` 테스트는 지연된 콜백을 부하로 취급하는 구현 기대값을 포함했습니다. |
| 해결하려던 문제와 위험 | 수정이 콜백 검사만 제거하고 true 역압 protections까지 약화시키거나 동시 콜백 완료에서 오래된 대기 중을 전송할 위험이 있었습니다. |
| 핵심 구현 결정 | 테스트용 소켓에서 콜백을 보류하되 버퍼에 쌓인 바이트는 0으로 유지하는 사례와, 버퍼에 쌓인 바이트를 경미한/강한 상한으로 올리는 사례를 분리합니다. 관측기와 프로세스 종료 호출을 각각 확인합니다. |
| 입력 → 처리 → 결과 | 낮은 버퍼 사용량에서 콜백을 지연한 채 여러 스냅샷을 추가하면 전송은 계속되고 혼잡 상태가 되지 않습니다. 콜백을 완료하면 내부 상태가 정리됩니다. 별도의 높은 버퍼 사용량 사례에서는 최신 스냅샷 교체와 재시도를 거쳐 부하가 해제되는지, 또는 강제 종료 임계값에 도달하는지를 확인합니다. |
| 소유 주체·수명·정리 | 테스트가 콜백 완료와 `bufferedAmount`를 독립 제어해 두 신호의 소유권을 분리합니다. 버퍼 정리는 종료/부하 경로에서 검증됩니다. |
| 실패·되돌리기·재시도 | 콜백 지연을 혼잡으로 잘못 판단하던 기존 동작을 직접 재현하고, 실제 완화 단계와 강제 종료 단계의 부하 회귀도 함께 보호합니다. |
| 보장하는 것 | 콜백 지연은 혼잡이 아니며 대기 중인 바이트 부하만 폐기/재시도/강제 종료를 유발한다는 수정된 불변 조건을 검증합니다. |
| 보장하지 않는 것 | 실제 네트워크 stack의 콜백과 bufferedAmount 상관관계는 테스트용 소켓 밖에서 측정하지 않습니다. |
| 후속 연결 | `d90f17fa765d`의 루트 원인 수정을 보호하고 최초 `125aa113a01c`의 잘못된 가정을 의도적으로 교정합니다. |
<!-- LEARNER-END:5cd54767858f:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:5cd54767858f:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 회귀/실패 discrimination 테스트 |
| 주입·재현 방식 | 전송 콜백 완료와 `bufferedAmount`를 독립적으로 제어하는 테스트용 소켓을 사용합니다. |
| 검증하는 것 | 콜백만 지연된 경우에는 메시지를 폐기하거나 연결을 종료하지 않고, 실제 혼잡에서는 최신 값만 유지하거나 연결을 종료하는지 구분해 검증합니다. |
| 검증하지 않는 것 | 실제 kernel 버퍼 동작이나 WAN 지연 시간에서의 throughput은 검증하지 않습니다. |
<!-- LEARNER-END:5cd54767858f:test -->

#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:5cd54767858f:fix -->
거짓 성공을 재현하는 회귀 테스트이며, 수정 전 테스트 기대 조건과 달리 낮은 버퍼 사용량 콜백 지연을 정상으로 고정합니다.
<!-- LEARNER-END:5cd54767858f:fix -->


#### 비교 기준

- 직전 관련 SHA: `d90f17fa765d` — `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음`
- 다음 관련 SHA: `8ea18a1b92db` — `fix(realtime): WebSocket transport payload 상한 설정`

### 5.15. `fix(realtime): WebSocket transport payload 상한 설정`

| 항목 | 값 |
| --- | --- |
| SHA | `8ea18a1b92db` |
| 중요도 | A |
| 태그 | AUTH, REALTIME, RISK |
| 원문에서 확인한 역할 | 애플리케이션 인증 전 상한과 동일한 8 KiB를 내부 `ws` 서버의 `maxPayload`에 설정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/ws-ticket.test.ts`
- 핵심 심벌: WebSocket 플러그인 옵션의 `maxPayload`, 기존 인증 전 메시지 본문 constant
- 부모 커밋에서 애플리케이션 메시지 버퍼가 8 KiB를 검사해도 전송 계층이 프레임 전체를 먼저 수신하는지 확인합니다.
- 플러그인 등록이 기존 인증 전 상수와 동일한 `maxPayload`를 사용하는지 확인합니다.
- 상한 초과 프레임이 애플리케이션 JSON 파서나 인증 버퍼에 도달하기 전에 `ws` 종료 1009로 종료되는지 확인합니다.
- 본문/바이너리 프레임 모두 전송 상한 적용 범위에 포함되는지 library 경계를 실제 설정으로 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:8ea18a1b92db:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 인증 전 메시지 처리 함수는 누적 메시지 본문 8 KiB를 제한했지만 내부 WebSocket 서버는 더 큰 프레임을 먼저 메모리에 받아 애플리케이션 콜백에 전달할 수 있었습니다. |
| 해결하려던 문제와 위험 | 애플리케이션 수준의 길이 확인만으로는 프레임 메모리 할당과 파서 진입 전 자원 소비를 막지 못합니다. 인증 전 공격자가 상한을 초과한 프레임을 반복할 수 있었습니다. |
| 핵심 구현 결정 | Fastify WebSocket/`ws` 서버 등록에 `maxPayload`를 기존 8 KiB 상수로 설정해 전송 계층 파서 자체가 상한을 초과한 프레임을 거부하도록 합니다. |
| 입력 → 처리 → 결과 | 클라이언트 프레임 수신 → `ws` 파서가 프레임 크기와 8 KiB maxPayload 비교 → 초과면 애플리케이션 처리 함수 호출 없이 프로토콜 종료 1009 → 허용 크기만 인증 전/인증된 메시지 경로로 전달됩니다. |
| 소유 주체·수명·정리 | 프레임 크기 강제 검사는 애플리케이션 버퍼가 아니라 전송 계층 서버가 소유합니다. 애플리케이션은 여전히 인증 전 메시지 수·누적 상태와 스키마 검증을 소유합니다. |
| 실패·되돌리기·재시도 | 8,193-바이트 프레임은 전송 계층 종료로 종료돼 JSON 파싱, 티켓 조회, GameHub 연결에 도달하지 않습니다. |
| 보장하는 것 | 단일 WebSocket 프레임이 8 KiB를 넘으면 allocation 이후 애플리케이션 처리로 확대되지 않고 프로토콜 수준 실패를 받습니다. |
| 보장하지 않는 것 | 허용 크기 이하의 다수 프레임, decompression 설정, 연결 수 자체는 다른 상한이 소유합니다. |
| 후속 연결 | `1afec49052b6`가 실제 인증된 WebSocket으로 8,193-바이트 종료 1009를 검증합니다. |
<!-- LEARNER-END:8ea18a1b92db:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:8ea18a1b92db:fix -->
이전 가정: 애플리케이션 메시지 본문 확인면 충분 → 실제 위험: 전송 계층이 상한을 초과한 프레임을 먼저 수신 → 수정: `ws.maxPayload=8 KiB` → 실제 소켓 회귀 `1afec49052b6`.
<!-- LEARNER-END:8ea18a1b92db:fix -->


#### 비교 기준

- 직전 관련 SHA: `5cd54767858f` — `test(game): callback 지연과 실제 congestion 구분`
- 다음 관련 SHA: `1afec49052b6` — `test(realtime): oversized WebSocket frame 거부 검증`

### 5.16. `test(realtime): oversized WebSocket frame 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `1afec49052b6` |
| 중요도 | B |
| 태그 | AUTH, REALTIME, TEST |
| 원문에서 확인한 역할 | 실제 서버/소켓 경로에서 8,193-바이트 프레임이 종료 코드 1009로 거부되는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/ws-ticket.test.ts`
- 핵심 심벌: 서버 시작, 인증된 WebSocket 티켓 실행 순서, 상한을 초과한 프레임 전송, 종료 이벤트 검증
- 메모리 저장소와 실제 listening 서버를 띄우고 인증/티켓을 거쳐 WebSocket을 여는 설정을 확인합니다.
- 정확히 8,193-바이트 메시지 본문을 전송하고 종료 코드 1009를 기다리는지 확인합니다.
- 애플리케이션 오류 이벤트가 아니라 전송 계층 종료를 관측하며 정리에서 소켓/애플리케이션/저장소를 닫는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1afec49052b6:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | `maxPayload` 설정은 있었지만 Fastify 플러그인과 실제 `ws` 서버까지 전달되는지 단위 검토만으로 확정하기 어려웠습니다. 프레임워크 옵션 연결이 잘못되면 설정이 존재해도 실제 전송 계층은 기본값 상한을 사용할 수 있습니다. |
| 구현 또는 검증 결정 | 실제 로컬 WebSocket 연결을 인증한 뒤 상한보다 1 바이트 큰 프레임을 보내 프로토콜 종료 이벤트를 검사합니다. |
| 실행/검증 경로 | 애플리케이션 포트 열기 → HTTP 인증/티켓 → WebSocket 업그레이드 → 8,193-바이트 전송 → `ws` 파서 실패 → 클라이언트 종료 코드 1009 → 자원 종료입니다. |
| 소유권과 실패 처리 | 테스트가 실제 프로세스 내부 서버와 소켓 수명을 소유하고 후속 정리에서 열린 핸들을 닫습니다. 전송 계층 파서의 메시지 크기 초과 경로를 실제 프레임으로 재현합니다. |
| 보장하는 것 | 8 KiB 설정이 프레임워크에서 실제 WebSocket 전송 계층까지 적용됨을 검증합니다. |
| 보장하지 않는 것 | 8,192-바이트 경계 허용, 압축 프레임 해제 시 크기 증폭, 분산 진입 지점의 전체 상한은 이 테스트가 검증하지 않습니다. |
| 후속 연결 | `8ea18a1b92db`의 전송 계층 수정을 직접 보호합니다. |
<!-- LEARNER-END:1afec49052b6:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:1afec49052b6:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 실제 소켓 통합 회귀 테스트 |
| 주입·재현 방식 | 인증된 실제 WebSocket에 8,193-바이트 프레임을 보내 종료 코드 1009를 관찰합니다. |
| 검증하는 것 | 애플리케이션 처리 함수 이전 전송 계층 크기 실패가 실제 연결에서 동작함을 검증합니다. |
| 검증하지 않는 것 | 모든 프레임 fragmentation/compression 조합이나 연결 빈도 defense는 검증하지 않습니다. |
<!-- LEARNER-END:1afec49052b6:test -->



#### 비교 기준

- 직전 관련 SHA: `8ea18a1b92db` — `fix(realtime): WebSocket transport payload 상한 설정`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:evolution -->
`a6a1f4fba60e`~`49ca3e778801`은 경기방/클라이언트에 고정 간격 단계, 연결 확인 신호/입력 제한기, 스냅샷 버퍼를 실제 연결합니다. `aed88c8a93e0`/`8d24b5e70837`이 타이머 구성 비교 경계를 만든 뒤 `d21a47ee92d2`/`fb5b1abc97f5`가 하나 시계 소유권을 도입·이전하고 `69fb44d2f0ca`가 복구 전이를 고정합니다. `ad482c200cea`/`db1ae3d47b96`은 20 Hz 시뮬레이션과 시간을 분산한 10 Hz 전달을 분리합니다. `d90f17fa765d`/`5cd54767858f`는 콜백 지연 오판을 교정하고, `8ea18a1b92db`/`1afec49052b6`는 8 KiB 프레임 종속 상한을 전송 계층에서 강제합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:failure-links -->
- 기본 구성 요소 미통합과 정리 누락 → GameHub 수명주기 연결 → 실행 시점 통합 테스트
- 경기방별 타이머 증식 → 제어되는 벤치마크 실행 지점 → 공유 스케줄러 추상화/소유권 이전 → 재연결 수명주기 테스트
- synchronized 스냅샷 폭주 → 20 Hz 시뮬레이션/10 Hz 시간을 분산한 전달 → 여러 경기방 주기 회귀
- 전송 콜백 false 혼잡 → `bufferedAmount`-전용 수정 → 지연된 콜백 대비 실제 부하 회귀
- 애플리케이션 전용 크기 확인 → 전송 계층 `maxPayload` → 실제 소켓 1009 회귀
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:ownership -->
초기에는 경기방이 타이머를 소유했지만 최종적으로 GameHub의 `SharedRoomScheduler`가 단 하나의 시계를 소유하고 경기방은 실행 가능한 콜백 소속 정보만 가집니다. 클라이언트 레코드는 연결 확인 신호와 최신 버퍼를 소유하고, InputGate는 사용자 수준 허용 시간을 소유합니다. `ws` 서버는 프레임 크기 강제 검사를, 버퍼는 대기 중인 바이트 부하와 최신 메시지 본문을 소유합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-state -->
GameHub는 한 공유 고정 간격 단계 시계로 실행 가능한 경기방을 20 Hz 진행하고 스냅샷만 10 Hz 두 슬롯 주기로 전달합니다. 클라이언트별 연결 확인 신호/입력/스냅샷 상한이 수명주기에 연결되며 콜백 지연은 혼잡으로 취급되지 않습니다. 8 KiB 초과 프레임은 애플리케이션 처리 함수 전에 전송 계층이 거부합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-flow -->
인증된 연결이 연결 확인 신호와 스냅샷 버퍼를 만들고 사용자 수준 입력 제한기를 공유합니다. 경기방이 경기 중이 되면 공유 스케줄러에 등록되고 매 50 ms 시뮬레이션이 진행됩니다. 틱 동작 일치에 맞는 경기방만 스냅샷을 최신 버퍼에 추가합니다. 대기 중인 바이트가 실제 임계값을 넘을 때만 재시도/강제 종료하며 상한을 초과한 프레임은 `ws` 파서에서 1009로 종료됩니다. 일시정지·경기 종료·연결 종료 시 스케줄러 등록 정보와 클라이언트 자원을 정리합니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:checks -->
- [x] 경기방별 타이머에서 공유 타이머로 소유권이 이동한 전후를 설명할 수 있습니다.
- [x] 준비 완료/일시정지/재연결/종료마다 스케줄러 소속 정보가 어떻게 변하는지 추적할 수 있습니다.
- [x] 시뮬레이션 주기와 스냅샷 frequency가 왜 분리됐는지 설명할 수 있습니다.
- [x] 콜백 지연, 버퍼에 쌓인 바이트, 프레임 크기를 서로 다른 부하 신호로 구분할 수 있습니다.
- [x] 각 수정을 해당 회귀 테스트와 연결할 수 있습니다.
<!-- LEARNER-END:04-gamehub-runtime-integration-shared-scheduling-and-congestion.md:checks -->
===== END FILE: 04-gamehub-runtime-integration-shared-scheduling-and-congestion.md =====

===== BEGIN FILE: 05-draining-readiness-and-graceful-shutdown.md =====
# 신규 작업 수락 중단과 단계적 종료

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

새 작업 수락을 즉시 중단하고 진행 중인 경기방이 정해진 시간 안에 끝나기를 기다린 뒤 프로세스 자원을 한 번만 정리하며, 외부 컨테이너 종료 허용 시간까지 애플리케이션 종료 준비와 정렬하는 종료 수명주기를 복원합니다.

범위 메모: Compose 설정을 참조하지만 일반적인 이미지/릴리스 전달이 아니라 애플리케이션 종료 준비 보장의 외부 상한이므로 이 카테고리에 포함합니다. 운영 산출물 구성 자체는 카테고리 09에 남습니다.

### 직접 관련된 불변 조건

- 종료 준비를 시작하는 즉시 준비 상태가 준비되지 않음으로 바뀌고 새 대기열, 토너먼트, AI 작업을 거부합니다.
- 기존 진행 중인 경기방은 최대 60초 동안 완료할 수 있지만 종료는 무기한 기다리지 않습니다.
- SIGTERM 또는 SIGINT를 반복해서 받아도 하나의 종료 준비와 종료 절차만 시작합니다.
- 컨테이너 종료 유예 시간은 애플리케이션의 60초 작업 종료 대기 시간보다 짧지 않습니다.

## 2. 핵심 질문

- 준비 상태 수명주기와 GameHub 참가 상태는 어떤 호출에서 동시에 종료 준비 중으로 전이됩니까?
- 대기 중 작업과 진행 중인 경기방은 종료 준비에서 왜 다르게 처리됩니까?
- 신호 중복·종료 준비 시간 초과·종료 실패는 어떤 결과/종료 상태로 수렴합니까?
- 애플리케이션 시간 초과와 Compose 종료 유예 시간의 소유권 관계는 어떻게 검증됩니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `44ef3e07e1a5` | `feat(game): 새 작업 차단과 active room drain 추가` | A | PROTOCOL, REALTIME, TOURNAMENT | GameHub와 준비 상태가 공유하는 종료 준비 중 상태를 도입해 새 대전 상대 찾기를 즉시 거부하고 진행 중인 경기방이 정해진 시간 안에 끝나기를 기다립니다. |
| 2 | `1c9981393973` | `feat(ops): graceful shutdown 절차 추가` | A | REALTIME, PERSISTENCE, OPERATIONS | SIGTERM/SIGINT를 한 번만 실행되는 종료 준비→애플리케이션 종료 절차로 연결하고 종료 오류를 프로세스 종료 상태에 반영합니다. |
| 3 | `9d05f47e7f4b` | `test(ops): GameHub drain과 graceful shutdown 검증` | A | REALTIME, OPERATIONS, RISK | 종료 준비 중 참가 거부, 활성 경기방 시간 초과, 준비 상태 전이, 반복 신호에서도 한 번만 실행되는 종료을 가상 시간과 주입한 신호 소스로 검증합니다. |
| 4 | `312ddbc6fbe2` | `fix(runtime): container 종료 유예를 room drain과 정렬` | A | REALTIME, OPERATIONS, RISK | API 컨테이너의 `stop_grace_period`를 70초로 설정해 애플리케이션의 60초 경기방 작업 종료 허용 시간보다 길게 만듭니다. |
| 5 | `73ba979841cd` | `test(docker): API 종료 유예 계약 검증` | B | OPERATIONS, TEST | 운영 Compose 실행 시간을 파싱해 API 종료 유예 시간이 애플리케이션 60초 작업 종료 대기 시간 이상인지 검증합니다. |

## 5. 커밋별 학습 기록

### 5.1. `feat(game): 새 작업 차단과 active room drain 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `44ef3e07e1a5` |
| 중요도 | A |
| 태그 | PROTOCOL, REALTIME, TOURNAMENT |
| 원문에서 확인한 역할 | GameHub와 준비 상태가 공유하는 종료 준비 중 상태를 도입해 새 대전 상대 찾기를 즉시 거부하고 진행 중인 경기방이 정해진 시간 안에 끝나기를 기다립니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/app.ts`, `apps/api/src/gameHub.ts`, `packages/shared/src/ws.ts`
- 핵심 심벌: `app.beginDrain`, `GameHub.beginDrain`, `acceptingMatches`, 종료 준비 완료를 기다리는 호출자/타이머, `server_draining` 오류
- `app.beginDrain`이 수명주기 상태 값을 먼저 `draining`으로 바꿔 `/health/ready`를 즉시 503으로 만드는지 확인합니다.
- `GameHub.beginDrain(timeoutMs)`가 대기열, AI 대체 처리 타이머, 토너먼트 대기 중 상태를 비우고 새 대기열 명령을 `server_draining`으로 거부하는지 추적합니다.
- 이미 활성인 경기방은 즉시 제거하지 않고 경기방 개수가 0이 되거나 시간 초과가 만료될 때 Promise를 판별하는지 확인합니다.
- 반복 `beginDrain` 호출이 같은 Promise/상태 전이를 재사용하고 시간 초과 핸들이 `unref`/정리되는지 확인합니다.
- `GameHub.close`가 공유 스케줄러, 재연결 타이머, 비회원 경기 결과 보존 기간, 연결 확인 신호, 스냅샷 버퍼, 소켓을 어떤 순서로 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:44ef3e07e1a5:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 서비스는 프로세스 신호나 배포 종료가 시작돼도 준비 상태가 계속 준비 완료였고, 새 대기열/토너먼트/AI 작업을 받으면서 기존 경기방과 함께 종료해야 했습니다. |
| 해결하려던 문제와 위험 | 새 작업 참가를 멈추지 않으면 진행 중인 경기방 수가 종료 준비 중에도 늘어나 종료가 수렴하지 않습니다. 반대로 모든 소켓을 즉시 닫으면 진행 중 경기의 서버가 확정한 결과와 영속 저장 기회를 잃습니다. |
| 핵심 구현 결정 | 애플리케이션과 GameHub에 명시적 종료 준비 중 상태를 추가합니다. 애플리케이션은 종료 준비 시작 즉시 준비 상태 수명주기를 `draining`으로 노출하고, GameHub는 대전 상대 찾기 참가를 닫아 대기 중 작업을 정리하되 진행 중인 경기방은 시간 초과 허용 시간 안에서 종료하도록 둡니다. 공유 프로토콜에 안정적인 `server_draining` 오류가 추가됩니다. |
| 입력 → 처리 → 결과 | `app.beginDrain(60_000)` → 수명주기=`draining` → `hub.beginDrain` → 대기열/타이머/대기 중인 참가자 해제·새 작업 거부 → 진행 중인 경기방 제거 때 종료 준비 완료를 기다리는 호출자 재평가 → 경기방=0이면 `{drained:true}` 또는 시간 초과이면 `{drained:false, activeRooms:n}` 판별 → 이후 애플리케이션 종료입니다. |
| 소유 주체·수명·정리 | 애플리케이션은 공개 준비 상태 수명주기를, GameHub는 대기열/경기방 참가와 종료 준비 Promise/타이머를 소유합니다. 기존 경기방은 정상 RoomSession/GameHub 소유 주체를 유지하며 종료 정리가 종료 준비 완료를 기다리는 호출자를 알립니다. |
| 실패·되돌리기·재시도 | 진행 중인 경기방이 종료하지 않아도 시간 초과가 상한을 보장합니다. 종료 준비를 반복 호출해도 새 타이머를 만들지 않고, 새 대기열 시도는 명시적 프로토콜 오류를 받습니다. 시간 초과 뒤 남은 경기방 소켓의 최종 처리는 후속 애플리케이션 종료가 담당합니다. |
| 보장하는 것 | 종료 준비 시작과 동시에 새 작업이 차단되고 준비 상태가 내려가며, 기존 진행 중인 경기방은 정해진 유예 시간 안에서 정상 완료할 기회를 갖습니다. |
| 보장하지 않는 것 | 이 SHA만으로 SIGTERM/SIGINT가 종료 준비를 호출하지 않으며 컨테이너가 60초보다 일찍 종료하면 허용 시간을 보장하지 못합니다. |
| 후속 연결 | `1c9981393973`이 프로세스 신호를 종료 준비→종료에 연결하고 `9d05f47e7f4b`가 참가/시간 초과/준비 상태 전이를 검증합니다. `312ddbc6fbe2`가 컨테이너 허용 시간을 정렬합니다. |
<!-- LEARNER-END:44ef3e07e1a5:record -->




#### 비교 기준

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`

### 5.2. `feat(ops): graceful shutdown 절차 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1c9981393973` |
| 중요도 | A |
| 태그 | REALTIME, PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | SIGTERM/SIGINT를 한 번만 실행되는 종료 준비→애플리케이션 종료 절차로 연결하고 종료 오류를 프로세스 종료 상태에 반영합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gracefulShutdown.ts`, `apps/api/src/index.ts`
- 핵심 심벌: `installGracefulShutdown`, 중복 실행을 막는 종료 콜백, 신호 리스너 dispose, `app.beginDrain(60_000)`
- `installGracefulShutdown`이 SIGTERM/SIGINT 리스너를 설치하고 첫 신호의 종료 Promise만 실행하는 보호 조건을 확인합니다.
- 진입점 종료 콜백이 60초 종료 준비를 기다린 뒤 `app.close()`를 호출하는 순서를 확인합니다.
- 종료 준비 또는 종료 실패 시 로깅/`process.exitCode=1`과 가능한 범위의 종료가 어떻게 수행되는지 확인합니다.
- 애플리케이션 종료 훅 또는 정리 함수가 신호 리스너를 제거해 테스트와 내장형 수명주기에서 중복 리스너를 남기지 않는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:1c9981393973:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 종료 준비 API는 수동 호출할 수 있었지만 프로세스 신호는 기본값 Node 종료 동작에 맡겨져 배포 프로세스 종료와 애플리케이션 수명주기가 연결되지 않았습니다. |
| 해결하려던 문제와 위험 | 신호마다 별도 종료를 시작하면 애플리케이션.종료, 저장소.종료, 소켓 정리가 중복 실행될 수 있습니다. 오류에서 즉시 `process.exit()`하면 정리가 중단되고, 오류를 무시하면 성공 종료로 오인됩니다. |
| 핵심 구현 결정 | 신호 소스와 종료 콜백을 주입받는 `installGracefulShutdown`을 추가합니다. 첫 신호만 비동기 처리 종료를 시작하고 후속 신호는 무시합니다. 진입점은 GameHub 작업 종료 대기 시간을 기다린 뒤 Fastify를 닫고, 실패를 로그와 `exitCode = 1`로 기록합니다. |
| 입력 → 처리 → 결과 | SIGTERM 또는 SIGINT → 중복 실행 방지 → `app.beginDrain(60_000)` → 종료 준비 결과 기록 → `app.close()` → Fastify `onClose`에서 GameHub와 저장소 자원 정리 → Promise가 실패하면 오류 기록·`exitCode = 1` 설정·가능한 범위의 정리를 수행합니다. |
| 소유 주체·수명·정리 | 신호 리스너와 중복 실행 방지 상태은 설치 함수가 소유하며 정리 함수가 해제합니다. 진입점 종료 콜백은 애플리케이션과 종료 준비 순서를, Fastify 훅은 하위 저장소와 GameHub 정리를 소유합니다. |
| 실패·되돌리기·재시도 | 종료 준비 시간 초과는 정상 결과로 보고 종료를 계속합니다. 종료 준비/종료 실패는 신호 처리 함수에서 잡지 못한 예외를 발생시켜 퍼뜨리지 않고 오류 콜백 및 0이 아닌 `exitCode`에 기록합니다. 반복 신호는 두 번째 정리를 시작하지 않습니다. |
| 보장하는 것 | 배포 신호가 상한이 있는 하나의 종료 준비와 종료 절차로 수렴하고, 실패를 성공 종료로 위장하지 않습니다. |
| 보장하지 않는 것 | 외부 실행 조정기가 애플리케이션의 60초 허용 시간보다 긴 종료 유예 시간을 주는지는 이 코드가 통제하지 못합니다. |
| 후속 연결 | `9d05f47e7f4b`가 반복 신호와 실패 콜백을 결정적으로 검증하고, `312ddbc6fbe2`가 Compose 종료 유예 시간을 70초로 늘립니다. |
<!-- LEARNER-END:1c9981393973:record -->




#### 비교 기준

- 직전 관련 SHA: `44ef3e07e1a5` — `feat(game): 새 작업 차단과 active room drain 추가`
- 다음 관련 SHA: `9d05f47e7f4b` — `test(ops): GameHub drain과 graceful shutdown 검증`

### 5.3. `test(ops): GameHub drain과 graceful shutdown 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `9d05f47e7f4b` |
| 중요도 | A |
| 태그 | REALTIME, OPERATIONS, RISK |
| 원문에서 확인한 역할 | 종료 준비 중 참가 거부, 활성 경기방 시간 초과, 준비 상태 전이, 반복 신호에서도 한 번만 실행되는 종료을 가상 시간과 주입한 신호 소스로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `apps/api/src/gameHub.drain.test.ts`, `apps/api/src/gracefulShutdown.test.ts`, `apps/api/src/health.test.ts`
- 핵심 심벌: `beginDrain` 사례, `EventEmitter` 신호 소스, 가상 타이머, 준비 상태 inject
- 대기 중 플레이어를 넣은 뒤 종료 준비 시작 즉시 대기 중인 개수가 0이 되고 새 AI/대기열 명령이 `server_draining`을 받는지 확인합니다.
- 경기방이 없을 때 `{drained:true, activeRooms:0}`으로 즉시 판별하는지 확인합니다.
- 활성 AI 경기방에서 59,999ms에는 완료되지 않고, 60,000 ms에 `{drained:false, activeRooms:1}`이 되는 가상 타이머 경계를 확인합니다.
- SIGTERM→SIGINT→SIGTERM을 연속 생성해 종료 콜백이 한 번, 첫 신호 값으로만 호출되는지 확인합니다.
- 종료 준비 직후 `/health/ready`가 503과 `checks.lifecycle='draining'`을 반환하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:9d05f47e7f4b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 종료 준비와 신호 코드는 여러 interacting 상태를 다뤘지만 시간 초과 경계, 새 참가 실패, 반복 신호 동작이 통합 증거로 고정되지 않았습니다. |
| 해결하려던 문제와 위험 | 종료 코드는 정상 경로만 보면 맞아 보이지만 진행 중인 경기방이 남거나 신호가 반복될 때 교착 상태·중복 종료·새 작업 참가가 발생할 수 있습니다. |
| 핵심 구현 결정 | GameHub에는 테스트용 소켓과 저장소, 가상 타이머를 사용하고 단계적 종료에는 주입한 `EventEmitter`와 제어 가능한 Promise를 사용합니다. 상태 확인 라우트는 Fastify `inject`로 종료 준비 직후 응답을 확인합니다. |
| 입력 → 처리 → 결과 | 대기열과 경기방 설정 → `beginDrain` 호출 → 즉시 상태 검증 → 제어 가능한 가상 시계를 기한까지 진행 → 결과 검증 순서입니다. 별도 신호 테스트에서는 신호를 반복 발생시켜 종료 작업이 한 번만 시작되는지, 미리 정한 성공·실패 결과와 `onError`·`dispose` 호출을 확인합니다. |
| 소유 주체·수명·정리 | 테스트가 시간, 신호 기록, 소켓 수명, 저장소 종료를 명시적으로 소유합니다. 운영 GameHub/애플리케이션/설치기 정리 호출이 afterEach에 남는 자원 없이 종료되는지 확인합니다. |
| 실패·되돌리기·재시도 | 진행 중인 경기방 진행 중이 아닌 상태, 반복 신호, 종료 실패를 각각 결정적으로 재현합니다. 원시 프로세스 신호나 외부 SIGKILL은 사용하지 않습니다. |
| 보장하는 것 | 종료 준비가 참가를 즉시 닫고 60초 상한을 지키며 준비 상태를 내리고, 신호가 한 번만 실행되는 종료로 수렴함을 검증합니다. |
| 보장하지 않는 것 | 실제 컨테이너 종료 유예 시간, OS 신호 전달, PostgreSQL 종료 지연 시간은 이 테스트가 검증하지 않습니다. |
| 후속 연결 | `44ef3e07e1a5`/`1c9981393973`의 수명주기 불변 조건을 보호하며 `73ba979841cd`가 외부 Compose 허용 시간을 정적 계약으로 추가합니다. |
<!-- LEARNER-END:9d05f47e7f4b:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:9d05f47e7f4b:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 수명주기 통합/회귀 테스트 |
| 주입·재현 방식 | 가상 타이머, 테스트용 소켓, 메모리 저장소, 주입한 `EventEmitter`, Fastify `inject`를 조합합니다. |
| 검증하는 것 | 대기열 정리와 새 작업 거부, 경기방 없음 성공, 60초 시간 초과, 준비 상태 503, 반복 신호의 한 번 처리, 오류 보고을 검증합니다. |
| 검증하지 않는 것 | 실제 Docker 데몬의 종료 시간 제어나 프로세스 수준 신호 통합은 검증하지 않습니다. |
<!-- LEARNER-END:9d05f47e7f4b:test -->



#### 비교 기준

- 직전 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`
- 다음 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`

### 5.4. `fix(runtime): container 종료 유예를 room drain과 정렬`

| 항목 | 값 |
| --- | --- |
| SHA | `312ddbc6fbe2` |
| 중요도 | A |
| 태그 | REALTIME, OPERATIONS, RISK |
| 원문에서 확인한 역할 | API 컨테이너의 `stop_grace_period`를 70초로 설정해 애플리케이션의 60초 경기방 작업 종료 허용 시간보다 길게 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.yml`
- 핵심 심벌: 운영 Compose `services.api.stop_grace_period`
- 애플리케이션 `beginDrain(60_000)`과 Compose API 서비스의 이전 기본값/누락된 종료 유예 시간을 비교합니다.
- `stop_grace_period: 70s`가 API 서비스에만 적용되고 마이그레이션/웹/db 수명주기와 혼동되지 않는지 확인합니다.
- 실행 조정기가 SIGTERM 뒤 강제 SIGKILL하기 전 애플리케이션 종료 준비+종료에 남기는 10초 여유 시간의 의미를 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:312ddbc6fbe2:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 애플리케이션은 최대 60초 동안 진행 중인 경기방을 기다렸지만 Compose에는 그보다 긴 명시적 종료 유예 시간이 없어 실행 환경이 종료 준비 완료 전 강제 종료될 수 있었습니다. |
| 해결하려던 문제와 위험 | 코드 내부 시간 초과만 길게 설정해도 외부 실행 조정기 종료 허용 시간이 짧으면 보장이 무효입니다. 이는 애플리케이션과 배포 설정 사이의 여러 영역에 걸친 계층 불변 조건였습니다. |
| 핵심 구현 결정 | 운영 Compose의 API 서비스에 `stop_grace_period: 70s`를 추가해 60초 작업 종료 대기 시간보다 10초 긴 종료 유예 시간을 제공합니다. |
| 입력 → 처리 → 결과 | Compose 중지 → API에 SIGTERM → 애플리케이션 최대 60초 종료 준비 → Fastify/GameHub/저장소 종료 → 70초 종료 유예 시간 만료 전 정상 종료; 그때까지 종료하지 못하면 실행 조정기 강제 종료입니다. |
| 소유 주체·수명·정리 | 애플리케이션은 60초 종료 준비와 정리를, Compose 실행 환경은 70초 종료 기한을 소유합니다. 10초 차이는 종료와 로깅에 필요한 부가 작업을 위한 외부 여유입니다. |
| 실패·되돌리기·재시도 | 이전 불일치는 진행 중인 경기방 결과나 저장소 정리가 SIGKILL로 중단될 위험이었습니다. 설정 수정은 종료 기한을 애플리케이션 허용 시간보다 뒤로 이동시킵니다. |
| 보장하는 것 | 지정 Compose 환경에서 API가 애플리케이션 작업 종료 대기 시간 전체를 사용할 최소 프로세스 종료 시간 구간이 생깁니다. |
| 보장하지 않는 것 | 호스트 강제 종료, Kubernetes 등 다른 실행 조정기, 70초를 넘는 저장소 종료는 보장하지 않습니다. |
| 후속 연결 | `73ba979841cd`가 Compose 실행 시간을 파싱해 최소 60초 계약을 회귀 테스트로 고정합니다. |
<!-- LEARNER-END:312ddbc6fbe2:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:312ddbc6fbe2:fix -->
이전 가정: 애플리케이션 60초 시간 초과이면 충분 → 실제 위험: 컨테이너 종료 유예 시간이 더 짧으면 SIGKILL → 수정: Compose 70초 → 정적 계약 `73ba979841cd`.
<!-- LEARNER-END:312ddbc6fbe2:fix -->

#### 최소 코드 근거

<!-- LEARNER-BEGIN:312ddbc6fbe2:snippet -->
- SHA: `312ddbc6fbe2`
- 위치: `docker-compose.yml`; 운영 Compose `services.api.stop_grace_period`

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
| 중요도 | B |
| 태그 | OPERATIONS, TEST |
| 원문에서 확인한 역할 | 운영 Compose 실행 시간을 파싱해 API 종료 유예 시간이 애플리케이션 60초 작업 종료 대기 시간 이상인지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/docker-production.test.mjs`
- 핵심 심벌: `parseDurationSeconds`, `services.api.stop_grace_period` 검증
- Compose를 파싱한 기존 운영 계약 테스트에 `stop_grace_period` 검증이 추가되는 위치를 확인합니다.
- `parseDurationSeconds`가 `XmYs` 형식을 분·초로 변환하고 지원하지 않는 값은 오류로 거부하는지 확인합니다.
- 검증이 특정 70초 문자열이 아니라 최소 60초 관계를 고정하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:73ba979841cd:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | Compose에 70초가 설정됐지만 후속 변경이 이를 제거하거나 60초 미만으로 낮춰도 애플리케이션 테스트는 감지하지 못했습니다. 애플리케이션 시간 초과와 배포 종료 유예 시간의 관계는 서로 다른 파일에 있어 한쪽만 변경될 때 쉽게 깨집니다. |
| 구현 또는 검증 결정 | 운영 Docker 계약 테스트가 파싱한 Compose API 서비스의 실행 시간을 초로 변환하고 `>= 60`을 요구합니다. |
| 실행/검증 경로 | Compose 파일 읽기·파싱 → API 서비스 조회 → 종료 유예 문자열 검증·초 단위 변환 → 애플리케이션 종료 준비 최소 시간과 비교 → 부족하면 계약 테스트를 실패시킵니다. |
| 소유권과 실패 처리 | 정적 계약 테스트가 여러 파일에 걸친 실행 시간 관계를 소유하며 실제 컨테이너를 시작하지 않습니다. 누락·비문자열·지원하지 않는 실행 시간 또는 60초 미만 값을 검증 실패로 만듭니다. |
| 보장하는 것 | 저장소 설정에서 API 종료 유예 시간이 애플리케이션 작업 종료 대기 시간 아래로 회귀하지 않습니다. |
| 보장하지 않는 것 | Docker 데몬이 실행 시간을 실제로 적용하는지, 프로세스가 60초 안에 종료되는지는 실행하지 않습니다. |
| 후속 연결 | `312ddbc6fbe2`의 여러 영역에 걸친 계층 수정을 보호하는 최종 회귀입니다. |
<!-- LEARNER-END:73ba979841cd:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:73ba979841cd:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 정적 배포 계약 테스트 |
| 주입·재현 방식 | Compose YAML을 파싱하고 실행 시간 문자열을 초로 환산해 `>=60`을 검사합니다. |
| 검증하는 것 | checked-in 운영 Compose와 애플리케이션 작업 종료 대기 시간의 최소 관계를 검증합니다. |
| 검증하지 않는 것 | 실제 컨테이너 중지·SIGTERM·SIGKILL 시간 제어는 검증하지 않습니다. |
<!-- LEARNER-END:73ba979841cd:test -->



#### 비교 기준

- 직전 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:evolution -->
`44ef3e07e1a5`는 준비 상태와 GameHub 참가를 공유하는 종료 준비 상태를 만들고 진행 중인 경기방만 정해진 시간까지 대기 대상으로 남깁니다. `1c9981393973`은 SIGTERM/SIGINT를 한 번만 실행되는 60초 종료 준비→종료 순서으로 연결하며, `9d05f47e7f4b`가 대기열·시간 초과·준비 상태·신호 경계를 고정합니다. `312ddbc6fbe2`는 애플리케이션 바깥의 Compose 종료 허용 시간을 70초로 정렬하고 `73ba979841cd`가 그 관계를 정적 계약으로 보호합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:failure-links -->
- 종료 준비 중 새 참가 거부 → `accepting` 상태 값 변경·대기열 정리·`server_draining` 오류 → 테스트용 소켓 회귀
- 진행 중인 경기방 진행 중이 아닌 상태 → 60초 시간 초과 결과 → 59,999/60,000 ms 경계 테스트
- 반복 신호/종료 실패 → 중복 실행 방지 상태·`exitCode`·`onError` → 주입한 신호 테스트
- 외부 종료 유예 시간 < 애플리케이션 시간 초과 → Compose 70초 수정 → 실행 시간 계약 테스트
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:ownership -->
애플리케이션은 준비 상태 수명주기를, GameHub는 대전 상대 찾기 참가 상태와 진행 중인 경기방 종료를 기다리는 타이머와 Promise를, 신호 설치기는 리스너와 중복 실행 방지 상태를, Fastify 종료 훅은 GameHub/저장소 정리를 소유합니다. Compose 실행 환경은 애플리케이션 밖에서 70초 강제 종료 기한을 소유합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:final-state -->
SIGTERM/SIGINT가 오면 준비 상태와 참가가 즉시 닫히고 대기 중 작업이 제거됩니다. 진행 중인 경기방은 최대 60초 완료 기회를 가지며 이후 애플리케이션 종료가 모든 실행 중 자원과 저장소 자원을 정리합니다. 중복 신호는 무시되고 Compose는 70초를 제공하며 정적 테스트가 최소 관계를 보호합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:final-flow -->
신호 → `installGracefulShutdown` 일회성 → `app.beginDrain(60_000)` → 준비 상태 503 + GameHub 새 작업 거부/대기열 해제 → 진행 중인 경기방 개수가 0 또는 시간 초과 → `app.close()` → GameHub 스케줄러/타이머/소켓과 저장소 종료 → 실패 시 `exitCode`를 0이 아닌 값으로 설정. Compose는 최대 70초 뒤 강제 종료합니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:05-draining-readiness-and-graceful-shutdown.md:checks -->
- [x] 종료 준비에서 대기 중 작업과 진행 중인 경기방의 처리 차이를 설명할 수 있습니다.
- [x] 준비 상태 503가 종료 준비 결과를 기다리지 않고 즉시 발생하는 이유를 설명할 수 있습니다.
- [x] 반복 신호가 중복 정리를 만들지 않는 구조를 추적할 수 있습니다.
- [x] 60초 애플리케이션 허용 시간과 70초 컨테이너 종료 유예 시간의 여러 영역에 걸친 계층 불변 조건을 설명할 수 있습니다.
<!-- LEARNER-END:05-draining-readiness-and-graceful-shutdown.md:checks -->
===== END FILE: 05-draining-readiness-and-graceful-shutdown.md =====

===== BEGIN FILE: 06-load-fault-recovery-and-pool-error-containment.md =====
# 부하·장애 복구와 연결 풀 오류 격리

- 카테고리: `07-runtime-observability-and-service-health` — 런타임 관측성과 서비스 상태
- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 1단계 상태: 검토 후 동결된 기준 작업 틀

## 1. 학습 목표

실시간 부하 허용 criteria를 테스트 우선으로 고정하고 k6/Toxiproxy로 연결·경기방·재연결·의존성 장애를 재현하며, 장애가 PostgreSQL 유휴 풀 이벤트로 나타날 때 프로세스 비정상 종료와 인증 정보 유출을 차단하는 과정을 복원합니다.

범위 메모: 부하와 장애 테스트 실행 틀은 검증 아키텍처와도 교차하지만, 여기서는 실행 상태 신호·측정 소스·실패 격리의 구현 과정만 다룹니다. CI 작업 구성이나 릴리스 산출물은 포함하지 않습니다.

### 직접 관련된 불변 조건

- 부하 테스트 도구는 연결/재연결/스냅샷/결과 확정을 제한된 임계값과 명시된 측정 소스로 평가합니다.
- 장애 제어는 루프백 대상만 조작하고 각 실행의 성공·실패와 무관하게 프록시 초기화를 시도합니다.
- 준비 상태는 데이터베이스 중단과 복구를 503/중단→200/up으로 관측하며 주기적 조회에는 최대 대기 시간이 있습니다.
- PostgreSQL 유휴 클라이언트 오류와 오류 보고 콜백 실패는 프로세스 예외를 발생시켜 탈출하지 않고 민감 정보를 제거하고 필드 수를 제한한 메타데이터만 남깁니다.

## 2. 핵심 질문

- 500 연결·50 경기방 프로필이 실제 인증/티켓/경기방/재연결 경로를 어떻게 구성합니까?
- 클라이언트 이벤트와 서버 측 영속 저장 지표 중 결과 확정 근거 소유 주체는 누구입니까?
- Toxiproxy 장애 순번과 준비 상태 예상 상태, 시간 초과, 항상 실행하는 초기화 정리는 어떻게 연결됩니까?
- 풀 `error` EventEmitter, 민감 정보 제거 함수, 초기 로깅 버퍼는 어떤 실패를 각각 격리합니까?

## 3. 완료 기준

- 커밋 목록의 모든 SHA를 `web/ft_transcendence` 커밋 이력에서 확인합니다.
- 각 SHA를 부모 커밋 또는 직전 관련 SHA와 비교해 해당 시점의 상태만 설명합니다.
- 파일, 심벌, 호출자와 피호출자, 상태 변경, 소유권, 정리 과정, 실패 분기를 실제 코드로 기록합니다.
- 수정 커밋은 이전 가정과 근본 원인을 연결하고, 테스트·벤치마크는 실제 코드 경로와 검증 범위·미검증 범위를 구분합니다.
- 실행하지 않은 명령이나 벤치마크 수치를 실행 증거로 기록하지 않습니다.
- 마지막으로 선택한 SHA까지만 사용해 개발 흐름의 최종 소유 주체, 불변 조건, 실행 순서를 정리합니다.

## 4. 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 문서에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ff1bffcd5296` | `test(load): 실시간 부하 임계값 정의` | B | REALTIME, PERSISTENCE, OPERATIONS | 500 연결·50 경기방을 기본으로 하는 k6/Toxiproxy 테스트 실행 틀의 서비스 수준 임계값과 구성 계약을 테스트 우선으로 정의합니다. |
| 2 | `7b0b5f086b41` | `test(load): 실시간 fault injection 도구 추가` | A | AUTH, REALTIME, PERSISTENCE | k6 실시간 부하 시나리오와 PostgreSQL·외부 경계용 Toxiproxy 제어 기능을 구현해 연결, 경기방, 재연결, 스냅샷, 결과 확정, 의존성 실패를 재현합니다. |
| 3 | `547d9943d30a` | `fix(load): 기본 부하 profile 측정 안정화` | A | REALTIME, OPERATIONS, OBSERVABILITY | 재연결 폭주와 클라이언트 측 결과 확정 오판을 제거해 기본 부하 프로필의 측정 소스와 시간 제어를 안정화합니다. |
| 4 | `84bec3bf57ae` | `test(load): fault recovery 검사 자동화` | A | PERSISTENCE, OPERATIONS, PERF | Toxiproxy 명령과 준비 상태 주기적 조회를 순서화해 데이터베이스/경계 실패와 복구를 버전이 명시된 JSON 보고서로 자동화합니다. |
| 5 | `335565908920` | `test(load): fault scenario 설정과 report 검증` | B | PERSISTENCE, OPERATIONS, PERF | 장애 실행기의 루프백 보호 조건, 명령 순서, 제한된 횟수의 주기적 조회, 보고서 스키마, 실패 정리를 결정적 의존성으로 검증합니다. |
| 6 | `eca21f115c1b` | `fix(db): idle connection pool 오류에서 복구` | A | PERSISTENCE, RISK | PostgreSQL 풀의 유휴 클라이언트 `error` 이벤트를 민감 정보를 제거한 오류 정보로 변환하고 잘못된 오류·오류 보고 콜백 실패가 프로세스 비정상 종료로 번지지 않게 격리합니다. |
| 7 | `493babe1cf30` | `test(db): 안전한 connection pool 오류 처리 검증` | B | PERSISTENCE, TEST | 실제 `pg.Pool` EventEmitter에서 유휴 오류가 민감 정보를 제거한 메타데이터로 관측되고, 오류 보고 함수가 없거나 예외를 던져도 오류가 외부로 전파되지 않는지 검증합니다. |

## 5. 커밋별 학습 기록

### 5.1. `test(load): 실시간 부하 임계값 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `ff1bffcd5296` |
| 중요도 | B |
| 태그 | REALTIME, PERSISTENCE, OPERATIONS |
| 원문에서 확인한 역할 | 500 연결·50 경기방을 기본으로 하는 k6/Toxiproxy 테스트 실행 틀의 서비스 수준 임계값과 구성 계약을 테스트 우선으로 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-harness.test.mjs`
- 핵심 심벌: `createLoadProfile` 예상 조건, 필수 k6 지표/소스 검증, 프록시 정의 검증
- 기본 500 연결, 50 경기방, 100 플레이어 연결, 495 최소 성공과 4분 시나리오를 확인합니다.
- 연결/재연결 99%, 스냅샷 p95≤150 ms·p99≤250 ms, 정상 상태의 스냅샷 폐기율이 1% 미만, 결과 확정 실패·중복 0건 등의 임계값을 확인합니다.
- k6 소스에 로그인, 일회용 티켓, 대기열 참가, 준비 완료, 순번이 있는 입력과 `serverTime` 측정이 필요하다고 소스 수준으로 고정하는지 확인합니다.
- PostgreSQL과 경계 프록시가 분리되고 부하용 추가 Compose 설정이 API DB 트래픽을 프록시로 라우팅하도록 요구하는지 확인합니다.
- 이 SHA에서 가져온 테스트 실행 틀 모듈이 아직 다음 커밋 구현 전이라는 테스트 우선 상태를 명시합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:ff1bffcd5296:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 실행 시점 지표와 호출 제한기는 있었지만 몇 연결/경기방에서 어떤 성공률·지연 시간·폐기/결과 확정 조건을 통과해야 하는지 실행 가능한 부하 계약이 없었습니다. 부하 도구를 먼저 만들면 측정 가능한 값에 맞춰 목표를 낮출 수 있고, 재연결·결과 확정·장애 경로 중 일부를 누락하기 쉽습니다. |
| 핵심 구현 결정 | Node 계약 테스트가 이후 구현할 `createLoadProfile`, k6 소스, Toxiproxy 정의, Compose 추가 설정에 필요한 숫자, 지표 이름, 라우트, 실패 경로를 먼저 고정합니다. |
| 실행/검증 경로 | 부하 계약 테스트 가져오기 → 프로필 생성 → 정확한 기본값과 임계값 검증 → k6 소스 순회 → 프록시 명령과 Compose 라우트 검증 순서입니다. 이 커밋만으로는 구현 모듈이 완성되지 않아 테스트 모음이 통과한다고 추정할 수 없습니다. |
| 소유권과 실패 처리 | 테스트 파일이 운영 허용 이벤트 종류를 소유합니다. 실제 연결/소켓/프록시/프로세스 수명은 후속 테스트 실행 틀이 소유합니다. 잘못된 연결 수 대비 경기방 수의 비율, 누락된 SLI, 단일 프록시로 DB/경계 실패를 혼합하는 구성을 검증 실패로 만듭니다. |
| 보장하는 것 | 후속 테스트 실행 틀이 충족해야 할 부하/장애 허용 계약이 숫자와 지표 이름 수준으로 명시됩니다. |
| 보장하지 않는 것 | 테스트 우선 커밋은 실제 500 연결을 실행하지 않고, 다음 커밋 전에는 참조 구현이 없을 수 있으므로 실행 시점 성공 근거가 아닙니다. |
| 후속 연결 | `7b0b5f086b41`이 k6와 Toxiproxy 구현을 추가하고 `547d9943d30a`가 측정 자체의 재연결·결과 확정 편향을 교정합니다. |
<!-- LEARNER-END:ff1bffcd5296:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:ff1bffcd5296:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 테스트 우선 정적/설정 계약 |
| 주입·재현 방식 | 부하 프로필 객체, 소스 파일 패턴, 프록시 정의, Compose 라우팅을 Node 검증으로 고정합니다. |
| 검증하는 것 | 요구되는 부하 규모·SLI·장애 구성이 소스 계약으로 명시됐음을 검증합니다. |
| 검증하지 않는 것 | 실제 k6 실행 결과, 임계값 통과, 네트워크/데이터베이스 복구를 검증하지 않습니다. |
<!-- LEARNER-END:ff1bffcd5296:test -->



#### 비교 기준

- 이 커밋의 부모 커밋의 상태와 비교합니다.
- 다음 관련 SHA: `7b0b5f086b41` — `test(load): 실시간 fault injection 도구 추가`

### 5.2. `test(load): 실시간 fault injection 도구 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `7b0b5f086b41` |
| 중요도 | A |
| 태그 | AUTH, REALTIME, PERSISTENCE |
| 원문에서 확인한 역할 | k6 실시간 부하 시나리오와 PostgreSQL/경계 Toxiproxy 제어 plane을 구현해 연결, 경기방, 재연결, 스냅샷, 결과 확정, 의존성 실패를 재현합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `docker-compose.load.yml`, `tests/load/load-profile.mjs`, `tests/load/pong-load.js`, `tests/load/toxiproxy-control.mjs`
- 핵심 심벌: `createLoadProfile`, k6 `setup/default`, `connectSession`, `buildProxyDefinitions`, `toxicForCommand`, `runCommand`
- 부하용 추가 Compose 설정이 Toxiproxy 제어 포트와 외부 경계 포트를 루프백에만 공개하고 API의 `DATABASE_URL`을 PostgreSQL 프록시로 바꾸는지 확인합니다.
- `createLoadProfile`이 0보다 큰 안전한 정수를 요구하고 연결≥2×경기방, 기본값/extended 프로필, 99% 성공 개수를 계산하는지 확인합니다.
- k6 VU가 개발 로그인→WS 티켓→버전이 명시된 WebSocket→대기열/준비 완료/입력→연결 해제/재연결→스냅샷/결과 확정을 어떤 순서로 수행하는지 확인합니다.
- 스냅샷 지연은 `Date.now() - serverTimeMs`, 전달 누락은 순번 차이, 재연결 복구는 같은 경기방 ID로 측정되는지 확인합니다.
- Toxiproxy 명령이 DB 지연 시간/중단·복구와 경계 지연 시간/초기화/중단·복구를 서로 다른 프록시/장애 조건으로 만들고 초기화/ensure를 어떻게 수행하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:7b0b5f086b41:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 임계값은 테스트 우선으로 정의됐지만 실제 부하 프로세스, 연결 순서, 의존성 장애 제어는 없었습니다. |
| 해결하려던 문제와 위험 | 단순 WebSocket 연결 수만 세면 인증 티켓, 대전 상대 찾기, 진행 중인 경기방, 순번이 있는 입력, 재연결 소유권, 스냅샷 최신성, 멱등 결과 확정을 검증하지 못합니다. DB와 외부 구간 장애도 분리 주입해야 원인을 구분할 수 있습니다. |
| 핵심 구현 결정 | k6 프로필과 시나리오를 추가해 기본 500 VU 중 100명이 50개 경기방을 만들고 나머지는 온라인 부하를 구성합니다. 로그인과 일회용 티켓을 거쳐 입력을 보내고 선택한 사용자를 연결 해제한 뒤 재연결합니다. Toxiproxy 추가 Compose 설정과 제어 스크립트가 PostgreSQL과 외부 경계를 별도 프록시로 구성합니다. |
| 입력 → 처리 → 결과 | Compose 부하용 추가 Compose 설정 기동 → Toxiproxy 준비 → k6 설정 준비 상태 → VU별 로그인/티켓/연결 → 접속 상태가 99% 이상일 때 플레이어 대기열 → 매칭 완료/준비 완료/입력 → 초기 연결 종료·새 티켓 재연결 → 스냅샷/결과 확정 지표 기록; 별도 제어 명령으로 DB/경계 장애 조건 적용·초기화입니다. |
| 소유 주체·수명·정리 | 각 k6 VU는 자신의 소켓, `inputSeq`, 관찰한 경기방·경기 집합, 타이머를 소유합니다. 부하 프로필은 임계값을, Toxiproxy 제어기는 프록시 정의와 장애 조건을, Compose 추가 설정은 프로세스 라우팅을 소유합니다. |
| 실패·되돌리기·재시도 | 로그인/티켓/업그레이드/재연결 실패는 빈도 지표로, 스냅샷 차이/지연은 Trend/빈도로, 중복/실패 결과 확정은 개수로 기록합니다. 프록시 명령 검증은 0 이하이거나 알 수 없는 인수를 즉시 실패합니다. |
| 보장하는 것 | 실시간 서비스 실행 순서와 DB/외부 구간 장애 주입을 같은 운영 테스트 실행 틀에서 재현하고 기계 판독 가능한 임계값으로 평가할 수 있습니다. |
| 보장하지 않는 것 | 초기 재연결이 모든 플레이어에게 같은 시점에 몰리고 결과 확정 성공을 클라이언트 `game.finished` 이벤트에 의존해 서버 영속 저장 결과와 혼동할 수 있습니다. `547d9943d30a`가 이를 교정합니다. |
| 후속 연결 | `547d9943d30a`가 재연결 시점을 분산하고 서버 Prometheus의 결과 확정 지표를 사용하도록 하며, `84bec3bf57ae`가 장애 순서 자체를 자동화합니다. |
<!-- LEARNER-END:7b0b5f086b41:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:7b0b5f086b41:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | k6 부하 테스트 도구와 Toxiproxy 장애 주입 구현 |
| 주입·재현 방식 | k6 VU가 HTTP 로그인·일회용 티켓·버전이 명시된 WebSocket·대기열/준비 완료/입력·재연결을 실행하고, Toxiproxy 제어 스크립트가 PostgreSQL/경계 지연 시간·중단·초기화를 적용하도록 구성합니다. |
| 검증하는 것 | 소스 검토상 요구된 실제 서비스 경로와 SLI/장애 제어를 실행할 테스트 실행 틀이 구현됐음을 검증합니다. |
| 검증하지 않는 것 | 이번 작업에서는 k6·Compose·Toxiproxy를 실행하지 않았으므로 임계값 통과나 실제 복구 결과를 검증하지 않습니다. |
<!-- LEARNER-END:7b0b5f086b41:test -->



#### 비교 기준

- 직전 관련 SHA: `ff1bffcd5296` — `test(load): 실시간 부하 임계값 정의`
- 다음 관련 SHA: `547d9943d30a` — `fix(load): 기본 부하 profile 측정 안정화`

### 5.3. `fix(load): 기본 부하 profile 측정 안정화`

| 항목 | 값 |
| --- | --- |
| SHA | `547d9943d30a` |
| 중요도 | A |
| 태그 | REALTIME, OPERATIONS, OBSERVABILITY |
| 원문에서 확인한 역할 | 재연결 폭주와 클라이언트 측 결과 확정 오판을 제거해 기본 부하 프로필의 측정 소스와 시간 제어를 안정화합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/load-profile.mjs`, `tests/load/pong-load.js`, `tests/load/load-harness.test.mjs`
- 핵심 심벌: `PLAYER_RECONNECT_STAGGER_MS`, `reconnectDelayFor`, k6 `teardown`, `readPrometheusSample`
- 기본 재연결 지연이 2초이고 플레이어 연결 수 전체에 5초 stagger를 분배하는 산술을 확인합니다.
- 연결 종료 타이머가 `queue.matched` 직후가 아니라 실제 `playing` 스냅샷을 본 뒤에만 arm되는지 확인합니다.
- VU의 `game.finished` 이벤트 대신 종료 정리에서 API Prometheus의 DB 결과 확정 성공, 실패, 중복 표본을 읽는지 확인합니다.
- `readPrometheusSample`이 지표 이름과 예상 정해진 값만 사용하는 라벨을 정확히 매칭하고 누락된/잘못된 이벤트 루프 지표를 실패 시 차단 처리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:547d9943d30a:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 테스트 실행 틀은 100 플레이어 연결이 거의 같은 시점에 끊겨 재연결 폭주를 만들었고, 클라이언트가 `game.finished`를 받았다는 사실을 저장된 결과 확정 성공처럼 세었습니다. |
| 해결하려던 문제와 위험 | 테스트 자체가 synchronized 장애를 만들어 운영 bottleneck을 과장할 수 있고, 전송 계층 이벤트 수신은 데이터베이스 커밋 성공·중복 부재를 검증하지 않습니다. |
| 핵심 구현 결정 | 플레이어 인덱스에 따라 2–7초 범위로 재연결 종료를 분산하고, 경기방이 실제 경기 중 스냅샷을 보낸 뒤에만 종료 타이머를 설정합니다. 경기 결과 확정 내역은 테스트 종료 시 서버 Prometheus 카운터에서 영속 저장 방식과 처리 결과 라벨로 읽습니다. |
| 입력 → 처리 → 결과 | 플레이어 경기/준비 완료 → 첫 경기 중 스냅샷 → `reconnectDelayFor(vuId)` 타이머 → 시간을 분산한 종료/재연결; 종료 정리 → `/metrics` 지표 수집 → 이벤트 루프 p95와 결과 확정 성공·실패·중복 파싱 → k6 지표에 기록입니다. |
| 소유 주체·수명·정리 | 부하 VU는 재연결 동작만 시작하고, 서버가 확정한 경기 결과의 처리 상태는 서버 지표에서 읽습니다. 종료 단계 파서는 지표 수집 결과 검증과 라벨 선택을 맡습니다. |
| 실패·되돌리기·재시도 | 경기 중 전에 종료해 재연결 시간 구간을 잘못 측정하는 사례와 클라이언트 이벤트를 영속 저장 성공으로 오인하는 사례를 제거합니다. 필수 이벤트 루프 지표가 없으면 0으로 대체하지 않고 테스트를 실패시킵니다. |
| 보장하는 것 | 기본 부하의 재연결이 시간축에 분산되고 결과 확정 SLI가 실제 서버 측 영속 저장 관찰을 반영합니다. |
| 보장하지 않는 것 | Prometheus 지표 수집 자체가 성공해야 하며 지표 수집 시점 이후의 늦은 결과 확정은 결과에 포함되지 않을 수 있습니다. 실제 실행 근거는 별도 실행이 필요합니다. |
| 후속 연결 | `db1ae3d47b96`의 서버 주기 수정 뒤 측정 테스트 실행 틀을 정렬하며 `84bec3bf57ae`의 자동화된 장애 복구와 함께 운영 근거를 강화합니다. |
<!-- LEARNER-END:547d9943d30a:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:547d9943d30a:fix -->
이전 가정: 모든 클라이언트가 동시에 재연결하고 클라이언트의 종료 이벤트만 확인해도 부하와 결과 확정을 판단할 수 있다고 보았습니다. 실제 실패: 재연결 요청이 한꺼번에 몰리고, 클라이언트가 영속 저장 성공 여부를 잘못 판단했습니다. 수정: 경기 중 상태인 연결만 시간차를 두고 재연결하며 서버 지표로 결과 확정을 판정하도록 바꾸고 부하 계약 테스트를 갱신했습니다.
<!-- LEARNER-END:547d9943d30a:fix -->


#### 비교 기준

- 직전 관련 SHA: `7b0b5f086b41` — `test(load): 실시간 fault injection 도구 추가`
- 다음 관련 SHA: `84bec3bf57ae` — `test(load): fault recovery 검사 자동화`

### 5.4. `test(load): fault recovery 검사 자동화`

| 항목 | 값 |
| --- | --- |
| SHA | `84bec3bf57ae` |
| 중요도 | A |
| 태그 | PERSISTENCE, OPERATIONS, PERF |
| 원문에서 확인한 역할 | Toxiproxy 명령과 준비 상태 주기적 조회를 순서화해 데이터베이스/경계 실패와 복구를 버전이 명시된 JSON 보고서로 자동화합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/fault-scenario.mjs`, `package.json`
- 핵심 심벌: `createFaultScenarioConfig`, `runFaultScenario`, `observeStep`, `probeReadiness`, `formatFaultReport`, `load:faults`
- 루프백 전용 Toxiproxy·API·경계 URL과 DB 지연 300ms, 경계 지연 150ms, 요청 시간 초과 5초, 복구 시간 초과 15초, 기본 폴링 주기 250ms를 확인합니다.
- `pollIntervalMs <= recoveryTimeoutMs`와 성공 integer/boolean 파싱이 즉시 실패하는지 확인합니다.
- 초기화→기준 상태→DB 지연 시간→DB 중단→DB up/복구→경계 지연 시간→경계 초기화→경계 up/복구 순서를 확인합니다.
- DB 중단 예상 상태가 HTTP 503, `status=not_ready`, `checks.database=down`인지 확인합니다.
- 시나리오 오류가 나도 최종 초기화를 시도하고 정리 오류를 원래 오류와 어떻게 결합하는지 확인합니다.
- 보고서가 `schemaVersion`, 타임스탬프, `settings`, `targets`, 정렬된 단계, `passed`를 보존하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:84bec3bf57ae:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Toxiproxy 명령은 수동으로 호출할 수 있었지만 어떤 순서로 장애를 넣고 어떤 준비 상태를 기다린 뒤 복구를 판단할지 자동화되지 않았습니다. |
| 해결하려던 문제와 위험 | 수동 장애 실험은 정리 누락으로 다음 run을 오염시키고, 한 번의 확인 요청으로 복구를 판정하면 시간 제어 경쟁 상태가 생깁니다. 실패 관찰과 복구 관찰을 버전이 명시된 근거로 남겨야 했습니다. |
| 핵심 구현 결정 | 설정 파서와 시나리오 실행기를 만들고 각 단계에서 제한된 횟수의 주기적 조회로 예상 준비 상태를 기다립니다. 모든 경로에서 프록시 초기화를 시도하며 성공한 관측 결과를 정렬된 JSON 보고서에 기록합니다. |
| 입력 → 처리 → 결과 | 설정 검증 → 프록시 초기화 → 기준 준비 상태 폴링 → DB 지연 주입 후 준비 상태 확인 → DB 중단 시 503과 `down` 확인 → DB 복구 후 `ready` 확인 → 선택적 경계 지연 주입 후 준비 상태 확인 → 경계 초기화 중 네트워크 오류·5xx 확인 → 경계 복구 후 `ready` 확인 → 최종 초기화 순서입니다. 모든 단계가 통과하면 보고서를 확정합니다. |
| 소유 주체·수명·정리 | 실행기가 장애 명령 순서, 주기적 조회 기한, 관측 결과, 보고서 수명주기를 소유합니다. Toxiproxy 제어기가 장애 조건 변경을, API 준비 상태가 의존성 상태 해석을 소유합니다. |
| 실패·되돌리기·재시도 | 명령/확인 요청/기한 실패는 시나리오 오류로 수렴하되 최종 초기화를 별도로 시도합니다. 정리도 실패하면 원래 오류를 잃지 않고 원인으로 연결합니다. 대상 URL은 루프백 외부를 거부해 잘못된 환경 조작을 막습니다. |
| 보장하는 것 | 데이터베이스와 외부 경계의 실패·복구가 명시된 순서와 상한이 있는 기한으로 자동 실행되고, 버전이 명시된 JSON 근거로 표현됩니다. |
| 보장하지 않는 것 | 실행기 자체가 운영 데이터베이스 정확성을 검증하지 않으며, 실제 Toxiproxy/Compose stack이 떠 있어야 종단 간 실행됩니다. 이 작업에서는 해당 명령을 실행하지 않았습니다. |
| 후속 연결 | `335565908920`이 설정/순서/보고서 작성·정리를 주입한 의존성으로 검증하고, 실제 DB 유휴 연결 실패 격리는 `eca21f115c1b`에서 애플리케이션 코드로 수정됩니다. |
<!-- LEARNER-END:84bec3bf57ae:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:84bec3bf57ae:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 자동화된 장애 복구 시나리오 |
| 주입·재현 방식 | Toxiproxy 명령과 준비 상태 확인 요청을 초기화→기준 상태→DB 지연 시간/중단·복구→경계 지연 시간/초기화/up 순서로 실행하고 제한된 횟수의 주기적 조회와 최종 초기화, 버전이 명시된 JSON 보고서를 적용합니다. |
| 검증하는 것 | 실행 시 장애/복구 상태를 정렬된 관측 결과와 기한으로 판정할 실행 조정 경로가 존재함을 검증합니다. |
| 검증하지 않는 것 | 실제 인프라를 실행하지 않았으므로 데이터베이스/경계가 해당 시간 안에 복구됐다는 실행 근거는 아닙니다. |
<!-- LEARNER-END:84bec3bf57ae:test -->



#### 비교 기준

- 직전 관련 SHA: `547d9943d30a` — `fix(load): 기본 부하 profile 측정 안정화`
- 다음 관련 SHA: `335565908920` — `test(load): fault scenario 설정과 report 검증`

### 5.5. `test(load): fault scenario 설정과 report 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `335565908920` |
| 중요도 | B |
| 태그 | PERSISTENCE, OPERATIONS, PERF |
| 원문에서 확인한 역할 | 장애 실행기의 루프백 보호 조건, 명령 순서, 제한된 횟수의 주기적 조회, 보고서 스키마, 실패 정리를 결정적 의존성으로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `tests/load/fault-scenario.test.mjs`
- 핵심 심벌: 주입한 `applyToxiproxyCommand`, `probeReadiness`, `sleep`, `now`, 가짜 관찰 queues
- 기본값 설정과 `FAULT_INCLUDE_EDGE=0`, 잘못된 외부 URL를 검사하는 사례를 확인합니다.
- prepared 준비 상태 관측 결과가 DB/경계 단계 순서와 두 번의 250 ms 대기를 만들고 정렬된 명령을 고정하는지 확인합니다.
- 보고서 생성 시각, 대상 URL, 단계 이름·상태·소요 시간·응답 본문·오류와 JSON 왕복 변환을 확인합니다.
- `db-down` 명령이 예외를 던져도 마지막 `reset` 명령이 호출되는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:335565908920:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 장애 실행기는 여러 주입 지점과 정리 경로를 가졌지만 실제 프록시나 네트워크 없이 그 순서를 결정적으로 검증할 근거가 없었습니다. 종단 간 장애 실행만으로는 실패 시 정리가 실행됐는지, 주기적 조회가 어떤 관찰값을 받아 통과했는지 재현하기 어렵습니다. |
| 구현 또는 검증 결정 | 명령/확인 요청/대기/시계를 가짜로 주입해 고정된 관찰 대기열을 소비하고 정확한 명령 목록, 대기 목록, 보고서 필드를 검사합니다. |
| 실행/검증 경로 | 설정 빌드 → 가짜 명령과 상태 확인 요청으로 전체 장애 시나리오 실행 → 예상 단계 목록과 보고서 확인 순서입니다. 별도 실패 사례에서는 `db-down` 예외 발생 → 오류 처리 → 마지막 초기화 호출 확인 순서입니다. |
| 소유권과 실패 처리 | 테스트가 모든 외부 부수 효과를 가짜 의존성으로 소유하고 실행기의 실행 조정 로직만 실행합니다. 루프백이 아닌 대상, 잘못된 설정, 명령 실행 중 실패, 정리 초기화를 결정적 검증으로 재현합니다. |
| 보장하는 것 | 장애 시나리오 제어 로직과 보고서 계약이 실제 인프라 사용 가능 상태와 무관하게 회귀 보호됩니다. |
| 보장하지 않는 것 | Toxiproxy API, 네트워크 초기화, PostgreSQL 준비 상태가 실제로 동작하는지는 검증하지 않습니다. |
| 후속 연결 | `84bec3bf57ae`의 운영 테스트 실행 틀 근거이며 `eca21f115c1b`/`493babe1cf30`은 장애가 애플리케이션 풀 이벤트로 나타날 때의 격리를 다룹니다. |
<!-- LEARNER-END:335565908920:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:335565908920:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 결정적 장애 실행 조정 테스트 |
| 주입·재현 방식 | 명령/확인 요청/대기/시계 의존성 주입과 대기 중인 관측 결과를 사용합니다. |
| 검증하는 것 | 정렬된 장애/복구 단계, 루프백 보호 조건, 보고서 스키마, 항상 실행하는 초기화 정리를 검증합니다. |
| 검증하지 않는 것 | 실제 DB/경계 장애와 복구 지연 시간은 검증하지 않습니다. |
<!-- LEARNER-END:335565908920:test -->



#### 비교 기준

- 직전 관련 SHA: `84bec3bf57ae` — `test(load): fault recovery 검사 자동화`
- 다음 관련 SHA: `eca21f115c1b` — `fix(db): idle connection pool 오류에서 복구`

### 5.6. `fix(db): idle connection pool 오류에서 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `eca21f115c1b` |
| 중요도 | A |
| 태그 | PERSISTENCE, RISK |
| 원문에서 확인한 역할 | PostgreSQL 풀의 유휴 클라이언트 `error` 이벤트를 민감 정보를 제거한 오류 정보로 변환하고 잘못된 오류·오류 보고 콜백 실패가 프로세스 비정상 종료로 번지지 않게 격리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/poolError.ts`, `packages/db/src/index.ts`, `apps/api/src/index.ts`
- 핵심 심벌: `installPostgresPoolErrorHandler`, `toSafePoolErrorEvent`, `safeLabel`, `PostgresRepositoryOptions.onPoolError`, 초기 오류 버퍼
- `Pool.on("error")` 리스너가 설치되지 않았던 부모 커밋과 Node EventEmitter의 처리되지 않은 오류 동작을 비교합니다.
- 원래 오류 메시지/연결 문자열 대신 `{kind,errorName,errorCode}`만 만들고 라벨을 `[A-Za-z0-9_]{1,64}`로 제한하는지 확인합니다.
- 잘못된 오류 객체 변환과 사용자가 제공한 오류 보고 콜백을 각각 try/catch로 격리하는지 확인합니다.
- `createPostgresRepository`가 `onPoolError` 옵션을 받고 풀 생성 직후 리스너를 설치하는 순서를 확인합니다.
- API 진입점이 Fastify 로거 준비 전 발생한 이벤트를 배열에 임시 보관한 뒤 로거 소유 주체가 생기면 구조화된 로그로 전달하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:eca21f115c1b:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | PostgreSQL 풀은 쿼리 중이 아닌 유휴 클라이언트에서 연결 실패를 `error` 이벤트로 방출할 수 있었고, 리스너가 없으면 EventEmitter 동작 의미상 프로세스에서 잡지 못한 오류가 될 수 있었습니다. |
| 해결하려던 문제와 위험 | 데이터베이스 장애 실험 중 유휴 연결 오류가 준비 상태 503으로만 표현되지 않고 API 프로세스를 종료시킬 수 있었습니다. 원래 오류를 그대로 로그하면 인증 정보와 호스트 세부 정보가 노출되고, 오류 보고 함수 자체가 예외를 던져도 비정상 종료가 재발합니다. |
| 핵심 구현 결정 | 풀 생성 즉시 오류 리스너를 설치해 민감 정보가 없고 필드가 제한된 메타데이터로 변환합니다. 변환과 선택적 오류 보고 콜백를 별도 try/catch로 감싸 이벤트 경계 밖으로 예외가 나오지 않게 합니다. API는 애플리케이션 로거가 준비되기 전 이벤트를 버퍼하고 준비 후 구조화된 오류 로그로 다음 전송을 시도합니다. |
| 입력 → 처리 → 결과 | 유휴 PG 클라이언트 오류 발생 → 풀 리스너 → 안전한 이벤트 변환 또는 기본값 적용 → 선택적 오류 보고 콜백 호출(예외가 발생해도 격리됨) → 시작 전이면 `earlyPoolErrors.push` → 애플리케이션 로거 준비 뒤 오류 보고 대상 교체·버퍼 비우기 → 이후 이벤트는 구조화된 로그로 직접 기록됩니다. |
| 소유 주체·수명·정리 | DB 패키지가 풀 리스너와 민감 정보 제거 경계를 소유합니다. API 객체 조립 루트는 로거 준비 상태 전 임시 이벤트 버퍼와 이후 보고 대상을 소유합니다. 저장소 종료는 기존 풀 수명 소유 주체를 유지합니다. |
| 실패·되돌리기·재시도 | 잘못된 이름이나 코드는 기본값 `UnknownError`/null로 축소되고 원시 메시지는 전달되지 않습니다. 오류 보고 콜백이 예외를 던져도 리스너가 흡수하므로 유휴 클라이언트 오류가 프로세스 비정상 종료로 이어지지 않습니다. 초기 버퍼는 로거 설치 후 `splice`로 비웁니다. |
| 보장하는 것 | 유휴 연결 풀 오류가 잡지 못한 EventEmitter 오류로 프로세스를 종료하지 않고, 인증 정보가 포함된 원문 메시지 없이 필드가 제한된 메타데이터로 관측됩니다. |
| 보장하지 않는 것 | 활성 쿼리 실패를 성공으로 바꾸거나 데이터베이스를 자동 재연결한다고 보장하지 않습니다. 준비 상태와 클라이언트 재시도는 별도 `pg`/저장소 동작입니다. |
| 후속 연결 | `493babe1cf30`이 실제 `pg.Pool` 이벤트와 의도적으로 예외를 던지는 오류 보고 콜백을 검증합니다. `84bec3bf57ae`의 DB 장애/복구 시나리오와 함께 프로세스 수준 격리 과정을 완성합니다. |
<!-- LEARNER-END:eca21f115c1b:record -->


#### 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:eca21f115c1b:fix -->
이전 가정: 쿼리 실패/준비 상태 처리만으로 DB 실패가 격리됨 → 실제 실패: 유휴 클라이언트 `error` EventEmitter 오류가 잡히지 않아 비정상 종료할 수 있음 → 근본 원인: 풀 리스너 부재 → 수정: 안전한 리스너+오류 보고 콜백 격리+초기 로깅 버퍼 → 회귀 `493babe1cf30`.
<!-- LEARNER-END:eca21f115c1b:fix -->


#### 비교 기준

- 직전 관련 SHA: `335565908920` — `test(load): fault scenario 설정과 report 검증`
- 다음 관련 SHA: `493babe1cf30` — `test(db): 안전한 connection pool 오류 처리 검증`

### 5.7. `test(db): 안전한 connection pool 오류 처리 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `493babe1cf30` |
| 중요도 | B |
| 태그 | PERSISTENCE, TEST |
| 원문에서 확인한 역할 | 실제 `pg.Pool` EventEmitter에서 유휴 오류가 민감 정보를 제거한 메타데이터로 관측되고 오류 보고 콜백이 없거나 예외를 던지는 경우에서도 외부로 예외가 전파되지 않는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 파일: `packages/db/src/poolError.test.ts`
- 핵심 심벌: 실제 `Pool`, `pool.emit('error', error)`, 오류 보고 콜백 호출 감시 객체/예외 발생 사례
- 실제 `new Pool()`에 리스너가 정확히 하나 설치되는지 확인합니다.
- 비밀값을 포함한 메시지·connectionString과 코드 `57P01`를 가진 오류를 생성하고 콜백 메시지 본문만 `{kind:'idle_client_error', errorName:'Error', errorCode:'57P01'}`인지 확인합니다.
- 직렬화된 가짜 객체 호출에 `secret`과 `Connection terminated`가 없는지 확인하는 검증을 확인합니다.
- 오류 보고 콜백 미설정과 오류 보고 콜백 자체의 예외 발생 두 사례 모두 `pool.emit`이 예외 발생하지 않는지 확인합니다.
- 각 사례가 `pool.end()`로 풀 수명을 정리하는지 확인합니다.

#### 학습자 기록

<!-- LEARNER-BEGIN:493babe1cf30:record -->
| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태와 문제 | 안전한 리스너 구현은 있었지만 EventEmitter의 실제 `error` 동작 의미, 원시 인증 정보 비밀값 제거, 오류 보고 콜백 실패 격리를 함께 고정하지 않았습니다. 테스트용 이벤트 방출기는 Node의 특별한 처리되지 않은 `error` 동작을 놓칠 수 있고, 민감 정보 제거 함수가 원래 오류 참조를 메시지 본문에 남기면 JSON/로그에서 비밀값이 노출될 수 있습니다. |
| 구현 또는 검증 결정 | 실제 `pg.Pool` 인스턴스에 처리 함수를 설치하고 의도적으로 만든 오류를 직접 생성합니다. 오류 보고 콜백 호출 형식과 직렬화된 실패 문자열 조각, 오류 보고 콜백이 없거나 예외를 던지는 경우 비 예외 발생 동작을 검사합니다. |
| 실행/검증 경로 | 풀 생성 → 오류 처리 함수 설치와 리스너 수 확인 → 비밀값을 포함한 오류 방출 → 프로세스 예외가 발생하지 않는지 확인 → 정제된 콜백 인수 확인 → `pool.end()` 호출 순서입니다. 별도 사례에서는 오류 보고 콜백이 없거나 오류 보고 콜백이 예외를 던져도 프로세스가 종료되지 않으며 풀이 닫히는지 확인합니다. |
| 소유권과 실패 처리 | 테스트가 풀 생성·종료를 소유하고 운영 처리 함수가 이벤트 변환과 보고 격리를 실행합니다. 처리되지 않은 EventEmitter 오류, 인증 정보 유출, 오류 보고 콜백이 유발하는 비정상 종료를 각각 직접 재현할 입력으로 검사합니다. |
| 보장하는 것 | 유휴 연결 풀 오류 경계가 실제 `pg.Pool`에서 프로세스 예외를 방지하고 제한된 민감 정보를 제거한 메타데이터만 보고함을 검증합니다. |
| 보장하지 않는 것 | 실제 PostgreSQL 서버 연결 해제/재연결이나 준비 상태 복구 시간 제어는 실행하지 않습니다. |
| 후속 연결 | `eca21f115c1b`의 루트 원인 수정을 보호하는 최종 데이터베이스 장애 격리 회귀입니다. |
<!-- LEARNER-END:493babe1cf30:record -->

#### 테스트·측정 기록

<!-- LEARNER-BEGIN:493babe1cf30:test -->
| 구분 | 기록 |
| --- | --- |
| 검증 종류 | 실제 라이브러리 EventEmitter 회귀 테스트 |
| 주입·재현 방식 | 실제 `pg.Pool`에 crafted 유휴 오류를 생성하고 콜백 형식·실패 비밀값 strings·예외 발생 격리를 검사합니다. |
| 검증하는 것 | 리스너 존재, 안전한 라벨, 원본 메시지와 비밀값이 없음, 오류 보고 콜백이 없거나 예외를 던지는 경우 비정상 종료하지 않는 동작을 검증합니다. |
| 검증하지 않는 것 | 실제 네트워크 장애와 풀 reconnection 성공은 검증하지 않습니다. |
<!-- LEARNER-END:493babe1cf30:test -->



#### 비교 기준

- 직전 관련 SHA: `eca21f115c1b` — `fix(db): idle connection pool 오류에서 복구`
- 이 개발 흐름의 마지막 선택한 SHA입니다.

## 6. 불변 조건 변화

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:evolution -->
`ff1bffcd5296`은 구현 전에 부하 규모와 SLI 임계값을 고정하고 `7b0b5f086b41`은 k6·Toxiproxy 실행 틀을 구현합니다. `547d9943d30a`는 동시에 몰리는 재연결과 클라이언트 측 결과 확정 오판을 수정해, 경기 중인 연결만 시간차를 두고 재연결하고 서버 Prometheus 지표를 판정 근거로 사용합니다. `84bec3bf57ae`·`335565908920`은 DB·경계 장애와 복구를 횟수 제한 폴링, 항상 실행하는 초기화, JSON 보고서로 자동화하고 검증합니다. `eca21f115c1b`·`493babe1cf30`은 DB 장애가 유휴 풀의 `error` 이벤트로 나타나도 안전한 메타데이터만 관측하도록 격리합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:evolution -->

## 7. 실패 → 수정 → 테스트 관계

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:failure-links -->
- 부하 목표/SLI 누락 → 테스트 우선 임계값 계약 → k6/Toxiproxy 구현
- 재연결 폭주·클라이언트 결과 확정 오판 → 경기 중 상태를 확인한 뒤 시간차를 둔 재연결/서버 지표 → 테스트 실행 틀 계약 갱신
- 수동 장애 실험의 환경 오염·경쟁 상태 → 실행 순서와 최대 시간이 정해진 실행기, 항상 수행하는 초기화 → 의존성을 주입한 보고서 작성·정리 테스트
- 유휴 풀의 `EventEmitter` 비정상 종료·비밀값 로그·오류 보고 콜백 예외 → 안전한 리스너와 초기 오류 버퍼 → 실제 `pg.Pool` 회귀 테스트
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:failure-links -->

## 8. 소유권·상태·정리 변화

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:ownership -->
k6 프로필은 허용 임계값을, 각 VU는 연결/입력/재연결 관찰을, 종료 정리는 서버 지표 수집을 소유합니다. Toxiproxy 제어기는 DB/경계 장애 조건 변경을, 장애 실행기는 정렬된 주기적 조회/보고서 작성·정리를 소유합니다. DB 패키지는 풀 오류 이벤트의 민감 정보 제거와 격리를, API 객체 조립 루트는 로거 준비 전 이벤트 버퍼를 소유합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:ownership -->

## 9. 최종 상태

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:final-state -->
기본 부하 계약은 500개 연결과 50개 경기방, 연결·재연결·스냅샷·결과 확정 임계값을 정의합니다. 재연결 시점은 경기 진행 중에 분산하고 저장된 결과 확정은 서버 지표에서 읽습니다. DB와 외부 경계 장애는 루프백 전용 Toxiproxy로 순서대로 주입해 준비 상태 복구 JSON 보고서를 만들며, 유휴 PG 연결 풀 오류는 프로세스를 종료시키지 않고 안전한 이름과 코드만 로그합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:final-state -->

## 10. 최종 실행 흐름

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:final-flow -->
부하용 추가 Compose 설정 → Toxiproxy 준비 → k6 준비 상태/로그인/티켓/ws/경기방/입력/재연결 → 종료 정리 Prometheus 평가. 장애 실행은 기준 상태→DB 지연 시간/중단·복구→경계 지연 시간/초기화/up→항상 실행하는 초기화를 주기적 조회/보고서로 기록합니다. DB 장애 중 유휴 풀 오류는 리스너→민감 정보 제거 함수→오류 보고 콜백 격리→초기 오류 버퍼와 로거로 흐르며 준비 상태는 별도로 의존성 상태를 반환합니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:final-flow -->

## 11. 실행 및 검증 근거

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:execution -->
- 저장소 실행 시점/테스트 명령은 실행하지 않았습니다.
- 실행을 시도한 명령: `git ls-remote --heads https://github.com/seungwoo7050/42-archive.git refs/heads/web/ft_transcendence`
- 실제 결과: 종료 상태 128, `Could not resolve host: github.com`.
- 따라서 테스트 통과, 벤치마크 수치, k6/Toxiproxy 복구 결과는 주장하지 않습니다. 각 기록은 GitHub 연결로 정확한 선택한 커밋의 변경 내용과 당시 파일을 확인한 정적 과거 검토 결과입니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:execution -->

## 12. 학습 완료 확인

<!-- LEARNER-BEGIN:06-load-fault-recovery-and-pool-error-containment.md:checks -->
- [x] 테스트 우선 부하 계약과 실제 테스트 실행 틀 구현을 구분할 수 있습니다.
- [x] 재연결 시점을 분산하는 처리과 서버 측 결과 확정 지표로 바뀐 근본 원인을 설명할 수 있습니다.
- [x] 장애 실행기의 정렬된 단계, 예상 준비 상태, 시간 초과, 항상 실행하는 초기화 정리를 추적할 수 있습니다.
- [x] 유휴 풀 오류가 준비 상태 실패와 별개로 프로세스 비정상 종료가 될 수 있었던 이유를 설명할 수 있습니다.
- [x] 민감 정보 제거 함수와 오류 보고 콜백 격리가 보장하는 것과 DB 재연결을 보장하지 않는 것을 구분할 수 있습니다.
<!-- LEARNER-END:06-load-fault-recovery-and-pool-error-containment.md:checks -->
===== END FILE: 06-load-fault-recovery-and-pool-error-containment.md =====

===== BEGIN FILE: README.md =====
# 런타임 관측성과 서비스 상태

이 카테고리는 애플리케이션 시작/준비 상태, 마이그레이션 상태 확인, Prometheus 관측 지점, 이벤트 루프 및 실시간 전달 측정, 실행량이 제한된 작업, GameHub 스케줄러 소유권, 단계적 종료 준비, 부하/장애 복구와 데이터베이스 연결 풀 오류 격리를 다룹니다.

## 범위

- 저장소: `https://github.com/seungwoo7050/42-archive`
- 브랜치: `web/ft_transcendence`
- 카테고리: `07-runtime-observability-and-service-health`
- 상태: 1단계 감사 완료, 동결된 작업 틀
- 제외: 일반적인 Docker/Caddy 이미지 구성, CI 배포 작업, 릴리스 산출물, 의존성 패치, 미디어 자산 생성은 `09-production-delivery-and-release-engineering`의 범위입니다.
- 교차 참조 허용: 애플리케이션 종료 준비 보장 범위를 직접 성립시키는 컨테이너 종료 유예 시간, 실행 상태 확인을 직접 검증하는 부하/장애 테스트 실행 틀은 이 카테고리에 포함합니다.

## 1단계 감사 결과

초기 초안은 3개 개발 흐름, 31개 선택한 커밋으로 구성돼 있었습니다. 실제 `web/ft_transcendence`의 선형 이력과 `commit/commit-importance.md` 분류를 대조한 뒤 6개 개발 흐름, 57개 고유 선택한 커밋으로 보정했습니다.

- 시작/준비 상태 개발 흐름은 기존 10개 커밋을 유지했습니다.
- 지표 개발 흐름은 `prom-client` 의존성과 이벤트 루프 부하 노출과 임계값 근거를 추가했습니다.
- 기존 런타임 상한 개발 흐름은 기본 요소, GameHub 통합/공유 스케줄링, 종료 준비, 부하/장애/풀 격리의 독립된 이야기로 분리했습니다.
- 경기방별 타이머에서 공유 스케줄러로 이동한 근거인 스케줄러 벤치마크와 결정적 수명주기 테스트를 추가했습니다.
- 애플리케이션의 60초 종료 준비와 컨테이너의 70초 종료 유예 시간을 하나의 여러 영역에 걸친 계층 불변 조건으로 연결했습니다.
- 기존에 잘못 배열된 뒤늦게 추가된 이력을 실제 브랜치 순서에 맞게 이동했습니다. 특히 주기/부하/장애/풀 수정과 콜백 혼잡 수정의 순서를 분리했습니다.
- 초안에 있던 커밋은 제거하지 않았으며, DB 풀 관련 커밋은 올바른 장애 격리 개발 흐름으로 이동했습니다.
- 일반 로깅 비밀값 제거, 빌드 이미지, CI 전달과 의존성 패치는 이 카테고리의 독립 구현 과정이 아니므로 추가하지 않았습니다.

1단계 종료 뒤 이 `scaffold/` 파일 집합을 동결했습니다. `completed/`는 동일한 파일명·구조·고정된 본문을 보존하고 학습자 작성 블록만 채운 사본입니다.

## 개발 흐름

1. [시작·생존 상태·준비 상태·저장소 상태](01-startup-liveness-readiness-and-storage-state.md)
2. [지표 관측 지점과 라벨 조합 수](02-metrics-observer-boundaries-and-cardinality.md)
3. [실행 중 호출 제한 기본 요소와 작업량 제한](03-runtime-limiter-primitives-and-bounded-work.md)
4. [GameHub 실행 시점 통합, 공유 스케줄링과 혼잡](04-gamehub-runtime-integration-shared-scheduling-and-congestion.md)
5. [Draining 준비 상태와 단계적 종료](05-draining-readiness-and-graceful-shutdown.md)
6. [부하·장애 복구와 연결 풀 오류 격리](06-load-fault-recovery-and-pool-error-containment.md)

## 사용 원칙

- 각 문서의 커밋 목록 순서를 유지합니다.
- 정확한 SHA의 코드와 부모 커밋 또는 직전 관련 SHA를 확인합니다.
- 다른 카테고리에서 같은 SHA를 교차 참조하더라도 이 문서의 실행 상태 확인/소유권/실패 질문에 맞는 근거만 기록합니다.
- 실행하지 않은 테스트·벤치마크·장애 시나리오의 결과는 기록하지 않습니다.
- `scaffold/`의 learner 블록 외 고정된 본문은 2단계에서 변경하지 않습니다.
===== END FILE: README.md =====

