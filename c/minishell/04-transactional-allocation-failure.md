# Thread: 치명적 할당 실패를 명령 단위 transaction으로 바꾸기

Project: `small-shell` · Branch: `c/minishell` · 문서 번호: 04

## 개요

초기 구현의 allocation helper는 메모리를 얻지 못하면 process를 바로 종료했다. 이 방식은 caller의 코드를 단순하게 만들지만, 실행 중인 shell에는 맞지 않는다. 한 command를 구성하다 실패했다는 이유로 shell process 전체가 임의의 utility 함수 안에서 종료되고, 이미 만들어진 부분 객체나 persistent state가 어떤 상태인지 caller가 판단할 기회도 없기 때문이다.

이 Thread의 중심은 allocator wrapper가 아니다. 핵심은 다음 failure contract로의 전환이다.

> 새 결과는 완전하게 구성된 뒤에만 공개한다. 실패하면 부분 결과를 전부 정리하고, 이미 유효했던 persistent state는 유지하며, caller가 command status로 실패를 처리한다.

이 원칙은 메모리 객체에만 적용되지 않는다.

- pipeline 실행 준비에서는 allocation이 `pipe()`보다 먼저 끝나야 한다.
- heredoc 준비에서는 이미 소비한 stdin 위치까지 복구해야 한다.
- 반복 실패로 입력 경계를 복구할 수 없다면 계속 실행하지 않아야 한다.

따라서 여기서 말하는 “transactional”은 일반 데이터베이스 transaction이 아니라, **command-processing phase마다 publish 가능한 완성 상태와 rollback 가능한 local state를 구분하는 구현 규칙**이다.

### 최종 failure model

| 구간 | 성공 시 publish | 할당 실패 시 유지해야 할 것 | 실패 결과 |
| --- | --- | --- | --- |
| lexer | 완성된 token node/list | 이전 token prefix는 cleanup 가능해야 함 | status 1, command 미실행 |
| parser | 완성된 argv/redirection/command/pipeline | 이미 publish된 tree와 현재 partial object를 모두 해제 | status 1, command 미실행 |
| expansion | 완성된 replacement string | persistent shell state는 변경하지 않음; ephemeral tree는 폐기 가능 | status 1, pipeline 미실행 |
| environment update | 새 key/value 또는 새 value | 기존 environment entry | builtin status 1 |
| executor setup | pipe/PID tables | 아직 OS pipe/child가 없어야 함 | pipeline status 1 |
| heredoc preparation | body entry와 delimiter boundary | 남은 body가 command로 해석되지 않도록 stdin 위치 복구 | status 1; 복구 불가 시 shell 중단 |

## 커밋 지도

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `0b2e76386678` | `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합` | A | `ARCH`, `FAILURE`, `TEST` | allocation 호출을 overflow-aware runtime boundary로 모음 |
| 2 | `0bb6f9de0947` | `fix(memory): 구조화 단계의 할당 실패를 명령 오류로 전파` | S | `ARCH`, `FAILURE`, `RISK` | fatal helper를 제거하고 construction/publish/cleanup/error propagation을 전역 재설계 |
| 3 | `6d95776ede59` | `fix(memory): 실행 자원 할당 실패를 pipeline 오류로 전파` | A | `PROCESS`, `FAILURE`, `RISK` | 모든 execution table을 OS side effect 전에 확보 |
| 4 | `c30b39c0bcf8` | `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구` | A | `HEREDOC`, `FAILURE`, `RISK` | allocation failure가 이미 소비한 stdin 위치까지 rollback 범위에 포함 |
| 5 | `476b082d55c7` | `test(memory): 범위별 할당 실패 순회 검증` | A | `TEST`, `FAILURE`, `RISK` | phase·command·call position별 failure를 순회해 atomicity와 continuation을 검증 |

## 1. wrapper는 실패를 보이게 할 뿐 정책을 결정하지 않는다

### `0b2e76386678` — allocation call site를 한 경계로 모은다

**중요도** `A` · **태그** `ARCH, FAILURE, TEST`

이 커밋은 `malloc`, `calloc`, `realloc`을 다음 wrapper로 치환한다.

