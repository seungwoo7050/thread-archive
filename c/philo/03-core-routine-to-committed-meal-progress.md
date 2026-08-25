# Thread: Core routine to committed and range-safe meal progress

> Project: `philo` · Branch: `c/philo` · Development Thread 3/5

이 Thread는 최초의 eat-sleep-think routine이 다음 네 문제를 차례로 드러내는 과정이다.

1. fork 두 개를 어떤 순서로 잠글 것인가
2. philosopher가 한 명일 때 같은 mutex를 두 번 잠그지 않으려면 어떻게 할 것인가
3. “식사를 시도했다”와 “식사를 끝까지 완료했다”를 어디서 구분할 것인가
4. public meal target이 `INT_MAX`여도 internal counter가 정의된 범위에 남으려면 어떤 type이 필요한가

최종 불변 조건은 다음과 같다.

> 두 fork를 획득하고 `is eating`을 출력한 사실만으로 meal progress를 기록하지 않는다. eating deadline을 끝까지 통과하고, `state_mutex` 아래의 commit 지점에서 simulation이 여전히 active일 때만 `meals`와 `full_count`를 변경한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b68f40819af4` | `feat(routine): 철학자의 식사·수면·사고 흐름 구현` | S | `CORE, CONCURRENCY, FORK_ORDER` | Introduces the eat-sleep-think worker and parity-dependent fork order. |
| 2 | `c8531c91f0fb` | `fix(single): 철학자가 한 명일 때 포크 재잠금 방지` | A | `FORK_ORDER, EDGE, RISK` | Handles the ring aliasing edge where one philosopher's two fork pointers are the same mutex. |
| 3 | `fe0a2d15b29b` | `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단` | A | `MEAL_ACCOUNTING, TERMINAL_STATE, RISK` | Commits global completion in the final meal critical section and prevents post-completion loop states. |
| 4 | `53e591effb4a` | `fix(routine): 중단된 식사를 완료 횟수에서 제외` | A | `MEAL_ACCOUNTING, TERMINAL_STATE, RISK` | Separates an eating attempt from a committed meal and rejects progress after interruption or terminal state. |
| 5 | `73b5551a76f4` | `test(routine): 중단된 식사의 카운터 불변식 검증` | B | `TEST, MEAL_ACCOUNTING` | Injects an interrupted eating wait and verifies that neither local nor global counters advance. |
| 6 | `4c224ae86f2b` | `fix(state): 식사 완료 횟수의 정수 범위 확장` | A | `MEAL_ACCOUNTING, EDGE, RISK` | Widens accumulated meals so a valid `INT_MAX` target can be exceeded without signed overflow. |
| 7 | `054ef46f80c7` | `test(routine): 최대 목표 이후 식사 카운터 검증` | B | `TEST, MEAL_ACCOUNTING, EDGE` | Verifies `INT_MAX + 1` and confirms the philosopher does not contribute to `full_count` twice. |

## `b68f40819af4` — fork 관계 위에 최초 worker loop를 만들다

### parity에 따라 첫 lock을 바꾼다

원형 table에서 philosopher마다 두 fork를 가리키지만, 이 fork는 이웃과 공유된다. 모든 worker가 동시에 왼쪽 fork를 먼저 잡으면 각자 하나를 보유한 채 오른쪽을 기다리는 circular wait가 가능하다.

최초 routine은 id parity로 lock 순서를 반대로 배치한다.

```c
/* b68f40819af4, src/routine.c */
static void lock_forks(t_philo *philo)
{
    if (philo->id % 2 == 0)
    {
        pthread_mutex_lock(philo->right_fork);
        philo_log(philo, "has taken a fork");
        pthread_mutex_lock(philo->left_fork);
        philo_log(philo, "has taken a fork");
    }
    else
    {
        pthread_mutex_lock(philo->left_fork);
        philo_log(philo, "has taken a fork");
        pthread_mutex_lock(philo->right_fork);
        philo_log(philo, "has taken a fork");
    }
}
```

| philosopher id | 첫 번째 fork | 두 번째 fork |
| --- | --- | --- |
| 홀수 | left | right |
| 짝수 | right | left |

이 선택은 모든 worker가 같은 방향으로 하나씩 점유하는 lock cycle을 깨는 장치다. 이후 짝수 philosopher에게 주는 1 ms initial delay는 동시 경합을 줄이기 위한 scheduling hint다.

```c
if (philo->id % 2 == 0)
    philo_sleep_ms(philo->table, 1);
