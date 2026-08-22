===== BEGIN FILE: 01-parsed-representation-to-conditional-execution.md =====
# Parsed representation to conditional execution

> 한국어 주제: **파싱 표현에서 조건부 실행까지**
>
> Project: `small-shell`
> Branch: `c/minishell`
> Development Thread order: 1/5

## 1. Thread 목표

인용 의미가 보존된 token이 소유권이 명확한 command/처리 단계 구조로 변환되고, pipe와 조건 연결자의 결합 규칙을 유지한 채 현재 status를 기준으로 선택된 처리 단계만 확장·실행되는 과정을 복원합니다.

**Source-defined significance**

> The progression separates three concerns that a shell must not conflate: lexical quote meaning, structural binding, and runtime 제어 흐름. The decisive commits are the 소유권 model, sequence representation, and delayed executor; the surrounding commits populate or integrate those choices. This thread explains why pipes bind within a 처리 단계, why conditional connectors link 처리 단계, and why expansion occurs only after a branch is selected.

**학습 관점**

이 흐름은 lexical quote 의미, 구조적 결합, runtime control flow를 분리합니다. 핵심은 command tree 소유권, complete 처리 단계 단위의 sequence 표현, 그리고 branch 선택 뒤에 수행되는 delayed expansion입니다.

### SHA 고정 원칙

- 각 commit은 반드시 표시된 exact SHA 또는 그 parent와 비교합니다.
- 먼저 `git show --name-status <SHA>`로 변경 파일을 식별한 뒤, 필요한 path만 `git diff <SHA>^ <SHA> -- <path>`로 봅니다.
- 실제 구현은 `git show <SHA>:<path>` 또는 detached worktree에서 확인합니다.
- final HEAD의 type, function, test를 과거 commit 설명에 소급하지 않습니다.
- later commit의 field나 fix가 아직 존재하지 않는 SHA에서는 그 부재 자체를 기록합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 입력 줄이 해제된 뒤에도 quote 효과가 token에 남도록 하는 표현은 무엇입니까?
- argv, redirection, command, 처리 단계는 각각 무엇을 소유하며 어느 cleanup 경로가 전체 구조를 해제합니까?
- pipe가 command 내부 결합이 아니라 처리 단계 구성 경계가 되는 코드는 어디입니까?
- `;`, `&&`, `||`가 complete 처리 단계를 연결한다는 사실은 자료구조와 parser state에 어떻게 나타납니까?
- 왜 전체 line을 먼저 확장하지 않고, connector gate를 통과한 처리 단계만 현재 `$?`로 확장합니까?
- 한 줄 처리에서 token, parsed structure, expanded fields, persistent shell state의 수명은 어디서 나뉩니까?

## 3. 완료 기준

- [x] 각 commit의 exact SHA에서 변경된 구조체와 핵심 함수의 caller/callee를 기록했습니다.
- [x] `source line → token → command/pipeline list → selected pipeline expansion → execution → cleanup` 흐름을 코드 근거로 설명했습니다.
- [x] pipe와 sequence connector의 binding 차이를 예제 입력과 parser 코드로 연결했습니다.
- [x] skipped 처리 단계가 확장되지 않는 branch와 `$?`가 갱신되는 순서를 확인했습니다.
- [x] S commit마다 소유권 graph, 실패 처리, 후속 연결을 작성했습니다.

> 실행 범위: exact SHA의 commit diff와 해당 시점 source를 GitHub repository에서 검사했습니다. 이 실행 환경에서는 branch checkout이 불가능해 binary build와 runtime command는 수행하지 않았습니다. 아래의 실행 결과 항목은 모두 이를 명시합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `729a6d2a7d4a` | `feat(lexer): 인용 단어와 토큰 수명 관리` | A | `LEX_PARSE`, `EXPANSION`, `CORE` | Preserves quote effects in owned word tokens. |
| 2 | `48670b845d7f` | `feat(parser): 명령 트리 소유권 모델 정의` | S | `ARCH`, `LEX_PARSE`, `CORE` | Establishes the command, redirection, pipeline, connector, and cleanup ownership hierarchy. |
| 3 | `a209a95a84d3` | `feat(parser): 인자와 리다이렉션 구문 구성` | B | `LEX_PARSE`, `CORE` | Populates commands and ordered redirections inside that model. |
| 4 | `8624028b83bb` | `feat(parser): pipe로 명령을 pipeline에 결합` | A | `LEX_PARSE`, `CORE`, `INTEGRATION` | Defines a pipeline as an ordered command group and validates pipe boundaries. |
| 5 | `f297aaad70fe` | `feat(parser): 조건 연결자를 sequence로 결합` | S | `ARCH`, `LEX_PARSE`, `CORE` | Extends the representation to a connector-linked sequence of complete pipelines. |
| 6 | `13a70b408e89` | `feat(exec): 조건 연결자와 지연 확장 실행` | S | `ARCH`, `EXPANSION`, `CORE` | Executes that sequence with short-circuiting and status-correct delayed expansion. |
| 7 | `91ded56b033d` | `feat(shell): 한 줄 해석과 실행 수명 연결` | B | `INTEGRATION`, `CORE` | Connects the concrete parse and execution lifetime to each input line. |

## 5. Commit별 학습 기록

### 5.1 `729a6d2a7d4a` — `feat(lexer): 인용 단어와 토큰 수명 관리`

#### 확정 정보
- SHA: `729a6d2a7d4a`
- Subject: `feat(lexer): 인용 단어와 토큰 수명 관리`
- Importance: **A**
- Tags: `LEX_PARSE`, `EXPANSION`, `CORE`
- Source-defined role: Preserves quote effects in owned word tokens.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
owned word token을 도입하고, quote delimiter를 제거한 뒤에도 single-quoted byte는 내부 literal marker로 보존합니다. token은 text와 source offset을 소유하며, unclosed quote에서는 이미 생성된 token list를 해제합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: scanner에는 operator token만 있었고, 인용된 단어를 입력 줄과 독립된 owned token으로 남기는 표현이 없었습니다.
- 해결하려던 문제: quote delimiter를 제거한 뒤에도 single quote 안의 `$` 같은 byte가 later expansion에서 literal임을 알아야 하며, 입력 줄 해제 뒤에도 token text가 살아 있어야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: 최종 text만 보관하면 single-quoted byte와 확장 가능한 byte가 같아지고, source line pointer만 참조하면 line 수명에 종속됩니다.
- 선택한 결정: `TOK_WORD`와 owned `text`를 추가하고, single-quoted 각 byte를 `LITERAL_MARK('\001') + byte`로 인코딩합니다. Double quote는 delimiter만 제거하고 내부 byte는 marker 없이 둡니다.
- publish 또는 state mutation이 일어나는 지점: `read_word`가 완성한 allocation을 token node의 `text`로 넘기고, node가 완성된 뒤 token list tail에 연결합니다. `start`에는 source offset을 기록합니다.
- failure 뒤 cleanup 또는 상태: unclosed quote 또는 allocation failure면 current word를 해제하고 `tokenize_line`이 이미 연결한 token list를 `free_tokens`로 폐기합니다. Empty quoted word는 failure가 아니라 길이 0인 유효 word입니다.

#### `729a6d2a7d4a`에서 확인할 실제 코드
- `include/shell.h`의 token type/structure에 `text`, `start`, `next`가 있습니다.
- `src/token.c::read_word`는 unquoted, single-quoted, double-quoted fragment를 한 word allocation에 이어 붙입니다.
- single quote branch만 literal marker를 먼저 기록하고, double quote branch는 내부 byte를 그대로 기록합니다.
- quote 직후 닫힘을 만나도 empty string token을 반환합니다.
- unclosed quote branch는 local text를 free하고 error를 설정하며, caller가 prefix token list를 정리합니다.

#### 학습자가 남길 코드 증거
- 확인한 lexer entry 함수와 word-scanning helper: `src/token.c::tokenize_line` → `read_word`; 완성된 word는 word token 생성 helper를 통해 list에 연결됩니다.
- token이 public list에 연결되는 publish 지점: node와 node-owned `text`가 모두 성공한 뒤 tail의 `next`를 갱신하는 append branch입니다.
- single quote marker의 byte 표현과 생성 조건: `#define LITERAL_MARK '\001'`; `quote == '\''`인 동안 각 source byte 앞에 marker를 추가합니다.
- double quote에서 expansion 가능성을 남기는 코드: `quote == '"'`인 branch는 delimiter만 건너뛰고 내부 byte를 marker 없이 append합니다.
- unclosed quote 실패 처리의 해제 순서: local word free → error set → caller의 `free_tokens(prefix)`입니다.
- 확인한 변경 파일: `Makefile`, `include/shell.h`, `src/token.c`.
- 핵심 caller → callee: `tokenize_line` → `read_word` → append helpers → token append; cleanup은 `free_tokens`입니다.
- parent SHA와 비교한 최소 before/after snippet:

```c
/* 729a6d2a7d4a, src/token.c::read_word */
if (quote == '\'')
    word = append_literal(word, line[*i]);
else
    word = append_char(word, line[*i]);
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact SHA source와 diff로 `''`, `'$HOME'`, `"$HOME"`, unclosed quote의 branch와 cleanup만 검증했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: quote delimiter가 사라진 뒤에도 later expansion이 single-quoted byte를 literal로 구분할 수 있고, token 수명이 input line과 분리됩니다.
- 아직 보장하지 않는 것: operator binding, parser hierarchy, runtime expansion은 아직 보장하지 않습니다. Heredoc의 모든 quote provenance는 later fix에서 별도 field로 보강됩니다.

#### Thread 내 다음 연결
`48670b845d7f`에서 이 owned token의 데이터를 복사해 hierarchical parsed representation을 만듭니다.

### 5.2 `48670b845d7f` — `feat(parser): 명령 트리 소유권 모델 정의`

#### 확정 정보
- SHA: `48670b845d7f`
- Subject: `feat(parser): 명령 트리 소유권 모델 정의`
- Importance: **S**
- Tags: `ARCH`, `LEX_PARSE`, `CORE`
- Source-defined role: Establishes the command, redirection, 처리 단계, connector, and cleanup 소유권 hierarchy.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
처리 단계 → command → argv/redirection의 계층과 connector metadata를 정의하고, leaf부터 root까지 해제하는 recursive cleanup 소유권을 확립합니다.

#### Source가 확정한 핵심 판단
- **문제**: Raw tokens are insufficient for execution: the shell needs a stable representation of arguments, ordered redirections, command groups, connectors, and all associated 소유권.
- **결정**: Represent a line as 처리 단계 containing commands, commands containing argv and redirections, and connectors attached to complete 처리 단계. Mirror that hierarchy with one recursive cleanup 진입점.
- **중요한 이유**: Every later parser, expander, heredoc entry, executor allocation, and failure cleanup assumes these 소유권 boundaries. The model is compact enough for the supported grammar while still making partial and complete destruction deterministic.
- **확정된 변경 범위**: The commit introduced `t_redir`, `t_command`, `t_pipeline`, connector metadata, element counts, and leaf-to-root cleanup for argv strings, redirection targets, commands, and 처리 단계 nodes.
- **프로젝트 이해에서의 위치**: This is the data architecture of the shell. It explains what each phase owns, why later stages copy rather than retain tokens, and how errors can release an arbitrarily partial parse without subsystem-specific cleanup knowledge.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: owned token list만 존재했고, 실행 단위·redirection order·connector를 보관할 parsed hierarchy가 없었습니다.
- 해결하려던 문제: parser가 부분 생성 중 실패하더라도 argv string, vectors, redirection targets, commands, 처리 단계를 누락 없이 한 경로에서 해제해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: token list는 syntax 순서만 제공하고 어느 문자열이 argv인지, redirection target인지, 어느 command/처리 단계가 소유하는지 표현하지 못합니다.
- 선택한 결정: `t_pipeline`이 command list와 `command_count`, connector/next를 소유하고, `t_command`가 null-terminated argv와 ordered redirection list를, `t_redir`가 target string을 소유하도록 계층을 정의했습니다.
- publish 또는 state mutation이 일어나는 지점: 이후 parser가 완성된 child node를 각 parent list에 연결합니다. 이 commit에서는 zero-initialized nodes와 count fields가 안전한 publish 전 상태를 제공합니다.
- failure 뒤 cleanup 또는 상태: `free_pipeline`이 처리 단계 sequence를 순회하며 command의 argv strings/vector, redirection target/nodes, command nodes, 처리 단계 nodes를 leaf-to-root로 해제합니다. NULL field와 partial list도 허용합니다.

#### `48670b845d7f`에서 확인할 실제 코드
- `include/shell.h`의 `t_redir`, `t_command`, `t_pipeline`, connector enum과 각 owned pointer를 확인했습니다.
- argv vector는 command-owned이며 각 element도 command hierarchy가 소유합니다.
- redirection list의 node와 `target`은 command-owned입니다.
- connector는 command가 아니라 왼쪽 complete 처리 단계의 metadata입니다.
- `src/parser.c::free_pipeline`은 nested 소유권을 한 entry에서 정리합니다.

#### 학습자가 남길 코드 증거
- 구조체별 owner/owned object 표:

| Owner | Owned object | Non-owning/metadata |
| --- | --- | --- |
| `t_pipeline` | ordered `t_command` nodes, following 처리 단계 chain | `command_count`, `next_op` |
| `t_command` | `argv[]`와 각 string, ordered `t_redir` nodes | `argc`, next command link |
| `t_redir` | `target` string | redirection type, next link |

- count field가 later executor allocation에 제공하는 값: `pipeline->command_count`는 PID table 크기와 `N - 1` pipe count 계산에 사용됩니다.
- partial construction에서도 안전한 initial state: zero/NULL 초기화된 pointer, `argc == 0`, `command_count == 0`, empty lists입니다.
- recursive cleanup entry와 내부 call order: `free_pipeline` → command loop → argv elements/vector → redirection target/node → command → 처리 단계.
- token 수명과 parsed 수명이 갈리는 지점: later `parse_tokens`가 token text를 duplicate해 parsed field에 저장하므로 token list를 먼저 free할 수 있습니다.
- 확인한 변경 파일: `include/shell.h`, `src/parser.c` 및 build source list.
- 핵심 caller → callee: later `parse_tokens`가 constructors/append helpers를 사용하고 모든 error/normal cleanup이 `free_pipeline`로 수렴합니다.
- parent SHA와 비교한 최소 before/after snippet: parent에는 token types만 있었고, 이 SHA에서 `t_redir`, `t_command`, `t_pipeline`과 `free_pipeline(t_pipeline *)`가 새로 생깁니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact SHA의 type definition과 destructor body를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: published parsed tree 전체를 subsystem-specific knowledge 없이 하나의 hierarchy cleanup으로 폐기할 수 있습니다.
- 아직 보장하지 않는 것: 이 commit은 representation과 destructor를 정의하며, argv/redirection population, 처리 단계/sequence parsing, execution은 후속 commit에 남습니다.

#### Thread 내 다음 연결
`a209a95a84d3`, `8624028b83bb`, `f297aaad70fe`가 동일 ownership model 안에 data와 binding을 채웁니다.

### 5.3 `a209a95a84d3` — `feat(parser): 인자와 리다이렉션 구문 구성`

#### 확정 정보
- SHA: `a209a95a84d3`
- Subject: `feat(parser): 인자와 리다이렉션 구문 구성`
- Importance: **B**
- Tags: `LEX_PARSE`, `CORE`
- Source-defined role: Populates commands and ordered redirections inside that model.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
word token을 source order의 null-terminated argv로 복사하고, redirection operator와 뒤따르는 word를 ordered redirection list로 분리합니다. Redirection-only command는 보존하며 missing target은 syntax failure로 처리합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: hierarchy와 destructor만 있고 parser가 argv/redirection nodes를 채우지 않았습니다.
- 해결하려던 문제: word와 redirection target을 같은 토큰열에서 읽되 서로 다른 owned destination에 source order대로 publish해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: operator 뒤 word를 일반 argv로도 넣으면 command semantics가 깨지고, append 중 실패하면 기존 vector나 partial node를 잃을 수 있습니다.
- 선택한 결정: `add_arg`가 새 null-terminated vector를 준비해 기존 pointer들을 옮기고 새 string을 복제한 후 command에 publish하며, `add_redir`는 target을 duplicate한 complete node만 tail에 연결합니다.
- publish 또는 state mutation이 일어나는 지점: argv는 새 vector와 새 string이 준비된 후 `command->argv/argc`가 바뀌고, redirection은 node와 target 성공 후 list tail에 연결됩니다.
- failure 뒤 cleanup 또는 상태: missing target/unsupported operator는 구문 오류로 전환되고 current command/처리 단계를 hierarchy destructor로 정리합니다.

#### `a209a95a84d3`에서 확인할 실제 코드
- `src/parser.c::parse_tokens`, `add_arg`, `add_redir`를 확인했습니다.
- redirection operator branch가 다음 token을 target으로 소비해 main word branch를 건너뜁니다.
- list append는 source order를 유지합니다.
- argv가 없어도 redirection이 있으면 command는 유효합니다.
- empty 토큰열은 구문 오류가 아니라 no parse result입니다.

#### 학습자가 남길 코드 증거
- argv append 전/후 구조: old `argv[0..argc-1]` → new `argv[0..argc]` + final NULL; old vector만 free하고 strings는 새 vector로 이동합니다.
- redirection target 소유권 transfer: token text를 직접 보관하지 않고 duplicate한 `redir->target`을 node가 소유합니다.
- redirection-only command 판정: `argc == 0`이어도 redirection list가 있으면 parser가 command를 유지합니다.
- syntax failure가 반환되는 정확한 조건: redirection operator 뒤 word가 없거나 해당 시점에 지원하지 않는 operator가 나오는 경우입니다.
- partial command cleanup 함수: `free_pipeline`의 nested command/redirection cleanup입니다.
- 확인한 변경 파일: `src/parser.c`, parser API declarations가 있는 `include/shell.h`.
- 핵심 caller → callee: `parse_tokens` → `add_arg` 또는 `add_redir`; error → shared parse cleanup → `free_pipeline`.
- parent SHA와 비교한 최소 before/after snippet: representation-only state에서 actual token traversal과 ordered population이 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. `echo a > out b`, `> out`, `echo >`에 대응하는 code branch를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 한 command 안에서 argument와 redirection syntax가 분리되고 각 target/argument가 독립 소유됩니다.
- 아직 보장하지 않는 것: pipe로 여러 command를 결합하거나 connector sequence를 구성하지 않습니다.

#### Thread 내 다음 연결
`8624028b83bb`가 pipe를 command-finalization boundary로 추가합니다.

### 5.4 `8624028b83bb` — `feat(parser): pipe로 명령을 pipeline에 결합`

#### 확정 정보
- SHA: `8624028b83bb`
- Subject: `feat(parser): pipe로 명령을 pipeline에 결합`
- Importance: **A**
- Tags: `LEX_PARSE`, `CORE`, `INTEGRATION`
- Source-defined role: Defines a 처리 단계 as an ordered command group and validates pipe boundaries.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
`|`를 만나면 current command를 처리 단계에 append하고 새 command를 시작합니다. `after_pipe` 상태로 leading, repeated, trailing pipe를 거부하고 partial 처리 단계를 정리합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: parser는 하나의 command만 채울 수 있었습니다.
- 해결하려던 문제: `|` 좌우의 command를 독립 소유권 node로 확정하고 source-order 처리 단계에 넣어야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: pipe를 일반 operator처럼 command 안에 남기면 stage boundary와 later FD topology를 계산할 `command_count`가 없습니다.
- 선택한 결정: pipe를 current command publish 지점으로 사용하고 새 zero-initialized command를 시작하며 `after_pipe` state로 빈 stage를 거부합니다.
- publish 또는 state mutation이 일어나는 지점: current command가 처리 단계 tail에 연결될 때 `command_count`가 증가하고 current pointer가 새 command로 교체됩니다.
- failure 뒤 cleanup 또는 상태: leading/repeated/trailing pipe에서 in-progress command와 already attached command list를 모두 hierarchy cleanup으로 폐기합니다.

#### `8624028b83bb`에서 확인할 실제 코드
- pipe token branch의 command finalize/append/new command 순서를 확인했습니다.
- `after_pipe`가 pipe 직후 true가 되고 정상 word/redirection을 읽으면 해제됩니다.
- leading `|`, `||`가 아닌 repeated `| |`, trailing `cmd |`가 empty-stage 구문 오류로 수렴합니다.
- redirection-only stage도 non-empty command로 유지됩니다.

#### 학습자가 남길 코드 증거
- pipe 직전 current command state: argv/redirection 중 하나 이상을 가진 local command입니다.
- append 후 새 command initial state: old command는 처리 단계-owned, `command_count++`; 새 command는 NULL/0 fields입니다.
- 세 가지 malformed pipe input과 branch: `| a`는 current empty, `a | | b`는 `after_pipe`, `a |`는 end-of-input에서 `after_pipe`가 남아 error입니다.
- command_count의 later consumer 후보: `a71f98de0d92`의 PID table과 `N - 1` pipe table 계산입니다.
- failure cleanup이 보유한 두 소유권 영역: 처리 단계에 published prefix와 아직 local인 current command입니다.
- 확인한 변경 파일: `src/parser.c`, `include/shell.h`의 existing count field 사용.
- 핵심 caller → callee: `parse_tokens` pipe branch → command append helper → new command constructor; failure → shared cleanup.
- parent SHA와 비교한 최소 before/after snippet: single command return에서 ordered command list construction과 `after_pipe` 검증이 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. 세 malformed inputs와 `a | b`의 parser state를 exact code로 추적했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 처리 단계는 ordered command group으로 표현되고, 빈 stage를 가진 pipe syntax가 partial structure로 남지 않습니다.
- 아직 보장하지 않는 것: `;`, `&&`, `||`로 여러 처리 단계를 연결하는 sequence와 runtime short-circuit는 아직 없습니다.

#### Thread 내 다음 연결
`f297aaad70fe`가 complete pipeline을 connector-linked sequence unit으로 사용합니다.

### 5.5 `f297aaad70fe` — `feat(parser): 조건 연결자를 sequence로 결합`

#### 확정 정보
- SHA: `f297aaad70fe`
- Subject: `feat(parser): 조건 연결자를 sequence로 결합`
- Importance: **S**
- Tags: `ARCH`, `LEX_PARSE`, `CORE`
- Source-defined role: Extends the representation to a connector-linked sequence of complete 처리 단계.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
`;`, `&&`, `||`에서 current 처리 단계를 끝내고 connector를 왼쪽 처리 단계의 `next_op`에 저장한 뒤 새 처리 단계를 시작합니다. Pipe는 처리 단계 내부에 남아 더 강한 binding을 유지합니다.

#### Source가 확정한 핵심 판단
- **문제**: A single-처리 단계 parser cannot represent semicolon sequencing or conditional execution, and treating all operators alike would lose the stronger binding of pipes.
- **결정**: Finish a 처리 단계 at `;`, `&&`, or `||`, store the connector on its left 처리 단계, and link the resulting 처리 단계 units in source order. Reject empty units and trailing conditionals while permitting a trailing semicolon.
- **중요한 이유**: The representation preserves the project's grammar without introducing an unnecessarily general AST. It also gives the executor exactly the unit required for short-circuit decisions and keeps cleanup correct when parsing fails after a completed prefix.
- **확정된 변경 범위**: The parser gained 처리 단계-list construction, connector translation, 오류 처리 for empty or incomplete operands, and cleanup of both the current 처리 단계 and the already parsed sequence prefix.
- **프로젝트 이해에서의 위치**: It explains the shell's precedence model: pipes form one executable unit first; sequence and conditional operators then connect those units from left to right.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: one 처리 단계에 여러 command를 넣을 수 있었지만 line-level sequence를 표현할 수 없었습니다.
- 해결하려던 문제: `a | b && c ; d`에서 pipe group을 먼저 완성한 뒤 `&&`와 `;`가 그 groups를 연결해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: 모든 operator를 같은 list level에 두면 pipe precedence와 connector의 left-result dependency가 사라집니다.
- 선택한 결정: connector를 왼쪽 complete 처리 단계의 `next_op`에 저장하고 처리 단계 nodes를 source order로 연결합니다. General AST 대신 현재 grammar에 필요한 최소 linked sequence를 사용합니다.
- publish 또는 state mutation이 일어나는 지점: connector를 만나면 current command를 current 처리 단계에 finalize하고, 처리 단계를 sequence tail에 연결한 후 `next_op`를 설정하고 새 처리 단계를 시작합니다.
- failure 뒤 cleanup 또는 상태: leading/empty/trailing conditional, pipe RHS 누락에서 current objects와 completed prefix를 모두 하나의 실패 처리로 해제합니다. Trailing `;`만 허용하고 final tail connector를 `CONN_NONE`으로 정규화합니다.

#### `f297aaad70fe`에서 확인할 실제 코드
- 처리 단계 node의 `next`와 `next_op`를 확인했습니다.
- `a | b && c ; d`는 `[a,b] --AND--> [c] --SEQ--> [d]`가 됩니다.
- connector token → internal enum 변환과 왼쪽 처리 단계 publish 순서를 확인했습니다.
- leading connector, empty operand, trailing `&&/||`, pipe RHS 누락은 error입니다.
- trailing semicolon은 accepted sequence termination입니다.

#### 학습자가 남길 코드 증거
- 예제 입력의 token → 처리 단계 list 변환:

```text
WORD(a) PIPE WORD(b) AND WORD(c) SEMI WORD(d)
  → pipeline#1 commands=[a,b], next_op=AND
  → pipeline#2 commands=[c],   next_op=SEQ
  → pipeline#3 commands=[d],   next_op=NONE
```

- 각 처리 단계에 저장된 `next_op`: 다음 처리 단계를 실행할 조건을 왼쪽 node가 보유합니다.
- trailing semicolon normalization 전/후: parse 중 left 처리 단계에는 sequence 의미가 생기지만 새 empty tail을 publish하지 않고 최종 published tail은 `CONN_NONE`입니다.
- completed prefix와 current object의 정리 과정: shared parse failure가 current command/current 처리 단계를 먼저 정리하고 sequence head에 `free_pipeline`을 적용합니다.
- 이 표현이 executor에 제공하는 최소 control-flow 정보: ordered 처리 단계 pointer와 left connector enum입니다.
- 확인한 변경 파일: `src/parser.c`, `include/shell.h`.
- 핵심 caller → callee: `parse_tokens` → 처리 단계 finalization/connector mapping/sequence append → `free_pipeline`.
- parent SHA와 비교한 최소 before/after snippet: one `t_pipeline` return에서 linked 처리 단계 list와 `next_op` assignment로 확장됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. 위 예제와 malformed connector states를 source branch로 추적했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: pipe가 먼저 complete executable unit을 만들고, sequence/conditional connector가 그 처리 단계 units를 왼쪽부터 연결합니다.
- 아직 보장하지 않는 것: representation만으로 short-circuit나 delayed expansion이 실행되지는 않습니다.

#### Thread 내 다음 연결
`13a70b408e89`가 `next_op`와 previous status를 사용해 실제 gate와 delayed expansion을 수행합니다.

### 5.6 `13a70b408e89` — `feat(exec): 조건 연결자와 지연 확장 실행`

#### 확정 정보
- SHA: `13a70b408e89`
- Subject: `feat(exec): 조건 연결자와 지연 확장 실행`
- Importance: **S**
- Tags: `ARCH`, `EXPANSION`, `CORE`
- Source-defined role: Executes that sequence with short-circuiting and status-correct delayed expansion.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
처리 단계 list를 source order로 순회하면서 preceding connector와 previous status로 실행 여부를 결정하고, 선택된 처리 단계만 현재 shell state로 확장한 뒤 dispatch합니다.

#### Source가 확정한 핵심 판단
- **문제**: Expanding an entire parsed line before execution would evaluate skipped branches and would give later 처리 단계 a stale value of `$?`.
- **결정**: Carry the previous connector through the 처리 단계 list, decide whether the next 처리 단계 runs, and expand only that selected 처리 단계 immediately before dispatch using current shell state.
- **중요한 이유**: 제어 흐름 and expansion are semantically coupled. A skipped branch must produce no expansion side effects or allocation failures, and an executed branch must observe the status produced by the 처리 단계 immediately before it.
- **확정된 변경 범위**: The commit separated one-처리 단계 expansion and execution, temporarily detached a 처리 단계 during expansion, implemented `&&` and `||` gating, propagated status after each executed unit, and stopped traversal when `exit` ended the shell.
- **프로젝트 이해에서의 위치**: It explains why parsing can happen for the complete line while expansion remains runtime state-dependent. This timing decision is one of the project's most important shell-semantic judgments.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: executor는 단일 처리 단계를 받아 전체 대상에 expansion을 적용한 뒤 실행했습니다.
- 해결하려던 문제: short-circuit로 skip될 처리 단계를 미리 확장하면 불필요한 allocation failure가 발생하고, later `$?`가 직전 실행 결과가 아닌 stale status를 볼 수 있습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: parser가 line 전체를 먼저 만드는 것은 가능하지만 runtime state-dependent expansion까지 parse 직후 일괄 수행하면 control flow와 timing이 어긋납니다.
- 선택한 결정: previous connector와 current `shell->last_status`로 gate를 먼저 계산하고, 선택된 current 처리 단계만 list에서 임시 분리해 expand한 뒤 실행합니다.
- publish 또는 state mutation이 일어나는 지점: 실행된 처리 단계의 return status만 `shell->last_status`에 기록되고 다음 gate의 입력이 됩니다. Skipped 처리 단계는 parsed fields와 status를 변경하지 않습니다.
- failure 뒤 cleanup 또는 상태: expansion failure는 dispatch 전에 status 1로 반환됩니다. Link는 restore되고, top-level hierarchy cleanup이 전체 parsed list를 해제합니다. `shell->running == 0`이면 traversal을 중단합니다.

#### `13a70b408e89`에서 확인할 실제 코드
- `src/exec.c::execute_pipeline_list_ctx`의 source-order loop를 확인했습니다.
- AND는 previous status가 0일 때, OR는 nonzero일 때, sequence는 항상 실행합니다.
- gate branch가 `expand_one_pipeline`보다 먼저입니다.
- `expand_one_pipeline`은 `next = pipeline->next; pipeline->next = NULL; ...; pipeline->next = next;` 순서로 current unit만 확장합니다.
- execution result가 `last_status`에 들어간 뒤 previous connector가 update됩니다.
- `running`이 false가 되면 remaining sequence를 실행하지 않습니다.

#### 학습자가 남길 코드 증거
- connector gate truth table과 실제 branch:

| Previous connector | Previous status | Current 처리 단계 |
| --- | ---: | --- |
| `CONN_NONE` / sequence | any | execute |
| `CONN_AND` | 0 | execute |
| `CONN_AND` | nonzero | skip |
| `CONN_OR` | 0 | skip |
| `CONN_OR` | nonzero | execute |

- gate → detach → expand → execute → relink 순서: gate가 false면 detach/expand 자체를 호출하지 않습니다. True면 current `next`를 임시 NULL로 만들고 expansion 후 원래 link를 복원한 뒤 dispatch합니다.
- `$?`가 참조하는 status와 update line: `expand_pipeline`은 호출 시점의 `shell->last_status`를 읽고, `execute_one_pipeline` 반환 뒤 executor가 그 field를 갱신합니다.
- skipped 처리 단계에 expansion allocation이 발생하지 않는 근거: gate false branch가 expansion call을 건너뜁니다.
- `running` 변화가 loop를 중단하는 위치: executed parent builtin `exit`가 state를 clear한 뒤 list loop condition/branch가 remaining node로 진행하지 않습니다.
- 확인한 변경 파일: `src/exec.c`와 executor declarations.
- 핵심 caller → callee: `execute_pipeline_list_ctx` → gate → `expand_one_pipeline` → `expand_pipeline` → `execute_one_pipeline` → parent command 또는 forked execution.
- parent SHA와 비교한 최소 before/after snippet:

```c
next = pipeline->next;
pipeline->next = NULL;
result = expand_pipeline(shell, pipeline);
pipeline->next = next;
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. `false && echo $?`, `true || echo $?`, `false || echo $?`의 gate/expansion order를 source로 추적했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: connector decision은 직전 실행 status를 사용하고, skipped 처리 단계는 확장도 실행도 하지 않으며, 실행된 처리 단계는 최신 `$?`를 봅니다.
- 아직 보장하지 않는 것: 이 commit만으로 top-level line/token/parsed cleanup 수명이 완전히 연결되지는 않으며 다음 integration commit에서 묶입니다.

