# Thread: 결정적 simulation, 시간 보정, snapshot 전달 압력

실시간 게임 서버에서는 “같은 입력이면 같은 상태가 나온다”와 “그 상태가 모든 client에게 즉시 전달된다”가 서로 다른 문제입니다. 이 Thread는 authoritative simulation의 결정성, event-loop 지연을 tick 수로 바꾸는 fixed-step scheduler, 최신 snapshot만 보존하는 전달 buffer를 각각 검증하고, 마지막으로 **send callback 지연을 congestion으로 오판한 설계**를 수정합니다.

```text
입력 + seed + timestep
        ↓
결정적인 authoritative state
        ↓ scheduler가 허용한 tick 수
최신 snapshot projection
        ↓ transport pressure 정책
client delivery
```

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4ef4beeb8611a17f47764065a9b4da2fc16cd463` | `test(game): 결정적 simulation 검증` | A | `SIMULATION, REALTIME, TEST` | seed·입력·delta가 같을 때 AI와 simulation이 같은 결과를 내는지 고정한다. |
| 2 | `0888e119036df23deeb961b9e26da305310da391` | `test(game): fixed-step 보정 범위 검증` | B | `SIMULATION, REALTIME, TEST` | wall-clock 지연을 bounded tick 수로 변환하는 scheduler 경계를 검증한다. |
| 3 | `125aa113a01c38605372c490637e1bfa4e86a009` | `test(game): snapshot replacement와 congestion 검증` | A | `REALTIME, PERF, RISK` | 최신 snapshot 교체, soft/hard pressure, 장기 congestion 종료 정책을 검증한다. |
| 4 | `37a7b2e4611b02d077e98c0e136f2b97313ff807` | `test(game): versioned match replay fixture 추가` | A | `SIMULATION, REALTIME, TEST` | seed·초기 상태·입력 sequence를 v1 fixture로 보존하고 최종 state hash를 비교한다. |
| 5 | `d90f17fa765d4e9548b8f52c850b7605698644eb` | `fix(game): callback 지연을 snapshot congestion으로 오판하지 않음` | A | `REALTIME, PERF, RISK` | callback 미도착과 socket buffer pressure를 분리한다. |
| 6 | `5cd54767858f60f0f10b9c86e680a9922c361fac` | `test(game): callback 지연과 실제 congestion 구분` | A | `REALTIME, PERF, TEST` | delayed callback과 높은 `bufferedAmount`가 서로 다른 결과를 내는지 회귀 검증한다. |

## 1. 결정성은 특정 출력 하나가 아니라 state transition의 성질이다

### `4ef4beeb8611...` — AI와 물리 simulation의 반복 가능성

이 commit은 AI와 Pong simulation을 같은 문제로 뭉뚱그리지 않고 서로 다른 결정성 원인을 검증합니다.

#### AI

- 같은 seed에서 생성한 PRNG stream은 같은 값을 냅니다.
- 같은 state와 같은 random stream을 받은 AI는 같은 paddle direction을 선택합니다.
- AI source에 `Math.random`이나 시간에 따라 흔들릴 수 있는 임의성이 다시 들어오지 않는지 source guard를 둡니다.

#### simulation

- 같은 초기 state, input, delta를 적용하면 같은 next state가 나옵니다.
- `step`이 caller가 넘긴 이전 state/input을 제자리에서 변형하지 않는지 확인합니다.
- delta scaling과 허용 범위 밖 delta 처리를 검증합니다.
- score가 종료 조건인 3점에 도달하면 winner/finished state가 결정됩니다.
- 0이나 `NaN` 같은 invalid delta는 정상 tick으로 처리되지 않습니다.
- 1,000 tick을 실행한 뒤 state hash를 비교해 작은 부동소수점·순서 회귀도 검출합니다.

이 테스트가 고정하는 것은 “현재 공 위치가 이 숫자다” 하나가 아닙니다.

```text
nextState = F(previousState, input, delta, deterministic random stream)
```

이 함수 관계가 process 재시작, replay, client projection과 무관하게 동일해야 한다는 계약입니다.

다만 이 commit은 JavaScript engine·architecture를 완전히 넘나드는 수치 동일성을 증명하지 않습니다. 같은 코드와 fixture에서 회귀를 잡는 repository-level 결정성 증거입니다.

### `37a7b2e4611b...` — 결정성을 history로 재생할 수 있게 만든다

unit test 안에 hard-coded loop만 있으면 입력 sequence가 바뀌었을 때 무엇이 재현 대상인지 리뷰하기 어렵습니다. 이 commit은 versioned replay fixture를 추가합니다.

fixture에는 다음 정보가 들어 있습니다.

```text
format version: 1
seed
50ms timestep
초기 state
1,000 tick 동안 적용할 input sequence
expected final state hash: f0a9...
```

테스트는 fixture를 읽어 initial state에서 동일한 순서로 simulation을 실행하고 마지막 hash를 비교합니다. 이 구조는 bug report나 과거 match를 동일한 입력 stream으로 재생하는 기반이 됩니다.

중요한 한계도 있습니다. 해당 test는 fixture를 TypeScript shape로 취급해 replay하며, 별도의 runtime JSON schema validation을 새로 도입한 commit은 아닙니다. 따라서 “깨진 replay 파일을 안전하게 거부한다”보다 “version 1 fixture의 semantic result가 유지된다”는 증거로 읽어야 합니다.

## 2. wall clock을 그대로 simulation에 넣지 않는다

### `0888e119036...` — fixed-step의 경계값

실제 timer는 정확히 50ms마다 실행되지 않습니다. event loop가 늦어지더라도 simulation timestep을 임의로 늘리면 collision과 AI behavior가 wall-clock jitter에 의존하게 됩니다. fixed-step scheduler는 지연 시간을 50ms 단위의 정수 tick으로 바꿉니다.

테스트는 경계 바로 전후를 사용합니다.

| 현재 시각 예 | due tick | 의미 |
| ---: | ---: | --- |
| 1049ms | 0 | 다음 50ms 경계 전 |
| 1050ms | 1 | 정확히 한 tick 도달 |
| 1149ms | 1 | 남은 99ms 중 한 tick만 소비 |
| 1150ms | 1 | 다음 호출에서 다시 한 tick |

장시간 멈춘 뒤 모든 지연 tick을 한 번에 실행하는 것도 위험합니다. 10초 pause 뒤 수백 tick을 catch-up하면 request 처리와 snapshot delivery가 더 늦어지는 spiral이 생깁니다. 테스트는 누적 보정 범위를 250ms, 즉 50ms tick 5개로 제한합니다.

```text
10초 지연
  → 실행 허용: 5 ticks
  → 오래된 나머지 backlog: 버림
  → 다음 cycle은 정상 cadence에서 재개
