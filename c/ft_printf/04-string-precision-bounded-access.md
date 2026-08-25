# Thread: 문자열 정밀도 — 출력 길이 제한에서 메모리 접근 제한까지

## 개요

`%s` 정밀도가 “전체 문자열을 읽은 뒤 출력만 자르는 값”에서 “호출자가 제공한 객체를 읽을 수 있는 최대 범위”로 바뀌는 과정을 다룹니다.

최종 불변 조건은 단순합니다. 정밀도 `N`이 지정된 `%s`는 인덱스 `[0, N)` 밖의 바이트를 길이 계산에도 요구하지 않습니다. 이 범위 안에서 NUL을 먼저 만나면 거기서 멈추고, NUL이 없어도 `N`바이트에 도달하면 멈춥니다. 같은 길이가 너비 계산과 실제 출력에 사용되므로, 관찰되는 출력 길이와 메모리 접근 범위가 어긋나지 않습니다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | :---: | --- | --- |
| `8e1cee3ed7f0` | feat(text): 문자열 정밀도와 퍼센트 0 채움 적용 | B | `FORMAT, EDGE` | 전체 문자열 길이를 먼저 구한 뒤 출력 길이만 정밀도로 줄이는 중간 구현 |
| `9ac825379180` | fix(text): 문자열 정밀도 범위까지만 읽기 | A | `FORMAT, EDGE, RISK` | 정밀도를 스캔 종료 조건으로 옮겨 범위 밖 읽기를 제거 |
| `e040e69db535` | test(text): NUL 없는 제한 문자열 회귀 검증 | B | `FORMAT, TEST, EDGE` | NUL이 없는 3바이트 객체로 수정 경계를 고정 |

## `8e1cee3ed7f0` — 보이는 출력만 정밀도로 자른 첫 구현

**중요도** B · **태그** `FORMAT, EDGE`

이 커밋에는 formatted percent의 `0` 채움도 함께 들어 있지만, 문자열의 읽기 범위와는 무관하므로 이 Thread에서는 제외합니다.

`src/ft_text.c`의 문자열 경로는 NUL까지 전체 길이를 구한 뒤 그 결과만 정밀도로 줄였습니다.

```diff
 	if (string == 0)
 		string = "(null)";
 	length = ft_local_strlen(string);
+	if (fmt->has_precision && fmt->precision < (int)length)
+		length = (size_t)fmt->precision;
 	padding = fmt->width - (int)length;
```

이 변경으로 `%.3s`는 최대 3바이트만 출력하고, width도 잘린 길이를 기준으로 계산합니다. 그러나 `ft_local_strlen(string)`은 정밀도를 알지 못합니다.

```c
static size_t	ft_local_strlen(const char *string)
{
	size_t	length;

	length = 0;
	while (string[length])
		length++;
	return (length);
}
```

따라서 “출력할 바이트”와 “출력 길이를 알아내기 위해 읽는 바이트”가 다릅니다. 호출자가 정확히 3바이트만 읽을 수 있는 객체를 넘기고 `%.3s`를 요청해도, 길이 계산은 그 뒤에서 NUL을 찾으려 합니다. 보이는 출력은 올바르지만 그 출력에 도달하는 과정은 안전하지 않은 중간 상태입니다.

## `9ac825379180` — 정밀도를 스캔 조건으로 이동

**중요도** A · **태그** `FORMAT, EDGE, RISK`

수정은 출력 호출을 바꾸는 대신 길이 발견 자체를 바꿉니다.

```diff
-static size_t	ft_local_strlen(const char *string)
+static size_t	ft_local_strlen(const char *string, t_format *fmt)
 {
 	size_t	length;

 	length = 0;
-	while (string[length])
+	while ((!fmt->has_precision || length < (size_t)fmt->precision)
+		&& string[length])
 		length++;
 	return (length);
 }
```

```diff
-	length = ft_local_strlen(string);
-	if (fmt->has_precision && fmt->precision < (int)length)
-		length = (size_t)fmt->precision;
+	length = ft_local_strlen(string, fmt);
 	padding = fmt->width - (int)length;
```

