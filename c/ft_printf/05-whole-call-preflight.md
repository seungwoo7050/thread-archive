# Thread: 전체 호출 선검증 — field 오류 검사에서 첫 `write` 전 판정까지

## 개요

한 format field의 width나 precision overflow를 안전하게 검출하는 것과, `ft_printf` 호출 전체가 아무 출력도 하지 않은 채 실패하도록 만드는 것은 다른 문제입니다.

Single-pass 구현은 앞부분을 출력한 뒤 뒤쪽의 잘못된 field를 만날 수 있습니다.

```text
"prefix:%q"
  → "prefix:"를 이미 write
  → %q가 잘못됐음을 발견
  → -1을 반환해도 외부 출력은 남음
```

이 Thread는 local parser validation이 다음 two-pass 구조로 확장되는 과정을 다룹니다.

```text
1차 pass: copied va_list로 전체 format과 총 길이 측정
          문법·argument type progression·INT_MAX 범위 확인
          write 없음

2차 pass: original va_list로 실제 rendering
          1차 pass가 성공한 경우에만 시작
```

그 결과 **format/length 오류는 첫 write 전에 원자적으로 실패**합니다. 반면 실제 device의 `write` 실패는 이미 받아들여진 바이트를 되돌릴 수 없으므로 부분 출력이 남을 수 있습니다. Preflight는 이 두 오류 종류의 경계를 명확히 합니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | S | `ARCH, PARSER, CORE` | 한 field의 width/precision overflow를 연산 전에 검출하는 기반 |
| `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | A | `FORMAT, TEST, VERIFY` | stdout capture와 field-at-start 오류 검증 도입, late failure 한계 노출 |
| `2d773acc5bd6` | fix(format): 지원 문법과 전체 출력 크기 선검증 | S | `ARCH, VARARGS, ATOMIC` | `va_copy` measurement pass로 모든 field와 총 길이를 write 전에 검증 |
| `14059bd24f3e` | test(format): 잘못된 포맷의 무출력 실패 검증 | A | `ATOMIC, TEST, RISK` | late invalid field와 표현 불가능한 총 길이가 0바이트로 실패함을 고정 |

## `7984ddf2dd57` — 안전한 field parser는 필요하지만 충분하지 않음

**중요도** S · **태그** `ARCH, PARSER, CORE`

Parser는 decimal 누적 전에 다음 조건을 검사합니다.

```c
if (*value > (INT_MAX - digit) / 10)
	return (-1);
*value = *value * 10 + digit;
```

이 결정 덕분에 `%2147483648d`나 `%.2147483648d`는 overflow된 `int` state를 만들지 않습니다. `t_format`은 flags, width, precision, `has_precision`, specifier를 한 번 해석한 결과로 보존합니다.

그러나 이 exact SHA에서는 parser가 아직 public loop에 연결되지 않았고, 연결된 뒤에도 single-pass라면 local failure 위치가 external effect를 결정합니다. 첫 field가 잘못됐다면 0바이트로 실패하지만, 앞에 literal이나 정상 conversion이 있으면 이미 출력한 뒤 실패할 수 있습니다.

즉 이 커밋이 만든 invariant는 다음 하나입니다.

> 한 field의 decimal 값을 `int` 범위 밖으로 계산하지 않는다.

아직 다음 invariant는 없습니다.

> 호출 전체가 유효하고 총 결과가 `int`로 표현 가능하다는 사실을 첫 write 전에 안다.

## `1b8049e411bb` — 출력 capture가 보여준 것과 보여주지 못한 것

**중요도** A · **태그** `FORMAT, TEST, VERIFY`

이 커밋은 stdout을 pipe로 교체해 `ft_printf`가 실제로 기록한 raw bytes와 public return을 함께 수집합니다.

```c
typedef struct s_capture
{
	int	saved_stdout;
	int	pipe_fd[2];
}	t_capture;
```

`capture_begin`은 원래 stdout을 저장하고 pipe write end를 `STDOUT_FILENO`에 복제합니다. `capture_end`는 stdout을 복원한 뒤 read end를 EOF까지 읽습니다. Assertion은 세 값을 따로 비교합니다.

```c
if (actual_ret != expected_ret
	|| actual_len != expected_ret
	|| memcmp(expected, actual, (size_t)expected_ret) != 0)
	fail_test(...);