```

두 결정을 같은 보장으로 해석하면 안 된다. parity order는 circular-wait 구조를 피하기 위한 lock ordering이고, 1 ms stagger는 특정 시점의 contention을 줄이는 실용적 조정이다. 어느 쪽도 philosopher별 공정한 fork 획득이나 starvation freedom을 증명하지 않는다.

### 최초 식사 상태 전이

```c
static void record_meal_start(t_philo *philo)
{
    pthread_mutex_lock(&philo->table->state_mutex);
    philo->last_meal_ms = philo_now_ms();
    pthread_mutex_unlock(&philo->table->state_mutex);
}

static void record_meal_done(t_philo *philo)
{
    t_table *table;

    table = philo->table;
    pthread_mutex_lock(&table->state_mutex);
    philo->meals++;
    if (table->config.has_meal_limit
        && philo->meals == table->config.must_eat)
        table->full_count++;
    pthread_mutex_unlock(&table->state_mutex);
}
```

```c
static void eat_once(t_philo *philo)
{
    lock_forks(philo);
    record_meal_start(philo);
    philo_log(philo, "is eating");
    philo_sleep_ms(philo->table,
        philo->table->config.time_to_eat);
    record_meal_done(philo);
    unlock_forks(philo);
}
```

`last_meal_ms`는 eating 시작 시점에 갱신되고, `meals`는 sleep helper가 돌아온 뒤 증가한다. 하지만 이 SHA의 `philo_sleep_ms`는 `void`다. deadline에 도달해서 반환했는지, simulation의 `ended`를 보고 중간에 빠져나왔는지 호출자가 구분할 수 없다.

따라서 최초 구현에는 다음 잘못된 경로가 존재한다.

```text
두 fork 획득
  → last_meal_ms 갱신
  → "is eating" 출력
  → eating 도중 terminal state 발생
  → sleep helper가 조기 반환
  → record_meal_done 실행
  → 완료하지 않은 식사가 meals/full_count에 반영됨
```

이 결함은 routine 자체는 완성했지만 meal의 commit 지점은 아직 정의하지 않았음을 보여준다.

## `c8531c91f0fb` — 한 명의 philosopher는 두 fork를 갖지 않는다

fork pointer는 다음처럼 배치되어 있다.

```c
left_fork  = &table->forks[i];
right_fork = &table->forks[(i + 1) % table->config.number];
```

`N == 1`이면 `(0 + 1) % 1 == 0`이므로 두 pointer가 같은 mutex를 가리킨다. 일반 routine은 첫 lock 성공 후 같은 non-recursive mutex를 다시 잠그려 하며 self-deadlock한다.

수정은 이 경우를 일반 two-fork protocol에 억지로 넣지 않는 것이다.

```c
/* c8531c91f0fb */
static void wait_single_philo(t_philo *philo)
{
    pthread_mutex_lock(philo->left_fork);
    philo_log(philo, "has taken a fork");
    philo_sleep_ms(philo->table,
        philo->table->config.time_to_die + 1);
    pthread_mutex_unlock(philo->left_fork);
}

if (philo->table->config.number == 1)
{
    wait_single_philo(philo);
    return (NULL);
}
```

한 fork만 획득하고 한 번만 로그를 남긴 뒤 monitor가 death를 게시할 때까지 기다린다. fork 하나로는 먹을 수 없으므로 meal start나 meal counter는 건드리지 않는다.

이것은 “lock 순서를 바꾸면 해결된다”는 문제가 아니다. 두 논리 역할이 같은 mutex identity로 aliasing되는 data-model edge이므로 전용 경로가 필요하다.

## `fe0a2d15b29b` — 마지막 목표 식사가 전체 종료 상태를 직접 게시하다

최초 monitor는 `full_count`를 보고 별도로 `ended`를 설정한다. 마지막 philosopher가 목표 식사를 기록한 뒤 monitor가 이를 관찰하기 전까지 해당 worker가 sleep/thinking 단계로 넘어갈 수 있다.

수정은 `meals`, `full_count`, `ended`를 같은 `state_mutex` section 안에서 결정하는 것이다.

```c
/* fe0a2d15b29b */
philo->meals++;
if (table->config.has_meal_limit
    && philo->meals == table->config.must_eat)
    table->full_count++;
if (table->config.has_meal_limit
    && table->full_count >= table->config.number)
    table->ended = 1;
```

`full_count`가 마지막 philosopher의 기여로 `number`에 도달하는 순간 global completion도 함께 게시된다. routine은 eating 뒤 terminal state를 다시 확인하고 sleep/thinking을 시작하지 않는다.

```c
if (eat_once(philo) != PHILO_OK)
    break ;
