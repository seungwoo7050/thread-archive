# 개발 Thread 04 — CI가 source 검사에서 compiled process와 browser flow까지 확장되는 과정

## 개요

이 Thread는 repository의 검증 명령을 정리하는 작은 build commit에서 시작해, CI가 서로 다른 failure domain을 독립 job으로 실행하고, 실제 compiled API·Next production server·PostgreSQL·Chromium을 연결하는 단계까지 확장되는 과정을 다룬다.

최종 CI가 구분하는 검증 층은 다음과 같다.

```text
정적 코드/단위 경계
  ├─ typecheck
  ├─ unit tests
  ├─ static contract tests
  └─ production build + artifact existence

실제 외부 resource 경계
  ├─ Testcontainers PostgreSQL integration
  ├─ compiled API + Next production server + HTTP/WS smoke
  ├─ registered-user browser E2E + PostgreSQL
  └─ demo-mode compiled process + guest browser E2E
```

여기서 “production process 검증”과 “production deployment 검증”은 다르다. `process-and-browser` job은 Thread 02의 emitted API와 `next start`를 실제로 실행하지만 `APP_MODE=development`이고, Caddy·production Compose·Docker image를 사용하지 않는다. 즉 compiled execution path와 browser delivery를 검증할 뿐, Thread 03의 배포 topology 전체를 재현하지 않는다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a9f8b8609711` | `build(repo): workspace 검증 명령 정리` | B | `PROTOCOL, REALTIME, OPERATIONS` | unit·HTTP smoke·WS smoke·E2E에 안정된 root command를 부여한다. |
| 2 | `68404e51ea53` | `ci(repo): typecheck·unit·build workflow 추가` | B | `PROTOCOL, PERSISTENCE, OPERATIONS` | clean runner에서 기본 정적/단위/build gate를 만든다. |
| 3 | `0e360d333540` | `ci(db): PostgreSQL integration 검사 실행` | B | `PERSISTENCE` | Testcontainers 기반 DB suite를 별도 job으로 분리한다. |
| 4 | `3367b4266049` | `ci(repo): process와 browser 검증 job 추가` | A | `REALTIME, PERSISTENCE, WEB` | PostgreSQL service, compiled API/Web, smoke, browser E2E를 한 lifecycle로 실행한다. |
| 5 | `7cb0d32b5be3` | `test(ci): process 검증 job contract 확인` | B | `PERSISTENCE, WEB, OPERATIONS` | workflow에 toolchain·검증 command·DB 준비 단계가 남아 있는지 source contract로 검사한다. |
| 6 | `bf0bc1199c84` | `ci(e2e): 비회원 체험 browser job 실행` | B | `PERSISTENCE, WEB, OPERATIONS` | demo mode 전용 compiled process와 guest browser flow를 독립 job으로 실행한다. |
| 7 | `0ae1ded2c56f` | `test(ci): guest browser job 요구 검증` | B | `WEB, OPERATIONS, TEST` | guest job의 mode 3종과 command/spec 연결을 정적 contract로 고정한다. |
| 8 | `f35728f4ef92` | `test(repo): 정적 계약 검사 명령 연결` | B | `PERSISTENCE, OPERATIONS, TEST` | CI·Docker·load contract를 하나의 root command로 묶는다. |
| 9 | `4f9e66c35586` | `ci(repo): 정적 계약 검사 실행` | B | `PERSISTENCE` | root static contract command를 verify job에서 실행한다. |
| 10 | `65512bc24161` | `fix(ci): 브라우저 E2E API origin 정렬` | B | `AUTH, REALTIME, WEB` | registered browser가 로그인 cookie를 받은 hostname과 direct API request hostname을 맞춘다. |
| 11 | `527921bc9d69` | `test(ci): 브라우저 E2E cookie origin 계약 검증` | B | `AUTH, WEB, OPERATIONS` | `API_BASE_URL=http://localhost:4000`을 workflow contract로 고정한다. |

