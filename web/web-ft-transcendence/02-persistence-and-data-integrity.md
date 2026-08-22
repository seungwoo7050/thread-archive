===== BEGIN FILE: 01-repository-abstraction-backend-parity-and-read-models.md =====
# Development Thread 01 — Repository 추상화·backend parity·read model

## 1. 학습 목표

- 초기 relational schema가 typed row mapper와 `AppRepository`로 감싸지는 과정을 exact SHA 순서로 재구성합니다.
- PostgreSQL과 memory 구현의 공통 surface와 실제로 남아 있던 의미 차이를 구분합니다.
- recent match·dashboard read model의 잘못된 파생 지표가 fix와 regression test로 닫히는 관계를 설명합니다.
- memory behavioral test와 실제 PostgreSQL integration harness가 서로 무엇을 증명하고 무엇을 증명하지 않는지 구분합니다.

## 2. 범위와 경계

- 포함: 초기 DB schema, user/session row mapping, repository lifecycle, seed/upsert, profile·leaderboard·recent-match·dashboard read model, backend test evidence.
- 포함: `bestStreak`의 fabricated 구현을 고친 `035b97ca7c58`과 regression `6b661420e060`.
- 포함: PostgreSQL integration command/dependency `e935054ce0c9`과 harness `c43b87694b29`.
- 제외: session security, match-result atomic finalization, chat persistence, tournament progression, admin audit atomicity는 독립 Thread/카테고리에서 다룹니다.
- 제외: 실제 WebSocket presence는 persistence user list가 아니라 realtime authority로 이동하므로 이 Thread의 parity 결론으로 일반화하지 않습니다.

## 3. 핵심 질문

- `AppRepository`가 concrete backend 자원과 API caller 사이에서 어떤 lifetime을 소유합니까?
- compile-time row type, row mapper, repository method는 각각 어떤 오류를 막고 어떤 오류는 막지 못합니까?
- memory와 PostgreSQL이 같은 interface를 구현해도 왜 동일한 의미를 자동으로 보장하지 않습니까?
- recent-match newest-first 순서에서 실제 최고 연승을 계산하려면 왜 reverse와 loss reset이 모두 필요합니까?
- unit/behavioral test와 Testcontainers integration test의 증거 범위는 어떻게 다릅니까?

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags |
| ---: | --- | --- | :---: | --- |
| 1 | `0e850d24406e` | `feat(db): 초기 PostgreSQL schema 정의` | B | PERSISTENCE, TOURNAMENT |
| 2 | `4aa060c0b8df` | `feat(db): 사용자와 세션 row schema 정의` | B | AUTH, PERSISTENCE |
| 3 | `9277572765e7` | `feat(db): 저장소 lifecycle 구성` | B | PERSISTENCE, OPERATIONS |
| 4 | `fb516f723cdf` | `feat(db): 개발 사용자 seed 저장 구현` | B | PERSISTENCE |
| 5 | `c5b96a06925c` | `feat(db): 프로필 조회와 변경 저장 구현` | B | PERSISTENCE |
| 6 | `0364c42f776b` | `feat(db): 순위 조회 구현` | B | PERSISTENCE |
| 7 | `8ab49e5f2dd4` | `feat(db): 경기 조회 row contract 정의` | B | PERSISTENCE |
| 8 | `c7ea1ff241c8` | `feat(db): 최근 경기와 대시보드 조회 구현` | B | PERSISTENCE |
| 9 | `6509e32ba95d` | `test(db): 메모리 저장소 흐름 검증` | B | PERSISTENCE, TOURNAMENT, TEST |
| 10 | `035b97ca7c58` | `fix(db): 최근 경기에서 최고 연승 계산` | B | PERSISTENCE |
| 11 | `6b661420e060` | `test(db): 최고 연승 계산 검증` | B | PERSISTENCE, TEST |
| 12 | `e935054ce0c9` | `build(db): PostgreSQL integration 의존성과 명령 추가` | B | PERSISTENCE |
| 13 | `c43b87694b29` | `test(db): PostgreSQL integration 환경과 계약 추가` | A | PERSISTENCE, RISK, TEST |

## 5. Commit별 조사

### 5.1. `0e850d24406e` — feat(db): 초기 PostgreSQL schema 정의

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `0e850d24406e` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | 사용자·세션·친구·경기·채팅·토너먼트·관리 작업의 최초 durable relational model을 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/migrations/001_initial.sql`의 `pgcrypto` extension, 각 `create table`, 기본값과 nullable column을 확인합니다.
- `sessions`, `friendships`, `tournament_entries`의 foreign key와 `on delete cascade`가 어떤 row lifetime을 묶는지 추적합니다.
- `friendships(requester_id, addressee_id)`와 `tournament_entries(tournament_id, user_id)`의 초기 방향성 unique 제약을 확인합니다.
- `matches_ended_at_idx`, `chat_scope_created_at_idx`가 지원하려는 실제 조회 순서를 확인합니다.
- self-friend, canonical pair, tournament capacity·seed uniqueness, status enum이 아직 DB 제약으로 강제되지 않는다는 점을 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | DB workspace는 있었지만 애플리케이션 상태를 보존할 PostgreSQL table과 관계가 없었습니다. |
| 해결하려던 문제 | 사용자·인증·사회 관계·경기 결과·토너먼트를 프로세스 재시작 뒤에도 식별하고 연결할 durable schema가 필요했습니다. |
| 핵심 결정 | `001_initial.sql` 한 migration에서 UUID 기본키, timestamp 기본값, foreign key, 초기 unique/index를 함께 선언했습니다. |
| 입력 → 상태 전이 → 출력 | migration 실행 → extension과 table 생성 → FK·unique·조회 index 설치 → 이후 repository가 동일 row를 읽고 씁니다. |
| ownership / lifetime / cleanup | PostgreSQL이 UUID·생성 시각·관계 row의 lifetime을 소유합니다. `sessions`와 참가 row는 부모 삭제 시 cascade되고, 애플리케이션은 아직 cleanup을 직접 조정하지 않습니다. |
| failure / rollback / retry | DDL 적용 중 오류가 나면 migration 실행이 실패합니다. 이 SHA 자체에는 기존 데이터 정규화, 재시도 정책, down migration이 없습니다. |
| 보장하는 것 | 핵심 엔터티와 참조 관계, 동일 방향 friendship 및 동일 tournament/user 중복 방지, 최근 경기·채팅 조회용 index를 제공합니다. |
| 보장하지 않는 것 | unordered friendship identity, 자기 자신과의 friendship 금지, tournament seed/capacity, status 값의 유효 범위, 경기 결과 idempotency는 보장하지 않습니다. |
| 후속 연결 | `4aa060c0b8df`가 row 타입과 mapper를 추가하고, `ffb0a8275a4f`·`3aa5958bb967`가 뒤늦게 데이터 무결성 제약을 보강합니다. |

#### 비교 기준

- parent 상태와 `0e850d24406e`의 diff를 먼저 비교합니다.
- 후속 관련 SHA `4aa060c0b8df`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.2. `4aa060c0b8df` — feat(db): 사용자와 세션 row schema 정의

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `4aa060c0b8df` |
| Importance | B |
| Tags | AUTH, PERSISTENCE |
| Source role | 초기 SQL column을 Kysely row type과 public/session model 변환 함수로 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/schema.ts`의 `UserTable`, `SessionTable`, `Database`, `UserRow`, `UserProjectionRow`를 SQL column과 대조합니다.
- `packages/db/src/rowMappers.ts`의 `toPublicUser`, `toSessionUser`가 snake_case를 camelCase로 바꾸고 어떤 column을 제외하는지 확인합니다.
- `Generated`, nullable `email`·`banned_at`, `Date`와 숫자 변환이 compile-time에서 어떻게 표현되는지 확인합니다.
- `online`이 row column이 아니라 mapper 인수라는 점과 public projection에서 email이 빠지는 이유를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `001_initial.sql`에는 column이 있었지만 TypeScript query 결과와 shared DTO 사이에 명시된 변환 지점이 없었습니다. |
| 해결하려던 문제 | SQL 이름과 애플리케이션 이름이 섞이고, public 응답이 email·내부 column을 우발적으로 노출할 위험이 있었습니다. |
| 핵심 결정 | Kysely table interface와 `Selectable` row type을 만들고 `toPublicUser`·`toSessionUser`를 유일한 초기 변환 지점으로 두었습니다. |
| 입력 → 상태 전이 → 출력 | `users` row → `UserProjectionRow` → `toPublicUser(online)` → public user, 또는 email을 더한 `SessionUser`로 변환됩니다. |
| ownership / lifetime / cleanup | DB driver가 row lifetime을 소유하고 mapper는 새 DTO 값을 반환합니다. `online` 상태는 저장 row가 아니라 호출자가 제공하는 일시적 정보입니다. |
| failure / rollback / retry | mapper는 입력을 runtime 검증하지 않습니다. 잘못된 driver shape나 유효하지 않은 enum은 TypeScript 타입만으로 실행 중 차단되지 않습니다. |
| 보장하는 것 | column 이름과 API 이름을 분리하고 public/session projection의 노출 범위를 compile-time에 고정합니다. |
| 보장하지 않는 것 | query 실행, repository lifecycle, backend parity, runtime schema validation은 아직 제공하지 않습니다. |
| 후속 연결 | `9277572765e7`가 이 타입을 사용하는 repository lifecycle을 만들고, Thread 3의 row-schema refactor가 projection을 canonical 형태로 정렬합니다. |

#### 비교 기준

- parent 상태와 `4aa060c0b8df`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `0e850d24406e`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `9277572765e7`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.3. `9277572765e7` — feat(db): 저장소 lifecycle 구성

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `9277572765e7` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source role | `AppRepository`와 PostgreSQL/memory factory를 통해 저장소 생성·종료 책임을 API에서 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `AppRepository`, `createPostgresRepository`, `createMemoryRepository`를 확인합니다.
- PostgreSQL factory가 `Pool`, `Kysely`, dialect를 어떤 순서로 만들고 closure가 무엇을 포획하는지 추적합니다.
- `PostgresRepository.close()`의 `db.destroy()`와 `pool.end()` 순서 및 오류 처리 범위를 확인합니다.
- 이 시점의 interface가 `close`, `ensureSeedData`만 포함하고 기능 parity를 아직 강제하지 않는다는 점을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | migration SQL과 row mapper는 있었지만 호출자가 Pool/Kysely 생성과 종료를 직접 알아야 했습니다. |
| 해결하려던 문제 | API가 concrete database 자원에 결합되면 memory test double을 같은 호출 계약으로 사용할 수 없고 종료 누락 위험도 커집니다. |
| 핵심 결정 | `AppRepository` interface와 두 factory를 도입해 backend 선택과 자원 구성을 factory 내부로 숨겼습니다. |
| 입력 → 상태 전이 → 출력 | `databaseUrl` → `Pool`/`Kysely` 생성 → `PostgresRepository` 반환 → 호출 종료 시 `close()`가 database 자원을 해제합니다. memory factory는 외부 자원 없는 구현을 반환합니다. |
| ownership / lifetime / cleanup | PostgreSQL repository가 Kysely와 Pool을 소유하며 repository를 만든 호출자가 `close()` 호출 책임을 가집니다. memory repository는 소유할 외부 handle이 없습니다. |
| failure / rollback / retry | 생성 뒤 호출자가 `close()`를 누락하면 pool lifetime이 남습니다. 이 SHA에는 공통 `try/finally` composition root나 운영 readiness가 없습니다. |
| 보장하는 것 | 호출자는 concrete backend를 몰라도 동일 lifecycle method를 사용하고, test에서 memory 구현으로 교체할 수 있습니다. |
| 보장하지 않는 것 | interface가 데이터 동작의 의미적 parity를 증명하지는 않으며, `ensureSeedData`가 schema migration과 seed를 아직 함께 취급합니다. |
| 후속 연결 | `fb516f723cdf`부터 실제 repository operation이 추가되고, Thread 2의 `f9bb622a1117`가 migration과 seed ownership을 분리합니다. |

#### 비교 기준

- parent 상태와 `9277572765e7`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `4aa060c0b8df`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `fb516f723cdf`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.4. `fb516f723cdf` — feat(db): 개발 사용자 seed 저장 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `fb516f723cdf` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | 개발용 identity seed와 handle 기반 upsert를 PostgreSQL·memory 구현에 함께 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `DevLoginInput`, 두 backend의 `ensureSeedData`, `upsertDevUser`를 비교합니다.
- PostgreSQL `insert ... on conflict (handle) do update ... returning *`가 갱신하는 column과 유지하는 identity를 확인합니다.
- `normalizeHandle`, deterministic email/display/avatar 생성과 고정 seed 목록을 확인합니다.
- memory `Map`의 key/value, UUID 생성, seed 반복 실행 시 중복 여부를 PostgreSQL 동작과 비교합니다.
- 전체 seed가 하나의 transaction으로 묶이지 않는다는 점을 failure 범위에 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | repository는 생성·종료만 지원했고 로컬 로그인이나 read model을 확인할 사용자 데이터가 없었습니다. |
| 해결하려던 문제 | 반복 가능한 개발 환경과 같은 handle로 재로그인할 때 동일 사용자를 갱신하는 identity 규칙이 필요했습니다. |
| 핵심 결정 | handle을 정규화한 뒤 PostgreSQL은 upsert하고 memory는 canonical map을 조회·갱신하며, `ensureSeedData`가 고정 개발 사용자를 준비합니다. |
| 입력 → 상태 전이 → 출력 | 개발 입력 → handle 정규화 → 기존 row/map 검색 → insert 또는 update → mapper를 거친 session user 반환입니다. |
| ownership / lifetime / cleanup | PostgreSQL row는 DB가, memory user는 repository의 `Map`이 소유합니다. 반환값은 projection이며 repository 종료 전까지 memory 원본이 유지됩니다. |
| failure / rollback / retry | 개별 upsert는 idempotent하지만 seed 전체가 하나의 transaction은 아닙니다. 중간 실패 후 재실행으로 수렴할 수는 있어도 이 SHA가 all-or-nothing을 보장하지 않습니다. |
| 보장하는 것 | 동일 handle 재실행이 새 identity를 무한 생성하지 않고 두 backend에서 개발 로그인 입력을 처리할 수 있습니다. |
| 보장하지 않는 것 | production에서 자동 seed가 안전하다는 보장, profile별 데이터 분리, 역할 부여의 보안성, 두 backend의 모든 정렬·limit parity는 없습니다. |
| 후속 연결 | Thread 2의 `8da6edef28eb`가 seed profile을 나누고 `e1a0316fbe84`가 API startup의 암시적 seed를 제거합니다. |

#### 비교 기준

- parent 상태와 `fb516f723cdf`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `9277572765e7`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `c5b96a06925c`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.5. `c5b96a06925c` — feat(db): 프로필 조회와 변경 저장 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `c5b96a06925c` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | identity 저장소를 profile lookup/update와 사용자 목록 read model로 확장합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppRepository`에 추가된 `getUserById`, `getUserByHandle`, `updateProfile`, `listOnlineUsers` signature를 확인합니다.
- PostgreSQL `update ... returning *`와 update 전 사용자 조회의 failure path를 확인합니다.
- PostgreSQL `status = 'active'`, rating 정렬, `limit 12`와 memory 구현의 filter/limit 차이를 비교합니다.
- public mapper가 email을 제외하고 `online`을 어떤 값으로 채우는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 개발 로그인은 가능했지만 다른 사용자를 조회하거나 자기 표시 이름·avatar를 갱신할 repository operation이 없었습니다. |
| 해결하려던 문제 | HTTP profile·lobby가 persistence 내부 SQL을 직접 작성하지 않고 identity-bound read/write를 수행해야 했습니다. |
| 핵심 결정 | ID/handle lookup, profile update, active-user 목록을 `AppRepository`에 올리고 두 backend에 구현했습니다. |
| 입력 → 상태 전이 → 출력 | 식별자 또는 handle → row 검색 → public projection 반환; profile patch → 대상 확인 → row/map 변경 → 갱신 projection 반환입니다. |
| ownership / lifetime / cleanup | repository가 원본 row/map 변경을 소유하고 호출자는 반환 projection만 소비합니다. memory update는 저장된 객체를 직접 바꿉니다. |
| failure / rollback / retry | 대상이 없으면 명시적 오류를 던집니다. 이 SHA에서 PostgreSQL은 active·limit 12를 적용하지만 memory 목록은 같은 filter/limit를 완전히 재현하지 않습니다. |
| 보장하는 것 | profile 접근과 변경이 repository 경계에 모이고 SQL column이 API route로 직접 새지 않습니다. |
| 보장하지 않는 것 | 실제 WebSocket presence, 두 backend의 완전한 목록 parity, optimistic concurrency, runtime 입력 검증은 보장하지 않습니다. |
| 후속 연결 | `0364c42f776b`·`c7ea1ff241c8`가 read model을 확장하고, 실제 online authority는 이후 realtime subsystem으로 이동합니다. |

#### 비교 기준

- parent 상태와 `c5b96a06925c`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `fb516f723cdf`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `0364c42f776b`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.6. `0364c42f776b` — feat(db): 순위 조회 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `0364c42f776b` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | rating과 승수에서 leaderboard projection을 계산하는 repository read model을 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 두 backend의 `listLeaderboard()` 정렬 key, limit, `rank` 생성 방식을 비교합니다.
- `percentage(wins, losses)`의 zero-total 처리와 소수점 한 자리 반올림을 확인합니다.
- PostgreSQL `order by rating desc, wins desc limit 20`과 memory sort의 동점·limit 차이를 기록합니다.
- `LeaderboardEntry`로 변환할 때 public user와 계산 field의 소유 위치를 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | profile 목록은 있었지만 순위, 승률, 표시 순서를 일관된 DTO로 제공하지 못했습니다. |
| 해결하려던 문제 | UI가 backend별 raw users를 받아 자체 정렬·계산하면 PostgreSQL과 memory 결과가 갈라질 수 있었습니다. |
| 핵심 결정 | `listLeaderboard()`가 rating·wins 순으로 정렬하고 rank와 승률을 repository에서 계산하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | 사용자 row 집합 → rating/wins 정렬 → index 기반 rank와 `percentage` 계산 → leaderboard entry 배열입니다. |
| ownership / lifetime / cleanup | 저장된 rating/wins는 repository가 소유하고, rank/winRate는 호출 시 생성되는 read-model 값입니다. |
| failure / rollback / retry | 빈 전적은 0%로 처리합니다. 동점의 추가 deterministic key와 memory의 `limit 20` parity는 이 SHA에서 완전하지 않습니다. |
| 보장하는 것 | leaderboard 계산 위치와 기본 정렬 규칙이 두 backend 계약에 포함됩니다. |
| 보장하지 않는 것 | rating 변경의 atomicity, 전체 시즌 범위, 안정적인 동점 순서, pagination은 보장하지 않습니다. |
| 후속 연결 | `6509e32ba95d`가 memory 흐름에서 결과를 간접 확인하고, rating finalization 자체는 별도의 persistence story에서 강화됩니다. |

#### 비교 기준

- parent 상태와 `0364c42f776b`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `c5b96a06925c`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `8ab49e5f2dd4`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.7. `8ab49e5f2dd4` — feat(db): 경기 조회 row contract 정의

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `8ab49e5f2dd4` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | persisted match와 joined handle을 `MatchSummary`로 바꾸는 typed row boundary를 정의합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/schema.ts`의 `MatchTable`, `MatchRow`, `MatchWithHandlesRow`를 initial SQL과 대조합니다.
- `packages/db/src/rowMappers.ts`의 `toMatchSummary(row, userId)`가 승패와 상대 handle을 어떻게 결정하는지 확인합니다.
- loser의 `ratingDelta`가 `-12`, AI 상대가 문자열 `AI`로 표현되는 초기 가정을 기록합니다.
- `ended_at.toISOString()`과 nullable winner/loser 처리 경계를 확인합니다.
- 이 SHA에는 실제 recent-match query가 아직 없다는 점을 분리해서 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `matches` table은 존재했지만 joined query 결과와 공개 경기 요약 사이에 타입·변환 계약이 없었습니다. |
| 해결하려던 문제 | viewer-relative 승패, 상대 식별, timestamp 형식을 각 caller가 임의로 계산하면 read model이 일관되지 않습니다. |
| 핵심 결정 | `MatchTable`·`MatchWithHandlesRow`와 `toMatchSummary`를 추가해 DB row에서 `MatchSummary`로의 변환을 고정했습니다. |
| 입력 → 상태 전이 → 출력 | match row + winner/loser handle + 선택적 viewer ID → 상대/승패/rating delta 계산 → ISO timestamp를 가진 summary입니다. |
| ownership / lifetime / cleanup | DB가 raw match를 소유하고 mapper가 일회성 public summary를 소유합니다. viewer ID는 저장 상태를 바꾸지 않습니다. |
| failure / rollback / retry | nullable participant는 AI fallback으로 처리됩니다. 실제 query가 잘못된 join shape를 반환하면 mapper 자체는 runtime 검증하지 않습니다. |
| 보장하는 것 | 경기 row의 snake_case와 공개 summary의 camelCase·viewer-relative 의미를 한 함수에 모읍니다. |
| 보장하지 않는 것 | 최근 경기 범위, 정렬, dashboard 구성, 실제 rating history 기반 delta는 아직 보장하지 않습니다. |
| 후속 연결 | `c7ea1ff241c8`가 query와 dashboard를 연결하고 Thread 3의 row-mapper test가 이 변환을 직접 고정합니다. |

#### 비교 기준

- parent 상태와 `8ab49e5f2dd4`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `0364c42f776b`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `c7ea1ff241c8`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.8. `c7ea1ff241c8` — feat(db): 최근 경기와 대시보드 조회 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `c7ea1ff241c8` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | recent-match query와 dashboard aggregate를 두 backend에 처음 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `PostgresRepository.listRecentMatches`의 winner/loser join, 선택적 participant filter, `ended_at desc limit 8`을 확인합니다.
- `MemoryRepository.listRecentMatches`의 filter, tail/reverse 처리와 `memoryMatchSummary`를 비교합니다.
- 두 `getDashboard`가 profile, recent matches, win rate, `bestStreak`를 어떻게 조립하는지 확인합니다.
- PostgreSQL의 임의 수식과 memory 상수로 만든 `bestStreak`가 실제 match sequence에서 파생되지 않는 결함을 명시합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | match mapper는 있었지만 query와 dashboard aggregate가 없었고, UI가 profile과 경기 목록을 따로 조립해야 했습니다. |
| 해결하려던 문제 | viewer-scoped 최근 경기와 요약 지표를 한 repository read operation으로 제공해야 했습니다. |
| 핵심 결정 | PostgreSQL은 participant 조건이 있는 join query, memory는 저장 배열 filter를 사용하고 둘 다 최근 8개를 역시간순으로 반환하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | `userId` → 사용자 조회 → 최근 경기 8개 → win rate와 bestStreak 계산 → `DashboardSummary` 반환입니다. |
| ownership / lifetime / cleanup | repository가 query와 aggregate 조립을 소유합니다. 반환 배열은 read model이며 underlying rows/map의 lifetime과 분리됩니다. |
| failure / rollback / retry | 사용자가 없으면 실패합니다. `bestStreak`는 PostgreSQL의 `wins-losses+3` 수식과 memory 상수 3이라 실제 sequence와 무관합니다. |
| 보장하는 것 | 최근 경기 정렬·limit와 dashboard 조립 위치가 공통 interface에 들어갑니다. |
| 보장하지 않는 것 | 이 SHA만으로 backend parity나 실제 최고 연승의 정확성은 보장하지 않습니다. optional SQL fragment도 이후 명시적 두 query로 정리됩니다. |
| 후속 연결 | `035b97ca7c58`가 실제 경기 순서에서 streak를 계산하고 `6b661420e060`이 회귀를 고정합니다. |

