# Thread: checker 명령 프레임과 판정 결과의 경계
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 05

## 개요

checker는 초기 정수 입력을 A에 만들고, stdin의 command frame을 순서대로 적용한 뒤 최종 상태를 판정한다. 유효한 stream이 A를 정렬하고 B를 비우면 `OK`, 유효하지만 목표 상태가 아니면 `KO`, 숫자 입력·command·transport가 잘못되면 `Error`와 실패 status가 된다.

초기 구현은 이 lifecycle을 만들었지만 reader를 임의 길이 line utility로 설계했다. 실제 command는 최대 3 byte이므로 후속 fix는 protocol limit를 allocation과 read의 직접 경계로 옮긴다. 그 결과 overlength와 embedded NUL을 dispatch 전에 거절하고, `EINTR`과 영구 read error를 구분할 수 있게 된다.

### 최종 판정

| stdin 처리 결과 | 최종 상태 | stdout | process status |
| --- | --- | --- | ---: |
| clean EOF까지 모든 command가 유효 | A sorted, B empty | `OK\n` | 0 |
| clean EOF까지 모든 command가 유효 | 그 외 | `KO\n` | 0 |
| malformed frame·unknown command·read/allocation failure | 판정하지 않음 | 없음 | 1, stderr `Error\n` |
| 값 인자가 없음 | stdin을 읽지 않음 | 없음 | 0 |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `0b87adebca2b` | feat(checker): 표준 입력 명령 프레임을 읽음 | B | `CHECKER, CORE` | newline 또는 EOF까지 동적 문자열을 읽는 tri-state reader 도입 |
| 2 | `f79ae7e86592` | feat(checker): 스택 연산 명령을 해석 | B | `CHECKER, INTEGRATION` | 11개 명령을 공통 operation wrapper의 silent 실행으로 연결 |
| 3 | `d906f4d86528` | feat(checker): 명령 실행 결과를 판정 | A | `CHECKER, CORE, INTEGRATION` | parse → read/apply → complete-sort 판정의 checker executable 완성 |
| 4 | `44ee0830e9f0` | test(checker): 명령 연산과 최종 판정을 검증 | B | `TEST, CHECKER` | 각 operation family, `OK`, `KO`, unknown command를 CLI로 검증 |
| 5 | `7713a31cf502` | fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도 | A | `CHECKER, RUNTIME, RISK` | protocol 최대 길이·NUL 거절·`EINTR` retry를 reader 자체에 적용 |
| 6 | `dbf76e147e68` | test(checker): 읽기 실패와 명령 경계를 검증 | A | `TEST, CHECKER, RISK` | read call별 EIO/EINTR와 malformed frame 경계를 deterministic하게 검증 |

## 0b87adebca2b — feat(checker): 표준 입력 명령 프레임을 읽음
**중요도** `B` · **태그** `CHECKER, CORE`

### 무엇이 바뀌었는가 (diff)

```diff
+static int	grow_line(char **line, size_t *capacity, size_t needed)
+{
+	char	*next;
+	size_t	new_capacity;
+	size_t	i;
+
+	if (*capacity > needed)
+		return (1);
+	new_capacity = 32;
+	if (*capacity != 0)
+		new_capacity = *capacity;
+	while (new_capacity <= needed)
+		new_capacity *= 2;
+	next = (char *)malloc(new_capacity);
+	if (next == NULL)
+		return (0);
+	/* ... */
+	return (1);
+}
+
+int	read_next_line(int fd, char **line)
+{
+	char	c;
+	ssize_t	bytes;
+	size_t	len;
+	size_t	capacity;
+
+	*line = NULL;
+	len = 0;
+	capacity = 0;
+	while (1)
+	{
+		bytes = read(fd, &c, 1);
+		if (bytes < 0)
+			return (free(*line), -1);
+		if (bytes == 0)
+			break ;
+		if (c == '\n')
+			break ;
+		if (!grow_line(line, &capacity, len + 1))
+			return (free(*line), -1);
+		(*line)[len++] = c;
+	}
+	if (bytes == 0 && len == 0)
+		return (free(*line), 0);
+	if (!grow_line(line, &capacity, len + 1))
+		return (free(*line), -1);
+	(*line)[len] = '\0';
+	return (1);
+}
```

### 무엇을 준비하는가, 아직 없는 것

반환값 1이면 caller가 NUL-terminated heap frame을 소유하고, 0이면 frame 시작 전 clean EOF, -1이면 read/allocation failure다. newline은 결과에 포함하지 않으며, `sa` 뒤 즉시 EOF가 와도 non-empty final frame으로 허용한다.

