# Thread: 숫자 변환은 접두사·정밀도·너비를 하나의 배치 규칙으로 합성한다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 03

## 개요

숫자 conversion은 digits를 만드는 것만으로 끝나지 않습니다. Sign이나 `0x` prefix, precision이 요구한 zero, field width, ZERO flag, LEFT alignment가 함께 들어오면 같은 바이트 수라도 순서가 다르면 결과가 달라집니다. 이 Thread는 decimal과 hexadecimal이 각자 width/alignment를 구현하고 같은 precision logic을 복제한 뒤, 공통 `ft_printf_write_numeric_layout` 하나로 수렴하는 과정을 다룹니다.

최종 배치에서 field-level zero와 precision zero는 모두 문자 `'0'`이지만 책임이 다릅니다. Field zero는 width를 채우고 precision이 있으면 비활성화되며, precision zero는 최소 digit 수를 맞추고 prefix 뒤에 놓입니다.

| 조건 | 최종 출력 순서 |
| --- | --- |
| 우측 정렬, 공백 채움 | spaces → prefix → precision zeros → digits |
| 우측 정렬, ZERO flag | prefix → field zeros → digits |
| LEFT 정렬 | prefix → precision zeros → digits → spaces |

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ac27a26affaa` | feat(decimal): 10진수 너비와 정렬 적용 | B | `FORMAT, LAYOUT` | decimal digits와 prefix를 하나의 field로 배치 |
| 2 | `c5ef742b84de` | feat(hex): 16진수와 포인터 너비와 정렬 적용 | B | `FORMAT, LAYOUT` | 같은 width/alignment 모델을 hex와 pointer에 복제 |
| 3 | `1fa064ca9d79` | feat(numeric): 숫자 정밀도와 0 채움 적용 | A | `FORMAT, LAYOUT, RISK` | digit suppression, precision zero, field zero precedence 도입 |
| 4 | `c5f627099ad9` | feat(flags): 숫자 플래그 우선순위 정규화 | A | `PARSER, FORMAT, LAYOUT` | LEFT/ZERO와 PLUS/SPACE 충돌을 줄이고 prefix 선택 적용 |
| 5 | `f276ee73087c` | test(numeric): 접두사와 정밀도 배치 회귀 검증 | B | `FORMAT, TEST, EDGE` | 대표 prefix·precision·zero·left 조합을 회귀 사례로 고정 |
| 6 | `177c8d03b353` | refactor(output): 숫자 출력 배치 로직 통합 | A | `ARCH, LAYOUT, REFACTOR` | decimal·hex·pointer의 중복 layout을 공통 writer로 이동 |
| 7 | `ed3750fd081a` | fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산 | A | `FORMAT, EDGE, RISK` | signed 최솟값 magnitude 계산에서 direct negation 의존 제거 |
| 8 | `12d715eba77d` | test(printf): 공개 계약 경계 사례 확대 | A | `FORMAT, TEST, EDGE` | zero precision, prefix, null pointer, 좁은 width의 public matrix 확대 |

## Conversion별 field 배치가 먼저 구현된다

## ac27a26affaa — feat(decimal): 10진수 너비와 정렬 적용
**중요도** `B` · **태그** `FORMAT, LAYOUT`

### 무엇이 바뀌었는가 (diff)

직전 구현은 역순 digit를 한 글자씩 출력하고 음수 부호도 별도 write로 처리했습니다. 이 커밋은 digits를 출력 순서의 buffer로 먼저 만들고 prefix와 함께 width를 계산합니다.

```diff
+static int	ft_write_decimal(t_printf *ctx, t_format *fmt,
+		const char *prefix, unsigned long number)
+{
+	char	digits[20];
+	int		digit_len;
+	int		prefix_len;
+	int		padding;
+
+	digit_len = ft_decimal_digits(digits, number);
+	prefix_len = 0;
+	while (prefix[prefix_len])
+		prefix_len++;
+	padding = fmt->width - prefix_len - digit_len;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
+}
```

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

Sign을 digits 밖의 prefix로 표현하므로 width가 완성된 signed field 전체에 적용됩니다. Right alignment는 spaces → prefix → digits, LEFT는 prefix → digits → spaces입니다. Precision과 ZERO flag는 아직 처리하지 않으며, 규칙은 decimal module 안에만 있습니다.

### 관련 커밋

`c5ef742b84de`는 같은 모델을 hex와 pointer에 복제합니다. `1fa064ca9d79`은 두 복사본에 precision과 zero precedence를 추가해 중복 비용을 더 분명하게 만듭니다.

## c5ef742b84de — feat(hex): 16진수와 포인터 너비와 정렬 적용
**중요도** `B` · **태그** `FORMAT, LAYOUT`

### 무엇이 바뀌었는가 (diff)

```diff
+static int	ft_write_hex(t_printf *ctx, t_format *fmt,
+		const char *prefix, unsigned long number)
+{
+	char	digits[2 + sizeof(unsigned long) * 2];
+	int		digit_len;
+	int		prefix_len;
+	int		padding;
+
+	if (fmt->spec == 'X')
+		digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
+	else
+		digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
+	prefix_len = 0;
+	while (prefix[prefix_len])
+		prefix_len++;
+	padding = fmt->width - prefix_len - digit_len;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
+}
```

```diff
-	if (fmt->spec == 'X')
-		return (ft_print_base(ctx, (unsigned long)number,
-				"0123456789ABCDEF"));
-	return (ft_print_base(ctx, (unsigned long)number,
-			"0123456789abcdef"));
+	return (ft_write_hex(ctx, fmt, "", (unsigned long)number));
@@
-	(void)fmt;
-	if (ft_printf_write(ctx, "0x", 2) < 0)
-		return (-1);
-	return (ft_print_base(ctx, (unsigned long)(uintptr_t)pointer,
-			"0123456789abcdef"));
+	return (ft_write_hex(ctx, fmt, "0x",
+			(unsigned long)(uintptr_t)pointer));
```

Pointer는 `"0x"`, `x`/`X`는 빈 prefix를 넘기므로 `%8p`의 width에 prefix와 digits가 함께 포함됩니다.

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

Decimal과 hex가 같은 padding 계산과 write 순서를 별도 함수로 갖게 됩니다. Width/alignment만 있을 때는 중복이 짧지만 precision, ZERO, alternate prefix가 추가되면 두 구현이 같은 순서를 유지해야 하는 문제가 생깁니다.

### 관련 커밋

`1fa064ca9d79`은 두 함수에 같은 precision/zero state를 추가합니다. `177c8d03b353`은 이 중복을 하나의 layout writer로 이동합니다.

## Prefix·precision·ZERO의 우선순위가 완성되고 회귀 사례가 추가된다

## 1fa064ca9d79 — feat(numeric): 숫자 정밀도와 0 채움 적용
**중요도** `A` · **태그** `FORMAT, LAYOUT, RISK`

### 무엇을 만들었는가 (diff)

아래 변화가 decimal과 hex의 local layout 함수에 같은 형태로 들어갑니다.

```diff
 	char	digits[20];
 	int		digit_len;
 	int		prefix_len;
