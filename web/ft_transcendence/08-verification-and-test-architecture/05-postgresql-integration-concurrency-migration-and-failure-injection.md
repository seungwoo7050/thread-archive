# Thread: PostgreSQL integration, 동시성, migration, 실패 주입

memory repository는 domain result를 빠르게 검증하는 데 유용하지만 PostgreSQL의 transaction, unique constraint, row lock, migration 순서와 rollback을 대신 증명할 수 없습니다. 이 Thread는 실제 PostgreSQL 16을 테스트 경계에 넣고, 고위험 persistence 규칙을 **관찰 가능한 row와 constraint 결과**로 검증하는 과정을 다룹니다.

```text
한 PostgreSQL container
        ↓
테스트마다 독립 schema + search_path
        ↓
실제 migration + repository operation
        ↓
동시 호출 / 중간 실패 / 이전 schema 상태
        ↓
DB row·constraint·rollback 직접 관찰
```

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `c43b87694b29bcee15291c6a752ecf8db77cb356` | `test(db): PostgreSQL integration 환경과 계약 추가` | A | `PERSISTENCE, RISK, TEST` | PostgreSQL 16 schema-per-test harness와 실패-safe cleanup 계약을 구축한다. |
| 2 | `582a1615a2c6de131dffee725f4a17e2223f86c8` | `test(db): 경기 결과 단일 확정 조건 검증` | A | `PERSISTENCE, TOURNAMENT, RISK` | 동일 result의 반복·동시 finalization이 한 durable effect로 수렴하는지 검증한다. |
| 3 | `cdaca35ccf7fa9f86e7a89c16686d852d210fdae` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | `PERSISTENCE, TOURNAMENT, RISK` | unordered friendship identity와 마지막 tournament slot 경쟁을 검증한다. |
| 4 | `0649b63a1ca9ee4fb171e5dfc0fdb35802747d14` | `test(db): 인증 migration 중 데이터 보존 검증` | A | `AUTH, PERSISTENCE, TEST` | 이전 schema에서 legacy session만 폐기하고 사용자·경기 기록을 보존하는지 검증한다. |
| 5 | `527b5f137425adadc486a6b071581f72c0005b16` | `test(db): test database reset guard 검증` | B | `PERSISTENCE, TEST` | destructive reset이 명시적 test target 하나에만 적용되는지 검증한다. |
| 6 | `d0137660cd9feb48e594a78b8db23eb8633d23aa` | `fix(db): 차단 감사 기록을 원자적으로 저장` | A | `AUTH, PERSISTENCE, RISK` | 사용자 차단 상태와 관리자 감사 row를 한 transaction으로 묶는다. |
| 7 | `9106abc10d0e9e191ed388d7879012664f0136bf` | `test(db): 차단 감사 기록 atomicity 검증` | A | `AUTH, PERSISTENCE, TEST` | 두 번째 write만 실제 constraint로 실패시켜 선행 update rollback을 확인한다. |

## 1. `c43b87694b29...` — 실제 DB 테스트도 자원 lifecycle을 가져야 한다

### 왜 shared database만으로 부족한가

여러 test가 같은 public schema를 쓰면 다음 문제가 생깁니다.

- 이전 test의 row가 다음 test의 unique constraint 결과를 바꿉니다.
- 한 test가 migration을 일부만 적용하면 다른 test의 schema 가정이 깨집니다.
- callback이 실패한 뒤 pool/schema가 남아 다음 test를 오염시킵니다.
- cleanup error가 원래 assertion failure를 덮어 root cause를 잃습니다.

반대로 test마다 container를 새로 띄우면 isolation은 강하지만 매우 느립니다. 이 commit은 한 PostgreSQL 16 Testcontainer를 session 동안 공유하고, 각 test에 schema namespace를 새로 만드는 절충안을 선택합니다.

### schema-per-test

schema 이름은 random UUID를 32자리 hex로 바꾼 안전한 형식입니다.

```ts
const schema = `test_${randomUUID().replaceAll("-", "")}`;
const databaseUrl = withSearchPath(container.getConnectionUri(), schema);
```

`withSearchPath`는 connection URL의 PostgreSQL `options`에 해당 schema만 넣습니다. migration table과 application table이 같은 namespace에 만들어지므로 각 test는 독립된 migration state를 가집니다.

```text
container: pong_pong_test
  ├─ test_a1...  ← test A의 tables + kysely_migration
  ├─ test_b2...  ← test B의 tables + kysely_migration
  └─ test_c3...  ← test C의 tables + kysely_migration
```

