# Thread: Serialized output to linearized terminal state

> Project: `philo` · Branch: `c/philo` · Development Thread 4/5

이 Thread는 line 단위 출력 직렬화가 왜 terminal correctness와 같지 않은지 보여준다. `print_mutex`로 로그가 서로 섞이지 않게 만들 수는 있지만, death/completion을 관찰한 시점과 `ended`를 게시한 시점, 마지막 terminal line이 출력되는 시점이 서로 떨어져 있으면 오래된 판정이나 terminal 이후 ordinary action이 남는다.

최종 설계는 두 종류의 terminal 전환을 각각 명확한 commit 지점에 둔다.

- meal completion: `state_mutex`를 보유한 상태에서 `ended = 1`
- death: `print_mutex → state_mutex`를 보유한 상태에서 fresh time과 latest meal state를 재검증한 뒤 `ended = 1`, 이어 같은 print section에서 `died` 출력

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `033ad537d166` | `feat(log): 상태 로그의 동시 출력 보호` | A | `TERMINAL_STATE, CONCURRENCY, ARCH` | Introduces synchronized terminal-state access and a print mutex, but does not yet couple death publication to final output atomically. |
| 2 | `40ea0f871300` | `feat(monitor): 사망과 식사 완료 조건 감시` | S | `CORE, CONCURRENCY, TERMINAL_STATE` | Establishes the main-thread monitor as the authority for starvation and global completion. |
| 3 | `a2e90b84641b` | `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정` | S | `TERMINAL_STATE, CONCURRENCY, RISK` | Rechecks death under `print_mutex → state_mutex`, commits completion while locked, and gives terminal state explicit linearization points. |
| 4 | `c424b7d91ed1` | `test(monitor): 완료 상태와 오래된 사망 판정 검증` | A | `TEST, TERMINAL_STATE, DEBUG` | Mutates state at the old unlock boundary to prove stale candidates are rejected and completion is already terminal before release. |

## `033ad537d166` — 완전한 한 줄과 종료 상태를 각각 보호하다

이 커밋은 shared `ended` 접근을 `state_mutex` 뒤로 옮기고, ordinary status line 전체를 `print_mutex` 안에서 출력한다.

```c
/* 033ad537d166, src/state.c */
int philo_has_ended(t_table *table)
{
    int ended;

    pthread_mutex_lock(&table->state_mutex);
    ended = table->ended;
    pthread_mutex_unlock(&table->state_mutex);
    return (ended);
}
```

```c
void philo_log(t_philo *philo, const char *message)
{
    t_table *table;
    long     timestamp;

    table = philo->table;
    pthread_mutex_lock(&table->print_mutex);
    if (!philo_has_ended(table))
    {
        timestamp = philo_now_ms() - table->start_ms;
        printf("%ld %d %s\n", timestamp, philo->id, message);
    }
    pthread_mutex_unlock(&table->print_mutex);
}
```

ordinary logger의 lock 순서는 실제로 다음과 같다.

```text
print_mutex lock
  → state_mutex lock/read/unlock  // philo_has_ended
  → timestamp + complete printf
print_mutex unlock
```

따라서 두 worker의 timestamp, id, message가 한 줄 안에서 섞이지 않는다. print lock을 기다리던 logger는 lock을 얻은 뒤 terminal state를 다시 확인하므로 이미 끝난 simulation의 새 line을 대체로 억제한다.

### 최초 death 경로에 남은 공백

```c
void philo_log_death(t_philo *philo)
{
    t_table *table;
    long     timestamp;
    int      should_print;

    table = philo->table;
    pthread_mutex_lock(&table->state_mutex);
    should_print = !table->ended;
    table->ended = 1;
    pthread_mutex_unlock(&table->state_mutex);
    if (should_print)
    {
        pthread_mutex_lock(&table->print_mutex);
        timestamp = philo_now_ms() - table->start_ms;
        printf("%ld %d died\n", timestamp, philo->id);
        pthread_mutex_unlock(&table->print_mutex);
    }
}
```

`should_print`와 `ended` 변경은 state lock 안에 있으므로 여러 caller가 동시에 death를 시도해도 at-most-one print attempt만 얻는다. 그러나 terminal publication과 final line은 하나의 critical section이 아니다.

다음 interleaving이 가능하다.

```text
normal logger: print_mutex 획득
normal logger: ended == 0 확인, state_mutex 해제

death path:   state_mutex 획득
              ended = 1 게시
              state_mutex 해제
              print_mutex 대기

normal logger: ordinary line 출력
normal logger: print_mutex 해제

death path:   print_mutex 획득, died 출력
```

output text 자체는 ordinary line 다음에 death line으로 직렬화되지만, ordinary line은 이미 terminal state가 게시된 뒤 실행된다. 즉 “한 줄씩 출력된다”는 보장과 “terminal decision 이후 ordinary output이 없다”는 보장이 다르다.

또한 death caller가 들고 온 판정이 여전히 유효한지는 이 함수가 확인하지 않는다. 이 문제는 monitor가 도입되면서 구체적인 stale-candidate race가 된다.

