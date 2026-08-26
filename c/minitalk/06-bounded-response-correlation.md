# Thread: 응답은 현재 operation과 일치할 때만 원래 deadline 안에서 수락된다
> Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 06

## 개요

Unix datagram이 client socket에 도착했다는 사실만으로 현재 operation이 성공한 것은 아니다. record는 이전 bit의 stale ACK, 다른 acquisition의 READY, 다른 endpoint에서 온 forged datagram, valid prefix 뒤에 trailing byte가 붙은 oversized frame일 수 있다. 따라서 response acceptance는 source와 record 내부 identity를 모두 만족하는 conjunction이어야 한다.

이 Thread는 shared record를 정의한 뒤 READY에는 random nonce, bit ACK에는 monotonically 증가하는 sequence를 token으로 연결한다. invalid datagram은 state를 바꾸지 않고 폐기하지만, wait budget은 매 receive마다 새로 만들지 않는다. request 또는 bit 전송 전에 계산한 absolute `CLOCK_MONOTONIC` deadline을 계속 재사용해 invalid flood가 timeout을 연장하지 못하게 한다.

### 최종 acceptance 조건

| 조건 | READY | bit ACK |
| --- | --- | --- |
| frame length | 정확히 `sizeof(t_mt_response)` | 동일 |
| source | expected server Unix socket path | 동일 |
| record identity | `magic`, target `server_pid` | 동일 |
| operation kind | `MT_RESPONSE_READY` | `MT_RESPONSE_ACK` |
| token | current nonzero nonce | current sequence |
| status | `OK` 성공, `BUSY` 거부, 그 밖은 error | `OK`만 성공 |
| wait budget | ACQUIRE 전 만든 absolute deadline | bit signal 전 만든 absolute deadline |

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | `ARCH, RESPONSE` | request와 response에 peer·operation·token identity를 표현할 fixed record를 정의한다. |
| 2 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | `RESPONSE, RISK, INTEGRATION` | exact source·PID·kind·nonce·status와 absolute deadline을 acquisition wait에 적용한다. |
| 3 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | `RESPONSE, RISK` | 현재 sequence의 datagram ACK만 bit success로 인정한다. |
| 4 | `b361ef9745ff` | test(protocol): 응답 출처와 token 검증 | A | `TEST, RESPONSE, RISK` | valid response 앞에 forged source와 field별 mismatch를 넣어 conjunctive validation을 검증한다. |
| 5 | `1ed2acbaa353` | test(response): oversized 응답과 invalid flood 검증 | A | `TEST, RESPONSE, RISK` | trailing byte frame과 지속적인 wrong-token traffic으로 exact framing과 bounded wait를 검증한다. |

---

**역할군: response가 current operation을 식별할 표현과 acceptance predicate를 만들기**

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

`magic`은 record family, `kind`는 acquisition과 bit ACK, PID는 claimed peer, nonce/token은 current transition, `status`는 success와 `BUSY`를 표현한다. 이 SHA는 schema만 정의하고 send/receive validation은 아직 없다.

raw C struct를 `sizeof` 그대로 local datagram에 사용하므로 padding, native byte order와 `pid_t` representation에 의존한다. portable serialized protocol이나 cryptographic authentication으로 해석해서는 안 된다.

### 관련 커밋과 어떤 관계인가

`f8e8444c5ded`은 request nonce를 echoed READY token으로 사용하고, `d3eacbbfeadc`는 response token을 current bit sequence로 해석한다. 두 acceptance path가 같은 record를 서로 다른 operation identity에 연결한다.

## f8e8444c5ded — feat(client): READY 응답을 출처와 nonce로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK, INTEGRATION`

### 무엇을 만들었는가 (diff)

client는 response보다 한 byte 큰 receive buffer를 사용하고 exact record size만 구조체로 복사한다.

```diff
+static int read_response(pid_t server_pid, uint32_t kind, uint32_t token,
+        const char *server_path)
+{
+    unsigned char      payload[sizeof(t_mt_response) + 1];
+    struct sockaddr_un source;
+    t_mt_response      response;
+    ssize_t            size;
+
+    size = recvfrom(g_response_socket, payload, sizeof(payload), 0,
+            (struct sockaddr *)&source, &source_size);
+    if (size == -1 && (errno == EAGAIN || errno == EWOULDBLOCK
+            || errno == EINTR))
+        return (0);
+    if (size == -1)
+        return (SEND_ERROR);
+    if (size != (ssize_t)sizeof(response))
+        return (0);
+    memcpy(&response, payload, sizeof(response));
```

valid record는 source path와 내부 fields를 모두 만족해야 한다.

```diff
+    if (!valid_source(&source, server_path)
+        || response.magic != MT_RESPONSE_MAGIC
+        || response.server_pid != server_pid || response.kind != kind
+        || response.token != token)
+        return (0);
+    if (response.status == MT_RESPONSE_BUSY)
+        return (SEND_REJECTED);
+    if (response.status != MT_RESPONSE_OK)
+        return (SEND_ERROR);
+    return (-1);
+}
```

