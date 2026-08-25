# Thread: 정렬 엔진 구축

입력 Thread가 만든 rank 배열은 중복 없는 `0..n-1` 순열입니다. 정렬 엔진은 이 성질을 이용해 입력 크기에 따라 두 전략을 선택합니다.

- 2~5개: 가능한 상태가 적으므로 직접적인 경우 분석과 최소 rank 추출을 사용합니다.
- 6개 이상: 각 rank의 bit를 낮은 자리부터 분류하는 LSD 기수 정렬을 사용합니다.

두 전략 모두 스택 배열을 직접 고치지 않고 공통 operation wrapper만 호출합니다. 따라서 메모리 상태 변화와 stdout에 보이는 명령열이 같은 실행 순서에서 만들어집니다.

> **검사 범위**: `c/push_swap` 브랜치의 exact SHA diff와 각 시점의 `src/sort.c`, `src/push_swap.c`, Makefile을 검사했습니다. 현재 환경에서는 생성 binary를 실행하지 않았습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `caa54cb306ad` | `feat(sort): 세 개 이하의 스택을 정렬` | B | `CORE, SORT` | 2개 교환과 3개 rank permutation의 직접 정렬 규칙 도입 |
| 2 | `160d1fb8d824` | `feat(sort): 네다섯 개의 스택을 정렬` | B | `CORE, SORT` | 가장 작은 rank를 B로 분리하고 3개 정렬 뒤 복원하는 tiny 전략 완성 |
| 3 | `1463a193a4f9` | `feat(sort): 큰 입력을 기수 정렬로 처리` | S | `CORE, SORT, HARD` | dense rank의 bit를 이용한 안정적인 다중 pass 정렬 구현 |
| 4 | `cf07495c97f7` | `feat(push_swap): 정렬 명령 생성 흐름을 연결` | B | `CORE, INTEGRATION` | parse → B 초기화 → sort → 양쪽 정리를 실제 `push_swap` binary로 연결 |

## `caa54cb306ad` — 3개 입력은 여섯 permutation을 직접 닫는다

두 원소는 top 두 rank가 역순일 때 `sa` 한 번이면 충분합니다.

```c
static void sort_two(t_stack *a)
{
    if (a->ranks[0] > a->ranks[1])
        op_sa(a, 1);
}
```

세 원소는 sorted case를 제외하면 다섯 상태만 남습니다. 구현은 현재 top, middle, bottom rank의 상대 관계를 보고 정해진 명령열을 선택합니다.

| rank 순서 | 선택한 명령 |
| --- | --- |
| `0 1 2` | 없음 |
| `1 0 2` | `sa` |
| `2 1 0` | `sa`, `rra` |
| `2 0 1` | `ra` |
| `0 2 1` | `sa`, `ra` |
| `1 2 0` | `rra` |

핵심 코드는 값 자체가 아니라 rank 관계만 읽습니다.

```c
first = a->ranks[0];
second = a->ranks[1];
third = a->ranks[2];
if (first > second && second < third && first < third)
    op_sa(a, 1);
else if (first > second && second > third)
{
    op_sa(a, 1);
    op_rra(a, 1);
}
/* 나머지 세 unsorted permutation */
```

`sort_stack`은 size 0·1 또는 이미 정렬된 입력에서 즉시 돌아갑니다. 이 early return은 정렬된 입력이 불필요한 명령을 출력하지 않는다는 외부 계약의 시작점입니다.

이 커밋에서는 size 4 이상에 대한 분기가 없습니다. 작은 경우 분석을 먼저 독립적으로 완성하고 후속 전략이 이를 재사용하도록 둡니다.

## `160d1fb8d824` — 4·5개는 가장 작은 원소를 잠시 분리한다

4개와 5개의 모든 permutation을 별도 조건문으로 늘리는 대신, rank가 가장 작은 원소를 A의 top으로 이동한 뒤 B로 보냅니다.

### top까지의 짧은 방향 선택

```c
static void move_index_to_top(t_stack *a, int index)
{
    if (index <= a->size / 2)
    {
        while (index-- > 0)
            op_ra(a, 1);
    }
    else
    {
        index = a->size - index;
        while (index-- > 0)
            op_rra(a, 1);
    }
}
```

index가 앞쪽 절반이면 `ra`, 뒤쪽 절반이면 `rra`를 사용합니다. 정확히 절반인 경우에는 정방향을 선택하지만 두 방향의 명령 수가 같으므로 correctness에는 영향이 없습니다.

