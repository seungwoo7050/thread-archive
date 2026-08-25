# Thread: POSIX transient read와 복구 가능한 parser state

streaming reader는 `read`의 반환값을 단순히 “성공/실패”로만 나눌 수 없습니다.

- 양수는 EOF가 아니라 progress입니다. 요청한 크기보다 작아도 정상입니다.
- 0은 현재 stream의 EOF입니다.
- `-1/EINTR`는 logical operation이 끝난 것이 아니므로 같은 read를 다시 시도해야 합니다.
- `-1/EAGAIN` 또는 `-1/EWOULDBLOCK`은 nonblocking fd에 현재 data가 부족하다는 뜻이며, 이미 받은 bytes를 보존한 채 caller에게 wait 상태를 알려야 합니다.
- 그 밖의 `-1/error`는 terminal result로 보고하되 explicit context가 이미 받아들인 bytes를 임의로 소비하거나 지우면 안 됩니다.

이 Thread는 이러한 POSIX 결과를 parser state transition으로 바꾸는 과정과, 실제 nonblocking pipe·deterministic fault script가 각각 무엇을 검증하는지 다룹니다.

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | A | `TEST`, `POSIX_IO`, `RISK` | short-read와 지정 호출 read error를 재현하는 baseline harness를 제공합니다. |
| 2 | `f0055ae5cf19` | `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존` | S | `CORE`, `POSIX_IO`, `RISK` | `EINTR` retry, `BLR_AGAIN`, hidden wait-state retention을 도입합니다. |
| 3 | `f3504f674c73` | `test(reader): 비차단 부분 입력 보존 검증` | A | `TEST`, `POSIX_IO`, `EDGE` | 실제 nonblocking pipe에서 partial input → wait → resume → EOF를 검증합니다. |
| 4 | `11033bd85c59` | `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증` | A | `TEST`, `POSIX_IO`, `RISK` | errno sequence를 순서대로 주입해 progress와 failure가 섞인 cursor 변화를 고정합니다. |

## 최종 read transition 표

| `read` 결과 | parser 해석 | `end` 변화 | unread bytes | explicit result | legacy `get_next_line` |
| --- | --- | --- | --- | --- | --- |
| `n > 0` | progress | `end += n` 후 sentinel | 보존 + 새 bytes 추가 | line이 생길 때까지 계속 | 같은 engine 계속 |
| `0` | EOF | 변화 없음 | tail이 있으면 line, 없으면 terminal | `BLR_LINE` 또는 `BLR_EOF` | line 또는 `NULL` |
| `-1`, `EINTR` | operation 미완료 | 변화 없음 | 그대로 | 내부 retry, caller에게 보이지 않음 | 내부 retry |
| `-1`, `EAGAIN/EWOULDBLOCK` | 현재 data 미준비 | 변화 없음 | 그대로 | `BLR_AGAIN` | `NULL`, hidden context 유지 |
| `-1`, 기타 errno | I/O error | 변화 없음 | explicit context에 그대로 남음 | `BLR_ERROR` | `NULL`, hidden context 폐기 |

non-line result에서는 `*line == NULL`입니다.

---

## `fd03a831686b` — short read와 단일 read failure를 재현하는 기반

**중요도** A · **태그** `TEST, POSIX_IO, RISK`

이 커밋의 fault binary는 production source를 다음 replacement로 컴파일합니다.

```make
-Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
```

POSIX transition 관점에서 중요한 기능은 두 가지입니다.

### positive read의 최대 길이 제한

```c
void fault_read_limit(size_t limit)
{
    g_read_limit = limit;
}
```

`test_read`는 target fd의 request count를 이 limit 이하로 줄인 뒤 실제 `read`를 호출합니다. 따라서 bytes 자체는 실제 pipe/file에서 오고, harness는 kernel이 작은 positive read를 반환한 것처럼 만듭니다.

`"short reads still work\nlast"`를 최대 3 byte씩 읽어도 다음 sequence가 나와야 합니다.

```text
"short reads still work\n"
"last"
NULL
```

