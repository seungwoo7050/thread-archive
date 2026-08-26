# Thread: 세션 소유권은 응답 가능한 하나의 sender에게만 유지된다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 02

## 개요

server에는 sender마다 별도의 decoder가 있는 것이 아니라 하나의 partial byte, received-bit count, sequence와 visible-line 상태가 있다. 따라서 두 PID가 이 state를 번갈아 바꾸면 각각의 signal은 유효해도 완성된 byte는 어느 message에도 속하지 않는다. 이 Thread는 shared decoder의 owner를 하나로 제한하고, owner가 사라졌을 때 coupled state를 함께 회수하는 책임을 확립한다.

처음에는 첫 data signal의 PID가 암묵적으로 owner가 된다. 이후 `ACQUIRE`/`READY` control exchange가 data보다 먼저 owner를 예약하며, 마지막 fix는 process 존재만 보던 liveness를 response endpoint의 사용 가능성까지 확장한다. test commits는 bit-only abandon, visible partial line, data 없는 reservation, exited-but-unreaped owner를 서로 다른 fixture로 고정한다.

### 최종 상태

| server 상태 | incoming actor | 결과 |
| --- | --- | --- |
| owner 없음 | 검증된 `ACQUIRE` | requester를 owner로 예약하고 matching `READY` 전송 |
| owner 있음·응답 가능 | 같은 owner | bit state를 변경하고 current sequence ACK 전송 |
| owner 있음·응답 가능 | competitor | decoder를 건드리지 않고 `BUSY` 반환 |
| owner process 또는 endpoint 사용 불가 | replacement requester | partial state를 회수한 뒤 새 reservation 시도 |
| visible partial line의 recovery newline 실패 | replacement requester | 회수를 성공으로 간주하지 않고 server failure로 종료 |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `10a7211969bf` | fix(server): 활성 세션에 다른 송신자 거부 | A | `SESSION, RISK` | 첫 accepted bit의 PID를 owner로 저장하고 NUL까지 competitor의 data를 거부한다. |
| 2 | `bc337552961a` | fix(server): 종료된 송신자의 세션 복구 | A | `SESSION, PROCESS_LIFECYCLE, RISK` | 사라진 owner의 partial byte·bit count·line state를 함께 reset한다. |
| 3 | `bdccf91f5a44` | test(server): 중단·경쟁 송신자 세션 검증 | A | `TEST, SESSION, RISK` | bit-only abandon, partial-line abandon, live competition과 owner exit recovery를 재현한다. |
| 4 | `caf2feec4971` | feat(server): 획득 요청을 검증해 세션 소유권 예약 | S | `ARCH, SESSION, CORE` | exact `ACQUIRE` datagram을 검증한 뒤 data 전에 owner를 예약하고 `READY` 또는 `BUSY`를 반환한다. |
| 5 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | `RESPONSE, RISK, INTEGRATION` | client가 nonzero nonce를 보내고 expected source·PID·nonce가 맞는 `READY`만 수락한다. |
| 6 | `e56e8cc87315` | test(session): 데이터 없는 활성 예약 경쟁 검증 | A | `TEST, SESSION` | acquisition만 마친 live owner도 exclusive하다는 사실을 `reserve` fixture로 고정한다. |
| 7 | `1e3da4580733` | fix(server): 응답 경로가 사라진 세션 소유자 회수 | S | `SESSION, PROCESS_LIFECYCLE, DEBUG` | owner availability를 PID 존재와 expected client socket 사용 가능성의 결합으로 정의한다. |
| 8 | `a481bfabb7b5` | test(session): 종료 송신자 회수 전 새 세션 복구 검증 | A | `TEST, SESSION, PROCESS_LIFECYCLE` | child를 unreaped 상태로 남겨 PID는 보이지만 endpoint는 사라진 owner를 재현한다. |

---

**역할군: 첫 data signal로 shared decoder의 owner를 정하기**

## 10a7211969bf — fix(server): 활성 세션에 다른 송신자 거부

