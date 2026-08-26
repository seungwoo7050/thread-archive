# Thread: 입력 문자열을 유일한 dense rank 상태로 정규화하는 계약
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 02

## 개요

정렬기는 임의의 부호 있는 정수를 그대로 bit-sort하지 않는다. 먼저 여러 `argv`를 하나의 token stream으로 해석하고, 각 token이 정확한 `int`인지 검증한 뒤, 중복 없는 원본 값들을 상대 순위 `0..n-1`로 바꾼다. 성공한 parser의 결과는 원본 의미를 보존하는 `values`와 정렬기가 소비할 `ranks`가 같은 순서로 나란히 놓인 상태다.

이 Thread는 단일 token 문법에서 시작해 공백으로 묶인 인자, 좌표 압축, CLI 경계 테스트로 확장된다. 마지막 fix는 논리적으로 센 token 수가 `int`와 allocation byte 계산에 안전하게 들어가는지 별도로 검사해, “문법상 유효함”과 “표현 가능한 크기임”을 구분한다.

### 최종 상태

| 입력 경로 | parser 결과 |
| --- | --- |
| `argc == 1` | allocation 없는 empty A, 성공 |
| 하나 이상의 유효하고 유일한 정수 token | 원본 `values`와 `0..n-1` dense `ranks`, 성공 |
| 문법 오류·`int` 범위 초과·중복·공백뿐인 전체 입력 | partial allocation 정리 후 실패 |
| token 합 또는 allocation byte 계산이 표현 범위를 넘음 | allocation 전에 실패 |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f36ad8899b5f` | feat(parse): 개별 인자의 부호 있는 정수를 파싱 | B | `INPUT, EDGE` | argv 하나를 부호·digit·`int` 범위가 검증된 정수 하나로 변환 |
| 2 | `3bfb465ebdb1` | feat(parse): 공백으로 결합된 인자 토큰을 처리 | B | `INPUT, EDGE` | 여러 argv와 각 argv 내부 공백을 하나의 token stream으로 통합 |
| 3 | `e09cf45e21cd` | feat(parse): 중복 입력을 거절하고 상대 순위를 계산 | S | `CORE, INPUT, SORT` | 원본 값을 보존하면서 정렬용 dense rank를 만들고 중복을 거절 |
| 4 | `4cc9783286c0` | test(parser): 정상 입력과 오류 입력을 검증 | B | `TEST, INPUT` | 기본 성공·실패 CLI 동작과 생성 명령의 checker 수용을 확인 |
| 5 | `44a4da8bc63d` | test(cli): 입력 경계와 무인자 실행을 검증 | B | `TEST, INPUT, EDGE` | 부호·공백·int 경계·긴 숫자·빈 argv·무인자 checker 경계를 확장 |
| 6 | `049ecd429548` | fix(parse): 토큰 수와 배열 크기 계산을 방어 | A | `INPUT, EDGE, RISK` | token 수 합산, 문자열 index, allocation 곱셈의 overflow를 사전 차단 |

## f36ad8899b5f — feat(parse): 개별 인자의 부호 있는 정수를 파싱
**중요도** `B` · **태그** `INPUT, EDGE`

### 무엇이 바뀌었는가 (diff)

```diff
+static int	parse_token(const char *arg, int start, int end, int *out)
+{
+	long long	value;
+	long long	limit;
+	int			sign;
+	int			i;
+
+	value = 0;
+	sign = 1;
+	i = start;
+	if (arg[i] == '+' || arg[i] == '-')
+	{
+		if (arg[i] == '-')
+			sign = -1;
+		i++;
+	}
+	if (i == end)
+		return (0);
+	limit = INT_MAX;
+	if (sign < 0)
+		limit = (long long)INT_MAX + 1;
+	while (i < end)
+	{
+		if (arg[i] < '0' || arg[i] > '9')
+			return (0);
+		value = value * 10 + (arg[i] - '0');
+		if (value > limit)
+			return (0);
+		i++;
+	}
+	*out = (int)(value * sign);
+	return (1);
+}
```

### 무엇을 준비하는가, 아직 없는 것

문법은 `[+|-]? ASCII_DIGIT+`다. 부호만 있는 token은 거절하고, locale 문자 분류가 아니라 `'0'..'9'`를 직접 검사한다. 음수 magnitude 한계를 `INT_MAX + 1`로 두므로 `INT_MIN`은 허용하지만 그보다 작은 값은 거절한다. 누적값이 작은 범위 초과 지점에서 즉시 실패하므로 매우 긴 숫자도 끝까지 계산하지 않는다.

`parse_input`은 이 시점에 `argc - 1` 크기로 A를 할당하고 각 argv 전체를 정수 하나로 본다. 성공한 값은 임시로 `values[index]`와 `ranks[index]` 양쪽에 복사된다. 아직 공백으로 묶인 여러 token, duplicate rejection, dense rank는 없다.

### 관련 커밋

`3bfb465ebdb1`은 이 token parser를 substring allocation 없이 `[start, end)` 범위에 재사용한다. `e09cf45e21cd`는 임시 mirror였던 `ranks`를 실제 상대 순위로 대체한다.

## 3bfb465ebdb1 — feat(parse): 공백으로 결합된 인자 토큰을 처리
**중요도** `B` · **태그** `INPUT, EDGE`

### 무엇이 바뀌었는가 (diff)

```diff
+static int	is_space(char c)
+{
+	return (c == ' ' || c == '\t' || c == '\n'
+		|| c == '\r' || c == '\v' || c == '\f');
+}
+
+static int	count_all_tokens(int argc, char **argv)
+{
+	int	count;
+	int	i;
+
+	count = 0;
+	i = 1;
+	while (i < argc)
+	{
+		count += count_tokens_in_arg(argv[i]);
+		i++;
+	}
+	return (count);
+}
```

```diff
-	count = argc - 1;
+	count = count_all_tokens(argc, argv);
+	if (count == 0)
+		return (0);
 	if (!stack_init(a, count))
 		return (0);
