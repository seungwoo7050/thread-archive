# Thread: benchmark, load, fault recovery 검증

성능·복구 문서에서 가장 위험한 오류는 “실행할 수 있는 harness가 있다”와 “실제 환경에서 목표를 달성했다”를 같은 증거로 취급하는 것입니다. 이 Thread는 다음 네 층을 분리합니다.

1. scheduler topology를 격리한 microbenchmark
2. 실제 service를 대상으로 한 k6 load/fault 도구
3. load가 드러낸 snapshot burst의 production 수정
4. Toxiproxy fault sequence와 cleanup/report를 검증하는 orchestrator

```text
격리 측정              실제 service 부하
microbenchmark    ≠     k6/Toxiproxy run
       │                       │
       └──── decision input ───┘
                    ↓
        production cadence 수정
                    ↓
       반복 가능한 recovery scenario
```

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `aed88c8a93e0c9d987e80a23d5180b85f07e2b78` | `perf(game): scheduler benchmark 실행 경계 추가` | B | `REALTIME, OBSERVABILITY, PERF` | room별 timer와 shared timer를 같은 합성 workload에서 비교하는 측정 함수를 만든다. |
| 2 | `8d24b5e70837afe3762622a29e802e40bc66071e` | `perf(game): scheduler benchmark 측정 결과 출력` | B | `REALTIME, OBSERVABILITY, PERF` | 반복 median·환경 metadata·5% 선택 기준을 JSON report로 출력한다. |
| 3 | `ff1bffcd5296ed89fca6f42b070f6ecac4c0eb65` | `test(load): 실시간 부하 임계값 정의` | B | `REALTIME, PERSISTENCE, OPERATIONS` | load profile·metric·threshold·proxy wiring이 drift하지 않도록 정적 실행 테스트로 고정한다. |
| 4 | `7b0b5f086b4193b9585e5a8bf6e55ec6e52fb447` | `test(load): 실시간 fault injection 도구 추가` | A | `AUTH, REALTIME, PERSISTENCE` | k6 realtime load와 DB/edge Toxiproxy fault 주입 도구를 구축한다. |
| 5 | `ad482c200cea41429c48419fc564069751505d5c` | `fix(game): 부하 중 snapshot cadence 안정화` | A | `SIMULATION, REALTIME, PERF` | simulation 20Hz를 유지하면서 snapshot을 room별 staggered 10Hz로 전달한다. |
| 6 | `84bec3bf57ae15af185ed17ce935008438461758` | `test(load): fault recovery 검사 자동화` | A | `PERSISTENCE, OPERATIONS, PERF` | DB/edge fault와 readiness recovery를 순서화하고 versioned JSON report를 만든다. |
| 7 | `335565908920371a20f939ae27df5bcd985792c9` | `test(load): fault scenario 설정과 report 검증` | B | `PERSISTENCE, OPERATIONS, PERF` | 외부 proxy 없이 command 순서·polling·report·실패 후 reset을 결정적으로 검증한다. |

> commit map의 순서는 기존 metadata를 유지합니다. 본문의 의존 관계는 “도구를 만든 `7b0b...`와 그 계약을 고정한 `ff1b...`”처럼 실제 source 관계를 기준으로 설명하며, 억지로 선형 개발 순서로 바꾸지 않습니다.

## 1. scheduler topology만 격리해 비교한다

### `aed88c8a93e0...` — 같은 work, 다른 timer 구조

실제 service load에서 room별 timer가 느리다고 보여도 원인이 timer 수인지 simulation, JSON serialization, socket send, DB query인지 분리하기 어렵습니다. 이 commit은 `tests/load/scheduler-benchmark.mjs`에 두 topology를 비교하는 `measure` 함수를 추가합니다.

고정 설정:

```js
const TIMESTEP_MS = 50;
const ROOM_COUNTS = [1, 20, 50, 100];
const REPEATS = Number(process.env.BENCHMARK_REPEATS ?? 3);
const DURATION_MS = Number(process.env.BENCHMARK_DURATION_MS ?? 1_500);
const WARMUP_MS = 250;
```

비교 대상은 다음과 같습니다.

```text
room strategy
  room 0 → setInterval(50ms)
  room 1 → setInterval(50ms)
  ...
  room N → setInterval(50ms)

shared strategy
  setInterval(50ms) 하나
      → 모든 room의 simulateRoomStep(room) 순회
```

두 전략은 동일한 `simulateRoomStep` 합성 CPU work를 수행합니다. timer callback이 실행된 실제 시각과 예상 시각의 차이를 lag sample로 모아 p95/p99를 계산합니다. 250ms warmup 전 sample은 버리고, duration 종료 시 생성한 timer를 모두 clear합니다.

이 exact commit에는 측정 함수와 helper만 있고 전체 room/strategy 반복과 report 출력은 아직 없습니다. 따라서 “benchmark result를 냈다”가 아니라 **공정하게 비교할 측정 경계를 만들었다**고 설명해야 합니다.

또한 이 workload는 실제 PongSimulation, socket, serialization을 포함하지 않습니다. scheduler topology라는 변수만 격리한 microbenchmark입니다.

### `8d24b5e70837...` — 숫자에 측정 조건과 decision rule을 붙인다

다음 commit이 room count×strategy×repeat loop를 실행하고, 각 run의 p95/p99 median을 계산합니다.

```js
for (const roomCount of ROOM_COUNTS) {
  for (const strategy of ["room", "shared"]) {
    const runs = [];
    for (let repeat = 0; repeat < REPEATS; repeat += 1) {
      runs.push(await measure(strategy, roomCount));
    }
    // p95/p99의 median을 report에 기록
  }
}
```

JSON report에는 결과뿐 아니라 해석에 필요한 조건이 포함됩니다.

- Node version
- OS platform/release
- CPU model/count
- total memory와 시작 시 free memory
- timestep/duration/warmup/repeats/room counts
- strategy별 sample count와 p95/p99

선택 규칙은 50-room 비교에 명시됩니다.

```js
const thresholdMs = room50.p95LagMs * 1.05;
const selectedStrategy =
  shared50.p95LagMs <= thresholdMs ? "shared" : "room";
```

즉 shared timer가 room-per-timer보다 p95 기준 5% 이상 나쁘지 않으면 shared를 선택합니다.

환경 metadata를 기록한다고 결과가 보편적 capacity 수치가 되지는 않습니다. 이는 같은 report를 재현·비교할 단서를 제공할 뿐, 다른 machine에서도 같은 p95를 보장하지 않습니다. 5% 기준도 사용자 latency SLO가 아니라 topology 선택을 위한 repository decision rule입니다.

## 2. load 도구와 load 계약은 서로 다른 artifact다

### `7b0b5f086b41...` — 실제 service를 흔드는 도구

이 commit은 세 부분을 함께 추가합니다.

1. `docker-compose.load.yml`의 Toxiproxy overlay
2. load 규모·threshold를 만드는 `load-profile.mjs`
3. k6 WebSocket scenario와 proxy control script

#### 기본 load profile

```text
connections: 500
rooms: 50
player connections: 100 (= rooms × 2)
minimum successful connections: 495
optional extended profile: 1,000 connections
max duration: 4m
```

`CONNECTIONS < ROOMS × 2`는 invalid profile로 거부됩니다. 각 room에 두 player를 배치할 최소 수조차 없으면 active room 목표가 논리적으로 달성 불가능하기 때문입니다.

threshold는 executable k6 option으로 정의됩니다.

| metric | threshold |
| --- | --- |
| connection success | `rate >= 0.99` |
| reconnect success | `rate >= 0.99` |
| snapshot delay | p95 ≤ 150ms, p99 ≤ 250ms |
| normal snapshot drop | rate < 1% |
| finalization failures | 0 |
| finalization duplicates | 0 |
| finalization results | room 수 이상 |
| online connections | 최소 성공 연결 이상 |
| active rooms | 설정 room 수 이상 |

