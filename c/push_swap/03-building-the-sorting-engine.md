# Thread: 작은 상태 분석과 안정적 기수 분류의 전략 분리
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 03

## 개요

parser가 만든 rank는 중복 없는 `0..n-1` 순열이다. 정렬 엔진은 이 제한된 상태 공간을 입력 크기에 따라 다르게 사용한다. 2~5개에서는 가능한 상태를 직접 닫거나 최소 rank를 잠시 B로 분리하고, 6개 이상에서는 낮은 bit부터 안정적으로 partition하는 LSD 기수 정렬을 사용한다.

두 전략은 stack 배열을 직접 수정하지 않고 operation wrapper만 호출한다. 따라서 알고리즘이 선택한 state transition과 stdout에 기록되는 command sequence가 같은 순서로 진행된다. 마지막 integration commit은 이 알고리즘을 parser·B 초기화·cleanup과 연결하지만, 알고리즘의 핵심 판단 자체는 앞의 세 커밋에 있다.

### 최종 전략

| 입력 크기 | 전략 | pass 종료 조건 |
| ---: | --- | --- |
| 0·1 또는 이미 정렬 | 아무 명령도 생성하지 않음 | A 그대로, B empty |
| 2·3 | rank permutation 직접 분기 | A 정렬, B empty |
| 4·5 | 최소 rank를 B로 보내 3개 문제로 축소한 뒤 복원 | A 정렬, B empty |
| 6 이상 | dense rank의 LSD binary radix | 매 bit pass마다 B를 다시 비움 |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `caa54cb306ad` | feat(sort): 세 개 이하의 스택을 정렬 | B | `CORE, SORT` | 2개 교환과 3개 rank permutation의 직접 정렬 규칙 도입 |
| 2 | `160d1fb8d824` | feat(sort): 네다섯 개의 스택을 정렬 | B | `CORE, SORT` | 가장 작은 rank를 B로 분리하고 3개 정렬 뒤 복원하는 tiny 전략 완성 |
| 3 | `1463a193a4f9` | feat(sort): 큰 입력을 기수 정렬로 처리 | S | `CORE, SORT, HARD` | dense rank의 bit를 이용한 안정적인 다중 pass 정렬 구현 |
| 4 | `cf07495c97f7` | feat(push_swap): 정렬 명령 생성 흐름을 연결 | B | `CORE, INTEGRATION` | parse → B 초기화 → sort → 양쪽 정리를 실제 `push_swap` binary로 연결 |

## caa54cb306ad — feat(sort): 세 개 이하의 스택을 정렬
**중요도** `B` · **태그** `CORE, SORT`

### 무엇이 바뀌었는가 (diff)

```diff
+static void	sort_two(t_stack *a)
+{
+	if (a->ranks[0] > a->ranks[1])
+		op_sa(a, 1);
+}
+
+static void	sort_three(t_stack *a)
+{
+	int	first;
+	int	second;
+	int	third;
+
+	if (stack_is_sorted(a))
+		return ;
+	first = a->ranks[0];
+	second = a->ranks[1];
+	third = a->ranks[2];
+	if (first > second && second < third && first < third)
+		op_sa(a, 1);
+	else if (first > second && second > third)
+	{
+		op_sa(a, 1);
+		op_rra(a, 1);
+	}
+	else if (first > second && second < third && first > third)
+		op_ra(a, 1);
+	else if (first < second && second > third && first < third)
+	{
+		op_sa(a, 1);
+		op_ra(a, 1);
+	}
+	else if (first < second && second > third && first > third)
+		op_rra(a, 1);
+}
```

### 무엇을 준비하는가, 아직 없는 것

두 원소는 역순일 때 `sa` 한 번이면 끝난다. 세 원소는 sorted case를 제외한 다섯 permutation을 rank 관계로 직접 분기한다.

| top → bottom rank | command sequence |
| --- | --- |
| `1 0 2` | `sa` |
| `2 1 0` | `sa`, `rra` |
| `2 0 1` | `ra` |
| `0 2 1` | `sa`, `ra` |
| `1 2 0` | `rra` |

`sort_stack`은 size 0·1 또는 이미 정렬된 입력에서 즉시 반환한다. 이 early return이 정렬된 입력의 stdout을 비워 두는 외부 동작으로 이어진다. size 4 이상에는 아직 어떤 경로도 없으므로 이 커밋은 완전한 sorter가 아니라 bounded state-space의 첫 조각이다.

### 관련 커밋

