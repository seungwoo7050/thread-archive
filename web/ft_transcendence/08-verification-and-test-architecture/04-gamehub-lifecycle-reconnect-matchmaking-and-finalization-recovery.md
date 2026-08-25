# Thread: GameHub lifecycle, 재연결, 매칭, 결과 저장 복구

GameHub는 단순한 WebSocket message dispatcher가 아닙니다. queue entry, matched reservation, room, side, active connection, reconnect timer, scheduler registration, finalization promise와 retry timer를 동시에 소유합니다. 이 Thread는 각 자원이 언제 publish되고 어떤 실패에서 회수되어야 하는지를 테스트와 fix로 복원합니다.

가장 중요한 원칙은 다음과 같습니다.

> simulation이 끝났다는 사실과, room의 외부 lifecycle이 끝났다는 사실은 같지 않다. durable result가 확정되고 관련 자원이 정리되기 전까지 room은 완료된 것이 아니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4026c3bf72adfd97868a9f8296c8899ce4d0ff44` | `test(game): 게임 방 상태 전이 검증` | B | `REALTIME, OPERATIONS, TEST` | ready·pause·disconnect·reconnect·forfeit의 순수 RoomSession 규칙을 고정한다. |
| 2 | `113e39acc85c9779c29eb62339bafa884e95b7e5` | `test(game): reconnect 복구 동작 검증` | A | `REALTIME, PERSISTENCE, RISK` | 실제 socket replacement와 15초 reconnect lease가 room identity·side·finalization을 보존하는지 검증한다. |
| 3 | `9d05f47e7f4bcbf12e8ff9ac36d446acd865a691` | `test(ops): GameHub drain과 graceful shutdown 검증` | A | `REALTIME, OPERATIONS, RISK` | 신규 작업 차단, 기존 room 대기, 60초 drain timeout, 중복 shutdown signal의 경계를 검증한다. |
| 4 | `fc7da13e935d4206b743e1eaf706ea9c3b942578` | `test(game): matchmaking 규칙 검증` | A | `REALTIME, TEST` | rating 거리, guest pool 격리, 6초 AI fallback, queue/reservation 상태를 순수 Matchmaker에서 검증한다. |
| 5 | `112228db8878078fae6e9322e258a646e9642e7a` | `test(game): matchmaking lifecycle 검증` | A | `REALTIME, RISK, TEST` | GameHub room 생성·종료·rollback이 reservation을 누수시키지 않는지 통합 검증한다. |
| 6 | `e939a50948b2934e5087ccc8b9169faf91c968e1` | `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지` | A | `REALTIME, RISK` | 일시적 DB 실패 뒤 room을 제거하지 않고 stable key로 finalization을 재시도한다. |
| 7 | `8f5b2e86f69bfa9faad1d942d097e2c6f9961df5` | `test(game): 일시적인 경기 결과 저장 실패 복구 검증` | A | `REALTIME, OBSERVABILITY, RISK` | 첫 저장 실패→250ms 재시도→한 번의 완료 event와 drain 완료를 검증한다. |

## 1. RoomSession은 transport와 무관한 상태 기계다

### `4026c3bf72ad...` — 경계값으로 정의된 lifecycle

`RoomSession` 테스트는 socket이나 repository 없이 방의 합법적인 상태 전이만 검증합니다.

```text
waiting
  ├─ left ready
  ├─ right ready
  └─ both ready → playing

playing
  ├─ pause → paused
  ├─ disconnect(side, now) → reconnect lease
  └─ finish/forfeit → finished
