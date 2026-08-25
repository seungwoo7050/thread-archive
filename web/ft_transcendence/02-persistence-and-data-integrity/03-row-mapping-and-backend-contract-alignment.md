# Thread: Row mapping과 backend contract 정렬 — 저장 표현과 조회 표현을 분리하기

## 개요

이 Thread는 새로운 기능보다 이미 존재하는 저장소 코드를 **어떤 타입이 무엇을 표현하는지 드러나는 상태**로 정리합니다. PostgreSQL row, join으로 확장된 row, memory 내부 record, API DTO가 한 파일 안에서 비슷한 모양으로 섞여 있으면 TypeScript가 통과해도 다음 문제가 남습니다.

- write command의 필드가 그대로 memory 저장 record로 퍼져 의도하지 않은 속성까지 보존될 수 있습니다.
- DB row와 관계를 조립한 aggregate가 같은 mapper 인수로 뭉쳐 nullable 관계와 변환 책임이 흐려집니다.
- tournament match를 찾을 때마다 aggregate와 child를 따로 탐색해 서로 다른 객체를 수정할 위험이 생깁니다.
- mapper가 snake_case, `Date`, nullable relation을 어떻게 바꾸는지 독립적으로 검증하기 어렵습니다.

11개 commit 모두 중요도 B입니다. 실제 동시성·무결성 규칙을 새로 만드는 Thread가 아니라, 뒤의 변경이 잘못된 표현을 통과하지 못하도록 저장·조립·출력 경계를 명시하는 정리 작업이기 때문입니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `73b8ce0f0c26` | `refactor(db): repository user projection 타입 정렬` | B | `AUTH, PERSISTENCE` | memory 사용자 저장값을 DB와 공유하는 canonical projection으로 맞춥니다. |
| 2 | `3d0ae79affd5` | `refactor(db): memory match record 계약 정렬` | B | `PERSISTENCE` | write command 전체 대신 실제 보존 필드만 갖는 match record를 정의합니다. |
| 3 | `3e3f21129369` | `refactor(db): canonical row schema 타입 정렬` | B | `PERSISTENCE, TOURNAMENT` | table enum·Selectable row·join row의 공통 용어를 schema module에 모읍니다. |
| 4 | `212650b2863d` | `refactor(db): row mapper record 타입 정렬` | B | `PERSISTENCE, TOURNAMENT` | tournament match record view와 mapper의 숫자/null 변환을 명시합니다. |
| 5 | `45144a3719bc` | `refactor(db): dashboard와 friendship 조회 경계 정렬` | B | `PERSISTENCE` | 조건부 SQL fragment를 두 개의 명시적 query path로 나눕니다. |
| 6 | `ce41a880d6c6` | `refactor(db): PostgreSQL tournament helper와 admin 경계 정렬` | B | `PERSISTENCE, TOURNAMENT` | tournament child 조립 helper와 admin method 영역을 분명히 배치합니다. |
| 7 | `5c8659ea233b` | `refactor(db): tournament relation mapper 계약 정렬` | B | `PERSISTENCE, TOURNAMENT` | entries·matches·winner를 하나의 명시적 related object로 mapper에 넘깁니다. |
| 8 | `f77e317de4c1` | `refactor(db): memory match completion과 admin 경계 정렬` | B | `REALTIME, PERSISTENCE, TOURNAMENT` | memory aggregate와 match를 함께 찾는 helper를 도입합니다. |
| 9 | `9d64ea406b03` | `refactor(db): memory tournament 확정 경계 정렬` | B | `PERSISTENCE, TOURNAMENT` | match finalization도 같은 aggregate lookup 결과를 재사용합니다. |
| 10 | `b34fdaa1e9c2` | `refactor(db): memory chat과 tournament 진입 경계 정렬` | B | `PERSISTENCE, TOURNAMENT` | memory DTO 생성과 tournament 진입 메서드를 명시적 타입·helper로 정리합니다. |
| 11 | `dc0e60e6aa35` | `test(db): database row mapping contract 검증` | B | `PERSISTENCE, TOURNAMENT, TEST` | 순수 row mapper의 column 변환·관계 조립·날짜 직렬화를 고정합니다. |

