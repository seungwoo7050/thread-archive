# Thread: server session — logout 폐기와 인증 migration

Project: `ft_transcendence` · Branch: `web/ft_transcendence` · Category: `03-identity-authorization-and-account-lifecycle`

## 개요

이 Thread는 등록 사용자의 session token을 repository가 발급·해석하는 최초 구현에서 시작해, logout의 의미를 **브라우저 cookie 제거**에서 **서버 credential 폐기**로 바로잡고, 인증 방식이 바뀐 뒤 남은 legacy session을 migration으로 일괄 만료하는 과정까지 다룬다.

여기에는 서로 다른 두 실패가 있다.

- 초기 logout은 `pp_session` cookie만 지웠다. 같은 token을 `Authorization: Bearer`나 query로 다시 보내면 repository row가 남아 있으므로 계속 인증됐다.
- 새 인증 계약으로 전환해도 과거에 발급된 PostgreSQL session row는 자동으로 사라지지 않았다. 새 코드만 배포해서는 이전 credential의 신뢰를 철회할 수 없었다.

최종적으로 session의 유효성은 client 저장 상태가 아니라 repository가 결정한다. 개별 logout은 요청에서 선택된 token 하나를 삭제하고, `005_expire_legacy_sessions` migration은 기존 PostgreSQL session을 전부 제거한다. migration 검증은 credential만 사라지고 사용자·경기·rating history가 유지되는지 실제 PostgreSQL 상태를 비교한다.

> 조사 범위: 아래 설명은 각 exact SHA의 diff와 그 SHA 시점의 source/test를 기준으로 작성했다. runtime test는 실행하지 않았으며 repository에 존재하는 test의 입력과 assertion만 분석했다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4f65c6214321` | feat(db): 사용자 session 저장 구현 | B | `AUTH, REALTIME, PERSISTENCE` | PostgreSQL row와 memory map으로 token 발급·해석을 도입한다 |
| 2 | `1779df300611` | feat(api): 로그인과 로비 HTTP 경계 구현 | B | `AUTH, PERSISTENCE, WEB` | 로그인에서 session을 만들고 여러 transport에서 token을 읽는다 |
| 3 | `252befef9527` | fix(api): logout 시 server session 폐기 | A | `AUTH, REALTIME, PERSISTENCE` | logout이 선택된 server token을 repository에서 삭제하게 한다 |
| 4 | `bc789124b20b` | test(api): logout session invalidation 검증 | B | `AUTH, SIMULATION, TEST` | 같은 bearer token의 logout 전후 인증 결과를 고정한다 |
| 5 | `b93910708330` | feat(db): legacy session을 안전하게 만료 | A | `AUTH, PERSISTENCE` | 인증 전환 시 기존 PostgreSQL session을 전부 폐기한다 |
| 6 | `0649b63a1ca9` | test(db): 인증 migration 중 데이터 보존 검증 | A | `AUTH, PERSISTENCE, TEST` | session 삭제와 durable game/account history 보존을 함께 검증한다 |

## 4f65c6214321 — feat(db): 사용자 session 저장 구현

**중요도** `B` · **태그** `AUTH, REALTIME, PERSISTENCE`

### 무엇을 만들었는가

`AppRepository`에 session 발급과 조회가 추가된다.

```diff
 export interface AppRepository {
   upsertDevUser(input: DevLoginInput): Promise<SessionUser>;
+  createSession(userId: string): Promise<string>;
+  getSessionUser(token: string | undefined): Promise<SessionUser | null>;
 }
```

PostgreSQL 구현은 UUID token과 14일 만료 시각을 `sessions`에 저장하고, 조회할 때 `users`와 join해 **현재 user row**를 반환한다.

```ts
async createSession(userId: string): Promise<string> {
  const token = randomUUID();
  await sql`
    insert into sessions (token, user_id, expires_at)
    values (${token}, ${userId}, now() + interval '14 days')
  `.execute(this.db);
  return token;
}

