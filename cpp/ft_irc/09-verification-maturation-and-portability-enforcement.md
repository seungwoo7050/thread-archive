# Thread 09. 검증 체계의 성숙과 이식성 강제

## 기능 테스트가 하나의 `smoke`로 충분하지 않은 이유

이 server의 위험은 서로 다른 관찰 방법을 요구합니다.

- 실제 TCP framing과 여러 command의 연결: process-level integration test
- exact numeric·parameter·CLI 문구: public contract test
- `EINTR`, partial send, overflow: deterministic unit seam
- callback 중 object 제거와 backend 실패: fake event manager lifecycle test
- 느린 recipient가 전체 loop를 막는지: 다중 socket 공정성 test
- epoll/kqueue 및 undefined behavior 차이: 플랫폼 CI와 sanitizer

이 Thread는 production feature를 새로 만들기보다, 기존 불변 조건이 어느 검증 계층에서 관찰되는지 발전시킨 기록입니다. 같은 commit이 다른 Thread에도 등장하지만 여기서는 구현 설명을 반복하지 않고 **무엇을 증명하고 무엇을 증명하지 않는가**에 집중합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | 검증 체계에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | `VERIFICATION, INTEGRATION, RISK` | server process와 여러 client를 연결하는 broad smoke 도입 |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | `VERIFICATION, IRC_PROTOCOL, RISK` | CLI·wire·shutdown의 exact public contract와 manifest 추가 |
| 3 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | `VERIFICATION, EVENT_IO, RISK` | injected send 결과로 queue state transition을 결정적으로 검증 |
| 4 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | `VERIFICATION, LIFECYCLE, RISK` | fake backend와 loopback socket으로 server rollback·재진입 검증 |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | `VERIFICATION, LIFECYCLE, RISK` | transport failure가 application cleanup까지 전파되는지 검증 |
| 6 | `de1dd0fc30d0` | `test(event): 160개 연결과 느린 수신자 처리 공정성 검증` | A | `VERIFICATION, EVENT_IO, RISK` | high-fd readiness와 slow-receiver isolation을 network pressure로 검증 |
| 7 | `416efc91e580` | `ci: Linux·macOS 회귀와 새니타이저 자동화` | A | `BUILD, VERIFICATION, RISK` | 같은 suite를 epoll/kqueue 플랫폼과 ASan/UBSan에서 강제 |

## 1. `6b4a7738a285` — 첫 broad smoke: 시스템이 서로 연결되는가

shell harness는 server를 background process로 시작하고 Python client가 실제 TCP socket을 엽니다. 여러 peer가 PASS/NICK/USER로 등록하고 channel 기능을 순서대로 실행합니다.

주요 coverage:

```text
wrong PASS
case-insensitive nickname collision
split TCP frame → 한 PING line 복원
registration numerics
JOIN / topic / NAMES
channel·direct PRIVMSG
INVITE / +i / +o / KICK / PART
QUIT
```

분할 입력은 `PI`와 `NG ...\r\n`을 별도 send로 보내므로, `Connection`의 read buffer와 IRC parser가 실제 socket 조각을 넘어 연결되는지 확인합니다. 여러 client를 사용하므로 channel fan-out도 process 내부 helper 호출이 아니라 wire에서 관찰합니다.

### 이 계층의 가치

- build된 executable의 main/config/server/application 전체가 함께 동작해야 통과합니다.
- callback 연결, socket 옵션, parser, registry, channel state 중 하나라도 빠지면 주요 flow가 깨집니다.
- test가 끝날 때 실제 process와 socket cleanup을 수행합니다.

### 초기 한계

client의 `expect`는 주로 line에 특정 substring이 나타나는지 찾습니다. 따라서 다음 회귀를 놓칠 수 있습니다.

- numeric target이나 parameter 순서가 틀렸지만 substring은 존재
- 기대 frame 앞뒤에 불필요한 추가 frame이 붙음
- CLI stderr가 바뀜
- shutdown exit status 또는 exact ERROR frame이 달라짐
- rare partial syscall/lifetime branch

smoke는 “주요 사용자 여정이 살아 있다”는 증거이지 public wire contract 전체를 고정하는 도구는 아닙니다.