#### Thread 내 다음 연결
`91ded56b033d`가 한 입력 줄의 tokenization, parsing, execution, status, cleanup을 하나의 transaction으로 연결합니다.

### 5.7 `91ded56b033d` — `feat(shell): 한 줄 해석과 실행 수명 연결`

#### 확정 정보
- SHA: `91ded56b033d`
- Subject: `feat(shell): 한 줄 해석과 실행 수명 연결`
- Importance: **B**
- Tags: `INTEGRATION`, `CORE`
- Source-defined role: Connects the concrete parse and execution 수명 to each input line.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
`shell_process_line`에서 tokenization, parsing, execution, diagnostics, status update, parsed cleanup을 한 줄 단위로 묶습니다. Syntax failure는 status 258, empty parse는 이전 status 유지, valid parse는 실행 후 전체 해제입니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: lexer, parser, list executor가 별도 API로 존재했지만 input loop의 transient object 수명이 한 곳에 묶이지 않았습니다.
- 해결하려던 문제: lexical/구문 분석 오류, empty line, valid execution마다 token/처리 단계/error allocation을 정확히 정리하고 status semantics를 달리해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: 각 subsystem caller가 cleanup을 따로 책임지면 early return에서 누락되기 쉽고 다음 prompt에 transient state가 남을 수 있습니다.
- 선택한 결정: `shell_process_line`을 line transaction entry로 두고 tokenize → parse → token free → execute → 처리 단계 free 순서를 고정합니다.
- publish 또는 state mutation이 일어나는 지점: lexical/구문 분석 오류는 `last_status = 258`; valid execution은 executor가 status를 갱신합니다. Empty parse는 이전 status를 유지합니다.
- failure 뒤 cleanup 또는 상태: error string, token prefix, parsed tree를 해당 branch에서 정리하며 persistent `t_shell`의 environment/status/running만 다음 input에 남깁니다.

#### `91ded56b033d`에서 확인할 실제 코드
- input loop가 owned line을 `shell_process_line`에 넘긴 뒤 해제하는 경계를 확인했습니다.
- `shell_process_line`은 token list와 처리 단계 list의 acquisition/cleanup을 모두 포함합니다.
- lexer/parser diagnostic branch는 258을 설정합니다.
- `pipelines == NULL && error == NULL`인 empty parse는 executor를 호출하지 않습니다.
- valid path는 executor 완료 뒤 `free_pipeline`을 호출합니다.

#### 학습자가 남길 코드 증거
- line owner와 derived representation owner: input loop가 line allocation을 소유하고, lexer가 별도 token text/list를, parser가 별도 hierarchy를 소유합니다.
- syntax failure status path: diagnostic 출력 → error free → transient objects free → `last_status = 258`.
- empty input status path: no parsed 처리 단계가면 기존 `last_status` 반환.
- valid execution status path: `execute_pipeline_list`/context executor → parsed cleanup → current status 반환.
- 다음 prompt 전에 반드시 해제되는 transient objects: input line, error text, token nodes/text, 처리 단계 hierarchy와 expanded replacements입니다.
- 확인한 변경 파일: line processor가 있는 `src/exec.c`, caller input loop가 있는 `src/input.c`.
- 핵심 caller → callee: `shell_loop` → `shell_process_line` → `tokenize_line` → `parse_tokens` → `free_tokens` → executor → `free_pipeline`.
- parent SHA와 비교한 최소 before/after snippet: subsystem 호출이 한 line-scoped entry에 결합되고 error/empty/valid branches가 분리됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact source에서 세 terminal branch와 cleanup 순서를 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 한 줄에서 파생된 transient representation은 다음 input 전에 정리되고, persistent `t_shell` state만 command 사이에 남습니다.
- 아직 보장하지 않는 것: later heredoc integration과 allocation/I/O hardening은 이 commit 이후의 별도 thread에서 추가됩니다.

#### Thread 내 다음 연결
이 Thread의 최종 integration 지점입니다. 이후 thread에서는 동일 parsed 수명에 heredoc과 failure recovery가 결합됩니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 명시한 항상 유지해야 하는 조건과 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.

| 항상 유지해야 하는 조건 | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Published parsed objects have one hierarchical cleanup owner. | 공개된 token, argv entry, redirection, command, pipeline의 소유권은 계층적으로 단일해야 합니다. | `48670b845d7f` | `f297aaad70fe` | `src/parser.c`의 node constructors/append와 `free_pipeline`; sequence parse failure도 completed prefix와 current objects를 같은 hierarchy cleanup으로 정리합니다. |
| Quote effects survive until the stage that needs them. | quote delimiter가 제거된 뒤에도 expansion에 필요한 literal 의미는 남아야 합니다. | `729a6d2a7d4a` | `13a70b408e89`에서 runtime expansion과 결합 | `src/token.c::read_word`가 single-quoted byte에 `LITERAL_MARK`를 붙이고, selected pipeline의 `expand_word`가 marker pair를 literal byte로 소비합니다. Token text는 line과 별도 allocation입니다. |
| Connector gating precedes expansion and execution. | connector는 직전 status를 사용하며, skip된 pipeline은 확장도 실행도 하지 않습니다. | `f297aaad70fe`에서 표현 | `13a70b408e89`에서 실행 invariant로 확정 | `execute_pipeline_list_ctx`의 gate가 `expand_one_pipeline`보다 앞서고, 실행 result를 `last_status`에 쓴 뒤 다음 node로 이동합니다. |

### Ledger 작성 시 확인한 것

- `48670b845d7f`는 ownership field/destructor를 도입했고, `f297aaad70fe`는 completed prefix까지 같은 cleanup이 적용되도록 실제 parser graph를 확장했습니다.
- quote marker는 lexical representation이고, runtime expansion timing은 별도 commit에서 결합됐습니다.
- Thread에 test commit이 없으므로 test evidence를 소급하지 않았습니다. Code path와 example state만 기록했습니다.
- 정상·syntax failure·partial parse 모두 hierarchy의 terminal owner가 `free_tokens`/`free_pipeline`로 수렴합니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| quote 의미가 raw delimiter와 함께 사라질 위험 | `729a6d2a7d4a`의 literal-marker token 표현 | 이 Thread에는 별도 fix commit이 포함되지 않습니다. | 해당 SHA의 lexer error path와 이후 expansion 연결을 직접 추적합니다. | `read_word`의 single-quote marker 생성과 `expand_word`의 marker 소비를 연결했습니다. Heredoc provenance 문제는 이 표현이 모든 quote participation을 보존하지 못해 Thread 2에서 별도 수정됩니다. |
| pipe와 connector를 같은 수준으로 처리하면 binding이 깨짐 | `8624028b83bb`의 pipeline 경계와 `f297aaad70fe`의 pipeline-linked sequence | 구조 자체가 문제를 예방하는 결정입니다. | source-defined Thread에는 test commit이 없으므로 parser code와 예제 입력으로 증거를 남깁니다. | `a | b && c ; d`가 `[a,b] --AND--> [c] --SEQ--> [d]`로 변환되는 state와 `next_op` assignment를 추적했습니다. |
| 전체 line 선확장 시 skipped branch가 확장되고 `$?`가 stale해짐 | `13a70b408e89`의 gate 후 delayed expansion | 동일 commit에서 execution order를 변경합니다. | gate 전후 호출 순서와 status mutation을 exact SHA에서 확인합니다. | `execute_pipeline_list_ctx` gate → `expand_one_pipeline` → execute → `last_status` update 순서와 false gate에서 expansion call 부재를 확인했습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| 입력 줄 | `shell_loop` | line 처리 종료 | tokenization 완료 뒤에도 token text가 독립 소유인지 확인 | line과 token text는 별도 allocation이며 input loop가 line을 회수합니다. |
| token text/list | lexer/token list cleanup | parse가 필요한 데이터를 복사한 뒤 | `free_tokens`와 parser copy 지점 기록 | parser가 argv/redirection target을 duplicate한 뒤 `shell_process_line`이 token list를 free합니다. |
| argv/redirection/command/처리 단계 | parsed hierarchy | line execution 완료 또는 parse failure | `free_pipeline` 및 sequence cleanup의 leaf-to-root 순서 기록 | command-owned strings/nodes부터 처리 단계 list까지 하나의 hierarchy로 해제됩니다. |
| expanded argv/redirection target | parsed field를 대체한 소유자 | 처리 단계 실행 후 parsed cleanup | old encoded string free와 new string publish 순서 기록 | selected 처리 단계에서 expansion 성공 후 field가 replacement를 소유하고, final `free_pipeline`이 해제합니다. Skipped fields는 encoded 상태 그대로 남았다가 같은 cleanup을 탑니다. |
| `t_shell` status/running/environment | top-level shell | process 종료 | per-line transient data와 분리되는 경계 기록 | line cleanup 뒤에도 `last_status`, `running`, environment만 남아 다음 gate/expansion에 사용됩니다. |

## 9. Thread 최종 상태

- 최종 자료구조:

```text
t_pipeline(head)
  ├─ commands -> t_command -> t_command ...
  │                 ├─ argv[] -> owned strings
  │                 └─ redirs -> t_redir(target) ...
  ├─ command_count
  ├─ next_op  -- 왼쪽 pipeline이 다음 pipeline에 적용할 connector
  └─ next -> t_pipeline ...
```

- 각 connector는 왼쪽 complete 처리 단계의 `next_op`에 저장됩니다.
- `execute_pipeline_list_ctx`가 previous connector와 current `last_status`로 gate한 뒤 selected 처리 단계만 `expand_one_pipeline`에 전달합니다.
- syntax failure는 transient objects를 정리하고 258, empty parse는 이전 status 유지, valid execution은 실행 결과를 status로 남긴 뒤 hierarchy를 정리합니다.

### 최종 상태 기록

- 최종적으로 유지되는 data/자원 소유권: line, token list, parsed hierarchy는 각 단계에서 독립 allocation을 소유하고 한 line 종료 전에 해제됩니다. `t_shell`만 process 수명을 가집니다.
- 최종적으로 보장되는 execution 또는 recovery rule: pipe는 한 처리 단계 내부에서 먼저 결합되고 connector는 complete 처리 단계를 연결합니다. Gate를 통과한 처리 단계만 현재 shell state로 확장·실행됩니다.
- Thread가 해결한 가장 어려운 failure: completed prefix와 in-progress objects가 함께 존재하는 sequence parse failure, 그리고 short-circuit branch를 선확장하지 않도록 timing을 분리한 문제입니다.
- Thread 밖에 남아 있는 보장 범위: heredoc provenance/input recovery, syscall·allocation failure, process/FD lifecycle, 성능 검증은 후속 Thread가 담당합니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
[source line: shell_loop가 line allocation 소유]
  ↓ tokenize_line
[owned tokens: text + source offset + single-quote literal marker]
  ↓ parse_tokens: argv/redirection target을 복사하고 complete node만 publish
[pipeline list: command list + ordered redirections + command_count + next_op]
  ↓ execute_pipeline_list_ctx connector gate
[selected pipeline only]
  ↓ expand_one_pipeline with current shell->last_status
[parent command 또는 forked pipeline dispatch]
  ↓ executed status를 last_status에 기록
[free_pipeline 전체 hierarchy cleanup]
  ↓
[next input line]
```

### 코드 기반 최종 설명

- 핵심 entry function: `shell_process_line`.
- 주요 caller → callee chain: `shell_loop` → `shell_process_line` → `tokenize_line` → `parse_tokens` → `free_tokens` → `execute_pipeline_list`/`execute_pipeline_list_ctx` → `expand_one_pipeline` → dispatch → `free_pipeline`.
- state mutation 순서: token/처리 단계 local publish → connector gate → selected field expansion → command execution → `last_status` update → optional `running` clear → transient cleanup.
- 소유권 transfer 순서: scanner local word → token node → parser duplicate to command/redirection → expanded replacement in parsed field → hierarchy destructor.
- failure convergence path: lexical/parse failure는 error allocation과 partial structures를 정리하고 258; expansion failure는 no dispatch/status 1; valid/skip path 모두 final hierarchy cleanup으로 수렴합니다.
- regression evidence: 이 Thread에는 source-defined test commit이 없습니다. Exact branch order와 소유권 cleanup을 code inspection으로 검증했으며 runtime test는 실행하지 않았습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] S commit은 problem, prior state, failure possibility, decision, core code, 소유권/lifecycle, 후속 작업을 설명했습니다.
- [x] A commit은 subsystem boundary 또는 실패 처리와 실제 핵심 code를 설명했습니다.
- [x] B commit은 Thread 내 구현 역할과 state/소유권 변화를 설명했습니다.
- [x] 이 Thread의 fix/test 부재를 명시하고 code evidence를 임의의 later test로 대체하지 않았습니다.
- [x] 항상 유지해야 하는 조건 ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 resource와 partial object의 terminal owner를 설명했습니다.
- [x] 이 Thread의 설계 → 구현 → 실행 timing → integration 흐름을 commit history 순서로 재구성했습니다.
===== END FILE: 01-parsed-representation-to-conditional-execution.md =====

===== BEGIN FILE: 02-heredoc-cross-stage-semantics.md =====
# Heredoc from stored input to recoverable cross-stage semantics

> 한국어 주제: **저장된 입력에서 복구 가능한 cross-stage heredoc 의미까지**
>
> Project: `small-shell`
> Branch: `c/minishell`
> Development Thread order: 2/5

## 1. Thread 목표

heredoc delimiter의 정규화와 quote provenance, body의 수집·저장·확장, stdin 설치, temporary-stream 오류 전파, 입력 경계 복구를 하나의 수명으로 추적합니다.

**Source-defined significance**

> Heredoc is the strongest integration thread in the history. It crosses parsed identity, quote provenance, input ordering, body expansion, descriptor installation, and recovery after a failure has already consumed part of stdin. The history exposes two distinct corrections: preserving semantic provenance rather than reconstructing it from text, and preserving the command stream boundary even when preparation fails.

**학습 관점**

Heredoc은 parser identity, quote provenance, 입력 소비 순서, expansion, descriptor 설치, 실패 뒤 stream position 복구를 모두 가로지르는 가장 강한 integration thread입니다.

### SHA 고정 원칙

- 각 commit은 반드시 표시된 exact SHA 또는 그 parent와 비교합니다.
- 먼저 `git show --name-status <SHA>`로 변경 파일을 식별한 뒤, 필요한 path만 `git diff <SHA>^ <SHA> -- <path>`로 봅니다.
- 실제 구현은 `git show <SHA>:<path>` 또는 detached worktree에서 확인합니다.
- final HEAD의 type, function, test를 과거 commit 설명에 소급하지 않습니다.
- later commit의 field나 fix가 아직 존재하지 않는 SHA에서는 그 부재 자체를 기록합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- dequoted delimiter text와 'quote syntax가 사용되었는가'라는 provenance는 왜 별도 정보입니까?
- 동일한 delimiter text가 여러 번 등장해도 body가 섞이지 않도록 어떤 identity를 key로 사용합니까?
- 왜 connector gating 전에 line 전체의 heredoc을 source order로 수집합니까?
- quoted delimiter와 unquoted delimiter에서 body expansion 경로는 어떻게 갈립니까?
- temporary stream이 stdin으로 공개되기 전에 어떤 단계가 모두 성공해야 합니까?
- 준비가 이미 stdin 일부를 소비한 뒤 실패하면 다음 command boundary를 어떻게 복구합니까?
- 복구 read까지 반복 실패할 때 shell이 계속 실행하면 안 되는 이유는 무엇입니까?

## 3. 완료 기준

- [x] 한 line에 heredoc이 둘 이상 있는 입력을 사용해 parse node identity와 body entry를 연결했습니다.
- [x] single, double, partial quote delimiter의 text와 provenance를 별도로 기록했습니다.
- [x] parse → precollect → store → expand policy → temp stream → dup2 → cleanup 순서를 코드로 검증했습니다.
- [x] quote provenance bug, temp stream failure, input-boundary failure의 각각에 대해 Fix → 회귀 테스트를 연결했습니다.
- [x] 정상 EOF warning, recoverable read failure, unrecoverable repeated failure를 구분했습니다.

> 실행 범위: exact SHA의 commit diff와 해당 시점 source/test를 GitHub repository에서 검사했습니다. Branch checkout이 불가능해 test binary와 shell script는 실행하지 않았으며, 아래에서 코드 검토와 실제 실행을 구분합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `e65591bb66f5` | `feat(heredoc): 구분자 정규화 버퍼 구현` | B | `HEREDOC`, `PRACTICAL` | Introduces delimiter dequoting support. |
| 2 | `7c9692346824` | `feat(heredoc): 수집 본문 저장소 수명 관리` | A | `HEREDOC`, `ARCH`, `RISK` | Defines body storage keyed by the owning redirection node. |
| 3 | `fc9c63a03db2` | `feat(heredoc): 구분자별 본문 순차 수집` | A | `HEREDOC`, `CORE`, `INTEGRATION` | Collects all pending bodies in source order. |
| 4 | `aeb0d6cba9c1` | `feat(heredoc): 인용 여부에 따라 본문 확장` | A | `HEREDOC`, `EXPANSION`, `CORE` | Adds quote-dependent body expansion. |
| 5 | `d297bd2e8908` | `feat(redirection): heredoc을 stdin으로 연결` | S | `HEREDOC`, `FD_IO`, `INTEGRATION` | Integrates heredoc syntax, precollection, lifetime, redirection order, and stdin installation. |
| 6 | `854f0f435c82` | `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존` | S | `HEREDOC`, `DEBUG`, `RISK` | Replaces an insufficient text heuristic with explicit lexical quote provenance. |
| 7 | `dce9e5c083fa` | `test(heredoc): 이중·부분 인용 구분자 회귀 검증` | A | `TEST`, `HEREDOC`, `EDGE` | Locks down double-quoted and partially quoted delimiters. |
| 8 | `9afdca85f5a5` | `fix(heredoc): 임시 파일 저장 오류를 전파` | A | `HEREDOC`, `FD_IO`, `FAILURE` | Propagates temporary-stream storage and positioning failures. |
| 9 | `2fbc4c73af2c` | `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증` | A | `TEST`, `HEREDOC`, `FAILURE` | Verifies that such failures cannot silently truncate command input. |
| 10 | `c30b39c0bcf8` | `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구` | A | `HEREDOC`, `FAILURE`, `RISK` | Restores future command boundaries after heredoc preparation failure. |
| 11 | `7e2fdea3affd` | `test(io): read·write와 heredoc 입력 실패 검증` | A | `TEST`, `FAILURE`, `HEREDOC` | Verifies read failure, recovery, continuation, and forced-stop behavior. |

## 5. Commit별 학습 기록

### 5.1 `e65591bb66f5` — `feat(heredoc): 구분자 정규화 버퍼 구현`

#### 확정 정보
- SHA: `e65591bb66f5`
- Subject: `feat(heredoc): 구분자 정규화 버퍼 구현`
- Importance: **B**
- Tags: `HEREDOC`, `PRACTICAL`
- Source-defined role: Introduces delimiter dequoting support.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
lexer literal-marker encoding을 제거해 exact delimiter text를 만드는 전용 normalization buffer를 추가하되, ordinary variable expansion은 수행하지 않습니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: ordinary word expansion은 있었지만 heredoc delimiter의 encoded quote marker만 제거하는 전용 API가 없었습니다.
- 해결하려던 문제: matching에는 `E\001O\001F` 같은 encoded parser text가 아니라 최종 `EOF`가 필요하지만, `$NAME`을 일반 argv처럼 확장해서는 안 됩니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: ordinary `expand_word`를 재사용하면 delimiter text가 environment/status에 따라 변하고 heredoc syntax의 matching 규칙이 깨집니다.
- 선택한 결정: local growable `strbuf`와 `dequote_runtime_word`를 추가해 marker pair는 literal byte 하나로, unmarked byte는 그대로 복사합니다.
- publish 또는 state mutation이 일어나는 지점: 완성된 buffer pointer만 caller에 반환합니다. Parsed redirection의 encoded target은 변경하지 않습니다.
- failure 뒤 cleanup 또는 상태: init/reserve/append 실패 시 partial buffer를 free하고 NULL을 반환합니다. 이 SHA에는 later `SIZE_MAX` capacity guard가 아직 없습니다.

#### `e65591bb66f5`에서 확인할 실제 코드
- `src/heredoc.c::dequote_runtime_word`와 local `struct strbuf`를 확인했습니다.
- `LITERAL_MARK`를 만나면 marker와 다음 byte를 함께 소비하고 next byte만 append합니다.
- `$`는 별도 branch 없이 ordinary byte로 복사됩니다.
- append 뒤 `data[len] = '\0'`를 유지하고 필요할 때 capacity를 늘립니다.

#### 학습자가 남길 코드 증거
- encoded delimiter 입력 예와 normalized output: `E\001O\001F` → `EOF`; `$TAG` → `$TAG`로 유지됩니다.
- marker pair 소비 branch: marker가 있고 following byte가 있으면 index를 2만큼 진행하고 following byte만 append합니다.
- NUL/capacity 항상 유지해야 하는 조건: initialized cap 64, `len` 위치에 항상 NUL, `len + 1`이 cap에 닿으면 grow합니다.
- failure cleanup: local buffer allocation을 free하고 NULL; source `redir->target`에는 mutation이 없습니다.
- ordinary word expansion과 분리되는 API boundary: `dequote_runtime_word`는 shell/environment 인자를 받지 않고 encoded text만 받습니다.
- 확인한 변경 파일: `src/heredoc.c`와 build source list.
- 핵심 caller → callee: later `read_heredoc` → `dequote_runtime_word` → strbuf init/append.
- parent SHA와 비교한 최소 before/after snippet:

```c
if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
    /* marker는 버리고 literal byte만 append */
    i += 2;
} else {
    /* '$'를 포함한 일반 byte를 그대로 append */
    i++;
}
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact function body와 allocation/error branches를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: parser-owned encoded delimiter를 보존한 채 matching에 필요한 dequoted text를 별도 owned buffer로 얻습니다.
- 아직 보장하지 않는 것: body storage, collection order, quote-dependent body expansion, stdin installation은 아직 없습니다.

#### Thread 내 다음 연결
`7c9692346824`가 normalized delimiter로 수집한 body를 line-scoped execution context에 저장할 ownership model을 정의합니다.

### 5.2 `7c9692346824` — `feat(heredoc): 수집 본문 저장소 수명 관리`

#### 확정 정보
- SHA: `7c9692346824`
- Subject: `feat(heredoc): 수집 본문 저장소 수명 관리`
- Importance: **A**
- Tags: `HEREDOC`, `ARCH`, `RISK`
- Source-defined role: Defines body storage keyed by the owning redirection node.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
각 heredoc body와 해당 parsed redirection pointer를 pair로 저장하는 execution-context-owned repository를 추가합니다. delimiter text가 같아도 redirection identity로 구분하며, destructor가 body와 entry를 해제합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: normalized delimiter를 만들 수 있었지만 body를 line execution 동안 보관하고 redirection과 연결하는 owner가 없었습니다.
- 해결하려던 문제: `cat <<EOF <<EOF`처럼 같은 text가 반복돼도 각 syntax occurrence가 별도 body를 가져야 합니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: delimiter string을 key로 사용하면 duplicate text가 collision하고 source-order redirection identity가 사라집니다.
- 선택한 결정: `heredoc_entry`가 non-owning `const t_redir *redir` key와 owned `char *body`를 pair로 보관하고 `exec_context`가 entry list를 소유합니다.
- publish 또는 state mutation이 일어나는 지점: body는 collector local로 완성되고 entry allocation이 성공한 뒤 `entry->body`로 소유권이 이전되어 repository list에 연결됩니다.
- failure 뒤 cleanup 또는 상태: entry allocation 실패면 caller가 body를 계속 소유해 free할 수 있습니다. Repository destructor는 body를 먼저 free하고 node를 free합니다.

#### `7c9692346824`에서 확인할 실제 코드
- `src/exec_internal.h`의 execution 문맥과 `src/heredoc.c`의 entry fields를 확인했습니다.
- lookup은 delimiter `strcmp`가 아니라 `entry->redir == redir` pointer equality를 사용합니다.
- missing lookup은 NULL 대신 empty string 대체 처리를 반환해 redirection caller가 read-only body contract를 유지합니다.
- repository가 key로 참조하는 parsed tree보다 먼저 해제되어야 합니다.

#### 학습자가 남길 코드 증거
- entry 소유권 graph: `exec_context → entry node → body allocation`; `entry->redir`는 parsed tree의 non-owning pointer입니다.
- 동일 delimiter 두 개의 서로 다른 key: text가 모두 `EOF`여도 `&redir1 != &redir2`이므로 별도 entry가 선택됩니다.
- lookup return contract: matching entry의 body, 없으면 `""`; caller는 free하지 않습니다.
- repository와 parsed tree 수명 ordering: `exec_heredoc_entries_free(ctx.heredocs)`가 `free_pipeline(pipelines)`보다 먼저여야 dangling key traversal을 피합니다.
- execution context init/free 호출자: line executor가 `ctx.heredocs = NULL`로 시작하고 line execution 종료/준비 실패에서 repository destructor를 호출합니다.
- 확인한 변경 파일: `src/exec_internal.h`, `src/heredoc.c`.
- 핵심 caller → callee: preparation → `add_heredoc_entry`; redirection apply → body lookup; final cleanup → entry destructor.
- parent SHA와 비교한 최소 before/after snippet: standalone normalized string 기능 위에 redirection-identity-keyed line repository가 새로 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Pointer-key lookup과 destructor order를 source로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 여러 heredoc이 같은 delimiter를 사용해도 body identity가 섞이지 않고 line execution 동안 안정적으로 조회됩니다.
- 아직 보장하지 않는 것: body를 언제 어떤 순서로 읽고 어떤 policy로 확장하는지는 아직 정하지 않습니다.

#### Thread 내 다음 연결
`fc9c63a03db2`가 parsed sequence를 source order로 순회해 repository를 채웁니다.

### 5.3 `fc9c63a03db2` — `feat(heredoc): 구분자별 본문 순차 수집`

#### 확정 정보
- SHA: `fc9c63a03db2`
- Subject: `feat(heredoc): 구분자별 본문 순차 수집`
- Importance: **A**
- Tags: `HEREDOC`, `CORE`, `INTEGRATION`
- Source-defined role: Collects all pending bodies in source order.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
sequence의 처리 단계, command, redirection을 source order로 순회하여 모든 heredoc body를 실행 전에 수집합니다. exact delimiter match까지 읽고 각 line에 newline을 붙여 identity-keyed repository에 저장합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: repository는 있었지만 parser graph를 순회해 stdin에서 body를 채우는 preparation phase가 없었습니다.
- 해결하려던 문제: line 안의 모든 pending heredoc을 lexical order로 소비하고, connector에서 skip될 command의 body도 다음 command parsing 전에 제거해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: selected command 실행 시점에만 heredoc을 읽으면 source input 순서와 conditional gate가 뒤섞이고 body line이 다음 shell command로 해석될 수 있습니다.
- 선택한 결정: top-level preparation이 처리 단계 → command → redirection 순으로 전부 순회하고 각 heredoc을 delimiter까지 읽어 repository에 저장합니다.
- publish 또는 state mutation이 일어나는 지점: delimiter match/EOF까지 body buffer가 local이고 `add_heredoc_entry` 성공 뒤 repository-owned가 됩니다.
- failure 뒤 cleanup 또는 상태: allocation/read preparation failure는 partial local body를 discard하고 failure를 반환합니다. 이 SHA는 즉시 abort하므로 unread body/pending heredoc이 stdin에 남을 수 있으며 later fix 대상입니다.