short positive return을 EOF로 오해하면 첫 호출이 incomplete record를 반환하거나 suffix를 잃게 됩니다.

### N번째 positive-count read를 EIO로 실패

```c
void fault_read_fail_on(int fd, size_t call_index)
```

이 API는 선택한 한 번의 호출에 `errno=EIO`, `-1`을 반환합니다. zero-length descriptor probe는 counter 대상이 아니며, 실제 data read 호출만 셉니다.

- 첫 read failure: progress가 전혀 없는 상태에서 error cleanup 확인
- 중간 read failure: 여러 short positive reads 뒤 error가 왔을 때 당시 policy 확인
- 한 fd failure: 다른 descriptor의 node가 살아 있는지 확인

### 이 baseline의 한계

이 exact SHA의 legacy implementation은 중간 read error가 오면 selected node와 accepted partial bytes를 **폐기**합니다. fixture 이름도 `"partial bytes must be discarded"`입니다.

또한 harness는 아직 `EINTR`, normal progress, `EAGAIN`을 순서대로 배열에 담을 수 없습니다. 가능한 것은 다음뿐입니다.

```text
- request size cap
- 지정한 한 call index에서 EIO
```

따라서 후속 commit의 `BLR_AGAIN`이나 errno sequence behavior를 이 커밋이 이미 검증했다고 설명하면 안 됩니다. 이 commit은 read fault infrastructure와 이전 policy의 기준점입니다.

---

## `f0055ae5cf19` — interruption, wait, terminal error를 분리

**중요도** S · **태그** `CORE, POSIX_IO, RISK`

### `EINTR`는 result가 아니라 retry 신호다

signal handler 실행 등으로 `read`가 중단되면 `-1/EINTR`가 올 수 있습니다. parser가 이를 `BLR_ERROR`로 노출하면 실제 stream 상태와 무관한 transient event가 caller-visible failure가 됩니다.

이 커밋은 모든 descriptor probe와 data read를 `read_retrying`으로 통일합니다.

```c
static ssize_t read_retrying(int fd, void *buffer, size_t count)
{
    ssize_t read_size;

    read_size = read(fd, buffer, count);
    while (read_size < 0 && errno == EINTR)
        read_size = read(fd, buffer, count);
    return (read_size);
}
```

`EINTR` 동안에는 buffer cursor를 전혀 바꾸지 않고 동일한 logical operation을 반복합니다. 결국 positive/zero/다른 error가 나왔을 때만 상위 state machine이 결과를 해석합니다.

zero-length fd probe도 같은 wrapper를 사용하므로 create나 next 진입 시 발생한 `EINTR` 역시 caller에게 보이지 않습니다.

### temporary unavailability를 `BLR_AGAIN`으로 표현

```diff
 typedef enum e_blr_result
 {
     BLR_ERROR = -1,
     BLR_EOF = 0,
-    BLR_LINE = 1
+    BLR_LINE = 1,
+    BLR_AGAIN = 2
 } t_blr_result;
```

read loop가 음수로 끝났을 때 errno를 분류합니다.

```c
if (read_size < 0)
{
    if (errno == EAGAIN || errno == EWOULDBLOCK)
        return (BLR_AGAIN);
    return (BLR_ERROR);
}
```

`BLR_AGAIN`은 EOF가 아닙니다. 현재 buffered bytes에 완전한 줄이 없고, nonblocking fd에서도 당장 더 받을 data가 없다는 뜻입니다.

### accepted bytes가 보존되는 이유

positive read에서만 `end`를 전진시킵니다.

```c
read_size = read_retrying(reader->fd, reader->bytes + reader->end,
        (size_t)BUFFER_SIZE);
if (read_size <= 0)
    break ;
reader->end += (size_t)read_size;
reader->bytes[reader->end] = '\0';
```

따라서 `EAGAIN`이나 일반 I/O error가 발생한 call은 `end`를 바꾸지 않습니다. 그 전에 성공한 positive read가 있다면 이미 `end` 안쪽에 publish된 bytes가 남습니다.

