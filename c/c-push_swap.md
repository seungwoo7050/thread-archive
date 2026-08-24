# 병렬 스택 상태와 연산 불변 조건

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** The thread progresses from representation to shared transition architecture, then to complete command semantics and two complementary levels of evidence. The decisive judgment is not any individual `memmove`; it is that original values and dense ranks remain one logical element across both stacks, allowing the sorter to reason about ranks while the checker and tests retain the original-value association.
- **학습 목표:** 병렬 `values`/`ranks` 표현이 어떻게 하나의 논리 원소를 구성하고, 모든 명령이 그 결합과 전체 원소 보존을 유지하는지 실제 commit 코드로 복원합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- `t_stack`의 live range, `size`, `capacity`, 두 버퍼의 소유권은 어떤 계약으로 묶여 있는가?
- silent primitive와 emit 가능한 wrapper의 경계가 generator와 checker의 공통 의미를 어떻게 만든는가?
- `swap`, `push`, `rotate`, `reverse rotate`가 두 배열을 항상 같은 논리 이동으로 처리하는가?
- 불충분한 stack에서 no-op이 되는 명령과 exact transition은 테스트에서 어떻게 분리 검증됩니까?
- 보존 불변식 테스트와 exact-state 테스트가 각각 무엇을 증명하고 무엇을 놓치는가?

## 3. 완료 기준
- 해당 SHA의 `t_stack` 필드와 초기화/해제 경로를 코드 인용으로 설명할 수 있습니다.
- 11개 명령을 pair preservation, size 변화, capacity 유지 관점에서 추적할 수 있습니다.
- emit/no-emit 경계의 caller/callee와 generator/checker 재사용 이유를 실제 호출 경로로 설명할 수 있습니다.
- 보존 테스트와 exact-state/no-op 테스트의 증명 범위를 구분할 수 있습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `96b5324448e4` | feat(model): 배열 기반 스택 상태를 구현 | S | ARCH, CORE, STACK_STATE | Establishes the parallel value/rank stack representation, ownership model, and completion predicate. |
| 2 | `c0de1a1b18bb` | feat(operation): 스택 교환 연산을 구현 | A | CORE, STACK_STATE, INTEGRATION | Introduces pair-preserving state transitions and the emit/no-emit wrapper boundary. |
| 3 | `73d2deb30224` | feat(operation): 스택 간 이동 연산을 구현 | B | CORE, STACK_STATE | Adds cross-stack push while conserving logical elements. |
| 4 | `745ec72850d2` | feat(operation): 스택 정방향 회전을 구현 | B | CORE, STACK_STATE | Adds forward rotation within the same representation. |
| 5 | `68dfd1b1fb58` | feat(operation): 스택 역방향 회전을 구현 | B | CORE, STACK_STATE | Completes the instruction vocabulary with reverse rotation. |
| 6 | `86364d27baac` | test(operation): 값과 순위의 보존 불변식을 검증 | A | TEST, STACK_STATE, RISK | Verifies pair association, element conservation, size bounds, and sequences across all commands. |
| 7 | `7eb6890c2c13` | test(operation): 정확한 상태 전이와 no-op을 검증 | B | TEST, STACK_STATE, EDGE | Locks down exact states and insufficient-stack no-op behavior. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Critical 불변 조건**
  - `values[i]` and `ranks[i]` always describe the same logical element; operations must move them together.
  - Across A and B, every input pair is present exactly once, total active size is conserved, and each stack remains within capacity.
- **Major engineering difficulties**
  - Maintaining value-rank pairing and total-element conservation while array-backed push and rotation operations perform overlapping `memmove` operations.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `96b5324448e4` — feat(model): 배열 기반 스택 상태를 구현
- **중요도:** S
- **태그:** ARCH, CORE, STACK_STATE
- **원자료에서 확인된 역할:** Establishes the parallel value/rank stack representation, 소유권 model, and completion predicate.
- **커밋 분류 요약:** Defines parallel value/rank arrays, stack 소유권, capacity, and sorted-state predicates.

#### 원자료에서 확인된 맥락
- **문제:** The project needs one representation that can serve parsing, command generation, silent checker replay, sortedness checks, and resource cleanup. Sorting can use relative order, but the implementation must not lose the original integer associated with that order.
- **결정:** Represent each stack as parallel `values` and `ranks` arrays with explicit `size` and `capacity`, and make stack initialization, cleanup, sortedness, and complete-state checks part of the model boundary.
- **중요한 이유:** Every later operation moves these arrays together, every sorter reads ranks, both executables own stacks through the same lifecycle, and the 불변 조건 tests are written around this representation. The decision determines both correctness obligations and the later physical cost of array-backed push and rotation.
- **변경 내용:** The commit adds `t_stack`, empty and allocated initialization, paired cleanup, rank-based sortedness, the A-sorted/B-empty completion predicate, and the first strict C99 build structure.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 96b5324448e4`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `96b5324448e4` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `t_stack`의 `values`, `ranks`, `size`, `capacity` 필드와 active prefix가 실제로 어떻게 해석되는지 확인합니다.
- `stack_init`, `stack_init_empty`, `stack_free`에서 두 allocation의 성공/실패 순서와 cleanup/reset 순서를 추적합니다.
- 정렬 판정과 complete-state 판정 함수가 `ranks` 및 B-empty 조건을 어떻게 사용하고 있는지 확인합니다.
- non-positive capacity 경로가 empty lifecycle과 어떻게 합쳐지는지 확인합니다.
- 해당 SHA의 Makefile에서 strict C99/warning scaffold와 model source가 정상 build에 포함되는 위치를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — S
- **이 commit 직전 상태:** 이 SHA의 parent에는 `t_stack`, 공통 헤더, stack model source, strict C99 Makefile이 없었습니다. 따라서 입력을 담을 소유 객체도, 정렬 완료를 판정할 공통 함수도 존재하지 않았습니다.
- **해결하려던 문제:** 원본 정수와 정렬 알고리즘이 사용할 상대 순위를 같은 논리 원소로 유지하면서 A/B 두 스택을 동일 API로 생성·해제·검사할 표현이 필요했습니다.
- **기존 설계가 충분하지 않았던 이유:** 단일 값 배열만 두면 이후 radix가 값 자체의 부호와 폭을 직접 처리해야 하고, 별도 rank 배열을 두더라도 이동 규칙이 정의되지 않으면 원본 값과 순위의 대응이 쉽게 깨집니다. 또한 `size`와 `capacity`가 없으면 active prefix와 할당 범위를 구분할 수 없습니다.
- **핵심 결정:** `96b5324448e4:include/push_swap.h:t_stack`에서 두 `int *` 버퍼와 `size`/`capacity`를 하나의 구조체에 묶고, `96b5324448e4:src/stack.c:stack_init_empty`를 모든 생명주기의 기준 상태로 사용했습니다.

```c
/* 96b5324448e4:include/push_swap.h:t_stack */
typedef struct s_stack
{
    int *values;
    int *ranks;
    int size;
    int capacity;
}   t_stack;
```

- **state / 불변 조건 / 소유권 / lifecycle 변화:** `stack_init_empty`는 두 포인터를 `NULL`, 두 정수를 0으로 만듭니다. `stack_init`은 먼저 empty 상태를 만든 뒤 capacity가 양수일 때 두 배열을 할당하고, 어느 한쪽이라도 실패하면 `stack_free`로 이미 얻은 버퍼까지 해제해 다시 empty 상태로 돌아갑니다. 성공하면 호출자가 두 버퍼를 소유하며 active range는 `[0, size)`이고 allocated range는 `[0, capacity)`입니다. `stack_free`는 두 버퍼를 모두 해제한 뒤 같은 empty 상태를 복원합니다.
- **failure scenario:** 두 번째 배열 할당 실패 때 첫 번째 배열만 남기거나, 해제 후 필드를 초기화하지 않으면 이후 오류 경로에서 누수·중복 해제·stale capacity가 발생할 수 있습니다. 한편 `values`와 `ranks`를 다른 길이로 다루면 이후 operation이 같은 인덱스를 논리 원소로 취급할 수 없습니다.
- **이 commit이 보장하는 것:** capacity가 양수인 정상 초기화에서는 동일 capacity의 두 버퍼를 한 `t_stack`이 소유하고, 실패 또는 해제 뒤에는 empty 상태가 됩니다. `stack_is_sorted`는 active `ranks`만 비내림차순인지 검사하며, `stack_is_complete_sorted`는 A 정렬과 B empty를 동시에 요구합니다. non-positive capacity는 할당 없는 valid empty stack으로 처리됩니다.
- **아직 보장하지 않는 것:** parser가 실제 pair를 채우는 방법, 두 배열을 함께 이동하는 명령, capacity를 넘지 않는다는 동작 증거, allocation byte overflow 방어는 아직 없습니다. 후자는 `049ecd429548`에서 보강됩니다.
- **후속 fix/test:** `c0de1a1b18bb`부터 모든 primitive가 두 배열을 함께 이동합니다. `86364d27baac`은 pair/원소/범위 보존을, `7eb6890c2c13`은 exact state와 no-op을 검증합니다. allocation 실패 cleanup은 Thread 6의 `63969f770a21`에서 fault sweep으로 별도 검증됩니다.
- **개발 흐름의 다음 관련 커밋:** `c0de1a1b18bb`는 이 model의 pair invariant를 실제 swap transition과 emit/no-emit API에서 어떻게 보존하는가?

### `c0de1a1b18bb` — feat(operation): 스택 교환 연산을 구현
- **중요도:** A
- **태그:** CORE, STACK_STATE, INTEGRATION
- **원자료에서 확인된 역할:** Introduces pair-preserving 상태 전이 and the emit/no-emit wrapper boundary.
- **커밋 분류 요약:** Implements pair-preserving swap and introduces optionally emitting operation wrappers.

#### 원자료에서 확인된 맥락
- **문제:** The generator must mutate stacks and emit commands, while the checker must replay the same command semantics without emitting them. Separate implementations would create an avoidable semantic divergence risk.
- **결정:** Split the silent stack transition from an operation wrapper controlled by an `emit` flag, beginning with `sa`, `sb`, and `ss`, while always moving value and rank entries together.
- **중요한 이유:** The pattern becomes the integration boundary for every later command. `push_swap` uses the wrappers as 상태 전이 plus serialization, and checker uses the same wrappers as silent replay. Combined commands also remain one public instruction even when they apply two internal transitions.
- **변경 내용:** The commit adds `stack_swap`, optional command emission, and the single- and dual-stack swap wrappers, then registers the operation module as common code.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only c0de1a1b18bb`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `c0de1a1b18bb` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `stack_swap`이 top의 `values`와 `ranks`를 같은 인덱스 단위로 교환하는지 확인합니다.
- `sa`, `sb`, `ss` wrapper의 caller/callee와 `emit` 분기를 추적합니다.
- `ss`가 두 stack mutation을 수행하되 public command는 한 번만 emit하는 위치를 확인합니다.
- size < 2에서 primitive와 wrapper가 어떤 상태/출력 결과를 만드는지 확인합니다.
- 직전 관련 SHA `96b5324448e4`의 model contract가 operation API에서 어떤 전제로 사용되는지 비교합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** `96b5324448e4`에는 병렬 배열 model만 있고 명령 의미가 없었습니다. generator와 checker가 앞으로 각각 swap을 구현하면 동일 command가 서로 다른 상태 전이를 만들 위험이 있었습니다.
- **주요 boundary/decision:** `c0de1a1b18bb:src/operations.c:stack_swap`은 출력 없는 state primitive이고, `op_sa`/`op_sb`/`op_ss`는 `emit`에 따라 command text를 추가하는 public wrapper입니다. `op_ss`는 A와 B에 primitive를 각각 적용한 뒤 `ss\n`을 한 번만 출력합니다.
- **state / 소유권 / failure 변화:** `stack_swap`은 size가 2 미만이면 즉시 반환하며 소유권·size·capacity를 바꾸지 않습니다. 그 외에는 top 두 `values`와 top 두 `ranks`를 같은 순서로 교환합니다. 이 SHA의 wrapper와 출력 helper는 `void`라 출력 실패를 표현하지 못하며, no-op 상태에서도 `emit=1`이면 명령 문자열은 출력됩니다.
- **보장 / 비보장:** swap 계열은 active top pair의 정확한 교환과 combined command의 단일 emission을 보장합니다. 아직 push/rotate/reverse-rotate가 없고, exact postcondition이나 출력 실패는 검증되지 않았습니다.
- **후속 검증 또는 수정 연결:** `73d2deb30224`~`68dfd1b1fb58`가 같은 wrapper 패턴을 전체 명령으로 확장합니다. `86364d27baac`과 `7eb6890c2c13`이 state semantics를 검증하고, 출력 실패 반환은 `315f4b91779b`에서 추가됩니다.
- **개발 흐름의 다음 관련 커밋:** `73d2deb30224`의 cross-stack 이동은 두 배열과 두 `size`를 어떤 순서로 갱신해 전체 원소 수를 보존하는가?

### `73d2deb30224` — feat(operation): 스택 간 이동 연산을 구현
- **중요도:** B
- **태그:** CORE, STACK_STATE
- **원자료에서 확인된 역할:** Adds cross-stack push while conserving logical elements.
- **커밋 분류 요약:** Implements `pa` and `pb` by moving the top pair between fixed-capacity array stacks.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 73d2deb30224`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `73d2deb30224` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `stack_push`의 source top 저장, destination/source `memmove`, pair copy, 두 `size` update 순서를 확인합니다.
- destination/source의 `values`와 `ranks`가 같은 논리 이동을 하는지 인덱스별로 기록합니다.
- empty source no-op branch와 `pa`/`pb` wrapper의 `emit` 처리 방식을 확인합니다.
- operation 중 새 allocation이 없는지, 기존 stack buffers만 재사용하는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** swap 계열만 존재하고 A/B 사이에 원소를 옮길 방법은 없었습니다.
- **이 commit의 구현 역할:** `73d2deb30224:src/operations.c:stack_push`가 source top pair를 지역 변수에 보존하고, destination active prefix를 오른쪽으로 한 칸 민 뒤 index 0에 pair를 씁니다. source는 나머지 prefix를 왼쪽으로 당기고 `dst->size++`, `src->size--`로 마칩니다. `op_pa`는 B→A, `op_pb`는 A→B를 연결합니다.
- **핵심 상태 전이 또는 boundary:** `values`와 `ranks`에 같은 길이·방향의 `memmove`를 적용하므로 pair가 분리되지 않고, 두 size의 합은 변하지 않습니다. operation 중 새 allocation은 없습니다.
- **failure/no-op/edge:** `src->size == 0`이면 primitive는 상태를 그대로 둡니다. 다만 이 SHA의 emitting wrapper는 no-op 여부와 무관하게 합법 command를 출력합니다. destination capacity 검사는 없으며 두 스택이 전체 입력 크기의 buffer를 갖는다는 상위 전제를 사용합니다.
- **이후 연결:** 회전 계열이 같은 배열 표현을 완성하고, `86364d27baac`이 모든 command에서 pair·원소 수·capacity 범위를 검사합니다.
- **개발 흐름의 다음 관련 커밋:** `745ec72850d2`는 같은 스택 내부의 순환 이동에서 active prefix와 tail을 어떻게 보존하는가?

### `745ec72850d2` — feat(operation): 스택 정방향 회전을 구현
- **중요도:** B
- **태그:** CORE, STACK_STATE
- **원자료에서 확인된 역할:** Adds forward rotation within the same representation.
- **커밋 분류 요약:** Adds forward rotation for one or both stacks while preserving value-rank pairing.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 745ec72850d2`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `745ec72850d2` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- forward rotation에서 top pair 저장 → active prefix shift → tail 복원의 정확한 범위를 확인합니다.
- size 0/1 no-op과 `ra`/`rb`/`rr` wrapper를 추적합니다.
- `rr`의 두 내부 transition과 한 번의 public emission이 분리되는 지점을 확인합니다.
- capacity와 total logical element 수가 변하지 않는지 코드 기준으로 기록합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** top pair 교환과 A/B 간 push만 있고, top을 bottom으로 보내는 명령은 없었습니다.
- **이 commit의 구현 역할:** `745ec72850d2:src/operations.c:stack_rotate`가 index 0의 value/rank를 저장하고 `[1, size)`를 `[0, size-1)`로 이동한 뒤 저장 pair를 `size - 1`에 복원합니다. `op_ra`, `op_rb`, `op_rr`가 이를 노출합니다.
- **핵심 상태 전이 또는 boundary:** size와 capacity는 그대로이고 active pair의 순환 순서만 바뀝니다. `rr`는 두 primitive를 호출하지만 public output은 `rr\n` 한 줄입니다.
- **failure/no-op/edge:** size가 0 또는 1이면 primitive는 no-op입니다. 이 시점에는 출력 API가 실패를 반환하지 않습니다.
- **이후 연결:** `68dfd1b1fb58`의 reverse rotate와 서로 역연산 관계를 이루며, 두 테스트 commit이 exact transition과 no-op을 잠급니다.
- **개발 흐름의 다음 관련 커밋:** `68dfd1b1fb58`은 마지막 pair를 top으로 올릴 때 겹치는 이동 범위를 어떻게 계산하는가?

### `68dfd1b1fb58` — feat(operation): 스택 역방향 회전을 구현
- **중요도:** B
- **태그:** CORE, STACK_STATE
- **원자료에서 확인된 역할:** Completes the instruction vocabulary with reverse rotation.
- **커밋 분류 요약:** Adds reverse rotation and the `rra`, `rrb`, and `rrr` command wrappers.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 68dfd1b1fb58`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `68dfd1b1fb58` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- reverse rotation에서 last active pair 저장 → preceding pairs tail shift → top 복원의 인덱스를 확인합니다.
- size >= 2에서 forward rotation과 inverse 관계가 실제 transition으로 성립하는 예를 직접 추적합니다.
- `rra`/`rrb`/`rrr`의 optional emission 경계를 확인합니다.
- empty/single-element no-op이 checker replay에서 별도 command error를 만들지 않는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** forward rotate까지 있어 top→bottom은 가능하지만 bottom→top 명령이 없었습니다.
- **이 commit의 구현 역할:** `68dfd1b1fb58:src/operations.c:stack_reverse_rotate`가 `size - 1`의 pair를 저장하고 `[0, size-1)`를 한 칸 오른쪽으로 이동한 뒤 index 0에 복원합니다. `op_rra`, `op_rrb`, `op_rrr`가 전체 명령 어휘를 완성합니다.
- **핵심 상태 전이 또는 boundary:** 예를 들어 `[a,b,c]`에 rotate를 적용하면 `[b,c,a]`, 이어 reverse rotate를 적용하면 `[a,b,c]`가 됩니다. 두 배열에 같은 처리를 하므로 pair·size·capacity가 유지됩니다.
- **failure/no-op/edge:** size 0/1에서는 no-op이며 command 자체는 합법입니다. 후속 checker의 exact dispatch는 이를 invalid command로 바꾸지 않고 silent no-op으로 재생합니다.
- **이후 연결:** `86364d27baac`이 11개 command 전체의 보존 성질을, `7eb6890c2c13`이 정확한 배열 상태와 inactive entry 보존까지 확인합니다.
- **개발 흐름의 다음 관련 커밋:** `86364d27baac`의 보존 검사는 exact ordering 오류까지 잡는가, 아니면 pair·개수·범위만 잡는가?

### `86364d27baac` — test(operation): 값과 순위의 보존 불변식을 검증
- **중요도:** A
- **태그:** TEST, STACK_STATE, RISK
- **원자료에서 확인된 역할:** Verifies pair association, element conservation, size bounds, and sequences across all commands.
- **커밋 분류 요약:** Checks pair preservation, element conservation, size bounds, and multi-operation sequences.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 86364d27baac`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `86364d27baac` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- fixture가 5개의 distinct value-rank pair를 A/B에 어떻게 분배하는지 확인합니다.
- pair association, combined size, per-stack capacity bound, rank uniqueness를 검사하는 helper/loop를 찾습니다.
- 11개 command 각각의 isolated 실행과 all-command sequence 실행이 어떤 차이로 구성되는지 확인합니다.
- `emit = 0`을 사용해 output과 state semantics를 분리하는 지점을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** `values[i]`와 `ranks[i]`의 고정 대응, A/B 합계 5, 각 stack의 `0 <= size <= capacity`, rank 0..4의 정확히 한 번 존재입니다. fixture는 A에 `(40,4),(10,1),(30,3)`, B에 `(20,2),(0,0)`을 둡니다.
- **재현하는 failure/boundary:** 각 command를 초기 fixture에 독립 적용하고, 별도로 11개 command를 연속 적용해 겹치는 `memmove`, 양 stack size 변경, combined operation의 조합을 통과시킵니다.
- **test technique:** C 함수 단위 불변 조건 test입니다. `emit=0`으로 stdout을 배제하고 shared production operation을 직접 호출합니다.
- **통과하는 production path:** `tests/operation_invariants.c`의 command 적용 helper → `op_*` wrapper → `stack_swap`/`stack_push`/`stack_rotate`/`stack_reverse_rotate`입니다.
- **이 테스트가 증명하는 것:** 각 관찰 지점에서 알려진 five-pair 집합이 유실·복제·오결합되지 않고 size/capacity 범위를 지키며, 단일 command와 한 복합 sequence가 불변 조건을 유지한다는 점입니다.
- **이 테스트가 증명하지 않는 것:** 명령별 exact post-state, no-op에서 backing array가 전혀 바뀌지 않는지, 모든 가능한 state, emitted text, write failure는 증명하지 않습니다. production operation과 같은 C 구현을 호출하므로 독립 oracle도 아닙니다.
- **성격:** 대표 fixture를 사용한 deterministic 불변 조건 regression입니다. exhaustive state-space test는 아닙니다.
- **막는 후속 회귀:** value만 이동하고 rank를 빠뜨리는 변경, push에서 size 한쪽만 갱신하는 변경, active 원소를 유실·복제하는 잘못된 `memmove`를 막습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** 11개 명령 구현은 완성됐지만 pair와 전체 원소 보존을 자동으로 확인하는 증거가 없었습니다.
- **주요 boundary/decision:** exact 배열을 먼저 고정하지 않고, 어떤 합법 transition에서도 유지되어야 하는 집합·pair·size 불변 조건을 공통 검사 함수로 분리했습니다.
- **state / 소유권 / failure 변화:** production state나 소유권은 바뀌지 않습니다. Makefile에 operation test executable과 `test` 실행 경로가 추가되어 shared operation의 상태 변경을 출력 없이 관찰할 수 있게 됐습니다.
- **보장 / 비보장:** 위 불변 조건은 대표 fixture의 모든 command와 한 sequence에서 검증되지만, 서로 다른 잘못된 순열처럼 보존 성질만 만족하는 오류는 통과할 수 있습니다.
- **후속 검증 또는 수정 연결:** `7eb6890c2c13`이 command별 exact-state/no-op 검사를 추가해 이 빈틈을 보완하고, Thread 4의 `5b7559278909`가 Python 독립 replay로 shared-implementation risk를 낮춥니다.
- **개발 흐름의 다음 관련 커밋:** `7eb6890c2c13`은 보존 검사를 통과할 수 있는 잘못된 순서를 어떤 expected-state 비교로 잡는가?

