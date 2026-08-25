# Thread: Wall-clock helper to one shared monotonic start epoch

> Project: `philo` · Branch: `c/philo` · Development Thread 2/5

이 Thread는 단순한 밀리초 helper가 **경과 시간을 측정하기 위한 단조 시계**로 교정되고, 다시 worker readiness 장벽과 결합해 모든 philosopher가 같은 시작 시각을 공유하는 과정이다.

두 문제를 구분해야 한다.

- 어떤 clock으로 elapsed time을 계산할 것인가
- worker가 실제 활동을 시작한 시점을 언제 하나의 기준 시각으로 게시할 것인가

`CLOCK_MONOTONIC`만 사용해도 늦게 시작한 worker가 이미 굶은 상태가 되는 문제는 해결되지 않는다. 반대로 시작 장벽만 있어도 wall-clock 보정으로 시간이 뒤로 가거나 앞으로 뛰는 문제는 남는다. 이 Thread의 최종 상태는 두 조건을 함께 만족한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `509453b01515` | `feat(time): 밀리초 시각 계산 함수 추가` | B | `TIME_MODEL, CORE` | Centralizes millisecond time and interruptible deadline waits, initially with `gettimeofday`. |
| 2 | `a21e4cc75272` | `fix(time): 짧은 대기 시간의 초과 지연 완화` | B | `TIME_MODEL, PRACTICAL` | Reduces final-interval polling granularity for short waits. |
| 3 | `5b32d5bdb955` | `fix(time): 단조 시계로 경과 시간 계산` | A | `TIME_MODEL, RISK, CORE` | Replaces wall time with `CLOCK_MONOTONIC`, widens time state, and makes clock failure fatal. |
| 4 | `f01d62cde8ce` | `test(time): 단조 시계와 시계 실패 경로 검증` | B | `TEST, TIME_MODEL` | Verifies the monotonic clock identifier, conversion, and failure exit. |
| 5 | `e7e62cbe185f` | `fix(thread): 시작 장벽으로 기준 시각 통일` | S | `START_BARRIER, CONCURRENCY, TIME_MODEL` | Adds a readiness barrier and publishes one start timestamp to all workers after they are actually ready. |
| 6 | `bfbfa0431732` | `test(thread): 지연된 작업자의 공통 시작 시각 검증` | A | `TEST, START_BARRIER, EDGE` | Deliberately delays one worker and verifies that the shared release prevents pre-start starvation accounting. |
| 7 | `f57f6ec0be87` | `test(thread): 시작 대기 실패 전파 검증` | B | `TEST, START_BARRIER, RESOURCE_LIFECYCLE` | Injects a condition-wait failure and checks that the barrier aborts and propagates the error. |

## `509453b01515` — 시간 접근을 한곳에 모으다

최초 helper는 `gettimeofday` 결과를 밀리초로 변환한다.

```c
/* 509453b01515, src/time.c */
long philo_now_ms(void)
{
    struct timeval tv;

    gettimeofday(&tv, NULL);
    return ((tv.tv_sec * 1000L) + (tv.tv_usec / 1000L));
}
```

`philo_sleep_ms`는 한 번의 긴 sleep 대신 deadline을 반복 확인한다. 매 반복에서 `ended`를 `state_mutex` 아래에서 읽으므로 simulation이 종료되면 예정된 대기 시간이 남아 있어도 빠져나올 수 있다.

```c
void philo_sleep_ms(t_table *table, long duration_ms)
{
    long deadline;
    int  ended;

    deadline = philo_now_ms() + duration_ms;
    while (philo_now_ms() < deadline)
    {
        pthread_mutex_lock(&table->state_mutex);
        ended = table->ended;
        pthread_mutex_unlock(&table->state_mutex);
        if (ended)
            break ;
        usleep(500);
    }
}
```

이 커밋이 해결한 것은 중복된 시간 변환과 종료를 무시하는 긴 block이다. 아직 다음 두 한계가 있다.

1. `gettimeofday`는 system wall clock이므로 NTP 보정이나 수동 시각 변경의 영향을 받는다.
2. 호출 반환값을 확인하지 않아 clock acquisition 실패를 정상 시각처럼 사용할 수 있다.

또한 `long`의 폭은 platform에 따라 다르다. elapsed-time state가 어떤 정수 폭을 요구하는지가 type에 드러나지 않는다.

