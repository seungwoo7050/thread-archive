# 개발 Thread 01 — 개발용 Compose에서 단일 진입점과 빌드 기반 실행으로

## 개요

이 Thread는 API·Web·PostgreSQL을 한 번에 띄우는 개발용 Compose가 생긴 뒤, 브라우저가 하나의 주소로 HTTP API·WebSocket·Web 화면을 소비하도록 Caddy 경계를 만들고, Web을 개발 서버가 아닌 `next build` 결과로 실행하도록 바뀌는 과정을 다룬다.

여기서 중요한 것은 **이 네 commit만으로 production 배포가 완성되지는 않는다는 점**이다. 마지막 commit에서도 API와 Web은 source bind mount 위에서 시작할 때 의존성을 설치하고, API는 compiled JavaScript가 아니라 `tsx src/index.ts`를 실행한다. 이 Thread가 확립한 것은 다음의 중간 계약이다.

- 브라우저의 외부 진입점은 Caddy `:8080`으로 통합할 수 있다.
- `/api/*`, `/ws`, 나머지 Web 요청은 서로 다른 upstream으로 분리된다.
- Web의 production server는 미리 만들어진 `.next` 결과를 소비해야 한다.
- `/api/metrics`는 일반 API proxy보다 먼저 차단해 외부 경로에서 노출하지 않는다.
- Caddy 설정은 bind-mounted 파일뿐 아니라 image에 포함할 수 있는 artifact가 된다.

source mount, startup install/build, 내부 서비스의 host port 공개를 제거하고 one-shot migration·health gate를 연결하는 작업은 [Thread 03](03-container-images-and-production-runtime-lifecycle.md)에서 이어진다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `19b4d9f9083d` | `build(runtime): Compose와 Caddy 라우팅 추가` | B | `REALTIME, PERSISTENCE, WEB` | 개발용 다중 서비스 구성과 브라우저 단일 진입점을 처음 만든다. |
| 2 | `ec000bed0414` | `build(web): production start와 TS cache 정책 구성` | B | `WEB` | Web에 `next start` 명령을 만들고 증분 TypeScript cache를 끈다. |
| 3 | `be15e937d718` | `fix(runtime): Compose에서 build 결과 실행` | B | `WEB, OPERATIONS` | Compose의 Web을 `build → start`로 바꾸고 API는 `dev` 대신 기존 `start` script를 사용한다. |
| 4 | `576eb97f8041` | `build(docker): Caddy reverse proxy 구성` | B | `REALTIME, OPERATIONS, OBSERVABILITY` | Caddy 설정을 image artifact로 만들고 public `/api/metrics`를 404로 닫는다. |

## `19b4d9f9083d` — 네 개의 process와 하나의 브라우저 주소

첫 구성은 production topology가 아니라 **로컬 통합 실행 환경**이다. PostgreSQL은 named volume에 데이터를 남기고, API와 Web은 repository root를 `/app`에 bind mount한 Node image 안에서 매번 `pnpm install` 후 개발 서버를 실행한다.

```yaml
# docker-compose.yml — 핵심 구조
services:
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    volumes:
      - pong-pong-db:/var/lib/postgresql/data

  api:
    image: node:23-bookworm-slim
    working_dir: /app
    command: >-
      sh -c "corepack enable &&
      corepack prepare pnpm@10.32.1 --activate &&
      pnpm install --frozen-lockfile &&
      pnpm --filter @pong-pong/api dev"
    volumes:
      - .:/app
      - api-node-modules:/app/node_modules
    ports:
      - "4000:4000"

  web:
    image: node:23-bookworm-slim
    working_dir: /app
    command: >-
      sh -c "corepack enable &&
      corepack prepare pnpm@10.32.1 --activate &&
      pnpm install --frozen-lockfile &&
      pnpm --filter @pong-pong/web dev"
    volumes:
      - .:/app
      - web-node-modules:/app/node_modules
    ports:
      - "3000:3000"

  caddy:
    image: caddy:2-alpine
    ports:
      - "8080:8080"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
```

Caddy는 path에 따라 upstream을 선택한다.

```caddyfile
:8080 {
	handle_path /api/* {
		reverse_proxy api:4000
	}

	handle /ws {
		reverse_proxy api:4000
	}

	handle {
		reverse_proxy web:3000
	}
}
```

