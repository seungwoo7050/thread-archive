# Thread: 파이프라인 프로세스와 descriptor 소유권 — 부분 실패까지
Project: `small-shell` · Branch: `c/minishell` · 문서 번호: 03
## 개요
단일 명령의 fork/exec가 N-stage pipe graph로 확장되는 정상 경로와, pipe/fork/wait/dup/open
실패 뒤 parent가 이미 기록한 PID와 이미 획득한 FD를 끝까지 회수하는 경로를 함께 다룬다.
정상적인 pipe 연결 자체는 문제의 절반일 뿐이다. parent가 PID를 기록하거나 FD를 획득한
순간부터, 그 뒤의 어떤 단계가 실패하더라도 그 자원을 종료·회수·close할 책임이 생긴다.
이 Thread는 그 책임이 커밋 11개에 걸쳐 어떻게 자리 잡는지를 다룬다 — 정상 wiring
(`a71f98de0d92`), 실패 주입을 가능하게 하는 계측(`915aa072298b`, `fd5c76c18c27`),
실제 회수 로직(`be2967a4b946`, `2ca9f4299c7f`, `6dff1ba86ba6`), 그리고 그 회수가 실제로
작동하는지 직접 관찰하는 테스트(`d611196b368e`, `13645f58d5e6`, `b42e57eb7755`) 순으로
얽혀 있다.
### 최종 상태
| 종료 경로 | Child / FD / 테이블 처리 | Shell에 보이는 결과 |
| --- | --- | ---: |
| 정상 완주 | parent/child가 안 쓰는 pipe end를 각자 close; 모든 PID를 정확히 wait; 테이블 free | 마지막 stage의 종료 상태 |
| pipe 생성 실패 | 이미 연 pipe들 close; 두 테이블 모두 free; 자식은 하나도 생성되지 않음 | 1 |
| fork 도중 실패 | parent 쪽 pipe end close; 이미 만든 자식 전원 SIGKILL 후 회수 시도 | 1 |
| wait 실패 | 남은 자식들도 회수를 계속 시도 | 1 (마지막 stage 상태 무시) |
| parent 복원 — 일시적 실패 | stdin/stdout 재시도 후 복원; saved fd close | 1, shell은 계속 실행 |
| parent 복원 — 영구 실패 | saved fd close; stdin/stdout을 더 이상 신뢰하지 않음 | 1, `shell->running = 0` |
### 커밋 목록
| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7c9646e7cd79` | feat(exec): 단일 명령을 자식에서 실행 | A | `PROCESS, CORE, INTEGRATION` | 단일 명령의 fork/exec와 종료 상태 매핑을 확립 |
| 2 | `ae988017efd5` | feat(exec): pipeline 자식 상태를 순서대로 회수 | B | `PROCESS, CORE` | 여러 명령에 대한 PID 기록과 순차 회수 추가 |
| 3 | `a71f98de0d92` | feat(exec): 다단 pipeline의 pipe FD 연결 | S | `PROCESS, FD_IO, CORE` | 다단 pipe graph 연결과 parent/child descriptor 정리 규칙 확립 |
| 4 | `915aa072298b` | refactor(runtime): 프로세스 시스템 호출 경계 분리 | A | `PROCESS, FAILURE, TEST` | pipe/fork/wait에 대한 결정적 실패 주입 seam 도입 |
| 5 | `be2967a4b946` | fix(exec): 부분 생성 파이프라인의 자식과 FD 회수 | S | `PROCESS, FD_IO, FAILURE` | 부분 생성된 파이프라인의 자식 종료와 FD 회수 |
| 6 | `d611196b368e` | test(exec): pipe·fork·wait 실패 회귀 검증 | A | `TEST, PROCESS, FAILURE` | pipe/fork/wait 실패를 결정적으로 재현 |
| 7 | `fd5c76c18c27` | refactor(runtime): FD 시스템 호출 경계 분리 | A | `FD_IO, FAILURE, TEST` | dup/dup2/open까지 실패 주입 범위 확장 |
| 8 | `2ca9f4299c7f` | fix(redirection): 부모 표준 입출력 복원 실패 전파 | A | `FD_IO, FAILURE, RISK` | parent stdio 복원 실패를 관찰 가능하게 만들고, 복구 불가능하면 shell을 중단 |
| 9 | `13645f58d5e6` | test(redirection): 저장·적용·복원 실패 회귀 검증 | A | `TEST, FD_IO, FAILURE` | 저장/적용/복원/open과 지속적 실패 경로를 검증 |
| 10 | `b42e57eb7755` | test(lifecycle): FD와 자식 프로세스 누수 검증 | A | `TEST, PROCESS, FD_IO` | FD 고갈과 미회수 자식을 직접 관찰 |
| 11 | `6dff1ba86ba6` | fix(exec): pipe 생성 실패 시 PID 배열 해제 | B | `FD_IO, FAILURE, DEBUG` | 자식이 하나도 생성되기 전 남아있던 PID 테이블 누수 제거 |
## 7c9646e7cd79 — feat(exec): 단일 명령을 자식에서 실행
**중요도** `A` · **태그** `PROCESS, CORE, INTEGRATION`
### 무엇을 만들었는가 (diff)
```diff
+static int status_from_wait(int wait_status)
+{
+    if (WIFEXITED(wait_status))
+        return WEXITSTATUS(wait_status);
+    if (WIFSIGNALED(wait_status))
+        return 128 + WTERMSIG(wait_status);
+    return 1;
+}
+
+static void run_child(t_shell *shell, const t_command *command,
+    const struct exec_context *ctx)
+{
+    if (exec_apply_redirections(command, ctx) != 0)
+        _exit(1);
+    if (command->argc == 0)
+        _exit(0);
+    if (builtin_is_known(command->argv[0])) {
+        int status;
+
+        status = builtin_run(shell, command->argv);
+        fflush(stdout);
+        fflush(stderr);
+        _exit(status & 0xff);
+    }
+    /* ... envp 구성 후 execvp, 실패 시 126/127 ... */
+}
+
+static int run_single_command(t_shell *shell, const t_command *command,
+    const struct exec_context *ctx)
+{
+    pid_t pid;
+    pid_t waited;
+    int wait_status;
+
+    pid = fork();
+    if (pid < 0) {
+        fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+        return 1;
+    }
+    if (pid == 0)
+        run_child(shell, command, ctx);
+    do {
+        waited = waitpid(pid, &wait_status, 0);
+    } while (waited < 0 && errno == EINTR);
+    if (waited != pid)
+        return 1;
+    return status_from_wait(wait_status);
+}
+
+int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
+{
+    /* ... */
+    if (command->argc == 0 || builtin_is_parent(command->argv[0]))
+        shell->last_status = exec_run_parent_command(shell, command, &ctx);
+    else
+        shell->last_status = run_single_command(shell, command, &ctx);
+    return shell->last_status;
+}
```
### 왜 가볍게 다루는가
이 시점에는 자원이 하나뿐이다 — 자식 PID 하나, 열려 있는 FD도 없다. 이 Thread가 다루는
"이미 획득한 자원을 실패 후에도 회수해야 한다"는 문제 자체가 아직 발생하지 않는다. 문제는
자원이 여러 개(pipe N-1개, PID N개)로 늘어나는 다음 커밋부터 시작된다.
다만 눈여겨볼 설계 결정이 하나 있다 — `execute_pipeline_list`가 이미 "빈 명령이거나
`builtin_is_parent`인 명령은 parent에서 그대로 실행"하고 "그 외는 child에서 실행"하는
분기를 갖고 있다는 점이다. 이 분기가 나중에 `2ca9f4299c7f`에서 parent stdio 복원 실패를
다루는 이유가 된다 — parent가 실제로 자기 stdin/stdout을 잠깐 바꿔가며 명령을 실행하는
경로이기 때문이다. 지금은 `pipeline->next != NULL || pipeline->command_count != 1`이면
그냥 실패 처리한다 — 파이프라인 자체는 아직 구현되어 있지 않다.
### 관련 커밋
다음 `ae988017efd5`가 이 단일 fork/wait를 여러 명령으로 확장한다.
## ae988017efd5 — feat(exec): pipeline 자식 상태를 순서대로 회수
**중요도** `B` · **태그** `PROCESS, CORE`
### 무엇이 바뀌었는가 (diff)
```diff
-static int run_single_command(t_shell *shell, const t_command *command,
+static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
     const struct exec_context *ctx)
 {
-    pid_t pid;
-    pid_t waited;
-    int wait_status;
+    pid_t           *pids;
+    const t_command *command;
+    size_t          i;
+    size_t          spawned;
+    int             result;
-    pid = fork();
-    if (pid < 0) {
-        fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+    pids = (pid_t *)calloc(pipeline->command_count, sizeof(pid_t));
+    if (pids == NULL) {
+        fprintf(stderr, "small-shell: allocation failure\n");
         return 1;
     }
-    if (pid == 0)
-        run_child(shell, command, ctx);
-    do {
-        waited = waitpid(pid, &wait_status, 0);
-    } while (waited < 0 && errno == EINTR);
-    if (waited != pid)
-        return 1;
-    return status_from_wait(wait_status);
+    command = pipeline->commands;
+    spawned = 0;
+    result = 1;
+    for (i = 0; i < pipeline->command_count && command != NULL; i++) {
+        pid_t pid;
+        pid = fork();
+        if (pid < 0) {
+            fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+            break;
+        }
+        if (pid == 0)
+            run_child(shell, command, ctx);
+        pids[i] = pid;
+        spawned++;
+        command = command->next;
+    }
+    for (i = 0; i < spawned; i++) {
+        int wait_status;
+        pid_t waited;
+        do {
+            waited = waitpid(pids[i], &wait_status, 0);
+        } while (waited < 0 && errno == EINTR);
+        if (waited == pids[i] && i + 1 == pipeline->command_count)
+            result = status_from_wait(wait_status);
+    }
+    free(pids);
+    return spawned == pipeline->command_count ? result : 1;
 }
```
### 무엇을 준비하는가, 아직 없는 것
단일 fork/wait를 "N개 명령을 순서대로 fork하고, PID 배열에 기록한 뒤 그 순서대로 wait"로
바꾼다. 하지만 이 자식들 사이에는 아직 **pipe가 없다** — 서로 독립적으로 stdin/stdout을
공유할 뿐, 실질적인 "파이프라인"은 아니다. `fork` 실패 시 이미 만든 자식들을 wait는 하지만,
아직 pipe가 없으므로 그 자식이 block될 위험 자체가 없다 — 이 안전성은 다음 커밋에서 pipe가
추가되는 순간 깨진다.
### 관련 커밋
`a71f98de0d92`가 이 PID 배열 위에 pipe wiring을 얹으면서 이 가정을 깨고, 그 문제를
`be2967a4b946`이 다시 고친다.
## a71f98de0d92 — feat(exec): 다단 pipeline의 pipe FD 연결
**중요도** `S` · **태그** `PROCESS, FD_IO, CORE`
### 무엇을 만들었는가 (diff)
```diff
+static void close_pipes(int (*pipes)[2], size_t pipe_count)
+{
+    size_t i;
+    if (pipes == NULL)
+        return;
+    for (i = 0; i < pipe_count; i++) {
+        if (pipes[i][0] >= 0)
+            close(pipes[i][0]);
+        if (pipes[i][1] >= 0)
+            close(pipes[i][1]);
+    }
+}
-static void run_child(t_shell *shell, const t_command *command,
-    const struct exec_context *ctx)
+static void run_child(t_shell *shell, const t_pipeline *pipeline, const t_command *command,
+    const struct exec_context *ctx, int (*pipes)[2], size_t pipe_count, size_t index)
 {
+    if (index > 0 && dup2(pipes[index - 1][0], STDIN_FILENO) < 0)
+        child_die("dup2");
+    if (index + 1 < pipeline->command_count && dup2(pipes[index][1], STDOUT_FILENO) < 0)
+        child_die("dup2");
+    close_pipes(pipes, pipe_count);
+
     if (exec_apply_redirections(command, ctx) != 0)
         _exit(1);
     /* ... */
 }
-static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
-    const struct exec_context *ctx)
+static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const struct exec_context *ctx)
 {
+    size_t pipe_count;
+    int (*pipes)[2];
     pid_t *pids;
     /* ... */
+    pipe_count = pipeline->command_count - 1;
+    if (pipe_count > 0) {
+        pipes = (int (*)[2])malloc(sizeof(int[2]) * pipe_count);
+        if (pipes == NULL)
+            goto alloc_error;
+        for (i = 0; i < pipe_count; i++) {
+            pipes[i][0] = -1;
+            pipes[i][1] = -1;
+        }
+        for (i = 0; i < pipe_count; i++) {
+            if (pipe(pipes[i]) < 0) {
+                fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
+                close_pipes(pipes, pipe_count);
+                free(pipes);
+                return 1;
+            }
+        }
+    }
     for (i = 0; i < pipeline->command_count && command != NULL; i++) {
         pid_t pid;
         pid = fork();
         if (pid < 0) {
             fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
             break;
         }
         if (pid == 0)
-            run_child(shell, command, ctx);
+            run_child(shell, pipeline, command, ctx, pipes, pipe_count, i);
         pids[i] = pid;
         spawned++;
         command = command->next;
     }
+    close_pipes(pipes, pipe_count);
     for (i = 0; i < spawned; i++) {
         /* ... wait ... */
     }
+    free(pipes);
     return spawned == pipeline->command_count ? result : 1;
 }