## `a21e4cc75272` — deadline 직전의 polling 간격만 줄이다

변경은 작다. 남은 시간이 1 ms보다 크면 기존 500 µs, 1 ms 이하면 100 µs만 쉰다.

```c
remaining = deadline - philo_now_ms();
if (remaining > 1)
    usleep(500);
else
    usleep(100);
```

이 수정은 program이 스스로 추가하는 마지막 polling 지연을 줄인다. 하지만 다음을 보장하지 않는다.

- scheduler가 100 µs 뒤에 thread를 반드시 깨운다는 보장
- strict real-time deadline
- wall clock adjustment에 대한 안전성
- 여러 worker의 공통 시작 시각

따라서 이 커밋은 time model 자체의 교정이 아니라 실용적인 polling refinement다.

## `5b32d5bdb955` — 경과 시간을 wall clock에서 분리하다

### 단조 시계와 실패 정책

```c
/* 5b32d5bdb955, src/time.c */
static void clock_failure(void)
{
    static const char message[] =
        "Error: monotonic clock unavailable\n";

    (void)write(2, message, sizeof(message) - 1);
    _exit(PHILO_ERR);
}

int64_t philo_now_ms(void)
{
    struct timespec now;

    if (clock_gettime(CLOCK_MONOTONIC, &now) != 0)
        clock_failure();
    return (((int64_t)now.tv_sec * 1000)
        + (now.tv_nsec / 1000000));
}
```

`CLOCK_MONOTONIC`은 calendar time을 표현하지 않는다. 이 프로그램이 필요한 것은 “몇 시인가”가 아니라 start, last meal, deadline 사이에 얼마나 시간이 흘렀는가이므로 단조 시계가 맞다. system clock이 조정되어도 elapsed value가 뒤로 가지 않는다.

clock 실패는 임의의 0이나 이전 값으로 대체하지 않는다. 잘못 만든 시각은 모든 starvation 판정을 오염시키므로, `write`로 진단한 뒤 `_exit(PHILO_ERR)`한다.

### 시간 상태의 폭을 함께 바꾸다

이 커밋은 helper 반환형만 바꾸지 않는다.

| 위치 | 이전 | 이후 |
| --- | --- | --- |
| `time_to_die`, `time_to_eat`, `time_to_sleep` | `long` | `int64_t` |
| `t_philo.last_meal_ms` | `long` | `int64_t` |
| `t_table.start_ms` | `long` | `int64_t` |
| monitor의 `now` | `long` | `int64_t` |
| parser 내부 누산 | `long`/`LONG_MAX` | `int64_t`/`INT64_MAX` |
| log timestamp | `long`/`%ld` | `int64_t`/`%lld` cast |

입력 계약 자체는 완화하지 않는다. parser는 안전하게 `int64_t`로 읽되 시간 인자와 meal target은 기존 public upper bound인 `INT_MAX`를 계속 적용한다. 넓어진 type은 parsing과 runtime arithmetic의 정의된 범위를 명시하기 위한 것이지, CLI 범위를 무제한으로 늘리기 위한 것이 아니다.

### 이 커밋이 보장하는 것

- start와 last-meal의 차이는 같은 monotonic clock domain에서 계산된다.
- elapsed time이 wall-clock 보정 때문에 음수 또는 비정상적으로 커지지 않는다.
- clock 실패를 정상 simulation으로 계속 진행하지 않는다.

아직 보장하지 않는 것은 worker startup skew다. `start_ms`를 thread 생성 전에 정하면, 단조 시계여도 마지막 worker가 실제 routine에 들어오기까지 소비된 시간이 starvation budget에서 빠진다.

## `f01d62cde8ce` — 시계 기준과 실패 종료를 따로 검증하다

테스트는 `clock_gettime`을 wrapper로 치환한다.

```c
int test_clock_gettime(clockid_t clock_id, struct timespec *now)
{
    if (fail_clock)
        return (-1);
    if (clock_id == CLOCK_MONOTONIC)
        used_monotonic_clock = 1;
    now->tv_sec = 12;
    now->tv_nsec = 345000000L;
    return (0);
}
```

첫 assertion은 `12 s + 345 ms == 12345 ms`와 `CLOCK_MONOTONIC` 사용을 함께 확인한다. 실패 경로는 `_exit`가 test process 전체를 종료하므로 child process에서 실행하고, parent가 exit status `PHILO_ERR`를 확인한다.

