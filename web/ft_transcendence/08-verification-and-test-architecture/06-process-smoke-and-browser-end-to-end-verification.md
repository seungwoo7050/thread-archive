# Thread: 실행 process smoke와 browser end-to-end 검증

unit test와 Fastify injection은 함수·plugin composition을 빠르게 검증하지만, 빌드된 process가 실제 port에서 listen하고 HTTP, WebSocket, browser rendering이 연결되는지는 보여 주지 못합니다. 이 Thread는 외부 client를 사용하는 process smoke와 Playwright browser E2E를 도입한 뒤, 초기 테스트가 제품의 인증·protocol 변화에 뒤처지자 **테스트 자체를 cookie/ticket/v1 경계로 migration**하는 과정을 다룹니다.

```text
unit/injection
  → process HTTP smoke
  → process WebSocket smoke
  → browser navigation/rendering
  → multi-browser guest/reconnect recovery
```

각 층은 아래층을 대체하지 않습니다. 더 많은 실제 구성 요소를 포함하는 대신 실패 원인을 더 넓게 관찰합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9a0562d395db8a2a9d37fa8496c33207913f5576` | `test(smoke): HTTP API 실행 검사 추가` | B | `AUTH, OPERATIONS, TEST` | 당시 bearer session으로 실행 중 API의 로그인·주요 read path를 확인한다. |
| 2 | `8a462f6f05b3612a8058cd8d4d474c9910d2b487` | `test(smoke): WebSocket 경기 실행 검사 추가` | B | `AUTH, PROTOCOL, SIMULATION` | 당시 query session과 unversioned event로 실제 matchmaking→playing→chat 경로를 확인한다. |
| 3 | `d755b8dae2c1a73507bc3526a510f51159fd2759` | `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성` | B | `PERSISTENCE, TOURNAMENT, WEB` | desktop/mobile Playwright runner와 한국어 navigation·Canvas paint 검증을 도입한다. |
| 4 | `17e7dab21a550d4bcd85fa1c4f93a57bf4381942` | `test(smoke): cookie 기반 realtime protocol 검증` | B | `AUTH, PROTOCOL, SIMULATION` | 기존 smoke를 cookie-only, one-time ticket, version-1 protocol로 교정한다. |
| 5 | `1abda1299ad80e299e79297060531eee90112fb8` | `test(e2e): 비회원 체험 브라우저 흐름 검증` | A | `AUTH, REALTIME, WEB` | guest capability, 두 browser PvP, 6초 AI fallback, fresh-ticket reconnect를 실제 browser에서 결합 검증한다. |

## 1. process smoke는 “서버를 만들 수 있음”이 아니라 “실제로 말할 수 있음”을 본다

### `9a0562d395db...` — 초기 HTTP smoke

`tests/smoke-api.mjs`는 기본적으로 `http://localhost:4000`의 실행 중 process를 호출합니다.

```js
const baseUrl = process.env.API_BASE_URL ?? "http://localhost:4000";

const login = await request("/auth/dev-login", {
  method: "POST",
  body: JSON.stringify({ handle: "smoke", displayName: "스모크" })
});

await request("/me", {
  headers: { authorization: `Bearer ${login.token}` }
});
```

순서는 개발 로그인→`/me`→`/lobby`→public leaderboard→`/dashboard`입니다. request helper는 non-2xx 응답을 path, status, response text가 포함된 error로 바꿉니다. script가 reject되면 shell/CI에서도 non-zero로 드러납니다.

이 smoke가 unit test와 다른 점은 다음을 실제 process boundary에서 포함한다는 것입니다.

- build artifact와 Node entrypoint
- port listen
- Fastify plugin/route registration
- repository composition
- 외부 HTTP serialization

그러나 이 commit의 bearer 방식은 당시 history에만 맞습니다. 이후 cookie-only 전환 뒤에도 이 script가 통과하도록 bearer fallback을 유지하면 테스트가 보안 경계를 약화시키게 됩니다. 따라서 이 초기 상태를 최종 계약처럼 서술해서는 안 됩니다.

