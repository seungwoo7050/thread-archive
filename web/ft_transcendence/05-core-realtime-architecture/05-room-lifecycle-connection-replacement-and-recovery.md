# Room lifecycle: 연결 교체와 연결 복구를 같은 문제로 다루지 않기

같은 사용자가 새 WebSocket을 열었다고 해서 항상 “경기가 끊겼다가 복구됐다”는 뜻은 아닙니다. 기존 socket이 아직 살아 있는 상태에서 새 socket이 들어올 수도 있고, transport가 먼저 끊긴 뒤 15초 안에 돌아올 수도 있습니다.

이 Thread는 두 상황을 분리합니다.

- **active replacement**: 같은 사용자의 새 socket이 즉시 기존 connection ownership을 인수합니다.
- **disconnected recovery**: room의 side를 사용자 ID로 15초 예약하고, 새 socket이 그 side를 되찾습니다.

예약이 만료되면 한 명만 남은 방은 몰수패로 확정하고, 둘 다 사라진 방은 경기 결과 없이 폐기합니다. browser는 그 15초 안에서 새 one-time ticket을 발급받아 재연결하되 최초 queue command를 다시 보내지 않습니다.

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `aa5d6a338690` | `refactor(game): 게임 방 상태 전이 모델링` | S | `REALTIME, ARCH, RISK` | waiting·playing·paused·reconnecting·finished 상태와 15초 deadline 정의 |
| `8f64dfc117f3` | `feat(game): 게임 방 상태를 RoomSession에 연결` | A | `SIMULATION, REALTIME` | ready/pause/resume와 simulation 실행 여부를 상태 기계에 연결 |
| `a06d1705bbc9` | `feat(game): 사용자별 active connection 교체` | S | `REALTIME, ARCH, RISK` | 사용자당 active socket 하나와 즉시 room-side ownership 이전 |
| `c98d4b1e8b43` | `feat(game): 예약된 room connection 복구` | A | `SIMULATION, REALTIME` | 끊긴 사용자 ID로 side를 찾고 latest snapshot을 재전송 |
| `e593b1dd9fcd` | `feat(game): reconnect 예약 만료와 room 정리` | A | `SIMULATION, REALTIME, RISK` | 15초 timeout 뒤 forfeit 또는 empty-room abandon |
| `113e39acc85c` | `test(game): reconnect 복구 동작 검증` | A | `REALTIME, PERSISTENCE, RISK` | replacement·14,999ms recovery·15,000ms 단일 forfeit 검증 |
| `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | `AUTH, REALTIME, WEB` | browser 재시도·새 ticket·중복 queue 방지·복구 화면 전환 |

## 1. 먼저 room 상태를 timer와 socket callback 밖으로 꺼냈다

### `aa5d6a338690` — `RoomSession`

room lifecycle이 boolean 조합으로 흩어져 있으면 `paused && disconnected`, `finished && reconnecting` 같은 모순 상태를 만들기 쉽습니다. `RoomSession`은 상태와 허용 전이를 하나의 객체로 묶습니다.

```text
waiting
  └─ 양쪽 ready → playing

playing
  ├─ pause → paused
  ├─ disconnect → reconnecting
  └─ finish → finished

paused
  ├─ resume → playing
  ├─ disconnect → reconnecting
  └─ finish → finished

reconnecting
  ├─ 모든 예약 side 복구 → disconnect 전 상태로 복귀
  ├─ deadline 만료 → absentee 정보 반환
  └─ finish → finished