```

핵심 boundary는 reconnect grace 15초입니다.

| 사건 | 시각 | 결과 |
| --- | ---: | --- |
| disconnect | 1,000ms | deadline = 16,000ms |
| reconnect | 15,999ms | 성공, 이전 상태 복구 |
| reconnect | 20,001ms | 실패 |
| expiry 평가 | 정확히 15,000ms 경과 | 한 번의 forfeit result |
| expiry 재평가 | 이미 처리된 뒤 | `null`, 중복 결과 없음 |

둘 다 끊긴 상태에서는 임의 winner를 만들지 않습니다. 이는 transport 장애를 gameplay 승패로 과도하게 해석하지 않기 위한 규칙입니다.

이 commit이 B급인 이유는 GameHub의 자원 회수까지 다루지 않고 순수 상태 규칙을 고정하기 때문입니다. 그러나 뒤의 reconnect·drain·finalization 테스트가 어떤 결과를 기대해야 하는지 정의하는 기준점입니다.

## 2. connection replacement와 실제 disconnect는 다른 사건이다

### `113e39acc85c...` — 같은 사용자, 같은 room, 새 transport

새 WebSocket이 같은 사용자 identity로 연결됐을 때 기존 connection을 그대로 두면 한 player가 두 socket에서 command를 보낼 수 있습니다. 반대로 이를 일반 disconnect로 처리하면 새 connection이 즉시 왔는데도 reconnect timer와 forfeit path가 시작됩니다.

테스트는 두 사건을 분리합니다.

#### connection replacement

```text
old socket(user A, room R, side left)
        +
new socket(user A)
        ↓
old socket close: code 4001, reason "connection replaced"
new socket: room R / side left / 최신 snapshot 수신
old socket에서 뒤늦게 오는 message: 무시
```

replacement는 gameplay absence가 아닙니다. 따라서 15초 뒤 forfeit/finalization이 발생하지 않아야 합니다.

#### 실제 transport 종료와 lease 내 복구

테스트는 old socket을 실제로 terminate한 뒤 14,999ms에 같은 user로 재연결합니다. 새 connection은 새 match를 만들지 않고 기존 room과 side를 회복하며, 이전보다 높은 snapshot sequence를 받습니다. grace timer가 만료되어도 결과 저장은 발생하지 않습니다.

#### lease 만료

정확히 15초가 지나면 상대가 winner인 result가 한 번만 저장됩니다. 저장 command는 room에서 파생된 stable result key를 사용하고, 시간이 더 지나도 duplicate finalization이 생기지 않습니다.

이 suite는 pure `RoomSession` 규칙이 GameHub의 다음 자원과 일치하는지 검증합니다.

- `clients[side]`
- user→room/side 연결
- reconnect timer
- latest snapshot
- repository finalization call

## 3. drain은 “모두 닫기”가 아니라 두 단계의 운영 상태다

### `9d05f47e7f4b...` — accepting과 draining 분리

배포 종료 시 신규 queue·room을 계속 받으면서 기존 room 종료를 기다리면 drain이 끝나지 않습니다. 반대로 즉시 모든 room을 삭제하면 진행 중 경기와 결과 저장이 손실됩니다.

`beginDrain(60_000)` 테스트가 고정하는 상태는 다음과 같습니다.

```text
accepting
  ↓ beginDrain
queued entry 정리
신규 game command → server_draining
기존 active room → 계속 실행/완료 대기
  ├─ 모두 완료 → { drained: true, activeRooms: 0 }
  └─ 60초 도달 → { drained: false, activeRooms: remaining }
```

시간 경계는 fake timer로 직접 확인합니다.

- 59,999ms: drain promise는 아직 settle되지 않습니다.
- 60,000ms: active room이 남아 있으면 timeout result가 resolve됩니다.

이것은 실제 wall-clock으로 60초간 process를 실행한 증거가 아닙니다. timer contract를 결정적으로 검증한 것입니다.

같은 commit의 graceful shutdown 검증은 다음을 확인합니다.

- shutdown signal을 여러 번 받아도 shutdown promise는 하나만 실행됩니다.
- drain 시작 즉시 readiness는 503으로 바뀝니다.
- shutdown 중 error가 나도 동일 절차를 중복 실행하지 않습니다.

따라서 readiness는 단순 DB 상태가 아니라 **이 process가 새 작업을 받아도 되는가**를 반영합니다.

## 4. queue entry와 matched reservation을 하나로 취급하지 않는다

### `fc7da13e935d...` — Matchmaker의 domain 규칙

순수 Matchmaker 테스트는 다음 조건을 고정합니다.

- rating 차이가 허용 범위 안인 가장 가까운 상대를 선택합니다.
- 범위 밖 상대는 queue에 남습니다.
- guest와 registered user는 같은 PvP pool에서 섞이지 않습니다.
- 6초 동안 상대를 찾지 못한 entry만 AI fallback 대상이 됩니다.
- 이미 queued 또는 matched-reserved인 사용자의 중복 join을 거부합니다.
- `leaveQueue`는 아직 queued인 entry만 제거합니다.
- match가 끝나거나 room 생성이 취소되면 reservation을 release할 수 있습니다.

여기서 상태는 최소 세 가지입니다.

```text
not present
    ↓ join
