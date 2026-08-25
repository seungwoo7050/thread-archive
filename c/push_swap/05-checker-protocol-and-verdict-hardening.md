# Thread: checker 프로토콜과 판정 강화

checker는 두 일을 동시에 수행합니다.

1. stdin을 줄 단위 명령 protocol로 해석해 A/B 상태에 적용합니다.
2. 입력이 끝났을 때 `OK`, `KO`, `Error` 중 하나를 선택합니다.

초기 구현은 명령 해석과 판정은 갖췄지만 reader가 일반적인 동적 문자열 입력기로 만들어졌습니다. 실제 instruction은 최대 세 글자뿐인데도 임의 길이를 할당했고, embedded NUL과 `EINTR`을 protocol 관점에서 구분하지 못했습니다. 후속 fix는 reader를 “문자열 유틸리티”가 아니라 **최대 3-byte command frame parser**로 다시 정의합니다.

> **검사 범위**: `c/push_swap` 브랜치의 exact SHA diff와 해당 시점의 checker source·tests를 검사했습니다. 현재 환경에서는 binary와 fault suite를 실행하지 않았으므로 테스트의 source assertion을 설명하며 실행 결과를 주장하지 않습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `0b87adebca2b` | `feat(checker): 표준 입력 명령 프레임을 읽음` | B | `CHECKER, CORE` | newline 또는 EOF까지 동적 문자열을 읽는 tri-state reader 도입 |
| 2 | `f79ae7e86592` | `feat(checker): 스택 연산 명령을 해석` | B | `CHECKER, INTEGRATION` | 11개 명령을 공통 operation wrapper의 silent 실행으로 연결 |
| 3 | `d906f4d86528` | `feat(checker): 명령 실행 결과를 판정` | A | `CHECKER, CORE, INTEGRATION` | parse → read/apply → complete-sort 판정의 checker executable 완성 |
| 4 | `44ee0830e9f0` | `test(checker): 명령 연산과 최종 판정을 검증` | B | `TEST, CHECKER` | 각 operation family, `OK`, `KO`, unknown command를 CLI로 검증 |
| 5 | `7713a31cf502` | `fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도` | A | `CHECKER, RUNTIME, RISK` | protocol 최대 길이·NUL 거절·`EINTR` retry를 reader 자체에 적용 |
| 6 | `dbf76e147e68` | `test(checker): 읽기 실패와 명령 경계를 검증` | A | `TEST, CHECKER, RISK` | read call별 EIO/EINTR와 malformed frame 경계를 deterministic하게 검증 |

## `0b87adebca2b` — 일반 line reader로 시작한 protocol 입력

첫 `read_next_line`은 다음 tri-state를 반환합니다.

| 반환값 | 의미 | `*line` |
| ---: | --- | --- |
| `1` | newline 또는 non-empty EOF로 한 frame 완성 | NUL-terminated heap string |
| `0` | frame 시작 전 clean EOF | 해제됨 |
| `-1` | read 또는 allocation 실패 | 해제됨 |

reader는 한 byte씩 읽고, 필요할 때 capacity를 32부터 두 배로 늘립니다.

```c
while (1)
{
    bytes = read(fd, &c, 1);
    if (bytes < 0)
        return (free(*line), -1);
    if (bytes == 0)
        break ;
    if (c == '\n')
        break ;
    if (!grow_line(line, &capacity, len + 1))
        return (free(*line), -1);
    (*line)[len++] = c;
}
```

개행은 결과 문자열에 포함하지 않습니다. `sa\n`은 `"sa"`로, 마지막 `sa`가 개행 없이 EOF로 끝나도 동일한 문자열로 반환됩니다. EOF 전에 한 byte도 없으면 stream 종료입니다.

기능을 연결하기 위한 출발점으로는 충분하지만 checker protocol과 맞지 않는 부분이 있습니다.

