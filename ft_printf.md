# Thread: 문자열 정밀도 — 출력 길이 제한에서 메모리 접근 제한까지

## 개요

`%s` precision이 "출력 후 truncation"이 아니라 "caller object를 읽을 수 있는 최대 범위"를 정하는
memory-access contract로 자리잡는 과정을 다룬다.

- **다루는 불변 조건**: precision이 있는 `%s`는 그 범위(`[0, precision)`) 밖의 byte를 요구하지 않는다.
  범위 밖에 NUL이 있는지 없는지도 상관하지 않는다.
- **최종 상태**: 문자열 길이 계산(`ft_local_strlen`)이 NUL과 precision 중 먼저 도달하는 조건에서
  멈춘다. 이 하나의 bounded length가 padding 계산과 실제 출력 길이에 그대로 쓰인다 — 출력과
  메모리 접근 범위가 항상 일치한다.

| SHA | 제목 | 중요도 | 태그 | 역할 |
| --- | --- | --- | --- | --- |
| `8e1cee3ed7f0` | feat(text): 문자열 정밀도와 퍼센트 0 채움 적용 | `B` | `FORMAT, EDGE` | 전체 길이를 먼저 구한 뒤 emitted length만 자른다 |
| `9ac825379180` | fix(text): 문자열 정밀도 범위까지만 읽기 | `A` | `FORMAT, EDGE, RISK` | precision을 스캔 종료 조건 자체로 옮겨 범위 밖 읽기를 없앤다 |
| `e040e69db535` | test(text): NUL 없는 제한 문자열 회귀 검증 | `B` | `FORMAT, TEST, EDGE` | NUL 없는 3-byte object로 이 경계를 회귀 테스트로 고정한다 |

---

## 8e1cee3ed7f0 — feat(text): 문자열 정밀도 출력 적용

**중요도** `A` · **태그** `FORMAT, EDGE`

> 이 커밋은 같은 diff 안에 percent(`%%`)의 zero-padding 변경도 포함하지만, 이 Thread의 주제(문자열
> precision)와 무관하므로 아래 diff와 서술에서 제외한다.

### 무엇을 만들었는가 (diff)
```diff
 	if (string == 0)
 		string = "(null)";
 	length = ft_local_strlen(string);
+	if (fmt->has_precision && fmt->precision < (int)length)
+		length = (size_t)fmt->precision;
 	padding = fmt->width - (int)length;
```

### 의도와 한계
`%s`에 precision이 있으면 출력 byte 수를 precision 이하로 줄이고, 그 줄어든 길이로 width padding을
계산하도록 만든 것이 이 커밋의 목적이다. 다만 `ft_local_strlen(string)`은 여전히 인자를 하나만 받고
NUL을 만날 때까지 전체 문자열을 스캔한 뒤, 그 결과를 여기서 사후적으로 자른다. 즉 **출력은
precision을 지키지만, 그 출력값을 얻기 위한 스캔 자체는 precision을 모른다.**

caller가 정확히 precision 길이만큼만 유효한 object를 넘긴다면 — 출력은 문제없이 잘려 나가는
것처럼 보여도 — 스캔이 그 범위 밖까지 진행된다는 위험이 여기서 만들어진다. 이 문제는 이 커밋
안에서 드러나지 않는다.

### 다음 커밋과의 연결
`9ac825379180`이 이 사후 truncation을 제거하고, precision을 스캔 조건 자체로 옮긴다.

---

## 9ac825379180 — fix(text): 문자열 정밀도 범위까지만 읽기

**중요도** `A` · **태그** `FORMAT, EDGE, RISK`

### 무엇이 바뀌었는가 (diff)
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
 	if (string == 0)
 		string = "(null)";
-	length = ft_local_strlen(string);
-	if (fmt->has_precision && fmt->precision < (int)length)
-		length = (size_t)fmt->precision;
+	length = ft_local_strlen(string, fmt);
 	padding = fmt->width - (int)length;
