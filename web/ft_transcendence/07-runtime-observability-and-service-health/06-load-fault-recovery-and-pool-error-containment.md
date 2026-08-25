# Thread: 부하·fault recovery 계약과 pool error containment

Project: `ft_transcendence` · 영역: load / fault recovery / database pool boundary

## 개요

이 Thread에는 서로 다른 두 운영 문제가 함께 있다.

첫째, “많은 WebSocket과 active room 아래에서도 서비스가 정상인가”를 감으로 판단하지 않고, connection·reconnect·snapshot·finalization·readiness recovery에 대한 실패 기준을 실행 가능한 profile과 report로 만든다. PostgreSQL 경로와 public edge 경로를 분리해 latency, down, reset을 독립적으로 주입한다.

둘째, request query와 무관하게 발생하는 PostgreSQL idle-client pool error가 `EventEmitter` 경계 밖으로 탈출해 process를 종료시키거나, connection string과 error message를 그대로 log에 노출하지 못하게 한다.

두 문제는 하나의 선형 기능이 아니다. load/fault harness는 **외부에서 서비스 상태 전이를 관찰하는 도구**이고, pool error handler는 **process 내부의 비동기 error boundary**다. 공통점은 failure를 정상 흐름 밖의 우연한 사건으로 두지 않고, 제한된 입력·관찰·복구 결과를 가진 운영 계약으로 만든다는 점이다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ff1bffcd5296` | test(load): 실시간 부하 임계값 정의 | B | `REALTIME, PERSISTENCE, OPERATIONS` | 기본/확장 profile과 service-level threshold를 source contract로 고정 |
| 2 | `7b0b5f086b41` | test(load): 실시간 fault injection 도구 추가 | A | `AUTH, REALTIME, PERSISTENCE` | k6 scenario, Toxiproxy overlay/control, load profile 구현 |
| 3 | `547d9943d30a` | fix(load): 기본 부하 profile 측정 안정화 | A | `REALTIME, OPERATIONS, OBSERVABILITY` | reconnect timing을 실제 playing 이후로 이동하고 authoritative metrics로 finalization 판정 |
| 4 | `84bec3bf57ae` | test(load): fault recovery 검사 자동화 | A | `PERSISTENCE, OPERATIONS, PERF` | DB/edge failure와 recovery를 polling하고 JSON report로 남기는 scenario 추가 |
| 5 | `335565908920` | test(load): fault scenario 설정과 report 검증 | B | `PERSISTENCE, OPERATIONS, PERF` | loopback 제한, step 순서, report, cleanup reset을 deterministic test로 고정 |
| 6 | `eca21f115c1b` | fix(db): idle connection pool 오류에서 복구 | A | `PERSISTENCE, RISK` | pool `error` listener와 sanitized reporter boundary 설치 |
| 7 | `493babe1cf30` | test(db): 안전한 connection pool 오류 처리 검증 | B | `PERSISTENCE, TEST` | secret 비노출, reporter 없음/실패에서도 error event가 탈출하지 않음을 검증 |

## 1. 부하 테스트의 먼저 할 일은 숫자를 만드는 것이 아니라 실패 기준을 고정하는 것이다

### `ff1bffcd5296` — 구현보다 먼저 놓인 default profile 계약

이 SHA는 test-first 단계다. `load-harness.test.mjs`가 `load-profile.mjs`, `toxiproxy-control.mjs`, `pong-load.js`에 요구할 계약을 먼저 추가하며, 실제 harness 구현은 다음 `7b0b5f086b41`에 들어온다. 따라서 exact SHA만으로 완성된 load run을 실행할 수 있다고 해석하면 안 된다.

테스트가 요구하는 기본 profile은 500 WebSocket connection을 시도하고 그중 100개 connection으로 50개 room을 만드는 것이다. 최소 성공 connection은 495개다.

| 관찰 항목 | threshold |
| --- | --- |
| connection success | `rate >= 0.99` |
| reconnect success | `rate >= 0.99` |
| snapshot delay | `p95 <= 150ms`, `p99 <= 250ms` |
| normal snapshot drop | `rate < 0.01` |
| persisted finalization failure | `count == 0` |
| duplicate finalization | `count == 0` |
| successful finalization | `count >= room count` |
| observed online connections | `max >= 495` |
| observed active rooms | `max >= 50` |

테스트가 고정하는 k6 executor contract은 `per-vu-iterations`, 500 VU, VU당 1회, 최대 4분이다. `EXTENDED_LOAD=1` 또는 명시적 `CONNECTIONS=1000`으로 1,000 connection profile을 선택할 수 있지만, 1,000을 기본값으로 숨기지 않는다. `connections >= rooms * 2`도 검증해 room 수보다 player connection이 부족한 profile을 거부한다.

이 commit의 tests는 threshold object와 scenario source가 다음 protocol path를 포함하는지 검사한다.

```text
POST /auth/dev-login
  → POST /auth/ws-ticket
  → WebSocket connect
  → queue.join
  → game.ready
  → increasing inputSeq 전송
  → snapshot.serverTimeMs로 delay 계산
  → reconnect와 persisted finalization 관찰
