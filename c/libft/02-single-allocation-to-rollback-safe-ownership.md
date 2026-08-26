# Thread: 단일 할당에서 완전 성공 또는 롤백까지

> Project: `libft` · Branch: `c/libft` · 문서 번호: 02

## 개요

이 Thread는 ownership 단위가 하나의 pointer에서 여러 allocation으로 이루어진 object graph로 커질 때도 같은 종료 규칙을 유지하는 과정을 다룬다. 결과가 완성되면 caller에게 이전하고, 중간 획득이 실패하면 partial root를 공개하지 않은 채 local이 가진 자원을 모두 회수한다. history는 크기 계산, graph lifecycle, 실패 관찰이라는 세 역할군으로 나뉜다.

마지막 커밋은 production source의 `malloc`과 `free`만 치환해 실패 위치를 결정적으로 고른다. 따라서 `ft_split`의 root와 field allocation은 모두 주입 대상이지만, test callback 내부의 allocation은 치환되지 않는다. 이 경계는 `ft_lstmap` 회귀가 실제로 무엇을 실패시키는지 해석할 때 중요하다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `3b1b30983876` | `feat(alloc): 0 초기화 메모리와 문자열 복제 추가` | A | `SIZE_ARITH, OWNERSHIP, RISK` | overflow를 검사한 단일 allocation과 caller-owned 반환 계약을 만든다. |
| 2 | `6d076de7185e` | `feat(string): 부분 문자열 생성을 구현` | B | `OWNERSHIP, SIZE_ARITH` | source 범위로 길이를 줄인 독립 substring을 반환한다. |
| 3 | `644b1c65444c` | `feat(string): 문자열 결합을 구현` | B | `OWNERSHIP, SIZE_ARITH` | 두 길이와 terminator를 더하기 전에 allocation 크기를 검증한다. |
| 4 | `8c0a35a50878` | `feat(string): 실패 시 정리되는 문자열 분리 구현` | S | `OWNERSHIP, CORE, RISK` | root table과 child 문자열을 완성하거나 전부 롤백하는 multi-allocation builder를 만든다. |
| 5 | `7a016ad8fd21` | `feat(list): 연결 리스트 순회와 삭제 구현` | A | `LIST_LIFECYCLE, OWNERSHIP, RISK` | node 해제와 caller-defined content 해제를 분리한다. |
| 6 | `6672ea67fae4` | `feat(list): 실패 시 정리되는 리스트 변환 구현` | A | `LIST_LIFECYCLE, OWNERSHIP, RISK` | callback 결과가 node에 편입되기 전의 실패와 이전 list의 rollback을 처리한다. |
| 7 | `fd3ae063139d` | `test(alloc): 할당 실패와 rollback을 검증` | A | `OWNERSHIP, TEST, RISK` | production allocator 호출을 위치별로 실패시키고 live allocation과 잘못된 해제를 측정한다. |

---

**역할군 1 — 단일 allocation에서 크기와 반환 책임을 고정한다.**

## 3b1b30983876 — feat(alloc): 0 초기화 메모리와 문자열 복제 추가
**중요도** `A` · **태그** `SIZE_ARITH, OWNERSHIP, RISK`

### 무엇을 만들었는가 (diff)

`src/alloc/ft_allocate.c`에 두 함수가 추가되고 `libft.h`와 `Makefile`에 공개·build 경계가 연결된다.

```diff
+void	*ft_calloc(size_t count, size_t size)
+{
+	size_t	allocation_size;
+	void	*allocation;
+
+	if (count != 0 && size > (size_t)-1 / count)
+		return (NULL);
+	allocation_size = count * size;
+	if (allocation_size == 0)
+		allocation_size = 1;
+	allocation = malloc(allocation_size);
+	if (allocation == NULL)
+		return (NULL);
+	ft_bzero(allocation, allocation_size);
+	return (allocation);
+}
+
+char	*ft_strdup(const char *text)
+{
+	char	*duplicate;
+	size_t	length;
+
+	if (text == NULL)
+		return (NULL);
+	length = ft_strlen(text);
+	if (length == (size_t)-1)
+		return (NULL);
+	duplicate = malloc(length + 1);
+	if (duplicate == NULL)
+		return (NULL);
+	ft_memcpy(duplicate, text, length + 1);
+	return (duplicate);
+}
```

### 어떤 책임 순서를 확립했는가

