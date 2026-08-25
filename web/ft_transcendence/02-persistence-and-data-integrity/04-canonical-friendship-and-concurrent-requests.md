# Thread: Canonical friendship — 방향이 다른 두 요청을 하나의 관계로 수렴시키기

## 개요

친구 관계는 요청 방향을 갖지만, 저장 identity는 사용자 두 명의 **순서 없는 쌍**이어야 합니다. 초기 구현은 `(requester_id, addressee_id)`에 unique constraint를 두었기 때문에 `A → B`와 `B → A`가 서로 다른 행으로 공존할 수 있었습니다. memory 구현은 더 약해서 requester/addressee 자체를 저장하지 않고 `FriendSummary`만 배열에 넣었으므로, 사용자별 목록과 수락 권한도 정확히 표현할 수 없었습니다.

이 Thread는 기존 데이터를 canonical pair로 정리하고, PostgreSQL의 expression unique index와 `INSERT ... ON CONFLICT` 한 문장으로 다음 규칙을 고정합니다.

- 자기 자신에게 친구 요청을 보낼 수 없습니다.
- 같은 방향의 반복 요청은 같은 pending/accepted 관계를 그대로 반환합니다.
- 반대 방향의 pending 요청은 별도 행을 만들지 않고 기존 관계를 accepted로 전환합니다.
- 어느 사용자 관점에서 조회해도 같은 friendship ID와 상대 사용자가 보입니다.
- memory 구현도 requester/addressee ID를 보존해 같은 상태 전이를 흉내 냅니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `645e5a3c8e96` | `feat(db): 친구 관계 저장 구현` | B | `PERSISTENCE` | 방향성 요청·목록·수락 API의 첫 구현을 추가합니다. |
| 2 | `ffb0a8275a4f` | `feat(db): friendship canonical pair 제약 추가` | A | `PERSISTENCE, RISK` | 기존 self/reverse duplicate를 정리하고 순서 없는 pair unique를 DB에 설치합니다. |
| 3 | `77c555aba9a0` | `feat(db): PostgreSQL friendship 요청을 원자화` | A | `PERSISTENCE, RISK` | insert와 역방향 자동 수락을 하나의 conflict statement로 만듭니다. |
| 4 | `34db79005f30` | `feat(db): memory friendship invariant 적용` | B | `PERSISTENCE` | memory가 관계의 양 끝과 수락 권한을 보존하도록 representation을 바꿉니다. |
| 5 | `cdaca35ccf7f` | `test(db): friendship와 tournament 경쟁 상태 검증` | A | `PERSISTENCE, TOURNAMENT, RISK` | friendship의 자기 요청·반복·역방향·단일 row 불변 조건을 두 backend에서 검증합니다. |

## 1. `645e5a3c8e96` — 방향성 API가 만든 첫 상태

PostgreSQL 구현은 친구 목록을 두 방향 모두에서 조회하고, 요청과 수락 메서드를 repository에 추가합니다.

```ts
async listFriends(userId: string): Promise<FriendSummary[]> {
  const result = await sql<FriendshipWithUserRow>`
    select f.id as friendship_id, f.status as friendship_status, u.*
    from friendships f
    join users u on u.id = case
      when f.requester_id = ${userId} then f.addressee_id
      else f.requester_id
    end
    where f.requester_id = ${userId} or f.addressee_id = ${userId}
    order by f.updated_at desc
  `.execute(this.db);
  return result.rows.map(toFriendSummary);
}
```

초기 요청은 방향성 unique key에만 충돌합니다.

```ts
insert into friendships (requester_id, addressee_id, status)
values (${requesterId}, ${addressee.id}, 'pending')
on conflict (requester_id, addressee_id)
do update set updated_at = now()
returning id, status
```

이 방식의 문제는 다음과 같습니다.

| 입력 | 저장 결과 |
| --- | --- |
| `A → B` 첫 요청 | pending 한 행 |
| `A → B` 반복 | 같은 방향 unique conflict로 기존 행 재사용 |
| `B → A` 요청 | 다른 key이므로 두 번째 pending 행 생성 가능 |
| `A → A` 요청 | application/DB 어느 쪽에서도 금지하지 않음 |

