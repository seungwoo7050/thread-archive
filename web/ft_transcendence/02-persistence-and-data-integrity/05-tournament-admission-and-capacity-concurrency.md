# Thread: Tournament admission — 마지막 한 자리와 대진 생성을 하나의 transaction으로 묶기

## 개요

토너먼트 참가에서 `count < capacity`를 확인하는 것만으로는 정원을 지킬 수 없습니다. 여러 요청이 동시에 같은 count를 읽으면 모두 자신이 마지막 자리라고 판단할 수 있고, 같은 seed를 계산하거나 정원을 초과해 insert할 수 있습니다. 마지막 참가자가 들어오는 순간 tournament status와 semifinal bracket도 함께 바뀌므로, entry 한 행만 원자적으로 넣어서는 충분하지 않습니다.

이 Thread는 다음 불변 조건을 만듭니다.

- 한 tournament 안에서 seed는 중복되지 않습니다.
- 이미 참가한 사용자의 재요청은 정원이 찬 뒤에도 idempotent하게 성공합니다.
- 신규 참가자는 capacity를 넘길 수 없습니다.
- 마지막 참가자의 entry, `running` 전환, 두 semifinal row 생성은 같은 transaction에서 commit되거나 함께 rollback됩니다.
- 동일 tournament의 참가 요청은 tournament row lock으로 직렬화됩니다.
- memory 구현은 canonical user record에서 참가 DTO를 만들고 같은 observable capacity 규칙을 유지합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9b1dabcc4bb4` | `feat(db): 토너먼트 참가 저장 구현` | B | `PERSISTENCE, TOURNAMENT` | 토너먼트 생성·목록·참가와 seed 배정의 첫 구현을 추가합니다. |
| 2 | `3aa5958bb967` | `feat(db): tournament seed 제약 추가` | B | `PERSISTENCE, TOURNAMENT` | 기존 seed를 다시 번호 매기고 `(tournament_id, seed)` unique를 설치합니다. |
| 3 | `d9a6d8dd8950` | `feat(db): PostgreSQL tournament 참가를 원자화` | A | `PERSISTENCE, TOURNAMENT, RISK` | row lock transaction 안에서 중복·정원·seed·status·bracket을 처리합니다. |
| 4 | `efdb5c3a4932` | `feat(db): memory tournament 참가자 원본 검증` | B | `PERSISTENCE, TOURNAMENT` | memory 참가 DTO를 파생 조회값이 아니라 canonical user record에서 만듭니다. |
| 5 | `cdaca35ccf7f` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | `PERSISTENCE, TOURNAMENT, RISK` | 마지막 한 자리에 10개 요청을 보내 capacity·seed·bracket·idempotency를 검증합니다. |

## 1. `9b1dabcc4bb4` — 참가 저장의 첫 구현과 경쟁 window

PostgreSQL은 토너먼트를 만든 뒤 creator를 첫 참가자로 넣고, 참가 시 현재 행 수에 1을 더해 seed를 정합니다.

```ts
async createTournament(input: {
  name: string;
  createdBy: string;
}): Promise<TournamentSummary> {
  const result = await sql<TournamentRow>`
    insert into tournaments (name, created_by, capacity)
    values (${input.name}, ${input.createdBy}, 4)
    returning *
  `.execute(this.db);

  await this.joinTournament(firstRow(result).id, input.createdBy);
  const tournaments = await this.listTournaments();
  return tournaments.find((item) => item.id === firstRow(result).id)
    ?? tournaments[0];
}
```

```ts
async joinTournament(
  tournamentId: string,
  userId: string
): Promise<TournamentSummary> {
  const count = await sql<{ count: string }>`
    select count(*)::text
    from tournament_entries
    where tournament_id = ${tournamentId}
  `.execute(this.db);

  await sql`
    insert into tournament_entries (tournament_id, user_id, seed)
    values (${tournamentId}, ${userId}, ${Number(firstRow(count).count) + 1})
    on conflict (tournament_id, user_id) do nothing
  `.execute(this.db);

  return (await this.listTournaments())
    .find((item) => item.id === tournamentId)!;
}
```

이 commit 자체에는 capacity 검사도 bracket 생성도 없습니다. `(tournament_id, user_id)` unique 덕분에 같은 사용자의 중복 행은 막지만, 서로 다른 사용자는 제한 없이 들어갈 수 있고 두 요청이 같은 count를 읽어 같은 seed를 계산할 수 있습니다.

Memory 구현은 creator를 entries에 직접 넣고, 중복 사용자가 아니면 배열에 추가한 뒤 `playerCount`와 status를 갱신합니다. single-process 호출의 기본 동작은 만들지만 실제 DB 경쟁과 같은 조건은 아닙니다.

> Thread는 연속 commit 묶음이 아닙니다. `d9a6d8dd8950`의 parent에는 이 사이의 다른 tournament 작업을 통해 capacity 사전 검사와 bracket helper가 이미 존재합니다. 아래 원자화 설명은 `9b1dabcc4bb4`를 현재 코드처럼 소급하지 않고, `d9a6d8dd8950`의 exact parent와 diff를 기준으로 합니다.

## 2. `3aa5958bb967` — seed를 DB identity의 일부로 만듦

Unique constraint를 추가하기 전에 기존 entry를 tournament별로 다시 번호 매깁니다.

```sql
with ranked_entries as (
  select
    id,
    row_number() over (
      partition by tournament_id
      order by seed asc, created_at asc, id asc
    )::integer as next_seed
  from tournament_entries
)
update tournament_entries as entry
set seed = ranked.next_seed
from ranked_entries as ranked
where entry.id = ranked.id;
```

정렬 기준은 기존 seed, 생성 시각, ID 순서입니다. 같은 seed가 이미 중복되어 있어도 결과는 tournament별 `1..N`으로 결정됩니다.

```sql
alter table tournament_entries
  add constraint tournament_entries_tournament_seed_unique
  unique (tournament_id, seed);
