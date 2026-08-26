# Thread: transient `read` 결과를 byte-preserving parser state로 분리하기
> Project: `get_next_line` · Branch: `c/get_next_line` · 문서 번호: 04

## 개요

streaming reader에서 positive short read, EOF, `EINTR`, `EAGAIN`/`EWOULDBLOCK`, 다른 I/O error는 서로 다른 state transition입니다. 이 결과를 단순 성공/실패로 합치면 partial bytes를 조기 line으로 반환하거나, wait를 EOF로 오해하거나, 이미 받아들인 bytes를 error와 함께 버릴 수 있습니다.

이 Thread는 먼저 short read와 지정 read failure를 만들 수 있는 baseline harness를 세우고, `EINTR` retry와 `BLR_AGAIN`을 production policy로 도입합니다. 실제 nonblocking pipe는 OS behavior를 end-to-end로 확인하고, ordered errno script는 progress와 failure가 섞인 정확한 cursor 변화를 결정적으로 고정합니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | A | `TEST, POSIX_IO, RISK` | short read와 지정 호출 EIO를 재현하는 baseline fault harness를 제공합니다. |
| 2 | `f0055ae5cf19` | `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존` | S | `CORE, POSIX_IO, RISK` | `EINTR` retry, `BLR_AGAIN`, accepted-byte preservation policy를 확립합니다. |
| 3 | `f3504f674c73` | `test(reader): 비차단 부분 입력 보존 검증` | A | `TEST, POSIX_IO, EDGE` | 실제 nonblocking pipe에서 partial input·wait·resume·EOF를 검증합니다. |
| 4 | `11033bd85c59` | `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증` | A | `TEST, POSIX_IO, RISK` | ordered errno script로 progress와 failure가 섞인 state를 검증합니다. |

### 최종 state transition은 어떻게 구분되는가

| `read` 결과 | state 변화 | explicit result | compatibility API |
| --- | --- | --- | --- |
| `n > 0` | bytes를 `[end, end+n)`에 publish하고 `end` 전진 | line이 생길 때까지 계속 | 같은 engine 계속 |
| `0` | `reached_eof=1`; tail이 있으면 line, 없으면 terminal | `BLR_LINE` 또는 `BLR_EOF` | line 또는 `NULL` |
| `-1/EINTR` | cursor 변화 없이 같은 system call 재시도 | caller에게 보이지 않음 | caller에게 보이지 않음 |
| `-1/EAGAIN` 또는 `EWOULDBLOCK` | accepted bytes와 cursor 유지 | `BLR_AGAIN` | `NULL`, hidden context 유지 |
| 다른 `-1/error` | accepted bytes와 cursor 유지 | `BLR_ERROR` | `NULL`, hidden context 폐기 |

non-line result에서는 `*line == NULL`입니다.

## fd03a831686b — test(failure): 메모리 할당과 읽기 실패 처리 검증
**중요도** `A` · **태그** `TEST, POSIX_IO, RISK`

### 왜 baseline fault harness가 필요한가

normal file과 pipe만으로는 “한 번에 최대 3 byte”, “세 번째 positive-count read만 EIO” 같은 transition을 안정적으로 만들기 어렵습니다. 이 commit은 production object의 `read`, `malloc`, `free`를 replacement로 바꾼 별도 binary를 추가합니다.

```diff
+FAULT_DEFINES := -Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
+
+failure-test:
+	@set -e; for size in $(MATRIX_SIZES); do \
+		$(MAKE) --no-print-directory failure-run BUFFER_SIZE=$$size; \
+	done
```

`fault_read_limit(3)`은 target fd의 request count를 줄인 뒤 실제 `read`를 호출하므로 bytes는 실제 stream에서 옵니다. short positive result가 EOF가 아니라 progress인지 확인합니다.

```diff
+	fault_read_limit(3);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "short reads still work\n") == 0);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "last") == 0);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(fault_read_calls() > 2);
```

지정한 positive-count read 하나를 EIO로 만들 수도 있습니다.

```diff
+	fault_read_limit(4);
+	fault_read_fail_on(fd, 3);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(fault_read_calls() == 3);
+	CHECK(fault_read_failed());
```

### 이 baseline이 증명하는 것과 한계는 무엇인가

short positive read를 이어 붙여 complete line과 EOF tail을 만드는 behavior, first/middle read error의 당시 cleanup, allocation ownership counter를 검증합니다. 그러나 이 exact SHA의 middle-error fixture는 partial bytes가 **discard**되는 legacy policy를 기대합니다. harness도 한 call의 EIO와 request cap만 제공하며 `EINTR → progress → EAGAIN` 같은 ordered sequence는 만들지 못합니다.

### 관련 커밋

`f0055ae5cf19`는 이 baseline과 달리 explicit context에서 accepted bytes를 transient/terminal error 뒤 유지하고, wait를 별도 result로 표현합니다. `11033bd85c59`는 baseline harness를 ordered errno script로 확장합니다.

