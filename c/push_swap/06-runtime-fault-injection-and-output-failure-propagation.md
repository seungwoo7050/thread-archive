# Thread: 런타임 실패 주입과 출력 실패 전파

`push_swap`의 정상 경로는 짧습니다. 메모리를 할당하고, checker는 stdin을 읽으며, generator와 checker는 몇 줄을 출력합니다. 그러나 이 세 시스템 호출은 부분 성공이나 일시적 실패를 가질 수 있습니다.

- 첫 번째 allocation은 성공하고 두 번째가 실패할 수 있습니다.
- `read`는 데이터 오류가 아니라 signal interruption인 `EINTR`로 돌아올 수 있습니다.
- `write`는 요청보다 적게 쓸 수 있고, 0을 반환하거나, 닫힌 pipe에서 `EPIPE`를 낼 수 있습니다.

초기 구현은 이런 실패를 관찰하지 않았습니다. 이 Thread는 시스템 호출을 공통 wrapper로 모으고, N번째 호출 실패를 결정적으로 만들며, 마지막에는 출력 성공 여부가 operation → sorter → executable entry point까지 끊기지 않고 전달되도록 수정합니다.

> **검사 범위**: `c/push_swap` 브랜치의 exact SHA diff와 해당 시점 source·fault tests를 검사했습니다. 현재 환경에서는 fault binary를 build하거나 실행하지 않았으므로 source가 구성한 failure와 assertion만 설명합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2e97f29961d8` | `feat(io): 문자열 비교와 기본 출력을 구현` | B | `RUNTIME, PRACTICAL` | 공통 문자열 비교와 단일 `write` 기반 출력 helper 도입 |
| 2 | `5faa9d7697af` | `refactor(runtime): 메모리와 입력 시스템 호출을 공통화` | A | `ARCH, REFACTOR, RUNTIME` | 프로젝트 allocation/free/read를 transparent wrapper 뒤로 이동 |
| 3 | `63969f770a21` | `test(memory): 할당 실패 뒤 자원 정리를 검증` | A | `TEST, RUNTIME, RISK` | N번째 allocation failure와 live-allocation 보고를 fault build에 추가 |
| 4 | `315f4b91779b` | `fix(io): 출력 실패를 호출 경로 끝까지 전파` | A | `RUNTIME, RISK, INTEGRATION` | short write·EINTR·EPIPE를 처리하고 operation 실패를 main까지 전파 |
| 5 | `e1154e181864` | `test(io): 부분 출력과 영구 쓰기 실패를 검증` | A | `TEST, RUNTIME, RISK` | write call별 실패, short/zero/EINTR, 닫힌 pipe, diagnostic failure를 검증 |

## `2e97f29961d8` — 출력 기능은 생겼지만 성공 여부는 사라진다

첫 utility 모듈은 세 가지 기본 기능을 제공합니다.

```c
size_t ps_strlen(const char *str);
int ps_strcmp(const char *a, const char *b);
void ps_putstr_fd(int fd, const char *str);
void write_error(void);
```

`ps_strcmp`가 비교 byte를 `unsigned char`로 변환한 뒤 차이를 반환하는 것은 음수 `char` 값에 따른 비교 오류를 피하는 일반적인 구현입니다. checker의 exact command dispatch가 이 helper를 사용합니다.

문제는 출력 helper입니다.

```c
void ps_putstr_fd(int fd, const char *str)
{
    if (str != NULL)
        (void)write(fd, str, ps_strlen(str));
}

