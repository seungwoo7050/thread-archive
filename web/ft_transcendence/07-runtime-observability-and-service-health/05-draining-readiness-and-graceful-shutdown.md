# Thread: 트래픽 수용을 먼저 멈추고 process를 나중에 닫는다

Project: `ft_transcendence` · 영역: draining readiness / graceful shutdown

## 개요

실시간 경기 서버의 종료는 `SIGTERM`을 받자마자 process를 닫는 문제가 아니다. 종료 신호를 받은 뒤에도 새 match를 계속 만들면 drain은 끝나지 않고, readiness를 계속 200으로 유지하면 load balancer가 종료 중인 instance로 새 요청을 보낸다. 반대로 active room을 즉시 끊으면 배포가 정상 경기 종료를 파괴한다.

이 Thread는 종료를 다음 순서의 상태 전이로 만든다.

```text
SIGTERM 또는 SIGINT
  → 첫 신호만 shutdown 시작
  → readiness를 즉시 not_ready / draining으로 전환
  → 대기열과 tournament waiter를 비우고 새 match 시작 거부
  → 이미 진행 중인 room이 끝나기를 기다림
  → room이 0개가 되거나 60초 timeout
  → app.close()로 WebSocket·timer·metrics·repository 정리
  → container runtime은 최소 60초보다 긴 종료 유예를 제공
```

핵심은 **transport를 즉시 모두 끊는 것**이 아니라 **새 작업의 admission을 먼저 닫고 이미 소유한 작업을 bounded하게 완료하는 것**이다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `44ef3e07e1a5` | feat(game): 새 작업 차단과 active room drain 추가 | A | `PROTOCOL, REALTIME, TOURNAMENT` | readiness와 GameHub admission을 draining state로 묶고 active-room waiter 추가 |
| 2 | `1c9981393973` | feat(ops): graceful shutdown 절차 추가 | A | `REALTIME, PERSISTENCE, OPERATIONS` | SIGTERM/SIGINT를 one-shot drain→close 절차에 연결 |
| 3 | `9d05f47e7f4b` | test(ops): GameHub drain과 graceful shutdown 검증 | A | `REALTIME, OPERATIONS, RISK` | admission 차단, timeout, readiness 전환, signal 중복 억제 검증 |
| 4 | `312ddbc6fbe2` | fix(runtime): container 종료 유예를 room drain과 정렬 | A | `REALTIME, OPERATIONS, RISK` | application의 60초 drain보다 긴 70초 container grace 설정 |
| 5 | `73ba979841cd` | test(docker): API 종료 유예 계약 검증 | B | `OPERATIONS, TEST` | production compose의 API 종료 유예가 최소 60초인지 정적 검증 |

## 1. `44ef3e07e1a5` — draining을 readiness와 match admission에 동시에 반영한다

### 종료 상태가 먼저 외부에 보여야 한다

`buildApp()`은 local `draining` flag를 두고 `app.beginDrain()`을 Fastify instance에 decorate한다.

```ts
let draining = false;

app.decorate("beginDrain", async (timeoutMs = 60_000) => {
  draining = true;
  return hub.beginDrain(timeoutMs);
});
```

readiness는 database만 확인하지 않는다. repository가 정상이어도 lifecycle이 draining이면 503이다.

```ts
const ready = !draining
  && repository.database === "up"
  && (
    repository.migrations === "current"
    || repository.migrations === "not_applicable"
  );
const body = parseOutput(http.readyHealthResponseSchema, {
  status: ready ? "ready" : "not_ready",
  service: "pong-pong-api",
  checks: {
    lifecycle: draining ? "draining" : "accepting",
    database: repository.database,
    migrations: repository.migrations
  }
});
```

이 순서 때문에 drain promise가 끝나기 전부터 load balancer는 해당 instance를 새 traffic 대상에서 제외할 수 있다. liveness는 process 생존 여부이므로 이 변경에서 내려가지 않는다. 종료 준비 중인 process는 살아 있지만 새 작업을 받을 준비는 되지 않은 상태다.

