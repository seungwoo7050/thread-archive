# Thread 01. 이식 가능한 준비 이벤트와 논블로킹 전송

## 이 Thread가 해결하는 문제

서버가 Linux의 `epoll`과 macOS의 `kqueue`를 모두 지원하려면 커널별 event flag를 application에 그대로 노출해서는 안 됩니다. 동시에 non-blocking socket은 한 번의 `recv`나 `send`가 한 IRC line 또는 전체 응답을 처리한다는 가정을 허용하지 않습니다.

이 Thread는 다음 두 불변 조건을 하나의 실행 경로로 묶습니다.

1. 상위 서버는 backend와 무관한 `Read`, `Write`, `error`, `hangup`만 소비합니다.
2. 각 연결은 fd, 입력 조각, 아직 보내지 못한 출력 suffix를 한 객체가 소유하며, 성공한 syscall만 상태를 전진시킵니다.

최종적으로 `Server`는 fd별 `Connection`을 유일 소유하고, event backend는 준비 상태만 전달합니다. 읽기는 완성된 IRC line만 application에 넘기고, 쓰기는 `writeOffset_` 이후의 byte만 재시도합니다.

## Commit map

| 순서 | SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `8e8a3db87950` | `feat(event): 이벤트 준비 상태 계약 정의` | A | `ARCH, EVENT_IO` | backend 공통 event/interest 표현을 정의 |
| 2 | `d3f74e2857da` | `feat(event): kqueue 관심 상태 등록 구현` | B | `EVENT_IO` | kqueue filter 등록과 shadow interest 동기화 |
| 3 | `769cd3094f71` | `feat(event): kqueue 준비 이벤트 변환 구현` | B | `EVENT_IO` | native `kevent`를 공통 event로 변환 |
| 4 | `2284a0e0d8bb` | `feat(event): epoll 관심 상태 등록 구현` | B | `EVENT_IO` | epoll add/mod/del과 공통 interest 연결 |
| 5 | `f1320f357bca` | `feat(event): epoll 준비 이벤트 변환 구현` | B | `EVENT_IO` | native `epoll_event`를 공통 event로 변환 |
| 6 | `8864f253ac15` | `feat(connection): 스트림 연결 상태 계약 정의` | A | `ARCH, EVENT_IO, LIFECYCLE` | fd와 read/write 상태를 묶는 `Connection` 경계 정의 |
| 7 | `d71a2549c0eb` | `feat(connection): 소켓 소유권과 이동 수명 구현` | A | `LIFECYCLE, RISK` | fd 단일 소유와 move-only 수명 보장 |
| 8 | `9b601b69de4f` | `feat(connection): IRC 입력 프레임 추출 구현` | A | `EVENT_IO, IRC_PROTOCOL, RISK` | 임의 byte 조각에서 완성된 IRC line 추출 |
| 9 | `b00589d4b1b1` | `feat(connection): 논블로킹 수신 상태 처리` | A | `EVENT_IO, RISK` | `EINTR`, `EAGAIN`, EOF, hard error를 분리 |
| 10 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | `CORE, EVENT_IO, LIFECYCLE` | 전송된 prefix와 남은 suffix를 정확히 관리 |
| 11 | `e6492e27cc30` | `feat(server): 이벤트 서버 공개 계약 정의` | A | `ARCH, EVENT_IO, LIFECYCLE` | backend와 connection map을 소유하는 서버 경계 정의 |
| 12 | `4ad1227e5119` | `feat(server): 논블로킹 연결 수락과 등록 구현` | A | `EVENT_IO, LIFECYCLE` | accept된 fd를 non-blocking connection으로 등록 |
| 13 | `378d5304828d` | `feat(server): 준비 이벤트 루프 구동` | S | `CORE, ARCH, EVENT_IO` | listen/client readiness를 하나의 poll cycle로 처리 |
| 14 | `625ffc924de8` | `feat(server): 송신 큐와 쓰기 관심 상태 연결` | A | `EVENT_IO, LIFECYCLE, INTEGRATION` | pending output과 write interest를 동기화 |
| 15 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | `LIFECYCLE, RISK` | backend·map·callback을 한 disconnect 경로로 수렴 |

## 1. 커널별 flag를 상위 계층에서 제거하다

### `8e8a3db87950` — 공통 event 표현

