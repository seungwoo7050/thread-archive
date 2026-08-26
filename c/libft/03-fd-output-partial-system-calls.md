# Thread: 부분 system call에서도 FD 출력을 끝까지 진행하기

> Project: `libft` · Branch: `c/libft` · 문서 번호: 03

## 개요

이 Thread는 public API가 error status를 반환하지 않는 조건에서도 내부 출력 경로가 system call의 진행 상태를 끝까지 소유해야 한다는 문제를 다룬다. history는 formatting surface를 만드는 단계, 공통 경로를 준비하는 단계, progress invariant를 복구하는 단계, rare failure를 결정적으로 관찰하는 단계로 나뉜다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `26509fd54c3d` | `feat(io): 파일 디스크립터 출력 함수 추가` | B | `FD_OUTPUT, CORE` | void-returning descriptor API와 one-shot `write` 기반의 초기 구현을 추가한다. |
| 2 | `60c35f2fb431` | `test(io): 파일 디스크립터 출력 검증` | B | `FD_OUTPUT, TEST` | normal byte sequence와 host pipe 오류의 초기 관찰 범위를 고정한다. |
| 3 | `1077556d1c4b` | `refactor(io): 숫자 출력을 자릿수 helper로 분리` | B | `FD_OUTPUT, REFACTOR` | 숫자 각 자릿수를 공통 character-output path로 보낸다. |
| 4 | `3f2bfbf11e1f` | `fix(io): 파일 디스크립터 출력을 끝까지 재시도` | S | `FD_OUTPUT, CORE, RISK` | progress loop, `EINTR` retry, zero-progress rejection, 후속 component 중단을 구현한다. |
| 5 | `b013c926ceb5` | `test(io): 부분 쓰기와 EINTR 이후 진행을 검증` | A | `FD_OUTPUT, TEST, RISK` | scripted `write`로 정확한 request sequence와 실패 중단을 검증한다. |

---

**역할군 1 — 정상 formatting API를 만들고 공통 출력 경로를 준비한다.**

## 26509fd54c3d — feat(io): 파일 디스크립터 출력 함수 추가
**중요도** `B` · **태그** `FD_OUTPUT, CORE`

### 무엇을 만들었는가 (diff)

`src/io/ft_fd_output.c`와 public declarations가 추가된다.

```diff
+void	ft_putchar_fd(char character, int fd)
+{
+	(void)write(fd, &character, 1);
+}
+
+void	ft_putstr_fd(char *text, int fd)
+{
+	(void)write(fd, text, ft_strlen(text));
+}
+
+void	ft_putendl_fd(char *text, int fd)
+{
+	ft_putstr_fd(text, fd);
+	ft_putchar_fd('\n', fd);
+}
+
+void	ft_putnbr_fd(int number, int fd)
+{
+	char		buffer[11];
+	unsigned int	magnitude;
+	size_t		index;
+
+	/* ... */
+	(void)write(fd, buffer + index, sizeof(buffer) - index);
+}
```

public API는 모두 `void`다. character와 string은 한 번씩 `write`하고, newline은 string 뒤 character helper를 호출한다. number는 `INT_MIN`도 처리할 수 있도록 signed negation을 피한 magnitude를 만들고 최종 buffer를 한 번 제출한다.

### 왜 가볍게 다루는가

이 커밋의 관심은 정상 formatting과 API surface다. 모든 `write` 반환값을 버리므로 positive short write가 남긴 suffix, `EINTR`, zero progress, permanent error를 구분하지 못한다. `ft_putendl_fd`도 앞선 string write가 실패했는지 모른 채 newline을 시도한다.

### 어떤 커밋과 왜 연결되는가

`60c35f2fb431`은 이 초기 normal output과 host 오류를 관찰한다. `1077556d1c4b`은 number를 common character path로 옮기고, `3f2bfbf11e1f`은 모든 public helper 아래에 completion-aware write primitive를 넣는다.

## 60c35f2fb431 — test(io): 파일 디스크립터 출력 검증
**중요도** `B` · **태그** `FD_OUTPUT, TEST`

### 무엇을 검증하는가

`tests/test_fd_output.c`는 ordinary pipe의 write end에 네 helper를 조합하고, close 뒤 read한 byte sequence를 고정 문자열과 비교한다. `0`, `-1`, `INT_MAX`, `INT_MIN`이 포함되어 number formatting의 극값도 함께 통과한다.