void write_error(void)
{
    ps_putstr_fd(2, "Error\n");
}
```

한 번의 `write`가 전체 문자열을 썼다고 가정하며 반환값을 버립니다. 이 상태에서는 다음을 구별할 수 없습니다.

- 전체 성공
- 일부 byte만 기록된 short write
- `EINTR`
- `EPIPE`, `EIO` 같은 영구 오류
- 0-byte write

더 큰 문제는 return type이 `void`라 caller가 성공 여부를 알고 싶어도 받을 방법이 없다는 점입니다. operation wrapper는 이미 stack state를 바꾼 뒤 이 함수를 호출하므로 output failure가 발생해도 sorter는 다음 명령을 계속 생성합니다.

이 커밋은 정상 출력 기능을 마련한 중간 상태이며, failure contract는 아직 없습니다.

## `5faa9d7697af` — 실패를 만들 수 있는 호출 경계

이 refactor는 `malloc`, `free`, `read`를 다음 wrapper로 치환합니다.

```c
void *ps_malloc(size_t size)
{
    return (malloc(size));
}

void ps_free(void *pointer)
{
    free(pointer);
}

ssize_t ps_read(int fd, void *buffer, size_t count)
{
    return (read(fd, buffer, count));
}
```

production 동작은 그대로입니다. 바뀌는 것은 모든 프로젝트 소유 allocation과 checker read가 한 file의 함수 경계를 통과한다는 점입니다.

적용 범위에는 다음이 포함됩니다.

- `t_stack.values`, `t_stack.ranks`
- parser의 정렬 scratch array
- 당시 checker reader의 동적 line buffer
- checker loop의 line 해제

이 seam은 자원의 owner가 아닙니다. stack과 parser/checker가 여전히 자신이 획득한 pointer를 정리합니다. wrapper는 후속 test build가 “정확히 N번째 호출”을 선택할 수 있는 관찰 지점만 제공합니다.

이 커밋 자체는 allocation failure cleanup을 새로 증명하지 않으며 read retry policy도 바꾸지 않습니다.

## `63969f770a21` — allocation failure를 모든 획득 지점에 순서대로 주입

### production과 분리된 fault object tree

Makefile은 `PS_FAULT_INJECTION`을 정의한 별도 object와 executable을 `.build/fault`에 만듭니다. 일반 build object에 계측 header나 global counter를 섞지 않습니다.

### alignment를 유지하는 allocation header

fault build의 `ps_malloc`은 요청한 payload 앞에 metadata를 붙입니다.

```c
typedef union u_allocation_header
{
    struct
    {
        size_t size;
        unsigned long magic;
    } data;
    long double align_long_double;
    void *align_pointer;
} t_allocation_header;
```

union에 넓은 alignment를 가진 member를 두어 `header + 1`로 반환하는 payload가 일반 pointer·scalar에 필요한 alignment를 잃지 않도록 합니다.

```c
g_malloc_calls++;
if (at_index("PS_FAIL_MALLOC_AT", g_malloc_calls))
    return (NULL);
if (size > (size_t)-1 - sizeof(*header))
    return (NULL);
