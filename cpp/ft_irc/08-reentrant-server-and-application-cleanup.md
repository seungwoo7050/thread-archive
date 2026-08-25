# Thread 08. 재진입 가능한 서버와 application 정리

## 이 Thread가 다루는 실패 형태

event-driven server의 callback은 단순 notification이 아닙니다. callback 안에서 `sendTo`, `disconnect`, channel broadcast가 다시 server를 호출할 수 있고, 그 호출은 현재 stack frame이 참조 중인 `Connection`, `ClientState`, `Channel`을 제거할 수 있습니다.

따라서 다음 코드는 안전하지 않습니다.

```text
raw pointer/reference 확보
→ callback 또는 send 호출
→ 같은 pointer/reference를 다시 사용
```

이 Thread는 server와 application 두 계층에서 같은 원칙을 세웁니다.

> 외부 호출 뒤에는 객체의 생존을 가정하지 않는다. stable identifier(fd, channel name)를 값으로 보존하고 authoritative container에서 다시 조회한다.

또한 connection map과 event backend 중 하나만 등록된 partial state를 남기지 않도록 registration/update failure를 rollback 또는 disconnect로 수렴시킵니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | `LIFECYCLE, RISK` | map에서 제거한 뒤 callback을 호출하는 기본 disconnect 순서 확립 |
| 2 | `5dcd882f0763` | `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장` | S | `DEBUG, LIFECYCLE, RISK` | callback 후 fd 재조회, backend registration rollback, interest failure cleanup 구현 |
| 3 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | `VERIFICATION, LIFECYCLE, RISK` | injectable event manager로 server partial state와 callback 제거를 재현 |
| 4 | `728aaabc4012` | `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인` | S | `DEBUG, LIFECYCLE, RISK` | send 결과 전파, 값 snapshot, client/channel 재조회로 application reference 보호 |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | `VERIFICATION, LIFECYCLE, RISK` | registration 응답 enqueue 실패가 connection과 app state를 정리하는지 검증 |
| 6 | `d48e1f1f8c04` | `fix(app): 응답 실패 뒤 명령 처리를 중단` | A | `DEBUG, LIFECYCLE, RISK` | 연속 LIST/NAMES와 복합 MODE가 실패 뒤 계속 진행하지 않게 보강 |
| 7 | `aee5edebe294` | `test(app): 연결 정리 뒤 모드 변경 중단 검증` | A | `VERIFICATION, LIFECYCLE, CHANNEL_STATE` | 첫 mode 방송에서 sender 제거 후 다음 mode가 적용되지 않음을 검증 |

## 1. `7a6bc7e1276a` — 올바른 disconnect 순서가 새로운 수명 문제를 드러내다

기본 disconnect는 fd를 event backend에서 제거하고, connection `unique_ptr`을 local로 옮긴 뒤 map entry를 지우고 callback을 호출합니다.

```text
removeFd(fd)
→ unique_ptr를 local로 move
→ connections_.erase(fd)
→ onDisconnect(*localConnection, reason)
→ localConnection 파괴, fd close
```

이 순서의 장점은 callback이 `findConnection(fd)`를 호출했을 때 이미 종료된 connection을 보지 않는다는 점입니다. application cleanup도 authoritative map 상태와 일치합니다.

하지만 같은 성질 때문에 callback 호출자의 raw pointer는 callback 안에서 무효가 될 수 있습니다. 예를 들어 line callback이 `server.disconnect(connection.fd(), ...)`를 호출하면 map의 object가 제거됩니다. callback이 return하거나 exception을 던진 뒤 caller가 기존 `Connection*`로 `requestClose`나 `wantsWrite`를 호출하면 use-after-free입니다.

즉 “map에서 먼저 제거”는 필요한 lifecycle invariant지만, **callback 이후 pointer를 다시 쓰지 않아야 한다**는 두 번째 invariant가 함께 필요합니다.

## 2. `5dcd882f0763` — server의 authoritative lookup과 rollback

### callback 이후 raw pointer 대신 fd로 재조회

수정된 event handler는 callback 전에 fd 값을 복사합니다.

```cpp
const int fd = found->second->fd();
Connection* connection = found->second.get();
```

line callback이 끝나거나 exception을 던진 뒤에는 기존 pointer를 사용하지 않습니다.

```cpp
try {
    onLine_(*connection, line);
} catch (const std::exception& exception) {
    reportError(exception.what());
    Connection* current = findConnection(fd);
    if (current != NULL)
        current->requestClose("line handler error");
}

connection = findConnection(fd);
if (connection == NULL)
    return;
```

write 처리 뒤와 최종 interest refresh 전에도 같은 재조회가 반복됩니다. callback이 연결을 유지했다면 새 pointer를 얻고, 제거했다면 현재 event 처리를 즉시 끝냅니다.

