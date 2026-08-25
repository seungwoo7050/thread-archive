# Thread: 게임 연결 상태기계와 transport 수명주기

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

초기 play page는 WebSocket 생성, ticket 요청, raw message parsing, room/snapshot/status state, input sequence를 한 component 안에서 직접 소유했습니다. 연결을 새로 열거나 닫을 때 어떤 callback이 아직 유효한지, 늦게 끝난 ticket request가 현재 연결을 덮을 수 있는지, 오래된 snapshot이 최신 state를 되돌릴 수 있는지 판단할 단일 경계가 없었습니다.

이 Thread는 책임을 두 축으로 분리합니다.

- **pure reducer**: connection event가 application state를 어떻게 바꾸는지 결정합니다.
- **`GameSocketClient`**: ticket request, socket, callback, generation, reconnect timer와 input sequence 같은 실제 resource를 소유합니다.

React hook과 page migration은 Thread 05에서 다룹니다. 여기서는 reducer와 transport가 스스로 지켜야 하는 규칙, 그리고 reconnect·room-scoped chat fix를 집중적으로 설명합니다.

### 최종 불변 조건

- snapshot은 `sequence`가 현재보다 클 때만 수용합니다.
- 새 연결은 기존 ticket request를 abort하고 기존 socket handler를 제거한 뒤 socket을 닫습니다.
- ticket 완료와 socket callback은 generation + socket identity가 현재 연결과 일치할 때만 효과를 냅니다.
- reconnect는 기존 room을 이어 가는 작업이며 최초 `queue.join`/`tournament.join`을 다시 보내지 않습니다.
- active room이 남아 있는 reconnect 상태에서는 새 match intent를 만들 수 없습니다.
- match chat은 `scope === "match"`이고 `roomId === activeRoomId`일 때만 reducer에 들어옵니다.

### 교차 Thread 주의

`4f5199097284`는 같은 commit에서 transport reconnect와 guest lobby recovery를 함께 수정합니다. 이 문서는 `GameSocketClient`, reducer, hook, play page의 연결 수명주기만 다룹니다. `HomePage`와 `demoPolicy` 변경은 Thread 08에서 설명합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1bf328ce92a5` | `refactor(web): game input 직렬화 경계 분리` | B | WEB | keyboard state를 protocol command 방향으로 바꾸는 pure helper를 분리합니다. |
| 2 | `ffcbdd403a06` | `refactor(web): game connection 상태 reducer 분리` | B | REALTIME, WEB | browser game connection state/action vocabulary를 정의합니다. |
| 3 | `d8311e74373e` | `refactor(web): game connection 전이 규칙 완성` | A | REALTIME, WEB, OPERATIONS | open/matched/snapshot/reconnecting/finished/failed transition을 reducer에 완성합니다. |
| 4 | `bfded21cd1ac` | `refactor(web): GameSocketClient 연결 수명주기 분리` | A | AUTH, REALTIME, WEB | ticket request, socket replacement, close/teardown을 transport-neutral client가 소유합니다. |
| 5 | `92ad229a23d3` | `refactor(web): GameSocketClient 메시지 처리를 분리` | A | AUTH, PROTOCOL, SIMULATION | message parse/dispatch까지 client로 이동해 ticket-to-socket lifecycle을 통합합니다. |
| 6 | `b5691b01a09b` | `test(web): game connection lifecycle 검증` | A | AUTH, PROTOCOL, REALTIME | connection replacement, teardown, stale event, command lifecycle을 deterministic test로 고정합니다. |
| 7 | `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | AUTH, REALTIME, WEB | fresh ticket으로 existing room을 복구하고 original matchmaking intent를 재전송하지 않습니다. |
| 8 | `85edd6d1e26a` | `fix(web): 현재 경기방의 채팅만 표시` | B | REALTIME, WEB | inbound match chat을 current active room predicate로 filtering합니다. |
| 9 | `02775797ab63` | `test(web): 매치 채팅 room filtering 검증` | B | REALTIME, WEB, TEST | match chat room filter의 positive/negative boundary를 검증합니다. |

## 상태와 입력 어휘를 side effect에서 떼어내기

### `1bf328ce92a5` — keyboard 의미를 pure helper로