```

또한 clock이 뒤로 움직이는 사례와 timer stop/cleanup도 확인합니다. 이 commit은 scheduler 구현을 새로 설명하는 핵심 feature라기보다 이미 선택된 bounded catch-up policy의 경계값을 고정하므로 B급입니다.

## 3. snapshot buffer는 모든 과거 state를 보존하지 않는다

### `125aa113a01c...` — latest-only 전달 정책

simulation tick이 transport보다 빠를 때 snapshot 1, 2, 3을 모두 queue하면 client는 이미 의미가 사라진 과거 state를 늦게 받습니다. latest snapshot buffer는 “미전송 상태 중 가장 최신 것 하나”만 남깁니다.

초기 테스트가 표현한 대표 흐름은 다음과 같습니다.

```text
snapshot 1 send 시작, callback 미도착
snapshot 2 도착 → pending = 2
snapshot 3 도착 → pending = 3 (2 대체)
callback 도착 → 3 전송
관찰 sequence: 1, 3
```

여기서 snapshot 2를 버리는 것은 데이터 손실 bug가 아니라 의도된 coalescing입니다. authoritative simulation state는 계속 진행되며 client는 가장 최신 projection으로 따라잡습니다.

같은 suite는 socket pressure를 두 단계로 나눕니다.

| 상태 | 기준/시간 | 처리 |
| --- | --- | --- |
| 정상 | `bufferedAmount`가 soft threshold 이하 | 최신 snapshot 전송 |
| 일시 soft pressure | threshold 초과 | 최신 pending 하나만 보존, 50ms 뒤 재확인 |
| 지속 soft pressure | 약 5초 계속 | 더 이상 회복 가능 connection으로 보지 않고 종료 |
| hard pressure | 1 MiB 수준 | 즉시 terminate |

이 정책은 무한 queue보다 안전하지만, 초기 구현에는 중요한 잘못된 가정이 숨어 있었습니다. **send callback이 아직 오지 않았다는 사실을 socket congestion의 일부로 사용했다는 점**입니다.

## 4. `d90f17fa765d...` — callback completion은 pressure metric이 아니다

### 잘못된 가정

Node `ws.send(payload, callback)`의 callback이 늦는 이유는 transport buffer가 찼기 때문일 수도 있지만, test double이나 event-loop scheduling, callback dispatch 순서 때문일 수도 있습니다. 반면 `bufferedAmount`는 실제로 socket이 아직 밀어내지 못한 byte 수를 나타내는 pressure 신호입니다.

초기 buffer는 개념적으로 다음 두 조건을 함께 사용했습니다.

```text
sending == true       → 이전 send가 끝나지 않음 → pending으로 교체
bufferedAmount 높음   → transport pressure       → pending으로 교체
```

이 둘을 같게 취급하면 callback만 지연되는 healthy socket에서 snapshot 2가 불필요하게 버려지고 전달 cadence가 callback 구현에 종속됩니다.

### 수정

fix는 callback-driven `sending` gate를 제거하고 pressure 판단을 `bufferedAmount`로 수렴시킵니다.

```diff
- if (sending) {
-   pending = snapshot;
-   return;
- }
-
- sending = true;
- socket.send(payload, () => {
-   sending = false;
-   drainPending();
- });
+ socket.send(payload);
```

실제 source의 세부 구조보다 중요한 state 변화는 다음과 같습니다.

```text
callback pending + bufferedAmount == 0
    → congestion 아님
    → snapshot 1, 2, 3 모두 send 호출 가능

