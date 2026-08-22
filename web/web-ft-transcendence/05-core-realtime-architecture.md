===== BEGIN FILE: 01-authoritative-deterministic-game-mechanics.md =====
# 서버 권위형 결정적 게임 메커니즘

원문 Development Thread: `Authoritative and deterministic game mechanics`

## 1. Thread 목표

- `GameHub` 내부 물리 계산에서 출발해 `PongSimulation.step`이 유일한 규칙 소유자가 되는 과정을 복원합니다.
- 점수, 충돌, 가속, 종료 결과가 transport·socket·timer와 분리된 결정적 상태 전이임을 해당 SHA의 코드와 테스트로 증명합니다.
- 마이그레이션 뒤 남아 있던 중복 물리 구현이 실제로 제거되고 장기 replay fixture가 호환성 계약을 고정하는지 확인합니다.

### Source에서 확정된 significance

> The project begins with server-owned gameplay, then extracts the mechanics into a pure, testable simulation and finally removes the duplicate implementation. The thread matters because authority is not merely a deployment choice: one deterministic state transition becomes the source of scores, collisions, timing, and replay compatibility.

### 직접 연결되는 Critical Invariants

> The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes.

### 직접 연결되는 Major Engineering Difficulties

> Separating deterministic simulation from connection orchestration without changing the observable realtime protocol.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 각 SHA에서 게임 규칙의 실제 소유자는 `GameHub`와 `PongSimulation` 중 어디입니까?
- simulation 입력, 내부 상태, 반환 상태 사이에 aliasing이나 in-place mutation이 남아 있습니까?
- paddle 충돌, 득점, serve reset, 속도 증가, 시간 제한, 승자 결정은 어떤 순서로 적용됩니까?
- wire snapshot을 유지하면서 mechanics ownership만 이동한 연결 지점은 어디입니까?
- 결정성은 단일 함수 테스트가 아니라 seed, AI, nested state, 1,000-tick replay까지 어떻게 고정됩니까?
- legacy helper가 제거된 뒤 `GameHub`에 남은 책임은 무엇입니까?

## 3. 완료 기준

- 입력 수집 → `PongSimulation.step` → 상태 저장 → snapshot projection → broadcast/finish 흐름을 코드 근거와 함께 그릴 수 있습니다.
- 충돌·득점·가속·종료 규칙을 해당 SHA의 실제 조건식과 상태 갱신 순서로 설명할 수 있습니다.
- immutability와 deterministic replay가 각각 무엇을 검증하고 무엇을 검증하지 않는지 구분할 수 있습니다.
- 마이그레이션 전후 `GameHub`와 simulation의 책임 표를 완성하고 중복 규칙이 남지 않았음을 확인할 수 있습니다.
- final HEAD가 아니라 각 commit 시점의 코드와 직전 관련 SHA를 근거로 설명할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `9e3664f5de48` | `feat(game): 서버 주도 퐁 물리 갱신` | A | SIMULATION, REALTIME, WEB | Introduces the first server-owned physics loop. |
| 2 | `2e4359f0625f` | `refactor(game): Pong simulation 상태와 초기화 분리` | A | SIMULATION, REALTIME, REFACTOR | Creates a transport-independent simulation state boundary. |
| 3 | `4afec2071e7a` | `refactor(game): 득점과 충돌을 simulation에 통합` | S | SIMULATION, CORE, ARCH | Moves complete scoring and terminal mechanics into one deterministic transition. |
| 4 | `4ef4beeb8611` | `test(game): 결정적 simulation 검증` | A | SIMULATION, REALTIME, TEST | Locks immutability, seed repeatability, and replay digest behavior. |
| 5 | `cf14c4052310` | `refactor(game): GameHub frame 계산을 simulation에 위임` | S | SIMULATION, ARCH, REALTIME | Makes GameHub consume the standalone authoritative transition. |
| 6 | `2cef070188ac` | `refactor(game): GameHub의 중복 물리 계산 제거` | B | SIMULATION, REALTIME | Deletes the competing legacy rules after migration. |
| 7 | `37a7b2e4611b` | `test(game): versioned match replay fixture 추가` | A | SIMULATION, REALTIME, TEST | Turns long-run deterministic behavior into a versioned compatibility fixture. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): 서버 주도 퐁 물리 갱신`

| 항목 | 값 |
| --- | --- |
| SHA | `9e3664f5de48` |
| Importance | A |
| Tags | SIMULATION, REALTIME, WEB |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Introduces the first server-owned physics loop.
- Classification summary: Implement the authoritative simulation step inside `GameHub`.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | parent에서는 서버가 room과 snapshot을 보유하지만 한 frame의 물리 규칙을 수행하는 `tick` 구현이 없습니다. |
| 핵심 boundary/decision | `apps/api/src/gameHub.ts`의 `tick`, `collidePaddle`, `resetBall`이 `room.snapshot`을 직접 변경합니다. phase가 `playing`일 때만 tick/serverTime을 올리고 paddle·AI·ball·score를 순서대로 갱신합니다. |
| 상태 또는 ownership 변화 | 이 SHA의 규칙 owner는 `GameHub`입니다. browser는 direction만 보내며 충돌·득점·serve reset 결과는 서버 snapshot에서 나옵니다. |
| 주요 failure/edge path | arena clamp와 wall reflection이 위치 이탈을 제한하고, court 바깥으로 나간 ball은 score 증가 뒤 center serve로 재설정됩니다. 다만 diff에는 `tick`을 호출하는 scheduler가 없어 실행 연결은 아직 보장하지 않습니다. |
| 보장/비보장 | 서버 내부에 하나의 authoritative physics transition이 생겼다는 사실만 보장합니다. timer lifecycle, 독립 simulation, 장기 replay는 보장하지 않습니다. |
| 다음 관련 commit 연결 | `2e4359...`가 state 초기화와 clone을 transport-independent `PongSimulation`으로 옮길 토대를 만듭니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `2e4359f0625f` — `refactor(game): Pong simulation 상태와 초기화 분리`

### 5.2. `refactor(game): Pong simulation 상태와 초기화 분리`

| 항목 | 값 |
| --- | --- |
| SHA | `2e4359f0625f` |
| Importance | A |
| Tags | SIMULATION, REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Creates a transport-independent simulation state boundary.
- Classification summary: Extract simulation state, input types, initialization, and cloning from transport concerns.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 게임 상태는 `GameHub`의 wire snapshot과 결합되어 있었고 독립적으로 초기화·복제할 경계가 없었습니다. |
| 핵심 boundary/decision | 새 `apps/api/src/game/pongSimulation.ts`에 `PongSimulationState`, `PongSimulationInputs`, `initialState`, `cloneState`를 추가합니다. |
| 상태 또는 ownership 변화 | state shape와 초기화 책임이 simulation module로 이동하지만 이 commit 자체에는 완전한 `step` mechanics가 아직 없습니다. |
| 주요 failure/edge path | nested paddles와 ball을 별도 객체로 복제해 caller state aliasing의 기초 위험을 제거합니다. 이 시점에는 충돌·득점 규칙 중복이 여전히 남습니다. |
| 보장/비보장 | transport 없이 simulation state를 만들고 복제할 수 있습니다. GameHub가 이를 사용하거나 rule owner가 바뀌었다고까지 보장하지 않습니다. |
| 다음 관련 commit 연결 | `4afec2...`가 득점·충돌·가속·종료를 `PongSimulation.step`에 통합합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `9e3664f5de48` — `feat(game): 서버 주도 퐁 물리 갱신`
- 다음 Thread 관련 SHA: `4afec2071e7a` — `refactor(game): 득점과 충돌을 simulation에 통합`

### 5.3. `refactor(game): 득점과 충돌을 simulation에 통합`

| 항목 | 값 |
| --- | --- |
| SHA | `4afec2071e7a` |
| Importance | S |
| Tags | SIMULATION, CORE, ARCH |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Moves complete scoring and terminal mechanics into one deterministic transition.
- Classification summary: Complete the standalone deterministic state transition.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | state boundary는 생겼지만 실제 physics는 여전히 GameHub와 simulation 사이에 분산될 수 있는 상태였습니다. |
| 핵심 boundary/decision | `PongSimulation.step(state, inputs, deltaMs)`가 clone된 state에 paddle 이동, wall/paddle collision, scoring, serve reset, acceleration, terminal 판정을 순서대로 적용합니다. |
| 상태 또는 ownership 변화 | 게임 규칙과 terminal result 계산이 pure simulation transition의 책임이 됩니다. input/state는 caller가 소유하고 반환 state는 새 객체입니다. |
| 주요 failure/edge path | invalid delta는 거부하고 paddle을 arena에 clamp합니다. 속도는 tick당 0.015씩 증가하되 18에서 제한하며, 45초 tick limit 또는 `WINNING_SCORE`에서 종료합니다. 시간 제한 동점은 left를 선택합니다. |
| 보장/비보장 | 동일 state/input/delta는 동일 next state를 만들고 rule ordering이 한 함수에 모입니다. GameHub가 아직 이 함수를 실제 frame에 사용한다는 보장은 다음 commit 전에는 없습니다. |
| 다음 관련 commit 연결 | `4ef4be...`가 immutability·delta scaling·seed repeatability·1,000-step digest를 검증하고, `cf14c4...`가 runtime caller를 전환합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | 충돌·득점·가속·종료가 transport orchestration과 함께 있으면 단위 검증과 replay compatibility를 독립적으로 보장할 수 없습니다. |
| 실패 위험 | GameHub와 별도 simulation이 서로 다른 규칙을 계산하거나 caller state를 in-place 변경하면 서버 내부에서도 결과가 갈라집니다. |
| 핵심 결정 | 모든 mechanics를 하나의 clone-first `PongSimulation.step`에 집중합니다. |
| 구현 경로 | 입력 방향 적용 → paddle clamp → ball 적분 → wall/paddle 반사 → score/reset → acceleration → winning/time-limit 판정. |
| 수명주기·상태 | simulation state는 호출 전 state와 반환 state로 구분되고 socket·timer·repository resource를 소유하지 않습니다. |
| 실패 처리 | 비정상 delta는 즉시 `RangeError`; 위치·속도는 명시적 bounds; terminal에서는 양쪽 direction을 0으로 고정합니다. |
| 후속 검증 | `4ef4be...` deterministic unit test와 `37a7b2...` versioned replay fixture가 후속 보호를 제공합니다. |
| Thread 전체 의미 | 이후 GameHub는 rule engine이 아니라 input/AI 수집, scheduling, projection, persistence를 조정하는 계층이 될 수 있습니다. |

비교 기준:
- 직전 Thread 관련 SHA: `2e4359f0625f` — `refactor(game): Pong simulation 상태와 초기화 분리`
- 다음 Thread 관련 SHA: `4ef4beeb8611` — `test(game): 결정적 simulation 검증`

### 5.4. `test(game): 결정적 simulation 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `4ef4beeb8611` |
| Importance | A |
| Tags | SIMULATION, REALTIME, TEST |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Locks immutability, seed repeatability, and replay digest behavior.
- Classification summary: Add deterministic unit and regression tests for simulation and AI.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | `PongSimulation.step`의 설계 의도는 코드에 존재하지만 동일 입력 반복, source mutation, 장기 drift를 자동 검출할 증거가 없었습니다. |
| 핵심 boundary/decision | `pongSimulation.test.ts`와 `pongAi.test.ts`가 동일 입력 deep equality, 입력 불변성, 25/50ms scaling, bounds, win, invalid delta, seeded PRNG/AI, 1,000-tick SHA-256을 고정합니다. |
| 상태 또는 ownership 변화 | production state owner는 바뀌지 않으며 test fixture가 deterministic contract의 회귀 감시자 역할을 맡습니다. |
| 주요 failure/edge path | `Math.random`·`Math.sin` 사용을 source 검사로 막고, 잘못된 delta와 bounds를 명시적으로 재현합니다. |
| 보장/비보장 | pure mechanics와 AI가 같은 seed/state에서 반복 가능하고 source state를 변경하지 않음을 증명합니다. GameHub·network·scheduler·DB 통합은 증명하지 않습니다. |
| 다음 관련 commit 연결 | `cf14c4...`에서 GameHub의 frame 계산이 실제로 검증된 simulation을 호출합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | simulation은 입력을 변경하지 않고 동일한 state/input/seed에서 동일한 결과를 냅니다. |
| 재현하는 failure/boundary | delta scaling, paddle bounds, terminal score, invalid delta, 1,000-step drift, 비결정적 난수 사용. |
| test technique | Vitest deterministic unit/regression, seeded PRNG, final-state SHA-256, source-pattern assertion. |
| 통과하는 production path | `PongSimulation.step` 및 `PongAi`를 transport 없이 직접 호출합니다. |
| 증명하는 것 | rule function과 AI의 반복 가능성·불변성·주요 bounds를 증명합니다. |
| 증명하지 않는 것 | GameHub scheduling, socket ordering, persistence 또는 multi-process determinism은 증명하지 않습니다. |
| test 성격 | Deterministic unit and long-run digest regression. |
| 후속 회귀 방지 설명 | state aliasing, physics ordering, constants, PRNG 구현이 바뀌어 같은 fixture의 결과가 달라지면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `4afec2071e7a` — `refactor(game): 득점과 충돌을 simulation에 통합`
- 다음 Thread 관련 SHA: `cf14c4052310` — `refactor(game): GameHub frame 계산을 simulation에 위임`

### 5.5. `refactor(game): GameHub frame 계산을 simulation에 위임`

| 항목 | 값 |
| --- | --- |
| SHA | `cf14c4052310` |
| Importance | S |
| Tags | SIMULATION, ARCH, REALTIME |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes GameHub consume the standalone authoritative transition.
- Classification summary: Replace GameHub's frame calculation with `PongSimulation.step` while preserving the realtime projection.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub와 simulation에 모두 mechanics가 있어 standalone transition이 production authority라고 말할 수 없었습니다. |
| 핵심 boundary/decision | `GameHub.tick`이 AI intent를 계산한 뒤 `PongSimulation.step`에 현재 state와 50ms 입력을 넘기고, 반환 state를 저장한 뒤 `syncSnapshot`으로 wire model을 투영합니다. |
| 상태 또는 ownership 변화 | mechanics는 simulation, scheduling/input/AI orchestration과 snapshot broadcast/finalization은 GameHub가 소유합니다. |
| 주요 failure/edge path | terminal 결과는 simulation output에서만 판정해 `finishRoom`으로 연결합니다. snapshot projection은 tick·score·paddle·ball을 복사해 기존 protocol을 유지합니다. |
| 보장/비보장 | production frame이 standalone rule engine을 사용하며 wire shape를 유지합니다. legacy helper가 실제로 삭제되었다는 보장은 `2cef...`에서 완성됩니다. |
| 다음 관련 commit 연결 | `2cef07...`가 GameHub의 경쟁 physics helper를 제거하고 single-owner invariant를 완성합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | 순수 simulation이 존재해도 production caller가 기존 코드를 사용하면 authoritative owner가 둘입니다. |
| 실패 위험 | test가 통과한 mechanics와 실제 socket clients가 받는 결과가 달라질 수 있습니다. |
| 핵심 결정 | GameHub의 frame path를 `PongSimulation.step` 하나로 교체하고 wire projection은 별도 `syncSnapshot`으로 유지합니다. |
| 구현 경로 | scheduler → `GameHub.tick` → AI/input 작성 → `PongSimulation.step` → `room.simulation` 저장 → `syncSnapshot` → broadcast/finish. |
| 수명주기·상태 | GameHub는 room·timer·client를 계속 소유하고 simulation은 resource-free state transition만 수행합니다. |
| 실패 처리 | terminal output에서만 finalization을 시작하며, 기존 realtime event shape는 projection으로 보존합니다. |
| 후속 검증 | `2cef...` 중복 삭제와 `37a7b2...` replay fixture가 이 ownership 이동을 고정합니다. |
| Thread 전체 의미 | 서버 권위는 socket 계층이 아니라 production에서 실제 호출되는 단일 deterministic transition으로 구체화됩니다. |

비교 기준:
- 직전 Thread 관련 SHA: `4ef4beeb8611` — `test(game): 결정적 simulation 검증`
- 다음 Thread 관련 SHA: `2cef070188ac` — `refactor(game): GameHub의 중복 물리 계산 제거`

### 5.6. `refactor(game): GameHub의 중복 물리 계산 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `2cef070188ac` |
| Importance | B |
| Tags | SIMULATION, REALTIME |
| 학습 깊이 | Thread 흐름에서 맡는 구현 역할과 필요한 상태 변화를 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Deletes the competing legacy rules after migration.
- Classification summary: Remove the obsolete GameHub physics and AI helpers after the production handoff.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | production call은 simulation으로 전환됐지만 old constants와 collision/reset/AI helper가 파일에 남아 향후 재사용될 가능성이 있었습니다. |
| 핵심 boundary/decision | `gameHub.ts`에서 legacy physics/AI constants와 `clamp`, reset, collision, acceleration 및 예측 helper를 삭제합니다. |
| 상태 또는 ownership 변화 | GameHub에는 scheduler, input/AI orchestration, `syncSnapshot`, room lifecycle만 남고 mechanics 구현은 simulation에만 존재합니다. |
| 주요 failure/edge path | 동작 추가가 아니라 competing implementation 제거입니다. 삭제 뒤에도 protocol projection은 유지됩니다. |
| 보장/비보장 | 해당 SHA에서 중복 rule implementation이 남지 않습니다. 장기 버전 호환성은 아직 fixture가 필요합니다. |
| 다음 관련 commit 연결 | `37a7b2...`가 1,000-tick versioned replay를 추가해 이후 mechanics 변경을 검출합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `cf14c4052310` — `refactor(game): GameHub frame 계산을 simulation에 위임`
- 다음 Thread 관련 SHA: `37a7b2e4611b` — `test(game): versioned match replay fixture 추가`

### 5.7. `test(game): versioned match replay fixture 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `37a7b2e4611b` |
| Importance | A |
| Tags | SIMULATION, REALTIME, TEST |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Turns long-run deterministic behavior into a versioned compatibility fixture.
- Classification summary: Add an explicit version-one replay input fixture and expected final digest.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | inline deterministic tests는 존재하지만 입력 series와 결과 digest를 외부 versioned artifact로 고정하지 않았습니다. |
| 핵심 boundary/decision | `fixtures/replay-v1.json`에 version, seed, 50ms, 1,000개 입력, initial state, expected final SHA-256을 저장하고 test가 순서대로 `PongSimulation.step`을 재생합니다. |
| 상태 또는 ownership 변화 | fixture가 replay compatibility의 고정 입력을 소유하고 production simulation이 이를 해석합니다. |
| 주요 failure/edge path | fixture shape/length/encoded direction을 먼저 검증해 불완전 fixture가 우연히 통과하지 않게 합니다. |
| 보장/비보장 | 동일 version-one fixture의 final state digest가 `f0a9...3f61`과 일치합니다. network timing, scheduler lag, persistence 결과는 포함하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 상태는 simulation이 mechanics의 유일한 owner이고 GameHub가 이를 projection하는 형태입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | version-one replay input은 구현·환경과 무관하게 동일 final simulation state를 만듭니다. |
| 재현하는 failure/boundary | 1,000 fixed steps 동안 작은 physics·AI·serialization 변화가 누적되는 장기 drift. |
| test technique | Versioned JSON fixture + deterministic replay + final-state SHA-256 regression. |
| 통과하는 production path | fixture decode → `PongSimulation.step` 1,000회 → canonical state hash 비교. |
| 증명하는 것 | 해당 fixture에 대한 mechanics 장기 호환성과 deterministic replay를 증명합니다. |
| 증명하지 않는 것 | 모든 possible input이나 actual WebSocket frame scheduling을 증명하지 않습니다. |
| test 성격 | Versioned deterministic replay compatibility test. |
| 후속 회귀 방지 설명 | physics ordering·constants·state serialization이 의도 없이 바뀌면 digest가 달라져 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `2cef070188ac` — `refactor(game): GameHub의 중복 물리 계산 제거`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes. | `9e3664f5de48` | `4afec2071e7a`, `cf14c4052310` | `cf14c4052310` 전까지 GameHub와 simulation의 rule ownership이 겹침 | `2cef070188ac` 중복 삭제 | `4ef4beeb8611`, `37a7b2e4611b` | `gameHub.ts::tick/syncSnapshot`, `game/pongSimulation.ts::step` |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| GameHub가 mechanics와 transport를 함께 소유 | `2e4359...` state 경계 → `4afec2...` complete step → `cf14c4...` production handoff | `4ef4be...` deterministic tests | 규칙과 resource orchestration 분리 |
| migration 뒤 legacy physics가 남음 | `2cef07...` constants/helper 삭제 | source inspection + `37a7b2...` replay | competing rule implementation 없음 |
| 짧은 unit case만으로 장기 drift를 놓침 | `37a7b2...` versioned 1,000-step fixture | final SHA-256 | version-one replay compatibility |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| physics/scoring/terminal rules | `9e3664...` GameHub | `4afec2...` PongSimulation에 완전 구현 | `cf14c4...` 이후 `PongSimulation.step` | resource 없음; 반환 state를 GameHub가 보유 | `pongSimulation.ts::step` |
| simulation state | wire snapshot 내부 | `2e4359...` 독립 state/clone | `room.simulation` | room 제거 시 GameHub가 함께 폐기 | `gameHub.ts`, `pongSimulation.ts` |
| timing/input/projection | GameHub | mechanics만 분리 | GameHub | scheduler/room lifecycle가 정리 | `gameHub.ts::tick/syncSnapshot` |
| deterministic evidence | 없음 | `4ef4be...` unit/digest | `37a7b2...` versioned fixture | test fixture가 repository에 고정 | `replayFixture.test.ts`, `replay-v1.json` |

## 9. Thread 최종 상태

- 최종 authoritative owner: `PongSimulation.step`이 mechanics와 terminal result의 유일한 owner이며 `GameHub`는 frame orchestration과 projection을 소유합니다.
- 최종 상태/invariant: 동일 state/input/delta/seed는 동일 next state를 만들고 browser는 score·collision을 결정하지 않습니다.
- 남아 있는 의도적 제한 또는 비보장: replay fixture는 하나의 versioned input series이며 network scheduling·persistence·multi-process를 검증하지 않습니다.
- 후속 Thread가 의존하는 contract: GameHub와 protocol 계층은 simulation state를 authoritative snapshot으로 투영하고 terminal output만 finalization에 넘깁니다.
- 대표 코드 근거: `4afec2071e7a apps/api/src/game/pongSimulation.ts::PongSimulation.step`, `cf14c4052310 apps/api/src/gameHub.ts::tick/syncSnapshot`, `37a7b2e4611b replayFixture.test.ts`

## 10. 최종 architecture 또는 execution flow 정리

```text
[WebSocket direction / AI intent]
    ↓  cf14c4052310 GameHub.tick
[PongSimulation.step: clone → move → collide → score/reset → accelerate → terminal]
    ↓  4afec2071e7a
[room.simulation 저장]
    ↓  GameHub.syncSnapshot
[versioned snapshot broadcast 또는 finishRoom]
    ↓
[37a7b2e4611b replay fixture로 장기 결과 검증]
```

- `PongSimulation`은 socket, timer, repository를 소유하지 않습니다.
- `GameHub`는 입력·AI·room lifecycle을 조정하지만 physics 식을 다시 구현하지 않습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 01-authoritative-deterministic-game-mechanics.md =====

===== BEGIN FILE: 02-cookie-identity-websocket-admission.md =====
# 쿠키 신원과 일회용 WebSocket 입장

원문 Development Thread: `Cookie identity and one-time WebSocket admission`

## 1. Thread 목표

- 여러 transport로 노출되던 durable session을 HttpOnly cookie 하나로 제한하는 보안 전환을 추적합니다.
- cookie 인증 HTTP 요청이 hash-only, short-lived, single-use ticket을 발급하고 socket handshake가 이를 원자적으로 소비하는 전체 trust chain을 복원합니다.
- 인증 지연 중 early message 보존과 unauthenticated buffering 상한, credential log redaction이 같은 경계를 어떻게 완성하는지 확인합니다.

### Source에서 확정된 significance

> The branch visibly moves away from reusable tokens in JavaScript and URLs. Authentication becomes a two-stage trust chain: the cookie authenticates an HTTP ticket request, and an atomic one-time ticket authenticates exactly one socket. Buffer limits and log redaction complete the security boundary rather than treating ticket issuance alone as sufficient.

### 직접 연결되는 Critical Invariants

> The durable browser session is carried only by an HttpOnly cookie; raw WebSocket tickets are short-lived, single-use, hashed at rest, bounded during authentication, and excluded from logs.

### 직접 연결되는 Major Engineering Difficulties

> Authenticating WebSockets without exposing durable session credentials while preserving early messages and bounding unauthenticated buffering.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 session은 cookie, bearer, query 중 어디에서 수용되었고 왜 하나의 cookie 경계로 축소되었습니까?
- 브라우저 JavaScript가 durable credential을 보유하지 않도록 API client와 socket 연결 흐름이 어떻게 바뀝니까?
- raw ticket, digest, expiry, protocol version, active-user check의 소유 계층은 각각 어디입니까?
- ticket 소비는 동시 요청과 replay에서도 정확히 한 번만 성공하도록 어떤 SQL/저장소 연산을 사용합니까?
- 인증 전 message buffering의 message-size, count, total-byte 제한은 어디에서 검사되고 어떤 close path로 수렴합니까?
- URL serializer와 redaction path가 서로 다른 credential 노출면을 어떻게 막습니까?

## 3. 완료 기준

