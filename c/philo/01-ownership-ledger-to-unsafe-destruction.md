# Thread: Ownership ledger to unsafe-destruction verdict

> Project: `philo` · Branch: `c/philo` · Development Thread 1/5

이 Thread는 `t_table`이 배열과 동기화 객체를 소유하는 최초 구조에서 출발해, **부분 초기화 상태를 사실대로 기록하는 ledger**와 **worker가 모두 멈췄다는 증거가 없으면 파괴를 금지하는 판정**까지 확장되는 과정을 다룬다.

핵심은 “정리 함수를 호출했는가”가 아니다. 실제로 생성된 자원만 한 번 파괴하고, 파괴가 실패하면 아직 남은 자원을 나타내는 상태를 보존하며, 어떤 worker가 공유 메모리를 계속 참조할 가능성이 있으면 정리 자체를 하지 않아야 한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `16343e76b54b` | `feat(init): 테이블 저장소와 철학자 관계 초기화` | S | `ARCH, CORE, RESOURCE_LIFECYCLE` | Establishes the table as the owner of allocations and the ring objects borrowed by philosophers. |
| 2 | `1d69df7db78c` | `feat(init): 뮤텍스 수명주기와 실패 롤백 구현` | A | `RESOURCE_LIFECYCLE, ARCH, RISK` | Adds staged mutex construction and resource-readiness ledgers, but still splits rollback responsibility. |
| 3 | `10665e0a5bf9` | `fix(init): 포크 초기화 실패 시 중복 정리 방지` | A | `RESOURCE_LIFECYCLE, DEBUG, RISK` | Centralizes partial fork rollback in the common destructor and restores exact-once cleanup. |
| 4 | `800408d6d84e` | `test(init): 부분 뮤텍스 초기화 롤백 검증` | A | `TEST, RESOURCE_LIFECYCLE, RISK` | Injects initialization failure and proves that each prepared mutex is destroyed once and allocations are released. |
| 5 | `a7783d04107f` | `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파` | S | `RESOURCE_LIFECYCLE, RISK, HARD` | Extends ownership evidence to worker creation, successful join, destruction permission, retryable cleanup, and `_exit` on unsafe state. |
| 6 | `7586b605302b` | `test(lifecycle): 생성·결합·정리 실패 경로 검증` | A | `TEST, RESOURCE_LIFECYCLE, EDGE` | Exercises create, join, and destroy failures across multiple partial-state positions. |
| 7 | `37b29557cccc` | `test(main): 결합 실패 시 안전하지 않은 정리 방지` | A | `TEST, RESOURCE_LIFECYCLE, RISK` | Proves the executable does not destroy resources or execute normal stdio and `atexit` teardown after an unsafe join result. |

## `16343e76b54b` — 안정된 저장소와 빌려 쓰는 주소

최초 구조에서 `t_table`은 두 개의 연속 배열을 할당한다. `t_philo`는 fork나 table을 복사해서 소유하지 않고, 배열 안의 객체와 table을 가리키는 포인터만 보관한다.

```c
/* 16343e76b54b, src/init.c */
table->forks = malloc(sizeof(*table->forks) * config->number);
table->philos = malloc(sizeof(*table->philos) * config->number);
if (table->forks == NULL || table->philos == NULL)
    return (philo_table_destroy(table), PHILO_ERR);
```

```c
static void assign_philos(t_table *table)
{
    int i;

    i = 0;
    while (i < table->config.number)
    {
        table->philos[i].id = i + 1;
        table->philos[i].left_fork = &table->forks[i];
        table->philos[i].right_fork
            = &table->forks[(i + 1) % table->config.number];
        table->philos[i].table = table;
        i++;
    }
}
```

이 주소 계산은 fork를 원형으로 배치한다. philosopher `i`의 오른쪽 fork와 다음 philosopher의 왼쪽 fork는 같은 배열 원소다.

```text
philo[i].right_fork == &forks[(i + 1) % N]
philo[(i + 1) % N].left_fork == &forks[(i + 1) % N]
```

따라서 `t_table` 안의 allocation이 살아 있는 동안에만 philosopher의 fork pointer가 유효하다. 이 소유 관계는 후반부 join 실패 처리의 직접적인 근거가 된다. worker가 아직 살아 있다면 `table`, `philos`, `forks` 중 어느 것도 먼저 해제할 수 없다.

