# Thread: 전체 호출은 형식과 총 길이를 첫 `write` 전에 판정한다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 05

## 개요

한 field의 width나 precision이 `int`를 넘는지 안전하게 검사하는 것만으로는 호출 전체의 무출력 실패를 보장할 수 없습니다. Single-pass formatter는 앞부분을 이미 출력한 뒤 뒤쪽의 unsupported specifier나 총 길이 초과를 발견할 수 있기 때문입니다. 이 Thread는 field-local parser guard와 초기 capture test를 바탕으로, copied `va_list`가 전체 format과 결과 길이를 먼저 측정하고 original `va_list`는 그 검사가 성공한 뒤에만 rendering하도록 바뀌는 과정을 다룹니다.

Preflight가 만드는 atomicity는 형식과 길이에 한정됩니다. 미리 판정할 수 있는 오류는 0바이트와 `-1`로 끝나지만, 실제 `write`가 일부 바이트를 받아들인 뒤 실패하면 그 외부 효과는 되돌릴 수 없습니다.

| 오류 종류 | 발견 시점 | 출력 결과 |
| --- | --- | --- |
| null format, field decimal overflow, unsupported/trailing specifier, total `INT_MAX` 초과 | measurement pass | `-1`, 출력 0바이트 |
| rendering 중 partial write 뒤 영구 오류 | output pass | `-1`, 이미 받아들인 바이트는 남을 수 있음 |

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | S | `ARCH, PARSER, CORE` | 한 field의 width·precision overflow를 연산 전에 검출하는 기반 |
| 2 | `1b8049e411bb` | test(printf): 기본 변환과 포맷 경계 검증 | A | `FORMAT, TEST, VERIFY` | stdout capture와 field-at-start 오류 검증을 도입하지만 late failure는 아직 구별하지 못함 |
| 3 | `2d773acc5bd6` | fix(format): 지원 문법과 전체 출력 크기 선검증 | S | `ARCH, VARARGS, ATOMIC` | `va_copy` measurement pass로 모든 field와 총 길이를 첫 출력 전에 판정 |
| 4 | `14059bd24f3e` | test(format): 잘못된 포맷의 무출력 실패 검증 | A | `ATOMIC, TEST, RISK` | 뒤쪽 형식 오류와 표현 불가능한 총 길이가 0바이트로 실패함을 고정 |

## 7984ddf2dd57 — feat(parser): 포맷 필드 모델과 해석기 추가
**중요도** `S` · **태그** `ARCH, PARSER, CORE`

### 무엇을 만들었는가 (diff)

Parser는 width와 precision의 10진 누적을 수행하기 전에 다음 digit을 더해도 `INT_MAX` 안에 있는지 검사합니다.

```diff
+static int	ft_parse_decimal(const char **format, int *value)
+{
+	int	digit;
+
+	while (ft_is_digit(**format))
+	{
+		digit = **format - '0';
+		if (*value > (INT_MAX - digit) / 10)
+			return (-1);
+		*value = *value * 10 + digit;
+		(*format)++;
+	}
+	return (0);
+}
```

이 guard는 overflow된 `int`를 만든 뒤 되돌리지 않고, 연산 자체가 안전한 범위에서만 state를 갱신합니다. 같은 helper가 width와 precision에 재사용됩니다.

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

이 커밋이 확립한 단위는 **한 field**입니다. `t_format`에 flags, width, precision, `has_precision`, specifier를 보존할 수 있지만, exact SHA에서는 parser module이 아직 public output loop에 연결되지 않았습니다. 연결된 뒤에도 single-pass라면 앞의 literal이나 정상 conversion이 이미 출력된 후 뒤쪽 field 오류를 발견할 수 있습니다.

따라서 이 시점에는 “한 field의 숫자를 overflow 없이 해석한다”는 기반만 있고, “호출 전체가 유효하며 총 결과가 `int`로 표현 가능함을 첫 출력 전에 안다”는 보장은 없습니다.

### 관련 커밋

`2d773acc5bd6`은 이 parser를 measurement와 rendering 양쪽에서 재사용해 field-local guard를 whole-call preflight로 확장합니다. `1b8049e411bb`의 초기 오류 test는 이 확장이 왜 아직 증명되지 않았는지를 보여주는 비교 기준입니다.

## 1b8049e411bb — test(printf): 기본 변환과 포맷 경계 검증
**중요도** `A` · **태그** `FORMAT, TEST, VERIFY`

### 무엇을 검증하는가

이 커밋은 stdout을 pipe로 capture해 실제 출력 바이트 수, 바이트 내용, public return을 함께 비교하는 harness를 만듭니다.

```diff
+static void	check_case(int line, const char *format, const char *expected,
+		int expected_ret, const char *actual, ssize_t actual_len,
+		int actual_ret)
+{
+	if (actual_ret != expected_ret || actual_len != expected_ret
+		|| memcmp(expected, actual, (size_t)expected_ret) != 0)
+	{
+		/* mismatch diagnostic 출력 생략 */
+		fail_test(line, "ft_printf output mismatch");
+	}
+}
```