#### `fc9c63a03db2`에서 확인할 실제 코드
- `exec_prepare_heredocs`의 nested source-order traversal을 확인했습니다.
- `read_heredoc`은 delimiter normalize → line read → exact `strcmp` → line+'
' append → entry publish 순서입니다.
- secondary prompt는 stdin과 stderr가 모두 tty인 interactive condition에서만 사용됩니다.
- delimiter 전 EOF는 warning을 출력하지만 collected-so-far body를 entry로 유지합니다.
- connector gate를 알기 전에 complete parsed line의 heredoc을 수집합니다.

#### 학습자가 남길 코드 증거
- 여러 heredoc의 실제 traversal order: 처리 단계 list order, 각 처리 단계의 command order, 각 command의 redirection list order입니다.
- line comparison에서 newline 제거/보존 처리: line reader가 newline 없는 logical line을 반환하고 delimiter와 exact compare하며, body line만 append 후 `\n`을 하나 추가합니다.
- stored body의 newline policy: delimiter line은 저장하지 않고 각 accepted body line 끝에 정확히 하나의 newline을 저장합니다.
- premature EOF warning과 return contract: warning 후 현재 body를 성공 결과로 publish하며 preparation 자체를 syntax failure로 만들지 않습니다.
- allocation failure의 stream position: 이미 읽은 line은 되돌릴 수 없고, immediate return 때문에 current remainder와 later delimiter가 남을 수 있습니다.
- 확인한 변경 파일: `src/heredoc.c`, `src/exec_internal.h`.
- 핵심 caller → callee: line processor/executor setup → `exec_prepare_heredocs` → `read_heredoc` → `shell_read_line`/buffer append → `add_heredoc_entry`.
- parent SHA와 비교한 최소 before/after snippet: passive repository에 graph traversal과 stdin-consuming collector가 연결됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. `cat <<ONE <<TWO`의 traversal과 EOF branch를 source로 추적했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: pending heredoc input은 lexical/source order로 결정적으로 소비되고, 각 body는 owning redirection identity에 저장됩니다.
- 아직 보장하지 않는 것: quoted delimiter에 따른 body expansion 차이는 아직 적용되지 않으며, preparation failure 뒤 unread input recovery도 later fix 대상입니다.

#### Thread 내 다음 연결
`aeb0d6cba9c1`가 body line transformation을 quote policy와 연결합니다.

### 5.4 `aeb0d6cba9c1` — `feat(heredoc): 인용 여부에 따라 본문 확장`

#### 확정 정보
- SHA: `aeb0d6cba9c1`
- Subject: `feat(heredoc): 인용 여부에 따라 본문 확장`
- Importance: **A**
- Tags: `HEREDOC`, `EXPANSION`, `CORE`
- Source-defined role: Adds quote-dependent body expansion.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
delimiter가 unquoted로 판단되면 body line에서 `$?`와 valid environment name을 확장하고, quoted로 판단되면 bytes를 그대로 보존합니다. 이 시점의 quote 판단은 delimiter의 literal marker 존재 여부에 의존합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: 모든 body line이 literal로 저장됐습니다.
- 해결하려던 문제: unquoted delimiter에는 limited variable/status expansion을 적용하고, quoted delimiter에는 적용하지 않아야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: collector는 final delimiter text만 필요하다고 가정했고 quote syntax participation을 별도 field로 보존하지 않았습니다.
- 선택한 결정: encoded target에 `LITERAL_MARK`가 있으면 quoted로 간주하고, quoted branch는 raw line append, unquoted branch는 heredoc-specific `$?`/`$NAME` expander를 사용합니다.
- publish 또는 state mutation이 일어나는 지점: transformed line이 body buffer에 append되고 delimiter 종료 뒤 entry로 publish됩니다.
- failure 뒤 cleanup 또는 상태: expansion/append 실패면 partial body를 discard하고 preparation failure를 반환합니다.

#### `aeb0d6cba9c1`에서 확인할 실제 코드
- quote heuristic은 encoded delimiter에서 literal marker가 존재하는지 검사합니다.
- unquoted expander는 `$?`, valid name, unset variable, unknown/incomplete `$`를 분기합니다.
- quoted branch는 expansion helper를 거치지 않고 original line bytes를 append합니다.
- body line마다 newline 하나를 추가합니다.
- Double-quoted text는 marker 없이 저장될 수 있어 hidden assumption이 깨집니다.

#### 학습자가 남길 코드 증거
- quote heuristic의 실제 condition: target bytes 중 `LITERAL_MARK` 발견 여부입니다.
- body expansion 상태 머신 또는 helper: `$?`는 decimal status, `$` 뒤 valid name은 environment value, unset은 empty, 그 외 `$`는 literal로 처리합니다.
- unset/unknown dollar 결과: unset valid name은 아무 byte도 append하지 않고, unknown form은 `$` 자체를 보존한 뒤 다음 byte를 normal scan합니다.
- quoted body literal path: line을 expansion helper 없이 body buffer에 append합니다.
- later fix가 필요한 hidden assumption: marker는 single-quoted literal byte를 나타낼 뿐, double quote 또는 partial double quote 참여를 항상 나타내지 않습니다.
- 확인한 변경 파일: `src/heredoc.c`.
- 핵심 caller → callee: `read_heredoc` → quote heuristic → literal append 또는 heredoc body expander → body buffer.
- parent SHA와 비교한 최소 before/after snippet: body append 전에 quoted/unquoted policy branch가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. `$?`, `$NAME`, unset, `<<'EOF'`, `<<"EOF"`의 code path를 비교했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: unquoted heredoc body는 제한된 variable/status expansion을 받고, quoted로 판정된 body는 literal로 보존됩니다.
- 아직 보장하지 않는 것: marker presence가 모든 quote syntax를 대표한다는 가정은 double quote와 partial quote에서 성립하지 않습니다.

#### Thread 내 다음 연결
`d297bd2e8908`가 이 policy를 parser, execution context, redirection installation과 통합한 뒤 `854f0f435c82`가 provenance root cause를 수정합니다.

### 5.5 `d297bd2e8908` — `feat(redirection): heredoc을 stdin으로 연결`

#### 확정 정보
- SHA: `d297bd2e8908`
- Subject: `feat(redirection): heredoc을 stdin으로 연결`
- Importance: **S**
- Tags: `HEREDOC`, `FD_IO`, `INTEGRATION`
- Source-defined role: Integrates heredoc syntax, precollection, 수명, redirection order, and stdin installation.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
`<<`를 first-class token/redirection으로 만들고, 실행 전에 모든 body를 수집한 뒤 ordinary redirection traversal에서 temporary stream을 stdin에 설치합니다. Body 수명은 parsed line execution에 한정됩니다.

#### Source가 확정한 핵심 판단
- **문제**: Heredoc requires more than recognizing `<<`: its body must be read before execution, associated with the correct parsed redirection, installed in source order, and released at the end of the line.
- **결정**: Make heredoc a first-class token and redirection type, precollect all bodies into an execution context keyed by redirection identity, dequote rather than normally expand the delimiter, and install the selected body through the ordinary redirection traversal.
- **중요한 이유**: Reusing ordered redirection application preserves interactions with incoming pipes and later input redirects. The line-scoped execution context also keeps body 수명 independent of child 수명 while retaining a stable link to parsed 소유권.
- **확정된 변경 범위**: Lexer and parser support, heredoc preparation, execution-context initialization, failure cleanup, temporary-stream installation on stdin, and post-execution body release were connected into the product path.
- **프로젝트 이해에서의 위치**: This commit shows how a shell feature crosses every major phase. It is the clearest example of the repository's integration and 소유권 design.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: collector/repository helpers는 있었지만 lexer/parser가 `<<`를 first-class redirection으로 만들고 product execution에 연결하지 않았습니다.
- 해결하려던 문제: syntax recognition, precollection, body identity/수명, pipe/redirection precedence, stdin installation을 한 line transaction으로 결합해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: heredoc을 ordinary `< file`처럼 처리할 수 없고, child가 실행할 때 stdin에서 body를 읽으면 multiple/conditional ordering과 parent input stream이 깨집니다.
- 선택한 결정: `TOK_HEREDOC`/`REDIR_HEREDOC`을 추가하고 line processor가 문맥을 initialize해 모든 body를 먼저 수집하며, ordered redirection traversal이 repository body를 temporary stream으로 stage한 뒤 stdin에 `dup2`합니다.
- publish 또는 state mutation이 일어나는 지점: parsed redirection node와 repository entry는 preparation 성공 후 stable identity로 연결됩니다. Staging은 temporary stream을 완성한 뒤 `dup2`에서 stdin을 바꿉니다.
- failure 뒤 cleanup 또는 상태: preparation 실패는 status 1, repository free, parsed tree free로 수렴합니다. Per-redirection staging failure는 stream close 후 command redirection failure가 됩니다. 이 SHA에서는 `fflush`/rewind failure 결과를 충분히 확인하지 않습니다.

#### `d297bd2e8908`에서 확인할 실제 코드
- lexer scanner의 `<<` longest-match와 token enum, parser의 heredoc redirection type을 확인했습니다.
- line processor는 context init → all-heredoc preparation → sequence executor → repository free → parsed free 순서입니다.
- ordinary argv/redirection expansion은 heredoc delimiter를 normal word expansion에서 제외합니다.
- redirection apply는 body lookup → `tmpfile` → `fputs` → `fflush` → 당시 `rewind` → `fileno` → `dup2` → close 순서입니다.
- child는 incoming pipe를 먼저 stdin에 wiring한 뒤 ordered redirection을 적용하므로 heredoc이 pipe를 override할 수 있고, 더 뒤의 `< file`은 다시 heredoc을 override합니다.

#### 학습자가 남길 코드 증거
- `<<`의 lexer → parser → redirection type 전파: scanner의 double-character operator → `TOK_HEREDOC` → parser target consumption → `REDIR_HEREDOC` node.
- parse/precollect/execute/free lifecycle: parsed tree 생성 → `ctx.heredocs=NULL` → `exec_prepare_heredocs` → execute list with context → entry repository free → parsed hierarchy free.
- temporary stream의 acquisition과 cleanup: `tmpfile()` owner는 redirection apply local; write/position/dup success 또는 any error 뒤 `fclose`합니다.
- pipe/heredoc/later input redirect precedence 예: `producer | cat <<EOF <file`에서 pipe wiring → heredoc stdin → file stdin 순서이므로 final source는 `file`입니다.
- delimiter dequote와 body expansion 책임 분리: delimiter matching text는 `dequote_runtime_word`; body policy는 quoted heuristic과 heredoc-specific expander입니다.
- 확인한 변경 파일: `include/shell.h`, `src/token.c`, `src/parser.c`, `src/heredoc.c`, `src/redirection.c`, `src/exec.c`, `src/exec_internal.h`.
- 핵심 caller → callee: `shell_process_line` → `exec_prepare_heredocs` → list executor → child/parent redirection apply → heredoc body lookup/staging → `shell_dup2`/`dup2`.
- parent SHA와 비교한 최소 before/after snippet: independent heredoc helpers가 token/parser/product execution path와 normal redirection order에 연결됩니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact integration diff와 function ordering을 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: heredoc은 parsed identity, ordered precollection, line-scoped storage, normal redirection precedence, stdin installation을 하나의 product path로 연결합니다.
- 아직 보장하지 않는 것: quote 여부는 아직 text marker heuristic이고, flush/seek 같은 temporary-stream failure 전파와 preparation failure 뒤 stream recovery는 later commits에서 보강됩니다.

#### Thread 내 다음 연결
`854f0f435c82`가 quote provenance를 explicit field로 복구하고, `9afdca85f5a5`와 `c30b39c0bcf8`가 failure semantics를 강화합니다.

### 5.6 `854f0f435c82` — `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존`

#### 확정 정보
- SHA: `854f0f435c82`
- Subject: `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존`
- Importance: **S**
- Tags: `HEREDOC`, `DEBUG`, `RISK`
- Source-defined role: Replaces an insufficient text heuristic with explicit lexical quote provenance.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
Delimiter text에서 quote 여부를 재구성하던 heuristic을 제거하고, token이 quote syntax 참여 여부를 기록해 parser의 `heredoc_quoted` field와 collector까지 전달합니다.

#### Source가 확정한 핵심 판단
- **문제**: The runtime inferred whether a delimiter had been quoted by looking for literal markers in its text. Double-quoted and partially quoted delimiters could contain no marker, so their bodies were expanded incorrectly.
- **결정**: Record quote participation explicitly in each token, copy that provenance into heredoc redirections, and use the preserved flag independently from the dequoted delimiter text.
- **중요한 이유**: Final text and lexical provenance answer different questions. Text is needed for delimiter matching; provenance is needed to decide expansion. Reconstructing one from the other is not reliable after token normalization.
- **확정된 변경 범위**: `t_token` gained a quoted flag, word scanning set it whenever quote syntax appeared, the parser stored it as `heredoc_quoted`, and collection used that field rather than marker inspection.
- **프로젝트 이해에서의 위치**: It is the strongest root-cause correction in the semantic history and demonstrates why representation layers must preserve information needed by later phases even when that information is absent from normalized text.

#### Fix 재구성 기록
- 기존 가정: encoded target에 `LITERAL_MARK`가 있으면 quoted이고 없으면 unquoted라는 가정이었습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: `cat <<"EOF"` 또는 `cat <<E"OF"`의 normalized delimiter는 `EOF`이고 encoded text에 single-quote marker가 없을 수 있어 body의 `$HD`가 잘못 확장됩니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: lexer가 quote delimiter를 제거한 뒤 final text만 보고 lexical participation을 복원하려 한 representation boundary입니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: delimiter matching text와 quote provenance를 서로 독립된 values로 보존합니다.
- 변경 전 코드 증거: collector가 target bytes에서 marker presence를 scan해 `quoted`를 계산했습니다.
- 변경 후 코드 증거: `t_token.quoted` → heredoc parse branch의 `t_redir.heredoc_quoted` → `read_heredoc`의 policy branch로 값이 전달됩니다.
- 연결되는 regression test와 그 한계: `dce9e5c083fa`가 fully double-quoted와 partial double-quoted cases를 고정합니다. 모든 가능한 quote 조합이나 I/O failure는 다루지 않습니다.

#### `854f0f435c82`에서 확인할 실제 코드
- parent SHA의 marker scan condition을 확인했습니다.
- `include/shell.h`에서 token quoted field와 redirection heredoc field가 추가됩니다.
- `src/token.c::read_word`는 single/double quote 어느 쪽이든 quote syntax 진입 시 flag를 set합니다.
- `src/parser.c`는 heredoc operator의 target token에서만 quoted flag를 redirection field로 복사합니다.
- `src/heredoc.c`는 normalized text는 matching에, `redir->heredoc_quoted`는 body expansion decision에 사용합니다.
- Token list가 free돼도 copied redirection flag가 parsed 수명 동안 유지됩니다.

#### 학습자가 남길 코드 증거
- 기존 가정: marker presence ≈ quote participation.
- 실제 failure input: `HD=expanded; cat <<"EOF"` body `$HD` 또는 `cat <<E"OF"` body `$HD`.
- root cause가 text normalization 이후 정보 손실인 근거: 두 inputs의 matching text는 unquoted `EOF`와 같지만 body policy는 달라야 합니다.
- token flag set → parser copy → collector branch 경로: `read_word(..., &quoted)` → token node `quoted` → heredoc target parse → `redir->heredoc_quoted` → `read_heredoc` literal/expand branch.
- 수정된 semantic 항상 유지해야 하는 조건: quote syntax가 delimiter의 어느 segment에라도 참여하면 body expansion을 억제합니다.
- 확인한 변경 파일: `include/shell.h`, `src/token.c`, `src/parser.c`, `src/heredoc.c`.
- 핵심 caller → callee: `tokenize_line` → `parse_tokens` → `exec_prepare_heredocs` → `read_heredoc`.
- parent SHA와 비교한 최소 before/after snippet:

```c
/* before: text-derived heuristic */
quoted = contains_literal_marker(redir->target);

/* after: parser가 보존한 lexical provenance */
quoted = redir->heredoc_quoted;
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact parent/commit diff에서 field propagation과 old heuristic 제거를 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: delimiter matching용 final text와 body expansion policy용 lexical provenance가 독립적으로 유지됩니다.
- 아직 보장하지 않는 것: 이 fix는 quote semantics를 다루며 temporary-stream I/O failure나 input recovery는 별도 문제입니다.

#### Thread 내 다음 연결
`dce9e5c083fa`가 double-quoted와 partially quoted delimiter를 deterministic regression으로 고정합니다.

### 5.7 `dce9e5c083fa` — `test(heredoc): 이중·부분 인용 구분자 회귀 검증`

#### 확정 정보
- SHA: `dce9e5c083fa`
- Subject: `test(heredoc): 이중·부분 인용 구분자 회귀 검증`
- Importance: **A**
- Tags: `TEST`, `HEREDOC`, `EDGE`
- Source-defined role: Locks down double-quoted and partially quoted delimiters.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Fully double-quoted delimiter와 unquoted/double-quoted segment가 섞인 delimiter 모두에서 final terminator는 `EOF`로 dequote되지만 body의 `$HD`는 literal이어야 함을 검증합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: delimiter text와 quote provenance는 독립이며 quote segment가 하나라도 있으면 body expansion을 억제합니다.
- 재현하는 failure 또는 boundary: marker가 없는 double/partial quote가 old heuristic에서 unquoted로 오인되는 boundary입니다.
- 사용한 test technique: environment를 고정한 end-to-end deterministic regression입니다.
- 실제 통과하는 production code path: input/tokenize → heredoc target parse/provenance copy → precollection → normalized `EOF` match → quoted literal body → temporary stdin → `cat`.
- 이 테스트가 증명하는 것: 두 target form 모두 `EOF`로 종료되면서 body `$HD`가 environment value로 바뀌지 않고 literal로 출력됩니다.
- 이 테스트가 증명하지 않는 것: single quote, multiple heredoc, temporary stream failures, allocation cleanup 전체를 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: 직전 root-cause fix에 대한 deterministic end-to-end regression입니다.
- 후속 변경에서 막는 회귀: token quoted field 제거, parser copy 누락, collector가 marker heuristic으로 돌아가는 변경을 잡습니다.

#### `dce9e5c083fa`에서 확인할 실제 코드
- `tests/smoke.sh`가 `HD`를 expansion과 구별되는 값으로 설정합니다.
- Exact fixtures는 fully double-quoted delimiter와 `E"OF"` 형태의 partial quote입니다.
- Expected output은 expansion 결과가 아니라 literal `$HD`이며 command status는 0입니다.
- 두 cases 모두 production lexer/parser/collector/redirection/external command 경로를 통과합니다.

#### 학습자가 남길 코드 증거
- 대상 production 항상 유지해야 하는 조건: any quote participation suppresses body expansion while dequoted matching text remains unchanged.
- 재현하는 failure/boundary: final text가 같아 provenance를 text에서 역산할 수 없는 경우입니다.
- test technique: fixed environment + exact stdin fixture + expected stdout/status comparison.
- 통과하는 production path: `shell_process_line` → heredoc preparation → `cat` stdin installation.
- 증명하는 것: `<<"EOF"`와 `<<E"OF"` 모두 delimiter matching은 `EOF`, body policy는 literal입니다.
- 증명하지 않는 것: 모든 quote 조합, all failure paths, 자원 누수 부재는 증명하지 않습니다.
- 막는 후속 회귀: lexical provenance field 또는 parser copy를 제거해 text heuristic으로 돌아가는 회귀입니다.
- 확인한 변경 파일: `tests/smoke.sh`.
- 핵심 caller → callee: test shell input → product `shell_process_line` → heredoc collector/redirection → external `cat`.
- parent SHA와 비교한 최소 before/after snippet: production 변경 없이 직전 fix를 재현하는 two fixtures와 expected outputs가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Test script와 expected bytes/status를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 어떤 quote segment라도 delimiter에 참여하면 body expansion이 억제된다는 provenance contract를 관찰 가능한 output으로 고정합니다.
- 아직 보장하지 않는 것: single quote 외의 provenance regression만 다루며 temp stream, read failure, multiple heredoc recovery는 증명하지 않습니다.

#### Thread 내 다음 연결
다음 failure chain은 quote가 아니라 temporary-stream integrity를 다룹니다.

### 5.8 `9afdca85f5a5` — `fix(heredoc): 임시 파일 저장 오류를 전파`

#### 확정 정보
- SHA: `9afdca85f5a5`
- Subject: `fix(heredoc): 임시 파일 저장 오류를 전파`
- Importance: **A**
- Tags: `HEREDOC`, `FD_IO`, `FAILURE`
- Source-defined role: Propagates temporary-stream storage and positioning failures.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
In-memory heredoc body를 temporary input stream으로 변환할 때 body write, flush, seek, descriptor lookup이 모두 성공한 뒤에만 `dup2`를 호출하도록 오류를 전파합니다.

#### Fix 재구성 기록
- 기존 가정: `fputs`가 성공하면 body가 readable stream에 완전히 저장됐다고 보고 `fflush`와 rewind/positioning result를 사실상 best-effort로 취급했습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: buffered write 뒤 flush가 실패하거나 seek가 실패하면 empty/truncated stream 또는 end-position stream을 stdin으로 설치할 수 있습니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: memory body에서 stdio stream/descriptor로 변환하는 staging boundary에서 각 intermediate operation의 success가 publish condition에 포함되지 않았습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: write → flush → seek-to-start → descriptor lookup이 모두 성공하기 전 stdin을 바꾸지 않습니다.
- 변경 전 코드 증거: `fflush` result가 무시되고 `rewind`는 failure를 반환하지 않는 API로 사용됐습니다.
- 변경 후 코드 증거: operation-specific wrappers의 return을 검사하고 shared `heredoc_stream_error`로 수렴한 뒤, 마지막에만 `shell_dup2(fd, STDIN_FILENO)`를 호출합니다.
- 연결되는 regression test와 그 한계: `2fbc4c73af2c`가 deterministic flush/seek failures를 검증합니다. 모든 possible write/fileno/dup2 조합을 단독 검증하지는 않습니다.

#### `9afdca85f5a5`에서 확인할 실제 코드
- `src/redirection.c`의 heredoc stream installation path를 parent SHA와 비교했습니다.
- Body write, `shell_fflush`, `shell_fseek(..., 0, SEEK_SET)`, `shell_fileno`, `shell_dup2`의 exact ordering을 확인했습니다.
- 각 checked operation은 common failure helper로 수렴합니다.
- Error helper는 `errno`를 먼저 저장하고, stdio가 errno를 남기지 않은 경우 `EIO`를 사용하며, operation name과 diagnostic을 기록합니다.
- Stream은 success/failure 모두에서 close됩니다.
- `dup2` 전까지 process stdin descriptor는 변경되지 않습니다.

#### 학습자가 남길 코드 증거
- 기존 best-effort assumption: write request 반환만으로 readable staging completion을 간주했습니다.
- 각 staging 단계의 성공 조건: `fputs != EOF`, `fflush == 0`, `fseek == 0`, `fileno >= 0`, 마지막 `dup2 >= 0`입니다.
- shared 오류 처리와 saved errno: failed operation에서 `saved_errno = errno != 0 ? errno : EIO`; close/diagnostic 때문에 원인이 덮이지 않도록 보존합니다.
- stdin publish point: 모든 staging checks 뒤 `shell_dup2(temp_fd, STDIN_FILENO)`입니다.
- 실패 뒤 following command가 가능한 resource state: temporary stream closed, stdin은 이 heredoc으로 교체되지 않으며 current command status 1로 돌아갑니다.
- 확인한 변경 파일: `src/redirection.c`, runtime wrapper declarations/definitions.
- 핵심 caller → callee: command redirection traversal → heredoc apply → body lookup → stdio staging wrappers → `shell_dup2`.
- parent SHA와 비교한 최소 before/after snippet:

```c
if (shell_fflush(stream) != 0)
    return heredoc_stream_error(stream, "fflush");
if (shell_fseek(stream, 0, SEEK_SET) != 0)
    return heredoc_stream_error(stream, "fseek");
fd = shell_fileno(stream);
if (fd < 0)
    return heredoc_stream_error(stream, "fileno");
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact diff에서 all success conditions와 stdin mutation ordering을 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 불완전하거나 end-position에 놓인 temporary stream이 성공한 heredoc stdin으로 공개되지 않습니다.
- 아직 보장하지 않는 것: 이 fix의 body는 이미 collection된 상태이므로 stdin command-boundary recovery와는 별개의 실패 영역입니다.

#### Thread 내 다음 연결
`2fbc4c73af2c`가 flush와 seek failure를 주입해 data truncation/EOF 노출을 막는지 검증합니다.

### 5.9 `2fbc4c73af2c` — `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증`

#### 확정 정보
- SHA: `2fbc4c73af2c`
- Subject: `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증`
- Importance: **A**
- Tags: `TEST`, `HEREDOC`, `FAILURE`
- Source-defined role: Verifies that such failures cannot silently truncate command input.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
`fflush`와 `fseek` failure를 deterministic하게 주입하고, heredoc command가 status 1로 실패하며 `cat` payload를 출력하지 않고 following command는 계속 실행되는지 검증합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: temporary stream staging이 완료되지 않으면 stdin publish와 command dispatch가 성공으로 진행돼서는 안 됩니다.
- 재현하는 failure 또는 boundary: body write 이후의 flush failure와 rewind/seek failure입니다.
- 사용한 test technique: `SMALL_SHELL_TESTING` runtime wrapper에서 operation별 selected call을 실패시키는 deterministic fault injection입니다.
- 실제 통과하는 production code path: heredoc precollection 성공 → command redirection apply → temporary stream body write → injected `fflush` 또는 `fseek` failure → shared staging error → current status 1 → next line.
- 이 테스트가 증명하는 것: payload가 partial/empty stdin으로 `cat`에 전달되지 않고 current command failure가 `$?`로 관찰되며 following command가 실행됩니다.
- 이 테스트가 증명하지 않는 것: write, fileno, dup2의 모든 call position과 OS별 stdio 동작을 포괄하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: operation-specific deterministic end-to-end regression입니다.
- 후속 변경에서 막는 회귀: `fflush`/`fseek` result 무시, error를 success로 변환, failure 뒤 loop 중단/stream leak 회귀입니다.

#### `2fbc4c73af2c`에서 확인할 실제 코드
- `src/runtime.c`의 test-only failure counter와 `shell_fflush`/`shell_fseek` wrapper를 확인했습니다.
- Wrapper는 selected occurrence에서 representative errno와 failure return을 제공합니다.
- `tests/faults.sh`의 각 case는 heredoc body를 먼저 수집한 뒤 staging operation을 실패시킵니다.
- Expected result는 current command status 1, `cat` body 출력 없음, following marker output 존재입니다.
- Temporary directory/isolated stdout·stderr files로 case side effect를 분리합니다.

#### 학습자가 남길 코드 증거
- 대상 production 항상 유지해야 하는 조건: no stdin replacement before complete staging.
- 재현하는 failure: flush completion failure, position reset failure.
- injection technique: compile-time test branch + operation call counter + environment-selected failure occurrence.
- 통과하는 production code path: `exec_apply_redirections`/heredoc apply → checked wrappers → `heredoc_stream_error`.
- 증명하는 것: silent truncation/EOF success가 status-bearing failure로 바뀌고 shell line processing은 신뢰 가능한 stdin에서 계속됩니다.
- 증명하지 않는 것: test가 명시적으로 선택하지 않은 operations와 all platform stdio semantics입니다.
- broad integration 또는 deterministic regression 판정: deterministic regression이며 product redirection path를 end-to-end로 통과합니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, `tests/faults.sh`.
- 핵심 caller → callee: test harness → test binary → heredoc redirection staging wrappers → current status → next command.
- parent SHA와 비교한 최소 before/after snippet: production fix 뒤 operation wrappers와 two failure fixtures/assertions가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Injection mechanism과 expected status/stdout continuation을 script/source로 검증했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: temporary stream의 completion/rewind 실패가 silent truncated input으로 바뀌지 않고 status-bearing redirection failure가 됩니다.
- 아직 보장하지 않는 것: 이 commit의 명시적 regression은 flush와 seek이며 write, fileno, dup2의 모든 조합을 단독으로 증명하지 않습니다.

#### Thread 내 다음 연결
`c30b39c0bcf8`는 staging 이후가 아니라 body preparation 중 failure로 흐트러진 stdin boundary를 복구합니다.

### 5.10 `c30b39c0bcf8` — `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`

#### 확정 정보
- SHA: `c30b39c0bcf8`
- Subject: `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`
- Importance: **A**
- Tags: `HEREDOC`, `FAILURE`, `RISK`
- Source-defined role: Restores future command boundaries after heredoc preparation failure.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Delimiter dequote, body buffer init, body expansion 등이 실패한 뒤 즉시 return하지 않고, current heredoc remainder와 later pending heredoc을 모두 delimiter까지 소비한 후 failure를 반환합니다.

#### Source가 확정한 핵심 판단
- **문제**: A heredoc preparation failure could return while body lines and later delimiters remained in stdin, causing data intended for the failed command to be parsed as future shell commands.
- **결정**: Mark preparation as failed, consume the remainder of the current and later pending heredocs without constructing bodies, and compare encoded delimiters directly when normal dequoting allocation is unavailable.
- **중요한 이유**: For a streaming command interpreter, preserving the next command boundary is as important as freeing memory. Returning an error without restoring input position would convert a local allocation failure into unintended command execution.
- **확정된 변경 범위**: The collector gained discard-through-delimiter behavior, marker-aware allocation-free delimiter matching, continued traversal of pending heredocs, and additional capacity-overflow protection.
- **프로젝트 이해에서의 위치**: This exceptional A-level commit reveals the depth of the failure model: recovery must account not only for objects and descriptors but also for semantic position in the input stream.

