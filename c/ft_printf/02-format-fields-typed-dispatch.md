# Thread: 포맷 필드와 typed dispatch — raw 문자열을 정규화된 실행 입력으로

## 개요

초기 `ft_printf`는 format cursor를 직접 보며 리터럴과 `%%`만 구분했습니다. 이 Thread는 `%` 뒤의 flags·width·precision·specifier를 `t_format` 하나로 표현하고, 변환 문자가 요구하는 정확한 promoted type을 dispatcher가 소비하도록 책임을 나누는 과정을 다룹니다.

최종 책임 분리는 다음과 같습니다.

```text
main traversal
  └─ '%'를 만나면 parser 호출, 반환된 next cursor로 이동

parser
  └─ raw field → t_format { flags, width, precision, has_precision, spec }

argument dispatcher
  └─ spec별 정확한 va_arg type 소비 → typed renderer 호출

renderer
  └─ 이미 정규화된 field와 typed value로 출력 구성
```

이 분리가 중요한 이유는 variadic argument consumption이 되돌릴 수 없는 상태 변화이기 때문입니다. `%d`를 `unsigned int`로 읽거나 `%p`를 정수형으로 추측하면 단순 표시 오류가 아니라 `va_list` 사용 규칙 자체를 깨뜨립니다. 또한 각 renderer가 raw format을 다시 해석하면 parser와 서로 다른 flag·width·precision 의미를 만들 수 있습니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `7984ddf2dd57` | feat(parser): 포맷 필드 모델과 해석기 추가 | S | `ARCH, PARSER, CORE` | normalized field representation과 overflow-checked parser 도입 |
| `9e6d785628f3` | feat(core): 포맷 필드 해석을 출력 루프에 연결 | B | `PARSER, INTEGRATION` | parser가 반환한 cursor/error를 main traversal에 연결 |
| `03c3e6e09fa1` | feat(text): 문자·문자열·퍼센트 변환 추가 | A | `ARCH, FORMAT, VARARGS` | `va_arg` 타입 선택을 소유하는 dispatcher와 text renderer 도입 |
| `95d6613a1c72` | feat(decimal): 부호 있는·없는 10진수 출력 추가 | B | `FORMAT, VARARGS` | `int`와 `unsigned int` conversion을 같은 boundary 안에 추가 |
| `93c883070a1b` | feat(hex): 16진수와 포인터 출력 추가 | B | `FORMAT, VARARGS` | hex와 pointer의 정확한 argument type 및 renderer 추가 |
| `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | A | `PARSER, FORMAT, LAYOUT` | 충돌 flag를 parser에서 canonical state로 줄이고 prefix 선택 적용 |

## `7984ddf2dd57` — 한 field를 표현하는 공통 데이터 모델

**중요도** S · **태그** `ARCH, PARSER, CORE`

### 이전 상태와 결정

직전 public loop는 `%`와 다음 `%`만 직접 검사했습니다. flags, width, precision이 생기면 각 conversion이 raw cursor를 별도로 읽거나 main loop가 모든 세부 규칙을 알아야 했습니다. 특히 아래 두 상태를 raw 숫자 값만으로는 구분할 수 없습니다.

- precision이 생략됨: `%s`
- precision이 명시적으로 0임: `%.0s`

둘 다 `precision == 0`일 수 있으므로 별도 `has_precision`이 필요합니다. 이 커밋은 field를 다음 구조체로 고정합니다.

```c
typedef struct s_format
{
	int		flags;
	int		width;
	int		precision;
	int		has_precision;
	char	spec;
}	t_format;
```

`flags`는 반복해서 등장해도 bitwise OR로 같은 상태에 수렴합니다. `width`와 `precision`은 public 계산에 쓰이는 `int`, `has_precision`은 optional state, `spec`은 dispatcher가 사용할 conversion 식별자입니다.

### grammar와 cursor 이동

`ft_printf_parse`는 `%` 다음 위치를 받아 다음 순서로 소비합니다.

```text
[flags 반복] → [width decimal] → ['.'가 있으면 precision decimal] → [specifier 1바이트]
```

```c
ft_printf_init_format(fmt);
flag = ft_flag_value(*format);
while (flag)
{
	fmt->flags |= flag;
	format++;
	flag = ft_flag_value(*format);
}
if (ft_parse_decimal(&format, &fmt->width) < 0)
	return (0);