## `a9f8b8609711` — 검증 종류를 command 이름으로 분리하다

root `package.json`과 Makefile이 같은 verification vocabulary를 갖는다.

```json
{
  "scripts": {
    "test": "pnpm unit",
    "unit": "pnpm -r test",
    "smoke:http": "node tests/smoke-api.mjs",
    "smoke:ws": "node tests/smoke-ws.mjs",
    "e2e": "playwright test",
    "test:e2e": "pnpm e2e"
  }
}
```

```make
unit:
	pnpm unit

smoke: smoke-http smoke-ws

smoke-http:
	pnpm smoke:http

smoke-ws:
	pnpm smoke:ws

e2e:
	pnpm e2e
```

이 commit은 새 test technique를 만들지 않는다. 중요한 변화는 caller가 `pnpm -r test`나 개별 script path를 기억하지 않고 **검증 목적을 안정된 root command로 선택**할 수 있다는 점이다.

- `unit`: 각 workspace의 process-local test
- `smoke:http`: 실행 중 API의 HTTP boundary
- `smoke:ws`: 실행 중 API의 WebSocket boundary
- `e2e`: browser flow

Makefile과 package script를 둘 다 유지하면 local operator와 CI가 다른 entry를 사용해도 실제 command는 root package에 모인다. 같은 diff의 `dev`·`down` target은 runtime convenience이지만, 이 Thread에서는 verification vocabulary에만 집중한다.

## `68404e51ea53` — clean checkout에서 기본 gate를 만들다

첫 workflow는 push와 pull request에 대해 하나의 `verify` job을 실행한다.

```yaml
name: CI

on:
  push:
  pull_request:

permissions:
  contents: read

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.32.1
      - uses: actions/setup-node@v4
        with:
          node-version: 24.18.0
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm unit
      - run: pnpm build
```

이 순서는 각 단계의 failure meaning을 보존한다.

1. frozen install이 실패하면 manifest와 lockfile의 불일치다.
2. typecheck가 실패하면 workspace type contract가 깨진 것이다.
3. unit이 실패하면 process-local behavior가 깨진 것이다.
4. build가 실패하면 production artifact graph를 만들 수 없는 것이다.

steps는 같은 workspace를 공유하므로 install 결과를 다음 단계가 소비하고, 앞 단계가 실패하면 뒤 단계가 실행되지 않는다. 15분 timeout은 job 전체가 무한 대기하는 것을 제한한다.

하지만 이 시점에는 PostgreSQL daemon, API process, Web server, browser가 없다. “build success”는 외부 resource와 process composition의 성공을 뜻하지 않는다.

## `0e360d333540` — DB integration을 unit과 다른 resource lifecycle로 분리하다

workflow에는 별도 `postgres-integration` job이 추가된다.

```yaml
postgres-integration:
  name: PostgreSQL integration tests
  runs-on: ubuntu-latest
  timeout-minutes: 15
  steps:
    - uses: actions/checkout@v4
    - uses: pnpm/action-setup@v4
      with:
        version: 10.32.1
    - uses: actions/setup-node@v4
      with:
        node-version: 24.18.0
        cache: pnpm
    - run: pnpm install --frozen-lockfile
    - run: pnpm postgres-integration
```

workflow YAML에는 PostgreSQL `services:` block이 없다. DB lifecycle은 test suite가 소유한다. root command는 DB package의 다음 script로 연결된다.

```json
"postgres-integration": "vitest run src/postgres.integration.test.ts \
  --testTimeout=120000 --hookTimeout=120000 \
  --teardownTimeout=30000 --no-file-parallelism --maxWorkers=1"
```

suite는 `@testcontainers/postgresql`로 `postgres:16-alpine`을 시작하고 종료한다.

```ts
const POSTGRES_IMAGE = "postgres:16-alpine";

beforeAll(async () => {
  container = await new PostgreSqlContainer(POSTGRES_IMAGE)
    /* configuration */
    .start();
  adminPool = new Pool({ connectionString: container.getConnectionUri() });
  await adminPool.query("select 1");
});

afterAll(async () => {
  try {
    await adminPool?.end();
  } finally {
    await container?.stop({ timeout: 10_000 });
  }
});
```

