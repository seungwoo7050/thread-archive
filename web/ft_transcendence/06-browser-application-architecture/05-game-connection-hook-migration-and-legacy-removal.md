# Thread: `useGameConnection` 전환과 중복 owner 제거

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

Thread 04에서 reducer와 `GameSocketClient`가 만들어졌다고 해서 play page가 자동으로 새 구조를 사용하는 것은 아닙니다. 실제 migration 동안에는 다음 두 구현이 잠시 공존합니다.

- 기존 page-local socket, room/snapshot/status, 50ms input loop, command functions
- 새 `useGameConnection`의 reducer state와 command surface

이 Thread의 핵심은 “한 번에 모두 지우기”가 아니라 **replacement를 먼저 연결하고 caller를 옮긴 뒤, 더 이상 참조되지 않는 legacy owner를 작은 commit으로 제거하는 순서**입니다. 최종 상태에서 page는 presentation과 user intent만 소유하고 raw transport·protocol state는 hook 아래로 사라집니다.

### migration 불변 조건

- 새 owner가 caller에 연결되기 전에 기존 owner를 삭제하지 않습니다.
- render state와 command가 서로 다른 connection owner를 사용하지 않습니다.
- 전환이 끝나면 raw WebSocket, ticket request, parse callback, room/snapshot/status duplicate state가 page에 남지 않습니다.
- keyboard/touch 입력은 방향 변화만 전송하고 release·blur·hidden·pointer 종료에서 `0`으로 복귀합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9d70eb12e1d7` | `refactor(web): game connection hook 상태 연결` | B | AUTH, REALTIME, TOURNAMENT | client와 reducer를 React hook 뒤에서 조합합니다. |
| 2 | `748079c73eea` | `refactor(web): game connection hook 명령 연결` | B | PROTOCOL, SIMULATION, REALTIME | ready/input/chat/pause/resume command surface를 hook에 추가합니다. |
| 3 | `c33412d639c5` | `refactor(play): connection hook 전환 경계 준비` | B | PERSISTENCE, WEB | legacy path를 유지한 채 hook을 play page boundary에 도입합니다. |
| 4 | `898d0884ee37` | `refactor(play): 자동 경기 진입을 connection hook으로 전환` | B | REALTIME, TOURNAMENT, WEB | URL-driven queue/AI/tournament intent를 hook으로 전환합니다. |
| 5 | `9a67234633b4` | `refactor(play): 경기 상태와 명령을 connection hook에 연결` | B | REALTIME, PERSISTENCE, WEB | render state와 ready/chat/pause command를 hook output으로 전환합니다. |
| 6 | `1ae6fa7836d8` | `feat(play): keyboard와 touch paddle 입력 연결` | B | PROTOCOL, SIMULATION, REALTIME | keyboard와 mobile pointer control을 transition-based paddle command에 연결합니다. |
| 7 | `31b5122add6f` | `refactor(play): legacy paddle input loop 제거` | B | SIMULATION, REALTIME, WEB | component-local keyboard state와 50ms command loop를 제거합니다. |
| 8 | `fa35e8d15b4e` | `refactor(play): legacy WebSocket lifecycle 제거` | B | AUTH, PROTOCOL, REALTIME | inline socket create/listener/close path를 제거합니다. |
| 9 | `365d66c72343` | `refactor(play): legacy 경기 명령 제거` | B | PROTOCOL, REALTIME, WEB | page-local ready/chat/pause/resume/close command path를 제거합니다. |
| 10 | `06664abbae7f` | `refactor(play): legacy socket 상태 제거` | B | AUTH, PROTOCOL, REALTIME | duplicate socket/game-state field와 old API import를 제거합니다. |
| 11 | `9faada0df1d7` | `refactor(play): connection hook 전환 마무리` | A | PROTOCOL, REALTIME, TOURNAMENT | page가 hook의 단일 state/command surface만 소비하도록 migration을 끝냅니다. |

## hook이 제공해야 할 두 표면

### `9d70eb12e1d7` — state adapter

`useGameConnection`은 `useReducer`, memoized `GameSocketClient`, server event mapping, unmount cleanup을 조합합니다.

```ts
const [state, dispatch] = useReducer(
  gameConnectionReducer,
  initialGameConnectionState
);

