# Thread: 병렬 스택 상태와 연산 불변 조건

`push_swap`의 원소는 단일 배열 한 칸이 아니라 같은 인덱스의 `(values[i], ranks[i])` 쌍으로 표현됩니다. 원본 정수는 오류 진단과 의미 보존을 위해 남고, 정렬기는 좌표 압축된 순위를 사용합니다. 따라서 모든 스택 연산은 두 배열을 같은 방식으로 이동해야 합니다. 한쪽만 바뀌면 이후 정렬 결과가 맞더라도 어떤 원본 값이 어떤 순위를 가리키는지 복구할 수 없습니다.

이 Thread는 배열 기반 스택의 소유권을 만든 뒤, 11개 명령이 그 쌍을 보존하도록 확장하고, 마지막으로 “원소가 보존되었다”는 넓은 검사와 “정확히 기대한 상태가 되었다”는 좁은 검사를 분리해 고정하는 과정입니다.

> **검사 범위**: 아래 내용은 `c/push_swap` 브랜치에 포함된 각 exact SHA의 diff와 해당 시점 source를 기준으로 작성했습니다. 현재 환경에서는 branch checkout·build·test 실행을 하지 않았으므로, source에 존재하는 테스트의 의도와 assertion만 설명하며 통과 결과를 주장하지 않습니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `96b5324448e4` | `feat(model): 배열 기반 스택 상태를 구현` | S | `ARCH, CORE, STACK_STATE` | 원본 값과 순위를 병렬 배열로 보유하는 상태 표현, 소유권, 완료 판정을 정의 |
| 2 | `c0de1a1b18bb` | `feat(operation): 스택 교환 연산을 구현` | A | `CORE, STACK_STATE, INTEGRATION` | 값·순위 쌍을 함께 교환하고 상태 변경과 명령 출력을 한 wrapper에서 연결 |
| 3 | `73d2deb30224` | `feat(operation): 스택 간 이동 연산을 구현` | B | `CORE, STACK_STATE` | 두 스택 사이에서 top 원소 쌍을 보존하며 이동 |
| 4 | `745ec72850d2` | `feat(operation): 스택 정방향 회전을 구현` | B | `CORE, STACK_STATE` | active prefix의 첫 원소 쌍을 마지막으로 이동 |
| 5 | `68dfd1b1fb58` | `feat(operation): 스택 역방향 회전을 구현` | B | `CORE, STACK_STATE` | active prefix의 마지막 원소 쌍을 처음으로 이동 |
| 6 | `86364d27baac` | `test(operation): 값과 순위의 보존 불변식을 검증` | A | `TEST, STACK_STATE, RISK` | 각 연산과 연속 실행 뒤 pairing·총원소·유일성 보존을 검사 |
| 7 | `7eb6890c2c13` | `test(operation): 정확한 상태 전이와 no-op을 검증` | B | `TEST, STACK_STATE, EDGE` | 각 명령의 정확한 배열 결과와 원소 부족 시 상태 no-op을 검사 |

## `96b5324448e4` — 한 원소를 두 배열의 같은 인덱스로 표현

이 커밋은 이후 모든 알고리즘이 의존하는 상태 표현을 먼저 고정합니다.

```c
typedef struct s_stack
{
    int *values;
    int *ranks;
    int size;
    int capacity;
} t_stack;
```

`values`와 `ranks`는 각각 독립적으로 할당되지만, active range인 `[0, size)`에서는 같은 인덱스가 하나의 논리 원소입니다. `capacity`는 할당된 칸 수이고 `size`는 실제로 스택에 들어 있는 원소 수입니다. 이 구분 덕분에 push는 새 배열을 매번 만들지 않고 이미 확보한 공간 안에서 active prefix만 이동할 수 있습니다.

### 초기화와 정리의 terminal state

