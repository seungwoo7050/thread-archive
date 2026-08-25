# Thread: Migration·seed·readiness·reset — schema 변경과 데이터 준비의 수명주기 분리

## 개요

이 Thread는 하나의 `ensureSeedData()`에 섞여 있던 schema 생성과 개발 데이터 주입을 분리하고, 그 결과를 운영자가 관찰하고 시험 환경에서 안전하게 초기화할 수 있는 수명주기로 정리합니다.

초기 CLI에서 `migrate`와 `seed`는 이름만 달랐습니다. 둘 다 repository를 만든 뒤 `ensureSeedData()`를 호출했고, 그 메서드는 SQL schema까지 적용했습니다. 이후 migration은 정렬된 SQL 파일과 Kysely migration table이 소유하고, seed는 명시적인 `development`·`demo` profile만 다룹니다. API startup은 둘 중 어느 것도 자동 실행하지 않으며, readiness는 database 연결과 migration 집합 상태를 별도로 보고합니다.

마지막 단계의 `reset:test`는 destructive operation이므로 실행 기능보다 **대상 판별**이 먼저 도입됩니다. 전용 test database나 정확한 형식의 격리 schema만 허용한 뒤, 선택된 schema만 drop/create하고 migration을 다시 적용합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1140fb868714` | `feat(db): migration 실행 경계 구성` | B | `PERSISTENCE` | SQL schema를 package 내부 TypeScript 상수로 노출해 첫 실행 경계를 만듭니다. |
| 2 | `dea169d587a3` | `feat(db): 데이터베이스 CLI 명령 연결` | B | `PERSISTENCE` | `migrate`·`seed`·`memory-smoke` 명령을 추가하지만 migration과 seed를 아직 구분하지 않습니다. |
| 3 | `f9bb622a1117` | `refactor(db): SQL migration lifecycle 분리` | A | `PERSISTENCE, RISK, REFACTOR` | SQL 파일·Kysely Migrator가 schema를 소유하도록 바꾸고 seed에서 DDL을 제거합니다. |
| 4 | `8da6edef28eb` | `feat(db): 환경별 seed profile 분리` | B | `AUTH, REALTIME, PERSISTENCE` | development와 demo가 생성할 계정 집합을 명시적으로 구분합니다. |
| 5 | `981ee655559b` | `refactor(db): migration과 seed CLI 연결` | B | `PERSISTENCE` | CLI 이름과 실제 실행 경로를 일치시키고 memory smoke의 불필요한 DB 의존을 제거합니다. |
| 6 | `30aac132e14e` | `feat(db): migration set 상태 검사 추가` | A | `PERSISTENCE, OPERATIONS, RISK` | bundled SQL 이름과 적용 이력을 비교해 current/pending/diverged를 계산합니다. |
| 7 | `2f05d5d79c64` | `feat(db): repository readiness 경계 추가` | A | `PERSISTENCE, OPERATIONS` | 연결 가능성과 migration 상태를 repository contract로 노출합니다. |
| 8 | `e1a0316fbe84` | `fix(api): startup seed 생성을 제거` | B | `PERSISTENCE` | API process 시작을 데이터 변경과 분리합니다. |
| 9 | `5cac4843fd9b` | `test(api): startup seed 금지 검증` | B | `REALTIME, TEST` | entry source에 직접 seed 호출이 다시 들어오지 못하도록 회귀 guard를 둡니다. |
| 10 | `113b3c422192` | `feat(db): test database reset target guard 추가` | A | `PERSISTENCE` | 파괴 가능한 대상의 protocol·database·schema·환경 조건을 선검증합니다. |
| 11 | `434403a7c16a` | `feat(db): test schema reset과 migration 실행 연결` | A | `PERSISTENCE, RISK` | 허용된 schema를 교체하고 최신 migration을 다시 적용하는 CLI를 만듭니다. |
| 12 | `527b5f137425` | `test(db): test database reset guard 검증` | B | `PERSISTENCE, TEST` | unsafe target 거부와 선택 schema만 초기화되는 동작을 unit/integration으로 고정합니다. |

## 1. 이름은 분리됐지만 책임은 아직 하나였던 초기 CLI

### `1140fb868714` — migration “파일 로더”가 아니라 inline SQL 상수

이 commit은 package export와 build/typecheck 명령을 추가하고 `packages/db/src/migrations.ts`에 `initialMigrationSql` 문자열을 둡니다.

```ts
export const initialMigrationSql = `
create extension if not exists pgcrypto;

create table if not exists users (...);
create table if not exists sessions (...);
/* ... */
`;
```

중요한 사실은 이 시점에 `.sql` asset을 runtime에서 읽지 않는다는 것입니다. schema 전체를 TypeScript 문자열로 다시 복제했으며, repository의 `ensureSeedData()`가 이 문자열을 실행합니다. SQL 원본과 실행 경로가 하나의 migration ledger에 묶여 있지 않으므로 파일 추가 순서, 적용 이력, pending 상태를 관리할 수 없습니다.

### `dea169d587a3` — `migrate`와 `seed`가 같은 일을 수행

초기 CLI는 명령 이름을 나누지만 실행은 다음과 같습니다.

```ts
const command = process.argv[2];
const databaseUrl = process.env.DATABASE_URL;

