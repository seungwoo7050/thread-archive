# Thread: 값과 순위를 하나의 원소로 보존하는 스택 연산
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 01

## 개요

`push_swap`은 원본 정수와 정렬용 순위를 별도 배열에 저장한다. 이 선택은 정렬기가 작은 비음수 rank만 다루게 해주지만, 동시에 새로운 책임을 만든다. `values[i]`와 `ranks[i]`는 독립 데이터가 아니라 하나의 논리 원소이므로, 어느 명령도 둘을 따로 이동해서는 안 된다.

이 Thread는 그 책임이 상태 표현에서 시작해 11개 명령 전체로 확장되고, 마지막에 서로 다른 두 종류의 테스트로 고정되는 과정을 다룬다. 첫 테스트는 원소 쌍과 총원소 보존을 넓게 검사하고, 두 번째 테스트는 각 명령의 정확한 배열 결과와 원소 부족 시 no-op을 좁게 검사한다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `96b5324448e4` | feat(model): 배열 기반 스택 상태를 구현 | S | `ARCH, CORE, STACK_STATE` | 원본 값과 순위를 병렬 배열로 보유하는 상태 표현, 소유권, 완료 판정을 정의 |
| 2 | `c0de1a1b18bb` | feat(operation): 스택 교환 연산을 구현 | A | `CORE, STACK_STATE, INTEGRATION` | 값·순위 쌍을 함께 교환하고 상태 변경과 명령 출력을 한 wrapper에서 연결 |
| 3 | `73d2deb30224` | feat(operation): 스택 간 이동 연산을 구현 | B | `CORE, STACK_STATE` | 두 스택 사이에서 top 원소 쌍을 보존하며 이동 |
| 4 | `745ec72850d2` | feat(operation): 스택 정방향 회전을 구현 | B | `CORE, STACK_STATE` | active prefix의 첫 원소 쌍을 마지막으로 이동 |
| 5 | `68dfd1b1fb58` | feat(operation): 스택 역방향 회전을 구현 | B | `CORE, STACK_STATE` | active prefix의 마지막 원소 쌍을 처음으로 이동 |
| 6 | `86364d27baac` | test(operation): 값과 순위의 보존 불변식을 검증 | A | `TEST, STACK_STATE, RISK` | 각 연산과 연속 실행 뒤 pairing·총원소·유일성 보존을 검사 |
| 7 | `7eb6890c2c13` | test(operation): 정확한 상태 전이와 no-op을 검증 | B | `TEST, STACK_STATE, EDGE` | 각 명령의 정확한 배열 결과와 원소 부족 시 상태 no-op을 검사 |

## 96b5324448e4 — feat(model): 배열 기반 스택 상태를 구현
**중요도** `S` · **태그** `ARCH, CORE, STACK_STATE`

### 무엇을 만들었는가 (diff)

```diff
+typedef struct s_stack
+{
+	int	*values;
+	int	*ranks;
+	int	size;
+	int	capacity;
+}	t_stack;
+
+void	stack_init_empty(t_stack *stack);
+int		stack_init(t_stack *stack, int capacity);
+void	stack_free(t_stack *stack);
+int		stack_is_sorted(const t_stack *stack);
+int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
```

```diff
+void	stack_init_empty(t_stack *stack)
+{
+	stack->values = NULL;
+	stack->ranks = NULL;
+	stack->size = 0;
+	stack->capacity = 0;
+}
+
+int	stack_init(t_stack *stack, int capacity)
+{
+	stack_init_empty(stack);
+	if (capacity <= 0)
+		return (1);
+	stack->values = (int *)malloc(sizeof(int) * (size_t)capacity);
+	stack->ranks = (int *)malloc(sizeof(int) * (size_t)capacity);
+	if (stack->values == NULL || stack->ranks == NULL)
+	{
+		stack_free(stack);
+		return (0);
+	}
+	stack->capacity = capacity;
+	return (1);
+}
```

### 설계 결정 — 두 배열을 하나의 소유 객체로 묶는다

`values`와 `ranks`는 각각 따로 할당되지만 `[0, size)`에서는 같은 인덱스가 반드시 같은 원소를 가리킨다. `capacity`는 확보한 저장 공간, `size`는 현재 살아 있는 prefix를 뜻한다. 이 구분이 있어야 이후 push와 rotate가 재할당 없이 active range만 이동할 수 있다.

초기화는 먼저 empty state를 만든 뒤 두 배열을 순서대로 획득한다. 두 번째 할당이 실패하면 `stack_free`가 첫 번째 배열까지 해제하고 다시 empty state로 돌린다. 성공·부분 실패·명시적 해제가 같은 terminal state를 공유하므로 caller는 별도 정리 분기를 만들 필요가 없다.