```

따라서 embedded NUL이 있는 `%c`도 C string 함수에 의존하지 않고 byte length와 memory를 비교할 수 있습니다. Portable semantics는 `snprintf`를 oracle로 쓰고, null string/pointer처럼 repository가 별도 표현을 택한 사례는 fixed output을 사용합니다.

Parser boundary test도 추가됩니다.

```c
expect_field_error(__LINE__, "%2147483648d");
expect_field_error(__LINE__, "%.2147483648d");
```

이 두 format은 잘못된 field가 첫 바이트에서 시작하므로 single-pass 구현도 write 전에 실패할 수 있습니다. 따라서 `actual_len == 0`이라는 결과만으로 “format 뒤쪽의 오류까지 미리 검사한다”는 architecture를 증명하지 못합니다. 다음과 같은 사례가 아직 없습니다.

```text
"prefix:%q"
"value:%d bad:%q"
"x%2147483647d"
```

이 한계가 preflight fix와 후속 regression의 정확한 필요 조건입니다.

## `2d773acc5bd6` — whole-call measurement pass 도입

**중요도** S · **태그** `ARCH, VARARGS, ATOMIC`

### 문제를 renderer에서 해결할 수 없는 이유

Unsupported specifier를 dispatcher에서 거부하거나 count overflow를 output writer에서 검출해도 시점이 늦습니다. Renderer에 도달했다는 것은 앞선 literal/conversion이 이미 write되었을 수 있다는 뜻입니다. No-output guarantee를 만들려면 모든 실패 가능한 format/length 판정을 external effect 이전으로 옮겨야 합니다.

이 커밋은 `src/ft_measure.c`를 추가해 실제 출력과 독립된 첫 pass를 만듭니다.

### 지원 문법과 총 길이의 공통 guard

```c
static int	ft_is_supported_specifier(char spec)
{
	return (spec == 'c' || spec == 's' || spec == 'd' || spec == 'i'
		|| spec == 'u' || spec == 'x' || spec == 'X' || spec == 'p'
		|| spec == '%');
}

static int	ft_add_length(size_t *total, size_t amount)
{
	if (amount > (size_t)INT_MAX
		|| *total > (size_t)INT_MAX - amount)
		return (-1);
	*total += amount;
	return (0);
}
```

`ft_add_length`는 두 곳에서 쓰입니다.

- 한 conversion의 content와 prefix를 합칠 때
- 전체 format의 누적 길이에 literal/conversion 길이를 더할 때

따라서 개별 width가 `INT_MAX` 이하여도 앞뒤 literal 또는 prefix와 합쳐 총 결과가 `INT_MAX`를 넘으면 실패합니다.

### Measurement가 rendering semantics를 다시 계산하는 방식

숫자는 shared layout의 길이 모델을 따라야 합니다.

```c
static int	ft_measure_numeric(t_format *fmt, int prefix_length,
		int digit_length, int is_zero, size_t *length)
{
	size_t	content_length;

	if (fmt->has_precision && fmt->precision == 0 && is_zero)
		digit_length = 0;
	content_length = (size_t)digit_length;
	if (fmt->has_precision && fmt->precision > digit_length)
		content_length = (size_t)fmt->precision;
	if (ft_add_length(&content_length, (size_t)prefix_length) < 0)
		return (-1);
	if ((size_t)fmt->width > content_length)
		content_length = (size_t)fmt->width;
	*length = content_length;
	return (0);
}
```

이 함수는 실제 padding 문자를 만들지 않고 최종 field 길이만 계산합니다. 다음 규칙이 renderer와 같아야 합니다.

- 값 0 + precision 0의 digit suppression
- precision이 digit보다 크면 content digit width를 precision으로 확장
- sign/alternate/pointer prefix 포함
- width가 content보다 크면 최종 길이는 width

문자열 측정도 `%s`의 bounded-read fix를 재현합니다.

```c
while ((!fmt->has_precision
		|| string_length < (size_t)fmt->precision)
	&& string[string_length])
{
	if (string_length == (size_t)INT_MAX)
		return (-1);
	string_length++;
}
```

정밀도가 있으면 그 범위 밖의 NUL을 찾지 않습니다. Measurement가 rendering보다 더 멀리 읽는다면 preflight 자체가 bounded-access invariant를 깨뜨리므로 두 pass 모두 같은 종료 조건을 가져야 합니다.

### 두 pass가 같은 promoted type을 소비

Measurement는 값을 출력하지 않더라도 argument sequence를 실제 rendering과 같은 방식으로 전진시켜야 합니다.

```c
if (fmt->spec == 'c')
	(void)va_arg(*args, int);