| 대상 | 관계 | 이 시점의 종료 처리 |
| --- | --- | --- |
| `t_table` 값 | `main`이 보유하는 중심 객체 | 자체는 stack value이며 destructor 대상이 아니다. |
| `table->forks` | table이 소유하는 heap allocation | `free` 후 `NULL` |
| `table->philos` | table이 소유하는 heap allocation | `free` 후 `NULL` |
| `t_philo.table` | table을 빌리는 pointer | table과 두 배열보다 오래 살 수 없다. |
| `left_fork`, `right_fork` | fork 배열 원소를 빌리는 pointer | fork 배열이 해제되면 모두 무효가 된다. |

이 커밋은 저장소와 관계만 만든다. 구조체에는 mutex와 readiness field가 이미 있지만, 실제 `pthread_mutex_init`과 부분 초기화 rollback은 아직 없다.

## 부분 초기화 ledger가 처음에는 두 번 소비된 이유

### `1d69df7db78c` — 성공한 초기화 수를 기록하다

`state_mutex`, `print_mutex`, fork mutex가 순서대로 초기화되고, 성공한 fork 수는 `fork_count`에 기록된다. readiness flag도 초기화 성공 직후에만 올라간다.

```c
if (pthread_mutex_init(&table->state_mutex, NULL) != 0)
    return (philo_table_destroy(table), PHILO_ERR);
table->state_ready = 1;

if (pthread_mutex_init(&table->print_mutex, NULL) != 0)
    return (philo_table_destroy(table), PHILO_ERR);
table->print_ready = 1;
```

문제는 fork 초기화 helper가 실패 시 이미 만든 mutex를 직접 파괴하면서도 `fork_count`를 되돌리지 않았다는 데 있다.

```c
/* 1d69df7db78c의 문제 경로 */
if (pthread_mutex_init(&table->forks[i], NULL) != 0)
{
    while (--i >= 0)
        pthread_mutex_destroy(&table->forks[i]);
    return (PHILO_ERR);
}
table->fork_count++;
```

호출자는 helper 실패 후 다시 `philo_table_destroy(table)`를 호출한다. 공통 destructor는 여전히 `fork_count`만큼 fork가 살아 있다고 믿으므로 helper가 이미 파괴한 mutex를 다시 파괴할 수 있다.

```text
fork mutex 초기화 성공
  → fork_count 증가
다음 fork 초기화 실패
  → init_forks가 앞선 mutex를 직접 destroy
  → fork_count는 그대로
philo_table_destroy
  → 같은 mutex를 fork_count 근거로 다시 destroy
```

rollback을 수행하는 주체가 helper와 destructor 두 곳으로 나뉜 것이 root cause다. `fork_count`는 성공한 초기화의 기록이어야 하는데, 한쪽 cleanup이 그 기록을 소비하지 않은 채 실제 자원만 없애 버렸다.

### `10665e0a5bf9` — destructor를 유일한 ledger 소비자로 만들다

수정은 helper의 rollback을 삭제하는 것이다. 초기화에 실패한 helper는 오류만 반환하고, 이미 성공한 자원은 모두 공통 destructor가 정리한다.

```diff
 if (pthread_mutex_init(&table->forks[i], NULL) != 0)
-{
-    while (--i >= 0)
-        pthread_mutex_destroy(&table->forks[i]);
     return (PHILO_ERR);
-}
```

```c
/* 10665e0a5bf9 */
while (table->fork_count > 0)
    pthread_mutex_destroy(&table->forks[--table->fork_count]);

if (table->print_ready)
{
    pthread_mutex_destroy(&table->print_mutex);
    table->print_ready = 0;
}
if (table->state_ready)
{
    pthread_mutex_destroy(&table->state_mutex);
    table->state_ready = 0;
}
```

이 시점부터 같은 destructor를 다시 호출해도 소비할 ledger가 남지 않는다. 다만 여기서는 `pthread_mutex_destroy`의 반환값을 보지 않으므로, 파괴 자체가 실패했을 때 truthful state를 남기는 문제는 아직 해결하지 않는다. 그 문제는 `a7783d04107f`에서 다시 열린다.

### `800408d6d84e` — 반환값이 아니라 실제 파괴 주소를 검사하다

테스트는 `src/init.c`를 다음처럼 재컴파일한다.