이 reader는 일반 line input으로는 동작하지만 checker protocol에는 과도하다. 3-byte command를 위해 arbitrary growth를 허용하고, `EINTR`을 영구 오류처럼 처리하며, embedded NUL도 저장한다. 이후 exact string 비교는 첫 NUL에서 멈추므로 `sa\0pb`가 `sa`처럼 보일 위험이 있다.

### 관련 커밋

`f79ae7e86592`가 이 frame을 11개 command에 dispatch한다. `7713a31cf502`는 general line abstraction을 유지하지 않고 protocol-sized buffer로 교체해 이 커밋의 세 한계를 root에서 제거한다.

## f79ae7e86592 — feat(checker): 스택 연산 명령을 해석
**중요도** `B` · **태그** `CHECKER, INTEGRATION`

### 무엇이 바뀌었는가 (diff)

```diff
+int	apply_checker_command(const char *line, t_stack *a, t_stack *b)
+{
+	if (ps_strcmp(line, "sa") == 0)
+		op_sa(a, 0);
+	else if (ps_strcmp(line, "sb") == 0)
+		op_sb(b, 0);
+	else if (ps_strcmp(line, "ss") == 0)
+		op_ss(a, b, 0);
+	else if (ps_strcmp(line, "pa") == 0)
+		op_pa(a, b, 0);
+	/* ... */
+	else if (ps_strcmp(line, "rrr") == 0)
+		op_rrr(a, b, 0);
+	else
+		return (0);
+	return (1);
+}
```

### 무엇을 준비하는가, 아직 없는 것

exact `ps_strcmp`이므로 prefix나 suffix가 붙은 문자열은 유효하지 않다. `emit = 0`으로 공통 operation wrapper를 호출해 checker가 command를 stdout에 다시 쓰지 않는다.

원소 부족으로 primitive가 no-op이 되어도 command 문자열 자체는 합법이므로 1을 반환한다. protocol membership과 실제 state change 유무를 분리한 것이다. 이 SHA에는 stdin loop와 final verdict가 아직 없다.

### 관련 커밋

`d906f4d86528`은 reader와 dispatcher를 하나의 ownership loop로 연결한다. operation의 exact state semantics는 `01-parallel-stack-state-and-operation-invariants`에서 별도로 검증된다.

## d906f4d86528 — feat(checker): 명령 실행 결과를 판정
**중요도** `A` · **태그** `CHECKER, CORE, INTEGRATION`

### 무엇을 만들었는가 (diff)

```diff
+static int	read_and_apply(t_stack *a, t_stack *b)
+{
+	char	*line;
+	int		status;
+
+	status = read_next_line(0, &line);
+	while (status > 0)
+	{
+		if (!apply_checker_command(line, a, b))
+		{
+			free(line);
+			return (0);
+		}
+		free(line);
+		status = read_next_line(0, &line);
+	}
+	return (status == 0);
+}
```

```diff
+int	main(int argc, char **argv)
+{
+	t_stack	a;
+	t_stack	b;
+
+	if (argc == 1)
+		return (0);
+	if (!parse_input(argc, argv, &a))
+		return (write_error(), 1);
+	if (!stack_init(&b, a.capacity))
+	{
+		stack_free(&a);
+		return (write_error(), 1);
+	}
+	if (!read_and_apply(&a, &b))
+	{
+		stack_free(&a);
+		stack_free(&b);
+		return (write_error(), 1);
+	}
+	if (stack_is_complete_sorted(&a, &b))
+		ps_putstr_fd(1, "OK\n");
+	else
+		ps_putstr_fd(1, "KO\n");
+	stack_free(&a);
+	stack_free(&b);
+	return (0);
+}
```

### 설계 결정 — clean EOF와 정상적인 `KO`를 오류에서 분리한다

reader가 frame을 반환하면 loop가 정확히 한 번 소유하고 apply 뒤 해제한다. unknown command에서는 현재 frame을 해제하고 실패하며, read failure의 partial buffer는 reader가 정리한다. clean EOF에 도달한 경우에만 complete-state predicate를 평가한다.

`KO`는 malformed input이 아니라 유효한 command sequence가 목표 상태를 만들지 못했다는 정상 verdict다. 따라서 `OK`와 `KO`는 모두 status 0이고, parse·allocation·read·unknown-command 실패만 status 1과 `Error\n`을 사용한다. 값이 없는 실행은 stdin loop 전에 반환해 stream을 소비하지 않는다.

이 SHA의 verdict와 diagnostic output은 여전히 write 결과를 확인하지 않는다.

