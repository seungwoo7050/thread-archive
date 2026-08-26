# Thread: stdout 반영 성공이 ACK 전송의 commit 조건이다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 04

## 개요

server가 bit를 memory state에 반영했다는 사실과 사용자가 stdout에서 그 byte를 관찰했다는 사실은 같지 않다. output이 실패했는데도 ACK를 보내면 client는 실제로 보이지 않은 data를 성공으로 판단한다. 이 Thread는 low-level `write`를 complete-or-fail primitive로 만든 뒤, payload·terminator·recovery delimiter·startup PID가 성공적으로 기록된 경우에만 protocol이 전진하도록 한다.

첫 두 커밋이 production contract를 만든다. 뒤의 두 test commit은 `EINTR`, short write, zero progress, selected `EPIPE`를 deterministic하게 주입하고, 일반 payload뿐 아니라 abandoned-session recovery newline도 같은 commit boundary에 포함되는지 확인한다.

### 최종 상태

| output 지점 | write 성공 | write 실패 |
| --- | --- | --- |
| startup PID line | event loop 시작 | diagnostic 후 status 1, endpoint cleanup |
| payload byte | decoder/sequence 진행 후 ACK | ACK 없이 server failure |
| NUL terminator newline | session reset 후 ACK | reset/ACK 없이 server failure |
| dead-owner recovery newline | old session reset 후 replacement 처리 | replacement 승인 없이 server failure |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `826dd34c378f` | fix(io): 중단·부분 쓰기를 끝까지 처리 | A | `OUTPUT_COMMIT, PRACTICAL, RISK` | `EINTR`과 short write를 누적 처리하고 zero progress를 `EIO`로 종료하는 `mt_write_all`을 추가한다. |
| 2 | `db2004556d8b` | fix(server): stdout 실패 뒤 ACK 전송 차단 | S | `CORE, OUTPUT_COMMIT, RISK` | protocol-visible output이 성공한 뒤에만 state reset과 ACK가 진행되게 한다. |
| 3 | `9aa80e047514` | test(server): 부분 쓰기와 출력 실패 검증 | A | `TEST, OUTPUT_COMMIT, RISK` | write seam으로 interruption, partial progress, zero write와 selected output failure를 재현한다. |
| 4 | `081a882d7fa3` | test(server): 회수 줄바꿈 출력 실패 검증 | A | `TEST, OUTPUT_COMMIT, SESSION` | dead owner의 partial line을 닫는 newline 실패가 replacement success로 오인되지 않는지 검증한다. |

## 826dd34c378f — fix(io): 중단·부분 쓰기를 끝까지 처리