- development login부터 cookie 설정, ticket 발급, digest 저장, handshake 소비, `GameHub` 등록까지의 trust chain을 그릴 수 있습니다.
- durable session과 one-time ticket의 수명·노출·저장·재사용 가능성 차이를 설명할 수 있습니다.
- forged, expired, reused, suspended, wrong-version ticket 각각의 처리와 소비 여부를 코드와 테스트로 구분할 수 있습니다.
- pre-auth buffering 제한과 listener/closed-socket cleanup을 실제 branch 순서로 기록할 수 있습니다.
- 인증 정보가 request URL 및 structured log에 남지 않는 근거를 제시할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1779df300611` | `feat(api): 로그인과 로비 HTTP 경계 구현` | B | AUTH, PERSISTENCE, WEB | Establishes repository-backed sessions and the initial browser identity boundary. |
| 2 | `d0531791406b` | `fix(auth): cookie-only session과 환경별 route 적용` | S | AUTH, ARCH, RISK | Restricts durable credentials to the HttpOnly cookie and scopes development login by mode. |
| 3 | `353ca9a17415` | `fix(web): browser token 저장 제거` | A | AUTH, PROTOCOL, REALTIME | Removes JavaScript-managed durable credentials from the browser. |
| 4 | `d9bde7485719` | `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의` | A | AUTH, PROTOCOL, REALTIME | Defines high-entropy raw tickets and hash-only storage semantics. |
| 5 | `c89a455fee06` | `feat(db): PostgreSQL WebSocket ticket 저장 추가` | A | AUTH, SIMULATION, REALTIME | Makes ticket consumption atomic and single-use in PostgreSQL. |
| 6 | `306d1946afb7` | `feat(auth): ticket 기반 WebSocket 인증 연결` | S | AUTH, REALTIME, RISK | Completes the cookie-to-ticket-to-socket trust handoff with bounded pre-auth buffering. |
| 7 | `b0ee833313c1` | `test(auth): WebSocket ticket 경계 검증` | A | AUTH, PROTOCOL, REALTIME | Exercises replay, expiry, suspension, protocol version, and concurrent consumption. |
| 8 | `ec9cb39babef` | `fix(log): 요청 비밀 정보 redaction 적용` | A | AUTH, REALTIME, OBSERVABILITY | Prevents tickets and durable credentials from becoming operational log data. |

## 5. Commit별 학습 기록

### 5.1. `feat(api): 로그인과 로비 HTTP 경계 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `1779df300611` |
| Importance | B |
| Tags | AUTH, PERSISTENCE, WEB |
| 학습 깊이 | Thread 흐름에서 맡는 구현 역할과 필요한 상태 변화를 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Establishes repository-backed sessions and the initial browser identity boundary.
- Classification summary: Establish the first HTTP boundary around repository-backed identity and lobby data.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | repository user/session 기능은 있었지만 Fastify route에서 login·current user·lobby를 일관되게 연결하는 HTTP boundary가 없었습니다. |
| 핵심 boundary/decision | `apps/api/src/app.ts`의 dev-login이 user를 생성/재사용하고 server session을 만든 뒤 `pp_session` HttpOnly·SameSite=Lax cookie를 설정합니다. 동시에 JSON에 raw `token`도 반환합니다. |
| 상태 또는 ownership 변화 | repository가 durable session을 저장하고 app의 `currentUser`가 credential 해석을 중앙화합니다. 당시에는 cookie, Bearer header, `session` query/raw URL query를 모두 허용합니다. |
| 주요 failure/edge path | durable token이 response body·JavaScript·Authorization header·URL에 재사용될 수 있어 XSS·history·proxy/log 노출면이 넓습니다. |
| 보장/비보장 | repository-backed identity와 protected `/me`/optional lobby identity는 동작합니다. cookie-only, environment route 제한, WebSocket 전용 credential은 아직 보장하지 않습니다. |
| 다음 관련 commit 연결 | `d05317...`이 durable credential source를 HttpOnly cookie 하나로 축소하고 dev-login을 mode별로 제한합니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `d0531791406b` — `fix(auth): cookie-only session과 환경별 route 적용`

### 5.2. `fix(auth): cookie-only session과 환경별 route 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `d0531791406b` |
| Importance | S |
| Tags | AUTH, ARCH, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Restricts durable credentials to the HttpOnly cookie and scopes development login by mode.
- Classification summary: Replace reusable cross-transport session tokens with a cookie-only durable identity boundary.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | dev-login이 raw token을 반환하고 `currentUser`가 cookie·Bearer·query를 모두 신뢰해 동일 durable credential이 여러 transport에 노출됐습니다. |
| 핵심 boundary/decision | `AppMode`를 도입해 dev-login은 development/test에서만 등록하고, response token과 CORS authorization 허용을 제거하며 `readSessionToken`은 `pp_session` cookie만 읽습니다. production/demo cookie에는 `secure:true`를 적용합니다. |
| 상태 또는 ownership 변화 | durable browser credential의 전달 owner가 browser cookie jar가 되고 JavaScript/URL/header는 더 이상 인증 source가 아닙니다. |
| 주요 failure/edge path | 해당 SHA의 `readAppMode`는 알 수 없는 값을 development로 떨어뜨려 mode typo가 fail-open할 수 있습니다. 이 문제는 guest runtime fix에서 별도로 수정됩니다. |
| 보장/비보장 | durable session은 HttpOnly cookie로만 읽히고 production/demo에서 secure cookie를 사용합니다. WebSocket 입장 자체는 아직 cookie를 대체할 one-time ticket이 없습니다. |
| 다음 관련 commit 연결 | `353ca9...`가 browser 저장/Authorization 사용을 제거하고, `d9bde7...`부터 one-time ticket chain을 구축합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | 동일한 장기 session token이 cookie, JS response, header, URL에서 재사용돼 한 곳의 노출이 모든 HTTP/WebSocket 접근권으로 이어집니다. |
| 실패 위험 | XSS, browser storage, access log, referrer, proxy가 durable account credential을 획득할 수 있습니다. |
| 핵심 결정 | durable credential은 HttpOnly cookie 하나에만 두고 개발 login route를 runtime mode로 제한합니다. |
| 구현 경로 | dev-login → server session → cookie; subsequent `currentUser` → `request.cookies.pp_session`만 조회. |
| 수명주기·상태 | cookie 수명은 server session과 결합하고 JavaScript는 값을 읽거나 전달하지 않습니다. |
| 실패 처리 | unknown `APP_MODE` fail-open은 남아 있으며 WebSocket은 별도 short-lived credential이 필요합니다. |
| 후속 검증 | `353ca9...` browser token 제거, `b0ee83...` boundary tests, `ec9cb3...` log redaction이 전체 경계를 완성합니다. |
| Thread 전체 의미 | 인증 architecture가 ‘어디서든 제출 가능한 token’에서 ‘HTTP cookie로 신원을 증명한 뒤 목적별 capability 발급’으로 바뀝니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | 개발 편의상 반환 token과 Bearer/query 인증을 함께 허용해도 안전 경계를 유지할 수 있다는 가정. |
| 실제 실패 또는 위험 | durable credential이 browser JavaScript와 URL/log를 통과해 재사용 가능한 비밀이 됩니다. |
| Root cause | credential transport를 route마다 허용해 identity resolution의 신뢰 source가 하나가 아니었습니다. |
| 수정된 invariant/decision | cookie-only session, mode-scoped dev route, secure cookie, no token response/header CORS. |
| 변경 코드 | `apps/api/src/app.ts::readSessionToken/readAppMode`, dev-login response/cookie/CORS 변경. |
| Regression evidence | `b0ee833...`은 WebSocket에서 session cookie/Bearer를 직접 수용하지 않고 ticket만 허용하는 경계를 검사합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `1779df300611` — `feat(api): 로그인과 로비 HTTP 경계 구현`
- 다음 Thread 관련 SHA: `353ca9a17415` — `fix(web): browser token 저장 제거`

### 5.3. `fix(web): browser token 저장 제거`

| 항목 | 값 |
| --- | --- |
| SHA | `353ca9a17415` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Removes JavaScript-managed durable credentials from the browser.
- Classification summary: Make browser HTTP and realtime clients consume cookie identity without storing a reusable token.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 서버가 cookie-only로 바뀌어도 browser code가 localStorage token·Bearer header·token URL을 유지하면 노출면과 불일치가 남습니다. |
| 핵심 boundary/decision | `apps/web/src/lib/api.ts`와 pages에서 token helper/localStorage/Bearer를 제거하고 모든 fetch에 `credentials:'include'`를 사용합니다. realtime은 `requestWsTicket`을 비동기로 호출하고 pending issuance를 `AbortController`로 취소합니다. |
| 상태 또는 ownership 변화 | browser는 durable credential 값을 소유하지 않고 cookie jar에 위임합니다. API client는 response schema와 `ApiError`, auth-loss event를 소유합니다. |
| 주요 failure/edge path | page unmount/replacement 중 ticket response가 늦게 도착하면 stale socket을 열 수 있으므로 abort cleanup으로 차단합니다. 이 commit 시점에는 server ticket route가 뒤 commit에 있어 consumer가 먼저 준비된 상태입니다. |
| 보장/비보장 | JavaScript storage/header에 durable token이 남지 않고 credentialed cookie request를 사용합니다. ticket의 entropy·storage·single-use는 아직 보장하지 않습니다. |
| 다음 관련 commit 연결 | `d9bde7...`가 raw ticket 형식과 HTTP handshake 계약을 정의합니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | 서버만 cookie-only로 바꾸면 browser에 남은 token code는 무해하다는 가정. |
| 실제 실패 또는 위험 | localStorage와 Authorization header가 XSS·extension·debug tooling에 durable secret을 계속 노출합니다. |
| Root cause | client auth state가 server cookie policy와 별개로 token 값을 소유했습니다. |
| 수정된 invariant/decision | browser auth는 cookie transport와 ephemeral ticket request만 사용합니다. |
| 변경 코드 | `apps/web/src/lib/api.ts`, lobby/play/admin call sites, ticket request cleanup. |
| Regression evidence | API client tests가 credentials inclusion, schema error, session expiry event, abort behavior를 고정합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `d0531791406b` — `fix(auth): cookie-only session과 환경별 route 적용`
- 다음 Thread 관련 SHA: `d9bde7485719` — `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의`

### 5.4. `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `d9bde7485719` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines high-entropy raw tickets and hash-only storage semantics.
- Classification summary: Define the raw one-time WebSocket ticket and strict HTTP/handshake shape.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | cookie-only session은 browser HTTP에는 적합하지만 WebSocket URL에 durable cookie 값을 직접 복제하지 않고 입장시킬 capability가 없었습니다. |
| 핵심 boundary/decision | `apps/api/src/wsTicket.ts`는 `randomBytes(32).toString('base64url')`로 43자 raw ticket을 만들고 SHA-256 digest만 저장하도록 helper를 제공합니다. TTL은 30초입니다. shared schema는 strict `{ticket, v:'1'}`를 정의합니다. |
| 상태 또는 ownership 변화 | raw ticket은 발급 response와 client가 짧게 소유하고 repository에는 digest만 전달됩니다. protocol version은 shared schema가 소유합니다. |
| 주요 failure/edge path | raw ticket 형식·TTL·version을 명시하지 않으면 약한 entropy, 장기 replay, handshake ambiguity가 생깁니다. 실제 atomic consumption은 아직 구현되지 않았습니다. |
| 보장/비보장 | high-entropy opaque raw value와 hash-only persistence contract, 30초 TTL, version-one query shape를 정의합니다. DB single-use와 socket integration은 다음 SHAs 범위입니다. |
| 다음 관련 commit 연결 | `c89a45...`가 digest table과 atomic consume를 PostgreSQL에 구현합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `353ca9a17415` — `fix(web): browser token 저장 제거`
- 다음 Thread 관련 SHA: `c89a455fee06` — `feat(db): PostgreSQL WebSocket ticket 저장 추가`

### 5.5. `feat(db): PostgreSQL WebSocket ticket 저장 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `c89a455fee06` |
| Importance | A |
| Tags | AUTH, SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes ticket consumption atomic and single-use in PostgreSQL.
- Classification summary: Persist only ticket digests and consume them with one destructive database operation.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ticket 형식은 정의됐지만 process/DB race에서 두 handshake가 같은 ticket을 동시에 사용할 수 있는 storage semantics가 없었습니다. |
| 핵심 boundary/decision | `packages/db/migrations/002_ws_tickets.sql`에 digest/user/expiry를 저장하고 `PostgresRepository.consumeWsTicket`은 `DELETE ... RETURNING` CTE로 row를 먼저 제거한 뒤 active user와 expiry를 확인합니다. |
| 상태 또는 ownership 변화 | PostgreSQL unique row가 pending capability의 authoritative owner이고 소비 시 row 삭제가 ownership 이전의 선형화 지점입니다. |
| 주요 failure/edge path | expired 또는 suspended user ticket도 destructive consume 뒤 null을 반환하므로 반복 시도에 남지 않습니다. transaction race에서 한 DELETE만 row를 획득합니다. |
| 보장/비보장 | at-rest에는 64-char digest만 남고 동일 digest 소비는 최대 한 번 성공합니다. HTTP issue route와 pre-auth buffer는 아직 연결되지 않았습니다. |
| 다음 관련 commit 연결 | `306d19...`가 cookie-authenticated issue route와 `/ws` admission을 연결합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `d9bde7485719` — `feat(auth): WebSocket ticket 생성과 HTTP 계약 정의`
- 다음 Thread 관련 SHA: `306d1946afb7` — `feat(auth): ticket 기반 WebSocket 인증 연결`

### 5.6. `feat(auth): ticket 기반 WebSocket 인증 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `306d1946afb7` |
| Importance | S |
| Tags | AUTH, REALTIME, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Completes the cookie-to-ticket-to-socket trust handoff with bounded pre-auth buffering.
- Classification summary: Integrate cookie-authenticated issuance, atomic ticket consumption, and bounded asynchronous WebSocket admission.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ticket generator와 DB consume는 존재하지만 browser cookie에서 socket authority로 이어지는 complete route/handshake가 없었습니다. |
| 핵심 boundary/decision | `POST /auth/ws-ticket`은 cookie-derived active user만 허용해 raw ticket을 한 번 반환하고 digest/30초를 저장합니다. `/ws`는 version을 먼저 검증한 후 ticket을 consume하고 성공 시 buffered payload와 함께 `GameHub.connect`로 handoff합니다. |
| 상태 또는 ownership 변화 | HTTP app이 issue/admission boundary를, DB가 pending digest를, GameHub가 authenticated socket 이후 lifecycle을 소유합니다. pre-auth listener와 buffers는 handoff 또는 close 전까지 app이 소유합니다. |
| 주요 failure/edge path | 인증 중 message는 item 8KiB, count 16, total 32KiB로 제한합니다. malformed/version/ticket/size/DB error는 idempotent `closeAuthentication`으로 listener 제거와 close code 1008/1009/1011에 수렴합니다. closed socket에는 handoff하지 않습니다. |
| 보장/비보장 | durable cookie → short-lived raw ticket → hash-only atomic consume → exactly one authenticated socket handoff가 완성됩니다. cross-process ticket issuance capacity와 operational logging은 별도 문제입니다. |
| 다음 관련 commit 연결 | `b0ee83...`이 replay·expiry·suspension·wrong version·concurrency·buffer limits를 실제 integration fixture로 검증합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | WebSocket upgrade 동안 DB 인증이 비동기인데 client는 즉시 message를 보낼 수 있어 credential handoff와 memory bounds를 동시에 해결해야 합니다. |
| 실패 위험 | ticket replay, wrong-version consumption, early-message loss, unauthenticated memory growth, close/handoff race. |
| 핵심 결정 | version-first validation, destructive consume, bounded message buffer, 단일 idempotent close/handoff owner를 둡니다. |
| 구현 경로 | cookie-auth HTTP ticket issue → raw response/digest insert → `/ws?v=1&ticket=...` → query validation → digest consume → active user → `GameHub.connect(pendingPayloads)`. |
| 수명주기·상태 | pending raw value는 client가 보유하고 DB row는 consume 시 삭제되며 pre-auth listener/buffer는 성공 handoff 또는 close에서 한 번 정리됩니다. |
| 실패 처리 | 8KiB/16/32KiB 상한, unsupported version before consume, DB failure 1011, invalid auth 1008, oversize 1009. |
| 후속 검증 | `b0ee833...` live Fastify/WebSocket와 memory/PostgreSQL concurrent consumption tests. |
| Thread 전체 의미 | 장기 account identity와 socket-specific admission capability가 분리되어 한 ticket 노출의 피해 범위가 한 socket/30초로 제한됩니다. |

비교 기준:
- 직전 Thread 관련 SHA: `c89a455fee06` — `feat(db): PostgreSQL WebSocket ticket 저장 추가`
- 다음 Thread 관련 SHA: `b0ee833313c1` — `test(auth): WebSocket ticket 경계 검증`

### 5.7. `test(auth): WebSocket ticket 경계 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `b0ee833313c1` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Exercises replay, expiry, suspension, protocol version, and concurrent consumption.
- Classification summary: Add deterministic integration and repository tests for the complete ticket boundary.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | trust chain은 구현됐지만 replay·concurrent consume·pre-auth pressure·version order가 회귀하지 않는다는 자동 증거가 없었습니다. |
| 핵심 boundary/decision | live Fastify/WebSocket fixture와 memory/PostgreSQL repository tests가 cookie-only issue, hash-at-rest, valid-once, forged/expired/suspended, unsupported version non-consumption, direct session rejection, pre-auth limits, concurrent one-winner를 검사합니다. |
| 상태 또는 ownership 변화 | test의 delayed consume gate가 인증 지연을 제어하고 fake/real socket payload가 buffer path를 재현합니다. |
| 주요 failure/edge path | 같은 ticket을 동시 consume해 정확히 한 non-null 결과만 허용하며, wrong version은 ticket을 쓰지 않아 뒤의 valid version이 소비할 수 있는지 구분합니다. |
| 보장/비보장 | ticket trust boundary의 핵심 failure와 memory limits가 deterministic regression으로 보호됩니다. 인터넷 proxy·multi-region latency·장기 capacity는 증명하지 않습니다. |
| 다음 관련 commit 연결 | `ec9cb3...`가 성공적인 security control이 로그를 통해 다시 비밀을 노출하지 않도록 redaction을 추가합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | raw ticket은 hash-only, 30초, one-time이며 version-one admission에서만 exactly one socket에 권한을 넘깁니다. |
| 재현하는 failure/boundary | replay, forged/expired/suspended, wrong version, direct cookie/Bearer WS, message size/count/total, concurrent consume. |
| test technique | Fastify/WebSocket integration, delayed authentication gate, memory and PostgreSQL concurrent repository tests. |
| 통과하는 production path | login cookie → issue → DB digest → WS handshake/pre-auth messages → consume → GameHub handoff 또는 close. |
| 증명하는 것 | 검사한 storage와 process 경로에서 single-use 및 bounded admission이 유지됩니다. |
| 증명하지 않는 것 | cross-region deployment, production proxy logs, sustained load capacity는 증명하지 않습니다. |
| test 성격 | Deterministic boundary integration plus PostgreSQL concurrency regression. |
| 후속 회귀 방지 설명 | credential source를 다시 늘리거나, consume order/limits/listener cleanup을 약화하면 테스트가 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `306d1946afb7` — `feat(auth): ticket 기반 WebSocket 인증 연결`
- 다음 Thread 관련 SHA: `ec9cb39babef` — `fix(log): 요청 비밀 정보 redaction 적용`

### 5.8. `fix(log): 요청 비밀 정보 redaction 적용`

| 항목 | 값 |
| --- | --- |
| SHA | `ec9cb39babef` |
| Importance | A |
| Tags | AUTH, REALTIME, OBSERVABILITY |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Prevents tickets and durable credentials from becoming operational log data.
- Classification summary: Redact credential-bearing headers, query objects, ticket fields, and raw URL query strings.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ticket 자체는 짧고 one-time이어도 Fastify/Pino request serializer와 structured fields에 URL query·cookie·Authorization·ticket이 기록될 수 있었습니다. |
| 핵심 boundary/decision | `apps/api/src/requestLogging.ts::createLoggerOptions`가 cookie, authorization, query variants, `ticket`, `*.ticket`을 redact하고 request serializer는 URL의 `?` 이후를 제거합니다. |
| 상태 또는 ownership 변화 | logger configuration이 operational serialization boundary를 소유하며 application route가 개별적으로 비밀을 지울 필요를 줄입니다. |
| 주요 failure/edge path | structured query field와 raw URL은 별도 노출면이므로 둘 다 차단합니다. 임의 application message에 비밀을 복사하면 이 path 밖일 수 있습니다. |
| 보장/비보장 | 표준 request log와 지정 structured paths에서 durable cookie와 raw ticket이 제외됩니다. 모든 제3자 로그·custom string까지 자동 sanitize하지는 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 trust chain은 cookie-only durable identity, hash-only one-time ticket, bounded admission, request-log redaction입니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | ticket이 short-lived/single-use이면 URL 또는 request log에 남아도 위험이 제한적이라는 가정. |
| 실제 실패 또는 위험 | valid window 내 replay와 credential telemetry 보존, durable cookie/header 노출이 가능합니다. |
| Root cause | security boundary가 storage/handshake만 다루고 observability serializer를 포함하지 않았습니다. |
| 수정된 invariant/decision | field redaction과 raw URL query stripping을 logger의 공통 serializer에서 수행합니다. |
| 변경 코드 | `apps/api/src/requestLogging.ts::createLoggerOptions`, Fastify logger wiring. |
| Regression evidence | request logging tests가 cookie/header/query/ticket과 query string이 serialized output에 없는지 확인해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `b0ee833313c1` — `test(auth): WebSocket ticket 경계 검증`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| The durable browser session is carried only by an HttpOnly cookie; raw WebSocket tickets are short-lived, single-use, hashed at rest, bounded during authentication, and excluded from logs. | `1779df300611` cookie/session | `d0531791406b` cookie-only → `d9bde7485719` ticket → `c89a455fee06` atomic storage → `306d1946afb7` admission → `ec9cb39babef` logging | `1779df300611` token/header/query 노출 | `d0531791406b`, `353ca9a17415`, `ec9cb39babef` | `b0ee833313c1` | `app.ts::readSessionToken,/auth/ws-ticket,/ws`, `postgresRepository::consumeWsTicket`, `requestLogging.ts` |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| durable session을 cookie·Bearer·query와 JSON token으로 재사용 | `d053179...` cookie-only + `353ca9...` browser token 제거 | API/auth tests | JavaScript와 URL에 durable secret 없음 |
| WebSocket이 account credential을 직접 사용 | `d9bde7...` raw/hash contract → `c89a45...` atomic consume → `306d19...` handoff | `b0ee83...` replay/concurrency integration | 30초 one-time capability만 socket에 사용 |
| 비동기 인증 중 unbounded early messages | `306d19...` 8KiB/16/32KiB 및 idempotent close | delayed consume fixture | unauthenticated memory work bounded |
| 비밀이 request log로 재노출 | `ec9cb3...` redaction + URL strip | logging regression inspection | 표준 operational log에 credential 제외 |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| durable browser session | `1779df...` cookie + returned token | `d053179...` cookie-only, `353ca9...` JS 제거 | HttpOnly `pp_session` cookie/server repository | logout/expiry가 session과 cookie 정리 | `app.ts::readSessionToken`, web `apiFetch` |
| pending WS capability | 없음 | `d9bde7...` raw/hash/TTL | PostgreSQL digest row | `DELETE ... RETURNING` consume 또는 expiry cleanup | `002_ws_tickets.sql`, `consumeWsTicket` |
| pre-auth messages | 직접 handoff 또는 미정 | `306d19...` bounded listener/buffer | Fastify `/ws` admission | success handoff/`closeAuthentication` | `apps/api/src/app.ts` |
| credential logging | default request serializer | `ec9cb3...` centralized redaction | Pino/Fastify logger options | serializer lifetime follows app | `requestLogging.ts` |

## 9. Thread 최종 상태

- 최종 authoritative owner: repository-backed session은 HttpOnly cookie가, pending socket capability는 digest store가, authenticated connection은 GameHub가 각각 소유합니다.
- 최종 상태/invariant: durable credential은 JavaScript·Bearer·query로 수용되지 않고 raw ticket은 30초·single-use·hash-only·bounded·log-excluded입니다.
- 남아 있는 의도적 제한 또는 비보장: DB/process outage, multi-region capability storage, custom application log 문자열까지 자동으로 해결하지 않습니다.
- 후속 Thread가 의존하는 contract: HTTP cookie로 `/auth/ws-ticket`을 호출한 active user만 version-one `/ws` admission을 한 번 획득할 수 있습니다.
- 대표 코드 근거: `d0531791406b apps/api/src/app.ts::readSessionToken`, `c89a455fee06 PostgresRepository.consumeWsTicket`, `306d1946afb7 /auth/ws-ticket,/ws`, `b0ee833313c1 ws-ticket.test.ts`, `ec9cb39babef requestLogging.ts`

## 10. 최종 architecture 또는 execution flow 정리

```text
[HttpOnly pp_session cookie]
    ↓  d0531791406b currentUser
[POST /auth/ws-ticket: raw 32-byte Base64URL 생성]
    ↓  d9bde7485719 / c89a455fee06
[SHA-256 digest + expiry 저장]
    ↓  306d1946afb7 /ws?v=1&ticket=raw
[version 검증 → DELETE RETURNING atomic consume → active user 확인]
    ↓  bounded pre-auth buffer handoff
[GameHub.connect authenticated socket]
    ↓
[ec9cb39babef request URL/fields redaction]
```

- unsupported protocol version은 ticket consume 전에 거부돼 valid version 재시도를 막지 않습니다.
- invalid·expired·suspended ticket은 consume 후 재사용할 수 없습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 02-cookie-identity-websocket-admission.md =====

===== BEGIN FILE: 03-versioned-realtime-protocol-and-monotonic-state.md =====
# 버전 기반 실시간 프로토콜과 단조 상태

원문 Development Thread: `Versioned realtime protocol and monotonic state`

## 1. Thread 목표

- TypeScript 타입 선언 수준의 message vocabulary가 양방향 strict runtime codec으로 발전하는 과정을 추적합니다.
- snapshot sequence와 input sequence가 서버 및 브라우저 상태를 뒤로 되돌리지 못하게 하는 위치와 범위를 확인합니다.
- persisted/transient result discriminator와 machine-readable error code가 wire contract에 어떤 의미를 부여하는지 복원합니다.

### Source에서 확정된 significance

