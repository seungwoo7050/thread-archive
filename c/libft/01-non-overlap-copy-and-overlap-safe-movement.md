# Thread: 비중첩 복사와 겹침 안전 이동의 책임 분리

> Project: `libft`  
> Branch: `c/libft`

## 개요

이 Thread는 `ft_memcpy`와 `ft_memmove`를 같은 함수의 약한 버전과 강한 버전으로 보지 않는다. 두 함수는 애초에 책임 범위가 다르다.

- `ft_memcpy`는 **두 범위가 겹치지 않는다는 호출자 전제** 아래에서 앞에서부터 byte를 복사한다.
- `ft_memmove`는 두 범위가 겹칠 수 있으므로, destination이 아직 읽지 않은 source byte를 먼저 덮지 않는 방향을 선택한다.
- 테스트는 같은 위치, 두 겹침 방향, 비겹침 배치를 system `memmove`와 비교해 이 차이를 고정한다.

핵심 불변 조건은 다음과 같다.

> 길이가 `n`인 복사는 유효한 `[0, n)` 범위만 읽고 쓴다. `n == 0`이면 memory에 접근하지 않는다. 겹치는 이동에서는 이후에 읽어야 할 source byte가 앞선 write로 파괴되지 않아야 한다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `4873fb11ac60` | `feat(memory): 겹치지 않는 메모리 복사 구현` | B | `BYTE_RANGE`, `CORE` | 비중첩 범위를 순방향으로 복사하는 기본 primitive와 호출자 전제를 도입한다. |
| 2 | `f2c4c042b339` | `feat(memory): 겹치는 메모리의 안전한 이동 구현` | A | `BYTE_RANGE`, `CORE`, `RISK` | range 배치에 따라 복사 방향을 선택해 아직 읽지 않은 source를 보존한다. |
| 3 | `69853cd4d3ce` | `test(memory): 겹치는 메모리 이동 검증` | B | `BYTE_RANGE`, `TEST` | 겹침 방향과 비겹침 배치를 system `memmove`와 전체 buffer 단위로 비교한다. |

## `4873fb11ac60` — 비중첩 복사의 최소 책임

이 커밋은 `Makefile`, `libft.h`, 새 구현 파일을 함께 변경해 `ft_memcpy`를 공개 API와 archive build에 추가한다.

```c
void	*ft_memcpy(void *destination, const void *source, size_t length)
{
	unsigned char		*destination_byte;
	const unsigned char	*source_byte;

	destination_byte = destination;
	source_byte = source;
	while (length > 0)
	{
		*destination_byte = *source_byte;
		destination_byte++;
		source_byte++;
		length--;
	}
	return (destination);
}
```

구현이 보장하는 범위는 분명하다.

- source는 `const unsigned char *`, destination은 `unsigned char *`로 다뤄 byte representation을 그대로 옮긴다.
- loop는 `length > 0`일 때만 pointer를 역참조한다. 따라서 길이가 0이면 source와 destination을 읽거나 쓰지 않는다.
- local pointer는 전진하지만 반환값은 원래 매개변수 `destination`이다.
- overlap 판정, 임시 buffer, 역방향 복사는 없다.

마지막 항목은 누락이 아니라 이 함수의 precondition이다. source와 destination이 겹치면 앞선 write가 뒤에서 읽어야 할 source를 바꿀 수 있지만, 그 경우의 보존 책임은 `ft_memcpy`에 없다. 모든 복사에 overlap 처리를 넣는 대신, 필요한 호출만 더 강한 `ft_memmove`를 선택하게 한다.

## `f2c4c042b339` — 겹침 방향을 보고 traversal을 바꾸다

### 순방향 복사가 깨지는 배치

초기 buffer가 다음과 같다고 하자.

```text
index:       0 1 2 3 4
before:      A B C D E
source:      0 ───── 3
destination:   1 ───── 4
```

`source = &buffer[0]`, `destination = &buffer[1]`, `length = 4`를 앞에서부터 복사하면 첫 write가 `buffer[1] = buffer[0]`을 수행한다. 아직 다음 iteration에서 읽어야 할 원본 `B`가 이미 `A`로 바뀐다. 이후에는 변경된 값이 연쇄적으로 전달된다.

따라서 destination이 source range 안쪽의 더 높은 위치에서 시작할 때는 끝 byte부터 복사해야 한다.

### 실제 결정 코드

```c
void	*ft_memmove(void *destination, const void *source, size_t length)
{
	unsigned char		*destination_byte;
	const unsigned char	*source_byte;
	size_t				offset;

	destination_byte = destination;
	source_byte = source;
	if (destination_byte == source_byte || length == 0)
		return (destination);
	offset = 1;
	while (offset < length)
	{
		if (destination_byte == source_byte + offset)
		{
			while (length > 0)
			{
				length--;
				destination_byte[length] = source_byte[length];
			}
			return (destination);
		}
		offset++;
	}
	ft_memcpy(destination_byte, source_byte, length);
	return (destination);
}
```