## 1. Memory 저장값을 canonical projection으로 맞추기

### `73b8ce0f0c26` — 별도 `MemoryUserRow`를 제거

Memory repository의 user map이 독자적인 row type 대신 `UserProjectionRow`를 저장합니다.

```diff
-class MemoryRepository implements AppRepository {
-  private readonly users = new Map<string, MemoryUserRow>();
+class MemoryRepository implements AppRepository {
+  private readonly users = new Map<string, UserProjectionRow>();
```

새 사용자를 만들 때도 객체가 실제 projection 필드를 모두 명시합니다.

```ts
const user: UserProjectionRow = existing ?? {
  id: randomUUID(),
  email: input.email ?? `${handle}@dev.pong-pong.local`,
  handle,
  display_name: input.displayName,
  avatar_key: avatarFor(handle),
  role: "user",
  status: "active",
  rating: 1200,
  wins: 0,
  losses: 0,
  is_npc: false
};
```

이 변경은 memory가 PostgreSQL row 전체를 흉내 내도록 만들지 않습니다. `created_at`, `banned_at`처럼 memory read model에 필요 없는 column은 projection에 없습니다. 대신 `toPublicUser`·`toSessionUser`가 받는 최소 입력 shape를 두 backend가 공유합니다.

### `3d0ae79affd5` — write command와 stored record를 분리

기존 memory match는 `CreateMatchInput` 전체를 spread하고 `ended_at`을 추가했습니다. 이 방식은 command가 확장될 때 저장할 필요가 없는 option까지 record에 묻어 들어갈 수 있습니다.

```diff
-type MemoryMatchRecord = CreateMatchInput & {
-  id: string;
-  ended_at: string;
-  resultKey: string;
-};
+type MemoryMatchRecord = {
+  id: string;
+  resultKey: string;
+  mode: MatchMode;
+  winnerId: string | null;
+  loserId: string | null;
+  scoreLeft: number;
+  scoreRight: number;
+  endedAt: string;
+};
```

삽입도 spread 대신 필드별 복사로 바뀝니다.

```ts
this.matches.push({
  id: matchId,
  resultKey: command.resultKey,
  mode: command.mode,
  winnerId: command.winnerId,
  loserId: command.loserId,
  scoreLeft: command.scoreLeft,
  scoreRight: command.scoreRight,
  endedAt: new Date().toISOString()
});
```

이제 command는 입력 계약, `MemoryMatchRecord`는 저장 계약입니다. `memoryMatchSummary`는 `endedAt`을 그대로 사용하므로 내부 record에 SQL식 `ended_at` 이름을 유지할 이유도 사라집니다.

## 2. Canonical schema type과 mapper 입력을 좁히기

### `3e3f21129369` — table enum과 row type을 schema module에 모음

`schema.ts`는 shared type을 inline `import(...)`로 반복하는 대신 명시적으로 가져오고, DB가 사용하는 좁은 literal type을 이름 붙입니다.

```ts
export type TournamentRound = "semifinal" | "final";
export type TournamentMatchStatus =
  | "pending"
  | "ready"
  | "running"
  | "finished";
export type ChatScope = "lobby" | "match";
export type AdminAction = "ban" | "unban";
```

`Database` table map과 `Selectable` row alias도 한곳에서 정리됩니다.

```ts
export type UserRow = Selectable<UserTable>;
export type MatchRow = Selectable<MatchTable>;
export type ChatMessageRow = Selectable<ChatMessageTable>;
export type TournamentRow = Selectable<TournamentTable>;
export type TournamentMatchRow = Selectable<TournamentMatchTable>;
export type AdminActionRow = Selectable<AdminActionTable>;
```

이 타입은 runtime DB constraint가 아닙니다. 예를 들어 실제 column에 정의되지 않은 문자열이 직접 들어오는 것을 TypeScript alias만으로 막지는 못합니다. 역할은 query·mapper 코드가 임의 객체가 아니라 canonical row 이름을 사용하도록 만드는 것입니다.

