# Thread: 생성기와 독립된 정답 판정 및 비용 측정
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 04

## 개요

`push_swap`과 product checker는 같은 C operation layer를 재사용한다. 이 구조는 command semantics를 한곳에 모으지만, 같은 결함을 generator와 checker가 함께 공유하면 잘못된 stream을 둘 다 정상으로 판단할 수 있다. 이 Thread는 Python list model로 command를 다시 구현해 그 common-mode risk를 줄이는 데서 시작한다.

그 뒤 검증은 서로 다른 질문으로 분리된다. 큰 입력의 command ceiling, 명시적 fixture 생성과 stream 재현성, array representation이 만드는 pair 이동량과 peak allocation, ASan/UBSan 아래에서의 memory/runtime 진단은 같은 “성능 테스트”가 아니다. 각 commit은 측정값과 관찰 방법을 분리해, 한 지표의 성공이 다른 속성의 증명처럼 보이지 않게 한다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `5b7559278909` | test(sort): 생성 명령의 정렬 결과를 독립 검증 | A | `TEST, SORT, RISK` | Python reference model로 명령을 재생하고 작은 입력을 exhaustive하게 검증 |
| 2 | `a16dde75d935` | test(sort): 큰 입력의 명령 수 상한을 검증 | B | `TEST, PERF, SORT` | 100·500개 입력에서 correctness 뒤 명령 수 ceiling을 확인 |
| 3 | `23198a9cdd55` | test(sort): 결정적 다중 시드 동치 검사를 추가 | B | `TEST, SORT` | 명시적 PRNG·shuffle로 fixture를 고정하고 같은 입력의 명령열 재현성을 검사 |
| 4 | `6569949742eb` | test(resource): 명령과 배열 이동 및 할당량을 기준화 | A | `TEST, RESOURCE, PERF` | 출력 명령 수, pair 이동량, peak bytes, live allocation을 별도 metric으로 고정 |
| 5 | `5505adf3e469` | build(sanitize): C99 sanitizer 검증 경로를 추가 | B | `TEST, RUNTIME, PRACTICAL` | 일반 binary와 분리된 ASan/UBSan build로 operation·functional suite를 재실행 |

## 5b7559278909 — test(sort): 생성 명령의 정렬 결과를 독립 검증
**중요도** `A` · **태그** `TEST, SORT, RISK`

### 왜 다른 기법이 필요한가

product checker는 generator와 같은 `op_*` 함수를 사용한다. 예를 들어 두 프로그램이 동일하게 잘못된 rotate를 공유하면 generator가 만든 stream을 checker가 그대로 `OK`로 판정할 수 있다. 따라서 “실제 checker가 받아들였다”만으로는 operation layer와 독립된 correctness evidence가 되지 않는다.

### 무엇을 검증하는가 (diff)

```diff
+VALID_MOVES = {"sa", "sb", "ss", "pa", "pb", "ra", "rb", "rr", "rra", "rrb", "rrr"}
+
+def apply_moves(values, moves):
+    a = list(values)
+    b = []
+
+    def swap(stack):
+        if len(stack) > 1:
+            stack[0], stack[1] = stack[1], stack[0]
+
+    def push(dst, src):
+        if src:
+            dst.insert(0, src.pop(0))
+
+    def rotate(stack):
+        if len(stack) > 1:
+            stack.append(stack.pop(0))
+
+    def reverse_rotate(stack):
+        if len(stack) > 1:
+            stack.insert(0, stack.pop())
+
+    /* ... */
+    return a, b
```

```diff
+def assert_sorted_by_program(values):
+    args = [str(value) for value in values]
+    result = run([PUSH_SWAP] + args)
+    assert_equal(result.returncode, 0, f"push_swap sorts {values!r}")
+    assert_equal(result.stderr, "", f"push_swap sort stderr for {values!r}")
+    moves = split_moves(result.stdout)
+    a, b = apply_moves(values, moves)
+    assert_equal(a, sorted(values), f"move stream sorts stack {values!r}")
+    assert_equal(b, [], f"move stream leaves stack b empty for {values!r}")
+    checked = run([CHECKER] + args, result.stdout)
+    assert_equal(checked.returncode, 0, f"checker validates stream for {values!r}")
+    assert_equal(checked.stdout, "OK\n", f"checker OK for {values!r}")
+    return moves
```

