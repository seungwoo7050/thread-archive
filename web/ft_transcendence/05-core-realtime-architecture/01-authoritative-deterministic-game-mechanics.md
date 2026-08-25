# 서버 권위 물리에서 결정적 replay 계약까지

이 Thread의 핵심은 “서버에서 공을 움직인다”가 아닙니다. **현재 상태, 이번 tick의 입력, 고정 규칙이 같으면 다음 상태도 같아야 하며, 그 규칙의 유일한 구현은 simulation이어야 한다**는 조건을 만드는 과정입니다.

초기 구현은 `GameHub` 안에서 공 위치와 충돌을 직접 계산했습니다. 이후 상태 표현과 전이 함수를 분리하고, 득점·속도 증가·종료 판정까지 하나의 `PongSimulation.step`으로 모았습니다. 마지막에는 GameHub의 중복 계산을 지우고, 고정 입력 fixture의 결과 digest를 versioned golden 값으로 고정했습니다.

## Commit map

| SHA | 제목 | Importance | Tags | Thread에서의 역할 |
| --- | --- | :---: | --- | --- |
| `9e3664f5de48` | `feat(game): 서버 주도 퐁 물리 갱신` | A | `SIMULATION, REALTIME, WEB` | GameHub가 authoritative state를 직접 갱신하는 최초 경로 |
| `2e4359f0625f` | `refactor(game): Pong simulation 상태와 초기화 분리` | A | `SIMULATION, REALTIME, REFACTOR` | 상태 표현·초기화·복사를 별도 simulation 모듈로 이동 |
| `4afec2071e7a` | `refactor(game): 득점과 충돌을 simulation에 통합` | S | `SIMULATION, CORE, ARCH` | 한 tick의 전체 규칙과 종료 판정을 하나의 전이 함수로 확정 |
| `4ef4beeb8611` | `test(game): 결정적 simulation 검증` | A | `SIMULATION, REALTIME, TEST` | 불변성·동일 입력 재실행의 동일 결과를 검증 |
| `cf14c4052310` | `refactor(game): GameHub frame 계산을 simulation에 위임` | S | `SIMULATION, ARCH, REALTIME` | production tick이 simulation 결과만 사용하도록 권위 이전 |
| `2cef070188ac` | `refactor(game): GameHub의 중복 물리 계산 제거` | B | `SIMULATION, REALTIME` | 이전 inline 규칙과 helper를 삭제해 두 번째 권위 제거 |
| `37a7b2e4611b` | `test(game): versioned match replay fixture 추가` | A | `SIMULATION, REALTIME, TEST` | 1,000개 입력 replay의 exact digest를 protocol-version과 함께 고정 |

## 1. 서버가 상태를 소유했지만, 규칙은 아직 GameHub에 섞여 있었다

### `9e3664f5de48` — 최초 authoritative tick

이 커밋은 클라이언트가 보낸 좌표를 받아들이는 대신, 서버의 room snapshot을 직접 변경하는 `tick` 경로를 추가했습니다. 패들 입력은 서버가 보유한 방향값으로 반영되고, 공은 서버에서 이동·벽 반사·패들 충돌·득점을 계산한 뒤 snapshot으로 방송됩니다.

중요한 제한이 있습니다. 이 SHA의 diff는 **tick 계산 함수 자체**를 만들지만, 그 함수를 일정 주기로 호출하는 scheduler까지 함께 추가하지는 않습니다. 따라서 이 시점에서 확정되는 것은 “서버가 계산할 규칙과 상태를 갖는다”는 사실이지, 실제 runtime이 이미 안정적인 20Hz loop를 돌린다는 사실이 아닙니다.

또한 물리 계산, snapshot mutation, 방송, 경기 종료가 모두 `GameHub`에 붙어 있습니다. authoritative owner는 생겼지만, 규칙을 독립적으로 재실행하거나 비교하기는 어렵습니다.

## 2. 상태를 분리한 뒤, 한 tick의 의미를 완결했다

### `2e4359f0625f` — 표현과 전이의 분리 준비

