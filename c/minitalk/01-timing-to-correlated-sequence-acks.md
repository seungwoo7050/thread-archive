# Thread: 다음 bit는 현재 sequence의 검증된 ACK 뒤에만 전송된다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 01

## 개요

초기 client는 bit signal을 보낸 뒤 고정 시간만 쉬었다. 이 방식은 receiver가 처리할 시간을 **추정**할 뿐, 어느 bit가 실제로 반영됐는지는 증명하지 못한다. 이 Thread는 그 추정을 signal ACK 기반 stop-and-wait로 바꾸고, 다시 ACK를 Unix datagram의 source·PID·kind·sequence와 연결한 뒤, 더 이상 필요 없어진 signal ACK와 pacing을 제거하는 과정이다.

커밋은 세 역할군으로 얽혀 있다. `89637d63b56f`부터 `4f17de94e025`까지는 timing과 signal ACK 사이의 과도기이고, `ebed06775b92`부터 `d3eacbbfeadc`까지는 식별 가능한 response channel을 만든다. 마지막 두 커밋은 성공 판단을 datagram ACK 하나로 수렴시키고, 그 결과 고정 지연을 삭제한다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `89637d63b56f` | feat(client): 메시지 바이트를 시그널로 전송 | A | `CORE, SIGNAL_DATA` | message byte를 MSB부터 signal bit로 직렬화하고 임시 고정 지연으로 전송 속도를 낮춘다. |
| 2 | `78de95b3cacb` | feat(protocol): 비트 처리마다 ACK 전송 | A | `CORE, SIGNAL_DATA, RISK` | bit마다 signal ACK를 기다리는 stop-and-wait와 ACK-before-wait race 방지를 도입한다. |
| 3 | `765efe7b75c9` | feat(client): ACK 대기 시간 초과 처리 | A | `RISK, PROCESS_LIFECYCLE, PRACTICAL` | ACK 부재를 무한 대기 대신 별도 timeout 결과로 노출한다. |
| 4 | `342aea9ce9a8` | fix(client): ACK 이후 시그널 전송 간격 안정화 | B | `SIGNAL_DATA, PRACTICAL` | ACK 뒤 짧은 inter-signal gap을 추가하고 ACK timeout을 늘린다. |
| 5 | `4f17de94e025` | fix(client): 인터럽트 뒤 남은 전송 간격 유지 | B | `SIGNAL_DATA, PRACTICAL` | `EINTR` 뒤 `nanosleep`의 남은 시간을 이어서 기다린다. |
| 6 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | `ARCH, RESPONSE` | acquisition과 bit transition을 식별할 request/response record를 정의한다. |
| 7 | `4234233ebd30` | feat(protocol): 비트 ACK를 sequence 응답으로 큐잉 | S | `ARCH, RESPONSE, CORE` | accepted bit에 sequence를 붙이고 datagram ACK 전송을 pipe queue 뒤 normal context로 넘긴다. |
| 8 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | `RESPONSE, RISK` | expected server source와 현재 sequence가 모두 맞는 datagram ACK만 성공으로 수락한다. |
| 9 | `aeb1b00867f4` | refactor(protocol): 이전 signal ACK 경로 제거 | A | `ARCH, RESPONSE, REFACTOR` | 더 이상 성공 판단에 쓰이지 않는 signal ACK/NACK machinery를 제거한다. |
| 10 | `1487a861046e` | perf(protocol): 검증된 ACK 뒤 고정 지연 제거 | A | `PERF, RESPONSE` | ACK가 one-bit-in-flight를 보장하므로 production과 test sender의 고정 sleep을 삭제한다. |

---

**역할군: elapsed time을 causal signal ACK로 대체하기**

## 89637d63b56f — feat(client): 메시지 바이트를 시그널로 전송

**중요도** `A` · **태그** `CORE, SIGNAL_DATA`

### 무엇을 만들었는가 (diff)

