# Thread: 런타임 작업을 유계화하는 네 가지 primitive

Project: `ft_transcendence` · 영역: simulation, WebSocket lifecycle, rate limit, backpressure

## 개요

실시간 서버는 하나의 “rate limit”으로 보호되지 않는다. 서로 다른 폭주 원인마다 상태와 종료 조건이 다르다.

- event loop가 늦었을 때 simulation catch-up을 몇 tick까지 허용할 것인가
- pong이 오지 않는 connection을 언제 종료할 것인가
- 같은 사용자가 중복·역순 입력 또는 과도한 입력을 보낼 때 어떤 상태를 소비할 것인가
- socket buffer가 밀릴 때 snapshot을 얼마나 보존하고 언제 connection을 포기할 것인가

이 Thread는 이 네 문제를 GameHub에 연결하기 전, 독립적으로 테스트 가능한 작은 상태 기계로 만든다. 따라서 여기의 최종 상태는 **primitive contract**이며 production room/client lifecycle 통합은 다음 Thread의 책임이다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3a2943ff385d` | feat(game): fixed-step scheduler 추가 | A | `SIMULATION, REALTIME, OBSERVABILITY` | monotonic elapsed time을 최대 5개의 50ms step으로 변환 |
| 2 | `0888e119036d` | test(game): fixed-step 보정 범위 검증 | B | `SIMULATION, REALTIME, TEST` | 경계 시간, 큰 지연, 역행 clock, stop semantics 검증 |
| 3 | `10a656e59864` | feat(game): WebSocket heartbeat 추가 | A | `REALTIME, RISK` | 15초 ping과 45초 pong deadline의 단일 timer owner 도입 |
| 4 | `81031dcd2c1c` | test(game): heartbeat timeout 검증 | B | `REALTIME, TEST` | timeout과 acknowledge deadline 연장 검증 |
| 5 | `207df3f47935` | feat(game): 입력 순서와 rate limit 보호 | A | `SIMULATION, REALTIME, RISK` | user token bucket과 user-room sequence gate 결합 |
| 6 | `1353e3eb99cc` | test(game): input gate 제한 검증 | B | `REALTIME, TEST` | burst/sustained rate, stale 무비용 처리, user scope 검증 |
| 7 | `8589ff3c4821` | feat(game): latest snapshot buffer 추가 | A | `SIMULATION, REALTIME, OPERATIONS` | soft/hard backpressure와 latest-value pending buffer 도입 |
| 8 | `125aa113a01c` | test(game): snapshot replacement와 congestion 검증 | A | `REALTIME, PERF, RISK` | replacement, pressure 해제, 즉시/지속 congestion 종료 검증 |

## 1. Fixed step: wall-clock 지연을 무한 catch-up으로 바꾸지 않는다

### `3a2943ff385d`

`FixedStepAccumulator`는 실제 경과 시간을 simulation step 수로 바꾼다. 기본값은 50ms step, loop당 최대 5 tick, 누적 lag 최대 250ms다.

```ts
advance(nowMs: number): number {
  if (!Number.isFinite(nowMs)) return 0;
  const elapsedMs = Math.max(0, nowMs - this.previousTimeMs);
  if (nowMs > this.previousTimeMs) this.previousTimeMs = nowMs;
  this.lagMs = Math.min(
    this.maxAccumulatedMs,
    this.lagMs + elapsedMs
  );

  const availableTicks = Math.floor(this.lagMs / this.timestepMs);
  const ticks = Math.min(this.maxTicksPerLoop, availableTicks);
  this.lagMs -= ticks * this.timestepMs;
  return ticks;
}
```

핵심은 정확한 시간을 모두 따라잡는 것이 아니라 callback 하나가 수행할 work를 제한하는 것이다. process가 10초 멈췄더라도 200 tick을 한 번에 실행하지 않는다. 최대 250ms만 인정해 5 tick을 실행하고 나머지 지연은 버린다. 이는 완벽한 wall-clock 보존보다 event-loop starvation 방지를 선택한 정책이다.

`FixedStepScheduler`는 하나의 interval에서 accumulator가 반환한 횟수만큼 `step()`을 호출한다. `start()`는 idempotent이고 `stop()`은 timer와 accumulator를 정리한다. step 도중 stop되면 남은 catch-up loop를 계속 실행하지 않는다.

### `0888e119036d`

fake clock 테스트가 다음 경계를 고정한다.

- 49ms에는 0 tick, 50ms에는 1 tick
- 10초 jump에도 5 tick만 실행하고 누적값은 0으로 잘림
- clock이 100→80으로 역행하면 0 tick이며 previous time을 뒤로 옮기지 않음
- scheduler stop 이후 timer를 진행해도 step count가 늘지 않음

이 테스트는 실제 OS scheduler jitter나 여러 room의 CPU 비용을 측정하지 않는다. 순수 시간 변환 상태 기계만 검증한다.

## 2. Heartbeat: connection deadline의 timer owner를 하나로 만든다

### `10a656e59864`

`ConnectionHeartbeat`는 ping interval과 inactivity timeout을 함께 소유한다.

```ts
export const HEARTBEAT_PING_INTERVAL_MS = 15_000;
export const HEARTBEAT_TIMEOUT_MS = 45_000;

