# Versioned realtime protocol과 뒤로 가지 않는 상태

실시간 연결에서는 JSON parsing에 성공했다고 메시지가 유효한 것이 아닙니다. 필드가 빠졌거나, 알 수 없는 필드가 섞였거나, 다른 protocol version이거나, 형식은 맞지만 이미 처리한 입력·snapshot일 수 있습니다.

이 Thread는 두 문제를 분리합니다.

1. **형식 경계**: 양방향 메시지는 runtime schema를 통과하고 `v: 1`을 가져야 합니다.
2. **순서 경계**: 형식이 맞아도 이전 sequence 이하라면 authoritative state나 browser view를 되돌리지 않습니다.

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `a974f8cd9712` | `feat(shared): WebSocket 이벤트 메시지 검증` | A | `PROTOCOL, SIMULATION, REALTIME` | client event의 첫 runtime parser와 server event type 도입 |
| `7d3437c49152` | `feat(protocol): versioned game snapshot 계약 정의` | A | `PROTOCOL, SIMULATION, REALTIME` | snapshot·finished result의 엄격한 v1 데이터 계약 정의 |
| `0595a386000a` | `feat(protocol): versioned WebSocket event codec 연결` | S | `PROTOCOL, REALTIME, ARCH` | 모든 client/server event에 `v: 1`과 strict runtime codec 적용 |
| `1567f5005ef8` | `feat(game): room별 input sequence 중복을 차단` | A | `SIMULATION, REALTIME, PERSISTENCE` | server가 연결별·room별 마지막 input sequence를 소유 |
| `8a8787d03a19` | `feat(play): versioned game input과 snapshot 소비` | A | `PROTOCOL, SIMULATION, REALTIME` | browser가 connection별 input/snapshot sequence를 관리 |
| `f655969b0d36` | `test(protocol): versioned realtime contract 검증` | A | `PROTOCOL, REALTIME, PERSISTENCE` | stale version·잘못된 sequence·불가능한 result 조합을 거부 |

## 1. 첫 parser는 client 입력을 검증했지만 protocol 전체를 닫지는 못했다

### `a974f8cd9712`

공유 패키지에 client event의 discriminated union과 `parseClientEvent`가 생깁니다. queue 참가, ready, pause/resume, paddle input, chat 같은 명령을 `type`별로 구분하고 JSON 문자열을 runtime schema로 검사합니다.

이 시점의 경계에는 세 가지 제한이 있습니다.

- client event에 protocol version이 없습니다.
- schema가 아직 모든 object에서 strict하지 않아 알 수 없는 필드가 조용히 제거될 수 있습니다.
- server event는 TypeScript type과 단순 `JSON.stringify` 중심이므로 runtime에서 잘못 구성된 outbound event를 막지 못합니다.

즉 “외부 JSON을 `unknown`으로 받아 검사한다”는 첫 경계는 생겼지만, 양방향 wire contract와 호환성 version은 아직 완성되지 않았습니다.

## 2. snapshot은 화면용 객체가 아니라 복구 가능한 authoritative envelope가 되었다

### `7d3437c49152`

이 커밋은 game snapshot의 필드를 명시적으로 고정합니다.

```ts
{
  roomId,
  tick,
  sequence,
  serverTimeMs,
  state: {
    phase,
    players,
    paddles,
    ball,
    leftScore,
    rightScore
  }
}
```

`tick`은 simulation 진행량, `sequence`는 전송되는 snapshot의 순서, `serverTimeMs`는 관측 시각을 나타냅니다. 서로 같은 숫자로 우연히 움직일 수 있어도 의미가 다릅니다. 특히 browser는 simulation tick이 아니라 snapshot sequence로 stale delivery를 판정합니다.

경기 결과는 `persisted`를 discriminator로 사용합니다.

| `persisted` | `matchId` | `ratingDelta` | 의미 |
| --- | --- | --- | --- |
| `true` | non-empty string | finite number | DB에 확정된 등록 사용자 경기 |
| `false` | `null` | `0` | 영속화하지 않은 임시 경기 |

따라서 `{ persisted: true, matchId: null }`처럼 의미상 불가능한 조합이 type뿐 아니라 runtime schema에서도 거부될 준비가 됩니다.

## 3. `v: 1`과 strict codec을 모든 wire event의 입구·출구에 배치했다

### `0595a386000a` — protocol boundary 확정

모든 client/server event가 literal version을 포함하고, 각 object schema가 strict해집니다. parser와 encoder 모두 schema를 통과합니다.

```ts
const version = { v: z.literal(1) } as const;

export function parseClientEvent(payload: string): ClientEvent {
  return clientEventSchema.parse(JSON.parse(payload));
}

export function parseServerEvent(payload: string): ServerEvent {
  return serverEventSchema.parse(JSON.parse(payload));
}

export function encodeServerEvent(event: ServerEvent): string {
  return JSON.stringify(serverEventSchema.parse(event));
}
```

위 코드는 실제 구조를 축약한 것입니다. 중요한 것은 encode도 검증한다는 사실입니다. 서버 내부 코드가 누락된 필드나 잘못된 result 조합을 만들면 그대로 client에 보내지 않고 outbound boundary에서 실패합니다.

`inputSeq`도 non-negative safe integer로 제한됩니다. 단순히 숫자형인지가 아니라 비교·증가에 안전한 정수 범위인지까지 wire contract에 포함됩니다.

strict schema가 만드는 차이는 다음과 같습니다.

