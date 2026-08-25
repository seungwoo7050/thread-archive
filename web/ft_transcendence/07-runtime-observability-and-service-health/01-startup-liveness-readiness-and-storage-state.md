# Thread: 기동 상태를 트래픽 수용 계약으로 분리하기

Project: `ft_transcendence` · 영역: runtime observability and service health

## 개요

프로세스가 port를 열었다는 사실과 요청을 안전하게 받을 수 있다는 사실은 다르다. 이 Thread는 API bootstrap을 만든 뒤, 저장소 연결과 migration 상태를 readiness에 포함하고, startup data mutation을 제거하며, production이 memory repository로 조용히 기동하지 못하게 만드는 과정을 다룬다.

최종 계약은 네 문장으로 요약된다.

- `/health/live`는 process loop가 응답 가능한지만 말한다.
- `/health/ready`는 repository가 응답하고 schema가 현재 상태일 때만 200을 반환한다.
- API startup은 seed data를 생성하지 않는다.
- production은 `DATABASE_URL` 없이 시작하지 않는다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4b43a284e637` | feat(api): 실행 환경과 service bootstrap 구성 | B | `PERSISTENCE, OPERATIONS` | 환경을 읽고 repository·Fastify app·listen/close를 묶는 composition root 도입 |
| 2 | `85ac2a949439` | test(api): 실행 환경 기본값 검증 | B | `PROTOCOL, PERSISTENCE, TEST` | 명시적 환경값과 로컬 기본값을 고정 |
| 3 | `30aac132e14e` | feat(db): migration set 상태 검사 추가 | A | `PERSISTENCE, OPERATIONS, RISK` | bundled migration과 적용 migration의 집합 차이를 current/pending/diverged로 분류 |
| 4 | `2f05d5d79c64` | feat(db): repository readiness 경계 추가 | A | `PERSISTENCE, OPERATIONS` | 저장소마다 연결·migration 상태를 같은 contract로 보고하도록 확장 |
| 5 | `15002e229acb` | feat(ops): liveness와 readiness endpoint 추가 | A | `PROTOCOL, PERSISTENCE, OPERATIONS` | process 생존과 traffic admission을 별도 endpoint로 노출 |
| 6 | `6937cf60aeea` | test(ops): health와 database readiness 검증 | B | `AUTH, PERSISTENCE, OPERATIONS` | memory, rejected readiness, PostgreSQL migration 전후를 검증 |
| 7 | `e1a0316fbe84` | fix(api): startup seed 생성을 제거 | B | `PERSISTENCE` | process startup에서 data mutation 제거 |
| 8 | `5cac4843fd9b` | test(api): startup seed 금지 검증 | B | `REALTIME, TEST` | bootstrap source와 smoke flow가 implicit seed에 의존하지 않도록 고정 |
| 9 | `eb675ef74af3` | fix(config): production에서 영속 저장소 요구 | A | `TOURNAMENT, OPERATIONS, RISK` | production의 memory fallback을 fail-closed로 변경 |
| 10 | `4633dfde208d` | test(config): production memory fallback 거부 검증 | A | `OPERATIONS, TEST` | 명시·추론 production mode 모두 DB URL 부재를 거부하는지 검증 |

## 1. Bootstrap은 repository 선택과 resource 정리의 소유자가 된다

### `4b43a284e637` — composition root

`readEnv()`는 port, database URL, web origin, session secret을 하나의 `ApiEnv`로 정규화하고, `index.ts`는 그 결과로 repository를 선택한다. 이 시점의 bootstrap에는 `ensureSeedData()`도 포함돼 있어 process startup이 application data를 바꾸는 책임까지 갖는다.

```ts
const env = readEnv();
const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl)
  : createMemoryRepository();
await repo.ensureSeedData();