Parser boundary에는 잘못된 field가 format 첫 위치에 있는 두 사례가 추가됩니다.

```diff
+static void	run_parser_boundary_cases(void)
+{
+	expect_field_error(__LINE__, "%2147483648d");
+	expect_field_error(__LINE__, "%.2147483648d");
+}
```

### 이 테스트가 증명하는 것 / 증명하지 않는 것

두 사례에서 return이 `-1`이고 capture 길이가 0임을 확인하므로 field decimal overflow가 public path에서 거부된다는 증거를 제공합니다. 그러나 오류가 첫 field에 있으므로 single-pass 구현도 아무것도 출력하기 전에 실패할 수 있습니다. `"prefix:%q"`나 `"value:%d bad:%q"`처럼 정상 prefix 뒤에 오류가 있는 format이 없어서, 전체 format을 선검증한다는 architecture까지는 증명하지 않습니다.

### 관련 커밋

`14059bd24f3e`은 같은 capture 기반에 late invalid field와 총 길이 초과 사례를 추가해 `2d773acc5bd6`의 no-output boundary를 직접 검증합니다. `1b8049e411bb`은 그보다 앞선 functional baseline입니다.

## 2d773acc5bd6 — fix(format): 지원 문법과 전체 출력 크기 선검증
**중요도** `S` · **태그** `ARCH, VARARGS, ATOMIC`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

- **문제**: dispatcher나 output writer에서 오류를 발견하면 앞선 literal·conversion이 이미 외부로 기록됐을 수 있었습니다.
- **원인**: 형식 검증, argument traversal, 결과 길이 계산이 rendering과 같은 single pass 안에 있었습니다.
- **결정**: copied `va_list`로 전체 format을 측정하고 지원 문법과 총 길이를 검증한 뒤, 성공한 호출만 original `va_list`로 실제 rendering을 시작하게 했습니다.

### public entry가 두 개의 `va_list`를 어떻게 분리하는가 (diff)

```diff
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
+	va_list	measure_args;
 	t_printf	ctx;
 	t_format	fmt;
 
 	if (format == 0)
 		return (-1);
 	va_start(args, format);
+	va_copy(measure_args, args);
+	if (ft_printf_measure(format, &measure_args) < 0)
+	{
+		va_end(measure_args);
+		va_end(args);
+		return (-1);
+	}
+	va_end(measure_args);
 	ft_printf_init(&ctx, 1);
 	/* original args로 기존 rendering을 수행 */
```

`measure_args`만 first pass에서 소비되므로 `args`는 첫 conversion argument를 그대로 가리킵니다. Measurement 성공과 실패 모두 copied list를 `va_end`하고, started original list도 모든 return path에서 종료합니다. Output context는 measurement가 성공한 뒤에야 초기화됩니다.

### measurement가 어떤 오류를 첫 `write` 앞으로 옮기는가

지원 specifier와 길이 누적은 별도 guard를 사용합니다.

```diff
+static int	ft_is_supported_specifier(char spec)
+{
+	return (spec == 'c' || spec == 's' || spec == 'd' || spec == 'i'
+		|| spec == 'u' || spec == 'x' || spec == 'X' || spec == 'p'
+		|| spec == '%');
+}
+
+static int	ft_add_length(size_t *total, size_t amount)
+{
+	if (amount > (size_t)INT_MAX
+		|| *total > (size_t)INT_MAX - amount)
+		return (-1);
+	*total += amount;
+	return (0);
+}
```

전체 traversal에서는 parser failure, unsupported/trailing specifier, conversion 길이 오류, total overflow 가운데 하나라도 발생하면 곧바로 `-1`을 반환합니다.

```diff
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0 || !ft_is_supported_specifier(fmt.spec)
+				|| ft_measure_conversion(&fmt, args,
+					&conversion_length) < 0
+				|| ft_add_length(&total, conversion_length) < 0)
+				return (-1);
+		}
```

### measurement와 rendering이 같은 의미를 소비해야 하는 이유

First pass는 바이트를 만들지 않지만 conversion마다 rendering과 호환되는 promoted type으로 `va_arg`를 소비해야 합니다. `%c`, `%d`/`%i`는 `int`, `%u`/`%x`/`%X`는 `unsigned int`, `%s`는 `char *`, `%p`는 `void *`입니다. 하나라도 다른 타입이나 개수를 소비하면 뒤쪽 argument 위치가 달라집니다.

길이 규칙도 동일해야 합니다. 숫자는 value 0 + precision 0의 digit suppression, sign/alternate/pointer prefix, precision, width를 반영합니다. 문자열은 정밀도 범위 안에서만 NUL을 찾습니다.