`ft_calloc`은 `count * size`를 계산하기 전에 나눗셈으로 multiplication 가능 여부를 확인한다. 논리 크기가 0이면 이 구현은 실제 1 byte를 할당해 0으로 초기화한다. `ft_strdup`도 terminator를 포함한 `length + 1`을 계산하기 전에 guard를 둔다.

두 함수 모두 **크기 검증 → allocation → 초기화 또는 복사 → 반환** 순서다. 성공한 pointer를 반환하는 순간 해제 책임이 caller에게 넘어가며, allocation 전에 보유한 다른 자원이 없으므로 실패는 `NULL` 하나로 끝난다.

### 무엇을 아직 다루지 않는가

반환 object가 하나뿐이어서 중간 child나 partial root를 되돌릴 필요는 없다. 여러 allocation을 하나의 결과로 묶을 때 필요한 rollback graph는 `8c0a35a50878`에서 처음 생긴다.

### 어떤 커밋과 왜 연결되는가

`6d076de7185e`와 `644b1c65444c`는 이 단일-owner 패턴을 다른 문자열 생성 함수에 적용한다. `8c0a35a50878`은 같은 규칙을 root와 여러 child로 확장한다.

## 6d076de7185e — feat(string): 부분 문자열 생성을 구현
**중요도** `B` · **태그** `OWNERSHIP, SIZE_ARITH`

### 무엇이 바뀌었는가 (diff)

`src/string/ft_string_build.c`에 `ft_substr`이 추가된다.

```diff
+char	*ft_substr(const char *text, unsigned int start, size_t length)
+{
+	char	*substring;
+	size_t	text_length;
+
+	if (text == NULL)
+		return (NULL);
+	text_length = ft_strlen(text);
+	if ((size_t)start >= text_length)
+		return (ft_strdup(""));
+	if (length > text_length - (size_t)start)
+		length = text_length - (size_t)start;
+	substring = malloc(length + 1);
+	if (substring == NULL)
+		return (NULL);
+	ft_memcpy(substring, text + start, length);
+	substring[length] = '\0';
+	return (substring);
+}
```

`start`가 문자열 끝 이상이면 `NULL`이 아니라 독립적으로 `free`할 수 있는 빈 문자열을 반환한다. 그 밖에는 `start < text_length`를 먼저 확정한 뒤 `text_length - start`를 계산하고, 요청 길이를 남은 source 범위로 줄인다.

### 무엇을 준비하고 무엇은 아직 없는가

정확히 `length + 1` byte를 할당하고 terminator를 직접 기록하므로 결과는 source와 독립된 caller-owned object다. 하지만 allocation이 하나뿐이어서 성공 이전에 되돌릴 child graph는 없다.

### 어떤 커밋과 왜 연결되는가

`3b1b30983876`의 단일 allocation 반환 규칙을 그대로 사용한다. `644b1c65444c`은 두 입력 길이의 addition까지 검증 범위를 넓히고, `8c0a35a50878`은 여러 결과를 한 root에 묶는다.

## 644b1c65444c — feat(string): 문자열 결합을 구현
**중요도** `B` · **태그** `OWNERSHIP, SIZE_ARITH`

### 무엇이 바뀌었는가 (diff)

같은 `src/string/ft_string_build.c`에 `ft_strjoin`이 이어서 추가된다.

```diff
+char	*ft_strjoin(const char *left, const char *right)
+{
+	char	*joined;
+	size_t	left_length;
+	size_t	right_length;
+
+	if (left == NULL || right == NULL)
+		return (NULL);
+	left_length = ft_strlen(left);
+	right_length = ft_strlen(right);
+	if (right_length == (size_t)-1
+		|| left_length > (size_t)-2 - right_length)
+		return (NULL);
+	joined = malloc(left_length + right_length + 1);
+	if (joined == NULL)
+		return (NULL);
+	ft_memcpy(joined, left, left_length);
+	ft_memcpy(joined + left_length, right, right_length + 1);
+	return (joined);
+}
```

guard는 `left_length + right_length + 1`이 `size_t` 범위 안에 있는지 allocation 전에 확인한다. left는 terminator 직전까지, right는 terminator까지 복사하므로 결과에는 terminator가 정확히 하나 남는다.

### 무엇을 준비하고 무엇은 아직 없는가

두 길이를 결합하는 addition 검증을 추가하지만 결과 allocation은 여전히 하나다. 실패 시 이미 획득한 child가 없으므로 `NULL` 반환 외의 rollback은 필요하지 않다.

### 어떤 커밋과 왜 연결되는가

