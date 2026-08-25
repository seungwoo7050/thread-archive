# Thread: GameHub 런타임 통합 — shared scheduling과 실제 congestion의 경계

Project: `ft_transcendence` · 영역: GameHub runtime integration

## 개요

독립 primitive가 존재한다고 해서 실시간 서버가 보호되는 것은 아니다. scheduler는 room 상태 전이와 함께 시작·중지돼야 하고, heartbeat와 input gate는 connection/user lifecycle에 맞춰 정리돼야 하며, snapshot buffer는 event 종류와 socket pressure를 구분해 적용돼야 한다.

이 Thread는 세 번의 설계 교정을 포함한다.

1. room마다 fixed-step timer를 붙여 simulation을 안정화한다.
2. benchmark 가능한 비교 경계를 만든 뒤 room별 timer를 하나의 shared scheduler로 통합한다.
3. snapshot 양을 줄이고 room별 delivery slot을 분산한 뒤, **send callback 지연을 congestion으로 오판한 초기 정책**을 실제 `bufferedAmount` 기준으로 수정한다.

마지막에는 application message 크기 검사보다 아래인 WebSocket transport 자체에 8KiB frame 상한을 둔다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a6a1f4fba60e` | feat(game): fixed-step scheduler를 GameHub에 연결 | A | `SIMULATION, REALTIME, OBSERVABILITY` | room phase와 per-room fixed-step scheduler lifecycle 연결 |
| 2 | `fc2a4451eed1` | feat(game): heartbeat와 input gate를 GameHub에 연결 | A | `PROTOCOL, REALTIME, RISK` | connection heartbeat와 user-scoped input state를 client lifecycle에 연결 |
| 3 | `49ca3e778801` | feat(game): latest snapshot buffer를 GameHub에 연결 | A | `PROTOCOL, REALTIME, RISK` | snapshot과 control event의 전송 정책 분리 |
| 4 | `400ea1589260` | test(game): GameHub runtime 제한 검증 | B | `PROTOCOL, REALTIME, TEST` | 실제 GameHub message path에서 stable rate-limit error 검증 |
| 5 | `aed88c8a93e0` | perf(game): scheduler benchmark 실행 경계 추가 | B | `REALTIME, OBSERVABILITY, PERF` | room별 timer와 shared timer를 같은 synthetic work로 비교하는 측정 함수 추가 |
| 6 | `8d24b5e70837` | perf(game): scheduler benchmark 측정 결과 출력 | B | `REALTIME, OBSERVABILITY, PERF` | room count별 반복 측정·환경 정보·선택 decision JSON 출력 |
| 7 | `d21a47ee92d2` | refactor(game): shared room scheduler 추가 | A | `SIMULATION, REALTIME, REFACTOR` | 여러 room callback을 하나의 fixed-step timer로 구동하는 owner 도입 |
| 8 | `518a8368e28f` | test(game): shared room scheduler 검증 | B | `REALTIME, TEST` | one-timer, unregister, tick 중 mutation 안전성 검증 |
| 9 | `fb5b1abc97f5` | refactor(game): GameHub가 shared room scheduler 사용 | A | `SIMULATION, REALTIME, REFACTOR` | room별 timer state 제거와 모든 room 종료 경로의 unregister 통합 |
| 10 | `69fb44d2f0ca` | test(game): shared scheduler lifecycle 검증 | A | `AUTH, SIMULATION, REALTIME` | disconnect/reconnect/finalize에서 scheduled room count 검증 |
| 11 | `ad482c200cea` | fix(game): 부하 중 snapshot cadence 안정화 | A | `SIMULATION, REALTIME, PERF` | 20Hz simulation을 유지하며 10Hz snapshot을 room별 stagger |
| 12 | `db1ae3d47b96` | test(load): 기본 부하 병목 구간 검증 | B | `SIMULATION, REALTIME, OPERATIONS` | cadence와 finalization/load 관측 계약을 회귀 테스트로 고정 |
| 13 | `d90f17fa765d` | fix(game): callback 지연을 snapshot congestion으로 오판하지 않음 | A | `REALTIME, PERF, RISK` | in-flight callback serialization을 제거하고 actual buffer pressure만 congestion으로 사용 |
| 14 | `5cd54767858f` | test(game): callback 지연과 실제 congestion 구분 | A | `REALTIME, PERF, TEST` | delayed callback과 soft-buffer replacement를 서로 다른 case로 검증 |
| 15 | `8ea18a1b92db` | fix(realtime): WebSocket transport payload 상한 설정 | A | `AUTH, REALTIME, RISK` | parser 이전 transport layer에 8KiB max payload 적용 |
| 16 | `1afec49052b6` | test(realtime): oversized WebSocket frame 거부 검증 | B | `AUTH, REALTIME, TEST` | 인증 완료 socket에서도 8KiB 초과 frame이 1009로 닫히는지 검증 |

## 1. Primitive를 domain lifecycle에 연결한다

### `a6a1f4fba60e` — room phase가 scheduler의 owner가 된다

기존 room은 `setInterval` handle을 직접 들고 있었다. 수정 후 room은 `FixedStepScheduler | null`을 소유하고, playing으로 들어갈 때 start, pause·finish에서 stop한다.

```ts
private startRoomScheduler(room: Room): void {
  room.scheduler ??= new FixedStepScheduler(() => this.tick(room), {
    timestepMs: SIMULATION_TIMESTEP_MS,
    maxTicksPerLoop: 5,
    maxAccumulatedMs: 250
  });
  room.scheduler.start();
}
```

ready transition은 단순히 timer가 없는지를 보지 않고 `phase === "waiting"`인지 확인한다. pause는 scheduler를 멈추고 phase를 paused로 바꾸며, resume은 같은 scheduler를 다시 시작한다. finalize는 scheduler를 먼저 멈춘 뒤 room을 finished로 전이한다.

`tick()`은 synchronous simulation step으로 바뀌고, finish persistence는 scheduler callback을 막지 않도록 `finishRoom(...).catch(...)`로 분리된다. 이 SHA의 핵심은 error를 해결했다는 것이 아니라 simulation timer callback이 unbounded async chain을 직접 await하지 않도록 boundary를 바꾼 것이다.

### `fc2a4451eed1` — connection과 user state의 종료 시점을 맞춘다

client state에서 per-connection sequence map을 제거하고 heartbeat와 snapshot buffer를 소유하게 한다. `connect()`는 다음 순서로 connection resource를 publish한다.

```text
heartbeat 생성
→ Client 객체에 heartbeat/snapshot buffer 저장
→ message, pong, close listener 등록
→ heartbeat.start()
→ pending pre-auth payload 처리
```

`pong`은 `heartbeat.acknowledge()`로 deadline을 연장한다. `disconnect()`는 이미 제거된 client면 즉시 반환해 cleanup을 idempotent하게 만들고, heartbeat와 snapshot buffer를 먼저 닫은 뒤 queue/room/client registry를 정리한다.

input rate state는 connection이 아니라 user owner다. 같은 사용자가 여러 WebSocket을 가질 수 있으므로 한 connection이 닫힐 때 무조건 `releaseUser()`를 호출하지 않는다.

```ts
this.clients.delete(client.id);
if (![...this.clients.values()].some(
  (candidate) => candidate.user.id === client.user.id
)) {
  this.inputGate.releaseUser(client.user.id);
}
```

게임 입력은 stale이면 조용히 무시하고, budget을 넘으면 protocol-stable `rate_limited` error를 보낸다. accepted일 때만 paddle direction이 바뀐다.

### `49ca3e778801` — snapshot과 control event를 같은 queue에 넣지 않는다

`send()`는 event type에 따라 정책을 나눈다.

```ts
if (client.socket.readyState !== WebSocket.OPEN) return;
const payload = encodeServerEvent({ ...event, v: 1 } as ServerEvent);
if (event.type === "game.snapshot") {
  client.snapshots.enqueue(payload);
  return;
}
if (client.socket.bufferedAmount >= HARD_BUFFERED_AMOUNT_BYTES) {
  client.socket.terminate();
  return;
}
client.socket.send(payload, (error) => {
  if (error && client.socket.readyState === WebSocket.OPEN) {
    client.socket.terminate();
  }
});
```

snapshot은 시간이 지나면 가치가 사라지므로 latest-value buffer로 보낸다. `queue.matched`, error, game finished 같은 lifecycle/control event는 중간값을 버리면 protocol 의미가 깨지므로 직접 전송한다. 다만 non-snapshot도 이미 hard limit에 도달한 connection에는 더 쌓지 않고 종료한다.

### `400ea1589260` — integration evidence의 실제 범위

fake WebSocket과 memory repository로 AI room을 만들고 ready 후 9개 input을 연속 전송한다. 기본 burst 8을 모두 쓴 뒤 error event가 정확히 하나이며 `code=rate_limited`인지 확인한다.

이 commit은 heartbeat timeout, snapshot congestion, scheduler catch-up을 GameHub 수준에서 모두 검증하지 않는다. 제목은 runtime 제한 전체처럼 보이지만 실제 case는 input gate integration 하나다.

## 2. Room별 timer를 shared scheduler로 바꾸는 근거와 구현

### `aed88c8a93e0`, `8d24b5e70837` — benchmark는 decision procedure다

benchmark script는 1, 20, 50, 100 room에서 두 전략을 같은 50ms timestep과 동일한 synthetic trigonometric work로 비교한다.

- `room`: room마다 `setInterval` 하나
- `shared`: `setInterval` 하나가 모든 room step을 순회

각 run은 250ms warm-up 뒤 p95/p99 timer lag를 수집하고, 기본 3회 결과의 median을 JSON으로 출력한다. runtime에는 Node version, OS, CPU model/count, memory, duration과 sample count가 포함된다.

50-room decision은 다음 규칙이다.

```ts
const thresholdMs = room50.p95LagMs * 1.05;
const selectedStrategy = shared50.p95LagMs <= thresholdMs
  ? "shared"
  : "room";