이 테스트는 실제 OS clock이 조정되는 상황을 만들지는 않는다. production helper가 어떤 clock id를 요청하고, 실패를 fabricated time으로 바꾸지 않는다는 코드 경계를 검증한다.

## 시작 시각이 너무 일찍 게시되던 문제

`e7e62cbe185f` 직전 `philo_run`은 대략 다음 순서였다.

```text
start_ms = now
for each philosopher:
    philosopher.last_meal_ms = start_ms
    pthread_create(...)
monitor 시작
```

모든 philosopher가 같은 숫자를 받기는 하지만, 그 숫자는 첫 thread 생성 전의 시각이다. thread 생성과 scheduling이 지연되면 마지막 worker는 routine을 한 번도 실행하지 않았는데 이미 `time_to_die` 일부 또는 전부를 소비한다.

`pthread_create` 성공은 kernel thread object가 만들어졌다는 뜻일 뿐이다. worker가 routine에 진입해 shared state를 읽을 준비가 되었다는 뜻은 아니다.

## `e7e62cbe185f` — readiness를 확인한 뒤 하나의 epoch를 게시하다

### 시작 장벽 상태

`t_table`에 다음 상태가 추가된다.

```c
int             start_cond_ready;
int             start_released;
int             ready_count;
int             run_error;
pthread_cond_t  start_cond;
```

이 field들의 관찰과 변경은 `state_mutex` 안에서 이루어진다. `start_released`는 condition variable notification 자체가 아니라 worker가 진행해도 되는지를 나타내는 predicate다.

### worker 쪽: 준비 사실을 먼저 게시한다

```c
/* e7e62cbe185f, src/routine.c */
static int wait_for_start(t_philo *philo)
{
    t_table *table;
    int      ended;

    table = philo->table;
    pthread_mutex_lock(&table->state_mutex);
    table->ready_count++;
    pthread_cond_broadcast(&table->start_cond);
    while (!table->start_released)
    {
        if (pthread_cond_wait(&table->start_cond,
                &table->state_mutex) != 0)
        {
            table->run_error = 1;
            table->ended = 1;
            table->start_released = 1;
            pthread_cond_broadcast(&table->start_cond);
        }
    }
    ended = table->ended;
    pthread_mutex_unlock(&table->state_mutex);
    return (ended);
}
```

worker는 fork를 잡거나 log를 남기기 전에 `ready_count`를 증가시킨다. `while` predicate loop는 spurious wakeup이 와도 `start_released`가 거짓이면 다시 기다리게 한다.

condition wait 자체가 실패하면 한 worker만 return하는 것으로 끝내지 않는다. `run_error`, `ended`, `start_released`를 함께 게시하고 broadcast하여 다른 worker와 main-side waiter가 영원히 잠들지 않게 한다.

### `philo_run` 쪽: 모든 readiness 뒤에 시각을 정한다

```c
/* e7e62cbe185f, src/run.c */
static int release_start(t_table *table, int should_end)
{
    int     i;
    int     status;
    int64_t start_ms;

    status = PHILO_OK;
    pthread_mutex_lock(&table->state_mutex);
    while (!should_end
        && table->ready_count < table->config.number)
    {
        if (pthread_cond_wait(&table->start_cond,
                &table->state_mutex) != 0)
        {
            table->run_error = 1;
            should_end = 1;
            status = PHILO_ERR;
        }
    }
    if (table->run_error)
    {
        should_end = 1;
        status = PHILO_ERR;
    }
    start_ms = philo_now_ms();
    table->start_ms = start_ms;
    i = 0;
    while (i < table->config.number)
        table->philos[i++].last_meal_ms = start_ms;
    if (should_end)
        table->ended = 1;
    table->start_released = 1;
    pthread_cond_broadcast(&table->start_cond);
    pthread_mutex_unlock(&table->state_mutex);
    return (status);
}
```

정상 경로에서는 `ready_count == number`가 된 뒤 하나의 `philo_now_ms()` 결과를 table과 모든 philosopher에게 함께 기록한다. 그 다음에만 release predicate를 참으로 바꾸고 broadcast한다.

