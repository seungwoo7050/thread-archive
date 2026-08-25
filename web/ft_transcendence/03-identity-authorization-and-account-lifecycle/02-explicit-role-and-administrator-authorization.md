# Thread: 명시적 role과 관리자 권한 — 이름 기반 승격 제거부터 정지 상태 반영까지

Project: `ft_transcendence` · Branch: `web/ft_transcendence` · Category: `03-identity-authorization-and-account-lifecycle`

## 개요

이 Thread는 관리자 기능의 존재보다 **누가 관리자 capability를 얻는가**를 다룬다. 초기 구현은 server-side role 검사를 두고 있었지만, 일반 login/upsert가 handle이 `admin`이라는 이유만으로 role을 부여했다. route의 검사는 맞아 보여도, 검사에 사용되는 role 자체를 사용자가 선택 가능한 문자열에서 만들었기 때문에 권한 경계가 무너져 있었다.

후속 변경은 identity 생성과 privilege provisioning을 분리한다. 일반 upsert는 언제나 `user`를 만들고, 별도 repository/CLI operation만 non-NPC 계정의 role을 바꾼다. 그 뒤 남은 결함은 account status였다. `admin` role만 검사하면 이미 정지된 계정의 기존 session도 관리자 route를 계속 호출할 수 있으므로, 최종 `requireAdmin`은 다음 세 조건을 순서대로 검사한다.

```text
session에서 현재 사용자 해석
  → 없으면 401 unauthorized
  → status가 active가 아니면 403 account_suspended
  → role이 admin이 아니면 403 forbidden
  → 관리자 handler 실행
```

> 조사 범위: 모든 설명은 표시된 exact SHA의 diff/source에 한정한다. 테스트 코드는 분석했지만 runtime command는 실행하지 않았다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fa6e7de259cf` | feat(db): 관리자 상태 변경 저장 구현 | B | `AUTH, PERSISTENCE` | 사용자 목록과 ban/unban 저장 연산을 repository에 추가한다 |
| 2 | `e8bb6a4bf68b` | feat(api): 토너먼트와 관리자 라우트 추가 | B | `AUTH, PERSISTENCE, TOURNAMENT` | 관리자 route에 session·role 검사를 연결한다 |
| 3 | `1395d45a3665` | test(api): 관리자 사용자 상태 변경 검증 | B | `AUTH, PERSISTENCE, TEST` | 당시 `admin` handle fixture의 성공 경로를 고정한다 |
| 4 | `45225adcfcd9` | feat(db): 명시적 사용자 role 할당 추가 | A | `AUTH, PERSISTENCE` | handle 기반 승격을 제거하고 별도 role provisioning을 도입한다 |
| 5 | `dae31d4a223c` | test(auth): 명시적 role assignment 검증 | B | `AUTH, PERSISTENCE, TEST` | PostgreSQL에서 일반 upsert와 명시적 승격을 구분한다 |
| 6 | `c577fe2603e3` | fix(auth): 정지된 관리자 login 거부 | A | `AUTH, RISK` | 관리자 capability에 현재 account status를 추가한다 |
| 7 | `aa037c5291fe` | test(auth): 정지된 관리자 session 거부 검증 | B | `AUTH, TEST` | 이미 발급된 관리자 cookie도 ban 뒤 거부되는지 검증한다 |

## fa6e7de259cf — feat(db): 관리자 상태 변경 저장 구현

**중요도** `B` · **태그** `AUTH, PERSISTENCE`

`AppRepository`에 관리 대상 목록과 status mutation이 추가된다.

```diff
+listAdminUsers(): Promise<PublicUser[]>;
+setUserBan(
+  actorId: string,
+  targetUserId: string,
+  banned: boolean,
+  reason: string
+): Promise<PublicUser>;
```

PostgreSQL 구현은 target user를 갱신하고 administrator action을 기록한다.

```ts
const result = await sql<UserRow>`
  update users
  set status = ${banned ? "banned" : "active"},
      banned_at = ${banned ? sql`now()` : null}
  where id = ${targetUserId}
  returning *
`.execute(this.db);