#### 비교 기준

- parent 상태와 `c7ea1ff241c8`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `8ab49e5f2dd4`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `6509e32ba95d`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.9. `6509e32ba95d` — test(db): 메모리 저장소 흐름 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `6509e32ba95d` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT, TEST |
| Source role | memory backend의 주요 repository operation이 한 계약으로 조합되는지 초기 행동 시험을 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 seed·upsert·session·match·dashboard 시나리오를 순서대로 추적합니다.
- friendship과 tournament 생성·참가 시나리오가 당시의 단순 memory 표현을 어떤 기대값으로 고정하는지 확인합니다.
- 시험이 `createMemoryRepository()`만 사용하며 PostgreSQL·concurrency·resource close를 실행하지 않는다는 점을 분리합니다.
- assertion이 개별 mapper보다 public repository output을 관찰한다는 점을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 두 backend에 operation이 늘었지만 memory 구현의 기본 흐름을 실행하는 회귀 시험이 없었습니다. |
| 해결하려던 문제 | seed 이후 identity/session/match/dashboard 및 social/tournament method가 함께 호출될 때 계약이 깨지지 않는지 확인해야 했습니다. |
| 핵심 결정 | Vitest에서 public `AppRepository` method만 호출하는 memory behavioral test를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | memory repository 생성 → seed/upsert → session과 match 생성 → dashboard 조회, 별도 friendship·tournament 흐름 → public 결과 assertion입니다. |
| ownership / lifetime / cleanup | 각 test가 repository를 지역 소유하며 외부 자원 cleanup은 필요 없습니다. 내부 Map/배열은 repository lifetime 동안 유지됩니다. |
| failure / rollback / retry | 동기적 memory 구현이라 DB 오류·transaction rollback·실제 병렬 경쟁 상태는 재현하지 않습니다. |
| 보장하는 것 | 초기 common interface가 호출 가능한 상태이고 핵심 happy path의 결과 shape가 유지됨을 보여 줍니다. |
| 보장하지 않는 것 | PostgreSQL schema/migration, SQL mapper, pool cleanup, 실제 concurrency나 backend parity 전체를 증명하지 않습니다. |
| 후속 연결 | `e935054ce0c9`·`c43b87694b29`가 실제 PostgreSQL integration 환경과 lifecycle evidence를 추가합니다. |

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | memory repository의 공통 operation이 순서대로 조합되고 public DTO를 반환한다는 계약입니다. |
| 재현한 실패·경계 | 비어 있는 memory 상태에서 seed 후 사용자·세션·경기·dashboard, friendship·tournament를 생성하는 초기 경계입니다. |
| 시험 기법 | 외부 의존성 없는 behavioral integration test입니다. |
| 통과하는 실제 코드 경로 | `createMemoryRepository` → `ensureSeedData`/`upsertDevUser`/session·match·dashboard 및 social/tournament methods입니다. |
| 시험이 증명하는 것 | memory backend의 해당 happy path와 method surface가 작동함을 증명합니다. |
| 시험이 증명하지 않는 것 | PostgreSQL SQL, transaction, migration, pool lifecycle, 다중 요청 경쟁 상태는 증명하지 않습니다. |
| 막으려는 회귀 | repository method 추가·refactor 중 memory happy path와 반환 shape가 조용히 깨지는 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `6509e32ba95d`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `c7ea1ff241c8`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `035b97ca7c58`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.10. `035b97ca7c58` — fix(db): 최근 경기에서 최고 연승 계산

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `035b97ca7c58` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | backend별 가짜 `bestStreak`를 실제 recent-match sequence에서 계산하는 공통 함수로 교체합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`에서 PostgreSQL 수식과 memory 상수가 제거된 parent diff를 확인합니다.
- `bestWinningStreak(matches)`가 역시간순 recent list를 복사·reverse하여 시간순으로 순회하는 이유를 확인합니다.
- win에서 current/best를 증가시키고 loss에서 current를 0으로 되돌리는 상태 전이를 추적합니다.
- 계산 범위가 전체 이력이 아니라 `listRecentMatches`가 반환한 최대 8경기라는 non-guarantee를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `c7ea1ff241c8`은 PostgreSQL에서 임의 수식, memory에서 상수 3을 반환해 동일 경기 이력도 실제 연승과 무관했습니다. |
| 해결하려던 문제 | dashboard가 persistence evidence가 아닌 fabricated metric을 표시하고 backend마다 다른 값을 낼 수 있었습니다. |
| 핵심 결정 | 두 backend가 같은 `bestWinningStreak(recentMatches)`를 호출하도록 하고 역시간순 배열을 시간순으로 순회하는 작은 상태 계산기를 추가했습니다. |
| 입력 → 상태 전이 → 출력 | 최근 경기 newest-first 배열 복사 → reverse → win이면 current 증가·best 갱신, loss면 current=0 → best 반환입니다. |
| ownership / lifetime / cleanup | 원본 `recentMatches` 배열은 spread로 복사해 변형하지 않습니다. 계산 함수가 파생 지표를 소유하고 repository는 결과만 dashboard에 넣습니다. |
| failure / rollback / retry | 빈 배열은 0을 반환합니다. `draw` 같은 값은 현재 `MatchSummary.result` domain에 없으며, 경기 목록 조회 실패는 상위로 전파됩니다. |
| 보장하는 것 | PostgreSQL과 memory가 동일한 실제 match-result sequence에서 같은 최고 연승을 계산합니다. |
| 보장하지 않는 것 | 최대 8개 최근 경기 밖의 더 긴 연승, 시즌 전체 지표, DB-side 계산은 보장하지 않습니다. |
| 후속 연결 | `6b661420e060`이 win/loss reset과 ordering을 deterministic memory regression으로 검증합니다. |

#### 최소 코드 근거

- `packages/db/src/index.ts::bestWinningStreak` — `[...matches].reverse()`로 newest-first read model을 chronological order로 바꾼 뒤 loss에서 연속값을 초기화합니다.

#### 비교 기준

- parent 상태와 `035b97ca7c58`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `6509e32ba95d`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `6b661420e060`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.11. `6b661420e060` — test(db): 최고 연승 계산 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `6b661420e060` |
| Importance | B |
| Tags | PERSISTENCE, TEST |
| Source role | `bestWinningStreak`의 order와 loss reset을 public dashboard 경로에서 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 `derives the best streak from recent match results` 시나리오를 확인합니다.
- 생성 순서 `win → loss → win → win`과 조회 결과 `win, win, loss, win`의 reverse chronology를 대조합니다.
- `dashboard.bestStreak === 2`가 loss reset과 newest-first 처리 둘 다 필요로 한다는 점을 설명합니다.
- 시험이 memory backend만 실행하며 8경기 window 밖의 streak는 다루지 않는다는 점을 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `035b97ca7c58`가 계산기를 추가했지만 역순 처리나 loss reset이 다시 깨져도 이를 탐지할 집중 시험이 없었습니다. |
| 해결하려던 문제 | recent list가 newest-first이므로 단순 순회하면 시간 방향을 잘못 해석할 수 있고, loss 뒤 current를 초기화하지 않으면 3으로 계산될 수 있었습니다. |
| 핵심 결정 | 실제 repository로 네 경기 결과를 저장하고 dashboard의 경기 순서와 `bestStreak`를 함께 assertion했습니다. |
| 입력 → 상태 전이 → 출력 | seed·두 사용자 생성 → 네 경기 저장 → dashboard 조회 → 결과 배열이 newest-first인지 확인 → 최고 연승 2 확인입니다. |
| ownership / lifetime / cleanup | 시험 repository가 저장 순서와 파생 지표 계산을 소유하며 test 종료 뒤 memory state는 폐기됩니다. |
| failure / rollback / retry | 외부 오류나 retry는 주입하지 않습니다. assertion 실패가 계산·정렬 회귀를 직접 드러냅니다. |
| 보장하는 것 | public dashboard 경로에서 order 변환과 loss reset이 결합되어 기대값 2를 만든다는 것을 증명합니다. |
| 보장하지 않는 것 | PostgreSQL query 자체, 8개를 넘는 history, 전체 시즌 streak는 증명하지 않습니다. |
| 후속 연결 | 후속 row-mapping·query refactor가 dashboard 의미를 바꾸지 않도록 보호합니다. |

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | recent-match ordering을 chronological streak 계산으로 바꾸고 loss에서 연속 승리를 끊는 불변식입니다. |
| 재현한 실패·경계 | newest-first 조회와 `win → loss → win → win` 저장 순서의 조합입니다. |
| 시험 기법 | memory repository를 통과하는 deterministic regression test입니다. |
| 통과하는 실제 코드 경로 | `createMatch` → `listRecentMatches` → `getDashboard` → `bestWinningStreak`입니다. |
| 시험이 증명하는 것 | 결과 순서와 최고 연승 2를 함께 확인하므로 역순·reset 두 조건을 검증합니다. |
| 시험이 증명하지 않는 것 | PostgreSQL 실행 계획, 전체 history 범위, concurrent match write는 증명하지 않습니다. |
| 막으려는 회귀 | fabricated constant/수식으로 되돌아가거나 recent 배열 방향을 잘못 순회하는 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `6b661420e060`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `035b97ca7c58`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `e935054ce0c9`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.12. `e935054ce0c9` — build(db): PostgreSQL integration 의존성과 명령 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `e935054ce0c9` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | unit suite와 실제 PostgreSQL integration suite를 분리하고 Testcontainers 실행 경계를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- root `package.json`의 `postgres-integration` script가 DB package command에 위임하는지 확인합니다.
- `packages/db/package.json`의 unit exclude pattern과 integration Vitest command의 timeout·worker·file-parallelism 설정을 확인합니다.
- `@testcontainers/postgresql` dev dependency와 lockfile 추가가 production dependency에 포함되지 않는지 확인합니다.
- 이 SHA는 실행 기반만 추가하고 실제 integration scenario는 다음 commit이라는 점을 분리합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | DB package의 `test`는 unit과 container-backed test의 비용·환경 차이를 구분할 실행 경계가 없었습니다. |
| 해결하려던 문제 | 실제 PostgreSQL을 쓰는 시험은 Docker, 긴 timeout, 순차 실행이 필요하며 일반 unit suite에 섞으면 불안정하거나 건너뛰기 쉽습니다. |
| 핵심 결정 | root/package script를 추가하고 unit은 `*.integration.test.ts`를 제외하며 integration command는 한 worker·no file parallelism과 확장 timeout을 사용하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | `pnpm postgres-integration` → `@pong-pong/db` 전용 Vitest command → Testcontainers 의존성을 사용할 준비가 됩니다. |
| ownership / lifetime / cleanup | 개발 의존성과 test process가 container lifetime을 소유합니다. production package runtime에는 이 도구를 요구하지 않습니다. |
| failure / rollback / retry | Docker daemon 부재, image pull 실패, timeout은 command 실패로 노출됩니다. 이 SHA 자체는 container cleanup 코드를 아직 추가하지 않습니다. |
| 보장하는 것 | unit과 real-PostgreSQL 검증을 명시적으로 선택·CI 연결할 수 있는 실행 경계를 제공합니다. |
| 보장하지 않는 것 | 어떤 schema·seed·cleanup 불변식이 검증되는지는 아직 보장하지 않으며 실제 통과 증거도 이 commit만으로 없습니다. |
| 후속 연결 | `c43b87694b29`가 isolated schema와 cleanup을 포함한 integration contract를 구현합니다. |

#### 비교 기준

- parent 상태와 `e935054ce0c9`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `6b661420e060`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `c43b87694b29`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.13. `c43b87694b29` — test(db): PostgreSQL integration 환경과 계약 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `c43b87694b29` |
| Importance | A |
| Tags | PERSISTENCE, RISK, TEST |
| Source role | 실제 PostgreSQL 16 container에서 migration·seed·schema 격리·cleanup 계약을 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/postgres.integration.test.ts`의 PostgreSQL 16-alpine container startup/stop lifecycle을 확인합니다.
- `withIsolatedDatabase`가 무작위 `test_<32hex>` schema와 `search_path` URL을 만들고 pool/repository를 등록·역순 cleanup하는 과정을 추적합니다.
- callback 오류가 있어도 cleanup을 실행하고, cleanup 오류가 원래 오류를 덮지 않도록 처리하는지 확인합니다.
- migration idempotence, development/demo seed idempotence와 profile 차이, sibling schema isolation assertion을 구분합니다.
- 실제 test command는 `e935054ce0c9`의 sequential integration command임을 연결합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory behavioral test만으로는 SQL 문법, Kysely migration table, PostgreSQL type conversion, pool/schema cleanup을 확인할 수 없었습니다. |
| 해결하려던 문제 | 한 test의 row가 다른 test에 남거나 callback 실패로 pool/container가 누수되면 결과가 비결정적이고 CI가 종료되지 않을 수 있었습니다. |
| 핵심 결정 | PostgreSQL 16 Testcontainer를 suite 단위로 띄우고 각 case마다 무작위 isolated schema를 생성해 migration·seed를 실행하며, 자원을 역순으로 정리하는 harness를 만들었습니다. |
| 입력 → 상태 전이 → 출력 | container 시작 → test별 schema/search_path 생성 → pool/repository 등록 → migration·seed·query assertion → callback 성공/실패와 무관하게 repository/pool close·schema drop → suite 종료 시 container stop입니다. |
| ownership / lifetime / cleanup | outer suite가 container를, `withIsolatedDatabase`가 schema와 열린 pool/repository 목록을 소유합니다. cleanup은 등록 역순으로 실행되어 의존 자원을 먼저 닫습니다. |
| failure / rollback / retry | callback이 던져도 cleanup은 계속됩니다. cleanup 오류는 수집하되 원래 callback 오류를 보존하고, container stop도 suite teardown에서 수행합니다. |
| 보장하는 것 | 실제 PostgreSQL에서 migration 재실행 안정성, seed profile idempotence, schema 격리, 실패 중 cleanup을 검증할 수 있는 기반을 제공합니다. |
| 보장하지 않는 것 | production traffic 규모, network partition, 모든 repository method parity, 이 환경에서의 실제 test 통과는 본 작업에서 증명하지 않았습니다. |
| 후속 연결 | Thread 2의 migration/readiness/reset 시험과 Thread 4·5의 concurrency integration test가 이 harness를 확장합니다. |

#### 최소 코드 근거

- `packages/db/src/postgres.integration.test.ts::withIsolatedDatabase` — schema·pool·repository를 test가 명시적으로 소유하고 callback 실패 뒤에도 역순 cleanup합니다.

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | migration·seed가 실제 PostgreSQL에서 반복 가능하고 test별 schema/resource가 서로 격리·정리된다는 불변식입니다. |
| 재현한 실패·경계 | callback 실패, 여러 pool/repository 등록, 같은 migration/seed 재실행, development/demo profile 차이입니다. |
| 시험 기법 | PostgreSQL 16 Testcontainers와 random schema `search_path`를 사용하는 integration/lifecycle test입니다. |
| 통과하는 실제 코드 경로 | `migrateDatabase`·`ensureSeedData`·repository query, `close`, pool end, schema drop, container stop입니다. |
| 시험이 증명하는 것 | SQL과 PostgreSQL driver를 실제로 통과하는 상태 전이 및 실패 시 cleanup 설계가 시험 코드상 검증됨을 보여 줍니다. |
| 시험이 증명하지 않는 것 | 부하·장애 복구·production 데이터 migration, 모든 backend operation parity는 증명하지 않습니다. 이 세션에서는 command를 실행하지 않았습니다. |
| 막으려는 회귀 | mock/memory에서는 보이지 않는 SQL 오류, schema 오염, 열린 handle 누수, seed 중복 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `c43b87694b29`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `e935054ce0c9`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.

## 6. 불변식 변화

| 단계 | 관련 SHA | 조사 초점 | 학습자 기록 |
| --- | --- | --- | --- |
| Durable model 도입 | `0e850d24406e` | row identity·FK lifetime·초기 unique/index의 범위를 구분합니다. | PostgreSQL이 UUID·timestamp·관계 row를 소유하게 되었지만 canonical friendship, seed/capacity, idempotent result 같은 고위험 불변식은 아직 schema 밖에 남았습니다. |
| Typed translation 경계 | `4aa060c0b8df` | DB row와 public/session DTO의 노출 범위를 추적합니다. | `UserProjectionRow`와 mapper가 snake_case·nullable field를 DTO로 바꾸며 public projection에서 email을 제외합니다. 다만 runtime validation은 하지 않습니다. |
| Backend lifecycle·surface | `9277572765e7` → `c7ea1ff241c8` | factory·close·operation 확장과 parity의 실제 범위를 기록합니다. | caller는 하나의 interface를 사용하지만 PostgreSQL filter/limit와 memory 배열 동작은 일부 달랐습니다. interface parity는 의미 parity가 아니라 교체 가능한 호출 surface만 제공합니다. |
| 파생 read-model correction | `035b97ca7c58` → `6b661420e060` | fabricated metric이 repository evidence 기반 계산으로 바뀌는 과정을 설명합니다. | backend별 수식/상수가 제거되고 동일 `recentMatches` sequence를 시간순으로 해석하는 계산기로 통일되었습니다. regression은 order와 loss reset을 함께 고정합니다. |
| 실제 DB evidence | `e935054ce0c9` → `c43b87694b29` | 실행 경계·격리·cleanup이 만드는 보장 범위를 구분합니다. | unit과 container suite가 분리되고 test마다 random schema와 명시적 cleanup owner를 둡니다. 실제 SQL·migration lifecycle을 시험할 수 있지만 production 장애·부하까지 증명하지는 않습니다. |

## 7. Failure → Fix → Test 관계

| 관계 | Failure / 이전 가정 | Fix / 결정 | Test / 근거 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 1 | `c7ea1ff241c8`: PostgreSQL은 임의 수식, memory는 상수 3으로 `bestStreak`를 생성했습니다. | `035b97ca7c58`: recent match 결과를 chronological order로 순회하는 공통 계산기를 추가했습니다. | `6b661420e060`: `win→loss→win→win`과 newest-first 반환을 이용해 기대값 2를 검증합니다. | 이 관계는 read model이 저장된 경기 evidence에서만 파생되어야 한다는 불변식을 복구합니다. test는 fabricated value뿐 아니라 reverse·reset 오류도 막습니다. |
| 2 | memory test만으로는 SQL·migration table·pool cleanup을 확인할 수 없었습니다. | `e935054ce0c9`: 전용 command와 Testcontainers dependency를 추가했습니다. | `c43b87694b29`: PostgreSQL 16, isolated schema, callback-failure cleanup을 구현했습니다. | 시험 실행 환경 자체를 먼저 독립시킨 뒤 실제 DB lifecycle을 검증합니다. harness의 존재와 이 세션에서의 실제 통과는 구분해야 합니다. |

## 8. Ownership·상태·책임 변화

| 구간 | 이전 소유자/표현 | 이후 소유자/표현 | 관련 SHA | 학습자 기록 |
| --- | --- | --- | --- | --- |
| Schema와 row | 애플리케이션 상태의 durable owner가 없었습니다. | PostgreSQL table/FK가 row identity와 관계 lifetime을 소유합니다. | `0e850d24406e` | DB가 durability를 맡지만 domain-level canonical identity와 capacity는 이후 제약·transaction이 보강해야 했습니다. |
| DB shape → DTO | caller가 raw column을 직접 해석할 수 있었습니다. | row mapper가 이름·nullable·공개 field 변환을 소유합니다. | `4aa060c0b8df`, `8ab49e5f2dd4` | mapper는 새 projection을 반환하고 raw row를 수정하지 않습니다. runtime 검증 책임은 별도 protocol 경계에 남습니다. |
| 자원 lifecycle | Pool/Kysely 생성·종료가 caller 세부사항이었습니다. | repository factory가 구성하고 caller가 `close()`를 호출합니다. | `9277572765e7` | 구성은 repository가, 종료 trigger는 composition root가 소유하는 분할 책임입니다. |
| Read-model 계산 | UI 또는 backend별 임의 계산으로 갈라질 수 있었습니다. | repository 공통 helper가 recent sequence에서 지표를 계산합니다. | `c7ea1ff241c8`, `035b97ca7c58` | 저장 원본은 DB/Map이 유지하고 파생 metric은 query 시 생성됩니다. 계산 범위는 recent 8개로 한정됩니다. |
| Integration resource | test 자원의 명시적 owner가 없었습니다. | suite와 helper가 container·schema·pool·repository cleanup을 소유합니다. | `c43b87694b29` | callback 오류와 teardown을 분리해 원래 실패를 보존하면서 열린 handle을 정리합니다. |

## 9. Thread 최종 상태

이 Thread의 최종 상태에서 `AppRepository`는 PostgreSQL과 memory를 같은 호출 surface 뒤에 배치하고, row mapper는 relational row를 shared read model로 변환합니다. profile·leaderboard·recent match·dashboard가 repository operation으로 제공되며 `bestStreak`는 최근 경기 결과에서 계산됩니다. 다만 interface만으로 완전한 backend 의미 parity를 증명하지 않으며, 실제 PostgreSQL 보장은 container integration test가 다루는 범위로 제한됩니다.

## 10. 최종 실행 흐름

1. 호출자가 factory로 PostgreSQL 또는 memory repository를 만들고 해당 repository lifetime의 종료 책임을 가집니다.
2. repository method가 DB query 또는 Map/배열 조회를 수행하고 raw row/record를 얻습니다.
3. row mapper가 snake_case, nullable 값, viewer-relative 의미를 shared DTO로 변환합니다.
4. `getDashboard`는 사용자와 newest-first 최근 경기 최대 8개를 조립합니다.
5. `bestWinningStreak`가 복사한 경기 배열을 시간순으로 순회해 loss에서 current를 초기화하고 best를 반환합니다.
6. memory test는 public method composition을, PostgreSQL integration harness는 migration·seed·schema/resource lifecycle을 서로 다른 범위에서 검증합니다.

## 11. 학습 완료 확인

