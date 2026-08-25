# Runtime work를 제한하고 종료 가능한 상태로 만들기

실시간 서버는 정상 기능만으로는 운영할 수 없습니다. event loop가 잠시 멈췄을 때 simulation이 무한 catch-up을 시도할 수 있고, 죽은 socket이 timer를 남길 수 있으며, 빠른 client와 느린 client가 각각 input queue와 outbound buffer를 키울 수 있습니다. 배포 신호가 왔는데도 새 match를 계속 받으면 process는 종료되지 않습니다.

이 Thread는 각 종류의 work에 hard bound와 owner를 둡니다.

```text
wall-clock 지연 → 최대 250ms, loop당 5 tick
socket 무응답 → 45초 timeout
input burst → user당 30/s, burst 8
snapshot backlog → latest 1개, 256KiB soft / 1MiB hard / 5초 congestion
room timer → process당 shared scheduler 1개
shutdown → 새 work 차단, active room 최대 60초 drain, 한 번만 close
```

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `3a2943ff385d` | `feat(game): fixed-step scheduler 추가` | A | `SIMULATION, REALTIME, OBSERVABILITY` | monotonic elapsed time을 bounded 50ms simulation work로 변환 |
| `10a656e59864` | `feat(game): WebSocket heartbeat 추가` | A | `REALTIME, RISK` | liveness timer의 시작·ack·종료 ownership 정의 |
| `207df3f47935` | `feat(game): 입력 순서와 rate limit 보호` | A | `SIMULATION, REALTIME, RISK` | room별 monotonic sequence와 user별 token bucket 결합 |
| `8589ff3c4821` | `feat(game): latest snapshot buffer 추가` | A | `SIMULATION, REALTIME, OPERATIONS` | 중간 snapshot을 버리고 혼잡 connection을 종료하는 규칙 정의 |
| `d21a47ee92d2` | `refactor(game): shared room scheduler 추가` | A | `SIMULATION, REALTIME, REFACTOR` | 등록된 모든 room을 하나의 fixed-step clock으로 실행 |
| `fb5b1abc97f5` | `refactor(game): GameHub가 shared room scheduler 사용` | A | `SIMULATION, REALTIME, REFACTOR` | room lifecycle의 runnable membership을 scheduler registry에 연결 |
| `44ef3e07e1a5` | `feat(game): 새 작업 차단과 active room drain 추가` | A | `PROTOCOL, REALTIME, TOURNAMENT` | readiness를 내리고 새 match를 거부한 뒤 기존 room 완료 대기 |
| `1c9981393973` | `feat(ops): graceful shutdown 절차 추가` | A | `REALTIME, PERSISTENCE, OPERATIONS` | SIGTERM/SIGINT를 one-shot drain-and-close sequence로 연결 |
| `7b0b5f086b41` | `test(load): 실시간 fault injection 도구 추가` | A | `AUTH, REALTIME, PERSISTENCE` | edge와 DB 장애를 분리해 connection·snapshot·finalization 지표를 관찰 |

## 1. wall-clock 지연을 simulation tick 수로 바꾸되 추격량을 제한했다

### `3a2943ff385d` — `FixedStepAccumulator`

`setInterval(50)`이 실제로 매번 정확히 50ms에 실행된다는 보장은 없습니다. event loop가 230ms 멈추면 한 번만 step해서 시간을 잃거나, 누적된 모든 시간을 무한히 따라잡아 CPU를 독점할 수 있습니다.

accumulator는 elapsed time을 lag로 모은 뒤 50ms 단위 tick 수를 반환합니다.

```ts
elapsedMs = Math.max(0, nowMs - previousTimeMs);
lagMs = Math.min(250, lagMs + elapsedMs);

availableTicks = Math.floor(lagMs / 50);
ticks = Math.min(5, availableTicks);
lagMs -= ticks * 50;
```

기본값은 다음과 같습니다.

- timestep: 50ms
- 한 loop의 최대 tick: 5
- 누적 lag 상한: 250ms

clock이 뒤로 가거나 non-finite 값을 반환하면 음수 시간을 simulation에 넣지 않습니다. `previousTimeMs`도 시간이 실제로 앞으로 갔을 때만 갱신됩니다.

