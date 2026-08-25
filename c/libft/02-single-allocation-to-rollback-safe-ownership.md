# Thread: 단일 할당에서 완전 성공 또는 롤백까지

> Project: `libft`  
> Branch: `c/libft`

## 개요

이 Thread는 반환할 allocation이 하나뿐인 함수에서 시작해, root와 여러 child를 가진 결과와 callback이 만든 content를 포함하는 list까지 ownership 규칙을 확장한다.

최종 규칙은 다음과 같다.

> 생성 함수는 완성된 결과만 caller에게 넘긴다. 중간 allocation 하나라도 실패하면, 그 시점까지 획득한 자원을 정확히 한 번씩 해제하고 partial root를 반환하지 않는다.

이 규칙은 세 층으로 진행된다.

1. `ft_calloc`, `ft_strdup`, `ft_substr`, `ft_strjoin`: 크기를 확정한 뒤 하나의 allocation을 caller에게 반환한다.
2. `ft_split`: pointer table과 여러 field 문자열을 하나의 결과로 묶고, 중간 실패 시 root까지 되돌린다.
3. list lifecycle과 `ft_lstmap`: node와 opaque content의 수명을 분리하고, 새 list 생성 실패 시 current content와 이미 연결된 list를 함께 정리한다.

마지막 failure harness는 이 규칙을 deterministic하게 검사하지만, 실제 diff를 보면 주입 범위에는 중요한 한계가 있다. `ft_split`의 root/field allocation은 모두 주입 대상이지만, `ft_lstmap` callback 안의 `malloc`은 test allocator로 치환되지 않는다. list test가 강제로 실패시키는 대상은 production `ft_lstnew`의 node allocation이다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3b1b30983876` | `feat(alloc): 0 초기화 메모리와 문자열 복제 추가` | A | `SIZE_ARITH`, `OWNERSHIP`, `RISK` | overflow를 확인한 단일 allocation과 caller-owned 반환 계약을 만든다. |
| 2 | `6d076de7185e` | `feat(string): 부분 문자열 생성을 구현` | B | `OWNERSHIP`, `SIZE_ARITH` | source 범위에 맞게 길이를 줄인 독립 substring을 반환한다. |
| 3 | `644b1c65444c` | `feat(string): 문자열 결합을 구현` | B | `OWNERSHIP`, `SIZE_ARITH` | 두 길이와 terminator를 더하기 전에 allocation size를 검증한다. |
| 4 | `8c0a35a50878` | `feat(string): 실패 시 정리되는 문자열 분리 구현` | S | `OWNERSHIP`, `CORE`, `RISK` | root table과 child 문자열을 완성하거나 전부 롤백하는 첫 multi-allocation builder를 만든다. |
| 5 | `7a016ad8fd21` | `feat(list): 연결 리스트 순회와 삭제 구현` | A | `LIST_LIFECYCLE`, `OWNERSHIP`, `RISK` | node 해제와 caller-defined content 해제를 분리한다. |
| 6 | `6672ea67fae4` | `feat(list): 실패 시 정리되는 리스트 변환 구현` | A | `LIST_LIFECYCLE`, `OWNERSHIP`, `RISK` | callback 결과와 새 node를 list에 편입하기 전의 실패 rollback을 구현한다. |
| 7 | `fd3ae063139d` | `test(alloc): 할당 실패와 rollback을 검증` | A | `OWNERSHIP`, `TEST`, `RISK` | production allocator 호출을 선택적으로 실패시키고 tracked allocation과 잘못된 해제를 측정한다. |

## 단일 allocation의 공통 규칙

### `3b1b30983876` — 크기 계산을 allocation보다 먼저 끝낸다

`ft_calloc`은 multiplication을 수행하기 전에 나눗셈으로 overflow 가능성을 확인한다.

```c
if (count != 0 && size > (size_t)-1 / count)
	return (NULL);
allocation_size = count * size;
if (allocation_size == 0)
	allocation_size = 1;
allocation = malloc(allocation_size);
if (allocation == NULL)
	return (NULL);
