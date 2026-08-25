# Thread: signal handler에서 fail-stop self-pipe event loop로

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 03

## 개요

초기 server의 signal handler는 sender 확인, session 검사, bit 조립, byte 출력, sequence 증가, ACK 생성까지 protocol 전체를 직접 수행했다. 이 구조에서는 normal code와 handler가 같은 state를 공유하고, handler 안에서 호출할 수 있는 함수도 async-signal-safe 범위에 묶인다.

이 Thread의 핵심은 “pipe를 하나 추가했다”가 아니다. **authoritative state mutation을 normal execution context 하나로 옮겼다**는 데 있다. handler는 `(sender PID, signal number)`라는 사실만 고정 크기 record로 남기고, `pselect` event loop가 그 record를 읽어 session·decoder·stdout·ACK 상태를 순서대로 바꾼다.

### 최종 불변 조건

- data/termination signal handler는 `errno`를 보존하고 self-pipe에 fixed event 하나를 쓰는 일만 한다.
- `g_client_pid`, partial byte, bit count, line state, sequence는 event loop에서만 변경된다.
- event record는 `PIPE_BUF` 이하이며 한 signal의 sender와 signal 값이 서로 다른 event와 섞이지 않는다.
- write end는 nonblocking이다. record를 완전히 기록하지 못하면 event loss로 간주하고 server가 중단된다.
- inherited process mask와 무관하게 server가 사용하는 data·termination signal은 handler 설치 뒤 명시적으로 unblock된다.
- `SIGHUP`, `SIGINT`, `SIGTERM`도 self-pipe를 거쳐 normal cleanup path로 종료된다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a7d994b0a4b6` | refactor(server): 비트 상태 전이 로직 추출 | B | `REFACTOR, SELF_PIPE` | bit processing을 explicit `(sender, signal)` event를 받는 function으로 추출하고 event size가 `PIPE_BUF` 안인지 compile-time으로 확인합니다. |
| 2 | `22363f83ff25` | refactor(server): signal 처리를 self-pipe event loop로 제한 | S | `ARCH, SELF_PIPE, RISK` | handler를 `errno` preservation과 fixed event self-pipe write로 제한하고 `pselect` loop가 모든 protocol work를 수행하게 합니다. |
| 3 | `4c535ac8657e` | test(server): self-pipe 이벤트 손실 시 fail-stop 검증 | A | `TEST, SELF_PIPE, RISK` | handler의 first self-pipe event write에 `EAGAIN`을 주입하고 server fail-stop을 검증합니다. |
| 4 | `e304c63bee3e` | fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리 | A | `SELF_PIPE, PROCESS_LIFECYCLE` | `SIGHUP`, `SIGINT`, `SIGTERM`을 data signal과 같은 self-pipe event로 전달하고 normal cleanup path에서 종료합니다. |
| 5 | `686b0d2a14e3` | fix(server): 상속된 이벤트 시그널 마스크 해제 | A | `PROCESS_LIFECYCLE, RISK` | handler 설치 후 two data signals와 three supported termination signals를 process mask에서 명시적으로 unblock합니다. |
| 6 | `72424469474c` | test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증 | B | `TEST, PROCESS_LIFECYCLE` | wrapper가 data와 termination signals를 block한 뒤 server를 `exec`하고 message delivery와 SIGTERM cleanup을 검증합니다. |

## `a7d994b0a4b6` — refactor(server): 비트 상태 전이 로직 추출

**중요도** `B` · **태그** `REFACTOR, SELF_PIPE`

### event representation을 먼저 만든 중간 단계

`siginfo_t`와 signal handler에 묶여 있던 protocol code를 `(sender, signal)`만 받는 `process_bit`으로 옮긴다.

```c
typedef struct s_bit_event
{
    pid_t  sender;
    int    signal;
}   t_bit_event;

typedef char t_event_must_fit_pipe_buf[
    (sizeof(t_bit_event) <= PIPE_BUF) * 2 - 1];