+	int		zero_len;
 	int		padding;
+	char	pad_char;
 
 	digit_len = ft_decimal_digits(digits, number);
+	if (fmt->has_precision && fmt->precision == 0 && number == 0)
+		digit_len = 0;
 	prefix_len = 0;
 	while (prefix[prefix_len])
 		prefix_len++;
-	padding = fmt->width - prefix_len - digit_len;
+	zero_len = 0;
+	if (fmt->has_precision && fmt->precision > digit_len)
+		zero_len = fmt->precision - digit_len;
+	padding = fmt->width - prefix_len - zero_len - digit_len;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
+		&& !fmt->has_precision)
+		pad_char = '0';
 	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == ' '
 		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
 		return (-1);
 	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
 		return (-1);
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == '0'
+		&& ft_printf_putnchar(ctx, '0', padding) < 0)
+		return (-1);
+	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
+		return (-1);
 	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
 		return (-1);
```

같은 hunk가 `ft_write_hex`에도 적용됩니다. Prefix 뒤에 field zero 또는 precision zero를 배치하고, LEFT일 때만 마지막에 spaces를 출력합니다.

### 설계 결정은 무엇인가

값 0에 precision 0이 명시되면 digit representation을 생략합니다. Precision이 digit 수보다 크면 그 차이만큼 prefix 뒤에 zero를 추가합니다. Width padding은 prefix, precision zero, digits의 실제 길이를 모두 제외한 나머지입니다.

ZERO flag는 precision이 없고 LEFT도 아닐 때만 field padding 문자로 사용됩니다. `%+08d`, 42가 `+0000042`가 되려면 prefix가 field zero보다 먼저 출력되어야 합니다. Precision이 있는 `%08.5d`에서는 field ZERO를 사용하지 않고 precision zero만 사용합니다.

### 아직 다루지 않는 것은 무엇인가

동일한 state calculation과 write 순서가 decimal과 hex에 복제되어 있습니다. 이 commit은 semantics를 완성하지만 책임을 한곳에 모으지는 않습니다.

### 관련 커밋

`c5f627099ad9`은 LEFT/ZERO와 PLUS/SPACE의 정적 충돌을 parser에서 제거해 layout 입력을 줄입니다. `f276ee73087c`은 대표 조합을 먼저 고정하고, `177c8d03b353`은 검증된 중복을 공통 writer로 통합합니다.

## c5f627099ad9 — feat(flags): 숫자 플래그 우선순위 정규화
**중요도** `A` · **태그** `PARSER, FORMAT, LAYOUT`

### 무엇이 바뀌었는가 (diff)

```diff
+static void	ft_normalize_flags(t_format *fmt)
+{
+	if (fmt->flags & FT_FLAG_LEFT)
+		fmt->flags &= ~FT_FLAG_ZERO;
+	if (fmt->flags & FT_FLAG_PLUS)
+		fmt->flags &= ~FT_FLAG_SPACE;
+}
```

Signed와 hexadecimal renderer는 normalized flags와 value를 사용해 prefix를 선택합니다.

```diff
+	if (fmt->flags & FT_FLAG_PLUS)
+		prefix = "+";
+	else if (fmt->flags & FT_FLAG_SPACE)
+		prefix = " ";
```

```diff
+	if ((fmt->flags & FT_FLAG_HASH) && number != 0)
+	{
+		if (fmt->spec == 'X')
+			prefix = "0X";
+		else
+			prefix = "0x";
+	}
```

### parser와 layout의 책임은 어떻게 나뉘는가

LEFT가 ZERO보다 우선하고 PLUS가 SPACE보다 우선한다는 사실은 raw flags만으로 결정할 수 있으므로 parser가 canonical state를 만듭니다. Precision이 field ZERO를 무효화하는 규칙은 `has_precision`과 field layout을 함께 봐야 하므로 layout에 남습니다. Value 0에서 HASH prefix를 생략하거나 음수에 `-`를 선택하는 판단도 renderer가 소유합니다.

### 관련 커밋

이 commit은 `02-format-fields-typed-dispatch.md`에서는 parser normalization으로 읽히고, 여기서는 layout state space를 줄이는 변경입니다. `177c8d03b353`의 공통 writer는 이 normalized input을 그대로 사용합니다.

## f276ee73087c — test(numeric): 접두사와 정밀도 배치 회귀 검증
**중요도** `B` · **태그** `FORMAT, TEST, EDGE`

### 무엇을 검증하는가

```diff
+static void	run_numeric_layout_cases(void)
+{
+	const char	*space_precision;
+
+	EXPECT_PRINTF("empty:'%#.0x' '%#.0X' '% .0d'", 0u, 0u, 0);
+	EXPECT_PRINTF("signed-zero:'%+08d'", 42);
+	EXPECT_PRINTF("hex-zero:'%#08x'", 42u);
+	EXPECT_PRINTF("hex-left-precision:'%-#10.4x'", 42u);
+	space_precision = "signed-space-precision:'% 08.5d'";
+	EXPECT_PRINTF(space_precision, 42);
+	EXPECT_PRINTF("hex-empty:'%#.0x'", 0u);
+}
```

이 사례들은 value 0 + precision 0의 digit suppression, prefix가 field zero보다 먼저 나오는지, precision이 ZERO flag를 무효화하는지, LEFT가 trailing spaces를 만드는지 확인합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 조합의 bytes와 return이 host `snprintf` 결과와 같다는 것을 증명합니다. Decimal과 hex의 중복 구현이 구조적으로 같은 code path를 사용한다는 보장은 주지 않으며, 모든 width·precision·value 조합을 exhaustive하게 검증하지도 않습니다.

### 관련 커밋

`177c8d03b353`은 이 테스트가 고정한 behavior를 유지하면서 중복 layout을 한 함수로 옮깁니다. `12d715eba77d`은 더 넓은 public boundary matrix와 project-specific pointer expectation을 추가합니다.

## 배치 책임이 공통 layout으로 수렴하고 표현 경계가 보강된다

## 177c8d03b353 — refactor(output): 숫자 출력 배치 로직 통합
**중요도** `A` · **태그** `ARCH, LAYOUT, REFACTOR`

이 커밋은 앞선 feature와 성격이 다릅니다. 새로운 formatting rule을 추가하지 않고 decimal·hex에 복제된 책임을 `src/ft_numeric_layout.c`로 이동합니다.

### 왜 필요한가

`1fa064ca9d79` 이후 두 module은 digit suppression, prefix length, precision zero, field padding, write 순서를 거의 동일하게 갖습니다. 어느 한쪽만 수정되면 conversion family가 서로 다른 semantics를 만들 수 있습니다.

### 무엇이 바뀌었는가 (diff)

삭제된 local block은 아래 diff에서 `/* ... */`로 접었고, 이동한 로직은 새 파일의 추가 hunk에 그대로 제시합니다.

```diff
 	digit_len = ft_decimal_digits(digits, number);
