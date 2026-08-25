# Thread: guest browser capability, transient result, 경기 복구

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

guest demo는 로그인 form 하나를 추가하는 기능이 아닙니다. 같은 application이 registered mode와 demo mode에서 서로 다른 능력을 노출해야 하고, 브라우저가 저장되지 않는 결과를 durable 전적으로 오해하게 만들지 않아야 하며, 경기 중 socket이 끊기거나 사용자가 lobby로 돌아온 경우 기존 room을 복구해야 합니다.

이 Thread는 deployment mode를 browser capability policy로 바꾸고, 그 policy를 다음 경계에 적용합니다.

- inputless guest entry
- navigation과 direct URL restriction
- lobby/play presentation
- persisted progress와 chat 노출 여부
- transient `game.finished` notice
- active room event를 lobby에서 회수했을 때 `/play` 복귀
- fresh-ticket reconnect와 duplicate matchmaking 차단
- browser E2E의 실제 두 사용자·시간·socket 단절 검증

### 최종 불변 조건

- server `APP_MODE`와 browser `NEXT_PUBLIC_APP_MODE`가 같은 deployment capability를 선택해야 합니다.
- demo navigation은 lobby/play만 보여 주고 middleware는 restricted URL을 404로 차단합니다.
- guest UI는 rating·ranking·history·chat 같은 unsupported durable 기능을 제공한다고 광고하지 않습니다.
- `persisted: false` 결과는 전적이 아니라 임시 결과로 표시합니다.
- lobby에서 `queue.matched` 또는 `game.snapshot`을 받으면 새 queue를 만들지 않고 game screen으로 돌아갑니다.
- reconnect는 새 ticket을 쓰되 최초 queue command를 재전송하지 않습니다.

### 범위 주의

이 문서는 browser-owned policy와 browser integration evidence를 다룹니다. guest session signing, ticket hash/retention, IP별 rate limit, transient result server producer는 upstream server subsystem입니다. mixed test commit의 server assertion을 browser 보장으로 해석하지 않습니다.

`4f5199097284`는 Thread 04와 중복됩니다. 여기서는 `HomePage`와 `demoPolicy` 변경만 다루고 transport/reducer 구현은 Thread 04에서 설명합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f801ccd09cf0` | `feat(guest): guest runtime 환경 경계 구성` | A | AUTH, WEB | application mode와 proxy trust를 runtime/public browser configuration에 명시합니다. |
| 2 | `e39316254e44` | `feat(web): 비회원 체험 정책 경계 추가` | B | WEB | guest demo surface용 centralized browser policy module을 도입합니다. |
| 3 | `a9fc8a8328b2` | `feat(web): guest login API와 middleware 연결` | B | AUTH, PROTOCOL, TOURNAMENT | guest session adapter와 registered-only screen 차단 middleware를 연결합니다. |
| 4 | `7fdef5d224c4` | `feat(web): LoginPanel guest 진입 연결` | B | WEB | demo mode에서 login panel을 server-managed guest entry에 연결합니다. |
| 5 | `584d17f3aad1` | `feat(web): guest lobby presentation 적용` | B | REALTIME, WEB | guest lobby가 durable progress와 unsupported interaction을 광고하지 않게 합니다. |
| 6 | `658fafd43f88` | `feat(web): demo navigation 정책 연결` | B | WEB | `AppShell`이 capability에 맞는 navigation만 구성합니다. |
| 7 | `fe0f3e0ad0ad` | `feat(web): guest play presentation 적용` | B | WEB | demo mode에서 지원하지 않는 match chat을 숨깁니다. |
| 8 | `618330916629` | `test(web): 비회원 체험 진입 흐름 검증` | B | AUTH, WEB, TEST | guest entry와 browser API boundary를 검증합니다. |
| 9 | `9f49db1d9f1d` | `test(guest): 체험 기능 오용 방지 검증` | A | AUTH, WEB, RISK | guest-mode abuse와 browser capability isolation의 negative boundary를 확장 검증합니다. |
| 10 | `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | AUTH, REALTIME, WEB | guest lobby에서 transient result를 명시하고 active room event를 회수하면 game screen으로 복귀합니다. |
| 11 | `06d2eb7a93cc` | `test(guest): 체험 환경의 복구 경계 검증` | A | AUTH, SIMULATION, REALTIME | fresh-ticket reconnect, duplicate match 차단, transient result presentation을 확장 검증합니다. |
| 12 | `1abda1299ad8` | `test(e2e): 비회원 체험 브라우저 흐름 검증` | A | AUTH, REALTIME, WEB | demo-mode guest entry, restricted navigation, two-browser PvP, 6초 AI fallback, ticket-based reconnect를 Playwright로 검증합니다. |