queued
    ↓ pair selected
reserved for room creation
    ↓ room publish success
owned by active room
```

`queued`와 `reserved`를 같은 boolean으로 표현하면 pair가 선택된 뒤 room 생성이 실패했을 때 사용자가 queue에도 없고 재참가도 못 하는 ghost state가 됩니다.

### `112228db8878...` — GameHub가 reservation을 회수하는가

통합 테스트는 Matchmaker가 올바른 pair를 반환하는 것보다 GameHub가 그 결과를 끝까지 책임지는지 확인합니다.

#### rating 기반 실제 room 생성

rating 차이가 큰 두 사용자는 바로 매칭되지 않고, 더 가까운 사용자가 들어오면 해당 pair가 room으로 전환됩니다. pure selection이 실제 `queue.matched`/room state와 연결되는지 보는 테스트입니다.

#### 정상 종료 뒤 release

forfeit나 빈 room abandonment 뒤 reservation이 해제되어 같은 사용자가 다시 queue에 들어갈 수 있어야 합니다. room map에서 사라졌다는 사실만으로 충분하지 않고 Matchmaker 내부 예약도 사라져야 합니다.

#### room publish 도중 실패

observer의 `roomCreated` hook을 첫 시도에 의도적으로 throw하게 합니다. 이 시점에는 pair/reservation, scheduler registration, room/client linkage 가운데 일부가 이미 만들어졌을 수 있습니다.

기대하는 rollback은 다음 terminal state입니다.

```text
activeRooms = 0
queued/reserved 사용자 = 0
client.roomId = null
scheduler에 해당 room 없음
같은 두 사용자 재join 가능
두 번째 시도에서 정상 match 가능
```

이 테스트는 rollback 코드의 개별 line보다 **partial room이 외부에서 존재하지 않는 상태로 수렴하는가**를 확인합니다.

## 5. simulation 완료와 durable finalization 사이의 실패

### 기존 문제

경기가 끝난 뒤 `repo.finalizeMatch(...)`가 일시적으로 실패하면 이전 경로는 reservation을 release하고 failure를 관찰한 뒤 예외를 던질 수 있었습니다. 그러면 다음 상태가 만들어집니다.

```text
simulation: finished
DB result: 없음
client event: 불확실
room/reservation: 정리될 수 있음
retry identity: 유지되지 않음
```

이 상태에서 단순히 room을 제거하면 결과가 영구 손실됩니다. client에게 먼저 `game.finished`를 보내고 나중에 저장을 재시도하면 외부 완료와 durable state가 갈라집니다.

### `e939a50948b2...` — room이 retry를 소유한다

fix는 `Room`에 `finalizationRetryTimer`를 추가하고 persistence success까지 같은 `finishRoom` 흐름 안에서 반복합니다.

```ts
const FINALIZATION_RETRY_BASE_DELAY_MS = 250;
const FINALIZATION_RETRY_MAX_DELAY_MS = 5_000;
```

retry delay는 250ms에서 시작해 지수적으로 증가하되 5초를 넘지 않습니다. 모든 시도는 같은 idempotency key를 사용합니다.

```text
resultKey = `room:${room.id}:finished`
```

핵심 loop는 다음 의미를 가집니다.

```text
while room이 여전히 동일 identity로 rooms map에 존재:
    finalizeMatch(stable resultKey)
      ├─ 성공 → 외부 완료 처리로 진행
      └─ 실패 → observer failure 기록
                  → bounded backoff timer
                  → room이 여전히 살아 있으면 재시도
```

retry timer도 room-owned resource이므로 다음 경로에서 반드시 clear됩니다.

- `abandonRoom`
- `close`
- `removeFinishedRoom`
- 새 retry timer를 잡기 전 기존 timer 교체

`timer.unref?.()`는 retry가 process 자체를 불필요하게 붙잡지 않게 하지만, GameHub drain은 active room이 남아 있으므로 durable success 전까지 완료되지 않습니다.

### 수정된 publish 순서

```text
simulation finished
    ↓