### `7eb6890c2c13` — test(operation): 정확한 상태 전이와 no-op을 검증
- **중요도:** B
- **태그:** TEST, STACK_STATE, EDGE
- **원자료에서 확인된 역할:** Locks down exact states and insufficient-stack no-op behavior.
- **커밋 분류 요약:** Adds exact expected states and empty or single-element no-op cases for every operation.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 7eb6890c2c13`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `7eb6890c2c13` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- table-driven expected state가 active `values`, `ranks`, `size`, unchanged `capacity`를 어떻게 표현하는지 확인합니다.
- 11개 command에 대한 exact postcondition 비교와 command enum/name/application 공통화 구조를 확인합니다.
- empty/single-element에서 swap/rotate no-op, empty-source push no-op 케이스를 추적합니다.
- no-op 검증이 inactive backing entries까지 확인하는지 실제 assertion을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** 각 command의 정확한 `values`/`ranks` active 순서, 두 `size`, 변경되지 않는 capacity, insufficient-stack 명령의 완전한 no-op입니다.
- **재현하는 failure/boundary:** 11개 command의 expected A/B state를 table로 고정하고, capacity 2의 작은 fixture로 empty/single-element swap·rotate와 empty-source push를 실행합니다.
- **test technique:** table-driven C unit regression입니다. command enum/name/application을 공통화하고 exact array comparison을 사용합니다.
- **통과하는 production path:** command table → operation wrapper(`emit=0`) → 해당 primitive → `stack_has_exact_state` 또는 `small_stack_is_unchanged`입니다.
- **이 테스트가 증명하는 것:** 선택 fixture에서 command별 정확한 순서·size·capacity를 확인하며, no-op에서는 active range뿐 아니라 inactive backing entries도 그대로임을 확인합니다.
- **이 테스트가 증명하지 않는 것:** 모든 크기와 모든 배열 상태, emitted stream, I/O 실패, allocation 실패, 독립 semantics는 증명하지 않습니다.
- **성격:** deterministic exact-state 및 edge/no-op regression입니다.
- **막는 후속 회귀:** rotate 방향 반전, source/destination 혼동, combined command 한쪽 누락, no-op에서 backing array를 건드리는 변경을 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** `86364d27baac`은 pair·개수·범위는 잡지만 순서가 잘못돼도 같은 pair 집합이면 통과할 수 있었습니다.
- **이 commit의 구현 역할:** command별 expected state와 작은 no-op fixture를 추가해 transition의 정확한 결과를 고정합니다.
- **핵심 상태 전이 또는 boundary:** active 배열, size, capacity를 함께 비교하고, no-op 검사는 inactive slot까지 비교합니다.
- **failure/no-op/edge:** A/B 모두 empty, 한쪽만 single-element인 push, single-element swap/rotate 등 불충분 state를 합법 no-op으로 확인합니다.
- **이후 연결:** Thread 1의 state semantics는 이 commit으로 두 수준의 증거를 갖추지만, 독립 correctness와 I/O 실패는 Thread 4와 6이 맡습니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| value-rank pairing | 96b5324448e4 | c0de1a1b18bb → 73d2deb30224 → 745ec72850d2 → 68dfd1b1fb58 | - | - | 86364d27baac, 7eb6890c2c13 | `96b5324448e4:include/push_swap.h:t_stack`에서 병렬 배열로 도입되고, 각 primitive가 동일 인덱스·동일 이동을 수행합니다. 첫 테스트는 pair 집합을, 둘째 테스트는 exact 배열을 확인합니다. |
| A/B 전체 원소 보존과 capacity 범위 | 96b5324448e4 | 73d2deb30224 및 회전 계열에서 유지 | - | - | 86364d27baac, 7eb6890c2c13 | push는 두 size를 반대 방향으로 한 번씩 갱신하고 회전은 size를 바꾸지 않습니다. 테스트는 합계 5와 각 capacity bound 및 exact size를 확인합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| 한 배열만 이동하여 value-rank association 손상 | 모든 operation이 pair를 함께 이동 | - | 86364d27baac / 7eb6890c2c13 | primitive마다 value와 rank에 대응되는 swap/copy/`memmove`가 쌍으로 존재합니다. 하나를 누락하면 mapping 또는 exact-state assertion이 실패합니다. |
| push/rotation의 size 또는 active prefix 손상 | operation state transition 규칙 | - | 86364d27baac / 7eb6890c2c13 | push는 저장→destination shift/copy→source shift→두 size 갱신 순서이고, rotation은 size를 유지한 채 active prefix만 순환합니다. 합계·bounds·exact-state 검사가 이를 잠급니다. |
| generator와 checker가 서로 다른 명령 의미를 가짐 | c0de1a1b18bb의 shared wrapper 경계 | - | 후속 independent oracle은 Thread 4의 5b7559278909 | 두 실행 파일이 같은 `op_*`를 emit on/off로 재사용합니다. 공유 결함 가능성은 남으므로 Python list model이 별도 oracle이 됩니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| A/B의 `values` 버퍼 | 소유 객체와 lifecycle이 없음 | 96b5324448e4 | 각 stack이 capacity 크기의 버퍼를 소유하고 active prefix를 operation이 이동하며 `stack_free`가 해제 | `96b5324448e4:src/stack.c:stack_init/stack_free` |
| A/B의 `ranks` 버퍼 | 없음 | 96b5324448e4 및 operation commits | `values`와 동일 lifetime·인덱스 이동을 갖는 병렬 버퍼 | `96b5324448e4:include/push_swap.h:t_stack`, `68dfd1b1fb58:src/operations.c` |
| operation primitive | 없음 | c0de1a1b18bb → 68dfd1b1fb58 | swap/push/rotate/reverse-rotate가 출력 없이 pair state를 변경 | `68dfd1b1fb58:src/operations.c:stack_*` |
| emit 가능한 command wrapper | 없음 | c0de1a1b18bb → 68dfd1b1fb58 | 11개 wrapper가 primitive를 재사용하고 `emit`으로 generator/checker 동작을 분리 | `68dfd1b1fb58:src/operations.c:op_*` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** `7eb6890c2c13` 시점에는 A/B가 각각 `values`와 `ranks`의 병렬 buffer, `size`, `capacity`를 가지며 11개 command가 두 배열을 같은 논리 이동으로 처리합니다. push만 두 stack의 size를 반대 방향으로 바꾸고 나머지는 size를 유지합니다. shared wrapper의 `emit`은 generator의 serialization과 checker의 silent replay를 같은 transition에 연결합니다. 보존 invariant test와 exact-state/no-op test가 서로 다른 오류 범위를 담당합니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** 이 개발 흐름에는 fix commit이 없고 테스트는 shared C operation을 직접 사용합니다. 입력이 유효한 pair/rank를 만드는 책임은 Thread 2, 정렬 sequence의 독립 correctness는 Thread 4, allocation·read·write failure는 Thread 6에 남습니다. 이 작업 환경에서는 repository checkout이 불가능해 테스트 executable을 실행하지 않았으며, 실행 결과를 주장하지 않고 각 SHA의 코드와 assertion만 확인했습니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: ``t_stack` 생성 → pair-preserving primitive → `sa/sb/ss`, `pa/pb`, rotate 계열 wrapper → 항상 유지해야 하는 조건 test → exact transition/no-op test``
- **학습자 최종 flow:** `96b5324448e4:stack_init`가 두 병렬 buffer를 소유하는 empty/allocated state를 만듭니다 → `c0de1a1b18bb`~`68dfd1b1fb58:stack_*`가 active pair를 silent mutation합니다 → 같은 SHA들의 `op_*`가 `emit`에 따라 하나의 public command를 직렬화하거나 silent replay합니다 → `86364d27baac:tests/operation_invariants.c`가 pair·합계·bounds를 확인합니다 → `7eb6890c2c13:tests/operation_invariants.c`가 exact state와 backing-array no-op을 확인합니다.
- **실제 코드 삽입:** 핵심 결정은 `t_stack`의 병렬 배열과 각 primitive의 동일 이동입니다. 예를 들어 `c0de1a1b18bb:src/operations.c:stack_swap`은 size 2 미만에서 반환한 뒤 `values[0/1]`과 `ranks[0/1]`을 각각 같은 방식으로 교환합니다. 이후 command도 이 pair 단위를 유지합니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# 입력 문법, 좌표 압축, 크기 계산 안전성

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** Parsing evolves from lexical validity into the normalization contract required by sorting. Coordinate compression is the turning point: arbitrary signed values become a permutation over `0..n-1`. Later tests define the external grammar precisely, and the size fix closes the remaining gap between logically counted tokens and safely allocated storage.
- **학습 목표:** 입력 문자열이 strict integer grammar를 통과해 unique dense rank permutation으로 정규화되고, 그 과정의 allocation과 크기 계산이 안전하게 닫히는 과정을 복원합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- 숫자 token의 sign, digit, `INT_MIN`/`INT_MAX` 경계는 어떤 계산 순서로 검증됩니까?
- argv 내부 C whitespace tokenization이 왜 count pass와 fill pass로 나뉘며, 임시 substring을 만들지 않는가?
- duplicate rejection과 lower-bound rank assignment가 `0..n-1` bijection을 어떻게 만든는가?
- parser의 all-or-nothing 소유권은 어느 failure branch에서 보장됩니까?
- 049ecd429548이 logical token count와 allocation byte count 양쪽을 왜 따로 방어하는가?

## 3. 완료 기준
- 허용/거절 입력을 실제 parser branch와 연결할 수 있습니다.
- coordinate compression 전후의 `values`와 `ranks`를 한 입력 예제로 직접 추적할 수 있습니다.
- temporary sorted buffer의 생성/해제와 parser 실패 시 A cleanup을 코드로 확인했습니다.
- 크기 narrowing/곱셈 overflow 방어가 들어간 정확한 위치와 이전 코드의 위험을 비교했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `f36ad8899b5f` | feat(parse): 개별 인자의 부호 있는 정수를 파싱 | B | INPUT, EDGE | Establishes strict signed ASCII integer parsing and `int` range checks. |
| 2 | `3bfb465ebdb1` | feat(parse): 공백으로 결합된 인자 토큰을 처리 | B | INPUT, EDGE | Extends the grammar to mixed argv and C-whitespace token spans with exact allocation sizing. |
| 3 | `e09cf45e21cd` | feat(parse): 중복 입력을 거절하고 상대 순위를 계산 | S | CORE, INPUT, SORT | Rejects duplicates and establishes the dense order-preserving rank bijection. |
| 4 | `4cc9783286c0` | test(parser): 정상 입력과 오류 입력을 검증 | B | TEST, INPUT | Adds public parser acceptance and rejection tests. |
| 5 | `44a4da8bc63d` | test(cli): 입력 경계와 무인자 실행을 검증 | B | TEST, INPUT, EDGE | Expands boundary evidence for signs, zero spellings, whitespace, integer endpoints, timeouts, and no-argument stdin behavior. |
| 6 | `049ecd429548` | fix(parse): 토큰 수와 배열 크기 계산을 방어 | A | INPUT, EDGE, RISK | Hardens logical token counts and byte-size calculations against narrowing and overflow. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Critical 불변 조건**
  - After parsing unique input of size `n`, ranks form a bijection over `0..n-1` and preserve the ordering of original values.
  - Parser construction is all-or-nothing, and every owned allocation is released on every exit path.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `f36ad8899b5f` — feat(parse): 개별 인자의 부호 있는 정수를 파싱
- **중요도:** B
- **태그:** INPUT, EDGE
- **원자료에서 확인된 역할:** Establishes strict signed ASCII integer parsing and `int` range checks.
- **커밋 분류 요약:** Parses one signed ASCII decimal integer per argument with explicit `int` bounds.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only f36ad8899b5f`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `f36ad8899b5f` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- optional `+`/`-`, sign-only reject, ASCII digit loop를 담당하는 numeric parser를 확인합니다.
- wider accumulator와 positive/negative magnitude limit 비교 순서가 `INT_MIN`을 허용하는 방식을 추적합니다.
- no arguments가 valid empty stack으로 끝나는 control flow를 확인합니다.
- stack allocation 후 token failure가 발생했을 때 partial stack을 free하고 failure만 반환하는 경로를 확인합니다.
- 이 SHA에서 `ranks`가 아직 original values를 임시 mirror하는 위치를 기록합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** stack model과 operation은 있지만 argv를 `t_stack`으로 만드는 parser가 없었습니다.
- **이 commit의 구현 역할:** `f36ad8899b5f:src/parser.c:parse_token`이 optional sign 뒤에 최소 한 자리 ASCII digit을 요구하고, 더 넓은 정수형 accumulator로 decimal magnitude를 누적합니다. 양수 한계는 `INT_MAX`, 음수 한계는 `INT_MAX + 1`이므로 `-2147483648`은 허용하고 그 밖의 범위 초과는 거절합니다.
- **핵심 상태 전이 또는 boundary:** `parse_arguments`는 인자 수만큼 A를 한 번 할당하고 각 token을 `values[index]`와 `ranks[index]` 양쪽에 임시로 복사한 뒤 size를 채웁니다. 인자가 없으면 allocation 없는 empty stack 성공입니다.
- **failure/no-op/edge:** `+`/`-`만 있는 문자열, 비 ASCII digit, suffix, 범위 초과가 실패합니다. allocation 뒤 어느 token에서든 실패하면 `stack_free`로 두 배열을 모두 해제하고 실패를 반환합니다.
- **이후 연결:** `3bfb465ebdb1`이 한 argv 안의 복수 token을 허용하고, `e09cf45e21cd`가 mirror rank를 실제 dense rank로 교체합니다.
- **개발 흐름의 다음 관련 커밋:** `3bfb465ebdb1`은 count pass와 fill pass가 같은 whitespace grammar를 공유해 exact capacity를 어떻게 보장하는가?

### `3bfb465ebdb1` — feat(parse): 공백으로 결합된 인자 토큰을 처리
- **중요도:** B
- **태그:** INPUT, EDGE
- **원자료에서 확인된 역할:** Extends the grammar to mixed argv and C-whitespace token spans with exact allocation sizing.
- **커밋 분류 요약:** Extends parsing to all C whitespace separators and mixed quoted or split arguments.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 3bfb465ebdb1`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `3bfb465ebdb1` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- C whitespace 판별과 각 argv 안의 token `[start, end)` span discovery를 확인합니다.
- 첫 pass의 token count와 두 번째 pass의 direct parse/fill가 동일한 grammar를 사용하는지 확인합니다.
- temporary substring allocation 없이 exact final capacity로 stack을 한 번 할당하는 흐름을 추적합니다.
- empty argument는 허용하지만 supplied argv 전체가 zero token이면 reject하는 분기를 확인합니다.
- conversion failure 시 allocated stack cleanup을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** 각 argv가 정확히 하나의 정수 token이어야 했습니다. 따라서 `"3 2"`처럼 quoted group이나 tab/newline을 포함한 입력을 처리하지 못했습니다.
- **이 commit의 구현 역할:** `3bfb465ebdb1:src/parser.c:is_space`가 space, tab, newline, vertical tab, form feed, carriage return을 구분자로 정의합니다. 첫 pass는 각 argv의 `[start,end)` token span 수를 세고, 두 번째 pass는 같은 span을 `parse_token`에 직접 전달합니다.
- **핵심 상태 전이 또는 boundary:** 전체 token 수를 먼저 얻어 A를 exact capacity로 한 번만 할당하며, substring을 별도로 소유하지 않습니다. 빈 argv는 다른 token이 있으면 무시되지만 제공된 모든 argv에서 token 수가 0이면 오류입니다.
- **failure/no-op/edge:** token conversion 실패 시 이미 할당된 A를 `stack_free`합니다. 이 SHA의 count/index는 아직 `int`이므로 매우 큰 논리 token 수의 narrowing 위험은 남습니다.
- **이후 연결:** `e09cf45e21cd`가 채워진 값에 uniqueness와 dense rank를 부여하고, `049ecd429548`이 count와 allocation byte 계산의 타입 범위를 보강합니다.
- **개발 흐름의 다음 관련 커밋:** `e09cf45e21cd`는 arbitrary signed values를 원래 순서를 잃지 않고 `0..n-1` rank permutation으로 어떻게 바꾸는가?

### `e09cf45e21cd` — feat(parse): 중복 입력을 거절하고 상대 순위를 계산
- **중요도:** S
- **태그:** CORE, INPUT, SORT
- **원자료에서 확인된 역할:** Rejects duplicates and establishes the dense order-preserving rank bijection.
- **커밋 분류 요약:** Rejects duplicates and maps arbitrary values to a dense, order-preserving rank permutation.

