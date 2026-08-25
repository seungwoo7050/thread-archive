# Thread: 계정 정지 — audit 원자성과 열린 WebSocket 권한 회수

Project: `ft_transcendence` · Branch: `web/ft_transcendence` · Category: `03-identity-authorization-and-account-lifecycle`

## 개요

account `status`를 database에 저장하는 것만으로는 정지가 완성되지 않는다. 정지된 사용자가 새 작업을 시작하지 못하게 해야 하고, status 변경에는 감사 기록이 반드시 따라야 하며, 이미 열려 있는 realtime connection도 회수해야 한다.

이 Thread는 세 단계의 보장을 연결한다.

1. **새 작업 차단** — 일부 보호된 HTTP mutation과 새 WebSocket admission이 현재 user status를 검사한다.
2. **durable state의 원자성** — PostgreSQL의 user status 변경과 `admin_actions` insert가 함께 commit되거나 함께 rollback된다.
3. **이미 부여된 realtime authority 회수** — ban이 성공하면 `GameHub`가 현재 authoritative client의 timer, buffer, queue/waiter, index, input ownership을 정리하고 socket을 닫는다.

초기 구현은 1번의 일부만 해결했다. 새 handshake는 막았지만 이미 연결된 socket은 살아 있었고, PostgreSQL의 status UPDATE와 audit INSERT는 별도 statement라 감사 없는 ban이 남을 수 있었다. 후속 fix와 test가 이 두 간극을 각각 닫는다.

### 최종 상태

| 정지 이후 경계 | 최종 동작 | 근거 |
| --- | --- | --- |
| 지정된 HTTP mutation | current user의 status가 active가 아니면 403 | `42033a6f2f3a`, `e07726592df5` |
| 새 WebSocket admission | inactive user는 hub에 연결되기 전에 거부 | `42033a6f2f3a` |
| PostgreSQL status + audit | 동일 transaction에서 commit/rollback | `d0137660cd9f`, `9106abc10d0e` |
| 이미 열린 current WebSocket | ban commit 뒤 hub ownership 정리, close 4003 | `40e5c520d49c`, `454cbf2c95e0` |
| 같은 기존 session의 새 WS ticket | current banned status로 403 | `454cbf2c95e0` |

> 조사 범위: exact SHA의 diff와 그 SHA 시점 source/test만 사용했다. repository에 정의된 테스트는 실행하지 않았으며, 아래 “증명”은 test code가 겨냥하는 assertion 범위를 뜻한다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `42033a6f2f3a` | feat(admin): 감사 가능한 사용자 상태 API 추가 | A | `AUTH, REALTIME, PERSISTENCE` | suspension을 일부 HTTP/WS 경계에서 실제 authorization state로 사용한다 |
| 2 | `bf797871007c` | feat(admin): 감사 기록과 상태 변경 UI 추가 | B | `AUTH, WEB` | reason-bearing status mutation과 audit read를 관리자 화면에 연결한다 |
| 3 | `e07726592df5` | test(app): 전체 서비스 흐름 검증 | B | `AUTH, REALTIME, PERSISTENCE` | ban→audit 조회→기존 token의 HTTP write 거부를 연결한다 |
| 4 | `d0137660cd9f` | fix(db): 차단 감사 기록을 원자적으로 저장 | A | `AUTH, PERSISTENCE, RISK` | status UPDATE와 audit INSERT를 같은 PostgreSQL transaction으로 옮긴다 |
| 5 | `9106abc10d0e` | test(db): 차단 감사 기록 atomicity 검증 | A | `AUTH, PERSISTENCE, TEST` | 두 번째 write만 실패시켜 첫 번째 UPDATE rollback을 검증한다 |
| 6 | `40e5c520d49c` | fix(auth): 정지된 사용자의 열린 연결 폐기 | A | `AUTH, REALTIME, TOURNAMENT` | ban 뒤 current GameHub client와 관련 runtime ownership을 회수한다 |
| 7 | `454cbf2c95e0` | test(auth): 계정 정지의 기존 WebSocket 차단 검증 | A | `AUTH, REALTIME, TEST` | real socket close와 같은 session의 새 ticket 거부를 검증한다 |

## 42033a6f2f3a — feat(admin): 감사 가능한 사용자 상태 API 추가

**중요도** `A` · **태그** `AUTH, REALTIME, PERSISTENCE`