따라서 이 job은 단순히 “DB 관련 test command가 존재한다”를 검사하지 않는다. Docker가 가능한 runner에서 실제 PostgreSQL container를 만들고 migration·repository behavior를 검증한 뒤 pool과 container를 회수한다.

독립 job으로 둔 이유도 분명하다.

- Docker image pull·container start failure를 unit failure와 분리한다.
- 긴 integration timeout을 기본 verify job과 분리한다.
- `--no-file-parallelism --maxWorkers=1`로 공유 container와 schema lifecycle을 순차화한다.

이 commit이 CI에 추가하는 것은 existing DB suite의 execution이다. test suite 내부의 migration·seed·repository semantics는 persistence Thread에서 다룬다.

## `3367b4266049` — build artifact를 process로 시작하고 browser까지 연결하다

이 commit은 Thread의 중심이다. `process-and-browser` job은 외부 PostgreSQL service를 만들고, production build 산출물로 API와 Web을 실제 시작한 뒤 HTTP·WebSocket·browser test를 실행한다.

### job-level infrastructure

```yaml
process-and-browser:
  name: HTTP, WebSocket, and browser tests
  runs-on: ubuntu-latest
  timeout-minutes: 20
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_DB: pong_pong_test
        POSTGRES_USER: pong
        POSTGRES_PASSWORD: pong
      ports:
        - 5432:5432
      options: >-
        --health-cmd "pg_isready -U pong -d pong_pong_test"
        --health-interval 5s
        --health-timeout 3s
        --health-retries 20
  env:
    DATABASE_URL: postgresql://pong:pong@127.0.0.1:5432/pong_pong_test
    API_BASE_URL: http://127.0.0.1:4000
    WS_URL: ws://127.0.0.1:4000/ws
```

이 job은 Testcontainers suite와 달리 workflow service를 사용한다. API process와 smoke/E2E가 같은 fixed database URL을 공유해야 하기 때문이다.

### artifact producer와 database preparation

```yaml
- name: Install Chromium
  run: pnpm exec playwright install --with-deps chromium

- name: Build production processes
  run: pnpm build

- name: Prepare the process-test database
  run: |
    pnpm --filter @pong-pong/db migrate
    pnpm --filter @pong-pong/db seed:dev
```

`pnpm build`는 Thread 02의 shared→DB→API→Web artifact graph를 생성한다. DB migration과 development seed는 process start 전에 명시적으로 실행된다. API startup이 schema/seed mutation을 암묵적으로 소유하지 않는다.

### process lifecycle와 readiness polling

```bash
set -euo pipefail
node apps/api/dist/index.js >/tmp/pong-api.log 2>&1 &
API_PID=$!
pnpm --filter @pong-pong/web start >/tmp/pong-web.log 2>&1 &
WEB_PID=$!

cleanup() {
  kill "$API_PID" "$WEB_PID" 2>/dev/null || true
  wait "$API_PID" "$WEB_PID" 2>/dev/null || true
}
trap cleanup EXIT
```

API는 source loader가 아니라 `apps/api/dist/index.js`로 시작한다. Web은 build된 `.next`를 `next start`로 제공한다. PID를 기록하고 `trap`에서 signal과 wait를 모두 수행하므로 test 성공·실패 어느 경로에서도 background process를 회수하려고 한다.

fixed sleep 대신 최대 60회 readiness polling을 사용한다.

```bash
for attempt in $(seq 1 60); do
  if curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null; then
    break
  fi
  sleep 1
done
curl --fail --silent http://127.0.0.1:4000/health/ready >/dev/null
```

Web에도 같은 구조를 적용한다. 마지막 `curl`이 필수이므로 loop가 끝날 때까지 ready가 아니면 다음 test로 진행하지 않고 job이 실패한다.

