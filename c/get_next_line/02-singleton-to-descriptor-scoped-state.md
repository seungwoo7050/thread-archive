# Thread: descriptor마다 unread state와 cleanup 책임을 격리하기
> Project: `get_next_line` · Branch: `c/get_next_line` · 문서 번호: 02

## 개요

한 개의 `g_reader`만 사용하는 구현은 fd A의 첫 줄을 반환하고 suffix를 user-space buffer에 남긴 뒤 fd B를 읽는 순간 A의 state를 버립니다. kernel offset은 이미 진행했으므로 버린 suffix를 다시 읽을 수 없고, 이는 단순한 캐시 손실이 아니라 data loss입니다.

이 Thread는 helper가 수정할 state를 인자로 받게 하는 behavior-preserving refactor와, fd별 linked node를 실제로 만드는 architecture change를 구분합니다. 뒤의 test commits는 interleaved 호출, unusable descriptor, allocation/read failure에서 lookup·mutation·cleanup이 선택된 node 하나에만 적용되는지 서로 다른 기법으로 고정합니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fc01012e8521` | `refactor(state): reader 상태를 helper 인자로 전달` | B | `REFACTOR, READER_LIFECYCLE` | helper가 암묵적 singleton 이름 대신 전달받은 state를 수정하게 합니다. |
| 2 | `a4f41cbf2cf0` | `feat(state): 디스크립터별 읽기 상태 분리` | S | `ARCH, READER_LIFECYCLE, RISK` | fd별 buffer와 cursor를 가진 linked reader node를 도입합니다. |
| 3 | `61f8b9858672` | `test(state): 교차 디스크립터 상태 격리 검증` | A | `TEST, READER_LIFECYCLE, RISK` | interleaved stream과 관계없는 invalid fd에서 suffix isolation을 검증합니다. |
| 4 | `d3e2b37fca03` | `test(error): 오류 발생 시 디스크립터 상태 정리 검증` | A | `TEST, READER_LIFECYCLE, RISK` | unusable descriptor cleanup과 fd borrowing이 local한지 검증합니다. |
| 5 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | A | `TEST, POSIX_IO, RISK` | deterministic allocation/read fault에서 node별 owner와 cleanup을 검사합니다. |

### 최종 node 수명은 어떻게 끝나는가

| 결과 | 선택된 node | 다른 node | supplied fd |
| --- | --- | --- | --- |
| line 뒤 suffix가 남음 | buffer와 cursor 유지 | 변화 없음 | caller가 계속 소유 |
| buffer 전부 소비 또는 empty EOF | unlink 후 free | 변화 없음 | 닫지 않음 |
| EOF tail 반환 | bytes를 caller에게 넘긴 뒤 node free | 변화 없음 | 닫지 않음 |
| invalid/read/allocation failure | 선택된 node만 unlink/free | 변화 없음 | 닫지 않음 |

**State 선택을 명시하고 fd별 node로 확장한 커밋**

## fc01012e8521 — refactor(state): reader 상태를 helper 인자로 전달
**중요도** `B` · **태그** `REFACTOR, READER_LIFECYCLE`

이 커밋은 앞뒤의 architecture change와 성격이 다릅니다. 외부 동작이나 state owner를 바꾸지 않고, helper가 어느 reader를 수정하는지 signature에 드러내는 enabling refactor입니다.

### 왜 필요한가

기존 helper는 file-scope `g_reader`를 직접 참조하므로 같은 구현을 여러 state object에 재사용할 수 없습니다. `t_reader *` 또는 `const t_reader *`를 받게 하면 state selection은 상위 caller가 담당하고, reserve·scan·extract·cleanup은 선택된 object 하나만 수정할 수 있습니다.

### 무엇이 바뀌었는가 (diff)

```diff
-static void	reset_reader(void)
+static void	reset_reader(t_reader *reader)
 {
-	free(g_reader.bytes);
-	g_reader.fd = -1;
-	g_reader.bytes = NULL;
+	free(reader->bytes);
+	reader->fd = -1;
+	reader->bytes = NULL;
 	/* ... */
 }
 
-static size_t	unread_length(void)
+static size_t	unread_length(const t_reader *reader)
 {
-	return (g_reader.end - g_reader.begin);
+	return (reader->end - reader->begin);
 }