```c
void stack_init_empty(t_stack *stack)
{
    stack->values = NULL;
    stack->ranks = NULL;
    stack->size = 0;
    stack->capacity = 0;
}

int stack_init(t_stack *stack, int capacity)
{
    stack_init_empty(stack);
    if (capacity <= 0)
        return (1);
    stack->values = (int *)malloc(sizeof(int) * (size_t)capacity);
    stack->ranks = (int *)malloc(sizeof(int) * (size_t)capacity);
    if (stack->values == NULL || stack->ranks == NULL)
    {
        stack_free(stack);
        return (0);
    }
    stack->capacity = capacity;
    return (1);
}
```

중요한 부분은 두 번째 배열 할당이 실패해도 첫 번째 배열이 남지 않는다는 점입니다. `stack_free`는 두 포인터를 모두 해제한 뒤 다시 empty state로 돌립니다. 따라서 성공한 스택과 부분 초기화된 스택이 서로 다른 정리 코드를 요구하지 않습니다.

`capacity <= 0`은 실패가 아니라 유효한 빈 스택입니다. 무인자 실행에서 A와 B를 빈 상태로 구성하고 아무 명령도 만들지 않는 경로가 이 표현을 사용합니다.

### 정렬 완료의 정의

```c
int stack_is_sorted(const t_stack *stack)
{
    int i;

    i = 1;
    while (i < stack->size)
    {
        if (stack->ranks[i - 1] > stack->ranks[i])
            return (0);
        i++;
    }
    return (1);
}

int stack_is_complete_sorted(const t_stack *a, const t_stack *b)
{
    return (b->size == 0 && stack_is_sorted(a));
}
```

A가 오름차순인 것만으로는 완료가 아닙니다. B가 비어 있어야 모든 원소가 최종 위치로 돌아왔다는 뜻이 됩니다. 이 판정은 후속 checker가 `OK`와 `KO`를 결정할 때 그대로 사용합니다.

이 SHA에서 `ranks`는 아직 좌표 압축 결과를 받지 않습니다. 이 커밋이 정의하는 것은 **표현과 소유권**이며, 실제 `0..n-1` 순위 부여는 입력 Thread의 `e09cf45e21cd`가 담당합니다.

## 연산층 — 상태 변화와 명령 출력을 같은 이름으로 묶기

연산은 두 층으로 나뉩니다.

- `stack_swap`, `stack_push`, `stack_rotate`, `stack_reverse_rotate`: 메모리 상태만 바꿉니다.
- `op_sa`부터 `op_rrr`: 상태 함수를 호출하고 `emit`이 참이면 대응 명령을 출력합니다.

checker는 같은 wrapper를 `emit = 0`으로 호출하고, generator는 `emit = 1`로 호출합니다. 두 프로그램이 별도의 연산 구현을 갖지 않기 때문에 같은 명령의 상태 의미가 갈라질 가능성을 줄입니다.

### `c0de1a1b18bb` — swap은 값과 순위를 함께 저장하고 되돌린다

```c
void stack_swap(t_stack *stack)
{
    int value;
    int rank;

    if (stack->size < 2)
        return ;
    value = stack->values[0];
    rank = stack->ranks[0];
    stack->values[0] = stack->values[1];
    stack->ranks[0] = stack->ranks[1];
    stack->values[1] = value;
    stack->ranks[1] = rank;
}
```

임시 변수도 `(value, rank)` 두 개입니다. 값만 임시 저장하고 순위 배열을 별도로 교환하는 식으로 구현하지 않아, 하나의 논리 원소를 명시적으로 같이 이동합니다.

`op_ss`는 A와 B를 각각 교환한 뒤 `ss\n` 한 줄만 출력합니다. `sa\n`과 `sb\n`을 두 줄 출력하는 구현과 관찰되는 명령 비용이 다르므로, 결합 명령의 출력도 공통 wrapper가 책임집니다.

상태 함수가 원소 부족으로 no-op이 되더라도 wrapper는 `emit = 1`이면 명령을 출력합니다. push_swap 명령은 부족한 스택에 적용되어도 유효한 no-op이므로, “상태가 바뀌었는가”와 “명령을 실행했는가”는 같은 조건이 아닙니다.