if (philo_has_ended(philo->table))
    break ;
```

fork 두 개를 획득한 뒤에도 terminal을 다시 확인한다.

```c
lock_forks(philo);
if (philo_has_ended(philo->table))
{
    unlock_forks(philo);
    return (PHILO_ERR);
}
```

outer loop의 이전 check와 실제 fork 획득 사이에 다른 thread가 simulation을 끝낼 수 있기 때문이다. 이미 잡은 fork는 이 branch에서 반드시 해제된다.

하지만 이 커밋도 `philo_sleep_ms` 반환 이유를 알지 못한다. eating 도중 death/completion이 발생해 helper가 조기 반환해도 `record_meal_done`은 실행된다. 즉 global completion publication 위치는 교정했지만, 무엇을 completed meal로 인정할지는 아직 불완전하다.

## `53e591effb4a` — 식사 시도에 명시적인 commit 지점을 만들다

이 fix는 두 단계로 이루어진다.

### 1. sleep 결과를 완료와 중단으로 구분한다

```c
/* 53e591effb4a, src/time.c */
int philo_sleep_ms(t_table *table, int64_t duration_ms)
{
    int64_t deadline;
    int64_t now;
    int     ended;

    deadline = philo_now_ms() + duration_ms;
    while (1)
    {
        now = philo_now_ms();
        if (now >= deadline)
            return (PHILO_OK);
        pthread_mutex_lock(&table->state_mutex);
        ended = table->ended;
        pthread_mutex_unlock(&table->state_mutex);
        if (ended)
            return (PHILO_ERR);
        /* 남은 시간에 따라 500 µs 또는 100 µs polling */
    }
}
```

`PHILO_OK`는 requested interval이 끝까지 지났다는 뜻이고, `PHILO_ERR`는 terminal state 때문에 중단되었다는 뜻이다.

deadline check가 terminal check보다 먼저이므로 같은 iteration에서 둘 다 참일 수 있다. 이 경우 helper는 완료를 반환할 수 있지만, 다음 단계의 locked recheck가 terminal 뒤 counter mutation을 다시 막는다. 두 검사가 서로 다른 race window를 닫는다.

### 2. 카운터 변경 직전에 종료 상태를 다시 확인한다

```c
/* 53e591effb4a, src/routine.c */
static int record_meal_done(t_philo *philo)
{
    t_table *table;

    table = philo->table;
    pthread_mutex_lock(&table->state_mutex);
    if (table->ended)
    {
        pthread_mutex_unlock(&table->state_mutex);
        return (PHILO_ERR);
    }
    philo->meals++;
    if (table->config.has_meal_limit
        && philo->meals == table->config.must_eat)
        table->full_count++;
    if (table->config.has_meal_limit
        && table->full_count >= table->config.number)
        table->ended = 1;
    pthread_mutex_unlock(&table->state_mutex);
    return (PHILO_OK);
}
```

sleep이 성공한 직후와 `state_mutex`를 획득하기 직전 사이에도 다른 thread가 terminal state를 게시할 수 있다. 그래서 helper success만으로 counter를 올리지 않고, mutation과 같은 lock 안에서 `ended`를 다시 본다.

### 모든 중단 경로에서 fork를 해제한다

```c
static int eat_once(t_philo *philo)
{
    lock_forks(philo);
    if (philo_has_ended(philo->table))
    {
        unlock_forks(philo);
        return (PHILO_ERR);
    }
    record_meal_start(philo);
    philo_log(philo, "is eating");
    if (philo_sleep_ms(philo->table,
            philo->table->config.time_to_eat) != PHILO_OK
        || record_meal_done(philo) != PHILO_OK)
    {
        unlock_forks(philo);
        return (PHILO_ERR);
    }
    unlock_forks(philo);
    return (PHILO_OK);
}
```

최종 meal transaction은 다음처럼 읽을 수 있다.

```text
[fork 두 개 획득]
  ↓ terminal recheck
[last_meal_ms 갱신 + "is eating"]
  ↓
[eating deadline 완료?]
  ├─ 아니오: fork 해제, counter 변화 없음
  └─ 예
       ↓ state_mutex 아래 terminal recheck
     [meals++ / 최초 target 도달이면 full_count++
      / 모든 target 도달이면 ended=1]
       ↓
     fork 해제