newline을 찾지 못해 `scan == end`가 되었더라도 다음 positive read는 같은 allocation tail에 append되고, `find_line_end`는 이전 `scan`부터 새 범위만 검사합니다.

```text
처음: begin=0, scan=0, end=0
read "par" -> begin=0, scan=3, end=3
EAGAIN     -> begin=0, scan=3, end=3, BLR_AGAIN
read "tial\n" -> end 증가, scan 3부터 재개
```

이 diff가 “error에서 buffer reset을 제거”한 것은 아닙니다. authoritative explicit engine은 이미 read error에서 context를 파괴하지 않고 `BLR_ERROR`를 반환하는 형태였습니다. 이 커밋은 그 보존 특성을 errno-aware state policy로 명시하고, transient wait를 terminal error와 분리합니다.

### explicit API와 compatibility adapter의 차이

explicit caller는 result를 직접 봅니다.

```text
BLR_AGAIN -> input readiness를 기다린 뒤 같은 context 재호출
BLR_ERROR -> 원인을 처리한 뒤 같은 context 재시도, reset, destroy 중 선택
```

legacy `get_next_line`은 return type이 `char *`이므로 `BLR_AGAIN`을 `NULL`로 축소합니다. 다만 hidden node를 제거하지 않습니다.

```c
result = blr_reader_next(reader, &line);
if (result == BLR_LINE)
    return (line);
if (result != BLR_AGAIN)
    discard_legacy_reader(reader);
return (NULL);
```

결과는 다음과 같습니다.

| engine result | legacy return | hidden context |
| --- | --- | --- |
| `BLR_LINE` | line | 유지 |
| `BLR_AGAIN` | `NULL` | 유지 |
| `BLR_EOF` | `NULL` | 폐기 |
| `BLR_ERROR` | `NULL` | 폐기 |

따라서 compatibility API는 nonblocking wait 뒤에는 resume할 수 있지만, terminal `BLR_ERROR` 뒤 accepted partial bytes를 다시 꺼낼 수는 없습니다. explicit context만 그 선택권을 보존합니다.

### 보장 범위

- `EINTR`는 무한정 같은 read를 retry합니다.
- `EAGAIN`과 `EWOULDBLOCK`은 `BLR_AGAIN`으로 분리됩니다.
- non-line result는 line pointer를 `NULL`로 둡니다.
- accepted positive bytes는 AGAIN/ERROR 때문에 cursor에서 제거되지 않습니다.
- legacy hidden context는 AGAIN에서만 유지됩니다.

계속 `EINTR`만 발생하는 비정상 환경에서 retry가 끝난다는 상한은 없습니다. readiness notification이나 polling 자체도 library가 제공하지 않습니다.

---

## `f3504f674c73` — 실제 nonblocking pipe에서 wait와 resume 확인

**중요도** A · **태그** `TEST, POSIX_IO, EDGE`

fault replacement가 아니라 실제 pipe read end에 `O_NONBLOCK`을 설정합니다.

```c
flags = fcntl(fds[0], F_GETFL);
fcntl(fds[0], F_SETFL, flags | O_NONBLOCK);
```

### explicit context sequence

#### 1. delimiter 없는 partial input

writer가 `"part"`만 씁니다.

```c
CHECK(write(fds[1], "part", 4) == 4);
CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
CHECK(line == NULL);
```

reader는 네 bytes를 받아 internal buffer에 넣지만 newline이 없고, 다음 nonblocking read는 `EAGAIN`입니다.

#### 2. 나머지 bytes가 도착해 첫 줄 완성

```c
CHECK(write(fds[1], "ial\nnext", 8) == 8);
CHECK(blr_reader_next(reader, &line) == BLR_LINE);
CHECK(strcmp(line, "partial\n") == 0);
```

이전에 보존한 `"part"`와 새 `"ial\n"`가 합쳐져야 합니다. 새 write에 함께 들어온 `"next"`는 unread suffix로 남습니다.

