# Thread: Layered evidence for concurrent behavior

> Project: `philo` · Branch: `c/philo` · Development Thread 5/5

이 Thread는 새로운 synchronization mechanism을 만드는 history가 아니다. 이미 구현된 CLI, progress, terminal logging, shared-state access를 **서로 다른 관찰 기법으로 검증하는 층**을 만든다.

동시성 테스트에서 “몇 번 실행해도 통과했다”와 “모든 schedule에서 옳다”는 같은 말이 아니다. 이 Thread의 가치도 완전한 증명에 있지 않다. timeout, output grammar, 반복 workload, 의도적 overlap, dynamic race detector가 서로 다른 failure class를 맡고, 각 layer가 놓치는 부분을 다른 layer가 보완하도록 만든 데 있다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `bd6bb8eb18f4` | `test(smoke): 주요 입력과 종료 조건 검증` | B | `TEST, CLI_CONTRACT, CORE` | Adds bounded public smoke cases for input, death, and finite completion. |
| 2 | `f145d33f2773` | `test(format): 필수 상태 로그 형식 검증` | B | `TEST, TERMINAL_STATE` | Treats the five-line status grammar as an executable output contract. |
| 3 | `3d24bea01441` | `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증` | A | `TEST, CONCURRENCY, TERMINAL_STATE` | Repeats progress and death schedules and adds a gated logger-versus-death race harness. |
| 4 | `20f8270c78bb` | `test(tsan): ThreadSanitizer 검증 경로 추가` | A | `TEST, CONCURRENCY, PRACTICAL` | Adds capability-probed ThreadSanitizer workloads while retaining semantic log and progress assertions. |

## 첫 번째 층: 공개 실행 파일의 smoke 계약

### `bd6bb8eb18f4`

`make test`가 `tests/smoke.sh`를 실행하도록 추가된다. test harness는 temporary directory를 만들고 trap으로 정리하며, 종료하지 않을 수 있는 simulation은 별도 process로 실행한 뒤 guard process가 제한 시간 뒤 `TERM`을 보낸다.

```sh
run_timeout()
{
    limit=$1
    outfile=$2
    shift 2
    "$@" >"$outfile" 2>&1 &
    pid=$!
    (
        sleep "$limit"
        kill -TERM "$pid" 2>/dev/null || true
    ) &
    guard=$!
    set +e
    wait "$pid"
    status=$?
    set -e
    kill "$guard" 2>/dev/null || true
    wait "$guard" 2>/dev/null || true
    return "$status"
}
```

이 helper는 hang을 무한 대기가 아니라 nonzero test result로 바꾼다. 내부 mutex나 state field를 읽지 않고 public executable의 exit와 output만 관찰한다.

### 실행 조건과 검증

| case | command 핵심 인자 | 시간 제한 | 확인하는 결과 |
| --- | --- | ---: | --- |
| invalid count | `0 100 10 10` | 즉시 | success가 아니며 usage가 출력됨 |
| numeric overflow | 매우 긴 `time_to_die` | 즉시 | parsing success가 아님 |
| single philosopher | `1 80 40 40` | 2 s | fork 한 번과 death line이 존재 |
| finite two-worker | `2 250 50 50 2` | 3 s | death 없음, `is eating` 최소 4회 |
| finite five-worker | `5 800 100 100 3` | 5 s | death 없음, `is eating` 최소 15회 |

meal log는 exact total이 아니라 minimum total을 검사한다.

```sh
eat_count=$(grep -c 'is eating' "$finite_out" || true)
[ "$eat_count" -ge 4 ] \
    || fail 'finite meal run did not eat enough'
```

목표에 도달한 worker가 global completion publication 전에 한 번 더 eating을 시작할 수 있고, scheduler에 따라 관찰되는 추가 line 수가 달라질 수 있기 때문이다. test가 요구하는 semantic minimum만 고정하고 우연한 schedule 순서를 golden output으로 만들지 않는다.