#### k6 VU가 통과하는 실제 경계

각 VU는 HTTP dev login으로 cookie를 받고 one-time WebSocket ticket을 발급합니다. socket URL은 `?ticket=...&v=1`입니다.

player VU는 다음 lifecycle을 수행합니다.

```text
login(cookie)
  → ticket 발급
  → initial socket
  → presence가 최소 online에 도달하면 queue.join
  → queue.matched
  → game.ready
  → 100ms마다 inputSeq 증가 + game.input
  → 일정 시간 뒤 initial socket close
  → 새 ticket 발급
  → reconnect socket이 같은 room을 복구하는지 확인
```

snapshot에서는 두 종류를 측정합니다.

```text
snapshot delay = client Date.now() - snapshot.serverTimeMs
sequence gap   = current.sequence - previous.sequence - 1
```

`game.finished`는 left side에서 한 번만 집계하며 다음 조건을 요구합니다.

- `persisted === true`
- 비어 있지 않은 `matchId`
- result room ID가 VU가 관찰한 room과 동일
- 같은 VU에서 match/room result가 중복되지 않음

#### DB fault와 edge fault 분리

Compose overlay는 PostgreSQL traffic과 public edge traffic을 별도 proxy로 보냅니다.

```text
API DATABASE_URL → toxiproxy:15432 → db:5432
client edge      → toxiproxy:18080 → caddy:8080
```

control script는 DB latency/down/up과 edge latency/reset/down/up을 별도 command로 표현합니다. 따라서 readiness 503이 DB fault 때문인지 edge connection reset 때문인지 scenario 단계에서 구분할 수 있습니다.

이 commit은 **실행 능력**을 제공합니다. repository 안에 실제 500 connection run 결과나 threshold 통과 report를 저장하지는 않습니다.

### `ff1bffcd5296...` — harness 자체의 drift를 검출한다

load test가 실행되기 전에도 profile이나 source가 조용히 바뀔 수 있습니다. 이 commit의 Node test는 actual 500 connection을 열지 않고 다음 contract를 검사합니다.

- default/extended VU 수와 minimum success 계산
- connection·reconnect·snapshot·finalization threshold의 정확한 값
- invalid `connections < rooms × 2` 거부
- k6 source에 필요한 metric constructor가 실제로 존재하는지
- login→ticket→queue.ready/inputSeq/serverTimeMs 경로가 source에 있는지
- Toxiproxy에 postgres와 edge proxy가 따로 있는지
- Compose의 API DB URL과 exposed edge가 proxy를 통과하는지

이 test는 “SLO를 달성했다”가 아니라 **무엇을 SLO로 실행할 것인지가 바뀌지 않았다**는 것을 증명합니다.

```text
load-harness.test 통과
    ≠ 500 connection SLO 통과

load-harness.test 통과
    = profile/metric/wiring contract가 source상 유지됨
```

## 3. `ad482c200cea...` — simulation을 느리게 하지 않고 delivery burst를 줄인다

### 부하에서 드러난 문제

room마다 20Hz simulation tick 직후 snapshot을 broadcast하면 여러 room이 같은 shared tick에서 serialization/send를 몰아서 수행할 수 있습니다. 병목은 authoritative mechanics보다 **동시에 발생하는 delivery burst**일 수 있습니다.

단순한 해결은 simulation을 10Hz로 낮추는 것입니다. 하지만 그러면 collision, input 반응, AI, score progression의 시간 의미가 바뀝니다.

### 수정된 cadence

production code는 50ms simulation timestep을 그대로 유지합니다.