`FixedStepScheduler`는 start 시 새 accumulator를 만들고 50ms interval에서 `advance`가 반환한 횟수만큼 `step`을 호출합니다. `start`와 `stop`은 idempotent이며, step 중 scheduler가 stop되면 남은 catch-up tick을 실행하지 않습니다.

이 커밋은 scheduler abstraction을 추가합니다. 모든 room이 실제로 이 owner를 사용하도록 통합되는 것은 뒤의 shared scheduler 커밋입니다.

## 2. connection별 inbound/outbound work에 서로 다른 제한을 뒀다

### `10a656e59864` — heartbeat owner

`ConnectionHeartbeat`는 start 시 즉시 45초 timeout을 arm하고, 15초마다 WebSocket ping을 보냅니다. pong이 도착하면 `acknowledge`가 timeout을 다시 45초로 연장합니다.

```text
start
  ├─ timeout(45s) arm
  └─ every 15s: ping

pong
  └─ timeout re-arm

ping throw 또는 45s 무응답
  └─ stop timers → terminate socket
```

`stop`은 interval과 timeout을 모두 clear합니다. connection replacement, normal close, GameHub close가 이 메서드를 호출해야 timer ownership이 끝납니다. heartbeat는 application message가 아니라 WebSocket ping/pong liveness를 관찰합니다.

### `207df3f47935` — stale input과 abusive input을 분리

`InputGate`는 두 map을 가집니다.

- `lastSequences[userId + "\0" + roomId]`
- `buckets[userId]`

sequence는 room별이고 rate budget은 사용자 전체에 공유됩니다. 여러 connection이나 room을 열어도 user가 사용할 수 있는 총 throughput은 기본 30 input/s, burst 8입니다.

검사 순서가 핵심입니다.

```ts
if (inputSeq <= previousSequence) return "stale";

refillUserBucket(nowMs);
if (tokens < 1) return "rate_limited";

tokens -= 1;
lastSequence = inputSeq;
return "accepted";
```

stale input은 token을 소모하지 않습니다. rate-limited input은 last sequence를 전진시키지 않으므로, client가 나중에 더 큰 sequence를 보내 정상 budget에서 다시 진행할 수 있습니다. `releaseUser`는 bucket과 그 user의 모든 room sequence를 지워 connection/user lifecycle 종료 후 map이 누적되지 않게 합니다.

### `8589ff3c4821` — snapshot은 queue가 아니라 최신 값 register

simulation snapshot은 채팅·경기 종료 event와 다릅니다. 늦은 client에게 200개의 과거 frame을 모두 보내는 것보다 최신 state 하나를 보내는 편이 맞습니다. `LatestSnapshotBuffer`는 `pendingSnapshot`을 하나만 보유하며 새 payload가 오면 이전 pending 값을 덮습니다.

혼잡 기준은 다음과 같습니다.

| `socket.bufferedAmount` | 처리 |
| --- | --- |
| `≤ 256 KiB` | pending latest snapshot을 전송 |
| `> 256 KiB` | 50ms 뒤 재시도, 새 snapshot은 latest로 덮어씀 |
| soft threshold가 5초 지속 | socket terminate |
| `≥ 1 MiB` | 즉시 terminate |

동시에 하나의 send callback만 진행합니다. send 중 새 snapshot이 오면 pending slot에 최신 값만 남고 callback 완료 뒤 다시 drain합니다. send가 throw하거나 callback error를 주면 연결을 종료합니다.

이 설계는 의도적으로 lossy입니다. 모든 frame 전달을 보장하지 않는 대신 memory와 latency를 bounded 상태로 유지합니다. 종료·replacement 때 `close`가 pending payload와 retry timer를 지웁니다.

## 3. room마다 timer를 두지 않고 runnable-room registry 하나를 소유했다

### `d21a47ee92d2` — shared scheduler abstraction

`SharedRoomScheduler`는 `roomId → step callback` map과 하나의 `FixedStepScheduler`를 가집니다.

```ts
register(roomId, step) {
  roomSteps.set(roomId, step);
  scheduler.start();
}

unregister(roomId) {
  roomSteps.delete(roomId);
  if (roomSteps.size === 0) scheduler.stop();
}
```

tick에서는 `[...roomSteps.values()]` 복사본을 순회합니다. 한 room callback이 실행 중 자신을 unregister하거나 다른 room을 등록해도 현재 iteration의 map iterator가 손상되지 않습니다. 같은 room ID를 다시 register하면 callback을 교체합니다.

