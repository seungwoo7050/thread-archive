# 개발 Thread 02 — source workspace를 production artifact graph로 바꾸기

## 개요

이 Thread의 출발점에서 `build`는 대부분 `tsc --noEmit`이었다. type error는 잡을 수 있지만, production process가 실행할 JavaScript·선언 파일·migration SQL·Next.js standalone server는 만들지 않았다. workspace package의 export도 `src/*.ts`를 직접 가리켜, consumer가 TypeScript loader와 repository layout을 함께 알고 있어야 했다.

이 Thread는 다음 artifact graph를 만든다.

```text
@pong-pong/shared dist
        ├──────────────> @pong-pong/db dist + migration assets
        ├──────────────> API dist
        └──────────────> Web standalone bundle

root build: shared → db → api → web
                         │
                         └─ tests/build-artifacts.mjs가 대표 산출물 존재 확인
```

핵심 불변 조건은 세 가지다.

1. production export는 `src/*.ts`가 아니라 emitted JavaScript와 declaration을 가리킨다.
2. Node ESM이 그대로 해석할 수 있도록 emitted relative import가 `.js` 확장자를 가진다.
3. CI의 `pnpm build` 뒤에는 runtime entry와 필수 비코드 asset이 실제 파일로 존재해야 한다.

다만 “파일이 존재한다”와 “그 파일이 실행된다”는 다르다. 이 Thread의 verifier는 존재 여부만 검사한다. compiled process를 실제로 시작하는 증거는 [Thread 04](04-ci-production-process-and-browser-delivery-verification.md), image와 Compose에서 소비하는 경계는 [Thread 03](03-container-images-and-production-runtime-lifecycle.md)에 있다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `37c735de0c37` | `build(shared): production package artifact 구성` | B | `PROTOCOL` | shared package가 JavaScript·declaration을 emit하고 production export를 `dist`로 돌린다. |
| 2 | `430389943b34` | `build(db): production package artifact 구성` | B | `PERSISTENCE` | DB package가 compiled CLI·library·migration asset을 묶을 build contract를 만든다. |
| 3 | `bb67a72882bf` | `build(app): API와 Web production artifact 구성` | A | `PERSISTENCE, WEB, OPERATIONS` | API를 `node dist` 실행으로 전환하고 Web standalone bundle과 workspace build 순서를 확립한다. |
| 4 | `6ab091ffa815` | `test(build): production artifact 생성 검증` | B | `PERSISTENCE, WEB, OPERATIONS` | build 뒤 대표 JavaScript·declaration·SQL·standalone entry가 존재하는지 검사한다. |
| 5 | `09b305b49768` | `ci(build): production artifact 검증 실행` | B | `PERSISTENCE, WEB, OPERATIONS` | artifact verifier를 CI의 build 직후 실행한다. |

## `37c735de0c37` — shared package가 source export를 벗어나다

기존 package는 production과 development 모두 TypeScript source를 직접 export했다.

```json
{
  "type": "module",
  "exports": {
    ".": "./src/index.ts"
  },
  "scripts": {
    "build": "tsc --noEmit"
  }
}
```

이 commit은 package의 public entry를 조건별로 나눈다.

```json
{
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "development": "./src/index.ts",
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json"
  }
}
```

이 구조는 두 consumer contract를 동시에 유지한다.

- development condition을 이해하는 tool은 `src/index.ts`를 직접 사용할 수 있다.
- 일반 ESM import와 production consumer는 `dist/index.js`를 사용한다.
- TypeScript는 `dist/index.d.ts`를 package type entry로 사용한다.