```

이 commit들은 benchmark를 실행하면 결과를 만들 수 있게 하지만 repository에 특정 실행 결과를 저장하지 않는다. 따라서 shared가 실제 환경에서 더 빨랐다고 이 history만으로 단정할 수 없다. 확정 가능한 것은 “shared p95가 room별 p95의 105% 이내면 timer 수 감소를 선택한다”는 decision rule이다.

### `d21a47ee92d2` — one timer, many room callbacks

`SharedRoomScheduler`는 room ID→step callback map과 하나의 `FixedStepScheduler`를 소유한다.

```ts
register(roomId: string, step: () => void): void {
  this.roomSteps.set(roomId, step);
  this.scheduler.start();
}

unregister(roomId: string): void {
  this.roomSteps.delete(roomId);
  if (this.roomSteps.size === 0) this.scheduler.stop();
}

private stepRooms(): void {
  for (const step of [...this.roomSteps.values()]) step();
}
```

동일 room ID 재등록은 callback을 replace하며 timer를 추가하지 않는다. 마지막 room이 빠질 때 underlying timer가 멈춘다. callback 목록을 배열로 복사한 뒤 순회하므로 한 room이 자기 callback에서 unregister돼도 뒤 room의 step을 건너뛰지 않는다.

### `518a8368e28f` — 구조상 중요한 두 case

- room 2개를 등록해도 fake timer count는 1이다.
- 첫 room callback이 tick 도중 자기 자신을 unregister해도 두 번째 room callback은 같은 tick에서 실행된다.

테스트는 callback 간 CPU 공정성이나 한 room throw가 다른 room에 미치는 영향은 다루지 않는다. map/timer lifecycle과 iteration mutation만 고정한다.

### `fb5b1abc97f5` — GameHub의 room timer state를 제거한다

room 객체에서 scheduler field를 없애고 GameHub가 하나의 `roomScheduler`를 소유한다. start는 `register(room.id, () => this.tick(room))`, 중지는 `unregister(room.id)`로 통일된다.

중요한 것은 happy path만 바꾼 것이 아니라 room이 active set에서 빠지는 모든 경로를 훑었다는 점이다.

| room transition | shared scheduler 처리 |
| --- | --- |
| ready/resume | register |
| player disconnect로 pause | unregister |
| explicit pause | unregister |
| abandon | unregister |
| finalize 시작 | unregister |
| finished room 제거 | 방어적으로 unregister |

`scheduledRoomCount` getter는 test/observability를 위해 active registration 수를 노출한다.

### `69fb44d2f0ca` — reconnect lifecycle의 registration 증거

기존 reconnect test에 active count assertion을 추가한다.

```text
게임 시작          → scheduledRoomCount = 1
player disconnect  → 0
reconnect 성공     → 1
forfeit/finalize   → 0
```

이 case는 authentication/session recovery 흐름과 shared scheduler lifecycle이 함께 맞물리는지를 검증한다. 동시에 여러 room 또는 drain 중 scheduler behavior는 다른 tests의 범위다.

## 3. Simulation은 20Hz로 유지하고 delivery burst만 줄인다

### `ad482c200cea`

부하 병목을 줄이기 위해 simulation timestep을 50ms에서 100ms로 바꾸지 않는다. simulation은 계속 20Hz로 진행하고 snapshot broadcast만 2 tick에 한 번 수행한다.

각 room에는 생성 순서대로 0 또는 1 delivery slot이 배정된다.

```ts
const snapshotDeliverySlot = this.nextSnapshotDeliverySlot;
this.nextSnapshotDeliverySlot =
  (this.nextSnapshotDeliverySlot + 1) % SNAPSHOT_DELIVERY_DIVISOR;
