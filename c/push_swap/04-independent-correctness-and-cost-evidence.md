# Thread: 독립적 정확성 검증과 비용 증거

정렬 프로그램이 정상 종료하고 자체 checker가 `OK`를 출력하는 것만으로는 충분하지 않습니다. generator와 checker가 같은 C operation layer를 공유하므로, 같은 방향 오류나 같은 상태 표현 오류를 함께 가지고 있으면 서로를 통과시킬 수 있습니다.

이 Thread는 검증 근거를 서로 다른 층으로 나눕니다.

1. Python 목록 모델로 명령열을 독립 재생합니다.
2. 작은 입력은 모든 permutation을 검사합니다.
3. 큰 입력은 재현 가능한 여러 seed와 크기로 correctness와 deterministic output을 확인합니다.
4. 명령 수뿐 아니라 배열 이동량·peak allocation·live allocation을 계측합니다.
5. 같은 functional suite를 ASan/UBSan binary에서도 실행할 경로를 만듭니다.

각 층은 다른 종류의 회귀를 잡으며, 어느 하나도 나머지를 대체하지 않습니다.

> **검사 범위**: 아래 내용은 `c/push_swap` 브랜치의 exact commit diff, Python/C test source, resource baseline, Makefile을 기준으로 작성했습니다. 테스트 command는 현재 환경에서 실행하지 않았으므로 기록된 수치는 repository의 baseline과 assertion이며 이번 작업의 실측 결과가 아닙니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `5b7559278909` | `test(sort): 생성 명령의 정렬 결과를 독립 검증` | A | `TEST, SORT, RISK` | Python reference model로 명령을 재생하고 작은 입력을 exhaustive하게 검증 |
| 2 | `a16dde75d935` | `test(sort): 큰 입력의 명령 수 상한을 검증` | B | `TEST, PERF, SORT` | 100·500개 입력에서 correctness 뒤 명령 수 ceiling을 확인 |
| 3 | `23198a9cdd55` | `test(sort): 결정적 다중 시드 동치 검사를 추가` | B | `TEST, SORT` | 명시적 PRNG·shuffle로 fixture를 고정하고 같은 입력의 명령열 재현성을 검사 |
| 4 | `6569949742eb` | `test(resource): 명령과 배열 이동 및 할당량을 기준화` | A | `TEST, RESOURCE, PERF` | 출력 명령 수, pair 이동량, peak bytes, live allocation을 별도 metric으로 고정 |
| 5 | `5505adf3e469` | `build(sanitize): C99 sanitizer 검증 경로를 추가` | B | `TEST, RUNTIME, PRACTICAL` | 일반 binary와 분리된 ASan/UBSan build로 operation·functional suite를 재실행 |

## `5b7559278909` — product checker와 다른 방식으로 명령을 재생

이 커밋은 stdout을 단순히 checker에 전달하는 기존 검사 위에 Python reference interpreter를 추가합니다.

### 먼저 명령 언어 자체를 제한한다

```python
VALID_MOVES = {
    "sa", "sb", "ss", "pa", "pb",
    "ra", "rb", "rr", "rra", "rrb", "rrr",
}


def split_moves(output):
    if output == "":
        return []
    moves = output.splitlines()
    for move in moves:
        assert_ok(move in VALID_MOVES, f"unknown move emitted: {move!r}")
    return moves
```

정렬 결과만 맞으면 되는 것이 아니라, 출력이 정확히 허용된 11개 instruction과 line framing으로 구성되어야 합니다. 빈 출력은 이미 정렬된 입력에서 유효한 빈 program입니다.

### Python 목록 모델은 C operation code를 호출하지 않는다

```python
def apply_moves(values, moves):
    a = list(values)
    b = []

    def swap(stack):
        if len(stack) > 1:
            stack[0], stack[1] = stack[1], stack[0]

    def push(dst, src):
        if src:
            dst.insert(0, src.pop(0))

    def rotate(stack):
        if len(stack) > 1:
            stack.append(stack.pop(0))

    def reverse_rotate(stack):
        if len(stack) > 1:
            stack.insert(0, stack.pop())
```

