# Thread: 시스템 호출 실패를 실행 결과까지 전달하는 책임
> Project: `push_swap` · Branch: `c/push_swap` · 문서 번호: 06

## 개요

정상 경로에서는 `malloc`, `read`, `write`가 단순해 보이지만 각각 부분 상태를 남길 수 있다. 첫 allocation만 성공할 수 있고, `read`는 `EINTR`로 중단될 수 있으며, `write`는 일부 byte만 기록하거나 0·`EPIPE`를 반환할 수 있다. 초기 helper는 이런 결과를 숨겨 caller가 정상 종료와 전달 실패를 구분하지 못했다.

이 Thread는 system call을 공통 runtime boundary로 모아 N번째 실패를 반복 재현하고, allocation cleanup을 먼저 검증한 뒤, output을 반환 가능한 작업으로 바꾸는 과정이다. 최종 계약은 in-memory state가 정렬됐다는 사실만으로 성공하지 않는다. 생성된 command나 verdict 전체를 전달하지 못하면 남은 출력을 중단하고 소유한 자원을 정리한 뒤 process가 실패해야 한다.

### 최종 실패 처리

| low-level 결과 | runtime 동작 | executable 결과 |
| --- | --- | --- |
| allocation 실패 | partial owner가 matching `ps_free`로 정리 | `Error\n`, status 1 |
| read `EINTR` | 같은 byte 위치 재시도 | 성공 경로 계속 |
| permanent read error | checker frame 처리 중단 | `Error\n`, status 1 |
| short positive write | cursor와 남은 길이를 전진해 계속 | 전체 전달 시 성공 |
| zero-byte write·permanent write error·closed pipe | 더 이상 progress하지 않고 실패 | 추가 command 중단, cleanup, status 1 |
| diagnostic write 자체 실패 | 기존 실패 status 유지 | stderr가 비어 있을 수 있음 |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `2e97f29961d8` | feat(io): 문자열 비교와 기본 출력을 구현 | B | `RUNTIME, PRACTICAL` | 공통 문자열 비교와 단일 `write` 기반 출력 helper 도입 |
| 2 | `5faa9d7697af` | refactor(runtime): 메모리와 입력 시스템 호출을 공통화 | A | `ARCH, REFACTOR, RUNTIME` | 프로젝트 allocation/free/read를 transparent wrapper 뒤로 이동 |
| 3 | `63969f770a21` | test(memory): 할당 실패 뒤 자원 정리를 검증 | A | `TEST, RUNTIME, RISK` | N번째 allocation failure와 live-allocation 보고를 fault build에 추가 |
| 4 | `315f4b91779b` | fix(io): 출력 실패를 호출 경로 끝까지 전파 | A | `RUNTIME, RISK, INTEGRATION` | short write·EINTR·EPIPE를 처리하고 operation 실패를 main까지 전파 |
| 5 | `e1154e181864` | test(io): 부분 출력과 영구 쓰기 실패를 검증 | A | `TEST, RUNTIME, RISK` | write call별 실패, short/zero/EINTR, 닫힌 pipe, diagnostic failure를 검증 |

## 2e97f29961d8 — feat(io): 문자열 비교와 기본 출력을 구현
**중요도** `B` · **태그** `RUNTIME, PRACTICAL`

### 무엇이 바뀌었는가 (diff)

```diff
+size_t	ps_strlen(const char *str)
+{
+	size_t	len;
+
+	len = 0;
+	while (str[len] != '\0')
+		len++;
+	return (len);
+}
+
+int	ps_strcmp(const char *a, const char *b)
+{
+	size_t	i;
+
+	i = 0;
+	while (a[i] != '\0' && b[i] != '\0' && a[i] == b[i])
+		i++;
+	return ((unsigned char)a[i] - (unsigned char)b[i]);
+}
+
+void	ps_putstr_fd(int fd, const char *str)
+{
+	if (str != NULL)
+		(void)write(fd, str, ps_strlen(str));
+}
+
+void	write_error(void)
+{
+	ps_putstr_fd(2, "Error\n");
+}
```

### 무엇을 준비하는가, 아직 없는 것

`ps_strcmp`는 checker command의 exact membership에 사용되고, `write_error`는 public diagnostic을 한곳에 모은다. 그러나 `ps_putstr_fd`는 single `write`의 반환값을 버리고 return type도 `void`다. short write, `EINTR`, zero, `EPIPE`를 caller가 구분할 방법이 없다.