`PongSimulationState`와 `initialState`, 상태 복사 로직이 `pongSimulation.ts`로 이동합니다. 이 단계의 핵심은 “GameHub의 여러 필드를 직접 바꾸는 방식”에서 “하나의 simulation state를 입력받아 다음 state를 만드는 방식”으로 옮길 자리를 만든 것입니다.

이 커밋만으로 규칙의 권위가 이동한 것은 아닙니다. 아직 핵심 `step`이 완성되지 않았기 때문에, 상태 모듈은 representation boundary에 가깝습니다.

### `4afec2071e7a` — 한 tick의 전체 전이

이 커밋에서 `PongSimulation.step`이 득점과 충돌을 포함한 전체 transition을 소유합니다. 처리 순서는 결과에 영향을 주므로 그 자체가 계약입니다.

```text
입력 state 복사
→ tick 증가
→ 양쪽 paddle 이동 및 court 범위 clamp
→ ball 위치 적분
→ 상·하단 벽 반사
→ 좌·우 paddle 충돌 판정과 속도 변경
→ 좌·우 득점 및 ball reset
→ rally 중 속도 증가와 최대 속도 clamp
→ 승리 점수 또는 시간 제한에 따른 종료 판정
```

핵심 형태는 다음과 같습니다.

```ts
const next = cloneState(state);
next.tick += 1;

movePaddles(next, inputs);
moveBall(next);
reflectVerticalWalls(next);
resolvePaddleCollisions(next);
resolveScore(next);
accelerateBall(next);

if (reachedScoreOrTimeLimit(next)) {
  next.phase = "finished";
  next.winnerSide = next.leftScore >= next.rightScore ? "left" : "right";
}
return next;
```

실제 코드에서는 paddle 충돌 시 x 방향을 뒤집고 속도를 `1.04`배로 증가시키며, paddle 중심에서의 충돌 오프셋으로 y 속도를 조정합니다. 공 속도는 상한을 넘지 않도록 clamp됩니다. 득점 시에는 score를 올리고 공을 중앙으로 되돌리며, 재시작 속도도 제한된 boost를 적용합니다.

이 결정의 효과는 세 가지입니다.

1. **입력 state를 직접 바꾸지 않습니다.** 반환 state가 다음 authoritative state입니다.
2. **득점·충돌·종료가 같은 규칙 집합에 있습니다.** 한쪽만 GameHub에 남아 divergence를 만들지 않습니다.
3. **같은 transition을 test와 production이 호출할 수 있습니다.** 결정성을 코드 경로 단위로 확인할 수 있습니다.

승리 조건도 simulation state 안에서 결정됩니다. 점수 도달 또는 45초 tick 제한에 이르면 `phase = "finished"`와 `winnerSide`가 next state에 기록되므로, GameHub가 같은 score를 보고 별도의 규칙으로 승자를 다시 계산할 필요가 없습니다.

## 3. “두 번 같음”과 “과거와 같음”은 다른 검증이다

### `4ef4beeb8611` — transition 성질 검증

이 테스트는 다음을 고정합니다.

- `step`이 입력 state를 mutate하지 않는다.
- 같은 초기 state와 같은 입력을 반복하면 같은 결과가 나온다.
- 장시간 반복해도 두 독립 실행의 state hash가 같다.

다만 1,000 tick 검사는 이 시점에 **정해진 golden digest**를 요구하지 않습니다. 두 번 실행한 digest가 서로 같고 hash 형태가 올바른지만 검사합니다. 따라서 이 테스트가 증명하는 것은 동일 구현 안의 재현성입니다. 물리 상수나 처리 순서가 바뀌어 두 실행이 함께 달라지는 회귀까지는 잡지 못합니다.

### `37a7b2e4611b` — versioned golden replay

마지막 커밋은 고정된 1,000개 방향 입력을 replay하고 결과를 다음 digest와 비교합니다.

```text
protocolVersion: 1
timestepMs: 50
expectedDigest: f0a9d7a9f2453620dac1c8a718b001bb939b4a546ec1ca192da89f329d2a3f61
```

