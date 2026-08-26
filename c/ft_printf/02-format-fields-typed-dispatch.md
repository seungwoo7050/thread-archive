# Thread: 포맷 필드는 한 번 정규화되고 가변 인자는 한 경계에서 타입별로 소비된다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 02

## 개요

초기 `ft_printf`는 format cursor에서 리터럴과 `%%`만 직접 구분했습니다. Flags, width, precision, specifier가 추가되면 main loop나 각 renderer가 raw format을 반복 해석할 수 있고, variadic argument를 어느 타입으로 소비할지도 여러 곳에 흩어질 수 있습니다. 이 Thread는 한 field를 `t_format`으로 정규화하는 parser와, specifier별 promoted type을 정확히 한 번 소비하는 dispatcher의 책임을 형성합니다.

Parser는 raw bytes와 cursor 이동을 소유하고 renderer는 정규화된 field와 typed value만 받습니다. 이 분리는 이후 measurement pass가 같은 grammar와 type map을 독립적으로 재현할 수 있는 기준도 제공합니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | S | `ARCH, PARSER, CORE` | flags·width·precision·specifier를 보존하는 normalized field와 overflow-safe parser 도입 |
| 2 | `9e6d785628f3` | feat(core): 포맷 필드 해석을 출력 루프에 연결 | B | `PARSER, INTEGRATION` | parser의 next cursor와 failure를 main traversal에 연결 |
| 3 | `03c3e6e09fa1` | feat(text): 문자·문자열·퍼센트 변환 추가 | A | `ARCH, FORMAT, VARARGS` | `va_arg` 타입 선택을 소유하는 dispatcher와 text renderer 도입 |
| 4 | `95d6613a1c72` | feat(decimal): 부호 있는·없는 10진수 출력 추가 | B | `FORMAT, VARARGS` | signed와 unsigned decimal의 서로 다른 argument type을 같은 dispatch boundary에 추가 |
| 5 | `93c883070a1b` | feat(hex): 16진수와 포인터 출력 추가 | B | `FORMAT, VARARGS` | hex와 pointer의 정확한 argument type과 renderer 추가 |
| 6 | `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | A | `PARSER, FORMAT, LAYOUT` | 정적 flag 충돌을 parser에서 줄이고 값에 따른 prefix 선택은 renderer에 유지 |

## 포맷 표현과 cursor 경계가 먼저 형성된다

## 7984ddf2dd57 — feat(parser): 포맷 필드 모델과 해석기 추가
**중요도** `S` · **태그** `ARCH, PARSER, CORE`

### 무엇을 만들었는가 (diff)

한 field의 상태를 raw cursor 대신 구조체로 보존합니다.

```diff
+typedef struct s_format
+{
+	int		flags;
+	int		width;
+	int		precision;
+	int		has_precision;
+	char	spec;
+}	t_format;
```

`has_precision`은 precision 생략과 명시적인 `.0`을 구분합니다. 숫자 값만 보면 둘 다 `precision == 0`이므로 별도 상태가 필요합니다.

Parser는 flags → width → optional precision → specifier 순서로 cursor를 이동합니다.

```diff
+const char	*ft_printf_parse(const char *format, t_format *fmt)
+{
+	int	flag;
+
+	ft_printf_init_format(fmt);
+	flag = ft_flag_value(*format);
+	while (flag)
+	{
+		fmt->flags |= flag;
+		format++;
+		flag = ft_flag_value(*format);
+	}
+	if (ft_parse_decimal(&format, &fmt->width) < 0)
+		return (0);
+	if (*format == '.')
+	{
+		fmt->has_precision = 1;
+		format++;
+		if (ft_parse_decimal(&format, &fmt->precision) < 0)
+			return (0);
+	}
+	fmt->spec = *format;
+	if (*format)
+		format++;
+	return (format);
+}
```

Width와 precision은 `value * 10 + digit` 전에 범위를 검사합니다.

```diff
+	if (*value > (INT_MAX - digit) / 10)
+		return (-1);
+	*value = *value * 10 + digit;
```

### 설계 결정은 무엇인가

Parser가 성공하면 반환 pointer는 field 뒤의 첫 unread byte를 가리키고, `t_format`은 그 field의 normalized representation이 됩니다. Repeated flag는 bitwise OR로 같은 state에 수렴하며, renderer는 raw bytes를 다시 읽지 않아도 됩니다.

이 exact SHA에서는 parser source와 representation만 추가되고 public `ft_printf`는 아직 호출하지 않습니다. Supported specifier 판정과 flag conflict normalization도 아직 없습니다.

### 관련 커밋

`9e6d785628f3`은 parser의 반환 pointer를 main cursor로 채택해 traversal 책임을 실제 public path에 연결합니다. `2d773acc5bd6`은 훗날 같은 parser를 measurement pass에도 사용합니다.

## 9e6d785628f3 — feat(core): 포맷 필드 해석을 출력 루프에 연결
**중요도** `B` · **태그** `PARSER, INTEGRATION`

### 무엇이 바뀌었는가 (diff)

```diff
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
 	t_printf	ctx;
