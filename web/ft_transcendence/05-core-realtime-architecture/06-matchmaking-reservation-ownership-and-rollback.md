# Matchmaking reservation을 누가 언제 해제하는가

매칭은 “가장 가까운 상대를 찾아 room을 만든다”로 끝나지 않습니다. 상대를 queue에서 꺼낸 순간부터 room 생성이 끝날 때까지 중간 상태가 생깁니다. 그 사이 socket이 닫히거나, NPC 조회가 늦어지거나, observer·send·room publish가 실패하면 사용자가 `matched`로 남아 다시 queue에 들어가지 못할 수 있습니다.

이 Thread는 matchmaking을 다음 ownership protocol로 바꿉니다.

```text
queued
  → matched reservation
  → room 생성 성공: room이 reservation을 소유
  → room 종료/abandon/failure: release
```

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `1122e6a4b901` | `feat(game): 대기 플레이어 NPC fallback 구성` | B | `REALTIME` | GameHub queue entry별 6초 timer와 ad-hoc NPC fallback 도입 |
| `1ec8335b0e75` | `refactor(game): matchmaking player와 fallback 계약 정의` | A | `REALTIME, REFACTOR` | player kind·join result·AI fallback result·reservation 상태 정의 |
| `a4f59a2e8192` | `refactor(game): rating 기반 closest-pair queue 구현` | A | `REALTIME, REFACTOR` | 동일 kind·rating 차이 상한 안에서 closest pair 선택 |
| `7871e29278c2` | `refactor(game): AI fallback과 reservation lifecycle 구현` | A | `REALTIME, REFACTOR` | `claimAiFallback`, `leaveQueue`, `release`로 상태 종료 API 제공 |
| `e53559ef3a11` | `refactor(game): Matchmaker queue reservation을 GameHub에 연결` | A | `REALTIME, REFACTOR` | Matchmaker 상태와 실제 socket entry를 연결하고 room 실패 rollback |
| `51f36aa50596` | `refactor(game): Matchmaker AI fallback를 GameHub에 연결` | A | `REALTIME, OPERATIONS, RISK` | async NPC lookup 전후 ownership 재검사와 release |
| `a23fc26a7f82` | `refactor(game): queue와 reservation cleanup 일원화` | S | `REALTIME, ARCH, RISK` | queue·room·drain·finalization 모든 종료 경로를 하나의 release 규칙으로 수렴 |
| `b5bfeee0e23e` | `refactor(game): room 생성과 finalization cleanup 보장` | A | `REALTIME, OBSERVABILITY, RISK` | room publish와 post-finalization observer/send 실패에도 rollback/finally 보장 |
| `112228db8878` | `test(game): matchmaking lifecycle 검증` | A | `REALTIME, RISK, TEST` | rating gap, forfeit/abandon release, room creation failure 재시도 검증 |

## 1. 최초 fallback은 timer와 queue entry가 직접 서로를 가리켰다

### `1122e6a4b901`

상대를 찾지 못한 player는 GameHub의 queue array에 들어가고 6초 timer를 받습니다. timer가 실행되면 entry가 아직 queue에 있고 socket이 open이며 room이 없는지 확인한 뒤 가장 가까운 NPC를 조회해 AI room을 만듭니다.

human match, queue leave, stale socket prune 시에는 `clearQueueTimer`로 timer를 취소합니다. room에는 `npcUser`를 따로 보관해 경기 종료 때 snapshot의 표시용 player가 아니라 실제 NPC identity를 사용합니다.

이 구현은 기능은 제공하지만 ownership이 여러 곳에 섞여 있습니다.

- queue array가 대기 여부를 나타냅니다.
- timer 존재 여부가 fallback 예정 상태를 나타냅니다.
- `client.roomId`가 매칭 완료를 암시합니다.
- matched 되었지만 room 생성 중인 상태를 명시하지 않습니다.

그래서 queue에서 entry를 제거한 뒤 실패하면 어느 구조가 사용자를 다시 풀어야 하는지 알기 어렵습니다.

## 2. Matchmaker가 상태와 반환값을 명시했다

### `1ec8335b0e75` — 계약부터 분리

`MatchmakingPlayer`는 repository user 전체가 아니라 매칭에 필요한 값만 가집니다.

```ts
interface MatchmakingPlayer {
  userId: string;
  rating: number;
  kind: "registered" | "guest";
}
```

join 결과도 union으로 정의됩니다.

```ts
type MatchmakerJoinResult =
  | { type: "queued"; queuedAtMs: number; aiFallbackAtMs: number }
  | { type: "matched"; match: MatchmakingPair }
  | { type: "duplicate"; status: "queued" | "matched" };
```

AI fallback은 `waiting`, `ready`, `unavailable`로 분리됩니다. 이 커밋은 아직 queue algorithm을 구현하지 않고, caller와 Matchmaker 사이의 상태 언어를 먼저 고정합니다. fallback 기준은 6초입니다.

### `a4f59a2e8192` — closest pair와 reservation

