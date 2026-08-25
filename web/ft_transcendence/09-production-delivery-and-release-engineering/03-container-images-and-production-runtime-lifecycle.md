# 개발 Thread 03 — compiled image에서 migration·readiness·종료 유예까지

## 개요

Thread 02가 JavaScript·SQL·Next.js standalone 산출물을 만들었다면, 이 Thread는 그 산출물을 **source tree 없이 실행할 image**로 옮기고 여러 container의 lifecycle을 하나의 production graph로 조립한다.

최종 목표는 단순히 `docker compose up`이 성공하는 것이 아니다.

- API와 Web은 build stage에서 만들어진 artifact를 runner가 실행한다.
- runtime process는 root가 아닌 `node` user로 실행한다.
- DB schema migration이 성공하기 전에는 API가 시작되지 않는다.
- API가 ready가 되기 전에는 Web이, API와 Web이 ready가 되기 전에는 Caddy가 공개되지 않는다.
- host에 publish되는 port는 Caddy 하나뿐이다.
- production secret이 없으면 Compose interpolation 단계에서 시작 자체가 실패한다.
- API가 SIGTERM을 받은 뒤 사용하는 60초 room-drain budget보다 container의 강제 종료 유예가 길다.

이 Thread의 정적 contract test는 이 구조의 일부를 source와 rendered Compose에서 검사하지만, 실제 image build·container start·signal 전달까지 수행하지는 않는다. 따라서 **구성 계약**과 **runtime 실행 증거**를 구분해 읽어야 한다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f8efb2656771` | `build(docker): production API image 구성` | B | `PERSISTENCE, OPERATIONS` | shared·DB·API를 build하고 compiled API를 non-root runner에서 실행한다. |
| 2 | `656893e8e1cb` | `build(docker): production Web image 구성` | B | `REALTIME, WEB, OPERATIONS` | public config를 build-time에 주입해 Next standalone runner를 만든다. |
| 3 | `2c44cb7cd71f` | `build(docker): production container lifecycle 구성` | A | `PERSISTENCE, OPERATIONS` | source-mounted Compose를 image·migration·health·single gateway graph로 교체한다. |
| 4 | `e2c12ded1d5f` | `test(docker): production container contract 검증` | B | `OPERATIONS, OBSERVABILITY, TEST` | rendered Compose와 Dockerfile/Caddyfile의 선택된 정적 규칙을 검사한다. |
| 5 | `312ddbc6fbe2` | `fix(runtime): container 종료 유예를 room drain과 정렬` | A | `REALTIME, OPERATIONS, RISK` | API의 stop grace를 application의 60초 drain보다 길게 만든다. |
| 6 | `73ba979841cd` | `test(docker): API 종료 유예 계약 검증` | B | `OPERATIONS, TEST` | rendered stop grace를 초로 환산해 60초 이상인지 검사한다. |

## API image — `f8efb2656771`

### build context를 먼저 줄인다

같은 commit에서 `.dockerignore`가 추가된다.

```dockerignore
.git
.github
node_modules
**/node_modules
**/.next
**/dist
coverage
playwright-report
test-results
output
.env
.env.*
*.log
README.md
```

host의 dependency/build output·secret file이 context에 들어가지 않으므로 image build는 repository source와 lockfile에서 다시 artifact를 만든다. 특히 `.env*`를 제외해 local secret이 image layer로 복사되는 위험을 줄인다.

이 파일이 보장하지 않는 것도 있다. Dockerfile에서 별도로 `COPY`한 source 안에 secret이 있거나 build argument로 secret을 넘기면 여전히 layer/history에 남을 수 있다. `.dockerignore`는 secret manager가 아니다.

### 세 stage가 서로 다른 책임을 갖는다

```dockerfile
FROM node:24.18.0-bookworm-slim AS dependencies

RUN corepack enable && corepack prepare pnpm@10.32.1 --activate
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/api/package.json apps/api/package.json
COPY packages/db/package.json packages/db/package.json
COPY packages/shared/package.json packages/shared/package.json
RUN pnpm install --frozen-lockfile

FROM dependencies AS builder

