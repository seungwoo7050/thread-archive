# Thread: 저장소 추상화와 조회 모델 — 공통 인터페이스에서 실제 backend 증거까지

## 개요

이 Thread는 초기 PostgreSQL schema가 TypeScript row type과 mapper를 거쳐 `AppRepository`로 감싸지고, 동일한 인터페이스를 구현하는 PostgreSQL·memory 저장소가 사용자·순위·경기·대시보드 조회를 제공하기까지의 과정을 다룹니다.

핵심은 “인터페이스가 같으면 동작도 같다”는 가정이 성립하지 않는다는 점입니다. 두 구현은 같은 메서드 이름과 반환형을 공유하지만, 초기에는 목록 제한과 온라인 표시 방식이 달랐고 `bestStreak`는 실제 경기 기록이 아닌 임의 공식과 상수로 만들어졌습니다. 마지막 두 단계는 이 차이를 실제 경기 순회로 고치고, memory 단위 시험과 Testcontainers 기반 PostgreSQL 통합 시험이 각각 무엇을 증명하는지 분리합니다.

### 최종적으로 남은 계약

- SQL column은 repository 밖으로 직접 노출되지 않고 mapper에서 API용 DTO로 변환됩니다.
- repository가 concrete connection/pool의 생성과 종료를 소유합니다.
- 최근 경기 목록은 최신순으로 반환하지만, 최고 연승은 시간순으로 다시 순회해 계산합니다.
- memory 시험은 빠른 동작 회귀를, PostgreSQL 통합 시험은 실제 migration·connection·schema cleanup을 검증합니다.
- 동일한 `AppRepository` 구현이라는 사실만으로 조회 제한이나 presence 의미까지 같다고 간주하지 않습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `0e850d24406e` | `feat(db): 초기 PostgreSQL schema 정의` | B | `PERSISTENCE, TOURNAMENT` | 핵심 엔터티와 관계를 durable relational model로 처음 고정합니다. |
| 2 | `4aa060c0b8df` | `feat(db): 사용자와 세션 row schema 정의` | B | `AUTH, PERSISTENCE` | SQL row와 public/session DTO 사이의 typed projection을 만듭니다. |
| 3 | `9277572765e7` | `feat(db): 저장소 lifecycle 구성` | B | `PERSISTENCE, OPERATIONS` | PostgreSQL pool·Kysely와 memory 구현을 하나의 repository lifetime으로 감쌉니다. |
| 4 | `fb516f723cdf` | `feat(db): 개발 사용자 seed 저장 구현` | B | `PERSISTENCE` | 반복 가능한 개발 사용자 upsert와 seed 경로를 추가합니다. |
| 5 | `c5b96a06925c` | `feat(db): 프로필 조회와 변경 저장 구현` | B | `PERSISTENCE` | 사용자 조회·프로필 변경·온라인 사용자 목록의 공통 surface를 확장합니다. |
| 6 | `0364c42f776b` | `feat(db): 순위 조회 구현` | B | `PERSISTENCE` | rating·승수·승률을 조합한 leaderboard read model을 추가합니다. |
| 7 | `8ab49e5f2dd4` | `feat(db): 경기 조회 row contract 정의` | B | `PERSISTENCE` | joined match row를 사용자 관점의 `MatchSummary`로 바꾸는 계약을 정의합니다. |
| 8 | `c7ea1ff241c8` | `feat(db): 최근 경기와 대시보드 조회 구현` | B | `PERSISTENCE` | 최근 경기와 대시보드를 조립하지만 잘못된 최고 연승 계산을 남깁니다. |
| 9 | `6509e32ba95d` | `test(db): 메모리 저장소 흐름 검증` | B | `PERSISTENCE, TOURNAMENT, TEST` | memory 구현의 사용자·세션·경기·대시보드 흐름을 빠르게 검증합니다. |
| 10 | `035b97ca7c58` | `fix(db): 최근 경기에서 최고 연승 계산` | B | `PERSISTENCE` | 임의 값을 제거하고 실제 경기 chronology에서 최고 연승을 계산합니다. |
| 11 | `6b661420e060` | `test(db): 최고 연승 계산 검증` | B | `PERSISTENCE, TEST` | 승·패가 섞인 기록으로 순서와 reset 규칙을 회귀 시험에 고정합니다. |
| 12 | `e935054ce0c9` | `build(db): PostgreSQL integration 의존성과 명령 추가` | B | `PERSISTENCE` | 실제 PostgreSQL 시험을 위한 Testcontainers 의존성과 실행 명령을 추가합니다. |
| 13 | `c43b87694b29` | `test(db): PostgreSQL integration 환경과 계약 추가` | A | `PERSISTENCE, RISK, TEST` | 격리 schema·connection cleanup·migration/seed 검증을 실제 PostgreSQL에서 수행합니다. |