```ts
const SIMULATION_TIMESTEP_MS = DEFAULT_TIMESTEP_MS; // 50ms, 20Hz
const SNAPSHOT_DELIVERY_DIVISOR = 2;                // 10Hz delivery
```

새 room은 0 또는 1의 delivery slot을 round-robin으로 받습니다.

```ts
const snapshotDeliverySlot = this.nextSnapshotDeliverySlot;
this.nextSnapshotDeliverySlot =
  (this.nextSnapshotDeliverySlot + 1) % SNAPSHOT_DELIVERY_DIVISOR;
```

매 50ms마다 simulation step과 `syncSnapshot(room)`은 계속 실행하지만 broadcast는 다음 조건에서만 합니다.

```ts
if (
  (room.simulation.tick + room.snapshotDeliverySlot)
    % SNAPSHOT_DELIVERY_DIVISOR === 0
) {
  this.broadcastSnapshot(room);
}
```

두 room의 전송은 교대로 분산됩니다.

```text
tick 0: slot 0 rooms broadcast
        slot 1 rooms skip

tick 1: slot 0 rooms skip
        slot 1 rooms broadcast
```

결과:

- authoritative simulation: 20Hz 유지
- snapshot projection sync: 매 tick 유지
- client broadcast: room당 10Hz
- 여러 room의 broadcast burst: 두 slot으로 분산

같은 diff는 finalization observer에 `created` metadata를 전달하고, database success인데 `created === false`이면 duplicate metric을 증가시킵니다. 이는 load scenario가 idempotent 기존 결과 반환을 별도로 관찰하기 위한 보강입니다.

이 fix가 보장하지 않는 것은 분명합니다. 모든 hardware에서 p95 150ms를 자동 달성한다는 증거가 아니며, 10Hz가 모든 UX에 최적이라는 일반 결론도 아닙니다. load가 드러낸 pressure source를 simulation correctness와 분리해 조절한 production decision입니다.

## 4. 수동 fault injection을 state machine으로 바꾼다

### `84bec3bf57ae...` — bounded recovery scenario

Toxiproxy command를 사람이 순서대로 실행하면 다음이 재현되지 않습니다.

- fault 적용 전 baseline 상태
- readiness가 기대 상태에 도달할 때까지의 polling
- timeout
- 중간 실패 뒤 proxy reset
- 결과 report 형식

`fault-scenario.mjs`는 설정을 먼저 검증합니다.

```text
Toxiproxy control: http://127.0.0.1:8474
API readiness:     http://127.0.0.1:14000/health/ready
edge readiness:    http://127.0.0.1:18080/api/health/ready
DB latency:        300ms
edge latency:      150ms
request timeout:   5s
recovery timeout:  15s
poll interval:     250ms
```

세 URL은 HTTP loopback만 허용합니다. remote hostname, HTTPS URL, 문서용 외부 IP는 command를 하나도 실행하기 전에 거부됩니다. fault harness가 실수로 staging/production proxy를 조작하지 않게 하는 안전 경계입니다.

#### scenario 순서

```text
reset all
  → baseline: API ready / database up
  → db-latency 300ms: 여전히 ready
  → db-down: HTTP 503 + not_ready + database down
  → db-up: ready까지 polling
  → edge-latency 150ms: edge ready
  → edge-reset: network error 또는 5xx 관찰
  → edge-up: ready까지 polling
  → reset all (항상)
```

각 step은 status, duration, parsed body 또는 network error를 report에 저장합니다. 성공 report는 `schemaVersion: 1`, 시작/종료 시각, target, settings, ordered steps를 가집니다.

cleanup은 정상 경로의 마지막 line에만 있지 않습니다. scenario error를 보존한 뒤 별도 block에서 `reset`을 시도합니다. cleanup까지 실패하면 원래 error를 유지하면서 cleanup error를 cause로 붙입니다.

```text
scenario success + reset success → report
scenario failure + reset success → original failure
scenario failure + reset failure → original failure(cause=cleanup failure)
```