## mode를 명시적 deployment contract로 만들기

### `f801ccd09cf0`

`.env.example`에 server와 browser mode가 함께 들어갑니다.

```env
APP_MODE=development
TRUST_PROXY=0
NEXT_PUBLIC_APP_MODE=development
```

API env는 `appMode`와 `trustProxy`를 typed value로 반환합니다. demo/production에서는 기본 개발 secret을 허용하지 않고 UTF-8 32바이트 이상의 `SESSION_SECRET`을 요구합니다.

```ts
if (
  (appMode === "demo" || appMode === "production")
  && (!configuredSecret || Buffer.byteLength(configuredSecret, "utf8") < 32)
) {
  throw new Error(...);
}
```

`TRUST_PROXY`는 값이 `1`일 때만 true입니다. public guest traffic에서 forwarded address를 무조건 신뢰하면 IP 기반 제한의 identity가 깨질 수 있으므로 explicit opt-in입니다.

이 SHA의 `readAppMode`는 `APP_MODE === "demo"`를 먼저 보고, 그 밖의 production/test는 `NODE_ENV`로 결정합니다. 후속 commit/test가 explicit production/test mode parsing을 강화하더라도 그 결과를 이 SHA에 소급하지 않습니다.

browser의 `NEXT_PUBLIC_APP_MODE`는 build-time 값입니다. server runtime mode와 자동으로 일치하는 교차 검증은 이 커밋에 없습니다. deployment가 두 값을 일관되게 설정해야 합니다.

## policy를 component 조건문 밖으로 빼기

### `e39316254e44`

`demoPolicy.ts`가 navigation item identity, registered navigation, demo presentation flags, restricted path를 한곳에 정의합니다.

```ts
export const demoLobbyPresentation = {
  description: "빠른 매칭으로 다른 게스트를 찾고, 상대가 없으면 인공지능과 바로 경기할 수 있습니다.",
  showPersistedProgress: false,
  showLeaderboardLink: false,
  showLobbyChat: false,
  showMatchChat: false
} as const;
```

```ts
export function createNavigation(demoMode, profileHref) {
  const navigation = registeredNavigation(profileHref);
  return demoMode
    ? navigation.filter((item) => item.id === "lobby" || item.id === "play")
    : navigation;
}
```

restricted prefix는 dashboard, leaderboard, tournaments, profile, admin입니다. navigation을 숨기는 것과 direct URL을 막는 것은 다른 책임이므로 pure policy가 두 consumer를 모두 지원합니다.

이 module은 server authorization을 구현하지 않습니다. browser surface가 mode에 따라 일관된 capability를 말하도록 하는 contract입니다.

## guest entry와 route boundary

### `a9fc8a8328b2` — API helper + middleware

browser adapter에 `guestLogin(signal)`이 추가됩니다.

```ts
return apiFetch("/auth/guest", guestAuthResponseSchema, {
  method: "POST",
  signal
});
```

사용자 입력이나 browser-generated display name을 보내지 않고 server가 생성한 guest identity와 cookie session을 validated response로 받습니다.

middleware는 demo mode에서 restricted pathname을 404로 반환합니다.

```ts
if (isDemoMode() && isDemoRestrictedPath(request.nextUrl.pathname)) {
  return new NextResponse("Not Found", { status: 404 });
}
```

404를 사용하면 guest에게 존재하지 않는 surface처럼 보이게 하며, navigation 숨김만 우회해 직접 URL로 들어가는 것을 막습니다. API server의 authorization은 별도 경계입니다.

### `7fdef5d224c4` — inputless LoginPanel

`LoginPanel`이 `isDemoMode()`에 따라 두 흐름을 렌더링합니다.

- registered/development: handle/display name 입력 + 개발 로그인
- demo: 입력 없이 “게스트로 시작” mutation

성공 response user를 `me` cache에 넣고 login 영향 key를 invalidate합니다. guest name을 form에서 받지 않으므로 identity naming policy는 server가 소유합니다. pending/error도 mutation state로 표시됩니다.