#### Fix 재구성 기록
- 기존 가정: local body allocation을 free하고 failure를 반환하면 heredoc preparation cleanup이 완료된다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: `cat <<ONE <<TWO`의 first body 중 allocation/read failure 후 `ONE`, second body, `TWO`가 stdin에 남으면 shell loop가 그 lines를 commands로 실행할 수 있습니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: memory 소유권은 정리됐지만 stream cursor라는 외부 state가 original command boundary까지 복구되지 않았습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: 첫 preparation failure를 기억하고 current heredoc의 remainder와 모든 later pending heredoc을 construction 없이 delimiter까지 소비한 후에만 caller로 돌아갑니다.
- 변경 전 코드 증거: dequote/init/append failure branch가 즉시 nonzero를 return했습니다.
- 변경 후 코드 증거: `failed` state, `discard_heredoc`, allocation-free `delimiter_matches`가 추가되고 traversal은 remaining redirections까지 계속됩니다.
- 연결되는 regression test와 그 한계: `7e2fdea3affd`가 read failure recovery와 repeated read failure forced stop을 검증하고 `476b082d55c7`가 allocation positions를 sweep합니다. 복구 read 자체가 불가능하면 boundary를 보장할 수 없어 shell을 중단합니다.

#### `c30b39c0bcf8`에서 확인할 실제 코드
- Parent의 early return branches와 새 failed-mode traversal을 비교했습니다.
- `exec_prepare_heredocs`는 first failure 뒤에도 처리 단계/command/redirection 순회를 계속하고 later heredoc은 `discard_heredoc`으로 처리합니다.
- Current `read_heredoc` 실패도 자신의 delimiter까지 discard하려고 시도합니다.
- `delimiter_matches`는 normal dequote allocation 없이 encoded target의 marker를 건너뛰며 exact line length/content를 비교합니다.
- Body builder capacity growth에 `SIZE_MAX / 2` guard가 추가됩니다.
- Failed mode에서는 body entry를 publish하지 않습니다.

#### 학습자가 남길 코드 증거
- 기존 가정: heap rollback만 완료하면 command transaction이 끝난다는 가정입니다.
- 실제 위험: unread body가 command로 재해석되는 입력 예: `cat <<EOF`, body `echo unintended`, delimiter `EOF`, following `echo safe`에서 failure 뒤 body가 top-level command가 될 수 있습니다.
- root cause: failure return과 stream position 불일치입니다.
- failed mode의 traversal: failure flag set → current remainder discard → outer traversal 계속 → each pending heredoc discard → final nonzero return.
- allocation-free delimiter matching: encoded target의 marker byte는 skip하고 following literal byte와 input line을 직접 비교합니다.
- 복구 완료 시 반환 status와 next input position: preparation returns failure/status 1이며 stdin은 모든 pending heredoc delimiter 뒤, 즉 다음 top-level command 시작에 위치합니다.
- 확인한 변경 파일: `src/heredoc.c`.
- 핵심 caller → callee: `exec_prepare_heredocs` → `read_heredoc` or `discard_heredoc` → `delimiter_matches`/`shell_read_line`.
- parent SHA와 비교한 최소 before/after snippet:

```text
before: failure → free partial body → return immediately
 after: failure → failed=1 → discard current/later heredocs → return failure
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact diff에서 failure flag, traversal continuation, no-publish 동작을 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: heredoc preparation이 실패해도 body data와 pending delimiters가 future shell commands로 이동하지 않습니다.
- 아직 보장하지 않는 것: recovery read 자체가 계속 실패하면 boundary를 보장할 수 없으며 그 경우의 forced-stop policy는 `7e2fdea3affd`가 검증합니다.

#### Thread 내 다음 연결
Heredoc Thread에서는 `7e2fdea3affd`로 이어지고, allocation Thread에서는 `476b082d55c7` sweep으로 재검증됩니다.

### 5.11 `7e2fdea3affd` — `test(io): read·write와 heredoc 입력 실패 검증`

#### 확정 정보
- SHA: `7e2fdea3affd`
- Subject: `test(io): read·write와 heredoc 입력 실패 검증`
- Importance: **A**
- Tags: `TEST`, `FAILURE`, `HEREDOC`
- Source-defined role: Verifies read failure, recovery, continuation, and forced-stop behavior.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Low-level read/write failure를 주입해 top-level input, builtin output, heredoc collection의 서로 다른 recovery 범위를 검증합니다. Heredoc read failure는 boundary를 복구하면 continuation, recovery read까지 반복 실패하면 shell stop이어야 합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: I/O failure 뒤에도 status뿐 아니라 다음 input의 의미가 신뢰 가능해야 하며, 신뢰할 수 없으면 shell이 종료돼야 합니다.
- 재현하는 failure 또는 boundary: top-level command read, builtin write, heredoc body read, recovery discard read의 failures입니다.
- 사용한 test technique: runtime read/write wrappers의 call-index와 repeat mode를 이용한 deterministic failure injection입니다.
- 실제 통과하는 production code path: input loop reader, builtin output helper, `read_heredoc`, `discard_heredoc`, `shell->running`/process termination branches입니다.
- 이 테스트가 증명하는 것: top-level unread buffer가 실행되지 않고, builtin write failure는 status 1로 관찰되며, one-shot heredoc read failure는 delimiter recovery 후 continuation하고, repeated recovery failure는 residual input을 실행하지 않고 stop합니다.
- 이 테스트가 증명하지 않는 것: 모든 libc buffering/platform interaction, allocation failure 전체, descriptor leak 전체는 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: 여러 recovery 범위를 분리한 deterministic I/O failure regression suite입니다.
- 후속 변경에서 막는 회귀: recoverable failure에서 needless stop, unrecoverable failure에서 unsafe continuation, body line command execution, write failure status loss를 막습니다.

#### `7e2fdea3affd`에서 확인할 실제 코드
- `src/runtime.c`의 read/write operation counters, selected occurrence, repeat-mode branch를 확인했습니다.
- Top-level read failure fixture는 buffered commands의 stdout이 비어 있고 process status 1이어야 합니다.
- Builtin write failure fixture는 next line의 `$?`가 1임을 관찰합니다.
- Heredoc one-shot read failure는 `discard_heredoc`을 통과한 뒤 following command marker를 출력합니다.
- Repeated read failure는 recovery도 실패해 shell을 중단하고 residual body/following marker를 출력하지 않습니다.
- Tests는 status, stdout, stderr/diagnostic, continuation을 별도 assertions로 다룹니다.

#### 학습자가 남길 코드 증거
- 대상 production 항상 유지해야 하는 조건: continue only after reliable command boundary restoration.
- 각 failure의 recovery scope: top-level read는 process input trust 상실로 stop; builtin write는 current command failure로 continue; heredoc read는 discard 성공 시 continue; discard read도 실패하면 stop입니다.
- injection technique과 call position: test build wrapper, operation-specific call counter, exact selected call, optional repeat-from-position mode입니다.
- heredoc recovery production path: body read failure → failed state → `discard_heredoc` → delimiter reached → preparation status 1 → next top-level command.
- continuation이 허용되는 조건: current/pending delimiter를 모두 소비하고 stdin cursor가 next command boundary에 도달한 경우입니다.
- forced stop 조건: repeated read failure로 recovery cursor를 신뢰할 수 없거나 top-level input 자체가 실패한 경우입니다.
- 증명하지 않는 failure class: arbitrary kernel faults, all write sites, memory failure combinations입니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, `src/input.c`, `src/heredoc.c`, `tests/faults.sh`.
- 핵심 caller → callee: harness → test binary wrappers → input/builtin/heredoc production paths → status/running decisions.
- parent SHA와 비교한 최소 before/after snippet: process/FD wrappers가 read/write까지 확장되고 recovery/forced-stop fixtures가 추가됩니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Injection state, fixtures, expected status/output와 production branches를 source로 연결했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: I/O failure status뿐 아니라 future input interpretation의 신뢰성까지 regression으로 고정합니다.
- 아직 보장하지 않는 것: allocation failure sweep 전체를 대체하지 않으며, test 범위는 주입된 read/write paths에 한정됩니다.

#### Thread 내 다음 연결
Heredoc Thread의 마지막 검증입니다. 동일 input-boundary 항상 유지해야 하는 조건은 allocation Thread에서 다른 failure source로 다시 학습합니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 명시한 항상 유지해야 하는 조건과 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.

| 항상 유지해야 하는 조건 | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Delimiter text and quote provenance remain independent. | delimiter는 비교를 위해 dequote할 수 있지만, body expansion 여부를 결정할 quote provenance는 별도로 남아야 합니다. | `aeb0d6cba9c1`의 초기 policy | `854f0f435c82`에서 explicit flag로 복구, `dce9e5c083fa`로 고정 | `read_word`의 token `quoted` set → parser의 `heredoc_quoted` copy → `read_heredoc` policy branch. Matching은 별도 normalized delimiter를 사용합니다. |
| Heredoc bodies are keyed by owning redirection identity. | body는 delimiter text가 아니라 해당 parsed redirection의 주소로 식별됩니다. | `7c9692346824` | `d297bd2e8908`에서 normal redirection path와 통합 | `heredoc_entry.redir` pointer equality lookup; repository entry/body는 parsed tree보다 먼저 free됩니다. |
| Temporary-stream staging must complete before stdin replacement. | body write, flush, seek, descriptor 획득, duplication 실패는 성공한 command input으로 보고될 수 없습니다. | `d297bd2e8908`의 temp-stream 설치 | `9afdca85f5a5`, `2fbc4c73af2c` | `fputs` → checked `fflush` → checked `fseek` → checked `fileno` → final `dup2`; failure injection은 payload suppression/status 1/continuation을 검사합니다. |
| Preparation failure must preserve the next command boundary. | heredoc 준비 실패가 body data를 이후 shell command로 재해석하게 해서는 안 됩니다. | `fc9c63a03db2`의 ordered input consumption | `c30b39c0bcf8`, `7e2fdea3affd` | collector-wide failed state, allocation-free delimiter match, discard traversal; recovery read 반복 실패면 unsafe continuation 대신 stop합니다. |

### Ledger 작성 시 확인한 것

- Body repository field는 `7c969...`에서 생기고 product path 항상 유지해야 하는 조건은 `d297...`에서 완성됩니다.
- `854f...`는 prior policy를 삭제한 것이 아니라 final text에 없던 lexical provenance를 별도 field로 보강합니다.
- Test evidence는 각각 quote, staging, stream recovery production path와 연결했습니다.
- Normal, staging failure, recoverable preparation failure는 entry/stream/local buffers가 해제되고 신뢰 가능한 stdin으로 수렴합니다. Recovery 자체 실패는 shell stop으로 수렴합니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| literal marker 존재 여부만으로 quote를 추정하면 double/partial quote를 놓침 | `aeb0d6cba9c1`, `d297bd2e8908`의 text-based heuristic | `854f0f435c82` — token quote participation을 `heredoc_quoted`까지 전달 | `dce9e5c083fa` — double-quoted와 partially quoted delimiter deterministic regression | Token quoted flag set, parser field copy, collector branch와 two literal `$HD` fixtures를 연결했습니다. |
| temp body write 뒤 flush/seek 실패를 무시하면 truncated input 또는 EOF가 설치됨 | `d297bd2e8908`의 temporary-stream installation | `9afdca85f5a5` — write/flush/seek/fileno 전체 성공 뒤에만 dup2 | `2fbc4c73af2c` — injected flush/seek failure, status 1, no payload, continuation | `src/redirection.c` staging order와 `src/runtime.c` wrappers, `tests/faults.sh` expected outputs를 연결했습니다. |
| 준비 실패 뒤 unread body와 later delimiter가 stdin에 남아 command로 실행될 수 있음 | `fc9c63a03db2`의 early-abort path | `c30b39c0bcf8` — current와 pending heredoc을 delimiter까지 discard | `7e2fdea3affd` — read failure recovery와 recovery read 반복 실패 시 forced stop | `failed` traversal, `discard_heredoc`, `delimiter_matches`, next marker/stop assertions를 연결했습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| encoded delimiter string | parsed redirection | parsed tree cleanup | ordinary word expansion과 분리되는 지점 확인 | Parser-owned target은 matching normalization source로 유지되고 heredoc normal expansion에서 제외됩니다. |
| normalized delimiter | collector local owner | 해당 heredoc collection 완료 | allocation·dequote failure cleanup 기록 | `dequote_runtime_word` 반환을 local이 free하며 entry에는 저장하지 않습니다. Failure면 partial buffer만 해제합니다. |
| body buffer | collector → heredoc repository entry | execution context cleanup | partial buffer discard와 successful transfer 구분 | Delimiter/EOF까지 local; entry allocation/list append 성공 시 body pointer 소유권 이전; failure면 local discard. |
| redirection identity pointer | repository key로 참조 | repository가 parsed tree보다 먼저 해제되어야 함 | 실제 destructor order 확인 | Entry는 pointer를 free하지 않으며 line cleanup이 repository를 먼저 해제한 뒤 `free_pipeline`을 호출합니다. |
| temporary stream / fd | redirection application | dup2 성공 또는 오류 처리 | write·flush·seek·fileno·dup2·fclose 순서 기록 | Local `FILE *` owner가 all stages를 관리하고 success/error 모두 `fclose`; stdin은 final dup에서만 mutate됩니다. |
| stdin stream position | collector/recovery path | 다음 command boundary 확보 시점 | 복구 실패 시 `running` 또는 종료 결정 기록 | Discard가 all delimiters를 소비하면 continue 가능; read가 반복 실패해 cursor 불명확하면 shell을 stop합니다. |

## 9. Thread 최종 상태

Delimiter 관련 state는 다음처럼 분리됩니다.

| State | 용도 | Owner/수명 |
| --- | --- | --- |
| encoded target | parser representation, literal marker 보존 | `t_redir`, line parse 수명 |
| normalized delimiter text | exact input-line matching | collector local |
| `heredoc_quoted` | body expansion policy | copied parser field |
| body bytes | selected command stdin material | execution-context repository |
| redirection node address | body identity key | parsed tree; repository가 non-owning reference |

모든 heredoc은 connector gate 전에 source order로 수집됩니다. Child pipe wiring이 먼저, ordered redirection apply가 다음이므로 later redirection이 earlier stdin source를 override합니다. Normal collection은 repository cleanup, staging failure는 unchanged stdin/current status 1, recoverable preparation failure는 delimiter discard 후 status 1, recovery read failure는 shell stop으로 끝납니다.

### 최종 상태 기록

- 최종적으로 유지되는 data/자원 소유권: parsed redirection은 encoded text와 provenance를, line 문맥은 identity-keyed body entries를, redirection apply local은 temporary stream을 소유합니다.
- 최종적으로 보장되는 execution 또는 recovery rule: all body input은 execution 전 source order로 소비되고, complete staging 후에만 stdin이 바뀌며, preparation failure도 next command boundary를 복구한 경우에만 continuation합니다.
- Thread가 해결한 가장 어려운 failure: 이미 stdin 일부를 소비한 뒤 allocation/read failure가 발생했을 때 body를 future command로 실행하지 않도록 semantic cursor를 복구하는 문제입니다.
- Thread 밖에 남아 있는 보장 범위: tests가 주입하지 않은 모든 OS/stdio failure 조합, descendant 프로세스 생성부터 종료까지의 처리, allocator implementation 전체는 다른 Thread 또는 범위 밖입니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
[parsed redirection: encoded target + heredoc_quoted + stable node identity]
  ↓ exec_prepare_heredocs: pipeline → command → redirection source-order traversal
[dequote_runtime_word / exact line matching]
  ↓ redir->heredoc_quoted branch
[literal line 또는 heredoc-specific $NAME/$? expansion]
  ↓ add_heredoc_entry(redir pointer, owned body)
[execution context repository]
  ↓ selected command's ordered redirection traversal
[tmpfile → write → checked flush → checked seek → checked descriptor → dup2]
  ↓
[command stdin]
  ↓ close temp stream; execute; free repository before parsed tree
[next command boundary recovered, 또는 recovery 불가 시 shell stopped]
```

### 코드 기반 최종 설명

- 핵심 entry function: `exec_prepare_heredocs`와 heredoc redirection apply helper입니다.
- 주요 caller → callee chain: `shell_process_line` → `exec_prepare_heredocs` → `read_heredoc`/`discard_heredoc` → repository → list executor → `exec_apply_redirections` → heredoc staging → `shell_dup2`.
- state mutation 순서: token quote flag → parsed `heredoc_quoted`; body local construction → repository publish; staging complete → stdin replace; status/running update; repository/tree cleanup.
- 소유권 transfer 순서: normalized delimiter remains local; body buffer local → entry; entry body freed by context; temporary stream remains local and is always closed.
- failure convergence path: semantic quote error is prevented by explicit provenance; staging failure closes stream without stdin publish; preparation failure discards current/later bodies; recovery failure stops shell.
- regression evidence: `dce9e5c083fa`, `2fbc4c73af2c`, `7e2fdea3affd`의 test implementation과 production path를 연결했습니다. 실제 test command는 실행하지 않았습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] S commit은 problem, prior state, failure possibility, decision, core code, 소유권/lifecycle, 후속 작업을 설명했습니다.
- [x] A commit은 subsystem boundary 또는 실패 처리와 실제 핵심 code를 설명했습니다.
- [x] B commit은 Thread 내 구현 역할과 state/소유권 변화를 설명했습니다.
- [x] Fix commit은 기존 가정 → failure → root cause → 수정 항상 유지해야 하는 조건 → code → regression 순으로 연결했습니다.
- [x] Test commit은 항상 유지해야 하는 조건, failure, technique, production path, prove/not prove를 구분했습니다.
- [x] 항상 유지해야 하는 조건 ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 resource와 partial object의 terminal owner를 설명했습니다.
- [x] 이 Thread의 설계 → 구현 → 실패 → 수정 → 검증 흐름을 commit history 순서로 재구성했습니다.
===== END FILE: 02-heredoc-cross-stage-semantics.md =====

===== BEGIN FILE: 03-처리 단계-process-and-descriptor-소유권.md =====
# 처리 단계 process and descriptor 소유권 under partial failure

> 한국어 주제: **부분 실패에서의 처리 단계 process와 descriptor 소유권**
>
> Project: `small-shell`
> Branch: `c/minishell`
> Development Thread order: 3/5

## 1. Thread 목표

single-command fork/exec에서 N-stage pipe graph로 확장되는 정상 경로와, pipe/fork/wait/dup/open 실패 뒤 parent가 PID와 FD를 끝까지 회수하는 경로를 함께 복원합니다.

**Source-defined significance**

> The normal 처리 단계 mechanism is only half of the engineering problem. Once a parent records a PID or acquires a descriptor, it owns that resource even if later construction fails. This thread moves from normal execution to deterministic failure injection, root-cause cleanup, unrecoverable parent-state handling, and direct lifecycle observation. Supporting wrappers and tests remain below S because the decisive 소유권 guarantees are established by the pipe graph and partial-construction cleanup commits.

**학습 관점**

정상 처리 단계 wiring만으로는 실행기가 완성되지 않습니다. Parent가 PID를 기록하거나 FD를 획득한 순간부터 후속 단계가 실패해도 그 자원을 종료·관찰·close할 책임이 생깁니다.

### SHA 고정 원칙

- 각 commit은 반드시 표시된 exact SHA 또는 그 parent와 비교합니다.
- 먼저 `git show --name-status <SHA>`로 변경 파일을 식별한 뒤, 필요한 path만 `git diff <SHA>^ <SHA> -- <path>`로 봅니다.
- 실제 구현은 `git show <SHA>:<path>` 또는 detached worktree에서 확인합니다.
- final HEAD의 type, function, test를 과거 commit 설명에 소급하지 않습니다.
- later commit의 field나 fix가 아직 존재하지 않는 SHA에서는 그 부재 자체를 기록합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 어떤 command가 parent에서 실행되고 어떤 command가 child에서 실행되며 그 이유는 무엇입니까?
- N개 command에 대해 왜 N-1개 pipe가 필요하고 child i는 어느 end를 stdin/stdout에 연결합니까?
- explicit redirection이 pipe wiring 뒤에 적용되는 이유는 무엇입니까?
- recorded PID가 생긴 뒤 later fork가 실패하면 단순 wait가 왜 hang할 수 있습니까?
- waitpid failure가 last-stage status보다 우선해야 하는 조건은 무엇입니까?
- parent stdin/stdout restore가 recoverable하게 실패한 경우와 unrecoverable하게 실패한 경우의 shell state는 어떻게 다릅니까?
- output assertion만으로 검출하기 어려운 FD leak과 zombie를 테스트가 어떻게 직접 관찰합니까?

## 3. 완료 기준

- [x] single command와 multi-stage 처리 단계의 parent/child responsibility를 표로 작성했습니다.
- [x] 각 process가 보유·dup·close하는 pipe end를 stage별로 그렸습니다.
- [x] mid-fork failure에서 close → signal → reap 순서를 exact code로 확인했습니다.
- [x] one-shot restore failure와 repeated unrecoverable restore failure의 status/running 변화를 구분했습니다.
- [x] fault-injection regression과 lifecycle stress/probe가 무엇을 증명하는지 기록했습니다.
- [x] pipe creation failure의 acquisition/cleanup matrix에 PID table까지 포함했습니다.

> 실행 범위: exact SHA의 commit diff와 source/test scripts를 GitHub repository에서 검사했습니다. Branch checkout이 불가능해 build, fault suite, lifecycle stress는 실행하지 않았습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `7c9646e7cd79` | `feat(exec): 단일 명령을 자식에서 실행` | A | `PROCESS`, `CORE`, `INTEGRATION` | Establishes fork/exec and child status mapping for a single command. |
| 2 | `ae988017efd5` | `feat(exec): pipeline 자식 상태를 순서대로 회수` | B | `PROCESS`, `CORE` | Adds PID bookkeeping and ordered reaping for multiple commands. |
| 3 | `a71f98de0d92` | `feat(exec): 다단 pipeline의 pipe FD 연결` | S | `PROCESS`, `FD_IO`, `CORE` | Connects the multi-stage pipe graph and defines parent/child descriptor closure. |
| 4 | `915aa072298b` | `refactor(runtime): 프로세스 시스템 호출 경계 분리` | A | `PROCESS`, `FAILURE`, `TEST` | Introduces deterministic pipe, fork, and wait failure seams. |
| 5 | `be2967a4b946` | `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수` | S | `PROCESS`, `FD_IO`, `FAILURE` | Terminates and reaps children after partial pipeline construction. |
| 6 | `d611196b368e` | `test(exec): pipe·fork·wait 실패 회귀 검증` | A | `TEST`, `PROCESS`, `FAILURE` | Reproduces pipe, mid-fork, and wait failure regressions. |
| 7 | `fd5c76c18c27` | `refactor(runtime): FD 시스템 호출 경계 분리` | A | `FD_IO`, `FAILURE`, `TEST` | Extends the runtime boundary to descriptor duplication and opening. |
| 8 | `2ca9f4299c7f` | `fix(redirection): 부모 표준 입출력 복원 실패 전파` | A | `FD_IO`, `FAILURE`, `RISK` | Makes parent standard-stream restoration failure observable and fatal when unrecoverable. |
| 9 | `13645f58d5e6` | `test(redirection): 저장·적용·복원 실패 회귀 검증` | A | `TEST`, `FD_IO`, `FAILURE` | Exercises save, application, restoration, open, and persistent failure paths. |
| 10 | `b42e57eb7755` | `test(lifecycle): FD와 자식 프로세스 누수 검증` | A | `TEST`, `PROCESS`, `FD_IO` | Directly checks for descriptor exhaustion and unreaped children. |
| 11 | `6dff1ba86ba6` | `fix(exec): pipe 생성 실패 시 PID 배열 해제` | B | `FD_IO`, `FAILURE`, `DEBUG` | Closes the remaining PID-table leak before any child is spawned. |

## 5. Commit별 학습 기록

### 5.1 `7c9646e7cd79` — `feat(exec): 단일 명령을 자식에서 실행`

#### 확정 정보
- SHA: `7c9646e7cd79`
- Subject: `feat(exec): 단일 명령을 자식에서 실행`
- Importance: **A**
- Tags: `PROCESS`, `CORE`, `INTEGRATION`
- Source-defined role: Establishes fork/exec and child status mapping for a single command.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Single parsed command를 parent path 또는 forked child path로 dispatch하고, child에서 redirection/builtin/exec를 수행한 뒤 exact PID를 wait하여 normal, signal, 126, 127 status로 변환합니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: parsed/expanded command를 실행할 process boundary와 child outcome mapping이 없었습니다.
- 해결하려던 문제: shell environment/cwd/running을 바꾸는 parent-stateful builtin은 parent에서 실행해야 하지만 ordinary builtin/external command는 child에 격리해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: 모든 command를 parent에서 실행하면 external replacement나 builtin side effect가 shell process를 오염시키고, 모두 child에서 실행하면 `cd`, `export`, `exit`가 parent state에 반영되지 않습니다.
- 선택한 결정: redirection-only command와 parent builtin은 parent path, 나머지는 `fork` 후 child에서 redirection → builtin/exec를 수행합니다.
- publish 또는 state mutation이 일어나는 지점: parent는 returned child PID를 소유하고 exact PID wait 결과를 shell-visible status로 변환합니다. Child side state는 process copy에 한정됩니다.
- failure 뒤 cleanup 또는 상태: child redirection/env serialization/exec failure는 child status로 종료됩니다. Parent `waitpid`는 `EINTR`에 retry하고 observed result를 변환합니다.

#### `7c9646e7cd79`에서 확인할 실제 코드
- `src/exec.c`의 parent/child dispatch predicate를 확인했습니다.
- `run_child`는 redirection을 먼저 적용하고 redirection-only, builtin, external command 순으로 분기합니다.
- External path는 exported environment vector를 만들고 `execvp`에 전달합니다.
- Child builtin은 buffered output을 flush한 뒤 `_exit`합니다.
- Parent `run_single_command`는 exact PID로 wait하고 `status_from_wait`가 normal/signal을 변환합니다.
- `execvp` errno `ENOENT`는 127, 그 외 실행 불가는 126입니다.

#### 학습자가 남길 코드 증거
- parent/child dispatch 조건: `command_count == 1`이면서 `argc == 0` 또는 `builtin_is_parent(argv[0])`이면 parent; 그 외 child입니다.
- child call path: `fork` child → apply redirections → if no argv exit 0 → builtin → flush/_exit → external env serialization/`execvp` → error mapping.
- environment vector owner: child local이 `env_to_environ` result를 소유하며 exec 성공 시 process image로 사라지고 실패 시 child가 정리한 뒤 `_exit`합니다.
- wait/status translation table:

| Child outcome | Shell status |
| --- | ---: |
| `WIFEXITED` | `WEXITSTATUS` |
| `WIFSIGNALED` | `128 + WTERMSIG` |
| command not found (`ENOENT`) | 127 |
| found but not executable/other exec error | 126 |

- parent persistent state와 child copy state의 차이: parent builtin은 original `t_shell`을 mutate하고, 처리 단계/ordinary child builtin의 mutation은 fork copy에만 남습니다.
- 확인한 변경 파일: `src/exec.c`, executor declarations, build source list.
- 핵심 caller → callee: `execute_one_pipeline` → parent command helper 또는 `run_single_command` → `run_child`/`waitpid` → `status_from_wait`.
- parent SHA와 비교한 최소 before/after snippet: direct/no execution state에서 fork/child dispatch and exact wait가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact SHA에서 dispatch, redirection-before-exec, status mapping을 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: ordinary execution은 child에 격리되고, shell state를 유지해야 하는 command만 parent에 남으며, child outcome이 shell-visible status로 변환됩니다.
- 아직 보장하지 않는 것: 한 command만 다루며 multi-stage pipe topology와 partial construction cleanup은 아직 없습니다.

#### Thread 내 다음 연결
`ae988017efd5`가 command마다 PID를 기록해 multi-command lifecycle skeleton을 만듭니다.

### 5.2 `ae988017efd5` — `feat(exec): pipeline 자식 상태를 순서대로 회수`

#### 확정 정보
- SHA: `ae988017efd5`
- Subject: `feat(exec): pipeline 자식 상태를 순서대로 회수`
- Importance: **B**
- Tags: `PROCESS`, `CORE`
- Source-defined role: Adds PID bookkeeping and ordered reaping for multiple commands.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
처리 단계의 각 command를 fork하고 one PID slot per command에 기록한 뒤 exact PID 순서로 reap합니다. 완전 spawn이면 last command status, partial spawn이면 existing children을 reap한 뒤 status 1입니다.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: one child PID만 local로 wait했습니다.
- 해결하려던 문제: multiple commands의 identity와 last-stage status를 유지하려면 source-order PID bookkeeping이 필요했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: generic `wait`는 어느 result가 last stage인지 보장하지 않고, fork 중간 실패 시 생성된 child range를 알 수 없습니다.
- 선택한 결정: `command_count` 크기의 PID table을 만들고 successful fork마다 동일 index slot에 기록하며 `spawned` count를 유지합니다.
- publish 또는 state mutation이 일어나는 지점: successful fork 반환 직후 `pids[index] = pid`, `spawned++`입니다.
- failure 뒤 cleanup 또는 상태: partial spawn이면 recorded range만 exact PID wait하고 status 1; complete spawn일 때만 final slot result를 처리 단계 status로 사용합니다.

#### `ae988017efd5`에서 확인할 실제 코드
- PID table allocation이 `command_count * sizeof(pid_t)`입니다.
- Fork loop는 command order이고 PID index와 command index가 같습니다.
- Wait loop는 `waitpid(pids[i], ...)`로 exact child를 관찰합니다.
- `spawned == command_count`일 때만 last status가 의미 있습니다.
- 이 SHA에는 pipe table/dup2 topology가 아직 없습니다.

