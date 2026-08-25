# Thread: 고정 지연에서 상관 검증된 sequence ACK까지

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 01

## 개요

초기 client는 각 bit를 signal로 보낸 뒤 잠시 쉬는 방식으로 receiver가 따라올 시간을 벌었다. 이 지연은 signal 병합 가능성을 낮출 뿐, server가 **어느 bit를 실제로 처리했는지**는 증명하지 못한다. 이후 구현은 signal ACK 기반 stop-and-wait, timeout, datagram 응답 형식, sequence 상관 검증을 차례로 도입하고, 마지막에는 검증된 ACK 자체가 다음 bit 전송 조건이 되면서 고정 지연을 제거한다.

이 Thread가 다루는 불변 조건은 다음과 같다.

- client에는 한 번에 아직 확인되지 않은 bit가 하나만 존재한다.
- 다음 bit로 이동하는 시점은 현재 bit에 대응하는 응답을 수락한 뒤다.
- datagram 응답은 출처 endpoint, server PID, magic, kind, status, 현재 token이 모두 맞을 때만 성공이다.
- 잘못되거나 오래된 응답은 sequence와 bit cursor를 바꾸지 않으며, 원래의 timeout도 연장하지 않는다.

### 최종 전송 흐름

```text
sequence = 0
각 byte를 MSB부터 순회
  └─ bit signal 전송
       └─ 같은 server endpoint에서 온 ACK 중
          magic / server_pid / kind=ACK / token=sequence / status를 모두 검증
            ├─ 일치: sequence++, 다음 bit
            ├─ 불일치: 폐기, 같은 deadline 안에서 계속 대기
            └─ deadline 만료 또는 I/O 실패: 현재 bit에서 전송 중단
```

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `89637d63b56f` | feat(client): 메시지 바이트를 시그널로 전송 | A | `CORE, SIGNAL_DATA` | message byte를 MSB부터 signal bit로 직렬화하고 provisional fixed delay로 전송 속도를 낮춥니다. |
| 2 | `78de95b3cacb` | feat(protocol): 비트 처리마다 ACK 전송 | A | `CORE, SIGNAL_DATA, RISK` | bit마다 signal ACK를 기다리는 stop-and-wait를 도입하고 ACK-before-wait race를 막습니다. |
| 3 | `765efe7b75c9` | feat(client): ACK 대기 시간 초과 처리 | A | `RISK, PROCESS_LIFECYCLE, PRACTICAL` | ACK wait를 alarm 기반 timeout으로 제한하고 timeout과 다른 send failure를 구분합니다. |
| 4 | `342aea9ce9a8` | fix(client): ACK 이후 시그널 전송 간격 안정화 | B | `SIGNAL_DATA, PRACTICAL` | signal ACK 뒤 short inter-signal gap을 유지하고 acknowledgement deadline을 늘립니다. |
| 5 | `4f17de94e025` | fix(client): 인터럽트 뒤 남은 전송 간격 유지 | B | `SIGNAL_DATA, PRACTICAL` | `nanosleep`이 `EINTR`로 중단되면 returned remainder로 같은 logical gap을 이어갑니다. |
| 6 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | `ARCH, RESPONSE` | `ACQUIRE`, `READY`, `ACK`를 표현하는 request/response records와 identity fields를 정의합니다. |
| 7 | `4234233ebd30` | feat(protocol): 비트 ACK를 sequence 응답으로 큐잉 | S | `ARCH, RESPONSE, CORE` | accepted bit에 sequence를 부여하고 datagram ACK send work를 pipe로 queue해 direct signal response에서 분리합니다. |
| 8 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | `RESPONSE, RISK` | client가 expected server source와 exact current sequence의 datagram ACK만 bit success로 수락합니다. |
| 9 | `aeb1b00867f4` | refactor(protocol): 이전 signal ACK 경로 제거 | A | `ARCH, RESPONSE, REFACTOR` | obsolete ACK/NACK signal machinery를 client/server/shared/test sender에서 제거하고 datagram response만 남깁니다. |
| 10 | `1487a861046e` | perf(protocol): 검증된 ACK 뒤 고정 지연 제거 | A | `PERF, RESPONSE` | matching sequence ACK 뒤 fixed sleep을 production client와 session sender에서 제거합니다. |