```diff
+static int send_bit(pid_t server_pid, int bit)
+{
+    int signal;
+
+    signal = MT_ZERO_SIGNAL;
+    if (bit != 0)
+        signal = MT_ONE_SIGNAL;
+    if (kill(server_pid, signal) == -1)
+        return (-1);
+    usleep(150);
+    return (0);
+}
+
+static int send_byte(pid_t server_pid, unsigned char byte)
+{
+    int shift;
+
+    shift = 7;
+    while (shift >= 0)
+    {
+        if (send_bit(server_pid, (byte >> shift) & 1) == -1)
+            return (-1);
+        shift--;
+    }
+    return (0);
+}
```

`main`은 PID 문자열을 `mt_parse_pid`로 검증하고 `kill(server_pid, 0)`으로 process 존재를 확인한 뒤, message를 byte 단위로 순회한다. 각 byte는 bit 7부터 bit 0까지 전송되므로 server의 left-shift assembly와 방향이 맞는다.

### 왜 이 단계만으로는 충분하지 않은가

`usleep(150)`은 receiver 완료 통지가 아니다. standard signal은 동일 종류의 pending instance를 counted queue처럼 보존하지 않으므로, 처리보다 빠르게 같은 signal을 보내면 여러 bit가 하나처럼 관찰될 수 있다. 이 SHA에서는 payload 뒤 NUL byte도 보내지 않으므로 message 종료 경계 역시 아직 없다.

### 관련 커밋과 어떤 관계인가

`78de95b3cacb`가 elapsed time을 다음 bit의 승인 조건으로 쓰는 가정을 깨고, server가 방금 처리한 bit에 대한 ACK를 관찰한 뒤에만 전진하도록 바꾼다.

## 78de95b3cacb — feat(protocol): 비트 처리마다 ACK 전송

**중요도** `A` · **태그** `CORE, SIGNAL_DATA, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
+static volatile sig_atomic_t g_ack_received;
+
+static void handle_ack(int signal)
+{
+    (void)signal;
+    g_ack_received = 1;
+}
@@
+    g_ack_received = 0;
     if (kill(server_pid, signal) == -1)
         return (-1);
-    usleep(150);
+    while (!g_ack_received)
+        sigsuspend(old_mask);
     return (0);
```

client는 ACK signal을 process mask에 먼저 block한 뒤 data signal을 보낸다. ACK를 block하지 않은 상태에서 `kill` 후 `sigsuspend`로 넘어가면, server ACK가 그 사이에 도착해 handler는 실행됐지만 wait는 이후 시작되는 lost-wakeup race가 생길 수 있다. block → send → ACK만 임시 unblock한 `sigsuspend` 순서가 그 race를 닫는다.

server는 한 bit를 accumulator에 반영한 뒤 sender PID로 ACK를 보낸다. client의 bit cursor는 `send_bit`이 성공한 뒤에만 감소한다. 이 커밋은 payload 뒤 NUL byte 전송도 추가해 message 종료를 protocol data에 포함한다.

### 무엇이 아직 없는가

ACK signal에는 bit identity나 session identity가 없다. ACK가 오지 않으면 client는 무한히 기다리고, 오래된 ACK와 현재 bit의 ACK를 구분할 수도 없다.

### 관련 커밋과 어떤 관계인가

`765efe7b75c9`는 ACK 부재를 bounded failure로 바꾸고, 훨씬 뒤의 `d3eacbbfeadc`는 signal 자체가 표현하지 못하는 bit identity를 sequence token으로 보강한다.

## 765efe7b75c9 — feat(client): ACK 대기 시간 초과 처리

**중요도** `A` · **태그** `RISK, PROCESS_LIFECYCLE, PRACTICAL`

### 무엇이 바뀌었는가 (diff)