start(): void {
  if (this.pingTimer || this.timeoutTimer) return;
  this.armTimeout();
  this.pingTimer = setInterval(() => {
    try {
      this.target.ping();
    } catch {
      this.terminate();
    }
  }, HEARTBEAT_PING_INTERVAL_MS);
}

acknowledge(): void {
  if (!this.pingTimer && !this.timeoutTimer) return;
  this.armTimeout();
}
```

pong을 받으면 새 timeout만 arm하고 ping cadence는 바꾸지 않는다. ping 자체가 throw하거나 45초 동안 acknowledge가 없으면 `stop()`으로 두 timer를 모두 지운 뒤 target connection을 terminate한다. start/stop을 반복 호출해도 중복 timer가 생기지 않는다.

### `81031dcd2c1c`

- 30초 동안 ping 2회, terminate 0회
- 45초 도달 시 terminate 1회, 추가 ping 없음
- 44초에 acknowledge하면 deadline이 그 시점부터 45초 뒤로 이동

실제 WebSocket `pong` event 연결이나 disconnect cleanup은 아직 이 commit 범위가 아니다.

## 3. Input gate: stale 입력은 rate budget을 쓰지 않는다

### `207df3f47935`

`InputGate`는 서로 다른 두 상태를 결합한다.

- token bucket: **user 단위**, 기본 30 input/s, burst 8
- last sequence: **user + room 단위**

검사 순서가 contract의 핵심이다.

```ts
const sequenceKey = `${command.userId}\u0000${command.roomId}`;
const previousSequence = this.lastSequences.get(sequenceKey);
if (previousSequence !== undefined && command.inputSeq <= previousSequence) {
  return "stale";
}

const bucket = this.refill(command.userId, command.nowMs);
if (bucket.tokens < 1) return "rate_limited";