```

`is eating`은 operation이 시작되었다는 observable event다. `meals++`는 operation이 commit되었다는 internal event다. 둘은 의도적으로 같은 사건이 아니다.

## `73b5551a76f4` — 중단된 식사를 scheduler 의존 없이 재현하다

테스트는 routine을 다음 fake sleep과 연결한다.

```c
int test_philo_sleep_ms(t_table *table, int64_t duration_ms)
{
    (void)duration_ms;
    pthread_mutex_lock(&table->state_mutex);
    table->ended = 1;
    pthread_mutex_unlock(&table->state_mutex);
    interrupted = 1;
    return (PHILO_ERR);
}
```

한 philosopher의 routine을 직접 호출한 뒤 다음을 요구한다.

```c
interrupted != 0
table.philos[0].meals == 0
table.full_count == 0
```

실제 timing race가 우연히 발생하기를 기다리지 않고, eating wait가 반환하는 정확한 경계에서 terminal state를 만든다. 이 테스트는 해당 중단 경로의 counter 불변 조건을 증명하지만, 모든 scheduler interleaving이나 fork fairness를 증명하지 않는다.

## 왜 `must_eat == INT_MAX`인데 `meals == INT_MAX + 1`이 가능한가

`must_eat`은 public target이며 `int` 범위 안으로 제한된다. 그러나 한 philosopher가 target에 도달한 순간 simulation이 항상 즉시 멈추는 것은 아니다. 다른 philosopher가 아직 target에 도달하지 않았다면 `full_count < number`이고, 먼저 목표를 채운 philosopher가 다시 한 번 먹을 수 있다.

기존 `int meals`에서 다음 진행은 signed overflow다.

```text
meals == INT_MAX
  → 다른 philosopher가 아직 target 미달
  → 현재 philosopher가 유효한 식사 한 번 더 완료
  → meals++  // int 범위 초과
```

### `4c224ae86f2b`

수정은 field 하나를 넓힌다.

```diff
-int     meals;
+int64_t meals;
```

작은 diff지만 목적은 target type을 바꾸는 것이 아니라 **target을 넘어 누적될 수 있는 internal state의 정의된 범위**를 확보하는 것이다.

`full_count` 기여는 여전히 equality에서만 발생한다.

```c
if (has_meal_limit && philo->meals == must_eat)
    table->full_count++;
```

따라서 `INT_MAX + 1` 식사는 counter를 계속 전진시키되 같은 philosopher가 두 번째로 `full_count`에 기여하지 않는다.

### `054ef46f80c7`

테스트는 긴 loop를 실제로 `INT_MAX`번 실행하지 않는다. state를 boundary 직전으로 직접 구성한다.

```c
config.must_eat = INT_MAX;
table.full_count = 1;
table.philos[0].meals = INT_MAX;
```

첫 fake eating sleep은 성공하고 다음 sleep에서 `ended`를 설정해 routine을 멈춘다. 기대 결과는 다음과 같다.

```c
table.philos[0].meals == (int64_t)INT_MAX + 1
table.full_count == 1
```

이 테스트는 두 가지를 동시에 확인한다.

- internal counter가 signed overflow 없이 target 이후에도 증가한다.
- 이미 target에 도달한 philosopher가 `full_count`에 중복 기여하지 않는다.

## 최종 불변 조건과 비보장 범위

| 항목 | 최종 규칙 |
| --- | --- |
| fork identity | 이웃은 같은 fork mutex 객체를 공유한다. |
| `N >= 2` lock order | 홀수는 left→right, 짝수는 right→left다. |
| `N == 1` | 같은 mutex를 두 번 잠그지 않고 한 fork만 보유한다. |
| meal start | fork 획득 뒤 terminal을 확인하고 `last_meal_ms`를 갱신한다. |
| meal commit | eating deadline 완료 + locked terminal recheck 뒤에만 counter를 변경한다. |
| `full_count` | 각 philosopher가 target과 정확히 같아지는 순간 한 번만 증가한다. |
| numeric range | public target은 `int`, 누적 `meals`는 `int64_t`다. |
| abort cleanup | eating 중단 또는 commit 거부 시 두 fork를 모두 해제한다. |

parity lock order와 반복 테스트는 fairness나 starvation freedom을 보장하지 않는다. 또한 log 출력 성공 여부가 meal commit을 결정하지 않는다. 이 Thread가 확립한 것은 더 좁고 명확하다. **관찰 가능한 eating 시작과 완료된 progress를 분리하고, 완료로 인정되는 한 지점에서만 shared counters를 변경한다.**

> 조사 범위: 표시된 exact SHA의 GitHub diff와 해당 SHA의 source/test를 확인했다. 이 환경에서는 branch를 checkout하여 build 또는 test suite를 실행하지 않았다.