if (*format == '.')
{
	fmt->has_precision = 1;
	format++;
	if (ft_parse_decimal(&format, &fmt->precision) < 0)
		return (0);
}
fmt->spec = *format;
if (*format)
	format++;
return (format);
```

성공하면 반환값은 다음 아직 읽지 않은 문자를 가리킵니다. specifier가 문자열 끝의 NUL이면 `spec = '\0'`이고 cursor는 더 전진하지 않습니다. 이 exact SHA의 parser는 field를 해석할 뿐 지원 여부까지 판정하지 않습니다.

### decimal overflow를 누적 전에 차단

```c
while (ft_is_digit(**format))
{
	digit = **format - '0';
	if (*value > (INT_MAX - digit) / 10)
		return (-1);
	*value = *value * 10 + digit;
	(*format)++;
}
```

검사는 `value * 10 + digit`를 수행하기 전에 이루어집니다. overflow된 값을 만든 뒤 되돌리는 방식이 아니므로 실패 시 `int` 연산 자체가 overflow하지 않습니다. width와 precision이 같은 helper를 써서 두 필드의 범위 규칙도 일치합니다.

이 커밋이 만든 것은 parser module과 representation입니다. public `ft_printf`는 아직 이를 호출하지 않으므로 실제 출력 behavior는 다음 integration commit 전까지 바뀌지 않습니다.

## `9e6d785628f3` — parser 결과를 main cursor에 연결

**중요도** B · **태그** `PARSER, INTEGRATION`

main loop에 `t_format fmt`가 생기고, `%`에서 parser가 반환한 pointer를 다시 `format`에 저장합니다.

```c
else if (*format == '%')
{
	format = ft_printf_parse(format + 1, &fmt);
	if (format == 0)
	{
		ctx.error = 1;
		break ;
	}
	if (ft_printf_putchar(&ctx, '%') < 0)
		break ;
	if (fmt.spec != '\0' && fmt.spec != '%'
		&& ft_printf_putchar(&ctx, fmt.spec) < 0)
		break ;
}
```

이 단계는 parser integration을 확인하기 위한 중간 동작입니다. field의 flags/width/precision은 아직 렌더링에 쓰이지 않고, parse한 specifier를 `%`와 함께 그대로 출력합니다. trailing `%`도 `spec == '\0'`인 field로 해석되어 `%`만 출력합니다. 지원 conversion과 잘못된 conversion을 구분하는 public validation은 아직 없습니다.

작은 커밋이지만 cursor 소유권은 여기서 확정됩니다. parser가 field 끝까지 전진시키고 main loop는 그 반환값을 신뢰합니다. renderer가 별도로 format cursor를 움직이지 않기 때문에 이후 conversion 추가가 traversal 규칙을 바꾸지 않습니다.

## `03c3e6e09fa1` — argument consumption을 한곳에 모으는 경계

**중요도** A · **태그** `ARCH, FORMAT, VARARGS`

이 커밋은 `ft_dispatch.c`와 `ft_text.c`를 추가하고 main loop의 placeholder 출력을 dispatcher 호출로 바꿉니다.

```c
int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
{
	if (fmt->spec == 'c')
		return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
	if (fmt->spec == 's')
		return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
	if (fmt->spec == '%')
		return (ft_printf_print_percent(ctx, fmt));
	if (ft_printf_putchar(ctx, '%') < 0)
		return (-1);
	if (fmt->spec)
		return (ft_printf_putchar(ctx, fmt->spec));
	return (0);
}
```

### 왜 dispatcher가 `va_arg`를 소유하는가

`%c`에 전달된 `char`는 default argument promotion을 거쳐 `int`로 전달되므로 `va_arg(*args, int)`로 읽어야 합니다. `%s`는 `char *`이고 `%%`는 argument를 소비하지 않습니다. renderer가 각자 `va_list`를 받는 대신 이미 typed value를 받으면 다음 특성이 생깁니다.

- 한 specifier가 몇 개의 argument를 어떤 타입으로 소비하는지 한 파일에서 확인할 수 있습니다.
- renderer는 variadic traversal이 아니라 값의 표시 규칙만 담당합니다.
- 측정 pass가 나중에 생길 때 dispatcher와 같은 type map을 재현해야 한다는 비교 기준이 생깁니다.

text renderer는 이 시점에는 단순합니다.

```c
int	ft_printf_print_char(t_printf *ctx, t_format *fmt, int c)
{
	(void)fmt;
	return (ft_printf_putchar(ctx, (char)c));
}

