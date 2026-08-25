# Thread 02. 프로토콜 경계, 식별자와 등록 전이

## 문제 정의

TCP 연결이 성립했다고 해서 곧바로 IRC 사용자로 취급할 수는 없습니다. transport가 넘긴 한 줄을 protocol message로 해석하고, 연결별 PASS·NICK·USER 상태를 축적하며, nickname의 전역 유일성을 유지한 뒤에만 일반 명령을 허용해야 합니다.

이 Thread는 다음 경계를 복원합니다.

```text
Connection이 완성 line 방출
        ↓
IrcMessage parser가 prefix/command/params로 정규화
        ↓
ClientRegistry가 fd별 등록 상태와 nickname 역색인 소유
        ↓
IrcApplication이 등록 전 허용 명령과 등록 완료 전이를 조율
```

최종 불변 조건은 세 가지입니다.

- malformed line은 handler로 전달되지 않습니다.
- canonical nickname 하나는 동시에 하나의 fd만 가리킵니다.
- `registered`는 PASS·NICK·USER 조건이 모두 충족된 한 지점에서만 `true`가 됩니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `a22bf6ddbd75` | `feat(parser): IRC 메시지 값과 직렬화 정의` | B | `IRC_PROTOCOL` | parser와 reply가 공유할 message 표현 도입 |
| 2 | `c31a32b6cb24` | `feat(parser): IRC 한 줄 구문 해석 구현` | A | `IRC_PROTOCOL, RISK` | line을 prefix·command·params로 분해하고 오류 차단 |
| 3 | `aeb1e9b709b9` | `feat(reply): IRC 서버 응답 생성` | B | `IRC_PROTOCOL` | numeric, 일반 message, hostmask 생성 통일 |
| 4 | `991b76b8d793` | `feat(client): 연결별 등록 상태 저장` | B | `IDENTITY` | fd별 PASS/NICK/USER/registered 상태 보관 |
| 5 | `b47135c51cfc` | `feat(client): 닉네임 색인 관리` | A | `IDENTITY, RISK` | case-insensitive nickname 역색인과 갱신 순서 확립 |
| 6 | `0b5ae6aef328` | `feat(app): IRC 동작 조율 계약 정의` | S | `CORE, ARCH, INTEGRATION` | transport callback, registry, command handler의 중심 경계 정의 |
| 7 | `2ed9331124bb` | `feat(app): 연결 수명 콜백 조율` | A | `LIFECYCLE, IRC_PROTOCOL, INTEGRATION` | connect/line/disconnect를 application state에 연결 |
| 8 | `035e1137e0dd` | `feat(app): 등록 전 명령 분배 구현` | A | `IRC_PROTOCOL, IDENTITY, RISK` | PASS/NICK/USER/PING/QUIT와 등록 후 명령을 분리 |
| 9 | `582317254e24` | `feat(registration): PASS 인증 상태 처리` | B | `IDENTITY, IRC_PROTOCOL` | password 검증과 실패 종료 처리 |
| 10 | `80a639321bad` | `feat(registration): 닉네임 검증과 색인 갱신` | B | `IDENTITY, IRC_PROTOCOL` | nickname 형식·충돌 검사 후 registry 반영 |
| 11 | `d9e420b570a0` | `feat(registration): USER 정보와 환영 응답 연결` | B | `IDENTITY, IRC_PROTOCOL` | USER 저장과 단일 등록 완료 전이 구현 |
| 12 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | `VERIFICATION, INTEGRATION, RISK` | 실제 socket에서 framing·등록·후속 명령을 통합 검증 |

## 1. line과 message 사이의 protocol boundary

### `a22bf6ddbd75` — 하나의 값 표현

`IrcMessage`는 optional prefix, 대문자로 정규화될 command, parameter vector를 소유합니다. 직렬화도 이 표현을 사용하므로 parser output과 outbound message가 같은 구조를 공유합니다.

이 commit은 parser 자체보다 표현 선택이 핵심입니다. handler가 원문 offset이나 공백 규칙을 다시 해석하지 않고 `message.command`, `message.params`만 사용하게 만듭니다.

### `c31a32b6cb24` — parse가 handler 앞에서 실패를 끝내다

parser는 매 호출마다 output message를 초기화한 뒤 다음 순서로 한 줄을 읽습니다.

```text
앞뒤 line terminator 정리
  → 전체 길이 검사
  → ':'로 시작하면 첫 공백까지 prefix
  → command token 추출·대문자화
  → 일반 parameter 반복
  → parameter 첫 글자가 ':'이면 나머지 전체를 trailing parameter 하나로 저장
```

핵심은 trailing parameter입니다. `PRIVMSG bob :hello there`의 `hello there`는 두 parameter가 아니라 하나여야 합니다.

대략적인 scan 형태는 다음과 같습니다.

```cpp
if (line[position] == ':') {
    message.params.push_back(line.substr(position + 1));
    position = line.size();
    break;
}
message.params.push_back(nextToken(line, position));
```