```c
void *shell_malloc(size_t size)
{
    return malloc(size);
}

void *shell_calloc(size_t count, size_t size)
{
    if (size != 0 && count > SIZE_MAX / size) {
        errno = ENOMEM;
        return NULL;
    }
    return calloc(count, size);
}

void *shell_realloc(void *ptr, size_t size)
{
    return realloc(ptr, size);
}
```

`shell_calloc`은 `count * size`가 `size_t`를 넘는 경우 실제 allocator를 호출하지 않고 `ENOMEM`을 반환한다. 이 경계로 이동한 주요 call site는 다음과 같다.

- executor의 pipe/PID table
- heredoc body buffer와 entry
- non-interactive input line buffer
- shared string utility
- environment serialization에 쓰이는 helper allocation

Production build에서는 raw allocator를 그대로 호출하므로 이 commit만으로 정상 동작은 바뀌지 않는다. 또한 caller별 error policy도 아직 통일되지 않는다. 일부 helper는 `NULL`을 반환하지만, `sh_xcalloc`·`sh_strdup` 같은 utility는 여전히 실패 시 `exit(1)`을 호출한다.

즉 이 commit이 제공하는 것은 **관찰과 주입이 가능한 공통 호출 경계**다. “복구 가능한가”, “기존 state를 유지해야 하는가”, “shell을 중단해야 하는가”는 각 caller의 책임으로 남는다.

### wrapper와 transaction policy의 차이

| 질문 | wrapper가 답하는가 | caller가 답하는가 |
| --- | :---: | :---: |
| overflow인가 | 예 (`shell_calloc`) | — |
| allocation이 성공했는가 | 예 | — |
| 기존 environment value를 언제 해제할까 | 아니오 | 예 |
| partial token list를 누가 정리할까 | 아니오 | 예 |
| `pipe()` 전에 어떤 table을 준비할까 | 아니오 | 예 |
| heredoc input을 어디까지 버릴까 | 아니오 | 예 |
| failure를 status 1로 할지 process exit로 할지 | 아니오 | 예 |

## 2. construction은 성공한 객체만 publish한다

### `0bb6f9de0947` — project-wide failure architecture 변경

**중요도** `S` · **태그** `ARCH, FAILURE, RISK`

이 commit은 단순히 `exit`를 `return NULL`로 바꾸지 않는다. Environment, lexer, parser, expansion, public API, command loop의 각 caller가 실패를 받아 정리·전파하도록 함께 바꾼다.

### fatal helper 제거

기존 `sh_xcalloc`은 실패하면 process를 종료했다. 이후 `sh_calloc`은 runtime wrapper의 결과를 그대로 반환한다.

```diff
-void *sh_xcalloc(size_t count, size_t size)
+void *sh_calloc(size_t count, size_t size)
 {
-    void *ptr = calloc(count, size);
-    if (!ptr) {
-        perror("small-shell: calloc");
-        exit(1);
-    }
-    return ptr;
+    return shell_calloc(count, size);
 }
```

`sh_strdup`, `sh_substr`, `sh_strjoin_free`, `shell_itoa_status`도 실패 시 process를 끝내지 않고 `NULL`을 반환한다. 따라서 utility 함수가 shell lifetime을 결정하지 않는다.

### environment: 새 값을 확보한 뒤 기존 값을 버린다

Persistent state mutation은 가장 엄격한 atomicity를 요구한다. 기존 entry의 value를 먼저 해제한 뒤 복제가 실패하면 environment가 손상된다. Fix는 순서를 바꾼다.

```c
if (node != NULL && value != NULL) {
    char *copy;

    copy = sh_strdup(value);
    if (copy == NULL)
        return 1;
    free(node->value);
    node->value = copy;
}
```

새 node도 key와 value가 모두 성공한 뒤에만 list에 연결한다.

```c
node = (t_env *)sh_calloc(1, sizeof(t_env));
if (node == NULL)
    return NULL;
node->key = sh_strdup(key);
node->value = sh_strdup(value != NULL ? value : "");
if (node->key == NULL || node->value == NULL) {
    free(node->key);
    free(node->value);
    free(node);
    return NULL;
}
```

따라서 `export ALLOC_SWEEP=new`의 allocation이 실패하면 기존 `ALLOC_SWEEP=old`가 남는다. 이 persistent-state guarantee는 후속 allocation sweep이 직접 확인한다.

