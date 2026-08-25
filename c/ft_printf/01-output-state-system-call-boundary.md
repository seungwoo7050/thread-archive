# Thread: 출력 상태와 `write` 경계 — 단순 helper에서 진행 상태 머신까지

## 개요

이 Thread는 `ft_printf`의 출력이 “한 번의 `write`가 요청 길이를 전부 처리하면 성공”이라는 초기 구현에서 출발해, 모든 변환이 공유하는 출력 상태와 POSIX 부분 진행 규칙으로 바뀌는 과정을 다룹니다.

최종적으로 출력 계층이 소유하는 것은 세 가지입니다.

- **출력 대상**: `fd`
- **이미 운영체제가 받아들인 바이트 수**: `count`
- **이후 출력을 중단해야 하는 오류 상태**: `error`

`ft_printf_write`는 부분 쓰기에서 남은 suffix를 계속 기록하고, `EINTR`에서는 같은 요청을 다시 시도합니다. 한 번이라도 영구 오류나 0바이트 진행, 반환 길이 범위 초과가 발생하면 `error`를 고정하고 이후 호출을 거부합니다. 이 정책은 숫자·문자열·padding을 포함한 모든 출력 경로가 공유합니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `1d6a5cee3041` | feat(core): 리터럴과 퍼센트 출력 구현 | B | `CORE, OUTPUT` | local write/count helper와 public entry의 최소 경로 도입 |
| `3f7b0ab926d0` | feat(output): 출력 컨텍스트와 쓰기 API 추가 | S | `ARCH, OUTPUT, CORE` | FD·count·sticky error를 한 상태로 묶고 부분 쓰기 진행을 보존 |
| `78e5d25d7df6` | refactor(core): 리터럴 출력을 컨텍스트 API로 이관 | B | `OUTPUT, REFACTOR` | public literal 경로를 공통 출력 상태로 이동 |
| `c627bd1f85bb` | fix(output): 쓰기 결과를 집계하기 전에 범위 검증 | A | `OUTPUT, RISK, DEBUG` | `ssize_t` 결과를 `int`로 줄이기 전에 표현 가능 여부 확인 |
| `8a3ec50cb689` | fix(output): 중단된 쓰기 재시도와 요청 크기 제한 | S | `OUTPUT, CORE, RISK` | `SSIZE_MAX` 제한, `EINTR` 재시도, 결정적 test seam 도입 |
| `22e65c176b5d` | perf(output): 반복 채움을 묶어서 출력 | A | `OUTPUT, PERF` | padding을 64바이트 단위로 묶되 공통 오류/count 경로 유지 |
| `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | A | `OUTPUT, TEST, RISK` | 부분 진행·중단·0 진행·`EPIPE`·`SIGPIPE`·chunk 크기 검증 |

## `1d6a5cee3041` — “한 번에 전부 기록”을 성공으로 본 시작점

**중요도** B · **태그** `CORE, OUTPUT`

첫 구현에서 `ft_printf`는 local `count`를 직접 소유하고 리터럴과 `%%`를 한 바이트씩 출력합니다. `ft_write_count`는 요청 길이가 `INT_MAX`를 넘기지 않는지 먼저 확인한 뒤 `write(1, ...)`를 한 번 호출합니다.

```c
static int	ft_write_count(const char *buffer, int length, int *count)
{
	int	written;

	if (length <= 0)
		return (0);
	if (*count > INT_MAX - length)
		return (-1);
	written = (int)write(1, buffer, (size_t)length);
	if (written < 0 || written != length)
		return (-1);
	*count += written;
	return (0);
}
```

이 코드는 최소 기능에는 충분하지만, positive short write를 실패로 취급합니다. 예를 들어 8바이트를 요청하고 커널이 3바이트를 받아들였다면, 이미 발생한 외부 효과 3바이트를 진행 상태로 보존하지 않고 곧바로 `-1`로 끝냅니다. 남은 5바이트를 이어 쓰는 책임을 표현할 상태도 없습니다.

또한 FD가 상수 `1`, count와 오류가 entry 함수의 local convention으로 묶여 있습니다. 이후 conversion마다 별도의 출력 helper가 생기면 같은 집계·오류 규칙을 중복 구현하기 쉬운 구조입니다.

## `3f7b0ab926d0` — 공통 출력 상태를 만드는 결정

**중요도** S · **태그** `ARCH, OUTPUT, CORE`

이 커밋은 `t_printf`를 도입합니다.

```c
typedef struct s_printf
{
	int	fd;
	int	count;
	int	error;
}	t_printf;
```

세 필드는 독립적인 편의 값이 아닙니다. 한 출력 작업의 상태 전이를 함께 표현합니다.

```text
초기화
  fd = caller가 정한 descriptor
  count = 0
  error = 0

