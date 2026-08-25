# Thread: stale·forged 응답 속에서도 bounded correlation 유지하기

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 06

## 개요

Unix datagram 하나가 client socket에 도착했다는 사실만으로 현재 operation이 성공한 것은 아니다. 그 datagram은 이전 bit의 지연된 ACK, 다른 request의 READY, 다른 endpoint에서 보낸 forged record, correct prefix 뒤에 trailing byte가 붙은 frame일 수 있다.

client는 response를 현재 state transition과 연결하기 위해 source endpoint와 record 내부 field를 함께 검증한다. READY에는 acquisition nonce를, bit ACK에는 monotonically 증가하는 sequence를 token으로 사용한다. 유효하지 않은 traffic은 폐기하되, 최초에 만든 monotonic deadline은 그대로 유지해 invalid flood가 wait를 무한히 연장하지 못하게 한다.

### 최종 acceptance 조건

| 조건 | READY | bit ACK |
| --- | --- | --- |
| datagram 크기 | `sizeof(t_mt_response)`와 정확히 같음 | 같음 |
| source | expected server Unix socket path | 같음 |
| `magic` | `MT_RESPONSE_MAGIC` | 같음 |
| `server_pid` | target PID | 같음 |
| `kind` | `MT_RESPONSE_READY` | `MT_RESPONSE_ACK` |
| `token` | current nonzero nonce | current sequence |
| `status` | `OK`이면 성공, `BUSY`이면 거부 | `OK`이어야 성공 |
| wait budget | request 전에 만든 absolute monotonic deadline | bit 전송 전에 만든 absolute monotonic deadline |

하나라도 맞지 않으면 response는 현재 operation의 증거가 아니다. nonce, sequence, bit cursor, deadline은 바뀌지 않는다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `ebed06775b92` | feat(protocol): 응답 메시지 wire 형식 정의 | A | `ARCH, RESPONSE` | request와 특정 bit transition을 식별할 수 있는 request/response fields를 정의합니다. |
| 2 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | `RESPONSE, RISK, INTEGRATION` | READY에 exact size, source, PID, kind, nonce, status와 absolute readiness deadline을 적용합니다. |
| 3 | `d3eacbbfeadc` | feat(client): 비트 ACK를 sequence로 상관 검증 | A | `RESPONSE, RISK` | 현재 sequence와 정확히 일치하는 datagram ACK만 bit success로 인정합니다. |
| 4 | `b361ef9745ff` | test(protocol): 응답 출처와 token 검증 | A | `TEST, RESPONSE, RISK` | valid READY와 first bit ACK 전에 forged-source와 mismatched-field responses를 주입해 conjunctive validation을 검증합니다. |
| 5 | `1ed2acbaa353` | test(response): oversized 응답과 invalid flood 검증 | A | `TEST, RESPONSE, RISK` | response record보다 한 byte 큰 datagram과 sustained wrong-token traffic을 이용해 exact framing과 absolute deadline을 검증합니다. |

## `ebed06775b92` — feat(protocol): 응답 메시지 wire 형식 정의

**중요도** `A` · **태그** `ARCH, RESPONSE`

signal ACK는 payload가 없으므로 “응답이 왔다”는 사실만 표현한다. shared header에 두 fixed record를 추가해 peer와 transition identity를 담는다.

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

### field가 답하는 질문

| field | 질문 |
| --- | --- |
| `magic` | minitalk response 형식인가? |
| `kind` | acquisition READY인가, bit ACK인가? |
| `nonce` / `token` | 현재 어느 request 또는 bit transition에 대한 응답인가? |
| `client_pid` / `server_pid` | record가 주장하는 endpoint process는 누구인가? |
| `status` | operation이 승인됐는가, BUSY인가, 그 밖의 오류인가? |

이 commit에는 send/receive code가 없다. 또한 C struct를 `sizeof` 그대로 local datagram payload로 쓰는 형식이므로 host ABI, alignment, byte order에 의존한다. portable serialized network protocol이나 versioned schema로 설명해서는 안 된다.

## `f8e8444c5ded` — feat(client): READY 응답을 출처와 nonce로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK, INTEGRATION`

### acquisition token 만들기

client는 `/dev/urandom`에서 `uint32_t` 크기만큼 읽고, `EINTR`과 short read를 처리해 nonce를 완성한다. 결과가 0이면 1로 바꾼다. request는 이 nonce와 current PID를 담는다.

```c
request.magic = MT_RESPONSE_MAGIC;
request.kind = MT_REQUEST_ACQUIRE;
request.nonce = nonce;
request.client_pid = getpid();
```

server path를 `sockaddr_un`에 넣어 request를 보내기 전에 `CLOCK_MONOTONIC` 기반 absolute deadline을 한 번 만든다. wall clock 조정이 wait budget을 늘리거나 줄이지 않게 하기 위한 clock 선택이다.

### exact frame과 source 검사