```

reconnecting으로 들어갈 때는 이전 상태가 playing이었는지 paused였는지를 기억합니다. 모든 side가 돌아오면 무조건 playing으로 만드는 대신 원래 상태로 복원합니다.

15초 deadline과 disconnected side set도 `RoomSession`이 소유합니다. GameHub는 timer를 예약하지만, “지금 reconnect가 유효한가”, “누가 아직 빠져 있는가”는 session state를 통해 판단합니다.

이 커밋은 architecture의 중심이므로 S입니다. 이후 socket replacement, recovery, timeout은 모두 이 상태 기계를 기준으로 움직입니다.

### deadline의 정확한 경계

이 SHA의 비교식은 exact deadline에서 양쪽 API가 모두 행동할 여지를 남깁니다. reconnect는 `now > deadline`일 때 거부하므로 equality를 허용하고, expiry는 `now < deadline`일 때만 대기하므로 equality에서 만료시킬 수 있습니다. 따라서 정확히 같은 millisecond에는 timer callback과 reconnect callback 중 어떤 것이 먼저 실행되는지가 결과를 정할 수 있습니다.

후속 테스트는 `14,999ms` 복구와 `15,000ms` timer 만료를 고정하지만, 모든 event-loop ordering에서 equality를 하나의 결과로 만든 별도 synchronization primitive까지 추가하지는 않습니다. 이 경계는 최종 보장 범위를 설명할 때 유지해야 합니다.

## 2. 상태 기계가 실제 simulation 실행 여부를 결정하기 시작했다

### `8f64dfc117f3`

GameHub의 room이 `RoomSession`을 보유하고, ready/pause/resume command가 session transition을 통과합니다.

- AI room의 right side는 생성 시 ready로 처리됩니다.
- 필요한 참가자가 모두 ready가 되면 session이 `playing`으로 전환되고 scheduler가 시작됩니다.
- pause 성공 시 scheduler를 멈추고 snapshot phase를 `paused`로 바꿉니다.
- resume 성공 시 `playing`으로 돌아가 scheduler를 다시 시작합니다.

이 시점에는 reconnect용 field와 session state가 room에 들어오지만, socket close 뒤 side 예약과 timer까지 완성되지는 않습니다. 상태 표현을 runtime에 연결한 중간 단계입니다.

## 3. 살아 있는 connection 교체는 “disconnect”로 처리하지 않는다

### `a06d1705bbc9` — 사용자당 active owner 하나

GameHub는 `clientsByUser`를 추가해 user ID마다 현재 active client 하나를 가리킵니다. 새 connection이 들어왔을 때 이전 client가 있으면 다음 순서로 소유권을 이전합니다.

```text
new client 등록
→ 이전 client가 room side를 보유했다면 그 자리에 new client 배치
→ new client.roomId 설정
→ queue.matched와 latest snapshot 전송
→ 이전 heartbeat·snapshot buffer 중지
→ 이전 socket을 4001 / "connection replaced"로 close
```

room side를 먼저 새 client로 바꾼 뒤 old socket을 닫는 순서가 중요합니다. old socket의 `close` callback이 뒤늦게 실행되어도 `clientsByUser.get(userId) !== oldClient`이므로 현재 connection을 disconnect로 처리하지 않습니다.

message 처리도 current client identity를 확인합니다. 교체된 old socket이 close 전 buffered message를 보내더라도 room 입력이나 queue command를 적용하지 않습니다.

이 경로에서는 reconnect timeout을 시작하지 않습니다. 사용자에게 active owner가 계속 존재하기 때문입니다. active replacement를 일반 disconnect로 취급했다면, 정상적으로 교체된 경기 뒤 15초 후 forfeit timer가 발동할 수 있습니다.

## 4. transport가 실제로 끊긴 경우에만 side를 예약했다

### `c98d4b1e8b43` — 예약된 side 복구

active client 없이 새 connection이 들어오면 GameHub는 room의 `disconnectedUsers`에서 같은 user ID를 찾습니다. `RoomSession.reconnect`가 deadline과 side를 확인하고 성공하면 다음을 수행합니다.

```text
reserved side에 new client 연결
→ client.roomId 복원
→ disconnectedUsers에서 side 제거
→ queue.matched(roomId, side, opponent) 재전송
→ 현재 latest snapshot 전송
→ 모든 side가 복구됐으면 원래 room state 복원
→ playing이면 scheduler 재시작
```

client는 이전 socket에서 받은 중간 frame을 재생하지 않습니다. server가 보유한 authoritative latest snapshot 하나를 받아 현재 상태로 점프합니다. snapshot sequence가 disconnect 전보다 증가했는지는 테스트에서 확인됩니다.

복구된 사용자는 queue에 새로 들어간 것이 아닙니다. 기존 room ownership을 되찾은 것이므로 새 room을 만들거나 opponent를 다시 찾지 않습니다.

### `e593b1dd9fcd` — disconnect와 만료 처리

현재 active connection이 실제로 닫히면 다음 일이 일어납니다.

1. 참가 side를 `disconnectedUsers[side] = userId`로 예약합니다.
2. `RoomSession.disconnect`로 `reconnecting` 상태에 들어갑니다.
3. shared/per-room scheduler를 중지합니다.
4. 끊긴 paddle의 `dy`를 0으로 만들어 stale input을 멈춥니다.
5. snapshot phase를 paused로 보여줍니다.
6. reconnect deadline timer를 하나 예약합니다.

deadline이 끝났을 때 terminal path는 남은 참가자 수로 갈립니다.

| 만료 시 상태 | 처리 | persistence |
| --- | --- | --- |
| 한 side만 미복구 | 남은 side가 이기도록 score를 승리 점수로 만들고 정상 finalization 호출 | 등록 사용자 경기라면 결과 저장 |
| 양 side 모두 미복구 | scheduler/timer/room link를 정리하고 abandon | match row를 만들지 않음 |
| 이미 모두 복구 | timer가 할 일 없음 | 기존 경기 계속 |

한 side forfeit도 별도 임시 insert가 아니라 Thread 04의 idempotent finalization 경로를 사용합니다. result key가 room에 묶여 있으므로 timeout callback이 중복 실행되어도 DB 결과가 두 번 반영되지 않습니다.

`client.roomId`가 남아 있는 동안 새 queue 참가를 거부하는 것도 이 commit의 안전 장치입니다. reconnecting 사용자가 다른 room을 만들면 한 user가 두 room의 side를 동시에 소유하게 됩니다.

## 5. fake timer 테스트가 서로 다른 경로를 구분한다

### `113e39acc85c`

테스트는 세 가지 lifecycle을 별도로 만듭니다.

#### active replacement

- 같은 user로 새 socket을 연결합니다.
- old socket이 4001 `connection replaced`로 닫혔는지 확인합니다.
- 새 socket이 같은 room의 `queue.matched`와 snapshot을 받습니다.
- 15,001ms가 지나도 `finalizeMatch`가 호출되지 않습니다.

이는 replacement가 reconnect timeout을 시작하지 않는다는 직접 증거입니다.

#### deadline 직전 recovery

- 경기 중 snapshot sequence를 기록합니다.
- socket을 terminate하고 14,999ms 진행합니다.
- 같은 user의 새 socket을 연결합니다.
- 동일 room/side와 더 큰 snapshot sequence를 받습니다.
- deadline을 조금 넘겨도 forfeit가 실행되지 않습니다.

#### deadline expiry

- 양쪽이 참가한 경기에서 한쪽만 끊습니다.
- 14,999ms에는 finalization이 없습니다.
- 15,000ms에 정확히 한 번 `finalizeMatch`가 호출됩니다.
- 추가 60초를 진행해도 중복 호출이 없습니다.

이 테스트는 실제 네트워크 jitter나 browser tab lifecycle을 재현하지 않습니다. GameHub의 timer/state/ownership branch를 fake socket과 fake clock으로 고정합니다.

## 6. browser는 새 ticket으로 기존 room만 복구한다

### `4f5199097284`

server가 15초 예약을 제공해도 browser가 socket close 직후 `failed`로 끝나면 복구할 수 없습니다. `GameSocketClient`는 room을 보유한 상태에서 close되었을 때만 reconnect를 예약합니다.

재시도 정책은 다음과 같습니다.

```text
reconnect window: 15,000ms
initial delay: 250ms
exponential backoff: 250 → 500 → 1,000 → 2,000ms
maximum delay: 2,000ms
```

각 시도는 `/auth/ws-ticket`에서 **새 one-time ticket**을 발급받습니다. 기존 ticket을 재사용하지 않습니다. 그러나 reconnect socket이 open되어도 최초 `queue.join` event는 다시 보내지 않습니다.

```ts
// 최초 연결
openSocket(generation, initialEvent, handlers, false);

