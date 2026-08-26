# Thread: signal handler는 사실만 기록하고 event loop가 protocol state를 소유한다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 03

## 개요

초기 server의 signal handler는 sender 확인, session 검사, bit 조립, stdout 출력, sequence 증가와 ACK 생성까지 protocol 전체를 직접 수행했다. 이 구조에서는 authoritative state가 async signal context 안에서 바뀌고, 호출 가능한 operation도 async-signal-safe 범위에 묶인다.

이 Thread는 `(sender PID, signal number)`를 고정 크기 event로 표현한 뒤 handler의 책임을 self-pipe write 하나로 줄이고, `pselect` event loop가 session·bit·output·ACK를 전담하도록 옮긴다. event 하나의 손실은 bit 하나의 손실이므로 queue overflow를 부분 복구하지 않고 fail-stop으로 다룬다. 마지막 두 커밋은 termination signal과 inherited signal mask를 같은 event architecture에 포함한다.

### 최종 책임 분리

| 실행 위치 | 책임 |
| --- | --- |
| signal handler | `errno` 보존, sender/signal을 `t_bit_event`에 복사, nonblocking self-pipe write, 실패 시 overflow flag 설정 |
| event loop | full event read, event validation, owner 검사, byte/sequence mutation, stdout 반영, ACK 전송, termination 처리 |
| process exit path | overflow·I/O failure는 status 1, supported termination은 `128 + signal`, `atexit` cleanup으로 socket과 pipe close/unlink |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a7d994b0a4b6` | refactor(server): 비트 상태 전이 로직 추출 | B | `REFACTOR, SELF_PIPE` | handler body의 bit transition을 explicit event 인자를 받는 함수로 추출한다. |
| 2 | `22363f83ff25` | refactor(server): signal 처리를 self-pipe event loop로 제한 | S | `ARCH, SELF_PIPE, RISK` | handler를 fixed event write로 제한하고 normal loop에 authoritative state mutation을 이전한다. |
| 3 | `4c535ac8657e` | test(server): self-pipe 이벤트 손실 시 fail-stop 검증 | A | `TEST, SELF_PIPE, RISK` | 첫 event write에 `EAGAIN`을 주입해 overflow가 server failure와 cleanup으로 이어지는지 검증한다. |
| 4 | `e304c63bee3e` | fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리 | A | `SELF_PIPE, PROCESS_LIFECYCLE` | `SIGHUP`, `SIGINT`, `SIGTERM`을 self-pipe event로 전달해 normal cleanup 뒤 종료한다. |
| 5 | `686b0d2a14e3` | fix(server): 상속된 이벤트 시그널 마스크 해제 | A | `PROCESS_LIFECYCLE, RISK` | handler 설치 뒤 data·termination signal을 process mask에서 명시적으로 unblock한다. |
| 6 | `72424469474c` | test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증 | B | `TEST, PROCESS_LIFECYCLE` | blocked mask를 상속한 `exec` 뒤 message delivery와 SIGTERM cleanup을 검증한다. |

---

**역할군: state transition을 event value로 분리한 뒤 실행 context를 이동하기**

## a7d994b0a4b6 — refactor(server): 비트 상태 전이 로직 추출

**중요도** `B` · **태그** `REFACTOR, SELF_PIPE`

이 커밋은 self-pipe를 완성하지 않는다. handler가 받던 `siginfo_t`에서 protocol input을 분리해, `22363f83ff25`가 caller를 normal loop로 옮길 수 있게 만드는 enabling refactor다.

### 무엇이 바뀌었는가 (diff)

```diff
+typedef struct s_bit_event
+{
+    pid_t sender;
+    int   signal;
+} t_bit_event;
+
+typedef char t_event_must_fit_pipe_buf[
+    (sizeof(t_bit_event) <= PIPE_BUF) * 2 - 1];
@@
-static void handle_bit(int signal, siginfo_t *info, void *context)
+static void process_bit(const t_bit_event *event)
 {
-    if (info == NULL || info->si_pid <= 0)
+    if (event->sender <= 0)
         return ;
-    if (g_client_pid == 0 || g_client_pid != info->si_pid)
+    if (g_client_pid == 0 || g_client_pid != event->sender)
         return ;
@@
-    if (signal == MT_ONE_SIGNAL)
+    if (event->signal == MT_ONE_SIGNAL)
         g_current_byte |= 1;
@@
-    queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
+    queue_response(event->sender, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
 }
```

handler는 event를 만들어 `process_bit`을 직접 호출한다. 따라서 authorization, output와 sequence mutation은 여전히 async signal context에서 실행된다. compile-time check는 record가 `PIPE_BUF` 이하라는 다음 단계의 atomic-write 전제를 미리 고정할 뿐, 이 SHA에서 pipe write를 수행하지는 않는다.

### production semantics가 유지되는 근거는 무엇인가

sender와 signal을 local event로 복사한 뒤 동일한 transition function을 같은 handler context에서 호출한다. state mutation 순서와 response 생성은 이동하지 않았고 입력 표현만 바뀌었다.

### 관련 커밋과 어떤 관계인가

`22363f83ff25`가 handler의 `process_bit` 호출을 self-pipe write로 교체하고, 동일한 event를 normal loop에서 소비한다. 이 refactor가 없으면 responsibility 이동과 transition logic 변경이 한 diff에 섞인다.

## 22363f83ff25 — refactor(server): signal 처리를 self-pipe event loop로 제한

**중요도** `S` · **태그** `ARCH, SELF_PIPE, RISK`

### 무엇이 바뀌었는가 (diff)

handler는 state를 직접 바꾸지 않고 fixed event를 nonblocking write한다.

```diff
 static void handle_bit(int signal, siginfo_t *info, void *context)
 {
     t_bit_event event;
     int         saved_errno;
@@
     if (info != NULL)
         event.sender = info->si_pid;
     event.signal = signal;
-    process_bit(&event);
+    if (write(g_event_pipe[1], &event, sizeof(event))
+        != (ssize_t)sizeof(event))
+        g_event_overflow = 1;
     errno = saved_errno;
 }
```

pipe의 read end는 `FD_CLOEXEC`, write end는 `O_NONBLOCK | FD_CLOEXEC`로 설정한다. event loop는 response socket과 event-pipe read end를 함께 `pselect`하고, full record를 읽은 뒤 `process_bit`을 호출한다.

```diff
+static int read_event(t_bit_event *event)
+{
+    unsigned char *bytes;
+    size_t         offset;
+    ssize_t        size;
+
+    bytes = (unsigned char *)event;
+    offset = 0;
+    while (offset < sizeof(*event))
+    {
+        size = read(g_event_pipe[0], bytes + offset,
+                sizeof(*event) - offset);
+        if (size == -1 && errno == EINTR)
+            continue ;
+        if (size <= 0)
+            return (-1);
+        offset += (size_t)size;
+    }
+    return (0);
+}
```

### 왜 이것이 단순한 pipe 추가가 아닌가

이전의 `volatile sig_atomic_t` protocol fields는 handler와 normal context가 공유하기 때문에 선택된 타입이었다. mutation owner가 event loop 하나로 수렴한 뒤 `g_current_byte`, `g_received_bits`, `g_client_pid`, `g_sequence`는 normal C types로 돌아간다. handler가 접근하는 mutable global은 pipe descriptor와 `g_event_overflow`뿐이다.

normal-loop 순서는 다음과 같다.

```text
full event read
  → sender/signal validation
  → current owner 확인
  → byte와 bit count mutation
  → complete byte면 stdout 반영
  → sequence 증가
  → exact sequence ACK 전송
```

### 왜 event 손실을 복구하지 않는가

standard signal로 전달된 bit 하나와 pipe event 하나가 1:1이다. handler write가 partial 또는 `EAGAIN`으로 실패하면 event stream의 어느 위치가 사라졌는지 normal loop는 복원할 수 없다. 계속 실행하면 이후 bit를 잘못된 byte 위치에 넣고도 ACK할 수 있으므로, overflow flag를 관찰한 event loop는 `ENOBUFS`와 failure로 종료한다.

### 관련 커밋과 어떤 관계인가

`4c535ac8657e`는 실제 pipe saturation을 기다리지 않고 첫 write failure를 deterministic하게 주입해 fail-stop branch를 고정한다. `e304c63bee3e`는 같은 event channel에 termination signal을 추가한다.

## 4c535ac8657e — test(server): self-pipe 이벤트 손실 시 fail-stop 검증

**중요도** `A` · **태그** `TEST, SELF_PIPE, RISK`

### 무엇을 검증하는가

production handler call site만 macro seam으로 바꾸고, fault build에서 첫 event write가 `EAGAIN`을 반환하도록 한다.

```diff
+#ifndef MT_EVENT_WRITE
+# define MT_EVENT_WRITE write
+#endif
@@
-    if (write(g_event_pipe[1], &event, sizeof(event))
+    if (MT_EVENT_WRITE(g_event_pipe[1], &event, sizeof(event))
         != (ssize_t)sizeof(event))
         g_event_overflow = 1;
```

```diff
+ssize_t mt_test_event_write(int fd, const void *buffer, size_t size)
+{
+    static int failed;
+
+    if (!failed && read_number("MT_TEST_EVENT_EAGAIN", 0) == 1)
+    {
+        failed = 1;
+        errno = EAGAIN;
+        return (-1);
+    }
+    return (write(fd, buffer, size));
+}
```

shell fixture는 client가 실패하고, server도 nonzero로 종료하며, stderr가 `server: signal event channel failed`를 포함하고, server socket path가 cleanup됐는지 요구한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

첫 event write failure가 silent bit loss로 이어지지 않고 fail-stop과 endpoint cleanup으로 수렴한다는 selected branch를 증명한다. 실제 kernel pipe를 가득 채우거나 partial positive write를 재현하지는 않는다. macro seam이 production build에서 raw `write`로 해석된다는 점은 유지된다.

### 관련 커밋과 어떤 관계인가

`22363f83ff25`가 정의한 “event 하나라도 잃으면 계속하지 않는다”는 정책을 직접 고정한다. output write fault와 같은 다른 failure seam은 `04-output-commit-boundary.md`의 테스트가 맡는다.

---

**역할군: lifecycle signal도 event loop와 cleanup 경로에 통합하기**

## e304c63bee3e — fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리

**중요도** `A` · **태그** `SELF_PIPE, PROCESS_LIFECYCLE`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

기본 signal disposition으로 process가 즉시 종료되면 normal loop가 종료 이유를 관찰하지 못한다. `atexit` cleanup과 shell-compatible exit status를 같은 path에서 보장하기 위해 supported termination signal도 `t_bit_event.signal`로 전달한다.

```diff
     if (sigaction(MT_ZERO_SIGNAL, &action, NULL) == -1
-        || sigaction(MT_ONE_SIGNAL, &action, NULL) == -1)
+        || sigaction(MT_ONE_SIGNAL, &action, NULL) == -1
+        || sigaction(SIGHUP, &action, NULL) == -1
+        || sigaction(SIGINT, &action, NULL) == -1
+        || sigaction(SIGTERM, &action, NULL) == -1)
         return (-1);
@@
+    if (event->signal == SIGHUP || event->signal == SIGINT
+        || event->signal == SIGTERM)
+        return (event->signal);
@@
-    if (run_event_loop() == -1)
+    status = run_event_loop();
+    if (status == -1)
         return (1);
+    if (status > 0)
+        return (128 + status);
```

handler는 data signal과 같은 방식으로 termination event를 pipe에 넣는다. event loop가 positive signal number를 반환하면 `main`이 `128 + signal`로 종료하고, normal return 과정에서 registered cleanup이 pipe descriptors와 response socket을 정리한다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

등록된 세 termination signal은 protocol state mutation과 같은 serialized input path를 통과한다. `SIGKILL`, `SIGSTOP`처럼 catch할 수 없는 signal이나 모든 possible signal을 일반화하지 않는다.

### 관련 커밋과 어떤 관계인가

handler disposition만 설치해도 inherited process mask가 signal을 계속 block할 수 있다. `686b0d2a14e3`가 startup 시 deliverability까지 복구한다.

## 686b0d2a14e3 — fix(server): 상속된 이벤트 시그널 마스크 해제

**중요도** `A` · **태그** `PROCESS_LIFECYCLE, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
+static int unblock_event_signals(void)
+{
+    sigset_t event_signals;
+
+    sigemptyset(&event_signals);
+    sigaddset(&event_signals, MT_ZERO_SIGNAL);
+    sigaddset(&event_signals, MT_ONE_SIGNAL);
+    sigaddset(&event_signals, SIGHUP);
+    sigaddset(&event_signals, SIGINT);
+    sigaddset(&event_signals, SIGTERM);
+    return (sigprocmask(SIG_UNBLOCK, &event_signals, NULL));
+}
@@
-    if (install_signal_handlers() == -1)
+    if (install_signal_handlers() == -1 || unblock_event_signals() == -1)
```

### 왜 이렇게 작은가

`exec`는 signal disposition 일부를 초기화하지만 process signal mask는 상속한다. parent가 data signal이나 SIGTERM을 block한 채 server를 `exec`하면 handler가 올바르게 설치돼도 signal은 pending 상태에 머물 수 있다. fix는 handler 설치 뒤 정확히 event channel이 소비하는 다섯 signal만 unblock한다.

### 관련 커밋과 어떤 관계인가

`72424469474c`가 wrapper에서 이 다섯 signal을 block한 뒤 server를 `exec`해, disposition과 mask가 서로 다른 startup 책임임을 regression으로 고정한다.

## 72424469474c — test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증

**중요도** `B` · **태그** `TEST, PROCESS_LIFECYCLE`

### 무엇을 검증하는가

`tests/masked_exec`가 data와 termination signal을 모두 block한 뒤 child에서 server를 `exec`한다.

```diff
     sigaddset(&blocked, MT_ZERO_SIGNAL);
     sigaddset(&blocked, MT_ONE_SIGNAL);
+    sigaddset(&blocked, SIGHUP);
+    sigaddset(&blocked, SIGINT);
+    sigaddset(&blocked, SIGTERM);
     if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
         return (1);
```

shell fixture는 normal client가 `inherited mask`를 전송해 stdout에 정확한 message가 나타나는지 확인한 뒤 server에 SIGTERM을 보낸다. wrapper의 wait status가 143이고 server socket path가 사라져야 한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

inherited blocked mask에서 startup fix가 data delivery와 supported termination cleanup을 모두 복구한다는 selected integration path를 검증한다. arbitrary parent dispositions, real-time signals, fork 없이 mask가 런타임 중 다시 바뀌는 상황은 범위 밖이다.

### 관련 커밋과 어떤 관계인가

`686b0d2a14e3`의 `unblock_event_signals`가 없다면 client data와 SIGTERM 모두 handler에 전달되지 않는다. `e304c63bee3e`의 termination event path까지 함께 통과해야 expected status와 cleanup이 성립한다.

## 이 Thread의 경계

- session owner를 예약하고 unavailable owner를 회수하는 정책은 `02-session-ownership-and-recovery.md`가 다룬다. event loop는 그 정책의 실행 context일 뿐이다.
- stdout write failure를 ACK보다 먼저 관찰하는 commit boundary는 `04-output-commit-boundary.md`의 책임이다.
- event loop가 함께 polling하는 Unix socket의 path·bind·cleanup과 `FD_SETSIZE` guard는 `05-endpoint-ownership-and-bounded-polling.md`가 다룬다.
- response record의 source·token·deadline 검증은 `06-bounded-response-correlation.md`가 다룬다.
- catch할 수 없는 signal, multi-threaded signal routing과 kernel-level signal queue semantics 일반론은 별개의 문제다.

> 검토 범위: `a7d994b0a4b6`, `22363f83ff25`, `4c535ac8657e`, `e304c63bee3e`, `686b0d2a14e3`, `72424469474c`의 diff와 해당 SHA의 `src/server.c`, fault seam, inherited-mask test source를 확인했다. branch checkout과 test 실행은 수행하지 않았으므로 shell fixture의 assertion을 실제 통과 결과로 주장하지 않는다.