**중요도** `A` · **태그** `SESSION, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

이전 server의 `g_current_byte`와 `g_received_bits`는 process-wide state였지만 어느 PID가 만든 partial data인지는 기록하지 않았다. 서로 다른 client가 signal을 교차 전송하면 같은 accumulator에 bit가 섞인다. 해결은 첫 accepted signal의 sender를 owner로 저장하고, message terminator가 완성될 때까지 다른 PID가 mutation code에 도달하지 못하게 하는 것이다.

```diff
+static volatile sig_atomic_t g_client_pid;
@@
+    if (g_client_pid != 0 && g_client_pid != info->si_pid)
+    {
+        kill(info->si_pid, MT_NACK_SIGNAL);
+        return ;
+    }
+    if (g_client_pid == 0)
+        g_client_pid = info->si_pid;
     g_current_byte <<= 1;
@@
     if (output == '\0')
     {
         write(STDOUT_FILENO, "\n", 1);
+        g_client_pid = 0;
     }
```

### 이 커밋이 보장하는 것 / 보장하지 않는 것

owner가 아닌 sender는 accumulator mutation 전에 return하므로 bit interleaving을 막는다. 그러나 owner는 첫 bit가 도착해야만 생기고, owner가 NUL 전에 종료되면 `g_client_pid`가 영구히 남을 수 있다. data-before-reservation race도 아직 그대로다.

### 관련 커밋과 어떤 관계인가

`bc337552961a`는 owner가 message를 끝내지 못하고 사라지는 경우를 회수한다. `caf2feec4971`은 더 근본적으로 owner 결정 시점을 첫 data bit보다 앞선 control request로 이동한다.

## bc337552961a — fix(server): 종료된 송신자의 세션 복구

**중요도** `A` · **태그** `SESSION, PROCESS_LIFECYCLE, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

owner PID만 저장하면 중도 종료된 client가 shared decoder를 계속 점유한다. competitor가 들어온 시점에 `kill(owner, 0)`이 `ESRCH`를 반환하면 기존 owner가 사라진 것으로 보고, owner identity만이 아니라 partial byte와 line state까지 함께 reset한다.

```diff
+static volatile sig_atomic_t g_line_started;
+
+static void reset_session(int close_partial_line)
+{
+    if (close_partial_line && g_line_started)
+        write(STDOUT_FILENO, "\n", 1);
+    g_current_byte = 0;
+    g_received_bits = 0;
+    g_client_pid = 0;
+    g_line_started = 0;
+}
@@
     if (g_client_pid != 0 && g_client_pid != info->si_pid)
     {
-        kill(info->si_pid, MT_NACK_SIGNAL);
-        return ;
+        if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
+            reset_session(1);
+        else
+        {
+            kill(info->si_pid, MT_NACK_SIGNAL);
+            errno = saved_errno;
+            return ;
+        }
     }
```

이미 complete byte가 stdout에 보인 뒤 partial next byte를 남긴 owner라면 recovery newline으로 그 줄을 닫고 새 sender를 받는다. bit 하나만 받은 상태처럼 visible output이 없으면 newline을 쓰지 않는다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

PID가 완전히 사라진 owner는 회수한다. 하지만 exited child가 아직 parent에게 reap되지 않은 zombie라면 PID는 여전히 process table에 남아 `kill(pid, 0)`만으로는 unavailable 상태를 구분하지 못한다.

### 관련 커밋과 어떤 관계인가

`bdccf91f5a44`가 bit-only·partial-line·live competitor를 process fixture로 고정한다. zombie와 vanished response socket의 불일치는 `1e3da4580733`에서 별도로 수정된다.

## bdccf91f5a44 — test(server): 중단·경쟁 송신자 세션 검증

**중요도** `A` · **태그** `TEST, SESSION, RISK`

### 무엇을 검증하는가

새 `tests/session_sender`는 정상 client가 만들기 어려운 중간 상태를 의도적으로 남긴다. `bit`는 한 bit만 보내고 종료하고, `partial`은 `'X'` 한 byte를 출력한 뒤 다음 byte의 일부만 보내고 종료한다. `hold`는 partial state를 만든 채 살아 있어 competitor가 `BUSY`를 받는지 확인한다.