int	ft_printf_print_string(t_printf *ctx, t_format *fmt,
		const char *string)
{
	(void)fmt;
	if (string == 0)
		string = "(null)";
	return (ft_printf_write(ctx, string, ft_local_strlen(string)));
}
```

field model은 이미 전달되지만 width/precision은 후속 commit에서 사용합니다. 즉 API boundary를 먼저 정하고 conversion semantics를 점진적으로 채우는 구조입니다.

이 exact SHA에서도 unknown specifier는 오류가 아니라 `%`와 해당 문자를 그대로 출력하는 fallback입니다. public 지원 문법을 제한하는 책임은 whole-call preflight가 도입되기 전까지 존재하지 않습니다.

## `95d6613a1c72` — decimal conversion은 type map을 확장할 뿐

**중요도** B · **태그** `FORMAT, VARARGS`

Dispatcher에 다음 두 분기가 추가됩니다.

```c
if (fmt->spec == 'd' || fmt->spec == 'i')
	return (ft_printf_print_signed(ctx, fmt, va_arg(*args, int)));
if (fmt->spec == 'u')
	return (ft_printf_print_unsigned(ctx, fmt,
			va_arg(*args, unsigned int)));
```

signed renderer는 `int`를 `long`으로 올리고, 음수면 `-`를 먼저 출력한 뒤 magnitude의 decimal digits를 만듭니다. unsigned renderer는 `unsigned int`를 그대로 unsigned magnitude로 전달합니다.

이 Thread에서 중요한 것은 digit algorithm 자체보다 argument type입니다. `%d`/`%i`와 `%u`가 비슷한 10진 문자열을 만들더라도 `va_arg` 타입은 서로 바꿀 수 없습니다. 이 커밋은 renderer 내부에 width/precision 배치를 아직 만들지 않으며, 그 문제는 numeric layout Thread에서 따로 다룹니다.

## `93c883070a1b` — hex와 pointer도 같은 dispatch boundary 안으로

**중요도** B · **태그** `FORMAT, VARARGS`

```c
if (fmt->spec == 'x' || fmt->spec == 'X')
	return (ft_printf_print_hex(ctx, fmt,
			va_arg(*args, unsigned int)));
if (fmt->spec == 'p')
	return (ft_printf_print_pointer(ctx, fmt, va_arg(*args, void *)));
```

`x`와 `X`는 `unsigned int`, `p`는 `void *`를 소비합니다. pointer renderer가 내부에서 `uintptr_t`를 거쳐 숫자 표현으로 바꾸더라도 variadic boundary에서는 반드시 원래 pointer type으로 꺼냅니다.

초기 renderer는 `%p`에 `"0x"`를 먼저 출력하고 나머지를 소문자 hex로 표시합니다. `x`/`X`는 base alphabet만 다르게 선택합니다. width와 prefix를 하나의 field로 배치하는 일은 후속 layout commit의 책임입니다.

### 최종 type-consumption 표

| specifier | `va_arg` 타입 | 전달되는 renderer 값 |
| --- | --- | --- |
| `c` | `int` | `(char)c`로 출력 |
| `s` | `char *` | null mapping 후 문자열 출력 |
| `d`, `i` | `int` | signed decimal |
| `u` | `unsigned int` | unsigned decimal |
| `x`, `X` | `unsigned int` | lower/upper hex |
| `p` | `void *` | pointer representation |
| `%` | 소비 없음 | literal percent |

## `c5f627099ad9` — parser가 충돌 상태를 줄이고 renderer가 prefix를 선택

**중요도** A · **태그** `PARSER, FORMAT, LAYOUT`

Parser는 이제 field를 반환하기 전에 두 충돌을 정규화합니다.

```c
static void	ft_normalize_flags(t_format *fmt)
{
	if (fmt->flags & FT_FLAG_LEFT)
		fmt->flags &= ~FT_FLAG_ZERO;
	if (fmt->flags & FT_FLAG_PLUS)
		fmt->flags &= ~FT_FLAG_SPACE;
}
```

이 결정은 각 renderer가 다음 네 조합을 매번 해석할 필요를 없앱니다.

- LEFT만 있음
- ZERO만 있음
- LEFT와 ZERO가 함께 있음
- 둘 다 없음

Parser 이후에는 LEFT가 있으면 ZERO는 존재하지 않습니다. PLUS/SPACE도 같은 방식으로 PLUS가 우선합니다. 정규화는 raw 입력을 잃어버리는 대신 downstream이 처리할 상태 공간을 줄입니다.

prefix의 실제 선택은 값과 specifier를 아는 renderer에 남습니다.

```c
/* nonnegative signed decimal */
prefix = "";
if (fmt->flags & FT_FLAG_PLUS)
	prefix = "+";
