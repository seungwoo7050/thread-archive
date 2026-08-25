# Thread: 반복 전체 복사를 growable builder로 바꾸고 관찰 가능하게 만들기

Project: `small-shell` · Branch: `c/minishell` · 문서 번호: 05

## 개요

초기 lexer와 expander는 문자 하나 또는 치환 문자열 하나를 붙일 때마다 다음 작업을 반복했다.

```text
현재 문자열 길이 계산
  → 새 전체 길이만큼 allocation
  → 지금까지 만든 prefix 전체 복사
  → 새 fragment 복사
  → 이전 buffer 해제
```

길이 `n`인 word를 한 글자씩 만들면 복사량은 대략 `1 + 2 + ... + n`이 된다. 기능적으로는 맞지만 입력이 커질수록 비용이 제곱에 가깝게 증가한다. Heredoc, input, expansion처럼 문자열을 계속 누적하는 코드가 늘어나면 같은 문제와 failure cleanup 규칙도 여러 곳에 복제된다.

이 Thread는 세 단계로 문제를 닫는다.

1. capacity를 기하급수적으로 늘리는 공통 builder를 도입한다.
2. lexer와 expansion을 builder로 옮기되 quote marker와 변수 확장 의미를 유지한다.
3. 512 KiB end-to-end deadline, ASan, UBSan 경로로 결과를 관찰한다.

Builder의 가치는 성능만이 아니다. 다음 상태 규칙을 하나의 abstraction에 모은다.

- `data[length]`는 항상 NUL이다.
- 필요한 크기 계산과 capacity 증가에서 overflow를 검사한다.
- `realloc` 실패 시 기존 buffer를 잃지 않는다.
- 실패는 `discard`, 성공은 `take`로 ownership을 명시한다.

## 커밋 지도

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `b8347c06b6c7` | `refactor(buffer): 가변 문자열 빌더 모듈 추가` | A | `ARCH`, `PERF`, `REFACTOR` | growth·overflow·NUL·discard/take contract를 공통 모듈로 정의 |
| 2 | `985f90b9cbc7` | `refactor(lexer): 단어 조립을 가변 버퍼로 전환` | B | `LEX_PARSE`, `PERF`, `REFACTOR` | quote-aware word construction에 builder 적용 |
| 3 | `89e1a06f06c9` | `refactor(expand): 확장 결과를 가변 버퍼로 조립` | B | `EXPANSION`, `PERF`, `REFACTOR` | `$?`, `$NAME`, dequote 결과 조립에 builder 적용 |
| 4 | `b36b9d324260` | `test(performance): 긴 입력 처리 시간 상한 검증` | B | `TEST`, `PERF` | 512 KiB word를 5초 제한 안에 end-to-end 처리하는지 확인 |
| 5 | `7d7dd7ad9d8a` | `build(test): ASan·UBSan 검증 경로 추가` | B | `TEST`, `PRACTICAL` | behavior·failure·lifecycle·performance suite를 sanitizer artifact로 재실행 가능하게 함 |

## 1. 공통 builder가 보장할 상태를 먼저 정한다

### `b8347c06b6c7` — append API보다 중요한 것은 state contract다

**중요도** `A` · **태그** `ARCH, PERF, REFACTOR`

새 구조체는 세 필드만 가진다.

```c
typedef struct s_string_builder {
    char    *data;
    size_t  length;
    size_t  capacity;
} t_string_builder;
```

### 초기 상태

```c
#define STRING_BUILDER_INITIAL_CAPACITY 64

int string_builder_init(t_string_builder *builder)
{
    builder->data = shell_malloc(STRING_BUILDER_INITIAL_CAPACITY);
    builder->length = 0;
    builder->capacity = 0;
    if (builder->data == NULL)
        return 1;
    builder->capacity = STRING_BUILDER_INITIAL_CAPACITY;
    builder->data[0] = '\0';
    return 0;
}
```

성공한 initialized builder는 다음 불변 조건을 갖는다.

```text
data != NULL
0 <= length < capacity
data[length] == '\0'
capacity >= 64
```

Allocation이 실패하면 `capacity`는 0이고 caller는 실패를 받는다. 이 시점에는 publish할 문자열이 없다.

### 필요한 크기 계산의 overflow

Append 전에 필요한 byte 수는 다음과 같다.

```text
needed = length + extra + 1
                           └─ terminating NUL
```