schema 이름은 `^test_[a-f0-9]{32}$`를 통과해야 quote됩니다. dynamic identifier를 임의 문자열로 조립하지 않는 안전장치입니다.

### tracked cleanup

context가 만든 pool과 repository는 cleanup stack에 등록됩니다.

```ts
openPool() {
  const isolatedPool = new Pool({ connectionString: databaseUrl, max: 2 });
  cleanupTasks.push(() => isolatedPool.end());
  return isolatedPool;
}

openRepository() {
  const repository = createPostgresRepository(databaseUrl);
  cleanupTasks.push(() => repository.close());
  return repository;
}
```

callback 종료 시 stack을 reverse하여 child resource부터 닫고 마지막에 schema를 `drop ... cascade`합니다. callback이 이미 실패한 경우 cleanup error가 그 원래 예외를 대체하지 않습니다. callback은 성공했지만 cleanup만 실패한 경우에는 `AggregateError`로 cleanup failure를 드러냅니다.

초기 integration suite 자체가 다음을 확인합니다.

- 빈 schema에 migration을 적용하고 재실행해도 migration 목록/table이 바뀌지 않음
- demo/development seed가 의도한 사용자만 만들고 반복 적용이 idempotent함
- 새 schema에서 이전 test의 user가 보이지 않음
- callback이 throw해도 schema와 backend connection이 사라짐
- temporary container callback이 실패해도 container가 stop되어 URI로 다시 연결할 수 없음

이 harness는 뒤의 모든 DB 테스트가 신뢰할 수 있는 전제입니다. test code도 production code처럼 획득한 자원을 끝까지 회수해야 합니다.

## 2. 같은 명령이 여러 번 실행되어도 durable effect는 하나다

### `582a1615a2c6...` — 20개 concurrent finalization

이 test는 memory와 PostgreSQL repository에 같은 logical command를 20번 동시에 전달합니다.

```ts
const results = await Promise.all(
  Array.from({ length: 20 }, () => repository.finalizeMatch(command))
);
```

모든 command는 같은 `resultKey`를 사용합니다. 기대 결과는 “함수가 한 번만 호출됨”이 아닙니다.

| 관찰 대상 | 기대 값 |
| --- | --- |
| 반환된 `matchId` 종류 | 1개 |
| `created: true` 응답 | 1개 |
| `matches` row | 1개 |
| winner wins/rating change | 한 번 |
| loser losses/rating change | 한 번 |
| `rating_history` | 두 participant에 각 1개 |

이것은 **exactly-once execution**이 아니라 **idempotent durable convergence**입니다. 여러 transaction이 시작될 수 있고 일부는 기존 result를 읽어 반환하지만, 외부에서 관찰되는 match/rating effect는 한 세트여야 합니다.

PostgreSQL test는 repository 응답만 믿지 않고 SQL로 rows를 다시 읽습니다.

```text
matches(result_key)
users(rating, wins, losses)
rating_history(rating_before, rating_after, delta)
```

따라서 implementation이 응답만 같은 ID로 속이고 실제 row를 중복 썼다면 실패합니다.

### tournament linkage 실패는 앞선 write까지 되돌려야 한다

같은 commit의 rollback case는 존재하지 않는 tournament match ID를 사용합니다. finalization 중 match row와 rating update가 먼저 실행된 뒤 linkage에서 실패할 수 있는 위치를 겨냥합니다.

실패 뒤 기대 state:

```text
matches(result_key) = 0
rating_history = 0
winner rating/wins = 원래 값
loser rating/losses = 원래 값
```

부분 row가 하나라도 남으면 repository method가 하나의 transaction boundary가 아니라 여러 독립 write의 묶음이라는 뜻입니다.

### concurrent semifinals는 final 하나로 수렴한다

두 semifinal result를 동시에 확정하면 양쪽 transaction이 “상대 semifinal도 끝났는가?”를 확인하고 final row를 만들려고 경쟁할 수 있습니다. 테스트는 다음을 직접 확인합니다.

- 두 semifinal row가 각각 올바른 match ID로 finished
- final row는 정확히 1개
- final의 left/right가 두 winner
- 전체 match 2개, rating history 4개, final bracket row 1개

이 case는 result idempotency와 다른 race입니다. 서로 다른 result key 두 개가 같은 tournament aggregate의 다음 상태를 만들기 때문입니다.

## 3. domain identity와 capacity는 요청 방향·순서보다 강해야 한다

### `cdaca35ccf7f...` — friendship canonical pair

친구 관계를 `(requester, addressee)` 방향 그대로 unique 처리하면 A→B와 B→A가 두 row가 될 수 있습니다. test는 다음 흐름을 memory와 PostgreSQL 양쪽에 적용합니다.