```sh
-Dpthread_mutex_init=test_mutex_init
-Dpthread_mutex_destroy=test_mutex_destroy
```

주입 함수는 네 번째 mutex 초기화 호출을 실패시킨다. 호출 순서는 state, print, 첫 번째 fork, 두 번째 fork이므로 실제로 준비된 mutex는 세 개다. destroy wrapper는 이미 본 주소가 다시 들어오면 `duplicate_destroy`를 설정한다.

검증하는 결과는 세 가지다.

- 초기화 실패가 `PHILO_ERR`로 전파된다.
- 서로 다른 세 mutex만 파괴된다.
- 두 allocation은 `NULL`이 되고, 실패 뒤 destructor를 한 번 더 호출해도 destroy 횟수가 늘지 않는다.

이 테스트는 real pthread implementation의 실패를 재현하는 것이 아니라, **ledger 소비 순서와 exact-once 호출**을 결정적으로 검증한다.

## `a7783d04107f` — 자원 ledger를 worker 수명까지 확장하다

mutex가 몇 개 초기화되었는지만으로는 table을 파괴해도 되는지 판단할 수 없다. `pthread_create`가 성공한 순간부터 worker는 `t_philo`, `t_table`, fork mutex를 역참조할 수 있다. 따라서 destruction permission에는 worker가 실제로 멈췄다는 증거가 필요하다.

이 커밋은 세 field를 추가한다.

```c
int threads_started;
int threads_joined;
int destroy_safe;
```

create 성공 직후에만 `threads_started`를 증가시키고, join 성공 직후에만 `threads_joined`를 증가시킨다.

```c
/* a7783d04107f, src/run.c */
if (pthread_create(&table->philos[i].thread, NULL,
        philo_routine, &table->philos[i]) != 0)
{
    release_start(table, 1);
    join_status = join_started(table, table->threads_started);
    if (join_status != PHILO_OK)
        return (join_status);
    return (PHILO_ERR);
}
table->threads_started++;
```

```c
static int join_started(t_table *table, int count)
{
    int i;
    int status;

    i = 0;
    status = PHILO_OK;
    while (i < count)
    {
        if (pthread_join(table->philos[i].thread, NULL) == 0)
            table->threads_joined++;
        else
        {
            table->destroy_safe = 0;
            status = PHILO_UNSAFE;
        }
        i++;
    }
    return (status);
}
```

중요한 차이는 join을 **호출했다**는 사실이 아니라 join이 **성공했다**는 사실만 ledger에 기록한다는 점이다. 한 worker라도 성공적으로 join되지 않았다면 해당 worker가 table을 더 이상 읽지 않는다고 증명할 수 없다.

### 파괴 허용 조건과 재시도 가능한 상태

```c
/* a7783d04107f, src/init.c */
if (!table->destroy_safe
    || table->threads_joined < table->threads_started)
    return (PHILO_UNSAFE);
```

이 조건이 거짓일 때만 mutex 파괴를 시작한다. 또한 fork ledger는 destroy 성공 뒤에만 감소한다.

```c
while (table->fork_count > 0)
{
    if (pthread_mutex_destroy(
            &table->forks[table->fork_count - 1]) != 0)
        return (PHILO_ERR);
    table->fork_count--;
}
```

readiness flag도 같은 방식으로 성공 뒤에만 내려간다.

```c
if (table->print_ready)
{
    if (pthread_mutex_destroy(&table->print_mutex) != 0)
        return (PHILO_ERR);
    table->print_ready = 0;
}
```

따라서 destroy가 중간에 실패하면 이미 성공한 항목은 ledger에서 빠지고, 실패한 항목과 아직 시도하지 않은 항목은 그대로 남는다. 다음 호출은 남아 있는 정확한 지점에서 다시 시작할 수 있다.

| 경로 | 남겨야 하는 증거 | 파괴 가능 여부 |
| --- | --- | --- |
| create 실패 전 worker 없음 | `threads_started == 0` | 가능 |
| 일부 create 성공, 모두 join 성공 | `joined == started` | 가능 |
| join 하나 이상 실패 | `destroy_safe == 0`, `joined < started` | 금지 |
| fork mutex destroy 중 실패 | 실패한 mutex를 포함한 `fork_count` 유지 | 이후 재시도 가능 |
| print/cond/state destroy 실패 | 해당 readiness flag 유지 | 이후 재시도 가능 |