판정은 단순한 pointer 대소 비교가 아니라, destination이 `source + 1`부터 `source + length - 1` 사이의 어느 위치와 같은지를 찾는 방식이다.

- `destination == source` 또는 `length == 0`: 즉시 반환한다. overlap 탐색이나 memory access가 없다.
- destination이 source range 내부의 더 높은 offset에서 시작한다: `length - 1`부터 0까지 역방향으로 복사한다.
- destination이 source보다 앞에 있는 겹침이거나 두 range가 떨어져 있다: 순방향 복사가 안전하므로 `ft_memcpy`를 재사용한다.
- destination이 정확히 `source + length`에서 시작하면 두 range는 맞닿아 있을 뿐 겹치지 않으므로 역시 `ft_memcpy` 경로로 간다.

역방향 branch에서는 가장 높은 source byte를 먼저 읽고 같은 offset의 destination에 쓴다. 이후에 읽을 낮은 source offset은 아직 건드리지 않으므로 원본 정보가 유지된다.

### 이 커밋이 보장하지 않는 것

`ft_memmove`도 source와 destination의 각 `length` byte 범위가 유효하다는 전제까지 대신 검증하지 않는다. nonzero length에 잘못된 pointer를 넘기는 경우나, 유효 범위 자체가 부족한 경우는 이 API의 방어 범위가 아니다.

또한 이 커밋 자체에는 새 테스트가 없다. 두 겹침 방향과 destination 바깥 byte가 실제로 보존되는지는 다음 커밋에서 고정된다.

## `69853cd4d3ce` — 방향별 결과와 범위 밖 write를 함께 검증

테스트는 128-byte `actual`과 `expected`를 같은 pattern으로 채운다. 한쪽에는 project `ft_memmove`, 다른 쪽에는 system `memmove`를 적용한 뒤 **destination만이 아니라 buffer 전체**를 비교한다.

```c
static void	check_move(size_t destination_offset, size_t source_offset,
		size_t length)
{
	unsigned char	actual[MOVE_BUFFER_SIZE];
	unsigned char	expected[MOVE_BUFFER_SIZE];

	seed_move_buffer(actual);
	memcpy(expected, actual, sizeof(actual));
	CHECK(ft_memmove(actual + destination_offset, actual + source_offset, length)
		== actual + destination_offset);
	memmove(expected + destination_offset, expected + source_offset, length);
	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
}
```

이 비교 방식은 두 가지를 동시에 확인한다.

1. destination의 결과 byte가 system `memmove`와 같다.
2. 지정한 destination range 밖의 byte가 예상하지 않게 바뀌지 않는다.

실제 case는 다음 offset 조합을 여러 길이와 교차한다.

| 호출 | 대표 의미 |
| --- | --- |
| `check_move(0, 0, length)` | 같은 위치 |
| `check_move(0, 1, length)` | destination이 source보다 앞선 겹침 또는 비겹침 |
| `check_move(1, 0, length)` | destination이 source 안쪽의 뒤에서 시작하는 겹침 |
| `check_move(7, 19, length)` | 길이에 따라 비겹침에서 destination-before-source 겹침으로 전환 |
| `check_move(23, 5, length)` | 길이에 따라 비겹침에서 역방향 복사가 필요한 겹침으로 전환 |

길이 집합은 `0, 1, 2, 3, 7, 8, 15, 16, 31, 63`이다. 별도로 다음 assertion이 zero-length no-access 경계를 직접 겨냥한다.

```c
CHECK(ft_memmove(NULL, NULL, 0) == NULL);
```

이 테스트는 선택된 offset·length 조합에서 반환 pointer와 전체 buffer 결과가 system 함수와 같음을 증명한다. 모든 가능한 배치나 잘못된 pointer를 증명하지는 않으며, sanitizer instrumentation도 이 커밋에는 포함되지 않는다.

## Thread가 확정한 책임 경계

```text
호출자가 non-overlap을 보장
    └─ ft_memcpy: 낮은 offset → 높은 offset

overlap 가능
    └─ ft_memmove
         ├─ same pointer 또는 length 0 → 즉시 반환
         ├─ destination ∈ source[1, length) → 높은 offset → 낮은 offset
         └─ 그 밖의 배치 → ft_memcpy 재사용
```

이 Thread의 결론은 “모든 복사를 더 복잡하게 만든다”가 아니다. 약한 연산의 precondition을 숨기지 않고, 겹침 보존이 필요한 곳에서만 방향 선택 비용을 갖는 더 강한 연산을 사용한다.

## Thread의 범위

이 문서는 byte 복사와 겹침 보존만 다룬다. memory fill, scan, 문자열 함수의 terminator 규칙, invalid pointer 방어, sanitizer build는 별개의 개발 단위다.

> 검토 범위: 표시된 exact SHA의 commit diff와 해당 SHA source를 확인했다. binary와 test suite는 실행하지 않았다.