#### 원자료에서 확인된 맥락
- **문제:** The input domain contains arbitrary signed integers, but the sorting strategies need only a compact, non-negative representation of relative order. Duplicate values would make a unique target permutation undefined under the project's contract.
- **결정:** Copy and sort the values, reject adjacent duplicates, and assign each original value the lower-bound index in the sorted copy as its rank.
- **중요한 이유:** The result is a bijection over `0..n-1` that preserves ordering. Tiny sorting can compare small relative ranks, radix sorting can traverse finite non-negative bit patterns, and the maximum required bit count depends only on input size.
- **변경 내용:** The parser adds an overflow-safe comparator, a binary lower-bound search, temporary sorted storage, duplicate rejection, rank assignment, and cleanup on every outcome.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only e09cf45e21cd`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `e09cf45e21cd` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- original `values` copy를 정렬하는 temporary buffer의 allocation, fill, `qsort`, free 경로를 확인합니다.
- `qsort` comparator가 subtraction이 아닌 relational result를 사용하는지 확인합니다.
- sorted copy의 adjacent equality로 duplicate를 reject하는 위치를 확인합니다.
- lower-bound binary search가 각 original value를 dense rank로 바꾸는 과정을 입력 하나로 추적합니다.
- ranking/duplicate failure의 temporary buffer와 stack cleanup 순서를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — S
- **이 commit 직전 상태:** `3bfb465ebdb1`은 strict token grammar와 exact token capacity를 제공하지만 `ranks[i] == values[i]`인 mirror 상태였습니다. duplicate도 허용했습니다.
- **해결하려던 문제:** tiny/radix sorter가 signed 32-bit 값 자체 대신 입력 크기로 제한된 비음수 순위를 다뤄야 했고, 각 원소가 유일한 최종 위치를 가져야 했습니다.
- **기존 설계가 충분하지 않았던 이유:** 원본 값에 직접 bit radix를 적용하면 음수 표현과 전체 32비트를 처리해야 합니다. mirror rank는 최대값이 입력 크기와 무관하고, duplicate가 있으면 `0..n-1`의 unique permutation과 단일 정렬 목표를 만들 수 없습니다.
- **핵심 결정:** `e09cf45e21cd:src/parser.c:assign_ranks`가 원본 `values`를 임시 배열에 복사해 `qsort`하고, 인접 값이 같으면 실패하며, 각 원본 값의 lower-bound index를 `ranks`에 씁니다. comparator는 뺄셈 overflow를 피하려고 관계 비교 결과만 반환합니다.

```c
/* e09cf45e21cd:src/parser.c:compare_ints */
return ((*left > *right) - (*left < *right));
```

- **state / 불변 조건 / 소유권 / lifecycle 변화:** parser 성공 후 `values`는 원본 정수를 유지하고 `ranks`는 `0..n-1`의 permutation이 됩니다. 임시 sorted buffer는 `assign_ranks`가 생성하고 성공·duplicate 실패 모두에서 해제합니다. `parse_arguments`는 rank assignment 실패를 받으면 A의 두 영구 buffer도 해제해 caller에 partial stack을 넘기지 않습니다.
- **failure scenario:** subtraction comparator는 `INT_MAX - INT_MIN`에서 overflow할 수 있고, duplicate 검사를 건너뛰면 같은 rank가 여러 원소에 배정될 수 있습니다. lower-bound가 잘못되면 value order와 rank order가 어긋나 radix가 rank를 정렬해도 원본 값은 정렬되지 않습니다.
- **이 commit이 보장하는 것:** unique input 크기 `n`에 대해 각 rank가 정확히 한 번 나타나고 `values[i] < values[j]`이면 `ranks[i] < ranks[j]`입니다. 예를 들어 `[30,-5,10]`은 sorted copy `[-5,10,30]`, rank `[2,0,1]`이 됩니다.
- **아직 보장하지 않는 것:** 이 SHA 자체에는 parser test가 없고, 거대한 token count와 `capacity * sizeof(int)`의 representability guard도 없습니다. sorting command의 correctness 역시 별도 Thread가 검증합니다.
- **후속 fix/test:** `4cc9783286c0`과 `44a4da8bc63d`가 accepted/rejected CLI grammar와 duplicate를 검증합니다. `049ecd429548`은 token count narrowing과 allocation byte overflow 위험을 수정합니다. Thread 4의 independent replay가 rank 기반 sorter의 최종 값 정렬을 간접 확인합니다.
- **개발 흐름의 다음 관련 커밋:** `4cc9783286c0`은 parser의 내부 rank 배열을 직접 보지 않고 어떤 public CLI 관찰로 grammar와 uniqueness를 검증하는가?

### `4cc9783286c0` — test(parser): 정상 입력과 오류 입력을 검증
- **중요도:** B
- **태그:** TEST, INPUT
- **원자료에서 확인된 역할:** Adds public parser acceptance and rejection tests.
- **커밋 분류 요약:** Adds end-to-end parser acceptance and rejection cases through both executables.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 4cc9783286c0`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `4cc9783286c0` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- Python harness가 `push_swap`의 stdout program을 checker stdin으로 전달하는 integration 흐름을 확인합니다.
- accepted mixed split/quoted input과 no-argument case가 어떤 status/stdout/stderr 조건으로 검증되는지 확인합니다.
- duplicate, overflow, suffix, sign-only, zero-token, cross-argv duplicate rejection case를 확인합니다.
- invalid case에서 status 1, empty stdout, exact `Error\n` stderr를 동시에 검사하는 assertion을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** accepted argv는 generator가 성공하고 그 command stream을 checker가 `OK`로 판정하며, invalid argv는 public error protocol을 지켜야 합니다.
- **재현하는 failure/boundary:** mixed `['3 2','1']`, no-argument success와 duplicate, int overflow, `12a`, sign-only `+`, zero-token argv, argv 경계를 넘는 duplicate를 사용합니다.
- **test technique:** Python CLI integration test입니다. generator stdout을 그대로 product checker stdin에 연결합니다.
- **통과하는 production path:** Python subprocess → `push_swap main` → `parse_arguments`/rank assignment → sort/output → `checker main` → 같은 parser와 command replay입니다.
- **이 테스트가 증명하는 것:** 나열된 valid/invalid 외부 입력에서 status, stdout, exact `Error\n`, checker verdict가 의도대로임을 확인합니다.
- **이 테스트가 증명하지 않는 것:** 내부 rank bijection을 직접 검사하지 않고 generator/checker가 parser와 operation을 공유합니다. 모든 whitespace·numeric spelling·크기 overflow·allocation failure도 다루지 않습니다.
- **성격:** 대표 acceptance/rejection을 묶은 broad CLI integration regression입니다.
- **막는 후속 회귀:** quoted token 분할 실패, duplicate 누락, invalid input에서 command를 일부 출력하는 변경, 오류 stream/status 변경을 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** parser와 rank assignment는 있었지만 public executable 기준의 정상·오류 증거가 없었습니다.
- **이 commit의 구현 역할:** `tests/run_tests.py`와 Make `test` 경로를 추가해 parser 결과를 generator와 checker를 통해 관찰합니다.
- **핵심 상태 전이 또는 boundary:** valid 입력은 command stream을 생성하고 checker가 최종 state를 판정하며, invalid 입력은 sort/replay로 진입하지 않고 status 1·empty stdout·`Error\n`로 끝납니다.
- **failure/no-op/edge:** no-argument `push_swap`은 status 0과 빈 출력입니다. zero-token argument가 제공된 경우는 오류입니다.
- **이후 연결:** `44a4da8bc63d`가 numeric/whitespace/no-values stdin 경계를 더 세밀하게 확장합니다.
- **개발 흐름의 다음 관련 커밋:** `44a4da8bc63d`는 같은 grammar에서 허용되는 여러 zero/sign/whitespace 표기와 no-values checker behavior를 어떻게 분리하는가?

### `44a4da8bc63d` — test(cli): 입력 경계와 무인자 실행을 검증
- **중요도:** B
- **태그:** TEST, INPUT, EDGE
- **원자료에서 확인된 역할:** Expands boundary evidence for signs, zero spellings, whitespace, integer endpoints, timeouts, and no-argument stdin behavior.
- **커밋 분류 요약:** Expands numeric, whitespace, sign, timeout, and no-argument stdin-consumption coverage.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 44a4da8bc63d`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `44a4da8bc63d` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- accepted `+`, negative zero, leading zero, empty argv, 모든 C whitespace, exact int endpoint 케이스를 확인합니다.
- whitespace-only, long decimal, non-ASCII digit, repeated/mixed sign, duplicate zero spelling rejection을 확인합니다.
- accepted input을 두 executable로 검증하는 helper 경로와 child timeout 사용 위치를 확인합니다.
- `checker` no-values 실행이 stdin을 소비하지 않는다는 file-position check를 실제로 추적합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** strict ASCII integer grammar, 여섯 C whitespace의 동일 tokenization, `INT_MIN`/`INT_MAX` 경계, no-values checker의 stdin non-consumption입니다.
- **재현하는 failure/boundary:** `+7`, `-0`, leading zero, empty argv 혼합, 모든 C whitespace, exact endpoints를 허용하고, whitespace-only, 4096자리 decimal, non-ASCII digit, 반복·혼합 sign, 서로 다른 zero spelling duplicate를 거절합니다. no-values checker에는 `sa\n`이 든 seekable stdin을 줍니다.
- **test technique:** timeout이 있는 deterministic CLI boundary test와 file-position observation입니다.
- **통과하는 production path:** accepted helper는 generator→checker를 통과하고, rejected helper는 parser 오류 처리를 관찰합니다. no-values case는 checker `main`의 argc early return까지만 통과합니다.
- **이 테스트가 증명하는 것:** 나열된 lexical 경계와 no-values에서 stdin offset이 0으로 유지됨을 확인합니다. child timeout은 hang을 failure로 만듭니다.
- **이 테스트가 증명하지 않는 것:** `INT_MAX`개 token이나 byte-size overflow처럼 현실적으로 거대한 입력, allocation/read/write fault, 모든 Unicode 입력을 exhaustively 다루지 않습니다.
- **성격:** deterministic CLI edge regression입니다.
- **막는 후속 회귀:** locale digit 허용, whitespace 집합 축소, endpoint off-by-one, no-values checker가 command stdin을 읽는 변경을 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** 기본 accepted/rejected parser cases는 있었지만 여러 동치 표기와 정확한 외부 경계가 충분히 고정되지 않았습니다.
- **이 commit의 구현 역할:** 허용되는 표기와 거절되는 표기를 확장하고 모든 subprocess에 timeout을 적용하며, no-values checker의 stdin consumption을 file position으로 검사합니다.
- **핵심 상태 전이 또는 boundary:** no-values는 parser·reader allocation과 command loop 전에 정상 반환하므로 stdin이 그대로 남습니다.
- **failure/no-op/edge:** signed zero는 numeric value가 같으므로 두 spelling을 함께 주면 duplicate로 거절됩니다. 빈 argv 하나는 다른 token이 있으면 허용되지만 whitespace-only 전체 입력은 오류입니다.
- **이후 연결:** `049ecd429548`은 이 테스트들이 직접 만들기 어려운 count/byte representability 위험을 코드 guard로 닫습니다.
- **개발 흐름의 다음 관련 커밋:** `049ecd429548`은 lexical validity 이후 logical count와 allocation byte 수를 각각 어느 타입 경계에서 거절하는가?

### `049ecd429548` — fix(parse): 토큰 수와 배열 크기 계산을 방어
- **중요도:** A
- **태그:** INPUT, EDGE, RISK
- **원자료에서 확인된 역할:** Hardens logical token counts and byte-size calculations against narrowing and overflow.
- **커밋 분류 요약:** Prevents token-count narrowing and allocation-size overflow before filling stack arrays.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 049ecd429548`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `049ecd429548` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- 직전 관련 parser 코드와 비교해 string position/per-argument count가 `size_t`로 바뀐 위치를 확인합니다.
- aggregate token count가 stack model의 `int` size에 들어가는지 변환 전에 검증하는 branch를 확인합니다.
- `stack_init`에서 `capacity * sizeof(int)`의 `size_t` representability를 두 buffer allocation 전에 검사하는지 확인합니다.
- guard가 없던 이전 코드에서 wrapped count/byte size가 어떤 under-allocation으로 이어질 수 있는지 직접 계산 예로 기록합니다.
- 이 SHA에 직접 회귀 테스트 변경이 있는지 `git show --name-only`와 test diff로 확인하고, 없다면 기존 boundary tests의 coverage 한계를 기록합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 수정 과정
- **기존 가정:** `3bfb465ebdb1` 계열 코드는 string index와 token count를 `int`로 누적하고, `stack_init`은 양수 capacity라면 곧바로 `capacity * sizeof(int)`를 allocation size로 사용했습니다.
- **실제 failure 또는 위험:** 실제 token 수가 `INT_MAX`를 넘으면 count가 stack model의 `int`에 들어가지 않습니다. 또한 `capacity`가 표현 가능해도 byte 곱셈이 `size_t`를 넘으면 작은 크기로 wrap되어 이후 fill이 allocation 밖에 쓸 수 있습니다.
- **root cause:** 논리 개수의 표현 범위(`int`)와 memory byte 수의 표현 범위(`size_t`)를 서로 다른 단계에서 검증하지 않고, narrow type로 scan/count한 뒤 곱셈을 신뢰한 것이 원인입니다.
- **수정된 불변 조건/decision:** scan position과 per-argument count는 `size_t`로 유지하고, aggregate count는 `INT_MAX` 이하임을 확인한 뒤에만 `int` capacity로 변환합니다. `stack_init`은 두 allocation 전에 `capacity <= SIZE_MAX / sizeof(int)`를 확인합니다.
- **실제 수정 코드:** `049ecd429548:src/parser.c:count_arguments`는 `argument_count > (size_t)INT_MAX - count`를 실패 조건으로 사용합니다. `049ecd429548:src/stack.c:stack_init`의 핵심 guard는 다음과 같습니다.

```c
/* 049ecd429548:src/stack.c:stack_init */
if ((size_t)capacity > (size_t)-1 / sizeof(int))
    return (0);
```

- **regression test:** 이 commit의 changed-file 목록에는 test 변경이 없습니다. `44a4da8bc63d`의 4096자리 decimal 등은 lexical magnitude를 다루지만 `INT_MAX`개 token이나 `SIZE_MAX` 곱셈을 직접 재현하지 않으므로 이 fix의 정확한 overflow branch를 실행 증명하지 않습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** grammar와 ordinary boundary는 검증됐지만, 매우 큰 입력의 count/narrowing 및 allocation-size 산술은 별도 전제가 없었습니다.
- **주요 boundary/decision:** parser의 logical token domain을 `size_t`로 세다가 model이 수용할 수 있는 `INT_MAX`에서 명시적으로 거절하고, model allocation은 byte 곱셈 representability를 다시 검사합니다.
- **state / 소유권 / failure 변화:** 성공 state는 바뀌지 않습니다. 실패는 allocation 전에 발생하므로 partial buffer 소유권이 생기지 않거나, 기존 all-or-nothing cleanup으로 종료됩니다.
- **보장 / 비보장:** count cast와 `sizeof(int)` 곱셈 wrap으로 인한 under-allocation을 막습니다. 운영체제가 큰 정상 allocation을 거절하는 경우는 여전히 allocator failure로 처리하며, 이 SHA에는 direct 회귀 테스트가 없습니다.
- **후속 검증 또는 수정 연결:** 별도 post-fix test는 source에 지정되지 않았습니다. Thread 6의 allocation fault sweep은 allocator가 `NULL`을 반환하는 cleanup을 다루지만 이 산술 guard 자체의 경계 입력을 대체하지 않습니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| parser construction all-or-nothing | f36ad8899b5f | 3bfb465ebdb1, e09cf45e21cd | - | - | 4cc9783286c0, 44a4da8bc63d | token 실패는 `stack_free`, rank temporary는 성공·중복 실패 모두 free, rank 실패는 A까지 free합니다. CLI tests는 invalid input에서 빈 stdout과 `Error\n`을 확인합니다. |
| dense rank bijection `0..n-1` | e09cf45e21cd | - | - | - | 4cc9783286c0 및 sorting verification으로 간접 확인 | sorted copy의 adjacent duplicate reject 후 각 원본 value의 lower-bound index를 rank로 사용하므로 unique input에서 permutation과 order preservation이 성립합니다. |
| logical count / allocation byte representability | - | - | 049ecd429548 이전 설계의 잠재 위험 | 049ecd429548 | source는 별도 post-fix regression commit을 지정하지 않음 | aggregate count를 cast 전에 `INT_MAX`와 비교하고, `stack_init`이 `SIZE_MAX / sizeof(int)`를 넘는 capacity를 allocation 전에 거절합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| 잘못된 sign/digit 또는 `int` 범위 초과 | f36ad8899b5f의 numeric parser | - | 4cc9783286c0 / 44a4da8bc63d | optional sign 뒤 digit 필수, ASCII digit만 허용, 부호별 magnitude limit을 누적 전에 검사합니다. |
| duplicate input | e09cf45e21cd의 sorted-copy duplicate rejection | - | 4cc9783286c0 / 44a4da8bc63d | 정렬된 임시 copy에서 인접 equality를 검사하므로 argv 경계나 zero spelling과 무관하게 같은 numeric value를 거절합니다. |
| token 수 narrowing 또는 `capacity * sizeof(int)` overflow | 049ecd429548 | 049ecd429548 | 직접 regression coverage 존재 여부를 해당 SHA에서 확인 | 원인은 scan/count와 model/allocator의 타입 범위를 구분하지 않은 것입니다. fix는 `size_t` scan, `INT_MAX` aggregate guard, byte multiplication guard를 각각 둡니다. test 변경은 없습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| parser가 생성하는 stack A | parser 없음 | f36ad8899b5f → e09cf45e21cd → 049ecd429548 | unique values와 dense ranks를 소유하며 모든 실패에서 empty/해제 상태만 caller에 남김 | `049ecd429548:src/parser.c:parse_arguments` |
| temporary sorted copy | 없음 | e09cf45e21cd | rank assignment 동안만 parser가 소유하고 모든 결과에서 해제 | `e09cf45e21cd:src/parser.c:assign_ranks` |
| argv token span | argv 전체를 단일 token으로 해석 | 3bfb465ebdb1, 049ecd429548 | C whitespace로 찾은 `[start,end)` view이며 별도 allocation 없음; index는 `size_t` | `049ecd429548:src/parser.c` token scan helpers |
| stack allocation byte calculation | capacity 기반 직접 곱셈 | 049ecd429548 | representability 확인 뒤 두 동일 크기 buffer 할당 | `049ecd429548:src/stack.c:stack_init` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** `049ecd429548` 시점의 parser는 모든 argv에서 여섯 C whitespace를 기준으로 token span을 두 번 순회하고, optional sign과 ASCII decimal 및 `int` 범위를 엄격히 검사합니다. 전체 token 수가 model의 `int` capacity에 들어가고 allocation byte가 `size_t`에 표현될 때만 A를 구성합니다. unique values는 sorted temporary와 lower-bound로 `0..n-1` dense rank가 되며, 어느 실패에서도 partial ownership을 반환하지 않습니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** count/byte safety fix에는 직접 회귀 테스트가 없습니다. parser tests는 public grammar를 넓게 확인하지만 내부 rank permutation과 극단적 count branch를 직접 관찰하지 않습니다. 정렬 결과와 독립 replay는 Thread 3·4, allocator 실패 cleanup은 Thread 6이 담당합니다. 이 환경에서는 checkout 제한으로 테스트를 실행하지 않았고 코드·test assertion만 확인했습니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: `argv/whitespace scan → signed integer parse → exact-size stack allocation → duplicate rejection → dense rank assignment → size-safety hardening`
- **학습자 최종 flow:** `049ecd429548:src/parser.c`의 count pass가 `size_t`로 token 수를 계산하고 `INT_MAX`를 넘으면 거절합니다 → `stack_init`이 byte 곱셈을 검증하고 A의 두 buffer를 할당합니다 → fill pass가 각 `[start,end)`를 `parse_token`으로 읽어 original values를 채웁니다 → `e09cf45e21cd:assign_ranks`가 sorted copy를 만들고 duplicate를 거절한 뒤 lower-bound rank를 씁니다 → temporary를 해제하고 완성 A를 caller에 넘기거나 모든 owned buffer를 정리합니다.
- **실제 코드 삽입:** 핵심 코드는 overflow 없는 comparator, lower-bound rank assignment, `argument_count > INT_MAX - count`, `capacity > SIZE_MAX / sizeof(int)`의 두 단계 크기 guard입니다. 후자의 최소 구문은 `049ecd429548:src/stack.c:stack_init`에 위와 같이 기록했습니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# 정렬 엔진 구현

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** The thread separates bounded small-state optimization from the scalable general mechanism. Tiny sorting minimizes avoidable setup for at most five elements, while radix sorting supplies deterministic `Θ(n log n)` command behavior for larger inputs. The final integration commit is important operationally but does not duplicate the algorithmic significance of the radix decision.
- **학습 목표:** 작은 입력의 bounded case analysis와 큰 입력의 stable LSD binary radix가 어떻게 분리되고, 최종 `push_swap` 실행 순서에서 하나의 command-generation 경로로 연결되는지 복원합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- 2~3개 입력의 각 unsorted rank pattern이 어떤 최소 command sequence로 매핑됩니까?
- 4~5개 입력에서 최소 rank를 B로 옮기는 순서와 짧은 회전 방향 선택은 어떻게 구현됩니까?
- radix pass에서 one-bit group과 zero-bit group의 상대 순서가 왜 보존됩니까?
- 각 bit round의 시작 A size를 고정해서 정확히 그 수만큼 검사하는 이유는 무엇입니까?
- `push_swap` main이 A/B 소유권, sort invocation, command emission, cleanup을 어떤 순서로 조합하는가?

## 3. 완료 기준
- 2~5개 입력의 대표 state를 실제 명령과 stack 변화로 손으로 추적했습니다.
- 1463a193a4f9에서 bit count, per-round loop, `ra`/`pb` partition, `pa` restore를 코드로 확인했습니다.
- stable partition을 operation sequence와 연결해 설명할 수 있습니다.
- command complexity와 array-backed physical movement cost를 혼동하지 않고 설명할 수 있습니다.
- cf07495c97f7의 success/failure cleanup 경로와 이 시점의 I/O 한계를 확인했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `caa54cb306ad` | feat(sort): 세 개 이하의 스택을 정렬 | B | CORE, SORT | Handles the complete two- and three-element state space directly. |
| 2 | `160d1fb8d824` | feat(sort): 네다섯 개의 스택을 정렬 | B | CORE, SORT | Reduces four and five elements to the verified three-element case through B. |
| 3 | `1463a193a4f9` | feat(sort): 큰 입력을 기수 정렬로 처리 | S | CORE, SORT, HARD | Introduces stable LSD binary radix sorting for the general case. |
| 4 | `cf07495c97f7` | feat(push_swap): 정렬 명령 생성 흐름을 연결 | B | CORE, INTEGRATION | Integrates parsing, B allocation, sorting, emission, and cleanup into `push_swap`. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Major engineering difficulties**
  - Designing a stable radix partition using only the permitted stack operations and proving that lower-bit order survives later passes.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `caa54cb306ad` — feat(sort): 세 개 이하의 스택을 정렬