- 가장 긴 유효 명령은 `rra`, `rrb`, `rrr`의 3 byte인데 reader는 임의 길이 입력을 계속 할당합니다.
- `read`가 `EINTR`로 중단돼도 영구 읽기 오류와 똑같이 `-1`을 반환합니다.
- embedded NUL을 일반 byte처럼 저장합니다. 후속 `ps_strcmp`는 첫 NUL에서 비교를 끝내므로 `sa\0pb` 같은 frame이 `sa`로 보일 수 있습니다.
- empty line은 길이 0의 frame으로 반환되어 dispatcher에서 unknown command가 됩니다. 최종 결과는 오류지만 framing 단계에서 거절하는 것은 아닙니다.

이 커밋의 reader는 “줄을 읽는 함수”이고 아직 “checker 명령을 읽는 함수”는 아닙니다.

## `f79ae7e86592` — 명령 문자열과 상태 연산을 1:1로 연결

`apply_checker_command`는 허용된 11개 문자열만 exact 비교합니다.

```c
if (ps_strcmp(line, "sa") == 0)
    op_sa(a, 0);
else if (ps_strcmp(line, "sb") == 0)
    op_sb(b, 0);
/* ... */
else if (ps_strcmp(line, "rrr") == 0)
    op_rrr(a, b, 0);
else
    return (0);
return (1);
```

`emit = 0`이므로 checker가 명령을 재생하면서 stdout에 같은 명령을 다시 쓰지 않습니다. 상태 변화는 generator가 사용하는 공통 operation layer와 같습니다.

원소가 부족한 상태의 명령도 유효합니다. 예를 들어 B가 비었을 때 `pa`는 상태 no-op이지만 unknown command가 아니므로 `apply_checker_command`는 성공합니다. protocol validity와 state mutation 유무를 구분한 것입니다.

이 SHA에서는 dispatcher 함수만 생기고 실제 stdin loop나 verdict는 아직 없습니다.

## `d906f4d86528` — clean EOF만 최종 판정으로 이어진다

checker executable은 다음 lifecycle을 만듭니다.

```text
argc == 1 -> 아무것도 읽지 않고 status 0
입력 parse -> B를 A.capacity로 초기화
반복:
  read_next_line
  frame이면 apply_checker_command 후 free
  clean EOF면 반복 종료
  unknown/read/allocation failure면 Error
최종:
  B empty && A ranks sorted -> OK
  그 외 valid execution -> KO
```

### frame 소유권

```c
status = read_next_line(0, &line);
while (status > 0)
{
    if (!apply_checker_command(line, a, b))
    {
        free(line);
        return (0);
    }
    free(line);
    status = read_next_line(0, &line);
}
return (status == 0);
```

reader가 반환한 heap string은 정확히 한 iteration 동안 checker loop가 소유합니다. command 성공과 unknown command 양쪽에서 해제합니다. read failure 시 reader가 자체 buffer를 정리하고 `-1`을 반환합니다.

### `KO`는 프로그램 오류가 아니다

```c
if (stack_is_complete_sorted(&a, &b))
    ps_putstr_fd(1, "OK\n");
else
    ps_putstr_fd(1, "KO\n");
```

유효한 명령열이 최종 정렬에 실패한 것은 checker 자체의 실행 실패가 아닙니다. 따라서 `KO`도 status 0입니다. 반면 다음은 `Error\n`과 status 1입니다.

- 잘못된 숫자 입력
- A/B allocation 실패
- stdin read 실패
- unknown command

`argc == 1`은 parser와 stdin reader보다 먼저 return합니다. checker가 값 없이 실행되었을 때 pipe input을 소비하거나 prompt를 기다리지 않습니다.

이 시점의 `OK`/`KO`/`Error` 출력은 write 결과를 확인하지 않습니다. verdict 선택은 완성됐지만 output delivery는 runtime Thread에서 보강됩니다.

## `44ee0830e9f0` — 명령군과 세 가지 결과를 CLI로 고정

테스트는 A만으로 확인 가능한 명령뿐 아니라 B를 실제로 채워야 하는 명령도 program sequence로 만듭니다.

```python
cases = [
    ("sa", ["2", "1"], "sa\n"),
    ("sb", ["2", "1", "3"], "pb\npb\nsb\npa\npa\n"),
    ("ss", ["2", "1", "4", "3"], "pb\npb\nss\npa\npa\n"),
    ("pa-pb", ["1", "2"], "pb\npa\n"),
    /* rotate와 reverse-rotate 계열 */
]
```