Python model은 C의 병렬 배열과 `memmove`를 사용하지 않는다. stdout의 각 줄이 11개 command 중 하나인지 먼저 제한하고, Python list에서 재생한 A가 numeric sort와 같고 B가 비었는지 확인한 뒤 같은 stream을 product checker에도 전달한다.

size 2부터 5까지 모든 permutation을 실행한다. 경우의 수는 `2! + 3! + 4! + 5! = 152`개이며 tiny sorter의 전체 입력 순서를 닫는다. 고정 fixture는 음수·`INT_MIN`·`INT_MAX`, size 6·10을 포함해 radix 경로도 대표적으로 통과한다. 이미 정렬된 5개 입력은 command가 0개여야 한다.

### 증명하는 것 / 증명하지 않는 것

이 테스트는 선택한 큰 입력과 모든 tiny permutation에서 command vocabulary, 독립 final state, B-empty, product checker integration을 함께 확인한다. Python model 자체의 결함 가능성까지 제거하지는 않지만 shared C implementation의 common-mode risk를 줄인다.

모든 size 6 이상 permutation, 최소 command 수, allocation·read·write failure, undefined behavior 부재는 증명하지 않는다.

### 관련 커밋

`a16dde75d935`는 이 correctness helper를 먼저 통과한 stream에만 command ceiling을 적용한다. `6569949742eb`은 command count만으로 보이지 않는 array movement와 peak allocation을 별도 측정한다.

## a16dde75d935 — test(sort): 큰 입력의 명령 수 상한을 검증
**중요도** `B` · **태그** `TEST, PERF, SORT`

### 무엇을 검증하는가

```diff
+def test_move_counts():
+    random.seed(4242)
+    limits = [
+        (100, 1500),
+        (500, 8000),
+    ]
+    for size, limit in limits:
+        values = random.sample(range(-10000, 10000), size)
+        moves = assert_sorted_by_program(values)
+        assert_ok(
+            len(moves) <= limit,
+            f"{size} value move count {len(moves)} exceeds {limit}",
+        )
```

wall-clock time이 아니라 stdout command line 수를 pass/fail 기준으로 사용한다. `assert_sorted_by_program`을 먼저 호출하므로 아무 명령도 내지 않는 잘못된 program이 “빠르다”는 이유로 통과하지 않는다.

### 증명하는 것 / 증명하지 않는 것

fixed-seed로 만든 100·500개 fixture에서 correctness와 각각 1500·8000 command ceiling을 확인한다. 모든 permutation의 worst case나 CPU time, `memmove`된 pair 수, Python 버전에 독립적인 fixture specification은 아직 고정하지 않는다.

### 관련 커밋

`23198a9cdd55`는 Python `random` 구현에 기대던 fixture를 명시적인 32-bit state transition과 shuffle로 교체하고 seed를 늘린다. command ceiling과 physical movement의 차이는 `6569949742eb`이 드러낸다.

## 23198a9cdd55 — test(sort): 결정적 다중 시드 동치 검사를 추가
**중요도** `B` · **태그** `TEST, SORT`

### 무엇이 바뀌었는가 (diff)

```diff
-import random
+import os
 
-PUSH_SWAP = ROOT / "push_swap"
-CHECKER = ROOT / "checker"
+PUSH_SWAP = ROOT / os.environ.get("PS_PUSH_SWAP", "push_swap")
+CHECKER = ROOT / os.environ.get("PS_CHECKER", "checker")
```