### 한 번의 lifecycle에서 세 관찰 층을 실행한다

```bash
pnpm smoke:http
pnpm smoke:ws
pnpm e2e
```

- HTTP smoke: API request/response와 health/session/domain route의 좁은 process boundary
- WS smoke: handshake와 protocol exchange
- browser E2E: Web UI·browser storage/cookie·HTTP/WS client·rendering flow

같은 API/Web process를 재사용하므로 앞 test의 state가 뒤 test에 영향을 줄 가능성은 있다. DB seed와 E2E identity isolation이 그 위험을 줄이지만, test마다 완전한 process/database reset을 제공하는 구조는 아니다.

### production artifact이지만 production mode는 아니다

run step의 environment는 다음을 사용한다.

```yaml
env:
  APP_MODE: development
  SESSION_SECRET: ci-process-session-secret-32-bytes
  LOG_LEVEL: warn
```

따라서 이 job이 검증하는 것은 다음이다.

- production compiler가 만든 API artifact가 Node에서 시작된다.
- Next production server가 build 결과를 제공한다.
- 실제 PostgreSQL과 migration/seed 뒤 HTTP·WS·browser flow가 동작한다.

반면 production-specific mode restriction, production Compose, API/Web Docker image, Caddy route, required Compose secret interpolation은 검증하지 않는다. 이름의 “production processes”는 **artifact/process 형태**를 뜻하며 전체 deployment mode를 뜻하지 않는다.

## `7cb0d32b5be3` — workflow 자체를 source contract로 검사하다

process job은 긴 YAML과 shell block에 의존한다. command 하나가 삭제되거나 Node version이 job마다 갈려도 YAML syntax는 유효할 수 있다. 새 test는 workflow text에서 선택된 requirement를 확인한다.

```js
const nodeVersions = [...workflow.matchAll(/node-version:\s*([^\s]+)/g)]
  .map((match) => match[1]);

assert.deepEqual([...new Set(nodeVersions)], ["24.18.0"]);
assert.match(workflow, /version: 10\.32\.1/);
assert.match(workflow, /pnpm install --frozen-lockfile/);
```

```js
for (const command of [
  "pnpm unit",
  "pnpm postgres-integration",
  "pnpm smoke:http",
  "pnpm smoke:ws",
  "pnpm e2e"
]) {
  assert.match(workflow, new RegExp(command.replace(":", "\\:")));
}

assert.match(workflow, /services:\s*\n\s+postgres:/);
assert.match(workflow, /pnpm --filter @pong-pong\/db migrate/);
assert.match(workflow, /pnpm --filter @pong-pong\/db seed:dev/);
```

이 test는 빠르고 deterministic하지만 source text 검사다.

- command가 comment나 잘못된 job에 있어도 regex가 match할 수 있다.
- YAML parser로 job semantics를 해석하지 않는다.
- GitHub Actions runner에서 action·service·shell이 실제 성공하는지 증명하지 않는다.
- 모든 job에 pnpm version이 같은지 구조적으로 세지 않고 text presence를 본다.

따라서 역할은 CI execution을 대체하는 것이 아니라, 중요한 command가 실수로 사라지는 것을 더 이른 unit-style test에서 감지하는 것이다.

## demo guest flow — `bf0bc1199c84`와 `0ae1ded2c56f`

registered-user E2E는 PostgreSQL·development login·seed account에 의존한다. 비회원 체험은 다른 runtime mode와 다른 data/auth boundary를 사용하므로 독립 job으로 분리된다.

```yaml
guest-demo-browser:
  name: Guest demo browser tests
  runs-on: ubuntu-latest
  timeout-minutes: 20
  env:
    APP_MODE: demo
    E2E_APP_MODE: demo
    E2E_BASE_URL: http://localhost:3000
    NEXT_PUBLIC_APP_MODE: demo
    NEXT_PUBLIC_API_BASE_URL: http://localhost:4000
    SESSION_SECRET: ci-guest-demo-session-secret-32-bytes
    LOG_LEVEL: warn
```