```
### 설계 결정 — 왜 "모든" pipe를 닫는가
N개 명령에 N-1개 pipe를 만들고, child `i`는 이전 pipe의 read end를 stdin에, 다음 pipe의
write end를 stdout에 `dup2`한다. dup2로 필요한 fd를 0/1에 복제하고 나면 원본 fd 번호는
더 이상 필요 없다. 만약 자식이 자신과 무관한 pipe(예: 3번째 자식 입장에서 1-2번째 사이
pipe)까지 열어둔 채로 두면, 그 pipe의 write end를 쥔 프로세스가 실제 필요 이상으로 많이
남아 하위 명령이 EOF를 받지 못하고 영원히 block될 수 있다. 그래서 `dup2` 직후
`close_pipes`가 pipe_count개 **전부**를 무조건 닫는다 — parent도 fork 루프가 끝나면
같은 이유로 자기 쪽 pipe end를 전부 닫는다.
### 이 커밋이 아직 다루지 않는 것
pipe 생성 도중 실패하면 이미 만든 pipe들을 정리하고 반환하지만, **fork 도중 실패하면
이미 살아있는 자식들은 어떻게 되는가?** 이 diff의 fork 루프는 실패 시 그냥 `break`한다.
`spawned == pipeline->command_count`가 거짓이면 `1`을 반환하긴 하지만, 이미 fork된
자식들에 대한 종료 처리는 없다 — 이들은 wait만 될 뿐 강제 종료되지 않는다. 만약 이들이
다음 자식이 없는 pipe에 쓰다가 block된 상태라면, 그 wait조차 끝나지 않을 수 있다. 이
위험이 `be2967a4b946`에서 고쳐진다.
### 정상 경로 최종 흐름
```text
[pipe_count = N-1개 pipe 생성 — 실패 시 이미 연 pipe 정리 후 즉시 반환]
  ↓