11개 명령을 Python list 연산으로 다시 구현하므로 C의 병렬 배열, `memmove`, shared wrapper와 구현 실패 방식이 다릅니다. generator와 product checker가 같은 C bug를 공유하더라도 reference model의 최종 `a`, `b` 상태까지 동시에 속여야 합니다.

한 fixture의 assertion은 세 층을 통과합니다.

```python
moves = split_moves(result.stdout)
a, b = apply_moves(values, moves)
assert_equal(a, sorted(values), "move stream sorts stack")
assert_equal(b, [], "move stream leaves stack b empty")

checked = run([CHECKER] + args, result.stdout)
assert_equal(checked.stdout, "OK\n", "checker OK")
```

- Python model: 명령 의미를 독립 검증
- Python `sorted`: 최종 목표 sequence를 독립 계산
- product checker: 실제 CLI protocol과 C 상태 경로를 검증

### 작은 입력의 exhaustive 범위

size 2부터 5까지 `itertools.permutations(range(size))`를 전부 실행합니다.

```text
2! + 3! + 4! + 5! = 2 + 6 + 24 + 120 = 152 cases
```

작은 정렬기는 직접 분기와 최소 rank 추출을 사용하므로 이 범위에서 모든 가능한 순서를 닫습니다. 이미 정렬된 5개 입력은 status 0, stdout empty, stderr empty여야 한다는 별도 assertion도 있습니다.

고정 fixture에는 음수, `INT_MIN`, `INT_MAX`, size 6과 size 10도 포함됩니다. 이들은 큰 입력 알고리즘의 대표 경로를 통과하지만, 모든 큰 permutation을 증명하지는 않습니다.

이 테스트가 증명하지 않는 것도 분명합니다.

- 생성 명령이 최소인지
- 100·500개 평가 상한을 지키는지
- C 배열 접근이 memory-safe한지
- allocation·write 실패에서 정리가 되는지

후속 commit들이 이 빈 곳을 각각 다른 기법으로 채웁니다.

## `a16dde75d935` — correctness와 명령 수 budget을 같은 fixture에서 확인

첫 비용 검사는 `random.seed(4242)`로 100개와 500개의 중복 없는 값을 만들고, 앞의 `assert_sorted_by_program`을 먼저 호출합니다. 따라서 line count만 작고 정렬은 틀린 program을 통과시키지 않습니다.

```python
limits = [
    (100, 1500),
    (500, 8000),
]
```

이 값은 ceiling입니다. 실제 명령 수가 이전 실행과 정확히 같을 필요는 없고 한도 이하면 됩니다. 또한 wall-clock time이나 내부 `memmove` 양을 측정하지 않습니다. push_swap 평가에서 관찰되는 command count를 고정하는 좁은 regression입니다.

이 SHA의 fixture는 Python `random` 구현에 의존하고 seed도 하나뿐입니다. 다음 커밋은 fixture 생성 규칙 자체를 source에 적어 환경과 실행 시점에 덜 의존하도록 바꿉니다.

## `23198a9cdd55` — 입력 생성과 출력 재현성을 결정적으로 만들기

### PRNG와 shuffle 규칙을 test source에 포함

```python
def deterministic_values(size, seed):
    values = list(range(size))
    state = seed & 0xFFFFFFFF
    for index in range(size - 1, 0, -1):
        state = (1664525 * state + 1013904223) & 0xFFFFFFFF
        selected = state % (index + 1)
        values[index], values[selected] = values[selected], values[index]
    offset = size * 23
    return [value * 37 - offset for value in values]
```

32-bit LCG와 Fisher–Yates 순서를 코드에 직접 적습니다. 마지막 affine transform은 순열의 상대 순서를 유지하면서 0부터 시작하는 단순 입력을 음수와 간격이 있는 값으로 바꿉니다. parser가 원본 magnitude가 아니라 상대 순위를 사용한다는 경로도 함께 통과합니다.

`PS_PUSH_SWAP`, `PS_CHECKER` 환경 변수로 test 대상 binary를 교체할 수 있게 되어 sanitizer build 등에도 같은 suite를 재사용할 수 있습니다.

### 여러 크기·seed와 exact output 반복

```python
for seed in (1, 7, 97, 4242, 9001):
    for size in (2, 3, 5, 6, 17, 64):
        values = deterministic_values(size, seed)
        first = assert_sorted_by_program(values)
        second = assert_sorted_by_program(values)
        assert_equal(second, first, "deterministic moves")
```