이 커밋은 account status를 단순 표시값에서 실제 admission 조건으로 바꾸고, administrator action을 읽을 수 있는 API를 추가한다.

### WebSocket: hub 연결 전에 status 검사

```ts
currentUser(repo, request)
  .then((user) => {
    if (!user) {
      socket.close(1008, "unauthorized");
      return;
    }
    if (user.status !== "active") {
      socket.close(1008, "account suspended");
      return;
    }
    socket.off("message", bufferPayload);
    hub.connect(socket as WebSocket, request.raw, user, pendingPayloads);
  })
  .catch(() => socket.close(1011, "authentication failed"));
```

inactive user는 `GameHub.connect`에 들어가지 않으므로 client map, heartbeat, queue 같은 realtime ownership을 새로 얻지 못한다. 다만 이 검사는 **handshake 시점에 한 번** 수행된다. 이미 `hub.connect`를 통과한 socket은 status가 바뀌어도 이 branch를 다시 지나지 않는다.

### HTTP: 실제로 추가된 active guard

이 exact SHA에서 `isActive` 검사가 추가된 mutation은 다음과 같다.

- `POST /chat/lobby`
- `POST /friends/request`
- `POST /friends`
- `POST /tournaments`
- `POST /tournaments/:id/join`

패턴은 동일하다.

```ts
const user = await currentUser(repo, request);
if (!user) return unauthorized(reply);
if (!isActive(user)) return suspended(reply);
```

이 목록보다 넓게 “모든 보호 route가 정지를 검사한다”고 일반화하면 안 된다. 같은 source에서 `POST /friends/:id/accept`, profile update, admin routes 등에는 이 active guard가 없다. 관리자 route의 status 검사는 별도 Thread의 `c577fe2603e3`에서 추가된다.

### audit read model

`GET /admin/actions`는 session user를 읽고 `role === "admin"`을 확인한 뒤 repository의 `listAdminActions`를 호출한다. PostgreSQL 구현은 최근 action row를 읽고 actor/target user를 조회해 `AdminActionSummary`로 변환한다. Memory 구현은 local `adminActions` 배열을 반환하고 `setUserBan`에서 새 action을 앞에 넣는다.

이로써 운영자는 ban/unban의 actor, target, action, reason, created time을 읽을 수 있다.

### 이 커밋이 남긴 두 핵심 결함

#### 1. PostgreSQL partial state

`setUserBan`은 여전히 두 statement를 base database handle에 따로 실행한다.

```ts
const result = await sql<UserRow>`update users ... returning *`
  .execute(this.db);
await sql`insert into admin_actions ...`
  .execute(this.db);
```

첫 UPDATE가 commit된 뒤 audit INSERT가 실패하면 user는 banned인데 대응 action은 없는 상태가 남는다.

#### 2. 이미 열린 WebSocket

handshake guard는 새 connection만 막는다. ban 전에 이미 `GameHub`에 등록된 client는 status를 재조회하지 않으므로 socket을 계속 사용할 수 있다.

즉 이 커밋은 status를 authorization state로 도입하지만, durable atomicity와 live revocation까지 완성하지는 않는다.

## bf797871007c — feat(admin): 감사 기록과 상태 변경 UI 추가

**중요도** `B` · **태그** `AUTH, WEB`

관리자 화면은 사용자 목록만 보여주던 상태에서 조치 사유와 audit 목록을 함께 다루게 된다.

```diff
 const [users, setUsers] = useState<PublicUser[]>([]);
+const [actions, setActions] = useState<AdminActionSummary[]>([]);
+const [reason, setReason] = useState("운영자 검토");
```

초기 load는 두 API를 함께 요청한다.

```ts
Promise.all([
  apiFetch<{ users: PublicUser[] }>("/admin/users"),
  getAdminActions()
]).then(([result, actionItems]) => {
  setUsers(result.users);
  setActions(actionItems);
});
```

status 변경에서는 사용자가 입력한 reason을 server에 보내고, 성공한 user row를 local list에 반영한 뒤 audit 목록을 다시 조회한다.

```ts
const updated = await setUserStatus(
  user.id,
  user.status === "active" ? "banned" : "active",
  reason.trim() || "운영자 검토"
);
setUsers((current) =>
  current.map((item) => item.id === updated.id ? updated : item)
);
setActions(await getAdminActions());
```