- **중요도:** B
- **태그:** CORE, SORT
- **원자료에서 확인된 역할:** Handles the complete two- and three-element state space directly.
- **커밋 분류 요약:** Implements direct two- and three-element sorting by relative-rank case analysis.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only caa54cb306ad`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `caa54cb306ad` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- size < 2 / already sorted early return을 확인합니다.
- 2-element case의 rank 비교와 최대 1회 swap 조건을 확인합니다.
- 3-element의 5개 unsorted relative-order case를 실제 조건식과 emitted command sequence로 표로 복원합니다.
- 모든 state change가 emitting operation wrapper를 통과하는지 확인합니다.
- 이 SHA에서 size > 3에 대한 동작이 의도적으로 아직 없는 경계를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** parser가 dense ranks를 만들고 11개 operation이 존재하지만, 정렬 command를 선택하는 함수는 없었습니다.
- **이 commit의 구현 역할:** `caa54cb306ad:src/sort.c:sort_two`는 두 rank가 역순일 때 `sa` 한 번을 호출합니다. `sort_three`는 3개 rank의 다섯 unsorted permutation을 직접 분기해 모두 emitting wrapper로 바꿉니다.
- **핵심 상태 전이 또는 boundary:** 세 rank를 `(top,middle,bottom)`으로 쓰면 `1 0 2 → sa`, `2 1 0 → sa,rra`, `2 0 1 → ra`, `0 2 1 → sa,ra`, `1 2 0 → rra`입니다. size 0/1 또는 이미 정렬이면 command가 없습니다.
- **failure/no-op/edge:** 이 SHA의 sorter는 size가 3을 넘으면 아무 일반 알고리즘도 수행하지 않습니다. operation/output API도 아직 실패를 반환하지 않습니다.
- **이후 연결:** `160d1fb8d824`가 4~5개를 3개 문제로 축소하고, Thread 4의 `5b7559278909`가 size 2~5의 모든 152개 permutation을 독립 replay합니다.
- **개발 흐름의 다음 관련 커밋:** `160d1fb8d824`는 최소 rank를 어떤 회전 방향으로 top에 올리고 B에 쌓아 3개 정렬을 재사용하는가?

### `160d1fb8d824` — feat(sort): 네다섯 개의 스택을 정렬
- **중요도:** B
- **태그:** CORE, SORT
- **원자료에서 확인된 역할:** Reduces four and five elements to the verified three-element case through B.
- **커밋 분류 요약:** Reduces four- and five-element inputs by moving successive minima to B before sorting three.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 160d1fb8d824`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `160d1fb8d824` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- 현재 stack에서 next smallest rank 위치를 찾는 함수/loop를 확인합니다.
- target index에 따라 forward/reverse rotation 중 짧은 방향을 선택하는 조건을 확인합니다.
- A가 3개 남을 때까지 minimum을 B로 보내는 횟수와 rank 0/1의 순서를 추적합니다.
- 3-element sorter 호출 후 B의 요소를 `pa`로 복원했을 때 최종 rank 순서가 되는 이유를 실제 stack trace로 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** 직접 case analysis는 최대 3개만 처리했습니다.
- **이 commit의 구현 역할:** `160d1fb8d824:src/sort.c:find_rank_index`로 다음 최소 rank의 위치를 찾고, `move_index_to_top`이 `index <= size / 2`이면 `ra`를 index회, 아니면 `rra`를 `size-index`회 호출합니다. A가 3개가 될 때까지 target rank를 증가시키며 `pb`합니다.
- **핵심 상태 전이 또는 boundary:** size 4는 rank 0 하나를 B로 보내고, size 5는 rank 0 뒤 rank 1을 B로 보냅니다. 두 번째 push 뒤 B top은 1, 그 아래는 0입니다. 남은 3개를 정렬한 뒤 `pa`를 반복하면 1이 먼저, 0이 나중에 top에 올라 최종 A의 prefix가 `0,1`이 됩니다.
- **failure/no-op/edge:** target은 dense rank라 반드시 A에서 발견된다는 parser 불변 조건을 사용합니다. 이 SHA에서도 operation failure를 표현하지 않습니다.
- **이후 연결:** `1463a193a4f9`가 bounded case를 넘어서는 general mechanism을 추가하며, exhaustive tiny test가 이 축소 논리를 검증합니다.
- **개발 흐름의 다음 관련 커밋:** `1463a193a4f9`는 각 bit round에서 zero/one group의 상대 순서를 어떤 두 stack operation 조합으로 안정적으로 유지하는가?

### `1463a193a4f9` — feat(sort): 큰 입력을 기수 정렬로 처리
- **중요도:** S
- **태그:** CORE, SORT, HARD
- **원자료에서 확인된 역할:** Introduces stable LSD binary radix sorting for the general case.
- **커밋 분류 요약:** Implements stable least-significant-bit binary radix sorting for inputs larger than five.

#### 원자료에서 확인된 맥락
- **문제:** Direct case analysis is practical only for a bounded tiny state space. The general case needs a legal command sequence with predictable growth while preserving ordering established by previously processed bits.
- **결정:** Apply stable LSD binary radix passes to dense ranks: rotate one-bit elements in A, push zero-bit elements to B, then push all of B back before advancing to the next bit.
- **중요한 이유:** Rotation preserves the one group, and the two reversals experienced by the zero group preserve its order. Each round therefore performs a stable partition, so later bits do not destroy lower-bit ordering. B returns to empty after every round, simplifying the next pass and the final correctness condition.
- **변경 내용:** The commit derives the necessary bit count from `size - 1`, scans exactly the round's starting A size, partitions by each bit with `ra` and `pb`, restores zeros with `pa`, and routes inputs above five to this strategy.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 1463a193a4f9`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `1463a193a4f9` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `size - 1`에서 필요한 bit count를 계산하는 코드를 확인합니다.
- 각 bit pass 시작 시 검사할 A의 원래 size를 고정하고 정확히 그 횟수만 loop하는지 확인합니다.
- bit=1이면 `ra`, bit=0이면 `pb`로 partition하는 branch를 확인합니다.
- B의 모든 요소를 `pa`로 복원하는 loop와 각 pass 종료 시 B-empty 상태를 확인합니다.
- one group의 rotation 안정성과 zero group의 two reversals를 구체적 rank 예제로 코드 실행 순서와 연결합니다.
- logical command count와 array `memmove` physical cost가 다른 레이어에서 발생함을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — S
- **이 commit 직전 상태:** `160d1fb8d824`까지 size 0~5는 처리하지만 size 6 이상은 정렬되지 않았습니다. parser는 이미 ranks를 `0..n-1`로 제한하고 있었습니다.
- **해결하려던 문제:** 허용 operation만으로 임의 크기 input을 결정적으로 정렬하면서, 이미 처리한 낮은 bit 순서를 다음 높은 bit pass가 파괴하지 않아야 했습니다.
- **기존 설계가 충분하지 않았던 이유:** tiny permutation 분기를 일반 크기로 확장하면 상태 수가 factorial로 증가합니다. 단순히 bit가 0인 원소를 B에 push한 뒤 임의로 복원하면 group 내부 순서가 깨져 LSD radix의 귀납 조건이 성립하지 않습니다.
- **핵심 결정:** `1463a193a4f9:src/sort.c:radix_sort`는 `size - 1`에서 필요한 bit 수만 계산하고, 각 pass 시작의 `a->size`를 `round_size`에 고정합니다. 매 iteration의 현재 top bit가 1이면 `ra`, 0이면 `pb`하고, 정확히 `round_size`개를 처리한 뒤 B가 빌 때까지 `pa`합니다.

```c
/* 1463a193a4f9:src/sort.c:radix_sort의 핵심 분기 */
round_size = a->size;
while (index < round_size)
{
    if (((a->ranks[0] >> bit) & 1) != 0)
        op_ra(a, 1);
    else
        op_pb(a, b, 1);
    index++;
}
while (b->size > 0)
    op_pa(a, b, 1);
```

- **state / 불변 조건 / 소유권 / lifecycle 변화:** 새로운 allocation이나 소유권은 없습니다. 한 pass에서 bit 1 원소는 A 내부 rotate 순서로 유지됩니다. bit 0 원소는 `pb`로 B에 들어갈 때 한 번 역순이 되고, B 전체를 `pa`할 때 다시 역순이 되어 원래 상대 순서를 회복합니다. pass 종료마다 B는 empty이고 A는 해당 bit까지 stable-sorted 상태입니다.
- **failure scenario:** loop 조건을 현재 `a->size`로 두면 `pb` 때 size가 줄어 일부 시작 원소를 검사하지 못합니다. zero group을 한 번만 reverse하거나 B를 완전히 비우지 않으면 낮은 bit 정렬이 무너집니다. bit 수를 값의 32비트로 잡으면 불필요한 pass가 생기고, 너무 적게 잡으면 최대 rank를 구분하지 못합니다.
- **이 commit이 보장하는 것:** 유효한 dense unique ranks와 정상 operation을 전제로 size>5에서 bit별 stable partition을 수행하고, 마지막에 B empty/A rank ascending 상태에 도달하는 algorithm을 제공합니다. command 수는 각 bit마다 A 검사 `n`회와 zero 복원 최대 `n`회이므로 `Θ(n log n)`입니다.
- **아직 보장하지 않는 것:** 이 SHA 자체에는 독립 correctness, command budget, resource movement, I/O failure 증거가 없습니다. 배열 기반 `pb`/`pa`/rotate는 각 command 내부에서 `memmove`하므로 logical command complexity가 곧 CPU memory movement complexity는 아닙니다.
- **후속 fix/test:** `5b7559278909`가 Python list replay와 product checker로 tiny/radix 결과를 검증하고, `a16dde75d935`·`23198a9cdd55`가 deterministic command budget을, `6569949742eb`가 physical pair movement를 별도 계측합니다. write failure propagation은 `315f4b91779b`에서 sorter return path 전체에 추가됩니다.
- **개발 흐름의 다음 관련 커밋:** `cf07495c97f7`은 parser가 만든 ranked A와 auxiliary B를 어떤 ownership/cleanup 순서로 이 sorter에 연결하는가?

### `cf07495c97f7` — feat(push_swap): 정렬 명령 생성 흐름을 연결
- **중요도:** B
- **태그:** CORE, INTEGRATION
- **원자료에서 확인된 역할:** Integrates parsing, B allocation, sorting, emission, and cleanup into `push_swap`.
- **커밋 분류 요약:** Links parsing, auxiliary-stack allocation, sorting, cleanup, and the `push_swap` executable.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only cf07495c97f7`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `cf07495c97f7` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `main`의 parse A → allocate B → `sort_stack` → free A/B 순서를 정상 처리에서 추적합니다.
- parse failure와 B allocation failure에서 canonical error, 종료 상태, 이미 소유한 A cleanup을 확인합니다.
- empty/already-sorted input도 동일한 stack 수명 cleanup을 거치는지 확인합니다.
- common objects와 generator-specific control flow가 Makefile에서 어떻게 링크되는지 확인합니다.
- 이 SHA의 output helper가 write failure를 아직 반환하지 않는다는 한계를 실제 API signature/caller에서 확인하고, 후속 `315f4b91779b`와 연결합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** model, parser, operation, sorter는 개별 모듈로 존재하지만 generator executable의 top-level 수명이 없었습니다.
- **이 commit의 구현 역할:** `cf07495c97f7:src/push_swap.c:main`이 A를 parse하고 A capacity와 같은 empty B를 할당한 뒤 `sort_stack(&a,&b)`을 호출하고 두 stack을 해제합니다. Makefile은 common objects와 generator main을 `push_swap`으로 링크합니다.
- **핵심 상태 전이 또는 boundary:** parse 실패는 아직 stack 소유권이 caller에 확정되지 않은 오류이고, B allocation 실패는 이미 소유한 A를 해제한 뒤 `Error\n`과 status 1로 종료합니다. 정상·empty·already-sorted 모두 최종적으로 A/B를 free합니다.
- **failure/no-op/edge:** 이 SHA의 `sort_stack`과 `ps_putstr_fd`는 `void`입니다. 따라서 private stack이 정렬됐더라도 stdout write가 실패한 사실을 main이 알 수 없고 status 0을 반환할 수 있습니다.
- **이후 연결:** Thread 4가 executable output의 correctness/cost를 검증하고, `315f4b91779b`가 output failure를 operation→sorter→main 끝까지 전파합니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| tiny sort 후 A 정렬 / B 최종 비움 | caa54cb306ad | 160d1fb8d824 | - | - | Thread 4의 5b7559278909에서 exhaustive small-state 검증 | 2~3개는 direct cases, 4~5개는 successive minima를 B에 격리한 뒤 3개 정렬과 역순 `pa` 복원으로 완료됩니다. |
| radix pass의 stable partition | 1463a193a4f9 | - | - | - | 5b7559278909 및 후속 deterministic sort tests | one group은 `ra` 순서를 유지하고 zero group은 `pb`/전체 `pa`의 두 번 reverse로 순서를 회복합니다. |
| 각 radix bit 종료 시 B empty | 1463a193a4f9 | - | - | - | 독립 replay에서 final B empty를 확인 | 매 round가 `while (b->size > 0) pa`로 끝나므로 다음 bit는 전체 원소가 A에 있는 동일한 시작 형태를 사용합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| tiny direct case로 일반 입력을 처리할 수 없음 | 1463a193a4f9의 general radix mechanism | - | Thread 4의 independent replay | factorial case expansion 대신 dense rank의 유한 bit를 순서대로 stable partition합니다. |
| zero-bit group의 상대 순서가 깨져 lower-bit ordering 손실 | push-to-B 후 전체 `pa`로 두 번 reverse되는 stable partition | - | 5b7559278909 | `pb`만 보면 zero group이 역순이지만, 모두 `pa`하면 두 번째 reverse로 원순서가 복원됩니다. |
| B allocation 또는 parse 실패 | cf07495c97f7의 main cleanup flow | - | 후속 fault-injection Thread 6 | parse 실패는 error 종료, B 실패는 이미 소유한 A를 먼저 free합니다. Nth-allocation sweep이 후속 검증합니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| stack A | ranked input은 존재하지만 executable owner 없음 | cf07495c97f7 | main이 parse 성공 후 소유하고 sort 뒤 항상 해제 | `cf07495c97f7:src/push_swap.c:main` |
| auxiliary stack B | tiny extension의 caller가 전달할 대상으로만 존재 | cf07495c97f7 | main이 A capacity로 할당하고 sorter가 임시 원소를 보관하며 종료 시 해제 | `cf07495c97f7:src/push_swap.c:main` |
| sort dispatcher | size<=3 범위부터 시작 | 160d1fb8d824, 1463a193a4f9 | size<=5 tiny, 그 이상 radix로 선택 | `1463a193a4f9:src/sort.c:sort_stack` |
| operation wrapper / stdout command stream | operation은 emit 가능하지만 top-level consumer 없음 | cf07495c97f7 | sorter가 `emit=1` wrapper로 state mutation과 command stream을 동시에 생성 | `cf07495c97f7:src/sort.c`, `src/operations.c` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** ranked A가 size에 따라 direct tiny 또는 stable LSD radix 경로로 들어갑니다. tiny는 최대 두 최소 rank를 B에 격리해 3개 정렬을 재사용하고, radix는 bit마다 시작 A size를 고정해 `ra`/`pb`로 stable partition한 뒤 B를 모두 `pa`합니다. `push_swap` main은 A/B를 소유하고 sorter가 emitting wrappers로 명령을 생성한 뒤 두 stack을 정리합니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** `cf07495c97f7` 시점에는 write 결과가 무시되어 완전한 external stream 전달을 성공 조건으로 삼지 못합니다. correctness independence, deterministic cost, memory movement, sanitizer는 Thread 4, write failure는 Thread 6이 담당합니다. 이 환경에서는 tests를 실행하지 않았으며 해당 SHA의 코드와 test 설계만 확인했습니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: `ranked A → size 기반 tiny/radix 선택 → shared operations로 state mutation + emission → B empty / A sorted → main cleanup`
- **학습자 최종 flow:** `e09cf45e21cd`가 만든 dense-ranked A → `1463a193a4f9:sort_stack`이 size<=5면 `sort_tiny`, 그 이상이면 `radix_sort` 선택 → sorter가 `op_*`를 `emit=1`로 호출해 A/B state와 stdout stream을 함께 진행 → tiny 복원 또는 각 radix pass 종료에서 B를 비우고 최종 A를 정렬 → `cf07495c97f7:main`이 A/B를 해제합니다.
- **실제 코드 삽입:** general decision은 위 `round_size` 고정, top bit에 따른 `ra`/`pb`, B 전체 `pa` 구문입니다. 이는 현재 size가 줄어도 round 시작 원소를 정확히 한 번씩 처리하고 zero group을 두 번 reverse합니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# 독립적인 정확성 검증과 비용 근거

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** Verification deliberately broadens from functional outcomes to independence, reproducibility, and cost. The Python model reduces common-mode risk from the product checker, deterministic fixtures make regressions comparable, resource instrumentation exposes the array representation's hidden movement cost, and sanitizers cover invalid memory behavior that explicit assertions may miss.
- **학습 목표:** product checker와 공유 구현만으로는 잡기 어려운 common-mode risk를 independent replay로 줄이고, correctness뿐 아니라 determinism, command cost, movement cost, allocation cost, sanitizer evidence를 분리해 기록합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- Python replay model은 C product code와 무엇을 공유하지 않아 독립 oracle이 되는가?
- tiny exhaustive permutations와 larger fixed cases가 각각 어떤 영역을 커버하는가?
- command budget이 wall-clock time 대신 사용되는 이유와 검증 순서는 무엇입니까?
- specified PRNG/permutation generator가 reproducibility와 stream determinism을 어떻게 고정하는가?
- 6569949742eb의 command/movement/peak-allocation metric이 서로 다른 어떤 비용을 나타내는가?
- ASan/UBSan 경로가 fault/resource suite를 대체하지 않는 이유는 무엇입니까?

## 3. 완료 기준
- 독립 Python interpreter의 11개 command semantics와 final predicate를 production C와 나란히 비교했습니다.
- 각 test layer가 증명하는 것과 증명하지 않는 것을 구분해 기록했습니다.
- deterministic fixture 생성기와 repeated-stream equality를 직접 추적했습니다.
- resource hooks가 normal build에서 no-op이고 fault build에서만 측정되는 경계를 확인했습니다.
- sanitizer build의 object-tree 분리와 실행 대상 범위를 확인했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `5b7559278909` | test(sort): 생성 명령의 정렬 결과를 독립 검증 | A | TEST, SORT, RISK | Adds an independent Python command interpreter and exhaustive permutations through size five. |
| 2 | `a16dde75d935` | test(sort): 큰 입력의 명령 수 상한을 검증 | B | TEST, PERF, SORT | Adds command-count ceilings for large deterministic cases. |
| 3 | `23198a9cdd55` | test(sort): 결정적 다중 시드 동치 검사를 추가 | B | TEST, SORT | Replaces ambient randomness with a specified permutation generator and checks repeated-stream determinism. |
| 4 | `6569949742eb` | test(resource): 명령과 배열 이동 및 할당량을 기준화 | A | TEST, RESOURCE, PERF | Separately baselines emitted commands, logical pair movements, peak project allocation, and final cleanup. |
| 5 | `5505adf3e469` | build(sanitize): C99 sanitizer 검증 경로를 추가 | B | TEST, RUNTIME, PRACTICAL | Runs the operation and functional suites against isolated ASan/UBSan builds. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Critical 불변 조건**
  - Resource metrics count only successfully emitted commands and remain reproducible for the fixed deterministic fixtures.
- **Major engineering difficulties**
  - Sharing operation semantics between generator and checker without allowing that sharing to become the sole correctness oracle.
  - Instrumenting allocation, operation, and movement costs in C while preserving normal-build behavior and separating logical command cost from physical array movement.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `5b7559278909` — test(sort): 생성 명령의 정렬 결과를 독립 검증
- **중요도:** A
- **태그:** TEST, SORT, RISK
- **원자료에서 확인된 역할:** Adds an independent Python command interpreter and exhaustive permutations through size five.
- **커밋 분류 요약:** Replays emitted commands in an independent Python model and exhausts all permutations through size five.

#### 원자료에서 확인된 맥락
- **문제:** The product checker and generator share the C operation implementation. A defect in push or rotation semantics could therefore cause both programs to accept the same incorrect behavior.
- **결정:** Implement the eleven commands again using Python lists, reject unknown emitted commands, replay every stream independently, require sorted A and empty B, and still pass the same stream through the product checker.
- **중요한 이유:** The two representations fail differently. Exhausting every permutation for sizes two through five strongly covers the direct sorting branches, while fixed larger cases exercise the radix path. The checker remains valuable for integration, but it is no longer the only oracle.
- **변경 내용:** The test suite adds command parsing, an independent A/B state model, fixed cases including integer extremes, the no-command already-sorted case, and all 152 permutations for sizes two through five.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 5b7559278909`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `5b7559278909` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- Python list 기반 A/B model에서 11개 command를 각각 구현한 부분을 확인합니다.
- unknown emitted command를 reject하고 final A == Python numeric sort, B empty를 검사하는 path를 확인합니다.
- 같은 stream을 product checker에도 전달하는 second-oracle 흐름을 확인합니다.
- fixed cases와 sizes 2..5의 모든 permutation 152개를 구성하는 loop를 확인합니다.
- already sorted input에서 zero command를 요구하는 assertion을 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** `push_swap`이 출력한 각 줄은 11개 합법 command 중 하나이고, 독립 replay 후 A가 Python의 numeric sort 결과와 같고 B가 비어 있어야 합니다. 같은 stream을 product checker도 `OK`로 판정해야 합니다.
- **재현하는 failure/boundary:** integer extremes를 포함한 fixed cases, already-sorted five-element zero-command case, radix 경로의 size 6/10, size 2~5의 모든 permutation 152개를 사용합니다.
- **test technique:** independent Python list replay + exhaustive small-state enumeration + product checker CLI integration입니다.
- **통과하는 production path:** test subprocess가 `push_swap` 실행 → stdout command split/validation → Python `apply_moves` → final A/B assertion → 같은 stream을 checker stdin으로 전달합니다.
- **이 테스트가 증명하는 것:** 나열한 모든 input에서 emitted vocabulary, independent final state, B-empty, product checker integration을 함께 확인합니다. tiny sorter의 전체 입력 상태 공간을 size 5까지 exhaust합니다.
- **이 테스트가 증명하지 않는 것:** size 6 이상 전체 상태, 최적 command 수, allocation/read/write failure, 정의되지 않은 동작 부재는 증명하지 않습니다. Python 구현 자체의 결함 가능성도 0은 아니지만 C shared-operation common-mode risk는 크게 줄입니다.
- **성격:** exhaustive small-state evidence와 broad dual-oracle integration evidence를 결합한 regression입니다.
- **막는 후속 회귀:** generator와 checker가 같은 잘못된 rotate/push를 공유해도 Python final state가 실패하며, tiny branch 누락·unknown command·sorted input 불필요 출력도 잡습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** parser/sorter/checker와 operation tests는 있었지만 generator와 checker가 같은 C operation semantics를 공유해 공통 결함을 함께 수용할 수 있었습니다.
- **주요 boundary/decision:** product C와 자료구조를 공유하지 않는 Python list A/B model을 새 oracle로 두고, product checker는 별도 integration oracle로 유지했습니다.
- **state / 소유권 / failure 변화:** production code에는 변화가 없습니다. test harness가 command list를 소유하고 unknown line을 즉시 실패시키며 두 oracle에 순차 전달합니다.
- **보장 / 비보장:** tiny 전체 permutation과 selected radix cases의 functional correctness를 강하게 증명하지만, 일반 크기의 완전성·performance·resource·runtime fault는 남습니다.
- **후속 검증 또는 수정 연결:** `a16dde75d935`가 큰 입력의 command ceiling, `23198a9cdd55`가 specified fixture와 stream determinism, `6569949742eb`가 movement/allocation, `5505adf3e469`가 sanitizer evidence를 추가합니다.
- **개발 흐름의 다음 관련 커밋:** `a16dde75d935`는 correctness를 먼저 확인한 뒤 command count를 평가함으로써 빠르지만 잘못된 stream을 어떻게 배제하는가?