const client = useMemo(() => new GameSocketClient({
  url: WS_URL,
  ticketProvider: requestWsTicket,
  socketFactory: (url) => new WebSocket(url) as unknown as GameWebSocket
}), []);
```

`queue.matched`, `game.snapshot`, `game.finished`, `chat.message`, `error`를 reducer action으로 바꾸고 `presence.changed`는 game state와 무관하므로 무시합니다. `connectQueue`와 `connectTournament`는 initial protocol intent와 open notice를 구성합니다. cleanup effect는 component unmount에서 `client.close()`를 호출합니다.

이 SHA의 hook은 아직 ready/chat/pause/input command를 제공하지 않습니다. state lifecycle을 먼저 React boundary에 연결한 commit입니다.

### `748079c73eea` — command adapter

hook이 current reducer state를 읽어 command precondition을 적용합니다.

```ts
const ready = useCallback(() => {
  if (!state.roomId) return false;
  const sent = client.send({ v: 1, type: "game.ready", roomId: state.roomId });
  if (sent) dispatch({ type: "readySent" });
  return sent;
}, [client, state.roomId]);
```

- chat은 body를 trim하고 room이 없거나 빈 값이면 보내지 않습니다.
- pause/resume은 status가 `playing`/`paused`일 때만 각각 전송합니다.
- direction은 current room ID와 함께 client에 위임합니다.

이제 page가 raw socket 없이 state와 command를 모두 사용할 수 있지만 아직 caller는 기존 구현을 사용합니다.

## replacement-first 전환

### `c33412d639c5` — 새 owner를 page에 주입

PlayPage가 `useGameConnection`, `directionForKey`, `isEditableTarget`을 import하고 hook instance를 만듭니다. 그러나 기존 `snapshot`, `roomId`, `status`, `messages`, `socketRef`, `ticketRequestRef`, sequence refs와 함수는 그대로 남습니다.

이 commit은 사용자 동작을 바꾸지 않는 migration seam입니다. 새 hook을 먼저 page lifecycle 안에 두어 후속 commit이 caller를 하나씩 옮길 수 있게 합니다. 이 단계에서 두 owner가 실제 command를 동시에 보내지 않도록 새 hook output을 아직 사용하지 않는 것이 중요합니다.

### `898d0884ee37` — URL-driven 최초 intent 이동

`mode=queue|ai`, `tournamentMatchId` query parameter를 읽는 auto-entry effect가 기존 `openGameSocket` 대신 hook의 `connectQueue`/`connectTournament`를 호출합니다. `autoStartedRef`는 rerender에서 같은 URL intent가 반복 전송되지 않게 합니다.

manual button과 일부 render/commands는 아직 legacy path에 남아 있으므로 migration은 완료되지 않았습니다. 이 commit은 “route가 연결을 시작하는 경계”만 이동합니다.

### `9a67234633b4` — render state와 주요 commands 이동

play page가 hook의 `state`를 기준으로 snapshot, room, status/notice, messages, opponent, canReady/canChat/canPause/canResume를 계산합니다. ready, chat, pause/resume도 hook command를 사용합니다.

이때 중요한 것은 render source와 command guard를 함께 옮기는 것입니다. 예를 들어 UI가 hook의 `roomId`를 렌더링하면서 command는 legacy `roomId`로 보내면 두 connection의 identity가 어긋날 수 있습니다. 이 commit은 해당 쌍을 같은 owner로 맞춥니다.

다만 legacy socket과 50ms input loop는 아직 남아 있어 paddle input만 두 번째 owner를 사용합니다.

## 입력 migration은 event의 “상태”가 아니라 “전이”를 전송한다

### `1ae6fa7836d8` — keyboard와 touch

새 `changeDirection` callback은 마지막 전송 방향과 같은 값이면 아무것도 하지 않습니다.

```ts
const changeDirection = useCallback((direction: -1 | 0 | 1) => {
  if (inputDirectionRef.current === direction) return;
  inputDirectionRef.current = direction;
  sendDirection(direction);
}, [sendDirection]);
```

keyboard branch는 editable target을 제외하고 keydown에서 `-1`/`1`, keyup에서 해당 방향을 놓았을 때 `0`을 보냅니다. touch/pointer button은 pointer down에서 방향, pointer up/cancel/leave에서 0을 보냅니다.

중립 복귀가 필요한 browser lifecycle도 처리합니다.

- `window.blur`
- `document.visibilitychange`에서 hidden
- room ID 변경
- component cleanup

이 설계는 50ms마다 같은 direction을 반복 전송하는 이전 loop와 다릅니다. input sequence는 실제 사용자 intent 전이마다 증가하고, lost keyup 상황은 blur/hidden neutralization으로 보완합니다.

## legacy owner를 작은 단위로 제거

### `31b5122add6f` — 50ms input loop 제거

component-local `directionRef`와 `setInterval(..., 50)` command loop를 삭제합니다. replacement인 `changeDirection -> hook.sendDirection -> GameSocketClient.sendDirection`이 이미 keyboard/touch caller에 연결된 뒤이므로 삭제가 기능 공백을 만들지 않습니다.

### `fa35e8d15b4e` — inline WebSocket lifecycle 제거

page의 `WS_URL`, `requestWsTicket`, `socketRef`, `ticketRequestRef`, `openGameSocket`, raw `onopen/onmessage/onclose`, `closeCurrentSocket` 경로를 제거합니다. ticket 취소와 socket replacement는 hook 아래의 client가 소유합니다.

이 commit은 대규모 behavior change가 아니라 duplicate transport owner를 없애는 cleanup입니다. reconnect/fresh-ticket behavior를 page가 다시 구현하지 않도록 하는 효과가 있습니다.

### `365d66c72343` — legacy command 함수 제거

page-local ready, chat send, pause/resume, close command가 삭제됩니다. controls는 이미 hook callbacks를 사용하므로 남은 함수는 dead/duplicate path였습니다.

### `06664abbae7f` — duplicate state와 old imports 제거

legacy room/snapshot/status/messages state, sequence refs, protocol parser/API imports를 정리합니다. 여기서 중요한 결과는 “새 구조를 사용한다”가 아니라 **같은 의미의 owner가 하나만 남는다**는 것입니다.

## `9faada0df1d7` — 최종 단일 surface

마지막 commit은 임시 alias와 `connection` object 우회 사용까지 제거하고 hook을 직접 구조 분해합니다.

```ts
const {
  state,
  connectQueue,
  connectTournament,
  ready,
  sendChat,
  togglePause,
  sendDirection
} = useGameConnection();

