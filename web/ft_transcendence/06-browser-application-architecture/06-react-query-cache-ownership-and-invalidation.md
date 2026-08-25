# Thread: React Query cache를 브라우저 server-state의 단일 owner로 만들기

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

sample fallback을 제거한 뒤에도 각 page는 `useEffect + useState`로 같은 endpoint를 독립적으로 요청했습니다. `HomePage`와 `AppShell`이 `/me`를 따로 읽고, mutation 뒤 어느 화면을 갱신해야 하는지 caller마다 결정했으며, 401 뒤 private data를 한 번에 버릴 owner가 없었습니다.

이 Thread는 browser server state를 다음 계약으로 이동합니다.

- application 수명 동안 하나의 `QueryClient`
- resource scope가 드러나는 stable tuple key
- endpoint별 query function·stale time·AbortSignal
- mutation별 exact invalidation
- 401에서 private cache만 제거하는 session transition
- HTTP load, mutation result, WebSocket push가 같은 cache entry를 갱신

component local state는 form input·notice처럼 browser presentation에 남고, server response는 canonical cache가 소유합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e000e3d6a460` | `refactor(web): query key와 retry 정책 정의` | B | AUTH, TOURNAMENT, WEB | screen migration 전에 cache key vocabulary와 retry policy를 정의합니다. |
| 2 | `d05a962d8829` | `refactor(web): session query와 cache invalidation 추가` | A | TOURNAMENT, WEB, OBSERVABILITY | endpoint별 query option과 session-expiry invalidation을 정의합니다. |
| 3 | `80ec34fde74c` | `refactor(web): React Query provider 연결` | B | AUTH, WEB, OBSERVABILITY | application root에 단일 `QueryClient`를 설치합니다. |
| 4 | `931800f796e1` | `refactor(web): lobby와 login을 query cache로 전환` | B | AUTH, REALTIME, WEB | HTTP load, socket event, login/chat mutation이 동일 lobby/session cache owner를 갱신합니다. |
| 5 | `8a44c23f15de` | `refactor(web): dashboard와 leaderboard를 query cache로 전환` | B | WEB | dashboard와 leaderboard component effect를 shared cache로 이동합니다. |
| 6 | `e2ccee689642` | `refactor(web): profile 조회를 query cache로 전환` | B | WEB | route handle을 query identity로 사용합니다. |
| 7 | `045d0cd2c171` | `refactor(web): tournament 조회와 mutation을 query cache로 전환` | B | TOURNAMENT, WEB | tournament read와 create/join mutation을 cache로 이동합니다. |
| 8 | `0b1a6bcb4311` | `refactor(web): admin 조회와 mutation을 query cache로 전환` | B | AUTH, WEB | admin read/status mutation을 cache로 이동합니다. |
| 9 | `0e0c9645ab2d` | `refactor(web): shell의 session 소비를 query cache로 통합` | B | AUTH, WEB | `AppShell`의 private session effect를 shared `me` query로 교체합니다. |
| 10 | `1ebdce4cdf0a` | `test(web): query cache key·retry·invalidation 검증` | A | AUTH, TOURNAMENT, WEB | key scoping, retry exclusion, mutation/session invalidation을 검증합니다. |

## key는 문자열 이름이 아니라 resource scope다

### `e000e3d6a460`

screen migration보다 먼저 query vocabulary를 고정합니다.

```ts
export const queryKeys = {
  me: () => ["user", "me"] as const,
  lobby: () => ["lobby"] as const,
  dashboard: () => ["dashboard"] as const,
  profile: (handle: string) => ["profiles", handle] as const,
  leaderboard: () => ["leaderboard"] as const,
  friends: () => ["friends"] as const,
  tournaments: () => ["tournaments"] as const,
  adminUsers: () => ["admin", "users"] as const,
  adminActions: () => ["admin", "actions"] as const
};
```

profile handle은 key의 일부이므로 `/profile/a`와 `/profile/b`가 같은 data slot을 덮지 않습니다. admin users/actions는 같은 prefix를 공유하지만 mutation invalidation은 `exact: true`로 지정합니다.

```ts
await Promise.all(keys.map((queryKey) =>
  client.invalidateQueries({ queryKey, exact: true })
));
```

mutation impact도 이름으로 모읍니다.

- login → me + lobby
- lobby chat → lobby
- friend request → friends
- tournament change → tournaments
- admin status → admin users + admin actions

retry는 401이면 즉시 false, 일반 오류는 첫 실패 뒤 한 번만 다시 시도합니다. expired credential은 같은 cookie로 반복해도 성공하지 않으므로 재시도 대상이 아닙니다.

## `d05a962d8829` — query option과 session transition

### endpoint마다 다른 freshness

query function은 React Query가 제공한 `signal`을 API helper에 전달합니다.

```ts
export const lobbyQueryOptions = () => queryOptions({
  queryKey: queryKeys.lobby(),
  queryFn: ({ signal }) => getLobby(signal),
  staleTime: 5_000
});
```

stale time은 resource 성격에 맞게 다릅니다.

| Resource | stale time |
| --- | ---: |
| lobby, admin users/actions | 5초 |
| dashboard, tournaments | 10초 |
| leaderboard | 15초 |
| me, profile | 30초 |

이는 “모두 같은 캐시 시간”이 아니라 변동성·민감도에 따른 policy입니다. query가 더 이상 관찰되지 않거나 새 request가 시작될 때 signal을 통해 in-flight fetch를 취소할 수 있습니다.

### 401 뒤 어떤 cache를 버릴 것인가

`expireSession`은 private key를 명시적으로 열거합니다.

```ts
const sessionScopedKeys = [
  queryKeys.lobby(),
  queryKeys.dashboard(),
  queryKeys.friends(),
  queryKeys.adminUsers(),
  queryKeys.adminActions()
] as const;
```

public leaderboard, public profile, tournament list는 제거하지 않습니다. `me`는 삭제가 아니라 `null`로 설정해 “현재 인증 사용자 없음”을 canonical value로 만듭니다.

### fetching query를 동기 remove하지 않는 이유

query가 자신의 callback 안에서 401을 감지한 순간 바로 remove되면 observer가 error/idle terminal state에 도달하기 전에 cache entry가 사라져 pending처럼 남을 수 있습니다. 따라서 active fetching key는 다음 task로 제거합니다.

```ts
if (client.getQueryState(queryKey)?.fetchStatus === "fetching") {
  setTimeout(() => client.removeQueries({ queryKey, exact: true }), 0);
} else {
  client.removeQueries({ queryKey, exact: true });
}
```

이 작은 scheduling decision이 session expiry를 단순 data clear가 아니라 observer lifecycle transition으로 만듭니다.

## application root에 owner 설치

### `80ec34fde74c`

`RootLayout`이 `QueryProvider`로 전체 application을 감싸고, provider는 `useState` initializer로 client를 한 번만 만듭니다.

```tsx
const [client] = useState(() => new QueryClient({
  defaultOptions: {
    queries: {
      retry: shouldRetryQuery,
      refetchOnWindowFocus: true
    },
    mutations: { retry: false }
  }
}));
```

render마다 새 client를 만들지 않으므로 route 이동과 rerender를 넘어 cache가 유지됩니다. browser API adapter가 `SESSION_EXPIRED_EVENT`를 dispatch하면 provider가 `expireSession(client)`를 실행하고 cleanup에서 listener를 제거합니다.

mutation은 자동 retry하지 않습니다. create/join/status 변경처럼 side effect가 있는 request를 caller 동의 없이 반복하지 않는 policy입니다.

## screen migration: local fetch 제거보다 중요한 것

### `931800f796e1` — lobby·login·WebSocket이 같은 entry를 사용

HomePage는 `useQuery(meQueryOptions())`, `useQuery(lobbyQueryOptions())`를 사용합니다. lobby response 안의 `me`가 있으면 그것을 우선하고, 없으면 `me` query를 사용합니다.

login mutation은 성공 user를 `queryKeys.me()`에 즉시 넣고 login 영향 key를 invalidation합니다. LoginPanel의 local `onLogin` callback이 사라지고 pending/error도 mutation state가 소유합니다.

lobby chat은 세 producer가 같은 `queryKeys.lobby()`를 사용합니다.

1. HTTP query가 최초 list를 채움
2. HTTP chat mutation 성공이 message를 cache에 merge하고 invalidation
3. WebSocket `chat.message`가 같은 cache entry에 dedupe/append

```ts
queryClient.setQueryData<LobbyResponse>(queryKeys.lobby(), (current) =>
  current ? {
    ...current,
    chat: [...current.chat.filter((item) => item.id !== message.id).slice(-19), message]
  } : current
);
```

presence event는 local `loadLobby` 함수를 부르는 대신 exact lobby key를 stale로 만듭니다. HTTP와 realtime이 별도 state owner를 갖지 않습니다.

### `8a44c23f15de` — dashboard·leaderboard

두 route의 mount effect와 local data state를 제거하고 query result의 `isPending`, `isError`, `data`로 rendering branch를 구성합니다. 동일 route를 다시 방문했을 때 stale policy 안이면 cache를 재사용하며, component unmount가 resource data를 잃는 기준이 되지 않습니다.

### `e2ccee689642` — profile handle이 identity

params에서 resolve된 handle을 `profileQueryOptions(handle)`에 전달합니다. handle 변경은 같은 component의 local setter 문제가 아니라 다른 query key가 됩니다. public profile cache는 session expiry에도 보존됩니다.

### `045d0cd2c171` — tournament read + mutation

list query와 create/join mutation을 같은 tournament key에 연결합니다. mutation success는 필요한 local feedback을 갱신하고 exact tournament key를 invalidate합니다. 다른 resource key를 광범위하게 refetch하지 않습니다.

### `0b1a6bcb4311` — admin users/actions

admin list와 audit action을 별도 key로 읽고 status mutation 뒤 두 exact key만 invalidate합니다. authorization failure는 query error로 남으며 sample users로 대체되지 않습니다.

### `0e0c9645ab2d` — shell의 `/me` 중복 제거

`AppShell`이 자체 `useEffect(getMe)`와 local `me` state를 제거하고 application의 `me` query를 소비합니다. HomePage/login과 navigation profile target이 같은 session value를 보게 됩니다. 이 commit으로 `/me`의 duplicate request owner가 사라집니다.

## `1ebdce4cdf0a` — cache 정책을 object state로 직접 검증

테스트는 DOM 대신 실제 `QueryClient`와 `QueryObserver`를 사용합니다.

### key와 exact invalidation

모든 tuple shape를 비교하고 admin status invalidation 뒤 admin users/actions는 `isInvalidated=true`, unrelated leaderboard는 false인지 확인합니다. prefix가 같다는 이유로 인접 cache를 지우지 않는 계약입니다.

### retry

401 `ApiError`는 첫 시도에서도 false, 일반 error는 failure count 0에서 true, 1에서 false입니다.

### private/public 분류

client에 private와 public data를 모두 넣고 `expireSession` 뒤 결과를 확인합니다.

| 제거/변경 | 보존 |
| --- | --- |
| me → null | leaderboard |
| lobby 제거 | profile(handle) |
| dashboard 제거 | tournaments |
| friends 제거 |  |
| admin users/actions 제거 |  |

### active unauthorized query가 pending에 남지 않음

`QueryObserver`의 query function 내부에서 `expireSession(client)`를 호출한 뒤 error를 throw합니다. 다음 task까지 기다린 후 observer 결과가 `status: "error"`, `fetchStatus: "idle"`인지 확인합니다. fetching query removal을 defer한 이유를 직접 겨냥합니다.

이 테스트는 실제 browser event wiring이나 network 취소를 실행하지 않습니다. cache policy와 observer state transition을 검증합니다. 이 작업 환경에서는 test source만 검사했습니다.

## 최종 ownership matrix

| 데이터 | Canonical key | 주요 writer | expiry 분류 |
| --- | --- | --- | --- |
| current session | `['user','me']` | me query, login mutation, expireSession | private; expiry 시 null |
| lobby | `['lobby']` | query, chat mutation, WS chat/presence invalidation | private; 제거 |
| dashboard | `['dashboard']` | query | private; 제거 |
| profile | `['profiles', handle]` | query | public; 보존 |
| leaderboard | `['leaderboard']` | query | public; 보존 |
| tournaments | `['tournaments']` | query, create/join invalidation | public 목록; 보존 |
| admin users/actions | `['admin', ...]` | queries, status invalidation | private; 제거 |

## 최종 흐름

```text
[RootLayout]
  -> one QueryClient
  -> API session-expired event listener

[Screen]
  -> queryOptions(key, queryFn(signal), staleTime)
  -> cache data / pending / error

[Mutation]
  -> server response
  -> optional exact setQueryData
  -> affected exact keys invalidation

[WebSocket]
  -> scope 확인
  -> same canonical cache update 또는 exact invalidation

[401]
  -> no retry
  -> session-expired event
  -> private idle keys remove
  -> private fetching keys deferred remove
  -> me = null
  -> public caches preserve
```

이 Thread는 cache ownership만 다룹니다. HTTP payload validation은 Thread 02, game connection reducer state는 Thread 04, guest capability 정책은 Thread 08의 책임입니다.

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. Vitest·browser runtime은 실행하지 않았습니다.