+	t_format	fmt;
@@
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0)
+			{
+				ctx.error = 1;
+				break ;
+			}
+			if (ft_printf_putchar(&ctx, '%') < 0)
+				break ;
+			if (fmt.spec != '\0' && fmt.spec != '%'
+				&& ft_printf_putchar(&ctx, fmt.spec) < 0)
+				break ;
+		}
```

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

Parser가 field 전체를 소비하고 main loop가 반환 pointer를 그대로 다음 cursor로 사용한다는 책임이 확정됩니다. Renderer나 dispatcher는 format pointer를 이동시키지 않습니다.

이 단계의 출력은 integration을 위한 placeholder입니다. Parsed flags·width·precision은 사용하지 않고 `%`와 specifier를 다시 출력하며, unknown specifier와 trailing `%`도 아직 public error로 분류하지 않습니다.

### 관련 커밋

`03c3e6e09fa1`은 placeholder 출력을 typed dispatcher 호출로 교체합니다. `2d773acc5bd6`은 이 단계에 남아 있는 unknown/trailing fallback이 public path에 도달하지 못하도록 whole-call supported-specifier check를 추가합니다.

## 가변 인자 소비와 conversion routing이 한곳으로 모인다

## 03c3e6e09fa1 — feat(text): 문자·문자열·퍼센트 변환 추가
**중요도** `A` · **태그** `ARCH, FORMAT, VARARGS`

### 무엇을 만들었는가 (diff)

Main loop의 placeholder branch가 dispatcher 호출로 바뀌고, dispatcher가 specifier별 `va_arg` 타입을 선택합니다.

```diff
+int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
+{
+	if (fmt->spec == 'c')
+		return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
+	if (fmt->spec == 's')
+		return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
+	if (fmt->spec == '%')
+		return (ft_printf_print_percent(ctx, fmt));
+	if (ft_printf_putchar(ctx, '%') < 0)
+		return (-1);
+	if (fmt->spec)
+		return (ft_printf_putchar(ctx, fmt->spec));
+	return (0);
+}
```

```diff
-			if (ft_printf_putchar(&ctx, '%') < 0)
-				break ;
-			if (fmt.spec != '\0' && fmt.spec != '%'
-				&& ft_printf_putchar(&ctx, fmt.spec) < 0)
+			if (ft_printf_dispatch(&ctx, &fmt, &args) < 0)
 				break ;
