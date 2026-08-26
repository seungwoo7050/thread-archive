# Thread: 문자열 정밀도는 출력 길이와 메모리 접근 범위를 함께 제한한다
> Project: `ft_printf` · Branch: `c/ft_printf` · 문서 번호: 04

## 개요

`%s` 정밀도는 처음에는 전체 NUL 종료 길이를 구한 뒤 실제 출력 길이만 줄이는 규칙으로 구현되었습니다. 이 방식은 화면에 보이는 결과는 맞출 수 있지만, 정밀도 밖의 메모리를 읽지 않아야 한다는 계약은 만족하지 못합니다. 이 Thread는 정밀도를 사후 truncation에서 길이 탐색의 종료 조건으로 옮기고, NUL이 없는 정확한 크기의 객체로 그 경계를 고정하는 세 커밋을 다룹니다.

최종 계약은 하나입니다. 정밀도 `N`이 지정된 `%s`는 `[0, N)` 범위 안에서만 NUL을 찾으며, NUL을 먼저 만나지 않더라도 `N`바이트에 도달하면 종료합니다. 그 bounded length가 너비 계산과 실제 출력 길이에 함께 사용됩니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `8e1cee3ed7f0` | feat(text): 문자열 정밀도와 퍼센트 0 채움 적용 | B | `FORMAT, EDGE` | 전체 길이를 먼저 구한 뒤 출력 길이만 정밀도로 줄이는 중간 구현 |
| 2 | `9ac825379180` | fix(text): 문자열 정밀도 범위까지만 읽기 | A | `FORMAT, EDGE, RISK` | 정밀도를 스캔 종료 조건으로 옮겨 범위 밖 읽기 가능성을 제거 |
| 3 | `e040e69db535` | test(text): NUL 없는 제한 문자열 회귀 검증 | B | `FORMAT, TEST, EDGE` | NUL 없는 3바이트 객체로 bounded-read 경계를 회귀 사례로 고정 |

## 8e1cee3ed7f0 — feat(text): 문자열 정밀도와 퍼센트 0 채움 적용
**중요도** `B` · **태그** `FORMAT, EDGE`

### 무엇을 만들었는가 (diff)

이 커밋은 `%s`의 출력 길이를 정밀도 이하로 줄이고, 줄어든 길이로 field padding을 계산합니다.

```diff
 	if (string == 0)
 		string = "(null)";
 	length = ft_local_strlen(string);
+	if (fmt->has_precision && fmt->precision < (int)length)
+		length = (size_t)fmt->precision;
 	padding = fmt->width - (int)length;
```

같은 diff에 formatted percent의 `0` 채움 변경도 포함되지만, 문자열 읽기 범위와 무관하므로 여기서는 제외했습니다.

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

`%.3s`가 최대 3바이트만 출력하고 그 길이를 기준으로 정렬되므로 visible formatting은 갖춰졌습니다. 그러나 `ft_local_strlen(string)`은 여전히 정밀도를 모르며 `while (string[length])`로 NUL까지 먼저 진행합니다.

따라서 출력할 길이와 그 길이를 발견하기 위해 읽는 범위가 다릅니다. 호출자가 정확히 정밀도만큼만 readable한 객체를 넘기면 출력은 잘리더라도 길이 탐색은 객체 밖에서 NUL을 찾으려 할 수 있습니다.

### 관련 커밋

`9ac825379180`은 이 커밋이 도입한 사후 truncation을 제거하고 정밀도를 스캔 조건 자체로 이동합니다. formatted percent의 `0` 채움은 그 수정 대상이 아닙니다.