await sql`
  insert into admin_actions (actor_id, target_user_id, action, reason)
  values (${actorId}, ${targetUserId}, ${banned ? "ban" : "unban"}, ${reason})
`.execute(this.db);
```

이 커밋은 **저장 능력**만 만든다. 어떤 caller가 이 operation을 실행할 수 있는지 결정하지 않는다. 두 SQL도 아직 같은 transaction에 묶이지 않는다. UPDATE 성공 뒤 INSERT가 실패할 수 있는 문제는 계정 정지·audit Thread에서 따로 다룬다.

Memory 구현은 target의 `status`만 바꾸며 actor와 reason을 사용하지 않는다. 따라서 이 시점의 backend parity는 status 결과에 한정되고, audit 기록까지 동일하지 않다.

## e8bb6a4bf68b — feat(api): 토너먼트와 관리자 라우트 추가

**중요도** `B` · **태그** `AUTH, PERSISTENCE, TOURNAMENT`

같은 커밋에는 tournament route도 들어 있지만, 이 Thread에서는 관리자 route만 다룬다.

각 route는 request에서 사용자를 해석하고 role을 직접 검사한 뒤 repository를 호출한다.

```ts
app.post("/admin/users/:id/ban", async (request, reply) => {
  const user = await currentUser(repo, request);
  if (!user) return unauthorized(reply);
  if (user.role !== "admin") {
    return reply.code(403).send({ message: "운영자 권한이 필요합니다." });
  }
  const { id } = request.params as { id: string };
  const body = request.body as { banned?: boolean; reason?: string };
  return {
    user: await repo.setUserBan(
      user.id,
      id,
      body.banned ?? true,
      body.reason ?? "manual review"
    )
  };
});
```

`/admin/users`, `/admin/users/:id/ban`, `/admin/users/:id/status`에 같은 패턴이 반복된다. 인증되지 않은 요청은 401, role이 다르면 403이며, 통과한 user id가 audit actor id가 된다.

### 아직 안전하지 않은 이유

route 자체는 role을 server-side에서 검사한다. 그러나 이 시점의 dev upsert는 다음과 같이 handle에서 role을 만든다.

```ts
role: handle === "admin" ? "admin" : "user"
```

따라서 caller가 login payload의 handle을 `admin`으로 선택할 수 있다면 role check를 통과할 수 있다. **검사가 없는 것이 아니라, 검사 대상의 발급 경로가 공격자 입력에 연결된 것**이 핵심 결함이다.

또한 이 route들은 status를 검사하지 않는다. `admin` role을 가진 user가 나중에 banned가 되어도 session 조회 결과에 role이 남아 있으면 계속 통과한다. 이 두 문제는 각각 `45225adcfcd9`, `c577fe2603e3`에서 고쳐진다.

## 1395d45a3665 — test(api): 관리자 사용자 상태 변경 검증

**중요도** `B` · **태그** `AUTH, PERSISTENCE, TEST`

최초의 관리자 API test는 memory repository를 seed한 뒤 handle `admin`으로 login한다.

```ts
const adminLogin = await app.inject({
  method: "POST",
  url: "/auth/dev-login",
  payload: { handle: "admin", displayName: "운영자" }
});
const targetLogin = await app.inject({
  method: "POST",
  url: "/auth/dev-login",
  payload: { handle: "target", displayName: "대상" }
});

const ban = await app.inject({
  method: "POST",
  url: `/admin/users/${targetId}/ban`,
  headers: { authorization: `Bearer ${adminToken}` },
  payload: { banned: true, reason: "smoke" }
});

expect(ban.statusCode).toBe(200);
expect(ban.json<{ user: { status: string } }>().user.status).toBe("banned");
```

이 테스트가 증명하는 것은 login → session lookup → role check → memory status mutation의 happy path다. non-admin 거부나 role 발급의 안전성을 검증하지 않는다. 오히려 별도 provisioning 없이 `admin` handle login만으로 성공하는 fixture가 다음 수정이 제거할 전제를 그대로 보여준다.

## 45225adcfcd9 — feat(db): 명시적 사용자 role 할당 추가

**중요도** `A` · **태그** `AUTH, PERSISTENCE`

### 문제 → 원인 → 결정

- **문제**: 사용자가 `admin`이라는 handle을 선택하는 것만으로 관리자 role을 얻을 수 있었다.
- **원인**: 일반 identity upsert가 계정 정보와 privilege를 동시에 결정했다.
- **결정**: 일반 upsert는 항상 `user` role을 저장하고, non-NPC user의 role 변경은 별도 repository operation과 CLI에서만 수행한다. development seed의 관리자 승격도 일반 login이 아니라 seed provisioning 단계로 분리한다.

### 일반 login은 privilege를 만들지 않는다

PostgreSQL의 insert와 conflict update가 모두 `user`로 고정된다.

```diff
-values (..., ${handle === "admin" ? "admin" : "user"}, false)
+values (..., 'user', false)
 on conflict (handle) do update set
   email = excluded.email,
   display_name = excluded.display_name,