ft_bzero(allocation, allocation_size);
return (allocation);
```

중요한 순서는 **검증 → 크기 확정 → allocation → 초기화 → 반환**이다.

- `count * size`가 wrap된 작은 값으로 `malloc`에 전달되지 않는다.
- 논리 크기가 0이면 이 구현은 실제 1 byte를 할당하고 0으로 초기화한다.
- 성공한 pointer를 반환하는 순간 해제 책임이 caller에게 넘어간다.
- 실패 시 이 함수가 이미 소유한 다른 자원이 없으므로 `NULL`만 반환하면 된다.

같은 커밋의 `ft_strdup`은 terminator 공간을 포함한 `length + 1`을 계산하기 전에 `length == (size_t)-1`을 거른다. allocation이 성공하면 source의 terminator까지 복사해 source와 독립된 caller-owned 문자열을 만든다.

```c
length = ft_strlen(text);
if (length == (size_t)-1)
	return (NULL);
duplicate = malloc(length + 1);
if (duplicate == NULL)
	return (NULL);
ft_memcpy(duplicate, text, length + 1);
```

### `6d076de7185e` — substring은 가능한 source 범위로 먼저 줄인다

`ft_substr`은 `start`가 문자열 끝을 넘으면 `NULL`이 아니라 독립적으로 `free`할 수 있는 빈 문자열을 반환한다.

```c
text_length = ft_strlen(text);
if ((size_t)start >= text_length)
	return (ft_strdup(""));
if (length > text_length - (size_t)start)
	length = text_length - (size_t)start;
substring = malloc(length + 1);
```

`start < text_length`를 먼저 확정한 뒤 `text_length - start`를 계산하므로 suffix 길이 계산에서 underflow가 생기지 않는다. 요청 길이는 남은 source byte 수로 줄어들고, 정확히 `length + 1` byte를 할당해 마지막 byte를 직접 `'\0'`으로 쓴다.

이 커밋은 반환 allocation이 하나뿐이므로 중간 rollback graph는 아직 없다.

### `644b1c65444c` — 두 길이와 terminator를 한 식으로 검증한다

```c
left_length = ft_strlen(left);
right_length = ft_strlen(right);
if (right_length == (size_t)-1
	|| left_length > (size_t)-2 - right_length)
	return (NULL);
joined = malloc(left_length + right_length + 1);
```

여기서 `(size_t)-2`는 `SIZE_MAX - 1`이다. 따라서 조건은 `left_length + right_length + 1`이 `size_t` 범위를 넘지 않는지 확인한다. 성공 후에는 left의 terminator를 복사하지 않고, right는 `right_length + 1`로 terminator까지 복사한다.

```c
ft_memcpy(joined, left, left_length);
ft_memcpy(joined + left_length, right, right_length + 1);
```

## `8c0a35a50878` — `ft_split`이 complete-or-rollback을 도입하다

`ft_split`은 이전 함수들과 달리 allocation 하나를 반환하지 않는다.

```text
fields root table
├─ fields[0] -> "alpha"
├─ fields[1] -> "beta"
├─ fields[2] -> "gamma"
├─ ...
└─ fields[n] == NULL
```

### 획득 순서

1. `count_fields`로 field 수를 센다.
2. `(field_count + 1)`개의 `char *` slot을 `ft_calloc`으로 만든다.
3. 입력을 다시 순회하며 각 field 문자열을 `copy_field`로 할당한다.
4. 모든 field가 만들어진 경우에만 root `fields`를 caller에게 반환한다.

root가 zero-initialized이므로 아직 채우지 않은 slot과 마지막 terminator slot은 `NULL`이다.

```c
fields = ft_calloc(count_fields(text, delimiter) + 1, sizeof(char *));
if (fields == NULL)
	return (NULL);
```

### 실패 시 되돌리는 범위

```c
static void	free_fields(char **fields, size_t count)
{
	while (count > 0)
	{
		count--;
		free(fields[count]);
	}
	free(fields);
}
```

field allocation이 실패하면 `field_index`는 **성공해 root에 기록된 child 수**다.

```c
fields[field_index] = copy_field(text + start, text_index - start);
if (fields[field_index] == NULL)
	return (free_fields(fields, field_index), NULL);