직접 더하면 `size_t` wraparound가 가능하므로 먼저 역방향 조건을 검사한다.

```c
if (extra > SIZE_MAX - builder->length - 1) {
    errno = ENOMEM;
    return 1;
}
needed = builder->length + extra + 1;
```

이 검사는 “실제 heap이 그만큼 있는가”가 아니라 arithmetic 자체가 표현 가능한가를 보장한다.

### geometric growth와 두 번째 overflow 경계

현재 capacity가 부족하면 두 배씩 늘린다.

```c
capacity = builder->capacity;
while (capacity < needed) {
    if (capacity > SIZE_MAX / 2) {
        capacity = needed;
        break;
    }
    capacity *= 2;
}
```

`capacity * 2`가 overflow할 수 있는 지점에서는 doubling을 멈추고 이미 검증된 exact `needed`를 사용한다. 따라서 growth policy 때문에 wraparound가 생기지 않는다.

### `realloc` 결과를 임시 pointer에 받는다

```c
grown = shell_realloc(builder->data, capacity);
if (grown == NULL)
    return 1;
builder->data = grown;
builder->capacity = capacity;
```

`builder->data = realloc(...)`로 바로 덮어쓰지 않는다. 실패하면 기존 buffer pointer와 지금까지의 bytes가 그대로 남아 caller가 `discard`할 수 있다.

### append 뒤 permanent NUL

한 문자를 붙인 뒤:

```c
builder->data[builder->length++] = value;
builder->data[builder->length] = '\0';
```

Text를 붙인 뒤:

```c
memcpy(builder->data + builder->length, text, length);
builder->length += length;
builder->data[builder->length] = '\0';
```

각 public append가 성공할 때마다 `data[length] == '\0'`가 다시 성립한다. Caller가 별도의 finalize-NUL 단계를 기억할 필요가 없다.

### 실패와 성공의 ownership protocol

```c
void string_builder_discard(t_string_builder *builder)
{
    free(builder->data);
    builder->data = NULL;
    builder->length = 0;
    builder->capacity = 0;
}
```

```c
char *string_builder_take(t_string_builder *builder)
{
    char *data;

    data = builder->data;
    builder->data = NULL;
    builder->length = 0;
    builder->capacity = 0;
    return data;
}
```

두 함수 모두 builder를 같은 empty/reset state로 만든다. 차이는 allocation의 최종 owner다.

| 종료 | allocation owner | builder state |
| --- | --- | --- |
| `discard` | 없음 — 해제됨 | `{NULL, 0, 0}` |
| `take` | 반환받은 caller | `{NULL, 0, 0}` |

이 분리 덕분에 call site의 cleanup은 “실패면 discard, 성공이면 take”로 읽힌다. Builder 자체가 final string의 장기 수명을 소유하지 않는다.

### 기존 방식과 비용 차이

기존 `sh_strjoin_free(left, right)`는 매 호출마다 `strlen(left)`, 새 allocation, prefix 전체 `memcpy`를 수행한다. 한 글자씩 `n`번 붙이면 prefix copy 합은 다음과 같다.

```text
1 + 2 + 3 + ... + n = n(n+1)/2
```

따라서 이 사용 패턴은 O(n²) byte copy와 O(n) allocation을 만든다.

Builder는 capacity가 64, 128, 256, ...으로 증가한다. Growth 횟수는 O(log n)이고, 각 기존 byte가 capacity growth 과정에서 복사되는 총량은 geometric series로 O(n)에 묶인다. 일반 append는 이미 확보한 tail에 직접 쓴다. 이 분석은 코드의 growth rule에서 나오는 점근적 설명이며, 실제 wall-clock 시간을 단독으로 보장하지는 않는다.

## 2. lexer migration은 representation을 바꾸지 않는다

### `985f90b9cbc7` — word 조립 방식만 교체

**중요도** `B` · **태그** `LEX_PARSE, PERF, REFACTOR`

기존 `read_word`는 `char *word`를 매 append마다 `sh_strjoin_free`로 교체했다. 이후에는 stack-local builder를 초기화하고, 성공한 마지막에만 owned string을 꺼낸다.

```diff
-    word = sh_strdup("");
-    if (word == NULL)
+    if (string_builder_init(&word) != 0)
         return NULL;
```

```diff
-    return word;
+    return string_builder_take(&word);
```

### single-quote marker pair 보존

Single-quoted 문자는 여전히 marker와 실제 문자 두 byte로 저장된다.

