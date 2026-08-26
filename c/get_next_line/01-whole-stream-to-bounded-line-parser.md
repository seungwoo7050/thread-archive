# Thread: `read` 분할과 무관하게 한 줄을 반환하고 unread suffix를 보존하기
> Project: `get_next_line` · Branch: `c/get_next_line` · 문서 번호: 01

## 개요

초기 구현은 positive `read`를 EOF까지 이어 붙인 뒤 누적 allocation 전체를 한 번에 반환했습니다. 이 방식은 newline 없는 마지막 조각을 보존하고 allocation failure 시 기존 bytes를 덮어쓰지 않는 기반은 만들었지만, 한 호출에 한 줄만 반환한다는 계약은 아직 충족하지 못했습니다.

이 Thread의 핵심은 buffer를 `[begin, end)` unread window로 바꾸고, 이미 검사한 범위를 기억하는 `scan`을 추가해 kernel이 입력을 어떤 크기로 나누어 주더라도 같은 line sequence를 만드는 것입니다. 이후 commit들은 여러 `BUFFER_SIZE`에서 그 경계를 검증하고, scratch buffer copy를 제거한 뒤 wall-clock 대신 read·allocation·copy 작업량을 고정합니다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85e4c2a41a4c` | `feat(reader): 파일 끝의 마지막 줄 반환` | A | `CORE, LINE_STATE, RISK` | EOF까지 누적하는 geometric buffer와 final allocation의 소유권 이전을 도입합니다. |
| 2 | `7e64d3d79ad4` | `refactor(buffer): 읽지 않은 입력을 구간으로 표현` | S | `ARCH, LINE_STATE, HARD` | active length를 `[begin, end)` unread window로 바꿉니다. |
| 3 | `39a2b9055728` | `feat(reader): 줄을 분리하고 남은 입력 보존` | S | `CORE, LINE_STATE, HARD` | persistent `scan`과 line extraction을 추가해 한 줄씩 소비합니다. |
| 4 | `656528529ade` | `test(reader): BUFFER_SIZE 경계값 검증` | A | `TEST, LINE_STATE, EDGE` | compile-time chunk 크기와 boundary input이 달라도 같은 record를 반환하는지 검증합니다. |
| 5 | `dbf1abd21121` | `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용` | A | `PERF, LINE_STATE, REFACTOR` | stack scratch buffer를 없애고 reserved tail로 직접 읽습니다. |
| 6 | `a0654d9de446` | `test(perf): 4 MiB 입력의 작업량 기준 고정` | A | `PERF, TEST, LINE_STATE` | wall-clock 대신 read·allocation·copy 작업량을 manifest로 고정합니다. |

**Buffer 표현과 line commit을 확립한 커밋**

## 85e4c2a41a4c — feat(reader): 파일 끝의 마지막 줄 반환
**중요도** `A` · **태그** `CORE, LINE_STATE, RISK`

### 무엇을 만들었는가 (diff)

`get_next_line.c`가 새로 생기며 process 전체에 하나인 accumulator와 geometric growth가 도입됩니다.

```diff
+typedef struct s_reader
+{
+	int			fd;
+	char		*bytes;
+	size_t		length;
+	size_t		capacity;
+}t_reader;
+
+static t_reader	g_reader = {-1, NULL, 0, 0};
```

capacity가 부족할 때는 새 allocation을 먼저 얻고 성공한 뒤에만 기존 bytes를 교체합니다.

```diff
+	allocation = malloc(capacity);
+	if (allocation == NULL)
+		return (0);
+	copy_bytes(allocation, g_reader.bytes, g_reader.length);
+	free(g_reader.bytes);
+	g_reader.bytes = allocation;
+	g_reader.capacity = capacity;
```

EOF에서 result를 새로 복사하지 않고 내부 allocation 자체를 caller에게 넘깁니다.

```diff
+static char	*release_final_line(void)
+{
+	char	*line;
+
+	line = g_reader.bytes;
+	g_reader.fd = -1;
+	g_reader.bytes = NULL;
+	g_reader.length = 0;
+	g_reader.capacity = 0;
+	return (line);
+}
```

`line = g_reader.bytes` 뒤 내부 pointer를 `NULL`로 만드는 순서가 소유권 이전의 핵심입니다. 반환값은 caller가 `free`하고, reader는 더 이상 같은 allocation을 해제하지 않습니다.

### 왜 이 구현은 아직 line reader가 아닌가

`get_next_line`은 positive `read`가 계속되는 동안 모든 chunk를 append하고, `read`가 0이 된 뒤에야 `release_final_line`을 호출합니다. 따라서 `"first\nsecond\n"`도 첫 호출에서 하나의 문자열로 반환됩니다. 이 commit이 확립한 것은 다음 단계가 재사용할 growth·overflow 방어·EOF tail·ownership 규칙이지, newline framing 자체가 아닙니다.

### 아직 다루지 않는 것은 무엇인가

- embedded newline을 찾거나 한 줄 뒤 suffix를 남기지 않습니다.
- `g_reader` 하나만 있으므로 fd를 바꾸면 이전 state를 reset합니다.
- result allocation 실패 뒤 같은 line을 재시도하는 의미론은 아직 없습니다.

### 관련 커밋

`7e64d3d79ad4`는 이 accumulator를 버리지 않고, allocation 안에서 이미 소비한 prefix와 아직 반환하지 않은 suffix를 동시에 표현할 수 있도록 상태 표현을 바꿉니다.

## 7e64d3d79ad4 — refactor(buffer): 읽지 않은 입력을 구간으로 표현
**중요도** `S` · **태그** `ARCH, LINE_STATE, HARD`

### 왜 단일 `length`로는 부족한가

전체 stream을 한 번에 반환할 때는 `[0, length)`만 알면 됩니다. 한 줄을 반환하고 나머지를 보존하려면 같은 allocation 안에서 consumed prefix와 unread suffix를 구분해야 하므로 `length`가 `begin`과 `end`로 분해됩니다.

```diff
 typedef struct s_reader
 {
 	int			fd;
 	char		*bytes;
-	size_t		length;
+	size_t		begin;
+	size_t		end;
 	size_t		capacity;
 }t_reader;