```

### 왜 dispatcher가 `va_arg`를 소유하는가

`%c` 인자는 default argument promotion 때문에 `int`로 소비해야 하고, `%s`는 `char *`, `%%`는 argument를 소비하지 않습니다. Renderer가 `va_list` 대신 typed value를 받으면 argument 개수와 타입의 변화는 dispatcher 한곳에서 확인할 수 있고, renderer는 표시 규칙에만 집중합니다.

이 exact SHA의 text renderer는 width와 precision을 아직 사용하지 않습니다. Unknown specifier는 `%`와 해당 문자를 그대로 출력하는 fallback으로 남아 있습니다.

### 관련 커밋

`95d6613a1c72`와 `93c883070a1b`은 같은 boundary에 decimal, hex, pointer type을 추가합니다. `2d773acc5bd6`의 measurement pass는 이 dispatcher와 호환되는 type consumption을 재현해야 original과 copied `va_list`가 같은 field sequence를 순회할 수 있습니다.

## 95d6613a1c72 — feat(decimal): 부호 있는·없는 10진수 출력 추가
**중요도** `B` · **태그** `FORMAT, VARARGS`

### 무엇이 바뀌었는가 (diff)

```diff
+	if (fmt->spec == 'd' || fmt->spec == 'i')
+		return (ft_printf_print_signed(ctx, fmt, va_arg(*args, int)));
+	if (fmt->spec == 'u')
+		return (ft_printf_print_unsigned(ctx, fmt,
+				va_arg(*args, unsigned int)));
```

### 왜 가볍게 다루는가

이 Thread에서 핵심은 decimal digit algorithm이 아니라 variadic type map의 확장입니다. `%d`/`%i`와 `%u`가 모두 10진 문자열을 만들더라도 전자는 `int`, 후자는 `unsigned int`로 소비해야 합니다. Width·precision·prefix 배치는 아직 conversion 내부에 없거나 후속 작업으로 남아 있습니다.

### 관련 커밋

`93c883070a1b`은 동일한 dispatcher에 hex와 pointer type을 추가합니다. Decimal의 접두사와 field 배치는 `03-shared-numeric-layout.md`에서 별도 문제로 다룹니다.

## 93c883070a1b — feat(hex): 16진수와 포인터 출력 추가
**중요도** `B` · **태그** `FORMAT, VARARGS`

### 무엇이 바뀌었는가 (diff)

```diff
+	if (fmt->spec == 'x' || fmt->spec == 'X')
+		return (ft_printf_print_hex(ctx, fmt,
+				va_arg(*args, unsigned int)));
+	if (fmt->spec == 'p')
+		return (ft_printf_print_pointer(ctx, fmt, va_arg(*args, void *)));
```

### 왜 가볍게 다루는가

`x`와 `X`는 `unsigned int`, `p`는 `void *`로 소비합니다. Pointer renderer가 내부에서 `uintptr_t`를 거쳐 숫자 표현으로 바꾸더라도 variadic boundary에서는 pointer type으로 꺼내야 합니다. 이 커밋은 새로운 dispatch case를 더하지만 parser와 cursor contract는 바꾸지 않습니다.

### 관련 커밋

`c5f627099ad9`은 dispatcher 뒤 renderer가 받는 `t_format`을 정규화해 prefix/layout 조합을 줄입니다. Hex와 pointer의 실제 field 배치는 `03-shared-numeric-layout.md`에서 이어집니다.

## parser와 renderer 사이의 정규화 책임이 분리된다

## c5f627099ad9 — feat(flags): 숫자 플래그 우선순위 정규화
**중요도** `A` · **태그** `PARSER, FORMAT, LAYOUT`

### 무엇이 바뀌었는가 (diff)

Parser는 raw flags를 반환하기 전에 정적인 충돌을 줄입니다.

```diff
+static void	ft_normalize_flags(t_format *fmt)
+{
+	if (fmt->flags & FT_FLAG_LEFT)
+		fmt->flags &= ~FT_FLAG_ZERO;
+	if (fmt->flags & FT_FLAG_PLUS)
+		fmt->flags &= ~FT_FLAG_SPACE;
+}
@@
 	fmt->spec = *format;
 	if (*format)
 		format++;
+	ft_normalize_flags(fmt);
 	return (format);
```

값과 specifier를 알아야 하는 prefix 선택은 renderer에 남습니다.

```diff
+	prefix = "";
+	if (fmt->flags & FT_FLAG_PLUS)
+		prefix = "+";
+	else if (fmt->flags & FT_FLAG_SPACE)
+		prefix = " ";
```

```diff
+	prefix = "";
+	if ((fmt->flags & FT_FLAG_HASH) && number != 0)
+	{
+		if (fmt->spec == 'X')
+			prefix = "0X";
+		else
+			prefix = "0x";
+	}
```

### 책임을 왜 이렇게 나누는가

LEFT가 있으면 ZERO를 제거하고 PLUS가 있으면 SPACE를 제거하는 규칙은 raw field만으로 결정할 수 있으므로 parser가 canonical state를 만듭니다. 음수 여부, 값 0 여부, `x`/`X` 대소문자처럼 runtime value가 필요한 선택은 renderer가 담당합니다.

숫자 precision이 명시되면 field-level ZERO를 비활성화하는 규칙은 parser에 들어가지 않습니다. 이는 flag끼리의 정적 충돌이 아니라 `has_precision`과 layout의 결합이므로 numeric layout에서 판정합니다.

### 관련 커밋

`1fa064ca9d79`와 `177c8d03b353`은 이 normalized flags를 사용해 숫자 배치 규칙을 구성하고 통합합니다. `7984ddf2dd57`이 만든 parser representation은 그대로 유지됩니다.

## 이 Thread의 경계

- 접두사, precision zero, field padding의 실제 출력 순서는 `03-shared-numeric-layout.md`가 다룹니다.
- `%s` 정밀도가 메모리 접근 범위까지 제한하는 문제는 `04-string-precision-bounded-access.md`가 다룹니다.
- Unsupported/trailing specifier와 총 길이를 first write 전에 거부하는 문제는 `05-whole-call-preflight.md`가 다룹니다.
- 출력 바이트의 부분 진행과 sticky error는 `01-output-state-system-call-boundary.md`가 다룹니다.

> 검토 범위: `7984ddf2dd57`, `9e6d785628f3`, `03c3e6e09fa1`, `95d6613a1c72`, `93c883070a1b`, `c5f627099ad9`의 exact diff와 해당 시점의 parser, main loop, dispatcher, text/number/hex source를 확인했습니다. Build와 test binary는 이 환경에서 실행하지 않았습니다.
