===== BEGIN FILE: 01-소유권-ledger-to-unsafe-destruction.md =====
# Thread: 소유권 ledger to unsafe-destruction verdict

이 문서는 source에 정의된 첫 번째 Development Thread를 그대로 따릅니다. commit 순서, SHA, importance, tags는 변경하지 않습니다. 모든 코드 기록은 해당 SHA에서 작성하며 final HEAD를 과거 구현의 근거로 사용하지 않습니다.

## 1. Thread 목표

이 Thread의 목표는 table 중심 소유권 graph가 어떻게 partial initialization ledger로 확장되고, 최종적으로 worker quiescence가 입증되지 않으면 destruction 자체를 금지하는 verdict로 발전하는지 복원하는 것입니다.

Source-confirmed significance는 다음과 같습니다.

- 초기에는 `t_table`이 allocation과 ring object의 owner가 되고 philosopher가 그 주소를 빌립니다.
- readiness flag와 `fork_count`가 partial construction을 기록하지만, 초기 구현은 rollback 책임을 helper와 destructor에 나눠 duplicate destruction 위험을 남깁니다.
- common destructor가 유일한 ledger consumer가 되면서 exact-once initialization rollback이 복구됩니다.
- 같은 원칙이 thread lifecycle로 확장되어 successful join만 borrower quiescence의 증거가 됩니다.
- unsafe join 결과에서는 table cleanup뿐 아니라 normal process teardown도 금지됩니다.
- 회귀 테스트는 반환값만 보지 않고 ledger 보존과 forbidden cleanup의 부재를 검증합니다.

### Source에 명시적으로 연결된 Critical 항상 유지해야 하는 조건

- Initialized resource는 소유권 ledger가 존재한다고 말할 때에만, 최대 한 번 파괴합니다.
- destruction 실패 후에는 아직 해제되지 않은 resource를 나타내는 truthful, retryable state가 남아야 합니다.
- started worker가 모두 successful join된 경우가 아니면 shared table memory와 synchronization object를 파괴하지 않습니다.
- `t_table`은 allocation과 synchronization object의 owner이며, `t_philo`는 table과 fork array 내부 주소를 빌립니다.

### Source에 명시적으로 연결된 Major Engineering Difficulties

- partial initialization, failed join, mid-destruction error 전 구간에서 exact 소유권 evidence를 보존하는 문제
- unjoined worker가 table을 계속 역참조할 수 있어 cleanup 자체가 unsafe한 경우를 처리하는 문제
- ordinary error와 unsafe lifecycle verdict가 함께 존재할 때 safety verdict의 우선순위를 유지하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- allocation owner와 worker가 빌리는 주소는 어느 구조체와 field로 표현됩니까?
- 요청한 최종 resource 수가 아니라 실제 성공한 초기화 수를 어떻게 기록하는가?
- 초기 rollback에서 왜 helper와 destructor의 중복 책임이 double destroy 위험을 만드는가?
- cleanup ledger는 resource destroy의 성공 전과 성공 후 중 언제 소비되어야 하는가?
- `pthread_join` 호출을 시도했다는 사실과 worker quiescence가 입증되었다는 사실은 왜 다른가?
- join failure가 ordinary error가 아니라 `PHILO_UNSAFE`여야 하는 이유는 무엇입니까?
- unsafe verdict에서 destructor만 생략하는 것으로 충분하지 않고 `_exit`가 필요한 이유는 무엇입니까?
- test가 failure를 주입한 뒤 어떤 count, pointer, flag, output의 존재 또는 부재를 관찰하는가?

## 3. 완료 기준

- [x] `t_table`과 `t_philo`의 owned/borrowed 관계를 실제 field로 설명할 수 있습니다.
- [x] ring fork mapping이 stable shared identity를 만드는 코드를 해당 SHA에서 제시할 수 있습니다.
- [x] staged initialization의 readiness flag와 count가 어떤 순서로 갱신되는지 설명할 수 있습니다.
- [x] duplicate rollback의 두 cleanup owner와 실제 double-destroy 가능 경로를 재구성할 수 있습니다.
- [x] common destructor가 exact-once cleanup을 복구하는 변경을 before/after로 제시할 수 있습니다.
- [x] started/joined ledger와 destruction permission의 조건식을 설명할 수 있습니다.
- [x] failed destroy 뒤 retry 가능한 state가 보존되는 코드를 제시할 수 있습니다.
- [x] unsafe `main` path가 destructor, buffered stdio, `atexit`를 건너뛰는 것을 test 근거로 설명할 수 있습니다.
- [x] 이 Thread가 보장하지 않는 graceful cleanup과 실제 failed worker 상태를 구분할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `16343e76b54b` | `feat(init): 테이블 저장소와 철학자 관계 초기화` | S | `ARCH, CORE, RESOURCE_LIFECYCLE` | Establishes the table as the owner of allocations and the ring objects borrowed by philosophers. |
| 2 | `1d69df7db78c` | `feat(init): 뮤텍스 수명주기와 실패 롤백 구현` | A | `RESOURCE_LIFECYCLE, ARCH, RISK` | Adds staged mutex construction and resource-readiness ledgers, but still splits rollback responsibility. |
| 3 | `10665e0a5bf9` | `fix(init): 포크 초기화 실패 시 중복 정리 방지` | A | `RESOURCE_LIFECYCLE, DEBUG, RISK` | Centralizes partial fork rollback in the common destructor and restores exact-once cleanup. |
| 4 | `800408d6d84e` | `test(init): 부분 뮤텍스 초기화 롤백 검증` | A | `TEST, RESOURCE_LIFECYCLE, RISK` | Injects initialization failure and proves that each prepared mutex is destroyed once and allocations are released. |
| 5 | `a7783d04107f` | `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파` | S | `RESOURCE_LIFECYCLE, RISK, HARD` | Extends ownership evidence to worker creation, successful join, destruction permission, retryable cleanup, and `_exit` on unsafe state. |
| 6 | `7586b605302b` | `test(lifecycle): 생성·결합·정리 실패 경로 검증` | A | `TEST, RESOURCE_LIFECYCLE, EDGE` | Exercises create, join, and destroy failures across multiple partial-state positions. |
| 7 | `37b29557cccc` | `test(main): 결합 실패 시 안전하지 않은 정리 방지` | A | `TEST, RESOURCE_LIFECYCLE, RISK` | Proves the executable does not destroy resources or execute normal stdio and `atexit` teardown after an unsafe join result. |

## 5. Commit별 학습 기록

### 5.1 `16343e76b54b` — `feat(init): 테이블 저장소와 철학자 관계 초기화`

- Importance: **S**
- Tags: `ARCH, CORE, RESOURCE_LIFECYCLE`
- Source-defined role: Establishes the table as the owner of allocations and the ring objects borrowed by philosophers.
- 코드 기준: 반드시 `16343e76b54b` 시점
- 직접 parent 비교: `git diff 16343e76b54b^ 16343e76b54b --`
- Thread 직전 관련 SHA: Thread 내 첫 commit

#### Source-confirmed 맥락

이 commit은 `t_table`을 configuration, global state, fork 배열, philosopher 배열과 이후 synchronization 객체의 소유자로 두고, 각 `t_philo`가 자신의 identity와 progress field를 가지면서 table과 두 fork를 가리키도록 소유권 graph를 만듭니다. fork 관계는 `forks[i]`와 `forks[(i + 1) % number]`의 ring으로 표현되며, 인접 철학자가 복사된 상태가 아니라 같은 fork 객체를 공유합니다.

이 시점의 범위는 storage와 topology입니다. synchronization 초기화는 다음 commit에서 추가되므로, final 구조를 이 SHA에 소급해서 적지 않습니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 문제 | worker가 장기간 빌려 쓸 configuration, shared state, fork identity, philosopher-local state의 안정적인 저장소가 필요합니다. | §12 완료 기록의 대응 근거 참조 |
| 직전 상태 | 이 소유권 graph와 ring topology가 아직 중심 구조로 확정되지 않은 상태를 parent에서 확인합니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | `t_table`이 두 contiguous array와 공유 상태를 소유하고, `t_philo`는 table 및 fork 객체의 주소를 빌립니다. | §12 완료 기록의 대응 근거 참조 |
| 수명 결과 | worker가 빌린 주소의 유효 기간은 table-owned storage와 이후 thread quiescence 규칙에 종속됩니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `git show --name-status 16343e76b54b`로 구조체 선언, 초기화, 정리와 관련된 실제 파일을 식별합니다.
- [x] `t_table`과 `t_philo`의 필드를 나눠 적고 각 필드가 owned value, owned allocation, borrowed pointer 중 무엇인지 표시합니다.
- [x] fork 배열과 philosopher 배열의 allocation 순서, 크기 계산, 초기값 설정 순서를 추적합니다.
- [x] philosopher `i`의 left/right fork 주소가 실제로 어떤 식으로 계산되는지 확인하고 `i == number - 1`의 연결을 별도로 검산합니다.
- [x] 인접한 두 philosopher가 동일한 fork mutex 주소를 참조한다는 것을 주소 식으로 증명합니다.
- [x] 첫 allocation 성공 후 두 번째 allocation이 실패하는 branch에서 어떤 destructor 또는 정리 과정이 호출되고, 어떤 pointer가 free·NULL 처리되는지 확인합니다.
- [x] 이 SHA에서 synchronization object가 실제로 초기화되는지 여부를 확인하여 storage/topology 범위를 넘겨 쓰지 않습니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 소유권 / 수명 추적

| 대상 | Source-confirmed 관계 | 해당 SHA에서 확인한 선언·초기화 | 파괴 책임과 유효 기간 |
| --- | --- | --- | --- |
| stack-resident table value | process-level owner가 보유할 중심 객체 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| fork 배열 | `t_table`이 소유 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| philosopher 배열 | `t_table`이 소유 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `t_philo.table` | table을 빌리는 pointer | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `left_fork`, `right_fork` | fork 배열 내부 객체를 빌리는 pointer | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### 후속 연결

- 다음 관련 commit `1d69df7db78c`는 이 저장소 위에 mutex readiness와 partial-construction ledger를 추가합니다.
- 이후 `a7783d04107f`의 join-safety 판단은 여기서 형성된 borrowed-address 관계를 전제로 합니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- fork와 philosopher의 소유권 graph 및 ring topology가 명시됩니다.
- 부분 allocation 실패가 table-level cleanup 경로로 모입니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- mutex의 staged initialization과 exact-once rollback은 아직 완성되지 않았습니다.
- worker 생성·join·destruction permission은 아직 정의되지 않았습니다.

#### 학습자 결론
- [x] 왜 fork를 philosopher 안에 값으로 복사하면 안 되는지 실제 주소 관계로 설명합니다.
- [x] table storage가 worker보다 먼저 파괴되면 어떤 borrowed pointer가 무효화되는지 설명합니다.
- [x] 이 commit만으로 확립된 것과 다음 init commit이 추가하는 것을 구분합니다.

### 5.2 `1d69df7db78c` — `feat(init): 뮤텍스 수명주기와 실패 롤백 구현`

- Importance: **A**
- Tags: `RESOURCE_LIFECYCLE, ARCH, RISK`
- Source-defined role: Adds staged mutex construction and resource-readiness ledgers, but still splits rollback responsibility.
- 코드 기준: 반드시 `1d69df7db78c` 시점
- 직접 parent 비교: `git diff 1d69df7db78c^ 1d69df7db78c --`
- Thread 직전 관련 SHA: `16343e76b54b`

#### Source-confirmed 맥락

이 commit은 state mutex, print mutex, fork별 mutex를 순차적으로 초기화하고 readiness flag와 `fork_count`로 성공한 자원만 기록합니다. 정상 destruction과 실패 rollback이 같은 table-level destructor를 사용하도록 설계되지만, fork 초기화 helper도 실패 시 이미 초기화한 fork를 로컬에서 파괴하면서 `fork_count`를 그대로 남깁니다. 따라서 common destructor가 같은 mutex를 다시 파괴할 수 있는 split-responsibility 결함이 이 SHA에 남아 있습니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | 요청된 최종 자원 수가 아니라 성공적으로 준비된 자원만 cleanup해야 합니다. | §12 완료 기록의 대응 근거 참조 |
| 도입한 결정 | readiness flag와 `fork_count`가 synchronization 소유권 ledger 역할을 합니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 위험 | fork helper와 common destructor가 동일 mutex cleanup을 모두 담당합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 root cause | 로컬 rollback 후 ledger가 소비되거나 수정되지 않아 destructor가 이미 파괴한 fork를 다시 owned로 볼 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 후속 수정 | `10665e0a5bf9`에서 fork helper의 로컬 destruction을 제거하고 destructor만 ledger를 소비합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `16343e76b54b`와 비교하여 새로 추가된 mutex와 resource readiness field의 초기값을 확인합니다.
- [x] state mutex, print mutex, fork mutex의 실제 초기화 순서를 호출 그래프로 기록합니다.
- [x] 각 `pthread_mutex_init` 성공 직후 어떤 flag 또는 count가 갱신되는지 확인합니다.
- [x] 중간 실패 시 호출되는 table destructor가 어떤 순서로 resource를 검사하고 파괴하는지 확인합니다.
- [x] fork 초기화 helper의 실패 branch가 이미 성공한 fork를 직접 파괴하는 구간을 찾습니다.
- [x] 그 로컬 파괴 후 `fork_count` 값이 어떻게 남는지 확인하고 common destructor의 반복 파괴 가능 경로를 코드 순서로 작성합니다.
- [x] 배열 free 시점과 mutex destroy 시점의 순서를 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Partial-construction ledger

| 초기화 단계 | 성공 증거로 기록되는 field | 실패 시 common destructor가 보는 상태 | 학습자 확인 |
| --- | --- | --- | --- |
| state mutex | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| print mutex | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| fork mutex `0..k-1` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| backing arrays | pointer 존재 여부 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### 이 SHA의 결함 재구성

```text
성공한 fork 초기화
→ helper 내부 rollback
→ fork_count가 여전히 성공 개수를 나타냄
→ common destructor 진입
→ 동일 fork에 대한 두 번째 destruction 가능
```

위 흐름의 각 화살표에 대응하는 실제 함수와 branch를 기록합니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 부분 초기화를 요청된 최종 상태가 아니라 recorded 소유권으로 정리하려는 lifecycle 모델이 도입됩니다.
- state, print, fork mutex의 준비 상태를 destructor가 구분할 수 있습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- fork mutex exact-once destruction은 split-responsibility 때문에 아직 보장되지 않습니다.
- thread creation/join의 quiescence evidence는 아직 ledger에 포함되지 않습니다.

#### 학습자 결론
- [x] 왜 readiness flag와 count가 단순 편의 필드가 아니라 cleanup authorization인지 설명합니다.
- [x] double destroy가 가능한 정확한 failure index와 두 cleanup owner를 제시합니다.
- [x] 이 commit의 설계 방향은 유지되지만 구현 책임 분리가 왜 수정되어야 하는지 설명합니다.

### 5.3 `10665e0a5bf9` — `fix(init): 포크 초기화 실패 시 중복 정리 방지`

- Importance: **A**
- Tags: `RESOURCE_LIFECYCLE, DEBUG, RISK`
- Source-defined role: Centralizes partial fork rollback in the common destructor and restores exact-once cleanup.
- 코드 기준: 반드시 `10665e0a5bf9` 시점
- 직접 parent 비교: `git diff 10665e0a5bf9^ 10665e0a5bf9 --`
- Thread 직전 관련 SHA: `1d69df7db78c`

#### Fix chain

이 fix는 새 feature가 아니라 `1d69df7db78c`에서 생긴 rollback ownership 충돌을 바로잡습니다. fork initialization helper는 실패만 보고하고, `fork_count`를 가진 `philo_table_destroy`가 유일한 rollback owner가 됩니다. destructor는 initialized fork range를 역순으로 소비하고, shared mutex readiness와 count를 release 성공 후 초기 상태로 되돌립니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | `fork_count`가 성공적으로 초기화된 fork mutex의 authoritative ledger입니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | helper가 먼저 파괴하고 destructor가 같은 ledger를 다시 소비하여 double destroy가 가능합니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | 동일 자원에 대한 cleanup 책임이 helper와 table destructor로 분산되어 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정된 decision | helper는 failure만 반환하고 common destructor만 `fork_count`를 소비합니다. | §12 완료 기록의 대응 근거 참조 |
| 수정된 항상 유지해야 하는 조건 | 각 initialized synchronization object는 한 cleanup owner에 의해 최대 한 번 파괴됩니다. | §12 완료 기록의 대응 근거 참조 |
| regression 연결 | `800408d6d84e`가 네 번째 init 실패를 주입하고 세 owned mutex만 한 번씩 파괴되는지 검증합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `1d69df7db78c` 대비 fork initialization helper에서 제거된 local destroy loop를 확인합니다.
- [x] 실패 반환 직전 `fork_count`가 성공 개수를 그대로 보존하는지 확인합니다.
- [x] `philo_table_destroy`가 fork range를 어떤 방향으로 순회하고 count를 언제 감소시키는지 확인합니다.
- [x] state/print mutex readiness flag가 destroy 성공 전후 언제 바뀌는지 확인합니다.
- [x] 두 번째 destructor 호출이 이미 해제된 자원을 다시 다루지 않도록 pointer, count, flag가 어떤 상태로 남는지 확인합니다.
- [x] pthread destroy 실패를 이 commit이 어떻게 취급하는지 확인하되, 후속 lifecycle commit의 retry model을 이 SHA에 소급하지 않습니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Before / after 근거

| 관점 | `1d69df7db78c` | `10665e0a5bf9` |
| --- | --- | --- |
| fork rollback owner | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `fork_count` 소비 주체 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| destruction 순서 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| 두 번째 destructor 호출 결과 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- partial fork initialization rollback의 cleanup owner가 common destructor 하나로 통일됩니다.
- ledger가 이미 해제된 table을 표현하도록 reset되어 repeated cleanup이 같은 destruction을 재생하지 않습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- worker가 남아 있을 때 table destruction이 안전한지는 아직 이 fix의 범위가 아닙니다.
- 모든 pthread destructor failure에 대한 retryable lifecycle 모델은 후속 `a7783d04107f`에서 확장됩니다.

#### 학습자 결론
- [x] 코드 재사용이 아니라 exact-once 소유권 항상 유지해야 하는 조건 때문에 common rollback이 필요한 이유를 설명합니다.
- [x] 삭제된 local cleanup과 유지된 ledger를 한 쌍으로 제시합니다.
- [x] 회귀 테스트가 관찰해야 할 주소·count·pointer 상태를 예측한 뒤 실제 test와 대조합니다.

### 5.4 `800408d6d84e` — `test(init): 부분 뮤텍스 초기화 롤백 검증`

- Importance: **A**
- Tags: `TEST, RESOURCE_LIFECYCLE, RISK`
- Source-defined role: Injects initialization failure and proves that each prepared mutex is destroyed once and allocations are released.
- 코드 기준: 반드시 `800408d6d84e` 시점
- 직접 parent 비교: `git diff 800408d6d84e^ 800408d6d84e --`
- Thread 직전 관련 SHA: `10665e0a5bf9`

#### Source-confirmed test 역할

이 commit은 production initializer를 그대로 사용하면서 compile-time interposition으로 `pthread_mutex_init`과 `pthread_mutex_destroy`를 대체합니다. 네 번째 initialization을 실패시켜 세 mutex만 owned 상태로 만들고, destruction address를 기록하여 duplicate release를 직접 탐지합니다. 초기화 실패 후 allocation이 free·NULL 처리되는지와 두 번째 destructor 호출이 추가 destruction을 만들지 않는지도 검사합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| 재현하는 failure 또는 boundary | §12 완료 기록의 대응 근거 참조 |
| test technique | §12 완료 기록의 대응 근거 참조 |
| failure injection call index | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production 함수 경로 | §12 완료 기록의 대응 근거 참조 |
| 기록하는 주소·count·pointer | §12 완료 기록의 대응 근거 참조 |
| 핵심 assertion | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| broad integration / deterministic regression 구분 | §12 완료 기록의 대응 근거 참조 |
| 후속 변경에서 막아야 하는 회귀 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] test build가 어떤 macro 또는 wrapper로 pthread 함수를 치환하는지 확인합니다.
- [x] 초기화 호출 순서를 세어 왜 네 번째 call 실패가 정확히 세 owned mutex를 만드는지 대응표를 작성합니다.
- [x] destroy wrapper가 주소를 어떤 자료구조에 기록하고 duplicate를 어떻게 판별하는지 확인합니다.
- [x] test가 호출하는 production initializer와 common destructor의 실제 symbol을 확인합니다.
- [x] allocation release 및 pointer NULL assertion을 확인합니다.
- [x] 두 번째 destructor 호출 전후 destroy-call count가 변하지 않는 assertion을 확인합니다.
- [x] test 전용 branch가 production source에 추가되지 않았는지 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 지정한 partial-init 경계에서 정확히 owned mutex만 한 번씩 파괴되는지 결정적으로 검증합니다.
- 초기화 실패 후 table cleanup의 idempotent 관찰 상태를 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 자연 발생하는 모든 allocation/pthread failure 조합을 포괄하지 않습니다.
- thread creation, join, live borrower가 있는 teardown은 검증하지 않습니다.

#### 학습자 결론
- [x] 이 test가 shell smoke보다 root cause에 더 직접적인 이유를 설명합니다.
- [x] 실패 index, owned resource 수, expected destruction address 수를 계산합니다.
- [x] test가 production rollback ledger를 실제로 통과한다는 근거를 제시합니다.

### 5.5 `a7783d04107f` — `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파`

- Importance: **S**
- Tags: `RESOURCE_LIFECYCLE, RISK, HARD`
- Source-defined role: Extends 소유권 evidence to worker creation, successful join, destruction permission, retryable cleanup, and `_exit` on unsafe state.
- 코드 기준: 반드시 `a7783d04107f` 시점
- 직접 parent 비교: `git diff a7783d04107f^ a7783d04107f --`
- Thread 직전 관련 SHA: `800408d6d84e`

#### Source-confirmed 맥락

이 S-level fix는 initialization ledger 원칙을 thread quiescence와 retryable destruction까지 확장합니다. table은 성공적으로 시작한 worker 수, 성공적으로 join한 worker 수, destruction safety를 기록합니다. join을 시도했다는 사실은 quiescence의 증거가 아니며, 성공한 join만 ledger를 전진시킵니다. 하나라도 join을 입증하지 못하면 `PHILO_UNSAFE`가 반환되고 table destruction은 거부됩니다.

destructor는 pthread resource destruction이 성공한 뒤에만 해당 ledger를 감소시키거나 readiness flag를 지웁니다. `main`은 unsafe verdict를 일반 오류와 구분해 unbuffered diagnostic 후 `_exit`하며, table destruction, stdio flush, normal exit handler를 거치지 않습니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 상태 | control flow가 `philo_run` 끝에 도달하면 worker가 종료되었다고 전제하고 cleanup할 수 있었습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | 실패한 `pthread_join`은 worker가 멈췄다는 증거가 아니므로 table free 또는 live mutex destroy가 해제 후 사용을 만들 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | thread handle에 대한 시도와 borrower quiescence에 대한 증거를 구분하지 않았습니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | `threads_started`, `threads_joined`, `destroy_safe`와 `PHILO_UNSAFE`로 destruction permission을 명시합니다. | §12 완료 기록의 대응 근거 참조 |
| cleanup 결정 | 모든 started worker의 successful join이 증명될 때만 shared table resource를 파괴합니다. | §12 완료 기록의 대응 근거 참조 |
| process 결정 | quiescence를 증명하지 못하면 graceful cleanup 대신 `_exit`로 normal teardown을 우회합니다. | §12 완료 기록의 대응 근거 참조 |
| retry 결정 | resource destroy 성공 후에만 ledger를 소비하여 mid-destruction failure 뒤 재시도를 허용합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] public status model에서 `PHILO_UNSAFE`가 정의되고 ordinary error와 어떻게 구분되는지 확인합니다.
- [x] `t_table`의 `threads_started`, `threads_joined`, `destroy_safe` 초기값과 갱신 지점을 모두 찾습니다.
- [x] worker creation 성공 직후 started ledger가 증가하는 순서를 확인합니다.
- [x] `join_started`가 모든 recorded handle을 시도하는지, 성공한 join만 joined ledger에 반영하는지 확인합니다.
- [x] creation error 또는 barrier error와 join unsafe가 동시에 존재할 때 반환 우선순위를 확인합니다.
- [x] `philo_table_destroy`가 quiescence ledger를 검사하고 unsafe일 때 pointer, fork_count, destruction call을 건드리지 않는지 확인합니다.
- [x] mutex/condition destroy 실패 시 count 또는 readiness flag가 성공 전에 지워지지 않는지 확인합니다.
- [x] `main`의 safe run failure, cleanup failure, unsafe run failure 분기를 나란히 추적합니다.
- [x] unsafe branch에서 unbuffered output 뒤 `_exit`가 호출되고 table destructor나 `exit`/return 경로로 이어지지 않는지 확인합니다.
- [x] stack-resident table과 worker가 빌린 arrays/mutex 사이의 수명을 sequence diagram으로 기록합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Quiescence와 destruction authorization

| 관찰 상태 | Source-confirmed verdict | 실제 조건식·반환 경로 |
| --- | --- | --- |
| `threads_started == threads_joined`, destroy state 정상 | destruction을 시도할 수 있음 | §12 완료 기록의 대응 근거 참조 |
| join 하나 이상 실패 | `PHILO_UNSAFE`; destruction 거부 | §12 완료 기록의 대응 근거 참조 |
| 일부 pthread resource destroy 실패 | 남은 소유권 ledger 보존; 재시도 가능 | §12 완료 기록의 대응 근거 참조 |
| unsafe verdict가 ordinary run error와 함께 존재 | unsafe가 safety verdict로 우선 | §12 완료 기록의 대응 근거 참조 |

#### 핵심 수명 설명 기록

```text
table-owned storage 생성
→ worker handle 시작 및 borrowed access 가능
→ terminal/abort publication
→ 모든 started handle에 join 시도
→ successful join 수로 quiescence 증명
→ [safe] ledger-driven destroy
   [unsafe] diagnostic + _exit
```

각 단계에 실제 함수, state field, lock 또는 return status를 붙입니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- shared table resource destruction은 successful join으로 모든 borrower의 quiescence를 입증한 뒤에만 허용됩니다.
- join failure는 ordinary cleanup error가 아니라 unsafe verdict로 전파됩니다.
- mid-destruction failure 뒤 아직 owned인 resource의 ledger가 보존되어 명시적 retry가 가능합니다.
- unsafe path는 normal process teardown을 의도적으로 우회합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 실패한 join 대상 worker가 실제로 종료되었다고 추정하지 않습니다.
- unsafe path에서 graceful resource release를 보장하지 않으며, 안전을 위해 의도적으로 포기합니다.
- pthread 또는 output의 모든 가능한 실패를 해결했다는 뜻은 아닙니다.

#### 후속 regression evidence

- `7586b605302b`: create, join, destroy failure를 여러 index에 주입하여 partial-state ledger와 retryability를 검증합니다.
- `37b29557cccc`: executable 수준에서 unsafe branch가 destructor, buffered stdio, `atexit`를 실행하지 않는지 검증합니다.


#### 학습자 결론
- [x] 왜 `pthread_join` 호출 시도와 successful join을 다른 lifecycle 사실로 취급해야 하는지 설명합니다.
- [x] borrowed pointer 모델이 `PHILO_UNSAFE`와 `_exit` 결정으로 이어지는 논리를 설명합니다.
- [x] destructor failure에서 ledger를 성공 후에만 소비해야 retry가 가능한 이유를 실제 code order로 제시합니다.
- [x] ordinary error와 unsafe verdict의 우선순위를 호출 흐름 전체에서 설명합니다.

### 5.6 `7586b605302b` — `test(lifecycle): 생성·결합·정리 실패 경로 검증`

- Importance: **A**
- Tags: `TEST, RESOURCE_LIFECYCLE, EDGE`
- Source-defined role: Exercises create, join, and destroy failures across multiple partial-state positions.
- 코드 기준: 반드시 `7586b605302b` 시점
- 직접 parent 비교: `git diff 7586b605302b^ 7586b605302b --`
- Thread 직전 관련 SHA: `a7783d04107f`

#### Source-confirmed test 역할

이 deterministic failure matrix는 thread creation, joining, mutex destruction을 wrapper로 대체합니다. 세 worker run의 각 creation 위치에서 실패를 주입하고, 성공적으로 시작된 prefix만 join되는지 확인합니다. join 실패 시에는 다른 worker join을 계속하더라도 failed worker가 unjoined로 남고, table destructor가 resource 상태를 변경하지 않은 채 `PHILO_UNSAFE`를 반환해야 합니다. destroy failure는 reverse cleanup의 여러 단계에 주입되며, 첫 실패 뒤 ledger가 owned state를 유지하고 재호출로 cleanup이 끝나는지 검사합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| create failure index별 started/joined 예상값 | §12 완료 기록의 대응 근거 참조 |
| join failure 후 unsafe verdict와 보존되어야 할 state | §12 완료 기록의 대응 근거 참조 |
| destroy failure stage별 남아야 할 ledger | §12 완료 기록의 대응 근거 참조 |
| test technique와 real API delegation | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production 코드 경로 | §12 완료 기록의 대응 근거 참조 |
| 핵심 positive assertion | §12 완료 기록의 대응 근거 참조 |
| 핵심 negative assertion | §12 완료 기록의 대응 근거 참조 |
| 재시도 절차 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic failure matrix 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] create wrapper가 몇 번째 호출에서 실패하도록 설정되고 각 case가 어떻게 반복되는지 확인합니다.
- [x] 각 case에서 production `philo_run` 또는 join helper가 보는 `threads_started`와 `threads_joined`를 기록합니다.
- [x] join wrapper가 특정 handle에서 실패한 뒤 나머지 join을 계속하는지 확인합니다.
- [x] unsafe 상태에서 destructor 호출 전후 fork pointer, fork_count, readiness flag, destroy-call count가 동일하다는 assertion을 확인합니다.
- [x] test가 실패한 worker를 real `pthread_join`으로 정리하고 test ledger를 수선한 뒤 production cleanup을 다시 허용하는 절차를 확인합니다.
- [x] destroy failure injection이 reverse order의 서로 다른 지점을 겨냥하는지 확인합니다.
- [x] 첫 destructor 실패 후 backing allocation과 remaining ledger가 truthfully owned state를 나타내는지 확인합니다.
- [x] 두 번째 destructor 호출에서 injection을 해제하고 cleanup이 완료되는 assertion을 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- zero, partial, nearly complete construction/teardown 위치에서 lifecycle arithmetic과 verdict를 결정적으로 검증합니다.
- unsafe refusal가 resource 상태를 보존하고 destroy failure가 retryable state를 남기는지 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 OS-level pthread failure 동작이나 실제 scheduler interleaving을 재현하지 않습니다.
- failed join worker의 실제 내부 상태를 증명하지 않으며, 바로 그 불확실성 때문에 unsafe로 취급합니다.

#### 학습자 결론
- [x] 각 failure index에 대해 expected started/joined/destruction count를 표로 계산합니다.
- [x] unsafe test가 실패한 worker를 사후 정리하는 test-only 절차와 production verdict를 구분합니다.
- [x] retryability를 단순한 두 번째 호출 성공이 아니라 ledger 보존으로 증명하는 assertion을 설명합니다.

### 5.7 `37b29557cccc` — `test(main): 결합 실패 시 안전하지 않은 정리 방지`

- Importance: **A**
- Tags: `TEST, RESOURCE_LIFECYCLE, RISK`
- Source-defined role: Proves the executable does not destroy resources or execute normal stdio and `atexit` teardown after an unsafe join result.
- 코드 기준: 반드시 `37b29557cccc` 시점
- 직접 parent 비교: `git diff 37b29557cccc^ 37b29557cccc --`
- Thread 직전 관련 SHA: `7586b605302b`

#### Source-confirmed test 역할

이 process-level negative test는 real `main` 주변의 parse, init, run, destroy를 대체합니다. run stub은 buffered standard output을 남기고 `PHILO_UNSAFE`를 반환하며, parse 단계는 `atexit` hook을 등록합니다. destroy stub도 호출되면 보이는 marker를 출력합니다. child process 결과에는 unbuffered join diagnostic만 있어야 하고 destroy marker, buffered output, normal exit-hook output은 없어야 합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| 주입하는 unsafe 결과 | §12 완료 기록의 대응 근거 참조 |
| buffered stdout의 역할 | §12 완료 기록의 대응 근거 참조 |
| `atexit` hook의 역할 | §12 완료 기록의 대응 근거 참조 |
| destroy marker의 역할 | §12 완료 기록의 대응 근거 참조 |
| 실제로 실행하는 `main` 분기 | §12 완료 기록의 대응 근거 참조 |
| 반드시 존재해야 하는 출력/상태 | §12 완료 기록의 대응 근거 참조 |
| 반드시 없어야 하는 출력/상태 | §12 완료 기록의 대응 근거 참조 |
| `_exit`와 return/`exit`의 관찰 차이 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic process-level regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] real `main`을 test binary에서 어떻게 포함하거나 연결하는지 확인합니다.
- [x] parse/init/run/destroy stub의 반환값과 side effect를 기록합니다.
- [x] stdout이 flush되지 않도록 어떤 방식으로 buffered output을 준비하는지 확인합니다.
- [x] `atexit` hook과 destroy stub marker가 각각 normal teardown의 어느 부분을 감지하는지 확인합니다.
- [x] 자식 프로세스의 stderr/stdout/종료 상태를 test runner가 어떻게 수집하는지 확인합니다.
- [x] unbuffered unsafe diagnostic이 존재하는 assertion을 확인합니다.
- [x] destroy marker, buffered output, exit-hook output의 부재를 각각 확인하는 negative assertion을 찾습니다.
- [x] 단순히 destructor를 건너뛰는 것과 `_exit`로 normal teardown 전체를 건너뛰는 것의 차이를 코드 경로로 설명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- unsafe verdict에서 `main`이 table destruction과 normal stdio/`atexit` teardown을 실행하지 않는 process contract를 검증합니다.
- unbuffered diagnostic은 `_exit` 전에 관찰 가능함을 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 실제 `pthread_join` failure 자체를 이 test가 발생시키는 것은 아닙니다.
- unsafe worker가 어떤 상태인지 또는 OS가 process 자원을 어떻게 회수하는지 증명하지 않습니다.

#### 학습자 결론
- [x] 왜 destroy 호출 부재만 확인해서는 `_exit` contract를 충분히 증명하지 못하는지 설명합니다.
- [x] 세 negative signal이 각각 어떤 잘못된 normal teardown 경로를 잡는지 설명합니다.
- [x] Thread 전체가 allocation owner에서 process-level destruction verdict로 발전한 과정을 이 test까지 연결합니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 확정한 항상 유지해야 하는 조건의 시간상 역할만 미리 배치했습니다. 실제 field, 함수, mutation 순서와 test evidence는 학습자가 해당 SHA에서 채웁니다.

| 항상 유지해야 하는 조건 | 최초 도입 또는 문제 노출 | 강화·복구 | regression evidence | 해당 SHA 코드 근거 | 최종 설명 |
| --- | --- | --- | --- | --- | --- |
| table이 allocation을 소유하고 philosopher는 주소를 빌림 | `16343e76b54b` | `a7783d04107f`에서 borrower quiescence가 destruction permission으로 확장 | `7586b605302b`, `37b29557cccc` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| partial initialization은 성공한 resource만 ledger에 기록 | `1d69df7db78c` | `10665e0a5bf9`에서 common destructor 단독 소비로 복구 | `800408d6d84e` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| initialized mutex는 최대 한 번 파괴 | `1d69df7db78c`에서 split rollback으로 부족함 노출 | `10665e0a5bf9` | `800408d6d84e` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| destroy 실패 뒤 remaining ownership state는 truthful하고 retryable | `a7783d04107f` | 동일 commit의 success-after-destroy ledger update | `7586b605302b` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| shared table resource는 모든 started worker의 successful join 뒤에만 파괴 | `a7783d04107f` | unsafe verdict와 destructor refusal | `7586b605302b`, `37b29557cccc` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| unsafe process path는 normal teardown을 실행하지 않음 | `a7783d04107f` | `_exit` process contract | `37b29557cccc` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

