# Thread: reader 수명과 결과 상태를 caller에게 드러내고 parser engine을 하나로 유지하기
> Project: `get_next_line` · Branch: `c/get_next_line` · 문서 번호: 03

## 개요

fd별 hidden node는 interleaved stream을 지원하지만, caller가 buffered read-ahead를 직접 취소하거나 external `lseek` 뒤 state를 초기화할 방법은 없습니다. `char *` 하나만 반환하는 compatibility API도 EOF와 error를 모두 `NULL`로 축소해 상태를 구분하지 못합니다.

이 Thread는 opaque reader context와 명시적 lifetime, `LINE`·`EOF`·`ERROR` result를 공개합니다. 핵심 architecture 결정은 새 API를 추가하는 데서 끝나지 않고, legacy `get_next_line`을 같은 `blr_reader_next` engine의 adapter로 축소하는 것입니다. 결과 allocation이 실패하면 cursor를 commit하지 않는다는 규칙과 descriptor borrowing 제약은 API test와 fault test로 고정됩니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `903768a43bf4` | `feat(context): 명시적 reader 수명 API 추가` | A | `ARCH, READER_LIFECYCLE, API_CONTRACT` | opaque context와 create/reset/destroy를 공개하고 fd ownership을 caller에게 남깁니다. |
| 2 | `2e681112b304` | `feat(reader): 명시적 결과 상태 API 추가` | S | `ARCH, API_CONTRACT, CORE` | `BLR_LINE`, `BLR_EOF`, `BLR_ERROR`와 stable EOF state를 도입합니다. |
| 3 | `9bd6ebf429e2` | `refactor(reader): legacy API를 context reader에 연결` | A | `REFACTOR, INTEGRATION, API_CONTRACT` | compatibility API를 authoritative context engine의 adapter로 바꿉니다. |
| 4 | `249093ba477a` | `test(context): 결과 상태와 컨텍스트 수명 검증` | A | `TEST, READER_LIFECYCLE, API_CONTRACT` | result state, descriptor borrowing, reset, fd reuse, dup alias를 검증합니다. |
| 5 | `a24ad4e49cc4` | `test(failure): 컨텍스트의 line 할당 재시도 검증` | A | `TEST, READER_LIFECYCLE, RISK` | newline line과 EOF tail allocation failure가 non-consuming인지 검증합니다. |

### 최종 API 결과는 무엇을 뜻하는가

| result | `*line` | context | caller 책임 |
| --- | --- | --- | --- |
| `BLR_LINE` | caller-owned allocation | 다음 unread 위치로 commit | line을 `free`하고 재호출 가능 |
| `BLR_EOF` | `NULL` | stable terminal state | reset 또는 destroy, 반복 호출도 EOF |
| `BLR_ERROR` | `NULL` | explicit context는 파괴되지 않음 | 원인에 따라 retry·reset·destroy 선택 |

context는 object와 internal buffer를 소유하지만 supplied fd는 빌릴 뿐 닫지 않습니다.

**수명과 결과 상태를 공개한 커밋**

## 903768a43bf4 — feat(context): 명시적 reader 수명 API 추가
**중요도** `A` · **태그** `ARCH, READER_LIFECYCLE, API_CONTRACT`

### 무엇을 만들었는가 (diff)

private `t_reader`가 public header에는 layout을 숨긴 opaque type으로 노출됩니다.

```diff
+typedef struct s_blr_reader	t_blr_reader;
+
+t_blr_reader	*blr_reader_create(int fd);
+void			blr_reader_reset(t_blr_reader *reader);
+void			blr_reader_destroy(t_blr_reader *reader);
```

implementation type은 기존 buffer/cursor fields를 그대로 보유합니다.

```diff
-typedef struct s_reader
+struct s_blr_reader
 {
 	int			fd;
 	char		*bytes;
 	size_t		begin;
 	size_t		scan;
 	size_t		end;
 	size_t		capacity;
-	struct s_reader	*next;
-}t_reader;
+	t_blr_reader	*next;
+};
```

create는 zero-length read probe 뒤 context object만 즉시 할당하고, byte buffer는 첫 reserve까지 `NULL`로 둡니다.