```text
A → A  : reject (self friendship)
A → B  : pending row 생성
A → B  : 같은 row 반환
B → A  : 같은 row를 accepted로 전환
```

최종 SQL 결과는 friendship row 하나입니다. PostgreSQL 쪽에서는 self pair를 직접 insert해 `friendships_distinct_users_check` constraint가 동작하는지도 확인합니다.

여기서 identity는 request 방향이 아니라 unordered user pair입니다.

```text
friendshipKey(A, B) == friendshipKey(B, A)
```

### 마지막 tournament slot 경쟁

creator와 두 early entry가 들어가 3/4 상태인 tournament에 10명이 동시에 join합니다.

```ts
const attempts = await Promise.allSettled(
  candidates.map((candidate) => repository.joinTournament(tournament.id, candidate.id))
);
```

기대 결과:

- fulfilled 1개
- `tournament full` rejection 9개
- 최종 entry 4개
- seed `[1, 2, 3, 4]`
- user ID 중복 없음
- semifinal row 정확히 2개
- 실제 입장에 성공한 사용자의 재요청은 idempotent하고 row 수가 늘지 않음

단순한 `count < 4` 조회 후 insert라면 10 transaction이 모두 3명을 보고 통과할 수 있습니다. 이 test가 요구하는 것은 capacity check와 insertion/bracket creation이 lock·constraint를 포함한 serialized boundary 안에 있다는 사실입니다.

memory implementation도 같은 domain 결과를 내야 하지만, 실제 row lock/constraint semantics의 증거는 PostgreSQL case입니다.

## 4. migration은 최종 schema에서 거꾸로 추측하지 않는다

### `0649b63a1ca9...` — migration 004의 실제 과거 상태 만들기

인증 transport 변경은 legacy session을 무효화해야 하지만 사용자·경기·rating history까지 손실해서는 안 됩니다. final HEAD schema를 새로 만든 뒤 “session이 없다”고 확인하는 것만으로는 migration이 기존 row를 어떻게 처리했는지 알 수 없습니다.

테스트는 migration을 의도적으로 `004_friendship_tournament_invariants`까지만 적용합니다.

```ts
await migrateDatabase(databaseUrl, "004_friendship_tournament_invariants");
```

그 상태에서 다음 데이터를 만듭니다.

- winner/loser user
- active legacy session
- finalized match
- rating history

users, matches, rating history projection을 snapshot한 뒤 나머지 migration을 적용합니다.

```text
migration 004 state
  ├─ users
  ├─ active session
  ├─ match
  └─ rating history
        ↓ apply migration 005
current state
  ├─ users             == before
  ├─ matches           == before
  ├─ rating_history    == before
  └─ sessions count    == 0
```

이는 “credential invalidation”과 “domain data preservation”을 한 테스트에서 분리해 확인합니다. migration 005가 목록에 기록되었는지도 검증합니다.

증명하지 않는 범위도 있습니다. 대규모 production table의 lock 시간이나 rollback/down migration은 다루지 않습니다.

## 5. destructive test helper는 fail-closed여야 한다

### `527b5f137425...` — reset target 검증

`resetTestDatabase`는 편리하지만 target parsing이 느슨하면 운영 schema를 drop할 수 있습니다. 이 commit은 reset을 실행하기 전에 다음 조건을 모두 검사합니다.

- `NODE_ENV === "test"`
- 일반 `DATABASE_URL`이 아니라 별도 `TEST_DATABASE_URL`
- PostgreSQL URL
- public schema를 지울 경우 database 이름이 정확히 명시된 test DB
- 또는 `test_<32hex>` 형식의 단일 isolated schema
- `search_path`에 public/other가 함께 있거나 임의 schema가 섞이면 거부

허용/거부 예시는 다음과 같습니다.

| target | 결과 |
| --- | --- |
| `pong_pong_test`, public | 허용 |
| application DB + `search_path=test_<32hex>` | 해당 isolated schema만 허용 |
| `pong_pong_test_backup` | 이름이 비슷해도 거부 |
| `search_path=test_x,public` | 모호하므로 거부 |
| production env | 거부 |

integration case는 target schema에 user를 만든 뒤 reset/remigrate하여 user count가 0이고 readiness가 migration current인지 확인합니다. 동시에 sibling schema에 marker table을 만들고 reset 뒤에도 남아 있는지 확인합니다.

이 commit은 B급 supporting guard지만 실패했을 때 영향은 큽니다. 중요도 체계는 기존 metadata대로 유지하되, destructive helper의 fail-closed 성격은 분명히 기록해야 합니다.