이 커밋만으로 GameHub의 모든 lifecycle path가 registry를 정확히 관리한다고 말할 수는 없습니다. 그 통합은 다음 커밋입니다.

### `fb5b1abc97f5` — registry membership이 실행 가능 상태가 되다

room 객체의 개별 `FixedStepScheduler` field가 삭제되고 GameHub가 하나의 `SharedRoomScheduler`를 소유합니다.

- ready/resume/reconnect 완료: `register(room.id, () => tick(room))`
- pause/disconnect/finalize/abandon/remove: `unregister(room.id)`
- GameHub close: shared scheduler stop

```text
room이 playing이고 실행 가능함
  ⇔ shared scheduler registry에 room ID가 있음
```

중복 unregister는 안전하고, room 수가 0이면 process에 simulation interval이 남지 않습니다. `scheduledRoomCount`는 운영·테스트가 registry state를 직접 관찰할 수 있게 합니다.

이 refactor의 주된 이점은 timer 수 감소만이 아닙니다. room lifecycle의 여러 terminal path가 같은 scheduler owner를 해제하므로 “finished room의 timer가 계속 돈다”는 종류의 leak를 한 지점에서 막습니다.

## 4. drain은 close 전에 새 소유권 취득을 멈춘다

### `44ef3e07e1a5`

process가 SIGTERM을 받은 뒤 즉시 socket과 DB를 닫으면 active match가 사라집니다. 반대로 계속 새 match를 받으면 영원히 drain되지 않습니다. `GameHub.beginDrain`은 먼저 `acceptingMatches = false`로 바꿉니다.

그 뒤 다음을 수행합니다.

1. queue entry와 AI fallback timer를 제거하고 `server_draining` error를 보냅니다.
2. tournament waiter를 제거하고 같은 error를 보냅니다.
3. 새 queue/AI/tournament join을 즉시 거부합니다.
4. 이미 존재하는 room은 계속 실행·finalize하도록 둡니다.
5. room 수가 0이면 `{ drained: true, activeRooms: 0 }`을 반환합니다.
6. timeout까지 room이 남으면 `{ drained: false, activeRooms }`로 resolve합니다.

동시에 여러 caller가 drain을 요청하면 하나의 `drainWaiter.promise`를 공유합니다. room remove/abandon 경로는 `notifyDrainProgress`를 호출해 마지막 room이 사라지는 순간 waiter를 완료합니다.

application readiness도 drain과 함께 내려갑니다.

```text
accepting + DB ready + migrations current → 200 ready

draining → 503 not_ready
           checks.lifecycle = "draining"
```

load balancer가 새 traffic을 보내지 않는 상태와 GameHub가 새 match를 거부하는 상태가 맞춰집니다.

`GameHub.close`는 drain과 다릅니다. close는 queue/tournament/room을 비우고 shared scheduler, reconnect timer, guest result timer, heartbeat, snapshot buffer를 정리한 뒤 open socket을 terminate합니다. drain timeout 뒤에도 unfinished room을 영속 보존하는 기능은 없습니다. timeout은 “기다리기를 끝내고 close로 이동할 수 있다”는 운영 budget입니다.

## 5. SIGTERM과 SIGINT를 한 번의 drain-and-close sequence로 합쳤다

### `1c9981393973`

`installGracefulShutdown`은 두 signal listener를 설치하지만 `started` flag로 첫 signal만 처리합니다.

```text
first SIGTERM/SIGINT
  → log start
  → app.beginDrain(60_000)
  → result log
  → app.close()

later signals
  → no-op
```

shutdown이 reject되면 error name을 기록하고 `process.exitCode = 1`로 설정한 뒤 best-effort `app.close`를 호출합니다. app close hook에서는 signal listener disposer를 실행해 listener ownership도 끝냅니다.

process entrypoint가 sequence를 소유하므로 GameHub, repository, metrics가 각자 signal을 받고 경쟁하지 않습니다. 60초 drain budget은 application 코드에 명시되어 있습니다. container termination grace period가 그보다 짧으면 외부 runtime이 먼저 kill할 수 있으므로 배포 설정과 함께 맞춰야 합니다.