async getSessionUser(token: string | undefined): Promise<SessionUser | null> {
  if (!token) return null;
  const result = await sql<UserRow>`
    select u.*
    from sessions s
    join users u on u.id = s.user_id
    where s.token = ${token} and s.expires_at > now()
    limit 1
  `.execute(this.db);
  return result.rows[0] ? toSessionUser(result.rows[0], true) : null;
}
```

Memory 구현은 `Map<token, userId>`를 사용한다. PostgreSQL과 달리 별도 만료 시각을 저장하거나 검사하지 않는다. 따라서 두 backend가 공유하는 최소 계약은 “발급한 token으로 현재 사용자를 찾는다”이고, 시간 기반 만료까지 동일하다는 보장은 이 커밋에 없다.

이 시점에는 token 삭제 연산도 없다. 한 번 발급된 memory token은 repository가 살아 있는 동안, PostgreSQL token은 만료되거나 row가 외부에서 삭제될 때까지 유효하다.

## 1779df300611 — feat(api): 로그인과 로비 HTTP 경계 구현

**중요도** `B` · **태그** `AUTH, PERSISTENCE, WEB`

이 커밋은 로그인 route에서 user를 upsert하고 session을 만든 뒤, 같은 token을 cookie와 response body 두 곳으로 전달한다.

```ts
const user = await repo.upsertDevUser({ /* ... */ });
const token = await repo.createSession(user.id);
reply.setCookie("pp_session", token, {
  path: "/",
  sameSite: "lax",
  httpOnly: true,
  maxAge: 60 * 60 * 24 * 14
});
return { user, token };
```

인증 요청은 다음 우선순위로 token을 선택한다.

```ts
const cookieToken = request.cookies?.pp_session;
const header = request.headers.authorization?.replace(/^Bearer\s+/i, "");
const queryToken = (request.query as { session?: string } | undefined)?.session;
const rawQueryToken = new URL(request.raw.url ?? "/", "http://localhost")
  .searchParams.get("session") ?? undefined;