- [x] 초기 SQL 제약과 나중에 보강된 불변식을 혼동하지 않고 설명할 수 있습니다.
- [x] mapper의 compile-time 보장과 runtime non-guarantee를 구분할 수 있습니다.
- [x] memory/PostgreSQL의 interface parity와 의미 parity 차이를 구체적인 filter·limit 사례로 설명할 수 있습니다.
- [x] `bestStreak` fix의 이전 가정, root cause, corrected invariant와 regression을 연결할 수 있습니다.
- [x] memory behavioral test와 PostgreSQL integration test의 증거 범위를 과장하지 않고 설명할 수 있습니다.

## 12. 실행 및 증거 기록

- 저장소 실행 시험: 실행하지 않았습니다.
- 이유: 로컬 `git clone --branch web/ft_transcendence --single-branch`가 DNS 해석 실패(`Could not resolve host: github.com`)로 중단되어 의존성을 포함한 실행 가능한 checkout을 만들 수 없었습니다.
- 코드 근거: 지정 브랜치의 source classification과 각 exact SHA의 GitHub commit diff를 확인했습니다. 따라서 본 문서의 시험 설명은 실제 시험 코드의 정적 검토 결과이며, 이 환경에서의 통과 결과가 아닙니다.
===== END FILE: 01-repository-abstraction-backend-parity-and-read-models.md =====

===== BEGIN FILE: 02-migration-seed-readiness-and-reset-lifecycle.md =====
# Development Thread 02 — Migration·seed·readiness·reset lifecycle

## 1. 학습 목표

- 초기 SQL 실행과 seed가 하나의 method에 묶인 상태에서 versioned migration·profile seed로 분리되는 ownership 변화를 재구성합니다.
- database 연결 가능성과 migration set의 current/pending/diverged 상태가 repository readiness로 결합되는 과정을 설명합니다.
- API startup의 암시적 seed mutation을 제거한 이유와 source-level regression guard의 한계를 구분합니다.
- 파괴적 test reset이 strict target resolver, transactional schema replacement, migration 재적용과 시험으로 보호되는 과정을 추적합니다.

## 2. 범위와 경계

- 포함: DB package migration asset/CLI, Kysely migration lifecycle, seed profile, migration-set inspection, repository readiness, startup seed removal, test reset guard·executor·test.
- 실제 역사 순서에 맞춰 `e1a0316fbe84`·`5cac4843fd9b`를 `113b3c422192` 이후가 아니라 그 앞에 배치합니다.
- 제외: HTTP liveness/readiness endpoint, Prometheus metric, graceful shutdown과 deployment orchestration은 operations category에서 다룹니다.
- 제외: role assignment, guest session, WebSocket ticket migration은 authentication category의 독립 데이터 전환입니다.
- reset은 test-only destructive lifecycle이며 production migration rollback 전략으로 일반화하지 않습니다.

## 3. 핵심 질문

- 왜 `migrate`와 `seed`가 같은 `ensureSeedData()`를 호출하는 상태가 잘못된 ownership입니까?
- applied migration이 bundled set의 prefix가 아닐 때 왜 단순 pending이 아니라 diverged입니까?
- repository readiness는 process liveness와 무엇이 다르고 어느 순간까지의 상태만 보장합니까?
- startup에서 seed를 제거한 뒤 필요한 데이터 준비 책임은 어디로 이동합니까?
- test reset guard는 어떤 입력을 DB 연결 전에 차단하며, drop/create와 migration이 왜 하나의 transaction이 아닙니까?

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags |
| ---: | --- | --- | :---: | --- |
| 1 | `1140fb868714` | `feat(db): migration 실행 경계 구성` | B | PERSISTENCE |
| 2 | `dea169d587a3` | `feat(db): 데이터베이스 CLI 명령 연결` | B | PERSISTENCE |
| 3 | `f9bb622a1117` | `refactor(db): SQL migration lifecycle 분리` | A | PERSISTENCE, RISK, REFACTOR |
| 4 | `8da6edef28eb` | `feat(db): 환경별 seed profile 분리` | B | AUTH, REALTIME, PERSISTENCE |
| 5 | `981ee655559b` | `refactor(db): migration과 seed CLI 연결` | B | PERSISTENCE |
| 6 | `30aac132e14e` | `feat(db): migration set 상태 검사 추가` | A | PERSISTENCE, OPERATIONS, RISK |
| 7 | `2f05d5d79c64` | `feat(db): repository readiness 경계 추가` | A | PERSISTENCE, OPERATIONS |
| 8 | `e1a0316fbe84` | `fix(api): startup seed 생성을 제거` | B | PERSISTENCE |
| 9 | `5cac4843fd9b` | `test(api): startup seed 금지 검증` | B | REALTIME, TEST |
| 10 | `113b3c422192` | `feat(db): test database reset target guard 추가` | A | PERSISTENCE |
| 11 | `434403a7c16a` | `feat(db): test schema reset과 migration 실행 연결` | A | PERSISTENCE, RISK |
| 12 | `527b5f137425` | `test(db): test database reset guard 검증` | B | PERSISTENCE, TEST |

## 5. Commit별 조사

### 5.1. `1140fb868714` — feat(db): migration 실행 경계 구성

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `1140fb868714` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | 초기 SQL 파일을 DB package가 읽고 실행할 수 있는 importable schema-initialization 경계로 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/migrations.ts`와 package export가 `001_initial.sql`을 runtime에서 찾는 방식을 확인합니다.
- `initialMigrationSql`을 사용하는 caller가 파일 경로·module URL에 의존하는 범위를 확인합니다.
- 이 시점에는 migration history table이나 개별 version 적용 상태가 없고, 하나의 초기 SQL payload라는 점을 기록합니다.
- package build/runtime에서 SQL asset이 존재해야 한다는 non-guarantee를 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `001_initial.sql`은 repository에 있었지만 TypeScript runtime이 이를 import·실행할 안정된 package 경계가 없었습니다. |
| 해결하려던 문제 | repository 초기화 코드가 임의 경로로 SQL을 읽으면 실행 위치나 package 소비 방식에 따라 schema setup이 깨질 수 있었습니다. |
| 핵심 결정 | DB package 내부에서 초기 migration SQL을 읽어 export하고 이후 repository/CLI가 그 값을 사용하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | module-relative SQL asset 조회 → `initialMigrationSql` 로드 → repository 초기화 path가 SQL을 execute합니다. |
| ownership / lifetime / cleanup | DB package가 SQL asset의 위치와 읽기 책임을 소유합니다. 실행 caller는 반환된 SQL을 소비하지만 version set을 소유하지 않습니다. |
| failure / rollback / retry | asset 누락·경로 오류·SQL 실행 오류는 호출자에게 전파됩니다. 적용된 migration 목록이나 rollback은 없습니다. |
| 보장하는 것 | schema initialization SQL을 API와 CLI가 같은 package 경계에서 사용할 수 있습니다. |
| 보장하지 않는 것 | versioned migration lifecycle, pending/diverged 판별, seed와 schema evolution 분리는 보장하지 않습니다. |
| 후속 연결 | `dea169d587a3`가 CLI에 연결하고 `f9bb622a1117`가 단일 SQL payload를 Kysely file migration lifecycle로 대체합니다. |

#### 비교 기준

- parent 상태와 `1140fb868714`의 diff를 먼저 비교합니다.
- 후속 관련 SHA `dea169d587a3`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.2. `dea169d587a3` — feat(db): 데이터베이스 CLI 명령 연결

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `dea169d587a3` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | `migrate`, `seed`, memory smoke를 DB package CLI에 노출하지만 당시 migration과 seed는 같은 method를 호출합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/cli.ts`의 command parsing과 unknown-command failure를 확인합니다.
- `migrate`와 `seed`가 모두 repository의 `ensureSeedData()`를 호출하는 parent 상태를 비교합니다.
- `try/finally` 또는 종료 경로에서 repository `close()`가 항상 실행되는지 확인합니다.
- `packages/db/package.json` scripts가 각 command를 어떻게 노출하는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | schema와 seed를 준비하는 repository method는 있었지만 개발자·운영 script가 호출할 명시적 CLI가 없었습니다. |
| 해결하려던 문제 | 수동 코드 호출 대신 반복 가능한 command가 필요했지만, 당시 `ensureSeedData`가 DDL과 seed를 동시에 소유해 두 command의 의미가 같았습니다. |
| 핵심 결정 | command parser를 추가하고 `migrate`, `seed`, `memory-smoke`를 package script에 연결했습니다. |
| 입력 → 상태 전이 → 출력 | CLI 인수 해석 → URL/backend repository 생성 → `ensureSeedData()` 또는 smoke operation → `close()`입니다. |
| ownership / lifetime / cleanup | CLI process가 repository lifetime과 종료를 소유합니다. schema/seed의 실제 변경은 repository method가 수행합니다. |
| failure / rollback / retry | 잘못된 command와 DB 오류는 non-zero failure로 전파됩니다. `migrate`만 실행해도 seed가 생성되는 의미적 결합이 남아 있습니다. |
| 보장하는 것 | 개발자가 DB 준비와 smoke를 명시적 package command로 실행할 수 있습니다. |
| 보장하지 않는 것 | migration과 seed의 분리, environment profile, applied-version 기록, readiness는 보장하지 않습니다. |
| 후속 연결 | `f9bb622a1117`가 잘못 결합된 ownership을 분리하고 `981ee655559b`가 CLI 의미를 다시 연결합니다. |

#### 비교 기준

- parent 상태와 `dea169d587a3`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `1140fb868714`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `f9bb622a1117`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.3. `f9bb622a1117` — refactor(db): SQL migration lifecycle 분리

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `f9bb622a1117` |
| Importance | A |
| Tags | PERSISTENCE, RISK, REFACTOR |
| Source role | schema evolution을 Kysely `Migrator`와 SQL file set이, seed를 repository가 소유하도록 책임을 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/migrator.ts`의 `FileMigrationProvider`, `Migrator`, migration directory resolution을 확인합니다.
- `packages/db/src/index.ts`에서 `ensureSeedData()`가 schema DDL 실행을 더 이상 수행하지 않는 parent diff를 확인합니다.
- 기존 `initialMigrationSql` import/asset 경로가 제거되거나 대체되는 범위를 확인합니다.
- migration failure result와 thrown error가 caller에게 어떻게 전달되는지 확인합니다.
- repository seed가 이미 migration된 schema를 전제로 하게 된 새로운 precondition을 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `migrate`와 `seed`가 같은 `ensureSeedData()`를 실행해 schema version과 데이터 준비의 책임이 섞여 있었습니다. |
| 해결하려던 문제 | 운영 startup·CI가 schema만 적용하거나 seed만 선택할 수 없고, migration history와 순서를 독립적으로 추적할 수 없었습니다. |
| 핵심 결정 | SQL 파일을 Kysely `FileMigrationProvider`와 `Migrator`에 맡기고, repository의 `ensureSeedData`는 데이터 upsert만 수행하도록 바꿨습니다. |
| 입력 → 상태 전이 → 출력 | `migrateDatabase(url)` → migration directory의 version file discovery → Kysely migration table과 비교 → pending SQL 순차 적용; seed command는 별도 repository path를 사용합니다. |
| ownership / lifetime / cleanup | Migrator가 schema evolution 순서와 applied state를 소유하고 repository가 seed 데이터 의미를 소유합니다. caller는 두 lifecycle을 명시적으로 조합합니다. |
| failure / rollback / retry | migration이 실패하면 이후 seed를 자동 실행하지 않으며 오류가 전파됩니다. repository method를 migration 없이 호출하면 table 부재로 실패할 수 있습니다. |
| 보장하는 것 | schema evolution과 seed 데이터 생성이 서로 독립적으로 실행·감사될 수 있고 applied migration history가 생깁니다. |
| 보장하지 않는 것 | bundled/applied set divergence 판별, readiness 노출, destructive reset 안전성은 아직 보장하지 않습니다. |
| 후속 연결 | `981ee655559b`가 CLI를 새 lifecycle에 맞추고 `30aac132e14e`가 migration set 일치 여부를 명시적으로 분류합니다. |

#### 최소 코드 근거

- `packages/db/src/migrator.ts` — Kysely `FileMigrationProvider`/`Migrator`가 SQL file 순서와 applied migration state를 소유하고, repository seed path에서는 DDL 책임이 제거됩니다.

#### 비교 기준

- parent 상태와 `f9bb622a1117`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `dea169d587a3`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `8da6edef28eb`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.4. `8da6edef28eb` — feat(db): 환경별 seed profile 분리

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `8da6edef28eb` |
| Importance | B |
| Tags | AUTH, REALTIME, PERSISTENCE |
| Source role | `development`와 `demo` seed profile을 분리해 환경별 사용자·NPC 데이터 범위를 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `SeedProfile`과 `ensureSeedData(profile = 'development')` signature를 두 backend에서 확인합니다.
- development profile의 일반·관리 사용자와 demo profile의 NPC/체험 데이터 차이를 확인합니다.
- NPC rating band와 `is_npc` upsert가 반복 실행에서 identity를 유지하는지 확인합니다.
- profile 선택이 migration을 실행하지 않는다는 `f9bb622a1117` 이후 precondition을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | migration과 seed는 분리됐지만 seed 데이터가 하나의 고정 집합이라 개발용 권한 사용자와 demo용 상대가 섞였습니다. |
| 해결하려던 문제 | 환경에 따라 필요한 sample identity가 다른데 같은 seed를 적용하면 demo에서 불필요한 계정·권한이 생길 수 있었습니다. |
| 핵심 결정 | `development`·`demo` profile을 도입하고 두 backend의 `ensureSeedData`가 profile별 사용자/NPC 집합을 upsert하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | profile 입력 → 공통 NPC 및 profile-specific 사용자 선택 → handle 기준 upsert → 반복 실행 시 기존 identity 갱신입니다. |
| ownership / lifetime / cleanup | repository가 profile별 seed set과 memory map/DB row를 소유합니다. caller가 환경에 맞는 profile 선택 책임을 가집니다. |
| failure / rollback / retry | 잘못된 profile은 타입/CLI parsing에서 차단됩니다. 여러 upsert가 하나의 transaction으로 묶인다는 보장은 없습니다. |
| 보장하는 것 | development와 demo가 의도한 데이터 집합을 명시적으로 선택하고 반복 적용해도 handle 중복이 생기지 않습니다. |
| 보장하지 않는 것 | 권한 assignment의 보안 전체, production 자동 seed 허용, migration current 상태는 보장하지 않습니다. |
| 후속 연결 | `981ee655559b`가 `seed:dev`·`seed:demo` command로 연결하고 PostgreSQL integration test가 idempotence와 차이를 확인합니다. |

#### 비교 기준

- parent 상태와 `8da6edef28eb`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `f9bb622a1117`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `981ee655559b`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.5. `981ee655559b` — refactor(db): migration과 seed CLI 연결

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `981ee655559b` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | CLI command를 분리된 migration function과 profile-specific seed operation에 정확히 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/cli.ts`의 `migrate`, `seed:dev`, `seed:demo`, `memory-smoke` branch를 확인합니다.
- `migrate`가 repository `ensureSeedData`가 아니라 `migrateDatabase(databaseUrl)`을 직접 호출하는지 확인합니다.
- seed command의 repository 생성·profile 인수·`finally close()`를 추적합니다.
- usage text와 `packages/db/package.json` scripts가 command 의미와 일치하는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `f9bb622a1117`가 내부 책임을 분리했지만 CLI는 이전 command 의미를 그대로 두면 사용자에게 잘못된 동작을 노출할 수 있었습니다. |
| 해결하려던 문제 | `migrate`는 schema만, `seed:*`는 이미 migration된 schema의 데이터만 바꿔야 했습니다. |
| 핵심 결정 | command dispatch를 `migrateDatabase`, `ensureSeedData('development'\|'demo')`에 각각 연결하고 package scripts 이름을 맞췄습니다. |
| 입력 → 상태 전이 → 출력 | `migrate` → migrator 실행; `seed:dev\|demo` → repository 생성 → profile seed → `finally close`; smoke는 memory path를 사용합니다. |
| ownership / lifetime / cleanup | migration command는 migrator resource를, seed command는 repository lifecycle을 소유합니다. CLI caller가 실행 순서를 선택합니다. |
| failure / rollback / retry | seed를 migration 전에 실행하면 DB 오류가 발생합니다. 두 command를 하나의 all-or-nothing operation으로 묶지 않습니다. |
| 보장하는 것 | command 이름과 실제 mutation 범위가 일치하고 seed backend 자원이 항상 close되는 경로를 제공합니다. |
| 보장하지 않는 것 | 배포가 올바른 순서로 command를 실행한다는 보장, migration divergence/readiness는 아직 없습니다. |
| 후속 연결 | `30aac132e14e`와 `2f05d5d79c64`가 적용 상태를 판별하고 repository readiness로 노출합니다. |

#### 비교 기준

- parent 상태와 `981ee655559b`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `8da6edef28eb`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `30aac132e14e`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.6. `30aac132e14e` — feat(db): migration set 상태 검사 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `30aac132e14e` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS, RISK |
| Source role | bundled SQL migration 이름과 DB의 Kysely applied record를 비교해 current/pending/diverged를 구분합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/migrator.ts`의 bundled migration name discovery와 applied migration query를 확인합니다.
- current, pending, diverged를 결정하는 순서·prefix 조건과 unknown applied migration 처리 방식을 추적합니다.
- 파일 이름 정렬이 실제 migration order와 어떻게 연결되는지 확인합니다.
- DB 연결 실패와 migration set divergence가 서로 다른 failure로 전달되는지 확인합니다.
- 상태 검사가 migration을 적용하거나 seed를 변경하지 않는 read-only operation인지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | Kysely는 pending migration을 적용할 수 있었지만 서비스가 현재 binary의 migration set과 DB applied state가 같은지 명시적으로 설명하지 못했습니다. |
| 해결하려던 문제 | DB가 연결돼도 pending file이 있거나 이미 적용된 migration이 bundle에서 사라진 diverged 상태면 API를 ready로 취급하면 안 됩니다. |
| 핵심 결정 | bundled file 이름과 applied record를 정렬·비교해 `current`, `pending`, `diverged` 상태를 계산하는 inspection function을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | migration directory 목록 읽기 + Kysely applied rows 조회 → ordered set 비교 → 동일하면 current, applied prefix 뒤 file이 남으면 pending, prefix가 깨지거나 unknown applied가 있으면 diverged입니다. |
| ownership / lifetime / cleanup | Migrator module이 migration vocabulary와 comparison logic을 소유합니다. 검사 caller는 결과를 정책으로 해석합니다. |
| failure / rollback / retry | DB query/파일 읽기 실패는 상태가 아니라 오류로 전파됩니다. divergence를 자동 수정하거나 down migration하지 않습니다. |
| 보장하는 것 | 연결 가능성과 별개로 binary/DB schema set의 합치·미적용·분기를 구분할 수 있습니다. |
| 보장하지 않는 것 | migration 내용의 semantic 호환성, production data migration 안전성, 자동 복구는 보장하지 않습니다. |
| 후속 연결 | `2f05d5d79c64`가 이 inspection을 repository readiness contract에 포함하고 운영 health endpoint는 다른 카테고리에서 소비합니다. |

#### 최소 코드 근거

- `packages/db/src/migrator.ts` — bundled migration name sequence와 Kysely applied sequence를 비교하며, applied prefix가 깨지는 경우를 단순 pending이 아닌 `diverged`로 분류합니다.

#### 비교 기준

- parent 상태와 `30aac132e14e`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `981ee655559b`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `2f05d5d79c64`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.7. `2f05d5d79c64` — feat(db): repository readiness 경계 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `2f05d5d79c64` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS |
| Source role | storage 연결성과 migration set 상태를 concrete backend 밖의 `AppRepository` readiness contract로 올립니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppRepository`에 추가된 readiness method와 반환/오류 shape를 확인합니다.
- PostgreSQL 구현의 `select 1` 또는 동등한 connectivity probe와 migration set inspection 순서를 추적합니다.
- pending/diverged 상태가 ready 결과가 아니라 failure로 해석되는지 확인합니다.
- memory 구현이 외부 DB 없이 어떤 readiness 값을 반환하는지 비교합니다.
- API가 Pool/Kysely 세부사항을 알지 않아도 readiness를 확인할 수 있는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | migration 상태를 계산할 수 있었지만 API가 이를 사용하려면 DB package의 concrete migrator·pool 세부사항을 알아야 했습니다. |
| 해결하려던 문제 | liveness와 달리 ready는 query 가능하고 현재 binary와 schema가 일치해야 하며 backend 선택과 무관한 호출 계약이 필요했습니다. |
| 핵심 결정 | `AppRepository`에 readiness operation을 추가하고 PostgreSQL은 connectivity probe와 migration set current를 확인하며 memory는 즉시 준비 상태를 반환하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | API readiness 요청 → repository readiness → PostgreSQL query 성공 여부 → migration set inspection → current이면 ready 응답, 아니면 오류/비준비입니다. |
| ownership / lifetime / cleanup | repository가 backend-specific health probe와 migration interpretation을 소유하고 API는 결과만 소비합니다. |
| failure / rollback / retry | 연결 실패·pending·diverged는 ready로 숨기지 않습니다. probe와 실제 다음 business query 사이의 race는 제거하지 못합니다. |
| 보장하는 것 | 서비스가 storage implementation을 추측하지 않고 동일 interface에서 query 가능성과 schema current 상태를 검사할 수 있습니다. |
| 보장하지 않는 것 | 장기 transaction, replica lag, 모든 table의 semantic integrity, 이후 순간의 availability는 보장하지 않습니다. |
| 후속 연결 | `e1a0316fbe84`가 startup mutation을 제거해 readiness와 initialization을 더 분리하고, health endpoint 통합은 operations category에서 검증됩니다. |

#### 비교 기준

