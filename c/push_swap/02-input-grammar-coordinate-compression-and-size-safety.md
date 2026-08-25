# Thread: 입력 문법, 좌표 압축, 크기 안전성

`push_swap`의 정렬기는 임의의 부호 있는 정수를 직접 비교하지 않습니다. 입력을 먼저 하나의 token stream으로 해석하고, 각 값을 `int` 범위로 검증하며, 중복이 없는 원본 값들을 `0..n-1`의 순위로 바꾼 뒤 그 순위만으로 정렬합니다.

이 Thread의 핵심은 단순한 `atoi` 구현이 아닙니다. 다음 세 단계가 하나의 입력 계약을 이룹니다.

```text
argv 문자열들
  -> 공백으로 분리된 정수 token stream
  -> 원본 int 배열
  -> 중복 없는 dense rank 배열
```

마지막 커밋은 이 논리를 아주 큰 입력에도 그대로 적용할 수 있도록 token count, 문자열 index, allocation byte 계산의 정수형을 다시 점검합니다.

> **검사 범위**: `c/push_swap` 브랜치에 포함된 각 exact SHA의 diff와 그 시점의 `src/parser.c`, `src/stack.c`, 테스트 source를 검사했습니다. 현재 환경에서는 checkout·build·test를 실행하지 않았습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f36ad8899b5f` | `feat(parse): 개별 인자의 부호 있는 정수를 파싱` | B | `INPUT, EDGE` | argv 하나를 부호·digit·`int` 범위가 검증된 정수 하나로 변환 |
| 2 | `3bfb465ebdb1` | `feat(parse): 공백으로 결합된 인자 토큰을 처리` | B | `INPUT, EDGE` | 여러 argv와 각 argv 내부 공백을 하나의 token stream으로 통합 |
| 3 | `e09cf45e21cd` | `feat(parse): 중복 입력을 거절하고 상대 순위를 계산` | S | `CORE, INPUT, SORT` | 원본 값을 보존하면서 정렬용 dense rank를 만들고 중복을 거절 |
| 4 | `4cc9783286c0` | `test(parser): 정상 입력과 오류 입력을 검증` | B | `TEST, INPUT` | 기본 성공·실패 CLI 동작과 생성 명령의 checker 수용을 확인 |
| 5 | `44a4da8bc63d` | `test(cli): 입력 경계와 무인자 실행을 검증` | B | `TEST, INPUT, EDGE` | 부호·공백·int 경계·긴 숫자·빈 argv·무인자 checker 경계를 확장 |
| 6 | `049ecd429548` | `fix(parse): 토큰 수와 배열 크기 계산을 방어` | A | `INPUT, EDGE, RISK` | token 수 합산, 문자열 index, allocation 곱셈의 overflow를 사전 차단 |

## `f36ad8899b5f` — 한 token이 정수로 인정되는 조건

첫 parser는 `argv[1]`부터 각 인자를 정수 하나로 해석합니다. token 문법은 다음과 같습니다.

```text
[ '+' | '-' ]? ASCII_DIGIT+
```

부호만 있는 `+`나 `-`는 숫자가 아니며, `0`부터 `9` 이외의 byte는 허용하지 않습니다. locale에 따라 숫자로 분류될 수 있는 다른 Unicode 문자를 받아들이지 않는 명시적인 ASCII 규칙입니다.

### 양수와 음수의 한계가 다른 이유

```c
limit = INT_MAX;
if (sign < 0)
    limit = (long long)INT_MAX + 1;