### `73d2deb30224` — push는 두 active prefix를 동시에 다시 정의한다

`stack_push(dst, src)`의 상태 전이는 다음과 같습니다.

```text
src = [(v0,r0), (v1,r1), ...]
dst = [(w0,s0), (w1,s1), ...]

push 이후
src = [(v1,r1), ...]
dst = [(v0,r0), (w0,s0), (w1,s1), ...]
```

실제 구현은 destination active prefix를 한 칸 오른쪽으로 이동하고, source top pair를 destination index 0에 복사한 뒤, source active prefix를 왼쪽으로 당깁니다. 두 배열에 같은 길이와 방향의 `memmove`를 적용합니다.

```c
if (dst->size > 0)
{
    memmove(dst->values + 1, dst->values,
        sizeof(int) * (size_t)dst->size);
    memmove(dst->ranks + 1, dst->ranks,
        sizeof(int) * (size_t)dst->size);
}
dst->values[0] = src->values[0];
dst->ranks[0] = src->ranks[0];
dst->size++;
src->size--;
```

이 함수는 destination에 여유 capacity가 있다는 전제를 자체 검사하지 않습니다. 후속 `push_swap`과 checker가 A와 B를 모두 전체 입력 크기만큼 할당하기 때문에 성립하는 caller contract입니다. 이 Thread가 보장하는 것은 그 전제 아래에서 원소 쌍과 총원소 수가 보존된다는 점입니다.

source가 비어 있으면 상태 함수는 즉시 돌아옵니다. `pa`·`pb` wrapper의 출력 규칙은 swap과 같습니다.

### `745ec72850d2`, `68dfd1b1fb58` — 회전은 active range만 순환시킨다

정방향 회전은 top pair를 저장한 뒤 나머지 active range를 앞으로 당기고 저장한 pair를 끝에 놓습니다.

```text
[(x0,r0), (x1,r1), ..., (xn,rn)]
-> [(x1,r1), ..., (xn,rn), (x0,r0)]
```

역방향 회전은 마지막 pair를 저장한 뒤 active range를 뒤로 밀고 index 0에 놓습니다.

```text
[(x0,r0), ..., (xn-1,rn-1), (xn,rn)]
-> [(xn,rn), (x0,r0), ..., (xn-1,rn-1)]
```

두 함수 모두 `size < 2`에서 메모리에 손대지 않습니다. `rr`과 `rrr`은 두 스택의 상태 함수를 각각 호출하되 결합 명령 한 줄만 출력합니다.

이 네 연산 커밋을 합쳐 얻는 공통 규칙은 다음과 같습니다.

| 명령군 | 바뀌는 크기 | 원소 집합 | `(value, rank)` pairing | 원소 부족 시 상태 |
| --- | --- | --- | --- | --- |
| swap | 없음 | 동일 | 보존 | no-op |
| push | A/B 사이에서 ±1 | 두 스택 합집합 동일 | 보존 | source가 비면 no-op |
| rotate | 없음 | 동일 | 보존 | size < 2면 no-op |
| reverse rotate | 없음 | 동일 | 보존 | size < 2면 no-op |

## 두 종류의 테스트가 필요한 이유

### `86364d27baac` — “무엇도 잃거나 섞지 않았다”를 검사

첫 operation test는 다섯 개의 알려진 pair를 A와 B에 나눠 둡니다. 각 연산을 단독으로 실행한 뒤 다음을 확인합니다.

1. `0 <= size <= capacity`
2. 각 active index의 값이 원래 기대한 rank와 짝을 이룸
3. `a.size + b.size == 5`
4. rank 0부터 4까지가 두 스택 전체에서 정확히 한 번씩 존재

```c
if (!stack_pairs_are_valid(&fixture->a)
    || !stack_pairs_are_valid(&fixture->b)
    || fixture->a.size + fixture->b.size != 5
    || !all_pairs_are_present(fixture))
    return (0);
```