이 UI state는 authoritative state가 아니다. status와 audit의 일관성은 server repository가 보장해야 한다. mutation은 성공했지만 후속 audit refresh가 실패하면 browser message/list가 실제 server state와 어긋날 수 있으며, 이 커밋에는 rollback이나 request cancellation이 없다.

## e07726592df5 — test(app): 전체 서비스 흐름 검증

**중요도** `B` · **태그** `AUTH, REALTIME, PERSISTENCE`

이 커밋에는 tournament, browser E2E 등 많은 변경이 함께 들어 있다. 이 Thread의 근거는 `apps/api/src/admin.test.ts`에 추가된 두 assertion뿐이다.

```ts
const actions = await app.inject({
  method: "GET",
  url: "/admin/actions",
  headers: { authorization: `Bearer ${adminToken}` }
});
const blockedChat = await app.inject({
  method: "POST",
  url: "/chat/lobby",
  headers: {
    authorization: `Bearer ${targetLogin.json<{ token: string }>().token}`
  },
  payload: { body: "정지 후 채팅" }
});

expect(actions.json<{ actions: Array<{ reason: string }> }>()
  .actions[0].reason).toBe("smoke");
expect(blockedChat.statusCode).toBe(403);
```

### 무엇을 검증하는가

ban 전에 target에게 발급된 bearer token을 ban 뒤 그대로 사용한다. 그 credential이 아직 session user를 찾더라도 `/chat/lobby`의 current status guard가 403을 반환해야 한다. 동시에 admin action 조회에서 request reason `smoke`가 보이는지 확인한다.

### 무엇을 검증하지 않는가

- memory repository를 사용하므로 PostgreSQL의 두-statement partial failure를 재현하지 않는다.
- HTTP lobby write만 확인하며 다른 route의 status guard 전체를 증명하지 않는다.
- WebSocket handshake나 이미 열린 socket은 다루지 않는다.
- broad commit의 tournament/E2E 변경은 suspension invariant의 증거가 아니다.

## d0137660cd9f — fix(db): 차단 감사 기록을 원자적으로 저장

**중요도** `A` · **태그** `AUTH, PERSISTENCE, RISK`

### 문제 → 원인 → 결정

- **문제**: audit INSERT가 실패해도 먼저 실행된 users UPDATE는 database에 남을 수 있었다.
- **원인**: 논리적으로 하나인 ban operation의 두 SQL이 서로 다른 autocommit statement로 실행됐다.
- **결정**: `setUserBan` 전체를 Kysely transaction callback으로 감싸고 UPDATE와 INSERT 모두 같은 `transaction` handle을 사용한다.

```diff
-const result = await sql<UserRow>`update users ... returning *`
-  .execute(this.db);
-await sql`insert into admin_actions ...`.execute(this.db);
-return toPublicUser(firstRow(result));
+return this.db.transaction().execute(async (transaction) => {
+  const result = await sql<UserRow>`update users ... returning *`
+    .execute(transaction);
+  await sql`insert into admin_actions ...`
+    .execute(transaction);
+  return toPublicUser(firstRow(result));
+});
```

### 수정된 불변 조건

PostgreSQL에서 한 `setUserBan` call의 결과는 둘 중 하나다.

| transaction 결과 | users row | admin_actions row |
| --- | --- | --- |
| callback 성공 | 새 status/banned_at commit | 대응 action commit |
| UPDATE·INSERT·mapping 중 throw | 이전 status 유지 | 새 action 없음 |

단순히 `await` 순서를 유지하는 것과 transaction은 다르다. 동일 handle을 두 statement에 전달해야 두 write가 하나의 commit decision을 공유한다.

이 fix는 PostgreSQL 구현에만 적용된다. Memory backend의 status mutation과 action array update는 database rollback 모델을 갖지 않는다. 또한 transaction이 끝난 뒤 수행되는 `GameHub` side effect까지 묶지는 않는다.

## 9106abc10d0e — test(db): 차단 감사 기록 atomicity 검증

**중요도** `A` · **태그** `AUTH, PERSISTENCE, TEST`

happy path만으로는 두 SQL 중 하나가 실수로 `this.db`를 사용해 transaction 밖으로 빠지는 회귀를 잡기 어렵다. 이 테스트는 두 번째 statement만 결정적으로 실패시킨다.

### 실패 주입