COPY tsconfig.base.json ./
COPY apps/api apps/api
COPY packages/db packages/db
COPY packages/shared packages/shared
RUN pnpm --filter @pong-pong/shared build \
  && pnpm --filter @pong-pong/db build \
  && pnpm --filter @pong-pong/api build

FROM node:24.18.0-bookworm-slim AS runner

ENV NODE_ENV=production
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/apps/api/node_modules ./apps/api/node_modules
COPY --from=builder /app/apps/api/dist ./apps/api/dist
COPY --from=builder /app/packages/db/node_modules ./packages/db/node_modules
COPY --from=builder /app/packages/db/dist ./packages/db/dist
COPY --from=builder /app/packages/shared/node_modules ./packages/shared/node_modules
COPY --from=builder /app/packages/shared/dist ./packages/shared/dist
USER node

CMD ["node", "apps/api/dist/index.js"]
```

각 stage의 역할은 명확하다.

| stage | 입력 | 산출·실행 책임 |
| --- | --- | --- |
| `dependencies` | workspace manifests + lockfile | frozen dependency graph 설치, source 변경과 dependency cache 분리 |
| `builder` | API/DB/shared source | Thread 02의 order대로 compiled artifact 생성 |
| `runner` | builder의 runtime files | `node apps/api/dist/index.js` 실행 |

`USER node`와 exec-form `CMD`는 application process가 non-root로 실행되고 signal을 직접 받을 수 있는 기반을 만든다. runtime command에는 `pnpm install`, `tsx`, `tsc`나 shell chain이 없다.

### “최소 production dependency image”라고까지 말할 수 없는 이유

runner는 builder의 root·package `node_modules`를 그대로 복사한다. production-only prune을 수행하지 않는다. 따라서 dev dependency와 build tool이 실제로 남는지는 pnpm layout과 copied tree에 따라 달라지며, 이 Dockerfile만 보고 “production dependency만 포함한다”고 단정할 수 없다.

이 commit이 확실히 제거한 것은 **startup install과 source interpreter**다. image 크기 최소화나 SBOM·package pruning은 별도 문제다.

## Web image — `656893e8e1cb`

Web의 public endpoint 값은 server start 때가 아니라 `next build` 때 browser bundle에 들어간다. Dockerfile은 이 lifetime을 `ARG → ENV → build`로 표현한다.

```dockerfile
FROM node:24.18.0-bookworm-slim AS builder

ARG NEXT_PUBLIC_API_BASE_URL=http://localhost:8080/api
ARG NEXT_PUBLIC_WS_URL=ws://localhost:8080/ws
ARG NEXT_PUBLIC_APP_MODE=production
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL
ENV NEXT_PUBLIC_WS_URL=$NEXT_PUBLIC_WS_URL
ENV NEXT_PUBLIC_APP_MODE=$NEXT_PUBLIC_APP_MODE

COPY tsconfig.base.json ./
COPY apps/web apps/web
COPY packages/shared packages/shared
RUN pnpm --filter @pong-pong/shared build \
  && pnpm --filter @pong-pong/web build
```

runner는 전체 workspace나 root `node_modules`를 복사하지 않고 Next standalone 결과와 static asset만 가져간다.

```dockerfile
FROM node:24.18.0-bookworm-slim AS runner

ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3000
WORKDIR /app
USER node
COPY --from=builder --chown=node:node /app/apps/web/.next/standalone ./
COPY --from=builder --chown=node:node /app/apps/web/.next/static ./apps/web/.next/static

CMD ["node", "apps/web/server.js"]
```

이 image의 핵심 contract는 다음과 같다.

- `NEXT_PUBLIC_*` 값은 image build 입력이다. 같은 image를 시작하면서 runtime env만 바꿔도 browser bundle은 바뀌지 않는다.
- server entry는 Thread 02의 `output: "standalone"`이 만들어야 한다.
- `.next/static`은 standalone root와 별도로 복사해야 browser asset이 제공된다.
- copied files는 `node:node` ownership으로 맞춰지고 process도 `node` user로 실행된다.

public URL에 secret을 넣어서는 안 된다. `NEXT_PUBLIC_*`는 browser에 공개되는 build-time configuration이다.

## `2c44cb7cd71f` — image들을 fail-closed startup graph로 조립하다

이 commit은 기존 source-mounted Compose를 production image composition으로 교체한다. 변화의 핵심은 container 수가 아니라 **dependency condition과 공개 시점**이다.

### 1. secret은 container 생성 전에 요구된다

```yaml
db:
  environment:
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}