초기 `acceptFriend`도 `friendshipId`와 `addressee_id`로 update한 뒤 전체 목록을 다시 읽어 결과를 찾습니다. update가 0행이어도 별도 결과를 확인하지 않고 후속 목록 조회에 의존합니다.

Memory 구현은 더 큰 표현 손실을 갖습니다.

```ts
private readonly friendships: FriendSummary[] = [];

async listFriends(): Promise<FriendSummary[]> {
  return this.friendships;
}
```

`requesterId`와 `addresseeId`를 저장하지 않으므로 모든 사용자가 같은 배열을 보고, 누가 수락할 수 있는지 확인할 수도 없습니다. 이 상태에서는 PostgreSQL과 메서드 이름만 같을 뿐 관계 의미는 같지 않습니다.

## 2. `ffb0a8275a4f` — 기존 데이터부터 canonical pair로 변환

새 unique index를 바로 만들면 이미 존재하는 self row와 reverse duplicate 때문에 migration이 실패할 수 있습니다. 따라서 migration은 정리 → constraint 교체 순서로 진행합니다.

### 2.1 자기 관계 제거

```sql
delete from friendships
where requester_id = addressee_id;
```

기존 잘못된 데이터는 유지하거나 별도 상태로 바꾸지 않고 삭제합니다.

### 2.2 역방향 pending 쌍을 accepted로 승격

```sql
update friendships as friendship
set
  status = 'accepted',
  updated_at = greatest(friendship.updated_at, reverse_friendship.updated_at)
from friendships as reverse_friendship
where friendship.requester_id = reverse_friendship.addressee_id
  and friendship.addressee_id = reverse_friendship.requester_id
  and friendship.id <> reverse_friendship.id
  and friendship.status = 'pending';
```

서로 상대에게 pending 요청을 보냈다면 관계 의도는 상호 확인된 것으로 해석합니다. 양쪽 행 중 pending인 행이 reverse row를 만나 accepted가 되고 최신 `updated_at`을 유지합니다.

### 2.3 사용자 쌍마다 한 행만 남김

```sql
with ranked_friendships as (
  select
    id,
    row_number() over (
      partition by
        least(requester_id, addressee_id),
        greatest(requester_id, addressee_id)
      order by
        case when status = 'accepted' then 0 else 1 end,
        created_at asc,
        id asc
    ) as position
  from friendships
)
delete from friendships
where id in (
  select id
  from ranked_friendships
  where position > 1
);
```

선택 기준은 결정적입니다.

1. accepted를 pending보다 우선
2. 상태가 같으면 먼저 생성된 row 우선
3. 생성 시각도 같으면 ID 오름차순 우선

따라서 migration을 적용할 때 어떤 ID가 살아남는지가 query 순서에 우연히 좌우되지 않습니다.

### 2.4 DB 불변 조건 설치

```sql
alter table friendships
  drop constraint if exists friendships_requester_id_addressee_id_key;

alter table friendships
  add constraint friendships_distinct_users_check
  check (requester_id <> addressee_id);

create unique index friendships_canonical_pair_unique
  on friendships (
    least(requester_id, addressee_id),
    greatest(requester_id, addressee_id)
  );
```

이제 DB identity는 `(A, B)`와 `(B, A)`에 대해 같은 expression key입니다. application code가 check를 빼먹더라도 self row와 두 번째 pair row는 DB가 거부합니다.

## 3. `77c555aba9a0` — 요청 처리 전체를 한 statement로 수렴

### 문제: check 후 insert로는 경쟁을 닫을 수 없음

Application에서 먼저 기존 관계를 조회하고 없으면 insert하는 방식은 두 요청이 동시에 같은 “없음” 상태를 읽을 수 있습니다. canonical unique index는 중복 저장을 막지만, loser 요청이 어떤 상태를 반환해야 하는지와 reverse pending을 accepted로 바꾸는 규칙까지는 정하지 않습니다.

### 결정: canonical unique index를 conflict target으로 사용

