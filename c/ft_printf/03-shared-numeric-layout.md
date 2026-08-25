# Thread: 숫자 배치 — conversion별 복제에서 하나의 layout 불변 조건으로

## 개요

숫자 출력은 digit를 만드는 일보다 배치 순서를 맞추는 일이 더 어렵습니다. 부호나 `0x` 접두사, field width, `0` flag, precision zero, left alignment가 함께 등장하면 같은 바이트 수라도 순서가 다르면 결과가 달라집니다.

이 Thread는 decimal과 hexadecimal이 각자 같은 규칙을 복제하던 상태에서, 다음 한 가지 emission order로 수렴하는 과정을 다룹니다.

```text
앞 공백
→ prefix (`-`, `+`, space, `0x`, `0X`)
→ field-level zero padding
→ precision이 요구한 zero
→ digits
→ LEFT 정렬의 뒤 공백
```

field-level zero와 precision zero는 둘 다 문자 `'0'`이지만 의미가 다릅니다. 전자는 width를 채우고 precision이 명시되면 비활성화됩니다. 후자는 최소 digit 수를 맞추며 prefix 뒤, 실제 digits 앞에 놓입니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `ac27a26affaa` | feat(decimal): 10진수 너비와 정렬 적용 | B | `FORMAT, LAYOUT` | decimal digits와 prefix를 하나의 field로 배치 |
| `c5ef742b84de` | feat(hex): 16진수와 포인터 너비와 정렬 적용 | B | `FORMAT, LAYOUT` | 같은 width/alignment 모델을 hex와 pointer에 복제 |
| `1fa064ca9d79` | feat(numeric): 숫자 정밀도와 0 채움 적용 | A | `FORMAT, LAYOUT, RISK` | digit suppression, precision zero, field zero precedence 도입 |
| `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | A | `PARSER, FORMAT, LAYOUT` | LEFT/ZERO, PLUS/SPACE 충돌을 정규화하고 prefix 선택 적용 |
| `f276ee73087c` | test(numeric): 접두사와 정밀도 배치 회귀 검증 | B | `FORMAT, TEST, EDGE` | 대표적인 prefix·precision·zero·left 조합을 회귀 사례로 고정 |
| `177c8d03b353` | refactor(output): 숫자 출력 배치 로직 통합 | A | `ARCH, LAYOUT, REFACTOR` | decimal·hex·pointer가 공유하는 한 layout writer 추출 |
| `ed3750fd081a` | fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산 | A | `FORMAT, EDGE, RISK` | signed 최솟값의 magnitude 계산에서 signed overflow 의존 제거 |
| `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | A | `FORMAT, TEST, EDGE` | zero precision, prefix, null pointer, 좁은 width의 public matrix 확대 |

## `ac27a26affaa` — decimal을 “완성된 field”로 보기 시작

**중요도** B · **태그** `FORMAT, LAYOUT`

직전 decimal 구현은 역순으로 만든 digit를 한 글자씩 출력했고, 음수 부호도 별도 preliminary write로 처리했습니다. 이 방식에서는 width가 부호를 포함한 전체 표현에 적용되기 어렵습니다.

이 커밋은 먼저 digits를 출력 순서로 materialize합니다.

```c
static int	ft_decimal_digits(char *buffer, unsigned long number)
{
	char	reversed[20];
	int		length;

	length = 0;
	if (number == 0)
		reversed[length++] = '0';
	while (number > 0)
	{
		reversed[length++] = (char)('0' + number % 10);
		number /= 10;
	}
	/* reversed를 buffer의 정방향으로 복사 */
	return (length);
}
```

그다음 prefix length와 digit length를 함께 width에서 뺍니다.

```c
padding = fmt->width - prefix_len - digit_len;
if (!(fmt->flags & FT_FLAG_LEFT))
	ft_printf_putnchar(ctx, ' ', padding);
ft_printf_write(ctx, prefix, (size_t)prefix_len);
ft_printf_write(ctx, digits, (size_t)digit_len);
if (fmt->flags & FT_FLAG_LEFT)
	ft_printf_putnchar(ctx, ' ', padding);
```

음수는 `"-"`, 비음수와 unsigned는 빈 prefix를 넘깁니다. 이 시점의 규칙은 right alignment의 `spaces → prefix → digits`, left alignment의 `prefix → digits → spaces`뿐이며 precision과 zero padding은 아직 없습니다.

