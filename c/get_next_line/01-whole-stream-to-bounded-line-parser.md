# Thread: 전체 입력 누적에서 bounded streaming line parser까지

이 Thread는 처음에는 입력 전체를 EOF까지 모아 한 번에 반환하던 reader가, 한 번의 호출마다 정확히 한 줄만 반환하고 나머지를 다음 호출에 보존하는 parser로 바뀌는 과정을 다룹니다. 이후에는 stack scratch buffer를 거치지 않고 internal buffer의 빈 tail로 직접 읽으며, 4 MiB 입력에서 read·allocation·copy 작업량을 수치로 고정합니다.

핵심은 세 가지입니다.

1. 누적 중 allocation이 실패해도 기존 bytes를 잃지 않는 성장 방식
2. 이미 반환한 prefix와 아직 읽지 않은 suffix를 구분하는 상태 표현
3. parser의 관찰 가능한 결과가 kernel의 `read` 분할 크기에 좌우되지 않도록 하는 cursor 규칙

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `85e4c2a41a4c` | `feat(reader): 파일 끝의 마지막 줄 반환` | A | `CORE`, `LINE_STATE`, `RISK` | 기하급수적 누적, 실패 시 상태 보존, EOF tail 반환의 기반을 만듭니다. |
| 2 | `7e64d3d79ad4` | `refactor(buffer): 읽지 않은 입력을 구간으로 표현` | S | `ARCH`, `LINE_STATE`, `HARD` | 단일 `length`를 `[begin, end)` unread window로 바꿉니다. |
| 3 | `39a2b9055728` | `feat(reader): 줄을 분리하고 남은 입력 보존` | S | `CORE`, `LINE_STATE`, `HARD` | `scan` cursor와 line extraction을 추가해 한 줄씩 소비합니다. |
| 4 | `656528529ade` | `test(reader): BUFFER_SIZE 경계값 검증` | A | `TEST`, `LINE_STATE`, `EDGE` | 여러 chunk 크기와 경계 입력에서 같은 줄 sequence가 나오는지 검증합니다. |
| 5 | `dbf1abd21121` | `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용` | A | `PERF`, `LINE_STATE`, `REFACTOR` | stack scratch buffer와 append copy를 제거하고 reserved tail로 직접 읽습니다. |
| 6 | `a0654d9de446` | `test(perf): 4 MiB 입력의 작업량 기준 고정` | A | `PERF`, `TEST`, `LINE_STATE` | wall-clock 대신 read·allocation·copy 횟수를 manifest로 고정합니다. |

## 상태 표현의 변화

```text
85e4c2a41a4c
[0 ................................ length) [free capacity]
                     전체 입력을 EOF까지 누적

7e64d3d79ad4
[consumed prefix] [begin ........ end) [free capacity]
                  아직 반환하지 않은 bytes

39a2b9055728
[consumed] [begin .... scan .... end) [free capacity]
           unread     이미 검사한 지점   buffered end
```

최종적으로 유지해야 하는 핵심 조건은 다음과 같습니다.

```text
0 <= begin <= scan <= end < capacity
bytes[end] == '\0'
```

`end < capacity`인 이유는 `end` 위치에 NUL sentinel 한 byte를 보관하기 때문입니다. 실제 입력에는 NUL이 없더라도 internal buffer는 항상 C string으로 검사 가능한 상태를 유지합니다.

---

## `85e4c2a41a4c` — EOF까지 누적한 bytes의 소유권 이전

**중요도** A · **태그** `CORE, LINE_STATE, RISK`

### 처음 만들어진 reader state

이 커밋은 `get_next_line.c`를 추가하고, 하나의 file-scope reader에 현재 descriptor와 누적 buffer를 보관합니다.

```c
/* 85e4c2a41a4c, get_next_line.c */
typedef struct s_reader
{
    int     fd;
    char    *bytes;
    size_t  length;
    size_t  capacity;
} t_reader;

static t_reader g_reader = {-1, NULL, 0, 0};
```

이 시점의 reader는 아직 줄을 찾지 않습니다. `read`가 양수를 반환하는 동안 모든 bytes를 이어 붙이고, 0이 나오면 전체 누적 결과를 반환합니다. 따라서 `"first\nsecond\n"`도 첫 호출에서 하나의 allocation으로 반환됩니다.

### 실패해도 기존 allocation을 덮어쓰지 않는 성장 방식

capacity가 부족하면 새 allocation을 먼저 얻고, 성공한 뒤에만 기존 bytes를 복사하고 이전 allocation을 해제합니다.