```ts
async requestFriend(
  requesterId: string,
  addresseeHandle: string
): Promise<FriendSummary> {
  const addressee = await this.getUserByHandle(addresseeHandle);
  if (!addressee) throw new Error("friend not found");
  if (requesterId === addressee.id) {
    throw new Error("cannot friend yourself");
  }

  const result = await sql<{
    id: string;
    status: FriendSummary["status"];
  }>`
    insert into friendships (requester_id, addressee_id, status)
    values (${requesterId}, ${addressee.id}, 'pending')
    on conflict (
      (least(requester_id, addressee_id)),
      (greatest(requester_id, addressee_id))
    ) do update set
      status = case
        when friendships.status = 'pending'
          and friendships.requester_id = excluded.addressee_id
          and friendships.addressee_id = excluded.requester_id
        then 'accepted'
        else friendships.status
      end,
      updated_at = case
        when friendships.status = 'pending'
          and friendships.requester_id = excluded.addressee_id
          and friendships.addressee_id = excluded.requester_id
        then now()
        else friendships.updated_at
      end
    returning id, status
  `.execute(this.db);

  const friendship = firstRow(result);
  return {
    id: friendship.id,
    status: friendship.status,
    user: addressee
  };
}
```

### 상태 전이

| 기존 row | 새 요청 | 결과 |
| --- | --- | --- |
| 없음 | `A → B` | 새 pending row |
| `A → B`, pending | `A → B` 반복 | 기존 ID·pending 유지, timestamp도 유지 |
| `A → B`, pending | `B → A` | 기존 ID를 accepted로 갱신 |
| canonical pair, accepted | 어느 방향 반복 | 기존 ID·accepted 유지 |

`excluded`는 새로 넣으려던 방향을, `friendships`는 이미 저장된 방향을 나타냅니다. 두 방향이 정확히 반대이고 기존 상태가 pending일 때만 accepted가 됩니다. 이 판정과 update는 unique conflict 처리 안에서 실행되므로 별도 read/write window가 없습니다.

### 수락 권한도 update 결과에서 확정

```ts
const result = await sql<{
  id: string;
  status: FriendSummary["status"];
  requester_id: string;
}>`
  update friendships
  set status = 'accepted', updated_at = now()
  where id = ${friendshipId} and addressee_id = ${userId}
  returning id, status, requester_id
`.execute(this.db);
```

수락자는 저장된 addressee여야 합니다. update된 row가 없으면 `firstRow` 단계에서 실패하고, 성공한 경우 반환된 `requester_id`로 상대 사용자를 읽습니다. 목록 전체를 다시 읽어 결과를 추측하던 경로가 사라집니다.

## 4. `34db79005f30` — memory도 관계의 양 끝을 보존

Memory storage는 API DTO 배열에서 canonical relation record로 바뀝니다.

```ts
type MemoryFriendship = {
  id: string;
  requesterId: string;
  addresseeId: string;
  status: FriendSummary["status"];
};

private readonly friendships: MemoryFriendship[] = [];
```

목록은 요청한 사용자가 관계의 어느 쪽인지 확인하고 반대쪽 ID로 DTO를 조립합니다.

```ts
async listFriends(userId: string): Promise<FriendSummary[]> {
  return this.friendships
    .filter((friendship) =>
      friendship.requesterId === userId
      || friendship.addresseeId === userId
    )
    .map((friendship) => {
      const otherUserId = friendship.requesterId === userId
        ? friendship.addresseeId
        : friendship.requesterId;
      const otherUser = this.users.get(otherUserId);
      if (!otherUser) throw new Error("friend not found");
      return {
        id: friendship.id,
        status: friendship.status,
        user: toPublicUser(otherUser, true)
      };
    });
}
```

요청은 두 방향을 모두 검색합니다.

```ts
const existing = this.friendships.find((friendship) =>
  (friendship.requesterId === requesterId
    && friendship.addresseeId === user.id)
  || (friendship.requesterId === user.id
    && friendship.addresseeId === requesterId)
);

if (existing) {
  const isReversePending = existing.status === "pending"
    && existing.requesterId === user.id
    && existing.addresseeId === requesterId;
  if (isReversePending) existing.status = "accepted";
  return { id: existing.id, status: existing.status, user };
}
```