```

이는 profile과 source wiring에 대한 정적 계약이다. 500 connection을 실제로 열거나 threshold를 통과한 결과를 repository에 기록한 commit은 아니다.

## 2. `7b0b5f086b41` — 실제 service path와 fault injection topology를 만든다

### k6 scenario가 통과하는 production boundary

`pong-load.js`는 각 VU가 development login을 수행하고 one-time WebSocket ticket을 발급받은 뒤 실제 `/ws` endpoint에 연결하도록 한다. 첫 `rooms * 2` VU만 player로 사용하고, 나머지는 connection pressure를 만든다.

player connection은 match가 잡히면 ready를 보내고 100ms마다 sequence가 증가하는 input을 전송한다. snapshot에서는 server timestamp와 client `Date.now()`의 차이를 delay로 기록하고, sequence gap을 normal drop sample로 만든다.

```js
if (event.type === "game.snapshot") {
  const snapshot = event.snapshot;
  if (expectedRoomId !== null && snapshot.roomId === expectedRoomId) {
    result.recovered = true;
  }
  snapshotDelay.add(Math.max(0, Date.now() - snapshot.serverTimeMs));
  if (lastSequence !== null && snapshot.sequence > lastSequence) {
    const missed = snapshot.sequence - lastSequence - 1;
    for (let index = 0; index < missed; index += 1) {
      normalSnapshotDropRate.add(true);
    }
    normalSnapshotDropRate.add(false);
  } else if (lastSequence === null) {
    normalSnapshotDropRate.add(false);
  }
  if (lastSequence === null || snapshot.sequence > lastSequence) {
    lastSequence = snapshot.sequence;
  }
  return;
}
```

초기 구현은 `queue.matched` 직후 reconnect용 close timer를 걸고, `game.finished` event를 받은 VU가 match/room ID를 local set에 넣어 finalization success와 duplicate를 세었다. 이 두 측정 방식은 후속 `547d9943d30a`에서 교정된다.

### PostgreSQL과 edge failure를 같은 proxy로 섞지 않는다

load compose overlay는 두 개의 Toxiproxy proxy를 정의한다.

| proxy | listen | upstream | 목적 |
| --- | --- | --- | --- |
| `postgres` | `0.0.0.0:15432` | `db:5432` | API→database 경로 latency/down |
| `edge` | `0.0.0.0:18080` | `caddy:8080` | client→public edge latency/reset/down |

API의 `DATABASE_URL`은 `toxiproxy:15432`를 거치며, public edge proxy port와 Toxiproxy control port는 host loopback에만 publish된다. bootstrap container가 proxy definition을 만든 뒤에만 API가 시작된다.

control tool은 command를 topology와 맞춘다.

```text
db-latency / db-down / db-up       → postgres proxy
edge-latency / edge-reset
edge-down / edge-up                 → edge proxy
reset                               → 모든 toxic 제거 및 proxy enable
```

DB latency와 edge reset은 서비스에 전혀 다른 결과를 낸다. DB down은 API process가 살아 있어도 readiness의 database check를 down으로 만들어야 하고, edge reset은 public route 자체가 network failure 또는 5xx로 관찰될 수 있다. 분리된 proxy가 있어야 어느 boundary가 실패했는지 report에서 구별할 수 있다.

## 3. `547d9943d30a` — 부하를 만드는 시점과 성공을 판정하는 출처를 바로잡는다

### Reconnect는 match allocation이 아니라 실제 playing 이후에 시작해야 한다

초기 scenario는 `queue.matched`를 받자마자 reconnect close timer를 시작했다. 그러나 matched는 양쪽이 ready해 simulation이 playing에 들어갔다는 뜻이 아니다. 많은 VU가 같은 시점에 matched되면 아직 경기가 시작되기도 전에 connection이 닫히고, reconnect test가 admission/matchmaking timing을 측정하게 된다.

수정 후 first playing snapshot을 확인한 뒤에만 close를 예약한다.

```js
if (
  !reconnectCloseArmed
  && phase === "initial"
  && player
  && snapshot.state.phase === "playing"
) {
  reconnectCloseArmed = true;
  socket.setTimeout(() => socket.close(), reconnectDelayMs);
}
```

기본 close delay는 10초에서 2초로 줄지만, 100명 player가 한 번에 끊기지 않도록 5초 stagger window를 둔다. player index에 비례한 offset을 더해 reconnect pressure를 분산한다. focused run에서는 `PLAYER_RECONNECT_STAGGER_MS=0`으로 같은 시점 close를 명시적으로 선택할 수 있다.

### Finalization은 VU가 받은 event가 아니라 server-side aggregate로 판정한다

한 match의 양쪽 connection 또는 reconnect connection이 같은 `game.finished`를 볼 수 있다. 각 VU local set은 VU 내부 중복만 막을 뿐 service 전체에서 persisted result가 한 번 만들어졌는지 증명하지 못한다.

수정은 teardown에서 Prometheus를 읽는다.

```text
pong_pong_api_match_finalizations_total
  {persistence="database", outcome="success"}