```diff
+static int	ft_measure_string(t_format *fmt, const char *string,
+		size_t *length)
+{
+	/* null mapping 생략 */
+	string_length = 0;
+	while ((!fmt->has_precision
+			|| string_length < (size_t)fmt->precision)
+		&& string[string_length])
+		string_length++;
+	/* width 반영 */
+}
```

Measurement가 renderer보다 더 멀리 문자열을 읽거나 다른 prefix 길이를 계산하면 preflight가 성공해도 두 번째 pass와 결과가 달라질 수 있습니다. 이 커밋은 두 구현이 같은 normalized `t_format`과 type map을 따르도록 구성합니다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

Null format, field decimal overflow, unsupported 또는 trailing specifier, conversion/total 결과가 `INT_MAX`를 넘는 경우에는 output context를 시작하기 전에 `-1`을 반환합니다. 반면 measurement 이후 실제 `write`가 부분 성공한 뒤 실패하는 상황은 여전히 partial external output을 남길 수 있습니다. Preflight는 device I/O를 transaction으로 만들지 않습니다.

### 관련 커밋

`14059bd24f3e`은 정상 prefix 뒤의 오류와 prefix를 포함한 총 길이 초과를 capture해 이 commit의 whole-call atomicity를 고정합니다. Parser와 typed dispatch의 원래 책임은 `02-format-fields-typed-dispatch.md`, 숫자·문자열 길이 규칙은 각각 `03`과 `04` 문서가 설명합니다.

## 14059bd24f3e — test(format): 잘못된 포맷의 무출력 실패 검증
**중요도** `A` · **태그** `ATOMIC, TEST, RISK`

### 무엇을 검증하는가

새 macro는 형식 오류가 `-1`뿐 아니라 정확히 0바이트를 남기는지 함께 검사합니다.

```diff
+#define EXPECT_FORMAT_ERROR(FORMAT, ...) do { \
+	char		actual[16]; \
+	t_capture	capture; \
+	int			actual_ret; \
+	ssize_t		actual_len; \
+	capture_begin(&capture, __LINE__); \
+	actual_ret = ft_printf(FORMAT, ##__VA_ARGS__); \
+	actual_len = capture_end(&capture, actual, sizeof(actual), __LINE__); \
+	if (actual_ret != -1 || actual_len != 0) \
+		fail_test(__LINE__, "invalid format produced output"); \
+} while (0)
```

추가 사례는 서로 다른 measurement branch를 겨냥합니다.

```diff
+	EXPECT_FORMAT_ERROR("prefix:%2147483648d", 1);
+	EXPECT_FORMAT_ERROR("prefix:%q", 1);
+	EXPECT_FORMAT_ERROR("prefix:%");
+	EXPECT_FORMAT_ERROR("value:%d bad:%q", 7, 1);
+	EXPECT_FORMAT_ERROR("x%2147483647d", 1);
+	EXPECT_FORMAT_ERROR("%2147483647dX", 1);
+	EXPECT_FORMAT_ERROR("%+.2147483647d", 1);
+	EXPECT_FORMAT_ERROR("%#.2147483647x", 1u);
```

앞의 네 사례는 late parser/specifier 오류를, 뒤의 네 사례는 literal·sign·alternate prefix까지 더한 총 결과가 `INT_MAX`를 넘는 경계를 확인합니다. `"value:%d bad:%q"`에서는 copied list가 `%d` argument를 소비한 뒤 오류를 발견하지만 original list로 rendering은 시작하지 않습니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 late format 오류와 총 길이 초과가 return `-1`, captured length 0으로 끝난다는 것을 증명합니다. 잘못된 pointer argument, 동시에 변경되는 format storage, 실제 output device 오류의 partial effect는 이 테스트의 범위가 아닙니다.

### 관련 커밋

이 회귀 suite는 `2d773acc5bd6`이 만든 two-pass 경계를 고정하고, `1b8049e411bb`의 “오류가 첫 field에만 있는 테스트”가 증명하지 못했던 late-failure 차이를 보완합니다.

## 이 Thread의 경계

- `t_format`의 grammar와 specifier별 promoted type 선택은 `02-format-fields-typed-dispatch.md`가 다룹니다.
- 숫자와 문자열의 effective length 규칙은 `03-shared-numeric-layout.md`와 `04-string-precision-bounded-access.md`가 각각 다룹니다.
- Partial write, `EINTR`, `EPIPE` 이후의 output state는 `01-output-state-system-call-boundary.md`가 다룹니다.
- Archive shape와 sanitizer 실행은 `06-runtime-artifact-verification.md`의 검증 경계입니다.

> 검토 범위: `7984ddf2dd57`, `1b8049e411bb`, `2d773acc5bd6`, `14059bd24f3e`의 exact diff와 해당 시점의 `src/ft_parse.c`, `src/ft_measure.c`, `src/ft_printf.c`, `tests/test_ft_printf.c`를 확인했습니다. 테스트 binary는 이 환경에서 실행하지 않았습니다.