### `212650b2863d` — DB row와 service record를 구분

Tournament match를 realtime/service code에 넘길 때 DB column 이름 전체가 필요하지 않으므로 별도 view를 둡니다.

```ts
export interface TournamentMatchRecordView {
  id: string;
  tournamentId: string;
  round: TournamentMatchRow["round"];
  slot: number;
  status: TournamentMatchRow["status"];
  leftUserId: string | null;
  rightUserId: string | null;
  winnerId: string | null;
}
```

같은 commit에서 score mapper는 nullable 여부를 보존한 뒤 숫자로 정규화합니다.

```diff
- scoreLeft: row.score_left,
- scoreRight: row.score_right,
+ scoreLeft: row.score_left == null ? null : Number(row.score_left),
+ scoreRight: row.score_right == null ? null : Number(row.score_right),
```

`null`과 `0`을 혼동하지 않으면서 driver가 number-like 값을 반환하는 경우에도 API DTO는 숫자로 고정됩니다.

## 3. Query 결과와 aggregate 조립 책임을 명시

### `45144a3719bc` — 조건부 SQL 조각 대신 두 query path

`listRecentMatches(userId?)`는 이전에 `where` fragment를 변수로 만들고 하나의 raw SQL template에 끼웠습니다. 이 commit은 user filter가 있는 경우와 없는 경우를 별도 query로 작성합니다.

```ts
const result = userId
  ? await sql<MatchWithHandlesRow>`
      select m.*, winner.handle as winner_handle, loser.handle as loser_handle
      from matches m
      left join users winner on winner.id = m.winner_id
      left join users loser on loser.id = m.loser_id
      where m.winner_id = ${userId} or m.loser_id = ${userId}
      order by m.ended_at desc
      limit 8
    `.execute(this.db)
  : await sql<MatchWithHandlesRow>`
      select m.*, winner.handle as winner_handle, loser.handle as loser_handle
      from matches m
      left join users winner on winner.id = m.winner_id
      left join users loser on loser.id = m.loser_id
      order by m.ended_at desc
      limit 8
    `.execute(this.db);
```

동작 의미는 유지됩니다. 얻는 것은 각 query가 완전한 SQL 문장으로 보이고 반환 row type이 branch별로 동일하다는 점입니다. 같은 diff의 dashboard 객체 줄바꿈과 friendship join 줄바꿈은 별도의 상태 변화가 아니므로 확대하지 않습니다.

### `ce41a880d6c6` — helper 위치와 조립 단위를 분명히 함

PostgreSQL repository에서 admin methods를 한 영역에 모으고, tournament match row 하나를 사용자 relation과 함께 변환하는 `tournamentMatchFromRow` helper를 추출합니다.

```ts
private async tournamentMatchFromRow(
  row: TournamentMatchRow
): Promise<TournamentMatchSummary> {
  return toTournamentMatchSummary(row, {
    left: row.left_user_id ? await this.getUserById(row.left_user_id) : null,
    right: row.right_user_id ? await this.getUserById(row.right_user_id) : null,
    winner: row.winner_id ? await this.getUserById(row.winner_id) : null
  });
}
```

이 commit은 query 수나 transaction 경계를 바꾸지 않습니다. relation을 조회해 nullable user로 만드는 책임이 repository helper에 있고, mapper는 이미 준비된 relation만 조립한다는 구분을 드러냅니다.

### `5c8659ea233b` — tournament mapper가 완성된 related data를 요구

기존 mapper는 entries와 matches를 위치 인수로 받고 winner는 호출 뒤 결과 객체에 대입했습니다. 변경 후 caller가 세 relation을 모두 준비해야 합니다.

```ts
return toTournamentSummary(row, {
  entries: entries.rows.map((entry) => toPublicUser(entry, true)),
  matches: await Promise.all(
    matches.rows.map((match) => this.tournamentMatchFromRow(match))
  ),
  winner: row.winner_id ? await this.getUserById(row.winner_id) : null
});
```