positive write
  count += written
  buffer += written
  remaining -= written

영구 실패 또는 0 진행
  error = 1
  이후 모든 write 요청 거부
```

새 `ft_printf_write`는 요청 전체가 끝날 때까지 반복합니다.

```c
while (length > 0)
{
	written = write(ctx->fd, buffer, length);
	if (written <= 0)
	{
		ctx->error = 1;
		return (-1);
	}
	if (ctx->count > INT_MAX - (int)written)
	{
		ctx->error = 1;
		return (-1);
	}
	ctx->count += (int)written;
	buffer += written;
	length -= (size_t)written;
}
```

여기서 중요한 변화는 “부분 쓰기를 허용했다”보다 더 큽니다. 이미 기록된 prefix와 아직 남은 suffix, public 반환값에 반영될 count, 이후 출력을 차단할 오류를 하나의 함수가 동시에 갱신합니다. 각 renderer는 자체적으로 `write` 결과를 해석하지 않고 이 API의 성공/실패만 따르면 됩니다.

다만 이 exact SHA에서는 새 모듈이 추가됐을 뿐, public `ft_printf`는 아직 이전 local helper를 사용합니다. architecture가 먼저 생기고 실제 entry가 다음 커밋에서 이관됩니다. 또한 다음 문제가 남아 있습니다.

- `write`의 `ssize_t` 결과를 `(int)`로 먼저 줄여 count 식에 사용합니다.
- 요청 `size_t`가 `SSIZE_MAX`보다 클 수 있습니다.
- `EINTR`도 영구 실패처럼 처리합니다.

## `78e5d25d7df6` — public 경로가 공통 상태를 실제로 사용

**중요도** B · **태그** `OUTPUT, REFACTOR`

이 커밋은 local `ft_write_count`를 제거하고 stack의 `t_printf ctx`를 초기화합니다.

```diff
-	int		count;
+	t_printf	ctx;
...
-	count = 0;
+	ft_printf_init(&ctx, 1);
...
-	if (ft_write_count(format, 1, &count) < 0)
-		count = -1;
+	if (ft_printf_putchar(&ctx, *format) < 0)
+		break ;
...
-	return (count);
+	if (ctx.error)
+		return (-1);
+	return (ctx.count);
```

이후 리터럴과 escaped percent도 다른 conversion과 같은 count/error 규칙을 사용합니다. `ft_printf_putchar`는 별도 정책을 갖지 않고 `ft_printf_write(ctx, &c, 1)`로 위임하므로, 한 바이트 출력도 공통 상태 머신 밖으로 빠져나가지 않습니다.

## `c627bd1f85bb` — 넓은 반환형을 먼저 검증해야 하는 이유

**중요도** A · **태그** `OUTPUT, RISK, DEBUG`

직전 코드는 다음 식에서 `written`을 먼저 `int`로 변환했습니다.

```c
ctx->count > INT_MAX - (int)written
```

`write`의 반환형은 `ssize_t`이고, 공통 API의 요청 길이는 `size_t`입니다. `written > INT_MAX`인 결과를 먼저 `int`로 줄이면 그 변환 결과에 의존한 뒤에 overflow를 검사하는 순서가 됩니다. 수정은 한 줄이지만 경계의 위치를 바로잡습니다.

```diff
-	if (ctx->count > INT_MAX - (int)written)
+	if (written > INT_MAX || ctx->count > INT_MAX - (int)written)
```

이제 `(int)written`은 `written <= INT_MAX`가 확인된 오른쪽 항에서만 평가됩니다. public 반환형이 `int`라는 제약을 시스템 호출의 넓은 결과형에 적용한 다음 count에 더합니다.

주의할 점은 overflow가 검사되는 시점입니다. `write`가 바이트를 받아들인 뒤 검사하므로 이미 발생한 외부 출력을 되돌릴 수는 없습니다. 이 방어는 잘못된 count를 반환하지 않게 하지만 출력 원자성을 만들지는 않습니다. 전체 결과 길이를 첫 write 전에 검사하는 문제는 별도의 preflight Thread에서 해결됩니다.

## `8a3ec50cb689` — POSIX 진행 규칙을 완성하는 핵심 수정

**중요도** S · **태그** `OUTPUT, CORE, RISK`

이 커밋은 세 가지를 한꺼번에 맞춥니다.

### 1. 한 번의 요청을 `SSIZE_MAX` 이하로 제한

```c
request = length;
if (request > (size_t)SSIZE_MAX)
	request = (size_t)SSIZE_MAX;