parser는 빈 command, 허용 길이 초과, 잘못된 prefix/parameter 배치에 오류 문자열을 반환합니다. `IrcApplication`은 이 결과가 실패면 numeric 417을 queue하고 handler dispatch를 하지 않습니다. 따라서 개별 명령 구현은 framing이나 command 대소문자 변환을 반복하지 않습니다.

이 Thread가 보장하는 것은 project가 정의한 line grammar입니다. 모든 RFC 확장 prefix·tag 문법까지 구현한다는 뜻은 아닙니다.

### `aeb1e9b709b9` — outbound 형식의 단일 생성점

`Replies`는 numeric code를 세 자리 문자열로 만들고, server prefix와 target, params, trailing text를 조합합니다. 사용자 prefix는 nickname/user/host로 hostmask를 만듭니다.

이 작은 commit이 중요한 이유는 각 handler가 `":server 433 ...\r\n"` 같은 wire text를 제각각 조립하지 않게 하기 때문입니다. 이후 정확한 contract test는 이 생성점이 만든 frame 전체를 비교할 수 있습니다.

## 2. fd 상태와 nickname identity를 분리하다

### `991b76b8d793` — 연결별 등록 상태

`ClientState`는 fd와 다음 상태를 보관합니다.

```text
passOk
hasNick / nick
hasUser / user / realname
registered
host
```

`ClientRegistry`의 주 owner는 fd map입니다. `state(fd)`는 없으면 기본 상태를 생성하고, `find`는 생성하지 않고 조회합니다. `fds()`는 maintenance나 전체 cleanup에서 map을 순회하는 동안 mutation이 일어나도 안전하도록 snapshot을 반환합니다.

### `b47135c51cfc` — nickname 역색인의 갱신 순서

nickname lookup을 매번 모든 client state에서 선형 검색하지 않고 canonical nickname → fd 역색인을 둡니다. canonicalization은 대소문자를 통일하므로 `Alice`와 `ALICE`는 같은 identity입니다.

nickname 변경은 다음 순서로 수행됩니다.

1. fd의 현재 nickname이 있으면 old canonical key를 역색인에서 제거
2. `ClientState.nick`와 `hasNick` 갱신
3. new canonical key → fd 등록

이 순서가 중요한 이유는 한 fd가 이름을 바꾼 뒤 old nickname이 계속 점유 상태로 남지 않아야 하기 때문입니다. 반대로 새 key를 먼저 넣고 이후 단계가 실패하는 transaction은 이 commit의 API에는 없습니다. handler가 형식과 충돌 검사를 먼저 끝낸 뒤 `setNickname`을 호출합니다.

연결 삭제도 fd state뿐 아니라 그 client가 소유한 nickname index를 함께 제거합니다. 이 작업이 누락되면 이미 끊어진 client의 이름을 새 연결이 사용할 수 없습니다.

## 3. `0b5ae6aef328` — application을 protocol authority로 세우다

이 S급 commit은 `IrcApplication`이 무엇을 소유하고 무엇을 server에 위임하는지 정합니다.

```text
Server: socket, readiness, Connection 수명
IrcApplication: ClientRegistry, command dispatch, registration rule, reply 생성
```

`IrcApplication`은 `Server&`, password, server name과 client registry를 가집니다. public callback은 connect, line, disconnect로 제한하고, 실제 PASS/NICK/USER 등의 handler는 내부 구현으로 둡니다.

이 경계 덕분에 transport는 “이 line이 등록 전에 허용되는가?”를 알 필요가 없고, registry는 “PASS 실패 시 어떤 numeric을 보내야 하는가?”를 알 필요가 없습니다. 등록 규칙은 application 한 곳에서 결정됩니다.

## 4. transport callback을 등록 state machine에 연결하다

### `2ed9331124bb` — connect, line, disconnect

- `onConnect`는 fd와 peer address로 초기 `ClientState`를 만듭니다. server password가 비어 있으면 `passOk`를 처음부터 참으로 둡니다.
- `onLine`은 registry에 fd가 없으면 connect state를 보완하고, parser를 호출합니다. parse 실패는 417로 끝나며 성공한 message만 dispatch합니다.
- `onDisconnect`는 application의 client state를 제거합니다.

이 commit 시점에는 channel membership cleanup이 아직 없습니다. Thread 03의 `a147d6994d58`이 disconnect를 nickname index뿐 아니라 모든 channel state까지 확장합니다.

### `035e1137e0dd` — 등록 전 command gate

명령 분배 순서는 등록 상태보다 먼저 처리해야 하는 명령과 등록 이후 명령을 분리합니다.

```text
PASS / NICK / USER / PING / QUIT
    → 등록 여부와 무관하게 각 handler로
그 밖의 command
    ├─ registered == false → 451
    └─ registered == true  → 일반 command dispatch 또는 421
```

PASS, NICK, USER를 등록 전에 허용하지 않으면 등록 자체가 불가능하고, PING/QUIT까지 막으면 handshake 중 연결 관리가 부자연스러워집니다. 반대로 PRIVMSG 같은 명령을 먼저 dispatch하면 identity가 완성되지 않은 사용자가 message를 보낼 수 있습니다.

## 5. 세 입력을 하나의 등록 완료 전이로 모으다