## 2. `e5e6c57db80d` — exact contract와 관찰 manifest

`tests/irc_contract.py`는 검증 함수를 exact/regex 두 종류로 구분합니다.

```python
peer.expect_next_exact(expected)
peer.expect_regex(pattern)
```

port처럼 실행마다 달라지는 일부 field만 정규화하고, numeric code, target, parameter order, trailing text는 전체 frame으로 비교합니다.

### CLI contract

server를 다양한 argv로 직접 실행해 다음을 함께 요구합니다.

```text
exit == 1
stdout == ""
stderr == exact expected text
```

플랫폼 C library에 따라 top-level errno suffix가 붙을 수 있는 한 사례는 허용 가능한 두 문자열을 명시합니다. 단순히 stderr substring을 찾는 대신 **같은 플랫폼에서 허용되는 public 결과 집합**을 기록합니다.

### wire contract

pre-registration 451, invalid nick 432, wrong password 464, collision 433, welcome 001~003, JOIN/NAMES/LIST, topic, message, invite, mode, kick, rate limit 등 대표 frame을 exact 비교합니다. metric처럼 counter 값이 변하는 response는 key 순서와 숫자 shape를 regex로 제한합니다.

### shutdown contract

별도 peer를 유지한 상태에서 signal을 보내고, 종료 ERROR와 connection close, process exit를 관찰합니다. 정상 command 결과뿐 아니라 lifecycle 끝도 public behavior로 포함합니다.

### manifest

검증한 CLI argv/result와 wire expectation을 JSON-compatible manifest에 쌓습니다. 이는 test 내부 assertion의 부수 산출물이며, 실행마다 어떤 계약을 확인했는지 사람이 추적할 수 있게 합니다.

이 test도 내부 offset이나 dangling pointer를 직접 볼 수 없습니다. exact output이 맞더라도 메모리 오류가 우연히 드러나지 않을 수 있으므로 focused seam과 sanitizer가 필요합니다.

## 3. `f34ab135c546` — network timing을 제거한 Connection state test

`Connection`에 주입 가능한 `SendOperation`을 사용하여 실제 kernel buffer 상태 없이 syscall 결과 sequence를 정합니다.

검증되는 branch:

- `SIZE_MAX` 경계의 overflow-safe append 판단
- line payload와 CRLF가 exact cap을 채우는 경우
- cap 초과 append의 atomic rejection
- EINTR 재시도
- partial send 뒤 EAGAIN/EWOULDBLOCK
- 전송한 prefix 용량을 새 append가 재사용
- 0-byte send에서 queue 보존
- EPIPE에서 error/close와 pending byte 보존
- 요청보다 큰 비정상 양수 반환에서 offset 보호

이 test의 강점은 같은 입력이 항상 같은 branch 순서를 만든다는 것입니다. 실제 socket test에서 “두 byte만 보내고 바로 EAGAIN”을 재현하려고 OS buffer 크기와 scheduler에 의존하지 않습니다.

반면 실제 `send` flags, kernel `SIGPIPE`, readiness notification과의 결합은 직접 시험하지 않습니다. 그 부분은 smoke/fairness와 플랫폼 실행이 보완합니다.

## 4. `928594ec160c` — event backend를 interface seam으로 대체하다

`FakeEventManager`는 fd interest map을 직접 보관하며 add/update 실패를 지정한 다음 호출에 던집니다. server는 실제 loopback listener와 accepted socket을 사용하되, ready event와 backend failure는 test가 결정합니다.

검증 matrix:

| 사례 | 주입/행동 | 직접 관찰하는 terminal state |
| --- | --- | --- |
| client event registration 실패 | 다음 `addFd` throw | connection map empty, backend에 client fd 없음 |
| connect callback 제거+throw | callback에서 disconnect 후 exception | stale pointer 접근 없이 map empty |
| line callback 제거+throw | readable event에서 disconnect 후 exception | event handler가 제거된 object를 재사용하지 않음 |
| write interest update 실패 | 다음 `updateFd` throw | `sendTo=false`, map/backend 모두 fd 제거 |
| queue limit close | 작은 cap에 oversized output | close request가 실제 disconnect로 끝남 |

error handler도 일부 사례에서 exception을 던지게 해 `reportError` 자체가 cleanup을 깨지 않는지 확인합니다.

