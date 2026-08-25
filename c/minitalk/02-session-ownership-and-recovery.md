# Thread: 암묵적 첫 bit 소유권에서 endpoint-aware 세션 회수까지

Project: `minitalk` · Branch: `c/minitalk` · 문서 번호: 02

## 개요

server에는 message마다 별도의 decoder가 있는 것이 아니라 하나의 `current_byte`, `received_bits`, `sequence`, visible-line 상태가 있다. 두 sender의 bit가 이 상태에 번갈아 들어오면 각각은 정상 signal이어도 결과 byte는 어느 message에도 속하지 않는다. 따라서 server는 현재 decoder를 사용할 수 있는 sender를 하나로 제한해야 한다.

이 Thread는 세션 소유권이 다음 세 단계로 강화되는 과정을 다룬다.

1. 첫 data signal을 보낸 PID가 암묵적으로 owner가 된다.
2. data 전 `ACQUIRE`/`READY` 교환으로 owner를 명시적으로 예약한다.
3. owner의 PID 존재뿐 아니라 응답 endpoint의 사용 가능성까지 확인해 zombie와 stale session을 회수한다.

### 최종 불변 조건

- 검증된 `ACQUIRE`가 성공하면 payload가 아직 하나도 없어도 `g_client_pid`가 예약된다.
- current owner가 아닌 sender의 bit는 accumulator를 변경하지 않는다.
- live owner가 있으면 competitor는 `BUSY`를 받으며 server output도 바뀌지 않는다.
- owner가 더 이상 사용 가능하지 않으면 partial decoder state와 sequence를 함께 reset한다.
- 이미 출력된 partial line이 있으면 replacement를 허용하기 전에 줄바꿈으로 닫는다. 이 줄바꿈이 실패하면 owner를 성공적으로 회수한 것으로 처리하지 않는다.

### 커밋 목록

| 순서 | SHA | 제목 | 중요도 | 태그 | 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `10a7211969bf` | fix(server): 활성 세션에 다른 송신자 거부 | A | `SESSION, RISK` | active sender PID를 기록하고 NUL terminator까지 다른 sender의 data signal을 거부합니다. |
| 2 | `bc337552961a` | fix(server): 종료된 송신자의 세션 복구 | A | `SESSION, PROCESS_LIFECYCLE, RISK` | recorded owner가 사라지면 partial byte, bit count, owner, line state를 함께 reset하고 new sender가 진행할 수 있게 합니다. |
| 3 | `bdccf91f5a44` | test(server): 중단·경쟁 송신자 세션 검증 | A | `TEST, SESSION, RISK` | dedicated sender로 one-bit abandon, complete-byte-plus-partial abandon, live competition, owner exit recovery를 재현합니다. |
| 4 | `caf2feec4971` | feat(server): 획득 요청을 검증해 세션 소유권 예약 | S | `ARCH, SESSION, CORE` | exact `ACQUIRE` datagram을 검증한 뒤 data 전에 owner를 예약하고 `READY` 또는 `BUSY`를 반환합니다. |
| 5 | `f8e8444c5ded` | feat(client): READY 응답을 출처와 nonce로 상관 검증 | A | `RESPONSE, RISK, INTEGRATION` | client가 nonzero nonce ACQUIRE를 보내고 expected source, PID, fields, deadline이 맞는 READY만 수락합니다. |
| 6 | `e56e8cc87315` | test(session): 데이터 없는 활성 예약 경쟁 검증 | A | `TEST, SESSION` | session helper의 `reserve` mode가 acquisition만 완료하고 data를 보내지 않은 채 live owner를 유지합니다. |
| 7 | `1e3da4580733` | fix(server): 응답 경로가 사라진 세션 소유자 회수 | S | `SESSION, PROCESS_LIFECYCLE, DEBUG` | owner availability를 process presence와 expected same-UID client response socket usability의 결합으로 정의합니다. |
| 8 | `a481bfabb7b5` | test(session): 종료 송신자 회수 전 새 세션 복구 검증 | A | `TEST, SESSION, PROCESS_LIFECYCLE` | partial-session child를 exit시킨 뒤 `waitid(..., WNOWAIT)`로 unreaped 상태를 유지해 zombie PID와 vanished endpoint를 동시에 관측합니다. |

## `10a7211969bf` — fix(server): 활성 세션에 다른 송신자 거부

**중요도** `A` · **태그** `SESSION, RISK`

### 첫 bit가 owner를 만든다

server는 첫 accepted signal의 `siginfo_t.si_pid`를 `g_client_pid`에 저장한다. 이후 NUL terminator로 message가 끝날 때까지 다른 PID의 signal은 accumulator mutation 전에 거부한다.