while (i < end)
{
    if (arg[i] < '0' || arg[i] > '9')
        return (0);
    value = value * 10 + (arg[i] - '0');
    if (value > limit)
        return (0);
    i++;
}
*out = (int)(value * sign);
```

2의 보수 `int`는 보통 음수 쪽에 한 값이 더 있습니다. 따라서 `-2147483648`을 받으려면 magnitude 한계를 `INT_MAX + 1`로 두어야 하고, 양수는 `INT_MAX`까지만 허용해야 합니다. 누적은 `long long`에서 수행되며, `int` 한계를 넘는 즉시 반환하므로 아주 긴 숫자도 `long long` 한계까지 계속 누적하지 않습니다.

이 시점의 `parse_input`은 `argc - 1`개 슬롯을 할당하고 각 argv를 통째로 하나의 token으로 봅니다. 성공한 값은 임시로 `values[index]`와 `ranks[index]` 양쪽에 복사됩니다. 아직 rank가 아니라 원본 값의 사본일 뿐입니다.

실패하면 `stack_free(a)`로 이미 확보한 두 배열을 함께 정리합니다. 무인자 실행(`argc == 1`)은 오류가 아니라 empty A를 반환합니다.

## `3bfb465ebdb1` — argv 경계를 없애고 token 순서만 남기기

42 과제 입력은 다음 두 형태를 모두 같은 값 sequence로 취급해야 합니다.

```text
./push_swap 3 2 1
./push_swap "3 2" "1"
```

이 커밋은 여섯 C whitespace 문자를 separator로 인정합니다.

```c
static int is_space(char c)
{
    return (c == ' ' || c == '\t' || c == '\n'
        || c == '\r' || c == '\v' || c == '\f');
}
```

구현은 두 번 스캔합니다.

1. `count_all_tokens`가 모든 argv 안의 token 수를 합산합니다.
2. 정확한 capacity로 A를 한 번 초기화합니다.
3. `fill_values`가 각 token의 `[start, end)` 범위를 `parse_token`에 넘깁니다.

substring을 새로 할당하거나 argv를 수정하지 않습니다. empty argv 하나는 token 0개로 지나가므로 다른 유효 token과 섞여 있으면 무시됩니다. 반대로 제공된 모든 argv가 비었거나 공백뿐이라 전체 token 수가 0이면 오류입니다.

```c
count = count_all_tokens(argc, argv);
if (count == 0)
    return (0);
if (!stack_init(a, count))
    return (0);
if (!fill_values(argc, argv, a))
{
    stack_free(a);
    return (0);
}
```

이 단계가 끝나도 정렬기는 아직 임의의 음수와 큰 양수를 그대로 보고 있습니다. 다음 커밋이 이 표현을 순위 공간으로 바꿉니다.

## `e09cf45e21cd` — 원본 값에서 dense rank 순열로

이 커밋은 이 Thread의 중심입니다. 결과적으로 다음 조건이 성립합니다.

- `values[i]`는 사용자가 입력한 정수와 순서를 그대로 유지합니다.
- `ranks[i]`는 `values[i]`가 전체 입력에서 몇 번째로 작은지 나타냅니다.
- 입력이 중복되지 않았다면 ranks는 정확히 `0..n-1`의 순열입니다.
- `values[i] < values[j]`이면 `ranks[i] < ranks[j]`입니다.

### 정렬용 복사본과 중복 검사

```c
sorted = (int *)malloc(sizeof(int) * (size_t)a->size);
if (sorted == NULL)
    return (0);