## `c5ef742b84de` — 같은 배치가 hex와 pointer에 복제됨

**중요도** B · **태그** `FORMAT, LAYOUT`

Hex renderer도 digits를 정방향 buffer로 만든 뒤 `ft_write_hex` 안에서 prefix를 포함한 padding을 계산합니다.

```c
padding = fmt->width - prefix_len - digit_len;
/* spaces → prefix → digits → optional trailing spaces */
```

`x`/`X`는 빈 prefix, pointer는 `"0x"`를 넘깁니다. 따라서 `%8p`의 width는 `0x`와 hex digits를 모두 포함한 field에 적용됩니다.

기능은 맞지만 decimal의 `ft_write_decimal`과 hex의 `ft_write_hex`가 같은 계산과 같은 write 순서를 따로 갖게 됩니다. 아직은 width와 alignment만 있으므로 중복이 짧지만, 다음 커밋에서 precision과 zero precedence가 들어가면 두 복사본의 일치가 중요한 유지보수 문제가 됩니다.

## `1fa064ca9d79` — 숫자 layout의 실제 상태 공간이 만들어짐

**중요도** A · **태그** `FORMAT, LAYOUT, RISK`

이 커밋은 decimal과 hex 양쪽에 동일한 로직을 추가합니다.

### 1. 값 0 + precision 0이면 digits를 생략

```c
if (fmt->has_precision && fmt->precision == 0 && number == 0)
	digit_len = 0;
```

이 규칙은 digit representation만 없앱니다. prefix가 별도로 존재하면 prefix까지 자동으로 지우지는 않습니다. 예를 들어 hash hex는 값 0일 때 애초에 prefix를 선택하지 않지만, pointer는 프로젝트 규칙상 `0x` prefix를 유지할 수 있습니다.

### 2. precision이 요구한 최소 digit 수 계산

```c
zero_len = 0;
if (fmt->has_precision && fmt->precision > digit_len)
	zero_len = fmt->precision - digit_len;
```

`zero_len`은 digits 앞에 추가할 0의 수입니다. `%.5d`와 값 42라면 digit length 2, precision zero 3입니다.

### 3. width에서 모든 실제 구성 요소를 제외

```c
padding = fmt->width - prefix_len - zero_len - digit_len;
```

field의 content length는 다음 합입니다.

```text
prefix_len + precision_zero_len + digit_len
```

width가 더 크면 차이가 field padding입니다. 음수가 되면 `ft_printf_putnchar`가 `length > 0`일 때만 출력하므로 추가 padding은 없습니다.

### 4. `0` flag와 precision의 우선순위

```c
pad_char = ' ';
if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
	&& !fmt->has_precision)
	pad_char = '0';
```

숫자 precision이 명시되면 field-level ZERO flag는 적용되지 않습니다. LEFT도 zero padding을 비활성화합니다. 따라서 같은 `'0'`을 쓰더라도 두 종류의 zero가 동시에 뒤섞이지 않습니다.

### 5. emission 순서

```c
if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == ' ')
	ft_printf_putnchar(ctx, ' ', padding);
ft_printf_write(ctx, prefix, (size_t)prefix_len);
if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == '0')
	ft_printf_putnchar(ctx, '0', padding);
ft_printf_putnchar(ctx, '0', zero_len);
ft_printf_write(ctx, digits, (size_t)digit_len);
if (fmt->flags & FT_FLAG_LEFT)
	ft_printf_putnchar(ctx, ' ', padding);
```

Prefix를 field zero보다 먼저 쓰는 것이 핵심입니다. `%+08d`, 42는 `+0000042`여야 하며 `00000+42`가 되어서는 안 됩니다. `%-#10.4x`, 42는 `0x002a`를 먼저 만든 뒤 뒤쪽을 공백으로 채웁니다.

이 시점에는 동일한 30여 줄이 decimal과 hex에 각각 있습니다. 기능을 확장하면서 중복 비용이 실제로 커진 상태입니다.

## `c5f627099ad9` — layout에 들어오기 전 충돌 상태를 줄임

**중요도** A · **태그** `PARSER, FORMAT, LAYOUT`

Parser가 다음 두 precedence를 canonical state로 만듭니다.

```c
if (fmt->flags & FT_FLAG_LEFT)
	fmt->flags &= ~FT_FLAG_ZERO;
if (fmt->flags & FT_FLAG_PLUS)
	fmt->flags &= ~FT_FLAG_SPACE;
```