`directionForKey`는 ArrowUp/W를 `-1`, ArrowDown/S를 `1`, 그 밖을 `null`로 바꿉니다. `isEditableTarget`은 input, textarea, select, contenteditable에서 game key가 작동하지 않도록 판단합니다.

이 커밋은 listener나 socket을 바꾸지 않습니다. 목적은 DOM event handler 내부에 있던 “어떤 key가 어떤 command 의미인가”를 transport와 독립된 함수로 만드는 것입니다. 실제 page adoption과 neutral input cleanup은 Thread 05에서 이어집니다.

### `ffcbdd403a06` — reducer vocabulary만 먼저 정의

room, snapshot, opponent, messages, notice, status, last snapshot sequence를 하나의 `GameConnectionState`에 모으고 action union을 정의합니다. 그러나 이 SHA의 reducer는 아직 state를 그대로 반환하는 skeleton입니다.

이 중간 단계는 의도적입니다. component의 여러 `useState`를 한 번에 제거하지 않고 먼저 “연결에서 일어날 수 있는 사건”을 이름으로 고정합니다. 실제 transition은 다음 commit에 들어갑니다.

## `d8311e74373e` — 연결 상태기계 완성

reducer는 resource를 직접 만지지 않으면서 event와 기존 state만으로 다음 state를 계산합니다.

```ts
case "snapshotReceived": {
  if (action.snapshot.sequence <= state.lastSnapshotSequence) return state;
  const status = statusForSnapshot(action.snapshot);
  return {
    ...state,
    status,
    roomId: action.snapshot.roomId,
    snapshot: action.snapshot,
    lastSnapshotSequence: action.snapshot.sequence,
    notice: noticeForStatus(status)
  };
}
```

### 각 action이 보존하거나 초기화하는 것

| Action | 핵심 전이 | 의도 |
| --- | --- | --- |
| `connectStarted` | `initialGameConnectionState`에서 다시 시작, status `connecting` | 이전 room/snapshot/message/sequence 제거 |
| `socketOpened` | status `matching`, caller notice | 최초 intent가 전송될 transport 준비 |
| `matched` | room/opponent 설정, `waitingReady` | room identity 확정 |
| `snapshotReceived` | 더 최신 sequence만 저장, phase에서 status 파생 | server snapshot을 단일 authority로 사용 |
| `gameFinished` | status `finished`, room null, 마지막 snapshot phase만 finished로 표시 | 새 match 가능 terminal state |
| `socketClosed` | room 있으면 `reconnecting`, 없으면 `failed` | “진행 중 room continuation”과 “연결 실패” 구분 |
| `failed` | status `failed`, visible notice | transport/schema failure 표면화 |

`gameFinished`에서 마지막 snapshot을 보존하는 것은 마지막 점수·장면을 표시하기 위한 presentation 선택입니다. room ID는 비워 새 match가 가능해집니다.

가장 중요한 방어는 sequence 비교입니다. WebSocket 자체는 전송 순서를 보존하지만 reconnect나 비동기 처리 경계에서 stale event가 application state에 늦게 도달할 수 있습니다. 같거나 작은 sequence를 state identity 그대로 반환하면 rendering과 interpolation buffer가 과거로 되돌아가지 않습니다.

## transport resource owner 만들기

### `bfded21cd1ac` — replacement를 먼저 정의

`GameSocketClient`는 browser WebSocket과 같은 최소 interface를 받습니다. 실제 browser 구현과 fake socket test가 같은 client를 사용할 수 있도록 `socketFactory`와 `ticketProvider`를 주입합니다.

처음 도입된 핵심은 `replaceConnection`입니다.

```ts
private replaceConnection(): number {
  this.generation += 1;
  this.ticketRequest?.abort();
  this.ticketRequest = null;

  const socket = this.socket;
  this.socket = null;
  if (socket) {
    socket.onopen = null;
    socket.onmessage = null;
    socket.onclose = null;
    socket.onerror = null;
    if (socket.readyState === CONNECTING || socket.readyState === OPEN) socket.close();
  }
  this.inputSequence = 0;
  return this.generation;
}
```

순서가 중요합니다.