emission용 `tsconfig.build.json`은 `NodeNext` module/resolution, `outDir: dist`, `rootDir: src`, declaration·source map 생성을 명시하고 test source는 제외한다.

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "noEmit": false,
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts"]
}
```

NodeNext emission에서 source의 relative specifier도 실제 runtime 파일명을 표현해야 한다.

```diff
-export * from "./http";
-export * from "./game";
-export * from "./ws";
+export * from "./http.js";
+export * from "./game.js";
+export * from "./ws.js";
```

TypeScript source에 `.js`를 적어도 compiler는 대응하는 `.ts` source를 resolve하고, emitted JavaScript에는 같은 `.js` specifier를 남긴다. 이 변경이 없으면 Node ESM이 `dist/http` 같은 extensionless path를 그대로 해석하지 못할 수 있다.

이 commit이 보장하는 것은 **shared protocol package의 emitted entry와 type entry**다. package tarball 생성, registry publish, tree-shaking 결과, runtime execution은 아직 다루지 않는다.

## `430389943b34` — DB code뿐 아니라 migration asset도 package가 소유하다

DB package도 같은 NodeNext·declaration·conditional export 구조로 이동한다. 차이는 SQL migration이 JavaScript가 아닌 runtime asset이라는 점이다.

```json
{
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "development": "./src/index.ts",
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json && cp -R migrations dist/",
    "migrate:prod": "node dist/cli.js migrate"
  }
}
```

이 commit으로 다음 파일군이 build output이 된다.

```text
packages/db/dist/
  index.js
  index.d.ts
  cli.js
  migrator.js
  rowMappers.js
  schema.js
  migrations/*.sql
```

source import도 emitted ESM을 위해 `.js`로 정렬된다.

```diff
-import { createMemoryRepository, createPostgresRepository } from "./index";
-import { migrateDatabase } from "./migrator";
+import { createMemoryRepository, createPostgresRepository } from "./index.js";
+import { migrateDatabase } from "./migrator.js";
```

### 이 SHA에 남아 있던 migration path 불일치

artifact를 복사하는 것만으로 runtime lookup까지 맞는 것은 아니다. 이 exact SHA의 migrator는 다음 path 하나만 사용했다.

```ts
const migrationsDirectory = fileURLToPath(
  new URL("../migrations", import.meta.url)
);
```

source를 `tsx packages/db/src/migrator.ts`로 실행하면 `../migrations`는 package root의 `packages/db/migrations`를 가리킨다. 그러나 emitted file은 `packages/db/dist/migrator.js`에 놓인다. 그 파일 기준의 `../migrations`도 package root `packages/db/migrations`를 가리키며, build가 새로 복사한 `dist/migrations`를 사용하지 않는다.

따라서 `430389943b34`가 직접 보장하는 것은 다음까지다.

- compiled DB entry·CLI·declaration을 emit한다.
- SQL files를 `dist/migrations`에 포함한다.
- production용 CLI command를 선언한다.

반면 **dist만 복사한 runtime에서 migrator가 그 SQL을 찾는 것**은 아직 이 commit만으로 닫히지 않는다.

이 경로는 persistence category의 `30aac132e14e` — `feat(db): migration set 상태 검사 추가`에서 수정된다. 그 commit은 이 Thread의 commit map에 중복 포함하지 않지만, 이후 production image가 의존하는 중요한 선행 조건이다.

```ts
const migrationDirectoryCandidates = [
  fileURLToPath(new URL("./migrations", import.meta.url)),  // dist/migrations
  fileURLToPath(new URL("../migrations", import.meta.url)) // source 실행 fallback
];
```

즉 최종 구조는 compiled runtime에서는 `./migrations`, development source 실행에서는 `../migrations`를 사용할 수 있다. 이 구분은 final HEAD를 과거에 소급한 것이 아니라, `430389...`의 미완성 지점과 실제로 이를 보완한 별도 commit을 분리해 기록한 것이다.

## `bb67a72882bf` — application artifact와 build order를 하나의 graph로 묶다

이 commit은 네 영역을 동시에 정렬한다.

### 1. API: typecheck-only에서 emitted process로

```diff
-"build": "tsc --noEmit",
+"build": "tsc -p tsconfig.build.json",

-"start": "tsx src/index.ts",
+"start": "node dist/index.js",
```

API에도 shared/DB와 같은 NodeNext emission config가 추가되고 local relative imports가 `.js`로 정리된다. 실행 책임이 `tsx` source loader에서 Node의 emitted ESM loader로 이동한다.

```text
이전: src/index.ts + tsx + 전체 source tree
이후: dist/index.js + Node ESM + runtime dependencies
```

이 차이는 단순 command 이름 변경이 아니다. `start`가 실제 build output의 consumer가 되므로, build가 누락되거나 relative specifier가 emitted layout과 맞지 않으면 process가 시작되지 않는다.

### 2. Web: standalone bundle을 deployment entry로

Next config는 monorepo tracing root와 standalone output을 설정한다.

```js
import { fileURLToPath } from "node:url";

const repositoryRoot = fileURLToPath(new URL("../..", import.meta.url));
const sharedRuntime = fileURLToPath(
  new URL("../../packages/shared/dist/index.js", import.meta.url)
);

const nextConfig = {
  output: "standalone",
  outputFileTracingRoot: repositoryRoot,
  transpilePackages: ["@pong-pong/shared"],
  webpack(config) {
    config.resolve.alias["@pong-pong/shared"] = sharedRuntime;
    return config;
  }
};
```

두 결정이 결합된다.

- `outputFileTracingRoot`가 app directory 밖 workspace dependency까지 trace할 수 있게 한다.
- `@pong-pong/shared`를 compiled `dist/index.js`로 alias해 Web build가 source path alias에 기대지 않는다.

Web `tsconfig.json`에서도 shared/DB source alias가 제거되고, package scripts는 dev·build·typecheck·test 전에 shared build를 실행한다.

```json
{
  "scripts": {
    "predev": "pnpm --filter @pong-pong/shared build",
    "prebuild": "pnpm --filter @pong-pong/shared build",
    "pretypecheck": "pnpm --filter @pong-pong/shared build",
    "pretest": "pnpm --filter @pong-pong/shared build"
  }
}
```

이 방식은 consumer command가 필요한 package artifact를 스스로 준비하게 하지만, 같은 root build 안에서는 shared가 여러 번 build될 가능성이 있다. 최적화보다 correctness와 직접 실행 가능성을 우선한 결정이다.

### 3. root build: workspace 순회를 명시적 dependency order로

```diff
-"build": "pnpm -r build",
+"build": "pnpm --filter @pong-pong/shared build &&
+           pnpm --filter @pong-pong/db build &&
+           pnpm --filter @pong-pong/api build &&
+           pnpm --filter @pong-pong/web build",
```

`pnpm -r`의 일반 순회에 기대지 않고 producer→consumer 순서를 직접 표현한다.

| 단계 | producer | 뒤 단계가 소비하는 것 |
| ---: | --- | --- |
| 1 | shared | DB/API/Web의 protocol runtime·types |
| 2 | DB | API의 repository runtime·migration CLI |
| 3 | API | production process entry |
| 4 | Web | standalone server·static assets |

### 4. verifier command는 먼저 선언되지만 구현은 아직 없다

같은 commit에서 root script가 추가된다.

```json
"verify:build": "node tests/build-artifacts.mjs"
```

그러나 `tests/build-artifacts.mjs` 파일은 다음 commit에서 생긴다. 따라서 `bb67...` 단독 state에서 root `build`는 실행 가능해도 `verify:build`는 아직 완성되지 않았다. history를 읽을 때 “script 선언”과 “검사 구현”을 같은 시점으로 합치면 안 된다.

이 commit이 Thread의 핵심인 이유는 package별 산출물을 **하나의 실행 가능한 dependency graph**로 연결했기 때문이다. 이후 image와 CI는 이 root build contract를 그대로 소비한다.

## `6ab091ffa815` — artifact의 존재를 명시적 postcondition으로 만들다

새 verifier는 build가 끝났다고 가정하고 대표 파일 12개를 검사한다.

```js
const requiredArtifacts = [
  "packages/shared/dist/index.js",
  "packages/shared/dist/index.d.ts",
  "packages/db/dist/index.js",
  "packages/db/dist/index.d.ts",
  "packages/db/dist/migrator.js",
  "packages/db/dist/cli.js",
  "packages/db/dist/migrations/001_initial.sql",
  "packages/db/dist/migrations/004_friendship_tournament_invariants.sql",
  "apps/api/dist/index.js",
  "apps/api/dist/app.js",
  "apps/api/dist/gameHub.js",
  "apps/web/.next/standalone/apps/web/server.js"
];

const missing = requiredArtifacts.filter(
  (artifact) => !existsSync(resolve(artifact))
);

if (missing.length > 0) {
  throw new Error(`Build output is incomplete:\n${missing.join("\n")}`);
}
```

이 목록은 artifact graph의 각 층을 sampling한다.

- shared: JavaScript와 declaration
- DB: public entry, CLI, migrator, migration set의 앞·뒤 SQL
- API: entry와 주요 module
- Web: standalone server entry

### 이 검사가 증명하는 것

- expected path에 파일이 실제로 존재한다.
- declaration emission·SQL copy·standalone output이 완전히 빠지는 회귀를 잡는다.
- 실패 시 missing path 목록을 한 번에 보여준다.

### 이 검사가 증명하지 않는 것

- JavaScript가 Node에서 import·실행되는지
- declaration이 실제 JavaScript API와 일치하는지
- 모든 migration SQL이 포함됐는지
- migration lookup이 올바른 directory를 선택하는지
- standalone server가 static asset과 workspace dependency를 모두 찾는지
- artifact가 이전 build의 stale file이 아닌지

특히 verifier는 clean step을 수행하지 않는다. stale `dist`가 남아 있는 local workspace에서는 source가 emission에 실패해도 오래된 파일이 존재 검사를 통과할 수 있다. clean CI checkout에서는 그 위험이 줄지만, script 자체의 contract는 여전히 existence-only다.

## `09b305b49768` — build와 artifact postcondition을 CI에서 연속 실행하다

CI verify job의 build 뒤에 한 단계가 추가된다.

```yaml
- name: Build
  run: pnpm build

- name: Verify production artifacts
  run: pnpm verify:build
```

순서가 중요하다. fresh checkout·frozen install 환경에서 producer를 실행한 직후 consumer-visible paths를 검사하므로, local stale artifact보다 더 신뢰할 수 있는 evidence가 된다. `pnpm build`가 실패하면 verifier까지 가지 않고, build는 성공했지만 필수 파일이 빠졌다면 verifier가 job을 실패시킨다.

다만 CI는 이 commit에서 artifact를 upload하거나 image로 package하지 않는다. verify job 종료 뒤 workspace는 폐기된다. artifact promotion·registry·checksum·provenance는 이 Thread의 범위 밖이다.

## 최종 artifact contract

```text
pnpm build
  │
  ├─ shared/dist/{index.js,index.d.ts,...}
  ├─ db/dist/{index.js,index.d.ts,cli.js,migrator.js,migrations/*.sql}
  ├─ api/dist/{index.js,app.js,gameHub.js,...}
  └─ web/.next/standalone/apps/web/server.js
             + web/.next/static (image 단계에서 별도 복사)

pnpm verify:build
  └─ 대표 path가 없으면 non-zero
```

| 불변 조건 | 처음 만들어진 commit | 자동 검증 | 남는 한계 |
| --- | --- | --- | --- |
| shared production import가 `dist`를 가리킨다 | `37c735de0c37` | 대표 JS·d.ts 존재 | import 실행은 안 함 |
| DB가 compiled CLI와 SQL asset을 만든다 | `430389943b34` | CLI/migrator/일부 SQL 존재 | lookup path는 별도 `30aac...`에서 보완 |
| API `start`가 emitted Node process다 | `bb67a72882bf` | `dist/index.js` 등 존재 | process start는 안 함 |
| Web이 monorepo-aware standalone server를 만든다 | `bb67a72882bf` | standalone `server.js` 존재 | HTTP serving·static asset은 안 봄 |
| root build가 dependency order를 가진다 | `bb67a72882bf` | CI에서 command 실행 | 중복 build·cache 전략은 최적화하지 않음 |
| 대표 artifact가 CI postcondition이다 | `6ab091ffa815`, `09b305b49768` | fresh job에서 existence check | upload·signature·runtime smoke 없음 |

이 Thread는 “typecheck가 통과했다”를 “배포 consumer가 읽을 파일이 생겼다”로 강화했다. 다음 단계는 그 파일을 image에 넣고, source나 build tool 없이 process를 시작하며, migration·health·gateway 순서로 외부에 공개하는 것이다.

## 조사 범위

표시된 SHA와 parent diff, package manifest, build tsconfig, Next config, migrator source를 exact historical state에서 확인했다. `30aac132e14e`는 다른 persistence Thread에 속하므로 commit map에 중복 편입하지 않고, `430389...`의 runtime asset path를 실제로 보완하는 cross-Thread dependency로만 명시했다. build와 verifier는 이 작업 환경에서 실행하지 않았다.