정렬 판정은 `ranks`의 비내림차순만 보고, 완료 판정은 여기에 `b->size == 0`을 추가한다. A가 정렬돼도 B에 원소가 남아 있으면 전체 작업은 끝난 것이 아니다.

### 이 커밋이 아직 다루지 않는 것

이 시점에는 parser가 두 배열을 채우지 않고, 어떤 operation도 존재하지 않는다. `ranks`가 `0..n-1`의 순열이 되는 계약은 `e09cf45e21cd`에, 두 배열을 함께 움직이는 명령 의미는 `c0de1a1b18bb`, `73d2deb30224`, `745ec72850d2`, `68dfd1b1fb58`에 속한다. `capacity * sizeof(int)` 계산의 overflow 방어도 아직 없으며 `049ecd429548`에서 보강된다.

### 관련 커밋

`c0de1a1b18bb`는 이 표현을 처음 소비해 top 두 원소 쌍을 함께 교환한다. `86364d27baac`와 `7eb6890c2c13`은 이 커밋에서 생긴 pairing·size·capacity 계약을 각각 보존 관점과 정확한 결과 관점에서 검증한다.

## c0de1a1b18bb — feat(operation): 스택 교환 연산을 구현
**중요도** `A` · **태그** `CORE, STACK_STATE, INTEGRATION`

### 무엇이 바뀌었는가 (diff)

```diff
+static void	emit_op(const char *name, int emit)
+{
+	if (emit)
+		ps_putstr_fd(1, name);
+}
+
+void	stack_swap(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[0];
+	rank = stack->ranks[0];
+	stack->values[0] = stack->values[1];
+	stack->ranks[0] = stack->ranks[1];
+	stack->values[1] = value;
+	stack->ranks[1] = rank;
+}
+
+void	op_sa(t_stack *a, int emit)
+{
+	stack_swap(a);
+	emit_op("sa\n", emit);
+}
```

`op_sb`도 같은 primitive를 B에 적용하고, `op_ss`는 A와 B를 각각 교환한 뒤 `ss\n`을 한 번만 출력한다.

### 설계 결정 — 상태 전이와 직렬화를 분리하되 같은 wrapper에서 연결한다

`stack_swap`은 출력 없는 상태 primitive이고 `op_sa`·`op_sb`·`op_ss`는 선택적으로 명령 문자열을 직렬화하는 public wrapper다. generator는 `emit = 1`, checker는 `emit = 0`으로 같은 상태 의미를 재사용한다. 별도 checker용 swap 구현을 만들지 않으므로 두 실행 파일이 같은 command를 다르게 해석할 위험을 줄인다.

`size < 2`에서는 상태만 no-op이다. 이 SHA의 wrapper는 `emit = 1`이면 실제 변화가 없어도 명령을 출력하며, 출력 helper가 `void`라 write 실패도 표현하지 못한다. 두 문제 중 전자는 command semantics로 허용되고, 후자는 `315f4b91779b`에서 반환 가능한 failure로 바뀐다.

### 관련 커밋

`73d2deb30224`, `745ec72850d2`, `68dfd1b1fb58`은 같은 primitive/wrapper 경계를 push와 두 회전에 그대로 확장한다. `f79ae7e86592`의 checker dispatcher는 이 경계를 `emit = 0`으로 소비한다.

## 73d2deb30224 — feat(operation): 스택 간 이동 연산을 구현
**중요도** `B` · **태그** `CORE, STACK_STATE`

### 무엇이 바뀌었는가 (diff)

```diff
+void	stack_push(t_stack *dst, t_stack *src)
+{
+	if (src->size == 0)
+		return ;
+	if (dst->size > 0)
+	{
+		memmove(dst->values + 1, dst->values,
+			sizeof(int) * (size_t)dst->size);
+		memmove(dst->ranks + 1, dst->ranks,
+			sizeof(int) * (size_t)dst->size);
+	}
+	dst->values[0] = src->values[0];
+	dst->ranks[0] = src->ranks[0];
+	dst->size++;
+	src->size--;
+	if (src->size > 0)
+	{
+		memmove(src->values, src->values + 1,
+			sizeof(int) * (size_t)src->size);
+		memmove(src->ranks, src->ranks + 1,
+			sizeof(int) * (size_t)src->size);
+	}
+}
```

`op_pa(a, b, emit)`는 B의 top을 A로, `op_pb(a, b, emit)`는 A의 top을 B로 옮긴다.

### 무엇을 준비하는가, 아직 없는 것

destination active prefix를 한 칸 뒤로 밀고 source top의 `(value, rank)`를 함께 복사한 뒤, source의 나머지 prefix를 앞으로 당긴다. 두 스택의 size 합은 유지되고 capacity는 바뀌지 않는다. source가 비어 있으면 두 상태 모두 그대로다.