> The protocol evolves from typed messages to an executable compatibility boundary. Versioning, strict object shapes, input sequence numbers, and snapshot sequence numbers ensure that malformed, stale, duplicated, or structurally incompatible traffic cannot silently mutate authoritative or rendered state.

### 직접 연결되는 Critical Invariants

> Every accepted wire message conforms to the supported versioned runtime schema; snapshot and input ordering cannot move state backward.

> The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes.

### 직접 연결되는 Major Engineering Difficulties

> Handling slow or stale transports through sequence gates, token buckets, latest-value snapshot delivery, measurable congestion, and hard termination limits.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 client validation과 server compile-time typing 사이의 비대칭은 무엇이었습니까?
- snapshot envelope에서 transport metadata와 nested game state는 어떻게 분리됩니까?
- `v: 1`은 handshake version과 event version에서 각각 어떤 호환성 경계를 형성합니까?
- server input gate와 browser snapshot gate는 어떤 key와 비교 규칙으로 stale traffic을 거부합니까?
- persisted result와 transient result가 잘못 섞이지 않도록 schema가 강제하는 조합은 무엇입니까?
- producer와 consumer 모두 같은 shared codec을 사용한다는 사실을 실제 call site로 증명할 수 있습니까?

## 3. 완료 기준

- client event parse와 server event encode/parse 경계를 실제 symbol과 함께 정리할 수 있습니다.
- snapshot의 `tick`, `sequence`, `serverTime`, nested `state` 역할을 구분할 수 있습니다.
- duplicate/reordered input과 delayed/duplicate snapshot이 각각 어디에서 무시되는지 설명할 수 있습니다.
- strict object, unknown-field rejection, version rejection, persistence discriminator의 negative tests를 분류할 수 있습니다.
- 프로토콜 변경 시 server, shared, browser에서 동시에 바뀌어야 하는 지점을 나열할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a974f8cd9712` | `feat(shared): WebSocket 이벤트 메시지 검증` | A | PROTOCOL, SIMULATION, REALTIME | Introduces the first runtime-validated discriminated event vocabulary. |
| 2 | `7d3437c49152` | `feat(protocol): versioned game snapshot 계약 정의` | A | PROTOCOL, SIMULATION, REALTIME | Reshapes snapshots and results into strict transport models with sequence and persistence metadata. |
| 3 | `0595a386000a` | `feat(protocol): versioned WebSocket event codec 연결` | S | PROTOCOL, REALTIME, ARCH | Makes both directions strict version-one codec boundaries. |
| 4 | `1567f5005ef8` | `feat(game): room별 input sequence 중복을 차단` | A | SIMULATION, REALTIME, PERSISTENCE | Rejects stale or reordered input per client and room. |
| 5 | `8a8787d03a19` | `feat(play): versioned game input과 snapshot 소비` | A | PROTOCOL, SIMULATION, REALTIME | Adds client-side event parsing and monotonic snapshot acceptance. |
| 6 | `f655969b0d36` | `test(protocol): versioned realtime contract 검증` | A | PROTOCOL, REALTIME, PERSISTENCE | Protects version, sequence, and persistence discriminator invariants. |

## 5. Commit별 학습 기록

### 5.1. `feat(shared): WebSocket 이벤트 메시지 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `a974f8cd9712` |
| Importance | A |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Introduces the first runtime-validated discriminated event vocabulary.
- Classification summary: Introduce a discriminated WebSocket protocol and validate client-originated messages before handlers consume them.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | client/server event는 call site별 object와 TypeScript type에 의존해 malformed JSON과 구조 오류가 handler까지 도달할 수 있었습니다. |
| 핵심 boundary/decision | `packages/shared/src/ws.ts`에 Zod discriminated union과 `parseClientEvent`를 추가합니다. queue, ready, input, chat shape와 direction `-1|0|1`, chat trim/1–240을 runtime 검증합니다. |
| 상태 또는 ownership 변화 | shared package가 client-originated message vocabulary와 parse boundary를 소유합니다. server event는 이 SHA에서 TypeScript union과 `JSON.stringify`만 사용합니다. |
| 주요 failure/edge path | JSON parse failure와 schema-invalid object를 구분해 typed event만 handler에 넘깁니다. server output은 runtime validation이 없어 방향별 비대칭이 남습니다. |
| 보장/비보장 | client event의 알려진 type/payload만 application handler에 도달합니다. event version, strict server output, sequence ordering은 아직 없습니다. |
| 다음 관련 commit 연결 | `7d3437...`가 snapshot/result를 strict runtime model로 바꾸고 transport ordering metadata를 추가합니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `7d3437c49152` — `feat(protocol): versioned game snapshot 계약 정의`

### 5.2. `feat(protocol): versioned game snapshot 계약 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `7d3437c49152` |
| Importance | A |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Reshapes snapshots and results into strict transport models with sequence and persistence metadata.
- Classification summary: Replace loose game interfaces with strict schemas for version-ready snapshots and completion results.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | snapshot은 transport metadata와 mechanics state가 평평하게 섞였고 result의 persisted 여부와 matchId/ratingDelta 조합이 타입상 모순될 수 있었습니다. |
| 핵심 boundary/decision | `packages/shared/src/game.ts`의 strict Zod schemas가 snapshot을 `{roomId,tick,sequence,serverTimeMs,state}`로 분리하고, finished result를 `persisted:true/false` discriminated union으로 정의합니다. |
| 상태 또는 ownership 변화 | shared game schema가 wire state envelope와 persistence semantics를 소유하며 server/browser 모두 같은 shape를 소비할 수 있습니다. |
| 주요 failure/edge path | sequence/tick/score는 nonnegative integer, vector는 finite로 제한합니다. `persisted:true`는 non-null matchId, `persisted:false`는 null matchId와 ratingDelta 0만 허용합니다. |
| 보장/비보장 | snapshot ordering field와 transport/state 분리, contradictory result rejection이 가능해집니다. 모든 WS event에 version이 붙고 양방향 codec이 강제되는 것은 다음 commit입니다. |
| 다음 관련 commit 연결 | `0595a3...`가 모든 client/server event를 strict version-one codec으로 통합합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `a974f8cd9712` — `feat(shared): WebSocket 이벤트 메시지 검증`
- 다음 Thread 관련 SHA: `0595a386000a` — `feat(protocol): versioned WebSocket event codec 연결`

### 5.3. `feat(protocol): versioned WebSocket event codec 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `0595a386000a` |
| Importance | S |
| Tags | PROTOCOL, REALTIME, ARCH |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes both directions strict version-one codec boundaries.
- Classification summary: Turn the shared event vocabulary into a symmetric strict runtime compatibility boundary.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | client input만 runtime 검증되고 server output은 compile-time type에 의존했으며 event version이 없었습니다. |
| 핵심 boundary/decision | 모든 client/server variant에 literal `v:1`과 `.strict()`를 붙이고 `game.input.inputSeq`, stable error code enum, `parseServerEvent`, validation-before-encode를 추가합니다. |
| 상태 또는 ownership 변화 | shared codec이 양방향 wire compatibility owner가 됩니다. server producer와 browser consumer가 같은 schemas를 사용합니다. |
| 주요 failure/edge path | unknown field, wrong/missing version, unsafe/negative inputSeq, malformed nested snapshot/result는 encode 또는 parse 경계에서 fail-closed됩니다. |
| 보장/비보장 | accepted client와 server message는 모두 version-one strict schema를 만족합니다. 순서상 오래된 message를 무시하는 runtime state는 뒤 commits가 담당합니다. |
| 다음 관련 commit 연결 | `1567f5...`가 server input sequence gate를, `8a8787...`이 browser snapshot sequence gate를 연결합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | compile-time type는 network에서 받은 JSON이나 producer의 runtime object가 실제 contract를 따르는지 보장하지 못합니다. |
| 실패 위험 | versionless/extra-field/incompatible nested shape가 조용히 authoritative 또는 rendered state를 변경합니다. |
| 핵심 결정 | client parse, server encode, browser parse를 같은 strict literal-version schemas에 묶습니다. |
| 구현 경로 | raw JSON → parse/schema → typed client event; typed server event → schema validation → JSON; browser raw → `parseServerEvent`. |
| 수명주기·상태 | codec은 상태를 소유하지 않고 각 message admission 순간만 책임집니다. ordering state는 caller별 map/ref가 소유합니다. |
| 실패 처리 | 구문 오류와 schema 오류 모두 application mutation 전에 차단됩니다. |
| 후속 검증 | `f655969...` negative protocol tests가 missing version, invalid sequence, persistence contradiction을 고정합니다. |
| Thread 전체 의미 | protocol change는 shared schema, server producer, browser consumer가 함께 바뀌어야 하는 실행 가능한 compatibility boundary가 됩니다. |

비교 기준:
- 직전 Thread 관련 SHA: `7d3437c49152` — `feat(protocol): versioned game snapshot 계약 정의`
- 다음 Thread 관련 SHA: `1567f5005ef8` — `feat(game): room별 input sequence 중복을 차단`

### 5.4. `feat(game): room별 input sequence 중복을 차단`

| 항목 | 값 |
| --- | --- |
| SHA | `1567f5005ef8` |
| Importance | A |
| Tags | SIMULATION, REALTIME, PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Rejects stale or reordered input per client and room.
- Classification summary: Apply monotonic input ordering before mutating authoritative paddle intent.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 형식상 valid한 `game.input`이라도 network duplicate/reordering으로 더 오래된 direction이 최신 intent를 덮을 수 있었습니다. |
| 핵심 boundary/decision | `apps/api/src/gameHub.ts`의 각 `Client`에 `lastInputSequenceByRoom` map을 두고 room/phase/side 확인 뒤 `inputSeq <= previous`를 무시합니다. accepted일 때만 sequence 저장과 paddle direction mutation을 수행합니다. |
| 상태 또는 ownership 변화 | ordering state는 이 SHA에서 connection별 client가 room key로 소유합니다. server authoritative paddle intent는 monotonic accepted sequence만 반영합니다. |
| 주요 failure/edge path | stale/duplicate input은 no-op입니다. reconnect는 새 Client map이므로 이전 connection sequence와 연속성은 보장하지 않습니다. |
| 보장/비보장 | 한 client/room lifetime 안에서 input sequence가 뒤로 가지 않습니다. user-wide rate budget과 reconnect continuity는 Thread 08의 InputGate integration 범위입니다. |
| 다음 관련 commit 연결 | `8a8787...`이 browser에서 inputSeq를 생성하고 snapshot sequence 역행을 차단합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `0595a386000a` — `feat(protocol): versioned WebSocket event codec 연결`
- 다음 Thread 관련 SHA: `8a8787d03a19` — `feat(play): versioned game input과 snapshot 소비`

### 5.5. `feat(play): versioned game input과 snapshot 소비`

| 항목 | 값 |
| --- | --- |
| SHA | `8a8787d03a19` |
| Importance | A |
| Tags | PROTOCOL, SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds client-side event parsing and monotonic snapshot acceptance.
- Classification summary: Make the browser a strict version-one producer and monotonic server-event consumer.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | browser는 versionless command를 보내고 server snapshot을 arrival order대로 렌더링해 delayed frame이 UI를 과거로 되돌릴 수 있었습니다. |
| 핵심 boundary/decision | play page가 모든 command에 `v:1`을 붙이고 50ms input timer에서 `inputSequenceRef`를 증가시킵니다. incoming data는 `parseServerEvent`를 통과하고 `snapshot.sequence <= last`이면 무시합니다. |
| 상태 또는 ownership 변화 | browser connection instance가 input sequence와 last snapshot sequence refs를 소유하며 새 connection setup에서 초기화합니다. |
| 주요 failure/edge path | duplicate/delayed snapshot은 렌더링하지 않습니다. gap을 보충하거나 missed frame을 replay하지는 않습니다. |
| 보장/비보장 | 한 socket/session에서 rendered snapshot state는 sequence 기준 단조 증가합니다. 서버 authoritative state나 transport delivery completeness를 보장하지 않습니다. |
| 다음 관련 commit 연결 | `f65596...`가 version/sequence/persistence negative cases를 shared contract regression으로 고정합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `1567f5005ef8` — `feat(game): room별 input sequence 중복을 차단`
- 다음 Thread 관련 SHA: `f655969b0d36` — `test(protocol): versioned realtime contract 검증`

### 5.6. `test(protocol): versioned realtime contract 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `f655969b0d36` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Protects version, sequence, and persistence discriminator invariants.
- Classification summary: Add negative and round-trip tests for the strict runtime codec.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | strict schemas와 caller gates는 구현됐지만 missing version, invalid sequence, contradictory result가 회귀하지 않는 자동 contract evidence가 부족했습니다. |
| 핵심 boundary/decision | `packages/shared/src/ws.test.ts`가 client/server variants round-trip과 versionless server event, snapshot sequence -1, `persisted:true + matchId:null` rejection을 검사합니다. |
| 상태 또는 ownership 변화 | test suite가 shared wire contract의 negative examples를 고정합니다. |
| 주요 failure/edge path | invalid data가 parse/encode 경계에서 reject되는지를 직접 확인하며 application handler나 renderer를 호출하지 않습니다. |
| 보장/비보장 | 검사한 contract combinations이 runtime schema에 의해 차단됩니다. 실제 packet reordering, GameHub/browser integration, load behavior는 증명하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 상태는 strict version-one codec과 server/client 양쪽 monotonic gates가 결합된 형태입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | accepted wire message는 version-one strict schema를 만족하고 ordering/persistence metadata가 유효합니다. |
| 재현하는 failure/boundary | missing version, negative sequence, persisted result contradiction, variant round-trip. |
| test technique | Shared Zod codec unit/negative contract tests. |
| 통과하는 production path | raw/typed event → `parseClientEvent`/`parseServerEvent`/`encodeServerEvent`. |
| 증명하는 것 | schema-level compatibility와 negative rejection을 증명합니다. |
| 증명하지 않는 것 | network reordering에서 GameHub와 React state가 실제로 단조적인지는 통합 검증하지 않습니다. |
| test 성격 | Deterministic runtime contract regression. |
| 후속 회귀 방지 설명 | version literal, strictness, sequence bounds, persisted discriminator를 약화하면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `8a8787d03a19` — `feat(play): versioned game input과 snapshot 소비`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| Every accepted wire message conforms to the supported versioned runtime schema; snapshot and input ordering cannot move state backward. | `a974f8cd9712` client runtime parse | `7d3437c49152` strict snapshot/result → `0595a386000a` symmetric v1 codec → `1567f5005ef8` input gate → `8a8787d03a19` snapshot gate | 초기 server output runtime validation과 sequence state 부재 | 해당 없음 — 순차 강화 | `f655969b0d36` | `packages/shared/src/ws.ts`, `game.ts`, `gameHub.ts::applyInput`, web play snapshot handler |
| The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes. | `7d3437c49152` authoritative snapshot/result shape | `0595a386000a` producer validation, `1567f5005ef8` stale input rejection | 해당 없음 | 해당 없음 | `f655969b0d36` contract evidence | shared schemas + GameHub input mutation path |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| client만 runtime 검증, server는 compile-time type | `7d3437...` strict models → `0595a3...` symmetric codec | `f65596...` negative tests | 양방향 version-one compatibility boundary |
| valid하지만 stale input이 최신 intent를 덮음 | `1567f5...` per-client/per-room sequence gate | code inspection; later InputGate tests | authoritative input state monotonic |
| delayed snapshot이 UI를 과거로 되돌림 | `8a8787...` browser last-sequence gate | browser reducer/page tests | rendered state monotonic |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| wire vocabulary/schema | 각 call site/TS type | `a974f8...` client schema, `7d3437...` game schema | `0595a3...` shared strict v1 codec | stateless | `packages/shared/src/ws.ts`, `game.ts` |
| input ordering | 없음 | `1567f5...` client map keyed room | GameHub connection state | client disconnect/replacement 시 map 폐기 | `gameHub.ts::lastInputSequenceByRoom/applyInput` |
| snapshot ordering | arrival order | `8a8787...` sequence ref | browser connection instance | new connection setup/cleanup에서 reset | `apps/web/src/app/play/page.tsx` |
| persistence result meaning | loose interface | `7d3437...` discriminated union | shared schema | stateless | `persistedGameFinishedSchema` |

## 9. Thread 최종 상태

- 최종 authoritative owner: shared package가 wire schema/codec을, GameHub가 authoritative input admission을, browser connection이 rendered snapshot admission을 소유합니다.
- 최종 상태/invariant: version-one strict message만 수용되고 accepted input 및 rendered snapshot sequence는 각 lifetime 안에서 감소하지 않습니다.
- 남아 있는 의도적 제한 또는 비보장: sequence gap을 복구하지 않고, reconnect 전후 sequence continuity와 actual delivery rate는 이 Thread만으로 보장하지 않습니다.
- 후속 Thread가 의존하는 contract: server producer와 browser consumer는 동일 shared schemas를 사용하며 persisted/transient 결과 조합도 wire에서 강제됩니다.
- 대표 코드 근거: `0595a386000a packages/shared/src/ws.ts`, `1567f5005ef8 apps/api/src/gameHub.ts`, `8a8787d03a19 apps/web/src/app/play/page.tsx`, `f655969b0d36 ws.test.ts`

## 10. 최종 architecture 또는 execution flow 정리

```text
[Browser command {v:1,inputSeq,...}]
    ↓  shared parseClientEvent
[GameHub room/phase/side + monotonic sequence check]
    ↓
[authoritative intent/state mutation]
    ↓  encodeServerEvent validates v1 strict snapshot
[WebSocket delivery]
    ↓  browser parseServerEvent + sequence > last
[nested snapshot.state 렌더링 또는 stale drop]
```

- schema validation은 구조 호환성을, sequence gate는 시간 순서를 담당합니다.
- persisted discriminator는 UI가 durable match와 transient result를 혼동하지 않게 합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 03-versioned-realtime-protocol-and-monotonic-state.md =====

===== BEGIN FILE: 04-atomic-idempotent-match-finalization.md =====
# 원자적·멱등적 경기 결과 확정

원문 Development Thread: `Atomic and idempotent match finalization`

## 1. Thread 목표

- 경기 row와 rating update가 순차 실행되던 초기 구현에서 하나의 논리적 `finalizeMatch` command로 수렴하는 과정을 복원합니다.
- result key, unique constraint, ordered row lock, transaction, rating history, tournament bracket update가 중복·부분 반영을 어떻게 차단하는지 확인합니다.
- GameHub가 durable boundary를 호출하고 persistence failure 동안 terminal room ownership을 유지하며 같은 idempotency key로 재시도하는 흐름을 추적합니다.

### Source에서 확정된 significance

> This thread converts a multi-write best-effort workflow into one logical domain command. Database uniqueness, ordered row locks, transaction-scoped tournament progression, runtime integration, and retryable room ownership together ensure that a completed game is neither duplicated nor partially reflected in ratings or brackets.

### 직접 연결되는 Critical Invariants

> A logical match result, participant statistics, rating history, and tournament progression commit atomically and idempotently.

> Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup.

### 직접 연결되는 Major Engineering Difficulties

> Preventing duplicate results, ratings, finals, friendship rows, tournament seeds, and admissions under retries or concurrent database operations.

> Preserving domain correctness during database failure, tournament-start rollback, match-finalization retry, process drain, and deployment shutdown.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 `createMatch` 경로는 어떤 write들을 어떤 순서로 실행하며 어디에서 부분 성공할 수 있었습니까?
- logical result identity와 persisted match identity는 어떻게 다르며 재시도에서 각각 어떤 역할을 합니까?
- 동시 finalization에서 DB unique constraint와 transaction 내부 readback은 어떻게 수렴합니까?
- participant lock order, rating floor, history row, missing-user rollback을 실제 SQL 순서로 설명할 수 있습니까?
- tournament semifinal 두 개가 동시에 끝날 때 final row가 하나만 생기는 근거는 무엇입니까?
- persistence 실패 뒤 room, reservation, retry timer, drain waiter의 ownership은 어디에 남습니까?

## 3. 완료 기준

- 초기 sequential write와 최종 atomic command의 상태 변화 차이를 표로 완성할 수 있습니다.
- 동일 result key 20회 동시 호출에서 one creation/one effect가 되는 SQL 및 test 근거를 제시할 수 있습니다.
- match, counters, ratings, rating history, tournament match, final creation, tournament finish의 transaction 범위를 그릴 수 있습니다.
- GameHub completion과 repository finalization 사이의 deterministic key, in-flight coalescing, retry 흐름을 설명할 수 있습니다.
- 성공 broadcast가 durable result 이후에만 발생하는지 해당 SHA의 순서로 확인할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `38504f041a6a` | `feat(db): 경기 결과 저장 구현` | B | REALTIME, PERSISTENCE | Introduces match persistence and rating effects, initially as sequential operations. |
| 2 | `75bbc762e06d` | `feat(db): match result key와 rating history schema 추가` | A | PERSISTENCE, RISK | Adds durable identities for logical outcomes and auditable participant deltas. |
| 3 | `83f9aee2522a` | `feat(db): PostgreSQL 경기 결과 중복 생성을 차단` | A | PERSISTENCE, RISK | Uses the unique result key to converge concurrent duplicate finalization. |
| 4 | `e9d577ebc1ab` | `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영` | A | PERSISTENCE | Locks participants and commits match, counters, ratings, and history together. |
| 5 | `e338ea32b2a6` | `feat(db): PostgreSQL tournament 경기 확정을 연결` | S | PERSISTENCE, TOURNAMENT, RISK | Adds bracket and tournament progression to the same transaction. |
| 6 | `582a1615a2c6` | `test(db): 경기 결과 단일 확정 조건 검증` | A | PERSISTENCE, TOURNAMENT, RISK | Proves one creation and one set of effects under repetition, concurrency, and rollback. |
| 7 | `10bf15723591` | `refactor(game): 경기 결과 확정 boundary 사용` | A | REALTIME, PERSISTENCE, TOURNAMENT | Routes GameHub completion through the canonical repository command. |
| 8 | `e939a50948b2` | `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지` | A | REALTIME, RISK | Keeps terminal room ownership and retries with the stable idempotency key. |

## 5. Commit별 학습 기록

### 5.1. `feat(db): 경기 결과 저장 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `38504f041a6a` |
| Importance | B |
| Tags | REALTIME, PERSISTENCE |
| 학습 깊이 | Thread 흐름에서 맡는 구현 역할과 필요한 상태 변화를 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Introduces match persistence and rating effects, initially as sequential operations.
- Classification summary: Add match completion to the repository contract and initial fixed rating effects.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | realtime 경기 종료를 repository abstraction으로 durable match와 사용자 통계에 반영하는 command가 없었습니다. |
| 핵심 boundary/decision | repository `createMatch` contract가 mode, winner/loser IDs, scores를 받고 generated match ID를 반환합니다. PostgreSQL은 match insert 뒤 winner win/+16, loser loss/-12 with 800 floor를 순차 실행합니다. |
| 상태 또는 ownership 변화 | repository가 persistence effects를 수행하지만 명시적 transaction이 없어 각 SQL statement가 독립 commit될 수 있습니다. memory implementation은 local state를 비슷한 순서로 변경합니다. |
| 주요 failure/edge path | match insert 후 winner update 또는 loser update가 실패하면 match만 존재하거나 한쪽 통계만 반영되는 부분 상태가 가능합니다. retry identity도 없어 중복 row를 막지 못합니다. |
| 보장/비보장 | 한 호출의 happy path에서 match ID와 초기 rating effects를 생성합니다. atomicity, idempotency, audit history, tournament progression은 보장하지 않습니다. |
| 다음 관련 commit 연결 | `75bbc7...`가 logical result key와 rating history schema를 추가해 durable identity와 audit 기반을 만듭니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `75bbc762e06d` — `feat(db): match result key와 rating history schema 추가`

### 5.2. `feat(db): match result key와 rating history schema 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `75bbc762e06d` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds durable identities for logical outcomes and auditable participant deltas.
- Classification summary: Introduce a unique logical result key and one rating-history row per match participant.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | persisted match ID는 DB insert 후에만 생겨 같은 logical room result의 retry/concurrent call을 미리 식별할 수 없었습니다. rating delta의 독립 audit row도 없었습니다. |
| 핵심 boundary/decision | migration이 `matches.result_key`를 legacy rows에 backfill하고 NOT NULL/UNIQUE로 만들며 `rating_history`에 `(match_id,user_id)` unique identity를 둡니다. |
| 상태 또는 ownership 변화 | caller가 stable logical result key를 제공하고 database unique constraint가 duplicate command의 선형화 지점을 소유합니다. history table이 participant delta audit를 소유합니다. |
| 주요 failure/edge path | schema만으로는 application이 transaction을 사용하거나 rating/tournament effects를 idempotently 실행하지 않습니다. 기존 row는 `legacy:<id>`로 충돌 없이 보존합니다. |
| 보장/비보장 | 동일 result key의 두 match row와 동일 participant history duplicate를 DB가 거부할 수 있습니다. canonical finalize behavior는 뒤 commits에서 구현됩니다. |
| 다음 관련 commit 연결 | `83f9ae...`가 `ON CONFLICT DO NOTHING`과 transaction readback으로 duplicate match creation을 수렴시킵니다. |

비교 기준:
- 직전 Thread 관련 SHA: `38504f041a6a` — `feat(db): 경기 결과 저장 구현`
- 다음 Thread 관련 SHA: `83f9aee2522a` — `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`

### 5.3. `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`

| 항목 | 값 |
| --- | --- |
| SHA | `83f9aee2522a` |
| Importance | A |
| Tags | PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Uses the unique result key to converge concurrent duplicate finalization.
- Classification summary: Create `finalizeMatch` and make match-row creation idempotent under retries and concurrency.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | unique schema는 있지만 caller가 충돌을 일반 error로 받거나 기존 match ID를 일관되게 재사용할 command path가 없었습니다. |
| 핵심 boundary/decision | `PostgresRepository.finalizeMatch`가 transaction 안에서 match를 `ON CONFLICT (result_key) DO NOTHING RETURNING`으로 insert합니다. insert가 없으면 existing row를 읽고 `{created:false}`로 즉시 반환합니다. |
| 상태 또는 ownership 변화 | result key가 logical command identity를, existing/new match row가 persisted identity를 소유합니다. duplicate caller는 동일 match ID에 수렴합니다. |
| 주요 failure/edge path | 이 SHA에서는 duplicate row 차단만 완성되며 rating/tournament side effect까지 같은 transaction에 포함됐다고 볼 수 없습니다. |
| 보장/비보장 | 같은 result key로 match row가 둘 생기지 않고 retry가 existing match를 돌려받습니다. participant effects의 atomicity는 `e9d577...` 범위입니다. |
| 다음 관련 commit 연결 | `e9d577...`가 participant lock·counters·ratings·history를 같은 transaction에 넣습니다. |