pong_pong_api_match_finalizations_total
  {persistence="database", outcome="failure"}

pong_pong_api_match_finalization_duplicates_total
```

label-aware `readPrometheusSample()`가 exact metric name과 expected labels를 찾아 숫자로 변환한다. k6 event path는 malformed/incorrect `game.finished`를 failure로 기록할 수 있지만, threshold의 전체 성공·실패·중복 count는 server가 실제 persistence 경계에서 기록한 aggregate를 source of truth로 사용한다.

이 변경은 load를 줄인 것이 아니라 측정 대상과 측정 시점을 domain state에 맞춘 것이다.

## 4. `84bec3bf57ae` — failure를 넣고 복구를 기다리는 전체 scenario

`load:faults` command는 다음 설정으로 loopback service만 조작한다.

| 설정 | 기본값 |
| --- | ---: |
| database latency | 300ms |
| edge latency | 150ms |
| readiness request timeout | 5초 |
| recovery deadline | 15초 |
| poll interval | 250ms |
| edge steps 포함 | true |

Toxiproxy control URL, direct API readiness URL, edge readiness URL은 모두 `http:` loopback URL이어야 한다. 외부 hostname, HTTPS remote target, 문서용 public IP는 config 단계에서 거부한다. 이는 자동 fault command가 실수로 원격 환경을 변경하는 범위를 제한한다.

### Scenario step과 expected state

```text
reset
  → baseline: direct API readiness = 200/ready/database up

db-latency 300ms
  → database_latency: 여전히 ready

db-down
  → database_down: 503/not_ready/database down

db-up
  → database_recovery: 다시 200/ready/database up

edge-latency 150ms
  → edge_latency: public edge를 통해 ready

edge-reset
  → edge_reset: network failure 또는 5xx

edge-up
  → edge_recovery: public edge를 통해 다시 ready