## `89637d63b56f` — feat(client): 메시지 바이트를 시그널로 전송

**중요도** `A` · **태그** `CORE, SIGNAL_DATA`

### 최초의 data path

`src/client.c`는 message를 별도 문자 인코딩으로 해석하지 않고 byte 배열로 순회한다. 각 byte는 bit 7부터 bit 0까지 전송되며, 0과 1은 서로 다른 user signal에 대응한다.

```c
signal = MT_ZERO_SIGNAL;
if (bit != 0)
    signal = MT_ONE_SIGNAL;
if (kill(server_pid, signal) == -1)
    return (-1);
usleep(150);
```

이 시점의 순서는 명확하다.

```text
message[index]
  → shift 7, 6, ... 0
  → MT_ZERO_SIGNAL 또는 MT_ONE_SIGNAL
  → kill(server_pid, signal)
  → 150µs 대기
  → 다음 bit
```

고정 지연은 receiver가 signal handler를 실행할 시간을 **추정**한다. 하지만 standard signal은 동일 종류의 pending signal을 개수대로 보존하는 queue가 아니므로, sender가 receiver의 처리 상태와 무관하게 계속 전진하면 같은 signal 여러 개가 하나처럼 관찰될 수 있다. `usleep(150)`은 이 가능성을 낮추는 경험적 pacing일 뿐 처리 완료 통지가 아니다.

또한 이 SHA에서는 message byte만 순회하며 종료를 나타내는 NUL byte 전송은 아직 없다. 따라서 server가 하나의 message가 끝났음을 protocol 수준에서 알 수 있는 경계도 완성되지 않았다.

## `78de95b3cacb` — feat(protocol): 비트 처리마다 ACK 전송

**중요도** `A` · **태그** `CORE, SIGNAL_DATA, RISK`

### 시간 추정 대신 stop-and-wait

server가 bit를 처리한 뒤 client에게 `MT_ACK_SIGNAL`을 돌려주고, client는 그 ACK를 받은 다음에만 다음 bit로 이동한다. 이 변경에서 중요한 부분은 handler 설치 자체보다 **ACK를 기다리기 전에 ACK signal을 block하는 순서**다.

```c
sigemptyset(&blocked);
sigaddset(&blocked, MT_ACK_SIGNAL);
if (sigprocmask(SIG_BLOCK, &blocked, &old_mask) == -1)
    return (1);
```

bit 전송에서는 flag를 초기화하고 signal을 보낸 뒤 `sigsuspend`로 기다린다.

```c
g_ack_received = 0;
if (kill(server_pid, signal) == -1)
    return (SEND_ERROR);
while (!g_ack_received)
    sigsuspend(old_mask);
```

ACK를 block하지 않은 채 `kill` 후 flag를 검사하고 sleep한다면 다음 순서가 가능하다.

```text
client: bit signal 전송
server: 즉시 처리하고 ACK 전송
client: ACK handler 실행, flag=1
client: flag 초기화 또는 sleep 진입
       → 이미 지나간 ACK를 다시 받을 수 없어 영구 대기
```

이 commit은 ACK를 미리 block해, 너무 일찍 도착한 ACK를 pending 상태로 남긴다. `sigsuspend(old_mask)`는 mask 교체와 sleep을 원자적으로 수행하므로 lost wakeup을 피한다. message 끝에는 NUL byte도 동일한 8-bit 전송 경로로 보내며, server는 이를 줄바꿈과 세션 종료의 표시로 사용할 수 있게 된다.

### 아직 남은 실패

server가 종료됐거나 ACK 경로가 깨지면 client는 무기한 기다린다. stop-and-wait는 ordering을 만들었지만 liveness 상한은 아직 없다.

## `765efe7b75c9` — feat(client): ACK 대기 시간 초과 처리

**중요도** `A` · **태그** `RISK, PROCESS_LIFECYCLE, PRACTICAL`

`SIGALRM`을 ACK와 함께 기다리는 signal 집합에 넣고, 각 bit마다 alarm을 시작한다.