```ts
export function toTournamentSummary(
  row: TournamentWithCreatorRow,
  related: {
    entries: PublicUser[];
    matches: TournamentMatchSummary[];
    winner: PublicUser | null;
  }
): TournamentSummary {
  return {
    id: row.id,
    name: row.name,
    status: row.status,
    createdBy: toPublicUser({
      id: row.creator_id,
      email: row.email,
      handle: row.handle,
      display_name: row.display_name,
      avatar_key: row.avatar_key,
      role: row.role,
      status: row.user_status,
      rating: row.rating,
      wins: row.wins,
      losses: row.losses,
      is_npc: row.is_npc
    }),
    playerCount: related.entries.length,
    capacity: Number(row.capacity),
    winner: related.winner,
    entries: related.entries,
    matches: related.matches
  };
}
```

`{ ...row, id: creator_id, status: user_status }` 같은 overwrite trick도 사라지고 creator projection이 필드별로 만들어집니다. 동일 이름의 column이 여러 relation에서 합쳐질 때 어떤 값을 선택했는지가 코드에 남습니다.

`ensureTournamentBracket`의 executor type도 `Kysely<Database> | Transaction<Database>`로 넓혀, 일반 connection과 transaction 중 어느 문맥에서 실행 가능한지 명시합니다. 실제 transaction 도입은 Thread 5의 원자적 참가 처리에서 설명합니다.

## 4. Memory aggregate를 한 번 찾고 같은 객체를 수정

### `f77e317de4c1` — aggregate와 child를 함께 반환

Memory tournament match 완료는 이전에 tournament를 먼저 찾고 그 안에서 match를 다시 찾았습니다. helper는 둘을 같은 탐색 결과로 묶습니다.

```ts
private findTournamentMatch(
  matchId: string
): { tournament: TournamentSummary; match: TournamentMatchSummary } | null {
  for (const tournament of this.tournaments) {
    const match = tournament.matches.find((item) => item.id === matchId);
    if (match) return { tournament, match };
  }
  return null;
}
```

`completeTournamentMatch`는 `found.match`를 수정하고, final이면 `found.tournament`의 상태와 winner를 수정합니다. 서로 다른 탐색 결과를 조합하지 않으므로 child가 속한 aggregate가 명시적으로 유지됩니다.

같은 commit의 admin method 정렬은 actor를 한 번 조회한 뒤 action record에 넣는 형태로 펼칩니다. 새로운 admin 정책을 추가한 변경은 아닙니다.

### `9d64ea406b03` — match finalization도 같은 lookup contract 사용

Memory match finalization의 tournament link 탐색도 `findTournamentMatch()`로 교체됩니다.

```ts
const tournament = command.tournament
  ? this.findTournamentMatch(command.tournament.tournamentMatchId)
  : null;

if (command.tournament && !tournament) {
  throw new Error("tournament match not found");
}
if (tournament?.match.matchId) {
  throw new Error("tournament match already finalized");
}
```

참가자 검증과 완료 상태 반영은 같은 `{ tournament, match }` 객체를 사용합니다. 이 refactor가 finalization의 원자성을 새로 보장하는 것은 아닙니다. 단지 어떤 aggregate가 수정되는지 representation을 일치시킵니다.

### `b34fdaa1e9c2` — memory DTO와 진입 경계의 명시적 타입

Chat message와 tournament 생성 결과에 `ChatMessage`, `TournamentSummary` 타입을 직접 붙이고, `getTournamentMatch`·`startTournamentMatch`도 공통 lookup helper를 사용합니다.

```ts
const message: ChatMessage = {
  id: randomUUID(),
  scope: input.scope,
  roomId: input.roomId ?? null,
  sender,
  body: input.body,
  createdAt: new Date().toISOString()
};
```

```ts
const match = this.findTournamentMatch(matchId)?.match;
if (!match) return null;
return {
  id: match.id,
  tournamentId: match.tournamentId,
  round: match.round,
  slot: match.slot,
  status: match.status,
  leftUserId: match.left?.id ?? null,
  rightUserId: match.right?.id ?? null,
  winnerId: match.winner?.id ?? null
};
```