field_index++;
```

따라서 실패한 slot은 해제 대상에 포함되지 않고, 성공한 `fields[0..field_index)`만 역순으로 해제한 뒤 root를 해제한다. partial `char **`는 caller에게 노출되지 않는다.

| 실패 시점 | 살아 있는 자원 | 정리 |
| --- | --- | --- |
| root table allocation 실패 | 없음 | `NULL` 반환 |
| 첫 field allocation 실패 | root table | root 해제 |
| k번째 field allocation 실패 | root + 앞선 k-1개 field | 성공한 field 전부 + root 해제 |
| 모든 allocation 성공 | 완성된 root + child | caller에게 ownership 이전 |

이 커밋에서 강하게 확인되는 것은 rollback 규칙이다. 다만 `count_fields(text, delimiter) + 1`의 addition 자체에는 별도 overflow guard가 없다. `ft_calloc`은 전달받은 count와 element size의 multiplication을 검사하지만, 그 전에 계산되는 `+ 1`까지 대신 검증하지는 않는다. 따라서 앞선 size-arithmetic 원칙을 이 Thread의 모든 식에 자동으로 일반화해서는 안 된다.

## list는 node와 content의 수명을 분리한다

### `7a016ad8fd21` — 삭제 정책을 callback으로 받는다

`t_list`는 `void *content`를 담으므로 library가 content의 실제 타입이나 해제 방법을 알 수 없다. 이 커밋은 node와 content의 해제를 분리한다.

```c
void	ft_lstdelone(t_list *node, void (*del)(void *))
{
	if (node == NULL || del == NULL)
		return ;
	del(node->content);
	free(node);
}
```

`del`은 content를, library의 `free`는 node를 처리한다. `ft_lstclear`는 삭제 전에 `next`를 보존하고, caller의 head를 매 iteration마다 다음 node로 갱신한다.

```c
while (*list != NULL)
{
	next = (*list)->next;
	ft_lstdelone(*list, del);
	*list = next;
}
```

정상 종료 후 `*list == NULL`이다. 반대로 `list == NULL` 또는 `del == NULL`이면 함수는 아무 node도 해제하지 않는다. `del == NULL`일 때 node만 해제하는 fallback은 이 구현의 정책이 아니다.

### `6672ea67fae4` — callback 결과가 node에 편입되기 전의 ownership

`ft_lstmap`의 iteration마다 자원 상태는 세 단계로 나뉜다.

```c
mapped_content = function(list->content);
node = ft_lstnew(mapped_content);
if (node == NULL)
{
	del(mapped_content);
	ft_lstclear(&mapped, del);
	return (NULL);
}
```

1. `function` 반환 직후: `mapped_content`는 아직 어느 list에도 연결되지 않은 current 자원이다.
2. `ft_lstnew` 성공 후: node가 content pointer를 보유한다.
3. node를 `mapped`에 연결한 후: 완성 중인 mapped list가 node/content pair를 소유한다.

node allocation이 실패하면 current content는 `del(mapped_content)`로 직접 정리한다. 이전 iteration에서 이미 연결된 node와 content는 `ft_lstclear(&mapped, del)`이 정리한다. 둘을 구분하지 않으면 current content가 빠지거나, 반대로 두 번 해제될 수 있다.

성공한 node는 `tail`로 O(1)에 이어 붙이며, source list는 읽기만 한다.

이 함수는 callback이 반환한 `NULL`을 별도의 실패 신호로 해석하지 않는다. `ft_lstnew(NULL)`이 성공하면 `NULL` content를 가진 node가 만들어진다. callback failure 의미와 `NULL` content 허용 여부는 caller의 callback/destructor 정책에 남아 있다.

## `fd3ae063139d` — 실패 하니스가 실제로 주입하는 범위

### production source만 allocator를 치환한다

Makefile은 library의 모든 production source를 별도 object tree로 다시 compile할 때만 다음 macro를 적용한다.

```make
FAIL_DEFINES := -Dmalloc=test_malloc -Dfree=test_free