## capability를 화면에 적용

### `584d17f3aad1` — lobby presentation

demo mode에서 hero description을 policy text로 바꾸고 persisted progress·leaderboard link·lobby chat을 숨깁니다. score/rating/history가 저장되는 것처럼 보이는 card와 “경기가 끝나면 전적과 순위가 갱신된다”는 문구를 노출하지 않습니다.

기능을 server에서 거부하기만 하고 UI에 남겨 두면 guest는 실패하기 전까지 지원 기능으로 오해합니다. 이 commit은 capability를 presentation에 반영합니다.

### `658fafd43f88` — `AppShell` navigation

AppShell의 inline navigation 배열을 `createNavigation(demoMode, profileHref)` 결과로 교체합니다. icon은 item ID에 대응시키고 stable ID는 그대로 key로 사용합니다. demo mode에서는 로비와 경기만 남습니다.

현재 사용자 확인 전 profile link를 비활성화하는 Thread 01의 규칙은 registered navigation에만 적용되며 demo navigation에는 profile item 자체가 없습니다.

### `fe0f3e0ad0ad` — play presentation

`demoLobbyPresentation.showMatchChat`가 false이면 match-chat card와 input을 렌더링하지 않습니다. transport가 chat event를 받을 수 있는지와 별개로 browser가 unsupported interaction을 제공하지 않습니다. game controls와 authoritative canvas는 그대로 유지됩니다.

## policy test와 mixed abuse test의 구분

### `618330916629`

unit/component/API tests가 다음을 확인합니다.

- demo navigation에 lobby/play만 있는가
- restricted path predicate가 정확한가
- guest presentation flags가 durable progress/chat을 false로 두는가
- guest login helper가 POST, cookie credential, runtime schema, AbortSignal 경계를 사용하는가
- LoginPanel이 demo mode에서 input field 없이 guest action을 제공하는가

이는 browser policy와 adapter wiring을 검증합니다.

### `9f49db1d9f1d`

이 A급 mixed test commit은 guest session/ticket/socket/resource abuse와 browser capability isolation을 함께 확장합니다. browser Thread에서 취할 수 있는 evidence는 다음과 같습니다.

- restricted UI와 direct route가 demo capability 밖에 있는가
- guest가 registered-only browser action을 정상 capability로 소비하지 못하는가
- public mode 설정이 browser/server test fixture에서 분리되는가

IP별 session/ticket limit, connection cap, persistence 차단 같은 assertion은 server subsystem의 보장입니다. 그것을 “browser policy가 보안을 완성한다”는 근거로 사용하지 않습니다.

## `4f5199097284` — lobby가 active room과 transient result를 해석

이 commit의 reconnect transport 변경은 Thread 04에서 다룹니다. 여기서는 HomePage와 `demoPolicy`만 봅니다.

### active room recovery

lobby socket이 다음 event를 받으면 demo browser는 `/play`로 이동합니다.

```ts
export function shouldResumeGameFromLobby(event: { type: string }): boolean {
  return event.type === "queue.matched" || event.type === "game.snapshot";
}
```

```ts
if (demoMode && shouldResumeGameFromLobby(message)) {
  window.location.assign("/play");
  return;
}
```

사용자가 game 중 lobby에 있거나 reconnect 과정에서 room event가 lobby socket으로 회수된 경우, 새 matchmaking button을 누르게 두지 않고 active game consumer로 복귀시킵니다. `game.finished`는 이미 terminal이므로 resume 대상이 아닙니다.

### transient result

`game.finished`의 `result.persisted`가 false면 durable history로 취급하지 않고 notice를 표시합니다.

```ts
export function formatTransientResultNotice(result): string {
  return `임시 경기 종료: ${result.leftScore} - ${result.rightScore} · 전적에 저장되지 않았습니다.`;
}
```

HomePage는 `role="status"`인 amber notice로 표시합니다. score는 알려 주되 leaderboard/dashboard에 저장됐다고 암시하지 않습니다.

## `06d2eb7a93cc` — recovery invariant를 deterministic test로

이 commit도 server와 browser test가 섞여 있습니다. browser 관련 assertion은 세 묶음입니다.

### fresh ticket, no duplicate queue