이 커밋은 `3b1b30983876`의 size-arithmetic 규칙을 addition으로 확장한다. 다음 multi-allocation builder인 `8c0a35a50878`은 크기 계산뿐 아니라 root와 child의 부분 생성 상태까지 관리해야 한다.

---

**역할군 2 — 여러 allocation을 하나의 결과로 묶고 실패 시 되돌린다.**

## 8c0a35a50878 — feat(string): 실패 시 정리되는 문자열 분리 구현
**중요도** `S` · **태그** `OWNERSHIP, CORE, RISK`

### 무엇을 만들었는가 (diff)

`src/string/ft_split.c`가 새 translation unit으로 추가된다.

```diff
+static void	free_fields(char **fields, size_t count)
+{
+	while (count > 0)
+	{
+		count--;
+		free(fields[count]);
+	}
+	free(fields);
+}
+
+char	**ft_split(const char *text, char delimiter)
+{
+	char	**fields;
+	size_t	text_index;
+	size_t	field_index;
+	size_t	start;
+
+	if (text == NULL)
+		return (NULL);
+	fields = ft_calloc(count_fields(text, delimiter) + 1, sizeof(char *));
+	if (fields == NULL)
+		return (NULL);
+	text_index = 0;
+	field_index = 0;
+	while (text[text_index] != '\0')
+	{
+		/* ... */
+		if (text_index > start)
+		{
+			fields[field_index] = copy_field(text + start,
+					text_index - start);
+			if (fields[field_index] == NULL)
+				return (free_fields(fields, field_index), NULL);
+			field_index++;
+		}
+	}
+	return (fields);
+}
```

위 diff는 field 경계 탐색의 반복 부분을 줄였지만, ownership을 결정하는 root allocation, child publish, rollback hunk는 그대로 남겼다.

### 왜 complete-or-rollback이 필요한가

`fields` root는 `field_count + 1`개의 pointer slot을 갖고 `ft_calloc`으로 0 초기화된다. 각 field가 성공할 때마다 `fields[field_index]`에 publish되며, 마지막 미사용 slot은 `NULL` sentinel로 남는다. `field_index`는 **완성되어 root에 편입된 child 수**이므로 다음 child allocation이 실패하면 `fields[0..field_index)`만 해제한 뒤 root를 해제할 수 있다. 실패한 slot은 해제 대상에 포함되지 않고 partial `char **`도 caller에게 노출되지 않는다.

| 실패 시점 | local이 소유한 자원 | 종료 상태 |
| --- | --- | --- |
| root allocation 실패 | 없음 | `NULL` 반환 |
| 첫 field allocation 실패 | root | root 해제 후 `NULL` |
| k번째 field allocation 실패 | root + 앞선 k-1개 field | 성공한 field와 root를 모두 해제 후 `NULL` |
| 모두 성공 | 완성된 root + child | caller에게 ownership 이전 |

### 무엇을 보장하고 무엇은 남기는가

각 child allocation 실패에서 이미 publish된 child와 root가 정확히 한 번씩 정리되는 것을 구현한다. 다만 `count_fields(text, delimiter) + 1`의 addition 자체에는 별도 overflow guard가 없다. `ft_calloc`의 multiplication 검사는 그 전에 계산되는 `+ 1`까지 대신 검증하지 않으므로, 앞선 size-arithmetic 규칙을 이 식에 자동으로 소급해서는 안 된다.

### 어떤 커밋과 왜 연결되는가

`3b1b30983876`이 제공한 checked allocation과 caller-owned 반환을 multi-allocation graph로 확장한다. `fd3ae063139d`는 root allocation과 각 field allocation 위치를 순서대로 실패시켜 rollback을 검증한다.

---

**역할군 3 — node와 opaque content의 수명을 분리해 list 변환에 적용한다.**

## 7a016ad8fd21 — feat(list): 연결 리스트 순회와 삭제 구현
**중요도** `A` · **태그** `LIST_LIFECYCLE, OWNERSHIP, RISK`

### 무엇을 만들었는가 (diff)

`src/list/ft_list_lifecycle.c`가 node/content cleanup API를 추가한다.

```diff
+void	ft_lstdelone(t_list *node, void (*del)(void *))
+{
+	if (node == NULL || del == NULL)
+		return ;
+	del(node->content);
+	free(node);
+}
+
+void	ft_lstclear(t_list **list, void (*del)(void *))
+{
+	t_list	*next;
+
+	if (list == NULL || del == NULL)
+		return ;
+	while (*list != NULL)
+	{
+		next = (*list)->next;
+		ft_lstdelone(*list, del);
+		*list = next;
+	}
+}
```

