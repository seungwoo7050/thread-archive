# Thread: singleton state에서 descriptor별 compatibility state까지

초기 reader는 process 전체에 하나뿐인 `g_reader`를 사용했습니다. 이 구조에서는 fd A에서 첫 줄을 반환하고 suffix를 보관한 뒤 fd B를 읽는 순간 A의 state를 버려야 합니다. 여러 descriptor를 번갈아 읽으려면 buffer, cursor, cleanup이 각각의 fd에 귀속되어야 합니다.

이 Thread는 두 단계를 분리합니다.

1. helper가 전역 이름을 직접 만지지 않고 explicit state pointer를 받게 하는 준비
2. fd를 key로 하는 linked reader node를 만들고, lookup·mutation·cleanup을 한 node에 한정하는 실제 architecture 변화

## 커밋 구성

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `fc01012e8521` | `refactor(state): reader 상태를 helper 인자로 전달` | B | `REFACTOR`, `READER_LIFECYCLE` | helper의 mutation 대상을 parameter로 드러내지만 singleton 자체는 유지합니다. |
| 2 | `a4f41cbf2cf0` | `feat(state): 디스크립터별 읽기 상태 분리` | S | `ARCH`, `READER_LIFECYCLE`, `RISK` | fd별 buffer와 cursor를 가진 linked node를 도입합니다. |
| 3 | `61f8b9858672` | `test(state): 교차 디스크립터 상태 격리 검증` | A | `TEST`, `READER_LIFECYCLE`, `RISK` | 두 stream을 번갈아 읽고 unrelated invalid fd가 기존 state를 지우지 않는지 확인합니다. |
| 4 | `d3e2b37fca03` | `test(error): 오류 발생 시 디스크립터 상태 정리 검증` | A | `TEST`, `READER_LIFECYCLE`, `RISK` | unusable descriptor의 cleanup이 해당 node에만 적용되고 fd를 library가 닫지 않는지 확인합니다. |
| 5 | `fd03a831686b` | `test(failure): 메모리 할당과 읽기 실패 처리 검증` | A | `TEST`, `POSIX_IO`, `RISK` | deterministic allocation/read failure에서 node별 소유권과 cleanup을 검증합니다. |

## 최종 state model

```text
static g_readers
      │
      ▼
+------------+     +------------+     +------------+
| fd = 8     | --> | fd = 5     | --> | fd = 3     | --> NULL
| bytes      |     | bytes      |     | bytes      |
| begin/scan |     | begin/scan |     | begin/scan |
| end/cap    |     | end/cap    |     | end/cap    |
+------------+     +------------+     +------------+
```

각 node는 자기 fd의 unread bytes와 cursor를 소유합니다. list는 node를 찾고 제거하기 위한 container일 뿐, 한 node의 failure를 이유로 list 전체를 reset하지 않습니다.

---

## `fc01012e8521` — helper mutation의 대상을 parameter로 드러내기

**중요도** B · **태그** `REFACTOR, READER_LIFECYCLE`

이 커밋은 observable behavior를 바꾸지 않습니다. 기존 helper가 `g_reader`를 직접 읽고 수정하던 코드를 `t_reader *reader` 또는 `const t_reader *reader`를 받는 형태로 바꿉니다.

```diff
-static void reset_reader(void)
+static void reset_reader(t_reader *reader)
 {
-    free(g_reader.bytes);
-    g_reader.fd = -1;
+    free(reader->bytes);
+    reader->fd = -1;
     /* ... */
 }
```

같은 변화가 `unread_length`, `compact_bytes`, `reserve_bytes`, `append_bytes`, `find_line_end`, `extract_line`, `release_final_line`에 적용됩니다.

### 아직 singleton인 이유

public entry는 다음처럼 전역 object의 주소를 고정해서 넘깁니다.

```c
char *get_next_line(int fd)
{
    t_reader *reader;

    reader = &g_reader;
    /* 모든 helper에 reader 전달 */
}
```

따라서 이 커밋만으로는 두 fd의 state를 동시에 유지할 수 없습니다. 바뀐 것은 **state selection**이 아니라 **helper 내부의 결합 방식**입니다.

```text
before: helper가 g_reader라는 전역 이름을 암묵적으로 선택
 after: caller가 넘긴 reader를 helper가 수정

그러나 caller는 아직 항상 &g_reader만 선택
```

이 준비가 있어야 다음 커밋에서 `find_reader(fd)`가 선택한 여러 node에 동일 helper를 재사용할 수 있습니다. B 중요도에 맞게 보면, architecture를 직접 완성한 commit이 아니라 architecture 변경을 가능하게 만든 refactor입니다.