격리 PostgreSQL database의 `admin_actions`에 특정 reason을 거부하는 CHECK constraint를 추가한다.

```sql
alter table admin_actions
add constraint admin_actions_reject_test_reason
check (reason <> 'force audit failure')
```

그 reason으로 `setUserBan`을 호출하면 users UPDATE는 먼저 실행될 수 있지만, audit INSERT에서 constraint violation이 발생한다.

```ts
await expect(repository.setUserBan(
  actor.id,
  target.id,
  true,
  "force audit failure"
)).rejects.toMatchObject({
  constraint: "admin_actions_reject_test_reason"
});
```

실패 뒤 application response만 보지 않고 raw SQL로 두 table을 직접 읽는다.

```ts
expect(storedUser.rows).toEqual([
  { status: "active", banned_at: null }
]);

await expect(pool.query(
  "select count(*)::integer as count from admin_actions"
)).resolves.toMatchObject({ rows: [{ count: 0 }] });
```

### 이 test가 주는 증거

실제 PostgreSQL에서 첫 write 이후 두 번째 write가 실패해도 user row가 rollback되고 audit row도 남지 않는다. transaction handle 누락을 겨냥하는 failure-injection integration test다.

process crash의 모든 timing, network disconnect, memory backend, DB commit 뒤 realtime revocation은 증명하지 않는다. 임시 constraint와 row의 정리는 isolated database fixture 수명에 의존한다.

## 40e5c520d49c — fix(auth): 정지된 사용자의 열린 연결 폐기

**중요도** `A` · **태그** `AUTH, REALTIME, TOURNAMENT`

### 기존 가정과 실제 간극

새 HTTP request와 새 WebSocket admission에서 current status를 검사하면 추가 작업은 막을 수 있다. 그러나 ban 전에 이미 handshake를 끝내고 `GameHub.clientsByUser`에 등록된 socket은 그 검사를 다시 수행하지 않는다. 저장된 status와 현재 realtime authority가 분리돼 있었다.

### ban commit 뒤 hub에 알린다

두 administrator status route는 repository mutation을 먼저 기다린다. 성공한 결과가 ban이면 `hub.revokeUser`를 호출한다.

```ts
const banned = body.banned ?? true;
const target = await repo.setUserBan(
  user.id,
  id,
  banned,
  body.reason ?? "manual review"
);
if (banned) hub.revokeUser(id);
return { user: target };
```

순서는 **durable status/audit commit → in-memory revocation**이다. repository call이 실패하면 revoke하지 않으며, unban에서도 revoke하지 않는다.

### `GameHub.revokeUser`가 회수하는 것

```ts
revokeUser(userId: string): void {
  const client = this.clientsByUser.get(userId);
  if (!client) return;

  client.heartbeat.stop();
  client.snapshots.close();
  this.leaveQueue(client);
  this.leaveTournamentWaiters(client);
  this.clients.delete(client.id);
  this.clientsByUser.delete(userId);
  this.inputGate.releaseUser(userId);

  if (client.roomId) {
    const room = this.rooms.get(client.roomId);
    const side = room ? sideFor(room, client) : null;
    if (room && side) this.reserveRoomSide(room, side, userId);
  }

  if (client.socket.readyState === WebSocket.OPEN) {
    client.socket.close(4003, "account suspended");
  }
  this.broadcastPresence();
}
```

`clientsByUser`가 가리키는 current authoritative client 하나를 대상으로 다음 ownership을 정리한다.

| 순서 | 회수 대상 | 이유 |
| ---: | --- | --- |
| 1 | heartbeat | 더 이상 ping/timeout task를 유지하지 않는다 |
| 2 | latest snapshot buffer | pending snapshot delivery와 socket write ownership을 닫는다 |
| 3 | matchmaking queue | 정지된 사용자가 match 후보로 남지 않게 한다 |
| 4 | tournament waiter | tournament 대기 ownership을 제거한다 |
| 5 | `clients`, `clientsByUser` indexes | 이후 `receive`가 authoritative client로 인정하지 않게 한다 |
| 6 | input gate | user 단위 입력 제한/예약을 해제한다 |
| 7 | room side handoff | room에 있었다면 기존 reconnect/forfeit lifecycle이 처리할 수 있게 side를 예약한다 |
| 8 | socket | OPEN이면 application close code 4003으로 닫는다 |
| 9 | presence | online projection을 다시 broadcast한다 |