이 함수는 `dst->size < dst->capacity`라는 상위 전제를 직접 검사하지 않는다. A와 B가 입력 크기만큼 같은 capacity를 갖고 총원소 수가 보존된다는 lifecycle 안에서는 넘치지 않지만, primitive 단독 호출에 대한 일반적인 bounds-checking API는 아니다.

### 관련 커밋

`86364d27baac`은 push 뒤에도 모든 rank가 A와 B 전체에서 정확히 한 번 존재하는지 검사한다. `160d1fb8d824`와 `1463a193a4f9`은 각각 tiny sorting과 radix partition에서 `pb`/`pa`를 핵심 이동 수단으로 사용한다.

## 745ec72850d2 — feat(operation): 스택 정방향 회전을 구현
**중요도** `B` · **태그** `CORE, STACK_STATE`

### 무엇이 바뀌었는가 (diff)

```diff
+void	stack_rotate(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[0];
+	rank = stack->ranks[0];
+	memmove(stack->values, stack->values + 1,
+		sizeof(int) * (size_t)(stack->size - 1));
+	memmove(stack->ranks, stack->ranks + 1,
+		sizeof(int) * (size_t)(stack->size - 1));
+	stack->values[stack->size - 1] = value;
+	stack->ranks[stack->size - 1] = rank;
+}
```

`op_ra`와 `op_rb`는 한 스택에 적용하고, `op_rr`은 두 스택을 각각 회전한 뒤 명령 한 줄만 출력한다.

### 무엇을 준비하는가, 아직 없는 것

정방향 회전은 active prefix의 top pair를 보관하고 나머지를 한 칸 앞으로 당긴 뒤 보관한 pair를 마지막에 쓴다. size·capacity·소유권은 변하지 않는다. 이 동작은 radix pass에서 bit가 1인 원소의 상대 순서를 A 안에서 유지하는 수단이 된다.

이 커밋은 반대 방향 회전이 없어 tiny sorter가 bottom 쪽 원소를 짧게 top으로 올리는 경우를 아직 표현하지 못한다.

### 관련 커밋

`68dfd1b1fb58`은 같은 active-range 계약의 반대 방향을 추가한다. `1463a193a4f9`은 `ra`를 stable partition의 한 축으로 사용한다.

## 68dfd1b1fb58 — feat(operation): 스택 역방향 회전을 구현
**중요도** `B` · **태그** `CORE, STACK_STATE`

### 무엇이 바뀌었는가 (diff)

```diff
+void	stack_reverse_rotate(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[stack->size - 1];
+	rank = stack->ranks[stack->size - 1];
+	memmove(stack->values + 1, stack->values,
+		sizeof(int) * (size_t)(stack->size - 1));
+	memmove(stack->ranks + 1, stack->ranks,
+		sizeof(int) * (size_t)(stack->size - 1));
+	stack->values[0] = value;
+	stack->ranks[0] = rank;
+}
```

`op_rra`, `op_rrb`, `op_rrr`이 추가되면서 11개 command vocabulary가 완성된다.

### 무엇을 준비하는가, 아직 없는 것

역방향 회전은 active prefix의 마지막 pair를 top으로 올린다. `160d1fb8d824`는 target rank가 bottom에 가까우면 `ra`를 여러 번 호출하는 대신 `rra`를 선택한다.

이 시점까지 각 primitive의 구현은 존재하지만 결과가 정확한지 자동으로 검사하는 source는 없다. 그 증거는 다음 두 test commit이 서로 다른 관점으로 추가한다.

### 관련 커밋

`160d1fb8d824`은 forward/reverse rotation 중 더 짧은 방향을 선택한다. `86364d27baac`과 `7eb6890c2c13`은 회전을 포함한 11개 명령 전체를 각각 보존성과 exact state 관점에서 고정한다.

## 86364d27baac — test(operation): 값과 순위의 보존 불변식을 검증
**중요도** `A` · **태그** `TEST, STACK_STATE, RISK`

### 무엇을 검증하는가

```diff
+static int	stack_pairs_are_valid(const t_stack *stack)
+{
+	int	i;
+
+	if (stack->size < 0 || stack->size > stack->capacity)
+		return (0);
+	i = 0;
+	while (i < stack->size)
+	{
+		if (expected_rank(stack->values[i]) != stack->ranks[i])
+			return (0);
+		i++;
+	}
+	return (1);
+}
+
+static int	fixture_is_valid(const char *operation,
+		const t_fixture *fixture)
+{
+	if (!stack_pairs_are_valid(&fixture->a)
+		|| !stack_pairs_are_valid(&fixture->b)
+		|| fixture->a.size + fixture->b.size != 5
+		|| !all_pairs_are_present(fixture))
+		return (0);
+	return (1);
+}
```