+  role = 'user',
   is_npc = false
```

Memory backend도 새 row의 role을 `user`로 만들고, 기존 row를 upsert할 때 다시 `user`로 설정한다.

```ts
const user = existing ?? {
  /* ... */
  role: "user",
  status: "active",
  rating: 1200,
  is_npc: false
};
user.role = "user";
user.is_npc = false;
```

이 구현은 “일반 dev login은 role을 보존한다”가 아니라 “일반 dev login은 role을 user로 reset한다”는 강한 정책이다. 이미 승격된 계정이 다시 `upsertDevUser`를 거치면 demotion될 수 있다. 이 trade-off를 role 보존 정책으로 오해하면 안 된다.

### role 변경은 별도 operation이다

```ts
async setUserRoleByHandle(
  handle: string,
  role: UserRole
): Promise<PublicUser> {
  const result = await sql<UserRow>`
    update users
    set role = ${role}
    where handle = ${normalizeHandle(handle)}
      and is_npc = false
    returning *
  `.execute(this.db);
  if (!result.rows[0]) throw new Error("user not found");
  return toPublicUser(result.rows[0]);
}
```

NPC는 조건에서 제외된다. handle은 repository에서 normalize되며 대상이 없으면 실패한다. Memory 구현도 같은 조건으로 user를 찾는다.

CLI는 `<handle> <user|admin>` 두 인자를 검증하고 repository를 열어 operation을 호출한 뒤 `finally`에서 닫는다.

```ts
if (!handle || (role !== "user" && role !== "admin")) {
  throw new Error(
    "Usage: pnpm --filter @pong-pong/db user:set-role -- <handle> <user|admin>"
  );
}
const user = await repo.setUserRoleByHandle(handle, role);
```

### development seed는 명시적으로 승격한다

seed가 일반 upsert를 모두 끝낸 뒤 `admin` row만 별도로 변경한다.

```ts
await sql`
  update users
  set role = 'admin', rating = 1680
  where handle = 'admin'
`.execute(this.db);
```

Memory seed도 `admin` row를 찾아 role/rating을 따로 설정한다. 기존 API test는 login 뒤 `repo.setUserRoleByHandle("admin", "admin")`을 호출하도록 바뀐다. 테스트가 성공하려면 이제 fixture가 privilege assignment를 명시해야 한다.

### 보장과 비보장

- 보장: 사용자 입력 handle만으로 관리자 role이 생기지 않는다.
- 보장: role 변경은 일반 login과 분리된 호출을 거쳐야 한다.
- 비보장: CLI를 누가 실행할 수 있는지, role 변경 actor와 reason을 어떻게 audit하는지, 승격 승인 절차는 구현하지 않는다.
- 비보장: ordinary upsert가 기존 admin role을 유지하지 않는다. 실제 코드는 role을 `user`로 reset한다.

## dae31d4a223c — test(auth): 명시적 role assignment 검증

**중요도** `B` · **태그** `AUTH, PERSISTENCE, TEST`

PostgreSQL integration test는 공격 가능한 handle 문자열과 stored role을 분리해 확인한다.

```ts
await repository.ensureSeedData("development");

const loginUser = await repository.upsertDevUser({
  handle: "admin",
  displayName: "일반 사용자"
});
expect(loginUser.role).toBe("user");

const promoted = await repository.setUserRoleByHandle("admin", "admin");
expect(promoted.role).toBe("admin");
```

seed가 한때 `admin`을 승격했더라도 ordinary upsert 결과가 다시 `user`라는 점, explicit operation 이후에만 `admin`이라는 점을 실제 PostgreSQL query 경로에서 증명한다.

이 테스트는 HTTP route나 CLI parser를 통과하지 않는다. role-change audit, OS-level CLI 접근 제어, memory backend parity도 검증 범위가 아니다.

## c577fe2603e3 — fix(auth): 정지된 관리자 login 거부

**중요도** `A` · **태그** `AUTH, RISK`

> commit subject에는 `login 거부`라고 적혀 있지만 실제 diff가 수정하는 곳은 login route가 아니라 `requireAdmin`이다. 이 커밋은 session 발급을 막는 것이 아니라, 정지된 계정의 **후속 관리자 request**를 거부한다.

### 기존 결함

명시적 role provisioning을 도입해도 `requireAdmin`이 role만 보면 충분하지 않다. ban 전에 발급된 session은 그대로 남을 수 있고, session lookup은 token에 저장된 role snapshot이 아니라 현재 users row를 읽는다. 따라서 현재 status도 authorization decision에 포함해야 한다.

수정은 한 줄이다.

```diff
 async function requireAdmin(
   repo: AppRepository,
   request: FastifyRequest
 ): Promise<SessionUser> {
   const user = await currentUser(repo, request);
   if (!user) unauthorized();
+  if (!isActive(user)) suspended();
   if (user.role !== "admin") forbidden();
   return user;
 }