```diff
+"$ROOT/tests/session_sender" "$SERVER_PID" bit
+"$ROOT/client" "$SERVER_PID" "bit recovered"
+
+"$ROOT/tests/session_sender" "$SERVER_PID" partial
+"$ROOT/client" "$SERVER_PID" "line recovered"
+
+"$ROOT/tests/session_sender" "$SERVER_PID" hold >"$READY" &
+if "$ROOT/client" "$SERVER_PID" "blocked" 2>"$BUSY_ERR"; then
+    printf 'server accepted a competitor while the owner was alive\n' >&2
+    exit 1
+fi
+diff -u "$BEFORE_BUSY" "$OUT"
```

### 이 테스트가 증명하는 것 / 증명하지 않는 것

selected scenarios에서 dead owner recovery가 진행되고, live competitor는 output을 바꾸지 못하며, visible partial line은 newline으로 닫힌다는 assertion을 제공한다. 이 시점의 holder는 data를 이미 보냈으므로 “data가 하나도 없는 explicit reservation도 exclusive하다”는 사실은 증명하지 않는다. zombie PID도 만들지 않는다.

### 관련 커밋과 어떤 관계인가

이 테스트는 `10a7211969bf`와 `bc337552961a`의 signal-era ownership을 고정한다. `e56e8cc87315`는 explicit acquisition 도입 뒤 fixture를 data-free reservation으로 바꿔 새로운 경계를 겨냥한다.

---

**역할군: control channel에서 data보다 먼저 owner를 예약하기**

## caf2feec4971 — feat(server): 획득 요청을 검증해 세션 소유권 예약

**중요도** `S` · **태그** `ARCH, SESSION, CORE`

### 무엇을 만들었는가 (diff)

server는 response socket에서 request record보다 한 byte 큰 buffer로 datagram을 받고, 길이가 정확히 `sizeof(t_mt_request)`인 경우에만 구조체로 복사한다. source path, claimed PID, socket type/UID와 process 존재도 함께 확인한다.

```diff
+static int valid_request_source(const struct sockaddr_un *source,
+        const t_mt_request *request)
+{
+    char client_path[MT_RESPONSE_PATH_SIZE];
+
+    if (request->magic != MT_RESPONSE_MAGIC
+        || request->kind != MT_REQUEST_ACQUIRE || request->client_pid <= 1
+        || source->sun_family != AF_UNIX
+        || source->sun_path[sizeof(source->sun_path) - 1] != '\0'
+        || mt_response_path(client_path, sizeof(client_path), "client",
+            request->client_pid) == -1
+        || strcmp(source->sun_path, client_path) != 0
+        || valid_client_socket(client_path) == -1
+        || kill(request->client_pid, 0) == -1)
+        return (0);
+    return (1);
+}
```

검증된 request가 들어오면 현재 owner와 비교해 `READY` 또는 `BUSY`를 만든다.

```diff
+    status = MT_RESPONSE_OK;
+    if (g_client_pid != 0 && g_client_pid != request.client_pid)
+    {
+        if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
+            reset_session(1);
+        else
+            status = MT_RESPONSE_BUSY;
+    }
+    if (status == MT_RESPONSE_OK && g_client_pid == 0)
+    {
+        g_client_pid = request.client_pid;
+        new_owner = 1;
+    }
+    response.kind = MT_RESPONSE_READY;
+    response.token = request.nonce;
```

### 설계 결정은 무엇인가

ownership을 첫 data signal의 우연한 도착 순서에서 검증된 control message로 옮긴다. `READY`가 전송되기 전에 `g_client_pid`가 설정되므로, payload가 아직 없어도 reservation은 이미 exclusive하다. 새 owner에게 `READY`를 보내지 못하면 reservation을 되돌려 phantom owner를 남기지 않는다.

이 exact SHA에서는 bit handler의 authoritative state mutation이 아직 signal context에 있다. explicit session reservation과 self-pipe event-loop 전환은 서로 다른 개발 문제다.

### 관련 커밋과 어떤 관계인가

`f8e8444c5ded`가 production client에 `ACQUIRE` 전송과 nonce-correlated `READY` wait를 연결한다. `e56e8cc87315`는 data를 보내지 않은 requester가 실제로 competitor를 막는지 검증한다.

## f8e8444c5ded — feat(client): READY 응답을 출처와 nonce로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK, INTEGRATION`

### 무엇이 바뀌었는가 (diff)