각 case는 status 0, stdout `OK\n`, stderr empty를 요구합니다. 별도 case는 다음을 확인합니다.

- 정렬되지 않은 `2 1`에 빈 program: `KO\n`, status 0
- valid `ra` 뒤 unknown `wat`: stdout 없음, stderr `Error\n`, nonzero status

unknown command 전에 적용된 valid operation을 rollback하는지는 검사하지 않습니다. checker는 malformed stream을 발견하면 최종 verdict를 내지 않고 실패하므로 중간 state는 외부에 publish되지 않습니다.

이 suite는 operation dispatch와 verdict를 검증하지만, embedded NUL, 너무 긴 frame, read interruption은 다루지 않습니다.

## `7713a31cf502` — reader를 protocol 크기로 다시 설계

### 문제의 위치

기존 구현에서 malformed command는 대부분 dispatcher가 문자열 비교를 마친 뒤 거절했습니다. 하지만 frame 길이와 NUL은 단순한 “알 수 없는 명령 이름”이 아니라 input representation의 문제입니다.

- 4번째 non-newline byte가 들어온 순간 어떤 유효 명령도 될 수 없습니다.
- NUL은 C string 비교 경계를 바꿔 뒤쪽 byte를 숨깁니다.
- `EINTR`은 데이터 오류가 아니라 같은 read를 다시 시도해야 하는 일시적 중단입니다.

따라서 fix는 dynamic grow logic을 제거하고 protocol 상수만큼만 할당합니다.

```c
#define PS_COMMAND_MAX 3

*line = (char *)ps_malloc(PS_COMMAND_MAX + 1);
if (*line == NULL)
    return (-1);
```

buffer는 heap에 4 byte만 할당됩니다. 세 command byte와 마지막 NUL을 위한 크기입니다. 매 frame마다 고정 크기 allocation을 하고 clean EOF를 확인하기 위한 다음 reader 호출에서도 하나를 할당했다가 해제합니다. 이 변화 때문에 allocation-failure sweep의 checker call count도 조정됩니다.

### 종료·오류 조건의 순서

```c
bytes = ps_read(fd, &c, 1);
if (bytes < 0 && errno == EINTR)
    continue ;
if (bytes < 0)
    return (ps_free(*line), *line = NULL, -1);
if (bytes == 0)
    break ;
if (c == '\n')
    break ;
if (c == '\0' || len >= PS_COMMAND_MAX)
    return (ps_free(*line), *line = NULL, -1);
(*line)[len++] = c;
```

개행을 먼저 확인하므로 3-byte 명령 뒤 newline은 정상입니다. 이미 3 byte를 읽은 뒤 네 번째 non-newline byte가 오면 즉시 실패합니다. 나머지 line을 drain하지 않지만 checker는 곧 오류로 종료하므로 다음 frame과 동기화할 필요가 없습니다.

모든 오류와 clean EOF에서 `*line = NULL`로 돌려 caller가 해제된 pointer를 보유하지 않게 합니다. EOF 시 `len > 0`이면 NUL을 붙여 유효한 마지막 frame으로 반환하므로 개행 없는 `sa`는 계속 허용됩니다.

### 최종 framing 규칙

| byte stream | reader 결과 |
| --- | --- |
| `sa\n` | frame `sa` |
| `rra\n` | frame `rra` |
| `sa` + EOF | frame `sa`, 다음 호출에서 clean EOF |
| 즉시 EOF | clean EOF |
| `\n` | empty frame, dispatcher가 오류 처리 |
| `rrrr\n` 또는 `rrrr` | 4번째 byte에서 read failure 의미의 `-1` |
| `sa\0pb\n` | NUL에서 즉시 `-1` |
| `read == -1, errno == EINTR` | 같은 frame read 계속 |
| 다른 read 오류 | buffer 정리 후 `-1` |

empty line은 이 fix에서도 reader가 길이 0 frame으로 반환하고 dispatcher가 거절합니다. source가 선택한 결과는 동일한 `Error`이므로 별도 frame status를 추가하지 않았습니다.