```

```ts
if (
  (room.simulation.tick + room.snapshotDeliverySlot)
    % SNAPSHOT_DELIVERY_DIVISOR === 0
) {
  this.broadcastSnapshot(room);
}
```

그 결과 room별 snapshot cadence는 10Hz지만 인접 room은 서로 다른 tick에 broadcast한다. “모든 room이 100ms마다 동시에 serialize/send”하는 burst를 피하면서 physics/input resolution은 20Hz로 유지한다.

같은 commit은 database finalization 결과의 `created` flag를 observer로 전달하고, `created=false`인 idempotent duplicate를 별도 counter로 기록한다. cadence fix와 직접 같은 알고리즘은 아니지만 load 결과에서 “성공 2회”와 “한 결과를 두 번 관찰”을 구분하기 위한 관측 보강이다.

### `db1ae3d47b96`

새 cadence test는 두 AI room을 동시에 시작하고 4개의 50ms tick을 진행한다.

- 각 tick의 두 room delivery 합은 정확히 1
- 첫 room 총 2회, 둘째 room 총 2회

따라서 simulation tick마다 전체 room이 동시에 보내지 않으면서 각 room이 절반 cadence를 받는다는 것을 검증한다. 추가 test는 forfeit finalization observer가 `created=true`를 받는지와 duplicate metric contract를 확인하고, load harness source가 reconnect/finalization 지표를 참조하는지 검사한다.

이는 fake timer와 source contract의 증거다. 실제 50-room network load에서 p95가 개선됐다는 실행 결과는 아니다.

## 4. Callback 지연과 socket congestion은 같은 상태가 아니다

### 이전 가정

초기 `LatestSnapshotBuffer`는 `sending=true`인 동안 다음 payload를 보내지 않고 pending 한 칸에서 최신값만 유지했다. 이 모델은 “send callback이 아직 오지 않음”을 “socket이 밀림”과 동일하게 취급한다.

하지만 `ws.send` callback이 늦게 실행되는 이유는 event-loop scheduling일 수도 있다. `bufferedAmount`가 낮아 transport가 payload를 수용할 수 있는데도 callback만 늦다면 snapshot을 불필요하게 drop하게 된다.

### `d90f17fa765d` — root-cause fix

수정은 `sending` state와 callback 완료 뒤 `drain()` 재호출을 제거한다. `drain()` 시점에 `bufferedAmount`가 soft limit 이하라면 pending 값을 즉시 `send()`하고 끝낸다. callback은 success delay를 기록하거나 error로 connection을 종료할 뿐 다음 전송 허가권이 아니다.

```diff
- private sending = false;
  private congestionStartedAtMs: number | null = null;

  this.congestionStartedAtMs = null;