operation wrapper는 state를 먼저 바꾸고 이 helper를 호출한다. 따라서 stdout 전달이 실패해도 sorter는 그 사실을 모른 채 다음 command를 계속 생성한다. 이 커밋은 output 기능만 제공하며 failure contract는 아직 만들지 않는다.

### 관련 커밋

`c0de1a1b18bb`부터 operation wrapper가 이 helper를 사용한다. `315f4b91779b`은 single unchecked side effect를 write-all + return status로 바꾸고 모든 caller signature를 연결한다.

## 5faa9d7697af — refactor(runtime): 메모리와 입력 시스템 호출을 공통화
**중요도** `A` · **태그** `ARCH, REFACTOR, RUNTIME`

이 커밋은 앞뒤의 feature와 fix처럼 동작을 바꾸지 않고, 실패를 선택적으로 만들 수 있는 seam을 추가하는 enabling refactor다.

### 왜 필요한가

stack의 두 buffer, parser scratch, checker frame이 직접 `malloc/free`를 사용하면 특정 획득 지점만 실패시키거나 live allocation을 일관되게 셀 수 없다. checker의 direct `read`도 call 위치별 transport failure를 주입하기 어렵다.

### 무엇이 바뀌었는가 (diff)

```diff
+void	*ps_malloc(size_t size)
+{
+	return (malloc(size));
+}
+
+void	ps_free(void *pointer)
+{
+	free(pointer);
+}
+
+ssize_t	ps_read(int fd, void *buffer, size_t count)
+{
+	return (read(fd, buffer, count));
+}
```

```diff
-	stack->values = (int *)malloc(sizeof(int) * (size_t)capacity);
-	stack->ranks = (int *)malloc(sizeof(int) * (size_t)capacity);
+	stack->values = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
+	stack->ranks = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
@@
-	free(stack->values);
-	free(stack->ranks);
+	ps_free(stack->values);
+	ps_free(stack->ranks);
```

parser scratch와 checker dynamic frame도 같은 matching pair로 이동하고, checker reader는 `ps_read`를 사용한다.

### production 동작이 바뀌지 않는 이유

normal build의 wrapper는 arguments와 반환값을 raw libc/POSIX call에 그대로 위임한다. ownership도 바뀌지 않는다. stack이 얻은 두 buffer는 `stack_free`, parser scratch는 `assign_ranks`, checker frame은 read/apply loop가 계속 정리한다. seam은 owner가 아니라 관찰 지점이다.

### 관련 커밋

`63969f770a21`은 이 matching boundary 위에 N번째 allocation failure와 live counter를 구현한다. `dbf76e147e68`은 `ps_read`에 EIO/EINTR 주입을 추가한다. output은 아직 이 boundary 밖이며 `315f4b91779b`에서 합류한다.

## 63969f770a21 — test(memory): 할당 실패 뒤 자원 정리를 검증
**중요도** `A` · **태그** `TEST, RUNTIME, RISK`

### 왜 다른 기법이 필요한가

일반 success/error fixture는 allocation이 어느 지점에서 실패했는지 제어하지 못한다. 특히 A의 첫 buffer는 성공하고 두 번째가 실패하거나, parser scratch·B·checker frame이 실패하는 경로는 반복 재현이 어렵다. 이 커밋은 fault-only binary에서 획득 순서를 직접 선택한다.

### 무엇이 바뀌었는가 (diff)

```diff
+#ifdef PS_FAULT_INJECTION
+typedef union u_allocation_header
+{
+	struct
+	{
+		size_t			size;
+		unsigned long	magic;
+	}	data;
+	long double	align_long_double;
+	void		*align_pointer;
+}	t_allocation_header;
+
+static unsigned long	g_malloc_calls;
+static unsigned long	g_live_allocations;
+#endif
```

```diff
 void	*ps_malloc(size_t size)
 {
+#ifdef PS_FAULT_INJECTION
+	t_allocation_header	*header;
+
+	g_malloc_calls++;
+	if (at_index("PS_FAIL_MALLOC_AT", g_malloc_calls))
+		return (NULL);
+	if (size > (size_t)-1 - sizeof(*header))
+		return (NULL);
+	header = (t_allocation_header *)malloc(sizeof(*header) + size);
+	if (header == NULL)
+		return (NULL);
+	header->data.size = size;
+	header->data.magic = 0x50535354UL;
+	g_live_allocations++;
+	return ((void *)(header + 1));
+#else
 	return (malloc(size));
+#endif
 }
```

union header는 payload pointer의 alignment를 유지하면서 requested size와 magic을 저장한다. matching `ps_free`는 header를 찾아 live count를 감소시킨다. 모든 executable return은 `ps_test_finish(status)`를 거치며, 요청된 report에서 live count가 0이 아니면 별도 status 99를 반환한다.