```

같은 치환이 `compact_bytes`, `reserve_bytes`, `append_bytes`, `find_line_end`, `extract_line`, `release_final_line`과 모든 call site에 적용됩니다.

### production 동작은 왜 그대로인가

entry는 여전히 하나의 global object만 선택합니다.

```diff
 char	*get_next_line(int fd)
 {
+	t_reader	*reader;
+
+	reader = &g_reader;
 	/* ... */
 }
```

따라서 fd를 바꾸면 이전 state를 reset하는 singleton behavior는 그대로입니다. 바뀐 것은 state selection이 아니라 helper와 전역 이름 사이의 결합입니다.

### 관련 커밋

`a4f41cbf2cf0`은 `&g_reader` 고정을 제거하고 `find_reader(fd)` 또는 `create_reader(fd)`가 선택한 node를 같은 helper에 전달해 이 seam을 실제 multi-descriptor architecture로 사용합니다.

## a4f41cbf2cf0 — feat(state): 디스크립터별 읽기 상태 분리
**중요도** `S` · **태그** `ARCH, READER_LIFECYCLE, RISK`

### 무엇을 만들었는가 (diff)

reader에 link가 추가되고 singleton object가 list head로 바뀝니다.

```diff
 	size_t		end;
 	size_t		capacity;
+	struct s_reader	*next;
 }t_reader;
 
-static t_reader	g_reader = {-1, NULL, 0, 0, 0, 0};
+static t_reader	*g_readers;
```

lookup과 create는 fd를 key로 독립 node를 찾거나 완전히 초기화한 뒤 list head에 publish합니다.

```diff
+static t_reader	*find_reader(int fd)
+{
+	t_reader	*reader;
+
+	reader = g_readers;
+	while (reader != NULL && reader->fd != fd)
+		reader = reader->next;
+	return (reader);
+}
+
+static t_reader	*create_reader(int fd)
+{
+	t_reader	*reader;
+
+	reader = malloc(sizeof(*reader));
+	if (reader == NULL)
+		return (NULL);
+	reader->fd = fd;
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
+	reader->next = g_readers;
+	g_readers = reader;
+	return (reader);
+}
```

cleanup은 pointer-to-pointer traversal로 head와 중간 node를 같은 코드로 unlink합니다.

```diff
+static void	discard_reader(t_reader *reader)
+{
+	t_reader	**link;
+
+	link = &g_readers;
+	while (*link != NULL && *link != reader)
+		link = &(*link)->next;
+	if (*link == NULL)
+		return ;
+	*link = reader->next;
+	free(reader->bytes);
+	free(reader);
+}
```

entry는 invalid fd에서도 list 전체를 지우지 않고, 같은 integer fd의 node가 실제로 있을 때만 그 node를 제거합니다.

```diff
 	if (fd < 0 || read(fd, buffer, 0) < 0)
 	{
-		if (fd == reader->fd)
-			reset_reader(reader);
+		reader = find_reader(fd);
+		if (reader != NULL)
+			discard_reader(reader);
 		return (NULL);
 	}