receive buffer는 valid record보다 한 byte 크다.

```c
unsigned char payload[sizeof(t_mt_response) + 1];
size = recvfrom(g_response_socket, payload, sizeof(payload), 0,
        (struct sockaddr *)&source, &source_size);
if (size != (ssize_t)sizeof(t_mt_response))
    return (0);
memcpy(&response, payload, sizeof(response));
```

valid prefix 뒤에 trailing byte가 있으면 `size == sizeof(response) + 1`이므로 폐기된다. 작은 frame도 같은 exact-size check에서 거부된다.

record 수락에는 source path와 내부 field가 모두 필요하다.

```c
if (source.sun_family != AF_UNIX
    || source.sun_path[sizeof(source.sun_path) - 1] != '\0'
    || strcmp(source.sun_path, server_path) != 0
    || response.magic != MT_RESPONSE_MAGIC
    || response.server_pid != server_pid
    || response.kind != kind
    || response.token != token)
    return (0);
if (response.status == MT_RESPONSE_BUSY)
    return (SEND_REJECTED);
if (response.status != MT_RESPONSE_OK)
    return (SEND_ERROR);
return (1);
```

source path만 맞고 내부 PID가 틀린 응답, 내부 field가 모두 맞지만 다른 bound socket에서 온 응답 모두 실패한다. predictable same-UID path를 사용하는 local protocol이므로 이 결합이 cryptographic authentication을 뜻하지는 않는다.

### invalid datagram이 deadline을 갱신하지 않는 구조

`wait_for_response`는 loop마다 새 timeout을 부여하지 않고, 동일한 absolute deadline까지 남은 시간만 다시 계산한다.

```text
deadline = now_monotonic + MT_RESPONSE_TIMEOUT_SECONDS
loop:
  remaining = deadline - now_monotonic
  pselect(..., remaining)
  readable이면 datagram 하나 검사
    valid response → return
    invalid response → 같은 deadline으로 loop
  remaining == 0 → timeout
```

따라서 wrong nonce datagram이 계속 도착해 socket이 즉시 readable 상태가 되어도 총 wait budget은 늘어나지 않는다.

## `d3eacbbfeadc` — feat(client): 비트 ACK를 sequence로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK`

READY에 사용한 같은 response reader를 bit ACK에도 적용한다. signal ACK wait를 success path에서 제거하고 현재 sequence token과 일치하는 datagram만 bit 완료로 인정한다.

```c
status = wait_for_response(server_pid, MT_RESPONSE_ACK, *sequence,
        server_path, &deadline);
if (status != 0)
    return (status);
(*sequence)++;
```

sequence mutation이 response validation **뒤**에 있다는 점이 핵심이다.

| 받은 ACK token | current sequence | 결과 |
| ---: | ---: | --- |
| 같음 | `n` | bit `n` 성공, sequence `n+1` |
| `n-1` | `n` | stale ACK 폐기, cursor 불변 |
| `n+1` | `n` | 미래/무관 ACK 폐기, cursor 불변 |
| correct token, wrong source/PID | `n` | 폐기, cursor 불변 |
| valid ACK 없음 | `n` | deadline에서 실패, 다음 bit 미전송 |

이 commit 직후 obsolete signal handler와 mask code 일부는 source에 남아 있지만 `send_bit`의 success를 결정하지 않는다. 실제 삭제는 `aeb1b00867f4`이며, 여기서는 correlation logic의 도입만 다룬다.

## `b361ef9745ff` — test(protocol): 응답 출처와 token 검증

**중요도** `A` · **태그** `TEST, RESPONSE, RISK`

### 하나씩 틀린 response를 valid response 앞에 배치

`tests/response_server`는 정상 server path와 별도의 `forger` path에 datagram sockets를 bind한다. READY와 첫 bit ACK 전에 다음 records를 순서대로 보낸다.

1. field는 맞지만 `forger` socket에서 보낸 response
2. expected server socket에서 보냈지만 `token + 1`
3. expected source/token이지만 `magic = 0`
4. expected source/token/magic이지만 `server_pid = getpid() + 1`
5. 20ms 뒤 모든 조건이 맞는 response

핵심 test helper는 대략 다음 순서다.

```c
response.magic = MT_RESPONSE_MAGIC;
response.kind = kind;
response.token = token;
response.status = MT_RESPONSE_OK;
response.server_pid = getpid();
send_response(g_forger_socket, client_pid, &response);

response.token = token + 1;
send_response(g_server_socket, client_pid, &response);

response.token = token;
response.magic = 0;
send_response(g_server_socket, client_pid, &response);

response.magic = MT_RESPONSE_MAGIC;
response.server_pid = getpid() + 1;
send_response(g_server_socket, client_pid, &response);

/* 마지막에 valid response */
```

client가 “처음 받은 datagram”이나 “correct source의 첫 datagram”을 성공으로 인정하면 test server와의 message 전송이 깨진다. fixture가 요구하는 정상 completion은 모든 predicate가 conjunction으로 적용된다는 증거다.

