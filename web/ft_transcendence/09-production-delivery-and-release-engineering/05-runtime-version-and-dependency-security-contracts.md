# 개발 Thread 05 — runtime version과 dependency graph를 하나의 release contract로 유지하기

## 개요

Node version은 한 파일에만 존재하지 않는다. local version manager, package engine, CI runner, Docker build/runner stage, auxiliary container가 각자 version을 선택한다. dependency version도 manifest의 semver 요구와 lockfile의 실제 resolution이라는 두 층을 가진다.

이 Thread는 두 종류의 drift를 다룬다.

1. **toolchain drift** — 개발자는 24.18.0을 쓰지만 CI나 image는 다른 Node를 사용하는 상태
2. **dependency drift** — manifest는 보안 패치 범위를 요구하지만 lockfile과 transitive graph는 이전 resolution을 유지하는 상태

최종 불변 조건은 “최신 version을 자동으로 사용한다”가 아니다. 오히려 반대다.

- repository가 지원하는 Node와 pnpm version을 명시적으로 고정한다.
- CI와 Dockerfile은 canonical Node version과 같아야 한다.
- version 변경은 canonical value와 모든 runtime consumer를 하나의 release change로 이동시킨다.
- direct dependency 요구와 lockfile resolution을 함께 갱신한다.
- direct upgrade만으로 닿지 않는 transitive package는 제한 조건이 있는 override로 최소 version을 강제한다.

commit 제목이 “보안 패치”라고 해도 diff가 직접 증명하는 것은 version graph의 이동이다. 취약점 scanner 통과, 특정 CVE의 제거, source compatibility는 별도 증거가 필요하다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ee4bebc84f95` | `build(runtime): 지원 Node.js·pnpm 범위 고정` | B | `OPERATIONS` | local selector·Compose image·package engine에 Node 24.18.0/pnpm 10.32.1 계약을 도입한다. |
| 2 | `9693b2a9ad3d` | `build(runtime): Node.js engine version을 정확히 고정` | B | `PERSISTENCE, OPERATIONS` | major range였던 package engine을 exact patch version으로 좁힌다. |
| 3 | `48c2188eb42a` | `test(runtime): Node 버전 계약을 기준 파일에서 읽음` | B | `OPERATIONS, TEST` | CI·Dockerfile contract test가 `.node-version`을 canonical source로 읽게 한다. |
| 4 | `3a8cd06a1098` | `build(runtime): Node.js 보안 패치 적용` | C | — | Node 24.18.0→24.18.1을 local·engine·CI·image consumer 전체에 반영한다. |
| 5 | `69e22da94cb4` | `build(web): Next.js 보안 패치 적용` | C | — | Next manifest requirement와 lockfile resolution을 15.5.21로 이동한다. |
| 6 | `0066e48ea3c9` | `build(api): WebSocket 보안 패치 적용` | C | — | API의 `ws` requirement와 resolution을 8.21.0으로 이동한다. |
| 7 | `4c4f7df2242a` | `build(security): 프로덕션 의존성 취약점 패치` | B | `WEB, OPERATIONS` | Fastify·Next·PostCSS direct update와 transitive minimum override를 하나의 graph change로 적용한다. |

## `ee4bebc84f95` — 여러 runtime selector에 같은 policy를 처음 배치하다

두 local selector file이 exact Node version을 갖는다.

```text
.node-version  → 24.18.0
.nvmrc         → 24.18.0
```

root package에는 기존의 exact `packageManager` pin과 함께 Node·pnpm engine policy가 추가된다.

```json
{
  "packageManager": "pnpm@10.32.1",
  "engines": {
    "node": ">=24 <25",
    "pnpm": "10.32.1"
  }
}
```

당시 source-mounted Compose의 API/Web image도 floating major `node:23-bookworm-slim`에서 exact patch image로 이동한다.

```diff
-image: node:23-bookworm-slim
+image: node:24.18.0-bookworm-slim
```

container command가 이미 사용하던 Corepack activation도 exact pnpm 10.32.1이므로, 새 engine policy와 같은 값을 가리킨다.

```sh
corepack prepare pnpm@10.32.1 --activate
pnpm install --frozen-lockfile
```

### 이 시점의 version contract는 완전히 동일하지 않다

| consumer | Node policy |
| --- | --- |
| `.node-version` | exact `24.18.0` |
| `.nvmrc` | exact `24.18.0` |
| Compose API/Web image | exact `24.18.0` |
| `engines.node` | `>=24 <25` — major 범위 |
| `packageManager`/`engines.pnpm` | exact `10.32.1` |

따라서 local/image는 24.18.0을 선택하지만 package manager engine check는 24.x의 다른 patch도 허용한다. commit 제목의 “범위 고정”과 맞는 initial policy지만, exact reproducibility를 원하면 engine도 같은 patch로 좁혀야 한다.

같은 diff에 root `dev` script도 추가되지만, runtime version contract와 무관하므로 이 Thread에서는 제외한다. 또한 이 commit은 CI workflow를 바꾸지 않는다. CI의 version pin은 후속 CI commit에서 생기며, canonical test는 더 뒤에서 연결된다.

## `9693b2a9ad3d` — supported range를 exact runtime identity로 바꾸다

변경은 한 줄이다.

```diff
 "engines": {
-  "node": ">=24 <25",
+  "node": "24.18.0",
   "pnpm": "10.32.1"
 }