- parent 상태와 `2f05d5d79c64`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `30aac132e14e`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `e1a0316fbe84`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.8. `e1a0316fbe84` — fix(api): startup seed 생성을 제거

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `e1a0316fbe84` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | API process startup이 memory backend를 선택했다는 이유만으로 데이터를 암시적으로 생성하던 동작을 제거합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/index.ts` parent diff에서 `if (!env.databaseUrl) await repo.ensureSeedData()` 제거를 확인합니다.
- repository 생성, application build, listen, shutdown 흐름에서 seed mutation이 사라졌는지 확인합니다.
- 명시적 CLI seed와 startup composition root의 책임 차이를 기록합니다.
- memory startup이 빈 상태일 수 있다는 새 non-guarantee와 demo/guest 흐름의 별도 준비 책임을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | API startup은 `DATABASE_URL`이 없으면 memory repository에 자동 seed를 넣어 process boot와 data mutation을 결합했습니다. |
| 해결하려던 문제 | 재시작이 데이터 상태를 바꾸고, production/demo 설정 오류가 sample data로 가려지며, readiness가 initialization 성공처럼 오해될 수 있었습니다. |
| 핵심 결정 | composition root에서 `ensureSeedData()` 호출을 제거하고 seed는 DB CLI나 명시적 test setup만 수행하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | 환경 parse → repository 생성 → application 구성 → listen으로 진행하며 startup 중 seed write는 없습니다. |
| ownership / lifetime / cleanup | API process는 runtime lifecycle만 소유하고 seed data ownership은 명시적 DB command/test fixture로 돌아갑니다. |
| failure / rollback / retry | 필요한 seed를 별도 실행하지 않으면 memory read는 빈 결과를 반환할 수 있습니다. startup이 이를 보완하지 않습니다. |
| 보장하는 것 | process boot가 암시적으로 사용자·NPC 데이터를 생성하지 않는 fail-transparent startup을 제공합니다. |
| 보장하지 않는 것 | 배포 orchestration이 migration/seed를 올바르게 실행했다는 보장이나 source 밖 동적 호출 금지는 아직 없습니다. |
| 후속 연결 | `5cac4843fd9b`가 entrypoint source에 seed 호출이 다시 들어오는 것을 회귀로 막습니다. |

#### 비교 기준

- parent 상태와 `e1a0316fbe84`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `2f05d5d79c64`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `5cac4843fd9b`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.9. `5cac4843fd9b` — test(api): startup seed 금지 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `5cac4843fd9b` |
| Importance | B |
| Tags | REALTIME, TEST |
| Source role | API entrypoint가 `ensureSeedData`를 호출하지 않는다는 정적 회귀 guard를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/startup.test.ts`가 어떤 source file을 읽고 어떤 정규식/문자열을 금지하는지 확인합니다.
- 시험이 API process를 실제 시작하지 않고 source text를 검사한다는 점을 분리합니다.
- 같은 commit의 WebSocket smoke 변경이 seeded NPC 의존성을 제거하는지 확인합니다.
- alias·wrapper를 통한 간접 seed 호출은 탐지하지 못하는 non-guarantee를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `e1a0316fbe84`가 한 줄을 제거했지만 후속 refactor가 같은 startup path에 seed 호출을 다시 넣어도 집중 guard가 없었습니다. |
| 해결하려던 문제 | startup lifecycle과 데이터 준비 책임의 분리가 source 변경으로 쉽게 회귀할 수 있었습니다. |
| 핵심 결정 | entrypoint source를 읽어 `.ensureSeedData(` 호출이 없는지 assertion하고 smoke가 seed 전제 없이 진행되도록 조정했습니다. |
| 입력 → 상태 전이 → 출력 | test → `apps/api/src/index.ts` text load → 금지 pattern 검색 → 존재 시 실패; 별도 smoke는 AI/queue 경로를 seed 없는 조건에 맞춥니다. |
| ownership / lifetime / cleanup | 시험이 source contract를 소유하며 runtime repository state는 만들지 않습니다. |
| failure / rollback / retry | wrapper·computed property·다른 startup module의 간접 호출은 놓칠 수 있습니다. 실제 process boot나 DB mutation을 관찰하지 않습니다. |
| 보장하는 것 | 현재 entrypoint에 직접적인 startup seed call이 다시 들어오면 unit suite가 실패합니다. |
| 보장하지 않는 것 | 전체 call graph에서 모든 암시적 seed를 금지한다거나 production startup이 실제로 mutation-free라는 runtime 증거는 아닙니다. |
| 후속 연결 | 이후 destructive reset은 startup과 별개인 explicit test-only CLI로 설계됩니다. |

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | API startup과 seed lifecycle이 분리되어 entrypoint가 직접 `ensureSeedData`를 호출하지 않는다는 규칙입니다. |
| 재현한 실패·경계 | 후속 수정이 memory fallback 편의를 위해 seed call을 다시 삽입하는 source-level 회귀입니다. |
| 시험 기법 | source file text를 읽는 static contract test입니다. |
| 통과하는 실제 코드 경로 | `apps/api/src/startup.test.ts` → `apps/api/src/index.ts` source inspection입니다. |
| 시험이 증명하는 것 | 직접 method call pattern이 entrypoint에 없음을 증명합니다. |
| 시험이 증명하지 않는 것 | 간접 wrapper, 다른 module, 실제 runtime mutation이나 deployment command 순서는 증명하지 않습니다. |
| 막으려는 회귀 | process boot가 sample data 생성 책임을 다시 떠안는 회귀를 빠르게 막습니다. |

#### 비교 기준

- parent 상태와 `5cac4843fd9b`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `e1a0316fbe84`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `113b3c422192`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.10. `113b3c422192` — feat(db): test database reset target guard 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `113b3c422192` |
| Importance | A |
| Tags | PERSISTENCE |
| Source role | 파괴적 reset을 허용할 database/schema target을 strict environment·URL 규칙으로 fail-closed 판정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/testReset.ts`의 `resolveTestResetTarget`에서 `NODE_ENV=test`, `TEST_DATABASE_URL` 필수 조건을 확인합니다.
- PostgreSQL URL protocol, database name pattern, `options=-c search_path=...` parsing과 허용 개수를 추적합니다.
- public schema가 허용되는 dedicated test database 이름과 isolated `test_[a-f0-9]{32}` schema regex를 확인합니다.
- ambiguous option, 일반 DB 이름, 잘못된 schema가 모두 동일한 unsafe failure로 차단되는지 확인합니다.
- guard가 target만 반환하고 실제 drop/create를 아직 수행하지 않는다는 점을 분리합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | test isolation을 위해 schema reset이 필요했지만 잘못된 URL을 받으면 개발·운영 데이터까지 삭제할 수 있었습니다. |
| 해결하려던 문제 | `NODE_ENV=test` 한 조건이나 database 이름에 `test`가 포함된 정도로는 encoded path·search_path option을 안전하게 판별할 수 없었습니다. |
| 핵심 결정 | `resolveTestResetTarget`이 환경, protocol, database name, option count, exact schema pattern을 모두 검사하고 dedicated public test DB 또는 generated isolated schema만 반환하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | env 입력 → 필수값 검증 → URL parse/decoded DB name → options/search_path 해석 → strict allow-list match → 안전 target 반환, 하나라도 어긋나면 throw입니다. |
| ownership / lifetime / cleanup | guard module이 destructive target validation을 소유하고 reset executor는 검증된 target만 받아야 합니다. 아직 DB resource는 만들지 않습니다. |
| failure / rollback / retry | 일반 DB, production env, 여러 options, 임의 schema, ambiguous search_path는 fail-closed합니다. URL이 규칙을 만족해도 실제 server가 test 전용인지 외부에서 증명하지는 못합니다. |
| 보장하는 것 | 실수로 평범한 application DB/public schema를 reset command에 넘기는 주요 경로를 코드 수준에서 차단합니다. |
| 보장하지 않는 것 | DB 권한 오설정, DNS가 다른 server를 가리키는 경우, privileged attacker의 환경 조작까지 보장하지 않습니다. |
| 후속 연결 | `434403a7c16a`가 이 resolver 뒤에 실제 drop/create+migrate를 연결하고 `527b5f137425`가 거부·허용 경계를 검증합니다. |

#### 최소 코드 근거

- `packages/db/src/testReset.ts::resolveTestResetTarget` — dedicated test DB의 `public` 또는 정확한 `test_[a-f0-9]{32}` schema만 허용하고 그 외 URL/option은 모두 거부합니다.

#### 비교 기준

- parent 상태와 `113b3c422192`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `5cac4843fd9b`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `434403a7c16a`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.11. `434403a7c16a` — feat(db): test schema reset과 migration 실행 연결

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `434403a7c16a` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| Source role | 검증된 test target만 transaction으로 drop/create한 뒤 migration을 다시 적용하는 explicit `reset:test` CLI를 구현합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/testReset.ts`의 `resetTestDatabase`가 resolver를 먼저 호출하는지 확인합니다.
- control URL에서 options를 제거하고 Pool/client를 만든 뒤 `begin`/`drop schema`/`create schema`/`commit` 또는 `rollback`하는 순서를 추적합니다.
- identifier quote helper가 strict resolver 결과만 받는지 확인합니다.
- client release와 pool end가 success/failure 모두에서 실행되는지 확인합니다.
- schema transaction이 끝난 뒤 `migrateDatabase(target.databaseUrl)`가 별도 lifecycle로 실행되는 이유와 실패 상태를 기록합니다.
- `packages/db/src/cli.ts`와 package script의 `reset:test` 연결을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 안전 target을 판별할 수 있었지만 실제 reset과 migration 재적용을 일관되게 수행하는 command가 없었습니다. |
| 해결하려던 문제 | 수동 drop/create는 guard를 우회하거나 중간 실패 때 열린 client·부분 schema를 남길 위험이 있었습니다. |
| 핵심 결정 | `reset:test`가 resolver를 통과한 schema를 transaction에서 drop/create하고 자원을 정리한 뒤 같은 target URL로 migration을 실행하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | CLI → target resolve → control Pool/client → BEGIN → quoted schema DROP CASCADE → CREATE → COMMIT → client release/pool end → `migrateDatabase`로 current schema 재구성입니다. |
| ownership / lifetime / cleanup | reset function이 control connection과 transaction cleanup을 소유하고 migrator가 새 schema version 적용을 소유합니다. CLI caller는 explicit invocation만 소유합니다. |
| failure / rollback / retry | DDL 단계 오류면 ROLLBACK 후 자원을 닫습니다. migration은 commit 뒤 별도 단계이므로 migration 실패 시 빈/부분 적용 test schema가 남고 command가 실패합니다. |
| 보장하는 것 | 허용된 test target만 파괴하며 sibling schema를 건드리지 않고 reset 뒤 current migration set을 재적용할 수 있습니다. |
| 보장하지 않는 것 | drop/create와 전체 migration이 하나의 transaction이라는 보장, production target의 물리적 식별, seed 자동 복구는 없습니다. |
| 후속 연결 | `527b5f137425`가 pure guard matrix와 실제 PostgreSQL sibling-schema 보존·readiness를 검증합니다. |

#### 최소 코드 근거

- `packages/db/src/testReset.ts::resetTestDatabase` — 검증된 schema의 `DROP ... CASCADE`와 `CREATE`를 한 transaction에서 수행하고, commit 이후 별도 `migrateDatabase` lifecycle을 실행합니다.

#### 비교 기준

- parent 상태와 `434403a7c16a`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `113b3c422192`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `527b5f137425`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.12. `527b5f137425` — test(db): test database reset guard 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `527b5f137425` |
| Importance | B |
| Tags | PERSISTENCE, TEST |
| Source role | 파괴적 reset의 허용·거부 matrix와 실제 isolated-schema reset 효과를 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/testReset.test.ts`의 non-test env, 일반 DB, ambiguous options, invalid schema 거부 case를 확인합니다.
- dedicated test DB public schema와 generated isolated schema 허용 case를 확인합니다.
- `packages/db/src/postgres.integration.test.ts`의 sibling schema marker, target reset, users table empty, migration readiness current assertion을 추적합니다.
- 시험이 target schema 외 sibling을 보존하는지 direct query로 확인하는 부분을 찾습니다.
- Testcontainers/Docker가 필요한 integration 부분과 pure resolver unit 부분을 구분합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `113b3c422192`와 `434403a7c16a`가 guard/executor를 구현했지만 allow-list가 너무 넓거나 reset 범위가 잘못돼도 탐지할 시험이 필요했습니다. |
| 해결하려던 문제 | 특히 encoded/복수 option, 평범한 DB 이름, target과 sibling schema의 혼동이 destructive data loss로 이어질 수 있었습니다. |
| 핵심 결정 | pure resolver table test와 PostgreSQL isolated-schema integration test를 함께 추가했습니다. |
| 입력 → 상태 전이 → 출력 | unit: env/URL matrix → resolve success 또는 unsafe rejection; integration: target·sibling schema 생성 → target reset → migration/current 확인 → sibling marker 보존 확인입니다. |
| ownership / lifetime / cleanup | unit test는 문자열 규칙을, integration harness는 schema·pool lifecycle을 소유하고 기존 `withIsolatedDatabase` cleanup을 재사용합니다. |
| failure / rollback / retry | unsafe 입력은 DB 연결 전에 실패합니다. 실제 reset 실패는 command error로 전파되며 test harness가 자원을 정리합니다. |
| 보장하는 것 | 정의된 allow-list가 fail-closed이고 reset이 target schema만 교체한 뒤 migration current 상태를 만든다는 코드를 검증합니다. |
| 보장하지 않는 것 | 실제 production URL 오지정, 권한 escalation, 이 환경에서의 container test 통과는 증명하지 않습니다. |
| 후속 연결 | 이후 test DB reset은 explicit CLI로 유지되며 startup seed removal과 함께 process boot/data mutation 책임을 분리합니다. |

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | 파괴적 reset은 test-only allow-list target에만 적용되고 sibling schema를 보존하며 migration current 상태로 끝나야 합니다. |
| 재현한 실패·경계 | non-test env, 일반 DB, 복수/모호 option, invalid schema, 실제 sibling schema가 있는 PostgreSQL입니다. |
| 시험 기법 | pure decision-table unit test와 Testcontainers integration test의 조합입니다. |
| 통과하는 실제 코드 경로 | `resolveTestResetTarget` → `resetTestDatabase` → transaction DDL → `migrateDatabase` → readiness/query입니다. |
| 시험이 증명하는 것 | guard의 주요 거부 조건과 target-only reset·migration 재적용을 코드상 검증합니다. |
| 시험이 증명하지 않는 것 | 물리 server identity, credential compromise, production data restore, 본 세션에서의 실제 실행 성공은 증명하지 않습니다. |
| 막으려는 회귀 | allow-list 완화, schema quote 오류, sibling 삭제, reset 후 migration 누락 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `527b5f137425`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `434403a7c16a`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.

## 6. 불변식 변화

| 단계 | 관련 SHA | 조사 초점 | 학습자 기록 |
| --- | --- | --- | --- |
| 초기 executable SQL | `1140fb868714` → `dea169d587a3` | SQL asset과 CLI가 만든 최초 실행 경계 및 결합 문제를 기록합니다. | DB package가 초기 SQL을 제공하고 CLI가 이를 호출할 수 있게 됐지만 `migrate`와 `seed`가 모두 `ensureSeedData`를 실행해 schema와 data mutation이 구분되지 않았습니다. |
| Lifecycle 분리 | `f9bb622a1117` → `981ee655559b` | migrator·repository·CLI의 새 책임을 구분합니다. | Kysely Migrator가 versioned schema evolution을, repository가 profile seed를, CLI가 명시적 command dispatch와 resource close를 소유합니다. |
| Readiness 판정 | `30aac132e14e` → `2f05d5d79c64` | connection과 migration-set current 조건을 결합합니다. | DB query 가능성만으로 ready가 되지 않습니다. bundled/applied sequence가 정확히 current일 때만 repository readiness가 성공하고 pending/diverged는 비준비로 노출됩니다. |
| Startup mutation 제거 | `e1a0316fbe84` → `5cac4843fd9b` | process boot와 data initialization 책임 분리를 설명합니다. | API entrypoint는 seed를 만들지 않으며 explicit CLI/test setup이 데이터 준비를 소유합니다. source test는 직접 호출 회귀만 막고 전체 runtime call graph를 증명하지는 않습니다. |
| Destructive reset 보호 | `113b3c422192` → `527b5f137425` | fail-closed target, transaction, migration, sibling 보존을 추적합니다. | strict environment·URL·schema allow-list를 통과한 target만 transaction으로 drop/create되고 migration이 별도로 재적용됩니다. unit matrix와 real-PostgreSQL test가 거부 조건과 target-only effect를 연결합니다. |

## 7. Failure → Fix → Test 관계

| 관계 | Failure / 이전 가정 | Fix / 결정 | Test / 근거 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 1 | `dea169d587a3`: migrate와 seed가 같은 method를 호출해 schema와 sample data가 함께 변했습니다. | `f9bb622a1117`, `8da6edef28eb`, `981ee655559b`: Migrator·profile seed·CLI command를 분리했습니다. | `c43b87694b29`의 migration/seed idempotence와 profile integration evidence가 후속 보호 역할을 합니다. | 문제는 command 이름이 아니라 ownership 결합이었습니다. schema history는 migrator, 데이터 집합은 repository, 실행 순서는 caller가 소유하도록 나뉩니다. |
| 2 | DB 연결 성공만으로는 pending/diverged schema를 ready로 오인할 수 있었습니다. | `30aac132e14e`가 set 상태를 분류하고 `2f05d5d79c64`가 repository readiness에 포함했습니다. | HTTP health 통합은 operations category에서 검증되며 이 Thread는 repository-level 근거만 고정합니다. | ready는 현재 binary가 기대하는 migration sequence와 DB applied sequence가 맞는 순간의 조건입니다. 미래 availability나 semantic data correctness는 아닙니다. |
| 3 | API startup이 memory fallback을 선택하면 sample data를 암시적으로 생성했습니다. | `e1a0316fbe84`가 startup seed call을 제거했습니다. | `5cac4843fd9b`가 entrypoint direct call pattern을 금지합니다. | process lifecycle과 data initialization을 분리합니다. test는 가벼운 static guard이므로 indirect call까지 보장한다고 해석하면 안 됩니다. |
| 4 | 수동 test reset은 잘못된 URL/schema를 파괴하거나 sibling을 지울 위험이 있었습니다. | `113b3c422192` strict resolver와 `434403a7c16a` explicit reset executor를 추가했습니다. | `527b5f137425`가 거부 matrix와 sibling-schema 보존을 검증합니다. | 검증은 DB 연결 전 fail-closed하며 executor는 검증 결과만 사용합니다. schema replacement와 migration은 별도 lifecycle이므로 migration 실패 시 command 실패 상태가 남을 수 있습니다. |

## 8. Ownership·상태·책임 변화

| 구간 | 이전 소유자/표현 | 이후 소유자/표현 | 관련 SHA | 학습자 기록 |
| --- | --- | --- | --- | --- |
| Schema evolution | repository `ensureSeedData`가 DDL과 data upsert를 함께 소유했습니다. | Kysely Migrator가 version file과 applied state를 소유합니다. | `f9bb622a1117` | repository는 이미 존재하는 schema를 전제로 seed만 수행하며 caller가 migrate-before-seed 순서를 명시합니다. |
| Seed dataset | 하나의 implicit 개발 집합이었습니다. | repository가 development/demo profile별 upsert set을 소유합니다. | `8da6edef28eb` | 환경 선택은 CLI/deployment caller가 소유하고 production startup은 이를 자동으로 실행하지 않습니다. |
| Readiness | API가 concrete DB 상태를 추측해야 했습니다. | repository가 connectivity와 migration current 판단을 소유합니다. | `30aac132e14e`, `2f05d5d79c64` | API는 backend 세부사항 대신 공통 readiness operation을 소비합니다. |
| Startup | composition root가 runtime boot와 seed mutation을 함께 했습니다. | API는 boot/listen만, DB CLI·test fixture가 data initialization을 소유합니다. | `e1a0316fbe84` | 빈 memory state는 허용되는 명시적 결과이며 startup이 sample data로 감추지 않습니다. |
| Test reset | 운영자/시험 코드가 임의 SQL과 URL을 직접 다룰 수 있었습니다. | resolver가 target safety, reset function이 transaction/resource, migrator가 재적용을 소유합니다. | `113b3c422192` → `434403a7c16a` | 각 소유자는 검증·파괴·재구성을 분리하고 error/cleanup 경계를 명시합니다. |

## 9. Thread 최종 상태

최종적으로 schema evolution은 Kysely file migration set과 applied record가, seed는 environment profile을 받는 repository operation이 소유합니다. repository readiness는 연결 성공과 migration set current를 함께 요구하며, API startup은 데이터를 만들지 않습니다. `reset:test`는 strict allow-list target에만 transactional schema replacement를 수행하고 이후 migration을 재적용합니다. 이 설계는 production rollback이나 deployment sequencing 전체를 대신하지 않습니다.

## 10. 최종 실행 흐름

1. 배포·개발 caller가 `migrate` command를 실행하면 Migrator가 bundled SQL 이름과 applied state를 기준으로 pending migration을 적용합니다.
2. 필요한 환경에서만 `seed:dev` 또는 `seed:demo`가 repository를 만들고 profile-specific upsert를 수행한 뒤 `finally`에서 close합니다.
3. API startup은 repository와 application을 구성할 뿐 seed write를 수행하지 않습니다.
4. readiness 요청은 repository에서 connectivity probe와 migration set comparison을 수행하고 current일 때만 성공합니다.
5. test reset은 env/URL/schema allow-list를 먼저 평가해 unsafe target이면 DB 연결 전 종료합니다.
6. 안전 target은 transaction에서 drop/create되고 client/pool이 정리된 뒤 같은 target에 migration이 재적용됩니다.
7. unit/static/integration tests는 각각 resolver matrix, startup direct-call 금지, 실제 schema 격리·sibling 보존을 서로 다른 범위에서 보호합니다.

## 11. 학습 완료 확인

- [x] migration과 seed가 분리돼야 하는 이유를 command 이름이 아니라 state ownership으로 설명할 수 있습니다.
- [x] current, pending, diverged의 차이와 readiness failure 의미를 설명할 수 있습니다.
- [x] startup seed 제거가 만드는 빈 상태와 명시적 initialization 책임을 설명할 수 있습니다.
- [x] source-level startup test의 증거 한계를 과장하지 않습니다.
- [x] reset guard의 exact allow-list, DDL transaction, 별도 migration 단계와 failure state를 설명할 수 있습니다.
- [x] Thread 2의 수정된 commit 순서가 실제 branch chronology와 일치함을 확인했습니다.

## 12. 실행 및 증거 기록

- 저장소 실행 시험: 실행하지 않았습니다.
- 이유: 로컬 `git clone --branch web/ft_transcendence --single-branch`가 DNS 해석 실패(`Could not resolve host: github.com`)로 중단되어 의존성을 포함한 실행 가능한 checkout을 만들 수 없었습니다.
- 코드 근거: 지정 브랜치의 source classification과 각 exact SHA의 GitHub commit diff를 확인했습니다. 따라서 본 문서의 시험 설명은 실제 시험 코드의 정적 검토 결과이며, 이 환경에서의 통과 결과가 아닙니다.
===== END FILE: 02-migration-seed-readiness-and-reset-lifecycle.md =====

===== BEGIN FILE: 03-row-mapping-and-backend-contract-alignment.md =====
# Development Thread 03 — Row mapping과 backend contract 정렬

## 1. 학습 목표

- 행 타입·memory stored record·mapper input/output의 중복 표현을 canonical schema로 모으는 refactor sequence를 재구성합니다.
- behavior-preserving refactor라는 주장을 각 exact diff의 변경 범위와 non-guarantee로 제한합니다.
- PostgreSQL relation assembly와 memory aggregate lookup에서 ownership이 어떻게 명시적으로 드러나는지 설명합니다.
- pure mapper regression test가 무엇을 고정하고 live query·transaction은 왜 증명하지 못하는지 구분합니다.

## 2. 범위와 경계

- 포함: user projection, memory match record, canonical row aliases, record view, explicit query shapes, tournament/admin relation helpers, memory aggregate lookup, mapper unit test.
- 제외: C-level formatting-only commits는 state·ownership·behavior 변화가 없어 추가하지 않았습니다.
- 제외: match finalization atomicity, admin audit transaction, tournament admission lock은 별도의 데이터 무결성 Thread에서 다룹니다.
- 이 Thread의 refactor는 runtime validation을 새로 제공하지 않습니다. TypeScript contract와 code reviewability가 중심입니다.

## 3. 핵심 질문

- write command와 stored record가 같은 타입을 공유하면 어떤 lifetime·확장 문제가 생깁니까?
- canonical row type은 실제 SQL constraint나 runtime validation과 어떻게 다릅니까?
- row mapper가 relation을 직접 조회하지 않고 caller가 related-data object를 조립하는 이유는 무엇입니까?
- memory child lookup이 aggregate와 child를 함께 반환하면 어떤 잘못된 mutation target을 줄입니까?
- pure mapper test로 behavior preservation을 어디까지 주장할 수 있습니까?

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags |
| ---: | --- | --- | :---: | --- |
| 1 | `73b8ce0f0c26` | `refactor(db): repository user projection 타입 정렬` | B | AUTH, PERSISTENCE |
| 2 | `3d0ae79affd5` | `refactor(db): memory match record 계약 정렬` | B | PERSISTENCE |
| 3 | `3e3f21129369` | `refactor(db): canonical row schema 타입 정렬` | B | PERSISTENCE, TOURNAMENT |
| 4 | `212650b2863d` | `refactor(db): row mapper record 타입 정렬` | B | PERSISTENCE, TOURNAMENT |
| 5 | `45144a3719bc` | `refactor(db): dashboard와 friendship 조회 경계 정렬` | B | PERSISTENCE |
| 6 | `ce41a880d6c6` | `refactor(db): PostgreSQL tournament helper와 admin 경계 정렬` | B | PERSISTENCE, TOURNAMENT |
| 7 | `5c8659ea233b` | `refactor(db): tournament relation mapper 계약 정렬` | B | PERSISTENCE, TOURNAMENT |
| 8 | `f77e317de4c1` | `refactor(db): memory match completion과 admin 경계 정렬` | B | REALTIME, PERSISTENCE, TOURNAMENT |
| 9 | `9d64ea406b03` | `refactor(db): memory tournament 확정 경계 정렬` | B | PERSISTENCE, TOURNAMENT |
| 10 | `b34fdaa1e9c2` | `refactor(db): memory chat과 tournament 진입 경계 정렬` | B | PERSISTENCE, TOURNAMENT |
| 11 | `dc0e60e6aa35` | `test(db): database row mapping contract 검증` | B | PERSISTENCE, TOURNAMENT, TEST |

## 5. Commit별 조사

### 5.1. `73b8ce0f0c26` — refactor(db): repository user projection 타입 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `73b8ce0f0c26` |
| Importance | B |
| Tags | AUTH, PERSISTENCE |
| Source role | memory-only user alias를 제거하고 canonical `UserProjectionRow`를 repository 저장 표현으로 사용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 memory `users` Map type이 이전 `MemoryUserRow`에서 무엇으로 바뀌는지 확인합니다.
- `packages/db/src/schema.ts`의 canonical `UserProjectionRow` field set과 memory object construction을 대조합니다.
- email·created_at·banned_at처럼 projection에 포함/제외된 field를 확인합니다.
- runtime mutation이나 query 결과가 바뀌지 않고 type ownership만 정렬되는지 parent diff로 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory repository는 SQL user projection과 별도의 `MemoryUserRow` alias를 유지해 같은 user shape가 두 곳에서 정의됐습니다. |
| 해결하려던 문제 | schema 변경 시 memory alias만 늦게 갱신되거나 mapper가 받는 shape와 저장 shape가 갈라질 위험이 있었습니다. |
| 핵심 결정 | memory user store의 값 타입을 canonical `UserProjectionRow`로 바꾸고 객체 생성 field를 명시했습니다. |
| 입력 → 상태 전이 → 출력 | memory user 입력 → canonical projection 객체 생성/저장 → 기존 mapper로 public/session DTO 변환입니다. |
| ownership / lifetime / cleanup | `schema.ts`가 user projection vocabulary를 소유하고 memory repository는 그 타입의 원본 객체를 Map lifetime 동안 소유합니다. |
| failure / rollback / retry | compile-time refactor이므로 runtime validation은 추가되지 않습니다. 누락 field는 typecheck에서 잡히지만 잘못된 값은 실행 중 그대로 들어갈 수 있습니다. |
| 보장하는 것 | PostgreSQL query projection과 memory storage가 같은 TypeScript field contract를 참조합니다. |
| 보장하지 않는 것 | PostgreSQL과 memory의 정렬·filter·transaction 의미가 같아진 것은 아니며 동작 parity 전체를 보장하지 않습니다. |
| 후속 연결 | `3e3f21129369`가 더 많은 canonical enum/row type을 schema module로 모으고 mapper 시험이 후속 보호를 제공합니다. |

#### 비교 기준

- parent 상태와 `73b8ce0f0c26`의 diff를 먼저 비교합니다.
- 후속 관련 SHA `3d0ae79affd5`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.2. `3d0ae79affd5` — refactor(db): memory match record 계약 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `3d0ae79affd5` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | write command를 상속하던 memory match record를 명시적 stored shape로 바꿉니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 이전 `MemoryMatchRecord`가 write input을 intersection/상속하던 부분과 새 explicit field를 비교합니다.
- `createMatch` 또는 finalization path가 command에서 어떤 field만 복사해 저장하는지 확인합니다.
- `memoryMatchSummary`가 새 `endedAt`·participant ID field를 읽는 경로를 확인합니다.
- tournament command metadata나 일시적 입력이 stored record에 우발적으로 남지 않는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory match record는 write command shape를 재사용해 저장해야 할 state와 호출 순간의 command metadata가 구분되지 않았습니다. |
| 해결하려던 문제 | command가 확장될 때 memory persistence shape도 자동 확장되어 ownership과 lifetime이 불명확해질 수 있었습니다. |
| 핵심 결정 | `MemoryMatchRecord`를 id, result key, mode, participant IDs, scores, endedAt 등 실제 저장 field로 명시했습니다. |
| 입력 → 상태 전이 → 출력 | match command → 필요한 field 선택·새 record 생성 → memory 배열 저장 → summary mapper가 stored field만 읽습니다. |
| ownership / lifetime / cleanup | command 객체는 caller lifetime, stored record는 repository 배열 lifetime을 가집니다. 두 객체가 별도 값으로 분리됩니다. |
| failure / rollback / retry | 저장 중 failure injection이나 transaction은 추가되지 않습니다. 단순 memory push의 process-local atomicity만 유지됩니다. |
| 보장하는 것 | memory 저장 표현이 write input 변화에 암시적으로 끌려가지 않고 stored contract를 명확히 유지합니다. |
| 보장하지 않는 것 | durable persistence, cross-process consistency, PostgreSQL row와 byte-level 동일성은 보장하지 않습니다. |
| 후속 연결 | `9d64ea406b03`가 tournament finalization에서도 aggregate/match lookup 결과를 명시적으로 다룹니다. |

#### 비교 기준

- parent 상태와 `3d0ae79affd5`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `73b8ce0f0c26`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `3e3f21129369`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.3. `3e3f21129369` — refactor(db): canonical row schema 타입 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `3e3f21129369` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | database enum·row projection vocabulary를 `schema.ts`의 canonical type으로 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/schema.ts`의 `TournamentRound`, `TournamentMatchStatus`, `ChatScope`, `AdminAction` alias를 shared contract와 대조합니다.
- `Database` table map과 각 `Selectable` row export의 재배치를 확인합니다.
- `UserProjectionRow`가 broad alias 대신 explicit `Pick`으로 어떤 column을 고정하는지 확인합니다.
- joined row interface가 canonical aliases를 참조하도록 바뀌고 중복 string union이 제거되는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | round/status/scope/action과 여러 row projection이 mapper·repository에 중복 선언되어 같은 DB column을 다른 union으로 표현할 수 있었습니다. |
| 해결하려던 문제 | row mapper refactor를 계속하려면 한 module이 DB-facing type vocabulary를 소유해야 했습니다. |
| 핵심 결정 | `schema.ts`에 canonical alias와 explicit row projection을 모으고 joined row가 이를 참조하도록 정렬했습니다. |
| 입력 → 상태 전이 → 출력 | SQL table interface → `Selectable` row type → joined projection → mapper 입력 type으로 한 방향의 type dependency를 만듭니다. |
| ownership / lifetime / cleanup | `schema.ts`가 DB row vocabulary를 소유하고 mapper/repository는 import해 소비합니다. runtime row lifetime은 변하지 않습니다. |
| failure / rollback / retry | DB constraint나 runtime parser는 추가되지 않습니다. TypeScript 선언이 실제 SQL과 어긋나면 실행 중 자동 검출하지 못합니다. |
| 보장하는 것 | 동일 column의 compile-time 표현을 한 곳에서 변경하고 downstream type drift를 줄입니다. |
| 보장하지 않는 것 | 실제 enum 값의 DB CHECK, query column completeness, behavior change 부재를 실행으로 증명하지는 않습니다. |
| 후속 연결 | `212650b2863d`와 `5c8659ea233b`가 mapper 출력·relation assembly를 이 canonical type에 맞춥니다. |