-	index = 0;
-	while (index < count)
+	if (!fill_values(argc, argv, a))
 	{
-		/* ... */
+		stack_free(a);
+		return (0);
 	}
```

### 무엇을 준비하는가, 아직 없는 것

첫 pass가 전체 token 수를 세고, exact capacity로 A를 한 번 할당한 뒤, 두 번째 pass가 같은 whitespace 규칙으로 token span을 찾아 `parse_token`에 넘긴다. `"3 2" "1"`과 `3 2 1`은 같은 순서를 만들며, 임시 substring을 생성하거나 원본 argv를 수정하지 않는다.

빈 argv 하나는 다른 token과 섞여 있으면 0개 token으로 지나간다. 반대로 제공된 argv 전체가 빈 문자열이나 whitespace뿐이면 `count == 0`으로 실패한다. 이 시점의 count와 string index는 아직 `int`이므로 극단적으로 큰 입력의 narrowing·합산 overflow 가능성은 남는다.

### 관련 커밋

`4cc9783286c0`은 quoted/split argv가 같은 결과를 만드는 기본 CLI 경로를 검사한다. `049ecd429548`은 이 커밋의 `int` count/index를 `size_t` 기반 계산과 명시적 upper bound로 보강한다.

## e09cf45e21cd — feat(parse): 중복 입력을 거절하고 상대 순위를 계산
**중요도** `S` · **태그** `CORE, INPUT, SORT`

### 무엇을 만들었는가 (diff)

```diff
+static int	compare_ints(const void *left, const void *right)
+{
+	int	a;
+	int	b;
+
+	a = *(const int *)left;
+	b = *(const int *)right;
+	return ((a > b) - (a < b));
+}
+
+static int	find_rank(const int *sorted, int size, int value)
+{
+	int	low;
+	int	high;
+	int	mid;
+
+	low = 0;
+	high = size;
+	while (low < high)
+	{
+		mid = low + (high - low) / 2;
+		if (sorted[mid] < value)
+			low = mid + 1;
+		else
+			high = mid;
+	}
+	return (low);
+}
```

```diff
+static int	assign_ranks(t_stack *a)
+{
+	int	*sorted;
+	int	i;
+
+	sorted = (int *)malloc(sizeof(int) * (size_t)a->size);
+	if (sorted == NULL)
+		return (0);
+	i = 0;
+	while (i < a->size)
+	{
+		sorted[i] = a->values[i];
+		i++;
+	}
+	qsort(sorted, (size_t)a->size, sizeof(int), compare_ints);
+	i = 1;
+	while (i < a->size)
+	{
+		if (sorted[i - 1] == sorted[i])
+		{
+			free(sorted);
+			return (0);
+		}
+		i++;
+	}
+	/* ... */
+	free(sorted);
+	return (1);
+}
```

### 설계 결정 — 원본 값은 유지하고 정렬 key만 압축한다

정렬된 임시 복사본에서 인접 값이 같으면 duplicate다. 중복이 없으면 원본 값 하나의 lower-bound index는 유일하며 `0..n-1` 중 하나다. 예를 들어 `[30, -5, 10]`의 정렬 복사본은 `[-5, 10, 30]`, rank는 `[2, 0, 1]`이 된다.

`qsort` comparator는 `a - b` 대신 `(a > b) - (a < b)`를 사용해 `INT_MAX`와 `INT_MIN` 차이에서 signed overflow가 발생하지 않게 한다. binary search의 `mid = low + (high - low) / 2`도 index 합산 overflow를 피하는 형태다.

임시 `sorted`는 성공과 duplicate 실패 양쪽에서 해제된다. `assign_ranks`가 실패하면 `parse_input`이 A의 두 영구 배열까지 정리하므로 caller는 부분적으로 rank가 채워진 상태를 받지 않는다.

### 이 커밋이 아직 다루지 않는 것

좌표 압축은 arbitrary signed value를 입력 크기에만 의존하는 rank 공간으로 바꾸지만, token count와 allocation byte 계산 자체의 representability는 아직 방어하지 않는다. 이 SHA에는 parser 전용 회귀 테스트도 없다.

### 관련 커밋

`1463a193a4f9`은 최대 rank가 `n - 1`이라는 성질로 필요한 bit 수를 계산한다. `4cc9783286c0`과 `44a4da8bc63d`은 duplicate와 정수 경계를 CLI에서 고정한다.

## 4cc9783286c0 — test(parser): 정상 입력과 오류 입력을 검증
**중요도** `B` · **태그** `TEST, INPUT`

### 무엇을 검증하는가

```diff
+def test_parser_inputs():
+    no_args = run([PUSH_SWAP])
+    assert_equal(no_args.returncode, 0, "push_swap without args exits cleanly")
+    assert_equal(no_args.stdout, "", "push_swap without args has no stdout")
+    assert_equal(no_args.stderr, "", "push_swap without args has no stderr")
+
+    valid = run([PUSH_SWAP, "3 2", "1"])
+    checked = run([CHECKER, "3 2", "1"], valid.stdout)
+    assert_equal(checked.stdout, "OK\n", "generated moves sort quoted input")
+
+    invalid_cases = [
+        ["1", "2", "2"], ["2147483648"], ["-2147483649"],
+        ["12a"], ["+"], [""], ["1", "2 1"],
+    ]
```

무인자 실행은 조용히 성공하고, quoted/split 입력은 generator가 만든 명령을 checker에 전달해 `OK`가 되는지 확인한다. duplicate, 범위 초과, 비숫자 suffix, sign-only, 전체 empty, 혼합 입력의 duplicate는 status non-zero·stdout empty·`Error\n`을 요구한다.

### 증명하는 것 / 증명하지 않는 것

이 테스트는 대표적인 public CLI 계약을 고정하지만 모든 whitespace·부호 표현과 정확한 `INT_MIN`/`INT_MAX`는 다루지 않는다. generator와 checker를 함께 사용하므로 두 프로그램이 공유하는 operation bug를 독립적으로 배제하는 테스트도 아니다.

### 관련 커밋

`44a4da8bc63d`은 같은 CLI 기법으로 해석이 갈리기 쉬운 입력 경계를 넓힌다. 정렬 명령의 shared-code 위험은 `5b7559278909`의 Python replay가 별도로 줄인다.

## 44a4da8bc63d — test(cli): 입력 경계와 무인자 실행을 검증
**중요도** `B` · **태그** `TEST, INPUT, EDGE`

### 무엇을 검증하는가

```diff
+valid_cases = [
+    (["+7"], "explicit plus sign"),
+    (["-0"], "negative zero"),
+    (["0003", "0002", "0001"], "leading zeroes"),
+    (["3", "", "2 1"], "empty argv mixed with values"),
+    (["3 2\t1\n0\r-1\v-2\f-3"], "all C whitespace separators"),
+    (["-2147483648", "0", "2147483647"], "exact int bounds"),
+]
+
+invalid_cases = [
+    ([" \t\n\r\v\f"], "whitespace-only argv"),
+    (["9" * 4096], "overlong integer"),
+    (["\u0661"], "non-ASCII digit"),
+    (["++1"], "repeated plus sign"),
+    (["--1"], "repeated minus sign"),
+    (["+-1"], "mixed signs"),
+    (["-0", "+0"], "signed duplicate zero"),
+]
```

subprocess timeout을 추가해 parser나 checker가 malformed input에서 멈추는 회귀도 실패로 바꾼다. 별도 temporary-file case는 값이 없는 checker가 stdin에 준비된 `sa\n`을 읽지 않고 즉시 status 0으로 끝나며 file position이 0에 남는지 확인한다.

### 증명하는 것 / 증명하지 않는 것

이 테스트는 public grammar와 no-values behavior를 넓게 고정한다. 4096자리 숫자는 조기 범위 초과를 관찰하지만, 실제 token count가 `INT_MAX`를 넘거나 `capacity * sizeof(int)`가 `SIZE_MAX`를 넘는 환경을 구성하지는 않는다.

### 관련 커밋

`049ecd429548`은 이 테스트가 현실적으로 만들기 어려운 크기 계산 overflow를 code guard로 막는다. checker가 stdin을 읽는 정확한 frame 규칙은 `05-checker-protocol-and-verdict-hardening`의 범위다.

## 049ecd429548 — fix(parse): 토큰 수와 배열 크기 계산을 방어
**중요도** `A` · **태그** `INPUT, EDGE, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

