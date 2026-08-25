# Guest mode를 격리된 임시 trust domain으로 만들기

게스트를 등록 사용자와 같은 repository user로 만들고 일부 버튼만 숨기면, 전적·rating·chat·친구·대회·WebSocket ticket 저장소까지 영속 경계가 섞입니다. 반대로 아무 인증 없이 anonymous socket을 열면 연결·ticket·세션 생성이 무제한 자원이 됩니다.

이 Thread는 guest를 “제한된 일반 계정”이 아니라 별도 trust domain으로 정의합니다.

- HMAC으로 위변조를 막고 IP·2시간 TTL에 묶인 cookie session
- PostgreSQL을 사용하지 않는 memory-only one-time WebSocket ticket
- per-IP와 process 전체에 걸친 session·ticket·connection 상한
- registered/guest가 서로 섞이지 않는 matchmaking과 room
- DB match·rating을 만들지 않는 임시 결과, 2분간의 memory recovery

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `cacd4c22d705` | `feat(guest): signed guest session token 정의` | A | `AUTH, PERSISTENCE` | HMAC·IP·2시간 TTL을 가진 self-contained guest cookie 정의 |
| `17a1dd501b1b` | `feat(guest): guest resource lease 수명주기 추가` | A | `AUTH, PROTOCOL, REALTIME` | 생성·ticket·connection에 메모리 상한과 lease ownership 추가 |
| `a5c06c561e00` | `feat(guest): guest session과 WebSocket 인증 연결` | A | `AUTH, REALTIME, PERSISTENCE` | demo route, memory ticket, connection lease, registered guard 연결 |
| `27ddc3fca2f1` | `feat(guest): guest 조회 범위와 lobby 격리` | A | `AUTH, PERSISTENCE, TOURNAMENT` | 전적·chat·profile·leaderboard·tournament 읽기/쓰기 격리 |
| `77a7c205ccd0` | `feat(game): guest matchmaking과 room을 격리` | A | `REALTIME, PERSISTENCE, RISK` | guest끼리만 매칭하고 persistent NPC lookup을 사용하지 않음 |
| `eaa4fdaba361` | `feat(game): guest 경기 결과 영속화 차단과 임시 보존` | A | `SIMULATION, REALTIME` | DB finalization을 건너뛰고 결과를 2분간 memory에 보존 |
| `2b274686e6d4` | `fix(guest): 체험 환경의 runtime 복구 제한` | A | `AUTH, REALTIME, RISK` | APP_MODE 검증, IP 추적 상한, ticket IP 귀속·rate/capacity 제한 |
| `06d2eb7a93cc` | `test(guest): 체험 환경의 복구 경계 검증` | A | `AUTH, SIMULATION, REALTIME` | 만료 정리·capacity·새 ticket reconnect·중복 queue 방지를 검증 |

## 1. DB row 없이도 위변조를 검출하는 guest session

### `cacd4c22d705`

`GuestAccess`는 최소 32-byte secret을 요구하고 guest payload를 HMAC-SHA-256으로 서명합니다.

```ts
type GuestPayload = {
  v: 1;
  user: GuestSessionUser;
  ip: string;
  expiresAtMs: number;
};

cookieValue = `${base64url(JSON.stringify(payload))}.${hmac(payload)}`;
```

guest user는 random UUID, random handle suffix, 기본 rating 1,200을 가지며 `sessionKind: "guest"`로 표시됩니다. TTL은 2시간입니다.

인증은 다음 순서로 진행됩니다.

1. 마지막 `.`을 기준으로 payload와 signature를 나눕니다.
2. 동일 payload의 HMAC을 다시 계산합니다.
3. 길이가 같을 때 `timingSafeEqual`로 비교합니다.
4. JSON을 읽고 `v`, `sessionKind`, role/status, expiry를 검사합니다.
5. expected IP가 주어졌다면 payload IP와 같은지 확인합니다.