### environment import/export: prefix owner를 명확히 한다

Startup의 `env_from_environ`은 key 또는 node allocation이 실패하면 지금까지 만든 linked-list prefix를 `env_free`로 해제한다. Child exec용 `env_to_environ`은 output vector와 각 `KEY=VALUE` string을 구성하다 실패하면 `sh_free_words(out)`으로 prefix를 해제한다.

이 두 함수는 같은 자료를 다루지만 publish 지점이 다르다.

- startup import는 완성된 environment head만 caller에 반환한다.
- serialization은 완성된 NULL-terminated vector만 child path에 반환한다.

### lexer: node를 list에 연결하기 전 text와 node가 모두 필요하다

`new_token`은 node allocation이 실패하면 전달받은 text까지 해제한다.

```c
static t_token *new_token(t_token_type type, char *text,
        size_t start, int quoted)
{
    t_token *token;

    token = (t_token *)sh_calloc(1, sizeof(t_token));
    if (token == NULL) {
        free(text);
        return NULL;
    }
    token->type = type;
    token->text = text;
    token->start = start;
    token->quoted = quoted;
    return token;
}
```

`push_token`은 non-NULL node만 list에 연결한다. Word/operator 생성이 실패하면 `tokenize_line`은 `allocation failure`를 설정하고 이미 publish된 token prefix를 `free_tokens`로 해제한다. 미완성 node가 public list에 연결되는 순간은 없다.

### parser: 현재 object와 publish된 prefix를 한 failure entry에서 정리한다

`add_arg`는 새 vector와 새 text copy를 모두 확보한 뒤 기존 pointers를 이동하고 `cmd->argv`를 교체한다.

```c
next = (char **)sh_calloc(n + 2, sizeof(char *));
if (next == NULL)
    return 1;
copy = sh_strdup(text);
if (copy == NULL) {
    free(next);
    return 1;
}
/* 기존 owned pointers 이동 */
free(cmd->argv);
cmd->argv = next;
```

`add_redir`도 node와 target copy가 완성된 뒤에만 command tail에 append한다. Parser 전체 실패는 `parse_failure` 하나로 모인다.

```text
현재 미완성 command
  + 현재 미완성 pipeline
  + 이미 sequence에 publish된 pipeline prefix
  → 모두 해제하고 NULL 반환
```

Syntax error와 allocation failure는 같은 cleanup 경로를 공유하지만, top-level status는 다르게 전파된다.

### expansion: persistent state atomicity와 ephemeral tree 폐기를 구분한다

`expand_word`는 새 string을 local에서 완성하고 실패하면 그 string만 해제한다. `expand_words`는 word 하나가 완성되었을 때 기존 argv element를 교체한다.

```c
expanded = expand_word(shell, (*words)[i]);
if (expanded == NULL)
    return 1;
free((*words)[i]);
(*words)[i] = expanded;
```

따라서 pipeline 내부의 앞선 word가 이미 확장된 뒤 뒤 word에서 failure가 날 수 있다. 이 commit이 모든 ephemeral field를 byte-for-byte 원상복구하는 것은 아니다. 대신 그 pipeline을 **실행하지 않고**, command-line lifetime 끝에서 parsed tree 전체를 해제한다. Persistent environment·cwd·running state에는 publish된 변화가 없다.

이 구분이 중요하다.

- Persistent state: failure 전의 유효 값을 유지해야 한다.
- 이번 line 전용 parsed state: partially transformed되어도 실행되지 않고 완전히 폐기되면 된다.

### top-level propagation: allocation failure와 syntax error를 구분한다

`shell_process_line`은 lexer/parser가 반환한 error text와 `errno == ENOMEM`을 보고 status를 정한다.

```c
static int process_error_status(const char *error)
{
    if (error != NULL && strcmp(error, "allocation failure") == 0)
        return 1;
    return 258;
}
```

- syntax error → status 258
- allocation failure → status 1
- expansion failure → diagnostic 후 status 1
- startup environment import failure → `main`이 diagnostic 후 1로 종료

Running shell에서는 한 command의 allocation failure가 utility-level `exit`로 바뀌지 않는다. 다음 command를 읽을 수 있는 상태라면 loop가 계속된다.