return repo.getSessionUser(cookieToken ?? header ?? queryToken ?? rawQueryToken);
```

### 이때의 logout 결함

logout은 request를 읽지 않고 cookie만 지운다.

```ts
app.post("/auth/logout", async (_request, reply) => {
  reply.clearCookie("pp_session", { path: "/" });
  return { ok: true };
});
```

이 구현에서 cookie는 credential 자체가 아니라 credential을 보관하는 한 장소일 뿐이다. 서버의 `sessions` row나 memory map entry는 그대로이므로 response body에서 받은 token을 bearer/query로 재사용할 수 있다. 즉 화면에서는 로그아웃된 것처럼 보여도 server session은 계속 살아 있다.

같은 커밋의 로비 조회와 CORS 구성은 session이 실제 HTTP route에 연결되는 배경이지만, logout 폐기 문제와 무관한 세부 변경은 이 Thread에서 제외한다.

## 252befef9527 — fix(api): logout 시 server session 폐기

**중요도** `A` · **태그** `AUTH, REALTIME, PERSISTENCE`

### 문제 → 원인 → 결정

- **문제**: cookie를 지운 뒤에도 같은 token을 bearer/query로 보내면 인증됐다.
- **원인**: logout이 client 저장소만 수정하고 token의 authoritative owner인 repository에는 아무 변경도 요청하지 않았다.
- **결정**: 인증과 logout이 같은 token 선택 함수를 사용하게 하고, logout에서는 cookie를 지우기 전에 그 token을 repository에서 삭제한다.

```diff
-app.post("/auth/logout", async (_request, reply) => {
+app.post("/auth/logout", async (request, reply) => {
+  await repo.deleteSession(readSessionToken(request));
   reply.clearCookie("pp_session", { path: "/" });
   return { ok: true };
 });
```

```ts
function readSessionToken(request: FastifyRequest): string | undefined {
  const cookieToken = request.cookies?.pp_session;
  const header = request.headers.authorization?.replace(/^Bearer\s+/i, "");
  const queryToken = (request.query as { session?: string } | undefined)?.session;
  const rawQueryToken = new URL(request.raw.url ?? "/", "http://localhost")
    .searchParams.get("session") ?? undefined;
  return cookieToken ?? header ?? queryToken ?? rawQueryToken;
}
```

Repository 양쪽에 같은 의미의 삭제 연산이 추가된다.

```ts
// PostgreSQL
if (token) {
  await sql`delete from sessions where token = ${token}`.execute(this.db);
}

// Memory
if (token) this.sessions.delete(token);
```

### 바뀐 불변 조건

logout 성공 후에는 `readSessionToken`이 선택한 credential을 다시 제시해도 `getSessionUser`가 사용자를 반환하지 않아야 한다. cookie 제거는 그 다음에 수행되는 client 정리이며, 인증 폐기의 근거는 repository deletion이다.

다만 이 연산은 **현재 선택된 token 하나**만 삭제한다. 같은 사용자의 다른 device/session 전체를 찾아 폐기하지 않는다. 또한 요청에 cookie와 bearer가 동시에 있으면 정해진 우선순위상 cookie token이 선택되므로 “모든 전달 credential을 동시에 폐기”하는 API도 아니다.

## bc789124b20b — test(api): logout session invalidation 검증

**중요도** `B` · **태그** `AUTH, SIMULATION, TEST`

테스트는 cookie UI를 확인하지 않고 server credential 재사용 가능성을 직접 겨냥한다.

```ts
const login = await app.inject({
  method: "POST",
  url: "/auth/dev-login",
  payload: { handle: "logout-tester", displayName: "로그아웃" }
});
const token = login.json<{ token: string }>().token;

expect((await app.inject({
  method: "GET", url: "/me",
  headers: { authorization: `Bearer ${token}` }
})).statusCode).toBe(200);

expect((await app.inject({
  method: "POST", url: "/auth/logout",
  headers: { authorization: `Bearer ${token}` }
})).statusCode).toBe(200);

expect((await app.inject({
  method: "GET", url: "/me",
  headers: { authorization: `Bearer ${token}` }
})).statusCode).toBe(401);
```

`200 → logout 200 → 401`이라는 한 state transition으로 cookie-only 구현으로의 회귀를 막는다. 같은 raw token을 bearer로 계속 사용하므로 browser가 cookie를 삭제했는지와 관계없이 repository entry가 사라졌는지를 관찰한다.

이 테스트는 Fastify injection과 memory repository를 통과한다. 실제 PostgreSQL `DELETE`, browser cookie jar, network transport, 여러 session의 동시 폐기는 증명하지 않는다.

## b93910708330 — feat(db): legacy session을 안전하게 만료

**중요도** `A` · **태그** `AUTH, PERSISTENCE`

개별 logout으로는 이미 발급돼 남아 있는 모든 credential을 회수할 수 없다. 인증 계약이 바뀐 시점에 이전 session을 계속 신뢰하지 않기로 한 결정은 migration 한 줄로 표현된다.

```sql
-- packages/db/migrations/005_expire_legacy_sessions.sql
delete from sessions;
```

조건절이 없으므로 “legacy인지 판별해 일부만 제거”하는 migration이 아니다. 005가 적용되는 database의 session row를 전부 지우고 모든 등록 사용자가 다시 인증하도록 한다. users나 matches를 건드리는 SQL은 없다.

같은 커밋에서 migrator는 특정 migration까지만 적용할 수 있게 바뀐다.

```diff
-export async function migrateDatabase(databaseUrl: string): Promise<void> {
+export async function migrateDatabase(
+  databaseUrl: string,
+  targetMigration?: string
+): Promise<void> {
   /* ... */
-  const { error, results } = await migrator.migrateToLatest();
+  const { error, results } = targetMigration
+    ? await migrator.migrateTo(targetMigration)
+    : await migrator.migrateToLatest();
 }
```

이 옵션은 production reset 정책 자체라기보다, 다음 테스트가 004 상태를 정확히 만든 뒤 005만 통과시키기 위한 재현 지점이다.

### 보장과 비용

- 보장: 005가 성공한 PostgreSQL database에는 기존 session row가 남지 않는다.
- 비용: 정상·비정상·구형 session을 구분하지 않고 모두 로그아웃시킨다.
- 비보장: memory repository의 살아 있는 token, traffic 중인 request와 migration의 조정, 무중단 재인증 절차는 이 SQL의 범위가 아니다.

## 0649b63a1ca9 — test(db): 인증 migration 중 데이터 보존 검증

**중요도** `A` · **태그** `AUTH, PERSISTENCE, TEST`

이 테스트는 migration의 두 주장—credential 전부 폐기와 durable history 비손상—을 분리해 확인한다.

### 재현 순서

1. 격리 PostgreSQL database를 `004_friendship_tournament_invariants`까지만 migration한다.
2. 사용자 두 명, winner의 session 하나, finalized match 하나를 만든다.
3. `users`, `matches`, `rating_history`에서 선택한 column을 정렬해 snapshot으로 보관한다.
4. latest migration을 적용한다.
5. 이전 token이 `null`이 되는지, `sessions` count가 0인지, 세 snapshot이 이전과 같은지, 005가 migration metadata에 기록됐는지 확인한다.

핵심 부분은 다음과 같다.

```ts
await migrateDatabase(databaseUrl, "004_friendship_tournament_invariants");

const sessionToken = await repository.createSession(winner.id);
await repository.finalizeMatch({
  resultKey: "room:auth-migration:finished",
  mode: "queue",
  winnerId: winner.id,
  loserId: loser.id,
  scoreLeft: 3,
  scoreRight: 1
});
const before = await authMigrationSnapshot(pool);

await migrateDatabase(databaseUrl);

await expect(repository.getSessionUser(sessionToken)).resolves.toBeNull();
expect(await authMigrationSnapshot(pool)).toEqual(before);
await expect(pool.query(
  "select count(*)::integer as count from sessions"
)).resolves.toMatchObject({ rows: [{ count: 0 }] });
expect(await appliedMigrations(pool)).toContain("005_expire_legacy_sessions");
```

### 무엇을 증명하는가

실제 PostgreSQL migration 경로에서 005가 적용되면 기존 token이 무효화되고 session table이 비며, 테스트가 선택한 사용자 통계·경기 결과·rating 변화 기록은 동일하다. 단순히 SQL 파일이 한 줄이라는 사실보다 강한 증거다. migration provider, repository 조회, raw SQL snapshot을 함께 통과한다.

### 무엇을 증명하지 않는가

snapshot에 포함되지 않은 모든 table/column의 완전 보존, production traffic과 동시에 수행되는 migration, backup/restore, memory backend의 credential 정리는 검증하지 않는다.

## 최종 상태

```text
[login]
  user upsert
    → repository.createSession(userId)
    → token + HttpOnly cookie 전달

[authenticated request]
  request에서 credential 하나 선택
    → repository.getSessionUser(token)
    → 현재 user row 또는 null

[individual logout]
  같은 선택 규칙으로 token 추출
    → repository.deleteSession(token)
    → pp_session cookie 제거

[auth contract migration]
  005 적용
    → DELETE FROM sessions
    → 모든 PostgreSQL legacy session 무효
    → users / matches / rating_history는 보존 대상
```

## 이 Thread의 경계

이 Thread는 **등록 session token의 생성·해석·개별 폐기와 인증 전환 시 일괄 만료**만 다룬다.

- WebSocket용 one-time ticket의 발급·hash 저장·단회 소비는 별도 realtime 인증 Thread의 문제다.
- cookie만 허용하도록 transport를 축소하고 bearer/query token을 제거하는 후속 인증 경계 변경은 이 Thread의 exact SHA보다 뒤에 있으며 여기로 소급하지 않는다.
- 한 사용자의 모든 device를 로그아웃시키는 operation, session 목록 UI, TTL 청소 작업은 포함되지 않는다.
- account suspension에 따른 기존 HTTP/관리자/WebSocket capability 회수는 이 카테고리의 다른 Thread가 다룬다.
- migration이 실행 중인 online request와 어떻게 조정되는지, 배포 실패 시 어떻게 보상하는지는 production delivery 범위다.