## 1. SQL schema에서 repository lifetime까지

### `0e850d24406e` — 관계를 먼저 DB에 고정

`001_initial.sql`은 users, sessions, friendships, matches, chat messages, tournaments, entries, admin actions를 한 번에 도입합니다. 이 시점의 중요한 선택은 행의 존재뿐 아니라 삭제 수명과 초기 중복 기준을 DB가 소유하게 만든 것입니다.

```sql
create table if not exists sessions (
  token text primary key,
  user_id uuid not null references users(id) on delete cascade,
  expires_at timestamptz not null,
  created_at timestamptz not null default now()
);

create table if not exists friendships (
  id uuid primary key default gen_random_uuid(),
  requester_id uuid not null references users(id) on delete cascade,
  addressee_id uuid not null references users(id) on delete cascade,
  status text not null,
  unique (requester_id, addressee_id)
);

create table if not exists tournament_entries (
  id uuid primary key default gen_random_uuid(),
  tournament_id uuid not null references tournaments(id) on delete cascade,
  user_id uuid not null references users(id) on delete cascade,
  seed integer not null,
  unique (tournament_id, user_id)
);
```

이 schema가 보장하는 것은 **같은 방향의 친구 요청 중복**과 **같은 사용자의 같은 토너먼트 중복 참가**까지입니다. 반대 방향 친구 요청을 같은 관계로 보는 규칙, 자기 자신과의 친구 관계 금지, 토너먼트별 seed 중복 금지와 capacity 경쟁 제어는 아직 없습니다. 해당 문제는 Thread 4·5에서 별도로 닫힙니다.

### `4aa060c0b8df` — row와 외부 DTO를 분리

SQL column 이름과 API 속성 이름을 그대로 섞지 않고, Kysely row type과 mapper를 사이에 둡니다.

```ts
export type UserProjectionRow = Pick<
  UserRow,
  | "id"
  | "email"
  | "handle"
  | "display_name"
  | "avatar_key"
  | "role"
  | "status"
  | "rating"
  | "wins"
  | "losses"
>;

export function toPublicUser(row: UserProjectionRow, online = false): PublicUser {
  return {
    id: row.id,
    handle: row.handle,
    displayName: row.display_name,
    avatarKey: row.avatar_key,
    role: row.role,
    status: row.status,
    rating: Number(row.rating),
    wins: Number(row.wins),
    losses: Number(row.losses),
    online
  };
}
```

여기서 `online`은 DB column이 아닙니다. mapper 호출자가 공급하는 일시적 값입니다. 따라서 이 mapper는 column 이름 누출과 public/session 필드 혼합은 줄이지만, presence의 진실성이나 runtime row validation까지 보장하지 않습니다.

### `9277572765e7` — concrete 자원을 repository가 소유

`AppRepository`가 `close()`와 `ensureSeedData()`를 갖고, PostgreSQL factory는 pool과 Kysely instance를 함께 만들어 구현체에 넘깁니다. memory 구현은 같은 surface를 제공하되 외부 connection을 갖지 않습니다.

이 단계의 의미는 단순히 메서드 목록을 맞추는 데 있지 않습니다. API caller는 PostgreSQL인지 memory인지에 따라 pool을 직접 닫는 대신 repository lifetime만 다루게 됩니다. 반면 당시 `ensureSeedData()`는 schema 준비까지 함께 수행하므로 migration과 seed 책임이 아직 결합되어 있습니다. 이 결합은 Thread 2의 `f9bb622a1117`에서 분리됩니다.

## 2. 같은 인터페이스, 아직 다른 조회 의미

### `fb516f723cdf` — seed와 upsert

개발 사용자를 handle로 반복 upsert하고 rating·role 등의 시범 데이터를 채웁니다. PostgreSQL은 durable row를 갱신하고 memory는 내부 `Map`을 갱신하지만, caller는 동일한 `SessionUser`를 받습니다.

이 commit의 seed는 “운영 시작 시 반드시 있어야 하는 데이터”가 아니라 개발 편의 데이터입니다. 이후 seed profile 분리와 startup seed 제거가 필요한 이유도 여기에 있습니다.

### `c5b96a06925c`·`0364c42f776b` — 공통 surface가 parity를 자동으로 만들지는 않음

두 구현 모두 다음 기능을 갖게 됩니다.