### 왜 `main`은 `_exit`를 사용하는가

`PHILO_UNSAFE`는 일반 오류보다 강한 판정이다. destructor를 생략해도 `return`이나 `exit`로 정상 종료하면 stdio buffer flush와 `atexit` handler가 실행된다. 그 과정이 아직 살아 있을 수 있는 worker와 공유 runtime state를 건드릴 가능성을 배제할 수 없다.

```c
run_status = philo_run(&table);
if (run_status == PHILO_UNSAFE)
{
    put_error("Error: worker thread could not be joined\n");
    _exit(1);
}
```

오류 문구는 `write`로 먼저 내보낸다. `_exit`는 table destructor뿐 아니라 buffered stdio와 `atexit`까지 건너뛴다. 이 선택은 graceful cleanup을 포기하는 대신, 증명되지 않은 worker quiescence 상태에서 user-space teardown을 진행하지 않는 선택이다.

## 실패 위치를 직접 고정한 검증

### `7586b605302b`

`pthread_create`, `pthread_join`, `pthread_mutex_destroy`를 wrapper로 치환해 호출 위치별 실패를 만든다.

- create 0·1·2번째 실패: `threads_started == threads_joined == fail_at`이고 정상 destructor가 가능해야 한다.
- join 0·1번째 실패: `PHILO_UNSAFE`, `threads_joined == N - 1`; destructor 호출 전후에 `forks` pointer, `fork_count`, destroy 호출 수가 그대로여야 한다.
- destroy 여러 위치 실패: 그 시점에 아직 남은 `fork_count`가 보존되고, 실패 주입을 해제한 두 번째 destructor 호출이 성공해야 한다.

join 실패 테스트는 실패한 thread를 test harness에서 직접 join한 뒤 `destroy_safe`와 `threads_joined`를 복구하고 cleanup을 재시도한다. production code가 자동으로 unsafe state를 정상 상태로 바꾼다는 뜻은 아니다. 테스트가 보존된 ledger의 재사용 가능성을 확인하기 위한 절차다.

### `37b29557cccc`

`main.c`의 의존 함수를 test double로 바꾼다. `test_run`은 buffered `printf` marker를 남긴 뒤 `PHILO_UNSAFE`를 반환하고, fake destructor와 `atexit` hook도 각각 marker를 출력하도록 한다.

기대 결과는 다음과 같다.

```text
보여야 함:   Error: worker thread could not be joined
보이면 실패: unsafe destroy called
보이면 실패: buffered stdio marker
보이면 실패: normal exit hook
```

즉 이 테스트는 단순히 exit status가 1인지가 아니라, **금지된 teardown이 실행되지 않았다는 부재 증거**를 확인한다.

## Thread가 확립한 불변 조건

1. `t_table`은 배열과 synchronization object의 owner이고, worker는 그 내부 주소를 빌린다.
2. 성공한 초기화만 ledger에 기록하며, 공통 destructor만 그 ledger를 소비한다.
3. destroy가 실패하면 실패한 자원을 아직 보유한다는 상태를 지우지 않는다.
4. 성공한 join 수가 시작된 worker 수와 같지 않으면 shared storage를 파괴하지 않는다.
5. join 실패의 안전 판정은 일반 run/cleanup 오류보다 우선하며, 정상 프로세스 teardown도 허용하지 않는다.

## 이 Thread의 경계

이 Thread는 unjoined worker를 안전하게 강제 종료하는 방법을 제공하지 않는다. join 실패 뒤에는 자원을 의도적으로 남기고 프로세스를 즉시 종료한다. 또한 실제 kernel이나 pthread 구현에서 모든 failure mode를 일으킨 것이 아니라 wrapper 기반의 결정적 실패 위치를 검증했다.

시간 기준, 시작 장벽의 의미, 식사 transaction, terminal logging은 각각 다른 Thread에서 다룬다. 여기서 필요한 사실은 하나다. **borrower가 멈췄다는 증거가 없으면 owner의 파괴는 cleanup이 아니라 use-after-free가 될 수 있다.**

> 조사 범위: 표시된 exact SHA의 GitHub diff와 해당 SHA의 source/test를 확인했다. 이 환경에서는 branch를 checkout하여 build 또는 test suite를 실행하지 않았다.
