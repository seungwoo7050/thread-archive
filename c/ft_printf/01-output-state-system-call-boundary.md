# Thread: 출력 상태는 부분 진행과 실패를 하나의 writer에서 관리한다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 01

## 개요

초기 출력 경로는 한 번의 `write`가 요청 길이를 모두 처리해야 성공으로 보았습니다. 이 Thread는 그 local helper가 descriptor·누적 count·sticky error를 공유하는 `t_printf` 상태로 바뀌고, positive short write와 `EINTR`을 진행 가능한 사건으로 처리하며, padding 최적화도 같은 writer 경계를 통과하도록 정리되는 과정을 다룹니다.

최종 writer는 이미 운영체제가 받아들인 바이트만 count하고, 아직 남은 suffix를 계속 처리합니다. 영구 오류나 0바이트 반환, public `int` count로 표현할 수 없는 결과가 발생하면 error state를 고정해 후속 출력을 거부합니다. Library는 이 과정에서 process-wide `SIGPIPE` disposition을 바꾸지 않습니다.

| 사건 | 상태 변화 | 호출 결과 |
| --- | --- | --- |
| 양의 전체·부분 쓰기 | buffer와 remaining을 전진시키고 count 증가 | 요청이 끝날 때까지 계속 |
| `EINTR` | buffer·remaining·count·error 유지 | 같은 요청 재시도 |
| non-`EINTR` 음수 또는 0바이트 | `error = 1` | `-1` |
| 개별 결과나 누적 count가 `INT_MAX`를 넘음 | 더 이상 count하지 않고 `error = 1` | `-1`; 이미 accepted된 바이트는 되돌릴 수 없음 |

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `1d6a5cee3041` | feat(core): 리터럴과 퍼센트 출력 구현 | B | `CORE, OUTPUT` | local write/count helper와 public entry의 최소 출력 경로 도입 |
| 2 | `3f7b0ab926d0` | feat(output): 출력 컨텍스트와 쓰기 API 추가 | S | `ARCH, OUTPUT, CORE` | FD·count·sticky error를 한 상태로 묶고 positive short write를 이어 처리 |
| 3 | `78e5d25d7df6` | refactor(core): 리터럴 출력을 컨텍스트 API로 이관 | B | `OUTPUT, REFACTOR` | public literal 경로를 공통 출력 상태에 연결 |
| 4 | `c627bd1f85bb` | fix(output): 쓰기 결과를 집계하기 전에 범위 검증 | A | `OUTPUT, RISK, DEBUG` | `ssize_t` 결과를 `int`로 줄이기 전에 표현 가능 여부 검사 |
| 5 | `8a3ec50cb689` | fix(output): 중단된 쓰기 재시도와 요청 크기 제한 | S | `OUTPUT, CORE, RISK` | `SSIZE_MAX` 요청 제한, `EINTR` 재시도, 결정적 writer seam 도입 |
| 6 | `22e65c176b5d` | perf(output): 반복 채움을 묶어서 출력 | A | `OUTPUT, PERF` | 반복 padding을 64바이트 chunk로 묶되 공통 오류·count 경계 유지 |
| 7 | `1223518652bd` | test(output): 쓰기 실패 시퀀스와 채움 전략 검증 | A | `OUTPUT, TEST, RISK` | 부분 진행·중단·0 진행·`EPIPE`·`SIGPIPE`·chunk 정책 검증 |

## 공통 출력 상태가 public 경계를 형성한다

## 1d6a5cee3041 — feat(core): 리터럴과 퍼센트 출력 구현
**중요도** `B` · **태그** `CORE, OUTPUT`

### 무엇을 만들었는가 (diff)

Public entry와 archive build가 처음 생기고, 리터럴과 `%%`를 한 바이트씩 기록하는 local helper가 추가됩니다.

```diff
+static int	ft_write_count(const char *buffer, int length, int *count)
+{
+	int	written;
+
+	if (length <= 0)
+		return (0);
+	if (*count > INT_MAX - length)
+		return (-1);
+	written = (int)write(1, buffer, (size_t)length);
+	if (written < 0 || written != length)
+		return (-1);
+	*count += written;
+	return (0);
+}
```

### 왜 가볍게 다루는가

이 시점에는 descriptor가 상수 `1`, count는 `ft_printf` local이고, error는 helper의 negative return으로만 표현됩니다. 요청 8바이트 중 3바이트가 기록된 positive short write도 `written != length` 때문에 즉시 실패하며, 남은 5바이트를 이어 처리할 상태가 없습니다.