```c
allocation = malloc(capacity);
if (allocation == NULL)
    return (0);
copy_bytes(allocation, g_reader.bytes, g_reader.length);
free(g_reader.bytes);
g_reader.bytes = allocation;
g_reader.capacity = capacity;
```

이 순서는 `malloc` 실패 시 기존 pointer와 `length`, `capacity`가 그대로 남도록 합니다. `append_bytes`도 새 길이에 NUL byte 하나를 더한 `required`가 wrap되지 않는지 먼저 검사합니다.

```c
if (length > (size_t)-1 - g_reader.length - 1)
    return (0);
required = g_reader.length + length + 1;
```

capacity는 1에서 시작해 필요한 크기 이상이 될 때까지 두 배씩 증가합니다. 아주 큰 값에서 곱셈 overflow가 임박하면 `required`로 직접 맞춥니다. 매 chunk마다 정확한 크기로 재할당하는 방식과 달리, 이미 누적한 전체 내용을 매번 다시 복사하는 비용을 줄이는 선택입니다.

### EOF tail은 복사가 아니라 ownership transfer다

기존 문서에는 EOF에서 accumulated bytes를 새 result로 “복사한다”는 표현이 있었지만, actual source는 다릅니다.

```c
static char *release_final_line(void)
{
    char *line;

    line = g_reader.bytes;
    g_reader.fd = -1;
    g_reader.bytes = NULL;
    g_reader.length = 0;
    g_reader.capacity = 0;
    return (line);
}
```

reader가 소유하던 allocation을 그대로 caller에게 넘기고, 내부 pointer만 `NULL`로 비웁니다. 따라서 반환값은 caller가 `free`해야 하며, reader는 더 이상 그 allocation을 해제하면 안 됩니다.

```text
반환 직전: g_reader ──owns──> allocation
반환 직후: caller   ──owns──> allocation
           g_reader.bytes == NULL
```

이 소유권 이전 덕분에 EOF에서 전체 입력을 한 번 더 복사하지 않습니다.

### 이 커밋이 보장하는 것

- 여러 positive short read를 EOF까지 누적합니다.
- 빈 입력은 `NULL`을 반환합니다.
- newline 없이 끝나는 nonempty tail도 data로 반환합니다.
- descriptor probe, read, allocation 실패 시 singleton buffer를 정리합니다.
- returned allocation의 owner는 caller입니다.

아직 한 번에 한 줄을 반환하지 않으며, reader state도 descriptor 하나만 보존합니다. 다른 fd로 호출하면 이전 singleton state를 reset합니다.

---

## `7e64d3d79ad4` — unread bytes를 `[begin, end)`로 표현

**중요도** S · **태그** `ARCH, LINE_STATE, HARD`

### 단일 `length`가 부족한 이유

전체 입력만 반환할 때는 `length` 하나로 충분했습니다. 하지만 한 줄만 반환하고 suffix를 남기려면 allocation 안의 bytes가 최소 세 구역으로 나뉩니다.

```text
[이미 반환한 prefix] [다음 호출에 남겨 둘 unread bytes] [아직 쓰지 않은 capacity]
```

이 커밋은 `length`를 `begin`과 `end`로 바꿉니다.

```diff
- size_t length;
+ size_t begin;
+ size_t end;
```

```c
static size_t unread_length(void)
{
    return (g_reader.end - g_reader.begin);
}
```

`begin` 앞의 bytes는 더 이상 의미가 없고, `[begin, end)`만 다음 결과 후보입니다. 아직 line extraction은 없지만, 다음 커밋이 prefix를 소비할 수 있는 상태 표현이 먼저 마련됩니다.

### reserve의 세 경로

`reserve_bytes(appended)`는 “새 allocation이 필요한가”만 검사하지 않습니다. 현재 allocation의 배치를 보고 세 경로 중 하나를 선택합니다.

#### 1. tail 공간이 충분함

```c
if (g_reader.capacity - g_reader.end >= appended + 1)
    return (1);
```

현재 `end` 뒤에 append bytes와 NUL sentinel을 넣을 공간이 있으면 아무것도 옮기지 않습니다.

#### 2. 앞쪽 consumed 공간을 회수하면 충분함

```c
if (g_reader.begin > 0 && required <= g_reader.capacity)
{
    compact_bytes();
    return (1);
}
```

전체 capacity는 충분하지만 tail이 부족한 경우, unread suffix만 allocation 앞쪽으로 당깁니다.