1. generation을 먼저 올려 이미 시작된 async completion을 stale로 만듭니다.
2. pending ticket request를 abort합니다.
3. current socket reference를 먼저 비웁니다.
4. callback을 떼어 explicit close가 application close event로 오인되지 않게 합니다.
5. 연결별 input sequence를 0으로 되돌립니다.

이 SHA에서는 connect/message 처리 전체가 아직 없습니다. resource replacement와 current identity predicate를 먼저 만든 준비 commit입니다.

### `92ad229a23d3` — ticket부터 parsed event까지 한 lifecycle로

`connect`는 replacement로 새 generation을 얻고, AbortController를 ticket provider에 넘긴 뒤 응답이 현재 generation인지 다시 확인합니다.

```ts
const generation = this.replaceConnection();
const controller = new AbortController();
this.ticketRequest = controller;

const ticket = await this.options.ticketProvider(controller.signal);
if (controller.signal.aborted || generation !== this.generation) return;
```

socket URL은 ticket과 protocol version만 포함합니다. callback마다 `isCurrent(socket, generation)`을 통과해야 하며, message는 shared `parseServerEvent`를 거칩니다.

```ts
socket.onmessage = (event) => {
  if (!this.isCurrent(socket, generation)) return;
  try {
    if (typeof event.data !== "string") throw new Error(...);
    handlers.onEvent(parseServerEvent(event.data));
  } catch (error) {
    handlers.onFailure(error);
  }
};
```

`sendDirection`은 연결별 monotonic `inputSequence`를 증가시키고 version, room identity, direction을 함께 직렬화합니다.

```ts
{
  v: 1,
  type: "game.input",
  roomId,
  inputSeq: ++this.inputSequence,
  direction
}
```

client는 command가 socket에 쓰였는지 boolean/null로 알려 주지만 application status를 직접 결정하지 않습니다. 상태 전이는 reducer/hook의 책임입니다.

## `b5691b01a09b` — resource와 state를 fake로 직접 관찰

테스트는 실제 network 대신 injected ticket provider와 `FakeSocket`을 사용해 비동기 순서를 결정적으로 만듭니다.

### 연결 교체

첫 `connect`의 ticket promise는 pending 상태로 두고 두 번째 `connect`를 시작합니다. 기대 결과는 다음과 같습니다.

- 첫 signal은 aborted
- ticket provider는 두 번 호출
- socket은 두 번째 ticket에서 하나만 생성
- URL은 `?ticket=<...>&v=1`

이는 “늦게 완료된 첫 request가 socket을 만들지 않는다”는 generation/abort 경계를 직접 확인합니다.

### protocol parse

versioned `queue.matched`는 `onEvent`로 한 번 전달되고 version이 빠진 payload는 `onFailure`로 갑니다. TypeScript cast가 아니라 shared parser가 runtime trust boundary임을 검증합니다.

### monotonic input

세 direction command가 inputSeq 1, 2, 3으로 전송되는지 payload 전체를 비교합니다.

### reducer

연결 상태가 명시적 status union 안에 머무는지, same/older snapshot에서 state object 자체를 그대로 반환하는지, `connectStarted`가 room/snapshot/messages/sequence를 초기화하는지 확인합니다.

이 테스트는 browser WebSocket implementation이나 실제 ticket server를 검증하지 않습니다. client의 deterministic lifecycle contract를 검증합니다.

## `4f5199097284` — reconnect는 새 match가 아니다

기존 socket close는 room이 있으면 reducer를 `reconnecting`으로 바꾸지만 실제 transport 복구는 없었습니다. 사용자가 다시 queue button을 누르면 새 match intent를 보내 기존 room과 중복될 수도 있었습니다.

### reconnect resource

client에 다음 state가 추가됩니다.

- `reconnectTimer`
- `reconnectDeadlineMs`
- `reconnectAttempts`
- 15초 reconnect window
- 250ms에서 시작해 최대 2초인 exponential backoff

최초 `connect`는 `openSocket(generation, initialEvent, handlers, false)`를 호출합니다. reconnect timer는 같은 generation에서 `openSocket(generation, null, handlers, true)`를 호출합니다.

핵심은 `initialEvent: null`입니다.