## 6. 부하와 장애를 분리해 관찰하는 도구를 추가했다

### `7b0b5f086b41`

이 커밋은 production fix가 아니라 operational evidence 도구입니다. k6 profile의 기본 규모는 다음과 같습니다.

- 500 connections, extended mode 1,000
- 50 active rooms
- room당 2 player connections
- 최초 연결 유지 90초
- player reconnect delay 10초
- reconnect 후 유지 60초

threshold는 구현 의도를 수치로 표현합니다.

| 지표 | 기준 |
| --- | --- |
| initial connection success | `≥ 99%` |
| reconnect success | `≥ 99%` |
| snapshot delay p95 | `≤ 150ms` |
| snapshot delay p99 | `≤ 250ms` |
| 정상 snapshot drop rate | `< 1%` |
| finalized result 수 | room 수 이상 |
| finalization failure | 0 |
| duplicate finalization | 0 |
| online connection max | 목표 연결의 99% 이상 |
| active room max | configured room 수 이상 |

각 virtual user는 개발 로그인 → one-time ticket → WebSocket 연결을 수행합니다. player는 queue/ready/input을 보내고 일부러 socket을 닫은 뒤 새 ticket으로 같은 room 복구를 확인합니다. snapshot `serverTimeMs`로 delay를 측정하고 sequence gap으로 drop rate를 계산합니다. left side만 finished result를 세어 같은 room/match 중복을 검출합니다.

Toxiproxy는 장애 원인을 둘로 분리합니다.

- PostgreSQL proxy: DB latency/down/up
- edge proxy: reverse-proxy/WebSocket latency, reset, down/up

API의 `DATABASE_URL`은 DB proxy를 향하고, edge test는 별도 proxy port를 사용합니다. 이로써 snapshot delay와 finalization failure가 transport 문제인지 DB 문제인지 구분할 수 있습니다.

중요한 검증 범위가 있습니다. repository에 profile, threshold, proxy control script가 존재한다는 것은 **실행 가능한 관찰 방법이 정의되었다**는 뜻입니다. 이 문서 작성 환경에서는 compose/k6/Toxiproxy suite를 실행하지 않았으므로 해당 threshold를 실제 환경에서 통과했다고 주장하지 않습니다.

## 최종 runtime ownership

| work/resource | owner | hard bound | cleanup |
| --- | --- | --- | --- |
| elapsed simulation work | `FixedStepAccumulator` | 250ms lag, loop당 5 tick | scheduler stop |
| runnable room callbacks | `SharedRoomScheduler` | room ID당 callback 1개, timer 1개 | unregister/stop |
| connection liveness | `ConnectionHeartbeat` | ping 15s, timeout 45s | stop/terminate |
| input throughput | `InputGate` | user당 30/s, burst 8 | `releaseUser` |
| pending snapshot | `LatestSnapshotBuffer` | latest 1개, 256KiB soft, 1MiB hard, 5s | close/terminate |
| drain waiter | `GameHub` | caller 공유 promise, configured timeout | last room 또는 timeout |
| OS signals | process entrypoint | first signal 1개 | app close disposer |
| DB/edge fault evidence | k6 + Toxiproxy | profile/threshold로 명시 | test environment teardown |

최종 불변 조건은 다음과 같습니다.

- event-loop stall이 무제한 simulation catch-up으로 바뀌지 않습니다.
- dead connection, abusive input, slow snapshot consumer가 각자의 hard bound를 넘으면 work를 더 쌓지 않습니다.
- runnable room은 shared registry 하나로 표현되고 모든 stop path가 같은 owner를 해제합니다.
- drain이 시작되면 readiness와 matchmaking admission이 동시에 닫힙니다.
- existing room은 bounded budget 안에서 완료를 시도하고, signal sequence는 한 번만 close로 이어집니다.
- operational harness는 DB와 edge failure를 분리해 connection, recovery, snapshot, finalization 결과를 관찰할 수 있습니다.

snapshot loss는 이 설계의 실패가 아니라 latest-state protocol의 의도적 선택입니다. 모든 frame의 exactly-once delivery를 보장하지 않습니다. drain timeout 뒤 unfinished room의 durable migration, multi-process shared scheduler, 실제 환경 threshold 통과 여부도 이 Thread의 보장 범위 밖입니다.