```c
length = unread_length();
copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
g_reader.begin = 0;
g_reader.end = length;
g_reader.bytes[g_reader.end] = '\0';
```

새 allocation 없이 consumed prefix를 재사용합니다.

#### 3. unread bytes 자체가 현재 capacity보다 큼

```c
allocation = malloc(capacity);
if (allocation == NULL)
    return (0);
if (length > 0)
    copy_bytes(allocation, g_reader.bytes + g_reader.begin, length);
free(g_reader.bytes);
g_reader.bytes = allocation;
g_reader.begin = 0;
g_reader.end = length;
g_reader.capacity = capacity;
g_reader.bytes[g_reader.end] = '\0';
```

새 allocation에 **전체 기존 buffer가 아니라 unread interval만** 복사합니다. 새 allocation을 확보하기 전에는 old state를 변경하지 않으므로 allocation failure에서도 `[begin, end)`는 보존됩니다.

### EOF에서 unread interval만 caller에게 넘기기

`begin > 0`인 경우 consumed prefix가 allocation 앞에 남아 있으므로, EOF tail을 ownership transfer하기 전에 unread bytes를 index 0으로 당깁니다.

```c
length = unread_length();
if (g_reader.begin > 0)
    copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
g_reader.bytes[length] = '\0';
line = g_reader.bytes;
```

이 커밋은 아직 `begin`을 전진시키는 line extractor를 추가하지 않았습니다. 따라서 관찰 가능한 출력은 여전히 전체 stream 한 덩어리입니다. 중요도 S인 이유는 기능의 완성이 아니라, 이후 parser가 안전하게 prefix를 소비할 수 있는 durable representation을 만든 데 있습니다.

---

## `39a2b9055728` — persistent scan과 한 줄 단위 commit

**중요도** S · **태그** `CORE, LINE_STATE, HARD`

이 커밋에서 reader가 비로소 “다음 줄”을 반환합니다. 핵심 추가 필드는 `scan`입니다.

```diff
  size_t begin;
+ size_t scan;
  size_t end;
```

### 왜 `scan`이 별도로 필요한가

newline을 아직 찾지 못한 상태에서 다음 read가 append되면, 이미 검사한 prefix를 다시 처음부터 훑을 필요가 없습니다.

```text
begin                  scan          end
  |---------------------|-------------|
       이미 검사함          새로 검사할 범위
```

`find_line_end`는 `scan`부터 시작하고, newline을 만나면 newline 다음 index를 exclusive line end로 반환합니다.

```c
static size_t find_line_end(void)
{
    while (g_reader.scan < g_reader.end)
    {
        if (g_reader.bytes[g_reader.scan] == '\n')
        {
            g_reader.scan++;
            return (g_reader.scan);
        }
        g_reader.scan++;
    }
    return (0);
}
```

newline을 포함해 반환해야 하므로, 발견한 위치에서 `scan++`한 값이 line end가 됩니다.

### extraction의 commit point

줄을 찾았더라도 internal cursor를 먼저 움직이지 않습니다. caller에게 줄을 줄 수 있는 allocation이 성공한 뒤에만 `begin`을 전진시킵니다.

```c
length = line_end - g_reader.begin;
line = malloc(length + 1);
if (line == NULL)
{
    reset_reader();
    return (NULL);
}
copy_bytes(line, g_reader.bytes + g_reader.begin, length);
line[length] = '\0';
g_reader.begin = line_end;
g_reader.scan = g_reader.begin;
```

다만 이 exact SHA에서는 allocation 실패 시 `reset_reader()`를 호출해 buffered state 전체를 버립니다. “allocation 실패 뒤 같은 context로 같은 줄을 재시도할 수 있다”는 보장은 아직 없습니다. 그 non-consuming retry semantics는 나중의 explicit context/authoritative engine Thread에서 확립됩니다.

성공한 경우에는 다음 상태가 됩니다.

```text
before extraction
[consumed] [begin ......... line_end .... end)

allocation + copy 성공 후
[새 consumed prefix] [begin=line_end ... end)
                     scan=begin
```

buffer를 모두 소비했다면 internal allocation을 즉시 free하고 cursor를 0으로 초기화합니다. suffix가 남아 있다면 allocation과 `end`는 그대로 두고 `begin`만 이동합니다.

### compaction과 growth에서 scan 보존

unread bytes를 allocation 앞쪽으로 당기면 기존 absolute index를 그대로 사용할 수 없습니다.

