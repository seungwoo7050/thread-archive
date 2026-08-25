# 개발 Thread 06 — production에서 memory fallback을 시작 전에 거부하기

## 개요

API startup은 `DATABASE_URL`이 있으면 PostgreSQL repository를, 없으면 memory repository를 선택한다. development·test·demo에서는 이 fallback이 유용하지만 production에서 같은 선택을 허용하면 configuration 누락이 정상 startup처럼 보일 수 있다.

```ts
const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl, options)
  : createMemoryRepository();
```

문제는 memory repository 자체가 아니라 **mode와 storage durability의 조합**이다. production process가 URL 없이 memory backend로 시작하면 healthcheck와 HTTP request는 성공할 수 있지만, process restart 때 사용자·세션·경기·토너먼트 상태가 사라진다. 잘못된 배포가 crash가 아닌 정상 동작으로 관찰되는 위험이다.

이 Thread는 storage selection code를 복잡하게 바꾸지 않는다. 그보다 앞선 environment parser에서 다음 invariant를 적용한다.

> effective app mode가 `production`이면 non-empty `DATABASE_URL`이 반드시 있어야 한다. 없으면 repository 생성과 port bind 전에 throw한다.

`demo`는 strong session secret을 요구하지만 memory repository를 계속 허용한다. production과 demo를 “보안상 강한 mode”로 함께 묶으면서도 storage durability policy는 분리한다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `97d7ca714293` | `test(config): production fixture에 영속 DB 명시` | C | — | 기존 production positive fixture에 `DATABASE_URL`을 넣어 새 invariant를 받아들일 준비를 한다. |
| 2 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | `TOURNAMENT, OPERATIONS, RISK` | environment parsing 단계에서 production의 DB URL 누락을 fail-fast한다. |
| 3 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | `OPERATIONS, TEST` | explicit/inferred production rejection과 demo allowance를 하나의 mode matrix로 검증한다. |

## `97d7ca714293` — positive production fixture를 먼저 현실적인 configuration으로 바꾸다

기존 test는 `APP_MODE=production`과 strong session secret만 제공하고 app mode resolution을 확인했다.

```diff
 const env = readEnv({
   APP_MODE: "production",
+  DATABASE_URL: "postgres://production.example/pong",
   SESSION_SECRET: "0123456789abcdef0123456789abcdef"
 });
```

이 commit은 production behavior를 아직 바꾸지 않는다. `readEnv()`도 repository selection도 그대로다. 다음 fix가 `DATABASE_URL`을 요구하게 되면 기존 positive fixture가 unrelated failure로 깨지므로, 먼저 fixture가 production의 intended durable configuration을 표현하도록 바꾼다.

중요도가 C인 이유도 여기에 있다. 새로운 invariant를 만들거나 검증하는 commit이 아니라 **후속 invariant를 받아들일 baseline 정리**다.

이 순서는 test가 무엇을 검증하는지 보존한다. app mode test는 계속 “explicit production이 `NODE_ENV` 없이 production으로 해석되는가”를 확인하고, DB requirement failure는 새 dedicated test가 맡는다.

## `eb675ef74af3` — storage policy를 repository factory보다 앞에서 강제하다

변경은 `readEnv()` 안에 있다.

```diff
 export function readEnv(input = process.env): ApiEnv {
   const appMode = readAppMode(input);
   const configuredSecret = input.SESSION_SECRET;
   if (
     (appMode === "demo" || appMode === "production")
     && (!configuredSecret || Buffer.byteLength(configuredSecret, "utf8") < 32)
   ) {
     throw new Error(
       "SESSION_SECRET must be at least 32 bytes in demo and production modes"
     );
   }
+
+  const databaseUrl = input.DATABASE_URL ?? null;
+  if (appMode === "production" && !databaseUrl) {
+    throw new Error("DATABASE_URL is required in production mode");
+  }
+
   return {
     port: Number(input.API_PORT ?? 4000),
-    databaseUrl: input.DATABASE_URL ?? null,
+    databaseUrl,
     /* ... */
   };
 }
```

### 왜 repository factory 안이 아니라 environment parser인가

API entry의 실제 order는 다음과 같다.