```

이 constraint가 보장하는 것은 **seed 중복 금지**입니다. capacity보다 많은 서로 다른 seed를 넣는 직접 SQL까지 막지는 않습니다. 정원은 application transaction이 소유합니다.

## 3. `d9a6d8dd8950` — preflight 검사에서 locked transaction으로

### 직전 상태의 문제

이 commit 직전 코드는 대략 다음 단계를 별도 query로 수행했습니다.

```text
entry count 읽기
  -> tournament capacity와 기존 참가 여부 읽기
  -> full 여부 판단
  -> seed = count + 1
  -> insert
  -> entry count를 다시 보고 status 갱신
  -> bracket 생성
```

각 query 사이에 다른 request가 commit될 수 있습니다. 예를 들어 참가자 3명, capacity 4인 상태에서 두 요청이 동시에 count 3을 읽으면 둘 다 seed 4와 빈자리 하나를 본다고 판단합니다. seed unique가 한 요청을 막을 수는 있지만, loser가 constraint error를 그대로 받게 되고 status/bracket 처리와 한 단위로 정의되지 않습니다.

### 결정: tournament row를 mutex처럼 잠금

```ts
await this.db.transaction().execute(async (transaction) => {
  const tournament = await sql<{ capacity: number }>`
    select capacity
    from tournaments
    where id = ${tournamentId}
    for update
  `.execute(transaction);

  const tournamentRow = firstRow(tournament);
  /* ... */
});
```

같은 tournament ID의 참가 transaction은 `FOR UPDATE` lock을 순서대로 획득합니다. 첫 transaction이 commit/rollback할 때까지 다음 요청은 최신 상태를 읽을 수 없습니다. 다른 tournament row는 별도 lock이므로 서로 독립적으로 진행할 수 있습니다.

### 중복 요청을 정원 검사보다 먼저 종료

```ts
const existing = await sql<{ id: string }>`
  select id
  from tournament_entries
  where tournament_id = ${tournamentId}
    and user_id = ${userId}
  limit 1
`.execute(transaction);

if (existing.rows[0]) return;
```

이미 참가한 사용자는 tournament가 가득 찼더라도 성공으로 처리됩니다. idempotent retry가 `tournament full`로 바뀌지 않는 중요한 순서입니다.

### Lock 안에서 count와 next seed를 함께 계산

```ts
const entryState = await sql<{
  count: number;
  next_seed: number;
}>`
  select
    count(*)::integer as count,
    (coalesce(max(seed), 0) + 1)::integer as next_seed
  from tournament_entries
  where tournament_id = ${tournamentId}
`.execute(transaction);