```diff
+t_blr_reader	*blr_reader_create(int fd)
+{
+	char			probe;
+	t_blr_reader	*reader;
+
+	if (fd < 0 || read(fd, &probe, 0) < 0)
+		return (NULL);
+	reader = malloc(sizeof(*reader));
+	if (reader == NULL)
+		return (NULL);
+	reader->fd = fd;
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
+	reader->next = NULL;
+	return (reader);
+}
```

reset과 destroy는 owned memory만 다룹니다.

```diff
+void	blr_reader_reset(t_blr_reader *reader)
+{
+	if (reader == NULL)
+		return ;
+	free(reader->bytes);
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
+}
+
+void	blr_reader_destroy(t_blr_reader *reader)
+{
+	if (reader == NULL)
+		return ;
+	free(reader->bytes);
+	free(reader);
+}
```

`reader->fd`를 close하거나 reset하지 않으므로 descriptor lifetime은 caller에게 남습니다. reset은 buffered bytes와 indices만 버리며 kernel offset을 rewind하지 않습니다.

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

legacy hidden list도 `blr_reader_create`와 `blr_reader_destroy`를 재사용하지만, 이 exact SHA에는 context로 다음 line을 요청하는 public operation이 없습니다. caller가 context lifetime을 가질 수 있게 된 중간 단계이고, actual result API는 `2e681112b304`에서 추가됩니다. `reached_eof` field도 아직 존재하지 않습니다.

### 관련 커밋

`2e681112b304`는 이 opaque object에 stable EOF state를 추가하고 `blr_reader_next`를 공개해 context를 실제 reader engine으로 만듭니다.

## 2e681112b304 — feat(reader): 명시적 결과 상태 API 추가
**중요도** `S` · **태그** `ARCH, API_CONTRACT, CORE`

### 무엇을 만들었는가 (diff)

line pointer와 status를 분리하는 enum이 public contract에 들어갑니다.

```diff
+typedef enum e_blr_result
+{
+	BLR_ERROR = -1,
+	BLR_EOF = 0,
+	BLR_LINE = 1
+}t_blr_result;
+
+t_blr_result	blr_reader_next(t_blr_reader *reader, char **line);
```

context는 EOF를 반복 read하지 않도록 terminal state를 기억합니다.

```diff
 	size_t		end;
 	size_t		capacity;
+	int			reached_eof;
 	t_blr_reader	*next;
```

`blr_reader_next`는 non-line result에서 stale pointer가 남지 않도록 먼저 output을 비웁니다.

```diff
+t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
+{
+	if (line != NULL)
+		*line = NULL;
+	if (reader == NULL || line == NULL)
+		return (BLR_ERROR);
+	/* ... */
+}
```

EOF가 한 번 관찰된 뒤 unread bytes가 없으면 새 system call 없이 `BLR_EOF`를 반복 반환합니다.

```diff
+	line_end = find_line_end(reader);
+	if (line_end == 0 && reader->reached_eof)
+	{
+		if (unread_length(reader) == 0)
+			return (BLR_EOF);
+		line_end = reader->end;
+	}
```

result allocation이 실패하면 `begin`과 `end`를 유지하고 `scan`만 unread 시작점으로 되돌립니다.

```diff
+	*line = malloc(length + 1);
+	if (*line == NULL)
+	{
+		reader->scan = reader->begin;
+		return (BLR_ERROR);
+	}
+	copy_bytes(*line, reader->bytes + reader->begin, length);
+	(*line)[length] = '\0';
+	reader->begin = line_end;
+	reader->scan = reader->begin;
+	return (BLR_LINE);
```

caller-visible allocation이 성공한 뒤에만 consumption이 commit되므로 allocation failure는 같은 bytes를 다시 시도할 수 있는 state를 남깁니다.

### 아직 두 engine이 공존하는 이유는 무엇인가

이 commit은 `blr_reader_next`를 추가하지만 기존 `get_next_line`의 read/scan/EOF loop를 제거하지 않습니다. helper를 공유해도 top-level state transition이 둘이면 future fix가 한쪽에만 적용될 위험이 남습니다.

```text
explicit API  -> blr_reader_next
legacy API    -> 기존 get_next_line loop
```

### 관련 커밋

`9bd6ebf429e2`는 legacy loop를 삭제하고 `get_next_line`이 `blr_reader_next` result만 map하도록 바꿔 authoritative engine을 하나로 만듭니다.

