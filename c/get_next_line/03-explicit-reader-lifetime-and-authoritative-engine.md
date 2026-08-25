# Thread: 명시적 reader 수명과 하나의 authoritative engine

fd별 hidden node는 여러 stream을 번갈아 읽을 수 있게 하지만, caller가 buffered state를 직접 취소하거나 외부 `lseek` 뒤 초기화할 방법은 없습니다. 또한 `char *` 하나만 반환하는 `get_next_line`은 EOF와 오류를 모두 `NULL`로 표현하므로 상태를 정확히 구분할 수 없습니다.

이 Thread는 다음 문제를 함께 해결합니다.

- caller가 reader context를 생성·reset·destroy하는 명시적 lifetime
- `LINE`, `EOF`, `ERROR`를 분리하는 result state
- context가 소유하는 memory와 caller가 소유하는 descriptor의 구분
- legacy `get_next_line`과 explicit API가 서로 다른 parser로 갈라지지 않도록 하나의 engine으로 수렴
- line result allocation이 실패했을 때 unread input을 소비하지 않는 commit rule

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `903768a43bf4` | `feat(context): 명시적 reader 수명 API 추가` | A | `ARCH`, `READER_LIFECYCLE`, `API_CONTRACT` | opaque context와 create/reset/destroy를 공개하고 fd ownership을 caller에게 둡니다. |
| 2 | `2e681112b304` | `feat(reader): 명시적 결과 상태 API 추가` | S | `ARCH`, `API_CONTRACT`, `CORE` | `BLR_LINE`, `BLR_EOF`, `BLR_ERROR`와 stable EOF state를 도입합니다. |
| 3 | `9bd6ebf429e2` | `refactor(reader): legacy API를 context reader에 연결` | A | `REFACTOR`, `INTEGRATION`, `API_CONTRACT` | `get_next_line`을 `blr_reader_next` adapter로 만들고 extraction을 하나로 합칩니다. |
| 4 | `249093ba477a` | `test(context): 결과 상태와 컨텍스트 수명 검증` | A | `TEST`, `READER_LIFECYCLE`, `API_CONTRACT` | descriptor borrowing, reset, reuse, dup alias, stable result를 검증합니다. |
| 5 | `a24ad4e49cc4` | `test(failure): 컨텍스트의 line 할당 재시도 검증` | A | `TEST`, `READER_LIFECYCLE`, `RISK` | newline line과 EOF tail allocation 실패가 input loss 없이 재시도되는지 확인합니다. |

## 최종 API 계약

| API | 역할 | memory ownership | fd ownership |
| --- | --- | --- | --- |
| `blr_reader_create(fd)` | fd에 결합된 context 생성 | context object를 caller가 보유 | fd는 빌릴 뿐 닫지 않음 |
| `blr_reader_next(reader, &line)` | 다음 result state와 optional line 반환 | `BLR_LINE`의 `line`은 caller가 free | fd offset을 읽으며 close하지 않음 |
| `blr_reader_reset(reader)` | buffered bytes/cursor/EOF state 폐기 | context object 자체는 유지 | fd와 현재 kernel offset은 유지 |
| `blr_reader_destroy(reader)` | buffer와 context object 해제 | context ownership 종료 | fd는 열린 채로 유지 |
| `get_next_line(fd)` | hidden context를 사용하는 compatibility adapter | non-NULL result를 caller가 free | fd는 caller 소유 |

---

## `903768a43bf4` — hidden node type을 opaque context로 공개

**중요도** A · **태그** `ARCH, READER_LIFECYCLE, API_CONTRACT`

### public header는 layout을 숨긴다

```c
/* get_next_line.h */
typedef struct s_blr_reader t_blr_reader;

t_blr_reader *blr_reader_create(int fd);
void          blr_reader_reset(t_blr_reader *reader);
void          blr_reader_destroy(t_blr_reader *reader);
```

caller는 pointer handle만 알며 `fd`, `bytes`, cursor를 직접 수정할 수 없습니다. 실제 layout은 implementation file에 남습니다.

```c
struct s_blr_reader
{
    int           fd;
    char          *bytes;
    size_t        begin;
    size_t        scan;
    size_t        end;
    size_t        capacity;
    t_blr_reader  *next;
};
```

이 exact SHA에는 `reached_eof`가 아직 없습니다. stable EOF field는 다음 커밋에서 추가됩니다.

### create는 context object만 즉시 할당한다