```c
static int append_literal(t_string_builder *word, char c)
{
    return (string_builder_append_char(word, LITERAL_MARK) != 0
        || string_builder_append_char(word, c) != 0);
}
```

첫 append가 성공하고 두 번째 append가 실패하면 builder 안에는 잠시 dangling marker가 남을 수 있다. 그러나 caller는 failure를 받는 즉시 builder 전체를 `discard`하고 token을 publish하지 않는다. 따라서 malformed encoded word가 후속 parser나 expander에 노출되지 않는다.

### quote별 의미가 유지되는 지점

| source fragment | builder append | 후속 의미 |
| --- | --- | --- |
| unquoted byte | 실제 byte 한 개 | `$`는 expansion trigger 가능 |
| single-quoted byte | `LITERAL_MARK`, byte | `$`도 literal |
| double-quoted byte | 실제 byte 한 개 | 이 프로젝트 규칙에서 `$` 확장 가능 |
| quote delimiter | append하지 않음 | delimiter는 결과 text에서 제거 |
| empty quote | byte append 없음, `quoted=1` | 길이 0인 유효 word token 가능 |

`quoted` flag를 설정하는 lexer state는 그대로다. 이 field는 Thread 02의 heredoc provenance fix가 사용한다.

### 모든 실패 path가 같은 local owner로 돌아온다

- builder init failure → `NULL`
- append failure → `string_builder_discard`
- unclosed quote → `string_builder_discard` 후 syntax error
- success → `string_builder_take`, token node가 반환 문자열을 소유

즉 migration은 token의 최종 ownership이나 marker encoding을 바꾸지 않고, construction cost와 partial-buffer cleanup만 바꾼다.

## 3. expansion migration도 observable semantics를 유지한다

### `89e1a06f06c9` — `$?`, `$NAME`, literal marker를 같은 builder에 append

**중요도** `B` · **태그** `EXPANSION`, `PERF`, `REFACTOR`

`expand_word`와 `dequote_word`가 builder를 사용하도록 바뀐다. 각 branch가 새 whole string을 반환하는 대신 append 성공 여부를 반환한다.

```c
if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
    failed = string_builder_append_char(&out, word[i + 1]);
    i += 2;
} else if (word[i] == '$' && word[i + 1] == '?') {
    failed = append_status(&out, shell->last_status);
    i += 2;
} else if (word[i] == '$'
    && sh_is_name_start((unsigned char)word[i + 1])) {
    /* key를 추출하고 env value를 append */
} else {
    failed = string_builder_append_char(&out, word[i]);
    i++;
}
```

### branch별 의미 보존

- `LITERAL_MARK + byte`: marker는 출력하지 않고 byte만 append한다.
- `$?`: current `last_status`를 decimal text로 append한다.
- `$NAME`: identifier 끝까지 scan하고 environment value를 append한다.
- unset name: `env_get`이 `NULL`을 반환해도 `string_builder_append_text`가 no-op으로 처리하므로 아무 byte도 추가하지 않는다.
- ordinary byte: 그대로 append한다.
- empty input 또는 모든 변수가 empty: initialized builder의 `""`를 `take`한다.

Environment key를 만들기 위한 `sh_substr`은 여전히 별도 temporary allocation이다. Key allocation이나 value append가 실패하면 key를 해제하고 builder를 discard한다.

```c
key = sh_substr(word, start, i - start);
if (key == NULL) {
    string_builder_discard(&out);
    return NULL;
}
failed = string_builder_append_text(&out, env_get(shell->env, key));
free(key);
```

`dequote_word`도 literal marker를 제거하면서 byte를 append하는 같은 구조를 사용한다. Heredoc delimiter normalization과 public dequote API의 의미는 migration 전후 동일하다.

### migration 전후의 경계

| 항목 | 이전 | 이후 | 바뀌지 않은 observable contract |
| --- | --- | --- | --- |
| char append | whole-string join | tail append | 같은 bytes |
| status append | decimal string join | decimal text append | 같은 `$?` text |
| env append | value와 join | value text append | unset은 empty |
| literal marker | join하며 제거 | append하며 제거 | quoted `$` literal |
| failure | current `char *` free | builder discard | partial output 비공개 |
| success | current pointer 반환 | builder take | caller가 owned NUL string 수령 |

이 commit은 expansion timing을 바꾸지 않는다. Connector gate를 통과한 pipeline만 확장한다는 규칙은 Thread 01의 `13a70b408e89`에 그대로 남는다.