**Engine 통합과 API 사용 규칙을 고정한 커밋**

## 9bd6ebf429e2 — refactor(reader): legacy API를 context reader에 연결
**중요도** `A` · **태그** `REFACTOR, INTEGRATION, API_CONTRACT`

### 왜 두 parser를 유지할 수 없는가

EOF tail, allocation failure, future errno mapping이 explicit loop와 legacy loop에서 따로 구현되면 같은 buffer representation을 써도 behavior가 쉽게 갈라집니다. 이 refactor는 외부 API를 없애지 않고 state-transition authority만 `blr_reader_next`로 집중합니다.

### 무엇이 바뀌었는가 (diff)

`get_next_line`의 자체 parser body가 사라지고 adapter만 남습니다.

```diff
 char	*get_next_line(int fd)
 {
-	char		probe;
-	ssize_t	read_size;
-	size_t	line_end;
+	char			*line;
 	t_blr_reader	*reader;
+	t_blr_result	result;
 
 	reader = find_reader(fd);
 	if (reader == NULL)
 		reader = create_legacy_reader(fd);
 	if (reader == NULL)
 		return (NULL);
-	/* ... */
-	return (release_final_line(reader));
+	result = blr_reader_next(reader, &line);
+	if (result == BLR_LINE)
+		return (line);
+	discard_legacy_reader(reader);
+	return (NULL);
 }
```

이 SHA에는 `BLR_AGAIN`이 없으므로 EOF와 ERROR 모두 `NULL`로 축소되고 hidden context는 폐기됩니다.

newline line과 EOF tail은 모두 같은 `extract_line`에서 independent allocation을 만듭니다. 별도 ownership-transfer 함수는 제거됩니다.

```diff
-static char	*release_final_line(t_blr_reader *reader)
-{
-	char	*line;
-	size_t	length;
-
-	length = unread_length(reader);
-	if (reader->begin > 0)
-		copy_bytes(reader->bytes, reader->bytes + reader->begin, length);
-	reader->bytes[length] = '\0';
-	line = reader->bytes;
-	reader->bytes = NULL;
-	discard_legacy_reader(reader);
-	return (line);
-}
-
 	if (reader->reached_eof)
 	{
 		if (unread_length(reader) == 0)
 			return (BLR_EOF);
+		*line = extract_line(reader, reader->end);
+		if (*line == NULL)
+			return (BLR_ERROR);
+		return (BLR_LINE);
 	}
```

`extract_line` allocation failure도 hidden node를 직접 폐기하지 않고 scan만 rewind합니다.

```diff
 	line = malloc(length + 1);
 	if (line == NULL)
 	{
-		discard_legacy_reader(reader);
+		reader->scan = reader->begin;
 		return (NULL);
 	}
```

engine의 state mutation은 non-consuming이지만, legacy adapter는 `BLR_ERROR` 뒤 node를 폐기합니다. “engine이 bytes를 보존한다”와 “특정 adapter가 context를 유지한다”는 서로 다른 결정입니다.

### descriptor borrowing은 어떤 사용 규칙을 만드는가

header comment는 buffered read-ahead와 kernel offset이 결합된다는 점을 명시합니다.

- context는 fd를 빌릴 뿐 닫지 않습니다.
- external seek 뒤에는 caller가 `blr_reader_reset`을 호출해야 합니다.
- context가 살아 있는 동안 fd를 close하고 같은 번호를 재사용하면 기존 context를 쓰지 않습니다.
- 같은 open file description을 공유하는 dup alias에는 context 하나만 사용합니다.
- 서로 다른 context는 별도 thread에서 사용할 수 있지만 같은 context의 concurrent call은 지원하지 않습니다.

### 관련 커밋

`249093ba477a`는 이 header contract를 actual seek, destroy, reuse, dup fixtures로 고정합니다. `a24ad4e49cc4`는 unified engine의 non-consuming extraction을 allocation failure에서 직접 검증합니다.

## 249093ba477a — test(context): 결과 상태와 컨텍스트 수명 검증
**중요도** `A` · **태그** `TEST, READER_LIFECYCLE, API_CONTRACT`

### 무엇을 검증하는가

`"first\nlast"`에서 `LINE`, `LINE`, `EOF`, `EOF` sequence와 non-line output `NULL`을 확인합니다.