i = 0;
while (i < a->size)
{
    sorted[i] = a->values[i];
    i++;
}
qsort(sorted, (size_t)a->size, sizeof(int), compare_ints);
```

원본 배열을 직접 정렬하지 않고 scratch array를 만듭니다. comparator는 `a - b`를 반환하지 않고 다음처럼 비교 결과만 반환합니다.

```c
return ((a > b) - (a < b));
```

큰 양수에서 큰 음수를 빼는 comparator의 signed overflow를 피하기 위한 형태입니다.

정렬된 배열에서는 같은 값이 인접하므로 한 번의 선형 스캔으로 중복을 찾습니다.

```c
if (sorted[i - 1] == sorted[i])
{
    free(sorted);
    return (0);
}
```

문자열 표기가 달라도 숫자 값이 같으면 중복입니다. 예를 들어 `-0`, `+0`, `0000`은 모두 정수 0이므로 함께 들어올 수 없습니다.

### binary lower bound가 순위를 결정한다

```c
static int find_rank(const int *sorted, int size, int value)
{
    int low;
    int high;
    int mid;

    low = 0;
    high = size;
    while (low < high)
    {
        mid = low + (high - low) / 2;
        if (sorted[mid] < value)
            low = mid + 1;
        else
            high = mid;
    }
    return (low);
}
```

중복이 이미 배제됐으므로 lower bound는 해당 값의 유일한 index입니다. 각 원본 위치에 이 index를 기록합니다.

```c
a->ranks[i] = find_rank(sorted, a->size, a->values[i]);
```

예를 들어 입력이 `[42, -7, 100]`이면 다음처럼 됩니다.

```text
values = [42, -7, 100]
ranks  = [ 1,  0,   2]
```

이 표현 전환이 중요한 이유는 기수 정렬이 음수 부호나 값의 폭을 다루지 않아도 되기 때문입니다. 최대 rank는 항상 `n - 1`이며 필요한 bit 수는 입력 개수만으로 결정됩니다.

scratch array는 성공과 duplicate failure 양쪽에서 해제됩니다. `assign_ranks`가 실패하면 `parse_input`이 A 전체를 정리해 부분적으로 rank가 써진 상태를 외부에 노출하지 않습니다.

## 기능 테스트가 고정한 입력 계약

### `4cc9783286c0` — 기본 문법과 오류 표면

첫 Python CLI suite는 다음을 검사합니다.

- 무인자 `push_swap`: status 0, stdout/stderr 없음
- `"3 2"`, `"1"`처럼 quoted와 split argv 혼합 수용
- 생성된 명령을 checker에 넣었을 때 `OK\n`
- duplicate, `INT_MAX + 1`, `INT_MIN - 1`, 숫자 뒤 문자, sign-only, 전체 empty token, argv 경계를 넘는 duplicate의 거절
- 오류 시 stdout 없음, stderr는 정확히 `Error\n`

parser 함수의 return만 보는 단위 테스트가 아니라 실제 binary의 parse → sort → output과 checker 재생까지 통과하는 통합 검사입니다. 반면 각 boundary를 세밀하게 분리하지는 않습니다.

### `44a4da8bc63d` — 해석이 갈리기 쉬운 경계

후속 suite는 유효 입력과 무효 입력을 명시적으로 나눕니다.

| 유효 | 이유 |
| --- | --- |
| `+7` | 하나의 명시적 plus sign 허용 |
| `-0` | 정수 0의 유효 표기 |
| `0003 0002 0001` | leading zero 허용 |
| `3`, `""`, `"2 1"` | empty argv는 다른 token이 있으면 무시 |
| 여섯 종류 whitespace 혼합 | `is_space`의 정확한 집합 |
| `INT_MIN`, `0`, `INT_MAX` | 양 끝값 포함 |

| 무효 | 거절 지점 |
| --- | --- |
| whitespace-only argv 전체 | token count 0 |
| 4096자리 숫자 | `int` magnitude 초과 시 조기 거절 |
| Arabic-Indic digit | ASCII digit 아님 |
| `++1`, `--1`, `+-1` | 부호 뒤에는 digit이 와야 함 |
| `-0`, `+0` 동시 입력 | 숫자 값 기준 duplicate |

각 child process에 5초 timeout도 추가됩니다. 긴 숫자나 잘못된 반복 입력에서 parser가 끝나지 않는 회귀를 test hang 대신 명시적 실패로 바꿉니다.

같은 커밋에는 무인자 checker가 stdin을 읽지 않는다는 검사도 있습니다. seek 가능한 임시 파일에 `sa\n`을 넣고 checker를 인자 없이 실행한 뒤 file position이 0인지 확인합니다. 이는 parser의 empty-input 결과와 checker의 “인자가 없으면 protocol 자체를 시작하지 않는다”는 CLI 규칙이 만나는 지점입니다.

## `049ecd429548` — 의미상 유효한 크기와 계산 가능한 크기를 구분

앞선 기능은 보통 크기의 입력에서는 충분하지만, count와 index가 `int`였기 때문에 매우 큰 argv에서 다음 두 문제가 남았습니다.

1. token 수를 합산하는 동안 signed overflow가 날 수 있음
2. `sizeof(int) * capacity`가 `size_t` 범위를 넘을 수 있음

fix는 문자열 길이와 count 계산을 `size_t`로 옮기고, `int` 기반 stack API로 넘기기 전에 범위를 확인합니다.

```c
static int count_all_tokens(int argc, char **argv, int *result)
{
    size_t count;
    size_t argument_count;

    count = 0;
    ...
    argument_count = count_tokens_in_arg(argv[i]);
    if (argument_count > (size_t)INT_MAX - count)
        return (0);
    count += argument_count;
    ...
    *result = (int)count;
    return (1);
}
```

`count_tokens_in_arg`도 count가 이미 `INT_MAX`이면 `INT_MAX + 1` sentinel을 반환합니다. 따라서 합산이 끝난 뒤 overflow를 발견하는 것이 아니라 **더하기 전에** 거절합니다.

`parse_token`과 `fill_values`의 `start`, `end`, scan index도 `size_t`가 됩니다. 긴 argv의 `ps_strlen` 결과를 먼저 `int`로 축소하던 경로가 사라집니다.

배열 byte 계산은 `stack_init`에서 별도로 방어합니다.

```c
if ((size_t)capacity > (size_t)-1 / sizeof(int))
    return (0);