[command 순서대로 fork]
  child i:  dup2(pipes[i-1][0] → stdin)   # 이전 pipe의 read end
            dup2(pipes[i][1]   → stdout)  # 다음 pipe의 write end
            → close_pipes(전체)           # 필요 없어진 원본 fd 전부 반환
            → redirection 적용 → builtin 또는 exec
  parent:   pids[i] = child pid
  ↓ (fork 루프 종료)
[parent: close_pipes(전체) — 부모가 쥔 pipe end는 이제 하나도 없다]
  ↓
[spawned == command_count인 경우, 기록된 각 PID를 순서대로 wait]
  ↓
[마지막 stage의 종료 상태를 shell의 종료 상태로 반환]
```
### 관련 커밋
`ae988017efd5`의 PID 배열 위에 pipe wiring을 얹었다. fork 중간 실패의 처리는
`be2967a4b946`으로 이어진다.
## 915aa072298b — refactor(runtime): 프로세스 시스템 호출 경계 분리
**중요도** `A` · **태그** `PROCESS, FAILURE, TEST`
이 커밋은 앞뒤의 feat/fix와 성격이 다르다. **아무것도 고치지 않고, 아무 기능도 추가하지
않는다.** `pipe`/`fork`/`waitpid` 호출을 wrapper로 감싸서, 이후 커밋들이 새로운 능력을
얻게 하는 "가능하게 만들기(enabling)" 성격의 refactor다.
### 왜 필요한가
지금까지의 실패 처리 로직(`a71f98de0d92`의 pipe 실패, 앞으로 올 `be2967a4b946`의
mid-fork 실패)은 전부 "만약 실패하면"이라는 가정 위에 쓰여 있다. 하지만 정상적인 테스트
환경에서 정확히 3번째 fork만 실패시키는 건 재현 불가능한 커널/자원 상태에 의존하므로
사실상 불가능하다. 이 refactor는 이후의 회귀 테스트(`d611196b368e`)가 "N번째 호출만
결정적으로 실패시킨다"는 조건을 만들 수 있도록 seam(이음매)만 마련해 둔다.
### 무엇이 바뀌는가 (diff)
```diff
+int shell_pipe(int fds[2])
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_PIPE", &calls)) {
+        errno = EMFILE;
+        return -1;
+    }
+#endif
+    return pipe(fds);
+}
+
+pid_t shell_fork(void)
+{
+#ifdef SMALL_SHELL_TESTING
+    /* ... SMALL_SHELL_FAIL_FORK 확인, EAGAIN ... */
+#endif
+    return fork();
+}
+
+pid_t shell_waitpid(pid_t pid, int *status, int options)
+{
+#ifdef SMALL_SHELL_TESTING
+    /* ... SMALL_SHELL_FAIL_WAITPID 확인, EIO ... */
+#endif
+    return waitpid(pid, status, options);
+}
```
```diff
-        if (pipe(pipes[i]) < 0) {
+        if (shell_pipe(pipes[i]) < 0) {
...
-        pid = fork();
+        pid = shell_fork();
...
-            waited = waitpid(pids[i], &wait_status, 0);
+            waited = shell_waitpid(pids[i], &wait_status, 0);
```
`SMALL_SHELL_TESTING`이 정의되지 않은 프로덕션 빌드에서는 `shell_pipe`가 그냥 `pipe()`를
그대로 호출한다 — 동작 변화가 전혀 없다. 테스트 빌드에서만 환경 변수로 지정된 호출
순번에 도달하면 실패를 흉내 낸다(`fail_call`이 "N번째 호출인가"를 확인).
### 관련 커밋
이 seam이 있어야 `d611196b368e`가 실제로 pipe/fork/wait 실패를 결정적으로 재현할 수 있다.
## be2967a4b946 — fix(exec): 부분 생성 파이프라인의 자식과 FD 회수
**중요도** `S` · **태그** `PROCESS, FD_IO, FAILURE`
### 왜 바뀌었는가 (문제 → 원인 → 결정)
- **이전 동작**: fork 도중 실패하면 루프를 `break`하고, 이미 spawn된 자식들은 wait만 될
  뿐 강제 종료되지 않았다.
- **문제**: 이미 spawn된 자식이 다음 pipe의 write end에 쓰려는 중이었다면(다음 자식이
  존재하지 않으므로 아무도 그 pipe를 읽지 않는다), 그 자식은 커널 pipe buffer가 찰
  때까지 write에서 block된다. 부모가 그 자식을 wait해도, 자식이 block 상태에서 끝나지
  않으면 **shell 전체가 멈춘다.**
- **원인**: "PID를 기록했다"는 사실이 "그 프로세스를 끝까지 책임진다"는 의미로 이어지지
  않았다 — spawn 실패 시 이미 있는 자식을 그냥 방치했다.
- **결정**: `spawned != pipeline->command_count`(전체를 다 못 만들었다)면, 부모가 먼저
  자기 쪽 pipe end를 전부 닫아 block된 자식의 write가 풀릴 여지를 만들고,
  `terminate_children`으로 이미 만든 자식 전원에게 `SIGKILL`을 보낸 뒤, `wait_for_child`로
  EINTR을 견디며 회수를 시도한다.
### 무엇이 바뀌었는가 (diff)
```diff
+static void terminate_children(const pid_t *pids, size_t count)
+{
+    size_t i;
+    for (i = 0; i < count; i++) {
+        if (pids[i] > 0 && kill(pids[i], SIGKILL) < 0 && errno != ESRCH)
+            fprintf(stderr, "small-shell: kill: %s\n", strerror(errno));
+    }
+}
+static int wait_for_child(pid_t pid, int *wait_status)
+{
+    int attempts;
+    int had_error;
+    attempts = 0;
+    had_error = 0;
+    while (attempts < 2) {
+        pid_t waited;
+        waited = shell_waitpid(pid, wait_status, 0);
+        if (waited == pid)
+            return had_error;
+        if (waited < 0 && errno == EINTR)
+            continue;
+        fprintf(stderr, "small-shell: waitpid: %s\n", strerror(errno));
+        had_error = 1;
+        attempts++;
+    }
+    return -1;
+}
```
```diff
     close_pipes(pipes, pipe_count);
+    if (spawned != pipeline->command_count)
+        terminate_children(pids, spawned);
     for (i = 0; i < spawned; i++) {
         int wait_status;
-        pid_t waited;
+        int wait_result;
-        do {
-            waited = shell_waitpid(pids[i], &wait_status, 0);
-        } while (waited < 0 && errno == EINTR);
-        if (waited == pids[i] && i + 1 == pipeline->command_count)
+        wait_result = wait_for_child(pids[i], &wait_status);
+        if (wait_result == 0 && i + 1 == pipeline->command_count)
             result = status_from_wait(wait_status);
+        if (wait_result != 0)
+            wait_error = 1;
     }
+    if (wait_error)
+        result = 1;
```
### 실패 시 최종 흐름
```text
[fork 루프 도중 실패 → break]
  ↓
[close_pipes(전체) — 부모 쪽 pipe end 전부 반환]
  ↓
[spawned != command_count 이면]
  → terminate_children(pids[0..spawned)) : 전원 SIGKILL, ESRCH(이미 종료됨)는 무시
  → 각 pid를 wait_for_child로 회수 시도
      · EINTR은 재시도 횟수에 포함하지 않는다
      · EINTR이 아닌 실패는 최대 2회까지 재시도, had_error=1로 기록
      · 2회를 넘기면 포기하고 -1 반환
  ↓
[wait_result가 0이 아닌 경우(회수 중 문제가 있었던 경우)가 하나라도 있으면 result=1로 강제]
  ↓
[free(pids); free(pipes); return 1]
```
### 이 커밋이 보장하는 것 / 보장하지 않는 것
- 보장: fork가 도중에 실패해도, 이미 만든 자식 전원에게 종료 시도(SIGKILL)와 회수 시도가
  이루어진다 — 방치되는 자식이 없다.
- 보장하지 않음: `wait_for_child`가 EINTR이 아닌 이유로 waitpid에 2회 넘게 실패하면
  회수를 포기하고 `-1`을 반환한다 — 이 경우 그 자식은 실제로 회수되지 않은 채 남을 수
  있다. `SIGKILL`을 보냈다고 해서 커널이 즉시 그 자식을 D-state 등에서 꺼내준다는
  보장도 없다. 이런 잔여 위험은 이 fix로 완전히 제거되지 않는다.
### 관련 커밋
`915aa072298b`의 seam이 있어야 이 fix를 `d611196b368e`가 결정적으로 검증할 수 있다.
## d611196b368e — test(exec): pipe·fork·wait 실패 회귀 검증
**중요도** `A` · **태그** `TEST, PROCESS, FAILURE`
### 무엇을 검증하는가
`SMALL_SHELL_FAIL_PIPE=2`, `SMALL_SHELL_FAIL_FORK=2`, `SMALL_SHELL_FAIL_WAITPID=1`
환경 변수로 각각 "2번째 pipe 생성", "2번째 fork", "1번째 waitpid"를 결정적으로 실패시킨
뒤, 3-stage pipeline을 실행해 shell이 **hang하지 않고** 정확히 종료 코드 `1`을
반환하는지 확인한다.
```diff
+run_fault pipe_second SMALL_SHELL_FAIL_PIPE 2 \
+    'printf alpha | cat | cat
+echo $?' \
+    '1
+'
+
+run_fault fork_second SMALL_SHELL_FAIL_FORK 2 \
+    'sleep 30 | cat | cat
+echo $?' \
+    '1
+'
+
+run_fault waitpid_first SMALL_SHELL_FAIL_WAITPID 1 \
+    'printf alpha | cat > /dev/null
+echo $?' \
+    '1
+'
```
Makefile에는 `SMALL_SHELL_TESTING`을 정의해 컴파일하는 별도 테스트 바이너리
(`%.test.o`, `small-shell-test`)와 `test` 타깃에 `tests/faults.sh` 실행이 추가된다.
특히 `sleep 30 | cat | cat`에서 2번째 fork를 실패시키는 케이스가 핵심이다 — 이미 만든
1번째 자식(`sleep 30`)이 종료 처리되지 않으면 이 테스트 자체가 30초 이상 멈추거나 영원히
hang한다. **이 테스트가 정상적으로 짧은 시간 안에 통과한다는 사실 자체가
`be2967a4b946`의 종료 처리가 실제로 작동한다는 증거다.**
### 증명하는 것 / 증명하지 않는 것
이 세 가지 결정적 실패 지점에서 shell이 멈추지 않고 올바른 종료 코드를 낸다는 것을
증명한다. FD가 실제로 새지 않는지, zombie가 실제로 안 남는지는 직접 관찰하지 않는다 —
그건 `b42e57eb7755`의 몫이다.
### 관련 커밋
`915aa072298b`(seam)와 `be2967a4b946`(fix)이 없으면 이 테스트는 만들 수 없거나 hang한다.
## fd5c76c18c27 — refactor(runtime): FD 시스템 호출 경계 분리
**중요도** `A` · **태그** `FD_IO, FAILURE, TEST`
`915aa072298b`와 같은 "enabling refactor" 장르다. `dup`, `dup2`, `open`을
`shell_dup`/`shell_dup2`/`shell_open`으로 감싸 같은 방식의 결정적 실패 주입을 확장한다.
```diff
-            if (dup2(fd, STDIN_FILENO) < 0) {
+            if (shell_dup2(fd, STDIN_FILENO) < 0) {
...
-    saved[0] = dup(STDIN_FILENO);
-    saved[1] = dup(STDOUT_FILENO);
+    saved[0] = shell_dup(STDIN_FILENO);
+    saved[1] = shell_dup(STDOUT_FILENO);
```
```diff
 static int fail_call(const char *name, unsigned long *calls)
 {
     /* ... */
     target = strtoul(text, &end, 10);
-    return (end != text && *end == '\0' && target != 0 && target == *calls);
+    if (end == text || *end != '\0' || target == 0)
+        return 0;
+    {
+        char repeat_name[64];
+        size_t length;
+        length = strlen(name);
+        if (length + sizeof("_REPEAT") <= sizeof(repeat_name)) {
+            memcpy(repeat_name, name, length);
+            memcpy(repeat_name + length, "_REPEAT", sizeof("_REPEAT"));
+            if (getenv(repeat_name) != NULL)
+                return *calls >= target;
+        }
+    }
+    return target == *calls;
 }
```
### 눈여겨볼 확장
`fail_call`에 `_REPEAT` suffix 환경 변수가 추가됐다 — 기존에는 "N번째 호출만 실패"라는
단발성 실패만 표현할 수 있었는데, 이제 "N번째 호출부터 계속 실패"라는 **지속적** 실패
모드까지 표현할 수 있다. 이건 곧이어 올 `2ca9f4299c7f`가 "재시도하면 복구되는 일시적
실패"와 "재시도해도 복구 안 되는 영구적 실패"를 구분해야 하기 때문에 미리 준비된
확장이다.
### 관련 커밋
이 seam의 `_REPEAT` 모드가 `13645f58d5e6`의 persistent-restore 케이스에서 실제로
쓰인다.
## 2ca9f4299c7f — fix(redirection): 부모 표준 입출력 복원 실패 전파
**중요도** `A` · **태그** `FD_IO, FAILURE, RISK`
### 왜 바뀌었는가 (문제 → 결정)
parent에서 builtin을 실행하기 전 `save_stdio`로 원래 stdin/stdout을 저장하고, 실행 후
`restore_stdio`로 되돌리는데, 이전에는 이 복원이 실패해도 반환값이 없어(`void`) 그냥
무시됐다.
복원 실패는 성격이 다른 두 상황을 가릴 수 있다.
- **일시적 실패** (예: `dup2`가 `EINTR`로 한 번 실패) — 재시도하면 될 뿐 shell은 계속
  정상 동작할 수 있다.
- **영구적 실패** (예: fd 테이블이 실제로 고갈됨) — parent의 stdin/stdout이 리다이렉션된
  상태(예: 파일로) 그대로 남는다. 이후 모든 명령의 출력이 사용자 터미널이 아니라 엉뚱한
  파일로 계속 새어나간다 — 사용자는 자기 출력이 사라진 것도 모른 채 shell을 계속 쓰게
  된다. 단순 실패보다 위험한 상태다.
결정은 이 두 상황을 구분해서 처리하는 것이다 — 재시도로 복구되면 이번 명령만 실패
처리하고 shell은 계속 살리고, 재시도로도 복구가 안 되면 shell 자체를 중단시킨다.
### 최종 코드 (학습용 주석)
```c
static int restore_one(int saved, int target)
{
    int attempts;
    int had_error;
    attempts = 0;
    had_error = 0;
    while (attempts < 2) {
        // 성공하면 즉시 반환. had_error가 이미 1이었다면
        // "재시도 끝에 결국 복구됐다"는 사실을 호출자에게 남긴다.
        if (shell_dup2(saved, target) >= 0)
            return had_error;
        // EINTR은 재시도 횟수에 포함하지 않는다 — 순수 신호 인터럽트일 뿐 진짜 실패가 아니다.
        if (errno == EINTR)
            continue;
        // 진짜 실패는 최대 2번까지만 재시도한다. 2번을 넘기면 포기하고 -1.
        fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
        had_error = 1;
        attempts++;
    }
    return -1;
}
static int restore_stdio(int saved[2])
{
    int input_result;
    int output_result;
    // stdin과 stdout을 독립적으로 복원 시도한다 — 하나가 실패해도
    // 다른 하나는 시도한다. 부분 복원이라도 하지 않는 것보다 낫다.
    input_result = restore_one(saved[0], STDIN_FILENO);
    output_result = restore_one(saved[1], STDOUT_FILENO);
    close(saved[0]);
    close(saved[1]);
    // -1(영구 실패)이 하나라도 있으면 -1을 전파한다.
    // 그렇지 않으면 0(완전 정상) 또는 1(재시도 끝에 복구됨)을 반환한다.
    if (input_result < 0 || output_result < 0)
        return -1;
    return (input_result != 0 || output_result != 0);
}
```
```diff
     if (exec_apply_redirections(command, ctx) != 0) {
-        restore_stdio(saved);
+        if (restore_stdio(saved) < 0)
+            shell->running = 0;
         return 1;
     }
     /* ... */
-    fflush(stdout);
-    restore_stdio(saved);
+    if (fflush(stdout) == EOF)
+        status = 1;
+    {
+        int restore_result;
+        restore_result = restore_stdio(saved);
+        if (restore_result != 0)
+            status = 1;
+        if (restore_result < 0)
+            shell->running = 0;
+    }
     return status;
```
### 이 커밋이 보장하는 것 / 보장하지 않는 것
- 보장: parent의 stdin/stdout이 신뢰할 수 없는 상태로는 다음 명령을 실행하지 않는다.
- 보장하지 않음 (트레이드오프): 최대 2번 재시도 이상 지속되는 실패에서는 shell 자체를
  멈춘다 — "출력이 조용히 새는 것"보다 "shell이 죽는 것"을 선택한 결정이며, 이 선택이
  모든 상황에서 옳다는 것을 이 커밋이 증명하지는 않는다.
### 관련 커밋
`fd5c76c18c27`의 `_REPEAT` seam이 있어야 이 두 상태(일시적/영구적)를 `13645f58d5e6`이
결정적으로 재현할 수 있다.
## 13645f58d5e6 — test(redirection): 저장·적용·복원 실패 회귀 검증
**중요도** `A` · **태그** `TEST, FD_IO, FAILURE`
### 무엇을 검증하는가
`save_stdio`의 두 `dup` 호출, `exec_apply_redirections`의 `dup2`, `restore_stdio`의 두
`dup2`, `open` 각각을 독립적으로 실패시켜 — 매 경우 명령 자체는 실패(종료 코드 1)하지만
**다음 줄 명령(`echo after`)은 정상 실행된다**는 것을 확인한다. "이번 명령 실패"와
"shell 전체 중단"이 다르다는 걸 다섯 개 케이스마다 증명한다.
```diff
+run_fault save_stdin SMALL_SHELL_FAIL_DUP 1 \
+    "echo hidden > $TMP/save-stdin
+echo \$?
+echo after" \
+    '1
+after
+'
+# ... save_stdout / apply_stdout / restore_stdin / restore_stdout / open_output 동일 패턴
```
마지막 케이스는 성격이 다르다 — 복원용 dup2가 2번째 호출부터 **계속** 실패하도록 만들어,
재시도로도 복구 안 되는 영구적 실패를 재현한다.
```diff
+env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
+    "$BIN" >"$TMP/persistent-restore.out" \
+    2>"$TMP/persistent-restore.err" <<EOF
+echo hidden > $TMP/persistent-restore-file
+echo never
+EOF
+status=$?
+[ "$status" -eq 1 ] || fail persistent-restore
+[ ! -s "$TMP/persistent-restore.out" ] || fail persistent-restore
+grep -q 'dup2' "$TMP/persistent-restore.err" || fail persistent-restore
```
`echo never`가 stdout에 전혀 나타나지 않는지(다음 명령이 실행되지 않았는지) 확인한다 —
`2ca9f4299c7f`의 `running = 0` 분기를 직접 겨냥하는 유일한 케이스다.
### 증명하는 것 / 증명하지 않는 것
5개의 일시적 실패 케이스는 "복구되고 계속 실행된다"를, 1개의 영구적 실패 케이스는
"shell이 멈춘다"를 각각 증명한다. 재시도 횟수(2번)가 실제 운영 환경에서 적절한 값인지는
이 테스트가 증명하지 않는다.
### 관련 커밋
`2ca9f4299c7f`(fix)를 검증하고, `fd5c76c18c27`의 `_REPEAT` 확장을 실제로 사용하는
첫 사례다.
## b42e57eb7755 — test(lifecycle): FD와 자식 프로세스 누수 검증
**중요도** `A` · **태그** `TEST, PROCESS, FD_IO`
### 왜 다른 기법이 필요한가
지금까지의 테스트는 전부 "종료 코드가 맞는가"만 확인했다 — 이건 간접 증거다. 종료 코드가
맞아도 FD나 자식 프로세스가 실제로 조금씩 새고 있을 수 있다(예: 매번 하나씩 새는 정도로는
종료 코드에 영향을 주지 않는다). 이 커밋은 두 가지 직접 관찰 기법을 추가한다.
**1. FD 고갈 스트레스**
```diff
+(
+    ulimit -n 48
+    SMALL_SHELL_CHECK_CHILDREN=1 "$TIMEOUT" 20 "$BIN" \
+        <"$TMP/fd-pressure.in" >"$TMP/fd-pressure.out" \
+        2>"$TMP/fd-pressure.err"
+)
```
`ulimit -n 48`로 열 수 있는 FD 수를 인위적으로 낮춘 뒤, pipeline을 60번 반복 실행한다.
매 반복마다 FD가 하나라도 새면 곧 48개 한도에 도달해 `open`/`pipe`가 실제로 실패하기
시작한다 — **60번을 전부 통과한다는 사실 자체가 FD 누수가 없다는 증거**다.
추가로 `SMALL_SHELL_CHECK_CHILDREN=1`이 활성화하는 검사가 있다.
```diff
+#ifdef SMALL_SHELL_TESTING
+    if (getenv("SMALL_SHELL_CHECK_CHILDREN") != NULL
+        && !shell_children_reaped()) {
+        fprintf(stderr, "small-shell: unreaped child process\n");
+        result = 1;
+    }
+#endif
```
```diff
+int shell_children_reaped(void)
+{
+    int     status;
+    int     found;
+    pid_t   pid;
+    found = 0;
+    for (;;) {
+        pid = waitpid(-1, &status, WNOHANG);
+        if (pid > 0) {
+            found = 1;
+            continue;
+        }
+        if (pid == 0)
+            return 0;
+        if (errno == EINTR)
+            continue;
+        if (errno == ECHILD)
+            return !found;
+        return 0;
+    }
+}
```
매 pipeline 실행 직후 `waitpid(-1, WNOHANG)`을 반복 호출해 아직 회수되지 않은 자식이
남아있는지 직접 확인한다.
**2. 외부 시그널로 강제 종료된 경우**
```diff
+"$TIMEOUT" 20 /bin/sh -c \
+    'printf "%s\n" "$$" >"$1"; exec sleep 30' \
+    timeout-child "$TMP/child.pid" &
+runner_pid=$!
+...
+kill -TERM "$runner_pid"
+...
+kill -0 "$child_pid" 2>/dev/null
+alive=$?
+[ "$alive" -ne 0 ] || fail external-signal
```
이 블록은 small-shell의 executor를 직접 검증하지 않는다. `tests/timeout_runner.c`를
확인하면 harness가 child를 새 process group leader로 만들고, 자신이 `SIGHUP`, `SIGINT`,
`SIGTERM`을 받거나 deadline을 넘기면 `kill(-pid, SIGKILL)`로 group 전체를 종료한 뒤 leader를
`waitpid`로 회수한다. 위 fixture는 runner에 `SIGTERM`을 보냈을 때 status가 `143`이 되고,
그 아래 `sleep` PID도 더 이상 존재하지 않는지 확인한다.

따라서 이 case의 직접 대상은 **timeout harness의 signal propagation과 process-group cleanup**이다.
바로 앞의 `sleep 30 | cat | cat` timeout case가 small-shell과 그 pipeline descendants를 시험한 뒤
orphan을 남기지 않았다고 신뢰하려면, test harness 자체가 descendant group을 실제로 정리한다는
증거가 필요하다. 이 fixture는 그 검증 기반을 제공하며 small-shell의 자체 signal handler나
executor cleanup을 독립적으로 증명하지는 않는다.
### 증명하는 것 / 증명하지 않는 것
FD-pressure case는 이 정도 규모(60회 반복, FD 48개 한도)에서 누수가 누적되지 않고 direct child가
pipeline 반환 뒤 남지 않는다는 것을 본다. Timeout case는 long-running pipeline을 harness가
process group 단위로 끊을 수 있는지, external-signal fixture는 그 harness가 자기 child group을
실제로 종료·회수하는지를 분리해 검증한다.

무한히 오래 실행되는 세션의 누적 누수, grandchildren을 포함한 모든 descendant 형태, 이 Thread가
다루지 않는 다른 코드 경로의 resource 사용, small-shell 자체의 일반 signal policy는 증명 범위
밖이다.
### 관련 커밋
`be2967a4b946`(fork 실패 시 회수)과 `a71f98de0d92`(정상 close 규칙) 둘 다 이 스트레스
테스트를 통과해야 하는 대상이다.
## 6dff1ba86ba6 — fix(exec): pipe 생성 실패 시 PID 배열 해제
**중요도** `B` · **태그** `FD_IO, FAILURE, DEBUG`
### 무엇이 바뀌었는가 (diff)
```diff
                 fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
                 close_pipes(pipes, pipe_count);
                 free(pipes);
+                free(pids);
                 return 1;
```
### 왜 이렇게 작은가
`run_forked_pipeline`에서 pipe 생성이 도중에 실패하면 `pipes` 배열은 정리
(`close_pipes` + `free`)했지만, 그보다 먼저 `calloc`한 `pids` 배열은 해제하지 않고 그대로
반환하고 있었다. `be2967a4b946`이 고친 "이미 만든 자식"의 문제와는 다른, 더 이른 시점 —
자식을 하나도 만들기 전, pipe 생성 단계 — 의 순수한 heap 메모리 누수다. 커널 자원(FD,
프로세스)이 아니라 메모리 누수이고 크래시나 hang을 유발하지 않으므로, 중요도가 `S`나
`A`가 아니라 `B`로 매겨진 것으로 보인다.
### 관련 커밋
`a71f98de0d92`가 만든 "pipe 테이블 먼저 할당, PID 테이블 그다음" 순서가 이 누수 지점을
만들었다.
## 이 Thread의 경계
이 Thread는 **파이프라인의 프로세스(PID)와 descriptor(FD)를, 이미 획득한 뒤 실패가
나더라도 끝까지 회수한다**는 문제만 다룬다. 인접한 관심사 중 아래는 이 Thread에 속하지
않는다.
- 명령을 파싱해 `t_pipeline`을 만드는 과정과 실행 여부 판단 —
  `01-parsed-representation-to-conditional-execution`의 몫이다. 이 Thread는 이미 유효한
  `t_pipeline`이 주어졌다고 가정한다.
- heredoc 자체의 다단계 확장·타이밍 의미론 — `02-heredoc-cross-stage-semantics`의 몫이다.
  이 Thread에서 다루는 리다이렉션은 이미 만들어진 heredoc 임시 파일의 fd를 `dup2`하는
  지점(`fd5c76c18c27`)뿐이며, 그 임시 파일이 언제 어떻게 채워지는지는 다루지 않는다.
- pipe/PID 테이블 밖에서 일어나는 일반적인 할당 실패 정책(예: word expansion, 환경
  변수 복제) — `04-transactional-allocation-failure`의 몫이다. 이 Thread는 pipe/PID
  테이블에 한정된 할당 실패(`be2967a4b946`, `6dff1ba86ba6`)만 다룬다.
- 문자열 구성 자체의 안전성 — `05-asymptotically-safe-text-construction`의 몫이며 이
  Thread의 프로세스/FD 소유권과는 무관하다.
- `SIGKILL`을 무시할 수 있는 커널 상태(D-state 등)나, 무한정 오래 실행되는 세션에서의
  누적 자원 문제 — 이 Thread의 테스트 규모(최대 60회 반복) 밖의 문제로 남겨둔다.
### 검증 범위

표시된 각 SHA의 diff와 해당 시점 source·test script를 `c/minishell` branch에서 확인했다.
Repository를 로컬 checkout할 수 없어 fault/lifecycle suite를 다시 실행하지 않았으며, test 통과를
새로 주장하지 않는다.
