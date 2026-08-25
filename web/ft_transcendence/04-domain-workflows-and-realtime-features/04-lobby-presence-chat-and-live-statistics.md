# 로비의 저장 이력과 실시간 상태를 합치기

원문 Thread: `Lobby presence, chat, and live statistics`

## 이 Thread가 다루는 문제

로비 한 화면에는 수명이 다른 데이터가 섞입니다.

- chat history는 재시작 뒤에도 남아야 하므로 repository가 소유합니다.
- 현재 WebSocket 접속자·queue·active room은 GameHub process memory가 소유합니다.
- 평균 대기시간은 최근 queue 사건에서 수집한 bounded sample입니다.
- 브라우저는 HTTP snapshot을 먼저 받고 이후 WebSocket event로 일부를 갱신합니다.

초기 구현은 DB의 active 사용자 목록을 online으로 표시하고, 서버 실패 시 sample 로비를 남기며, socket이 open된 직후 HTTP presence에도 즉시 보일 것이라 가정했습니다. 이 Thread는 각 데이터의 owner를 분리하고 eventual visibility를 UI와 test에 반영합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a6fa5a187eec` | `feat(db): 채팅 메시지 저장 구현` | B | REALTIME, PERSISTENCE | sender·scope·room·body를 가진 durable chat history를 repository에 추가한다. |
| 2 | `dabd8d5c2a49` | `feat(game): 실시간 경기 채팅 전달` | B | PROTOCOL, REALTIME, PERSISTENCE | WebSocket chat을 저장한 뒤 room 또는 전체 client에 broadcast한다. |
| 3 | `1d9aa3902614` | `feat(lobby): 실시간 로비 지표 API 추가` | B | REALTIME | GameHub의 client·queue·room·wait sample로 live statistics를 만든다. |
| 4 | `de9a173e6eb1` | `feat(chat): 쓰기 가능한 로비 채팅 API 추가` | B | REALTIME, PERSISTENCE | 인증된 HTTP lobby-chat write endpoint를 추가한다. |
| 5 | `e0ef3fec89a6` | `feat(chat): 로비 채팅 입력 화면 추가` | B | WEB | controlled form과 HTTP 응답 기반 local append를 연결한다. |
| 6 | `4f9b3b312d0e` | `fix(lobby): 로비 상태 표현 개선` | B | REALTIME | 고정 활동량·대기시간 문구를 실제 server stats로 교체한다. |
| 7 | `8ce1199ffd12` | `fix(api): body 없는 로비 채팅 요청 처리` | B | - | missing body를 빈 객체로 정규화한 뒤 validation한다. |
| 8 | `8078ac6f92ba` | `test(app): 실시간 지표·채팅·경기 기록 검증` | B | REALTIME, PERSISTENCE, WEB | API·memory repository·E2E·smoke 범위에서 지표와 채팅 저장을 검증한다. |
| 9 | `cd3787eefd6a` | `feat(chat): 로비 채팅과 접속 상태 실시간 반영` | B | AUTH, REALTIME, WEB | HTTP snapshot에 WebSocket chat/presence event를 합치고 chat HTTP fallback을 둔다. |
| 10 | `8debb1ea3ad3` | `feat(lobby): 연결 중인 WebSocket 사용자 목록 추가` | B | REALTIME | DB active 계정이 아니라 GameHub client map을 online authority로 사용한다. |
| 11 | `c3ff9ed2402f` | `test(lobby): WebSocket 사용자 목록 검증` | B | REALTIME, PERSISTENCE, TEST | socket이 없으면 onlinePlayers도 비어야 함을 검증한다. |
| 12 | `23a978879b81` | `test(smoke): WebSocket 접속 상태 반영 대기` | B | REALTIME, OPERATIONS, TEST | socket open 직후 즉시 단정하지 않고 bounded polling으로 eventual visibility를 기다린다. |

## 1. durable chat와 process-local statistics

### `a6fa5a187eec` — chat history 저장

repository에 `createChatMessage`와 `listLobbyChat`가 추가됩니다. PostgreSQL은 sender user와 join해 최신 20개를 내림차순으로 고른 뒤 reverse하여 화면에는 오래된 것부터 보여 줍니다. memory 구현도 lobby scope를 filter한 뒤 마지막 20개를 같은 순서로 반환합니다.

```text
DB selection: newest 20
API display:  그 20개 안에서 oldest → newest
```

sender·body·timestamp를 가진 durable record가 생겼지만, scope-room 조합의 강제와 room authorization은 아직 없습니다. 그 보안 경계는 Thread 05가 담당합니다.

### `dabd8d5c2a49` — WebSocket command를 저장한 뒤 전달

GameHub는 `chat.send`를 받으면 repository에 먼저 저장하고 반환된 `ChatMessage`를 broadcast합니다. match scope면 room audience, lobby scope면 모든 client가 대상입니다.

이 시점에는 요청자가 그 match room의 좌석인지 확인하지 않았습니다. 즉 persistence와 broadcast 연결은 생겼지만 authorization은 미완성입니다. 후속 `7759eef59b67`이 저장 전 seat check를 추가합니다.

### `1d9aa3902614` — live stats의 owner

queue entry에 `queuedAt`이 추가되고 GameHub가 다음 값을 계산합니다.

- `onlinePlayers`: 현재 client 수
- `queuedPlayers`: queue 길이
- `activeRooms`: room 수
- playing count
- `averageWaitSeconds`: 최근 최대 20개 wait sample의 평균, 표본이 없으면 `null`

이 값들은 DB 통계가 아닙니다. 현재 process가 알고 있는 연결과 room에 대한 snapshot입니다. 서버 재시작 시 wait sample은 사라지고, 여러 GameHub instance를 합치는 로직도 이 commit에는 없습니다.

## 2. HTTP 쓰기와 첫 브라우저 연결

### `de9a173e6eb1` — 인증된 lobby write endpoint

`POST /chat/lobby`가 현재 사용자를 확인하고 body를 trim한 뒤 비어 있거나 240자를 넘는 입력을 거부합니다. 성공하면 repository record를 반환합니다. WebSocket이 없는 환경에서도 로비 채팅을 쓸 수 있는 경로가 생깁니다.

### `e0ef3fec89a6` — controlled input과 local append

로비 form은 HTTP helper를 호출하고 성공 응답을 최근 20개 local state에 붙입니다. 이 시점의 갱신은 sender 자신에게만 즉시 보이며, 다른 client가 받는 realtime event 연결은 아직 없습니다.

초기 state에 sample users/chat가 남아 있어 request failure가 실제 로비처럼 보일 수 있다는 문제도 유지됩니다.

이 sample fallback의 제거는 여러 로그인 화면을 한꺼번에 교정한 cross-cutting commit `be31566ac0fd`가 담당합니다. 해당 commit은 profile/dashboard/ranking Thread가 주 소유하므로 이 문서의 commit map에 중복 포함하지 않았지만, 로비의 최종 화면도 실패 시 sample data를 정상 사실처럼 유지하지 않고 명시적 오류 상태를 표시합니다.

### `4f9b3b312d0e` — 고정 문구를 server stats로 교체

“주간 활동”이나 항상 30초 같은 고정 표시를 제거하고 `/lobby`가 준 `onlinePlayers`, `queuedPlayers`, `averageWaitSeconds`를 사용합니다. 표본이 없으면 “대기 없음”으로 표시합니다.

이 수정도 online user의 원천까지는 바로잡지 않습니다. 당시 `/lobby.onlinePlayers`는 repository의 active 사용자 목록이었습니다.

### `8ce1199ffd12` — body 없음은 validation 실패여야 한다

Fastify request body가 `undefined`인 경우 곧바로 속성을 읽어 예외가 나던 경로를 `request.body ?? {}`로 정규화합니다. 이후 optional body를 꺼내 기존 trim/length validation으로 보냅니다.

작은 commit이지만 malformed request가 내부 TypeError가 아니라 의도한 4xx validation 결과로 수렴하게 합니다.

### `8078ac6f92ba` — 여러 층의 회귀를 추가

이 commit은 production behavior를 크게 바꾸지 않고 다음 증거를 추가합니다.

- API: 초기 live stats가 0/0/null인지
- API: 인증된 lobby write 뒤 `/lobby` history에서 같은 body를 읽는지
- memory repository: lobby/match chat sender와 room을 보존하는지
- E2E: 로비 입력 뒤 메시지가 보이는지
- smoke: 실제 endpoint 호출 경로에 lobby write가 포함되는지

같은 diff의 경기 기록 assertion은 최근 match 순서와 rating 반영을 보강하지만, 이 Thread에서는 chat/lobby 관련 부분만 다룹니다. 테스트는 실행하지 않았습니다.

## 3. HTTP snapshot과 WebSocket event 합치기

### `cd3787eefd6a` — 실시간 로비 연결

브라우저는 로그인 사용자가 생기면 session token으로 WebSocket을 열고 두 종류의 event를 다르게 처리합니다.

```ts
if (message.type === "chat.message"
    && message.message.scope === "lobby") {
  setChat((current) => [
    ...current
      .filter((item) => item.id !== message.message.id)
      .slice(-19),
    message.message
  ]);
}