const app = buildApp({ repo, webOrigin: env.webOrigin });
app.addHook("onClose", async () => {
  await repo.close();
});
```

중요한 설계점은 repository를 만든 주체가 정상 종료와 listen 실패 양쪽에서 `close()` 책임도 진다는 것이다. 반면 DB URL 부재를 memory mode로 해석하는 정책은 아직 환경에 관계없이 허용된다.

### `85ac2a949439` — 이 commit이 실제로 고정한 범위

테스트는 두 경우만 다룬다.

1. `API_PORT`, `DATABASE_URL`, `WEB_ORIGIN`, `SESSION_SECRET`을 명시했을 때 그대로 읽는 경우
2. 빈 환경에서 local prototype 기본값을 사용하는 경우

잘못된 숫자, URL, mode를 거부하는 negative test는 이 SHA에 없다. 따라서 이 commit의 증거는 “기본값과 override가 정해졌다”까지이며, 일반적인 configuration validation을 증명하지 않는다.

## 2. Readiness는 DB ping이 아니라 schema 상태까지 포함한다

### `30aac132e14e` — migration set을 집합으로 비교한다

migration readiness는 단순히 “migration table이 존재하는가”가 아니다. repository에 번들된 migration 이름과 DB에 적용된 이름을 비교해 세 상태를 만든다.

```ts
export function compareMigrationSets(
  expectedNames: string[],
  appliedNames: string[]
): MigrationSetComparison {
  const expected = new Set(expectedNames);
  const applied = new Set(appliedNames);
  const missing = expectedNames.filter((name) => !applied.has(name));
  const unexpected = appliedNames.filter((name) => !expected.has(name));
  return {
    status: unexpected.length > 0
      ? "diverged"
      : missing.length > 0
        ? "pending"
        : "current",
    missing,
    unexpected
  };
}
```

`unexpected`가 하나라도 있으면 `diverged`가 우선한다. DB가 현재 binary가 모르는 migration을 이미 적용한 상태는 단순 pending보다 위험하기 때문이다. migration table 조회에서 PostgreSQL `42P01`만 “아직 적용된 migration이 없음”으로 변환하고, 다른 오류는 숨기지 않고 다시 던진다.

이 결정으로 schema state는 다음처럼 해석된다.

| DB 관찰 | migration 판정 | readiness 의미 |
| --- | --- | --- |
| expected와 applied 동일 | `current` | 수용 가능 |
| expected 중 일부가 없음 | `pending` | binary가 요구하는 schema 미완료 |
| applied에 알 수 없는 이름 존재 | `diverged` | 배포 artifact와 DB history 불일치 |
| migration table 없음(`42P01`) | empty applied set과 비교 | 모든 expected migration이 missing이면 pending |
| 그 밖의 query 오류 | 예외 전파 | DB down/unknown으로 처리할 상위 경계에 전달 |

### `2f05d5d79c64` — repository contract로 올린다

`AppRepository`에 `checkReadiness()`가 추가된다. PostgreSQL 구현은 먼저 `select 1`로 연결을 확인한 뒤 migration set을 검사하고, memory 구현은 DB가 process-local이므로 `database: "up"`, `migrations: "not_applicable"`를 반환한다.

```ts
export interface RepositoryReadiness {
  database: "up";
  migrations: "current" | "pending" | "diverged" | "not_applicable";
}
```

이 경계 덕분에 HTTP layer는 PostgreSQL 내부 query나 migration table을 알 필요 없이 traffic admission만 판단한다.

### `15002e229acb` — liveness와 readiness를 분리한다

- `/health/live`: repository를 호출하지 않고 항상 process 생존 응답을 만든다.
- `/health/ready`: `checkReadiness()` 결과가 `database=up`이고 migration이 `current` 또는 `not_applicable`일 때만 200이다.
- `pending`, `diverged`, repository exception은 503이다.
- exception response에는 내부 연결 문자열이나 query text를 넣지 않고 `down/unknown`으로 축약한다.

```ts
const repository = await repo.checkReadiness();
const ready = repository.database === "up"
  && (
    repository.migrations === "current"
    || repository.migrations === "not_applicable"
  );
const body = parseOutput(http.readyHealthResponseSchema, {
  status: ready ? "ready" : "not_ready",
  service: "pong-pong-api",
  checks: {
    lifecycle: "accepting",
    database: repository.database,
    migrations: repository.migrations
  }
});
return reply.code(ready ? 200 : 503).send(body);
```

이 SHA에서 `lifecycle`은 아직 항상 `accepting`이다. draining 상태는 별도 shutdown Thread의 후속 변경이며 여기로 소급하지 않는다.

### `6937cf60aeea` — evidence와 한계

테스트는 다음을 확인한다.

- legacy health와 새 live/ready route가 존재한다.
- memory repository는 `not_applicable`로 ready다.
- `checkReadiness()`가 reject하면 503이며 내부 PostgreSQL 문자열이 body에 노출되지 않는다.
- PostgreSQL integration에서 migration 실행 전은 pending, 실행 후는 current다.
- pure comparison에서 current/pending/diverged가 구분된다.

실제 DB process를 중단한 채 `/health/live`가 계속 200인지 관찰하는 fault test는 이 commit에 없다. liveness의 dependency 독립성은 route code 구조로 확인되고, DB failure recovery는 후속 fault scenario의 관심사다.

## 3. Startup은 data migration·seed command가 아니다

### `e1a0316fbe84` — implicit seed 제거

이 commit의 parent에서는 memory path에 남아 있던 startup seed 호출을 삭제한다.

```diff
 const env = readEnv();
 const repo = env.databaseUrl
   ? createPostgresRepository(env.databaseUrl)
   : createMemoryRepository();
