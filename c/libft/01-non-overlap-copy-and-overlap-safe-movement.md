# Thread: 비중첩 복사와 겹침 안전 이동의 책임 분리

> Project: `libft` · Branch: `c/libft` · 문서 번호: 01

## 개요

이 Thread는 “모든 byte 복사를 항상 가장 강한 연산으로 처리한다”는 방향 대신, 호출 전제가 다른 두 API를 분리한다. 핵심은 구현 복잡도를 숨기는 것이 아니라 caller가 선택해야 할 계약을 분명히 하고, 더 강한 계약이 필요한 경우에만 추가 비용과 분기를 부담하게 만드는 것이다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4873fb11ac60` | `feat(memory): 겹치지 않는 메모리 복사 구현` | B | `BYTE_RANGE, CORE` | 비중첩 범위를 순방향으로 복사하는 기본 primitive와 호출자 전제를 도입한다. |
| 2 | `f2c4c042b339` | `feat(memory): 겹치는 메모리의 안전한 이동 구현` | A | `BYTE_RANGE, CORE, RISK` | 겹침 배치에 따라 traversal 방향을 선택해 아직 읽지 않은 source byte를 보존한다. |
| 3 | `69853cd4d3ce` | `test(memory): 겹치는 메모리 이동 검증` | B | `BYTE_RANGE, TEST` | 방향별 겹침과 비겹침을 system `memmove` 및 전체 buffer 비교로 검증한다. |

## 4873fb11ac60 — feat(memory): 겹치지 않는 메모리 복사 구현
**중요도** `B` · **태그** `BYTE_RANGE, CORE`

### 무엇을 만들었는가 (diff)

이 커밋은 `Makefile`, `libft.h`, `src/memory/ft_memory_copy.c`를 함께 변경해 public declaration과 순방향 byte-copy loop를 archive build에 추가한다.

```diff
 SRC := \
 	src/char/ft_char.c \
-	src/memory/ft_memory_fill.c
+	src/memory/ft_memory_fill.c \
+	src/memory/ft_memory_copy.c

+void	*ft_memcpy(void *destination, const void *source, size_t length);

+void	*ft_memcpy(void *destination, const void *source, size_t length)
+{
+	unsigned char		*destination_byte;
+	const unsigned char	*source_byte;
+
+	destination_byte = destination;
+	source_byte = source;
+	while (length > 0)
+	{
+		*destination_byte = *source_byte;
+		destination_byte++;
+		source_byte++;
+		length--;
+	}
+	return (destination);
+}
```

`unsigned char` pointer를 사용하므로 object representation을 byte 단위로 옮긴다. loop 조건이 `length > 0`이어서 길이가 0이면 pointer를 역참조하지 않고, local pointer가 전진해도 반환값은 최초 `destination`이다.

### 왜 가볍게 다루는가

이 시점에는 traversal이 하나뿐이고, source와 destination이 겹치지 않는다는 전제를 caller가 지킨다. overlap 판정, 임시 buffer, 역방향 복사가 없는 것은 누락된 복구 로직이 아니라 이 primitive가 맡지 않는 책임이다. 겹침을 허용하는 순간 생기는 unread-source 파괴 위험은 다음 커밋에서 처음 다뤄진다.

### 어떤 커밋과 왜 연결되는가

`f2c4c042b339`은 이 순방향 primitive를 안전한 경우에 재사용하면서, destination이 source range 안에서 뒤쪽에 놓일 때만 더 강한 역방향 복사를 추가한다. `69853cd4d3ce`는 그 분기가 실제 byte 결과와 범위 보존으로 이어지는지 검증한다.

## f2c4c042b339 — feat(memory): 겹치는 메모리의 안전한 이동 구현
**중요도** `A` · **태그** `BYTE_RANGE, CORE, RISK`

### 무엇을 만들었는가 (diff)

`Makefile`, `libft.h`, `src/memory/ft_memory_move.c`에 새 API와 구현이 추가된다.

```diff
+void	*ft_memmove(void *destination, const void *source, size_t length);

+void	*ft_memmove(void *destination, const void *source, size_t length)
+{
+	unsigned char		*destination_byte;
+	const unsigned char	*source_byte;
+	size_t			offset;
+
+	destination_byte = destination;
+	source_byte = source;
+	if (destination_byte == source_byte || length == 0)
+		return (destination);
+	offset = 1;
+	while (offset < length)
+	{
+		if (destination_byte == source_byte + offset)
+		{
+			while (length > 0)
+			{
+				length--;
+				destination_byte[length] = source_byte[length];
+			}
+			return (destination);
+		}
+		offset++;
+	}
+	ft_memcpy(destination_byte, source_byte, length);
+	return (destination);
+}
```

### 왜 traversal 방향을 바꿔야 하는가

초기 buffer가 `A B C D E`이고 `source = &buffer[0]`, `destination = &buffer[1]`, `length = 4`라면 순방향 첫 write가 `buffer[1] = buffer[0]`을 수행한다. 다음 iteration에서 읽어야 할 원본 `B`가 이미 `A`로 바뀌므로 결과가 연쇄적으로 오염된다.

구현은 destination이 `source + 1`부터 `source + length - 1` 중 하나와 같은지 equality로 찾는다. 이 범위 안에서 시작하면 높은 offset부터 낮은 offset으로 복사한다. 반대로 destination이 source보다 앞선 겹침이거나 두 range가 떨어져 있으면, 앞선 write가 이후 source read를 파괴하지 않으므로 `ft_memcpy`의 순방향 loop를 재사용한다. `destination == source`와 `length == 0`은 탐색 전에 반환한다.

```text
source:       [0][1][2][3]
destination:    [0][1][2][3]
                 ↑ source + 1에서 시작