$(FAIL_OBJ_DIR)/%.o: %.c libft.h tests/support/fail_alloc.h
	$(CC) $(CPPFLAGS) $(FAIL_DEFINES) $(CFLAGS) $(DEPFLAGS) -c $< -o $@

$(FAIL_BIN): $(FAIL_OBJ) $(FAIL_TEST_SRC) tests/support/fail_alloc.h
	$(CC) $(CPPFLAGS) $(CFLAGS) \
		$(FAIL_TEST_SRC) $(FAIL_OBJ) -o $@
```

`FAIL_TEST_SRC`는 link command에서 일반 flags로 compile된다. 따라서 test callback 안의 `malloc`과 `free`는 `test_malloc`/`test_free`로 치환되지 않는다.

### `ft_split`: root와 모든 field 위치를 sweep한다

먼저 `"alpha,beta,gamma,delta"`의 정상 실행에서 allocation 시도 수가 5인지 확인한다.

```text
1: root pointer table
2: "alpha"
3: "beta"
4: "gamma"
5: "delta"
```

이후 failure index 1부터 5까지 각각 다시 실행한다.

```c
while (failure_index <= allocation_count)
{
	test_allocator_reset(failure_index);
	fields = ft_split("alpha,beta,gamma,delta", ',');
	VERIFY(fields == NULL);
	VERIFY(test_allocator_live() == 0);
	VERIFY(test_allocator_invalid_frees() == 0);
	failure_index++;
}
```

이 경로의 allocation과 free는 모두 production object 안에 있으므로 tracker가 root와 field를 전부 관찰한다. 각 위치에서 `NULL`, live allocation 0, invalid free 0을 요구한다.

### `ft_lstmap`: callback allocation이 아니라 node allocation을 실패시킨다

test callback `map_integer`의 allocation은 일반 `malloc`이다. 반면 production `ft_lstnew`는 failure object로 compile되어 node allocation에 `test_malloc`을 사용한다. 정상 실행에서 tracker가 관찰하는 attempt가 3인 이유는 source element 세 개에 대한 **node 세 개**다.

failure index 1부터 3까지의 assertion은 다음을 확인한다.

- `ft_lstmap`은 `NULL`을 반환한다.
- current content와 앞서 연결된 content에 대해 destructor가 정확히 `failure_index`번 호출된다.
- tracked node allocation은 0개 남는다.
- production `test_free` 경로에서 잘못된 node free가 없다.
- stack에 있는 source list와 원본 정수 값은 바뀌지 않는다.

```c
VERIFY(g_content_deletes == (int)failure_index);
VERIFY(test_allocator_live() == 0);
VERIFY(test_allocator_invalid_frees() == 0);
VERIFY(values[0] == 1 && values[1] == 2 && values[2] == 3);
```

따라서 이 test가 증명하는 것은 **각 node allocation 실패 뒤 callback 결과와 이전 mapped list를 정리한다**는 사실이다. callback 자체의 allocation 실패를 주입하거나 callback allocation을 tracker의 live count로 측정하지는 않는다. 이 범위를 “callback과 node의 모든 allocation 위치를 sweep한다”로 확대해서 설명하면 실제 repository보다 강한 주장이 된다.

## 최종 ownership graph

```text
single allocation
  local pointer ──성공 반환──> caller
        └─malloc 실패────────> NULL

ft_split
  local root table
    ├─ child[0]
    ├─ child[1]
    └─ current child allocation 실패
         └─ completed children free → root free → NULL
  모두 성공
    └─ root 전체 ownership → caller

ft_lstmap
  callback current content
    ├─ node allocation 성공 → mapped list에 편입
    └─ node allocation 실패
         ├─ current content: del
         └─ 이전 mapped list: ft_lstclear
  모두 성공
    └─ mapped list root → caller
```

## Thread의 범위

이 Thread는 allocation 크기 계산, root/child rollback, list node/content destructor 정책을 다룬다. callback이 스스로 사용하는 allocator의 실패 의미, `NULL` callback result의 의미, 동시성, project 전체 leak 검사는 별개의 문제다.

> 검토 범위: 표시된 exact SHA의 commit diff와 해당 SHA source/test build를 확인했다. failure binary와 test suite는 실행하지 않았다.