비교 기준:
- 직전 Thread 관련 SHA: `75bbc762e06d` — `feat(db): match result key와 rating history schema 추가`
- 다음 Thread 관련 SHA: `e9d577ebc1ab` — `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영`

### 5.4. `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영`

| 항목 | 값 |
| --- | --- |
| SHA | `e9d577ebc1ab` |
| Importance | A |
| Tags | PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Locks participants and commits match, counters, ratings, and history together.
- Classification summary: Extend the idempotent match transaction to participant statistics, ratings, and audit history.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | match row는 idempotent하지만 user counters/rating/history가 transaction 밖이거나 중복 call에서 다시 실행되면 partial/double effects가 남을 수 있었습니다. |
| 핵심 boundary/decision | new match인 경우 participant IDs를 정렬해 `FOR UPDATE`로 잠그고 pre-rating을 읽은 뒤 winner +16, loser max(800, -12), win/loss counters, 두 rating history rows를 같은 transaction에서 반영합니다. |
| 상태 또는 ownership 변화 | transaction이 match와 participant effects의 atomic owner가 됩니다. user ID 정렬은 concurrent matches의 lock order를 고정합니다. |
| 주요 failure/edge path | participant가 없거나 update/history insert가 실패하면 transaction 전체가 rollback됩니다. duplicate result는 new effects 전에 existing row return path로 빠집니다. |
| 보장/비보장 | match row, counters, ratings, rating history가 one logical result에 대해 한 번만 all-or-nothing commit됩니다. tournament bracket은 아직 포함되지 않습니다. |
| 다음 관련 commit 연결 | `e338ea...`가 tournament match와 final creation/finish를 같은 transaction에 편입합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `83f9aee2522a` — `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`
- 다음 Thread 관련 SHA: `e338ea32b2a6` — `feat(db): PostgreSQL tournament 경기 확정을 연결`

### 5.5. `feat(db): PostgreSQL tournament 경기 확정을 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `e338ea32b2a6` |
| Importance | S |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds bracket and tournament progression to the same transaction.
- Classification summary: Make tournament progression part of the canonical atomic/idempotent match finalization command.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 일반 match와 rating은 atomic해졌지만 tournament match state, semifinal winner slots, final creation, tournament winner가 별도 write라면 bracket이 durable result와 어긋날 수 있었습니다. |
| 핵심 boundary/decision | `finalizeMatch` transaction이 tournament match와 tournament row를 잠그고 final/participant mismatch를 거부하며, 해당 match winner를 조건부 update합니다. 두 semifinal이 완료되면 final row를 한 번 만들고 final 완료 시 tournament를 finished/winner로 변경합니다. |
| 상태 또는 ownership 변화 | 하나의 database transaction이 logical result, participant effects, bracket progression의 authoritative commit owner가 됩니다. |
| 주요 failure/edge path | 이미 final인 tournament match, input participant mismatch, missing tournament data는 rollback됩니다. conditional writes와 locked parent state가 concurrent semifinal/final duplicate를 막습니다. |
| 보장/비보장 | match·ratings·history·tournament progression은 하나의 idempotent transaction으로 commit됩니다. runtime GameHub가 이 boundary를 사용하고 failure를 재시도하는 것은 뒤 commits입니다. |
| 다음 관련 commit 연결 | `582a16...`이 repetition/concurrency/rollback을 PostgreSQL integration으로 검증하고 `10bf15...`가 GameHub를 연결합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | durable match만 원자적이어도 tournament bracket이 별도 처리되면 semifinal/final duplication과 partial progression이 가능합니다. |
| 실패 위험 | match는 finished인데 bracket slot이 비거나, final row가 둘 생기거나, tournament winner가 누락될 수 있습니다. |
| 핵심 결정 | tournament match/parent tournament lock과 progression을 existing finalize transaction에 포함합니다. |
| 구현 경로 | result-key insert/read → participant locks/effects → tournament match lock/validate → winner update → final create 또는 tournament finish → commit. |
| 수명주기·상태 | transaction 시작부터 commit까지 DB rows가 command ownership을 갖고 rollback 시 외부에 partial state를 남기지 않습니다. |
| 실패 처리 | participant mismatch, terminal replay, missing row, any SQL failure는 모두 rollback; duplicate result key는 side effects 전에 existing result로 수렴합니다. |
| 후속 검증 | `582a161...` concurrent same-key, rating/history once, semifinal/final progression, invalid participant rollback tests. |
| Thread 전체 의미 | 경기 종료를 여러 테이블 write의 순서가 아니라 하나의 domain command로 취급할 수 있습니다. |

비교 기준:
- 직전 Thread 관련 SHA: `e9d577ebc1ab` — `feat(db): PostgreSQL 참가자 rating을 원자적으로 반영`
- 다음 Thread 관련 SHA: `582a1615a2c6` — `test(db): 경기 결과 단일 확정 조건 검증`

### 5.6. `test(db): 경기 결과 단일 확정 조건 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `582a1615a2c6` |
| Importance | A |
| Tags | PERSISTENCE, TOURNAMENT, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Proves one creation and one set of effects under repetition, concurrency, and rollback.
- Classification summary: Add PostgreSQL integration regressions for idempotency, atomic participant effects, and tournament progression.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | transaction code는 존재하지만 concurrent calls와 mid-command failure에서 실제 PostgreSQL constraints/locks/rollback이 의도대로 작동한다는 증거가 부족했습니다. |
| 핵심 boundary/decision | PostgreSQL integration fixtures가 동일 result key 반복 및 동시 호출에서 created가 한 번만 true인지, ratings/history가 한 번만 변하는지, semifinal/final progression과 invalid participant rollback을 검사합니다. |
| 상태 또는 ownership 변화 | real PostgreSQL transaction과 constraints를 test fixture가 사용하며 in-memory mock으로 대체하지 않습니다. |
| 주요 failure/edge path | duplicate 20회, participant/tournament mismatch, command failure에서 partial match/rating/bracket row가 남지 않는지를 DB query로 확인합니다. |
| 보장/비보장 | 검사한 concurrency와 rollback 사례에서 one creation/one effect를 증명합니다. 모든 isolation anomaly, external process kill, production load를 일반화하지 않습니다. |
| 다음 관련 commit 연결 | `10bf15...`가 runtime completion을 검증된 repository boundary로 바꿉니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | logical result, participant effects, history, tournament progression은 atomic·idempotent합니다. |
| 재현하는 failure/boundary | 같은 result key 반복/20-way concurrency, participant mismatch, tournament semifinal/final progression, rollback. |
| test technique | Real PostgreSQL integration tests with concurrent promises and direct post-state queries. |
| 통과하는 production path | `PostgresRepository.finalizeMatch` → unique insert/locks/updates/history/bracket → commit/rollback. |
| 증명하는 것 | 해당 DB schema/implementation에서 one creation/one effect와 tested rollback을 증명합니다. |
| 증명하지 않는 것 | process crash at every possible point, distributed DB topology, GameHub retry lifecycle는 증명하지 않습니다. |
| test 성격 | PostgreSQL deterministic concurrency and rollback regression. |
| 후속 회귀 방지 설명 | unique key, lock order, transaction scope, rating/history/tournament writes가 분리되면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `e338ea32b2a6` — `feat(db): PostgreSQL tournament 경기 확정을 연결`
- 다음 Thread 관련 SHA: `10bf15723591` — `refactor(game): 경기 결과 확정 boundary 사용`

### 5.7. `refactor(game): 경기 결과 확정 boundary 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `10bf15723591` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Routes GameHub completion through the canonical repository command.
- Classification summary: Replace ad-hoc completion persistence with `finalizeMatch` and a stable room-derived result key.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | repository command가 atomic해도 GameHub가 기존 `createMatch` 또는 별도 writes를 사용하면 production completion은 invariant를 얻지 못합니다. |
| 핵심 boundary/decision | `apps/api/src/gameHub.ts::finalizeRoom`이 `repo.finalizeMatch`를 호출하고 result key를 `room:<roomId>:finished`로 고정하며 mode, participants, scores, optional tournamentMatchId를 한 command에 넘깁니다. |
| 상태 또는 ownership 변화 | GameHub는 terminal room context와 deterministic key를 소유하고 repository는 durable effects를 소유합니다. client event의 matchId는 returned finalization result에서 나옵니다. |
| 주요 failure/edge path | 동일 room completion의 재호출은 같은 key로 DB existing result에 수렴합니다. 이 SHA에서 persistence reject 후 room을 어떻게 유지하는지는 아직 강화 전입니다. |
| 보장/비보장 | production completion이 canonical atomic/idempotent boundary를 사용합니다. transient DB failure 중 retry ownership은 `e939a5...`에서 완성됩니다. |
| 다음 관련 commit 연결 | `e939a5...`가 failed finalization 뒤 room을 버리지 않고 in-flight coalescing과 bounded backoff retry를 추가합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `582a1615a2c6` — `test(db): 경기 결과 단일 확정 조건 검증`
- 다음 Thread 관련 SHA: `e939a50948b2` — `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지`

### 5.8. `fix(game): 경기 결과 저장 실패를 재시도 가능한 상태로 유지`

| 항목 | 값 |
| --- | --- |
| SHA | `e939a50948b2` |
| Importance | A |
| Tags | REALTIME, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Keeps terminal room ownership and retries with the stable idempotency key.
- Classification summary: Preserve the terminal room and coalesce/retry finalization until durable commit succeeds or the hub is closed.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub가 terminal state에서 repository failure를 받은 뒤 room을 제거하거나 ownership을 놓으면 결과가 영구 누락되고 retry할 context/key가 사라질 수 있었습니다. |
| 핵심 boundary/decision | room에 shared `finishing` promise와 retry timer를 두고 failure 시 room을 유지합니다. retry delay는 250ms부터 증가해 5초에서 상한이며 모든 시도는 같은 `room:<id>:finished` key를 사용합니다. |
| 상태 또는 ownership 변화 | terminal room이 successful durable finalization까지 context와 retry ownership을 유지합니다. concurrent finalize caller는 동일 promise를 공유합니다. |
| 주요 failure/edge path | 실패 시 broadcast/remove를 하지 않고 timer를 예약합니다. 성공 후에만 terminal result를 보내고 room을 제거합니다. room discard/hub close는 retry timer를 정리합니다. process restart를 넘는 retry는 보장하지 않습니다. |
| 보장/비보장 | transient repository failure가 in-process terminal result를 잃게 하지 않고 duplicate work는 DB idempotency와 promise coalescing으로 제한됩니다. |
| 다음 관련 commit 연결 | 후속 room/drain Thread는 terminal room이 retry 중에도 owned resource라는 contract에 의존합니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | 게임이 terminal이면 persistence 호출 성공 여부와 관계없이 room lifecycle을 끝낼 수 있다는 가정. |
| 실제 실패 또는 위험 | DB 일시 장애에서 match/rating/tournament 결과가 누락되고 재시도 key/context가 사라집니다. |
| Root cause | terminal simulation state와 durable finalization 완료 상태를 같은 lifecycle 단계로 취급했습니다. |
| 수정된 invariant/decision | durable success 전까지 room을 유지하고 하나의 in-flight promise와 bounded backoff timer가 재시도를 소유합니다. |
| 변경 코드 | `apps/api/src/gameHub.ts` room `finishing`/retry timer, `finalizeRoom`, cleanup paths. |
| Regression evidence | repository rejection 뒤 room 유지, retry same key, success only-once removal/broadcast를 GameHub tests가 검사해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `10bf15723591` — `refactor(game): 경기 결과 확정 boundary 사용`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| A logical match result, participant statistics, rating history, and tournament progression commit atomically and idempotently. | `75bbc762e06d` durable identities | `83f9aee2522a` match idempotency → `e9d577ebc1ab` participant atomicity → `e338ea32b2a6` tournament transaction | `38504f041a6a` sequential partial-write 위험 | `83f9aee2522a`~`e338ea32b2a6` | `582a1615a2c6` | DB migration, `PostgresRepository.finalizeMatch` |
| Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup. | `e939a50948b2` finalization retry ownership | 후속 room/drain commits | persistence failure 시 terminal room 제거 위험 | `e939a50948b2` | GameHub retry tests/inspection | `gameHub.ts` finishing promise/retry timer/cleanup |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| match insert와 rating writes가 sequential | result key/history schema → idempotent transaction → ordered participant locks | `582a16...` concurrent/rollback DB tests | match/counters/ratings/history one commit |
| tournament progression이 durable match 밖 | `e338ea...` bracket locks/updates를 transaction에 편입 | semifinal/final integration cases | final row/winner one-time progression |
| GameHub가 canonical command를 사용하지 않음 | `10bf15...` stable room key + finalizeMatch | runtime code inspection | production path도 DB invariant 사용 |
| DB failure 뒤 terminal room ownership 상실 | `e939a5...` coalesced retry/backoff/cleanup | GameHub failure regression inspection | in-process transient failure에서 결과 보존 |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| logical result identity | 없음/generated match ID | `75bbc7...` caller result key | stable `room:<id>:finished` | DB unique row lifetime | migration/result_key, `10bf15...` |
| participant ratings/history | sequential statements | `e9d577...` ordered locks + transaction | finalize transaction | commit/rollback | `PostgresRepository.finalizeMatch` |
| tournament progression | 별도/없음 | `e338ea...` same transaction | finalize transaction | commit/rollback | tournament SQL path |
| terminal room/retry | GameHub terminal callback | `10bf15...` canonical command | `e939a5...` room finishing promise/timer | success/remove 또는 hub close | `gameHub.ts::finalizeRoom` |
| database resource | repository calls | transaction client | repository/pool | transaction release/pool close | DB implementation |

## 9. Thread 최종 상태

- 최종 authoritative owner: repository `finalizeMatch` transaction이 durable command를 소유하고 GameHub terminal room이 성공 전까지 retry context를 소유합니다.
- 최종 상태/invariant: 동일 result key는 하나의 match와 한 세트의 participant/history/tournament effects로 수렴하며 부분 commit을 남기지 않습니다.
- 남아 있는 의도적 제한 또는 비보장: retry timer는 process memory에 있어 crash/restart를 넘는 delivery guarantee는 없고 DB availability 자체를 보장하지 않습니다.
- 후속 Thread가 의존하는 contract: GameHub는 stable room-derived key와 complete terminal context를 넘기고 durable success 뒤에만 result broadcast/room removal을 수행합니다.
- 대표 코드 근거: `e338ea32b2a6 PostgresRepository.finalizeMatch`, `582a1615a2c6` PostgreSQL tests, `10bf15723591`/`e939a50948b2 apps/api/src/gameHub.ts`

## 10. 최종 architecture 또는 execution flow 정리

```text
[terminal PongSimulation result]
    ↓  GameHub.finalizeRoom / stable resultKey
[PostgreSQL transaction]
    ↓ unique match insert or existing read
[ordered participant locks → counters/ratings/history]
    ↓ tournament match/parent locks and progression
[COMMIT: one logical result]
    ↓ success
[game.finished broadcast → room removal]
    ↘ failure: same room + same key + 250ms..5s retry
```

- duplicate result key는 participant/tournament effects 전에 existing result로 수렴합니다.
- failure 중 terminal room은 drain이 기다려야 하는 owned work로 남습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 04-atomic-idempotent-match-finalization.md =====

===== BEGIN FILE: 05-room-lifecycle-connection-replacement-and-recovery.md =====
# 경기방 수명주기·연결 교체·복구

원문 Development Thread: `Room lifecycle, connection replacement, and recovery`

## 1. Thread 목표

- readiness, play, pause, disconnect, reconnect, expiry, finish를 explicit room-session state machine으로 복원합니다.
- 동일 사용자 새 socket이 기존 transport를 교체하면서 room side와 authority를 보존하는 atomic handoff를 추적합니다.
- 실제 disconnect와 same-user replacement를 구분하고 15초 reservation, forfeit/abandonment, browser fresh-ticket retry가 협력하는 방식을 확인합니다.

### Source에서 확정된 significance

> Socket loss and socket replacement are treated as lifecycle events, not immediate match termination or fresh matchmaking. The state machine, user-indexed connection map, reserved side, deadline, and browser retry policy cooperate to preserve one room and one player authority across transient transport failure.

### 직접 연결되는 Critical Invariants

> One user has one authoritative realtime connection, and reconnect replacement transfers ownership without creating a second match or losing the reserved room side.

> Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup.

### 직접 연결되는 Major Engineering Difficulties

> Coordinating readiness, pause, disconnect, reconnect, replacement, forfeit, retry, and final cleanup across interacting state machines.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 어떤 transition이 합법이며 idempotent이고, 어떤 transition이 무시되거나 terminal 결과를 만듭니까?
- snapshot phase는 state machine을 정의합니까, 아니면 accepted state를 반영합니까?
- same-user replacement에서 이전 heartbeat/buffer/membership/room side는 어떤 순서로 이전·정리됩니까?
- 실제 socket loss에서 disconnected-side reservation과 reconnect deadline의 소유자는 누구입니까?
- 한쪽/양쪽 disconnect expiry가 persisted forfeit와 non-persisted abandonment로 갈리는 조건은 무엇입니까?
- 브라우저 reconnect는 새 ticket을 쓰면서 왜 원래 queue/AI intent를 재전송하지 않습니까?

## 3. 완료 기준

- RoomSession의 상태와 transition 표를 실제 메서드 조건으로 완성할 수 있습니다.
- replacement, recoverable disconnect, expired disconnect 세 경로의 client/room/timer/scheduler 변화를 비교할 수 있습니다.
- displaced socket의 stale message가 새 room을 만들거나 현재 room을 조작하지 못하는 근거를 제시할 수 있습니다.
- 15초 경계의 reconnect와 expiry 결과를 deterministic test로 설명할 수 있습니다.
- browser reducer와 socket client가 recovery 동안 matchmaking을 막는 이유를 설명할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `aa5d6a338690` | `refactor(game): 게임 방 상태 전이 모델링` | S | REALTIME, ARCH, RISK | Defines legal readiness, pause, reconnect, expiry, and finish transitions. |
| 2 | `8f64dfc117f3` | `feat(game): 게임 방 상태를 RoomSession에 연결` | A | SIMULATION, REALTIME | Makes GameHub lifecycle changes subordinate to the state machine. |
| 3 | `a06d1705bbc9` | `feat(game): 사용자별 active connection 교체` | S | REALTIME, ARCH, RISK | Enforces one current transport and transfers room ownership atomically. |
| 4 | `c98d4b1e8b43` | `feat(game): 예약된 room connection 복구` | A | SIMULATION, REALTIME | Reattaches a returning identity only to an explicit reserved side. |
| 5 | `e593b1dd9fcd` | `feat(game): reconnect 예약 만료와 room 정리` | A | SIMULATION, REALTIME, RISK | Turns disconnect into a bounded reservation, forfeit, or non-persisted abandonment. |
| 6 | `113e39acc85c` | `test(game): reconnect 복구 동작 검증` | A | REALTIME, PERSISTENCE, RISK | Verifies replacement, deadline recovery, one finalization, and stale-socket rejection. |
| 7 | `4f5199097284` | `fix(web): 중단된 game reconnect 복구` | A | AUTH, REALTIME, WEB | Makes the browser request fresh tickets without replaying matchmaking intent. |

## 5. Commit별 학습 기록

### 5.1. `refactor(game): 게임 방 상태 전이 모델링`

| 항목 | 값 |
| --- | --- |
| SHA | `aa5d6a338690` |
| Importance | S |
| Tags | REALTIME, ARCH, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines legal readiness, pause, reconnect, expiry, and finish transitions.
- Classification summary: Introduce an explicit room-session state machine for readiness, play, pause, reconnection, and completion.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | ready/pause/disconnect/finish가 socket callbacks와 snapshot phase 조건에 흩어져 있어 합법 transition과 idempotency를 한 곳에서 판단할 수 없었습니다. |
| 핵심 boundary/decision | 새 `apps/api/src/game/roomSession.ts`의 `RoomSession`이 `waiting|playing|paused|reconnecting|finished`, ready/disconnected sides, resume state, 15초 deadline을 소유합니다. |
| 상태 또는 ownership 변화 | RoomSession이 lifecycle decision을 소유하고 snapshot은 accepted state를 반영하는 projection이 됩니다. socket·scheduler·DB는 이 class가 소유하지 않습니다. |
| 주요 failure/edge path | 두 side ready 전에는 playing이 되지 않고, pause/resume은 대응 state에서만 유효합니다. disconnect는 prior state를 기억하고, 한쪽 expiry는 opposite winner, 양쪽 expiry는 winner 없음, `finish`는 reconnect data를 제거해 반복 expiry를 막습니다. |
| 보장/비보장 | legal transition과 15초 recoverability/terminal outcome을 transport에서 분리합니다. GameHub가 이 state machine을 실제 자원 조작에 사용한다는 보장은 다음 commit입니다. |
| 다음 관련 commit 연결 | `8f64df...`가 GameHub room에 RoomSession을 연결하고 scheduler/snapshot mutation을 accepted transition에 종속시킵니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | snapshot phase와 socket callback만으로 lifecycle을 추론하면 replacement, reconnect, forfeit가 중복·역행할 수 있습니다. |
| 실패 위험 | ready 전 play, finished room resume, 같은 disconnect 두 번, timeout 후 reconnect, 양쪽 disconnect를 잘못된 winner로 확정할 수 있습니다. |
| 핵심 결정 | lifecycle 상태와 legal transition을 `RoomSession` 한 객체로 모델링합니다. |
| 구현 경로 | markReady → playing; pause/resume; disconnect(side, now) → reconnecting/deadline; reconnect(side, now) → restore; expireReconnect → winner/null; finish → terminal cleanup. |
| 수명주기·상태 | ready/disconnected sets, resume state, deadline은 RoomSession lifetime에 귀속되고 finish에서 제거됩니다. |
| 실패 처리 | 잘못된 current state 또는 deadline 밖의 operation은 state를 변경하지 않으며 expiry 결과는 terminal로 한 번만 전환됩니다. |
| 후속 검증 | `113e39...` fake-timer replacement/14,999ms/15,000ms/finalize-once tests. |
| Thread 전체 의미 | transport loss를 곧바로 match loss로 처리하지 않고 domain lifecycle event로 해석할 기반이 됩니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `8f64dfc117f3` — `feat(game): 게임 방 상태를 RoomSession에 연결`

### 5.2. `feat(game): 게임 방 상태를 RoomSession에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `8f64dfc117f3` |
| Importance | A |
| Tags | SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes GameHub lifecycle changes subordinate to the state machine.
- Classification summary: Integrate RoomSession decisions with room snapshots and scheduler start/stop.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | RoomSession은 독립 model이지만 GameHub가 snapshot phase와 timer를 직접 바꾸면 두 상태가 어긋날 수 있었습니다. |
| 핵심 boundary/decision | Room이 `session`, reconnect metadata를 보유하고 human/AI ready를 `markReady`에 전달합니다. accepted state만 snapshot phase에 복사하고 playing일 때 scheduler를 시작하며 pause/reconnecting에서 멈춥니다. |
| 상태 또는 ownership 변화 | RoomSession은 legal state, GameHub는 실제 scheduler/socket/snapshot resource를 소유합니다. GameHub는 반환 state를 projection합니다. |
| 주요 failure/edge path | invalid transition이면 resource side effect를 적용하지 않습니다. AI room은 right side를 ready로 표시해 left ready 후에만 시작합니다. |
| 보장/비보장 | domain lifecycle과 runtime scheduler/snapshot이 같은 accepted transition을 따릅니다. 동일 사용자 connection replacement와 reserved reconnect는 아직 없습니다. |
| 다음 관련 commit 연결 | `a06d17...`가 user-indexed active connection map과 atomic replacement를 구현합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `aa5d6a338690` — `refactor(game): 게임 방 상태 전이 모델링`
- 다음 Thread 관련 SHA: `a06d1705bbc9` — `feat(game): 사용자별 active connection 교체`

### 5.3. `feat(game): 사용자별 active connection 교체`

| 항목 | 값 |
| --- | --- |
| SHA | `a06d1705bbc9` |
| Importance | S |
| Tags | REALTIME, ARCH, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Enforces one current transport and transfers room ownership atomically.
- Classification summary: Add user-indexed connection authority and replace an existing socket without treating it as a match disconnect.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | connection map이 socket 중심이면 같은 user가 새 ticket으로 연결할 때 old/new socket이 동시에 room/queue를 조작하거나 old close가 forfeit를 유발할 수 있었습니다. |
| 핵심 boundary/decision | `clientsByUser`가 current client를 가리키고 replacement 시 이전 heartbeat와 snapshot buffer를 중지한 뒤 map에서 old client를 제거합니다. room side/roomId를 new client로 이전하고 current context/snapshot을 전송한 후 old socket을 4001로 닫습니다. |
| 상태 또는 ownership 변화 | user authority는 `clientsByUser`의 한 client에만 있습니다. replacement는 match membership을 유지하면서 transport resource owner만 원자적으로 바꿉니다. |
| 주요 failure/edge path | `receive`는 `clientsByUser.get(userId) !== client`인 stale client를 무시합니다. old close callback은 이미 map에서 제거돼 실제 disconnect/forfeit path로 들어가지 않습니다. |
| 보장/비보장 | 한 user가 동시에 authoritative realtime client 둘을 갖지 않고 same-user reconnect가 새 matchmaking을 만들지 않습니다. 연결이 완전히 끊긴 뒤 side reservation은 다음 commits 범위입니다. |
| 다음 관련 commit 연결 | `c98d4b...`가 current client가 없는 returning identity를 disconnected reservation과 연결합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | socket identity와 user identity가 동일하지 않아 reconnect가 duplicate authority 또는 accidental forfeit로 이어집니다. |
| 실패 위험 | old socket stale input, two rooms/queue entries, old close가 new connection의 match를 종료. |
| 핵심 결정 | user-indexed current-client map을 authority source로 두고 replacement 순서를 고정합니다. |
| 구현 경로 | new connect → old transport stop/remove → map authority 교체 → room side/client pointer 이전 → context/snapshot send → old close 4001. |
| 수명주기·상태 | heartbeat·snapshot buffer는 transport별, room side는 user/domain별이며 replacement에서 transport resource만 교체됩니다. |
| 실패 처리 | stale receive no-op, old close는 non-current라 disconnect mutation 없음. |
| 후속 검증 | `113e39...` replacement가 timeout/forfeit를 만들지 않고 stale socket이 무시되는 test. |
| Thread 전체 의미 | ‘한 사용자 한 권위 연결’이 명시적인 lookup invariant가 되어 reconnect 설계의 기준이 됩니다. |