```diff
+static void	check_output(void)
+{
+	static const char	expected[] =
+		"Afoundation\n0|-1|2147483647|-2147483648";
+	char			actual[sizeof(expected) + 8];
+	int			pipe_fd[2];
+	ssize_t			bytes_read;
+
+	CHECK(pipe(pipe_fd) == 0);
+	ft_putchar_fd('A', pipe_fd[1]);
+	ft_putendl_fd("foundation", pipe_fd[1]);
+	ft_putnbr_fd(0, pipe_fd[1]);
+	/* ... */
+	CHECK(close(pipe_fd[1]) == 0);
+	bytes_read = read(pipe_fd[0], actual, sizeof(actual));
+	CHECK(bytes_read == (ssize_t)(sizeof(expected) - 1));
+	if (bytes_read == (ssize_t)(sizeof(expected) - 1))
+		CHECK(memcmp(actual, expected, sizeof(expected) - 1) == 0);
+}
```

broken pipe case는 `SIGPIPE`를 잠시 무시하고 read end를 닫은 뒤 `errno == EPIPE`인지 확인한다. 닫힌 descriptor에 네 helper를 호출하는 case도 있지만 exact `errno`나 호출 횟수를 assertion하지 않고, test process가 계속 진행하는지만 간접적으로 포함한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

선택한 normal formatting의 길이·순서와 suppressed `SIGPIPE` 아래 `EPIPE` 관찰을 증명한다. kernel이 short write나 `EINTR`을 반환하도록 통제하지 않으므로 one-shot 구현의 progress 결함은 잡지 못한다.

### 어떤 커밋과 왜 연결되는가

이 테스트는 `26509fd54c3d`의 정상 결과를 baseline으로 남긴다. `b013c926ceb5`는 같은 API를 대상으로 host timing 대신 scripted result를 사용해 이 테스트가 다루지 못한 부분 호출을 검증한다.

## 1077556d1c4b — refactor(io): 숫자 출력을 자릿수 helper로 분리
**중요도** `B` · **태그** `FD_OUTPUT, REFACTOR`

### 무엇이 바뀌었는가 (diff)

`src/io/ft_fd_output.c`의 number path만 재구성한다.

```diff
+static void	put_unsigned(unsigned int magnitude, int fd)
+{
+	if (magnitude >= 10U)
+		put_unsigned(magnitude / 10U, fd);
+	ft_putchar_fd((char)('0' + magnitude % 10U), fd);
+}
+
 void	ft_putnbr_fd(int number, int fd)
 {
-	char		buffer[11];
 	unsigned int	magnitude;
-	size_t		index;
 
 	if (number < 0)
+	{
+		ft_putchar_fd('-', fd);
 		magnitude = (unsigned int)(-(number + 1)) + 1U;
+	}
 	else
 		magnitude = (unsigned int)number;
-	index = sizeof(buffer);
-	while (magnitude >= 10U)
-	{
-		index--;
-		buffer[index] = (char)('0' + magnitude % 10U);
-		magnitude /= 10U;
-	}
-	index--;
-	buffer[index] = (char)('0' + magnitude);
-	if (number < 0)
-	{
-		index--;
-		buffer[index] = '-';
-	}
-	(void)write(fd, buffer + index, sizeof(buffer) - index);
+	put_unsigned(magnitude, fd);
 }
```

### 무엇을 준비하고 무엇은 아직 없는가

number output을 sign과 recursive unsigned digit emission으로 분리하고, 각 digit을 `ft_putchar_fd`로 보낸다. formatting result는 유지되지만 one-shot character write의 반환값은 여전히 버린다. 오히려 number 하나가 여러 system call로 나뉘므로, 앞선 digit에서 실패했을 때 이후 digit을 멈출 내부 status propagation이 필요해진다.

### 어떤 커밋과 왜 연결되는가

`3f2bfbf11e1f`은 `put_unsigned`의 반환형을 `int`로 바꾸고 recursion 실패를 상위 frame으로 전파한다. 따라서 이 refactor는 공통 completion path를 만들기 위한 중간 단계다.

---

**역할군 2 — system call progress를 복구하고 결정적으로 검증한다.**