map 삭제가 socket close보다 먼저 일어나므로 close frame 처리 전에도 이후 message는 current client check를 통과하지 못한다.

### 남는 failure window

PostgreSQL transaction과 `GameHub` memory는 하나의 transaction manager를 공유하지 않는다. DB commit 직후 process가 죽거나 `revokeUser` 호출 전에 예외적인 중단이 생기면 durable ban은 남지만 그 process의 기존 connection 정리가 완료되지 않을 수 있다. multi-instance 환경에서는 다른 instance의 `clientsByUser`도 이 local call만으로는 알 수 없다.

이 커밋은 정상 단일-process handler 경로에서의 즉시 회수를 보장하지만, distributed atomicity나 보상 worker를 만들지는 않는다.

## 454cbf2c95e0 — test(auth): 계정 정지의 기존 WebSocket 차단 검증

**중요도** `A` · **태그** `AUTH, REALTIME, TEST`

method-level mock만으로는 실제 WebSocket handshake, hub index, close event가 이어지는지 알 수 없다. 이 테스트는 Fastify를 loopback의 임의 port에 실제로 listen하고 `ws` client를 연결한다.

### 재현 순서

```text
target dev-login
  → target cookie로 /auth/ws-ticket 발급
  → ws://127.0.0.1:<ephemeral>/ws?ticket=...&v=1 연결
  → open event 확인
  → close listener 먼저 등록
  → admin cookie로 target ban
  → close code/reason 확인
  → 같은 target cookie로 새 ticket 요청
```

핵심 assertion은 두 개다.

```ts
expect(ban.statusCode).toBe(200);
await expect(closed).resolves.toEqual({
  code: 4003,
  reason: "account suspended"
});

const newTicket = await app.inject({
  method: "POST",
  url: "/auth/ws-ticket",
  headers: { cookie: sessionCookie(targetLogin) }
});
expect(newTicket.statusCode).toBe(403);
```

첫 assertion은 이미 열린 control channel의 live revocation을, 두 번째는 같은 기존 session이 새 channel을 다시 얻지 못하는 admission control을 검증한다. 둘 중 하나만 확인하면 권한 철회가 불완전할 수 있다.

suite는 생성한 socket을 배열로 소유하고 `afterEach`에서 CLOSED가 아니면 `terminate`한 뒤 app/repository를 닫는다. test 자체가 connection leak을 남기지 않도록 cleanup owner가 명시돼 있다.

### 증명 범위

- 증명: 한 process의 memory repository + 실제 HTTP server/WebSocket transport 조합에서 ban이 current socket을 4003으로 닫고 새 ticket을 403으로 막는다.
- 비증명: PostgreSQL transaction, multi-instance broadcast, close frame의 원격 전달 보장, `GameHub` 내부 queue/room/input field 각각의 최종 값을 직접 검사하지 않는다.

## 최종 execution flow

```text
[admin ban request]
  requireAdmin(current session user)
    → repository.setUserBan(actor, target, banned=true, reason)
      → PostgreSQL transaction
         users UPDATE
         admin_actions INSERT
         commit
    → GameHub.revokeUser(target)
       heartbeat stop
       snapshot buffer close
       queue / tournament waiter leave
       client indexes delete
       input ownership release
       room-side recovery handoff
       socket close 4003
       presence broadcast

[subsequent work]
  protected HTTP mutation → current status 확인 → 403
  new WS ticket/admission → current status 확인 → 403/close
```

## 이 Thread의 경계

이 Thread는 **account suspension을 durable audit state와 현재 realtime authority에 반영하는 문제**만 다룬다.

- explicit admin role의 발급과 정지된 administrator의 HTTP authorization은 이전 Thread가 다룬다.
- 이 exact SHA에서 active guard가 없는 모든 route까지 정지 정책이 자동으로 확장된다고 보지 않는다.
- role 변경 audit, ban approval workflow, appeal/unban 후 resource 복구는 포함되지 않는다.
- DB commit과 GameHub revocation 사이의 distributed transaction, outbox, 재시도·보상 worker는 구현하지 않는다.
- 다른 process/instance에 연결된 socket을 pub/sub로 찾아 닫는 multi-instance revocation도 범위 밖이다.
- socket close frame이 network 단절 상황에서도 client에 전달된다는 보장은 없으며, server-side authority 제거가 핵심 보장이다.
