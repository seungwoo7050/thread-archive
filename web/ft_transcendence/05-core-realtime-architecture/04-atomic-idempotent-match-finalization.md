# 경기 결과를 한 번만, 전부 함께 확정하기

경기가 끝났다는 event는 여러 번 발생할 수 있습니다. 같은 room의 tick, disconnect timeout, 재시도 timer가 겹칠 수 있고, API process가 DB 응답을 받기 전에 연결이 끊길 수도 있습니다. 단순히 `matches` row의 중복 insert만 막아서는 충분하지 않습니다. 승패, rating, rating history, tournament bracket도 **같은 논리적 결과에 대해 정확히 한 번, 한 transaction에서** 바뀌어야 합니다.

이 Thread는 다음 세 층을 연결합니다.

- DB의 논리적 식별자 `result_key`
- 참가자·tournament row를 잠그는 atomic finalization transaction
- GameHub의 in-flight Promise 중복 억제와 실패 후 backoff retry

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `38504f041a6a` | `feat(db): 경기 결과 저장 구현` | B | `REALTIME, PERSISTENCE` | match insert 뒤 승자·패자를 별도 update하는 초기 비원자 경로 |
| `75bbc762e06d` | `feat(db): match result key와 rating history schema 추가` | A | `PERSISTENCE, RISK` | 논리적 결과 identity와 사용자별 rating history uniqueness 도입 |
| `83f9aee2522a` | `feat(db): PostgreSQL 경기 결과 중복 생성을 차단` | A | `PERSISTENCE, RISK` | `result_key` conflict를 같은 기존 결과로 수렴 |
| `e9d577ebc1ab` | `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영` | A | `PERSISTENCE` | 정렬된 row lock 뒤 match·stats·rating·history를 한 transaction에 포함 |
| `e338ea32b2a6` | `feat(db): PostgreSQL tournament 경기 확정을 연결` | S | `PERSISTENCE, TOURNAMENT, RISK` | bracket 진행과 final 생성·대회 종료를 같은 transaction으로 확장 |
| `582a1615a2c6` | `test(db): 경기 결과 단일 확정 조건 검증` | A | `PERSISTENCE, TOURNAMENT, RISK` | 20-way concurrency와 rollback, concurrent semifinal을 검증 |
| `10bf15723591` | `refactor(game): 경기 결과 확정 boundary 사용` | A | `REALTIME, PERSISTENCE, TOURNAMENT` | GameHub가 한 repository finalization API와 in-flight Promise를 사용 |
| `e939a50948b2` | `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지` | A | `REALTIME, RISK` | 실패한 room을 삭제하지 않고 bounded backoff로 같은 result를 재시도 |

## 1. 최초 구현은 부분 성공을 허용했다

### `38504f041a6a`

초기 repository는 match row를 insert한 뒤 승자의 wins/rating, 패자의 losses/rating을 별도 statement로 갱신했습니다.

```text
INSERT match
→ UPDATE winner
→ UPDATE loser
```

중간 statement가 실패하면 match는 존재하지만 rating 일부가 반영되지 않을 수 있습니다. 같은 요청을 재시도하면 새 match가 추가될 수 있고 rating이 다시 변할 수 있습니다. tournament 연결도 별도 호출이라면 더 큰 partial state가 생깁니다.

이 커밋의 역할은 persistence의 첫 경로를 제공한 것이며, 이 Thread의 최종 보장을 제공하지는 않습니다.

## 2. 중복 판정 기준을 transport 호출이 아니라 논리적 결과로 바꿨다

### `75bbc762e06d` — schema identity

`matches.result_key`가 unique, non-null column으로 추가됩니다. 기존 row는 `legacy:<match-id>` 형식으로 backfill되어 migration 뒤에도 제약을 만족합니다.

rating history는 `(match_id, user_id)` unique constraint를 갖습니다. 한 match에서 같은 사용자 history가 두 번 생기는 것을 schema 수준에서 막습니다.

`result_key`는 request ID나 DB-generated match ID가 아닙니다. caller가 “이 방의 이 종료 결과”를 안정적으로 식별하는 key를 제공합니다. GameHub는 나중에 다음 값을 사용합니다.

```text
room:<roomId>:finished
```