### 이 층이 잡는 것과 놓치는 것

잡을 수 있는 것:

- CLI validation 회귀
- single-philosopher self-deadlock 또는 death 누락
- finite run hang
- global progress 부족
- finite target에서 잘못된 death

단독으로 잡기 어려운 것:

- 특정 philosopher만 굶고 다른 philosopher가 extra meal로 total count를 채운 경우
- timestamp 감소
- malformed/interleaved line
- death 뒤 ordinary line
- data race가 발생했지만 visible output은 우연히 정상인 경우

## 두 번째 층: 출력을 부분 문자열이 아니라 문법으로 검사하다

### `f145d33f2773`

smoke output마다 다음 `awk` validator를 실행한다.

```awk
/^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$/ {
    next
}
{ bad = 1 }
END { exit bad }
```

허용되는 line은 정확히 세 부분이다.

```text
<0 이상의 정수 timestamp> <1 이상의 philosopher id> <허용된 phrase>
```

허용 phrase는 다음 다섯 개뿐이다.

- `has taken a fork`
- `is eating`
- `is sleeping`
- `is thinking`
- `died`

단순 `grep 'is eating'`은 앞뒤에 garbage가 붙거나 두 thread 출력이 한 줄에 섞여도 substring을 찾을 수 있다. full-line grammar는 field 누락, 잘못된 id, 음수 또는 비정수 timestamp, 허용되지 않은 phrase, line interleaving을 더 엄격하게 거부한다.

이 validator는 line syntax만 본다. timestamp 순서, philosopher별 progress, terminal line 위치는 다음 layer가 담당한다.

## 세 번째 층: schedule에 민감한 불변 조건을 반복 관찰하다

### `3d24bea01441`

`tests/concurrency.sh`는 세 종류의 validator를 조합한다.

### 1. 문법과 감소하지 않는 timestamp

```awk
/^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$/ {
    if (seen && $1 < previous) bad = 1
    previous = $1
    seen = 1
    next
}
{ bad = 1 }
END { exit bad }
```

동일한 millisecond timestamp는 허용하지만, 이전 line보다 작은 timestamp는 거부한다. 이 check는 monotonic clock과 serialized output이 실제 관찰 결과에서 함께 유지되는지 본다.

### 2. philosopher별 progress

```awk
$3 == "is" && $4 == "eating" { meals[$2]++ }
END {
    for (id = 1; id <= count; id++)
        if (meals[id] < target) exit 1
}
```

전체 eating 수만 세지 않고 id별 count를 만든다. 한 philosopher가 많이 먹어 total threshold를 채워도 다른 philosopher가 target에 못 미치면 실패한다.

### 3. terminal line 위치

```awk
{
    if (terminal) after = 1
    if ($3 == "died") {
        deaths++
        terminal = 1
    }
}
END { exit after || deaths != 1 }
```

정확히 하나의 death line이 있어야 하고, 그 뒤에는 line이 하나도 없어야 한다.

### 실행 조건 표

| 종류 | 설정 | 반복 | 주요 assertion |
| --- | --- | ---: | --- |
| 규모별 finite | philosopher `2`, `5`, `17`; `2000 5 5 4` | 각 1회 | timeout 내 종료, death 없음, 모든 id가 4회 이상 eating |
| repeated finite | `7 1000 4 4 3` | 8회 | 매 run마다 grammar/timestamp/progress 유지 |
| repeated death | `5 60 80 10` | 10회 | 매 run마다 death 정확히 한 번, 마지막 line |

여러 번 반복하는 이유는 schedule에 따라 드러나는 증상의 표본을 늘리기 위해서다. 8회 또는 10회 성공은 가능한 모든 interleaving을 열거한 proof가 아니다.

## 일반 실행에서 드물게 겹치는 구간을 직접 확대하다