## 7. Failure → Fix → Test 연결

### 7.1 Partial fork initialization rollback

```text
`1d69df7db78c`
recorded fork ownership 도입
→ helper와 destructor가 모두 cleanup
→ duplicate destruction 위험
→ `10665e0a5bf9`
common destructor가 유일한 ledger consumer
→ `800408d6d84e`
네 번째 init failure + destruction address 기록
```

- 기존 가정의 실제 코드: §12 완료 기록의 대응 근거에 정리했습니다.
- failure branch와 root cause: §12 완료 기록의 대응 근거에 정리했습니다.
- fix에서 삭제·이동된 cleanup 책임: §12 완료 기록의 대응 근거에 정리했습니다.
- regression assertion: §12 완료 기록의 대응 근거에 정리했습니다.
- 이 연결이 보장하는 exact-once 범위: §12 완료 기록의 대응 근거에 정리했습니다.
- 이 연결이 보장하지 않는 범위: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.2 Join failure와 unsafe destruction

```text
worker가 started됨
→ join call 실패
→ quiescence를 입증하지 못함
→ table을 파괴하면 live borrower와 충돌 가능
→ `a7783d04107f`
started/joined ledger + PHILO_UNSAFE + destructor refusal + _exit
→ `7586b605302b`
join/destroy failure matrix
→ `37b29557cccc`
normal process teardown 부재 검증
```

- 기존 cleanup 가정: §12 완료 기록의 대응 근거에 정리했습니다.
- failed join이 남기는 불확실성: §12 완료 기록의 대응 근거에 정리했습니다.
- unsafe verdict가 ordinary error보다 우선하는 코드: §12 완료 기록의 대응 근거에 정리했습니다.
- destructor refusal 시 보존되는 state: §12 완료 기록의 대응 근거에 정리했습니다.
- `_exit` 전후 관찰 가능한 output: §12 완료 기록의 대응 근거에 정리했습니다.
- 두 test가 서로 다르게 증명하는 범위: §12 완료 기록의 대응 근거에 정리했습니다.

## 8. 소유권 / state / responsibility 변화

| 시점 | Allocation 소유권 | Synchronization 소유권 | Worker 수명 evidence | Destruction responsibility | 학습자 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| `16343e76b54b` | table이 arrays 소유 | 아직 후속 단계 | 아직 없음 | table-level cleanup | §12 완료 기록의 대응 근거 참조 |
| `1d69df7db78c` | 기존 유지 | readiness flag와 `fork_count` 도입 | 아직 없음 | helper와 destructor가 충돌 | §12 완료 기록의 대응 근거 참조 |
| `10665e0a5bf9` | 기존 유지 | common destructor가 fork ledger 단독 소비 | 아직 없음 | exact-once rollback owner 통일 | §12 완료 기록의 대응 근거 참조 |
| `a7783d04107f` | worker가 빌린 storage까지 lifetime 판단에 포함 | destroy 성공 후 ledger 소비 | started/joined로 quiescence 입증 | safe면 retryable destroy, unsafe면 거부 | §12 완료 기록의 대응 근거 참조 |
| executable unsafe path | process가 즉시 종료 | normal teardown 생략 | 미입증 상태를 그대로 unsafe로 취급 | `_exit`가 최종 process decision | §12 완료 기록의 대응 근거 참조 |

## 9. Thread 최종 상태

### Source-confirmed 최종 상태

- table 소유권 graph는 allocation뿐 아니라 synchronization resource와 worker borrower 수명의 근거가 됩니다.
- partial initialization과 partial destruction은 recorded 소유권을 통해 정리됩니다.
- successful join 수가 모든 started worker의 quiescence를 입증해야 table destruction이 허용됩니다.
- join safety가 불명확하면 `PHILO_UNSAFE`가 반환되고 normal cleanup 대신 `_exit`가 사용됩니다.
- destroy failure는 remaining ledger를 보존해 safe한 명시적 retry를 허용합니다.

### 학습자가 작성할 최종 설명

- 소유권 graph: §12 완료 기록의 대응 근거에 정리했습니다.
- construction ledger: §12 완료 기록의 대응 근거에 정리했습니다.
- exact-once rollback: §12 완료 기록의 대응 근거에 정리했습니다.
- quiescence proof: §12 완료 기록의 대응 근거에 정리했습니다.
- safe destruction predicate: §12 완료 기록의 대응 근거에 정리했습니다.
- retryable destroy: §12 완료 기록의 대응 근거에 정리했습니다.
- unsafe process termination: §12 완료 기록의 대응 근거에 정리했습니다.
- 의도적으로 보장하지 않는 graceful cleanup: §12 완료 기록의 대응 근거에 정리했습니다.

## 10. 최종 architecture 또는 실행 순서 정리

다음 골격에 실제 함수명, field, return status, lock/resource mutation을 채웁니다.

```text
main의 stack-resident table
    ↓ initialize
table-owned arrays + synchronization ledger
    ↓ create workers
started ledger + borrowed table/fork addresses
    ↓ terminal 또는 abort
join every recorded worker
    ├─ all required joins successful
    │      ↓
    │  destroy_safe verdict
    │      ↓
    │  reverse, ledger-driven, retryable destruction
    └─ one or more joins not proven
           ↓
       PHILO_UNSAFE
           ↓
       unbuffered diagnostic
           ↓
       _exit without destructor / stdio flush / atexit
```

- 각 화살표의 production symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- 각 단계의 success ledger mutation: §12 완료 기록의 대응 근거에 정리했습니다.
- 각 failure branch의 return precedence: §12 완료 기록의 대응 근거에 정리했습니다.
- worker가 table을 역참조할 수 있는 마지막 지점: §12 완료 기록의 대응 근거에 정리했습니다.
- destruction이 합법화되는 정확한 조건: §12 완료 기록의 대응 근거에 정리했습니다.
- test가 각 branch를 관찰하는 방식: §12 완료 기록의 대응 근거에 정리했습니다.

## 11. 학습 완료 자가 점검

- [x] final HEAD의 field나 cleanup 순서를 과거 SHA에 소급하지 않았습니다.
- [x] `16343e76b54b`의 owned/borrowed 주소 관계를 실제 선언으로 증명했습니다.
- [x] `1d69df7db78c`의 double-destroy 가능 경로를 두 cleanup owner로 설명했습니다.
- [x] `10665e0a5bf9`에서 authoritative ledger consumer가 하나가 되는 변경을 제시했습니다.
- [x] `800408d6d84e`의 failure index와 destruction address assertion을 설명했습니다.
- [x] `a7783d04107f`의 started/joined/destruction verdict와 `_exit`를 호출 흐름으로 설명했습니다.
- [x] `7586b605302b`에서 각 failure index의 expected ledger를 계산했습니다.
- [x] `37b29557cccc`의 negative assertions가 normal teardown 부재를 어떻게 증명하는지 설명했습니다.
- [x] exact-once cleanup과 graceful cleanup을 혼동하지 않았습니다.
- [x] successful join을 borrower quiescence의 증거로 사용하는 이유를 설명할 수 있습니다.

## 12. 저장소 기반 완료 기록

### 12.1 검토 범위와 증거 등급

- 검토 브랜치는 `c/philo` 하나로 제한했습니다.
- 이 Thread의 7개 SHA는 모두 브랜치 HEAD `12b29d75ccc98311cd8da1217ababbe21de64026`의 조상이며, 각 비교에서 merge base가 해당 SHA와 일치했습니다.
- 아래 구현 설명은 각 SHA의 commit diff와 그 SHA의 파일을 확인한 결과입니다. 이후 commit의 구현을 이전 상태에 소급하지 않았습니다.
- 저장소 checkout이 로컬 네트워크 제한으로 불가능했으므로 production binary와 test target은 실행하지 않았습니다. 테스트 결과로 적은 내용은 test source의 injection·assertion을 분석한 결과가지 실제 실행 통과 기록이 아닙니다.

### 12.2 `16343e76b54b` — ownership graph와 ring identity

#### 실제 변경 위치

- `include/philo.h`: `t_config`, `t_philo`, `t_table`의 중심 저장 구조를 정의합니다.
- `src/init.c`: `philo_table_init`, philosopher별 pointer 배치, allocation 실패 rollback을 구현합니다.
- `src/destroy.c` 또는 해당 SHA의 table destruction 구현: table-owned allocation을 해제하고 pointer를 초기 상태로 되돌립니다.

#### 소유 관계

| 대상 | 실제 표현 | 관계 | 수명 조건 |
| --- | --- | --- | --- |
| table | `main` 쪽 stack-resident `t_table` | process-level 중심 값 | worker가 table을 빌리지 않는 시점까지 살아 있어야 합니다. |
| fork 저장소 | `t_table.forks` | table이 `malloc`으로 소유 | 모든 philosopher의 fork pointer보다 오래 살아 있어야 합니다. |
| philosopher 저장소 | `t_table.philos` | table이 `malloc`으로 소유 | 생성한 worker와 join이 끝날 때까지 살아 있어야 합니다. |
| table 참조 | `t_philo.table` | borrowed pointer | table destruction 전까지만 유효합니다. |
| fork 참조 | `t_philo.left_fork`, `right_fork` | fork 배열 내부를 빌림 | fork 배열 free 또는 mutex destruction 전까지만 유효합니다. |

초기 header에서 `t_philo`는 `id`, `meals`, `last_meal_ms`, `thread` 값을 직접 가지며, 세 pointer만 table-owned storage를 빌립니다. `t_table`은 configuration과 shared state, `forks`, `philos`, mutex 값의 owner입니다.

#### ring mapping

`src/init.c`의 philosopher 초기화는 philosopher `i`에 다음 주소를 배치합니다.

```c
left_fork  = &table->forks[i];
right_fork = &table->forks[(i + 1) % table->config.number];
```

따라서 philosopher `i`의 `right_fork`와 philosopher `(i + 1) % N`의 `left_fork`는 같은 객체를 가리킵니다. 마지막 philosopher의 오른쪽 포크는 modulo 연산으로 `forks[0]`에 연결됩니다. 포크를 philosopher 구조체 안에 값으로 복사하지 않기 때문에 이웃이 같은 mutex identity를 공유합니다.

#### construction과 rollback

`philo_table_init`은 table을 초기 상태로 만든 뒤 fork 배열과 philosopher 배열을 순서대로 할당합니다. 뒤쪽 allocation이 실패하면 table-level destructor로 이동하며, 이미 존재하는 pointer만 free하고 NULL로 되돌립니다. 이 SHA에서는 mutex staged initialization이 아직 없으므로 보장 범위는 allocation과 topology까지입니다.

### 12.3 `1d69df7db78c` — staged mutex construction과 split rollback 결함

#### 도입된 ledger

- `state_ready`: state mutex의 성공적인 초기화를 나타냅니다.
- `print_ready`: print mutex의 성공적인 초기화를 나타냅니다.
- `fork_count`: 성공적으로 초기화된 fork mutex prefix의 개수입니다.
- backing-array pointer: allocation 소유권의 존재를 나타냅니다.

초기화 순서는 state mutex, print mutex, fork mutex `0..N-1`입니다. 각 `pthread_mutex_init`이 성공한 직후에만 대응 flag 또는 count가 갱신됩니다. 요청한 최종 자원 수가 아니라 실제 성공한 횟수가 cleanup authorization이 됩니다.

#### 남은 결함

`init_forks`는 fork `k` 초기화 실패 시 이미 성공한 `0..k-1`을 로컬 loop에서 파괴합니다. 그러나 `table->fork_count`는 성공한 개수 `k`를 그대로 유지합니다. 상위 `philo_table_init`이 실패 후 common destructor를 호출하면 destructor가 같은 `0..k-1`을 다시 owned로 해석해 두 번째 `pthread_mutex_destroy`를 시도할 수 있습니다.

```text
fork 0..k-1 init 성공
→ fork k init 실패
→ init_forks가 0..k-1 로컬 destroy
→ fork_count == k 유지
→ common destructor가 k개를 다시 destroy
```

문제는 rollback 코드의 존재가 아니라 동일 자원의 cleanup 권한이 helper와 owner destructor 양쪽에 있다는 점입니다.

### 12.4 `10665e0a5bf9` — common destructor 단독 소비

이 fix는 `init_forks`의 local destroy loop를 제거합니다. helper는 실패 상태만 반환하고, `philo_table_destroy`가 `fork_count`를 가진 유일한 rollback owner가 됩니다.

- fork mutex는 initialized prefix를 역순으로 파괴합니다.
- 성공적으로 파괴한 항목만 ledger에서 제거합니다.
- state/print readiness도 destructor가 소비합니다.
- allocation을 free한 뒤 pointer를 NULL로 바꿔 repeated cleanup이 같은 free를 반복하지 않게 합니다.

이 시점의 핵심 복구는 **한 ledger에 한 consumer**입니다. 다만 pthread destroy 실패를 retry 가능한 상태로 보존하는 완전한 모델은 아직 `a7783d04107f` 전입니다.

### 12.5 `800408d6d84e` — partial-init deterministic regression

#### test mechanism

`tests/init_failure.c`는 compile-time function substitution으로 production initializer가 호출하는 `pthread_mutex_init`과 `pthread_mutex_destroy`를 wrapper로 바꿉니다. production source에 test-only branch를 넣지 않습니다.

초기화 호출 순서는 다음과 같습니다.

| 호출 index | 대상 | 결과 |
| --- | --- | --- |
| 1 | state mutex | 성공 |
| 2 | print mutex | 성공 |
| 3 | fork 0 | 성공 |
| 4 | fork 1 | 주입 실패 |

따라서 실패 시 authoritative ledger가 소유한다고 말하는 mutex는 정확히 3개입니다. destroy wrapper는 파괴 요청 주소를 기록하고 같은 주소가 두 번 들어오는지 검사합니다.

#### assertion

- initializer가 오류를 반환합니다.
- destroy call 수는 3입니다.
- 세 주소는 state, print, fork 0에 해당하며 duplicate가 없습니다.
- backing arrays는 해제되고 pointer는 NULL입니다.
- 동일 table에 destructor를 다시 호출해도 destroy call 수가 늘지 않습니다.

이 테스트는 특정 partial-init 경계에서 exact-once ledger 소비를 결정적으로 검증하도록 작성됐습니다. 실제 pthread 구현의 모든 실패 형태, allocation failure matrix, worker lifecycle은 증명하지 않습니다.

### 12.6 `a7783d04107f` — worker quiescence를 destruction permission으로 확장

#### 새 lifecycle state

`include/philo.h`에 다음 상태가 추가됩니다.

- `threads_started`: `pthread_create`가 성공한 handle 수
- `threads_joined`: `pthread_join`이 성공한 handle 수
- `destroy_safe`: shared table destruction 허용 여부
- `PHILO_UNSAFE`: ordinary operation error와 구분되는 safety verdict

초기 상태는 started/joined가 0이고 destruction이 허용된 상태입니다. creation 성공 직후에만 `threads_started`가 증가합니다.

#### join은 시도가 아니라 증거입니다

`src/run.c`의 `join_started`는 recorded prefix 전체에 join을 시도합니다. 성공한 경우에만 `threads_joined`를 증가시킵니다. 하나라도 실패하면 `destroy_safe = 0`, 최종 상태는 `PHILO_UNSAFE`가 됩니다. 실패 후 나머지 handle에 대한 join 시도는 계속하지만, failed handle을 종료됐다고 가정하지 않습니다.

```text
pthread_join(handle) == 0
    → threads_joined++
pthread_join(handle) != 0
    → destroy_safe = 0
    → PHILO_UNSAFE 유지
```

creation 또는 barrier의 ordinary error와 join failure가 함께 발생하면 `PHILO_UNSAFE`가 우선합니다. 이는 오류 심각도 표현이 아니라 shared borrower의 quiescence를 입증하지 못했다는 뜻입니다.

#### destruction predicate

`philo_table_destroy`는 다음 상태에서 shared mutex 또는 allocation을 건드리지 않습니다.

```text
destroy_safe == 0
또는
threads_started != threads_joined
```

started worker는 `t_philo.table`, fork-array 내부 주소, shared mutex를 계속 역참조할 수 있습니다. 실패한 join 뒤 table을 free하거나 mutex를 파괴하면 해제 후 사용 또는 live synchronization-object destruction이 될 수 있으므로 cleanup 자체를 거부합니다.

#### retryable destroy

fork/condition/state/print resource는 destroy call 성공 후에만 count 또는 readiness flag를 지웁니다. 중간 단계에서 pthread destroy가 실패하면 아직 release되지 않은 자원의 ledger가 그대로 남습니다. caller가 안전한 상태에서 destructor를 다시 호출하면 남은 지점부터 재시도할 수 있습니다.

#### unsafe process path

`src/main.c`는 `philo_run`이 `PHILO_UNSAFE`를 반환하면 fixed diagnostic을 unbuffered `write`로 남긴 뒤 `_exit(PHILO_ERR)`를 호출합니다. 이 branch는 다음을 실행하지 않습니다.

- `philo_table_destroy`
- buffered stdio flush
- `atexit` handler
- stack unwinding을 전제로 한 normal `return`/`exit`

이 경로는 graceful cleanup을 실패한 것이 아니라, safety evidence가 없는 cleanup을 의도적으로 금지한 결과입니다.

### 12.7 `7586b605302b` — create/join/destroy failure matrix

`tests/lifecycle_failure.c`는 pthread create, join, mutex destroy를 wrapper로 대체하고 여러 partial-state 위치를 반복합니다.

#### create failure

3-worker configuration에서 create failure index 0, 1, 2를 주입합니다. 기대값은 성공한 prefix만 `threads_started`에 포함되고 그 prefix만 실제 join 대상이 된다는 것입니다. join이 모두 성공하면 ordinary `PHILO_ERR`이며 cleanup은 허용됩니다.

#### join failure

특정 started handle의 join을 실패시킨 뒤에도 production helper가 나머지 handle을 계속 join하는지 확인합니다. 결과는 다음과 같습니다.

- 반환값: `PHILO_UNSAFE`
- `threads_joined`: 성공한 join 수만 반영
- `destroy_safe`: false
- destructor 호출 전후: fork pointer, `fork_count`, readiness flag, destroy-call count가 동일

테스트는 failed worker를 real `pthread_join`으로 사후 정리한 뒤 test-only로 ledger를 복구하고 production destructor를 다시 호출합니다. 이 수선 절차는 production이 failed join을 안전하다고 간주한다는 뜻이 아니라 테스트 프로세스가 남은 thread를 회수하기 위한 장치입니다.

#### destroy failure와 retry

reverse cleanup의 여러 stage에 failure를 주입합니다. 첫 호출은 failure 이전까지 성공한 resource만 ledger에서 제거하고 실패 대상 및 이후 resource를 owned 상태로 유지해야 합니다. injection을 해제한 두 번째 호출이 남은 cleanup을 끝내는지 확인합니다.

이 matrix는 zero/partial/nearly-complete lifecycle arithmetic과 retry state를 직접 자극합니다. 실제 OS가 모든 pthread error를 같은 방식으로 발생시킨다는 것은 증명하지 않습니다.

### 12.8 `37b29557cccc` — process-level forbidden-cleanup regression

`tests/main_unsafe.c`는 real `main`을 사용하되 parse/init/run/destroy를 stub으로 대체합니다.

- parse stub: `atexit` handler를 등록합니다.
- run stub: buffered stdout marker를 기록하고 `PHILO_UNSAFE`를 반환합니다.
- destroy stub: 호출되면 별도 marker를 남깁니다.
- real main unsafe branch: unbuffered join diagnostic 후 `_exit`합니다.

child output에서 반드시 존재해야 하는 것은 unsafe diagnostic입니다. 반드시 없어야 하는 것은 destroy marker, buffered stdout marker, `atexit` marker입니다. 세 negative assertion은 각각 destructor 호출, stdio flush, normal exit handler 실행을 탐지합니다.

따라서 이 테스트가 겨냥하는 항상 유지해야 하는 조건은 단순한 “destructor를 호출하지 않습니다”가 아니라 **normal process teardown 전체를 우회합니다**입니다. 실제 `pthread_join`을 실패시키거나 failed worker의 상태를 증명하지는 않습니다.

### 12.9 항상 유지해야 하는 조건 evolution 완성

| 항상 유지해야 하는 조건 | 도입 | 부족함 노출 | 복구·확장 | regression evidence |
| --- | --- | --- | --- | --- |
| table이 allocation을 소유하고 philosopher가 내부 주소를 빌림 | `16343e76b54b` | worker가 생기면 storage lifetime이 join에 종속됨 | `a7783d04107f`의 quiescence predicate | `7586b605302b`, `37b29557cccc` |
| 성공한 초기화만 cleanup authorization을 만듦 | `1d69df7db78c` | helper와 destructor의 split rollback | `10665e0a5bf9`의 단일 consumer | `800408d6d84e` |
| initialized mutex는 최대 한 번 파괴 | ledger 의도는 `1d69df7db78c` | local rollback 뒤 stale `fork_count` | common destructor 역순 소비 | destruction-address duplicate 검사 |
| destroy 실패 뒤 ledger는 truthful·retryable | `a7783d04107f` | 성공 전에 flag/count를 지우면 재시도 불가 | destroy 성공 후에만 mutation | `7586b605302b` destroy matrix |
| shared storage 파괴 전 모든 borrower quiescence 필요 | 소유권 graph에서 암묵적 위험 | failed join이 종료 증거가 아님 | started/joined + `destroy_safe` + `PHILO_UNSAFE` | lifecycle 및 main unsafe tests |
| unsafe process는 normal teardown을 실행하지 않음 | `a7783d04107f` | destructor 생략만으로 stdio/atexit는 남음 | `_exit` | `37b29557cccc`의 세 negative marker |

### 12.10 최종 실행 순서

```text
main의 t_table
    ↓ philo_table_init
allocation + mutex/condition readiness ledger
    ↓ philo_run
pthread_create 성공마다 threads_started 증가
    ↓ start release / monitor / abort publication
recorded handle 전체에 pthread_join 시도
    ├─ 모든 started handle join 성공
    │      ↓ threads_started == threads_joined
    │      ↓ destroy_safe == 1
    │      ↓ philo_table_destroy
    │      ↓ 성공한 destroy 뒤에만 ledger 소비
    │      ├─ 완료: allocation free + pointer NULL
    │      └─ 중간 실패: remaining ledger 보존, 명시적 retry 가능
    └─ 하나 이상 join 미입증
           ↓ destroy_safe = 0
           ↓ PHILO_UNSAFE가 ordinary error보다 우선
           ↓ table destruction 금지
           ↓ write diagnostic
           ↓ _exit(PHILO_ERR)
```

### 12.11 최종 보장과 비보장

보장하는 범위:

- table-owned allocation과 synchronization object는 recorded 소유권에 따라 처리됩니다.
- partial mutex initialization은 common destructor 하나가 exact-once로 rollback합니다.
- successful join으로 모든 started borrower의 quiescence가 입증된 경우에만 destruction을 시도합니다.
- resource destroy 실패는 아직 owned인 state를 보존해 재시도를 허용합니다.
- unsafe join verdict에서는 forbidden cleanup과 normal process teardown을 실행하지 않습니다.

보장하지 않는 범위:

- failed join worker가 실제로 종료됐는지 추정하지 않습니다.
- unsafe branch에서 graceful cleanup이나 leak-free exit를 보장하지 않습니다.
- wrapper 기반 test가 실제 pthread 구현의 모든 오류 조합을 재현한다고 보지 않습니다.
- 이 Thread는 scheduler fairness, 기아 상태 freedom, 모든 interleaving의 안전성을 증명하지 않습니다.
===== END FILE: 01-소유권-ledger-to-unsafe-destruction.md =====

===== BEGIN FILE: 02-wall-clock-to-shared-monotonic-start.md =====
# Thread: Wall-clock helper to one shared monotonic start epoch

이 문서는 source에 정의된 두 번째 Development Thread를 그대로 따릅니다. commit 순서, SHA, importance, tags는 변경하지 않습니다. time field와 barrier field는 반드시 해당 SHA에서 확인하고 final HEAD의 완성된 protocol을 이전 commit에 소급하지 않습니다.

## 1. Thread 목표

이 Thread의 목표는 단순한 millisecond helper가 어떻게 monotonic elapsed-time model로 교정되고, 다시 worker readiness barrier와 결합해 모든 philosopher가 하나의 valid start epoch를 공유하게 되는지 복원하는 것입니다.

Source-confirmed significance는 다음과 같습니다.

- 초기 helper는 time logic을 한곳에 모으고 terminal-aware polling sleep을 제공하지만 wall clock과 unchecked failure를 사용합니다.
- near-deadline polling refinement는 program-controlled overshoot를 줄이지만 time source나 scheduler guarantee를 바꾸지 않습니다.
- `CLOCK_MONOTONIC`과 `int64_t`가 elapsed-time arithmetic을 calendar correction 및 platform `long` width와 분리합니다.
- start barrier는 thread object creation과 worker readiness를 구분하고, 모든 worker가 ready인 뒤 하나의 timestamp와 initial 기아 상태 reference를 publish합니다.
- partial creation과 condition-wait failure도 release predicate를 publish하여 barrier가 failure 교착 상태로 변하지 않게 합니다.
- tests는 clock-domain correctness, delayed-start regression, wait-failure propagation을 각각 다른 technique으로 고정합니다.

### Source에 명시적으로 연결된 Critical 항상 유지해야 하는 조건

- 모든 worker는 creation 또는 scheduling delay와 무관하게 같은 published monotonic start epoch와 initial `last_meal_ms`에서 시작합니다.
- elapsed time과 기아 상태 decision은 monotonic clock을 사용하며 clock acquisition failure를 fabricated time으로 대체하지 않습니다.
- barrier predicate, `ready_count`, `start_released`, `run_error`의 관찰과 변경은 정의된 `state_mutex` 경계에서 수행합니다.

### Source에 명시적으로 연결된 Major Engineering Difficulties

- successful thread creation과 actual worker readiness를 구분하는 문제
- partial start 및 condition-wait failure 상황에서도 common temporal origin 또는 abort release를 일관되게 publish하는 문제
- timing correctness를 scheduler fairness, 기아 상태 freedom, strict latency guarantee로 과장하지 않는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 처음 time abstraction은 어떤 중복을 제거하고 어떤 clock-domain 문제를 그대로 남기는가?
- deadline polling sleep은 terminal responsiveness와 wakeup precision 사이에서 어떤 감수할 점을 선택하는가?
- wall clock adjustment가 기아 상태, deadline, log offset에 어떤 잘못된 상태 전이를 만들 수 있는가?
- monotonic time과 fixed-width field가 runtime 전 영역에 어떻게 전파됩니까?
- clock acquisition failure가 왜 ordinary recoverable error가 아니라 process-fatal입니까?
- `pthread_create` 성공과 worker readiness가 왜 다른 lifecycle event입니까?
- common timestamp는 어떤 lock boundary와 predicate transition에서 publish됩니까?
- condition variable의 spurious wakeup과 lost notification 문제를 predicate loop가 어떻게 다루는가?
- barrier failure가 peer release, joining, final return status로 어떻게 전파됩니까?
- tests가 clock source, startup skew, wait failure를 각각 어떻게 결정적으로 구성하는가?

## 3. 완료 기준

- [x] `philo_now_ms`와 `philo_sleep_ms`의 최초 구현과 당시 한계를 해당 SHA에서 설명할 수 있습니다.
- [x] polling quantum refinement가 바꾸는 것과 바꾸지 않는 것을 구분할 수 있습니다.
- [x] wall-clock failure scenario와 monotonic correction을 call-site 전파까지 설명할 수 있습니다.
- [x] `int64_t`로 바뀐 timing state를 실제 field와 signature로 제시할 수 있습니다.
- [x] clock failure의 `write` + `_exit` 경로를 설명할 수 있습니다.
- [x] barrier의 readiness, publication, release, abort 상태 전이를 실제 field로 그릴 수 있습니다.
- [x] common timestamp가 table 및 모든 philosopher에 설정된 뒤 worker가 fork activity를 시작함을 증명할 수 있습니다.
- [x] delayed-start test의 150 ms와 80 ms 관계가 regression을 민감하게 만드는 이유를 설명할 수 있습니다.
- [x] wait-failure test가 peer 교착 상태를 bounded error로 바꾸는 경로를 설명할 수 있습니다.
- [x] monotonic/common epoch guarantee와 fairness/strict latency non-guarantee를 구분할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `509453b01515` | `feat(time): 밀리초 시각 계산 함수 추가` | B | `TIME_MODEL, CORE` | Centralizes millisecond time and interruptible deadline waits, initially with `gettimeofday`. |
| 2 | `a21e4cc75272` | `fix(time): 짧은 대기 시간의 초과 지연 완화` | B | `TIME_MODEL, PRACTICAL` | Reduces final-interval polling granularity for short waits. |
| 3 | `5b32d5bdb955` | `fix(time): 단조 시계로 경과 시간 계산` | A | `TIME_MODEL, RISK, CORE` | Replaces wall time with `CLOCK_MONOTONIC`, widens time state, and makes clock failure fatal. |
| 4 | `f01d62cde8ce` | `test(time): 단조 시계와 시계 실패 경로 검증` | B | `TEST, TIME_MODEL` | Verifies the monotonic clock identifier, conversion, and failure exit. |
| 5 | `e7e62cbe185f` | `fix(thread): 시작 장벽으로 기준 시각 통일` | S | `START_BARRIER, CONCURRENCY, TIME_MODEL` | Adds a readiness barrier and publishes one start timestamp to all workers after they are actually ready. |
| 6 | `bfbfa0431732` | `test(thread): 지연된 작업자의 공통 시작 시각 검증` | A | `TEST, START_BARRIER, EDGE` | Deliberately delays one worker and verifies that the shared release prevents pre-start starvation accounting. |
| 7 | `f57f6ec0be87` | `test(thread): 시작 대기 실패 전파 검증` | B | `TEST, START_BARRIER, RESOURCE_LIFECYCLE` | Injects a condition-wait failure and checks that the barrier aborts and propagates the error. |

## 5. Commit별 학습 기록

### 5.1 `509453b01515` — `feat(time): 밀리초 시각 계산 함수 추가`

- Importance: **B**
- Tags: `TIME_MODEL, CORE`
- Source-defined role: Centralizes millisecond time and interruptible deadline waits, initially with `gettimeofday`.
- 코드 기준: 반드시 `509453b01515` 시점
- 직접 parent 비교: `git diff 509453b01515^ 509453b01515 --`
- Thread 직전 관련 SHA: Thread 내 첫 commit

#### Source-confirmed 맥락

이 B-level commit은 `philo_now_ms`와 `philo_sleep_ms`로 time access를 중앙화합니다. current time, start time, last-meal time, log offset, deadline을 밀리초 표현으로 다루며, sleep은 한 번의 긴 block 대신 deadline loop로 구현됩니다. 각 iteration은 `state_mutex` 경계에서 terminal flag를 확인하고, 종료되지 않았다면 500 microsecond 단위로 양보합니다.

이 SHA의 clock은 `gettimeofday`입니다. wall-clock adjustment를 그대로 받으며 clock call failure도 처리하지 않습니다. 또한 common worker start epoch은 아직 없으므로 이 abstraction의 도입과 최종 time model을 구분해야 합니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 문제 | worker, monitor, logger에 흩어진 time conversion과 uninterruptible wait는 일관된 elapsed-time reasoning과 responsive shutdown을 어렵게 합니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | time acquisition과 deadline wait를 공통 helper로 묶고 terminal-aware polling sleep을 사용합니다. | §12 완료 기록의 대응 근거 참조 |
| 즉시 얻는 역할 | start, last meal, deadline, log timestamp가 같은 millisecond representation을 사용합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 한계 | wall clock, unchecked clock failure, sequential worker startup skew가 남습니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `git show --name-status 509453b01515`로 time helper와 public declaration이 추가된 파일을 식별합니다.
- [x] `philo_now_ms`가 seconds와 microseconds를 millisecond로 변환하는 실제 산식을 확인합니다.
- [x] 반환 type과 table/philosopher timing field의 당시 type을 기록합니다.
- [x] `philo_sleep_ms`가 deadline을 계산하고 current time을 반복 비교하는 순서를 확인합니다.
- [x] 각 iteration에서 terminal flag를 읽을 때 실제로 어떤 helper 또는 `state_mutex` 경계를 사용하는지 확인합니다.
- [x] 500-microsecond `usleep` 호출과 loop 종료 조건을 확인합니다.
- [x] clock call의 반환값이 무시되는지, 실패 시 어떤 값이 사용될 수 있는지 확인합니다.
- [x] 이 SHA에서 start time이 언제 설정되는지는 orchestration code가 아직 없을 수 있으므로 실제 호출 지점 범위를 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Time state 추적

| 값 또는 operation | 해당 SHA의 producer | consumer 또는 비교 지점 | clock domain / type | 학습자 근거 |
| --- | --- | --- | --- | --- |
| current milliseconds | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | wall clock /  | §12 완료 기록의 대응 근거 참조 |
| sleep deadline | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| terminal flag polling | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | `state_mutex` boundary | §12 완료 기록의 대응 근거 참조 |
| configured duration | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- time acquisition과 interruptible deadline wait의 공통 abstraction이 생깁니다.
- terminal state가 설정되면 worker wait가 전체 configured interval을 계속 잠들 필요가 없습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- wall-clock adjustment로부터 elapsed time을 보호하지 않습니다.
- clock acquisition failure를 처리하지 않습니다.
- 모든 worker가 같은 실제 start epoch에서 출발한다고 보장하지 않습니다.
- real-time wakeup 또는 strict latency를 보장하지 않습니다.

#### 학습자 결론
- [x] 왜 deadline loop가 단일 `usleep(duration)`보다 terminal responsiveness에 유리한지 설명합니다.
- [x] wall time을 elapsed-time truth로 사용했을 때 가능한 backward/forward adjustment 영향을 적습니다.
- [x] abstraction의 도입과 correct clock source의 선택을 별도 단계로 설명합니다.

### 5.2 `a21e4cc75272` — `fix(time): 짧은 대기 시간의 초과 지연 완화`

- Importance: **B**
- Tags: `TIME_MODEL, PRACTICAL`
- Source-defined role: Reduces final-interval polling granularity for short waits.
- 코드 기준: 반드시 `a21e4cc75272` 시점
- 직접 parent 비교: `git diff a21e4cc75272^ a21e4cc75272 --`
- Thread 직전 관련 SHA: `509453b01515`

#### Source-confirmed 맥락

이 B-level fix는 기존 cooperative polling model을 유지하면서 deadline까지 남은 시간이 1 millisecond를 초과하면 500 microseconds, 마지막 구간이면 100 microseconds로 polling interval을 줄입니다. 목적은 짧은 configured duration과 near-deadline wakeup에서 프로그램 자체가 추가하는 overshoot를 줄이는 것입니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 상태 | 모든 남은 시간 구간에서 500-microsecond polling quantum을 사용합니다. | §12 완료 기록의 대응 근거 참조 |
| 관찰된 한계 | 짧은 wait에서는 고정 quantum이 전체 duration에 비해 커서 avoidable overshoot 비중이 커질 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 결정 | remaining time에 따라 500 또는 100 microseconds를 선택합니다. | §12 완료 기록의 대응 근거 참조 |
| 유지되는 한계 | scheduler latency와 real-time guarantee 부재는 그대로입니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `509453b01515` 대비 sleep loop의 조건식과 `usleep` argument 변화만 분리해 확인합니다.
- [x] remaining time이 어떤 type과 산식으로 계산되는지 확인합니다.
- [x] 1-millisecond 경계에서 어떤 branch가 선택되는지 `remaining == 1`, `< 1`, `> 1` 상황으로 나눠 기록합니다.
- [x] terminal flag 확인 빈도와 lock 경계가 변경되지 않았는지 확인합니다.
- [x] deadline 도달 판정이 polling branch 앞뒤 중 어디에 있는지 확인합니다.
- [x] polling interval 조정이 clock source나 worker start semantics를 바꾸지 않는다는 것을 diff로 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- deadline 마지막 구간에서 프로그램이 선택하는 polling sleep 단위를 줄여 avoidable overshoot를 완화합니다.
- terminal-aware polling 구조는 유지됩니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- scheduler가 정확한 시각에 thread를 재개한다고 보장하지 않습니다.
- wall-clock source 문제와 common start skew를 해결하지 않습니다.
- strict death-detection latency 또는 real-time scheduling을 보장하지 않습니다.

