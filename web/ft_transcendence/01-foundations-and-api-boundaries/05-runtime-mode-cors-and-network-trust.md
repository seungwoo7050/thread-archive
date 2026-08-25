# Thread: Runtime mode·CORS·network trust를 fail-closed 설정으로 수렴시킨다

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread에는 세 종류의 외부 신뢰 결정이 함께 있다.

1. 브라우저가 어떤 origin·method·header로 credentialed 요청을 보낼 수 있는가
2. 프로세스가 development/test/demo/production 중 어떤 capability로 시작하는가
3. client IP를 계산할 때 proxy가 보낸 주소를 신뢰할 것인가

초기 상태는 local prototype에 맞춰져 있다. port 4000, null `DATABASE_URL`, localhost web origin, 짧은 개발 secret이 기본값이고, DB URL이 없으면 memory repository를 선택한다. 이후 demo와 production이 생기면서 이 fallback은 더 이상 모든 mode에서 안전하지 않다.

최종 규칙은 다음과 같다.

```text
APP_MODE가 있으면 네 값 중 하나여야 함
  ├─ development
  ├─ test
  ├─ demo        → 32-byte 이상 SESSION_SECRET 필수, memory DB 허용
  └─ production  → 32-byte 이상 SESSION_SECRET + DATABASE_URL 필수

APP_MODE가 없으면 NODE_ENV로 production/test를 추론
그 외는 development

잘못된 explicit APP_MODE
  → development로 fallback하지 않음
  → startup 전 environment parsing 실패
```

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85ac2a949439` | `test(api): 실행 환경 기본값 검증` | B | `PROTOCOL, PERSISTENCE, TEST` | 명시적 환경 값과 local prototype default를 `readEnv` unit test로 고정한다. |
| 2 | `66155cf8a27d` | `fix(api): 변경 요청용 CORS method와 header 허용` | B | `AUTH, WEB` | credentialed browser mutation에 필요한 method와 request header를 Fastify CORS 설정에 명시한다. |
| 3 | `f801ccd09cf0` | `feat(guest): guest runtime 환경 경계 구성` | A | `AUTH, WEB` | `APP_MODE`, proxy-trust flag, 강한 session secret, browser-facing runtime URL을 환경 contract에 추가한다. |
| 4 | `2b274686e6d4` | `fix(guest): 체험 환경의 runtime 복구 제한` | A | `AUTH, REALTIME, RISK` | 중복 runtime-mode parser를 제거하고 explicit mode를 fail-closed로 해석하며 guest ticket·IP 추적 자원의 상한을 추가한다. |
| 5 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | `TOURNAMENT, OPERATIONS, RISK` | production mode에서 `DATABASE_URL`이 없으면 memory repository로 fallback하지 않고 startup 전에 실패시킨다. |
| 6 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | `OPERATIONS, TEST` | explicit·inferred production 모두 DB URL 없이는 실패하고 demo만 null DB를 허용하는지 검증한다. |

## `85ac2a949439` — local prototype의 기준선을 test로 고정한다

첫 test는 `readEnv`에 object를 직접 넘겨 process-global environment를 바꾸지 않고 parser만 검사한다.

```ts
const env = readEnv({
  API_PORT: "5001",
  DATABASE_URL: "postgres://example",
  WEB_ORIGIN: "http://web.local",
  SESSION_SECRET: "secret"
});