### GameHub가 닫는 것은 connection이 아니라 새 match의 입구다

GameHub에는 `acceptingMatches`가 추가된다. `beginDrain()`이 호출되면 이 값이 false가 되고 다음 두 entry point가 `server_draining` protocol error를 반환한다.

- 일반 queue 또는 AI match를 시작하는 `joinQueue`
- tournament match에 들어가는 `joinTournamentMatch`

즉 새 WebSocket handshake 자체를 막는 코드는 아니다. 이미 연결된 client도 presence나 error를 받을 수 있지만 **새 room을 만드는 operation**은 통과하지 못한다.

대기 중인 작업은 active room과 다르게 보존하지 않는다.

```ts
this.acceptingMatches = false;
for (const entry of this.queue.splice(0)) {
  clearQueueTimer(entry);
  this.sendDrainingError(entry.client);
}
for (const waiters of this.tournamentWaiters.values()) {
  for (const client of waiters) this.sendDrainingError(client);
}
this.tournamentWaiters.clear();
this.broadcastPresence();
```

waiting player는 아직 실행 중인 domain work를 소유하지 않는다. 따라서 queue timer를 취소하고 즉시 error를 주는 것이 종료 시간을 유한하게 만든다. 반면 `rooms`에 들어간 경기는 기존 scheduler와 reconnect/finalization lifecycle을 계속 사용한다.

### Active room 수가 drain promise의 완료 조건이다

room이 하나도 없으면 즉시 성공한다.

```ts
if (this.rooms.size === 0) {
  return Promise.resolve({ drained: true, activeRooms: 0 });
}
```

active room이 있으면 한 개의 waiter를 만들고, 반복 호출은 같은 promise를 돌려준다. timeout은 `Math.max(0, timeoutMs)` 뒤 남은 room 수를 결과로 남긴다.

```ts
const timer = setTimeout(() => {
  this.finishDrain({
    drained: false,
    activeRooms: this.rooms.size
  });
}, Math.max(0, timeoutMs));
timer.unref?.();
```

room이 abandon되거나 finished room으로 제거되는 경로는 `notifyDrainProgress()`를 호출한다. 마지막 room이 사라지는 순간 waiter가 `{ drained: true, activeRooms: 0 }`으로 resolve된다.

```ts
private notifyDrainProgress(): void {
  if (this.drainWaiter && this.rooms.size === 0) {
    this.finishDrain({ drained: true, activeRooms: 0 });
  }
}
```

따라서 종료 성공은 timer가 아니라 **domain owner인 `rooms` collection이 비었는가**로 판단한다.

### Drain과 close는 다른 책임이다

`beginDrain()`은 existing room이 자연스럽게 끝날 기회를 준다. `close()`는 최종 cleanup이다.

```text
GameHub.close()
  → acceptingMatches=false
  → queue timer와 tournament waiter 제거
  → shared room scheduler stop
  → reconnect timer 제거, room registry clear
  → retained guest-result cleanup timer 제거
  → client/clientsByUser registry clear
  → heartbeat와 snapshot buffer close
  → 열린 WebSocket terminate
```

`buildApp()`의 `onClose`가 `hub.close()`를 호출하므로 drain이 timeout으로 끝나더라도 이후 `app.close()`는 남은 runtime resource를 강제로 정리한다. 즉 timeout 결과의 `drained=false`는 “process를 무한히 기다린다”가 아니라 “자연 종료 예산을 모두 사용했으므로 final close 단계로 간다”는 뜻이다.

## 2. `1c9981393973` — signal을 one-shot drain→close orchestration으로 바꾼다

### 중복 신호는 두 번째 shutdown을 만들지 않는다

`installGracefulShutdown()`은 signal source와 shutdown callback을 분리해 test 가능한 경계로 만든다.