#### 학습자 결론
- [x] 이 commit이 time model 변경이 아니라 local precision refinement인 이유를 설명합니다.
- [x] 각 branch의 sleep quantum과 remaining-time 경계를 실제 조건식으로 제시합니다.
- [x] 개선 가능한 program-controlled delay와 통제할 수 없는 scheduler delay를 구분합니다.

### 5.3 `5b32d5bdb955` — `fix(time): 단조 시계로 경과 시간 계산`

- Importance: **A**
- Tags: `TIME_MODEL, RISK, CORE`
- Source-defined role: Replaces wall time with `CLOCK_MONOTONIC`, widens time state, and makes clock failure fatal.
- 코드 기준: 반드시 `5b32d5bdb955` 시점
- 직접 parent 비교: `git diff 5b32d5bdb955^ 5b32d5bdb955 --`
- Thread 직전 관련 SHA: `a21e4cc75272`

#### Source-confirmed 맥락

이 A-level fix는 elapsed-time source를 `gettimeofday`에서 `clock_gettime(CLOCK_MONOTONIC)`으로 교체하고 timing field와 parser intermediate를 `int64_t`로 통일합니다. simulation start, last meal, sleep deadline, monitor 기아 상태 decision, log offset이 calendar adjustment의 영향을 받지 않는 하나의 clock domain을 사용합니다.

monotonic clock을 얻지 못하면 fixed diagnostic을 `write`로 출력하고 `_exit`합니다. 모든 worker와 monitor decision이 동일 time source에 의존하므로 fabricated 또는 stale timestamp로 계속 실행하지 않는 결정입니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | wall clock이 elapsed-time ordering을 안정적으로 제공한다고 간주했습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | calendar correction이 backward/forward jump를 만들어 기아 상태, wait, log offset을 왜곡할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | civil time과 elapsed time을 같은 source로 취급했습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 결정 | `CLOCK_MONOTONIC`과 fixed-width millisecond state를 사용합니다. | §12 완료 기록의 대응 근거 참조 |
| failure decision | clock source 부재는 simulation correctness를 유지할 수 없으므로 process-fatal입니다. | §12 완료 기록의 대응 근거 참조 |
| 후속 한계 | worker가 실제로 준비되기 전에 start timestamp가 잡히는 문제는 `e7e62cbe185f`까지 남습니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `a21e4cc75272` 대비 header의 timing field와 parser intermediate type이 `int64_t`로 바뀌는 모든 위치를 찾습니다.
- [x] `philo_now_ms`가 `clock_gettime`에 전달하는 clock identifier를 확인합니다.
- [x] `timespec.tv_sec`와 `tv_nsec`의 millisecond conversion 산식을 확인합니다.
- [x] monitor, logger, sleeper, parser의 function signature와 format conversion이 fixed-width type에 맞게 변경되는지 추적합니다.
- [x] log formatting에서 explicit cast 또는 format type이 어떻게 처리되는지 해당 SHA 코드로 확인합니다.
- [x] `clock_gettime` failure branch의 fixed message, `write` destination, `_exit` status를 확인합니다.
- [x] fatal path가 ordinary cleanup이나 cross-thread unwinding을 시도하지 않는지 확인합니다.
- [x] start timestamp의 sampling 위치가 아직 thread creation 전 또는 during creation인지 확인하여 barrier fix 전 상태를 기록합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Clock-domain migration 기록

| 영역 | 변경 전 | `5b32d5bdb955` | 실제 파일·symbol |
| --- | --- | --- | --- |
| current time source | `gettimeofday` | `CLOCK_MONOTONIC` | §12 완료 기록의 대응 근거 참조 |
| time field type | host `long` 기반 | `int64_t` | §12 완료 기록의 대응 근거 참조 |
| sleep deadline | §12 완료 기록의 대응 근거 참조 | monotonic millisecond domain | §12 완료 기록의 대응 근거 참조 |
| 기아 상태 comparison | §12 완료 기록의 대응 근거 참조 | monotonic millisecond domain | §12 완료 기록의 대응 근거 참조 |
| log offset | §12 완료 기록의 대응 근거 참조 | monotonic millisecond domain | §12 완료 기록의 대응 근거 참조 |
| clock failure | unchecked | diagnostic + `_exit` | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- elapsed-time, 기아 상태, deadline, log offset이 calendar adjustment와 분리된 monotonic domain을 사용합니다.
- timing state의 intended numeric width가 platform `long` width와 분리됩니다.
- clock acquisition failure가 unchecked time으로 runtime을 계속 진행하지 않습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 worker가 같은 readiness 시점에서 start budget을 받는 것은 아직 보장하지 않습니다.
- monotonic clock은 scheduler fairness나 exact wakeup latency를 보장하지 않습니다.

#### Regression 연결

- `f01d62cde8ce`는 requested clock identifier, known `timespec` conversion, fatal failure exit를 검증합니다.
- `e7e62cbe185f`는 monotonic timestamp가 concurrency 상으로도 유효한 common start epoch가 되도록 barrier를 추가합니다.


#### 학습자 결론
- [x] civil time과 elapsed time을 분리해야 하는 이유를 기아 상태 예시로 설명합니다.
- [x] 모든 timing field가 같은 domain과 type을 사용한다는 것을 call-site 목록으로 증명합니다.
- [x] clock failure를 ordinary error return이 아니라 process-fatal로 취급하는 source-confirmed 판단을 설명합니다.
- [x] monotonic source만으로 sequential-start skew가 해결되지 않는 이유를 설명합니다.

### 5.4 `f01d62cde8ce` — `test(time): 단조 시계와 시계 실패 경로 검증`

- Importance: **B**
- Tags: `TEST, TIME_MODEL`
- Source-defined role: Verifies the monotonic clock identifier, conversion, and failure exit.
- 코드 기준: 반드시 `f01d62cde8ce` 시점
- 직접 parent 비교: `git diff f01d62cde8ce^ f01d62cde8ce --`
- Thread 직전 관련 SHA: `5b32d5bdb955`

#### Source-confirmed test 역할

이 test는 compile-time replacement로 `clock_gettime`을 대체합니다. success stub은 requested clock identifier를 기록하고 known `timespec`을 반환하여 정확한 millisecond conversion을 검사합니다. failure mode는 자식 프로세스에서 `philo_now_ms`를 호출하고 parent가 child의 `PHILO_ERR` termination을 요구합니다. intentional `_exit`가 test runner 자체를 종료하지 않도록 process boundary를 사용합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| success stub이 기록하는 값 | §12 완료 기록의 대응 근거 참조 |
| known `timespec`과 expected milliseconds | §12 완료 기록의 대응 근거 참조 |
| failure stub의 동작 | §12 완료 기록의 대응 근거 참조 |
| child process 사용 이유 | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production 함수 | §12 완료 기록의 대응 근거 참조 |
| clock identifier assertion | §12 완료 기록의 대응 근거 참조 |
| conversion assertion | §12 완료 기록의 대응 근거 참조 |
| fatal exit assertion | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic unit/boundary regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] test build에서 `clock_gettime` replacement가 production time object에 어떻게 적용되는지 확인합니다.
- [x] success stub이 clock id와 호출 횟수를 저장하는 자료구조를 확인합니다.
- [x] known seconds/nanoseconds를 expected milliseconds로 직접 계산합니다.
- [x] parent/child 분기와 child의 intentional fatal call을 확인합니다.
- [x] parent가 `wait` 계열 API 결과에서 종료 상태를 어떻게 해석하는지 확인합니다.
- [x] `PHILO_ERR` 값과 `_exit` argument가 실제로 일치하는지 확인합니다.
- [x] test가 wall-clock API 호출 부재까지 직접 검사하는지 여부를 실제 assertion으로 구분합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- `philo_now_ms`가 requested clock으로 `CLOCK_MONOTONIC`을 사용하고 known time을 정확히 millisecond로 바꾸는지 검증합니다.
- clock failure가 unchecked return이 아니라 process-level fatal status로 끝나는지 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- worker start barrier나 multi-thread timing semantics를 검증하지 않습니다.
- scheduler delay, sleep precision, real-world clock implementation의 모든 동작을 검증하지 않습니다.

#### 학습자 결론
- [x] 왜 failure branch를 같은 process에서 직접 호출할 수 없는지 설명합니다.
- [x] stubbed `timespec` 계산과 production conversion을 수식으로 대조합니다.
- [x] 이 test가 time source decision의 두 절반인 정상 domain과 실패 contract를 각각 어떻게 고정하는지 설명합니다.

### 5.5 `e7e62cbe185f` — `fix(thread): 시작 장벽으로 기준 시각 통일`

- Importance: **S**
- Tags: `START_BARRIER, CONCURRENCY, TIME_MODEL`
- Source-defined role: Adds a readiness barrier and publishes one start timestamp to all workers after they are actually ready.
- 코드 기준: 반드시 `e7e62cbe185f` 시점
- 직접 parent 비교: `git diff e7e62cbe185f^ e7e62cbe185f --`
- Thread 직전 관련 SHA: `f01d62cde8ce`

#### Source-confirmed 맥락

이 S-level fix는 thread object creation과 worker readiness를 분리하는 condition-variable barrier를 도입합니다. 각 worker는 `state_mutex`를 잡고 `ready_count`를 증가시킨 뒤 coordinator에 알리고, `start_released` predicate가 true가 될 때까지 loop에서 기다립니다. coordinator는 성공적으로 생성된 worker가 모두 ready가 된 뒤 monotonic timestamp 하나를 sampling하고 table의 `start_ms`와 모든 philosopher의 initial `last_meal_ms`에 배치한 다음 broadcast합니다.

partial creation 또는 condition wait failure에서는 `run_error`, `ended`, `start_released`를 publish하고 peers를 broadcast하여 impossible ready count를 기다리거나 worker를 영구 block하지 않습니다. condition variable의 spurious wakeup 가능성 때문에 notification 횟수가 아니라 predicate loop가 correctness 기준입니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 상태 | start timestamp가 creation loop 전에 잡히고 worker는 각 `pthread_create` 직후 실행을 시작할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | 늦게 생성·scheduled된 worker가 준비되기 전 시간을 기아 상태 budget으로 잃고 false death가 날 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | successful thread creation을 worker readiness 및 common simulation epoch와 동일시했습니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | `ready_count`, `start_released`, `start_cond`, `state_mutex`로 readiness barrier를 구성합니다. | §12 완료 기록의 대응 근거 참조 |
| start linearization | 모든 intended worker가 ready인 상태에서 한 timestamp와 initial last-meal 값을 publish한 뒤 broadcast합니다. | §12 완료 기록의 대응 근거 참조 |
| failure protocol | partial create/wait failure도 terminal·release predicate를 publish하여 peers가 빠져나오고 join될 수 있게 합니다. | §12 완료 기록의 대응 근거 참조 |
| condition semantics | signal/broadcast는 stored event가 아니므로 predicate loop가 실제 state를 재검사합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `t_table`에 추가된 `start_cond`, `ready_count`, `start_released`, `run_error`와 readiness flag를 모두 찾습니다.
- [x] condition variable initialization과 destruction이 기존 resource ledger에 어떤 순서로 편입되는지 확인합니다.
- [x] worker entry가 fork acquisition 전에 barrier helper를 호출하는지 확인합니다.
- [x] worker가 `state_mutex` 아래에서 `ready_count`를 증가시키고 coordinator notification을 보내는 순서를 기록합니다.
- [x] `pthread_cond_wait`가 predicate loop 안에 있는지, wakeup 후 어떤 state를 재검사하는지 확인합니다.
- [x] coordinator가 full readiness를 기다리는 predicate와 partial creation failure 시 expected ready count를 어떻게 다루는지 확인합니다.
- [x] 한 monotonic timestamp가 table `start_ms`와 모든 philosopher `last_meal_ms`에 설정되는 loop를 확인합니다.
- [x] timestamp publication, `start_released = true`, broadcast가 어느 lock boundary 안에서 일어나는지 확인합니다.
- [x] worker wait failure와 coordinator wait failure 각각에서 `run_error`, `ended`, `start_released`, broadcast가 어떻게 설정되는지 추적합니다.
- [x] abort된 worker가 barrier를 빠져나온 뒤 fork activity 없이 종료하고 join 가능한지 확인합니다.
- [x] creation failure, wait failure, monitor entry, join result 사이의 return precedence를 기록합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Barrier 상태 머신

| 상태 | predicate / count | 허용되는 transition | mutation owner와 lock | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| worker created, not ready | §12 완료 기록의 대응 근거 참조 | ready count 증가 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| worker ready, release 대기 | §12 완료 기록의 대응 근거 참조 | cond wait / spurious wakeup 재검사 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| all intended workers ready | §12 완료 기록의 대응 근거 참조 | common timestamp publication | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| normal release | `start_released` true | worker fork activity 시작 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| partial create abort | §12 완료 기록의 대응 근거 참조 | ended + release + broadcast | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| wait failure abort | `run_error` | ended + release + broadcast | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### Common epoch 증명 기록

- timestamp sampling symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- initial `last_meal_ms`를 설정하는 loop: §12 완료 기록의 대응 근거에 정리했습니다.
- `start_released` publication: §12 완료 기록의 대응 근거에 정리했습니다.
- worker가 publication 후 처음 관찰하는 state: §12 완료 기록의 대응 근거에 정리했습니다.
- barrier 이전에 실행할 수 없는 fork/log operation: §12 완료 기록의 대응 근거에 정리했습니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 모든 successfully created intended worker가 readiness barrier에 도달한 뒤 하나의 monotonic start epoch를 공유합니다.
- thread creation과 scheduling delay before readiness가 기아 상태 budget에 포함되지 않습니다.
- partial creation과 condition-wait failure가 peers를 영구 대기시키지 않고 run-level error로 전파됩니다.
- predicate loop가 spurious wakeup에도 released state를 재검사합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- scheduler fairness, 기아 상태 freedom, strict start latency를 보장하지 않습니다.
- barrier release 이후 각 worker가 동일 시각에 CPU를 얻는다고 보장하지 않습니다.
- 모든 pthread failure 종류를 완전히 처리한다는 뜻은 아닙니다.

#### 후속 regression evidence

- `bfbfa0431732`는 한 worker를 `time_to_die`보다 오래 지연시켜 pre-start time이 starvation으로 계산되지 않는지 검증합니다.
- `f57f6ec0be87`는 worker-side condition wait failure가 run-level error와 peer release로 전파되는지 검증합니다.


#### 학습자 결론
- [x] `pthread_create` 성공과 worker readiness가 다른 사건인 이유를 실제 control flow로 설명합니다.
- [x] common epoch publication의 linearization point를 lock, field mutation, broadcast 순서로 제시합니다.
- [x] 왜 condition variable notification 수가 아니라 predicate state를 기준으로 해야 하는지 설명합니다.
- [x] normal release와 partial-start abort가 같은 barrier state를 어떻게 다르게 종료하는지 설명합니다.
- [x] barrier가 time model, thread lifecycle, 기아 상태 semantics를 동시에 연결하는 이유를 설명합니다.

### 5.6 `bfbfa0431732` — `test(thread): 지연된 작업자의 공통 시작 시각 검증`

- Importance: **A**
- Tags: `TEST, START_BARRIER, EDGE`
- Source-defined role: Deliberately delays one worker and verifies that the shared release prevents pre-start 기아 상태 accounting.
- 코드 기준: 반드시 `bfbfa0431732` 시점
- 직접 parent 비교: `git diff bfbfa0431732^ bfbfa0431732 --`
- Thread 직전 관련 SHA: `e7e62cbe185f`

#### Source-confirmed test 역할

이 deterministic skew test는 `pthread_create`를 wrap하여 모든 created thread를 test gate 뒤에 두고, 다섯 번째 worker에 150-millisecond 추가 delay를 주입합니다. configured `time_to_die`는 80 milliseconds로 delay보다 짧습니다. barrier가 없다면 delayed worker가 ready가 되기 전에 death budget을 소진할 수 있지만, test는 다섯 worker가 모두 one-meal completion으로 성공하고 `ready_count`가 full worker count에 도달했는지 요구합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| 주입하는 startup skew | §12 완료 기록의 대응 근거 참조 |
| `time_to_die`와 delay의 수치 관계 | §12 완료 기록의 대응 근거 참조 |
| test gate가 만드는 실행 순서 | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production barrier 경로 | §12 완료 기록의 대응 근거 참조 |
| full readiness 관찰 | §12 완료 기록의 대응 근거 참조 |
| one-meal completion assertion | §12 완료 기록의 대응 근거 참조 |
| false death 부재 assertion | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `pthread_create` wrapper가 real thread object 생성과 test gate 대기를 어떤 순서로 결합하는지 확인합니다.
- [x] 왜 모든 다섯 thread object가 존재한 뒤 routines가 release되는지 확인합니다.
- [x] 다섯 번째 worker만 150 milliseconds 지연되는 조건을 확인합니다.
- [x] 80-millisecond death budget과 injected delay를 test configuration에서 직접 확인합니다.
- [x] production `ready_count`가 다섯에 도달했다는 assertion을 확인합니다.
- [x] successful finite meal completion을 어떤 return/output/state로 판단하는지 확인합니다.
- [x] death가 없어야 한다는 negative assertion을 확인합니다.
- [x] barrier implementation이 pre-start timestamp를 사용하면 test가 실패하는 경로를 설명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 구성된 startup skew에서 delayed worker가 common release 전 시간 때문에 false death 처리되지 않음을 검증합니다.
- 모든 worker가 production readiness count에 참여하고 one-meal target을 완료하는지 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 가능한 scheduler delay나 barrier interleaving을 포괄하지 않습니다.
- barrier 이후 fairness, strict timing precision, 기아 상태 freedom을 증명하지 않습니다.

#### 학습자 결론
- [x] 이 test가 chance-based repeated run이 아니라 barrier 목적을 직접 겨냥한 이유를 설명합니다.
- [x] delay가 death budget보다 길어야 regression이 민감해지는 이유를 설명합니다.
- [x] test gate와 production start barrier의 역할을 혼동하지 않고 각각의 경계를 설명합니다.

### 5.7 `f57f6ec0be87` — `test(thread): 시작 대기 실패 전파 검증`

- Importance: **B**
- Tags: `TEST, START_BARRIER, RESOURCE_LIFECYCLE`
- Source-defined role: Injects a condition-wait failure and checks that the barrier aborts and propagates the error.
- 코드 기준: 반드시 `f57f6ec0be87` 시점
- 직접 parent 비교: `git diff f57f6ec0be87^ f57f6ec0be87 --`
- Thread 직전 관련 SHA: `bfbfa0431732`

#### Source-confirmed test 역할

이 test는 routine object의 첫 번째 `pthread_cond_wait`를 `EINVAL`로 실패시킵니다. one-shot injection flag 자체는 mutex로 보호되어 test가 새로운 data race를 만들지 않습니다. 이후 wait는 real pthread function에 위임됩니다. run은 external timeout 안에 `PHILO_ERR`로 끝나고 `run_error`를 보존하며 모든 peer를 release해야 합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| 주입하는 API failure | §12 완료 기록의 대응 근거 참조 |
| one-shot flag synchronization | §12 완료 기록의 대응 근거 참조 |
| 실제로 실패하는 worker-side production path | §12 완료 기록의 대응 근거 참조 |
| peer release 경로 | §12 완료 기록의 대응 근거 참조 |
| `run_error` 관찰 | §12 완료 기록의 대응 근거 참조 |
| `PHILO_ERR` propagation | §12 완료 기록의 대응 근거 참조 |
| external timeout의 역할 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic failure regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `pthread_cond_wait` replacement가 routine object에만 적용되는 build 방식을 확인합니다.
- [x] 첫 호출만 `EINVAL`을 반환하고 이후 call은 real function으로 위임하는 조건을 확인합니다.
- [x] injection flag를 보호하는 test mutex와 접근 순서를 확인합니다.
- [x] production worker가 failed wait를 받은 뒤 어떤 helper가 `run_error`, `ended`, `start_released`를 설정하는지 확인합니다.
- [x] broadcast가 peer waiters를 깨우는지 실제 production call path로 추적합니다.
- [x] coordinator 또는 `philo_run`이 `run_error`를 최종 `PHILO_ERR`로 전파하는지 확인합니다.
- [x] external timeout이 hang을 bounded failure로 바꾸는 구성을 확인합니다.
- [x] test가 normal completion이나 false death가 아니라 run-level error를 요구하는 assertion을 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 주입된 worker-side condition wait failure가 barrier 내부에 갇히지 않고 run-level error로 전파됨을 검증합니다.
- peer release와 bounded termination을 검증합니다.
- test injection 자체가 unrelated race를 만들지 않도록 flag access를 synchronization합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 condition-variable API failure 위치를 포괄하지 않습니다.
- scheduler fairness나 정상 barrier의 모든 schedule을 증명하지 않습니다.

#### 학습자 결론
- [x] wait failure를 단순 worker-local return으로 끝내면 peers가 왜 block될 수 있는지 설명합니다.
- [x] `run_error`, `ended`, `start_released`, broadcast의 실제 mutation 순서를 제시합니다.
- [x] test harness synchronization이 production race 검증을 오염시키지 않는 이유를 설명합니다.

## 6. 항상 유지해야 하는 조건 ledger

| 항상 유지해야 하는 조건 | 최초 도입 또는 부족함 | 강화·복구 | regression evidence | 해당 SHA 코드 근거 | 최종 설명 |
| --- | --- | --- | --- | --- | --- |
| time access와 deadline wait가 공통 abstraction을 사용 | `509453b01515` | `a21e4cc75272`에서 near-deadline polling refinement | 별도 deterministic sleep test는 이 Thread source에 없음 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| elapsed-time state는 monotonic clock domain을 사용 | `509453b01515`에서 wall-clock 한계 존재 | `5b32d5bdb955` | `f01d62cde8ce` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| timing state는 fixed-width millisecond representation을 사용 | 초기 host `long` 기반 | `5b32d5bdb955` | `f01d62cde8ce`의 conversion 확인 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| clock acquisition failure는 unchecked time으로 계속 진행하지 않음 | `509453b01515`에서 미처리 | `5b32d5bdb955`의 fatal path | `f01d62cde8ce` child process | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| 모든 worker가 같은 start와 initial starvation reference를 공유 | creation-loop 이전 timestamp로 부족함 | `e7e62cbe185f` | `bfbfa0431732` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| barrier failure는 peer release와 run-level error로 전파 | barrier 도입 시 함께 정의 | `e7e62cbe185f` abort protocol | `f57f6ec0be87` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| condition wait는 notification이 아니라 predicate loop를 기준으로 함 | `e7e62cbe185f` | 동일 commit의 readiness/release protocol | wait-failure test는 failure propagation을 검증 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

## 7. Failure → Fix → Test 연결

### 7.1 Wall clock에서 monotonic elapsed time으로

```text
`509453b01515`
gettimeofday 기반 millisecond helper
→ calendar adjustment와 unchecked failure 위험
→ `5b32d5bdb955`
CLOCK_MONOTONIC + int64_t + fatal clock failure
→ `f01d62cde8ce`
clock id, conversion, child-process failure exit 검증
```

- wall-clock dependency가 있는 실제 code: §12 완료 기록의 대응 근거에 정리했습니다.
- 기아 상태/deadline/log에 전파되는 호출 지점: §12 완료 기록의 대응 근거에 정리했습니다.
- fixed-width migration 범위: §12 완료 기록의 대응 근거에 정리했습니다.
- fatal branch의 process contract: §12 완료 기록의 대응 근거에 정리했습니다.
- test stub의 expected values: §12 완료 기록의 대응 근거에 정리했습니다.
- 이 연결이 보장하지 않는 scheduler 속성: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.2 Created thread에서 ready worker로

```text
start timestamp를 creation 전에 설정
→ delayed worker가 pre-start 시간을 death budget으로 잃음
→ `e7e62cbe185f`
ready_count + start_cond + start_released
→ all ready 상태에서 한 timestamp publish
→ `bfbfa0431732`
150 ms startup skew / 80 ms time_to_die
→ false death 없이 one-meal completion
```

- 기존 start sampling 위치: §12 완료 기록의 대응 근거에 정리했습니다.
- delayed worker가 false death에 이르는 계산: §12 완료 기록의 대응 근거에 정리했습니다.
- barrier의 normal predicate: §12 완료 기록의 대응 근거에 정리했습니다.
- common timestamp publication 순서: §12 완료 기록의 대응 근거에 정리했습니다.
- test gate와 production barrier의 관계: §12 완료 기록의 대응 근거에 정리했습니다.
- test가 증명하지 않는 fairness 범위: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.3 Barrier wait failure

```text
worker가 condition wait 중 API failure
→ worker-local return만 하면 peers가 영구 대기할 위험
→ `e7e62cbe185f`
run_error + ended + start_released + broadcast
→ `f57f6ec0be87`
첫 wait에 EINVAL 주입 + timeout
→ peer release와 PHILO_ERR propagation
```

- failed wait를 받는 actual symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- abort state mutation 순서: §12 완료 기록의 대응 근거에 정리했습니다.
- broadcast 대상: §12 완료 기록의 대응 근거에 정리했습니다.
- coordinator/join/final return 연결: §12 완료 기록의 대응 근거에 정리했습니다.
- timeout이 탐지하는 실패: §12 완료 기록의 대응 근거에 정리했습니다.
- injection flag가 test race를 만들지 않는 근거: §12 완료 기록의 대응 근거에 정리했습니다.

## 8. 소유권 / state / responsibility 변화

| 시점 | Time source 책임 | Start state 책임 | Worker가 관찰하는 predicate | Failure 책임 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| `509453b01515` | common helper, wall clock | orchestration에 아직 common barrier 없음 | terminal flag polling | clock failure 미처리 | §12 완료 기록의 대응 근거 참조 |
| `a21e4cc75272` | 기존 helper 유지 | 변화 없음 | terminal flag polling | 변화 없음 | §12 완료 기록의 대응 근거 참조 |
| `5b32d5bdb955` | monotonic helper가 elapsed-time truth 제공 | 아직 sequential-start skew 존재 | 동일 clock domain | clock failure는 process-fatal | §12 완료 기록의 대응 근거 참조 |
| `e7e62cbe185f` | monotonic timestamp sampling | coordinator가 all-ready 후 publish | ready/released predicate loop | worker/coordinator wait failure가 run-level abort publish | §12 완료 기록의 대응 근거 참조 |
| barrier release 후 | workers가 published state를 소비 | `start_ms`, initial `last_meal_ms` 확정 | `start_released == true` | 이후 run/monitor lifecycle로 이동 | §12 완료 기록의 대응 근거 참조 |

## 9. Thread 최종 상태

### Source-confirmed 최종 상태

- time acquisition은 monotonic millisecond abstraction을 사용하며 timing state는 `int64_t`입니다.
- clock failure는 simulation을 계속할 수 없는 process-fatal 상태입니다.
- workers는 readiness barrier에서 all-ready가 확인된 뒤 하나의 `start_ms`와 initial `last_meal_ms`를 공유합니다.
- partial creation과 wait failure는 terminal/release predicate와 broadcast를 통해 peers를 해제하고 error를 전파합니다.
- 이 설계는 scheduler fairness, 기아 상태 freedom, strict wakeup 또는 death-detection latency를 보장하지 않습니다.

### 학습자가 작성할 최종 설명

- clock domain: §12 완료 기록의 대응 근거에 정리했습니다.
- numeric representation: §12 완료 기록의 대응 근거에 정리했습니다.
- interruptible wait: §12 완료 기록의 대응 근거에 정리했습니다.
- readiness predicate: §12 완료 기록의 대응 근거에 정리했습니다.
- common epoch linearization: §12 완료 기록의 대응 근거에 정리했습니다.
- partial-start abort: §12 완료 기록의 대응 근거에 정리했습니다.
- wait-failure propagation: §12 완료 기록의 대응 근거에 정리했습니다.
- guarantees: §12 완료 기록의 대응 근거에 정리했습니다.
- non-guarantees: §12 완료 기록의 대응 근거에 정리했습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
worker thread objects 생성
    ↓
각 worker가 state_mutex 아래 ready_count 증가
    ↓
start_released predicate loop에서 대기
    ↓
coordinator가 all-ready 확인
    ↓
CLOCK_MONOTONIC timestamp 한 번 sampling
    ↓
table.start_ms + 모든 philo.last_meal_ms publish
    ↓
start_released = true + broadcast
    ↓
workers가 fork activity 시작

failure:
partial create 또는 cond wait failure
    ↓
run_error + ended + start_released publish
    ↓
broadcast
    ↓
workers return
    ↓
join 및 run-level error propagation
```

- 실제 worker barrier symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- 실제 coordinator barrier symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- `state_mutex` hold 범위: §12 완료 기록의 대응 근거에 정리했습니다.
- condition wait predicate: §12 완료 기록의 대응 근거에 정리했습니다.
- timestamp assignment loop: §12 완료 기록의 대응 근거에 정리했습니다.
- normal broadcast: §12 완료 기록의 대응 근거에 정리했습니다.
- abort broadcast: §12 완료 기록의 대응 근거에 정리했습니다.
- join으로 이어지는 path: §12 완료 기록의 대응 근거에 정리했습니다.
- monitor가 common epoch를 소비하는 지점: §12 완료 기록의 대응 근거에 정리했습니다.

## 11. 학습 완료 자가 점검

- [x] `509453b01515`의 wall-clock helper와 polling sleep을 당시 코드로 확인했습니다.
- [x] `a21e4cc75272`의 500/100 microsecond branch 경계를 정확히 기록했습니다.
- [x] `5b32d5bdb955`의 `CLOCK_MONOTONIC`, `int64_t`, fatal failure를 모든 주요 call site와 연결했습니다.
- [x] `f01d62cde8ce`의 known `timespec` 계산과 child exit assertion을 설명했습니다.
- [x] `e7e62cbe185f`의 barrier state machine과 common epoch linearization을 실제 code order로 제시했습니다.
- [x] `bfbfa0431732`의 injected delay가 death budget보다 긴 이유를 설명했습니다.
- [x] `f57f6ec0be87`의 wait failure가 peer release와 final error로 전파되는 경로를 설명했습니다.
- [x] condition variable을 stored event처럼 설명하지 않았습니다.
- [x] common start와 simultaneous scheduling을 혼동하지 않았습니다.
- [x] monotonic timing과 real-time guarantee를 혼동하지 않았습니다.

## 12. 저장소 기반 완료 기록

### 12.1 검토 범위와 실행 증거

- 이 Thread의 7개 SHA는 모두 `c/philo` HEAD의 조상으로 확인했습니다.
- 각 설명은 해당 SHA의 `src/time.c`, `src/run.c`, `src/routine.c`, header와 test diff를 기준으로 작성했습니다.
- 저장소 checkout을 만들 수 없어 `make test`와 개별 test binary는 실행하지 않았습니다. 아래 test 결과는 구현된 wrapper, child process, timeout, assertion을 정적 검토한 결과입니다.

### 12.2 `509453b01515` — 공통 millisecond helper와 interruptible wait

`src/time.c`에 `philo_now_ms`와 `philo_sleep_ms`가 도입됩니다.

#### time acquisition

`philo_now_ms`는 `gettimeofday`의 seconds와 microseconds를 다음과 같이 millisecond로 바꿉니다.

```text
milliseconds = tv_sec × 1000 + tv_usec ÷ 1000
```

반환형과 timing state는 이 시점에 host `long`을 사용합니다. `gettimeofday`의 반환값은 검사하지 않습니다. 따라서 system clock이 뒤로 조정되면 기아 상태 elapsed가 작아질 수 있고, 앞으로 조정되면 deadline이나 death condition이 갑자기 충족될 수 있습니다. call failure 때 timestamp가 유효하다는 근거도 없습니다.

#### deadline wait

`philo_sleep_ms`는 시작 시 `deadline = philo_now_ms() + duration_ms`를 계산합니다. loop마다 다음 순서를 반복합니다.

1. 현재 시각이 deadline에 도달했는지 확인합니다.
2. `state_mutex` 아래 `ended`를 읽습니다.
3. terminal이면 빠져나옵니다.
4. 그렇지 않으면 `usleep(500)`으로 짧게 양보합니다.

단일 `usleep(duration)`과 달리 terminal publication을 주기적으로 관찰하므로 종료 후 configured duration 전체를 계속 block하지 않습니다. 반대로 polling, scheduler wakeup, wall clock을 사용하므로 strict timing primitive는 아닙니다.

### 12.3 `a21e4cc75272` — near-deadline polling refinement

기존 500-microsecond polling 구조는 유지되고 remaining time에 따라 quantum만 달라집니다.

| remaining milliseconds | 호출 | 의미 |
| --- | --- | --- |
| `remaining > 1` | `usleep(500)` | deadline에서 충분히 멀 때 호출 횟수를 줄입니다. |
| `remaining <= 1` | `usleep(100)` | 마지막 millisecond 구간에서 프로그램이 추가하는 overshoot를 줄입니다. |

terminal check와 `state_mutex` 경계, deadline comparison, wall-clock source는 변하지 않습니다. 따라서 이 commit은 local precision refinement이며 monotonicity, real-time wakeup, scheduler latency를 해결하지 않습니다.

### 12.4 `5b32d5bdb955` — monotonic domain과 fixed-width timing

#### clock source

`src/time.c::philo_now_ms`는 `clock_gettime(CLOCK_MONOTONIC, &now)`를 사용합니다. 변환식은 다음과 같습니다.

```c
((int64_t)now.tv_sec * 1000) + (now.tv_nsec / 1000000)
```

calendar correction과 elapsed-time decision을 분리합니다. simulation start, philosopher `last_meal_ms`, sleep deadline, monitor 기아 상태 comparison, log offset이 같은 monotonic millisecond domain을 사용합니다.

#### type migration

- `t_config.time_to_die`, `time_to_eat`, `time_to_sleep`: `int64_t`
- `t_table.start_ms`: `int64_t`
- `t_philo.last_meal_ms`: `int64_t`
- `philo_now_ms`, `philo_sleep_ms`의 timing argument와 intermediate: `int64_t`
- parser intermediate도 fixed-width range에서 계산합니다.

public CLI의 intended bound는 계속 별도로 검증됩니다. fixed-width migration은 platform `long` 폭에 따른 표현 차이를 제거하기 위한 것이지 입력 범위를 무조건 64-bit 전체로 확대한다는 뜻은 아닙니다.

#### clock failure

`clock_gettime`이 실패하면 다음 fixed diagnostic을 fd 2에 `write`하고 `_exit(PHILO_ERR)`합니다.

```text
Error: monotonic clock unavailable
```

이 time source는 worker, monitor, logger의 공통 truth이므로 fabricated time을 반환해 계속 실행하지 않습니다. fatal path는 cross-thread cleanup이나 buffered I/O에 의존하지 않습니다.

#### 아직 남은 문제

monotonic clock은 올바른 elapsed-time ordering을 제공하지만 start timestamp의 sampling 시점 자체를 고치지 않습니다. timestamp를 worker readiness 전에 잡으면 늦게 생성되거나 scheduled된 worker가 실행 전 시간을 기아 상태 budget으로 잃는 문제는 그대로입니다.

### 12.5 `f01d62cde8ce` — clock boundary regression

`tests/monotonic_clock.c`는 production time object의 `clock_gettime`을 replacement로 연결합니다.

#### success mode

- wrapper가 전달받은 clock id를 기록합니다.
- known `timespec`으로 `tv_sec = 12`, `tv_nsec = 345000000`을 반환합니다.
- expected result는 `12000 + 345 = 12345` milliseconds입니다.
- assertion은 clock id가 `CLOCK_MONOTONIC`이고 conversion result가 `12345`인지 확인합니다.

#### failure mode

wrapper가 failure를 반환하도록 설정한 뒤 자식 프로세스에서 `philo_now_ms`를 호출합니다. production function이 `_exit`하므로 같은 process에서 호출하면 test runner까지 종료됩니다. parent는 wait status에서 child가 `PHILO_ERR`로 끝났는지 검사합니다.

이 테스트는 requested clock id, conversion, fatal failure contract를 결정적으로 고정합니다. 실제 kernel clock의 정확도, sleep precision, multi-thread start semantics는 검사하지 않습니다.

### 12.6 `e7e62cbe185f` — readiness barrier와 common epoch

#### 도입된 state

- `pthread_cond_t start_cond`
- `start_cond_ready`
- `ready_count`
- `start_released`
- `run_error`

condition variable도 table resource ledger에 편입되어 성공적으로 초기화된 경우에만 destroy 대상이 됩니다.

#### worker path: `src/routine.c::wait_for_start`

worker는 fork를 잡기 전에 다음 sequence를 수행합니다.

```text
state_mutex lock
→ ready_count++
→ start_cond broadcast
→ while (!start_released)
       pthread_cond_wait(start_cond, state_mutex)
