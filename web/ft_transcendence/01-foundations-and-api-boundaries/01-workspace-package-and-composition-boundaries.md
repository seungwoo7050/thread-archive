# Thread: 모노레포 패키지 경계에서 서비스 조립까지

- 카테고리: `01-foundations-and-api-boundaries` — 애플리케이션 기반과 API 경계
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 하나의 저장소가 `shared`, `db`, `api`, `web`으로 나뉘고, API 진입점이 저장소 구현을 선택해 생성·초기화·종료하는 지점까지를 다룬다. 핵심은 디렉터리를 여러 개 만드는 일이 아니라 다음 책임을 서로 다른 위치에 고정하는 것이다.

- 루트는 workspace 목록과 공통 TypeScript 기준, 전체 검증 명령을 소유한다.
- `@pong-pong/shared`는 API와 브라우저가 함께 사용하는 전송 표현을 소유한다.
- `@pong-pong/db`는 PostgreSQL 드라이버, Kysely 인스턴스, migration SQL과 저장소 생명주기를 소유한다.
- `apps/api`의 composition root는 환경을 읽고 저장소 구현을 선택한 뒤 애플리케이션 시작과 종료를 조립한다.
- `apps/web`은 shared TypeScript source를 소비하는 브라우저 애플리케이션으로 추가된다.

