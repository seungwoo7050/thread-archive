# 일시정지는 타이머뿐 아니라 입력 의도도 멈춰야 한다

원문 Thread: `Pause, resume, and input neutralization`

## 이 Thread가 다루는 문제

서버 simulation의 tick을 멈추면 공과 paddle 위치는 더 이상 바뀌지 않습니다. 하지만 pause 직전 사용자가 오른쪽 이동을 누르고 있었다면 `direction = 1`은 메모리에 남을 수 있습니다. resume이 timer/scheduler만 다시 시작하면 새 입력이 없어도 paddle이 곧바로 움직입니다.

따라서 pause는 시간 진행만 끊는 기능이 아니라 상태 경계입니다.

> `playing → paused`가 성공한 순간, 이전 frame의 지속 입력은 공개 snapshot과 내부 simulation 양쪽에서 모두 무효가 되어야 한다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `655bc7bd8df7` | `feat(protocol): 일시정지 WebSocket 계약 추가` | B | PROTOCOL, REALTIME, WEB | `paused` phase와 room-scoped pause/resume command를 추가한다. |
| 2 | `d93612c18e6f` | `feat(game): 서버 주도 일시정지 기능 추가` | B | SIMULATION, REALTIME | 참가자·phase를 검사해 timer와 server snapshot을 전환한다. |
| 3 | `e4e2dec55805` | `feat(play): 일시정지와 재개 UI 연결` | B | REALTIME, WEB | server snapshot에서 버튼 상태를 파생하고 command를 전송한다. |
| 4 | `f46bbab95ea5` | `fix(game): 일시정지 시 paddle 입력 상태 초기화` | A | SIMULATION, REALTIME, RISK | pause 진입 때 공개 snapshot과 내부 simulation의 direction을 모두 0으로 만든다. |
| 5 | `632cbf13b616` | `test(game): pause 전 입력이 재개 뒤 남지 않음 검증` | B | SIMULATION, REALTIME, TEST | fake time으로 pause 전 입력이 resume 이후 되살아나지 않는지 검증한다. |

## 1. protocol이 pause를 표현하기 시작했다

### `655bc7bd8df7` — phase와 command vocabulary

공유 `GamePhase`가 다음처럼 확장됩니다.

```ts
export type GamePhase =
  | "waiting"
  | "countdown"
  | "playing"
  | "paused"
  | "finished";
```

WebSocket client event union에는 `game.pause`와 `game.resume`이 추가되고 두 event 모두 `roomId`를 요구합니다.

이 commit은 가능한 메시지를 표현할 뿐입니다. sender가 그 room의 참가자인지, 현재 phase에서 전이가 유효한지, timer가 실제로 멈추는지는 server가 판단합니다.

## 2. server가 room phase와 timer를 소유한다

### `d93612c18e6f` — 최초 pause/resume 구현

이 SHA의 GameHub는 room마다 `timer: NodeJS.Timeout | null`과 `snapshot.phase`를 직접 소유합니다. `pauseRoom`은 다음 조건을 모두 만족할 때만 동작합니다.

- room이 존재
- 현재 phase가 `playing`
- `sideFor(room, client)`가 left/right 좌석을 반환

성공하면 interval을 clear하고 `room.timer = null`, phase를 paused로 바꾼 뒤 serverTime을 갱신해 snapshot을 room에 broadcast합니다. `resumeRoom`은 paused 참가자만 허용하고 phase를 playing으로 바꾼 뒤 timer가 없을 때만 새 interval을 만듭니다.

같은 commit에서 input 수용 조건도 바뀝니다.

```ts
private applyInput(
  client: Client,
  roomId: string,
  direction: -1 | 0 | 1
): void {
  const room = this.rooms.get(roomId);
  if (!room || room.snapshot.phase !== "playing") return;
  const side = sideFor(room, client);
  if (side) room.snapshot.paddles[side].dy = direction;
}
```

paused 동안 도착한 새 input은 무시됩니다. 그러나 **pause 이전에 이미 저장된 `dy`는 그대로**입니다. tick이 멈춰 있어 당장은 보이지 않지만 resume 뒤 다시 사용됩니다.

### `e4e2dec55805` — UI는 server snapshot에서 파생한다

play 화면은 local toggle state를 별도로 만들지 않고 snapshot phase로 버튼을 결정합니다.

```ts
const canPause = Boolean(roomId && phase === "playing");
const canResume = Boolean(roomId && phase === "paused");
```