### 왜 destructor callback이 필요한가

`t_list`는 `void *content`만 보유하므로 library가 content의 concrete type이나 해제 방법을 알 수 없다. caller가 제공한 `del`은 content를 처리하고 library는 node를 `free`한다. `ft_lstclear`는 node를 지우기 전에 `next`를 저장하고 head를 계속 갱신하므로 정상 종료 후 caller의 `*list`는 `NULL`이다.

`list == NULL` 또는 `del == NULL`이면 아무것도 해제하지 않는다. content destructor가 없을 때 node만 해제하는 fallback은 이 구현의 정책이 아니다.

### 어떤 커밋과 왜 연결되는가

`6672ea67fae4`는 이 lifecycle API를 새 list 생성 실패의 rollback primitive로 사용한다. `fd3ae063139d`는 destructor 호출 수와 tracked node가 모두 정리되는지 함께 측정한다.

## 6672ea67fae4 — feat(list): 실패 시 정리되는 리스트 변환 구현
**중요도** `A` · **태그** `LIST_LIFECYCLE, OWNERSHIP, RISK`

### 무엇을 만들었는가 (diff)

`src/list/ft_list_map.c`에 변환 builder가 추가된다.

```diff
+t_list	*ft_lstmap(t_list *list, void *(*function)(void *),
+		void (*del)(void *))
+{
+	t_list	*mapped;
+	t_list	*tail;
+	t_list	*node;
+	void	*mapped_content;
+
+	if (function == NULL || del == NULL)
+		return (NULL);
+	mapped = NULL;
+	tail = NULL;
+	while (list != NULL)
+	{
+		mapped_content = function(list->content);
+		node = ft_lstnew(mapped_content);
+		if (node == NULL)
+		{
+			del(mapped_content);
+			ft_lstclear(&mapped, del);
+			return (NULL);
+		}
+		if (mapped == NULL)
+			mapped = node;
+		else
+			tail->next = node;
+		tail = node;
+		list = list->next;
+	}
+	return (mapped);
+}
```

### ownership은 어느 지점에서 이동하는가

`function`이 반환한 직후의 `mapped_content`는 아직 어느 node에도 편입되지 않은 current 자원이다. `ft_lstnew`가 성공하면 node가 pointer를 보유하고, node를 `mapped`에 연결한 뒤에는 완성 중인 list가 node/content pair를 소유한다.

node allocation이 실패하면 current content는 `del(mapped_content)`로 직접 정리하고, 이전 iteration에서 이미 연결된 list는 `ft_lstclear(&mapped, del)`로 되돌린다. current content와 이전 list를 분리하지 않으면 한쪽이 빠지거나 중복 해제될 수 있다.

### 무엇을 보장하고 무엇은 남기는가

node allocation 실패에서는 current content와 이전 mapped list가 모두 정리되고 partial root가 반환되지 않는다. 반면 callback이 반환한 `NULL`을 별도 failure 신호로 해석하지 않는다. `ft_lstnew(NULL)`이 성공하면 `NULL` content를 가진 node가 만들어지므로 callback failure의 표현은 caller 정책에 남아 있다.

### 어떤 커밋과 왜 연결되는가

`7a016ad8fd21`의 `ft_lstclear`가 이전 iteration의 node/content graph를 되돌리는 핵심 primitive다. `fd3ae063139d`는 production `ft_lstnew` allocation을 위치별로 실패시켜 current destructor와 이전 list cleanup이 함께 실행되는지 확인한다.

---

**역할군 4 — 성공 사례만으로 보이지 않는 rollback을 실패 주입으로 관찰한다.**

## fd3ae063139d — test(alloc): 할당 실패와 rollback을 검증
**중요도** `A` · **태그** `OWNERSHIP, TEST, RISK`

### 왜 다른 기법이 필요한가

정상 입력이 성공하는지만 확인하면 partial root가 실패 경로에서 새는지 알 수 없다. 이 커밋은 production source를 별도 object tree로 다시 compile하면서 `malloc`과 `free`를 tracked substitute로 바꾸고, 지정한 allocation attempt에서만 `NULL`을 반환한다.