```c
g_ack_received = 0;
g_timed_out = 0;
if (kill(server_pid, signal) == -1)
    return (SEND_ERROR);
alarm(MT_ACK_TIMEOUT_SECONDS);
while (!g_ack_received && !g_timed_out)
    sigsuspend(old_mask);
alarm(0);
if (g_timed_out)
    return (SEND_TIMEOUT);
```

이 timeout은 signal 전달을 보장하지 않는다. 단지 **현재 bit가 성공으로 확인되지 않은 채 client가 영구 정지하지 않도록** 기다림의 상한을 둔다. `send_bit`가 오류를 반환하면 `send_byte`는 shift를 감소시키지 않으므로 실패한 bit 다음으로 cursor가 넘어가지 않는다. caller는 timeout과 일반적인 `kill` 실패를 서로 다른 diagnostic으로 보고한다.

## `342aea9ce9a8` — fix(client): ACK 이후 시그널 전송 간격 안정화

**중요도** `B` · **태그** `SIGNAL_DATA, PRACTICAL`

ACK가 도입됐어도 ACK handler 복귀 직후 다음 data signal을 보내는 환경에서 관찰된 불안정을 줄이기 위해, 성공한 ACK 뒤에 짧은 간격을 추가하고 ACK deadline을 늘린다.

```diff
-# define MT_ACK_TIMEOUT_SECONDS 1
+# define MT_ACK_TIMEOUT_SECONDS 3
+# define MT_SIGNAL_GAP_US 500
```

이 간격은 **성공 경로**에서만 적용된다. timeout이나 send failure 뒤에 다음 bit를 보내기 위한 pacing으로 사용되지 않는다. 다만 `usleep` 한 번으로 구현되므로 signal에 의해 중단될 경우 요청한 전체 간격이 유지된다는 보장은 없다.

## `4f17de94e025` — fix(client): 인터럽트 뒤 남은 전송 간격 유지

**중요도** `B` · **태그** `SIGNAL_DATA, PRACTICAL`

단일 `usleep`을 remainder-aware `nanosleep` loop로 바꾼다. kernel이 `EINTR`과 함께 돌려준 남은 시간을 다음 호출에 그대로 넘기므로, 여러 signal이 끼어도 하나의 논리적 gap이 조기에 끝나지 않는다.

```c
remaining.tv_sec = 0;
remaining.tv_nsec = MT_SIGNAL_GAP_US * 1000L;
while (nanosleep(&remaining, &remaining) == -1)
{
    if (errno != EINTR)
        break ;
}
```

이 변경은 임시 pacing을 더 정확하게 만들지만, pacing 자체를 protocol proof로 승격시키지는 않는다. non-`EINTR` sleep 오류도 send failure로 전파하지 않는다. 이 지연은 나중에 sequence ACK가 authoritative condition이 되면서 제거된다.

## `ebed06775b92` — feat(protocol): 응답 메시지 wire 형식 정의

**중요도** `A` · **태그** `ARCH, RESPONSE`

signal ACK에는 어떤 요청 또는 bit에 대한 응답인지 담을 payload가 없다. shared header에 control request와 response record를 추가해 identity를 명시한다.

```c
typedef struct s_mt_request
{
    uint32_t    magic;
    uint32_t    kind;
    uint32_t    nonce;
    pid_t       client_pid;
}   t_mt_request;

typedef struct s_mt_response
{
    uint32_t    magic;
    uint32_t    kind;
    uint32_t    token;
    int32_t     status;
    pid_t       server_pid;
}   t_mt_response;
```

field의 역할은 다음과 같다.

| field | 의미 |
| --- | --- |
| `magic` | 이 프로젝트의 응답 record인지 확인 |
| `kind` | `ACQUIRE`, `READY`, `ACK` 구분 |
| `nonce` / `token` | READY에서는 acquisition nonce, ACK에서는 bit sequence |
| `client_pid` / `server_pid` | record 내부에서 주장하는 peer identity |
| `status` | `OK`, `BUSY` 등 결과 |