이 커밋은 최소 public 경로를 만들지만 이후 모든 conversion이 공유할 output ownership이나 retry 정책은 아직 형성하지 않습니다.

### 관련 커밋

`3f7b0ab926d0`은 descriptor·count·error를 `t_printf`에 모으고 short write 뒤 suffix를 계속 기록합니다. `78e5d25d7df6`은 새 API를 실제 public entry에 연결합니다.

## 3f7b0ab926d0 — feat(output): 출력 컨텍스트와 쓰기 API 추가
**중요도** `S` · **태그** `ARCH, OUTPUT, CORE`

### 무엇을 만들었는가 (diff)

```diff
+typedef struct s_printf
+{
+	int	fd;
+	int	count;
+	int	error;
+}	t_printf;
```

`ft_printf_write`는 남은 길이가 0이 될 때까지 positive progress를 상태에 반영합니다.

```diff
+int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
+{
+	ssize_t	written;
+
+	if (ctx->error)
+		return (-1);
+	while (length > 0)
+	{
+		written = write(ctx->fd, buffer, length);
+		if (written <= 0)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		if (ctx->count > INT_MAX - (int)written)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		ctx->count += (int)written;
+		buffer += written;
+		length -= (size_t)written;
+	}
+	return (0);
+}
```

### 설계 결정은 무엇인가

한 출력 작업의 mutable state를 output API가 독점합니다. Positive result가 나오면 count, buffer pointer, remaining length가 함께 전진하고, 실패하면 `error`를 sticky하게 만들어 이후 writer 호출을 차단합니다. Renderer는 `write` 결과를 각자 해석하지 않고 이 API의 성공 여부만 확인하면 됩니다.

### 아직 다루지 않는 것은 무엇인가

이 exact SHA에서는 모듈만 추가되고 public `ft_printf`는 기존 local helper를 계속 사용합니다. 또한 `ssize_t` 결과를 `(int)`로 줄인 뒤 범위를 검사하고, `EINTR`을 영구 실패처럼 취급하며, 한 요청이 `SSIZE_MAX`를 넘는 상황도 제한하지 않습니다.

### 관련 커밋

`78e5d25d7df6`은 public path를 이 상태로 이관합니다. `c627bd1f85bb`과 `8a3ec50cb689`은 각각 count type boundary와 POSIX retry/request boundary를 보강합니다.

## 78e5d25d7df6 — refactor(core): 리터럴 출력을 컨텍스트 API로 이관
**중요도** `B` · **태그** `OUTPUT, REFACTOR`

### 무엇이 바뀌었는가 (diff)

```diff
-	int		count;
+	t_printf	ctx;
@@
-	count = 0;
+	ft_printf_init(&ctx, 1);
@@
-			if (ft_write_count(format, 1, &count) < 0)
-				count = -1;
+			if (ft_printf_putchar(&ctx, *format) < 0)
+				break ;
@@
-	return (count);
+	if (ctx.error)
+		return (-1);
+	return (ctx.count);
```

### 이 refactor가 확정하는 책임은 무엇인가

Local helper가 제거되고 literal과 escaped percent도 `ft_printf_putchar` → `ft_printf_write` 경로를 사용합니다. 한 바이트 출력도 별도 count/error 규칙을 갖지 않으므로 이후 conversion과 동일한 output state를 공유합니다.

### 관련 커밋

`3f7b0ab926d0`이 만든 abstraction이 이 커밋에서 public behavior를 실제로 소유하게 됩니다. 이후 `c627bd1f85bb`, `8a3ec50cb689`, `22e65c176b5d`의 수정과 최적화는 이 단일 경계에 적용됩니다.

## 시스템 호출의 타입·진행·비용 경계가 강화된다

## c627bd1f85bb — fix(output): 쓰기 결과를 집계하기 전에 범위 검증
**중요도** `A` · **태그** `OUTPUT, RISK, DEBUG`

### 무엇이 바뀌었는가 (diff)

```diff
-		if (ctx->count > INT_MAX - (int)written)
+		if (written > INT_MAX || ctx->count > INT_MAX - (int)written)
```

### 왜 이렇게 작은가