여기서 `0`은 “아직 current operation의 응답이 아님”, `-1`은 internal sentinel로 “valid success”, positive value는 client-visible failure다. caller `wait_for_response`가 sentinel을 0 success로 변환한다.

### deadline은 어떻게 bounded wait를 만드는가

`request_session`은 nonzero nonce 생성, server socket validation과 `clock_gettime(CLOCK_MONOTONIC, &deadline)`을 ACQUIRE 전 한 번 수행하고 timeout seconds를 더한다. wait loop는 매 iteration마다 current time과 **같은 absolute deadline**의 차이를 계산한다.

```diff
+    while (1)
+    {
+        status = read_response(server_pid, kind, token, server_path);
+        if (status != 0)
+        {
+            if (status == -1)
+                return (0);
+            return (status);
+        }
+        remaining = time_until(deadline);
+        if (remaining.tv_sec == 0 && remaining.tv_nsec == 0)
+            return (SEND_TIMEOUT);
+        status = pselect(g_response_socket + 1, &read_set, NULL, NULL,
+                &remaining, NULL);
+        if (status == -1 && errno != EINTR)
+            return (SEND_ERROR);
+    }
```

invalid frame은 nonce, operation kind와 deadline을 바꾸지 않는다. receive가 반복돼도 새 timeout budget을 만들지 않는다.

### 이 커밋이 보장하는 것 / 보장하지 않는 것

READY는 exact frame, source, server PID, kind, nonce와 status가 맞아야만 acquisition success다. same-UID predictable path는 local integrity check일 뿐 secret-based peer authentication은 아니다. random nonce도 cryptographic session protocol 전체를 제공하지 않는다.

### 관련 커밋과 어떤 관계인가

`d3eacbbfeadc`가 같은 predicate와 absolute-deadline machinery를 bit ACK에 적용한다. `b361ef9745ff`와 `1ed2acbaa353`은 각각 field mismatch와 framing/deadline failure를 분리해 검증한다.

## d3eacbbfeadc — feat(client): 비트 ACK를 sequence로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK`

### 무엇이 바뀌었는가 (diff)

signal ACK/NACK/alarm wait를 `send_bit`에서 제거하고, bit signal을 보내기 전에 current sequence의 absolute deadline을 만든다.

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
     if (clock_gettime(CLOCK_MONOTONIC, &deadline) == -1)
         return (SEND_ERROR);
     deadline.tv_sec += MT_ACK_TIMEOUT_SECONDS;
+    if (kill(server_pid, signal) == -1)
+        return (SEND_ERROR);
     status = wait_for_response(server_pid, MT_RESPONSE_ACK, sequence,
             server_path, &deadline);
