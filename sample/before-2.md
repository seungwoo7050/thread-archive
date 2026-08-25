# Pipeline process and descriptor ownership under partial failure
> 한국어 주제: **부분 실패에서의 pipeline process와 descriptor 소유권**
>
> Project: `small-shell`  
> Branch: `c/minishell`  
> Development Thread order: 3/5
## 1. Thread 목표
single-command fork/exec에서 N-stage pipe graph로 확장되는 정상 경로와, pipe/fork/wait/dup/open 실패 뒤 parent가 PID와 FD를 끝까지 회수하는 경로를 함께 복원합니다.
**Source-defined significance**
> The normal pipeline mechanism is only half of the engineering problem. Once a parent records a PID or acquires a descriptor, it owns that resource even if later construction fails. This thread moves from normal execution to deterministic failure injection, root-cause cleanup, unrecoverable parent-state handling, and direct lifecycle observation. Supporting wrappers and tests remain below S because the decisive ownership guarantees are established by the pipe graph and partial-construction cleanup commits.
**학습 관점**
정상 pipeline wiring만으로는 실행기가 완성되지 않습니다. Parent가 PID를 기록하거나 FD를 획득한 순간부터 후속 단계가 실패해도 그 자원을 종료·관찰·close할 책임이 생깁니다.
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
- [x] single command와 multi-stage pipeline의 parent/child responsibility를 표로 작성했습니다.
- [x] 각 process가 보유·dup·close하는 pipe end를 stage별로 그렸습니다.
- [x] mid-fork failure에서 close → signal → reap 순서를 exact code로 확인했습니다.
- [x] one-shot restore failure와 repeated unrecoverable restore failure의 status/running 변화를 구분했습니다.
- [x] fault-injection regression과 lifecycle stress/probe가 무엇을 증명하는지 기록했습니다.
- [x] pipe creation failure의 acquisition/cleanup matrix에 PID table까지 포함했습니다.
> 실행 범위: exact SHA의 commit diff와 source/test scripts를 GitHub repository에서 검사했습니다. Branch checkout이 불가능해 build, fault suite, lifecycle stress는 실행하지 않았습니다.
## 4. Commit map
| 순서 | SHA | Subject | Importance | Tags | Source-defined role |
| ---: | --- | --- | :---: | --- | --- |
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
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
- parent persistent state와 child copy state의 차이: parent builtin은 original `t_shell`을 mutate하고, pipeline/ordinary child builtin의 mutation은 fork copy에만 남습니다.
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
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/ownership 변화를 확인합니다.
#### Source에서 확정된 변화
Pipeline의 각 command를 fork하고 one PID slot per command에 기록한 뒤 exact PID 순서로 reap합니다. 완전 spawn이면 last command status, partial spawn이면 existing children을 reap한 뒤 status 1입니다.
#### 설계·상태 변화 기록
- 이 commit 직전 상태: one child PID만 local로 wait했습니다.
- 해결하려던 문제: multiple commands의 identity와 last-stage status를 유지하려면 source-order PID bookkeeping이 필요했습니다.
- 기존 표현·실행 순서가 충분하지 않았던 이유: generic `wait`는 어느 result가 last stage인지 보장하지 않고, fork 중간 실패 시 생성된 child range를 알 수 없습니다.
- 선택한 결정: `command_count` 크기의 PID table을 만들고 successful fork마다 동일 index slot에 기록하며 `spawned` count를 유지합니다.
- publish 또는 state mutation이 일어나는 지점: successful fork 반환 직후 `pids[index] = pid`, `spawned++`입니다.
- failure 뒤 cleanup 또는 상태: partial spawn이면 recorded range만 exact PID wait하고 status 1; complete spawn일 때만 final slot result를 pipeline status로 사용합니다.
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
- wait ownership 종료 지점: each exact PID의 `waitpid` success 후 해당 slot lifecycle responsibility가 끝납니다.
- 아직 없는 descriptor topology: child들이 fork되지만 stdin/stdout pipe wiring은 없습니다.
- 확인한 변경 파일: `src/exec.c`.
- 핵심 caller → callee: pipeline dispatcher → `run_forked_commands` → fork loop → exact wait loop.
- parent SHA와 비교한 최소 before/after snippet: single PID local에서 command-count PID table + spawned count로 확장됐습니다.
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact source에서 bookkeeping과 status branch를 검사했습니다.
#### 보장 범위
- 이 commit이 보장하는 것: parent는 child identity를 결정적으로 소유·관찰하고 pipeline status를 last stage에 연결할 bookkeeping을 갖습니다.
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
- 학습 깊이: Architecture / invariant 핵심. 변경 전 가정, failure 가능성, 결정, core code, ownership/lifecycle, follow-up을 추적합니다.
#### Source에서 확정된 변화
N command에 대해 N-1 pipe를 만들고, child i의 stdin/stdout을 neighboring pipe에 연결한 뒤 모든 original pipe end를 닫고 explicit redirection을 나중에 적용합니다.
#### Source가 확정한 핵심 판단
- **문제**: Multiple forked commands do not form a pipeline unless each child receives the correct neighboring descriptors and every process closes all unused pipe ends.
- **결정**: Allocate `N - 1` pipes for `N` commands, map the previous read end to stdin and the next write end to stdout in each child, close all original ends, then apply explicit command redirections afterward.
- **중요한 이유**: The ordering defines both data flow and redirection precedence. Parent and child closure rules prevent readers from waiting forever on hidden writers, while child execution of pipeline builtins prevents state mutations from leaking into the parent shell.
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
- Single parent builtin만 parent path이고 pipeline builtin은 child path입니다.
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
- pipeline builtin isolation: command count > 1이면 builtin도 child에서 실행되어 parent env/cwd에 반영되지 않습니다.
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
- 이 commit이 보장하는 것: multi-stage data flow, descriptor precedence, parent/child closure, last-stage status의 defining execution topology를 제공합니다.
- 아직 보장하지 않는 것: later fork failure 시 already spawned child를 terminate하지 않아 block/hang할 수 있는 failure path가 남습니다.
#### Thread 내 다음 연결
`915aa072298b`가 pipe/fork/wait failure를 재현할 seam을 만들고 `be2967a4b946`가 lifecycle invariant를 복구합니다.
### 5.4 `915aa072298b` — `refactor(runtime): 프로세스 시스템 호출 경계 분리`
#### 확정 정보
- SHA: `915aa072298b`
- Subject: `refactor(runtime): 프로세스 시스템 호출 경계 분리`
- Importance: **A**
- Tags: `PROCESS`, `FAILURE`, `TEST`
- Source-defined role: Introduces deterministic pipe, fork, and wait failure seams.
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
`pipe`, `fork`, `waitpid`를 runtime wrapper 뒤로 옮기고 selected call count에서 representative `errno`로 실패시키는 deterministic test seam을 추가합니다. Production wrapper는 transparent합니다.
#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: raw system calls의 rare later-call failure를 반복 재현할 수 없었습니다.
- 새 boundary가 제공하는 contract: production에서는 raw call을 그대로 위임하고 test build에서 operation별 N번째 call을 deterministic하게 실패시킵니다.
- production semantics가 유지된다는 코드 근거: `#ifdef SMALL_SHELL_TESTING` injection branch 외에는 wrapper가 arguments/return/errno를 raw call에 그대로 전달합니다.
- ownership 또는 call-site responsibility 변화: executor의 cleanup policy는 변하지 않고 call site만 wrappers를 사용합니다. 따라서 seam 자체는 자원 owner가 아닙니다.
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
- parent SHA와 비교한 최소 before/after snippet: raw system call sites가 transparent wrappers로 치환됐습니다.
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
- Source-defined role: Terminates and reaps children after partial pipeline construction.
- 학습 깊이: Architecture / invariant 핵심. 변경 전 가정, failure 가능성, 결정, core code, ownership/lifecycle, follow-up을 추적합니다.
#### Source에서 확정된 변화
Later fork failure 시 parent-held pipe ends를 모두 닫고, 이미 기록된 child에 `SIGKILL`을 보낸 뒤 every PID를 reap하여 pipeline execution이 complete cleanup state로 수렴하도록 합니다.
#### Source가 확정한 핵심 판단
- **문제**: If a later fork failed, already spawned stages could remain blocked or running. Closing descriptors and waiting was insufficient and could hang indefinitely or leave zombies.
- **결정**: On partial construction, close all parent-held pipe ends, send termination to every recorded child, tolerate children that already exited, and still reap every PID. Treat any wait failure as pipeline failure.
- **중요한 이유**: Recording a PID transfers lifecycle responsibility to the parent even when the pipeline never becomes complete. The cleanup path must converge to the same terminal ownership state as successful execution.
- **확정된 변경 범위**: The executor gained child termination, structured wait retry and error reporting, status suppression when the last child was not observed cleanly, and complete cleanup after a short spawn sequence.
- **프로젝트 이해에서의 위치**: This commit converts the pipeline from a normal-path mechanism into a reliable lifecycle owner. It explains the later fault-injection and leak-verification architecture.
#### Fix 재구성 기록
- 기존 가정: parent pipe ends를 닫고 recorded children을 ordinary wait하면 partial pipeline도 자연 종료한다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: first child가 `sleep 30`이거나 incomplete pipe를 기다리는 상태에서 second fork가 실패하면 wait가 장시간 block합니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: PID가 recorded된 순간 parent ownership이 생겼지만 incomplete graph에서 child progress를 보장할 termination policy가 없었습니다.
- 수정된 invariant 또는 decision: partial spawn이면 parent FDs close → every recorded child signal → every recorded PID reap 순서로 강제 수렴합니다.
- 변경 전 코드 증거: `spawned < command_count`에서도 same wait loop만 사용했습니다.
- 변경 후 코드 증거: `terminate_children`와 structured `wait_for_child`를 추가하고 partial branch가 kill/reap를 호출합니다.
- 연결되는 regression test와 그 한계: `d611196b368e`가 mid-fork failure와 wait failure를 deterministic하게 검증하고 `b42e57eb7755`가 direct child probe를 추가합니다. Arbitrary descendants까지 증명하지 않습니다.
#### `be2967a4b946`에서 확인할 실제 코드
- Parent SHA의 partial branch와 simple wait를 비교했습니다.
- Parent pipe close가 termination 전에 일어납니다.
- Recorded range만 `kill(pid, SIGKILL)`하며 already-exited child 오류를 허용합니다.
- Signal 후 every PID를 `wait_for_child`로 관찰합니다.
- Wait helper는 `EINTR`를 계속 retry하고 non-EINTR error 후 제한된 재시도를 하며 any error evidence를 남깁니다.
- Any wait failure면 otherwise valid last-stage status를 버리고 pipeline status 1입니다.
- Success/partial paths 모두 pipe FDs closed, PIDs observed, tables freed terminal state로 수렴합니다.
#### 학습자가 남길 코드 증거
- 기존 가정: descriptor closure alone guarantees child exit.
- block 가능한 concrete pipeline scenario: `sleep 30 | cat`에서 second fork failure 후 `sleep`은 own timer까지 살아 있고 parent wait가 block합니다.
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
- 해당 SHA에서 실행한 test 또는 수동 재현 결과: 실행하지 않았습니다. Exact parent/fix diff와 terminal ownership states를 검사했습니다.
#### 보장 범위
- 이 commit이 보장하는 것: recorded PID는 pipeline 완성 여부와 무관하게 parent가 종료하거나 관찰하며, partial construction이 hang/zombie를 남기지 않습니다.
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
`SMALL_SHELL_TESTING` fault binary로 later pipe creation, mid-pipeline fork, waitpid failure를 주입하고 각 failed pipeline이 status 1로 끝난 뒤 following `echo $?`가 이를 관찰하는지 검증합니다.
#### Test commit 학습 기록
- 대상 production invariant: partial acquisition/spawn과 wait observation failure가 hang, zombie, false last-stage success를 남기지 않아야 합니다.
- 재현하는 failure 또는 boundary: second pipe creation, second fork, first waitpid failure입니다.
- 사용한 test technique: production source를 `SMALL_SHELL_TESTING`으로 재build하고 runtime wrapper call counter를 environment로 선택하는 deterministic failure injection입니다.
- 실제 통과하는 production code path: pipe table create cleanup, partial spawn close/kill/reap, wait error override, shell line continuation/status expansion입니다.
- 이 테스트가 증명하는 것: 각 injected failure가 pipeline status 1로 귀결되고 same shell의 next `echo $?`가 1을 출력하며 test가 종료됩니다.
- 이 테스트가 증명하지 않는 것: 장기간 FD 누수, all call positions, arbitrary child descendants, real kernel resource pressure는 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: operation/call-position-specific deterministic regression입니다.
- 후속 변경에서 막는 회귀: partial child termination 제거, wait error 무시, partial pipe cleanup 누락, failure 뒤 status continuation 손실입니다.
#### `d611196b368e`에서 확인할 실제 코드
- `Makefile`의 `small-shell-test`가 same production sources에 test definition만 추가합니다.
- `tests/faults.sh`는 later call occurrence를 선택합니다.
- Mid-fork fixture는 long-running first child를 사용해 kill/reap가 없으면 종료가 지연되도록 만듭니다.
- Each input은 failed pipeline 뒤 `echo $?`를 같은 process에 제공합니다.
- Wait failure case는 otherwise successful last-stage result보다 status 1이 우선함을 검사합니다.
#### 학습자가 남길 코드 증거
- 대상 production invariant: every acquired FD/recorded PID reaches cleanup and only cleanly observed last-stage status is trusted.
- 각 injected operation/call position: pipe #2, fork #2, waitpid #1입니다.
- partial resource state: one pipe pair 또는 one child PID가 이미 parent-owned인 상태입니다.
- production cleanup path: created pipe ends close; partial child kill/reap; tables free; wait error flag forces 1.
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
`open`, `dup`, `dup2`를 `shell_open`, `shell_dup`, `shell_dup2` wrapper 뒤로 옮기고, selected position부터 반복 실패하는 repeat mode를 추가합니다.
#### Refactor 판단 기록
- 기존 abstraction 또는 cost/failure 관찰 한계: parent stdio save/apply/restore의 특정 later call과 retry까지 연속 실패시키기 어려웠습니다.
- 새 boundary가 제공하는 contract: one-shot N번째 failure와 N번째부터 계속 실패하는 repeat mode를 제공하고 production에서는 raw operation을 그대로 위임합니다.
- production semantics가 유지된다는 코드 근거: test-only selector branch 외 wrapper arguments/return/errno는 raw `open`/`dup`/`dup2`와 동일합니다.
- ownership 또는 call-site responsibility 변화: descriptor ownership은 executor/redirection caller에 남고 wrappers는 only call boundary입니다.
- 후속 fix/test가 이 seam을 사용하는 방식: save failure, apply failure, one-shot restore failure, retry까지 실패하는 persistent restore failure를 별도로 재현합니다.
#### `fd5c76c18c27`에서 확인할 실제 코드
- `src/runtime.h/.c`의 new wrappers와 call sites를 확인했습니다.
- Pipeline child wiring, file redirection, heredoc installation, parent stdio save/restore가 wrapper를 사용합니다.
- One-shot mode는 selected call 한 번만 실패하고 repeat mode는 target 이후 matching operation을 계속 실패시킵니다.
- Production build는 injection state를 포함하지 않습니다.
- 이 commit은 error policy 자체를 바꾸지 않고 seam만 확장합니다.
#### 학습자가 남길 코드 증거
- descriptor wrapper coverage map:
| Wrapper | 주요 call site |
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
Parent-executed command의 stdin/stdout restore를 독립적으로 retry하고, transient error도 status 1로 남기며, 어느 descriptor라도 복구 불가능하면 `running`을 clear해 shell을 중단합니다.
#### Fix 재구성 기록
- 기존 가정: parent builtin 뒤 saved stdin/stdout `dup2` 복원은 실패하더라도 diagnostic만 내거나 결과를 무시해도 다음 command를 실행할 수 있다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: stdout이 redirected target에 남거나 stdin이 wrong source에 남으면 다음 prompt/command의 I/O가 예측 불가능합니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: process-persistent descriptors를 temporary command state로 바꾼 뒤 restoration success를 next-command precondition으로 취급하지 않았습니다.
- 수정된 invariant 또는 decision: stdin과 stdout을 독립적으로 복원하고 `EINTR`는 retry, other one-shot failure도 한 번 더 시도합니다. Error가 한 번이라도 있었으면 status 1, final failure면 `running=0`입니다.
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
- Redirection setup failure도 same restore cleanup path로 합류합니다.
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
- setup failure와 normal execution이 합류하는 cleanup path: both go to restore + close saved descriptors.
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
Parent redirection의 descriptor save, target open, replacement apply, original restore 각 phase에 failure를 주입하고, recoverable case의 continuation과 repeated restore failure의 forced stop을 검증합니다.
#### Test commit 학습 기록
- 대상 production invariant: parent command는 state mutation 전에 stdio setup이 성공해야 하고, command 뒤 stdio가 restored이거나 shell이 stopped여야 합니다.
- 재현하는 failure 또는 boundary: stdin save dup, stdout save dup, output open, replacement dup2, stdin restore, stdout restore, persistent restore입니다.
- 사용한 test technique: operation/call-position fault injection과 same-process next-command assertions입니다.
- 실제 통과하는 production code path: parent command save → redirection open/apply → builtin → flush → restore retry/stop.
- 이 테스트가 증명하는 것: setup failures suppress command payload/state effect, recoverable errors yield status 1 and continuation on normal stdout, repeated restore failure suppresses following command and exits status 1.
- 이 테스트가 증명하지 않는 것: long-run descriptor exhaustion, all close errors, child pipeline redirection combinations은 별도입니다.
- broad integration / deterministic regression / stress·probe 중 분류: phase-specific deterministic regression matrix입니다.
- 후속 변경에서 막는 회귀: save cleanup 누락, failure 후 builtin execution, restore error suppression, unsafe continuation입니다.
#### `13645f58d5e6`에서 확인할 실제 코드
- `tests/faults.sh`의 case마다 `dup`, `open`, `dup2` operation과 occurrence가 다릅니다.
- One-shot setup failure는 builtin output/state effect가 발생하지 않고 current status 1입니다.
- Recoverable restore case 뒤 marker는 original stdout에서 관찰됩니다.
- Repeat-mode restore case는 retry까지 실패하고 `echo never`가 출력되지 않으며 diagnostic/status 1입니다.
- Test input은 exact parent-executed builtin path를 사용합니다.
#### 학습자가 남길 코드 증거
- 대상 production invariant: parent stdio lifecycle is transactional across save/apply/execute/restore.
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
- 이 commit이 보장하는 것: parent descriptor lifecycle의 negative behavior와 recovery behavior, unrecoverable stop decision을 regression으로 고정합니다.
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
- 학습 깊이: 주요 subsystem boundary, integration point 또는 failure path. 핵심 코드와 설계 판단을 확인합니다.
#### Source에서 확정된 변화
48-descriptor limit 아래 parent redirection, three-stage pipeline, file I/O를 반복하고, test-only `waitpid(-1, ..., WNOHANG)` probe로 live/zombie direct child가 남는지 검사합니다.
#### Test commit 학습 기록
- 대상 production invariant: repeated normal execution 뒤 FD count가 누적되지 않고 each direct child가 already reaped돼야 합니다.
- 재현하는 failure 또는 boundary: single-run output으로 보이지 않는 descriptor leak과 unreaped/zombie child accumulation입니다.
- 사용한 test technique: low FD limit stress workload + test-only post-pipeline direct child probe + timeout/process-group harness입니다.
- 실제 통과하는 production code path: parent builtin redirection save/apply/restore, 3-stage pipe creation/fork/close/wait, file input/output redirection, timeout cleanup입니다.
- 이 테스트가 증명하는 것: configured repeated workload가 descriptor exhaustion 없이 final marker에 도달하고 test process에 waitable/live direct child가 남지 않습니다.
- 이 테스트가 증명하지 않는 것: all descendant grandchildren, unlimited workloads, every OS resource leak, production build에 probe가 존재함을 증명하지 않습니다.
- broad integration / deterministic regression / stress·probe 중 분류: broad lifecycle stress와 deterministic direct-child probe의 결합입니다.
- 후속 변경에서 막는 회귀: parent saved FD close 누락, pipe end close 누락, child reap 누락, timeout child orphaning입니다.
#### `b42e57eb7755`에서 확인할 실제 코드
- `tests/lifecycle.sh`가 subshell의 descriptor limit을 48로 설정합니다.
- Workload는 60회 parent redirection, three-stage pipeline, temporary file write/read를 섞습니다.
- Expected stdout는 final `fd-ok` marker이고 stderr는 비어 있어야 합니다.
- Test-only `shell_children_reaped`는 `waitpid(-1, ..., WNOHANG)`을 반복해 child가 있거나 probe가 직접 하나라도 reap하면 failure로 간주하고, `ECHILD`에서 only no-found success입니다.
- Timeout fixture는 long-running pipeline이 timeout status 124가 되는지 확인합니다.
- Timeout runner는 launched process group을 종료·회수해 orphan을 피합니다.
- Probe는 `SMALL_SHELL_TESTING` 안에만 있습니다.
#### 학습자가 남길 코드 증거
- 대상 resource invariant: no cumulative parent FD ownership and no unreaped direct child after a pipeline returns.
- workload 반복 횟수/구성: 60 iterations, parent redirection + 3-stage pipe + file copy/read/write.
- FD exhaustion 관찰 방식: `ulimit -n 48` 아래 final marker 도달; leak가 누적되면 later `open`/`pipe`/`dup`이 실패합니다.
- direct child probe 결과 해석: `waitpid` returns child PID이면 leak/zombie; 0이면 live child; `-1/ECHILD` and no prior found만 success입니다.
- timeout/process-group 관련 assertion: 30-second child pipeline을 short timeout으로 끊고 124를 요구하며 runner가 process group을 cleanup합니다.
- broad stress와 deterministic probe의 역할 구분: FD는 exhaustion stress로 간접 관찰, child는 waitpid probe로 직접 관찰합니다.
- 증명하지 않는 descendant 범위: direct children만 확인하며 grandchildren/external daemonization은 범위 밖입니다.
- 확인한 변경 파일: `tests/lifecycle.sh`, test-only executor hook, `tests/timeout_runner.c`, `Makefile`.
- 핵심 caller → callee: lifecycle script → test binary repeated workloads → executor cleanup → test-only child probe; timeout script → runner → process group cleanup.
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
- 학습 깊이: Thread 흐름에서 맡는 구현 역할과 필요한 state/ownership 변화를 확인합니다.
#### Source에서 확정된 변화
PID table과 pipe table을 모두 할당한 뒤 pipe creation이 실패하는 return path에서 opened pipe ends, pipe table뿐 아니라 아직 local인 PID table도 해제합니다.
#### Fix 재구성 기록
- 기존 가정: pipe creation failure cleanup에서 pipe-related resources만 정리하면 된다고 보았습니다.
- 실제 failure 또는 위험을 드러내는 입력·상태: prior allocation-order fix로 PID table을 pipe creation 전에 확보하므로 second/first pipe failure 시 unused PID table allocation이 live입니다.
- root cause가 위치한 representation / lifecycle / ordering boundary: acquisition order가 바뀌었지만 old failure label의 cleanup list에 newly preallocated PID table이 추가되지 않았습니다.
- 수정된 invariant 또는 decision: every local preparation allocation is listed in every exit after its acquisition, even before child spawn.
- 변경 전 코드 증거: partial pipe ends close + pipe table free + return, without `free(pids)`.
- 변경 후 코드 증거: same branch에 PID table free 한 줄이 추가됩니다.
- 연결되는 regression test와 그 한계: 이 commit에는 전용 test가 없습니다. Existing pipe fault seam, allocation sweep, later sanitizer path가 관찰 수단이지만 실제 실행 결과를 이 commit에 소급하지 않습니다.
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
- 핵심 caller → callee: pipeline executor → table allocations → pipe creation loop → failure close/free branch.
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
- 아직 보장하지 않는 것: project-wide ownership model을 바꾸지 않는 narrow cleanup fix입니다.
#### Thread 내 다음 연결
Pipeline Thread의 마지막 commit입니다. 최종 ledger에서 모든 acquisition path가 terminal cleanup state로 수렴하는지 확인합니다.
## 6. Invariant ledger
Source가 명시한 invariant와 engineering difficulty를 유지하고 exact code 근거를 채웠습니다.
| Invariant | Source에서 확정된 의미 | 처음 도입/표현 | 강화·복구·검증 | 학습자가 확인한 코드 근거 |
| --- | --- | --- | --- | --- |
| Every recorded child PID remains parent-owned until termination or observation. | parent가 PID를 기록한 뒤에는 pipeline이 완성되지 않아도 해당 child를 종료하거나 wait로 관찰해야 합니다. | `ae988017efd5` | `be2967a4b946`, `d611196b368e`, `b42e57eb7755` | Fork success 직후 PID slot/`spawned` publish; partial branch close→kill→exact wait; wait fault status override; test-only no-child probe. |
| Each process closes every pipe end it does not need. | 숨은 writer/read end가 EOF와 resource lifetime을 방해하지 않도록 parent와 child 모두 불필요한 pipe end를 닫아야 합니다. | `a71f98de0d92` | `be2967a4b946`, `b42e57eb7755` | Child dup formula 뒤 all originals close, parent spawn loop 뒤 all ends close, failure도 close before kill/reap; low-FD stress observes accumulation. |
| Parent standard streams are restored before the next command. | restore가 불가능하면 이후 command I/O가 신뢰 불가능하므로 shell은 계속 실행하지 않습니다. | `fd5c76c18c27`의 injectable boundary | `2ca9f4299c7f`, `13645f58d5e6` | save/apply/restore tri-state; one-shot recovered error→status 1, repeat failure→`running=0`; next marker/no-marker assertions. |
| Shell-visible status reflects cleanly observed process results. | 정상 exit, signal, 126, 127 mapping과 wait failure를 구분해야 합니다. | `7c9646e7cd79` | `be2967a4b946`, `d611196b368e` | `status_from_wait`, exec errno mapping, any wait error suppresses last-stage status and returns 1. |
### Ledger 작성 시 확인한 것
- PID slot은 `ae988...`에서 도입됐지만 incomplete graph ownership은 `be296...`에서 완성됐습니다.
- Pipe topology 결정은 `a71f...`, failure convergence는 `be296...`, observation evidence는 `d611...`/`b42e...`로 분리했습니다.
- Test seam commits는 ownership policy를 만들지 않고 rare branch를 deterministic하게 관찰합니다.
- Success, pipe failure, partial fork, wait failure, restore failure 모두 acquired FDs/tables/PIDs의 explicit terminal state를 가집니다.
## 7. Failure → Fix → Test 연결
| 기존 가정 또는 문제 | Feature / 기존 상태 | Fix 또는 결정 | Regression / 확인 방법 | 학습자 코드 근거 |
| --- | --- | --- | --- | --- |
| later fork failure 뒤 이미 생성된 child가 pipe에서 block되어 wait가 끝나지 않음 | `a71f98de0d92`의 normal pipe graph | `be2967a4b946` — parent pipe close, recorded child termination, complete reaping | `915aa072298b` seam → `d611196b368e` deterministic pipe/fork/wait failures → `b42e57eb7755` lifecycle probe | PID publish, close→kill→wait order, second-fork fixture, direct `waitpid(-1,WNOHANG)` probe를 연결했습니다. |
| parent builtin redirection restore failure를 무시하면 persistent stdin/stdout이 손상됨 | `fd5c76c18c27`의 dup/open seam | `2ca9f4299c7f` — independent retry, status 1, unrecoverable stop | `13645f58d5e6` — save/apply/restore/open/repeated failure matrix | `restore_one` tri-state와 repeat-mode no-following-command assertion을 연결했습니다. |
| pipe creation failure 전 preallocated PID table이 return path에 남음 | execution table을 먼저 할당하는 preparation order | `6dff1ba86ba6` — partial pipe cleanup에 PID table free 추가 | Thread 내 별도 전용 test commit은 없으므로 fault-injection 또는 sanitizer 실행 결과와 allocation/free code를 직접 기록합니다. | Exact diff에서 `free(pids)` 추가, fork-before/after ordering으로 child count 0을 확인했습니다. Runtime sanitizer는 실행하지 않았습니다. |
## 8. Ownership / state / responsibility 변화
| 대상 | Owner / 책임 주체 | 책임 종료 시점 | 해당 SHA에서 확인할 내용 | 학습자 기록 |
| --- | --- | --- | --- | --- |
| PID slot | parent executor | wait 또는 partial-failure termination/reap 완료 | record 시점과 valid count를 확인 | Fork success 직후 index slot에 publish하고 `spawned`가 valid range를 정의합니다. |
| pipe table | parent 준비 단계 | parent close 후 table free | 각 slot `-1` 초기화와 partial creation cleanup 확인 | All entries `-1`; pipe success마다 pair가 live; success/failure 모두 close helper 후 memory free입니다. |
| child inherited pipe ends | 각 child stage | dup2 뒤 모든 original end close | stage index별 필요한 end와 불필요한 end 기록 | previous read/current write만 stdio에 duplicate하고 table의 every original end를 close합니다. |
| saved stdin/stdout | parent-command executor | restore attempts 완료 후 close | 둘을 독립적으로 복원하는 code 확인 | Acquisition failure cleanup, per-target restore retry, final close가 outcome과 무관하게 수행됩니다. |
| external environment vector | child before exec | exec 성공 시 process image로 이전, 실패 시 child cleanup | serialization과 failure status 기록 | Child local owner; successful exec replaces image, failure path frees/terminates with 126/127 or allocation status. |
| last-stage wait result | parent status calculation | 모든 required wait가 clean할 때만 사용 | wait error override branch 기록 | Full spawn and no wait error일 때만 last recorded PID result를 사용합니다. |
## 9. Thread 최종 상태
N-stage graph는 `N-1` pipe pairs를 사용합니다. Single stateful builtin만 parent에서 실행되고, multi-stage pipeline의 모든 commands/builtins는 child에서 실행됩니다.
| Terminal path | Child/FD/table 결과 | Shell-visible result |
| --- | --- | ---: |
| normal complete pipeline | parent/children close unused ends; every PID exact wait; tables free | last clean stage status |
| pipe creation failure | opened prefix close; both tables free; no child | 1 |
| mid-fork failure | parent ends close; recorded children kill/reap; tables free | 1 |
| wait failure | remaining waits attempted; tables free | 1, last status ignored |
| parent restore transient error | stdio restored; saved FDs close | 1, continue |
| parent restore permanent error | saved FDs close; stdio untrusted | 1, `running=0` |
### 최종 상태 기록
- 최종적으로 유지되는 data/resource ownership: executor local이 tables와 parent pipe FDs를, parent가 recorded PIDs를, each child가 inherited descriptors를 dup/close 시점까지 소유합니다. Parent command helper는 saved stdio를 소유합니다.
- 최종적으로 보장되는 execution 또는 recovery rule: record/acquire한 자원은 graph 완성 여부와 무관하게 close·terminate·wait·free로 회수되며, parent stdio를 신뢰할 수 없으면 다음 command를 실행하지 않습니다.
- Thread가 해결한 가장 어려운 failure: incomplete pipeline에서 이미 실행 중이거나 block된 child를 남기지 않고 deterministic terminal ownership으로 수렴시키는 문제입니다.
- Thread 밖에 남아 있는 보장 범위: arbitrary descendants, all possible close errors/kernel failures, infinite workload는 test 범위 밖입니다.
## 10. 최종 architecture 또는 execution flow 정리
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
- 핵심 entry function: pipeline dispatcher와 `run_forked_commands`; parent path의 parent-command executor입니다.
- 주요 caller → callee chain: sequence executor → one-pipeline dispatcher → table/pipe setup → fork loop → child pipe setup/redirection/builtin-or-exec; parent close → `wait_for_child`; failure branch → `terminate_children`.
- state mutation 순서: memory allocations → live pipe slots → PID records → child stdio dup → parent closes → wait observations → result selection; parent builtin에서는 saved descriptors → temporary redirection → restore result → status/running.
- ownership transfer 순서: pipe() returns to parent table and is inherited by child; child duplicates then closes originals; parent closes its copies; PID record stays parent-owned through wait; saved stdio stays local through restoration.
- failure convergence path: pre-fork pipe failure closes/free locals; mid-fork failure close/kill/reap; wait failure forces 1; permanent stdio restore failure stops shell.
- regression evidence: process failure matrix, FD restoration matrix, low-FD lifecycle stress, direct child probe가 source에 존재합니다. 실제 commands는 실행하지 않았습니다.
## 11. 학습 완료 자가 점검
- [x] 모든 commit을 exact SHA에서 확인했고 final HEAD를 소급하지 않았습니다.
- [x] Commit map의 SHA, subject, importance, tags, order를 변경하지 않았습니다.
- [x] S commit은 problem, prior state, failure possibility, decision, core code, ownership/lifecycle, follow-up을 설명했습니다.
- [x] A commit은 subsystem boundary 또는 failure path와 실제 핵심 code를 설명했습니다.
- [x] B commit은 Thread 내 구현 역할과 state/ownership 변화를 설명했습니다.
- [x] Fix commit은 기존 가정 → failure → root cause → 수정 invariant → code → regression 순으로 연결했습니다.
- [x] Test commit은 invariant, failure, technique, production path, prove/not prove를 구분했습니다.
- [x] Invariant ledger의 각 행에 실제 file/function/branch 근거가 있습니다.
- [x] 정상·실패 경로 모두에서 resource와 partial object의 terminal owner를 설명했습니다.
- [x] 이 Thread의 설계 → 구현 → 실패 → 수정 → 검증 흐름을 commit history 순서로 재구성했습니다.