→ ended 복사
→ state_mutex unlock
→ ended면 routine 종료, 아니면 fork activity 시작
```

`pthread_cond_wait`는 spurious wakeup이 가능하고 notification은 저장되는 event가 아닙니다. 그래서 wakeup 횟수가 아니라 `start_released` predicate를 loop에서 다시 검사합니다.

wait failure가 발생하면 worker는 같은 state critical section에서 `run_error = 1`, `ended = 1`, `start_released = 1`을 publish하고 broadcast합니다. peer가 impossible predicate를 계속 기다리는 상황을 없앱니다.

#### coordinator path: `src/run.c::release_start`

정상 release에서는 coordinator가 `state_mutex` 아래 다음을 수행합니다.

1. `ready_count == config.number`가 될 때까지 predicate loop에서 기다립니다.
2. worker가 보고한 `run_error`가 있는지 확인합니다.
3. `philo_now_ms()`를 한 번 호출해 `start_ms`를 얻습니다.
4. `table->start_ms`와 모든 `table->philos[i].last_meal_ms`에 같은 값을 씁니다.
5. `start_released = 1`을 publish합니다.
6. `pthread_cond_broadcast` 후 lock을 놓습니다.

common epoch의 linearization point는 one timestamp assignment와 release predicate publication이 같은 `state_mutex` hold 안에 있는 구간입니다. worker는 release predicate를 관찰한 뒤에만 lock을 벗어나므로 그 전에 fork/log activity를 시작할 수 없습니다.

#### partial creation abort

`pthread_create`가 중간에 실패하면 coordinator는 `release_start(table, 1)`을 호출합니다. 이 branch는 full readiness를 기다리지 않고 shared timestamp를 배치한 뒤 `ended = 1`, `start_released = 1`, broadcast를 수행합니다. 이미 생성돼 barrier에 도달했거나 곧 도달할 worker가 빠져나와 join될 수 있습니다. 아직 생성되지 않은 philosopher의 timestamp를 설정하는 것은 harmless initialization이며 worker existence를 의미하지 않습니다.

#### coordinator wait failure

coordinator-side `pthread_cond_wait` failure도 `run_error`와 abort release로 전환됩니다. 이후 started threads를 join하고 `PHILO_ERR`를 caller에 반환합니다.

### 12.7 `bfbfa0431732` — delayed-worker deterministic skew

`tests/start_barrier.c`는 `pthread_create`를 감싸서 real thread object를 만들되 test gate 뒤에서 routine 실행을 제어합니다.

- 모든 5개 worker를 gate에 모읍니다.
- 다섯 번째 worker에 150 ms 추가 delay를 줍니다.
- configured `time_to_die`는 80 ms입니다.
- 각 worker의 목표는 one-meal completion입니다.

barrier 전 timestamp를 사용한다면 delayed worker는 fork activity를 시작하기 전에 80 ms budget을 넘겨 false death 후보가 됩니다. production barrier가 all-ready 후 start와 initial `last_meal_ms`를 publish하면 150 ms pre-ready delay는 기아 상태 elapsed에 포함되지 않습니다.

핵심 assertion은 다음입니다.

- run이 success로 끝납니다.
- `ready_count == 5`입니다.
- `full_count == 5`입니다.
- death result가 없습니다.

이 테스트는 설정한 skew에서 common epoch 목적을 직접 검증합니다. barrier release 이후 모든 worker가 동시에 CPU를 얻거나 모든 scheduler delay에 안전하다는 것은 증명하지 않습니다.

### 12.8 `f57f6ec0be87` — worker wait failure propagation

`tests/worker_wait_failure.c`는 routine object의 첫 번째 `pthread_cond_wait`만 `EINVAL`로 실패시킵니다. one-shot injection flag는 test mutex로 보호되고, 이후 wait는 real pthread function에 위임됩니다. 따라서 harness 자체가 unrelated data race를 추가하지 않도록 합니다.

production worker의 failed wait path는 `run_error`, `ended`, `start_released`를 publish하고 broadcast합니다. coordinator는 `run_error`를 보고 abort 결과를 만들며 started workers를 join합니다.

테스트가 요구하는 결과:

- injection이 실제로 사용됩니다.
- `table.run_error`가 true입니다.
- `philo_run`은 `PHILO_ERR`입니다.
- external timeout 안에 종료합니다.

external timeout은 peer가 release되지 않아 barrier에서 영구 block하는 회귀를 bounded failure로 바꿉니다. 이 test는 첫 worker-side wait failure 한 위치를 겨냥하며 condition-variable API의 모든 failure 위치를 포괄하지 않습니다.

### 12.9 항상 유지해야 하는 조건 evolution 완성

| 항상 유지해야 하는 조건 | 최초 상태 | 부족함 | 수정 | evidence |
| --- | --- | --- | --- | --- |
| time access와 deadline wait의 공통화 | `509453b01515` | wall clock, unchecked failure | `5b32d5bdb955`에서 source/contract 교정 | `f01d62cde8ce` |
| near-deadline program overshoot 완화 | 500 µs fixed polling | 짧은 duration 대비 quantum이 큼 | `a21e4cc75272`의 500/100 µs branch | 별도 deterministic sleep test 없음 |
| elapsed time은 monotonic domain | `gettimeofday` | calendar jump가 기아 상태/deadline 왜곡 | `CLOCK_MONOTONIC` | clock-id assertion |
| timing state는 fixed-width | host `long` | platform 폭 의존 | `int64_t` migration | known conversion과 call-site inspection |
| clock failure를 time으로 위장하지 않음 | return 미검사 | invalid timestamp로 진행 가능 | diagnostic + `_exit` | child-process failure test |
| 모든 worker가 같은 start/initial meal reference 사용 | creation 전 timestamp | late worker가 pre-ready budget 상실 | all-ready barrier + one publish | 150 ms/80 ms skew test |
| barrier failure는 peer release와 run error로 전파 | barrier 도입 시 정의 | worker-local return이면 peers hang | error/ended/released + broadcast | wait failure + timeout |
| condition wait는 predicate loop 사용 | `e7e62cbe185f` | signal count를 event로 취급하면 lost/spurious wakeup 취약 | `while (!start_released)` | source path inspection |

### 12.10 최종 temporal 실행 순서

```text
pthread_create로 worker objects 생성
    ↓
worker: state_mutex 아래 ready_count 증가
    ↓
worker: while (!start_released) cond_wait
    ↓
coordinator: ready_count == number 확인
    ↓
CLOCK_MONOTONIC timestamp 한 번 sampling
    ↓
table.start_ms와 모든 philo.last_meal_ms에 같은 값 publish
    ↓
start_released = 1 + broadcast
    ↓
worker가 lock을 벗어나 fork activity 시작

partial create / worker wait / coordinator wait failure
    ↓
run_error 또는 should_end
    ↓
ended = 1, start_released = 1
    ↓
broadcast
    ↓
started workers barrier 탈출
    ↓
join
    ↓
PHILO_ERR propagation
```

### 12.11 최종 보장과 비보장

보장하는 범위:

- 기아 상태, deadline, log offset은 monotonic millisecond domain을 사용합니다.
- timing state는 `int64_t`로 표현됩니다.
- clock acquisition failure는 unchecked time으로 계속 진행하지 않습니다.
- successfully created workers는 readiness barrier 이후 같은 start와 initial `last_meal_ms`를 받습니다.
- partial creation과 wait failure는 release predicate와 broadcast로 peer를 해제합니다.

보장하지 않는 범위:

- barrier release 뒤 simultaneous CPU scheduling은 보장하지 않습니다.
- polling sleep의 정확한 wakeup 시각, strict death-detection latency, real-time 동작은 보장하지 않습니다.
- monotonic clock이나 barrier는 fairness와 기아 상태 freedom을 제공하지 않습니다.
- source inspection만 수행했으므로 이 환경에서 tests가 실제 통과했다는 실행 증거는 없습니다.
===== END FILE: 02-wall-clock-to-shared-monotonic-start.md =====

===== BEGIN FILE: 03-core-routine-to-committed-meal-progress.md =====
# Thread: Core routine to committed and range-safe meal progress

이 문서는 source에 정의된 세 번째 Development Thread를 그대로 따릅니다. commit 순서, SHA, importance, tags는 변경하지 않습니다. worker routine의 완성된 final HEAD를 첫 구현 commit에 소급하지 않고, 각 fix가 어떤 기존 가정과 transaction boundary를 수정했는지 해당 SHA에서 확인합니다.

## 1. Thread 목표

이 Thread의 목표는 최초 eat-sleep-think worker가 어떻게 fork identity edge, global completion transition, interrupted operation, internal counter range 문제를 드러내고, 최종적으로 committed and range-safe meal progress를 갖게 되는지 복원하는 것입니다.

Source-confirmed significance는 다음과 같습니다.

- 최초 routine은 parity-dependent fork order와 worker-local 상태 전이를 도입하여 domain core를 만듭니다.
- `N == 1`에서는 ring mapping 때문에 두 fork pointer가 같은 mutex가 되어 general two-lock path가 실패합니다.
- final required meal과 global `ended` publication을 같은 critical section으로 묶어 post-completion work를 막습니다.
- eating log 또는 fork 소유권이 meal completion의 증거가 아니며, full interval과 synchronized active-state check를 통과해야 counter가 commit됩니다.
- public `INT_MAX` target과 그 이후에도 증가할 수 있는 internal counter의 numeric range를 분리합니다.
- deterministic tests는 interrupted meal과 former overflow boundary에서 local/global counter 항상 유지해야 하는 조건을 직접 검증합니다.

### Source에 명시적으로 연결된 Critical 항상 유지해야 하는 조건

- fork 하나는 두 neighbor가 공유하는 하나의 mutex identity입니다.
- 두 명 이상에서는 acquisition order가 uniform all-left circular wait를 깨야 하며, 한 명에서는 같은 mutex를 재잠금하지 않아야 합니다.
- meal은 eating interval이 끝까지 완료되고 synchronized commit point에서 simulation이 active인 경우에만 count합니다.
- `full_count`는 각 philosopher가 target에 처음 도달할 때만 한 번 증가합니다.
- internal meal accumulation은 public `INT_MAX` target 이후에도 defined 상태로 유지되고 second `full_count` contribution을 만들지 않아야 합니다.
- meal state와 terminal state는 정의된 `state_mutex` boundary에서 관찰·변경됩니다.

### Source에 명시적으로 연결된 Major Engineering Difficulties

- circular wait를 끊으면서 fairness 또는 기아 상태 freedom까지 보장한다고 과장하지 않는 문제
- fork 획득, `is eating` log, eating wait, counter mutation 사이에서 operation commit 지점을 정의하는 문제
- valid public bound와 장기간 증가하는 internal state의 numeric range를 분리하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- odd/even fork order가 ring lock graph에서 어떤 circular-wait edge를 제거하는가?
- parity rule과 initial stagger가 각각 correctness와 contention에 어떤 다른 역할을 하는가?
- `N == 1`에서 왜 left/right fork pointer가 aliasing되며 general routine은 어디서 self-deadlock하는가?
- final required meal을 기록한 worker가 global completion을 언제 publish해야 하는가?
- fork를 두 개 보유했다는 사실이 terminal 이후 새 meal을 허용하지 않는 이유는 무엇입니까?
- `is eating` log와 completed meal counter는 왜 동일한 사건이 아닌가?
- eating wait가 interruption과 deadline completion을 어떻게 구분해야 하는가?
- sleep 완료 직후 counter lock을 잡기 전 terminal이 바뀌는 race는 어디서 다시 확인해야 하는가?
- 모든 abort path에서 fork 소유권은 어떻게 해제됩니까?
- `must_eat <= INT_MAX`인데 internal `meals`가 `INT_MAX + 1`이 될 수 있는 valid scenario는 무엇입니까?
- 회귀 테스트가 counter mutation을 scheduler luck 없이 어떻게 직접 자극하는가?

## 3. 완료 기준

- [x] 최초 routine의 fork lock graph와 worker 상태 전이를 실제 code order로 설명할 수 있습니다.
- [x] parity order가 circular wait를 깨는 범위와 fairness non-guarantee를 구분할 수 있습니다.
- [x] `N == 1` pointer 같은 메모리를 가리키는 경우와 dedicated path를 before/after로 제시할 수 있습니다.
- [x] final meal commit, `full_count`, global `ended` publication의 critical section을 설명할 수 있습니다.
- [x] fork acquisition 후 terminal recheck와 post-meal terminal recheck의 역할을 구분할 수 있습니다.
- [x] interrupted eating이 counter를 변경하지 않는 operation transaction을 그릴 수 있습니다.
- [x] wait success 뒤 mutation 전 terminal race를 locked recheck로 설명할 수 있습니다.
- [x] 모든 logical abort에서 두 fork가 release되는지 path별로 확인했습니다.
- [x] public target과 internal accumulated counter의 range를 구분할 수 있습니다.
- [x] `INT_MAX + 1` test가 numeric width와 duplicate contribution을 동시에 확인하는 이유를 설명할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `b68f40819af4` | `feat(routine): 철학자의 식사·수면·사고 흐름 구현` | S | `CORE, CONCURRENCY, FORK_ORDER` | Introduces the eat-sleep-think worker and parity-dependent fork order. |
| 2 | `c8531c91f0fb` | `fix(single): 철학자가 한 명일 때 포크 재잠금 방지` | A | `FORK_ORDER, EDGE, RISK` | Handles the ring aliasing edge where one philosopher's two fork pointers are the same mutex. |
| 3 | `fe0a2d15b29b` | `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단` | A | `MEAL_ACCOUNTING, TERMINAL_STATE, RISK` | Commits global completion in the final meal critical section and prevents post-completion loop states. |
| 4 | `53e591effb4a` | `fix(routine): 중단된 식사를 완료 횟수에서 제외` | A | `MEAL_ACCOUNTING, TERMINAL_STATE, RISK` | Separates an eating attempt from a committed meal and rejects progress after interruption or terminal state. |
| 5 | `73b5551a76f4` | `test(routine): 중단된 식사의 카운터 불변식 검증` | B | `TEST, MEAL_ACCOUNTING` | Injects an interrupted eating wait and verifies that neither local nor global counters advance. |
| 6 | `4c224ae86f2b` | `fix(state): 식사 완료 횟수의 정수 범위 확장` | A | `MEAL_ACCOUNTING, EDGE, RISK` | Widens accumulated meals so a valid `INT_MAX` target can be exceeded without signed overflow. |
| 7 | `054ef46f80c7` | `test(routine): 최대 목표 이후 식사 카운터 검증` | B | `TEST, MEAL_ACCOUNTING, EDGE` | Verifies `INT_MAX + 1` and confirms the philosopher does not contribute to `full_count` twice. |

## 5. Commit별 학습 기록

### 5.1 `b68f40819af4` — `feat(routine): 철학자의 식사·수면·사고 흐름 구현`

- Importance: **S**
- Tags: `CORE, CONCURRENCY, FORK_ORDER`
- Source-defined role: Introduces the eat-sleep-think worker and parity-dependent fork order.
- 코드 기준: 반드시 `b68f40819af4` 시점
- 직접 parent 비교: `git diff b68f40819af4^ b68f40819af4 --`
- Thread 직전 관련 SHA: Thread 내 첫 commit

#### Source-confirmed 맥락

이 S-level commit은 Dining Philosophers domain의 핵심 worker mechanism을 처음 완성합니다. philosopher는 두 fork mutex를 획득하고, `state_mutex` 아래에서 meal start를 기록하며, eating log와 wait를 수행한 뒤 meal completion을 기록하고 fork를 release합니다. 이후 sleeping과 thinking을 반복하며 terminal flag를 관찰합니다.

fork acquisition order는 philosopher parity에 따라 달라집니다. even identifier는 right→left, odd identifier는 left→right 순서로 lock하여 모든 worker가 같은 left-first circular wait를 만드는 구조를 끊습니다. even worker의 1-millisecond initial delay는 contention 완화 장치일 뿐 fairness guarantee가 아닙니다.

이 최초 구현은 left/right fork가 서로 다르다고 가정하고, interruptible wait가 completion status를 반환하지 않으므로 interrupted eating도 완료된 meal로 count할 수 있습니다. 이 두 한계는 후속 fix의 출발점입니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 문제 | 각 worker가 shared fork 두 개를 사용해 eat-sleep-think state를 진행하고 monitor가 신뢰할 meal state를 publish해야 합니다. | §12 완료 기록의 대응 근거 참조 |
| 교착 상태 위험 | 모든 philosopher가 left fork를 먼저 잡으면 ring 전체가 one-fork hold 상태로 circular wait할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | identifier parity에 따라 fork acquisition order를 반대로 배치합니다. | §12 완료 기록의 대응 근거 참조 |
| meal state 결정 | `last_meal_ms`는 eating 시작 시 갱신하고, meal counter와 `full_count`는 shared state lock 아래 갱신합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 edge | `N == 1`에서 두 fork pointer가 같은 mutex를 가리킵니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 transaction 결함 | interruptible wait가 끝까지 완료되었는지 구분하지 못해 aborted eating도 count될 수 있습니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `git show --name-status b68f40819af4`로 worker routine이 추가된 실제 파일과 public entry symbol을 확인합니다.
- [x] routine의 outer loop와 terminal check 위치를 순서대로 기록합니다.
- [x] odd/even identifier별 첫 번째·두 번째 fork pointer와 실제 `pthread_mutex_lock` 순서를 표로 만듭니다.
- [x] fork acquisition 후 각 fork status log가 어느 lock 성공 뒤에 호출되는지 확인합니다.
- [x] `last_meal_ms`가 eating log, eating wait, counter increment 중 어느 시점에 갱신되는지 확인합니다.
- [x] meal start와 meal completion state mutation이 `state_mutex` 아래에서 이뤄지는지 확인합니다.
- [x] `full_count`가 philosopher의 `meals == must_eat` 조건에서만 증가하는지 확인합니다.
- [x] 두 fork의 release 순서와 모든 early return/failure branch에서의 release 여부를 확인합니다.
- [x] even worker initial delay가 어느 identifier 조건에서, barrier 또는 fork lock 전후 중 어디에 위치하는지 확인합니다.
- [x] `left_fork == right_fork`인 `N == 1`에서 실행되는 lock sequence를 실제 pointer mapping과 결합해 재구성합니다.
- [x] eating wait의 반환 type과 호출자가 그 결과를 사용하는지 확인하여 interrupted-meal 결함을 증명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Worker 상태 전이 기록

| 단계 | 보유 fork | `state_mutex` mutation | log | wait | 실패/terminal 시 cleanup |
| --- | --- | --- | --- | --- | --- |
| routine 진입 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | optional initial delay | §12 완료 기록의 대응 근거 참조 |
| 첫 fork 획득 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| 두 번째 fork 획득 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| meal start commit | §12 완료 기록의 대응 근거 참조 | `last_meal_ms` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| eating interval | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | `is eating` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| meal completion | §12 완료 기록의 대응 근거 참조 | `meals`, `full_count` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| sleeping | none | §12 완료 기록의 대응 근거 참조 | `is sleeping` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| thinking | none | §12 완료 기록의 대응 근거 참조 | `is thinking` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### Fork order 검증

| Philosopher parity | First fork | Second fork | 제거하는 uniform wait edge | fairness 보장 여부 |
| --- | --- | --- | --- | --- |
| odd | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | 보장하지 않음 |
| even | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | 보장하지 않음 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- multi-worker ring에서 모든 worker가 동일한 left-first order를 갖는 circular-wait 구조를 깨뜨립니다.
- worker-local eat-sleep-think cycle과 monitor가 읽을 meal timestamp/counter의 producer가 정의됩니다.
- `full_count` contribution을 threshold equality에 연결하여 한 philosopher가 최초 target 도달 시 한 번만 기여하려는 규칙을 둡니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- `N == 1`에서 동일 non-recursive mutex를 두 번 lock하지 않는 것은 아직 보장하지 않습니다.
- interrupted eating interval이 counter를 증가시키지 않는 것은 아직 보장하지 않습니다.
- scheduler fairness 또는 기아 상태 freedom을 보장하지 않습니다.

#### 학습자 결론
- [x] parity order가 classic all-left circular wait를 끊는 lock graph를 실제 fork indices로 그립니다.
- [x] 1-millisecond delay가 correctness 항상 유지해야 하는 조건이 아닌 이유를 설명합니다.
- [x] meal start와 meal completion이 다른 state transition인 이유를 후속 commits와 연결합니다.
- [x] 이 commit의 worker responsibility와 monitor가 나중에 맡는 global policy를 구분합니다.

### 5.2 `c8531c91f0fb` — `fix(single): 철학자가 한 명일 때 포크 재잠금 방지`

- Importance: **A**
- Tags: `FORK_ORDER, EDGE, RISK`
- Source-defined role: Handles the ring 같은 메모리를 가리키는 경우 edge where one philosopher's two fork pointers are the same mutex.
- 코드 기준: 반드시 `c8531c91f0fb` 시점
- 직접 parent 비교: `git diff c8531c91f0fb^ c8531c91f0fb --`
- Thread 직전 관련 SHA: `b68f40819af4`

#### Fix chain

ring mapping에서 `N == 1`이면 `left_fork`와 `right_fork`가 같은 mutex 주소입니다. 기존 worker는 non-recursive mutex를 한 번 lock한 뒤 같은 thread에서 다시 lock하여 영구 block될 수 있습니다. 이 fix는 single-philosopher 전용 path에서 하나의 fork만 한 번 획득하고 가능한 fork event를 남긴 뒤 `time_to_die`보다 조금 더 기다리고 release합니다. death 판단과 log는 multi-worker와 동일하게 monitor가 담당합니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | 두 fork pointer는 서로 다른 mutex이므로 normal two-lock path를 사용할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure | `N == 1`에서 ring 같은 메모리를 가리키는 경우로 두 pointer가 같아 self-교착 상태가 결정적으로 발생합니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | topology가 만드는 pointer identity edge를 routine이 구분하지 않았습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 결정 | single worker는 fork 하나만 lock·log·wait·unlock하고 meal을 시작하지 않습니다. | §12 완료 기록의 대응 근거 참조 |
| 유지되는 책임 | worker는 자원 제약을 모델링하고 monitor가 기아 상태 death를 확정합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `b68f40819af4` 대비 single-philosopher branch의 진입 조건과 호출 위치를 확인합니다.
- [x] `left_fork`와 `right_fork`의 실제 pointer equality가 `N == 1`에서 성립하는지 initializer mapping으로 검산합니다.
- [x] single path가 mutex를 정확히 한 번 lock하고 한 번 unlock하는지 확인합니다.
- [x] fork acquisition log가 몇 번 발생하는지 확인합니다.
- [x] wait duration이 `time_to_die`보다 어떻게 길게 계산되는지 실제 식과 type으로 확인합니다.
- [x] terminal-aware wait가 조기 종료될 때 fork가 release되는지 확인합니다.
- [x] single path가 meal-start state, `is eating`, meal counter를 변경하지 않는지 확인합니다.
- [x] worker return 후 monitor가 death를 발견할 수 있는 shared state와 execution order를 확인합니다.
- [x] multi-worker acquisition path가 변경되지 않았는지 diff로 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Edge-case execution trace

```text
N == 1
→ left_fork address == right_fork address
→ dedicated branch
→ lock once
→ one fork event
→ no meal-start transition
→ wait beyond death threshold or terminal interruption
→ unlock once
→ worker return
→ monitor remains death authority
```

- 실제 branch symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- pointer equality 근거: §12 완료 기록의 대응 근거에 정리했습니다.
- wait expression: §12 완료 기록의 대응 근거에 정리했습니다.
- unlock 보장: §12 완료 기록의 대응 근거에 정리했습니다.
- monitor observation path: §12 완료 기록의 대응 근거에 정리했습니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- valid input `N == 1`에서 worker가 같은 non-recursive mutex를 재잠금하지 않습니다.
- 한 philosopher가 하나의 실제 fork만 획득할 수 있고 meal을 시작할 수 없다는 resource constraint를 보존합니다.
- death authority를 worker로 옮기지 않고 monitor responsibility를 유지합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- single philosopher가 meal을 완료하거나 meal target을 만족한다고 보장하지 않습니다.
- multi-worker fairness나 기아 상태 freedom을 바꾸지 않습니다.
- 이 Thread 내 dedicated deterministic test commit은 source에 포함되어 있지 않습니다.

#### 학습자 결론
- [x] general ring topology가 valid edge input에서 같은 메모리를 가리키는 경우를 만드는 과정을 주소로 설명합니다.
- [x] 가짜 두 번째 fork를 만들지 않고 실제 resource constraint를 모델링한 이유를 설명합니다.
- [x] single worker와 monitor 사이의 responsibility split을 execution trace로 제시합니다.

### 5.3 `fe0a2d15b29b` — `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단`

- Importance: **A**
- Tags: `MEAL_ACCOUNTING, TERMINAL_STATE, RISK`
- Source-defined role: Commits global completion in the final meal critical section and prevents post-completion loop states.
- 코드 기준: 반드시 `fe0a2d15b29b` 시점
- 직접 parent 비교: `git diff fe0a2d15b29b^ fe0a2d15b29b --`
- Thread 직전 관련 SHA: `c8531c91f0fb`

#### Fix chain

이 A-level fix는 final required meal을 기록하는 worker가 `state_mutex` 안에서 `meals`, `full_count`, global `ended`를 한 transaction으로 갱신하게 합니다. `eat_once`는 status를 반환하고, 두 fork를 모두 획득한 뒤 다른 worker가 이미 completion을 commit했다면 meal을 시작하지 않고 두 fork를 release합니다. completed meal 뒤에도 terminal state를 다시 확인해 sleeping 또는 thinking log로 진행하지 않습니다.

다만 eating delay가 terminal로 중단되었는지 구분하지 못하는 문제는 남아 있어, interrupted meal도 counter에 반영될 수 있습니다. 이 한계가 `53e591effb4a`의 직접 원인입니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 상태 | monitor polling이 global completion을 나중에 발견하며, worker loop가 target 도달 뒤에도 다음 state로 진행할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 위험 | last required meal과 `ended` publication 사이의 delay 동안 다른 worker가 새 meal이나 post-completion log를 시작할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 decision | final meal commit을 기록하는 worker가 lock 안에서 global completion까지 publish합니다. | §12 완료 기록의 대응 근거 참조 |
| fork-boundary decision | fork 소유권만으로 terminal 이후 새 meal을 시작할 권한이 생기지 않으므로 acquisition 후 state를 재확인합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 root cause | eating wait가 deadline completion과 terminal interruption을 구분하지 않습니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `c8531c91f0fb` 대비 `record_meal_done`의 mutation 순서와 return/side effect 변화를 확인합니다.
- [x] per-philosopher `meals` 증가, equality check, `full_count` 증가, all-full check, `ended` publication이 하나의 `state_mutex` critical section에 있는지 확인합니다.
- [x] `full_count`가 threshold equality에서만 증가하고 target 초과 시 재기여하지 않는지 확인합니다.
- [x] `eat_once`의 return status가 routine loop에 어떻게 사용되는지 확인합니다.
- [x] 두 fork 획득 직후 terminal recheck branch가 두 mutex를 모두 release하는지 확인합니다.
- [x] completed meal 직후 routine이 terminal을 확인하고 sleeping/thinking으로 진행하지 않는지 확인합니다.
- [x] monitor의 meal-completion 역할이 이 SHA에서 어떻게 줄거나 유지되는지 실제 코드로 확인합니다.
- [x] `philo_sleep_ms` return type과 eating 호출 지점을 확인하여 interrupted wait 후에도 `record_meal_done`이 호출되는 경로를 찾습니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Completion transaction

| mutation 또는 check | lock 경계 | 조건 | 다음 상태 | 실제 코드 |
| --- | --- | --- | --- | --- |
| `meals` increment | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| threshold equality | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `full_count` increment | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| all philosophers full | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `ended` publication | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | routine stop | §12 완료 기록의 대응 근거 참조 |
| acquisition 후 terminal recheck | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | release without meal | §12 완료 기록의 대응 근거 참조 |
| meal 후 terminal recheck | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | suppress sleep/think | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- global meal completion이 final required meal의 locked completion path에서 즉시 publish됩니다.
- terminal state가 fork 대기 중 확정된 worker는 acquisition 후 새 meal을 시작하지 않습니다.
- global completion 뒤 sleeping/thinking state로 불필요하게 진행하는 것을 막습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- eating interval이 끝까지 완료되지 않았는데도 meal counter가 증가하지 않는 것은 아직 보장하지 않습니다.
- fairness 또는 모든 worker의 equal progress를 보장하지 않습니다.

#### 학습자 결론
- [x] fork 두 개를 가졌다는 사실과 새 meal을 commit할 권한을 구분합니다.
- [x] final meal의 local mutation이 global termination으로 연결되는 critical section을 설명합니다.
- [x] 이 fix가 닫은 polling gap과 아직 남긴 interrupted-meal gap을 구분합니다.

### 5.4 `53e591effb4a` — `fix(routine): 중단된 식사를 완료 횟수에서 제외`

- Importance: **A**
- Tags: `MEAL_ACCOUNTING, TERMINAL_STATE, RISK`
- Source-defined role: Separates an eating attempt from a committed meal and rejects progress after interruption or terminal state.
- 코드 기준: 반드시 `53e591effb4a` 시점
- 직접 parent 비교: `git diff 53e591effb4a^ 53e591effb4a --`
- Thread 직전 관련 SHA: `fe0a2d15b29b`

#### Fix chain

이 A-level fix는 eating attempt와 committed meal을 분리합니다. `philo_sleep_ms`는 deadline에 도달했는지 terminal state 때문에 중단되었는지를 status로 반환합니다. eating wait가 중단되면 worker는 두 fork를 release하고 local/global meal counter를 변경하지 않습니다.

wait가 deadline에 도달한 직후 다른 thread가 terminal state를 commit할 수 있으므로 `record_meal_done`도 `state_mutex` 안에서 mutation 직전 `ended`를 재확인합니다. 즉, full eating interval 완료와 synchronized active-state check를 모두 통과해야 meal이 commit됩니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | `is eating` log와 wait 호출 이후에는 meal을 완료한 것으로 count할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | terminal interrupt로 eating interval이 짧아져도 `meals`와 `full_count`가 증가할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | wait API가 deadline completion과 terminal interruption을 구분하지 않고, counter mutation 직전 state revalidation도 부족합니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 decision | sleep status를 eating operation contract로 사용하고 interruption은 abort로 처리합니다. | §12 완료 기록의 대응 근거 참조 |
| race 보완 | deadline 도달 후 mutation 전에 terminal이 commit되는 window를 locked `ended` recheck로 닫습니다. | §12 완료 기록의 대응 근거 참조 |
| resource 항상 유지해야 하는 조건 | 논리 operation이 commit되지 않아도 acquired fork 둘은 모든 exit에서 release합니다. | §12 완료 기록의 대응 근거 참조 |
| regression 연결 | `73b5551a76f4`가 interrupted sleep을 주입해 local/global counter가 0으로 유지되는지 검증합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `fe0a2d15b29b` 대비 `philo_sleep_ms`의 반환 type과 return condition을 확인합니다.
- [x] deadline reached와 terminal observed가 각각 어떤 status로 반환되는지 확인합니다.
- [x] `eat_once`가 eating sleep result를 검사하고 실패 시 어떤 release helper를 호출하는지 확인합니다.
- [x] interrupted branch에서 `record_meal_done`이 호출되지 않는지 확인합니다.
- [x] `record_meal_done`이 `state_mutex`를 잡은 뒤 mutation 직전 `ended`를 재확인하는 순서를 확인합니다.
- [x] sleep success 후 lock 획득 전 terminal이 바뀌는 interleaving을 코드 순서로 작성합니다.
- [x] 해당 interleaving에서 recheck가 false commit을 막는지 확인합니다.
- [x] 모든 abort/return path에서 first/second fork 소유권이 정확히 한 번 release되는지 표로 검증합니다.
- [x] sleep API 변경이 sleeping phase나 single-philosopher path의 호출 지점에 어떻게 반영되는지 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Meal transaction boundary

```text
fork ownership 획득
→ meal-start state/log
→ full eating deadline wait
    ├─ terminal interruption
    │      → release forks
    │      → no counter mutation
    └─ deadline reached
           → state_mutex 획득
           → ended 재검사
               ├─ ended
               │      → no counter mutation
               └─ active
                      → meals/full_count commit