## 4. 큰 입력을 실제 product path로 통과시킨다

### `b36b9d324260` — 512 KiB, 5초, 정확한 output length

**중요도** `B` · **태그** `TEST, PERF`

`tests/performance.sh`는 다음 input file을 만든다.

```text
"echo " + 524,288개의 'x' + "\n"
```

Shell을 timeout runner로 5초 제한 안에서 실행한 뒤 네 가지를 검사한다.

```sh
[ "$status" -eq 0 ]
[ ! -s "$TMP/long.err" ]
output_size=$(wc -c <"$TMP/long.out")
[ "$output_size" -eq $((524288 + 1)) ]
```

마지막 `+1`은 `echo`가 붙이는 newline이다.

### 통과하는 production path

```text
non-interactive line input growth
  → lexer가 512 KiB word token 구성
  → parser argv construction
  → selected pipeline expansion/dequote
  → parent/child builtin dispatch
  → echo output
```

따라서 builder helper만 microbenchmark하는 test가 아니다. 실제 command-processing graph에서 긴 word가 deadline을 넘지 않고, stderr 없이, 기대한 byte 수로 출력되는지 본다.

### 증명하는 것

- 이 hardware/test environment에서 해당 512 KiB workload가 5초 안에 완료되어야 한다.
- output이 조용히 잘리거나 추가 byte가 생기면 length assertion이 실패한다.
- 기존 repeated full-copy behavior로 돌아가 성능이 크게 악화되면 timeout이 회귀 신호가 될 수 있다.

### 증명하지 않는 것

- Builder 구현이 수학적으로 O(n)이라는 형식적 증명
- 모든 입력 크기에서 선형 scaling
- payload byte가 모두 `x`인지에 대한 byte-for-byte comparison — test는 크기와 stderr/status를 본다
- 느린 machine이나 sanitizer 환경에서도 반드시 같은 5초를 만족한다는 보편적 보장
- heredoc body나 수많은 작은 variable substitution 조합 전체의 성능

Deadline test는 점근적 분석을 대신하지 않고, 구조적 개선이 product path에서 관찰 가능한 상한을 갖도록 한다.

## 5. sanitizer가 normal·failure·performance path를 같은 source로 다시 만든다

### `7d7dd7ad9d8a` — 별도 instrumented artifact와 격리된 container 경로

**중요도** `B` · **태그** `TEST`, `PRACTICAL`

Makefile은 ordinary binary를 덮어쓰지 않고 sanitizer별 artifact를 따로 만든다.

| 종류 | production binary | fault-injection binary | parser API binary |
| --- | --- | --- | --- |
| ASan | `small-shell-asan` | `small-shell-test-asan` | `tests/parser-api-asan` |
| UBSan | `small-shell-ubsan` | `small-shell-test-ubsan` | `tests/parser-api-ubsan` |

공통 compile option은 `-O1 -g -fno-omit-frame-pointer`이며, 각각 `-fsanitize=address` 또는 `-fsanitize=undefined`를 추가한다. Test variant에는 `SMALL_SHELL_TESTING`도 유지되므로 allocation/I/O/process failure seam이 instrumented code 안에서 그대로 동작한다.

### ASan target

```text
ASAN_OPTIONS=detect_leaks=1:halt_on_error=1
```

다음 suite를 ASan artifact로 실행한다.

- smoke behavior
- system-call fault injection
- allocation sweep
- lifecycle stress
- parser API
- long-input performance

이 조합은 builder의 normal append뿐 아니라 allocation failure 후 `discard`, parser cleanup, heredoc body cleanup, PID/FD lifecycle까지 같은 instrumented source에서 관찰한다.

### UBSan target

```text
UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1
```

같은 suite를 undefined-behavior instrumentation 아래 실행한다. Builder의 `SIZE_MAX` 계산과 indexing, parser/expander의 marker access 같은 경계에서 UB가 발생하면 즉시 중단하도록 한다.

### `env -i`와 sanitizer option 전달

Allocation tests는 환경을 격리하기 위해 `env -i`를 사용한다. 이때 sanitizer option까지 지우면 intended runtime behavior가 바뀔 수 있으므로 script가 `ASAN_OPTIONS`와 `UBSAN_OPTIONS`를 명시적으로 다시 전달한다.

### container 경로