`EventInterest`는 `Read`와 `Write`를 bitmask로 표현하고, `Event`는 fd와 준비 interest 외에 `error`, `hangup`, `errorCode`를 담습니다. `EventManager`는 `addFd`, `updateFd`, `removeFd`, `wait`만 노출합니다.

중요한 점은 이 추상화가 “모든 backend가 똑같다”고 가정하지 않는다는 것입니다. kqueue의 `EV_EOF`나 epoll의 `EPOLLRDHUP` 같은 차이는 backend 내부에서 해석하고, 상위 서버에는 다음 질문에 답할 정보만 넘깁니다.

- 이 fd에서 지금 읽을 수 있는가?
- 지금 쓰기를 계속할 수 있는가?
- 커널이 오류나 peer 종료를 보고했는가?

이 commit은 아직 실제 backend를 선택하지 않습니다. 대신 이후 구현이 지켜야 할 변환 경계를 고정합니다.

### `d3f74e2857da`, `769cd3094f71` — kqueue

kqueue 등록 구현은 관심 상태를 바꿀 때 필요한 `EV_ADD`, `EV_ENABLE`, `EV_DISABLE`, `EV_DELETE`를 구성합니다. 여기서 shadow interest map은 커널 변경이 성공한 뒤에만 갱신됩니다. `kevent`가 실패했는데 메모리 쪽 상태부터 바뀌면 다음 `updateFd`가 잘못된 이전 상태를 기준으로 delta를 계산하기 때문입니다.

준비 이벤트 변환은 최대 128개의 native event를 받고 다음처럼 공통 표현으로 바꿉니다.

- `EVFILT_READ` → `Read`
- `EVFILT_WRITE` → `Write`
- `EV_ERROR` → `error`, native data를 `errorCode`로 보존
- `EV_EOF` → `hangup`
- `EINTR`로 중단된 wait → 오류가 아니라 빈 준비 목록

### `2284a0e0d8bb`, `f1320f357bca` — epoll

epoll 쪽도 add/mod/del이 성공한 뒤 shadow map을 갱신합니다. read interest에는 peer half-close를 감지하기 위한 `EPOLLRDHUP`가 함께 등록되고, ready event의 `EPOLLERR`, `EPOLLHUP`, `EPOLLRDHUP`는 공통 `error`/`hangup`으로 변환됩니다. 오류 코드는 필요하면 `SO_ERROR`로 조회합니다.

두 backend의 핵심 공통점은 native flag를 억지로 일대일 대응시키는 데 있지 않습니다. **상위 서버가 동일한 lifecycle 결정을 내릴 수 있는 최소 정보로 정규화한다**는 데 있습니다.

## 2. `Connection`을 fd의 유일 owner로 만들다

### `8864f253ac15` — 연결 상태의 범위

`Connection`은 다음 상태를 한 곳에 묶습니다.

```text
fd
peer address
read buffer
write buffer + write offset
maximum IRC line length
peer-closed / close-requested / close reason
```

이 표현 덕분에 event loop는 raw socket state를 따로 추적하지 않고 다음 질문만 묻습니다.

- `readAvailable()`이 어떤 완성 line을 만들었는가?
- `wantsWrite()`가 참인가?
- `flushPending()`이 끝났는가, block됐는가, 실패했는가?
- 연결이 종료 요청 상태인가?

### `d71a2549c0eb` — move-only fd 소유권

복사는 금지되고 move construction/assignment만 허용됩니다. move 뒤 원본 fd는 `-1`이 되며, move assignment는 대상 객체가 이미 소유하던 fd를 먼저 닫습니다. destructor는 유효한 fd만 닫습니다.

이 규칙이 필요한 이유는 단순합니다. `Server`가 `unique_ptr<Connection>`을 map 안팎으로 이동할 때 두 객체가 같은 fd를 닫거나, 어느 객체도 닫지 않는 상태를 만들지 않아야 합니다.

`pendingBytes()`는 전체 buffer 크기가 아니라 `writeBuffer_.size() - writeOffset_`입니다. 이미 전송한 prefix는 저장 공간에 남아 있을 수 있어도 더 이상 pending 자원이 아닙니다.

## 3. byte stream을 IRC line으로 바꾸다

### `9b601b69de4f` — framing