→ release forks
```

- 각 단계의 actual symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- operation start와 commit의 구분: §12 완료 기록의 대응 근거에 정리했습니다.
- terminal interruption status: §12 완료 기록의 대응 근거에 정리했습니다.
- mutation 전 recheck: §12 완료 기록의 대응 근거에 정리했습니다.
- fork release proof: §12 완료 기록의 대응 근거에 정리했습니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- configured eating interval을 완료하고 synchronized commit point에서 simulation이 active인 meal만 count합니다.
- interrupted meal은 per-philosopher `meals`와 global `full_count`를 변경하지 않습니다.
- 모든 logical abort exit가 acquired fork를 release합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- meal-start log가 곧 meal completion을 의미하지 않습니다.
- 이 변경은 scheduler fairness나 기아 상태 freedom을 보장하지 않습니다.

#### 학습자 결론
- [x] operation start, wait completion, state commit을 세 단계로 분리해 설명합니다.
- [x] sleep success 직후 terminal publication race를 실제 lock timing으로 설명합니다.
- [x] resource cleanup 항상 유지해야 하는 조건과 logical meal accounting 항상 유지해야 하는 조건이 함께 유지되는 이유를 설명합니다.

### 5.5 `73b5551a76f4` — `test(routine): 중단된 식사의 카운터 불변식 검증`

- Importance: **B**
- Tags: `TEST, MEAL_ACCOUNTING`
- Source-defined role: Injects an interrupted eating wait and verifies that neither local nor global counters advance.
- 코드 기준: 반드시 `73b5551a76f4` 시점
- 직접 parent 비교: `git diff 73b5551a76f4^ 73b5551a76f4 --`
- Thread 직전 관련 SHA: `53e591effb4a`

#### Source-confirmed test 역할

이 deterministic routine test는 `philo_sleep_ms`를 대체하여 `ended`를 publish하고 `PHILO_ERR`를 반환합니다. worker는 start barrier를 이미 지난 상태로 직접 호출되며 local meal count와 global completion count는 0에서 시작합니다. production routine의 실제 fork locking과 state update 경로는 유지하고, injected interruption 뒤 두 counter가 모두 0인지 검사합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| 주입하는 interrupted operation | §12 완료 기록의 대응 근거 참조 |
| stub이 publish하는 state와 return status | §12 완료 기록의 대응 근거 참조 |
| direct worker invocation 조건 | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production fork/state path | §12 완료 기록의 대응 근거 참조 |
| 초기 counter 상태 | §12 완료 기록의 대응 근거 참조 |
| expected local counter | §12 완료 기록의 대응 근거 참조 |
| expected global counter | §12 완료 기록의 대응 근거 참조 |
| resource release 관찰 방법 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] routine object에서 `philo_sleep_ms`가 test stub으로 치환되는 build 방식을 확인합니다.
- [x] stub이 production `state_mutex` 경계를 사용해 `ended`를 publish하는지 실제 코드를 확인합니다.
- [x] stub return status가 `PHILO_ERR`인지 확인합니다.
- [x] directly invoked worker가 barrier를 기다리지 않도록 table/philo state를 어떻게 seed하는지 확인합니다.
- [x] fork mutex는 real production locking을 사용하는지 확인합니다.
- [x] initial `meals == 0`, `full_count == 0` 설정을 확인합니다.
- [x] worker return 후 두 counter가 0이라는 assertion을 확인합니다.
- [x] fork가 잠긴 채 남지 않았음을 test가 직접 또는 간접으로 어떻게 확인하는지 구분합니다.
- [x] shell-level output count로는 같은 semantic bug를 결정적으로 찾기 어려운 이유를 기록합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- injected eating interruption에서 production routine이 local/global meal progress를 commit하지 않음을 결정적으로 검증합니다.
- scheduler variability 없이 `eat_once` transaction boundary를 직접 통과합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- full executable의 모든 interleaving이나 monitor interaction을 검증하지 않습니다.
- 모든 sleep 호출 지점 또는 모든 terminal timing을 포괄하지 않습니다.

#### 학습자 결론
- [x] direct invocation이 regression target을 좁히는 장점과 integration coverage를 줄이는 한계를 설명합니다.
- [x] stubbed sleep 앞뒤의 production lock/state path를 구분합니다.
- [x] 두 counter assertion이 meal transaction 항상 유지해야 하는 조건을 어떻게 고정하는지 설명합니다.

### 5.6 `4c224ae86f2b` — `fix(state): 식사 완료 횟수의 정수 범위 확장`

- Importance: **A**
- Tags: `MEAL_ACCOUNTING, EDGE, RISK`
- Source-defined role: Widens accumulated meals so a valid `INT_MAX` target can be exceeded without signed overflow.
- 코드 기준: 반드시 `4c224ae86f2b` 시점
- 직접 parent 비교: `git diff 4c224ae86f2b^ 4c224ae86f2b --`
- Thread 직전 관련 SHA: `73b5551a76f4`

#### Fix chain

public `must_eat` target은 `INT_MAX`로 제한되지만, 한 philosopher가 target에 일찍 도달한 뒤 다른 philosopher의 completion을 기다리는 동안 추가 meal을 완료할 수 있습니다. per-philosopher accumulated counter도 `int`이면 valid execution에서 signed overflow가 발생할 수 있습니다. mutex protection은 data race를 막을 뿐 signed overflow undefined 동작을 안전하게 만들지 않습니다.

이 fix는 internal `meals`를 `int64_t`로 확장하면서 public target bound와 equality-based `full_count` contribution을 유지합니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | public target이 `INT_MAX`이므로 runtime accumulated counter도 `int`면 충분합니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | target 도달 후에도 다른 philosopher를 기다리며 추가 meal이 발생해 `INT_MAX + 1`이 valid path에서 필요할 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | bounded public input과 계속 증가할 수 있는 internal state를 같은 numeric range로 취급했습니다. | §12 완료 기록의 대응 근거 참조 |
| 수정 decision | internal `meals`만 `int64_t`로 넓히고 public `must_eat` contract는 유지합니다. | §12 완료 기록의 대응 근거 참조 |
| completion 항상 유지해야 하는 조건 | `full_count` contribution은 equality 시점 한 번만 발생하며 target 초과 후 재기여하지 않습니다. | §12 완료 기록의 대응 근거 참조 |
| regression 연결 | `054ef46f80c7`이 `INT_MAX`에서 한 meal을 더 완료해 `INT_MAX + 1`과 unchanged `full_count`를 검증합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `73b5551a76f4` 대비 `meals` field의 type 변경과 관련 선언·format·comparison을 모두 찾습니다.
- [x] public `must_eat` field/type 및 parser bound가 변경되지 않는지 확인합니다.
- [x] counter increment expression이 wider type에서 수행되는지 확인합니다.
- [x] `meals == must_eat` 비교에서 implicit conversion과 intended equality semantics를 확인합니다.
- [x] target 초과 후 `full_count`가 다시 증가하지 않는 control flow를 확인합니다.
- [x] counter를 읽는 monitor 또는 test code가 type 변경에 맞게 수정되는지 확인합니다.
- [x] mutex가 numeric overflow 해결책이 아닌 이유를 해당 increment path와 C signed-overflow semantics로 설명하되 source 밖의 다른 numeric guarantee를 추가하지 않습니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Public contract와 internal state 분리

| 항목 | Public bound | Runtime accumulation | 해당 SHA type | mutation/compare 위치 |
| --- | --- | --- | --- | --- |
| `must_eat` | `INT_MAX` | fixed target | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `meals` | 직접 입력 아님 | target 이후도 증가 가능 | `int64_t` | §12 완료 기록의 대응 근거 참조 |
| `full_count` contribution | philosopher당 한 번 | equality에서만 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- valid `INT_MAX` target을 넘어서 한 번 이상 계속되는 internal meal accumulation이 즉시 signed `int` overflow를 일으키지 않습니다.
- public target bound와 threshold equality-based one-time contribution을 유지합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- internal counter가 무한히 증가해도 절대 overflow하지 않는다고 보장하지 않습니다.
- public `must_eat` 범위를 `INT_MAX`보다 넓히지 않습니다.
- progress fairness를 보장하지 않습니다.

#### 학습자 결론
- [x] public input range와 internal accumulated state range가 다른 이유를 concrete execution으로 설명합니다.
- [x] mutex synchronization과 defined numeric range가 서로 다른 correctness 문제임을 설명합니다.
- [x] widening과 one-time `full_count` contribution이 함께 유지되는 코드를 제시합니다.

### 5.7 `054ef46f80c7` — `test(routine): 최대 목표 이후 식사 카운터 검증`

- Importance: **B**
- Tags: `TEST, MEAL_ACCOUNTING, EDGE`
- Source-defined role: Verifies `INT_MAX + 1` and confirms the philosopher does not contribute to `full_count` twice.
- 코드 기준: 반드시 `054ef46f80c7` 시점
- 직접 parent 비교: `git diff 054ef46f80c7^ 054ef46f80c7 --`
- Thread 직전 관련 SHA: `4c224ae86f2b`

#### Source-confirmed test 역할

이 회귀 테스트는 두-philosopher table에서 public target을 `INT_MAX`로 두고 한 philosopher의 `meals`를 이미 `INT_MAX`, `full_count`를 이미 contribution이 반영된 상태로 seed합니다. substituted sleep은 eating interval 하나를 완료시키고 다음 sleep에서 routine을 종료합니다. 결과는 `meals == INT_MAX + 1`, `full_count` unchanged여야 합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| former numeric boundary | §12 완료 기록의 대응 근거 참조 |
| seeded philosopher state | §12 완료 기록의 대응 근거 참조 |
| seeded global completion state | §12 완료 기록의 대응 근거 참조 |
| substituted sleep sequence | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production meal-completion path | §12 완료 기록의 대응 근거 참조 |
| `INT_MAX + 1` assertion | §12 완료 기록의 대응 근거 참조 |
| unchanged `full_count` assertion | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic boundary regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] 두-philosopher table이 필요한 이유와 second philosopher state를 확인합니다.
- [x] `must_eat == INT_MAX`, target philosopher `meals == INT_MAX` seed를 확인합니다.
- [x] `full_count`가 이미 해당 philosopher의 threshold contribution을 포함하도록 어떤 값으로 설정되는지 확인합니다.
- [x] sleep stub이 첫 eating wait를 success로, 후속 sleep을 termination으로 만드는 call sequence를 확인합니다.
- [x] production `record_meal_done` 또는 equivalent path가 실제로 `INT_MAX + 1` increment를 수행하는지 확인합니다.
- [x] assertion expression이 wider type에서 `INT_MAX + 1`을 계산하도록 작성되었는지 확인합니다.
- [x] `full_count`가 증가하지 않는 assertion을 확인합니다.
- [x] test가 duplicate contribution bug와 numeric overflow bug를 동시에 감지하는 이유를 설명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- former `int` boundary에서 한 번 더 meal을 완료했을 때 internal counter가 defined `INT_MAX + 1` 값을 갖는지 검증합니다.
- target 초과 후 같은 philosopher가 `full_count`에 두 번째로 기여하지 않는지 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- arbitrarily long execution의 모든 numeric boundary를 검증하지 않습니다.
- multi-thread schedule, fairness, monitor timing을 검증하지 않습니다.

#### 학습자 결론
- [x] seeded state가 실제 valid execution의 어느 순간을 모델링하는지 설명합니다.
- [x] numeric widening만 통과하고 threshold accounting이 깨진 구현도 이 test가 실패시키는 이유를 설명합니다.
- [x] Thread 전체의 meal progress가 fork acquisition에서 committed, range-safe state로 발전한 과정을 요약합니다.

## 6. 항상 유지해야 하는 조건 ledger

| 항상 유지해야 하는 조건 | 최초 도입 또는 부족함 | 강화·복구 | regression evidence | 해당 SHA 코드 근거 | 최종 설명 |
| --- | --- | --- | --- | --- | --- |
| multi-worker fork order는 uniform all-left circular wait를 깨뜨림 | `b68f40819af4` | 이후 Thread commits는 이 order를 유지 | 이 Thread source에는 전용 order test commit 없음 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| single philosopher는 같은 mutex를 재잠금하지 않음 | `b68f40819af4`에서 distinct-fork 가정 부족 | `c8531c91f0fb` | 이 Thread source에는 전용 deterministic test 없음 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| final target 도달과 global completion publication이 같은 state transaction | 최초 routine/monitor polling으로 gap 존재 | `fe0a2d15b29b` | 직접 test commit은 이 Thread에 없음 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| terminal 이후 새 meal 또는 post-completion sleep/think로 진행하지 않음 | 최초 routine에서 부족 | `fe0a2d15b29b` | 관련 code path 직접 확인 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| meal은 full eating interval과 active commit point를 통과해야 count | `fe0a2d15b29b`까지 interrupted wait count 가능 | `53e591effb4a` | `73b5551a76f4` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| 모든 aborted eating path는 acquired fork를 release | 최초 routine의 exit path에서 확인 시작 | `53e591effb4a`에서 logical abort와 결합 | `73b5551a76f4`의 production locking path | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| internal meal counter는 public target 이후에도 defined range 유지 | `int` counter로 valid overflow 위험 | `4c224ae86f2b` | `054ef46f80c7` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| philosopher는 `full_count`에 threshold equality에서 한 번만 기여 | `b68f40819af4`에서 최초 규칙 | `fe0a2d15b29b`, `4c224ae86f2b`에서 유지 | `054ef46f80c7` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

## 7. Failure → Fix → Test 연결

### 7.1 Ring 같은 메모리를 가리키는 경우와 single-philosopher self-교착 상태

```text
`b68f40819af4`
normal two-fork acquisition
→ N == 1에서 left/right pointer alias
→ 같은 non-recursive mutex 재잠금
→ `c8531c91f0fb`
single path: lock once, one fork event, no meal, monitor death
```

- pointer alias를 만드는 initializer expression: §12 완료 기록의 대응 근거에 정리했습니다.
- self-교착 상태가 발생하는 두 lock call: §12 완료 기록의 대응 근거에 정리했습니다.
- dedicated branch의 lock/log/wait/unlock 순서: §12 완료 기록의 대응 근거에 정리했습니다.
- meal state가 변경되지 않는 근거: §12 완료 기록의 대응 근거에 정리했습니다.
- monitor가 death authority를 유지하는 근거: §12 완료 기록의 대응 근거에 정리했습니다.
- 이 fix가 fairness를 보장하지 않는 이유: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.2 Final meal publication과 interrupted meal

```text
monitor가 completion을 polling
→ final meal과 ended publication 사이 gap
→ `fe0a2d15b29b`
worker completion critical section에서 ended publish
→ 그러나 sleep interruption과 deadline completion을 구분하지 못함
→ `53e591effb4a`
sleep status + mutation 전 ended recheck
→ `73b5551a76f4`
interruption 주입 후 meals/full_count == 0
```

- `fe0a2d15b29b`이 닫은 polling gap: §12 완료 기록의 대응 근거에 정리했습니다.
- 같은 commit에 남은 interrupted-meal root cause: §12 완료 기록의 대응 근거에 정리했습니다.
- `53e591effb4a`의 operation commit point: §12 완료 기록의 대응 근거에 정리했습니다.
- abort path의 fork release: §12 완료 기록의 대응 근거에 정리했습니다.
- test stub이 production path에 들어가는 지점: §12 완료 기록의 대응 근거에 정리했습니다.
- local/global counter assertions: §12 완료 기록의 대응 근거에 정리했습니다.
- 이 연결이 증명하지 않는 integration 범위: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.3 Public target과 internal accumulated state

```text
must_eat <= INT_MAX
→ target 도달 philosopher가 다른 worker를 기다리며 추가 meal 가능
→ int meals increment에서 signed overflow 위험
→ `4c224ae86f2b`
internal meals를 int64_t로 확장
→ equality contribution 규칙 유지
→ `054ef46f80c7`
INT_MAX → INT_MAX + 1
+ full_count unchanged
```

- valid execution scenario: §12 완료 기록의 대응 근거에 정리했습니다.
- former overflow expression: §12 완료 기록의 대응 근거에 정리했습니다.
- widened field와 관련 호출 지점: §12 완료 기록의 대응 근거에 정리했습니다.
- equality check: §12 완료 기록의 대응 근거에 정리했습니다.
- test seed state: §12 완료 기록의 대응 근거에 정리했습니다.
- stubbed sleep sequence: §12 완료 기록의 대응 근거에 정리했습니다.
- two final assertions: §12 완료 기록의 대응 근거에 정리했습니다.
- 여전히 남는 numeric/non-progress 한계: §12 완료 기록의 대응 근거에 정리했습니다.

## 8. 소유권 / state / responsibility 변화

| 시점 | Fork 소유권 | Meal state producer | Global completion responsibility | Operation commit 기준 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| `b68f40819af4` | worker가 두 shared mutex를 일시 보유 | worker가 start/completion state 갱신 | monitor가 후속 전역 정책을 관찰 | eating wait 뒤 counter 증가 | §12 완료 기록의 대응 근거 참조 |
| `c8531c91f0fb` | single worker는 동일 mutex를 한 번만 보유 | meal state 변경 없음 | monitor가 death 담당 | meal 시작 불가 | §12 완료 기록의 대응 근거 참조 |
| `fe0a2d15b29b` | acquisition 후 terminal이면 즉시 release | final meal worker가 `ended`까지 publish | final quota transition은 worker critical section | wait interruption 구분은 아직 없음 | §12 완료 기록의 대응 근거 참조 |
| `53e591effb4a` | 모든 aborted eating exit에서 release | completed interval + active state일 때만 mutation | 기존 completion model 유지 | explicit commit boundary | §12 완료 기록의 대응 근거 참조 |
| `4c224ae86f2b` | 변화 없음 | wider internal counter | equality contribution 유지 | numeric state defined range 확장 | §12 완료 기록의 대응 근거 참조 |

## 9. Thread 최종 상태

### Source-confirmed 최종 상태

- multi-worker routine은 parity-dependent fork order로 uniform all-left circular wait 구조를 깨뜨립니다.
- one-philosopher path는 동일 mutex를 재잠금하지 않고 하나의 fork만 보유한 채 monitor death를 기다립니다.
- final meal completion은 locked worker path에서 global `ended`까지 publish됩니다.
- meal은 full eating interval과 synchronized active-state check를 통과한 경우에만 local/global progress로 commit됩니다.
- internal `meals`는 `int64_t`이며 public `INT_MAX` target 이후에도 defined progress를 유지하고 threshold contribution을 반복하지 않습니다.
- 이 Thread는 scheduler fairness 또는 기아 상태 freedom을 보장하지 않습니다.

### 학습자가 작성할 최종 설명

- fork lock graph: §12 완료 기록의 대응 근거에 정리했습니다.
- one-philosopher execution: §12 완료 기록의 대응 근거에 정리했습니다.
- meal-start state: §12 완료 기록의 대응 근거에 정리했습니다.
- meal-completion transaction: §12 완료 기록의 대응 근거에 정리했습니다.
- global quota transition: §12 완료 기록의 대응 근거에 정리했습니다.
- interruption abort: §12 완료 기록의 대응 근거에 정리했습니다.
- fork cleanup: §12 완료 기록의 대응 근거에 정리했습니다.
- public target vs internal counter: §12 완료 기록의 대응 근거에 정리했습니다.
- guarantees: §12 완료 기록의 대응 근거에 정리했습니다.
- non-guarantees: §12 완료 기록의 대응 근거에 정리했습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
worker routine
    ↓ terminal 확인
    ↓ parity별 fork 1 획득 + log
    ↓ fork 2 획득 + log
    ↓ terminal 재확인
        ├─ ended → 두 fork release → stop
        └─ active
             ↓ last_meal_ms commit + is eating
             ↓ interruptible full-duration wait
                 ├─ interrupted → 두 fork release → no meal progress
                 └─ deadline reached
                      ↓ state_mutex
                      ↓ ended 재확인
                          ├─ ended → no progress
                          └─ active
                               ↓ meals increment
                               ↓ equality에서 full_count 1회 contribution
                               ↓ all full이면 ended publish
             ↓ 두 fork release
             ↓ terminal이면 stop
             ↓ sleeping
             ↓ thinking
             ↓ repeat

N == 1:
one fork lock → one fork log → no meal → wait/release → monitor death
```

- 실제 helper 분해: §12 완료 기록의 대응 근거에 정리했습니다.
- 각 lock acquisition/release: §12 완료 기록의 대응 근거에 정리했습니다.
- `last_meal_ms` mutation: §12 완료 기록의 대응 근거에 정리했습니다.
- sleep status: §12 완료 기록의 대응 근거에 정리했습니다.
- meal commit function: §12 완료 기록의 대응 근거에 정리했습니다.
- `full_count` equality: §12 완료 기록의 대응 근거에 정리했습니다.
- global `ended` publication: §12 완료 기록의 대응 근거에 정리했습니다.
- loop stop condition: §12 완료 기록의 대응 근거에 정리했습니다.
- `meals` type과 increment: §12 완료 기록의 대응 근거에 정리했습니다.
- single path symbol: §12 완료 기록의 대응 근거에 정리했습니다.

## 11. 학습 완료 자가 점검

- [x] `b68f40819af4`의 odd/even lock order와 state transition을 실제 code로 그렸습니다.
- [x] parity order와 initial delay의 보장 범위를 구분했습니다.
- [x] `c8531c91f0fb`의 pointer aliasing, one-lock path, monitor authority를 설명했습니다.
- [x] `fe0a2d15b29b`의 final meal critical section과 acquisition/post-meal terminal check를 설명했습니다.
- [x] 같은 commit에 interrupted-meal 결함이 남아 있음을 실제 call path로 확인했습니다.
- [x] `53e591effb4a`의 sleep status, mutation 전 recheck, fork cleanup을 transaction으로 설명했습니다.
- [x] `73b5551a76f4`의 direct invocation과 counter-zero assertions를 설명했습니다.
- [x] `4c224ae86f2b`의 public/internal range 분리를 valid scenario로 설명했습니다.
- [x] `054ef46f80c7`의 `INT_MAX + 1`과 unchanged `full_count`를 설명했습니다.
- [x] meal-start log를 completed quota progress로 취급하지 않았습니다.
- [x] 교착 상태 avoidance를 fairness 또는 기아 상태 freedom으로 과장하지 않았습니다.

## 12. 저장소 기반 완료 기록

### 12.1 검토 범위와 실행 상태

- 이 Thread의 7개 SHA는 모두 `c/philo` HEAD의 조상으로 확인했습니다.
- 각 routine, time helper, header, deterministic test는 해당 SHA의 파일을 기준으로 확인했습니다.
- 로컬 checkout 제한으로 production test binary는 실행하지 않았습니다. 아래 test 설명은 실제 injection과 assertion source를 분석한 결과입니다.

### 12.2 `b68f40819af4` — 최초 worker routine

#### worker execution order

`src/routine.c::philo_routine`은 terminal을 관찰하며 `eat_once → sleeping → thinking`을 반복합니다. `eat_once`는 두 fork를 잡은 뒤 meal start state를 publish하고 eating interval을 기다린 다음 meal count를 갱신하고 fork를 놓습니다.

#### parity-dependent fork order

| philosopher id | 첫 번째 fork | 두 번째 fork | 목적 |
| --- | --- | --- | --- |
| odd | `left_fork` | `right_fork` | 모든 worker가 left-first가 되는 uniform cycle을 끊습니다. |
| even | `right_fork` | `left_fork` | 적어도 인접 edge에서 반대 방향으로 acquisition합니다. |

각 successful lock 직후 `has taken a fork`를 기록합니다. even worker가 routine 시작 시 1 ms 대기하는 것은 initial contention을 낮추는 heuristic입니다. correctness는 parity order에 기대며 이 delay는 fairness, 기아 상태 freedom, 고정 schedule을 보장하지 않습니다.

#### meal state

- `record_meal_start`: `state_mutex` 아래 `last_meal_ms = philo_now_ms()`를 저장합니다.
- eating log: meal attempt 시작을 외부에 나타냅니다.
- `record_meal_done`: `state_mutex` 아래 `meals++`를 수행합니다.
- `meals == must_eat`인 최초 threshold equality에서만 `full_count++`를 수행합니다.

초기 구현은 `philo_sleep_ms`의 completion/interruption 결과를 받지 않습니다. terminal 때문에 eating wait가 일찍 끝나도 이어서 `record_meal_done`이 호출될 수 있습니다. 또한 `N == 1`에서 두 fork pointer가 같은 객체라는 edge를 구분하지 않습니다.

#### 최초 구현의 자원 경로

```text
terminal check
→ parity 순서로 fork 1 lock/log
→ fork 2 lock/log
→ last_meal_ms publish
→ is eating
→ wait
→ meals/full_count update
→ left/right unlock
→ is sleeping → wait
→ is thinking
```

normal path에서 fork를 놓지만, operation이 중간에 abort되는 명시적 transaction contract는 아직 없습니다.

### 12.3 `c8531c91f0fb` — one-philosopher aliasing fix

initializer ring mapping은 `N == 1`일 때 다음 결과를 만듭니다.

```text
left_fork  = &forks[0]
right_fork = &forks[(0 + 1) % 1] = &forks[0]
```

기존 two-lock path는 같은 non-recursive mutex를 한 thread가 연속 두 번 lock해 self-deadlock합니다. fix는 `wait_single_philo` 전용 경로를 추가합니다.

```text
left_fork 한 번 lock
→ has taken a fork 한 줄
→ time_to_die + 1 동안 terminal-aware wait
→ left_fork 한 번 unlock
→ worker return
```

이 path는 `last_meal_ms`, `meals`, `full_count`, `is eating`을 변경하지 않습니다. 실제 fork가 하나뿐이라는 constraint를 유지하며 가짜 두 번째 fork를 만들지 않습니다. death 판정과 terminal log는 계속 main-thread monitor가 담당합니다. wait가 terminal로 일찍 끝나도 unlock 뒤 return하므로 fork 소유권이 남지 않습니다.

### 12.4 `fe0a2d15b29b` — final quota와 global completion의 같은 transaction

#### 이전 gap

최초 routine에서는 worker가 target meal을 기록한 뒤 monitor가 `full_count == number`를 polling해 나중에 `ended`를 publish할 수 있었습니다. 이 시간 동안 다른 worker가 새 meal을 시작하거나 final completion 뒤 sleeping/thinking log로 진행할 수 있습니다.

#### 수정된 state transaction

`record_meal_done`은 `state_mutex` hold 안에서 다음을 연속 수행합니다.

1. `philo->meals++`
2. `meals == must_eat`이면 `full_count++`
3. `full_count >= number`이면 `ended = 1`

final required meal과 global terminal publication 사이에 별도 monitor polling gap이 없습니다. equality check를 유지하므로 target을 초과한 같은 philosopher가 `full_count`에 다시 기여하지 않습니다.

#### two rechecks

- 두 fork 획득 직후 `philo_has_ended`를 다시 확인합니다. fork를 기다리는 동안 다른 worker가 terminal을 commit했으면 두 fork를 모두 release하고 meal을 시작하지 않습니다.
- `eat_once` 반환 뒤 routine은 terminal을 확인하고, final completion이면 sleeping/thinking으로 진행하지 않습니다.

이 commit이 닫은 것은 final quota publication과 post-completion work gap입니다. `philo_sleep_ms`가 아직 completion status를 반환하지 않으므로 interrupted eating을 count하는 결함은 남아 있습니다.

### 12.5 `53e591effb4a` — eating attempt와 committed meal 분리

#### sleep operation contract

`src/time.c::philo_sleep_ms`는 `int` status를 반환합니다.

- `now >= deadline`: `PHILO_OK`
- `ended` 관찰: `PHILO_ERR`

따라서 caller는 configured interval이 실제로 끝났는지 terminal interruption으로 중단됐는지 구분합니다.

#### `eat_once` transaction

```text
lock_forks
→ terminal recheck
    ├─ terminal: unlock_forks + PHILO_ERR
    └─ active
         → record_meal_start
         → is eating
         → philo_sleep_ms(time_to_eat)
             ├─ PHILO_ERR: unlock_forks + no counter mutation
             └─ PHILO_OK
                  → record_meal_done
                      ├─ ended recheck 실패: no mutation
                      └─ active: meals/full_count/ended commit
                  → unlock_forks
```

`record_meal_done`은 `state_mutex`를 획득한 뒤 mutation 전에 `table->ended`를 다시 확인합니다. 이유는 sleep이 deadline에 도달해 `PHILO_OK`를 반환한 직후, counter lock을 얻기 전에 다른 thread가 terminal을 commit할 수 있기 때문입니다. sleep result만으로는 commit 권한이 충분하지 않습니다.

모든 logical abort path에서 `unlock_forks`를 호출합니다. 즉 meal accounting 실패와 fork cleanup은 독립된 항상 유지해야 하는 조건으로 함께 유지됩니다.

#### commit 조건

한 meal이 count되려면 두 조건을 모두 만족해야 합니다.

1. configured eating interval이 deadline까지 완료됐습니다.
2. `state_mutex` 아래 counter mutation 직전 simulation이 active입니다.

`is eating` log, 두 fork 보유, `last_meal_ms` 갱신은 operation start를 나타낼 뿐 committed meal progress는 아닙니다.

### 12.6 `73b5551a76f4` — interrupted-meal regression

`tests/interrupted_meal.c`는 routine object가 호출하는 `philo_sleep_ms`를 test stub으로 바꿉니다.

- table은 barrier가 이미 release된 상태로 seed됩니다.
- `meals = 0`, `full_count = 0`입니다.
- fork mutex와 production locking path는 그대로 사용합니다.
- sleep stub은 `state_mutex` 아래 `ended = 1`을 publish하고 `PHILO_ERR`를 반환합니다.
- worker routine을 직접 호출해 target path만 통과시킵니다.

assertion은 worker return 뒤 `meals == 0`, `full_count == 0`입니다. test가 fork unlock을 별도 trylock으로 직접 검증하는지 여부와 관계없이 production abort path가 `unlock_forks`를 호출하는 것은 source에서 확인됩니다. 이 test의 강점은 scheduler timing에 기대지 않고 eating wait의 interruption branch를 직접 만드는 것입니다. full executable의 monitor/barrier/join integration은 증명하지 않습니다.

### 12.7 `4c224ae86f2b` — public target과 internal accumulation의 range 분리

#### valid overflow scenario

`must_eat`은 public input으로 `INT_MAX`까지 허용됩니다. 한 philosopher가 `INT_MAX`에 먼저 도달해 `full_count`에 한 번 기여했더라도 다른 philosopher가 아직 target에 도달하지 않았다면 simulation은 끝나지 않습니다. 먼저 도달한 philosopher가 한 번 더 먹으면 internal state는 `INT_MAX + 1`이어야 합니다.

기존 `int meals`에서 이 increment는 signed overflow undefined 동작이 될 수 있습니다. `state_mutex`는 concurrent access를 serialize하지만 numeric overflow를 정의된 연산으로 바꾸지 않습니다.

#### 변경

`include/philo.h`에서 `t_philo.meals`만 `int64_t`로 넓힙니다.

- `t_config.must_eat`은 계속 `int`입니다.
- public parser bound도 `INT_MAX`입니다.
- increment는 `int64_t` storage에서 수행됩니다.
- comparison `meals == must_eat`는 target 도달 시 한 번만 true입니다.
- target 초과 후에는 equality가 false이므로 duplicate `full_count` contribution이 없습니다.

이 변경은 practical internal range를 넓히지만 무한 실행에서 `int64_t`가 절대 overflow하지 않는다고 보장하지 않습니다.

### 12.8 `054ef46f80c7` — former boundary regression

`tests/meal_counter_range.c`는 실제로 도달 가능한 target-after-completion 상태를 직접 seed합니다.

| state | 초기값 |
| --- | --- |
| philosopher count | 2 |
| target philosopher `meals` | `INT_MAX` |
| `must_eat` | `INT_MAX` |
| global `full_count` | 1, 즉 이 philosopher의 기여가 이미 반영됨 |
| simulation | 다른 philosopher가 아직 full이 아니므로 active |

substituted sleep은 첫 eating interval을 `PHILO_OK`로 완료시키고, 다음 sleep call에서 terminal을 만들어 routine을 끝냅니다. production `record_meal_done`이 한 번 실행된 뒤 기대값은 다음입니다.

- `meals == (int64_t)INT_MAX + 1`
- `full_count == 1`
- configured sleep-call sequence가 예상 횟수대로 진행됨

첫 assertion은 wider accumulation을, 두 번째 assertion은 target 초과 후 duplicate contribution 부재를 검사합니다. 단순히 field type만 바꾸고 equality logic을 손상한 구현도 실패합니다.

### 12.9 항상 유지해야 하는 조건 evolution 완성

| 항상 유지해야 하는 조건 | 도입·문제 | 수정 | regression evidence | 최종 해석 |
| --- | --- | --- | --- | --- |
| multi-worker가 uniform all-left circular wait를 만들지 않음 | `b68f40819af4` parity order | 이후 유지 | 전용 deterministic order test 없음 | deadlock의 한 필요 조건을 깨지만 fairness 증명은 아님 |
| `N == 1`은 같은 mutex를 재잠금하지 않음 | 최초 routine의 distinct-pointer 가정 | `c8531c91f0fb` one-lock path | Thread 내 전용 test 없음 | 실제 하나의 fork constraint를 보존 |
| final quota와 global ended가 같은 state transaction | monitor polling gap | `fe0a2d15b29b` worker completion section | source path inspection | post-completion new work 억제 |
| terminal 이후 fork 보유만으로 새 meal을 시작하지 않음 | fork wait 중 terminal 가능 | acquisition 후 recheck | routine path inspection | fork 소유권과 operation authorization 분리 |
| meal은 completed interval + active commit point에서만 count | interrupted wait도 count 가능 | `53e591effb4a` status + locked recheck | `73b5551a76f4` | attempt와 committed progress 분리 |
| aborted meal은 fork를 남기지 않음 | explicit abort path 부족 | 모든 error branch `unlock_forks` | interrupted test의 production path | logical rollback과 resource release 결합 |
| internal counter는 public target 이후 defined range 유지 | valid `INT_MAX + 1`에서 int overflow | `int64_t meals` | `054ef46f80c7` | public contract와 runtime accumulation 분리 |
| `full_count` contribution은 philosopher당 한 번 | threshold equality로 도입 | fixes에서도 equality 유지 | boundary test의 unchanged count | target 초과 meal은 duplicate global progress 아님 |

### 12.10 최종 worker flow

```text
wait_for_start
→ N == 1 ?
    ├─ yes: one fork lock/log → no meal → wait/unlock → monitor death
    └─ no
         ↓ optional even-id 1 ms stagger
         ↓ terminal check
         ↓ parity order로 fork 1 lock/log
         ↓ fork 2 lock/log
         ↓ terminal recheck
             ├─ ended: both unlock → stop
             └─ active
                  ↓ state_mutex: last_meal_ms = now
                  ↓ is eating
                  ↓ full-duration interruptible wait
                      ├─ interrupted: both unlock → no progress
                      └─ deadline complete
                           ↓ state_mutex
                           ↓ ended recheck
                               ├─ ended: no progress
                               └─ active
                                    ↓ int64_t meals++
                                    ↓ meals == must_eat이면 full_count++
                                    ↓ full_count == number이면 ended = 1
                  ↓ both forks unlock
                  ↓ ended면 stop
                  ↓ is sleeping / wait
                  ↓ is thinking
                  ↓ repeat
```

### 12.11 최종 보장과 비보장

보장하는 범위:

- multi-worker acquisition order는 모든 worker가 같은 left-first cycle을 만드는 구조를 피합니다.
- one-philosopher path는 동일 mutex를 두 번 잠그지 않습니다.
- fork 대기 중 terminal이 확정되면 새 meal을 시작하지 않습니다.
- full eating interval과 active state recheck를 통과한 경우만 counter에 반영합니다.
- interrupted/aborted meal은 local `meals`와 global `full_count`를 변경하지 않고 fork를 해제합니다.
- `meals`는 `int64_t`여서 valid `INT_MAX + 1` state를 표현하고, `full_count`는 equality에서 한 번만 증가합니다.

보장하지 않는 범위:

- parity order와 initial stagger는 scheduler fairness나 기아 상태 freedom을 보장하지 않습니다.
- `is eating` line은 completed meal 증거가 아닙니다.
- `int64_t`는 무한 accumulation을 보장하지 않습니다.
- deterministic 단위 테스트는 full executable의 모든 interleaving을 포괄하지 않습니다.
===== END FILE: 03-core-routine-to-committed-meal-progress.md =====

===== BEGIN FILE: 04-serialized-output-to-linearized-terminal-state.md =====
# Thread: Serialized output to linearized terminal state

이 문서는 source에 정의된 네 번째 Development Thread를 그대로 따릅니다. commit 순서, SHA, importance, tags는 변경하지 않습니다. monitor와 logger의 final lock order를 이전 SHA에 소급하지 않고, observation과 commitment 사이의 race가 어느 commit에서 남고 어느 commit에서 닫히는지 실제 코드로 확인합니다.

## 1. Thread 목표

이 Thread의 목표는 단순한 output serialization이 왜 terminal correctness에 충분하지 않은지 확인하고, authoritative monitor의 candidate observation이 fresh revalidation, terminal-state publication, final death line과 하나의 linearized transaction으로 결합되는 과정을 복원하는 것입니다.

Source-confirmed significance는 다음과 같습니다.

- `state_mutex`와 `print_mutex`는 terminal access와 complete-line output boundary를 만들지만 최초 death path는 두 책임을 원자적으로 결합하지 못합니다.
- main-thread monitor는 worker가 publish한 meal state를 읽고 global death/completion policy를 소유합니다.
- 최초 monitor는 predicate를 관찰한 lock을 놓은 뒤 terminal state를 publish하여 completion gap과 stale-death gap을 남깁니다.
- final fix는 completion을 locked state transaction으로 만들고, death를 `print_mutex → state_mutex`에서 fresh time과 latest meal state로 재확인합니다.
- terminal publication과 output serialization이 같은 lock order에 들어가면서 one-death, no-post-terminal ordinary status 항상 유지해야 하는 조건이 성립합니다.
- deterministic boundary test는 timing luck이 아니라 old unlock window에 직접 state change를 주입합니다.

### Source에 명시적으로 연결된 Critical 항상 유지해야 하는 조건