-if (!env.databaseUrl) {
-  await repo.ensureSeedData();
-}
 
 const app = buildApp({
```

핵심은 seed 구현 자체를 없애는 것이 아니라 **process startup의 책임에서 제외**하는 것이다. 재시작 횟수, replica 수, deployment order가 application data를 바꾸지 않게 된다. 필요한 fixture나 demo data는 명시적인 명령·test setup이 소유해야 한다.

### `5cac4843fd9b` — 금지 규칙을 어떻게 고정했는가

이 테스트는 API를 기동해 mutation을 관찰하지 않는다. `index.ts` source를 읽어 `.ensureSeedData(`가 없는지 확인하는 정적 회귀다. 같은 commit의 WebSocket smoke flow도 미리 존재하는 NPC seed를 기대하지 않고 명시적으로 AI mode를 선택한다.

따라서 증명 범위는 다음과 같다.

- bootstrap source에 직접 seed 호출이 다시 들어오는 단순 회귀를 잡는다.
- smoke test가 implicit seeded NPC에 의존하지 않는다.
- 간접 호출, 다른 startup module의 mutation, 실제 DB write 부재까지 증명하지는 않는다.

## 4. Production 저장소 선택은 fail-closed다

### `eb675ef74af3` — mode와 storage를 함께 검증한다

기존 정책은 DB URL이 없으면 어디서든 memory repository를 선택했다. 이는 production misconfiguration이 정상 기동처럼 보이고, restart마다 data가 사라지는 위험을 만든다. 수정 후 `readEnv()`는 `APP_MODE` 또는 production 환경 판정 결과가 production인데 `DATABASE_URL`이 없으면 즉시 예외를 던진다.

```ts
const databaseUrl = input.DATABASE_URL ?? null;
if (appMode === "production" && !databaseUrl) {
  throw new Error("DATABASE_URL is required in production mode");
}
```

repository factory까지 도달하기 전에 configuration boundary에서 실패하므로, production의 memory fallback은 불가능해진다. demo/local mode의 memory repository는 의도된 별도 운용 형태로 남는다.

### `4633dfde208d` — 두 production 진입 경로를 고정한다

테스트는 다음을 구분한다.

- `APP_MODE=production` + DB URL 없음 → 거부
- `NODE_ENV=production`으로 production이 추론됨 + DB URL 없음 → 거부
- demo mode + DB URL 없음 → memory 허용

이 commit은 configuration function의 결과를 검증한다. 실제 container가 잘못된 환경으로 restart loop에 들어가는지까지 실행하는 deployment test는 아니다.

## 최종 상태

| 상황 | liveness | readiness | startup/storage 결과 |
| --- | ---: | ---: | --- |
| memory repository를 허용한 local/demo | 200 | 200 (`not_applicable`) | process-local 저장소 사용 |
| PostgreSQL up + migrations current | 200 | 200 | traffic 수용 |
| PostgreSQL up + migrations pending | 200 | 503 | migration 완료 전 traffic 차단 |
| PostgreSQL history diverged | 200 | 503 | artifact/DB 불일치 노출 |
| repository check 예외 | 200 | 503 (`down/unknown`) | 내부 오류 세부는 response에서 숨김 |
| production + DB URL 없음 | process 시작 전 실패 | 해당 없음 | memory fallback 금지 |
| API 재시작 | process 재기동 | 상태 재평가 | startup seed mutation 없음 |

## 이 Thread의 경계

- draining 중 readiness를 먼저 내리고 active room을 기다리는 절차는 `05-draining-readiness-and-graceful-shutdown`에서 다룬다.
- DB down/up을 Toxiproxy로 실제 전환해 recovery를 관찰하는 harness는 `06-load-fault-recovery-and-pool-error-containment`의 책임이다.
- metrics route와 readiness latency 측정은 `02-metrics-observer-boundaries-and-cardinality`에 속한다.
- 이 문서는 source/diff를 검사했으며 build·test·container 실행 결과를 주장하지 않는다.

> 검토 범위: 표시된 exact SHA의 diff와 해당 시점 source/test를 조사했습니다. build, unit/integration test, 실제 PostgreSQL·container 기동은 실행하지 않았습니다.