```c
g_reader.scan -= g_reader.begin;
g_reader.begin = 0;
g_reader.end = length;
```

새 allocation으로 성장할 때는 기존 unread bytes가 이미 전부 scan된 상태이므로 다음과 같이 둡니다.

```c
g_reader.begin = 0;
g_reader.scan = length;
g_reader.end = length;
```

그 뒤 append되는 새 bytes만 `find_line_end`가 검사합니다.

### 호출 흐름

```text
get_next_line(fd)
  ├─ 기존 buffered 범위에서 find_line_end
  │    └─ 발견: extract_line 후 즉시 반환
  └─ read 반복
       ├─ chunk append
       ├─ 새 범위만 find_line_end
       │    └─ 발견: extract_line 후 즉시 반환
       ├─ read error: state 폐기, NULL
       └─ EOF
            ├─ unread 없음: state 정리, NULL
            └─ unread 있음: final tail ownership transfer
```

이 구조 때문에 newline이 한 `read` 안에 있든, 두 read 경계에 걸리든, 한 read에 여러 줄이 함께 들어오든 관찰되는 line sequence는 같습니다.

---

## `656528529ade` — BUFFER_SIZE가 바뀌어도 같은 record가 나오는가

**중요도** A · **태그** `TEST, LINE_STATE, EDGE`

이 커밋은 test target을 `BUFFER_SIZE=1, 2, 42, 1024`로 반복 실행하도록 바꾸고, 경계 입력을 집중적으로 추가합니다.

```make
MATRIX_SIZES := 1 2 42 1024

test:
	@set -e; for size in $(MATRIX_SIZES); do \
		$(MAKE) --no-print-directory test-run BUFFER_SIZE=$$size; \
	done
```

### parser와 직접 연결되는 사례

- `BUFFER_SIZE - 1`, 정확히 `BUFFER_SIZE`, `BUFFER_SIZE + 1` 길이에서 newline 포함/미포함
- chunk 세 개를 넘는 긴 한 줄
- 32769-byte line 다음 32771-byte tail
- pipe에서 `"pipe one\npipe two"`
- empty stream과 repeated EOF
- 첫 번째 반환 allocation이 두 번째 호출 뒤에도 내용이 유지되고, 두 pointer가 서로 다른지 확인

특히 반환 storage 독립성 테스트는 internal buffer를 caller와 공유하지 않는다는 점을 검증합니다.

```c
first = get_next_line(fd);
second = get_next_line(fd);
CHECK(strcmp(first, "retained first\n") == 0);
CHECK(first != second);
```

같은 커밋에는 descriptor 교차 호출, 높은 fd 번호 등 다른 Thread의 관심사도 함께 포함되어 있습니다. 이 문서에서는 line framing과 returned allocation independence에 필요한 부분만 사용합니다.

### 증명 범위

이 matrix는 네 compile-time chunk 크기에서 지정된 boundary cases의 결과가 동일함을 검증합니다. 모든 가능한 kernel short-read pattern이나 메모리 오류를 exhaustive하게 증명하지는 않습니다. 그래도 “한 read가 곧 한 줄”이라는 잘못된 구현이나, chunk 경계에서 newline/suffix를 잃는 회귀를 강하게 잡습니다.

---

## `dbf1abd21121` — scratch buffer를 없애고 reserved tail로 직접 읽기

**중요도** A · **태그** `PERF, LINE_STATE, REFACTOR`

기존 경로는 다음과 같았습니다.

```text
read(fd, stack_buffer, BUFFER_SIZE)
    → reserve internal buffer
    → stack_buffer에서 internal tail로 copy
    → end 증가
```

이 커밋은 `append_bytes`와 stack `buffer[BUFFER_SIZE]`를 제거합니다. 먼저 internal buffer에 `BUFFER_SIZE + NUL` 공간을 확보한 뒤, `end` 위치로 직접 읽습니다.

```c
while (1)
{
    if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
    {
        discard_reader(reader);
        return (NULL);
    }
    read_size = read(fd, reader->bytes + reader->end,
            (size_t)BUFFER_SIZE);
    if (read_size <= 0)
        break ;
    reader->end += (size_t)read_size;
    reader->bytes[reader->end] = '\0';
    line_end = find_line_end(reader);
    if (line_end != 0)
        return (extract_line(reader, line_end));
}
```

### ordering이 중요한 이유