기존 token counter와 string index는 `int`였다. 개별 argv의 token 수와 전체 합이 커지면 합산 과정에서 overflow한 뒤 작은 capacity로 allocation될 수 있고, 이후 fill pass가 그 범위를 넘어 쓸 위험이 있었다. `stack_init`도 `capacity`를 양수라고만 확인한 뒤 `sizeof(int) * capacity`를 계산했다.

결정은 길이·개수 계산을 `size_t`에서 수행하되, `t_stack.size`와 `capacity`가 `int`인 현재 representation에 들어갈 수 있는지 먼저 검사하는 것이다. allocation byte 곱셈은 별도 guard로 닫는다.

### 무엇이 바뀌었는가 (diff)

```diff
-static int	count_tokens_in_arg(const char *arg)
+static size_t	count_tokens_in_arg(const char *arg)
 {
-	int	count;
-	int	i;
+	size_t	count;
+	size_t	i;
 	/* ... */
+		if (count == (size_t)INT_MAX)
+			return ((size_t)INT_MAX + 1);
 		count++;
 }
```

```diff
-static int	count_all_tokens(int argc, char **argv)
+static int	count_all_tokens(int argc, char **argv, int *result)
 {
-	int	count;
+	size_t	count;
+	size_t	argument_count;
 	/* ... */
-		count += count_tokens_in_arg(argv[i]);
+		argument_count = count_tokens_in_arg(argv[i]);
+		if (argument_count > (size_t)INT_MAX - count)
+			return (0);
+		count += argument_count;
 	/* ... */
-	return (count);
+	*result = (int)count;
+	return (1);
 }
```