```

검사 순서는 의미가 있다.

| 현재 상태 | 결과 |
| --- | --- |
| 유효한 session user 없음 | 401 `unauthorized` |
| user는 있으나 `status !== active` | 403 `account_suspended` |
| active지만 role이 admin 아님 | 403 `forbidden` |
| active + admin | handler 실행 |

이제 administrator capability는 stored role 하나가 아니라 **현재 active status와 admin role의 교집합**이다. session row를 삭제하지 않아도 다음 admin HTTP request에서 권한을 잃는다.

이 fix는 이미 열린 WebSocket을 닫지 않고, session 자체를 지우지도 않는다. “다음 request에서 거부”와 “현재 사용 중인 realtime authority를 즉시 회수”는 다른 문제다.

## aa037c5291fe — test(auth): 정지된 관리자 session 거부 검증

**중요도** `B` · **태그** `AUTH, TEST`

테스트는 ban 전에 만든 `adminCookie`를 그대로 유지한다. 새로 login해 실패하는지 보는 것이 아니라 **이미 발급된 credential의 권한 재평가**를 확인한다.

```ts
const admin = await repo.getUserByHandle("admin");
if (!admin) throw new Error("seed:dev admin was not created");

await repo.setUserBan(
  admin.id,
  admin.id,
  true,
  "운영자 계정 정지 검사"
);

const actions = await app.inject({
  method: "GET",
  url: "/admin/actions",
  headers: { cookie: adminCookie }
});

expect(actions.statusCode).toBe(403);
expect(actions.json()).toEqual({
  error: expect.objectContaining({
    code: "account_suspended",
    message: expect.any(String),
    requestId: expect.any(String)
  })
});
```

같은 cookie가 여전히 session user를 찾을 수 있어도 `requireAdmin`이 현재 status에서 차단한다. typed error envelope까지 확인하므로 단순 403뿐 아니라 suspension으로 분류되는지도 고정한다.

Memory repository와 Fastify injection을 사용하므로 PostgreSQL query, session row 삭제, 실제 network/WebSocket은 증명하지 않는다.

## 최종 상태

```text
[ordinary identity creation]
  upsertDevUser(handle, ...)
    → role = user
    → handle 문자열은 privilege source가 아님

[provisioning]
  validated CLI / repository operation
    → normalize handle
    → non-NPC user만 role 변경

[administrator request]
  cookie session 해석
    → no user: unauthorized
    → inactive: account_suspended
    → role != admin: forbidden
    → admin repository operation
```

| 관심사 | 최종 owner |
| --- | --- |
| identity 생성 | `upsertDevUser`; privilege 없이 `user`로 생성·갱신 |
| role 상태 | user row / memory user row |
| role 변경 | `setUserRoleByHandle`와 이를 호출하는 provisioning CLI |
| 요청별 관리자 capability | API `requireAdmin`; 현재 status와 role을 함께 판정 |
| 기존 session | identity를 찾는 credential일 뿐, 영구적인 admin capability snapshot이 아님 |

## 이 Thread의 경계

이 Thread는 **role의 발급 출처와 관리자 HTTP authorization**에 한정된다.

- status 변경과 audit row를 원자적으로 저장하는 문제는 다음 계정 정지 Thread가 다룬다.
- ban 직후 이미 열린 WebSocket, queue, tournament waiter, input ownership을 회수하는 문제도 다음 Thread의 범위다.
- role 변경 자체의 actor/reason audit, 다중 승인, CLI 호출자의 OS 권한, role expiration은 구현하지 않는다.
- 일반 사용자 route 전반의 suspension policy를 중앙화하는 문제는 이 Thread의 `requireAdmin`보다 넓다.
- 관리자 UI가 role을 부여하거나 변경하는 workflow는 포함되지 않는다.