TCP는 message boundary를 보존하지 않습니다. 한 번의 `recv`가 반쪽 line만 주거나 여러 line을 한꺼번에 줄 수 있습니다. 이 commit은 read buffer에서 `\n`을 찾고, 앞의 선택적 `\r`을 제거하여 완성된 line만 결과에 넣습니다.

최대 길이는 delimiter까지 포함해 검사됩니다. 허용 범위를 넘긴 frame을 계속 누적하지 않고 buffer를 비우며 연결 종료를 요청합니다. 따라서 application parser는 “아직 덜 온 line”을 받을 수 없습니다.

### `b00589d4b1b1` — non-blocking receive 결과의 분류

수신 loop는 syscall 결과를 다음처럼 구분합니다.

| `recv` 결과 | 상태 변화 |
| --- | --- |
| `> 0` | byte를 buffer에 append하고 line 추출을 계속 |
| `-1`, `EINTR` | 상태를 바꾸지 않고 같은 연산 재시도 |
| `-1`, `EAGAIN/EWOULDBLOCK` | 현재 cycle 종료, `wouldBlock=true` |
| `0` | peer가 송신 측을 닫았음을 기록 |
| 그 밖의 오류 | error와 close request를 기록 |

`EAGAIN`은 실패가 아니라 “현재 준비된 byte를 모두 소비했다”는 뜻입니다. 이 구분이 없으면 정상적인 non-blocking 흐름이 연결 오류로 처리됩니다.

## 4. `a10fe961e2b1` — 부분 송신을 suffix 상태로 표현하다

이 Thread의 첫 번째 핵심 commit입니다. non-blocking `send`는 요청한 모든 byte를 보내지 않을 수 있으므로, 성공한 byte만 소비하고 나머지를 다음 write-ready event까지 유지해야 합니다.

핵심 흐름은 다음과 같습니다.

```cpp
while (wantsWrite()) {
    const char* data = writeBuffer_.data() + writeOffset_;
    const std::size_t size = writeBuffer_.size() - writeOffset_;
    const ssize_t count = ::send(fd_, data, size, sendFlags());

    if (count > 0) {
        writeOffset_ += static_cast<std::size_t>(count);
        continue;
    }
    if (count == 0) {
        result.wouldBlock = true;
        break;
    }
    if (errno == EINTR)
        continue;
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        result.wouldBlock = true;
        break;
    }
    requestClose(/* send error */);
    break;
}
```

이 코드가 만드는 불변 조건은 다음과 같습니다.

```text
[0, writeOffset_)                  이미 성공적으로 전송한 prefix
[writeOffset_, writeBuffer_.size()) 아직 전송해야 할 suffix
```

`EINTR`, `EAGAIN`, 0-byte send는 offset을 전진시키지 않습니다. 양수 반환만 정확히 그 수만큼 전진시킵니다. 전부 전송하면 buffer와 offset을 함께 초기화하고, 앞부분이 오래 남아 저장 공간을 낭비하면 일정 크기 이후 compact합니다.

`queueLine`은 입력 끝의 CR/LF를 제거하고 정확히 한 번 `\r\n`을 붙입니다. Linux에서는 가능한 경우 `MSG_NOSIGNAL`을 사용해 끊어진 peer로의 send가 프로세스 전체 `SIGPIPE`로 번지지 않게 합니다.

이 commit이 아직 보장하지 않는 두 가지는 후속 Thread 07에서 고쳐집니다.

- pending-byte 한도 계산의 정수 overflow
- 비정상적인 send 구현이 요청 크기보다 큰 수를 반환할 때 offset 보호

## 5. `Server`가 backend, listener, connection map을 조율하다

### `e6492e27cc30` — 유일 소유 구조

`Server`는 event manager를 `unique_ptr`로, 연결을 `fd → unique_ptr<Connection>` map으로 소유합니다. application에는 connect/line/disconnect/error callback만 제공합니다.

```text
Server
├─ listen fd
├─ EventManager (epoll 또는 kqueue)
└─ connections_[fd] → unique_ptr<Connection>
```

map entry가 존재하는 동안 그 `Connection`이 fd와 read/write 상태의 authoritative owner입니다.

### `4ad1227e5119` — accept와 초기 등록

listen fd가 readable하면 accept를 반복하고, 새 socket에 close-on-exec와 non-blocking을 설정한 뒤 event backend와 connection map에 등록합니다. 한 client의 설정 또는 등록이 실패해도 accept loop 자체를 완전히 중단하지 않고 해당 fd를 닫습니다.