### `8a462f6f05b3...` — 초기 WebSocket smoke

다음 commit은 Makefile의 `smoke` target에 HTTP와 WebSocket script를 함께 넣습니다. `tests/smoke-ws.mjs`는 두 사용자를 로그인하고 당시 JSON token을 `?session=` query에 넣어 두 socket을 엽니다.

```js
function connect(token) {
  return new WebSocket(`${wsUrl}?session=${token}`);
}
```

두 socket이 open되면 unversioned command를 보냅니다.

```json
{ "type": "queue.join", "mode": "queue" }
{ "type": "game.ready", "roomId": "..." }
{ "type": "chat.send", "scope": "match", "roomId": "...", "body": "준비됐습니다." }
```

script는 `queue.matched`, playing snapshot, chat message를 10초 polling timeout 안에서 기다리고 마지막에 두 socket을 닫습니다.

#### 초기 script가 실제로 증명하는 범위

이 exact diff는 양쪽 event를 하나의 `events` 배열에 넣고 **처음 발견한 `queue.matched` 하나**의 room ID로 두 ready command를 보냅니다. 양쪽 socket이 각각 match event를 받았고 room ID가 동일하다는 assertion은 아직 없습니다.

따라서 이 commit이 증명하는 것은 다음 정도입니다.

- 실제 process에서 두 connection을 열 수 있음
- queue command 뒤 어떤 match event가 발생함
- ready 뒤 playing snapshot이 발생함
- match chat이 왕복함

“두 client 모두 같은 room assignment를 관찰한다”는 더 강한 조건은 `17e7...`에서 left/right event를 분리해 확인할 때 비로소 고정됩니다. 기존 문서의 설명보다 repository diff를 우선해야 하는 대표 사례입니다.

## 2. browser test는 DOM 존재보다 실제 사용자 표면을 본다

### `d755b8dae2c...` — Playwright 실행 경계

이 commit은 root package에 Playwright를 추가하고 다음 runner 설정을 만듭니다.

```ts
export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  expect: { timeout: 10_000 },
  use: {
    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:3000",
    trace: "retain-on-failure",
    screenshot: "only-on-failure"
  },
  projects: [
    {
      name: "chromium-desktop",
      use: { ...devices["Desktop Chrome"], viewport: { width: 1448, height: 1086 } }
    },
    {
      name: "chromium-mobile",
      use: { ...devices["Pixel 7"] }
    }
  ]
});
```

첫 E2E는 한국어 accessible name을 사용합니다. heading “퐁퐁”, label “핸들/표시 이름”, button “개발 로그인”, link “대시보드/순위표/토너먼트”를 따라 실제 browser navigation과 rendered UI를 검증합니다.

### Canvas element가 아니라 paint를 확인한다

`<canvas>`가 DOM에 존재해도 renderer가 한 번도 그리지 않았을 수 있습니다. 두 번째 test는 2D context의 전체 pixel data를 읽고 alpha channel 중 하나라도 0이 아닌지 확인합니다.

```ts
const data = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
for (let index = 3; index < data.length; index += 4) {
  if (data[index] !== 0) return true;
}
return false;
```

이 assertion이 증명하는 것:

- canvas context를 얻을 수 있음
- renderer가 적어도 하나의 visible pixel을 그림

증명하지 않는 것:

- 그 pixel이 authoritative game state와 정확히 일치함
- animation이 올바른 cadence로 진행됨
- visual regression 전체가 없음
- Chromium 외 engine에서 동작함

즉 element-presence test보다 강하지만 screenshot/pixel-perfect simulation 검증은 아닙니다.

## 3. 테스트가 obsolete transport를 계속 사용하면 보안 회귀를 숨긴다

### `17e7dab21a55...` — smoke contract migration