#### 비교 기준

- parent 상태와 `3e3f21129369`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `3d0ae79affd5`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `212650b2863d`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.4. `212650b2863d` — refactor(db): row mapper record 타입 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `212650b2863d` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | `toTournamentMatchRecord`의 반환값을 canonical row type에서 파생한 explicit view로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/rowMappers.ts`의 `TournamentMatchRecordView`와 `toTournamentMatchRecord` return annotation을 확인합니다.
- round/status type이 hard-coded union이 아니라 `TournamentMatchRow` field에서 파생되는지 확인합니다.
- nullable score·ID field에서 `Number`/null 변환이 유지되는지 parent와 비교합니다.
- public summary mapper와 internal record mapper의 field 차이를 구분합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | tournament match record mapper는 반환 타입이 암시적이어서 schema alias가 바뀌어도 caller 계약이 명확히 드러나지 않았습니다. |
| 해결하려던 문제 | internal finalization/lookup code가 public summary와 다른 최소 record shape를 안전하게 소비해야 했습니다. |
| 핵심 결정 | `TournamentMatchRecordView`를 정의하고 round/status를 canonical row field에서 파생해 mapper 반환 타입을 명시했습니다. |
| 입력 → 상태 전이 → 출력 | `TournamentMatchRow` → snake_case ID/status/participant field 선택 → camelCase internal record view 반환입니다. |
| ownership / lifetime / cleanup | mapper가 새 value를 소유하고 DB row는 query result lifetime에 남습니다. view는 public relation user를 소유하지 않고 ID만 담습니다. |
| failure / rollback / retry | runtime validation은 없고 malformed null/type은 차단하지 않습니다. 변환 failure는 일반 JS error로 드러날 수 있습니다. |
| 보장하는 것 | internal record caller가 어떤 field를 받을 수 있는지 compile-time에 고정되고 schema alias 변경이 전파됩니다. |
| 보장하지 않는 것 | query가 올바른 row를 가져왔다는 보장, match lifecycle의 legal transition, relation user 존재성은 제공하지 않습니다. |
| 후속 연결 | `dc0e60e6aa35`가 정확한 record shape를 pure mapper test로 검증합니다. |

#### 비교 기준

- parent 상태와 `212650b2863d`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `3e3f21129369`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `45144a3719bc`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.5. `45144a3719bc` — refactor(db): dashboard와 friendship 조회 경계 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `45144a3719bc` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | optional SQL fragment를 제거하고 scoped/unscoped recent-match query를 두 개의 명시적 shape로 분리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `PostgresRepository.listRecentMatches` parent에서 conditional `sql` fragment가 사용되던 부분과 두 explicit query branch를 비교합니다.
- 두 query의 select column, joins, order, limit가 동일하고 where만 다른지 확인합니다.
- `listFriends` SQL formatting과 dashboard object construction이 behavior를 바꾸는지 확인합니다.
- 분기 duplication이 늘어나는 대신 query shape reviewability가 좋아지는 trade-off를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | recent-match query는 optional raw SQL fragment를 중간에 삽입해 scoped/unscoped query의 최종 shape를 한눈에 검토하기 어려웠습니다. |
| 해결하려던 문제 | typed query 결과와 participant predicate가 branch별로 정확히 일치하는지 reviewer가 확인하기 쉬운 표현이 필요했습니다. |
| 핵심 결정 | userId 존재 여부에 따라 완전한 두 SQL query를 선택하고 나머지 mapping·dashboard behavior는 유지했습니다. |
| 입력 → 상태 전이 → 출력 | `userId` 있음 → participant where가 포함된 query, 없음 → global query → 같은 `toMatchSummary` mapping과 newest-first limit입니다. |
| ownership / lifetime / cleanup | repository method가 query branch 선택을 소유하고 mapper가 출력 변환을 계속 소유합니다. |
| failure / rollback / retry | SQL duplication으로 한 branch만 수정할 위험이 생깁니다. 이 commit은 transaction·failure handling을 추가하지 않습니다. |
| 보장하는 것 | 실제 실행될 SQL shape와 bind parameter 위치를 branch마다 명시적으로 검토할 수 있습니다. |
| 보장하지 않는 것 | 성능 개선, result parity의 실행 증거, friendship semantics 변경은 보장하지 않습니다. |
| 후속 연결 | `dc0e60e6aa35`는 mapper shape를 보호하지만 두 SQL branch의 integration behavior 자체는 직접 실행하지 않습니다. |

#### 비교 기준

- parent 상태와 `45144a3719bc`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `212650b2863d`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `ce41a880d6c6`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.6. `ce41a880d6c6` — refactor(db): PostgreSQL tournament helper와 admin 경계 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `ce41a880d6c6` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | tournament match relation 조립과 admin query를 명시적 helper/statement로 정리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `tournamentMatchFromRow`가 left/right/winner ID를 `getUserById`로 해석하는 순서를 확인합니다.
- `ensureFinalMatch` 또는 tournament helper가 어떤 executor/row를 받는지 확인합니다.
- `listAdminUsers`, `listAdminActions`, `setUserBan`의 query와 mapper 호출을 parent와 비교합니다.
- 이 refactor가 admin status update와 audit insert를 하나의 transaction으로 만들지는 않는다는 점을 명시합니다.
- relation user가 삭제·누락된 경우 null 처리와 thrown failure를 구분합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | PostgreSQL tournament relation 조립과 admin methods가 긴 inline query·mapping에 섞여 책임 위치를 파악하기 어려웠습니다. |
| 해결하려던 문제 | 한 tournament match row의 세 user relation을 동일 규칙으로 해석하고 admin query도 독립적으로 검토할 수 있어야 했습니다. |
| 핵심 결정 | `tournamentMatchFromRow` helper를 추출하고 admin query/mapper 단계를 명시적으로 배치했습니다. |
| 입력 → 상태 전이 → 출력 | match row → left/right/winner ID별 user lookup → relation object → summary mapper; admin row → actor/target lookup → admin summary입니다. |
| ownership / lifetime / cleanup | helper가 relation assembly를 소유하고 mapper가 shape conversion을 소유합니다. repository가 DB call lifetime을 관리합니다. |
| failure / rollback / retry | 여러 relation lookup 중 하나가 실패하면 상위 operation이 실패할 수 있습니다. admin update와 audit insert atomicity는 이 refactor가 보장하지 않습니다. |
| 보장하는 것 | tournament/admin의 caller-callee 관계와 relation resolution 위치가 분명해집니다. |
| 보장하지 않는 것 | N+1 query 제거, transaction 추가, behavior equivalence의 runtime proof는 제공하지 않습니다. |
| 후속 연결 | `5c8659ea233b`가 tournament aggregate mapper에 explicit related-data object를 요구하며 relation ownership을 더 명확히 합니다. |

#### 비교 기준

- parent 상태와 `ce41a880d6c6`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `45144a3719bc`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `5c8659ea233b`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.7. `5c8659ea233b` — refactor(db): tournament relation mapper 계약 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `5c8659ea233b` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | tournament mapper가 row spread와 사후 mutation 대신 explicit related-data object를 받도록 바꿉니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/rowMappers.ts`의 이전 `toTournamentSummary(row, entries, matches)`와 새 `{ entries, matches, winner }` 인수를 비교합니다.
- creator projection이 `{...row, id: creator_id, status: user_status}` spread에서 explicit field object로 바뀌는지 확인합니다.
- `PostgresRepository.tournamentFromRow`가 entries query, matches helper, winner lookup을 모두 끝낸 뒤 mapper를 호출하는지 추적합니다.
- `ensureTournamentBracket` executor type이 `Kysely | Transaction`으로 명시되는지 확인합니다.
- mapper가 더 이상 반환 후 `summary.winner = ...` mutation을 요구하지 않는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 기존 mapper는 tournament row에 creator field를 spread해 다른 row처럼 보이게 하고, winner는 mapper 반환 후 mutation으로 채웠습니다. |
| 해결하려던 문제 | row provenance와 related data ownership이 숨겨져 누락 relation·잘못된 field alias가 compile-time에서 드러나기 어려웠습니다. |
| 핵심 결정 | `toTournamentSummary(row, {entries, matches, winner})`를 도입하고 creator를 explicit field로 구성하며 모든 relation을 호출 전에 조립했습니다. |
| 입력 → 상태 전이 → 출력 | tournament+creator row → entries/matches/winner 비동기 조회 → related object → 한 번의 pure summary construction입니다. |
| ownership / lifetime / cleanup | repository가 relation fetch와 async lifetime을 소유하고 mapper가 완성된 immutable-style DTO construction을 소유합니다. 반환 후 mutation이 사라집니다. |
| failure / rollback / retry | relation query 중 하나가 실패하면 aggregate 전체가 실패합니다. batch loading이나 transaction-consistent snapshot은 추가되지 않습니다. |
| 보장하는 것 | mapper 입력에서 raw row와 related data가 명시적으로 구분되고 winner 누락을 사후 mutation에 의존하지 않습니다. |
| 보장하지 않는 것 | N+1 query, concurrent row 변화 사이의 snapshot consistency, runtime validation은 보장하지 않습니다. |
| 후속 연결 | `dc0e60e6aa35`가 entries/matches/winner를 넣은 mapper output을 고정합니다. |

#### 최소 코드 근거

- `packages/db/src/rowMappers.ts::toTournamentSummary` — `row`와 `{ entries, matches, winner }`를 분리해 relation provenance와 필수 조립 단계를 명시합니다.

#### 비교 기준