각 명령을 독립 fixture에 한 번 적용하는 경우와 11개 명령을 연속 적용하는 경우 모두에서 다음을 검사한다.

- `(value, rank)` 대응이 유지된다.
- 각 stack의 `0 <= size <= capacity`가 유지된다.
- A와 B의 size 합이 5로 유지된다.
- rank 0부터 4까지가 두 스택 전체에 정확히 한 번 존재한다.

### 증명하는 것 / 증명하지 않는 것

이 테스트는 원소 손실·중복·pair 분리를 잡지만, 각 명령이 올바른 순서로 배치했는지는 판정하지 않는다. 예를 들어 rotate가 pair 전체를 잘못된 위치로 옮겨도 모든 원소가 한 번씩 존재하면 통과할 수 있다. 또한 `emit = 0`만 사용하므로 출력 문자열과 write failure는 범위 밖이다.

### 관련 커밋

`7eb6890c2c13`은 이 테스트가 의도적으로 보지 않는 exact post-state와 no-op을 추가한다. 두 테스트는 대체 관계가 아니라 보존 invariant와 명령별 semantics를 나눠 맡는다.

## 7eb6890c2c13 — test(operation): 정확한 상태 전이와 no-op을 검증
**중요도** `B` · **태그** `TEST, STACK_STATE, EDGE`

### 무엇을 검증하는가

```diff
+typedef struct s_expected_fixture
+{
+	int	a_values[5];
+	int	a_ranks[5];
+	int	a_size;
+	int	b_values[5];
+	int	b_ranks[5];
+	int	b_size;
+}	t_expected_fixture;
+
+static const t_expected_fixture	expected[TEST_OPERATION_COUNT] = {
+	{{10, 40, 30}, {1, 4, 3}, 3, {20, 0}, {2, 0}, 2},
+	{{40, 10, 30}, {4, 1, 3}, 3, {0, 20}, {0, 2}, 2},
+	/* ... */
+};
```

```diff
+static int	test_operation_noops(void)
+{
+	int	index;
+
+	index = 0;
+	while (index < TEST_OPERATION_COUNT)
+	{
+		if (!run_noop_case((t_test_operation)index, 0, 0))
+			return (0);
+		if (index == TEST_PA)
+		{
+			if (!run_noop_case(TEST_PA, 1, 0))
+				return (0);
+		}
+		else if (index == TEST_PB)
+		{
+			if (!run_noop_case(TEST_PB, 0, 1))
+				return (0);
+		}
+		else if (!run_noop_case((t_test_operation)index, 1, 1))
+			return (0);
+		index++;
+	}
+	return (1);
+}
```

11개 명령 각각에 대해 A/B의 `values`, `ranks`, `size`, `capacity`를 기대값과 비교한다. 별도 small fixture는 source가 비었거나 swap/rotate 대상이 2개 미만일 때 두 stack의 저장 내용과 크기가 바뀌지 않는지 확인한다.

### 증명하는 것 / 증명하지 않는 것

이 테스트는 고정 fixture에서 명령별 정확한 전이와 원소 부족 시 상태 no-op을 증명한다. arbitrary capacity, 모든 가능한 배열 내용, `emit = 1`의 stdout 동작까지 완전 탐색하지는 않는다.

### 관련 커밋

`86364d27baac`의 넓은 보존 검사와 함께 사용해야 “정확한 위치”와 “전체 원소 보존”을 모두 확인할 수 있다. sorter의 명령열 전체가 최종 정렬을 만드는지는 `5b7559278909`의 독립 Python replay가 별도로 검증한다.

## 이 Thread의 경계

이 문서는 stack representation과 command state transition만 다룬다. 입력 문자열을 원본 값과 dense rank로 만드는 과정은 `02-input-grammar-coordinate-compression-and-size-safety`의 몫이고, 이 명령들을 조합해 tiny/radix sorting을 만드는 과정은 `03-building-the-sorting-engine`이 담당한다.

명령열 전체의 최종 정렬 결과와 비용은 `04-independent-correctness-and-cost-evidence`, stdin command framing과 `OK`/`KO` 판정은 `05-checker-protocol-and-verdict-hardening`에서 다룬다. output failure가 operation 반환값으로 바뀌는 후속 계약은 `06-runtime-fault-injection-and-output-failure-propagation`의 범위다.

> 검토 범위: `96b5324448e4`, `c0de1a1b18bb`, `73d2deb30224`, `745ec72850d2`, `68dfd1b1fb58`, `86364d27baac`, `7eb6890c2c13`의 diff와 해당 시점 source/test를 확인했다. 로컬 checkout, build, test 실행은 수행하지 않았다.
