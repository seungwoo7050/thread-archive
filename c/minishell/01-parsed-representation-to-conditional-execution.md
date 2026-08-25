# Thread: 인용 의미를 보존한 파싱 표현에서 조건부 실행까지

Project: `small-shell` · Branch: `c/minishell` · 문서 번호: 01

## 개요

이 Thread는 한 줄의 입력이 단순한 문자열에서 실행 가능한 구조로 바뀌는 동안, 서로 다른 세 가지 정보를 섞지 않는 과정을 다룬다.

1. lexer는 quote 문법이 만든 **문자 단위의 의미**를 token에 남긴다.
2. parser는 token을 **command → pipeline → connector-linked sequence**로 묶어 연산자 결합 규칙과 소유권을 고정한다.
3. executor는 직전 종료 상태로 pipeline을 먼저 선택한 뒤, **선택된 pipeline만 현재 shell state로 확장하고 실행**한다.

따라서 `a | b && c`는 세 command의 평평한 목록이 아니다. `a | b`가 하나의 pipeline이고, `&&`가 그 완성된 pipeline과 `c`의 pipeline을 연결한다. 또한 `c`가 short-circuit로 건너뛰어지면 그 내부의 `$?`나 환경 변수도 확장되지 않는다.

### 최종 표현과 수명

```text
source line
  └─ owned token list
       └─ pipeline list
            ├─ pipeline 0: command 0 ─pipe─ command 1
            │    next_op = CONN_AND
            └─ pipeline 1: command 0

실행 시:
  gate 판단 → 선택된 pipeline만 확장 → parent command 또는 forked pipeline 실행
```

| 객체 | 직접 소유하는 것 | 해제 주체 |
| --- | --- | --- |
| `t_token` | 복제된 `text`, source offset, 다음 token | `free_tokens` |
| `t_command` | `argv`의 각 문자열과 배열, 순서가 있는 redirection list | `free_commands` |
| `t_pipeline` | command list, command count, 다음 connector와 pipeline link | `free_pipeline` |
| `t_shell` | 여러 입력 줄에 걸쳐 유지되는 environment, `last_status`, `running` | top-level shell lifetime |

## 커밋 지도

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `729a6d2a7d4a` | `feat(lexer): 인용 단어와 토큰 수명 관리` | A | `LEX_PARSE`, `EXPANSION`, `CORE` | 입력 줄과 분리된 word token에 quote 효과를 남김 |
| 2 | `48670b845d7f` | `feat(parser): 명령 트리 소유권 모델 정의` | S | `ARCH`, `LEX_PARSE`, `CORE` | command·redirection·pipeline·connector의 소유 계층을 정의 |
| 3 | `a209a95a84d3` | `feat(parser): 인자와 리다이렉션 구문 구성` | B | `LEX_PARSE`, `CORE` | command 내부의 argv와 ordered redirection을 채움 |
| 4 | `8624028b83bb` | `feat(parser): pipe로 명령을 pipeline에 결합` | A | `LEX_PARSE`, `CORE`, `INTEGRATION` | pipe가 command 경계이고 같은 pipeline 안에서 결합됨을 구현 |
| 5 | `f297aaad70fe` | `feat(parser): 조건 연결자를 sequence로 결합` | S | `ARCH`, `LEX_PARSE`, `CORE` | `;`, `&&`, `||`가 완성된 pipeline을 연결하도록 표현을 확장 |
| 6 | `13a70b408e89` | `feat(exec): 조건 연결자와 지연 확장 실행` | S | `ARCH`, `EXPANSION`, `CORE` | gate를 통과한 pipeline만 현재 상태로 확장·실행 |
| 7 | `91ded56b033d` | `feat(shell): 한 줄 해석과 실행 수명 연결` | B | `INTEGRATION`, `CORE` | tokenize·parse·execute·cleanup을 한 입력 줄의 수명으로 연결 |

## `729a6d2a7d4a` — quote delimiter가 사라진 뒤에도 의미를 남긴다

**중요도** `A` · **태그** `LEX_PARSE, EXPANSION, CORE`

이 커밋은 lexer가 source line의 일부를 가리키는 view가 아니라, 자체 문자열을 소유하는 token list를 만들도록 한다. 핵심은 quote 문자를 결과 문자열에 그대로 남기는 것이 아니라, **후속 expansion에 필요한 효과만 내부 표현으로 보존**한다는 점이다.

단일 인용부의 각 문자는 `LITERAL_MARK`와 실제 문자 두 byte로 기록된다. 이 marker는 나중에 expansion 단계가 `$`를 일반 문자로 취급해야 함을 알려 줍니다. 반면 이 프로젝트의 단순화된 double-quote 경로는 delimiter만 제거하고 내부 문자를 marker 없이 복사하므로 `$NAME`과 `$?`가 후속 단계에서 확장될 수 있다.