## `40ea0f871300` — monitor가 전체 종료 판정을 맡다

worker는 각자의 `last_meal_ms`, `meals`와 table의 `full_count`를 게시한다. main thread에서 실행되는 monitor는 이 state를 읽어 두 가지 global condition을 판단한다.

```c
static int all_meals_done(t_table *table)
{
    return (table->config.has_meal_limit
        && table->full_count >= table->config.number);
}

static t_philo *find_dead_philo(t_table *table, long now)
{
    int i;

    i = 0;
    while (i < table->config.number)
    {
        if (now - table->philos[i].last_meal_ms
            >= table->config.time_to_die)
            return (&table->philos[i]);
        i++;
    }
    return (NULL);
}
```

이 responsibility split은 타당하다. worker는 local progress를 생산하고 monitor는 global termination policy를 소유한다. 문제는 최초 구현이 **관찰 lock을 놓은 뒤 commitment를 수행했다는 것**이다.

### 완료 판정과 게시 사이의 공백

```c
pthread_mutex_lock(&table->state_mutex);
if (all_meals_done(table))
{
    pthread_mutex_unlock(&table->state_mutex);
    philo_finish(table);
    return ;
}
```

`all_meals_done`이 참인 순간과 `philo_finish`가 다시 state lock을 얻어 `ended = 1`을 쓰는 순간 사이에 gap이 있다. 그 사이 ordinary logger는 여전히 `ended == 0`을 보고 진행할 수 있다.

completion predicate와 terminal publication은 같은 shared state에 대한 하나의 decision인데 두 critical section으로 나뉘어 있다.

### 오래된 사망 후보가 생기는 공백

```c
now = philo_now_ms();
pthread_mutex_lock(&table->state_mutex);
dead = find_dead_philo(table, now);
pthread_mutex_unlock(&table->state_mutex);
if (dead != NULL)
{
    philo_log_death(dead);
    return ;
}
```

monitor가 `dead` pointer를 얻은 뒤 state lock을 놓으면 해당 philosopher가 fork를 획득하고 `last_meal_ms`를 갱신할 수 있다. 하지만 최초 `philo_log_death`는 latest meal state를 보지 않고 무조건 `ended = 1`을 시도한다.

```text
monitor: now - old_last_meal >= time_to_die 확인
monitor: state_mutex 해제
worker:  식사 시작, last_meal_ms = fresh_now
monitor: old candidate로 philo_log_death 호출
         → recheck 없이 ended = 1
```

관찰 당시에는 맞았던 predicate가 commitment 순간에는 틀릴 수 있다. concurrency에서 candidate observation은 terminal decision 자체가 아니다.

## `a2e90b84641b` — 관찰과 확정 사이를 닫다

### completion은 관찰한 lock 안에서 바로 commit한다

```c
/* a2e90b84641b, src/monitor.c */
pthread_mutex_lock(&table->state_mutex);
if (table->ended)
{
    pthread_mutex_unlock(&table->state_mutex);
    return ;
}
if (all_meals_done(table))
{
    table->ended = 1;
    pthread_mutex_unlock(&table->state_mutex);
    return ;
}
dead = find_dead_philo(table, now);
pthread_mutex_unlock(&table->state_mutex);
```

`full_count`를 보고 completion을 결정하는 순간 같은 lock 아래에서 `ended`를 게시한다. unlock 시점에는 predicate observation과 terminal publication이 모두 끝난 상태다.

### 사망 후보는 최종 출력 경계에서 다시 확인한다

```c
/* a2e90b84641b, src/state.c */
int philo_try_log_death(t_philo *philo)
{
    t_table *table;
    int64_t  now;
    int64_t  timestamp;
    int      should_print;

    table = philo->table;
    should_print = 0;
    timestamp = 0;
    pthread_mutex_lock(&table->print_mutex);
    pthread_mutex_lock(&table->state_mutex);
    now = philo_now_ms();
    if (!table->ended
        && now - philo->last_meal_ms
            >= table->config.time_to_die)
    {
        table->ended = 1;
        timestamp = now - table->start_ms;
        should_print = 1;
    }
    pthread_mutex_unlock(&table->state_mutex);
    if (should_print)
        printf("%lld %d died\n",
            (long long)timestamp, philo->id);
    pthread_mutex_unlock(&table->print_mutex);
    return (should_print);
}
```

monitor의 첫 scan은 candidate를 찾는 빠른 탐색일 뿐이다. 실제 commitment 함수는 다음을 새로 읽는다.

- fresh `philo_now_ms()`
- latest `philo->last_meal_ms`
- current `table->ended`

이 세 값이 같은 state section에서 death predicate를 만족할 때만 `ended = 1`을 게시한다. 오래된 candidate라면 0을 반환하고 monitor는 scan loop를 계속한다.

### 왜 `print_mutex → state_mutex`인가

ordinary logger도 print lock을 먼저 잡고 그 안에서 state를 확인한다. death path가 같은 순서를 사용하면 terminal decision과 ordinary line이 하나의 total order에 들어간다.

