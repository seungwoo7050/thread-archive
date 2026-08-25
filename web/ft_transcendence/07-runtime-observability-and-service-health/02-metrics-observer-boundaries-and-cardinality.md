# Thread: 관측은 의미를 소유한 경계에서만 기록한다

Project: `ft_transcendence` · 영역: metrics, observer boundary, cardinality

## 개요

metric을 추가하는 일보다 어려운 문제는 **어디서 기록할지**와 **무엇을 label로 허용할지**다. HTTP hook에서 모든 domain 결과를 추측하거나, raw URL·user ID·room ID를 label로 넣으면 관측값은 부정확해지고 series 수는 입력 크기에 비례해 늘어난다.

이 Thread가 만든 기준은 다음과 같다.

- metric registry는 app instance가 소유하고 app close와 함께 정리한다.
- HTTP는 route template, repository는 bounded operation name, domain observer는 bounded outcome만 label로 쓴다.
- snapshot 전달·drop은 실제 queue state를 소유한 `LatestSnapshotBuffer`가 기록 시점을 결정한다.
- request/user/room/match 식별자는 metric label이 아니라 structured log correlation에만 남긴다.
- event-loop lag는 runtime metric으로 노출되고 load profile의 threshold 입력이 되지만, 이 history 자체가 실제 부하 성능을 증명하지는 않는다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `6bf29a5acf35` | build(api): metrics 수집 의존성 추가 | C | - | `prom-client`를 API runtime dependency로 추가 |
| 2 | `69278d8fc456` | feat(metrics): runtime gauge registry 추가 | B | `REALTIME, OPERATIONS, OBSERVABILITY` | app 전용 registry와 connection/queue/room gauge 도입 |
| 3 | `02b3b3a32f14` | feat(metrics): HTTP와 readiness 측정 추가 | B | `OPERATIONS, OBSERVABILITY, PERF` | route template 기반 request latency와 readiness latency, `/metrics` 추가 |
| 4 | `843d355afc69` | feat(metrics): repository operation 측정 추가 | B | `PERSISTENCE, OBSERVABILITY` | repository call을 원래 receiver semantics를 보존한 채 관측 |
| 5 | `e08367a1be5e` | feat(metrics): game room과 reconnect 관측 추가 | B | `AUTH, PROTOCOL, REALTIME` | GameHub domain event를 observer로 app 관측 경계에 전달 |
| 6 | `e850b3356b9b` | feat(metrics): match finalization 결과 관측 추가 | B | `REALTIME, PERSISTENCE, OBSERVABILITY` | memory/database finalization의 success/failure를 의미가 확정된 지점에서 기록 |
| 7 | `c0d184bcc928` | feat(metrics): snapshot delivery와 drop 관측 추가 | B | `REALTIME, OBSERVABILITY, PERF` | latest-value buffer가 delivery delay와 drop reason을 직접 보고 |
| 8 | `685d85c863a4` | test(metrics): database와 snapshot 지표 검증 | B | `AUTH, REALTIME, PERSISTENCE` | metric 존재, route label, 식별자 비노출과 snapshot 의미를 회귀 검증 |
| 9 | `1baf4c5a57ba` | feat(metrics): event-loop lag 측정 추가 | B | `OBSERVABILITY` | Node event-loop delay histogram에서 p95 gauge 수집 |
| 10 | `66b8f07c2387` | test(load): event-loop lag를 부하 profile에 노출 | B | `OPERATIONS, OBSERVABILITY, PERF` | loopback metrics endpoint와 k6 teardown 수집 경로 추가 |
| 11 | `697a63ebb8c8` | test(load): event-loop lag 임계값 검증 | B | `OPERATIONS, OBSERVABILITY, PERF` | p95 50ms threshold와 load overlay 계약을 source test로 고정 |

## 1. Registry는 전역 singleton이 아니라 app resource다

### `6bf29a5acf35`, `69278d8fc456`

첫 commit은 `prom-client`만 추가한다. 실제 ownership은 다음 commit의 `ApiMetrics`에서 생긴다. `ApiMetrics`는 자체 `Registry`를 만들고 default process metrics와 application gauge를 그 registry에만 등록한다.

```ts
export class ApiMetrics {
  private readonly registry = new Registry();
  private readonly connections = new Gauge({
    name: "pong_pong_api_connections",
    help: "Current websocket connection count",
    registers: [this.registry]
  });

  close(): void {
    this.registry.clear();
  }
}
```

connection·queued player·active room 값은 gauge 내부에 별도 mutable truth를 복제하지 않고, scrape 시 `readGameStats()`가 반환한 현재 GameHub 상태로 갱신한다. 테스트 app을 여러 번 만들거나 같은 process 안에서 app을 재기동해도 global registry의 중복 등록과 이전 instance의 stale metric을 남기지 않는 구조다.