- parent 상태와 `5c8659ea233b`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `ce41a880d6c6`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `f77e317de4c1`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.8. `f77e317de4c1` — refactor(db): memory match completion과 admin 경계 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `f77e317de4c1` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| Source role | memory tournament match lookup을 aggregate와 child를 함께 반환하는 helper로 통합합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.ts`의 `findTournamentMatch(matchId)` 반환 `{ tournament, match } | null`을 확인합니다.
- `completeTournamentMatch`가 helper 결과와 winner lookup을 재사용해 어느 객체를 mutation하는지 추적합니다.
- `listAdminUsers`, `listAdminActions`, `setUserBan`이 explicit object construction으로 바뀐 부분을 확인합니다.
- memory mutation이 transaction/rollback 없이 process-local object를 직접 바꾼다는 점을 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory completion은 tournament 배열과 match를 별도로 다시 찾고 non-null assertion에 의존했습니다. |
| 해결하려던 문제 | child match를 찾은 뒤 소유 aggregate를 잃으면 status·winner를 다른 객체에 반영하거나 lookup logic이 중복될 수 있었습니다. |
| 핵심 결정 | `findTournamentMatch`가 aggregate와 child를 한 결과로 반환하고 completion/admin methods가 explicit local 값을 사용하도록 정렬했습니다. |
| 입력 → 상태 전이 → 출력 | match ID → `{tournament, match}` lookup → winner projection 조회 → child status/score 변경 → semifinal이면 final 생성, 아니면 aggregate 완료입니다. |
| ownership / lifetime / cleanup | repository 배열이 tournament aggregate와 child object lifetime을 소유하고 helper는 alias pair를 일시적으로 반환합니다. |
| failure / rollback / retry | winner lookup 후 mutation 중 예외가 발생하면 자동 rollback이 없습니다. process-local sequential method라는 전제입니다. |
| 보장하는 것 | match와 그 소유 tournament가 같은 lookup 결과로 함께 이동해 mutation target이 분명해집니다. |
| 보장하지 않는 것 | PostgreSQL transaction parity, concurrent mutation safety, deep copy 반환은 보장하지 않습니다. |
| 후속 연결 | `9d64ea406b03`가 같은 helper를 atomic finalization의 memory tournament branch에 재사용합니다. |

#### 비교 기준

- parent 상태와 `f77e317de4c1`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `5c8659ea233b`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `9d64ea406b03`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.9. `9d64ea406b03` — refactor(db): memory tournament 확정 경계 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `9d64ea406b03` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | memory `finalizeMatch`의 tournament participant 검증과 aggregate mutation을 하나의 lookup result에 정렬합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `finalizeMatch`의 이전 `tournamentLink` map/find 표현과 `findTournamentMatch` 사용을 비교합니다.
- already-finalized `matchId` 검사와 winner/loser가 해당 tournament match participant인지 검증하는 순서를 확인합니다.
- 일반 match record/stat update 뒤 tournament match/aggregate를 mutation하는 경로를 추적합니다.
- memory operation이 한 call stack에 있지만 예외 시 deep rollback이 없다는 점을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory finalization은 tournament와 child를 임시 map 구조로 찾고 각 조건에서 optional chain을 반복했습니다. |
| 해결하려던 문제 | participant 검증·중복 확정·semifinal/final mutation이 같은 aggregate를 대상으로 한다는 사실을 코드가 명확히 유지해야 했습니다. |
| 핵심 결정 | `findTournamentMatch` 결과를 하나의 local value로 사용해 participant set과 subsequent mutation을 정렬했습니다. |
| 입력 → 상태 전이 → 출력 | command → tournament child lookup → already-finalized·participant 검증 → match/stat update → child result 기록 → semifinal final 생성 또는 tournament winner/status 확정입니다. |
| ownership / lifetime / cleanup | repository가 match record, user stats, tournament aggregate를 같은 method 동안 소유합니다. helper result는 원본 object alias입니다. |
| failure / rollback / retry | 검증 전에는 mutation하지 않지만 일반 match/stat update 이후 tournament mutation에서 예외가 나면 PostgreSQL transaction과 같은 rollback은 없습니다. |
| 보장하는 것 | memory 구현에서 tournament validation과 mutation이 동일 child/aggregate reference를 사용합니다. |
| 보장하지 않는 것 | durable atomicity, failure injection rollback, cross-process idempotency는 보장하지 않습니다. |
| 후속 연결 | `dc0e60e6aa35`는 mapper를 검증하고, match-finalization atomicity 자체는 별도의 persistence Thread에서 다룹니다. |

#### 비교 기준

- parent 상태와 `9d64ea406b03`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `f77e317de4c1`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `b34fdaa1e9c2`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.10. `b34fdaa1e9c2` — refactor(db): memory chat과 tournament 진입 경계 정렬

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `b34fdaa1e9c2` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | memory chat/tournament object construction과 lookup을 explicit shared-domain shape에 맞춥니다. |

#### 해당 SHA에서 확인할 실제 코드

- `createChatMessage`의 `ChatMessage` type annotation과 field별 object construction을 확인합니다.
- `createTournament`가 `TournamentSummary`를 명시하고 entries/matches 초기값을 모두 제공하는지 확인합니다.
- `joinTournament`의 `alreadyJoined`, capacity check, append, playerCount/status 전이를 추적합니다.
- `getTournamentMatch`, `startTournamentMatch`가 `findTournamentMatch`를 재사용하는지 확인합니다.
- 반복 join의 idempotence와 full tournament에서 기존 참가자의 재join 허용을 구분합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory methods는 compact object literal과 여러 lookup 방식에 의존해 shared domain shape와 상태 전이가 숨겨졌습니다. |
| 해결하려던 문제 | chat/tournament의 생성·참가·match 시작이 같은 typed boundary를 사용하고 중복 lookup을 줄일 필요가 있었습니다. |
| 핵심 결정 | 반환 객체에 explicit type을 주고 join 조건을 분기하며 tournament match lookup을 공통 helper로 통일했습니다. |
| 입력 → 상태 전이 → 출력 | chat input → sender 검증 → typed message 저장; tournament create → typed aggregate 저장; join → existing/full 검사 → entry append → count/status/bracket 갱신입니다. |
| ownership / lifetime / cleanup | repository 배열이 message와 tournament aggregate를 소유하고 caller는 같은 객체 projection을 받습니다. helper가 child alias를 찾습니다. |
| failure / rollback / retry | full 상태에서 신규 사용자는 실패하고 기존 사용자는 no-op 후 summary를 받습니다. method 중간 예외에 대한 rollback은 없습니다. |
| 보장하는 것 | memory domain object의 필수 field와 join idempotence/capacity 분기가 명시적으로 드러납니다. |
| 보장하지 않는 것 | 실제 concurrent request serialization, deep immutability, PostgreSQL transaction과 동일한 failure semantics는 보장하지 않습니다. |
| 후속 연결 | Thread 5의 `efdb5c3a4932`가 entrant를 canonical user store에서 검증하고 concurrency test가 final-slot 결과를 확인합니다. |

#### 비교 기준

- parent 상태와 `b34fdaa1e9c2`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `9d64ea406b03`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `dc0e60e6aa35`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.11. `dc0e60e6aa35` — test(db): database row mapping contract 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `dc0e60e6aa35` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT, TEST |
| Source role | pure row fixture에서 user·match·friendship·chat·tournament·admin mapper 출력을 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/rowMappers.test.ts`의 canonical `UserRow` fixture와 고정 UUID/Date를 확인합니다.
- `toPublicUser`가 snake_case를 노출하지 않고 online/role/rating을 변환하는 assertion을 확인합니다.
- joined match/friendship/chat row에서 viewer-relative result, opponent, sender, ISO timestamp를 확인합니다.
- tournament record/summary의 entries·matches·winner와 admin actor/target relation assertion을 확인합니다.
- live query나 PostgreSQL을 사용하지 않는 pure mapper unit test라는 범위를 명시합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 여러 refactor가 TypeScript contract를 정리했지만 runtime object shape가 바뀌지 않았다는 집중 regression evidence가 없었습니다. |
| 해결하려던 문제 | explicit projection·relation object 변경 중 snake_case 누출, viewer-relative sign, nullable relation, timestamp conversion이 깨질 수 있었습니다. |
| 핵심 결정 | 고정 row fixture로 모든 주요 mapper를 직접 호출하고 exact/partial object assertion을 추가했습니다. |
| 입력 → 상태 전이 → 출력 | typed fixture row → mapper 호출 → public/internal DTO의 ID·이름·enum·numeric·ISO·relation field 비교입니다. |
| ownership / lifetime / cleanup | test fixture가 입력 value를 소유하고 mapper는 새 output object를 반환합니다. DB/pool resource는 없습니다. |
| failure / rollback / retry | malformed runtime row, SQL join 누락, query ordering은 재현하지 않습니다. pure function assertion 실패로 shape regression을 드러냅니다. |
| 보장하는 것 | 현재 mapper가 relational naming을 공개 DTO에 누출하지 않고 relation data를 의도한 shape로 조립함을 증명합니다. |
| 보장하지 않는 것 | 실제 query가 이 row shape를 반환한다는 것, transaction consistency, repository behavior 전체는 증명하지 않습니다. |
| 후속 연결 | 후속 mapper/schema refactor가 shared API contract를 바꾸면 이 test가 명시적으로 실패합니다. |

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | DB row에서 shared domain DTO로 변환할 때 naming, nullable relation, viewer-relative 결과, numeric/time 변환이 유지돼야 합니다. |
| 재현한 실패·경계 | joined row와 tournament relation object의 복합 shape, null winner/target, loss rating sign입니다. |
| 시험 기법 | 고정 fixture를 쓰는 pure unit/contract test입니다. |
| 통과하는 실제 코드 경로 | `toPublicUser`, `toMatchSummary`, `toFriendSummary`, `toChatMessage`, tournament/admin mapper functions입니다. |
| 시험이 증명하는 것 | mapper 함수 자체의 output shape와 핵심 field 의미를 deterministic하게 증명합니다. |
| 시험이 증명하지 않는 것 | SQL query, PostgreSQL driver conversion, relation snapshot consistency, repository transaction은 증명하지 않습니다. |
| 막으려는 회귀 | schema alias·mapper refactor 중 snake_case 누출, 누락 relation, 잘못된 sign/ISO 변환 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `dc0e60e6aa35`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `b34fdaa1e9c2`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.

## 6. 불변식 변화

| 단계 | 관련 SHA | 조사 초점 | 학습자 기록 |
| --- | --- | --- | --- |
| Canonical storage vocabulary | `73b8ce0f0c26` → `3e3f21129369` | 중복 alias·command inheritance가 제거되는 범위를 확인합니다. | memory user는 canonical projection을, memory match는 explicit stored shape를 사용하고 DB enum/row aliases는 `schema.ts`가 소유합니다. type ownership이 한 방향으로 정렬됩니다. |
| Explicit mapper view | `212650b2863d` | internal record와 public summary의 차이를 기록합니다. | tournament match internal record는 IDs·status 중심의 명시적 view를 반환하며 public relation user가 포함된 summary와 분리됩니다. |
| Explicit query·relation assembly | `45144a3719bc` → `5c8659ea233b` | query shape와 related-data provenance를 추적합니다. | conditional SQL fragment가 완전한 두 query로 바뀌고 tournament row, entries, matches, winner가 mapper 호출 전 명시적으로 조립됩니다. 사후 mutation과 row masquerading이 제거됩니다. |
| Memory aggregate lookup | `f77e317de4c1` → `b34fdaa1e9c2` | child와 owner aggregate의 alias·mutation 범위를 확인합니다. | `findTournamentMatch`가 원본 aggregate와 child를 함께 반환해 participant 검증·completion·start가 동일 객체를 대상으로 수행됩니다. process-local direct mutation이라는 한계는 남습니다. |
| Mapper regression evidence | `dc0e60e6aa35` | 정적 type refactor 뒤 runtime object shape를 확인합니다. | 고정 row fixture가 user/match/social/tournament/admin mapper의 naming·nullable·relation·time 변환을 검증하지만 live SQL row 생성은 다루지 않습니다. |

## 7. Failure → Fix → Test 관계

| 관계 | Failure / 이전 가정 | Fix / 결정 | Test / 근거 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 1 | memory-only alias와 write-command inheritance가 stored shape를 암시적으로 확장했습니다. | `73b8ce0f0c26`, `3d0ae79affd5`, `3e3f21129369`가 canonical projection과 explicit record를 도입했습니다. | `dc0e60e6aa35`가 downstream mapper output shape를 고정합니다. | compile-time source of truth를 줄여 schema 변화가 한 경로로 전파되게 합니다. test는 output을 보호하지만 storage/query parity 전체는 증명하지 않습니다. |
| 2 | tournament mapper가 row spread와 반환 후 winner mutation으로 relation provenance를 숨겼습니다. | `ce41a880d6c6`, `5c8659ea233b`가 helper와 `{ entries, matches, winner }` 계약을 추가했습니다. | `dc0e60e6aa35`의 tournament/admin fixture assertion입니다. | repository가 관계 조회를, mapper가 순수 shape construction을 소유합니다. 여러 query의 snapshot consistency는 별도 문제로 남습니다. |
| 3 | memory completion이 tournament와 child를 반복 lookup하고 optional chain/non-null assertion에 의존했습니다. | `f77e317de4c1` → `9d64ea406b03` → `b34fdaa1e9c2`가 aggregate+child helper를 공통 사용합니다. | 후속 memory repository·concurrency tests가 public behavior를 간접 보호합니다. | 동일 원본 object를 mutation한다는 사실이 명확해졌지만 transaction rollback이나 cross-request serialization을 추가한 것은 아닙니다. |

## 8. Ownership·상태·책임 변화

| 구간 | 이전 소유자/표현 | 이후 소유자/표현 | 관련 SHA | 학습자 기록 |
| --- | --- | --- | --- | --- |
| User/match stored type | memory-only alias와 write command가 shape를 간접 소유했습니다. | `schema.ts` projection과 explicit `MemoryMatchRecord`가 stored field를 소유합니다. | `73b8ce0f0c26`, `3d0ae79affd5` | caller command lifetime과 repository stored-record lifetime이 type에서도 분리됩니다. |
| DB type vocabulary | round/status/scope/action union이 여러 파일에 중복됐습니다. | canonical aliases와 row exports를 `schema.ts`가 소유합니다. | `3e3f21129369`, `212650b2863d` | mapper는 schema field에서 타입을 파생하지만 runtime row validation은 여전히 별도 책임입니다. |
| Relation assembly | mapper 입력 masquerading과 사후 mutation에 분산됐습니다. | repository helper가 relation fetch, mapper가 완성 DTO construction을 소유합니다. | `ce41a880d6c6`, `5c8659ea233b` | raw row와 related data가 분리되어 caller-callee contract가 명시됩니다. |
| Memory aggregate mutation | child와 owner를 별도 탐색했습니다. | helper가 `{tournament, match}` 원본 alias pair를 반환합니다. | `f77e317de4c1` → `b34fdaa1e9c2` | mutation target은 분명해졌지만 반환 projection의 deep immutability는 제공하지 않습니다. |
| Regression owner | typecheck만 behavior-preserving 근거였습니다. | pure mapper test가 public/internal output shape를 소유합니다. | `dc0e60e6aa35` | DB 없이 빠르게 shape를 고정하고 live query는 integration suite 책임으로 남깁니다. |

## 9. Thread 최종 상태

최종 상태에서 `schema.ts`가 DB-facing row와 enum vocabulary를 소유하고, memory repository는 command와 분리된 explicit stored record를 사용합니다. PostgreSQL repository는 query와 relation fetch를 명시적으로 수행한 뒤 related-data object를 mapper에 넘기며, memory repository는 aggregate와 child를 같은 lookup 결과로 다룹니다. mapper test는 변환 shape를 보호하지만 SQL 실행·transaction·concurrency를 증명하지 않습니다.

## 10. 최종 실행 흐름

1. SQL schema에 대응하는 canonical row/enum type을 `schema.ts`에서 import합니다.
2. PostgreSQL query는 완전한 query branch로 row를 읽고 relation helper가 left/right/winner 또는 entries/matches/winner를 조립합니다.
3. mapper는 raw row와 explicit related-data object를 받아 새 shared DTO를 한 번에 구성합니다.
4. memory backend는 canonical projection/explicit record를 저장하고 `findTournamentMatch`로 owner aggregate와 child를 함께 찾습니다.
5. completion·start·join은 그 원본 object를 직접 변경하며 transaction-like rollback은 제공하지 않습니다.
6. pure fixture test가 mapper의 naming, viewer-relative 값, nullable relation, ISO 변환을 고정합니다.

## 11. 학습 완료 확인

- [x] command type과 stored record type의 lifetime 차이를 설명할 수 있습니다.
- [x] canonical TypeScript row type이 DB CHECK나 runtime parser를 대신하지 않는다고 설명할 수 있습니다.
- [x] relation fetch와 mapper construction의 책임 분리를 실제 helper·function 이름으로 설명할 수 있습니다.
- [x] memory aggregate+child lookup의 장점과 rollback non-guarantee를 함께 말할 수 있습니다.
- [x] C-level formatting commit을 제외한 이유를 독립적인 state·ownership 변화 부재로 설명할 수 있습니다.
- [x] mapper unit test의 증거 범위를 live PostgreSQL query로 과장하지 않습니다.

## 12. 실행 및 증거 기록

- 저장소 실행 시험: 실행하지 않았습니다.
- 이유: 로컬 `git clone --branch web/ft_transcendence --single-branch`가 DNS 해석 실패(`Could not resolve host: github.com`)로 중단되어 의존성을 포함한 실행 가능한 checkout을 만들 수 없었습니다.
- 코드 근거: 지정 브랜치의 source classification과 각 exact SHA의 GitHub commit diff를 확인했습니다. 따라서 본 문서의 시험 설명은 실제 시험 코드의 정적 검토 결과이며, 이 환경에서의 통과 결과가 아닙니다.
===== END FILE: 03-row-mapping-and-backend-contract-alignment.md =====

===== BEGIN FILE: 04-canonical-friendship-and-concurrent-requests.md =====
# Development Thread 04 — Canonical friendship과 동시 요청

## 1. 학습 목표

- 초기 directional friendship row와 memory summary가 왜 하나의 관계 identity를 표현하지 못하는지 설명합니다.
- legacy data normalization, self-check, canonical expression unique index의 적용 순서를 재구성합니다.
- PostgreSQL single-statement upsert가 repeat/reverse request를 어떤 state transition으로 원자화하는지 추적합니다.
- memory parity와 regression test의 실제 concurrency 증거 범위를 과장하지 않고 구분합니다.

## 2. 범위와 경계

- 포함: friendship repository operation, canonical-pair migration, PostgreSQL atomic request, memory relationship record, cross-backend regression.
- `cdaca35ccf7f`에서는 friendship assertion만 이 Thread의 핵심 근거로 사용합니다. 같은 commit의 tournament test는 Thread 5에서 다룹니다.
- 제외: profile/friend HTTP authorization, guest capability, browser cache invalidation은 auth/web category 책임입니다.
- 제외: friendship 삭제·차단·notification policy는 이 history에 구현되지 않았으므로 일반론으로 채우지 않습니다.

## 3. 핵심 질문

- 왜 `(requester_id, addressee_id)` unique만으로 A↔B 관계 하나를 보장할 수 없습니까?
- 기존 duplicate를 정리하지 않고 canonical unique index를 추가하면 어떤 migration failure가 생깁니까?
- same-direction repeat와 reverse pending request는 각각 어떤 상태를 반환해야 합니까?
- PostgreSQL upsert에서 existing row와 `excluded` row의 방향 비교는 무엇을 결정합니까?
- memory parity test와 실제 concurrent database request 증거는 어떻게 다릅니까?

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags |
| ---: | --- | --- | :---: | --- |
| 1 | `645e5a3c8e96` | `feat(db): 친구 관계 저장 구현` | B | PERSISTENCE |
| 2 | `ffb0a8275a4f` | `feat(db): friendship canonical pair 제약 추가` | A | PERSISTENCE, RISK |
| 3 | `77c555aba9a0` | `feat(db): PostgreSQL friendship 요청을 원자화` | A | PERSISTENCE, RISK |
| 4 | `34db79005f30` | `feat(db): memory friendship invariant 적용` | B | PERSISTENCE |
| 5 | `cdaca35ccf7f` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | PERSISTENCE, TOURNAMENT, RISK |

## 5. Commit별 조사

### 5.1. `645e5a3c8e96` — feat(db): 친구 관계 저장 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `645e5a3c8e96` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | friendship list/request/accept를 처음 repository contract에 올리지만 관계 identity가 요청 방향과 backend 표현에 묶여 있습니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppRepository`의 `listFriends`, `requestFriend`, `acceptFriend` signature를 확인합니다.
- PostgreSQL `listFriends`의 `case when requester_id = userId then addressee_id else requester_id end` join을 추적합니다.
- `requestFriend`의 초기 `on conflict (requester_id, addressee_id)`가 같은 방향만 충돌로 보는지 확인합니다.
- `acceptFriend`의 `where id = friendshipId and addressee_id = userId` authorization 조건을 확인합니다.
- memory 구현이 `FriendSummary[]`만 저장하고 requester/addressee identity를 보존하지 않는 차이를 기록합니다.
- 자기 자신 요청과 reverse duplicate를 DB 제약/코드가 아직 막지 않는다는 점을 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | friendship table은 초기 schema에 있었지만 repository에서 관계를 생성·조회·수락할 operation이 없었습니다. |
| 해결하려던 문제 | social route가 raw SQL을 직접 다루지 않고 양쪽 사용자가 같은 관계를 조회할 수 있어야 했습니다. |
| 핵심 결정 | 공통 interface에 list/request/accept를 추가하고 PostgreSQL은 directional row, memory는 caller-facing `FriendSummary` 배열로 구현했습니다. |
| 입력 → 상태 전이 → 출력 | requester+상대 handle → user lookup → pending row/summary 생성; list → requester/addressee 어느 쪽인지 계산해 상대 user join; accept → addressee가 지정 row status를 accepted로 변경합니다. |
| ownership / lifetime / cleanup | PostgreSQL row는 두 user ID와 방향을 소유하지만 memory 배열은 requester/addressee를 버리고 public summary만 소유합니다. 이 차이가 이후 parity 문제의 원인입니다. |
| failure / rollback / retry | 동일 방향 재요청은 upsert되지만 역방향 요청은 별도 row가 될 수 있습니다. self-friend도 허용될 수 있고 memory accept는 actor 권한을 충분히 확인하지 않습니다. |
| 보장하는 것 | 기본 friendship lifecycle과 addressee-only accept 조건을 PostgreSQL path에 제공합니다. |
| 보장하지 않는 것 | unordered pair당 한 관계, self-friend 금지, reverse pending 자동 수락, backend parity, concurrent request atomicity는 보장하지 않습니다. |
| 후속 연결 | `ffb0a8275a4f`가 기존 데이터를 정규화하고 canonical pair를 DB invariant로 만들며 `34db79005f30`이 memory 표현을 같은 의미로 바꿉니다. |

#### 비교 기준