else if (fmt->spec == 's')
	va_arg(*args, char *);
else if (fmt->spec == 'd' || fmt->spec == 'i')
	va_arg(*args, int);
else if (fmt->spec == 'u' || fmt->spec == 'x' || fmt->spec == 'X')
	va_arg(*args, unsigned int);
else if (fmt->spec == 'p')
	va_arg(*args, void *);
```

한 field에서 잘못된 타입을 소비하면 이후 모든 argument 위치가 틀어집니다. 이 type map이 dispatcher와 호환되어야 두 pass가 같은 호출을 독립적으로 순회할 수 있습니다.

### 전체 format traversal

```c
int	ft_printf_measure(const char *format, va_list *args)
{
	t_format	fmt;
	size_t		conversion_length;
	size_t		total;

	total = 0;
	while (*format)
	{
		if (*format == '%' && *(format + 1) == '%')
		{
			if (ft_add_length(&total, 1) < 0)
				return (-1);
			format += 2;
		}
		else if (*format == '%')
		{
			format = ft_printf_parse(format + 1, &fmt);
			if (format == 0
				|| !ft_is_supported_specifier(fmt.spec)
				|| ft_measure_conversion(&fmt, args,
					&conversion_length) < 0
				|| ft_add_length(&total, conversion_length) < 0)
				return (-1);
		}
		else
		{
			if (ft_add_length(&total, 1) < 0)
				return (-1);
			format++;
		}
	}
	return ((int)total);
}
```

Trailing `%`는 parser가 `spec == '\0'`을 만들고 supported-specifier check에서 거부됩니다. `%q`도 같은 지점에서 거부됩니다. 이전 dispatcher의 literal fallback은 public call에서 preflight를 통과할 수 없으므로 도달 불가능한 방어 경로가 됩니다.

### `va_list` lifecycle

Public entry는 original list와 measurement copy를 분리합니다.

```c
if (format == 0)
	return (-1);
