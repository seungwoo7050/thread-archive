# Thread: 브라우저 HTTP 신뢰 경계 — unchecked JSON에서 cookie·schema·취소 가능한 요청으로

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 화면마다 흩어질 수 있는 HTTP 호출을 `apiFetch`로 모으는 데서 시작해, 그 adapter가 **무엇을 신뢰하고 무엇을 거부해야 하는가**를 점차 좁혀 갑니다.

초기 adapter는 TypeScript generic만으로 응답을 `T`라고 단언하고, durable token을 `localStorage`에서 읽어 `Authorization` header에 붙였습니다. 모든 요청에 JSON content type을 붙였고, 일부 read helper는 실패를 sample data로 바꿨습니다. 이후 수정은 이 네 가지 가정을 차례로 제거합니다.

1. body가 없는 요청은 JSON body를 보낸다고 선언하지 않습니다.
2. caller가 지정한 header는 `Headers`를 통해 보존합니다.
3. 브라우저 JavaScript는 durable session token을 저장하거나 읽지 않습니다.
4. 성공 응답도 shared runtime schema를 통과해야 하며, WebSocket은 일회용 ticket만 사용합니다.

최종 adapter의 신뢰 경계는 다음과 같습니다.

```text
browser intent
  -> endpoint helper
  -> apiFetch(path, schema, init)
  -> cookie 포함 fetch + optional AbortSignal
  -> non-2xx: structured ApiError
  -> 2xx: JSON parse + runtime schema validation
  -> validated domain value
```

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `20618b30eda9` | `feat(web): 인증 API client 구현` | B | AUTH, REALTIME, TOURNAMENT | authentication과 초기 read model용 공통 browser HTTP adapter를 구현합니다. |
| 2 | `bfae9539cfe5` | `feat(web): 사용자 동작용 API 함수 추가` | B | TOURNAMENT, WEB | tournament join, profile/friend, admin mutation adapter를 추가합니다. |
| 3 | `177fa0b8502a` | `fix(web): body 없는 요청에서 JSON header 제외` | B | AUTH, WEB | body 없는 요청에서 JSON content type을 선언하지 않도록 request header construction을 수정합니다. |
| 4 | `4bc5bba93c4a` | `test(web): API client 동작 검증` | B | AUTH, PROTOCOL, WEB | request headers/body, response parsing, error, abort behavior를 검증합니다. |
| 5 | `353ca9a17415` | `fix(web): browser token 저장 제거` | A | AUTH, PROTOCOL, REALTIME | browser-managed durable token을 제거하고 cookie-only HTTP 및 one-time WebSocket ticket 경계로 전환합니다. |
| 6 | `2aa5fbca9890` | `test(web): cookie 기반 API 경계 검증` | B | AUTH, REALTIME, WEB | cookie-only와 runtime-validated browser API 경계를 확장 검증합니다. |

## 1. 공통 adapter가 만든 초기 일관성과 초기 위험

### `20618b30eda9` — 인증·read helper의 단일 진입점

`apps/web/src/lib/api.ts`가 API base URL, token helper, generic `apiFetch<T>`와 주요 endpoint helper를 소유합니다. 요청은 `credentials: "include"`를 사용하면서도 `localStorage` token이 있으면 Bearer header를 추가합니다.

초기 흐름은 개념적으로 다음과 같습니다.

```ts
const token = getToken();
const response = await fetch(`${API_BASE}${path}`, {
  ...init,
  credentials: "include",
  headers: {
    "Content-Type": "application/json",
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
    ...init.headers
  }
});
return response.json() as Promise<T>;
```

공통화 자체는 의미가 있습니다. login, `/me`, lobby, dashboard, leaderboard, tournament가 동일한 base URL·credential·error path를 사용합니다. 그러나 generic `T`는 compile-time 약속일 뿐 network payload를 검증하지 않습니다. 또한 durable credential을 JavaScript가 읽을 수 있는 storage에 두고, GET처럼 body가 없는 요청에도 JSON content type을 붙입니다.

`getMe`는 실패를 `null`로 바꾸고, 다른 read helper는 catch에서 sample data를 반환합니다. 이 정책은 adapter가 transport failure와 정상 domain value를 구분하지 못하게 합니다. sample fallback의 화면 영향은 Thread 03에서 다룹니다.

### `bfae9539cfe5` — mutation endpoint 확장