```diff
+#define SEND_ERROR 1
+#define SEND_TIMEOUT 2
+
 static volatile sig_atomic_t g_ack_received;
+static volatile sig_atomic_t g_timed_out;
@@
+    g_timed_out = 0;
     if (kill(server_pid, signal) == -1)
-        return (-1);
-    while (!g_ack_received)
+        return (SEND_ERROR);
+    alarm(MT_ACK_TIMEOUT_SECONDS);
+    while (!g_ack_received && !g_timed_out)
         sigsuspend(old_mask);
+    alarm(0);
+    if (g_timed_out)
+        return (SEND_TIMEOUT);
```

ACK와 `SIGALRM`을 같은 blocked/wait mask에 넣어 어느 쪽이 먼저 상태를 바꾸는지 관찰한다. timeout이면 `send_byte`가 즉시 중단되므로 실패한 bit 뒤의 cursor와 다음 byte로 진행하지 않는다. client diagnostic도 일반 signal 전송 실패와 ACK timeout을 구분한다.

### 이 커밋이 보장하지 않는 것은 무엇인가

alarm은 ACK가 현재 bit의 것인지 확인하지 않는다. 또 timeout은 delivery를 보장하는 장치가 아니라, 확인할 수 없는 상태에서 더 진행하지 않고 포기하는 상한이다.

### 관련 커밋과 어떤 관계인가

`342aea9ce9a8`은 실제 signal 처리 간격에 맞춰 timeout과 pacing을 조정한다. 이후 datagram path는 같은 “현재 operation 안에서만 기다린다”는 책임을 monotonic deadline으로 다시 구현한다.

## 342aea9ce9a8 — fix(client): ACK 이후 시그널 전송 간격 안정화

**중요도** `B` · **태그** `SIGNAL_DATA, PRACTICAL`

### 무엇이 바뀌었는가 (diff)

```diff
-# define MT_ACK_TIMEOUT_SECONDS 1
+# define MT_ACK_TIMEOUT_SECONDS 3
+# define MT_SIGNAL_GAP_US 500
@@
     if (g_timed_out)
         return (SEND_TIMEOUT);
+    usleep(MT_SIGNAL_GAP_US);
     return (0);
```

### 왜 이렇게 작은가

이 커밋은 ACK 의미를 바꾸지 않는다. signal ACK가 관찰된 직후 다음 signal을 보내는 대신 500µs를 쉬고, 느린 환경을 위해 timeout을 3초로 넓힌다. 즉 correctness를 identity로 강화한 것이 아니라 과도기 signal transport의 운용 여유를 늘린 조정이다.

### 관련 커밋과 어떤 관계인가

`4f17de94e025`는 이 gap이 signal interruption으로 짧아지지 않도록 보강한다. `1487a861046e`는 sequence ACK가 ordering을 맡은 뒤 gap 자체를 제거한다.

## 4f17de94e025 — fix(client): 인터럽트 뒤 남은 전송 간격 유지

**중요도** `B` · **태그** `SIGNAL_DATA, PRACTICAL`

### 무엇이 바뀌었는가 (diff)

```diff
-# define MT_SIGNAL_GAP_US 500
+# define MT_SIGNAL_GAP_US 5000
@@
+static void wait_signal_gap(void)
+{
+    struct timespec remaining;
+
+    remaining.tv_sec = 0;
+    remaining.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
+    while (nanosleep(&remaining, &remaining) == -1 && errno == EINTR)
+        ;
+}
@@
-    usleep(MT_SIGNAL_GAP_US);
+    wait_signal_gap();
```

`nanosleep`이 `EINTR`를 반환하면 두 번째 `remaining` 인자에 아직 자지 못한 시간이 기록된다. 같은 object를 다음 호출에 넘겨 logical gap의 남은 부분을 이어간다. 다만 helper가 `void`이고 loop는 non-`EINTR` 오류에서 끝나므로, 그 오류를 caller failure로 전파하지는 않는다.

### 관련 커밋과 어떤 관계인가

이 pacing은 signal ACK transport의 과도기 보조 장치다. `1487a861046e`가 exact sequence ACK 이후 삭제하므로, 최종 protocol 불변 조건에는 포함되지 않는다.

---

**역할군: ACK를 식별 가능한 datagram response로 바꾸기**

