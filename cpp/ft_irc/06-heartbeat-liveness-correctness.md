# Thread 06. heartbeat 생존 판정의 정확성

## 단순한 “PING을 보냈다”를 넘어 무엇을 증명해야 하는가

유휴 연결에 PING을 보내는 것만으로는 생존 확인이 되지 않습니다. 서버가 보낸 특정 challenge에 대한 응답인지 확인해야 하고, timeout은 시스템 시각 조정과 무관하게 실제 경과 시간으로 계산해야 합니다.

이 Thread는 초기 boolean heartbeat를 다음 상태 machine으로 교정합니다.

```text
Active
  └─ idle duration 경과
       → AwaitingPong(token, sentAt)
            ├─ 정확히 일치하는 PONG(token) → Active
            ├─ 무관한 line/잘못된 PONG      → AwaitingPong 유지
            └─ ping timeout 경과             → Closing
```

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | `RESILIENCE, LIFECYCLE, RISK` | idle/PING/PONG/timeout 상태의 최초 구현 |
| 2 | `d710f29f38a4` | `test(client): 서버 PING에 응답하는 검사 클라이언트 구현` | B | `VERIFICATION, IRC_PROTOCOL` | smoke client에 server PING 자동 응답 기능 추가 |
| 3 | `f313e707474f` | `test(client): 유휴 연결의 PING·PONG 흐름 검증` | B | `VERIFICATION, RESILIENCE` | 짧은 timeout으로 heartbeat가 실제 발생하는지 통합 확인 |
| 4 | `3f2b3ae1d3f9` | `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리` | A | `DEBUG, RESILIENCE, RISK` | elapsed-time clock과 exact challenge token 도입 |
| 5 | `0c76aad19579` | `test(heartbeat): PONG 토큰과 시간 경계 검증` | A | `VERIFICATION, RESILIENCE, RISK` | matching/forged PONG의 서로 다른 terminal behavior 고정 |

## 1. `764361c52b2a` — heartbeat 골격과 그 안의 허점

연결마다 다음 값이 추가됩니다.

```text
lastActivityAt
awaitingPong
lastPingAt
```

`onLine`은 어떤 line이 들어오면 `lastActivityAt`을 현재 시간으로 갱신합니다. maintenance는 등록 timeout을 먼저 처리한 뒤 heartbeat 설정이 활성화되어 있으면 다음을 수행합니다.

```cpp
if (client.awaitingPong &&
    now - client.lastPingAt >= pingTimeout) {
    sendRaw(fd, Replies::error("Ping timeout"));
    requestClose(fd, "ping timeout");
    return;
}
if (!client.awaitingPong &&
    now - client.lastActivityAt >= idleTimeout) {
    const std::string token =
        "heartbeat-" + std::to_string(fd) + "-" + std::to_string(now);
    sendRaw(fd, Replies::formatMessage(serverName, "PING", {token}));
    client.awaitingPong = true;
    client.lastPingAt = now;
}
```

PONG handler는 message 내용을 보지 않습니다.

```cpp
void IrcApplication::handlePong(int fd, const IrcMessage&) {
    client.awaitingPong = false;
    client.lastPingAt = 0;
}
```

따라서 다음 문제가 남습니다.

### challenge와 response가 연결되지 않는다

서버 PING을 받기 전의 PONG, 오래된 PING에 대한 PONG, 임의 문자열의 PONG도 현재 deadline을 해제합니다. 공격자나 오작동 client가 `PONG :anything`을 주기적으로 보내면 실제 server challenge를 확인하지 않고도 살아남을 수 있습니다.

### wall clock이 elapsed time에 사용된다

`std::time(NULL)`은 시스템 시각입니다. NTP 보정이나 관리자의 시간 변경으로 뒤로 움직이면 timeout이 늦어지고, 앞으로 크게 이동하면 즉시 만료될 수 있습니다. heartbeat와 rate limit이 원하는 것은 달력 시각이 아니라 두 event 사이의 경과 시간입니다.

### token 생성의 유일성이 약하다

