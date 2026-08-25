# Thread 04. 운영 보호 장치와 제어된 종료

## Thread의 범위

기능이 올바른 IRC server도 한 연결이 등록을 끝내지 않거나, 명령을 폭주시키거나, 출력을 읽지 않으면 전체 process의 자원을 계속 점유할 수 있습니다. 반대로 운영 한도만 추가하고 어떤 경로에서 차단됐는지 관찰하지 못하면 실제 장애 원인을 찾기 어렵습니다. 종료 신호를 받자마자 socket을 닫아 버리면 이미 queue된 ERROR와 종료 사유가 client에 전달되지 않습니다.

이 Thread는 연결별·서버별 resource budget, 그 budget이 발동한 지점의 metric/log, signal 이후의 bounded drain을 하나의 운영 lifecycle로 묶습니다.

```text
정상 처리
  ├─ 등록 대기 시간
  ├─ heartbeat 대기 시간
  ├─ 명령 호출 window
  ├─ 연결별 pending output
  └─ 서버 전체 connection 수
        ↓
발동 지점에서 close request + metric/log
        ↓
SIGTERM/SIGINT 이후 ERROR enqueue + 제한된 drain + 최종 stop
```

heartbeat의 token·clock 정확성과 output queue 산술은 각각 Thread 06과 07에서 더 깊게 다룹니다. 여기서는 운영 보호 장치로서 언제 close를 요청하고 어떤 상태를 관찰하는지에 집중합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `c4df44554866` | `feat(registration): 등록 대기 시간 제한` | B | `RESILIENCE, LIFECYCLE` | 미등록 연결의 점유 시간을 제한 |
| 2 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | `RESILIENCE, LIFECYCLE, RISK` | idle connection을 probe하고 응답 deadline 부여 |
| 3 | `9e2b214f9227` | `feat(throttle): 클라이언트별 명령 호출 횟수 제한` | B | `RESILIENCE` | connection별 sliding command window 도입 |
| 4 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | `EVENT_IO, RESILIENCE, RISK` | 느린 수신자의 pending output을 hard cap으로 제한 |
| 5 | `adb49d9466e4` | `feat(server): 최대 연결 수 제한` | B | `RESILIENCE, EVENT_IO` | process 전체 connection cardinality 제한 |
| 6 | `e05e35ca7da9` | `feat(metrics): 서버 실행 지표 조회 기능 추가` | B | `OBSERVABILITY` | resource/lifecycle 사건을 누적 counter로 노출 |
| 7 | `c34aa18f89af` | `feat(log): 연결 상태와 실행 지표 기록` | B | `OBSERVABILITY, LIFECYCLE` | 주요 상태 전이를 구조화된 stderr event로 기록 |
| 8 | `dd04279c47fd` | `feat(shutdown): 종료 전 송신 대기열 처리` | A | `LIFECYCLE, RESILIENCE, RISK` | signal handler와 정상 종료·drain 경로 분리 |
| 9 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | `VERIFICATION, IRC_PROTOCOL, RISK` | CLI, wire, 보호 장치, 종료 동작을 exact contract로 고정 |

## 1. 연결이 등록 전 상태로 무한정 머물지 않게 하다

### `c4df44554866` — registration deadline

연결 수립 시점에 `connectedAt`을 기록하고 main loop가 `server.pollOnce()` 뒤 `app.onTick()`을 호출합니다. `onTick`은 registry의 fd snapshot을 순회해 각 client를 유지보수합니다.

```cpp
if (!client.registered &&
    now - client.connectedAt >= _runtime.registrationTimeoutSeconds) {
    sendNumeric(fd, 451, {}, "Registration timeout");
    requestClose(fd, "registration timeout");
}
```

snapshot을 사용하는 이유는 `maintainClient`가 timeout을 발견해 close를 요청하고, server의 queue/interest 갱신 과정에서 registry mutation이 일어날 수 있기 때문입니다.

이 commit은 연결별 deadline을 도입하지만 `std::time(NULL)` wall clock을 사용합니다. 시스템 시간이 뒤로 이동하면 경과 시간이 줄어드는 문제가 남으며, Thread 06의 `3f2b3ae1d3f9`가 registration timeout을 포함한 시간 기반 상태를 `steady_clock`으로 옮깁니다.

## 2. 유휴 상태를 probe하고 command burst를 차단하다

### `764361c52b2a` — 초기 heartbeat 운영 정책

각 client에 `lastActivityAt`, `awaitingPong`, `lastPingAt`을 추가합니다. line을 받을 때 activity time을 갱신하고, maintenance는 다음 순서로 판단합니다.

1. registration timeout
2. heartbeat 비활성 설정 여부
3. PONG 대기 중 deadline 초과 → ERROR, close request
4. PONG 대기 중이 아니고 idle threshold 초과 → server PING enqueue, 대기 상태 설정