repository finalization retry loop
    ↓ durable success
observer success / guest result 기억 / game.finished 전송
    ↓
room·reservation·timer 정리
```

즉 외부 `game.finished`는 “물리 simulation이 점수에 도달했다”가 아니라 “필요한 durable finalization이 성공했다” 뒤에 publish됩니다.

## 6. `8f5b2e86f69b...` — 실패와 성공 사이의 중간 상태를 검증한다

테스트는 repository spy를 첫 호출에서 reject하고 두 번째 호출에서 성공하도록 설정합니다.

첫 실패 직후 기대 상태:

- `finalizeMatch` 호출 1회
- client의 `game.finished` 0개
- `activeRooms` 1개
- observer에는 failure event

250ms fake timer 진행 뒤:

- `finalizeMatch` 호출 2회
- 두 호출의 `resultKey`가 동일
- `game.finished` 정확히 1개
- `activeRooms` 0개
- observer 순서가 failure → success

두 번째 테스트는 drain과 결합합니다. finalization 첫 실패 뒤 `beginDrain(60_000)` promise는 pending이어야 하며, 250ms 재시도가 성공하면 `{drained: true, activeRooms: 0}`으로 resolve됩니다.

이 회귀 테스트가 막는 것은 단순 “retry가 호출되지 않음”뿐만이 아닙니다.

- 첫 실패 뒤 room을 너무 일찍 제거하는 회귀
- success 전에 finished event를 보내는 회귀
- retry마다 다른 result key를 만드는 회귀
- retry room을 drain 계산에서 빼는 회귀
- 성공 뒤 timer/room을 정리하지 않는 회귀

## 최종 소유권 ledger

| 자원/상태 | owner | publish/획득 시점 | 책임 종료 시점 |
| --- | --- | --- | --- |
| queue entry | Matchmaker | join 성공 | pair reservation 또는 leave/fallback |
| matched reservation | Matchmaker + GameHub room creation | pair 선택 | room publish 성공 뒤 room ownership, 실패 시 rollback/release |
| room scheduler registration | GameHub | room 생성 중 register | finish/abandon/rollback/close에서 unregister |
| room side | GameHub room | match publish | finish/abandon 뒤 client linkage clear |
| active socket | GameHub client slot | connect/replacement | close/replacement/room removal |
| reconnect timer | room | 실제 disconnect | reconnect, expiry, finish, close에서 clear |
| finalization promise | room | simulation finish | durable success 또는 room identity 폐기 |
| finalization retry timer | room | persistence failure | retry fire, success, abandon, close에서 clear |
| stable result key | room identity | 첫 finalization attempt | repository가 기존/신규 한 결과로 수렴 |

## 최종 lifecycle

```text
queue join
  ↓ Matchmaker pair + reservation
room construction
  ├─ 실패 → scheduler/client/room rollback + reservation release
  └─ 성공 → room publish
                 ↓
         ready → playing
            ├─ socket replacement → same room/side, old transport close
            ├─ disconnect → 15s lease
            │      ├─ reconnect → same room 복구
            │      └─ expiry → single forfeit
            └─ normal finish
                    ↓
             durable finalization retry
                    ↓ 성공
             game.finished publish
                    ↓
       scheduler/timers/reservation/room 정리
```

## 이 Thread의 경계

- WebSocket ticket 인증과 frame 제한은 Thread 02입니다.
- simulation 자체의 결정성은 Thread 03입니다.
- PostgreSQL이 stable result key를 동시 호출에서도 한 결과로 수렴시키는지는 Thread 05입니다.
- browser가 실제로 socket을 끊고 fresh ticket으로 복구하는 흐름은 Thread 06입니다.

## 조사·실행 기록

각 설명은 exact SHA의 diff와 해당 시점 source/test를 기준으로 작성했습니다. fake timer assertion을 실제 wall-clock 실행 결과로 확대하지 않았으며, 이 작성 환경에서는 test suite를 실행하지 않았습니다.
