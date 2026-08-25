# Thread: authoritative snapshot을 렌더링하고 room-scoped input만 전송하기

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 sample canvas와 browser-local presentation에서 출발해, server snapshot이 score·phase·player·ball·paddle의 유일한 source가 되는 과정과, browser input이 game state를 계산하지 않고 versioned intent만 전송하는 구조를 다룹니다.

초기 화면은 connection이 없어도 sample score와 opponent를 보여 줬습니다. WebSocket이 연결된 뒤에도 raw JSON cast, stale snapshot 수용, unversioned command, room과 무관한 input sequence가 남았습니다. rendering smoothness를 위해 interpolation을 추가하는 순간에는 또 다른 위험이 생겼습니다. browser가 interpolation을 넘어 실제 simulation rule을 계산하면 server authority와 갈라질 수 있습니다.

최종 규칙은 다음과 같습니다.

- browser는 score, phase, winner, room state를 계산하지 않습니다.
- snapshot이 없을 때 실제 경기처럼 보이는 sample score/opponent를 만들지 않습니다.
- accepted snapshot은 sequence가 단조 증가해야 합니다.
- interpolation은 accepted snapshot 두 개의 복사본 사이에서 위치만 보간합니다.
- input은 protocol version, room ID, monotonic input sequence, direction intent만 전송합니다.
- input loop와 render animation은 room/phase 변경 또는 unmount에서 정리됩니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3449f7988e1b` | `feat(web): 퐁 캔버스 미리보기 구현` | B | SIMULATION, REALTIME, WEB | shared dimensions와 `GameSnapshot`으로 field/paddle/ball/score를 그립니다. |
| 2 | `91962d36bd59` | `feat(play): 경기장 화면 구성` | B | PROTOCOL, REALTIME, WEB | Pong canvas 중심의 초기 play route를 추가합니다. |
| 3 | `737aa99cb4cb` | `feat(play): WebSocket 경기 연결 구현` | B | AUTH, PROTOCOL, SIMULATION | play screen을 realtime protocol client로 전환합니다. |
| 4 | `977ca863050f` | `feat(play): keyboard paddle 입력 연결` | B | PROTOCOL, SIMULATION, REALTIME | Arrow/W/S input을 room-scoped game command로 mapping합니다. |
| 5 | `afbd8847b1dd` | `feat(play): 경기 채팅 입력 연결` | B | REALTIME, WEB | 현재 room과 inline WebSocket에 match-chat form을 연결합니다. |
| 6 | `3cd56054bdab` | `fix(play): 패들 조작과 Canvas rendering 개선` | B | SIMULATION, REALTIME, WEB | persistent direction을 50ms마다 sampling하고 snapshot을 80ms 보간합니다. |
| 7 | `6a7aa285fe68` | `fix(play): 실제 경기 상태에 맞게 세션 표시` | A | SIMULATION, REALTIME, WEB | score/opponent/ready/chat/input/terminal cleanup을 latest server snapshot에서 파생합니다. |
| 8 | `e4e2dec55805` | `feat(play): 일시정지와 재개 UI 연결` | B | REALTIME, WEB | server snapshot phase에서 pause/resume availability를 파생하고 current room command를 전송합니다. |
| 9 | `8a8787d03a19` | `feat(play): versioned game input과 snapshot 소비` | A | PROTOCOL, SIMULATION, REALTIME | inputSeq와 shared event parser, monotonic snapshot acceptance를 browser에 적용합니다. |
| 10 | `868ced55a626` | `refactor(web): PongCanvas snapshot state 렌더링` | B | SIMULATION, REALTIME, WEB | canvas/interpolation buffer를 nested snapshot state에 맞춥니다. |

## preview에서 실제 화면까지

### `3449f7988e1b` — `PongCanvas`의 첫 projection

`PongCanvas`는 shared `GAME_WIDTH`, `GAME_HEIGHT`, paddle/ball dimensions를 사용해 DPR에 맞는 canvas를 만들고 field, center line, paddles, ball, score를 그립니다. prop이 없으면 `sampleSnapshot()`을 사용합니다.

이 선택은 component를 독립적으로 preview하기 쉽지만, caller가 snapshot을 넘기지 않아도 실제 game처럼 보인다는 의미입니다. 이 시점에는 interpolation이나 animation loop가 없고 snapshot 변경마다 drawing effect가 한 번 실행됩니다.

### `91962d36bd59` — sample match를 가진 `/play`

PlayPage가 sample snapshot, score, opponent/status, ready/control UI를 구성합니다. network와 room identity가 없는데도 score와 canvas가 채워져 있습니다. button 일부는 handler가 없거나 local presentation만 바꿉니다.

이 커밋은 화면 layout을 만드는 단계입니다. “현재 게임이 존재한다”는 보장은 전혀 없습니다. 후속 authoritative fix가 왜 sample을 제거해야 하는지 보여 주는 출발점입니다.

## inline realtime client의 성장과 한계

### `737aa99cb4cb` — 첫 WebSocket 연결

PlayPage가 localStorage token을 query에 넣어 socket을 열고, `queue.join`, `game.ready`를 전송합니다. message는 다음처럼 TypeScript 단언으로 처리합니다.

```ts
const message = JSON.parse(event.data) as ServerEvent;
```

`queue.matched`는 room/opponent status를, `game.snapshot`은 local snapshot을, `game.finished`는 status를 갱신합니다. 그러나 snapshot initial value는 sample이므로 연결 전과 실패 뒤에도 fake scene이 남습니다. runtime schema, stale sequence gate, robust replacement cleanup도 없습니다.

credential 문제는 Thread 02에서 ticket 방식으로 교정되고, lifecycle 분리는 Thread 04·05에서 이루어집니다. 이 Thread에서는 authoritative state와 rendering 관점만 추적합니다.

### `977ca863050f` — key event를 command로

ArrowUp/W와 ArrowDown/S keydown이 현재 room으로 `game.input` direction을 전송하고, keyup은 neutral `0`을 보냅니다. page가 key listener와 socket send를 직접 소유합니다.

초기 방식은 key repeat 빈도와 browser event timing에 전송량이 의존하고, editable field focus와 lost keyup 상황을 충분히 구분하지 않습니다. 입력 helper와 transition-based 최종 구조는 Thread 04·05에서 완성됩니다.

### `afbd8847b1dd` — match chat

chat form이 body를 trim하고 current room ID와 socket에 `chat.send`를 보냅니다. inbound message를 문자열 history로 표시합니다. 이 시점에는 room scope filtering이 없어 다른 room/lobby message를 차단하는 별도 predicate가 없습니다. 그 fix는 Thread 04의 `85edd6d1e26a`입니다.

## `3cd56054bdab` — input sampling과 visual interpolation

### persistent direction + 50ms sampling

keydown은 socket에 즉시 packet을 보내는 대신 `directionRef.current`를 바꿉니다. keyup은 0으로 되돌립니다. room이 있을 때 interval이 50ms마다 현재 direction을 전송합니다.

```ts
const timer = window.setInterval(() => {
  const socket = socketRef.current;
  if (!socket || socket.readyState !== WebSocket.OPEN) return;
  socket.send(JSON.stringify({
    type: "game.input",
    roomId,
    direction: directionRef.current
  }));
}, 50);
```

room 변경/unmount에서 interval을 clear합니다. 입력 빈도는 key repeat가 아니라 fixed sampling에 의해 결정됩니다. 후속 hook migration은 이 polling loop를 제거하고 방향 전이만 전송합니다.

### 80ms 뒤를 보는 interpolation buffer

canvas는 snapshot을 `receivedAt`과 함께 최대 8개 복사해 저장합니다. animation frame마다 `now - 80ms` 시점의 앞뒤 sample을 찾아 paddle/ball position을 선형 보간합니다.

중요한 ownership 결정은 snapshot을 그대로 ref에 넣지 않고 nested object를 복사한다는 점입니다. 이후 caller가 같은 object를 mutate하거나 다른 reference를 재사용해도 이미 받은 render sample이 바뀌지 않습니다.

```ts
function interpolateSnapshot(previous, next, ratio) {
  return {
    ...next,
    paddles: { /* y only mixed */ },
    ball: {
      ...next.ball,
      position: { x: mix(...), y: mix(...) }
    }
  };
}
```

score, phase, player identity는 `next` snapshot을 사용하고 물리 position만 보간합니다. browser가 collision, velocity update, score rule을 재실행하지 않습니다. RAF는 cleanup에서 cancel됩니다.

## `6a7aa285fe68` — fabricated game session 제거

이 A급 fix에는 server matchmaking 변경도 함께 들어 있지만, 이 Thread는 browser files만 다룹니다.

### sample snapshot을 nullable state로

```diff
-const [snapshot, setSnapshot] = useState<GameSnapshot>(sampleSnapshot());
+const [snapshot, setSnapshot] = useState<GameSnapshot | null>(null);
```

score는 snapshot이 없으면 “경기 전”, opponent는 “대기 중”입니다. `PongCanvas`도 default sample 대신 nullable snapshot을 받고 buffer를 비웁니다. no-snapshot canvas를 유지해야 할 때는 empty neutral state를 그리지만 room/player/score를 실제 match처럼 꾸미지 않습니다.

### command capability는 snapshot phase에서 파생

```ts
const phase = snapshot?.phase ?? "waiting";
const canReady = Boolean(roomId && phase === "waiting");
const canChat = Boolean(roomId && phase !== "finished" && chatInput.trim());
```

input interval은 `roomId && phase === "playing"`일 때만 살아 있습니다. ready/chat button도 같은 server-derived phase와 room identity를 사용합니다.

### connection 전환과 terminal cleanup

새 connect 전에 current socket을 닫고 room/snapshot/messages/chat input/direction을 초기화합니다. `game.finished`는 room을 null로 만들고 direction을 0으로 되돌리며 마지막 snapshot의 phase만 finished로 표시합니다. current socket의 onclose만 state를 정리하도록 identity guard를 둡니다.

unmount cleanup은 handler를 제거하고 OPEN/CONNECTING socket을 닫습니다. 이 시점에도 lifecycle은 page에 있지만, 최소한 session presentation과 resource cleanup이 실제 connection state에 맞습니다.

### 보장과 한계

보장:

- no-snapshot 상태에서 fake score/opponent가 보이지 않음
- input/ready/chat capability가 current room·server phase와 맞음
- 새 connection과 finished/closed path가 stale local match state를 정리함

한계:

- snapshot parser는 아직 full shared runtime parser가 아님
- sequence ordering이 없음
- transport owner가 page에 남음
- neutral empty canvas는 “no game” placeholder이지 server snapshot이 아님

## pause/resume과 versioned protocol

### `e4e2dec55805` — server phase가 control을 결정

`canPause`는 room이 있고 phase가 playing일 때, `canResume`은 paused일 때만 true입니다. button은 current room으로 `game.pause` 또는 `game.resume`을 보냅니다. browser는 phase를 스스로 toggle하지 않고 다음 server snapshot을 기다립니다.

### `8a8787d03a19` — protocol version, inputSeq, snapshot sequence

raw `JSON.parse(...) as ServerEvent`를 shared `parseServerEvent`로 바꿉니다. 모든 client command에 `v: 1`을 넣고, input마다 sequence를 증가시킵니다.

```ts
inputSequenceRef.current += 1;
socket.send(JSON.stringify({
  v: 1,
  type: "game.input",
  roomId,
  inputSeq: inputSequenceRef.current,
  direction: directionRef.current
}));
```

새 connection을 시작할 때 input sequence를 0, snapshot sequence를 -1로 reset합니다. snapshot은 다음 guard를 통과해야 합니다.

```ts
if (message.snapshot.sequence <= snapshotSequenceRef.current) return;
snapshotSequenceRef.current = message.snapshot.sequence;
setSnapshot(message.snapshot);
```

이제 transport에서 받아들인 payload가 schema를 통과하고, application state가 같은/과거 snapshot으로 되돌아가지 않습니다. snapshot shape도 metadata와 nested `state`로 분리됩니다.

이 commit이 browser/server clock synchronization을 구현하는 것은 아닙니다. ordering identity로 `sequence`를 사용하고 serverTime은 snapshot metadata로 보존합니다.

## `868ced55a626` — canvas를 nested authoritative state에 맞추기

protocol shape가 다음처럼 바뀐 뒤 canvas도 동일 source를 읽습니다.

```text
GameSnapshot
  roomId, tick, sequence, serverTimeMs
  state
    phase, leftScore, rightScore
    paddles, ball, players