Layout 입장에서는 LEFT와 ZERO가 동시에 남지 않습니다. Signed prefix도 PLUS가 있으면 `+`, 아니면 SPACE가 있으면 한 칸을 선택합니다. Hex alternate prefix는 HASH이면서 값이 0이 아닐 때만 `0x` 또는 `0X`입니다.

```c
if ((fmt->flags & FT_FLAG_HASH) && number != 0)
	prefix = (fmt->spec == 'X') ? "0X" : "0x";
```

이 커밋은 Thread 02에서는 parser/dispatch 책임 분리로 읽히지만, 여기서는 layout 함수가 처리해야 할 조합을 줄이는 변경입니다. precision이 있을 때 ZERO를 무효화하는 조건은 여전히 layout에 남습니다. 그것은 field flag끼리의 정적 충돌이 아니라 `has_precision`이라는 다른 state와의 동적 precedence이기 때문입니다.

## `f276ee73087c` — 중복 구현이 같은 결과를 내는지 고정

**중요도** B · **태그** `FORMAT, TEST, EDGE`

이 커밋은 `snprintf`와 비교하는 기존 harness에 대표 조합을 추가합니다.

```c
EXPECT_PRINTF("empty:'%#.0x' '%#.0X' '% .0d'", 0u, 0u, 0);
EXPECT_PRINTF("signed-zero:'%+08d'", 42);
EXPECT_PRINTF("hex-zero:'%#08x'", 42u);
EXPECT_PRINTF("hex-left-precision:'%-#10.4x'", 42u);
EXPECT_PRINTF("signed-space-precision:'% 08.5d'", 42);
```

이 사례들은 다음 경계를 겨냥합니다.

- 값 0 + precision 0의 digit suppression
- sign 또는 `0x` prefix가 field zero보다 먼저 나오는지
- precision이 ZERO flag를 무효화하는지
- LEFT가 trailing spaces로 바뀌는지
- prefix와 precision zero가 width 안에 함께 계산되는지

테스트는 visible bytes와 반환 길이를 검증하지만, decimal과 hex에 중복된 코드가 앞으로도 함께 수정될 것이라는 구조적 보장은 주지 않습니다. 그 문제를 다음 refactor가 해결합니다.

## `177c8d03b353` — 하나의 layout writer로 수렴

**중요도** A · **태그** `ARCH, LAYOUT, REFACTOR`

이 커밋은 `src/ft_numeric_layout.c`를 추가하고 decimal/hex의 중복 계산을 삭제합니다. 각 conversion module은 이제 다음 두 가지만 책임집니다.

1. digits buffer와 길이 만들기
2. value에 맞는 prefix와 `is_zero` 전달하기

```c
/* decimal */
return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
		digit_len, number == 0));

/* hex / pointer */
return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
		digit_len, number == 0));
```

공통 함수의 실제 구현은 다음과 같습니다.

```c
int	ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
		const char *prefix, const char *digits, int digit_len, int is_zero)
{
	int		prefix_len;
	int		zero_len;
	int		padding;
	char	pad_char;

	if (fmt->has_precision && fmt->precision == 0 && is_zero)
		digit_len = 0;
	prefix_len = ft_prefix_length(prefix);
	zero_len = 0;
	if (fmt->has_precision && fmt->precision > digit_len)
		zero_len = fmt->precision - digit_len;
	padding = fmt->width - prefix_len - zero_len - digit_len;
	pad_char = ' ';
	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
		&& !fmt->has_precision)
		pad_char = '0';
	if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == ' '
		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
		return (-1);
	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
		return (-1);
	if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == '0'
		&& ft_printf_putnchar(ctx, '0', padding) < 0)
		return (-1);
	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
		return (-1);
	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
		return (-1);
	if ((fmt->flags & FT_FLAG_LEFT)
		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
		return (-1);
	return (0);
}
```

이 refactor의 의미는 단순 중복 제거가 아닙니다. decimal, unsigned, hex, pointer가 모두 같은 ordering invariant를 실행합니다. 이후 수정은 한 함수에서 이루어지고 모든 conversion에 동시에 적용됩니다. 각 write가 공통 output API를 통과하므로 부분 쓰기와 sticky error 처리도 우회하지 않습니다.

## `ed3750fd081a` — signed 최솟값을 표현 가능한 연산으로 바꿈