비교 기준:
- 직전 Thread 관련 SHA: `8f64dfc117f3` — `feat(game): 게임 방 상태를 RoomSession에 연결`
- 다음 Thread 관련 SHA: `c98d4b1e8b43` — `feat(game): 예약된 room connection 복구`

### 5.4. `feat(game): 예약된 room connection 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `c98d4b1e8b43` |
| Importance | A |
| Tags | SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Reattaches a returning identity only to an explicit reserved side.
- Classification summary: Recover a disconnected player into the existing room reservation instead of matchmaking again.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | active replacement은 처리하지만 실제 socket loss 후 `clientsByUser`에 old client가 없으면 returning user와 기존 room side를 연결할 정보가 없었습니다. |
| 핵심 boundary/decision | GameHub가 rooms의 `disconnectedUsers`를 찾아 user ID와 reserved side를 확인하고 `session.reconnect(side, now)`가 허용할 때 new client를 그 side에 배치합니다. |
| 상태 또는 ownership 변화 | room이 disconnected user/side reservation을 소유하고 RoomSession이 deadline과 restore state를 판정합니다. new client는 transport ownership만 획득합니다. |
| 주요 failure/edge path | 명시적 reservation이 없거나 deadline/state가 허용하지 않으면 임의 room에 붙이지 않습니다. 다른 side가 아직 disconnected면 reconnecting 유지와 해당 client snapshot만 전송하고, 모두 돌아오면 timer clear·state projection·scheduler resume·broadcast를 수행합니다. |
| 보장/비보장 | returning identity가 같은 room/side에만 복구됩니다. reservation expiry와 terminal cleanup은 `e593b1...`에서 완성됩니다. |
| 다음 관련 commit 연결 | `e593b1...`가 disconnect 시 15초 reservation timer와 expiry forfeit/abandonment를 구현합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `a06d1705bbc9` — `feat(game): 사용자별 active connection 교체`
- 다음 Thread 관련 SHA: `e593b1dd9fcd` — `feat(game): reconnect 예약 만료와 room 정리`

### 5.5. `feat(game): reconnect 예약 만료와 room 정리`

| 항목 | 값 |
| --- | --- |
| SHA | `e593b1dd9fcd` |
| Importance | A |
| Tags | SIMULATION, REALTIME, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Turns disconnect into a bounded reservation, forfeit, or non-persisted abandonment.
- Classification summary: Own reconnect deadlines and terminal cleanup for one- and two-sided socket loss.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | reserved reconnect path는 있지만 실제 close에서 언제 reservation을 만들고 deadline 후 room을 어떻게 끝낼지 명확한 owner가 없었습니다. |
| 핵심 boundary/decision | 실제 current socket disconnect는 `session.disconnect(side, Date.now())`, `disconnectedUsers`, stopped scheduler, zero dy, paused snapshot, reconnect timer를 설정합니다. expiry는 RoomSession 결과에 따라 single-side forfeit를 score로 만들거나 both-disconnected room을 abandon합니다. |
| 상태 또는 ownership 변화 | room reconnect timer가 deadline wake-up을, RoomSession이 winner/null decision을, GameHub가 finalization/abandon cleanup을 소유합니다. |
| 주요 failure/edge path | winner 없음이면 persistence 없이 room을 버리고, winner가 있으면 winning score를 설정해 canonical finish path를 호출합니다. finalization/remove/close는 timer와 markers를 clear해 stale callback을 막습니다. |
| 보장/비보장 | disconnect는 최대 15초의 bounded reservation이며 expiry가 한 번의 forfeit 또는 non-persisted abandonment로 수렴합니다. browser가 자동으로 fresh ticket을 요청하는 것은 뒤 fix입니다. |
| 다음 관련 commit 연결 | `113e39...`가 replacement, 14,999ms recovery, 15,000ms expiry/finalize-once를 deterministic test로 고정합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `c98d4b1e8b43` — `feat(game): 예약된 room connection 복구`
- 다음 Thread 관련 SHA: `113e39acc85c` — `test(game): reconnect 복구 동작 검증`

### 5.6. `test(game): reconnect 복구 동작 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `113e39acc85c` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies replacement, deadline recovery, one finalization, and stale-socket rejection.
- Classification summary: Add fake-timer GameHub lifecycle regressions around replacement and the exact reconnect boundary.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | replacement/reservation/expiry 코드가 상호작용하지만 timer race와 old socket callback에서 exactly-once 결과를 보장하는 test evidence가 없었습니다. |
| 핵심 boundary/decision | fake sockets, memory repository mock, fake timers로 active replacement, 14,999ms reconnect/current snapshot/play, 15,000ms expiry, one finalization/no duplicate, stale socket rejection을 검사합니다. |
| 상태 또는 ownership 변화 | test clock이 deadline을 deterministic하게 이동하고 repository spy가 finalize call count를 관찰합니다. |
| 주요 failure/edge path | replacement는 reconnect timeout을 만들지 않아야 하고, deadline 직전은 recover, exact deadline은 terminal입니다. timer를 더 진행해도 finalize가 두 번 호출되면 실패합니다. |
| 보장/비보장 | in-process GameHub state machine과 timers에서 검사한 race가 보호됩니다. 실제 TCP proxy, browser process, PostgreSQL transaction은 포함하지 않습니다. |
| 다음 관련 commit 연결 | `4f5199...`가 browser transport가 같은 reservation window 안에서 fresh ticket으로 재접속하도록 수정합니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | 한 user 한 authoritative transport, 15초 reserved side, expiry/result exactly once. |
| 재현하는 failure/boundary | same-user replacement, stale socket, 14,999ms recovery, 15,000ms timeout, repeated timer advancement. |
| test technique | Vitest fake timers, fake WebSocket, repository spies/memory fixture. |
| 통과하는 production path | GameHub.connect/close → RoomSession disconnect/reconnect/expire → scheduler/timer → finalize/abandon. |
| 증명하는 것 | 검사한 in-process lifecycle에서 replacement와 boundary transition이 정확합니다. |
| 증명하지 않는 것 | real network partition, browser retry, DB failure, process restart는 증명하지 않습니다. |
| test 성격 | Deterministic state-machine and timer regression. |
| 후속 회귀 방지 설명 | old socket authority, deadline off-by-one, duplicate finalization, timer leak가 다시 생기면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `e593b1dd9fcd` — `feat(game): reconnect 예약 만료와 room 정리`
- 다음 Thread 관련 SHA: `4f5199097284` — `fix(web): 중단된 game reconnect 복구`

### 5.7. `fix(web): 중단된 game reconnect 복구`

| 항목 | 값 |
| --- | --- |
| SHA | `4f5199097284` |
| Importance | A |
| Tags | AUTH, REALTIME, WEB |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes the browser request fresh tickets without replaying matchmaking intent.
- Classification summary: Reconnect the browser transport within the room reservation window using a fresh one-time ticket.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | server는 15초 side reservation을 유지하지만 browser socket client가 close를 terminal로 처리하면 user가 새 ticket으로 돌아오지 못해 항상 forfeit됩니다. |
| 핵심 boundary/decision | `GameSocketClient`가 roomId가 있는 unexpected close에서 최대 15초 동안 250ms부터 2초 상한의 backoff로 매 시도 새 `/auth/ws-ticket`을 발급받아 연결합니다. reconnect open에는 initial `queue.join`/AI intent를 보내지 않습니다. |
| 상태 또는 ownership 변화 | browser connection generation이 abort controller와 retry timer를 소유하고 reducer가 recovery state 동안 새 matchmaking intent를 막습니다. |
| 주요 failure/edge path | room이 없으면 reconnect하지 않고, generation 변경/unmount/terminal event에서 pending ticket과 timer를 취소합니다. 새 ticket은 single-use이므로 재사용하지 않습니다. |
| 보장/비보장 | 일시 transport 중단에서 existing room reservation으로 복귀할 수 있고 duplicate queue/room intent를 만들지 않습니다. full process/browser E2E나 server restart 복구는 이 commit만으로 증명하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 contract는 server 15초 reservation과 browser 15초 fresh-ticket retry가 같은 room identity를 보존하는 것입니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | server가 room side를 예약하면 browser는 별도 retry 정책 없이도 복구할 수 있다는 가정. |
| 실제 실패 또는 위험 | unexpected close 뒤 client가 socket을 다시 만들지 않아 reservation이 항상 expiry/forfeit로 끝납니다. |
| Root cause | domain recovery window와 browser transport lifecycle이 연결되지 않았습니다. |
| 수정된 invariant/decision | room context가 있을 때만 bounded retry하고 각 attempt에 fresh one-time ticket을 사용하며 matchmaking intent는 replay하지 않습니다. |
| 변경 코드 | web `GameSocketClient`/play reducer의 generation, abort, retry timer, reconnect open branch. |
| Regression evidence | browser tests는 fresh ticket count, no duplicate queue.join, cleanup, 15초 budget을 검사해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `113e39acc85c` — `test(game): reconnect 복구 동작 검증`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| One user has one authoritative realtime connection, and reconnect replacement transfers ownership without creating a second match or losing the reserved room side. | `a06d1705bbc9` | `c98d4b1e8b43` reserved recovery → `e593b1dd9fcd` bounded deadline → `4f5199097284` browser retry | 기존 socket 중심 connection map | `a06d1705bbc9`, `4f5199097284` | `113e39acc85c` | `clientsByUser`, `disconnectedUsers`, `RoomSession`, browser `GameSocketClient` |
| Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup. | `8f64dfc117f3` scheduler follows session | `e593b1dd9fcd` reconnect timer cleanup, `4f5199097284` browser timer/abort cleanup | stale timeout/old socket 가능성 | `a06d1705bbc9`, `e593b1dd9fcd` | `113e39acc85c` | room timers, client heartbeat/snapshot buffer, browser generation |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| snapshot phase와 callbacks가 lifecycle을 암묵적으로 정의 | `aa5d6...` RoomSession → `8f64df...` GameHub integration | `113e39...` fake-timer tests | legal transition와 runtime side effect 일치 |
| 같은 user의 old/new socket이 동시에 authoritative | `a06d17...` clientsByUser atomic handoff | replacement/stale-socket test | 한 user 한 current transport |
| disconnect 즉시 match 종료 또는 room 상실 | `c98d4...` reservation recovery + `e593b1...` 15초 expiry | 14,999/15,000ms tests | bounded recoverable loss |
| browser가 reservation을 사용하지 못함 | `4f5199...` fresh-ticket retry/no intent replay | browser code/tests | same room transport recovery |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| room domain state | snapshot phase/callbacks | `aa5d6...` RoomSession | RoomSession | finish에서 reconnect state clear | `roomSession.ts` |
| current connection authority | socket/client collections | `a06d17...` user index | `clientsByUser` | replacement/close/hub close | `gameHub.ts` |
| reserved side/deadline | 없음 | `c98d4...` disconnectedUsers, `e593b1...` timer | room + RoomSession | reconnect/finalize/abandon/close | GameHub reconnect methods |
| browser retry | 없음 | `4f5199...` generation/backoff | GameSocketClient | success/terminal/unmount/15초 expiry | web socket client/reducer |
| scheduler | room timer | session-gated start/stop | GameHub scheduler owner | pause/disconnect/finalize/remove | `startRoomScheduler`, unregister/stop paths |

## 9. Thread 최종 상태

- 최종 authoritative owner: RoomSession이 lifecycle을, `clientsByUser`가 current transport authority를, room timer가 reservation deadline을, browser generation이 retry work를 소유합니다.
- 최종 상태/invariant: same-user replacement는 disconnect가 아니며 실제 loss는 15초간 같은 room side를 보존하고 정확히 한 recovery/forfeit/abandonment로 끝납니다.
- 남아 있는 의도적 제한 또는 비보장: process restart 후 in-memory room 복구, multi-instance routing, real network proxy E2E는 보장하지 않습니다.
- 후속 Thread가 의존하는 contract: browser는 room context가 있을 때만 fresh ticket으로 재접속하고 queue/AI intent를 다시 보내지 않으며 server는 identity가 일치하는 reserved side만 반환합니다.
- 대표 코드 근거: `aa5d6a338690 roomSession.ts`, `a06d1705bbc9`/`e593b1dd9fcd gameHub.ts`, `113e39acc85c` tests, `4f5199097284` web reconnect code

## 10. 최종 architecture 또는 execution flow 정리

```text
[current socket/user]
    ├─ same-user new socket → atomic transport replacement → same room/side
    └─ actual close
          ↓ RoomSession.disconnect(side, now)
       [reconnecting + 15s deadline, scheduler stopped]
          ├─ fresh-ticket return before deadline → reserved side reattach
          │      ↓ all sides present → prior state + scheduler resume
          └─ deadline expiry
                 ├─ one side absent → opposite winner → canonical finalize
                 └─ both absent → non-persisted abandonment
```

- snapshot phase는 RoomSession decision을 반영하며 lifecycle source of truth가 아닙니다.
- replacement old socket은 current map에서 제거된 뒤 닫히므로 close callback이 forfeit를 만들지 않습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 05-room-lifecycle-connection-replacement-and-recovery.md =====

===== BEGIN FILE: 06-matchmaking-reservation-ownership-and-rollback.md =====
# 매치메이킹 예약 소유권과 롤백

원문 Development Thread: `Matchmaking reservation ownership and rollback`

## 1. Thread 목표

- GameHub 내부 timer-backed queue에서 독립 `Matchmaker` state machine으로 queue/reservation ownership이 이동하는 과정을 추적합니다.
- closest-rating pairing, guest/registered pool 분리, six-second AI fallback, duplicate membership, leave/release 의미를 복원합니다.
- room 생성 중 부분 실패와 모든 terminal path가 reservation을 누락 없이 해제하는 rollback/cleanup 구조를 확인합니다.

### Source에서 확정된 significance

> The initial timer-backed queue works but leaves GameHub with multiple representations of availability. The later Matchmaker abstraction makes reservation state explicit; integration and cleanup commits then eliminate split ownership. The progression matters because stale reservations would permanently remove users or allow one user to occupy two matches.

### 직접 연결되는 Critical Invariants

> Queue and reservation membership have one owner and are released on every leave, disconnect, rollback, drain, abandonment, failure, and finalization path.

> The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes.

### 직접 연결되는 Major Engineering Difficulties

> Coordinating readiness, pause, disconnect, reconnect, replacement, forfeit, retry, and final cleanup across interacting state machines.

> Preserving domain correctness during database failure, tournament-start rollback, match-finalization retry, process drain, and deployment shutdown.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- queued, matched/reserved, duplicate, fallback-ready 상태는 어떤 자료구조와 반환형으로 표현됩니까?
- closest candidate 선택에서 rating window, pool kind, tie order, injected clock이 어떻게 사용됩니까?
- `leaveQueue`와 `release`는 왜 분리되며 잘못 사용하면 어떤 invariant가 깨집니까?
- asynchronous NPC lookup 전후 어떤 socket/room/drain 조건을 다시 검증합니까?
- GameHub에 남는 transport metadata와 Matchmaker가 소유하는 domain state는 각각 무엇입니까?
- room publication 도중 observer/send/snapshot failure가 발생하면 어떤 획득 자원을 역순으로 되돌립니까?

## 3. 완료 기준

- Matchmaker 상태 전이와 반환 outcome을 실제 타입 및 메서드로 그릴 수 있습니다.
- 동일 사용자가 queued/reserved/active room 중 둘 이상에 동시에 존재하지 않는 근거를 제시할 수 있습니다.
- normal match, AI fallback, leave, disconnect, drain, room creation failure, finalization/abandonment의 release 경로를 모두 기록할 수 있습니다.
- GameHub의 duplicate queue 표현이 제거되기 전후 ownership 차이를 설명할 수 있습니다.
- rollback test가 partial publication과 stale reservation을 어떻게 재현하는지 구분할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1122e6a4b901` | `feat(game): 대기 플레이어 NPC fallback 구성` | B | REALTIME | Introduces timed AI fallback with explicit timer cleanup. |
| 2 | `1ec8335b0e75` | `refactor(game): matchmaking player와 fallback 계약 정의` | A | REALTIME, REFACTOR | Defines queued, matched, duplicate, fallback, and release outcomes independently of sockets. |
| 3 | `a4f59a2e8192` | `refactor(game): rating 기반 closest-pair queue 구현` | A | REALTIME, REFACTOR | Implements deterministic compatible-pool, closest-rating pairing. |
| 4 | `7871e29278c2` | `refactor(game): AI fallback과 reservation lifecycle 구현` | A | REALTIME, REFACTOR | Separates queue cancellation from releasing an assigned match reservation. |
| 5 | `e53559ef3a11` | `refactor(game): Matchmaker queue reservation을 GameHub에 연결` | A | REALTIME, REFACTOR | Moves duplicate-user reservation and PvP selection behind Matchmaker. |
| 6 | `51f36aa50596` | `refactor(game): Matchmaker AI fallback를 GameHub에 연결` | A | REALTIME, OPERATIONS, RISK | Claims delayed fallback through the same state machine and revalidates asynchronous work. |
| 7 | `a23fc26a7f82` | `refactor(game): queue와 reservation cleanup 일원화` | S | REALTIME, ARCH, RISK | Removes duplicate queue ownership and centralizes every release path. |
| 8 | `b5bfeee0e23e` | `refactor(game): room 생성과 finalization cleanup 보장` | A | REALTIME, OBSERVABILITY, RISK | Rolls back partially published rooms and guarantees terminal removal. |
| 9 | `112228db8878` | `test(game): matchmaking lifecycle 검증` | A | REALTIME, RISK, TEST | Protects matchability after failure, abandonment, forfeit, and rollback. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): 대기 플레이어 NPC fallback 구성`

| 항목 | 값 |
| --- | --- |
| SHA | `1122e6a4b901` |
| Importance | B |
| Tags | REALTIME |
| 학습 깊이 | Thread 흐름에서 맡는 구현 역할과 필요한 상태 변화를 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Introduces timed AI fallback with explicit timer cleanup.
- Classification summary: Add a bounded waiting policy for the human matchmaking queue.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | human queue가 상대를 못 찾으면 무기한 대기하며 queue entry에 fallback lifecycle이 없었습니다. |
| 핵심 boundary/decision | GameHub `QueueEntry`가 6초 `npcFallbackTimer`를 소유하고 timeout 시 entry가 여전히 queue에 있고 socket open/room 없음이면 rating이 가장 가까운 active NPC로 room을 만듭니다. |
| 상태 또는 ownership 변화 | fallback timer는 queue entry가 소유하고 normal match, leave, disconnect, prune가 clear합니다. room은 socket이 없는 `npcUser` identity를 별도로 보관합니다. |
| 주요 failure/edge path | timeout callback은 current membership/socket/room을 재검증해 이미 matched user에게 두 번째 room을 만들지 않습니다. |
| 보장/비보장 | human wait가 6초로 bounded되고 stale timer cleanup이 있습니다. queue/reservation domain owner는 여전히 GameHub 배열과 timer에 분산돼 있습니다. |
| 다음 관련 commit 연결 | `1ec833...`이 socket과 독립된 Matchmaker player/outcome contract를 정의합니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `1ec8335b0e75` — `refactor(game): matchmaking player와 fallback 계약 정의`

### 5.2. `refactor(game): matchmaking player와 fallback 계약 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `1ec8335b0e75` |
| Importance | A |
| Tags | REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines queued, matched, duplicate, fallback, and release outcomes independently of sockets.
- Classification summary: Define Matchmaker types and state transitions without binding them to WebSocket clients.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | queue state가 GameHub entry/timer와 boolean 조건으로만 표현돼 duplicate, matched reservation, fallback deadline, release 의미가 명시적이지 않았습니다. |
| 핵심 boundary/decision | `MatchmakingKind`, `MatchmakingPlayer`, pair, join outcomes(`queued|matched|duplicate`), fallback outcomes(`waiting|ready|unavailable`), status와 injected clock/rating window options를 정의합니다. |
| 상태 또는 ownership 변화 | Matchmaker가 domain status vocabulary를 소유할 준비를 하고 GameHub/socket은 transport metadata로 분리될 수 있습니다. |
| 주요 failure/edge path | 6,000ms fallback constant와 option validation을 명시합니다. 아직 complete queue algorithm/integration은 뒤 commits 범위입니다. |
| 보장/비보장 | 상태 전이를 typed outcome으로 설명할 수 있습니다. 실제 closest pairing, leave/release semantics, single ownership은 아직 없습니다. |
| 다음 관련 commit 연결 | `a4f59a...`가 compatible pool 내 deterministic closest-rating pairing을 구현합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `1122e6a4b901` — `feat(game): 대기 플레이어 NPC fallback 구성`
- 다음 Thread 관련 SHA: `a4f59a2e8192` — `refactor(game): rating 기반 closest-pair queue 구현`

### 5.3. `refactor(game): rating 기반 closest-pair queue 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `a4f59a2e8192` |
| Importance | A |
| Tags | REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Implements deterministic compatible-pool, closest-rating pairing.
- Classification summary: Implement Matchmaker queue membership and deterministic closest-candidate selection.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | typed contract는 있으나 실제 queue와 candidate selection을 Matchmaker가 소유하지 않았습니다. |
| 핵심 boundary/decision | Matchmaker가 queue와 `playerStatuses`를 보유하고 같은 kind, rating difference ≤ configured max 중 absolute difference가 가장 작은 기존 candidate를 선택합니다. `< closestDifference` 조건으로 동률이면 먼저 들어온 candidate가 유지됩니다. |
| 상태 또는 ownership 변화 | queue membership과 queued/matched status가 Matchmaker 내부 map/list로 이동합니다. injected clock이 joined/fallback timing을 결정합니다. |
| 주요 failure/edge path | duplicate join은 status를 변경하지 않고 duplicate outcome을 반환합니다. 반환 player/pair는 defensive copy로 외부 mutation을 막습니다. |
| 보장/비보장 | same pool에서 deterministic closest pairing과 duplicate membership 차단을 제공합니다. AI fallback claim과 reservation release 차이는 다음 commit입니다. |
| 다음 관련 commit 연결 | `7871e2...`가 queued cancellation과 matched reservation release를 분리합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `1ec8335b0e75` — `refactor(game): matchmaking player와 fallback 계약 정의`
- 다음 Thread 관련 SHA: `7871e29278c2` — `refactor(game): AI fallback과 reservation lifecycle 구현`

### 5.4. `refactor(game): AI fallback과 reservation lifecycle 구현`

| 항목 | 값 |
| --- | --- |
| SHA | `7871e29278c2` |
| Importance | A |
| Tags | REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Separates queue cancellation from releasing an assigned match reservation.
- Classification summary: Implement fallback claiming and distinct leave/release transitions for queued and matched players.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | matched pair와 queued player를 같은 remove operation으로 다루면 assigned reservation을 잘못 해제하거나 queued status를 남길 수 있었습니다. |
| 핵심 boundary/decision | fallback claim은 queued player/deadline에서만 `waiting|ready|unavailable`을 반환하고 ready 시 queue에서 제거해 matched로 표시합니다. `leaveQueue`는 queued만 제거하고 `release`는 queued/matched status 모두 정리합니다. |
| 상태 또는 ownership 변화 | Matchmaker가 queued→matched reservation lifecycle과 deadline을 authoritative하게 소유합니다. |
| 주요 failure/edge path | deadline 전 claim은 remaining time과 waiting을 반환하며 이미 matched/unknown이면 unavailable입니다. 잘못된 `leaveQueue`가 matched reservation을 조용히 없애지 않습니다. |
| 보장/비보장 | queue cancellation과 assigned match cleanup의 의미가 분리됩니다. GameHub가 이 API를 모든 path에서 사용하는지는 integration commits가 필요합니다. |
| 다음 관련 commit 연결 | `e53559...`가 PvP join/selection을 Matchmaker 뒤로 옮기지만 duplicate GameHub queue 표현은 잠시 남습니다. |

비교 기준:
- 직전 Thread 관련 SHA: `a4f59a2e8192` — `refactor(game): rating 기반 closest-pair queue 구현`
- 다음 Thread 관련 SHA: `e53559ef3a11` — `refactor(game): Matchmaker queue reservation을 GameHub에 연결`

### 5.5. `refactor(game): Matchmaker queue reservation을 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `e53559ef3a11` |
| Importance | A |
| Tags | REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Moves duplicate-user reservation and PvP selection behind Matchmaker.
- Classification summary: Integrate Matchmaker join outcomes with GameHub transport clients and room creation.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | GameHub 배열이 membership/selection을 직접 결정해 Matchmaker state와 production behavior가 연결되지 않았습니다. |
| 핵심 boundary/decision | GameHub가 registered/guest kind와 rating을 `Matchmaker.join`에 넘기고 duplicate/queued/matched outcome에 따라 error, queue metadata, room creation을 수행합니다. rating window는 200입니다. |
| 상태 또는 ownership 변화 | domain status는 Matchmaker로 이동하지만 이 SHA에는 기존 queue/queueEntries 표현이 함께 남아 split ownership이 완전히 제거되지 않았습니다. |
| 주요 failure/edge path | matched opponent transport가 없거나 createRoom이 throw하면 두 player를 `release`합니다. 그렇지 않으면 stale reservation이 남습니다. |
| 보장/비보장 | PvP selection과 duplicate reservation은 Matchmaker outcome에 종속됩니다. AI fallback과 all-path cleanup은 아직 통합 중입니다. |
| 다음 관련 commit 연결 | `51f36a...`가 fallback timer callback도 Matchmaker claim과 asynchronous revalidation을 사용하게 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `7871e29278c2` — `refactor(game): AI fallback과 reservation lifecycle 구현`
- 다음 Thread 관련 SHA: `51f36aa50596` — `refactor(game): Matchmaker AI fallback를 GameHub에 연결`