같은 adapter 위에 profile 조회, 친구 요청, tournament 참가, admin 상태 변경이 추가됩니다. 화면은 URL·method·body serialization을 직접 만들지 않고 endpoint helper를 호출할 수 있게 됩니다.

이 커밋의 역할은 새 신뢰 경계를 만들기보다 기존 경계를 더 많은 caller가 공유하게 하는 것입니다. 따라서 unchecked response, storage token, 항상 붙는 JSON header라는 초기 가정도 함께 전파됩니다. 후속 fix가 중요한 이유는 이 adapter가 이미 read와 mutation 양쪽의 공통 dependency가 되었기 때문입니다.

## 2. request metadata를 실제 body와 일치시키기

### `177fa0b8502a` — body가 있을 때만 JSON 선언

초기 구현은 public GET에도 `Content-Type: application/json`을 붙였습니다. 수정은 object literal merge 대신 `Headers`를 사용해 caller header를 먼저 보존하고, body가 있을 때만 content type을 보충합니다.

```ts
const headers = new Headers(init.headers);
if (init.body && !headers.has("content-type")) {
  headers.set("content-type", "application/json");
}
if (token && !headers.has("authorization")) {
  headers.set("authorization", `Bearer ${token}`);
}
```

이 결정에는 두 가지 효과가 있습니다.

- body 없는 GET·DELETE는 JSON representation을 보낸다고 거짓 선언하지 않습니다.
- caller가 `text/plain` 같은 명시적 content type을 준 경우 adapter가 덮어쓰지 않습니다.

이 시점에도 token과 response cast는 그대로입니다. 작은 fix를 credential·runtime validation 수정과 섞지 않은 점이 commit 경계를 명확하게 합니다.

### `4bc5bba93c4a` — 초기 adapter 계약을 테스트로 고정

Vitest가 token storage, authenticated JSON request, public GET, explicit content type, error body, helper response envelope를 검증합니다. 이 테스트는 당시 구현의 계약을 정확히 기록하지만, 이후 cookie-only 전환에서는 token storage 테스트 자체가 삭제됩니다.

중요한 점은 테스트가 영구적인 설계를 선언하는 것이 아니라 **그 SHA의 실제 계약과 회귀 범위**를 고정한다는 것입니다. TypeScript generic cast가 malformed success payload를 거부하는지는 이 커밋에서 검증되지 않습니다.

## 3. `353ca9a17415` — credential과 응답 신뢰 경계를 함께 교정

이 A급 fix는 adapter와 WebSocket 연결 양쪽에서 browser-managed durable token을 제거합니다. 핵심 변화는 세 묶음입니다.

### 3.1 durable credential은 cookie에만 둔다

`getToken`, `setToken`, `clearToken`과 localStorage 접근이 제거됩니다. HTTP request는 계속 `credentials: "include"`를 사용하지만 Authorization header를 만들지 않습니다.

회귀 테스트는 일부러 `window.localStorage.getItem`을 spy로 두고도 호출되지 않는지 확인합니다.

```ts
expect(init).toMatchObject({ method: "POST", credentials: "include" });
expect(headers.has("authorization")).toBe(false);
expect(getItem).not.toHaveBeenCalled();
```

이로써 JavaScript 코드가 durable session secret을 읽거나 bearer token으로 복제하는 경로가 사라집니다. HttpOnly cookie의 실제 발급·검증은 server 책임이며, 이 Thread는 브라우저 consumer 경계만 다룹니다.

### 3.2 성공 응답도 runtime schema를 통과한다

`apiFetch`는 단순 generic 대신 shared schema를 인자로 받습니다. `response.json() as T`가 아니라 parse 결과를 schema로 검증한 값만 반환합니다.

```ts
await apiFetch("/resource", okResponseSchema, {
  method: "POST",
  body: JSON.stringify({ value: 1 })
});
```

이 변경은 TypeScript type과 network trust를 분리합니다.

- TypeScript는 caller와 renderer 사이의 compile-time contract입니다.
- schema parse는 server가 보낸 runtime data가 그 contract를 실제로 만족하는지 확인합니다.

성공 status라도 malformed payload면 정상 값으로 전달되지 않습니다. endpoint helper는 각자 `userResponseSchema`, `lobbyResponseSchema`, `wsTicketResponseSchema`처럼 구체적인 schema를 선택합니다.

### 3.3 오류와 session expiry를 구조화한다