cookie-only session과 one-time ticket, versioned realtime protocol이 도입된 뒤 초기 smoke는 제품과 다른 경로를 사용하게 되었습니다. 이 commit은 production fallback을 되살리는 대신 smoke를 새 경계로 교정합니다.

#### HTTP smoke

로그인 helper는 `Set-Cookie`를 추출합니다. response body에 `token` field가 있으면 즉시 실패합니다.

```text
POST /auth/dev-login
  → Set-Cookie 필요
  → JSON body에 token이 있으면 실패
```

`/me`, `/lobby`, lobby chat, `/dashboard`, tournament create 등 protected request는 cookie를 보냅니다. `admin` handle로 로그인한 일반 사용자가 `/admin/actions`를 호출하면 403이어야 합니다.

이 assertion은 “admin이라는 문자열을 아는 것”과 authorization role을 분리합니다.

#### WebSocket smoke

각 사용자 cookie로 `/auth/ws-ticket`을 호출한 뒤 다음 URL을 사용합니다.

```text
/ws?ticket=<one-time-ticket>&v=1
```

outbound command는 helper가 `v: 1`을 붙이고, inbound event는 version을 확인한 뒤 parse합니다. snapshot은 구형 flat shape가 아니라 `snapshot.state` 아래 authoritative state를 읽습니다.

수정된 script는 초기보다 더 강한 조건을 검증합니다.

- 두 handle이 presence에 보임
- left/right socket이 각각 `queue.matched`를 수신
- 두 event의 `roomId`가 동일함
- 양쪽 ready 뒤 playing state
- 초기 공 속도가 최소 기준을 만족하고 이후 tick에서 가속됨
- pause→paused, resume→playing
- match chat 수신
- solo queue가 제한 시간 안에 AI 상대를 받고 snapshot에 `ai` player가 존재

이 commit에는 tournament/admin E2E 변경도 같은 diff에 포함되어 있지만, process smoke의 인증·protocol migration과 직접 관련 없는 부분은 이 Thread 설명에서 제외합니다.

### 이 변화가 중요한 이유

테스트는 production policy의 예외 통로가 아닙니다.

```text
나쁜 대응
  새 cookie-only 구현
  + 옛 smoke를 위해 Bearer fallback 유지

선택한 대응
  새 cookie-only 구현
  + smoke도 cookie→ticket→v1로 migration
```

후자는 테스트가 실제 공격 표면과 같은 경계를 통과하게 합니다.

## 4. `1abda1299ad8...` — guest recovery를 browser lifecycle로 검증한다

unit guest test는 signature·lease·capability를 확인할 수 있지만, 실제 browser가 제한된 navigation을 보이고 두 identity를 격리하며 fresh ticket reconnect를 수행하는지는 보여 주지 못합니다.

이 suite는 `E2E_APP_MODE=demo`일 때만 실행되고 serial mode/worker 1을 사용합니다. 일반 development server에 잘못 적용되지 않게 실행 mode를 분리합니다.

### 입력 없는 guest 진입과 capability 제한

page에는 handle input이 없어야 하며 “게스트로 시작” button으로 session을 만듭니다. 생성된 표시 이름은 `게스트 0000` 형식입니다.

로그인 뒤 navigation은 정확히 로비와 경기만 보여야 합니다. 관리 link는 없어야 하고 guest가 이용할 수 있는 빠른 매칭/AI 안내가 표시됩니다.

이 test는 backend 403만 보는 것이 아니라 **client가 허용되지 않은 capability를 애초에 노출하지 않는지** 확인합니다. 물론 UI 비노출이 authorization을 대신하지는 않습니다.

### 두 BrowserContext로 identity 격리

하나의 context 안에서 page 두 개를 열면 cookie jar를 공유합니다. test는 `browser.newContext()`를 두 번 호출해 완전히 독립된 guest session을 만듭니다.

```text
Context L → guest L cookie → socket L
Context R → guest R cookie → socket R
```