### 5.6. `refactor(game): Matchmaker AI fallback를 GameHub에 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `51f36aa50596` |
| Importance | A |
| Tags | REALTIME, OPERATIONS, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Claims delayed fallback through the same state machine and revalidates asynchronous work.
- Classification summary: Route AI fallback deadlines and claims through Matchmaker while guarding async NPC lookup races.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | PvP는 Matchmaker를 사용하지만 fallback timer가 GameHub local queue state만 보면 두 ownership model이 충돌합니다. |
| 핵심 boundary/decision | timer는 Matchmaker fallback deadline에 맞춰 arm하고 callback에서 current entry/socket/room을 확인한 뒤 `claimFallback`을 호출합니다. `waiting`이면 reschedule, `unavailable`이면 metadata cleanup, `ready`이면 NPC lookup 후 조건을 다시 검사합니다. |
| 상태 또는 ownership 변화 | fallback status/deadline은 Matchmaker, timer/client reference는 GameHub `queueEntries`가 소유합니다. |
| 주요 failure/edge path | async NPC lookup 동안 drain, disconnect, room 생성, entry replacement가 발생할 수 있어 lookup 전후 acceptingMatches/socket/room/entry를 재검증하고 모든 실패에서 release합니다. |
| 보장/비보장 | AI fallback도 동일 reservation state machine을 통과합니다. duplicate queue representation과 cleanup scatter는 `a23fc2...`에서 제거됩니다. |
| 다음 관련 commit 연결 | `a23fc2...`가 GameHub queue array를 없애고 release path를 중앙화해 single owner invariant를 완성합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `e53559ef3a11` — `refactor(game): Matchmaker queue reservation을 GameHub에 연결`
- 다음 Thread 관련 SHA: `a23fc26a7f82` — `refactor(game): queue와 reservation cleanup 일원화`

### 5.7. `refactor(game): queue와 reservation cleanup 일원화`

| 항목 | 값 |
| --- | --- |
| SHA | `a23fc26a7f82` |
| Importance | S |
| Tags | REALTIME, ARCH, RISK |
| 학습 깊이 | Architecture/invariant 중심으로 직전 상태, 결정, 핵심 전이, ownership, failure, 후속 검증까지 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Removes duplicate queue ownership and centralizes every release path.
- Classification summary: Make Matchmaker the sole domain owner and reduce GameHub state to transport/timer metadata.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | Matchmaker status와 GameHub queue array/entry map이 동시에 user availability를 표현해 한쪽 cleanup 누락이 stale reservation 또는 duplicate match를 만들 수 있었습니다. |
| 핵심 boundary/decision | legacy queue array/findClosest를 삭제하고 Matchmaker를 sole domain owner로 둡니다. GameHub `queueEntries`는 client와 fallback timer를 찾는 transport metadata만 보관하며 shared cleanup helper가 leave/prune/drain/close/abandon/failure/finalization에서 `release`를 호출합니다. |
| 상태 또는 ownership 변화 | queued/matched membership은 Matchmaker 한 곳, timer/socket reference는 GameHub 한 곳으로 분리됩니다. release는 idempotent domain cleanup primitive입니다. |
| 주요 failure/edge path | normal match, AI fallback, disconnect, drain, room abandonment, finalization failure, hub close 중 하나라도 release를 빠뜨리면 user가 영구 matched로 남습니다. 공통 helper로 모든 terminal path를 수렴시킵니다. |
| 보장/비보장 | availability의 competing representation이 없고 every known terminal path가 reservation을 해제합니다. room publication 중 partial resource rollback은 다음 commit에서 강화됩니다. |
| 다음 관련 commit 연결 | `b5bfee...`가 createRoom publish failure와 finalization observer/broadcast failure에서도 acquired resource를 rollback/finally cleanup합니다. |

#### Architecture / invariant 복원

| 축 | 복원 결과 |
| --- | --- |
| 문제 | domain queue와 transport queue가 동일 사실을 각각 보유해 cleanup 원자성이 깨집니다. |
| 실패 위험 | user가 영구 queue 밖/예약 상태에 남거나 두 match에 동시에 배정됩니다. |
| 핵심 결정 | Matchmaker를 sole status owner로 하고 GameHub에는 transport/timer metadata만 남깁니다. |
| 구현 경로 | join/claim/leave/release는 Matchmaker; GameHub path는 metadata helper를 거쳐 timer clear + release. |
| 수명주기·상태 | queued → matched/reserved → room active → release; fallback timer는 queued metadata lifetime에만 존재합니다. |
| 실패 처리 | leave, socket prune, drain, async lookup failure, create failure, abandon, finalization/remove 모두 shared cleanup에 수렴합니다. |
| 후속 검증 | `112228...` forfeit/abandon/observer throw 뒤 같은 users가 즉시 다시 match되는 regression. |
| Thread 전체 의미 | matchability가 여러 array의 우연한 동기화가 아니라 단일 state machine invariant가 됩니다. |

비교 기준:
- 직전 Thread 관련 SHA: `51f36aa50596` — `refactor(game): Matchmaker AI fallback를 GameHub에 연결`
- 다음 Thread 관련 SHA: `b5bfeee0e23e` — `refactor(game): room 생성과 finalization cleanup 보장`

### 5.8. `refactor(game): room 생성과 finalization cleanup 보장`

| 항목 | 값 |
| --- | --- |
| SHA | `b5bfeee0e23e` |
| Importance | A |
| Tags | REALTIME, OBSERVABILITY, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Rolls back partially published rooms and guarantees terminal removal.
- Classification summary: Treat room publication as an acquisition sequence with explicit rollback and terminal cleanup in `finally`.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | reservation을 release해도 createRoom 중 room map/client roomId/scheduler/timer/observer 일부가 publish된 뒤 throw하면 ghost room과 stale ownership이 남을 수 있었습니다. |
| 핵심 boundary/decision | createRoom publication을 `try/catch`로 감싸 scheduler/reconnect timer/rooms/client roomIds 등 이미 획득한 자원을 역순 rollback하고 original error를 재throw합니다. finalization observer/broadcast는 `try/finally`로 room removal을 보장합니다. |
| 상태 또는 ownership 변화 | createRoom call이 publication transaction의 in-memory owner이며 success 전 자원은 provisional입니다. terminal cleanup은 `finally`가 소유합니다. |
| 주요 failure/edge path | observer/send/snapshot setup가 throw해도 active room과 reservation이 남지 않습니다. cleanup error로 original failure를 덮지 않도록 원래 exception을 보존합니다. |
| 보장/비보장 | partial room publication이 외부 state에 남지 않고 terminal observer failure도 room removal을 막지 않습니다. DB transaction 자체는 Thread 04 invariant를 사용합니다. |
| 다음 관련 commit 연결 | `112228...`가 observer throw failure injection 뒤 activeRooms/queuedPlayers=0과 immediate rematch를 검증합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `a23fc26a7f82` — `refactor(game): queue와 reservation cleanup 일원화`
- 다음 Thread 관련 SHA: `112228db8878` — `test(game): matchmaking lifecycle 검증`

### 5.9. `test(game): matchmaking lifecycle 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `112228db8878` |
| Importance | A |
| Tags | REALTIME, RISK, TEST |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Protects matchability after failure, abandonment, forfeit, and rollback.
- Classification summary: Add deterministic GameHub/Matchmaker lifecycle regressions including injected room-publication failure.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | single-owner와 rollback code는 있지만 모든 terminal path 후 user가 실제로 다시 matchable한지를 상태와 behavior 모두로 검증하지 않았습니다. |
| 핵심 boundary/decision | fake timers/GameHub fixtures가 rating window, forfeit release/rematch, empty-room abandonment/rematch를 검사하고 observer가 room creation에서 한 번 throw하도록 deterministic failure injection합니다. |
| 상태 또는 ownership 변화 | observer throw가 publication sequence의 지정 지점에서 실패를 주입하고 stats가 `activeRooms:0`, `queuedPlayers:0`을 관찰합니다. |
| 주요 failure/edge path | 첫 create가 실패한 뒤 같은 두 user를 다시 join시켜 즉시 match되는지 확인해 단순 map size뿐 아니라 reservation 재획득 가능성을 증명합니다. |
| 보장/비보장 | 검사한 in-process failure/terminal paths가 stale reservation과 ghost room을 남기지 않습니다. 장기 fairness, multi-instance queue, production load는 증명하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 상태는 Matchmaker sole ownership과 all-path release/room rollback입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | queued/reserved/active room ownership은 중복되지 않고 terminal/rollback 뒤 모두 재획득 가능합니다. |
| 재현하는 failure/boundary | rating window, forfeit, empty abandonment, room-created observer throw after partial publication. |
| test technique | Fake timers, GameHub state inspection, deterministic observer failure injection, behavioral rematch assertion. |
| 통과하는 production path | Matchmaker.join/release → GameHub createRoom/rollback → terminal cleanup → same users rejoin. |
| 증명하는 것 | 검사한 path에서 activeRooms/queuedPlayers가 0으로 수렴하고 stale reservation이 남지 않습니다. |
| 증명하지 않는 것 | global fairness, starvation, distributed queue coordination, sustained fault load는 증명하지 않습니다. |
| test 성격 | Deterministic lifecycle, rollback, and failure-injection regression. |
| 후속 회귀 방지 설명 | release 누락, provisional room leak, original error masking, rematch 불가 상태가 생기면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `b5bfeee0e23e` — `refactor(game): room 생성과 finalization cleanup 보장`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| Queue and reservation membership have one owner and are released on every leave, disconnect, rollback, drain, abandonment, failure, and finalization path. | `1ec8335b0e75` explicit statuses | `a4f59a2e8192` queue owner → `7871e29278c2` release semantics → `a23fc26a7f82` sole ownership → `b5bfeee0e23e` rollback | `e53559ef3a11`~`51f36aa50596` 동안 GameHub duplicate representation | `a23fc26a7f82`, `b5bfeee0e23e` | `112228db8878` | `Matchmaker`, GameHub queueEntries/cleanup/createRoom |
| The server is the sole authority for game rules, scores, phases, room membership, matchmaking, and persisted outcomes. | `1122e6a4b901` server fallback | `a4f59a2e8192` deterministic selection, `a23fc26a7f82` sole owner | 해당 없음 | 해당 없음 | `112228db8878` | server-side Matchmaker outcomes |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| timer-backed GameHub queue가 availability를 직접 소유 | Matchmaker contract/algorithm/fallback integration | unit/GameHub lifecycle tests | socket-independent explicit statuses |
| Matchmaker와 GameHub가 queue를 중복 표현 | `a23fc2...` legacy queue 삭제 + centralized release | forfeit/abandon/rematch tests | single domain owner |
| room publication 중 throw가 partial resources를 남김 | `b5bfee...` reverse rollback/finally removal | `112228...` observer throw injection | ghost room·stale reservation 없음 |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| queued/matched status | GameHub queue array | `a4f59...` Matchmaker + temporary duplicate | Matchmaker | leave/release/close/drain/final paths | `matchmaker.ts`, `a23fc...` |
| transport/timer metadata | QueueEntry | `51f36...` deadline integration | GameHub `queueEntries` | metadata cleanup helper | `gameHub.ts` |
| fallback deadline/claim | entry timer | `7871e...` Matchmaker claim | Matchmaker status + GameHub timer | claim/leave/release | fallback methods |
| provisional room resources | ad-hoc create | `b5bfee...` publication transaction | GameHub createRoom scope | catch rollback/finally remove | createRoom/finalization |

## 9. Thread 최종 상태

- 최종 authoritative owner: Matchmaker가 queue/reservation domain state를 유일하게 소유하고 GameHub는 socket/timer metadata와 room publication을 소유합니다.
- 최종 상태/invariant: user는 queued·reserved·active room 중 하나에만 있고 모든 leave/disconnect/drain/rollback/abandon/failure/finalization path가 release합니다.
- 남아 있는 의도적 제한 또는 비보장: single-process in-memory queue이며 distributed matchmaking, persistence, fairness/starvation guarantee는 없습니다.
- 후속 Thread가 의존하는 contract: GameHub는 Matchmaker outcome만으로 room을 만들고 async 작업 후 상태를 재검증하며 publication 실패 시 획득 자원을 역순으로 되돌립니다.
- 대표 코드 근거: `a4f59a2e8192`/`7871e29278c2 Matchmaker`, `a23fc26a7f82` centralized cleanup, `b5bfeee0e23e` rollback, `112228db8878` failure injection tests

## 10. 최종 architecture 또는 execution flow 정리

```text
[queue.join user/kind/rating]
    ↓ Matchmaker.join
[duplicate | queued(deadline) | matched(pair)]
    ├─ queued → GameHub timer metadata → claimFallback → async NPC revalidation
    └─ matched → provisional createRoom
            ├─ success → active room; reservation released at terminal ownership transfer
            └─ throw → scheduler/timer/map/client rollback + Matchmaker.release
[leave/disconnect/drain/abandon/finalize/close]
    ↓ shared cleanup
[timer clear + metadata delete + idempotent Matchmaker.release]
```

- `leaveQueue`는 queued cancellation만, `release`는 queued/matched reservation 전체 정리입니다.
- failure test는 map size와 함께 같은 users의 즉시 rematch를 확인합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 06-matchmaking-reservation-ownership-and-rollback.md =====

===== BEGIN FILE: 07-guest-mode-as-isolated-transient-trust-domain.md =====
# 격리된 임시 신뢰 도메인으로서의 게스트 모드

원문 Development Thread: `Guest mode as an isolated transient trust domain`

## 1. Thread 목표

- guest를 약한 registered account가 아니라 별도의 signed identity, ticket, lease, capability, matchmaking, persistence domain으로 추적합니다.
- database 없이 발급·검증되는 guest cookie와 one-time ticket, IP/process capacity lease의 lifecycle을 복원합니다.
- registered data 조회와 write, mixed matchmaking, durable result를 차단하면서 제한된 reconnect recovery만 허용하는 경계를 확인합니다.

### Source에서 확정된 significance

> Guest mode is not implemented as a weaker registered account. It is a separate signed identity, capability, matchmaking, persistence, and resource domain. The sequence is significant because each integration point explicitly prevents transient public traffic from acquiring durable data or unbounded process state.

### 직접 연결되는 Critical Invariants

> Guest identities, matchmaking pools, capabilities, persistence, tickets, leases, and retained results remain isolated and resource-bounded.

> The durable browser session is carried only by an HttpOnly cookie; raw WebSocket tickets are short-lived, single-use, hashed at rest, bounded during authentication, and excluded from logs.

> Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup.

### 직접 연결되는 Major Engineering Difficulties

> Offering a public guest mode without allowing transient identities to cross into registered data, social features, ratings, or unbounded in-memory resources.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- guest token payload와 signature는 어떤 값으로 구성되며 address/version/expiry를 어디에서 검증합니까?
- ticket issuance limit과 live connection lease limit은 왜 별도이며 reconnect replacement에서 stale release를 어떻게 막습니까?
- demo mode가 registered session/database fallback을 하지 않는 branch는 어디입니까?
- guest route가 repository data를 조회한 뒤 필터링하는 것이 아니라 호출 자체를 피하는 근거는 무엇입니까?
- guest와 registered matchmaking pool 및 AI fallback source는 어떻게 분리됩니까?
- non-persisted result의 2분 보존은 어떤 map/timer ownership과 race guard를 사용합니까?
- IP window와 pending ticket map이 무한 증가하지 않도록 어떤 capacity와 prune path를 둡니까?

## 3. 완료 기준

- signed cookie → in-memory ticket → live lease의 trust chain과 각 TTL/capacity를 설명할 수 있습니다.
- guest가 접근 가능한 HTTP/WebSocket 기능과 registered-only 기능을 코드 branch로 구분할 수 있습니다.
- guest room이 registered user/NPC/persistence와 섞이지 않는 근거를 제시할 수 있습니다.
- transient result recovery가 durable history나 rating을 만들지 않는지 확인할 수 있습니다.
- fake-timer tests가 window/ticket/result cleanup과 reconnect intent를 어떻게 검증하는지 정리할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `cacd4c22d705` | `feat(guest): signed guest session token 정의` | A | AUTH, PERSISTENCE | Creates tamper-evident, address-bound, expiring guest identity without database sessions. |
| 2 | `17a1dd501b1b` | `feat(guest): guest resource lease 수명주기 추가` | A | AUTH, PROTOCOL, REALTIME | Binds guest connection limits to lease identity and socket lifetime. |
| 3 | `a5c06c561e00` | `feat(guest): guest session과 WebSocket 인증 연결` | A | AUTH, REALTIME, PERSISTENCE | Connects signed cookies, one-time tickets, and bounded live leases. |
| 4 | `27ddc3fca2f1` | `feat(guest): guest 조회 범위와 lobby 격리` | A | AUTH, PERSISTENCE, TOURNAMENT | Avoids fetching registered social and historical data for demo traffic. |
| 5 | `77a7c205ccd0` | `feat(game): guest matchmaking과 room을 격리` | A | REALTIME, PERSISTENCE, RISK | Partitions matchmaking and AI fallback by identity kind. |
| 6 | `eaa4fdaba361` | `feat(game): guest 경기 결과 영속화 차단과 임시 보존` | A | SIMULATION, REALTIME | Skips durable finalization while retaining a short in-memory recovery result. |
| 7 | `2b274686e6d4` | `fix(guest): 체험 환경의 runtime 복구 제한` | A | AUTH, REALTIME, RISK | Bounds IP windows and pending ticket structures and validates runtime mode. |
| 8 | `06d2eb7a93cc` | `test(guest): 체험 환경의 복구 경계 검증` | A | AUTH, SIMULATION, REALTIME | Verifies bounded cleanup, fresh-ticket reconnect, and no duplicate match intent. |

## 5. Commit별 학습 기록

### 5.1. `feat(guest): signed guest session token 정의`

| 항목 | 값 |
| --- | --- |
| SHA | `cacd4c22d705` |
| Importance | A |
| Tags | AUTH, PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Creates tamper-evident, address-bound, expiring guest identity without database sessions.
- Classification summary: Introduce a self-contained signed session representation for transient guests.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | public demo identity를 만들려면 DB account/session을 생성하거나 검증 없는 client-supplied identity를 신뢰해야 하는 상태였습니다. |
| 핵심 boundary/decision | `apps/api/src/guestAccess.ts`가 server-generated guest ID/handle/displayName과 `{v:1,user,ip,expiresAtMs}` payload를 Base64URL로 직렬화하고 HMAC-SHA-256으로 서명합니다. TTL은 2시간이며 secret은 최소 32 bytes입니다. |
| 상태 또는 ownership 변화 | server가 identity fields와 signature를 만들고 HttpOnly guest cookie가 signed payload를 운반합니다. database는 guest identity를 소유하지 않습니다. |
| 주요 failure/edge path | verify는 malformed, signature length/mismatch, wrong version, non-guest/non-user/non-active, nonfinite/expired, request IP mismatch를 거부하며 `timingSafeEqual`을 사용합니다. |
| 보장/비보장 | DB 없이 tamper-evident·address-bound·expiring guest identity를 제공합니다. live connection 수와 ticket issuance capacity는 아직 제한하지 않습니다. |
| 다음 관련 commit 연결 | `17a1dd...`가 guest WebSocket connection capacity를 lease identity와 socket lifetime에 결합합니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `17a1dd501b1b` — `feat(guest): guest resource lease 수명주기 추가`

### 5.2. `feat(guest): guest resource lease 수명주기 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `17a1dd501b1b` |
| Importance | A |
| Tags | AUTH, PROTOCOL, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Binds guest connection limits to lease identity and socket lifetime.
- Classification summary: Add bounded live-connection leases with stale-release protection.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | signed identity만으로는 같은 IP나 process 전체에서 무제한 live guest sockets을 만들어 memory/socket 자원을 소모할 수 있었습니다. |
| 핵심 boundary/decision | GuestAccess가 default IP당 4, process 전체 200 live connection limit을 두고 `connections: Map<guestId,{ip,leaseId}>`에서 lease를 획득합니다. reconnect는 새 leaseId로 current record를 교체합니다. |
| 상태 또는 ownership 변화 | GuestAccess connection map이 live capacity를 소유하고 각 socket admission은 opaque lease handle을 보유합니다. |
| 주요 failure/edge path | release는 current record의 `leaseId`가 일치할 때만 delete해 old socket의 stale close가 replacement lease를 제거하지 못합니다. IP가 바뀌는 replacement도 target IP capacity를 다시 검사합니다. |
| 보장/비보장 | live guest sockets이 per-IP/process 상한을 따르고 replacement stale release가 current lease를 깨지 않습니다. signed cookie와 ticket/route integration은 다음 commit입니다. |
| 다음 관련 commit 연결 | `a5c06c...`이 signed guest cookie, one-time ticket, admission lease를 실제 HTTP/WebSocket boundary에 연결합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `cacd4c22d705` — `feat(guest): signed guest session token 정의`
- 다음 Thread 관련 SHA: `a5c06c561e00` — `feat(guest): guest session과 WebSocket 인증 연결`

### 5.3. `feat(guest): guest session과 WebSocket 인증 연결`

| 항목 | 값 |
| --- | --- |
| SHA | `a5c06c561e00` |
| Importance | A |
| Tags | AUTH, REALTIME, PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Connects signed cookies, one-time tickets, and bounded live leases.
- Classification summary: Integrate demo guest identity with HTTP login/logout, in-memory one-time tickets, and WebSocket admission.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest token과 lease는 독립 primitives였고 route가 DB session fallback을 하거나 WebSocket admission이 registered path와 섞일 수 있었습니다. |
| 핵심 boundary/decision | demo `POST /auth/guest`가 secure HttpOnly SameSite=Lax `pp_guest`를 설정합니다. ws-ticket은 guest면 in-memory ticket을 발급하고 `/ws`는 guest ticket을 먼저 consume해 lease를 획득하며 close에서 정확히 한 번 release합니다. demo mode는 registered DB fallback을 하지 않습니다. |
| 상태 또는 ownership 변화 | GuestAccess가 signed identity·pending ticket·lease를, app route가 cookie/trust handoff를, GameHub가 admitted socket 이후를 소유합니다. |
| 주요 failure/edge path | lease acquisition failure는 socket auth close로 수렴하고 logout은 guest에서 DB session delete를 호출하지 않으며 두 auth cookies를 지웁니다. `requireRegistered`가 durable-only route의 공통 guard가 됩니다. |
| 보장/비보장 | signed cookie → in-memory one-time ticket → bounded live lease trust chain이 동작하고 demo guest는 DB session으로 승격되지 않습니다. data/read/match/persistence isolation은 뒤 commits입니다. |
| 다음 관련 commit 연결 | `27ddc3...`가 HTTP read surface에서 registered repository calls를 호출 전에 차단합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `17a1dd501b1b` — `feat(guest): guest resource lease 수명주기 추가`
- 다음 Thread 관련 SHA: `27ddc3fca2f1` — `feat(guest): guest 조회 범위와 lobby 격리`

### 5.4. `feat(guest): guest 조회 범위와 lobby 격리`

| 항목 | 값 |
| --- | --- |
| SHA | `27ddc3fca2f1` |
| Importance | A |
| Tags | AUTH, PERSISTENCE, TOURNAMENT |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Avoids fetching registered social and historical data for demo traffic.
- Classification summary: Partition public guest reads from registered profile, social, leaderboard, history, and tournament data.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 인증이 guest여도 일반 route가 repository를 조회한 뒤 response에서 일부 field만 숨기면 transient traffic이 registered data access와 DB load를 획득합니다. |
| 핵심 boundary/decision | `/me`는 guest identity를 허용하지만 profile, leaderboard, tournaments, social/history routes는 `requireRegistered`로 repository 호출 전에 거부합니다. demo/guest lobby는 repository match/chat/history를 조회하지 않고 제한된 empty/ephemeral view를 반환합니다. |
| 상태 또는 ownership 변화 | HTTP mode/identity guard가 capability boundary를 소유하고 repository는 registered data만 처리합니다. |
| 주요 failure/edge path | post-filter 방식이 아니라 call-before-data branch이므로 accidental leakage와 guest-triggered durable read workload를 함께 막습니다. |
| 보장/비보장 | guest가 registered social/history/leaderboard/tournament data를 조회하지 않습니다. realtime matchmaking과 AI/persistence isolation은 다음 commits입니다. |
| 다음 관련 commit 연결 | `77a7c2...`가 guest/registered pool과 AI fallback source를 분리합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `a5c06c561e00` — `feat(guest): guest session과 WebSocket 인증 연결`
- 다음 Thread 관련 SHA: `77a7c205ccd0` — `feat(game): guest matchmaking과 room을 격리`

### 5.5. `feat(game): guest matchmaking과 room을 격리`

| 항목 | 값 |
| --- | --- |
| SHA | `77a7c205ccd0` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Partitions matchmaking and AI fallback by identity kind.
- Classification summary: Prevent guest and registered players or persistent NPC sources from entering the same room.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | common queue/AI fallback이 identity kind를 고려하지 않으면 guest가 registered user/NPC와 매칭돼 durable ratings·identity가 room에 섞일 수 있었습니다. |
| 핵심 boundary/decision | matchmaking candidate selection은 guest kind가 일치하지 않으면 skip하고 room에 `guest` flag를 저장합니다. guest fallback은 repository NPC lookup 대신 process-local practice opponent를 사용합니다. |
| 상태 또는 ownership 변화 | Matchmaker/GameHub가 pool kind와 room trust domain을 소유합니다. DB NPC repository는 registered rooms에만 사용됩니다. |
| 주요 failure/edge path | mixed pair나 guest의 durable NPC lookup을 candidate 단계에서 차단해 room 생성 후 filtering에 의존하지 않습니다. |
| 보장/비보장 | guest room participant source와 registered queue가 격리됩니다. terminal result가 DB finalize를 우회하는 것은 다음 commit입니다. |
| 다음 관련 commit 연결 | `eaa4fd...`가 guest finalization을 non-persisted result와 2분 process-local retention으로 분리합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `27ddc3fca2f1` — `feat(guest): guest 조회 범위와 lobby 격리`
- 다음 Thread 관련 SHA: `eaa4fdaba361` — `feat(game): guest 경기 결과 영속화 차단과 임시 보존`