fixture의 `seed`는 replay 자료의 metadata입니다. 테스트가 seed로 입력을 새로 생성하는 것이 아니라, repository에 기록된 방향 sequence를 그대로 재생합니다. 그러므로 이 테스트는 “동일 seed의 난수 생성기”를 검증하는 것이 아니라 **기록된 입력과 v1 규칙의 호환성**을 고정합니다.

이제 두 종류의 회귀가 분리됩니다.

| 검증 | 잡는 문제 | 잡지 못하는 문제 |
| --- | --- | --- |
| 동일 입력을 두 번 실행해 비교 | 비결정적 상태 전이, 입력 state mutation | 규칙이 일관되게 바뀌는 semantic drift |
| versioned golden digest 비교 | 처리 순서·상수·reset 규칙의 변경 | 다른 JS 엔진까지 포함한 보편적 bit-level 동일성 |

## 4. production의 권위를 simulation으로 옮기고 이전 규칙을 삭제했다

### `cf14c4052310` — GameHub는 orchestration만 담당

GameHub의 frame 계산이 `PongSimulation.step` 호출로 바뀝니다. room이 보유한 simulation state와 현재 paddle input을 넘기고, 반환 state를 snapshot에 반영한 뒤 방송합니다. `winnerSide`가 있으면 GameHub는 경기 확정 경로를 시작합니다.

```ts
room.simulation = PongSimulation.step(room.simulation, {
  left: room.snapshot.paddles.left.dy,
  right: room.snapshot.paddles.right.dy
}, SIMULATION_TIMESTEP_MS);

syncSnapshot(room);
this.broadcastRoom(room.id, { type: "game.snapshot", snapshot: room.snapshot });
if (room.simulation.phase === "finished" && room.simulation.winnerSide) {
  await this.finishRoom(room, room.simulation.winnerSide);
}
```

핵심은 GameHub가 더 이상 공의 위치나 충돌 공식을 알 필요가 없다는 점입니다. GameHub의 책임은 다음으로 줄어듭니다.

- runnable room의 현재 입력을 수집합니다.
- simulation transition을 한 번 호출합니다.
- 반환 state를 wire snapshot으로 반영합니다.
- 종료 신호를 persistence/finalization 경로로 넘깁니다.

### `2cef070188ac` — 중복 구현 제거

위임만 하고 이전 helper를 남겨두면, 나중에 누군가 잘못된 경로를 다시 호출하거나 두 구현을 각각 수정할 수 있습니다. 이 커밋은 GameHub 안의 inline collision·score·AI 보조 계산과 관련 상수를 삭제합니다.

작은 diff이지만 의미는 명확합니다. `PongSimulation.step`이 **유일한 물리 규칙 구현**이 되며, production과 replay test가 같은 함수를 통과합니다.

## 최종 불변 조건

```text
(authoritative state, input at tick N, v1 rules)
                    │
                    ▼
          PongSimulation.step
                    │
                    ▼
   next authoritative state
   (phase / winnerSide 포함)
                    │
          GameHub orchestration
                    │
            versioned snapshot / finalization
```

이 Thread가 최종적으로 보장하는 것은 다음과 같습니다.

- 클라이언트는 좌표를 확정하지 않고 방향 입력만 보냅니다.
- authoritative game state와 승자 판정은 서버 simulation 결과입니다.
- 한 tick의 규칙은 `PongSimulation.step` 한 곳에만 있습니다.
- production과 replay가 같은 transition을 사용합니다.
- v1 fixture의 기록된 입력은 exact digest로 semantic drift를 드러냅니다.

범위 밖도 분명합니다. scheduler의 wall-clock 보정과 overload 제한은 Thread 08, wire schema와 sequence는 Thread 03, 결과의 DB 원자성은 Thread 04가 담당합니다. 또한 golden digest는 현재 구현·runtime 조합의 호환성 신호이지, 모든 JavaScript 엔진에서 부동소수점 결과가 영원히 동일하다는 증명은 아닙니다.