```c
t_blr_reader *blr_reader_create(int fd)
{
    char          probe;
    t_blr_reader  *reader;

    if (fd < 0 || read(fd, &probe, 0) < 0)
        return (NULL);
    reader = malloc(sizeof(*reader));
    if (reader == NULL)
        return (NULL);
    reader->fd = fd;
    reader->bytes = NULL;
    reader->begin = 0;
    reader->scan = 0;
    reader->end = 0;
    reader->capacity = 0;
    reader->next = NULL;
    return (reader);
}
```

internal byte buffer는 lazy allocation입니다. 따라서 create의 partial construction은 “context allocation 성공 후 buffer allocation 실패”가 아니라, 이 시점에는 context object 하나의 성공/실패뿐입니다.

zero-length `read` probe는 fd가 읽기 가능한지 확인하지만 data를 소비하지 않습니다. 성공한 context는 supplied fd 번호를 보관하되 fd 자체를 소유하지 않습니다.

### reset과 destroy의 차이

```c
void blr_reader_reset(t_blr_reader *reader)
{
    if (reader == NULL)
        return ;
    free(reader->bytes);
    reader->bytes = NULL;
    reader->begin = 0;
    reader->scan = 0;
    reader->end = 0;
    reader->capacity = 0;
}
```

reset은 buffered read-ahead를 버리지만 context object와 `reader->fd`는 유지합니다. kernel fd offset을 rewind하지도 않습니다.

```c
void blr_reader_destroy(t_blr_reader *reader)
{
    if (reader == NULL)
        return ;
    free(reader->bytes);
    free(reader);
}
```

destroy는 context-owned memory를 모두 해제하지만 `close(reader->fd)`는 호출하지 않습니다.

```text
context owns: context object, internal byte allocation
caller owns:  supplied descriptor, descriptor close timing
```

### legacy list도 같은 lifecycle primitive를 사용

hidden compatibility list는 그대로 남아 있지만 node 생성과 폐기에 public primitive를 재사용합니다.

```c
static t_blr_reader *create_legacy_reader(int fd)
{
    t_blr_reader *reader;

    reader = blr_reader_create(fd);
    if (reader == NULL)
        return (NULL);
    reader->next = g_readers;
    g_readers = reader;
    return (reader);
}
```

```c
static void discard_legacy_reader(t_blr_reader *reader)
{
    /* list에서 unlink */
    blr_reader_destroy(reader);
}
```

그러나 아직 explicit “next line” API는 없습니다. caller가 context를 만들 수는 있어도 읽기 engine을 직접 호출할 수 없는 중간 단계입니다.

---

## `2e681112b304` — data와 status를 분리하는 result state

**중요도** S · **태그** `ARCH, API_CONTRACT, CORE`

### `char *` / `NULL`만으로 표현할 수 없는 상태

legacy API에서는 다음 세 상황이 모두 `NULL`입니다.

- 정상 EOF
- invalid/closed fd 또는 read error
- allocation failure

explicit API는 enum으로 이를 분리합니다.

```c
typedef enum e_blr_result
{
    BLR_ERROR = -1,
    BLR_EOF = 0,
    BLR_LINE = 1
} t_blr_result;
```

이 SHA에는 아직 `BLR_AGAIN`이 없습니다. nonblocking wait state는 POSIX Thread의 `f0055ae5cf19`에서 추가됩니다.

### output pointer rule

`blr_reader_next`는 가능한 경우 가장 먼저 output을 `NULL`로 만듭니다.

```c
if (line != NULL)
    *line = NULL;
if (reader == NULL || line == NULL)
    return (BLR_ERROR);
```

따라서 `BLR_EOF`와 `BLR_ERROR`에서 caller가 stale pointer를 실수로 사용하는 것을 막습니다. `BLR_LINE`일 때만 non-NULL allocation의 ownership이 caller에게 넘어갑니다.

### stable EOF를 context state로 저장

```diff
  size_t capacity;
+ int    reached_eof;
```

create/reset은 `reached_eof = 0`으로 초기화합니다. `read`가 0을 반환하면 1로 바뀝니다.

```c
if (read_size == 0)
{
    reader->reached_eof = 1;
    if (unread_length(reader) == 0)
        return (BLR_EOF);
    line_end = reader->end;
}
```

다음 호출에서 이미 EOF에 도달했고 unread bytes도 없다면 새 `read` 없이 바로 `BLR_EOF`를 반환합니다.