`write`는 `ssize_t`를 반환하지만 public count는 `int`입니다. 이전 식은 넓은 결과를 먼저 `(int)`로 줄인 뒤 overflow guard에 사용했습니다. 수정 후에는 `written > INT_MAX`가 먼저 평가되고, 그 조건이 거짓일 때만 오른쪽 항의 cast가 실행됩니다.

이 fix는 잘못된 count를 저장하지 않게 하지만, range check가 `write` 뒤에 있으므로 이미 운영체제가 받아들인 바이트를 되돌리지는 못합니다. 형식과 총 길이를 첫 출력 전에 판정하는 atomicity는 `05-whole-call-preflight.md`의 별도 문제입니다.

### 관련 커밋

`8a3ec50cb689`은 한 요청 자체를 `SSIZE_MAX` 이하로 제한하고 `EINTR`을 재시도해 같은 writer의 syscall contract를 완성합니다.

## 8a3ec50cb689 — fix(output): 중단된 쓰기 재시도와 요청 크기 제한
**중요도** `S` · **태그** `OUTPUT, CORE, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

- **문제**: 남은 `size_t` 길이가 `ssize_t`가 표현할 수 있는 요청보다 클 수 있고, 신호로 중단된 `write`도 영구 오류로 처리되었습니다.
- **원인**: Writer가 positive/zero/negative만 구분했고 request size와 `errno` 의미를 상태 전이에 반영하지 않았습니다.
- **결정**: 각 syscall 요청을 `SSIZE_MAX` 이하로 제한하고, `EINTR`은 상태를 바꾸지 않은 채 재시도하며, 0과 non-`EINTR` 음수만 sticky error로 전환했습니다.

### 무엇이 바뀌었는가 (diff)

```diff
+#include <errno.h>
@@
+	request = length;
+	if (request > (size_t)SSIZE_MAX)
+		request = (size_t)SSIZE_MAX;
-	written = write(ctx->fd, buffer, length);
+	written = FT_PRINTF_SYSTEM_WRITE(ctx->fd, buffer, request);
+	if (written < 0 && errno == EINTR)
+		continue ;
 	if (written <= 0)
 	{
 		ctx->error = 1;
 		return (-1);
 	}
```

Test build에서만 system call target을 바꿀 seam도 추가됩니다.

```diff
+#ifdef FT_PRINTF_TEST_WRITE
+ssize_t	ft_printf_test_write(int fd, const void *buffer, size_t length);
+# define FT_PRINTF_SYSTEM_WRITE ft_printf_test_write
+#else
+# define FT_PRINTF_SYSTEM_WRITE write
+#endif
```

Production branch는 raw `write`를 그대로 호출합니다. 결과를 해석하는 loop는 production과 test에서 동일하므로 seam은 별도 output policy를 만들지 않습니다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

Positive short write는 suffix 처리로 이어지고, `EINTR`은 count·pointer·remaining·error를 바꾸지 않습니다. 0바이트 반환은 progress가 없으므로 무한 loop를 피하기 위해 실패로 처리합니다. Non-`EINTR` 오류 이후에는 후속 출력이 거부됩니다.

Library는 여기서 `SIGPIPE` disposition을 설정하지 않습니다. Broken pipe에서 process가 살아남아 `write`가 `EPIPE`를 반환하는지는 caller가 선택한 signal policy에 달려 있습니다.

### 관련 커밋

`1223518652bd`은 이 seam을 사용해 partial, `EINTR`, zero, `EPIPE` 순서를 결정적으로 만들고, 실제 pipe를 사용해 caller의 `SIGPIPE` handler가 보존되는지도 별도로 확인합니다.

## 22e65c176b5d — perf(output): 반복 채움을 묶어서 출력
**중요도** `A` · **태그** `OUTPUT, PERF`

### 무엇이 바뀌었는가 (diff)

```diff
 int	ft_printf_putnchar(t_printf *ctx, char c, int length)
 {
+	char	buffer[64];
+	int		index;
+	int		chunk;
+
+	index = 0;
+	while (index < (int)sizeof(buffer))
+		buffer[index++] = c;
 	while (length > 0)
 	{
-		if (ft_printf_putchar(ctx, c) < 0)
+		chunk = length;
+		if (chunk > (int)sizeof(buffer))
+			chunk = (int)sizeof(buffer);
+		if (ft_printf_write(ctx, buffer, (size_t)chunk) < 0)
 			return (-1);
-		length--;
+		length -= chunk;
 	}
 	return (0);
 }