```ts
export function installGracefulShutdown(
  signals: SignalSource,
  shutdown: (signal: ShutdownSignal) => Promise<void>,
  onError: (error: unknown) => void
): () => void {
  let started = false;

  const start = (signal: ShutdownSignal) => {
    if (started) return;
    started = true;
    void shutdown(signal).catch(onError);
  };

  const onSigterm = () => start("SIGTERM");
  const onSigint = () => start("SIGINT");
  signals.on("SIGTERM", onSigterm);
  signals.on("SIGINT", onSigint);

  return () => {
    signals.off("SIGTERM", onSigterm);
    signals.off("SIGINT", onSigint);
  };
}
```

첫 signal이 `started=true`를 publish한 뒤에는 SIGTERM과 SIGINT가 섞여 다시 와도 새로운 drain promise나 `app.close()` 호출을 만들지 않는다. disposer는 app close 시 listener를 제거해 test process나 reload lifecycle에 handler를 남기지 않는다.

### Runtime entry point의 실제 종료 순서

`index.ts`는 다음 orchestration을 설치한다.

```ts
const disposeShutdownSignals = installGracefulShutdown(
  process,
  async (signal) => {
    app.log.info({ signal }, "graceful shutdown started");
    const result = await app.beginDrain(60_000);
    app.log.info(result, "game room drain finished");
    await app.close();
  },
  (error) => {
    app.log.error(
      { errorName: error instanceof Error ? error.name : "UnknownError" },
      "graceful shutdown failed"
    );
    process.exitCode = 1;
    void app.close().catch(() => undefined);
  }
);
```

정상 경로는 drain 결과가 true인지 false인지와 무관하게 `app.close()`로 끝난다. 실패 경로는 raw error message나 connection detail 대신 error name만 기록하고 exit code를 1로 설정한 뒤 best-effort close를 시도한다.

이 commit은 `process.exit()`로 즉시 종료하지 않는다. Node event loop가 Fastify와 repository close를 수행할 수 있도록 종료 코드를 남기는 방식이다.

## 3. `9d05f47e7f4b` — 세 종류의 경계를 분리해 검증한다

### Admission과 readiness

GameHub drain test는 queue에 한 명을 넣어 `queuedPlayers=1`을 확인한 뒤 drain을 시작한다.

- 기존 queue는 즉시 비어 `queuedPlayers=0`
- drain 시작 뒤 들어온 AI queue command는 `server_draining`
- active room이 없으므로 drain은 `{ drained: true, activeRooms: 0 }`

health test는 같은 시점에 `/health/ready`가 503이며 body가 `status=not_ready`, `checks.lifecycle=draining`인지 확인한다. repository failure를 기다린 뒤 readiness를 내리는 것이 아니라 lifecycle flag가 바뀌는 즉시 내려간다는 증거다.

### Active-room timeout

두 번째 GameHub test는 AI room 하나를 만든 뒤 60초 drain을 시작한다.

```text
59,999ms 경과 → promise 미완료
60,000ms 경과 → { drained: false, activeRooms: 1 }
```

이 test는 room이 timeout 전에 정상 완료될 때의 positive path를 직접 만들지는 않는다. 그 경로는 production의 room removal마다 `notifyDrainProgress()`를 호출하는 코드로 연결된다. test가 직접 증명하는 것은 active room이 있어도 timeout을 넘겨 무기한 기다리지 않는다는 점이다.

### Signal one-shot과 failure propagation

fake `EventEmitter`에 SIGTERM→SIGINT→SIGTERM을 연속 emit해 shutdown callback이 첫 SIGTERM으로 정확히 한 번만 호출되는지 확인한다. rejected shutdown은 `onError`에 전달되며 그 뒤 SIGINT가 와도 재시작되지 않는다.