두 guest가 queue에 참가하면 각 page에서 상대 guest 이름과 “준비 대기 중”을 보고, 양쪽 ready 뒤 둘 다 “경기 진행 중”이 되어야 합니다. `finally`에서 두 context를 모두 닫아 cookie/socket/browser resource를 정리합니다.

### UI timeout이 아니라 WebSocket frame으로 AI fallback 시간을 잰다

fallback test는 page text가 나타났다는 사실만 사용하지 않습니다. Playwright의 WebSocket observer로 sent/received JSON frame의 timestamp를 기록합니다.

```text
sent     queue.join      t0
received queue.matched   t1

5,500ms ≤ t1 - t0 < 10,000ms
```

서버의 nominal fallback은 6초입니다. 하한은 너무 이른 AI 전환을 막고 상한은 browser/test 환경의 scheduling 여유를 둡니다. 이어 AI 상대를 보고 ready 후 playing까지 진행합니다.

### routeWebSocket으로 진행 중 connection을 끊는다

reconnect test는 network를 끄는 대신 `page.routeWebSocket`으로 browser-side connection과 server-side forwarding route를 관찰합니다.

1. guest가 AI room에서 ready→playing
2. 첫 connection 수가 1인지 확인
3. first page-side socket을 code 1012, reason `e2e reconnect`로 close
4. UI가 “재연결 대기 중”을 표시
5. connection 수가 2가 될 때까지 poll
6. 같은 경기 진행 상태가 다시 보임
7. “일시정지” command가 다시 enabled

새 WebSocket connection은 기존 one-time ticket을 재사용할 수 없으므로 application client가 fresh ticket을 발급해야 합니다. test가 ticket 문자열을 직접 비교하지는 않지만, ticket이 one-time인 production boundary에서 두 번째 authenticated connection이 성립한다는 behavior를 검증합니다.

## 증거 층 비교

| 층 | 포함하는 실제 구성 | 잘 잡는 실패 | 단독으로 증명하지 못하는 것 |
| --- | --- | --- | --- |
| unit/shared schema | 함수·schema | edge case, invalid shape | process listen/composition |
| Fastify injection | plugin·route handler | request/response contract | 실제 port/network/build artifact |
| HTTP process smoke | 실행 API + 외부 fetch | listen, route/repository composition | browser rendering, socket lifecycle |
| WebSocket process smoke | 실행 API + 실제 upgrade/socket | auth→match→snapshot→chat | browser UI/context behavior |
| Playwright E2E | web app + API + browser | navigation, paint, cookie/context, reconnect UX | 장시간/대규모 부하, 다른 browser engine |

## 최종 실행 흐름

```text
[실행 중 API/Web readiness]
        ↓
[dev 또는 guest login → HttpOnly cookie]
        ↓
[HTTP request는 cookie 사용]
        ↓
[POST /auth/ws-ticket]
        ↓
[WebSocket ?ticket=...&v=1]
        ↓
[queue / ready / input / chat / pause / reconnect]
        ↓
smoke: protocol/state 관찰
Playwright: UI/rendering/context lifecycle 관찰
        ↓
finally/timeout에서 socket·context 정리
```

## 이 Thread의 경계

- cookie/ticket의 보안 설계와 payload limit은 Thread 02입니다.
- reconnect room lease와 GameHub 자원 ownership은 Thread 04입니다.
- 결정적 simulation 자체는 Thread 03입니다.
- 수백 connection의 capacity와 fault recovery는 Thread 07입니다.
- CI가 이 suite를 실제 job에서 실행하는지는 이 category 범위 밖입니다.

## 조사·실행 기록

각 commit의 exact diff와 당시 script/config/test source를 확인했습니다. 초기 smoke의 bearer/query 방식을 후대 contract로 소급하지 않았고, 초기 WebSocket script가 양쪽 same-room을 명시적으로 확인하지 않았다는 범위도 repository 우선으로 수정했습니다. 이 환경에서는 API process나 Playwright를 실행하지 않았습니다.