## 3f2bfbf11e1f — fix(io): 파일 디스크립터 출력을 끝까지 재시도
**중요도** `S` · **태그** `FD_OUTPUT, CORE, RISK`

### 왜 바뀌었는가 (문제→원인→결정)

`src/io/ft_fd_output.c`의 이전 구현은 `write`가 요청 길이보다 작은 양수를 반환해도 성공으로 끝냈고, `EINTR`에도 남은 byte를 재시도하지 않았다. 원인은 public `void` API가 아니라, 내부에서 progress와 failure를 표현하는 helper가 없었다는 데 있다. 결정은 private `write_all`이 offset을 소유하고 성공 여부를 `int`로 반환하게 하는 것이다.

### progress를 보존하는 코드는 어떻게 동작하는가

상태 전이가 조밀하므로 핵심 helper는 주석을 붙인 최종 코드로 본다.

```c
static int	write_all(int fd, const char *buffer, size_t length)
{
	ssize_t	written;
	size_t	offset;
	size_t	request;

	offset = 0;
	while (offset < length)
	{
		request = length - offset;
		if (request > (size_t)SSIZE_MAX)
			request = (size_t)SSIZE_MAX;
		written = write(fd, buffer + offset, request);
		if (written > 0)
			// 실제로 전달된 byte만큼만 진행한다.
			offset += (size_t)written;
		else if (written < 0 && errno == EINTR)
			// progress가 없으므로 같은 offset과 remaining length로 재시도한다.
			continue ;
		else
		{
			// 0은 loop를 영원히 반복시킬 수 있으므로 permanent failure로 바꾼다.
			if (written == 0)
				errno = EIO;
			return (0);
		}
	}
	return (1);
}
```

positive short write만 offset을 증가시킨다. `EINTR`은 같은 suffix를 다시 제출하고, zero progress는 `EIO`로 바꾼 뒤 중단한다. 다른 negative result는 `errno`를 유지한 채 실패를 반환한다.

### public `void`를 유지하면서 후속 출력을 어떻게 멈추는가 (diff)

```diff
 void	ft_putendl_fd(char *text, int fd)
 {
-	ft_putstr_fd(text, fd);
-	ft_putchar_fd('\n', fd);
+	if (write_all(fd, text, ft_strlen(text)))
+		(void)write_all(fd, "\n", 1);
 }
 
-static void	put_unsigned(unsigned int magnitude, int fd)
+static int	put_unsigned(unsigned int magnitude, int fd)
 {
-	if (magnitude >= 10U)
-		put_unsigned(magnitude / 10U, fd);
-	ft_putchar_fd((char)('0' + magnitude % 10U), fd);
+	char	digit;
+
+	if (magnitude >= 10U && !put_unsigned(magnitude / 10U, fd))
+		return (0);
+	digit = (char)('0' + magnitude % 10U);
+	return (write_all(fd, &digit, 1));
 }
 
 	if (number < 0)
 	{
-		ft_putchar_fd('-', fd);
+		if (!write_all(fd, "-", 1))
+			return ;
 		magnitude = (unsigned int)(-(number + 1)) + 1U;
 	}
-	put_unsigned(magnitude, fd);
+	(void)put_unsigned(magnitude, fd);
```

public signature는 바뀌지 않지만 composite helper는 private success bit를 사용한다. string write가 실패하면 newline을 쓰지 않고, sign이나 상위 digit이 실패하면 아래 digit emission도 멈춘다.

### 무엇을 보장하고 무엇은 남기는가

positive progress가 있는 동안 suffix를 끝까지 제출하고, `EINTR`을 retry하며, zero 또는 permanent error 뒤에는 같은 composite operation의 후속 component를 출력하지 않는다. public caller에게 error status를 돌려주지는 않고 `errno`와 실제 bytes만 외부에 남긴다. output atomicity, 여러 thread의 interleaving, `NULL` text 방어는 이 fix의 범위가 아니다.

### 어떤 커밋과 왜 연결되는가

`1077556d1c4b`이 만든 recursive digit path에 failure propagation을 추가한다. `b013c926ceb5`는 이 helper의 offset·request 변화와 composite stop branch를 exact call sequence로 검증한다.

## b013c926ceb5 — test(io): 부분 쓰기와 EINTR 이후 진행을 검증
**중요도** `A` · **태그** `FD_OUTPUT, TEST, RISK`