### 5.6. `feat(game): guest 경기 결과 영속화 차단과 임시 보존`

| 항목 | 값 |
| --- | --- |
| SHA | `eaa4fdaba361` |
| Importance | A |
| Tags | SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Skips durable finalization while retaining a short in-memory recovery result.
- Classification summary: Return a non-persisted guest result and retain it briefly for reconnect recovery without touching ratings/history.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | common terminal path가 `repo.finalizeMatch`를 호출하면 guest identity가 match/rating/history/tournament persistence에 들어갑니다. 반대로 즉시 폐기하면 disconnect 직후 결과를 못 받을 수 있습니다. |
| 핵심 boundary/decision | guest room은 repository finalize 전에 branch해 `{matchId:null,persisted:false,ratingDelta:0}`를 만들고 user별 recent result map에 2분 timer와 함께 보존합니다. |
| 상태 또는 ownership 변화 | GameHub process memory가 transient result와 cleanup timer를 소유합니다. 같은 guest의 새 result는 이전 timer를 clear하고 교체합니다. |
| 주요 failure/edge path | timer callback은 stored `expiresAtMs` identity를 다시 확인해 replacement entry를 stale timer가 삭제하지 못합니다. reconnect 시 active/recovered room이 없을 때만 retained result를 전송합니다. |
| 보장/비보장 | guest match는 DB match/rating/history/tournament를 만들지 않으며 2분 안에서 result recovery가 가능합니다. process restart/multi-instance를 넘는 보존은 의도적으로 없습니다. |
| 다음 관련 commit 연결 | `2b2746...`가 ticket/rate IP maps와 runtime mode를 fail-closed bounded state로 강화합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `77a7c205ccd0` — `feat(game): guest matchmaking과 room을 격리`
- 다음 Thread 관련 SHA: `2b274686e6d4` — `fix(guest): 체험 환경의 runtime 복구 제한`

### 5.7. `fix(guest): 체험 환경의 runtime 복구 제한`

| 항목 | 값 |
| --- | --- |
| SHA | `2b274686e6d4` |
| Importance | A |
| Tags | AUTH, REALTIME, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Bounds IP windows and pending ticket structures and validates runtime mode.
- Classification summary: Close unbounded in-memory growth and fail-open runtime-mode behavior in the public guest surface.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | TTL이 있어도 공격자가 계속 새 IP/window/ticket key를 만들면 cleanup 전에 maps가 무한 증가할 수 있고 unknown `APP_MODE`가 development로 해석될 수 있었습니다. |
| 핵심 boundary/decision | tracked IP windows는 10,000개, pending guest tickets는 IP당 4개, issuance는 IP당 분당 30회로 제한합니다. ticket map/reverse index/timer는 공통 delete path로 정리하고 runtime mode는 알려진 enum만 허용합니다. |
| 상태 또는 ownership 변화 | GuestAccess가 rate window·pending ticket·reverse index·timer capacity를 명시적으로 소유하며 app startup mode parser가 deployment capability를 fail-closed합니다. |
| 주요 failure/edge path | capacity 초과는 새 state 할당 전에 거부합니다. consume/expiry/replacement/close가 같은 cleanup 함수를 사용해 timer와 양방향 index를 동시에 제거합니다. unknown mode는 development route를 열지 않고 error입니다. |
| 보장/비보장 | guest public endpoint의 주요 process-local maps가 TTL뿐 아니라 hard cardinality/rate/pending limits를 갖습니다. distributed IP identity와 proxy correctness는 별도 trust 설정에 의존합니다. |
| 다음 관련 commit 연결 | `06d2eb...`가 fake timers와 browser recovery fixture로 bounded cleanup과 fresh-ticket/no-duplicate-intent를 검증합니다. |

#### Fix 재구성

| 단계 | 근거 |
| --- | --- |
| 이전 가정 | 각 entry에 TTL이 있으면 public guest runtime state가 충분히 bounded하다는 가정. |
| 실제 실패 또는 위험 | expiry 속도보다 빠르게 distinct IP/ticket keys를 만들면 map/timer 수가 계속 증가하고 mode typo가 dev capability를 노출합니다. |
| Root cause | 시간 제한은 있었지만 cardinality·issuance rate·per-IP pending 상한과 shared cleanup invariant가 없었습니다. |
| 수정된 invariant/decision | 10,000 tracked IP, 30/min issuance, 4 pending/IP, 공통 delete path, strict runtime mode validation. |
| 변경 코드 | `apps/api/src/guestAccess.ts` capacity/rate/ticket indexes 및 environment mode parser. |
| Regression evidence | `06d2eb...` fake-timer window/ticket cleanup과 limit tests가 보호합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `eaa4fdaba361` — `feat(game): guest 경기 결과 영속화 차단과 임시 보존`
- 다음 Thread 관련 SHA: `06d2eb7a93cc` — `test(guest): 체험 환경의 복구 경계 검증`

### 5.8. `test(guest): 체험 환경의 복구 경계 검증`

| 항목 | 값 |
| --- | --- |
| SHA | `06d2eb7a93cc` |
| Importance | A |
| Tags | AUTH, SIMULATION, REALTIME |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Verifies bounded cleanup, fresh-ticket reconnect, and no duplicate match intent.
- Classification summary: Add deterministic guest runtime and browser recovery regressions for bounded transient state.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | guest domain은 격리·bounded하도록 구현됐지만 rate window, pending tickets, leases, retained result timers와 browser reconnect intent가 함께 회귀하지 않는 evidence가 부족했습니다. |
| 핵심 boundary/decision | fake timers와 guest/app/GameHub/browser fixtures가 issuance rate/pending cap/expiry cleanup, stale lease release, retained result expiry/replacement, fresh-ticket reconnect, reconnect 시 queue intent 미재전송을 검사합니다. |
| 상태 또는 ownership 변화 | test clock이 window/ticket/result timer를 deterministic하게 전진시키고 spies가 repository 호출 부재와 ticket/queue call counts를 관찰합니다. |
| 주요 failure/edge path | limit에 도달한 뒤 expiry/window 이동으로 capacity가 실제 반환되는지, old timer/lease가 new entry를 지우지 않는지, reconnect가 매 attempt fresh ticket을 쓰면서 duplicate match를 만들지 않는지 확인합니다. |
| 보장/비보장 | 검사한 process/browser 경계에서 isolation과 bounded cleanup을 증명합니다. source IP spoofing 방지, proxy configuration, multi-instance/global cap은 증명하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 상태는 database-independent signed identity와 bounded transient capability domain입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | guest identity/tickets/leases/results는 registered data와 분리되고 모든 process-local state는 bounded·cleanup됩니다. |
| 재현하는 failure/boundary | rate window, pending per-IP cap, expiry, replacement stale release/timer, retained result, fresh-ticket reconnect/no queue replay. |
| test technique | Vitest fake timers, app/GameHub fixtures, repository spies, browser socket-client tests. |
| 통과하는 production path | guest cookie → in-memory ticket → lease → guest room → non-persisted result → reconnect/expiry cleanup. |
| 증명하는 것 | 검사한 single-process trust domain과 browser recovery behavior가 의도한 bounds를 지킵니다. |
| 증명하지 않는 것 | multi-instance aggregate capacity, real reverse proxy IP trust, internet adversarial load는 증명하지 않습니다. |
| test 성격 | Deterministic bounded-resource, isolation, and reconnect regression. |
| 후속 회귀 방지 설명 | TTL-only growth, stale release deletion, DB access, durable result, duplicate matchmaking intent가 생기면 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `2b274686e6d4` — `fix(guest): 체험 환경의 runtime 복구 제한`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| Guest identities, matchmaking pools, capabilities, persistence, tickets, leases, and retained results remain isolated and resource-bounded. | `cacd4c22d705` signed identity | `17a1dd501b1b` leases → `a5c06c561e00` trust handoff → `27ddc3fca2f1` reads → `77a7c205ccd0` matchmaking → `eaa4fdaba361` persistence → `2b274686e6d4` hard bounds | TTL-only structures와 unknown mode fail-open | `2b274686e6d4` | `06d2eb7a93cc` | `guestAccess.ts`, app guards, GameHub guest branches |
| The durable browser session is carried only by an HttpOnly cookie; raw WebSocket tickets are short-lived, single-use, hashed at rest, bounded during authentication, and excluded from logs. | Thread 02 contract 재사용 | `a5c06c561e00` guest in-memory ticket/lease, `2b274686e6d4` pending/rate bounds | guest public issuance의 별도 capacity 문제 | `2b274686e6d4` | `06d2eb7a93cc` | guest cookie/ticket/admission paths |
| Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup. | `17a1dd501b1b` lease handle | `eaa4fdaba361` result timer guard, `2b274686e6d4` shared ticket cleanup | stale close/timer가 replacement 삭제 위험 | leaseId/expiresAt identity guards | `06d2eb7a93cc` | GuestAccess maps/timers, recentGuestResults |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| guest를 DB registered session의 약한 변형으로 취급 | signed DB-less identity + no DB fallback + registered-only guards | guest route/repository spy tests | 별도 trust/capability domain |
| guest/registered pool·NPC·result가 섞임 | pool kind/room guest flag + transient finalization branch | GameHub guest tests | mixed matchmaking/durable result 없음 |
| TTL만으로 public in-memory state를 제한 | `2b274...` hard cardinality/rate/pending caps + shared cleanup | `06d2eb...` fake-timer/limit tests | process-local state bounded |
| old socket/timer가 replacement resource를 삭제 | leaseId/expiresAt identity checks | stale release/timer regression | current owner만 cleanup 가능 |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| guest identity | 없음/DB account 가능성 | `cacd4c...` signed payload | GuestAccess + HttpOnly guest cookie | expiry/logout | `guestAccess.ts::issue/verify` |
| pending ticket | registered DB ticket | `a5c06...` in-memory guest ticket | GuestAccess bounded maps | consume/expiry/shared delete | guest ticket methods |
| live capacity | 없음 | `17a1dd...` lease map | GuestAccess leaseId record | current lease release/socket close | connection lease methods |
| read/match capability | common route/queue | `27ddc...` route guards, `77a7c...` pool kind | app guard + Matchmaker/GameHub | request/room lifetime | registered guards/guest flag |
| result | common DB finalize | `eaa4fd...` transient branch | GameHub recentGuestResults | 2분 timer/replacement | guest finalization path |

## 9. Thread 최종 상태

- 최종 authoritative owner: GuestAccess가 signed identity·pending tickets·live leases·rate windows를, app guards가 capabilities를, GameHub가 guest room/transient result를 소유합니다.
- 최종 상태/invariant: guest는 registered data·pool·NPC·ratings·history·tournament와 섞이지 않고 모든 transient state는 TTL과 hard capacity를 함께 가집니다.
- 남아 있는 의도적 제한 또는 비보장: single-process state라 restart/multi-instance 복구·global capacity를 제공하지 않으며 IP binding은 trusted proxy 설정에 의존합니다.
- 후속 Thread가 의존하는 contract: signed `pp_guest` cookie는 in-memory one-time ticket을 발급할 수 있고 admitted lease 범위 안에서 guest-only room과 non-persisted result만 사용합니다.
- 대표 코드 근거: `cacd4c22d705`/`17a1dd501b1b`/`2b274686e6d4 apps/api/src/guestAccess.ts`, `27ddc3fca2f1 app.ts`, `77a7c205ccd0`/`eaa4fdaba361 gameHub.ts`, `06d2eb7a93cc` tests

## 10. 최종 architecture 또는 execution flow 정리

```text
[POST /auth/guest: server-generated identity + HMAC cookie, IP/2h bound]
    ↓
[guest-only in-memory ws ticket: 30/min, 4 pending/IP, tracked-IP cap]
    ↓ one-time consume
[live lease: 4/IP, 200/process, leaseId stale-release guard]
    ↓
[registered data call-before-read guards + guest-only matchmaking/AI]
    ↓
[guest room simulation]
    ↓ terminal
[no DB finalize → persisted:false, matchId:null, ratingDelta:0]
    ↓ 2-minute process-local retained result → cleanup/reconnect
```

- demo mode는 registered session/database fallback을 하지 않습니다.
- hard caps는 TTL cleanup이 따라가지 못하는 공격에서도 state cardinality를 제한합니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 07-guest-mode-as-isolated-transient-trust-domain.md =====

===== BEGIN FILE: 08-runtime-timing-backpressure-drain-and-operational-evidence.md =====
# 런타임 타이밍·백프레셔·드레인·운영 증거

원문 Development Thread: `Runtime timing, backpressure, drain, and operational evidence`

## 1. Thread 목표

- simulation catch-up, socket liveness, input burst, snapshot backlog, timer topology, shutdown work를 각각 명시적으로 제한하는 runtime 경계를 추적합니다.
- per-room timing이 shared scheduler로 이동하며 runnable room membership이 lifecycle 상태와 일치하는지 확인합니다.
- drain/readiness/signal shutdown과 database/edge fault harness가 같은 운영 invariant를 어떻게 검증하는지 복원합니다.

### Source에서 확정된 significance

> The completed runtime bounds every major source of unbounded work: elapsed catch-up, dead sockets, input bursts, snapshot backlog, timer multiplicity, and shutdown. Observability and fault tooling then measure the same boundaries under process and network degradation rather than relying only on unit behavior.

### 직접 연결되는 Critical Invariants

> Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup.

> Draining rejects new work immediately, allows owned rooms to finish within a bounded budget, and remains aligned with the container termination grace period.

> Every accepted wire message conforms to the supported versioned runtime schema; snapshot and input ordering cannot move state backward.

### 직접 연결되는 Major Engineering Difficulties

> Handling slow or stale transports through sequence gates, token buckets, latest-value snapshot delivery, measurable congestion, and hard termination limits.

> Preserving domain correctness during database failure, tournament-start rollback, match-finalization retry, process drain, and deployment shutdown.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- wall-clock 지연이 fixed 50 ms step으로 변환될 때 remainder, five-tick cap, 250 ms ceiling은 어떻게 적용됩니까?
- heartbeat의 ping interval과 authoritative deadline은 어떤 timer ownership과 idempotent cleanup을 사용합니까?
- input sequence와 per-user token bucket은 어떤 순서로 검사되며 왜 stale input이 budget을 쓰지 않습니까?
- latest snapshot buffer는 obsolete frame replacement와 measurable congestion을 어떻게 구분합니까?
- shared scheduler registry는 room register/unregister 중 iteration mutation을 어떻게 격리합니까?
- draining 시작 직후 readiness, waiting clients, active rooms, final close 상태는 어떻게 바뀝니까?
- fault harness가 database path와 edge path를 분리해 어떤 evidence를 수집합니까?

## 3. 완료 기준

- 각 runtime limiter의 key, threshold, timer, cleanup owner를 표로 완성할 수 있습니다.
- room이 simulation 가능한 동안에만 shared scheduler에 등록된다는 lifecycle invariant를 설명할 수 있습니다.
- snapshot delivery가 lossy 최신 상태 전송이고 control event는 다른 경계를 갖는 이유를 설명할 수 있습니다.
- 첫 signal부터 60초 drain, Fastify close, repository release까지 단일 teardown sequence를 그릴 수 있습니다.
- load/fault test가 unit test와 달리 어떤 실제 service-level 경로와 실패를 증명하는지 구분할 수 있습니다.

> 검토 방식: 지정 브랜치에 속한 exact SHA의 diff와 해당 시점 파일을 GitHub에서 확인했습니다. 로컬 실행 환경은 GitHub clone이 차단되어 테스트 명령은 실행하지 않았으며, 아래 테스트 결과 설명은 test implementation 검토에 한정합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source에서 확정된 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3a2943ff385d` | `feat(game): fixed-step scheduler 추가` | A | SIMULATION, REALTIME, OBSERVABILITY | Separates monotonic elapsed time from bounded 50 ms simulation work. |
| 2 | `10a656e59864` | `feat(game): WebSocket heartbeat 추가` | A | REALTIME, RISK | Adds deterministic liveness ownership and timeout cleanup. |
| 3 | `207df3f47935` | `feat(game): 입력 순서와 rate limit 보호` | A | SIMULATION, REALTIME, RISK | Combines monotonic input admission with per-user token-bucket limits. |
| 4 | `8589ff3c4821` | `feat(game): latest snapshot buffer 추가` | A | SIMULATION, REALTIME, OPERATIONS | Defines latest-value delivery and congestion termination rules. |
| 5 | `d21a47ee92d2` | `refactor(game): shared room scheduler 추가` | A | SIMULATION, REALTIME, REFACTOR | Creates one fixed-step clock for all registered rooms. |
| 6 | `fb5b1abc97f5` | `refactor(game): GameHub가 shared room scheduler 사용` | A | SIMULATION, REALTIME, REFACTOR | Makes runnable-room membership the hub’s single timing topology. |
| 7 | `44ef3e07e1a5` | `feat(game): 새 작업 차단과 active room drain 추가` | A | PROTOCOL, REALTIME, TOURNAMENT | Rejects new work while waiting for owned rooms to finish. |
| 8 | `1c9981393973` | `feat(ops): graceful shutdown 절차 추가` | A | REALTIME, PERSISTENCE, OPERATIONS | Makes signals enter one bounded drain-and-close sequence. |
| 9 | `7b0b5f086b41` | `test(load): 실시간 fault injection 도구 추가` | A | AUTH, REALTIME, PERSISTENCE | Provides separate database and edge degradation paths for service-level evidence. |

## 5. Commit별 학습 기록

### 5.1. `feat(game): fixed-step scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `3a2943ff385d` |
| Importance | A |
| Tags | SIMULATION, REALTIME, OBSERVABILITY |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Separates monotonic elapsed time from bounded 50 ms simulation work.
- Classification summary: Introduce a fixed-step accumulator that separates elapsed wall-clock time from simulation updates.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | parent에는 elapsed wall-clock 지연을 simulation step 수로 바꾸면서 catch-up 양을 제한하는 독립 accumulator/scheduler가 없습니다. |
| 핵심 boundary/decision | `apps/api/src/game/fixedStepScheduler.ts`의 `FixedStepAccumulator.advance`가 `elapsed=max(0, now-previous)`를 250 ms까지 누적하고, 50 ms 단위 whole ticks 중 한 loop 최대 5개만 반환합니다. `FixedStepScheduler`는 injectable monotonic clock과 interval을 소유합니다. |
| 상태 또는 ownership 변화 | accumulator가 `previousTimeMs`와 remainder `lagMs`를, scheduler가 interval handle과 현재 accumulator를 단독 소유합니다. `start`/`stop`은 중복 호출에 안전합니다. |
| 주요 failure/edge path | non-finite time은 0 tick, backward clock은 elapsed 0으로 처리하고 previous time을 뒤로 옮기지 않습니다. invalid timestep/cap/ceiling은 constructor에서 거부합니다. callback이 scheduler를 stop하면 loop 조건의 `this.timer`가 같은 batch의 남은 tick을 중단합니다. |
| 보장/비보장 | event-loop stall 뒤에도 한 loop work는 5 ticks, backlog는 250 ms로 제한되고 remainder는 유지됩니다. 아직 room lifecycle이나 GameHub timing owner가 이 abstraction을 사용한다는 보장은 없습니다. |
| 다음 관련 commit 연결 | `10a656...`가 socket liveness timer ownership을 별도 abstraction으로 추가합니다. |

비교 기준:
- 이 commit의 parent에서 동일 책임을 담당하던 코드를 비교했습니다.
- 다음 Thread 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`

### 5.2. `feat(game): WebSocket heartbeat 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `10a656e59864` |
| Importance | A |
| Tags | REALTIME, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Adds deterministic liveness ownership and timeout cleanup.
- Classification summary: Introduce an explicit heartbeat lifecycle for realtime connections.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | socket close event만 기다리면 peer가 사라졌지만 transport가 즉시 닫히지 않는 경우 connection·room resource가 남을 수 있습니다. |
| 핵심 boundary/decision | `apps/api/src/game/heartbeat.ts::ConnectionHeartbeat`는 start 시 45초 timeout을 먼저 arm하고 15초마다 transport ping을 보냅니다. `acknowledge`는 기존 deadline timer를 clear한 뒤 새 45초 deadline으로 교체합니다. |
| 상태 또는 ownership 변화 | heartbeat instance 하나가 `pingTimer`와 `timeoutTimer`를 모두 소유합니다. socket owner는 pong에서 `acknowledge`, close에서 `stop`을 호출하고 heartbeat는 stale/throw에서 target termination을 호출합니다. |
| 주요 failure/edge path | ping synchronous exception과 deadline expiry는 모두 `terminate`로 수렴하며, `terminate`가 먼저 `stop`해 두 handle을 null로 만든 뒤 socket을 종료합니다. start/ack/stop 재호출은 새 competing timer를 만들지 않습니다. |
| 보장/비보장 | 응답 없는 connection은 최대 45초 liveness budget 안에서 정리되고 close race 뒤 timer가 남지 않습니다. 실제 GameHub socket에 연결되는 통합은 이 abstraction만으로 증명하지 않습니다. |
| 다음 관련 commit 연결 | `207df3...`가 stale ordering과 per-user token budget을 하나의 input admission 결과로 만듭니다. |

비교 기준:
- 직전 Thread 관련 SHA: `3a2943ff385d` — `feat(game): fixed-step scheduler 추가`
- 다음 Thread 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`

### 5.3. `feat(game): 입력 순서와 rate limit 보호`

| 항목 | 값 |
| --- | --- |
| SHA | `207df3f47935` |
| Importance | A |
| Tags | SIMULATION, REALTIME, RISK |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Combines monotonic input admission with per-user token-bucket limits.
- Classification summary: Introduce an input gate that combines sequence ordering with per-user token-bucket throttling.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | input sequence만 검사하면 순서는 보호해도 한 사용자가 유효한 새 sequence를 무제한 보내 server work를 만들 수 있고, room별 bucket이면 room 수로 budget을 늘릴 수 있습니다. |
| 핵심 boundary/decision | `apps/api/src/game/inputGate.ts::InputGate.check`는 sequence key를 `userId\u0000roomId`, bucket key를 `userId`로 둡니다. stale/duplicate를 먼저 반환한 뒤 default burst 8, refill 30/s token을 검사하고 accepted일 때만 token과 last sequence를 갱신합니다. |
| 상태 또는 ownership 변화 | InputGate가 모든 user bucket과 user-room last sequence를 소유하며 caller는 `accepted | stale | rate_limited` 결과만 해석합니다. `releaseUser`가 bucket과 해당 user의 모든 room sequence를 제거합니다. |
| 주요 failure/edge path | `inputSeq <= previous`는 bucket refill/charge 전에 탈락하므로 stale replay가 정상 사용자의 rate budget을 소모하지 않습니다. backward/non-advancing clock은 refill하지 않고 constructor는 nonpositive rate/capacity를 거부합니다. |
| 보장/비보장 | 여러 room/connection을 열어도 user 전체 throughput은 같은 bucket에 묶이고 room별 ordering은 독립적으로 단조 증가합니다. transport가 outcome을 어떻게 노출하는지는 caller 통합 범위입니다. |
| 다음 관련 commit 연결 | `8589ff...`가 outbound snapshot backlog를 latest-value 하나와 congestion deadline으로 제한합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `10a656e59864` — `feat(game): WebSocket heartbeat 추가`
- 다음 Thread 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`

### 5.4. `feat(game): latest snapshot buffer 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `8589ff3c4821` |
| Importance | A |
| Tags | SIMULATION, REALTIME, OPERATIONS |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Defines latest-value delivery and congestion termination rules.
- Classification summary: Introduce a latest-value outbound buffer for game snapshots.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 simulation snapshot을 그대로 queue하면 느린 client가 obsolete frames와 application memory를 무제한 누적시킬 수 있습니다. |
| 핵심 boundary/decision | `apps/api/src/game/latestSnapshotBuffer.ts::LatestSnapshotBuffer`는 send in-flight 하나와 `pendingSnapshot` 하나만 유지하며 새 enqueue가 pending old snapshot을 교체합니다. `bufferedAmount` 256 KiB 초과는 50 ms retry, 5초 지속은 terminate, 1 MiB 이상은 즉시 terminate합니다. |
| 상태 또는 ownership 변화 | connection별 buffer가 pending payload, in-flight flag, congestion start, retry timer, closed state를 단독 소유합니다. `close`가 pending과 timer를 제거합니다. |
| 주요 failure/edge path | closed socket, hard threshold, soft congestion timeout, async send error, synchronous send throw가 모두 idempotent `terminate`/`close`로 수렴합니다. retry timer는 하나만 arm되고 종료 뒤 재실행하지 않습니다. |
| 보장/비보장 | application-level snapshot backlog는 latest pending 하나로 제한되고 recovering client는 obsolete queue 대신 최신 상태를 받습니다. 이 lossy 정책은 snapshot에만 적합하며 control/result event 전달 보장은 별도입니다. |
| 다음 관련 commit 연결 | `d21a47...`가 여러 room을 하나의 bounded fixed-step clock으로 구동할 registry를 만듭니다. |