안전한 순서: 3 → 2 → 1 → 0
```

### 무엇을 보장하고 무엇은 남기는가

두 range가 각각 `length` byte만큼 유효하다는 전제 아래, 겹침 방향과 무관하게 호출 전 source byte가 destination에 보존된다. 길이가 0이면 memory access가 없다. 반면 nonzero length의 invalid pointer, 실제 object 범위 부족, 범위 유효성 검사는 이 API가 방어하지 않는다. 이 커밋 자체에는 새 테스트가 없어, 선택된 분기와 destination 바깥 byte 보존은 다음 커밋의 검증 대상이다.

### 어떤 커밋과 왜 연결되는가

`4873fb11ac60`의 순방향 복사는 안전한 배치의 하위 primitive로 남는다. `69853cd4d3ce`는 same-position, 양쪽 겹침 방향, 비겹침을 하나의 differential fixture로 통과시켜 이 방향 선택을 회귀 조건으로 만든다.

## 69853cd4d3ce — test(memory): 겹치는 메모리 이동 검증
**중요도** `B` · **태그** `BYTE_RANGE, TEST`

### 무엇을 검증하는가

`tests/test_memory_move.c`는 동일한 128-byte pattern을 가진 `actual`과 `expected`를 만들고 project 함수와 system `memmove`를 각각 적용한다. 반환 pointer뿐 아니라 buffer 전체를 비교하므로 destination 결과와 destination 밖의 예상치 못한 write를 함께 관찰한다.

```diff
+static void	check_move(size_t destination_offset, size_t source_offset,
+		size_t length)
+{
+	unsigned char	actual[MOVE_BUFFER_SIZE];
+	unsigned char	expected[MOVE_BUFFER_SIZE];
+
+	seed_move_buffer(actual);
+	memcpy(expected, actual, sizeof(actual));
+	CHECK(ft_memmove(actual + destination_offset, actual + source_offset, length)
+		== actual + destination_offset);
+	memmove(expected + destination_offset, expected + source_offset, length);
+	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
+}
+
+void	test_memory_move(void)
+{
+	static const size_t	lengths[] = {0, 1, 2, 3, 7, 8, 15, 16, 31, 63};
+	size_t			index;
+
+	CHECK(ft_memmove(NULL, NULL, 0) == NULL);
+	index = 0;
+	while (index < sizeof(lengths) / sizeof(lengths[0]))
+	{
+		check_move(0, 0, lengths[index]);
+		check_move(0, 1, lengths[index]);
+		check_move(1, 0, lengths[index]);
+		check_move(7, 19, lengths[index]);
+		check_move(23, 5, lengths[index]);
+		index++;
+	}
+}
```

`(0, 0)`은 same-position, `(0, 1)`은 destination-before-source, `(1, 0)`은 destination-inside-source 배치를 만든다. `(7, 19)`와 `(23, 5)`는 길이에 따라 비겹침과 겹침 사이를 오가므로 한 fixture에서 여러 branch를 통과한다.

### 무엇을 증명하고 무엇은 증명하지 않는가

선택된 offset·length 조합에서 반환 pointer와 전체 128-byte 결과가 system `memmove`와 같고, `ft_memmove(NULL, NULL, 0)`이 no-access 경로로 반환됨을 증명한다. 모든 가능한 배치, nonzero length의 잘못된 pointer, sanitizer가 탐지하는 모든 memory error까지 증명하지는 않는다.

### 어떤 커밋과 왜 연결되는가

이 테스트는 `f2c4c042b339`의 방향 판정과 `4873fb11ac60` 재사용 경로를 모두 통과한다. 특히 whole-buffer comparison은 구현이 destination 밖을 건드리는 회귀까지 함께 막는다.

## 이 Thread의 경계

이 Thread는 byte 복사에서 비겹침 전제와 겹침 보존 책임만 다룬다. allocation과 rollback은 [`02-single-allocation-to-rollback-safe-ownership`](02-single-allocation-to-rollback-safe-ownership.md), system call 진행 보장은 [`03-fd-output-partial-system-calls`](03-fd-output-partial-system-calls.md), sanitizer와 archive 검증은 [`04-static-archive-release-verification`](04-static-archive-release-verification.md)의 관심사다. memory fill·scan과 문자열 terminator 규칙은 이 문서 세트 밖의 별도 개발 단위다.

> 검토 범위: `4873fb11ac60`, `f2c4c042b339`, `69853cd4d3ce`의 commit diff와 해당 SHA의 `Makefile`, `libft.h`, memory 구현 및 `tests/test_memory_move.c`를 확인했다. binary, ordinary test suite, sanitizer는 실행하지 않았다.