- parent 상태와 `645e5a3c8e96`의 diff를 먼저 비교합니다.
- 후속 관련 SHA `ffb0a8275a4f`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.2. `ffb0a8275a4f` — feat(db): friendship canonical pair 제약 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `ffb0a8275a4f` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| Source role | 기존 directional friendship 데이터를 정리한 뒤 unordered user pair당 하나의 row만 허용하는 migration을 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/migrations/004_friendship_tournament_invariants.sql`에서 self row 삭제가 가장 먼저 수행되는지 확인합니다.
- reverse pair가 있고 한쪽이 pending일 때 accepted로 승격하고 `greatest(updated_at)`를 사용하는 update를 추적합니다.
- `row_number() over (partition by least(...), greatest(...))`의 survivor 우선순위 accepted → created_at → id를 확인합니다.
- 기존 directional unique constraint drop과 `friendships_distinct_users_check` 추가를 확인합니다.
- `friendships_canonical_pair_unique` expression index가 requester/addressee 순서를 무시하는지 확인합니다.
- 데이터 정규화와 constraint 설치가 한 migration apply 안에서 수행되는 이유를 설명합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 초기 schema의 unique는 `(requester_id, addressee_id)` 순서를 보존해 A→B와 B→A가 서로 다른 관계로 저장됐고 self row도 막지 못했습니다. |
| 해결하려던 문제 | 새 constraint만 바로 추가하면 기존 reverse duplicate와 self row 때문에 migration이 실패하거나 어느 상태를 보존할지 불명확했습니다. |
| 핵심 결정 | migration이 self row를 삭제하고 reverse pending을 accepted로 합친 뒤 canonical pair별 survivor를 deterministic하게 남기고 expression unique/check를 설치합니다. |
| 입력 → 상태 전이 → 출력 | 기존 rows → self 삭제 → reverse 상태 reconciliation → canonical pair partition/rank → 중복 삭제 → directional unique 제거 → distinct-user CHECK와 unordered expression UNIQUE 설치입니다. |
| ownership / lifetime / cleanup | migration이 legacy data normalization과 새 DB invariant 설치를 소유합니다. PostgreSQL index가 이후 모든 writer의 unordered identity를 소유합니다. |
| failure / rollback / retry | 삭제되는 중복 row의 외부 참조가 있었다면 영향이 생길 수 있으나 초기 schema에는 해당 friendship ID를 참조하는 별도 FK가 없습니다. migration 실패 시 새 constraint는 적용되지 않습니다. |
| 보장하는 것 | 자기 관계를 DB에서 거부하고 requester/addressee 순서와 무관하게 한 pair당 한 row만 저장되도록 합니다. |
| 보장하지 않는 것 | 요청이 어떤 상태 전이를 해야 하는지, concurrent writer가 어떤 row를 반환하는지, memory backend parity는 아직 보장하지 않습니다. |
| 후속 연결 | `77c555aba9a0`이 expression index를 conflict target으로 사용하는 atomic upsert를 추가하고 `cdaca35ccf7f`가 실제 DB row 하나를 검증합니다. |

#### 최소 코드 근거

- `004_friendship_tournament_invariants.sql` — `least(requester_id, addressee_id), greatest(...)` expression unique index가 방향과 무관한 관계 identity를 DB에 강제합니다.

#### 비교 기준

- parent 상태와 `ffb0a8275a4f`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `645e5a3c8e96`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `77c555aba9a0`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.3. `77c555aba9a0` — feat(db): PostgreSQL friendship 요청을 원자화

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `77c555aba9a0` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| Source role | canonical expression index를 conflict target으로 사용해 repeat/reverse friendship request를 한 SQL statement에서 처리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `PostgresRepository.requestFriend`의 self-check와 상대 user lookup 순서를 확인합니다.
- `insert ... on conflict ((least(...)), (greatest(...))) do update` 전체 SQL을 추적합니다.
- 기존 pending row의 방향이 `excluded`와 반대일 때만 status를 accepted로 바꾸는 `case`를 확인합니다.
- 동일 방향 재요청에서는 status와 `updated_at`을 유지해 idempotent readback이 되는지 확인합니다.
- `returning id, status`가 conflict insert/update 모두에서 같은 relationship identity를 반환하는지 확인합니다.
- `acceptFriend`가 authorized update 뒤 `returning requester_id`로 상대를 조회하며 0-row update가 `firstRow` failure가 되는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | DB constraint는 unordered duplicate를 막았지만 기존 request code는 directional conflict target을 사용해 새 index와 맞지 않았고 reverse request의 상태 전이를 여러 query로 처리할 수 있었습니다. |
| 해결하려던 문제 | repeat·reverse 요청이 동시에 와도 한 row ID를 유지하고 reverse pending은 accepted로 바뀌어야 했습니다. |
| 핵심 결정 | canonical expression conflict target과 conditional `DO UPDATE`를 사용해 insert, same-direction no-op, reverse-pending accept를 한 statement로 표현했습니다. |
| 입력 → 상태 전이 → 출력 | 상대 handle lookup/self-check → pending insert 시도 → unordered pair conflict 시 existing/excluded 방향 비교 → reverse pending이면 accepted+timestamp 갱신, 아니면 기존 상태 유지 → id/status 반환입니다. |
| ownership / lifetime / cleanup | PostgreSQL unique index가 pair identity를, single SQL statement가 state transition atomicity를 소유합니다. caller는 반환 summary만 받습니다. |
| failure / rollback / retry | 없는 상대·self는 statement 전에 실패합니다. accept에서 addressee 조건에 맞는 row가 없으면 `RETURNING`이 비어 `firstRow`가 실패합니다. transaction 밖 상대 user 조회 사이에 account state가 바뀔 수는 있습니다. |
| 보장하는 것 | 같은 방향 반복은 같은 pending row, 반대 방향 pending request는 같은 row의 accepted 상태를 반환하며 duplicate row를 만들지 않습니다. |
| 보장하지 않는 것 | memory backend, 분산 DB 간 global uniqueness, friendship 삭제/차단 policy, serializable isolation 전체는 보장하지 않습니다. |
| 후속 연결 | `34db79005f30`이 동일 state model을 memory에 적용하고 `cdaca35ccf7f`가 양방향·반복 요청과 direct SQL row count를 검증합니다. |

#### 최소 코드 근거

- `PostgresRepository.requestFriend` — canonical pair expression을 conflict target으로 사용하고 reverse pending일 때만 `status = 'accepted'`로 바꾸는 단일 upsert입니다.

#### 비교 기준

- parent 상태와 `77c555aba9a0`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `ffb0a8275a4f`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `34db79005f30`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.4. `34db79005f30` — feat(db): memory friendship invariant 적용

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `34db79005f30` |
| Importance | B |
| Tags | PERSISTENCE |
| Source role | memory 저장 표현을 caller-specific summary에서 requester/addressee ID를 보존하는 canonical relationship record로 바꿉니다. |

#### 해당 SHA에서 확인할 실제 코드

- `MemoryFriendship { id, requesterId, addresseeId, status }`와 기존 `FriendSummary[]`를 비교합니다.
- `listFriends(userId)`가 actor와 관련된 row만 filter하고 other user ID를 계산하는지 확인합니다.
- `requestFriend`의 self-check, unordered existing lookup, reverse pending→accepted, same-direction repeat 반환을 추적합니다.
- `acceptFriend`가 `friend.addresseeId === userId`를 요구하는지 확인합니다.
- memory user 원본 lookup 실패와 public projection 생성 경로를 확인합니다.
- 동시 Promise가 실제 thread lock이 아니라 JS call-stack 수준의 순차 mutation이라는 한계를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory backend는 friendship의 두 주체를 저장하지 않아 사용자별 목록·accept authorization·reverse request 의미를 PostgreSQL처럼 재현할 수 없었습니다. |
| 해결하려던 문제 | 공통 interface 뒤에서 동일 테스트를 돌리려면 memory도 관계 identity와 방향을 원본 상태로 보존해야 했습니다. |
| 핵심 결정 | `MemoryFriendship` record를 도입하고 list/request/accept가 requester/addressee ID에서 상대와 권한을 계산하도록 했습니다. |
| 입력 → 상태 전이 → 출력 | requester+상대 → self 검사 → unordered existing 검색 → same 방향이면 기존 반환, reverse pending이면 status accepted, 없으면 새 pending record 저장; list/accept는 actor ID로 projection합니다. |
| ownership / lifetime / cleanup | memory repository가 relation record와 user Map을 process lifetime 동안 소유하고 반환 시 other user를 새 public projection으로 만듭니다. |
| failure / rollback / retry | 없는 user/unauthorized accept는 실패합니다. JS memory method에는 DB row lock·transaction rollback이 없으며 여러 process 사이의 상태는 공유되지 않습니다. |
| 보장하는 것 | self-reject, unordered one-record identity, repeated request idempotence, reverse pending accept와 addressee-only acceptance를 memory semantics에 반영합니다. |
| 보장하지 않는 것 | durability, cross-process concurrency, DB constraint 수준의 강제, mutation failure rollback은 보장하지 않습니다. |
| 후속 연결 | `cdaca35ccf7f`가 같은 scenario를 memory와 PostgreSQL에 적용해 observable parity를 검증합니다. |

#### 비교 기준

- parent 상태와 `34db79005f30`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `77c555aba9a0`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `cdaca35ccf7f`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.5. `cdaca35ccf7f` — test(db): friendship와 tournament 경쟁 상태 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `cdaca35ccf7f` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| Source role | 같은 test commit의 friendship 부분에서 self/repeat/reverse 요청과 canonical DB row를 memory·PostgreSQL 양쪽에 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 `keeps one friendship for both request directions`를 확인합니다.
- self request rejection, same-direction repeat equality, reverse request의 same ID/accepted 상태를 순서대로 추적합니다.
- 양쪽 `listFriends`가 동일 relationship ID와 상대 user를 반환하는 assertion을 확인합니다.
- `packages/db/src/postgres.integration.test.ts`의 PostgreSQL 동일 scenario와 direct `select requester_id, addressee_id, status from friendships`를 확인합니다.
- self row direct insert가 `friendships_distinct_users_check` constraint로 거부되는 assertion을 확인합니다.
- tournament final-slot test는 Thread 5에서 별도 해석하고 이 Thread에서는 friendship evidence만 사용합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | canonical migration, PostgreSQL upsert, memory record가 구현됐지만 같은 observable scenario에서 한 relationship identity를 유지하는지 regression evidence가 필요했습니다. |
| 해결하려던 문제 | self request, 같은 방향 반복, 반대 방향 요청은 초기 구현이 각각 self row·duplicate/no-op inconsistency를 만들던 경계입니다. |
| 핵심 결정 | memory unit scenario와 Testcontainers PostgreSQL scenario에 동일한 요청 순서를 적용하고 public 결과와 raw DB row를 함께 assertion했습니다. |
| 입력 → 상태 전이 → 출력 | 두 사용자 생성 → self reject → A→B pending → A→B repeat → B→A reverse accepted → 양쪽 list 확인 → PostgreSQL raw table 한 row와 CHECK rejection 확인입니다. |
| ownership / lifetime / cleanup | 각 test repository가 relationship state를 소유하고 integration harness가 schema/pool cleanup을 소유합니다. |
| failure / rollback / retry | memory는 실제 병렬 요청이 아니라 Promise가 순차적으로 실행되는 process-local behavior입니다. PostgreSQL test도 reverse request는 순차이며 friendship에 대한 simultaneous race를 직접 발사하지는 않습니다. |
| 보장하는 것 | self 금지, repeat idempotence, reverse transition, 양쪽 projection, raw canonical one-row invariant가 두 backend에서 유지됨을 코드상 검증합니다. |
| 보장하지 않는 것 | 분산 transaction, 높은 동시성 throughput, deadlock, 삭제/차단 policy, 이 세션에서의 실제 test 통과는 증명하지 않습니다. |
| 후속 연결 | 후속 refactor가 pair ordering이나 actor projection을 바꾸면 public/DB assertion이 회귀를 탐지합니다. |

#### 최소 코드 근거

- `postgres.integration.test.ts` — public repository 결과 뒤 raw `friendships` table이 정확히 한 accepted row인지 확인하고 self insert의 constraint 이름까지 검사합니다.

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | unordered user pair당 하나의 friendship ID가 있고 self relation은 없으며 reverse pending request는 accepted로 전이해야 합니다. |
| 재현한 실패·경계 | self request, same-direction repeat, reverse-direction request, 양쪽 list view, direct self-row insert입니다. |
| 시험 기법 | memory behavioral test와 real-PostgreSQL integration/constraint test를 결합합니다. |
| 통과하는 실제 코드 경로 | `requestFriend`, `listFriends`, `acceptFriend` 관련 memory/PG code, canonical unique index와 distinct-users CHECK입니다. |
| 시험이 증명하는 것 | 정해진 request sequence에서 두 backend가 같은 public 의미를 보이고 PostgreSQL table에는 한 row만 남는다는 것을 증명합니다. |
| 시험이 증명하지 않는 것 | friendship 요청을 진짜 동시에 발사하는 race, multi-process memory, production load와 본 세션의 실행 결과는 증명하지 않습니다. |
| 막으려는 회귀 | directional duplicate, self row, reverse pending이 별도 관계로 남는 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `cdaca35ccf7f`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `34db79005f30`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.

## 6. 불변식 변화

| 단계 | 관련 SHA | 조사 초점 | 학습자 기록 |
| --- | --- | --- | --- |
| Directional initial model | `645e5a3c8e96` | 초기 DB·memory 관계 identity와 authorization 차이를 기록합니다. | PostgreSQL은 방향을 가진 두 ID를 저장하지만 reverse duplicate를 허용하고, memory는 actor identity 자체를 버린 `FriendSummary[]`라 양쪽 projection과 accept 권한을 정확히 표현하지 못했습니다. |
| Canonical DB identity | `ffb0a8275a4f` | legacy cleanup과 새 constraint 설치 순서를 설명합니다. | self rows 제거, reverse 상태 reconciliation, deterministic survivor 선택 뒤 directional unique를 canonical expression index와 distinct-user CHECK로 교체합니다. |
| Atomic PostgreSQL transition | `77c555aba9a0` | insert/repeat/reverse 상태 전이를 한 statement에서 추적합니다. | unordered expression conflict가 한 row를 선택하고 reverse pending인 경우만 accepted로 갱신합니다. 같은 방향 repeat는 ID·status·timestamp를 유지합니다. |
| Memory semantic parity | `34db79005f30` | 관계 원본과 caller projection을 분리합니다. | memory는 requester/addressee ID를 원본 record로 저장하고 actor별 other-user projection을 매번 구성합니다. DB lock과 durability는 제공하지 않습니다. |
| Regression evidence | `cdaca35ccf7f` | public result와 raw row constraint를 연결합니다. | 두 backend에서 self/repeat/reverse/list scenario를 검증하고 PostgreSQL table의 한 accepted row 및 CHECK 위반을 직접 확인합니다. simultaneous friendship race는 직접 실행하지 않습니다. |

## 7. Failure → Fix → Test 관계

| 관계 | Failure / 이전 가정 | Fix / 결정 | Test / 근거 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 1 | A→B와 B→A가 별도 row가 되고 self row도 허용될 수 있었습니다. | `ffb0a8275a4f`가 legacy data를 정리하고 canonical unique/CHECK를 설치했습니다. | `cdaca35ccf7f`가 raw table one-row와 self constraint rejection을 확인합니다. | DB가 unordered identity의 최종 enforcement owner가 됩니다. migration은 기존 데이터를 deterministic하게 수렴시킨 뒤 제약을 설치합니다. |
| 2 | 초기 request code는 directional conflict만 처리하고 reverse pending의 의미를 여러 operation에 남겼습니다. | `77c555aba9a0`이 canonical conflict target과 conditional update로 한 statement에 넣었습니다. | `cdaca35ccf7f`가 same ID, pending→accepted, 양쪽 projection을 확인합니다. | unique index와 upsert가 같은 canonical expression을 사용해야 conflict detection과 state transition이 일치합니다. |
| 3 | memory `FriendSummary[]`는 requester/addressee를 잃어 PostgreSQL 의미를 재현하지 못했습니다. | `34db79005f30`이 `MemoryFriendship`을 도입했습니다. | `cdaca35ccf7f`의 memory scenario입니다. | 원본 relation state와 caller-specific read model을 분리해 parity를 복구하지만 process-local mutation이라는 한계는 남습니다. |

## 8. Ownership·상태·책임 변화

| 구간 | 이전 소유자/표현 | 이후 소유자/표현 | 관련 SHA | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 관계 identity | 요청 방향 또는 caller-facing summary가 identity를 사실상 정의했습니다. | DB canonical expression과 memory unordered lookup이 pair identity를 정의합니다. | `ffb0a8275a4f`, `34db79005f30` | ID 하나가 양쪽 사용자에게 공유되고 projection만 actor에 따라 달라집니다. |
| 상태 전이 | insert·accept·list 조합에 분산됐습니다. | PostgreSQL request upsert가 reverse pending transition을 원자적으로 소유합니다. | `77c555aba9a0` | DB statement completion 시점에 row ID/status가 결정되며 caller는 `RETURNING` 결과를 소비합니다. |
| Authorization | memory accept가 actor identity를 보존하지 못했습니다. | 두 backend가 addressee ID를 확인합니다. | `645e5a3c8e96`, `34db79005f30` | PostgreSQL은 `WHERE addressee_id=userId`, memory는 record field 비교로 unauthorized accept를 실패시킵니다. |
| Legacy data | reverse/self rows가 이미 존재할 수 있었습니다. | migration이 cleanup·survivor selection·constraint install을 소유합니다. | `ffb0a8275a4f` | application writer를 바꾸기 전에 저장된 데이터가 새 invariant를 만족하도록 정규화됩니다. |

## 9. Thread 최종 상태

최종 상태에서 friendship은 requester/addressee 순서와 무관한 사용자 pair 하나로 식별됩니다. PostgreSQL은 distinct-user CHECK와 canonical expression unique index를 최종 enforcement owner로 사용하고, request는 한 upsert에서 반복·역방향 상태를 결정합니다. memory는 같은 관계 record와 actor-specific projection을 구현합니다. 시험은 주요 sequence와 raw constraint를 검증하지만 simultaneous friendship request 부하나 multi-process memory consistency까지 증명하지 않습니다.

## 10. 최종 실행 흐름

1. caller가 requester ID와 상대 handle로 `requestFriend`를 호출하고 repository가 상대 user를 찾은 뒤 self 요청을 거부합니다.
2. PostgreSQL은 pending insert를 시도하고 canonical pair conflict 시 existing/excluded 방향을 비교합니다.
3. same-direction repeat는 기존 row를 유지하고 reverse pending은 같은 row ID에서 accepted로 전이합니다.
4. memory는 unordered pair를 배열에서 찾아 같은 상태 규칙을 process-local object에 적용합니다.
5. `listFriends(userId)`는 관계 원본에서 other user ID를 계산해 actor별 `FriendSummary`를 만듭니다.
6. regression test는 public 결과, 양쪽 view, raw PostgreSQL row와 self CHECK를 서로 연결합니다.

## 11. 학습 완료 확인

- [x] directional unique와 canonical unordered unique의 차이를 SQL expression으로 설명할 수 있습니다.
- [x] migration의 self 삭제·상태 reconciliation·survivor rank·constraint 설치 순서를 설명할 수 있습니다.
- [x] same-direction repeat와 reverse pending upsert의 `CASE` 조건을 설명할 수 있습니다.
- [x] memory relationship 원본과 caller-specific projection을 구분할 수 있습니다.
- [x] `cdaca35ccf7f`가 friendship에 대해 실제 simultaneous race를 증명하지 않는다는 점을 명시할 수 있습니다.

## 12. 실행 및 증거 기록

- 저장소 실행 시험: 실행하지 않았습니다.
- 이유: 로컬 `git clone --branch web/ft_transcendence --single-branch`가 DNS 해석 실패(`Could not resolve host: github.com`)로 중단되어 의존성을 포함한 실행 가능한 checkout을 만들 수 없었습니다.
- 코드 근거: 지정 브랜치의 source classification과 각 exact SHA의 GitHub commit diff를 확인했습니다. 따라서 본 문서의 시험 설명은 실제 시험 코드의 정적 검토 결과이며, 이 환경에서의 통과 결과가 아닙니다.
===== END FILE: 04-canonical-friendship-and-concurrent-requests.md =====

===== BEGIN FILE: 05-tournament-admission-and-capacity-concurrency.md =====
# Development Thread 05 — Tournament admission과 capacity concurrency

## 1. 학습 목표

- 초기 count→seed→insert 구현의 race와 memory capacity 차이를 exact SHA에서 확인합니다.
- legacy seed normalization과 unique constraint가 capacity control과 다른 불변식임을 구분합니다.
- tournament row lock을 기준으로 admission·status·bracket를 한 transaction에 묶는 이유를 설명합니다.
- memory canonical user source 정렬과 PostgreSQL concurrency evidence의 범위 차이를 구분합니다.

## 2. 범위와 경계

- 포함: tournament CRUD의 최초 admission, seed unique migration, PostgreSQL locked transaction, memory entrant source, final-slot concurrency regression.
- `cdaca35ccf7f`에서는 tournament capacity assertion만 이 Thread의 핵심 근거로 사용합니다. friendship assertion은 Thread 4에서 다룹니다.
- 제외: tournament match start/completion, final creation, match finalization과 rollback은 독립 tournament/realtime persistence story입니다.
- 제외: UI join mutation, browser cache, HTTP authorization은 web/auth category에서 다룹니다.

## 3. 핵심 질문

- 왜 unique seed constraint만으로 exactly one final-slot admission을 보장할 수 없습니까?
- row lock은 어떤 row를 잠그며 왜 같은 tournament의 모든 admission이 그 row를 먼저 읽어야 합니까?
- existing-entry check를 capacity check보다 먼저 두면 rejoin semantics가 어떻게 달라집니까?
- entry insert·status running·semifinal bracket가 같은 transaction이어야 하는 이유는 무엇입니까?
- commit 후 aggregate 재조회 실패는 durable admission과 API 응답 사이에 어떤 non-guarantee를 남깁니까?

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags |
| ---: | --- | --- | :---: | --- |
| 1 | `9b1dabcc4bb4` | `feat(db): 토너먼트 참가 저장 구현` | B | PERSISTENCE, TOURNAMENT |
| 2 | `3aa5958bb967` | `feat(db): tournament seed 제약 추가` | B | PERSISTENCE, TOURNAMENT |
| 3 | `d9a6d8dd8950` | `feat(db): PostgreSQL tournament 참가를 원자화` | A | PERSISTENCE, TOURNAMENT, RISK |
| 4 | `efdb5c3a4932` | `feat(db): memory tournament 참가자 원본 검증` | B | PERSISTENCE, TOURNAMENT |
| 5 | `cdaca35ccf7f` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | PERSISTENCE, TOURNAMENT, RISK |

## 5. Commit별 조사

### 5.1. `9b1dabcc4bb4` — feat(db): 토너먼트 참가 저장 구현

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `9b1dabcc4bb4` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | tournament list/create/join을 처음 repository에 추가하지만 count→seed→insert가 분리되어 capacity와 concurrency를 보호하지 못합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `AppRepository`의 `listTournaments`, `createTournament`, `joinTournament` signature를 확인합니다.
- PostgreSQL `createTournament`가 tournament insert 뒤 creator를 `joinTournament`로 참가시키는 순서를 추적합니다.
- 초기 `joinTournament`의 `select count(*)`, `seed = count + 1`, `insert ... on conflict (tournament_id, user_id) do nothing`을 확인합니다.
- count와 insert 사이에 row lock/transaction이 없고 capacity check도 없는 parent 상태를 명시합니다.
- `tournamentFromRow`가 entries를 seed 순으로 읽고 mapper에 넘기는지 확인합니다.
- memory 구현이 중복 user를 막지만 capacity 이상에서도 status만 running으로 바꾸고 신규 참가를 차단하지 않는지 확인합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | tournament table과 row type은 있었지만 repository를 통해 대회를 생성·조회·참가할 operation이 없었습니다. |
| 해결하려던 문제 | HTTP/tournament UI가 raw SQL을 모르고 creator 자동 참가와 참가자 목록을 일관된 aggregate로 받아야 했습니다. |
| 핵심 결정 | 공통 interface에 list/create/join을 추가하고 PostgreSQL은 count 기반 seed insert, memory는 entries 배열 append로 구현했습니다. |
| 입력 → 상태 전이 → 출력 | create → tournament row 저장 → creator join; join → 현재 entry count 조회 → seed=count+1 계산 → unique user insert → aggregate 재조회입니다. |
| ownership / lifetime / cleanup | PostgreSQL이 rows를, memory tournament object가 entries 배열을 소유합니다. create caller가 별도 transaction 없이 두 operation을 연속 호출합니다. |
| failure / rollback / retry | 동시 join이 같은 count/seed를 볼 수 있고 capacity를 초과할 수 있습니다. create 후 creator join이 실패하면 빈 tournament row가 남을 수 있습니다. memory도 신규 참가를 capacity에서 거부하지 않습니다. |
| 보장하는 것 | tournament aggregate의 기본 CRUD와 동일 user 재join의 row-level no-duplicate를 제공합니다. |
| 보장하지 않는 것 | seed uniqueness, capacity enforcement, final slot serialization, status/bracket atomicity, backend 의미 parity는 보장하지 않습니다. |
| 후속 연결 | `3aa5958bb967`이 seed unique를 DB에 추가하고 `d9a6d8dd8950`이 row lock transaction으로 admission 전체를 원자화합니다. |

#### 비교 기준

- parent 상태와 `9b1dabcc4bb4`의 diff를 먼저 비교합니다.
- 후속 관련 SHA `3aa5958bb967`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.2. `3aa5958bb967` — feat(db): tournament seed 제약 추가

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `3aa5958bb967` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | 기존 entry seed를 deterministic하게 재번호 매긴 뒤 tournament별 seed uniqueness를 DB constraint로 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/migrations/004_friendship_tournament_invariants.sql`의 `ranked_entries` CTE를 확인합니다.
- `partition by tournament_id order by seed, created_at, id`의 deterministic rank 기준을 추적합니다.
- 모든 기존 entry의 seed를 `row_number()` 결과로 갱신해 gap/duplicate를 정규화하는지 확인합니다.
- `unique (tournament_id, seed)` constraint 추가를 확인합니다.
- 이 constraint가 capacity나 user uniqueness를 새로 보장하지 않고 seed collision만 막는다는 점을 구분합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 초기 join은 `count+1`을 transaction 밖에서 계산해 동일 tournament에서 같은 seed가 생성될 수 있었고 DB는 이를 허용했습니다. |
| 해결하려던 문제 | 새 unique constraint를 바로 추가하면 기존 duplicate/gap seed 때문에 migration이 실패할 수 있었습니다. |
| 핵심 결정 | 각 tournament entry를 기존 seed·created_at·id 순으로 rank해 1부터 재번호 매긴 뒤 `(tournament_id, seed)` unique를 설치했습니다. |
| 입력 → 상태 전이 → 출력 | 기존 entries → tournament별 deterministic sort/rank → seed rewrite → unique constraint install → 이후 duplicate seed insert 거부입니다. |
| ownership / lifetime / cleanup | migration이 legacy seed normalization을, PostgreSQL constraint가 future seed uniqueness를 소유합니다. |
| failure / rollback / retry | 동시 writer는 duplicate seed error를 받을 수 있지만 이 commit만으로 retry하거나 다른 seed를 선택하지 않습니다. capacity도 확인하지 않습니다. |
| 보장하는 것 | 한 tournament 안에서 seed가 유일하고 migration 직후 contiguous 1..N 상태로 정규화됩니다. |
| 보장하지 않는 것 | 최대 N=capacity, 신규 참가 serialization, creator auto-join atomicity, bracket 생성은 보장하지 않습니다. |
| 후속 연결 | `d9a6d8dd8950`이 locked transaction 안에서 `max(seed)+1`을 계산해 constraint violation을 예방하고 capacity/status/bracket를 함께 갱신합니다. |