```

reserve는 tail 공간 사용, compaction, 새 allocation의 세 경로를 선택합니다.

```diff
+static size_t	unread_length(void)
+{
+	return (g_reader.end - g_reader.begin);
+}
+
+static int	reserve_bytes(size_t appended)
+{
+	/* ... */
+	if (g_reader.capacity - g_reader.end >= appended + 1)
+		return (1);
+	if (g_reader.begin > 0 && required <= g_reader.capacity)
+	{
+		compact_bytes();
+		return (1);
+	}
+	/* ... */
+}
```

새 allocation 경로도 `[begin, end)`만 index 0으로 옮깁니다.

```diff
-	copy_bytes(allocation, g_reader.bytes, g_reader.length);
+	if (length > 0)
+		copy_bytes(allocation, g_reader.bytes + g_reader.begin, length);
 	free(g_reader.bytes);
 	g_reader.bytes = allocation;
+	g_reader.begin = 0;
+	g_reader.end = length;
 	g_reader.capacity = capacity;
+	g_reader.bytes[g_reader.end] = '\0';
```

새 allocation을 확보하기 전에 기존 pointer와 indices를 바꾸지 않으므로 `reserve_bytes` 자체는 실패 시 unread interval을 덮어쓰지 않습니다. 다만 이 시점의 public call은 reserve failure를 받으면 상위에서 reader를 reset하고 `NULL`을 반환하므로, 같은 호출 state를 재시도하는 계약까지 제공하는 것은 아닙니다.

### 무엇을 준비하는가, 아직 없는 것은 무엇인가

이 commit은 consumed prefix를 표현할 수 있는 `begin`을 만들었지만 실제로 `begin`을 line end까지 전진시키는 extractor는 추가하지 않습니다. 관찰 가능한 출력은 여전히 EOF까지 누적한 전체 stream입니다. 즉 representation이 먼저 바뀌고 framing behavior는 `39a2b9055728`에서 완성됩니다.

### 관련 커밋

`39a2b9055728`은 `[begin, end)`에 `scan`을 추가하고, caller-visible allocation을 만든 뒤에만 `begin`을 전진시켜 이 표현을 실제 line parser로 사용합니다.

## 39a2b9055728 — feat(reader): 줄을 분리하고 남은 입력 보존
**중요도** `S` · **태그** `CORE, LINE_STATE, HARD`

### 무엇을 만들었는가 (diff)

`scan`은 unread bytes 중 이미 newline 검사를 끝낸 위치를 기억합니다.

```diff
 	char		*bytes;
 	size_t		begin;