```

작은 diff지만 policy 의미는 다르다.

- 이전: Node 24 major의 어느 minor/patch도 repository contract를 만족
- 이후: package install consumer도 24.18.0만 support target으로 본다

이 선택은 patch update를 자동으로 흡수하지 않고 explicit commit으로 수행하게 한다. 보안 patch를 적용하려면 local selector·CI·Dockerfile·engine을 함께 변경하고 검증해야 한다. `3a8cd06a1098`이 실제로 그 update transaction을 보여준다.

`engines`의 enforcement 강도는 package manager 설정에 따라 warning 또는 hard failure가 될 수 있다. 이 field 자체가 모든 shell에서 wrong Node process를 즉시 차단한다고 단정할 수는 없다. repository policy의 machine-readable declaration이라는 것이 확정 범위다.

## `48c2188eb42a` — test literal이 아니라 canonical file과 consumer를 비교하다

초기 CI/Docker contract test에는 `24.18.0` literal이 직접 들어 있었다. version을 올릴 때 production files를 올바르게 바꿔도 test literal을 잊으면 false failure가 나고, 반대로 test와 한 consumer만 함께 바꾸면 다른 consumer drift를 놓칠 수 있다.

이 commit은 `.node-version`을 읽는다.

```js
const nodeVersion = readFileSync(
  resolve(import.meta.dirname, "../.node-version"),
  "utf8"
).trim();
```

CI contract는 workflow의 모든 `node-version:` 값을 수집해 canonical value 하나와 같은지 확인한다.

```js
const nodeVersions = [...workflow.matchAll(/node-version:\s*([^\s]+)/g)]
  .map((match) => match[1]);

assert.ok(nodeVersions.length > 0);
assert.deepEqual([...new Set(nodeVersions)], [nodeVersion]);
```

Docker contract도 각 API/Web Dockerfile source에 canonical value를 사용한 base image line이 **하나 이상 존재하는지** regex로 확인한다.

```js
assert.match(
  source,
  new RegExp(`FROM node:${escapeRegExp(nodeVersion)}-bookworm-slim`)
);
```

### canonicalization이 실제로 보호하는 범위

- CI workflow에 나타나는 모든 `node-version` literal이 동일한지
- API/Web Dockerfile 각각에 `.node-version`과 같은 `node:<version>-bookworm-slim` line이 하나 이상 있는지

### 이 commit이 직접 비교하지 않는 범위

- `.nvmrc`
- `package.json`의 `engines.node`
- auxiliary Compose/load image
- developer machine의 실제 `node --version`
- multi-stage Dockerfile의 모든 `FROM` line이 canonical version인지(현재 assertion은 파일당 한 번의 match만 요구)
- Docker image tag가 동일 digest를 계속 가리키는지

즉 `.node-version`이 repository의 완전한 single source of truth가 된 것이 아니라, **CI와 두 production Dockerfile test의 기준 파일**이 됐다. 나머지 consumer는 update commit에서 함께 바꾸지만 이 test가 자동으로 모두 검사하지는 않는다.

## `3a8cd06a1098` — patch update를 분산된 consumer 전체에 적용하다

Node 24.18.0에서 24.18.1로의 변경은 다음 위치를 함께 이동시킨다.

```text
.node-version
.nvmrc
package.json engines.node
.github/workflows/ci.yml의 모든 Node setup (4곳)
apps/api/Dockerfile의 dependencies + runner stage
apps/web/Dockerfile의 dependencies + runner stage
docker-compose.load.yml의 toxiproxy-bootstrap Node image
```

대표 diff는 다음과 같다.

```diff
-24.18.0
+24.18.1
```

```diff
-FROM node:24.18.0-bookworm-slim AS dependencies
+FROM node:24.18.1-bookworm-slim AS dependencies
...
-FROM node:24.18.0-bookworm-slim AS runner
+FROM node:24.18.1-bookworm-slim AS runner
```

```diff
-node-version: 24.18.0
+node-version: 24.18.1
```

이 commit은 version contract가 release graph라는 사실을 잘 보여준다. build stage만 올리고 runner를 남기면 compile/runtime이 달라지고, CI만 올리면 local/image와 검증 environment가 달라진다. auxiliary load bootstrap을 놓치면 test infrastructure만 old runtime에 남는다.

`48c...`의 test는 `.node-version` 변경 뒤 CI literal drift를 모두 검출하고, API/Web Dockerfile에 새 canonical tag가 전혀 없는 경우를 검출할 수 있다. 그러나 Dockerfile마다 한 번의 regex match만 요구하므로 dependencies stage만 새 version이고 runner stage가 예전 version인 부분 drift는 놓칠 수 있다. `.nvmrc`, engine, load bootstrap도 이 commit이 수동으로 함께 갱신했지만 canonical test의 직접 assertion 대상은 아니다.

commit subject는 security patch intent를 기록한다. exact diff에는 advisory ID, vulnerability scanner output, regression test result가 없으므로 이 문서에서 특정 취약점이 제거됐다고 추가 추론하지 않는다.

## `69e22da94cb4` — manifest 요구와 실제 Next graph를 함께 이동하다

Web manifest의 Next requirement가 바뀐다.

```diff
-"next": "^15.3.2",
+"next": "^15.5.21",
```

lockfile의 importer는 specifier와 resolution을 모두 기록한다.

```diff
 next:
-  specifier: ^15.3.2
-  version: 15.5.15(...)
+  specifier: ^15.5.21
+  version: 15.5.21(...)
```

중요한 점은 manifest가 이전에 `^15.3.2`였어도 lockfile은 이미 15.5.15를 선택하고 있었다는 사실이다. 이 commit은 단순히 15.3→15.5 feature jump를 처음 수행한 것이 아니라, minimum requested line을 15.5.21로 올리고 actual resolution도 15.5.21로 이동시킨다.

lockfile에서는 다음 graph node도 함께 바뀐다.

- `next@15.5.15` → `next@15.5.21`
- `@next/env`
- macOS/Linux/Windows용 optional `@next/swc-*`
- snapshot의 Next dependency references

Next는 platform-specific SWC package를 optional dependency로 갖기 때문에 top-level package 한 줄이 lock graph 여러 node를 바꾸는 것이 정상이다. 이 대규모 lock diff를 “잡음”으로 버리면 어느 platform artifact가 release graph에 포함됐는지 잃는다.

이 commit에는 application source adaptation이 없다. version update 뒤 build·unit·browser CI가 compatibility evidence를 제공할 수 있지만, diff 자체는 실행 결과를 포함하지 않는다.

## `0066e48ea3c9` — WebSocket direct requirement와 resolution을 같은 version으로

API manifest는 `ws` minimum을 올린다.

```diff
-"ws": "^8.18.0",
+"ws": "^8.21.0",
```

lockfile의 actual resolution도 8.20.0에서 8.21.0으로 바뀐다.

```diff
 ws:
-  specifier: ^8.18.0
-  version: 8.20.0
+  specifier: ^8.21.0
+  version: 8.21.0
```

lockfile에는 root API importer뿐 아니라 `@fastify/websocket` snapshot이 참조하는 `ws` node도 8.21.0으로 정렬된다. 같은 package instance를 공유하는 resolution graph가 이동한 것이다.

commit은 protocol handler source를 바꾸지 않는다. WebSocket behavior compatibility는 smoke/CI 실행으로 관찰해야 하며, version diff만으로 handshake·frame handling의 무회귀를 증명하지 않는다.

## `4c4f7df2242a` — direct update와 transitive minimum을 한 graph change로 묶다

마지막 commit은 direct manifest update 세 종류를 포함한다.

```diff
# apps/api/package.json
-"fastify": "^5.2.1",
+"fastify": "^5.11.3",

# apps/web/package.json
-"next": "^15.5.21",
+"next": "^15.5.23",