- reserve가 먼저 성공해야 kernel에 destination을 넘길 수 있습니다.
- `read_size > 0`인 경우에만 `end`를 증가시킵니다.
- `read`가 0이나 음수를 반환하면 reserved capacity는 늘어났을 수 있지만 unread interval은 바뀌지 않습니다.
- NUL sentinel은 positive progress를 state에 publish한 직후 새 `end`에 씁니다.

제거되는 것은 **매 read마다 발생하던 scratch-to-internal copy**입니다. buffer growth나 compaction, caller result allocation을 위한 copy까지 사라지는 것은 아닙니다.

---

## `a0654d9de446` — 실행 시간 대신 작업량을 고정

**중요도** A · **태그** `PERF, TEST, LINE_STATE`

이 커밋은 4 MiB의 newline 없는 입력을 `BUFFER_SIZE=4096`으로 읽고, production source의 `malloc`, `free`, `read`, `copy_bytes`를 계측합니다. stdout은 manifest와 exact diff하고, wall-clock은 stderr의 참고 정보로만 남깁니다.

고정된 manifest는 다음과 같습니다.

```text
input_bytes=4194304
line_bytes=4194304
checksum=790796585941148453
read_calls=1025
read_bytes=4194304
allocation_calls=13
allocation_bytes=20963393
copy_calls=11
copy_bytes=12533760
```

### 숫자가 의미하는 것

- `read_calls=1025`: 4096-byte positive read 1024회와 마지막 EOF read 1회입니다.
- `read_bytes=4194304`: 입력을 중복해서 읽지 않았습니다.
- 제한된 allocation/copy 횟수: geometric growth가 유지되며 매 chunk마다 새 allocation을 만들지 않습니다.
- `line_bytes`와 checksum: 작업량만 줄인 것이 아니라 전체 payload도 정확히 보존합니다.

`wall_ns`는 manifest에 들어가지 않습니다. CPU, filesystem cache, VM 부하에 따라 달라지는 시간값으로 회귀를 판정하지 않고, 구현 선택에 직접 연결되는 operation count를 판정 기준으로 삼은 것입니다.

### 증명하지 않는 것

- 모든 입력 분포에서 최적이라는 사실
- 다른 플랫폼에서도 똑같은 wall-clock 성능
- newline이 매우 많은 workload의 비용
- allocation 실패나 nonblocking read의 recovery

이 테스트가 고정하는 것은 “4 MiB 한 줄에서 repeated full scan, per-chunk allocation, per-chunk scratch copy 같은 곱셈적 작업이 다시 생기지 않는다”는 구조적 특성입니다.

---

## 최종 parser의 불변 조건

| 대상 | 최종 규칙 | 코드상 의미 |
| --- | --- | --- |
| unread bytes | `[begin, end)` | caller에게 아직 반환하지 않은 입력만 포함합니다. |
| scan progress | `[begin, scan)`은 이미 검사됨 | 새 read 뒤 기존 prefix를 다시 훑지 않습니다. |
| sentinel | `bytes[end] == '\0'` | buffered range를 안전하게 문자열로 다룰 수 있습니다. |
| line result | `[begin, line_end)`의 독립 allocation | newline이 있으면 포함하고 caller가 `free`합니다. |
| cursor commit | result allocation/copy 성공 뒤 `begin` 이동 | 성공한 결과만 소비로 확정합니다. 다만 이 재시도 보장은 later authoritative engine에서 완성됩니다. |
| EOF tail | unread nonempty suffix를 한 번 반환 | 빈 stream과 unterminated tail을 구분합니다. |
| growth | unread bytes만 geometric allocation으로 이동 | 실패 전 기존 unread state를 덮어쓰지 않습니다. |

## 이 Thread의 경계

이 Thread는 한 descriptor 안에서 bytes를 줄로 frame하고, buffer와 cursor의 비용을 통제하는 문제만 다룹니다.

- 여러 descriptor의 state를 동시에 보존하는 방법은 `02-singleton-to-descriptor-scoped-state.md`에서 다룹니다.
- caller가 context lifetime과 result enum을 직접 다루는 방법은 `03-explicit-reader-lifetime-and-authoritative-engine.md`에서 다룹니다.
- `EINTR`, `EAGAIN`, terminal read error 뒤 state 보존은 `04-posix-transient-read-and-recovery.md`에서 다룹니다.

> 검증 메모: 표시된 exact SHA의 GitHub diff와 source를 확인했습니다. 이 환경에서는 checkout/build/test를 실행하지 않았습니다.