11개 명령을 하나씩 검사할 뿐 아니라 하나의 fixture에 연속 적용하면서 매 단계 같은 불변 조건을 다시 확인합니다. 이 방식은 중간 연산에서 pairing이 깨졌다가 우연히 최종 상태에서만 복구되는 경우도 잡습니다.

다만 이 테스트만으로는 `ra`가 실제로 왼쪽 회전했는지, 실수로 아무 일도 하지 않았는지 구별할 수 없습니다. 원소 집합과 pairing은 둘 다 보존되기 때문입니다.

### `7eb6890c2c13` — 정확한 순서와 no-op을 별도로 고정

후속 테스트는 11개 명령 각각에 대해 A/B의 expected values, ranks, size를 정적 표로 둡니다. 따라서 보존 조건을 만족하지만 방향이 반대인 rotate, destination이 뒤바뀐 push, 결합 명령이 한쪽만 적용되는 오류를 구별합니다.

또한 capacity 2의 작은 fixture를 만들고 다음 edge를 검사합니다.

- 두 스택 모두 비어 있는 상태에서 모든 명령
- swap/rotate/reverse rotate 계열에 원소가 하나뿐인 상태
- `pa`의 source인 B가 빈 상태
- `pb`의 source인 A가 빈 상태

검사는 active prefix뿐 아니라 fixture에 미리 채워 둔 두 칸의 value/rank와 capacity까지 그대로인지 비교합니다. 즉 no-op이 size만 유지하고 backing array를 불필요하게 덮어쓰는 구현도 허용하지 않습니다.

두 테스트는 모두 `emit = 0`으로 연산을 호출합니다. 따라서 이 Thread가 직접 검증하는 것은 **상태 의미**이며, 실제 stdout 명령 문자열과 출력 실패 처리는 각각 정렬 통합 Thread와 runtime Thread의 범위입니다.

## 최종 불변 조건

| 불변 조건 | 코드에서 성립하는 지점 | 검증 근거 |
| --- | --- | --- |
| active index 하나는 `(values[i], ranks[i])` 한 쌍이다 | 모든 primitive가 두 배열에 같은 index·길이 변환을 적용 | pair mapping 검사 |
| 스택이 소유한 메모리는 두 배열이며 정리 후 empty state가 된다 | `stack_init` 실패가 `stack_free`로 합류, `stack_free`가 포인터·크기 초기화 | allocation failure Thread가 후속 검증 |
| 연산은 두 스택 전체의 원소 집합을 보존한다 | push도 copy 후 양쪽 size와 prefix를 함께 조정 | 총 size와 rank 유일성 검사 |
| 원소 부족 명령은 상태를 바꾸지 않는다 | primitive의 `size`/source guard | exact no-op fixture |
| 결합 명령은 두 상태 변화를 적용하고 한 명령으로 표현된다 | `op_ss`, `op_rr`, `op_rrr` | exact state test; 출력 문자열 자체는 이 Thread 밖 |
| 완료 상태는 A의 rank 오름차순이면서 B가 비어 있는 상태다 | `stack_is_complete_sorted` | checker verdict가 사용 |

## Thread의 경계

이 Thread는 스택 표현과 명령 의미를 다룹니다. 다음 항목은 여기서 확정하지 않습니다.

- 문자열 입력을 어떤 정수와 순위로 바꾸는지: 입력 문법·좌표 압축 Thread
- 어떤 명령 조합으로 정렬하는지: 정렬 엔진 Thread
- 명령열이 실제로 정렬되는지 독립적으로 재생하는 방법: 정확성·비용 증거 Thread
- stdin에서 명령 frame을 읽고 `OK`/`KO`를 출력하는 규칙: checker 프로토콜 Thread
- `write` 실패 시 이미 바꾼 상태와 출력 stream을 어떻게 종료하는지: runtime 실패 전파 Thread

최종적으로 다른 모든 Thread는 “같은 인덱스의 value와 rank는 분리되지 않는다”는 이 표현 불변 조건을 전제로 동작합니다.