### `a16dde75d935` — test(sort): 큰 입력의 명령 수 상한을 검증
- **중요도:** B
- **태그:** TEST, PERF, SORT
- **원자료에서 확인된 역할:** Adds command-count ceilings for large deterministic cases.
- **커밋 분류 요약:** Adds reproducible command-count ceilings for 100- and 500-element inputs.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only a16dde75d935`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `a16dde75d935` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- 100/500 unique value fixture와 fixed seed 생성 방식을 확인합니다.
- command budget 비교 전에 independent correctness helper가 먼저 stream correctness를 검증하는 순서를 확인합니다.
- configured command-count ceiling과 실제 emitted line 수 비교 위치를 확인합니다.
- wall-clock time이 pass criterion에 사용되지 않는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** 100/500 unique input에서 stream이 먼저 정확히 정렬되고, 그 뒤 명령줄 수가 각각 설정된 ceiling 이하이어야 합니다.
- **재현하는 failure/boundary:** Python `random.seed(4242)`와 unique sample로 크기 100, 500 fixture를 고정합니다.
- **test technique:** deterministic large-case command-budget regression이며 correctness helper를 선행합니다.
- **통과하는 production path:** fixture 생성 → `assert_sorted_by_program`의 independent replay/checker → stdout line count → ceiling 비교입니다.
- **이 테스트가 증명하는 것:** 해당 두 fixture에서 정확성과 command-count 상한을 함께 확인합니다. wall-clock 변동에 영향받지 않습니다.
- **이 테스트가 증명하지 않는 것:** 모든 100/500 permutation의 worst case, CPU 시간, array movement, memory, 다른 Python 구현의 random algorithm 차이까지 고정하지는 않습니다.
- **성격:** fixed-seed deterministic performance regression입니다.
- **막는 후속 회귀:** correctness는 유지하지만 불필요한 pass/command가 크게 늘어 ceiling을 넘는 변경을 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** correctness evidence는 있었지만 large input command growth를 수치로 제한하지 않았습니다.
- **이 commit의 구현 역할:** fixed-seed 100/500 fixture와 1500/8000 command ceiling을 추가합니다.
- **핵심 상태 전이 또는 boundary:** budget assertion 전에 independent correctness가 완료되므로 잘못된 짧은 stream은 performance 성공으로 계산되지 않습니다.
- **failure/no-op/edge:** elapsed wall-clock은 pass/fail에 쓰지 않습니다. 이 SHA의 fixture reproducibility는 Python `random` 구현에 기대며 generator specification 자체는 아직 문서화되지 않았습니다.
- **이후 연결:** `23198a9cdd55`가 명시적 32-bit PRNG/Fisher–Yates로 fixture 생성 규칙을 고정하고 seed를 늘립니다.
- **개발 흐름의 다음 관련 커밋:** `23198a9cdd55`는 Python ambient randomness 대신 어떤 명시적 상태 전이와 permutation 규칙을 사용하며 동일 input의 stream equality를 어떻게 확인하는가?

### `23198a9cdd55` — test(sort): 결정적 다중 시드 동치 검사를 추가
- **중요도:** B
- **태그:** TEST, SORT
- **원자료에서 확인된 역할:** Replaces ambient randomness with a specified permutation generator and checks repeated-stream determinism.
- **커밋 분류 요약:** Uses an explicit deterministic permutation generator and checks repeatable streams across seeds and sizes.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 23198a9cdd55`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `23198a9cdd55` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- specified 32-bit LCG state update와 Fisher–Yates permutation 구현을 확인합니다.
- permutation 값을 unique signed integers로 변환하는 규칙을 확인합니다.
- tiny/radix boundary 양쪽 size와 multiple seed coverage를 확인합니다.
- 같은 fixture를 두 번 실행해 command list equality를 검사하는 assertion을 확인합니다.
- 100/500 budgets가 세 seed로 확장되는 부분과 executable path env override를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** 동일한 입력은 동일한 command list를 생성하며, 여러 seed/size에서 독립 replay correctness와 budget을 유지해야 합니다.
- **재현하는 failure/boundary:** seed 1, 7, 97, 4242, 9001과 size 2,3,5,6,17,64로 tiny/radix 경계를 가로지릅니다. budget은 size 100/500에 seed 7,4242,9001을 사용합니다.
- **test technique:** specified deterministic fixture generation, repeated-run stream equality, multi-seed CLI regression입니다.
- **통과하는 production path:** 32-bit LCG → Fisher–Yates permutation → `value * 37 - size * 23` unique signed values → generator 두 번 실행 → command list equality와 independent replay/checker입니다.
- **이 테스트가 증명하는 것:** 같은 generator specification으로 fixture가 재현되고 동일 input의 command sequence가 반복 실행에서 정확히 같으며 여러 경계 size가 정렬됩니다.
- **이 테스트가 증명하지 않는 것:** process scheduling이나 환경과 무관한 모든 형태의 determinism, 모든 seed, 최적 sequence는 증명하지 않습니다.
- **성격:** deterministic equivalence 및 multi-seed regression입니다.
- **막는 후속 회귀:** uninitialized state나 ambient random에 의한 command variation, 특정 seed/boundary size에만 나타나는 sorter 오류를 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** fixed seed는 있었지만 fixture algorithm이 Python library 동작에 기대고 같은 input의 stream equality도 직접 검사하지 않았습니다.
- **이 commit의 구현 역할:** `tests/run_tests.py:deterministic_values`에 다음 명시적 update를 두고 executable path도 environment override로 바꿉니다.

```python
# 23198a9cdd55:tests/run_tests.py:deterministic_values
state = (1664525 * state + 1013904223) & 0xFFFFFFFF
```

- **핵심 상태 전이 또는 boundary:** Fisher–Yates가 `0..size-1` permutation을 만들고 affine mapping이 uniqueness를 유지한 signed values를 만듭니다. 같은 argv를 두 번 실행해 parsed command list 자체를 비교합니다.
- **failure/no-op/edge:** size 5/6이 tiny/radix dispatch 경계입니다. path override는 후속 sanitizer binaries에 같은 functional suite를 재사용할 기반이 됩니다.
- **이후 연결:** `6569949742eb`이 동일 deterministic cases를 resource baseline으로 확장하고, `5505adf3e469`가 path override로 sanitizer 실행 파일을 사용합니다.
- **개발 흐름의 다음 관련 커밋:** `6569949742eb`은 emitted command 수와 array pair movement 및 peak requested bytes를 어떤 서로 다른 hook에서 기록하는가?

### `6569949742eb` — test(resource): 명령과 배열 이동 및 할당량을 기준화
- **중요도:** A
- **태그:** TEST, RESOURCE, PERF
- **원자료에서 확인된 역할:** Separately 기준 상태 emitted commands, logical pair movements, peak project allocation, and final cleanup.
- **커밋 분류 요약:** Instruments successful commands, logical pair movements, peak project allocation, and deterministic resource 기준 상태.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 6569949742eb`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `6569949742eb` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- fault build에서 command count가 text emission 성공 뒤에만 증가하는 hook 위치를 확인합니다.
- array operation에서 logical value-rank pair movement/rewrite를 세는 instrumentation 지점을 확인합니다.
- current/peak requested allocation bytes가 instrumentation header를 제외해 계산되는지 확인합니다.
- versioned JSON 기준 상태의 10/100/500 × 3 seed case, exact command count, movement/peak upper bound를 확인합니다.
- zero live allocations와 recorded operation count == emitted 명령줄 count 검증을 확인합니다.
- normal build에서 instrumentation hooks가 no-op으로 컴파일되는 경계를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** counter는 성공적으로 출력된 command만 세고, fixed fixture의 command 수는 exact 기준 상태와 같으며, pair movement·peak requested bytes는 upper bound 이내이고 종료 시 live allocation은 0이어야 합니다.
- **재현하는 failure/boundary:** versioned JSON에 size 10/100/500 × seed 7/4242/9001을 고정합니다. exact command 수는 각각 65/1084/6784이며 movement 상한은 650/105000/3200000, peak bytes 상한은 160/1600/8000입니다.
- **test technique:** compile-time instrumentation + deterministic resource 기준 상태 + subprocess report parsing입니다.
- **통과하는 production path:** fault-build generator → successful `ps_putstr_fd` 뒤 `ps_record_operation` → operation primitive의 `ps_record_movements` → `ps_malloc` current/peak tracking → exit `ps_test_finish` report → Python 기준 상태 comparison입니다.
- **이 테스트가 증명하는 것:** selected fixtures에서 emitted line 수와 recorded operation 수가 같고 exact command determinism, movement/peak ceiling, zero live allocation을 확인합니다.
- **이 테스트가 증명하지 않는 것:** libc 내부 allocation, actual bytes copied by libc, CPU time, 모든 input worst case, normal build binary의 instrumentation 없는 runtime 동작을 직접 측정하지 않습니다.
- **성격:** deterministic resource/performance regression입니다.
- **막는 후속 회귀:** command counter가 실패 전 증가하는 변경, array movement 급증, project allocation peak 증가, cleanup 누락, fixture stream 변화가 기준 상태를 깨뜨립니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** 명령줄 수만으로는 array-backed operation의 `memmove` 비용과 memory peak가 보이지 않았습니다.
- **주요 boundary/decision:** logical command, logical value-rank pair rewrite, project-requested allocation bytes를 서로 다른 metric으로 분리하고 fault build에만 hook을 활성화했습니다.
- **state / 소유권 / failure 변화:** normal build에서는 hook macro가 no-op이라 production semantics를 바꾸지 않습니다. fault build allocator header가 requested size를 보관하되 peak 계산에서는 header를 제외하고 caller가 요청한 bytes만 사용합니다.
- **보장 / 비보장:** fixed cases에서 재현 가능한 relative cost를 제공합니다. movement metric은 실제 CPU/cache cost가 아니라 구현이 정의한 pair 이동/재작성 횟수입니다.
- **후속 검증 또는 수정 연결:** `5505adf3e469`의 sanitizer는 invalid memory/UB를 다루지만 baseline cost나 deterministic fault cleanup을 대체하지 않습니다. Thread 6의 I/O fault tests는 write 실패에서 counter와 cleanup 의미를 보강합니다.
- **개발 흐름의 다음 관련 커밋:** `5505adf3e469`는 normal/fault objects와 섞이지 않는 별도 object tree에서 어떤 executable/test를 ASan/UBSan으로 재실행하는가?

### `5505adf3e469` — build(sanitize): C99 sanitizer 검증 경로를 추가
- **중요도:** B
- **태그:** TEST, RUNTIME, PRACTICAL
- **원자료에서 확인된 역할:** Runs the operation and functional suites against isolated ASan/UBSan builds.
- **커밋 분류 요약:** Builds separate ASan and UBSan binaries and runs operation and functional suites against them.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 5505adf3e469`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `5505adf3e469` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- Makefile의 sanitizer 전용 object directory와 normal object reuse 방지 구조를 확인합니다.
- ASan/UBSan compile/link flags, debug info, optimization, frame-pointer 설정을 확인합니다.
- 두 executable과 C operation-불변 조건 test가 sanitizer target에 포함되는지 확인합니다.
- configurable executable paths로 full Python functional suite를 sanitizer binaries에 재사용하는 흐름을 확인합니다.
- fault/resource suites와 sanitizer가 서로 다른 defect class를 다룬다는 실행 경계를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** functional, deterministic, resource evidence는 있었지만 ASan/UBSan instrumented object를 분리해 build/run하는 경로가 없었습니다.
- **이 commit의 구현 역할:** Makefile이 `.build/sanitize` 전용 object tree와 `-O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined` compile/link flags를 사용해 sanitized `push_swap`, `checker`, operation 불변 조건 test를 만듭니다.
- **핵심 상태 전이 또는 boundary:** sanitizer target은 operation C test를 실행하고, `PS_PUSH_SWAP`/`PS_CHECKER` environment override로 full Python functional suite를 sanitized executables에 재사용합니다. normal object와 섞이지 않습니다.
- **failure/no-op/edge:** sanitizer target은 fault-injection 및 resource suite를 대신 실행하지 않습니다. ASan/UBSan은 invalid access/UB 관찰이고 deterministic injected failure·metric 기준 상태는 별도 build의 책임입니다.
- **이후 연결:** 이 개발 흐름의 최종 evidence stack은 independent replay, deterministic budgets/resources, sanitizer가 서로 다른 결함 종류를 담당하는 형태입니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| product checker가 유일한 correctness oracle이 아님 | 5b7559278909 | 23198a9cdd55에서 deterministic repetition 강화 | - | - | Python replay + product checker의 dual evidence | Python list model이 11개 semantics와 final predicate를 독립 구현하고 checker는 integration oracle로 병행됩니다. |
| fixed fixtures에서 resource metrics reproducible | 6569949742eb | - | - | - | versioned JSON baseline | specified generator와 JSON의 exact command/upper bounds를 사용하며 operation count는 emitted line 수와 비교됩니다. |
| sanitizer instrumentation과 normal objects 분리 | 5505adf3e469 | - | - | - | sanitizer target 실행 | `.build/sanitize` 전용 objects와 flags를 사용해 normal/fault objects의 flag 혼합을 방지합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| generator와 checker가 같은 잘못된 operation semantics를 공유 | 5b7559278909의 independent Python model | - | 5b7559278909 자체가 regression evidence | shared C implementation만 oracle로 사용한 common-mode risk를 자료구조와 코드가 다른 Python replay로 낮춥니다. |
| ambient randomness로 fixture/stream 비교가 불안정 | 23198a9cdd55의 specified generator | - | 반복 실행 command-list equality | 32-bit LCG와 Fisher–Yates를 test code에 직접 규정하고 같은 argv를 두 번 실행해 stream을 비교합니다. |
| command count만 보면 array movement/memory trade-off가 숨음 | 6569949742eb의 분리 instrumentation | - | versioned resource baseline | output success 뒤 command counter, primitive별 pair movement, allocator requested bytes를 별도 metric으로 둡니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| C generator/checker product code | shared operation만으로 상호 검증 | 5b7559278909 이후 test layers | product code는 그대로이고 외부 독립·resource·sanitizer oracle이 둘러쌈 | `5b7559278909:tests/run_tests.py` |
| independent Python A/B model | 없음 | 5b7559278909 | command list를 독립 replay하고 sorted A/B-empty를 판정 | `5b7559278909:tests/run_tests.py:apply_moves` |
| fault-build instrumentation counters | allocation fault 기본 counter만 존재 | 6569949742eb | command, movement, current/peak bytes, live allocation을 보고 | `6569949742eb:src/runtime.c`, `src/operations.c` |
| resource baseline JSON | 없음 | 6569949742eb | versioned deterministic cases와 exact/upper bounds 소유 | `6569949742eb:tests/resource_baseline.json` |
| sanitizer object tree | 없음 | 5505adf3e469 | normal/fault와 분리된 ASan/UBSan objects와 binaries | `5505adf3e469:Makefile` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** emitted stream은 Python list model과 product checker 두 경로로 검증되고, tiny size 2~5는 152개 permutation을 exhaust합니다. specified LCG/Fisher–Yates fixture는 repeated command equality와 multi-seed budget을 고정합니다. fault build는 성공 command, pair movement, project allocation peak, final live count를 versioned JSON과 비교하며, 별도 sanitizer object tree가 operation/functional suite를 ASan/UBSan binaries로 재실행하도록 구성됩니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** 각 layer의 증명 범위는 교환 불가능합니다. sanitizer는 deterministic fault injection이나 resource ceiling을 대체하지 않고, selected deterministic cases는 모든 input의 worst case를 증명하지 않습니다. 이 환경에서는 GitHub source checkout이 불가능해 targets를 실제 실행하지 않았으므로 문서의 결과 설명은 test 코드의 assertion/expected 기준 상태를 정적으로 확인한 것이며 새 runtime evidence가 아닙니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: `emitted stream → independent Python replay + checker → deterministic fixture repetition → command/resource baseline → ASan/UBSan functional replay`
- **학습자 최종 flow:** `5b7559278909:run_tests.py`가 generator stream을 legal command로 제한하고 Python A/B에 replay한 뒤 checker에도 전달합니다 → `23198a9cdd55:deterministic_values`가 fixture와 두 번의 stream equality를 고정합니다 → `6569949742eb` fault build가 output-success command, primitive movement, requested allocation metrics를 report하고 JSON과 비교합니다 → `5505adf3e469:Makefile` sanitizer target이 별도 binaries로 C operation test와 같은 Python functional suite를 실행하도록 연결합니다.
- **실제 코드 삽입:** 독립성의 핵심은 Python list operation이고, 재현성의 핵심은 `state = (1664525 * state + 1013904223) & 0xFFFFFFFF`입니다. resource 측정은 command 출력 성공 뒤에만 operation counter를 증가시키는 hook 배치로 external stream과 metric을 일치시킵니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# 체커 입력 규칙과 판정 강화

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** The visible progression demonstrates an intermediate implementation becoming too general for a tiny fixed protocol. The correction moves the limit into the reader, where hostile or malformed input can be rejected before dispatch, and the fault tests then distinguish valid EOF framing, transient interruption, permanent transport failure, and protocol invalidity.
- **학습 목표:** checker가 command stream을 읽고 silent replay한 뒤 `OK`/`KO`를 판정하는 lifecycle을 복원하고, 초기 general line reader가 protocol-specific bounded reader로 교정되는 실패→수정→검증 과정을 추적합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- 초기 reader의 tri-state `1/0/-1` 계약과 frame 소유권은 어떻게 구현됩니까?
- command dispatch가 exact string match와 `emit = 0`을 통해 shared operations를 어떻게 재사용하는가?
- `OK`, `KO`, malformed stream error의 process/output semantics는 어떻게 다른가?
- 왜 arbitrarily long line reader가 최대 3-byte command protocol에는 과도한 추상화였는가?
- `EINTR`, permanent read failure, NUL, overlength, EOF-delimited final frame이 reader에서 어떻게 구분됩니까?