이 commit은 **형식만 정의**한다. send/receive와 acceptance predicate는 아직 없다. 또한 record를 `sizeof(struct)` 그대로 datagram으로 전송하는 host-local ABI이며, padding·byte order·version을 별도로 직렬화하는 portable network protocol은 아니다.

## `4234233ebd30` — feat(protocol): 비트 ACK를 sequence 응답으로 큐잉

**중요도** `S` · **태그** `ARCH, RESPONSE, CORE`

### 핵심 전환: bit 처리와 응답 identity를 하나의 sequence로 묶기

server는 현재 sequence를 **bit state를 바꾸기 전에** 캡처하고, bit를 처리한 뒤 그 값으로 ACK work를 만든다.

```c
sequence = (uint32_t)g_sequence;
g_current_byte <<= 1;
if (signal == MT_ONE_SIGNAL)
    g_current_byte |= 1;
g_received_bits++;
/* 8 bit이면 byte flush */
if (g_client_pid != 0)
    g_sequence++;
queue_response(info->si_pid, MT_RESPONSE_ACK, sequence, MT_RESPONSE_OK);
```

`queue_response`는 signal handler에서 datagram을 직접 보내지 않고, 고정 크기 내부 record를 pipe에 쓴다.

```c
typedef struct s_response_request
{
    pid_t      client_pid;
    uint32_t   kind;
    uint32_t   token;
    int32_t    status;
}   t_response_request;

request.client_pid = client_pid;
request.kind = kind;
request.token = token;
request.status = status;
if (write(g_response_pipe[1], &request, sizeof(request))
    != (ssize_t)sizeof(request))
    g_response_overflow = 1;
```

normal context의 response loop가 pipe에서 record를 읽어 Unix datagram을 보낸다. 이렇게 sequence 생성과 실제 datagram I/O가 분리되지만, ACK의 token은 bit transition 순간에 이미 결정된다.

client도 같은 cursor를 공유한다.

```c
status = wait_for_response(server_pid, MT_RESPONSE_ACK, *sequence,
        server_path, &deadline);
if (status != 0)
    return (status);
(*sequence)++;
```

따라서 sequence `n`의 ACK가 검증되기 전에는 sequence와 bit cursor 모두 `n`에 머문다.

### 이 SHA는 아직 과도기다

이 commit의 exact tree에서는 datagram ACK가 기존 signal ACK를 즉시 대체하지 않는다.

1. client는 먼저 기존 ACK signal을 기다린다.
2. 이어 같은 bit의 datagram ACK를 sequence로 검증한다.
3. server signal handler는 여전히 session/byte/output state를 직접 바꾼다.
4. handler가 ACK work를 pipe에 넣고, event loop가 datagram 전송을 수행한다.

즉 여기의 pipe는 후속 self-pipe Thread에서 등장하는 **data event pipe**가 아니라, handler가 만든 response send work를 normal context로 넘기는 pipe다. 후대의 `22363f83ff25` 구조를 이 SHA에 소급해서는 안 된다.

### 응답 수락 조건

client의 `read_response`는 단순히 ACK kind만 보지 않는다.

```c
if (!valid_source(&source, server_path)
    || response.magic != MT_RESPONSE_MAGIC
    || response.server_pid != server_pid
    || response.kind != kind
    || response.token != token)
    return (0);        /* 현재 요청과 무관: 폐기하고 계속 대기 */
```

`wait_for_response`는 하나의 monotonic deadline에서 남은 시간을 다시 계산한다. 잘못된 datagram을 읽었다고 deadline을 새로 만들지 않는다.

### 보장과 한계

- 보장: server의 accepted bit transition과 client의 bit cursor를 sequence token으로 연결한다.
- 보장: 오래된 ACK나 다른 kind/token의 response는 다음 bit를 열지 못한다.
- 한계: 이 SHA에는 legacy signal ACK가 함께 남아 success path가 이중화돼 있다.
- 한계: signal handler가 protocol state와 stdout을 직접 변경하는 문제는 별도 Thread에서 해결된다.

## `d3eacbbfeadc` — feat(client): 비트 ACK를 sequence로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK`

client의 bit 성공 조건에서 signal ACK wait를 제거한다. server endpoint를 확인하고 monotonic deadline을 만든 뒤 signal을 보내며, 성공 여부는 datagram ACK만으로 정한다.