### `02b3b3a32f14` — HTTP label은 raw URL이 아니다

Fastify `onResponse` hook은 method, route, status, duration을 기록한다. 핵심은 route label이다.

```ts
app.addHook("onResponse", (request, reply, done) => {
  metrics.observeRequest(
    request.method,
    request.routeOptions.url ?? "unmatched",
    reply.statusCode,
    reply.elapsedTime
  );
  done();
});
```

`/users/123?tab=history` 같은 raw URL을 쓰지 않고 `/users/:id` 형태의 route template을 사용하므로 user ID와 query 값이 새로운 series를 만들지 않는다. 같은 commit에서 readiness duration/result가 별도 histogram으로 기록되고 `/metrics`가 registry exposition을 반환한다. app `onClose`는 `metrics.close()`를 호출한다.

## 2. Repository 계측은 원래 객체의 호출 semantics를 보존해야 한다

### `843d355afc69`

repository를 `Proxy`로 감쌀 때 흔한 오류는 method를 꺼낸 뒤 wrapper의 `this`로 호출하는 것이다. 이 commit은 property lookup과 invocation 모두 원래 target을 receiver로 유지한다.

```ts
const value = Reflect.get(target, property, target);
if (typeof property !== "string" || typeof value !== "function") {
  return value;
}
return (...args: unknown[]) => {
  const startedAt = performance.now();
  let result: unknown;
  try {
    result = Reflect.apply(
      value as (...methodArgs: unknown[]) => unknown,
      target,
      args
    );
  } catch (error) {
    metrics.observeDatabaseOperation(
      property,
      "failure",
      performance.now() - startedAt
    );
    throw error;
  }
  return Promise.resolve(result).then(
    (resolved) => {
      metrics.observeDatabaseOperation(
        property,
        "success",
        performance.now() - startedAt
      );
      return resolved;
    },
    (error) => {
      metrics.observeDatabaseOperation(
        property,
        "failure",
        performance.now() - startedAt
      );
      throw error;
    }
  );
};
```

관측 wrapper는 정책을 바꾸지 않는다.

- synchronous throw는 failure로 기록한 뒤 그대로 다시 던진다.
- Promise rejection도 failure로 기록한 뒤 그대로 전달한다.
- 성공 결과는 원래 값으로 resolve한다.
- label의 `operation`은 알려진 repository method 집합으로 제한하고 나머지는 `other`로 접는다.

따라서 repository method 이름이 bounded vocabulary인 동안 argument의 user ID, room ID, query text는 series key가 되지 않는다.

## 3. Domain 결과는 결과를 확정한 owner가 observer에 전달한다

### `e08367a1be5e` — room과 reconnect

`GameHubObserver`는 room creation과 reconnect 결과를 app에 알린다. room/user/request 식별자는 structured log context에는 남지만 metric은 reconnect `success | expired`처럼 유한한 결과만 사용한다. observer가 없는 GameHub도 같은 domain 동작을 유지하므로 관측은 optional side effect다.

### `e850b3356b9b` — finalization

match finalization은 persistence 종류와 결과가 실제로 결정되는 분기에서 기록된다.

- guest/memory 결과는 memory retention과 broadcast 전에 success로 관찰된다.
- database call이 reject하면 failure를 관찰한 뒤 error를 다시 던진다.
- database finalize가 성공하면 repository가 돌려준 match ID와 함께 success를 관찰한다.

metric label은 `persistence=memory|database`, `outcome=success|failure`뿐이다. room ID, match ID, user ID는 log correlation용 context이며 Prometheus label로 들어가지 않는다.

### `c0d184bcc928` — snapshot observer는 buffer 안에 있어야 한다

snapshot drop은 GameHub가 `enqueue()`를 호출한 시점에는 아직 확정되지 않는다. 기존 pending snapshot이 새 값으로 교체되는지, socket이 닫혔는지, buffered amount가 hard limit에 닿았는지, send callback이 성공했는지는 `LatestSnapshotBuffer`만 안다.

이 commit은 pending 값을 payload 문자열에서 enqueue time을 가진 object로 바꾸고, 실제 state transition에서 callback을 호출한다.

| Buffer transition | 관측 결과 |
| --- | --- |
| pending snapshot을 새 snapshot이 대체 | drop reason `replaced` |
| close되며 pending 값 폐기 | close 이유에 따른 drop |
| hard limit 또는 soft congestion timeout | `congestion` |
| socket close/send error | `connection_closed` |
| send callback 성공 | enqueue→callback delivery delay |