`160d1fb8d824`은 4·5개 입력을 이 3개 정렬 문제로 축소해 재사용한다. `5b7559278909`은 size 2부터 5까지 모든 permutation을 독립 replay해 이 분기와 후속 tiny 전략을 함께 고정한다.

## 160d1fb8d824 — feat(sort): 네다섯 개의 스택을 정렬
**중요도** `B` · **태그** `CORE, SORT`

### 무엇이 바뀌었는가 (diff)

```diff
+static int	find_rank_index(const t_stack *stack, int rank)
+{
+	int	i;
+
+	i = 0;
+	while (i < stack->size)
+	{
+		if (stack->ranks[i] == rank)
+			return (i);
+		i++;
+	}
+	return (-1);
+}
+
+static void	move_index_to_top(t_stack *a, int index)
+{
+	if (index <= a->size / 2)
+	{
+		while (index-- > 0)
+			op_ra(a, 1);
+	}
+	else
+	{
+		index = a->size - index;
+		while (index-- > 0)
+			op_rra(a, 1);
+	}
+}
```

```diff
+static void	sort_tiny(t_stack *a, t_stack *b)
+{
+	int	target_rank;
+	int	index;
+
+	if (a->size == 2)
+	{
+		sort_two(a);
+		return ;
+	}
+	target_rank = 0;
+	while (a->size > 3)
+	{
+		index = find_rank_index(a, target_rank);
+		move_index_to_top(a, index);
+		op_pb(a, b, 1);
+		target_rank++;
+	}
+	sort_three(a);
+	while (b->size > 0)
+		op_pa(a, b, 1);
+}
```

### 무엇을 준비하는가, 아직 없는 것

현재 A에서 다음 최소 rank의 index를 찾고, top에서 가까우면 `ra`, bottom에서 가까우면 `rra`를 선택한다. size 4는 rank 0 하나, size 5는 rank 0과 1을 차례로 B에 보낸다. size 5에서 두 번째 `pb` 뒤 B는 top부터 `1, 0`이므로, 남은 세 원소를 정렬한 뒤 `pa`를 반복하면 1이 먼저, 0이 나중에 A top으로 올라 최종 prefix가 `0, 1`이 된다.

`find_rank_index`가 `-1`을 반환하는 경우를 caller가 따로 처리하지 않는다. parser가 unique dense rank를 만들었다는 전제에서는 target rank가 반드시 A에 존재하지만, 이 함수만 떼어 놓은 일반 방어 API는 아니다. size 6 이상에는 여전히 경로가 없다.

### 관련 커밋

`e09cf45e21cd`의 dense-rank bijection이 target rank의 존재를 보장한다. `1463a193a4f9`은 factorial하게 늘어나는 직접 분기를 더 확장하지 않고 일반 크기용 bit strategy를 추가한다.

## 1463a193a4f9 — feat(sort): 큰 입력을 기수 정렬로 처리
**중요도** `S` · **태그** `CORE, SORT, HARD`

### 무엇을 만들었는가 (diff)

```diff
+static int	count_bits(int size)
+{
+	int	bits;
+	int	max_rank;
+
+	bits = 0;
+	max_rank = size - 1;
+	while ((max_rank >> bits) != 0)
+		bits++;
+	return (bits);
+}
+
+static void	radix_sort(t_stack *a, t_stack *b)
+{
+	int	bits;
+	int	bit;
+	int	i;
+	int	round_size;
+
+	bits = count_bits(a->size);
+	bit = 0;
+	while (bit < bits)
+	{
+		round_size = a->size;
+		i = 0;
+		while (i < round_size)
+		{
+			if (((a->ranks[0] >> bit) & 1) == 1)
+				op_ra(a, 1);
+			else
+				op_pb(a, b, 1);
+			i++;
+		}
+		while (b->size > 0)
+			op_pa(a, b, 1);
+		bit++;
+	}
+}
```

### 설계 결정 — 각 pass를 안정적인 partition으로 만든다

최대 rank는 `size - 1`이므로 그 값을 표현하는 bit 수만 계산한다. 각 pass 시작의 `a->size`를 `round_size`에 고정하는 것이 중요하다. loop 조건을 현재 `a->size`로 두면 `pb`가 size를 줄일 때 시작 원소 일부를 검사하지 못한다.

현재 bit가 1이면 `ra`로 A의 뒤에 보내고, 0이면 `pb`로 B에 보낸다. 1-group은 rotation 순서가 유지된다. 0-group은 B에 들어갈 때 한 번 역순이 되고, B 전체를 `pa`로 되돌릴 때 다시 역순이 되어 원래 상대 순서를 회복한다. 따라서 pass는 lower bits에서 이미 정해진 순서를 깨지 않는 stable partition이다.