```diff
 	if (capacity <= 0)
 		return (1);
+	if ((size_t)capacity > (size_t)-1 / sizeof(int))
+		return (0);
 	stack->values = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
```

### 이 커밋이 보장하는 것 / 보장하지 않는 것

전체 token 수가 `INT_MAX`를 넘으면 A를 할당하기 전에 실패하고, 문자열 span index는 `size_t`로 유지된다. positive capacity라도 byte 곱셈이 `size_t`에 들어가지 않으면 allocation을 시도하지 않는다.

이 guard는 실제 메모리가 충분하다는 뜻이 아니다. 계산이 표현 가능해도 `ps_malloc`은 실패할 수 있으며, 그 경우 기존 all-or-nothing cleanup이 처리한다.

### 관련 커밋

`3bfb465ebdb1`에서 도입한 two-pass tokenization의 count/fill 일치를 크기 안전성까지 확장한 fix다. `63969f770a21`은 계산이 안전한 뒤 실제 각 allocation 획득 지점이 실패할 때 누수가 없는지 별도 fault build로 검사한다.

## 이 Thread의 경계

이 문서는 문자열 입력을 원본 값과 dense rank 상태로 바꾸는 데까지 다룬다. pair를 보존하며 A/B 사이에서 움직이는 명령은 `01-parallel-stack-state-and-operation-invariants`, rank를 tiny/radix strategy에 사용하는 과정은 `03-building-the-sorting-engine`의 몫이다.

parser와 checker가 `Error\n`을 쓰는 도중 발생하는 output failure는 `06-runtime-fault-injection-and-output-failure-propagation`에서 다룬다. checker stdin의 명령 framing은 입력 숫자 grammar와 별개이며 `05-checker-protocol-and-verdict-hardening`에 속한다.

> 검토 범위: `f36ad8899b5f`, `3bfb465ebdb1`, `e09cf45e21cd`, `4cc9783286c0`, `44a4da8bc63d`, `049ecd429548`의 diff와 해당 시점 parser/stack/test source를 확인했다. 로컬 checkout, build, test 실행은 수행하지 않았다.