```ts
socket.onopen = () => {
  if (!this.isCurrent(socket, generation)) return;
  handlers.onOpen(reconnected);
  if (initialEvent) socket.send(JSON.stringify(initialEvent));
};
```

새 ticket을 받아 새 socket을 열지만 최초 `queue.join` 또는 `tournament.join`을 다시 보내지 않습니다. server가 cookie/ticket identity로 기존 room state를 복구하도록 기다립니다.

### reconnect 여부 결정

hook의 `onClosed`는 현재 room이 있는지 반환합니다. client는 `true`일 때만 reconnect를 schedule합니다. `stateRef`를 사용해 callback이 stale render state가 아니라 최신 room ID를 읽습니다.

### duplicate intent 차단

```ts
export function canStartNewMatch(state: GameConnectionState): boolean {
  return state.roomId === null
    && ["idle", "finished", "failed"].includes(state.status);
}
```

hook의 `connect`와 play button disabled가 같은 predicate를 사용합니다. room이 남은 `reconnecting` 상태는 새 match를 시작할 수 없습니다.

### 명시적 close와 remote close

`replaceConnection`은 reconnect timer를 clear하고 deadline/attempt를 초기화합니다. 따라서 unmount나 새 의도에 의한 explicit close는 자동 reconnect를 남기지 않습니다. remote close만 handler 반환값을 통해 continuation 여부를 결정합니다.

이 commit의 HomePage route recovery와 transient result notice는 Thread 08 범위입니다.

## room-scoped chat boundary

### `85edd6d1e26a` — predicate를 별도 함수로

초기 hook은 모든 `chat.message`를 current game message list에 넣었습니다. lobby message나 과거 room의 delayed message도 보일 수 있었습니다.

```ts
export function isChatForActiveRoom(
  message: ChatMessage,
  activeRoomId: string | null
): boolean {
  return activeRoomId !== null
    && message.scope === "match"
    && message.roomId === activeRoomId;
}
```

hook은 reducer dispatch 전에 이 predicate를 적용합니다. filtering을 reducer 뒤에서 하지 않으므로 잘못된 message가 state에 잠시라도 들어가지 않습니다.

### `02775797ab63` — positive/negative matrix

focused test는 현재 room의 match chat만 true이고 다음은 false임을 고정합니다.

- active room 없음
- `scope: "lobby"`
- 다른 room ID
- 이전 room에서 늦게 도착한 match message

이 test는 server가 routing을 정확히 한다는 것을 증명하지 않습니다. browser consumer가 자신의 active-room invariant를 독립적으로 지킨다는 것을 증명합니다.

## 최종 ownership

| 자원/상태 | Owner | 종료 또는 교체 조건 |
| --- | --- | --- |
| connection status, room, snapshot, notice, messages | reducer state | action이 명시적으로 reset/replace |
| ticket request | `GameSocketClient.ticketRequest` | resolve/reject, abort, replacement |
| socket + callbacks | `GameSocketClient.socket` | remote close, explicit replacement/close |
| stale completion 식별 | `generation` + socket identity | replacement마다 generation 증가 |
| input sequence | `GameSocketClient` connection instance | direction send마다 증가, replacement에서 0 |
| reconnect timer/deadline | `GameSocketClient` | 성공, deadline, replacement/explicit close |
| new-match eligibility | pure `canStartNewMatch` | room/status 조합에서 계산 |
| match chat acceptance | `isChatForActiveRoom` | scope + current room match |

## 최종 흐름

```text
[connect(initial intent)]
  -> replace old connection / generation++ / abort / detach / close
  -> request one-time ticket(signal)
  -> generation 확인
  -> socket 생성(ticket, protocol version)
  -> open: 최초 연결이면 initial intent 1회 전송
  -> message: current socket 확인 -> shared parser -> reducer action
  -> close:
       room 없음 -> failed
       room 있음 -> reconnecting -> timer -> fresh ticket -> no initial intent

[snapshot]
  -> parse
  -> sequence > last only
  -> reducer accepts and derives phase/status

[chat]
  -> scope=match && message.roomId=current room
  -> reducer append bounded history
```

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. Vitest와 browser runtime은 이 환경에서 실행하지 않았습니다.