이 token은 **서명된 것**이지 암호화된 것이 아닙니다. browser는 base64url payload를 해석할 수 있지만, user ID·rating·expiry·IP를 바꿔도 signature를 만들 수 없으므로 서버가 거부합니다. DB session lookup 없이 identity를 복원하지만, 권한 범위는 이후 route guard와 room policy가 별도로 제한합니다.

## 2. self-contained session에도 process 자원 owner가 필요하다

### `17a1dd501b1b`

서명이 stateless하다고 해서 runtime 자원까지 무제한이어도 되는 것은 아닙니다. `GuestAccess`는 다음 기본 상한을 갖습니다.

| 자원 | 기본 상한 |
| --- | ---: |
| IP별 guest 생성 | 1분에 10회 |
| IP별 active guest connection | 4개 |
| process 전체 active guest connection | 200개 |
| process 전체 pending guest ticket | 400개 |

WebSocket ticket은 등록 사용자와 마찬가지로 32-byte raw 값의 hash만 map key로 저장하고 30초 뒤 만료됩니다. 같은 guest가 새 ticket을 발급하면 이전 pending ticket을 지워 한 guest당 유효한 pending ticket을 하나로 유지합니다.

connection은 단순 count 증가/감소가 아니라 lease로 표현됩니다.

```ts
const leaseId = randomUUID();
connections.set(guestId, { ip, leaseId });

return {
  release: () => {
    if (connections.get(guestId)?.leaseId === leaseId) {
      connections.delete(guestId);
    }
  }
};
```

같은 guest가 새 socket으로 교체되면 새 lease ID가 기존 entry를 덮습니다. old socket의 `close`가 늦게 도착해 old lease를 release해도 현재 lease ID와 다르므로 새 connection을 지우지 않습니다. 이는 Thread 05의 active connection replacement와 같은 stale-cleanup 문제를 guest capacity map에서도 해결합니다.

## 3. demo mode에서는 guest credential이 PostgreSQL로 fallback하지 않는다

### `a5c06c561e00`

`POST /auth/guest`는 demo mode에서만 활성화되고 다음 cookie를 설정합니다.

```text
name: pp_guest
HttpOnly: true
Secure: true
SameSite: lax
Max-Age: 2 hours
Path: /
```

`getCurrentUser`는 등록 사용자 session과 guest cookie를 함께 해석하지만, guest WebSocket ticket은 `GuestAccess` memory map에서만 발급·소비합니다.

handshake의 중요한 분기는 다음과 같습니다.

```text
hash(raw ticket)
  → GuestAccess.consumeWsTicket(hash)
      ├─ guest 반환: guest 인증 완료
      └─ null
          ├─ demo mode: 거부
          └─ normal mode: PostgreSQL ticket consume 시도
```

즉 demo mode에서 guest ticket이 memory에 없다고 해서 등록 사용자 DB ticket store를 조회하지 않습니다. trust domain이 섞여 “우연히 같은 hash의 DB ticket”이나 잘못된 fallback으로 인증되는 경로를 차단합니다.

인증 후 guest는 `acquireConnection(request.ip, user.id)` lease를 얻어야 GameHub로 넘어갑니다. 상한을 넘으면 policy violation으로 닫고, 성공한 lease는 socket `close`에서 정확히 한 번 release합니다.

logout은 `pp_session`과 `pp_guest`를 모두 clear하지만, guest에는 DB session 삭제를 호출하지 않습니다. 등록 사용자 전용 기능은 `requireRegistered`가 403 `guest_feature_forbidden`으로 막습니다.

## 4. 데이터 조회와 mutation을 UI가 아니라 API 경계에서 격리했다

### `27ddc3fca2f1`

게스트 lobby는 현재 사용자와 live presence/stat만 볼 수 있고 다음 데이터는 빈 배열입니다.

- recent matches
- lobby chat history

chat 작성과 dashboard는 guest에게 `requireRegistered`를 적용합니다. demo mode의 user lookup, profile, leaderboard, tournament 목록은 “빈 registered dataset”을 흉내 내기보다 route 자체를 제공하지 않는 것으로 처리합니다.