```c
if (line_end == 0 && reader->reached_eof)
{
    if (unread_length(reader) == 0)
        return (BLR_EOF);
    line_end = reader->end;
}
```

EOF는 “이번 system call이 0을 반환했다”는 순간적 사건이 아니라 context의 stable terminal state가 됩니다.

### line allocation과 cursor commit 순서

```c
length = line_end - reader->begin;
*line = malloc(length + 1);
if (*line == NULL)
{
    reader->scan = reader->begin;
    return (BLR_ERROR);
}
copy_bytes(*line, reader->bytes + reader->begin, length);
(*line)[length] = '\0';
reader->begin = line_end;
reader->scan = reader->begin;
return (BLR_LINE);
```

`begin`은 result allocation과 copy가 성공한 뒤에만 전진합니다. allocation 실패 시 `scan`을 `begin`으로 되돌려 다음 호출이 같은 delimiter를 다시 찾게 합니다.

```text
실패 전: [begin ........ line_end .... end)
malloc 실패
실패 후: begin unchanged, end unchanged, scan = begin

재시도: 같은 bytes에서 같은 line_end를 다시 발견
```

이것이 line extraction의 transaction boundary입니다. caller-visible result를 만들지 못한 시도는 input consumption을 commit하지 않습니다.

### 아직 두 engine이 공존한다

중요한 제한이 있습니다. 이 커밋은 `blr_reader_next`를 추가하지만 기존 `get_next_line` body를 제거하지 않습니다. explicit API와 legacy API가 같은 helper를 일부 공유하더라도 top-level parsing control flow는 둘입니다.

```text
explicit caller -> blr_reader_next(...)
legacy caller   -> 기존 get_next_line parser body
```

따라서 이 커밋만으로 “하나의 authoritative engine”이 완성됐다고 설명하면 안 됩니다. 그 수렴은 다음 커밋에서 일어납니다.

---

## `9bd6ebf429e2` — legacy API를 adapter로 축소

**중요도** A · **태그** `REFACTOR, INTEGRATION, API_CONTRACT`

### 두 parser가 유지될 때의 위험

같은 buffer representation을 사용해도 EOF 처리, allocation failure, future errno mapping이 두 top-level loop에서 따로 바뀌면 behavior가 쉽게 diverge합니다.

```text
fix explicit engine only -> legacy regression
fix legacy path only      -> context API regression
```

이 커밋은 `blr_reader_next`만 record state를 전이하는 engine으로 두고, `get_next_line`은 result를 축소해 반환하는 adapter로 만듭니다.

```c
char *get_next_line(int fd)
{
    char          *line;
    t_blr_reader  *reader;
    t_blr_result  result;

    reader = find_reader(fd);
    if (reader == NULL)
        reader = create_legacy_reader(fd);
    if (reader == NULL)
        return (NULL);
    result = blr_reader_next(reader, &line);
    if (result == BLR_LINE)
        return (line);
    discard_legacy_reader(reader);
    return (NULL);
}
```

이 SHA에는 `BLR_AGAIN`이 없으므로 non-line result는 EOF든 ERROR든 hidden node를 폐기하고 `NULL`로 축소합니다.

### newline line과 EOF tail도 같은 extractor 사용

별도 `release_final_line` ownership-transfer 경로가 제거되고, buffered newline과 EOF tail 모두 `extract_line`으로 independent allocation을 만듭니다.

```c
if (reader->reached_eof)
{
    if (unread_length(reader) == 0)
        return (BLR_EOF);
    *line = extract_line(reader, reader->end);
    if (*line == NULL)
        return (BLR_ERROR);
    return (BLR_LINE);
}
```

결과 종류에 따라 서로 다른 ownership 방식이 섞이지 않습니다.

```text
newline-terminated line -> new caller-owned allocation
EOF tail                -> new caller-owned allocation
internal buffer         -> context가 계속 소유
```

이 때문에 context는 EOF tail을 반환한 뒤에도 살아 있고, 다음 호출에서 stable `BLR_EOF`를 반환할 수 있습니다.

### extraction failure에서 context를 보존

```c
line = malloc(length + 1);
if (line == NULL)
{
    reader->scan = reader->begin;
    return (NULL);
}
```