```c
#define LITERAL_MARK '\001'

static char *append_literal(char *word, char c)
{
    word = append_char(word, LITERAL_MARK);
    if (word == NULL)
        return NULL;
    return append_char(word, c);
}
```

`read_word`는 unquoted fragment, single-quoted fragment, double-quoted fragment를 한 token text에 이어 붙이다. 예를 들어 다음 입력은 하나의 word token이 된다.

```text
ab'$X'"-$Y"
```

개념적으로 저장되는 값은 다음과 같습니다.

```text
'a' 'b' MARK '$' MARK 'X' '-' '$' 'Y' '\0'
```

여기서 source line과 token text는 별도 allocation이다. 따라서 top-level code가 입력 줄을 해제해도 token은 유효하다. 닫히지 않은 quote를 만나면 현재 word를 버리고, caller는 이미 publish된 token prefix까지 `free_tokens`로 정리한다.

이 시점의 보장은 제한적이다. 일반 word expansion에 필요한 single-quote 효과는 남지만, “quote 문법이 한 번이라도 사용되었는가”라는 별도 provenance는 아직 없다. 이 차이는 heredoc delimiter에서 문제가 되어 Thread 02의 `854f0f435c82`가 별도 필드를 추가한다.

## `48670b845d7f` — 실행기가 소비할 구조와 해제 순서를 먼저 정한다

**중요도** `S` · **태그** `ARCH, LEX_PARSE, CORE`

이 커밋은 구문을 채우기 전에 소유권 골격부터 정의한다.

```c
typedef struct s_redir {
    t_redir_type      type;
    char              *target;
    struct s_redir    *next;
} t_redir;

typedef struct s_command {
    char                **argv;
    size_t              argc;
    t_redir             *redirs;
    struct s_command    *next;
} t_command;

typedef struct s_pipeline {
    t_command             *commands;
    size_t                command_count;
    t_connector           next_op;
    struct s_pipeline     *next;
} t_pipeline;
```

중요한 점은 `pipeline`이 단순히 pipe를 뜻하는 file descriptor 구조가 아니라, parser와 executor가 공유하는 **명령 결합 단위**라는 점이다.

- `t_command`는 한 실행 stage의 argv와 redirection 순서를 소유한다.
- `t_pipeline`은 pipe로 이어질 command list를 소유한다.
- `next_op`는 이 pipeline이 다음 pipeline과 어떤 관계인지 저장한다.
- `next`는 sequence를 별도 wrapper 없이 linked list로 표현한다.

해제는 leaf에서 root 방향으로 진행된다.

```text
argv strings / redirection target
        ↓
argv vector / redirection nodes
        ↓
command nodes
        ↓
pipeline nodes
```

이 구조 덕분에 parser가 어느 단계에서 실패하더라도 “현재 command”, “아직 sequence에 publish되지 않은 pipeline”, “이미 완성된 pipeline prefix”를 구분해 각각 해제할 수 있다. 이후 commit들이 구현을 채우지만, 소유권 규칙 자체는 여기서 이미 결정된다.

### 이 커밋이 정한 불변 조건

- parsed structure는 token text를 빌려 쓰지 않고 자체 문자열을 소유한다.
- redirection은 source order를 보존하는 linked list이다.
- pipe는 같은 pipeline의 command를 늘립니다.
- `;`, `&&`, `||`는 완성된 pipeline 사이의 관계이다.
- `free_pipeline` 하나로 sequence 전체를 재귀적이지 않은 반복 해제할 수 있다.

## `a209a95a84d3` — command 내부를 채우되 부분 결과를 공개하지 않는다

**중요도** `B` · **태그** `LEX_PARSE, CORE`

`add_arg`는 기존 argv를 바로 `realloc`하지 않는다. 새 vector와 새 word 복사가 모두 성공한 뒤 기존 pointer들을 옮기고 `cmd->argv`를 교체한다.

```c
next = sh_xcalloc(n + 2, sizeof(char *));
copy = sh_strdup(text);
for (i = 0; i < n; i++)
    next[i] = cmd->argv[i];
next[n] = copy;
free(cmd->argv);
cmd->argv = next;
cmd->argc = n + 1;
```

이 SHA의 helper는 아직 allocation failure에서 process를 끝내는 초기 정책을 사용하지만, **publish 순서**는 이미 이후의 nullable allocation 모델과 맞습니다. Thread 04의 `0bb6f9de0947`는 같은 순서를 유지하면서 failure를 return 값으로 전파하도록 바꿉니다.

redirection은 operator 뒤의 word를 필수 target으로 요구하고, 생성 순서대로 command의 tail에 연결한다. 이 순서는 실행 단계에서 중요하다. 동일한 fd를 여러 번 바꾸는 다음 입력은 source order대로 적용되어야 하기 때문이다.