`fd`와 초 단위 `now`를 조합하므로 같은 초·fd 재사용 같은 상황에서 이전 challenge와 같은 문자열이 재생성될 수 있습니다. 그 자체가 즉시 오류를 만들지는 않아도 challenge identity로는 불필요하게 약합니다.

이 commit은 operational heartbeat의 시작점이지 최종 생존 판정 contract가 아닙니다.

## 2. 자동 응답 client와 초기 smoke가 보여 주는 범위

### `d710f29f38a4`

Python `IrcPeer`에 `auto_pong` option이 추가됩니다. 받은 line이 server PING 형태면 token을 추출해 그대로 돌려줍니다.

```python
if " PING " in line:
    token = line.split(" PING ", 1)[1]
elif line.startswith("PING "):
    token = line.split(" ", 1)[1]
self.send_line(f"PONG :{token.lstrip(':')}")
```

기본값이 `True`이므로 기존 smoke client가 짧은 idle timeout 아래에서도 의도치 않게 끊기지 않습니다. 이 helper는 exact token을 echo하지만, production server가 token을 실제로 비교하는지는 아직 증명하지 않습니다. 잘못된 token을 일부러 보내는 negative path가 없기 때문입니다.

### `f313e707474f`

smoke server를 `--idle-timeout=1 --ping-timeout=2 --registration-timeout=5`로 시작하고 별도 idle peer가 server PING을 관찰합니다. 그 뒤 client-originated `PING :still-alive`에 대한 PONG도 확인합니다.

이 test가 확인하는 것:

- main loop의 `onTick`이 실제로 실행됨
- idle threshold 뒤 server PING이 wire에 나타남
- 일반 IRC PING handler도 동작함
- auto-pong client가 heartbeat traffic 아래 계속 연결됨

이 test가 확인하지 않는 것:

- server PING token과 PONG token이 일치해야만 하는지
- forged PONG 뒤에도 timeout이 유지되는지
- wall clock 변화와 무관한지
- matching PONG이 정확히 어떤 deadline state를 지우는지

따라서 테스트가 통과해도 초기 `handlePong`의 무조건 clear 버그는 남습니다.

## 3. `3f2b3ae1d3f9` — elapsed time과 pending token을 상태에 넣다

### 모든 maintenance time을 `steady_clock`으로 통일

`ClientState`는 `std::time_t` 대신 다음 alias를 사용합니다.

```cpp
typedef std::chrono::steady_clock MonotonicClock;
typedef MonotonicClock::time_point MonotonicTime;
```

`connectedAt`, `lastActivityAt`, `lastPingAt`, rate-limit `commandWindow`가 모두 같은 monotonic time domain으로 이동합니다. 비교는 명시적인 duration으로 바뀝니다.

```cpp
now - client.lastPingAt >=
    std::chrono::seconds(_runtime.pingTimeoutSeconds)
```

`steady_clock`은 wall clock 조정과 분리되어 있고, 같은 process 안에서 time point의 순서가 뒤로 가지 않는 elapsed-time source입니다.

### challenge identity를 저장

`ClientState`에 `pendingPongToken`을 추가하고 application은 증가하는 `_nextHeartbeatToken`을 소유합니다.

```cpp
const std::string token =
    "heartbeat-" + std::to_string(fd) + "-" +
    std::to_string(++_nextHeartbeatToken);

sendRaw(fd, Replies::formatMessage(_serverName, "PING", {token}));
client.awaitingPong = true;
client.pendingPongToken = token;
client.lastPingAt = now;
```

counter는 wall-clock text를 token identity로 사용하지 않게 합니다. token은 암호학적 nonce가 아니며 인증 목적도 아닙니다. 한 server process 안에서 pending challenge를 구별하기 위한 protocol state입니다.

### PONG의 acceptance condition

```cpp
void IrcApplication::handlePong(int fd, const IrcMessage& message) {
    ClientState& client = _clients.state(fd);
    if (!client.awaitingPong ||
        message.params.size() != 1 ||
        message.params[0] != client.pendingPongToken) {
        return;
    }
    client.awaitingPong = false;
    client.pendingPongToken.clear();
    client.lastPingAt = MonotonicTime();
}
```

세 조건을 모두 만족해야 state가 바뀝니다.