client는 `/dev/urandom`에서 `uint32_t` nonce를 short-read와 `EINTR`에 대응해 채우고, 0이면 1로 바꾼다. `ACQUIRE`를 보내기 전에 server socket을 확인하고 monotonic deadline을 한 번 만든다.

```diff
+    request.magic = MT_RESPONSE_MAGIC;
+    request.kind = MT_REQUEST_ACQUIRE;
+    request.client_pid = getpid();
+    deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+    if (sendto(g_response_socket, &request, sizeof(request), 0,
+            (struct sockaddr *)&address, sizeof(address))
+        != (ssize_t)sizeof(request))
+        return (SEND_ERROR);
+    return (wait_for_response(server_pid, MT_RESPONSE_READY, request.nonce,
+            server_path, &deadline));
```

`wait_for_response`는 datagram source path, `magic`, `server_pid`, `kind`, echoed nonce와 `status`를 함께 검사한다. 하나라도 다르면 같은 deadline 안에서 폐기한다. `BUSY`는 timeout이나 generic I/O error와 다른 `SEND_REJECTED` 결과다.

### 관련 커밋과 어떤 관계인가

이 client-side correlation이 있어야 `caf2feec4971`의 reservation을 production path에서 신뢰할 수 있다. invalid response를 더 공격적으로 주입하는 검증은 `06-bounded-response-correlation.md`에서 다룬다.

## e56e8cc87315 — test(session): 데이터 없는 활성 예약 경쟁 검증

**중요도** `A` · **태그** `TEST, SESSION`

### 무엇을 검증하는가 (diff)

```diff
-"$ROOT/tests/session_sender" "$SERVER_PID" hold >"$READY" &
+"$ROOT/tests/session_sender" "$SERVER_PID" reserve >"$READY" &
@@
-else if (strcmp(argv[2], "partial") == 0
-    || strcmp(argv[2], "hold") == 0)
+else if (strcmp(argv[2], "partial") == 0)
     send_partial(...);
-else
+else if (strcmp(argv[2], "reserve") != 0)
     return (1);
```

`reserve` mode는 `ACQUIRE`와 matching `READY`까지만 수행하고 signal data를 하나도 보내지 않은 채 살아 있다. 그동안 normal client가 실패하고 server output이 변하지 않으면 exclusivity의 시작점이 “첫 bit”가 아니라 “accepted acquisition”임을 구별할 수 있다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

live data-free owner가 competitor를 차단하고, owner가 종료된 뒤 새 client가 진행한다는 selected scenario를 고정한다. exited-but-unreaped owner처럼 PID가 남은 상태는 아직 만들지 않는다.

### 관련 커밋과 어떤 관계인가

`caf2feec4971`의 server reservation과 `f8e8444c5ded`의 client handshake를 함께 통과하는 regression이다. `a481bfabb7b5`는 owner lifecycle을 zombie 경계까지 확장한다.

---

**역할군: PID 존재와 response endpoint 사용 가능성을 함께 회수 조건으로 삼기**

## 1e3da4580733 — fix(server): 응답 경로가 사라진 세션 소유자 회수

**중요도** `S` · **태그** `SESSION, PROCESS_LIFECYCLE, DEBUG`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

`kill(owner, 0)`은 exited child가 아직 reap되지 않은 동안에도 PID가 존재하는 것처럼 보일 수 있다. 하지만 client의 `atexit` cleanup은 이미 Unix response socket을 삭제했으므로, server는 ACK를 돌려줄 수 없는 owner를 계속 live로 판단한다. 세션 availability는 process identity 하나가 아니라 **process 존재와 expected response endpoint 사용 가능성**의 결합이어야 한다.

```diff
+static int session_owner_available(void)
+{
+    char client_path[MT_RESPONSE_PATH_SIZE];
+
+    if (g_client_pid <= 1)
+        return (0);
+    if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
+        return (0);
+    if (mt_response_path(client_path, sizeof(client_path), "client",
+            g_client_pid) == -1 || valid_client_socket(client_path) == -1)
+        return (0);
+    return (1);
+}
@@
-        if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
+        if (!session_owner_available())
         {
             if (reset_session(1) == -1)
                 return (-1);
```