```text
{ v: 1, type: "game.ready", roomId: "r1" }
  → 허용

{ v: 1, type: "game.ready", roomId: "r1", admin: true }
  → 알 수 없는 필드 때문에 거부

{ type: "game.ready", roomId: "r1" }
  → version 누락으로 거부
```

이 커밋이 protocol version negotiation을 구현한 것은 아닙니다. 지원되는 version은 1 하나이며, 다른 version은 명시적으로 거부합니다.

## 4. 형식상 유효한 오래된 입력을 server에서 no-op으로 만들었다

### `1567f5005ef8`

retransmission, 지연 delivery, client bug로 같은 room input이 다시 올 수 있습니다. 이 커밋은 `Client`가 `lastInputSequenceByRoom`을 보유하고 다음 조건을 적용합니다.

```ts
const previous = client.lastInputSequenceByRoom.get(roomId);
if (previous !== undefined && inputSeq <= previous) return;

client.lastInputSequenceByRoom.set(roomId, inputSeq);
applyDirection(room, side, direction);
```

검사 순서는 중요합니다. room이 존재하고, 경기가 `playing`이며, 해당 client가 실제 참가 side인지를 확인한 뒤 sequence를 반영합니다. 존재하지 않는 room이나 권한 없는 input이 sequence만 먼저 소비해 정상 입력을 막지 않습니다.

이 SHA의 sequence owner는 **WebSocket 연결의 Client 객체**입니다. 사용자 계정 전체에 대한 global sequence가 아닙니다. 연결이 교체되면 새 Client map에서 다시 시작할 수 있으며, room ID가 다르면 각 sequence가 독립적입니다. 후대의 Thread 08은 rate limit까지 포함한 `InputGate`로 이 책임을 사용자 단위에 더 넓히지만, 그 구현을 이 커밋에 소급해서는 안 됩니다.

## 5. browser도 연결 수명마다 독립적인 단조 증가 규칙을 갖는다

### `8a8787d03a19`

web client는 v1 event를 사용하고 두 counter를 분리합니다.

- `inputSequence`: 보낼 `game.input`마다 증가
- `lastSnapshotSequence`: 마지막으로 화면에 반영한 snapshot sequence

연결을 새로 시작할 때 두 값은 함께 초기화됩니다. paddle input은 50ms 주기로 현재 방향을 전송하며, server event는 `parseServerEvent`를 통과한 뒤 reducer로 들어갑니다.

```ts
inputSequence += 1;
socket.send(JSON.stringify({
  v: 1,
  type: "game.input",
  roomId,
  inputSeq: inputSequence,
  direction
}));
```

snapshot 수신은 다음 조건을 갖습니다.

```ts
if (snapshot.sequence <= lastSnapshotSequence) return;
lastSnapshotSequence = snapshot.sequence;
dispatch({ type: "snapshotReceived", snapshot });
```

따라서 네트워크가 snapshot 12 뒤에 snapshot 11을 전달해도 화면이 과거 위치로 되돌아가지 않습니다. sequence gap은 허용됩니다. slow client에게 모든 중간 snapshot을 재생하는 것이 아니라 최신 authoritative state를 받아들이는 모델이기 때문입니다.

연결별 reset도 의도적입니다. 새 socket이 복구한 room의 최신 snapshot을 sequence 1부터 보낼 수 있다면, 이전 socket의 마지막 sequence 500과 비교해서 버리면 안 됩니다. sequence의 비교 범위는 protocol에 암묵적으로 전체 서비스가 아니라 **하나의 connection lifetime**입니다.

## 6. negative contract를 테스트로 고정했다

### `f655969b0d36`

테스트는 정상 예시뿐 아니라 과거 구현이 허용했을 법한 payload를 직접 거부합니다.

- `v`가 없는 `presence.changed`는 server event parser를 통과하지 못합니다.
- 음수 snapshot sequence는 거부됩니다.
- `persisted: true`인데 `matchId: null`인 결과는 거부됩니다.
- client/server codec이 v1의 정확한 shape를 round-trip합니다.

이 테스트는 packet reorder를 실제 네트워크에서 재현하지 않습니다. schema의 rejection과 sequence 처리 함수의 branch를 고정하며, 부하·지연 환경의 서비스 수준 관찰은 Thread 08의 load harness가 담당합니다.

## 최종 protocol 흐름

```text
client object
  → v1 strict encode/JSON
  → server parseClientEvent
  → room/side/state authorization
  → inputSeq monotonic check
  → authoritative input state

server object
  → serverEventSchema validation
  → JSON
  → browser parseServerEvent
  → snapshot sequence monotonic check
  → UI reducer
```

최종 불변 조건은 다음과 같습니다.

- 모든 accepted realtime event는 `v: 1`의 strict runtime schema를 통과합니다.
- unknown field, unsafe sequence, 의미상 불가능한 result 조합은 wire boundary에서 거부됩니다.
- server는 같은 connection·room에서 이전 input sequence 이하를 적용하지 않습니다.
- browser는 같은 connection에서 이전 snapshot sequence 이하를 반영하지 않습니다.
- sequence gap은 허용하며, latest authoritative state로 전진합니다.

이 Thread는 ticket이 누구에게 socket 입장을 허용하는지 다루지 않습니다. 그 앞단은 Thread 02입니다. socket 교체·재연결 때 connection lifetime이 어떻게 바뀌는지는 Thread 05, backlog에서 중간 snapshot을 의도적으로 버리는 정책은 Thread 08에서 다룹니다.