#### 학습자가 남길 코드 증거
- PID table index ↔ command index mapping: `pids[0]`은 first command, `pids[N-1]`은 last command입니다.
- recorded count mutation: fork 성공 뒤에만 `spawned`를 증가시켜 valid PID range를 정의합니다.
- complete/partial status branch: full count면 last child status, otherwise all recorded children wait 후 1입니다.
- wait 소유권 종료 지점: each exact PID의 `waitpid` success 후 해당 slot lifecycle responsibility가 끝납니다.
- 아직 없는 descriptor topology: child들이 fork되지만 stdin/stdout pipe wiring은 없습니다.
- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: 처리 단계 dispatcher → `run_forked_commands` → fork loop → exact wait loop.
- parent SHA와 비교한 최소 before/after snippet: single PID local에서 command-count PID table + spawned count로 확장됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact source에서 bookkeeping과 status branch를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: parent는 child identity를 결정적으로 소유·관찰하고 처리 단계 status를 last stage에 연결할 bookkeeping을 갖습니다.
- 아직 보장하지 않는 것: child 사이 data flow와 mid-fork block/hang 회복은 아직 해결하지 않습니다.

#### Thread 내 다음 연결
`a71f98de0d92`가 PID skeleton에 N-1 pipe graph와 descriptor closure를 연결합니다.

### 5.3 `a71f98de0d92` — `feat(exec): 다단 pipeline의 pipe FD 연결`

#### 확정 정보
- SHA: `a71f98de0d92`
- Subject: `feat(exec): 다단 pipeline의 pipe FD 연결`
- Importance: **S**
- Tags: `PROCESS`, `FD_IO`, `CORE`
- Source-defined role: Connects the multi-stage pipe graph and defines parent/child descriptor closure.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
N command에 대해 N-1 pipe를 만들고, child i의 stdin/stdout을 neighboring pipe에 연결한 뒤 모든 original pipe end를 닫고 explicit redirection을 나중에 적용합니다.

#### Source가 확정한 핵심 판단
- **문제**: Multiple forked commands do not form a 처리 단계 unless each child receives the correct neighboring descriptors and every process closes all unused pipe ends.
- **결정**: Allocate `N - 1` pipes for `N` commands, map the previous read end to stdin and the next write end to stdout in each child, close all original ends, then apply explicit command redirections afterward.
- **중요한 이유**: The ordering defines both 데이터 흐름 and redirection precedence. Parent and child closure rules prevent readers from waiting forever on hidden writers, while child execution of 처리 단계 builtins prevents state mutations from leaking into the parent shell.
- **확정된 변경 범위**: The executor gained pipe-table creation and cleanup, per-stage descriptor duplication, one child per command, parent-side closure and reaping, last-stage status selection, and parent-only execution for a single stateful builtin.
- **프로젝트 이해에서의 위치**: This is the defining process topology. It is the basis for every later fork-failure, descriptor-leak, timeout, and child-lifecycle correction.

#### 설계·상태 변화 기록
- 이 commit 직전 상태: multiple child PID lifecycle만 있고 data flow는 서로 독립적이었습니다.
- 해결하려던 문제: N stages를 N-1 edges로 연결하고 parent/each child가 불필요한 pipe ends를 닫아 EOF semantics를 보장해야 했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: inherited hidden write end 하나만 남아도 downstream reader가 EOF를 기다리며 block할 수 있고, redirection precedence가 정의되지 않으면 `cmd | next <file` 의미가 불명확합니다.
- 선택한 결정: all pipes를 fork 전에 생성하고 stage index로 previous read/current write end를 dup한 뒤 child는 모든 originals를 close하고 explicit redirections를 적용합니다. Parent는 spawn 뒤 all pipe ends를 close하고 exact PIDs를 wait합니다.
- publish 또는 state mutation이 일어나는 지점: pipe() success마다 table slots가 live FD로 바뀌고, fork success마다 PID slot이 parent-owned가 됩니다. Child `dup2`가 stdio binding을 publish합니다.
- failure 뒤 cleanup 또는 상태: partial pipe creation은 initialized `-1` slots를 사용해 created ends만 close합니다. Fork short sequence는 parent ends를 close하고 existing children을 wait하지만 아직 terminate하지 않아 hang risk가 남습니다.

#### `a71f98de0d92`에서 확인할 실제 코드
- Pipe table은 `pipe_count = command_count - 1`, each pair `[read, write]`입니다.
- Slots는 `-1`로 초기화되어 partial cleanup이 valid FDs만 close합니다.
- Child i는 `i > 0`이면 pipe `i-1` read를 stdin, `i+1 < N`이면 pipe `i` write를 stdout에 dup합니다.
- Dup 후 child는 필요 여부와 상관없이 table의 all original FDs를 close합니다.
- Explicit redirection은 pipe wiring 뒤 적용됩니다.
- Parent는 all fork attempts 뒤 pipe ends를 close하고 waits합니다.
- Single parent builtin만 parent path이고 처리 단계 builtin은 child path입니다.

#### 학습자가 남길 코드 증거
- N-stage descriptor graph:

```text
cmd0 --pipe0--> cmd1 --pipe1--> ... --pipe(N-2)--> cmd(N-1)
```

- child i의 dup2 formula: stdin ← `pipes[i-1][0]` if `i>0`; stdout ← `pipes[i][1]` if `i<N-1`.
- parent/child close matrix:

| Process | dup target | originals after dup/fork |
| --- | --- | --- |
| child i | neighboring read/write only | all pipe table ends close |
| parent | none | all pipe table ends close after spawn loop |

- pipe wiring vs explicit redirection order: pipe default first, then command redirections override stdio in source order.
- 처리 단계 builtin isolation: command count > 1이면 builtin도 child에서 실행되어 parent env/cwd에 반영되지 않습니다.
- partial failure에 남은 위험: later fork failure 시 first stages가 incomplete graph에서 read/write/sleep block하고 parent의 simple wait가 끝나지 않을 수 있습니다.
- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: `run_forked_commands` → pipe table init/create → fork loop → child pipe setup/close/redirections → parent close → wait loop.
- parent SHA와 비교한 최소 before/after snippet:

```c
if (index > 0)
    dup2(pipes[index - 1][0], STDIN_FILENO);
if (index + 1 < command_count)
    dup2(pipes[index][1], STDOUT_FILENO);
close_all_pipes(pipes, pipe_count);
apply_redirections(...);
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. 3-stage index mapping과 all-close loops를 exact source에서 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: multi-stage 데이터 흐름, descriptor precedence, parent/child closure, last-stage status의 defining execution topology를 제공합니다.
- 아직 보장하지 않는 것: later fork failure 시 already spawned child를 terminate하지 않아 block/hang할 수 있는 실패 처리가 남습니다.

#### Thread 내 다음 연결
`915aa072298b`가 pipe/fork/wait failure를 재현할 seam을 만들고 `be2967a4b946`가 lifecycle invariant를 복구합니다.

### 5.4 `915aa072298b` — `refactor(runtime): 프로세스 시스템 호출 경계 분리`

#### 확정 정보
- SHA: `915aa072298b`
- Subject: `refactor(runtime): 프로세스 시스템 호출 경계 분리`
- Importance: **A**
- Tags: `PROCESS`, `FAILURE`, `TEST`
- Source-defined role: Introduces deterministic pipe, fork, and wait failure seams.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
`pipe`, `fork`, `waitpid`를 runtime wrapper 뒤로 옮기고 selected call count에서 representative `errno`로 실패시키는 deterministic test seam을 추가합니다. Production wrapper는 transparent합니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: raw system calls의 rare later-call failure를 반복 재현할 수 없었습니다.
- 새 boundary가 제공하는 contract: production에서는 raw call을 그대로 위임하고 test build에서 operation별 N번째 call을 deterministic하게 실패시킵니다.
- production semantics가 유지된다는 코드 근거: `#ifdef SMALL_SHELL_TESTING` injection branch 외에는 wrapper가 arguments/return/errno를 raw call에 그대로 전달합니다.
- 소유권 또는 call-site responsibility 변화: executor의 cleanup policy는 변하지 않고 호출 지점만 wrappers를 사용합니다. 따라서 seam 자체는 자원 owner가 아닙니다.
- 후속 fix/test가 이 seam을 사용하는 방식: second pipe/second fork/first wait failure로 partial acquisition/spawn states를 만들고 cleanup을 검증합니다.

#### `915aa072298b`에서 확인할 실제 코드
- `src/runtime.h/.c`의 `shell_pipe`, `shell_fork`, `shell_waitpid`를 확인했습니다.
- Test-only counters와 `fail_call`이 environment-selected occurrence를 비교합니다.
- Failure errno는 operation별 representative value이며 normal branch는 raw call입니다.
- Executor diff는 raw call names를 wrapper names로 치환하고 recovery policy는 그대로 둡니다.

#### 학습자가 남길 코드 증거
- wrapper API와 raw syscall mapping: `shell_pipe`→`pipe`, `shell_fork`→`fork`, `shell_waitpid`→`waitpid`.
- production/test branch: test macro에서만 counters/env checks; production은 direct delegation입니다.
- call-index injection state: operation별 static call count를 increment한 뒤 target occurrence와 비교합니다.
- later-call failure가 만드는 partial state: second pipe failure는 one pipe pair live, second fork failure는 one recorded child live입니다.
- 아직 unchanged인 cleanup policy: partial fork child를 terminate하지 않고 close/wait만 합니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, `src/exec.c`, build definitions.
- 핵심 caller → callee: executor → `shell_pipe`/`shell_fork`/`shell_waitpid` → injection check or raw call.
- parent SHA와 비교한 최소 before/after snippet: raw system 호출 지점이 transparent wrappers로 치환됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Test/production branches와 counters를 source로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: rare process-resource failure를 production behavior 변경 없이 반복 재현할 수 있습니다.
- 아직 보장하지 않는 것: seam은 관찰 가능성만 제공하며 partial child/FD cleanup을 아직 수정하지 않습니다.

#### Thread 내 다음 연결
`be2967a4b946`가 이 seam으로 드러난 mid-construction ownership 문제를 수정합니다.

### 5.5 `be2967a4b946` — `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수`

#### 확정 정보
- SHA: `be2967a4b946`
- Subject: `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수`
- Importance: **S**
- Tags: `PROCESS`, `FD_IO`, `FAILURE`
- Source-defined role: Terminates and reaps children after partial 처리 단계 construction.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
Later fork failure 시 parent-held pipe ends를 모두 닫고, 이미 기록된 child에 `SIGKILL`을 보낸 뒤 every PID를 reap하여 처리 단계 execution이 complete cleanup state로 수렴하도록 합니다.

#### Source가 확정한 핵심 판단
- **문제**: If a later fork failed, already spawned stages could remain blocked or running. Closing descriptors and waiting was insufficient and could hang indefinitely or leave zombies.
- **결정**: On partial construction, close all parent-held pipe ends, send termination to every recorded child, tolerate children that already exited, and still reap every PID. Treat any wait failure as 처리 단계 failure.
- **중요한 이유**: Recording a PID transfers lifecycle responsibility to the parent even when the 처리 단계 never becomes complete. The 정리 과정 must converge to the same terminal 소유권 state as successful execution.
- **확정된 변경 범위**: The executor gained child termination, structured wait retry and error reporting, status suppression when the last child was not observed cleanly, and complete cleanup after a short spawn sequence.
- **프로젝트 이해에서의 위치**: This commit converts the 처리 단계 from a normal-path mechanism into a reliable lifecycle owner. It explains the later fault-injection and leak-verification architecture.

#### Fix 재구성 기록
- 기존 가정: parent pipe ends를 닫고 recorded children을 ordinary wait하면 partial 처리 단계도 자연 종료한다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: first child가 `sleep 30`이거나 incomplete pipe를 기다리는 상태에서 second fork가 실패하면 wait가 장시간 block합니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: PID가 recorded된 순간 parent 소유권이 생겼지만 incomplete graph에서 child progress를 보장할 termination policy가 없었습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: partial spawn이면 parent FDs close → every recorded child signal → every recorded PID reap 순서로 강제 수렴합니다.
- 변경 전 코드 증거: `spawned < command_count`에서도 same wait loop만 사용했습니다.
- 변경 후 코드 증거: `terminate_children`와 structured `wait_for_child`를 추가하고 partial branch가 kill/reap를 호출합니다.
- 연결되는 regression test와 그 한계: `d611196b368e`가 mid-fork failure와 wait failure를 deterministic하게 검증하고 `b42e57eb7755`가 direct child probe를 추가합니다. Arbitrary descendants까지 증명하지 않습니다.

#### `be2967a4b946`에서 확인할 실제 코드
- Parent SHA의 partial branch와 simple wait를 비교했습니다.
- Parent pipe close가 termination 전에 일어납니다.
- Recorded range만 `kill(pid, SIGKILL)`하며 already-exited child 오류를 허용합니다.
- Signal 후 every PID를 `wait_for_child`로 관찰합니다.
- Wait helper는 `EINTR`를 계속 retry하고 non-EINTR error 후 제한된 재시도를 하며 any error evidence를 남깁니다.
- Any wait failure면 otherwise valid last-stage status를 버리고 처리 단계 status 1입니다.
- Success/partial paths 모두 pipe FDs closed, PIDs observed, tables freed terminal state로 수렴합니다.

#### 학습자가 남길 코드 증거
- 기존 가정: descriptor closure alone guarantees child exit.
- block 가능한 concrete 처리 단계 scenario: `sleep 30 | cat`에서 second fork failure 후 `sleep`은 own timer까지 살아 있고 parent wait가 block합니다.
- root cause: recorded child가 incomplete graph에서 살아 있음입니다.
- close → signal → reap 순서: parent originals close → `SIGKILL` loop over `[0, spawned)` → exact wait loop over same range.
- wait failure가 status를 override하는 조건: any PID가 cleanly observed되지 않았다는 flag가 있으면 last-stage result를 사용하지 않고 1입니다.
- cleanup convergence 표:

| Path | Parent FDs | Recorded children | Tables | Result |
| --- | --- | --- | --- | ---: |
| complete | close | exact wait | free | last clean stage status |
| partial fork | close | kill then exact wait | free | 1 |
| wait error | already closed | retry/observe as possible | free | 1 |

- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: `run_forked_commands` partial branch → close pipes → `terminate_children` → `wait_for_child` for every PID → free tables.
- parent SHA와 비교한 최소 before/after snippet:

```text
before: partial fork → close pipes → wait
 after: partial fork → close pipes → kill recorded PIDs → wait every PID
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact parent/fix diff와 terminal 소유권 states를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: recorded PID는 처리 단계 완성 여부와 무관하게 parent가 종료하거나 관찰하며, partial construction이 hang/zombie를 남기지 않습니다.
- 아직 보장하지 않는 것: descriptor save/restore failure와 direct lifecycle stress 검증은 후속 commits가 담당합니다.

#### Thread 내 다음 연결
`d611196b368e`가 later pipe/fork/wait failures를 deterministic regression으로 고정합니다.

### 5.6 `d611196b368e` — `test(exec): pipe·fork·wait 실패 회귀 검증`

#### 확정 정보
- SHA: `d611196b368e`
- Subject: `test(exec): pipe·fork·wait 실패 회귀 검증`
- Importance: **A**
- Tags: `TEST`, `PROCESS`, `FAILURE`
- Source-defined role: Reproduces pipe, mid-fork, and wait failure regressions.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
`SMALL_SHELL_TESTING` fault binary로 later pipe creation, mid-처리 단계 fork, waitpid failure를 주입하고 각 failed 처리 단계가 status 1로 끝난 뒤 following `echo $?`가 이를 관찰하는지 검증합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: partial acquisition/spawn과 wait observation failure가 hang, zombie, false last-stage success를 남기지 않아야 합니다.
- 재현하는 failure 또는 boundary: second pipe creation, second fork, first waitpid failure입니다.
- 사용한 test technique: production source를 `SMALL_SHELL_TESTING`으로 재build하고 runtime wrapper call counter를 environment로 선택하는 deterministic failure injection입니다.
- 실제 통과하는 production code path: pipe table create cleanup, partial spawn close/kill/reap, wait error override, shell line continuation/status expansion입니다.
- 이 테스트가 증명하는 것: 각 injected failure가 처리 단계 status 1로 귀결되고 same shell의 next `echo $?`가 1을 출력하며 test가 종료됩니다.
- 이 테스트가 증명하지 않는 것: 장기간 FD 누수, all call positions, arbitrary child descendants, real kernel resource pressure는 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: operation/call-position-specific deterministic regression입니다.
- 후속 변경에서 막는 회귀: partial child termination 제거, wait error 무시, partial pipe cleanup 누락, failure 뒤 status continuation 손실입니다.

#### `d611196b368e`에서 확인할 실제 코드
- `Makefile`의 `small-shell-test`가 same production sources에 test definition만 추가합니다.
- `tests/faults.sh`는 later call occurrence를 선택합니다.
- Mid-fork fixture는 long-running first child를 사용해 kill/reap가 없으면 종료가 지연되도록 만듭니다.
- Each input은 failed 처리 단계 뒤 `echo $?`를 같은 process에 제공합니다.
- Wait failure case는 otherwise successful last-stage result보다 status 1이 우선함을 검사합니다.

#### 학습자가 남길 코드 증거
- 대상 production 항상 유지해야 하는 조건: every acquired FD/recorded PID reaches cleanup and only cleanly observed last-stage status is trusted.
- 각 injected operation/call position: pipe #2, fork #2, waitpid #1입니다.
- partial resource state: one pipe pair 또는 one child PID가 이미 parent-owned인 상태입니다.
- production 정리 과정: created pipe ends close; partial child kill/reap; tables free; wait error flag forces 1.
- expected status/output: process remains able to read next line, `echo $?` outputs `1`.
- 증명하는 것과 증명하지 않는 것: deterministic selected branches를 증명하지만 stress/exhaustion과 every errno는 증명하지 않습니다.
- deterministic regression 근거: identical environment input selects the same wrapper call and expected exact output.
- 확인한 변경 파일: `Makefile`, `tests/faults.sh`, test-target supporting runtime code.
- 핵심 caller → callee: script → fault binary → runtime wrapper → executor failure cleanup → next command expansion.
- parent SHA와 비교한 최소 before/after snippet: production code change 없이 fault target와 three regression cases가 build/test graph에 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Scripts, injection selectors, expected outputs를 source로 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: partial process construction과 wait failure가 hang, zombie, false success로 바뀌지 않는다는 regression evidence를 제공합니다.
- 아직 보장하지 않는 것: long-run FD exhaustion과 direct post-pipeline child probe는 `b42e57eb7755`가 추가로 검증합니다.

#### Thread 내 다음 연결
`fd5c76c18c27`가 같은 runtime-boundary pattern을 descriptor operations로 확장합니다.

### 5.7 `fd5c76c18c27` — `refactor(runtime): FD 시스템 호출 경계 분리`

#### 확정 정보
- SHA: `fd5c76c18c27`
- Subject: `refactor(runtime): FD 시스템 호출 경계 분리`
- Importance: **A**
- Tags: `FD_IO`, `FAILURE`, `TEST`
- Source-defined role: Extends the runtime boundary to descriptor duplication and opening.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
`open`, `dup`, `dup2`를 `shell_open`, `shell_dup`, `shell_dup2` wrapper 뒤로 옮기고, selected position부터 반복 실패하는 repeat mode를 추가합니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: parent stdio save/apply/restore의 특정 later call과 retry까지 연속 실패시키기 어려웠습니다.
- 새 boundary가 제공하는 contract: one-shot N번째 failure와 N번째부터 계속 실패하는 repeat mode를 제공하고 production에서는 raw operation을 그대로 위임합니다.
- production semantics가 유지된다는 코드 근거: test-only selector branch 외 wrapper arguments/return/errno는 raw `open`/`dup`/`dup2`와 동일합니다.
- 소유권 또는 call-site responsibility 변화: descriptor 소유권은 executor/redirection caller에 남고 wrappers는 only call boundary입니다.
- 후속 fix/test가 이 seam을 사용하는 방식: save failure, apply failure, one-shot restore failure, retry까지 실패하는 persistent restore failure를 별도로 재현합니다.

#### `fd5c76c18c27`에서 확인할 실제 코드
- `src/runtime.h/.c`의 new wrappers와 호출 지점을 확인했습니다.
- 처리 단계 child wiring, file redirection, heredoc installation, parent stdio save/restore가 wrapper를 사용합니다.
- One-shot mode는 selected call 한 번만 실패하고 repeat mode는 target 이후 matching operation을 계속 실패시킵니다.
- Production build는 injection state를 포함하지 않습니다.
- 이 commit은 error policy 자체를 바꾸지 않고 seam만 확장합니다.

#### 학습자가 남길 코드 증거
- descriptor wrapper coverage map:

| Wrapper | 주요 호출 지점 |
| --- | --- |
| `shell_open` | input/output/append file redirection |
| `shell_dup` | parent stdin/stdout save |
| `shell_dup2` | pipe wiring, redirection apply, heredoc install, parent restore |

- one-shot vs repeat injection: one-shot은 failure 뒤 selector가 다시 발동하지 않고, repeat은 restore helper의 second attempt도 실패시킵니다.
- restore retry에 필요한 repeated failure scenario: first `dup2(saved, target)` failure만으로는 retry 성공이 가능하므로 permanent failure 검증에는 repeat mode가 필요합니다.
- production transparency 근거: non-test branch returns raw call result directly.
- 후속 fix가 사용할 observability point: `restore_stdio`의 per-descriptor `shell_dup2` attempts입니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, `src/exec.c`, `src/redirection.c`.
- 핵심 caller → callee: parent/child redirection code → shell FD wrapper → test selector or raw call.
- parent SHA와 비교한 최소 before/after snippet: raw descriptor calls replaced with wrappers; repeat selector added.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Wrapper coverage and repeat state를 source로 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: descriptor save, application, restoration, open exhaustion/permission failure를 하나의 deterministic boundary에서 재현할 수 있습니다.
- 아직 보장하지 않는 것: wrapper 자체는 restore failure를 어떻게 처리할지 결정하지 않습니다.

#### Thread 내 다음 연결
`2ca9f4299c7f`가 recoverable/unrecoverable restore policy를 구현합니다.

### 5.8 `2ca9f4299c7f` — `fix(redirection): 부모 표준 입출력 복원 실패 전파`

#### 확정 정보
- SHA: `2ca9f4299c7f`
- Subject: `fix(redirection): 부모 표준 입출력 복원 실패 전파`
- Importance: **A**
- Tags: `FD_IO`, `FAILURE`, `RISK`
- Source-defined role: Makes parent standard-stream restoration failure observable and fatal when unrecoverable.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Parent-executed command의 stdin/stdout restore를 독립적으로 retry하고, transient error도 status 1로 남기며, 어느 descriptor라도 복구 불가능하면 `running`을 clear해 shell을 중단합니다.

#### Fix 재구성 기록
- 기존 가정: parent builtin 뒤 saved stdin/stdout `dup2` 복원은 실패하더라도 diagnostic만 내거나 결과를 무시해도 다음 command를 실행할 수 있다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: stdout이 redirected target에 남거나 stdin이 wrong source에 남으면 다음 prompt/command의 I/O가 예측 불가능합니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: process-persistent descriptors를 temporary command state로 바꾼 뒤 restoration success를 next-command precondition으로 취급하지 않았습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: stdin과 stdout을 독립적으로 복원하고 `EINTR`는 retry, other one-shot failure도 한 번 더 시도합니다. Error가 한 번이라도 있었으면 status 1, final failure면 `running=0`입니다.
- 변경 전 코드 증거: restoration result를 caller-visible outcome에 반영하지 않았습니다.
- 변경 후 코드 증거: `restore_one` tri-state와 `restore_stdio` aggregate result가 command status와 shell running을 결정합니다.
- 연결되는 regression test와 그 한계: `13645f58d5e6`가 save/apply/open/restore and repeat failure를 분리합니다. Arbitrary close/flush errors 전체는 포괄하지 않습니다.

#### `2ca9f4299c7f`에서 확인할 실제 코드
- Parent command는 stdin copy, stdout copy를 `shell_dup`으로 획득하며 second save failure면 first copy를 close합니다.
- Redirection apply 또는 builtin path 뒤 both descriptors를 restore합니다.
- `restore_one`은 `EINTR`을 계속 retry하고 non-EINTR error 뒤 limited second attempt를 수행합니다.
- Retry 성공도 prior error evidence 때문에 status 1입니다.
- Final failure는 diagnostic 후 `shell->running = 0`입니다.
- Saved copies는 outcome과 무관하게 close됩니다.
- Redirection setup failure도 same restore 정리 과정으로 합류합니다.
- Builtin output flush는 original stdout restore 전에 확인됩니다.

#### 학습자가 남길 코드 증거
- 기존 best-effort assumption: restore failure는 current command diagnostic일 뿐 future command safety와 무관하다는 가정입니다.
- stdin/stdout save/apply/restore state table:

| Restore outcome | Descriptor state | Command status | `running` |
| --- | --- | ---: | ---: |
| first attempt success | restored | original command result | unchanged |
| first error, retry success | restored | 1 | unchanged |
| retry도 실패 | untrusted | 1 | 0 |

- transient failure 후 recovered state/status: both descriptors 복원은 완료되지만 error evidence를 숨기지 않아 1입니다.
- unrecoverable failure 후 descriptor/running state: target descriptor를 신뢰할 수 없으므로 remaining input을 실행하지 않고 stop합니다.
- setup failure와 normal execution이 합류하는 정리 과정: both go to restore + close saved descriptors.
- 확인한 변경 파일: `src/exec.c`/parent execution and restoration helpers.
- 핵심 caller → callee: parent command executor → save stdio → apply redirections/builtin → flush → `restore_stdio` → `restore_one`.
- parent SHA와 비교한 최소 before/after snippet:

```text
restore result 0  → no restore error
restore result 1  → eventually restored, command status 1
restore result -1 → shell->running = 0, command status 1
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact restore loop and caller state mutations를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 다음 command 전에 parent descriptors가 신뢰 가능한 상태로 복구되거나 shell이 중단되며, restore 오류가 성공으로 숨겨지지 않습니다.
- 아직 보장하지 않는 것: failure matrix의 실제 회귀 증거는 다음 test commit에서 제공합니다.

#### Thread 내 다음 연결
`13645f58d5e6`가 save, open, apply, restore, repeated unrecoverable failure를 분리해 검증합니다.

### 5.9 `13645f58d5e6` — `test(redirection): 저장·적용·복원 실패 회귀 검증`

#### 확정 정보
- SHA: `13645f58d5e6`
- Subject: `test(redirection): 저장·적용·복원 실패 회귀 검증`
- Importance: **A**
- Tags: `TEST`, `FD_IO`, `FAILURE`
- Source-defined role: Exercises save, application, restoration, open, and persistent failure paths.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Parent redirection의 descriptor save, target open, replacement apply, original restore 각 phase에 failure를 주입하고, recoverable case의 continuation과 repeated restore failure의 forced stop을 검증합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: parent command는 state mutation 전에 stdio setup이 성공해야 하고, command 뒤 stdio가 restored이거나 shell이 stopped여야 합니다.
- 재현하는 failure 또는 boundary: stdin save dup, stdout save dup, output open, replacement dup2, stdin restore, stdout restore, persistent restore입니다.
- 사용한 test technique: operation/call-position fault injection과 same-process next-command assertions입니다.
- 실제 통과하는 production code path: parent command save → redirection open/apply → builtin → flush → restore retry/stop.
- 이 테스트가 증명하는 것: setup failures suppress command payload/state effect, recoverable errors yield status 1 and continuation on normal stdout, repeated restore failure suppresses following command and exits status 1.
- 이 테스트가 증명하지 않는 것: long-run descriptor exhaustion, all close errors, child 처리 단계 redirection combinations은 별도입니다.
- broad integration / deterministic regression / stress·probe 중 분류: phase-specific deterministic regression matrix입니다.
- 후속 변경에서 막는 회귀: save cleanup 누락, failure 후 builtin execution, restore error suppression, unsafe continuation입니다.

#### `13645f58d5e6`에서 확인할 실제 코드
- `tests/faults.sh`의 case마다 `dup`, `open`, `dup2` operation과 occurrence가 다릅니다.
- One-shot setup failure는 builtin output/state effect가 발생하지 않고 current status 1입니다.
- Recoverable restore case 뒤 marker는 original stdout에서 관찰됩니다.
- Repeat-mode restore case는 retry까지 실패하고 `echo never`가 출력되지 않으며 diagnostic/status 1입니다.
- Test input은 exact parent-executed builtin path를 사용합니다.

#### 학습자가 남길 코드 증거
- 대상 production 항상 유지해야 하는 조건: parent stdio lifecycle is transactional across save/apply/execute/restore.
- phase별 injected operation: save=`dup`, target acquisition=`open`, apply/restore=`dup2` with selected occurrence.
- expected command execution 여부: save/open/apply failure에서는 command body를 실행하지 않습니다.
- following command behavior: reliable restoration이면 executes on normal stdio; persistent restore failure이면 not executed.
- forced stop 조건: repeat mode가 restore first and retry attempts를 모두 실패시킵니다.
- broad integration 또는 deterministic regression 판정: deterministic matrix; production parent path를 end-to-end 통과합니다.
- 증명하지 않는 path: child-only redirection, every errno/close failure, cumulative leaks입니다.
- 확인한 변경 파일: `tests/faults.sh`, runtime wrappers/build target.
- 핵심 caller → callee: harness → fault binary → parent executor save/apply/restore → status/running → following input.
- parent SHA와 비교한 최소 before/after snippet: restore policy fix 뒤 phase별 fixtures/assertions가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Case matrix와 expected output/status를 source로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: parent descriptor lifecycle의 negative 동작과 recovery behavior, unrecoverable stop decision을 regression으로 고정합니다.
- 아직 보장하지 않는 것: 장기 반복에서 FD 누적이 없는지와 direct child leak은 다음 lifecycle test가 검증합니다.

#### Thread 내 다음 연결
`b42e57eb7755`가 repeated mixed workload에서 descriptor/child ownership을 직접 관찰합니다.

### 5.10 `b42e57eb7755` — `test(lifecycle): FD와 자식 프로세스 누수 검증`