va_start(args, format);
va_copy(measure_args, args);
if (ft_printf_measure(format, &measure_args) < 0)
{
	va_end(measure_args);
	va_end(args);
	return (-1);
}
va_end(measure_args);
ft_printf_init(&ctx, 1);
/* original args로 rendering */
...
va_end(args);
```

Ownership은 다음과 같습니다.

| list | 생성 | 소비 | 종료 |
| --- | --- | --- | --- |
| `args` | `va_start` | measurement 성공 뒤 rendering | 모든 started path에서 `va_end(args)` |
| `measure_args` | `va_copy(args)` | first pass only | measurement 성공/실패 모두 `va_end(measure_args)` |

Measurement가 copy를 소비하므로 original list는 첫 renderer argument를 그대로 가리킵니다. 실패 시에는 두 list를 모두 종료하고 output context를 초기화하거나 사용하기 전에 반환합니다.

### 새 atomicity 경계

| 오류 종류 | 발견 시점 | 출력 가능성 |
| --- | --- | --- |
| null format | `va_start` 전 | 없음 |
| width/precision decimal overflow | measurement parser | 없음 |
| unsupported/trailing specifier | measurement supported check | 없음 |
| conversion/total length > `INT_MAX` | measurement length accumulation | 없음 |
| runtime `write`의 부분 성공 뒤 오류 | rendering/output writer | 이미 accepted된 prefix가 남을 수 있음 |

Preflight는 device I/O를 transaction으로 만들지 않습니다. Format과 size에 관해 미리 알 수 있는 실패만 외부 효과 앞으로 옮깁니다.

## `14059bd24f3e` — late failure와 total overflow를 0바이트 assertion으로 고정

**중요도** A · **태그** `ATOMIC, TEST, RISK`

새 macro는 return과 captured byte length를 함께 검사합니다.

```c
#define EXPECT_FORMAT_ERROR(FORMAT, ...) do { \
	/* stdout capture */ \
	actual_ret = ft_printf(FORMAT, ##__VA_ARGS__); \
	actual_len = capture_end(...); \
	if (actual_ret != -1 || actual_len != 0) \
		fail_test(__LINE__, "invalid format produced output"); \
} while (0)
```

추가된 사례는 서로 다른 preflight branch를 겨냥합니다.

| 사례 | 이전 single-pass에서 가능한 잘못된 결과 | preflight가 거부하는 이유 |
| --- | --- | --- |
| `"prefix:%2147483648d"` | `prefix:` 출력 후 parser 실패 | 뒤쪽 field decimal overflow |
| `"prefix:%q"` | prefix 또는 `%q` literal fallback 출력 | unsupported specifier |
| `"prefix:%"` | prefix와 trailing `%` 출력 | unterminated field (`spec == '\0'`) |
| `"value:%d bad:%q"` | 정상 argument와 literal 출력 후 실패 | later unsupported field |
| `"x%2147483647d"` | 앞 literal 뒤 최대 width 출력 시도 | 1 + `INT_MAX` total overflow |
| `"%2147483647dX"` | 거대한 field 뒤 literal `X`에서 total 초과 | `INT_MAX` + 1 |
| `"%+.2147483647d"` | precision과 sign prefix 합이 초과 | numeric content + prefix overflow |
| `"%#.2147483647x"` | precision과 `0x` prefix 합이 초과 | numeric content + alternate prefix overflow |

특히 `"value:%d bad:%q"`에서 measurement는 copied list로 첫 `%d` argument를 소비하지만, 오류 뒤 실제 rendering은 시작하지 않습니다. Original `args`는 output에 쓰이지 않은 채 종료됩니다.

이 테스트는 열거된 format/length 오류의 no-output behavior를 증명합니다. Invalid pointer argument, concurrent format mutation, 실제 device failure 같은 문제는 범위 밖입니다.

## 최종 실행 흐름

```text
ft_printf(format, ...)
  ├─ format == NULL → -1
  ├─ va_start(original)
  ├─ va_copy(measure)
  ├─ ft_printf_measure
  │    ├─ parser로 동일 grammar 소비
  │    ├─ supported spec 확인
  │    ├─ copied arguments를 정확한 promoted type으로 소비
  │    ├─ string/numeric/text 길이 계산
  │    └─ total <= INT_MAX 확인
  ├─ 실패: va_end(copy), va_end(original), write 없이 -1
  └─ 성공:
       va_end(copy)
       output context 초기화
       original arguments로 parse/dispatch/render
       va_end(original)
       runtime output 상태에 따라 count 또는 -1
```

## Thread 경계

- Parser의 field representation과 dispatch type map 자체는 format-fields Thread에서 설명합니다.
- 숫자와 문자열의 길이 semantics는 각각 numeric layout과 bounded-access Thread가 source of truth이며, measurement는 이를 동일하게 재현해야 합니다.
- 실제 `write`가 partial/EINTR/EPIPE를 반환한 뒤의 동작은 output-state Thread의 책임입니다.
- Late format error를 0바이트로 거부하는 것과 release artifact/sanitizer를 검증하는 것은 다른 verification layer입니다.

Exact SHA의 `ft_measure.c`, `ft_printf.c`, test diff를 검사했으며, 테스트는 이 환경에서 실행하지 않았습니다.