bufferedAmount > soft limit
    → 실제 pressure
    → 2, 3 중 최신 3만 pending
```

이 변경은 coalescing 정책을 없애지 않습니다. **coalescing을 발동시키는 관찰값만 올바른 transport signal로 교체**합니다.

### 왜 root-cause fix인가

- callback을 더 빨리 호출하도록 timeout을 추가하지 않습니다.
- pending queue 크기를 늘리지 않습니다.
- delivery frequency를 낮추지 않습니다.
- “busy” representation 자체에서 callback state를 제거합니다.

따라서 callback semantics가 달라져도 pressure policy는 같은 의미를 유지합니다.

## 5. `5cd54767858f...` — 두 상태를 같은 test double에서 분리한다

후속 회귀 테스트는 source를 읽는 assertion이 아니라 behavior를 직접 비교합니다.

### delayed callback, pressure 없음

fake socket의 send callback을 의도적으로 호출하지 않되 `bufferedAmount = 0`으로 유지합니다. snapshot 1, 2, 3을 넣으면 세 payload 모두 send 기록에 나타나야 합니다.

```text
callback queue: [미완료, 미완료, 미완료]
socket bufferedAmount: 0
sent snapshots: 1, 2, 3
```

### 실제 soft pressure

`bufferedAmount`를 soft limit 위로 올리고 snapshot 2와 3을 넣습니다. 이때 둘 다 즉시 보내지 않고 pending은 3으로 교체됩니다. pressure를 해제하고 50ms retry를 진행하면 3만 전송됩니다.

```text
bufferedAmount: high
incoming: 2 → 3
pending: 3
bufferedAmount: 0 + retry
sent: 3
```

이 pair가 중요한 이유는 이전 구현도 “congestion에서 latest-only” test는 통과할 수 있었기 때문입니다. bug는 실제 congestion이 아니라 **congestion이 아닌 상태를 congestion으로 분류한 것**이므로, negative classification test가 필요했습니다.

## 최종 상태와 책임 분리

| 책임 | 입력 | 결정 기준 | 결과 |
| --- | --- | --- | --- |
| `PongSimulation` | state, input, fixed delta | 순수 transition 규칙 | authoritative next state |
| AI | state, seeded random stream | deterministic policy | direction |
| fixed-step scheduler | wall-clock elapsed | 50ms 단위, 최대 250ms 보정 | 이번 cycle tick 수 |
| replay test | versioned fixture | seed/input 순서와 final hash | history 회귀 증거 |
| snapshot buffer | 최신 projection | `bufferedAmount`, soft/hard 시간 | send/coalesce/terminate |
| send callback | library completion 알림 | pressure 판정에 사용하지 않음 | delivery correctness와 분리 |

최종적으로 보장되는 것은 다음과 같습니다.

- 같은 seed·초기 state·입력·timestep은 같은 simulation result를 만듭니다.
- event-loop 지연은 무제한 catch-up으로 전환되지 않습니다.
- 전달 backlog는 과거 snapshot 전체가 아니라 최신 projection 하나로 제한됩니다.
- callback 지연만으로 snapshot을 버리지 않습니다.
- 실제 socket buffer pressure가 있을 때만 coalescing과 종료 정책이 작동합니다.

## 이 Thread의 경계

- room 전체의 snapshot broadcast cadence를 20Hz simulation/10Hz delivery로 분리한 변경은 Thread 07의 `ad482c...`입니다. 그 후대 결정을 이 Thread의 과거 commit에 소급하지 않습니다.
- reconnect 뒤 어떤 snapshot을 다시 보내는지는 Thread 04입니다.
- versioned wire schema 일반 규칙은 Thread 01입니다.
- 실제 k6에서 snapshot delay/drop을 측정하는 방식은 Thread 07입니다.

## 조사·실행 기록

각 commit의 exact diff와 해당 SHA의 test/fixture/source를 확인했습니다. 이 작성 환경에서는 unit test나 replay를 실행하지 않았으므로 hash 일치·suite 통과를 새 실행 결과로 주장하지 않습니다.