### 이 commit이 완성한 것과 남긴 것

완성:

- low-level helper는 nullable result를 반환한다.
- partial object는 caller가 cleanup한다.
- persistent environment replacement는 allocate-before-free다.
- command-level allocation error는 status 1로 전파된다.

남음:

- executor table allocation과 `pipe()`의 side-effect 순서는 아직 완전히 transactional하지 않다.
- heredoc failure는 memory 외에 stdin position을 이미 바꿨을 수 있다.
- call position별 failure를 결정적으로 주입하는 scope/counter는 아직 없다.

## 3. OS 자원 획득 전에 메모리 준비를 끝낸다

### `6d95776ede59` — pipeline preparation 순서 변경

**중요도** `A` · **태그** `PROCESS, FAILURE, RISK`

Pipeline은 두 memory table과 여러 kernel resource를 사용한다.

```text
pipe table memory
PID table memory
N-1개의 pipe descriptor pair
N개의 child process
```

이전 순서는 pipe table을 할당하고 실제 pipe descriptors를 만든 뒤 PID table을 할당했다. PID table allocation이 실패하면 이미 여러 FD를 획득한 상태이므로 cleanup이 필요하고, 한 누락만 있어도 resource leak가 된다.

Fix는 모든 memory preparation을 side effect 전에 끝낸다.

```c
if (pipe_count > 0) {
    pipes = shell_calloc(pipe_count, sizeof(int[2]));
    if (pipes == NULL)
        goto alloc_error;
    for (i = 0; i < pipe_count; i++) {
        pipes[i][0] = -1;
        pipes[i][1] = -1;
    }
}

pids = shell_calloc(command_count, sizeof(pid_t));
if (pids == NULL)
    goto alloc_error;

/* 두 table 확보가 끝난 뒤에만 shell_pipe loop */
```

이 순서의 결과는 명확하다.

| 실패 지점 | live memory | live FD | child |
| --- | --- | --- | --- |
| pipe table allocation | 없음 | 없음 | 없음 |
| PID table allocation | pipe table | 없음 | 없음 |
| pipe creation | 두 table | 성공한 pipe prefix | 없음 |
| fork | 두 table | pipe graph | 성공한 child prefix |

Allocation failure만 보면 항상 side-effect-free preparation 구간에서 멈춘다. `alloc_error`는 table을 해제하고 status 1을 반환한다.

다만 이 commit 직후 pipe creation failure branch에는 preallocated PID table 해제가 빠져 있었다. 그 narrow leak는 Thread 03의 `6dff1ba86ba6`가 `free(pids)` 한 줄을 추가해 닫는다. 이 사실은 “순서를 개선했다”와 “모든 후속 failure label이 새 acquisition list를 반영했다”가 별도 검증 대상임을 보여 준다.

## 4. stdin position도 transaction state다

### `c30b39c0bcf8` — heredoc allocation failure 뒤 command boundary 복구

**중요도** `A` · **태그** `HEREDOC, FAILURE, RISK`

Heredoc 준비에서는 body buffer나 expanded line allocation이 실패하기 전에 stdin에서 여러 line을 이미 읽었을 수 있다. Partial body object를 해제하는 것만으로는 다음 command의 시작 위치가 복구되지 않는다.

```sh
cat <<ONE <<TWO
first
ONE
second
TWO
echo after
```

첫 body를 읽는 중 allocation failure가 나고 즉시 반환하면 `ONE`, `second`, `TWO`가 다음 command line reader에 노출된다. 메모리 transaction은 정리됐지만 input transaction은 깨진 상태다.

Fix는 failure 후에도 다음을 수행한다.

- 현재 heredoc의 delimiter까지 남은 line을 discard한다.
- 이후 redirection에 남은 heredoc들도 source order대로 delimiter까지 discard한다.
- delimiter matching은 새 allocation 없이 encoded target의 marker를 건너뛴다.

이 commit이 allocation Thread에 포함되는 이유는 분명하다. Failure rollback 범위가 heap object를 넘어 **이미 발생한 input side effect**까지 확장되기 때문이다.