-	/* zero suppression부터 trailing padding까지의 local layout block ... */
-	return (0);
+	return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
+			digit_len, number == 0));
```

```diff
 	if (fmt->spec == 'X')
 		digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
 	else
 		digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
-	/* zero suppression부터 trailing padding까지의 local layout block ... */
-	return (0);
+	return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
+			digit_len, number == 0));
```

```diff
+static int	ft_prefix_length(const char *prefix)
+{
+	int	length;
+
+	length = 0;
+	while (prefix[length])
+		length++;
+	return (length);
+}
+
+int	ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
+		const char *prefix, const char *digits, int digit_len, int is_zero)
+{
+	int		prefix_len;
+	int		zero_len;
+	int		padding;
+	char	pad_char;
+
+	if (fmt->has_precision && fmt->precision == 0 && is_zero)
+		digit_len = 0;
+	prefix_len = ft_prefix_length(prefix);
+	zero_len = 0;
+	if (fmt->has_precision && fmt->precision > digit_len)
+		zero_len = fmt->precision - digit_len;
+	padding = fmt->width - prefix_len - zero_len - digit_len;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
+		&& !fmt->has_precision)
+		pad_char = '0';
+	if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == ' '
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (!(fmt->flags & FT_FLAG_LEFT) && pad_char == '0'
+		&& ft_printf_putnchar(ctx, '0', padding) < 0)
+		return (-1);
+	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
+}
```

### production behavior가 유지된다고 볼 수 있는 이유는 무엇인가

Conversion module은 같은 digits, prefix, `number == 0` 정보를 넘기고, 새 함수의 branch와 write 순서는 삭제된 두 local 구현과 같습니다. 모든 component는 계속 `ft_printf_write` 또는 `ft_printf_putnchar`를 통과하므로 output error contract도 바뀌지 않습니다.

### 관련 커밋

`f276ee73087c`의 regression이 refactor 전후 visible behavior를 지지합니다. `ed3750fd081a`은 layout 외부에서 signed magnitude를 안전하게 만들어 같은 common writer에 전달합니다.

## ed3750fd081a — fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산
**중요도** `A` · **태그** `FORMAT, EDGE, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
-		return (ft_write_decimal(ctx, fmt, "-",
-				(unsigned long)(-value)));
+		return (ft_write_decimal(ctx, fmt, "-",
+				(unsigned long)(-(value + 1)) + 1));
```

### 왜 이렇게 작은가

Signed 최솟값의 양수 magnitude는 같은 signed type으로 표현되지 않을 수 있습니다. Direct `-value`에 의존하지 않고 먼저 `value + 1`을 representable range로 옮겨 negation한 뒤, cast 이후 unsigned 영역에서 1을 더합니다. Layout writer는 이 magnitude의 origin을 알 필요 없이 digits와 `"-"` prefix를 배치합니다.

이 fix는 signed magnitude 계산만 바꾸며 prefix·precision·padding 순서는 건드리지 않습니다.

### 관련 커밋

`12d715eba77d`의 signed boundary matrix는 extrema와 width/precision 조합이 public path에서 유지되는지 확인합니다. Common layout 책임은 `177c8d03b353`에 남아 있습니다.

## 12d715eba77d — test(printf): 공개 계약 경계 사례 확대
**중요도** `A` · **태그** `FORMAT, TEST, EDGE`

### 무엇을 검증하는가

Signed, unsigned, hex에 대해 precision 0, 좁은 width, LEFT, ZERO, PLUS, SPACE, HASH 조합을 행렬로 확대합니다. Portable numeric cases는 `EXPECT_PRINTF`로 libc와 비교하고, repository가 명시적으로 선택한 pointer 결과는 fixed expectation으로 분리합니다.

```diff
+	EXPECT_OUTPUT("0x", "%.0p", (void *)0);
+	EXPECT_OUTPUT("      0x", "%8.0p", (void *)0);
+	EXPECT_OUTPUT("0x      ", "%-8.0p", (void *)0);
+	EXPECT_OUTPUT("  0x0000", "%8.4p", (void *)0);
+	EXPECT_OUTPUT("0x0000  ", "%-8.4p", (void *)0);
```

Null pointer에 precision 0이면 digit `0`은 억제되지만 project-defined `0x` prefix는 유지됩니다. Precision 4에서는 prefix 뒤에 네 개의 precision zero가 놓입니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

열거된 public bytes와 return을 고정하고, libc가 이식 가능한 oracle이 아닌 pointer 규칙을 repository contract로 분리합니다. 모든 정수 조합을 exhaustive하게 증명하거나, fixed pointer 표현이 표준적으로 유일한 선택임을 주장하지는 않습니다.

### 관련 커밋

이 commit은 `177c8d03b353`의 common layout과 `ed3750fd081a`의 signed magnitude fix를 넓은 public surface에서 검증합니다. Fixed expectation과 differential oracle의 구분 자체는 `06-runtime-artifact-verification.md`에서 검증 방법으로 설명합니다.

## 이 Thread의 경계

- Raw field parsing과 flag normalization의 parser 책임은 `02-format-fields-typed-dispatch.md`가 다룹니다.
- `%s` 정밀도의 메모리 접근 경계는 `04-string-precision-bounded-access.md`가 다룹니다.
- 같은 layout semantics를 first pass에서 길이로 재현하는 문제는 `05-whole-call-preflight.md`가 다룹니다.
- Padding chunk와 partial write failure는 `01-output-state-system-call-boundary.md`가 다룹니다.

> 검토 범위: `ac27a26affaa`, `c5ef742b84de`, `1fa064ca9d79`, `c5f627099ad9`, `f276ee73087c`, `177c8d03b353`, `ed3750fd081a`, `12d715eba77d`의 exact diff와 해당 시점의 decimal, hex, parser, common layout, public test source를 확인했습니다. 테스트 binary는 이 환경에서 실행하지 않았습니다.