세 mode 값의 owner가 다르다.

| 변수 | consumer | 의미 |
| --- | --- | --- |
| `APP_MODE` | API runtime | demo auth/data policy |
| `NEXT_PUBLIC_APP_MODE` | Web build | browser bundle의 demo UI/behavior |
| `E2E_APP_MODE` | test harness/spec | expected flow와 skip/selection |

이 셋 중 하나만 다르면 API·browser UI·test expectation이 서로 다른 mode를 가정할 수 있다.

job은 Chromium과 dependencies를 설치하고 `pnpm build` 후 compiled API와 Next production server를 시작한다. PostgreSQL service나 `DATABASE_URL`은 없으므로 demo mode가 허용하는 memory repository 경로를 사용한다. readiness 뒤 `pnpm e2e:guest-demo`만 실행한다.

`0ae1ded2c56f`는 다음 source-level contract를 추가한다.

```js
assert.match(workflow, /guest-demo-browser:/);
assert.match(workflow, /APP_MODE:\s*demo/);
assert.match(workflow, /NEXT_PUBLIC_APP_MODE:\s*demo/);
assert.match(workflow, /E2E_APP_MODE:\s*demo/);
assert.match(workflow, /pnpm e2e:guest-demo/);
assert.match(workflow, /tests\/e2e\/guest-demo\.spec\.ts/);
```

이 test 역시 job 실행을 증명하지 않는다. dedicated job과 mode alignment를 source에서 보호한다.

## static contract를 root와 CI에 연결 — `f35728f4ef92`, `4f9e66c35586`

CI/Docker/load configuration test가 개별 파일로 존재해도 호출자가 모르면 실행되지 않는다. root package와 Makefile에 명시적 command가 생긴다.

```json
"test:contracts": "node --test \
  tests/ci-contract.test.mjs \
  tests/docker-production.test.mjs \
  tests/load/fault-scenario.test.mjs \
  tests/load/load-harness.test.mjs"
```

```make
contracts:
	pnpm test:contracts
```

이 commit에는 load 관련 contract도 함께 연결되지만, 그 내용은 runtime/load architecture Thread의 관심사다. 여기서 중요한 것은 configuration contract가 root verification graph의 named node가 됐다는 점이다.

다음 commit은 verify job에서 unit 뒤, build 전에 이를 실행한다.

```yaml
- name: Run unit tests
  run: pnpm unit

- name: Run static contract tests
  run: pnpm test:contracts

- name: Build
  run: pnpm build
```

이 순서의 장점은 Docker/CI configuration drift처럼 build artifact와 무관한 오류를 expensive build보다 먼저 실패시키는 것이다. 다만 `docker-production.test.mjs`는 `docker compose config` command를 호출하므로 “static”이라는 이름과 달리 runner에 Docker Compose CLI가 필요하다. image/container를 시작하지 않을 뿐 외부 executable dependency는 있다.

## `65512bc24161` — 같은 loopback이라도 cookie host는 같지 않다

registered process job의 browser base URL은 Playwright config 기본값인 `http://localhost:3000`이다.

```ts
use: {
  baseURL: process.env.E2E_BASE_URL ?? "http://localhost:3000"
}
```

E2E suite는 login을 Web UI에서 수행한 뒤 일부 domain action을 Playwright의 page-bound request context로 direct API에 보낸다.

```ts
const apiBase = process.env.API_BASE_URL ?? "http://localhost:4000";

async function login(page, handle, displayName) {
  await page.goto("/");
  /* form 입력 */
  await page.getByRole("button", { name: "개발 로그인" }).click();
}

// 뒤의 test에서 같은 page context로 direct API 호출
const created = await page.request.post(`${apiBase}/tournaments`, { /* ... */ });
```