fake backend는 deterministic rollback evidence를 제공하지만 epoll/kqueue의 native errno와 shadow map 구현을 대체하지는 않습니다. CI에서 실제 backend source를 함께 compile하고 network suite를 실행해야 합니다.

## 5. `5edcafda8a4d` — server failure가 application state까지 닫히는가

application lifetime test는 `maxPendingBytes=1`인 server에 NICK/USER 등록을 보냅니다. 첫 welcome numeric도 queue에 들어갈 수 없으므로 다음 chain이 실행됩니다.

```text
maybeRegister
→ sendNumeric
→ Server::sendTo queue rejection
→ refreshInterest가 close-requested/no-pending 확인
→ Server::disconnect
→ application onDisconnect
→ registry/channel cleanup
→ sendNumeric false 반환
→ registration sequence 중단
```

assertion은 세 계층을 함께 봅니다.

- server connection count가 0
- server는 계속 running
- `client_registered` log가 없음

Connection unit test가 “queue가 false를 반환한다”까지만 보았다면, 이 test는 그 false가 application의 stale state와 잘못된 success log를 남기지 않는지 확인합니다.

이 commit의 초기 application test는 registration failure 한 사례에 집중합니다. compound MODE 중간의 sender 제거는 후속 `aee5edebe294`가 같은 test binary에 추가하지만, Thread 09 commit map은 검증 계층을 만든 기준 commit만 포함합니다.

## 6. `de1dd0fc30d0` — readiness 공정성과 느린 수신자 격리

단위 test가 각 연결의 local state를 증명해도 event loop가 많은 fd 사이에서 progress를 제공하는지는 별도 문제입니다. `tests/irc_event_fairness.py`는 두 scenario를 실제 server process에서 실행합니다.

### 160개 동시 connection

상수는 다음과 같습니다.

```python
PEER_COUNT = 160
```

160개 socket을 열어 client 쪽 `selectors.DefaultSelector`에 등록하고, 각 peer가 고유 token의 PING을 보냅니다. 15초 deadline 안에 모든 socket에서 정확히 대응하는 PONG을 받아야 합니다.

```text
peer-0   → PING fanout-0   → PONG ... fanout-0
...
peer-159 → PING fanout-159 → PONG ... fanout-159
```

한 socket만 반복 처리하고 다른 ready fd를 굶기거나, event backend가 일부 descriptor를 잃으면 pending set이 남아 실패합니다. 160이라는 수는 production capacity를 증명하는 benchmark가 아니라, 소수 connection smoke보다 fd readiness 집합을 훨씬 넓히는 회귀 규모입니다.

### 읽지 않는 recipient 옆의 unrelated progress

세 client를 등록합니다.

- `slow`: receive buffer를 1024로 낮추고 이후 response를 읽지 않음
- `sender`: slow에게 4096개 PRIVMSG 전송
- `probe`: 무관한 PING을 보냄

```python
FLOOD_FRAME_COUNT = 4096
FLOOD_PAYLOAD = "x" * 400
```

server는 이 test에서 rate limit 10000/60, pending limit 16 MiB, max connections 256으로 시작합니다. sender의 flood가 slow socket의 kernel/connection output을 압박하는 동안 probe의 exact PONG이 15초 안에 와야 합니다.

이 test가 증명하려는 것은 slow client에 모든 4096 frame이 전달된다는 것이 아닙니다. **한 connection의 blocked output이 event loop thread를 blocking send에 묶어 unrelated connection의 read/write progress를 멈추지 않는다**는 것입니다.

### 종료까지 포함

test는 SIGTERM을 보내고 최대 10초 안에 exit 0을 요구합니다. 실패 시 server log를 출력하고 process를 kill해 harness orphan을 남기지 않습니다.

### 한계

- 160 connections와 이 payload 규모까지만 관찰합니다.
- WAN packet loss, 매우 긴 세션, 수만 connection 성능을 증명하지 않습니다.
- 느린 recipient의 queue가 반드시 cap을 초과해 끊기는지보다 unrelated progress에 초점을 둡니다.
- 성능 수치나 latency percentile을 기록하는 benchmark가 아닙니다.