api:
  environment:
    DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}@db:5432/pong_pong
    SESSION_SECRET: ${SESSION_SECRET:?SESSION_SECRET must be set}
```

`${VAR:?message}`는 Compose interpolation 때 평가된다. 값이 없으면 placeholder를 default credential로 대체하지 않고 configuration rendering 자체를 실패시킨다. 이는 application 내부 validation보다 더 이른 fail-closed boundary다.

다만 secret의 길이·형식·rotation은 Compose가 검증하지 않는다. API의 environment parser가 session secret 길이를 추가로 검사한다.

### 2. migration은 API와 분리된 one-shot process가 된다

```yaml
migrate:
  image: pong-pong-api:local
  build:
    context: .
    dockerfile: apps/api/Dockerfile
  command: ["node", "packages/db/dist/cli.js", "migrate"]
  environment:
    DATABASE_URL: postgres://pong:${POSTGRES_PASSWORD:?POSTGRES_PASSWORD must be set}@db:5432/pong_pong
  depends_on:
    db:
      condition: service_healthy
  restart: "no"
```

API image를 재사용하므로 migration CLI와 SQL asset은 application과 같은 build revision이다. `restart: "no"`는 migration을 장기 service가 아닌 startup transaction으로 취급한다.

Thread 02의 `430389...` 당시에는 compiled migrator의 asset path가 아직 불완전했지만, 이 exact SHA에는 persistence Thread의 `30aac132e14e`가 이미 포함되어 있다. migrator는 먼저 emitted file 옆의 `./migrations`, 즉 `packages/db/dist/migrations`를 찾고 source 실행을 위해 `../migrations`를 fallback으로 사용한다. API Dockerfile이 `packages/db/dist`를 복사하므로 one-shot container가 bundled SQL을 찾을 수 있는 구조가 이 시점에는 성립한다.

### 3. health와 completion이 공개 순서를 결정한다

```yaml
api:
  depends_on:
    migrate:
      condition: service_completed_successfully
  healthcheck:
    test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:4000/health/ready').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
  expose:
    - "4000"

web:
  depends_on:
    api:
      condition: service_healthy
  healthcheck:
    test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
  expose:
    - "3000"

caddy:
  build:
    context: .
    dockerfile: Caddy.Dockerfile
  depends_on:
    api:
      condition: service_healthy
    web:
      condition: service_healthy
  ports:
    - "8080:8080"
```

startup graph는 다음과 같다.

```text
Compose interpolation에서 secret 검사
   ↓
PostgreSQL start + pg_isready
   ↓ service_healthy
migrate: compiled CLI 실행
   ↓ exit 0 / service_completed_successfully
API start
   ↓ /health/ready 2xx
Web start
   ↓ / 2xx
Caddy start + host :8080 publish
```

migration이 non-zero면 API는 시작 조건을 충족하지 못한다. API가 ready가 아니면 Web과 Caddy가 gate 뒤에 남고, Web이 healthy가 아니어도 Caddy가 시작되지 않는다.

### 4. network exposure와 storage ownership이 정리된다

- DB·API·Web은 `ports`가 아니라 internal `expose` 또는 기본 Compose network만 사용한다.
- host에 publish되는 port는 Caddy `8080` 하나다.
- source·root node_modules·`.next` bind/named volume은 제거된다.
- durable data를 갖는 named volume은 PostgreSQL data volume만 남는다.

이 구조가 보장하지 않는 것도 중요하다.

- migration rollback·backup restore·partially applied SQL 복구
- TLS·public domain·certificate lifecycle
- secret manager·runtime secret file·rotation
- image registry, immutable digest, signature
- 다중 replica에서 migration singleton을 보장하는 orchestrator 수준의 coordination
- health endpoint가 모든 실제 traffic 조건을 완전히 대표한다는 보장

이 commit은 single-host Compose production contract다.

## `e2c12ded1d5f` — 실행하지 않고도 구조 회귀를 잡는 static contract

새 Node test는 required secret 값을 넣고 `docker compose config --format json`을 실행해 interpolation·normalization된 model을 읽는다.

```js
const result = spawnSync(
  "docker",
  ["compose", "config", "--format", "json"],
  {
    cwd: root,
    encoding: "utf8",
    env: {
      ...process.env,
      POSTGRES_PASSWORD: "compose-contract-password",
      SESSION_SECRET: "compose-contract-session-secret-32-bytes"
    }
  }
);