### 관련 커밋

`44ee0830e9f0`은 11개 command family와 `OK`/`KO`/`Error`의 public 차이를 CLI로 고정한다. `315f4b91779b`은 후속으로 verdict write 실패도 process status에 반영한다.

## 44ee0830e9f0 — test(checker): 명령 연산과 최종 판정을 검증
**중요도** `B` · **태그** `TEST, CHECKER`

### 무엇을 검증하는가

```diff
+def test_checker_operations():
+    cases = [
+        ("sa", ["2", "1"], "sa\n"),
+        ("sb", ["2", "1", "3"], "pb\npb\nsb\npa\npa\n"),
+        ("ss", ["2", "1", "4", "3"], "pb\npb\nss\npa\npa\n"),
+        /* ... */
+    ]
+    for label, args, program in cases:
+        checker_ok(args, program, label)
+
+    ko = run([CHECKER, "2", "1"], "")
+    assert_equal(ko.returncode, 0, "unsorted checker input exits cleanly")
+    assert_equal(ko.stdout, "KO\n", "unsorted checker input reports KO")
+
+    invalid = run([CHECKER, "1", "2"], "ra\nwat\n")
+    assert_ok(invalid.returncode != 0, "invalid checker command fails")
+    assert_equal(invalid.stderr, "Error\n", "invalid command reports Error")
```

각 single/combined operation이 checker executable의 stdin dispatch를 실제로 통과해 known sorted state를 만드는지 확인한다. no-command unsorted input은 `KO`와 status 0, valid prefix 뒤 unknown command는 verdict 없이 `Error`와 non-zero status여야 한다.

### 증명하는 것 / 증명하지 않는 것

command mapping과 세 public outcome을 고정하지만 reader의 overlength, embedded NUL, read fault, unterminated final frame은 직접 겨냥하지 않는다. operation 내부의 모든 배열 상태를 exhaustive하게 검사하는 테스트도 아니다.

### 관련 커밋

`7713a31cf502`가 framing boundary를 변경하더라도 이 기존 public command behavior는 유지돼야 한다. `dbf76e147e68`은 이 테스트가 다루지 않은 transport와 byte-level malformed frame을 추가한다.

## 7713a31cf502 — fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도
**중요도** `A` · **태그** `CHECKER, RUNTIME, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

가장 긴 command가 3 byte인데도 dynamic reader는 arbitrary input을 계속 할당했다. embedded NUL은 C string 비교를 조기 종료시킬 수 있고, `read`의 `EINTR`은 재시도 가능한 interruption인데 영구 오류와 같은 결과가 됐다.

원인은 checker가 요구하는 fixed protocol을 일반 line abstraction 뒤에 숨긴 데 있다. 결정은 `PS_COMMAND_MAX = 3`을 reader의 allocation·length·validation 경계로 직접 사용하는 것이다.

### 무엇이 바뀌었는가 (diff)

```diff
+# define PS_COMMAND_MAX 3
```

```diff
-static int	grow_line(char **line, size_t *capacity, size_t needed)
-{
-	/* ... */
-}
-
+#include <errno.h>
+
 int	read_next_line(int fd, char **line)
 {
 	char	c;
 	ssize_t	bytes;
 	size_t	len;
-	size_t	capacity;
 
-	*line = NULL;
+	*line = (char *)ps_malloc(PS_COMMAND_MAX + 1);
+	if (*line == NULL)
+		return (-1);
 	len = 0;
-	capacity = 0;
 	while (1)
 	{
 		bytes = ps_read(fd, &c, 1);
+		if (bytes < 0 && errno == EINTR)
+			continue ;
 		if (bytes < 0)
-			return (ps_free(*line), -1);
+			return (ps_free(*line), *line = NULL, -1);
 		if (bytes == 0)
 			break ;
 		if (c == '\n')
 			break ;
-		if (!grow_line(line, &capacity, len + 1))
-			return (ps_free(*line), -1);
+		if (c == '\0' || len >= PS_COMMAND_MAX)
+			return (ps_free(*line), *line = NULL, -1);
 		(*line)[len++] = c;
 	}
 	if (bytes == 0 && len == 0)
-		return (ps_free(*line), 0);
-	if (!grow_line(line, &capacity, len + 1))
-		return (ps_free(*line), -1);
+		return (ps_free(*line), *line = NULL, 0);
 	(*line)[len] = '\0';
 	return (1);
 }