bucket.tokens -= 1;
this.lastSequences.set(sequenceKey, command.inputSeq);
return "accepted";
```

중복·과거 sequence는 token refill/차감 전에 빠져나간다. 공격자가 stale packet을 반복해 정상 입력의 budget을 고갈시키지 못한다. 반대로 새 sequence가 rate limited되면 last sequence를 publish하지 않으므로, budget이 회복된 뒤 같은 sequence를 다시 보낼 수 있다.

bucket은 user 전체 room이 공유한다. room마다 별도 bucket을 두면 room을 바꾸어 제한을 우회할 수 있기 때문이다. `releaseUser()`는 user bucket과 해당 user의 모든 room sequence를 함께 지운다.

### `1353e3eb99cc`

테스트는 다음을 검증한다.

- 시각 0에 8개 burst 허용, 9번째 거부
- 이후 100ms마다 3개씩, 1초 동안 총 30/s refill
- accepted 10 뒤 duplicate 10과 older 9는 stale이며 token을 쓰지 않아 11이 accepted
- 같은 user의 두 room은 capacity를 공유하고 다른 user는 격리

clock이 뒤로 가거나 동일하면 refill하지 않는 구현도 source에서 확인되지만, 이 commit의 test에는 별도 역행 clock case가 없다.

## 4. Latest snapshot buffer: 유한 대기와 connection termination

### `8589ff3c4821`

초기 `LatestSnapshotBuffer`는 다음 상수를 사용한다.

```ts
export const SOFT_BUFFERED_AMOUNT_BYTES = 256 * 1_024;
export const HARD_BUFFERED_AMOUNT_BYTES = 1_024 * 1_024;
export const MAX_CONGESTION_MS = 5_000;
const RETRY_INTERVAL_MS = 50;
```

상태는 pending snapshot 하나, send 진행 여부, congestion 시작 시각, retry timer, closed flag다.

- `enqueue()`는 pending 값을 항상 최신 payload로 바꾸고 `drain()`을 시도한다.
- socket이 open이 아니면 buffer를 닫는다.
- `bufferedAmount >= 1MiB`면 즉시 terminate한다.
- `bufferedAmount > 256KiB`면 50ms 뒤 재시도하며, 5초 지속되면 terminate한다.
- 낮은 pressure에서 send가 진행 중이면 새 값은 pending 한 칸에서 최신값으로 교체된다.
- send callback error 또는 synchronous throw는 terminate로 수렴한다.
- `close()`는 pending과 retry timer를 버리고 이후 enqueue를 무시한다.

이 initial design은 callback completion 전에는 다음 send를 시작하지 않는다는 가정을 둔다. 따라서 callback 지연 중 snapshot 2와 3이 들어오면 2는 3으로 대체된다. 후속 integration history는 이 가정이 실제 socket congestion과 같지 않다는 것을 발견해 수정한다.

### `125aa113a01c`

네 테스트가 초기 contract를 고정한다.

| 상황 | 기대 결과 |
| --- | --- |
| snapshot 1 send 중 2, 3 enqueue | 1 전송 후 3만 전송 |
| soft limit 상태에서 1, 2 enqueue 후 pressure 해제 | 50ms retry에서 최신 2 전송 |
| exactly 1MiB buffered | send 없이 즉시 terminate |
| soft limit 초과가 4,999ms/5,000ms | 전자는 유지, 후자는 terminate |

fake socket은 send callback을 test가 직접 완료할 수 있게 한다. 실제 kernel/socket buffer, network throughput, callback scheduling을 측정하는 테스트는 아니다.

## Primitive별 최종 불변 조건

| Primitive | 유계 상태 | 종료/정리 조건 | 아직 연결하지 않은 owner |
| --- | --- | --- | --- |
| fixed-step | 50ms, loop당 5 tick, lag 250ms | scheduler stop | room lifecycle |
| heartbeat | ping 15s, deadline 45s | pong 재arm, timeout/ping error terminate | WebSocket connection |
| input gate | user bucket 30/s·burst 8, user-room sequence | 마지막 user connection에서 release 예정 | GameHub client registry |
| snapshot buffer | pending 1개, soft 256KiB, hard 1MiB, 5s | close 또는 terminate | GameHub event routing |

## 이 Thread의 경계

- primitive를 실제 room/client/send path에 연결하는 작업은 `04-gamehub-runtime-integration-shared-scheduling-and-congestion`에서 다룬다.
- 초기 snapshot buffer의 “callback in flight = congestion” 가정은 `d90f17fa765d`에서 폐기된다. 이 후대 수정 결과를 `8589ff3c4821`의 원래 의도에 소급하지 않는다.
- metrics callback이 추가되는 과정은 `02-metrics-observer-boundaries-and-cardinality`의 관측 Thread에 속한다.
- source/diff만 확인했으며 timer test나 실제 WebSocket traffic을 실행하지 않았다.
