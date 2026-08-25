# Thread 03. 채널 권한, 팬아웃과 종료 정리

## 이 Thread의 중심 질문

채널은 단순한 이름→socket 목록이 아닙니다. 구성원, 운영자, 초대, topic, mode가 서로 모순 없이 변해야 하고, 한 사용자의 행위가 어느 peer에게 전달되는지도 channel membership으로 계산해야 합니다. 연결이 끊기거나 nickname이 바뀌면 이 파생 상태를 모두 정리한 뒤에만 identity를 제거할 수 있습니다.

최종 불변 조건은 다음과 같습니다.

- 운영자는 항상 현재 구성원입니다.
- 한 채널의 fan-out 대상은 전송 시작 시점의 membership snapshot으로 계산합니다.
- 여러 공통 채널을 가진 peer에게 QUIT/NICK은 한 번만 전달합니다.
- client 삭제 전에 필요한 old prefix와 recipient를 확보하고, membership·operator·빈 channel·nickname index를 모두 정리합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `786ed5d839d9` | `feat(channel): 채널 상태 계약 정의` | A | `ARCH, CHANNEL_STATE` | channel aggregate가 보유할 상태와 연산 정의 |
| 2 | `966f5663dbcc` | `feat(channel): 구성원과 운영자 상태 관리` | B | `CHANNEL_STATE` | membership/operator 불변 조건 구현 |
| 3 | `0d0d850f007d` | `feat(channel): 주제·초대·모드와 이름 규칙 구현` | B | `CHANNEL_STATE, IRC_PROTOCOL` | topic/invite/mode와 channel name 유효성 규칙 추가 |
| 4 | `c1762e011fd6` | `feat(channel): 채널 탐색과 대상 해석 지원` | B | `CHANNEL_STATE` | application의 channel map과 lookup helper 구성 |
| 5 | `46e2b7785bee` | `feat(channel): 채널 방송 대상 팬아웃 지원` | A | `CHANNEL_STATE, INTEGRATION` | membership snapshot 및 공통 채널 recipient 중복 제거 |
| 6 | `a147d6994d58` | `feat(channel): 구성원 정리와 식별자 변경 방송` | S | `CORE, CHANNEL_STATE, LIFECYCLE` | PART/QUIT/NICK이 membership와 identity를 일관되게 변경 |
| 7 | `7ac793d3b695` | `feat(message): 채널 대상 메시지 방송` | B | `IRC_PROTOCOL, CHANNEL_STATE` | PRIVMSG 대상이 nickname인지 channel인지 분기 |
| 8 | `22e5f82bc693` | `feat(channel): JOIN 채널 입장 처리` | A | `CHANNEL_STATE, IRC_PROTOCOL, RISK` | 생성·권한·초대 소비·응답 순서를 연결 |
| 9 | `0e601da7d2bc` | `feat(channel): 채널 모드 조회와 i·t 변경` | B | `CHANNEL_STATE, IRC_PROTOCOL` | channel mode query와 operator gate 도입 |
| 10 | `bce6933f69ab` | `feat(channel): 채널 운영자 모드 변경` | B | `CHANNEL_STATE, IRC_PROTOCOL` | mode별 argument cursor와 +o/-o 처리 추가 |

## 1. `Channel`을 하나의 일관된 상태 단위로 만들다

### `786ed5d839d9` — aggregate 경계

`Channel`은 이름과 함께 다음 상태를 소유합니다.

```text
members: fd 집합
operators: fd 집합
invited nicknames
optional topic
invite-only / topic-protected mode
```

socket이나 `Connection`을 소유하지는 않습니다. fd는 application registry와 server connection을 가리키는 식별자일 뿐입니다. 이 구분 덕분에 channel destructor가 socket을 닫거나, connection owner와 수명이 충돌하지 않습니다.

### `966f5663dbcc` — membership와 operator의 결합 규칙

구성원 추가는 이미 존재해도 중복되지 않는 set 삽입입니다. 첫 구성원을 operator로 지정할 수 있고, `setOperator(fd, true)`는 해당 fd가 member일 때만 성공합니다.

구성원 제거는 두 집합을 함께 갱신합니다.

```cpp
members_.erase(fd);
operators_.erase(fd);
```

이 한 줄의 결합이 중요합니다. KICK/PART/QUIT 뒤 운영자 집합에 탈퇴한 fd가 남으면 MODE나 TOPIC 권한 검사가 잘못 통과합니다.

### `0d0d850f007d` — channel 이름과 부가 상태

channel name은 최소 두 글자이며 `#` 또는 `&`로 시작하는지, whitespace·comma·bell 문자를 포함하지 않는지 검사합니다. 이 commit은 channel 이름 자체를 case-fold하지 않습니다. canonicalization은 초대 목록의 nickname에만 적용됩니다. topic은 유무와 text를 함께 관리하고, invite는 lowercase nickname으로 저장·조회·소비합니다.

`modeString()`은 현재 invite-only와 topic-protected 상태를 wire 응답에 사용할 문자열로 조립합니다. 이 commit은 state primitive만 만들고, 누가 mode를 바꿀 수 있는지는 handler에서 결정합니다.