후속 `2d3791748571`은 EOF와 read failure를 구분하고, recovery read까지 실패하면 shell을 중단한다. 그 commit 자체는 I/O contract이므로 Thread 02에 포함하지만, `476b082d55c7`의 persistent-input allocation test가 안전하게 동작하려면 이 cross-thread 보강도 필요하다.

## 5. phase와 call position을 순회해 두 개의 coherent outcome만 허용한다

### `476b082d55c7` — allocation sweep

**중요도** `A` · **태그** `TEST, FAILURE, RISK`

이 commit은 test build의 allocator wrapper에 다음 상태를 추가한다.

- command number
- allocation scope
- 해당 scope 안의 call count
- one-shot 또는 repeat mode

Product path는 phase가 바뀔 때 scope를 명시한다.

```text
command-input
  → token
  → parser
  → heredoc / input
  → expand
  → execute
```

`shell_runtime_begin_command`는 command number를 증가시키고 allocation call count를 초기화한다. `shell_runtime_set_alloc_scope`는 같은 allocator를 호출해도 어느 processing phase인지 구분한다.

Test 환경 변수는 다음 조합으로 failure를 선택한다.

```text
SMALL_SHELL_FAIL_ALLOC_COMMAND=<몇 번째 command>
SMALL_SHELL_FAIL_ALLOC_SCOPE=<token|parser|expand|input|heredoc|execute>
SMALL_SHELL_FAIL_ALLOC=<scope 안의 N번째 call>
SMALL_SHELL_FAIL_ALLOC_REPEAT=1   # N번째 이후 계속 실패
```

Production build에서는 scope setter가 no-op이고 wrapper는 raw allocator를 호출한다.

### `sweep`가 허용하는 결과

`tests/allocation.sh`는 call index 1부터 phase별 maximum까지 같은 입력을 반복한다. 각 실행의 stdout는 정확히 둘 중 하나여야 한다.

```text
1. selected allocation이 실제 경로 안에 있음
   → 해당 command는 실패
   → payload/state mutation 없음
   → status 1
   → following command는 정상 실행

2. selected call index가 실제 allocation 수를 넘음
   → command 전체 성공
   → 정상 payload/state mutation
   → status 0
```

Script는 failure outcome과 success outcome이 각각 적어도 한 번 나타났는지도 검사한다. 예상하지 못한 partial output은 허용하지 않는다.

### phase별 대표 assertion

| Scope | 입력/상태 | 실패 outcome | 성공 outcome | 보호하는 invariant |
| --- | --- | --- | --- | --- |
| `token` | `echo marker` | marker 없음, status 1, `after` 출력 | marker, status 0, `after` | partial token list가 실행되지 않음 |
| `parser` | 같은 입력 | payload 없음 | 정상 payload | partial parse tree가 실행되지 않음 |
| `expand` | expansion 경로 | payload 없음 | 정상 payload | partial expanded pipeline 폐기 |
| `input` | heredoc line read buffer | body 출력 없음, status 1, `after` | body, status 0, `after` | input boundary 복구 |
| `heredoc` | quoted/unquoted/multiple | body publish 없음 | 정책에 맞는 body | entry/body transaction과 quote 의미 |
| `execute` builtin | `ALLOC_SWEEP=old` 후 export | status 1, old 유지 | status 0, new | persistent environment atomicity |
| `execute` external | `true` | status 1, `after` | status 0, `after` | envp/table setup failure 전 child 미실행 |

### persistent failure

One-shot mode는 target command에서 선택한 한 번만 실패하고, 이후 allocation은 다시 성공할 수 있다. Repeat mode는 같은 target command의 N번째 allocation부터 계속 실패시켜, cleanup이나 입력 경계 복구가 새 allocation을 요구할 때도 실패하도록 만든다. Command number filter가 있으므로 다른 command까지 무조건 실패시키는 전역 OOM 모드는 아니다.

Test는 두 경계를 별도로 본다.

- persistent heredoc input allocation failure: delimiter boundary를 신뢰할 수 없으면 `echo never`를 실행하지 않고 exit 1
- persistent token allocation failure: command를 구성할 수 없으므로 diagnostic을 남기고 output 없이 exit 1

이는 “항상 계속 실행”이 목표가 아님을 보여 준다. 계속 실행할 조건은 다음 command boundary와 persistent state가 신뢰 가능할 때뿐이다.