#### 확정 정보
- SHA: `b42e57eb7755`
- Subject: `test(lifecycle): FD와 자식 프로세스 누수 검증`
- Importance: **A**
- Tags: `TEST`, `PROCESS`, `FD_IO`
- Source-defined role: Directly checks for descriptor exhaustion and unreaped children.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
48-descriptor limit 아래 parent redirection, three-stage 처리 단계, file I/O를 반복하고, test-only `waitpid(-1, ..., WNOHANG)` probe로 live/zombie direct child가 남는지 검사합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: repeated normal execution 뒤 FD count가 누적되지 않고 each direct child가 already reaped돼야 합니다.
- 재현하는 failure 또는 boundary: single-run output으로 보이지 않는 descriptor leak과 unreaped/zombie child accumulation입니다.
- 사용한 test technique: low FD limit stress workload + test-only post-처리 단계 direct child probe + timeout/process-group harness입니다.
- 실제 통과하는 production code path: parent builtin redirection save/apply/restore, 3-stage pipe creation/fork/close/wait, file input/output redirection, timeout cleanup입니다.
- 이 테스트가 증명하는 것: configured repeated workload가 descriptor exhaustion 없이 final marker에 도달하고 test process에 waitable/live direct child가 남지 않습니다.
- 이 테스트가 증명하지 않는 것: all descendant grandchildren, unlimited workloads, every OS 자원 누수, production build에 probe가 존재함을 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: broad lifecycle stress와 deterministic direct-child probe의 결합입니다.
- 후속 변경에서 막는 회귀: parent saved FD close 누락, pipe end close 누락, child reap 누락, timeout child orphaning입니다.

#### `b42e57eb7755`에서 확인할 실제 코드
- `tests/lifecycle.sh`가 subshell의 descriptor limit을 48로 설정합니다.
- Workload는 60회 parent redirection, three-stage 처리 단계, temporary file write/read를 섞습니다.
- Expected stdout는 final `fd-ok` marker이고 stderr는 비어 있어야 합니다.
- Test-only `shell_children_reaped`는 `waitpid(-1, ..., WNOHANG)`을 반복해 child가 있거나 probe가 직접 하나라도 reap하면 failure로 간주하고, `ECHILD`에서 only no-found success입니다.
- Timeout fixture는 long-running 처리 단계가 timeout status 124가 되는지 확인합니다.
- Timeout runner는 launched 프로세스 그룹을 종료·회수해 orphan을 피합니다.
- Probe는 `SMALL_SHELL_TESTING` 안에만 있습니다.

#### 학습자가 남길 코드 증거
- 대상 resource 항상 유지해야 하는 조건: no cumulative parent FD 소유권 and no unreaped direct child after a 처리 단계 returns.
- workload 반복 횟수/구성: 60 iterations, parent redirection + 3-stage pipe + file copy/read/write.
- FD exhaustion 관찰 방식: `ulimit -n 48` 아래 final marker 도달; leak가 누적되면 later `open`/`pipe`/`dup`이 실패합니다.
- direct child probe 결과 해석: `waitpid` returns child PID이면 leak/zombie; 0이면 live child; `-1/ECHILD` and no prior found만 success입니다.
- timeout/process-group 관련 assertion: 30-second child 처리 단계를 short timeout으로 끊고 124를 요구하며 runner가 프로세스 그룹을 cleanup합니다.
- broad stress와 deterministic probe의 역할 구분: FD는 exhaustion stress로 간접 관찰, child는 waitpid probe로 직접 관찰합니다.
- 증명하지 않는 descendant 범위: direct children만 확인하며 grandchildren/external daemonization은 범위 밖입니다.
- 확인한 변경 파일: `tests/lifecycle.sh`, test-only executor hook, `tests/timeout_runner.c`, `Makefile`.
- 핵심 caller → callee: lifecycle script → test binary repeated workloads → executor cleanup → test-only child probe; timeout script → runner → 프로세스 그룹 cleanup.
- parent SHA와 비교한 최소 before/after snippet: existing functional fault suite 옆에 long-run lifecycle suite와 direct probe가 추가됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. 반복 횟수, FD limit, probe result interpretation, timeout expectations를 source로 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 정상 output만으로 보이지 않는 descriptor 누적과 direct child lifecycle을 반복 workload와 direct probe로 관찰합니다.
- 아직 보장하지 않는 것: 모든 종류의 descendant process나 무한한 workload를 증명하지 않으며 test-specific probe는 production에 포함되지 않습니다.

#### Thread 내 다음 연결
`6dff1ba86ba6`가 preparation 단계의 남은 narrow PID-table leak을 닫습니다.

### 5.11 `6dff1ba86ba6` — `fix(exec): pipe 생성 실패 시 PID 배열 해제`

#### 확정 정보
- SHA: `6dff1ba86ba6`
- Subject: `fix(exec): pipe 생성 실패 시 PID 배열 해제`
- Importance: **B**
- Tags: `FD_IO`, `FAILURE`, `DEBUG`
- Source-defined role: Closes the remaining PID-table leak before any child is spawned.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
PID table과 pipe table을 모두 할당한 뒤 pipe creation이 실패하는 return path에서 opened pipe ends, pipe table뿐 아니라 아직 local인 PID table도 해제합니다.

#### Fix 재구성 기록
- 기존 가정: pipe creation failure cleanup에서 pipe-related resources만 정리하면 된다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: prior allocation-order fix로 PID table을 pipe creation 전에 확보하므로 second/first pipe failure 시 unused PID table allocation이 live입니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: acquisition order가 바뀌었지만 old failure label의 cleanup list에 newly preallocated PID table이 추가되지 않았습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: every local preparation allocation is listed in every exit after its acquisition, even before child spawn.
- 변경 전 코드 증거: partial pipe ends close + pipe table free + return, without `free(pids)`.
- 변경 후 코드 증거: same branch에 PID table free 한 줄이 추가됩니다.
- 연결되는 회귀 테스트와 그 한계: 이 commit에는 전용 test가 없습니다. Existing pipe fault seam, allocation sweep, later sanitizer path가 관찰 수단이지만 실제 실행 결과를 이 commit에 소급하지 않습니다.

#### `6dff1ba86ba6`에서 확인할 실제 코드
- Executor는 PID table과 pipe table을 먼저 allocate하고 descriptor slots를 `-1`로 init한 뒤 `shell_pipe` loop를 시작합니다.
- Pipe failure 시 child count는 0입니다.
- Close helper는 `-1` slots를 skip하고 created ends만 close합니다.
- Failure return 전에 both tables가 free됩니다.
- Diff의 functional change는 missing PID table free입니다.

#### 학습자가 남길 코드 증거
- acquisition list: PID table → pipe table → each pipe pair; child PID는 아직 없습니다.
- failure 시점의 live resources: both memory tables와 prefix of opened pipe ends입니다.
- cleanup list 전/후: 전에는 opened FDs + pipe table, 후에는 opened FDs + pipe table + PID table입니다.
- child count가 0인 근거: create-all-pipes loop가 fork loop보다 앞섭니다.
- narrow leak의 관찰 방법: selected pipe failure를 repeated invocation하고 ASan/LSan 또는 allocation accounting으로 PID table leak를 관찰할 수 있습니다. 이 환경에서는 실행하지 않았습니다.
- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: 처리 단계 executor → table allocations → pipe creation loop → failure close/free branch.
- parent SHA와 비교한 최소 before/after snippet:

```c
close_all_pipes(pipes, pipe_count);
free(pipes);
free(pids); /* 6dff1ba86ba6에서 추가 */
return 1;
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact one-line diff와 acquisition order를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: first child가 생기기 전 pipe creation failure에서도 preparation memory가 모두 회수됩니다.
- 아직 보장하지 않는 것: project-wide 소유권 model을 바꾸지 않는 narrow cleanup fix입니다.

#### Thread 내 다음 연결
처리 단계 Thread의 마지막 commit입니다. 최종 ledger에서 모든 acquisition path가 terminal cleanup state로 수렴하는지 확인합니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 명시한 항상 유지해야 하는 조건과 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.

| 항상 유지해야 하는 조건 | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Every recorded child PID remains parent-owned until termination or observation. | parent가 PID를 기록한 뒤에는 pipeline이 완성되지 않아도 해당 child를 종료하거나 wait로 관찰해야 합니다. | `ae988017efd5` | `be2967a4b946`, `d611196b368e`, `b42e57eb7755` | Fork success 직후 PID slot/`spawned` publish; partial branch close→kill→exact wait; wait fault status override; test-only no-child probe. |
| Each process closes every pipe end it does not need. | 숨은 writer/read end가 EOF와 resource lifetime을 방해하지 않도록 parent와 child 모두 불필요한 pipe end를 닫아야 합니다. | `a71f98de0d92` | `be2967a4b946`, `b42e57eb7755` | Child dup formula 뒤 all originals close, parent spawn loop 뒤 all ends close, failure도 close before kill/reap; low-FD stress observes accumulation. |
| Parent standard streams are restored before the next command. | restore가 불가능하면 이후 command I/O가 신뢰 불가능하므로 shell은 계속 실행하지 않습니다. | `fd5c76c18c27`의 injectable boundary | `2ca9f4299c7f`, `13645f58d5e6` | save/apply/restore tri-state; one-shot recovered error→status 1, repeat failure→`running=0`; next marker/no-marker assertions. |
| Shell-visible status reflects cleanly observed process results. | 정상 exit, signal, 126, 127 mapping과 wait failure를 구분해야 합니다. | `7c9646e7cd79` | `be2967a4b946`, `d611196b368e` | `status_from_wait`, exec errno mapping, any wait error suppresses last-stage status and returns 1. |

### Ledger 작성 시 확인한 것

- PID slot은 `ae988...`에서 도입됐지만 incomplete graph 소유권은 `be296...`에서 완성됐습니다.
- Pipe topology 결정은 `a71f...`, failure convergence는 `be296...`, observation evidence는 `d611...`/`b42e...`로 분리했습니다.
- Test seam commits는 소유권 policy를 만들지 않고 rare branch를 deterministic하게 관찰합니다.
- Success, pipe failure, partial fork, wait failure, restore failure 모두 acquired FDs/tables/PIDs의 explicit terminal state를 가집니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| later fork failure 뒤 이미 생성된 child가 pipe에서 block되어 wait가 끝나지 않음 | `a71f98de0d92`의 normal pipe graph | `be2967a4b946` — parent pipe close, recorded child termination, complete reaping | `915aa072298b` seam → `d611196b368e` deterministic pipe/fork/wait failures → `b42e57eb7755` lifecycle probe | PID publish, close→kill→wait order, second-fork fixture, direct `waitpid(-1,WNOHANG)` probe를 연결했습니다. |
| parent builtin redirection restore failure를 무시하면 persistent stdin/stdout이 손상됨 | `fd5c76c18c27`의 dup/open seam | `2ca9f4299c7f` — independent retry, status 1, unrecoverable stop | `13645f58d5e6` — save/apply/restore/open/repeated failure matrix | `restore_one` tri-state와 repeat-mode no-following-command assertion을 연결했습니다. |
| pipe creation failure 전 preallocated PID table이 return path에 남음 | execution table을 먼저 할당하는 preparation order | `6dff1ba86ba6` — partial pipe cleanup에 PID table free 추가 | Thread 내 별도 전용 test commit은 없으므로 fault-injection 또는 sanitizer 실행 결과와 allocation/free code를 직접 기록합니다. | Exact diff에서 `free(pids)` 추가, fork-before/after ordering으로 child count 0을 확인했습니다. Runtime sanitizer는 실행하지 않았습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| PID slot | parent executor | wait 또는 partial-failure termination/reap 완료 | record 시점과 valid count를 확인 | Fork success 직후 index slot에 publish하고 `spawned`가 valid range를 정의합니다. |
| pipe table | parent 준비 단계 | parent close 후 table free | 각 slot `-1` 초기화와 partial creation cleanup 확인 | All entries `-1`; pipe success마다 pair가 live; success/failure 모두 close helper 후 memory free입니다. |
| child inherited pipe ends | 각 child stage | dup2 뒤 모든 original end close | stage index별 필요한 end와 불필요한 end 기록 | previous read/current write만 stdio에 duplicate하고 table의 every original end를 close합니다. |
| saved stdin/stdout | parent-command executor | restore attempts 완료 후 close | 둘을 독립적으로 복원하는 code 확인 | Acquisition failure cleanup, per-target restore retry, final close가 outcome과 무관하게 수행됩니다. |
| external environment vector | child before exec | exec 성공 시 process image로 이전, 실패 시 child cleanup | serialization과 failure status 기록 | Child local owner; successful exec replaces image, 실패 처리 frees/terminates with 126/127 or allocation status. |
| last-stage wait result | parent status calculation | 모든 required wait가 clean할 때만 사용 | wait error override branch 기록 | Full spawn and no wait error일 때만 last recorded PID result를 사용합니다. |

## 9. Thread 최종 상태

N-stage graph는 `N-1` pipe pairs를 사용합니다. Single stateful builtin만 parent에서 실행되고, multi-stage 처리 단계의 모든 commands/builtins는 child에서 실행됩니다.

| Terminal path | Child/FD/table 결과 | Shell-visible result |
| --- | --- | ---: |
| normal complete 처리 단계 | parent/children close unused ends; every PID exact wait; tables free | last clean stage status |
| pipe creation failure | opened prefix close; both tables free; no child | 1 |
| mid-fork failure | parent ends close; recorded children kill/reap; tables free | 1 |
| wait failure | remaining waits attempted; tables free | 1, last status ignored |
| parent restore transient error | stdio restored; saved FDs close | 1, continue |
| parent restore permanent error | saved FDs close; stdio untrusted | 1, `running=0` |

### 최종 상태 기록

- 최종적으로 유지되는 data/자원 소유권: executor local이 tables와 parent pipe FDs를, parent가 recorded PIDs를, each child가 inherited descriptors를 dup/close 시점까지 소유합니다. Parent command helper는 saved stdio를 소유합니다.
- 최종적으로 보장되는 execution 또는 recovery rule: record/acquire한 자원은 graph 완성 여부와 무관하게 close·terminate·wait·free로 회수되며, parent stdio를 신뢰할 수 없으면 다음 command를 실행하지 않습니다.
- Thread가 해결한 가장 어려운 failure: incomplete 처리 단계에서 이미 실행 중이거나 block된 child를 남기지 않고 deterministic terminal 소유권으로 수렴시키는 문제입니다.
- Thread 밖에 남아 있는 보장 범위: arbitrary descendants, all possible close errors/kernel failures, infinite workload는 test 범위 밖입니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
[command_count 확인]
  ↓ shell_calloc PID table + pipe table; slots -1
[create N-1 pipes]
  ↓ fork in command order; successful PID record
[child i: dup previous read / next write → close all originals
          → apply explicit redirections → builtin or exec]
[parent: after spawn loop close all parent pipe ends]
  ↓ wait exact recorded PIDs
[last clean stage status 또는 any observation error면 1]

[partial fork failure]
  ↓ close parent FDs → SIGKILL recorded children → reap every PID
  ↓ free both tables → return 1

[parent builtin]
  ↓ save stdin/stdout → apply redirections → run/flush → restore independently
  ├─ reliable restore: continue
  └─ permanent restore failure: running=0
```

### 코드 기반 최종 설명

- 핵심 entry function: 처리 단계 dispatcher와 `run_forked_commands`; parent path의 parent-command executor입니다.
- 주요 caller → callee chain: sequence executor → one-처리 단계 dispatcher → table/pipe setup → fork loop → child pipe setup/redirection/builtin-or-exec; parent close → `wait_for_child`; failure branch → `terminate_children`.
- state mutation 순서: memory allocations → live pipe slots → PID records → child stdio dup → parent closes → wait observations → result selection; parent builtin에서는 saved descriptors → temporary redirection → restore result → status/running.
- 소유권 transfer 순서: pipe() returns to parent table and is inherited by child; child duplicates then closes originals; parent closes its copies; PID record stays parent-owned through wait; saved stdio stays local through restoration.
- failure convergence path: pre-fork pipe failure closes/free locals; mid-fork failure close/kill/reap; wait failure forces 1; permanent stdio restore failure stops shell.
- regression evidence: process failure matrix, FD restoration matrix, low-FD lifecycle stress, direct child probe가 source에 존재합니다. 실제 commands는 실행하지 않았습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] S commit은 problem, prior state, failure possibility, decision, core code, 소유권/lifecycle, 후속 작업을 설명했습니다.
- [x] A commit은 subsystem boundary 또는 실패 처리와 실제 핵심 code를 설명했습니다.
- [x] B commit은 Thread 내 구현 역할과 state/소유권 변화를 설명했습니다.
- [x] Fix commit은 기존 가정 → failure → root cause → 수정 항상 유지해야 하는 조건 → code → regression 순으로 연결했습니다.
- [x] Test commit은 항상 유지해야 하는 조건, failure, technique, production path, prove/not prove를 구분했습니다.
- [x] 항상 유지해야 하는 조건 ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 resource와 partial object의 terminal owner를 설명했습니다.
- [x] 이 Thread의 설계 → 구현 → 실패 → 수정 → 검증 흐름을 commit history 순서로 재구성했습니다.
===== END FILE: 03-처리 단계-process-and-descriptor-소유권.md =====

===== BEGIN FILE: 04-transactional-allocation-failure.md =====
# From fatal allocation to transactional command failure

> 한국어 주제: **Fatal allocation에서 transactional command failure로**
>
> Project: `small-shell`
> Branch: `c/minishell`
> Development Thread order: 4/5

## 1. Thread 목표

low-level allocation failure가 임의의 process exit로 끝나던 초기 모델을, 각 subsystem이 complete result만 publish하고 partial state를 정리한 뒤 command status로 전파하는 모델로 전환한 과정을 복원합니다.

**Source-defined significance**

> The central change is not the wrapper itself but the failure model: construction must either publish a complete owned result or leave no partial state. The later executor and heredoc work shows why a single `NULL` return is insufficient unless side effects and input position are also controlled. The sweep then verifies that this policy holds across the actual command-processing graph.

**학습 관점**

핵심은 wrapper 도입이 아니라 failure model의 변경입니다. `NULL`을 반환하는 것만으로는 부족하며, side effect, existing state, OS resource acquisition, stdin position까지 transaction boundary에 포함해야 합니다.

### SHA 고정 원칙

- 각 commit은 반드시 표시된 exact SHA 또는 그 parent와 비교합니다.
- 먼저 `git show --name-status <SHA>`로 변경 파일을 식별한 뒤, 필요한 path만 `git diff <SHA>^ <SHA> -- <path>`로 봅니다.
- 실제 구현은 `git show <SHA>:<path>` 또는 detached worktree에서 확인합니다.
- final HEAD의 type, function, test를 과거 commit 설명에 소급하지 않습니다.
- later commit의 field나 fix가 아직 존재하지 않는 SHA에서는 그 부재 자체를 기록합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- allocation wrapper가 정책을 결정합니까, 아니면 caller가 recoverable/fatal 여부를 결정합니까?
- environment replacement에서 새 value allocation이 성공하기 전에 기존 value를 해제하면 어떤 항상 유지해야 하는 조건이 깨집니까?
- lexer/parser append는 어느 시점에 partial object를 public structure에 연결합니까?
- 처리 단계 table allocation이 OS pipe 생성보다 앞서야 하는 이유는 무엇입니까?
- heredoc allocation failure는 object cleanup 외에 어떤 input side effect를 복구해야 합니까?
- phase·command·call-position scoped injection이 어떤 two coherent outcomes만 허용합니까?
- persistent allocator failure에서 계속 loop를 돌면 왜 residual input 실행 위험이 생깁니까?

## 3. 완료 기준

- [x] fatal helper의 이전 call path와 nullable helper 이후 propagation path를 비교했습니다.
- [x] environment, lexer, parser, expansion 각각에서 `allocate → validate → publish → replace/free` 순서를 기록했습니다.
- [x] 실행 resource table이 side-effect-free preparation으로 바뀐 지점을 확인했습니다.
- [x] heredoc failure에서 memory transaction과 input-position recovery를 함께 설명했습니다.
- [x] allocation sweep의 phase, command number, one-shot/repeat mode, accepted outcomes를 구분했습니다.

> 실행 범위: exact SHA의 commit diff와 source/test scripts를 GitHub repository에서 검사했습니다. Branch checkout이 불가능해 allocation sweep과 sanitizer는 실행하지 않았습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `0b2e76386678` | `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합` | A | `ARCH`, `FAILURE`, `TEST` | Centralizes allocation and adds overflow-aware wrappers across execution paths. |
| 2 | `0bb6f9de0947` | `fix(memory): 구조화 단계의 할당 실패를 명령 오류로 전파` | S | `ARCH`, `FAILURE`, `RISK` | Replaces process-terminating helpers with nullable, transactional construction and command-level propagation. |
| 3 | `6d95776ede59` | `fix(memory): 실행 자원 할당 실패를 pipeline 오류로 전파` | A | `PROCESS`, `FAILURE`, `RISK` | Extends side-effect-free preparation ordering to executor resource tables. |
| 4 | `c30b39c0bcf8` | `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구` | A | `HEREDOC`, `FAILURE`, `RISK` | Protects heredoc stream boundaries when preparation fails after input consumption begins. |
| 5 | `476b082d55c7` | `test(memory): 범위별 할당 실패 순회 검증` | A | `TEST`, `FAILURE`, `RISK` | Sweeps allocation positions by phase and verifies cleanup, state atomicity, continuation, and persistent-failure termination. |

## 5. Commit별 학습 기록

### 5.1 `0b2e76386678` — `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합`

#### 확정 정보
- SHA: `0b2e76386678`
- Subject: `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합`
- Importance: **A**
- Tags: `ARCH`, `FAILURE`, `TEST`
- Source-defined role: Centralizes allocation and adds overflow-aware wrappers across execution paths.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
처리 단계 setup, heredoc buffering, input growth, shared string utilities의 allocation을 runtime layer로 모으고, `shell_calloc`에 multiplication-overflow check를 추가합니다. Caller의 기존 fatal/recoverable policy는 아직 유지됩니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: raw `malloc`/`calloc`/`realloc`이 subsystem 곳곳에 있어 overflow 동작과 deterministic injection seam을 한 곳에서 관리할 수 없었습니다.
- 새 boundary가 제공하는 contract: `shell_malloc`, `shell_calloc`, `shell_realloc`이 allocation request를 통과시키고, `shell_calloc`은 `count * size` overflow를 `ENOMEM` failure로 바꿉니다.
- production semantics가 유지된다는 코드 근거: wrapper는 overflow case 외에는 libc allocator를 그대로 호출하고 caller return handling은 이 commit에서 그대로입니다.
- 소유권 또는 call-site responsibility 변화: returned pointer의 owner와 cleanup responsibility는 각 caller에 남습니다. Wrapper는 policy owner가 아니라 common call boundary입니다.
- 후속 fix/test가 이 seam을 사용하는 방식: `0bb6...`가 nullable propagation을 project-wide로 만들고 `476b...`가 scope/call-position failure injection을 wrapper에 추가합니다.

#### `0b2e76386678`에서 확인할 실제 코드
- `src/runtime.h/.c`의 three allocation wrappers를 확인했습니다.
- `shell_calloc`은 nonzero size에서 `count > SIZE_MAX / size`를 검사하고 `errno = ENOMEM`, NULL을 반환합니다.
- 처리 단계 table/PID allocation, heredoc local buffers/entries, input growth, shared duplicate/join utilities가 wrappers로 이동합니다.
- Caller별로 NULL을 이미 처리하는 곳과 fatal helper에 의존하는 곳이 섞여 있습니다.
- Production 동작은 centralization과 overflow guard 외에 변하지 않습니다.

#### 학습자가 남길 코드 증거
- wrapper API map:

| API | Contract |
| --- | --- |
| `shell_malloc(size)` | libc `malloc` delegation |
| `shell_calloc(count,size)` | multiplication overflow면 `ENOMEM`/NULL, 아니면 `calloc` |
| `shell_realloc(ptr,size)` | libc `realloc` delegation, old pointer 소유권은 success 전 caller에 유지 |

- overflow check expression: `size != 0 && count > SIZE_MAX / size`.
- routed subsystem 목록: executor tables, heredoc buffers/repository, input buffer, string utilities.
- caller별 failure policy: some callers return failure and cleanup; old `sh_xcalloc`/duplicate helpers may still diagnose and exit.
- 아직 남은 fatal helper와 partial construction 위험: deep lexer/parser/env utility에서 allocation failure가 process termination으로 끝나고 existing state publication ordering이 통일되지 않았습니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, `src/exec.c`, `src/heredoc.c`, `src/input.c`, `src/utils.c`.
- 핵심 caller → callee: subsystem allocator call → shell wrapper → libc allocation; cleanup은 caller-specific입니다.
- parent SHA와 비교한 최소 before/after snippet:

```c
if (size != 0 && count > SIZE_MAX / size) {
    errno = ENOMEM;
    return NULL;
}
return calloc(count, size);
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Wrapper definitions와 routed 호출 지점을 exact diff로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: allocation 동작을 한 boundary에서 관찰·주입하고 overflow-aware zero allocation을 공통 사용하게 합니다.
- 아직 보장하지 않는 것: global failure model은 아직 바뀌지 않아 low-level fatal exit와 partial publication 문제가 남습니다.

#### Thread 내 다음 연결
`0bb6f9de0947`가 wrapper 위에서 project-wide transactional failure policy를 구현합니다.

### 5.2 `0bb6f9de0947` — `fix(memory): 구조화 단계의 할당 실패를 명령 오류로 전파`

#### 확정 정보
- SHA: `0bb6f9de0947`
- Subject: `fix(memory): 구조화 단계의 할당 실패를 명령 오류로 전파`
- Importance: **S**
- Tags: `ARCH`, `FAILURE`, `RISK`
- Source-defined role: Replaces process-terminating helpers with nullable, transactional construction and command-level propagation.
- 학습 깊이: Architecture / 항상 유지해야 하는 조건 핵심. 변경 전 가정, failure 가능성, 결정, core code, 소유권/lifecycle, 후속 작업을 추적합니다.

#### Source에서 확정된 변화
Fatal allocation helpers를 nullable operation으로 바꾸고, environment, lexer, parser, expansion, 공개 API, command loop까지 complete result만 publish하고 partial construction을 해제하도록 failure propagation을 재설계합니다.

#### Source가 확정한 핵심 판단
- **문제**: Fatal allocation helpers could terminate the shell from deep inside tokenization, parsing, environment mutation, or expansion, bypassing 소유권 cleanup and potentially exposing partial state.
- **결정**: Make allocation helpers nullable and require each construction layer to publish only complete results, preserve existing state until replacements succeed, release partial prefixes, and propagate allocation failure through command or startup boundaries.
- **중요한 이유**: This is a project-wide change from exception-like process termination to explicit transactional failure. It affects almost every owned representation and determines whether a running shell can diagnose one failed command and continue safely.
- **확정된 변경 범위**: Utilities gained size checks and nullable returns; environment creation, replacement, import, and serialization became transactional; lexer and parser publishing became failure-aware; expansion and public APIs propagated allocation errors; and the loop distinguished syntax status from command-level memory failure.
- **프로젝트 이해에서의 위치**: It is the central failure-architecture commit. It unifies the 소유권 lessons from parsing, environment state, execution, and heredoc into one 항상 유지해야 하는 조건: no incomplete object escapes and no arbitrary helper owns process termination.

#### Fix 재구성 기록
- 기존 가정: allocation failure is unrecoverable anywhere, so `sh_xcalloc`/fatal duplicate helper may diagnose and call `exit` deep in the call graph.
- 실제 failure 또는 위험을 드러내는 입력·상태: env replacement가 old value를 먼저 free한 뒤 new copy 실패, parser가 partial prefix를 publish한 뒤 deep exit, token creation failure가 already built list cleanup을 우회하는 상태입니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: low-level helper가 process 수명 policy를 소유하고 constructors의 publish/rollback protocol이 명시되지 않았습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: allocation은 nullable; each layer keeps new work local until complete, publishes after all dependencies succeed, preserves old state until replacement ready, and propagates failure to command/startup boundary.
- 변경 전 코드 증거: `sh_xcalloc`/old helpers가 NULL에서 diagnostic + `exit`; env replacement sequence may destroy old value before replacement is guaranteed.
- 변경 후 코드 증거: `sh_calloc`/string utilities return NULL; env/token/parser/expand functions check and unwind; line loop distinguishes allocation failure status 1 from syntax 258.
- 연결되는 regression test와 그 한계: `476b082d55c7`가 configured scopes/call positions를 sweep합니다. Unscoped startup/allocator internals 전체를 mathematically prove하지 않습니다.

#### `0bb6f9de0947`에서 확인할 실제 코드
- Parent SHA의 `sh_xcalloc` fatal path와 new nullable utility API를 비교했습니다.
- `src/utils.c`는 size arithmetic failure와 NULL return을 caller-visible하게 만듭니다.
- `src/env.c`의 node construction은 structure/key/value success 뒤 list에 link합니다.
- `env_set` replacement는 new value duplicate가 성공한 뒤 old value를 free/publish합니다.
- `env_from_environ`은 import prefix failure 시 whole partial list를 free합니다.
- `env_to_environ`은 vector/each `KEY=VALUE` failure 시 created prefix를 free합니다.
- Token node가 text 소유권을 받을 수 없으면 text/list를 cleanup하고 publish하지 않습니다.
- Parser append는 new argv/string or redirection node/target success 뒤 fields/list를 mutate합니다.
- Shared parse failure는 current command/current 처리 단계/completed prefix를 정리합니다.
- Expansion/dequote/public parser APIs propagate allocation failure separately from syntax diagnostic.
- `shell_process_line`은 syntax status 258와 allocation status 1을 구분합니다.
- Startup environment import failure는 usable shell state가 없으므로 diagnosed process return 경계입니다.

#### 학습자가 남길 코드 증거
- fatal model의 이전 call graph: lexer/parser/env/expand → fatal utility → diagnostic → `exit`, caller cleanup bypass.
- subsystem별 local partial object와 publish point:

| Subsystem | Local partial | Publish point | Failure cleanup |
| --- | --- | --- | --- |
| environment node | node/key/value | all fields complete then list link | free fields/node |
| environment replace | new value copy | copy success then swap/free old | old value unchanged |
| lexer | word text + token node | both complete then list append | free local + token prefix |
| parser argv/redir | new vector/string or node/target | complete aggregate then field/list update | free new partial, hierarchy cleanup |
| expansion | new string/buffer | complete result then replace field | preserve encoded old field |

- environment replace-before-free transaction: allocate copy → on NULL return with old value intact → on success assign new pointer and free old.
- parser single failure convergence: error label frees local command, local 처리 단계, already completed sequence prefix.
- 공개 API return/diagnostic 소유권: NULL/오류 코드 communicates allocation; optional error string itself is checked/owned by caller.
- loop의 status 258 vs 1: syntax/lexical grammar error gets 258; `ENOMEM`/allocation failure gets 1.
- startup fatal boundary vs running-shell recoverable boundary: initial env import failure returns from `main`; per-command token/parse/expand failure diagnoses and leaves loop usable when other state/input is trustworthy.
- 확인한 변경 파일: `include/shell.h`, `src/utils.c`, `src/env.c`, `src/token.c`, `src/parser.c`, `src/expand.c`, `src/exec.c`, `src/input.c`, `src/main.c`.
- 핵심 caller → callee: shell loop → line processor → token/parser/expander; every nullable result bubbles to line boundary; startup env import bubbles to `main`.
- parent SHA와 비교한 최소 before/after snippet:

```text
before: allocate failure → deep helper exit
 after: allocate local → NULL? rollback + return → command boundary status 1
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact large diff에서 subsystem publish/rollback and status branches를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: allocation failure는 이전 valid state를 유지하거나 partial result 전체를 정리하며, arbitrary utility가 running shell을 종료하지 않습니다.
- 아직 보장하지 않는 것: executor bookkeeping tables와 stdin을 이미 소비한 heredoc side effect는 별도 commits에서 같은 policy를 확장합니다.