모든 커밋이 중요도 `B`인 이유도 여기에 있다. 개별 커밋은 작고 대부분 기반 설정이지만, 열 개를 함께 보면 이후 기능이 어느 package에 놓여야 하는지와 장기 자원을 누가 닫아야 하는지가 처음으로 드러난다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b625c4f9dfdc` | `chore(workspace): pnpm 모노레포 경계 구성` | B | `PERSISTENCE` | 루트 명령과 `apps/*`, `packages/*` workspace를 만들고 공통 TypeScript 기준을 둔다. |
| 2 | `7753ad1fafaf` | `chore(shared): 공유 패키지 경계 구성` | B | `PROTOCOL` | API와 browser가 함께 소비할 `@pong-pong/shared` workspace를 독립 package로 만든다. |
| 3 | `573d11acb75e` | `feat(shared): 사용자와 서비스 DTO 정의` | B | `PERSISTENCE, TOURNAMENT, WEB` | 사용자·경기·대시보드·친구·채팅·토너먼트의 초기 compile-time DTO와 공개/세션 개인정보 경계를 정의한다. |
| 4 | `41471c2c2d55` | `feat(shared): 퐁 시뮬레이션 계약 추가` | B | `SIMULATION, REALTIME, WEB` | 서버 simulation과 browser renderer가 공유할 geometry·timing·snapshot·finished-result 계약을 정의한다. |
| 5 | `f77297697c66` | `chore(db): PostgreSQL 패키지 경계 구성` | B | `PERSISTENCE` | PostgreSQL, Kysely, pg 의존성을 API와 분리한 `@pong-pong/db` workspace를 만든다. |
| 6 | `1140fb868714` | `feat(db): migration 실행 경계 구성` | B | `PERSISTENCE` | DB package를 import 가능하게 만들고 초기 schema SQL을 runtime에서 실행할 entry로 노출한다. |
| 7 | `9277572765e7` | `feat(db): 저장소 lifecycle 구성` | B | `PERSISTENCE, OPERATIONS` | PostgreSQL과 memory implementation을 공통 `AppRepository` lifecycle 뒤에 두고 생성·초기화·종료 owner를 정의한다. |
| 8 | `51484e00a1c2` | `chore(api): Fastify 패키지 경계 구성` | B | `AUTH, PROTOCOL, REALTIME` | Fastify HTTP·cookie·CORS·WebSocket과 shared/DB dependency를 가진 API workspace를 구성한다. |
| 9 | `4b43a284e637` | `feat(api): 실행 환경과 service bootstrap 구성` | B | `PERSISTENCE, OPERATIONS` | 환경을 읽고 repository를 선택·초기화한 뒤 Fastify를 시작하고 종료 시 repository를 닫는 composition root를 도입한다. |
| 10 | `f5c151c7cc7d` | `chore(web): Next.js runtime 경계 구성` | B | `PROTOCOL, PERSISTENCE, WEB` | Next.js browser application workspace와 shared source transpilation 경계를 구성한다. |

## 1. 루트는 “모든 코드를 소유하는 package”가 아니라 조정자다

### `b625c4f9dfdc` — workspace와 공통 컴파일 기준

첫 커밋은 `apps/*`와 `packages/*`를 pnpm workspace로 선언하고, 루트 `build`와 `typecheck`를 각 package의 명령으로 위임한다.

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```json
{
  "scripts": {
    "build": "pnpm -r build",
    "typecheck": "pnpm -r typecheck"
  }
}
```

`tsconfig.base.json`은 `ES2022`, `moduleResolution: "Bundler"`, `strict: true`를 공통 기준으로 둔다. 이 시점의 루트는 애플리케이션 구현이나 외부 자원을 만들지 않는다. package를 발견하고 동일한 컴파일 의미로 검사하도록 조정할 뿐이다.

이 커밋이 보장하는 것은 **정해진 glob 아래의 package가 공통 명령에 참여한다는 것**이다. dependency 방향이나 순환 참조를 검사하는 규칙은 아직 없다.

### `7753ad1fafaf` — shared package의 빈 자리부터 만든 이유

`@pong-pong/shared`는 API나 web보다 먼저 독립 package로 생긴다. package entry는 빌드된 JavaScript가 아니라 `./src/index.ts`를 직접 가리키고, package 자체의 `build`도 `tsc --noEmit`이다. 따라서 이 단계의 “build”는 배포 artifact 생산이 아니라 source typecheck다.

루트 path mapping도 consumer가 같은 source를 해석하도록 연결한다. 이 결정은 계약의 원본을 한곳에 두지만, consumer가 TypeScript source를 실제로 처리할 수 있어야 한다는 조건을 남긴다. 이 조건은 web package를 추가할 때 `transpilePackages`로 다시 나타난다.

## 2. 공유 package가 먼저 소유한 것은 구현이 아니라 표현이다

### `573d11acb75e` — 공개 사용자와 세션 사용자를 분리한다

`packages/shared/src/http.ts`에는 최초의 HTTP DTO가 들어온다. 가장 중요한 분리는 `PublicUser`와 `SessionUser`다.

```ts
export interface PublicUser {
  id: string;
  handle: string;
  displayName: string;
  avatarKey: string;
  role: UserRole;
  status: UserStatus;
  rating: number;
  wins: number;
  losses: number;
  online: boolean;
}

export interface SessionUser extends PublicUser {
  email: string | null;
}
```

동일 파일은 경기 결과, 대시보드, 순위표, 친구, 채팅, 토너먼트 표현도 정의한다. 이때의 `interface`는 컴파일러가 사용하는 정적 계약일 뿐이다. 네트워크에서 들어온 JSON이나 저장소가 반환한 객체를 실행 시점에 검증하지 않는다. 이후 HTTP contract Thread에서 이 한계가 Zod schema 전환의 출발점이 된다.

### `41471c2c2d55` — simulation과 renderer가 같은 좌표계를 사용한다

두 번째 shared 변경은 게임 규칙과 snapshot 표현을 추가한다.

```ts
export const GAME_WIDTH = 960;
export const GAME_HEIGHT = 540;
export const WINNING_SCORE = 3;
export const TICK_RATE = 20;

export interface GameSnapshot {
  roomId: string;
  phase: GamePhase;
  tick: number;
  leftScore: number;
  rightScore: number;
  paddles: Record<PlayerSide, PaddleState>;
  ball: BallState;
  players: PlayerSlot[];
  serverTime: string;
}
```

여기서 shared package는 simulation loop나 canvas renderer를 구현하지 않는다. 양쪽이 같은 필드 이름, 좌표 크기, tick rate, 종료 결과를 해석하도록 표현만 소유한다. 따라서 dependency 방향은 “서버와 브라우저가 shared를 소비”하는 방향이지, shared가 API·DB·React 구현을 가져오는 방향이 아니다.

## 3. DB package는 SQL 문자열에서 장기 자원 소유자로 발전한다

### `f77297697c66` — 드라이버 의존성을 API 밖으로 옮긴다

`@pong-pong/db`는 `pg`, `kysely`, `@pong-pong/shared`를 자신의 dependency로 선언한다. 아직 repository 구현이나 실행 entry는 없지만, PostgreSQL-specific dependency가 어느 package에 속하는지는 이때 결정된다.

### `1140fb868714` — 제목보다 실제 diff가 보장하는 범위

이 커밋의 제목은 “migration 실행 경계”이지만, 실제 diff가 직접 추가하는 것은 다음 두 가지다.

1. `@pong-pong/db`의 package export와 typecheck 명령
2. `initialMigrationSql`이라는 SQL 문자열

SQL은 `users`, `sessions`, `friendships`, `matches`, `chat_messages`, `tournaments`, `tournament_entries`, `admin_actions`와 두 index를 만든다. 그러나 **이 커밋 자체에는 SQL을 실행하는 runner가 없다.** 실행은 다음 커밋의 `ensureSeedData()`에서 처음 연결된다.

```ts
export const initialMigrationSql = `
create extension if not exists pgcrypto;

create table if not exists users (...);
create table if not exists sessions (...);
create table if not exists friendships (...);
/* ... */
`;
```