if (!databaseUrl) {
  throw new Error("DATABASE_URL is required for database CLI commands");
}

const repo = createPostgresRepository(databaseUrl);

try {
  if (command === "migrate" || command === "seed") {
    await repo.ensureSeedData();
    console.log(command === "migrate" ? "migrated" : "seeded");
  } else if (command === "memory-smoke") {
    /* ... */
  }
} finally {
  await repo.close();
}
```

여기에는 세 가지 문제가 있습니다.

1. `migrate`도 개발 사용자 seed를 만들고, `seed`도 schema를 변경합니다.
2. `memory-smoke`도 분기 전에 `DATABASE_URL`을 요구합니다.
3. schema 적용 이력이 없으므로 “이미 적용됨”, “새 파일이 pending”, “알 수 없는 migration이 존재함”을 구분할 수 없습니다.

CLI는 `finally`에서 PostgreSQL repository를 닫는 수명주기는 갖췄지만, 명령의 이름과 실제 부작용은 아직 일치하지 않습니다.

## 2. Schema 변경과 데이터 준비를 분리

### `f9bb622a1117` — SQL 파일과 Kysely Migrator가 schema를 소유

이 refactor는 `initialMigrationSql` 상수를 삭제하고, `packages/db/migrations/*.sql`을 이름순으로 읽는 `MigrationProvider`를 도입합니다. repository의 `ensureSeedData()`에서는 DDL 실행이 제거됩니다.

```ts
class SqlMigrationProvider implements MigrationProvider {
  async getMigrations(): Promise<Record<string, Migration>> {
    const filenames = (await readdir(migrationsDirectory))
      .filter((filename) => extname(filename) === ".sql")
      .sort();

    const migrations = await Promise.all(
      filenames.map(async (filename) => {
        const statement = await readFile(join(migrationsDirectory, filename), "utf8");
        return [
          basename(filename, ".sql"),
          {
            up: async (db) => {
              await sql.raw(statement).execute(db);
            }
          } satisfies Migration
        ] as const;
      })
    );
    return Object.fromEntries(migrations);
  }
}
```

```ts
export async function migrateDatabase(databaseUrl: string): Promise<void> {
  const pool = new Pool({ connectionString: databaseUrl });
  const db = new Kysely<Database>({ dialect: new PostgresDialect({ pool }) });

  try {
    const migrator = new Migrator({ db, provider: new SqlMigrationProvider() });
    const { error, results } = await migrator.migrateToLatest();
    if (error) {
      const failed = results?.find((result) => result.status === "Error");
      throw new Error(
        `Database migration failed${failed ? ` (${failed.migrationName})` : ""}`,
        { cause: error }
      );
    }
  } finally {
    await db.destroy();
  }
}
```

이 시점부터 책임은 다음처럼 나뉩니다.

| 작업 | 소유자 | 상태 기록 |
| --- | --- | --- |
| schema 변경 | SQL migration 파일 + Kysely Migrator | `kysely_migration` |
| 개발/demo 데이터 생성 | `ensureSeedData(profile)` | 대상 table의 실제 row |
| connection 종료 | 각 command가 만든 migrator/repository | `finally` |

각 SQL 파일은 하나의 `up` migration으로 실행되며 이 commit에는 down migration이 없습니다. SQL 파일 내용 중 schema 확장도 함께 변경되지만, 이 Thread에서는 DDL 내용보다 lifecycle 분리를 다룹니다.

### `8da6edef28eb` — seed profile의 데이터 범위 명시

`SeedProfile = "development" | "demo"`가 repository contract에 추가됩니다.

```ts
async ensureSeedData(profile: SeedProfile = "development"): Promise<void> {
  if (profile === "development") {
    // 개발 사용자·admin 생성 및 rating 보정
  }
  for (const npc of NPC_PLAYERS) {
    await this.upsertNpc(npc);
  }
}
```

- `development`: 개발 사용자, admin, NPC를 모두 준비합니다.
- `demo`: NPC만 준비합니다.
- 기본값: `development`입니다.

PostgreSQL과 memory 구현이 같은 profile 분기를 갖지만, 이 메서드는 여전히 명시적으로 호출해야 합니다. profile은 runtime environment를 자동 추론하지 않습니다.

### `981ee655559b` — CLI 이름과 실행 부작용을 일치시킴

CLI는 `migrate`, `seed:dev`, `seed:demo`, `memory-smoke`를 별도 경로로 분리합니다.

```ts
if (command === "memory-smoke") {
  const memory = createMemoryRepository();
  await memory.ensureSeedData();
  await memory.close();
} else {
  const databaseUrl = requireDatabaseUrl();

  if (command === "migrate") {
    await migrateDatabase(databaseUrl);
  } else {
    const repo = createPostgresRepository(databaseUrl);
    try {
      await repo.ensureSeedData(
        command === "seed:dev" ? "development" : "demo"
      );
    } finally {
      await repo.close();
    }
  }
}
```

`memory-smoke`는 더 이상 PostgreSQL URL을 요구하지 않습니다. 다만 이 commit의 API startup에는 memory backend일 때 `ensureSeedData()`를 자동 호출하는 코드가 남아 있습니다. 해당 implicit mutation은 뒤의 `e1a0316fbe84`에서 제거됩니다.

## 3. 적용 여부를 “연결 성공”과 분리해 관찰

### `30aac132e14e` — migration 집합을 set difference로 비교

source 실행과 build 산출물에서 migration directory 위치가 다를 수 있어 `./migrations`, `../migrations` 두 후보를 순서대로 찾습니다. 이어 bundled filename과 `kysely_migration.name`을 비교합니다.

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
    status:
      unexpected.length > 0
        ? "diverged"
        : missing.length > 0
          ? "pending"
          : "current",
    missing,
    unexpected
  };
}
```

판정 규칙은 다음과 같습니다.

| 상태 | 조건 |
| --- | --- |
| `current` | expected와 applied의 원소가 모두 일치 |
| `pending` | unexpected는 없고 expected 중 일부가 미적용 |
| `diverged` | applied에 bundled set에 없는 이름이 하나라도 존재 |

이 구현은 **순서가 올바른 prefix인지 검사하지 않습니다.** 두 배열의 원소 집합이 같으면 순서가 달라도 `current`입니다. 또한 checksum이나 SQL 내용 변경도 비교하지 않습니다. 기존 migration table이 없을 때의 PostgreSQL 오류 `42P01`만 “적용된 migration 없음”으로 해석하고 다른 오류는 전파합니다.

### `2f05d5d79c64` — database와 migration 상태를 하나의 readiness 결과로 노출

```ts
export interface RepositoryReadiness {
  database: "up";
  migrations: "current" | "pending" | "diverged" | "not_applicable";
}

async checkReadiness(): Promise<RepositoryReadiness> {
  await sql<{ ok: number }>`select 1 as ok`.execute(this.db);
  const migrationSet = await inspectMigrationSet(this.db);
  return { database: "up", migrations: migrationSet.status };
}
```

PostgreSQL은 먼저 실제 query를 실행한 뒤 migration 상태를 검사합니다. 연결 또는 query가 실패하면 `{ database: "down" }` 같은 값으로 변환하지 않고 promise가 reject됩니다. 따라서 반환된 `database: "up"`은 성공한 경우에만 존재합니다.

Memory repository는 `{ database: "up", migrations: "not_applicable" }`를 반환합니다. 이는 memory가 migration 최신 상태라는 뜻이 아니라 migration 개념을 적용하지 않는다는 뜻입니다.

## 4. API startup을 데이터 변경과 분리

### `e1a0316fbe84` — implicit memory seed 제거

API entry에서 다음 코드만 삭제합니다.

```diff
 const repo = env.databaseUrl
   ? createPostgresRepository(env.databaseUrl)
   : createMemoryRepository();
-if (!env.databaseUrl) {
-  await repo.ensureSeedData();
-}
```

이후 API process를 시작하는 행위는 persistent/memory backend 어느 쪽에도 seed row를 만들지 않습니다. 개발·demo 데이터가 필요하면 별도 CLI나 test fixture가 명시적으로 준비해야 합니다.

### `5cac4843fd9b` — source-level 회귀 guard

시험은 API entry source를 문자열로 읽어 직접 `.ensureSeedData(` 호출이 없는지 검사합니다.

```ts
const source = readFileSync(
  fileURLToPath(new URL("./index.ts", import.meta.url)),
  "utf8"
);
expect(source).not.toMatch(/\.ensureSeedData\s*\(/);
```

이 방식은 entry file에 직접 호출이 다시 들어오는 회귀를 매우 명확하게 막습니다. 반면 imported helper가 내부에서 seed를 호출하는 간접 경로, 다른 startup module의 mutation, 실제 빈 저장소 상태까지는 증명하지 않습니다.

같은 commit의 WebSocket smoke 변경은 seeded NPC 의존을 없애기 위한 후속 조정이지만, 이 Thread의 persistence 설명에는 startup guard만 포함합니다.

## 5. 파괴적 test reset은 대상 판별부터 시작

### `113b3c422192` — 허용 목록이 아니라 형식 기반 deny-by-default guard

`resolveTestResetTarget()`은 drop/create를 수행하지 않습니다. 먼저 environment와 URL을 해석해 안전한 대상으로 확정하거나 즉시 거부합니다.

```ts
const ISOLATED_TEST_SCHEMA = /^test_[a-f0-9]{32}$/;
const DEDICATED_TEST_DATABASE =
  /^(?:test(?:_[a-z0-9][a-z0-9_-]*)?|[a-z0-9][a-z0-9_-]*_test)$/;
```

허용 조건은 다음과 같습니다.

1. `NODE_ENV === "test"`
2. `DATABASE_URL`이 아니라 별도 `TEST_DATABASE_URL` 사용
3. protocol이 `postgres:` 또는 `postgresql:`
4. database name이 비어 있지 않고 decode 후 `/`를 포함하지 않음
5. `options`가 0개 또는 정확히 1개
6. options가 있으면 정확히 `-c search_path=test_<32 hex>` 하나
7. public schema라면 database 이름 자체가 명백한 test 이름
8. non-public schema라면 생성형 격리 schema 정규식과 정확히 일치

`public,other`, 수동 이름 `test_manual`, statement timeout을 섞은 options, 일반 application database는 모두 `Unsafe test reset target`으로 거부합니다.

### `434403a7c16a` — 선택 schema 교체 후 migration 재적용

검증된 schema만 identifier로 사용하므로 단순 double-quote wrapping이 허용됩니다. control connection에서는 `search_path` options를 제거하고 schema 자체를 drop/create합니다.

```ts
const target = resolveTestResetTarget(env);
const controlUrl = new URL(target.databaseUrl);
controlUrl.searchParams.delete("options");
const pool = new Pool({ connectionString: controlUrl.toString() });
const quotedSchema = `"${target.schema}"`;

try {
  const client = await pool.connect();
  try {
    await client.query("begin");
    await client.query(`drop schema if exists ${quotedSchema} cascade`);
    await client.query(`create schema ${quotedSchema}`);
    await client.query("commit");
  } catch (error) {
    await client.query("rollback").catch(() => undefined);
    throw error;
  } finally {
    client.release();
  }
} finally {
  await pool.end();
}

await migrateDatabase(target.databaseUrl);
```

여기서 transaction은 schema drop/create까지만 포함합니다. `migrateDatabase()`는 commit과 pool 종료 뒤 별도 connection으로 실행됩니다. 따라서 migration이 실패하면 이전 schema로 rollback되는 것이 아니라 **새로 만들어졌지만 migration이 덜 적용된 schema**가 남을 수 있습니다. 이 경로의 안전성 목표는 운영 DB를 지키고 대상 격리를 보장하는 것이며, reset+migrate 전체의 원자성까지 보장하지 않습니다.

### `527b5f137425` — guard와 실제 격리를 각각 검증

Unit matrix는 다음을 거부합니다.

- `NODE_ENV !== test`
- `TEST_DATABASE_URL` 누락
- 일반 application DB나 이름이 애매한 DB
- public과 다른 schema를 함께 지정한 search path
- 수동 schema 이름과 unrelated PostgreSQL option

Integration case는 대상 schema에 사용자를 만들고, 별도의 sibling schema에는 marker table을 둔 뒤 reset을 실행합니다. 이후 다음을 확인합니다.

- 대상 `users`는 0행
- repository readiness는 `database: up`, `migrations: current`
- sibling schema의 marker table은 그대로 존재

이 시험은 선택 schema만 파괴한다는 것을 직접 확인하지만, 실제 운영 credentials나 모든 PostgreSQL URL 변형을 포괄하지는 않습니다.

## 수명주기 최종 정리

```text
[배포/운영 준비]
  pnpm ... migrate
    -> bundled *.sql 읽기
    -> Kysely migrateToLatest
    -> migration table에 적용 이력 기록

[명시적 데이터 준비]
  pnpm ... seed:dev | seed:demo
    -> repository 생성
    -> profile별 upsert
    -> repository close

[API 시작]
  repository 생성
    -> 자동 migration 없음
    -> 자동 seed 없음
    -> 필요 시 checkReadiness로 연결/migration 상태 관찰

[test reset]
  NODE_ENV + TEST_DATABASE_URL 검증
    -> 안전한 DB/schema 판별
    -> 선택 schema drop/create transaction
    -> pool close
    -> 해당 search_path에 migration 재적용
```

## 보장 범위와 남는 한계

| 항목 | 보장 | 비보장 |
| --- | --- | --- |
| migration/seed 분리 | 명령과 코드 경로가 분리됨 | operator가 순서를 올바르게 실행하는 것 |
| migration 상태 | bundled/applied 이름 집합의 missing·unexpected 감지 | 순서 prefix, checksum, 수정된 SQL 내용 감지 |
| readiness | query 성공과 migration set 상태 보고 | failure를 `down` 값으로 정규화하거나 자동 복구 |
| startup | API entry의 직접 seed 호출 금지 | 모든 간접 startup mutation |
| test reset | test-only 대상 형식 검증, 선택 schema만 교체 | reset+migration 전체 원자성, 운영 credential 오용의 모든 형태 |

## 이 Thread의 경계

- migration 내부의 friendship/tournament 제약 의미는 Thread 4·5에서 설명합니다.
- row type·mapper 정렬은 Thread 3의 책임입니다.
- container image, Compose startup order, production deploy migration job은 제품 전달 카테고리의 책임입니다.
- 이 문서는 자동 rollback/down migration, migration checksum 정책, backup/restore 전략을 새로 설계하지 않습니다.

> 검증 기록: 위 설명은 `web/ft_transcendence` branch의 표시된 exact SHA diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 CLI·PostgreSQL integration suite를 실행하지 않았으며, test source가 선언한 assertion과 code path만 근거로 삼았습니다.