non-2xx 응답은 가능한 경우 server error envelope를 읽어 `ApiError`로 바꿉니다. status, code, message, requestId, fieldErrors가 한 경계에 모입니다. malformed error body도 공통 `ApiError` 바깥으로 새지 않도록 fallback message와 request ID header를 사용합니다.

401은 browser-wide session expiry event로 이어집니다. 이 Thread에서는 event 발행까지만 다루며, private cache를 제거하는 구체적 정책은 Thread 06의 `expireSession`이 소유합니다.

### 3.4 WebSocket은 one-time ticket으로 연다

HomePage와 PlayPage는 localStorage token을 URL에 넣는 대신 먼저 `requestWsTicket(signal)`을 호출합니다.

```ts
const controller = new AbortController();
const { ticket, protocolVersion } = await requestWsTicket(controller.signal);
const socket = new WebSocket(
  `${WS_URL}?ticket=${encodeURIComponent(ticket)}&v=${protocolVersion}`
);
```

연결 교체·unmount 시 pending ticket request를 abort하고, 이미 만들어진 socket handler를 해제한 뒤 OPEN/CONNECTING socket을 닫습니다. durable credential은 cookie-authenticated HTTP request에만 사용되고, WebSocket URL에는 짧은 수명의 일회용 값만 노출됩니다.

이 커밋은 여러 화면과 test를 함께 수정하지만, 이 Thread가 다루는 범위는 다음뿐입니다.

- token storage/Authorization 제거
- cookie 포함 HTTP
- success/error runtime validation
- session-expiry event
- AbortSignal을 받는 ticket request
- ticket 기반 WebSocket URL과 취소/정리

관리 화면 helper 교체나 화면별 presentation 변화는 해당 화면 Thread의 관심사입니다.

## 4. `2aa5fbca9890` — 새 경계가 실제로 무엇을 증명하는가

후속 테스트는 cookie-only와 schema 경계를 넓게 고정합니다.

- localStorage를 읽지 않고 Authorization을 추가하지 않는가
- caller의 explicit content type을 유지하는가
- body 없는 요청에 JSON content type을 붙이지 않는가
- structured error가 `ApiError`의 필드로 보존되는가
- malformed error도 공통 오류 boundary 안에 남는가
- malformed success payload가 schema error로 거부되는가
- endpoint helper가 올바른 shared schema를 선택하는가
- `AbortSignal`이 fetch와 ticket helper까지 전달되는가
- `requestWsTicket`이 validated ticket response만 반환하는가

이 테스트가 보장하지 않는 것도 분명합니다.

- cookie의 `HttpOnly`, `Secure`, `SameSite` 실제 속성은 server/integration 영역입니다.
- browser가 실제로 401 event 이후 모든 private 화면을 즉시 비우는지는 query cache Thread의 테스트가 담당합니다.
- WebSocket ticket이 server에서 일회성으로 소비되는지는 server-side access subsystem의 책임입니다.
- 이 작업에서는 테스트를 실행하지 않았으므로 source상 assertion과 production path만 검사했습니다.

## 최종 adapter 계약

| 입력/상황 | 최종 동작 | caller가 받는 것 |
| --- | --- | --- |
| body 없는 request | content type 자동 추가 없음 | validated success 또는 `ApiError` |
| JSON body request | caller가 지정하지 않았을 때만 JSON content type | validated success 또는 `ApiError` |
| durable session | `credentials: "include"`로 cookie 전달 | JS에는 token 노출 없음 |
| 2xx + malformed JSON shape | shared schema parse 실패 | 성공 값으로 취급하지 않음 |
| non-2xx structured envelope | fields를 보존한 `ApiError` | status/code/requestId/fieldErrors |
| 401 | `ApiError`와 session-expiry event | cache owner가 후속 정리 가능 |
| WebSocket 연결 요청 | cookie HTTP로 one-time ticket 발급 | `ticket + protocolVersion` |
| connection 교체/unmount | pending ticket abort, handler detach, socket close | stale completion 무시 기반 |

## 범위

이 Thread는 transport adapter와 credential trust boundary만 다룹니다. 다음은 다른 Thread의 책임입니다.

- 각 route의 loading/error/empty 화면과 sample 제거 — Thread 03
- `QueryClient`의 private/public cache 분류와 invalidation — Thread 06
- `GameSocketClient`의 generation guard·reconnect timer·message lifecycle — Thread 04
- guest session/ticket server producer와 rate limit — 다른 카테고리의 server-side subsystem

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. build·Vitest·browser test는 이 환경에서 실행하지 않았습니다.