```

compile-time array 크기는 조건이 참이면 1, 거짓이면 -1이 되어 build를 실패시킨다. 앞으로 이 record를 pipe 한 번의 write로 넘길 수 있다는 전제를 type 정의 근처에서 고정한다.

handler는 `siginfo_t`에서 event를 만든다.

```c
event.sender = 0;
if (info != NULL)
    event.sender = info->si_pid;
event.signal = signal;
process_bit(&event);
```

그러나 이 SHA에서는 handler가 `process_bit`을 **직접 호출한다**. 추출된 함수 안에서는 여전히 다음 작업이 handler context에서 실행된다.

- current owner 검사
- partial byte shift와 bit count 증가
- 완성 byte 출력 및 session reset
- sequence 증가
- response work queueing

따라서 이 commit은 async-signal boundary를 완성한 것이 아니라, 다음 commit이 state transition을 event loop로 옮길 수 있도록 입력 표현을 정리한 준비 작업이다.

## `22363f83ff25` — refactor(server): signal 처리를 self-pipe event loop로 제한

**중요도** `S` · **태그** `ARCH, SELF_PIPE, RISK`

### 책임 이동

이전과 이후의 차이는 함수 위치가 아니라 실행 context다.

| 작업 | 변경 전 | 변경 후 |
| --- | --- | --- |
| sender/signal snapshot | handler | handler |
| owner 검사 | handler가 호출한 `process_bit` | event loop의 `process_bit` |
| byte/bit/sequence mutation | handler | event loop |
| stdout write | handler | event loop |
| ACK datagram 전송 | response loop | event loop |
| event loss 판단 | response pipe overflow | event pipe overflow, fail-stop |

handler에는 다음 코드만 남는다.

```c
static void handle_bit(int signal, siginfo_t *info, void *context)
{
    t_bit_event event;
    int         saved_errno;

    saved_errno = errno;
    (void)context;
    event.sender = 0;
    if (info != NULL)
        event.sender = info->si_pid;
    event.signal = signal;
    if (write(g_event_pipe[1], &event, sizeof(event))
        != (ssize_t)sizeof(event))
        g_event_overflow = 1;
    errno = saved_errno;
}
```

`write`는 async-signal-safe 함수이고 event 크기는 `PIPE_BUF` 이하이다. write end는 `O_NONBLOCK | FD_CLOEXEC`, read end는 `FD_CLOEXEC`로 설정된다. handler가 pipe capacity를 기다리며 block하지 않도록 하고, exec된 다른 program이 내부 channel을 상속하지 않게 한다.

### state를 normal type으로 되돌린 이유

이전에는 handler가 직접 읽고 쓰므로 여러 field가 `volatile sig_atomic_t`였다. 이동 후에는 handler와 event loop가 공유하는 값이 `g_event_overflow` 하나뿐이다.

```diff
-static volatile sig_atomic_t g_current_byte;
-static volatile sig_atomic_t g_received_bits;
-static volatile sig_atomic_t g_client_pid;
+static unsigned char          g_current_byte;
+static unsigned int           g_received_bits;
+static pid_t                  g_client_pid;
 ...