---

## `a4f41cbf2cf0` — fd별 linked reader node

**중요도** S · **태그** `ARCH, READER_LIFECYCLE, RISK`

### singleton이 일으키는 correctness 문제

fd A의 첫 read가 두 줄 이상을 가져오면 첫 줄만 caller에게 반환하고 나머지는 internal buffer에 남습니다. 그 상태에서 fd B를 호출할 때 singleton을 B로 바꾸며 reset하면 A의 suffix가 사라집니다.

```text
A: "left one\nleft two"
                 └────── buffered suffix

get_next_line(A) -> "left one\n"
get_next_line(B) -> singleton이 B로 교체되며 A suffix 손실
get_next_line(A) -> kernel offset은 이미 진행했으므로 "left two" 복구 불가
```

buffered read-ahead 때문에 이 문제는 단순한 캐시 miss가 아니라 data loss입니다.

### 새 node와 list head

```diff
  size_t capacity;
+ struct s_reader *next;
 } t_reader;

-static t_reader g_reader = {-1, NULL, 0, 0, 0, 0};
+static t_reader *g_readers;
```

각 fd는 독립 node를 갖고, list head가 그 node들을 연결합니다.

### lookup과 create

```c
static t_reader *find_reader(int fd)
{
    t_reader *reader;

    reader = g_readers;
    while (reader != NULL && reader->fd != fd)
        reader = reader->next;
    return (reader);
}
```

```c
static t_reader *create_reader(int fd)
{
    t_reader *reader;

    reader = malloc(sizeof(*reader));
    if (reader == NULL)
        return (NULL);
    reader->fd = fd;
    reader->bytes = NULL;
    reader->begin = 0;
    reader->scan = 0;
    reader->end = 0;
    reader->capacity = 0;
    reader->next = g_readers;
    g_readers = reader;
    return (reader);
}
```

node allocation과 초기화가 모두 끝난 뒤 list head에 publish합니다. allocation 실패 시 list에는 partial node가 들어가지 않습니다.

### pointer-to-pointer unlink

```c
static void discard_reader(t_reader *reader)
{
    t_reader **link;

    link = &g_readers;
    while (*link != NULL && *link != reader)
        link = &(*link)->next;
    if (*link == NULL)
        return ;
    *link = reader->next;
    free(reader->bytes);
    free(reader);
}
```

`link`는 “현재 node를 가리키는 pointer의 주소”입니다. 첫 node든 중간 node든 별도 분기 없이 unlink할 수 있습니다.

```text
head 제거: link == &g_readers
중간 제거: link == &previous->next

*link = reader->next
```

제거 순서는 list에서 먼저 끊고, node가 소유한 buffer와 node 자체를 해제합니다. 다른 node의 link, buffer, cursor는 건드리지 않습니다.

### entry에서 정확히 한 node 선택

```c
if (fd < 0 || read(fd, buffer, 0) < 0)
{
    reader = find_reader(fd);
    if (reader != NULL)
        discard_reader(reader);
    return (NULL);
}
reader = find_reader(fd);
if (reader == NULL)
    reader = create_reader(fd);
```

invalid descriptor가 들어와도 전체 list를 초기화하지 않습니다. 같은 integer fd의 node가 실제로 존재할 때만 그 node를 제거합니다.

### 결과와 failure별 node lifecycle

| 경로 | selected node 처리 | 다른 node 처리 |
| --- | --- | --- |
| line + unread suffix | node 유지, `begin/scan` 전진 | 변화 없음 |
| line이 buffer를 모두 소비 | selected node 제거 | 변화 없음 |
| EOF tail | bytes ownership을 caller에게 넘긴 뒤 node 제거 | 변화 없음 |
| empty EOF | selected node 제거 | 변화 없음 |
| invalid/closed fd | 그 fd의 node가 있으면 제거 | 변화 없음 |
| reserve/read 오류 | selected node 제거 | 변화 없음 |
| line allocation 오류 | 이 exact SHA에서는 selected node 제거 | 변화 없음 |

### fd number를 key로 쓸 때의 위험

list key는 open file description 자체가 아니라 integer fd입니다. OS는 닫힌 fd 번호를 나중에 재사용할 수 있습니다. unusable fd의 stale node를 남기면 완전히 다른 파일이 같은 숫자를 받았을 때 이전 unread bytes가 새 stream 앞에 붙을 수 있습니다.

따라서 closed/error/EOF 경로에서 node를 제거하는 것은 단순 leak cleanup이 아니라 correctness 조건입니다. 이 커밋은 stale node가 남지 않도록 selected node를 폐기하는 정책을 갖습니다. 실제로 같은 숫자를 `dup2`로 재사용하는 명시적 fixture는 이후 explicit context test에서 추가됩니다.