비교 기준:
- 직전 Thread 관련 SHA: `207df3f47935` — `feat(game): 입력 순서와 rate limit 보호`
- 다음 Thread 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`

### 5.5. `refactor(game): shared room scheduler 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `d21a47ee92d2` |
| Importance | A |
| Tags | SIMULATION, REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Creates one fixed-step clock for all registered rooms.
- Classification summary: Introduce a scheduler abstraction capable of driving all active rooms from one fixed-step clock.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | room마다 timer를 소유하면 active match 수만큼 interval이 생기고 register/unregister 중 callback collection mutation이 iteration을 흔들 수 있습니다. |
| 핵심 boundary/decision | `apps/api/src/game/sharedRoomScheduler.ts::SharedRoomScheduler`는 room ID→step callback map과 하나의 `FixedStepScheduler(50 ms, 5 ticks, 250 ms)`를 가집니다. 첫 register가 clock을 시작하고 마지막 unregister가 멈춥니다. |
| 상태 또는 ownership 변화 | shared scheduler가 runnable room registry와 단일 timing loop를 소유합니다. `stop`은 registry와 underlying scheduler를 함께 clear합니다. |
| 주요 failure/edge path | `stepRooms`는 `[...roomSteps.values()]` snapshot을 순회하므로 한 callback이 자신을 unregister하거나 다른 room을 register해도 현재 tick의 iteration이 손상되지 않습니다. 같은 room ID register는 callback을 교체합니다. |
| 보장/비보장 | room 수와 무관하게 하나의 bounded clock으로 registered callbacks를 실행하고 empty registry에는 timer가 없습니다. GameHub lifecycle이 정확히 register/unregister하는지는 다음 commit 범위입니다. |
| 다음 관련 commit 연결 | `fb5b1a...`가 per-room scheduler를 제거하고 GameHub lifecycle을 registry membership에 연결합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `8589ff3c4821` — `feat(game): latest snapshot buffer 추가`
- 다음 Thread 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`

### 5.6. `refactor(game): GameHub가 shared room scheduler 사용`

| 항목 | 값 |
| --- | --- |
| SHA | `fb5b1abc97f5` |
| Importance | A |
| Tags | SIMULATION, REALTIME, REFACTOR |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes runnable-room membership the hub’s single timing topology.
- Classification summary: Move simulation timing ownership from each room to one scheduler owned by `GameHub`.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | 각 `Room`이 `FixedStepScheduler | null`을 가졌으므로 timer ownership이 room object마다 흩어지고 모든 terminal/pause path가 개별 handle을 기억해야 했습니다. |
| 핵심 boundary/decision | `apps/api/src/gameHub.ts`에서 room scheduler field를 제거하고 GameHub에 `roomScheduler = new SharedRoomScheduler()` 하나를 둡니다. play/resume은 register하고 pause, disconnect, abandonment, finalization, finished-room removal은 unregister합니다. |
| 상태 또는 ownership 변화 | GameHub가 application-wide timing topology를 소유하고 room은 simulation state만 보유합니다. `scheduledRoomCount`는 registry size를 관찰 가능한 상태로 노출합니다. |
| 주요 failure/edge path | unregister는 absent ID에도 안전하며 terminal path가 반복돼도 timer가 중복 정리되지 않습니다. disconnect/finish 전에 unregister해 paused/terminal room이 같은 tick에 더 진행하지 못하게 합니다. |
| 보장/비보장 | room은 simulation이 advance 가능한 동안에만 shared registry에 존재하고 active room 수와 무관하게 interval은 하나입니다. process admission/drain은 아직 별도입니다. |
| 다음 관련 commit 연결 | `44ef3e...`가 새 work를 차단하면서 이미 소유한 rooms만 bounded wait하는 drain state를 추가합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `d21a47ee92d2` — `refactor(game): shared room scheduler 추가`
- 다음 Thread 관련 SHA: `44ef3e07e1a5` — `feat(game): 새 작업 차단과 active room drain 추가`

### 5.7. `feat(game): 새 작업 차단과 active room drain 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `44ef3e07e1a5` |
| Importance | A |
| Tags | PROTOCOL, REALTIME, TOURNAMENT |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Rejects new work while waiting for owned rooms to finish.
- Classification summary: Introduce an explicit draining state shared by the Fastify readiness boundary and GameHub.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | shutdown 직전에 readiness가 계속 ready이고 queue/tournament admission이 열려 있으면 active room을 기다리는 동안 새 room이 계속 생겨 drain completion이 수렴하지 않습니다. |
| 핵심 boundary/decision | `app.beginDrain`은 lifecycle을 draining으로 바꾸고 `GameHub.beginDrain`은 `acceptingMatches=false`로 설정합니다. queued/tournament waiters를 `server_draining`으로 해제하고 existing rooms만 유지하며, room set empty 또는 timeout까지 하나의 waiter를 반환합니다. |
| 상태 또는 ownership 변화 | Fastify app가 readiness lifecycle을, GameHub가 admission flag·active room set·single drain promise/timer를 소유합니다. room removal이 `notifyDrainProgress`를 통해 waiter를 완료합니다. |
| 주요 failure/edge path | 재진입은 기존 promise를 반환해 competing deadline을 만들지 않습니다. timeout은 `{drained:false, activeRooms}`를 반환하고 즉시 room을 파괴하지 않습니다. final `close`는 shared scheduler, queue/reconnect/result timers, rooms, clients, heartbeat, snapshot buffer와 sockets를 정리합니다. |
| 보장/비보장 | drain 시작 즉시 readiness는 not_ready이고 새 queue/tournament match는 거부되며 owned rooms는 주어진 budget 안에서 끝날 기회를 가집니다. timeout 뒤 결과 보존 여부는 caller의 close 결정에 달려 있습니다. |
| 다음 관련 commit 연결 | `1c9981...`가 SIGTERM/SIGINT를 60초 drain→app close의 single-entry process sequence로 연결합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `fb5b1abc97f5` — `refactor(game): GameHub가 shared room scheduler 사용`
- 다음 Thread 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`

### 5.8. `feat(ops): graceful shutdown 절차 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `1c9981393973` |
| Importance | A |
| Tags | REALTIME, PERSISTENCE, OPERATIONS |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Makes signals enter one bounded drain-and-close sequence.
- Classification summary: Install a single-entry graceful-shutdown handler for SIGTERM and SIGINT.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | drain API가 있어도 process signal handler가 이를 호출하지 않거나 여러 signal이 parallel close를 시작하면 repository/socket cleanup 순서가 경쟁할 수 있습니다. |
| 핵심 boundary/decision | `apps/api/src/gracefulShutdown.ts::installGracefulShutdown`은 SIGTERM/SIGINT listener와 `started` guard를 설치합니다. 첫 signal이 `app.beginDrain(60_000)` 뒤 `app.close()`를 호출하고 onClose hook이 listeners와 repository를 해제합니다. |
| 상태 또는 ownership 변화 | signal installer가 one-shot guard/listeners를, app shutdown callback이 drain/close 순서를 소유합니다. 반환 disposer는 normal close 시 두 listener를 제거합니다. |
| 주요 failure/edge path | 두 번째 signal은 `started` guard로 무시됩니다. drain/close reject는 error callback이 `process.exitCode=1`을 설정하고 `app.close()`를 다시 best-effort로 시도합니다. |
| 보장/비보장 | process teardown은 첫 signal에서 한 번만 시작되고 60초 room budget 뒤 Fastify/GameHub/repository cleanup으로 수렴합니다. OS/container가 그보다 짧게 강제 종료하면 completion은 보장하지 않습니다. |
| 다음 관련 commit 연결 | `7b0b5f...`가 실제 HTTP/WebSocket/database/edge path에서 같은 limits를 측정하는 load/fault harness를 추가합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `44ef3e07e1a5` — `feat(game): 새 작업 차단과 active room drain 추가`
- 다음 Thread 관련 SHA: `7b0b5f086b41` — `test(load): 실시간 fault injection 도구 추가`

### 5.9. `test(load): 실시간 fault injection 도구 추가`

| 항목 | 값 |
| --- | --- |
| SHA | `7b0b5f086b41` |
| Importance | A |
| Tags | AUTH, REALTIME, PERSISTENCE |
| 학습 깊이 | 주요 subsystem·boundary·failure path의 코드와 설계 판단을 복원했습니다. |

#### Source에서 확정된 역할과 범위

- Thread 역할: Provides separate database and edge degradation paths for service-level evidence.
- Classification summary: Add a k6 realtime-load harness and independent Toxiproxy controls for persistence and transport faults.

#### 해당 SHA에서 확인한 실제 코드

| 기록 항목 | 해당 SHA의 근거 |
| --- | --- |
| 직전 관련 상태 | unit/fake-timer tests는 local state machine과 cleanup을 결정적으로 검증하지만 실제 process/network 경로의 latency, reconnect, snapshot gap, duplicate finalization을 함께 측정하지 못합니다. |
| 핵심 boundary/decision | `tests/load` k6 harness는 기본 500 connections/50 rooms(extended 1,000), dev-login→ticket→WebSocket→queue/input→fresh-ticket reconnect를 실행합니다. Compose overlay는 PostgreSQL과 public edge를 별도 Toxiproxy endpoint로 라우팅하고 control script가 latency/reset/down/up을 독립 적용합니다. |
| 상태 또는 ownership 변화 | k6 VU가 connection/room lifecycle과 sequence/result sets를, custom metrics가 connection/reconnect success, snapshot delay/gap, online/room count, finalization uniqueness를 소유합니다. Toxiproxy control utility가 named proxy/toxic lifecycle을 소유합니다. |
| 주요 failure/edge path | profile은 connections≥2×rooms를 검증하고 99% connection/reconnect, snapshot p95≤150 ms/p99≤250 ms, normal drop<1%, finalize failures/duplicates=0 등의 thresholds를 둡니다. DB fault와 edge fault를 분리해 persistence degradation과 transport degradation을 혼동하지 않습니다. |
| 보장/비보장 | 실제 service-level auth/ticket/socket/reconnect/snapshot/finalize 경로를 장애 아래 측정할 수 있는 executable evidence를 제공합니다. 이 환경에서는 harness를 실행하지 않았으므로 수치 통과를 주장하지 않으며 장기 production capacity를 일반화하지 않습니다. |
| 다음 관련 commit 연결 | Thread 최종 상태는 모든 주요 runtime work source에 bound·owner·cleanup·operational measurement가 있는 구조입니다. |

#### Test commit 학습 기록

| 구분 | 기록 |
| --- | --- |
| 대상 production invariant | database와 edge degradation을 분리해도 server-authoritative admission, reconnect, snapshot delivery와 one-result finalization을 관찰할 수 있습니다. |
| 재현하는 failure/boundary | connection population, room creation, forced reconnect, snapshot latency/gaps, DB latency/outage, edge latency/reset/outage, duplicate finalization. |
| test technique | k6 multi-connection load, Docker Compose overlay, independent Toxiproxy controls, custom thresholds/metrics. |
| 통과하는 production path | dev-login/cookie → one-time ticket → WebSocket queue/ready/input → snapshot → forced fresh-ticket reconnect → durable game.finished. |
| 증명하는 것 | 실행 시 actual process/network path에서 fault source별 service metrics와 threshold pass/fail을 수집합니다. |
| 증명하지 않는 것 | unit-level physics correctness, 모든 production topology, 장기 soak/capacity, 그리고 이 작업 환경에서 실제 threshold 통과를 증명하지 않습니다. |
| test 성격 | Broad service-level load and fault-injection evidence. |
| 후속 회귀 방지 설명 | ticket/reconnect protocol, snapshot monotonicity, liveness, finalization uniqueness, DB/edge fault controls가 깨지면 명시된 metric/threshold가 실패해야 합니다. |

비교 기준:
- 직전 Thread 관련 SHA: `1c9981393973` — `feat(ops): graceful shutdown 절차 추가`
- 이 Thread의 마지막 상태와 비교해 최종 보장을 정리했습니다.

## 6. Invariant ledger

Source에서 확정된 invariant를 commit 시점별로 연결했습니다. `해당 없음`은 해당 Thread 안에서 별도 fix/test가 없음을 뜻합니다.

| Invariant | 처음 도입/관찰한 SHA | 강화한 SHA | 부족함이 드러난 SHA | 복구한 fix | 고정한 regression test | 코드 근거 |
| --- | --- | --- | --- | --- | --- | --- |
| Timers, schedulers, heartbeat handles, retry work, snapshot buffers, and database resources have explicit single-owner cleanup. | `3a2943ff385d` scheduler handles | `10a656e59864` heartbeat → `8589ff3c4821` snapshot retry → `d21a47ee92d2` shared clock → `fb5b1abc97f5` lifecycle membership → `44ef3e07e1a5` final close | per-room timer multiplicity와 shutdown competing ownership | `fb5b1abc97f5`/`44ef3e07e1a5` | 각 abstraction unit tests와 `7b0b5f086b41` operational harness | fixedStepScheduler.ts, heartbeat.ts, latestSnapshotBuffer.ts, sharedRoomScheduler.ts, GameHub.close |
| Draining rejects new work immediately, allows owned rooms to finish within a bounded budget, and remains aligned with the container termination grace period. | `44ef3e07e1a5` | `1c9981393973` 60초 single signal sequence | signal 경쟁과 readiness/admission 불일치 | `1c9981393973` | graceful shutdown tests와 load/fault 운영 evidence | app.beginDrain, GameHub.beginDrain, installGracefulShutdown |
| Every accepted wire message conforms to the supported versioned runtime schema; snapshot and input ordering cannot move state backward. | Thread 03 versioned codec/sequence | `207df3f47935` stale-before-token admission, `8589ff3c4821` latest-value delivery | 유효한 새 sequence burst와 slow consumer backlog | input rate gate/latest buffer | InputGate/LatestSnapshotBuffer tests + `7b0b5f086b41` metrics | inputGate.ts, latestSnapshotBuffer.ts, k6 sequence metrics |

## 7. Failure → Fix → Test 연결

| 기존 상태/가정 | Fix 또는 강화 과정 | Test/evidence | 최종 보장 |
| --- | --- | --- | --- |
| long event-loop stall이 무제한 catch-up을 유발 | `3a2943ff385d` 50 ms step, 5-tick loop cap, 250 ms lag ceiling | deterministic accumulator/scheduler tests | bounded simulation work와 remainder |
| dead·abusive·slow client가 timer/input/snapshot work를 누적 | `10a656e59864` heartbeat → `207df3f47935` input gate → `8589ff3c4821` latest buffer | 각 abstraction tests와 k6 connection/snapshot metrics | bounded connection work와 hard termination |
| room마다 timer를 갖고 lifecycle path마다 개별 cleanup | `d21a47ee92d2` shared scheduler → `fb5b1abc97f5` complete register/unregister integration | scheduler/GameHub lifecycle tests | runnable-room registry가 timing topology의 단일 owner |
| shutdown 중 readiness와 admission이 열리고 여러 signal이 경쟁 | `44ef3e07e1a5` drain → `1c9981393973` one-shot signal sequence | drain/shutdown tests와 fault harness | bounded operational teardown |
| DB와 edge degradation을 구분하지 못하는 측정 | `7b0b5f086b41` separate proxies와 source-specific controls | k6 metrics/thresholds | failure source별 service-level evidence |

## 8. Ownership / state / responsibility 변화

| 축 | 초기 SHA의 owner/state | 중간 전환 | Thread 최종 owner/state | 해제·cleanup 책임 | 근거 |
| --- | --- | --- | --- | --- | --- |
| elapsed-time accumulation and simulation steps | `3a2943...` accumulator/scheduler | `d21a47...` room registry | `fb5b1...` GameHub-owned shared scheduler | last unregister/GameHub.close | fixedStepScheduler.ts, sharedRoomScheduler.ts, gameHub.ts |
| heartbeat timers | 없음 | `10a656...` connection heartbeat | connection별 heartbeat instance | stop/terminate/socket close | heartbeat.ts |
| input budget and sequence state | sequence-only or scattered checks | `207df3...` user bucket + user-room sequence | InputGate | releaseUser | inputGate.ts |
| snapshot queue and congestion retry | transport send backlog | `8589ff...` latest pending + one retry | LatestSnapshotBuffer | close/terminate | latestSnapshotBuffer.ts |
| shared room scheduler membership | room별 scheduler | `d21a47...` abstraction | `fb5b1...` GameHub registry | pause/disconnect/abandon/finalize/remove/close | SharedRoomScheduler/GameHub |
| drain waiter and process signal teardown | 없음/즉시 close | `44ef3...` single drain waiter | `1c998...` one-shot signal→60초 drain→close | finishDrain/dispose listeners/app.close | GameHub.beginDrain, gracefulShutdown.ts |
| fault experiment | unit evidence만 존재 | `7b0b5...` DB/edge proxies | k6 profile + toxiproxy control | reset/up/down commands/compose teardown | tests/load, compose overlay |

## 9. Thread 최종 상태

- 최종 authoritative owner: GameHub가 shared room scheduler와 drain lifecycle을, connection abstractions가 heartbeat/input/snapshot resources를, process entrypoint가 signal→close sequence를 소유합니다.
- 최종 상태/invariant: elapsed catch-up, liveness, input burst, snapshot backlog, timer multiplicity, admission과 shutdown이 모두 hard bound와 단일 cleanup owner를 가집니다.
- 남아 있는 의도적 제한 또는 비보장: snapshot 전송은 의도적으로 lossy latest-value이며 drain timeout 뒤 unfinished room 보존을 보장하지 않습니다. load/fault harness의 threshold는 실행 환경과 topology에 종속됩니다.
- 후속 Thread가 의존하는 contract: runnable room만 shared scheduler에 등록되고 draining은 새 work를 즉시 거부한 뒤 60초 budget에서 existing room completion을 기다리고 전체 runtime resource를 한 번 정리합니다.
- 대표 코드 근거: `3a2943ff385d fixedStepScheduler.ts`, `10a656e59864 heartbeat.ts`, `207df3f47935 inputGate.ts`, `8589ff3c4821 latestSnapshotBuffer.ts`, `fb5b1abc97f5`/`44ef3e07e1a5 gameHub.ts`, `1c9981393973 gracefulShutdown.ts`, `7b0b5f086b41 tests/load`

## 10. 최종 architecture 또는 execution flow 정리

```text
[wall-clock / client input / socket state]
    ↓ monotonic accumulator + version/sequence/token admission
[shared 50 ms scheduler: ≤5 ticks/loop, ≤250 ms lag]
    ↓ runnable room callbacks
[authoritative simulation snapshot]
    ↓ latest-value buffer: one pending, 256 KiB soft, 1 MiB hard, 5 s deadline
[WebSocket client + heartbeat: 15 s ping / 45 s deadline]

[SIGTERM/SIGINT]
    ↓ one-shot
[readiness not_ready + reject queue/tournament + release waiters]
    ↓ wait existing rooms ≤60 s
[GameHub.close → timers/scheduler/buffers/heartbeats/sockets]
    ↓
[Fastify close → repository close → signal listener dispose]

[k6 workload] → [separate DB/edge Toxiproxy faults] → [latency/gap/reconnect/finalize metrics]
```

- stale input은 token budget을 쓰기 전에 제거되고 accepted input만 simulation intent를 바꿉니다.
- snapshot은 최신 상태가 가치이므로 obsolete pending frame을 교체하지만 `game.finished` 같은 control event의 delivery 정책과 동일하지 않습니다.
- 이 작업에서는 load/fault command를 실행하지 않았으며 harness와 threshold의 구현만 검토했습니다.

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA를 원문 순서대로 확인했습니다.
- [x] 각 commit의 subject, importance, tags를 변경하지 않았습니다.
- [x] S/A/B 깊이를 구분해 코드 근거를 남겼습니다.
- [x] final HEAD의 구현을 과거 SHA에 소급하지 않았습니다.
- [x] 핵심 상태 필드, caller/callee, ownership, failure branch, cleanup을 실제 코드로 확인했습니다.
- [x] Fix를 기존 가정 → failure/risk → root cause → decision → code → regression 순서로 연결했습니다.
- [x] Test commit에서 production invariant, failure, technique, path, 증명/비증명 범위를 구분했습니다.
- [x] Thread 최종 execution flow를 별도 프로젝트 재학습 없이 설명할 수 있습니다.
===== END FILE: 08-runtime-timing-backpressure-drain-and-operational-evidence.md =====

===== BEGIN FILE: README.md =====
# ft_transcendence Development Thread 학습 골격

## 목적

이 문서 세트는 완성형 프로젝트 해설서가 아닙니다. 학습자가 실제 commit history와 해당 SHA의 코드를 읽고
설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 직접 복원하기 위한 기록 골격입니다.

문서에 미리 적힌 commit 순서, SHA, subject, importance, tags, 역할, significance는 source에서 확정된 내용입니다.
실제 함수 동작, 변경 전후 코드, ownership/lifetime, failure path, test 결과, 최종 설명은 학습자가 채웁니다.

## 권장 학습 순서

1. [서버 권위형 결정적 게임 메커니즘](01-authoritative-deterministic-game-mechanics.md)
2. [쿠키 신원과 일회용 WebSocket 입장](02-cookie-identity-websocket-admission.md)
3. [버전 기반 실시간 프로토콜과 단조 상태](03-versioned-realtime-protocol-and-monotonic-state.md)
4. [원자적·멱등적 경기 결과 확정](04-atomic-idempotent-match-finalization.md)
5. [경기방 수명주기·연결 교체·복구](05-room-lifecycle-connection-replacement-and-recovery.md)
6. [매치메이킹 예약 소유권과 롤백](06-matchmaking-reservation-ownership-and-rollback.md)
7. [격리된 임시 신뢰 도메인으로서의 게스트 모드](07-guest-mode-as-isolated-transient-trust-domain.md)
8. [런타임 타이밍·백프레셔·드레인·운영 증거](08-runtime-timing-backpressure-drain-and-operational-evidence.md)

앞 문서의 결과가 뒤 문서의 필수 선행조건인 것은 아닙니다. 다만 위 순서는 source의 Development Thread 배열을
그대로 따르므로 전체 복습 시 이 순서를 권장합니다.

## Thread 문서 사용법

- 먼저 Thread 목표, 핵심 질문, 완료 기준을 읽습니다.
- Commit map의 순서를 바꾸지 않고 각 SHA를 차례대로 checkout합니다.
- 각 commit의 “Source에서 확정된 역할과 범위”를 기준으로 확인 범위를 제한합니다.
- “해당 SHA에서 확인할 실제 코드”의 항목마다 파일, symbol, caller/callee, 상태 전이, failure branch를 기록합니다.
- Invariant ledger와 Failure → Fix → Test 표는 commit별 기록을 마친 뒤 채웁니다.
- 마지막으로 Thread 최종 상태와 execution flow를 자기 언어로 작성합니다.

## 해당 SHA 코드 확인 원칙

- `git checkout <SHA>` 또는 `git show <SHA>:<path>`로 그 시점의 코드를 확인합니다.
- 변경 자체는 `git show <SHA>`로 보고, Thread의 직전 관련 commit과는 `git diff <OLD>..<NEW> -- <path>`로 비교합니다.
- 파일명이 source에 확정되어 있지 않으면 symbol을 검색해 실제 경로를 기록합니다.
- 함수 하나만 보지 말고 caller, callee, 상태 필드, resource 생성/해제, error branch, 관련 test를 함께 추적합니다.
- source가 명시하지 않은 결론은 확정 사실로 적지 않고 “코드에서 관찰한 해석”으로 표시합니다.

## final HEAD 소급 사용 금지

- final HEAD의 코드로 과거 commit의 동작을 설명하지 않습니다.
- 같은 symbol이 나중에 이동·분리·삭제되었더라도 해당 SHA의 실제 정의와 caller를 사용합니다.
- 필요한 경우 Thread의 직전 관련 SHA와 비교하되, 그 사이의 다른 commit에서 바뀐 내용을 자동으로 귀속하지 않습니다.
- 최종 architecture는 모든 commit 기록을 끝낸 뒤 Thread 마지막 SHA까지의 변화로만 정리합니다.

## S/A/B/C별 학습 깊이

- S: 프로젝트 핵심 architecture/invariant입니다. 문제, 직전 상태, 실패 가능성, 결정, 핵심 코드, ownership/lifecycle/state transition, 후속 fix/test까지 깊게 추적합니다.
- A: 주요 subsystem, trust boundary, failure path, integration point입니다. 핵심 코드와 설계 판단, 주요 edge case를 확인합니다.
- B: Thread 흐름에서 맡는 구현 역할과 필요한 상태 변화를 확인합니다. S/A와 같은 분량을 기계적으로 반복하지 않습니다.
- C: Thread 이해에 필요한 맥락만 기록합니다. source의 Thread map에 포함되지 않은 C commit을 임의로 끼워 넣지 않습니다.

## 실제 코드 삽입 기준

- 해당 SHA의 판단을 증명하는 최소 코드만 삽입합니다.
- 코드 앞에 SHA, 파일 경로, symbol, 확인 목적을 적습니다.
- 상태 mutation 전후 순서, ownership 이전, error/cleanup branch처럼 문장만으로 모호한 부분을 우선합니다.
- 대규모 파일이나 전체 diff를 복사하지 않습니다.
- 변경 전/후 비교가 필요하면 두 SHA의 대응 코드 조각을 나란히 두고 차이를 학습자가 설명합니다.

## Test commit 학습 방법

- 대상 production invariant를 먼저 적습니다.
- 재현하는 failure 또는 boundary를 실제 fixture와 주입 지점으로 확인합니다.
- test technique이 unit, deterministic regression, PostgreSQL integration, browser/process, load/fault 중 무엇인지 구분합니다.
- test가 통과하는 production 코드 경로를 caller 순서로 연결합니다.
- 증명하는 것과 증명하지 않는 것을 모두 기록합니다.
- 후속 변경에서 어떤 회귀를 막는지 설명합니다.

## 문서 완료 기준

- 모든 Thread 문서의 Commit map을 source 순서 그대로 확인했습니다.
- 모든 중요 commit에서 해당 SHA의 구체적인 코드 근거를 기록했습니다.
- S/A/B/C별 학습 깊이가 구분되어 있습니다.
- Invariant ledger와 Failure → Fix → Test 연결이 실제 commit code와 test에 근거합니다.
- ownership, state, responsibility, cleanup 변화를 Thread 단위로 설명할 수 있습니다.
- final HEAD를 소급하지 않고 각 시점의 보장과 비보장을 구분했습니다.
- 완성된 문서만으로 별도의 프로젝트 재학습 없이 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 설명할 수 있습니다.
===== END FILE: README.md =====