동일한 room 종료를 여러 번 시도해도 같은 key로 수렴합니다.

### `83f9aee2522a` — insert-or-existing

PostgreSQL finalization은 transaction 안에서 먼저 match insert를 시도합니다.

```sql
INSERT INTO matches (..., result_key)
VALUES (..., $resultKey)
ON CONFLICT (result_key) DO NOTHING
RETURNING id;
```

row가 반환되지 않으면 같은 `result_key`의 기존 match를 조회하고 `{ created: false }`를 반환합니다. 이 시점에서 **match row 중복**은 막혔지만, rating·history까지 한 transaction에 들어가는 것은 다음 커밋입니다.

## 3. 참가자 상태를 정렬해 잠그고 모든 사용자 변경을 같은 transaction에 넣었다

### `e9d577ebc1ab`

동시 경기가 같은 사용자의 rating을 갱신할 수 있으므로, winner/loser row를 일정한 순서로 잠그지 않으면 lost update 또는 교차 deadlock 위험이 있습니다. 구현은 participant ID를 중복 제거하고 정렬한 뒤 `SELECT … FOR UPDATE`로 가져옵니다.

```text
participantIds = unique([winnerId, loserId]).sort()
SELECT users WHERE id IN (...) ORDER BY id FOR UPDATE
```

그 뒤 같은 transaction에서 다음을 수행합니다.

1. `result_key`로 match를 insert하거나 기존 결과를 확인합니다.
2. 새 결과일 때만 winner/loser의 현재 rating을 읽습니다.
3. winner는 wins와 rating을 증가시킵니다.
4. loser는 losses와 rating을 감소시키되 최저 rating을 지킵니다.
5. 두 사용자의 rating history를 같은 match ID로 기록합니다.
6. 모든 작업이 성공해야 commit합니다.

실제 정책은 winner `+16`, loser `-12`, loser rating 하한 `800`입니다. retry가 기존 match를 찾은 경우 rating/stat/history branch를 다시 실행하지 않습니다.

따라서 최종 원자 단위는 다음과 같습니다.

```text
match row
+ winner wins/rating
+ loser losses/rating
+ both rating history rows
```

어느 하나라도 실패하면 transaction 전체가 rollback됩니다.

## 4. tournament bracket도 별도 후처리가 아니라 같은 결과 transaction에 들어갔다

### `e338ea32b2a6` — 가장 중요한 확장

일반 match를 저장한 뒤 tournament match를 따로 완료하면 다음과 같은 상태가 가능합니다.

```text
match/rating commit 성공
→ tournament link 실패
→ 재시도 시 일반 결과는 existing, bracket은 미완료
```

이 커밋은 tournament match와 tournament row를 lock하고, 참가자가 해당 bracket match와 일치하는지 확인한 뒤 같은 transaction에서 진행 상태를 바꿉니다.

준결승 결과라면 두 승자를 연결하는 final row를 생성합니다. 두 준결승이 동시에 완료될 수 있으므로 final 생성도 unique constraint와 conflict 처리를 사용해 정확히 하나로 수렴합니다. final 결과라면 tournament 자체를 finished로 전환합니다.

```text
lock participants
→ insert-or-get match
→ update stats/rating/history
→ lock tournament match + tournament
→ validate participants
→ complete bracket match
→ semifinal이면 final 한 개 생성
→ final이면 tournament finish
→ COMMIT
```

이제 tournament link 오류는 일반 match·rating·history까지 함께 rollback시킵니다. “경기 결과는 저장됐는데 bracket은 진행되지 않은” half-commit을 만들지 않습니다.

## 5. concurrency와 rollback을 결과 상태로 검증했다

### `582a1615a2c6`

memory repository와 PostgreSQL 경로의 테스트가 다음을 고정합니다.

- 같은 `result_key`를 20개 concurrent call로 확정해도 match는 한 개입니다.
- winner/loser rating과 stats는 한 번만 변합니다.
- rating history도 사용자별 한 row입니다.
- 잘못된 tournament link로 transaction이 실패하면 match, history, rating 변화가 모두 남지 않습니다.
- 두 semifinal을 동시에 완료해도 final match는 정확히 하나 생성됩니다.