+	size_t		scan;
 	size_t		end;
```

newline을 만나면 newline 다음 index를 exclusive line end로 반환합니다.

```diff
+static size_t	find_line_end(void)
+{
+	while (g_reader.scan < g_reader.end)
+	{
+		if (g_reader.bytes[g_reader.scan] == '\n')
+		{
+			g_reader.scan++;
+			return (g_reader.scan);
+		}
+		g_reader.scan++;
+	}
+	return (0);
+}
```

line result가 실제로 만들어진 뒤에만 unread 시작점을 전진시킵니다.

```diff
+static char	*extract_line(size_t line_end)
+{
+	char	*line;
+	size_t	length;
+
+	length = line_end - g_reader.begin;
+	line = malloc(length + 1);
+	if (line == NULL)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	copy_bytes(line, g_reader.bytes + g_reader.begin, length);
+	line[length] = '\0';
+	g_reader.begin = line_end;
+	g_reader.scan = g_reader.begin;
+	/* ... */
+	return (line);
+}
```

`get_next_line`은 먼저 buffered bytes를 검사하고, 새 chunk를 append할 때마다 새 범위만 검사합니다.

```diff
+	line_end = find_line_end();
+	if (line_end != 0)
+		return (extract_line(line_end));
 	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	while (read_size > 0)
 	{
 		/* ... */
+		line_end = find_line_end();
+		if (line_end != 0)
+			return (extract_line(line_end));
 		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	}
```

### 왜 `begin`과 `scan`을 따로 두는가

`begin`은 caller에게 아직 반환하지 않은 첫 byte이고, `scan`은 그 unread range에서 newline 검사를 끝낸 위치입니다. newline이 없는 chunk 뒤에 새 bytes가 붙어도 `[begin, scan)`을 다시 훑지 않으므로 long line에서 repeated full scan을 피합니다.

```text
[consumed prefix] [begin .... scan | 새로 append된 범위 .... end)
                  unread       다음 검사는 scan부터