```ts
const env = readEnv();

const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl, {
      onPoolError: /* ... */
    })
  : createMemoryRepository();

const app = buildApp({
  repo,
  webOrigin: env.webOrigin,
  appMode: env.appMode,
  sessionSecret: env.sessionSecret,
  trustProxy: env.trustProxy
});

await app.listen({ port: env.port, host: "0.0.0.0" });
```

`readEnv()`가 throw하면 다음 동작이 모두 일어나지 않는다.

- memory/PostgreSQL repository 생성
- Fastify app graph 생성
- graceful shutdown hook 설치
- network port bind
- readiness endpoint 공개

즉 fail-closed boundary가 “첫 request”나 “첫 DB write”가 아니라 **process construction 이전**이다. 잘못된 production이 잠시라도 healthy로 보이는 시간을 만들지 않는다.

### mode resolution이 먼저다

`readAppMode()`의 precedence는 다음과 같다.

```ts
export function readAppMode(input = process.env) {
  if (input.APP_MODE !== undefined) {
    if (["development", "test", "production", "demo"].includes(input.APP_MODE)) {
      return input.APP_MODE;
    }
    throw new Error(`APP_MODE must be ...: ${input.APP_MODE}`);
  }
  if (input.NODE_ENV === "production") return "production";
  if (input.NODE_ENV === "test") return "test";
  return "development";
}
```

따라서 DB requirement는 raw environment variable 하나가 아니라 **effective mode**에 적용된다.

| 입력 | effective mode | `DATABASE_URL` 누락 처리 |
| --- | --- | --- |
| `APP_MODE=production` | production | throw |
| `APP_MODE` 없음, `NODE_ENV=production` | production | throw |
| `APP_MODE=demo` | demo | 허용 |
| `APP_MODE=test` | test | 허용 |
| 아무 mode 없음 | development | 허용 |
| invalid `APP_MODE=staging` | 없음 | mode validation에서 먼저 throw |

`APP_MODE`가 있으면 `NODE_ENV`보다 우선한다. 예를 들어 `APP_MODE=demo`, `NODE_ENV=production`이면 effective mode는 demo이고 memory storage가 허용된다. 이 precedence는 source에서 확정되지만, 후속 DB regression은 대표 세 case만 직접 실행한다.

### secret validation과 storage validation은 별개의 gate다

`demo`와 `production`은 둘 다 최소 32-byte session secret을 요구한다. DB check는 그 뒤에 있다. 따라서 production input이 secret과 DB를 모두 누락하면 먼저 session secret error가 발생한다.

이 ordering은 production storage test가 strong secret을 제공해야 하는 이유다. test가 DB failure를 관찰하려면 앞선 security gate를 통과해야 한다.

### non-empty 여부만 검사한다

```ts
const databaseUrl = input.DATABASE_URL ?? null;
if (appMode === "production" && !databaseUrl) throw ...;
```

- `undefined`와 `null` fallback은 `null`이 되어 거부된다.
- empty string `""`도 falsy라 거부된다.
- whitespace-only string이나 malformed URL은 truthy라 이 check를 통과한다.
- DB network reachability, credential validity, schema current 여부는 repository connect/readiness 단계가 확인해야 한다.

따라서 이 fix의 invariant는 “valid PostgreSQL connectivity”가 아니라 **production이 명시적인 durable storage configuration 없이 memory backend를 선택하지 않는다**는 것이다.

### 기존 fallback을 유지한 이유

repository selection은 그대로다.

```ts
const repo = env.databaseUrl
  ? createPostgresRepository(env.databaseUrl, options)
  : createMemoryRepository();
```

이 branch를 production-aware factory로 바꾸지 않고 parser가 invalid combination을 제거한다. 결과적으로 factory는 단순한 data dependency selection을 유지하고, environment policy는 `readEnv()` 한 곳에 모인다.

이 구조의 장점은 testability다. network나 repository를 만들지 않고 plain object input만으로 production/storage matrix를 검증할 수 있다.

## `4633dfde208d` — explicit production, inferred production, demo를 한 regression으로 고정하다

후속 test는 strong secret을 공통 입력으로 사용해 DB gate만 관찰한다.