```

이 검사는 각각의 `values`와 `ranks` allocation 크기 곱셈이 wrap되지 않음을 보장합니다. 실제 메모리가 충분한지는 이후 `ps_malloc` 결과로 판정합니다. 즉 arithmetic validity와 resource availability를 구분합니다.

## 최종 입력 상태

성공한 `parse_input` 뒤 상태는 다음과 같습니다.

```text
0 <= a.size == a.capacity <= INT_MAX
values[0..size) = 입력 token의 숫자 값, 입력 순서 유지
ranks[0..size)  = 0..size-1의 중복 없는 순열
B는 아직 생성되지 않음
parser scratch allocation은 모두 해제됨
```

입력별 최종 결과는 다음과 같습니다.

| 입력 형태 | 결과 |
| --- | --- |
| 프로그램 인자 없음 | 성공, 빈 A |
| empty argv가 유효 token과 혼합 | empty argv 무시, 성공 |
| 모든 argv가 empty/whitespace-only | 실패 |
| optional sign + ASCII digits, int 범위 | 성공 가능 |
| 같은 숫자의 다른 표기 | duplicate로 실패 |
| token 수가 `INT_MAX`를 넘음 | allocation 전 실패 |
| 배열 byte 곱셈이 `size_t`를 넘음 | allocation 전 실패 |
| scratch 또는 stack allocation 실패 | 이미 획득한 자원 정리 후 실패 |

## Thread의 경계

이 Thread는 입력 문자열을 정렬 가능한 state로 바꾸는 데까지 다룹니다.

- rank 배열과 원본 값 배열을 이후 명령이 함께 움직이는 규칙은 스택 연산 Thread의 책임입니다.
- dense rank를 작은 입력 분석과 LSD 기수 정렬에 사용하는 방법은 정렬 엔진 Thread의 책임입니다.
- parser 오류의 `Error\n` 출력이 실제 write failure까지 전달되는지는 runtime Thread에서 완성됩니다.
- 테스트가 사용하는 checker의 명령 frame 규칙은 checker 프로토콜 Thread에서 별도로 강화됩니다.

핵심 결정은 원본 값을 버리는 것이 아니라 **원본과 정렬 key를 나란히 보존**하는 것입니다. 이 덕분에 입력 의미는 유지하면서 정렬 알고리즘은 크기 `n`에만 의존하는 비음수 순위 공간을 사용합니다.