#### 3. suffix는 있지만 아직 EOF가 아님

writer가 열린 상태에서 `"next"` 뒤 newline이 없으므로 다음 호출은 `BLR_AGAIN`입니다. suffix를 EOF tail처럼 조기에 반환하면 안 됩니다.

#### 4. writer close가 EOF boundary를 확정

writer를 닫은 뒤 다음 read가 0을 반환합니다. 그때 buffered `"next"`가 EOF tail `BLR_LINE`으로 반환되고, 다음 호출은 `BLR_EOF`입니다.

```text
part -> AGAIN
+ ial\nnext -> LINE "partial\n"
next only, writer open -> AGAIN
writer close -> LINE "next"
next call -> EOF
```

### compatibility wrapper sequence

```text
write "leg"
get_next_line -> NULL      # AGAIN을 표현할 수 없음, state는 유지
write "acy\n"
get_next_line -> "legacy\n"
```

legacy caller는 첫 `NULL`이 EOF인지 wait인지 return value만으로 구분할 수 없습니다. 하지만 adapter가 hidden context를 유지했기 때문에 두 번째 호출에서 partial bytes가 복원됩니다.

### 이 test가 증명하는 것

- 실제 OS nonblocking pipe에서 `EAGAIN`이 발생하는 경로
- partial bytes가 write call 사이에 보존됨
- writer가 열린 상태의 unterminated suffix는 line으로 조기 반환되지 않음
- writer close 후 suffix와 stable EOF가 순서대로 나옴
- compatibility adapter가 AGAIN state를 보존함

실제 signal delivery로 `EINTR`를 만드는 test는 아닙니다. 특정 errno ordering이나 terminal EIO 뒤 resume도 다음 deterministic test가 맡습니다.

---

## `11033bd85c59` — progress와 errno를 순서대로 주입

**중요도** A · **태그** `TEST, POSIX_IO, RISK`

기존 “N번째 read 하나를 EIO로 실패”하던 harness를 per-call errno script로 확장합니다.

```c
#define MAX_READ_SCRIPT 64

static int    g_read_script[MAX_READ_SCRIPT];
static size_t g_read_script_length;
```

각 script entry는 다음 의미입니다.

```text
0          -> 실제 read 수행
nonzero    -> errno를 그 값으로 설정하고 -1 반환
```

positive result의 bytes는 여전히 실제 fd에서 오며, `fault_read_limit`으로 최대 길이를 제한할 수 있습니다.

### `EINTR → progress → EAGAIN`

```c
const int errors[] = {EINTR, 0, EAGAIN};
fault_read_limit(3);
fault_read_script(fd, errors, 3);
```

첫 `blr_reader_next` 안의 실제 sequence는 다음과 같습니다.

```text
read call #1 -> -1/EINTR
read_retrying이 즉시 재호출
read call #2 -> 실제 3-byte progress
parser end += 3, scan은 새 bytes 끝까지 진행
read call #3 -> -1/EAGAIN
blr_reader_next -> BLR_AGAIN, line=NULL
```

assertion은 첫 API call 동안 read counter가 3인지도 확인합니다. 이는 `EINTR`가 caller에게 결과로 노출되지 않고 wrapper 안에서 retry됐음을 보여 줍니다.

fault script가 끝난 뒤 같은 context를 호출하면 나머지 실제 input을 읽어 다음 sequence가 나옵니다.

```text
BLR_LINE "partial\n"
BLR_LINE "last"
BLR_EOF
```

초기 3 bytes를 버리거나 다시 읽으면 첫 line 비교가 실패합니다.

### `progress → EIO → retry`

```c
const int errors[] = {0, EIO};
fault_read_limit(3);
```

첫 read는 3 bytes를 받아들인 뒤 두 번째 read가 EIO입니다.

```text
1차 next -> BLR_ERROR, line=NULL
           accepted 3 bytes는 context에 유지
2차 next -> 남은 input과 결합해 BLR_LINE "recoverable\n"
3차 next -> BLR_EOF
```