```c
if (g_client_pid == 0)
    g_client_pid = info->si_pid;
if (g_client_pid != info->si_pid)
{
    kill(info->si_pid, MT_NACK_SIGNAL);
    errno = saved_errno;
    return ;
}
```

이 순서에서 중요한 것은 competitor가 다음 코드에 도달하지 못한다는 점이다.

```c
g_current_byte <<= 1;
if (signal == MT_ONE_SIGNAL)
    g_current_byte |= 1;
g_received_bits++;
```

따라서 owner A가 3 bit를 보낸 상태에서 B의 signal이 도착해도 A의 partial byte에는 섞이지 않는다. client는 NACK를 별도 상태로 기록해 “server가 다른 sender를 처리 중”이라는 오류로 반환한다.

### 한계

소유권은 첫 data signal이 들어오기 전에는 존재하지 않는다. 두 client가 거의 동시에 시작하면 control-plane reservation 없이 첫 signal 도착 순서가 owner를 정한다. 또한 owner가 중간에 종료되면 `g_client_pid`가 남아 새 sender가 계속 거부된다.

## `bc337552961a` — fix(server): 종료된 송신자의 세션 복구

**중요도** `A` · **태그** `SESSION, PROCESS_LIFECYCLE, RISK`

다른 sender가 들어왔을 때 recorded owner에 `kill(owner, 0)`을 보내고, `ESRCH`이면 dead session으로 판단한다.

```c
if (g_client_pid != 0 && g_client_pid != info->si_pid)
{
    if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
        reset_session(1);
    else
    {
        kill(info->si_pid, MT_NACK_SIGNAL);
        errno = saved_errno;
        return ;
    }
}
```

`reset_session`은 owner PID 하나만 지우지 않는다.

```c
if (close_partial_line && g_line_started)
    write(STDOUT_FILENO, "\n", 1);
g_current_byte = 0;
g_received_bits = 0;
g_client_pid = 0;
g_line_started = 0;
```

완성된 byte가 하나라도 stdout에 보인 뒤 owner가 사라졌다면, replacement message가 그 뒤에 같은 줄로 붙지 않도록 newline을 먼저 쓴다. 아직 8 bit가 되지 않은 `g_current_byte`는 출력하지 않고 버린다.

ACK 전송 자체가 `ESRCH`로 실패한 경우에도 session을 reset해 다음 sender가 복구할 수 있게 한다. signal handler는 내부 probe와 ACK/NACK 전송이 바꾼 `errno`를 caller context에 누출하지 않도록 저장·복원한다.

### PID-only liveness의 잔여 문제

`kill(pid, 0)`은 프로세스가 종료됐지만 parent가 아직 wait하지 않은 zombie에도 성공할 수 있다. 따라서 “PID가 존재한다”와 “client가 protocol 응답을 받을 수 있다”는 아직 같은 것으로 취급된다. 이 가정은 `1e3da4580733`에서 깨진다.

## `bdccf91f5a44` — test(server): 중단·경쟁 송신자 세션 검증

**중요도** `A` · **태그** `TEST, SESSION, RISK`

`tests/session_sender`가 정상 client로 만들기 어려운 중간 상태를 직접 만든다.

| mode | 만든 상태 | 뒤따르는 assertion |
| --- | --- | --- |
| `bit` | 한 bit만 ACK 받은 뒤 종료 | 다음 정상 client가 성공해야 함 |
| `partial` | `'X'` 한 byte와 다음 byte의 3 bit를 보낸 뒤 종료 | server가 `X\n`으로 partial line을 닫고 다음 message를 받아야 함 |
| `hold` | partial 상태를 만든 뒤 살아서 `pause()` | competitor는 BUSY, output 불변 |

live holder를 종료한 뒤에는 replacement가 성공해야 한다. 이 test는 owner가 살아 있는 동안의 exclusivity와 owner 종료 뒤 recovery를 같은 process-level 시나리오에서 비교한다.

이 시점의 test는 signal ACK 시대의 first-bit ownership을 검증한다. 아직 `ACQUIRE`만 하고 data를 보내지 않는 owner는 표현하지 못한다.

## `caf2feec4971` — feat(server): 획득 요청을 검증해 세션 소유권 예약

**중요도** `S` · **태그** `ARCH, SESSION, CORE`

### 소유권 결정을 data channel에서 control channel로 이동

server는 Unix datagram으로 받은 `t_mt_request`를 owner assignment 전에 검증한다.

```c
if (request->magic != MT_RESPONSE_MAGIC
    || request->kind != MT_REQUEST_ACQUIRE || request->client_pid <= 1
    || source->sun_family != AF_UNIX
    || source->sun_path[sizeof(source->sun_path) - 1] != '\0'
    || mt_response_path(client_path, sizeof(client_path), "client",
        request->client_pid) == -1
    || strcmp(source->sun_path, client_path) != 0
    || valid_client_socket(client_path) == -1
    || kill(request->client_pid, 0) == -1)
    return (0);
```