## 7. `416efc91e580` — 플랫폼 차이를 개발자 환경 밖에서도 강제하다

GitHub Actions workflow는 push와 pull request에서 두 종류의 job을 실행하도록 정의합니다.

### platform regression matrix

```yaml
matrix:
  os:
    - ubuntu-latest
    - macos-latest
```

각 platform에서 warning을 error로 처리하는 기본 flags로 `make -j2`를 실행하고 `make test`를 호출합니다. 같은 source가 Linux에서는 epoll, macOS에서는 kqueue backend를 선택하므로 공통 `EventManager` 계약뿐 아니라 두 native implementation이 build와 network regression을 통과해야 합니다.

`make test`에 연결된 계층은 다음과 같습니다.

```text
production build
Connection scripted test
Server lifetime fake-backend test
Application lifetime test
IRC smoke client
IRC exact contract (`irc_smoke.sh`가 `irc_contract.py` 호출)
Event fairness test
```

`make test`는 `tests/irc_smoke.sh`를 실행하고, 이 shell harness가 먼저 `tools/irc_smoke_client.py`, 이어서 `tests/irc_contract.py`를 같은 server process에 연결합니다. 따라서 exact contract도 CI의 canonical suite에 포함됩니다. workflow가 개별 test command를 복제하지 않고 `make test` 하나를 호출하므로 local과 CI entry point가 갈라지는 것을 막습니다.

### Linux ASan·UBSan

별도 job은 다음 flags로 전체 suite를 다시 build/run합니다.

```text
-O1
-fno-omit-frame-pointer
-fsanitize=address,undefined
ASAN detect_leaks=1, halt/abort on error
UBSAN halt_on_error=1, stacktrace
```

focused lifetime tests가 의도한 use-after-free, buffer/offset 오류, leak가 실제 instrumentation에서 진단되면 즉시 job이 실패합니다.

### CI가 추가로 보장하지 않는 것

- workflow file이 존재한다는 사실만으로 특정 run이 성공했다고 주장할 수는 없습니다.
- ThreadSanitizer, MemorySanitizer, fuzzing은 포함하지 않습니다.
- Ubuntu/macOS latest 외 compiler·OS 조합은 포함하지 않습니다.
- timing-sensitive fairness test가 모든 CI 환경에서 영구히 flaky하지 않다는 수학적 보장은 없습니다.

## 검증 계층별 역할 정리

| 계층 | 실제 process/socket | 실패 주입 | exact wire | 내부 state 직접 관찰 | 주요 위험 |
| --- | :---: | :---: | :---: | :---: | --- |
| smoke | O | X | 부분 | X | 전체 기능 연결 |
| IRC contract | O | 제한적 | O | X | public CLI/wire/lifecycle |
| Connection test | X | scripted send | 해당 없음 | O | offset, cap, errno branch |
| Server lifetime | loopback 일부 | fake add/update/callback | 해당 없음 | O | rollback, stale `Connection*` |
| Application lifetime | loopback 일부 | tiny cap/fake update | 일부 | O | stale client/channel state |
| Event fairness | O | network pressure | PONG exact | 간접 | high-fd progress, slow peer isolation |
| CI/sanitizer | O | 위 suite 재사용 | 위 test에 따름 | instrumentation | platform drift, memory/UB |

## 최종 판단

이 history의 핵심은 test 수가 늘었다는 사실이 아닙니다. production invariant에 맞춰 관찰 기법이 분화됐다는 점입니다.

- wire에 보이는 결과는 exact contract로
- syscall interleaving은 scripted seam으로
- container/backend partial state는 fake owner로
- event-loop 공정성은 다중 socket pressure로
- 잠복 memory/UB와 backend portability는 CI matrix로

어느 한 계층도 나머지를 대체하지 않습니다. smoke만으로는 overflow를 못 보고, unit test만으로는 실제 readiness 공정성을 못 보며, sanitizer만으로는 잘못된 numeric text를 판단하지 못합니다.

> 검사 범위: 각 test/CI commit의 source와 Makefile/workflow diff를 확인했습니다. GitHub Actions의 실제 실행 이력이나 성공 상태는 조회하지 않았고, 이 환경에서 suite를 직접 실행하지도 않았습니다.