```

`send_byte`는 `send_bit(..., *sequence, ...)`가 성공한 뒤에만 `(*sequence)++`를 수행한다. wrong token, stale ACK, forged source는 current bit의 success가 아니므로 bit cursor와 sequence가 그대로다.

### 무엇을 준비하고 아직 어디까지인가

이 commit으로 production bit success는 datagram correlation에 연결된다. source file에 남은 obsolete signal handler와 mask machinery를 실제로 삭제하는 작업은 `aeb1b00867f4`이며, 그 정리와 pacing 제거의 history는 `01-timing-to-correlated-sequence-acks.md`가 다룬다.

### 관련 커밋과 어떤 관계인가

`b361ef9745ff`의 response server는 first bit ACK 전에 forged-source와 wrong-field records를 넣어, sequence가 valid ACK 전에 전진하지 않는지 확인한다.

---

**역할군: conjunctive validation과 bounded wait를 adversarial input으로 고정하기**

## b361ef9745ff — test(protocol): 응답 출처와 token 검증

**중요도** `A` · **태그** `TEST, RESPONSE, RISK`

### 무엇을 검증하는가

전용 `response_server`는 valid READY와 first bit ACK를 바로 보내지 않는다. 먼저 각 identity 요소가 하나씩 틀린 response를 client socket에 넣고, 짧은 pause 뒤 exact valid response를 보낸다.

```diff
+    response.magic = MT_RESPONSE_MAGIC;
+    response.kind = kind;
+    response.token = token;
+    response.status = MT_RESPONSE_OK;
+    response.server_pid = getpid();
+    if (send_response(g_forger_socket, client_pid, &response) == -1)
+        return (-1);
+    response.token = token + 1;
+    if (send_response(g_server_socket, client_pid, &response) == -1)
+        return (-1);
+    response.token = token;
+    response.magic = 0;
+    if (send_response(g_server_socket, client_pid, &response) == -1)
+        return (-1);
+    response.magic = MT_RESPONSE_MAGIC;
+    response.server_pid = getpid() + 1;
+    if (send_response(g_server_socket, client_pid, &response) == -1)
+        return (-1);
+    /* ... */
```

같은 sequence가 READY와 first ACK 양쪽에 적용된다. client가 invalid response 하나라도 성공으로 받아들이면 sender/server의 byte assembly와 expected output이 어긋나거나 client가 잘못 종료한다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

selected forged source, wrong token, bad magic와 wrong server PID를 각각 무시하고 뒤의 valid response를 기다린다는 conjunctive validation evidence다. oversized frame, 지속적인 invalid traffic과 timeout budget은 아직 다루지 않는다. same-UID path spoofing 전체를 security proof로 만들지도 않는다.

### 관련 커밋과 어떤 관계인가

`f8e8444c5ded`의 READY predicate와 `d3eacbbfeadc`의 ACK predicate를 같은 adversarial server로 통과시킨다. `1ed2acbaa353`은 field 값이 아니라 frame length와 wait budget을 공격한다.

## 1ed2acbaa353 — test(response): oversized 응답과 invalid flood 검증

**중요도** `A` · **태그** `TEST, RESPONSE, RISK`

### 왜 다른 기법이 필요한가

한두 개의 invalid response 뒤 valid response를 보내는 test는 validation predicate는 확인하지만, invalid traffic이 계속 도착할 때 wait budget을 새로 시작하는 bug는 잡지 못한다. 또한 field가 전부 valid인 prefix 뒤 trailing byte가 붙은 frame은 값 비교만으로 구별되지 않는다.

### exact frame boundary는 어떻게 검증하는가

```diff
+static int send_oversized_response(int socket_fd, pid_t client_pid,
+        const t_mt_response *response)
+{
+    unsigned char payload[sizeof(*response) + 1];
+    /* ... */
+    memcpy(payload, response, sizeof(*response));
+    payload[sizeof(*response)] = 0xa5;
+    if (sendto(socket_fd, payload, sizeof(payload), 0,
+            (struct sockaddr *)&address, sizeof(address))
+        != (ssize_t)sizeof(payload))
+        return (-1);
+    return (0);
+}
```

record prefix가 모두 맞아도 received size가 `sizeof(t_mt_response) + 1`이므로 `read_response`는 구조체로 해석하지 않고 폐기해야 한다.

### invalid flood가 deadline을 늘리는지 어떻게 확인하는가

response server는 correct source에서 wrong nonce READY를 반복 전송한다.

```diff
+    response.magic = MT_RESPONSE_MAGIC;
+    response.kind = MT_RESPONSE_READY;
+    response.token = token + 1;
+    response.status = MT_RESPONSE_OK;
+    response.server_pid = getpid();
+    while (tries < 100000 && lstat(path, &info) == 0)
+    {
+        (void)sendto(g_server_socket, &response, sizeof(response), 0,
+                (struct sockaddr *)&address, sizeof(address));
+        pause_time.tv_nsec = 100000L;
+        while (nanosleep(&pause_time, &pause_time) == -1 && errno == EINTR)
+            ;
+        tries++;
+    }
```

shell fixture는 client가 timeout diagnostic과 nonzero status를 내고, wall-clock elapsed time이 2초 이상 6초 이하인지 확인한다. invalid response가 올 때마다 3초 budget을 새로 만들면 flood가 끝날 때까지 client가 살아 있어 이 상한을 넘는다.

### 이 테스트가 증명하는 것 / 증명하지 않는 것

valid prefix에 trailing byte가 붙은 frame을 거부하고, sustained wrong-token traffic 아래에서도 original deadline이 만료된다는 selected regression evidence다. wall-clock assertion은 test scheduling 여유를 포함한 간접 관찰이며 모든 load environment에서 정밀한 latency guarantee를 뜻하지 않는다. datagram truncation flag, credentials ancillary data와 cryptographic authentication은 범위 밖이다.

### 관련 커밋과 어떤 관계인가

`f8e8444c5ded`의 `payload[sizeof(response) + 1]`, exact size branch와 absolute deadline reuse가 모두 유지돼야 두 scenario가 성립한다. `b361ef9745ff`가 field-level conjunction을, 이 커밋이 framing과 liveness bound를 보완한다.

## 이 Thread의 경계

- signal ACK에서 sequence datagram ACK로 이동하고 obsolete timing을 제거하는 history는 `01-timing-to-correlated-sequence-acks.md`가 다룬다. 이 문서는 최종 response acceptance predicate에 집중한다.
- 어느 client가 session을 소유하는지는 `02-session-ownership-and-recovery.md`의 책임이다. nonce-correlated READY는 그 ownership decision을 전달하는 response일 뿐이다.
- server/client endpoint path의 생성·bind·unlink ownership은 `05-endpoint-ownership-and-bounded-polling.md`가 다룬다.
- predictable same-UID path와 random nonce를 넘어선 peer credential 검증, MAC·encryption·portable serialization은 별개의 security/protocol 문제다.

> 검토 범위: `ebed06775b92`, `f8e8444c5ded`, `d3eacbbfeadc`, `b361ef9745ff`, `1ed2acbaa353`의 diff와 해당 SHA의 shared header, `src/client.c`, `tests/response_server.c`, response-validation/protocol-regression scripts를 확인했다. branch checkout과 adversarial test 실행은 수행하지 않았으므로 elapsed-time assertion을 실제 측정 결과처럼 주장하지 않는다.