+static volatile sig_atomic_t  g_event_overflow;
```

이 변화는 단순 type cleanup이 아니다. protocol state의 writer가 event loop 하나로 수렴했다는 코드 증거다.

### self-pipe 소비 경로

`read_event`는 fixed record가 완성될 때까지 offset을 전진한다.

```c
bytes = (unsigned char *)event;
offset = 0;
while (offset < sizeof(*event))
{
    size = read(g_event_pipe[0], bytes + offset,
            sizeof(*event) - offset);
    if (size == -1 && errno == EINTR)
        continue ;
    if (size <= 0)
        return (-1);
    offset += (size_t)size;
}
```

`run_event_loop`는 control socket과 event pipe를 같은 `pselect`에서 관찰한다.

```c
FD_SET(g_event_pipe[0], &read_set);
FD_SET(g_response_socket, &read_set);
status = pselect(max_fd + 1, &read_set, NULL, NULL, NULL, NULL);
```

ready source에 따라 session request 또는 bit event를 하나 처리한다. `process_bit`은 event의 signal 값과 sender를 검증하고, current owner의 bit일 때만 decoder state를 변경한다. 완성 byte를 output한 뒤 sequence ACK를 보내는 순서도 이제 normal context에서 실행된다.

### 왜 overflow를 복구하지 않는가

```c
if (g_event_overflow)
{
    errno = ENOBUFS;
    return (-1);
}
```

signal 하나는 bit 하나다. self-pipe write가 실패한 뒤 server가 계속 실행하면 accumulator가 실제 sender sequence보다 한 bit 뒤처진다. 이후의 모든 byte 경계와 ACK token이 틀어질 수 있지만 어느 위치에서 event가 빠졌는지 알 수 없다. 그래서 queue pressure를 “나중에 다시 시도할 일”로 보지 않고 **protocol state를 더 이상 신뢰할 수 없는 상태**로 보고 중단한다.

이 SHA에서 output helper는 아직 write 오류를 제대로 전파하지 않는다. stdout-before-ACK invariant는 Thread 04의 `db2004556d8b`에서 추가된다.

## `4c535ac8657e` — test(server): self-pipe 이벤트 손실 시 fail-stop 검증

**중요도** `A` · **태그** `TEST, SELF_PIPE, RISK`

production source의 event write call만 test build에서 교체할 수 있게 한다.

```c
#ifndef MT_EVENT_WRITE
# define MT_EVENT_WRITE write
#endif
```

fault implementation은 첫 event write에 `EAGAIN`을 한 번 반환한다.

```c
if (!failed && read_number("MT_TEST_EVENT_EAGAIN", 0) == 1)
{
    failed = 1;
    errno = EAGAIN;
    return (-1);
}
```

fixture가 요구하는 관측값은 다음과 같다.

- client의 message 전송은 ACK를 받지 못해 실패한다.
- server는 `server: signal event channel failed`를 출력하고 nonzero로 종료한다.
- server endpoint path는 normal exit cleanup으로 제거된다.

이 test는 pipe를 실제로 가득 채우지는 않는다. handler가 “한 event를 완전히 기록하지 못했다”는 production branch를 deterministic하게 재현해, overflow flag를 무시하고 계속 실행하는 회귀를 막는다.

## `e304c63bee3e` — fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리

**중요도** `A` · **태그** `SELF_PIPE, PROCESS_LIFECYCLE`

`SIGHUP`, `SIGINT`, `SIGTERM`에도 같은 `handle_bit`을 설치한다. handler가 직접 `_exit`하거나 cleanup하지 않고 termination event를 self-pipe에 넣는다.

`process_bit`은 sender나 data signal validation보다 먼저 termination signal을 구분한다.

```c
if (event->signal == SIGHUP || event->signal == SIGINT
    || event->signal == SIGTERM)
    return (event->signal);
```

`run_event_loop`는 이 양수 값을 main에 반환하고, main은 conventional status로 바꾼다.

```c
status = run_event_loop();
if (status == -1)
    return (1);
if (status > 0)
    return (128 + status);