## 6. 감사 기록은 부가 로그가 아니라 상태 전이의 일부다

### 기존 split-write

`setUserBan`은 이전에 두 SQL을 독립적으로 실행했습니다.

```text
1. users.status / banned_at update
2. admin_actions insert
```

첫 번째가 commit되고 두 번째가 constraint·connection 오류로 실패하면 사용자는 차단되었지만 감사 row가 없습니다. authorization state와 accountability state가 서로 다른 사실을 말하게 됩니다.

### `d0137660cd9f...` — 한 transaction으로 묶기

실제 diff는 두 statement의 실행 대상을 `this.db`에서 동일 transaction handle로 바꿉니다.

```diff
- const result = await updateUser.execute(this.db);
- await insertAdminAction.execute(this.db);
- return toPublicUser(firstRow(result));
+ return this.db.transaction().execute(async (transaction) => {
+   const result = await updateUser.execute(transaction);
+   await insertAdminAction.execute(transaction);
+   return toPublicUser(firstRow(result));
+ });
```

수정된 불변 조건은 다음과 같습니다.

```text
user status transition
      +
admin audit record
      =
하나의 commit 또는 하나의 rollback
```

단순히 `Promise.all`로 묶거나 error를 catch해 user update를 되돌리는 보상 로직이 아니라 DB transaction이 원자성의 owner가 됩니다.

### `9106abc10d0e...` — 두 번째 statement만 실제 DB에서 실패시킨다

mock으로 `insert` 함수를 throw하게 만들면 transaction/driver가 실제로 rollback하는지는 알 수 없습니다. 이 test는 isolated schema의 `admin_actions`에 임시 CHECK constraint를 추가합니다.

```sql
alter table admin_actions
add constraint admin_actions_reject_test_reason
check (reason <> 'force audit failure');
```

그 뒤 정확히 해당 reason으로 `setUserBan`을 호출합니다. 실행 순서는 다음과 같습니다.

```text
transaction begin
  → users row를 banned로 update
  → admin_actions insert
       → CHECK constraint failure
  → transaction rollback
```

repository promise는 constraint name을 포함한 실제 PostgreSQL error로 reject되어야 합니다. 별도 SQL 조회 결과는 다음과 같아야 합니다.

```text
users.status = active
users.banned_at = null
admin_actions count = 0
```

이 test가 강한 이유는 first write가 실제로 성공 가능한 statement이고 second write만 database engine에서 실패하기 때문입니다. fix가 transaction 밖에 있거나 서로 다른 connection을 사용한다면 user row가 banned로 남습니다.

## 최종 persistence invariant

| 문제 | 실제 DB에서 관찰하는 기준 |
| --- | --- |
| test isolation | test별 schema·migration table, 실패 뒤 schema/backend 제거 |
| repeated finalization | 20 caller가 한 match ID·한 rating effect로 수렴 |
| tournament rollback | linkage failure 뒤 match/history/user 변경이 모두 없음 |
| concurrent semifinals | final bracket row 정확히 1개 |
| friendship identity | A→B와 B→A가 한 row |
| capacity race | 마지막 slot 경쟁에서 1 success / 9 full |
| auth migration | session만 제거되고 user/match/history projection 보존 |
| destructive reset | target schema만 reset, sibling schema 보존 |
| admin audit atomicity | audit insert 실패 시 user update도 rollback |

## 최종 실행 흐름

```text
[Testcontainers PostgreSQL 16 시작]
        ↓
[test_<32hex> schema 생성 + search_path 고정]
        ↓
[필요한 migration 시점까지 적용]
        ↓
[repository operation / concurrent calls / injected constraint failure]
        ↓
[repository response + 별도 SQL row/constraint 관찰]
        ↓
[pool/repository close → schema drop]
        ↓
[session 종료 시 container stop]
```

## 이 Thread의 경계

- GameHub가 같은 `resultKey`로 finalization을 retry하는 lifecycle은 Thread 04입니다.
- cookie/ticket handshake 자체는 Thread 02입니다.
- 실제 load에서 finalization duplicate metric을 보는 것은 Thread 07입니다.
- CI가 container suite를 어떤 job에서 실행하는지는 이 category의 범위 밖입니다.

## 조사·실행 기록

모든 설명은 표시된 SHA의 diff와 해당 시점 test/production source를 바탕으로 작성했습니다. 이 환경에서는 Docker/Testcontainers와 PostgreSQL integration suite를 실행하지 않았으므로 실제 통과 결과나 실행 시간을 주장하지 않습니다.