header = malloc(sizeof(*header) + size);
...
g_live_allocations++;
return ((void *)(header + 1));
```

`sizeof(header) + size`도 더하기 전에 overflow를 검사합니다. `ps_free`는 header로 돌아가 magic을 지우고 live counter를 감소시킨 뒤 실제 allocation을 해제합니다.

이 계측은 일반적인 heap debugger 전체를 대신하지 않습니다. 예를 들어 arbitrary invalid pointer나 이미 해제된 pointer를 안전하게 복구하는 기능은 없습니다. 목표는 프로젝트 wrapper를 통해 정상 획득된 allocation의 live count입니다.

### 모든 entry point가 cleanup 뒤 보고 지점에 도달한다

`push_swap`과 checker의 return이 `ps_test_finish(status)`로 바뀝니다. fault build에서 `PS_REPORT_ALLOCATIONS`가 설정되면 다음 중 하나를 raw stderr에 기록합니다.

```text
PS_LIVE_ALLOCATIONS=0
PS_LIVE_ALLOCATIONS=NONZERO
```

nonzero이면 원래 status 대신 99를 반환합니다. report는 계측 중인 출력 helper를 사용하지 않고 direct `write` loop를 사용하므로 allocation report 자체가 project allocation이나 후속 write seam을 재귀적으로 건드리지 않습니다.

### N번째 실패 sweep

Python fault test는 실제 allocation 순서를 따라 한 위치씩 실패시킵니다.

`push_swap 4 3 2 1`에서는 초기 source 기준으로 다음 다섯 allocation을 각각 실패시키고, 6번째를 지정하면 정상 성공해야 합니다.

```text
A.values -> A.ranks -> sorted scratch -> B.values -> B.ranks
```

checker의 `2 1` + `sa\n` 경로는 A/B와 parser scratch 외에 command line allocation이 있어 여섯 지점을 sweep하고 7번째에서 성공을 기대합니다. 후속 fixed-frame reader는 clean EOF 확인 때도 buffer를 할당하므로 `7713a31cf502`에서 이 expected count가 7 failure positions와 8번째 success로 갱신됩니다.

각 injected failure는 다음을 동시에 요구합니다.

- program status가 실패
- 일반 오류 표면은 `Error\n`
- allocation report가 존재
- `NONZERO` report는 없음

따라서 “실패를 감지했다”뿐 아니라 그 시점까지 획득한 모든 프로젝트 allocation을 정리했는지 확인합니다.

## `315f4b91779b` — output을 best-effort side effect에서 반환 가능한 작업으로

이 fix는 한 함수만 바꾸지 않습니다. output 실패가 중간 helper에서 사라지지 않도록 호출 graph 전체의 반환형을 바꿉니다.

```text
ps_write_all
  -> ps_putstr_fd
    -> emit_op
      -> op_sa ... op_rrr
        -> sort_two / sort_three / sort_tiny / radix_sort
          -> sort_stack
            -> push_swap main
```

checker verdict도 `ps_putstr_fd` 결과를 status로 바꿉니다.

### short write는 이미 기록한 prefix 뒤에서 계속한다

```c
int ps_write_all(int fd, const void *buffer, size_t count)
{
    const unsigned char *cursor;
    ssize_t written;

    cursor = (const unsigned char *)buffer;
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
    return (1);
}
```

- positive short write: cursor와 remaining count를 갱신해 나머지만 씁니다.
- `EINTR`: byte가 기록되지 않았으므로 같은 cursor에서 재시도합니다.
- 0 또는 다른 음수: progress가 없거나 영구 실패로 보고 0을 반환합니다.

이미 출력한 prefix를 처음부터 반복하지 않는 것이 중요합니다. 명령 이름 일부가 써진 뒤 실패한 경우 rollback할 수는 없지만, 재시도 과정에서 중복 byte를 만들지는 않습니다.

### 닫힌 pipe를 signal death가 아니라 cleanup 가능한 오류로

```c
int ps_ignore_sigpipe(void)
{
    return (signal(SIGPIPE, SIG_IGN) != SIG_ERR);
}
```

`push_swap`과 값이 있는 checker는 시작 단계에서 SIGPIPE를 무시합니다. stdout reader가 사라지면 process가 signal로 즉시 종료되는 대신 `write`가 `-1/EPIPE`를 반환하고 정상 failure path를 통과합니다. 그 결과 A/B와 scratch allocation을 정리하고 status 1을 반환할 수 있습니다.

signal 설정 자체가 실패하면 입력 처리 전에 오류 status로 끝납니다. checker의 무인자 경로는 stdin·stdout 작업 없이 바로 반환하므로 signal 설정보다 먼저 종료합니다.

### operation은 상태 변경 뒤 출력 결과를 반환한다

```c
static int emit_op(const char *name, int emit)
{
    if (emit)
        return (ps_putstr_fd(1, name));
    return (1);
}