connect callback도 map iterator의 object를 호출 인자로 넘긴 뒤, exception 처리에서는 `findConnection(fd)`로 다시 찾습니다. callback이 자기 연결을 삭제한 경우에는 requestClose를 시도하지 않습니다.

### accept registration을 transaction으로 만들다

기존 흐름은 event backend 등록, raw pointer 확보, map 삽입 순이었습니다. map 삽입이 실패하거나 callback이 object를 제거하면 backend와 map의 상태가 어긋날 수 있었습니다.

수정 후 순서는 다음과 같습니다.

```cpp
const auto inserted = connections_.emplace(fd, std::move(connection));
if (!inserted.second)
    throw std::logic_error("accepted descriptor is already registered");
try {
    eventManager_->addFd(fd, EventInterest::Read);
} catch (...) {
    connections_.erase(inserted.first);
    throw;
}
```

- map 삽입 실패: local `unique_ptr`가 fd를 정리하고 backend에는 아직 등록되지 않음
- backend add 실패: map entry erase가 `Connection`을 파괴하고 fd를 닫음
- 둘 다 성공: accepted metric과 connect callback으로 이동

backend가 성공한 뒤에만 자체 shadow interest를 갱신한다는 Thread 01의 규칙과 결합되어 partial registration이 남지 않습니다.

listener 등록 실패도 `start()`에서 listen fd를 닫고 event manager를 reset한 뒤 exception을 다시 던집니다. `running_`은 성공 완료 뒤에만 true가 됩니다.

### write-interest update 실패를 무시하지 않다

`refreshInterest`는 `Connection&` 대신 fd를 받고 bool을 반환합니다.

```cpp
bool Server::refreshInterest(int fd) {
    Connection* connection = findConnection(fd);
    if (connection == NULL)
        return false;

    if (connection->closeRequested() && !connection->wantsWrite()) {
        disconnect(fd, connection->closeReason());
        return false;
    }

    try {
        eventManager_->updateFd(fd, interests);
    } catch (const std::exception& exception) {
        reportError(exception.what());
        disconnect(fd, "event interest update failed");
        return false;
    }
    return true;
}
```

output이 queue됐지만 backend에 Write interest를 등록하지 못하면 그 byte는 영원히 flush되지 않을 수 있습니다. 따라서 “queue는 성공했지만 refresh는 실패”를 성공으로 보고하지 않고 connection 전체를 제거합니다.

`sendTo`와 `queueRawTo`는 `queued && refreshed`를 반환합니다. 이 bool이 application 재진입 방어의 시작점입니다.

### error callback도 cleanup을 깨뜨릴 수 없다

`reportError`는 `noexcept`이며 user-provided error handler의 모든 exception을 삼킵니다. cleanup을 보고하는 callback이 다시 exception을 던져 원래 rollback을 중단시키지 않게 합니다.

## 3. `928594ec160c` — server lifecycle을 fake backend로 직접 흔들다

`FakeEventManager`는 관심 fd map과 queued event를 보유하고 다음 실패를 one-shot으로 주입합니다.

- 다음 `addFd` 실패
- 다음 `updateFd` 실패

실제 loopback socket은 사용하지만 readiness delivery는 fake backend가 통제합니다.

### registration rollback

server start 뒤 다음 client `addFd`만 실패시킵니다. accept는 일어났지만 최종 상태는 다음이어야 합니다.

```text
server.connectionCount() == 0
event backend에 client fd 없음
```

map만 남거나 backend만 남는 partial registration을 잡습니다.

### connect callback이 제거하고 throw

connect callback은 전달받은 fd를 즉시 disconnect한 뒤 exception을 던집니다. error handler마저 exception을 던지게 구성합니다. server가 stale pointer를 사용하거나 cleanup exception을 전파하면 test process가 실패합니다. 최종 map size 0을 요구합니다.

### line callback이 제거하고 throw

client가 line을 보내고 fake readable event를 queue합니다. line callback도 connection을 제거하고 throw합니다. event handler가 callback 이후 기존 pointer를 사용하지 않아야 합니다.

### interest update failure

accepted client에 output을 queue하기 직전 다음 `updateFd`를 실패시킵니다.

```text
server.sendTo(...) == false
connection map에 fd 없음
backend interest map에도 fd 없음
```

### queue-limit close

매우 작은 pending limit에서 oversized response를 보내면 enqueue가 false와 close request를 만들고, `refreshInterest(fd)`가 pending output 없는 close를 즉시 disconnect로 끝내는지 확인합니다.

이 test는 fake backend의 deterministic state를 검증합니다. 실제 epoll/kqueue kernel failure timing 전체를 재현하는 것은 아닙니다.

## 4. server fix만으로 application reference가 안전하지 않은 이유

application helper는 이전에 다음 형태였습니다.