## 3. 완료 기준
- reader → dispatch → state mutation → verdict의 실제 caller/callee 경로를 해당 SHA에서 추적했습니다.
- no-values 실행이 stdin을 읽지 않는 경로를 확인했습니다.
- 0b87adebca2b와 7713a31cf502를 비교해 dynamic growth가 fixed frame으로 바뀐 지점을 설명할 수 있습니다.
- dbf76e147e68에서 EIO/EINTR와 protocol-invalid cases가 각각 어떤 production path를 통과하는지 확인했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `0b87adebca2b` | feat(checker): 표준 입력 명령 프레임을 읽음 | B | CHECKER, CORE | Adds an initial general-purpose dynamic line reader. |
| 2 | `f79ae7e86592` | feat(checker): 스택 연산 명령을 해석 | B | CHECKER, INTEGRATION | Dispatches legal command names to shared silent operations. |
| 3 | `d906f4d86528` | feat(checker): 명령 실행 결과를 판정 | A | CHECKER, CORE, INTEGRATION | Establishes the complete checker lifecycle and `OK`/`KO` protocol. |
| 4 | `44ee0830e9f0` | test(checker): 명령 연산과 최종 판정을 검증 | B | TEST, CHECKER | Verifies every command family and the distinction among verdicts and malformed streams. |
| 5 | `7713a31cf502` | fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도 | A | CHECKER, RUNTIME, RISK | Replaces unbounded lines with protocol-sized frames and retries interrupted reads. |
| 6 | `dbf76e147e68` | test(checker): 읽기 실패와 명령 경계를 검증 | A | TEST, CHECKER, RISK | Injects read faults and verifies NUL, empty, overlength, long-stream, and EOF-delimited boundaries. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Critical 불변 조건**
  - Checker returns `OK` only for sorted A with empty B; `KO` is a normal verdict, whereas malformed input, malformed commands, allocation failure, and I/O failure are errors.
  - Checker frames contain at most three non-newline bytes, contain no NUL, retry `read` interrupted by `EINTR`, and reject other read errors.
- **Major engineering difficulties**
  - Handling allocation failure, interrupted reads and writes, short writes, zero-byte writes, closed pipes, and already-visible output prefixes without transactional rollback.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `0b87adebca2b` — feat(checker): 표준 입력 명령 프레임을 읽음
- **중요도:** B
- **태그:** CHECKER, CORE
- **원자료에서 확인된 역할:** Adds an initial general-purpose dynamic line reader.
- **커밋 분류 요약:** Introduces a tri-state dynamically growing line reader for checker command frames.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 0b87adebca2b`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `0b87adebca2b` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- line reader의 tri-state `1`/`0`/`-1` return contract와 caller의 소유권 전달을 확인합니다.
- newline을 제외한 frame 반환과 EOF의 final unterminated non-empty frame acceptance를 확인합니다.
- buffer geometric growth와 allocation failure/read failure cleanup을 추적합니다.
- clean EOF with zero bytes에서 allocation이 caller에 남지 않는지 확인합니다.
- 이 SHA에서는 arbitrary length가 가능하고 `EINTR`도 failure로 처리되는 위치를 기록합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** checker가 stdin command를 frame 단위로 읽는 API가 없었습니다.
- **이 commit의 구현 역할:** `0b87adebca2b:src/checker_reader.c:read_next_line`이 한 바이트씩 읽어 newline을 제외한 NUL-terminated heap string을 반환합니다. `grow_line`은 초기 32바이트에서 필요한 길이보다 커질 때까지 두 배로 늘립니다.
- **핵심 상태 전이 또는 boundary:** 반환값 1이면 caller가 `*line`을 소유하고, 0이면 clean EOF이며 buffer는 남지 않고, -1이면 allocation/read failure로 내부 buffer를 해제합니다. EOF 전에 읽은 non-empty final frame은 newline 없이도 1로 반환합니다.
- **failure/no-op/edge:** read 실패는 errno를 구분하지 않고 -1이므로 `EINTR`도 permanent error가 됩니다. line 길이는 제한이 없고 embedded NUL도 저장 후 C string 비교에서 조기 종료될 수 있습니다.
- **이후 연결:** `f79ae7e86592`가 frame을 exact command로 dispatch하고, `7713a31cf502`가 reader를 최대 3바이트 protocol로 축소해 NUL/overlength와 EINTR를 직접 처리합니다.
- **개발 흐름의 다음 관련 커밋:** `f79ae7e86592`는 reader가 넘긴 문자열을 부분 일치 없이 11개 command로 매핑하고 어떻게 stdout을 억제하는가?

### `f79ae7e86592` — feat(checker): 스택 연산 명령을 해석
- **중요도:** B
- **태그:** CHECKER, INTEGRATION
- **원자료에서 확인된 역할:** Dispatches legal command names to shared silent operations.
- **커밋 분류 요약:** Maps all legal command names to the shared operation layer with emission disabled.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only f79ae7e86592`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `f79ae7e86592` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- 11개 exact command string과 operation wrapper mapping을 확인합니다.
- prefix/suffix/unknown name이 partial match 없이 reject되는 비교를 확인합니다.
- 각 dispatch가 `emit = 0`으로 shared operations를 호출해 stdout에 echo하지 않는지 확인합니다.
- insufficient stack no-op semantics가 command error로 바뀌지 않는지 shared operation 경로와 연결합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** reader는 arbitrary line을 반환하지만 그 문자열을 stack transition으로 해석하지 않았습니다.
- **이 commit의 구현 역할:** `f79ae7e86592:src/checker.c:apply_checker_command`가 `ps_strcmp`로 `sa`, `sb`, `ss`, `pa`, `pb`, `ra`, `rb`, `rr`, `rra`, `rrb`, `rrr`를 exact 비교하고 해당 `op_*`를 `emit=0`으로 호출합니다.
- **핵심 상태 전이 또는 boundary:** known command는 상태 전이 성공 여부가 아니라 protocol membership을 뜻하는 1을 반환합니다. shared primitive가 size 부족으로 no-op해도 command 자체는 합법이므로 1입니다. unknown/prefix/suffix는 0입니다.
- **failure/no-op/edge:** stdout에 command를 echo하지 않습니다. 이 시점 operation API는 fallible하지 않으므로 dispatch가 I/O failure를 다룰 필요도 없습니다.
- **이후 연결:** `d906f4d86528`이 read/apply loop와 final verdict를 연결합니다.
- **개발 흐름의 다음 관련 커밋:** `d906f4d86528`은 valid-but-unsorted stream과 malformed/read-failed stream을 status/output에서 어떻게 구분하는가?

### `d906f4d86528` — feat(checker): 명령 실행 결과를 판정
- **중요도:** A
- **태그:** CHECKER, CORE, INTEGRATION
- **원자료에서 확인된 역할:** Establishes the complete checker lifecycle and `OK`/`KO` protocol.
- **커밋 분류 요약:** Builds the checker executable, replays stdin commands, and emits `OK` or `KO` from complete state.

#### 원자료에서 확인된 맥락
- **문제:** A command generator alone cannot establish that a stream is legal and reaches the required terminal state. The validator must also distinguish a valid but insufficient stream from malformed input or execution failure.
- **결정:** Build a separate checker that parses the same initial values, replays stdin frames through shared silent operations, and emits `OK` only for sorted A with empty B; otherwise a valid completed stream receives `KO`.
- **중요한 이유:** The commit establishes the public validation protocol and a second consumer of the common model. It also makes `KO` a normal status-zero judgment while reserving non-zero status and `Error` for malformed input, commands, allocation, or reading.
- **변경 내용:** The Makefile gains the checker executable, and checker 제어 흐름 gains frame 소유권, command application, cleanup, complete-state evaluation, and verdict output.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only d906f4d86528`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `d906f4d86528` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- checker `main`에서 parse → B allocate → read frame → apply → frame free 반복 → EOF verdict → stack cleanup 순서를 추적합니다.
- invalid command/read failure/parse/allocation failure 각각의 cleanup + `Error` path를 확인합니다.
- complete-state predicate가 A sorted와 B empty를 동시에 요구하는 실제 호출을 확인합니다.
- `OK`와 `KO` 모두 normal status인 반면 malformed stream은 failure status인 분기를 확인합니다.
- no-values invocation이 stdin loop 전에 return하는 위치를 확인합니다.
- 이 SHA에서 verdict/error writes가 아직 checked status를 제공하지 않는 한계를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** reader와 dispatcher는 있었지만 executable lifecycle, frame 소유권 loop, final-state protocol이 없었습니다.
- **주요 boundary/decision:** `d906f4d86528:src/checker.c:read_and_apply`가 frame 반환값이 양수인 동안 apply→free를 반복하고, invalid command에서 현재 frame을 free한 뒤 failure를 반환합니다. `main`은 clean EOF에만 complete-state predicate를 평가합니다.
- **state / 소유권 / failure 변화:** parse 성공 후 main이 A를, B init 성공 후 A/B를 소유합니다. reader가 반환한 각 frame은 loop가 한 번만 free합니다. parse/B/read/command failure는 stack cleanup 뒤 `Error\n`과 status 1입니다. valid EOF 뒤 sorted A+B empty면 `OK\n`, 아니면 `KO\n`이며 둘 다 status 0입니다.
- **보장 / 비보장:** valid command stream의 정상 verdict와 malformed transport/protocol의 error 구분을 제공합니다. argc==1은 parse와 stdin read 전에 status 0으로 반환하므로 입력 stream을 소비하지 않습니다. verdict/error write 결과는 아직 확인하지 않습니다.
- **후속 검증 또는 수정 연결:** `44ee0830e9f0`이 command families와 OK/KO/error를 검증하고, `7713a31cf502`/`dbf76e147e68`이 reader protocol과 read faults를 수정·검증합니다. write status는 `315f4b91779b`에서 보강됩니다.
- **개발 흐름의 다음 관련 커밋:** `44ee0830e9f0`은 primitive unit test가 아니라 checker CLI를 통과해 세 public outcome을 어떤 case로 구분하는가?

### `44ee0830e9f0` — test(checker): 명령 연산과 최종 판정을 검증
- **중요도:** B
- **태그:** TEST, CHECKER
- **원자료에서 확인된 역할:** Verifies every command family and the distinction among verdicts and malformed streams.
- **커밋 분류 요약:** Exercises all checker commands and distinguishes `OK`, `KO`, and invalid-stream failure.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 44ee0830e9f0`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `44ee0830e9f0` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- 각 instruction family를 stdin command program으로 실행해 known sorted result를 만드는 테스트 항목을 확인합니다.
- `ss`, `rr`, `rrr` combined command도 executable dispatch를 통해 검증되는지 확인합니다.
- unsorted/no-command → `KO` normal status와 valid-prefix + unknown command → no verdict + `Error` failure를 비교합니다.
- operation primitive 단위 테스트와 달리 checker CLI path를 실제 통과하는 범위를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** 11개 exact command가 checker CLI에서 silent replay되고 complete state만 `OK`, valid incomplete state는 `KO`, malformed command는 error가 되어야 합니다.
- **재현하는 failure/boundary:** swap, push, rotate, reverse rotate 및 `ss`/`rr`/`rrr`를 포함한 command programs로 알려진 sorted state를 만들고, unsorted/no-command와 valid prefix 뒤 `wat`을 별도로 사용합니다.
- **test technique:** deterministic checker CLI integration입니다.
- **통과하는 production path:** subprocess stdin → `read_next_line` → `apply_checker_command` → shared `op_*` emit=0 → EOF → `stack_is_complete_sorted` → verdict/error입니다.
- **이 테스트가 증명하는 것:** 나열한 command families의 dispatch와 CLI-level OK/KO/error status/stdout/stderr 구분을 확인합니다.
- **이 테스트가 증명하지 않는 것:** read interruption/permanent error, overlength/NUL, allocation/write fault, 모든 command sequence를 exhaust하지 않습니다. shared operation 자체의 독립 semantics도 아닙니다.
- **성격:** broad checker integration regression입니다.
- **막는 후속 회귀:** command mapping 누락, combined dispatch 한쪽 미적용, KO를 error로 바꾸는 변경, malformed stream 뒤 verdict를 출력하는 변경을 막습니다.

#### 구현 기록 — B
- **직전 관련 상태:** complete checker lifecycle은 있었지만 public command/vote 동작의 자동 증거가 없었습니다.
- **이 commit의 구현 역할:** 모든 instruction family를 checker executable stdin으로 통과시키고 세 terminal outcome을 고정합니다.
- **핵심 상태 전이 또는 boundary:** valid prefix가 이미 stack을 일부 바꿨더라도 뒤의 unknown command가 나오면 verdict 없이 전체 stream이 malformed error로 끝납니다.
- **failure/no-op/edge:** unsorted empty stream은 transport/protocol 성공이므로 `KO` status 0입니다.
- **이후 연결:** 다음 fix는 dispatch가 아니라 reader의 frame size와 EINTR policy를 교정합니다.
- **개발 흐름의 다음 관련 커밋:** `7713a31cf502`는 general line reader의 어떤 상태·allocation 정책을 protocol-sized frame으로 바꾸는가?

### `7713a31cf502` — fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도
- **중요도:** A
- **태그:** CHECKER, RUNTIME, RISK
- **원자료에서 확인된 역할:** Replaces unbounded lines with protocol-sized frames and retries interrupted reads.
- **커밋 분류 요약:** Replaces the unbounded reader with a four-byte frame buffer, rejects NUL or overlength input, and retries `EINTR`.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 7713a31cf502`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `7713a31cf502` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `0b87adebca2b`와 diff하여 dynamic buffer가 `PS_COMMAND_MAX + 1` fixed frame으로 교체된 위치를 확인합니다.
- 4번째 byte, embedded NUL, allocation failure를 frame reader가 dispatch 전에 reject하는지 확인합니다.
- `read`가 `EINTR`일 때 retry하고 다른 error에서 buffer free + caller pointer null + error return하는 경로를 확인합니다.
- clean EOF/no bytes와 valid EOF-delimited final command의 서로 다른 cleanup/return을 확인합니다.
- fault-allocation sweep이 revised reader allocation 동작에 맞춰 변경되는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 수정 과정
- **기존 가정:** `0b87adebca2b:read_next_line`은 checker 입력을 일반 text line으로 보고 32바이트에서 geometric growth하며 모든 non-newline byte를 허용했습니다. 모든 negative `read`를 동일 failure로 처리했습니다.
- **실제 failure 또는 위험:** checker protocol의 longest command는 3바이트인데 arbitrary line을 계속 할당·읽을 수 있었고, embedded NUL은 dispatch 문자열을 실제 frame보다 짧게 보이게 할 수 있었습니다. transient `EINTR`도 malformed/transport error로 잘못 종료됐습니다.
- **root cause:** reader가 protocol boundary를 소유하지 않고 general-purpose line abstraction을 제공했으며, errno별 retry policy가 없었습니다.
- **수정된 불변 조건/decision:** frame은 최대 3 non-newline bytes, NUL 금지, allocation은 정확히 `PS_COMMAND_MAX + 1`, `EINTR`만 retry, 다른 read error는 free+NULL+-1입니다. clean EOF/no bytes는 free+NULL+0, EOF-delimited nonempty frame은 1입니다.
- **실제 수정 코드:** `7713a31cf502:src/checker_reader.c:read_next_line`의 핵심은 다음과 같습니다.

```c
*line = (char *)ps_malloc(PS_COMMAND_MAX + 1);
...
if (bytes < 0 && errno == EINTR)
    continue ;
if (c == '\0' || len >= PS_COMMAND_MAX)
    return (ps_free(*line), *line = NULL, -1);
```

- **regression test:** 같은 commit에서 fault-allocation sweep의 reader allocation 횟수가 새 fixed allocation에 맞게 조정됩니다. protocol/read-fault의 직접 회귀는 후속 `dbf76e147e68`이 EIO/EINTR와 NUL/overlength/EOF cases로 제공합니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** checker lifecycle은 완성됐지만 reader가 실제 명령 protocol보다 넓고 transient interruption을 구분하지 않았습니다.
- **주요 boundary/decision:** 길이·NUL 유효성 검사를 dispatch가 아니라 byte를 받는 reader에 배치해 invalid frame이 command 비교 전에 차단됩니다.
- **state / 소유권 / failure 변화:** frame은 매 호출 한 번의 4-byte project allocation입니다. error와 clean EOF에서 reader가 해제하고 pointer를 NULL로 만들며, status 1일 때만 caller가 frame을 소유합니다.
- **보장 / 비보장:** bounded memory, NUL/4번째 byte reject, EINTR retry, EOF framing을 보장합니다. exact command membership은 여전히 dispatcher 책임이고 write failure는 다루지 않습니다.
- **후속 검증 또는 수정 연결:** `dbf76e147e68`이 read call별 fault injection과 protocol boundaries를 deterministic하게 검증합니다. output hardening은 Thread 6으로 이어집니다.
- **개발 흐름의 다음 관련 커밋:** `dbf76e147e68`은 command bytes와 final EOF probe 각각의 read call을 어떻게 선택해 EIO/EINTR를 주입하는가?

### `dbf76e147e68` — test(checker): 읽기 실패와 명령 경계를 검증
- **중요도:** A
- **태그:** TEST, CHECKER, RISK
- **원자료에서 확인된 역할:** Injects read faults and verifies NUL, empty, overlength, long-stream, and EOF-delimited boundaries.
- **커밋 분류 요약:** Injects permanent and interrupted reads and tests malformed, overlong, NUL, empty, and unterminated frames.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only dbf76e147e68`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `dbf76e147e68` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- runtime read counter와 selected-call `EIO`/`EINTR` injection 구현을 확인합니다.
- `sa\n` + EOF 전체 read sequence에 permanent failure를 sweep하는 test를 확인합니다.
- early command byte와 final EOF probe의 `EINTR`가 retry success로 이어지는 test를 확인합니다.
- embedded NUL, >3 bytes, empty command, standalone NUL, 64 KiB overlong frame rejection을 확인합니다.
- EOF-only terminated valid `sa`가 적용되어 `OK`가 되는 케이스를 확인합니다.
- allocation-reporting fault build를 통해 same cases의 frame leak도 함께 검사하는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** max-3/no-NUL frame, `EINTR` retry, other read error reject, EOF-delimited final command acceptance, 모든 종료에서 frame allocation cleanup입니다.
- **재현하는 failure/boundary:** runtime `ps_read` call counter가 environment-selected call에서 EIO 또는 EINTR를 반환합니다. `sa\n`의 `s`, `a`, newline, final EOF probe에 EIO를 각각 주입하고, 첫 byte와 EOF probe에는 EINTR를 주입합니다. embedded NUL, `rrrr`, empty line, NUL-only, 64KiB overlong, unterminated `sa`도 사용합니다.
- **test technique:** deterministic read fault injection + protocol boundary CLI regression + live-allocation reporting입니다.
- **통과하는 production path:** fault checker → `read_next_line` → runtime `ps_read` injection → retry/error/frame return → dispatch/verdict 또는 error cleanup → `ps_test_finish` allocation report입니다.
- **이 테스트가 증명하는 것:** 지정 read calls의 permanent failure가 error가 되고 transient EINTR는 같은 read를 재시도해 성공하며, malformed frames는 dispatch 전에 거절되고 valid EOF final frame은 적용되어 `OK`가 됨을 확인합니다. 각 case의 live allocation도 0이어야 합니다.
- **이 테스트가 증명하지 않는 것:** arbitrary errno, signal timing의 실제 비결정성, write failure, 모든 possible byte stream을 exhaustive하게 증명하지 않습니다.
- **성격:** deterministic fault regression과 protocol boundary regression입니다.
- **막는 후속 회귀:** EINTR를 permanent failure로 되돌리기, 4번째 byte/NUL 허용, EOF probe 실패 누락, 오류 처리 frame leak을 막습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** fix의 branch는 존재했지만 실제 read call 위치별 permanent/transient fault와 hostile frame을 자동으로 재현하는 증거가 없었습니다.
- **주요 boundary/decision:** runtime seam에서 call count를 세고 selected call만 EIO/EINTR로 바꿔 reader의 정확한 branch를 반복 가능하게 실행합니다.
- **state / 소유권 / failure 변화:** production normal 동작은 유지되고 fault build에 read counter/env controls가 생깁니다. test harness는 stderr의 allocation report까지 검사해 frame cleanup을 함께 관찰합니다.
- **보장 / 비보장:** source에 명시된 read/protocol cases의 regression을 제공합니다. output write와 closed-pipe 동작은 Thread 6이 담당합니다.
- **후속 검증 또는 수정 연결:** `315f4b91779b`/`e1154e181864`가 같은 runtime 경계를 write-all과 output fault injection으로 확장합니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| checker final success = sorted A + empty B | d906f4d86528 | - | - | - | 44ee0830e9f0 | `stack_is_complete_sorted` 뒤에만 `OK`; valid EOF지만 predicate false면 `KO` status 0입니다. |
| frame 최대 3 non-newline bytes / NUL 금지 | - | - | 0b87adebca2b는 unbounded general reader | 7713a31cf502 | dbf76e147e68 | reader가 4-byte buffer를 한 번 할당하고 4번째 byte 또는 NUL을 저장 전에 오류로 만듭니다. |
| `read` EINTR retry / other error reject | - | - | 0b87adebca2b에서는 interrupted read도 failure | 7713a31cf502 | dbf76e147e68 | errno==EINTR만 loop를 계속하고 EIO 등은 buffer free+NULL+-1입니다. selected-call injection이 두 경로를 구분합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| valid but unsorted/unfinished stream | `KO` normal verdict | d906f4d86528 | 44ee0830e9f0 | protocol과 transport가 정상 종료됐으므로 process error가 아니라 final-state negative verdict입니다. |
| unbounded command frame / embedded NUL / overlength | bounded protocol reader | 7713a31cf502 | dbf76e147e68 | general line abstraction이 protocol length와 binary NUL을 reader에서 제한하지 않은 것이 원인입니다. |
| transient `EINTR`를 permanent failure로 취급 | `EINTR` retry | 7713a31cf502 | dbf76e147e68 | 모든 negative read를 동일 처리했던 정책을 errno별 retry/reject로 나눴습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| checker stack A/B | checker executable ownership 없음 | d906f4d86528 | main이 parse/init 후 소유하고 verdict/error 모든 경로에서 해제 | `d906f4d86528:src/checker.c:main` |
| reader-owned command frame | 없음 | 0b87adebca2b, 7713a31cf502 | status 1에서 caller 소유, EOF/error에서는 reader가 해제+NULL; 고정 4바이트 | `7713a31cf502:src/checker_reader.c:read_next_line` |
| command dispatcher | 없음 | f79ae7e86592 | 11개 exact name을 shared `op_*` emit=0으로 매핑 | `f79ae7e86592:src/checker.c:apply_checker_command` |
| verdict output boundary | 없음 | d906f4d86528 | clean EOF 뒤 complete predicate로 OK/KO, malformed/read failure는 Error | `d906f4d86528:src/checker.c:main` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** checker는 값 인자를 parse해 A를 소유하고 동일 capacity B를 만든 뒤, 최대 3바이트·NUL 금지 frame을 한 번의 fixed allocation으로 읽습니다. `EINTR`는 재시도하고 다른 read error는 정리 후 실패합니다. 11개 exact command는 shared operation을 `emit=0`으로 적용합니다. clean EOF에만 A sorted+B empty를 검사해 `OK` 또는 normal `KO`를 출력하고, malformed input/command, allocation, I/O failure는 `Error`와 non-zero status입니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** 이 Thread 마지막 시점의 핵심 reader 경계는 fault tests로 보호되지만 output write 자체의 short/zero/EPIPE와 SIGPIPE는 Thread 6의 `315f4b91779b`/`e1154e181864`가 맡습니다. 현재 환경에서는 test executable을 실행하지 않았고, 문서의 test 결과 범위는 해당 SHA의 test code와 assertions를 확인한 것입니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: ``parse initial A → allocate B → bounded frame read → exact dispatch with `emit=0` → clean EOF → complete-state predicate → `OK`/`KO` or error cleanup``
- **학습자 최종 flow:** `d906f4d86528:main`이 no-values를 reader 전에 반환하거나 A/B를 구성합니다 → `7713a31cf502:read_next_line`이 4-byte buffer에서 protocol frame을 만들고 EINTR를 재시도합니다 → `f79ae7e86592:apply_checker_command`가 exact command를 `op_*` emit=0으로 적용합니다 → frame을 해제하고 반복합니다 → clean EOF면 `stack_is_complete_sorted`, 그 외 오류면 A/B cleanup+Error입니다.
- **실제 코드 삽입:** 초기 `0b87adebca2b`는 `grow_line`으로 arbitrary buffer를 두 배 확장했지만, fix는 위와 같이 `PS_COMMAND_MAX + 1`을 한 번 할당하고 NUL/4번째 byte를 reader에서 차단합니다. 이 비교가 protocol boundary 이동의 핵심입니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# 실행 시점 오류 주입과 출력 실패 전파