int op_ra(t_stack *a, int emit)
{
    stack_rotate(a);
    return (emit_op("ra\n", emit));
}
```

checker는 `emit = 0`이므로 operation이 항상 output success를 반환하며 상태 재생 semantics는 유지됩니다. generator는 `emit = 1`이므로 write failure를 즉시 받습니다.

여기에는 의도적인 비대칭이 있습니다. 상태 mutation이 먼저 일어나고 output이 뒤따릅니다. 출력에 실패하면 내부 A/B는 해당 명령을 적용한 상태지만 외부 stream은 전체 명령을 받지 못했을 수 있습니다. fix는 이를 rollback하려 하지 않습니다. 대신 정렬을 즉시 중단하고 process 전체를 실패로 표시합니다. 실패한 실행의 stdout prefix를 유효한 완성 program으로 약속하지 않는 선택입니다.

### sorter는 첫 실패에서 더 이상 명령을 만들지 않는다

모든 sort helper가 `int`를 반환합니다.

```c
if (!move_index_to_top(a, index) || !op_pb(a, b, 1))
    return (0);
```

```c
if (((a->ranks[0] >> bit) & 1) == 1)
{
    if (!op_ra(a, 1))
        return (0);
}
else if (!op_pb(a, b, 1))
    return (0);
```

두 명령이 필요한 3-element case도 첫 명령 실패 시 두 번째를 실행하지 않습니다. radix loop도 실패한 pass에서 즉시 끊깁니다.

`push_swap main`은 sort result와 관계없이 A/B를 해제한 뒤, 실패했으면 `Error\n`을 시도하고 status 1을 `ps_test_finish`에 넘깁니다. diagnostic 출력마저 실패해도 primary failure status는 이미 결정됐으므로 1을 유지합니다.

### checker verdict write도 status가 된다

checker는 valid program을 모두 적용한 뒤 `OK\n` 또는 `KO\n` write 결과를 반전해 status로 저장합니다. verdict 출력이 실패하면 A/B를 정리하고 `Error\n`을 시도한 뒤 status 1로 끝납니다.

unknown command나 parse failure에서 `write_error`가 실패해 stderr가 비어도 original semantic failure는 status 1로 남습니다. 오류를 보고하는 작업이 원래 오류를 성공으로 바꾸지 않습니다.

## `e1154e181864` — write 결과의 네 종류를 결정적으로 재현

fault build의 `ps_write_once`에 call counter와 네 selector가 추가됩니다.

```c
if (at_index("PS_EINTR_WRITE_AT", g_write_calls))
    return (errno = EINTR, -1);
if (at_index("PS_FAIL_WRITE_AT", g_write_calls))
    return (errno = EPIPE, -1);
if (at_index("PS_ZERO_WRITE_AT", g_write_calls))
    return (0);
if (at_index("PS_SHORT_WRITE_AT", g_write_calls) && count > 1)
    count = 1;