```diff
+def deterministic_values(size, seed):
+    values = list(range(size))
+    state = seed & 0xFFFFFFFF
+    for index in range(size - 1, 0, -1):
+        state = (1664525 * state + 1013904223) & 0xFFFFFFFF
+        selected = state % (index + 1)
+        values[index], values[selected] = values[selected], values[index]
+    offset = size * 23
+    return [value * 37 - offset for value in values]
```

```diff
+def test_seeded_differential_properties():
+    for seed in (1, 7, 97, 4242, 9001):
+        for size in (2, 3, 5, 6, 17, 64):
+            values = deterministic_values(size, seed)
+            first = assert_sorted_by_program(values)
+            second = assert_sorted_by_program(values)
+            assert_equal(second, first,
+                f"deterministic moves for seed {seed}, size {size}")
```

### 무엇을 준비하는가, 아직 없는 것

32-bit LCG와 Fisher–Yates 순서를 test source에 직접 넣어 fixture 생성 규칙을 명시한다. permutation 값은 `value * 37 - size * 23`으로 유일한 signed integer가 되고, 동일 input을 두 번 실행해 command list까지 같아야 한다.

환경 변수로 executable path를 바꿀 수 있게 한 것은 같은 functional suite를 fault/sanitize binary에 재사용할 기반이 된다. 이 테스트는 deterministic output을 요구하지만, 다른 올바른 command sequence를 허용하도록 sorter를 바꾸는 경우에도 regression이 발생한다. 즉 재현성 계약을 의도적으로 고정한 것이다.

### 관련 커밋

`5505adf3e469`은 `PS_PUSH_SWAP`·`PS_CHECKER` override를 사용해 sanitizer binary에서 같은 suite를 실행한다. `6569949742eb`은 동일한 deterministic fixture 규칙을 resource baseline에도 재사용한다.

## 6569949742eb — test(resource): 명령과 배열 이동 및 할당량을 기준화
**중요도** `A` · **태그** `TEST, RESOURCE, PERF`

### 왜 다른 기법이 필요한가

command 하나의 외부 비용과 array-backed implementation의 내부 비용은 다르다. `ra` 한 줄은 protocol상 operation 1개지만, size가 큰 배열에서는 많은 `(value, rank)` pair를 `memmove`한다. command ceiling만 확인하면 representation change가 만드는 물리 비용 증가와 project allocation peak를 볼 수 없다.

### 무엇이 바뀌었는가 (diff)

```diff
 static int	emit_op(const char *name, int emit)
 {
 	if (emit)
-		return (ps_putstr_fd(1, name));
+	{
+		if (!ps_putstr_fd(1, name))
+			return (0);
+		ps_record_operation();
+	}
 	return (1);
 }
```

```diff
 	stack->values[1] = value;
 	stack->ranks[1] = rank;
+	ps_record_movements(2);
 }
@@
+	ps_record_movements((size_t)(dst->size + src->size));
@@
+	ps_record_movements((size_t)stack->size);
```

```diff
+# ifdef PS_FAULT_INJECTION
+void	ps_record_operation(void);
+void	ps_record_movements(size_t count);
+# else
+#  define ps_record_operation() ((void)0)
+#  define ps_record_movements(count) ((void)0)
+# endif
```

normal build에서는 두 계측 호출이 macro no-op이다. fault build는 successfully emitted command만 operation count에 더하고, primitive가 이동하거나 다시 쓴 pair 수를 movement count에 더한다. allocation header의 requested size를 이용해 current/peak bytes도 계산한다.

baseline은 size 10·100·500과 seed 7·4242·9001에 대해 exact command count, movement ceiling, peak-byte ceiling을 기록한다. 예를 들어 size 500은 command 6784개를 exact로 요구하지만 movement는 3,200,000 이하의 ceiling으로 관리한다. stderr metric의 operation count가 실제 stdout line 수와 같은지도 확인한다.

### 증명하는 것 / 증명하지 않는 것

고정 fixture에서 command stream 변화, pair movement 급증, project-requested peak bytes 증가, 종료 시 live allocation 잔존을 서로 다른 값으로 관찰한다. elapsed time은 출력만 하고 pass/fail에 쓰지 않는다.