```diff
+def test_nth_allocation_failures():
+    for index in range(1, 6):
+        result = run(PUSH_SWAP, ["4", "3", "2", "1"],
+            faults={"PS_FAIL_MALLOC_AT": index})
+        assert_true(result.returncode != 0, f"push_swap malloc #{index} fails")
+        assert_equal(result.stderr, b"Error\n", "allocation error")
+
+    for index in range(1, 7):
+        result = run(CHECKER, ["2", "1"], b"sa\n",
+            faults={"PS_FAIL_MALLOC_AT": index})
+        assert_true(result.returncode != 0, f"checker malloc #{index} fails")
```

각 selected failure와 one-past-last success에서 `PS_LIVE_ALLOCATIONS=0`을 요구한다.

### 증명하는 것 / 증명하지 않는 것

선택한 push_swap/checker 경로의 모든 project allocation 획득 지점에서 failure status와 zero live allocations를 확인한다. libc 내부 allocation, invalid/double free의 일반 탐지, read/write fault, 모든 argv/command 조합까지 증명하지는 않는다.

### 관련 커밋

`5faa9d7697af`의 matching wrappers가 없으면 이 sweep은 project allocation 전체를 관찰할 수 없다. `e1154e181864`은 같은 live-report 경계를 write failure와 실제 closed pipe cleanup에도 적용한다.

## 315f4b91779b — fix(io): 출력 실패를 호출 경로 끝까지 전파
**중요도** `A` · **태그** `RUNTIME, RISK, INTEGRATION`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

기존 output helper는 write 결과를 버렸고 operation·sorter·main도 모두 `void` 또는 무조건 성공을 반환했다. command program의 일부만 stdout에 보였는데도 process가 status 0으로 끝날 수 있었다. 닫힌 pipe에서는 default `SIGPIPE`가 cleanup code보다 먼저 process를 종료할 수도 있었다.

결정은 세 단계다. write를 progress loop로 만들고, output을 수행하는 모든 API가 성공 여부를 반환하게 하며, executable이 `SIGPIPE`를 무시해 `EPIPE`를 일반 오류 경로로 처리한다.

### 최종 write loop (학습용 주석)

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
            continue;               // byte progress가 없으므로 같은 범위를 재시도한다.
        if (written <= 0)
            return (0);             // 0도 무한 loop를 막기 위해 영구 실패로 본다.
        cursor += (size_t)written;   // 이미 공개된 prefix 다음으로 이동한다.
        count -= (size_t)written;
    }
    return (1);
}
```

### 기존 호출부가 어떻게 바뀌었는가 (diff)

```diff
-void	op_sa(t_stack *a, int emit);
+int		op_sa(t_stack *a, int emit);
 /* ... */
-void	sort_stack(t_stack *a, t_stack *b);
+int		sort_stack(t_stack *a, t_stack *b);
-void	ps_putstr_fd(int fd, const char *str);
-void	write_error(void);
+int		ps_putstr_fd(int fd, const char *str);
+int		write_error(void);
```

```diff
-static void	emit_op(const char *name, int emit)
+static int	emit_op(const char *name, int emit)
 {
 	if (emit)
-		ps_putstr_fd(1, name);
+		return (ps_putstr_fd(1, name));
+	return (1);
 }
@@
-void	op_sa(t_stack *a, int emit)
+int	op_sa(t_stack *a, int emit)
 {
 	stack_swap(a);
-	emit_op("sa\n", emit);
+	return (emit_op("sa\n", emit));
 }