따라서 이 SHA에서 확정되는 것은 “초기 schema를 import 가능한 값으로 노출한다”는 사실이다. migration version table, 순차 migration registry, rollback은 아직 없다.

### `9277572765e7` — repository가 Pool과 Kysely를 소유한다

이 커밋부터 실제 장기 자원이 생긴다.

```ts
export interface AppRepository {
  close(): Promise<void>;
  ensureSeedData(): Promise<void>;
}

export function createPostgresRepository(databaseUrl: string): AppRepository {
  const pool = new Pool({ connectionString: databaseUrl });
  const db = new Kysely<Database>({
    dialect: new PostgresDialect({ pool })
  });
  return new PostgresRepository(db, pool);
}
```

`PostgresRepository`는 `Pool`과 `Kysely`를 private field로 보유한다. `ensureSeedData()`는 앞 커밋의 SQL을 `sql.raw(...).execute(this.db)`로 실행하고, `close()`는 `db.destroy()` 뒤 `pool.end()`를 시도한다. memory 구현은 같은 interface를 만족하지만 두 메서드가 no-op이다.

```ts
async close(): Promise<void> {
  await this.db.destroy();
  await this.pool.end().catch(() => undefined);
}

async ensureSeedData(): Promise<void> {
  await sql.raw(initialMigrationSql).execute(this.db);
}
```

이 구조의 경계는 명확하다.

| 대상 | 생성·보유 주체 | 외부에 노출되는 책임 |
| --- | --- | --- |
| PostgreSQL `Pool` | `createPostgresRepository` / `PostgresRepository` | 직접 노출하지 않음 |
| `Kysely<Database>` | `PostgresRepository` | 직접 노출하지 않음 |
| 초기 schema 실행 | repository | `ensureSeedData()` |
| 자원 종료 | repository | `close()` |
| backend 선택 | 아직 없음 | 다음 composition root가 담당 |

`AppRepository`가 생겼다고 해서 초기화 실패 시 자동 정리가 보장되는 것은 아니다. caller가 `ensureSeedData()`와 `close()`를 올바른 순서로 호출해야 한다.

## 4. API package와 composition root가 runtime을 조립한다

### `51484e00a1c2` — HTTP 서버의 dependency 목록만 먼저 고정한다

`apps/api`는 Fastify, cookie, CORS, WebSocket plugin과 `@pong-pong/shared`, `@pong-pong/db`를 dependency로 갖는다. `dev`/`start`는 `tsx`, `build`/`typecheck`는 `tsc --noEmit`을 사용한다.

이 커밋은 아직 서버를 시작하지 않는다. package boundary와 사용할 runtime 기술만 정한다.

### `4b43a284e637` — backend 선택, 초기화, 시작, 종료

`apps/api/src/index.ts`가 처음 composition root 역할을 맡는다.

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

try {
  await app.listen({ port: env.port, host: "0.0.0.0" });
} catch (error) {
  app.log.error(error);
  await repo.close();
  process.exit(1);
}
```

정상 흐름은 다음과 같다.

```text
readEnv
  → databaseUrl 유무로 repository 구현 선택
  → ensureSeedData
  → buildApp(repo 주입)
  → Fastify onClose에 repo.close 등록
  → listen