```

main에서 return하므로 등록된 `atexit(cleanup_server)`가 socket과 pipe descriptors를 닫고 bound path를 unlink한다. 종료 signal도 data signal과 동일한 ordered input channel을 사용하므로, handler에서 filesystem이나 descriptor cleanup을 시도하지 않는다.

## `686b0d2a14e3` — fix(server): 상속된 이벤트 시그널 마스크 해제

**중요도** `A` · **태그** `PROCESS_LIFECYCLE, RISK`

handler를 설치하는 것과 signal을 받을 수 있게 만드는 것은 별개다. process가 blocked mask를 상속한 채 `exec`되면 새 disposition은 설치돼도 signal은 pending 상태로 남아 handler가 실행되지 않는다.

```c
static int unblock_event_signals(void)
{
    sigset_t event_signals;

    sigemptyset(&event_signals);
    sigaddset(&event_signals, MT_ZERO_SIGNAL);
    sigaddset(&event_signals, MT_ONE_SIGNAL);
    sigaddset(&event_signals, SIGHUP);
    sigaddset(&event_signals, SIGINT);
    sigaddset(&event_signals, SIGTERM);
    return (sigprocmask(SIG_UNBLOCK, &event_signals, NULL));
}
```

초기화 순서는 handler 설치 후 unblock이다.

```c
if (install_signal_handlers() == -1 || unblock_event_signals() == -1)
    return (1);
```

이 순서는 inherited pending signal이 default disposition으로 처리되는 창을 만들지 않고, 이 program이 실제로 소비하는 다섯 signal만 명시적으로 정상화한다. caller의 전체 mask를 빈 집합으로 덮어쓰지 않는다.

## `72424469474c` — test(server): 차단된 시그널 마스크 상속 뒤 메시지 검증

**중요도** `B` · **태그** `TEST, PROCESS_LIFECYCLE`

`tests/masked_exec`가 data signal 두 개와 termination signal 세 개를 block한 뒤 server를 `exec`한다. mask는 exec를 지나 유지되므로, `686b0d2a14e3`이 없으면 handler가 설치돼도 message를 받을 수 없다.

script는 다음을 함께 확인한다.

1. masked wrapper 아래 server가 PID line을 publish한다.
2. 정상 client가 `inherited mask`를 전송하고 server output에 같은 line이 생긴다.
3. server에 `SIGTERM`을 보낸다.
4. wrapper가 관찰한 status가 `143`이다.
5. server Unix socket path가 제거된다.

즉 이 test는 data deliverability만 확인하지 않는다. 같은 unblock set에 포함된 termination signal이 event loop를 통과해 status와 cleanup까지 보존하는지도 고정한다.

## 최종 execution flow

```text
[kernel signal delivery]
  → handle_bit
       sender/signal snapshot
       fixed record를 nonblocking pipe에 한 번 write
       실패/short write면 g_event_overflow=1
       errno 복원
  → pselect loop
       ├─ control socket: ACQUIRE 처리
       └─ event pipe: exact record read
            ├─ termination event → signal number 반환 → main return 128+signal
            └─ data event
                 owner 검사
                 bit accumulator 변경
                 8 bit이면 output
                 sequence 증가
                 ACK datagram 전송

[event write loss]
  → overflow flag 관찰
  → event loop -1
  → server status 1
  → atexit cleanup
```

## 이 Thread의 경계

이 Thread는 **signal이라는 비동기 입력을 normal context의 ordered event로 바꾸는 책임 분리**를 다룬다.

- 어떤 sender가 owner인지와 stale owner를 언제 회수하는지는 `02-session-ownership-and-recovery.md`의 주제다.
- output이 성공한 뒤에만 ACK를 보내는 commit boundary는 `04-output-commit-boundary.md`에서 다룬다.
- event pipe와 response socket descriptor가 `FD_SETSIZE` 범위 안에 있어야 한다는 조건은 `05-endpoint-ownership-and-bounded-polling.md`에서 다룬다.
- datagram 응답의 source/token validation은 `06-bounded-response-correlation.md`에서 다룬다.

## 조사 범위

각 commit은 exact SHA의 `src/server.c`, 관련 test source와 diff를 기준으로 설명했다. self-pipe 이후의 output fix를 `22363f83ff25`에 소급하지 않았고, 과도기 `a7d994b0a4b6`도 handler-safe 구조로 과장하지 않았다. test는 실행하지 않았으며 script와 fault seam이 요구하는 결과만 기록했다.