### `335565908920...` — 실제 proxy 없이 orchestrator를 검증한다

외부 Toxiproxy/PostgreSQL을 띄워야만 runner logic을 테스트할 수 있다면 command ordering bug를 빠르게 재현하기 어렵습니다. runner는 다음 dependency를 주입할 수 있게 설계되어 있습니다.

- `applyToxiproxyCommand`
- `probeReadiness`
- `sleep`
- `now`

test는 fake probe sequence를 만들고 exact command 배열을 비교합니다.

```text
reset
→ db-latency 300 0
→ db-down
→ db-up
→ edge-latency 150 0
→ edge-reset 0
→ edge-up
→ reset
```

DB recovery 전에 두 번의 not-ready probe가 나오도록 해 sleep 배열도 `[250, 250]`인지 확인합니다. report에는 고정 timestamp, 318ms DB latency observation, DB down body, `socket reset` error가 보존되어야 합니다.

별도 test는 `db-down` control command가 throw하도록 만들고, 그 뒤에도 마지막 `reset`이 호출되는지 확인합니다. non-loopback URL은 command array가 생기기 전에 거부됩니다.

이 regression이 증명하는 것은 orchestration·report·cleanup입니다. 실제 Toxiproxy가 packet을 지연시키고 실제 API가 15초 안에 회복하는지는 operational run이 별도로 증명해야 합니다.

## 증거 유형과 해석

| artifact | 실제 실행 시 포함하는 범위 | repository에서 고정하는 것 | 이번 작성에서 없는 것 |
| --- | --- | --- | --- |
| scheduler benchmark | Node timer + synthetic CPU | 동일 work, p95/p99, environment report, 5% rule | 새 측정 JSON 결과 |
| load profile contract test | Node source/config | 500/50, SLO metric/threshold, proxy wiring | 실제 VU 연결 |
| k6 scenario | 실행 API/WS/DB/edge | cookie-ticket-v1 lifecycle와 metric 수집 코드 | 실제 k6 summary |
| cadence fix | production GameHub | 20Hz state / 10Hz staggered delivery | 모든 환경의 SLO 달성 |
| fault runner unit test | injected command/probe/clock | ordered state machine, loopback guard, cleanup/report | 실제 network/DB fault |
| Toxiproxy operational run | proxy+API+DB+edge | 실제 degradation/recovery | 이번 환경의 실행 report |

## 최종 실행 관계

```text
[microbenchmark]
  room timers vs shared timer
  → environment-tagged decision report

[load harness]
  500 VUs / 50 rooms
  → cookie→ticket→v1 socket
  → reconnect / snapshot / finalization metrics
  → Toxiproxy DB·edge fault capability

[production correction]
  simulation 20Hz 유지
  snapshot 10Hz + room slot staggering

[fault scenario]
  baseline→degrade→outage→recovery
  → loopback-only target
  → bounded polling
  → schemaVersion 1 JSON
  → success/failure 모두 reset
```

## 이 Thread의 경계

- snapshot buffer가 callback 지연과 실제 `bufferedAmount`를 구분하는 문제는 Thread 03입니다.
- finalization idempotency의 실제 PostgreSQL row 의미는 Thread 05입니다.
- GameHub가 persistence failure를 retry하며 room을 유지하는 lifecycle은 Thread 04입니다.
- process/browser 기능 smoke는 Thread 06입니다.
- CI가 load/fault job을 어떤 조건에서 실행하는지는 이 category 밖입니다.

## 조사·실행 기록

각 commit의 exact diff와 해당 시점 source/config/test를 확인했습니다. scheduler benchmark, k6, Docker Compose, Toxiproxy fault scenario는 이 작성 환경에서 실행하지 않았습니다. 따라서 문서에 적힌 p95/p99·99%·500 connection·300ms/150ms 값은 source에 정의된 **측정 설정과 통과 기준**이며, 새 실행 결과가 아닙니다.