### 왜 다른 기법이 필요한가

ordinary pipe test는 정확히 어느 `write`가 short result, `EINTR`, zero, permanent error를 반환할지 통제할 수 없다. 이 커밋은 `ft_fd_output.c`만 `-Dwrite=test_write`로 다시 compile하고, test가 준비한 `(result, errno)` sequence를 한 호출씩 소비하게 한다.

```diff
+WRITE_OBJ_DIR := build/write-failure
+WRITE_OUTPUT_OBJ := $(WRITE_OBJ_DIR)/ft_fd_output.o
+WRITE_BIN := tests/bin/test_write_failure
+WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
+	tests/support/fail_write.c
+WRITE_DEFINES := -Dwrite=test_write
+
+$(WRITE_OUTPUT_OBJ): src/io/ft_fd_output.c libft.h \
+		tests/support/fail_write.h
+	$(CC) $(CPPFLAGS) $(WRITE_DEFINES) $(CFLAGS) \
+		$(DEPFLAGS) -c $< -o $@
```

대표 sequence `{2, 0}, {-1, EINTR}, {1, 0}, {3, 0}`를 `"abcdef"`에 적용하면 request는 `6 → 4 → 4 → 3`이 된다. 첫 call의 2-byte progress 뒤 남은 길이는 4이고, `EINTR`은 progress를 만들지 않으므로 다음 request도 4다.

```diff
+	static const t_write_step	steps[] = {
+	{2, 0}, {-1, EINTR}, {1, 0}, {3, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	ft_putstr_fd("abcdef", 91);
+	VERIFY(test_writer_calls() == 4);
+	VERIFY(test_writer_request(0) == 6);
+	VERIFY(test_writer_request(1) == 4);
+	VERIFY(test_writer_request(2) == 4);
+	VERIFY(test_writer_request(3) == 3);
+	VERIFY(test_writer_output_size() == 6);
+	VERIFY(memcmp(test_writer_output(), "abcdef", 6) == 0);
```

같은 binary는 다음 다섯 경로를 구분한다.

| case | scripted 결과 | 확인하는 것 |
| --- | --- | --- |
| partial + `EINTR` | `2, EINTR, 1, 3` | suffix request와 동일-offset retry |
| zero progress | `2, 0, ...` | 두 call 뒤 중단하고 `errno = EIO` |
| permanent error | `1, EIO, ...` | `ft_putendl_fd`가 newline을 쓰지 않음 |
| `INT_MIN` retry | digit 중간 `EINTR` | sign과 모든 digits의 최종 byte sequence |
| number error | sign·첫 digit 뒤 `EPIPE` | 이후 digit을 출력하지 않음 |

### 무엇을 증명하고 무엇은 증명하지 않는가

scripted sequence에서 production `write_all`의 request length, call count, output bytes, error-stop behavior가 예상과 일치함을 증명한다. 실제 device별 short-write 조건, real signal handler와의 동시성, thread-safe serialization은 증명하지 않는다.

### 어떤 커밋과 왜 연결되는가

이 test seam은 `3f2bfbf11e1f`의 private progress invariant를 직접 관찰한다. `60c35f2fb431`이 남긴 normal formatting baseline과 역할이 달라, 두 테스트를 함께 봐야 정상 결과와 rare failure path를 모두 설명할 수 있다.

## 이 Thread의 경계

이 Thread는 descriptor output에서 short write progress, `EINTR` retry, zero-progress failure, composite stop을 다룬다. allocation rollback은 [`02-single-allocation-to-rollback-safe-ownership`](02-single-allocation-to-rollback-safe-ownership.md), archive·compiler·sanitizer orchestration은 [`04-static-archive-release-verification`](04-static-archive-release-verification.md)의 관심사다. public API에 status return을 추가하는 설계, buffering, output atomicity, thread synchronization은 별개의 문제다.

> 검토 범위: `26509fd54c3d`, `60c35f2fb431`, `1077556d1c4b`, `3f2bfbf11e1f`, `b013c926ceb5`의 commit diff와 해당 SHA의 `src/io/ft_fd_output.c`, `tests/test_fd_output.c`, write-failure Makefile rules, `tests/failure/test_fd_output_failure.c`, `tests/support/fail_write.c`를 확인했다. ordinary test binary와 write-failure binary는 실행하지 않았다.