이 배치가 만드는 경계는 다음과 같다.

| 요청 | Caddy의 처리 | upstream에 전달되는 핵심 path |
| --- | --- | --- |
| `/api/users` | `handle_path /api/*` | `/users` — `/api` prefix를 제거 |
| `/ws` | `handle /ws` | `/ws` 그대로 전달 |
| `/`, `/profile/...` | fallback `handle` | Web server로 전달 |

따라서 브라우저는 `http://localhost:8080`이라는 하나의 origin을 사용할 수 있고, API 내부 route는 `/api` prefix를 몰라도 된다. WebSocket도 같은 gateway를 통과한다.

다만 이 시점의 보장은 제한적이다.

- DB·API·Web·Caddy가 모두 host port를 가진다. Caddy는 유일한 외부 경계가 아니다.
- API는 DB의 healthcheck를 기다리지만, Web과 Caddy의 `depends_on`은 upstream이 실제 요청을 받을 준비가 됐다는 뜻이 아니다.
- 고정된 `pong` password와 개발용 session secret을 사용한다.
- source code와 package installation이 startup 입력이다. image 자체만으로 같은 process를 재현할 수 없다.
- Caddy 설정은 host file의 현재 내용에 의존한다.

즉 이 commit의 핵심 성과는 “배포 image”가 아니라 **다중 process를 한 origin으로 조립하는 route graph**다.

## `ec000bed0414` — Web server가 빌드 결과를 소비할 명령을 얻다

Web package에는 `next build`는 있었지만 production server를 시작하는 script가 없었다. 이 commit은 다음 script를 추가한다.

```diff
 "scripts": {
   "dev": "next dev --hostname 0.0.0.0 --port 3000",
   "build": "next build",
+  "start": "next start --hostname 0.0.0.0 --port 3000",
   "typecheck": "tsc --noEmit"
 }
```

`next start`는 source compiler를 겸하는 개발 서버가 아니다. 같은 working directory에 존재하는 `.next` production build를 읽어 HTTP server를 연다. 따라서 이 script 하나만으로는 충분하지 않고, caller가 먼저 `next build`를 성공시켜야 한다. 바로 다음 commit이 Compose에서 그 producer/consumer 순서를 연결한다.

같은 commit에서 Web `tsconfig.json`의 증분 compiler cache도 끈다.

```diff
-"incremental": true,
+"incremental": false,
```

이 변경은 Web runtime 기능을 바꾸지 않는다. typecheck가 과거 `.tsbuildinfo` cache에 의존하지 않도록 해, clean checkout·CI·container처럼 매 실행이 독립적이어야 하는 환경의 결과를 단순하게 만든다.

> 같은 diff에는 `vitest run --passWithNoTests` script도 추가되지만, Web production start와 TypeScript cache가 이 Thread의 주제이므로 여기서는 다루지 않는다.

## `be15e937d718` — Compose가 개발 server 대신 start 경계를 사용하다

Compose의 두 application command가 바뀐다.

```diff
- pnpm --filter @pong-pong/api dev
+ pnpm --filter @pong-pong/api start

- pnpm --filter @pong-pong/web dev
+ pnpm --filter @pong-pong/web build && pnpm --filter @pong-pong/web start
```

Web에는 `.next` 결과를 담을 named volume도 추가된다.

```yaml
web:
  volumes:
    - .:/app
    - web-node-modules:/app/node_modules
    - web-next:/app/apps/web/.next
```

### Web: producer와 consumer가 처음 연결된다

Web command는 같은 container startup 안에서 다음 순서로 동작한다.

```text
frozen install
  → next build       # .next producer
  → next start       # .next consumer
```

`.next`를 별도 volume에 두면 host의 source bind mount와 build output의 lifecycle을 분리할 수 있다. 하지만 image build 때 artifact를 고정하는 구조는 아니다. container를 새로 시작할 때마다 install과 build가 다시 실행되며, 결과는 source와 network/package cache 상태에 영향을 받는다.

### API: 이름은 `start`지만 아직 compiled artifact가 아니다

commit 제목만 보면 API도 “build 결과”를 실행하는 것처럼 읽힐 수 있으나, exact SHA의 API package script는 다음과 같다.

```json
{
  "scripts": {
    "build": "tsc --noEmit",
    "dev": "tsx watch src/index.ts",
    "start": "tsx src/index.ts"
  }
}
```