`Matchmaker`는 queue array와 `playerStatuses` map을 소유합니다. enqueue 시 이미 status가 있으면 새 entry를 만들지 않고 `duplicate`를 반환합니다.

상대 선택 조건은 다음과 같습니다.

1. `registered`는 registered끼리, `guest`는 guest끼리만 비교합니다.
2. rating 차이가 configured maximum 이하여야 합니다.
3. 가능한 후보 중 절대 rating 차이가 가장 작은 entry를 선택합니다.
4. 같은 차이면 먼저 queue를 순회하며 발견한 entry가 유지됩니다.

GameHub integration에서 최대 차이는 `200`입니다.

match가 생기면 기존 상대를 queue에서 제거하고 두 사용자의 status를 모두 `matched`로 바꿉니다.

```ts
this.playerStatuses.set(opponent.userId, "matched");
this.playerStatuses.set(entrant.userId, "matched");
```

여기서 `matched`는 room 생성 성공을 뜻하지 않습니다. **두 사용자가 다른 매칭에 사용되지 않도록 잡아 둔 reservation**입니다. 반드시 이후 성공 경로가 소유권을 이어받거나 실패 경로가 release해야 합니다.

## 3. queue 이탈과 reservation 해제를 다른 연산으로 만들었다

### `7871e29278c2`

Matchmaker에 세 lifecycle API가 생깁니다.

| API | 허용 상태 | 효과 |
| --- | --- | --- |
| `claimAiFallback(userId)` | `queued` | 6초 전이면 `waiting`, 이후면 queue에서 제거하고 `matched` reservation 반환 |
| `leaveQueue(userId)` | `queued` | queue entry와 status 제거 |
| `release(userId)` | `queued` 또는 `matched` | 남은 queue entry가 있으면 제거하고 status를 무조건 삭제 |

`leaveQueue`와 `release`를 구분한 이유는 호출자의 의도를 드러내기 위해서입니다. 사용자가 단순 대기를 취소할 때는 queued 상태만 끝내야 하지만, room 생성 실패나 경기 종료에서는 이미 `matched`인 reservation도 풀어야 합니다.

`claimAiFallback`이 `ready`를 반환하는 순간 player는 queue에서 빠지고 `matched`가 됩니다. 그 뒤 NPC 조회나 room 생성이 실패하면 caller가 반드시 `release`해야 합니다.

## 4. Matchmaker 상태와 실제 connection entry를 연결하면서 rollback을 추가했다

### `e53559ef3a11`

GameHub는 다음 두 구조를 함께 사용합니다.

- `Matchmaker`: user ID 기준 논리적 queue/reservation 상태
- `queueEntries`: user ID → 실제 socket client, queued time, fallback timer

queue 참가 결과가 `matched`이면 Matchmaker가 돌려준 left user ID로 실제 opponent connection을 찾습니다. entry가 없다면 이미 논리 상태와 transport 상태가 어긋난 것이므로 두 reservation을 즉시 해제하고 오류를 냅니다.

```ts
const opponent = queueEntries.get(join.match.left.userId);
if (!opponent) {
  matchmaker.release(join.match.left.userId);
  matchmaker.release(join.match.right.userId);
  throw new Error("대기 중인 상대 연결을 찾지 못했습니다.");
}
```

room 생성도 `try/catch`로 감싸 두 사용자 reservation을 함께 rollback합니다. 이 시점에는 legacy queue array가 아직 남아 있어 두 표현이 공존하지만, 새 매칭 결정은 Matchmaker를 사용합니다.

## 5. async NPC lookup 뒤에는 상태를 다시 확인해야 한다

### `51f36aa50596`

fallback timer가 발동해 `claimAiFallback`이 `ready`를 반환한 뒤 NPC repository 조회가 await됩니다. 그 사이 다음이 바뀔 수 있습니다.

- client socket이 닫힙니다.
- client가 다른 room을 얻습니다.
- server가 drain에 들어갑니다.
- queue entry가 교체되거나 제거됩니다.

따라서 await 전 검사만으로는 충분하지 않습니다. 코드는 NPC 조회 뒤 다음 조건을 다시 확인합니다.

```ts
queueEntries.get(userId) === originalEntry
&& acceptingMatches
&& socket.readyState === OPEN
&& !client.roomId
```

하나라도 거짓이면 `matchmaker.release(userId)`로 reservation을 되돌립니다. 등록 사용자에게 NPC가 없거나 room 생성이 throw해도 같은 release 경로를 탑니다. guest는 persistent NPC를 조회하지 않고 내부 AI room을 사용합니다.

fallback timer도 Matchmaker가 반환한 `aiFallbackAtMs`를 기준으로 arm합니다. timer가 조금 일찍 실행되면 `claimAiFallback`의 `waiting.remainingMs`로 다시 예약하므로 wall-clock 기준 6초보다 앞서 reservation을 가져가지 않습니다.

## 6. 모든 terminal path를 하나의 cleanup 규칙으로 수렴시켰다

### `a23fc26a7f82` — reservation ownership 완성