```diff
+	check_line(reader, "first\n");
+	check_line(reader, "last");
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
```

external seek와 context reset은 caller가 함께 조정해야 합니다.

```diff
+	check_line(reader, "repeat\n");
+	CHECK(lseek(fd, 0, SEEK_SET) == 0);
+	blr_reader_reset(reader);
+	check_line(reader, "repeat\n");
```

destroy 뒤 fd가 여전히 valid한지 확인하고, 같은 integer fd가 새 open file description을 가리키게 된 경우에는 새 context를 생성합니다.

```diff
+	blr_reader_destroy(reader);
+	CHECK(fcntl(fd, F_GETFD) >= 0);
+	CHECK(lseek(fd, 0, SEEK_SET) == 0);
+	reader = blr_reader_create(fd);
```

```diff
+	close(first);
+	CHECK(dup2(replacement, first) == first);
+	close(replacement);
+	reader = blr_reader_create(first);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "new\n");
+		blr_reader_destroy(reader);
+	}
```

### 무엇을 증명하고 무엇을 증명하지 않는가

stable EOF, EOF/error 구분, reset-after-seek, descriptor borrowing, fd number reuse 시 new context, dup alias 하나에 context 하나를 사용하는 fixture를 증명합니다. 같은 open file description에 context 둘을 만들어 경쟁시키는 behavior나 same-context concurrency를 지원한다는 뜻은 아닙니다.

### 관련 커밋

`903768a43bf4`의 lifetime primitives와 `9bd6ebf429e2`의 header usage rule이 직접 검증 대상입니다. allocation failure에서 state가 실제로 재사용 가능한지는 `a24ad4e49cc4`가 별도로 고정합니다.

## a24ad4e49cc4 — test(failure): 컨텍스트의 line 할당 재시도 검증
**중요도** `A` · **태그** `TEST, READER_LIFECYCLE, RISK`

### 무엇을 검증하는가

newline 하나만 있는 input에서 result allocation을 실패시킨 뒤 같은 context로 다시 호출합니다.

```diff
+	fault_allocation_fail_at(3);
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	CHECK(fault_allocation_failed());
+	fault_allocation_fail_at(0);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "\n") == 0);
+	test_free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
```

EOF tail은 baseline allocation count를 구한 뒤 context object 이후의 각 allocation을 하나씩 실패시키며 재시도합니다.

```diff
+	baseline = consume_context_tail(0, &failed);
+	index = 2;
+	while (index <= baseline)
+	{
+		consume_context_tail(index, &failed);
+		CHECK(failed);
+		index++;
+	}
```

`consume_context_tail`은 allocation fault를 해제한 뒤 같은 reader에서 `"tail"`을 정확히 한 번 받고 EOF까지 진행하는지 검사합니다.

### 무엇을 증명하고 무엇을 증명하지 않는가

newline-delimited line과 EOF tail에서 result allocation failure가 `begin`을 전진시키지 않고, retry가 byte loss나 duplicate 없이 성공함을 증명합니다. 이 commit은 `EINTR`, `EAGAIN`, terminal read error sequence를 다루지 않으며, 그 state transition은 `04-posix-transient-read-and-recovery.md`의 범위입니다.

### 관련 커밋

`2e681112b304`가 도입하고 `9bd6ebf429e2`가 모든 caller에 통합한 “allocation 성공 뒤 cursor commit” 규칙이 이 test의 production target입니다.

## 이 Thread의 경계

이 문서는 caller-visible lifetime/result와 하나의 parser engine을 유지하는 책임에 집중합니다.

- hidden descriptor list를 도입하고 node별 cleanup을 격리하는 문제는 `02-singleton-to-descriptor-scoped-state.md`가 담당합니다.
- `BLR_AGAIN`, `EINTR` retry, terminal read error 뒤 accepted bytes resume는 `04-posix-transient-read-and-recovery.md`가 담당합니다.
- same-context concurrent access를 안전하게 만드는 synchronization은 이 API가 제공하지 않습니다.

> 검토 범위: `903768a43bf4`, `2e681112b304`, `9bd6ebf429e2`, `249093ba477a`, `a24ad4e49cc4`의 diff와 해당 SHA의 header, source, test fixture를 확인했습니다. 저장소 checkout, build, context test, fault suite, sanitizer는 실행하지 않았습니다.