```

여기서 중요한 미완성 경로가 하나 있다. `repo.ensureSeedData()`는 `try` 블록 **밖**에서 실행된다. 따라서 PostgreSQL repository를 만든 뒤 초기 schema 실행이 실패하면, 이 SHA의 코드에는 해당 실패를 잡아 `repo.close()`를 호출하는 경로가 없다. 반면 `app.listen()` 실패는 catch에서 repository를 닫고, 정상적인 Fastify close는 `onClose` hook을 통해 닫는다.

| 종료 지점 | `repo.close()` 보장 |
| --- | --- |
| Fastify 정상 close | `onClose` hook이 호출 |
| `app.listen()` throw | catch에서 호출 |
| `readEnv()` throw | repository가 아직 없음 |
| `createPostgresRepository()` 중 throw | repository 객체가 반환되기 전 |
| `ensureSeedData()` throw | 이 SHA에서는 명시적 close 경로가 없음 |

또한 `readEnv`의 `SESSION_SECRET`은 이 커밋에서 읽기만 하고 `buildApp`에 전달하지 않는다. 환경 parser가 값을 가진다는 사실과 실제 subsystem이 이를 소비한다는 사실을 구분해야 한다.

## 5. web package는 shared source를 소비하지만 DB 차단 규칙까지 만들지는 않는다

### `f5c151c7cc7d` — Next.js consumer 추가

`apps/web`은 Next.js·React package로 추가되고 `@pong-pong/shared`를 workspace dependency로 선언한다. `next.config.mjs`는 shared source를 transpile한다.

```js
const nextConfig = {
  transpilePackages: ["@pong-pong/shared"]
};
```

이 커밋 기준으로 web package manifest에는 `@pong-pong/db` dependency가 없고, Next transpilation 목록에도 shared만 있다. 따라서 **실제 추가된 browser runtime은 shared contract만 소비한다.**

다만 루트 `tsconfig`의 path mapping에는 이전 커밋에서 `@pong-pong/db`도 추가돼 있다. 그러므로 “web에서 DB import를 구조적으로 불가능하게 만든다”거나 “lint가 자동 차단한다”고까지 말할 수는 없다. 이 SHA가 보여주는 것은 현재 dependency와 source가 DB 구현을 포함하지 않는다는 사실이지, 미래의 잘못된 import를 강제 차단하는 별도 검증 규칙이 아니다.

## 최종 경계와 남은 위험

```text
[root workspace]
  ├─ packages/shared  ── transport/game representation
  ├─ packages/db      ── SQL, Kysely, Pool, repository lifecycle
  ├─ apps/api         ── HTTP runtime + repository 선택/초기화/종료 조립
  └─ apps/web         ── Next.js + shared source transpilation
```

| 불변 조건 | 이 Thread에서 확인한 근거 | 남는 범위 |
| --- | --- | --- |
| 공통 계약의 원본은 shared package에 있다 | HTTP DTO와 game snapshot을 shared에서 export | 초기 계약은 compile-time only |
| PostgreSQL 자원은 repository 내부에 있다 | `Pool`/`Kysely` private field와 `close()` | caller가 close를 호출해야 함 |
| backend 선택은 composition root가 한다 | `DATABASE_URL` 유무 분기 | production에서 memory fallback을 막는 규칙은 후속 Thread |
| 정상 서버 종료는 repository 종료로 이어진다 | Fastify `onClose` hook | 초기화 실패 전 경로는 미정리 |
| web은 shared source를 transpile한다 | package dependency와 `transpilePackages` | DB import 금지 규칙은 별도로 없음 |

이 Thread의 마지막 상태는 “각 package가 무엇을 담는가”뿐 아니라 “실행 중 자원을 누가 끝까지 책임지는가”를 처음 표현한다. 그러나 startup을 하나의 예외 안전한 transaction으로 만들거나 production capability를 검증하는 일은 아직 완료되지 않았다.

> 검사 범위: 지정 브랜치의 exact SHA diff와 해당 SHA source를 확인했다. 로컬 checkout, build, typecheck, runtime test는 실행하지 않았다.