## f0055ae5cf19 — fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존
**중요도** `S` · **태그** `CORE, POSIX_IO, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

이전 engine은 모든 negative `read`를 `BLR_ERROR`로 처리했습니다. `EINTR`는 system call이 signal에 의해 중단됐다는 뜻일 뿐 stream failure가 아니고, `EAGAIN`은 nonblocking fd에 현재 data가 없다는 뜻일 뿐 EOF가 아닙니다. 둘을 terminal error와 합치면 caller가 잘못된 recovery 결정을 내립니다.

결정은 `EINTR`를 wrapper 안에서 재시도하고, `EAGAIN`/`EWOULDBLOCK`을 `BLR_AGAIN`으로 분리하며, positive read에서만 `end`를 전진시키는 기존 publish 순서를 유지하는 것입니다.

### 무엇이 바뀌었는가 (diff)

모든 descriptor probe와 data read가 같은 retry wrapper를 사용합니다.

```diff
+#include <errno.h>
+
+static ssize_t	read_retrying(int fd, void *buffer, size_t count)
+{
+	ssize_t	read_size;
+
+	read_size = read(fd, buffer, count);
+	while (read_size < 0 && errno == EINTR)
+		read_size = read(fd, buffer, count);
+	return (read_size);
+}
```

public result enum에 wait state가 추가됩니다.

```diff
 typedef enum e_blr_result
 {
 	BLR_ERROR = -1,
 	BLR_EOF = 0,
-	BLR_LINE = 1
+	BLR_LINE = 1,
+	BLR_AGAIN = 2
 }t_blr_result;
```

read loop가 음수로 끝난 뒤 errno를 분류합니다.

```diff
 	if (read_size < 0)
+	{
+		if (errno == EAGAIN || errno == EWOULDBLOCK)
+			return (BLR_AGAIN);
 		return (BLR_ERROR);
+	}
```

compatibility adapter는 `BLR_AGAIN`에서만 hidden node를 유지합니다.

```diff
 	result = blr_reader_next(reader, &line);
 	if (result == BLR_LINE)
 		return (line);
-	discard_legacy_reader(reader);
+	if (result != BLR_AGAIN)
+		discard_legacy_reader(reader);
 	return (NULL);