**중요도** A · **태그** `FORMAT, EDGE, RISK`

Signed renderer는 `int`를 `long`으로 변환한 뒤 음수 magnitude를 구했습니다.

```c
(unsigned long)(-value)
```

`long`이 `int`보다 넓은 환경에서는 `INT_MIN`을 negation할 수 있지만, 둘의 폭이 같은 환경에서는 `INT_MIN == LONG_MIN`일 수 있습니다. signed 최솟값의 양수 대응값은 같은 signed type으로 표현할 수 없으므로 `-value`가 overflow합니다.

수정은 1을 더해 representable range 안으로 옮긴 뒤 negation하고, 마지막 1은 unsigned 영역에서 더합니다.

```diff
-	(unsigned long)(-value)
+	(unsigned long)(-(value + 1)) + 1
```

예를 들어 `value == LONG_MIN`이면 `value + 1`은 표현 가능하고, 그 negation도 표현 가능합니다. cast 후 1을 더해 정확한 magnitude를 얻습니다. Layout 함수는 signed/unsigned origin을 알 필요 없이 이 magnitude의 digits와 `"-"` prefix를 배치합니다.

## `12d715eba77d` — public 경계 행렬과 프로젝트 고유 pointer 규칙

**중요도** A · **태그** `FORMAT, TEST, EDGE`

이 커밋은 signed, unsigned, hex에 대해 width가 content보다 작거나 비슷한 경우, precision 0, LEFT, ZERO, PLUS, SPACE, HASH 조합을 행렬로 확대합니다. libc와 동일한 의미가 기대되는 숫자 변환은 `EXPECT_PRINTF`로 비교합니다.

Pointer는 repository가 고정한 표현을 explicit expectation으로 검사합니다.

```c
EXPECT_OUTPUT("0x", "%.0p", (void *)0);
EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
EXPECT_OUTPUT("0x      ", "%-8.0p", (void *)0);
EXPECT_OUTPUT("  0x0000", "%8.4p", (void *)0);
EXPECT_OUTPUT("0x0000  ", "%-8.4p", (void *)0);
```

Null pointer에서 precision 0이면 shared layout이 digit `0`을 억제하지만 pointer prefix `0x`는 남깁니다. precision 4이면 prefix 뒤에 네 개의 precision zero가 옵니다. 이는 portable libc 출력에 맡기지 않고 프로젝트가 선택한 contract로 고정한 사례입니다.

이 테스트들이 확인하는 것은 열거된 public bytes와 반환 길이입니다. 모든 가능한 width/precision 정수 조합을 증명하지 않으며, 공통 layout 함수의 내부 산술 overflow를 별도 정적 증명하는 것도 아닙니다.

## 최종 layout 모델

먼저 유효 digit 길이를 정합니다.

```text
if precision is present and precision == 0 and value == 0:
    digit_len = 0
```

그다음 길이를 계산합니다.

```text
precision_zero = max(precision - digit_len, 0)  # precision이 있을 때
content = prefix_len + precision_zero + digit_len
padding = width - content
```

출력 순서는 정렬과 ZERO flag에 따라 다음과 같습니다.

| 조건 | 출력 순서 |
| --- | --- |
| 우측 정렬, space padding | spaces → prefix → precision zeros → digits |
| 우측 정렬, field zero padding | prefix → field zeros → digits |
| LEFT 정렬 | prefix → precision zeros → digits → spaces |
| precision 명시 | field ZERO는 사용하지 않고 precision zeros만 사용 |

Field zero path에는 `has_precision == false`이므로 별도의 precision zero가 없습니다. 모든 branch에서 prefix는 digits보다 앞에 있으며, error가 나면 이후 component는 출력하지 않습니다.

## Thread 경계

- Raw flags를 읽고 conflict를 정규화하는 parser 자체는 format-field Thread에서도 다룹니다.
- 전체 출력 길이를 같은 semantics로 미리 계산하는 measurement는 whole-call preflight Thread의 책임입니다.
- `%s` precision의 bounded read는 숫자 layout과 별개의 memory-access 문제입니다.
- `ft_printf_putnchar`가 chunk를 어떻게 기록하고 실패를 전파하는지는 output-state Thread의 문제입니다.

검사 과정에서는 각 commit의 exact diff와 `ft_numeric_layout.c` source를 확인했으며, 테스트는 실행하지 않았습니다.
