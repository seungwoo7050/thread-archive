# Thread: 부분 system call에서도 FD 출력을 끝까지 진행하기

> Project: `libft`  
> Branch: `c/libft`

## 개요

파일 디스크립터 출력 API는 처음부터 `void`다. caller는 반환값으로 성공 byte 수나 오류를 받을 수 없다. 그렇다고 one-shot `write`의 반환값을 버려도 되는 것은 아니다.

이 Thread가 확립하는 내부 규칙은 다음과 같다.

> 양수의 짧은 write 뒤에는 그만큼만 진행하고 남은 suffix를 다시 요청한다. `EINTR`은 progress 없이 재시도한다. 0 byte progress와 영구 오류는 실패로 취급하며, 복합 출력은 앞 구성요소가 실패한 뒤 후속 byte를 쓰지 않는다.

이 규칙은 이미 출력된 prefix를 되돌리는 atomicity를 제공하지 않는다. 대신 byte를 중복하거나 건너뛰지 않고, 실패가 확정된 뒤 newline이나 나머지 숫자 자릿수를 추가로 출력하지 않게 한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `26509fd54c3d` | `feat(io): 파일 디스크립터 출력 함수 추가` | B | `FD_OUTPUT`, `CORE` | `void` FD API와 one-shot write 구현을 도입한다. |
| 2 | `60c35f2fb431` | `test(io): 파일 디스크립터 출력 검증` | B | `FD_OUTPUT`, `TEST` | 정상 byte sequence, closed descriptor, broken pipe의 초기 관찰을 추가한다. |
| 3 | `1077556d1c4b` | `refactor(io): 숫자 출력을 자릿수 helper로 분리` | B | `FD_OUTPUT`, `REFACTOR` | 숫자 sign/digit 출력을 공통 character path로 모은다. |
| 4 | `3f2bfbf11e1f` | `fix(io): 파일 디스크립터 출력을 끝까지 재시도` | S | `FD_OUTPUT`, `CORE`, `RISK` | progress loop, `EINTR` retry, zero-progress rejection, component stop을 구현한다. |
| 5 | `b013c926ceb5` | `test(io): 부분 쓰기와 EINTR 이후 진행을 검증` | A | `FD_OUTPUT`, `TEST`, `RISK` | scripted `write` 결과로 exact request/progress/stop sequence를 검증한다. |

## `26509fd54c3d` — formatting은 맞지만 write 결과를 사용하지 않는다

이 커밋은 네 public 함수를 추가한다.

```c
void	ft_putchar_fd(char character, int fd);
void	ft_putstr_fd(char *text, int fd);
void	ft_putendl_fd(char *text, int fd);
void	ft_putnbr_fd(int number, int fd);
```

초기 구현은 각 component를 한 번의 `write`에 제출하고 반환값을 버린다.

```c
void	ft_putchar_fd(char character, int fd)
{
	(void)write(fd, &character, 1);
}

void	ft_putstr_fd(char *text, int fd)
{
	(void)write(fd, text, ft_strlen(text));
}
```

`ft_putendl_fd`는 string을 출력한 뒤 무조건 newline helper를 호출한다. number는 stack buffer 뒤에서부터 decimal representation을 만든 뒤 sign과 digits를 한 번에 쓴다.

```c
(void)write(fd, buffer + index, sizeof(buffer) - index);
```

`INT_MIN`은 `-number`를 직접 계산하지 않고 다음 식으로 magnitude를 만든다.

```c
magnitude = (unsigned int)(-(number + 1)) + 1U;
```

따라서 formatting 자체는 signed overflow를 피한다. 하지만 system call 결과에 대해서는 다음 상태를 구분하지 않는다.

- 요청 길이와 같은 양수: 전부 출력
- 요청보다 작은 양수: prefix만 출력됐지만 suffix 재시도 없음
- `-1/EINTR`: 아무 byte도 출력되지 않았지만 재시도 없음
- 영구 오류: caller나 다음 component에 전달되는 내부 신호 없음