size 5와 6을 함께 둬 tiny/radix 경계를 통과하고, 17과 64는 bit count가 바뀌는 입력을 포함합니다. 같은 입력을 두 번 실행해 final state만 같은 것이 아니라 exact move list도 같은지 확인합니다.

100·500 command ceiling도 seed `7`, `4242`, `9001` 세 개로 확장됩니다. 다만 이 검사는 deterministic이라는 성질만 확인할 뿐, 서로 다른 정렬 program 중 어느 것이 더 좋은지는 판단하지 않습니다.

## `6569949742eb` — command count와 실제 배열 비용을 분리

명령 수가 같아도 배열 기반 구현의 내부 비용은 달라질 수 있습니다. `ra` 한 번과 `pb` 한 번은 모두 stdout 한 줄이지만, active stack 크기에 따라 이동하는 pair 수가 달라집니다. 이 커밋은 fault-injection build에서 다음 metric을 기록합니다.

| metric | source 정의 |
| --- | --- |
| `PS_OPERATIONS` | 성공적으로 stdout에 기록된 operation line 수 |
| `PS_ARRAY_MOVEMENTS` | 이동하거나 다시 쓴 `(value, rank)` pair 수 |
| `PS_PEAK_BYTES` | 계측 header를 제외하고 동시에 살아 있던 프로젝트 allocation 요청 합의 최대값 |
| `PS_LIVE_ALLOCATIONS` | 종료 시 아직 해제되지 않은 프로젝트 allocation 수 |

### 계측은 production build에서 제거된다

header는 `PS_FAULT_INJECTION`일 때만 recorder 함수를 선언하고, 일반 build에서는 no-op macro로 바꿉니다.

```c
#ifdef PS_FAULT_INJECTION
void ps_record_operation(void);
void ps_record_movements(size_t count);
#else
# define ps_record_operation() ((void)0)
# define ps_record_movements(count) ((void)0)
#endif
```

operation count는 출력 성공 뒤에만 증가합니다.

```c
if (emit)
{
    if (!ps_putstr_fd(1, name))
        return (0);
    ps_record_operation();
}
```

따라서 stdout line 수와 internal count가 다르면 출력 경계 또는 계측 위치가 잘못된 것입니다.

pair movement는 primitive별로 다음처럼 기록됩니다.

- swap: 2 pair
- push: destination shift + top copy + source shift를 합친 전체 active 원소 수
- rotate/reverse rotate: 해당 stack의 현재 size

counter addition이 `size_t`를 넘으면 wrap하지 않고 `(size_t)-1`로 saturate합니다. metric 자체의 overflow가 작은 값으로 돌아와 budget을 거짓 통과시키지 않도록 한 처리입니다.

### baseline은 exact count와 ceiling을 함께 사용한다

repository의 `resource_baseline.json`은 세 seed마다 다음 값을 둡니다.

| size | exact commands | max array movements | max peak bytes |
| ---: | ---: | ---: | ---: |
| 10 | 65 | 650 | 160 |
| 100 | 1084 | 105000 | 1600 |
| 500 | 6784 | 3200000 | 8000 |

각 size는 seed `7`, `4242`, `9001`에 대해 같은 항목을 가집니다.

`resource_tests.py`는 다음 순서로 검사합니다.

1. `PS_` prefix의 외부 환경 변수를 제거해 숨은 fault 설정을 차단합니다.
2. allocation report와 metric report를 켭니다.
3. 결정적 fixture를 5초 timeout으로 실행합니다.
4. 종료 status 0과 네 metric의 정확한 존재를 확인합니다.
5. `PS_LIVE_ALLOCATIONS == 0`을 요구합니다.
6. stdout line 수와 `PS_OPERATIONS`가 같아야 합니다.
7. command count는 baseline의 exact 값이어야 합니다.
8. movement와 peak bytes는 각각 ceiling 이하여야 합니다.

elapsed time도 출력하지만 assertion에는 사용하지 않습니다. CI host 부하에 민감한 wall-clock을 correctness budget으로 만들지 않고, code가 직접 통제할 수 있는 작업량과 allocation 양을 기준으로 삼은 결정입니다.