같은 커밋의 `tests/log_terminal_race.c`는 production `philo_log`와 `philo_try_log_death`를 직접 호출한다.

```c
#define LOGGER_COUNT 12
#define LOGS_PER_LOGGER 200
```

12개 logger thread는 gate 뒤에서 모두 준비될 때까지 기다린다.

```c
pthread_mutex_lock(&g_gate_mutex);
g_ready++;
pthread_cond_broadcast(&g_gate_cond);
while (!g_go)
    pthread_cond_wait(&g_gate_cond, &g_gate_mutex);
pthread_mutex_unlock(&g_gate_mutex);
```

main은 모든 logger가 준비된 뒤 한꺼번에 release하고, 이미 death threshold를 넘긴 philosopher에 대해 `philo_try_log_death`를 호출한다. 각 logger는 최대 200번 `is thinking`을 시도한다.

```text
12 logger가 같은 gate에서 release
  ↘ philo_log(... "is thinking") 반복
main → philo_try_log_death(...)
```

이 harness의 목표는 자연 실행에서 우연히 겹치기를 기다리는 것이 아니라 `print_mutex → state_mutex` terminal boundary에 contention을 집중시키는 것이다. 최종 output에는 grammar가 맞는 line만 있어야 하며, death는 정확히 한 번이고 마지막 line이어야 한다.

이 테스트는 production monitor 전체를 통과하지 않는다. 대신 logger와 terminal commit 함수를 좁은 경계에서 직접 stress한다. broad simulation과 focused overlap harness는 서로 대체 관계가 아니다.

## 네 번째 층: 실제로 실행된 메모리 접근을 race detector로 관찰하다

### `20f8270c78bb`

Makefile은 일반 test와 분리된 target을 제공한다.

```make
TSAN_CC ?= $(CC)
TSAN_REQUIRED ?= 0

test-tsan:
	TSAN_CC="$(TSAN_CC)" TSAN_REQUIRED="$(TSAN_REQUIRED)" \
		./tests/tsan.sh
```

ThreadSanitizer는 compiler와 runtime 지원 여부가 platform마다 다르다. project를 바로 instrument한 뒤 실패하면 다음을 구분하기 어렵다.

- compiler가 `-fsanitize=thread`를 모름
- runtime이 현재 환경에서 시작되지 않음
- project 자체가 TSAN build에서 깨짐
- project에서 실제 race가 검출됨

### 지원 여부 probe를 먼저 실행한다

script는 작은 pthread program을 만들고 TSAN으로 compile한 뒤 실제 실행한다. compile 실패, executable 미생성, runtime nonzero, sanitizer runtime diagnostic을 project 결과와 분리한다.

```sh
if ! "$TSAN_CC" ... -fsanitize=thread ... probe.c; then
    skip "$TSAN_CC cannot build a ThreadSanitizer probe"
fi

TSAN_OPTIONS='halt_on_error=1:exitcode=66' \
    run_timeout 10 ... "$TMP_DIR/tsan-probe"
```

`skip`의 의미는 설정에 따라 달라진다.

```sh
skip()
{
    printf 'tsan: skipped (%s)\n' "$1" >&2
    if [ "$TSAN_REQUIRED" -eq 1 ]; then
        exit 1
    fi
    exit 77
}
```

- `TSAN_REQUIRED=0`: 지원되지 않는 환경은 status 77로 명시적 skip
- `TSAN_REQUIRED=1`: 같은 환경 제약도 CI requirement 위반이므로 failure

status 77은 race가 없다는 뜻이 아니다. detector를 신뢰할 수 있게 실행하지 못했다는 뜻이다.

### probe 통과 뒤 프로젝트 실패는 skip하지 않는다

capability probe가 성공한 뒤 production source 전체를 같은 sanitizer option으로 compile한다.