```

`drawSnapshot`, `toRenderSample`, `emptySnapshot`, `interpolateSnapshot`이 모두 `snapshot.state`를 사용합니다.

copy는 metadata를 보존하면서 state 내부의 paddle, ball, players를 깊게 복사합니다. interpolation 결과는 `next` snapshot의 sequence/tick/serverTime/phase/score/player를 유지하고 paddle y와 ball position만 previous→next 사이로 섞습니다.

```ts
return {
  ...next,
  state: {
    ...next.state,
    paddles: { /* y interpolation */ },
    ball: {
      ...next.state.ball,
      position: { /* x/y interpolation */ }
    }
  }
};
```

이 구조는 visual smoothing이 authoritative metadata와 game rule을 덮지 않게 합니다.

## 최종 source-of-truth 표

| 화면 값/동작 | Source | browser가 하는 일 |
| --- | --- | --- |
| score | `snapshot.state.leftScore/rightScore` | 문자열로 표시 |
| phase | `snapshot.state.phase` | control enable 여부 파생 |
| opponent | `snapshot.state.players` | right side player 표시 |
| paddle/ball | accepted snapshots | 80ms buffer에서 위치만 보간 |
| room | matched/snapshot event room ID | command scope에 포함 |
| input order | connection-local `inputSeq` | intent마다 증가 |
| snapshot order | server `sequence` | same/older 거부 |
| pause/resume | current phase + room | versioned command 전송, local toggle 없음 |

## 최종 흐름

```text
[server event text]
  -> shared runtime parser
  -> snapshot sequence gate
  -> accepted GameSnapshot 저장
  -> page derives score/phase/opponent/capabilities
  -> PongCanvas copies sample
  -> RAF selects now-80ms pair
  -> position-only interpolation
  -> draw

[key/pointer intent]
  -> direction (-1/0/1)
  -> current room + v + monotonic inputSeq
  -> server command
  -> browser는 결과 state를 계산하지 않고 다음 snapshot을 기다림
```

이 Thread는 component가 authoritative snapshot을 소비하는 규칙을 다룹니다. reducer/transport ownership과 reconnect는 Thread 04, page migration과 최종 transition-based input cleanup은 Thread 05에서 다룹니다.

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. canvas/E2E/실시간 테스트는 이 환경에서 실행하지 않았습니다.