### 최소 rank 추출과 복원

```c
target_rank = 0;
while (a->size > 3)
{
    index = find_rank_index(a, target_rank);
    move_index_to_top(a, index);
    op_pb(a, b, 1);
    target_rank++;
}
sort_three(a);
while (b->size > 0)
    op_pa(a, b, 1);
```

- size 4에서는 rank 0 하나를 B로 보냅니다.
- size 5에서는 rank 0, rank 1 순서로 보냅니다.

size 5의 B는 push 특성 때문에 `[1, 0]`이 됩니다. 남은 A의 `[2, 3, 4]`를 정렬한 뒤 `pa`를 두 번 하면 먼저 1, 다음 0이 A top으로 돌아와 `[0, 1, 2, 3, 4]`가 됩니다. 별도의 merge 로직이 필요 없는 이유입니다.

이 전략은 `ranks`가 정확히 `0..n-1`을 한 번씩 포함한다는 입력 계약에 의존합니다. `find_rank_index`가 `-1`을 반환할 가능성을 따로 처리하지 않는 것도 parser가 그 불변 조건을 이미 확정했기 때문입니다.

## `1463a193a4f9` — 큰 입력을 위한 안정적인 LSD 기수 정렬

이 커밋이 전체 정렬 엔진의 일반 해법을 완성합니다. 음수와 큰 정수는 parser에서 dense rank로 바뀌었기 때문에 각 rank를 비음수 이진수로 볼 수 있습니다.

### 필요한 pass 수

```c
static int count_bits(int size)
{
    int bits;
    int max_rank;

    bits = 0;
    max_rank = size - 1;
    while ((max_rank >> bits) != 0)
        bits++;
    return (bits);
}
```

최대 rank가 `size - 1`이므로 그 값을 표현하는 bit 수만 검사하면 됩니다. 원본 정수의 magnitude와 부호는 pass 수에 영향을 주지 않습니다.

### 한 bit pass의 실행

```c
round_size = a->size;
i = 0;
while (i < round_size)
{
    if (((a->ranks[0] >> bit) & 1) == 1)
        op_ra(a, 1);
    else
        op_pb(a, b, 1);
    i++;
}
while (b->size > 0)
    op_pa(a, b, 1);
```

`round_size`를 pass 시작 시점에 고정하는 것이 중요합니다. loop 조건을 현재 `a->size`로 두면 `pb`가 A를 줄일 때 아직 검사하지 않은 원소 일부가 loop 밖에 남습니다. 고정된 횟수만큼 top을 처리하면 pass 시작 당시 A에 있던 모든 원소를 정확히 한 번씩 분류합니다.

bit가 1인 원소는 A의 뒤로 rotate되고, bit가 0인 원소는 B로 push됩니다. 분류가 끝나면 B의 모든 원소를 A로 되돌립니다. 각 pass의 종료 상태는 항상 다음 조건을 만족합니다.

```text
A에는 전체 n개 원소가 있음
B는 비어 있음
해당 bit가 0인 군이 1인 군보다 앞에 있음
각 군 내부의 이전 상대 순서는 보존됨
```

### 왜 안정적인가

A에서 만난 1-bit 원소는 `ra`로 차례대로 뒤에 붙으므로 상대 순서를 유지합니다.

0-bit 원소는 `pb`할 때 B top에 쌓여 한 번 역순이 됩니다. 이후 B가 빌 때까지 `pa`하면 다시 한 번 역순이 되어 A 앞부분에서 원래 만난 순서로 복구됩니다. 따라서 한 pass는 안정적입니다.

낮은 bit부터 처리하는 LSD 정렬에서는 이 안정성이 필수입니다. bit `k`를 처리할 때 bit `0..k-1`로 이미 정해진 상대 순서를 각 0/1 군 내부에 유지해야 하기 때문입니다. 이 조건을 pass마다 반복하면 마지막에는 전체 rank 오름차순이 됩니다.

### 명령 수와 실제 배열 이동 비용은 다르다

코드에서 직접 도출되는 한 pass의 명령 수는 다음과 같습니다.

- 분류: 정확히 `n`개의 `ra` 또는 `pb`
- 복원: 해당 bit가 0인 원소 수만큼 `pa`

따라서 pass당 명령 수는 `n` 이상 `2n` 이하이고, 전체 command count는 대략 `n × bit_count` 규모입니다. 그러나 현재 스택 표현은 배열입니다. `push`와 `rotate`가 `memmove`로 active range를 이동하므로 한 명령의 CPU·메모리 이동 비용은 상수가 아닙니다. 이 차이는 후속 resource test가 “명령 수”와 “이동한 pair 수”를 별도 metric으로 두는 이유입니다.