Memory는 single-process 배열을 순차적으로 수정하므로 PostgreSQL의 lock/constraint와 같은 동시성 primitive를 제공하지 않습니다. 그러나 같은 repository API를 순차 호출했을 때의 identity와 상태 전이는 맞춥니다. 수락도 `friend.addresseeId === userId`를 확인해 요청자가 자기 요청을 수락하지 못하게 합니다.

## 5. `cdaca35ccf7f` — 무엇을 실제로 검증하는가

이 commit은 friendship과 tournament 시험을 한 diff에 추가합니다. 이 문서에는 friendship 관련 assertion만 포함합니다.

### Memory case

1. 자기 요청이 `cannot friend yourself`로 실패
2. `A → B` 첫 요청은 pending
3. 같은 방향 반복은 첫 결과와 동일
4. `B → A`는 같은 ID를 accepted로 전환
5. A의 목록에는 B가, B의 목록에는 A가 같은 friendship ID로 보임

### PostgreSQL case

동일한 API assertion에 더해 실제 table을 읽습니다.

```ts
const stored = await pool.query<{
  requester_id: string;
  addressee_id: string;
  status: string;
}>("select requester_id, addressee_id, status from friendships");

expect(stored.rows).toEqual([{
  requester_id: firstUser.id,
  addressee_id: secondUser.id,
  status: "accepted"
}]);
```

또한 raw SQL로 self row를 넣어 `friendships_distinct_users_check`가 실제 DB에서 작동하는지 확인합니다.

```ts
await expect(pool.query(
  "insert into friendships (requester_id, addressee_id, status) values ($1, $1, 'pending')",
  [firstUser.id]
)).rejects.toMatchObject({
  constraint: "friendships_distinct_users_check"
});
```

### 시험의 한계

Friendship case의 요청들은 순차적으로 실행됩니다. 두 방향을 실제 `Promise.all`로 동시에 보내 unique-index conflict 경쟁을 강제로 재현하지는 않습니다. 따라서 다음을 구분해야 합니다.

- **시험이 직접 증명**: 반복/역방향 API 의미, 한 행 저장, self constraint
- **코드와 DB 제약이 제공하는 동시성 방어**: canonical unique index와 atomic `ON CONFLICT`
- **직접 시험하지 않음**: 높은 동시 요청에서 lock 대기·deadlock·격리 수준별 동작

## 최종 불변 조건

```text
friendship identity = unordered(requesterId, addresseeId)

A == B                         -> application과 DB가 거부
pair 없음, A -> B              -> pending 한 행
같은 방향 반복                -> 같은 ID와 상태
반대 방향 요청, 기존 pending   -> 같은 ID를 accepted로 전환
accepted 뒤 어느 방향 반복     -> accepted 유지
명시적 accept                  -> 저장된 addressee만 수행 가능
```

| 계층 | 책임 |
| --- | --- |
| migration | 기존 self/reverse duplicate 정리, canonical constraint 설치 |
| PostgreSQL request method | 새 요청·반복·역방향 수락을 한 statement로 전이 |
| PostgreSQL accept method | addressee 권한과 반환 row를 같은 update에서 확정 |
| memory implementation | requester/addressee identity와 동일 상태 전이 재현 |
| tests | 두 backend의 observable 결과와 DB row/constraint 확인 |

## 보장하지 않는 것

- 친구 삭제, 차단, 요청 취소, 재요청 만료 같은 추가 상태 전이
- friendship 상태를 enum/check constraint로 제한하는 일반 정책
- user 조회와 request statement 전체를 한 transaction snapshot으로 묶는 것
- memory implementation의 multi-process 또는 실제 병렬 안전성
- 동시 reverse request를 강제로 충돌시키는 dedicated stress test

## 이 Thread의 경계

같은 migration `004_friendship_tournament_invariants.sql`에 tournament seed 제약도 들어가지만 Thread 5에서만 설명합니다. 공유 test commit의 tournament final-slot 경쟁 assertion도 Thread 5로 분리합니다.

> 검증 기록: 위 설명은 `web/ft_transcendence` branch의 표시된 exact SHA diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 memory/PostgreSQL test를 실행하지 않았으며, 시험 코드와 DB constraint가 표현하는 범위를 구분해 서술했습니다.