이 순서는 이미 PONG deadline을 지난 연결에 새 PING을 덮어쓰지 않게 합니다. 다만 최초 구현은 어떤 PONG이든 대기 상태를 해제하고 wall clock을 사용합니다. 운영 보호 장치의 골격은 여기서 생겼지만, 생존 판정의 정확성은 Thread 06의 fix에서 완성됩니다.

### `9e2b214f9227` — connection-local sliding window

parser가 성공한 뒤, command handler로 들어가기 전에 `recordCommand`를 호출합니다. malformed line은 rate limit counter에 포함되지 않고 417 처리로 끝납니다.

```cpp
while (!window.empty() && now - window.front() >= seconds)
    window.pop_front();
window.push_back(now);
if (limit != 0 && window.size() > limit) {
    sendNumeric(fd, 439, {}, "Command rate limit exceeded");
    requestClose(fd, "command rate limit exceeded");
    return false;
}
```

window는 `ClientState` 안에 있으므로 한 client의 burst가 다른 client의 quota를 소비하지 않습니다. 초과한 현재 command는 `false` 반환으로 dispatch 전에 중단됩니다. `rateLimitCount == 0`은 제한 비활성화 의미입니다.

이 구현은 초 단위 timestamp를 쓰는 초기 버전이고, monotonic time 전환은 heartbeat fix에서 함께 수행됩니다. limit의 의미와 prune 조건은 그대로 유지됩니다.

## 3. 느린 수신자가 memory를 무한히 점유하지 못하게 하다

### `d7d85e518177` — pending output hard cap

`Server::Config`와 각 `Connection`에 `maxPendingBytes`를 추가하고, `queueRaw`/`queueLine`이 성공 여부를 반환하도록 바꿉니다. line queue는 기존 terminator를 제거한 payload와 새 `\r\n`까지 합친 byte 수를 cap에 포함합니다.

```cpp
bool Connection::queueRaw(const std::string& bytes) {
    if (!canAppendPending(bytes.size())) {
        requestClose("outbound queue limit exceeded");
        return false;
    }
    writeBuffer_.append(bytes);
    return true;
}
```

Server는 enqueue 성공 여부와 상관없이 interest를 refresh합니다. 실패 시 connection은 close-requested이고 기존 pending bytes가 있다면 그것을 drain하기 위해 Write interest가 유지됩니다. 반환값은 application까지 올라가 이후 command 처리를 중단할 수 있는 근거가 됩니다. 실제로 이 실패가 즉시 connection cleanup을 일으키는 재진입 문제는 Thread 08에서 다룹니다.

최초 `canAppendPending`은 다음 덧셈을 사용했습니다.

```cpp
pendingBytes() + byteCount <= maxPendingBytes_
```

`size_t` 덧셈 overflow 시 작은 값으로 wrap해 cap을 우회할 수 있습니다. 따라서 이 commit은 operational cap을 도입하지만 산술적으로 완전하지 않습니다. Thread 07의 `881e59734a9a`가 subtraction 기반 검사로 수정합니다.

## 4. server 전체 connection 수를 제한하다

### `adb49d9466e4`

`accept`가 성공한 직후 현재 `connections_.size()`와 `maxConnections`를 비교합니다. 이미 한도 이상이면 새 client fd를 즉시 닫고 error handler에 rejection을 보고한 뒤, accept loop를 계속합니다.

```cpp
if (config_.maxConnections != 0 &&
    connections_.size() >= config_.maxConnections) {
    rejectReadyClient();
    ::close(clientFd);
    continue;
}
```

한도 검사를 accept 전에 할 수는 없습니다. listen socket readiness는 대기 중 연결이 있다는 뜻일 뿐 실제 fd가 아직 없고, backlog를 비우려면 accept 후 거절해야 합니다. `0`은 이 옵션에 한해 unlimited입니다.

거절된 fd는 event backend나 connection map에 들어가지 않으므로 connect callback도 호출되지 않습니다.

## 5. 발동한 보호 장치를 관찰 가능하게 만들다

### `e05e35ca7da9` — counter를 실제 state transition에 붙이다

Server metric은 transport/lifecycle 지점에서 증가합니다.

- `acceptedConnections`: connection map과 backend 등록 성공 뒤
- `closedConnections`: map에서 연결을 제거할 때
- `linesReceived`: 완성 line을 application callback에 넘길 때
- `outboundQueueDrops`: queue append가 실제 거절됐을 때

Application metric은 protocol/application 지점에 있습니다.

- `commandsHandled`: parse와 rate-limit gate를 통과한 command
- `messagesRelayed`: direct/channel PRIVMSG relay
- `rateLimitedClients`: rate-limit close가 발동한 횟수

`METRICS`는 현재 connection/room 수와 누적 counter를 정해진 key 순서의 NOTICE로 반환합니다. counter를 최종 결과가 아니라 **결정이 확정된 지점**에서 올리는 것이 중요합니다. 예를 들어 accept syscall만 성공하고 backend 등록이 실패했다면 accepted connection으로 세지 않습니다.

### `c34aa18f89af` — 구조화된 stderr event

`logEvent`는 다음 형태로 기록합니다.