핵심은 `&&`의 왼쪽 조건입니다. C는 논리 AND를 왼쪽부터 평가하므로 `length == precision`이면 `string[length]`는 평가되지 않습니다. 정밀도가 0이면 `string[0]`도 읽지 않습니다.

상태별 종료 조건은 다음과 같습니다.

| 입력 상태 | 길이 계산의 종료 지점 |
| --- | --- |
| 정밀도 없음 | NUL을 만날 때까지 진행 |
| 정밀도보다 앞에 NUL 존재 | NUL에서 종료 |
| 정밀도 범위 안에 NUL 없음 | `length == precision`에서 종료하고 다음 바이트는 읽지 않음 |
| 정밀도 0 | 첫 바이트를 평가하지 않고 길이 0 |

반환된 하나의 bounded length가 앞/뒤 공백 계산과 `ft_printf_write`의 길이에 그대로 들어갑니다. 사후 잘라내기가 사라졌기 때문에 출력과 읽기 범위를 따로 계산해 불일치시킬 여지도 줄었습니다.

이 커밋이 보장하지 않는 범위도 분명합니다. 정밀도가 없으면 정상적인 NUL 종료 문자열이 필요하고, 정밀도 범위 자체가 읽을 수 없는 잘못된 포인터까지 방어하지는 않습니다.

## `e040e69db535` — NUL 종료 문자열로는 보이지 않던 회귀 사례

**중요도** B · **태그** `FORMAT, TEST, EDGE`

일반 문자열 리터럴은 정밀도 뒤에도 NUL이 있으므로, 잘못된 전체 스캔과 올바른 bounded scan이 같은 출력으로 끝날 수 있습니다. 이 커밋은 의도적으로 NUL이 없는 정확히 3바이트짜리 객체를 만듭니다.

```diff
+static void	run_text_differential_cases(void)
+{
+	char	bounded[3];
+
+	bounded[0] = 'a';
+	bounded[1] = 'b';
+	bounded[2] = 'c';
+	EXPECT_PRINTF("%.3s", bounded);
+}
```

이 사례가 통과하는 production 경로는 다음과 같습니다.

```text
ft_printf
  → field parse (`precision = 3`)
  → `%s` dispatch
  → ft_printf_print_string
  → ft_local_strlen(string, fmt)
  → 0, 1, 2번 바이트만 검사하고 길이 3 반환
  → 정확히 "abc" 출력
```

이 테스트 커밋 자체는 `snprintf`와 반환값·출력 바이트를 비교하는 기능 테스트입니다. 따라서 단독으로는 “범위 밖 읽기가 절대로 발생하지 않았다”를 계측해 증명하지 못합니다. 잘못된 전체 스캔이 우연히 접근 가능한 메모리에서 NUL을 만나도 출력 비교가 통과할 수 있기 때문입니다. 다만 회귀를 재현할 입력을 정확히 고정하며, 후속 sanitizer 빌드에서 같은 사례를 실행하면 범위 밖 읽기를 훨씬 직접적으로 관찰할 수 있습니다.

## 최종 상태와 Thread 경계

```c
static size_t	ft_local_strlen(const char *string, t_format *fmt)
{
	size_t	length;

	length = 0;
	while ((!fmt->has_precision || length < (size_t)fmt->precision)
		&& string[length])
		length++;
	return (length);
}
```

이 Thread가 다루는 것은 `%s` 정밀도의 읽기 범위 하나입니다.

- `8e1cee3ed7f0`에 함께 포함된 percent(`%%`)의 `0` 채움은 별도의 formatting 규칙입니다.
- `%d`, `%x`, `%p`의 정밀도와 숫자 배치는 숫자 layout Thread의 문제입니다.
- 잘못된 전체 format을 첫 출력 전에 거부하는 문제는 whole-call preflight Thread의 문제입니다.
- ASan/UBSan target을 어떻게 구성하는지는 verification Thread에서 다룹니다.

이 경계들 사이에 반드시 선형 순서가 있는 것은 아닙니다. 여기서는 “정밀도가 출력 후처리가 아니라 메모리 접근 조건이어야 한다”는 한 가지 수정만 닫습니다.