이 결정은 두 효과가 있습니다.

1. guest UI가 실수로 등록 사용자 API를 호출해도 server에서 차단됩니다.
2. demo environment가 production DB의 social/tournament data를 읽지 않는다는 운영 경계가 명확해집니다.

숨겨진 menu나 client-side redirect는 편의 기능일 뿐 보안 경계가 아닙니다. 최종 권위는 route의 app mode와 `sessionKind` 검사입니다.

## 5. guest와 registered player는 같은 queue·room에 섞이지 않는다

### `77a7c205ccd0`

기존 closest-rating search에 session kind 비교가 추가됩니다.

```ts
if (isGuest(candidate.user) !== isGuest(client.user)) continue;
```

room은 생성 시 `guest` flag를 가집니다. left 참가자의 kind가 room kind를 정하고, matching 단계에서 양쪽 kind가 같도록 보장합니다.

6초 AI fallback에서도 guest는 persistent repository의 NPC 목록을 조회하지 않습니다. guest fallback은 내부 AI paddle을 사용하고 `npcUser`를 요구하지 않습니다. 등록 사용자 queue만 실제 NPC identity를 선택합니다.

이 분리는 matchmaking fairness보다 persistence 안전성에 중요합니다. mixed room이 허용되면 한쪽은 rating/history를 저장해야 하고 다른 쪽은 저장하지 않아야 하는 모순된 finalization 정책이 생깁니다.

## 6. guest 결과는 “저장 실패”가 아니라 의도적으로 비영속이다

### `eaa4fdaba361`

room이 guest이면 `repo.finalizeMatch`를 호출하지 않습니다. 결과 envelope은 다음 형태입니다.

```ts
{
  roomId,
  matchId: null,
  persisted: false,
  winnerSide,
  leftScore,
  rightScore,
  ratingDelta: 0
}
```

따라서 DB match, wins/losses, rating history, tournament progress가 생기지 않습니다. 이는 Thread 04 transaction의 실패 branch가 아니라 그 boundary를 아예 호출하지 않는 별도 정책입니다.

다만 socket이 경기 종료 직전에 끊기면 결과를 전혀 보지 못할 수 있습니다. GameHub는 guest user ID별로 결과를 memory에 2분 보존합니다.

```text
recentGuestResults[userId]
  = { result, expiresAtMs, cleanupTimer }
```

같은 guest가 돌아왔는데 active room 복구가 일어나지 않았다면 `game.finished`를 다시 보냅니다. 만료 timer는 `unref`되고, 새 결과가 들어오면 이전 timer를 clear합니다. 2분 뒤 entry는 삭제됩니다.

이 retention은 durable history가 아닙니다. process restart, 다른 replica로의 reconnect, 2분 초과 뒤에는 결과가 사라질 수 있습니다. 그 제한이 바로 guest의 transient contract입니다.

## 7. runtime 제한 자체가 무한히 커지는 문제를 닫았다

### `2b274686e6d4`

기존 per-IP rate map은 IP마다 entry를 계속 만들 수 있고, ticket total limit만으로는 한 IP가 많은 guest를 만들어 pending slot을 독점할 수 있습니다. fix는 다음 제한을 추가합니다.

| 자원 | 기본 상한/정책 |
| --- | --- |
| 추적하는 IP window 수 | 10,000 |
| IP별 pending guest ticket | 4개 |
| IP별 ticket 발급 | 1분에 30회 |
| creation/ticket window | 60초 뒤 timer와 opportunistic prune로 제거 |
| guest ticket | 발급 IP를 함께 저장 |

`issueWsTicket`은 이제 guest뿐 아니라 request IP를 받습니다. 발급 횟수를 먼저 기록하고, 같은 guest의 이전 ticket을 제거한 뒤 IP별 pending 수와 process 전체 수를 검사합니다. ticket entry도 IP를 보유합니다. 이 IP는 발급량과 pending capacity를 계산하기 위한 값입니다. 이 커밋의 `consumeWsTicket`은 hash만 받으므로, ticket을 발급한 IP와 소비 시 IP가 반드시 같아야 한다는 인증 조건까지 추가된 것은 아닙니다.

