===== BEGIN FILE: 01-runtime-composition-and-reverse-proxy-evolution.md =====
# 런타임 조립과 reverse proxy의 발전

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

PostgreSQL, Fastify API, Next.js Web, Caddy를 하나의 실행 단위로 조립한 초기 Compose에서 시작해, 개발용 source-mounted runtime과 외부 gateway가 production delivery 구조로 발전하는 과정을 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- 브라우저가 접근하는 외부 진입점은 Caddy의 단일 origin이며 API, WebSocket, Web UI의 전달 경로가 구분됩니다.
- API는 DB availability 이후에 시작하고 Web은 API와 함께 재현 가능한 multi-service runtime으로 조립됩니다.
- production start를 선택한 runtime은 Web build 결과를 실제 production server로 실행해야 합니다.
- 외부에 공개하면 안 되는 내부 운영 endpoint는 gateway에서 명시적으로 차단됩니다.
- 초기 source-mounted Compose와 최종 immutable image 기반 runtime의 차이를 구분할 수 있어야 합니다.

## 2. 핵심 질문

- 초기 `docker-compose.yml`은 어떤 service, port, volume, dependency를 소유합니까?
- Caddy의 `/api/*`, `/ws`, default handler가 외부 URL을 내부 service로 어떻게 전달합니까?
- API/Web가 dev command에서 production start 경로로 이동하면서 build와 실행 순서가 어떻게 달라집니까?
- Caddy configuration이 bind mount에서 image-owned artifact로 바뀌는 이유는 무엇입니까?
- `/api/metrics` 같은 내부 endpoint가 외부 gateway에서 차단되는 실제 route rule을 확인할 수 있습니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `19b4d9f9083d` | `build(runtime): Compose와 Caddy 라우팅 추가` | B | REALTIME, PERSISTENCE, WEB | PostgreSQL, API, Web, Caddy를 하나의 multi-service runtime으로 구성합니다. |
| 2 | `ec000bed0414` | `build(web): production start와 TS cache 정책 구성` | B | WEB | Web package에 production server 실행 경계를 마련합니다. |
| 3 | `be15e937d718` | `fix(runtime): Compose에서 build 결과 실행` | B | WEB, OPERATIONS | Compose가 API production start와 Web build→production start 순서를 실제로 사용하도록 수정합니다. |
| 4 | `576eb97f8041` | `build(docker): Caddy reverse proxy 구성` | B | REALTIME, OPERATIONS, OBSERVABILITY | Caddy configuration을 immutable image로 옮기고 내부 metrics 경로의 외부 노출을 차단합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(runtime): Compose와 Caddy 라우팅 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `19b4d9f9083d` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE, WEB |
| Source에서 확정된 역할 | PostgreSQL, API, Web, Caddy를 하나의 multi-service runtime으로 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent에 없던 `docker-compose.yml`의 `db`, `api`, `web`, `caddy` service와 `pong-pong-db`, `api-node-modules`, `web-node-modules` volume의 소유 관계를 확인합니다.
- `api`/`web` command가 `corepack enable`, frozen install, `dev`를 startup마다 수행하는지와 source bind mount가 image보다 우선하는 범위를 확인합니다.
- DB `pg_isready` healthcheck와 `depends_on` 조건, host port 4000/3000/8080 공개, 고정 credential 및 migration 부재를 기록합니다.
- `Caddyfile`의 `handle_path /api/*`, `handle /ws`, default `reverse_proxy`가 prefix와 WebSocket upgrade를 어떻게 전달하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | parent에는 네 서비스를 함께 기동하는 Compose와 외부 gateway가 없었습니다. API/Web는 repository source와 package script를 개별 실행해야 했고, DB·origin·WebSocket 경로를 한 번에 조립하는 실행 계약도 없었습니다. |
| 해결하려던 문제 | 브라우저, API, WebSocket, PostgreSQL을 동일한 개발 환경에서 재현할 표준 진입점이 없었습니다. 각 개발자가 port와 dependency를 따로 맞추면 URL·startup 순서·DB 상태가 달라질 수 있었습니다. |
| 핵심 결정 | `docker-compose.yml`에 PostgreSQL 16, Node 기반 API/Web, Caddy를 추가하고 `Caddyfile`이 `:8080` 단일 origin에서 API·WS·Web을 분기하도록 했습니다. DB healthcheck 뒤 API가 시작하고, Web/Caddy는 service-start 조건으로 연결했습니다. |
| build → package → execute 흐름 | `docker compose up` → DB volume 생성과 `pg_isready` → API/Web container가 source를 bind mount하고 startup마다 pnpm frozen install → 각각 dev server 실행 → Caddy가 `/api/*`를 prefix 제거 후 `api:4000`, `/ws`를 upgrade 가능한 채로 API, 나머지를 `web:3000`에 전달합니다. |
| ownership/lifetime/cleanup | Compose가 container·network·named volume lifetime을 소유합니다. PostgreSQL data는 `pong-pong-db`에 남고, API/Web dependency cache는 각각 named volume에 남습니다. source와 `Caddyfile`은 host bind mount라 host가 내용을 소유합니다. |
| failure/rollback/fail-closed | DB healthcheck 실패 시 API 시작이 막히지만 migration은 실행하지 않습니다. API/Web는 install 또는 dev process 실패 시 해당 container가 종료될 뿐 rollback이 없습니다. Caddy는 API/Web의 readiness가 아니라 시작 여부만 보고 기동합니다. |
| 보장하는 것 | 한 명령으로 네 service와 단일 browser origin을 조립하며, API는 DB health 이후 시작합니다. route precedence와 내부 service 이름은 파일로 재현됩니다. |
| 보장하지 않는 것 | production artifact, immutable image, required secret, migration success, API/Web readiness, 외부 port 최소화, metrics 차단을 보장하지 않습니다. 고정 개발 credential과 source mount를 사용합니다. |
| 후속 연결 | `ec000bed0414`가 Web package에 production `start` 경계를 추가하고, `be15e937d718`이 이 Compose가 실제 build 결과를 실행하도록 바꿉니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `ec000bed0414` — `build(web): production start와 TS cache 정책 구성`

### 5.2. `build(web): production start와 TS cache 정책 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `ec000bed0414` |
| Importance | B |
| Tags | WEB |
| Source에서 확정된 역할 | Web package에 production server 실행 경계를 마련합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/package.json`에서 `build: next build`와 새 `start: next start --hostname 0.0.0.0 --port 3000`의 producer/consumer 관계를 확인합니다.
- `apps/web/tsconfig.json`의 `incremental: false`가 `.tsbuildinfo` cache를 production build 계약에서 제외하는지 확인합니다.
- 같은 SHA의 `docker-compose.yml`은 여전히 Web `dev`를 실행하므로 새 `start` script의 runtime consumer가 아직 없다는 점을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 초기 Compose와 Web package는 `next dev`만 runtime entrypoint로 사용했습니다. `next build` 결과를 독립 production server로 소비하는 package script가 없었습니다. |
| 해결하려던 문제 | delivery 환경이 dev server를 그대로 사용하면 build-time 검증과 production process 동작을 분리할 수 없고, container나 CI가 어떤 command로 결과물을 실행해야 하는지 명시되지 않습니다. |
| 핵심 결정 | Web package에 `start`를 추가해 `next build`가 만든 `.next`를 `0.0.0.0:3000`에서 실행하도록 했고, TypeScript incremental cache를 끄는 정책을 적용했습니다. |
| build → package → execute 흐름 | producer는 `pnpm --filter @pong-pong/web build`, consumer는 `pnpm --filter @pong-pong/web start`입니다. 이 SHA에서는 package-level 경계만 생겼고 Compose는 아직 `dev`를 호출합니다. |
| ownership/lifetime/cleanup | Next build가 `.next` artifact를 생성하고 Web package가 이를 소비합니다. process lifetime은 `next start` 호출자가 소유합니다. incremental cache를 끄므로 `.tsbuildinfo` 재사용은 이 경로의 입력이 아닙니다. |
| failure/rollback/fail-closed | `start` 전에 build가 없거나 `.next`가 없으면 server는 실행할 수 없습니다. 이 commit은 호출 순서를 강제하지 않으며 artifact 정리·rollback도 추가하지 않습니다. |
| 보장하는 것 | Web package가 dev server와 production server를 서로 다른 명령으로 표현합니다. |
| 보장하지 않는 것 | Compose·Docker·CI가 이 명령을 실제로 사용한다는 보장은 없고, standalone artifact나 static asset packaging도 아직 구성하지 않습니다. |
| 후속 연결 | `be15e937d718`이 Compose Web command를 `build && start`로 바꾸고 `.next` volume을 추가합니다. |

비교 기준:
- 직전 관련 SHA: `19b4d9f9083d` — `build(runtime): Compose와 Caddy 라우팅 추가`
- 다음 관련 SHA: `be15e937d718` — `fix(runtime): Compose에서 build 결과 실행`

### 5.3. `fix(runtime): Compose에서 build 결과 실행`

| 항목 | 값 |
| --- | --- |
| SHA | `be15e937d718` |
| Importance | B |
| Tags | WEB, OPERATIONS |
| Source에서 확정된 역할 | Compose가 API production start와 Web build→production start 순서를 실제로 사용하도록 수정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent의 `docker-compose.yml`과 비교해 API command가 `dev`에서 `start`로, Web command가 `dev`에서 `build && start`로 바뀐 정확한 shell chain을 확인합니다.
- 새 `web-next` named volume이 `/workspace/apps/web/.next` artifact를 소유하고 source bind mount와 어떤 순서로 결합되는지 확인합니다.
- install은 여전히 startup마다 수행되고 API/Web source도 bind mount된다는 점을 기록해 production-like process와 immutable delivery를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `ec000bed0414`에서 Web `start`가 생겼지만 Compose는 API/Web 모두 dev command를 사용했습니다. 따라서 Compose 성공은 production server가 build 결과를 실행할 수 있다는 증거가 아니었습니다. |
| 해결하려던 문제 | runtime 선언과 package delivery 경계가 불일치했습니다. Web artifact를 만들지 않고 `start`할 수 없고, dev server를 계속 쓰면 production process 전환이 실제 조립 경로에 반영되지 않습니다. |
| 핵심 결정 | API는 package `start`, Web은 startup shell에서 `build && start`를 실행하도록 바꿨습니다. `.next`를 보존할 `web-next` volume도 추가했습니다. |
| build → package → execute 흐름 | container 시작 → corepack/frozen install → API `start`; Web은 install → `next build` 성공 → `next start`. `&&` 때문에 build가 실패하면 production server는 실행되지 않습니다. |
| ownership/lifetime/cleanup | Web container가 startup 중 `.next`를 만들고 `web-next` volume이 container 재생성 사이 artifact를 보유합니다. Compose가 process와 volume lifecycle을 소유하지만 source는 계속 host가 소유합니다. |
| failure/rollback/fail-closed | Web build 실패는 `&&`에서 startup을 차단합니다. API start 실패도 container 종료로 드러납니다. 다만 startup마다 install/build하므로 느린 기동·network dependency·stale volume 위험은 남습니다. |
| 보장하는 것 | 이 Compose 경로에서는 dev server 대신 API production start와 Web build 결과 실행이 선택됩니다. |
| 보장하지 않는 것 | image build 시점의 artifact 고정, one-shot migration, readiness-gated gateway, non-root runner를 보장하지 않습니다. `.next` volume의 오래된 파일 정리 정책도 없습니다. |
| 후속 연결 | `576eb97f8041`이 gateway config 자체를 image artifact로 만들고 metrics 차단을 추가합니다. source mount 제거와 built image 소비는 `2c44cb7cd71f`에서 완성됩니다. |

비교 기준:
- 직전 관련 SHA: `ec000bed0414` — `build(web): production start와 TS cache 정책 구성`
- 다음 관련 SHA: `576eb97f8041` — `build(docker): Caddy reverse proxy 구성`

### 5.4. `build(docker): Caddy reverse proxy 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `576eb97f8041` |
| Importance | B |
| Tags | REALTIME, OPERATIONS, OBSERVABILITY |
| Source에서 확정된 역할 | Caddy configuration을 immutable image로 옮기고 내부 metrics 경로의 외부 노출을 차단합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 새 `Caddy.Dockerfile`의 `FROM caddy:2-alpine`과 `COPY Caddyfile /etc/caddy/Caddyfile`로 config ownership이 image layer로 이동하는지 확인합니다.
- `Caddyfile`의 `route`, `@metrics path /api/metrics`, `respond @metrics 404`가 일반 `/api/*` proxy보다 먼저 평가되는지 확인합니다.
- 이 SHA의 `docker-compose.yml`은 Caddy image build를 아직 소비하지 않고 기존 `caddy:2-alpine`과 bind mount를 유지한다는 historical gap을 명시합니다.
- 후속 `2c44cb7cd71f`에서야 Compose `caddy.build.dockerfile: Caddy.Dockerfile`이 연결되는 handoff를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | gateway는 upstream Caddy image와 host `Caddyfile` bind mount에 의존했고 `/api/*`가 metrics endpoint까지 그대로 API로 전달했습니다. |
| 해결하려던 문제 | host config 변경이 runtime behavior를 바꿀 수 있어 배포 artifact와 route 정책이 분리됐고, 내부 운영 endpoint가 공개 gateway를 통해 노출될 수 있었습니다. |
| 핵심 결정 | Caddy config를 전용 image에 COPY하고, route 첫 단계에서 `/api/metrics`를 404로 종료한 뒤 나머지 API/WS/Web proxy를 유지했습니다. |
| build → package → execute 흐름 | image build가 Caddyfile을 `/etc/caddy/Caddyfile`에 고정 → request가 `route`에 진입 → metrics exact path는 404 → `/api/*`는 API → `/ws`는 WebSocket upstream → default는 Web입니다. |
| ownership/lifetime/cleanup | 정의상 Caddy image layer가 config를 소유합니다. 그러나 이 exact SHA의 Compose는 아직 새 image를 선택하지 않아 실제 조립 runtime의 config owner는 계속 host bind mount입니다. |
| failure/rollback/fail-closed | 잘못된 Caddyfile은 image build 또는 Caddy startup에서 실패할 수 있습니다. metrics는 gateway에서 fail-closed 404가 되지만 직접 API port 접근은 이 SHA의 Compose가 여전히 host 4000을 공개하므로 차단되지 않습니다. |
| 보장하는 것 | 전용 image를 사용하는 consumer는 route config와 metrics 차단을 immutable artifact로 받을 수 있습니다. |
| 보장하지 않는 것 | 이 commit만으로 Compose가 그 image를 사용하거나 내부 API port를 닫는 것은 아닙니다. route runtime test도 추가하지 않습니다. |
| 후속 연결 | `2c44cb7cd71f`이 Caddy image build를 production Compose에 연결하고 API/Web host port와 bind mount를 제거하며, `e2c12ded1d5f`이 그 정적 계약을 검사합니다. |

비교 기준:
- 직전 관련 SHA: `be15e937d718` — `fix(runtime): Compose에서 build 결과 실행`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `19b4d9f9083d` | PostgreSQL, API, Web, Caddy를 하나의 multi-service runtime으로 구성합니다. | 도입 | `docker-compose.yml`의 네 service, DB health dependency, named volume과 `Caddyfile`의 세 route가 최초 multi-service runtime을 정의합니다. |
| `ec000bed0414` | Web package에 production server 실행 경계를 마련합니다. | 확장 | `apps/web/package.json`의 `start`가 build artifact 소비 경계를 만들지만 같은 SHA의 Compose는 아직 이를 사용하지 않습니다. |
| `be15e937d718` | Compose가 API production start와 Web build→production start 순서를 실제로 사용하도록 수정합니다. | 수정 | `docker-compose.yml`의 command와 `web-next` volume이 package-level production start를 실제 runtime consumer에 연결합니다. |
| `576eb97f8041` | Caddy configuration을 immutable image로 옮기고 내부 metrics 경로의 외부 노출을 차단합니다. | 확장 | `Caddy.Dockerfile`과 route 우선순위가 immutable gateway artifact와 metrics fail-closed 정책을 정의하지만 Compose 소비는 후속 Thread로 이관됩니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| Web package에 `start`는 생겼지만 Compose가 계속 `dev`를 호출함 | `be15e937d718` | 동일 SHA의 `docker-compose.yml` command | Web은 `build && start`를 거치므로 build 실패 시 server가 시작되지 않습니다. 다만 artifact 존재 회귀 검사는 Thread 02에서 추가됩니다. |
| host bind-mounted gateway config와 `/api/metrics` 외부 전달 | `576eb97f8041` | 후속 `2c44cb7cd71f` + `e2c12ded1d5f` | 전용 Caddy image와 404 route가 정책을 정의하고, 후속 production Compose/contract test가 실제 consumer와 정적 회귀를 연결합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| Web `.next` artifact | Compose Web startup의 `next build` | 같은 container의 `next start` | `web-next` named volume lifetime | build 실패 시 `&&`가 start를 차단; stale artifact 정리는 없음 |
| Caddy configuration | 처음에는 host file, `576...`부터 image build | Caddy process | bind mount 또는 image layer lifetime | config 오류 시 Caddy startup 실패; exact SHA의 Compose handoff는 아직 없음 |
| PostgreSQL data/credential | Postgres service와 Compose env | API DB client | `pong-pong-db` volume | DB health 실패 시 API start 차단; migration·secret fail-closed는 없음 |
| route contract evidence | exact SHA의 Caddyfile diff 검사 | 학습자/후속 contract test | repository history | 실행하지 않았으므로 runtime routing 성공은 주장하지 않음 |

## 9. Thread 최종 상태

- 최종 delivery owner: 이 Thread 마지막 SHA 기준으로 Compose가 service 조립을, package scripts가 API/Web process를, Caddy image 정의가 gateway config artifact를 소유합니다. 실제 Caddy image 소비는 후속 `2c44cb7cd71f`의 책임입니다.
- source와 production artifact의 관계: API/Web는 여전히 source bind mount와 startup install에 의존하지만 production package command를 실행합니다. gateway config만 image artifact로 정의되었습니다.
- build-time과 runtime configuration의 관계: Web build와 start가 같은 startup shell에 있어 `NEXT_PUBLIC_*` 값은 startup build 시 고정됩니다. Caddy route는 image build 시 고정될 수 있으나 이 SHA의 Compose는 bind mount를 사용합니다.
- startup/readiness/shutdown contract: DB health 뒤 API가 시작하지만 API/Web/Caddy는 readiness gate가 없고 shutdown budget도 없습니다.
- fail-closed 조건: Web build 실패는 start를 막고 gateway metrics path는 404입니다. required secret·migration·내부 port 공개는 아직 fail-closed가 아닙니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: exact SHA 파일/diff로 command, dependency, route를 확인했습니다. Docker/Compose/Caddy process는 실행하지 않았으므로 image build·route runtime·network 동작 증거는 없습니다.

## 10. 최종 execution/delivery flow