expect(env.port).toBe(5001);
expect(env.databaseUrl).toBe("postgres://example");
expect(env.webOrigin).toBe("http://web.local");
```

빈 object의 기대값은 다음과 같다.

```ts
expect(env.port).toBe(4000);
expect(env.databaseUrl).toBeNull();
expect(env.webOrigin).toBe("http://localhost:3000");
```

이 시점의 null DB는 의도된 local fallback이다. mode, secret 강도, proxy trust, production persistence는 아직 test 대상이 아니다. 잘못된 port 문자열도 거부하지 않는다. `Number(...)` 결과가 `NaN`인지 검사하는 규칙은 이 Thread에 없다.

## `66155cf8a27d` — credentialed mutation의 preflight를 허용한다

기존 CORS 설정은 origin과 `credentials: true`만 지정했다. 커밋은 mutation route에 필요한 method와 header를 명시한다.

```ts
app.register(cors, {
  origin: [
    webOrigin,
    "http://localhost:3000",
    "http://localhost:8080"
  ],
  credentials: true,
  methods: ["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
  allowedHeaders: ["content-type", "authorization"]
});
```

이 변경은 브라우저 preflight가 PATCH/DELETE와 Authorization header를 거부하던 구성을 보완한다. 그러나 origin allowlist에 localhost 두 개가 항상 포함되는 정책, CSRF 방어, 실제 browser에서의 cookie 속성은 검증하지 않는다.

또한 이 코드는 **`66155cf8a27d` 시점의 CORS 계약**이다. 이후 인증 경계 변경으로 header 목록이 바뀐 diff는 이 Thread의 포함 commit이 아니므로, 이 커밋을 현재 최종 목록의 근거로 소급하지 않는다.

## `f801ccd09cf0` — mode와 secret을 환경 값으로 만든 첫 단계

`.env.example`에 backend와 browser가 공유해야 할 runtime 값이 한 번에 등장한다.

```dotenv
DATABASE_URL=postgres://pong:pong@localhost:5432/pong_pong
SESSION_SECRET=dev-session-secret
APP_MODE=development
TRUST_PROXY=0
API_PORT=4000
WEB_ORIGIN=http://localhost:3000
NEXT_PUBLIC_API_BASE_URL=http://localhost:4000
NEXT_PUBLIC_WS_URL=ws://localhost:4000/ws
NEXT_PUBLIC_APP_MODE=development
```

`ApiEnv`에는 `appMode`와 `trustProxy`가 추가된다. demo 또는 production으로 판정되면 UTF-8 byte 길이 32 이상인 explicit secret을 요구한다.

```ts
if (
  (appMode === "demo" || appMode === "production")
  && (!configuredSecret
    || Buffer.byteLength(configuredSecret, "utf8") < 32)
) {
  throw new Error(
    "SESSION_SECRET must be at least 32 bytes in demo and production modes"
  );
}
```

proxy flag는 오직 문자열 `"1"`일 때만 true다.

```ts
trustProxy: input.TRUST_PROXY === "1"
```

### 아직 잘못된 mode parser

이 SHA의 `readAppMode`는 explicit `APP_MODE` 중 `"demo"`만 읽는다.

```ts
function readAppMode(input: NodeJS.ProcessEnv): ApiEnv["appMode"] {
  if (input.APP_MODE === "demo") return "demo";
  if (input.NODE_ENV === "production") return "production";
  if (input.NODE_ENV === "test") return "test";
  return "development";
}
```

그 결과 다음 explicit 값은 의도대로 처리되지 않는다.

- `APP_MODE=production`, `NODE_ENV` 없음 → development
- `APP_MODE=test`, `NODE_ENV` 없음 → development
- `APP_MODE=staging` → 오류가 아니라 development

첫 번째 경우에는 production secret rule도 우회된다. 따라서 이 커밋은 mode-aware validation을 도입하지만 explicit mode의 전체 집합을 아직 신뢰할 수 없다.

또한 이 diff가 직접 추가하는 것은 `TRUST_PROXY` **parsing**이다. Fastify `trustProxy` option으로 실제 전달하는 wiring을 이 커밋이 추가했다고 말할 수는 없다.

## `2b274686e6d4` — mode parser를 하나로 만들고 guest 자원 상한을 복구한다

이 커밋은 실제 diff가 넓다. 환경 parser 정리만으로 축소해서 읽으면 guest runtime의 핵심 변경을 놓친다.

### 1. explicit mode는 우선하며, 오타는 실패한다

`readAppMode`를 `env.ts`에서 export하고 `app.ts`의 중복 구현을 제거한다.

```ts
export function readAppMode(
  input: NodeJS.ProcessEnv = process.env
): ApiEnv["appMode"] {
  if (input.APP_MODE !== undefined) {
    if (
      ["development", "test", "production", "demo"]
        .includes(input.APP_MODE)
    ) {
      return input.APP_MODE as ApiEnv["appMode"];
    }
    throw new Error(
      `APP_MODE must be development, test, production, or demo: ${input.APP_MODE}`
    );
  }

  if (input.NODE_ENV === "production") return "production";
  if (input.NODE_ENV === "test") return "test";
  return "development";
}
```

우선순위는 명확하다.

```text
APP_MODE 존재
  → 유효한 네 값이면 그대로 사용
  → 그 외 startup error

APP_MODE 부재
  → NODE_ENV=production/test fallback
  → 그 외 development
```

`buildApp`의 기본 mode도 이 함수를 import해 사용하므로 환경 parser와 application-local parser가 서로 다른 값을 내는 문제가 사라진다.

### 2. guest WebSocket ticket이 client IP와 결합된다

이전에는 `issueWsTicket(user)`였지만 이후에는 Fastify가 계산한 `request.ip`를 함께 넘긴다.

```diff
- guests.issueWsTicket(user)
+ guests.issueWsTicket(user, request.ip)
```

ticket record는 IP를 저장하고, 같은 IP의 미사용 ticket 수와 분당 발급 수를 제한한다. 기본값은 다음과 같다.

```ts
DEFAULT_GUEST_TRACKED_IP_LIMIT = 10_000;
DEFAULT_GUEST_TICKETS_PER_IP = 4;
DEFAULT_GUEST_TICKET_ISSUE_LIMIT_PER_MINUTE = 30;
```

전체 active ticket limit도 유지되며, 같은 guest가 새 ticket을 받으면 이전 ticket을 지운다. 생성·발급 rate window는 `Map<string, RollingWindow>`로 저장하고 expiry timer에 `unref()`를 호출한다. 새 IP를 더 추적할 수 없는 경우와 rate limit 초과도 각각 typed `GuestAccessError`로 종료한다.

이 변화는 network trust와 직접 연결된다. quota key가 `request.ip`이므로 Fastify가 proxy header를 신뢰할지 여부가 공격자가 IP identity를 바꿀 수 있는지 결정한다. 해당 SHA의 application source는 `trustProxy` option을 Fastify에 전달하고 기본값은 false다. 다만 그 wiring 자체를 추가한 commit은 이 Thread map에 포함되어 있지 않으므로 `2b274...`의 diff로 소급하지 않는다.

### 보장과 남은 비용

- tracked IP map은 10,000개를 넘는 새 key를 무제한 추가하지 않는다.
- window와 ticket은 timer로 정리된다.
- per-IP pending ticket과 issue rate에 상한이 있다.
- default `trustProxy=false`에서는 forwarded address가 자동으로 client identity가 되지 않는다.

반면 pending ticket 수를 계산할 때 `this.tickets.values()`를 순회해 같은 IP를 센다. 이 commit은 bounded total ticket count를 전제로 하지만 별도 IP index를 만들지는 않는다. timer scheduling과 in-memory rate limit은 다중 API instance 사이에서 공유되지 않는다.

## `eb675ef74af3` — production의 memory fallback을 금지한다

composition root는 `databaseUrl`이 없으면 memory repository를 만든다. local과 demo에는 유용하지만 production에서 환경 변수 하나가 빠졌을 때 서버가 정상 기동해 휘발성 상태를 제공하는 것은 위험하다.

수정은 repository 생성 전에 있는 `readEnv`에서 실패시킨다.

```ts
const databaseUrl = input.DATABASE_URL ?? null;

if (appMode === "production" && !databaseUrl) {
  throw new Error(
    "DATABASE_URL is required in production mode"
  );
}
```

이 위치가 중요하다.

```text
readEnv
  ├─ failure: repository를 만들기 전 startup 중단
  └─ success: composition root가 URL 유무로 backend 선택
```

production은 이제 “DB URL이 없으면 memory로 복구”하지 않는다. demo는 별도 mode이며 memory storage를 계속 허용한다.

이 검증은 URL 문자열이 실제 PostgreSQL에 연결되는지, schema가 준비됐는지, credential이 올바른지는 확인하지 않는다. 그것은 connection/readiness 단계의 책임이다.

## `4633dfde208d` — explicit·inferred production을 모두 고정한다

테스트는 강한 secret을 준비한 뒤 DB URL만 비운다. secret 오류가 먼저 나서 persistence rule을 가리지 않게 하기 위해서다.

```ts
const strongSecret =
  "0123456789abcdef0123456789abcdef";
```

두 production 판정 경로가 모두 `DATABASE_URL` 오류인지 확인한다.

```ts
expect(() => readEnv({
  APP_MODE: "production",
  SESSION_SECRET: strongSecret
})).toThrow("DATABASE_URL");

expect(() => readEnv({
  NODE_ENV: "production",
  SESSION_SECRET: strongSecret
})).toThrow("DATABASE_URL");
```

마지막으로 demo의 의도된 차이를 고정한다.

```ts
expect(readEnv({
  APP_MODE: "demo",
  SESSION_SECRET: strongSecret
}).databaseUrl).toBeNull();
```

이 test는 pure environment-parser test다. process가 실제로 exit하는지, PostgreSQL 연결에 실패하면 어떻게 되는지, production container가 올바른 변수를 주입하는지는 증명하지 않는다.

## 최종 mode별 capability

| Mode | 판정 | `SESSION_SECRET` | `DATABASE_URL` | memory repository |
| --- | --- | --- | --- | --- |
| development | explicit 또는 fallback | 개발 기본값 허용 | 선택 | 허용 |
| test | explicit 또는 `NODE_ENV=test` | 개발 기본값 허용 | 선택 | 허용 |
| demo | explicit `APP_MODE=demo` | 32 byte 이상 필수 | 선택 | 허용 |
| production | explicit 또는 `NODE_ENV=production` | 32 byte 이상 필수 | 필수 | 금지 |
| 잘못된 explicit 값 | 해당 없음 | 검사 전 실패 | 검사 전 실패 | 기동하지 않음 |

```text
process.env
  → readAppMode (explicit 우선, invalid fail)
  → secret 강도 검사
  → production DB capability 검사
  → port/origin/trustProxy 파싱
  → composition root
      ├─ PostgreSQL repository
      └─ 허용된 mode에서만 memory repository
```

이 Thread가 만드는 핵심 변화는 “누락된 환경을 편리한 local 값으로 복구한다”는 초기 전략을 mode별로 제한한 것이다. development와 demo의 편의는 유지하지만, production은 identity secret과 persistent storage가 없으면 시작하지 않는다.

> 검사 범위: 지정 브랜치의 exact SHA diff와 해당 SHA source/test를 확인했다. 실제 process startup, browser preflight, reverse proxy, PostgreSQL 연결은 실행하지 않았다.