```

### 모든 명령 write 위치에서 영구 실패

먼저 `push_swap 3 2 1`의 baseline stdout line 수를 구합니다. fixture가 두 줄 이상을 내는지 확인한 뒤 1번째부터 마지막 write까지 각각 `EPIPE`를 주입합니다. 매 case는 nonzero status와 `Error\n`을 요구합니다.

이 방식은 첫 operation뿐 아니라 이미 여러 명령을 성공적으로 쓴 뒤 발생하는 late failure도 main까지 전달되는지 검사합니다.

### 복구 가능한 write

첫 write에 `EINTR` 또는 1-byte short write를 주입했을 때 status 0과 baseline과 정확히 같은 stdout을 요구합니다. short write 뒤 cursor가 전진하지 않거나, 전체 문자열을 다시 쓰면 byte stream이 달라져 실패합니다.

### progress가 없는 write

0-byte write는 무한 loop로 재시도하지 않고 영구 실패로 취급해야 합니다. test는 nonzero와 `Error\n`을 요구합니다.

### partial prefix 뒤 영구 실패

```python
faults={"PS_SHORT_WRITE_AT": 1, "PS_FAIL_WRITE_AT": 2}
```

첫 call이 한 byte를 쓰고 두 번째가 실패하도록 합니다. expected stdout은 `baseline.stdout[:1]`입니다. 이미 성공한 한 byte가 사라지지도, 처음부터 반복되지도 않아야 하며 status는 정확히 1입니다.

### checker·diagnostic·실제 pipe 경계

추가 case는 다음을 분리합니다.

- checker의 `OK\n` 첫 write 실패가 status 1과 `Error\n`으로 전파됨
- parse error나 unknown command에서 `Error\n` 자체의 write가 실패해도 status 1 유지, stdout/stderr empty 허용
- 실제 OS pipe의 read end를 닫고 `push_swap` stdout을 write end에 연결했을 때 signal로 죽지 않고 status 1, stderr `Error\n`, live allocation 0 report로 종료

마지막 case는 injected `EPIPE`뿐 아니라 실제 closed-pipe behavior와 SIGPIPE 무시 설정을 함께 통과합니다.

이 suite가 모든 write 환경을 증명하는 것은 아닙니다. 여러 번 연속되는 `EINTR`, 다양한 positive short length, stderr도 닫힌 상태, signal handler 경쟁 등은 범위 밖입니다. 그러나 정상·일시 중단·부분 progress·무진행·영구 오류의 대표 상태를 각각 구분합니다.

## 최종 failure convergence

| 실패 지점 | 이미 획득·변경된 상태 | 종료 처리 | process 결과 |
| --- | --- | --- | ---: |
| A 첫 배열 allocation | 없음 또는 부분 A | `stack_free`가 empty state로 수렴 | 1 |
| parser scratch allocation | A 두 배열 | parser가 A까지 정리 | 1 |
| B 두 번째 배열 allocation | 완성 A, 부분 B | B 내부 cleanup + main의 A cleanup | 1 |
| checker frame allocation/read | A/B, 필요 시 이전 frames 적용 | 현재 buffer 정리, A/B 정리 | 1 |
| operation stdout short write 후 회복 | state mutation, partial byte progress | 남은 byte만 계속 기록 | 계속 실행 |
| operation stdout 영구 실패 | 해당 operation state는 이미 적용, stdout은 prefix | sorter 즉시 중단, A/B 정리 | 1 |
| checker verdict write 실패 | 최종 A/B state 완성 | A/B 정리, Error write 시도 | 1 |
| Error diagnostic write 실패 | primary failure 확정 | cleanup 결과 유지 | 1 |
| closed stdout pipe | write 시 EPIPE | SIGPIPE 대신 동일 output-failure path | 1 |

## 최종 호출 흐름

```text
[production]
ps_malloc / ps_free / ps_read -> libc syscall 그대로 위임
ps_write_all -> EINTR retry, short-write continuation, 0/error failure

[fault build]
선택한 malloc/read/write call에서 deterministic result 주입
live allocation과 resource metric 보고

[generator]
operation state mutation
  -> command write
  -> 실패면 sort helper chain이 0 반환
  -> main이 A/B free
  -> Error 시도
  -> status 1

[checker]
frame read/apply
  -> final verdict write
  -> 실패면 A/B free
  -> Error 시도
  -> status 1
```

## Thread의 경계

- fault wrapper는 production owner를 대신하지 않습니다. 각 caller가 여전히 자신이 획득한 자원을 정리합니다.
- 출력 실패 전에 이미 stdout에 전달된 prefix는 회수할 수 없습니다. 실패 실행은 완성된 명령 program을 보장하지 않습니다.
- operation은 state를 먼저 바꾸므로 output failure 시 내부 state와 외부 stream이 일치하지 않을 수 있습니다. process를 즉시 실패시키는 것으로 publish를 중단합니다.
- live-allocation counter는 프로젝트 wrapper를 통한 allocation을 측정하며 libc·Python·kernel resource 전체를 측정하지 않습니다.
- sanitizer 검증과 resource budget은 독립 정확성·비용 Thread에서 별도 build로 구성됩니다.

이 Thread가 확립한 최종 규칙은 간단합니다. **메모리·입력·출력 실패는 helper 내부에서 사라지지 않으며, 이미 획득한 자원을 정리한 뒤 executable의 실패 status까지 도달합니다.**