```

### 왜 바뀌었는가 (문제 → 원인 → 결정)
- **이전 동작**: precision은 "이후 emitted length를 줄이는 값"으로만 쓰였고, 길이를 구하는 스캔
  자체는 항상 NUL까지 진행했다.
- **문제**: caller가 넘긴 object가 정확히 precision 길이만큼만 readable하고 그 뒤에 NUL이 없다면,
  출력은 안전해도(precision까지만 나감) **스캔이 유효 범위 밖을 읽는다.**
- **원인**: precision이 "출력 길이 제한"으로만 구현되어 있었고, "메모리 접근 범위 제한"으로는
  구현되지 않았다.
- **결정**: `ft_local_strlen`이 `t_format *fmt`를 받아 스캔 종료 조건 자체에 precision을 넣는다.
  `!fmt->has_precision || length < precision`을 `&&`의 좌항에 두어, C의 short-circuit 평가에 의해
  `length == precision`이 되는 순간 `string[length]`는 평가조차 되지 않는다. 그 결과 하나의
  bounded length가 padding 계산과 출력 길이 양쪽에 그대로 쓰인다 — 따로 자르는 후처리가
  사라진다.

### 최종 코드 (학습용 주석)
```c
static size_t	ft_local_strlen(const char *string, t_format *fmt)
{
	size_t	length;

	length = 0;
	// precision이 없으면 좌항이 항상 참 -> 기존과 동일하게 NUL까지 스캔.
	// precision이 있으면 length가 precision에 도달하는 순간 좌항이 거짓이 되고,
	// && short-circuit에 의해 string[length]는 평가되지 않는다.
	// 즉 precision 밖의 메모리는 읽기 시도 자체를 하지 않는다.
	while ((!fmt->has_precision || length < (size_t)fmt->precision)
		&& string[length])
		length++;
	return (length);
}
```

### 이 커밋이 보장하는 것 / 보장하지 않는 것
- 보장: precision이 있는 `%s`는 `[0, precision)` 범위 밖의 byte를 요구하지 않는다.
- 비보장: precision이 없는 `%s`는 여전히 정상 NUL-terminated string이 필요하다. precision 범위
  자체도 유효하지 않은 잘못된 포인터까지는 방어하지 않는다.

### 관련 커밋
- 앞: `8e1cee3ed7f0` — 이 fix가 되돌리는 사후 truncation 방식의 도입.
- 뒤: `e040e69db535` — 이 경계를 재현하는 회귀 테스트.

---

## e040e69db535 — test(text): NUL 없는 제한 문자열 회귀 검증

**중요도** `B` · **태그** `FORMAT, TEST, EDGE`

### 무엇을 추가했는가 (diff)
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
+
 static void	run_error_cases(void)
```
```diff
 	run_numeric_layout_cases();
+	run_text_differential_cases();
 	run_error_cases();
```

### 무엇을 검증하는가
`9ac825379180`이 고친 bounded-read 불변 조건은 일반적인 NUL-terminated string으로는 검증할 수
없다 — 문자열이 NUL로 끝나 있으면, 스캔이 precision에서 멈추든 NUL에서 멈추든 결과가 같아서
회귀를 구별하지 못한다.

이 테스트는 **NUL이 전혀 없는 정확히 3-byte 배열**을 만들고, 그 배열에 정확히 일치하는
`%.3s`를 적용한다. 만약 length discovery가 다시 "먼저 NUL을 찾는" 방식으로 되돌아간다면,
이 배열에는 3번 인덱스 밖에 NUL이 있다는 보장이 없으므로 스캔이 배열 경계를 넘어간다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것
- 증명: 이 정확한 경계 사례(길이 3, precision 3, NUL 없음)에서 정확히 3 byte만 요구된다.
- 증명하지 않음: 모든 길이/precision 조합의 메모리 안전성. 이 테스트는 functional comparison
  (`snprintf`와의 반환값/바이트 비교)만 수행하며, out-of-bounds read 자체를 감지하는
  instrumentation(ASan/UBSan 등)은 포함하지 않는다 — 그 부분은 이 Thread 밖의 관심사다.

### 관련 커밋
- 앞: `9ac825379180` — 이 테스트가 고정하는 fix.

---

## 이 Thread의 경계

이 Thread는 **`%s` precision이 memory-access 범위를 정의한다는 사실 하나**만 다룬다. 같은 커밋이나
인접한 작업 중 아래 항목들은 이 Thread에 속하지 않는다.

- `8e1cee3ed7f0`에 함께 포함된 percent(`%%`)의 zero-padding 동작 — 별개의 formatting 규칙이며
  memory-access와 무관하다.
- 이 테스트를 ASan/UBSan 같은 sanitizer 빌드로 실행하는 설정 — out-of-bounds read를 실제로
  "관찰"하는 문제이며, 이 Thread는 그 이전 단계(접근 자체를 막는 것)까지만 다룬다.
- `%s` 이외의 conversion(`%d`, `%x` 등)에서의 precision 의미 — 이 Thread의 불변 조건은 문자열
  conversion에 한정된다.
- 전체 format 문자열을 출력 전에 검증하는 문제, 실제 output syscall 실패 처리 — 이 프로젝트의
  다른 관심사이며 이 Thread의 커밋 3개와는 별도로 진행된다.

이 Thread들은 프로젝트 안에서 순서 관계로 존재하지 않을 수 있다 — 병렬로 진행된 다른 단위일
수도 있으므로, 여기서는 "다음 Thread"를 지정하지 않고 위와 같이 **이 Thread가 다루지 않는
범위**만 명시한다.