## 2. application map과 recipient 계산

### `c1762e011fd6` — channel lookup의 오류 의미

`IrcApplication`은 `std::map<std::string, Channel>`을 소유합니다. `ensureChannel`은 전달받은 channel name key가 없으면 생성하고, command용 lookup은 다음 오류를 구분합니다. 이 시점의 map은 channel 이름에 별도 case-folding을 적용하지 않습니다.

- channel 자체가 없음 → 403
- channel은 있으나 membership 필수 command에서 사용자가 비회원 → 442

handler마다 map lookup과 numeric 조립을 반복하지 않도록 authority 검사 전의 공통 경계를 만듭니다.

### `46e2b7785bee` — fan-out snapshot

`broadcastToChannel`은 `Channel::members()`가 반환한 vector snapshot을 순회하고, 제외 fd를 건너뜁니다. 전송 도중 한 recipient의 queue 실패가 disconnect와 membership mutation을 유발해도 현재 vector iterator는 channel 내부 container에 매달려 있지 않습니다.

사용자가 공유하는 모든 채널의 peer를 찾을 때는 `std::set<int>`에 recipient를 모읍니다.

```cpp
std::set<int> peers;
for (각 channel) {
    if (!channel.hasMember(fd))
        continue;
    for (member in channel.members())
        if (member != fd)
            peers.insert(member);
}
```

A와 B가 `#one`, `#two`를 모두 공유해도 NICK 또는 QUIT frame은 B에게 한 번만 전달됩니다. channel별 broadcast를 단순 반복하면 동일한 protocol event가 중복됩니다.

이 commit 시점에는 send 실패 뒤 application container를 다시 조회하는 방어가 충분하지 않습니다. 그 재진입 문제는 Thread 08에서 수정됩니다.

## 3. `a147d6994d58` — identity가 사라지기 전에 필요한 정보를 확보하다

이 Thread의 핵심 commit입니다. PART, QUIT, nickname 변경을 단순한 field update가 아니라 fan-out과 cleanup을 포함한 state transition으로 만듭니다.

### PART ALL과 map mutation

`partAllChannels(fd, reason)`은 먼저 사용자가 속한 channel 이름들을 vector에 복사한 뒤 하나씩 `partChannel`을 호출합니다.

```text
channel map 순회 중 membership 제거/빈 channel erase
```

를 직접 수행하면 iterator가 무효화될 수 있으므로, 이름 snapshot과 mutation phase를 분리합니다.

각 PART는 다음 순서입니다.

1. channel 존재와 membership 확인
2. old identity로 PART frame 생성
3. 현재 구성원에게 broadcast
4. member/operator에서 fd 제거
5. channel이 비었으면 map에서 제거

방송 전에 membership을 제거하면 떠나는 사용자 자신이 PART를 보지 못하거나 recipient 계산이 달라집니다.

### QUIT와 disconnect cleanup

`removeClientState(fd, reason, notifyPeers)`는 registry를 바로 지우지 않습니다. 먼저 `ClientState`를 값으로 복사해 old nick/user/host를 보존하고, 사용자가 속한 모든 channel에서 peer를 set으로 수집합니다.

```cpp
const ClientState client = *found;
const std::string quitLine = Replies::formatMessage(
    prefixFor(client), "QUIT", std::vector<std::string>(1, reason));
```

그 뒤 recipient에게 한 번씩 QUIT을 보내고, 모든 channel에서 fd를 제거합니다. 빈 channel 이름을 별도 vector로 모아 순회가 끝난 뒤 erase하고, 마지막에 `_clients.erase(fd)`를 호출합니다.

```text
old identity snapshot
→ peer set 계산
→ QUIT fan-out
→ channel member/operator 제거
→ empty channel 제거
→ nickname reverse index 포함 client erase
```

identity를 먼저 지웠다면 prefix를 만들 수 없고, membership을 먼저 모두 지웠다면 어느 peer가 QUIT을 받아야 하는지 계산할 수 없습니다.

### 등록 후 NICK 변경

nickname 변경 전 `wasRegistered`와 old prefix를 저장합니다. registry가 새 nickname으로 index를 갱신한 뒤, 등록된 client였다면 old prefix를 사용해 공통 peer와 자기 자신에게 NICK frame을 보냅니다.

```text
:old!user@host NICK new
```

이 순서로 recipient는 event source를 old identity로 해석하고, frame parameter에서 새 identity를 얻습니다.

### 이 commit이 남긴 한계

fan-out 중 output queue가 실패하면 server가 sender 또는 channel member를 즉시 제거할 수 있습니다. 그 뒤 보관 중인 `Channel&`나 `ClientState&`를 계속 사용하면 dangling reference가 됩니다. 이 Thread는 정상적인 cleanup 순서를 세웠고, 재진입 안전성은 Thread 08의 `728aaabc4012`와 `d48e1f1f8c04`에서 보강됩니다.

## 4. command가 channel authority를 사용하는 방식