1. 실제로 PONG을 기다리는 중
2. parameter가 정확히 하나
3. 그 parameter가 현재 pending token과 동일

잘못된 PONG은 numeric 오류를 만들지 않고 무시합니다. 더 중요한 사실은 timeout state를 지우지 않는다는 것입니다.

### 이 fix에 남아 있던 별도 수명 문제

이 SHA에서는 PING을 `sendRaw`한 뒤 `ClientState&`를 갱신합니다. output queue 또는 event-interest 갱신 실패가 즉시 connection을 제거할 수 있다는 후속 조건까지 고려하면 send 이후 reference가 무효가 될 수 있습니다. Thread 08의 `728aaabc4012`가 heartbeat state를 전송 전에 publish하고 실패 반환 시 즉시 중단하도록 바꿉니다.

이는 heartbeat 의미론과 application 재진입 수명이 서로 다른 Thread인 이유입니다.

## 4. `0c76aad19579` — positive와 negative path를 분리해 검증하다

focused contract test는 자동 응답을 끕니다.

```python
heartbeat = register_contract_peer(..., "ctheartbeat")
heartbeat.auto_pong = False
```

### matching token

server PING을 regex로 받아 실제 token을 추출하고 그대로 PONG합니다.

```python
heartbeat_ping = record_regex(
    ..., rf":irc\.relay\.local PING heartbeat-\d+-\d+")
heartbeat_token = heartbeat_ping.rsplit(" ", 1)[1].lstrip(":")
heartbeat.send_line(f"PONG :{heartbeat_token}")
```

그 뒤 timeout 경계를 넘는 sleep과 `METRICS` 요청을 반복해 연결이 계속 살아 있고 이전 PING deadline이 지워졌음을 확인합니다. 이후 idle 상태에서 새 heartbeat가 생길 수는 있지만 old challenge의 timeout으로 끊기면 안 됩니다.

### forged token

별도 client는 server PING을 받은 뒤 다른 token을 보냅니다.

```python
forged.send_line("PONG :forged-heartbeat-token")
record_exact(..., "ERROR :Ping timeout")
forged.wait_closed(2.0)
```

이 사례는 production handler의 negative branch를 직접 겨냥합니다. 잘못된 PONG이 `awaitingPong`을 해제한다면 ERROR와 close가 나타나지 않아 test가 실패합니다.

## 최종 heartbeat state ledger

| 현재 상태 | 입력/시간 조건 | 갱신 | 결과 |
| --- | --- | --- | --- |
| Active | 어떤 line 수신 | `lastActivityAt=now` | idle deadline 재시작 |
| Active | idle timeout 경과 | token 생성, pending state와 sent time 기록 | PING queue |
| AwaitingPong | exact `PONG token` | pending token clear, awaiting=false | 정상 Active 복귀 |
| AwaitingPong | forged/old/malformed PONG | 상태 변화 없음 | 기존 deadline 유지 |
| AwaitingPong | 다른 일반 line | activity는 갱신되지만 pending challenge 유지 | PONG deadline 유지 |
| AwaitingPong | ping timeout 경과 | ERROR queue, close request | Closing |
| 미등록 | registration timeout 경과 | 451, close request | Closing |

## 보장 범위

보장:

- elapsed-time policy가 system wall clock 변화에 의존하지 않습니다.
- 한 connection의 pending challenge는 exact token으로 식별됩니다.
- forged PONG은 deadline을 해제하지 않습니다.
- focused test가 matching과 mismatching behavior를 분리합니다.

비보장:

- token은 보안 인증값이나 예측 불가능한 nonce가 아닙니다.
- network가 심하게 지연될 때 timeout 숫자가 운영상 최적인지는 이 Thread가 결정하지 않습니다.
- PING enqueue 실패 뒤 application object 수명은 Thread 08의 재진입 보강이 필요합니다.
- 다양한 platform에서 timing test의 안정성을 반복 확인하는 것은 Thread 09의 CI 범위입니다.

> 검사 범위: 각 SHA의 heartbeat state, client helper, contract test diff를 확인했습니다. timing test를 실제 실행해 wall-clock duration을 측정하지 않았습니다.