else if (fmt->flags & FT_FLAG_SPACE)
	prefix = " ";
```

```c
/* x / X */
prefix = "";
if ((fmt->flags & FT_FLAG_HASH) && number != 0)
{
	if (fmt->spec == 'X')
		prefix = "0X";
	else
		prefix = "0x";
}
```

Parser는 `+`가 `space`보다 우선한다는 field-level 규칙을 알고, renderer는 음수인지·0인지·대문자 hex인지에 따라 구체적인 prefix를 결정합니다. “정규화”와 “값에 따른 표현”을 같은 함수에 섞지 않은 경계입니다.

precision이 명시된 숫자에서 ZERO flag를 무효화하는 규칙은 이 parser 정규화에 들어가지 않습니다. 그 규칙은 numeric layout이 `has_precision`과 함께 판단합니다. 따라서 이 커밋이 모든 flag precedence를 parser 하나로 옮겼다고 해석하면 안 됩니다.

## 최종 데이터 흐름과 불변 조건

```text
raw bytes after '%'
  → ft_printf_parse
      flags OR
      width/precision overflow-before-mutation 검사
      omitted precision과 .0 구분
      spec 및 next cursor 확정
      LEFT>ZERO, PLUS>SPACE 정규화
  → ft_printf_dispatch
      spec별 정확한 va_arg type 한 번 소비
  → typed renderer
      value-dependent prefix/digits/text 결정
  → shared output API
```

| 불변 조건 | 실제 책임 위치 |
| --- | --- |
| field grammar를 한 번만 해석 | `ft_printf_parse` |
| width/precision은 `int` 범위 안에서만 저장 | decimal parser의 사전 검사 |
| `.0`과 precision 생략을 구분 | `has_precision` |
| main cursor는 한 field 뒤 정확한 위치로 이동 | parser 반환 pointer를 main loop가 채택 |
| specifier별 promoted type을 정확히 소비 | `ft_printf_dispatch` |
| LEFT와 ZERO, PLUS와 SPACE 충돌 제거 | `ft_normalize_flags` |
| 부호·alternate prefix는 값과 spec을 아는 곳에서 선택 | decimal/hex renderer |

## Thread 경계

- 이 Thread의 마지막 시점에도 unknown/trailing specifier fallback이 dispatcher에 남아 있습니다. 지원 문법 전체를 first write 전에 거부하는 변경은 whole-call preflight Thread에서 다룹니다.
- width·precision·prefix가 실제 어떤 순서로 배치되는지는 numeric layout Thread의 문제입니다.
- `%s` precision이 읽기 범위까지 제한하는지는 string bounded-access Thread의 문제입니다.
- 같은 type map을 measurement pass가 어떻게 재현하는지는 preflight Thread에서 다시 확인합니다.

검사 과정에서는 각 SHA의 parser, dispatcher, renderer diff/source를 확인했으며, final HEAD의 validation behavior를 앞선 commit에 소급하지 않았습니다.