```

compaction에서는 `scan -= begin`, growth에서는 unread bytes가 모두 검사된 상태이므로 `scan = length`로 좌표를 다시 맞춥니다.

### 이 SHA에서 allocation failure는 어떤 의미인가

`extract_line`의 `malloc`이 실패하면 `reset_reader()`가 전체 state를 폐기합니다. 따라서 “실패한 result 생성은 input consumption으로 확정하지 않는다”는 재시도 계약은 아직 없습니다. 이 commit에서 확정되는 것은 성공한 allocation 뒤에만 `begin`을 이동한다는 순서와 정상 framing behavior입니다.

### 관련 커밋

`656528529ade`는 이 cursor 규칙이 compile-time chunk 크기나 newline 위치에 따라 달라지지 않는지 검증합니다. allocation failure 뒤 재시도 가능한 commit rule은 `03-explicit-reader-lifetime-and-authoritative-engine.md`의 explicit engine에서 별도로 다룹니다.

**경계와 비용을 고정한 커밋**

## 656528529ade — test(reader): BUFFER_SIZE 경계값 검증
**중요도** `A` · **태그** `TEST, LINE_STATE, EDGE`

### 무엇을 검증하는가

test target을 네 `BUFFER_SIZE`로 다시 빌드해 같은 fixture를 반복합니다.

```diff
+MATRIX_SIZES := 1 2 42 1024
+
+test:
+	@set -e; for size in $(MATRIX_SIZES); do \
+		$(MAKE) --no-print-directory test-run BUFFER_SIZE=$$size; \
+	done
```

`tests/test_boundaries.c`는 chunk 바로 전·정확히 경계·경계 다음 위치의 newline과 EOF tail을 만듭니다.

```diff
+static void	test_chunk_boundaries(void)
+{
+	size_t	chunk;
+
+	chunk = (size_t)BUFFER_SIZE;
+	check_single_line(chunk - 1, 1);
+	check_single_line(chunk, 1);
+	check_single_line(chunk + 1, 1);
+	check_single_line(chunk * 3 + 7, 1);
+	check_single_line(chunk, 0);
+	check_single_line(chunk + 1, 0);
+	check_single_line(chunk * 2 + 1, 0);
+}
```

반환 allocation이 다음 호출 뒤에도 독립적으로 유지되는지도 확인합니다.

```diff
+	first = get_next_line(fd);
+	second = get_next_line(fd);
+	CHECK(first != NULL && strcmp(first, "retained first\n") == 0);
+	CHECK(second != NULL && strcmp(second, "second line\n") == 0);
+	CHECK(first != second);
```

같은 commit에 descriptor 교차 호출과 높은 fd 번호 test도 포함되지만, 이 Thread에서는 line framing과 result storage independence에 직접 연결되는 hunk만 인용했습니다.

### 무엇을 증명하고 무엇을 증명하지 않는가

네 compile-time 크기와 지정된 file/pipe boundary에서 line sequence, newline 포함, EOF tail, repeated EOF가 유지됨을 증명합니다. 모든 kernel short-read pattern을 exhaustive하게 만들거나 out-of-memory behavior를 검증하지는 않습니다.

### 관련 커밋

`39a2b9055728`의 `[begin, scan, end)` 규칙이 이 matrix의 production 대상입니다. `dbf1abd21121`은 같은 observable contract를 유지하면서 chunk를 internal tail로 직접 읽도록 copy 경로만 바꿉니다.

## dbf1abd21121 — refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용
**중요도** `A` · **태그** `PERF, LINE_STATE, REFACTOR`

이 커밋은 앞선 feat와 달리 line semantics를 추가하지 않고, 같은 parser state에 bytes를 넣는 경로만 줄이는 성능 refactor입니다.

### 왜 필요한가

기존 구현은 `read(fd, stack_buffer, BUFFER_SIZE)` 뒤 `append_bytes`가 stack buffer를 internal tail로 다시 복사했습니다. reserve가 이미 writable tail을 보장하므로 kernel이 그 위치에 직접 쓰게 하면 chunk마다 한 번 발생하던 copy를 제거할 수 있습니다.

### 무엇이 바뀌었는가 (diff)

```diff
-static int	append_bytes(t_reader *reader, const char *bytes, size_t length)
-{
-	if (!reserve_bytes(reader, length))
-		return (0);
-	copy_bytes(reader->bytes + reader->end, bytes, length);
-	reader->end += length;
-	reader->bytes[reader->end] = '\0';
-	return (1);
-}
-
 char	*get_next_line(int fd)
 {
-	char	buffer[BUFFER_SIZE];
+	char	probe;
 	/* ... */
-	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
-	while (read_size > 0)
+	while (1)
 	{
-		if (!append_bytes(reader, buffer, (size_t)read_size))
+		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
 		{
 			discard_reader(reader);
 			return (NULL);
 		}
+		read_size = read(fd, reader->bytes + reader->end,
+				(size_t)BUFFER_SIZE);
+		if (read_size <= 0)
+			break ;
+		reader->end += (size_t)read_size;
+		reader->bytes[reader->end] = '\0';
 		/* ... */
 	}
```

reserve가 성공한 뒤에만 buffer pointer를 kernel에 넘기고, positive result에서만 `end`와 sentinel을 publish합니다. `read`가 0 또는 음수이면 capacity는 늘었을 수 있어도 unread interval은 변하지 않습니다.

### 외부 계약은 왜 그대로인가

line search, newline 포함, EOF tail, result ownership은 바뀌지 않습니다. 제거되는 것은 scratch-to-internal copy뿐이며 growth·compaction·result allocation copy는 그대로 남습니다.

### 관련 커밋

`a0654d9de446`은 이 refactor가 per-chunk scratch copy와 linear growth로 되돌아가지 않도록 작업량을 직접 계측합니다.

## a0654d9de446 — test(perf): 4 MiB 입력의 작업량 기준 고정
**중요도** `A` · **태그** `PERF, TEST, LINE_STATE`

### 왜 다른 기법이 필요한가

wall-clock은 filesystem cache와 host 부하에 따라 달라져 repeated scan이나 copy 회귀를 안정적으로 구분하기 어렵습니다. 이 commit은 production object를 계측 wrapper와 함께 빌드해 operation count를 exact manifest와 비교합니다.

```diff
+METRIC_CPPFLAGS := -I. -Itests/metrics -DBUFFER_SIZE=4096
+METRIC_DEFINES := -Dmalloc=metric_malloc -Dfree=metric_free \
+	-Dread=metric_read -DBLR_COPY_OBSERVER=metric_copy_observer
+
+metrics: $(METRIC_BIN)
+	./$(METRIC_BIN) >$(METRIC_ACTUAL)
+	diff -u tests/manifests/metrics-4mib.txt $(METRIC_ACTUAL)
```

`copy_bytes`에는 test build에서만 observer가 연결됩니다.

```diff
+#ifdef BLR_COPY_OBSERVER
+	if (length > 0)
+		BLR_COPY_OBSERVER(length);
+#endif
```

4 MiB newline 없는 입력의 고정 결과는 다음과 같습니다.

```diff
+input_bytes=4194304
+line_bytes=4194304
+checksum=790796585941148453
+read_calls=1025
+read_bytes=4194304
+allocation_calls=13
+allocation_bytes=20963393
+copy_calls=11
+copy_bytes=12533760
```

1025 read calls는 4096-byte positive read 1024회와 EOF read 1회에 해당합니다. 제한된 allocation/copy 횟수는 geometric growth와 direct-tail read가 유지된다는 구조적 증거이고, checksum과 `line_bytes`는 payload가 그대로임을 함께 확인합니다.

### 무엇을 증명하고 무엇을 증명하지 않는가

이 exact 4 MiB single-line workload에서 per-chunk allocation, per-chunk scratch copy, repeated full scan으로 되돌아가는 회귀를 잡습니다. 모든 입력 분포의 최적성이나 다른 플랫폼의 wall-clock 성능은 증명하지 않으며, `wall_ns`는 manifest가 아니라 stderr 참고값으로만 출력됩니다.

### 관련 커밋

`dbf1abd21121`이 제거한 scratch copy와 `7e64d3d79ad4`의 geometric growth가 이 manifest의 직접 대상입니다. newline이 많은 workload나 transient I/O recovery는 이 metric의 범위가 아닙니다.

## 이 Thread의 경계

이 문서는 한 descriptor 안에서 bytes를 frame하고 buffer/cursor 비용을 통제하는 문제만 다룹니다.

- descriptor마다 hidden state를 격리하는 문제는 `02-singleton-to-descriptor-scoped-state.md`가 담당합니다.
- caller-visible context와 result enum, allocation failure 뒤 non-consuming retry는 `03-explicit-reader-lifetime-and-authoritative-engine.md`가 담당합니다.
- `EINTR`, `EAGAIN`, 일반 I/O error 뒤 state 보존은 `04-posix-transient-read-and-recovery.md`가 담당합니다.

> 검토 범위: `85e4c2a41a4c`, `7e64d3d79ad4`, `39a2b9055728`, `656528529ade`, `dbf1abd21121`, `a0654d9de446`의 diff와 해당 SHA의 관련 source/test를 확인했습니다. 저장소 checkout, build, test, sanitizer, metric binary는 실행하지 않았습니다.