```text
worker 생성
  ↓
각 worker가 routine 진입
  ↓ state_mutex
ready_count++ → start_cond broadcast
  ↓
main이 ready_count == N 확인
  ↓ 같은 state_mutex 안에서
start_ms = now
all last_meal_ms = start_ms
start_released = 1
broadcast
  ↓
모든 worker가 실제 활동 시작
```

### 부분 생성과 장벽 실패도 release로 끝낸다

중간 `pthread_create` 실패에서는 `release_start(table, 1)`을 호출한다. 이 경로는 전체 `ready_count`를 기다리지 않고 `ended = 1`, `start_released = 1`을 게시한다. 이미 시작된 worker가 barrier에서 빠져나와 종료한 뒤 join될 수 있게 하기 위해서다.

main-side condition wait가 실패하거나 worker가 `run_error`를 게시한 경우도 같은 abort release로 수렴한다. failure path에서도 predicate를 열지 않으면 cleanup을 위해 join하려는 main과 barrier에 갇힌 worker가 서로 기다리는 deadlock이 된다.

## `bfbfa0431732` — 150 ms 늦은 worker가 80 ms 안에 죽지 않아야 한다

테스트는 `pthread_create`를 감싸 실제 routine 호출 전에 외부 gate를 둔다. 다섯 wrapper thread를 모두 만든 뒤 gate를 열고, 다섯 번째 worker만 추가로 150 ms 기다린 다음 `philo_routine`에 들어간다.

```c
g_starts[index].delay_us = (index == 4) * 150000;
```

simulation 설정은 다음과 같다.

```c
config.number = 5;
config.time_to_die = 80;
config.time_to_eat = 5;
config.time_to_sleep = 5;
config.must_eat = 1;
```

start가 thread 생성 전에 게시되는 이전 방식이라면 마지막 worker는 routine 진입 전 150 ms를 소비하므로 80 ms death threshold를 이미 넘는다. 장벽이 있으면 main은 마지막 worker가 `ready_count`를 올릴 때까지 기다리고, 그 뒤에 start epoch를 정한다.

테스트는 `philo_run == PHILO_OK`, `full_count == 5`, `ready_count == 5`를 요구한다. 단순히 death line이 없다는 것보다 모든 worker가 장벽에 실제 도착하고 목표 식사를 완료했다는 상태를 확인한다.

## `f57f6ec0be87` — wait 실패 시 다른 worker도 풀어 주다

`routine.c`만 `pthread_cond_wait=test_pthread_cond_wait`로 컴파일하고, 첫 worker-side wait 호출이 `EINVAL`을 반환하도록 한다. production code는 그 실패를 다음 상태로 바꾼다.

```text
run_error = 1
ended = 1
start_released = 1
broadcast
```

기대 결과는 `philo_run == PHILO_ERR`, `table.run_error != 0`이며, 전체 test는 5초 timeout 안에 끝나야 한다. 이 timeout은 status assertion만으로는 잡을 수 없는 barrier deadlock을 bounded failure로 만든다.

## 최종 시간/시작 불변 조건

| 관심사 | 최종 규칙 |
| --- | --- |
| elapsed-time source | `CLOCK_MONOTONIC`만 사용한다. |
| 시간 표현 | runtime timing state와 계산은 `int64_t`다. |
| clock failure | 오류를 출력하고 `_exit(PHILO_ERR)`한다. |
| worker readiness | routine에 들어와 `ready_count`를 게시해야 준비된 것이다. |
| start publication | 모든 worker가 준비된 뒤 table과 모든 `last_meal_ms`에 같은 값으로 기록한다. |
| barrier abort | create/wait 실패도 `start_released`와 broadcast를 게시해 peer를 깨운다. |

이 보장은 scheduler fairness나 starvation freedom을 뜻하지 않는다. release 뒤 어떤 worker가 먼저 fork를 얻는지, 100 µs polling이 정확히 그 시간 안에 깨어나는지, death가 threshold 직후 몇 µs 안에 출력되는지는 보장하지 않는다. 이 Thread가 확립한 것은 **모든 시간 비교가 같은 단조 clock domain에 있고, worker가 활동하기 전부터 서로 다른 양의 starvation debt를 떠안지 않는다는 것**이다.

> 조사 범위: 표시된 exact SHA의 GitHub diff와 해당 SHA의 source/test를 확인했다. 이 환경에서는 branch를 checkout하여 build 또는 test suite를 실행하지 않았다.