이 위치 선택이 중요하다. 바깥 caller에서 “보냈다”고 세면 실제 drop을 success로 오인하고, 반대로 enqueue마다 drop을 추정하면 정상 전송을 누락한다.

## 4. Cardinality 회귀는 metric text 자체를 검사한다

### `685d85c863a4`

테스트는 두 종류의 경계를 확인한다.

1. snapshot buffer에서 pending replacement와 fake clock 기반 50ms delivery delay가 observer callback으로 전달된다.
2. `/metrics` exposition에 필요한 metric과 route template `/health/live`가 존재하되 `requestId`, `userId`, `roomId`, `matchId` 문자열이 나타나지 않는다.

같은 history의 logging test는 correlation ID가 log에는 남고 secret/query가 redaction되는지 확인한다. 즉 “식별자를 버린다”가 아니라 **metric과 log의 역할을 분리**한다.

이 테스트는 전체 가능한 label value 수를 수학적으로 증명하지 않는다. 새 metric을 추가할 때 임의 식별자를 label로 넣는 회귀를 대표적인 exposition에서 잡는 focused contract다.

## 5. Event-loop lag는 runtime signal에서 load gate로 이어진다

### `1baf4c5a57ba`

`monitorEventLoopDelay({ resolution: 20 })`를 app metric owner가 활성화하고, scrape 시 p95 nanosecond 값을 second gauge로 변환한다. non-finite 값은 0으로 정규화하며 `close()`에서 monitor를 disable한다.

```ts
this.eventLoopLagP95 = new Gauge({
  name: "pong_pong_api_event_loop_lag_p95_seconds",
  help: "95th percentile of recorded event loop delay in seconds",
  registers: [this.registry],
  collect: () => {
    const delayNanoseconds = this.eventLoopDelay.percentile(95);
    this.eventLoopLagP95.set(
      Number.isFinite(delayNanoseconds)
        ? delayNanoseconds / 1_000_000_000
        : 0
    );
  }
});
```

이 gauge는 request latency와 다르다. 특정 route가 아니라 process가 timer/callback을 제때 실행하지 못한 정도를 나타낸다.

### `66b8f07c2387`, `697a63ebb8c8`

load overlay는 API의 metrics port를 loopback `14000`에만 노출한다. k6 `teardown()`은 `/metrics`에서 `pong_pong_api_event_loop_lag_p95_seconds`를 읽어 millisecond Trend에 넣고, load profile은 `p(95)<=50`을 threshold로 둔다.

후속 source test는 다음 계약을 고정한다.

- metric 이름이 production observability code에 존재한다.
- load script가 metrics URL과 해당 sample을 읽는다.
- threshold가 p95 50ms다.
- compose mapping은 loopback으로 제한된다.

이 두 commit에는 k6 실행 output이나 실제 p95 수치가 없다. 따라서 “50ms를 만족했다”는 증거가 아니라 “50ms를 넘으면 run을 실패시키도록 구성했다”는 증거다.

## 최종 관측 경계

| 신호 | 실제 의미를 소유한 위치 | bounded label | 식별자 처리 |
| --- | --- | --- | --- |
| HTTP duration | Fastify response hook | method, route template, status | raw URL/query 사용 안 함 |
| readiness duration | readiness handler | ready/not_ready | DB 상세 오류 미노출 |
| repository duration | repository proxy | known operation/other, success/failure | arguments와 query text 사용 안 함 |
| room/reconnect | GameHub transition | bounded outcome | IDs는 log context |
| match finalization | persistence 결과 분기 | persistence, outcome | room/match/user IDs는 log context |
| snapshot delivery/drop | `LatestSnapshotBuffer` | drop reason 또는 무label delay | connection ID 없음 |
| event-loop lag | Node delay monitor | label 없음 | process gauge |
| live connection/queue/room | scrape-time GameHub stats | label 없음 | aggregate gauge |

## 이 Thread의 경계

- fixed-step, heartbeat, input gate, latest snapshot buffer 자체의 알고리즘은 `03-runtime-limiter-primitives-and-bounded-work`에서 다룬다.
- callback delay를 congestion으로 오판한 초기 snapshot 정책의 수정은 `04-gamehub-runtime-integration-shared-scheduling-and-congestion`에 속한다.
- 500-connection load scenario와 fault recovery automation은 `06-load-fault-recovery-and-pool-error-containment`에서 다룬다.
- source/diff만 검사했으며 실제 metric scrape, load run, p95 측정 결과를 주장하지 않는다.

> 검토 범위: 표시된 exact SHA의 diff와 해당 시점 source/test를 조사했습니다. unit test, metrics scrape, k6 load는 실행하지 않았으며 실제 event-loop p95 수치를 주장하지 않습니다.