이 test는 terminal error를 “EOF”로 바꾸지 않고, 이미 accepted한 bytes를 context reset 없이 재사용한다는 점을 고정합니다.

### cursor 관점의 trace

```text
input: recoverable\n
limit: 3

normal read "rec"
  begin=0, end=3
  find_line_end -> scan=3

EIO
  begin=0, scan=3, end=3
  BLR_ERROR

retry normal read "ove" ...
  end 증가
  scan은 3부터 진행

newline 발견
  result allocation 성공
  begin=line_end, scan=begin
```

`end`를 EIO call 전에 미리 늘리거나, error에서 `begin/end`를 reset하거나, retry에서 scan을 0으로 잘못 다루면 이 ordered regression이 결과 mismatch로 드러납니다.

### 증명 범위와 한계

이 커밋은 지정된 두 sequence에서 다음을 증명합니다.

- `EINTR`가 wrapper 안에서 retry됨
- positive progress 뒤 `EAGAIN`이 wait state가 됨
- positive progress 뒤 EIO가 `BLR_ERROR`가 되지만 accepted bytes는 남음
- retry에서 byte duplicate/skip 없이 complete line과 tail을 반환함

다음을 exhaustive하게 증명하지는 않습니다.

- 모든 POSIX errno
- 실제 asynchronous signal handler가 만든 EINTR timing
- script 64개를 넘는 sequence
- compatibility wrapper가 terminal error 뒤 state를 보존함
- multi-threaded concurrent calls

---

## failure와 state mutation의 순서

```text
[reserve capacity]
      ↓ 성공
[read_retrying(fd, bytes + end, BUFFER_SIZE)]
      ├─ EINTR ──────────────┐
      │                      └─ 같은 call 다시 시도
      ├─ n > 0
      │    ├─ end += n
      │    ├─ bytes[end] = '\0'
      │    └─ scan 새 범위
      │          ├─ newline: result allocation 후 begin commit
      │          └─ 없음: read 반복
      ├─ 0
      │    ├─ reached_eof = 1
      │    ├─ unread 있음: tail line
      │    └─ unread 없음: EOF
      ├─ EAGAIN/EWOULDBLOCK
      │    └─ cursor 유지, AGAIN
      └─ 다른 error
           └─ cursor 유지, ERROR
```

## 최종 recovery 계약

| 상황 | 보존되는 것 | 소비되는 것 | 다음 행동 |
| --- | --- | --- | --- |
| short positive read | 받은 bytes | 없음, delimiter line 성공 전까지 | 계속 read |
| EINTR | 모든 context state | 없음 | 내부 자동 retry |
| AGAIN | buffer, `begin/scan/end`, EOF=false | 없음 | readiness 뒤 같은 context 호출 |
| terminal ERROR | explicit context의 accepted bytes와 cursor | 없음 | caller가 retry/reset/destroy 결정 |
| line allocation ERROR | delimiter/tail bytes | 없음 | 같은 context retry 가능 |
| EOF with tail | tail result allocation 성공 후 해당 bytes | 한 줄로 commit | 다음 호출 EOF |
| empty EOF | stable EOF state | 없음 | 반복 EOF 또는 reset/destroy |

## 이 Thread의 경계

- 이 문서는 input readiness를 기다리는 event loop나 `poll`/`select`/`epoll` API를 제공하지 않습니다. `BLR_AGAIN`은 caller가 그런 mechanism과 연결할 수 있는 상태값입니다.
- explicit context는 terminal I/O error 뒤 state를 보존하지만, 오류 원인이 자동으로 복구된다는 보장은 없습니다.
- legacy `get_next_line`은 API 제약 때문에 EOF, wait, error를 모두 `NULL`로 축소합니다. 내부 유지 정책만 다릅니다.
- descriptor별 node architecture와 context ownership 자체는 앞선 두 Thread에서 다룹니다.

> 검증 메모: exact SHA의 GitHub diff와 source/test harness를 확인했습니다. 현재 환경에서는 nonblocking test와 fault suite를 실행하지 않았습니다.