const { snapshot, roomId, messages } = state;
```

page에 남는 local state는 presentation/user input에 한정됩니다.

- `chatInput`: 아직 전송되지 않은 form 값
- `autoStartedRef`: URL intent를 한 번만 소비했는지
- `inputDirectionRef`: 같은 방향 transition 중복 방지

server-derived score, phase, opponent, room, messages, notice는 모두 hook state에서 읽습니다. route query auto-start도 hook command를 사용하고, canvas는 hook snapshot을 받습니다. `aria-live="polite"`가 connection notice 변화를 보조기술에 전달합니다.

### 최종 page가 더 이상 소유하지 않는 것

- raw `WebSocket`
- ticket request/AbortController
- protocol JSON parse
- connection generation/reconnect timer
- room/snapshot/status/messages server state
- input sequence
- ready/chat/pause packet shape

## migration 상태표

| 단계 | Connection 시작 | Render state | Commands | Paddle input | Raw socket owner |
| --- | --- | --- | --- | --- | --- |
| hook 도입 전 | legacy page | legacy page | legacy page | legacy 50ms loop | page |
| `c334...` | legacy | legacy | legacy | legacy | page + unused hook client |
| `898...` | URL auto-entry만 hook | legacy 중심 | legacy 중심 | legacy | page + hook |
| `9a672...` | hook | hook | hook | legacy | page + hook |
| `1ae6...` | hook | hook | hook | hook transition | page + hook |
| legacy 제거 commits | hook | hook | hook | hook | hook/client만 |
| `9faada...` | hook | hook | hook | hook | hook/client만 |

## 최종 흐름

```text
[PlayPage]
  presentation local state: chatInput / autoStarted / last input direction
  user intent:
    route query -> connectQueue/connectTournament
    buttons -> ready/sendChat/togglePause
    keyboard/touch -> sendDirection
          ↓
[useGameConnection]
  reducer state + command preconditions + event mapping
          ↓
[GameSocketClient]
  ticket/socket/generation/reconnect/input sequence
```

이 Thread는 reconnect 알고리즘 자체보다 migration의 안전한 순서를 설명합니다. transport/reducer의 내부 불변 조건은 Thread 04, authoritative canvas projection은 Thread 07에서 다룹니다.

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. build·Vitest·브라우저 입력 테스트는 실행하지 않았습니다.