## 1. 개발 흐름 목표
- **원자료에서 확인된 중요성:** The thread changes 실패 처리 from implicit assumptions into an explicit runtime contract. The generator's product is an externally visible command stream, so successful in-memory sorting cannot compensate for an incomplete write. The final design preserves already-written prefixes, stops further emission, cleans all owned memory, and reports failure when the transport cannot deliver the complete result.
- **학습 목표:** 초기 output helper의 실패 무시 상태에서 runtime seam, allocation fault sweep, write-all contract, `SIGPIPE` 처리, partial-output regression evidence까지 발전하는 cross-cutting failure architecture를 복원합니다.

## 2. 이 개발 흐름을 이해하기 위한 핵심 질문
- 초기 `ps_putstr_fd`의 single-write/no-status 계약은 어떤 failure를 관찰할 수 없게 하는가?
- `ps_malloc`/`ps_free`/`ps_read` wrapper가 domain code를 바꾸지 않고 fault injection seam을 어떻게 제공하는가?
- Nth-allocation failure sweep이 partial construction cleanup을 어떻게 검증하는가?
- `ps_write_all`은 `EINTR`, short positive write, zero-byte write, permanent error를 어떻게 구분하는가?
- 이미 stdout에 보인 prefix가 있는 상태에서 왜 rollback을 시도하지 않고 emission을 중단하는가?
- closed pipe를 `SIGPIPE` termination 대신 ordinary write failure로 바꾸는 경로는 어디입니까?

## 3. 완료 기준
- allocation/read/write system boundary가 runtime wrapper로 모이는 실제 호출 경로를 확인했습니다.
- fault build의 allocation header/live-count와 `ps_test_finish` exit reporting을 추적했습니다.
- 315f4b91779b의 status propagation을 output helper → operation → sorter → main까지 따라갔습니다.
- partial stdout prefix, failed verdict, failed diagnostic, closed pipe 케이스의 exit/cleanup을 test 코드와 production 코드로 연결했습니다.

## 4. 커밋 목록

| 순서 | 커밋 | 제목 | 중요도 | 태그 | 원자료에서 확인된 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `2e97f29961d8` | feat(io): 문자열 비교와 기본 출력을 구현 | B | RUNTIME, PRACTICAL | Centralizes basic text output but initially ignores write results. |
| 2 | `5faa9d7697af` | refactor(runtime): 메모리와 입력 시스템 호출을 공통화 | A | ARCH, REFACTOR, RUNTIME | Creates runtime wrappers for allocation and input, providing an instrumentation seam. |
| 3 | `63969f770a21` | test(memory): 할당 실패 뒤 자원 정리를 검증 | A | TEST, RUNTIME, RISK | Uses that seam to sweep allocation failures and prove zero live allocations. |
| 4 | `315f4b91779b` | fix(io): 출력 실패를 호출 경로 끝까지 전파 | A | RUNTIME, RISK, INTEGRATION | Extends the runtime contract to write-all behavior, `SIGPIPE` handling, and end-to-end failure propagation. |
| 5 | `e1154e181864` | test(io): 부분 출력과 영구 쓰기 실패를 검증 | A | TEST, RUNTIME, RISK | Verifies interrupted, short, zero, permanent, diagnostic, and closed-pipe write paths. |

### 원자료에서 확인된 불변 조건과 구현 난점
- **Critical 불변 조건**
  - A successful `push_swap` execution means the complete emitted stream was written successfully; a merely sorted in-memory state is insufficient.
  - Short positive writes advance the output cursor, zero-byte writes are failures, and a closed pipe is handled as an ordinary error rather than process termination.
- **Major engineering difficulties**
  - Handling allocation failure, interrupted reads and writes, short writes, zero-byte writes, closed pipes, and already-visible output prefixes without transactional rollback.

## 5. 커밋별 학습 기록

> 모든 코드 확인은 반드시 해당 commit SHA 시점에서 수행합니다. 최종 HEAD의 구현을 소급해 해석하지 않습니다.

### `2e97f29961d8` — feat(io): 문자열 비교와 기본 출력을 구현
- **중요도:** B
- **태그:** RUNTIME, PRACTICAL
- **원자료에서 확인된 역할:** Centralizes basic text output but initially ignores write results.
- **커밋 분류 요약:** Adds project-local string comparison, descriptor output, and the canonical 오류 메시지.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 2e97f29961d8`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `2e97f29961d8` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `ps_strlen`, exact string comparison, descriptor output, canonical `Error\n` utility boundary를 확인합니다.
- `ps_strcmp`가 `unsigned char` 비교를 사용하는지 확인합니다.
- `ps_putstr_fd`가 single `write`를 수행하고 failure status를 노출하지 않는 API를 확인합니다.
- parser/operation/executable에서 direct `write` 대신 utility를 사용하게 되는 경계를 확인합니다.
- Makefile common-object 분류가 stack/utils를 shared core로 만드는 위치를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 구현 기록 — B
- **직전 관련 상태:** project-local exact string comparison과 공통 output/error helper가 없거나 direct system call에 흩어질 수 있는 상태였습니다.
- **이 commit의 구현 역할:** `2e97f29961d8:src/utils.c`에 `ps_strlen`, unsigned-char 기반 `ps_strcmp`, descriptor에 문자열을 쓰는 `ps_putstr_fd`, canonical `write_error`를 추가해 checker dispatch와 두 executable이 공통 text API를 사용하게 합니다.
- **핵심 상태 전이 또는 boundary:** `ps_strcmp`는 exact NUL-terminated command membership을 제공하고, `write_error`는 stderr에 `Error\n`을 쓰는 한 지점이 됩니다. output helper는 문자열 길이를 구해 `write` 한 번을 호출합니다.
- **failure/no-op/edge:** `ps_putstr_fd`와 `write_error`의 반환형이 `void`이고 `write` 결과를 버리므로 short write, `EINTR`, zero, EPIPE를 caller가 알 수 없습니다.
- **이후 연결:** `5faa9d7697af`가 memory/read system call도 runtime boundary로 모으고, `315f4b91779b`가 output helper를 fallible write-all API로 바꿉니다.
- **개발 흐름의 다음 관련 커밋:** `5faa9d7697af`는 parser/stack/checker의 direct allocation/read를 어떤 matching wrapper pair로 치환해 fault injection seam을 만드는가?

### `5faa9d7697af` — refactor(runtime): 메모리와 입력 시스템 호출을 공통화
- **중요도:** A
- **태그:** ARCH, REFACTOR, RUNTIME
- **원자료에서 확인된 역할:** Creates runtime wrappers for allocation and input, providing an instrumentation seam.
- **커밋 분류 요약:** Routes allocation, free, and read operations through a dedicated runtime boundary.

#### 원자료에서 확인된 맥락
- **문제:** Allocation and read failures occur below domain logic, but direct calls scattered across parser, stack, and checker code make those failures difficult to inject, count, and verify consistently.
- **결정:** Introduce `ps_malloc`, `ps_free`, and `ps_read` as a shared runtime boundary and migrate project-owned memory and checker input through it without changing normal semantics.
- **중요한 이유:** The abstraction enables later Nth-allocation failure sweeps, live-allocation accounting, read fault injection, write instrumentation, and resource metrics. It also gives all project allocations one matching release path.
- **변경 내용:** A runtime module is added, parser scratch storage, stack buffers, and checker command buffers migrate to it, and the low-level operation test links only the objects it actually requires.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 5faa9d7697af`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `5faa9d7697af` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `ps_malloc`, `ps_free`, `ps_read` wrapper 정의와 normal build delegation을 확인합니다.
- parser scratch buffer, stack buffers, checker command buffer가 direct libc/POSIX call에서 runtime wrapper로 이동하는 diff를 확인합니다.
- runtime에서 얻은 memory가 matching `ps_free`로 해제되는지 caller별로 추적합니다.
- operation-불변 조건 test가 필요한 object만 링크하도록 dependency graph가 줄어든 부분을 확인합니다.
- behavior change 없이 observability/testability seam만 추가되었는지 비교합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** stack 두 buffer, parser sorted scratch, checker frame이 각각 direct `malloc/free`를 쓰고 reader가 direct `read`를 호출해, 특정 acquisition만 실패시키거나 outstanding allocation을 일관되게 셀 수 없었습니다.
- **주요 boundary/decision:** `5faa9d7697af:src/runtime.c`에 normal build에서는 libc/POSIX에 그대로 위임하는 `ps_malloc`, `ps_free`, `ps_read`를 추가하고 project-owned memory/input call site를 모두 이 경계로 이동했습니다.
- **state / 소유권 / failure 변화:** 소유권 자체는 변하지 않습니다. `stack_init`이 얻은 두 buffer는 `stack_free`가 `ps_free`, `assign_ranks` scratch는 같은 함수가 `ps_free`, status 1 checker frame은 loop가 `ps_free`합니다. matching pair가 한 API로 모여 instrumentation 가능한 상태가 됩니다.
- **보장 / 비보장:** normal build의 의도된 semantics는 direct call과 같고 domain API를 바꾸지 않습니다. 아직 wrapper 자체는 failure를 주입·보고하지 않으며 output은 여전히 이 경계 밖에서 unchecked입니다.
- **후속 검증 또는 수정 연결:** `63969f770a21`이 compile-time fault build에서 Nth malloc과 live count를 구현하고, `dbf76e147e68`이 `ps_read`에 selected-call EIO/EINTR를 추가합니다. `315f4b91779b`는 write boundary까지 확장합니다.
- **개발 흐름의 다음 관련 커밋:** `63969f770a21`은 successful allocation 앞에 어떤 header를 배치하고 모든 executable exit에서 outstanding count를 어떻게 검증하는가?

### `63969f770a21` — test(memory): 할당 실패 뒤 자원 정리를 검증
- **중요도:** A
- **태그:** TEST, RUNTIME, RISK
- **원자료에서 확인된 역할:** Uses that seam to sweep allocation failures and prove zero live allocations.
- **커밋 분류 요약:** Adds an instrumented build that fails the Nth allocation and reports live allocations at exit.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 63969f770a21`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `63969f770a21` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `PS_FAULT_INJECTION`에서 allocation call counter와 selected Nth `NULL` injection을 확인합니다.
- successful allocation에 붙는 aligned header와 live-allocation tracking 구조를 확인합니다.
- 모든 executable exit가 `ps_test_finish`를 거쳐 non-zero live count를 별도 failure로 보고하는지 확인합니다.
- Python suite가 representative push_swap/checker path의 모든 allocation point와 one-past-last 기준 상태를 sweep하는지 확인합니다.
- 각 injected failure에서 public `Error` 동작과 zero live allocations를 동시에 검사하는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** project allocation이 어느 acquisition에서 실패해도 executable은 partial 소유권을 정리하고 종료 시 live project allocation이 0이어야 합니다.
- **재현하는 failure/boundary:** `PS_FAIL_MALLOC_AT=N`이 N번째 `ps_malloc`을 `NULL`로 만듭니다. representative `push_swap`은 allocation 1~5를, checker는 1~6을 각각 실패시키고 one-past-last 6/7을 정상 기준 상태로 실행합니다.
- **test technique:** compile-time deterministic Nth-allocation fault injection + exhaustive acquisition-point sweep for selected paths + exit live-count reporting입니다.
- **통과하는 production path:** fault binary → parser stack/sorted scratch 또는 B/checker frame `ps_malloc` → selected NULL → 해당 cleanup/오류 처리 → 모든 main return의 `ps_test_finish` → Python stderr/status assertion입니다.
- **이 테스트가 증명하는 것:** 선택한 representative paths에서 각 allocation point의 실패가 public non-zero/`Error` 동작으로 끝나고 `PS_LIVE_ALLOCATIONS=0`을 보고하며, one-past-last가 실제 모든 acquisition을 통과하는 baseline임을 확인합니다.
- **이 테스트가 증명하지 않는 것:** libc 내부 allocation, read/write fault, 모든 argv/command path, invalid/이중 해제의 일반 검출은 증명하지 않습니다. header magic/live count는 project wrapper를 통과한 memory에 한정됩니다.
- **성격:** deterministic fault regression이며 selected path의 allocation points를 sweep합니다.
- **막는 후속 회귀:** 두 번째 stack buffer 실패 시 첫 번째 buffer 누수, parser scratch 실패 cleanup 누락, checker frame/오류 처리 누수, main early return이 `ps_test_finish`를 우회하는 변경을 막습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** runtime seam은 있었지만 allocation 실패 위치와 live 소유권을 관찰하는 구현·test가 없었습니다.
- **주요 boundary/decision:** `PS_FAULT_INJECTION` build에서 allocation call count와 env-selected failure를 두고, successful allocation 앞에는 `long double`/pointer alignment를 만족하는 union header를 붙여 magic·size를 보관합니다.
- **state / 소유권 / failure 변화:** successful `ps_malloc`은 live count를 증가시키고 `ps_free`는 header magic을 지운 뒤 감소시킵니다. 모든 executable return은 `ps_test_finish(status)`를 거쳐 requested report에서 nonzero live count면 status 99를 반환합니다.
- **보장 / 비보장:** selected normal/error construction의 leak-free cleanup을 deterministic하게 확인합니다. source mutation, invalid pointer, output delivery는 범위 밖입니다.
- **후속 검증 또는 수정 연결:** `6569949742eb`이 같은 header size로 current/peak requested bytes를 추가하고, `e1154e181864`이 write failures에서도 live count 0을 확인합니다.
- **개발 흐름의 다음 관련 커밋:** `315f4b91779b`은 unchecked single write를 어떤 return contract로 바꾸고, 그 status가 첫 failed emission 뒤 sorter를 어떻게 중단시키는가?

### `315f4b91779b` — fix(io): 출력 실패를 호출 경로 끝까지 전파
- **중요도:** A
- **태그:** RUNTIME, RISK, INTEGRATION
- **원자료에서 확인된 역할:** Extends the runtime contract to write-all behavior, `SIGPIPE` handling, and end-to-end failure propagation.
- **커밋 분류 요약:** Adds write-all semantics, `SIGPIPE` handling, and status propagation through operations, sorting, checker, and both mains.

#### 원자료에서 확인된 맥락
- **문제:** The earlier output helper ignored write results. A command could mutate in-memory state but fail to reach stdout, and a closed pipe could terminate the process through `SIGPIPE` before normal cleanup.
- **결정:** Add a write-all loop that retries `EINTR`, advances after short writes, rejects zero or permanent failures, ignores `SIGPIPE`, and returns status through output helpers, operation wrappers, sort helpers, checker verdicts, and both executable 진입점.
- **중요한 이유:** The generator's actual product is the external command stream, not the final private stack state. The change prevents incomplete delivery from being reported as success, stops further generation after failure, preserves already-visible prefixes without repetition, and still releases all owned resources.
- **변경 내용:** Operation and sorting APIs become fallible, checker verdict writes are checked, both mains initialize pipe behavior and convert output failure to status one, and error reporting is attempted without replacing the original failure.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only 315f4b91779b`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `315f4b91779b` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- `ps_write_all` loop의 `EINTR` retry, short-positive cursor advance, zero-byte failure, permanent error branch를 확인합니다.
- `ps_putstr_fd`/diagnostic → operation wrapper → sorting helper → top-level sort → `main`까지 status return이 어떻게 전파되는지 추적합니다.
- 첫 failed emission 뒤 추가 command generation을 중단하는 caller branch를 확인합니다.
- 이미 written prefix를 rollback/repeat하지 않고 stack cleanup + failure status로 끝내는 흐름을 확인합니다.
- checker `OK`/`KO` write result 검증과 secondary `Error` write failure가 original failure status를 덮지 않는지 확인합니다.
- output-capable execution 전에 `SIGPIPE`를 ignore하는 위치와 closed-pipe write error 경로를 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 수정 과정
- **기존 가정:** `2e97f29961d8:ps_putstr_fd`는 command/error string을 single `write`로 보내고 return을 버렸으며 operation/sorter/main은 모두 `void` 또는 unconditional success 흐름이었습니다.
- **실제 failure 또는 위험:** short write는 command 일부만 보이게 하고도 성공으로 끝날 수 있고, `EINTR`는 재시도되지 않으며, zero write는 진척 없이 누락되고, EPIPE는 기본 `SIGPIPE`로 cleanup 전에 process를 종료할 수 있습니다. private stack 정렬은 incomplete public stream을 보상하지 못합니다.
- **root cause:** output transport result가 API contract에 포함되지 않아 호출 계층 어디에서도 실패를 관찰·중단·보고할 수 없었고, pipe signal policy도 초기화되지 않았습니다.
- **수정된 불변 조건/decision:** success는 requested byte 전부가 쓰인 경우만 1입니다. `EINTR`는 같은 cursor/count로 retry하고, positive short write는 cursor/count를 전진시키며, zero/permanent error는 0입니다. operation과 sorter는 첫 0을 즉시 반환하고 두 main은 cleanup 후 status 1을 유지합니다. `SIGPIPE`는 ignore해 EPIPE return path로 바꿉니다.
- **실제 수정 코드:** `315f4b91779b:src/runtime.c:ps_write_all`은 다음 loop를 사용합니다.