이전 hidden implementation처럼 node 전체를 제거하지 않습니다. `begin`, `end`, bytes는 유지하고 scan만 rewind합니다. explicit API에서 same context retry가 가능해지고, authoritative engine을 사용하는 모든 caller가 같은 commit rule을 따릅니다.

legacy adapter는 `BLR_ERROR`를 받은 뒤 hidden node를 폐기하므로 legacy caller가 allocation error 뒤 retry할 수 있다는 뜻은 아닙니다. “engine은 non-consuming”과 “adapter가 context를 유지하는가”는 별개의 결정입니다.

### public usage rule의 명문화

header comment는 context와 fd의 관계를 구체적으로 제한합니다.

- context는 fd를 빌릴 뿐 닫지 않습니다.
- 같은 open file description을 공유하는 dup fd에는 context 하나만 사용해야 합니다.
- 외부에서 offset을 바꾼 뒤에는 `blr_reader_reset`이 필요합니다.
- context가 살아 있는 동안 fd를 닫고 같은 번호를 재사용했다면 기존 context를 쓰면 안 됩니다.
- 서로 다른 context를 다른 thread에서 쓸 수는 있지만 한 context의 concurrent call은 지원하지 않습니다.

이 규칙은 buffered read-ahead가 kernel offset보다 앞서 있을 수 있다는 사실에서 나옵니다.

```text
kernel offset: BUFFER_SIZE만큼 이미 진행
context begin/end: 그중 caller에게 아직 안 준 bytes 보유
```

external `lseek`만 바꾸고 context를 reset하지 않으면 kernel과 user-space state가 서로 다른 stream 위치를 가리키게 됩니다.

---

## `249093ba477a` — lifetime과 descriptor coupling을 실제 사용 시나리오로 고정

**중요도** A · **태그** `TEST, READER_LIFECYCLE, API_CONTRACT`

### result states와 output pointer

`"first\nlast"`에서 다음 sequence를 검사합니다.

```text
BLR_LINE, line="first\n"
BLR_LINE, line="last"
BLR_EOF,  line=NULL
BLR_EOF,  line=NULL
```

repeated EOF가 stable하고 non-line outcome이 output pointer를 `NULL`로 만든다는 계약을 함께 검증합니다.

### empty EOF와 closed-fd error는 다르다

빈 파일은 `BLR_EOF`입니다. 같은 context를 reset한 뒤 fd를 close하고 호출하면 `BLR_ERROR`입니다. legacy `NULL` 하나로는 구분할 수 없었던 두 상태가 explicit API에서는 분리됩니다.

### external seek에는 reset이 필요하다

첫 줄을 읽고 `lseek(fd, 0, SEEK_SET)`만 수행하면 context 안에 prefetched suffix가 남을 수 있습니다. test는 seek 직후 `blr_reader_reset(reader)`도 호출한 뒤 첫 줄을 다시 읽습니다.

```c
check_line(reader, "repeat\n");
CHECK(lseek(fd, 0, SEEK_SET) == 0);
blr_reader_reset(reader);
check_line(reader, "repeat\n");
```

reset은 fd offset을 바꾸지 않으므로 caller가 offset 변경과 context 초기화를 함께 조정합니다.

### destroy는 fd를 닫지 않는다

context를 destroy한 뒤 `fcntl(fd, F_GETFD)`가 성공하는지 확인하고, fd를 rewind해 새 context를 생성합니다. context cancellation과 descriptor lifetime이 분리되어 있음을 직접 확인합니다.

### 같은 integer fd를 재사용할 때는 새 context

```text
first fd -> old stream context 생성/사용/파괴
close(first)
dup2(replacement, first) -> 같은 숫자에 new stream 연결
blr_reader_create(first) -> 새 context 생성
```

새 context에서 `"new\n"`가 나와야 합니다. fd 숫자가 같다는 이유로 이전 buffered state를 재사용하면 안 된다는 가장 직접적인 regression입니다.

### dup alias는 하나의 context로 읽는다

`dup(fd)`로 같은 open file description을 공유하는 alias를 만든 뒤 original fd를 닫고 alias 하나에 context 하나를 생성합니다. 두 descriptor에 context를 따로 만들어 경쟁시키는 행동을 지원한다는 test가 아닙니다. 오히려 header의 “같은 open file description에는 context 하나”라는 사용 규칙과 일치합니다.

### invalid argument

- `blr_reader_create(-1) == NULL`
- `blr_reader_next(NULL, &line) == BLR_ERROR`이며 `line == NULL`
- `blr_reader_next(NULL, NULL) == BLR_ERROR`
- `reset(NULL)`과 `destroy(NULL)`은 안전하게 반환