특히 `ft_putendl_fd`는 string write가 실패해도 newline을 계속 시도할 수 있다.

## `60c35f2fb431` — 정상 formatting과 host 오류만 관찰한다

초기 test는 pipe write end에 네 helper를 조합하고 다음 byte sequence를 확인한다.

```text
Afoundation\n0|-1|2147483647|-2147483648
```

또한 `SIGPIPE`를 잠시 무시한 상태에서 read end가 닫힌 pipe에 문자열을 쓰고 `errno == EPIPE`를 확인한다. closed descriptor에도 네 함수를 호출하지만, 그 부분에는 exact `errno`나 호출 수 assertion이 없다. 함수가 control을 반환해 test process가 계속 진행하는 정도만 포함된다.

이 테스트는 작은 정상 output과 host가 자연스럽게 만드는 `EPIPE`를 관찰한다. kernel `write`가 원하는 시점에 short write나 `EINTR`을 반환하도록 제어하지 않으므로, partial-progress contract는 검증하지 못한다.

## `1077556d1c4b` — 숫자를 공통 출력 경로로 옮긴 중간 단계

number buffer가 사라지고, 상위 자릿수부터 재귀적으로 출력하는 helper가 추가된다.

```c
static void	put_unsigned(unsigned int magnitude, int fd)
{
	if (magnitude >= 10U)
		put_unsigned(magnitude / 10U, fd);
	ft_putchar_fd((char)('0' + magnitude % 10U), fd);
}
```

negative sign도 `ft_putchar_fd`로 출력한다. 이 refactor는 모든 숫자 byte를 공통 character primitive로 모으지만, helper와 public 함수가 모두 `void`이므로 앞선 sign이나 digit write가 실패해도 뒤 자릿수가 계속 호출된다.

즉 이 커밋은 format decomposition을 단순하게 만들면서, 곧이어 필요한 실패 전파 위치를 분명하게 드러낸다.

## `3f2bfbf11e1f` — write를 “한 번 호출”이 아니라 “progress를 끝낼 때까지”로 바꾸다

### 핵심 loop

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
			offset += (size_t)written;
		else if (written < 0 && errno == EINTR)
			continue ;
		else
		{
			if (written == 0)
				errno = EIO;
			return (0);
		}
	}
	return (1);
}
```

state 변화는 반환값별로 다르다.

| `write` 결과 | `offset` | 다음 동작 |
| --- | ---: | --- |
| `written > 0` | 정확히 `written`만큼 증가 | 남은 `buffer + offset` 재요청 |
| `written < 0 && errno == EINTR` | 변화 없음 | 같은 suffix 재시도 |
| `written == 0` | 변화 없음 | `errno = EIO`, 실패 |
| 그 밖의 음수 | 변화 없음 | 현재 `errno`를 유지하고 실패 |

`length > SSIZE_MAX`인 경우에도 한 호출의 request가 `ssize_t` 표현 범위를 넘지 않도록 잘라낸다. `length == 0`이면 loop에 들어가지 않고 성공으로 처리한다.

### public `void`를 유지하면서 내부 실패를 전달한다

public signature는 바뀌지 않는다. 단일 component 함수는 최종 boolean을 caller에게 반환할 수 없지만, 복합 함수는 private helper 결과로 후속 출력을 막는다.

```c
void	ft_putendl_fd(char *text, int fd)
{
	if (write_all(fd, text, ft_strlen(text)))
		(void)write_all(fd, "\n", 1);
}
```

string이 완성되지 않으면 newline을 시도하지 않는다.

숫자 helper도 `int`를 반환하도록 바뀐다.

```c
static int	put_unsigned(unsigned int magnitude, int fd)
{
	char	digit;

	if (magnitude >= 10U && !put_unsigned(magnitude / 10U, fd))
		return (0);
	digit = (char)('0' + magnitude % 10U);
	return (write_all(fd, &digit, 1));
}
```

상위 자릿수 출력이 실패하면 현재와 이후의 낮은 자릿수를 쓰지 않는다. 음수 sign write가 실패한 경우에는 digit recursion 자체를 시작하지 않는다.

```c
if (number < 0)
{
	if (!write_all(fd, "-", 1))
		return ;
	...
}
```

이 fix가 보장하는 것은 “이미 쓴 prefix + 아직 쓰지 않은 suffix”의 정확한 progress다. 이미 kernel에 전달된 prefix를 취소할 수는 없으므로 전체 message의 all-or-nothing 출력은 보장하지 않는다.

## `b013c926ceb5` — system timing 대신 scripted `write`를 사용한다

### targeted substitution

production archive 전체를 바꾸지 않고 `src/io/ft_fd_output.c`만 `write=test_write`로 다시 compile한다.

```make
WRITE_DEFINES := -Dwrite=test_write