rolling window map은 새 IP를 추가하기 전에 만료 entry를 prune합니다. limit에 도달했는데 기존 key가 아니면 capacity error를 반환합니다. 각 entry의 cleanup timer에는 generation 역할을 하는 `expiresAtMs`가 있어, 과거 timer가 갱신된 현재 window를 삭제하지 않습니다.

환경 해석도 함께 수정됩니다. `APP_MODE`가 명시되면 development/test/production/demo 중 하나여야 하며, `staging` 같은 알 수 없는 값은 development로 조용히 떨어지지 않고 startup error가 됩니다. production deployment가 잘못된 mode로 실행되어 dev-login이나 비보안 cookie 정책을 얻는 위험을 줄입니다.

## 8. recovery test는 “새 ticket, 같은 room, queue 재전송 없음”을 고정한다

### `06d2eb7a93cc`

테스트 범위는 인증 성공만이 아닙니다.

- 명시적 `APP_MODE=production/test`가 `NODE_ENV` 없이도 존중됩니다.
- 잘못된 `APP_MODE`는 startup에서 거부됩니다.
- 생성 IP window가 60초 뒤 제거되고 tracked-IP capacity가 회복됩니다.
- pending ticket이 30초 뒤 timer로 제거됩니다.
- IP별 pending ticket과 1분 ticket 발급 상한이 적용됩니다.
- guest connection lease의 IP별·전체 상한과 stale lease release가 검증됩니다.
- browser reconnect는 ticket provider를 다시 호출해 fresh ticket을 얻습니다.
- reconnect socket이 열려도 최초 `queue.join`을 다시 보내지 않습니다.
- `roomId`가 있는 reconnecting 상태에서는 새 match를 시작할 수 없습니다.
- recovered transient result는 전적에 저장되지 않았다는 문구로 표시됩니다.

이 테스트들은 fake timer와 fake socket을 사용합니다. 실제 multi-process replica 사이에 guest memory state가 공유된다는 보장은 없으며, 오히려 설계상 공유되지 않습니다.

## 최종 trust-domain 표

| 항목 | registered | guest/demo |
| --- | --- | --- |
| 장기 identity | DB session + HttpOnly cookie | HMAC self-contained cookie, IP/2h TTL |
| WebSocket ticket | PostgreSQL hash, 30초 단발 | process memory hash, 발급 IP별 capacity accounting, 30초 단발 |
| connection capacity | 일반 GameHub 정책 | IP 4 / process 200 lease |
| matchmaking | registered끼리 | guest끼리 |
| AI fallback identity | persistent NPC 가능 | internal AI, DB NPC 조회 없음 |
| match/rating history | atomic DB finalization | 생성하지 않음 |
| 종료 결과 복구 | persisted match | process memory에 2분 |
| social/tournament API | 권한에 따라 사용 | 차단 또는 빈 transient view |

최종 불변 조건은 다음과 같습니다.

- guest credential은 등록 사용자 DB session/ticket으로 fallback하지 않습니다.
- guest의 생성·ticket 발급·connection에는 per-IP와 process 전체 hard bound가 있습니다.
- stale socket의 lease release가 replacement connection을 지우지 않습니다.
- registered와 guest는 같은 match에 들어가지 않습니다.
- guest 경기 결과는 의도적으로 non-persistent이며 rating을 바꾸지 않습니다.
- 짧은 reconnect 경험은 제공하지만 process restart를 넘는 durable recovery는 약속하지 않습니다.

이 Thread의 HMAC cookie는 payload 기밀성을 제공하지 않으며, IP binding은 proxy 설정과 `request.ip` 신뢰가 올바르다는 전제에 의존합니다. 일반 WebSocket handshake buffer와 request-log redaction은 Thread 02, room reconnect 좌석 예약은 Thread 05가 담당합니다.