-"postcss": "^8.5.3",
+"postcss": "^8.5.26",
```

그리고 root `pnpm.overrides`가 추가된다.

```json
{
  "pnpm": {
    "overrides": {
      "fast-uri@<3.1.5": "3.1.5",
      "nanoid@<3.3.17": "3.3.17",
      "postcss@<8.5.23": "8.5.26",
      "sharp@<0.35.0": "0.35.3"
    }
  }
}
```

### direct dependency와 override는 서로 다른 문제를 푼다

| 방식 | owner가 표현하는 것 | 예시 |
| --- | --- | --- |
| direct requirement update | application/package가 직접 요구하는 supported range | Fastify, Next, PostCSS |
| lockfile regeneration | 현재 release에서 실제 선택된 complete graph | Fastify 5.11.3, Next 15.5.23 등 |
| conditional override | 어느 parent가 요구하더라도 특정 취약 minimum 아래 resolution을 허용하지 않음 | fast-uri, nanoid, postcss, sharp |

`fast-uri@<3.1.5`처럼 selector를 제한하면 이미 3.1.5 이상인 future dependency까지 무조건 3.1.5로 낮추지 않는다. vulnerable/undesired lower range에만 floor를 적용하는 형태다.

lockfile에는 overrides block과 함께 Fastify·Next·PostCSS·Sharp 및 platform-specific Sharp/libvips graph가 갱신된다. Native/optional package가 많은 Sharp upgrade는 lock diff가 특히 크다.

### override의 대가

transitive package owner가 선언하지 않은 version을 root가 강제로 선택한다. 따라서 다음 위험이 생긴다.

- parent package가 아직 새 transitive API/behavior를 support하지 않을 수 있다.
- optional native package의 platform resolution이 달라질 수 있다.
- dependency update 뒤 build는 통과해도 runtime corner case가 남을 수 있다.
- override가 오래 남으면 upstream requirement가 개선된 뒤에도 불필요한 policy debt가 된다.

따라서 override는 “안전함을 자동 증명하는 설정”이 아니라 **resolution policy를 root가 인수하는 결정**이다. CI의 typecheck·unit·build·process/browser test가 호환성 evidence를 보강하지만, 이 commit 자체에는 `pnpm audit`, advisory check, SBOM diff, benchmark가 없다.

## 최종 release contract

### Node/pnpm

```text
canonical value: .node-version
  ├─ CI node-version values  ── static contract로 비교
  └─ API/Web Dockerfile에서 canonical tag 존재 ── static contract로 비교

같은 release commit에서 수동 정렬:
  .nvmrc
  package.json engines.node
  auxiliary Node image

pnpm:
  packageManager = 10.32.1
  engines.pnpm   = 10.32.1
  CI setup       = 10.32.1
  Docker Corepack= 10.32.1
```

### dependencies

```text
package manifest semver requirement
          ↓ pnpm install --frozen-lockfile
lockfile importer + complete resolution graph
          ↑
root conditional overrides가 minimum floor 강제
```

| 보장되는 것 | 보장되지 않는 것 |
| --- | --- |
| repository가 의도한 exact Node/pnpm version을 선언 | 모든 developer shell이 이를 hard-enforce |
| CI의 모든 Node literal과 API/Web Dockerfile의 canonical tag 존재를 검사 | Dockerfile 모든 stage·`.nvmrc`·engine·aux image drift 전체 자동 검사 |
| Node patch release에서 주요 consumer를 함께 갱신 | base image tag의 immutable digest |
| direct requirement와 lockfile resolution을 함께 commit | 특정 CVE가 scanner에서 사라짐 |
| lower transitive version을 override로 차단 | override compatibility·장기 필요성 |
| frozen lockfile로 같은 graph를 설치 | registry artifact·native binary supply-chain 검증 |

이 Thread가 정립한 방식은 “version을 자주 올린다”가 아니라 **version change를 application release와 같은 atomic graph update로 취급한다**는 것이다. version literal 하나를 고치는 대신 producer, verifier, runtime consumer, lock graph를 함께 이동시킨다.

## 조사 범위

각 commit의 manifest·lockfile·CI·Dockerfile diff를 exact SHA에서 확인했다. `48c218...`의 full contract source를 확인해 canonical comparison 범위를 CI와 API/Web Dockerfile로 제한했다. commit subject가 보안 목적을 명시한 경우에도 advisory·scanner·test 실행 결과는 source에 없으므로 추가로 주장하지 않았다. package install, build, audit, image build는 이 작업 환경에서 실행하지 않았다.