// 재연결
openSocket(generation, null, handlers, true);
```

queue command를 다시 보내면 server가 이미 예약된 room을 복구하는 동시에 새 매칭을 시도할 수 있습니다. browser reducer도 `roomId`가 있고 `reconnecting`인 동안 새 매칭 버튼을 막습니다.

connection generation은 사용자가 명시적으로 새 연결을 시작하거나 client를 닫을 때 증가합니다. 오래된 ticket request, timer, socket callback은 generation 비교로 무시됩니다. reconnect 성공 시 server가 보내는 `queue.matched`와 latest snapshot으로 room state를 복구합니다.

이 커밋에는 guest 결과 notice 등 다른 UI 변경도 섞여 있지만, 이 Thread에서는 reconnect client와 중복 매칭 방지만 다룹니다.

## 최종 ownership 모델

| 자원/상태 | owner | ownership 종료 |
| --- | --- | --- |
| 현재 user socket | `clientsByUser[userId]` | replacement 또는 실제 close |
| room side | room의 `clients[side]` | disconnect 시 reservation으로 이전, 경기 종료/abandon 시 해제 |
| 끊긴 side 예약 | `disconnectedUsers` + `RoomSession` | reconnect 성공 또는 15초 만료 |
| reconnect timer | room/GameHub | 모든 side 복구, 경기 종료, abandon, close |
| browser reconnect loop | `GameSocketClient` generation | 성공, 15초 초과, 명시적 close/new connect |

최종 불변 조건은 다음과 같습니다.

- 사용자마다 active socket owner는 하나입니다.
- active replacement는 room ownership을 즉시 이전하고 forfeit timer를 만들지 않습니다.
- 실제 disconnect는 side를 사용자 ID로 최대 15초 예약합니다.
- 복구 시 새 socket은 같은 room/side와 latest authoritative snapshot을 받습니다.
- 한 명만 돌아오지 않으면 forfeit, 둘 다 돌아오지 않으면 결과 없이 abandon합니다.
- browser reconnect는 새 ticket을 쓰며 최초 queue command를 반복하지 않습니다.

이 Thread는 room 종료 뒤 DB 반영의 원자성을 직접 구현하지 않습니다. forfeit를 포함한 persistence는 Thread 04에 의존합니다. exact 15초 equality에서 timer와 reconnect callback의 실행 순서를 완전히 직렬화하지 않는 제한도 남아 있습니다.