written = FT_PRINTF_SYSTEM_WRITE(ctx->fd, buffer, request);
```

`write`가 반환할 수 있는 양의 값은 `ssize_t`로 표현되어야 합니다. 따라서 남은 전체 길이가 더 크더라도 한 호출의 요청은 `SSIZE_MAX`를 넘기지 않고, 성공한 만큼 pointer와 remaining length를 전진시켜 다음 호출로 이어갑니다.

### 2. `EINTR`과 영구 실패를 구분

```c
if (written < 0 && errno == EINTR)
	continue ;
if (written <= 0)
{
	ctx->error = 1;
	return (-1);
}
```

신호로 중단된 호출은 바이트 진행이 없고 동일 상태에서 다시 시도할 수 있으므로 `count`, `buffer`, `length`, `error`를 바꾸지 않습니다. 반면 non-`EINTR` 음수와 0은 더 진행할 수 없는 결과로 취급합니다. 0을 성공으로 인정하면 `length`가 줄지 않아 무한 루프가 되므로 반드시 실패 상태로 전환해야 합니다.

### 3. 운영체제 타이밍에 의존하지 않는 seam

```c
#ifdef FT_PRINTF_TEST_WRITE
ssize_t	ft_printf_test_write(int fd, const void *buffer, size_t length);
# define FT_PRINTF_SYSTEM_WRITE ft_printf_test_write
#else
# define FT_PRINTF_SYSTEM_WRITE write
#endif
```

production에서는 그대로 `write`를 호출하고, test build에서만 scripted writer로 치환합니다. seam은 출력 정책을 소유하지 않습니다. 동일한 `ft_printf_write`가 실제 호출과 주입된 결과를 모두 해석하므로, 후속 테스트가 검증하는 대상은 별도의 mock 구현이 아니라 production 상태 전이입니다.

이 시점의 최종 loop는 다음과 같습니다.

```c
int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
{
	ssize_t	written;
	size_t	request;

	if (ctx->error)
		return (-1);
	while (length > 0)
	{
		request = length;
		if (request > (size_t)SSIZE_MAX)
			request = (size_t)SSIZE_MAX;
		written = FT_PRINTF_SYSTEM_WRITE(ctx->fd, buffer, request);
		if (written < 0 && errno == EINTR)
			continue ;
		if (written <= 0)
		{
			ctx->error = 1;
			return (-1);
		}
		if (written > INT_MAX
			|| ctx->count > INT_MAX - (int)written)
		{
			ctx->error = 1;
			return (-1);
		}
		ctx->count += (int)written;
		buffer += written;
		length -= (size_t)written;
	}
	return (0);
}
```

## `22e65c176b5d` — 성능 개선이 출력 규칙을 우회하지 않도록

**중요도** A · **태그** `OUTPUT, PERF`

기존 `ft_printf_putnchar`는 padding 한 글자마다 `ft_printf_putchar`를 호출했습니다. width 1000이면 거의 천 번의 `write`가 발생할 수 있습니다. 수정은 64바이트 stack buffer를 같은 문자로 채우고 bounded chunk로 보냅니다.

```c
char	buffer[64];
...
while (length > 0)
{
	chunk = length;
	if (chunk > (int)sizeof(buffer))
		chunk = (int)sizeof(buffer);
	if (ft_printf_write(ctx, buffer, (size_t)chunk) < 0)
		return (-1);
	length -= chunk;
}
```

중요한 점은 direct `write`로 최적화하지 않았다는 것입니다. 각 chunk는 계속 `ft_printf_write`를 통과하므로 부분 쓰기, `EINTR`, count 범위, sticky error의 의미는 변하지 않습니다. 최적화 대상은 호출 횟수이고, correctness 경계는 그대로 유지됩니다.

## `1223518652bd` — 상태 전이와 process policy를 나누어 검증

**중요도** A · **태그** `OUTPUT, TEST, RISK`

이 커밋은 test build에 scripted writer를 연결합니다. writer는 호출 순서대로 다음 결과를 만들 수 있습니다.

```c
typedef enum e_write_action
{
	WRITE_ALL,
	WRITE_PART,
	WRITE_EINTR,
	WRITE_EPIPE,
	WRITE_ZERO
}	t_write_action;
```

### 결정적으로 재현하는 경로

| 주입 순서 | production path에서 기대하는 상태 |
| --- | --- |
| `PART(2) → ALL` | 앞 2바이트를 count하고 pointer를 전진시킨 뒤 suffix 기록 |
| `EINTR → PART(3) → EINTR → ALL` | 중단에서는 상태 유지, 실제 progress만 count |
| `EPIPE` | `error = 1`, public `-1`, accepted byte 0 |
| `PART(3) → EPIPE` | public `-1`이지만 이미 받아들인 앞 3바이트는 남음 |
| `ZERO` | 무한 재시도하지 않고 오류로 종료 |

부분 실패 사례가 중요한 이유는 `ft_printf`가 transaction이 아니기 때문입니다. 커널이 이미 받은 `"par"`를 되돌릴 수 없으므로 테스트도 `-1`과 함께 정확히 3바이트가 남아 있음을 요구합니다.

padding 테스트는 `%1000d`에 대해 총 1000바이트, 최대 요청 64바이트, 총 17회의 writer 호출을 요구합니다. 999개의 공백은 64바이트 단위 호출 16회로 처리되고 마지막 digit 출력이 한 번 추가됩니다. 이 assertion은 “빠르다”는 추상적 주장보다 최적화 전략이 실제 공통 writer 호출 단위에 반영됐는지를 고정합니다.

### `SIGPIPE`에서 library가 소유하지 않는 것

별도 실제 pipe 테스트는 읽기 끝을 닫고 stdout을 write end에 연결한 뒤, 호출자가 설치한 `SIGPIPE` handler 아래에서 `ft_printf`를 실행합니다. 기대 조건은 다음 두 가지입니다.

1. handler가 한 번 호출되고 `write` 오류가 `-1`로 전파됩니다.
2. 호출 후에도 같은 handler가 설치되어 있습니다.

즉 library는 process-wide signal disposition을 임의로 `SIG_IGN`으로 바꾸거나 자체 handler로 덮어쓰지 않습니다. 기본 disposition을 유지한 프로세스라면 `SIGPIPE`로 종료될 수 있으며, 그것도 호출자가 선택한 process policy입니다. 이 Thread의 출력 계층은 살아남은 `write` 호출이 반환한 오류를 처리할 뿐 signal 정책을 소유하지 않습니다.

## 최종 불변 조건

| 사건 | buffer/remaining | `count` | `error` | public 결과 |
| --- | --- | ---: | ---: | --- |
| 양의 전체/부분 쓰기 | `written`만큼 전진 | 증가 | 유지 | 요청 완료까지 계속 |
| `EINTR` | 변화 없음 | 변화 없음 | 유지 | 재시도 |
| non-`EINTR` 음수 | 변화 없음 | 변화 없음 | 1 | -1 |
| 0바이트 반환 | 변화 없음 | 변화 없음 | 1 | -1 |
| 개별 결과 또는 누적 count가 `INT_MAX` 초과 | 이미 accepted된 바이트는 되돌릴 수 없음 | 더하지 않음 | 1 | -1 |
| 오류 뒤 후속 출력 요청 | 처리하지 않음 | 변화 없음 | 1 | -1 |

최종 호출 흐름은 다음과 같습니다.

```text
renderer / literal / padding
  → ft_printf_putchar 또는 ft_printf_putnchar
  → ft_printf_write
      → request를 SSIZE_MAX 이하로 제한
      → write 또는 test writer
      → EINTR: 그대로 재시도
      → positive: count/pointer/remaining 갱신
      → zero/permanent error/range failure: sticky error
  → public ft_printf가 error면 -1, 아니면 count 반환
```

## Thread 경계

- 포맷 전체 길이를 첫 출력 전에 계산해 `INT_MAX` 초과를 원자적으로 거부하는 문제는 whole-call preflight Thread에서 다룹니다.
- 숫자 접두사와 padding 순서는 numeric layout Thread의 문제이며, 이 Thread는 그 바이트들이 공통 writer를 통과한 뒤의 상태만 다룹니다.
- archive의 symbol/dependency와 sanitizer 실행은 verification Thread의 문제입니다.

검사 과정에서는 exact SHA의 diff와 source/test를 확인했으며, 이 환경에서 test binary를 직접 실행하지는 않았습니다.