#### Thread 내 다음 연결
`6d95776ede59`가 OS resource acquisition 전 execution tables를 준비하고, `c30b39c0bcf8`가 input-position transaction을 보강합니다.

### 5.3 `6d95776ede59` — `fix(memory): 실행 자원 할당 실패를 pipeline 오류로 전파`

#### 확정 정보
- SHA: `6d95776ede59`
- Subject: `fix(memory): 실행 자원 할당 실패를 pipeline 오류로 전파`
- Importance: **A**
- Tags: `PROCESS`, `FAILURE`, `RISK`
- Source-defined role: Extends side-effect-free preparation ordering to executor resource tables.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
처리 단계 pipe-end table과 PID table을 모두 OS pipe 생성 전에 overflow-aware `shell_calloc`으로 확보하여, allocation failure를 child/FD 없는 pure preparation error로 만듭니다.

#### Fix 재구성 기록
- 기존 가정: table allocation과 pipe creation을 interleave하거나 one table 뒤 OS resource를 acquire해도 failure cleanup으로 처리할 수 있다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: some pipe FDs가 live인 뒤 PID table allocation이 실패하면 memory failure가 descriptor cleanup/side effect rollback까지 요구합니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: side-effect-free memory preparation과 externally visible OS acquisition이 섞여 있었습니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: both bookkeeping tables를 먼저 allocate/initialize한 뒤에만 first `pipe` call을 합니다.
- 변경 전 코드 증거: resource acquisition 전 all table success가 보장되지 않았습니다.
- 변경 후 코드 증거: PID allocation → pipe table allocation → `-1` initialization → pipe creation 순서입니다.
- 연결되는 회귀 테스트와 그 한계: `476b...`의 execute 범위가 table allocation positions를 실패시킵니다. Pipe syscall failure cleanup은 process/FD Thread tests가 다룹니다.

#### `6d95776ede59`에서 확인할 실제 코드
- 처리 단계 execution entry의 two allocation order를 확인했습니다.
- Both success 전 `shell_pipe` call이 없습니다.
- Each allocation failure는 status 1 and local memory cleanup only입니다.
- Pipe slots를 explicit `-1`로 채워 later partial pipe cleanup이 unopened slots를 skip합니다.
- Size multiplication은 `shell_calloc` overflow guard를 통과합니다.

#### 학습자가 남길 코드 증거
- preparation acquisition order: PID table → pipe table → initialize descriptors → OS pipes → children.
- allocation failure 시 live OS resources: none.
- status/정리 과정: first allocation failure는 no local free; second failure는 PID table free; both return 1.
- `-1` initialization 필요성: zero is stdin and cannot represent unopened slot; close helper must skip only negative descriptors.
- global transactional policy의 executor 적용점: all fallible memory preparation before external side effects.
- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: 처리 단계 dispatcher → `shell_calloc` twice → init loop → pipe creation.
- parent SHA와 비교한 최소 before/after snippet:

```text
allocate PID table
allocate pipe table
initialize every fd slot to -1
only then call pipe()
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact acquisition order and failure labels를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: execution bookkeeping allocation failure는 side effect 없이 status 1로 반환되고 child나 pipe descriptor를 남기지 않습니다.
- 아직 보장하지 않는 것: pipe creation 이후의 syscall failure와 later cleanup은 process/FD Thread의 별도 fixes가 담당합니다.

#### Thread 내 다음 연결
`c30b39c0bcf8`는 allocation failure가 이미 heredoc input을 소비한 뒤 발생하는 더 어려운 transaction boundary를 다룹니다.

### 5.4 `c30b39c0bcf8` — `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`

#### 확정 정보
- SHA: `c30b39c0bcf8`
- Subject: `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`
- Importance: **A**
- Tags: `HEREDOC`, `FAILURE`, `RISK`
- Source-defined role: Protects heredoc stream boundaries when preparation fails after input consumption begins.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Delimiter dequote, body buffer init, body expansion 등이 실패한 뒤 즉시 return하지 않고, current heredoc remainder와 later pending heredoc을 모두 delimiter까지 소비한 후 failure를 반환합니다.

#### Source가 확정한 핵심 판단
- **문제**: A heredoc preparation failure could return while body lines and later delimiters remained in stdin, causing data intended for the failed command to be parsed as future shell commands.
- **결정**: Mark preparation as failed, consume the remainder of the current and later pending heredocs without constructing bodies, and compare encoded delimiters directly when normal dequoting allocation is unavailable.
- **중요한 이유**: For a streaming command interpreter, preserving the next command boundary is as important as freeing memory. Returning an error without restoring input position would convert a local allocation failure into unintended command execution.
- **확정된 변경 범위**: The collector gained discard-through-delimiter behavior, marker-aware allocation-free delimiter matching, continued traversal of pending heredocs, and additional capacity-overflow protection.
- **프로젝트 이해에서의 위치**: This exceptional A-level commit reveals the depth of the failure model: recovery must account not only for objects and descriptors but also for semantic position in the input stream.

#### Fix 재구성 기록
- 기존 가정: partial heap objects를 free하고 NULL/failure를 return하면 transaction이 rollback됐다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: current body line과 later heredoc delimiters가 stdin에 남아 top-level shell commands로 실행됩니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: streaming input cursor는 heap object가 아니지만 command transaction의 semantic state입니다.
- 수정된 항상 유지해야 하는 조건 또는 decision: first failure 후 construction을 멈추되 traversal/input consumption은 all pending delimiters까지 계속합니다.
- 변경 전 코드 증거: dequote/init/append allocation failure에서 immediate return.
- 변경 후 코드 증거: failed flag, `discard_heredoc`, allocation-free `delimiter_matches`, no body publish in recovery mode.
- 연결되는 회귀 테스트와 그 한계: `476b...` heredoc scope sweep가 allocation source로 이를 검증하고 I/O repeat failure는 Thread 2의 `7e2f...`가 다룹니다.

#### `c30b39c0bcf8`에서 확인할 실제 코드
- Outer traversal은 failure 후에도 later pending redirections를 방문합니다.
- Current 실패 처리도 own delimiter까지 discard합니다.
- Encoded delimiter matching은 marker를 건너뛰어 no-allocation recovery가 가능합니다.
- Buffer doubling overflow check `SIZE_MAX / 2`가 추가됩니다.
- Recovery path는 body entry를 add하지 않습니다.

#### 학습자가 남길 코드 증거
- 기존 가정: memory state only transaction.
- 실제 위험: `echo unintended` body line이 failed command 뒤 shell command로 이동합니다.
- root cause: failure return과 stream position 불일치.
- failed mode의 traversal: mark failed → discard current → continue nested traversal → discard every remaining heredoc.
- allocation-free delimiter matching: encoded target marker pairs를 direct compare해 dequote allocation이 실패해도 delimiter를 찾습니다.
- 복구 완료 시 반환 status와 next input position: status 1; cursor는 all pending delimiters 뒤 next command boundary입니다.
- 확인한 변경 파일: `src/heredoc.c`.
- 핵심 caller → callee: `exec_prepare_heredocs` → `read_heredoc`/`discard_heredoc` → `delimiter_matches`.
- parent SHA와 비교한 최소 before/after snippet: immediate memory-error return이 discard traversal + delayed failure return으로 변경됩니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact source에서 stream-position rollback을 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: heredoc preparation이 실패해도 body data와 pending delimiters가 future shell commands로 이동하지 않습니다.
- 아직 보장하지 않는 것: recovery read 자체가 계속 실패하면 boundary를 보장할 수 없으며 그 경우의 forced-stop policy는 `7e2fdea3affd`가 검증합니다.

#### Thread 내 다음 연결
Allocation Thread에서는 `476b082d55c7` sweep으로 재검증됩니다.

### 5.5 `476b082d55c7` — `test(memory): 범위별 할당 실패 순회 검증`

#### 확정 정보
- SHA: `476b082d55c7`
- Subject: `test(memory): 범위별 할당 실패 순회 검증`
- Importance: **A**
- Tags: `TEST`, `FAILURE`, `RISK`
- Source-defined role: Sweeps allocation positions by phase and verifies cleanup, state atomicity, continuation, and persistent-failure termination.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Allocation wrapper에 phase와 command number 범위를 추가하고 tokenization, parsing, heredoc input/body, expansion, execution의 successive call positions를 sweep하여 clean failure 또는 untouched normal completion만 허용합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: any injected allocation point must yield either fully normal command result or coherent status-1 rollback without partial state/side effect.
- 재현하는 failure 또는 boundary: token, parser, expand, execute, heredoc body/input and persistent command-input allocation failures입니다.
- 사용한 test technique: phase + command ordinal + Nth allocation + one-shot/repeat selector를 runtime wrapper에 설정하고 N을 increasing sweep하는 deterministic fault injection입니다.
- 실제 통과하는 production code path: shell command boundary, each phase scope marker, environment mutation, external dispatch, heredoc recovery, input loop stop.
- 이 테스트가 증명하는 것: configured positions에서 only failure output or normal output appears; parent state and external side effects are atomic; heredoc boundary recovers; persistent failure stops residual input execution.
- 이 테스트가 증명하지 않는 것: sweep maxima 밖, unmarked startup allocation, allocator internals, all platform interactions를 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: systematic bounded failure-position sweep입니다.
- 후속 변경에서 막는 회귀: publish-before-success, partial list leak, env mutation on failed setup, external launch before complete preparation, residual heredoc/input execution입니다.

#### `476b082d55c7`에서 확인할 실제 코드
- `src/runtime.c`의 allocation state는 command number, scope string, call index, target index, repeat mode를 가집니다.
- `shell_runtime_begin_command`가 command ordinal을 갱신하고 `shell_runtime_set_alloc_scope`가 token/parser/expand/execute/heredoc/input scopes를 설정합니다.
- One-shot failure 후 state가 disarm되고 repeat mode는 target 이후 계속 fail합니다.
- `tests/allocation.sh`의 sweep은 call index를 증가시키며 exact two outcome patterns만 허용하고 최소 one failure/one success를 요구합니다.
- Failed parent builtin preparation이 environment를 mutate하지 않는 case가 있습니다.
- Failed external preparation이 external program을 실행하지 않는 case가 있습니다.
- Heredoc allocation failure case는 delimiters를 소비한 뒤 next marker를 실행합니다.
- Persistent input/token failure cases는 process status 1 and no residual marker입니다.

#### 학습자가 남길 코드 증거
- phase/command/call-position model: current command ordinal + current scope + scope-local allocation count가 failure key입니다.
- one-shot와 repeat mode: one-shot은 selected allocation 하나만 NULL; repeat은 selected point부터 matching later allocations도 NULL입니다.
- 허용되는 두 outcome: complete expected normal stdout/status 또는 exact diagnosed status-1 failure/rollback; mixed partial output/state는 reject합니다.
- state atomicity case: parent `export`/environment modification preparation failure에서 old environment observation remains.
- external side-effect suppression case: command/envp preparation failure면 marker external program이 실행되지 않습니다.
- heredoc boundary recovery case: current/pending body allocations failure 뒤 body lines suppressed, next top-level command only executes.
- persistent failure termination case: input/token allocation이 계속 실패하면 loop를 돌며 residual commands를 해석하지 않고 process exits 1.
- test가 포괄하지 않는 startup/path: initial environment import와 configured maxima 밖 positions입니다.
- 확인한 변경 파일: `src/runtime.c`, `src/runtime.h`, phase 호출 지점 across `src/input.c`, `src/token.c`, `src/parser.c`, `src/expand.c`, `src/exec.c`, `src/heredoc.c`, `tests/allocation.sh`.
- 핵심 caller → callee: test sweep → fault binary → command begin/scope markers → allocation wrappers → subsystem rollback → output/status assertion.
- parent SHA와 비교한 최소 before/after snippet:

```text
SMALL_SHELL_FAIL_ALLOC_SCOPE=<phase>
SMALL_SHELL_FAIL_ALLOC=<N>
[optional command number / repeat]
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Injection 상태 머신, sweep loops, accepted patterns를 source로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: recoverable-allocation architecture를 normal-path 추정이 아니라 systematic failure-position regression으로 검증합니다.
- 아직 보장하지 않는 것: 모든 allocator implementation이나 모든 startup allocation을 수학적으로 증명하지 않으며 설정된 production scopes에 한정됩니다.

#### Thread 내 다음 연결
Allocation Thread의 최종 검증입니다. 각 subsystem ledger의 publish point와 sweep case를 연결해 완료합니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 명시한 항상 유지해야 하는 조건과 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.

| 항상 유지해야 하는 조건 | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Allocation failure publishes either a complete result or no result. | 이전 valid state는 유지되거나, partial construction 전체가 해제되어야 합니다. | `0b2e76386678`의 common allocation boundary | `0bb6f9de0947`에서 project-wide transactional policy로 확정 | Env copy-before-free, token/node append-after-complete, parser single cleanup path, expansion replacement-after-success를 확인했습니다. |
| Low-level helpers do not terminate a running shell arbitrarily. | startup처럼 usable shell state가 없는 경계를 제외하면 allocation failure는 command-level status로 전파됩니다. | `0bb6f9de0947` | `476b082d55c7` failure sweep | Nullable utilities → subsystem returns → `shell_process_line` allocation branch status 1; startup import only returns from `main`. |
| Preparation allocation failure is side-effect free. | PID/pipe bookkeeping allocation이 실패하면 child나 OS pipe가 아직 존재하지 않아야 합니다. | `6d95776ede59` | `476b082d55c7` execution-phase injection | Both tables allocated/initialized before first `shell_pipe`; execute scope sweep checks no external/builtin side effect. |
| Input position is part of transactional recovery. | heredoc body를 일부 소비한 뒤의 allocation failure는 pending delimiter까지 정리해야 합니다. | `c30b39c0bcf8` | `476b082d55c7` heredoc sweep | Failed state + discard-only traversal + allocation-free delimiter matching; next marker only after all delimiters. |

### Ledger 작성 시 확인한 것

- Wrapper introduction and failure policy completion을 구분했습니다.
- `0bb6...`는 existing representations의 publish protocol을 바꾸고, `6d957...`/`c30b...`는 OS/input side effects까지 same 항상 유지해야 하는 조건을 확장합니다.
- Sweep evidence는 actual phase markers와 production call graph를 통과합니다.
- Success/failure 모두 local partial objects의 owner가 명확하며, continuation은 resource/input boundary가 trustworthy일 때만 허용됩니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| allocation helper가 deep call stack에서 `exit`하여 partial ownership cleanup과 shell continuation을 우회함 | `0b2e76386678`의 central wrapper seam | `0bb6f9de0947` — nullable utilities, transactional construction, caller-visible failure | `476b082d55c7` — phase/command/call-position sweep | Fatal helper removal, env/token/parser/expand publish points, status 1 branch와 scope sweep를 연결했습니다. |
| executor bookkeeping allocation이 OS resource acquisition 뒤 실패하면 cleanup state가 복잡해짐 | pipeline setup의 table allocation | `6d95776ede59` — both tables first, then pipe creation | `476b082d55c7` execution scope에서 side-effect-free failure를 확인 | Two `shell_calloc` success 전 `shell_pipe` 부재와 no external side effect assertion을 연결했습니다. |
| heredoc construction failure 뒤 unread input이 future commands로 이동 | nullable dequote/buffer/expansion operations | `c30b39c0bcf8` — discard through every pending delimiter | `476b082d55c7`의 heredoc failure and continuation cases | Failed flag/discard traversal, heredoc allocation scope, body suppression/next marker assertions를 연결했습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| new allocation result | local constructor | all dependent fields가 성공할 때까지 local | public list/field에 연결되는 exact line 기록 | Env/token/parser/expander 모두 dependent allocations success 뒤에만 link/replace합니다. |
| existing environment value/list | environment store | replacement allocation 성공 전까지 유지 | copy-before-free ordering 기록 | `env_set`는 new copy failure 시 old pointer/value를 그대로 둡니다. |
| partial token/parser prefix | current construction scope | failure 시 단일 정리 과정 | current object와 completed prefix 모두 포함되는지 확인 | Lexer prefix는 `free_tokens`, parser current + completed sequence는 hierarchy cleanup에 포함됩니다. |
| PID/pipe tables | executor preparation | OS resource acquisition 전 모두 확보 | 실패 시 local memory만 free되는지 확인 | Both tables success and `-1` init 전 pipe/fork 없음; allocation failure side-effect-free입니다. |
| heredoc input position | collector/recovery | pending delimiter consumption 완료 | memory cleanup과 stream recovery의 별도 책임 기록 | Heap rollback 후 discard traversal이 separate semantic cleanup을 담당합니다. |
| startup environment import | program startup boundary | 실패 시 diagnosed process return | running shell command failure와 구분 | No usable `t_shell` env state이므로 `main`이 failure를 반환합니다. |

## 9. Thread 최종 상태

Subsystem별 transaction boundary는 다음과 같습니다.

| Subsystem | Old state | Local partial | Publish point | Failure result |
| --- | --- | --- | --- | --- |
| environment replace | existing value | new copy | copy complete then swap | old value preserved |
| lexer | token prefix | word/node | both complete then append | local + prefix cleanup, status 1 |
| parser | completed prefix | current cmd/처리 단계/argv/redir | complete node/list append | one hierarchy cleanup, status 1 |
| expansion | encoded field | expanded result | full result then replace | old field preserved, no dispatch |
| executor | no OS side effect | PID/pipe tables | after both allocations, begin pipes | memory only cleanup, status 1 |
| heredoc | stdin at body cursor | partial body | entry after complete body | body cleanup + discard to boundary |

Syntax failure status 258와 allocation failure status 1은 line processor에서 별도 branch입니다. Startup import failure는 process return, running-shell one-shot command failure는 safe boundary에서 continuation, persistent/unrecoverable input/resource failure는 stop입니다.

### 최종 상태 기록

- 최종적으로 유지되는 data/자원 소유권: new allocations remain constructor-local until complete; old persistent state remains owner until successful replacement; executor/heredoc side effects have separate rollback rules.
- 최종적으로 보장되는 execution 또는 recovery rule: allocation failure는 complete result 또는 no result만 publish하고, status 1 continuation은 memory/resource/input boundary가 신뢰 가능한 경우에만 허용됩니다.
- Thread가 해결한 가장 어려운 failure: memory allocation failure가 이미 consumed stdin이라는 non-memory side effect를 가진 heredoc에서 unintended command execution으로 번지지 않도록 한 문제입니다.
- Thread 밖에 남아 있는 보장 범위: configured scopes/maxima 밖 allocation, allocator internals, all startup paths와 arbitrary combined faults는 증명하지 않습니다.

## 10. 최종 architecture 또는 실행 순서 정리

```text
[allocation request through shell_malloc/calloc/realloc]
  ↓ nullable result
[constructor-local partial state only]
  ↓ all dependent allocations/validation succeed?
    ├─ yes: publish complete object / then replace or free old state
    └─ no: free partial result / preserve old state / return explicit failure
  ↓ command boundary
[status 1 and continue only when heap + OS resources + input cursor are trustworthy]
  ↓ persistent or unrecoverable boundary failure
[stop shell rather than execute residual state/input]
```

### 코드 기반 최종 설명

- 핵심 entry function: runtime allocation wrappers, subsystem constructors, `shell_process_line`, `exec_prepare_heredocs`.
- 주요 caller → callee chain: input loop → command scope → tokenize → parse → heredoc prepare → expand → execute; each phase sets allocation scope and propagates NULL to line boundary.
- state mutation 순서: allocate local → validate all dependencies → publish/link/swap → release previous state; executor memory completes before pipe/fork; heredoc failure restores stream cursor before return.
- 소유권 transfer 순서: local allocation remains local until publish; old environment/field stays owned until replacement; successful body transfers to entry; failed body remains local and is freed.
- failure convergence path: subsystem rollback → explicit allocation error → status 1; unsafe persistent input/resource state → `running=0`/process stop.
- regression evidence: `tests/allocation.sh`의 scoped sweeps와 coherent-outcome assertions를 source로 확인했습니다. 실제 sweep은 실행하지 않았습니다.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] S commit은 problem, prior state, failure possibility, decision, core code, 소유권/lifecycle, 후속 작업을 설명했습니다.
- [x] A commit은 subsystem boundary 또는 실패 처리와 실제 핵심 code를 설명했습니다.
- [x] Fix commit은 기존 가정 → failure → root cause → 수정 항상 유지해야 하는 조건 → code → regression 순으로 연결했습니다.
- [x] Test commit은 항상 유지해야 하는 조건, failure, technique, production path, prove/not prove를 구분했습니다.
- [x] 항상 유지해야 하는 조건 ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 resource와 partial object의 terminal owner를 설명했습니다.
- [x] 이 Thread의 wrapper → transactional policy → side-effect ordering → stream recovery → sweep 흐름을 commit history 순서로 재구성했습니다.
===== END FILE: 04-transactional-allocation-failure.md =====

===== BEGIN FILE: 05-asymptotically-safe-text-construction.md =====
# Making text construction asymptotically safe and observable

> 한국어 주제: **점근적으로 안전하고 관찰 가능한 text construction**
>
> Project: `small-shell`
> Branch: `c/minishell`
> Development Thread order: 5/5

## 1. Thread 목표

문자 또는 치환마다 전체 문자열을 다시 복사하던 경로를 overflow-safe growable builder로 바꾸고, lexer와 expansion semantics를 유지하면서 end-to-end time bound와 sanitizer로 검증한 흐름을 복원합니다.

**Source-defined significance**

> The shared abstraction removes repeated whole-string copies while keeping overflow and partial-소유권 rules explicit. Only the builder introduction is A because it makes the structural decision; the migrations are applications of that choice. The performance and sanitizer paths provide observable evidence without inflating those supporting commits to architecture-level importance.

**학습 관점**

공통 builder는 성능만 개선한 것이 아니라 permanent NUL, overflow check, discard/take 소유권 protocol을 여러 text-processing stage에 통일합니다. Migration commit은 그 결정을 적용하고, performance와 sanitizer path는 결과를 관찰합니다.

### SHA 고정 원칙

- 각 commit은 반드시 표시된 exact SHA 또는 그 parent와 비교합니다.
- 먼저 `git show --name-status <SHA>`로 변경 파일을 식별한 뒤, 필요한 path만 `git diff <SHA>^ <SHA> -- <path>`로 봅니다.
- 실제 구현은 `git show <SHA>:<path>` 또는 detached worktree에서 확인합니다.
- final HEAD의 type, function, test를 과거 commit 설명에 소급하지 않습니다.
- later commit의 field나 fix가 아직 존재하지 않는 SHA에서는 그 부재 자체를 기록합니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- builder의 data, length, capacity 항상 유지해야 하는 조건과 permanent NUL terminator는 어느 함수에서 유지됩니까?
- `length + extra + 1`과 capacity doubling의 overflow를 각각 어떻게 검사합니까?
- `discard`와 `take`를 분리하면 failure와 success의 소유권이 어떻게 명확해집니까?
- lexer에서 single-quote marker와 character 두 byte를 append할 때 기존 representation이 보존됩니까?
- expansion에서 `$?`, `$NAME`, unset value, literal marker, empty result semantics가 migration 전후 동일합니까?
- 512 KiB end-to-end test가 실제로 증명하는 것과 수학적으로 증명하지 않는 것은 무엇입니까?
- sanitizer build graph를 ordinary build와 분리하는 이유는 무엇입니까?

## 3. 완료 기준

- [x] builder의 growth equation과 overflow branches를 실제 코드로 설명했습니다.
- [x] success `take`와 failure `discard` 뒤 builder state를 기록했습니다.
- [x] lexer와 expansion migration의 before/after loop를 비교해 repeated whole-string copy 제거를 확인했습니다.
- [x] semantic equivalence를 marker, quote flag, variable/status expansion 항목별로 검증했습니다.
- [x] performance test의 input size, deadline, status, stderr, output-length assertion을 기록했습니다.
- [x] ASan/UBSan artifact와 test seam이 모두 instrument되는 build graph를 확인했습니다.

> 실행 범위: exact SHA의 commit diff와 source/test/build graph를 GitHub repository에서 검사했습니다. Branch checkout이 불가능해 performance와 sanitizer targets는 실행하지 않았습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- |:---: | --- | --- |
| 1 | `b8347c06b6c7` | `refactor(buffer): 가변 문자열 빌더 모듈 추가` | A | `ARCH`, `PERF`, `REFACTOR` | Defines the shared builder's growth, overflow, discard, and ownership-transfer contracts. |
| 2 | `985f90b9cbc7` | `refactor(lexer): 단어 조립을 가변 버퍼로 전환` | B | `LEX_PARSE`, `PERF`, `REFACTOR` | Applies it to quote-aware lexer word construction. |
| 3 | `89e1a06f06c9` | `refactor(expand): 확장 결과를 가변 버퍼로 조립` | B | `EXPANSION`, `PERF`, `REFACTOR` | Applies it to expansion and dequoting. |
| 4 | `b36b9d324260` | `test(performance): 긴 입력 처리 시간 상한 검증` | B | `TEST`, `PERF` | Verifies a large word end to end under an explicit time bound. |
| 5 | `7d7dd7ad9d8a` | `build(test): ASan·UBSan 검증 경로 추가` | B | `TEST`, `PRACTICAL` | Runs the complete behavior, failure, lifecycle, and performance suites under sanitizers. |

## 5. Commit별 학습 기록

### 5.1 `b8347c06b6c7` — `refactor(buffer): 가변 문자열 빌더 모듈 추가`

#### 확정 정보
- SHA: `b8347c06b6c7`
- Subject: `refactor(buffer): 가변 문자열 빌더 모듈 추가`
- Importance: **A**
- Tags: `ARCH`, `PERF`, `REFACTOR`
- Source-defined role: Defines the shared builder's growth, overflow, discard, and 소유권-transfer contracts.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 실패 처리. 핵심 코드와 설계 판단을 확인합니다.

#### Source에서 확정된 변화
Initialization, append, discard, take를 가진 reusable string builder를 도입합니다. 항상 NUL을 유지하고 geometric growth와 overflow check를 수행하며 success/failure 소유권을 분리합니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: `sh_strjoin_free` 방식은 문자나 substitution 하나를 추가할 때마다 old prefix 전체를 새 allocation으로 복사하고 old allocation을 free했습니다.
- 새 boundary가 제공하는 contract: builder owns one growable allocation; `length`, `capacity`, terminal NUL을 유지하고 append reserve arithmetic을 검사하며 failure는 old builder state를 보존합니다.
- production semantics가 유지된다는 코드 근거: 이 commit은 module과 build source만 추가하고 lexer/expander 호출 지점은 아직 바꾸지 않습니다.
- 소유권 또는 call-site responsibility 변화: builder init 후 allocation owner는 builder; failure는 caller가 `discard`; success는 `take`에서 completed allocation을 caller로 이전합니다.
- 후속 fix/test가 이 seam을 사용하는 방식: lexer와 expander가 repeated join을 append/take로 교체하고 performance/sanitizer suites가 observable 동작을 검사합니다.

#### `b8347c06b6c7`에서 확인할 실제 코드
- `src/string_builder.h`의 `t_string_builder { data, length, capacity }`를 확인했습니다.
- Init은 fields를 zero/reset한 뒤 initial capacity 64 allocation을 시도하고 성공 시 `data[0] = '\0'`입니다.
- Append는 reserve 성공 뒤 bytes를 copy하고 `length`를 갱신한 다음 `data[length] = '\0'`를 씁니다.
- Required size 계산 전 `extra > SIZE_MAX - length - 1`을 검사합니다.
- Capacity는 geometric doubling하되 doubling overflow 위험이면 exact `needed` capacity로 fallback합니다.
- Realloc result는 temporary pointer에 받고 success 뒤에만 `data/capacity`를 갱신합니다.
- `discard`는 allocation free 후 all fields reset, `take`는 pointer를 return하고 builder를 reset합니다.

#### 학습자가 남길 코드 증거
- builder state 항상 유지해야 하는 조건:

```text
initialized/successful state:
  data != NULL
  length < capacity
  data[length] == '\0'

reset state after discard/take or failed init:
  data == NULL, length == 0, capacity == 0
```