```

### accepted bytes는 왜 사라지지 않는가

`read_retrying`의 destination은 `reader->bytes + reader->end`이고, `read_size > 0`인 경우에만 `end += read_size`와 sentinel write가 실행됩니다. `EAGAIN`이나 다른 error가 발생한 call은 indices를 바꾸지 않으므로 이전 positive read가 이미 publish한 bytes가 그대로 남습니다.

```text
read "par"  -> begin=0, scan=3, end=3
EAGAIN      -> begin=0, scan=3, end=3, BLR_AGAIN
read "tial\n" -> end 증가, scan=3부터 새 범위 검사
```

explicit caller는 `BLR_ERROR` 뒤에도 같은 context를 retry·reset·destroy할 수 있습니다. legacy caller는 return type 때문에 EOF·wait·error를 모두 `NULL`로 보지만, hidden state retention은 AGAIN에서만 제공됩니다.

### 무엇을 보장하고 무엇을 보장하지 않는가

`EINTR`가 caller-visible result가 되지 않고, nonblocking wait가 EOF와 분리되며, accepted bytes가 AGAIN/ERROR 때문에 소비되거나 지워지지 않음을 보장합니다. 계속 `EINTR`만 발생하는 환경에서 retry 횟수의 상한은 없고, readiness를 기다리는 `poll`/`select` 자체는 제공하지 않습니다.

### 관련 커밋

`f3504f674c73`은 실제 `O_NONBLOCK` pipe에서 wait와 resume를 확인합니다. `11033bd85c59`는 exact `EINTR`, progress, `EAGAIN`, EIO 순서를 주입해 cursor preservation을 더 직접적으로 검증합니다.

## f3504f674c73 — test(reader): 비차단 부분 입력 보존 검증
**중요도** `A` · **태그** `TEST, POSIX_IO, EDGE`

### 무엇을 검증하는가

pipe read end에 실제 `O_NONBLOCK`을 설정하고 writer를 단계적으로 사용합니다.

```diff
+static int	make_nonblocking_pipe(int fds[2])
+{
+	int	flags;
+
+	if (pipe(fds) != 0)
+		return (0);
+	flags = fcntl(fds[0], F_GETFL);
+	if (flags < 0 || fcntl(fds[0], F_SETFL, flags | O_NONBLOCK) < 0)
+	{
+		close(fds[0]);
+		close(fds[1]);
+		return (0);
+	}
+	return (1);
+}
```

explicit context fixture는 partial bytes, wait, complete line, buffered suffix, EOF tail을 한 sequence로 검사합니다.

```diff
+	CHECK(write(fds[1], "part", 4) == 4);
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	CHECK(write(fds[1], "ial\nnext", 8) == 8);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "partial\n") == 0);
+	free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	close(fds[1]);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "next") == 0);
+	free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
```

compatibility fixture도 `"leg"` 뒤 첫 `NULL`에서 hidden state를 유지하고, `"acy\n"`이 도착한 뒤 `"legacy\n"`을 반환하는지 확인합니다.

### 무엇을 증명하고 무엇을 증명하지 않는가

실제 kernel nonblocking pipe에서 partial input이 write call 사이에 보존되고, writer가 열린 동안 unterminated suffix를 EOF tail로 조기 반환하지 않으며, close 뒤 tail과 stable EOF가 순서대로 나옴을 증명합니다. 실제 asynchronous signal delivery가 만든 `EINTR`나 terminal EIO 뒤 resume는 이 fixture의 범위가 아닙니다.

### 관련 커밋

`f0055ae5cf19`의 `BLR_AGAIN` mapping과 adapter retention이 직접 검증 대상입니다. `11033bd85c59`는 실제 scheduler에 맡기기 어려운 errno ordering을 deterministic harness로 보완합니다.

## 11033bd85c59 — test(failure): EINTR·EAGAIN·I/O 오류 순서 검증
**중요도** `A` · **태그** `TEST, POSIX_IO, RISK`

### 왜 다른 기법이 필요한가

실제 pipe는 `EAGAIN`을 만들 수 있지만 정확히 첫 call은 `EINTR`, 둘째는 progress, 셋째는 `EAGAIN`으로 강제하기 어렵습니다. 기존 single-failure harness에 per-call errno array를 추가합니다.

```diff
+#define MAX_READ_SCRIPT 64
+
+static int		g_read_script[MAX_READ_SCRIPT];
+static size_t	g_read_script_length;
+
+void	fault_read_script(int fd, const int *errors, size_t length)
+{
+	size_t	index;
+
+	if (length > MAX_READ_SCRIPT)
+		length = MAX_READ_SCRIPT;
+	g_read_fd = fd;
+	g_read_calls = 0;
+	g_read_fail_at = 0;
+	g_read_script_length = length;
+	index = 0;
+	while (index < length)
+	{
+		g_read_script[index] = errors[index];
+		index++;
+	}
+}
```

`test_read`는 script의 nonzero entry에서 지정 errno와 `-1`을 반환하고, zero entry에서는 실제 read를 수행합니다.

```diff
+	if (g_read_calls <= g_read_script_length
+		&& g_read_script[g_read_calls - 1] != 0)
+	{
+		g_read_failed = 1;
+		errno = g_read_script[g_read_calls - 1];
+		return (-1);
+	}
```

첫 regression은 `{EINTR, 0, EAGAIN}`와 read limit 3을 사용합니다.

```diff
+	const int	errors[] = {EINTR, 0, EAGAIN};
+	fault_read_limit(3);
+	fault_read_script(fd, errors, sizeof(errors) / sizeof(errors[0]));
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	CHECK(fault_read_calls() == 3);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "partial\n") == 0);
```

둘째 regression은 progress 뒤 EIO를 반환하고 같은 context의 retry가 complete line을 만드는지 확인합니다.

```diff
+	const int	errors[] = {0, EIO};
+	fault_read_limit(3);
+	fault_read_script(fd, errors, sizeof(errors) / sizeof(errors[0]));
+	CHECK(blr_reader_next(reader, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "recoverable\n") == 0);
```

### 무엇을 증명하고 무엇을 증명하지 않는가

지정된 sequence에서 `EINTR`가 wrapper 내부에서 retry되고, positive progress 뒤 `EAGAIN`이나 EIO가 와도 `begin/scan/end`가 duplicate·skip 없이 이어지는지 증명합니다. 모든 POSIX errno, 실제 signal timing, 64개를 넘는 script, compatibility API의 terminal-error retention, concurrent access는 증명하지 않습니다.

### 관련 커밋

`f0055ae5cf19`의 retry/result policy가 production 대상이고, `fd03a831686b`의 one-shot EIO harness가 이 ordered script의 기반입니다. 실제 nonblocking behavior는 `f3504f674c73`가 별도의 integration evidence로 제공합니다.

## 이 Thread의 경계

이 문서는 system-call result를 parser state로 해석하고 accepted bytes를 보존하는 책임에 한정합니다.

- readiness를 기다리는 event loop나 `poll`/`select`/`epoll` integration은 제공하지 않습니다.
- terminal I/O error 뒤 context가 bytes를 보존해도 error 원인이 자동으로 복구된다는 뜻은 아닙니다.
- descriptor별 hidden node와 explicit context ownership은 `02-singleton-to-descriptor-scoped-state.md`와 `03-explicit-reader-lifetime-and-authoritative-engine.md`가 담당합니다.

> 검토 범위: `fd03a831686b`, `f0055ae5cf19`, `f3504f674c73`, `11033bd85c59`의 diff와 해당 SHA의 관련 source와 test harness를 확인했습니다. 저장소 checkout, build, nonblocking test, fault suite, sanitizer는 실행하지 않았습니다.