### 최종 dispatcher

```c
void sort_stack(t_stack *a, t_stack *b)
{
    if (a->size < 2 || stack_is_sorted(a))
        return ;
    if (a->size <= 5)
        sort_tiny(a, b);
    else
        radix_sort(a, b);
}
```

이 SHA에서 sort 함수의 반환형은 아직 `void`입니다. operation 출력이 실패해도 정렬 loop는 이를 알 수 없습니다. 출력 failure가 각 helper에서 `main`까지 올라가도록 반환형을 바꾸는 작업은 runtime Thread의 `315f4b91779b`에서 이루어집니다.

## `cf07495c97f7` — library 조각을 실제 generator로 연결

마지막 커밋은 `push_swap` executable과 entry point를 만듭니다.

```c
int main(int argc, char **argv)
{
    t_stack a;
    t_stack b;

    if (!parse_input(argc, argv, &a))
    {
        write_error();
        return (1);
    }
    if (!stack_init(&b, a.capacity))
    {
        stack_free(&a);
        write_error();
        return (1);
    }
    sort_stack(&a, &b);
    stack_free(&a);
    stack_free(&b);
    return (0);
}
```

B를 A와 같은 capacity로 초기화하므로 operation layer의 “destination에 전체 입력을 담을 여유가 있다”는 전제가 성립합니다. parse failure에서는 A가 parser 내부에서 정리되고, B allocation failure에서는 이미 확보한 A만 정리합니다. 정상 경로는 정렬 뒤 두 스택을 모두 해제합니다.

무인자 입력은 parser가 빈 A를 성공으로 반환하고, capacity 0의 B 초기화도 성공하며, sorter가 바로 돌아옵니다. 결과는 stdout·stderr가 없는 status 0입니다.

이 시점에는 `write_error`와 operation output의 성공 여부를 확인하지 않습니다. 따라서 정렬 state와 명령 생성 흐름은 연결됐지만 “모든 명령이 실제로 출력되었는가”까지는 보장하지 않습니다.

## 최종 실행 흐름

```text
[parse_input]
  원본 values + dense ranks를 가진 A
        ↓
[stack_init(B, A.capacity)]
        ↓
[sort_stack]
  ├─ size < 2 또는 이미 정렬: 명령 없음
  ├─ size <= 5: 최소 rank 분리 + sort_three + 복원
  └─ size > 5: bit 0부터 LSD radix passes
        ↓
[operation wrappers]
  상태를 바꾸고 해당 명령을 stdout에 기록
        ↓
[A와 B 해제]
```

정상 완료 시 기대되는 상태는 `A`의 rank가 `0..n-1` 오름차순이고 `B`가 비어 있는 것입니다. generator 자체는 마지막 상태를 다시 판정하지 않습니다. 알고리즘의 정확성은 source 논리와 별도로 독립 interpreter·checker를 사용하는 테스트 Thread에서 검증합니다.

## 이 Thread가 보장하는 것과 남기는 것

### 보장하는 설계

- 입력 크기 0·1과 이미 정렬된 입력은 명령을 만들지 않습니다.
- 2·3개는 닫힌 경우 분석으로 처리합니다.
- 4·5개는 최소 rank를 분리한 뒤 3개 정렬을 재사용합니다.
- 6개 이상은 모든 bit pass 종료 시 B를 비우는 안정적인 LSD radix를 사용합니다.
- 모든 상태 변화는 공통 operation layer를 통해 이루어집니다.
- A와 B는 전체 입력을 담을 동일 capacity를 갖습니다.

### 이 Thread 밖의 보장

- 작은 모든 permutation과 큰 결정적 입력에서 실제 명령열이 맞는지: 독립 정확성 증거 Thread
- command count, 배열 이동량, peak allocation budget: 비용 증거 Thread
- checker가 명령 stream을 어떤 frame 규칙으로 수용하는지: checker 프로토콜 Thread
- stdout이 짧게 쓰이거나 닫혔을 때 즉시 중단·정리하는지: runtime 실패 전파 Thread

정렬 알고리즘의 핵심 선택은 원본 값의 범위를 다루는 대신 parser가 제공한 순위 공간을 사용하는 것입니다. 이 선택이 작은 입력의 명시적 경우 분석과 큰 입력의 bit 분류를 하나의 operation model 위에 결합합니다.