### `582317254e24` — PASS

PASS는 이미 등록된 client에게 462, parameter가 없으면 461을 반환합니다. password가 일치하지 않으면 464를 보내고 연결 종료를 요청합니다. 일치할 때만 `passOk=true`가 됩니다.

중요한 점은 실패한 password가 단순히 `passOk=false`로 남아 다음 시도를 무한히 허용되는 경로가 아니라, 명시적 종료 요청으로 이어진다는 것입니다.

### `80a639321bad` — NICK

handler는 parameter 유무, nickname 형식, canonical collision을 검사한 뒤에만 registry index를 갱신합니다. 충돌 검사 전에 기존 index를 제거하지 않으므로 실패한 이름 변경이 현재 nickname을 잃게 만들지 않습니다.

오류는 대표적으로 다음처럼 구분됩니다.

- parameter 없음 → 431
- 형식 오류 → 432
- 이미 사용 중 → 433

등록 전 NICK 설정과 등록 후 NICK 변경은 같은 registry API를 사용하지만, peer broadcast는 이후 channel Thread에서 추가됩니다.

### `d9e420b570a0` — USER와 `maybeRegister`

USER는 최소 네 parameter를 요구하고, 이미 USER 정보가 있거나 등록 완료 뒤 다시 호출하면 462를 반환합니다. 필요한 값을 저장한 뒤 PASS/NICK/USER handler 모두 `maybeRegister(fd)`에 도달할 수 있습니다.

등록 완료 조건은 한 곳에 모입니다.

```cpp
if (client.registered || !client.passOk || !client.hasNick || !client.hasUser)
    return;
client.registered = true;
sendNumeric(fd, 1, /* welcome */);
sendNumeric(fd, 2, /* host */);
sendNumeric(fd, 3, /* server info */);
```

`registered`를 환영 응답 전에 먼저 세우므로 같은 line 처리 중 재차 `maybeRegister`가 호출돼도 001~003을 중복 전송하지 않습니다. 다만 후속 Thread 08에서 밝혀지듯, output queue 실패가 즉시 연결 제거를 일으킬 수 있는데도 연속 numeric 전송과 등록 log를 계속하는 위험은 이 시점에 남아 있습니다.

## 6. `6b4a7738a285` — 실제 TCP에서 경계를 함께 통과시키다

smoke test는 server process를 띄우고 Python socket client 여러 개로 다음 흐름을 통과합니다.

- 잘못된 password와 nickname collision
- PASS/NICK/USER 등록과 welcome numerics
- `PI`와 `NG ...\r\n`으로 나눠 보낸 frame이 하나의 PING으로 복원되는지
- 등록 뒤 JOIN, topic, direct/channel PRIVMSG, INVITE, MODE, KICK, PART

분할 frame 사례가 transport와 parser의 경계를 직접 연결합니다.

```python
alice.send_raw(b"PI")
time.sleep(0.05)
alice.send_raw(b"NG :split-token\r\n")
alice.expect(" PONG ")
```

이 테스트는 실제 socket과 process를 사용하는 broad integration evidence입니다. 그러나 당시 `expect`는 주로 substring을 찾으므로 정확한 parameter 순서, 불필요한 추가 frame, 모든 numeric의 exact wire text를 완전히 고정하지는 않습니다. 그 역할은 Thread 09의 `e5e6c57db80d` contract test가 맡습니다.

## 등록 state machine 최종 정리

| 상태 | 허용되는 주요 입력 | 전이 |
| --- | --- | --- |
| 연결 직후 | PASS, NICK, USER, PING, QUIT | 각 field를 독립적으로 축적 |
| password 실패 | PASS | 464 queue 후 close request |
| 일부 정보만 존재 | PASS/NICK/USER 보완 | 아직 `registered=false` |
| `passOk && hasNick && hasUser` | 마지막 요건을 채운 handler | `registered=true`, 001~003 |
| 등록 완료 | 일반 IRC command | command별 authority 검사로 이동 |
| disconnect | 없음 | fd state와 nickname index 제거 |

## 이 Thread가 보장하는 것과 남기는 것

보장:

- transport가 방출한 한 줄은 parser를 통과한 뒤에만 handler로 갑니다.
- nickname lookup은 canonical key를 사용하고 연결 종료·이름 변경 때 index가 정리됩니다.
- 일반 명령은 등록 완료 전 실행되지 않습니다.
- 등록 완료와 welcome sequence는 하나의 gate에서 결정됩니다.

이 Thread 밖:

- nickname 변경을 shared-channel peer에게 fan-out하고 disconnect 시 membership을 정리하는 일은 Thread 03입니다.
- registration timeout, rate limit, heartbeat 같은 시간·부하 정책은 Thread 04와 06입니다.
- output 실패가 등록 handler 도중 연결을 제거하는 재진입 문제는 Thread 08입니다.
- exact CLI/wire contract와 플랫폼 회귀는 Thread 09입니다.

> 검사 범위: 각 commit의 exact GitHub diff와 해당 SHA의 source/test를 확인했습니다. 로컬 build와 smoke 실행은 수행하지 않았습니다.