- `ended`, `full_count`, `meals`, `last_meal_ms`는 정의된 `state_mutex` 경계에서 관찰·변경됩니다.
- death candidate는 fresh time과 latest meal state로 terminal commit 직전에 다시 확인합니다.
- at most one death가 commit되며 terminal publication 뒤 ordinary status line은 시도되지 않습니다.
- `print_mutex`와 `state_mutex`의 common lock order는 terminal decision과 final output ordering을 결합합니다.

### Source에 명시적으로 연결된 Major Engineering Difficulties

- stale death candidate를 제거하면서 monitor의 global authority를 유지하는 문제
- terminal-state publication과 output을 linearize하여 `died` 뒤 ordinary line이 나오지 않게 하는 문제
- repeated run이 아니라 정확한 synchronization boundary에서 state를 바꾸는 deterministic test를 만드는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `print_mutex`로 line interleaving을 막는 것과 terminal decision을 linearize하는 것은 왜 다른가?
- normal logger와 initial death logger는 각각 어떤 lock 순서를 사용하며 race window는 어디입니까?
- worker가 publish하는 state와 monitor가 소유하는 policy decision은 어떻게 분리됩니까?
- completion predicate를 lock 안에서 보고 lock 밖에서 `ended`를 설정하면 어떤 gap이 생기는가?
- death candidate가 scan 뒤 meal을 시작하면 initial observation은 왜 더 이상 authoritative하지 않은가?
- final death path가 왜 `print_mutex`를 먼저 잡고 `state_mutex`를 다음에 잡는가?
- fresh monotonic time과 latest `last_meal_ms`는 어느 critical section에서 비교됩니까?
- normal logger가 먼저 print lock을 얻는 경우와 death path가 먼저 얻는 경우 모두에서 line order는 어떻게 결정됩니까?
- at-most-one death commitment와 actual output delivery는 어떻게 구분해야 하는가?
- boundary injection test는 completion atomicity와 stale candidate rejection을 각각 어떻게 강제하는가?

## 3. 완료 기준

- [x] 최초 normal/death logger의 lock trace와 race window를 실제 code로 제시할 수 있습니다.
- [x] main-thread monitor와 workers의 responsibility split을 state producer/authority 관점으로 설명할 수 있습니다.
- [x] completion observation과 publication 사이 gap을 실제 unlock/call 순서로 재구성할 수 있습니다.
- [x] stale death candidate interleaving을 meal-state mutation과 함께 설명할 수 있습니다.
- [x] `philo_try_log_death`의 fresh recheck와 exact lock order를 설명할 수 있습니다.
- [x] normal logger와 death path의 두 경쟁 순서 모두에서 terminal order를 증명할 수 있습니다.
- [x] completion publication이 state unlock 전에 끝나는 코드를 제시할 수 있습니다.
- [x] boundary test의 두 mode와 핵심 positive/negative assertion을 설명할 수 있습니다.
- [x] one-death/no-post-terminal guarantee와 strict latency/output failure non-guarantee를 구분할 수 있습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `033ad537d166` | `feat(log): 상태 로그의 동시 출력 보호` | A | `TERMINAL_STATE, CONCURRENCY, ARCH` | Introduces synchronized terminal-state access and a print mutex, but does not yet couple death publication to final output atomically. |
| 2 | `40ea0f871300` | `feat(monitor): 사망과 식사 완료 조건 감시` | S | `CORE, CONCURRENCY, TERMINAL_STATE` | Establishes the main-thread monitor as the authority for starvation and global completion. |
| 3 | `a2e90b84641b` | `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정` | S | `TERMINAL_STATE, CONCURRENCY, RISK` | Rechecks death under `print_mutex → state_mutex`, commits completion while locked, and gives terminal state explicit linearization points. |
| 4 | `c424b7d91ed1` | `test(monitor): 완료 상태와 오래된 사망 판정 검증` | A | `TEST, TERMINAL_STATE, DEBUG` | Mutates state at the old unlock boundary to prove stale candidates are rejected and completion is already terminal before release. |

## 5. Commit별 학습 기록

### 5.1 `033ad537d166` — `feat(log): 상태 로그의 동시 출력 보호`

- Importance: **A**
- Tags: `TERMINAL_STATE, CONCURRENCY, ARCH`
- Source-defined role: Introduces synchronized terminal-state access and a print mutex, but does not yet couple death publication to final output atomically.
- 코드 기준: 반드시 `033ad537d166` 시점
- 직접 parent 비교: `git diff 033ad537d166^ 033ad537d166 --`
- Thread 직전 관련 SHA: Thread 내 첫 commit

#### Source-confirmed 맥락

이 A-level commit은 terminal flag 접근과 status output을 state API 뒤로 이동합니다. `philo_has_ended`와 `philo_finish`는 `state_mutex`를 사용하고, normal status logger는 `print_mutex`를 잡은 뒤 terminal state를 다시 확인하고 하나의 complete line을 출력합니다. dedicated death path는 `ended`를 한 번만 변경하고 guard로 death line을 최대 한 번 출력하려고 합니다.

하지만 terminal publication과 death output은 하나의 atomic logging transaction이 아닙니다. death path는 `state_mutex`를 놓은 뒤 `print_mutex`를 획득합니다. 이미 print lock을 가진 normal logger가 termination check를 통과한 뒤 death path가 기다리는 interleaving에서는 death publication과 output order가 분리될 수 있습니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 문제 | 여러 worker의 log field가 interleave되거나 terminal 이후 ordinary status가 출력될 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 핵심 결정 | terminal state access를 `state_mutex` 경계에 두고 output을 `print_mutex`로 serialize합니다. | §12 완료 기록의 대응 근거 참조 |
| 얻는 boundary | normal line은 print lock 안에서 terminal recheck와 complete output을 수행합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 race | death state publication과 final line output 사이에 lock gap이 있어 normal logger와 순서가 엇갈릴 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 후속 연결 | `a2e90b84641b`이 `print_mutex → state_mutex` common order에서 death를 revalidate하고 terminal publication과 final line을 결합합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `git show --name-status 033ad537d166`로 state API와 logger 구현 파일을 식별합니다.
- [x] `philo_has_ended`가 `state_mutex`를 acquire/read/release하는 실제 순서를 확인합니다.
- [x] `philo_finish`가 `ended`를 언제 검사하고 언제 변경하는지 확인합니다.
- [x] normal logger의 `print_mutex` acquire, terminal recheck, timestamp calculation, `printf`, unlock 순서를 기록합니다.
- [x] normal logger의 terminal recheck가 직접 state lock을 중첩하는지 helper를 통하는지 actual call graph로 확인합니다.
- [x] dedicated death path가 `state_mutex`와 `print_mutex`를 각각 언제 잡고 놓는지 순서도로 그립니다.
- [x] death guard가 at-most-one attempt를 어떤 state로 표현하는지 확인합니다.
- [x] normal logger가 print lock을 가진 채 termination check를 통과하고 death path가 대기하는 interleaving을 실제 lock 순서로 재구성합니다.
- [x] timestamp가 table start 기준으로 계산되는 위치와 output line 전체가 print lock 안에 있는지 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Logger lock trace

| Path | 첫 lock | state check/mutation | 두 번째 lock 또는 output | unlock 순서 | race window |
| --- | --- | --- | --- | --- | --- |
| normal status | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| death path | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### 문제 interleaving 기록

```text
normal logger: print lock 획득
normal logger: ended == false 확인
death path:    ended publish
death path:    print lock 대기
normal logger: ordinary line 출력
normal logger: print unlock
death path:    died 출력
```

실제 구현의 lock acquire/release와 state mutation에 맞게 위 순서를 수정합니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- normal status line의 field interleaving을 `print_mutex`로 막습니다.
- terminal flag에 대한 직접 unsynchronized access를 state API로 줄입니다.
- death state를 한 번만 설정하고 death line을 최대 한 번 시도하려는 guard를 둡니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- terminal publication과 death line이 하나의 linearized transaction인 것은 아닙니다.
- stale death candidate revalidation은 아직 없습니다.
- death가 final successful status attempt가 된다고 아직 보장하지 않습니다.

#### 학습자 결론
- [x] 단순 output serialization과 terminal decision serialization이 다른 문제인 이유를 설명합니다.
- [x] normal/death path의 lock 순서를 실제 code로 나란히 제시합니다.
- [x] 후속 fix가 왜 새로운 print mutex가 아니라 existing locks의 order와 revalidation을 바꾸는지 설명합니다.

### 5.2 `40ea0f871300` — `feat(monitor): 사망과 식사 완료 조건 감시`

- Importance: **S**
- Tags: `CORE, CONCURRENCY, TERMINAL_STATE`
- Source-defined role: Establishes the main-thread monitor as the authority for 기아 상태 and global completion.
- 코드 기준: 반드시 `40ea0f871300` 시점
- 직접 parent 비교: `git diff 40ea0f871300^ 40ea0f871300 --`
- Thread 직전 관련 SHA: `033ad537d166`

#### Source-confirmed 맥락

이 S-level commit은 calling thread를 authoritative monitor로 두고 global termination policy를 worker routine에서 분리합니다. workers는 자신의 `last_meal_ms`, `meals`, `full_count`에 필요한 state를 publish하고, monitor는 `state_mutex` 아래에서 optional completion과 기아 상태를 관찰합니다. completion이면 run을 끝내고, current sampled time과 last-meal state로 첫 death candidate를 선택합니다.

최초 구현은 observation과 commitment를 분리합니다. completion predicate를 lock 안에서 확인한 뒤 unlock 후 `philo_finish`를 호출하고, sampled time으로 candidate를 고른 뒤 state critical section 밖에서 death path를 호출합니다. 그 사이 state가 바뀌어 completion publication이 늦어지거나 candidate가 stale해질 수 있습니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 문제 | local worker state만으로 simulation-wide death 또는 all-meals completion을 결정할 수 없습니다. | §12 완료 기록의 대응 근거 참조 |
| responsibility decision | calling thread monitor가 global terminal policy의 유일한 authority가 됩니다. | §12 완료 기록의 대응 근거 참조 |
| worker 역할 | 자신의 meal progress와 기아 상태 reference를 synchronized state로 publish합니다. | §12 완료 기록의 대응 근거 참조 |
| monitor 역할 | completion predicate와 각 philosopher의 elapsed 기아 상태를 관찰해 candidate를 선택합니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 race 1 | completion 관찰 후 unlock과 `ended` publication 사이에 gap이 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 남은 race 2 | death candidate scan 후 actual death commit 사이에 meal start 등 state change가 생길 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| 후속 수정 | `a2e90b84641b`이 completion을 lock 안에서 commit하고 death를 fresh recheck합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `033ad537d166` 대비 monitor translation unit과 public entry point를 확인합니다.
- [x] monitor loop가 terminal state, `full_count`, meal limit을 어떤 순서로 검사하는지 확인합니다.
- [x] philosopher array scan 중 current time을 한 번 sampling하는지, 각 philosopher마다 sampling하는지 실제 code로 확인합니다.
- [x] `last_meal_ms`와 `time_to_die` 비교식 및 equality boundary를 기록합니다.
- [x] candidate가 어떤 값 또는 pointer로 critical section 밖까지 전달되는지 확인합니다.
- [x] completion predicate가 true일 때 `state_mutex` unlock과 `philo_finish` 호출 순서를 확인합니다.
- [x] death candidate 발견 시 unlock과 death logger 호출 순서를 확인합니다.
- [x] monitor polling의 500-microsecond yield 위치를 확인합니다.
- [x] worker가 peer death를 판단하는 코드가 없는지 responsibility boundary를 확인합니다.
- [x] sampled candidate가 lock release 후 meal start를 갱신하는 구체적 interleaving을 작성합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Worker–monitor responsibility split

| State 또는 decision | Producer | Observer/authority | Lock boundary | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| `last_meal_ms` | worker | monitor | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| per-worker meals | worker | monitor/worker completion logic | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| `full_count` | worker | monitor | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| death candidate | monitor | monitor | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| global `ended` | monitor/state API | workers/loggers | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

#### Observation–commit gaps

| Predicate | 관찰 시 lock | unlock 뒤 호출 | 그 사이 바뀔 수 있는 state | 잘못된 결과 |
| --- | --- | --- | --- | --- |
| all meals complete | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| philosopher dead | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- worker-local state production과 simulation-wide terminal policy가 분리됩니다.
- calling thread monitor가 death 및 optional global completion의 authoritative observer가 됩니다.
- monitor는 shared meal state를 `state_mutex` 경계에서 관찰합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- completion predicate와 `ended` publication이 같은 transaction인 것은 아닙니다.
- death candidate가 commit 시점에도 유효하다고 보장하지 않습니다.
- death line 이후 ordinary log가 없다고 보장하지 않습니다.
- polling interval은 strict detection latency guarantee가 아닙니다.

#### 학습자 결론
- [x] worker가 peer death를 결정하지 않고 monitor가 global policy를 소유하는 이유를 설명합니다.
- [x] state production과 policy interpretation을 실제 symbols로 구분합니다.
- [x] candidate discovery와 terminal commitment 사이 gap이 stale decision을 만드는 interleaving을 제시합니다.
- [x] 이 S commit의 architecture는 유지되면서 후속 fix가 commitment protocol만 강화하는 이유를 설명합니다.

### 5.3 `a2e90b84641b` — `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정`

- Importance: **S**
- Tags: `TERMINAL_STATE, CONCURRENCY, RISK`
- Source-defined role: Rechecks death under `print_mutex → state_mutex`, commits completion while locked, and gives terminal state explicit linearization points.
- 코드 기준: 반드시 `a2e90b84641b` 시점
- 직접 parent 비교: `git diff a2e90b84641b^ a2e90b84641b --`
- Thread 직전 관련 SHA: `40ea0f871300`

#### Source-confirmed 맥락

이 S-level fix는 terminal-state publication에 명시적인 linearization point를 부여합니다. meal completion은 `state_mutex`를 유지한 채 predicate 확인과 `ended` publication을 수행합니다.

death는 two-stage operation이 됩니다. monitor scan은 candidate를 advisory하게 고를 수 있지만, `philo_try_log_death`가 `print_mutex`를 먼저, `state_mutex`를 다음에 획득하고 fresh monotonic time과 latest `last_meal_ms`로 candidate를 다시 검사합니다. 아직 terminal이 아니고 기아 상태 predicate가 여전히 참일 때만 `ended`를 설정하고 death timestamp를 계산해 `died` line을 출력합니다. normal logger와 동일한 nested lock order가 terminal transition과 output을 serialize합니다.


#### 변화 연결

| 단계 | Source-confirmed 기준 | 해당 SHA 코드 근거 |
| --- | --- | --- |
| 기존 가정 | monitor scan에서 dead로 보인 candidate는 lock을 놓은 뒤에도 유효하고, state publication과 print를 분리해도 final order가 유지됩니다. | §12 완료 기록의 대응 근거 참조 |
| 실제 failure/위험 | candidate가 그 사이 meal을 시작할 수 있고, normal logger가 death publication 전 check를 통과해 `died` 뒤 ordinary line을 만들 수 있습니다. | §12 완료 기록의 대응 근거 참조 |
| root cause | observation 시점과 terminal commit 시점이 분리되고 state/output lock order가 하나의 transaction을 형성하지 않습니다. | §12 완료 기록의 대응 근거 참조 |
| completion fix | all-full predicate와 `ended` publication을 같은 state critical section에서 수행합니다. | §12 완료 기록의 대응 근거 참조 |
| death fix | `print_mutex → state_mutex` 아래 fresh time, latest meal, `!ended`를 재검사합니다. | §12 완료 기록의 대응 근거 참조 |
| linearization point | revalidation이 성공한 critical section 안에서 `ended`를 설정하고 terminal line을 emit합니다. | §12 완료 기록의 대응 근거 참조 |
| output result | death commit 뒤 도착한 normal logger는 print boundary 안 state check에서 suppress됩니다. | §12 완료 기록의 대응 근거 참조 |
| regression 연결 | `c424b7d91ed1`이 old unlock boundary에 state mutation을 주입해 completion atomicity와 stale candidate rejection을 검증합니다. | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `40ea0f871300` 대비 monitor completion branch가 `state_mutex`를 놓기 전에 `ended`를 설정하도록 바뀐 diff를 확인합니다.
- [x] 기존 death logger가 `philo_try_log_death` 또는 equivalent boolean attempt로 교체되는 위치를 확인합니다.
- [x] `philo_try_log_death`의 exact lock order가 `print_mutex` 다음 `state_mutex`인지 확인합니다.
- [x] normal logger의 nested lock order와 death path의 order가 동일한지 call graph로 확인합니다.
- [x] death revalidation에서 fresh monotonic time을 다시 sampling하는 지점을 확인합니다.
- [x] `!ended`, candidate의 latest `last_meal_ms`, `time_to_die` 비교가 같은 critical section 안에 있는지 확인합니다.
- [x] stale candidate일 때 어떤 반환값으로 monitor loop가 계속되는지 확인합니다.
- [x] valid death일 때 `ended` mutation, timestamp calculation, `printf` 순서를 확인합니다.
- [x] state/print mutex가 valid death와 stale death에서 각각 어떤 순서로 unlock되는지 확인합니다.
- [x] normal logger가 print lock을 먼저 잡은 경우와 death path가 먼저 잡은 경우의 두 interleaving을 각각 작성합니다.
- [x] at-most-one death commitment가 guard와 lock serialization으로 어떻게 성립하는지 확인합니다.
- [x] output failure 자체는 source에서 해결되지 않은 non-guarantee이므로 terminal attempt와 physical delivery를 구분합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Linearization trace

| Case | `print_mutex` | `state_mutex` | revalidated state | mutation/output | result |
| --- | --- | --- | --- | --- | --- |
| normal logger wins first | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | ordinary line precedes death attempt |
| death path wins first | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | later ordinary logger suppressed |
| stale candidate | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | recent meal or terminal | no death commit | monitor continues |
| valid candidate | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | 기아 상태 still true | `ended` + `died` | unique terminal attempt |

#### Lock-order 근거

```text
ordinary log: print_mutex → state_mutex → check → output
death commit: print_mutex → state_mutex → fresh recheck → ended → died
```

- actual ordinary logger symbols: §12 완료 기록의 대응 근거에 정리했습니다.
- actual death symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- nested hold 범위: §12 완료 기록의 대응 근거에 정리했습니다.
- reverse order가 존재하지 않는지 검색 결과: §12 완료 기록의 대응 근거에 정리했습니다.


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- meal completion predicate와 terminal publication이 같은 state transaction에 있습니다.
- death candidate는 fresh time과 latest meal state로 commit 직전에 revalidate됩니다.
- stale candidate는 terminal death로 확정되지 않습니다.
- at most one death가 commit되고, terminal publication 뒤 ordinary status attempt는 suppress됩니다.
- common `print_mutex → state_mutex` order가 terminal decision과 output ordering을 결합합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- monitor의 initial scan 자체가 final truth인 것은 아닙니다.
- strict death-detection latency를 보장하지 않습니다.
- I/O failure나 모든 pthread failure를 해결했다는 뜻은 아닙니다.
- scheduler fairness 또는 기아 상태 freedom을 보장하지 않습니다.

#### 후속 regression evidence

- `c424b7d91ed1`은 completion unlock boundary에서 `ended`가 이미 true인지 관찰합니다.
- 같은 test의 stale-death mode는 initial scan 뒤 `last_meal_ms`와 completion state를 바꿔 final revalidation이 current state를 사용하는지 확인합니다.


#### 학습자 결론
- [x] candidate scan을 advisory로, `philo_try_log_death`를 authoritative commit으로 구분합니다.
- [x] linearization point를 특정 lock hold와 state mutation 순서로 제시합니다.
- [x] normal logger와 death path의 두 경쟁 순서 모두에서 no-post-terminal-log가 유지되는 이유를 설명합니다.
- [x] at-most-one death와 physical output success를 구분합니다.
- [x] monitor architecture, lock order, time model이 이 fix에서 어떻게 결합되는지 설명합니다.

### 5.4 `c424b7d91ed1` — `test(monitor): 완료 상태와 오래된 사망 판정 검증`

- Importance: **A**
- Tags: `TEST, TERMINAL_STATE, DEBUG`
- Source-defined role: Mutates state at the old unlock boundary to prove stale candidates are rejected and completion is already terminal before release.
- 코드 기준: 반드시 `c424b7d91ed1` 시점
- 직접 parent 비교: `git diff c424b7d91ed1^ c424b7d91ed1 --`
- Thread 직전 관련 SHA: `a2e90b84641b`

#### Source-confirmed test 역할

이 deterministic boundary test는 mutex unlock wrapper를 사용해 monitor가 state critical section을 놓는 정확한 경계를 관찰하거나 state를 변경합니다. completion mode에서는 unlock 시점에 `ended`가 이미 true인지 확인하여 predicate evaluation과 terminal publication이 하나의 transaction임을 검증합니다.

stale-death mode에서는 처음에 philosopher가 dead로 보이도록 만들고, monitor의 initial observation 뒤 old unlock boundary에서 `last_meal_ms`를 갱신하고 meal completion을 표시합니다. monitor는 candidate를 다시 검사해 valid completion으로 끝나야 하며 stale `died` line을 출력하면 안 됩니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| mutex unlock wrapper가 관찰하는 boundary | §12 완료 기록의 대응 근거 참조 |
| completion mode의 초기 state | §12 완료 기록의 대응 근거 참조 |
| completion mode의 핵심 assertion | §12 완료 기록의 대응 근거 참조 |
| stale-death mode의 초기 candidate state | §12 완료 기록의 대응 근거 참조 |
| boundary에서 주입하는 `last_meal_ms`/completion mutation | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production monitor와 death-attempt path | §12 완료 기록의 대응 근거 참조 |
| valid completion assertion | §12 완료 기록의 대응 근거 참조 |
| death line 부재 assertion | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| deterministic interleaving regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] mutex unlock wrapper가 어떤 mutex와 호출 위치를 대상으로 mode별 동작을 선택하는지 확인합니다.
- [x] completion mode에서 production monitor가 unlock을 호출하기 전에 `ended`를 이미 설정했는지 wrapper가 어떻게 읽는지 확인합니다.
- [x] stale-death mode의 initial `last_meal_ms`, sampled time, `time_to_die` 관계를 확인합니다.
- [x] old race boundary에서 `last_meal_ms`를 갱신하고 meal completion state를 만드는 injection을 확인합니다.
- [x] injection mutation 자체가 필요한 synchronization을 지키는지 확인합니다.
- [x] monitor가 initial candidate를 가진 뒤 `philo_try_log_death`에서 fresh state를 읽는 production path를 추적합니다.
- [x] run이 valid completion 상태로 끝나는 assertion을 확인합니다.
- [x] `died` line이 전혀 없어야 한다는 output assertion을 확인합니다.
- [x] repeated timing run이 아니라 exact interleaving을 강제하는 이유를 설명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- completion predicate 평가와 `ended` publication이 state unlock 전에 끝나는지 결정적으로 검증합니다.
- candidate discovery와 commitment 사이 state가 바뀌면 fresh revalidation이 stale death를 거부하는지 검증합니다.
- valid completion state가 stale death보다 우선해 terminal result가 되는지 검증합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 monitor schedule이나 output race를 포괄하지 않습니다.
- lock-order 전체를 정적으로 증명하지 않으며, target boundary의 production 동작을 검증합니다.
- strict detection latency나 fairness를 검증하지 않습니다.

#### 학습자 결론
- [x] wrapper가 old unlock boundary를 정확히 겨냥하는 방법을 설명합니다.
- [x] completion atomicity와 stale-death rejection 두 mode의 state setup을 분리해 기록합니다.
- [x] 이 test가 initial observation이 아니라 synchronized commit point를 correctness 기준으로 삼는다는 것을 설명합니다.

## 6. 항상 유지해야 하는 조건 ledger

| 항상 유지해야 하는 조건 | 최초 도입 또는 부족함 | 강화·복구 | regression evidence | 해당 SHA 코드 근거 | 최종 설명 |
| --- | --- | --- | --- | --- | --- |
| terminal state access는 `state_mutex` 경계를 사용 | `033ad537d166` | monitor와 final death path에서 유지 | `c424b7d91ed1` boundary observation | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| ordinary output은 complete line 단위로 serialize | `033ad537d166` | final common lock order와 결합 | 후속 concurrency evidence는 다른 Thread에 존재 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| monitor가 global death/completion policy의 authority | `40ea0f871300` | `a2e90b84641b`에서 advisory scan + authoritative commit으로 정교화 | `c424b7d91ed1` | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| completion predicate와 `ended` publication은 같은 state transaction | `40ea0f871300`에서 unlock gap 존재 | `a2e90b84641b` | `c424b7d91ed1` completion mode | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| death candidate는 fresh time/latest meal로 commit 직전 revalidate | `40ea0f871300`에서 stale candidate 가능 | `a2e90b84641b` | `c424b7d91ed1` stale-death mode | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| at most one death가 commit되고 terminal 뒤 ordinary status attempt가 없음 | `033ad537d166`에서 guard는 있으나 atomicity 부족 | `a2e90b84641b` | `c424b7d91ed1`은 stale decision을 직접 검증 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| death/normal logging은 `print_mutex → state_mutex` common order 사용 | 최초 logger boundary에서 불완전 | `a2e90b84641b` | actual lock trace와 별도 concurrency evidence로 확인 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

## 7. Failure → Fix → Test 연결

### 7.1 Output serialization만으로는 부족한 terminal ordering

```text
`033ad537d166`
state access + print serialization
→ death publication 뒤 print lock 획득
→ normal logger가 old terminal check를 통과할 window
→ `a2e90b84641b`
normal/death 모두 print_mutex → state_mutex
→ terminal commit과 final output을 같은 serialization boundary에 배치
```

- initial normal logger lock trace: §12 완료 기록의 대응 근거에 정리했습니다.
- initial death logger lock trace: §12 완료 기록의 대응 근거에 정리했습니다.
- terminal publication과 final death output 사이 ordinary attempt가 가능한 interleaving: §12 완료 기록의 대응 근거에 정리했습니다.
- final common order: §12 완료 기록의 대응 근거에 정리했습니다.
- death-first/normal-first 두 결과: §12 완료 기록의 대응 근거에 정리했습니다.
- physical I/O success와 state guarantee의 구분: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.2 Monitor observation과 terminal commitment

```text
`40ea0f871300`
state lock에서 completion/death candidate 관찰
→ unlock
→ finish/death path 호출
→ completion publication delay 또는 stale candidate
→ `a2e90b84641b`
completion은 lock 안에서 commit
death는 fresh time/latest meal로 revalidate
→ `c424b7d91ed1`
old unlock boundary에서 state mutation 주입
```

- completion old gap: §12 완료 기록의 대응 근거에 정리했습니다.
- stale-death old gap: §12 완료 기록의 대응 근거에 정리했습니다.
- final completion mutation point: §12 완료 기록의 대응 근거에 정리했습니다.
- final death revalidation predicate: §12 완료 기록의 대응 근거에 정리했습니다.
- stale candidate return path: §12 완료 기록의 대응 근거에 정리했습니다.
- test completion mode: §12 완료 기록의 대응 근거에 정리했습니다.
- test stale-death mode: §12 완료 기록의 대응 근거에 정리했습니다.
- no-death negative assertion: §12 완료 기록의 대응 근거에 정리했습니다.

## 8. 소유권 / state / responsibility 변화

| 시점 | Worker responsibility | Monitor responsibility | Logger responsibility | Terminal commit point | 학습자 코드 근거 |
| --- | --- | --- | --- | --- | --- |
| `033ad537d166` | terminal state를 확인하며 status 요청 | 아직 후속 도입 | print serialization과 초기 death path | state/output 분리 | §12 완료 기록의 대응 근거 참조 |
| `40ea0f871300` | meal state 생산 | completion/death candidate 관찰 | existing logger 호출 | observation 뒤 별도 publication | §12 완료 기록의 대응 근거 참조 |
| `a2e90b84641b` | synchronized meal state 생산 유지 | candidate scan은 advisory, final revalidation 호출 | common lock order에서 terminal output | locked completion 또는 revalidated death transaction | §12 완료 기록의 대응 근거 참조 |
| `c424b7d91ed1` test | production state를 boundary에서 자극 | current state를 사용해야 함 | stale death를 출력하지 않아야 함 | wrapper가 old gap을 직접 관찰 | §12 완료 기록의 대응 근거 참조 |

## 9. Thread 최종 상태

### Source-confirmed 최종 상태

- workers는 synchronized meal facts를 publish하고 main-thread monitor는 global terminal policy를 소유합니다.
- completion은 predicate가 true인 state critical section 안에서 `ended`로 commit됩니다.
- death scan은 advisory이며 final attempt가 `print_mutex → state_mutex`에서 fresh time과 latest meal state를 다시 확인합니다.
- valid death만 `ended`와 terminal line을 같은 serialization boundary에서 commit합니다.
- at most one death가 commit되고 terminal publication 뒤 ordinary status attempt는 suppress됩니다.
- strict detection latency, fairness, output call success는 이 항상 유지해야 하는 조건이 보장하는 범위가 아닙니다.

### 학습자가 작성할 최종 설명

- state producers: §12 완료 기록의 대응 근거에 정리했습니다.
- global policy authority: §12 완료 기록의 대응 근거에 정리했습니다.
- ordinary logger boundary: §12 완료 기록의 대응 근거에 정리했습니다.
- completion linearization: §12 완료 기록의 대응 근거에 정리했습니다.
- death revalidation: §12 완료 기록의 대응 근거에 정리했습니다.
- common lock order: §12 완료 기록의 대응 근거에 정리했습니다.
- unique terminal attempt: §12 완료 기록의 대응 근거에 정리했습니다.
- stale candidate handling: §12 완료 기록의 대응 근거에 정리했습니다.
- guarantees: §12 완료 기록의 대응 근거에 정리했습니다.
- non-guarantees: §12 완료 기록의 대응 근거에 정리했습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
workers
    ↓ state_mutex 아래 last_meal/meals/full_count publish

main-thread monitor
    ↓ state_mutex 아래 completion 확인
        ├─ all full
        │      → 같은 critical section에서 ended commit
        └─ death candidate 발견
               → scan은 advisory
               → philo_try_log_death
                    ↓ print_mutex
                    ↓ state_mutex
                    ↓ fresh monotonic time
                    ↓ !ended + latest last_meal recheck
                        ├─ stale/terminal
                        │      → no death commit
                        └─ still dead
                               → ended commit
                               → died line attempt
                    ↓ unlock state
                    ↓ unlock print

ordinary logger
    ↓ print_mutex
    ↓ state_mutex
    ↓ ended check
        ├─ ended → suppress
        └─ active → complete ordinary line
```

- actual monitor loop symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- completion condition: §12 완료 기록의 대응 근거에 정리했습니다.
- candidate representation: §12 완료 기록의 대응 근거에 정리했습니다.
- actual death-attempt symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- fresh time call: §12 완료 기록의 대응 근거에 정리했습니다.
- nested lock scope: §12 완료 기록의 대응 근거에 정리했습니다.
- normal logger symbol: §12 완료 기록의 대응 근거에 정리했습니다.
- stale return semantics: §12 완료 기록의 대응 근거에 정리했습니다.
- death output call: §12 완료 기록의 대응 근거에 정리했습니다.
- unlock order: §12 완료 기록의 대응 근거에 정리했습니다.

## 11. 학습 완료 자가 점검

- [x] `033ad537d166`의 state API, normal logger, death logger lock 순서를 당시 코드로 확인했습니다.
- [x] initial terminal/output gap에서 ordinary line이 뒤늦게 나오는 interleaving을 설명했습니다.
- [x] `40ea0f871300`의 worker–monitor responsibility split을 state producer와 authority로 구분했습니다.
- [x] completion과 death observation–commit gap을 실제 unlock/call 순서로 제시했습니다.
- [x] `a2e90b84641b`의 completion commit과 death fresh revalidation을 설명했습니다.
- [x] `print_mutex → state_mutex` common order를 normal/death path 모두에서 확인했습니다.
- [x] stale candidate, normal-first, death-first 세 case를 각각 설명했습니다.
- [x] `c424b7d91ed1`의 completion mode와 stale-death mode를 분리해 기록했습니다.
- [x] one-death/no-post-terminal 항상 유지해야 하는 조건을 strict latency 또는 guaranteed I/O success로 확대하지 않았습니다.
- [x] final HEAD의 lock order를 이전 SHA에 소급하지 않았습니다.

## 12. 저장소 기반 완료 기록

### 12.1 검토 범위와 실행 상태

- 이 Thread의 4개 SHA는 모두 `c/philo` HEAD의 조상으로 확인했습니다.
- normal logger, death logger, monitor, boundary test는 각 SHA의 구현을 직접 대조했습니다.
- test binary는 실행하지 못했으므로 runtime 통과를 주장하지 않습니다. 테스트 technique과 예상 결과는 source inspection에 근거합니다.

### 12.2 `033ad537d166` — line serialization과 terminal access의 첫 경계

#### state API

`src/state.c::philo_has_ended`는 `state_mutex`를 lock하고 `ended`를 읽은 뒤 unlock합니다. `philo_finish`도 같은 mutex 아래 `ended = 1`을 수행합니다. terminal flag의 direct unsynchronized access를 줄이는 첫 단계입니다.

#### normal logger

`philo_log`의 lock trace는 다음과 같습니다.

```text
print_mutex lock
→ philo_has_ended
     → state_mutex lock
     → ended read
     → state_mutex unlock
→ active이면 timestamp 계산 + printf 한 줄
→ print_mutex unlock
```

출력 전체가 `print_mutex` 안에 있으므로 여러 thread의 timestamp/id/message field가 한 line 안에서 섞이는 것을 막습니다. nested order는 `print_mutex → state_mutex`입니다.

#### initial death logger

`philo_log_death`는 다른 순서를 사용합니다.

```text
state_mutex lock
→ should_print = !ended
→ ended = 1
→ state_mutex unlock
→ should_print이면 print_mutex lock
→ timestamp + died printf
→ print_mutex unlock
```

terminal state publication과 terminal line 출력 사이에 lock gap이 있습니다. `should_print`가 false인 후속 caller는 death line을 시도하지 않으므로 at-most-one intent는 있지만, death line을 final line으로 만드는 serialization은 아직 없습니다.

#### race interleaving

```text
normal: print_mutex 획득
normal: state_mutex 아래 ended == 0 확인 후 state lock 해제
death:  state_mutex 획득, ended = 1, state lock 해제
death:  print_mutex 대기
normal: ordinary line 출력
normal: print_mutex 해제
death:  died line 출력
```

이 interleaving에서 ordinary line은 terminal publication 뒤에 실제 출력됩니다. line corruption은 없지만 terminal ordering은 잘못될 수 있습니다. 따라서 print serialization과 terminal-decision linearization은 별도 문제입니다.

### 12.3 `40ea0f871300` — main-thread monitor authority

#### responsibility split

| 사실 또는 결정 | producer | authority/consumer | 보호 |
| --- | --- | --- | --- |
| `last_meal_ms` | worker의 meal-start path | monitor | `state_mutex` |
| `meals`, `full_count` | worker completion path | monitor 및 global completion logic | `state_mutex` |
| 기아 상태 candidate | monitor scan | monitor | scan 시 `state_mutex` |
| global completion/death | monitor가 결정 | workers/loggers가 `ended`를 소비 | state API |

worker는 자신의 progress fact를 publish하고 peer death를 결정하지 않습니다. `philo_monitor`가 calling thread에서 simulation-wide policy를 소유합니다.

#### monitor loop

`src/monitor.c`는 loop마다 `now = philo_now_ms()`를 sampling하고 `state_mutex`를 잡습니다.

1. meal limit가 있고 `full_count >= number`인지 봅니다.
2. 아니면 philosopher 배열을 순회하며 `now - last_meal_ms >= time_to_die`인 첫 candidate를 선택합니다.
3. state lock을 놓습니다.
4. completion이면 `philo_finish`를, death candidate면 `philo_log_death`를 호출합니다.
5. 아무것도 아니면 `usleep(500)` 후 반복합니다.

#### completion observation–commit gap

```text
state_mutex 아래 all_meals_done == true
→ state_mutex unlock
→ philo_finish가 state_mutex를 다시 lock
→ ended = 1
```

두 lock 사이에는 worker가 ordinary work/log를 시도할 수 있는 gap이 있습니다. predicate 관찰과 publication이 한 transaction이 아닙니다.

#### stale death candidate gap

```text
state_mutex 아래 old now/last_meal로 candidate 선택
→ state_mutex unlock
→ candidate worker가 새 meal을 시작해 last_meal_ms 갱신 가능
→ philo_log_death는 starvation predicate를 다시 보지 않고 ended를 commit
```

initial scan은 관찰 시점에는 맞더라도 commit 시점의 current state가 아닐 수 있습니다. monitor authority 자체는 유지할 설계지만 commitment protocol은 부족합니다.

### 12.4 `a2e90b84641b` — terminal linearization

#### completion fix

`philo_monitor`는 `state_mutex`를 유지한 채 다음을 수행합니다.

```text
if (table->ended) return
if (all_meals_done(table)) {
    table->ended = 1;
    unlock;
    return;
}
```

all-full predicate와 `ended` publication이 같은 critical section에 있으므로 completion을 본 뒤 unlock할 때 terminal state는 이미 확정돼 있습니다.

#### death is a two-stage decision

monitor의 `find_dead_philo(table, now)`는 빠른 advisory scan입니다. candidate가 존재해도 바로 terminal을 확정하지 않습니다. `philo_try_log_death`가 authoritative commit을 수행합니다.

정확한 lock/order는 다음입니다.

```text
print_mutex lock
→ state_mutex lock
→ now = philo_now_ms()  // fresh sample
→ !ended && now - latest last_meal_ms >= time_to_die 재검사
    ├─ false: no mutation, no print
    └─ true:
         ended = 1
         timestamp 계산
         should_print = 1