이 commit은 실제 OS process에 signal을 보내거나 실제 container를 종료하지 않는다. GameHub/Fastify 단위의 상태 전이와 signal adapter의 호출 계약을 검증한다.

## 4. Container의 강제 종료 예산을 application보다 길게 둔다

### `312ddbc6fbe2`

production compose의 API service에 다음 설정이 추가된다.

```yaml
stop_grace_period: 70s
```

application drain 예산은 60초다. container runtime이 60초에 바로 강제 종료한다면 `beginDrain()` timeout 직후 실행될 `app.close()`와 repository/resource cleanup 시간이 없다. 70초는 drain보다 10초 긴 예산을 제공한다.

이 값은 “모든 환경에서 cleanup이 10초 안에 끝난다”는 측정 결과가 아니다. application timeout보다 platform kill deadline이 뒤에 있어야 한다는 ordering constraint를 구현한 값이다.

### `73ba979841cd`

production compose test는 duration string을 초로 바꾸고 API의 `stop_grace_period >= 60`을 요구한다.

```js
assert.ok(parseDurationSeconds(
  services.api.stop_grace_period
) >= 60);
```

테스트는 exact `70s`를 고정하지 않는다. 이후 값을 더 늘리거나 `1m10s`처럼 표현해도 application drain budget 이상이면 계약을 만족한다. 또한 Docker를 실제로 띄워 signal→drain→kill 순서를 재현하지 않는 정적 compose 검증이다.

## 최종 상태와 불변 조건

| 시점 | readiness | 새 match | waiting work | active room | 최종 cleanup |
| --- | --- | --- | --- | --- | --- |
| 정상 운영 | 200 / accepting | 허용 | 유지 | 실행 | 없음 |
| `beginDrain()` 직후 | 503 / draining | `server_draining` | timer 제거·error 통지 | 계속 실행 | 대기 |
| room이 0개가 됨 | 503 / draining | 거부 | 없음 | 없음 | drain success 후 `app.close()` |
| 60초 timeout | 503 / draining | 거부 | 없음 | 결과에 남은 수 기록 | `app.close()`가 강제 runtime cleanup |
| shutdown callback 실패 | 이미 가능한 범위에서 draining | 거부 방향 유지 | best effort | 보장되지 않음 | exitCode=1, best-effort `app.close()` |

최종적으로 성립하는 규칙은 다음과 같다.

- readiness는 database 상태뿐 아니라 traffic admission state를 표현한다.
- drain을 시작한 뒤에는 active-room count가 증가할 수 있는 match entry point를 통과시키지 않는다.
- waiting queue는 보존하지 않고 제거하지만, 이미 active인 room은 bounded timeout 동안 계속 진행한다.
- 여러 signal은 하나의 shutdown sequence만 만든다.
- application의 최대 drain 시간보다 container 강제 종료 유예가 짧아서는 안 된다.
- timeout은 cleanup 생략이 아니라 자연 종료 단계에서 final close 단계로 넘어가는 경계다.

## 이 Thread가 다루지 않는 것

- startup 시 migration/database readiness 판정은 `01-startup-liveness-readiness-and-storage-state.md`의 범위다.
- active room의 scheduler·snapshot·reconnect lifecycle은 `04-gamehub-runtime-integration-shared-scheduling-and-congestion.md`에서 다룬다.
- database 또는 edge failure를 실제로 주입하고 readiness 회복을 polling하는 절차는 `06-load-fault-recovery-and-pool-error-containment.md`의 범위다.
- Kubernetes `preStop`, load balancer deregistration propagation, multi-instance room handoff는 이 커밋 집합에 없다.
- 70초가 운영 환경에서 충분하다는 실측 결과와 실제 Docker signal 통합 실행 결과는 이 Thread의 source가 제공하지 않는다.

> 검토 범위: 표시된 exact SHA의 diff와 해당 시점 source/test를 조사했습니다. test suite, 실제 signal process test, Docker 종료 시나리오는 실행하지 않았습니다.