-	/* ... */
+	reader = find_reader(fd);
+	if (reader == NULL)
+		reader = create_reader(fd);
```

### 왜 cleanup이 correctness 조건인가

fd A의 첫 read가 여러 줄을 가져오면 suffix는 A node에 남고 kernel offset은 그 뒤로 진행합니다. 다른 fd 호출 때문에 A node를 버리면 suffix는 복구할 수 없습니다. 반대로 closed fd의 stale node를 남기면 OS가 같은 integer fd를 다른 file에 재사용했을 때 이전 bytes가 새 stream에 섞일 수 있습니다.

이 때문에 node lifecycle은 다음 두 규칙을 동시에 지켜야 합니다.

- 관계없는 call이나 다른 fd failure는 기존 node를 건드리지 않습니다.
- selected fd가 EOF·invalid·read/allocation failure로 끝나면 stale node를 남기지 않습니다.

supplied fd는 node의 key일 뿐 library-owned resource가 아니므로 `discard_reader`는 `close(fd)`를 호출하지 않습니다.

### 아직 다루지 않는 것은 무엇인가

caller는 hidden node를 직접 reset하거나 destroy할 수 없습니다. integer fd와 open file description의 차이, external seek, dup alias, close/reuse를 caller-visible contract로 다루는 작업은 explicit context Thread의 범위입니다. global list에 대한 synchronization도 제공하지 않습니다.

### 관련 커밋

`61f8b9858672`와 `d3e2b37fca03`은 각각 정상 interleaving과 unusable descriptor 경로에서 이 local-state/local-cleanup 결정을 고정합니다. `fd03a831686b`는 allocation/read failure까지 같은 isolation을 확장합니다.

**격리와 failure 경계를 고정한 커밋**

## 61f8b9858672 — test(state): 교차 디스크립터 상태 격리 검증
**중요도** `A` · **태그** `TEST, READER_LIFECYCLE, RISK`

### 무엇을 검증하는가

두 fd를 번갈아 읽어 각 node가 자기 suffix를 유지하는지 확인합니다.

```diff
+	left_expected[0] = "left one\n";
+	left_expected[1] = "left two";
+	right_expected[0] = "right one\n";
+	right_expected[1] = "right two\n";
+	right_expected[2] = "right three";
+	left = reader_for("left one\nleft two", 17);
+	right = reader_for("right one\nright two\nright three", 31);
+	index = 0;
+	while (index < 3)
+	{
+		if (index < 2)
+		{
+			line = get_next_line(left);
+			CHECK(line != NULL);
+			/* ... */
+		}
+		line = get_next_line(right);
+		CHECK(line != NULL);
+		/* ... */
+		index++;
+	}
```

실제 source는 각 반환 pointer를 검사하고 `free`하며, 배열과 loop로 같은 sequence를 구현합니다. 관계없는 invalid fd가 valid node를 지우지 않는 case도 추가됩니다.

```diff
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "first\n") == 0);
+	free(line);
+	CHECK(get_next_line(-1) == NULL);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "second") == 0);
```

### 무엇을 증명하고 무엇을 증명하지 않는가

지정된 두 stream의 interleaving과 invalid `-1` call에서 unread suffix가 섞이거나 사라지지 않음을 증명합니다. concurrent thread access, list race, dup alias에 context를 둘 이상 만드는 misuse는 다루지 않습니다.

### 관련 커밋

`a4f41cbf2cf0`의 fd lookup과 node retention이 production 대상입니다. `d3e2b37fca03`은 정상 interleaving에서 한 단계 더 나아가 read 불가능하거나 닫힌 descriptor의 cleanup이 다른 node와 caller-owned fd에 영향을 주지 않는지 확인합니다.

## d3e2b37fca03 — test(error): 오류 발생 시 디스크립터 상태 정리 검증
**중요도** `A` · **태그** `TEST, READER_LIFECYCLE, RISK`

### 무엇을 검증하는가

pipe write end는 읽을 수 없으므로 `get_next_line`이 `NULL`을 반환해야 하지만, library가 fd 자체를 닫아서는 안 됩니다.

```diff
+static void	test_write_only_descriptor(void)
+{
+	int	fds[2];
+
+	if (pipe(fds) != 0)
+	{
+		CHECK(0);
+		return ;
+	}
+	CHECK(get_next_line(fds[1]) == NULL);
+	CHECK(write(fds[1], "x", 1) == 1);
+	close(fds[0]);
+	close(fds[1]);
+}
```

다른 fixture는 두 fd에 suffix를 남긴 뒤 하나만 닫습니다.

```diff
+	line = get_next_line(closed);
+	CHECK(line != NULL && strcmp(line, "first\n") == 0);
+	free(line);
+	line = get_next_line(kept);
+	CHECK(line != NULL && strcmp(line, "keep\n") == 0);
+	free(line);
+	close(closed);
+	CHECK(get_next_line(closed) == NULL);
+	line = get_next_line(kept);
+	CHECK(line != NULL && strcmp(line, "survive") == 0);
```

### 무엇을 증명하고 무엇을 증명하지 않는가

read 불가능한 fd를 library가 닫지 않고, closed fd의 selected node cleanup이 다른 node를 지우지 않음을 증명합니다. 같은 integer 번호를 실제 새 file에 재할당하는 fixture는 이 commit에 없으며, 그 boundary는 `249093ba477a`의 explicit context test가 담당합니다.

### 관련 커밋

`a4f41cbf2cf0`의 `find_reader(fd)`와 `discard_reader(reader)`가 이 test의 production path입니다. `fd03a831686b`은 OS-level descriptor error 외에 exact allocation/read transition을 주입해 cleanup owner를 더 세밀하게 관찰합니다.

## fd03a831686b — test(failure): 메모리 할당과 읽기 실패 처리 검증
**중요도** `A` · **태그** `TEST, POSIX_IO, RISK`

### 왜 다른 기법이 필요한가

normal file/pipe fixture만으로는 node allocation, buffer growth, result allocation 또는 특정 positive-count `read`를 원하는 순서에서 안정적으로 실패시키기 어렵습니다. 이 commit은 production object를 replacement function과 함께 별도 빌드합니다.

```diff
+FAULT_DEFINES := -Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
+
+$(FAULT_READER_OBJ): get_next_line.c get_next_line.h
+	$(CC) $(FAULT_CPPFLAGS) $(FAULT_DEFINES) $(CFLAGS) \
+		-c $< -o $@
```

allocation baseline의 모든 attempt를 하나씩 실패시키고 live/double/invalid free counter를 확인합니다.

```diff
+	baseline = consume_all(0, &failed);
+	index = 1;
+	while (index <= baseline)
+	{
+		attempts = consume_all(index, &failed);
+		CHECK(failed);
+		CHECK(attempts == index);
+		index++;
+	}
```

한 fd의 read failure가 이미 suffix를 보유한 다른 node를 지우지 않는지도 검사합니다.

```diff
+	line = get_next_line(left);
+	CHECK(line != NULL && strcmp(line, "left one\n") == 0);
+	fault_read_fail_on(right, 1);
+	CHECK(get_next_line(right) == NULL);
+	line = get_next_line(left);
+	CHECK(line != NULL && strcmp(line, "left two") == 0);
```

### 이 baseline이 증명하는 것과 한계는 무엇인가

node object, internal buffer, result allocation의 각 failure에서 leak·double free·invalid free가 없고, selected fd failure가 다른 node를 제거하지 않음을 검증합니다. `fault_read_limit(3)`은 short positive read가 EOF가 아님도 확인합니다.

이 exact SHA의 `test_middle_read_error`는 progress 뒤 EIO가 오면 selected legacy node와 partial bytes를 폐기하는 정책을 기대합니다. `EINTR → progress → EAGAIN` 같은 ordered errno script도 아직 없습니다. accepted bytes를 error 뒤 보존하는 explicit-context 정책과 script harness는 `04-posix-transient-read-and-recovery.md`에서 별도로 설명합니다.

### 관련 커밋

`a4f41cbf2cf0`의 node acquisition/cleanup이 allocation sweep의 직접 대상이고, `61f8b9858672`의 isolation 규칙이 injected read failure에서도 유지되는지 확인합니다. 같은 commit은 POSIX Thread에서 후속 errno semantics를 비교하는 baseline 역할도 합니다.

## 이 Thread의 경계

이 문서는 compatibility API가 fd별 hidden node를 lookup·유지·폐기하는 책임에 한정합니다.

- caller가 context를 직접 create/reset/destroy하고 fd reuse·external seek·dup alias 규칙을 다루는 문제는 `03-explicit-reader-lifetime-and-authoritative-engine.md`가 담당합니다.
- `EINTR`, `EAGAIN`, terminal I/O error 뒤 accepted bytes를 보존하는 정책은 `04-posix-transient-read-and-recovery.md`가 담당합니다.
- global linked list의 concurrent mutation을 안전하게 만드는 synchronization은 별개의 문제이며 이 문서의 commit에는 구현되어 있지 않습니다.

> 검토 범위: `fc01012e8521`, `a4f41cbf2cf0`, `61f8b9858672`, `d3e2b37fca03`, `fd03a831686b`의 diff와 해당 SHA의 관련 source/test harness를 확인했습니다. 저장소 checkout, build, normal test, fault suite, sanitizer는 실행하지 않았습니다.