#### 최소 코드 근거

- `004_friendship_tournament_invariants.sql` — tournament별 `row_number()`로 기존 seed를 재번호 매긴 뒤 `(tournament_id, seed)` unique를 설치합니다.

#### 비교 기준

- parent 상태와 `3aa5958bb967`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `9b1dabcc4bb4`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `d9a6d8dd8950`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.3. `d9a6d8dd8950` — feat(db): PostgreSQL tournament 참가를 원자화

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `d9a6d8dd8950` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| Source role | tournament row lock을 중심으로 duplicate check, capacity, seed, status와 semifinal bracket 생성을 하나의 transaction에 묶습니다. |

#### 해당 SHA에서 확인할 실제 코드

- `PostgresRepository.joinTournament`의 `this.db.transaction().execute` 범위를 확인합니다.
- `select capacity from tournaments where id = ... for update`가 동일 tournament admission을 직렬화하는지 추적합니다.
- lock 획득 뒤 existing `(tournament_id,user_id)` 확인이 rejoin을 idempotent no-op으로 만드는지 확인합니다.
- `count(*)`와 `coalesce(max(seed),0)+1`이 같은 transaction/lock 아래 계산되는지 확인합니다.
- capacity 초과 시 insert 전에 `tournament full`을 던져 transaction rollback되는지 확인합니다.
- 마지막 자리에서 status=`running`과 `ensureTournamentBracket(tournamentId, transaction)`가 같은 executor를 사용하는지 확인합니다.
- transaction commit 뒤 aggregate를 재조회하는 부분이 lock snapshot 밖이라는 non-guarantee를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | 초기 admission은 count, seed, insert, status, bracket를 별도 statement로 실행해 동시 요청이 같은 마지막 자리를 모두 통과할 수 있었습니다. |
| 해결하려던 문제 | seed unique만 추가하면 race가 constraint error로 바뀔 뿐, 정확히 한 참가자를 받고 status/bracket까지 일관되게 전이하는 domain 결과는 보장하지 못했습니다. |
| 핵심 결정 | tournament row를 `FOR UPDATE`로 잠그는 transaction 안에서 existing check, count/next_seed, capacity check, insert, running status와 bracket 생성을 수행했습니다. |
| 입력 → 상태 전이 → 출력 | transaction 시작 → tournament row lock → same user existing이면 no-op → count/max seed 조회 → full이면 throw → insert → count가 capacity에 도달하면 status running·semifinal 2개 upsert → commit → aggregate 재조회입니다. |
| ownership / lifetime / cleanup | PostgreSQL transaction이 해당 tournament admission state와 bracket side effect를 commit/rollback 단위로 소유합니다. helper는 전달받은 transaction executor를 사용합니다. |
| failure / rollback / retry | 없는 tournament는 `firstRow` failure, full은 insert 전 throw로 rollback됩니다. bracket insert 실패도 entry/status와 함께 rollback됩니다. commit 뒤 summary read가 실패하면 durable admission은 이미 완료됐을 수 있습니다. |
| 보장하는 것 | 동일 tournament의 concurrent admissions가 row lock에서 직렬화되고 exactly one final slot, unique seed, capacity, running+bracket 전이가 한 transaction으로 결정됩니다. |
| 보장하지 않는 것 | 여러 tournament 사이의 global ordering, fairness, long lock wait policy, deadlock retry, commit 후 read availability는 보장하지 않습니다. |
| 후속 연결 | `efdb5c3a4932`가 memory entrant source를 canonical store로 정렬하고 `cdaca35ccf7f`가 10개 concurrent final-slot 요청으로 PostgreSQL 결과를 검증합니다. |

#### 최소 코드 근거

- `PostgresRepository.joinTournament` — `SELECT ... FOR UPDATE`로 tournament row를 잠근 transaction 안에서 count·next seed·capacity·entry·status·bracket를 처리합니다.

#### 비교 기준

- parent 상태와 `d9a6d8dd8950`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `3aa5958bb967`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `efdb5c3a4932`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.4. `efdb5c3a4932` — feat(db): memory tournament 참가자 원본 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `efdb5c3a4932` |
| Importance | B |
| Tags | PERSISTENCE, TOURNAMENT |
| Source role | memory join이 public lookup 결과가 아니라 canonical user Map 원본 존재를 먼저 검증한 뒤 projection을 생성하도록 바꿉니다. |

#### 해당 SHA에서 확인할 실제 코드

- `MemoryRepository.joinTournament`의 `getUserById` 호출이 `this.users.get(userId)`로 바뀐 parent diff를 확인합니다.
- `rawUser` 존재 확인 뒤 `toPublicUser(rawUser, true)`를 생성하는 순서를 추적합니다.
- 기존 joined/full/idempotent/status/bracket logic이 그대로 유지되는지 확인합니다.
- canonical store 존재성 확인이 database row lock과 같은 concurrency mechanism은 아니라는 점을 명시합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | memory join은 public lookup method를 통해 entrant를 얻어 저장 원본 존재 검증과 projection 생성이 한 호출에 숨겨졌습니다. |
| 해결하려던 문제 | repository 내부 mutation은 canonical user store의 원본을 기준으로 해야 하며, public read-model helper의 후속 filtering 변화에 영향을 받지 않아야 했습니다. |
| 핵심 결정 | `this.users.get(userId)`로 원본을 확인하고 그 row에서 `toPublicUser` projection을 만든 뒤 기존 admission 규칙을 적용했습니다. |
| 입력 → 상태 전이 → 출력 | tournament lookup + canonical user Map lookup → public projection 생성 → existing/full 검사 → entries mutation과 status/bracket update입니다. |
| ownership / lifetime / cleanup | memory user Map이 entrant identity의 source of truth를 소유하고 tournament aggregate는 projection을 entries로 보유합니다. |
| failure / rollback / retry | 없는 tournament/user는 동일 오류로 실패합니다. method 중간 mutation rollback이나 true parallel lock은 추가되지 않습니다. |
| 보장하는 것 | memory admission이 public lookup policy가 아니라 실제 repository 원본 사용자 존재에 의존합니다. |
| 보장하지 않는 것 | PostgreSQL과 같은 durability·transaction·row lock, stale projection 자동 갱신, multi-process capacity는 보장하지 않습니다. |
| 후속 연결 | `cdaca35ccf7f`가 memory final-slot scenario에서 capacity·unique entries·bracket와 rejoin idempotence를 검증합니다. |

#### 비교 기준

- parent 상태와 `efdb5c3a4932`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `d9a6d8dd8950`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.
- 후속 관련 SHA `cdaca35ccf7f`가 이 결정의 부족한 점을 보완하거나 검증하는지 확인합니다.

### 5.5. `cdaca35ccf7f` — test(db): friendship와 tournament 경쟁 상태 검증

| 항목 | 고정 정보 |
| --- | --- |
| SHA | `cdaca35ccf7f` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| Source role | 같은 test commit의 tournament 부분에서 10개 concurrent 요청이 마지막 한 자리를 두 backend에서 어떻게 결정하는지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/src/index.test.ts`의 `admits one of ten users into the final tournament slot` memory scenario를 확인합니다.
- creator+두 early entries로 3/4 상태를 만든 뒤 `Promise.allSettled`로 후보 10명의 `joinTournament`를 호출하는지 확인합니다.
- fulfilled 1, rejected 9, 모든 rejection에 `tournament full`, playerCount 4, unique user 4, semifinal slots [1,2] assertion을 확인합니다.
- accepted user의 재join이 성공하고 count/bracket가 늘지 않는 idempotence assertion을 확인합니다.
- `packages/db/src/postgres.integration.test.ts`의 동일 concurrent scenario와 raw entry seed [1,2,3,4], 두 match row assertion을 확인합니다.
- memory Promise concurrency와 PostgreSQL row-lock concurrency의 실제 의미 차이를 기록합니다.

#### 학습자 기록

| 항목 | 기록 |
| --- | --- |
| 직전 관련 상태 | `d9a6d8dd8950`이 transaction을 도입했지만 마지막 자리 경쟁에서 exactly one admission과 bracket side effect가 실제 SQL path에서 유지되는지 고가치 regression이 필요했습니다. |
| 해결하려던 문제 | 10개 요청이 모두 count=3을 보거나 duplicate seed/bracket를 만들면 capacity·status·read model이 서로 모순될 수 있었습니다. |
| 핵심 결정 | memory와 PostgreSQL 모두 3/4 상태를 만든 뒤 후보 10명을 `Promise.allSettled`로 동시에 요청하고 public aggregate와 raw rows를 확인했습니다. |
| 입력 → 상태 전이 → 출력 | creator 자동 entry + early 2명 → 10 candidate join promises → 1 fulfilled/9 full → entries 4·unique IDs·seeds 1..4 → semifinal slots 1,2 → accepted user rejoin 후 count/matches 불변입니다. |
| ownership / lifetime / cleanup | PostgreSQL transaction/row lock이 final-slot winner를 소유하고 test harness가 schema/pool cleanup을 소유합니다. memory는 event-loop 내 method mutation 순서가 결과를 결정합니다. |
| failure / rollback / retry | full 요청은 entry/status/bracket를 남기지 않아야 합니다. PostgreSQL direct query가 committed state를 확인합니다. test 자체가 fairness나 어떤 candidate가 이길지는 고정하지 않습니다. |
| 보장하는 것 | PostgreSQL에서 concurrent final-slot 요청 중 정확히 하나만 commit되고 seed·entry·bracket가 일관됨을 검증합니다. memory도 observable capacity/idempotence 결과를 맞춥니다. |
| 보장하지 않는 것 | 높은 부하의 lock wait/deadlock, 여러 API process의 retry policy, createTournament creator join atomicity, 본 세션의 실제 test 통과는 증명하지 않습니다. |
| 후속 연결 | count-before-insert 구현으로 회귀하거나 bracket helper가 transaction 밖으로 이동하면 success/rejection 수와 raw row assertion이 실패합니다. |

#### 최소 코드 근거

- `postgres.integration.test.ts` — 10개의 final-slot 요청 뒤 raw `tournament_entries` seed `[1,2,3,4]`와 정확히 두 semifinal row를 확인합니다.

#### Test commit 학습 기록

| 항목 | 기록 |
| --- | --- |
| 검증 대상 불변식 | tournament admission은 capacity를 넘지 않고 final slot 하나만 배정하며 seed와 semifinal bracket가 중복 없이 같은 committed state를 반영해야 합니다. |
| 재현한 실패·경계 | 3/4 tournament에 10명의 신규 user가 동시에 참가하고 성공한 user가 다시 참가하는 경계입니다. |
| 시험 기법 | `Promise.allSettled` 기반 concurrent repository test와 Testcontainers PostgreSQL raw-state verification입니다. |
| 통과하는 실제 코드 경로 | `joinTournament` transaction, tournament row `FOR UPDATE`, entry insert, status update, `ensureTournamentBracket`, memory equivalent입니다. |
| 시험이 증명하는 것 | PostgreSQL에서 1 success/9 full, entries 4·seeds 1..4·unique users, semifinals 2개와 rejoin idempotence를 증명합니다. |
| 시험이 증명하지 않는 것 | winner fairness, long-running lock behavior, deadlock retry, production load, 이 세션에서의 실제 실행 결과는 증명하지 않습니다. |
| 막으려는 회귀 | over-capacity admission, duplicate seed, duplicate/missing bracket, failed request의 partial commit, rejoin side effect 회귀를 막습니다. |

#### 비교 기준

- parent 상태와 `cdaca35ccf7f`의 diff를 먼저 비교합니다.
- 이 Thread의 직전 관련 SHA `efdb5c3a4932`와 책임·상태·보장 범위가 어떻게 달라졌는지 비교합니다.

## 6. 불변식 변화

| 단계 | 관련 SHA | 조사 초점 | 학습자 기록 |
| --- | --- | --- | --- |
| 초기 non-atomic admission | `9b1dabcc4bb4` | count·seed·insert·capacity의 분리 상태를 기록합니다. | PostgreSQL은 count+1을 transaction 밖에서 계산하고 capacity를 확인하지 않았으며 memory도 capacity 이후 신규 entry를 막지 않았습니다. 동일 user unique만 존재했습니다. |
| Seed uniqueness | `3aa5958bb967` | legacy normalization과 future constraint를 구분합니다. | 기존 entry를 deterministic 1..N으로 재번호 매긴 뒤 tournament별 seed unique를 설치합니다. duplicate seed는 막지만 final-slot winner를 선택하거나 capacity를 강제하지 않습니다. |
| Atomic admission | `d9a6d8dd8950` | lock·idempotence·capacity·status·bracket transaction을 추적합니다. | tournament row `FOR UPDATE`가 동일 tournament의 admission을 직렬화하고 existing user no-op, count/max seed, full reject, insert, running+bracket가 하나의 commit/rollback 단위가 됩니다. |
| Memory source alignment | `efdb5c3a4932` | entrant identity source와 projection을 분리합니다. | memory는 canonical user Map 원본 존재를 검증한 뒤 public projection을 entries에 넣습니다. 이것은 source alignment이지 DB-style lock이나 rollback이 아닙니다. |
| Final-slot evidence | `cdaca35ccf7f` | success/rejection 수와 raw state를 연결합니다. | 10개 요청 중 1개만 성공하고 9개는 full, entries/seeds/users가 4개로 유일하며 semifinal 2개와 rejoin no-op을 memory·PostgreSQL에서 확인합니다. |

## 7. Failure → Fix → Test 관계

| 관계 | Failure / 이전 가정 | Fix / 결정 | Test / 근거 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 1 | `9b1dabcc4bb4`의 count+1과 insert가 분리돼 같은 seed·over-capacity race가 가능했습니다. | `3aa5958bb967`이 seed unique를 추가했지만 collision을 domain result로 해결하지는 못했습니다. | constraint 자체는 `cdaca35ccf7f`의 raw seed assertion에서 후속 보호됩니다. | DB constraint는 최소 무결성 safety net이며 admission winner 선택과 status/bracket transition은 transaction logic이 별도로 소유해야 합니다. |
| 2 | capacity check·entry·running status·bracket가 분리되면 partial state가 남을 수 있었습니다. | `d9a6d8dd8950`이 tournament row lock transaction에 모두 묶었습니다. | `cdaca35ccf7f`가 10-way final-slot competition과 raw committed state를 검증합니다. | row lock을 모든 admission의 첫 serialization point로 사용해 exactly one final slot을 만들고 helper도 같은 transaction executor를 사용합니다. |
| 3 | memory join이 public lookup helper에 entrant source를 의존했습니다. | `efdb5c3a4932`가 canonical user Map을 직접 확인합니다. | `cdaca35ccf7f`의 memory capacity·unique-entry·bracket scenario입니다. | memory의 identity source는 정렬되지만 event-loop 순서에 따른 process-local mutation일 뿐 PostgreSQL의 concurrency mechanism과 같다고 볼 수 없습니다. |

## 8. Ownership·상태·책임 변화

| 구간 | 이전 소유자/표현 | 이후 소유자/표현 | 관련 SHA | 학습자 기록 |
| --- | --- | --- | --- | --- |
| Seed identity | application의 stale count가 seed를 사실상 결정했습니다. | DB unique constraint와 locked transaction의 `max(seed)+1`이 결정합니다. | `3aa5958bb967`, `d9a6d8dd8950` | constraint는 collision을 거부하고 transaction은 collision이 발생하지 않도록 serialized state에서 다음 seed를 계산합니다. |
| Capacity decision | 각 request가 독립 count를 읽고 신규 참가를 허용했습니다. | 잠긴 tournament row 아래 transaction이 capacity와 final-slot winner를 소유합니다. | `d9a6d8dd8950` | 같은 tournament admission은 lock을 통과해야 하며 다른 tournament는 서로 독립적으로 진행할 수 있습니다. |
| Bracket side effect | entry/status/bracket가 별도 statement/executor일 수 있었습니다. | 마지막 admission transaction이 running status와 semifinal rows를 함께 소유합니다. | `d9a6d8dd8950` | bracket 생성 실패는 entry와 status도 rollback되어 partial running tournament를 막습니다. |
| Memory entrant | public lookup output이 source처럼 사용됐습니다. | canonical user Map이 identity source, mapper가 entry projection을 소유합니다. | `efdb5c3a4932` | 원본 존재성과 read model 생성이 분리되지만 stored projection freshness는 별도 문제입니다. |

## 9. Thread 최종 상태

최종 상태에서 PostgreSQL tournament admission은 tournament row lock을 얻은 transaction 하나가 existing-entry idempotence, capacity, next seed, entry insert, running status와 semifinal bracket를 결정합니다. seed unique constraint가 DB-level safety net을 제공합니다. memory는 canonical user source와 같은 observable capacity/idempotence를 구현하지만 durable transaction이나 cross-process serialization은 없습니다. concurrency regression은 final slot 결과를 직접 검증하되 fairness·deadlock retry·production 부하까지 보장하지 않습니다.

## 10. 최종 실행 흐름

1. caller가 tournament ID와 user ID로 `joinTournament`를 호출합니다.
2. PostgreSQL transaction이 tournament row를 `FOR UPDATE`로 읽어 같은 tournament admission의 serialization point를 확보합니다.
3. 이미 참가한 user면 capacity와 무관하게 no-op하고 기존 aggregate를 반환할 준비를 합니다.
4. 신규 user면 current count와 max seed를 읽고 full이면 insert 전에 실패합니다.
5. 자리 있으면 next seed로 entry를 넣고 capacity에 도달한 요청이 status를 running으로 바꾸며 같은 transaction에서 semifinal 두 개를 생성합니다.
6. commit 뒤 aggregate를 재조회해 caller에 반환합니다. 이 read 실패는 committed write를 되돌리지 않습니다.
7. memory는 같은 분기를 canonical user Map과 tournament object에 적용하고 test는 10-way final-slot 결과와 raw PostgreSQL rows를 비교합니다.

## 11. 학습 완료 확인

- [x] seed uniqueness와 capacity serialization이 서로 다른 불변식임을 설명할 수 있습니다.
- [x] `FOR UPDATE` row와 모든 admission의 lock 순서를 설명할 수 있습니다.
- [x] existing-entry check가 full check보다 먼저인 rejoin 의미를 설명할 수 있습니다.
- [x] entry·status·bracket가 같은 transaction이어야 하는 failure/rollback 이유를 설명할 수 있습니다.
- [x] commit 후 summary read failure의 non-guarantee를 설명할 수 있습니다.
- [x] memory Promise test와 실제 PostgreSQL row-lock concurrency evidence를 구분할 수 있습니다.

## 12. 실행 및 증거 기록

- 저장소 실행 시험: 실행하지 않았습니다.
- 이유: 로컬 `git clone --branch web/ft_transcendence --single-branch`가 DNS 해석 실패(`Could not resolve host: github.com`)로 중단되어 의존성을 포함한 실행 가능한 checkout을 만들 수 없었습니다.
- 코드 근거: 지정 브랜치의 source classification과 각 exact SHA의 GitHub commit diff를 확인했습니다. 따라서 본 문서의 시험 설명은 실제 시험 코드의 정적 검토 결과이며, 이 환경에서의 통과 결과가 아닙니다.
===== END FILE: 05-tournament-admission-and-capacity-concurrency.md =====

===== BEGIN FILE: README.md =====
# 영속성·데이터 무결성

AppRepository 추상화, memory/PostgreSQL parity, migration·seed lifecycle, row mapping,
friendship canonical identity, tournament admission concurrency와 destructive test reset
안전성을 다룹니다.

## 범위

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Category: `02-persistence-and-data-integrity`
- 상태: Phase 1 감사 완료 후 동결된 scaffold
- 제품 전달 제외: Docker/Caddy/Compose image, 배포 job, release artifact, dependency
  보안 패치와 media asset 생성은 이 카테고리의 학습 대상에서 제외합니다.

## Category 감사 결론

- 카테고리 경계는 적절합니다. 영속성의 공통 repository·migration·row-mapping
  기반과 데이터 무결성 경쟁 상태만 포함합니다.
- 5개 Thread를 유지합니다. match finalization, 인증 session·WebSocket ticket,
  guest 격리, chat room 무결성, admin audit atomicity와 tournament match progression은
  독립된 engineering story이므로 다른 카테고리의 책임으로 남깁니다.
- Thread 1에는 `035b97ca7c58`, `6b661420e060`을 추가해 가짜 `bestStreak`
  구현의 fix와 regression test를 연결하고, `e935054ce0c9`를 추가해
  PostgreSQL integration 실행 경계를 포함했습니다.
- Thread 2에서는 `e1a0316fbe84`, `5cac4843fd9b`를 2026년 1월의 reset
  guard 작업보다 앞에 배치해 실제 branch history 순서를 복구했습니다.
- `9b62117a6909`, `d8050004e4ce`, `d2329e8dfc1d`,
  `7926b1366993`, `1b60b0a79963` 등 C-level formatting-only DB
  refactor는 독립적인 상태·책임 변화가 없어 Thread 3에 추가하지 않았습니다.
- 같은 `cdaca35ccf7f`는 friendship identity와 tournament capacity를 한 시험 파일에서
  함께 검증하므로 Thread 4와 Thread 5에서 각 불변식 관점으로 교차 참조합니다.

## Thread

1. [Repository 추상화·backend parity·read model](01-repository-abstraction-backend-parity-and-read-models.md)
2. [Migration·seed·readiness·reset lifecycle](02-migration-seed-readiness-and-reset-lifecycle.md)
3. [Row mapping과 backend contract 정렬](03-row-mapping-and-backend-contract-alignment.md)
4. [Canonical friendship과 동시 요청](04-canonical-friendship-and-concurrent-requests.md)
5. [Tournament admission과 capacity concurrency](05-tournament-admission-and-capacity-concurrency.md)

## 사용 원칙

- 각 문서의 Commit map 순서를 유지합니다.
- exact SHA의 코드와 parent 상태를 확인합니다.
- 다른 카테고리에서 같은 SHA를 교차 참조하더라도 이 문서의 질문에 맞는 근거만 기록합니다.
- Phase 2는 이 디렉터리의 동결 이후 scaffold를 수정하지 않고 completed counterpart만 채웁니다.
===== END FILE: README.md =====