```sh
"$ROOT_DIR/src/init.c"
"$ROOT_DIR/src/main.c"
"$ROOT_DIR/src/monitor.c"
"$ROOT_DIR/src/parse.c"
"$ROOT_DIR/src/routine.c"
"$ROOT_DIR/src/run.c"
"$ROOT_DIR/src/state.c"
"$ROOT_DIR/src/time.c"
```

이 build가 실패하면 infrastructure limitation으로 숨기지 않고 project build failure로 처리한다. capability가 이미 확인되었기 때문이다.

### TSAN 진단 부재뿐 아니라 실제 동작 수행도 요구한다

instrumented binary는 세 workload를 실행한다.

| 이름 | 설정 | semantic assertion |
| --- | --- | --- |
| finite | `7 1000 5 5 4` | grammar/timestamp, death 없음, 각 id 4회 이상 eating |
| death | `5 60 80 10` | grammar/timestamp, death 정확히 하나이며 마지막 line |
| contention | `17 2000 5 5 3` | grammar/timestamp, death 없음, 각 id 3회 이상 eating |

각 실행에는 다음 option이 적용된다.

```sh
TSAN_OPTIONS='halt_on_error=1:exitcode=66'
```

race가 관찰되면 detector가 즉시 멈추고 66을 반환한다. stderr에 `ThreadSanitizer` 문자열이 남아도 실패한다.

하지만 diagnostic 부재만으로 test를 통과시키지 않는다. program이 startup 초기에 잘못 종료해 shared access를 거의 수행하지 않았다면 race report가 없어도 무의미하다. 그래서 log grammar와 philosopher별 progress 또는 terminal semantics를 다시 검사한다.

```text
TSAN report 없음
    + required workload 실제 수행
    + expected terminal/progress semantics
    = 이 schedule에서 의미 있는 dynamic evidence
```

## 각 검증 층이 맡는 실패 유형

| layer | 직접 관찰하는 것 | 단독으로 증명하지 않는 것 |
| --- | --- | --- |
| smoke + timeout | public CLI, bounded termination, 최소 global work | id별 progress, race, exact terminal order |
| format grammar | complete line syntax와 허용 phrase | event order와 shared-state correctness |
| repeated concurrency | id별 progress, timestamp 비감소, one terminal line | 모든 schedule, formal fairness/deadlock freedom |
| focused logger race | logger/death lock boundary의 높은 overlap | 전체 routine/monitor integration |
| TSAN | 실행된 memory access의 data-race diagnostic | 실행되지 않은 path, semantic correctness, unsupported 환경 |
| TSAN + semantic assertions | detector가 실제 required work를 관찰했는지 | exhaustive race freedom |

## 최종 해석

이 test architecture가 제공하는 evidence는 누적적이다.

```text
public behavior가 끝나는가
  → output이 contract grammar를 따르는가
  → 각 philosopher가 실제 progress하는가
  → terminal event가 하나이고 마지막인가
  → logger/terminal boundary를 강제로 겹쳐도 유지되는가
  → instrumented workload에서 data race가 보고되지 않는가
```

어느 한 단계도 다음 주장을 허용하지 않는다.

- 모든 가능한 thread schedule에서 race가 없다.
- philosopher starvation이 이론적으로 불가능하다.
- lock ordering이 formal deadlock-freedom proof를 제공한다.
- TSAN이 지원되지 않아 skip된 환경에서도 race freedom이 확인되었다.
- finite 반복 횟수 이상으로 무기한 실행해도 같은 결과가 보장된다.

이 Thread가 확립한 것은 더 실용적인 원칙이다. **동시성 correctness를 한 종류의 성공 신호로 축약하지 않고, syntax·progress·terminal order·의도적 contention·dynamic memory observation을 서로 다른 증거로 유지한다.**

> 조사 범위: 표시된 exact SHA의 GitHub diff와 해당 SHA의 source/test를 확인했다. 이 환경에서는 branch를 checkout하여 build, concurrency suite 또는 TSAN target을 실행하지 않았다.