fake timer와 fake socket으로 첫 socket을 연 뒤 close합니다. reconnect timer를 진행시키면 ticket provider가 두 번째로 호출되고 두 번째 socket이 생깁니다. 첫 socket은 최초 queue command 하나를 보냈지만 reconnect socket의 sent list는 비어 있어야 합니다.

```ts
expect(ticketProvider).toHaveBeenCalledTimes(2);
expect(sockets).toHaveLength(2);
expect(sockets[1].sent).toEqual([]);
```

### new-match predicate

room이 있는 `reconnecting` state에서 `canStartNewMatch`는 false이고 `finished`에서는 true입니다. UI disabled와 hook guard가 같은 pure predicate를 사용합니다.

### transient/result route policy

- `persisted:false`, 1–3 결과의 exact Korean notice
- `queue.matched`와 `game.snapshot`은 resume true
- `game.finished`는 false

같은 commit의 env parser, guest access cleanup, ticket/IP capacity test는 upstream server evidence입니다.

## `1abda1299ad8` — 실제 browser 흐름

별도 command는 demo mode에서 guest E2E 파일을 worker 1로 serial 실행합니다.

```json
"e2e:guest-demo": "E2E_APP_MODE=demo playwright test tests/e2e/guest-demo.spec.ts --workers=1"
```

### 1. inputless entry와 제한된 navigation

홈에서 handle input이 없는지 확인하고 “게스트로 시작”을 누릅니다. server-generated `게스트 0000` 형태 welcome heading을 확인한 뒤 navigation link text가 정확히 `로비`, `경기`인지 검증합니다.

### 2. 두 browser PvP

서로 다른 browser context에서 guest session을 만들고 두 page가 queue button을 누릅니다. 양쪽이 서로의 display name을 보고 `준비 대기 중`에 도달한 뒤 ready를 눌러 `경기 진행 중`을 확인합니다. 같은 cookie jar를 공유하지 않는 실제 두 session 흐름입니다.

### 3. 6초 AI fallback

WebSocket sent/received frame의 JSON type과 `Date.now()`를 기록합니다. `queue.join` 전송과 `queue.matched` 수신 간격이 5.5초 이상 10초 미만인지 확인하고 opponent가 “연습 AI”인지 봅니다. 단순 text timeout이 아니라 protocol frame 간 시간을 측정합니다.

### 4. forced reconnect

Playwright `routeWebSocket`으로 page-server connection pair를 기록하고 경기 진행 중 첫 page-side socket을 code 1012로 닫습니다. UI가 2초 안에 “재연결 대기 중”, 5초 안에 두 번째 connection과 “경기 진행 중”에 도달하며 pause button이 다시 enabled인지 확인합니다.

이 E2E가 증명하지 않는 것:

- browser/server public mode 값이 production deployment에서 자동으로 일치함
- 모든 network timing과 reconnect deadline
- server guest store의 장기 memory bound
- production에서 실제 command가 통과했다는 사실 — 이 작업 환경에서는 실행하지 않음

## capability matrix

| Surface | Registered mode | Demo mode |
| --- | --- | --- |
| 로그인 | 개발/등록 사용자 입력 | inputless guest entry |
| navigation | lobby, play, dashboard, leaderboard, tournaments, profile, admin | lobby, play |
| direct restricted route | route별 auth/권한 처리 | middleware 404 |
| durable progress/rating claim | 표시 가능 | 숨김 |
| leaderboard link | 표시 | 숨김 |
| lobby/match chat | 지원 경로 표시 | 숨김 |
| game state | authoritative snapshot | authoritative snapshot |
| finished result | persisted result로 소비 가능 | `persisted:false`면 임시 결과 명시 |
| active-room recovery | game connection policy | lobby event도 `/play`로 복귀 |

## 최종 흐름

```text
[deployment]
  server APP_MODE + secret/proxy validation
  browser NEXT_PUBLIC_APP_MODE
          ↓
[demoPolicy]
  navigation / restricted paths / presentation flags
          ↓
[entry]
  guestLogin POST -> cookie session + validated server-named user
          ↓
[shell/pages]
  lobby/play만 노출, unsupported durable/chat surface 숨김
          ↓
[lobby realtime]
  queue.matched | game.snapshot -> /play
  game.finished persisted:false -> transient notice
          ↓
[game transport]
  remote close with active room -> fresh ticket reconnect
  original matchmaking intent 재전송 없음
```

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. unit/integration/Playwright 명령은 이 환경에서 실행하지 않았습니다.