#### 일반 logger가 먼저 print lock을 얻은 경우

```text
normal: print lock
normal: state lock → ended == 0 → state unlock
normal: ordinary line
normal: print unlock

death:  print lock
        state lock → fresh predicate true → ended = 1
        state unlock → died line → print unlock
```

ordinary line은 death commitment보다 먼저 선형화된다.

#### death path가 먼저 print lock을 얻은 경우

```text
death:  print lock
        state lock → ended = 1
        state unlock → died line → print unlock

normal: print lock
        state lock → ended == 1
        state unlock → 출력 없음 → print unlock
```

`died` 뒤에 ordinary line이 나오지 않는다. 여러 death candidate가 경쟁해도 `ended` 변경은 state lock 아래에서 한 번만 성공한다.

state lock은 `printf` 동안 유지하지 않는다. terminal flag와 timestamp를 확정한 뒤 state lock은 놓되 print lock은 final line이 끝날 때까지 유지한다. shared state mutation은 짧게, output ordering은 complete-line boundary까지 보호하는 분리다.

## `c424b7d91ed1` — 우연한 timing 대신 이전 unlock 경계를 직접 건드리다

테스트는 `monitor.c`만 다음 macro로 컴파일한다.

```sh
-Dpthread_mutex_unlock=test_mutex_unlock
```

wrapper는 실제 `pthread_mutex_unlock`을 먼저 수행한 뒤, monitor가 처음 `state_mutex`를 놓는 바로 그 경계에서 test state를 바꾼다.

### 완료 모드

`full_count = number`인 상태로 monitor를 시작한다. wrapper가 state unlock을 관찰할 때 `ended`가 이미 참이어야 한다.

```c
g_ended_at_unlock = g_table->ended;
```

이 assertion은 “monitor가 결국 종료했다”보다 강하다. completion publication이 unlock 뒤 별도 helper에 남아 있지 않고, lock을 놓기 전에 끝났음을 확인한다.

### 오래된 사망 후보 모드

초기 state는 philosopher가 death threshold를 넘은 것처럼 만든다. monitor가 candidate를 scan하고 state lock을 놓는 순간 wrapper가 다음 변경을 수행한다.

```c
pthread_mutex_lock(&g_table->state_mutex);
g_table->philos[0].last_meal_ms = philo_now_ms();
g_table->full_count = 1;
pthread_mutex_unlock(&g_table->state_mutex);
```

이제 이전 candidate는 stale이고 completion condition은 참이다. `philo_try_log_death`가 latest state를 재검증하면 death를 거부하고, 다음 monitor iteration이 completion을 commit한다.

테스트는 stdout에 `died`가 하나라도 있으면 실패한다.

```sh
grep -q 'died' terminal_state.out && fail 'stale death was printed'
```

반복 실행으로 race가 우연히 발생하기를 기다리는 대신, 잘못된 구현에서 gap이 있던 정확한 unlock 위치에 mutation을 넣는 deterministic boundary test다.

## 최종 종료 전이

```text
[monitor scan]
  ↓ state_mutex
ended? → return
all meals done? → ended = 1, unlock, return
candidate dead? → pointer만 얻고 unlock
  ↓
[philo_try_log_death]
  ↓ print_mutex
  ↓ state_mutex
fresh now + latest last_meal + current ended 재검증
  ├─ 거짓: unlock state/print, monitor 계속
  └─ 참: ended = 1 + timestamp 확정
           ↓ state unlock
         died line 출력
           ↓ print unlock
```

| 불변 조건 | 코드상의 선형화 지점 |
| --- | --- |
| meal completion은 한 번 terminal이 된다 | `state_mutex` 안의 `table->ended = 1` |
| stale death는 commit되지 않는다 | `philo_try_log_death`의 fresh predicate |
| death commitment는 최대 한 번이다 | `!table->ended` 검사와 변경을 묶은 state section |
| terminal death line 뒤 ordinary line이 없다 | normal/death 공통 `print_mutex → state_mutex` 순서 |
| 한 status line의 field가 섞이지 않는다 | complete `printf`를 감싸는 `print_mutex` |

## 보장하지 않는 것

이 구조는 monitor polling과 scheduler 지연을 제거하지 않으므로 death line의 strict real-time latency를 보장하지 않는다. `printf` 반환값도 확인하지 않으므로 terminal state가 commit되었다고 해서 final line이 OS에 성공적으로 전달되었다는 보장은 없다. at-most-one **commit/attempt**와 exactly-one successful I/O delivery는 다른 주장이다.

또한 이 Thread는 worker routine의 meal commit 의미나 startup epoch를 다시 정의하지 않는다. 이미 게시된 state를 monitor와 logger가 어떤 순서로 관찰하고 terminal decision으로 확정하는지만 다룬다.

> 조사 범위: 표시된 exact SHA의 GitHub diff와 해당 SHA의 source/test를 확인했다. 이 환경에서는 branch를 checkout하여 build 또는 test suite를 실행하지 않았다.