예를 들어 bit 0을 처리한 뒤 A가 `0-group + 1-group` 순서가 되고 B는 비어 있다. 다음 bit에서도 같은 안정성이 유지되므로 마지막 bit까지 처리하면 전체 rank가 오름차순이 된다. 매 pass 종료 시 B를 비우는 조건은 다음 pass의 시작 상태와 최종 completion predicate를 단순하게 만든다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

unique dense rank와 정상 operation을 전제로 size 6 이상을 결정적으로 정렬한다. command 수는 대략 원소 수와 bit 수의 곱에 비례하지만, array-backed push/rotate가 내부에서 `memmove`하는 pair 수까지 같은 복잡도라는 뜻은 아니다. 그 물리 비용은 `6569949742eb`에서 별도 metric으로 측정한다.

### 관련 커밋

`e09cf45e21cd`가 rank 범위를 `0..n-1`로 제한했기 때문에 signed 원본 값이나 32개 bit 전체를 다룰 필요가 없다. `5b7559278909`은 Python list model로 대표 radix 입력을 재생하고, `a16dde75d935`와 `6569949742eb`은 command와 movement 비용을 각각 고정한다.

## cf07495c97f7 — feat(push_swap): 정렬 명령 생성 흐름을 연결
**중요도** `B` · **태그** `CORE, INTEGRATION`

### 무엇이 바뀌었는가 (diff)

```diff
+NAME := push_swap
 
-PUSH_SRCS := src/sort.c
+PUSH_SRCS := src/push_swap.c src/sort.c
 
-all: $(COMMON_OBJS) $(PUSH_OBJS)
+all: $(COMMON_OBJS) $(PUSH_OBJS) $(NAME)
+
+$(NAME): $(COMMON_OBJS) $(PUSH_OBJS)
+	$(CC) $(CFLAGS) $(COMMON_OBJS) $(PUSH_OBJS) -o $@
```

```diff
+int	main(int argc, char **argv)
+{
+	t_stack	a;
+	t_stack	b;
+
+	if (!parse_input(argc, argv, &a))
+	{
+		write_error();
+		return (1);
+	}
+	if (!stack_init(&b, a.capacity))
+	{
+		stack_free(&a);
+		write_error();
+		return (1);
+	}
+	sort_stack(&a, &b);
+	stack_free(&a);
+	stack_free(&b);
+	return (0);
+}
```

### 왜 가볍게 다루는가

이 커밋의 주된 역할은 이미 존재하는 parser와 sorter를 executable lifecycle로 연결하는 것이다. parse 성공 뒤 A를 소유하고, B allocation까지 성공하면 A/B 둘을 소유하며, 정상 정렬 뒤 둘 다 해제한다. B 초기화가 실패하면 A만 정리하고 `Error\n`과 status 1을 반환한다.

이 SHA의 `sort_stack`과 output helper는 실패를 반환하지 않으므로 in-memory sort가 끝나면 main은 stdout 전달 성공 여부와 무관하게 status 0을 반환한다. 이 integration limitation은 `315f4b91779b`에서 sorter return value와 write-all contract로 바뀐다.

### 관련 커밋

`1463a193a4f9`까지 만든 두 전략을 실제 command generator로 공개한다. `5b7559278909`은 이 binary의 stdout을 독립적으로 재생하고, `315f4b91779b`은 같은 call path에 output failure를 끝까지 전파한다.

## 이 Thread의 경계

이 문서는 rank를 어떤 command sequence로 정렬하는지만 다룬다. rank를 만드는 입력 계약은 `02-input-grammar-coordinate-compression-and-size-safety`, operation 하나가 `(value, rank)` pair를 정확히 이동하는 규칙은 `01-parallel-stack-state-and-operation-invariants`에 속한다.

생성된 stream의 독립 correctness와 command/movement budget은 `04-independent-correctness-and-cost-evidence`, checker의 frame protocol은 `05-checker-protocol-and-verdict-hardening`이 담당한다. stdout failure 뒤 sorter를 중단하는 동작은 `06-runtime-fault-injection-and-output-failure-propagation`의 범위다.

> 검토 범위: `caa54cb306ad`, `160d1fb8d824`, `1463a193a4f9`, `cf07495c97f7`의 diff와 해당 시점 `src/sort.c`, `src/push_swap.c`, Makefile을 확인했다. 로컬 checkout, build, 실행은 수행하지 않았다.