```sh
echo x > first > second
```

이 커밋의 parser는 아직 command 하나와 pipeline 하나만 만든다. pipe와 connector의 결합은 다음 두 커밋에서 분리해 추가된다.

## `8624028b83bb` — `|`는 새 pipeline이 아니라 새 command를 만든다

**중요도** `A` · **태그** `LEX_PARSE, CORE, INTEGRATION`

`TOK_PIPE`를 만나면 parser는 지금까지 채운 command를 현재 pipeline에 append하고, 다음 stage를 위한 빈 command를 만든다.

```c
if (cur->type == TOK_PIPE) {
    if (command_empty(cmd))
        return parse_failure(..., "syntax error: empty command before pipe");
    append_command(pipeline, cmd);
    cmd = new_command();
    after_pipe = 1;
}
```

다음 두 오류가 서로 다른 지점에서 검출된다.

- `| echo`: pipe 앞의 command가 비어 있으므로 즉시 실패한다.
- `echo |`: token loop가 끝난 뒤 `after_pipe`가 남아 있으므로 “pipe 뒤 command 필요”로 실패한다.

`command_empty`는 argv뿐 아니라 redirection도 본다. 따라서 redirection-only command는 이 프로젝트 표현에서 유효한 stage가 될 수 있다.

```text
command_empty = (argc == 0 && redirs == NULL)
```

이 커밋 뒤 `a | b | c`는 다음처럼 표현된다.

```text
pipeline
  command_count = 3
  commands: a → b → c
  next_op = CONN_NONE
```

아직 `&&`, `||`, `;`는 pipeline list를 만들지 않는다.

## `f297aaad70fe` — connector는 왼쪽의 완성된 pipeline에 붙는다

**중요도** `S` · **태그** `ARCH, LEX_PARSE, CORE`

이 커밋은 parser의 최종 결합 규칙을 정한다. connector를 만나면 현재 command를 마감하고 현재 pipeline을 sequence tail에 publish한다. connector 종류는 **왼쪽 pipeline의 `next_op`**에 저장된다.

```c
pipeline->next_op = connector_type(cur->type);
append_pipeline(&head, &tail, pipeline);

pipeline = new_pipeline();
cmd = new_command();
```

따라서 다음 입력은 아래 구조가 된다.

```sh
a | b && c ; d || e | f
```

```text
P0: [a, b]  next_op=AND
P1: [c]     next_op=SEQ
P2: [d]     next_op=OR
P3: [e, f]  next_op=NONE
```

이 표현은 full POSIX AST는 아니지만, 이 프로젝트가 지원하는 우선순위를 직접 보존한다. pipe는 command를 묶고 connector는 pipeline을 묶으므로 별도의 precedence 재계산이 필요하지 않는다.

### 끝 connector 처리

- trailing `&&`와 `||`는 다음 pipeline이 반드시 필요하므로 syntax error이다.
- trailing `;`는 허용하고, 이미 publish된 tail의 `next_op`을 `CONN_NONE`으로 되돌립니다.
- `a | && b`처럼 pipe 직후 connector가 오면 `after_pipe` 검사가 먼저 실패한다.

### 실패 시 소유권

`parse_failure`는 세 종류의 partial state를 모두 받습니다.

```text
이미 publish된 head sequence
현재 미완성 pipeline
현재 미완성 command
```

실패하면 current command, current pipeline, published head를 각각 해제한다. 정상 종료에서만 완성된 pipeline이 head list에 남습니다.

## `13a70b408e89` — short-circuit 결정 뒤에 expansion을 둔다

**중요도** `S` · **태그** `ARCH, EXPANSION, CORE`

parser가 구조를 정했다면 executor는 상태를 사용해 어느 pipeline을 실행할지 결정한다. 이 커밋의 핵심은 expansion을 parse 직후 전체 sequence에 적용하지 않는 것이다.

```c
previous = CONN_NONE;
while (pipeline != NULL && shell->running) {
    should_run = 1;
    if (previous == CONN_AND && shell->last_status != 0)
        should_run = 0;
    else if (previous == CONN_OR && shell->last_status == 0)
        should_run = 0;
    if (should_run)
        shell->last_status = execute_one_pipeline(shell, pipeline, ctx);
    previous = pipeline->next_op;
    pipeline = pipeline->next;
}
```

`previous`는 직전에 방문한 pipeline의 connector이다. 선택된 pipeline을 실행한 뒤에만 `shell->last_status`가 바뀐다. 건너뛴 pipeline은 status를 덮어쓰지 않는다.

### 왜 pipeline을 잠깐 분리하는가

기존 `expand_pipeline`은 linked list의 `next`를 따라 여러 pipeline을 처리할 수 있다. 선택된 하나만 확장하려면 호출 전후에 link를 잠시 떼어야 한다.