### 이 test가 증명하지 않는 것

- 모든 possible allocation count와 무한히 긴 command
- allocator 내부 fragmentation 또는 실제 system-wide OOM behavior
- leak 자체를 instrumentation으로 검출하는 것
- 모든 subsystem의 allocation을 하나도 빠짐없이 wrapper로 통과시켰다는 형식적 증명

후속 sanitizer build는 이 동일 suite를 ASan 아래 다시 실행할 수 있게 하지만, 그것은 Thread 05의 build/test 경로에서 다룬다.

## transaction 유형별 최종 규칙

### 1. 새 object construction

```text
allocate pieces
  → initialize/validate
  → all pieces complete
  → parent/list에 publish
failure before publish
  → local pieces free
```

적용: token node, redirection node, environment node, heredoc entry, envp vector.

### 2. 기존 persistent value replacement

```text
allocate new value
  → allocation success
  → free old value
  → pointer replace
failure
  → old value untouched
```

적용: existing environment entry update.

### 3. ephemeral parsed field replacement

```text
expand one field into new buffer
  → success 시 해당 field replace
later field failure
  → pipeline 실행 금지
  → whole parsed line lifetime 끝에서 tree free
```

이는 full rollback이 아니라 **unpublished ephemeral result의 폐기**다.

### 4. side-effect-free execution preparation

```text
allocate pipe table + PID table
  → only then create pipes
  → only then fork children
```

Allocation failure에는 FD와 child가 존재하지 않는다.

### 5. input-consuming preparation

```text
read heredoc body
  → allocation failure
  → local body free
  → current/remaining delimiter까지 input drain
  ├─ drain 성공: status 1, continue 가능
  └─ drain 실패: future boundary 불명, stop shell
```

## failure → owner → terminal state

| 실패 지점 | partial owner | cleanup/rollback | shell-visible 결과 |
| --- | --- | --- | ---: |
| string duplicate | caller local | local buffer 또는 node free | 1 |
| token append | tokenizer | token prefix free | 1 |
| parser append | parser | current + published prefix free | 1 |
| environment replacement | environment helper | 새 copy만 버리고 old 유지 | builtin 1 |
| expansion | selected pipeline lifetime | 실행하지 않고 tree 폐기 | pipeline 1 |
| pipe/PID table allocation | executor | allocated table free, FD/child 없음 | 1 |
| heredoc body/entry allocation | heredoc preparation | partial body free + input drain | 1 |
| recovery input allocation도 반복 실패 | shell input owner | 안전한 boundary 없음 | 1, `running=0` |

## 최종 흐름

```text
[command input]
  ↓ begin command + scope=token
[token construction]
  - 완성 node만 list publish
  ↓ scope=parser
[parsed hierarchy construction]
  - argv/redirection/pipeline partial cleanup
  ↓ scope=heredoc/input
[heredoc preparation]
  - complete entry만 publish
  - failure면 delimiter boundary 복구
  ↓ connector gate
  ↓ scope=expand
[selected pipeline expansion]
  - failure면 실행 금지, ephemeral tree 폐기
  ↓ scope=execute
[execution preparation]
  - all memory tables before pipe/fork
  - environment replacement allocate-before-free
  ↓
[status 0/command result 또는 allocation status 1]
```

## 이 Thread의 경계

- Parser의 정상 소유 계층과 connector gating은 Thread 01의 주제다. 여기서는 그 구조가 failure 시 어떻게 폐기되는지만 다룬다.
- Heredoc quote provenance와 read-failure recovery의 전체 의미는 Thread 02에서 다룬다. 여기서는 allocation이 input position에 미치는 영향만 포함한다.
- Pipe/fork/wait 이후의 PID·FD cleanup은 Thread 03에 속한다. 이 문서는 OS 자원 획득 **전** allocation 순서에 집중한다.
- Growable string builder의 성능·overflow·take/discard abstraction은 Thread 05다.

### 검증 범위

표시된 SHA의 diff와 해당 시점 source·test script를 `c/minishell` branch에서 확인했다. Repository를 로컬 checkout할 수 없어 allocation sweep과 sanitizer를 다시 실행하지 않았으며, 실행 통과를 새로 주장하지 않는다.