- growth/overflow equation: first check `extra <= SIZE_MAX - length - 1`; then `needed = length + extra + 1`. While `new_capacity < needed`, double only when safe; otherwise set `new_capacity = needed`.
- discard 전/후 state: before owns partial allocation/content; after allocation freed and fields zero.
- take 전/후 owner: before builder owns `data`; return pointer becomes caller-owned and builder fields become reset, preventing 이중 해제.
- allocation 실패 처리: initial malloc leaves reset builder; realloc NULL leaves original allocation/content/metadata unchanged.
- old/new copy pattern 비교: old append copies prefix length 1+2+...+N; builder copies each input byte once plus occasional geometric reallocation copies, amortized linear.
- 확인한 변경 파일: `src/string_builder.c`, `src/string_builder.h`, `Makefile`.
- 핵심 caller → callee: later lexer/expander → builder init → reserve/append → take or discard → runtime allocation wrappers.
- parent SHA와 비교한 최소 before/after snippet:

```c
if (extra > SIZE_MAX - builder->length - 1)
    return -1;
needed = builder->length + extra + 1;
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact module implementation과 Makefile source inclusion을 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: text construction이 overflow-safe geometric buffer와 explicit discard/take 소유권 protocol을 공유할 수 있습니다.
- 아직 보장하지 않는 것: 이 commit은 abstraction만 도입하며 lexer/expansion 동작과 end-to-end performance는 아직 바꾸지 않습니다.

#### Thread 내 다음 연결
`985f90b9cbc7`와 `89e1a06f06c9`가 각각 lexer와 expansion을 migration합니다.

### 5.2 `985f90b9cbc7` — `refactor(lexer): 단어 조립을 가변 버퍼로 전환`

#### 확정 정보
- SHA: `985f90b9cbc7`
- Subject: `refactor(lexer): 단어 조립을 가변 버퍼로 전환`
- Importance: **B**
- Tags: `LEX_PARSE`, `PERF`, `REFACTOR`
- Source-defined role: Applies it to quote-aware lexer word construction.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
Lexer word construction을 shared builder로 전환하며 single-quoted character의 literal marker+byte encoding, unquoted/double-quoted semantics, token-level quoted flag를 유지합니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: source character마다 `sh_strjoin_free`가 current word 전체를 다시 allocate/copy했습니다.
- 새 boundary가 제공하는 contract: one builder per word scan; each fragment appends into reserved capacity; only completed word is taken into token 소유권.
- production semantics가 유지된다는 코드 근거: quote state branches, marker insertion, quoted flag set, empty quoted word and unclosed quote conditions are preserved in the diff.
- 소유권 또는 call-site responsibility 변화: local `char *word` 소유권 becomes local builder 소유권; token receives allocation only through final `string_builder_take`.
- 후속 fix/test가 이 seam을 사용하는 방식: performance fixture drives input → lexer builder and sanitizer suites exercise quote/error paths.

#### `985f90b9cbc7`에서 확인할 실제 코드
- Parent SHA's append helper repeatedly allocated/joined full prefix.
- New `read_word` initializes builder once, appends each byte/fragment, and takes at success.
- Single-quote branch appends marker then character, preserving two-byte encoding.
- Unquoted/double-quoted byte branch remains marker-free.
- Quote syntax participation still sets token-level `quoted` flag.
- If second append in marker+byte pair fails, builder may contain a local marker but 오류 처리 discards it, so partial representation is never published.
- Allocation failure or unclosed quote discards builder; successful token is sole owner of taken buffer.

#### 학습자가 남길 코드 증거
- old construction loop: each `append_char` produced new allocation containing entire old word plus one byte and freed old word.
- new builder call sequence: init → scan/append → on error discard → on success take → token node publish.
- marker encoding equivalence: single-quoted byte still emits `[LITERAL_MARK, byte]` in exact order.
- quoted flag equivalence: entering either single or double quote sets flag independent from encoded marker.
- failure discard와 success take: any scan/append/unclosed quote error calls discard; success only calls take once.
- token 소유권 after publish: taken pointer is node-owned and freed by `free_tokens`.
- 확인한 변경 파일: `src/token.c`, build/include references to builder.
- 핵심 caller → callee: `tokenize_line` → `read_word` → `string_builder_init/append_char/take` or discard.
- parent SHA와 비교한 최소 before/after snippet:

```text
before: word = sh_strjoin_free(word, one_or_two_bytes)
 after: string_builder_append_char(&builder, byte); ...; word = string_builder_take(&builder)
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Before/after diff에서 all quote branches and cleanup mapping을 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: lexer의 quote-aware representation을 유지하면서 long word construction을 geometric buffer에 옮깁니다.
- 아직 보장하지 않는 것: expansion/dequote의 repeated copying은 아직 남고, 성능 개선의 end-to-end evidence도 후속 test가 제공합니다.

#### Thread 내 다음 연결
`89e1a06f06c9`가 expansion과 dequote를 같은 builder contract로 옮깁니다.

### 5.3 `89e1a06f06c9` — `refactor(expand): 확장 결과를 가변 버퍼로 조립`

#### 확정 정보
- SHA: `89e1a06f06c9`
- Subject: `refactor(expand): 확장 결과를 가변 버퍼로 조립`
- Importance: **B**
- Tags: `EXPANSION`, `PERF`, `REFACTOR`
- Source-defined role: Applies it to expansion and dequoting.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
Expanded/dequoted output을 `sh_strjoin_free` 반복 대신 builder append로 조립하여 amortized linear construction으로 바꾸고, literal marker, `$?`, `$NAME`, unset value, empty result semantics를 유지합니다.

#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: ordinary byte와 each variable/status substitution을 append할 때 full accumulated output을 반복 복사했습니다.
- 새 boundary가 제공하는 contract: one builder owns partial expanded output; branch-specific bytes/strings append; complete result만 take합니다.
- production semantics가 유지된다는 코드 근거: old/new branch mapping for marker, ordinary char, `$?`, valid name, unset and unknown `$` remains equivalent.
- 소유권 또는 call-site responsibility 변화: partial expansion becomes builder-owned; caller field replacement happens only after successful take, preserving encoded source on failure.
- 후속 fix/test가 이 seam을 사용하는 방식: 512 KiB full-product deadline catches repeated copying/truncation and sanitizer suites run expansion/failure cases under instrumentation.

#### `89e1a06f06c9`에서 확인할 실제 코드
- Parent's per-character/per-substitution `sh_strjoin_free` loop is removed.
- Expansion/dequote entries init builder and final success takes allocation.
- Literal marker consumes marker+next and appends only literal byte.
- `$?` appends decimal current status; `$NAME` scans valid name and appends environment value; unset appends nothing.
- Unknown/incomplete dollar retains literal behavior.
- Environment name substring allocation can fail; 오류 처리 frees substring if needed and discards builder.
- Empty final output remains a valid owned empty string from initialized builder.
- Caller publishes replacement only after whole expand succeeds.

#### 학습자가 남길 코드 증거
- old quadratic copy source: output-growing join inside scan loop and substitution branches.
- new builder branch mapping:

| Encoded input branch | Builder action |
| --- | --- |
| literal marker + byte | append byte, advance 2 |
| ordinary byte | append char |
| `$?` | convert status, append text |
| `$NAME` | allocate/lookup name, append value if present |
| unset name | append zero bytes |

- `$?`/`$NAME`/unset/empty semantics: same as parent; initialized empty buffer ensures all-unset input returns `""`, not NULL.
- substring allocation failure cleanup: free temporary key if allocated, discard builder, return failure; old encoded field remains owned by caller.
- take 후 소유권 replacement: complete new string becomes parsed field; only then old encoded string is freed.
- semantic equivalence evidence: each parent branch has one corresponding builder append branch; no connector/quote timing change in this commit.
- 확인한 변경 파일: `src/expand.c`, builder headers/build dependencies.
- 핵심 caller → callee: selected 처리 단계 expansion → word/dequote helper → builder operations → take → field replacement.
- parent SHA와 비교한 최소 before/after snippet:

```text
before: result = sh_strjoin_free(result, fragment)
 after: string_builder_append_text(&builder, fragment)
```

- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact semantic branch mapping and replacement order를 검사했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: expansion과 dequoting이 complete-or-no-result 소유권을 유지하면서 repeated whole-string copies를 제거합니다.
- 아직 보장하지 않는 것: amortized 동작의 observable upper bound는 다음 end-to-end test가 제공하며 이 commit 자체가 시간 제한을 증명하지 않습니다.

#### Thread 내 다음 연결
`b36b9d324260`가 512 KiB word를 complete product path로 통과시켜 performance regression을 고정합니다.

### 5.4 `b36b9d324260` — `test(performance): 긴 입력 처리 시간 상한 검증`

#### 확정 정보
- SHA: `b36b9d324260`
- Subject: `test(performance): 긴 입력 처리 시간 상한 검증`
- Importance: **B**
- Tags: `TEST`, `PERF`
- Source-defined role: Verifies a large word end to end under an explicit time bound.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
512 KiB word를 input, tokenization, parsing, expansion, builtin output까지 통과시키고 five-second deadline, status 0, no diagnostics, exact payload length를 요구합니다.

#### Test commit 학습 기록
- 대상 production 항상 유지해야 하는 조건: large single word must complete without pathological repeated-copy delay, truncation, error, or unexpected diagnostic.
- 재현하는 failure 또는 boundary: 524,288-byte payload that makes old per-character whole-prefix copying prohibitively expensive.
- 사용한 test technique: generated end-to-end shell input + timeout runner + status/stderr/stdout-size assertions.
- 실제 통과하는 production code path: input allocation/read → lexer builder → parser argv → selected expansion builder → builtin `echo` output.
- 이 테스트가 증명하는 것: configured product/build/hardware에서 512 KiB payload completes under 5 seconds, exits 0, emits no stderr, and outputs all payload bytes plus newline.
- 이 테스트가 증명하지 않는 것: Big-O를 수학적으로 증명하거나 all hardware/compiler absolute latency, all token/expansion patterns을 보장하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: broad end-to-end performance regression with explicit upper bound입니다.
- 후속 변경에서 막는 회귀: repeated full-prefix join 재도입, truncation/overflow, large-output failure입니다.

#### `b36b9d324260`에서 확인할 실제 코드
- `tests/performance.sh`가 exactly 524,288 `x` bytes를 shell `echo` input에 넣고 command newline을 추가합니다.
- Product binary를 timeout runner through 5-second limit으로 실행합니다.
- 종료 상태 0과 empty stderr를 별도 검사합니다.
- `wc -c`/equivalent output-length assertion은 payload 524,288 + echo newline 1을 요구합니다.
- Exact length check catches truncation without comparing a huge expected string.

#### 학습자가 남길 코드 증거
- 대상 performance contract: 512 KiB one-word echo completes in <= test timeout with exact output.
- input size와 generated bytes: payload 524,288 bytes of `x`; shell command framing/newline separate.
- 통과하는 production stages: input, tokenization, parse argv allocation, expansion/dequote, builtin output.
- deadline/status/stderr/output assertions: 5 seconds, status 0, stderr empty, stdout 524,289 bytes.
- regression으로 잡는 old failure mode: O(N²)-like cumulative prefix copying and any truncation/failure caused by long word handling.
- 증명하지 않는 것: theoretical complexity, environment-independent performance, variable-heavy worst cases.
- broad integration 또는 performance regression 판정: broad product performance regression.
- 확인한 변경 파일: `tests/performance.sh`, Makefile test suite inclusion.
- 핵심 caller → callee: script generator → timeout runner → `small-shell` → full input/token/parse/expand/builtin path.
- parent SHA와 비교한 최소 before/after snippet: no production change; large fixture and four observable assertions added.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact input size, timeout, expected status/stderr/length를 script로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 긴 입력에서 repeated whole-string copying이나 truncation이 재도입되면 observable test failure로 드러납니다.
- 아직 보장하지 않는 것: 다른 hardware/compiler의 절대 성능이나 이론적 Big-O를 직접 증명하지 않습니다.

#### Thread 내 다음 연결
`7d7dd7ad9d8a`가 동일 behavior/failure/lifecycle/performance suites를 sanitizer artifacts로 실행합니다.

### 5.5 `7d7dd7ad9d8a` — `build(test): ASan·UBSan 검증 경로 추가`

#### 확정 정보
- SHA: `7d7dd7ad9d8a`
- Subject: `build(test): ASan·UBSan 검증 경로 추가`
- Importance: **B**
- Tags: `TEST`, `PRACTICAL`
- Source-defined role: Runs the complete behavior, failure, lifecycle, and performance suites under sanitizers.
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/소유권 변화를 확인합니다.

#### Source에서 확정된 변화
Production binary, fault-injection binary, source-level parser test에 별도 ASan/UBSan build graph를 만들고 existing smoke, failure, allocation, lifecycle, parser, performance suites를 instrumented artifacts로 실행합니다.

#### Build / validation boundary 기록
- 생성되는 artifact와 source set: ordinary product-equivalent sanitizer binary, test-seam sanitizer binary, parser API sanitizer test; ASan and UBSan variants each use dedicated objects/binaries.
- ordinary build와 분리되는 이유: instrumentation compiler/link flags must apply consistently to every object; ordinary objects cannot be reused without leaving code uninstrumented or causing runtime/link mismatch.
- 실행되는 validation path: smoke, process/FD/I/O faults, allocation sweep, lifecycle, parser API, performance suites run against sanitizer artifacts.
- build change가 runtime semantics를 바꾸지 않는 근거: production source logic is unchanged; only separate compile/link flags, artifact paths, test environment propagation, and targets are added.

#### `7d7dd7ad9d8a`에서 확인할 실제 코드
- Makefile has separate ASan/UBSan object dirs and binary targets rather than reusing ordinary objects.
- Flags include `-O1 -g -fno-omit-frame-pointer` and `-fsanitize=address` or `-fsanitize=undefined` at compile/link.
- Fault binary retains `SMALL_SHELL_TESTING` under instrumentation.
- Parser API test builds production sources excluding `main.c` with sanitizer instrumentation.
- `test-asan` and `test-ubsan` invoke smoke, faults, allocation, lifecycle, parser, performance suites.
- Tests using `env -i` explicitly preserve sanitizer option variables.
- Container target uses `gcc:13-bookworm`, disables network, mounts source read-only, copies to writable temporary space, then builds/runs tests.

#### 학습자가 남길 코드 증거
- sanitizer build graph:

```text
ordinary objects ──> ordinary binaries
ASan objects     ──> ASan product / ASan fault / ASan parser test
UBSan objects    ──> UBSan product / UBSan fault / UBSan parser test
```

- instrumented artifact별 source set: product all production sources; fault same plus testing macro/seams; parser API production library-like sources without normal main plus `tests/parser_api.c`.
- 실행되는 suite 목록: `tests/smoke.sh`, `tests/faults.sh`, `tests/allocation.sh`, `tests/lifecycle.sh`, parser API executable, `tests/performance.sh`.
- `env -i` option preservation: ASAN/UBSAN option environment is reintroduced so isolated test invocations keep sanitizer behavior.
- container reproducibility boundary: pinned GCC 13 bookworm image, network none, read-only repository input, writable temp copy.
- sanitizer가 증명하는 것과 증명하지 않는 것: exercised paths contain no sanitizer-detected address/정의되지 않은 동작 under configured runtime; unexecuted paths/all bug classes/formal memory safety are not proven.
- 확인한 변경 파일: `Makefile`, `tests/container_sanitizers.sh`, environment setup in existing test scripts.
- 핵심 caller → callee: make target → dedicated objects/binaries → complete shell/test suites under sanitizer runtime.
- parent SHA와 비교한 최소 before/after snippet: ordinary graph remains and parallel sanitizer graphs/targets are added.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Make dependency graph, recipes, flags, suite list, container script를 source로 확인했습니다.

#### 보장 범위
- 이 commit이 보장하는 것: 동작과 fault seams가 sanitizer instrumentation 아래 동일하게 검증되고 incompatible object reuse를 피합니다.
- 아직 보장하지 않는 것: sanitizer가 모든 memory/수명 bug를 증명하지 않으며 configured compiler/runtime와 exercised paths에 한정됩니다.

#### Thread 내 다음 연결
Text-construction Thread의 마지막 validation layer입니다.

## 6. 항상 유지해야 하는 조건 ledger

Source가 명시한 항상 유지해야 하는 조건과 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.

| 항상 유지해야 하는 조건 | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Builder output is always NUL-terminated. | 초기화와 모든 append 뒤 `data[length]`가 NUL이어야 합니다. | `b8347c06b6c7` | `985f90b9cbc7`, `89e1a06f06c9`에서 실제 사용 | Init `data[0]='\0'`; append copies bytes, updates length, writes final NUL; take/discard reset metadata. |
| Growth arithmetic cannot wrap. | `length + extra + 1`과 geometric doubling 모두 `SIZE_MAX`를 넘기지 않아야 합니다. | `b8347c06b6c7` | runtime allocation failure injection과 sanitizer path | `extra > SIZE_MAX - length - 1` guard, safe doubling and exact-needed fallback, temporary realloc publish. |
| Partial output does not escape on failure. | 실패 시 builder를 discard하고, 성공 시에만 allocation을 take하여 caller에 이전합니다. | `b8347c06b6c7` | `985f90b9cbc7`, `89e1a06f06c9` | Lexer unclosed/allocation errors and expander substring/append errors discard; token/field gets pointer only after take. |
| Performance change preserves lexical and expansion semantics. | literal marker, quote flag, `$?`, environment name, unset value, empty result 동작은 유지되어야 합니다. | `985f90b9cbc7`, `89e1a06f06c9` | `b36b9d324260`, `7d7dd7ad9d8a` | Parent/new branch mapping preserves semantics; large full-product regression and complete sanitizer suites provide observable evidence. Runtime not executed here. |

### Ledger 작성 시 확인한 것

- Builder 항상 유지해야 하는 조건 is introduced before callers migrate.
- Migration commits apply existing semantics rather than redefining quote/expansion policy.
- Performance evidence and sanitizer evidence are observational, not formal complexity/memory proofs.
- 실패 처리 always leaves builder owner capable of one discard; 정상 처리 transfers exactly once through take.

## 7. Failure → Fix → Test 연결

| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| character/substitution마다 whole output을 재할당·복사하여 긴 입력에서 비용이 누적됨 | 기존 `sh_strjoin_free` 중심 construction | `b8347c06b6c7` builder 도입 → `985f90b9cbc7`, `89e1a06f06c9` migration | `b36b9d324260` 512 KiB end-to-end deadline | Parent/new loops, builder reserve math, 524,288-byte fixture와 5-second/exact-length assertions를 연결했습니다. |
| 성능 refactor가 marker encoding이나 ownership cleanup을 깨뜨릴 위험 | lexer/expansion의 기존 semantics | migration commit에서 동일 branch semantics와 discard/take protocol 유지 | `7d7dd7ad9d8a`의 complete suites under ASan/UBSan | Marker/quote/status/env/unset branch mapping과 instrumented suite graph를 연결했습니다. 실제 sanitizer는 실행하지 않았습니다. |

## 8. 소유권 / state / responsibility 변화

| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| builder allocation | builder object | discard 또는 take | init 이후 pointer/length/capacity state 기록 | Init success부터 builder-owned; realloc uses temp so failure preserves owner/state. |
| partial text | builder | failure 시 discard | caller에 노출되지 않는지 확인 | All lexer/expander error labels discard before return; no field/list append. |
| completed text | caller after take | token 또는 expanded field cleanup | take 뒤 builder reset state 확인 | Take returns pointer and zeroes builder; token/free_pipeline later owns cleanup. |
| old lexer/expansion string | caller field/local | new result publish 뒤 free | migration의 replacement ordering 기록 | New result complete/taken first; caller then swaps/frees old encoded string. |
| sanitizer artifacts | build graph | target별 clean/rebuild | ordinary object 재사용 금지 여부 확인 | Dedicated object dirs and link targets ensure all units carry matching instrumentation. |

## 9. Thread 최종 상태

Builder API 소유권 transition:

| Operation | Input state | Success state | Failure state |
| --- | --- | --- | --- |
| init | reset | owned empty NUL buffer cap 64 | reset/no allocation |
| append | valid builder | bytes appended, terminal NUL | previous builder/content unchanged |
| discard | any builder-owned allocation | reset, allocation freed | not applicable |
| take | valid builder | caller owns returned allocation; builder reset | not applicable |

Old complexity source was per-fragment whole-prefix join. New growth uses reserved geometric capacity, so each append writes only new bytes while reallocations happen logarithmically in capacity growth. Semantic equivalence is established by branch mapping; performance is observed by a 512 KiB full-product deadline. Sanitizer targets cover configured paths but do not prove all platforms or unexecuted code.

### 최종 상태 기록

- 최종적으로 유지되는 data/자원 소유권: one builder owns partial text; success transfers one completed allocation to token/parsed field, failure discards without publication.
- 최종적으로 보장되는 execution 또는 recovery rule: growth arithmetic is checked, terminal NUL is maintained, and lexer/expander preserve previous semantics while avoiding per-byte full-prefix copies.
- Thread가 해결한 가장 어려운 failure: overflow or second append/allocation failure in a partially encoded/expanded word must not publish malformed text or lose the original field.
- Thread 밖에 남아 있는 보장 범위: theoretical proof, all hardware latency, all sanitizer bug classes, unexercised paths are outside evidence.

## 10. 최종 architecture 또는 실행 순서 정리

```text
[builder init: reset fields → allocate cap 64 → data[0]=NUL]
  ↓ append request(extra)
[check extra <= SIZE_MAX - length - 1]
  ↓ needed = length + extra + 1
[geometric grow, or exact needed when doubling would overflow]
  ↓ append bytes → update length → data[length]=NUL
  ↓ final outcome
    ├─ success: take → caller owns completed allocation; builder reset
    └─ failure: discard → no partial output escapes
  ↓ lexer + expansion migrations preserve semantic branches
[512 KiB deadline + ASan/UBSan build/test graphs]
```

### 코드 기반 최종 설명

- 핵심 entry function: string builder init/reserve/append/discard/take; lexer `read_word`; expansion/dequote helpers.
- 주요 caller → callee chain: tokenization/selected expansion → builder APIs → runtime allocators → take/publish or discard.
- state mutation 순서: reserve check → optional realloc temporary → append bytes → length update → terminal NUL; final take resets builder before caller publication.
- 소유권 transfer 순서: builder local owns allocation throughout partial construction; take returns sole pointer; token/parsed hierarchy becomes owner.
- failure convergence path: reserve/append/sub-allocation/unclosed quote → discard; realloc failure preserves existing builder until discard; original field remains until new complete result.
- regression evidence: performance script and sanitizer build/suite graph were inspected. Their commands were not executed in this environment.

## 11. 학습 완료 자가 점검

- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] A commit은 subsystem boundary, growth/소유권 contract, 실패 처리와 핵심 code를 설명했습니다.
- [x] B commit은 Thread 내 migration/test/build 역할과 state/소유권 변화를 설명했습니다.
- [x] Test/build commit은 항상 유지해야 하는 조건, technique, production path, prove/not prove를 구분했습니다.
- [x] 항상 유지해야 하는 조건 ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 partial/completed text의 terminal owner를 설명했습니다.
- [x] 이 Thread의 abstraction → migration → performance observation → sanitizer validation 흐름을 commit history 순서로 재구성했습니다.
===== END FILE: 05-asymptotically-safe-text-construction.md =====

===== BEGIN FILE: README.md =====
# minishell Development Thread 학습 골격

## 목적

이 문서 세트는 `small-shell`의 `c/minishell` commit history를 다시 설명하는 완성형 해설서가 아닙니다. 학습자가 각 exact SHA의 diff와 당시 code를 직접 읽고, 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 복원하기 위한 기록 골격입니다.

Source of truth는 제공된 `commit-importance.md`와 `commit-bodies.md`뿐입니다. Importance, tags, Development Thread, commit 관계와 순서는 재평가하지 않습니다.

## 권장 학습 순서

1. [`01-parsed-representation-to-conditional-execution.md`](01-parsed-representation-to-conditional-execution.md)
2. [`02-heredoc-cross-stage-semantics.md`](02-heredoc-cross-stage-semantics.md)
3. [`03-pipeline-process-and-descriptor-ownership.md`](03-pipeline-process-and-descriptor-ownership.md)
4. [`04-transactional-allocation-failure.md`](04-transactional-allocation-failure.md)
5. [`05-asymptotically-safe-text-construction.md`](05-asymptotically-safe-text-construction.md)

같은 commit이 여러 Thread에 있으면 각 관점에서 다시 확인합니다. 이 세트에서는 `c30b39c0bcf8`이 heredoc recovery와 transactional allocation 양쪽에 의도적으로 등장합니다.

## Thread 문서 사용법

- 먼저 Thread 목표, 핵심 질문, 완료 기준을 읽습니다.
- Commit map 순서를 바꾸지 않고 한 commit씩 확인합니다.
- 각 commit의 `Source에서 확정된 변화`는 전제로 사용합니다.
- `확인할 실제 코드`에서 요구한 structure, function, caller/callee, state mutation, error branch, cleanup, test를 exact SHA에서 찾습니다.
- 확인한 최소 code snippet, file path, symbol, before/after 차이, 실행 결과를 학습 기록란에 남깁니다.
- Thread 끝에서 commit별 기록을 항상 유지해야 하는 조건 ledger와 Failure → Fix → Test 표로 다시 연결합니다.
- 마지막 architecture/execution flow는 source 문장을 복사하는 대신 실제 code evidence로 완성합니다.

## 해당 SHA 코드 확인 원칙

- `git show --name-status <SHA>`로 변경 파일을 먼저 확인합니다.
- 변경 전 상태는 `<SHA>^`, 변경 후 상태는 `<SHA>`에서 봅니다.
- 필요한 파일만 `git diff <SHA>^ <SHA> -- <path>`로 비교합니다.
- implementation 전체가 필요하면 `git show <SHA>:<path>` 또는 detached worktree를 사용합니다.
- Commit subject만으로 함수나 file을 추측하지 않습니다.
- Later fix에서 추가된 field, wrapper, test seam을 이전 SHA에 있다고 기록하지 않습니다.
- Thread에 같은 commit이 중복되더라도 제거하지 않습니다.

## final HEAD 소급 사용 금지

Final HEAD는 과거 commit의 code를 대신할 수 없습니다.

- 과거 commit의 소유권, 실패 처리, type field, function signature는 해당 SHA에서만 확정합니다.
- Later refactor로 이름이 바뀐 function을 과거 SHA의 이름처럼 쓰지 않습니다.
- Later 회귀 테스트를 과거 feature commit의 이미 존재한 증거처럼 쓰지 않습니다.
- 비교가 필요하면 source가 연결한 later fix/test를 별도 SHA로 확인하고, 당시 feature가 보장하지 못한 범위를 구분합니다.

## S/A/B/C별 학습 깊이

### S

Project architecture 또는 항상 유지해야 하는 조건을 설명하는 핵심 commit입니다.

- Problem과 commit 직전 상태
- 기존 설계의 failure 가능성
- 핵심 decision과 실제 중심 code
- 소유권, lifecycle, 상태 전이
- partial failure와 cleanup
- 후속 fix 또는 regression evidence
- 보장하는 것과 아직 보장하지 않는 것

을 모두 기록합니다.

### A

주요 subsystem, boundary, integration point, 실패 처리를 이해해야 합니다.

- 변경 전 assumption
- 핵심 function과 caller/callee
- state 또는 resource responsibility
- 실패 처리
- 설계 판단과 test evidence

를 확인합니다.

### B

Thread 흐름에서 맡는 구현 역할과 필요한 code/state 변화를 확인합니다.

- 해당 commit이 추가한 좁은 mechanism
- 핵심 data/function
- 정상·오류 분기
- 다음 중요한 commit에 제공하는 전제

를 기록합니다.

### C

Thread 이해에 필요한 문맥일 때만 확인합니다. 같은 깊이의 분석란을 억지로 만들지 않습니다.

## 실제 코드 삽입 기준

Code는 설명을 장식하기 위해 붙이지 않고, 다음 중 하나를 증명할 때만 최소 범위로 삽입합니다.

- 소유권을 획득·이전·해제하는 지점
- state mutation 전후 순서
- caller와 callee의 contract
- pipe/redirection 같은 resource acquisition·replacement·cleanup
- failure branch와 recovery branch
- partial result의 publish point
- retry, short read/write 또는 forced-stop 조건
- 회귀 테스트가 failure를 주입하고 production path를 통과하는 지점

각 snippet에는 exact SHA, file path, symbol, 왜 필요한 근거인지 적습니다. 긴 함수 전체보다 branch와 주변 문맥을 우선합니다.

## Test commit 학습 방법

각 test commit에서 다음을 분리해 기록합니다.

- 대상 production 항상 유지해야 하는 조건
- 재현하는 failure 또는 boundary
- fault injection, end-to-end, source-level API, stress, timeout, sanitizer 등 test technique
- 실제 통과하는 production code path
- expected status, stdout, stderr, state, resource 결과
- 이 테스트가 증명하는 것
- 이 테스트가 증명하지 않는 것
- broad integration인지 deterministic regression인지
- 이후 어떤 회귀를 막는지

Test script만 읽지 말고 injection seam과 production branch를 함께 연결합니다.

## 문서 완료 기준

- 모든 Development Thread가 정확히 한 문서에 존재합니다.
- 각 Thread의 commit order, SHA, subject, importance, tags가 source와 동일합니다.
- 여러 Thread에 속한 commit을 제거하지 않았습니다.
- 모든 S/A/B commit의 학습 깊이가 구분되어 있습니다.
- 실제 code를 읽지 않고 임의로 완성한 설명이 없습니다.
- 각 중요한 commit에 exact SHA의 file/function/branch 근거가 있습니다.
- Fix와 회귀 테스트가 기존 가정, failure, root cause, 수정 항상 유지해야 하는 조건을 통해 연결됩니다.
- 항상 유지해야 하는 조건 ledger에 도입, 강화, 실패 노출, fix, test evidence가 기록되어 있습니다.
- 각 Thread의 최종 architecture/execution flow를 commit history에 근거해 설명할 수 있습니다.
- 별도의 프로젝트 재학습 없이 설계 → 구현 → 실패 → 수정 → 검증의 발전 과정을 재구성할 수 있습니다.
===== END FILE: README.md =====