```text
event=client_connected fd=7 peer=127.0.0.1:54321
event=client_rate_limited fd=7 nick=alice
event=server_metrics accepted=... closed=... lines=...
```

field 값의 whitespace는 `_`로 치환해 한 event가 한 줄의 key=value record로 유지됩니다. room 생성, connect/disconnect, registration, ping timeout, rate limit, server start/error, 종료 시 metric snapshot이 같은 형식을 사용합니다.

metric과 log의 역할은 다릅니다.

- metric: 세션 동안 몇 번 일어났는가
- event log: 어느 fd/nickname/reason에서 일어났는가

이 commit은 logging 실패를 별도 recovery 대상으로 다루지 않습니다. stderr stream 자체의 I/O failure까지 보장하는 Thread는 아닙니다.

## 6. `dd04279c47fd` — signal handler 밖에서 종료 절차를 수행하다

기존 signal handler는 global `Server*`를 통해 `server.stop()`까지 호출했습니다. 하지만 `stop()`은 container, callback, fd를 만지는 일반 C++ 코드이므로 async-signal-safe하지 않습니다.

수정 후 handler는 `volatile sig_atomic_t gRunning`만 0으로 만듭니다.

```cpp
void handleSignal(int) {
    gRunning = 0;
}
```

main loop가 정상 control flow로 빠져나온 뒤 application shutdown을 실행합니다.

```text
app.shutdown("Server shutting down")
  → registry fd snapshot
  → 각 client에 ERROR enqueue
  → 각 connection에 close request
  → metric log

최대 8회:
  server.pollOnce(50ms)
  → pending output flush와 disconnect 진행

callback 해제
server.stop()
```

최대 drain budget은 400ms입니다. 끝까지 읽지 않는 client 때문에 종료가 무한정 지연되는 것을 막으면서, 즉시 close보다 ERROR가 전달될 기회를 줍니다. budget이 끝나도 남은 connection은 `server.stop()`의 강제 정리 경로로 수렴합니다.

이 선택은 모든 client가 종료 ERROR를 반드시 받는다는 보장이 아닙니다. “전달을 시도하되 process 종료는 bounded”라는 운영 계약입니다.

## 7. `e5e6c57db80d` — public behavior를 exact contract로 기록하다

기존 smoke가 주요 기능의 존재를 substring으로 확인했다면, `tests/irc_contract.py`는 public CLI와 wire frame을 더 엄격하게 비교합니다.

CLI 쪽은 다음을 확인합니다.

- 인자 부족 시 exact usage와 exit 1
- port, timeout, rate-limit shape, unknown option 오류
- stdout이 비어 있고 stderr가 정해진 문구인지
- 플랫폼에 따라 errno suffix가 달라질 수 있는 경우 허용 집합을 명시

wire 쪽은 exact frame 또는 제한된 regex를 사용합니다.

```python
record_exact(..., ":irc.relay.local 451 * :You have not registered")
record_exact(..., ":irc.relay.local 433 * CTTAKEN :Nickname is already in use")
record_regex(..., metrics_pattern("ctalpha"))
```

등록, split frame, JOIN/TOPIC/NAMES/LIST, direct/channel message, INVITE/MODE/KICK, rate limit, heartbeat, shutdown을 한 process에서 통과시키고 확인한 기대값을 manifest에 기록합니다.

이 test는 운영 계약의 breadth를 제공합니다. queue 내부 offset, event registration rollback 같은 rare internal state는 별도의 focused unit test가 필요하며 Thread 07~09에서 보강됩니다.

## 보호 장치별 owner와 terminal action

| 보호 대상 | 상태 owner | 발동 조건 | terminal action |
| --- | --- | --- | --- |
| registration 시간 | `ClientState` + application maintenance | 미등록 상태로 deadline 경과 | 451, close request |
| heartbeat | `ClientState` | PONG 대기 deadline 경과 | ERROR, close request |
| command burst | client별 deque | 현재 window에서 count 초과 | 439, 현재 command 중단, close request |
| pending output | `Connection` | append 후 cap 초과 예상 | byte 미추가, close request, false 반환 |
| 전체 connection 수 | `Server::connections_` | accept 후 map size가 cap 도달 | 새 fd만 즉시 close |
| process shutdown | main/application/server | signal flag 또는 loop 종료 | ERROR enqueue, bounded drain, 강제 stop |

## Thread의 경계

- config 문자열이 overflow 없이 대상 타입으로 변환되는지는 Thread 05입니다.
- heartbeat의 monotonic clock과 exact PONG token은 Thread 06입니다.
- pending queue cap의 overflow-safe 계산과 partial-send state는 Thread 07입니다.
- enqueue/interest 갱신 실패가 callback 중 connection을 제거할 때의 재진입 안전성은 Thread 08입니다.
- 공정성, sanitizer, Linux/macOS 반복 실행은 Thread 09입니다.

> 검사 범위: exact SHA diff와 test source를 검토했습니다. 해당 executable을 실제로 실행하거나 metric/log 출력 결과를 수집하지는 않았습니다.