finally reset
```

각 step은 한 번의 순간값으로 판정하지 않는다. expected predicate를 만족할 때까지 250ms 간격으로 polling하고 15초 recovery deadline을 넘으면 마지막 observation을 포함한 error를 낸다.

report에는 schema version, 시작/종료 시각, target, settings, step별 status/duration/body 또는 network error가 들어간다. CLI 실패도 JSON `{ passed:false, error:{message} }`를 stdout에 남기고 exit code 1을 설정한다.

### Cleanup도 scenario 결과의 일부다

main sequence에서 command나 probe가 실패해도 마지막 `reset`을 별도로 시도한다. cleanup reset도 실패하면 원래 error를 보존하고 cleanup error를 cause로 연결한다. fault가 다음 실행에 남아 baseline을 오염시키는 것을 줄이는 구조다.

다만 process가 강제 kill되거나 host가 중단되는 상황까지 reset을 보장하지는 않는다. 그런 경우 운영자가 control tool의 `reset` command를 별도로 실행해야 한다.

## 5. `335565908920` — 자동화 자체를 deterministic하게 검증한다

이 commit은 실제 Toxiproxy container와 API를 사용하지 않고 dependency를 주입한다.

- config default와 `FAULT_INCLUDE_EDGE=0`
- non-loopback target 거부
- command 순서와 각 argument
- readiness observation sequence와 250ms polling
- schema version, timestamps, targets, settings, step names
- DB down body와 edge reset network error 보존
- report JSON round trip
- 중간 command 실패 뒤에도 마지막 `reset` 실행

mock observation은 database latency 318ms, DB down 503, edge reset `socket reset` 같은 값을 제공하고 report가 그대로 보존하는지 확인한다. 따라서 이 test가 증명하는 것은 scenario controller와 report 형식이다. 실제 PostgreSQL이 300ms latency 뒤 ready를 유지하거나, down/up 뒤 15초 안에 회복했다는 통합 증거는 아니다.

## 6. `eca21f115c1b` — request 밖의 idle pool error를 process boundary 안에 가둔다

### 왜 query의 `try/catch`로는 잡을 수 없는가

PostgreSQL pool의 idle client connection이 끊어지면 현재 repository method의 awaited query가 아니라 pool의 `error` event로 전달될 수 있다. 이 경로는 request handler의 `try/catch`와 분리돼 있다. listener가 없거나 listener/reporting code가 다시 throw하면 process-level failure가 될 수 있다.

`createPostgresRepository()`는 pool을 만든 직후 handler를 설치한다.

```ts
export function createPostgresRepository(
  databaseUrl: string,
  options: PostgresRepositoryOptions = {}
): AppRepository {
  const pool = new Pool({ connectionString: databaseUrl });
  installPostgresPoolErrorHandler(pool, options.onPoolError);
  const db = new Kysely<Database>({
    dialect: new PostgresDialect({ pool })
  });
  return new PostgresRepository(db, pool);
}
```

### Raw error 대신 bounded event만 reporter에 넘긴다

외부로 공개하는 event는 세 field뿐이다.

```ts
export interface PostgresPoolErrorEvent {
  kind: "idle_client_error";
  errorName: string;
  errorCode: string | null;
}
```

`error.name`과 `error.code`는 `[A-Za-z0-9_]` 문자 1~64개만 허용한다. 그 외 값은 `UnknownError` 또는 null로 바꾼다. raw message, stack, connection string, host, user, password는 event에 들어가지 않는다.

```ts
pool.on("error", (error) => {
  let event = FALLBACK_EVENT;
  try {
    event = toSafePoolErrorEvent(error);
  } catch {
    // A malformed error object must not escape the pool's EventEmitter boundary.
  }

  try {
    onPoolError?.(event);
  } catch {
    // Reporting is best-effort and must not turn an idle client failure into a process crash.
  }
});
```

여기서 복구의 의미는 failed idle connection 하나를 애플리케이션이 직접 재사용한다는 뜻이 아니다. pool implementation이 이후 connection lifecycle을 관리할 수 있도록, 비동기 error event가 unhandled exception으로 process를 종료시키지 않는다는 뜻이다. database availability는 계속 readiness/query 결과가 판단한다.

### Logger가 만들어지기 전 발생한 error도 잃지 않는다

repository는 `buildApp()`보다 먼저 생성된다. app logger를 callback에 직접 capture하려 하면 logger가 아직 없다. entry point는 잠시 `earlyPoolErrors` 배열에 safe event를 모으고, app 생성 뒤 reporter function을 logger로 교체한 다음 backlog를 flush한다.

```ts
const earlyPoolErrors: PostgresPoolErrorEvent[] = [];
let reportPoolError = (event: PostgresPoolErrorEvent) => {
  earlyPoolErrors.push(event);
};
const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl, {
      onPoolError: (event) => {
        reportPoolError(event);
      }
    })
  : createMemoryRepository();