## 9ac825379180 — fix(text): 문자열 정밀도 범위까지만 읽기
**중요도** `A` · **태그** `FORMAT, EDGE, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

- **문제**: 출력은 정밀도를 지켜도 length discovery가 NUL까지 진행하므로, 정밀도만큼만 유효한 caller object 밖을 읽을 수 있었습니다.
- **원인**: 정밀도가 출력 길이의 사후 보정에만 쓰이고 메모리 접근 조건에는 들어가지 않았습니다.
- **결정**: length helper가 `t_format`을 받아 정밀도와 NUL 중 먼저 도달한 조건에서 멈추게 하고, renderer의 별도 truncation을 삭제했습니다.

### short-circuit이 읽기 범위를 어떻게 닫는가

이 로직은 줄 단위 평가 순서가 핵심이므로 수정 후 코드를 주석과 함께 봅니다.

```c
static size_t	ft_local_strlen(const char *string, t_format *fmt)
{
	size_t	length;

	length = 0;
	/*
	 * precision이 있으면 length < precision을 먼저 확인한다.
	 * length == precision이면 &&의 우항인 string[length]는
	 * 평가되지 않으므로 허용 범위 밖의 바이트를 요구하지 않는다.
	 */
	while ((!fmt->has_precision || length < (size_t)fmt->precision)
		&& string[length])
		length++;
	return (length);
}
```

호출부에서는 full scan 뒤 잘라내던 분기가 사라지고, 하나의 bounded length만 남습니다.

```diff
-	length = ft_local_strlen(string);
-	if (fmt->has_precision && fmt->precision < (int)length)
-		length = (size_t)fmt->precision;
+	length = ft_local_strlen(string, fmt);
 	padding = fmt->width - (int)length;
```

정밀도 0이면 `string[0]`도 평가하지 않습니다. 정밀도보다 앞에서 NUL을 만나면 NUL에서 멈추고, NUL이 없어도 정밀도에 도달하면 종료합니다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

정밀도가 있는 `%s`는 길이 탐색에서 `[0, precision)` 밖의 바이트를 요구하지 않습니다. 정밀도가 생략된 `%s`는 여전히 정상적인 NUL 종료 문자열을 필요로 하며, 정밀도 범위 자체도 읽을 수 없는 잘못된 포인터까지 방어하지는 않습니다.

### 관련 커밋

`e040e69db535`는 NUL이 없는 `char[3]`와 `%.3s`를 사용해 이 종료 조건을 직접 겨냥합니다. 후속 sanitizer target은 같은 테스트를 계측된 binary에서 실행할 수 있게 하지만, 그 빌드 구성은 이 Thread의 구현 변경과 별개의 검증 문제입니다.

## e040e69db535 — test(text): NUL 없는 제한 문자열 회귀 검증
**중요도** `B` · **태그** `FORMAT, TEST, EDGE`

### 무엇을 검증하는가

일반적인 문자열 리터럴은 정밀도 뒤에도 NUL이 있으므로 full scan과 bounded scan이 같은 출력으로 끝날 수 있습니다. 이 테스트는 NUL이 없는 정확히 3바이트짜리 객체를 만들어 그 차이를 드러냅니다.

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

`EXPECT_PRINTF`는 `snprintf`가 만든 기대 반환값·바이트와 stdout capture로 얻은 `ft_printf` 결과를 비교합니다. Production 경로는 `%s` dispatch에서 `ft_printf_print_string`으로 들어가고, 정밀도 3을 받은 helper가 인덱스 0·1·2만 검사한 뒤 길이 3을 반환합니다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

이 exact case에서 반환값과 출력 바이트가 `abc`로 일치하고, full-NUL-scan 방식으로 되돌아가는 회귀를 겨냥하는 입력이 존재함을 증명합니다. 이 커밋 자체에는 ASan 같은 범위 밖 접근 instrumentation이 없으므로, 우연히 접근 가능한 다음 메모리에서 NUL을 만나는 모든 경우까지 단독으로 검출한다고 주장할 수는 없습니다. 모든 길이·정밀도 조합의 메모리 안전성도 증명 범위 밖입니다.

### 관련 커밋

이 테스트는 `9ac825379180`의 bounded scan을 고정합니다. `1b474fa2a5e3`의 sanitizer build는 동일 functional suite를 계측해 out-of-bounds 회귀의 관찰력을 보강합니다.

## 이 Thread의 경계

- `8e1cee3ed7f0`에 함께 포함된 formatted percent의 `0` 채움은 문자열 메모리 접근과 별개의 formatting 규칙입니다.
- `%d`, `%x`, `%p`의 정밀도와 접두사·너비 배치는 `03-shared-numeric-layout.md`가 다룹니다.
- 전체 format과 총 결과 길이를 첫 출력 전에 판정하는 문제는 `05-whole-call-preflight.md`가 다룹니다.
- ASan/UBSan target과 artifact 검증의 역할은 `06-runtime-artifact-verification.md`가 다룹니다.

> 검토 범위: `8e1cee3ed7f0`, `9ac825379180`, `e040e69db535`의 exact diff와 해당 시점의 `src/ft_text.c`, `tests/test_ft_printf.c`를 확인했습니다. 테스트 binary와 sanitizer target은 이 환경에서 실행하지 않았습니다.