```text
browser → Caddy :8080
Caddy `/api/metrics` → 404
Caddy `/api/*` → Fastify :4000 (`handle_path`가 `/api` 제거)
Caddy `/ws` → Fastify WebSocket :4000
Caddy default → Next.js :3000
PostgreSQL health → API production start
Web startup → frozen install → `next build` → `next start`
Caddy image 정의 → 후속 `2c44cb7cd71f`에서 Compose consumer 연결
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `02-production-build-and-package-artifacts.md`: source 실행이 compiled artifact 실행으로 바뀌는 단계
- `03-container-images-and-production-runtime-lifecycle.md`: source-mounted service와 Caddy image 정의가 built image와 health-gated lifecycle로 연결되는 단계
- `07-runtime-observability-and-service-health`: readiness와 graceful drain 자체의 애플리케이션 의미

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 01-runtime-composition-and-reverse-proxy-evolution.md =====

===== BEGIN FILE: 02-production-build-and-package-artifacts.md =====
# Production build와 package artifact

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

TypeScript source를 직접 import하거나 실행하던 workspace를 shared/db/API/Web별 명시적인 production artifact로 전환하고, runtime에 필요한 JavaScript, declaration, migration, Next.js standalone output이 실제 build 결과에 포함되는지 검증하는 과정을 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- production runtime은 workspace의 TypeScript source tree를 실행 계약으로 삼지 않습니다.
- `@pong-pong/shared`와 `@pong-pong/db`는 compiled JavaScript와 type declaration을 production export로 제공합니다.
- DB artifact에는 production migration 실행에 필요한 migration set이 함께 포함됩니다.
- API start는 compiled `dist/index.js`를 실행하고 Web은 Next.js standalone artifact를 생성합니다.
- root build는 shared → db → api → web의 dependency 순서를 보존합니다.
- CI는 compile 성공만 보지 않고 실제 runtime artifact의 존재와 형태를 별도 contract로 검증합니다.

## 2. 핵심 질문

- development export와 production `types`/`import`/`default` export는 package 소비 경로를 어떻게 분리합니까?
- NodeNext ESM build에서 상대 import에 `.js` 확장자를 붙이는 이유가 emitted artifact에서 어떻게 드러납니까?
- DB migration directory를 `dist/`에 포함하고 `migrate:prod`를 추가한 이유는 무엇입니까?
- Next.js `output: standalone`, tracing root, shared runtime alias가 monorepo production artifact에 어떤 영향을 줍니까?
- `verify:build`가 일반 `build` 성공과 별도로 무엇을 증명합니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `37c735de0c37` | `build(shared): production package artifact 구성` | B | PROTOCOL | shared package를 compiled production dependency로 구성합니다. |
| 2 | `430389943b34` | `build(db): production package artifact 구성` | B | PERSISTENCE | DB package에 compiled artifact, migration copy, production migration CLI를 구성합니다. |
| 3 | `bb67a72882bf` | `build(app): API와 Web production artifact 구성` | A | PERSISTENCE, WEB, OPERATIONS | API를 compiled `dist` 실행으로, Web을 standalone output으로 전환하고 root build dependency 순서를 고정합니다. |
| 4 | `6ab091ffa815` | `test(build): production artifact 생성 검증` | B | PERSISTENCE, WEB, OPERATIONS | production runtime에 필요한 build output을 post-build contract로 검증합니다. |
| 5 | `09b305b49768` | `ci(build): production artifact 검증 실행` | B | PERSISTENCE, WEB, OPERATIONS | CI가 workspace build 직후 artifact verifier를 실행하도록 연결합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(shared): production package artifact 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `37c735de0c37` |
| Importance | B |
| Tags | PROTOCOL |
| Source에서 확정된 역할 | shared package를 compiled production dependency로 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/shared/package.json`의 `main`, `types`, conditional `exports`에서 `development` source와 production `dist` consumer를 구분합니다.
- `packages/shared/tsconfig.build.json`의 declaration/source map/output 설정과 test exclusion을 확인합니다.
- `packages/shared/src/index.ts`와 내부 상대 import에 `.js` 확장자가 추가되어 NodeNext emitted ESM이 실제 파일을 해석할 수 있는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared package는 workspace TypeScript source를 직접 가리키는 개발 중심 경계였습니다. runtime JavaScript와 consumer용 declaration을 독립 artifact로 배포하는 계약이 없었습니다. |
| 해결하려던 문제 | API·DB·Web production build가 shared source tree와 TypeScript loader에 암묵적으로 의존하면 package 단위 artifact를 image나 Node process가 안정적으로 소비할 수 없습니다. |
| 핵심 결정 | `package.json`의 production entry를 `dist/index.js`와 `dist/index.d.ts`로 지정하고, conditional export의 `development`만 source를 가리키게 했습니다. build용 tsconfig가 JS·d.ts·map을 `dist`에 emit하며 tests는 제외합니다. |
| build → package → execute 흐름 | `pnpm --filter @pong-pong/shared build` → TypeScript compiler가 `src`를 `dist`로 emit → production resolver는 package export의 `import/default`와 `types`를 소비합니다. 개발 조건에서는 source export를 선택할 수 있습니다. |
| ownership/lifetime/cleanup | shared package build가 `dist`를 생성하고 downstream package가 이를 읽습니다. artifact lifetime은 build workspace 또는 이후 image layer까지이며 source와 독립적으로 교체할 수 있습니다. |
| failure/rollback/fail-closed | compiler error면 artifact가 생성되지 않습니다. NodeNext ESM 상대 import가 `.js`를 가리키지 않으면 emitted JS가 런타임에 module을 찾지 못할 수 있어 source import를 수정했습니다. |
| 보장하는 것 | shared protocol/types가 production JavaScript와 declaration으로 제공되고 test source는 artifact에서 제외됩니다. |
| 보장하지 않는 것 | artifact 존재를 post-build로 검사하거나 downstream API/Web가 올바른 순서로 build한다는 보장은 아직 없습니다. |
| 후속 연결 | `430389943b34`가 같은 경계를 DB package와 migration artifact에 적용합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `430389943b34` — `build(db): production package artifact 구성`

### 5.2. `build(db): production package artifact 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `430389943b34` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | DB package에 compiled artifact, migration copy, production migration CLI를 구성합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `packages/db/package.json`의 production exports, `build` script, `migrate:prod: node dist/cli.js migrate`를 확인합니다.
- build가 TypeScript emit 뒤 `migrations` directory를 `dist/migrations`로 복사하는 실제 command를 확인합니다.
- `packages/db/src/migrator.ts`의 `new URL('../migrations', import.meta.url)`가 compiled `dist` 기준으로 어떤 directory를 요구하는지 추적합니다.
- DB source 상대 import의 `.js` 확장자와 `tsconfig.build.json`의 declaration/output 범위를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | DB package도 source export와 source migration 경로에 의존했습니다. compiled CLI가 존재하더라도 runtime에서 읽을 SQL migration set이 같은 artifact에 포함된다는 보장이 없었습니다. |
| 해결하려던 문제 | production runner가 source tree를 복사하지 않으면 `import.meta.url` 기준 migration directory를 찾지 못합니다. DB API와 migration CLI를 독립 artifact로 실행할 수 있어야 했습니다. |
| 핵심 결정 | DB package를 `dist` export로 전환하고, build 후 SQL directory를 `dist/migrations`에 복사했습니다. `migrate:prod`는 `node dist/cli.js migrate`를 실행합니다. |
| build → package → execute 흐름 | shared build → DB TypeScript emit → migration SQL copy → production process가 `dist/cli.js migrate` → `migrator.ts`가 자신의 emitted 위치에서 `../migrations`를 URL로 해석해 SQL을 순서대로 소비합니다. |
| ownership/lifetime/cleanup | DB package build가 JavaScript·declaration·SQL set을 함께 소유합니다. migration process는 DB connection과 SQL 실행 lifetime을 소유하고 종료 후 process resource를 반환해야 합니다. |
| failure/rollback/fail-closed | SQL copy가 누락되면 compiled migrator가 directory를 찾지 못해 startup/migration이 실패합니다. 이 commit은 migration을 API startup과 자동 연결하거나 transaction rollback 정책을 새로 정의하지 않습니다. |
| 보장하는 것 | production artifact만 복사한 환경에서도 DB package API와 migration CLI에 필요한 파일 구조가 존재하도록 설계됩니다. |
| 보장하지 않는 것 | 모든 migration 파일의 존재를 자동 검사하거나 PostgreSQL 연결·migration 성공을 실행 검증하지 않습니다. |
| 후속 연결 | `bb67a72882bf`가 API/Web artifact와 root dependency build 순서를 추가하고, `6ab091ffa815`가 migration 포함 artifact 존재를 검사합니다. |

비교 기준:
- 직전 관련 SHA: `37c735de0c37` — `build(shared): production package artifact 구성`
- 다음 관련 SHA: `bb67a72882bf` — `build(app): API와 Web production artifact 구성`

### 5.3. `build(app): API와 Web production artifact 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `bb67a72882bf` |
| Importance | A |
| Tags | PERSISTENCE, WEB, OPERATIONS |
| Source에서 확정된 역할 | API를 compiled `dist` 실행으로, Web을 standalone output으로 전환하고 root build dependency 순서를 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/package.json`의 `build`, `start: node dist/index.js`와 `apps/api/tsconfig.build.json`이 API source/test를 어떻게 분리하는지 확인합니다.
- API 내부 상대 import 전반의 `.js` suffix가 emitted NodeNext ESM dependency graph를 완성하는지 확인합니다.
- `apps/web/next.config.mjs`의 `output: 'standalone'`, tracing root, shared runtime alias, `transpilePackages`가 monorepo file tracing에 미치는 영향을 확인합니다.
- Web `predev`/`prebuild`/`pretypecheck`/`pretest`가 shared build를 선행하고, root `build`가 shared → db → api → web 순서를 고정하는지 확인합니다.
- 이 SHA에서 root `verify:build` entrypoint는 추가되지만 `tests/build-artifacts.mjs`는 아직 없다는 incomplete handoff를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shared와 DB만 compiled package 경계를 갖고 API는 source 실행에 의존했습니다. Web도 monorepo dependency를 포함한 독립 standalone server artifact를 명시적으로 만들지 않았습니다. root build 순서도 delivery dependency를 한 곳에서 고정하지 않았습니다. |
| 해결하려던 문제 | API runner가 TypeScript loader/source를 요구하고, Next file tracing이 workspace shared runtime을 놓치면 image에서 server가 시작되지 않습니다. downstream package가 upstream artifact보다 먼저 build되면 alias와 type/runtime entry가 불완전해질 수 있습니다. |
| 핵심 결정 | API build/start를 `dist` 기반으로 전환하고 build tsconfig와 ESM `.js` import를 정리했습니다. Web은 standalone output, monorepo tracing root, shared `dist` alias를 사용합니다. root build가 shared→db→api→web을 직렬화합니다. |
| build → package → execute 흐름 | root `pnpm build` → shared `dist` → DB `dist`+migrations → API `dist/index.js` → Next build가 shared runtime을 trace해 `.next/standalone` server와 static output 생성 → API는 `node apps/api/dist/index.js`, Web은 standalone server consumer가 실행합니다. |
| ownership/lifetime/cleanup | 각 workspace가 자신의 artifact를 생성하지만 root script가 dependency ordering을 소유합니다. API artifact lifetime은 `dist`, Web server artifact는 `.next/standalone`; build workspace나 image builder가 정리 책임을 가집니다. |
| failure/rollback/fail-closed | upstream build 실패는 shell chain을 중단해 downstream artifact 생성을 막습니다. standalone tracing/alias가 잘못되면 build 성공 후 runtime module 누락이 생길 수 있습니다. `verify:build` script 이름은 생겼지만 verifier 파일이 아직 없어 호출 시 실패합니다. |
| 보장하는 것 | source loader 없이 실행할 API entry와 monorepo-aware standalone Web artifact, 그리고 재현 가능한 workspace build 순서가 코드로 고정됩니다. |
| 보장하지 않는 것 | 생성된 파일이 실제로 모두 존재하는지, standalone server가 기동하는지, migration이 적용되는지, static/public asset이 완전한지는 이 commit만으로 증명하지 않습니다. |
| 후속 연결 | `6ab091ffa815`가 누락된 verifier 구현을 추가하고 12개 핵심 artifact 존재를 검사합니다. `09b305b49768`이 이를 CI build 뒤에 연결합니다. |

비교 기준:
- 직전 관련 SHA: `430389943b34` — `build(db): production package artifact 구성`
- 다음 관련 SHA: `6ab091ffa815` — `test(build): production artifact 생성 검증`

### 5.4. `test(build): production artifact 생성 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `6ab091ffa815` |
| Importance | B |
| Tags | PERSISTENCE, WEB, OPERATIONS |
| Source에서 확정된 역할 | production runtime에 필요한 build output을 post-build contract로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 새 `tests/build-artifacts.mjs`의 artifact path list를 확인해 shared JS/d.ts, DB JS/d.ts/CLI/migrator/migrations, API entry/modules, Web standalone server를 정확히 나열합니다.
- missing path를 수집해 한 번에 throw하는 failure 방식과 성공 시 verified count를 출력하는 방식을 확인합니다.
- 검사가 file existence만 확인하며 artifact 내용, executable startup, 모든 migration, `.next/static` 또는 public asset은 검사하지 않는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `bb67a72882bf`는 production artifact를 만들도록 구성하고 `verify:build` script까지 선언했지만 실제 verifier module이 없었습니다. build exit 0만으로 runtime 필수 파일의 누락을 구분할 수 없었습니다. |
| 해결하려던 문제 | compiler/Next build가 성공해도 copy script, migration directory, standalone tracing 결과가 기대 경로에 없으면 image assembly 또는 startup 단계에서 늦게 실패합니다. |
| 핵심 결정 | Node script가 12개 핵심 path의 존재를 검사하고 누락 목록이 있으면 exception으로 non-zero 종료하도록 했습니다. |
| build → package → execute 흐름 | production build 완료 → `node tests/build-artifacts.mjs` → repository root 기준 path별 `stat/access` 검사 → 누락 목록이면 throw → 모두 있으면 verified count log. |
| ownership/lifetime/cleanup | verifier는 artifact를 생성·수정하지 않고 관찰만 합니다. build command가 producer, script가 contract consumer이며 CI나 개발 shell이 process exit를 소유합니다. |
| failure/rollback/fail-closed | 한 파일이라도 없으면 누락 경로를 포함해 실패하므로 delivery pipeline을 차단합니다. cleanup이나 rebuild는 하지 않습니다. |
| 보장하는 것 | 선택된 shared/DB/API/Web production artifact와 일부 migration SQL이 expected path에 존재함을 결정적으로 확인합니다. |
| 보장하지 않는 것 | 파일이 유효한 JS/SQL인지, 모든 migration이 포함됐는지, process가 시작되는지, standalone static/public asset이 완전한지는 증명하지 않습니다. |
| 후속 연결 | `09b305b49768`이 CI의 build 직후 이 verifier를 실행해 회귀 차단 경계를 repository workflow로 올립니다. |

비교 기준:
- 직전 관련 SHA: `bb67a72882bf` — `build(app): API와 Web production artifact 구성`
- 다음 관련 SHA: `09b305b49768` — `ci(build): production artifact 검증 실행`

### 5.5. `ci(build): production artifact 검증 실행`

| 항목 | 값 |
| --- | --- |
| SHA | `09b305b49768` |
| Importance | B |
| Tags | PERSISTENCE, WEB, OPERATIONS |
| Source에서 확정된 역할 | CI가 workspace build 직후 artifact verifier를 실행하도록 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`의 build step 직후 `pnpm verify:build` step이 배치되는지 확인합니다.
- 동일 checkout/install/build workspace의 artifact를 verifier가 소비하므로 job filesystem lifetime과 step failure propagation을 확인합니다.
- workflow가 verifier의 정적 존재 검사만 추가하며 process startup, Docker image build, PostgreSQL migration은 아직 수행하지 않는 점을 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | artifact verifier는 로컬 command로만 존재해 호출하지 않으면 build 성공 상태가 그대로 통과했습니다. |
| 해결하려던 문제 | 중요 artifact의 copy/export/tracing 경로가 깨져도 CI가 단순 build exit 0만 보고 release candidate를 허용할 수 있었습니다. |
| 핵심 결정 | repository CI build 직후 `pnpm verify:build`를 같은 job에 추가했습니다. |
| build → package → execute 흐름 | checkout → pinned install → typecheck/unit → root build → 같은 workspace에서 verifier → missing artifact면 step과 job 실패. |
| ownership/lifetime/cleanup | CI runner filesystem이 build output과 verifier evidence를 job lifetime 동안 소유합니다. workflow가 invocation order와 실패 전파를 소유합니다. |
| failure/rollback/fail-closed | build 실패 또는 verifier failure가 이후 성공 상태를 차단합니다. 별도 artifact upload나 cleanup 정책은 없고 hosted runner 종료로 workspace가 폐기됩니다. |
| 보장하는 것 | push/PR CI에서 production build와 선택된 artifact existence contract가 함께 통과해야 합니다. |
| 보장하지 않는 것 | artifact runtime 실행, Docker packaging, 실제 DB migration, browser delivery는 이 job이 증명하지 않습니다. |
| 후속 연결 | Thread 03의 image commits가 이 artifact를 runner layer로 복사하고, Thread 04의 process/browser job이 실제 production process를 기동합니다. |

비교 기준:
- 직전 관련 SHA: `6ab091ffa815` — `test(build): production artifact 생성 검증`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `37c735de0c37` | shared package를 compiled production dependency로 구성합니다. | 도입 | `packages/shared/package.json` production exports와 `tsconfig.build.json`이 source와 compiled consumer를 분리합니다. |
| `430389943b34` | DB package에 compiled artifact, migration copy, production migration CLI를 구성합니다. | 확장 | `packages/db/package.json`, `tsconfig.build.json`, `migrator.ts`가 compiled code와 SQL migration을 하나의 production package로 결합합니다. |
| `bb67a72882bf` | API를 compiled `dist` 실행으로, Web을 standalone output으로 전환하고 root build dependency 순서를 고정합니다. | 도입·불충분 | API `dist`, Web standalone, root build 순서는 도입됐지만 새 `verify:build` command의 target file은 이 SHA에 아직 없습니다. |
| `6ab091ffa815` | production runtime에 필요한 build output을 post-build contract로 검증합니다. | 검증 | `tests/build-artifacts.mjs`가 production consumer가 요구하는 선택된 path의 존재를 post-build contract로 검사합니다. |
| `09b305b49768` | CI가 workspace build 직후 artifact verifier를 실행하도록 연결합니다. | 통합 검증 | `.github/workflows/ci.yml`이 root build의 결과를 `pnpm verify:build`로 소비해 artifact contract 실패를 CI 상태에 반영합니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| production package가 source export와 loader에 의존함 | `37c735de0c37`, `430389943b34`, `bb67a72882bf` | `6ab091ffa815` | workspace별 `dist`와 migration/standalone artifact를 만든 뒤 selected path를 정적으로 검사합니다. |
| `verify:build` command만 선언되고 verifier file이 없음 | `6ab091ffa815` | `09b305b49768` | 구현을 추가하고 CI build 직후 실행해 호출 누락도 막습니다. |
| build exit 0이면 runtime artifact가 완전하다고 가정함 | `6ab091ffa815` | 후속 Thread 04 process/browser job | existence 검사는 늦은 path 누락을 막지만 runtime validity는 별도 process 검증이 필요합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| shared package artifact | `@pong-pong/shared` build | DB/API/Web production resolver | `packages/shared/dist` 또는 image layer | compiler failure 시 downstream build 차단 |
| DB code + migrations | `@pong-pong/db` build와 SQL copy | production migration CLI/API | `packages/db/dist` 또는 image layer | copy 누락은 verifier 또는 runtime migrator 실패 |
| API/Web artifacts | API tsc와 Next build | Node API process와 standalone Web server | `dist`/`.next`에서 image copy까지 | root shell 순서가 upstream failure에서 중단 |
| CI verification evidence | `tests/build-artifacts.mjs` exit status | GitHub Actions build job | 단일 job lifetime | 누락 시 job failure; artifact 업로드는 없음 |

## 9. Thread 최종 상태

- 최종 delivery owner: root `build` script가 workspace producer 순서를, 각 package가 자신의 artifact를, `verify:build`와 CI가 selected output contract를 소유합니다.
- source와 production artifact의 관계: development conditional export는 source를 유지하지만 production export/start는 `dist`와 `.next/standalone`을 소비합니다.
- build-time과 runtime configuration의 관계: Next standalone과 public configuration은 build 단계에서 만들어지고 API/DB는 compiled JS를 runtime에 소비합니다. 이 Thread는 runtime secret 주입을 다루지 않습니다.
- startup/readiness/shutdown contract: artifact producer/consumer만 정의하며 process readiness와 shutdown은 후속 Threads의 책임입니다.
- fail-closed 조건: root build의 선행 단계 실패와 post-build missing path가 shell/CI를 non-zero로 종료합니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: exact SHA diff로 export, build script, verifier path와 CI invocation을 확인했습니다. 실제 pnpm build나 artifact verifier를 실행하지 않았으므로 runtime evidence는 기록하지 않습니다.

## 10. 최종 execution/delivery flow

```text
root `pnpm build`
→ `@pong-pong/shared` TypeScript → `packages/shared/dist`
→ `@pong-pong/db` TypeScript + SQL copy → `packages/db/dist`
→ API TypeScript → `apps/api/dist/index.js`
→ Next standalone build → `apps/web/.next/standalone` + `.next/static`
→ `pnpm verify:build`가 12개 핵심 path 검사
→ CI build job이 exit status를 delivery gate로 소비
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `01-runtime-composition-and-reverse-proxy-evolution.md`: source-driven runtime의 이전 상태
- `03-container-images-and-production-runtime-lifecycle.md`: 생성된 artifact를 image runner가 소비하는 후속 단계
- `08-verification-and-test-architecture`: artifact verifier를 테스트 관점에서 해석하는 카테고리

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 02-production-build-and-package-artifacts.md =====

===== BEGIN FILE: 03-container-images-and-production-runtime-lifecycle.md =====
# Container image와 production runtime lifecycle

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

compiled artifact를 API/Web multi-stage image로 패키징하고 source mount와 startup-time install/build를 제거한 뒤, migration → API readiness → Web readiness → Caddy 공개 순서의 production Compose lifecycle로 전환하는 과정을 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- production API/Web container는 startup 시 dependency install이나 source build를 수행하지 않습니다.
- runner image에는 실행에 필요한 artifact와 dependency만 포함되며 application process는 non-root로 실행됩니다.
- DB migration은 API와 분리된 one-shot service가 완료된 뒤 API가 시작됩니다.
- DB/API/Web 내부 port는 필요한 service 사이에만 노출되고 외부 공개 지점은 gateway로 제한됩니다.
- required secret은 default credential로 조용히 대체되지 않습니다.
- application room drain budget보다 container termination grace가 짧아서는 안 됩니다.

## 2. 핵심 질문

- API/Web Dockerfile의 dependencies → builder → runner stage가 각각 어떤 파일과 dependency를 소유합니까?
- `.dockerignore`가 build context에서 제외하는 항목은 무엇이며 final image와 context에 어떤 영향을 줍니까?
- Web의 `NEXT_PUBLIC_*` build argument가 runtime environment와 다른 lifecycle을 갖는 이유는 무엇입니까?
- Compose의 one-shot `migrate` service와 `service_healthy`/`service_completed_successfully` 조건이 startup 순서를 어떻게 강제합니까?
- source mount, startup install, host port 노출이 production lifecycle 전환에서 각각 어떻게 제거됩니까?
- `stop_grace_period`가 application의 60초 room drain과 어떤 관계를 갖습니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f8efb2656771` | `build(docker): production API image 구성` | B | PERSISTENCE, OPERATIONS | shared/db/API를 build한 뒤 non-root runner에 필요한 결과만 복사하는 multi-stage API image를 만듭니다. |
| 2 | `656893e8e1cb` | `build(docker): production Web image 구성` | B | REALTIME, WEB, OPERATIONS | Next.js standalone output과 static asset만 runner에 포함하는 multi-stage Web image를 만듭니다. |
| 3 | `2c44cb7cd71f` | `build(docker): production container lifecycle 구성` | A | PERSISTENCE, OPERATIONS | source-mounted Compose를 built images, one-shot migration, health-gated startup, required secrets로 교체합니다. |
| 4 | `e2c12ded1d5f` | `test(docker): production container contract 검증` | B | OPERATIONS, OBSERVABILITY, TEST | rendered Compose와 Dockerfile이 production image/lifecycle 규칙을 유지하는지 검사합니다. |
| 5 | `312ddbc6fbe2` | `fix(runtime): container 종료 유예를 room drain과 정렬` | A | REALTIME, OPERATIONS, RISK | API container termination grace를 application room-drain budget보다 길게 맞춥니다. |
| 6 | `73ba979841cd` | `test(docker): API 종료 유예 계약 검증` | B | OPERATIONS, TEST | Compose의 stop grace가 60초 application drain budget 이상인지 회귀 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(docker): production API image 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `f8efb2656771` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | shared/db/API를 build한 뒤 non-root runner에 필요한 결과만 복사하는 multi-stage API image를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.dockerignore`가 `.git`, `.github`, `node_modules`, `.next`, `dist`, coverage/report, `.env*`, log와 문서를 build context에서 제외하는지 확인합니다.
- `apps/api/Dockerfile`의 base/dependencies/builder/runner stage에서 root manifests, workspace package files, frozen install, shared→db→api build가 어떻게 이어지는지 확인합니다.
- runner가 API/DB/shared `dist`, migration, workspace manifests와 `node_modules`를 복사하고 `USER node`, `EXPOSE 4000`, `CMD node apps/api/dist/index.js`를 선택하는지 확인합니다.
- runner에 전체 install 결과가 복사되어 dev dependency까지 남을 수 있는 범위와 Dockerfile healthcheck 부재를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API production command와 compiled artifact는 있었지만 runtime은 source-mounted Compose에서 startup마다 install했습니다. 실행 환경을 build 단계에서 고정하는 API image가 없었습니다. |
| 해결하려던 문제 | startup network/install/source 상태에 따라 runtime이 달라지고 TypeScript source 및 build toolchain이 production container lifetime에 남았습니다. image가 DB/shared workspace artifact까지 정확히 포함해야 했습니다. |
| 핵심 결정 | multi-stage API Dockerfile을 추가했습니다. dependencies stage가 frozen lockfile로 workspace dependency를 설치하고 builder가 shared→db→api를 build한 뒤 runner가 manifests, install tree와 세 package의 compiled output을 복사해 non-root Node process를 실행합니다. |
| build → package → execute 흐름 | Docker context 정리 → base에서 exact Node/pnpm 준비 → dependencies에서 frozen install → builder에서 shared/DB/API artifact 생성 → runner에 필요한 tree 복사 → `USER node` → `node apps/api/dist/index.js`. |
| ownership/lifetime/cleanup | image builder가 dependency cache와 artifact 생성 lifetime을, final image layer가 copied files를, container runtime이 Node process를 소유합니다. `USER node`로 application write 권한을 제한합니다. |
| failure/rollback/fail-closed | frozen install 또는 어느 workspace build든 실패하면 image가 생성되지 않습니다. copy path가 없으면 Docker build가 실패합니다. runtime healthcheck와 DB/migration dependency는 이 image 자체에 없습니다. |
| 보장하는 것 | API container가 startup-time install/build 없이 compiled JS로 시작하고 root가 아닌 `node` 사용자로 실행됩니다. |
| 보장하지 않는 것 | production-only dependency pruning, image vulnerability scan, healthcheck, secret 주입, migration 완료, API readiness는 보장하지 않습니다. |
| 후속 연결 | `656893e8e1cb`가 Web standalone image를 추가하고 `2c44cb7cd71f`가 두 image를 production Compose lifecycle에 연결합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `656893e8e1cb` — `build(docker): production Web image 구성`

### 5.2. `build(docker): production Web image 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `656893e8e1cb` |
| Importance | B |
| Tags | REALTIME, WEB, OPERATIONS |
| Source에서 확정된 역할 | Next.js standalone output과 static asset만 runner에 포함하는 multi-stage Web image를 만듭니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/Dockerfile`에서 frozen workspace install과 shared build, Web build가 stage별로 어떻게 수행되는지 확인합니다.
- builder의 `ARG`/`ENV NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_WS_URL`, `NEXT_PUBLIC_APP_MODE`가 Next build에 주입되어 browser bundle에 고정되는 시점을 확인합니다.
- runner가 `.next/standalone`을 root로, `.next/static`을 `apps/web/.next/static`으로 복사하고 `USER node`, `CMD node apps/web/server.js`를 사용하는지 확인합니다.
- runtime env 변경만으로 public variable을 바꿀 수 없는 점, image healthcheck와 `public/` copy가 이 diff에 없는 점을 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Web standalone artifact는 만들어졌지만 container runner가 없었고 source-mounted Compose가 startup마다 Next build를 수행했습니다. |
| 해결하려던 문제 | browser-visible origin/mode와 server artifact를 하나의 immutable image에 고정하지 않으면 배포 시점마다 build 결과가 달라집니다. monorepo standalone layout을 올바른 path로 복사해야 했습니다. |
| 핵심 결정 | multi-stage Web image가 shared와 Next app을 build하고, runner에는 standalone server tree와 static output만 복사합니다. public API/WS/mode 값은 build ARG/ENV로 주입하며 non-root Node가 `apps/web/server.js`를 실행합니다. |
| build → package → execute 흐름 | frozen install → shared build → Next build with `NEXT_PUBLIC_*` → `.next/standalone` + `.next/static` 생성 → runner copy → `USER node` → standalone server start on port 3000. |
| ownership/lifetime/cleanup | image build가 public browser configuration과 server/static artifact를 소유합니다. final image가 배포 단위이며 container process는 artifact를 읽기만 합니다. |
| failure/rollback/fail-closed | 필수 upstream artifact나 Next build가 실패하면 image가 생성되지 않습니다. 잘못된 public URL도 build 자체는 성공할 수 있어 browser runtime에서 늦게 드러날 수 있습니다. |
| 보장하는 것 | source mount와 startup build 없이 Next standalone process를 non-root로 실행할 image 구조를 제공합니다. |
| 보장하지 않는 것 | runtime 환경변수로 browser URL을 교체할 수 없고, readiness/healthcheck, 실제 static/public asset 완전성, browser cookie origin은 이 commit에서 검증하지 않습니다. |
| 후속 연결 | `2c44cb7cd71f`가 Compose build args와 health-gated service로 이 image를 소비합니다. |

비교 기준:
- 직전 관련 SHA: `f8efb2656771` — `build(docker): production API image 구성`
- 다음 관련 SHA: `2c44cb7cd71f` — `build(docker): production container lifecycle 구성`

### 5.3. `build(docker): production container lifecycle 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `2c44cb7cd71f` |
| Importance | A |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | source-mounted Compose를 built images, one-shot migration, health-gated startup, required secrets로 교체합니다. |

#### 해당 SHA에서 확인할 실제 코드

- parent의 source-mounted `docker-compose.yml`과 비교해 API/Web/Caddy `build`, bind mount 제거, startup install/build 제거, internal `expose`와 단일 Caddy host port를 확인합니다.
- DB `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?required}`와 API `SESSION_SECRET: ${SESSION_SECRET:?required}`가 Compose interpolation 단계에서 fail-fast하는지 확인합니다.
- `migrate` service가 API image의 `node packages/db/dist/cli.js migrate`를 one-shot으로 실행하고 DB `service_healthy` 뒤, API는 `service_completed_successfully` 뒤 시작하는지 확인합니다.
- API/Web HTTP healthcheck와 Caddy의 `service_healthy` dependencies가 DB → migrate → API → Web → gateway ordering을 형성하는지 확인합니다.
- DB volume 외 source/node_modules/.next volume이 제거되고 rollback/backup/automatic migration recovery가 별도로 없다는 점을 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API/Web image는 존재했지만 Compose는 source bind mount, startup install/build, 여러 host port와 Caddy config bind mount를 사용했습니다. migration이 독립 startup gate가 아니었고 required secret에 fail-fast interpolation이 없었습니다. |
| 해결하려던 문제 | 배포 runtime이 host source와 network install에 의존하고 API가 schema 준비 전에 시작할 수 있었습니다. 내부 DB/API/Web port가 host에 공개되고 gateway가 unhealthy upstream보다 먼저 열릴 수 있었습니다. 기본 credential로 잘못된 production이 시작할 위험도 있었습니다. |
| 핵심 결정 | Compose를 built API/Web/Caddy images로 교체하고 DB password/session secret을 required interpolation로 만들었습니다. API image를 재사용하는 one-shot migrate service, HTTP healthchecks, dependency conditions, internal `expose`, Caddy만 8080 publish를 구성했습니다. |
| build → package → execute 흐름 | Compose interpolation에서 secret 검사 → DB volume/container → `pg_isready` healthy → migrate container가 compiled CLI 실행 후 exit 0 → API image start·readiness → Web image start·readiness → Caddy image start·host 8080 publish. failure 단계 뒤 service는 condition 때문에 시작하지 않습니다. |
| ownership/lifetime/cleanup | DB service와 named volume이 durable data를, one-shot migrate process가 schema transition을, API/Web images가 immutable artifact를, Caddy가 외부 port를 소유합니다. Compose가 dependency graph와 container lifetime을 소유합니다. |
| failure/rollback/fail-closed | required interpolation 누락은 container creation 전 실패합니다. DB unhealthy는 migration을, migration non-zero는 API를, API/Web unhealthy는 downstream service를 차단합니다. 자동 rollback, backup restore, partially applied migration 복구는 정의하지 않습니다. |
| 보장하는 것 | production 조립에서 source/startup build를 제거하고 schema completion·readiness에 의해 공개 순서를 fail-closed로 만듭니다. host에는 gateway만 publish됩니다. |
| 보장하지 않는 것 | health endpoint의 의미가 실제 traffic readiness와 완전히 같다는 보장, migration atomicity/rollback, orchestrator 다중 replica ordering, TLS·secret manager·image signature는 없습니다. 종료 grace도 아직 application drain과 정렬되지 않았습니다. |
| 후속 연결 | `e2c12ded1d5f`가 rendered Compose/Dockerfile 정적 계약을 추가합니다. `312ddbc6fbe2`는 종료 시 application drain보다 짧은 container grace 위험을 수정합니다. |

비교 기준:
- 직전 관련 SHA: `656893e8e1cb` — `build(docker): production Web image 구성`
- 다음 관련 SHA: `e2c12ded1d5f` — `test(docker): production container contract 검증`

### 5.4. `test(docker): production container contract 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `e2c12ded1d5f` |
| Importance | B |
| Tags | OPERATIONS, OBSERVABILITY, TEST |
| Source에서 확정된 역할 | rendered Compose와 Dockerfile이 production image/lifecycle 규칙을 유지하는지 검사합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/docker-production.test.mjs`가 required env를 주입해 `docker compose config`를 render하고, secret 누락 시 render 실패를 별도로 검사하는 방식을 확인합니다.
- rendered model에서 migrate completion, API/Web health dependency, published port가 Caddy뿐인지, bind mount가 없는지 검사하는 assertion을 확인합니다.
- API/Web Dockerfile source text에서 exact Node/toolchain, frozen install, non-root runner, expected CMD와 Caddy metrics block을 정적으로 검사하는지 확인합니다.
- test가 image를 build하거나 container를 기동하지 않으므로 healthcheck 성공, migration 실행, network isolation의 runtime 결과는 증명하지 않는다고 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production Compose 구조는 사람이 YAML/Dockerfile을 읽어야 했고, source mount 재도입·secret default·dependency condition 삭제 같은 회귀가 build 단계에서 자동으로 차단되지 않았습니다. |
| 해결하려던 문제 | 구성 파일은 문법적으로 유효해도 delivery invariant를 깨뜨릴 수 있습니다. 특히 Compose interpolation 이후 실제 model을 기준으로 port, mount, dependency를 확인할 필요가 있었습니다. |
| 핵심 결정 | Node test가 `docker compose config` 성공/실패와 rendered model을 검사하고 Dockerfile/Caddyfile source assertion을 결합했습니다. |
| build → package → execute 흐름 | test env 구성 → required secret 누락 case는 compose config non-zero 기대 → 정상 env로 rendered config parse → service/dependency/port/volume assertion → Dockerfile/Caddyfile text assertion. |
| ownership/lifetime/cleanup | test process가 temporary command output과 parsed model을 소유하며 production files는 읽기 전용입니다. failure evidence는 assertion/exit status로 test runner에 반환됩니다. |
| failure/rollback/fail-closed | command 실행 불가 또는 assertion mismatch면 test가 실패합니다. 이 환경에서는 해당 command를 실제 실행하지 않았고 exact SHA source만 검사했습니다. |
| 보장하는 것 | 테스트를 실행하는 환경에서는 selected production Compose/image/gateway 정적 규칙의 회귀를 결정적으로 감지합니다. |
| 보장하지 않는 것 | Docker daemon에서 image가 build되는지, healthcheck가 통과하는지, signal/cleanup, migration·browser traffic이 동작하는지는 증명하지 않습니다. |
| 후속 연결 | `312ddbc6fbe2`가 기존 contract가 보지 못하던 termination budget을 수정하고 `73ba979841cd`가 새 duration assertion을 추가합니다. |

비교 기준:
- 직전 관련 SHA: `2c44cb7cd71f` — `build(docker): production container lifecycle 구성`
- 다음 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`

### 5.5. `fix(runtime): container 종료 유예를 room drain과 정렬`

| 항목 | 값 |
| --- | --- |
| SHA | `312ddbc6fbe2` |
| Importance | A |
| Tags | REALTIME, OPERATIONS, RISK |
| Source에서 확정된 역할 | API container termination grace를 application room-drain budget보다 길게 맞춥니다. |

#### 해당 SHA에서 확인할 실제 코드

- `docker-compose.yml`의 API service에 `stop_grace_period: 70s`가 추가되는 exact diff를 확인합니다.
- 동일 historical state의 application room drain budget 60초와 container orchestrator의 SIGTERM→grace→SIGKILL sequence를 연결합니다.
- 이전 Compose가 명시적 grace를 두지 않아 Docker 기본 timeout이 application cleanup 완료 전 강제 종료할 수 있었던 가정을 복원합니다.
- 70초가 60초 drain보다 10초 큰 이유를 process close와 scheduling overhead를 위한 최소 headroom으로 해석하되 실제 종료 시간 측정은 없다고 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production lifecycle은 startup readiness를 갖췄지만 API service에 `stop_grace_period`가 없었습니다. application은 room drain에 최대 60초를 사용할 수 있었으므로 container 기본 종료 예산이 더 짧을 수 있었습니다. |
| 해결하려던 문제 | orchestrator가 SIGTERM 뒤 grace 만료 시 SIGKILL하면 active room drain, final snapshot/result handoff, socket close가 중간에 끊길 수 있습니다. application의 graceful shutdown 보장은 외부 process manager 예산보다 길 수 없습니다. |
| 핵심 결정 | API service에 70초 stop grace를 명시해 60초 room-drain budget보다 긴 container termination window를 부여했습니다. |
| build → package → execute 흐름 | Compose stop → API process에 SIGTERM → application이 새 admission을 닫고 room drain/connection cleanup 수행 → 최대 60초 budget 안에 server close/exit 기대 → container는 70초까지 기다린 뒤에만 강제 종료 가능. |
| ownership/lifetime/cleanup | application이 room/session cleanup state를, Compose/Docker가 signal과 최종 process lifetime을 소유합니다. 수정은 두 owner의 timeout 계약을 정렬합니다. |
| failure/rollback/fail-closed | 이전에는 Docker grace가 먼저 만료해 cleanup을 절단할 위험이 있었습니다. 수정 후에도 application이 70초 안에 exit하지 못하면 SIGKILL될 수 있으며, crash/OOM에는 graceful path가 적용되지 않습니다. |
| 보장하는 것 | 정적 configuration에서 container grace가 known 60초 drain budget보다 깁니다. 외부 runtime이 Compose semantics를 따른다면 application에 명시된 cleanup window를 제공합니다. |
| 보장하지 않는 것 | 실제 room이 60초 안에 drain되는지, 모든 signal handler가 완료되는지, process가 70초 전에 exit하는지, 강제 종료 후 data recovery가 가능한지는 이 commit이 증명하지 않습니다. |
| 후속 연결 | `73ba979841cd`가 stop grace duration을 parse해 60초 이상이라는 회귀 계약을 추가합니다. |

비교 기준:
- 직전 관련 SHA: `e2c12ded1d5f` — `test(docker): production container contract 검증`
- 다음 관련 SHA: `73ba979841cd` — `test(docker): API 종료 유예 계약 검증`

### 5.6. `test(docker): API 종료 유예 계약 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `73ba979841cd` |
| Importance | B |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | Compose의 stop grace가 60초 application drain budget 이상인지 회귀 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/docker-production.test.mjs`의 duration parser가 `s`, `m` 등 Compose duration을 seconds로 변환하는 실제 구현을 확인합니다.
- rendered API service의 `stop_grace_period`가 존재하고 parsed value가 `>= 60`인지 assertion하는지 확인합니다.
- 정적 duration contract는 SIGTERM delivery, active room drain, actual exit time이나 SIGKILL 부재를 관찰하지 않는다고 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `312ddbc6fbe2`가 70초 grace를 추가했지만 기존 Docker contract test는 그 값을 보지 않아 후속 YAML 수정으로 삭제·축소돼도 회귀를 놓칠 수 있었습니다. |
| 해결하려던 문제 | application drain과 container timeout의 교차 계층 invariant가 문서/코드 상수에만 분산돼 있었습니다. |
| 핵심 결정 | contract test에 duration parser와 API stop grace `>= 60s` assertion을 추가했습니다. |
| build → package → execute 흐름 | Compose config render → API service stop grace 문자열/normalized value 획득 → seconds 변환 → 60 이상 assertion → 미지정·짧은 값이면 test failure. |
| ownership/lifetime/cleanup | test가 cross-layer timeout relation의 정적 evidence를 소유하고 Compose가 actual runtime timeout을 소유합니다. |
| failure/rollback/fail-closed | 값이 없거나 parser가 지원하지 않는 형식이거나 60초 미만이면 contract test가 실패합니다. runtime signal test나 cleanup observation은 없습니다. |
| 보장하는 것 | 테스트 실행 시 known application drain budget보다 짧은 Compose grace 회귀를 차단합니다. |
| 보장하지 않는 것 | graceful shutdown 성공, room/result persistence, 실제 signal timing은 증명하지 않습니다. |
| 후속 연결 | Thread 04의 CI static contract command가 이 test를 workflow에서 실행하도록 연결합니다. |

비교 기준:
- 직전 관련 SHA: `312ddbc6fbe2` — `fix(runtime): container 종료 유예를 room drain과 정렬`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `f8efb2656771` | shared/db/API를 build한 뒤 non-root runner에 필요한 결과만 복사하는 multi-stage API image를 만듭니다. | 도입 | `.dockerignore`와 `apps/api/Dockerfile`이 build-time producer와 non-root compiled runtime을 분리합니다. |
| `656893e8e1cb` | Next.js standalone output과 static asset만 runner에 포함하는 multi-stage Web image를 만듭니다. | 확장 | `apps/web/Dockerfile`이 public config의 build-time lifetime과 standalone runner를 production image로 고정합니다. |
| `2c44cb7cd71f` | source-mounted Compose를 built images, one-shot migration, health-gated startup, required secrets로 교체합니다. | 도입 | `docker-compose.yml`의 required interpolation, one-shot migration, health conditions, image builds와 단일 published gateway가 production lifecycle을 정의합니다. |
| `e2c12ded1d5f` | rendered Compose와 Dockerfile이 production image/lifecycle 규칙을 유지하는지 검사합니다. | 검증·불충분 | `tests/docker-production.test.mjs`가 lifecycle 구조를 정적으로 보호하지만 stop grace와 실제 runtime behavior는 아직 검사하지 않습니다. |
| `312ddbc6fbe2` | API container termination grace를 application room-drain budget보다 길게 맞춥니다. | 수정 | API `stop_grace_period: 70s`가 orchestrator termination budget을 application 60초 drain invariant보다 크게 만듭니다. |
| `73ba979841cd` | Compose의 stop grace가 60초 application drain budget 이상인지 회귀 검증합니다. | 회귀 검증 | duration parser와 `>= 60s` assertion이 수정된 termination invariant를 정적 contract로 보호합니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| source mount·startup install/build·다중 host port에 의존한 runtime | `2c44cb7cd71f` | `e2c12ded1d5f` | built image, one-shot migration, readiness dependencies, required secret, 단일 gateway port로 전환하고 rendered config를 정적으로 검사합니다. |
| application 60초 drain보다 container termination grace가 짧을 수 있음 | `312ddbc6fbe2` | `73ba979841cd` | API grace를 70초로 올리고 test가 60초 이상인지 검사합니다. actual signal/drain 성공은 별도 runtime evidence가 필요합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| API/Web image artifact | multi-stage Docker builder | non-root Node runner | image/container lifetime | install/build failure 시 image 생성 차단; runtime health는 Compose가 관찰 |
| schema migration | compiled DB CLI를 실행하는 `migrate` service | PostgreSQL | one-shot container lifetime | non-zero exit면 API `service_completed_successfully` dependency가 차단 |
| secret/config | Compose caller env와 Web build args | DB/API/Web/Caddy | Compose project 또는 image build lifetime | required DB password/session secret 누락은 interpolation fail-fast |
| termination evidence | Compose `stop_grace_period`와 static test | Docker daemon/test runner | container stop 또는 test process lifetime | 60초 미만은 test 실패; actual cleanup failure는 측정하지 않음 |

## 9. Thread 최종 상태

- 최종 delivery owner: Dockerfiles가 immutable API/Web artifacts를, Compose가 DB/migration/readiness/gateway/termination lifecycle을, static contract test가 selected configuration invariant를 소유합니다.
- source와 production artifact의 관계: runtime container에는 source bind mount나 startup build가 없고 builder가 만든 compiled/standalone artifact만 runner에 복사됩니다.
- build-time과 runtime configuration의 관계: Web `NEXT_PUBLIC_*`는 image build-time, DB URL과 session secret은 container runtime입니다. required secret은 Compose interpolation에서 먼저 검사됩니다.
- startup/readiness/shutdown contract: DB healthy → migration success → API healthy → Web healthy → Caddy publish이며, API stop grace는 70초로 60초 room drain보다 깁니다.
- fail-closed 조건: secret 누락, DB unhealthy, migration non-zero, API/Web unhealthy는 downstream 공개를 막습니다. grace 60초 미만은 contract test를 실패시킵니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: exact SHA Dockerfile/Compose/test source와 diff를 검사했습니다. Docker daemon을 사용한 image build, Compose startup, signal/drain command는 실행하지 않았습니다.

## 10. 최종 execution/delivery flow

```text
build context (`.dockerignore`)
→ dependencies stage (frozen workspace install)
→ builder (shared → DB/API 또는 shared → Next)
→ non-root API/Web runner images
Compose interpolation: required secrets
→ PostgreSQL healthy
→ one-shot compiled migration exit 0
→ API healthcheck healthy
→ Web healthcheck healthy
→ Caddy image가 :8080만 publish
stop: SIGTERM → application drain ≤ 60s → expected process exit before 70s grace
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `02-production-build-and-package-artifacts.md`: image가 복사하는 compiled artifact의 생성 규칙
- `01-runtime-composition-and-reverse-proxy-evolution.md`: 초기 source-mounted Compose와 gateway의 출발점
- `07-runtime-observability-and-service-health`: readiness, drain, graceful shutdown의 application semantics

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 03-container-images-and-production-runtime-lifecycle.md =====

===== BEGIN FILE: 04-ci-production-process-and-browser-delivery-verification.md =====
# CI production process와 browser delivery 검증

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

typecheck/unit/build 수준의 CI에서 production artifact를 실제 process로 기동하고 PostgreSQL migration, readiness, HTTP/WebSocket smoke, production Web, browser E2E까지 연결한 delivery verification으로 발전하는 과정을 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- production delivery 검증은 source dev server가 아니라 build된 artifact/process를 실행해야 합니다.
- CI는 동일한 Node/pnpm toolchain과 frozen lockfile을 사용해 dependency resolution drift를 제한합니다.
- 실제 PostgreSQL을 준비하고 migration한 뒤 API readiness를 확인한 후 smoke/browser 검증을 진행합니다.
- guest/demo browser job도 production-style build/start 경로를 사용하되 PostgreSQL 없이 제한된 capability를 검증합니다.
- cookie origin은 HTTP session semantics와 일치해야 하며 CI의 hostname 차이도 contract로 취급합니다.
- workflow 자체의 필수 job/command/runtime 규칙은 정적 contract test로 보호됩니다.

## 2. 핵심 질문

- 초기 CI가 typecheck/unit/build만 수행할 때 production delivery에 대해 증명하지 못하는 것은 무엇입니까?
- PostgreSQL integration job과 process/browser job은 각각 어떤 persistence·delivery 경계를 검증합니까?
- process/browser job은 build, DB 준비, migration, process start, readiness, smoke, E2E를 어떤 순서로 수행합니까?
- workflow contract test는 Node/pnpm, frozen install, command 존재를 어떻게 고정합니까?
- guest-only browser job이 일반 process job과 분리되는 이유와 공유하는 production build/start 계약은 무엇입니까?
- `localhost`와 `127.0.0.1` 차이가 cookie origin에서 실제 failure가 되는 이유는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a9f8b8609711` | `build(repo): workspace 검증 명령 정리` | B | PROTOCOL, REALTIME, OPERATIONS | root script와 Make target을 verification layer별 entrypoint로 정리합니다. |
| 2 | `68404e51ea53` | `ci(repo): typecheck·unit·build workflow 추가` | B | PROTOCOL, PERSISTENCE, OPERATIONS | push/PR에서 typecheck, unit, build를 실행하는 첫 repository CI 경계를 추가합니다. |
| 3 | `0e360d333540` | `ci(db): PostgreSQL integration 검사 실행` | B | PERSISTENCE | PostgreSQL integration test를 독립 CI job에서 실행합니다. |
| 4 | `3367b4266049` | `ci(repo): process와 browser 검증 job 추가` | A | REALTIME, PERSISTENCE, WEB | production artifact와 PostgreSQL을 실제로 기동해 HTTP/WS smoke와 browser E2E까지 검증합니다. |
| 5 | `7cb0d32b5be3` | `test(ci): process 검증 job contract 확인` | B | PERSISTENCE, WEB, OPERATIONS | workflow의 toolchain, frozen install, unit/integration/process/browser command 존재를 contract로 고정합니다. |
| 6 | `bf0bc1199c84` | `ci(e2e): 비회원 체험 browser job 실행` | B | PERSISTENCE, WEB, OPERATIONS | API/Web을 demo mode로 build/start하고 PostgreSQL 없이 guest-only Playwright suite를 실행합니다. |
| 7 | `0ae1ded2c56f` | `test(ci): guest browser job 요구 검증` | B | WEB, OPERATIONS, TEST | 별도 guest job, mode 값, Playwright command와 guest spec 존재를 workflow contract로 검증합니다. |
| 8 | `f35728f4ef92` | `test(repo): 정적 계약 검사 명령 연결` | B | PERSISTENCE, OPERATIONS, TEST | CI, production Docker, load harness contract suite를 하나의 root command와 Make target으로 연결합니다. |
| 9 | `4f9e66c35586` | `ci(repo): 정적 계약 검사 실행` | B | PERSISTENCE | repository static contract suite를 CI에서 실행합니다. |
| 10 | `65512bc24161` | `fix(ci): 브라우저 E2E API origin 정렬` | B | AUTH, REALTIME, WEB | browser E2E API origin을 cookie semantics와 맞는 `localhost`로 정렬합니다. |
| 11 | `527921bc9d69` | `test(ci): 브라우저 E2E cookie origin 계약 검증` | B | AUTH, WEB, OPERATIONS | CI contract가 API base URL의 exact cookie origin을 고정하도록 합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(repo): workspace 검증 명령 정리`

| 항목 | 값 |
| --- | --- |
| SHA | `a9f8b8609711` |
| Importance | B |
| Tags | PROTOCOL, REALTIME, OPERATIONS |
| Source에서 확정된 역할 | root script와 Make target을 verification layer별 entrypoint로 정리합니다. |

#### 해당 SHA에서 확인할 실제 코드

- root `package.json`에서 typecheck, unit, PostgreSQL integration, build, aggregate test command가 어떤 workspace command를 호출하는지 확인합니다.
- `Makefile`의 `check`, `integration`, `build`, `test` target이 package script의 stable operator entrypoint가 되는지 확인합니다.
- unit과 실제 PostgreSQL integration이 별도 command로 분리되어 필요한 external resource와 failure scope를 구분하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | workspace마다 검증 command는 존재했지만 repository 사용자가 typecheck, unit, PostgreSQL integration, build를 어떤 순서·경계로 호출해야 하는지 root entrypoint가 정리되지 않았습니다. |
| 해결하려던 문제 | CI와 개발자가 서로 다른 command 조합을 사용하면 일부 verification layer가 누락되고, unit failure와 external DB integration failure를 구분하기 어렵습니다. |
| 핵심 결정 | root package script와 Make targets를 verification layer별로 명명하고 aggregate command가 동일 entrypoint를 재사용하도록 정리했습니다. |
| build → package → execute 흐름 | operator/CI → root script 또는 Make target → workspace typecheck/unit/postgres-integration/build. 각 command의 exit status가 상위 shell로 전달됩니다. |
| ownership/lifetime/cleanup | root package metadata가 canonical command graph를, Makefile이 CLI alias를, 각 workspace가 실제 test/build implementation을 소유합니다. |
| failure/rollback/fail-closed | 하위 command non-zero면 상위 target이 실패합니다. 이 commit은 CI runner, database service, browser process를 만들지 않습니다. |
| 보장하는 것 | 검증 층을 반복 가능한 repository command로 호출할 수 있고 unit과 DB integration 경계를 구분합니다. |
| 보장하지 않는 것 | 어떤 command가 push/PR에서 자동 실행되는지, production process가 시작되는지, external PostgreSQL이 준비되는지는 보장하지 않습니다. |
| 후속 연결 | `68404e51ea53`이 typecheck/unit/build를 첫 CI job에 연결하고 `0e360d333540`이 PostgreSQL integration을 독립 job으로 추가합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `68404e51ea53` — `ci(repo): typecheck·unit·build workflow 추가`

### 5.2. `ci(repo): typecheck·unit·build workflow 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `68404e51ea53` |
| Importance | B |
| Tags | PROTOCOL, PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | push/PR에서 typecheck, unit, build를 실행하는 첫 repository CI 경계를 추가합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 새 `.github/workflows/ci.yml`의 push/pull_request trigger, verify job, timeout, checkout→pnpm→Node→frozen install 순서를 확인합니다.
- toolchain이 pnpm `10.32.1`, Node `24.18.0`으로 고정되고 cache key가 lockfile과 연결되는지 확인합니다.
- verify job이 `pnpm typecheck`, `pnpm unit`, `pnpm build`만 실행해 PostgreSQL·process startup·browser delivery는 아직 검증하지 않는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | repository command는 정리됐지만 push/PR에 대한 자동 gate가 없었습니다. |
| 해결하려던 문제 | review 전에 type error, deterministic unit failure, build breakage가 합쳐지지 않을 수 있었고 toolchain/install 방식도 사용자 환경에 따라 달랐습니다. |
| 핵심 결정 | GitHub Actions verify job이 exact pnpm/Node와 frozen lockfile을 사용해 typecheck→unit→build를 실행하도록 했습니다. |
| build → package → execute 흐름 | push/PR → hosted runner checkout → pnpm setup → Node setup/cache → `pnpm install --frozen-lockfile` → typecheck → unit → build. |
| ownership/lifetime/cleanup | workflow가 toolchain과 step order를, hosted runner가 temporary dependency/artifact filesystem을, job status가 merge signal을 소유합니다. |
| failure/rollback/fail-closed | install/typecheck/unit/build 중 하나라도 non-zero면 job이 종료됩니다. runner 종료 시 workspace는 폐기되며 별도 rollback이 필요하지 않습니다. |
| 보장하는 것 | 동일 pinned toolchain과 lockfile로 static/type/unit/build gate를 push/PR에 적용합니다. |
| 보장하지 않는 것 | 실제 PostgreSQL backend, migration, compiled process start, health/readiness, HTTP/WS/browser traffic은 검증하지 않습니다. |
| 후속 연결 | `0e360d333540`이 DB integration을 별도 job으로 추가하고 `3367b4266049`가 production-style process/browser delivery를 추가합니다. |

비교 기준:
- 직전 관련 SHA: `a9f8b8609711` — `build(repo): workspace 검증 명령 정리`
- 다음 관련 SHA: `0e360d333540` — `ci(db): PostgreSQL integration 검사 실행`

### 5.3. `ci(db): PostgreSQL integration 검사 실행`

| 항목 | 값 |
| --- | --- |
| SHA | `0e360d333540` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | PostgreSQL integration test를 독립 CI job에서 실행합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`의 새 `postgres-integration` job 이름, 15분 timeout과 verify job과 독립된 execution boundary를 확인합니다.
- checkout, pnpm `10.32.1`, Node `24.18.0`, frozen install 뒤 `pnpm postgres-integration`을 실행하는지 확인합니다.
- 이 job diff에는 PostgreSQL service 선언이 보이지 않으므로 test command 자체가 어떤 DB provision mechanism을 사용하는지 해당 SHA의 package/test source에서 확인해야 함을 기록합니다.
- 후속 `7cb0d32b5be3` contract가 이 독립 job/command 존재를 요구하는 연결을 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 첫 CI는 DB-independent unit/build만 실행했습니다. root에는 PostgreSQL integration command가 있었지만 workflow consumer가 없었습니다. |
| 해결하려던 문제 | memory repository나 mock을 통과해도 SQL schema, PostgreSQL query/transaction behavior가 깨질 수 있었습니다. |
| 핵심 결정 | `postgres-integration`을 verify와 분리된 15분 job으로 추가하고 동일 pinned toolchain/frozen install 뒤 root integration command를 실행했습니다. |
| build → package → execute 흐름 | push/PR → independent runner setup/install → `pnpm postgres-integration` → test command가 준비한 PostgreSQL integration path 실행 → exit status를 job에 반영합니다. |
| ownership/lifetime/cleanup | workflow가 job isolation과 timeout을, integration harness가 DB resource lifecycle을, runner가 dependency workspace를 소유합니다. |
| failure/rollback/fail-closed | DB provision/connect/query/test failure는 해당 job만 non-zero로 만듭니다. verify job과 분리되어 failure category를 식별할 수 있습니다. |
| 보장하는 것 | CI에 PostgreSQL-backed integration command가 별도 mandatory status로 존재합니다. |
| 보장하지 않는 것 | production API/Web process, migration-before-start ordering, browser journey를 검증하지 않습니다. 이 diff만으로 external DB provision의 모든 세부도 확정할 수 없습니다. |
| 후속 연결 | `3367b4266049`가 명시적 PostgreSQL service, migration, compiled API/Web, smoke/E2E를 하나의 delivery job에 연결합니다. |

비교 기준:
- 직전 관련 SHA: `68404e51ea53` — `ci(repo): typecheck·unit·build workflow 추가`
- 다음 관련 SHA: `3367b4266049` — `ci(repo): process와 browser 검증 job 추가`

### 5.4. `ci(repo): process와 browser 검증 job 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `3367b4266049` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, WEB |
| Source에서 확정된 역할 | production artifact와 PostgreSQL을 실제로 기동해 HTTP/WS smoke와 browser E2E까지 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`의 `process-and-browser` job PostgreSQL service, health option, environment와 25분 timeout을 확인합니다.
- frozen install/build 뒤 package migration command, compiled API `dist` start, Next production/standalone Web start를 background process로 기동하는 shell을 확인합니다.
- PID capture와 `trap`/`kill` cleanup, API readiness와 Web root polling이 downstream smoke/E2E 전에 실행되는지 확인합니다.
- HTTP smoke, WebSocket smoke, Playwright browser suite의 실제 command와 public API/WS build-time values를 연결합니다.
- compiled production artifact를 실행하지만 `APP_MODE=development`와 package migration command를 사용하므로 full production configuration/Compose image lifecycle과 동일하지 않다는 점을 명시합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | CI는 type/unit/build와 독립 PostgreSQL integration을 수행했지만 build artifact가 실제 Node/Next process로 시작되는지, migration 후 browser가 HTTP session과 WebSocket journey를 완료하는지는 보지 않았습니다. |
| 해결하려던 문제 | artifact path, environment, startup order, readiness, process cleanup, browser-visible URL의 통합 오류는 compile/unit 단계에서 드러나지 않습니다. release candidate가 build는 되지만 실행 불가능할 수 있었습니다. |
| 핵심 결정 | PostgreSQL service가 포함된 `process-and-browser` job을 추가했습니다. build와 migration 후 compiled API와 production Web을 background로 시작하고 readiness를 polling한 다음 HTTP/WS smoke와 Playwright를 실행하며 trap으로 process를 정리합니다. |
| build → package → execute 흐름 | pinned setup/frozen install → root production build → PostgreSQL health → package migration → API `dist` process + Web production process start/PID capture → readiness/root polling → HTTP smoke → WS smoke → browser E2E → trap이 background process 종료. |
| ownership/lifetime/cleanup | Actions service container가 PostgreSQL lifetime을, shell이 API/Web PID와 cleanup trap을, build가 artifact를, smoke/Playwright가 client connection lifetime을 소유합니다. hosted runner 종료가 최종 cleanup boundary입니다. |
| failure/rollback/fail-closed | DB health/migration/build/start/readiness/smoke/E2E 어느 단계든 non-zero면 job이 실패하고 trap이 시작된 process를 kill합니다. readiness timeout은 downstream test를 차단합니다. crash 전 partial DB state rollback은 정의하지 않습니다. |
| 보장하는 것 | CI 환경에서 compiled API와 production-style Web이 실제 PostgreSQL schema 위에서 시작하고 HTTP/WS/browser 경로가 성공해야 합니다. 이는 단순 static contract보다 강한 실행 증거입니다. |
| 보장하지 않는 것 | Docker image/Compose networking, `APP_MODE=production`의 fail-closed DB contract, TLS, production secret manager, real deployment load를 증명하지 않습니다. `APP_MODE=development`를 사용하므로 production configuration 전체와 동일하지 않습니다. |
| 후속 연결 | `7cb0d32b5be3`가 이 workflow의 필수 job/command를 정적으로 고정하고 `bf0bc1199c84`가 DB 없는 demo guest delivery를 별도 job으로 추가합니다. |

비교 기준:
- 직전 관련 SHA: `0e360d333540` — `ci(db): PostgreSQL integration 검사 실행`
- 다음 관련 SHA: `7cb0d32b5be3` — `test(ci): process 검증 job contract 확인`

### 5.5. `test(ci): process 검증 job contract 확인`

| 항목 | 값 |
| --- | --- |
| SHA | `7cb0d32b5be3` |
| Importance | B |
| Tags | PERSISTENCE, WEB, OPERATIONS |
| Source에서 확정된 역할 | workflow의 toolchain, frozen install, unit/integration/process/browser command 존재를 contract로 고정합니다. |

#### 해당 SHA에서 확인할 실제 코드

- 새 `tests/ci-contract.test.mjs`가 `.github/workflows/ci.yml` text를 읽고 exact Node/pnpm version, frozen install을 assertion하는지 확인합니다.
- verify, PostgreSQL integration, process/browser job 이름과 production build/start, migration, smoke, Playwright command literal을 검사하는지 확인합니다.
- YAML semantic execution이 아니라 text/literal contract이므로 command가 실제 성공하거나 correct order로 실행된다는 것을 독립적으로 증명하지 않는 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 실행 job은 존재했지만 workflow edit로 pinned version, frozen install, separate DB job, production process command가 삭제돼도 해당 YAML을 별도로 검사하는 unit contract가 없었습니다. |
| 해결하려던 문제 | delivery verification 자체가 configuration file이므로, test code 변경 없이 CI 경계가 약화되는 회귀를 빠르게 감지할 필요가 있었습니다. |
| 핵심 결정 | Node test가 workflow source의 필수 literal과 job/command를 정적으로 assertion하도록 했습니다. |
| build → package → execute 흐름 | test runner → workflow text read → version/install/job/command assertion → mismatch 시 non-zero. workflow process는 이 test에서 기동하지 않습니다. |
| ownership/lifetime/cleanup | contract test가 workflow source schema의 selected requirements를 소유하고 Actions가 actual execution을 소유합니다. |
| failure/rollback/fail-closed | literal rename/삭제/version drift면 test가 실패합니다. YAML이 문법적으로 실행 가능하더라도 assertion 밖의 ordering/permission/cache 문제는 놓칠 수 있습니다. |
| 보장하는 것 | 테스트 실행 시 pinned toolchain, frozen install, unit/DB/process/browser command의 존재가 정적으로 보호됩니다. |
| 보장하지 않는 것 | Actions service health, process startup, browser result, command semantics는 이 static test가 증명하지 않습니다. |
| 후속 연결 | `bf0bc1199c84`와 `0ae1ded2c56f`가 guest job과 그 정적 contract를 추가합니다. `f35728f4ef92`가 모든 static contract를 root command로 묶습니다. |

비교 기준:
- 직전 관련 SHA: `3367b4266049` — `ci(repo): process와 browser 검증 job 추가`
- 다음 관련 SHA: `bf0bc1199c84` — `ci(e2e): 비회원 체험 browser job 실행`

### 5.6. `ci(e2e): 비회원 체험 browser job 실행`

| 항목 | 값 |
| --- | --- |
| SHA | `bf0bc1199c84` |
| Importance | B |
| Tags | PERSISTENCE, WEB, OPERATIONS |
| Source에서 확정된 역할 | API/Web을 demo mode로 build/start하고 PostgreSQL 없이 guest-only Playwright suite를 실행합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml`의 새 guest browser job이 PostgreSQL service 없이 `APP_MODE=demo`와 guest/session secret을 사용하는지 확인합니다.
- demo public values로 Web을 build하고 API `dist`와 Web standalone/production server를 기동한 뒤 readiness와 `pnpm e2e:guest-demo`를 실행하는지 확인합니다.
- 일반 process/browser job과 PID/trap/readiness pattern을 공유하면서 persistence capability를 의도적으로 제거한 차이를 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 일반 browser job은 PostgreSQL과 authenticated/full application path에 묶여 있어 DB 없이 제공하는 guest/demo delivery contract를 독립적으로 검증하지 못했습니다. |
| 해결하려던 문제 | demo mode가 memory-backed 제한 capability로 기동돼야 하는데 일반 job 성공만으로는 DB absence, guest secret, guest-only browser journey가 유지되는지 알 수 없었습니다. |
| 핵심 결정 | PostgreSQL service 없이 `APP_MODE=demo`로 API/Web production artifacts를 build/start하고 guest Playwright spec만 실행하는 별도 job을 추가했습니다. |
| build → package → execute 흐름 | pinned setup/install → demo public env로 build → compiled API와 production Web start → readiness → guest-only E2E → trap cleanup. |
| ownership/lifetime/cleanup | job shell이 in-memory API process와 Web process/PID를, guest test가 temporary browser/session을 소유합니다. persistent DB resource는 의도적으로 없습니다. |
| failure/rollback/fail-closed | demo configuration/start/readiness/guest journey가 실패하면 job이 non-zero입니다. trap이 process를 정리합니다. |
| 보장하는 것 | DB 없이도 intended demo production-style process와 guest browser journey가 CI에서 실행됩니다. |
| 보장하지 않는 것 | durable persistence, authenticated user journey, production DB requirement, Docker Compose는 이 job 범위가 아닙니다. |
| 후속 연결 | `0ae1ded2c56f`가 guest job, mode, command와 spec 존재를 static contract로 고정합니다. |

비교 기준:
- 직전 관련 SHA: `7cb0d32b5be3` — `test(ci): process 검증 job contract 확인`
- 다음 관련 SHA: `0ae1ded2c56f` — `test(ci): guest browser job 요구 검증`

### 5.7. `test(ci): guest browser job 요구 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `0ae1ded2c56f` |
| Importance | B |
| Tags | WEB, OPERATIONS, TEST |
| Source에서 확정된 역할 | 별도 guest job, mode 값, Playwright command와 guest spec 존재를 workflow contract로 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/ci-contract.test.mjs`에 guest job 이름, `APP_MODE: demo`, public app mode, guest Playwright command assertion이 추가되는지 확인합니다.
- guest spec file 존재 검사와 workflow text literal 검사가 어떤 producer/consumer 경계를 보호하는지 확인합니다.
- static assertion이 guest browser process를 실제로 실행하거나 DB 비접속을 network level에서 증명하지 않는다고 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest job은 실행됐지만 workflow에서 job/mode/spec가 제거되거나 full E2E로 잘못 대체돼도 별도 source-level guard가 없었습니다. |
| 해결하려던 문제 | demo delivery는 production/full mode와 다른 capability contract이므로 generic process job assertion만으로 보호할 수 없습니다. |
| 핵심 결정 | CI contract test에 guest job·demo env·guest command·spec existence assertions를 추가했습니다. |
| build → package → execute 흐름 | contract test가 workflow/spec source read → guest job/mode/command literal 및 file existence 검사 → mismatch 시 failure. |
| ownership/lifetime/cleanup | static test가 guest delivery configuration contract를, actual guest job이 process/browser execution을 소유합니다. |
| failure/rollback/fail-closed | job rename, mode drift, command/spec 삭제 시 contract test가 실패합니다. |
| 보장하는 것 | 테스트 실행 시 guest-only CI path의 필수 configuration이 존재합니다. |
| 보장하지 않는 것 | Playwright 성공, memory isolation, PostgreSQL 미사용을 runtime 관찰로 증명하지 않습니다. |
| 후속 연결 | `f35728f4ef92`가 CI/Docker/load static tests를 한 root contract command로 묶습니다. |

비교 기준:
- 직전 관련 SHA: `bf0bc1199c84` — `ci(e2e): 비회원 체험 browser job 실행`
- 다음 관련 SHA: `f35728f4ef92` — `test(repo): 정적 계약 검사 명령 연결`

### 5.8. `test(repo): 정적 계약 검사 명령 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `f35728f4ef92` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS, TEST |
| Source에서 확정된 역할 | CI, production Docker, load harness contract suite를 하나의 root command와 Make target으로 연결합니다. |

#### 해당 SHA에서 확인할 실제 코드

- root `package.json`의 `test:contracts`가 CI contract, Docker production contract, load/fault harness static tests를 어떤 순서로 호출하는지 확인합니다.
- `Makefile` target이 동일 package script를 재사용해 local/CI entrypoint drift를 줄이는지 확인합니다.
- 한 하위 static test의 non-zero가 aggregate command를 중단시키는 shell semantics와 실행 증거가 아닌 source contract 성격을 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | CI/Docker/guest/load 관련 정적 test가 개별 command로 흩어져 있어 일괄 실행을 누락할 수 있었습니다. |
| 해결하려던 문제 | release configuration 회귀를 확인하려면 여러 파일을 수동으로 호출해야 했고 CI에 연결할 안정적인 단일 entrypoint가 없었습니다. |
| 핵심 결정 | root `test:contracts`와 Make target이 selected static contract suites를 직렬 호출하도록 했습니다. |
| build → package → execute 흐름 | operator/CI → `pnpm test:contracts` → CI workflow assertions → Docker/Compose assertions → load/fault harness assertions; 첫 failure가 aggregate exit를 실패시킵니다. |
| ownership/lifetime/cleanup | root script가 static contract suite composition을, 각 test file이 domain assertion을 소유합니다. |
| failure/rollback/fail-closed | 하위 test non-zero면 aggregate command도 non-zero입니다. runtime process cleanup은 이 static suite의 책임이 아닙니다. |
| 보장하는 것 | 한 command로 주요 delivery configuration contract를 반복 실행할 수 있습니다. |
| 보장하지 않는 것 | production process/image/browser/load를 실제 실행하는 것은 아니며 assertion에 포함되지 않은 drift는 감지하지 못합니다. |
| 후속 연결 | `4f9e66c35586`이 이 root contract command를 verify CI job에 추가합니다. |

비교 기준:
- 직전 관련 SHA: `0ae1ded2c56f` — `test(ci): guest browser job 요구 검증`
- 다음 관련 SHA: `4f9e66c35586` — `ci(repo): 정적 계약 검사 실행`

### 5.9. `ci(repo): 정적 계약 검사 실행`

| 항목 | 값 |
| --- | --- |
| SHA | `4f9e66c35586` |
| Importance | B |
| Tags | PERSISTENCE |
| Source에서 확정된 역할 | repository static contract suite를 CI에서 실행합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml` verify job에서 unit 뒤, build 전 `pnpm test:contracts`가 실행되는 exact 위치를 확인합니다.
- static contract failure가 production build/process jobs 전에 repository gate를 fail-fast하는지 확인합니다.
- hosted runner에서 실제 command 결과를 이 환경에서는 재실행하지 않았음을 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | aggregate static contract command는 존재했지만 CI가 자동 호출하지 않았습니다. |
| 해결하려던 문제 | 개발자가 local contract suite를 생략하면 workflow/Docker/load configuration drift가 merge gate를 통과할 수 있었습니다. |
| 핵심 결정 | verify job에 `pnpm test:contracts` step을 추가했습니다. |
| build → package → execute 흐름 | frozen install → typecheck → unit → static contracts → production build. contract failure는 build와 후속 CI 결과 전에 job을 종료합니다. |
| ownership/lifetime/cleanup | CI workflow가 invocation을, test suite가 assertions를, job status가 gate evidence를 소유합니다. |
| failure/rollback/fail-closed | 어느 contract test든 non-zero면 verify job이 실패합니다. |
| 보장하는 것 | push/PR에서 selected static delivery contracts가 항상 실행되도록 workflow가 연결됩니다. |
| 보장하지 않는 것 | static pass가 process/browser/Docker runtime 성공을 대체하지 않습니다. |
| 후속 연결 | `65512bc24161`은 실제 browser process job에서 드러난 hostname/cookie boundary를 수정하고 `527921bc9d69`가 그 exact origin을 static contract에 추가합니다. |

비교 기준:
- 직전 관련 SHA: `f35728f4ef92` — `test(repo): 정적 계약 검사 명령 연결`
- 다음 관련 SHA: `65512bc24161` — `fix(ci): 브라우저 E2E API origin 정렬`

### 5.10. `fix(ci): 브라우저 E2E API origin 정렬`

| 항목 | 값 |
| --- | --- |
| SHA | `65512bc24161` |
| Importance | B |
| Tags | AUTH, REALTIME, WEB |
| Source에서 확정된 역할 | browser E2E API origin을 cookie semantics와 맞는 `localhost`로 정렬합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.github/workflows/ci.yml` process/browser env에서 `API_BASE_URL`이 `http://127.0.0.1:4000`에서 `http://localhost:4000`으로 바뀌는 exact diff를 확인합니다.
- browser가 Web/login을 `localhost`로 방문하고 host-only HttpOnly session cookie를 받은 뒤 API request host가 달라 cookie를 보내지 않는 previous failure를 연결합니다.
- WebSocket URL은 one-time ticket admission path라 동일 cookie transport 요구가 없는 범위를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Web과 login origin은 `localhost`였지만 browser E2E의 API base는 `127.0.0.1`이었습니다. 두 문자열은 같은 loopback interface라도 cookie host matching에서는 다른 host입니다. |
| 해결하려던 문제 | login response가 `localhost`에 host-only HttpOnly session cookie를 설정하면 browser는 `127.0.0.1` API request에 그 cookie를 보내지 않습니다. 인증된 E2E가 unauthorized로 실패할 수 있었습니다. |
| 핵심 결정 | API base URL을 `http://localhost:4000`으로 바꿔 browser page/login/API가 같은 cookie host를 사용하게 했습니다. |
| build → package → execute 흐름 | Playwright가 localhost Web 방문 → localhost API/login response가 session cookie 설정 → 이후 localhost API request에 cookie 자동 첨부 → authenticated HTTP flow → ticket을 받아 WS 연결. |
| ownership/lifetime/cleanup | browser cookie jar가 host-bound credential lifetime을, workflow env가 client base URL을, API가 session validation을 소유합니다. |
| failure/rollback/fail-closed | 이전 host mismatch는 network connectivity와 무관하게 cookie omission을 만들었습니다. 수정 후에도 scheme, port, SameSite/Secure/CORS 설정이 잘못되면 다른 인증 failure는 남습니다. |
| 보장하는 것 | CI browser HTTP calls의 host가 login cookie host와 일치합니다. |
| 보장하지 않는 것 | 외부 production domain, HTTPS cookie, proxy origin, all browser 정책을 검증하지 않습니다. WS origin은 별도 ticket contract를 따릅니다. |
| 후속 연결 | `527921bc9d69`가 workflow에서 exact `http://localhost:4000` literal을 요구해 127.0.0.1 회귀를 차단합니다. |

비교 기준:
- 직전 관련 SHA: `4f9e66c35586` — `ci(repo): 정적 계약 검사 실행`
- 다음 관련 SHA: `527921bc9d69` — `test(ci): 브라우저 E2E cookie origin 계약 검증`

### 5.11. `test(ci): 브라우저 E2E cookie origin 계약 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `527921bc9d69` |
| Importance | B |
| Tags | AUTH, WEB, OPERATIONS |
| Source에서 확정된 역할 | CI contract가 API base URL의 exact cookie origin을 고정하도록 합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/ci-contract.test.mjs`에 `API_BASE_URL: http://localhost:4000` exact assertion이 추가되는지 확인합니다.
- generic loopback reachability가 아니라 browser cookie host invariant를 literal로 보호하는 이유를 `65512bc24161`의 failure와 연결합니다.
- static source assertion은 실제 Set-Cookie, browser cookie attachment, authenticated response를 관찰하지 않는다고 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | origin fix는 workflow 한 줄에만 존재해 후속 정리에서 `127.0.0.1`로 되돌아가도 generic CI contract가 감지하지 못했습니다. |
| 해결하려던 문제 | hostname 차이는 network test가 아니라 credential transport contract이므로 exact value를 보호할 regression evidence가 필요했습니다. |
| 핵심 결정 | CI contract test가 API base URL의 exact localhost literal을 요구하도록 했습니다. |
| build → package → execute 흐름 | static test가 workflow text read → exact origin assertion → mismatch 시 test/verify job failure. actual browser proof는 process/browser job이 담당합니다. |
| ownership/lifetime/cleanup | static test가 configuration regression을, Playwright/browser가 runtime cookie behavior를 소유합니다. |
| failure/rollback/fail-closed | origin literal이 삭제·변경되면 contract suite가 실패합니다. |
| 보장하는 것 | 테스트 실행 시 known cookie-host-safe CI origin이 유지됩니다. |
| 보장하지 않는 것 | cookie flags, login success, browser engine별 behavior나 production domain은 static assertion이 증명하지 않습니다. |
| 후속 연결 | 이 Thread의 최종 상태에서 static contract와 process/browser runtime job이 서로 다른 증거 층으로 함께 남습니다. |

비교 기준:
- 직전 관련 SHA: `65512bc24161` — `fix(ci): 브라우저 E2E API origin 정렬`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `a9f8b8609711` | root script와 Make target을 verification layer별 entrypoint로 정리합니다. | 도입 | root `package.json`과 `Makefile`이 verification layer별 stable entrypoint를 정의합니다. |
| `68404e51ea53` | push/PR에서 typecheck, unit, build를 실행하는 첫 repository CI 경계를 추가합니다. | 도입·불충분 | `.github/workflows/ci.yml` verify job이 compile-level gate를 만들지만 external service와 process delivery는 포함하지 않습니다. |
| `0e360d333540` | PostgreSQL integration test를 독립 CI job에서 실행합니다. | 확장 | 독립 `postgres-integration` job이 DB-specific regression을 compile/unit gate와 분리해 실행합니다. |
| `3367b4266049` | production artifact와 PostgreSQL을 실제로 기동해 HTTP/WS smoke와 browser E2E까지 검증합니다. | 확장·실행 검증 | `process-and-browser` job이 build artifact, PostgreSQL migration, process readiness, HTTP/WS/browser client를 실제 실행 순서로 연결합니다. |
| `7cb0d32b5be3` | workflow의 toolchain, frozen install, unit/integration/process/browser command 존재를 contract로 고정합니다. | 정적 회귀 검증 | `tests/ci-contract.test.mjs`가 workflow delivery gate의 selected literal contract를 repository test로 만듭니다. |
| `bf0bc1199c84` | API/Web을 demo mode로 build/start하고 PostgreSQL 없이 guest-only Playwright suite를 실행합니다. | 확장·실행 검증 | 별도 guest job이 DB-less demo capability를 production-style artifact/process와 browser journey로 검증합니다. |
| `0ae1ded2c56f` | 별도 guest job, mode 값, Playwright command와 guest spec 존재를 workflow contract로 검증합니다. | 회귀 검증 | guest mode/job/spec의 selected source contract가 `tests/ci-contract.test.mjs`에 추가됩니다. |
| `f35728f4ef92` | CI, production Docker, load harness contract suite를 하나의 root command와 Make target으로 연결합니다. | 통합 | root package/Make entrypoint가 여러 static delivery contract의 invocation ownership을 하나로 묶습니다. |
| `4f9e66c35586` | repository static contract suite를 CI에서 실행합니다. | CI 통합 | verify job이 root static contract suite를 mandatory step으로 소비합니다. |
| `65512bc24161` | browser E2E API origin을 cookie semantics와 맞는 `localhost`로 정렬합니다. | 수정 | workflow API base의 host를 cookie를 발급받은 `localhost`와 동일하게 만들어 browser session transport를 복구합니다. |
| `527921bc9d69` | CI contract가 API base URL의 exact cookie origin을 고정하도록 합니다. | 회귀 검증 | exact localhost assertion이 hostname/cookie regression을 static CI contract로 보호합니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| typecheck/unit/build만 통과하면 delivery 가능하다고 간주 | `0e360d333540`, `3367b4266049` | `7cb0d32b5be3` + 실제 jobs | DB integration을 분리하고 compiled process·migration·readiness·smoke·browser를 실행합니다. static test는 job 존재를, job은 runtime을 증명합니다. |
| workflow configuration이 조용히 약화될 수 있음 | `7cb0d32b5be3`, `0ae1ded2c56f`, `f35728f4ef92` | `4f9e66c35586` | selected job/version/command를 source test로 고정하고 root command를 CI에서 실행합니다. |
| `localhost` cookie를 `127.0.0.1` API에 보낼 수 있다고 가정 | `65512bc24161` | `527921bc9d69` + process/browser job | host-only cookie semantics에 맞춰 origin을 통일하고 exact literal 회귀를 차단합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| CI toolchain/dependencies | workflow setup + frozen install | all verification jobs | hosted runner job lifetime | setup/install failure 시 해당 job 차단 |
| PostgreSQL/migration | Actions service 또는 integration harness + DB CLI | integration/API process | job lifetime | health/migration non-zero면 downstream start/test 차단 |
| API/Web/browser processes | production build와 job shell | smoke scripts/Playwright | background PID와 browser context lifetime | trap/runner teardown이 process 정리 |
| workflow contract evidence | static Node tests | verify job/merge gate | test process와 commit history | literal drift 시 non-zero; runtime success는 별도 job |

## 9. Thread 최종 상태

- 최종 delivery owner: workflow가 toolchain·service·process·test ordering을, root scripts가 stable command를, shell trap이 process cleanup을, static tests가 selected workflow contract를 소유합니다.
- source와 production artifact의 관계: jobs는 root build 뒤 compiled API와 production/standalone Web을 실행합니다. dev source server가 delivery evidence가 되지 않습니다.
- build-time과 runtime configuration의 관계: Web public API/WS/mode는 build 전에 주입되고 API mode/DB/session 값은 process runtime에 주입됩니다. 일반 job은 compiled artifact라도 `APP_MODE=development`입니다.
- startup/readiness/shutdown contract: PostgreSQL health와 migration 뒤 API/Web process를 시작하고 polling 성공 후 smoke/browser를 실행하며 trap이 PID를 정리합니다.
- fail-closed 조건: install/build/migration/readiness/smoke/E2E/static assertion 중 하나라도 실패하면 job이 non-zero입니다. exact localhost origin도 static gate입니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: repository exact SHA에서 workflow와 test implementation을 검사했습니다. 이 세션에서 GitHub Actions·pnpm·PostgreSQL·Playwright를 재실행하지 않았으므로 새 runtime result는 주장하지 않습니다.

## 10. 최종 execution/delivery flow

```text
push/PR
→ pinned Node/pnpm + frozen install
→ typecheck/unit/static contracts/build + artifact verification
├─ PostgreSQL integration job
├─ process/browser job:
│  PostgreSQL health → package migration → compiled API + production Web
│  → readiness → HTTP smoke → WS smoke → authenticated Playwright
└─ guest browser job:
   demo build → compiled API + production Web → readiness → guest-only Playwright
all background processes → shell trap/hosted-runner teardown
workflow source → `tests/ci-contract.test.mjs` exact contract assertions
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `08-verification-and-test-architecture`: smoke/E2E 자체의 test design과 증명 범위
- `02-production-build-and-package-artifacts.md`: CI가 먼저 생성하고 검사하는 artifact
- `03-container-images-and-production-runtime-lifecycle.md`: Docker/Compose delivery contract와의 연결

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 04-ci-production-process-and-browser-delivery-verification.md =====

===== BEGIN FILE: 05-runtime-version-and-dependency-security-contracts.md =====
# Runtime version과 dependency security contract

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

Node/pnpm toolchain을 local metadata, package engine, CI, Docker image와 contract test 사이에서 하나의 release contract로 고정하고, Node·Next.js·WebSocket·production dependency security patch가 manifest와 lock/override graph에 일관되게 반영되는 과정을 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- local version file, package manager, package engine, CI, Docker image가 같은 exact runtime version contract를 사용합니다.
- runtime version 기대값은 여러 contract test에 literal로 복제하지 않고 canonical `.node-version`에서 읽습니다.
- CI install은 exact pnpm과 frozen lockfile을 사용해 manifest와 resolved graph의 drift를 차단합니다.
- direct dependency security patch는 해당 workspace manifest와 `pnpm-lock.yaml` resolution을 함께 갱신합니다.
- 취약한 transitive dependency는 root `pnpm.overrides`와 lockfile graph에서 최소 patched version으로 강제할 수 있습니다.
- version 변경 commit은 dependency/runtime graph를 바꾸지만 vulnerability scan 통과나 application runtime 성공 자체를 증명하지 않습니다.

## 2. 핵심 질문

- Node/pnpm 범위를 처음 고정한 파일과 이후 exact engine pin이 추가로 필요했던 이유는 무엇입니까?
- `.node-version`을 canonical source로 읽는 contract test가 어떤 duplicated literal drift를 제거합니까?
- Node patch가 local file, engine, CI, API/Web/load image에 어떤 순서로 반영됩니까?
- Next.js와 `@next/swc` platform package, API `ws`, Fastify/PostCSS direct requirement가 lockfile에서 어떻게 함께 이동합니까?
- root `pnpm.overrides`는 `fast-uri`, `nanoid`, `postcss`, `sharp`의 transitive resolution을 어떻게 강제합니까?
- manifest/lock consistency와 실제 취약점 부재·runtime compatibility 사이에는 어떤 증거 차이가 있습니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ee4bebc84f95` | `build(runtime): 지원 Node.js·pnpm 범위 고정` | B | OPERATIONS | local version manager, package metadata, container runtime을 Node/pnpm 기준에 맞춥니다. |
| 2 | `9693b2a9ad3d` | `build(runtime): Node.js engine version을 정확히 고정` | B | PERSISTENCE, OPERATIONS | package engine을 repository가 실제 사용하는 exact Node version으로 좁힙니다. |
| 3 | `48c2188eb42a` | `test(runtime): Node 버전 계약을 기준 파일에서 읽음` | B | OPERATIONS, TEST | CI/Docker contract test가 중복 literal 대신 canonical version file을 사용하도록 합니다. |
| 4 | `3a8cd06a1098` | `build(runtime): Node.js 보안 패치 적용` | C | - | 고정된 Node patch를 version file, engine, CI, API/Web/load image에 함께 적용합니다. |
| 5 | `69e22da94cb4` | `build(web): Next.js 보안 패치 적용` | C | - | Next.js direct requirement와 resolved compiler package를 patched version으로 갱신합니다. |
| 6 | `0066e48ea3c9` | `build(api): WebSocket 보안 패치 적용` | C | - | API `ws` dependency와 workspace resolution을 patched version으로 갱신합니다. |
| 7 | `4c4f7df2242a` | `build(security): 프로덕션 의존성 취약점 패치` | B | WEB, OPERATIONS | Fastify/Next.js/PostCSS와 취약한 transitive dependency override를 production graph에 반영합니다. |

## 5. Commit별 학습 기록

### 5.1. `build(runtime): 지원 Node.js·pnpm 범위 고정`

| 항목 | 값 |
| --- | --- |
| SHA | `ee4bebc84f95` |
| Importance | B |
| Tags | OPERATIONS |
| Source에서 확정된 역할 | local version manager, package metadata, container runtime을 Node/pnpm 기준에 맞춥니다. |

#### 해당 SHA에서 확인할 실제 코드

- 새 `.node-version`과 `.nvmrc`가 `24.18.0`을 동일하게 기록하는지 확인합니다.
- root `package.json`의 `engines.node: >=24 <25`와 `packageManager: pnpm@10.32.1`이 local version file보다 넓거나 정확한 범위를 어떻게 표현하는지 확인합니다.
- `.github/workflows/ci.yml`의 각 setup step이 Node `24.18.0`, pnpm `10.32.1`을 사용하는지 확인합니다.
- 이 SHA에서 Dockerfiles가 아직 version contract consumer인지 실제 diff로 구분하고, 존재하지 않는 consumer를 소급해 설명하지 않습니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 개발자 local runtime, package manager metadata와 CI setup이 하나의 repository-level version source로 명시되지 않았습니다. |
| 해결하려던 문제 | Node major/minor 또는 pnpm resolver가 달라지면 install graph, ESM/Next build, native package 결과가 달라질 수 있고 CI와 local reproduction이 분리됩니다. |
| 핵심 결정 | `.node-version`/`.nvmrc`에 Node 24.18.0을 추가하고 package engine을 Node 24 범위로, package manager를 pnpm 10.32.1로 명시하며 CI setup 값을 맞췄습니다. |
| build → package → execute 흐름 | version manager가 canonical file 선택 → Corepack/package metadata가 pnpm version 안내 → CI가 exact Node/pnpm setup → frozen lockfile install → verification/build. |
| ownership/lifetime/cleanup | repository version files가 local selection을, root package metadata가 compatibility/pnpm contract를, workflow가 CI runtime을 소유합니다. |
| failure/rollback/fail-closed | unsupported engine은 package manager 경고/차단 정책에 따라 드러나고 CI는 exact setup 실패 시 job을 차단합니다. `>=24 <25`는 patch drift를 허용합니다. |
| 보장하는 것 | local tooling과 CI가 같은 Node patch/pnpm을 선택할 수 있는 명시적 기준이 생깁니다. |
| 보장하지 않는 것 | package engine은 아직 exact patch가 아니며 모든 Docker/load consumer가 자동으로 이 파일을 읽는 것도 아닙니다. runtime compatibility test도 추가하지 않습니다. |
| 후속 연결 | `9693b2a9ad3d`가 package engine을 exact patch로 좁히고 `48c2188eb42a`가 CI/Docker contract tests의 expected version을 canonical file에서 읽게 합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `9693b2a9ad3d` — `build(runtime): Node.js engine version을 정확히 고정`

### 5.2. `build(runtime): Node.js engine version을 정확히 고정`

| 항목 | 값 |
| --- | --- |
| SHA | `9693b2a9ad3d` |
| Importance | B |
| Tags | PERSISTENCE, OPERATIONS |
| Source에서 확정된 역할 | package engine을 repository가 실제 사용하는 exact Node version으로 좁힙니다. |

#### 해당 SHA에서 확인할 실제 코드

- root `package.json`의 `engines.node`가 `>=24 <25`에서 exact `24.18.0`으로 바뀌는 단일 diff를 확인합니다.
- `.node-version`, `.nvmrc`, CI value는 이미 24.18.0이므로 package consumer validation만 뒤늦게 exact contract에 합류하는지 확인합니다.
- 이 commit은 Dockerfile, Compose, persistence code를 변경하지 않으므로 container lifecycle Thread에 중복 배치하지 않습니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | local/CI는 24.18.0을 사용했지만 package engine은 모든 Node 24 patch를 허용했습니다. |
| 해결하려던 문제 | 설치 환경이 24.x의 다른 patch를 선택해도 engine check를 통과하므로 repository가 검증한 runtime과 실제 consumer runtime이 달라질 수 있었습니다. |
| 핵심 결정 | root `engines.node`를 exact `24.18.0`으로 좁혔습니다. |
| build → package → execute 흐름 | package manager가 root metadata read → Node version exact comparison → mismatch 시 configured enforcement/warning → matching runtime에서 frozen install/build. |
| ownership/lifetime/cleanup | root package metadata가 workspace-wide compatibility gate를 소유합니다. 실제 runtime installation은 developer/CI/image builder가 소유합니다. |
| failure/rollback/fail-closed | 다른 patch는 engine mismatch가 됩니다. 다만 engine-strict 설정이 없으면 일부 package manager는 경고만 할 수 있습니다. |
| 보장하는 것 | repository가 지원 대상으로 선언한 Node version이 local/CI exact patch와 일치합니다. |
| 보장하지 않는 것 | Node binary integrity, automatic installation, Docker base alignment, security 상태를 단독으로 보장하지 않습니다. |
| 후속 연결 | `48c2188eb42a`가 version assertions의 expected value를 `.node-version` 한 곳에서 읽도록 합니다. |

비교 기준:
- 직전 관련 SHA: `ee4bebc84f95` — `build(runtime): 지원 Node.js·pnpm 범위 고정`
- 다음 관련 SHA: `48c2188eb42a` — `test(runtime): Node 버전 계약을 기준 파일에서 읽음`

### 5.3. `test(runtime): Node 버전 계약을 기준 파일에서 읽음`

| 항목 | 값 |
| --- | --- |
| SHA | `48c2188eb42a` |
| Importance | B |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | CI/Docker contract test가 중복 literal 대신 canonical version file을 사용하도록 합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `tests/ci-contract.test.mjs`와 `tests/docker-production.test.mjs`가 hardcoded Node literal을 제거하고 `.node-version`을 read/trim하는지 확인합니다.
- workflow setup, package engine, API/Web Dockerfile base assertions이 같은 `expectedNodeVersion`을 소비하는지 확인합니다.
- canonical file만 변경하고 consumer files가 갱신되지 않은 경우 test가 어떤 mismatch를 보고하는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Node expected value가 CI contract와 Docker contract test에 각각 literal로 복제돼 version patch 때 test 자체도 수동 수정해야 했습니다. |
| 해결하려던 문제 | 여러 expected literal이 서로 달라지면 실제 consumer drift를 놓치거나, 올바른 version patch가 stale test 때문에 잘못 실패할 수 있었습니다. |
| 핵심 결정 | 두 contract test가 repository `.node-version`을 canonical expected value로 읽고 package/CI/Docker consumer text를 비교하도록 변경했습니다. |
| build → package → execute 흐름 | test startup → `.node-version` read/trim → workflow/package/Dockerfile source read → expected version assertion → mismatch list/exception. |
| ownership/lifetime/cleanup | `.node-version`이 expected version source를, contract tests가 consumer alignment evidence를 소유합니다. |
| failure/rollback/fail-closed | canonical file이 없거나 비어 있거나 consumer 값이 다르면 test가 실패합니다. |
| 보장하는 것 | 테스트 실행 시 Node patch expected value의 duplicated literal drift를 제거하고 여러 delivery consumer의 정렬을 확인합니다. |
| 보장하지 않는 것 | 해당 Node binary가 실제 설치·실행되는지, application이 호환되는지, base image digest가 안전한지는 증명하지 않습니다. |
| 후속 연결 | `3a8cd06a1098`의 24.18.1 security patch가 canonical file과 모든 consumer를 함께 갱신하며 이 contract를 통과하도록 구성됩니다. |

비교 기준:
- 직전 관련 SHA: `9693b2a9ad3d` — `build(runtime): Node.js engine version을 정확히 고정`
- 다음 관련 SHA: `3a8cd06a1098` — `build(runtime): Node.js 보안 패치 적용`

### 5.4. `build(runtime): Node.js 보안 패치 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `3a8cd06a1098` |
| Importance | C |
| Tags | - |
| Source에서 확정된 역할 | 고정된 Node patch를 version file, engine, CI, API/Web/load image에 함께 적용합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `.node-version`, `.nvmrc`, root engine, 모든 CI setup, API/Web Docker stage와 load bootstrap image가 `24.18.0`에서 `24.18.1`로 함께 이동하는지 확인합니다.
- `48c2188eb42a`의 canonical version contract가 이 mechanical patch에서 어떤 stale consumer를 감지할 수 있는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | release contract 전체가 Node 24.18.0에 고정돼 있었습니다. |
| 해결하려던 문제 | security patch를 일부 consumer에만 적용하면 local·CI·image·load 환경이 다시 달라집니다. |
| 핵심 결정 | canonical files, engine, workflow, API/Web/load image 값을 24.18.1로 일괄 갱신했습니다. |
| build → package → execute 흐름 | canonical version patch → local/CI/image/load consumer 갱신 → 이후 contract test가 일치 여부 검사. |
| ownership/lifetime/cleanup | 각 delivery file이 자신의 runtime selector를 소유하며 canonical test가 정렬을 관찰합니다. |
| failure/rollback/fail-closed | 한 consumer가 stale하면 contract test가 실패할 수 있습니다. |
| 보장하는 것 | repository에 기록된 Node selectors가 24.18.1로 일치합니다. |
| 보장하지 않는 것 | Node 보안 advisory 해소 여부나 application runtime 성공을 commit 자체가 실행 증거로 제공하지 않습니다. |
| 후속 연결 | 같은 날 `69e22da94cb4`, `0066e48ea3c9`가 framework/WebSocket dependency graph를 patch합니다. |

비교 기준:
- 직전 관련 SHA: `48c2188eb42a` — `test(runtime): Node 버전 계약을 기준 파일에서 읽음`
- 다음 관련 SHA: `69e22da94cb4` — `build(web): Next.js 보안 패치 적용`

### 5.5. `build(web): Next.js 보안 패치 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `69e22da94cb4` |
| Importance | C |
| Tags | - |
| Source에서 확정된 역할 | Next.js direct requirement와 resolved compiler package를 patched version으로 갱신합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/web/package.json`의 Next requirement가 `^15.3.2`에서 `^15.5.21`로 바뀌는지 확인합니다.
- `pnpm-lock.yaml`의 resolved Next 15.5.21과 Darwin/Linux/Windows별 `@next/swc-*` package가 동일 patch line으로 갱신되는지 확인합니다.
- source/API contract 변경이 없고 dependency graph만 바뀌는 C급 release maintenance 범위를 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Web manifest의 direct range와 lockfile은 이전 Next/SWC resolution을 가리켰습니다. |
| 해결하려던 문제 | manifest만 patch하거나 lockfile만 바꾸면 frozen install이 기대한 patched framework/compiler graph를 재현하지 못합니다. |
| 핵심 결정 | Next direct requirement와 lockfile의 Next 및 platform compiler packages를 15.5.21 계열로 갱신했습니다. |
| build → package → execute 흐름 | manifest range 변경 → pnpm resolution refresh → frozen install이 Next와 host platform SWC artifact를 lockfile대로 선택 → existing Next build가 이를 소비합니다. |
| ownership/lifetime/cleanup | Web manifest가 direct intent를, lockfile이 exact cross-platform resolution을 소유합니다. |
| failure/rollback/fail-closed | manifest-lock 불일치면 frozen install이 실패하거나 stale graph가 남습니다. |
| 보장하는 것 | repository dependency graph가 patched Next/SWC versions를 재현합니다. |
| 보장하지 않는 것 | 취약점 scanner 결과, framework compatibility, build/E2E pass는 이 diff만으로 증명하지 않습니다. |
| 후속 연결 | `0066e48ea3c9`가 API WebSocket library를 patch하고 `4c4f7df2242a`가 더 넓은 direct/transitive graph를 갱신합니다. |

비교 기준:
- 직전 관련 SHA: `3a8cd06a1098` — `build(runtime): Node.js 보안 패치 적용`
- 다음 관련 SHA: `0066e48ea3c9` — `build(api): WebSocket 보안 패치 적용`

### 5.6. `build(api): WebSocket 보안 패치 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `0066e48ea3c9` |
| Importance | C |
| Tags | - |
| Source에서 확정된 역할 | API `ws` dependency와 workspace resolution을 patched version으로 갱신합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/package.json`의 `ws` requirement가 patched `^8.21.0`으로 이동하는지 확인합니다.
- `pnpm-lock.yaml`에서 API direct resolution과 `@fastify/websocket`이 공유하는 `ws` resolution이 8.21.0으로 정렬되는지 확인합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | API direct `ws`와 lockfile의 shared/transitive WebSocket runtime이 이전 8.x patch에 고정돼 있었습니다. |
| 해결하려던 문제 | direct requirement와 Fastify WebSocket consumer가 서로 다른 stale resolution을 쓰면 patched runtime graph를 보장할 수 없습니다. |
| 핵심 결정 | API manifest와 lockfile의 relevant `ws` resolution을 8.21.0으로 갱신했습니다. |
| build → package → execute 흐름 | manifest change → lock resolution → frozen install → Fastify/WebSocket runtime이 patched `ws` package 소비. |
| ownership/lifetime/cleanup | API manifest가 direct requirement를, lockfile이 workspace-wide exact resolution을 소유합니다. |
| failure/rollback/fail-closed | lock mismatch면 frozen install이 실패하거나 예상하지 않은 version이 선택될 수 있습니다. |
| 보장하는 것 | repository가 설치할 WebSocket runtime graph가 patched `ws` version으로 정렬됩니다. |
| 보장하지 않는 것 | WebSocket protocol regression, load behavior, advisory scan 통과는 증명하지 않습니다. |
| 후속 연결 | `4c4f7df2242a`가 Fastify/Next/PostCSS와 transitive overrides를 함께 patch합니다. |

비교 기준:
- 직전 관련 SHA: `69e22da94cb4` — `build(web): Next.js 보안 패치 적용`
- 다음 관련 SHA: `4c4f7df2242a` — `build(security): 프로덕션 의존성 취약점 패치`

### 5.7. `build(security): 프로덕션 의존성 취약점 패치`

| 항목 | 값 |
| --- | --- |
| SHA | `4c4f7df2242a` |
| Importance | B |
| Tags | WEB, OPERATIONS |
| Source에서 확정된 역할 | Fastify/Next.js/PostCSS와 취약한 transitive dependency override를 production graph에 반영합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/package.json` Fastify `^5.11.3`, `apps/web/package.json` Next `^15.5.23`와 PostCSS `^8.5.26` direct patch를 확인합니다.
- root `package.json`의 `pnpm.overrides`가 `fast-uri@<3.1.5`, `nanoid@<3.3.17`, `postcss@<8.5.23`, `sharp@<0.35.0`을 patched minimum으로 강제하는지 확인합니다.
- `pnpm-lock.yaml`의 top-level overrides, importer versions, Fastify/Next/PostCSS 및 platform-specific Sharp/Next compiler graph가 함께 refresh되는지 확인합니다.
- commit에 vulnerability scanner report나 executed build/test evidence가 포함되지 않으므로 resolution graph 변경과 실제 보안/호환성 증거를 구분합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 직접 의존성 Fastify/Next/PostCSS와 여러 transitive package가 이전 resolution에 남아 있었습니다. direct manifest만 올려도 nested package가 vulnerable range를 선택할 수 있었습니다. |
| 해결하려던 문제 | production graph는 workspace direct dependency뿐 아니라 validator/asset/build chain의 transitive dependency로 구성됩니다. lockfile refresh와 override가 없으면 frozen install이 vulnerable nested version을 계속 재현할 수 있습니다. |
| 핵심 결정 | API/Web direct requirements를 patch하고 root pnpm overrides로 fast-uri, nanoid, postcss, sharp의 최소 안전 resolution을 강제했습니다. lockfile 전체 importer/package graph를 새 resolution에 맞췄습니다. |
| build → package → execute 흐름 | root/workspace manifest + overrides → pnpm resolution → lockfile에 exact direct/transitive/platform graph 기록 → CI/Docker frozen install → API/Web build와 runtime이 그 graph를 소비합니다. |
| ownership/lifetime/cleanup | workspace manifests가 direct compatibility intent를, root overrides가 cross-workspace transitive policy를, lockfile이 exact resolved graph를 소유합니다. |
| failure/rollback/fail-closed | override와 상위 package의 peer/engine 요구가 충돌하면 install/build/runtime에서 실패할 수 있습니다. frozen install은 manifest-lock mismatch를 fail-closed합니다. |
| 보장하는 것 | repository가 재현하는 production dependency graph에서 named vulnerable ranges가 patched versions로 강제되고 direct requirements와 lockfile이 일치합니다. |
| 보장하지 않는 것 | 모든 알려진/미공개 취약점 부재, scanner pass, API/Web behavior compatibility, image SBOM/signature는 증명하지 않습니다. 실제 install/build/test를 이 세션에서 실행하지 않았습니다. |
| 후속 연결 | 최종 release flow에서는 이 graph가 Thread 02 build, Thread 03 image, Thread 04 CI process 검증의 입력이 됩니다. |

비교 기준:
- 직전 관련 SHA: `0066e48ea3c9` — `build(api): WebSocket 보안 패치 적용`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `ee4bebc84f95` | local version manager, package metadata, container runtime을 Node/pnpm 기준에 맞춥니다. | 도입·불충분 | `.node-version`, `.nvmrc`, root package와 CI가 기준을 공유하지만 engine range는 patch drift를 허용합니다. |
| `9693b2a9ad3d` | package engine을 repository가 실제 사용하는 exact Node version으로 좁힙니다. | 수정 | root package engine이 exact 24.18.0으로 좁아져 declared compatibility와 verified runtime을 정렬합니다. |
| `48c2188eb42a` | CI/Docker contract test가 중복 literal 대신 canonical version file을 사용하도록 합니다. | 회귀 검증 | 두 static contract test가 `.node-version`을 단일 expected source로 사용해 version consumer drift를 검사합니다. |
| `3a8cd06a1098` | 고정된 Node patch를 version file, engine, CI, API/Web/load image에 함께 적용합니다. | 갱신 | canonical file과 local/CI/API/Web/load runtime selectors가 24.18.1로 함께 이동합니다. |
| `69e22da94cb4` | Next.js direct requirement와 resolved compiler package를 patched version으로 갱신합니다. | 갱신 | Web manifest와 lockfile의 Next 및 platform SWC resolution이 patched version으로 정렬됩니다. |
| `0066e48ea3c9` | API `ws` dependency와 workspace resolution을 patched version으로 갱신합니다. | 갱신 | API direct 및 Fastify WebSocket dependency graph의 `ws` resolution이 patched version으로 이동합니다. |
| `4c4f7df2242a` | Fastify/Next.js/PostCSS와 취약한 transitive dependency override를 production graph에 반영합니다. | 확장·갱신 | direct manifest, root overrides와 lockfile의 exact graph가 production dependency security policy를 함께 표현합니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| local/CI exact patch와 package engine의 넓은 Node 24 range가 공존 | `9693b2a9ad3d` | `48c2188eb42a` | engine을 exact patch로 좁히고 contract tests가 canonical `.node-version`과 consumer를 비교합니다. |
| Node expected version이 여러 test literal에 복제됨 | `48c2188eb42a` | `3a8cd06a1098`의 일괄 patch | canonical file 하나를 읽어 stale CI/Docker consumer를 감지합니다. |
| direct patch만으로 transitive vulnerable resolution을 제거할 수 있다고 가정 | `4c4f7df2242a` | frozen lockfile install + existing CI build/process gates | root overrides와 lockfile graph를 함께 갱신하지만 vulnerability scan 자체는 별도 evidence가 필요합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| Node/pnpm version contract | `.node-version`, `.nvmrc`, root package | developer, CI, Docker/load builders | checkout/build lifetime | consumer mismatch는 static contract 또는 setup/install failure |
| direct dependency intent | workspace `package.json` | pnpm resolver/build/runtime | release manifest lifetime | manifest-lock mismatch는 frozen install 차단 |
| transitive resolution policy | root `pnpm.overrides` | all workspace dependency graphs | lockfile/release lifetime | incompatible override는 install/build/runtime failure 가능 |
| release contract evidence | CI/Docker static tests와 lockfile diff | CI gate/reviewer | test/job/commit lifetime | runtime/security scan 결과는 별도 evidence |

## 9. Thread 최종 상태

- 최종 delivery owner: canonical version files와 root package가 toolchain policy를, workspace manifests/root overrides/lockfile이 dependency graph를, static tests와 frozen CI install이 consistency gate를 소유합니다.
- source와 production artifact의 관계: source dependency declarations과 lockfile resolution이 Thread 02 build와 Thread 03 image에 들어가는 정확한 runtime graph를 결정합니다.
- build-time과 runtime configuration의 관계: Node/pnpm과 framework/build dependencies는 install/build-time release 입력이며 결과 image/process는 그 resolution을 포함합니다. runtime env는 이 Thread 범위가 아닙니다.
- startup/readiness/shutdown contract: version/security contract는 startup sequencing을 직접 정의하지 않고, downstream build/process가 동일 graph를 사용하게 합니다.
- fail-closed 조건: engine mismatch 정책, frozen install, static version assertions와 manifest-lock consistency가 drift를 차단합니다. 취약점 scanner fail-closed는 구성되지 않았습니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: exact SHA manifest/version/workflow/Dockerfile/lockfile diff를 검사했습니다. package install, build, audit/scanner, runtime compatibility를 재실행하지 않았습니다.

## 10. 최종 execution/delivery flow

```text
`.node-version` / `.nvmrc`
→ exact root `engines.node` + `packageManager`
→ CI setup / API-Web-load Docker selectors
→ contract tests compare all consumers with canonical version
workspace direct manifests + root `pnpm.overrides`
→ pnpm resolver → exact `pnpm-lock.yaml`
→ frozen CI/Docker install
→ shared/DB/API/Web build → image/process verification
(security patch graph is verified structurally; vulnerability scan result is not present)
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `03-container-images-and-production-runtime-lifecycle.md`: pinned runtime과 resolved dependencies가 실제 image에 포함되는 위치
- `04-ci-production-process-and-browser-delivery-verification.md`: exact toolchain/frozen graph가 CI에서 소비되는 위치
- `02-production-build-and-package-artifacts.md`: patched graph로 production artifacts를 생성하는 위치

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 05-runtime-version-and-dependency-security-contracts.md =====

===== BEGIN FILE: 06-production-configuration-and-durable-storage-fail-closed.md =====
# Production configuration과 durable storage fail-closed

- 카테고리: `09-production-delivery-and-release-engineering` — 제품 전달과 릴리스 엔지니어링
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 1. Thread 목표

production configuration fixture를 먼저 durable PostgreSQL 전제에 맞춘 뒤, 명시적 `APP_MODE=production`과 `NODE_ENV=production` 모두에서 `DATABASE_URL`이 없으면 API startup 전에 거부하고, demo mode의 의도된 memory repository만 허용하는 release boundary를 복원합니다.

이 문서는 완성된 해설이 아니라 exact SHA를 순서대로 확인해 제품 전달 구조의 발전을 복원하기 위한 scaffold입니다.

### 직접 연결되는 불변식

- 명시적인 `APP_MODE=production`과 `NODE_ENV=production`에서 추론된 production은 모두 durable database URL을 요구합니다.
- production은 `DATABASE_URL` 부재를 memory repository 선택으로 조용히 대체하지 않습니다.
- DB requirement는 application/repository construction 이전 configuration parsing 단계에서 fail-closed합니다.
- demo mode는 PostgreSQL 없이 memory repository를 사용할 수 있는 별도의 제한된 trust/capability mode입니다.
- configuration unit test는 mode decision과 rejection을 증명하지만 PostgreSQL reachability, migration, transaction durability는 증명하지 않습니다.

## 2. 핵심 질문

- production fixture에 DB URL을 먼저 추가한 commit은 후속 parser 강화와 어떤 준비 관계를 가집니까?
- `APP_MODE`가 명시된 경우와 `NODE_ENV`에서 mode가 추론되는 경우는 어떤 동일 predicate로 DB requirement를 통과합니까?
- configuration parser가 언제 error를 반환하며 repository factory와 server startup은 그 이후에만 실행됩니까?
- production memory fallback이 match/tournament 결과에 어떤 거짓 성공과 durability 손실을 만들 수 있습니까?
- demo mode가 DB 없이 허용되는 것은 production 예외가 아니라 어떤 명시적 capability contract입니까?
- unit regression이 증명하는 configuration boundary와 실제 PostgreSQL delivery 검증 사이의 차이는 무엇입니까?

## 3. 완료 기준

- Commit map의 모든 SHA가 `web/ft_transcendence` ancestry에 속하는지 확인합니다.
- 각 SHA의 parent 또는 직전 관련 SHA와 비교해 development 실행과 production delivery 실행을 구분합니다.
- build artifact, package export, image layer, Compose service, workflow job, runtime config의 실제 owner를 파일과 command로 기록합니다.
- Fix는 이전 delivery 가정과 root cause를, test/CI는 실제 검증 대상과 증명/비증명 범위를 연결합니다.
- 실행하지 않은 build, Docker, Compose, CI 결과를 실행 증거처럼 기록하지 않습니다.
- 마지막 SHA까지만 사용해 Thread 최종 artifact/lifecycle/verification flow를 작성합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `97d7ca714293` | `test(config): production fixture에 영속 DB 명시` | C | - | production configuration fixture에 PostgreSQL URL을 명시합니다. |
| 2 | `eb675ef74af3` | `fix(config): production에서 영속 저장소 요구` | A | TOURNAMENT, OPERATIONS, RISK | `DATABASE_URL`이 없는 production configuration을 parsing 단계에서 거부합니다. |
| 3 | `4633dfde208d` | `test(config): production memory fallback 거부 검증` | A | OPERATIONS, TEST | 명시/추론 production의 DB 누락 거부와 demo memory 허용을 검증합니다. |

## 5. Commit별 학습 기록

### 5.1. `test(config): production fixture에 영속 DB 명시`

| 항목 | 값 |
| --- | --- |
| SHA | `97d7ca714293` |
| Importance | C |
| Tags | - |
| Source에서 확정된 역할 | production configuration fixture에 PostgreSQL URL을 명시합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.test.ts`의 shared `explicitProductionEnv` fixture에 `DATABASE_URL`이 추가되는 exact diff를 확인합니다.
- 이 commit은 parser behavior를 아직 변경하지 않고 기존 production tests의 valid input contract만 선행 정렬한다는 점을 확인합니다.
- 후속 `eb675ef74af3`에서 DB requirement를 추가해도 unrelated production test가 fixture 누락으로 깨지지 않게 하는 준비 commit으로 해석합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production env test fixture는 mode와 secret 등은 제공했지만 durable DB URL을 명시하지 않았습니다. 당시 parser가 이를 요구하지 않아 valid production fixture로 사용됐습니다. |
| 해결하려던 문제 | 곧 production DB requirement를 fail-closed하면 기존 valid fixture가 모두 의도와 무관하게 실패해 핵심 regression case와 fixture maintenance failure가 섞입니다. |
| 핵심 결정 | 공통 explicit production fixture에 PostgreSQL `DATABASE_URL`을 먼저 추가했습니다. |
| build → package → execute 흐름 | test setup → valid production fixture 생성 → 후속 parser/tests가 durable DB가 있는 정상 production baseline으로 재사용합니다. |
| ownership/lifetime/cleanup | test fixture가 valid input baseline을 소유하고 production parser는 다음 commit까지 unchanged입니다. |
| failure/rollback/fail-closed | 이 SHA 단독으로 DB 없는 production을 거부하지 않습니다. |
| 보장하는 것 | production test baseline이 durable DB를 명시하는 새 의도에 준비됩니다. |
| 보장하지 않는 것 | runtime behavior, memory fallback 차단, PostgreSQL 연결을 증명하지 않습니다. |
| 후속 연결 | `eb675ef74af3`이 parser에 production DB requirement를 추가하고 이 fixture를 정상 path로 소비합니다. |

비교 기준:
- 이 commit의 parent와 비교합니다.
- 다음 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

### 5.2. `fix(config): production에서 영속 저장소 요구`

| 항목 | 값 |
| --- | --- |
| SHA | `eb675ef74af3` |
| Importance | A |
| Tags | TOURNAMENT, OPERATIONS, RISK |
| Source에서 확정된 역할 | `DATABASE_URL`이 없는 production configuration을 parsing 단계에서 거부합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.ts`에서 raw env schema parse 뒤 application mode를 명시값 또는 `NODE_ENV`로 결정하는 흐름을 확인합니다.
- resolved mode가 `production`이고 `DATABASE_URL`이 없을 때 issue/error를 추가하거나 throw해 parsed config 반환을 차단하는 exact branch를 확인합니다.
- `demo` mode는 DB URL 없이 허용되고 이후 repository selection이 memory backend를 선택할 수 있는 경계를 확인합니다.
- configuration parsing이 `apps/api/src/index.ts`의 application/repository/server construction보다 앞서 호출되어 실패 시 port bind·memory state 생성·background runtime 시작이 없는지 caller를 추적합니다.
- 이전 assumption → production에서 silent memory fallback → process는 healthy지만 durable match/tournament state는 사라지는 risk → parser fail-closed invariant를 연결합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | DB URL이 없을 때 repository composition은 memory implementation을 선택할 수 있었고 production mode도 configuration parsing을 통과했습니다. process가 시작·health 응답하면서 실제로는 durable PostgreSQL을 쓰지 않는 거짓 성공 상태가 가능했습니다. |
| 해결하려던 문제 | production match/tournament/profile 결과가 process memory에만 저장되면 restart·replica 이동에서 사라지고, operator는 healthy service를 durable deployment로 오인할 수 있습니다. 이는 단순 feature degradation이 아니라 data-integrity/release contract 위반입니다. |
| 핵심 결정 | resolved mode가 production이면 `DATABASE_URL`을 필수로 검사하고 부재 시 configuration parsing 자체를 실패시켰습니다. explicit `APP_MODE=production`과 `NODE_ENV=production` inference가 같은 requirement를 거칩니다. demo mode는 의도된 memory path로 남겼습니다. |
| build → package → execute 흐름 | raw process env → schema/basic parse → application mode resolve (`APP_MODE` 우선 또는 `NODE_ENV` inference) → production predicate → DB URL present 확인 → 없으면 error/throw 후 startup 중단; 있으면 parsed config 반환 → repository/application/server construction. |
| ownership/lifetime/cleanup | environment provider가 raw value를, parser가 mode/DB consistency invariant를, repository factory가 parser 성공 이후 backend lifetime을 소유합니다. fail-closed하면 backend·server resource는 아직 획득되지 않습니다. |
| failure/rollback/fail-closed | production + missing DB는 startup 전 deterministic configuration error입니다. 잘못된 URL·접속 거부·migration 누락은 URL 존재 검사를 통과한 뒤 DB connection/readiness 단계에서 별도로 실패해야 합니다. demo는 missing DB가 failure가 아닙니다. |
| 보장하는 것 | production process가 durable backend 주소 없이 memory repository로 조용히 기동하지 않습니다. 명시/추론 production 모두 동일하게 차단됩니다. |
| 보장하지 않는 것 | URL이 실제 PostgreSQL인지, credential이 유효한지, schema가 최신인지, writes가 transactionally durable한지는 보장하지 않습니다. production mode 자체의 모든 env/secret 요구도 이 한 branch가 포괄하지 않습니다. |
| 후속 연결 | `4633dfde208d`가 explicit production, inferred production, demo 세 boundary를 deterministic unit regression으로 고정합니다. |

비교 기준:
- 직전 관련 SHA: `97d7ca714293` — `test(config): production fixture에 영속 DB 명시`
- 다음 관련 SHA: `4633dfde208d` — `test(config): production memory fallback 거부 검증`

### 5.3. `test(config): production memory fallback 거부 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4633dfde208d` |
| Importance | A |
| Tags | OPERATIONS, TEST |
| Source에서 확정된 역할 | 명시/추론 production의 DB 누락 거부와 demo memory 허용을 검증합니다. |

#### 해당 SHA에서 확인할 실제 코드

- `apps/api/src/env.test.ts`에서 `APP_MODE=production`이 명시됐지만 DB URL이 없는 case의 expected error를 확인합니다.
- `APP_MODE`가 없고 `NODE_ENV=production`으로 mode가 추론될 때도 동일하게 DB 누락을 거부하는 case를 확인합니다.
- `APP_MODE=demo` 또는 demo fixture에서 DB URL 없이 parse가 성공하고 memory repository 선택에 필요한 mode가 유지되는 case를 확인합니다.
- test가 parser function을 직접 호출해 외부 DB 없이 결정적으로 boundary를 재현하지만 actual PostgreSQL connection/migration/durability는 실행하지 않는다고 기록합니다.

#### 학습자 기록

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | parser fix는 존재했지만 explicit/inferred mode의 한쪽 branch가 후속 refactor에서 빠지거나 demo까지 과도하게 차단해도 별도 regression matrix가 없었습니다. |
| 해결하려던 문제 | mode resolution과 persistence requirement는 두 입력 경로와 하나의 의도된 예외를 포함합니다. 한 happy-path test만으로는 silent production fallback과 demo regression을 동시에 막을 수 없습니다. |
| 핵심 결정 | 세 deterministic unit case를 추가했습니다. explicit production missing DB 거부, NODE_ENV-inferred production missing DB 거부, demo missing DB 허용을 각각 assertion합니다. |
| build → package → execute 흐름 | 각 isolated env object → parser 호출 → production 두 case는 error/throw expectation → demo case는 parsed config/mode expectation. network·filesystem DB resource는 획득하지 않습니다. |
| ownership/lifetime/cleanup | unit test가 input env와 expected outcome을 소유하고 parser가 decision을 수행합니다. test 종료 시 external cleanup은 없습니다. |
| failure/rollback/fail-closed | production case가 성공하거나 demo case가 실패하면 test가 non-zero입니다. 이 test는 malformed URL, connection refusal, migration failure를 재현하지 않습니다. |
| 보장하는 것 | 테스트 실행 시 explicit/inferred production이 DB 없이 통과하지 않고 demo는 memory-capable mode로 남는 configuration matrix가 보호됩니다. |
| 보장하지 않는 것 | PostgreSQL reachability, schema version, write durability, container/CI production startup은 증명하지 않습니다. 실제 DB integration/process evidence는 다른 Threads의 책임입니다. |
| 후속 연결 | Thread 03 production Compose의 required DB/one-shot migration과 Thread 04 PostgreSQL process job이 URL 존재 이후의 delivery path를 보완합니다. |

비교 기준:
- 직전 관련 SHA: `eb675ef74af3` — `fix(config): production에서 영속 저장소 요구`

## 6. Invariant evolution ledger

| 시점 | 불변식 | 상태 | 실제 근거 |
| --- | --- | --- | --- |
| `97d7ca714293` | production configuration fixture에 PostgreSQL URL을 명시합니다. | 준비 | `apps/api/src/env.test.ts`의 valid production fixture가 후속 fail-closed parser contract에 필요한 DB URL을 선행 포함합니다. |
| `eb675ef74af3` | `DATABASE_URL`이 없는 production configuration을 parsing 단계에서 거부합니다. | 수정 | `apps/api/src/env.ts`의 mode-resolved validation이 production missing DB를 application construction 전에 거부합니다. |
| `4633dfde208d` | 명시/추론 production의 DB 누락 거부와 demo memory 허용을 검증합니다. | 회귀 검증 | `apps/api/src/env.test.ts`의 세-case matrix가 production fail-closed와 demo exception을 결정적으로 보호합니다. |

## 7. Failure → Fix → Test 연결

| 이전 가정 또는 failure | Fix | Regression/contract evidence | 학습자 설명 |
| --- | --- | --- | --- |
| production에서 DB URL 부재를 memory repository로 fallback해도 process가 정상처럼 시작할 수 있음 | `eb675ef74af3` | `4633dfde208d` | mode resolution 직후 fail-closed하고 explicit/inferred production 두 경로를 test합니다. |
| production requirement를 강화하면서 demo까지 DB 필수로 만들 위험 | `eb675ef74af3`의 mode-specific branch | `4633dfde208d` demo case | durability requirement를 production에만 적용하고 demo의 제한된 transient storage contract를 유지합니다. |

## 8. Artifact·process·resource ownership

| 대상 | 생성/빌드 주체 | 소비/실행 주체 | lifetime | 실패 시 정리/차단 |
| --- | --- | --- | --- | --- |
| raw environment | process/Compose/CI operator | `apps/api/src/env.ts` parser | process startup lifetime | schema/mode/DB invariant 실패 시 parsed config 미생성 |
| resolved application mode | env parser | repository/application composition | API process lifetime | production missing DB는 consumer 호출 전 차단 |
| durable repository config | `DATABASE_URL` provider | PostgreSQL repository/pool | process/connection lifetime | URL 부재는 parse fail; 접속/migration failure는 후속 DB layer |
| regression evidence | `apps/api/src/env.test.ts` | unit test/CI | test process lifetime | three-case matrix mismatch 시 test failure; external DB evidence 없음 |

## 9. Thread 최종 상태

- 최종 delivery owner: environment parser가 production mode와 durable DB requirement의 fail-closed boundary를, repository composition이 parser 성공 후 backend lifetime을, unit tests가 mode matrix regression을 소유합니다.
- source와 production artifact의 관계: compiled API artifact도 startup에서 동일 env parser를 실행하므로 source/build 형식과 무관하게 production DB requirement가 적용됩니다.
- build-time과 runtime configuration의 관계: `DATABASE_URL`, `APP_MODE`, `NODE_ENV`는 API runtime configuration이며 Web build-time public mode와 별도입니다.
- startup/readiness/shutdown contract: configuration parse가 가장 앞선 gate입니다. 통과 후에만 DB pool/repository/server startup과 readiness가 진행됩니다. shutdown 동작은 이 Thread 범위가 아닙니다.
- fail-closed 조건: explicit 또는 inferred production에서 DB URL이 없으면 startup 전에 거부합니다. demo missing DB는 의도된 허용 조건입니다.
- 검증 가능한 것과 외부 배포 환경에 남는 것: exact SHA parser/test diff로 decision matrix를 확인했습니다. unit command, PostgreSQL connection, migration, process startup은 이 세션에서 실행하지 않았습니다.

## 10. 최종 execution/delivery flow

```text
process environment
→ basic schema parse
→ application mode resolve
   ├─ `APP_MODE=production`
   └─ `APP_MODE` absent + `NODE_ENV=production`
→ production? `DATABASE_URL` required
   ├─ missing → configuration error → no repository/server acquisition
   └─ present → parsed config → PostgreSQL repository/startup path
`APP_MODE=demo` + no DB
→ parsed demo config → intentional memory repository path
unit matrix protects explicit production / inferred production / demo
```

위 흐름을 각 단계의 실제 파일, command, artifact, process와 연결해 다시 작성합니다.

## 11. 교차 카테고리 연결

- `03-container-images-and-production-runtime-lifecycle.md`: production Compose의 required DB secret, one-shot migration, API startup gate
- `04-ci-production-process-and-browser-delivery-verification.md`: 실제 PostgreSQL과 compiled process를 기동하는 delivery evidence
- `01-foundations-and-api-boundaries`: runtime mode와 configuration parsing 자체의 API boundary

## 12. 학습 완료 체크

- [x] 모든 Commit map SHA를 exact historical state에서 확인했습니다.
- [x] build와 runtime을 final HEAD에서 과거로 소급하지 않았습니다.
- [x] artifact producer/consumer와 package/image/process owner를 설명할 수 있습니다.
- [x] production config와 secret의 fail-closed 조건을 설명할 수 있습니다.
- [x] CI/test가 실제로 증명하는 delivery 범위와 증명하지 않는 범위를 구분할 수 있습니다.
- [x] fix와 regression evidence를 실제 이전 failure/가정에 연결했습니다.
- [x] 실행하지 않은 Docker/CI 결과를 실행 증거로 기록하지 않았습니다.
===== END FILE: 06-production-configuration-and-durable-storage-fail-closed.md =====

===== BEGIN FILE: README.md =====
# 제품 전달과 릴리스 엔지니어링

이 카테고리는 애플리케이션 기능이 완성된 뒤 그것을 **재현 가능한 production artifact와 실행 가능한 서비스 조합으로 전달하는 과정**을 학습합니다.

기존 카테고리와의 경계는 다음과 같습니다.

- `07-runtime-observability-and-service-health`는 실행 중인 애플리케이션의 readiness, metrics, drain, failure containment를 다룹니다.
- `08-verification-and-test-architecture`는 unit/integration/process/browser/load/fault 검증의 **테스트 설계**를 다룹니다.
- 이 `09` 카테고리는 source를 production artifact로 만들고, image/runtime을 조립하고, CI에서 그 **전달 가능한 결과물**을 검증하는 과정을 다룹니다.

## 권장 학습 순서

1. [런타임 조립과 reverse proxy의 발전](01-runtime-composition-and-reverse-proxy-evolution.md)
2. [Production build와 package artifact](02-production-build-and-package-artifacts.md)
3. [Container image와 production runtime lifecycle](03-container-images-and-production-runtime-lifecycle.md)
4. [CI production process와 browser delivery 검증](04-ci-production-process-and-browser-delivery-verification.md)
5. [Runtime version과 dependency security contract](05-runtime-version-and-dependency-security-contracts.md)
6. [Production configuration과 durable storage fail-closed](06-production-configuration-and-durable-storage-fail-closed.md)

번호는 이 영역의 실제 개발사에서 처음 등장하는 흐름을 우선해 배치했습니다. 초기 Compose/Caddy runtime이 먼저 등장하고, 이후 compiled package artifact, production image/lifecycle, CI delivery verification이 이어집니다. Runtime/toolchain과 dependency graph의 release contract는 별도로 추적하고, production persistence의 fail-closed configuration은 독립된 startup 정책으로 분리합니다.

## 공통 학습 원칙

- 각 SHA의 실제 historical file과 parent diff를 확인합니다.
- `Dockerfile`, Compose, Caddy, workflow, package script는 단순 설정 파일이 아니라 실행 순서와 resource ownership을 결정하는 코드로 취급합니다.
- build가 성공했다는 사실과 production process가 실제로 실행 가능하다는 사실을 구분합니다.
- runtime evidence는 실제로 실행한 command에 한해서만 기록합니다.
- 같은 SHA가 07/08에도 등장하면 이 카테고리에서는 **제품 전달 관점**만 기록합니다.
===== END FILE: README.md =====