이 커밋은 legacy queue array와 자체 closest search를 삭제합니다. queue count도 Matchmaker의 `queuedCount`를 사용합니다. 더 중요한 변화는 cleanup이 각 기능에 흩어져 있지 않다는 점입니다.

```ts
private releaseMatchmakingReservations(room: Room): void {
  for (const client of Object.values(room.clients)) {
    if (client) this.matchmaker.release(client.user.id);
  }
}
```

이 helper가 호출되는 경로는 다음과 같습니다.

- 정상 경기 종료 뒤 room 제거
- forfeit 결과 확정 뒤 room 제거
- 양쪽 disconnect로 room abandon
- DB finalization 실패로 room ownership을 풀어야 하는 branch
- GameHub close
- drain 중 queue entry 제거

queue 이탈은 `leaveQueue`가 Matchmaker와 `queueEntries`, fallback timer를 함께 정리합니다. stale socket prune도 직접 map을 삭제하지 않고 같은 함수로 들어갑니다.

이 통합이 S인 이유는 matcher가 선택 알고리즘을 제공해서가 아니라, **queued/matched reservation의 terminal owner가 모든 runtime path에서 명시되었기 때문**입니다.

### `b5bfeee0e23e` — room publish 자체도 rollback 대상

`createRoom`은 단순 map insert가 아닙니다.

```text
room map 등록
→ observer.roomCreated
→ 양쪽 client.roomId 설정
→ queue.matched 전송
→ 첫 snapshot/presence 방송
```

observer나 send가 throw하면 일부 publish만 끝난 상태가 될 수 있습니다. 이 커밋은 전체 publish를 `try/catch`로 감싸고 실패 시 다음을 되돌립니다.

- scheduler unregister
- reconnect timer clear
- room map delete
- 양쪽 `client.roomId`가 해당 room이면 null
- drain progress/presence 갱신

그 뒤 오류를 다시 throw하므로 바깥 matching branch가 reservation을 release합니다.

경기 finalization이 DB commit까지 성공한 뒤 observer나 broadcast가 실패하는 경우에는 결과를 rollback할 수 없습니다. 대신 `finally`에서 room 제거와 reservation release를 보장합니다. persistence 성공 뒤 runtime room이 유령 상태로 남지 않게 하는 방향입니다.

## 7. 재매칭 가능 여부로 cleanup을 검증했다

### `112228db8878`

테스트는 내부 map 크기만 보지 않고 사용자가 다시 매칭되는지를 확인합니다.

- rating 1,000과 2,000은 최대 차이 200을 넘으므로 둘 다 queue에 남습니다.
- rating 1,050 사용자가 들어오면 1,000 사용자와 매칭되고 2,000 사용자는 계속 대기합니다.
- forfeit finalization 뒤 두 사용자가 다시 queue에 들어가 서로 매칭됩니다.
- 양쪽 disconnect로 room이 abandon된 뒤에도 두 사용자가 다시 매칭됩니다.
- `roomCreated` observer가 첫 생성에서 throw하면 active room과 queued player가 0으로 돌아오고, 같은 두 사용자의 두 번째 시도는 성공합니다.

마지막 case는 room map rollback과 Matchmaker reservation release가 모두 작동해야 통과합니다. 둘 중 하나라도 남으면 retry가 `duplicate: matched` 또는 active-room 충돌로 막힙니다.

## 최종 acquisition / release 표

| 상태 변화 | 획득한 소유권 | 성공 시 다음 owner | 실패·종료 시 해제 |
| --- | --- | --- | --- |
| enqueue → queued | Matchmaker queue status, queue entry, fallback timer | human match 또는 AI claim | leave/prune/drain/close |
| queued → matched | 두 user의 matched reservation | 생성된 room | missing entry·room create failure |
| AI claim → async lookup | user matched reservation | AI room | stale entry·closed socket·drain·NPC 없음·throw |
| room active | 참가 user reservation | room lifecycle | normal finish·forfeit·abandon·close |
| persistence 성공 뒤 broadcast | DB result는 이미 확정 | 없음 | `finally` room removal/release |

최종 불변 조건은 다음과 같습니다.

- 한 user는 `queued` 또는 `matched` reservation을 하나만 가집니다.
- same-kind 후보 중 rating 차이 200 이내의 가장 가까운 player만 매칭됩니다.
- `matched`는 성공 완료가 아니라 room이 인수해야 할 reservation입니다.
- async 작업 뒤에는 entry/socket/drain/room 상태를 다시 확인합니다.
- queue leave, room failure, finish, abandon, drain, close 모두 reservation을 명시적으로 해제합니다.
- room publish가 중간에 실패하면 map·scheduler·client link를 되돌린 뒤 reservation도 풀립니다.

방이 만들어진 뒤의 reconnect reservation은 Matchmaker reservation과 다른 개념입니다. 전자는 room side의 15초 소유권이며 Thread 05가 담당합니다. 결과 transaction의 rollback은 Thread 04의 책임입니다.