이 시점에는 event 등록과 map 삽입, callback 호출 사이의 rollback·재진입 문제가 완전히 해결되지 않았습니다. 특히 callback이 자기 연결을 제거한 뒤 raw pointer를 다시 사용하는 위험은 Thread 08의 `5dcd882f0763`에서 수정됩니다.

### `378d5304828d` — readiness-driven 실행 루프

두 번째 핵심 commit입니다. `start()`가 backend와 listen socket을 준비하고, `pollOnce()`는 ready event를 받아 listen fd와 client fd를 분기합니다.

```text
wait(timeout)
  ├─ listen fd + Read → accept 가능한 client를 반복 수락
  └─ client fd
       ├─ error → disconnect
       ├─ Read → recv, line callback, close 여부 확인
       ├─ Write → pending suffix flush
       └─ hangup → pending output이 없으면 disconnect
```

모든 연결을 매 cycle 순회하지 않고 커널이 준비됐다고 알린 fd만 처리합니다. 한 `Connection`의 buffer/offset 상태가 readiness 사이에서 유지되므로 부분 read/write가 자연스럽게 이어집니다.

### `625ffc924de8` — queue와 interest의 동기화

application이 line 또는 raw bytes를 enqueue하면 server는 해당 연결의 interest를 다시 계산합니다.

- pending output 없음, 정상 연결 → `Read`
- pending output 있음 → `Read | Write`
- close requested, 아직 pending output 있음 → `Write`
- close requested, pending output 없음 → 즉시 disconnect

close 요청 뒤 read interest를 끄는 것은 새로운 application input을 더 받지 않고 이미 queue된 종료 응답만 drain하기 위해서입니다.

### `7a6bc7e1276a` — 하나의 disconnect 수렴점

연결 해제는 다음 순서로 처리됩니다.

1. event backend에서 fd 제거
2. map의 `unique_ptr`을 local owner로 이동
3. map entry 삭제
4. disconnect callback 호출
5. local `unique_ptr` 파괴 시 fd close

callback 전에 map에서 제거하기 때문에 callback이 같은 fd를 다시 조회하면 이미 종료 상태임을 볼 수 있습니다. 전체 종료도 fd snapshot을 만든 뒤 각 fd를 같은 `disconnect` 경로로 보냅니다.

peer hangup이 보고돼도 pending output이 남아 있으면 즉시 버리지 않고 write를 먼저 시도합니다. 다만 이 시점의 callback 재진입 안전성은 완성되지 않았으며 Thread 08에서 fd 재조회 방식으로 강화됩니다.

## 최종 상태와 실패 수렴

| 경로 | authoritative state 변화 | 종료 조건 |
| --- | --- | --- |
| 정상 read | recv된 byte append → 완성 line만 방출 | `EAGAIN`에서 cycle 종료 |
| peer EOF | `peerClosed_` 기록 | pending output 처리 뒤 disconnect 가능 |
| 부분 write | 성공한 byte만 `writeOffset_` 증가 | 남은 suffix가 있으면 Write interest 유지 |
| hard send/recv error | close reason 기록 | server disconnect 경로로 수렴 |
| accept/setup 실패 | 해당 client fd만 close | listener와 기존 연결은 계속 유지 |
| application close request | Read 중단, pending output drain | queue가 비면 map/backend에서 제거 |
| server stop | fd snapshot을 순회해 모두 disconnect | map empty, listener close |

## 이 Thread의 경계

- IRC command 문법과 등록 권한은 Thread 02에서 다룹니다.
- 채널 membership과 broadcast recipient 계산은 Thread 03의 책임입니다.
- output queue의 정수 overflow와 scripted partial-send 검증은 Thread 07에서 다룹니다.
- callback이 연결을 제거할 수 있는 재진입 문제와 event 등록 rollback은 Thread 08에서 다룹니다.
- epoll/kqueue 양쪽에서 고연결 수와 느린 수신자 격리가 실제로 유지되는지는 Thread 09의 공정성·CI 검증에서 다룹니다.

> 검사 범위: 위 설명은 각 표기 SHA의 GitHub diff와 해당 시점 source를 기준으로 작성했습니다. 이 환경에서는 브랜치를 checkout하여 build 또는 test를 실행하지 않았습니다.