`alreadyJoined` 같은 이름과 block 확장은 읽기 경계를 분명히 하지만 capacity 동시성 자체를 해결하지는 않습니다. 해당 문제는 Thread 5의 transaction과 lock에서 다룹니다.

## 5. `dc0e60e6aa35` — mapper를 DB 없이 독립 검증

새 시험은 실제 query를 실행하지 않고 canonical row와 joined row를 직접 구성합니다. 다음 변환을 고정합니다.

- `display_name` → `displayName`, `avatar_key` → `avatarKey`
- public user에서 email·DB timestamp를 노출하지 않음
- `Date` → ISO string
- match의 사용자 관점 win/loss·opponent·rating delta
- friendship/chat joined row의 relation 조립
- tournament match record의 camelCase와 nullable ID
- tournament aggregate의 creator·entries·matches·winner
- admin action의 nullable actor/target

```ts
expect(toPublicUser(userRow, true)).toEqual({
  id: USER_ID,
  handle: "typed-player",
  displayName: "타입 선수",
  avatarKey: "blue",
  role: "user",
  status: "active",
  rating: 1342,
  wins: 12,
  losses: 7,
  online: true,
  isNpc: false
});
```

```ts
expect(toTournamentMatchRecord(match)).toEqual({
  id: MATCH_ID,
  tournamentId: TOURNAMENT_ID,
  round: "semifinal",
  slot: 1,
  status: "ready",
  leftUserId: USER_ID,
  rightUserId: OTHER_USER_ID,
  winnerId: null
});
```

이 시험은 pure mapper contract에는 강한 증거지만 다음을 증명하지 않습니다.

- SQL select alias가 실제로 이 row shape를 반환하는지
- PostgreSQL driver가 모든 column을 예상 타입으로 decode하는지
- memory와 PostgreSQL query의 정렬·limit·transaction 의미가 같은지
- runtime에 잘못된 row가 들어왔을 때 validation error가 발생하는지

## 최종 표현 흐름

```text
[PostgreSQL]
SQL table row / joined row
  -> schema.ts의 Selectable·join row type
  -> repository가 relation 조회와 nullable 처리
  -> rowMappers가 API DTO로 변환

[memory]
canonical UserProjectionRow / explicit MemoryMatchRecord
  -> aggregate lookup이 owner와 child를 함께 반환
  -> 동일 mapper 또는 명시적 DTO 생성
  -> AppRepository 반환형
```

## 보장 범위

| 보장 | 근거 | 비보장 |
| --- | --- | --- |
| user projection의 공통 최소 shape | `73b8ce0f0c26`, `3e3f21129369` | DB full row와 memory storage가 완전히 동일함 |
| write input과 memory stored match 분리 | `3d0ae79affd5` | runtime immutable record |
| tournament relation을 완성한 뒤 mapper 호출 | `5c8659ea233b` | N+1 query 제거나 transaction snapshot 일관성 |
| memory aggregate와 child의 동일 탐색 결과 사용 | `f77e317de4c1`, `9d64ea406b03`, `b34fdaa1e9c2` | 동시 mutation 제어 |
| row→DTO 순수 변환 회귀 | `dc0e60e6aa35` | SQL query·backend parity·runtime schema validation |

## 이 Thread의 경계

이 Thread에는 C-level formatting-only commit을 추가하지 않습니다. 포함된 commit도 줄바꿈·순서 이동 자체가 아니라 저장 record, row alias, mapper 인수, aggregate lookup처럼 실제 표현 책임이 달라진 부분만 설명합니다.

Friendship canonical identity, tournament capacity transaction, migration lifecycle, match finalization의 원자성은 각각 다른 Thread가 담당합니다.

> 검증 기록: 위 설명은 `web/ft_transcendence` branch의 표시된 exact SHA diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 typecheck와 mapper test를 실행하지 않았으므로, test source의 assertion을 실행 결과로 표현하지 않았습니다.