## `dbf76e147e68` — call 위치와 byte 경계를 분리해 회귀 고정

runtime wrapper에 read call counter가 추가됩니다.

```c
g_read_calls++;
if (at_index("PS_EINTR_READ_AT", g_read_calls))
    return (errno = EINTR, -1);
if (at_index("PS_FAIL_READ_AT", g_read_calls))
    return (errno = EIO, -1);
```

`sa\n` program을 끝까지 읽는 경로에는 다음 네 호출이 있습니다.

1. `s`
2. `a`
3. newline
4. 다음 frame을 시작하며 확인하는 EOF

테스트는 1~4번째 각각을 EIO로 실패시켜 checker가 nonzero와 `Error\n`으로 끝나는지 확인합니다. EINTR는 첫 byte와 terminal EOF 위치인 1·4번째에 주입해 retry 뒤 `OK\n`이 유지되는지 봅니다. frame 내부와 stream 종료 양쪽의 retry를 구분한 선택입니다.

malformed stream 목록은 protocol의 대표 경계를 직접 포함합니다.

```python
invalid_streams = [
    b"sa\x00pb\n",
    b"rrrr\n",
    b"rrrr",
    b"\x00\n",
    b"\n",
    b"r" * 65536 + b"\n",
]
```

64 KiB line은 reader가 전체를 할당하지 않고 네 번째 byte에서 이미 거절해야 하는 경로를 압박합니다. 테스트는 invalid stream마다 stdout empty, stderr `Error\n`, nonzero status를 요구합니다.

마지막으로 `b"sa"`를 개행 없이 전달해 EOF-delimited final frame이 적용되고 `OK\n`이 되는지 확인합니다.

이 테스트가 다루지 않는 범위도 있습니다.

- 모든 가능한 `errno`
- 여러 valid frame 사이 모든 call index의 EINTR 조합
- stdin을 계속 보내는 공격자가 있을 때 process-level backpressure
- verdict stdout/stderr의 write 실패

마지막 항목은 runtime Thread의 write fault suite가 담당합니다.

## 최종 checker 상태

```text
[argc 확인]
  argc == 1 -> stdin untouched, status 0
        ↓
[parse values -> A, init B]
        ↓
[fixed 4-byte frame allocation]
  byte-by-byte ps_read
  EINTR retry
  NUL 또는 4번째 command byte 거절
        ↓
[exact 11-command dispatch, emit=0]
        ↓
[clean EOF]
  A sorted && B empty -> OK
  valid but incomplete -> KO

어느 parse/read/frame/dispatch 오류든 -> Error, status 1
```

| 결과 | stdin/program 상태 | process status |
| --- | --- | ---: |
| `OK` | 모든 frame valid, 최종 A 정렬, B empty | 0 |
| `KO` | 모든 frame valid, 최종 완료 조건 불충족 | 0 |
| `Error` | 입력 숫자·allocation·read·frame·명령 오류 | 1 |
| 출력 자체 실패 | 이 Thread의 초기 구현에서는 미관찰; runtime fix 이후 1 | runtime Thread |

## Thread의 경계

이 Thread는 command stream의 **입력 protocol과 verdict 의미**를 다룹니다.

- 각 명령이 `(value, rank)` pair를 어떻게 움직이는지는 operation Thread에 정의됩니다.
- generator가 올바른 stream을 만드는지는 정렬·독립 검증 Thread가 다룹니다.
- `ps_malloc`, `ps_read` seam과 allocation leak 보고의 구현은 runtime Thread에 속하지만, 여기서는 reader failure를 만들기 위해 사용합니다.
- `OK`, `KO`, `Error`를 쓰는 도중 short write·EPIPE가 발생하는 경우는 `315f4b91779b`와 `e1154e181864`에서 완성됩니다.

가장 중요한 변화는 무제한 line reader를 더 복잡하게 보강한 것이 아니라, **프로토콜이 허용하는 최대 frame 자체를 메모리·읽기 경계로 사용하도록 단순화한 것**입니다.