```text
server socket 확인
  → deadline = CLOCK_MONOTONIC now + timeout
  → bit signal 전송
  → expected source + ACK kind + current sequence 응답 대기
  → 일치한 경우에만 sequence 증가
```

이 시점에도 client source에는 signal ACK용 handler·mask 일부가 남아 있지만 `send_bit`의 success를 결정하지 않는다. 실제로 obsolete machinery를 삭제하는 작업은 다음 commit이다. 이 구분은 “사용하지 않음”과 “코드에서 제거됨”을 혼동하지 않기 위해 중요하다.

## `aeb1b00867f4` — refactor(protocol): 이전 signal ACK 경로 제거

**중요도** `A` · **태그** `ARCH, RESPONSE, REFACTOR`

client/server/shared header/test sender에서 ACK·NACK signal 상태를 제거하고, datagram response를 유일한 authoritative response path로 남긴다.

이 diff에서 Thread와 직접 관련된 변화는 다음과 같다.

- `MT_ACK_SIGNAL`, `MT_NACK_SIGNAL` 상수 제거
- client의 ACK/rejected/alarm signal handler와 signal mask 제거
- server의 signal ACK/NACK 전송 제거
- first data signal로 owner를 잡던 fallback 제거: acquisition을 완료한 owner의 bit만 처리
- response send 실패 시 해당 session을 reset하는 branch 유지
- test sender도 datagram response만 기다리도록 변경

이 commit은 새 기능보다 **중복된 성공 근거를 없애는 작업**이다. 이후 “bit가 성공했다”는 판단은 하나의 조건식으로 수렴한다.

## `1487a861046e` — perf(protocol): 검증된 ACK 뒤 고정 지연 제거

**중요도** `A` · **태그** `PERF, RESPONSE`

`MT_SIGNAL_GAP_US`, `wait_signal_gap`, test sender의 동일한 sleep을 제거한다.

```diff
-    wait_signal_gap();
     return (0);
```

이 변경은 여러 bit를 동시에 보내도록 pipeline depth를 늘린 것이 아니다. client는 여전히 한 bit를 보내고 정확한 sequence ACK를 기다린다. 달라진 것은 **검증이 끝난 즉시** 다음 bit를 보낼 수 있다는 점이다.

고정 지연 제거가 안전한 근거는 시간 자체가 아니라 다음 순서다.

```text
server: bit n을 accepted state에 반영
server: token n ACK 생성
client: source와 token n을 검증
client: sequence를 n+1로 증가
client: bit n+1 전송
```

ACK datagram이 유실되면 client는 timeout으로 실패하며 다음 bit로 전진하지 않는다. 이 Thread는 재전송이나 exactly-once delivery를 추가하지 않는다.

## 이 Thread의 경계

이 Thread는 **다음 bit로 넘어갈 권한을 어떤 응답이 부여하는가**만 다룬다.

- `ACQUIRE`와 session owner의 생성·회수는 `02-session-ownership-and-recovery.md`의 주제다.
- Unix socket 경로의 생성·bind·unlink 책임과 `FD_SETSIZE`는 `05-endpoint-ownership-and-bounded-polling.md`에서 다룬다.
- signal handler의 work를 self-pipe event loop로 옮기는 변경은 `03-self-pipe-event-loop.md`에서 다룬다.
- stdout 성공을 ACK보다 앞선 commit condition으로 만드는 변경은 `04-output-commit-boundary.md`에서 다룬다.
- forged·oversized·flood response를 포함한 acceptance predicate의 적대적 검증은 `06-bounded-response-correlation.md`에서 더 좁고 깊게 다룬다.

## 조사 범위

각 설명은 표시된 exact SHA의 diff와 그 SHA의 source를 기준으로 작성했다. branch의 final HEAD 구현을 과거 commit에 소급하지 않았다. 이 작업 환경에서는 repository를 checkout해 build와 test script를 실행하지 못했으므로, 테스트 항목은 **test code가 요구하는 관측값**으로만 서술하며 실제 실행 성공을 주장하지 않는다.