const app = buildApp({
  repo,
  webOrigin: env.webOrigin,
  appMode: env.appMode,
  sessionSecret: env.sessionSecret,
  trustProxy: env.trustProxy
});
reportPoolError = (event) => {
  app.log.error(event, "PostgreSQL idle client connection failed");
};
for (const event of earlyPoolErrors.splice(0)) {
  reportPoolError(event);
}
```

이렇게 repository creation과 logger availability 사이의 짧은 bootstrap window도 같은 sanitized event format으로 수렴한다.

## 7. `493babe1cf30` — containment의 정확한 증명 범위

테스트는 실제 `pg.Pool` 객체에 handler를 설치하고 `pool.emit("error", error)`를 호출한다.

첫 error에는 message와 별도 `connectionString` property에 password가 포함되며 code는 `57P01`이다. assertion은 reporter가 다음 값만 받는지 확인한다.

```json
{
  "kind": "idle_client_error",
  "errorName": "Error",
  "errorCode": "57P01"
}
```

serialized reporter calls에는 `secret`과 원래 connection error message가 없어야 한다. 추가 case는 reporter가 없을 때와 reporter가 직접 throw할 때 모두 `pool.emit()`이 밖으로 throw하지 않는지 확인한다.

이 테스트는 실제 PostgreSQL server를 중단해 pool이 새 connection을 확보하고 query가 회복되는지를 확인하지 않는다. 또한 log sink 실패, 수천 개 error burst의 buffer 정책, readiness와 pool event의 시간 관계도 다루지 않는다. 검증 대상은 **event boundary containment와 payload 최소화**다.

## Failure → observation → recovery matrix

| 주입/사건 | 관찰 지점 | 기대 상태 | 회복 판정 |
| --- | --- | --- | --- |
| 500 connection / 50 room workload | k6 custom metrics + Prometheus | connection/reconnect·delay·drop·finalization threshold | run summary가 모든 threshold 만족 |
| PostgreSQL 300ms latency | direct API readiness | 200, database up | latency 상태에서도 ready predicate 도달 |
| PostgreSQL down | direct API readiness | 503, database down | DB up 뒤 200/ready 재도달 |
| public edge latency | edge readiness URL | 200/ready | latency 상태에서 predicate 도달 |
| public edge reset | edge readiness URL | network error 또는 5xx | edge up 뒤 200/ready 재도달 |
| idle pool client error | `pool.on("error")` | sanitized event만 report, throw 없음 | process가 계속 살아 readiness/query가 후속 상태 판단 |
| fault controller 중간 실패 | controller error path | 원래 error 보존 | 모든 proxy reset 재시도 |

## 최종 상태와 한계

이 Thread가 만든 최종 운영 계약은 다음과 같다.

- 부하 profile은 connection 수만 세지 않고 room, reconnect, snapshot freshness, persisted finalization까지 함께 판정한다.
- reconnect disruption은 match allocation이 아니라 playing state가 관찰된 뒤 시작하고, 기본 실행에서는 player별로 분산한다.
- finalization success/duplicate는 client event 수가 아니라 server persistence metric을 authoritative source로 사용한다.
- database와 public edge fault는 서로 다른 proxy와 readiness URL로 관찰한다.
- fault automation은 loopback target만 허용하며 각 step 뒤 expected state를 polling하고 machine-readable report를 남긴다.
- idle pool error의 raw object는 logger/reporter boundary로 나가지 않으며 reporter가 실패해도 process exception으로 변하지 않는다.
- pool error containment와 database readiness는 역할이 다르다. 전자는 process 생존, 후자는 traffic admission을 결정한다.

이 source가 제공하지 않는 증거도 명확하다.

- repository에는 실제 500/1,000 connection k6 실행 결과나 threshold 통과 report가 없다.
- fault scenario unit test는 mocked command/probe를 사용하며 실제 Toxiproxy·PostgreSQL·Caddy recovery 실행 결과가 아니다.
- pool test는 error event containment을 검증하지만 실제 network outage 뒤 query recovery를 증명하지 않는다.
- host crash, forced kill, Toxiproxy control plane 자체의 장기 장애, multi-instance traffic failover는 범위 밖이다.

startup/readiness의 의미는 `01-startup-liveness-readiness-and-storage-state.md`, bounded metrics는 `02-metrics-observer-boundaries-and-cardinality.md`, snapshot cadence와 shared scheduling은 `04-gamehub-runtime-integration-shared-scheduling-and-congestion.md`, draining lifecycle은 `05-draining-readiness-and-graceful-shutdown.md`에서 각각 다룬다.

> 검토 범위: 표시된 exact SHA의 diff와 해당 시점 source/test를 조사했습니다. k6 load, Toxiproxy fault scenario, 실제 PostgreSQL outage 또는 전체 test suite는 실행하지 않았습니다.