```c
next = pipeline->next;
pipeline->next = NULL;
result = expand_pipeline(shell, pipeline);
pipeline->next = next;
```

이 작은 detach/restore가 지연 확장을 실제로 보장한다. 이를 생략하면 현재 branch만 선택했더라도 뒤 pipeline까지 확장된다.

### 상태가 달라지는 예

```sh
false && echo "$?" ; echo "$?"
```

- 첫 pipeline이 실패해 status가 non-zero가 된다.
- `&&` 오른쪽 pipeline은 선택되지 않으며 그 argv도 확장되지 않는다.
- `;` 뒤 pipeline은 실행되고, 그때의 `$?`는 앞에서 유지된 실패 상태를 읽습니다.

즉 expansion 결과는 source text만의 함수가 아니라 **선택 시점의 shell state**에도 의존한다. 이 때문에 gate와 expansion 순서는 바꿀 수 없다.

### 보장 범위

이 커밋은 connector gating과 expansion timing을 정한다. 실제 fork, pipe FD, parent builtin redirection 같은 실행 자원은 Thread 03의 관심사이다.

## `91ded56b033d` — 입력 한 줄의 임시 객체를 한곳에서 닫는다

**중요도** `B` · **태그** `INTEGRATION, CORE`

`shell_process_line`은 앞선 표현과 실행 결정을 product path에 연결한다.

```c
tokens = tokenize_line(line, &error);
pipelines = parse_tokens(tokens, &error);
free_tokens(tokens);

execute_pipeline_list(shell, pipelines);
free_pipeline(pipelines);
```

여기서 token을 parse 직후 해제할 수 있다는 사실은 parser가 필요한 문자열을 모두 복제해 소유한다는 증거이다. pipeline은 실행이 끝날 때까지 유지되며, expansion은 그 내부 문자열을 교체한다. 반면 environment와 status는 `t_shell`에 남아 다음 입력 줄로 이어진다.

syntax error는 실행하지 않고 status `258`을 남긴다. 빈 line 또는 token이 없는 line은 기존 status를 유지한다. 이후 heredoc이 도입되면 이 함수의 수명 안에 `exec_context`와 precollection 단계가 추가되지만, tokenize → parse → execute → cleanup이라는 outer lifetime은 그대로 유지된다.

## 불변 조건 정리

| 불변 조건 | 처음 표현된 커밋 | 완성한 커밋 | 코드 근거 |
| --- | --- | --- | --- |
| quote delimiter가 없어져도 single-quoted byte는 확장되지 않음 | `729a6d2a7d4a` | 후속 expansion commits가 소비 | literal marker pair |
| parsed object는 token/input line과 독립된 수명을 가짐 | `48670b845d7f` | `91ded56b033d` | parse 후 token 즉시 해제 |
| pipe는 command를, connector는 pipeline을 결합 | `8624028b83bb` | `f297aaad70fe` | command append와 pipeline append가 별도 branch |
| skipped pipeline은 확장도 실행도 하지 않음 | — | `13a70b408e89` | gate 뒤 `expand_one_pipeline` 호출 |
| connector 판단은 직전 실행 status를 사용 | — | `13a70b408e89` | `previous`와 `shell->last_status` |
| 한 줄의 temporary parse state는 실행 뒤 전부 해제 | `48670b845d7f` | `91ded56b033d` | `free_tokens`, `free_pipeline` |

## 최종 실행 흐름

```text
[shell_loop가 line 획득]
  ↓
[tokenize_line]
  - quote delimiter 제거
  - single-quoted byte에 literal marker
  - token text를 자체 소유
  ↓
[parse_tokens]
  - word/redirection → current command
  - | → current pipeline에 command 마감
  - ;/&&/|| → current pipeline을 sequence에 마감
  ↓
[free_tokens]
  ↓
[execute_pipeline_list_ctx]
  - previous connector + last_status로 gate 판단
  - selected pipeline만 일시 분리하여 expand
  - 실행 후 last_status 갱신
  ↓
[free_pipeline]
  ↓
[environment, last_status, running만 다음 line으로 유지]
```

## 이 Thread의 경계

이 문서는 parsed representation과 조건부 실행 선택만 다룬다.

- heredoc delimiter provenance와 connector gating 전 body 수집은 Thread 02이다.
- child PID, pipe FD, parent stdio 복원은 Thread 03이다.
- allocation failure를 process exit가 아닌 command failure로 바꾸는 작업은 Thread 04이다.
- 반복 문자열 결합의 점근적 비용을 줄이는 builder migration은 Thread 05이다.

### 검증 범위

표시된 각 SHA의 diff와 해당 시점 source를 `c/minishell` branch에서 확인했다. 이 작업 환경에서는 checkout/build가 불가능해 executable test를 다시 실행하지 않았으며, 실행 결과를 새로 주장하지 않는다.