```cpp
void sendNumeric(...);
void sendRaw(...) {
    _server.sendTo(fd, line);  // bool 결과 무시
}
```

`Server::sendTo`는 queue limit 또는 interest update failure에서 false를 반환하고, 그 과정에서 `disconnect`를 실행할 수 있습니다. disconnect callback은 `_clients.erase(fd)`와 channel membership cleanup을 수행합니다.

따라서 아래 패턴은 위험합니다.

```cpp
ClientState& client = _clients.state(fd);
sendNumeric(fd, ...);       // 여기서 fd가 제거될 수 있음
client.registered = true;   // dangling reference
```

```cpp
Channel& channel = ensureChannel(name);
broadcastToChannel(...);    // member 전송 실패로 channel이 erase될 수 있음
channel.removeMember(fd);   // dangling reference
```

server가 callback 이후 자기 pointer를 재조회해도, application이 보유한 별도 container reference까지 자동으로 안전해지지는 않습니다.

## 5. `728aaabc4012` — 출력은 수명 경계이며 결과를 반드시 소비한다

이 S급 fix는 application send helper 전체를 bool로 바꿉니다.

```cpp
bool IrcApplication::sendRaw(int fd, const std::string& line) {
    return _server.sendTo(fd, line);
}

bool IrcApplication::sendNumeric(...) {
    return sendRaw(fd, Replies::numeric(...));
}
```

그 위에서 세 가지 규칙을 적용합니다.

### 규칙 1: send 전에 필요한 값을 복사한다

`maybeRegister`는 `ClientState*`에서 nickname을 값으로 복사하고 등록 flag를 먼저 publish합니다. 각 welcome numeric이 실패하면 즉시 return하며, 모든 응답이 성공한 뒤에만 registered log를 남깁니다.

```cpp
client->registered = true;
const std::string nick = client->nick;
if (!sendNumeric(fd, 1, {}, "Welcome ... " + nick))
    return;
if (!sendNumeric(fd, 2, {}, "Your host is ..."))
    return;
if (!sendNumeric(fd, 3, {}, "This server is running ..."))
    return;
logEvent("client_registered", ...);
```

첫 send가 connection을 제거해도 이후에는 `client` pointer를 읽지 않습니다. `nick`은 독립된 값입니다.

INVITE도 source prefix, channel name, acknowledgement, invitation frame을 전송 전에 모두 만듭니다. sender acknowledgement 중 sender state가 사라져도 target frame 생성에 stale state가 필요 없습니다.

### 규칙 2: fan-out 뒤 container를 다시 찾는다

`partChannel`은 broadcast 뒤 기존 iterator를 버리고 channel name으로 다시 조회합니다.

```cpp
broadcastToChannel(channelName, partLine, -1);
it = _channels.find(channelName);
if (it == _channels.end())
    return;
it->second.removeMember(fd);
```

KICK도 channel name과 target 정보로 frame을 만든 뒤 broadcast하고, map에서 channel을 다시 찾아 target을 제거합니다.

JOIN은 member 추가와 JOIN broadcast 뒤 `_clients.contains(fd)`를 확인하고, channel도 `_channels.find(name)`으로 다시 얻은 뒤 topic과 NAMES를 보냅니다. topic reply 뒤에도 다시 lookup하고 NAMES를 시작합니다.

### 규칙 3: 연속 응답은 첫 실패에서 중단한다

`sendNames`는 353이 성공해야 366을 보냅니다. `sendTopicReply`, `sendNumericRaw`도 성공을 상위 caller에 전달합니다. handler가 이미 제거된 fd에 후속 response를 계속 queue하지 않습니다.

### heartbeat state는 전송 전에 publish한다

PING을 보내기 전에 `awaitingPong`, pending token, lastPing time을 기록합니다.

```cpp
client->awaitingPong = true;
client->pendingPongToken = token;
client->lastPingAt = now;
if (!sendRaw(fd, pingLine))
    return;
++_metrics.heartbeatPings;
```

send 실패가 disconnect를 일으켜도 그 뒤 `client`를 쓰지 않습니다. 성공한 enqueue에만 heartbeat metric을 증가시킵니다.

### 이 commit의 범위와 남은 구멍

큰 lifecycle 경로는 수정됐지만 모든 multi-response/compound loop가 첫 실패를 확인하도록 완전히 바뀐 것은 아닙니다. LIST/NAMES 전체 반복과 channel MODE의 연속 문자 처리는 다음 commit에서 추가로 닫힙니다.

## 6. `5edcafda8a4d` — 등록 중 queue failure를 실제 application state까지 관찰하다

application lifetime test는 fake event manager와 loopback client를 사용하고 `maxPendingBytes=1`로 설정합니다. password가 비어 있어 NICK과 USER만 보내면 registration gate에 도달하지만 welcome numeric은 1 byte cap을 넘습니다.

기대 terminal state:

- queue rejection이 server connection을 제거
- disconnect callback이 client registry와 channel state를 정리
- server process 자체는 계속 running
- 모든 welcome numeric이 성공하지 않았으므로 `event=client_registered` log가 없음

이 test는 단순히 `sendTo == false`를 보는 것이 아니라 server→disconnect callback→application cleanup의 재진입 chain을 통과합니다.

## 7. `d48e1f1f8c04` — compound command의 나머지 동작을 멈추다

### LIST와 NAMES

LIST 시작 321이 실패하면 channel loop를 시작하지 않습니다. 각 322가 실패하면 뒤 channel과 323을 보내지 않습니다. NAMES도 각 channel의 353/366 sequence가 실패하면 전체 loop를 종료합니다.

이는 제거된 fd에 계속 응답을 만드는 낭비뿐 아니라, send가 다른 application state mutation을 유발한 뒤 map iterator를 계속 사용하는 위험을 줄입니다.

### MODE

handler는 먼저 `channelName`을 값으로 복사합니다. `+i`, `+t`, `+o` 각각 state를 변경하고 `broadcastMode`를 호출한 뒤 false면 즉시 return합니다.

```cpp
channel->setInviteOnly(adding);
if (!broadcastMode(fd, *channel, adding ? "+i" : "-i", ""))
    return;
```

operator 대상도 `_clients.state(targetFd)`처럼 없는 state를 생성할 수 있는 API 대신 `_clients.find(targetFd)`를 사용하고, nickname을 값으로 복사한 뒤 mode state를 바꿉니다.

중요한 의미는 compound mode의 atomicity가 아닙니다. `MODE #room +it`에서 `+i` state mutation과 broadcast가 완료된 뒤 sender가 제거되면 `+i`는 유지되고 `+t`만 실행하지 않습니다. 이미 관찰된 첫 변경을 rollback하지 않고, **failure 이후 추가 side effect를 중단**합니다.

## 8. `aee5edebe294` — 첫 mode만 남고 이후 mode는 멈추는지 검증하다

테스트는 application private state를 직접 구성하기 위해 test translation unit에서 private 접근을 열고, 두 client와 `#room` channel을 만듭니다. sender는 operator, peer는 일반 member이며 topic-protected는 false입니다.

fake backend는 sender fd의 다음 interest update를 실패시킵니다. `MODE #room +it`를 호출하면 다음 일이 일어나야 합니다.

1. `+i`가 channel state에 적용됨
2. 첫 MODE broadcast가 sender에게 enqueue되면서 interest update 실패
3. server가 sender connection을 disconnect
4. application disconnect callback이 sender membership/state를 제거
5. `broadcastMode`가 sender 부재를 보고 false 반환
6. handler return, 뒤의 `+t`는 적용되지 않음

assertion은 다음을 분리합니다.

```text
channel은 peer 때문에 여전히 존재
inviteOnly == true        (첫 mode는 적용됨)
topicProtected == false   (뒤 mode는 중단됨)
sender state 없음
peer state는 유지
```

이 test가 보장하는 것은 targeted partial-command semantics입니다. 모든 command가 transaction처럼 전부 성공하거나 전부 rollback된다는 뜻은 아닙니다.

## 최종 수명 규칙

| 외부 호출 지점 | 호출 전 보존할 값 | 호출 후 허용되는 참조 |
| --- | --- | --- |
| server connect/line callback | fd | `findConnection(fd)` 결과만 |
| event backend add/update | fd, map iterator 또는 config | 실패 시 map/backend rollback 후 기존 pointer 사용 금지 |
| application `sendRaw/sendNumeric` | fd, 필요한 nick/prefix/text 복사 | `_clients.find(fd)`, `_channels.find(name)` 결과만 |
| channel fan-out | channel name, member snapshot, frame text | fan-out 뒤 channel map 재조회 |
| multi-response sequence | 다음 frame을 만들 독립 값 | bool false면 즉시 중단 |
| compound MODE | channel name, 현재 mode argument | broadcast 실패면 뒤 mode 처리 중단 |

## Thread 경계

- `Connection` 내부의 offset/cap 정확성은 Thread 07입니다.
- channel의 정상 membership/fan-out/cleanup 의미는 Thread 03입니다. 이 Thread는 fan-out이 재진입할 때의 수명만 다룹니다.
- fake backend test를 실제 epoll/kqueue와 sanitizer에서 반복하는 것은 Thread 09입니다.
- multi-threaded concurrent mutation은 이 server가 채택한 단일 event-loop 모델 밖이며, 이 Thread가 thread safety를 제공한다는 뜻은 아닙니다.

> 검사 범위: exact server/application fix diff와 세 focused test source를 확인했습니다. ASan 실행이나 실제 backend failure 주입 결과는 이 환경에서 재현하지 않았습니다.