- if (this.sending) {
-   this.armRetry();
-   return;
- }
-
  const snapshot = this.pendingSnapshot;
  if (snapshot === null) return;
  this.pendingSnapshot = null;
- this.sending = true;
  try {
    this.socket.send(snapshot.payload, (error) => {
-     this.sending = false;
      if (error) {
        this.onDropped("connection_closed");
        this.terminate("connection_closed");
        return;
      }
      this.onDelivered(Math.max(0, this.now() - snapshot.enqueuedAtMs));
-     this.drain();
    });
  } catch {
-   this.sending = false;
    this.onDropped("connection_closed");
    this.terminate("connection_closed");
    return;
  }
- if (this.sending || this.pendingSnapshot !== null) this.armRetry();
```

최신값 replacement 정책이 사라진 것은 아니다. `bufferedAmount > 256KiB`인 실제 congestion 구간에서는 pending 한 칸을 유지하므로 새 snapshot이 이전 pending 값을 대체한다. 바뀐 것은 **replacement를 시작하는 조건**이다.

### `5cd54767858f` — 두 상태를 서로 다른 fixture로 고정한다

1. `bufferedAmount=0`, callback 3개 미완료: snapshot 1·2·3이 모두 `send()`되고 drop callback은 없다.
2. snapshot 1 전송 후 `bufferedAmount`를 soft limit 초과로 변경: 2와 3 중 2가 대체되고 pressure 해제 뒤 3만 전송된다.

fake socket도 completion 하나가 아니라 queue를 보유하도록 바뀌어 여러 in-flight callback을 표현한다. 이 test는 callback 지연을 congestion으로 오판하는 regression을 정확히 겨냥한다.

## 5. Frame 크기 상한은 authentication state보다 아래에 둔다

### `8ea18a1b92db`

기존 pre-auth logic에는 pending payload byte 수 제한이 있었지만, 인증 후에는 transport가 큰 frame을 먼저 받아 application parser까지 넘길 수 있었다. Fastify WebSocket plugin 등록 시 기존 8KiB constant를 `ws`의 `maxPayload`로 전달한다.

```ts
await realtime.register(websocket, {
  options: { maxPayload: PRE_AUTH_MESSAGE_MAX_BYTES }
});
```

이 상한은 pre-auth queue나 authenticated GameHub handler보다 아래에서 모든 frame에 적용된다. 8KiB 초과 시 `ws`가 close code 1009를 생성하므로 application custom reason 대신 빈 reason이 관찰된다.

### `1afec49052b6`

login→one-time ticket→WebSocket accepted까지 완료한 뒤 `Buffer.alloc(8 * 1024 + 1)`을 보낸다. 기대 결과는 `{ code: 1009, reason: "" }`다. 앞 commit에서 기존 pre-auth case의 기대값도 같은 transport close로 갱신했으므로 인증 전·후 모두 같은 하위 경계에서 막힌다.

테스트는 8KiB+1 거부를 검증하며 fragmented message, compressed expansion, exactly 8KiB payload의 application schema 유효성까지 포괄하지 않는다.

## 최종 runtime 구조

```text
[GameHub]
  ├─ SharedRoomScheduler 1개
  │    └─ active room callbacks
  │         simulation 20Hz / catch-up 최대 5 tick
  │         snapshot delivery는 room별 10Hz, slot 0/1 stagger
  ├─ InputGate 1개
  │    └─ token bucket은 user 단위, sequence는 user+room 단위
  └─ Client마다
       ├─ ConnectionHeartbeat (15s ping / 45s deadline)
       └─ LatestSnapshotBuffer
            ├─ bufferedAmount 정상: callback 미완료여도 send 가능
            ├─ >256KiB: pending 최신값만, 50ms retry
            ├─ 5s 지속 또는 >=1MiB: terminate
            └─ snapshot 외 control event는 직접 send

[WebSocket transport]
  └─ 인증 상태와 무관하게 maxPayload 8KiB
```

## 이 Thread의 경계

- 독립 primitive의 원래 알고리즘과 initial snapshot serialization contract는 `03-runtime-limiter-primitives-and-bounded-work`에서 다룬다.
- metric callback과 bounded label 설계는 `02-metrics-observer-boundaries-and-cardinality`의 관심사다.
- 새 match admission을 중단하고 active room을 기다리는 lifecycle은 `05-draining-readiness-and-graceful-shutdown`에 속한다.
- k6 500-connection 실행과 Toxiproxy recovery automation은 `06-load-fault-recovery-and-pool-error-containment`에서 다룬다.
- benchmark와 tests는 source/diff로 확인했으며 실제 실행 결과를 주장하지 않는다.