- ID/handle로 사용자 조회
- display name·avatar 변경
- 온라인 사용자 목록
- rating·wins 기준 leaderboard와 승률 계산

하지만 실제 diff에는 의미 차이가 남습니다.

| 조회 | PostgreSQL | memory | 당시 결론 |
| --- | --- | --- | --- |
| 온라인 사용자 | rating 내림차순, `limit 12` | 전체 사용자를 정렬해 반환 | 반환형은 같아도 목록 크기가 다릅니다. |
| leaderboard | rating·wins 정렬, `limit 20` | 전체 사용자를 정렬해 반환 | pagination/limit 계약이 interface에 표현되지 않았습니다. |
| `online` 값 | 조회 mapper에서 `true` 전달 | 같은 방식으로 `true` 전달 | 실제 WebSocket presence가 아니라 조회 경로의 표시값입니다. |

따라서 “둘 다 `AppRepository`를 구현한다”는 사실은 compile-time surface만 맞춥니다. limit, 정렬 tie-break, presence authority 같은 의미는 별도의 시험이나 명시적 계약이 필요합니다.

## 3. 경기 row에서 대시보드 파생 지표까지

### `8ab49e5f2dd4` — 사용자 관점으로 경기 row 변환

`MatchWithHandlesRow`는 winner/loser handle을 join 결과에 추가하고 mapper는 선택된 `userId`를 기준으로 결과와 상대를 결정합니다.

```ts
const won = userId ? row.winner_id === userId : true;
return {
  id: row.id,
  mode: row.mode,
  opponentHandle: won ? row.loser_handle ?? "AI" : row.winner_handle ?? "AI",
  result: won ? "win" : "loss",
  ratingDelta: won ? Number(row.rating_delta) : -12,
  endedAt: row.ended_at.toISOString()
};
```

이 commit에서 패배 rating delta는 row의 반대값을 계산하지 않고 `-12`로 고정됩니다. 즉 mapper contract는 만들어졌지만 rating history 전체를 일반화한 모델은 아닙니다. 이 Thread는 해당 수치 정책을 확장하지 않고, 최근 경기와 연승 계산에 필요한 사용자 관점 결과만 다룹니다.

### `c7ea1ff241c8` — 최근 경기 순서는 맞지만 `bestStreak`는 가짜

PostgreSQL은 joined match를 `ended_at desc limit 8`로 읽고, memory는 저장 배열을 잘라 reverse하여 최신순 결과를 만듭니다. 문제는 대시보드가 이 실제 목록을 갖고도 최고 연승을 사용하지 않았다는 점입니다.

```ts
// PostgreSQL의 당시 구현
bestStreak: Math.max(1, Math.min(12, user.wins - user.losses + 3))

// memory의 당시 구현
bestStreak: 3
```

두 값 모두 경기의 연속성을 보지 않습니다. 총 승패만으로는 `승-승-패-승`과 `승-패-승-승`을 구분할 수 없으므로 최고 연승을 계산할 수 없습니다.

### `6509e32ba95d` — memory 흐름 시험의 범위

이 commit은 memory repository에서 사용자 생성, session, match, dashboard 등의 기본 동작을 실행합니다. 같은 diff에 friendship·tournament 시험도 섞여 있지만 이 Thread에서는 사용자·경기·대시보드 부분만 근거로 사용합니다.

이 시험은 빠르고 결정적이지만 PostgreSQL query, migration, connection 종료, timestamp/driver 변환을 통과하지 않습니다. 따라서 “repository 동작의 예시”이지 backend parity의 최종 증거는 아닙니다.

### `035b97ca7c58` — 최신순 표시와 시간순 계산을 분리

수정은 임의 값을 제거하고 `recentMatches` 자체를 순회합니다. 목록은 화면 표시를 위해 최신순이므로, 연속 기록은 복사본을 reverse해 과거에서 현재로 읽습니다.

```ts
function bestWinningStreak(matches: MatchSummary[]): number {
  let best = 0;
  let current = 0;

  for (const match of [...matches].reverse()) {
    if (match.result === "win") {
      current += 1;
      best = Math.max(best, current);
    } else {
      current = 0;
    }
  }
  return best;
}
```

이 함수가 고정하는 불변 조건은 두 가지입니다.

1. 패배를 만나면 현재 연승은 반드시 0으로 돌아갑니다.
2. API가 반환하는 최신순 배열을 변형하지 않고, 계산용 복사본만 시간순으로 뒤집습니다.