movement는 source가 정의한 추상 pair count이며 CPU instruction·cache miss·kernel cost가 아니다. peak bytes도 project wrapper를 통과한 allocation 요청만 포함하고 계측 header와 외부 runtime allocation은 제외한다.

### 관련 커밋

`23198a9cdd55`의 fixture specification을 그대로 사용해 functional test와 resource test가 같은 입력을 가리키게 한다. `5faa9d7697af`과 `63969f770a21`의 runtime seam·allocation header가 metric 수집 기반이 된다.

## 5505adf3e469 — build(sanitize): C99 sanitizer 검증 경로를 추가
**중요도** `B` · **태그** `TEST, RUNTIME, PRACTICAL`

이 커밋은 production semantics를 바꾸지 않고 별도 instrumented object tree를 만드는 enabling build다.

### 왜 필요한가

assertion 기반 테스트는 기대값 불일치를 찾지만 out-of-bounds, use-after-free, signed undefined behavior를 직접 관찰하지 못할 수 있다. 반대로 sanitizer는 알고리즘의 최종 정렬이 맞는지 스스로 판단하지 않는다. 기존 operation/functional suite를 instrumented binary에서 다시 실행해야 두 관찰을 결합할 수 있다.

### 무엇이 바뀌었는가 (diff)

```diff
+SANITIZE_DIR := $(OBJ_DIR)/sanitize
+SANITIZE_CFLAGS := $(CFLAGS) -O1 -g -fno-omit-frame-pointer \
+	-fsanitize=address,undefined
+SANITIZE_COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_PUSH_SWAP := $(SANITIZE_DIR)/push_swap
+SANITIZE_CHECKER := $(SANITIZE_DIR)/checker
+SANITIZE_OPERATION_TEST := $(SANITIZE_DIR)/operation_invariants
```

```diff
+sanitize: $(SANITIZE_PUSH_SWAP) $(SANITIZE_CHECKER) \
+		$(SANITIZE_OPERATION_TEST)
+	ASAN_OPTIONS=halt_on_error=1 \
+		UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		$(SANITIZE_OPERATION_TEST)
+	ASAN_OPTIONS=halt_on_error=1 \
+		UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		PS_PUSH_SWAP=$(SANITIZE_PUSH_SWAP) PS_CHECKER=$(SANITIZE_CHECKER) \
+		python3 tests/run_tests.py
```

### production 동작이 바뀌지 않는 이유

sanitize object와 binary는 `.build/sanitize`에 분리되고 일반 `all` target의 flags를 바꾸지 않는다. target은 operation invariant test와 functional Python suite를 실행하지만 fault/resource suite까지 sanitizer 아래서 실행하지는 않는다.

### 관련 커밋

`5b7559278909`의 독립 replay와 `23198a9cdd55`의 executable override를 재사용한다. `6569949742eb`의 resource metric은 별도 fault build에서 관찰되므로 sanitizer target과 증명 범위가 겹치지 않는다.

## 이 Thread의 경계

이 문서는 production sorting code를 새로 구현하지 않고 외부에서 생성 결과와 비용을 관찰한다. tiny/radix strategy 자체는 `03-building-the-sorting-engine`, command primitive의 정확한 state transition은 `01-parallel-stack-state-and-operation-invariants`에 속한다.

allocation·read·write failure를 만들고 cleanup/status propagation을 검증하는 fault architecture는 `06-runtime-fault-injection-and-output-failure-propagation`이 담당한다. checker framing의 적법성은 `05-checker-protocol-and-verdict-hardening`의 문제다.

> 검토 범위: `5b7559278909`, `a16dde75d935`, `23198a9cdd55`, `6569949742eb`, `5505adf3e469`의 diff와 해당 시점 test, runtime instrumentation, Makefile을 확인했다. 로컬 checkout, functional/resource/sanitizer 실행은 수행하지 않았다.