```

### 이 커밋이 보장하는 것 / 보장하지 않는 것

reader는 최대 3개의 non-newline byte만 저장하고, 네 번째 byte나 NUL을 읽는 즉시 frame 전체를 오류로 만든다. `EINTR`은 같은 위치에서 다시 읽고, 그 외 negative read는 failure다. clean EOF before any byte는 0, 최대 3-byte의 non-empty final frame은 newline이 없어도 1이다.

고정 buffer는 매 frame마다 allocation된다. allocation failure 자체와 caller cleanup은 여전히 runtime wrapper와 checker loop에 의존한다. 문자열이 3 byte 이하라고 해서 command가 합법이라는 뜻도 아니며 exact dispatcher가 최종 membership을 판단한다.

### 관련 커밋

`dbf76e147e68`은 `EINTR`과 EIO를 call 위치별로 주입하고, NUL·empty·overlength·unterminated frame을 직접 검증한다. `5faa9d7697af`의 `ps_read`/`ps_malloc` boundary가 이 fix와 test를 가능하게 한다.

## dbf76e147e68 — test(checker): 읽기 실패와 명령 경계를 검증
**중요도** `A` · **태그** `TEST, CHECKER, RISK`

### 무엇을 검증하는가

```diff
 ssize_t	ps_read(int fd, void *buffer, size_t count)
 {
+#ifdef PS_FAULT_INJECTION
+	g_read_calls++;
+	if (at_index("PS_EINTR_READ_AT", g_read_calls))
+		return (errno = EINTR, -1);
+	if (at_index("PS_FAIL_READ_AT", g_read_calls))
+		return (errno = EIO, -1);
+#endif
 	return (read(fd, buffer, count));
 }
```

```diff
+def test_read_failures_and_command_bounds():
+    for index in range(1, 5):
+        result = run(CHECKER, ["2", "1"], b"sa\n",
+            faults={"PS_FAIL_READ_AT": index})
+        assert_true(result.returncode != 0, f"checker read #{index} fails")
+
+    for index in (1, 4):
+        result = run(CHECKER, ["2", "1"], b"sa\n",
+            faults={"PS_EINTR_READ_AT": index})
+        assert_equal(result.stdout, b"OK\n", "checker result after EINTR")
+
+    invalid_streams = [
+        b"sa\x00pb\n", b"rrrr\n", b"rrrr", b"\x00\n",
+        b"\n", b"r" * 65536 + b"\n",
+    ]
+
+    unterminated = run(CHECKER, ["2", "1"], b"sa")
+    assert_equal(unterminated.stdout, b"OK\n",
+        "unterminated command is applied")
```

EIO는 `sa\n`을 읽는 1~4번째 call 어디에서 발생해도 no verdict·`Error`·non-zero status가 되어야 한다. EINTR는 첫 byte와 EOF 확인 위치에서 주입해 retry 뒤 `OK`를 요구한다.

byte boundary cases는 embedded NUL, 4-byte frame, newline 없는 4-byte frame, NUL-only, empty line, 65536-byte overlength를 거절한다. 반대로 newline 없는 합법 `sa`는 EOF-delimited final frame으로 적용한다.

### 증명하는 것 / 증명하지 않는 것

고정 fixture에서 transport error와 protocol invalidity를 분리하고, overlength input이 큰 allocation으로 이어지지 않는 경로를 검증한다. 모든 errno, arbitrary binary stream, write-side verdict failure를 포괄하지는 않는다.

### 관련 커밋

`7713a31cf502`의 fixed-frame decision을 직접 고정한다. verdict와 `Error` write가 실패하는 경우는 `315f4b91779b`과 `e1154e181864`가 별도 runtime contract로 다룬다.

## 이 Thread의 경계

이 문서는 stdin command framing, silent dispatch, `OK`/`KO`/`Error` 의미만 다룬다. command가 `(value, rank)` pair를 어떻게 이동하는지는 `01-parallel-stack-state-and-operation-invariants`, generator가 올바른 stream을 만드는지는 `03-building-the-sorting-engine`과 `04-independent-correctness-and-cost-evidence`에 속한다.

숫자 argv의 문법과 coordinate compression은 `02-input-grammar-coordinate-compression-and-size-safety`의 문제다. allocation/read seam의 전체 구조와 output failure propagation은 `06-runtime-fault-injection-and-output-failure-propagation`이 담당한다.

> 검토 범위: `0b87adebca2b`, `f79ae7e86592`, `d906f4d86528`, `44ee0830e9f0`, `7713a31cf502`, `dbf76e147e68`의 diff와 해당 시점 checker/runtime/test source를 확인했다. 로컬 checkout, build, functional/fault test 실행은 수행하지 않았다.