const state = firstRow(entryState);
if (Number(state.count) >= Number(tournamentRow.capacity)) {
  throw new Error("tournament full");
}
```

`count`는 정원 판정에, `max(seed)+1`은 새 seed에 사용됩니다. 기존 seed가 연속이라는 migration 전제를 유지하지만, 중간 seed가 사라져도 이미 사용 중인 값과 충돌하지 않는 다음 번호를 고릅니다.

### Entry·status·bracket을 같은 transaction에 포함

```ts
await sql`
  insert into tournament_entries (tournament_id, user_id, seed)
  values (${tournamentId}, ${userId}, ${state.next_seed})
`.execute(transaction);

const playerCount = Number(state.count) + 1;
if (playerCount >= Number(tournamentRow.capacity)) {
  await sql`
    update tournaments
    set status = 'running'
    where id = ${tournamentId}
  `.execute(transaction);

  await this.ensureTournamentBracket(tournamentId, transaction);
}
```

Bracket helper도 같은 executor를 받습니다.

```ts
private async ensureTournamentBracket(
  tournamentId: string,
  executor: Kysely<Database> = this.db
): Promise<void> {
  const entries = await sql<{ user_id: string; seed: number }>`
    select user_id, seed
    from tournament_entries
    where tournament_id = ${tournamentId}
    order by seed asc
  `.execute(executor);

  if (entries.rows.length < 4) return;

  await sql`
    insert into tournament_matches
      (tournament_id, round, slot, left_user_id, right_user_id, status)
    values
      (${tournamentId}, 'semifinal', 1,
       ${entries.rows[0].user_id}, ${entries.rows[3].user_id}, 'ready'),
      (${tournamentId}, 'semifinal', 2,
       ${entries.rows[1].user_id}, ${entries.rows[2].user_id}, 'ready')
    on conflict (tournament_id, round, slot) do nothing
  `.execute(executor);
}
```

마지막 insert나 bracket 생성이 실패하면 transaction 전체가 rollback되어 네 번째 entry와 `running` 상태만 남는 부분 성공을 만들지 않습니다. 성공 후 summary 조회는 transaction 밖에서 다시 수행하므로 반환 DTO는 commit된 상태를 읽습니다.

### 경쟁 요청의 terminal state

| 요청 종류 | Lock 획득 후 판단 | 결과 |
| --- | --- | --- |
| 이미 참가한 사용자 | existing row 발견 | mutation 없이 성공 |
| 신규 사용자, 자리 있음 | count < capacity | 새 seed로 insert |
| 마지막 신규 사용자 | insert 후 count == capacity | 같은 transaction에서 running + bracket |
| 신규 사용자, 이미 가득 참 | count >= capacity | `tournament full`, mutation 없음 |
| bracket 생성 실패 | transaction error | entry/status/bracket 모두 rollback |

## 4. `efdb5c3a4932` — memory entry의 원본을 canonical record로 고정

이 commit은 capacity algorithm을 바꾸지 않고 참가자 DTO의 출처를 바꿉니다.

```diff
 const tournament = this.tournaments.find((item) => item.id === tournamentId);
-const user = await this.getUserById(userId);
-if (!tournament || !user) throw new Error("tournament not found");
+const rawUser = this.users.get(userId);
+if (!tournament || !rawUser) throw new Error("tournament not found");
+const user = toPublicUser(rawUser, true);
```

`getUserById()`가 반환한 파생 read model을 다시 aggregate 내부 storage source로 쓰지 않고, canonical `users` map의 row를 검증한 뒤 tournament entry projection을 만듭니다. `online: true`도 PostgreSQL의 tournament entry mapping과 맞게 명시됩니다.

Memory는 transaction이나 lock을 제공하지 않습니다. JavaScript 한 process에서 이 메서드의 mutation 구간은 동기적으로 실행되므로 동일 event loop 안의 일반 호출에서는 순서대로 처리되지만, multi-process 공유 저장소나 실제 병렬 쓰기 안전성을 의미하지 않습니다.

## 5. `cdaca35ccf7f` — 마지막 한 자리에 10개 요청

공유 test commit의 tournament 부분만 이 Thread에 포함합니다.

### Memory case

Creator와 두 early entry로 참가자 3명을 만든 뒤 후보 10명의 `joinTournament()` promise를 `Promise.allSettled`로 모읍니다. 기대 결과는 다음과 같습니다.

- fulfilled 1개
- rejected 9개, 모두 `tournament full`
- 최종 `playerCount` 4
- entry ID 4개 모두 고유
- semifinal slot `[1, 2]`
- 실제로 들어간 후보가 다시 요청하면 `playerCount: 4`로 성공

Memory 메서드의 capacity 검사와 배열 mutation은 await 없이 동기적으로 진행되므로 이 case는 실제 interleaving race를 만들기보다 **같은 tick에 대량 요청을 시작했을 때의 순차/idempotent 결과**를 검증합니다.

### PostgreSQL case

같은 준비 상태에서 10개의 repository call을 동시에 시작합니다.

```ts
const attempts = await Promise.allSettled(
  candidates.map((candidate) =>
    repository.joinTournament(tournament.id, candidate.id)
  )
);
```

각 call은 별도 pool connection을 사용할 수 있고 같은 tournament row lock을 경쟁합니다. 시험은 API 결과뿐 아니라 table을 직접 확인합니다.

```ts
expect(accepted).toHaveLength(1);
expect(rejected).toHaveLength(9);