acceptance에는 다음 정보가 함께 필요하다.

- exact request 크기
- protocol magic과 `ACQUIRE` kind
- 유효한 claimed client PID
- datagram source path가 그 PID에서 계산한 client path와 일치
- 해당 path가 current UID 소유의 Unix socket
- claimed process가 존재

검증 후 owner 처리 순서는 다음과 같다.

```c
status = MT_RESPONSE_OK;
if (g_client_pid != 0 && g_client_pid != request.client_pid)
{
    if (kill((pid_t)g_client_pid, 0) == -1 && errno == ESRCH)
        reset_session(1);
    else
        status = MT_RESPONSE_BUSY;
}
if (status == MT_RESPONSE_OK && g_client_pid == 0)
{
    g_client_pid = request.client_pid;
    new_owner = 1;
}
```

`READY`를 보내기 전에 owner가 먼저 publish된다. 그렇지 않으면 client가 READY를 받고 즉시 bit를 보내는 동안 server가 아직 free로 보이는 race가 생길 수 있다. 반대로 newly reserved owner에게 READY 전송이 실패하면 `reset_session(0)`으로 예약을 roll back한다.

### exact SHA의 과도기 상태

이 commit은 explicit acquisition을 추가하지만 first-bit capture를 같은 diff에서 완전히 제거하지는 않는다. old data handler의 fallback은 뒤의 `aeb1b00867f4`에서 삭제된다. 따라서 이 시점에 이미 “소유권은 오직 ACQUIRE로만 생긴다”고 설명하면 후속 구현을 소급한 것이 된다.

## `f8e8444c5ded` — feat(client): READY 응답을 출처와 nonce로 상관 검증

**중요도** `A` · **태그** `RESPONSE, RISK, INTEGRATION`

client는 payload를 보내기 전에 자신의 response socket을 만들고, `/dev/urandom`에서 32-bit nonce를 정확히 읽는다. 0이 나오면 1로 바꿔 nonzero token을 보장한다.

```c
request.magic = MT_RESPONSE_MAGIC;
request.kind = MT_REQUEST_ACQUIRE;
request.nonce = nonce;
request.client_pid = getpid();
```

request를 server endpoint로 보낸 뒤 `READY`를 기다린다. 수락되는 응답은 expected server path에서 왔고, record의 magic·server PID·kind·token이 현재 request와 일치해야 한다. `BUSY`는 정상 READY와 구분된 rejection으로 caller에 전달된다.

여기서 nonce는 같은 PID의 이전 acquisition response나 지연된 datagram이 현재 예약을 열지 못하게 한다. 자세한 acceptance predicate와 absolute deadline은 Thread 06에서 다룬다.

## `e56e8cc87315` — test(session): 데이터 없는 활성 예약 경쟁 검증

**중요도** `A` · **태그** `TEST, SESSION`

`session_sender`의 `hold` mode를 `reserve` mode로 바꾼다. `reserve`는 ACQUIRE/READY까지만 완료하고 bit는 하나도 보내지 않은 채 살아 있는다.

```c
else if (strcmp(argv[2], "reserve") != 0)
    return (1);
if (strcmp(argv[2], "reserve") == 0)
{
    write(STDOUT_FILENO, "ready\n", 6);
    while (1)
        pause();
}
```

이 helper가 살아 있는 동안 정상 client는 BUSY여야 하며 server output은 reservation 전과 같아야 한다. reserver를 종료한 뒤 replacement가 성공해야 한다. expected output에서 이전 `hold`가 만들던 `X\n`이 사라진 것도 중요하다. **세션 소유권은 visible data의 존재가 아니라 acquisition 성공으로 시작한다.**

## `1e3da4580733` — fix(server): 응답 경로가 사라진 세션 소유자 회수

**중요도** `S` · **태그** `SESSION, PROCESS_LIFECYCLE, DEBUG`

### 실패한 가정

종료한 child가 zombie로 남아 있으면 PID table entry가 아직 존재하므로 `kill(pid, 0)`은 성공한다. 그러나 client의 normal exit cleanup은 자신의 Unix response socket을 이미 unlink했다. server 관점에서 이 owner는 PID만 있고 ACK/READY를 받을 endpoint는 없다.

### availability를 두 자원으로 정의

```c
static int session_owner_available(void)
{
    char client_path[MT_RESPONSE_PATH_SIZE];

    if (g_client_pid <= 1)
        return (0);
    if (kill(g_client_pid, 0) == -1 && errno == ESRCH)
        return (0);
    if (mt_response_path(client_path, sizeof(client_path), "client",
            g_client_pid) == -1 || valid_client_socket(client_path) == -1)
        return (0);
    return (1);
}
```