**중요도** `A` · **태그** `OUTPUT_COMMIT, PRACTICAL, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

기존 helper는 한 번의 `write` 반환만 보고 요청한 buffer 전체가 출력됐다고 간주했다. 그러나 descriptor write는 `EINTR`로 중단되거나 요청보다 적은 positive count를 반환할 수 있다. 반환값 0을 무조건 retry하면 progress 없이 무한 반복할 수도 있다. 따라서 success를 “모든 byte의 누적 전달”로 정의하는 공통 primitive가 필요하다.

### 최종 코드는 어떤 상태를 추적하는가

```c
int mt_write_all(int fd, const void *buffer, size_t size)
{
    const unsigned char *bytes;
    size_t               offset;
    ssize_t              written;

    bytes = (const unsigned char *)buffer;
    offset = 0;
    while (offset < size)
    {
        written = write(fd, bytes + offset, size - offset);
        // interruption은 progress가 없으므로 같은 offset에서 다시 시도한다.
        if (written == -1 && errno == EINTR)
            continue ;
        // EINTR 이외의 terminal error는 caller가 판단하도록 그대로 실패한다.
        if (written == -1)
            return (-1);
        // zero progress를 무한 loop로 숨기지 않는다.
        if (written == 0)
        {
            errno = EIO;
            return (-1);
        }
        // short write도 실제로 전달된 byte만큼은 보존한다.
        offset += (size_t)written;
    }
    return (0);
}
```

같은 commit에서 string/number helper의 raw write도 새 primitive를 사용한다.

```diff
-    write(fd, text, mt_strlen(text));
+    mt_write_all(fd, text, mt_strlen(text));
@@
-        write(fd, &buffer[--index], 1);
+        mt_write_all(fd, &buffer[--index], 1);
```

### 이 커밋이 보장하는 것 / 보장하지 않는 것

`mt_write_all`의 success는 요청한 byte가 모두 descriptor interface에 전달됐다는 뜻이다. `mt_putstr_fd`와 `mt_putnbr_fd`의 반환형은 여전히 `void`이므로 diagnostic output failure를 상위 caller가 관찰하는 API까지 바꾸지는 않는다. protocol-critical output은 `db2004556d8b`이 `mt_write_all` 반환값을 직접 확인한다.

### 관련 커밋과 어떤 관계인가

`db2004556d8b`는 이 complete-or-fail 결과를 payload, newline과 PID publication의 protocol decision에 연결한다. `9aa80e047514`는 loop의 `EINTR`, short count와 zero count branch를 deterministic하게 통과시킨다.

## db2004556d8b — fix(server): stdout 실패 뒤 ACK 전송 차단

**중요도** `S` · **태그** `CORE, OUTPUT_COMMIT, RISK`

### 왜 바뀌었는가 (문제 → 원인 → 결정)

이전 `process_bit`은 8번째 bit를 조립한 뒤 `flush_byte`를 호출했지만 output 결과를 관찰하지 않았다. write가 실패해도 decoder reset, sequence 증가와 ACK 전송이 이어져 client가 false success를 받았다. 결정은 output을 local commit point로 두고, 실패하면 event loop를 종료해 triggering ACK를 보내지 않는 것이다.

```diff
-static void flush_byte(unsigned char output)
+static int flush_byte(unsigned char output)
 {
     if (output == '\0')
     {
-        write(STDOUT_FILENO, "\n", 1);
-        reset_session(0);
+        if (mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
+            return (-1);
+        return (reset_session(0));
     }
-    else
-    {
-        write(STDOUT_FILENO, &output, 1);
-        g_line_started = 1;
-    }
+    if (mt_write_all(STDOUT_FILENO, &output, 1) == -1)
+        return (-1);
+    g_line_started = 1;
+    return (0);
 }
@@
     if (g_received_bits == 8)
     {
         output = g_current_byte;
-        flush_byte(output);
+        if (flush_byte(output) == -1)
+            return (-1);
         g_current_byte = 0;
         g_received_bits = 0;
     }
     if (g_client_pid != 0)
         g_sequence++;
```

`process_bit`이 `-1`을 반환하면 event loop와 `main`이 status 1로 종료한다. ACK call은 이 code 이후에 있으므로 output failure를 success로 publish하지 않는다.

### delimiter도 payload와 같은 commit인 이유는 무엇인가

NUL byte는 visible data가 아니지만 server는 이를 newline과 session release로 materialize한다. newline이 실패한 상태에서 owner를 reset하고 ACK하면 client는 message가 정상적으로 닫혔다고 믿게 된다. 그래서 terminator newline을 먼저 쓰고, 성공한 경우에만 `reset_session(0)`을 수행한다.

abandoned-session recovery도 같다.

```diff
-static void reset_session(int close_partial_line)
+static int reset_session(int close_partial_line)
 {
-    if (close_partial_line && g_line_started)
-        write(STDOUT_FILENO, "\n", 1);
+    if (close_partial_line && g_line_started
+        && mt_write_all(STDOUT_FILENO, "\n", 1) == -1)
+        return (-1);
     g_current_byte = 0;
     g_received_bits = 0;
     g_client_pid = 0;
     g_line_started = 0;
     g_sequence = 0;
+    return (0);
 }
```

recovery newline을 기록하지 못하면 old owner state를 지우지 않고 replacement `READY`도 보내지 않는다.

### startup publication과 `SIGPIPE`는 어떻게 처리되는가

PID와 newline을 하나의 buffer로 만들어 `mt_write_all`하고, 실패하면 event loop를 시작하지 않는다. `SIGPIPE`를 ignore해 closed stdout을 즉시 process termination이 아니라 `write == -1`, `errno == EPIPE`로 관찰하고 같은 failure path로 보낸다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

local stdout commit이 성공하기 전에 ACK를 보내지 않는다. output 성공 후 datagram ACK 자체가 유실되는 경우까지 exactly-once를 제공하지는 않는다. ACK를 못 받은 client가 retry protocol을 갖고 있지 않으므로, 이 Thread의 보장은 “보이지 않은 output을 성공으로 ACK하지 않는다”까지다.

### 관련 커밋과 어떤 관계인가

`9aa80e047514`는 startup, payload와 terminator output을 selected fault로 고정한다. `081a882d7fa3`는 별도의 recovery newline branch가 같은 contract를 지키는지 추가한다.

## 9aa80e047514 — test(server): 부분 쓰기와 출력 실패 검증

**중요도** `A` · **태그** `TEST, OUTPUT_COMMIT, RISK`

### 왜 다른 기법이 필요한가

normal stdout에서는 short write나 특정 byte의 `EPIPE`를 안정적으로 만들기 어렵다. production `mt_write_all`의 call site는 유지하고 fault build에서만 low-level write implementation을 바꾼다.

```diff
+#ifndef MT_WRITE_CALL
+# define MT_WRITE_CALL write
+#endif
@@
-        written = write(fd, bytes + offset, size - offset);
+        written = MT_WRITE_CALL(fd, bytes + offset, size - offset);
```

fault implementation은 environment로 한 번의 `EINTR`, 한 번의 zero return, maximum positive count, selected byte 또는 N번째 newline failure를 구성한다. shell suite는 다음 경로를 분리한다.

```diff
+MT_TEST_ZERO_ONCE=1 "$ROOT/tests/fault_server" \
+    >"$STARTUP_OUT" 2>"$STARTUP_ERR"
+grep -qx 'server: failed to publish pid' "$STARTUP_ERR"
+
+MT_TEST_MAX_WRITE=1 MT_TEST_EINTR_ONCE=1 \
+    "$ROOT/tests/fault_server" >"$PARTIAL_OUT" 2>"$PARTIAL_ERR" &
+"$ROOT/client" "$SERVER_PID" partial
+
+MT_TEST_FAIL_BYTE=X MT_TEST_FAIL_EPIPE=1 \
+    "$ROOT/tests/fault_server" >"$BYTE_OUT" 2>"$BYTE_ERR" &
+"$ROOT/client" "$SERVER_PID" X 2>"$BYTE_CLIENT_ERR" || client_status=$?
```

### 이 테스트가 증명하는 것 / 증명하지 않는 것

`EINTR`와 one-byte short write 뒤에도 full output이 완성되고, zero progress는 startup failure가 되며, selected payload/terminator failure에서는 server와 client가 모두 success를 반환하지 않는다는 regression evidence다. 실제 kernel backpressure, 모든 errno, output 성공 후 ACK loss는 재현하지 않는다.

### 관련 커밋과 어떤 관계인가

`826dd34c378f`의 progress loop와 `db2004556d8b`의 output-before-ACK ordering을 같은 production path에서 검증한다. recovery delimiter는 `081a882d7fa3`이 별도 scenario로 추가한다.

## 081a882d7fa3 — test(server): 회수 줄바꿈 출력 실패 검증

**중요도** `A` · **태그** `TEST, OUTPUT_COMMIT, SESSION`

### 무엇을 검증하는가 (diff)

```diff
+RECOVERY_OUT="$TEST_TMP/recovery.out"
+RECOVERY_ERR="$TEST_TMP/recovery.err"
+RECOVERY_CLIENT_ERR="$TEST_TMP/recovery-client.err"
+MT_TEST_FAIL_NEWLINE_NUMBER=2 MT_TEST_FAIL_EPIPE=1 \
+    "$ROOT/tests/fault_server" >"$RECOVERY_OUT" 2>"$RECOVERY_ERR" &
+SERVER_PID=$!
+wait_ready "$RECOVERY_OUT"
+"$ROOT/tests/session_sender" "$SERVER_PID" partial
+"$ROOT/client" "$SERVER_PID" recovered \
+    2>"$RECOVERY_CLIENT_ERR" || recovery_client_status=$?
+wait "$SERVER_PID" || recovery_server_status=$?
+[ "$recovery_client_status" -ne 0 ]
+[ "$recovery_server_status" -ne 0 ]
+grep -qx 'client: timed out waiting for acknowledgement' \
+    "$RECOVERY_CLIENT_ERR"
```

첫 newline은 startup PID line이고, 두 번째 newline failure를 selected recovery delimiter에 맞춘다. partial owner가 visible `'X'`를 남긴 뒤 replacement client가 acquisition을 시도할 때 `reset_session(1)`이 실패해야 한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

recovery delimiter failure가 old owner reset이나 replacement success로 숨겨지지 않고 server failure와 client timeout으로 나타난다는 exact scenario를 고정한다. newline 번호에 의존하는 test fixture이므로 모든 future output ordering을 일반적으로 증명하지는 않는다.

### 관련 커밋과 어떤 관계인가

`db2004556d8b`가 recovery newline을 fallible commit으로 만든 branch를 직접 겨냥한다. owner availability 자체는 `02-session-ownership-and-recovery.md`가 설명한다.

## 이 Thread의 경계

- 어느 sender가 session owner이고 언제 recovery가 필요한지는 `02-session-ownership-and-recovery.md`의 책임이다. 이 문서는 recovery output이 성공해야 state를 release할 수 있다는 조건만 다룬다.
- signal event를 self-pipe로 이동하고 overflow를 fail-stop으로 처리하는 구조는 `03-self-pipe-event-loop.md`가 다룬다.
- socket path·descriptor cleanup은 `05-endpoint-ownership-and-bounded-polling.md`의 주제다. 이 문서에서는 output failure 뒤 registered cleanup이 실행된다는 연결만 확인한다.
- output 성공 후 ACK datagram 유실, retransmission과 exactly-once delivery는 별개의 protocol 문제다.

> 검토 범위: `826dd34c378f`, `db2004556d8b`, `9aa80e047514`, `081a882d7fa3`의 diff와 해당 SHA의 `src/write_utils.c`, `src/server.c`, fault build seam, `tests/output_failure.sh`를 확인했다. branch checkout, fault binary와 shell suite 실행은 수행하지 않았으므로 assertion을 실제 통과 결과로 주장하지 않는다.