`test-sanitizers-container`는 다음 환경에서 suite를 다시 build한다.

```text
image: gcc:13-bookworm
network: none
root filesystem/mount: read-only
writable work area: /tmp tmpfs
source: read-only mount 후 /tmp로 복사
```

Host compiler와 runtime 차이에서 오는 sanitizer 편차를 줄이고, build가 repository 밖의 network나 writable source tree에 의존하지 않는지 확인할 수 있다.

### 이 commit의 정확한 범위

이 commit은 sanitizer **실행 경로를 추가**한다. 해당 SHA에서 모든 platform의 test가 실제 통과했다는 사실을 diff만으로 주장할 수는 없다. ASan과 UBSan도 data race, 모든 logical leak, 성능 complexity를 전부 증명하지 않는다.

## 핵심 불변 조건과 근거

| 불변 조건 | 도입 | 적용/검증 | 코드 근거 |
| --- | --- | --- | --- |
| `data[length] == '\0'` | `b8347c06b6c7` | lexer/expand migration | init과 모든 append 끝의 NUL assignment |
| size 계산이 wraparound하지 않음 | `b8347c06b6c7` | UBSan path | `extra > SIZE_MAX - length - 1`, doubling guard |
| realloc 실패가 기존 pointer를 잃지 않음 | `b8347c06b6c7` | allocation sweep/ASan 가능 | temporary `grown` |
| failure result는 publish되지 않음 | builder contract | `985f...`, `89e1...` | append failure → `discard` → NULL |
| success 시 allocation owner가 caller로 이동 | builder contract | lexer/expand | `take` 후 builder reset |
| quote marker 의미가 migration 전후 동일 | `985f90b9cbc7` | smoke/parser/heredoc tests | marker+byte append, quoted flag 유지 |
| variable/status expansion 의미가 동일 | `89e1a06f06c9` | behavior tests | 동일 branch와 environment lookup |
| 긴 word 처리에 explicit deadline이 있음 | `b36b9d324260` | performance suite | 512 KiB, 5초, status/stderr/size |
| normal·failure path를 sanitizer로 build 가능 | `7d7dd7ad9d8a` | ASan/UBSan targets | 별도 production/test/parser artifacts |

## 복사 비용과 ownership 흐름

### 이전

```text
word=""
for each fragment:
  strlen(word)
  malloc(old + fragment + 1)
  memcpy(all old bytes)
  memcpy(fragment)
  free(old word)
```

### 이후

```text
builder init(capacity=64, length=0)
for each fragment:
  reserve only if needed
    capacity *= 2
    realloc through temporary pointer
  memcpy only new fragment at data+length
  update length and NUL
success: take → caller owns char*
failure: discard → no partial string escapes
```

## 최종 상태

Builder가 들어간 뒤 lexer와 expansion의 final observable representation은 바뀌지 않는다.

```text
source word
  → same literal-marker encoding
  → same token quote flag
  → same variable/status expansion
  → same NUL-terminated result
```

바뀐 것은 result를 만드는 비용과 실패 시 ownership이다.

- 매 append마다 prefix 전체를 복사하지 않는다.
- overflow가 계산 단계에서 명시적으로 실패한다.
- allocation failure에서도 기존 builder pointer를 잃지 않는다.
- 성공과 실패의 final owner가 `take`와 `discard`로 구분된다.
- large-input deadline과 sanitizer build가 회귀 관찰 지점을 제공한다.

## 이 Thread의 경계

- Token과 parser가 어떤 의미 구조를 만드는지는 Thread 01이다. 여기서는 word bytes를 만드는 방식만 다룬다.
- Heredoc quote provenance와 input-boundary recovery는 Thread 02다. Builder는 그 과정의 buffer implementation일 뿐이다.
- PID·FD와 process lifecycle은 Thread 03에 속한다.
- Nullable allocation과 command-level failure propagation은 Thread 04다. 이 문서는 builder 자체의 allocation/ownership와 performance에 집중한다.
- `7d7dd7ad9d8a`가 sanitizer로 다시 실행하는 lifecycle/fault suite의 개별 invariant는 각 해당 Thread에서 설명한다.

### 검증 범위

표시된 SHA의 diff와 source·Makefile·test script를 `c/minishell` branch에서 확인했다. Repository를 로컬 checkout할 수 없어 performance, ASan, UBSan target을 다시 실행하지 않았으며, source에 정의된 assertions와 build graph만 기술했다.