```diff
+FAIL_OBJ_DIR := build/failure
+FAIL_OBJ := $(SRC:%.c=$(FAIL_OBJ_DIR)/%.o)
+FAIL_BIN := tests/bin/test_failure
+FAIL_TEST_SRC := tests/failure/test_failure.c tests/support/fail_alloc.c
+FAIL_DEFINES := -Dmalloc=test_malloc -Dfree=test_free
+
+$(FAIL_OBJ_DIR)/%.o: %.c libft.h tests/support/fail_alloc.h
+	$(CC) $(CPPFLAGS) $(FAIL_DEFINES) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(FAIL_BIN): $(FAIL_OBJ) $(FAIL_TEST_SRC) tests/support/fail_alloc.h
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(FAIL_TEST_SRC) $(FAIL_OBJ) -o $@
```

`FAIL_TEST_SRC`는 치환 macro 없이 compile된다. 따라서 test callback의 `malloc`과 `free`는 ordinary allocator를 사용하고, production object 안의 node allocation과 node free만 tracking 대상이 된다.

### failure sweep은 무엇을 확인하는가

`ft_split("alpha,beta,gamma,delta", ',')`의 성공 실행은 root 하나와 field 네 개, 총 다섯 production allocation을 만든다. 테스트는 failure index 1부터 5까지 반복하고 매번 `fields == NULL`, tracked live count 0, invalid free count 0을 요구한다.

```diff
+	test_allocator_reset(0);
+	fields = ft_split("alpha,beta,gamma,delta", ',');
+	allocation_count = test_allocator_attempts();
+	VERIFY(allocation_count == 5);
+	/* ... */
+	while (failure_index <= allocation_count)
+	{
+		test_allocator_reset(failure_index);
+		fields = ft_split("alpha,beta,gamma,delta", ',');
+		VERIFY(fields == NULL);
+		VERIFY(test_allocator_live() == 0);
+		VERIFY(test_allocator_invalid_frees() == 0);
+		failure_index++;
+	}
```

`ft_lstmap` fixture의 `map_integer`는 test source에 있어 content를 ordinary `malloc`으로 만든다. 선택된 failure는 각 iteration의 production `ft_lstnew` node allocation이다. 실패 index가 1, 2, 3일 때 destructor 호출 수가 각각 1, 2, 3인지, tracked node가 0개인지, source stack list가 변하지 않았는지를 확인한다.

```diff
+	while (failure_index <= 3)
+	{
+		test_allocator_reset(failure_index);
+		g_content_deletes = 0;
+		mapped = ft_lstmap(source, map_integer, delete_map_content);
+		VERIFY(mapped == NULL);
+		VERIFY(g_content_deletes == (int)failure_index);
+		VERIFY(test_allocator_live() == 0);
+		VERIFY(test_allocator_invalid_frees() == 0);
+		VERIFY(values[0] == 1 && values[1] == 2 && values[2] == 3);
+		failure_index++;
+	}
```

### 무엇을 증명하고 무엇은 증명하지 않는가

선택된 single-allocation 함수, `ft_split`의 모든 production allocation 위치, `ft_lstmap`의 세 node allocation 위치에서 tracked resource가 남지 않고 invalid free가 발생하지 않음을 증명한다. callback 자체의 allocation 실패, host allocator가 실제로 반환하는 특수 상태, 모든 caller destructor 정책까지 증명하지는 않는다.

### 어떤 커밋과 왜 연결되는가

`8c0a35a50878`의 root/field rollback과 `6672ea67fae4`의 current-content/previous-list rollback을 직접 겨냥한다. `7a016ad8fd21`의 destructor boundary가 정확해야 delete count와 node tracking이 동시에 맞는다.

## 이 Thread의 경계

이 Thread는 allocation 크기 계산, partial root rollback, list node/content destructor 경계를 다룬다. partial `write`와 system call 진행은 [`03-fd-output-partial-system-calls`](03-fd-output-partial-system-calls.md), archive·sanitizer·host leak orchestration은 [`04-static-archive-release-verification`](04-static-archive-release-verification.md)의 관심사다. callback 내부 allocator 실패 의미, shared content, concurrency, project 전체 ownership 정책은 별개의 문제다.

> 검토 범위: `3b1b30983876`, `6d076de7185e`, `644b1c65444c`, `8c0a35a50878`, `7a016ad8fd21`, `6672ea67fae4`, `fd3ae063139d`의 commit diff와 해당 SHA의 allocation/string/list source, failure Makefile rules, `tests/failure/test_failure.c`, `tests/support/fail_alloc.c`를 확인했다. failure binary, ordinary test suite, sanitizer는 실행하지 않았다.