if (message.type === "presence.changed") {
  loadLobby().catch(() =>
    setNotice("로비 지표를 갱신하지 못했습니다.")
  );
}
```

chat event에는 완성된 durable message가 있으므로 payload를 바로 합칩니다. ID filter는 HTTP fallback 응답과 나중 event가 같은 record를 두 번 넣는 것을 막습니다. presence event는 “무엇이 바뀌었다”는 신호만 주므로 `/lobby` snapshot을 다시 읽습니다.

전송도 이중 경로입니다.

- socket이 OPEN: `chat.send`
- socket이 없거나 닫힘: HTTP `POST /chat/lobby`

effect cleanup은 handler를 제거하고 CONNECTING/OPEN socket을 닫으며 ref를 비웁니다. 실시간 기능을 추가하면서 socket lifetime도 component lifetime에 묶었습니다.

## 4. online의 의미를 실제 연결로 바꾸기

### `8debb1ea3ad3` — GameHub client map이 presence authority

`/lobby`는 더 이상 `repo.listOnlineUsers()`를 호출하지 않고 `hub.onlinePlayers()`를 사용합니다. hub는 현재 clients를 user ID로 deduplicate하고 email을 제외한 public projection에 `online: true`를 붙인 뒤 rating/displayName 순으로 정렬합니다.

```ts
const users = new Map<string, PublicUser>();
for (const client of this.clients.values()) {
  const { email: _email, ...user } = client.user;
  users.set(user.id, { ...user, online: true });
}
```

한 사용자가 여러 tab/socket을 열어도 목록에는 한 번만 나타납니다. DB에 active 계정이 많아도 현재 연결이 없으면 online list는 비어 있습니다.

### `c3ff9ed2402f` — socket 없는 hub의 기준점

API test는 seed 사용자나 active 계정이 존재해도 WebSocket client가 0이면 `onlinePlayers`가 `[]`이고 stats의 online count도 0임을 확인합니다. 저장 사용자와 접속 사용자를 다시 혼동하는 회귀를 막습니다.

### `23a978879b81` — open 완료와 HTTP 가시성은 같은 순간이 아니다

smoke test는 두 socket의 `open` event 직후 `/lobby`를 한 번 읽고 실패시키던 방식을 버립니다. 최대 10초 동안 50ms 간격으로 비동기 predicate를 실행해 두 handle이 snapshot에 들어올 때까지 기다립니다.

이 변경은 무제한 sleep이 아닙니다.

- 성공 조건이 구체적입니다.
- timeout이 있어 실패가 숨지 않습니다.
- 매 시도마다 최신 HTTP snapshot을 다시 읽습니다.

WebSocket handshake 완료와 GameHub registration/broadcast/HTTP read 사이의 짧은 eventual delay를 테스트 계약에 포함한 것입니다.

## 최종 합성 모델

```text
[HTTP /lobby]
  ├─ repository.listLobbyChat()  → durable 최근 20개
  ├─ hub.onlinePlayers()        → 현재 연결 user
  └─ hub.liveStats()            → queue/room/wait sample

[WebSocket]
  ├─ chat.message     → durable message payload를 ID 중복 제거 후 append
  └─ presence.changed → /lobby snapshot 재조회

[chat send]
  ├─ WebSocket OPEN → chat.send → 저장 → broadcast
  └─ socket 없음    → HTTP write → 저장 → sender local append
```

| 데이터 | owner | 비보장 |
| --- | --- | --- |
| chat history | repository | 전체 무제한 이력이 아니라 최근 20개 |
| online users | 현재 GameHub clients | 여러 instance 통합 presence 아님 |
| queued/rooms | 현재 GameHub | process restart 뒤 유지되지 않음 |
| average wait | 최근 20개 process sample | 장기 통계나 percentile 아님 |
| browser state | HTTP snapshot + 이후 event | network 단절 중 즉시 일치 보장 없음 |

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. API/E2E/smoke test는 실행하지 않았습니다.