## ebed06775b92 — feat(protocol): 응답 메시지 wire 형식 정의

**중요도** `A` · **태그** `ARCH, RESPONSE`

### 무엇을 준비했는가 (diff)

```diff
+# define MT_RESPONSE_MAGIC 0x4d54414bU
+# define MT_REQUEST_ACQUIRE 1U
+# define MT_RESPONSE_READY 1U
+# define MT_RESPONSE_ACK 2U
+# define MT_RESPONSE_OK 0
+# define MT_RESPONSE_BUSY 1
+
+typedef struct s_mt_request
+{
+    uint32_t magic;
+    uint32_t kind;
+    uint32_t nonce;
+    pid_t    client_pid;
+} t_mt_request;
+
+typedef struct s_mt_response
+{
+    uint32_t magic;
+    uint32_t kind;
+    uint32_t token;
+    int32_t  status;
+    pid_t    server_pid;
+} t_mt_response;
```

signal ACK에는 payload가 없지만 이 record는 acquisition인지 bit ACK인지, 어느 request 또는 sequence인지, 어느 PID가 응답했다고 주장하는지를 표현할 수 있다. 이 SHA는 representation만 추가하며 `sendto`/`recvfrom` acceptance logic은 아직 없다.

C struct를 `sizeof` 그대로 local datagram에 쓰는 전제이므로 portable serialization이나 versioned network protocol은 아니다. padding, `pid_t`, native byte order가 같은 host ABI 범위에서만 해석된다.

### 관련 커밋과 어떤 관계인가

`4234233ebd30`은 `token`을 server-side bit sequence로 사용해 ACK를 만든다. `f8e8444c5ded`은 같은 schema의 nonce를 READY correlation에 사용하지만, 그 acquisition 상세는 세션 Thread와 response-correlation Thread에서 다룬다.

## 4234233ebd30 — feat(protocol): 비트 ACK를 sequence 응답으로 큐잉

**중요도** `S` · **태그** `ARCH, RESPONSE, CORE`

### 무엇을 만들었는가 (diff)

server는 accepted bit의 현재 sequence를 capture하고, response 전송에 필요한 최소 record를 pipe에 넣는다.

```diff
+static volatile sig_atomic_t g_response_overflow;
+static volatile sig_atomic_t g_sequence;
+static int g_response_pipe[2] = {-1, -1};
@@
+static void queue_response(pid_t client_pid, uint32_t kind,
+        uint32_t token, int status)
+{
+    t_response_request request;
+
+    request.client_pid = client_pid;
+    request.kind = kind;
+    request.token = token;
+    request.status = status;
+    if (write(g_response_pipe[1], &request, sizeof(request))
+        != (ssize_t)sizeof(request))
+        g_response_overflow = 1;
+}
@@
+    sequence = (uint32_t)g_sequence;
     g_current_byte <<= 1;
@@
+    if (g_client_pid != 0)
+        g_sequence++;
+    queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
```

client는 bit마다 expected sequence를 넘기고, matching datagram이 올 때까지 기다린 뒤에만 sequence를 증가시킨다.

```diff
-static int send_bit(pid_t server_pid, int bit, const sigset_t *old_mask)
+static int send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
+        uint32_t sequence, const char *server_path)
@@
+    status = wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
+            server_path, &deadline);
+    if (status != 0)
+        return (status);
@@
-    status = send_bit(server_pid, (byte >> shift) & 1, old_mask);
+    status = send_bit(server_pid, (byte >> shift) & 1, old_mask,
+            *sequence, server_path);
     if (status != 0)
         return (status);
+    (*sequence)++;
```

### 왜 이 커밋이 Thread의 중심인가

sequence는 server state transition과 client cursor를 같은 token으로 묶는다. server가 bit `n`을 accepted state에 반영한 순간 token `n`을 생성하고, client는 token `n`의 ACK를 수락한 뒤에만 `n+1`로 간다. stale ACK는 현재 token과 일치하지 않아 cursor를 전진시킬 수 없다.