### 보장 범위

- 여러 integer fd의 unread buffer와 cursor가 독립적으로 유지됩니다.
- 한 fd의 invalid/read/allocation failure는 다른 node를 제거하지 않습니다.
- result allocation은 caller 소유입니다.
- reader는 fd를 key로 빌려 사용할 뿐 `close`하지 않습니다.

아직 caller가 hidden node lifetime을 직접 cancel/reset할 API는 없습니다. 또한 하나의 global list를 사용하므로 thread-safe collection을 제공한다고 볼 수 없습니다.

---

## `61f8b9858672` — interleaved calls가 suffix를 섞지 않는가

**중요도** A · **태그** `TEST, READER_LIFECYCLE, RISK`

이 커밋은 두 descriptor를 번갈아 호출합니다.

```text
left  = "left one\nleft two"
right = "right one\nright two\nright three"

call sequence:
left  -> "left one\n"
right -> "right one\n"
left  -> "left two"
right -> "right two\n"
right -> "right three"
```

각 첫 호출은 kernel fd offset을 앞당기면서 suffix를 user-space node에 남길 수 있습니다. 이후 다른 fd를 읽고 돌아왔을 때 정확한 suffix가 나와야 하므로, 이 test는 list에 node가 존재한다는 것뿐 아니라 buffer/cursor가 실제로 descriptor별로 유지됨을 확인합니다.

두 번째 사례는 valid fd에서 첫 줄을 읽은 뒤 `get_next_line(-1)`을 호출하고, 다시 valid fd의 suffix를 확인합니다.

```c
line = get_next_line(fd);       /* "first\n" */
CHECK(get_next_line(-1) == NULL);
line = get_next_line(fd);       /* "second" */
```

전역 reset 방식이라면 `-1` 호출이 valid node까지 지워 이 test가 실패합니다.

### 증명 범위

- 지정된 두 stream의 interleaving에서 line sequence가 보존됩니다.
- unrelated invalid fd가 existing node를 제거하지 않습니다.

동시 thread 호출, list corruption under races, 동일 open file description을 공유하는 dup alias의 별도 node 문제까지 증명하지는 않습니다.

---

## `d3e2b37fca03` — unusable descriptor cleanup은 local이어야 한다

**중요도** A · **태그** `TEST, READER_LIFECYCLE, RISK`

### write-only descriptor를 reader가 소유하지 않는다는 확인

pipe의 write end는 읽을 수 없으므로 `get_next_line`은 `NULL`을 반환해야 합니다. 그 뒤 같은 fd에 `write`가 성공하는지 확인합니다.

```c
CHECK(get_next_line(fds[1]) == NULL);
CHECK(write(fds[1], "x", 1) == 1);
```

library가 invalid-for-reading fd를 cleanup한다며 `close`했다면 두 번째 assertion이 실패합니다. node의 buffer와 node allocation은 library가 소유하지만, supplied descriptor의 close 책임은 caller에게 남습니다.

### closed fd 하나가 다른 node를 지우지 않는다는 확인

```text
closed: "first\ndiscarded"
kept:   "keep\nsurvive"
```

두 fd에서 첫 줄을 읽어 각각 suffix를 buffer에 남긴 뒤 `closed`만 닫습니다. `get_next_line(closed)`가 `NULL`을 반환한 후에도 `kept`에서 `"survive"`가 나와야 합니다.

```c
close(closed);
CHECK(get_next_line(closed) == NULL);
line = get_next_line(kept);
CHECK(strcmp(line, "survive") == 0);
```

이 exact test는 **같은 fd 번호를 새 파일에 다시 할당하는 상황까지 직접 만들지는 않습니다.** 다만 closed fd의 node를 local하게 폐기하는 production path를 고정해, stale integer-key node가 남을 가능성을 줄입니다. 명시적인 number reuse test는 `249093ba477a`에서 수행됩니다.

---

## `fd03a831686b` — partial construction과 read failure를 결정적으로 만들기

**중요도** A · **태그** `TEST, POSIX_IO, RISK`

이 커밋은 production object를 다음 macro replacement로 컴파일하는 fault binary를 추가합니다.

```make
-Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
```

### allocation ledger

`test_malloc`은 allocation attempt와 live pointer를 기록하고, 지정한 N번째 allocation만 실패시킬 수 있습니다. `test_free`는 다음 세 상태를 구분합니다.

- 정상 live allocation 해제
- 이미 해제한 pointer의 double free
- 추적한 적 없는 pointer의 invalid free