여기서 중요한 assertion은 “20개 호출 중 하나만 `created: true`”에 그치지 않습니다. 최종 DB row 수와 사용자 rating, history, bracket 상태를 함께 확인해야 atomic idempotency가 증명됩니다.

## 6. GameHub도 같은 논리적 결과의 동시 확정을 하나로 합쳤다

### `10bf15723591`

GameHub는 room별 `finishing` Promise를 보유합니다. 같은 room에서 finalization이 다시 요청되면 새 repository call을 만들지 않고 기존 Promise를 돌려줍니다.

```ts
if (room.finishing) return room.finishing;

room.finishing = this.finalizeRoom(room, winnerSide)
  .catch((error) => {
    room.finishing = null;
    throw error;
  });

return room.finishing;
```

repository 호출도 `createMatch`와 별도 tournament completion이 아니라 하나의 `finalizeMatch` boundary로 바뀝니다.

```ts
await repo.finalizeMatch({
  resultKey: `room:${room.id}:finished`,
  mode: room.mode,
  winnerId,
  loserId,
  tournamentMatchId: room.tournamentMatchId
});
```

in-process Promise는 같은 process 안의 동시 호출을 줄이는 최적화이자 ownership 표식입니다. 정확히 한 번 반영되는 최종 권위는 DB의 `result_key`와 transaction에 있습니다. process가 재시작되어 Promise가 사라져도 같은 key로 재시도할 수 있습니다.

## 7. persistence 실패를 room 삭제가 아니라 재시도 가능한 상태로 남겼다

### `e939a50948b2`

DB가 잠시 내려간 상태에서 room을 바로 삭제하면 authoritative result를 다시 확정할 자료가 사라집니다. 이 fix는 finalization 실패 시 room과 결과 key를 유지하고 timer로 재시도합니다.

backoff는 다음과 같습니다.

```text
250 ms → 500 ms → 1 s → 2 s → 4 s → 최대 5 s
```

retry timer는 `unref`되어 이것만으로 process 종료를 붙잡지 않습니다. 다음 시도는 같은 room, 같은 winner, 같은 `result_key`를 사용합니다. 성공하면 room을 정리하고, 실패하면 observer에 failure를 알린 뒤 다음 backoff를 예약합니다.

room이 abandon, close, 정상 제거되는 경로에서는 retry timer를 clear합니다. 그렇지 않으면 이미 소유권을 잃은 room을 나중에 다시 확정할 수 있습니다.

이 커밋에는 최대 재시도 횟수가 없습니다. room이 process 안에 남아 있는 동안 계속 시도하는 정책입니다. process 자체가 종료되면 메모리 room과 timer는 사라지므로, 별도 durable job queue까지 보장하는 설계는 아닙니다.

## 최종 terminal states

| 경로 | DB 결과 | GameHub 상태 |
| --- | --- | --- |
| 최초 성공 | match·rating·history·tournament 함께 commit | room 종료·정리 |
| 동일 결과 동시 호출 | 한 transaction만 `created: true`, 나머지는 existing 결과 | 하나의 in-flight Promise 또는 같은 persisted result 사용 |
| transaction 중간 실패 | 관련 변경 전체 rollback | room과 result key 유지, retry 예약 |
| retry 성공 | 기존 rollback 상태에서 한 번 commit | timer 해제, room 정리 |
| process close/room abandon | 새 DB commit 없음 | retry timer 해제, local ownership 종료 |

최종 불변 조건은 다음과 같습니다.

- `result_key`가 논리적 경기 결과를 식별합니다.
- match, stats, rating, rating history, tournament progress는 하나의 transaction입니다.
- participant와 tournament row는 결정적 순서로 잠깁니다.
- 같은 결과의 동시·반복 요청은 기존 match로 수렴하고 rating을 다시 적용하지 않습니다.
- GameHub는 같은 room의 in-flight call을 하나로 합치고, transient DB failure 뒤 같은 결과를 재시도합니다.

이 Thread는 server simulation이 winner를 어떻게 결정하는지 다루지 않습니다. 그 값의 권위는 Thread 01입니다. disconnect timeout이 언제 몰수패를 발생시키는지는 Thread 05, guest 결과를 아예 DB에 넣지 않는 분기는 Thread 07에서 다룹니다.