const services = JSON.parse(result.stdout).services;
```

실제 assertion은 다음 범위를 가진다.

- service 집합이 `api`, `caddy`, `db`, `migrate`, `web`인지
- `ports`를 가진 service가 Caddy 하나인지
- migrate command가 compiled DB CLI인지
- migrate가 `restart: "no"`인지
- API가 migration의 successful completion에 의존하는지
- 어느 service에도 bind mount가 없는지
- API/Web Dockerfile이 exact Node base와 `USER node`를 갖는지
- API/Web `CMD`가 `pnpm`/`npm`을 직접 실행하지 않는지
- Compose source에 required password/session placeholders와 readiness path가 있는지
- Caddyfile이 `/api/metrics`를 404로 차단하는지

```js
assert.deepEqual(
  Object.entries(services)
    .filter(([, service]) => Array.isArray(service.ports) && service.ports.length > 0)
    .map(([name]) => name),
  ["caddy"]
);

assert.deepEqual(
  services.migrate.command,
  ["node", "packages/db/dist/cli.js", "migrate"]
);
assert.equal(
  services.api.depends_on.migrate.condition,
  "service_completed_successfully"
);
```

### 실제 source와 구분해야 할 점

이 commit의 test는 secret을 누락한 별도 render case를 실행하지 않는다. required interpolation syntax를 source text regex로 확인하고, 정상 render에는 fake secret을 제공한다. 또한 Web→API·Caddy→API/Web dependency condition 전체를 모두 assert하지는 않는다.

따라서 이 test가 증명하는 것은 **선택된 configuration property가 파일과 rendered model에 존재한다는 것**이다. 다음은 증명하지 않는다.

- Dockerfile이 실제로 build되는지
- migration container가 DB에 연결하고 종료하는지
- healthcheck가 성공하는지
- Caddy가 실제 network traffic을 전달하는지
- non-root process가 필요한 file을 읽을 수 있는지
- container stop signal과 cleanup이 동작하는지

정적 contract는 runtime integration test를 대체하지 않고, expensive runtime 이전에 명백한 배포 구조 drift를 빠르게 잡는다.

## `312ddbc6fbe2` — application cleanup budget보다 짧은 container timeout을 허용하지 않다

startup graph가 완성되어도 stop graph가 application과 맞지 않으면 active game room을 정리하는 중에 process가 강제 종료될 수 있다. exact SHA의 API signal path는 다음 순서다.

```ts
installGracefulShutdown(
  process,
  async (signal) => {
    app.log.info({ signal }, "graceful shutdown started");
    const result = await app.beginDrain(60_000);
    app.log.info(result, "game room drain finished");
    await app.close();
  },
  /* failure callback */
);
```

`beginDrain(60_000)`은 새 queue admission을 막고 active room이 끝나기를 기다리되 60초를 넘기지 않는다. drain test도 active room이 59,999ms에는 남고 60,000ms에 timeout result를 반환하는 경계를 고정한다.

Compose에는 API service의 외부 lifetime budget이 추가된다.

```diff
 api:
   expose:
     - "4000"
+  stop_grace_period: 70s
```

이제 정상적인 stop 순서는 다음과 같이 맞춰진다.

```text
Compose stop
  → API process에 SIGTERM
  → admission close + room drain (최대 60초)
  → app.close()와 repository/socket cleanup
  → process exit