이 commit은 wrong kind, non-OK status, truncated frame 전체를 각각 주입하지는 않는다. source, token, magic, internal PID를 집중적으로 검증한다.

## `1ed2acbaa353` — test(response): oversized 응답과 invalid flood 검증

**중요도** `A` · **태그** `TEST, RESPONSE, RISK`

### valid prefix + trailing byte

기존 invalid sequence 앞에 정확히 한 byte 큰 datagram을 추가한다.

```c
unsigned char payload[sizeof(*response) + 1];

memcpy(payload, response, sizeof(*response));
payload[sizeof(*response)] = 0xa5;
sendto(socket_fd, payload, sizeof(payload), 0, ...);
```

앞 `sizeof(response)` byte만 복사해 보면 완전히 valid한 record다. parser가 prefix만 읽거나 oversized datagram을 truncate해 수락하면 현재 transition이 잘못 열린다. production reader의 `sizeof(response) + 1` buffer와 exact-size 비교가 이 frame을 폐기한다.

### invalid traffic이 timeout을 갱신하는지 확인

`MT_TEST_INVALID_FLOOD` mode는 valid server socket에서 다음 READY를 반복 전송한다.

```c
response.magic = MT_RESPONSE_MAGIC;
response.kind = MT_RESPONSE_READY;
response.token = token + 1;      /* 유일한 핵심 mismatch */
response.status = MT_RESPONSE_OK;
response.server_pid = getpid();
```

최대 100000회, 각 전송 사이 100µs를 두고 client endpoint가 없어질 때까지 보낸다. source, magic, kind, status, PID가 모두 맞고 nonce만 틀리므로 socket은 계속 readable하지만 client는 성공할 수 없다.

shell test는 다음을 요구한다.

- client가 nonzero로 끝난다.
- diagnostic은 acknowledgement timeout이다.
- elapsed wall-clock seconds가 2 이상 6 이하이다.
- test server는 client endpoint가 사라진 뒤 정상 종료하고 stderr가 비어 있다.

project timeout은 약 3초다. 매 invalid response 뒤 timeout을 새로 시작했다면 flood가 지속되는 동안 client는 끝나지 않는다. 정해진 범위에서 종료한다는 assertion은 absolute monotonic deadline이 invalid traffic에도 소비되고 있음을 겨냥한다.

이 test의 elapsed check는 `date +%s`의 초 단위 관찰이며 정밀 timer 검증은 아니다. 또한 같은 UID의 hostile process에 대한 보안 증명을 제공하지 않는다. exact framing과 bounded wait라는 두 regression에 집중한다.

## response 상태 전이

```text
[current operation]
  READY: expected token = nonce
  ACK:   expected token = sequence

[datagram receive]
  size != exact record ───────────────┐
  source path mismatch ───────────────┤
  magic/PID/kind/token mismatch ──────┤→ discard
  unknown/non-OK status ──────────────┘   state와 deadline 불변

  all identity fields match
    ├─ BUSY → acquisition rejected
    └─ OK
         ├─ READY → data 전송 시작 가능
         └─ ACK → sequence 증가, 다음 bit 가능

[absolute deadline reached]
  → timeout
  → nonce/sequence/cursor 전진 없음
```

## 보장 범위와 한계

이 Thread가 보장하는 것은 **현재 client가 현재 operation의 응답으로 무엇을 수락하는가**다.

- exact record보다 작거나 큰 datagram은 수락하지 않는다.
- stale, future, wrong-source, inconsistent-PID response는 state를 전진시키지 않는다.
- invalid traffic 양과 무관하게 하나의 operation wait는 원래 deadline 안에 끝난다.
- response 유실 시 재전송은 하지 않는다. client는 timeout으로 실패한다.
- host-local native struct ABI이므로 서로 다른 ABI/byte order의 peer interoperability는 범위 밖이다.
- per-UID filesystem path와 field 검사는 cooperative local identity boundary이며 cryptographic authentication이 아니다.

## 이 Thread의 경계

- sequence ACK가 고정 지연을 대체하는 역사적 흐름은 `01-timing-to-correlated-sequence-acks.md`에서 다룬다.
- ACQUIRE가 owner를 만들고 stale owner를 회수하는 규칙은 `02-session-ownership-and-recovery.md`의 주제다.
- expected server/client path의 생성·bind·cleanup은 `05-endpoint-ownership-and-bounded-polling.md`에서 다룬다.
- signal handler의 event 전달 방식과 output-before-ACK ordering은 각각 Thread 03과 Thread 04에서 다룬다.

## 조사 범위

각 설명은 표시된 exact SHA의 shared header, `src/client.c`, `tests/response_server.c`, shell regression diff를 기준으로 작성했다. adversarial tests는 실행하지 않았으며, source가 구성한 frame 순서와 assertion만 근거로 검증 범위를 서술했다.