→ state_mutex unlock
→ should_print이면 died printf
→ print_mutex unlock
```

fresh time, latest meal timestamp, terminal state가 같은 state critical section에서 평가됩니다. stale candidate면 `philo_try_log_death`는 false를 반환하고 monitor loop가 계속됩니다.

#### common lock order

- ordinary log: `print_mutex → state_mutex → ended check → ordinary printf`
- death commit: `print_mutex → state_mutex → fresh starvation check → ended → died printf`

두 path가 같은 nested order를 사용합니다. reverse `state_mutex → print_mutex` hold는 final logger/death transaction에 없습니다.

#### two competing orders

**normal logger가 먼저 `print_mutex`를 얻은 경우**

1. normal logger가 state를 확인합니다.
2. active라면 ordinary line을 완성합니다.
3. death path는 그 뒤 print lock을 얻어 fresh revalidation 후 died를 commit합니다.
4. ordinary line은 death보다 앞에 있으므로 terminal ordering을 위반하지 않습니다.

**death path가 먼저 `print_mutex`를 얻은 경우**

1. death path가 state lock도 얻고 `ended = 1`을 commit합니다.
2. died line을 쓰고 print lock을 해제합니다.
3. 이후 normal logger는 print lock을 얻어 state check에서 ended를 보고 suppress됩니다.

따라서 valid death가 commit된 뒤 ordinary line을 실제 출력하는 path가 없습니다.

#### at-most-one과 physical delivery 구분

`ended` check와 mutation이 state lock 아래 있으므로 valid death commit은 최대 한 번입니다. `printf` return을 검사하거나 I/O failure를 복구하지 않으므로 terminal attempt가 물리적으로 반드시 전달된다고 보장하지는 않습니다.

### 12.5 `c424b7d91ed1` — old unlock boundary를 겨냥한 deterministic test

`tests/terminal_state.c`는 monitor object의 `pthread_mutex_unlock`을 wrapper로 대체하여 이전 race window에 정확히 개입합니다.

#### completion mode

초기 state는 meal limit completion predicate가 이미 true인 상태입니다. monitor가 `state_mutex`를 해제하는 순간 wrapper가 `table->ended`를 읽습니다. assertion은 unlock 시점에 이미 true여야 한다는 것입니다. 이전 구현처럼 unlock 후 `philo_finish`를 호출하면 이 assertion이 실패합니다.

#### stale-death mode

1. initial `last_meal_ms`를 과거로 두어 monitor scan에서 candidate가 선택되게 합니다.
2. monitor가 scan 후 state mutex를 해제하는 old boundary에서 wrapper가 state를 바꿉니다.
3. `last_meal_ms`를 현재 시각으로 갱신합니다.
4. `full_count`를 completion 상태로 만듭니다.
5. monitor가 `philo_try_log_death`를 호출하면 fresh revalidation이 기아 상태를 거부해야 합니다.
6. 다음 loop에서 valid completion을 commit해야 합니다.

assertion은 run이 completion으로 끝나고 captured output에 `died`가 없다는 것입니다. 초기 candidate를 그대로 신뢰하면 stale death line이 생겨 실패합니다.

이 test는 repeated timing luck이 아니라 exact synchronization boundary를 조작합니다. 모든 monitor schedule, 전체 lock-order proof, output I/O failure를 포괄하지 않습니다.

### 12.6 항상 유지해야 하는 조건 evolution 완성

| 항상 유지해야 하는 조건 | 최초 상태 | 부족함 | 복구 | evidence |
| --- | --- | --- | --- | --- |
| terminal state access는 `state_mutex` 사용 | `033ad537d166` state API | death/output과 atomic하게 결합되지 않음 | final monitor/logger에서도 유지 | boundary wrapper와 source trace |
| ordinary output은 complete line 단위로 serialize | `033ad537d166` print lock | terminal ordering은 별도 | common nested order에 통합 | 후속 concurrency harness와 source trace |
| monitor가 global policy authority | `40ea0f871300` | candidate observation을 final truth로 사용 | advisory scan + authoritative revalidation | stale-death test |
| completion predicate와 ended는 같은 transaction | 최초 monitor는 unlock 후 finish | observation-publication gap | `a2e90b84641b` lock 안 mutation | completion mode |
| death는 current state로 commit | initial scan의 old `now`/meal | candidate stale 가능 | fresh now + latest meal recheck | stale-death mode |
| terminal 뒤 ordinary status 없음 | initial guard만 존재 | publication/print lock gap | `print → state` common order | no-death/no-post-line tests |
| at most one death commit | initial should-print guard | final ordering 불충분 | ended check/mutation under common locks | source trace + terminal tests |

### 12.7 Failure → Fix → Test 연결

#### output serialization에서 terminal serialization으로

```text
033ad537d166
complete-line print lock
+ state-protected ended
→ death는 state unlock 뒤 print lock
→ ordinary logger가 old check를 통과할 수 있음
→ a2e90b84641b
ordinary/death 모두 print_mutex → state_mutex
→ terminal mutation과 final line을 같은 serialization boundary에 배치
```

#### monitor observation에서 authoritative commit으로

```text
40ea0f871300
state lock 아래 completion/death candidate 관찰
→ unlock 후 별도 finish/death call
→ completion gap + stale candidate
→ a2e90b84641b
completion은 lock 안에서 ended commit
death는 fresh time/latest meal revalidation
→ c424b7d91ed1
old unlock boundary에 state mutation 주입
```

두 fix는 monitor를 authority에서 제거하지 않습니다. observation은 candidate selection 역할로 남고, synchronized commit point가 최종 truth가 됩니다.

### 12.8 최종 architecture

```text
workers
    ↓ state_mutex 아래 last_meal_ms / meals / full_count publish

main-thread monitor
    ↓ fresh scan time sampling
    ↓ state_mutex
    ↓ ended 또는 all-full 확인
        ├─ all-full: 같은 lock 안 ended = 1 → return
        └─ candidate 발견: pointer만 보존 → state unlock
               ↓ philo_try_log_death
               ↓ print_mutex
               ↓ state_mutex
               ↓ fresh now + latest last_meal + !ended
                   ├─ stale/terminal: no mutation, false
                   └─ valid starvation:
                        ended = 1
                        timestamp 확정
                        died line attempt
               ↓ state unlock
               ↓ print unlock

ordinary logger
    ↓ print_mutex
    ↓ state_mutex를 helper로 획득해 ended 확인
        ├─ ended: suppress
        └─ active: ordinary complete line
```

### 12.9 최종 보장과 비보장

보장하는 범위:

- workers의 meal facts와 terminal state는 defined state-lock boundary에서 관찰·변경됩니다.
- completion predicate와 `ended` publication은 같은 critical section입니다.
- death candidate는 final commit 직전 fresh monotonic time과 latest meal state로 다시 검증됩니다.
- ordinary/death logger의 common `print_mutex → state_mutex` order가 terminal decision과 output ordering을 결합합니다.
- stale candidate는 death로 commit되지 않으며, valid death는 최대 한 번 commit됩니다.
- valid terminal publication 뒤 ordinary status line은 suppress됩니다.

보장하지 않는 범위:

- initial scan은 authoritative final truth가 아닙니다.
- monitor polling은 strict detection latency를 보장하지 않습니다.
- `printf` failure나 physical output delivery를 보장하지 않습니다.
- lock order와 tests는 scheduler fairness 또는 기아 상태 freedom을 제공하지 않습니다.
===== END FILE: 04-serialized-output-to-linearized-terminal-state.md =====

===== BEGIN FILE: 05-layered-evidence-for-concurrent-behavior.md =====
# Thread: Layered evidence for concurrent behavior

이 문서는 source에 정의된 다섯 번째 Development Thread를 그대로 따릅니다. commit 순서, SHA, importance, tags는 변경하지 않습니다. 모든 test code와 workload는 해당 SHA에서 확인하며 final HEAD의 test harness를 과거 commit에 소급하지 않습니다. 이 Thread는 production mechanism을 새로 설명하는 문서가 아니라 서로 다른 failure class를 관찰하는 verification layer의 범위와 한계를 복원하는 문서입니다.

## 1. Thread 목표

이 Thread의 목표는 black-box smoke에서 output grammar, repeated concurrent schedules, focused logger race, ThreadSanitizer까지 검증이 어떻게 단계적으로 확장되는지 확인하고, 각 layer가 증명하는 것과 증명하지 않는 것을 분리하는 것입니다.

Source-confirmed significance는 다음과 같습니다.

- smoke layer는 public CLI와 termination/progress를 검사하며 timeout으로 hang을 bounded failure로 만듭니다.
- format layer는 다섯 required status phrase와 line grammar를 executable contract로 고정합니다.
- concurrency layer는 여러 philosopher 수, repeated progress/death workload, nondecreasing timestamp, terminal-line position을 검사합니다.
- focused race harness는 많은 logger와 death commit을 gate 뒤에서 겹치게 하여 terminal lock boundary를 직접 stress합니다.
- ThreadSanitizer layer는 지원되는 환경에서 exercised memory access를 관찰하고, capability probe와 skip status로 infrastructure limitation을 project race와 구분합니다.
- sanitizer workload도 semantic assertions를 유지하여 required work를 하지 않은 조기 실패가 false success가 되지 않게 합니다.
- 어떤 layer도 fairness, 기아 상태 freedom, all schedules, formal 교착 상태 freedom을 단독으로 증명하지 않습니다.

### Source에 명시적으로 연결된 Critical 항상 유지해야 하는 조건

- observable log는 required grammar를 따르고 terminal death는 최대 한 번이며 뒤에 ordinary line이 없어야 합니다.
- finite meal workloads는 intended global progress를 달성하고 death 없이 종료해야 합니다.
- elapsed timestamps는 exercised output에서 nondecreasing이어야 합니다.
- dynamic race detection 결과는 실제 required behavior assertion과 함께 해석합니다.

### Source에 명시적으로 연결된 Major Engineering Difficulties

- schedule-dependent 동작을 test하면서 repeated success를 exhaustive proof로 과장하지 않는 문제
- timeout, deterministic overlap, repeated workload, sanitizer가 서로 다른 failure class를 관찰하도록 계층화하는 문제
- unsupported ThreadSanitizer compiler/runtime를 actual race 또는 project build failure와 구분하는 문제

## 2. 이 Thread를 이해하기 위한 핵심 질문

- smoke test가 internal implementation을 몰라도 잡을 수 있는 contract failure는 무엇입니까?
- timeout은 hang과 missed terminal condition을 어떤 observable result로 바꾸는가?
- exact event order 대신 minimum work count를 사용하는 이유는 무엇입니까?
- log grammar validator는 expected substring test보다 어떤 corrupted output을 더 잡는가?
- syntax 검증과 event-order 검증을 왜 분리하는가?
- repeated workload는 어떤 schedule-sensitive symptom을 노출하며 왜 proof가 아닌가?
- one-death/no-line-after-death assertion은 어떤 production terminal 항상 유지해야 하는 조건과 연결됩니까?
- gated 12-logger harness는 일반 repeated run보다 어떤 boundary overlap을 의도적으로 높이는가?
- ThreadSanitizer capability probe는 compiler unsupported, runtime unsupported, production failure를 어떻게 구분하는가?
- skip status 77은 무엇을 의미하며 무엇을 의미하지 않는가?
- sanitizer diagnostic 부재와 semantic progress assertion을 함께 유지해야 하는 이유는 무엇입니까?
- deterministic failure injection, repeated behavior test, dynamic race detector의 역할은 어떻게 다르고 보완적입니까?

## 3. 완료 기준

- [x] smoke suite의 CLI, one-philosopher, finite completion, timeout contract를 실제 command/assertion으로 설명할 수 있습니다.
- [x] minimum count와 exact schedule assertion의 차이를 설명할 수 있습니다.
- [x] `awk` grammar가 허용·거부하는 line을 실제 condition으로 설명할 수 있습니다.
- [x] finite/repeated/death workload matrix와 각 observable 항상 유지해야 하는 조건을 표로 정리했습니다.
- [x] nondecreasing timestamp와 terminal-line-position check를 구현 수준에서 설명할 수 있습니다.
- [x] 12-logger gate와 death path가 실제 production logger/terminal path를 통과함을 확인했습니다.
- [x] TSAN probe, optional skip, required failure, production instrumentation을 구분할 수 있습니다.
- [x] sanitizer options와 semantic assertions를 함께 설명할 수 있습니다.
- [x] 각 layer의 증명 범위와 비증명 범위를 작성했습니다.
- [x] test 통과를 fairness, 기아 상태 freedom, all-schedule race freedom으로 확대하지 않았습니다.

## 4. Commit map

| 순서 | Commit | Subject | Importance | Tags | Source-defined role |
| --- | --- | --- | --- | --- | --- |
| 1 | `bd6bb8eb18f4` | `test(smoke): 주요 입력과 종료 조건 검증` | B | `TEST, CLI_CONTRACT, CORE` | Adds bounded public smoke cases for input, death, and finite completion. |
| 2 | `f145d33f2773` | `test(format): 필수 상태 로그 형식 검증` | B | `TEST, TERMINAL_STATE` | Treats the five-line status grammar as an executable output contract. |
| 3 | `3d24bea01441` | `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증` | A | `TEST, CONCURRENCY, TERMINAL_STATE` | Repeats progress and death schedules and adds a gated logger-versus-death race harness. |
| 4 | `20f8270c78bb` | `test(tsan): ThreadSanitizer 검증 경로 추가` | A | `TEST, CONCURRENCY, PRACTICAL` | Adds capability-probed ThreadSanitizer workloads while retaining semantic log and progress assertions. |

## 5. Commit별 학습 기록

### 5.1 `bd6bb8eb18f4` — `test(smoke): 주요 입력과 종료 조건 검증`

- Importance: **B**
- Tags: `TEST, CLI_CONTRACT, CORE`
- Source-defined role: Adds bounded public smoke cases for input, death, and finite completion.
- 코드 기준: 반드시 `bd6bb8eb18f4` 시점
- 직접 parent 비교: `git diff bd6bb8eb18f4^ bd6bb8eb18f4 --`
- Thread 직전 관련 SHA: Thread 내 첫 commit

#### Source-confirmed test 역할

이 B-level smoke suite는 `make test`에서 executable의 public CLI와 output을 black-box로 검사합니다. temporary output은 trap으로 정리하고, potentially blocking simulation마다 separate timeout process를 사용해 교착 상태 또는 missed termination을 bounded failure로 바꿉니다.

cases는 invalid philosopher count, numeric overflow, one-philosopher fork acquisition과 death, finite meal schedules의 no-death completion을 포함합니다. meal assertion은 scheduler-dependent exact event order가 아니라 required global work threshold에 도달했음을 보이는 minimum count를 사용합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 public contract | §12 완료 기록의 대응 근거 참조 |
| invalid-input cases | §12 완료 기록의 대응 근거 참조 |
| overflow case | §12 완료 기록의 대응 근거 참조 |
| single-philosopher case | §12 완료 기록의 대응 근거 참조 |
| finite completion cases | §12 완료 기록의 대응 근거 참조 |
| timeout technique와 종료 기준 | §12 완료 기록의 대응 근거 참조 |
| temporary output lifecycle | §12 완료 기록의 대응 근거 참조 |
| 실제로 실행하는 production path | §12 완료 기록의 대응 근거 참조 |
| minimum-count assertion 이유 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| broad black-box integration 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `make test`가 어떤 script 또는 target을 실행하는지 Makefile과 test file에서 확인합니다.
- [x] temporary file/directory 생성과 trap cleanup을 확인합니다.
- [x] 각 potentially blocking run에 timeout이 별도 process로 적용되는지 확인합니다.
- [x] invalid philosopher count와 overflow input의 exact command, 종료 상태, stderr assertion을 기록합니다.
- [x] single-philosopher run에서 fork event와 `died`를 각각 어떻게 검사하는지 확인합니다.
- [x] finite meal schedules의 philosopher count, timing, meal target을 기록합니다.
- [x] no-death assertion과 per-run completion 판단을 확인합니다.
- [x] meal log를 exact count가 아니라 minimum count로 검사하는 이유를 test implementation과 scheduler variability로 설명합니다.
- [x] timeout expiration을 ordinary nonzero exit와 어떻게 구분하는지 확인합니다.
- [x] test가 internal lock order나 race detector를 사용하지 않는 black-box boundary임을 확인합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 주요 invalid CLI, overflow, one-philosopher death, finite no-death completion을 executable public interface에서 반복 가능한 기준 상태로 검증합니다.
- hang 또는 missed terminal condition을 timeout으로 bounded test failure로 바꿉니다.
- scheduler-dependent exact ordering을 요구하지 않으면서 intended global work threshold를 검사합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- race freedom, all possible schedules, fairness, 교착 상태 freedom 전체를 증명하지 않습니다.
- internal lock order나 exact state linearization을 직접 검증하지 않습니다.
- log grammar 전체와 terminal-line ordering은 후속 layers에서 강화됩니다.

#### 학습자 결론
- [x] black-box smoke가 production internals를 몰라도 잡을 수 있는 failure 종류를 설명합니다.
- [x] timeout이 hang을 관찰 가능한 test result로 바꾸는 방식을 설명합니다.
- [x] minimum meal-count assertion이 exact interleaving assertion보다 적절한 이유와 남는 한계를 설명합니다.

### 5.2 `f145d33f2773` — `test(format): 필수 상태 로그 형식 검증`

- Importance: **B**
- Tags: `TEST, TERMINAL_STATE`
- Source-defined role: Treats the five-line status grammar as an executable output contract.
- 코드 기준: 반드시 `f145d33f2773` 시점
- 직접 parent 비교: `git diff f145d33f2773^ f145d33f2773 --`
- Thread 직전 관련 SHA: `bd6bb8eb18f4`

#### Source-confirmed test 역할

이 B-level commit은 textual log grammar를 executable contract로 만듭니다. `awk` validator는 각 line이 numeric timestamp, positive philosopher identifier, 다섯 required status phrase 중 하나만 포함하는지 검사합니다. single-philosopher, finite-meal, larger no-death run에 적용되어 expected substring이 존재하더라도 malformed 또는 interleaved output이 함께 있으면 실패합니다.

validator는 valid scheduler ordering을 하나로 고정하지 않고 syntax만 검사합니다. nondecreasing timestamp와 terminal-line position은 후속 concurrency suite가 담당합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| numeric timestamp grammar | §12 완료 기록의 대응 근거 참조 |
| positive philosopher id grammar | §12 완료 기록의 대응 근거 참조 |
| 허용하는 다섯 status phrase | §12 완료 기록의 대응 근거 참조 |
| reject하는 extra/malformed line | §12 완료 기록의 대응 근거 참조 |
| 적용 workload | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production logging path | §12 완료 기록의 대응 근거 참조 |
| syntax-only 설계 이유 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| black-box grammar regression 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `awk` validator의 field count, numeric pattern, id positivity, allowed phrase 조건을 실제 code로 분해합니다.
- [x] 다섯 status phrase를 test source에서 정확히 기록합니다.
- [x] 빈 line, extra token, unknown phrase, nonnumeric timestamp가 각각 어떤 condition으로 reject되는지 확인합니다.
- [x] validator 종료 상태가 shell suite의 failure로 전파되는 방식을 확인합니다.
- [x] single, finite-meal, larger no-death workload output 각각에 validator가 적용되는 위치를 확인합니다.
- [x] expected substring 검사가 통과해도 malformed extra line이 있으면 validator가 실패하는 이유를 확인합니다.
- [x] event order를 검사하지 않는다는 것을 assertion 부재로 확인합니다.
- [x] line interleaving이 grammar를 깨뜨릴 때 이 validator가 어떤 형태로 감지하는지 예시를 학습자가 직접 작성합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 모든 관찰된 line이 required status grammar에 맞는지 검증합니다.
- expected event substring과 함께 malformed, extra, corrupted line이 섞이는 회귀를 잡습니다.
- valid thread schedule의 event order를 불필요하게 고정하지 않습니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- timestamp가 nondecreasing인지 검증하지 않습니다.
- `died`가 유일하고 마지막인지 검증하지 않습니다.
- semantic progress, race freedom, all schedules를 증명하지 않습니다.

#### 학습자 결론
- [x] content assertion과 grammar assertion이 잡는 failure가 어떻게 다른지 설명합니다.
- [x] validator의 정확한 accepted language를 field 수준으로 설명합니다.
- [x] syntax를 검사하면서 valid interleaving order를 고정하지 않는 이유를 설명합니다.

### 5.3 `3d24bea01441` — `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증`

- Importance: **A**
- Tags: `TEST, CONCURRENCY, TERMINAL_STATE`
- Source-defined role: Repeats progress and death schedules and adds a gated logger-versus-death race harness.
- 코드 기준: 반드시 `3d24bea01441` 시점
- 직접 parent 비교: `git diff 3d24bea01441^ 3d24bea01441 --`
- Thread 직전 관련 SHA: `f145d33f2773`

#### Source-confirmed test 역할

이 A-level concurrency suite는 public schedule stress와 focused race harness를 결합합니다. finite runs는 2, 5, 17 philosophers가 death 없이 meal target을 모두 달성해야 하며, 7-philosopher workload를 여러 번 반복해 schedule-sensitive progress failure를 노출합니다. death workloads도 반복하며 정확히 하나의 `died` line과 그 뒤 line 부재를 요구합니다. 모든 log는 grammar와 nondecreasing timestamp를 검사합니다.

별도 harness는 12 logger thread를 gate 뒤에서 동시에 출발시켜 각자 수백 개 normal status write를 시도하는 동안 다른 path가 같은 table에 death를 commit합니다. stream은 여전히 terminal death 하나와 post-terminal status 부재를 만족해야 합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| finite workload matrix | §12 완료 기록의 대응 근거 참조 |
| repeated seven-philosopher workload | §12 완료 기록의 대응 근거 참조 |
| death workload matrix와 반복 | §12 완료 기록의 대응 근거 참조 |
| per-philosopher progress assertion | §12 완료 기록의 대응 근거 참조 |
| grammar assertion | §12 완료 기록의 대응 근거 참조 |
| nondecreasing timestamp assertion | §12 완료 기록의 대응 근거 참조 |
| unique terminal death assertion | §12 완료 기록의 대응 근거 참조 |
| no-line-after-death assertion | §12 완료 기록의 대응 근거 참조 |
| 12-logger gated race harness | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production logger/death path | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| repeated integration vs focused contention harness 구분 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] finite runs의 2, 5, 17 philosopher command와 meal target을 확인합니다.
- [x] 각 philosopher가 target을 달성했는지 output을 어떻게 group/count하는지 확인합니다.
- [x] 7-philosopher workload의 반복 횟수와 failure aggregation 방식을 확인합니다.
- [x] death workload의 configuration, 반복 횟수, exact-one-death assertion을 확인합니다.
- [x] `died` line index 이후 다른 line이 없는지 어떤 shell/awk logic으로 검사하는지 확인합니다.
- [x] timestamp sequence를 numeric nondecreasing으로 검사하는 implementation을 확인합니다.
- [x] focused harness에서 12 logger thread를 생성하고 gate로 overlap을 높이는 순서를 확인합니다.
- [x] 각 logger가 시도하는 normal status 횟수와 production logging symbol을 확인합니다.
- [x] death commit path가 actual production `philo_try_log_death` 또는 해당 SHA의 symbol을 통과하는지 확인합니다.
- [x] harness output capture가 unique death와 post-terminal suppression을 어떻게 검사하는지 확인합니다.
- [x] repeated executable workloads와 in-process focused harness가 서로 다른 failure class를 겨냥하는 이유를 기록합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Evidence matrix

| Evidence path | Workload 또는 injection | 관찰 항상 유지해야 하는 조건 | 강점 | 남는 한계 |
| --- | --- | --- | --- | --- |
| finite executable runs | 2, 5, 17 philosophers | all reach target, no death | broad integration | schedule exhaustive 아님 |
| repeated progress run | 7 philosophers 반복 | schedule-sensitive progress symptom | 반복으로 exposure 증가 | fairness proof 아님 |
| repeated death runs | forced death | one `died`, no later line | terminal behavior | 모든 interleaving 아님 |
| gated logger harness | 12 logger threads + death | lock-order/terminal suppression | focused overlap | static proof 아님 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 여러 finite workload에서 모든 philosopher의 observable target progress와 no-death completion을 반복 검증합니다.
- death workload에서 exactly one terminal death와 이후 line 부재를 검증합니다.
- grammar와 nondecreasing monotonic timestamp를 함께 검사합니다.
- focused logger-versus-death overlap으로 terminal lock-order decision을 직접 stress합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- 모든 scheduler interleaving, 기아 상태 freedom, fairness를 증명하지 않습니다.
- 반복 통과가 formal 교착 상태-freedom proof는 아닙니다.
- dynamic data race detection은 다음 TSAN layer가 보완하지만 그것도 exhaustive하지 않습니다.

#### 학습자 결론
- [x] broad repeated workloads와 focused race harness의 목적을 분리해 설명합니다.
- [x] one-death/no-post-line assertion이 terminal linearization과 어떻게 연결되는지 설명합니다.
- [x] nondecreasing timestamp가 monotonic time과 serialized output의 어떤 observable 결과를 검사하는지 설명합니다.
- [x] 반복 횟수가 confidence를 높여도 proof가 되지 않는 이유를 설명합니다.

### 5.4 `20f8270c78bb` — `test(tsan): ThreadSanitizer 검증 경로 추가`

- Importance: **A**
- Tags: `TEST, CONCURRENCY, PRACTICAL`
- Source-defined role: Adds capability-probed ThreadSanitizer workloads while retaining semantic log and progress assertions.
- 코드 기준: 반드시 `20f8270c78bb` 시점
- 직접 parent 비교: `git diff 20f8270c78bb^ 20f8270c78bb --`
- Thread 직전 관련 SHA: `3d24bea01441`

#### Source-confirmed test 역할

이 A-level commit은 optional ThreadSanitizer build/workload path를 `make test-tsan`으로 추가합니다. compiler는 configurable하며 `TSAN_REQUIRED`가 optional local capability와 mandatory environment를 구분합니다.

script는 먼저 작은 pthread probe를 instrumented build/run합니다. compiler가 `-fsanitize=thread`를 지원하지 않거나 runtime이 instrumented binary를 실행하지 못하면, required가 아닌 경우 documented skip status 77로 끝납니다. probe가 성공하면 complete executable을 thread instrumentation과 debug information으로 rebuild하고 finite, forced-death, contention workloads를 `halt_on_error`와 dedicated error exit code로 실행합니다. sanitizer diagnostic 부재뿐 아니라 log grammar, progress, no-death, one-terminal-death semantic assertion도 다시 수행합니다.


#### Test commit 분석

| 구분 | 학습자 기록 |
| --- | --- |
| 대상 production 항상 유지해야 하는 조건 | §12 완료 기록의 대응 근거 참조 |
| compiler configuration | §12 완료 기록의 대응 근거 참조 |
| `TSAN_REQUIRED` 의미 | §12 완료 기록의 대응 근거 참조 |
| pthread capability probe | §12 완료 기록의 대응 근거 참조 |
| compile failure 처리 | §12 완료 기록의 대응 근거 참조 |
| runtime probe failure 처리 | §12 완료 기록의 대응 근거 참조 |
| skip status 77 조건 | §12 완료 기록의 대응 근거 참조 |
| instrumented production build flags | §12 완료 기록의 대응 근거 참조 |
| TSAN runtime options와 dedicated exit code | §12 완료 기록의 대응 근거 참조 |
| finite/death/contention workloads | §12 완료 기록의 대응 근거 참조 |
| sanitizer diagnostic assertion | §12 완료 기록의 대응 근거 참조 |
| semantic behavior assertions | §12 완료 기록의 대응 근거 참조 |
| 실제로 통과하는 production paths | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하는 것 | §12 완료 기록의 대응 근거 참조 |
| 이 테스트가 증명하지 않는 것 | §12 완료 기록의 대응 근거 참조 |
| dynamic schedule-dependent verification 분류 | §12 완료 기록의 대응 근거 참조 |
| 후속 회귀 방지 대상 | §12 완료 기록의 대응 근거 참조 |

#### 해당 SHA에서 직접 확인할 코드
- [x] `make test-tsan` target에서 compiler와 flags가 어떻게 전달되는지 확인합니다.
- [x] `TSAN_REQUIRED`의 default와 required mode 분기를 확인합니다.
- [x] 작은 pthread probe source, compile command, run command를 확인합니다.
- [x] compiler unsupported와 runtime unsupported를 project build/test failure와 구분하는 조건을 확인합니다.
- [x] optional mode에서 skip message와 종료 상태 77이 나오는 모든 path를 기록합니다.
- [x] required mode에서는 같은 capability failure가 어떤 non-skip failure로 전환되는지 확인합니다.
- [x] complete executable rebuild에 `-fsanitize=thread`와 debug information이 compile/link 모두 적용되는지 확인합니다.
- [x] `TSAN_OPTIONS`의 `halt_on_error`와 dedicated `exitcode` 값을 확인합니다.
- [x] finite, forced-death, higher-contention workload command를 기록합니다.
- [x] sanitizer output/exit status뿐 아니라 grammar, per-philosopher progress, no-death, one-terminal-death assertion을 다시 실행하는 위치를 확인합니다.
- [x] instrumented run이 required work를 하지 않고 조기 종료해도 통과하지 못하도록 semantic assertions가 어떻게 보완하는지 설명합니다.

#### 코드 근거 기록

| 항목 | 학습자 기록 |
| --- | --- |
| 파일과 symbol | §12 완료 기록의 대응 근거 참조 |
| 최소 코드 구간 | §12 완료 기록의 대응 근거 참조 |
| caller → callee | §12 완료 기록의 대응 근거 참조 |
| state 또는 소유권 변화 | §12 완료 기록의 대응 근거 참조 |
| failure/cleanup 경로 | §12 완료 기록의 대응 근거 참조 |
| 직전 상태와의 차이 | §12 완료 기록의 대응 근거 참조 |

#### Capability와 correctness 결과 구분

| 상황 | Optional mode 결과 | Required mode 결과 | project race로 해석하는가 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| compiler가 TSAN flag 미지원 | skip 77 | §12 완료 기록의 대응 근거 참조 | 아니오 | §12 완료 기록의 대응 근거 참조 |
| instrumented pthread probe runtime 실패 | skip 77 | §12 완료 기록의 대응 근거 참조 | 아니오 | §12 완료 기록의 대응 근거 참조 |
| production instrumented build 실패 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| TSAN diagnostic 발생 | failure | failure | exercised race evidence | §12 완료 기록의 대응 근거 참조 |
| semantic assertion 실패 | failure | failure | behavioral failure | §12 완료 기록의 대응 근거 참조 |
| diagnostics 없이 모든 workload/assertion 통과 | success | success | exercised schedules에서 evidence | §12 완료 기록의 대응 근거 참조 |


#### 보장 범위

이 commit이 source 기준으로 보장하는 것:
- 지원되는 환경에서 instrumented production workloads의 exercised memory access에 대해 ThreadSanitizer 검사를 수행합니다.
- sanitizer support 부재를 project race 또는 build defect와 혼동하지 않도록 capability probe와 skip semantics를 둡니다.
- instrumented run도 required observable work와 terminal 항상 유지해야 하는 조건을 수행해야 통과합니다.

이 commit 시점에 아직 보장하지 않거나 검증 범위를 넘는 것:
- ThreadSanitizer 통과가 모든 schedule의 race freedom을 증명하지 않습니다.
- 교착 상태 freedom, fairness, 기아 상태 freedom, strict timing을 증명하지 않습니다.
- unsupported environment의 skip은 project correctness success를 의미하지 않습니다.

#### 학습자 결론
- [x] capability probe가 project test result의 해석을 정직하게 만드는 이유를 설명합니다.
- [x] optional skip과 mandatory failure의 차이를 실제 branch/status로 설명합니다.
- [x] sanitizer diagnostic 부재와 semantic work completion을 함께 검사해야 하는 이유를 설명합니다.
- [x] deterministic boundary tests, repeated workload, TSAN이 서로 대체되지 않고 보완하는 이유를 설명합니다.

## 6. 항상 유지해야 하는 조건 ledger

이 Thread에서는 production 항상 유지해야 하는 조건 자체보다 그것을 관찰하는 evidence layer의 도입 순서를 기록합니다.

| Production 항상 유지해야 하는 조건 또는 risk | 최초 evidence | 강화된 evidence | 동적 evidence | 실제 test code 근거 | 최종 해석 |
| --- | --- | --- | --- | --- | --- |
| invalid input과 overflow가 public boundary에서 거부됨 | `bd6bb8eb18f4` | 기존 smoke 유지 | TSAN 범위의 핵심 대상은 아님 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| one-philosopher run이 hang하지 않고 fork/death behavior를 보임 | `bd6bb8eb18f4` timeout + content | format validator 적용 | instrumented death workload 일부와 구분 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| finite meal run이 death 없이 intended work threshold에 도달 | `bd6bb8eb18f4` | `3d24bea01441`의 2/5/17 및 반복 workload | `20f8270c78bb` instrumented finite workload | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| 모든 line이 required grammar를 따름 | expected substring baseline | `f145d33f2773` | concurrency/TSAN workloads에서도 재검사 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| timestamp가 nondecreasing | 초기 smoke에는 없음 | `3d24bea01441` | TSAN semantic assertions에서 유지 여부 확인 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| death가 하나이며 이후 line이 없음 | smoke의 single death content | `3d24bea01441` repeated death + focused harness | `20f8270c78bb` instrumented death/contention | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| exercised shared access에 data-race diagnostic이 없음 | behavioral tests만 존재 | focused contention은 symptom/contract stress | `20f8270c78bb` ThreadSanitizer | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |
| unsupported sanitizer infrastructure와 project failure를 구분 | 없음 | 없음 | `20f8270c78bb` probe + skip 77 | §12 완료 기록의 대응 근거 참조 | §12 완료 기록의 대응 근거 참조 |

## 7. Failure → Fix → Test 연결

이 Development Thread의 commit map에는 production fix commit이 없습니다. 따라서 이 영역은 source가 확정한 verification progression을 `위험 또는 관찰 한계 → 다음 evidence layer`로 연결합니다. 다른 Thread의 commit을 이 Thread map에 추가하지 않습니다.

### 7.1 Hang와 public contract failure

```text
potential deadlock / missed terminal / invalid CLI
→ `bd6bb8eb18f4`
public commands + timeout + exit/output assertions
```

- bounded failure로 바꾸는 timeout: §12 완료 기록의 대응 근거에 정리했습니다.
- public path: §12 완료 기록의 대응 근거에 정리했습니다.
- observable success/failure: §12 완료 기록의 대응 근거에 정리했습니다.
- internal root cause를 직접 증명하지 못하는 범위: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.2 Expected substring의 한계

```text
expected event가 존재해도 malformed/extra/interleaved line 가능
→ `f145d33f2773`
strict line grammar validator
```

- smoke content assertion: §12 완료 기록의 대응 근거에 정리했습니다.
- grammar validator: §12 완료 기록의 대응 근거에 정리했습니다.
- malformed example: §12 완료 기록의 대응 근거에 정리했습니다.
- event order를 의도적으로 고정하지 않는 범위: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.3 Schedule-sensitive terminal behavior

```text
한 번의 정상 run은 schedule-sensitive regression을 놓칠 수 있음
→ `3d24bea01441`
repeated progress/death workloads
+ nondecreasing timestamps
+ gated 12-logger-versus-death harness
```

- repeated matrix: §12 완료 기록의 대응 근거에 정리했습니다.
- focused overlap: §12 완료 기록의 대응 근거에 정리했습니다.
- terminal assertions: §12 완료 기록의 대응 근거에 정리했습니다.
- proof가 아닌 이유: §12 완료 기록의 대응 근거에 정리했습니다.

### 7.4 Behavioral evidence와 dynamic race detection

```text
observable behavior가 정상이어도 exercised data race가 있을 수 있음
→ `20f8270c78bb`
capability-probed ThreadSanitizer
+ semantic assertions 유지
```

- capability probe: §12 완료 기록의 대응 근거에 정리했습니다.
- optional skip: §12 완료 기록의 대응 근거에 정리했습니다.
- required mode: §12 완료 기록의 대응 근거에 정리했습니다.
- diagnostic failure: §12 완료 기록의 대응 근거에 정리했습니다.
- semantic failure: §12 완료 기록의 대응 근거에 정리했습니다.
- TSAN 통과가 증명하지 않는 범위: §12 완료 기록의 대응 근거에 정리했습니다.

## 8. Verification responsibility 변화

| 시점 | Test boundary | Failure를 만드는 방식 | 관찰 대상 | 한계 관리 | 학습자 근거 |
| --- | --- | --- | --- | --- | --- |
| `bd6bb8eb18f4` | executable public interface | input/workload + timeout | exit, output content, bounded termination | black-box/non-exhaustive 명시 | §12 완료 기록의 대응 근거 참조 |
| `f145d33f2773` | captured output language | malformed line을 validator가 reject | line grammar | order를 고정하지 않음 | §12 완료 기록의 대응 근거 참조 |
| `3d24bea01441` | repeated executable + in-process contention | 반복 schedule + gate overlap | progress, timestamps, terminal position | fairness/proof claim 배제 | §12 완료 기록의 대응 근거 참조 |
| `20f8270c78bb` | instrumented executable/runtime | TSAN instrumentation | exercised memory access + behavior | capability probe, skip semantics, schedule dependence | §12 완료 기록의 대응 근거 참조 |

## 9. Thread 최종 상태

### Source-confirmed 최종 상태

- public smoke, grammar validation, repeated schedule stress, focused terminal contention, ThreadSanitizer가 서로 다른 evidence layer를 이룹니다.
- timeout은 hangs를 bounded failures로 만들고 grammar validator는 expected substring만으로 놓치는 malformed output을 잡습니다.
- repeated workloads와 logger gate는 schedule-sensitive observable 항상 유지해야 하는 조건을 더 강하게 자극합니다.
- supported environment의 TSAN은 exercised memory access를 관찰하며 semantic progress와 terminal assertions를 함께 요구합니다.
- unsupported sanitizer infrastructure는 skip status로 project race와 분리됩니다.
- 이 verification stack은 fairness, 기아 상태 freedom, 교착 상태 freedom의 formal proof 또는 all-schedule race freedom을 제공하지 않습니다.

### 학습자가 작성할 최종 설명

- public smoke layer: §12 완료 기록의 대응 근거에 정리했습니다.
- grammar layer: §12 완료 기록의 대응 근거에 정리했습니다.
- repeated schedule layer: §12 완료 기록의 대응 근거에 정리했습니다.
- focused contention layer: §12 완료 기록의 대응 근거에 정리했습니다.
- dynamic race layer: §12 완료 기록의 대응 근거에 정리했습니다.
- capability/skip semantics: §12 완료 기록의 대응 근거에 정리했습니다.
- semantic assertions: §12 완료 기록의 대응 근거에 정리했습니다.
- 각 layer가 잡는 failure: §12 완료 기록의 대응 근거에 정리했습니다.
- 각 layer가 놓치는 failure: §12 완료 기록의 대응 근거에 정리했습니다.
- 전체 evidence를 해석하는 기준: §12 완료 기록의 대응 근거에 정리했습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
make test
    ↓ public CLI and workload execution
    ↓ timeout으로 hangs bounded
    ↓ content assertions
    ↓ required line grammar
    ↓ repeated finite/death schedules
    ↓ per-philosopher progress
    ↓ nondecreasing timestamps
    ↓ one terminal death / no later line
    ↓ focused 12-logger-versus-death overlap

make test-tsan
    ↓ configurable compiler
    ↓ instrumented pthread capability probe
        ├─ unsupported + optional
        │      → documented skip 77
        ├─ unsupported + required
        │      → failure
        └─ supported
               ↓ full instrumented build
               ↓ finite/death/contention workloads
               ↓ TSAN diagnostic check
               ↓ grammar/progress/terminal semantic checks
```