```

sort helper는 각 `op_*` 반환값을 즉시 검사하고 첫 실패에서 0을 반환한다. `push_swap` main은 A/B를 해제한 뒤 status 1과 diagnostic을 시도한다. checker도 `OK`/`KO` write 결과를 status로 바꾼다. 두 executable은 시작 시 `signal(SIGPIPE, SIG_IGN)`을 설치해 closed stdout을 cleanup 가능한 `EPIPE`로 받는다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

성공 status는 필요한 output 전체가 전달됐다는 뜻이 된다. short positive write는 이미 쓴 prefix를 반복하지 않고 남은 suffix만 계속하며, `EINTR`은 재시도하고, zero/permanent error는 추가 command 생성을 중단한다.

operation은 state를 먼저 mutate한 뒤 명령을 쓴다. 따라서 write failure 시 내부 A/B 상태와 외부의 partial stream을 rollback하지 않는다. 이미 kernel에 전달된 prefix를 회수할 수 없으므로 이 commit은 transaction을 구현하는 대신 process를 실패시키고 publish를 즉시 중단한다. diagnostic write 자체가 실패해도 원래 failure status는 유지된다.

### 관련 커밋

`2e97f29961d8`에서 생긴 unchecked output contract를 root에서 바꾼 fix다. `e1154e181864`은 short·EINTR·zero·permanent failure와 실제 closed pipe를 서로 다른 fixture로 고정한다.

## e1154e181864 — test(io): 부분 출력과 영구 쓰기 실패를 검증
**중요도** `A` · **태그** `TEST, RUNTIME, RISK`

### 무엇을 검증하는가

```diff
 static ssize_t	ps_write_once(int fd, const void *buffer, size_t count)
 {
+#ifdef PS_FAULT_INJECTION
+	g_write_calls++;
+	if (at_index("PS_EINTR_WRITE_AT", g_write_calls))
+		return (errno = EINTR, -1);
+	if (at_index("PS_FAIL_WRITE_AT", g_write_calls))
+		return (errno = EPIPE, -1);
+	if (at_index("PS_ZERO_WRITE_AT", g_write_calls))
+		return (0);
+	if (at_index("PS_SHORT_WRITE_AT", g_write_calls) && count > 1)
+		count = 1;
+#endif
 	return (write(fd, buffer, count));
 }
```

```diff
+def test_write_failures():
+    baseline = run(PUSH_SWAP, ["3", "2", "1"])
+    write_count = len(baseline.stdout.splitlines())
+
+    for index in range(1, write_count + 1):
+        result = run(PUSH_SWAP, ["3", "2", "1"],
+            faults={"PS_FAIL_WRITE_AT": index})
+        assert_true(result.returncode != 0, f"write #{index} fails")
+
+    for fault in ("PS_EINTR_WRITE_AT", "PS_SHORT_WRITE_AT"):
+        result = run(PUSH_SWAP, ["3", "2", "1"], faults={fault: 1})
+        assert_equal(result.stdout, baseline.stdout,
+            f"{fault} preserves command stream")
+
+    partial = run(PUSH_SWAP, ["3", "2", "1"],
+        faults={"PS_SHORT_WRITE_AT": 1, "PS_FAIL_WRITE_AT": 2})
+    assert_equal(partial.stdout, baseline.stdout[:1],
+        "partial prefix is not repeated")
```

zero-byte write는 status non-zero와 `Error`, checker verdict의 첫 write failure도 status non-zero를 요구한다. parse/command error의 diagnostic write를 실패시키는 fixture는 stderr가 비어도 기존 status 1이 유지되는지 확인한다.

마지막 case는 read end를 닫은 실제 OS pipe에 push_swap stdout을 연결한다. default SIGPIPE 종료가 아니라 main이 status 1과 cleanup report에 도달하며 live allocation이 0이어야 한다.

### 증명하는 것 / 증명하지 않는 것

selected write call마다 permanent failure가 sorter/main까지 도달하고, recoverable short/EINTR가 baseline과 같은 stream을 만들며, partial prefix 뒤 실패가 prefix를 중복하지 않는지 확인한다. 실제 closed pipe에서 signal policy와 cleanup도 함께 관찰한다.

모든 filesystem·terminal·kernel write behavior, concurrent consumer, partial stream의 외부 복구 가능성을 증명하지 않는다. failure 이후 state rollback도 테스트 대상이 아니다.

### 관련 커밋

`315f4b91779b`의 write-all과 return propagation을 직접 고정한다. command와 movement를 세는 `6569949742eb`은 성공한 emission만 metric에 포함하도록 같은 `emit_op` 성공 지점을 사용한다.

## 이 Thread의 경계

이 문서는 system call failure seam, cleanup 관찰, output status propagation만 다룬다. stack primitive의 정상 state semantics는 `01-parallel-stack-state-and-operation-invariants`, checker frame grammar는 `05-checker-protocol-and-verdict-hardening`에 속한다.

sorting correctness와 command/resource baseline은 `04-independent-correctness-and-cost-evidence`가 담당한다. parser의 token-count와 allocation-size arithmetic guard는 `02-input-grammar-coordinate-compression-and-size-safety`의 별도 문제다.

> 검토 범위: `2e97f29961d8`, `5faa9d7697af`, `63969f770a21`, `315f4b91779b`, `e1154e181864`의 diff와 해당 시점 runtime, operation, sorter, executable, fault-test source를 확인했다. 로컬 checkout, fault build, test 실행은 수행하지 않았다.