### 성공 경로와 실패 경로는 어떻게 갈리는가

owner PID와 same-UID Unix socket이 모두 사용 가능하면 competitor는 계속 `BUSY`다. 둘 중 하나라도 없으면 `reset_session(1)`로 partial decoder와 sequence를 회수한다. 이 SHA의 `reset_session`은 visible partial line의 newline write가 실패하면 `-1`을 반환하므로, 그런 상태에서 새 owner를 승인하지 않고 event loop 전체가 failure로 끝난다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

zombie처럼 PID는 남았지만 response endpoint가 사라진 owner를 회수한다. path와 UID 검사는 cooperative local integrity boundary일 뿐 cryptographic peer authentication은 아니다. 같은 UID의 다른 process가 path를 조작하는 위협까지 해결하지 않는다.

### 관련 커밋과 어떤 관계인가

`a481bfabb7b5`는 `waitid(..., WNOWAIT)`로 child를 의도적으로 unreaped 상태에 두고 client socket disappearance를 함께 관찰해 이 root cause를 직접 재현한다.

## a481bfabb7b5 — test(session): 종료 송신자 회수 전 새 세션 복구 검증

**중요도** `A` · **태그** `TEST, SESSION, PROCESS_LIFECYCLE`

### 왜 다른 기법이 필요한가

일반 shell script에서 child를 `wait`하면 zombie 상태가 바로 사라져 PID-only liveness의 문제를 재현할 수 없다. 새 `unreaped_exec` helper는 partial-session child를 실행한 뒤 `WNOWAIT`로 종료를 확인하되 reap하지 않는다.

```diff
+static int child_exited_unreaped(pid_t child)
+{
+    siginfo_t info;
+
+    info.si_pid = 0;
+    if (waitid(P_PID, (id_t)child, &info,
+            WEXITED | WNOHANG | WNOWAIT) == -1)
+        return (-1);
+    return (info.si_pid == child);
+}
@@
+    if (status == 1 && lstat(client_path, &info) == -1
+        && errno == ENOENT)
+        return (0);
```

harness는 published child PID에 `kill -0`이 성공하는 상태를 확인한 뒤, 새 client가 `unreaped recovered`를 전송할 수 있어야 한다고 요구한다. expected output에는 abandoned owner가 남긴 `X`, recovery newline, replacement message가 순서대로 포함된다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

PID는 관찰 가능하지만 endpoint는 사라진 exact fixture에서 server가 기존 owner를 회수한다는 regression evidence다. arbitrary PID reuse, same-UID path spoofing, 다른 zombie lifecycle 조합 전체를 증명하지는 않는다.

### 관련 커밋과 어떤 관계인가

`1e3da4580733`의 `session_owner_available`가 PID-only check를 대체하지 않으면 이 fixture의 replacement acquisition은 계속 `BUSY`가 된다. 따라서 test technique과 production fix가 직접 대응한다.

## 이 Thread의 경계

- bit별 ACK와 sequence가 다음 전송을 승인하는 과정은 `01-timing-to-correlated-sequence-acks.md`가 다룬다.
- signal handler에서 session state mutation을 event loop로 옮기는 책임은 `03-self-pipe-event-loop.md`에 있다.
- recovery newline을 포함한 stdout write가 ACK·READY보다 먼저 성공해야 하는 규칙은 `04-output-commit-boundary.md`가 다룬다.
- response socket path의 생성·bind·unlink ownership은 `05-endpoint-ownership-and-bounded-polling.md`의 주제다. 이 문서는 그 endpoint를 owner availability의 한 조건으로 사용할 뿐이다.
- READY/ACK의 forged source, oversized frame과 invalid flood를 어떻게 거부하는지는 `06-bounded-response-correlation.md`에서 다룬다.

> 검토 범위: `10a7211969bf`, `bc337552961a`, `bdccf91f5a44`, `caf2feec4971`, `f8e8444c5ded`, `e56e8cc87315`, `1e3da4580733`, `a481bfabb7b5`의 diff와 해당 SHA의 server/client/session test source를 확인했다. branch checkout과 test 실행은 수행하지 않았으므로 fixture가 요구하는 결과를 실제 통과 결과처럼 표현하지 않는다.