```c
while (count > 0)
{
    written = ps_write_once(fd, cursor, count);
    if (written < 0 && errno == EINTR)
        continue ;
    if (written <= 0)
        return (0);
    cursor += (size_t)written;
    count -= (size_t)written;
}
```

  `315f4b91779b:src/operations.c:op_*`는 state primitive를 먼저 적용한 뒤 `emit_op` status를 반환합니다. `315f4b91779b:src/sort.c`의 모든 연속 command branch는 이전 call이 0이면 다음 command를 호출하지 않습니다. `src/push_swap.c:main`은 A/B를 free한 뒤 diagnostic을 시도하고 원래 status를 `ps_test_finish`에 전달합니다.
- **regression test:** 후속 `e1154e181864`이 EINTR, one-byte short, zero, permanent failure, short-then-fail prefix, verdict/diagnostic failure, real closed pipe를 직접 주입·관찰합니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** allocation/read는 runtime seam과 fault tests를 가졌지만 output은 unchecked single write라 external product completeness가 성공 조건이 아니었습니다.
- **주요 boundary/decision:** write-all을 가장 낮은 runtime boundary에 두고 status를 `ps_putstr_fd` → `emit_op`/`op_*` → tiny/radix helpers → `sort_stack` → main까지 동일 방향으로 올립니다. checker verdict도 같은 helper를 사용합니다.
- **state / 소유권 / failure 변화:** operation은 private state를 먼저 mutate한 뒤 emit합니다. 따라서 failed command의 state가 이미 진행됐을 수 있지만 sorter는 즉시 중단하고 private A/B를 해제합니다. 이미 성공한 stdout prefix는 되돌리거나 다시 쓰지 않습니다. 이는 transaction rollback이 아니라 prefix-preserving failure입니다.
- **보장 / 비보장:** complete write만 success, no zero-write loop, short-write 정확 전진, closed pipe ordinary failure, cleanup과 original status 보존을 보장합니다. stdout에 이미 보인 prefix를 원자적으로 회수하거나 command 단위 atomicity를 보장하지는 않습니다.
- **후속 검증 또는 수정 연결:** `e1154e181864`이 runtime write seam을 확장해 각 branch와 partial prefix를 검증합니다. resource command counter는 성공 emission 뒤에만 증가해야 한다는 Thread 4 invariant와도 연결됩니다.
- **개발 흐름의 다음 관련 커밋:** `e1154e181864`은 baseline stream의 각 write 위치와 short-then-fail 조합을 어떻게 주입해 누락·중복 없는 prefix를 확인하는가?

### `e1154e181864` — test(io): 부분 출력과 영구 쓰기 실패를 검증
- **중요도:** A
- **태그:** TEST, RUNTIME, RISK
- **원자료에서 확인된 역할:** Verifies interrupted, short, zero, permanent, diagnostic, and closed-pipe write paths.
- **커밋 분류 요약:** Injects short, zero, interrupted, and permanent writes, including closed-pipe and diagnostic failures.

#### 해당 SHA에서 확인할 코드
- 먼저 `git show --name-only e1154e181864`로 이 commit이 실제로 건드린 파일을 확정합니다.
- 아래 항목은 반드시 `e1154e181864` 시점의 파일에서 확인합니다. 최종 HEAD의 같은 함수로 대체하지 않습니다.
- write fault runtime의 interrupted/short/zero/permanent mode와 selected-call injection을 확인합니다.
- successful multi-command 기준 상태를 기록한 뒤 각 command write를 차례로 실패시키는 sweep을 확인합니다.
- `EINTR` 및 one-byte short write가 exact 기준 상태 stream을 재구성하는 assertion을 확인합니다.
- short write 후 permanent failure에서 stdout이 정확한 successfully-written prefix만 포함하는지 확인합니다.
- checker verdict write failure, diagnostic write failure, already-closed pipe case를 각각 확인합니다.
- 모든 write failure case에서 allocation cleanup이 끝까지 도달하는지 확인합니다.
- 코드 인용을 남길 때는 `SHA:path:symbol` 형식으로 위치를 기록하고, 설명에 필요한 최소 구문만 삽입합니다.

#### 테스트 커밋 분석
- **대상 production 불변 조건:** `ps_write_all`은 EINTR/short를 완성하고 zero/permanent를 실패시키며, 첫 unrecoverable failure 뒤 추가 output을 만들지 않고 이미 쓴 prefix만 유지하고 cleanup/status failure에 도달해야 합니다.
- **재현하는 failure/boundary:** fault runtime은 selected write call에 `EINTR`, EPIPE, 0, 또는 count를 1로 줄이는 short를 주입합니다. `push_swap 3 2 1`의 multi-command 기준 상태 각 write failure, first-call EINTR/short/zero, short first+permanent second, checker verdict, parse/command diagnostic, 실제 closed stdout pipe를 사용합니다.
- **test technique:** deterministic selected-call write fault injection + exact-byte 기준 상태/prefix comparison + real pipe closure + live-allocation report입니다.
- **통과하는 production path:** fault binary → output helper → `ps_write_all`/injected `ps_write_once` → operation/sorter or verdict/diagnostic caller → stack/frame cleanup → `ps_test_finish` allocation report입니다.
- **이 테스트가 증명하는 것:** EINTR와 one-byte short가 기준 상태 byte stream을 정확히 완성하고, zero/permanent는 non-zero status가 되며, short-then-fail stdout은 기준 상태의 첫 1바이트만 포함해 반복/건너뜀 없이 멈춥니다. verdict/diagnostic failure와 real closed pipe도 process signal termination 대신 failure cleanup으로 끝나고 live allocation 0이어야 합니다.
- **이 테스트가 증명하지 않는 것:** 모든 OS/device behavior, concurrent writers, command 단위 atomicity, 임의의 여러 short/EINTR 조합 전체를 exhaust하지 않습니다.
- **성격:** deterministic I/O fault regression과 closed-pipe integration regression입니다.
- **막는 후속 회귀:** short write를 full success로 오인, cursor 미전진으로 prefix 반복, zero write 무한 loop, failure 뒤 다음 command emission, SIGPIPE cleanup 우회, diagnostic failure가 original status를 성공으로 바꾸는 변경을 막습니다.

#### 핵심 기록 — A
- **직전 관련 상태와 문제:** production fix는 있었지만 syscall edge와 partial externally visible stream을 반복 가능하게 검증하는 test seam이 없었습니다.
- **주요 boundary/decision:** write call counter와 env-selected mode를 `ps_write_once`에 배치하고, exact 기준 상태 byte string을 먼저 얻어 성공 복구는 equality, permanent failure는 exact prefix로 비교합니다.
- **state / 소유권 / failure 변화:** fault build write counter가 추가되지만 normal build semantics는 유지됩니다. 모든 테스트 항목은 allocation report를 요청해 output failure가 main/frame cleanup을 건너뛰지 않는지 함께 봅니다.
- **보장 / 비보장:** source가 열거한 interrupted/short/zero/permanent/verdict/diagnostic/closed-pipe 회귀를 제공합니다. 이미 출력된 byte rollback은 요구하지 않고 prefix preservation과 failure status를 요구합니다.
- **후속 검증 또는 수정 연결:** Thread 최종 regression layer이며 source는 이 이후 추가 output hardening commit을 지정하지 않습니다.
- **개발 흐름의 다음 커밋:** 없음. 개발 흐름의 최종 상태에서 이 commit의 남은 역할을 정리합니다.

## 6. 불변 조건 ledger

| 불변 조건 / contract | 처음 도입 | 강화 | 부족함이 드러난 지점 | fix | regression / evidence | 학습자 확인 메모 |
| --- | --- | --- | --- | --- | --- | --- |
| complete emitted stream까지 성공해야 `push_swap` success | - | - | 2e97f29961d8은 write result를 무시 | 315f4b91779b | e1154e181864 | `ps_write_all` 0이 operation/sorter/main으로 올라가며 main은 A/B free 후 status 1을 반환합니다. baseline write sweep이 incomplete delivery를 success로 숨기지 못하게 합니다. |
| short write cursor advance / zero write failure / closed pipe ordinary error | - | - | - | 315f4b91779b | e1154e181864 | positive count만큼 cursor/count를 갱신하고 `written <= 0`은 실패합니다. `SIGPIPE` ignore 후 closed pipe가 EPIPE return으로 들어갑니다. |
| owned allocation은 모든 exit path에서 release | 5faa9d7697af runtime boundary | 63969f770a21에서 failure sweep | - | - | 63969f770a21, e1154e181864 | wrapper allocation은 matching `ps_free`, 모든 main exit는 `ps_test_finish`; allocation 및 write fault cases에서 report 0을 요구합니다. |

## 7. 실패 → 수정 → 검증 연결

| 실패 또는 위험 | 기존 또는 선택한 대응 | Fix commit | 테스트 또는 근거 | 학습자 root-cause 기록 |
| --- | --- | --- | --- | --- |
| allocation 중간 실패 후 leak | runtime allocation seam + cleanup paths | - | 63969f770a21 | direct allocation을 wrapper로 모으고 Nth failure 뒤 stack/scratch/frame owner가 cleanup한 후 live count 0을 확인합니다. |
| write failure가 성공으로 숨음 | `ps_write_all` + end-to-end status propagation | 315f4b91779b | e1154e181864 | output helper가 status를 버리고 sorter/main이 fallible하지 않았던 것이 원인입니다. 모든 계층을 int success contract로 변경했습니다. |
| short write에서 중복/누락, zero write 무한 반복 위험 | cursor advance + zero-write failure | 315f4b91779b | e1154e181864 | positive actual bytes만큼 cursor/count를 갱신하고 0을 terminal failure로 처리합니다. exact baseline/prefix가 이를 검증합니다. |
| closed pipe가 cleanup 전에 `SIGPIPE`로 종료 | `SIGPIPE` ignore 후 write error 처리 | 315f4b91779b | e1154e181864 | output-capable main이 먼저 signal policy를 설정해 EPIPE가 return path와 stack cleanup을 통과하게 합니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | 이 Thread 시작 시 | 변화 commit | 이 Thread 종료 시 | 실제 코드 근거 |
| --- | --- | --- | --- | --- |
| runtime allocation boundary | project code의 direct malloc/free/read | 5faa9d7697af | 모든 project-owned memory/input이 `ps_malloc`/`ps_free`/`ps_read`를 통과 | `5faa9d7697af:src/runtime.c` 및 migrated callers |
| fault-injection allocation header/counters | 없음 | 63969f770a21, 6569949742eb | aligned header가 magic/size를 보유하고 live/current/peak를 fault build에서 계측 | `63969f770a21:src/runtime.c:ps_malloc/ps_free` |
| stdout command stream | single unchecked write, delivery 책임 없음 | 315f4b91779b | complete write만 success; prefix 뒤 failure면 즉시 generation 중단 | `315f4b91779b:src/runtime.c:ps_write_all`, `src/sort.c` |
| stderr diagnostic stream | unchecked `Error\n` | 315f4b91779b | diagnostic도 write-all을 시도하지만 실패해도 이미 선택된 original error status는 유지 | `315f4b91779b:src/push_swap.c:main`, `src/checker.c:main` |
| main-level stack cleanup | ordinary parse/sort/checker errors에서 cleanup | 63969f770a21, 315f4b91779b | allocation/output/closed-pipe failure에서도 owned stacks를 free하고 `ps_test_finish` 경유 | `315f4b91779b:src/push_swap.c:main`, `src/checker.c:main` |

## 9. 개발 흐름의 최종 상태
- **원자료 기준 최종 상태:** project memory와 input은 runtime wrappers를 통과하고 fault build는 Nth allocation, selected read/write, live allocation을 관찰합니다. output은 `ps_write_all`이 EINTR를 재시도하고 short write만큼 전진하며 zero/permanent를 실패시킵니다. emitting operation이 실패하면 tiny/radix caller가 즉시 반환하고 두 main은 private state를 정리한 뒤 status 1을 보존합니다. `SIGPIPE`는 ignore되어 closed pipe도 같은 ordinary 오류 처리를 통과합니다. 이미 쓰인 stdout prefix는 rollback하지 않되 반복하지도 않습니다.
- **남아 있는 한계 / 다른 Thread로 넘어가는 책임:** command 단위 원자성이나 visible prefix rollback은 보장하지 않으며 의도된 contract도 아닙니다. fault tests는 열거된 deterministic syscall 결과와 selected positions를 다루며 모든 kernel/device/concurrency 조합을 증명하지 않습니다. 이 환경에서는 binary를 실행하지 못했으므로 test result를 새로 주장하지 않고 각 SHA의 implementation과 assertion만 확인했습니다.

## 10. 최종 architecture 또는 실행 순서 정리
- Source-derived flow anchor: `basic output → runtime seam → allocation fault sweep → write-all + status propagation + SIGPIPE policy → injected write regressions`
- **학습자 최종 flow:** `2e97f29961d8:ps_putstr_fd`의 unchecked single write → `5faa9d7697af:runtime.c`의 malloc/free/read seam → `63969f770a21`의 aligned header/Nth failure/`ps_test_finish` → `315f4b91779b:ps_write_all`의 retry/progress/failure contract → `operations.c:emit_op/op_*` → `sort.c`의 첫 failure 즉시 반환 → `push_swap.c`/`checker.c` cleanup, optional diagnostic, original status → `e1154e181864`의 exact baseline/prefix와 closed-pipe regression입니다.
- **실제 코드 삽입:** 핵심 decision은 위 `ps_write_all` loop와 `sort.c`의 반복적인 `if (!op_*(...)) return (0);`입니다. operation이 private state를 먼저 바꾸더라도 emission failure가 곧 caller 중단으로 이어져 external stream에 뒤 command를 추가하지 않습니다.

## 11. 학습 완료 자가 점검
- [x] Thread commit 순서를 source와 동일하게 유지했습니다.
- [x] 모든 commit에서 지정된 SHA의 코드를 직접 확인했습니다.
- [x] 최종 HEAD를 과거 commit 설명에 소급 사용하지 않았습니다.
- [x] Source-confirmed fact와 직접 코드 확인 결과를 구분했습니다.
- [x] S/A commit은 decision, 불변 조건, 소유권/failure, 후속 evidence까지 추적했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 필요한 state/boundary만 충분히 확인했습니다.
- [x] test commit마다 production 불변 조건, failure/boundary, technique, production path, 증명/비증명 범위를 구분했습니다.
- [x] fix commit은 기존 가정 → failure/risk → root cause → 수정 불변 조건 → 실제 코드 → regression evidence 순서로 연결했습니다.
- [x] 불변 조건 ledger와 실패 → 수정 → 검증 표를 실제 코드 근거로 채웠습니다.
- [x] 별도 프로젝트 재학습 없이 이 개발 흐름의 설계 → 구현 → 실패/위험 → 수정/검증 흐름을 commit history에 근거해 설명할 수 있습니다.

---

# push_swap 개발 흐름 학습 기록

## 목적

이 디렉터리는 `commit-importance.md`에 정의된 개발 흐름s를 그대로 따라가며 `push_swap`의 commit history를 복습하기 위한 학습 골격입니다. 완성형 해설서가 아니라, 학습자가 각 commit SHA의 실제 코드를 직접 읽고 설계 → 구현 → 실패/위험 → 수정 → 검증의 발전 과정을 복원하도록 구성되어 있습니다.

## 권장 학습 순서

1. [Parallel stack state and operation invariants](01-parallel-stack-state-and-operation-invariants.md)
2. [Input grammar, coordinate compression, and size safety](02-input-grammar-coordinate-compression-and-size-safety.md)
3. [Building the sorting engine](03-building-the-sorting-engine.md)
4. [Independent correctness and cost evidence](04-independent-correctness-and-cost-evidence.md)
5. [Checker protocol and verdict hardening](05-checker-protocol-and-verdict-hardening.md)
6. [Runtime fault injection and output failure propagation](06-runtime-fault-injection-and-output-failure-propagation.md)

이 순서는 source의 개발 흐름 나열 순서를 그대로 사용합니다.

## Thread 문서 사용법

- 먼저 개발 흐름 목표, 핵심 질문, 완료 기준을 읽습니다.
- 커밋 목록의 순서를 바꾸지 않고 각 SHA를 차례대로 checkout 또는 `git show`로 확인합니다.
- `Source-confirmed` 항목은 두 source 문서가 이미 확정한 사실입니다. 재평가하지 않습니다.
- 학습 기록란에는 실제 해당 SHA의 코드에서 직접 확인한 내용만 채웁니다.
- 불변 조건 ledger와 실패 → 수정 → 검증 표는 commit을 개별 기능 목록으로 외우지 않고 시간에 따른 contract 변화를 연결하는 용도로 사용합니다.

## 해당 SHA 코드 확인 원칙

- 반드시 기록 대상 commit SHA 시점의 코드를 확인합니다.
- 먼저 `git show --name-only <sha>`로 실제 변경 파일을 확정한 뒤 `git show <sha> -- <path>` 또는 `git show <sha>:<path>`로 확인합니다.
- 변경 전 상태가 필요하면 parent 또는 문서가 지정한 직전 관련 SHA와 비교합니다.
- 같은 이름의 함수가 현재도 존재한다는 이유로 최종 HEAD 구현을 과거 commit의 근거로 사용하지 않습니다.
- 파일명이나 symbol이 source에 명시되지 않은 경우 임의로 추정하지 말고 해당 commit의 changed-file 목록에서 먼저 확정합니다.

## 최종 HEAD 소급 사용 금지

최종 HEAD에는 후속 fix, failure propagation, testability seam이 이미 섞여 있을 수 있습니다. 과거 commit의 설계와 한계를 설명할 때 최종 HEAD 코드를 사용하면 실제 발전 순서가 사라집니다. 각 문서의 모든 코드 근거에는 가능하면 `SHA:path:symbol`을 함께 기록합니다.

## S/A/B/C별 학습 깊이

- **S:** 프로젝트 핵심 architecture/불변 조건 또는 일반 sorting mechanism으로 다룹니다. 직전 상태, 문제, 기존 설계의 한계, 결정, 실제 핵심 코드, 소유권/lifecycle/상태 전이, failure scenario, 보장/비보장, 후속 fix/test까지 추적합니다.
- **A:** 주요 subsystem, boundary, 실패 처리, integration point를 추적합니다. 핵심 코드와 설계 판단, 책임 변화, 검증 연결을 확인합니다.
- **B:** Thread 흐름에서 맡는 구현 역할과 필요한 코드/state 변화를 확인합니다. S/A와 같은 깊이를 기계적으로 반복하지 않습니다.
- **C:** source의 개발 흐름s에는 C commit이 포함되어 있지 않습니다. 다른 문맥에서 C commit을 볼 때는 Thread 이해에 필요한 경우만 배경으로 사용합니다.

## 실제 코드 삽입 기준

- 설명에 직접 필요한 최소 코드만 삽입합니다.
- 코드 앞에 대상 SHA와 path/symbol을 기록합니다.
- 함수 전체를 복사하기보다 불변 조건, 소유권 transfer, state mutation order, failure branch, cleanup, test injection을 보여주는 구문을 우선합니다.
- 변경 전/후 비교가 핵심인 fix에서는 두 SHA의 대응 구문을 함께 남깁니다.
- source에 없는 구현 세부를 추측해서 정답처럼 채우지 않습니다.

## Test commit 학습 방법

각 test commit에서는 다음을 구분해서 기록합니다.

- 대상 production 불변 조건
- 재현하는 failure 또는 boundary
- 사용한 test technique
- 실제 통과하는 production code path
- 테스트가 증명하는 것
- 테스트가 증명하지 않는 것
- broad integration인지 deterministic regression인지 또는 다른 명시적 성격인지
- 후속 변경에서 막는 회귀

특히 product code와 test oracle이 구현을 공유하는지 여부를 확인합니다. 독립 모델, fault injection, deterministic resource 기준 상태, sanitizer는 서로 다른 종류의 evidence이므로 한 종류가 다른 종류를 대체한다고 가정하지 않습니다.

## 문서 완료 기준

- 모든 Thread 문서를 source 순서대로 완료했습니다.
- 각 commit의 SHA, subject, importance, tags를 바꾸지 않았습니다.
- 모든 코드 근거를 대상 SHA에서 직접 확인했습니다.
- S/A/B 깊이를 구분했고 test/fix commit의 학습 구조를 채웠습니다.
- 각 Thread의 불변 조건 ledger와 실패 → 수정 → 검증 연결이 실제 code/test 근거로 완성되었습니다.
- 프로젝트를 다시 처음부터 읽지 않아도 commit history를 근거로 설계 → 구현 → 실패/위험 → 수정 → 검증의 변화를 설명할 수 있습니다.