### `7ac793d3b695` — PRIVMSG target 분기

각 comma-separated target이 channel name이면 channel 존재와 sender membership을 확인합니다. 실패하면 404를 보내고, 성공하면 sender를 제외한 member snapshot에 message를 fan-out합니다. nickname target이면 reverse index로 fd를 찾아 직접 전송하고, 없으면 401입니다.

이 commit은 “channel에 존재하기만 하면 누구나 쓸 수 있다”가 아니라 **현재 member만 해당 channel로 message를 보낼 수 있다**는 정책을 구현합니다.

### `22e5f82bc693` — JOIN의 상태 전이 순서

JOIN은 여러 channel name을 comma로 받을 수 있고, `JOIN 0`은 모든 channel에서 떠나는 별도 명령입니다. 일반 입장 순서는 다음과 같습니다.

```text
name validation
→ ensureChannel
→ 이미 member면 NAMES만 재전송
→ +i이면 invite 확인
→ 입장 전 channel.empty() 기록
→ addMember(fd, firstMember)
→ nickname invite 소비
→ JOIN broadcast
→ topic reply
→ NAMES reply
```

첫 구성원을 operator로 만드는 결정은 `empty()`를 membership 추가 전에 확인해야 합니다. invite를 membership 확정 전에 지우면 이후 단계가 실패했을 때 초대만 사라질 수 있으므로, 권한 검사를 통과하고 member로 추가하는 시점에 소비합니다.

JOIN frame을 먼저 방송한 뒤 topic/NAMES를 보내므로 기존 구성원과 입장자가 같은 membership 전이를 관찰합니다. output 실패에 의한 중간 제거는 당시 완전히 방어되지 않았고 Thread 08에서 재조회가 추가됩니다.

### `0e601da7d2bc` — MODE query와 `+i`, `+t`

MODE target이 channel이면 channel handler로, nickname이면 user-mode 응답 경로로 나뉩니다. channel 조회는 비회원에게도 현재 mode string을 돌려줄 수 있지만, 변경은 member이면서 operator인 경우만 허용합니다.

mode string을 왼쪽부터 읽으며 `+`/`-`가 현재 adding 상태를 바꿉니다.

```text
MODE #room +it-i
             ↑ 상태를 유지하며 각 mode를 순서대로 적용
```

`i`는 invite-only, `t`는 topic-protected를 갱신하고 각각 MODE frame을 channel에 방송합니다. 모르는 mode는 472입니다.

### `bce6933f69ab` — mode 문자와 argument cursor 분리

`o`는 nickname parameter를 소비하므로 mode index와 parameter index를 따로 둡니다.

```cpp
bool adding = true;
std::size_t argIndex = 2;
for (char mode : modes) {
    // +/-는 adding만 갱신
    // o는 params[argIndex++] 사용
}
```

parameter가 없으면 461, nickname을 찾을 수 없거나 해당 channel member가 아니면 441입니다. 검사를 통과한 뒤에만 `setOperator(targetFd, adding)`을 호출하고 registry에 저장된 실제 nickname으로 mode broadcast를 만듭니다.

`Channel::setOperator` 자체도 비회원 승격을 거부하므로 handler와 aggregate 양쪽에 불변 조건이 있습니다.

## 상태와 fan-out의 최종 순서

| 사건 | fan-out에 필요한 snapshot | state mutation | 후속 정리 |
| --- | --- | --- | --- |
| JOIN | 입장 전 empty 여부, source prefix | member 추가, invite 소비 | JOIN → topic → NAMES |
| PART | channel name, source prefix, member snapshot | member/operator 제거 | empty channel erase |
| KICK | channel/target/source 정보 | target member/operator 제거 | empty channel erase |
| NICK | old prefix, 공통 peer set | nickname와 reverse index 갱신 | NICK을 중복 없이 방송 |
| QUIT/disconnect | full old ClientState, 공통 peer set | 모든 membership 제거 | empty channel 및 registry erase |
| MODE +o/-o | target fd와 nickname, source 권한 | operator set 변경 | channel MODE 방송 |

## 보장 범위

이 Thread가 보장합니다.

- channel의 구성원·운영자·초대·topic·mode가 한 aggregate에서 갱신됩니다.
- operator는 member subset으로 유지됩니다.
- fan-out은 snapshot과 set을 이용해 mutation 및 중복 전달 위험을 줄입니다.
- disconnect와 nickname 변경은 old identity를 잃기 전에 recipient와 frame을 계산합니다.

이 Thread가 단독으로 보장하지 않습니다.

- 전송 실패 callback이 application state를 재진입해 제거하는 상황의 dangling reference 방지는 Thread 08입니다.
- 느린 recipient 하나가 다른 connection의 event 처리를 막지 않는다는 실행 증거는 Thread 09입니다.
- rate limit, output queue cap, 최대 연결 수 같은 운영 보호는 Thread 04입니다.

> 검사 범위: commit map의 각 SHA에서 관련 `Channel`, application support, command handler diff를 확인했습니다. 실제 network test는 이 환경에서 실행하지 않았습니다.