버튼은 현재 가능한 command만 보내고, `game.snapshot`을 받을 때 playing/paused/waiting status 문구를 갱신합니다. 사용자가 눌렀다는 이유로 즉시 paused 화면을 만드는 optimistic transition이 아니라, server가 broadcast한 phase가 UI의 사실 원천입니다.

이 시점까지의 정상 전이는 다음과 같습니다.

```text
participant sends pause
→ server validates room/seat/playing
→ interval clear
→ phase paused
→ snapshot broadcast
→ UI enables resume
```

남아 있는 결함은 timer 밖의 입력 상태입니다.

## 3. pause를 입력 세대의 경계로 완성하기

### `f46bbab95ea5` — 공개 상태와 내부 상태를 함께 neutralize

이 commit 시점에는 앞선 구현 뒤 realtime core가 리팩터링되어 room이 `session`, `roomScheduler`, `snapshot.state`, `simulation`을 가집니다. 이 후대 표현을 `d93612c18e6f`에 소급하지 않습니다.

해당 SHA의 pause 순서는 다음과 같습니다.

```ts
this.roomScheduler.unregister(room.id);
const sessionState = room.session.pause();
if (sessionState !== "paused") return;

for (const side of ["left", "right"] as const) {
  room.snapshot.state.paddles[side].dy = 0;
  room.simulation.paddles[side].direction = 0;
}

room.snapshot.state.phase = sessionState;
this.broadcastSnapshot(room);
```

두 곳을 모두 0으로 만드는 이유가 다릅니다.

| 상태 | 역할 | 하나만 초기화했을 때의 문제 |
| --- | --- | --- |
| `snapshot.state.paddles[side].dy` | client가 관찰하는 공개 방향 | UI는 계속 이동 입력이 남은 snapshot을 받음 |
| `simulation.paddles[side].direction` | 다음 tick이 실제로 소비하는 내부 의도 | resume 뒤 새 입력 없이 paddle이 움직임 |

scheduler unregister가 먼저 수행되므로 neutralization 중에 tick이 진행되지 않습니다. session pause가 성공한 경우에만 상태를 지우고 snapshot을 broadcast합니다. pause 진입을 **이전 input generation을 폐기하는 지점**으로 만든 수정입니다.

### `632cbf13b616` — fake time으로 stale intent를 재현

테스트는 memory repository와 fake socket으로 AI room을 만들고 fake timer를 사용합니다.

1. `game.ready`
2. input sequence 0, direction 1
3. `game.pause`
4. paused snapshot의 left `dy === 0` 확인
5. paused 상태에서 input sequence 1, direction 0 전송
6. `game.resume`
7. fake timer를 100ms 진행
8. playing snapshot에서도 left `dy === 0` 확인

paused input이 무시되는 상황에서도 결과가 0인 이유는 pause 시점의 neutralization이어야 합니다. 이 fixture는 “pause 뒤 정지 input을 받았기 때문에 우연히 0이 됐다”는 잘못된 구현을 구별하려고 phase gating과 resume tick까지 통과합니다.

테스트가 증명하는 것은 이 한 room·한 player·100ms 경계입니다. 장시간 pause, reconnect, 두 human participant의 모든 입력 interleaving을 포괄하지는 않습니다. source는 확인했지만 실행하지 않았습니다.

## 최종 상태 전이

| 요청 | 선행 상태 | server 처리 | input 상태 |
| --- | --- | --- | --- |
| pause | playing + seated participant | scheduler/timer 중단, session paused, snapshot broadcast | 양쪽 공개·내부 direction 0 |
| pause | waiting/paused/finished 또는 비참가자 | 무시 | 변경 없음 |
| input | playing + seated participant | sequence/policy에 따라 반영 | 새 세대 input |
| input | paused | 무시 | 0 유지 |
| resume | paused + seated participant | session playing, scheduler 재등록, snapshot broadcast | 새 input 전까지 0 |
| resume | playing/finished 또는 비참가자 | 무시 | 변경 없음 |

일시정지 기능의 핵심은 “화면이 멈춘다”가 아닙니다. **재개한 simulation이 pause 이전 사용자의 의도를 다시 실행하지 않는 것**입니다.

## 조사 범위

각 설명은 `web/ft_transcendence`의 표시 SHA diff와 해당 시점 source를 기준으로 작성했습니다. fake-timer test는 실행하지 않았습니다.