- 실제 Make targets: §12 완료 기록의 대응 근거에 정리했습니다.
- smoke script: §12 완료 기록의 대응 근거에 정리했습니다.
- timeout command: §12 완료 기록의 대응 근거에 정리했습니다.
- grammar function: §12 완료 기록의 대응 근거에 정리했습니다.
- workload loop: §12 완료 기록의 대응 근거에 정리했습니다.
- timestamp validator: §12 완료 기록의 대응 근거에 정리했습니다.
- terminal-position validator: §12 완료 기록의 대응 근거에 정리했습니다.
- logger harness entry: §12 완료 기록의 대응 근거에 정리했습니다.
- TSAN probe compile/run: §12 완료 기록의 대응 근거에 정리했습니다.
- skip branch: §12 완료 기록의 대응 근거에 정리했습니다.
- required branch: §12 완료 기록의 대응 근거에 정리했습니다.
- instrumentation flags/options: §12 완료 기록의 대응 근거에 정리했습니다.
- semantic assertion reuse: §12 완료 기록의 대응 근거에 정리했습니다.

## 11. 학습 완료 자가 점검

- [x] `bd6bb8eb18f4`의 commands, timeout, trap, positive/negative assertions를 확인했습니다.
- [x] smoke suite가 race freedom을 증명하지 않는다고 명시했습니다.
- [x] `f145d33f2773`의 accepted line grammar를 정확히 설명했습니다.
- [x] syntax와 event order를 구분했습니다.
- [x] `3d24bea01441`의 2/5/17, repeated 7, death workload와 logger harness를 분리해 기록했습니다.
- [x] per-philosopher progress, nondecreasing timestamp, unique terminal death, no-later-line assertion을 확인했습니다.
- [x] 12 logger threads가 actual production logging path를 통과하는지 확인했습니다.
- [x] `20f8270c78bb`의 compiler/runtime probe와 skip 77을 설명했습니다.
- [x] optional environment limitation과 project race/build failure를 구분했습니다.
- [x] instrumented run의 semantic assertions가 required work를 재검사하는 이유를 설명했습니다.
- [x] repeated success나 TSAN success를 fairness, 교착 상태 freedom, all-schedule proof로 확대하지 않았습니다.

## 12. 저장소 기반 완료 기록

### 12.1 검토 범위와 실행 상태

- 이 Thread의 4개 SHA는 모두 `c/philo` HEAD의 조상으로 확인했습니다.
- `Makefile`, `tests/smoke.sh`, `tests/concurrency.sh`, `tests/log_terminal_race.c`, `tests/tsan.sh`를 각각 도입 SHA에서 확인했습니다.
- 저장소 checkout이 불가능해 `make test`와 `make test-tsan`은 실행하지 않았습니다. 따라서 아래에는 script가 수행하도록 작성된 명령·assertion과 그 증명 범위만 기록합니다.

### 12.2 `bd6bb8eb18f4` — public black-box smoke

`Makefile`의 `test` target은 project를 build한 뒤 `tests/smoke.sh`를 실행합니다. script는 `mktemp -d`로 output directory를 만들고 `trap cleanup EXIT INT TERM`으로 정리합니다.

#### timeout mechanism

`run_timeout`은 target command를 background로 실행하고 별도 guard process가 limit 뒤 `SIGTERM`을 보냅니다. command가 먼저 끝나면 guard를 종료합니다. 교착 상태, single-philosopher self-block, missed completion을 무한 test hang이 아니라 bounded failure로 바꿉니다.

이 timeout은 내부 root cause를 식별하지 않습니다. 단지 public command가 제한 안에 종료하지 못했다는 observable failure를 만듭니다.

#### workload와 assertion

| case | command | assertion | 관찰 범위 |
| --- | --- | --- | --- |
| invalid count | `./philo 0 100 10 10` | success이면 실패, usage text 필요 | CLI validation |
| numeric overflow | 매우 긴 `time_to_die` argument | success이면 실패 | parser overflow rejection |
| one philosopher | `./philo 1 80 40 40` | 2초 안 clean exit, fork line, death line | edge topology와 terminal behavior |
| finite two | `./philo 2 250 50 50 2` | 3초 안 exit, no death, `is eating` 최소 4개 | global target work |
| finite five | `./philo 5 800 100 100 3` | 5초 안 exit, no death, `is eating` 최소 15개 | larger no-death completion |

meal log 수는 exact count가 아니라 lower bound입니다. final target 이후 terminal publication 직전 추가 attempt가 보일 수 있고 scheduler ordering은 고정되지 않으므로, required global work를 했는지만 검사합니다. 반대로 최소 count는 philosopher별 공정한 progress를 보장하지 않으며 후속 concurrency suite가 per-id count를 강화합니다.

#### 증명 범위

smoke는 public executable의 argument handling, bounded termination, basic single/final-meal 동작을 폭넓게 잡습니다. internal lock order, data race, 모든 schedule, fairness는 검사하지 않습니다.

### 12.3 `f145d33f2773` — output language를 executable contract로 고정

`tests/smoke.sh::check_log_format`은 각 line 전체가 다음 regex와 일치해야 통과합니다.

```text
^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$
```

#### accepted grammar

- field 1: 0 이상의 decimal timestamp
- field 2: leading zero 없는 positive philosopher id
- remainder: 다섯 phrase 중 정확히 하나
- line 앞뒤에 extra token이나 임의 text 없음

#### rejected examples

```text
abc 1 is eating          # timestamp가 숫자가 아님
10 0 is eating           # id가 positive가 아님
10 1 is running          # unknown status
10 1 is eating extra     # extra token
10 1 is eat10 2 died     # interleaved/corrupted line
```

validator는 every line에 적용되므로 required substring이 존재해도 malformed extra line이 하나 있으면 실패합니다. single, finite, larger no-death workload output에 적용됩니다.

이 commit은 syntax만 검증합니다. timestamp ordering, `died`의 유일성·마지막 위치, philosopher별 progress는 의도적으로 후속 layer에 남깁니다. valid scheduler event order를 하나로 강제하지 않습니다.

### 12.4 `3d24bea01441` — repeated schedules와 focused terminal contention

`tests/concurrency.sh`는 `check_log`, `check_progress`, `check_terminal_line`을 추가합니다.

#### `check_log`

- 모든 line이 required grammar와 일치해야 합니다.
- 이전 timestamp를 기억하고 현재 `$1 < previous`이면 실패합니다.
- 따라서 captured output의 timestamp는 numeric nondecreasing이어야 합니다.

이 check는 monotonic clock과 serialized line order가 외부에 만든 결과를 검사합니다. 같은 millisecond timestamp는 허용합니다.

#### `check_progress`

`is eating` line을 philosopher id별로 count하고, `1..N` 모두 target 이상인지 확인합니다. aggregate lower bound보다 강하며 한 philosopher가 모든 work를 독점한 output을 통과시키지 않습니다. 다만 `is eating`은 operation start이므로 internal committed counter와 완전히 동일한 evidence는 아닙니다. finite no-death workload에서 intended progress symptom을 검사합니다.

#### `check_terminal_line`

line을 순서대로 읽으면서 `died`를 본 뒤 다음 line이 하나라도 있으면 `after = 1`로 실패하고, death count가 정확히 1인지 확인합니다. terminal-state linearization의 observable contract를 겨냥합니다.

#### workload matrix

| workload | 반복 | timeout | assertion |
| --- | --- | --- | --- |
| `N = 2, 5, 17`, `2000 5 5 4` | 각 1회 | 6초 | grammar/monotonic log, no death, 각 id 4 meals 이상 |
| `N = 7`, `1000 4 4 3` | 8회 | 4초 | no death, 각 id 3 meals 이상 |
| `N = 5`, `60 80 10` | 10회 | 3초 | exactly one death, no later line |

반복은 schedule-sensitive regression 노출 확률을 높입니다. 8회 또는 10회 성공이 모든 scheduler interleaving의 proof는 아닙니다.

#### focused `log_terminal_race` harness

`tests/log_terminal_race.c`는 다음 상수를 사용합니다.

```text
LOGGER_COUNT = 12
LOGS_PER_LOGGER = 200
```

각 logger thread는 test gate에서 모두 ready가 될 때까지 기다린 뒤 동시에 release됩니다. 각자 actual production `philo_log(philo, "is thinking")`을 최대 200회 시도합니다.

main thread는 같은 table을 death-eligible 상태로 seed합니다.

- `time_to_die = 1`
- `start_ms = now - 100`
- `last_meal_ms = start_ms`

loggers를 release한 직후 production `philo_try_log_death`를 호출합니다. 따라서 ordinary logger의 `print_mutex → state_mutex` path와 death commit의 같은 lock order가 강하게 겹칩니다. output은 여전히 grammar, exactly one death, no line after death를 만족해야 합니다.

이 harness는 target boundary의 overlap을 의도적으로 높이지만 static lock-order proof나 모든 interleaving 검증은 아닙니다.

### 12.5 `20f8270c78bb` — capability-probed ThreadSanitizer

#### 진입점과 configuration

- Make target: `make test-tsan`
- compiler: `TSAN_CC`, default `cc`
- capability policy: `TSAN_REQUIRED`, default `0`
- accepted `TSAN_REQUIRED`: `0` 또는 `1`만 허용

#### capability probe

script는 작은 pthread program을 `-pthread -fsanitize=thread -g`로 compile합니다. program은 worker가 `g_value = 1`을 쓰고 main이 join 뒤 값을 확인합니다. probe 목적은 race를 만들기 위한 것이 아니라 compiler와 runtime이 basic instrumented pthread binary를 build/run할 수 있는지 확인하는 것입니다.

capability 결과는 다음과 같이 해석합니다.

| 상황 | optional `TSAN_REQUIRED=0` | required `TSAN_REQUIRED=1` | project race로 해석 |
| --- | --- | --- | --- |
| compiler가 probe build 실패 | message + exit 77 | failure exit 1 | 아니오 |
| probe executable 누락 | failure | failure | toolchain inconsistency |
| instrumented probe runtime 실패 | message + exit 77 | failure exit 1 | 아니오 |
| probe stderr에 TSAN runtime error | exit 77 | failure | infrastructure/runtime 문제 |
| probe 통과 뒤 project build 실패 | failure | failure | project/toolchain integration failure |

skip status 77은 sanitizer evidence를 얻지 못했다는 뜻입니다. project correctness success나 race absence를 뜻하지 않습니다.

#### production instrumentation

probe 통과 후 모든 production source를 다음 핵심 flags로 다시 compile/link합니다.

```text
-Wall -Wextra -Werror -pthread -fsanitize=thread -g
```

project build failure는 더 이상 capability skip으로 처리하지 않습니다. probe가 통과했으므로 actual project instrumentation/build failure입니다.

#### runtime options

각 workload는 다음 options를 사용합니다.

```text
TSAN_OPTIONS=halt_on_error=1:exitcode=66
```

TSAN이 issue를 발견하면 첫 diagnostic에서 멈추고 dedicated status 66을 사용합니다. script는 nonzero status와 stderr의 `ThreadSanitizer` marker를 모두 검사합니다.

#### instrumented workloads와 semantic checks

| name | command arguments | semantic assertion |
| --- | --- | --- |
| finite | `7 1000 5 5 4` | valid/nondecreasing log, no death, 각 id 4 eating lines 이상 |
| death | `5 60 80 10` | valid log, exactly one death, no line after |
| contention | `17 2000 5 5 3` | valid/nondecreasing log, no death, 각 id 3 eating lines 이상 |

sanitizer diagnostic이 없다는 이유만으로 통과하지 않습니다. instrumented binary가 조기 종료하거나 required work를 하지 않으면 semantic assertion이 실패합니다. 반대로 semantic output이 정상이어도 TSAN diagnostic이 있으면 실패합니다.

#### 증명 범위

TSAN success는 실행된 workloads와 실제로 발생한 memory accesses에 대한 dynamic evidence입니다. 실행되지 않은 schedule, 교착 상태, fairness, 기아 상태 freedom, strict timing을 증명하지 않습니다.

### 12.6 Verification progression 완성

| risk 또는 항상 유지해야 하는 조건 | smoke | format | concurrency/focused | TSAN |
| --- | --- | --- | --- | --- |
| invalid CLI/overflow | exit와 usage/content | 해당 없음 | 해당 없음 | 핵심 대상 아님 |
| one-philosopher hang/death | timeout + fork/death substring | line grammar | 별도 workload가 주 대상은 아님 | instrumented death workload와 구분 |
| finite global progress | aggregate minimum count | grammar | per-id target, 여러 N, 반복 | instrumented per-id progress |
| malformed/interleaved output | 일부 substring만으로 부족 | every-line regex | 같은 validator + ordering | instrumented output에도 재검사 |
| nondecreasing timestamps | 없음 | 없음 | `check_log` | semantic validator 유지 |
| unique/final death line | single death substring | syntax만 | repeated death + focused harness | instrumented death workload |
| terminal logger overlap | chance에 의존 | syntax symptom만 | 12×200 gated actual logger calls | general instrumented contention |
| data race diagnostic | 없음 | 없음 | behavior symptom만 | exercised accesses에 TSAN |
| infrastructure limitation | timeout 정도 | 해당 없음 | 해당 없음 | probe + optional skip 77/required failure |

### 12.7 각 layer가 잡는 실패와 놓치는 실패

#### public smoke

잡는 것:

- 명백한 CLI acceptance/rejection 회귀
- bounded time 안에 종료하지 않는 public scenario
- single fork/death와 finite work의 기본 observable failure

놓치는 것:

- internal race가 아직 output symptom을 만들지 않은 경우
- exact line grammar의 모든 위반
- philosopher별 progress imbalance

#### grammar

잡는 것:

- interleaved, truncated, unknown, extra-token line
- required phrase가 있어도 함께 섞인 malformed line

놓치는 것:

- syntactically valid하지만 timestamp가 감소하는 output
- unique death, terminal position, semantic progress

#### repeated concurrency와 focused harness

잡는 것:

- 여러 N과 반복에서 드러나는 progress/death symptom
- timestamp ordering과 terminal-line-position 회귀
- logger/death lock boundary의 high-contention symptom

놓치는 것:

- 반복 중 선택되지 않은 schedule
- symptom이 없는 latent data race
- formal fairness/교착 상태 proof

#### ThreadSanitizer

잡는 것:

- instrumented runs에서 실제 발생한 conflicting memory access
- sanitizer 진단 없이 조기 종료하는 false success를 semantic assertions로 차단

놓치는 것:

- 실행되지 않은 code/schedule의 race
- 교착 상태, 기아 상태, fairness, real-time timing
- unsupported environment에서의 project correctness 판단

### 12.8 최종 verification flow

```text
make test
    ↓ build
    ↓ smoke public commands
    ↓ timeout guard
    ↓ exit/content/minimum-work assertions
    ↓ every-line grammar
    ↓ concurrency workloads: 2/5/17 finite
    ↓ repeated 7-philosopher progress ×8
    ↓ forced-death schedule ×10
    ↓ nondecreasing timestamps
    ↓ per-id progress
    ↓ exactly one death / no later line
    ↓ in-process 12 logger × 200 + production death commit

make test-tsan
    ↓ TSAN_CC / TSAN_REQUIRED validation
    ↓ instrumented pthread probe build/run
        ├─ unsupported optional → exit 77
        ├─ unsupported required → failure
        └─ supported
             ↓ full production instrumented build
             ↓ finite/death/contention workloads
             ↓ halt-on-error TSAN diagnostic check
             ↓ grammar/progress/terminal semantic checks
```

### 12.9 최종 해석

이 Thread의 결과는 하나의 “test가 충분합니다”는 결론이 아닙니다. 각 layer가 다른 관찰 실패를 줄입니다.

- deterministic failure injection은 특정 error branch와 ledger mutation을 직접 만듭니다.
- boundary injection은 오래된 synchronization gap을 정확히 재현합니다.
- repeated black-box workloads는 wider integration과 schedule-sensitive symptom을 관찰합니다.
- focused gated harness는 특정 contention boundary의 overlap을 높입니다.
- TSAN은 supported environment에서 exercised memory access를 동적으로 검사합니다.

이 evidence stack이 모두 통과해도 all-schedule race freedom, formal 교착 상태 freedom, fairness, 기아 상태 freedom을 증명한 것으로 해석하면 안 됩니다. 반대로 TSAN이 unsupported여서 77로 skip된 경우에도 project가 통과했다고 볼 수 없습니다.
===== END FILE: 05-layered-evidence-for-concurrent-behavior.md =====

===== BEGIN FILE: README.md =====
# philo Development Thread 학습 골격

## 1. 목적

이 문서 세트는 완성된 프로젝트 해설서가 아닙니다. 학습자가 실제 commit history와 각 SHA의 코드를 직접 읽고 다음 발전 과정을 복원하기 위한 기록 골격입니다.

`설계 → 구현 → 실패 가능성 또는 실제 결함 → 수정 → 검증`

문서에 미리 적힌 Thread 구성, commit 순서, SHA, subject, importance, tags, 역할과 source-confirmed 항상 유지해야 하는 조건은 고정 정보입니다. 학습자는 이를 재평가하지 않고, 각 SHA에서 확인한 실제 코드 근거와 실행 결과만 빈 기록란에 추가합니다.

## 2. 권장 학습 순서

1. 이 README에서 기록 원칙과 importance별 깊이를 확인합니다.
2. [`01-ownership-ledger-to-unsafe-destruction.md`](01-ownership-ledger-to-unsafe-destruction.md)
3. [`02-wall-clock-to-shared-monotonic-start.md`](02-wall-clock-to-shared-monotonic-start.md)
4. [`03-core-routine-to-committed-meal-progress.md`](03-core-routine-to-committed-meal-progress.md)
5. [`04-serialized-output-to-linearized-terminal-state.md`](04-serialized-output-to-linearized-terminal-state.md)
6. [`05-layered-evidence-for-concurrent-behavior.md`](05-layered-evidence-for-concurrent-behavior.md)
7. 모든 Thread의 ledger와 최종 설명을 서로 대조합니다.

Thread 순서는 source에 정의된 Development Threads의 순서를 따릅니다. 한 commit이 여러 Thread에 포함된 문서 세트에서는 중복을 제거하지 않고 각 Thread의 관점으로 다시 확인합니다.

## 3. Thread 문서 사용법

각 문서는 다음 순서로 사용합니다.

1. `Thread 목표`, `핵심 질문`, `완료 기준`을 먼저 읽습니다.
2. `Commit map`에서 source가 확정한 순서와 역할을 확인합니다.
3. commit마다 반드시 해당 SHA로 이동하거나 해당 SHA의 파일을 직접 엽니다.
4. source-confirmed 설명과 실제 코드를 구분하여 기록합니다.
5. `Invariant ledger`에서 항상 유지해야 하는 조건이 도입·강화·실패·복구·검증되는 지점을 연결합니다.
6. `Failure → Fix → Test`에서 수정 commit을 독립 feature가 아니라 기존 가정의 수정으로 복원합니다.
7. 마지막에 Thread 최종 상태와 architecture 또는 execution flow를 자신의 코드 근거로 작성합니다.

빈칸은 감상문이 아니라 다음 중 하나로 채웁니다.

- 파일 경로와 symbol
- 최소 코드 구간
- caller와 callee
- state mutation 전후 순서
- 소유권 또는 borrow 관계
- lock 획득·해제 순서
- 실패 분기와 반환 상태
- cleanup 경로와 ledger 변화
- test injection 지점과 assertion
- 직접 실행한 명령과 결과

## 4. 해당 SHA 코드 확인 원칙

각 commit의 구현은 final HEAD가 아니라 해당 SHA에서 확인합니다.

```sh
git show --name-status --format=fuller <sha>
git diff <sha>^ <sha> --
git switch --detach <sha>
```

확인 규칙은 다음과 같습니다.

- 먼저 `<sha>^ → <sha>`의 직접 변경을 확인합니다.
- Thread의 직전 관련 SHA와 비교할 때에는 중간 commit이 존재할 수 있음을 기록하고, 직접 parent diff를 대체하지 않습니다.
- final HEAD의 함수명, 구조체 필드, lock 순서, test harness를 과거 SHA에 소급하지 않습니다.
- source에 symbol이 명시되어 있어도 실제 선언·정의·호출 관계는 해당 SHA에서 다시 확인합니다.
- 삭제되거나 이름이 바뀐 코드는 final HEAD에서 추측하지 않고 `git show <sha>:<path>`로 확인합니다.
- commit 직전 상태를 쓸 때에는 해당 commit의 parent 또는 문서가 지정한 직전 관련 SHA의 코드를 근거로 삼습니다.

## 5. Importance별 학습 깊이

| Importance | 기록 깊이 |
| --- | --- |
| S | 프로젝트를 설명하는 핵심 architecture 또는 항상 유지해야 하는 조건으로 다룹니다. 문제, 직전 상태, 실패 가능성, 핵심 결정, 실제 핵심 코드, 소유권·lifecycle·상태 전이, 후속 fix와 test까지 연결합니다. |
| A | 주요 subsystem, boundary, 실패 처리, integration point를 설명할 수 있어야 합니다. 핵심 코드와 설계 판단, 수정 전 가정, regression evidence를 확인합니다. |
| B | Thread 흐름에서 맡는 구현 역할과 필요한 코드·상태 변화를 확인합니다. S/A와 같은 분량을 기계적으로 요구하지 않습니다. |
| C | Thread 이해에 필요한 맥락만 기록합니다. 실행 mechanism 또는 항상 유지해야 하는 조건을 만들지 않는 commit에 과도한 분석란을 채우지 않습니다. |

## 6. 실제 코드 삽입 기준

코드는 설명을 대신하기 위해 대량 복사하지 않습니다. 다음을 증명하는 최소 연속 구간만 삽입합니다.

- 구조체가 무엇을 소유하고 무엇을 빌리는지
- state가 어느 mutex 경계에서 읽히거나 변경되는지
- 두 lock의 획득 순서와 해제 순서
- 성공한 자원만 ledger에 반영되는 지점
- failure branch가 어떤 상태를 남기고 어디로 전파되는지
- operation이 시작되는 지점과 commit되는 지점
- test wrapper가 failure를 주입하는 지점과 production path로 연결되는 지점

코드 기록에는 다음 정보를 함께 적습니다.

```text
SHA:
파일:
symbol:
line 범위 또는 주변 문맥:
이 구간이 증명하는 내용:
직전 SHA와 달라진 점:
```

전체 파일, 관계없는 helper, final HEAD의 정리된 버전은 삽입하지 않습니다.

## 7. Test commit 학습 방법

Test commit에서는 production 설명과 test technique을 분리합니다. 각 기록에는 반드시 다음을 포함합니다.

- 대상으로 삼는 production 항상 유지해야 하는 조건
- 재현하는 failure 또는 boundary
- 사용하는 technique
- 실제로 통과하는 production 코드 경로
- 핵심 assertion과 관찰값
- 테스트가 증명하는 것
- 테스트가 증명하지 않는 것
- broad integration test인지 deterministic regression인지
- 후속 변경에서 막아야 하는 회귀

테스트가 timeout, wrapper, macro substitution, child process, 반복 실행, sanitizer를 사용한다면 그 장치가 어떤 불확실성을 제거하거나 어떤 한계를 남기는지 기록합니다. 한 번 통과한 실행을 race freedom, fairness, 교착 상태 freedom 또는 모든 schedule의 증명으로 확대하지 않습니다.

## 8. 문서 완료 기준

문서 세트는 다음 조건을 모두 만족할 때 완료된 것으로 봅니다.

- 모든 Thread의 commit을 source 순서대로 검토했습니다.
- 모든 기록이 해당 SHA의 코드 또는 해당 SHA에서 실행한 test 결과를 가리킵니다.
- S commit마다 architecture, 항상 유지해야 하는 조건, failure, 후속 fix/test의 연결을 설명할 수 있습니다.
- A commit마다 주요 boundary 또는 실패 처리와 설계 판단을 설명할 수 있습니다.
- B/C commit을 필요 이상으로 부풀리지 않았습니다.
- fix마다 기존 가정, failure 또는 위험, root cause, 수정된 항상 유지해야 하는 조건, 실제 수정 코드, regression evidence가 연결됩니다.
- test마다 증명 범위와 비증명 범위가 분리되어 있습니다.
- 각 항상 유지해야 하는 조건 ledger가 도입·강화·부족함·복구·검증의 흐름을 보여 줍니다.
- Thread 최종 상태와 architecture 또는 execution flow를 commit history에 근거해 설명할 수 있습니다.
- final HEAD를 과거 commit의 근거로 사용한 기록이 없습니다.

## 9. 완료본 검증 기록

### 적용한 범위

| 항목 | 값 |
| --- | --- |
| Repository | `seungwoo7050/42-archive` |
| Branch | `c/philo` |
| 확인한 branch HEAD | `12b29d75ccc98311cd8da1217ababbe21de64026` |
| Scaffold source | `development-thread-workbook/scaffold/` |
| Completed output | `development-thread-workbook/completed/` |
| Thread 수 | 5 |
| 참조 commit 수 | 29 |

다른 branch의 구현·test·문서·build logic은 사용하지 않았습니다.

### 스캐폴드 원본 일치 검증

로컬 작업용 scaffold 파일은 Git blob hash를 계산해 `c/philo`의 원격 blob과 일치시켰습니다.

| 파일 | Git blob SHA |
| --- | --- |
| `README.md` | `b030e86400c669094baa0a21b4687c0752ce3a34` |
| `01-ownership-ledger-to-unsafe-destruction.md` | `ac2b19be323faa17ca967b66b78fdb83358cc561` |
| `02-wall-clock-to-shared-monotonic-start.md` | `174cceeff5a52f411f58a1bc4ad0f3274ae88720` |
| `03-core-routine-to-committed-meal-progress.md` | `4bcb96fee5e311455c9eca32163962c02e9b0d2e` |
| `04-serialized-output-to-linearized-terminal-state.md` | `b8b6de07d7567f4d389654d5ea2e5a290e0ebda8` |
| `05-layered-evidence-for-concurrent-behavior.md` | `40cacef3a1d6c0b3fd7f4d5f7af800563984e28c` |

고정된 Thread 구성, commit map, SHA, subject, importance, tags, source-defined role과 항상 유지해야 하는 조건 문구는 유지했습니다. learner-facing checkbox·빈 표·빈 결론란만 완료 상태로 바꾸고, 각 문서의 `§12 저장소 기반 완료 기록`에 실제 SHA별 근거를 추가했습니다.

### branch ancestry 검증

아래 29개 SHA 각각을 `c/philo` HEAD와 비교했습니다. 모든 비교에서 해당 SHA가 merge base와 일치하고 branch가 그 SHA보다 앞선 상태여서 branch ancestry에 속함을 확인했습니다.

```text
16343e76b54b  1d69df7db78c  10665e0a5bf9  800408d6d84e
 a7783d04107f  7586b605302b  37b29557cccc
509453b01515  a21e4cc75272  5b32d5bdb955  f01d62cde8ce
 e7e62cbe185f  bfbfa0431732  f57f6ec0be87
b68f40819af4  c8531c91f0fb  fe0a2d15b29b  53e591effb4a
 73b5551a76f4  4c224ae86f2b  054ef46f80c7
033ad537d166  40ea0f871300  a2e90b84641b  c424b7d91ed1
bd6bb8eb18f4  f145d33f2773  3d24bea01441  20f8270c78bb
```

### 증거 구분

| 증거 종류 | 수행 여부 | 기록 방식 |
| --- | --- | --- |
| 해당 SHA의 commit diff와 파일 검토 | 수행 | 각 Thread §12에 SHA·파일·symbol·state mutation·failure/test mechanism을 기록했습니다. |
| branch ancestry 확인 | 수행 | 29개 SHA를 branch HEAD와 개별 비교했습니다. |
| scaffold blob 일치 확인 | 수행 | 위 Git blob SHA 표로 기록했습니다. |
| production build 및 test 실행 | 미수행 | 로컬 환경에서 GitHub checkout을 위한 DNS/network 연결이 차단됐습니다. 실행 통과를 주장하지 않습니다. |
| test source의 injection/assertion 검토 | 수행 | deterministic wrapper, child process, timeout, repeated workload, TSAN probe를 code-inspection evidence로 구분했습니다. |
| completed directory 구조 검증 | 수행 | scaffold와 동일한 6개 상대 경로만 포함하고 추가 문서를 만들지 않았습니다. |

문서의 `[x]`는 해당 코드·test mechanism을 저장소에서 확인하고 설명을 작성했다는 뜻입니다. production 명령을 실제 실행했다는 표시는 아닙니다. 실행 결과가 필요한 항목은 각 Thread에서 source inspection과 runtime evidence를 구분했습니다.

### Thread별 최종 연결

| Thread | 최종 복원 결과 |
| --- | --- |
| 01 | table 소유권과 partial-init ledger가 successful join 기반 destruction permission 및 `PHILO_UNSAFE`/`_exit`까지 확장됩니다. |
| 02 | wall-clock helper가 monotonic `int64_t` time model로 교정되고 all-ready barrier가 one shared start epoch를 publish합니다. |
| 03 | fork acquisition attempt가 full-duration·active-state recheck를 통과한 committed meal progress로 정교화되며 internal counter range가 확장됩니다. |
| 04 | complete-line serialization이 fresh death revalidation과 common lock order를 통한 terminal-state linearization으로 강화됩니다. |
| 05 | smoke, grammar, repeated concurrency, focused contention, TSAN이 서로 다른 failure class를 관찰하는 verification stack을 구성합니다. |

### 실행한 로컬 완료본 검증

다음 검증은 문서·archive 구조에 대한 것이며 production code test가 아닙니다.

```sh
python /tmp/validate_philo_workbook.py
```

결과:

```text
scaffold files: 6
completed files: 6
commit-map SHAs: 29
markdown_it parsed all files
VALIDATION OK
```

validator는 다음을 확인했습니다.

- scaffold와 completed의 상대 파일 집합이 동일합니다.
- `README.md`와 5개 Thread 문서 외 파일이 없습니다.
- commit map의 SHA, 순서, subject, importance, tags, source-defined role이 원본과 같습니다.
- section 12 이전의 고정 스캐폴드 문구는 checkbox·빈 learner field를 제외하고 원본과 같습니다.
- unchecked checkbox, 빈 table cell, 비어 있는 결론 placeholder가 남지 않았습니다.
- fenced code block과 Markdown table 구조가 유효하고 모든 파일을 CommonMark parser가 읽었습니다.
===== END FILE: README.md =====