초기 workflow는 `API_BASE_URL=http://127.0.0.1:4000`이었다. `localhost`와 `127.0.0.1`은 같은 machine의 loopback을 가리킬 수 있지만 cookie의 host matching에서는 다른 host다. UI login response가 `localhost` host의 session cookie를 만들었다면, `page.request`가 `127.0.0.1`로 보내는 request에는 그 host-only cookie가 포함되지 않는다. 결과는 network failure가 아니라 authenticated request가 unauthenticated로 보이는 failure다.

fix는 HTTP API base만 바꾼다.

```diff
-API_BASE_URL: http://127.0.0.1:4000
+API_BASE_URL: http://localhost:4000
 WS_URL: ws://127.0.0.1:4000/ws
```

WebSocket URL은 그대로다. 따라서 이 commit의 범위는 generic loopback normalization이나 WS origin이 아니라 **registered browser E2E의 HTTP session cookie host**다.

## `527921bc9d69` — cookie origin을 workflow contract로 고정하다

후속 test는 workflow의 exact line을 요구한다.

```js
test("CI keeps registered browser API requests on the login cookie host", () => {
  assert.match(
    workflow,
    /^      API_BASE_URL: http:\/\/localhost:4000$/m
  );
});
```

이 regression은 hostname이 다시 `127.0.0.1`로 돌아가는 것을 빠르게 잡는다. 다만 login cookie의 attributes, `SameSite`, `Secure`, domain, browser storage state를 직접 검사하지 않는다. 원인에 대응하는 configuration literal을 보호하는 narrow test다.

## 최종 CI verification matrix

| job/단계 | 실제로 만드는 resource | 실행 mode | 주요 증거 | 증명하지 않는 것 |
| --- | --- | --- | --- | --- |
| `verify` typecheck/unit | Node process만 | package별 test mode | type contract·unit behavior | DB/process/browser |
| `verify` contracts | source parse + `docker compose config` | 없음 | selected workflow/Docker/config invariant | Actions/Docker runtime |
| `verify` build/artifact | emitted files | build-time | production files 존재 | process start·artifact promotion |
| `postgres-integration` | Testcontainers PostgreSQL | test | actual DB migration/repository suite | API/Web integration |
| `process-and-browser` | workflow PostgreSQL + compiled API + Next server + Chromium | `APP_MODE=development` | HTTP·WS·registered browser flow | Docker images·Caddy·production mode |
| `guest-demo-browser` | compiled API + Next server + Chromium, memory repository | `demo` | guest/demo browser flow | durable DB·registered auth |

## Thread의 최종 불변 조건

- 검증 종류는 root command로 명시되어 CI와 local caller가 같은 entry를 사용한다.
- unit, PostgreSQL integration, compiled process/browser는 resource cost와 failure meaning에 따라 job을 분리한다.
- process job은 build output을 직접 시작하고 PID를 trap으로 회수한다.
- test는 readiness를 확인한 뒤 traffic을 보내며 fixed startup sleep만 믿지 않는다.
- registered/development와 guest/demo browser path는 mode와 storage dependency가 달라 별도 job이다.
- CI/Docker configuration의 선택된 requirement는 static contract에서도 보호된다.
- browser login host와 direct authenticated API host는 `localhost`로 일치한다.

동시에 다음은 이 Thread 밖에 남는다.

- production Compose/Caddy/image를 실제로 띄우는 end-to-end deployment test
- CI artifact upload·promotion·release approval
- browser failure 시 API/Web log 자동 첨부
- workflow YAML의 완전한 semantic schema 검사
- every cookie attribute와 cross-origin policy 검증
- flaky retry·test sharding·parallel data isolation 정책

## 조사 범위

각 workflow change와 test contract는 exact SHA diff에서 확인했다. PostgreSQL job은 해당 시점의 package script와 Testcontainers suite를, cookie fix는 Playwright base URL과 E2E direct request path를 함께 확인했다. GitHub Actions job·PostgreSQL container·API/Web process·Chromium은 이 작업 환경에서 직접 실행하지 않았다.