```

### 최적화가 correctness 경계를 왜 바꾸지 않는가

Padding 한 글자마다 writer를 호출하던 구현을 최대 64바이트 chunk로 묶습니다. Direct `write`를 새로 호출하지 않고 각 chunk를 계속 `ft_printf_write`에 전달하므로 partial progress, `EINTR`, count range, sticky error 규칙은 그대로 유지됩니다. 바뀌는 것은 syscall 호출 단위뿐입니다.

### 관련 커밋

`1223518652bd`은 `%1000d`에서 총 17회의 writer 호출과 최대 요청 64바이트를 요구해 이 chunk 전략이 문자별 호출로 되돌아가지 않도록 고정합니다.

## 상태 전이와 process policy를 서로 다른 테스트로 관찰한다

## 1223518652bd — test(output): 쓰기 실패 시퀀스와 채움 전략 검증
**중요도** `A` · **태그** `OUTPUT, TEST, RISK`

### 왜 다른 기법이 필요한가

실제 커널에서 “첫 호출은 2바이트만 성공, 다음 호출은 전부 성공”이나 “`EINTR` → 부분 성공 → `EINTR`” 순서를 안정적으로 만들기 어렵습니다. 이 commit은 `FT_PRINTF_TEST_WRITE` binary에 scripted writer를 연결해 syscall 결과의 순서를 테스트 입력으로 바꿉니다.

```diff
+typedef enum e_write_action
+{
+	WRITE_ALL,
+	WRITE_PART,
+	WRITE_EINTR,
+	WRITE_EPIPE,
+	WRITE_ZERO
+}	t_write_action;
```

### 어떤 경로를 검증하는가

```diff
+	reset_writer();
+	add_step(WRITE_PART, 2);
+	add_step(WRITE_ALL, 0);
+	expect_success("partial", 2, 7);
+
+	reset_writer();
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_PART, 3);
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_ALL, 0);
+	expect_success("interrupt", 4, 9);
```

Failure cases는 첫 `EPIPE`, partial 3바이트 뒤 `EPIPE`, 0바이트 반환을 구분합니다. Partial failure에서는 public return이 `-1`이어도 이미 accepted된 `"par"`가 남아야 합니다. Padding case는 `%1000d`가 총 1000바이트를 만들고 writer 호출 17회, 최대 요청 64바이트를 사용한다고 검사합니다.

Mock `EPIPE`만으로는 signal disposition을 볼 수 없으므로 normal suite에는 실제 broken pipe test도 추가됩니다. Caller가 설치한 handler가 한 번 호출되고, `ft_printf`가 `-1`을 반환하며, 호출 뒤에도 같은 handler가 설치되어 있어야 합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 syscall sequence에서 production writer의 pointer·remaining·count·error 전이가 맞고, padding이 64바이트 bounded chunk를 사용하며, library가 caller의 `SIGPIPE` handler를 변경하지 않음을 증명합니다. Default `SIGPIPE` disposition에서 process가 생존한다는 보장은 하지 않으며, 모든 가능한 errno와 커널 타이밍을 포괄하지도 않습니다.

### 관련 커밋

이 test는 `8a3ec50cb689`의 request/retry 정책과 `22e65c176b5d`의 chunk 전략을 함께 고정합니다. Public formatting bytes를 폭넓게 비교하는 harness 자체는 `06-runtime-artifact-verification.md`에서 별도 검증 계층으로 설명합니다.

## 이 Thread의 경계

- 포맷 전체의 문법과 총 길이를 첫 출력 전에 거부하는 문제는 `05-whole-call-preflight.md`가 다룹니다.
- 접두사·정밀도·너비의 바이트 순서는 `03-shared-numeric-layout.md`가 다룹니다.
- `%s` 정밀도의 읽기 범위는 `04-string-precision-bounded-access.md`가 다룹니다.
- Archive member·symbol·dependency와 sanitizer target은 `06-runtime-artifact-verification.md`의 검증 문제입니다.

> 검토 범위: `1d6a5cee3041`, `3f7b0ab926d0`, `78e5d25d7df6`, `c627bd1f85bb`, `8a3ec50cb689`, `22e65c176b5d`, `1223518652bd`의 exact diff와 해당 시점의 `src/ft_printf.c`, `src/ft_output.c`, internal header, output fault test, `SIGPIPE` test를 확인했습니다. Build와 test binary는 이 환경에서 실행하지 않았습니다.