baseline 실행에서 필요한 allocation 횟수를 구한 뒤 1번부터 마지막 attempt까지 하나씩 실패시킵니다.

```text
baseline attempts = N
fail malloc #1
fail malloc #2
...
fail malloc #N
```

매 경우 live allocation이 0이고 invalid/double free가 없어야 합니다. node object, internal buffer growth, result allocation 등 서로 다른 acquisition 지점의 cleanup을 같은 방식으로 훑습니다.

### short read는 EOF가 아니다

`fault_read_limit(3)`은 실제 fd에서 읽되 한 번에 최대 3 byte만 반환하도록 request count를 줄입니다. `"short reads still work\nlast"`가 여러 positive reads로 나뉘어도 첫 line, EOF tail, EOF 순서가 유지되는지 확인합니다.

short positive return을 EOF처럼 처리하는 구현은 이 test를 통과할 수 없습니다.

### 한 fd의 read failure가 다른 node를 지우지 않는가

left stream의 첫 줄을 반환해 suffix를 node에 남긴 뒤 right stream의 첫 positive-count read를 EIO로 실패시킵니다. right는 `NULL`이지만 left는 계속 `"left two"`를 반환해야 합니다.

```text
left node:  unread suffix 유지
right node: read EIO로 selected node cleanup
```

이것이 descriptor-scoped failure의 핵심 assertion입니다.

### 이 SHA의 read-error policy를 소급해서 바꾸지 않기

`test_middle_read_error`는 4-byte short reads 두 번으로 partial bytes를 받아들인 뒤 세 번째 read를 EIO로 실패시킵니다. 이 exact SHA의 expected policy는 다음과 같습니다.

```text
partial bytes must be discarded
```

즉 selected legacy node를 폐기하고 `NULL`을 반환합니다. 후속 POSIX Thread에서 explicit context가 terminal error 뒤 accepted bytes를 보존하고 재시도할 수 있게 되지만, 그 보장을 이 baseline commit에 소급하면 안 됩니다.

또한 이 시점의 harness는 다음만 지원합니다.

- 특정 positive-count read 호출 하나를 EIO로 실패
- 한 read의 최대 길이 제한

`EINTR → progress → EAGAIN` 같은 per-call errno sequence array는 아직 없습니다. 그 기능은 `11033bd85c59`에서 추가됩니다.

---

## node 소유권과 cleanup 정리

| 자원 | 획득 | owner | 정상 종료 | 실패 종료 |
| --- | --- | --- | --- | --- |
| reader node | `create_reader(fd)`의 `malloc` | hidden `g_readers` list | buffer를 모두 소비하거나 EOF면 unlink/free | selected fd의 invalid/read/allocation 오류면 unlink/free |
| internal bytes | reserve/growth | 해당 reader node | line 사이에는 유지, node 종료 시 free 또는 EOF tail로 transfer | selected node 폐기 시 free |
| line result | `extract_line`의 `malloc` | caller | caller가 free | allocation 실패 시 result 없음 |
| supplied fd | caller가 open/pipe/dup | caller | library는 close하지 않음 | read 불가여도 library는 close하지 않음 |
| 다른 descriptor node | 별도 create | 별도 node/list | 독립 진행 | unrelated failure에 변화 없음 |

## 최종 call flow

```text
get_next_line(fd)
  ├─ fd probe 실패
  │    └─ find_reader(fd)한 node만 discard → NULL
  └─ find_reader(fd)
       ├─ 없음: create_reader(fd), list head에 publish
       └─ 있음: 기존 unread state 재사용
            ↓
       buffered newline 탐색 / direct read / extraction
            ├─ line + suffix: node 유지
            ├─ buffer 전부 소비: node discard
            ├─ EOF tail: bytes를 caller에게 넘기고 node discard
            ├─ empty EOF: node discard
            └─ failure: selected node만 discard
```

## 이 Thread의 경계

- 이 문서는 compatibility API가 hidden fd별 state를 관리하는 방법에 한정합니다.
- caller가 context를 직접 create/reset/destroy하는 API는 다음 Thread에서 다룹니다.
- integer fd는 open file description과 같지 않습니다. dup alias, external seek, close/reuse를 caller-visible rule로 만드는 작업도 explicit context Thread에 속합니다.
- `EINTR`, `EAGAIN`, terminal I/O error 뒤 재시도 정책은 POSIX transient-read Thread에서 다룹니다.

> 검증 메모: 표시된 exact SHA의 GitHub diff와 source/test code를 확인했습니다. 이 환경에서는 test binary를 실행하지 않았습니다.