competitor acquisition은 PID-only check 대신 이 helper를 사용한다.

```c
if (!session_owner_available())
{
    if (reset_session(1) == -1)
        return (-1);
}
else
    status = MT_RESPONSE_BUSY;
```

availability는 다음 결합이다.

| process probe | expected client socket | 판단 |
| --- | --- | --- |
| `ESRCH` | 어떤 상태든 | unavailable |
| PID 존재 | 같은 UID의 valid socket 존재 | available |
| PID 존재 | path 없음 또는 invalid | unavailable |

endpoint는 cryptographic process identity proof가 아니다. 같은 UID의 협력적 local namespace 안에서 “이 owner에게 response를 전달할 수 있는가”를 판단하는 operational resource다.

`reset_session(1)`이 partial-line newline을 쓰지 못하면 owner state를 지우지 않고 오류를 event loop로 전파한다. 출력 경계를 만들지 못한 채 replacement에 READY를 보내는 것보다 server를 실패시키는 결정이다.

## `a481bfabb7b5` — test(session): 종료 송신자 회수 전 새 세션 복구 검증

**중요도** `A` · **태그** `TEST, SESSION, PROCESS_LIFECYCLE`

`tests/unreaped_exec`는 이 bug를 의도적으로 만든다.

1. child가 `session_sender ... partial`을 실행해 partial session을 만든다.
2. child는 정상 종료하며 자신의 response socket을 unlink한다.
3. parent는 `waitid(P_PID, child, ..., WEXITED | WNOHANG | WNOWAIT)`로 종료를 확인하되 reap하지 않는다.
4. test는 `kill -0 child`가 여전히 성공하는지 확인한다.
5. 새 client가 같은 server에서 `unreaped recovered`를 보내야 한다.

```c
if (waitid(P_PID, (id_t)child, &info,
        WEXITED | WNOHANG | WNOWAIT) == -1)
    return (-1);
return (info.si_pid == child);
```

이 fixture는 “PID가 존재한다”는 관찰과 “client endpoint가 사라졌다”는 관찰을 동시에 유지한다. expected server output에는 abandoned owner가 남긴 `X\n`과 replacement message가 모두 들어간다.

이 test는 same-UID adversary가 예상 path에 별도 socket을 만드는 상황이나 PID 재사용 전체를 증명하지 않는다. 검증 대상은 zombie 때문에 PID-only liveness가 stale ownership을 유지하는 정확한 regression이다.

## 세션 상태 전이 요약

| 현재 상태 | 입력 | 처리 |
| --- | --- | --- |
| free | valid ACQUIRE(P) | `g_client_pid=P`, READY/OK |
| owner=P | ACQUIRE(P) | 현재 owner에게 READY/OK |
| owner=P, available | ACQUIRE(Q) | READY/BUSY, decoder와 output 불변 |
| owner=P, unavailable, line 없음 | ACQUIRE(Q) | state reset 후 Q 예약 |
| owner=P, unavailable, visible partial line | ACQUIRE(Q) | newline commit 후 reset, Q 예약 |
| recovery newline 실패 | ACQUIRE(Q) | server failure, Q에 성공 READY를 보내지 않음 |
| owner=P | bit from Q | 무시; accumulator 불변 |
| owner=P | terminating NUL from P | line 종료, session free |

## 이 Thread의 경계

이 Thread는 **누가 하나의 decoder state를 사용할 권한을 갖는가와 그 권한을 언제 회수하는가**를 다룬다.

- ACK sequence와 다음 bit 전진 조건은 `01-timing-to-correlated-sequence-acks.md`의 주제다.
- signal handler를 self-pipe로 제한하는 방식은 `03-self-pipe-event-loop.md`에서 다룬다.
- partial-line newline을 포함한 stdout failure의 commit 의미는 `04-output-commit-boundary.md`에서 다룬다.
- socket path를 생성하고 소유·삭제하는 규칙은 `05-endpoint-ownership-and-bounded-polling.md`에서 다룬다.
- nonce/sequence response의 전체 acceptance predicate는 `06-bounded-response-correlation.md`에서 다룬다.

## 조사 범위

각 설명은 표시된 exact SHA의 diff와 그 SHA의 source를 기준으로 작성했다. final HEAD의 endpoint-aware helper나 self-pipe 구조를 earlier SHA에 소급하지 않았다. repository를 checkout해 test binary를 실행하지 못했으므로, test commit은 fixture와 assertion이 요구하는 결과만 설명하며 이 세션에서 통과했다고 주장하지 않는다.