response pipe는 signal handler 안에서 filesystem path 검사와 `sendto`를 직접 수행하지 않도록 send work를 normal loop로 넘기는 중간 장치다. 다만 이 exact SHA에서는 handler가 여전히 bit state를 직접 바꾸고 signal ACK도 함께 보낸다. self-pipe event loop로 authoritative state mutation 전체가 이동하는 시점은 별도 Thread의 `22363f83ff25`다.

### 보장하는 것과 아직 보장하지 않는 것은 무엇인가

이 SHA는 sequence datagram ACK를 추가하지만 success source가 하나로 수렴하지 않았다. client는 먼저 signal ACK를 기다리고 그 뒤 datagram ACK를 기다리므로, legacy signal path와 new response path가 동시에 존재한다.

### 관련 커밋과 어떤 관계인가

`d3eacbbfeadc`가 datagram ACK를 bit 성공의 단일 조건으로 만들고, `aeb1b00867f4`가 그 뒤 남은 legacy handler·mask·server signal response를 코드에서 제거한다.

## d3eacbbfeadc — feat(client): 비트 ACK를 sequence로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK`

### 무엇이 바뀌었는가 (diff)

```diff
-static int send_bit(pid_t server_pid, int bit, const sigset_t *old_mask,
-        uint32_t sequence, const char *server_path)
+static int send_bit(pid_t server_pid, int bit, uint32_t sequence,
+        const char *server_path)
@@
-    g_ack_received = 0;
-    g_timed_out = 0;
-    g_rejected = 0;
-    if (kill(server_pid, signal) == -1)
+    if (validate_server_socket(server_path) == -1)
         return (SEND_ERROR);
-    alarm(MT_ACK_TIMEOUT_SECONDS);
-    while (!g_ack_received && !g_timed_out && !g_rejected)
-        sigsuspend(old_mask);
-    alarm(0);
-    if (g_rejected)
-        return (SEND_REJECTED);
-    if (g_timed_out)
-        return (SEND_TIMEOUT);
     if (clock_gettime(CLOCK_MONOTONIC, &deadline) == -1)
         return (SEND_ERROR);
     deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+    if (kill(server_pid, signal) == -1)
+        return (SEND_ERROR);
     status = wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
             server_path, &deadline);
```

signal ACK/NACK wait가 `send_bit`에서 사라진다. expected server path를 확인하고 absolute deadline을 만든 뒤 bit signal을 보낸다. `wait_for_response`가 exact source, magic, PID, kind, token, status가 맞는 response만 성공으로 반환하며, `send_byte`는 그 성공 뒤에만 `(*sequence)++`를 수행한다.

### 무엇이 아직 남아 있는가

이 SHA의 client source에는 더 이상 성공 판단에 사용되지 않는 signal handler와 mask setup 일부가 남아 있다. “실행 경로에서 사용하지 않음”과 “코드에서 제거됨”은 다르며, 후자는 다음 refactor의 책임이다.

### 관련 커밋과 어떤 관계인가

`aeb1b00867f4`는 이 커밋이 이미 무효화한 signal response machinery를 client·server·test helper에서 제거해 authoritative success path를 datagram 하나로 고정한다.

---

**역할군: 중복 성공 경로와 timing 보조 장치를 제거하기**

## aeb1b00867f4 — refactor(protocol): 이전 signal ACK 경로 제거

**중요도** `A` · **태그** `ARCH, RESPONSE, REFACTOR`

이 커밋은 새 response 기능을 추가하는 refactor가 아니라, `d3eacbbfeadc` 이후 죽은 성공 경로를 제거해 protocol authority를 하나로 수렴시킨다.

### 무엇이 제거되었는가 (diff)

```diff
-# define MT_ACK_SIGNAL SIGUSR1
-# define MT_NACK_SIGNAL SIGUSR2
@@
-static volatile sig_atomic_t g_ack_received;
-static volatile sig_atomic_t g_timed_out;
-static volatile sig_atomic_t g_rejected;
@@
-static void handle_client_signal(int signal)
-{
-    /* ... */
-}
@@
-    if (kill(info->si_pid, MT_ACK_SIGNAL) == -1 && errno == ESRCH)
-        reset_session(1);
```