따라서 API에서 달라진 것은 watch mode를 제거한 것뿐이다. `start`는 여전히 TypeScript source와 `tsx`를 runtime dependency로 사용하고, `build`도 파일을 emit하지 않는다. API가 `node dist/index.js`를 실행하는 compiled boundary는 [Thread 02](02-production-build-and-package-artifacts.md)의 `bb67a72882bf`에서 만들어진다.

이 commit 뒤에도 남는 production 격차는 명확하다.

- application source가 bind mount되어 있다.
- startup 때 package install과 Web build를 수행한다.
- API/Web/DB port가 host에 직접 노출된다.
- API는 source interpreter로 시작한다.
- migration과 readiness가 application startup 전제 조건으로 연결되어 있지 않다.

따라서 이 commit은 **개발 server를 종료하고 Web build를 실제로 소비하는 중간 단계**이지, immutable production runtime의 완성은 아니다.

## `576eb97f8041` — gateway 설정을 image에 넣고 metrics를 public route에서 차단하다

Caddy 설정을 image에 포함할 수 있는 Dockerfile이 생긴다.

```dockerfile
FROM caddy:2-alpine

COPY Caddyfile /etc/caddy/Caddyfile
```

설정도 `route` block으로 재구성하고 `/api/metrics`를 API proxy보다 먼저 처리한다.

```caddyfile
:8080 {
	route {
		@internalMetrics path /api/metrics
		respond @internalMetrics 404

		handle_path /api/* {
			reverse_proxy api:4000
		}

		handle /ws {
			reverse_proxy api:4000
		}

		handle {
			reverse_proxy web:3000
		}
	}
}
```

라우팅 순서가 핵심이다. `/api/metrics`는 일반 `/api/*` matcher에도 들어가므로, 차단 rule이 뒤에 있으면 API로 proxy된다. 이 commit은 deny rule을 먼저 배치해 public gateway에서 404로 끝낸다. metric endpoint 자체의 생성·내용·내부 수집 방식은 observability Thread의 관심사이며, 여기서는 **gateway 노출 정책**만 다룬다.

`Caddy.Dockerfile`은 host의 설정 파일을 runtime에 mount하지 않고 image build 시점의 설정을 고정할 수 있게 한다. 그러나 이 exact commit은 `docker-compose.yml`을 바꾸지 않는다. 따라서 image artifact는 생겼지만, Compose가 실제로 이를 build·실행하는 consumer는 아직 없다. 그 연결은 `2c44cb7cd71f`에서 이뤄진다.

## 이 Thread가 남긴 최종 상태

```text
Browser :8080
   │
   ├─ /api/metrics ──> 404
   ├─ /api/* ────────> API :4000  (/api prefix 제거)
   ├─ /ws ───────────> API :4000  (path 유지)
   └─ 그 밖의 요청 ──> Web :3000
```

| 관심사 | 마지막 commit 기준 상태 | 아직 남은 문제 |
| --- | --- | --- |
| 브라우저 진입점 | Caddy route graph로 통합 | API/Web/DB host port도 여전히 공개 |
| Web 실행 | startup에서 `build → next start` | build가 image가 아니라 container startup에 종속 |
| API 실행 | `dev` 대신 `start` | `start`가 `tsx src/index.ts`; emitted artifact 아님 |
| Caddy 설정 | image에 COPY 가능 | Compose가 아직 이 image를 사용하지 않음 |
| metrics 노출 | public `/api/metrics`는 404 | 내부 인증·수집 topology는 이 Thread 밖 |
| 재현성 | frozen lockfile install | startup network/install/source mount 의존 |

이 Thread의 끝에서 “어디로 요청을 보내고 어떤 process mode로 시작할지”는 정리되었다. 하지만 **무엇을 빌드 산출물로 배포할지**, **어떤 image가 그 산출물을 소유할지**, **어떤 순서와 health condition으로 공개할지**는 아직 별개의 문제다. 그 세 문제를 각각 Thread 02와 Thread 03이 이어받는다.

## 조사 범위

각 설명은 표시된 SHA의 diff와 해당 SHA에서 필요한 package/config source를 기준으로 작성했다. repository의 다른 branch나 final HEAD 구현을 과거 commit에 소급하지 않았으며, build·Compose·Caddy process는 이 작업 환경에서 직접 실행하지 않았다.