expect(entries.rows).toHaveLength(4);
expect(entries.rows.map((entry) => entry.seed)).toEqual([1, 2, 3, 4]);
expect(new Set(entries.rows.map((entry) => entry.user_id)).size).toBe(4);
expect(matches.rows).toEqual([
  { round: "semifinal", slot: 1 },
  { round: "semifinal", slot: 2 }
]);
```

성공한 후보를 다시 join한 뒤 entries 4, matches 2가 그대로인지도 검사합니다. 이 assertion은 “가득 찬 뒤의 idempotent retry”가 새 seed나 중복 bracket을 만들지 않는다는 것을 고정합니다.

## 최종 불변 조건

```text
create tournament
  -> creator가 seed 1로 참가

join(tournament, user)
  -> tournament row FOR UPDATE
  -> 이미 참가했으면 성공 반환
  -> count >= capacity면 신규 요청 거부
  -> max(seed)+1로 entry 삽입
  -> 마지막 자리면 같은 transaction에서
       status = running
       semifinal slot 1, 2 생성
  -> commit
  -> committed summary 재조회
```

| 불변 조건 | DB 방어 | Repository 방어 | 시험 증거 |
| --- | --- | --- | --- |
| 사용자당 한 entry | 기존 `(tournament_id, user_id)` unique | existing check로 idempotent return | 재요청 뒤 4행 유지 |
| seed 중복 금지 | `tournament_entries_tournament_seed_unique` | lock 안의 `max(seed)+1` | seed `[1,2,3,4]` |
| capacity 초과 금지 | 직접 capacity check constraint는 없음 | locked count와 capacity 비교 | 10개 중 1개만 성공 |
| 마지막 entry와 bracket 원자성 | transaction | 같은 executor로 status/bracket 실행 | 4 entries·2 semis |
| bracket 중복 금지 | round/slot conflict key | `on conflict ... do nothing` | 재요청 뒤 matches 2 |

## 보장하지 않는 것

- Repository를 우회한 raw SQL insert의 capacity 준수. DB는 seed/user 중복은 막지만 “행 수 ≤ capacity” 자체를 constraint로 표현하지 않습니다.
- tournament가 open 상태일 때만 참가할 수 있다는 별도 lifecycle 정책. 이 diff의 lock query는 capacity만 읽습니다.
- 여러 service instance가 memory repository 하나를 안전하게 공유하는 것.
- PostgreSQL deadlock, connection cancellation, lock timeout을 별도로 주입하는 시험.
- capacity가 4가 아닌 임의 크기에서 bracket을 일반화하는 것. helper는 4명과 semifinal 두 경기를 전제로 합니다.
- 참가 취소와 seed 재배치. migration은 기존 데이터를 한 번 정규화하지만 runtime leave flow는 이 Thread에 없습니다.

## 이 Thread의 경계

같은 `004_friendship_tournament_invariants.sql`의 friendship canonicalization은 Thread 4에서만 설명합니다. Tournament match 진행·결과 확정·winner propagation은 별도 Development Thread이며, 여기서는 admission과 bracket 생성 시점까지만 다룹니다.

> 검증 기록: 위 설명은 `web/ft_transcendence` branch의 표시된 exact SHA diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 Testcontainers 경쟁 시험을 실행하지 않았으며, test source가 선언한 동시 호출과 table assertion을 실행 결과로 과장하지 않았습니다.