$(WRITE_OUTPUT_OBJ): src/io/ft_fd_output.c libft.h \
		tests/support/fail_write.h
	$(CC) $(CPPFLAGS) $(WRITE_DEFINES) $(CFLAGS) \
		$(DEPFLAGS) -c $< -o $@
```

test writer는 각 호출에 대해 반환값과 `errno`를 미리 정한 step을 소비한다. 동시에 fd, request length, 누적 output, 허용되지 않은 호출을 기록한다. 따라서 실제 pipe buffer 크기나 signal timing에 의존하지 않고 exact sequence를 재현한다.

### 검증하는 다섯 sequence

| case | scripted 결과 | 확인하는 규칙 |
| --- | --- | --- |
| partial + `EINTR` | `2`, `-1/EINTR`, `1`, `3` | request가 `6 → 4 → 4 → 3`으로 줄고 `abcdef`가 중복 없이 완성된다. |
| zero progress | `2`, `0`, 뒤에 사용되면 `4` | 두 번째 호출에서 멈추고 output은 `ab`, `errno == EIO`다. |
| permanent error in `putendl` | `1`, `-1/EIO`, 뒤에 사용되면 `5` | string prefix `e` 뒤에 남은 text와 newline을 쓰지 않는다. |
| `INT_MIN` retry | 1-byte 성공들 사이 `-1/EINTR` | sign과 10개 digit이 완성되고 EINTR 위치의 같은 1 byte를 다시 요청한다. |
| number permanent error | sign 성공, 첫 digit 성공, `-1/EPIPE` | output은 `-2`에서 멈추고 뒤 digit 호출이 없다. |

첫 case의 request 변화가 핵심이다.

```text
call 1: buffer[0..6), request 6 -> 2 bytes
call 2: buffer[2..6), request 4 -> EINTR
call 3: buffer[2..6), request 4 -> 1 byte
call 4: buffer[3..6), request 3 -> 3 bytes
```

`EINTR`은 progress를 만들지 않았으므로 call 2와 call 3의 request와 시작 위치가 같다.

이 test는 production `write_all`의 분기와 복합 출력 중단을 결정적으로 증명한다. 실제 kernel의 pipe/socket/device가 어떤 조건에서 short write를 만드는지, signal handler와의 실제 동시성, thread-safe output은 증명하지 않는다.

## 최종 실행 흐름

```text
public fd output
  └─ write_all(buffer, length)
       ├─ positive short write → offset 증가 → suffix 재요청
       ├─ EINTR                → offset 유지 → 재시도
       ├─ zero                 → EIO → 실패
       └─ permanent error      → 실패

composite output
  ├─ putendl: text 성공일 때만 newline
  └─ putnbr:
       sign 실패 → 즉시 중단
       higher digit 실패 → lower digit 중단
```

## Thread의 범위

이 Thread는 부분 write의 progress, interruption retry, 영구 오류 이후의 중단을 다룬다. public API에 error return을 추가하는 설계, output atomicity, buffering, thread synchronization, `NULL` text 처리는 범위 밖이다.

> 검토 범위: 표시된 exact SHA의 commit diff와 source/test substitute를 확인했다. ordinary/failure test binary는 실행하지 않았다.