이 test set은 public API가 error state에서도 caller가 정리 가능한 형태를 유지하는지 확인합니다.

---

## `a24ad4e49cc4` — allocation 실패를 consumption으로 확정하지 않기

**중요도** A · **태그** `TEST, READER_LIFECYCLE, RISK`

### delimiter가 있는 최소 line

input `"\n"`에서 line result allocation만 실패시킵니다.

```text
1차 blr_reader_next
  -> BLR_ERROR
  -> line == NULL
  -> context 안의 "\n"은 아직 unread

fault 해제 후 2차 호출
  -> BLR_LINE
  -> line == "\n"

3차 호출
  -> BLR_EOF
```

`find_line_end`가 newline을 발견하면서 `scan`을 전진시켰더라도 allocation failure branch가 `scan=begin`으로 되돌리지 않으면 재시도에서 delimiter를 놓칠 수 있습니다. 이 test는 그 rewind를 겨냥합니다.

### EOF tail allocation sweep

`"tail"`처럼 newline 없이 끝나는 입력에서는 EOF를 먼저 관찰한 뒤 tail result allocation이 필요합니다. baseline allocation attempt 수를 구하고, context object 이후의 각 allocation 지점을 하나씩 실패시킵니다.

실패가 result allocation에 해당하면 fault를 해제하고 같은 context를 계속 호출합니다. `"tail"`은 정확히 한 번만 반환되고 그 뒤 EOF가 나와야 합니다.

```text
read EOF -> reached_eof=1, unread="tail"
result malloc failure -> BLR_ERROR, unread 유지
retry -> BLR_LINE "tail", begin=end
next -> BLR_EOF
```

### 증명 범위

- newline-delimited line과 EOF tail의 result allocation failure가 non-consuming입니다.
- retry 뒤 byte loss나 duplicate result가 없습니다.
- context destroy 뒤 allocation ledger가 깨끗합니다.

이 커밋은 `EINTR`, `EAGAIN`, terminal read error sequence를 다루지 않습니다. read recovery는 다음 Thread에서 별도로 검증합니다.

---

## authoritative engine의 최종 흐름

```text
caller
  ├─ explicit API: blr_reader_next(context, &line)
  └─ compatibility: get_next_line(fd)
         └─ hidden context find/create
              └─ blr_reader_next(context, &line)  <-- 유일한 parser engine

blr_reader_next
  1. *line = NULL
  2. argument/fd probe
  3. buffered newline 검색
  4. stable EOF + unread tail 처리
  5. reserve → read → end publish → scan 반복
  6. result allocation
  7. allocation/copy 성공 뒤 begin commit
  8. BLR_LINE / BLR_EOF / BLR_ERROR 반환

get_next_line adapter
  BLR_LINE -> line 반환
  그 외   -> context 정책에 따라 hidden node 정리, NULL 반환
```

## 상태별 caller 책임

| result | `*line` | context state | caller action |
| --- | --- | --- | --- |
| `BLR_LINE` | caller-owned allocation | 다음 unread 위치로 commit | line free 후 다음 호출 가능 |
| `BLR_EOF` | `NULL` | stable EOF | destroy/reset 가능, 반복 호출도 EOF |
| `BLR_ERROR` | `NULL` | context 자체는 valid; unread input은 engine에서 보존 가능 | retry, reset, destroy 중 선택 |

`BLR_ERROR` 뒤 실제 retry가 유용한지는 오류 원인에 따라 다릅니다. closed fd처럼 지속되는 오류는 같은 호출을 반복해도 해결되지 않습니다. API는 context를 파괴하지 않고 선택권을 caller에게 남깁니다.

## 이 Thread의 경계

- hidden descriptor list의 도입 자체는 이전 Thread에서 다룹니다.
- 이 문서는 blocking reader의 lifetime/result/engine 통합에 집중합니다.
- `BLR_AGAIN`, `EINTR` retry, terminal read error 뒤 accepted bytes resume는 POSIX transient-read Thread에서 이어집니다.
- context의 same-object concurrent access를 안전하게 만드는 synchronization은 제공하지 않습니다.

> 검증 메모: exact SHA의 GitHub diff, source, header contract, test fixture를 확인했습니다. 이 환경에서는 test target을 실행하지 않았습니다.