Compose는 최대 70초까지 기다린 뒤에만 강제 종료 가능
```

70초와 60초 사이의 10초는 drain 이후 server/repository close와 scheduling을 위한 headroom으로 해석할 수 있다. 그러나 commit에는 실제 shutdown duration 측정이나 “10초가 충분하다”는 benchmark가 없다. 확정된 사실은 **configured grace가 known drain timeout보다 길다**는 것뿐이다.

이 fix가 막지 못하는 경로도 있다.

- process crash·OOM·host failure처럼 SIGTERM handler가 실행되지 않는 종료
- room drain이 60초 timeout 뒤에도 application resource를 남기는 bug
- `app.close()`가 10초를 넘겨 전체 70초를 소진하는 경우
- Docker/Compose 외 orchestrator가 다른 termination policy를 적용하는 경우

## `73ba979841cd` — cross-layer timeout relation을 정적 contract로 고정하다

기존 Docker contract test는 `stop_grace_period`를 보지 않았다. 후속 YAML 변경이 70초를 삭제하거나 30초로 줄여도 test가 통과할 수 있었다. 이 commit은 rendered API service의 duration을 파싱해 최소 60초를 요구한다.

```js
assert.ok(parseDurationSeconds(services.api.stop_grace_period) >= 60);

function parseDurationSeconds(value) {
  assert.equal(typeof value, "string");
  const match = /^(?:(\d+)m)?(?:(\d+)s)?$/.exec(value);
  assert.ok(match, `unsupported duration: ${value}`);
  return Number(match[1] ?? 0) * 60 + Number(match[2] ?? 0);
}
```

이 assertion은 정확히 `70s`만 허용하지 않는다. `60s`, `1m`, `1m10s`처럼 parser가 이해하는 형식 중 60초 이상이면 contract를 만족한다. 즉 test가 보호하는 것은 구현 literal이 아니라 **application drain budget과 container grace의 관계**다.

반대로 signal을 보내거나 active room을 만든 뒤 container가 60초 동안 기다리는 것은 관찰하지 않는다. 실제 graceful shutdown test가 아니라 rendered configuration invariant다.

## 최종 lifecycle

### startup

| 단계 | owner | 성공 조건 | 실패 시 차단되는 다음 단계 |
| --- | --- | --- | --- |
| secret interpolation | Compose | required env 존재 | 모든 container 생성 |
| DB | PostgreSQL container + volume | `pg_isready` | migration |
| migration | one-shot API image process | compiled CLI exit 0 | API |
| API | compiled non-root process | `/health/ready` 성공 | Web·Caddy |
| Web | standalone non-root process | `/` 성공 | Caddy |
| gateway | Caddy image | process start | host traffic |

### shutdown

| 단계 | owner | budget/행동 |
| --- | --- | --- |
| stop signal | Compose/Docker | API에 SIGTERM 전달 |
| admission·room drain | API | 최대 60초 |
| server/repository close | API | drain 뒤 남은 grace에서 수행 |
| 강제 종료 상한 | Compose/Docker | 총 70초 후 강제 종료 가능 |

### 최종 보장과 비보장

| 보장 | 비보장 |
| --- | --- |
| source mount·startup install 없이 built process 실행 | API image에서 dev dependency가 완전히 제거됨 |
| DB migration completion 뒤 API start | migration rollback·distributed singleton |
| API/Web health 뒤 gateway 공개 | health가 실제 모든 기능을 대표함 |
| host에는 Caddy port만 publish | TLS·WAF·secret manager |
| required secret 누락 시 interpolation failure | secret 값의 rotation·강도 전체 |
| container grace ≥ application drain timeout | 실제 signal/drain/close가 70초 안에 성공함 |
| selected rule의 static contract test | image build·container integration 실행 |

## 조사 범위

각 Dockerfile·Compose·contract test는 표시된 exact SHA의 diff와 source에서 확인했다. 60초 drain은 `312ddbc6fbe2` 시점의 `apps/api/src/index.ts`와 drain test를 대조했다. image build, `docker compose up`, migration, healthcheck, signal 종료는 이 작업 환경에서 실행하지 않았으므로 실행 성공을 주장하지 않는다.