이 metric도 모든 성능을 대표하지는 않습니다. `qsort` 비교 횟수, allocator 내부 overhead, cache behavior, system call 비용은 측정하지 않습니다. 특히 `peak_bytes`는 프로젝트가 요청한 payload 합이며 instrumentation header와 allocator overhead를 제외합니다.

## `5505adf3e469` — 같은 논리 검사를 instrumented binary에서 재실행

Makefile은 일반 `.build`, fault `.build/fault`와 별도로 `.build/sanitize` object tree를 만듭니다.

```make
SANITIZE_CFLAGS := $(CFLAGS) -O1 -g -fno-omit-frame-pointer \
    -fsanitize=address,undefined
```

별도 object tree를 사용하므로 일반 object를 sanitizer executable에 섞지 않습니다. `sanitize` target은 다음을 실행하도록 구성됩니다.

- sanitizer build의 `operation_invariants`
- sanitizer `push_swap`과 `checker`를 대상으로 기존 `tests/run_tests.py`
- `ASAN_OPTIONS=halt_on_error=1`
- `UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1`

이는 exact state tests와 Python functional suite를 memory/undefined-behavior instrumentation 아래에서 다시 통과시키는 경로입니다. 별도 fault suite와 resource suite는 이 target에 포함되지 않습니다. 즉 sanitizer와 deterministic fault injection을 한 binary에서 동시에 수행한다고 해석하면 안 됩니다.

또한 compiler와 platform이 해당 sanitizer를 지원해야 한다는 실행 환경 전제가 있습니다. 이 commit은 build path를 제공하지만 모든 OS에서 동일한 runtime availability를 보장하지 않습니다.

## 증거층별 보장 범위

| 증거 | 직접 관찰하는 것 | 잡기 어려운 것 |
| --- | --- | --- |
| Python reference model | 명령 vocabulary, 최종 A 정렬, B empty | C 내부 OOB, allocation leak |
| product checker | 실제 checker protocol과 C operation 결과 | generator와 공유한 operation bug |
| 2~5 exhaustive | tiny sorter의 모든 permutation | 큰 입력 전체 공간 |
| 결정적 multi-seed | radix 경계·여러 bit 크기·재현 가능한 output | 모든 seed와 모든 size |
| command ceilings | 평가용 line count budget | 내부 배열 이동 비용 |
| resource baseline | exact lines, pair movement ceiling, peak/live allocation | allocator overhead, CPU time 전체 |
| ASan/UBSan target | 실행된 path의 memory/UB 진단 | 미실행 path, fault build와의 조합 |

## Thread의 최종 상태

정렬 correctness는 더 이상 “checker가 OK라고 했다”는 한 문장으로 축약되지 않습니다.

```text
push_swap stdout
  ├─ vocabulary 검사
  ├─ 독립 Python model 재생 -> sorted A, empty B
  ├─ product checker 재생 -> OK
  ├─ exact/ceiling command count
  ├─ instrumented operation·movement·allocation metrics
  └─ sanitizer binary에서 functional suite 재실행
```

각 layer의 failure message도 대상이 다릅니다. unknown move, 잘못된 final stack, checker disagreement, nondeterministic command list, baseline drift, movement budget 초과, allocation leak, sanitizer abort를 서로 다른 원인으로 식별할 수 있습니다.

## Thread의 경계

- 이 Thread는 정렬 알고리즘을 새로 구현하지 않습니다. `src/sort.c`가 만든 명령을 외부에서 관찰합니다.
- command count baseline은 특정 구현의 회귀 기준이지 전역 최적성 증명이 아닙니다.
- resource metric은 정의된 pair movement와 프로젝트 allocation만 셉니다.
- sanitizer target이 존재한다는 사실과 실제 특정 환경에서 통과했다는 주장은 다릅니다. 이 문서 작성 과정에서는 실행하지 않았습니다.
- allocation/read/write 실패를 만드는 runtime seam과 그 cleanup assertion은 별도의 runtime Thread에서 다룹니다.

이 Thread의 가장 중요한 결정은 **정답 판정기와 생성기가 같은 구현을 공유하는 위험을 독립 모델로 끊고, correctness·출력 비용·메모리 비용을 서로 다른 측정값으로 유지한 것**입니다.