단, 계산 대상은 repository가 반환한 최근 8경기입니다. 사용자 전체 경기 이력의 역대 최고 연승을 보장하는 함수는 아닙니다.

### `6b661420e060` — 순서와 reset을 함께 고정

시험은 시간순으로 `승 → 패 → 승 → 승`을 저장합니다. 최근 경기 반환은 `승 → 승 → 패 → 승`이고, 최고 연승은 2여야 합니다. 단순히 승리 개수만 세거나 최신순을 그대로 누적하면 이 사례를 안정적으로 통과할 수 없습니다.

## 4. 실제 PostgreSQL을 통과하는 증거

### `e935054ce0c9` — 별도 integration 실행 경계

Testcontainers PostgreSQL 의존성과 `postgres-integration` 명령을 추가합니다. lockfile의 대량 변경은 이 Thread의 학습 대상이 아니며, 중요한 변화는 실제 PostgreSQL을 띄우는 시험이 일반 unit suite와 분리되었다는 점입니다.

### `c43b87694b29` — schema와 connection의 cleanup까지 시험

통합 fixture는 `postgres:16-alpine` container 하나를 시작하고, 각 test callback마다 UUID 기반 schema를 새로 만듭니다. repository와 pool을 열 때마다 cleanup task를 등록하고 역순으로 닫은 뒤 schema를 drop합니다.

```ts
const schema = `test_${randomUUID().replaceAll("-", "")}`;
const databaseUrl = withSearchPath(container.getConnectionUri(), schema);
const cleanupTasks: Array<() => Promise<void>> = [];

try {
  await migrateDatabase(databaseUrl);
  return await callback(context);
} finally {
  for (const cleanup of cleanupTasks.reverse()) {
    await cleanup();
  }
  await adminPool.query(`drop schema if exists ${quoteSchema(schema)} cascade`);
}
```

시험이 직접 확인하는 범위는 다음과 같습니다.

- 빈 schema에서 migration을 적용하고 다시 적용해도 table/migration 목록이 바뀌지 않음
- demo seed는 NPC만, development seed는 개발 사용자와 admin을 포함함
- 서로 다른 test schema 사이에 row가 보이지 않음
- callback이 실패해도 tracked connection이 닫히고 schema가 제거됨
- 임시 container callback이 실패해도 container가 중지됨

반대로 이 commit만으로 모든 `AppRepository` 메서드가 memory와 PostgreSQL에서 같은 결과를 낸다고 증명할 수는 없습니다. profile limit, leaderboard limit, match mapper 같은 개별 parity는 별도 test가 필요합니다.

## 불변 조건과 증거 범위

| 불변 조건 | 도입/문제 | 수정·검증 | 남는 한계 |
| --- | --- | --- | --- |
| DB row와 외부 DTO를 분리한다 | `4aa060c0b8df` | mapper와 이후 row-mapper 시험 | runtime row validation은 없음 |
| repository가 backend lifetime을 소유한다 | `9277572765e7` | `c43b87694b29`의 tracked cleanup | 모든 caller가 `close()`를 호출하는지는 별도 문제 |
| 최고 연승은 실제 경기 연속성에서 계산한다 | `c7ea1ff241c8`의 임의 값 | `035b97ca7c58`, `6b661420e060` | 최근 8경기 범위만 계산 |
| 실제 PostgreSQL schema가 test 간 격리된다 | memory 시험으로는 증명 불가 | `c43b87694b29` | PostgreSQL이 필요한 별도 suite이며 여기서는 실행하지 않음 |
| 동일 interface와 동일 의미를 구분한다 | profile/leaderboard limit 차이 | 문서에서 차이를 명시 | 전 메서드 parity matrix는 이 Thread 범위 밖 |

## 이 Thread의 경계

이 Thread는 repository 도입, 기본 read model, 최고 연승 수정, backend 시험 경계만 다룹니다. 다음 항목은 같은 파일이나 인접 commit에 있어도 여기서 확장하지 않습니다.

- migration·seed 분리, readiness, startup seed 금지, test reset: Thread 2
- canonical row type과 mapper refactor: Thread 3
- friendship canonical identity와 역방향 경쟁: Thread 4
- tournament capacity·seed 경쟁: Thread 5
- session/token 보안, WebSocket ticket 소비, match finalization, chat room 무결성, admin audit atomicity: 다른 Development Thread

> 검증 기록: 위 설명은 `web/ft_transcendence` branch의 표시된 exact SHA diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 repository를 checkout해 unit/integration suite를 실행하지 않았으므로, 시험 결과가 아니라 시험 코드가 고정한 assertion 범위만 서술합니다.