server는 owner가 아니면 signal NACK를 보내는 대신 data event를 무시하고, accepted bit의 결과는 datagram ACK로만 보낸다. client의 signal handlers, blocked masks, alarm flags와 test sender의 같은 machinery도 삭제된다.

### refactor 뒤 production contract는 무엇인가

bit 성공은 matching datagram response 하나로만 결정된다. signal은 data bit를 전달하는 채널이고, response socket은 READY·BUSY·ACK를 전달하는 control channel이다. 두 채널의 책임이 분리되면서 같은 event에 두 종류의 ACK를 기다리는 상태가 사라진다.

### 관련 커밋과 어떤 관계인가

`1487a861046e`는 이 단일 ACK 경로가 one-bit-in-flight를 이미 보장한다는 사실을 이용해 마지막 timing gap을 제거한다.

## 1487a861046e — perf(protocol): 검증된 ACK 뒤 고정 지연 제거

**중요도** `A` · **태그** `PERF, RESPONSE`

### 무엇이 바뀌었는가 (diff)

```diff
-# define MT_SIGNAL_GAP_US 5000
@@
-static void wait_signal_gap(void)
-{
-    /* ... */
-}
@@
     if (status != 0)
         return (status);
-    wait_signal_gap();
     return (0);
```

같은 삭제가 production client와 `tests/session_sender.c`에 적용된다. 이제 다음 bit의 조건은 “ACK 뒤 일정 시간이 지났다”가 아니라 “현재 sequence의 ACK를 검증했다” 하나뿐이다.

### 왜 성능 변경이 protocol 변경의 결과인가

sleep을 단순히 줄인 것이 아니다. `d3eacbbfeadc`와 `aeb1b00867f4`가 현재 bit의 정확한 ACK만 성공으로 인정하고 그 뒤에만 sequence를 증가시키므로, ACK가 돌아온 시점에 이전 bit는 이미 server의 accepted transition과 연결돼 있다. timing은 correctness 조건에서 제거할 수 있다.

### 관련 커밋과 어떤 관계인가

이 커밋은 `4234233ebd30`의 sequence 생성, `d3eacbbfeadc`의 correlation, `aeb1b00867f4`의 단일 authority가 모두 성립한 뒤에만 안전하다. 셋 중 하나라도 되돌리면 sleep 제거만으로 ordering을 설명할 수 없다.

## 이 Thread의 경계

- session을 누가 소유하고 언제 회수하는지는 `02-session-ownership-and-recovery.md`의 책임이다. 이 문서는 bit별 전진 조건만 다룬다.
- signal handler에서 state mutation을 normal event loop로 옮기는 문제는 `03-self-pipe-event-loop.md`가 다룬다. `4234233ebd30`의 response pipe를 최종 self-pipe architecture로 소급하지 않는다.
- stdout 반영 성공을 ACK 조건으로 만드는 문제는 `04-output-commit-boundary.md`의 책임이다.
- Unix socket path의 생성·bind·unlink ownership과 `FD_SETSIZE` 경계는 `05-endpoint-ownership-and-bounded-polling.md`가 다룬다.
- forged, oversized, invalid-flood response를 포함한 acceptance predicate와 bounded deadline의 상세 검증은 `06-bounded-response-correlation.md`가 다룬다.

> 검토 범위: `89637d63b56f`, `78de95b3cacb`, `765efe7b75c9`, `342aea9ce9a8`, `4f17de94e025`, `ebed06775b92`, `4234233ebd30`, `d3eacbbfeadc`, `aeb1b00867f4`, `1487a861046e`의 diff와 해당 SHA의 관련 client/server/header/test source를 확인했다. branch checkout, build, test 실행은 수행하지 않았으므로 runtime 통과를 주장하지 않는다.