```ts
it("rejects an in-memory repository in every production mode", () => {
  const strongSecret = "0123456789abcdef0123456789abcdef";

  expect(() => readEnv({
    APP_MODE: "production",
    SESSION_SECRET: strongSecret
  })).toThrow("DATABASE_URL");

  expect(() => readEnv({
    NODE_ENV: "production",
    SESSION_SECRET: strongSecret
  })).toThrow("DATABASE_URL");

  expect(readEnv({
    APP_MODE: "demo",
    SESSION_SECRET: strongSecret
  }).databaseUrl).toBeNull();
});
```

### 첫 두 assertion이 서로 다른 이유

- 첫 case는 explicit operator contract인 `APP_MODE=production`을 검증한다.
- 둘째 case는 legacy/general Node convention인 `NODE_ENV=production` fallback을 검증한다.

둘 중 하나만 있으면 mode resolution refactor가 다른 production entry를 memory fallback으로 다시 열 수 있다.

### demo assertion은 단순 예외가 아니다

production rejection만 검사하면 후속 구현이 `demo`에도 DB를 강제하는 over-restriction으로 바뀌어도 test가 통과한다. 마지막 assertion은 intentional allowance를 함께 고정한다.

```text
production → durable storage required
     demo → strong secret required, memory storage allowed
```

이 차이는 비회원 demo job이 DB 없이 compiled API를 시작하는 Thread 04의 구조와도 맞는다.

### test 이름보다 실제 검증 범위는 좁다

test 이름은 “in-memory repository 거부”지만 repository를 생성하지 않는다. 검증하는 것은 다음뿐이다.

- `readEnv()`가 production no-DB input에서 throw하는가
- error message에 `DATABASE_URL`이 포함되는가
- demo no-DB input의 normalized `databaseUrl`이 `null`인가

API process가 실제로 port bind 전에 종료하는지, memory repository factory가 호출되지 않았는지, malformed URL로 PostgreSQL connect가 실패하는지는 직접 관찰하지 않는다. call order는 entry source와 결합해 설명할 수 있지만 test evidence는 parser unit boundary다.

## 최종 startup decision

```text
process.env
   ↓
readAppMode()
   ├─ invalid APP_MODE → throw
   └─ effective mode
          ↓
strong SESSION_SECRET 검사 (demo/production)
          ↓
DATABASE_URL normalize
   ├─ production + missing/empty → throw
   └─ otherwise ApiEnv 반환
          ↓
repository 선택
   ├─ URL 있음 → PostgreSQL
   └─ URL 없음 → memory
          ↓
buildApp → shutdown hooks → listen
```

## 최종 mode/storage contract

| mode | strong session secret | DB URL | 허용 repository | startup 결과 |
| --- | --- | --- | --- | --- |
| development | 선택 | 선택 | PostgreSQL 또는 memory | 허용 |
| test | 선택 | 선택 | PostgreSQL 또는 memory | 허용 |
| demo | 필수 | 선택 | PostgreSQL 또는 memory | 허용 |
| production | 필수 | **필수** | PostgreSQL만 선택 가능 | 누락 시 construction 전 실패 |

## 해결한 위험과 남은 위험

### 해결한 위험

- production environment typo/누락이 silent memory fallback으로 바뀌는 문제
- process가 healthy로 보인 뒤 restart에서 data가 사라지는 잘못된 배포
- `APP_MODE`와 `NODE_ENV` 중 한 production entry만 보호하는 부분 fix
- production rule을 demo에 잘못 확장하는 regression

### 남은 위험

- malformed/whitespace `DATABASE_URL`
- database authentication·DNS·network failure
- schema pending/diverged 상태
- DB는 durable하지만 volume/backup이 잘못 구성된 deployment
- process start 뒤 DB connection이 끊기는 runtime failure
- secret rotation·external secret manager

이 문제들은 URL 존재 여부가 아니라 connection/readiness, migration, Compose volume, operational recovery의 책임이다. 이 Thread는 production이 **의도하지 않은 memory backend로 정상 시작하는 한 경로**를 가장 이른 configuration boundary에서 닫는다.

## 조사 범위

세 commit의 exact diff와 `eb675ef74af3` 시점의 `env.ts`·`index.ts`를 확인해 validation과 repository construction 순서를 연결했다. test는 source를 읽어 실제 assertion 범위를 parser unit boundary로 제한했다. API process나 PostgreSQL은 이 작업 환경에서 실행하지 않았다.
