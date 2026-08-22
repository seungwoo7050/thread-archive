===== BEGIN FILE: 01-portable-readiness-and-nonblocking-transport.md =====
# Thread 01 — Portable readiness and non-blocking transport

부제: 이벤트 준비 상태와 논블로킹 전송

## 1. Thread 목표

native readiness API의 차이를 공통 의미로 번역하고, move-only connection state와 single-threaded event loop가 입력·출력·종료를 어떻게 진행시키는지 commit 순서대로 복원합니다.

Source에서 확정된 significance:

> This progression builds the server from a portable event vocabulary into an operational single-threaded reactor. The decisive choices are not the platform calls themselves, but the separation of native readiness from server semantics, the move-only per-connection state machine, persistent partial-write progress, and the authoritative server cleanup path. Later reliability work depends directly on these ownership and I/O contracts.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- `epoll`과 `kqueue`의 등록/notification 차이를 어느 경계에서 공통 의미로 바꾸는가?
- 한 fd의 socket lifetime, input fragment, output suffix, close state를 누가 함께 소유하는가?
- readiness 한 번이 여러 line, partial line, partial write, EOF, backpressure를 만들 때 각 상태는 어디에 남는가?
- accepted fd가 event registry와 connection map에 들어가고 빠지는 순서는 무엇인가?
- pending output과 write interest, graceful close completion은 어떤 조건으로 동기화되는가?

### Source와 직접 연결되는 invariant

- accepted descriptor마다 authoritative `Connection` owner가 하나이며 fd는 정확히 한 번 닫혀야 합니다. event registration과 connection map은 분리되어서는 안 됩니다.
- partial send는 실제 성공 byte 수만큼만 전진하고 retry는 unsent suffix의 순서를 보존해야 합니다.
- input framing은 incremental하고 output framing은 CRLF로 정규화되며 malformed/oversized frame이 이후 valid command로 남아서는 안 됩니다.

### Source와 직접 연결되는 engineering difficulty

- `epoll`과 `kqueue`의 서로 다른 등록/notification 모델을 server loop에 노출하지 않고 통합하는 문제.
- 한 readiness가 여러 line, partial line/write, interruption, backpressure, EOF, hard error를 만들 수 있는 byte-stream 모델링.
- callback 또는 interest update가 현재 처리 중인 connection을 제거할 수 있는 lifetime 문제의 기반.

## 3. 완료 기준

- 공통 `Event`에서 `Server::pollOnce()`와 client read/write 처리까지 caller/callee 흐름을 해당 SHA 코드로 설명할 수 있습니다.
- `Connection`의 fd·buffer·offset·close state 소유권과 move 이후 source/destination 상태를 증거 코드로 제시할 수 있습니다.
- `EINTR`, `EAGAIN`/`EWOULDBLOCK`, EOF, hard error, hangup을 read/write/wait별로 구분할 수 있습니다.
- queue된 byte와 write interest가 어긋나지 않는 조건 및 disconnect의 erase-before-callback 순서를 설명할 수 있습니다.
- final HEAD가 아니라 각 commit SHA의 코드와 필요한 직전 관련 SHA만으로 변화 과정을 기록했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `8e8a3db87950` | `feat(event): 이벤트 준비 상태 계약 정의` | A | ARCH, EVENT_IO | Defines the common readiness vocabulary and backend contract. |
| 2 | `d3f74e2857da` | `feat(event): kqueue 관심 상태 등록 구현` | B | EVENT_IO | Maps logical interest changes onto kqueue filters. |
| 3 | `769cd3094f71` | `feat(event): kqueue 준비 이벤트 변환 구현` | B | EVENT_IO | Converts kqueue notifications into common events. |
| 4 | `2284a0e0d8bb` | `feat(event): epoll 관심 상태 등록 구현` | B | EVENT_IO | Maps logical interest changes onto epoll registrations. |
| 5 | `f1320f357bca` | `feat(event): epoll 준비 이벤트 변환 구현` | B | EVENT_IO | Converts epoll notifications and socket errors into common events. |
| 6 | `8864f253ac15` | `feat(connection): 스트림 연결 상태 계약 정의` | A | ARCH, EVENT_IO, LIFECYCLE | Defines the move-only Connection and its I/O outcome model. |
| 7 | `d71a2549c0eb` | `feat(connection): 소켓 소유권과 이동 수명 구현` | A | LIFECYCLE, RISK | Implements exactly-once descriptor ownership and move transfer. |
| 8 | `9b601b69de4f` | `feat(connection): IRC 입력 프레임 추출 구현` | A | EVENT_IO, IRC_PROTOCOL, RISK | Establishes incremental IRC framing and the line-size boundary. |
| 9 | `b00589d4b1b1` | `feat(connection): 논블로킹 수신 상태 처리` | A | EVENT_IO, RISK | Implements the non-blocking receive state machine. |
| 10 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | CORE, EVENT_IO, LIFECYCLE | Implements persistent partial-write progress and graceful output draining. |
| 11 | `e6492e27cc30` | `feat(server): 이벤트 서버 공개 계약 정의` | A | ARCH, EVENT_IO, LIFECYCLE | Defines Server ownership and the transport callback boundary. |
| 12 | `4ad1227e5119` | `feat(server): 논블로킹 연결 수락과 등록 구현` | A | EVENT_IO, LIFECYCLE | Integrates accepted descriptors into connection and event ownership. |
| 13 | `378d5304828d` | `feat(server): 준비 이벤트 루프 구동` | S | CORE, ARCH, EVENT_IO | Makes the portable event loop operational. |
| 14 | `625ffc924de8` | `feat(server): 송신 큐와 쓰기 관심 상태 연결` | A | EVENT_IO, LIFECYCLE, INTEGRATION | Couples pending output to write readiness and close completion. |
| 15 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | LIFECYCLE, RISK | Centralizes transport removal and disconnect notification. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `8e8a3db87950` — `feat(event): 이벤트 준비 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | ARCH, EVENT_IO |
| 원문 확정 역할 | Defines the common readiness vocabulary and backend contract. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Defines the common readiness vocabulary and backend contract.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | portable readiness 의미와 backend 수명 계약이 아직 존재하지 않았습니다. |
| 주요 문제와 설계 판단 | `EventInterest`를 Read/Write 조합형 비트마스크로 만들고, `Event`가 readiness·error·hangup·socket error를 공통 형태로 보유하도록 했습니다. `EventManager`는 add/update/remove/wait와 `createDefault()`를 제공하는 추상 경계가 되었습니다. |
| 변경 파일·symbol | `include/EventManager.hpp — EventInterest, Event, EventManager` |
| state / ownership 변화 | native flag는 공개 계약 밖에 남고 backend 인스턴스는 `std::unique_ptr<EventManager>`가 단독 소유합니다. |
| failure 또는 boundary | 이 SHA는 실패를 공통 예외형으로 강제하지는 않으며 실제 kernel 오류 처리는 backend 구현에 맡깁니다. |
| 보장 / 비보장 | 보장: 상위 `Server`가 epoll/kqueue 상수 없이 읽기·쓰기 관심과 준비 결과를 표현할 수 있습니다.<br>비보장: 실제 등록·대기·오류 번역은 아직 구현되지 않았습니다. |
| 다음 관련 변화 | d3f74e2857da와 2284a0e0d8bb가 등록을, 769cd3094f71과 f1320f357bca가 notification 번역을 채웁니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `8e8a3db87950` | `include/EventManager.hpp` | `EventInterest, operator\|, hasInterest` | read와 write를 동시에 표현하는 공통 비트 연산 |
| `8e8a3db87950` | `include/EventManager.hpp` | `EventManager::createDefault` | backend lifetime을 unique_ptr로 상위 계층에 이전 |

- 조사 방법: GitHub에서 `8e8a3db87950`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d3f74e2857da` — `feat(event): kqueue 관심 상태 등록 구현`. 원문 역할은 “Maps logical interest changes onto kqueue filters.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d3f74e2857da` — `feat(event): kqueue 관심 상태 등록 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | EVENT_IO |
| 원문 확정 역할 | Maps logical interest changes onto kqueue filters. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Maps logical interest changes onto kqueue filters.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | 논리적 Read/Write를 `EVFILT_READ`/`EVFILT_WRITE`의 독립 filter로 비교·적용하고, kernel 변경이 성공한 뒤 shadow map을 갱신했습니다. |
| 확인한 변경 파일·symbol | `src/KqueueEventManager.cpp — addFd, updateFd, removeFd, applyInterestChange, applyFilterChange` |
| 핵심 state / branch | fd별 설치 관심은 backend shadow map이 보유하며 kernel과 local state가 같은 순서로 전진합니다. |
| failure handling | 등록 변경 실패는 `std::system_error`로 전달합니다. remove 중 이미 사라진 대상의 `ENOENT`/`EBADF`는 정리의 멱등성을 위해 허용합니다. |
| 보장과 다음 연결 | read와 write filter의 추가·삭제가 이전/새 관심 차이에 맞게 적용됩니다. 769cd3094f71가 같은 backend의 wait 결과를 번역합니다. |
| 이 시점의 한계 | 준비 이벤트의 공통 `Event` 변환은 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d3f74e2857da` | `src/KqueueEventManager.cpp` | `applyInterestChange` | 이전/새 관심을 비교해 filter별 변경을 생성 |
| `d3f74e2857da` | `src/KqueueEventManager.cpp` | `removeFd` | 허용 errno와 재전파 errno를 분리 |

- 조사 방법: GitHub에서 `d3f74e2857da`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `769cd3094f71` — `feat(event): kqueue 준비 이벤트 변환 구현`. 원문 역할은 “Converts kqueue notifications into common events.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `769cd3094f71` — `feat(event): kqueue 준비 이벤트 변환 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | EVENT_IO |
| 원문 확정 역할 | Converts kqueue notifications into common events. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Converts kqueue notifications into common events.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | 최대 128개의 `kevent`를 받고 filter를 Read/Write로, `EV_ERROR`를 error와 code로, `EV_EOF`를 hangup으로 변환했습니다. |
| 확인한 변경 파일·symbol | `src/KqueueEventManager.cpp — wait, createDefault` |
| 핵심 state / branch | 한 native notification은 fd와 공통 interest/error/hangup 상태를 가진 `Event`로 복사됩니다. |
| failure handling | `kevent`가 `EINTR`로 끝나면 빈 결과로 돌아가 loop가 다시 진행하며 그 외 오류는 예외로 전달합니다. |
| 보장과 다음 연결 | kqueue-specific flag layout이 `Server`에 노출되지 않습니다. 2284a0e0d8bb와 f1320f357bca가 Linux backend에 같은 공통 의미를 구현합니다. |
| 이 시점의 한계 | 동일 fd의 여러 native record 병합 여부와 상위 처리 semantics는 server 단계에서 결정됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `769cd3094f71` | `src/KqueueEventManager.cpp` | `KqueueEventManager::wait` | EVFILT/EV_ERROR/EV_EOF를 공통 Event 필드로 번역 |

- 조사 방법: GitHub에서 `769cd3094f71`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `2284a0e0d8bb` — `feat(event): epoll 관심 상태 등록 구현`. 원문 역할은 “Maps logical interest changes onto epoll registrations.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `2284a0e0d8bb` — `feat(event): epoll 관심 상태 등록 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | EVENT_IO |
| 원문 확정 역할 | Maps logical interest changes onto epoll registrations. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Maps logical interest changes onto epoll registrations.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `nativeEventsFor()`로 Read/Write 관심을 epoll mask로 만들고 `epoll_ctl` ADD/MOD/DEL을 backend 메서드에 배치했습니다. |
| 확인한 변경 파일·symbol | `src/EpollEventManager.cpp — nativeEventsFor, addFd, updateFd, removeFd` |
| 핵심 state / branch | fd 등록 상태는 kernel epoll set이 authoritative하며 공통 interest가 호출 입력입니다. |
| failure handling | `epoll_ctl` 실패는 system error로 전달하고 제거 중 이미 닫히거나 사라진 fd는 멱등 정리 대상으로 취급합니다. |
| 보장과 다음 연결 | Linux에서도 동일한 add/update/remove 계약을 제공합니다. f1320f357bca가 readiness/error/hangup 변환을 완성합니다. |
| 이 시점의 한계 | `epoll_wait` 결과와 `SO_ERROR` 번역은 다음 commit 전에는 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `2284a0e0d8bb` | `src/EpollEventManager.cpp` | `nativeEventsFor` | 공통 Read/Write를 EPOLLIN/EPOLLOUT로 변환 |
| `2284a0e0d8bb` | `src/EpollEventManager.cpp` | `addFd/updateFd/removeFd` | epoll_ctl operation별 등록 수명 |

- 조사 방법: GitHub에서 `2284a0e0d8bb`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `f1320f357bca` — `feat(event): epoll 준비 이벤트 변환 구현`. 원문 역할은 “Converts epoll notifications and socket errors into common events.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `f1320f357bca` — `feat(event): epoll 준비 이벤트 변환 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | EVENT_IO |
| 원문 확정 역할 | Converts epoll notifications and socket errors into common events. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Converts epoll notifications and socket errors into common events.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `epoll_wait` 결과의 IN/OUT/ERR/HUP/RDHUP를 공통 `Event`로 만들고, error 시 `getsockopt(SO_ERROR)`로 socket error를 보충했습니다. |
| 확인한 변경 파일·symbol | `src/EpollEventManager.cpp — wait, socketErrorFor, createDefault` |
| 핵심 state / branch | native event array에서 공통 Event vector로 값이 복사되며 backend lifetime은 factory 반환 unique_ptr가 관리합니다. |
| failure handling | `epoll_wait`의 `EINTR`은 빈 readiness로 처리하고 다른 오류는 예외로 전달합니다. `SO_ERROR` 조회 실패도 오류 정보를 잃지 않도록 처리됩니다. |
| 보장과 다음 연결 | epoll과 kqueue가 상위 loop에 같은 필드 집합을 제공합니다. 8864f253ac15부터 공통 Event가 구동할 connection 상태를 정의합니다. |
| 이 시점의 한계 | 두 OS에서 실제로 동일 suite가 실행되는 보장은 416efc91e580의 CI에서 추가됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f1320f357bca` | `src/EpollEventManager.cpp` | `EpollEventManager::wait` | EPOLL* notification을 공통 Event로 변환 |
| `f1320f357bca` | `src/EpollEventManager.cpp` | `socketErrorFor` | EPOLLERR를 socket-level error code와 연결 |

- 조사 방법: GitHub에서 `f1320f357bca`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `8864f253ac15` — `feat(connection): 스트림 연결 상태 계약 정의`. 원문 역할은 “Defines the move-only Connection and its I/O outcome model.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `8864f253ac15` — `feat(connection): 스트림 연결 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | ARCH, EVENT_IO, LIFECYCLE |
| 원문 확정 역할 | Defines the move-only Connection and its I/O outcome model. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Defines the move-only Connection and its I/O outcome model.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | fd별 byte-stream 진행 상태와 소유권을 한 객체로 묶는 계약이 없었습니다. |
| 주요 문제와 설계 판단 | `Connection`을 copy 금지·move 허용 객체로 정의하고 fd, peer address, read/write buffer, `writeOffset_`, line limit, peer/close 상태와 read/write 결과를 한곳에 모았습니다. |
| 변경 파일·symbol | `include/Connection.hpp — Connection, ReadResult, WriteResult` |
| state / ownership 변화 | 한 `Connection`이 하나의 fd와 그 fd의 미완성 입력·미전송 출력·종료 상태를 함께 소유합니다. |
| failure 또는 boundary | 결과 구조는 would-block, peer close, hard error를 분리해 server가 즉시 제거와 재시도를 구분할 수 있게 합니다. |
| 보장 / 비보장 | 보장: transport state의 authoritative owner와 move-only 경계가 정의됩니다.<br>비보장: destructor와 move의 exactly-once close 구현은 다음 commit 전에는 선언뿐입니다. |
| 다음 관련 변화 | d71a2549c0eb이 fd 이동 수명을, 9b601b69de4f부터 실제 I/O state machine을 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `8864f253ac15` | `include/Connection.hpp` | `Connection copy/move declarations` | copy를 막고 descriptor ownership 이전만 허용 |
| `8864f253ac15` | `include/Connection.hpp` | `readBuffer_, writeBuffer_, writeOffset_` | framing과 partial output 진행 상태를 fd owner에 결합 |

- 조사 방법: GitHub에서 `8864f253ac15`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d71a2549c0eb` — `feat(connection): 소켓 소유권과 이동 수명 구현`. 원문 역할은 “Implements exactly-once descriptor ownership and move transfer.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `d71a2549c0eb` — `feat(connection): 소켓 소유권과 이동 수명 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | LIFECYCLE, RISK |
| 원문 확정 역할 | Implements exactly-once descriptor ownership and move transfer. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Implements exactly-once descriptor ownership and move transfer.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | move-only 선언은 있었지만 fd를 정확히 한 번 닫는 구현이 없었습니다. |
| 주요 문제와 설계 판단 | destructor가 유효 fd를 닫고, move constructor/assignment가 모든 상태를 옮긴 뒤 source fd를 -1로 만들었습니다. move assignment는 destination의 기존 fd를 먼저 닫습니다. |
| 변경 파일·symbol | `src/Connection.cpp — destructor, move constructor, move assignment` |
| state / ownership 변화 | ownership 이전 후 destination만 fd를 보유하고 moved-from 객체는 비소유 상태가 됩니다. |
| failure 또는 boundary | self-assignment를 피하고 기존 destination resource를 먼저 해제해 overwrite 누수를 막습니다. |
| 보장 / 비보장 | 보장: 정상 destruction과 move chain에서 descriptor가 중복 close되거나 유실되지 않습니다.<br>비보장: event registry와 server map 등록 실패까지의 cross-object rollback은 아직 다루지 않습니다. |
| 다음 관련 변화 | 4ad1227e5119가 이 move-only 객체를 server map에 넣고 5dcd882f0763가 등록 rollback을 보강합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d71a2549c0eb` | `src/Connection.cpp` | `Connection::~Connection` | 유효 fd의 최종 close owner |
| `d71a2549c0eb` | `src/Connection.cpp` | `operator=(Connection&&)` | 기존 fd close 후 state 이동, source fd=-1 |

- 조사 방법: GitHub에서 `d71a2549c0eb`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `9b601b69de4f` — `feat(connection): IRC 입력 프레임 추출 구현`. 원문 역할은 “Establishes incremental IRC framing and the line-size boundary.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `9b601b69de4f` — `feat(connection): IRC 입력 프레임 추출 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Establishes incremental IRC framing and the line-size boundary. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Establishes incremental IRC framing and the line-size boundary.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | read buffer는 있었지만 TCP fragment에서 IRC line을 증분 추출하는 규칙이 없었습니다. |
| 주요 문제와 설계 판단 | newline을 기준으로 complete frame을 반복 추출하고 직전 CR을 제거하며 incomplete suffix는 buffer에 남겼습니다. 최대 line 길이를 newline 포함 기준으로 검사했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — extractLines` |
| state / ownership 변화 | read buffer는 아직 line이 되지 않은 suffix만 유지하고 결과 vector가 완성 line의 순서를 보유합니다. |
| failure 또는 boundary | oversized complete frame 또는 delimiter 없이 limit를 넘은 fragment는 close/error 상태를 만들고 해당 bytes를 이후 valid command로 남기지 않습니다. |
| 보장 / 비보장 | 보장: TCP packet 경계와 무관한 incremental IRC framing을 제공합니다.<br>비보장: 실제 recv drain, EOF, EINTR/would-block 처리는 다음 commit에 없습니다. |
| 다음 관련 변화 | b00589d4b1b1가 socket bytes를 이 framing 함수로 공급합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `9b601b69de4f` | `src/Connection.cpp` | `Connection::extractLines` | newline 반복 추출, CR 제거, suffix 보존과 size 경계 |

- 조사 방법: GitHub에서 `9b601b69de4f`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `b00589d4b1b1` — `feat(connection): 논블로킹 수신 상태 처리`. 원문 역할은 “Implements the non-blocking receive state machine.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `b00589d4b1b1` — `feat(connection): 논블로킹 수신 상태 처리`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, RISK |
| 원문 확정 역할 | Implements the non-blocking receive state machine. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Implements the non-blocking receive state machine.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | frame extractor는 있었지만 non-blocking socket을 끝까지 읽는 state machine이 없었습니다. |
| 주요 문제와 설계 판단 | `recv`를 반복해 positive bytes를 buffer에 붙이고 line을 추출하며, 0은 peer close, EINTR은 재시도, EAGAIN/EWOULDBLOCK은 현재 drain 완료, 나머지는 hard error로 분류했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — readAvailable` |
| state / ownership 변화 | 한 read readiness에서 가능한 bytes와 lines를 모두 소비하고 incomplete suffix는 다음 event까지 남습니다. |
| failure 또는 boundary | EOF와 hard error는 close 경로로 전달되고 would-block은 연결 상태를 유지합니다. |
| 보장 / 비보장 | 보장: edge/level-trigger 차이와 무관하게 준비된 입력을 drain하며 retryable 상태를 오류로 오인하지 않습니다.<br>비보장: application callback 중 connection이 제거되는 reentrancy는 server 단계에서 아직 해결되지 않습니다. |
| 다음 관련 변화 | 378d5304828d가 read result를 line callback과 disconnect 판단에 연결합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b00589d4b1b1` | `src/Connection.cpp` | `Connection::readAvailable` | recv 결과별 progress/retry/terminal branch |

- 조사 방법: GitHub에서 `b00589d4b1b1`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`. 원문 역할은 “Implements persistent partial-write progress and graceful output draining.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | CORE, EVENT_IO, LIFECYCLE |
| 원문 확정 역할 | Implements persistent partial-write progress and graceful output draining. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Implements persistent partial-write progress and graceful output draining.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 출력 bytes와 한 번의 send를 연결할 persistent progress state가 없었습니다. |
| failure / boundary | EINTR은 같은 suffix를 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 핵심 결정 | `writeOffset_`로 sent prefix를 나타내고 `flushPending()`이 unsent suffix만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — flushPending, queueRaw, queueLine, pendingBytes, wantsWrite` |
| ownership / lifecycle / state transition | `writeBuffer_[0:writeOffset_]`는 이미 전송된 prefix, 나머지는 순서가 보존된 pending suffix입니다. 완료 시 buffer/offset을 reset하고 큰 consumed prefix만 compact합니다. |
| 이 commit이 보장하는 것 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 아직 보장하지 않는 것 | queue 크기는 무제한이며 `pending + append` overflow와 send가 요청보다 큰 값을 돌려주는 비정상 결과는 아직 방어하지 않습니다. |
| 후속 fix/test 연결 | 625ffc924de8이 write interest에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 출력 bytes와 한 번의 send를 연결할 persistent progress state가 없었습니다. |
| 2. failure conditions / boundary | EINTR은 같은 suffix를 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 3. 선택한 표현과 순서 | `writeOffset_`로 sent prefix를 나타내고 `flushPending()`이 unsent suffix만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 4. authoritative state | `writeBuffer_[0:writeOffset_]`는 이미 전송된 prefix, 나머지는 순서가 보존된 pending suffix입니다. 완료 시 buffer/offset을 reset하고 큰 consumed prefix만 compact합니다. |
| 5. 결과 invariant | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 6. 후속 보강 | 625ffc924de8이 write interest에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a10fe961e2b1` | `src/Connection.cpp` | `Connection::flushPending` | unsent suffix pointer/length와 return-count 기반 offset 전진 |
| `a10fe961e2b1` | `src/Connection.cpp` | `queueLine` | 기존 CR/LF 제거 후 CRLF 하나 append |
| `a10fe961e2b1` | `src/Connection.cpp` | `pendingBytes/wantsWrite` | logical unsent bytes가 server interest 판단의 입력 |

- 조사 방법: GitHub에서 `a10fe961e2b1`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `e6492e27cc30` — `feat(server): 이벤트 서버 공개 계약 정의`. 원문 역할은 “Defines Server ownership and the transport callback boundary.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.11 `e6492e27cc30` — `feat(server): 이벤트 서버 공개 계약 정의`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | ARCH, EVENT_IO, LIFECYCLE |
| 원문 확정 역할 | Defines Server ownership and the transport callback boundary. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Defines Server ownership and the transport callback boundary.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection과 EventManager는 있었지만 listener·connection map·callback을 조율하는 owner가 없었습니다. |
| 주요 문제와 설계 판단 | `Server` 공개 계약에 config, lifecycle, poll, send/disconnect API와 connect/line/disconnect/error callback을 정의했습니다. event manager와 fd→`unique_ptr<Connection>` map을 소유하게 했습니다. |
| 변경 파일·symbol | `include/Server.hpp — Server, Config, callback types` |
| state / ownership 변화 | Server가 listener, backend, 모든 live Connection의 authoritative transport owner가 됩니다. application은 callback을 통해서만 protocol state를 연결합니다. |
| failure 또는 boundary | public API는 start/poll/send/disconnect 실패가 transport cleanup으로 수렴할 수 있는 경계를 제공합니다. |
| 보장 / 비보장 | 보장: event reactor와 protocol coordinator 사이의 소유·호출 경계가 정의됩니다.<br>비보장: accept, dispatch, interest refresh, cleanup은 아직 구현되지 않았습니다. |
| 다음 관련 변화 | 4ad1227e5119부터 listener와 accepted fd를 실제 owner graph에 편입합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e6492e27cc30` | `include/Server.hpp` | `connections_` | fd별 unique_ptr Connection 소유 |
| `e6492e27cc30` | `include/Server.hpp` | `ConnectHandler/LineHandler/DisconnectHandler` | transport와 application의 callback 경계 |

- 조사 방법: GitHub에서 `e6492e27cc30`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `4ad1227e5119` — `feat(server): 논블로킹 연결 수락과 등록 구현`. 원문 역할은 “Integrates accepted descriptors into connection and event ownership.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.12 `4ad1227e5119` — `feat(server): 논블로킹 연결 수락과 등록 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, LIFECYCLE |
| 원문 확정 역할 | Integrates accepted descriptors into connection and event ownership. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Integrates accepted descriptors into connection and event ownership.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Server 계약은 있었지만 accepted fd를 non-blocking connection과 event registry에 넣지 못했습니다. |
| 주요 문제와 설계 판단 | accept를 would-block까지 반복하고 socket을 설정한 뒤 Connection을 만들고 event backend에 Read 관심을 등록하고 map에 이동시킨 다음 connect callback을 호출했습니다. |
| 변경 파일·symbol | `src/Server.cpp — acceptReadyClients, socket configuration helpers` |
| state / ownership 변화 | accept된 raw fd는 Connection 생성 이후 그 객체로, map 삽입 이후 Server로 ownership이 이전됩니다. |
| failure 또는 boundary | accept EINTR은 재시도하고 EAGAIN/EWOULDBLOCK은 drain 종료입니다. 설정/등록/삽입/callback 예외는 catch 경로로 전달되지만 이 버전은 event add와 map 삽입의 완전한 rollback 및 callback self-removal 안전성이 부족합니다. |
| 보장 / 비보장 | 보장: listener readiness 하나에서 accept queue를 drain하고 live fd를 event loop에 등록합니다.<br>비보장: 등록 성공 뒤 raw pointer를 callback 이후 재사용할 수 있어 후속 reentrancy fix 대상입니다. |
| 다음 관련 변화 | 5dcd882f0763가 map-first insertion, add rollback, fd relookup으로 수명 결함을 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `4ad1227e5119` | `src/Server.cpp` | `Server::acceptReadyClients` | accept drain과 fd→Connection→event/map/callback 순서 |

- 조사 방법: GitHub에서 `4ad1227e5119`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `378d5304828d` — `feat(server): 준비 이벤트 루프 구동`. 원문 역할은 “Makes the portable event loop operational.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.13 `378d5304828d` — `feat(server): 준비 이벤트 루프 구동`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | CORE, ARCH, EVENT_IO |
| 원문 확정 역할 | Makes the portable event loop operational. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Makes the portable event loop operational.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | listener와 connection 등록은 가능했지만 공통 readiness를 지속 처리하는 reactor가 없었습니다. |
| failure / boundary | listener error/hangup은 server-level failure이며 client error는 해당 fd disconnect로 수렴합니다. stop request를 확인해 더 이상 dispatch하지 않는 경계가 있습니다. |
| 핵심 결정 | `start()`, `run()`, `pollOnce()`를 연결해 listener event는 accept로, client event는 read/write/error/hangup 처리로 분배했습니다. |
| 변경 파일·symbol | `src/Server.cpp — start, run, pollOnce, handleClientEvent` |
| ownership / lifecycle / state transition | single-threaded loop가 backend wait 결과를 순서대로 처리하고 callback을 통해 complete lines를 application에 넘깁니다. |
| 이 commit이 보장하는 것 | portable backend 위에서 동작하는 실제 reactor와 connection-scoped failure isolation을 제공합니다. |
| 아직 보장하지 않는 것 | callback이 현재 connection을 삭제할 때 보유 pointer를 재사용하는 위험과 interest update failure rollback은 남아 있습니다. |
| 후속 fix/test 연결 | 625ffc924de8이 출력 readiness를, 7a6bc7e1276a가 authoritative cleanup을 추가합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | listener와 connection 등록은 가능했지만 공통 readiness를 지속 처리하는 reactor가 없었습니다. |
| 2. failure conditions / boundary | listener error/hangup은 server-level failure이며 client error는 해당 fd disconnect로 수렴합니다. stop request를 확인해 더 이상 dispatch하지 않는 경계가 있습니다. |
| 3. 선택한 표현과 순서 | `start()`, `run()`, `pollOnce()`를 연결해 listener event는 accept로, client event는 read/write/error/hangup 처리로 분배했습니다. |
| 4. authoritative state | single-threaded loop가 backend wait 결과를 순서대로 처리하고 callback을 통해 complete lines를 application에 넘깁니다. |
| 5. 결과 invariant | portable backend 위에서 동작하는 실제 reactor와 connection-scoped failure isolation을 제공합니다. |
| 6. 후속 보강 | 625ffc924de8이 출력 readiness를, 7a6bc7e1276a가 authoritative cleanup을 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `378d5304828d` | `src/Server.cpp` | `Server::pollOnce` | 공통 Event에서 listener/client 분기 |
| `378d5304828d` | `src/Server.cpp` | `Server::handleClientEvent` | read lines, write flush, error/hangup dispatch |

- 조사 방법: GitHub에서 `378d5304828d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `625ffc924de8` — `feat(server): 송신 큐와 쓰기 관심 상태 연결`. 원문 역할은 “Couples pending output to write readiness and close completion.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.14 `625ffc924de8` — `feat(server): 송신 큐와 쓰기 관심 상태 연결`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, LIFECYCLE, INTEGRATION |
| 원문 확정 역할 | Couples pending output to write readiness and close completion. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Couples pending output to write readiness and close completion.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection은 pending output을 보유했지만 server backend의 Write 관심과 자동 동기화되지 않았습니다. |
| 주요 문제와 설계 판단 | `sendTo()`/`queueRawTo()`가 queue 후 `refreshInterest()`를 호출하고, pending bytes가 있을 때만 Write 관심을 추가했습니다. closeRequested 상태는 pending이 비면 disconnect하도록 했습니다. |
| 변경 파일·symbol | `src/Server.cpp — sendTo, queueRawTo, refreshInterest, writable handling` |
| state / ownership 변화 | Connection의 `wantsWrite()`가 kernel write interest의 단일 입력이고 drain 완료가 graceful close completion 조건입니다. |
| failure 또는 boundary | queue나 flush 오류는 close/disconnect로 수렴합니다. 이 버전의 update failure 처리는 예외 후 stale ownership을 남길 수 있어 후속 fix가 필요합니다. |
| 보장 / 비보장 | 보장: busy-loop 없이 pending output이 있을 때만 writable notification을 요청하고, queued final response를 보낸 뒤 닫을 수 있습니다.<br>비보장: interest update가 callback/cleanup을 유발한 뒤 reference를 다시 쓰는 문제는 5dcd882f0763 이전에 남습니다. |
| 다음 관련 변화 | 7a6bc7e1276a가 disconnect 종착점을 만들고 5dcd882f0763가 refresh를 fd 기반으로 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `625ffc924de8` | `src/Server.cpp` | `Server::refreshInterest` | pending/close state를 Read/Write interest와 연결 |
| `625ffc924de8` | `src/Server.cpp` | `Server::sendTo/queueRawTo` | queue 결과와 backend 관심 갱신 연결 |

- 조사 방법: GitHub에서 `625ffc924de8`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`. 원문 역할은 “Centralizes transport removal and disconnect notification.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.15 `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | LIFECYCLE, RISK |
| 원문 확정 역할 | Centralizes transport removal and disconnect notification. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Centralizes transport removal and disconnect notification.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 여러 event/error/stop 경로에서 connection 제거 순서가 하나의 함수로 고정되지 않았습니다. |
| 주요 문제와 설계 판단 | `disconnect()`가 backend remove를 시도하고 `unique_ptr`를 map 밖 local로 옮겨 entry를 먼저 erase한 뒤 disconnect callback을 호출했습니다. bulk close는 fd snapshot을 사용했습니다. |
| 변경 파일·symbol | `src/Server.cpp — disconnect, closeAllConnections, handleClientEvent` |
| state / ownership 변화 | callback 시점에는 server map에서 fd가 이미 사라졌지만 local unique_ptr가 callback 동안 객체 lifetime을 유지하고 callback 후 destructor가 fd를 닫습니다. |
| failure 또는 boundary | backend remove와 callback exception은 report되더라도 object destruction을 막지 않습니다. hangup은 pending output이 없을 때 terminal입니다. |
| 보장 / 비보장 | 보장: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 iterator가 깨지지 않습니다.<br>비보장: callback 전에 잡은 raw pointer를 callback 뒤 다시 사용하는 call site와 event add/update rollback은 아직 안전하지 않습니다. |
| 다음 관련 변화 | 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 rollback 결함을 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::disconnect` | event remove → map ownership move/erase → callback → destruction 순서 |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::closeAllConnections` | fd snapshot으로 반복 erase 안전성 |

- 조사 방법: GitHub에서 `7a6bc7e1276a`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| 공통 readiness 의미 | `8e8a3db87950` | d3f74e2857da, 769cd3094f71, 2284a0e0d8bb, f1320f357bca | 각 backend의 native registration/notification을 같은 `Event` 의미로 번역합니다. | 별도 runtime test는 이 Thread에 없고 416efc91e580의 두 OS CI로 이어집니다. |
| fd 단일 소유권 | `8864f253ac15` | d71a2549c0eb, 4ad1227e5119, 7a6bc7e1276a | Connection move와 map/local unique_ptr 수명으로 close owner를 하나로 유지합니다. | 5dcd882f0763/928594ec160c가 등록 rollback과 callback lifetime을 보강합니다. |
| incremental input framing | `9b601b69de4f` | b00589d4b1b1 | complete line만 반환하고 incomplete suffix를 보존하며 oversize를 terminal로 만듭니다. | 6b4a7738a285의 fragmented TCP smoke가 통합 경로를 확인합니다. |
| persistent partial output | `a10fe961e2b1` | 625ffc924de8 | writeOffset와 Write interest를 결합해 unsent suffix를 event 사이에 보존합니다. | 881e59734a9a/f34ab135c546가 overflow와 모든 send outcome을 수정·검증합니다. |
| authoritative cleanup | `e6492e27cc30` | 378d5304828d, 7a6bc7e1276a | event remove → map erase → callback → descriptor destruction의 종착점을 만듭니다. | 5dcd882f0763/928594ec160c가 reentrant call site와 rollback을 고정합니다. |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| 한 send가 전부 완료된다는 가정 | short send/backpressure에서 suffix 유실·중복 | a10fe961e2b1 → 881e59734a9a | persistent offset과 overflow-safe admission/return guard | f34ab135c546 |
| callback 뒤 기존 pointer가 유효하다는 가정 | self-disconnect 뒤 stale Connection 접근 | 7a6bc7e1276a → 5dcd882f0763 | erase-before-callback과 fd relookup | 928594ec160c |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| native readiness registration/translation | 없음 | `EventManager` backend | 8e8a3db87950 → f1320f357bca | path/symbol은 위 commit 증거표에 연결했습니다. |
| socket·input·output·close progress | raw fd 중심 | move-only `Connection` | 8864f253ac15 → a10fe961e2b1 | path/symbol은 위 commit 증거표에 연결했습니다. |
| listener와 connection orchestration | 구성 요소 분리 | `Server`/`pollOnce()` | e6492e27cc30 → 378d5304828d | path/symbol은 위 commit 증거표에 연결했습니다. |
| write readiness 정책 | Connection queue만 존재 | `Server::refreshInterest()` | 625ffc924de8 | path/symbol은 위 commit 증거표에 연결했습니다. |
| descriptor removal/notification | 분산 가능성 | `Server::disconnect()` | 7a6bc7e1276a | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: portable readiness 의미와 backend 수명 계약이 아직 존재하지 않았습니다.
- 마지막 commit `7a6bc7e1276a` 시점의 상태: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 iterator가 깨지지 않습니다.
- Thread 안에서 강화된 핵심 invariant: 공통 readiness 의미, fd 단일 소유권, incremental input framing, persistent partial output, authoritative cleanup.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: callback 전에 잡은 raw pointer를 callback 뒤 다시 사용하는 call site와 event add/update rollback은 아직 안전하지 않습니다. 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 rollback 결함을 수정·검증합니다.
- 최종 설명: `8e8a3db87950`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `7a6bc7e1276a`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| backend wait | 769cd3094f71 / f1320f357bca | `Kqueue/EpollEventManager::wait` | 공통 `Event` vector | EINTR은 빈 결과, 다른 wait 오류는 예외 |
| event dispatch | 378d5304828d | `Server::pollOnce` | listener는 accept, client는 handleClientEvent | event error는 connection-scoped disconnect |
| read path | b00589d4b1b1 / 378d5304828d | `Server::handleClientEvent` | Connection::readAvailable → line callback | EOF/hard error/close request는 disconnect |
| write path | a10fe961e2b1 / 625ffc924de8 | `Server writable branch` | Connection::flushPending → offset/interest 갱신 | would-block은 suffix 보존, hard error는 disconnect |
| cleanup | 7a6bc7e1276a | `Server::disconnect` | backend remove → map erase/local owner → callback | 예외를 report해도 local owner destructor가 fd close |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 01-portable-readiness-and-nonblocking-transport.md =====

===== BEGIN FILE: 02-protocol-boundary-identity-and-registration.md =====
# Thread 02 — Protocol boundary, identity, and registration

부제: 프로토콜 경계, 식별자와 등록

## 1. Thread 목표

wire line을 구조화 message로 바꾸는 문법 경계, descriptor 기반 client state와 nickname index, `IrcApplication` 책임 분리, PASS/NICK/USER 등록 gate가 하나의 실행 경로로 결합되는 과정을 복원합니다.

Source에서 확정된 significance:

> The sequence separates syntax, connection identity, and registration authorization rather than mixing them into the event loop. The application boundary and registration gate are the durable decisions; the individual registration commands are normal implementations within those decisions. The first smoke suite then verifies that framing, parsing, identity indexes, replies, and lifecycle transitions compose correctly.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- transport framing과 IRC grammar parsing은 어디에서 분리되는가?
- display nickname과 canonical lookup key는 왜 별도로 유지되는가?
- `Server` callback에서 protocol state를 생성·조회·삭제하는 authoritative application boundary는 어디인가?
- syntax-valid command가 registration state 때문에 거부되는 지점은 어디인가?
- PASS/NICK/USER prerequisite와 001–003 welcome의 단일 transition은 어떻게 보장되는가?
- 첫 real-TCP smoke가 어떤 계층 연결을 증명하고 무엇은 아직 정확히 고정하지 않는가?

### Source와 직접 연결되는 invariant

- IRC registration은 password, nickname, USER prerequisites가 모두 성립한 뒤에만 완료됩니다.
- nickname은 registry canonical lookup rule 아래 unique해야 하며 display spelling과 reverse key가 일치해야 합니다.
- malformed 또는 oversized frame은 이전 structured state를 남기거나 이후 valid command로 처리되어서는 안 됩니다.

### Source와 직접 연결되는 engineering difficulty

- byte-stream framing과 IRC syntax/authorization을 서로 다른 책임으로 유지하는 문제.
- callback lifecycle와 descriptor-keyed protocol state를 연결하면서 phantom state나 stale index를 만들지 않는 문제.

## 3. 완료 기준

- 한 TCP line이 parser, normalized command, registration gate, handler, reply queue까지 이동하는 흐름을 해당 SHA 코드로 설명할 수 있습니다.
- descriptor map과 nickname reverse index의 consistency 조건 및 collision-before-mutation 순서를 설명할 수 있습니다.
- transport connection과 IRC registration을 같은 상태로 취급하지 않고 각 lifecycle 필드를 구분할 수 있습니다.
- registration prerequisites와 welcome reply ordering을 state transition으로 정리했습니다.
- smoke test의 production path, test technique, 증명/비증명 범위를 구분했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a22bf6ddbd75` | `feat(parser): IRC 메시지 값과 직렬화 정의` | B | IRC_PROTOCOL | Introduces the structured IRC message representation. |
| 2 | `c31a32b6cb24` | `feat(parser): IRC 한 줄 구문 해석 구현` | A | IRC_PROTOCOL, RISK | Defines the line grammar and parameter parsing boundary. |
| 3 | `aeb1e9b709b9` | `feat(reply): IRC 서버 응답 생성` | B | IRC_PROTOCOL | Adds canonical server messages, numerics, and hostmasks. |
| 4 | `991b76b8d793` | `feat(client): 연결별 등록 상태 저장` | B | IDENTITY | Stores descriptor-keyed registration state. |
| 5 | `b47135c51cfc` | `feat(client): 닉네임 색인 관리` | A | IDENTITY, RISK | Establishes the case-insensitive nickname index invariant. |
| 6 | `0b5ae6aef328` | `feat(app): IRC 동작 조율 계약 정의` | S | CORE, ARCH, INTEGRATION | Defines IrcApplication as the protocol/domain coordinator above Server. |
| 7 | `2ed9331124bb` | `feat(app): 연결 수명 콜백 조율` | A | LIFECYCLE, IRC_PROTOCOL, INTEGRATION | Connects transport lifecycle callbacks to client state, parsing, and cleanup. |
| 8 | `035e1137e0dd` | `feat(app): 등록 전 명령 분배 구현` | A | IRC_PROTOCOL, IDENTITY, RISK | Centralizes the pre-registration command gate. |
| 9 | `582317254e24` | `feat(registration): PASS 인증 상태 처리` | B | IDENTITY, IRC_PROTOCOL | Adds password authorization state. |
| 10 | `80a639321bad` | `feat(registration): 닉네임 검증과 색인 갱신` | B | IDENTITY, IRC_PROTOCOL | Adds nickname validation and collision handling. |
| 11 | `d9e420b570a0` | `feat(registration): USER 정보와 환영 응답 연결` | B | IDENTITY, IRC_PROTOCOL | Completes PASS/NICK/USER registration and welcome replies. |
| 12 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | VERIFICATION, INTEGRATION, RISK | Proves the integrated registration and command path over real TCP. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `a22bf6ddbd75` — `feat(parser): IRC 메시지 값과 직렬화 정의`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IRC_PROTOCOL |
| 원문 확정 역할 | Introduces the structured IRC message representation. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Introduces the structured IRC message representation.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `IrcMessage`에 optional prefix, uppercase command, ordered params, raw frame을 분리하고 safe indexed access와 `toLine()`을 구현했습니다. |
| 확인한 변경 파일·symbol | `include/IrcMessage.hpp — IrcMessage`<br>`src/IrcMessage.cpp — normalization, param access, toLine` |
| 핵심 state / branch | parser/handler는 command casing을 따로 다루지 않고 message 값 하나를 공유합니다. |
| failure handling | out-of-range parameter access는 unchecked indexing 대신 fallback을 돌려 handler의 UB를 막습니다. |
| 보장과 다음 연결 | 마지막 parameter의 공백/colon/empty 조건에 trailing marker를 적용하고 CRLF frame을 만듭니다. c31a32b6cb24가 line parser를, aeb1e9b709b9가 server reply helper를 추가합니다. |
| 이 시점의 한계 | wire text를 구조화하는 parse grammar는 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a22bf6ddbd75` | `src/IrcMessage.cpp` | `IrcMessage::toLine` | prefix/params/trailing 규칙과 CRLF 직렬화 |
| `a22bf6ddbd75` | `src/IrcMessage.cpp` | `command normalization/param` | uppercase dispatch key와 safe fallback |

- 조사 방법: GitHub에서 `a22bf6ddbd75`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `c31a32b6cb24` — `feat(parser): IRC 한 줄 구문 해석 구현`. 원문 역할은 “Defines the line grammar and parameter parsing boundary.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `c31a32b6cb24` — `feat(parser): IRC 한 줄 구문 해석 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Defines the line grammar and parameter parsing boundary. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Defines the line grammar and parameter parsing boundary.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 값 객체는 있었지만 raw IRC line의 prefix/command/parameter 문법을 일관되게 해석하지 못했습니다. |
| 주요 문제와 설계 판단 | frame terminator를 제거하고 output을 초기화한 뒤 optional prefix, 필수 command, middle/trailing parameters를 순서대로 파싱했습니다. command를 uppercase로 정규화하고 raw를 보존했습니다. |
| 변경 파일·symbol | `src/IrcMessage.cpp — parseLine` |
| state / ownership 변화 | 성공 시 output이 한 frame의 완전한 structured state가 되고 실패 시 이전 state가 남지 않습니다. |
| failure 또는 boundary | empty line, prefix 뒤 command 누락, 510-byte 초과 등 malformed boundary를 false로 반환합니다. |
| 보장 / 비보장 | 보장: trailing colon 이후 공백 포함 text를 한 parameter로 보존하고 packet framing과 grammar parsing을 분리합니다.<br>비보장: authorization·registration 상태에 따른 command 허용은 application 단계에 없습니다. |
| 다음 관련 변화 | 035e1137e0dd가 parse 성공 후 registration gate를 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c31a32b6cb24` | `src/IrcMessage.cpp` | `parseLine` | output reset, prefix/command/middle/trailing parameter 순서와 length 검사 |

- 조사 방법: GitHub에서 `c31a32b6cb24`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `aeb1e9b709b9` — `feat(reply): IRC 서버 응답 생성`. 원문 역할은 “Adds canonical server messages, numerics, and hostmasks.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `aeb1e9b709b9` — `feat(reply): IRC 서버 응답 생성`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IRC_PROTOCOL |
| 원문 확정 역할 | Adds canonical server messages, numerics, and hostmasks. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds canonical server messages, numerics, and hostmasks.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `Replies`에 numeric code formatting, generic message, ERROR, hostmask helper를 두고 target이 없을 때 `*`를 사용하도록 했습니다. |
| 확인한 변경 파일·symbol | `include/Replies.hpp`<br>`src/Replies.cpp — numeric, formatMessage, error, hostmask` |
| 핵심 state / branch | server response formatting의 authoritative 위치가 reply helper로 이동합니다. |
| failure handling | 마지막 parameter만 trailing 처리해 중간 parameter의 공백이 wire grammar를 깨뜨리지 않도록 호출자가 구조화 값을 넘기게 합니다. |
| 보장과 다음 연결 | numeric width와 prefix/target/CRLF 형태가 handler마다 달라지지 않습니다. IrcApplication handlers가 이 helper를 통해 Connection output queue로 응답합니다. |
| 이 시점의 한계 | 실제 send 실패와 lifecycle invalidation은 이 helper가 다루지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `aeb1e9b709b9` | `src/Replies.cpp` | `Replies::numeric/formatMessage` | server prefix, code, target, trailing serialization 중앙화 |

- 조사 방법: GitHub에서 `aeb1e9b709b9`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `991b76b8d793` — `feat(client): 연결별 등록 상태 저장`. 원문 역할은 “Stores descriptor-keyed registration state.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `991b76b8d793` — `feat(client): 연결별 등록 상태 저장`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IDENTITY |
| 원문 확정 역할 | Stores descriptor-keyed registration state. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Stores descriptor-keyed registration state.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `ClientState`와 `ClientRegistry`를 만들고 fd-keyed map, create-on-demand `state()`, non-creating `find()`, contains/fds/erase를 제공했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — ClientState, ClientRegistry`<br>`src/ClientRegistry.cpp` |
| 핵심 state / branch | socket lifetime은 Server/Connection이, IRC identity/registration lifetime은 ClientRegistry가 같은 fd를 key로 각각 관리합니다. |
| failure handling | `find()`는 phantom state를 만들지 않고 `erase()`는 없는 fd에도 안전합니다. |
| 보장과 다음 연결 | transport connected와 IRC registered를 다른 상태로 표현할 기반이 생깁니다. b47135c51cfc가 canonical nickname index와 erase consistency를 추가합니다. |
| 이 시점의 한계 | nickname uniqueness를 위한 reverse index는 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `991b76b8d793` | `src/ClientRegistry.cpp` | `state/find/erase/fds` | 생성 조회와 비생성 조회, snapshot, 멱등 erase 분리 |

- 조사 방법: GitHub에서 `991b76b8d793`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `b47135c51cfc` — `feat(client): 닉네임 색인 관리`. 원문 역할은 “Establishes the case-insensitive nickname index invariant.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `b47135c51cfc` — `feat(client): 닉네임 색인 관리`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | IDENTITY, RISK |
| 원문 확정 역할 | Establishes the case-insensitive nickname index invariant. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Establishes the case-insensitive nickname index invariant.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | fd→client state만 있어 nickname lookup이 linear하거나 대소문자 collision을 놓칠 수 있었습니다. |
| 주요 문제와 설계 판단 | canonical nickname→fd reverse map을 추가하고 `setNickname()`이 old canonical key를 먼저 제거한 뒤 new key를 삽입하도록 했습니다. client erase는 reverse key도 함께 제거합니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp/.cpp — canonical nickname index, setNickname, findByNick, erase`<br>`include/Channel.hpp — canonicalNick helper` |
| state / ownership 변화 | display spelling은 ClientState에, lookup key는 canonical map에 있으며 두 표현이 한 mutation API에서 함께 갱신됩니다. |
| failure 또는 boundary | collision 여부를 mutation 전에 확인할 수 있고 erase 후 stale nickname이 다른 fd를 가리키지 않습니다. |
| 보장 / 비보장 | 보장: case-insensitive uniqueness와 fd↔nickname index consistency를 유지할 수 있습니다.<br>비보장: 실제 NICK validation/collision numeric은 handler에 아직 없습니다. |
| 다음 관련 변화 | 80a639321bad가 validation과 collision-before-mutation을 명령 경로에 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b47135c51cfc` | `src/ClientRegistry.cpp` | `ClientRegistry::setNickname` | old reverse key 제거 후 state/new key 갱신 |
| `b47135c51cfc` | `src/ClientRegistry.cpp` | `findByNick/erase` | canonical lookup과 lifecycle 정리 |

- 조사 방법: GitHub에서 `b47135c51cfc`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `0b5ae6aef328` — `feat(app): IRC 동작 조율 계약 정의`. 원문 역할은 “Defines IrcApplication as the protocol/domain coordinator above Server.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `0b5ae6aef328` — `feat(app): IRC 동작 조율 계약 정의`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | CORE, ARCH, INTEGRATION |
| 원문 확정 역할 | Defines IrcApplication as the protocol/domain coordinator above Server. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Defines IrcApplication as the protocol/domain coordinator above Server.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server callback과 parser/registry/command handler를 묶는 protocol owner가 없었습니다. |
| failure / boundary | callback boundary가 transport removal을 protocol cleanup으로 전달할 위치를 고정합니다. |
| 핵심 결정 | `IrcApplication`이 Server reference, password/runtime/server name, ClientRegistry와 command handlers를 소유하는 coordinator 계약을 정의했습니다. |
| 변경 파일·symbol | `src/IrcApplication.hpp — IrcApplication public callbacks and private handlers` |
| ownership / lifecycle / state transition | Server는 transport만 소유하고 application은 IRC/domain state와 dispatch를 소유합니다. |
| 이 commit이 보장하는 것 | event loop에 parser/authorization/channel state를 직접 섞지 않는 architecture 경계가 생깁니다. |
| 아직 보장하지 않는 것 | callback 구현과 registration gate는 아직 없습니다. |
| 후속 fix/test 연결 | 2ed9331124bb와 035e1137e0dd가 lifecycle와 dispatch를 구현합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server callback과 parser/registry/command handler를 묶는 protocol owner가 없었습니다. |
| 2. failure conditions / boundary | callback boundary가 transport removal을 protocol cleanup으로 전달할 위치를 고정합니다. |
| 3. 선택한 표현과 순서 | `IrcApplication`이 Server reference, password/runtime/server name, ClientRegistry와 command handlers를 소유하는 coordinator 계약을 정의했습니다. |
| 4. authoritative state | Server는 transport만 소유하고 application은 IRC/domain state와 dispatch를 소유합니다. |
| 5. 결과 invariant | event loop에 parser/authorization/channel state를 직접 섞지 않는 architecture 경계가 생깁니다. |
| 6. 후속 보강 | 2ed9331124bb와 035e1137e0dd가 lifecycle와 dispatch를 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0b5ae6aef328` | `src/IrcApplication.hpp` | `onConnect/onLine/onDisconnect/onTick` | transport callback에서 protocol coordinator로 진입하는 공개 경계 |
| `0b5ae6aef328` | `src/IrcApplication.hpp` | `_clients and handler declarations` | protocol state/dispatch ownership |

- 조사 방법: GitHub에서 `0b5ae6aef328`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `2ed9331124bb` — `feat(app): 연결 수명 콜백 조율`. 원문 역할은 “Connects transport lifecycle callbacks to client state, parsing, and cleanup.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `2ed9331124bb` — `feat(app): 연결 수명 콜백 조율`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | LIFECYCLE, IRC_PROTOCOL, INTEGRATION |
| 원문 확정 역할 | Connects transport lifecycle callbacks to client state, parsing, and cleanup. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Connects transport lifecycle callbacks to client state, parsing, and cleanup.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | application 계약은 있었지만 Connection lifecycle와 ClientRegistry가 연결되지 않았습니다. |
| 주요 문제와 설계 판단 | onConnect가 ClientState를 만들고 host/fd/time을 기록하며, onLine이 필요 시 state를 복구하고 parse 후 dispatch하며, onDisconnect가 application state를 정리하도록 했습니다. |
| 변경 파일·symbol | `src/IrcApplication.cpp — onConnect, onLine, onDisconnect` |
| state / ownership 변화 | transport callback의 fd가 protocol registry key로 이어지고 disconnect callback이 state lifetime의 종착점입니다. |
| failure 또는 boundary | parse 실패는 command handler로 넘어가지 않으며 없는 state는 연결 시점 정보로 생성됩니다. |
| 보장 / 비보장 | 보장: connection lifecycle와 parser/registry cleanup이 한 coordinator를 통과합니다.<br>비보장: 응답 send가 동기 disconnect를 일으킨 뒤 borrowed state를 재사용하는 문제는 후속 08 thread에 남습니다. |
| 다음 관련 변화 | 035e1137e0dd가 parsed command에 registration gate를 적용합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `2ed9331124bb` | `src/IrcApplication.cpp` | `onConnect/onLine/onDisconnect` | fd별 state 생성 → parse/dispatch → cleanup lifecycle |

- 조사 방법: GitHub에서 `2ed9331124bb`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `035e1137e0dd` — `feat(app): 등록 전 명령 분배 구현`. 원문 역할은 “Centralizes the pre-registration command gate.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `035e1137e0dd` — `feat(app): 등록 전 명령 분배 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | IRC_PROTOCOL, IDENTITY, RISK |
| 원문 확정 역할 | Centralizes the pre-registration command gate. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Centralizes the pre-registration command gate.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | syntax-valid command가 registration 전후 동일하게 dispatch될 수 있었습니다. |
| 주요 문제와 설계 판단 | PASS/NICK/USER/PING/QUIT는 등록 전 허용하고, 그 외 command는 unregistered면 451, registered인데 unsupported면 421로 분리하는 central gate를 구현했습니다. |
| 변경 파일·symbol | `src/IrcApplication.cpp — handleMessage` |
| state / ownership 변화 | ClientState.registered가 syntax parsing 뒤 authorization/dispatch 상태를 결정합니다. |
| failure 또는 boundary | 등록 전 금지 명령은 handler side effect 없이 numeric으로 끝납니다. |
| 보장 / 비보장 | 보장: 각 handler가 registration check를 중복하지 않고 단일 정책을 적용합니다.<br>비보장: PASS/NICK/USER prerequisite state와 welcome transition 구현은 다음 commits에 있습니다. |
| 다음 관련 변화 | 582317254e24, 80a639321bad, d9e420b570a0가 gate를 통과하는 등록 상태를 채웁니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `035e1137e0dd` | `src/IrcApplication.cpp` | `IrcApplication::handleMessage` | pre-registration allowlist와 451/421 분기 |

- 조사 방법: GitHub에서 `035e1137e0dd`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `582317254e24` — `feat(registration): PASS 인증 상태 처리`. 원문 역할은 “Adds password authorization state.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `582317254e24` — `feat(registration): PASS 인증 상태 처리`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IDENTITY, IRC_PROTOCOL |
| 원문 확정 역할 | Adds password authorization state. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds password authorization state.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `handlePass()`가 재등록 462, parameter 부족 461, 잘못된 password 464+close를 처리하고 성공 시 `passOk`를 세운 뒤 `maybeRegister()`를 호출했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handlePass` |
| 핵심 state / branch | ClientState.passOk가 registration prerequisite 중 하나가 됩니다. |
| failure handling | 잘못된 password는 상태를 승인하지 않고 protocol error를 queue한 뒤 connection close를 요청합니다. |
| 보장과 다음 연결 | password가 정확히 확인되기 전 registered transition이 일어나지 않습니다. 80a639321bad와 d9e420b570a0가 나머지 prerequisites를 추가합니다. |
| 이 시점의 한계 | nickname/USER prerequisite가 아직 없으므로 PASS만으로 등록되지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `582317254e24` | `src/RegistrationCommands.cpp` | `IrcApplication::handlePass` | 462/461/464 branch와 passOk mutation-before-maybeRegister |

- 조사 방법: GitHub에서 `582317254e24`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `80a639321bad` — `feat(registration): 닉네임 검증과 색인 갱신`. 원문 역할은 “Adds nickname validation and collision handling.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `80a639321bad` — `feat(registration): 닉네임 검증과 색인 갱신`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IDENTITY, IRC_PROTOCOL |
| 원문 확정 역할 | Adds nickname validation and collision handling. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds nickname validation and collision handling.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | NICK missing/invalid/collision을 431/432/433으로 거부하고, canonical collision 검사를 마친 뒤에만 registry nickname을 갱신했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handleNick, nickname validation` |
| 핵심 state / branch | hasNick/display nick/reverse index가 성공 branch에서만 함께 전진합니다. |
| failure handling | collision 전에 old mapping을 제거하지 않으므로 실패가 기존 identity를 보존합니다. |
| 보장과 다음 연결 | case-insensitive uniqueness와 validation-before-mutation ordering을 command 경로에서 지킵니다. d9e420b570a0가 USER와 final registration transition을 연결합니다. |
| 이 시점의 한계 | 이미 registered client의 NICK fan-out/cleanup semantics는 channel thread에서 추가됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `80a639321bad` | `src/RegistrationCommands.cpp` | `IrcApplication::handleNick` | 431/432/433 검사 후 registry mutation |

- 조사 방법: GitHub에서 `80a639321bad`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d9e420b570a0` — `feat(registration): USER 정보와 환영 응답 연결`. 원문 역할은 “Completes PASS/NICK/USER registration and welcome replies.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.11 `d9e420b570a0` — `feat(registration): USER 정보와 환영 응답 연결`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IDENTITY, IRC_PROTOCOL |
| 원문 확정 역할 | Completes PASS/NICK/USER registration and welcome replies. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Completes PASS/NICK/USER registration and welcome replies.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | USER fields를 저장하고 `maybeRegister()`가 `passOk && hasNick && hasUser`일 때만 `registered=true`로 만든 뒤 001, 002, 003을 순서대로 queue했습니다. |
| 확인한 변경 파일·symbol | `src/RegistrationCommands.cpp — handleUser, maybeRegister` |
| 핵심 state / branch | 등록은 세 prerequisite의 conjunction에서 false→true로 한 번 전이하며 welcome send 전에 flag를 먼저 세워 재진입 중 중복 등록을 막습니다. |
| failure handling | parameter 부족/이미 등록은 numeric으로 거부합니다. 다만 welcome enqueue가 connection을 제거해도 이 버전은 이후 send/log를 계속할 수 있습니다. |
| 보장과 다음 연결 | PASS/NICK/USER가 모두 충족된 client만 registered가 되고 welcome 순서가 정해집니다. 6b4a7738a285가 실제 TCP에서 정상 등록을 통합 검증하고 08 thread가 failure semantics를 보강합니다. |
| 이 시점의 한계 | response failure 후 application state revalidation은 728aaabc4012/5edcafda8a4d 이전에 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d9e420b570a0` | `src/RegistrationCommands.cpp` | `IrcApplication::maybeRegister` | 세 prerequisite conjunction, registered mutation, 001→002→003 |

- 조사 방법: GitHub에서 `d9e420b570a0`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`. 원문 역할은 “Proves the integrated registration and command path over real TCP.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.12 `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, INTEGRATION, RISK |
| 원문 확정 역할 | Proves the integrated registration and command path over real TCP. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Proves the integrated registration and command path over real TCP.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | parser·registry·registration·channel 기능이 실제 process/socket 안에서 함께 동작한다는 통합 증거가 없었습니다. |
| 주요 문제와 설계 판단 | Makefile test/smoke, shell runner, Python peer를 추가해 loopback server를 실행하고 다중 client 시나리오를 수행했습니다. |
| 변경 파일·symbol | `Makefile — test, smoke`<br>`tests/irc_smoke.sh`<br>`tools/irc_smoke_client.py` |
| state / ownership 변화 | test harness가 process PID, free port, temp log, peer sockets와 수신 line buffer를 소유하고 EXIT trap으로 정리합니다. |
| failure 또는 boundary | wrong password, fragmented input, case-insensitive collision, channel/mode/message/departure 오류를 실제 wire에서 관찰합니다. |
| 보장 / 비보장 | 보장: transport framing부터 application/channel/output queue까지 주요 정상·일부 오류 경로가 한 process에서 조합됩니다.<br>비보장: assertion은 주로 substring 중심이어서 exact 전체 frame/order/CRLF와 rare syscall failure, capacity/fairness를 증명하지 않습니다. |
| 다음 관련 변화 | e5e6c57db80d가 exact public contract로 정밀도를 높이고 계층별 deterministic tests가 후속됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `6b4a7738a285` | `tests/irc_smoke.sh` | `runner and cleanup trap` | 실제 server process 기동·readiness·로그·종료 관리 |
| `6b4a7738a285` | `tools/irc_smoke_client.py` | `IrcPeer and scenario` | fragmented write와 registration/channel/messaging 시나리오 |

- 조사 방법: GitHub에서 `6b4a7738a285`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | Connection framing, parser, registration, nickname index, channel state와 queued send가 실제 process에서 결합됩니다. |
| 재현 failure / boundary | wrong password, fragmented PING, nickname collision, JOIN/TOPIC/PRIVMSG/INVITE/MODE/KICK/PART/QUIT입니다. |
| test technique | compiled server + real loopback TCP + shell process harness + Python peer의 broad end-to-end smoke입니다. |
| 통과하는 production path | process startup → Server event loop → Connection framing → parser → IrcApplication → registry/channel → output queue입니다. |
| 이 test가 증명하는 것 | 대표 정상·오류 흐름이 실제 socket에서 연결되어 진행됨을 증명합니다. |
| 이 test가 증명하지 않는 것 | exact 모든 frame, 모든 ordering, rare send/event failure, formal capacity·fairness는 증명하지 않습니다. |
| test 성격 | broad integration smoke |
| 막는 회귀 | packet boundary를 command boundary로 오인하거나 identity/channel fan-out 통합이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `6b4a7738a285`
- repository에 정의된 실행 명령: `make test` 또는 `make smoke`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| wire syntax value | `a22bf6ddbd75` | c31a32b6cb24, aeb1e9b709b9 | structured value, parser reset/grammar, reply serializer를 분리합니다. | 6b4a7738a285가 실제 fragmented wire와 response 조합을 확인합니다. |
| descriptor-keyed registration state | `991b76b8d793` | 2ed9331124bb | transport lifecycle와 별도 ClientState를 fd로 연결합니다. | 5edcafda8a4d가 response failure cleanup을 보강합니다. |
| canonical nickname uniqueness | `b47135c51cfc` | 80a639321bad | collision-before-mutation과 erase 시 reverse index 정리를 지킵니다. | 6b4a7738a285의 case-insensitive collision scenario |
| transport/protocol 책임 분리 | `0b5ae6aef328` | 2ed9331124bb, 035e1137e0dd | Server callback 위에서 parser·registry·dispatch를 application이 소유합니다. | 728aaabc4012/5edcafda8a4d가 send reentrancy를 보강합니다. |
| registration monotonic transition | `582317254e24` | 80a639321bad, d9e420b570a0 | PASS/NICK/USER conjunction에서 registered를 한 번 세우고 001→002→003을 queue합니다. | 6b4a7738a285 정상 흐름, 5edcafda8a4d 실패 흐름 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| handler별 command casing/serialization | dispatch와 wire shape 불일치 | a22bf6ddbd75 + c31a32b6cb24 + aeb1e9b709b9 | structured parser/reply boundary | 6b4a7738a285 |
| display nick만 비교 | case variant collision·stale reverse mapping | b47135c51cfc + 80a639321bad | canonical collision-before-mutation | 6b4a7738a285 |
| welcome send는 실패하지 않음 | 등록 중 synchronous disconnect 후 stale state | 728aaabc4012 | send bool 전파와 state relookup | 5edcafda8a4d |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| frame text ↔ structured message | handler별 문자열 | `IrcMessage`/parser/reply helper | a22bf6ddbd75 → aeb1e9b709b9 | path/symbol은 위 commit 증거표에 연결했습니다. |
| connection별 IRC identity | transport와 혼합 가능 | `ClientRegistry` | 991b76b8d793 | path/symbol은 위 commit 증거표에 연결했습니다. |
| nickname lookup/uniqueness | linear/display-only 가능 | canonical reverse index | b47135c51cfc | path/symbol은 위 commit 증거표에 연결했습니다. |
| protocol orchestration | Server에 혼합 가능 | `IrcApplication` | 0b5ae6aef328 | path/symbol은 위 commit 증거표에 연결했습니다. |
| registration authorization | handler별 검사 가능 | central gate + `maybeRegister()` | 035e1137e0dd → d9e420b570a0 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: IRC line을 구조화 값과 wire serialization으로 표현하는 공통 타입이 없었습니다.
- 마지막 commit `6b4a7738a285` 시점의 상태: transport framing부터 application/channel/output queue까지 주요 정상·일부 오류 경로가 한 process에서 조합됩니다.
- Thread 안에서 강화된 핵심 invariant: wire syntax value, descriptor-keyed registration state, canonical nickname uniqueness, transport/protocol 책임 분리, registration monotonic transition.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: assertion은 주로 substring 중심이어서 exact 전체 frame/order/CRLF와 rare syscall failure, capacity/fairness를 증명하지 않습니다. e5e6c57db80d가 exact public contract로 정밀도를 높이고 계층별 deterministic tests가 후속됩니다.
- 최종 설명: `a22bf6ddbd75`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `6b4a7738a285`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| complete line | 2ed9331124bb | `Server line callback` | IrcApplication::onLine | missing state는 생성, disconnect는 cleanup |
| grammar | c31a32b6cb24 | `IrcApplication::onLine` | parseLine → IrcMessage | malformed/oversize는 dispatch하지 않음 |
| gate | 035e1137e0dd | `handleMessage` | ClientState.registered와 allowlist | 미등록 금지 451, 등록 후 미지원 421 |
| registration | 582317254e24 / 80a639321bad / d9e420b570a0 | `PASS/NICK/USER handlers` | maybeRegister conjunction → registered | invalid/collision/password failure는 mutation 없이 numeric/close |
| welcome output | d9e420b570a0 | `maybeRegister` | 001 → 002 → 003 → Server::sendTo | send failure revalidation은 728aaabc4012에서 보강 |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 02-protocol-boundary-identity-and-registration.md =====

===== BEGIN FILE: 03-channel-authority-fanout-and-cleanup.md =====
# Thread 03 — Channel authority, fan-out, and cleanup

부제: 채널 권한, 팬아웃과 정리

## 1. Thread 목표

Channel aggregate가 membership·operator·invitation·topic·mode를 authoritative하게 유지하고, application이 fan-out과 identity/departure cleanup을 통해 client/index/channel graph를 일관되게 만드는 과정을 복원합니다.

Source에서 확정된 significance:

> Channel functionality becomes reliable only when the same state model governs admission, fan-out, identity changes, explicit departures, kicks, and transport disconnects. The S-level cleanup commit is the decisive point: it makes client, nickname, and channel state converge after every termination path and prevents stale descriptors or duplicate peer notifications.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- operator가 항상 member라는 invariant는 aggregate 어느 연산에서 강제되는가?
- channel lookup과 create-on-demand는 어떤 command에서 분리되는가?
- 여러 shared channel을 가진 peer에게 NICK/QUIT를 한 번만 보내는 recipient 계산은 무엇인가?
- NICK, PART, QUIT, transport disconnect에서 old identity와 membership을 어느 시점까지 유지하는가?
- JOIN의 validation, invitation consumption, first-member operator, broadcast/topic/NAMES 순서는 무엇인가?
- MODE `+i`, `+t`, `+o`/`-o`가 authorization과 parameter cursor를 어떻게 유지하는가?

### Source와 직접 연결되는 invariant

- channel membership, operator state, invitations, empty-channel lifetime은 client/nickname state와 일치해야 합니다.
- disconnected descriptor는 어떤 channel에도 남아서는 안 되며 shared peer는 identity/quit transition을 최대 한 번 받아야 합니다.
- invite-only admission, topic protection, kick, invitation, operator mode는 membership과 operator authority를 강제해야 합니다.

### Source와 직접 연결되는 engineering difficulty

- NICK/PART/KICK/QUIT/transport failure/fan-out/empty-channel erase를 하나의 shared-state graph에서 일관되게 처리하는 문제.
- broadcast 중 lifecycle mutation이 borrowed channel/client state를 지울 수 있는 후속 reentrancy 문제.

## 3. 완료 기준

- `Channel`의 모든 authoritative state와 non-owning client identifier 관계를 정리했습니다.
- member/operator/invitation/topic/mode mutation을 실제 aggregate method와 command handler caller로 연결했습니다.
- deduplicated common-peer fan-out과 disconnect cleanup 순서를 snapshot/erase 시점까지 설명합니다.
- JOIN과 MODE의 권한 및 mutation-before-broadcast ordering을 해당 SHA 코드로 확인했습니다.
- send failure와 reentrant cleanup 한계는 Thread 08의 후속 fix/test와 연결했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `786ed5d839d9` | `feat(channel): 채널 상태 계약 정의` | A | ARCH, CHANNEL_STATE | Defines the authoritative Channel aggregate. |
| 2 | `966f5663dbcc` | `feat(channel): 구성원과 운영자 상태 관리` | B | CHANNEL_STATE | Implements membership and operator state. |
| 3 | `0d0d850f007d` | `feat(channel): 주제·초대·모드와 이름 규칙 구현` | B | CHANNEL_STATE, IRC_PROTOCOL | Implements topic, invitation, mode, and name rules. |
| 4 | `c1762e011fd6` | `feat(channel): 채널 탐색과 대상 해석 지원` | B | CHANNEL_STATE | Adds application-owned channel storage and command lookup. |
| 5 | `46e2b7785bee` | `feat(channel): 채널 방송 대상 팬아웃 지원` | A | CHANNEL_STATE, INTEGRATION | Introduces reusable, deduplicated channel and common-peer fan-out. |
| 6 | `a147d6994d58` | `feat(channel): 구성원 정리와 식별자 변경 방송` | S | CORE, CHANNEL_STATE, LIFECYCLE | Coordinates nickname transitions, PART, QUIT, membership removal, and empty-channel erasure. |
| 7 | `7ac793d3b695` | `feat(message): 채널 대상 메시지 방송` | B | IRC_PROTOCOL, CHANNEL_STATE | Routes PRIVMSG through membership-authorized channel fan-out. |
| 8 | `22e5f82bc693` | `feat(channel): JOIN 채널 입장 처리` | A | CHANNEL_STATE, IRC_PROTOCOL, RISK | Implements the central JOIN transition, invite-only admission, and first-member operator assignment. |
| 9 | `0e601da7d2bc` | `feat(channel): 채널 모드 조회와 i·t 변경` | B | CHANNEL_STATE, IRC_PROTOCOL | Exposes and mutates +i and +t under operator authorization. |
| 10 | `bce6933f69ab` | `feat(channel): 채널 운영자 모드 변경` | B | CHANNEL_STATE, IRC_PROTOCOL | Extends channel authority with +o and -o. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `786ed5d839d9` — `feat(channel): 채널 상태 계약 정의`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | ARCH, CHANNEL_STATE |
| 원문 확정 역할 | Defines the authoritative Channel aggregate. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Defines the authoritative Channel aggregate.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | application에 채널별 구성원·운영자·초대·주제·모드를 함께 보존하는 authoritative aggregate가 없었습니다. |
| 주요 문제와 설계 판단 | `Channel`이 이름, member fd set, operator fd set, canonical nickname invitation set, topic 존재/문자열, `+i`·`+t` 상태를 private field로 소유하고 mutation/query API를 제공하도록 계약을 정의했습니다. |
| 변경 파일·symbol | `include/Channel.hpp — Channel state and public mutation/query API` |
| state / ownership 변화 | 채널은 client fd를 비소유 식별자로만 보관합니다. socket과 `Connection`의 lifetime은 계속 `Server`가 소유하고 `Channel`은 membership graph만 소유합니다. |
| failure 또는 boundary | container를 handler에 직접 노출하지 않아 비회원 운영자, casing이 다른 invitation, topic presence와 text의 불일치를 aggregate 내부에서 막을 수 있게 했습니다. |
| 보장 / 비보장 | 보장: membership·authority·invitation·topic·mode를 한 객체에서 갱신할 수 있는 단일 state owner가 생깁니다.<br>비보장: 각 mutation의 실제 구현과 command authorization은 아직 없습니다. |
| 다음 관련 변화 | 966f5663dbcc가 member/operator invariant를, 0d0d850f007d가 topic/invite/mode/name 규칙을 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `786ed5d839d9` | `include/Channel.hpp` | `Channel private fields` | member/operator/invitation/topic/mode의 authoritative 저장 위치 |
| `786ed5d839d9` | `include/Channel.hpp` | `member and mode API` | handler가 container를 직접 수정하지 않는 aggregate 경계 |

- 조사 방법: GitHub에서 `786ed5d839d9`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `966f5663dbcc` — `feat(channel): 구성원과 운영자 상태 관리`. 원문 역할은 “Implements membership and operator state.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `966f5663dbcc` — `feat(channel): 구성원과 운영자 상태 관리`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | CHANNEL_STATE |
| 원문 확정 역할 | Implements membership and operator state. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Implements membership and operator state.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | member 추가·삭제와 operator 승격·해제를 aggregate method로 구현했습니다. `removeMember()`는 같은 fd를 operator set에서도 지우고 `setOperator(true)`는 현재 member인 fd만 허용했습니다. |
| 확인한 변경 파일·symbol | `src/Channel.cpp — addMember, removeMember, hasMember, setOperator, isOperator, members` |
| 핵심 state / branch | operator set은 member set의 부분집합으로 유지되며 `members()`는 fan-out과 안전한 iteration에 사용할 값 snapshot을 반환합니다. |
| failure handling | 비회원에게 운영자 권한을 부여하려는 요청은 상태를 변경하지 않으며, 퇴장한 fd가 operator로 남지 않습니다. |
| 보장과 다음 연결 | “operator는 항상 member”라는 핵심 채널 invariant가 모든 aggregate mutation에서 유지됩니다. 0d0d850f007d가 나머지 채널 상태를 구현하고 bce6933f69ab가 `+o/-o` command에 연결합니다. |
| 이 시점의 한계 | invitation·topic·mode와 실제 MODE authorization은 아직 없습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `966f5663dbcc` | `src/Channel.cpp` | `Channel::removeMember` | member와 operator state를 함께 제거 |
| `966f5663dbcc` | `src/Channel.cpp` | `Channel::setOperator` | 비회원 운영자 승격을 거부 |

- 조사 방법: GitHub에서 `966f5663dbcc`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `0d0d850f007d` — `feat(channel): 주제·초대·모드와 이름 규칙 구현`. 원문 역할은 “Implements topic, invitation, mode, and name rules.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `0d0d850f007d` — `feat(channel): 주제·초대·모드와 이름 규칙 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | CHANNEL_STATE, IRC_PROTOCOL |
| 원문 확정 역할 | Implements topic, invitation, mode, and name rules. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Implements topic, invitation, mode, and name rules.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | topic presence/text, canonical nickname invitation, invite-only와 topic-protected flag, 채널 이름 검증 및 nickname canonicalization을 구현했습니다. |
| 확인한 변경 파일·symbol | `src/Channel.cpp — topic, invitation, mode, name/canonical helpers` |
| 핵심 state / branch | 초대는 display spelling이 아니라 canonical key로 저장되고, topic은 text와 존재 여부를 함께 표현합니다. |
| failure handling | 잘못된 channel name은 생성 전에 거부할 수 있고, clearInvite는 없는 초대에도 안전하며 mode query는 현재 aggregate state만 읽습니다. |
| 보장과 다음 연결 | 입장·TOPIC·MODE command가 의존할 authoritative channel policy state가 완성됩니다. c1762e011fd6가 application-owned channel map을, 22e5f82bc693과 0e601da7d2bc가 command transition을 추가합니다. |
| 이 시점의 한계 | application map과 command lookup/authorization은 아직 연결되지 않았습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0d0d850f007d` | `src/Channel.cpp` | `Channel::canonicalNick / invite / clearInvite` | case-insensitive invitation state |
| `0d0d850f007d` | `src/Channel.cpp` | `topic and mode methods` | topic presence와 +i/+t mutation을 aggregate에 집중 |

- 조사 방법: GitHub에서 `0d0d850f007d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `c1762e011fd6` — `feat(channel): 채널 탐색과 대상 해석 지원`. 원문 역할은 “Adds application-owned channel storage and command lookup.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `c1762e011fd6` — `feat(channel): 채널 탐색과 대상 해석 지원`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | CHANNEL_STATE |
| 원문 확정 역할 | Adds application-owned channel storage and command lookup. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds application-owned channel storage and command lookup.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `IrcApplication`에 channel map을 추가하고 create-on-demand `ensureChannel()`과 non-creating lookup/command helper를 분리했습니다. command helper는 존재·membership 요구에 따라 403/442를 보냅니다. |
| 확인한 변경 파일·symbol | `src/IrcApplication.hpp — _channels`<br>`src/ApplicationSupport.cpp — ensureChannel, findChannelForCommand, eraseChannelIfEmpty` |
| 핵심 state / branch | channel object lifetime의 owner는 `IrcApplication::_channels`이고 handler가 보유한 pointer/reference는 해당 map entry가 존재하는 동안만 유효합니다. |
| failure handling | 조회 command가 실수로 채널을 생성하지 않으며, 없는 채널과 비회원 접근이 mutation 전에 종료됩니다. |
| 보장과 다음 연결 | 채널 생성과 조회의 의미가 command별로 구분되고 empty-channel erase 종착점이 생깁니다. 46e2b7785bee가 map의 member snapshots를 이용한 fan-out을 추가하고 728aaabc4012가 send 뒤 relookup 규칙을 보강합니다. |
| 이 시점의 한계 | broadcast 중 map entry가 지워지는 reentrancy는 아직 방어하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c1762e011fd6` | `src/ApplicationSupport.cpp` | `ensureChannel` | create-on-demand가 필요한 command만 map entry 생성 |
| `c1762e011fd6` | `src/ApplicationSupport.cpp` | `findChannelForCommand` | 403/442를 mutation 전 공통 검사 |

- 조사 방법: GitHub에서 `c1762e011fd6`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `46e2b7785bee` — `feat(channel): 채널 방송 대상 팬아웃 지원`. 원문 역할은 “Introduces reusable, deduplicated channel and common-peer fan-out.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `46e2b7785bee` — `feat(channel): 채널 방송 대상 팬아웃 지원`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | CHANNEL_STATE, INTEGRATION |
| 원문 확정 역할 | Introduces reusable, deduplicated channel and common-peer fan-out. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Introduces reusable, deduplicated channel and common-peer fan-out.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | handler마다 channel member를 직접 순회해야 했고 여러 공통 채널을 가진 peer에게 동일 event가 중복될 수 있었습니다. |
| 주요 문제와 설계 판단 | 한 channel의 `members()` snapshot을 순회하는 broadcast와, sender가 속한 모든 channel의 peer fd를 `std::set<int>`에 합치는 common-peer fan-out을 구현했습니다. |
| 변경 파일·symbol | `src/ApplicationSupport.cpp — broadcastToChannel, broadcastToCommonPeers` |
| state / ownership 변화 | fan-out 도중 순회 기준은 live container iterator가 아니라 fd snapshot이며 recipient dedup set이 한 transition당 최대 한 번 전송을 보장합니다. |
| failure 또는 boundary | 없는 client/connection으로 보내는 개별 실패는 server send 결과와 cleanup으로 넘어가며 다른 recipient 계산은 snapshot에 의해 계속 결정됩니다. |
| 보장 / 비보장 | 보장: NICK/QUIT 같은 identity transition이 shared channel 수와 무관하게 같은 peer에게 한 번만 전달될 수 있습니다.<br>비보장: send가 sender나 channel을 동기적으로 지울 수 있다는 cross-layer invalidation은 아직 처리하지 않습니다. |
| 다음 관련 변화 | a147d6994d58가 identity/departure cleanup을 이 fan-out 위에 구현하고 08 thread가 reentrancy를 수정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `46e2b7785bee` | `src/ApplicationSupport.cpp` | `broadcastToChannel` | member fd snapshot 기반 channel fan-out |
| `46e2b7785bee` | `src/ApplicationSupport.cpp` | `broadcastToCommonPeers` | std::set recipient dedup으로 중복 notification 방지 |

- 조사 방법: GitHub에서 `46e2b7785bee`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `a147d6994d58` — `feat(channel): 구성원 정리와 식별자 변경 방송`. 원문 역할은 “Coordinates nickname transitions, PART, QUIT, membership removal, and empty-channel erasure.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `a147d6994d58` — `feat(channel): 구성원 정리와 식별자 변경 방송`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | CORE, CHANNEL_STATE, LIFECYCLE |
| 원문 확정 역할 | Coordinates nickname transitions, PART, QUIT, membership removal, and empty-channel erasure. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Coordinates nickname transitions, PART, QUIT, membership removal, and empty-channel erasure.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | channel fan-out은 있었지만 NICK/PART/QUIT/transport disconnect가 client index와 모든 membership을 하나의 순서로 정리하지 못했습니다. |
| failure / boundary | 여러 shared channel을 가진 peer는 dedup set 때문에 QUIT/NICK을 한 번만 받으며 transport disconnect도 explicit QUIT과 같은 cleanup primitives를 재사용합니다. |
| 핵심 결정 | 공통 peer를 먼저 계산하고 old identity prefix로 NICK/QUIT/PART를 방송한 뒤, 각 channel membership을 제거하고 empty channel을 erase하며 마지막에 registry nickname/client state를 갱신·삭제하도록 수명 전이를 조율했습니다. |
| 변경 파일·symbol | `src/ApplicationSupport.cpp — partAllChannels, partChannel, removeClientFromChannels, broadcastToCommonPeers`<br>`src/RegistrationCommands.cpp — registered NICK transition`<br>`src/IrcApplication.cpp — QUIT/disconnect cleanup` |
| ownership / lifecycle / state transition | old nick와 membership은 outbound event를 구성할 때까지 유지되고, recipient snapshot 이후 channel graph와 nickname index/client record가 수렴합니다. |
| 이 commit이 보장하는 것 | 퇴장한 fd가 channel/operator set이나 nickname reverse index에 남지 않고 empty channel이 map에 잔존하지 않습니다. |
| 아직 보장하지 않는 것 | broadcast 중 send failure가 map/client를 지운 뒤 borrowed reference를 사용하는 위험은 728aaabc4012 이전에 남습니다. |
| 후속 fix/test 연결 | 22e5f82bc693 등 정상 command가 같은 graph를 확장하고 08 thread의 fix/test가 reentrant cleanup을 강화합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | channel fan-out은 있었지만 NICK/PART/QUIT/transport disconnect가 client index와 모든 membership을 하나의 순서로 정리하지 못했습니다. |
| 2. failure conditions / boundary | 여러 shared channel을 가진 peer는 dedup set 때문에 QUIT/NICK을 한 번만 받으며 transport disconnect도 explicit QUIT과 같은 cleanup primitives를 재사용합니다. |
| 3. 선택한 표현과 순서 | 공통 peer를 먼저 계산하고 old identity prefix로 NICK/QUIT/PART를 방송한 뒤, 각 channel membership을 제거하고 empty channel을 erase하며 마지막에 registry nickname/client state를 갱신·삭제하도록 수명 전이를 조율했습니다. |
| 4. authoritative state | old nick와 membership은 outbound event를 구성할 때까지 유지되고, recipient snapshot 이후 channel graph와 nickname index/client record가 수렴합니다. |
| 5. 결과 invariant | 퇴장한 fd가 channel/operator set이나 nickname reverse index에 남지 않고 empty channel이 map에 잔존하지 않습니다. |
| 6. 후속 보강 | 22e5f82bc693 등 정상 command가 같은 graph를 확장하고 08 thread의 fix/test가 reentrant cleanup을 강화합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a147d6994d58` | `src/ApplicationSupport.cpp` | `partAllChannels / removeClientFromChannels` | channel snapshot → membership 제거 → empty erase 순서 |
| `a147d6994d58` | `src/RegistrationCommands.cpp` | `registered NICK handling` | old prefix fan-out 후 canonical index 갱신 |
| `a147d6994d58` | `src/IrcApplication.cpp` | `QUIT/onDisconnect cleanup` | explicit/transport 종료가 동일 state graph를 정리 |

- 조사 방법: GitHub에서 `a147d6994d58`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `7ac793d3b695` — `feat(message): 채널 대상 메시지 방송`. 원문 역할은 “Routes PRIVMSG through membership-authorized channel fan-out.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `7ac793d3b695` — `feat(message): 채널 대상 메시지 방송`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | IRC_PROTOCOL, CHANNEL_STATE |
| 원문 확정 역할 | Routes PRIVMSG through membership-authorized channel fan-out. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Routes PRIVMSG through membership-authorized channel fan-out.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | channel target이면 channel 존재와 sender membership을 확인하고, 성공 시 sender를 제외한 member snapshot에 source hostmask의 PRIVMSG를 방송했습니다. |
| 확인한 변경 파일·symbol | `src/MessagingCommands.cpp — handlePrivmsg channel branch` |
| 핵심 state / branch | message text는 channel aggregate를 변경하지 않고 현재 membership snapshot만 읽습니다. |
| failure handling | 없는 channel은 403, 비회원 발신은 404로 끝나며 fan-out side effect가 없습니다. |
| 보장과 다음 연결 | channel 메시지는 authorized member에서 현재 member들로만 전달되고 sender echo는 제외됩니다. de1dd0fc30d0가 실제 slow receiver 아래 unrelated connection progress를 검증합니다. |
| 이 시점의 한계 | slow receiver와 send failure 격리는 output/server layer에 맡기며 이 commit 자체는 fairness를 보장하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7ac793d3b695` | `src/MessagingCommands.cpp` | `IrcApplication::handlePrivmsg` | channel 존재·membership 검사 후 sender 제외 fan-out |

- 조사 방법: GitHub에서 `7ac793d3b695`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `22e5f82bc693` — `feat(channel): JOIN 채널 입장 처리`. 원문 역할은 “Implements the central JOIN transition, invite-only admission, and first-member operator assignment.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `22e5f82bc693` — `feat(channel): JOIN 채널 입장 처리`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | CHANNEL_STATE, IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Implements the central JOIN transition, invite-only admission, and first-member operator assignment. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Implements the central JOIN transition, invite-only admission, and first-member operator assignment.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 채널 aggregate와 helper는 있었지만 JOIN의 admission·membership·첫 운영자·응답 순서가 한 transition으로 결합되지 않았습니다. |
| 주요 문제와 설계 판단 | JOIN 0은 전체 PART로 처리하고, 쉼표 목록의 각 이름을 검증한 뒤 channel을 확보합니다. invite-only admission을 확인하고 첫 member만 operator로 추가하며 초대를 소비한 뒤 JOIN 방송, TOPIC reply, NAMES reply 순서로 전송했습니다. |
| 변경 파일·symbol | `src/ChannelCommands.cpp — handleJoin`<br>`src/ApplicationSupport.cpp — sendTopicReply, sendNames` |
| state / ownership 변화 | membership mutation은 JOIN fan-out 전에 완료되고 invitation은 성공 입장 시에만 제거됩니다. 첫 member 여부는 mutation 전 `empty()` 결과로 결정됩니다. |
| failure 또는 boundary | invalid name은 403, invite-only 미초대는 473이며 기존 member의 재JOIN은 state를 중복 추가하지 않고 NAMES만 보냅니다. |
| 보장 / 비보장 | 보장: 입장 authorization과 first-member operator invariant, 응답 순서가 하나의 handler에서 고정됩니다.<br>비보장: 각 send 뒤 actor/channel lifetime 재검증은 728aaabc4012 이전에 부족합니다. |
| 다음 관련 변화 | 0e601da7d2bc와 bce6933f69ab가 입장 후 operator가 변경할 channel mode를 구현합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `22e5f82bc693` | `src/ChannelCommands.cpp` | `IrcApplication::handleJoin` | validation → admission → addMember(first) → invite clear → JOIN/TOPIC/NAMES 순서 |

- 조사 방법: GitHub에서 `22e5f82bc693`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `0e601da7d2bc` — `feat(channel): 채널 모드 조회와 i·t 변경`. 원문 역할은 “Exposes and mutates +i and +t under operator authorization.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `0e601da7d2bc` — `feat(channel): 채널 모드 조회와 i·t 변경`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | CHANNEL_STATE, IRC_PROTOCOL |
| 원문 확정 역할 | Exposes and mutates +i and +t under operator authorization. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Exposes and mutates +i and +t under operator authorization.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | MODE 조회는 324와 current mode string을 보내고, 변경은 membership과 operator를 확인한 뒤 mode 문자열을 순회해 `+i/-i`, `+t/-t`를 aggregate에 반영하고 방송했습니다. |
| 확인한 변경 파일·symbol | `src/ChannelCommands.cpp — handleChannelMode, broadcastMode` |
| 핵심 state / branch | adding/removing cursor가 mode string 전체에 적용되고 각 성공 mutation 직후 현재 channel members에게 mode event를 fan-out합니다. |
| failure handling | 비회원 442, 비운영자 482, 알 수 없는 mode 472로 mutation 없이 거부합니다. |
| 보장과 다음 연결 | invite-only와 topic-protection policy를 channel operator만 변경할 수 있습니다. bce6933f69ab가 argument를 소비하는 `o` mode를 추가하고 d48e1f1f8c04/aee5edebe294가 continuation semantics를 수정·검증합니다. |
| 이 시점의 한계 | compound mode 중 첫 broadcast failure 뒤 다음 mode가 계속 적용되는 문제는 d48e1f1f8c04까지 남습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0e601da7d2bc` | `src/ChannelCommands.cpp` | `handleChannelMode +i/+t` | membership/operator 검사, mode cursor, mutation 후 broadcast |

- 조사 방법: GitHub에서 `0e601da7d2bc`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `bce6933f69ab` — `feat(channel): 채널 운영자 모드 변경`. 원문 역할은 “Extends channel authority with +o and -o.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.10 `bce6933f69ab` — `feat(channel): 채널 운영자 모드 변경`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | CHANNEL_STATE, IRC_PROTOCOL |
| 원문 확정 역할 | Extends channel authority with +o and -o. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Extends channel authority with +o and -o.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | mode cursor와 별도의 parameter cursor를 사용해 `+o/-o` 대상 nick을 소비하고, nickname lookup과 channel membership을 확인한 뒤 `Channel::setOperator()`를 호출해 실제 display nick으로 방송했습니다. |
| 확인한 변경 파일·symbol | `src/ChannelCommands.cpp — handleChannelMode o branch` |
| 핵심 state / branch | operator state는 Channel aggregate만 변경하고 nickname은 ClientRegistry에서 lookup한 현재 display value를 사용합니다. |
| failure handling | 인자 부족은 461, 존재하지 않거나 비회원인 target은 441이며 operator set을 변경하지 않습니다. |
| 보장과 다음 연결 | 운영자 권한 변경도 “operator는 member” invariant와 command parameter ordering을 따릅니다. d48e1f1f8c04와 aee5edebe294가 response failure 후 더 이상의 mode transition을 막습니다. |
| 이 시점의 한계 | send failure 뒤 compound mode continuation과 target lifetime revalidation은 08 thread에서 보강됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `bce6933f69ab` | `src/ChannelCommands.cpp` | `handleChannelMode o branch` | argument cursor, nick lookup/member 검사, setOperator와 broadcast |

- 조사 방법: GitHub에서 `bce6933f69ab`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| authoritative Channel aggregate | `786ed5d839d9` | 966f5663dbcc, 0d0d850f007d | member/operator/invitation/topic/mode를 한 객체에서 관리합니다. | command integration은 22e5f82bc693, 0e601da7d2bc, bce6933f69ab |
| operator ⊆ member | `966f5663dbcc` | bce6933f69ab | removeMember가 operator도 제거하고 비회원 승격을 거부합니다. | aee5edebe294가 failure 중 unrelated member/channel 보존을 확인 |
| deduplicated peer fan-out | `46e2b7785bee` | a147d6994d58 | member snapshot과 std::set recipient로 shared peer 중복을 제거합니다. | 6b4a7738a285의 multi-client flow |
| identity/departure graph cleanup | `a147d6994d58` | PART/QUIT/NICK/transport disconnect | old identity로 notification 후 membership/index/client state를 수렴시킵니다. | 728aaabc4012/5edcafda8a4d가 send-failure reentrancy 보강 |
| authorized admission/mode | `22e5f82bc693` | 0e601da7d2bc, bce6933f69ab | invite/member/operator 검사를 mutation 전에 수행합니다. | d48e1f1f8c04/aee5edebe294가 compound failure continuation 고정 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| handler가 여러 container를 직접 수정 | 비회원 operator·stale invite/index | 786ed5d839d9 → 0d0d850f007d | aggregate method 뒤에 state mutation 집중 | 6b4a7738a285 |
| shared channel마다 NICK/QUIT 전송 | 같은 peer 중복 notification | 46e2b7785bee + a147d6994d58 | recipient fd set dedup | 6b4a7738a285 |
| broadcast가 local side effect뿐 | send 중 actor/channel erase 후 dangling reference | 728aaabc4012 | stable value copy와 container relookup | 5edcafda8a4d / aee5edebe294 |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| channel membership/authority | 분산 container 가능 | `Channel` aggregate | 786ed5d839d9 → 0d0d850f007d | path/symbol은 위 commit 증거표에 연결했습니다. |
| channel object lifetime | 없음 | `IrcApplication::_channels` map | c1762e011fd6 | path/symbol은 위 commit 증거표에 연결했습니다. |
| recipient computation | handler별 반복 | broadcast helper/snapshot/dedup set | 46e2b7785bee | path/symbol은 위 commit 증거표에 연결했습니다. |
| identity/departure cleanup | 명령별 분산 가능 | IrcApplication cleanup helpers | a147d6994d58 | path/symbol은 위 commit 증거표에 연결했습니다. |
| socket lifetime | Channel이 소유하지 않음 | Server/Connection, Channel에는 fd만 저장 | 전체 Thread | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: application에 채널별 구성원·운영자·초대·주제·모드를 함께 보존하는 authoritative aggregate가 없었습니다.
- 마지막 commit `bce6933f69ab` 시점의 상태: 운영자 권한 변경도 “operator는 member” invariant와 command parameter ordering을 따릅니다.
- Thread 안에서 강화된 핵심 invariant: authoritative Channel aggregate, operator ⊆ member, deduplicated peer fan-out, identity/departure graph cleanup, authorized admission/mode.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: send failure 뒤 compound mode continuation과 target lifetime revalidation은 08 thread에서 보강됩니다. d48e1f1f8c04와 aee5edebe294가 response failure 후 더 이상의 mode transition을 막습니다.
- 최종 설명: `786ed5d839d9`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `bce6933f69ab`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| channel resolve | c1762e011fd6 | `command handler` | ensureChannel/findChannelForCommand | invalid/missing/nonmember는 403/442 |
| authorize | 22e5f82bc693 / 0e601da7d2bc | `JOIN/MODE handler` | invite/member/operator state | 473/482 등에서 mutation 없이 종료 |
| mutate | 966f5663dbcc / 0d0d850f007d | `handler` | Channel aggregate methods | 비회원 operator와 stale operator를 방지 |
| fan-out | 46e2b7785bee | `application helper` | member snapshot/common-peer set | 개별 send failure는 Server cleanup으로 전달 |
| departure cleanup | a147d6994d58 | `PART/QUIT/onDisconnect` | old prefix notification → member 제거 → empty erase → registry cleanup | send reentrancy는 728aaabc4012에서 relookup |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 03-channel-authority-fanout-and-cleanup.md =====

===== BEGIN FILE: 04-operational-protections-and-controlled-shutdown.md =====
# Thread 04 — Operational protections and controlled shutdown

부제: 운영 보호와 제어된 종료

## 1. Thread 목표

등록·유휴·PING·명령률·pending output·connection 수를 bounded policy로 만들고, 그 상태를 metrics/log로 관찰하며 signal 이후 final output을 제한된 시간 안에 drain하는 public process behavior를 복원합니다.

Source에서 확정된 significance:

> The server evolves from functional protocol handling into an operable bounded process. Each protection addresses a distinct resource or liveness risk, while shutdown and the executable contract define how those protections appear at the public process boundary. The sequence also provides the failure signals later used to expose reentrant cleanup defects.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 각 timeout/limit은 어떤 authoritative state와 transition에서 적용되는가?
- 보호 정책 위반은 server 전체가 아니라 한 connection의 queued response와 close로 어떻게 수렴하는가?
- pending-output failure가 왜 이후 reentrant cleanup defect를 드러내는 signal이 되는가?
- counter와 gauge는 어느 계층이 기록하며 live container와 중복 source of truth를 만들지 않는가?
- signal handler와 normal control flow의 책임은 어떻게 분리되는가?
- exact executable contract가 CLI, wire, log, shutdown 순서를 어디까지 고정하는가?

### Source와 직접 연결되는 invariant

- registration, idle, ping, rate-limit decision은 coherent per-client state를 사용해야 합니다.
- pending output, connection count, registration time, command rate, heartbeat wait는 bounded 또는 명시적으로 governed되어야 합니다.
- shutdown은 teardown 전에 final protocol error queue를 시도하되 non-reading peer 때문에 무한 대기하지 않아야 합니다.

### Source와 직접 연결되는 engineering difficulty

- 서로 다른 resource/liveness risk를 한 event-loop tick과 connection-scoped cleanup에 통합하는 문제.
- heartbeat를 wall-clock movement 및 unrelated PONG에서 안전하게 만드는 문제는 후속 Thread에서 복구됩니다.
- external process behavior를 brittle하지 않으면서 exact하게 검증하는 문제.

## 3. 완료 기준

- registration/idle/ping/rate/output/connection limit마다 state field, check timing, rejection response, cleanup path를 정리했습니다.
- metrics/log가 사실이 확정되는 authoritative transition에 붙어 있음을 확인했습니다.
- SIGTERM/SIGINT에서 flag 변경, application shutdown, bounded poll drain, callback detach, stop 순서를 설명합니다.
- exact contract suite의 public-boundary 증명/비증명 범위를 기록했습니다.
- heartbeat/config/output failure의 후속 fix Thread와 연결했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `c4df44554866` | `feat(registration): 등록 대기 시간 제한` | B | RESILIENCE, LIFECYCLE | Adds a registration deadline. |
| 2 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | RESILIENCE, LIFECYCLE, RISK | Adds idle heartbeat and ping-timeout state. |
| 3 | `9e2b214f9227` | `feat(throttle): 클라이언트별 명령 호출 횟수 제한` | B | RESILIENCE | Adds a per-client command-rate window. |
| 4 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | EVENT_IO, RESILIENCE, RISK | Bounds each connection's unsent output and propagates queue failure. |
| 5 | `adb49d9466e4` | `feat(server): 최대 연결 수 제한` | B | RESILIENCE, EVENT_IO | Bounds the number of live connections. |
| 6 | `e05e35ca7da9` | `feat(metrics): 서버 실행 지표 조회 기능 추가` | B | OBSERVABILITY | Makes connection, command, message, queue-drop, and rate-limit state queryable. |
| 7 | `c34aa18f89af` | `feat(log): 연결 상태와 실행 지표 기록` | B | OBSERVABILITY, LIFECYCLE | Records structured lifecycle, protection, and aggregate metrics events. |
| 8 | `dd04279c47fd` | `feat(shutdown): 종료 전 송신 대기열 처리` | A | LIFECYCLE, RESILIENCE, RISK | Replaces immediate signal-time stop with application shutdown and output draining. |
| 9 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | Locks the resulting CLI, wire, timeout, rate, metric, shutdown, and log-order contracts. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `c4df44554866` — `feat(registration): 등록 대기 시간 제한`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | RESILIENCE, LIFECYCLE |
| 원문 확정 역할 | Adds a registration deadline. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds a registration deadline.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | connect 시 `connectedAt`을 저장하고 main event loop가 각 `pollOnce()` 뒤 `onTick()`을 호출하도록 했습니다. maintenance는 fd snapshot을 돌며 미등록 client의 elapsed time을 검사해 451을 queue하고 normal close path를 요청했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — connectedAt`<br>`src/IrcApplication.cpp — onConnect, onTick, maintainClient`<br>`src/main.cpp — pollOnce/onTick order`<br>`src/RuntimeConfig.cpp — --registration-timeout` |
| 핵심 state / branch | registration deadline state는 ClientState가 보유하고 maintenance는 registry의 fd snapshot을 사용해 disconnect 중 iterator invalidation을 피합니다. |
| failure handling | registered client는 검사에서 제외되고 timeout client만 response+close로 수렴합니다. option은 positive decimal과 1일 상한을 요구합니다. |
| 보장과 다음 연결 | 미등록 connection이 registry와 connection map을 무기한 점유하지 못합니다. 764361c52b2a가 같은 maintenance loop에 heartbeat를 추가하고 3f2b3ae1d3f9가 monotonic clock으로 교정합니다. |
| 이 시점의 한계 | 시간 기준은 이 시점의 wall clock이며 엄격한 target-width parsing은 b6c10bc51937 이전에 부족합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c4df44554866` | `src/IrcApplication.cpp` | `maintainClient registration branch` | 미등록 elapsed deadline → 451 → requestClose |
| `c4df44554866` | `src/main.cpp` | `pollOnce then onTick` | readiness 처리와 policy tick 순서 |

- 조사 방법: GitHub에서 `c4df44554866`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`. 원문 역할은 “Adds idle heartbeat and ping-timeout state.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | RESILIENCE, LIFECYCLE, RISK |
| 원문 확정 역할 | Adds idle heartbeat and ping-timeout state. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds idle heartbeat and ping-timeout state.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다. |
| 주요 문제와 설계 판단 | 모든 received line에서 `lastActivityAt`을 갱신하고, idle client에 PING token을 queue한 뒤 `awaitingPong`과 `lastPingAt`을 기록했습니다. deadline을 넘으면 ERROR를 queue하고 close를 요청했으며 PONG handler가 wait를 해제했습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — awaitingPong, lastActivityAt, lastPingAt`<br>`src/IrcApplication.cpp — onLine, maintainClient, handlePong`<br>`src/RuntimeConfig.cpp — idle/ping options` |
| state / ownership 변화 | maintenance 우선순위는 registration timeout → registered 여부 → awaiting PONG timeout → idle probe입니다. |
| failure 또는 boundary | 초기 구현은 `std::time` wall clock을 사용하고 outstanding token을 저장하지 않아 어떤 PONG도 wait를 해제합니다. |
| 보장 / 비보장 | 보장: idle probe와 timeout이라는 liveness state machine의 최초 정책을 제공합니다.<br>비보장: clock correction과 forged/unrelated PONG에 안전하지 않습니다. |
| 다음 관련 변화 | d710f29f38a4/f313e707474f가 양성 흐름을 먼저 검증하고 3f2b3ae1d3f9/0c76aad19579가 invariant를 수정·고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `764361c52b2a` | `src/IrcApplication.cpp` | `maintainClient` | registration → ping timeout → idle PING 순서 |
| `764361c52b2a` | `src/IrcApplication.cpp` | `handlePong` | 초기 버전은 token 검사 없이 awaiting 상태 해제 |

- 조사 방법: GitHub에서 `764361c52b2a`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `9e2b214f9227` — `feat(throttle): 클라이언트별 명령 호출 횟수 제한`. 원문 역할은 “Adds a per-client command-rate window.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `9e2b214f9227` — `feat(throttle): 클라이언트별 명령 호출 횟수 제한`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | RESILIENCE |
| 원문 확정 역할 | Adds a per-client command-rate window. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds a per-client command-rate window.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | ClientState에 command timestamp deque를 두고 parsed command마다 오래된 항목을 제거한 후 현재 시각을 추가했습니다. window 내 count가 설정값을 넘으면 439를 queue하고 close를 요청했습니다. |
| 확인한 변경 파일·symbol | `src/ClientRegistry.hpp — commandWindow`<br>`src/IrcApplication.cpp — recordCommand/onLine`<br>`src/RuntimeConfig.cpp — --rate-limit` |
| 핵심 state / branch | rate window는 client별로 독립이며 parser가 frame을 command로 인정한 뒤에만 기록됩니다. |
| failure handling | 초과 client만 close path로 보내고 server 전체나 다른 client state는 유지합니다. |
| 보장과 다음 연결 | 명령 호출량이 구성된 count/window 아래에서 connection 단위로 governed됩니다. e05e35ca7da9/c34aa18f89af가 limit event를 지표와 로그로 관찰 가능하게 합니다. |
| 이 시점의 한계 | 초기 timestamp는 wall-clock 기반이고 rate-count parsing의 target-width overflow는 b6c10bc51937 전까지 남습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `9e2b214f9227` | `src/IrcApplication.cpp` | `recordCommand` | old timestamp eviction → current push → count compare |
| `9e2b214f9227` | `src/IrcApplication.cpp` | `onLine rate branch` | parsed command 초과를 439와 close로 수렴 |

- 조사 방법: GitHub에서 `9e2b214f9227`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`. 원문 역할은 “Bounds each connection's unsent output and propagates queue failure.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, RESILIENCE, RISK |
| 원문 확정 역할 | Bounds each connection's unsent output and propagates queue failure. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Bounds each connection's unsent output and propagates queue failure.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | partial output queue가 무제한이라 느리거나 읽지 않는 peer가 memory를 계속 점유할 수 있었습니다. |
| 주요 문제와 설계 판단 | Connection에 `maxPendingBytes_`를 추가하고 `pendingBytes()` 기준으로 `queueRaw/queueLine` admission을 제한했습니다. queue 함수는 bool을 반환하고 초과 시 close를 요청하며 Server send API가 실패를 caller에 전달했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — maxPendingBytes and bool queue API`<br>`src/Connection.cpp — pendingBytes, canAppendPending, queueRaw, queueLine`<br>`src/Server.cpp — sendTo/queueRawTo propagation` |
| state / ownership 변화 | limit은 backing buffer size가 아니라 `writeBuffer_.size() - writeOffset_`인 logical unsent bytes를 지배합니다. |
| failure 또는 boundary | 초과 append는 기존 pending bytes를 변경하지 않고 `outbound queue limit exceeded` close state를 만듭니다. |
| 보장 / 비보장 | 보장: 한 connection의 unsent output에 hard bound와 observable failure result를 제공합니다.<br>비보장: `pending + byteCount <= limit` 덧셈은 size_t wrap 가능성이 있고 CRLF 추가의 두 단계 경계도 충분히 증명되지 않았습니다. |
| 다음 관련 변화 | 881e59734a9a가 subtraction predicate와 send guard로 수정하고 f34ab135c546가 deterministic하게 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d7d85e518177` | `src/Connection.cpp` | `pendingBytes/canAppendPending` | logical unsent byte 기준 admission |
| `d7d85e518177` | `src/Server.cpp` | `sendTo/queueRawTo` | queue failure를 application까지 bool로 전달 |

- 조사 방법: GitHub에서 `d7d85e518177`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `adb49d9466e4` — `feat(server): 최대 연결 수 제한`. 원문 역할은 “Bounds the number of live connections.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `adb49d9466e4` — `feat(server): 최대 연결 수 제한`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | RESILIENCE, EVENT_IO |
| 원문 확정 역할 | Bounds the number of live connections. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Bounds the number of live connections.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | accept 직후 현재 connection count와 configured maximum을 비교해 초과 fd를 event registration/map ownership 이전에 거부하고 닫았습니다. |
| 확인한 변경 파일·symbol | `include/Server.hpp — maxConnections config/metrics`<br>`src/Server.cpp — acceptReadyClients, rejectReadyClient`<br>`src/RuntimeConfig.cpp — --max-connections` |
| 핵심 state / branch | 거부된 raw fd는 임시 accept scope가 닫고 Server connection map이나 application registry에 들어가지 않습니다. |
| failure handling | limit 도달은 server-level exception이 아니라 해당 incoming connection만 drop/reject하는 정상 보호 branch입니다. |
| 보장과 다음 연결 | live transport connection 수가 명시된 최대값을 넘지 않습니다. e05e35ca7da9가 live/accepted/rejected 수를 metrics로 노출하고 contract test가 public option을 검증합니다. |
| 이 시점의 한계 | listen backlog와 OS fd limit, 순간 accept rate 자체의 보장은 아닙니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `adb49d9466e4` | `src/Server.cpp` | `acceptReadyClients/rejectReadyClient` | map 편입 전 connection-limit 판정과 raw fd close |

- 조사 방법: GitHub에서 `adb49d9466e4`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `e05e35ca7da9` — `feat(metrics): 서버 실행 지표 조회 기능 추가`. 원문 역할은 “Makes connection, command, message, queue-drop, and rate-limit state queryable.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `e05e35ca7da9` — `feat(metrics): 서버 실행 지표 조회 기능 추가`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | OBSERVABILITY |
| 원문 확정 역할 | Makes connection, command, message, queue-drop, and rate-limit state queryable. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Makes connection, command, message, queue-drop, and rate-limit state queryable.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | Server transport metrics와 IrcApplication metrics를 각각 authoritative transition에서 누적하고 `METRICS` command가 fixed-order `key=value` 값을 반환하도록 했습니다. |
| 확인한 변경 파일·symbol | `include/Server.hpp — Metrics`<br>`src/IrcApplication.hpp — AppMetrics`<br>`src/IrcApplication.cpp/ApplicationSupport.cpp — counters and METRICS` |
| 핵심 state / branch | live connection 같은 gauge는 server container에서 읽고 event count는 실제 transition 직후 증가해 별도 mutable source of truth를 만들지 않습니다. |
| failure handling | queue rejection/rate limit/idle timeout도 성공 경로와 분리된 counter로 기록됩니다. |
| 보장과 다음 연결 | 운영 보호 결과와 command/message activity를 query 가능한 stable field set으로 노출합니다. c34aa18f89af가 같은 authoritative transitions를 structured log로 남기고 e5e6c57db80d가 ordering/shape를 고정합니다. |
| 이 시점의 한계 | 지표는 in-process 누적값이며 persistent monitoring/storage나 latency 분포를 제공하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e05e35ca7da9` | `include/Server.hpp` | `Server::Metrics` | transport transition counter owner |
| `e05e35ca7da9` | `src/IrcApplication.cpp` | `METRICS handling` | server/app 수치를 고정 순서 response로 구성 |

- 조사 방법: GitHub에서 `e05e35ca7da9`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `c34aa18f89af` — `feat(log): 연결 상태와 실행 지표 기록`. 원문 역할은 “Records structured lifecycle, protection, and aggregate metrics events.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `c34aa18f89af` — `feat(log): 연결 상태와 실행 지표 기록`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | OBSERVABILITY, LIFECYCLE |
| 원문 확정 역할 | Records structured lifecycle, protection, and aggregate metrics events. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Records structured lifecycle, protection, and aggregate metrics events.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | 공백을 underscore로 정규화한 `key=value` structured log helper를 만들고 connect/register/disconnect/rate-limit/timeout/metrics/shutdown의 확정 시점에 event를 기록했습니다. |
| 확인한 변경 파일·symbol | `src/ApplicationSupport.cpp/IrcApplication.cpp — structured log helpers and call sites` |
| 핵심 state / branch | 로그는 state를 소유하지 않고 authoritative mutation 직후 그 결과를 직렬화합니다. |
| failure handling | 사용자 제공 reason/nick의 공백은 한 field를 깨지 않도록 escape/normalize되고 missing state에서는 추측한 identity를 기록하지 않습니다. |
| 보장과 다음 연결 | lifecycle 및 protection event의 machine-readable ordering을 관찰할 수 있습니다. dd04279c47fd의 shutdown ordering과 e5e6c57db80d contract test가 final log sequence를 검증합니다. |
| 이 시점의 한계 | logger failure 격리나 durability/rotation은 이 프로젝트 범위에서 보장하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `c34aa18f89af` | `src/ApplicationSupport.cpp` | `log event/value sanitization` | 공백 없는 key=value record 생성 |
| `c34aa18f89af` | `src/IrcApplication.cpp` | `lifecycle/protection log call sites` | 확정 mutation 뒤 event 기록 |

- 조사 방법: GitHub에서 `c34aa18f89af`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `dd04279c47fd` — `feat(shutdown): 종료 전 송신 대기열 처리`. 원문 역할은 “Replaces immediate signal-time stop with application shutdown and output draining.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.8 `dd04279c47fd` — `feat(shutdown): 종료 전 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | LIFECYCLE, RESILIENCE, RISK |
| 원문 확정 역할 | Replaces immediate signal-time stop with application shutdown and output draining. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Replaces immediate signal-time stop with application shutdown and output draining.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | SIGINT/SIGTERM이 server를 즉시 멈춰 queued protocol output과 disconnect cleanup을 건너뛸 수 있었습니다. |
| 주요 문제와 설계 판단 | signal handler는 async-signal-safe flag만 변경하고 normal loop가 `IrcApplication::shutdown()`으로 각 client에 final ERROR를 queue·close 요청한 뒤 최대 8회, 각 50ms poll로 pending output을 drain하도록 했습니다. 그 후 callback을 detach하고 server를 stop했습니다. |
| 변경 파일·symbol | `src/main.cpp — signal flag, main loop shutdown/drain order`<br>`src/IrcApplication.cpp — shutdown` |
| state / ownership 변화 | shutdown transition은 signal context가 아니라 main control flow가 소유하며 connection은 final queue가 비거나 bounded drain budget가 끝날 때까지 Server map에 남습니다. |
| failure 또는 boundary | 읽지 않는 peer가 있어도 8×50ms 이후 teardown으로 진행해 무한 대기를 막습니다. queue 실패는 해당 connection cleanup으로 즉시 수렴할 수 있습니다. |
| 보장 / 비보장 | 보장: 종료 전에 final protocol output을 시도하면서 process termination latency를 제한합니다.<br>비보장: 모든 peer가 ERROR를 실제 수신한다는 보장은 없고 drain budget 이후 unsent bytes는 teardown됩니다. |
| 다음 관련 변화 | e5e6c57db80d가 signal→ERROR→close/log ordering을 public process boundary에서 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `dd04279c47fd` | `src/main.cpp` | `signal handler and shutdown sequence` | signal flag → application shutdown → bounded poll drain → callback detach/stop |
| `dd04279c47fd` | `src/IrcApplication.cpp` | `IrcApplication::shutdown` | client snapshot에 ERROR queue와 close request |

- 조사 방법: GitHub에서 `dd04279c47fd`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “Locks the resulting CLI, wire, timeout, rate, metric, shutdown, and log-order contracts.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.9 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Locks the resulting CLI, wire, timeout, rate, metric, shutdown, and log-order contracts. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Locks the resulting CLI, wire, timeout, rate, metric, shutdown, and log-order contracts.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact frame/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable contract suite를 추가해 process startup 오류, runtime options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 assertion으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| state / ownership 변화 | test가 server process, sockets, manifest, logs와 expected frames를 소유하며 dynamic fd/token 같은 값만 regex로 허용합니다. |
| failure 또는 boundary | invalid/missing option과 runtime timeout/rate/queue/shutdown boundary를 subprocess return code, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: public CLI·wire·shutdown·observability contract의 성공/실패 shape와 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update failure, pointer lifetime, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level deterministic failure를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 frame은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown public ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 문서화된 CLI syntax, IRC frame/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 public boundary와 일치합니다. |
| 재현 failure / boundary | missing/invalid args, registration/heartbeat/rate/output limits, exact numerics, signal shutdown과 final ERROR입니다. |
| test technique | compiled server subprocess + real loopback TCP + exact/limited-regex assertions + captured logs/manifest입니다. |
| 통과하는 production path | argv parser → Server/IrcApplication runtime policy → wire output → signal flag → shutdown/drain → logs입니다. |
| 이 test가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 frame과 ordering을 정확히 고정합니다. |
| 이 test가 증명하지 않는 것 | 모든 syscall/event failure, memory safety, formal latency·fairness, 내부 lifetime 안전성은 증명하지 않습니다. |
| test 성격 | exact process and wire contract integration |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log contract가 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make target
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| registration deadline | `c4df44554866` | onTick snapshot maintenance | 미등록 client만 deadline 뒤 451+close로 수렴합니다. | e5e6c57db80d public contract |
| heartbeat liveness | `764361c52b2a` | initial idle PING/PONG | 초기에는 wall-clock/any-PONG이라는 부족함이 있습니다. | 3f2b3ae1d3f9/0c76aad19579가 수정·검증 |
| command-rate bound | `9e2b214f9227` | per-client timestamp window | 초과 actor만 439+close 처리합니다. | e05e35ca7da9/c34aa18f89af 관찰, e5e6c57db80d contract |
| output/connection bounds | `d7d85e518177` | adb49d9466e4 | logical pending bytes와 live connection 수를 제한합니다. | 881e59734a9a/f34ab135c546 arithmetic fix/test |
| observable controlled shutdown | `e05e35ca7da9` | c34aa18f89af, dd04279c47fd | metrics/log와 bounded final-output drain을 연결합니다. | e5e6c57db80d exact process contract |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| functional server는 무제한 resource를 감당 | 미등록·slow reader·connection flood가 state/memory/fd 점유 | c4df44554866 / d7d85e518177 / adb49d9466e4 | 각 resource를 connection-scoped bound로 제한 | e5e6c57db80d |
| wall-clock 및 any PONG이면 충분 | clock 이동·forged PONG으로 liveness 오판 | 3f2b3ae1d3f9 | steady_clock + exact token | 0c76aad19579 |
| signal에서 즉시 stop | queued final output과 cleanup 유실 | dd04279c47fd | flag-only handler와 bounded normal-flow drain | e5e6c57db80d |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| registration/rate/heartbeat state | 없음 | ClientState + IrcApplication maintenance | c4df44554866 → 9e2b214f9227 | path/symbol은 위 commit 증거표에 연결했습니다. |
| unsent-output bound | 무제한 Connection queue | Connection maxPendingBytes | d7d85e518177 | path/symbol은 위 commit 증거표에 연결했습니다. |
| live-connection bound | listener가 모두 수락 | Server accept policy | adb49d9466e4 | path/symbol은 위 commit 증거표에 연결했습니다. |
| transport/app metrics | 관찰 불가 | Server::Metrics / AppMetrics | e05e35ca7da9 | path/symbol은 위 commit 증거표에 연결했습니다. |
| shutdown orchestration | signal-time immediate stop | main normal control flow + IrcApplication::shutdown | dd04279c47fd | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: transport를 받아 놓고 등록을 끝내지 않는 client를 제거할 deadline이 없었습니다.
- 마지막 commit `e5e6c57db80d` 시점의 상태: public CLI·wire·shutdown·observability contract의 성공/실패 shape와 ordering을 broad substring 수준보다 정확히 고정합니다.
- Thread 안에서 강화된 핵심 invariant: registration deadline, heartbeat liveness, command-rate bound, output/connection bounds, observable controlled shutdown.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: rare syscall return, event add/update failure, pointer lifetime, formal fairness는 직접 주입하지 않습니다. b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level deterministic failure를 추가합니다.
- 최종 설명: `c4df44554866`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `e5e6c57db80d`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| connection/command event | c4df44554866 / 9e2b214f9227 | `Server callbacks/IrcApplication` | ClientState timestamps/window 갱신 | deadline/rate 초과는 response+requestClose |
| output admission | d7d85e518177 | `IrcApplication send helper` | Server.sendTo → Connection queue | limit 초과는 false+connection close |
| accept admission | adb49d9466e4 | `Server::acceptReadyClients` | count 확인 후 map/backend 편입 | limit 도달 raw fd는 편입 전 close |
| observability | e05e35ca7da9 / c34aa18f89af | `authoritative transition` | counter/gauge와 key=value log | 실패 branch도 별도 metric/event |
| signal shutdown | dd04279c47fd | `signal flag/main loop` | ERROR queue → close request → 8×50ms drain → detach/stop | budget 만료 후 unsent bytes는 teardown |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 04-operational-protections-and-controlled-shutdown.md =====

===== BEGIN FILE: 05-strict-runtime-configuration-boundaries.md =====
# Thread 05 — Strict runtime configuration boundaries

부제: 엄격한 런타임 구성 경계

## 1. Thread 목표

CLI 문자열을 실제 destination type과 operational bound에 정확히 맞는 값으로 바꾸는 경계를 복원하고, permissive C conversion과 narrowing의 결함이 digit-by-digit parser 및 cross-width regression으로 어떻게 수정되는지 학습합니다.

Source에서 확정된 significance:

> The public configuration surface initially relied on conversion functions whose accepted syntax and intermediate width did not exactly match the documented contract. The fix moves range checking into digit-by-digit accumulation against the destination type, and the follow-up tests preserve that cross-platform boundary without treating the test itself as a second architectural decision.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 `RuntimeConfig`와 `Server::Config`의 책임은 어떻게 구분되는가?
- `strtol`/`strtoul`의 accepted syntax와 intermediate width가 public contract와 어디서 어긋나는가?
- 곱셈·덧셈 전에 overflow를 증명하는 비교식은 무엇인가?
- port, timeout, `std::size_t` option이 각 destination maximum을 직접 사용하는가?
- 32/64-bit 모두에서 first-unrepresentable size를 test가 어떻게 계산하는가?

### Source와 직접 연결되는 invariant

- accepted numeric option은 실제 socket, buffer, connection, timing limit을 지배하는 destination type에 정확히 representable해야 합니다.
- public CLI syntax는 non-empty ASCII decimal이라는 확정 계약과 일치해야 하며 sign/whitespace를 암묵적으로 허용해서는 안 됩니다.

### Source와 직접 연결되는 engineering difficulty

- external input을 intermediate C integer semantics에 맡기지 않고 target-width에서 overflow 없이 검증하는 문제.
- host width가 달라도 같은 representability invariant를 검증하는 test boundary 설계.

## 3. 완료 기준

- 초기 parser가 허용/거부하는 입력을 해당 SHA 코드로 정리했습니다.
- old conversion+narrowing과 new bounded accumulation을 숫자 예제로 비교했습니다.
- sign, whitespace, exact maximum, maximum+1, extremely long decimal의 control flow를 설명합니다.
- CLI contract test와 boundary regression의 증명 범위를 구분했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `52b6f1ce8f0f` | `feat(config): 기본 실행 인자 해석 모듈 구성` | B | BUILD, RESILIENCE | Establishes the initial RuntimeConfig parsing boundary. |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | Records exact CLI success and failure behavior as an executable contract. |
| 3 | `b6c10bc51937` | `fix(config): 서버 크기 옵션을 오버플로 없이 해석` | A | DEBUG, RESILIENCE, RISK | Replaces permissive C conversion with overflow-safe, target-width decimal parsing. |
| 4 | `5d1286620994` | `test(config): 크기 옵션 경계와 오류 입력 검증` | B | VERIFICATION, RESILIENCE, RISK | Covers signs, whitespace, port narrowing, timeout bounds, size_t overflow, and rate-count overflow. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `52b6f1ce8f0f` — `feat(config): 기본 실행 인자 해석 모듈 구성`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | BUILD, RESILIENCE |
| 원문 확정 역할 | Establishes the initial RuntimeConfig parsing boundary. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Establishes the initial RuntimeConfig parsing boundary.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | `RuntimeConfig`에 rate, idle/ping, registration timeout 기본값을 두고 `<port> <password>`를 필수 인자로 해석했습니다. 초기 port parser는 `strtol`, end pointer, 1..65535 range를 확인하고 unknown extra argument를 usage error로 처리했습니다. |
| 확인한 변경 파일·symbol | `src/RuntimeConfig.hpp — RuntimeConfig fields/defaults`<br>`src/RuntimeConfig.cpp — parseRuntimeConfig, initial port parser`<br>`src/main.cpp — Server::Config mapping` |
| 핵심 state / branch | application policy는 RuntimeConfig가, bind address/port와 transport limit은 Server::Config가 소유하도록 책임을 분리합니다. |
| failure handling | empty/trailing garbage, zero/negative/out-of-port-range와 잘못된 argument count는 server 시작 전에 diagnostic과 실패 return으로 끝납니다. |
| 보장과 다음 연결 | public argv를 typed runtime state로 바꾸는 단일 parsing boundary와 최소 CLI contract가 생깁니다. e5e6c57db80d가 exact CLI behavior를 기록하고 b6c10bc51937가 destination-width decimal parser로 교체합니다. |
| 이 시점의 한계 | `strtol`은 leading whitespace와 sign을 허용하는 C lexical semantics를 가지며 optional flags는 이 scaffold 단계에서 아직 실제 config를 모두 변경하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `52b6f1ce8f0f` | `src/RuntimeConfig.cpp` | `port parsing and parseRuntimeConfig` | complete consumption/range 검사와 required argv shape |
| `52b6f1ce8f0f` | `src/RuntimeConfig.hpp` | `RuntimeConfig` | application policy와 transport config의 책임 분리 |

- 조사 방법: GitHub에서 `52b6f1ce8f0f`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “Records exact CLI success and failure behavior as an executable contract.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Records exact CLI success and failure behavior as an executable contract. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Records exact CLI success and failure behavior as an executable contract.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact frame/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable contract suite를 추가해 process startup 오류, runtime options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 assertion으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| state / ownership 변화 | test가 server process, sockets, manifest, logs와 expected frames를 소유하며 dynamic fd/token 같은 값만 regex로 허용합니다. |
| failure 또는 boundary | invalid/missing option과 runtime timeout/rate/queue/shutdown boundary를 subprocess return code, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: public CLI·wire·shutdown·observability contract의 성공/실패 shape와 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update failure, pointer lifetime, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level deterministic failure를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 frame은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown public ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 문서화된 CLI syntax, IRC frame/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 public boundary와 일치합니다. |
| 재현 failure / boundary | missing/invalid args, registration/heartbeat/rate/output limits, exact numerics, signal shutdown과 final ERROR입니다. |
| test technique | compiled server subprocess + real loopback TCP + exact/limited-regex assertions + captured logs/manifest입니다. |
| 통과하는 production path | argv parser → Server/IrcApplication runtime policy → wire output → signal flag → shutdown/drain → logs입니다. |
| 이 test가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 frame과 ordering을 정확히 고정합니다. |
| 이 test가 증명하지 않는 것 | 모든 syscall/event failure, memory safety, formal latency·fairness, 내부 lifetime 안전성은 증명하지 않습니다. |
| test 성격 | exact process and wire contract integration |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log contract가 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make target
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `b6c10bc51937` — `fix(config): 서버 크기 옵션을 오버플로 없이 해석`. 원문 역할은 “Replaces permissive C conversion with overflow-safe, target-width decimal parsing.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `b6c10bc51937` — `fix(config): 서버 크기 옵션을 오버플로 없이 해석`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | DEBUG, RESILIENCE, RISK |
| 원문 확정 역할 | Replaces permissive C conversion with overflow-safe, target-width decimal parsing. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Replaces permissive C conversion with overflow-safe, target-width decimal parsing.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | numeric options가 `strtol/strtoul`의 permissive syntax와 host intermediate width를 거친 뒤 port·timeout·size_t로 narrowing되어 destination contract보다 넓은 입력을 받아들일 수 있었습니다. |
| 주요 문제와 설계 판단 | ASCII digit만 한 자리씩 누적하는 `parseUnsignedDecimal<Unsigned>()`를 도입하고 각 digit 전에 `value > max/10 \|\| (value == max/10 && digit > max%10)`을 검사했습니다. 각 option은 실제 destination type의 maximum을 template argument/limit로 사용했습니다. |
| 변경 파일·symbol | `src/RuntimeConfig.cpp — parseUnsignedDecimal and option parsers` |
| state / ownership 변화 | 외부 문자열은 intermediate signed/unsigned long이 아니라 최종 target-width에서 직접 representability를 검증받습니다. |
| failure 또는 boundary | empty, `+/-`, whitespace, non-digit, exact maximum 초과, 매우 긴 decimal은 곱셈·덧셈 전에 거부되어 wrap이나 narrowing이 일어나지 않습니다. |
| 보장 / 비보장 | 보장: accepted option은 non-empty ASCII decimal이고 실제 port/timeout/size_t/rate-count destination에 표현 가능합니다.<br>비보장: semantic relation인 idle/ping 조합 등은 별도 policy 검증이며 이 helper는 decimal representability만 보장합니다. |
| 다음 관련 변화 | 5d1286620994가 플랫폼 width를 계산한 first-unrepresentable values와 lexical 오류를 regression으로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `b6c10bc51937` | `src/RuntimeConfig.cpp` | `parseUnsignedDecimal<Unsigned>` | `maximum / 10`과 `maximum % 10` pre-check로 overflow 전 거부 |
| `b6c10bc51937` | `src/RuntimeConfig.cpp` | `port/size/rate option parsing` | 각 destination maximum을 직접 전달 |

- 조사 방법: GitHub에서 `b6c10bc51937`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | C conversion이 complete consumption과 errno를 확인하면 public decimal contract 및 모든 destination width에도 충분하다는 가정이었습니다. |
| 실제 failure / risk | leading sign/whitespace가 허용되고 wider intermediate를 좁은 port·count·size_t로 cast하면 플랫폼에 따라 wrap/narrowing할 수 있었습니다. |
| root cause | lexical contract와 range check가 최종 destination type이 아니라 C library intermediate semantics에 묶여 있었습니다. |
| 수정된 invariant / decision | ASCII digit accumulation을 최종 Unsigned maximum에 대해 매 step pre-check하고 성공한 값만 config에 저장합니다. |
| regression evidence | 5d1286620994가 sign/whitespace, 65536, huge timeout, size_t max+1, rate-count overflow를 실행 경계로 고정합니다. |

#### 다음 연결

- 다음 Thread commit: `5d1286620994` — `test(config): 크기 옵션 경계와 오류 입력 검증`. 원문 역할은 “Covers signs, whitespace, port narrowing, timeout bounds, size_t overflow, and rate-count overflow.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `5d1286620994` — `test(config): 크기 옵션 경계와 오류 입력 검증`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | VERIFICATION, RESILIENCE, RISK |
| 원문 확정 역할 | Covers signs, whitespace, port narrowing, timeout bounds, size_t overflow, and rate-count overflow. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Covers signs, whitespace, port narrowing, timeout bounds, size_t overflow, and rate-count overflow.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | contract test에 sign/whitespace, port 65536, timeout upper bound 초과, 매우 긴 decimal, runtime에서 계산한 `std::size_t` maximum의 다음 값, rate-count destination overflow cases를 추가했습니다. |
| 확인한 변경 파일·symbol | `tests/irc_contract.py — numeric boundary and invalid CLI cases` |
| 핵심 state / branch | test는 Python integer로 host/process가 노출한 size width에 맞는 first-unrepresentable decimal을 구성하고 각 subprocess failure를 manifest에 기록합니다. |
| failure handling | 각 invalid input은 server가 listen하기 전에 nonzero exit와 정해진 diagnostic으로 거부되어야 합니다. |
| 보장과 다음 연결 | 32/64-bit 차이와 무관하게 destination representability 및 strict ASCII-decimal syntax가 유지됩니다. 416efc91e580의 Linux/macOS matrix가 서로 다른 platform environment에서 같은 contract suite를 반복합니다. |
| 이 시점의 한계 | parser 내부의 모든 template instantiation을 unit level에서 직접 호출하는 테스트는 아니며 process CLI 경계만 통과합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5d1286620994` | `tests/irc_contract.py` | `configuration boundary cases` | sign/whitespace/port/timeout/rate/size overflow를 process 실패로 검증 |

- 조사 방법: GitHub에서 `5d1286620994`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | accepted numeric CLI value가 strict ASCII decimal이고 실제 destination type에 표현 가능해야 합니다. |
| 재현 failure / boundary | `+1`, `-1`, leading/trailing whitespace, port 65536, timeout 상한 초과, size_t max+1, rate-count overflow입니다. |
| test technique | real executable CLI subprocess와 Python arbitrary-precision integer로 platform first-unrepresentable 값을 생성하는 boundary regression입니다. |
| 통과하는 production path | argv → `parseUnsignedDecimal` → option-specific bound → RuntimeConfig/Server::Config 또는 startup rejection입니다. |
| 이 test가 증명하는 것 | lexical strictness와 target-width overflow rejection이 public CLI에서 유지됨을 증명합니다. |
| 이 test가 증명하지 않는 것 | 모든 internal caller나 non-CLI serialization 경로, semantic option 조합 전체는 증명하지 않습니다. |
| test 성격 | cross-width boundary regression |
| 막는 회귀 | C conversion으로 회귀하거나 narrowing이 32/64-bit 중 한쪽에서만 통과하는 문제를 막습니다. |

실행 기록:

- 실행 SHA: `5d1286620994`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| runtime parsing boundary | `52b6f1ce8f0f` | e5e6c57db80d | RuntimeConfig와 Server::Config 책임 및 public CLI behavior를 고정합니다. | 초기 C conversion의 lexical/width 한계가 남음 |
| target-width representability | `b6c10bc51937` | digit-by-digit accumulation | destination maximum에 대해 mutation 전 overflow를 거부합니다. | 5d1286620994 cross-width regression |
| strict decimal syntax | `b6c10bc51937` | all numeric option parsers | sign/whitespace/non-digit를 허용하지 않습니다. | 5d1286620994 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| strtol/strtoul + cast면 충분 | sign/whitespace 허용과 intermediate-width narrowing | b6c10bc51937 | ASCII digit + destination max pre-check | 5d1286620994 |
| 한 platform boundary만 고정 | 32/64-bit 중 한쪽에서만 overflow 재현 | 5d1286620994 | runtime first-unrepresentable decimal 계산 | 416efc91e580 OS matrix |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| application runtime policy | 없음 | RuntimeConfig | 52b6f1ce8f0f | path/symbol은 위 commit 증거표에 연결했습니다. |
| transport bind/size policy | 문자열 argv | Server::Config typed fields | 52b6f1ce8f0f → b6c10bc51937 | path/symbol은 위 commit 증거표에 연결했습니다. |
| numeric representability check | C library conversion | parseUnsignedDecimal<Unsigned> | b6c10bc51937 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: 실행 인자에서 transport와 application runtime policy를 분리해 보관·검증하는 모듈이 없었습니다.
- 마지막 commit `5d1286620994` 시점의 상태: 32/64-bit 차이와 무관하게 destination representability 및 strict ASCII-decimal syntax가 유지됩니다.
- Thread 안에서 강화된 핵심 invariant: runtime parsing boundary, target-width representability, strict decimal syntax.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: parser 내부의 모든 template instantiation을 unit level에서 직접 호출하는 테스트는 아니며 process CLI 경계만 통과합니다. 416efc91e580의 Linux/macOS matrix가 서로 다른 platform environment에서 같은 contract suite를 반복합니다.
- 최종 설명: `52b6f1ce8f0f`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `5d1286620994`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| argv shape | 52b6f1ce8f0f | `main/parser` | required port/password와 option 분리 | missing/unknown은 usage failure |
| decimal scan | b6c10bc51937 | `option parser` | ASCII digit를 target Unsigned에 누적 | digit 전 max/10·max%10 검사에서 reject |
| typed assignment | b6c10bc51937 | `parseRuntimeConfig` | RuntimeConfig/Server::Config에 성공 값 저장 | 실패 값은 config를 변경하지 않음 |
| regression | 5d1286620994 | `real CLI subprocess` | exact max/first-unrepresentable 및 lexical cases | startup 전에 diagnostic/nonzero exit |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 05-strict-runtime-configuration-boundaries.md =====

===== BEGIN FILE: 06-heartbeat-liveness-correctness.md =====
# Thread 06 — Heartbeat liveness correctness

부제: 하트비트 생존성 정확성

## 1. Thread 목표

초기 idle PING/PONG state machine의 실제 보장과 부족함을 분리하고, monotonic clock과 exact outstanding token correlation로 liveness invariant를 복구한 뒤 forged-response regression으로 고정합니다.

Source에서 확정된 significance:

> The initial feature supplied liveness policy but relied on wall-clock time and accepted any PONG as evidence for the outstanding heartbeat. The correction restores the actual invariant: deadline calculation must be monotonic, and only the response corresponding to the server's current challenge can preserve the connection. The regression test makes the forged-response failure deterministic.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- idle, awaitingPong, pingSentAt, token 상태는 어떤 전이로 움직이는가?
- registration timeout, heartbeat deadline, idle probe의 maintenance 우선순위는 무엇인가?
- ordinary activity와 outstanding challenge completion은 왜 다른 상태인가?
- wall-clock 대신 `steady_clock`이 필요한 failure scenario는 무엇인가?
- matching PONG과 unrelated PONG을 test가 어떻게 deterministic하게 구분하는가?

### Source와 직접 연결되는 invariant

- heartbeat deadline은 monotonic elapsed time으로 계산되어야 합니다.
- idle client는 outstanding probe가 없거나 하나의 식별 가능한 probe와 send time을 가져야 합니다.
- 현재 outstanding token과 정확히 일치하는 PONG만 wait state를 해제해야 합니다.

### Source와 직접 연결되는 engineering difficulty

- system clock correction과 protocol traffic을 liveness proof로 잘못 결합하지 않는 state-machine 설계.
- real time/socket 통합 test와 exact forged-token regression의 역할을 분리하는 문제.

## 3. 완료 기준

- 764361c52b2a의 state machine과 any-PONG/wall-clock 한계를 코드로 기록했습니다.
- test peer 자동응답이 broad smoke 안정화와 dedicated failure scenario를 모두 지원하는 구조를 설명합니다.
- 3f2b3ae1d3f9의 clock type, token generation/storage/validation/clear 순서를 복원했습니다.
- 0c76aad19579가 matching response와 forged response를 각각 검증하는 이유를 설명합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `764361c52b2a` | `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기` | A | RESILIENCE, LIFECYCLE, RISK | Introduces heartbeat PING, PONG waiting, and timeout state. |
| 2 | `d710f29f38a4` | `test(client): 서버 PING에 응답하는 검사 클라이언트 구현` | B | VERIFICATION, IRC_PROTOCOL | Adds automatic PONG behavior to the test peer. |
| 3 | `f313e707474f` | `test(client): 유휴 연결의 PING·PONG 흐름 검증` | B | VERIFICATION, RESILIENCE | Verifies the initial positive PING/PONG flow. |
| 4 | `3f2b3ae1d3f9` | `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리` | A | DEBUG, RESILIENCE, RISK | Moves timing to steady_clock and correlates PONG with the exact outstanding token. |
| 5 | `0c76aad19579` | `test(heartbeat): PONG 토큰과 시간 경계 검증` | A | VERIFICATION, RESILIENCE, RISK | Proves matching PONG clears the deadline and forged PONG does not. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `764361c52b2a` — `feat(heartbeat): 유휴 연결에 PING을 보내고 응답 대기`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | RESILIENCE, LIFECYCLE, RISK |
| 원문 확정 역할 | Introduces heartbeat PING, PONG waiting, and timeout state. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Introduces heartbeat PING, PONG waiting, and timeout state.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다. |
| 주요 문제와 설계 판단 | 모든 received line에서 `lastActivityAt`을 갱신하고, idle client에 PING token을 queue한 뒤 `awaitingPong`과 `lastPingAt`을 기록했습니다. deadline을 넘으면 ERROR를 queue하고 close를 요청했으며 PONG handler가 wait를 해제했습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — awaitingPong, lastActivityAt, lastPingAt`<br>`src/IrcApplication.cpp — onLine, maintainClient, handlePong`<br>`src/RuntimeConfig.cpp — idle/ping options` |
| state / ownership 변화 | maintenance 우선순위는 registration timeout → registered 여부 → awaiting PONG timeout → idle probe입니다. |
| failure 또는 boundary | 초기 구현은 `std::time` wall clock을 사용하고 outstanding token을 저장하지 않아 어떤 PONG도 wait를 해제합니다. |
| 보장 / 비보장 | 보장: idle probe와 timeout이라는 liveness state machine의 최초 정책을 제공합니다.<br>비보장: clock correction과 forged/unrelated PONG에 안전하지 않습니다. |
| 다음 관련 변화 | d710f29f38a4/f313e707474f가 양성 흐름을 먼저 검증하고 3f2b3ae1d3f9/0c76aad19579가 invariant를 수정·고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `764361c52b2a` | `src/IrcApplication.cpp` | `maintainClient` | registration → ping timeout → idle PING 순서 |
| `764361c52b2a` | `src/IrcApplication.cpp` | `handlePong` | 초기 버전은 token 검사 없이 awaiting 상태 해제 |

- 조사 방법: GitHub에서 `764361c52b2a`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d710f29f38a4` — `test(client): 서버 PING에 응답하는 검사 클라이언트 구현`. 원문 역할은 “Adds automatic PONG behavior to the test peer.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d710f29f38a4` — `test(client): 서버 PING에 응답하는 검사 클라이언트 구현`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | VERIFICATION, IRC_PROTOCOL |
| 원문 확정 역할 | Adds automatic PONG behavior to the test peer. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Adds automatic PONG behavior to the test peer.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | test peer에 `auto_pong` option을 추가하고 server `PING`의 parameter를 그대로 echo한 `PONG`을 자동 전송하도록 했습니다. |
| 확인한 변경 파일·symbol | `tools/irc_smoke_client.py — IrcPeer auto_pong and receive loop` |
| 핵심 state / branch | test peer가 수신 buffer를 parse하는 시점에 PING을 감지해 같은 socket으로 PONG을 보내며 scenario는 필요할 때 자동응답을 끌 수 있습니다. |
| failure handling | PING에 parameter가 없거나 malformed인 경우 임의 token을 만들지 않고 기존 assertion/timeout 경로가 실패를 드러냅니다. |
| 보장과 다음 연결 | broad test 시나리오가 heartbeat policy 때문에 우연히 종료되지 않고 production PING→PONG 경로를 통과합니다. f313e707474f가 짧은 timeout으로 양성 흐름을 전용 시나리오로 확인하고 0c76aad19579는 auto_pong을 꺼 forged case를 만듭니다. |
| 이 시점의 한계 | 자동응답은 matching PONG 양성 경로만 만들며 forged token이나 deadline correctness를 검증하지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d710f29f38a4` | `tools/irc_smoke_client.py` | `IrcPeer auto_pong receive handling` | server PING token을 같은 parameter의 PONG으로 자동 응답 |

- 조사 방법: GitHub에서 `d710f29f38a4`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 검사 client가 server heartbeat challenge에 protocol-compatible response를 보낼 수 있어야 합니다. |
| 재현 failure / boundary | server PING 수신과 token echo입니다. |
| test technique | real TCP peer helper의 automatic response seam입니다. |
| 통과하는 production path | socket receive buffer → PING line parse → PONG sendall → server PONG handler입니다. |
| 이 test가 증명하는 것 | test harness가 양성 heartbeat path를 자동으로 유지할 수 있음을 증명합니다. |
| 이 test가 증명하지 않는 것 | server가 exact token만 인정하거나 deadline을 올바르게 계산하는지는 증명하지 않습니다. |
| test 성격 | test-harness capability |
| 막는 회귀 | heartbeat 도입 후 unrelated smoke가 idle timeout으로 불안정해지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `d710f29f38a4`
- repository에 정의된 실행 명령: 이 helper를 사용하는 smoke/contract scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `f313e707474f` — `test(client): 유휴 연결의 PING·PONG 흐름 검증`. 원문 역할은 “Verifies the initial positive PING/PONG flow.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `f313e707474f` — `test(client): 유휴 연결의 PING·PONG 흐름 검증`

| 항목 | 값 |
| --- | --- |
| Importance | B |
| Tags | VERIFICATION, RESILIENCE |
| 원문 확정 역할 | Verifies the initial positive PING/PONG flow. |
| 학습 깊이 | Thread 흐름에서 맡는 구체적 구현 역할과 핵심 state/branch만 기록합니다. |

#### Source에서 확정된 역할

> Verifies the initial positive PING/PONG flow.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| Thread 내 구현 역할 | test scenario가 짧은 heartbeat 옵션으로 server를 실행하고 idle peer가 PING을 받은 뒤 자동 PONG하며 이후 command/PING response가 계속 진행되는지 확인했습니다. |
| 확인한 변경 파일·symbol | `tools/irc_smoke_client.py or tests scenario — idle heartbeat positive flow` |
| 핵심 state / branch | real monotonic test deadline과 socket timeout이 scenario lifetime을 제한하고 peer connection은 PONG 뒤에도 유지됩니다. |
| failure handling | expected server PING 또는 후속 response가 deadline 안에 없거나 connection이 닫히면 test가 실패합니다. |
| 보장과 다음 연결 | 초기 heartbeat의 positive integration path가 listener/event loop/parser/application/output queue를 거쳐 동작합니다. 3f2b3ae1d3f9가 실제 liveness invariant를 고치고 0c76aad19579가 matching/forged response를 분리합니다. |
| 이 시점의 한계 | 초기 implementation의 wall-clock 사용과 any-PONG acceptance는 이 양성 test로 드러나지 않습니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f313e707474f` | `tools/irc_smoke_client.py` | `idle heartbeat scenario` | 짧은 idle timeout에서 PING→automatic PONG→connection progress |

- 조사 방법: GitHub에서 `f313e707474f`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | idle probe에 정상 PONG을 보낸 connection은 ping deadline으로 제거되지 않아야 합니다. |
| 재현 failure / boundary | 짧은 idle interval과 positive PING/PONG round trip입니다. |
| test technique | real process/socket timing integration with automatic PONG입니다. |
| 통과하는 production path | onTick idle branch → send PING → test peer auto-PONG → handlePong → later progress입니다. |
| 이 test가 증명하는 것 | 정상 heartbeat round trip과 connection 유지가 통합 환경에서 동작합니다. |
| 이 test가 증명하지 않는 것 | forged token rejection, wall-clock rollback, exact deadline ordering은 증명하지 않습니다. |
| test 성격 | positive liveness integration |
| 막는 회귀 | heartbeat feature가 정상 응답 client도 timeout시키는 기본 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f313e707474f`
- repository에 정의된 실행 명령: 해당 smoke/contract heartbeat scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `3f2b3ae1d3f9` — `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`. 원문 역할은 “Moves timing to steady_clock and correlates PONG with the exact outstanding token.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `3f2b3ae1d3f9` — `fix(heartbeat): 단조 시계와 토큰으로 응답 대기 상태 관리`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | DEBUG, RESILIENCE, RISK |
| 원문 확정 역할 | Moves timing to steady_clock and correlates PONG with the exact outstanding token. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Moves timing to steady_clock and correlates PONG with the exact outstanding token.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | heartbeat와 rate/registration timing이 wall clock을 사용하고, `awaitingPong`만 있어 unrelated PONG도 outstanding challenge를 완료할 수 있었습니다. |
| 주요 문제와 설계 판단 | client timing을 `std::chrono::steady_clock` 기반 `MonotonicTime`으로 바꾸고, 증가하는 heartbeat sequence로 token을 만들었습니다. PING 전 `awaitingPong`, `pendingPongToken`, `lastPingAt`을 저장하고 PONG parameter가 exact token과 일치할 때만 wait state/token을 지웠습니다. |
| 변경 파일·symbol | `src/ClientRegistry.hpp — MonotonicTime, pendingPongToken`<br>`src/IrcApplication.hpp/.cpp — heartbeat sequence, maintainClient, handlePong` |
| state / ownership 변화 | 한 client는 outstanding probe가 없거나 정확히 하나의 token과 send time을 가지며 ordinary activity는 그 challenge를 완료하지 않습니다. |
| failure 또는 boundary | system clock correction은 elapsed deadline에 영향을 주지 않고, parameter 누락·추가·불일치 PONG은 liveness proof가 아니므로 state를 그대로 둡니다. |
| 보장 / 비보장 | 보장: deadline이 monotonic elapsed time을 사용하고 현재 challenge와 정확히 상관된 response만 connection을 보존합니다.<br>비보장: network delay/clock scheduling 자체의 upper bound와 cryptographic token unpredictability는 보장하지 않습니다. |
| 다음 관련 변화 | 0c76aad19579가 matching response와 forged response를 deterministic한 real-socket 시나리오로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `3f2b3ae1d3f9` | `src/IrcApplication.cpp` | `maintainClient heartbeat branch` | token 생성과 awaiting/token/time mutation을 send 전에 설정 |
| `3f2b3ae1d3f9` | `src/IrcApplication.cpp` | `handlePong` | 정확히 하나의 parameter가 pending token과 같을 때만 clear |
| `3f2b3ae1d3f9` | `src/ClientRegistry.hpp` | `steady_clock-based fields` | wall-clock과 분리된 elapsed deadline |

- 조사 방법: GitHub에서 `3f2b3ae1d3f9`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | 최근 activity나 아무 PONG이면 현재 peer가 server heartbeat에 응답했다고 간주해도 된다는 가정이었습니다. |
| 실제 failure / risk | clock rollback은 timeout을 지연시키고, forged/unrelated PONG은 실제 challenge를 보지 않은 client도 wait state를 해제했습니다. |
| root cause | deadline clock과 protocol correlation이 각각 wall-clock 및 boolean state에만 의존했습니다. |
| 수정된 invariant / decision | steady_clock elapsed time과 outstanding exact token을 함께 authoritative liveness state로 사용합니다. |
| regression evidence | 0c76aad19579가 matching PONG은 유지하고 forged PONG은 timeout ERROR/close로 끝나는 두 경로를 분리합니다. |

#### 다음 연결

- 다음 Thread commit: `0c76aad19579` — `test(heartbeat): PONG 토큰과 시간 경계 검증`. 원문 역할은 “Proves matching PONG clears the deadline and forged PONG does not.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `0c76aad19579` — `test(heartbeat): PONG 토큰과 시간 경계 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, RESILIENCE, RISK |
| 원문 확정 역할 | Proves matching PONG clears the deadline and forged PONG does not. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Proves matching PONG clears the deadline and forged PONG does not.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | steady-clock/exact-token fix는 있었지만 양성·위조 response가 서로 다른 production branch를 통과한다는 regression 증거가 없었습니다. |
| 주요 문제와 설계 판단 | test peer의 auto-PONG을 끈 채 server PING token을 직접 읽어 exact PONG을 보내는 생존 case와, 다른 token을 보낸 뒤 exact `ERROR :Ping timeout` 및 close를 기다리는 forged case를 추가했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py or tools/irc_smoke_client.py — heartbeat token/deadline scenarios` |
| state / ownership 변화 | test가 server가 발급한 token을 capture하고 scenario별 socket/deadline을 독립 관리합니다. |
| failure 또는 boundary | forged PONG은 awaiting state를 바꾸지 않아 original `lastPingAt` 기준 deadline에서 timeout되고 connection이 닫혀야 합니다. |
| 보장 / 비보장 | 보장: matching response만 liveness deadline을 해제하며 unrelated response는 connection을 보존하지 못합니다.<br>비보장: OS wall-clock을 실제로 뒤로 움직이는 test는 아니며 steady_clock type과 code inspection이 그 부분의 근거입니다. |
| 다음 관련 변화 | 416efc91e580가 이 contract를 Linux/macOS와 sanitizer build에서 반복하도록 합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `0c76aad19579` | `tests/irc_contract.py` | `matching PONG scenario` | captured exact token response 후 connection progress |
| `0c76aad19579` | `tests/irc_contract.py` | `forged PONG scenario` | 불일치 token 뒤 exact Ping timeout ERROR와 close |

- 조사 방법: GitHub에서 `0c76aad19579`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 현재 outstanding heartbeat token과 정확히 일치하는 PONG만 wait state를 해제해야 합니다. |
| 재현 failure / boundary | matching token과 forged/unrelated token, ping deadline입니다. |
| test technique | real process/socket + auto-PONG disabled + server token capture + exact timeout frame assertion입니다. |
| 통과하는 production path | maintainClient PING state → wire token → handlePong exact comparison → clear 또는 unchanged → timeout/close입니다. |
| 이 test가 증명하는 것 | token correlation의 positive/negative branch와 deadline outcome을 deterministic하게 구분합니다. |
| 이 test가 증명하지 않는 것 | 실제 system-clock jump나 모든 scheduler delay, token 보안성은 증명하지 않습니다. |
| test 성격 | deterministic liveness regression |
| 막는 회귀 | any-PONG acceptance와 forged response로 idle connection을 무기한 유지하는 문제를 막습니다. |

실행 기록:

- 실행 SHA: `0c76aad19579`
- repository에 정의된 실행 명령: `make test`의 heartbeat contract scenario
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| idle heartbeat policy | `764361c52b2a` | PING/awaiting/timeout state | registration timeout 뒤 idle probe와 deadline을 적용합니다. | 초기 wall-clock/any-PONG이 부족함 |
| test peer response seam | `d710f29f38a4` | f313e707474f | matching PONG 자동응답과 positive real-socket 흐름을 제공합니다. | negative token branch는 아직 없음 |
| monotonic exact challenge | `3f2b3ae1d3f9` | steady_clock + pendingPongToken | ordinary activity와 unrelated PONG은 challenge를 완료하지 않습니다. | 0c76aad19579 matching/forged regression |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| wall-clock elapsed time | clock rollback/adjustment으로 deadline 왜곡 | 3f2b3ae1d3f9 | steady_clock timestamp | 0c76aad19579은 code path, 실제 clock jump는 inspection 근거 |
| 아무 PONG이면 현재 challenge 응답 | forged/unrelated token으로 timeout 회피 | 3f2b3ae1d3f9 | exact pending token comparison | 0c76aad19579 |
| 양성 PING/PONG만 검증 | 실제 결함 branch가 드러나지 않음 | 0c76aad19579 | auto-PONG off + forged response | exact ERROR/close assertion |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| activity/deadline timestamps | wall-clock time_t | ClientState steady_clock time points | 764361c52b2a → 3f2b3ae1d3f9 | path/symbol은 위 commit 증거표에 연결했습니다. |
| outstanding challenge identity | awaiting bool만 존재 | ClientState pendingPongToken | 3f2b3ae1d3f9 | path/symbol은 위 commit 증거표에 연결했습니다. |
| token sequence | fd/time 조합 | IrcApplication monotonic counter | 3f2b3ae1d3f9 | path/symbol은 위 commit 증거표에 연결했습니다. |
| test response policy | 항상 일반 line | IrcPeer auto_pong flag | d710f29f38a4 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: 등록 deadline 외에는 유휴 연결의 생존 여부를 확인하거나 제거하는 정책이 없었습니다.
- 마지막 commit `0c76aad19579` 시점의 상태: matching response만 liveness deadline을 해제하며 unrelated response는 connection을 보존하지 못합니다.
- Thread 안에서 강화된 핵심 invariant: idle heartbeat policy, test peer response seam, monotonic exact challenge.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: OS wall-clock을 실제로 뒤로 움직이는 test는 아니며 steady_clock type과 code inspection이 그 부분의 근거입니다. 416efc91e580가 이 contract를 Linux/macOS와 sanitizer build에서 반복하도록 합니다.
- 최종 설명: `764361c52b2a`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `0c76aad19579`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| activity receive | 764361c52b2a / 3f2b3ae1d3f9 | `IrcApplication::onLine` | lastActivityAt 갱신 | ordinary activity는 outstanding token을 clear하지 않음 |
| idle probe | 3f2b3ae1d3f9 | `maintainClient` | token 생성 → awaiting/token/time 저장 → PING send | send failure면 lifecycle 종료 후 state 재사용 안 함 |
| response | 3f2b3ae1d3f9 | `handlePong` | exact one-param token이면 awaiting/token clear | 불일치/누락은 unchanged |
| deadline | 3f2b3ae1d3f9 | `maintainClient` | steady elapsed가 ping timeout 미만이면 대기 | 초과 시 ERROR + requestClose |
| regression | 0c76aad19579 | `real socket scenario` | matching PONG은 progress | forged PONG은 exact Ping timeout ERROR/close |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 06-heartbeat-liveness-correctness.md =====

===== BEGIN FILE: 07-output-queue-correctness-under-partial-failure.md =====
# Thread 07 — Output-queue correctness under partial failure

부제: 부분 실패에서의 송신 대기열 정확성

## 1. Thread 목표

persistent partial-send state가 resource limit 도입 뒤 어떤 arithmetic·retry 경계에서 깨질 수 있었는지 추적하고, overflow-safe admission과 injected send script가 exact-once ordered delivery invariant를 어떻게 복구·검증하는지 학습합니다.

Source에서 확정된 significance:

> This thread shows a foundational mechanism being revisited after resource limits make its boundary conditions observable. The later fix is not a feature extension: it protects the exact-once output invariant against arithmetic wraparound, impossible syscall results, and retry-state corruption. Injecting the send operation turns those rare conditions into reproducible unit tests.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- sent prefix와 unsent suffix를 나타내는 최소 state는 무엇인가?
- pending-byte limit은 backing buffer가 아니라 logical unsent bytes를 어떻게 측정하는가?
- `pending + byteCount` 비교가 어떤 값에서 wrap될 수 있으며 subtraction predicate는 왜 안전한가?
- zero, EINTR, EAGAIN, EWOULDBLOCK, EPIPE, impossible oversize return에서 offset과 queue는 어떻게 남는가?
- scripted sender가 real socket으로 만들기 어려운 sequence를 어떻게 재현하는가?

### Source와 직접 연결되는 invariant

- partial send는 kernel이 보고한 exact byte 수만큼만 offset을 전진해야 합니다.
- retryable/zero/terminal outcome에서도 아직 전송되지 않은 suffix는 순서와 내용이 보존되어야 합니다.
- queue-limit arithmetic은 wrap할 수 없고 rejected bytes는 이미 pending인 state를 변경하지 않아야 합니다.

### Source와 직접 연결되는 engineering difficulty

- rare syscall outcome과 size arithmetic failure를 timing-dependent network accident 없이 재현하는 문제.
- slow receiver pressure를 connection-local pending state로 격리하면서 exact byte accounting을 유지하는 문제.

## 3. 완료 기준

- a10fe961e2b1의 buffer/offset invariant와 graceful drain을 byte sequence로 설명합니다.
- d7d85e518177의 exact admission 기준과 rejection mutation 순서를 기록했습니다.
- 881e59734a9a의 arithmetic proof와 impossible send-count guard를 parent change로 설명합니다.
- f34ab135c546의 script steps와 expected pending/output 상태를 재구성했습니다.
- 각 test가 증명하는 exact-order/boundary와 증명하지 않는 kernel scheduling 범위를 구분했습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `a10fe961e2b1` | `feat(connection): 부분 송신 대기열 처리` | S | CORE, EVENT_IO, LIFECYCLE | Establishes persistent partial-send state and exact-order delivery. |
| 2 | `d7d85e518177` | `feat(buffer): 송신 대기열 크기 제한` | A | EVENT_IO, RESILIENCE, RISK | Adds a hard pending-output boundary and a visible queue-failure result. |
| 3 | `881e59734a9a` | `fix(connection): 송신 대기열 계산과 재시도 상태 보호` | S | DEBUG, EVENT_IO, RISK | Makes limit arithmetic overflow-safe and protects offset state across every send outcome. |
| 4 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | VERIFICATION, EVENT_IO, RISK | Scripts partial sends, interruptions, backpressure, zero sends, terminal errors, and invalid return counts. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `a10fe961e2b1` — `feat(connection): 부분 송신 대기열 처리`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | CORE, EVENT_IO, LIFECYCLE |
| 원문 확정 역할 | Establishes persistent partial-send state and exact-order delivery. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Establishes persistent partial-send state and exact-order delivery.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | 출력 bytes와 한 번의 send를 연결할 persistent progress state가 없었습니다. |
| failure / boundary | EINTR은 같은 suffix를 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 핵심 결정 | `writeOffset_`로 sent prefix를 나타내고 `flushPending()`이 unsent suffix만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 변경 파일·symbol | `src/Connection.cpp — flushPending, queueRaw, queueLine, pendingBytes, wantsWrite` |
| ownership / lifecycle / state transition | `writeBuffer_[0:writeOffset_]`는 이미 전송된 prefix, 나머지는 순서가 보존된 pending suffix입니다. 완료 시 buffer/offset을 reset하고 큰 consumed prefix만 compact합니다. |
| 이 commit이 보장하는 것 | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 아직 보장하지 않는 것 | queue 크기는 무제한이며 `pending + append` overflow와 send가 요청보다 큰 값을 돌려주는 비정상 결과는 아직 방어하지 않습니다. |
| 후속 fix/test 연결 | 625ffc924de8이 write interest에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | 출력 bytes와 한 번의 send를 연결할 persistent progress state가 없었습니다. |
| 2. failure conditions / boundary | EINTR은 같은 suffix를 재시도하고, EAGAIN/EWOULDBLOCK 및 0은 offset을 유지한 채 다음 writable event를 기다리며, hard error는 close를 요청합니다. `MSG_NOSIGNAL`을 사용할 수 있는 플랫폼에서는 SIGPIPE를 억제합니다. |
| 3. 선택한 표현과 순서 | `writeOffset_`로 sent prefix를 나타내고 `flushPending()`이 unsent suffix만 전송하도록 했습니다. `queueRaw()`와 `queueLine()`은 bytes를 append하고 line terminator를 CRLF 하나로 정규화했습니다. |
| 4. authoritative state | `writeBuffer_[0:writeOffset_]`는 이미 전송된 prefix, 나머지는 순서가 보존된 pending suffix입니다. 완료 시 buffer/offset을 reset하고 큰 consumed prefix만 compact합니다. |
| 5. 결과 invariant | short send와 backpressure 사이에서도 성공한 byte 수만큼만 전진해 exact-order delivery의 기반을 만듭니다. |
| 6. 후속 보강 | 625ffc924de8이 write interest에 연결하고, d7d85e518177가 limit를 추가한 뒤 881e59734a9a/f34ab135c546가 경계를 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `a10fe961e2b1` | `src/Connection.cpp` | `Connection::flushPending` | unsent suffix pointer/length와 return-count 기반 offset 전진 |
| `a10fe961e2b1` | `src/Connection.cpp` | `queueLine` | 기존 CR/LF 제거 후 CRLF 하나 append |
| `a10fe961e2b1` | `src/Connection.cpp` | `pendingBytes/wantsWrite` | logical unsent bytes가 server interest 판단의 입력 |

- 조사 방법: GitHub에서 `a10fe961e2b1`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`. 원문 역할은 “Adds a hard pending-output boundary and a visible queue-failure result.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `d7d85e518177` — `feat(buffer): 송신 대기열 크기 제한`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | EVENT_IO, RESILIENCE, RISK |
| 원문 확정 역할 | Adds a hard pending-output boundary and a visible queue-failure result. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds a hard pending-output boundary and a visible queue-failure result.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | partial output queue가 무제한이라 느리거나 읽지 않는 peer가 memory를 계속 점유할 수 있었습니다. |
| 주요 문제와 설계 판단 | Connection에 `maxPendingBytes_`를 추가하고 `pendingBytes()` 기준으로 `queueRaw/queueLine` admission을 제한했습니다. queue 함수는 bool을 반환하고 초과 시 close를 요청하며 Server send API가 실패를 caller에 전달했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — maxPendingBytes and bool queue API`<br>`src/Connection.cpp — pendingBytes, canAppendPending, queueRaw, queueLine`<br>`src/Server.cpp — sendTo/queueRawTo propagation` |
| state / ownership 변화 | limit은 backing buffer size가 아니라 `writeBuffer_.size() - writeOffset_`인 logical unsent bytes를 지배합니다. |
| failure 또는 boundary | 초과 append는 기존 pending bytes를 변경하지 않고 `outbound queue limit exceeded` close state를 만듭니다. |
| 보장 / 비보장 | 보장: 한 connection의 unsent output에 hard bound와 observable failure result를 제공합니다.<br>비보장: `pending + byteCount <= limit` 덧셈은 size_t wrap 가능성이 있고 CRLF 추가의 두 단계 경계도 충분히 증명되지 않았습니다. |
| 다음 관련 변화 | 881e59734a9a가 subtraction predicate와 send guard로 수정하고 f34ab135c546가 deterministic하게 검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d7d85e518177` | `src/Connection.cpp` | `pendingBytes/canAppendPending` | logical unsent byte 기준 admission |
| `d7d85e518177` | `src/Server.cpp` | `sendTo/queueRawTo` | queue failure를 application까지 bool로 전달 |

- 조사 방법: GitHub에서 `d7d85e518177`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `881e59734a9a` — `fix(connection): 송신 대기열 계산과 재시도 상태 보호`. 원문 역할은 “Makes limit arithmetic overflow-safe and protects offset state across every send outcome.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `881e59734a9a` — `fix(connection): 송신 대기열 계산과 재시도 상태 보호`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | DEBUG, EVENT_IO, RISK |
| 원문 확정 역할 | Makes limit arithmetic overflow-safe and protects offset state across every send outcome. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Makes limit arithmetic overflow-safe and protects offset state across every send outcome.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | pending limit은 `pending + append <= limit` 덧셈을 사용했고 send return count를 요청 길이보다 작거나 같다고 암묵적으로 가정했습니다. rare retry sequence를 직접 주입할 seam도 없었습니다. |
| failure / boundary | size_t wrap, payload+CRLF 경계, impossible oversize return은 기존 pending bytes/offset을 보존한 채 rejection 또는 close로 수렴합니다. EINTR/would-block/zero/hard error의 기존 suffix 보존 semantics도 주입 가능한 operation을 통과합니다. |
| 핵심 결정 | `ConnectionLimits::canAppendPending()`을 `pending <= limit && byteCount <= limit - pending`으로 바꾸고 line payload와 CRLF를 별도 safe admission으로 검사했습니다. `SendOperation`을 constructor에 주입하고 move state에 포함했으며, positive send count가 요청 size를 넘으면 offset을 전진시키지 않고 hard error/close로 처리했습니다. |
| 변경 파일·symbol | `include/Connection.hpp — SendOperation constructor seam`<br>`src/Connection.cpp — send wrapper, move, flushPending, queueLine`<br>`src/ConnectionLimits.hpp — canAppendPending` |
| ownership / lifecycle / state transition | send operation도 Connection의 transport state와 함께 move되고, write offset은 검증된 실제 successful count에 의해서만 전진합니다. |
| 이 commit이 보장하는 것 | queue admission 산술이 overflow할 수 없고 모든 send outcome에서 unsent suffix와 exact-order offset invariant가 보호됩니다. |
| 아직 보장하지 않는 것 | 주입 seam은 send syscall 결과를 모델링하지만 실제 kernel scheduling, TCP peer behavior, write readiness fairness 자체는 보장하지 않습니다. |
| 후속 fix/test 연결 | f34ab135c546가 scripted sender로 산술·retry·partial·zero·terminal·invalid-count branch를 모두 고정합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | pending limit은 `pending + append <= limit` 덧셈을 사용했고 send return count를 요청 길이보다 작거나 같다고 암묵적으로 가정했습니다. rare retry sequence를 직접 주입할 seam도 없었습니다. |
| 2. failure conditions / boundary | size_t wrap, payload+CRLF 경계, impossible oversize return은 기존 pending bytes/offset을 보존한 채 rejection 또는 close로 수렴합니다. EINTR/would-block/zero/hard error의 기존 suffix 보존 semantics도 주입 가능한 operation을 통과합니다. |
| 3. 선택한 표현과 순서 | `ConnectionLimits::canAppendPending()`을 `pending <= limit && byteCount <= limit - pending`으로 바꾸고 line payload와 CRLF를 별도 safe admission으로 검사했습니다. `SendOperation`을 constructor에 주입하고 move state에 포함했으며, positive send count가 요청 size를 넘으면 offset을 전진시키지 않고 hard error/close로 처리했습니다. |
| 4. authoritative state | send operation도 Connection의 transport state와 함께 move되고, write offset은 검증된 실제 successful count에 의해서만 전진합니다. |
| 5. 결과 invariant | queue admission 산술이 overflow할 수 없고 모든 send outcome에서 unsent suffix와 exact-order offset invariant가 보호됩니다. |
| 6. 후속 보강 | f34ab135c546가 scripted sender로 산술·retry·partial·zero·terminal·invalid-count branch를 모두 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `881e59734a9a` | `src/ConnectionLimits.hpp` | `detail::canAppendPending` | subtraction predicate로 size_t wrap 방지 |
| `881e59734a9a` | `src/Connection.cpp` | `Connection::flushPending` | send count > requested guard가 offset mutation 전에 실행 |
| `881e59734a9a` | `src/Connection.cpp` | `queueLine` | payload와 CRLF를 각각 overflow-safe하게 admission |
| `881e59734a9a` | `include/Connection.hpp` | `SendOperation` | rare syscall result를 production path에 주입 |

- 조사 방법: GitHub에서 `881e59734a9a`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | unsigned 덧셈이 limit 비교 전에 wrap하지 않고 send가 요청 길이 이하를 반환한다는 가정이었습니다. |
| 실제 failure / risk | wrap된 합은 oversize queue를 허용하고 impossible return은 offset을 buffer 끝 너머로 이동시켜 pending 계산과 retry state를 깨뜨릴 수 있었습니다. |
| root cause | admission과 progress mutation 전에 arithmetic/syscall postcondition을 명시적으로 검증하지 않았습니다. |
| 수정된 invariant / decision | limit에서 pending을 빼는 representable 비교와 return-count upper bound 검사를 mutation 전에 수행합니다. |
| regression evidence | f34ab135c546가 size_t maximum, exact CRLF limit, EINTR/partial/EAGAIN/EWOULDBLOCK/zero/EPIPE/oversize return을 script로 검증합니다. |

#### 다음 연결

- 다음 Thread commit: `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`. 원문 역할은 “Scripts partial sends, interruptions, backpressure, zero sends, terminal errors, and invalid return counts.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, EVENT_IO, RISK |
| 원문 확정 역할 | Scripts partial sends, interruptions, backpressure, zero sends, terminal errors, and invalid return counts. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Scripts partial sends, interruptions, backpressure, zero sends, terminal errors, and invalid return counts.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection fix의 rare syscall/overflow branch는 real socket timing에 맡기면 반복적으로 재현하기 어려웠습니다. |
| 주요 문제와 설계 판단 | `SendStep`과 `ScriptedSender`가 Bytes/Error/Zero sequence를 반환하는 unit binary를 추가하고 production `Connection::flushPending()`에 주입했습니다. Makefile에 `connection-test`를 연결했습니다. |
| 변경 파일·symbol | `tests/connection_test.cpp — SendStep, ScriptedSender and five test groups`<br>`Makefile — connection-test` |
| state / ownership 변화 | script index, captured bytes와 Connection pending/offset/close state를 각 단계 후 동시에 assertion합니다. |
| failure 또는 boundary | EINTR→partial→EAGAIN→partial→EWOULDBLOCK→final sequence, zero then progress, EPIPE, requested size보다 큰 return, size_t max arithmetic와 exact CRLF limit을 결정적으로 재현합니다. |
| 보장 / 비보장 | 보장: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.<br>비보장: fake sender는 kernel buffer, actual readiness notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. |
| 다음 관련 변화 | de1dd0fc30d0가 real process/slow reader에서 connection-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f34ab135c546` | `tests/connection_test.cpp` | `testLimitArithmetic/testHardLimitAndLineTerminator` | size_t max와 CRLF 포함 exact-limit mutation |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInterruptedAndPartialSendState` | scripted retry마다 pending offset과 exact output |
| `f34ab135c546` | `tests/connection_test.cpp` | `testZeroAndTerminalErrorPreservePendingBytes` | zero/EPIPE가 suffix를 소비하지 않음 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInvalidSendCountCannotAdvanceOffset` | oversize return이 offset을 손상시키지 않음 |

- 조사 방법: GitHub에서 `f34ab135c546`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | logical pending limit은 overflow하지 않고, offset은 검증된 successful byte만큼만 전진하며 unsent suffix가 exact order로 보존됩니다. |
| 재현 failure / boundary | size_t max, CRLF exact limit, EINTR, partial, EAGAIN/EWOULDBLOCK, zero, EPIPE, impossible oversize return입니다. |
| test technique | injected send operation을 사용하는 deterministic scripted unit test입니다. |
| 통과하는 production path | queueRaw/queueLine → overflow-safe admission → flushPending → injected send result → offset/close/pending query입니다. |
| 이 test가 증명하는 것 | rare outcome sequence에서 exact byte accounting과 mutation-free rejection을 직접 증명합니다. |
| 이 test가 증명하지 않는 것 | real kernel readiness, TCP delivery acknowledgement, end-to-end fairness는 증명하지 않습니다. |
| test 성격 | deterministic transport boundary regression |
| 막는 회귀 | partial retry 중 byte 중복·유실, unsigned wrap, invalid return으로 offset이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f34ab135c546`
- repository에 정의된 실행 명령: `make connection-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| persistent sent-prefix state | `a10fe961e2b1` | writeOffset/pending suffix | positive return만큼만 전진하고 retry 사이에 offset을 보존합니다. | 881e59734a9a invalid-count guard |
| hard logical pending bound | `d7d85e518177` | pendingBytes + bool admission | limit 초과 append는 기존 queue를 변경하지 않습니다. | 초기 addition wrap이 부족함 |
| overflow-safe exact-order invariant | `881e59734a9a` | subtraction predicate + injected sender | all outcome에서 offset/suffix를 mutation 전에 검증합니다. | f34ab135c546 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| 한 send가 전부 또는 정상 범위 반환 | partial/retry/invalid count에서 byte 중복·유실·offset 손상 | a10fe961e2b1 → 881e59734a9a | persistent offset + count upper bound | f34ab135c546 |
| pending + append 비교는 안전 | size_t wrap 후 oversize admission | 881e59734a9a | append <= limit - pending | f34ab135c546 |
| rare syscall sequence는 real socket으로 충분 | timing에 따라 branch 미재현 | 881e59734a9a SendOperation seam | scripted deterministic outcomes | f34ab135c546 |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| output backing bytes/offset | 없음 | Connection | a10fe961e2b1 | path/symbol은 위 commit 증거표에 연결했습니다. |
| logical pending limit | 무제한 | Connection maxPendingBytes | d7d85e518177 | path/symbol은 위 commit 증거표에 연결했습니다. |
| send operation | 직접 syscall 고정 | Connection-owned SendOperation | 881e59734a9a | path/symbol은 위 commit 증거표에 연결했습니다. |
| failure script/captured output | 없음 | connection_test ScriptedSender | f34ab135c546 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: 출력 bytes와 한 번의 send를 연결할 persistent progress state가 없었습니다.
- 마지막 commit `f34ab135c546` 시점의 상태: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.
- Thread 안에서 강화된 핵심 invariant: persistent sent-prefix state, hard logical pending bound, overflow-safe exact-order invariant.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: fake sender는 kernel buffer, actual readiness notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. de1dd0fc30d0가 real process/slow reader에서 connection-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다.
- 최종 설명: `a10fe961e2b1`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `f34ab135c546`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| admission | 881e59734a9a | `queueRaw/queueLine` | pending <= limit && append <= limit-pending | reject 시 existing queue/offset unchanged |
| writable call | a10fe961e2b1 / 881e59734a9a | `flushPending` | unsent suffix pointer/length로 SendOperation 호출 | count > requested는 terminal before offset mutation |
| retry outcomes | 881e59734a9a | `flushPending` | EINTR retry, partial positive offset advance | EAGAIN/EWOULDBLOCK/zero는 suffix 보존, EPIPE는 close |
| completion | a10fe961e2b1 | `flushPending` | offset가 size에 도달하면 buffer/offset reset | closeRequested라면 Server가 drain 후 disconnect |
| regression | f34ab135c546 | `ScriptedSender` | exact `abcdefghijkl` captured once | every failure step에서 pending/output assertions |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 07-output-queue-correctness-under-partial-failure.md =====

===== BEGIN FILE: 08-reentrant-server-and-application-cleanup.md =====
# Thread 08 — Reentrant server and application cleanup

부제: 재진입 가능한 서버와 애플리케이션 정리

## 1. Thread 목표

callback, interest update, response enqueue가 현재 처리 중인 connection과 application aggregate를 동기적으로 제거할 수 있다는 사실을 중심으로, descriptor relookup·rollback·failure propagation·command continuation 중단 invariant를 fix/test 쌍으로 복원합니다.

Source에서 확정된 significance:

> The difficult integration problem is that a seemingly local send or callback can synchronously remove the connection, client record, memberships, and possibly the channel currently referenced by the caller. The server fix restores descriptor and event-registry ownership; the application fix generalizes the rule to domain state. The follow-up commit and test refine the chosen semantics: earlier mutations are not rolled back transactionally, but command processing must stop immediately after the actor's lifecycle ends.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- 초기 erase-before-callback cleanup은 무엇을 보장하고 무엇을 아직 보장하지 않는가?
- callback self-disconnect 뒤 raw pointer/reference가 왜 즉시 위험해지는가?
- accept/listener registration failure에서 map, event backend, fd lifetime을 어떻게 rollback하는가?
- `sendTo()` failure가 application client/index/membership/channel erase로 이어지는 synchronous call stack은 무엇인가?
- send 뒤 언제 return하고 언제 stable id로 re-resolve해야 하는가?
- compound MODE의 부분 완료를 rollback하지 않으면서 후속 transition을 막는 semantics는 무엇인가?

### Source와 직접 연결되는 invariant

- callback, interest update, response enqueue 뒤에는 이전 `Connection`, client, channel, pointer, reference, iterator를 재사용하지 않고 필요한 state를 다시 resolve해야 합니다.
- connection map, event-manager registration, descriptor lifetime은 함께 advance하거나 함께 teardown되어야 합니다.
- output failure로 actor lifecycle이 끝나면 이전 mutation을 rollback하지 않더라도 이후 command transition은 실행하지 않아야 합니다.

### Source와 직접 연결되는 engineering difficulty

- single-threaded code에서도 external callback이 current owner를 동기적으로 지우는 reentrancy lifetime 문제.
- send처럼 보이는 local side effect가 disconnect callback과 channel erase까지 일으키는 cross-layer invalidation 문제.
- rare add/update failure와 exact continuation point를 fake backend/limits로 deterministic하게 만드는 문제.

## 3. 완료 기준

- 7a6bc7e1276a의 cleanup ordering을 baseline으로 기록했습니다.
- 5dcd882f0763의 callback/registration/update failure 경계와 fd-based relookup/rollback을 설명합니다.
- 928594ec160c의 five failure contracts와 fake backend injection path를 구분했습니다.
- 728aaabc4012의 cross-layer send-failure invalidation rule을 stable copy/relookup으로 복원했습니다.
- d48e1f1f8c04/aee5edebe294의 non-transactional stop-after-failure semantics를 실제 state assertions로 설명합니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `7a6bc7e1276a` | `feat(server): 연결 해제와 오류 정리 구현` | A | LIFECYCLE, RISK | Establishes the original erase-before-disconnect-callback cleanup path. |
| 2 | `5dcd882f0763` | `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장` | S | DEBUG, LIFECYCLE, RISK | Handles callbacks that remove their own connection and rolls back event registration failures. |
| 3 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | VERIFICATION, LIFECYCLE, RISK | Injects add/update failures and callback-driven removal to verify server lifetime safety. |
| 4 | `728aaabc4012` | `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인` | S | DEBUG, LIFECYCLE, RISK | Propagates output failure and requires application state to be re-resolved after any send that can disconnect. |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | VERIFICATION, LIFECYCLE, RISK | Forces registration response failure and proves application cleanup and logging order. |
| 6 | `d48e1f1f8c04` | `fix(app): 응답 실패 뒤 명령 처리를 중단` | A | DEBUG, LIFECYCLE, RISK | Stops LIST, NAMES, and compound MODE after response failure. |
| 7 | `aee5edebe294` | `test(app): 연결 정리 뒤 모드 변경 중단 검증` | A | VERIFICATION, LIFECYCLE, CHANNEL_STATE | Proves a compound MODE retains completed work but performs no later transition after sender removal. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `7a6bc7e1276a` — `feat(server): 연결 해제와 오류 정리 구현`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | LIFECYCLE, RISK |
| 원문 확정 역할 | Establishes the original erase-before-disconnect-callback cleanup path. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Establishes the original erase-before-disconnect-callback cleanup path.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 여러 event/error/stop 경로에서 connection 제거 순서가 하나의 함수로 고정되지 않았습니다. |
| 주요 문제와 설계 판단 | `disconnect()`가 backend remove를 시도하고 `unique_ptr`를 map 밖 local로 옮겨 entry를 먼저 erase한 뒤 disconnect callback을 호출했습니다. bulk close는 fd snapshot을 사용했습니다. |
| 변경 파일·symbol | `src/Server.cpp — disconnect, closeAllConnections, handleClientEvent` |
| state / ownership 변화 | callback 시점에는 server map에서 fd가 이미 사라졌지만 local unique_ptr가 callback 동안 객체 lifetime을 유지하고 callback 후 destructor가 fd를 닫습니다. |
| failure 또는 boundary | backend remove와 callback exception은 report되더라도 object destruction을 막지 않습니다. hangup은 pending output이 없을 때 terminal입니다. |
| 보장 / 비보장 | 보장: recursive lookup/repeated disconnect는 이미 제거된 상태를 보고, map iteration 중 erase로 iterator가 깨지지 않습니다.<br>비보장: callback 전에 잡은 raw pointer를 callback 뒤 다시 사용하는 call site와 event add/update rollback은 아직 안전하지 않습니다. |
| 다음 관련 변화 | 5dcd882f0763/928594ec160c가 이 baseline의 reentrancy와 rollback 결함을 수정·검증합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::disconnect` | event remove → map ownership move/erase → callback → destruction 순서 |
| `7a6bc7e1276a` | `src/Server.cpp` | `Server::closeAllConnections` | fd snapshot으로 반복 erase 안전성 |

- 조사 방법: GitHub에서 `7a6bc7e1276a`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### 다음 연결

- 다음 Thread commit: `5dcd882f0763` — `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장`. 원문 역할은 “Handles callbacks that remove their own connection and rolls back event registration failures.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `5dcd882f0763` — `fix(server): 연결 콜백 수명과 이벤트 등록 롤백 보장`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | DEBUG, LIFECYCLE, RISK |
| 원문 확정 역할 | Handles callbacks that remove their own connection and rolls back event registration failures. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Handles callbacks that remove their own connection and rolls back event registration failures.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server는 callback 전에 map에서 connection을 지우는 disconnect 경로는 가졌지만, connect/line callback이 자신을 제거한 뒤 기존 raw pointer를 다시 쓰고 event add/update 실패 시 map·backend·fd state가 갈라질 수 있었습니다. |
| failure / boundary | listener add 실패는 listen fd를 닫고 backend owner를 reset합니다. client add 실패는 inserted map entry를 erase해 Connection destructor가 fd를 닫습니다. connect/line callback self-disconnect 뒤 lookup이 null이면 즉시 중단하고 update failure는 backend/map 모두 제거합니다. |
| 핵심 결정 | test용 EventManager 주입 constructor를 추가하고 listener add를 rollback 가능하게 했습니다. accept에서는 map에 먼저 `emplace`한 뒤 backend add가 실패하면 entry를 erase했습니다. callback과 read/write 단계 후에는 fd로 `findConnection()`을 다시 호출하고, `refreshInterest(int fd)`는 update exception을 잡아 disconnect한 뒤 bool을 반환하도록 바꿨습니다. error callback 예외는 `noexcept` report path에서 흡수했습니다. |
| 변경 파일·symbol | `include/Server.hpp — injected EventManager constructor, refreshInterest(int)`<br>`src/Server.cpp — start rollback, acceptReadyClients, handleClientEvent, refreshInterest, reportError` |
| ownership / lifecycle / state transition | stable identity는 fd이고 `Connection*`는 다음 external callback/interest operation 전까지만 유효합니다. connection map, backend registration, descriptor lifetime이 성공 시 함께 전진하고 실패 시 authoritative disconnect/erase로 함께 teardown됩니다. |
| 이 commit이 보장하는 것 | single-threaded callback reentrancy와 event registration/update 실패 뒤 stale pointer, leaked connection, ghost backend entry가 남지 않습니다. |
| 아직 보장하지 않는 것 | application handler가 response send 후 borrowed ClientState/Channel/iterator를 재사용하는 cross-layer 문제는 아직 남습니다. |
| 후속 fix/test 연결 | 928594ec160c가 fake backend로 five server failure contracts를 검증하고 728aaabc4012가 동일 규칙을 application state까지 확장합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server는 callback 전에 map에서 connection을 지우는 disconnect 경로는 가졌지만, connect/line callback이 자신을 제거한 뒤 기존 raw pointer를 다시 쓰고 event add/update 실패 시 map·backend·fd state가 갈라질 수 있었습니다. |
| 2. failure conditions / boundary | listener add 실패는 listen fd를 닫고 backend owner를 reset합니다. client add 실패는 inserted map entry를 erase해 Connection destructor가 fd를 닫습니다. connect/line callback self-disconnect 뒤 lookup이 null이면 즉시 중단하고 update failure는 backend/map 모두 제거합니다. |
| 3. 선택한 표현과 순서 | test용 EventManager 주입 constructor를 추가하고 listener add를 rollback 가능하게 했습니다. accept에서는 map에 먼저 `emplace`한 뒤 backend add가 실패하면 entry를 erase했습니다. callback과 read/write 단계 후에는 fd로 `findConnection()`을 다시 호출하고, `refreshInterest(int fd)`는 update exception을 잡아 disconnect한 뒤 bool을 반환하도록 바꿨습니다. error callback 예외는 `noexcept` report path에서 흡수했습니다. |
| 4. authoritative state | stable identity는 fd이고 `Connection*`는 다음 external callback/interest operation 전까지만 유효합니다. connection map, backend registration, descriptor lifetime이 성공 시 함께 전진하고 실패 시 authoritative disconnect/erase로 함께 teardown됩니다. |
| 5. 결과 invariant | single-threaded callback reentrancy와 event registration/update 실패 뒤 stale pointer, leaked connection, ghost backend entry가 남지 않습니다. |
| 6. 후속 보강 | 928594ec160c가 fake backend로 five server failure contracts를 검증하고 728aaabc4012가 동일 규칙을 application state까지 확장합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5dcd882f0763` | `src/Server.cpp` | `Server::start` | listener add 예외 시 listen fd close와 backend reset |
| `5dcd882f0763` | `src/Server.cpp` | `acceptReadyClients` | map emplace → backend add → callback, add 실패 시 erase rollback |
| `5dcd882f0763` | `src/Server.cpp` | `handleClientEvent` | callback/read/write 후 fd relookup과 null이면 즉시 return |
| `5dcd882f0763` | `src/Server.cpp` | `refreshInterest(int)` | update failure report → disconnect → false |
| `5dcd882f0763` | `src/Server.cpp` | `reportError noexcept` | user error callback 예외가 cleanup을 중단하지 않음 |

- 조사 방법: GitHub에서 `5dcd882f0763`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | single-threaded callback과 event update가 반환되면 이전 Connection pointer/reference가 계속 유효하고 등록 단계가 부분 실패하지 않는다는 가정이었습니다. |
| 실제 failure / risk | callback self-disconnect는 map object를 파괴해 use-after-free를 만들고 add/update exception은 map과 kernel registry 중 한쪽만 남길 수 있었습니다. |
| root cause | callback/foreign operation을 lifecycle mutation boundary로 취급하지 않았고 map/backend/fd 변화가 transaction-like ordering을 갖지 않았습니다. |
| 수정된 invariant / decision | fd만 stable key로 보존하고 각 boundary 뒤 relookup하며, 등록은 map owner를 먼저 세운 뒤 backend 실패를 erase로 rollback하고 update 실패는 disconnect로 수렴합니다. |
| regression evidence | 928594ec160c가 add/update injection 및 connect/line callback removal을 production Server 경로에서 검증합니다. |

#### 다음 연결

- 다음 Thread commit: `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`. 원문 역할은 “Injects add/update failures and callback-driven removal to verify server lifetime safety.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, LIFECYCLE, RISK |
| 원문 확정 역할 | Injects add/update failures and callback-driven removal to verify server lifetime safety. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Injects add/update failures and callback-driven removal to verify server lifetime safety.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | server lifetime fix는 있었지만 add/update failure와 callback self-removal을 real backend/timing에 의존하지 않고 재현하는 regression suite가 없었습니다. |
| 주요 문제와 설계 판단 | `FakeEventManager`에 interest map, queued events, fail-next-add/update seam을 두고 loopback ClientSocket과 `Server::pollOnce()`를 결합한 `server_lifetime_test`를 추가했습니다. |
| 변경 파일·symbol | `tests/server_lifetime_test.cpp — FakeEventManager, ClientSocket, five tests`<br>`Makefile — unit/server lifetime target` |
| state / ownership 변화 | fake backend가 등록 fd와 호출 횟수를 관찰하고 test가 real accepted socket과 Server connection count를 함께 assertion합니다. |
| failure 또는 boundary | (1) client add 실패 rollback, (2) connect callback self-disconnect+throw, (3) line callback self-disconnect+throw, (4) write interest update 실패, (5) outbound queue rejection close를 각각 결정적으로 주입합니다. |
| 보장 / 비보장 | 보장: 각 case 뒤 connection map과 fake backend에 client fd가 남지 않고 callback 예외가 cleanup을 무효화하지 않으며 queue/update failure가 false를 반환합니다.<br>비보장: fake backend는 epoll/kqueue native semantics 자체를 검증하지 않고 callback 이후 application channel/client graph도 직접 검사하지 않습니다. |
| 다음 관련 변화 | 728aaabc4012/5edcafda8a4d가 send failure가 application state와 log까지 정리되는 경로를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `registrationRollbackTest` | injected add failure 뒤 map/backend가 모두 비어 있음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `connectCallbackLifetimeTest/lineCallbackLifetimeTest` | callback self-disconnect와 throw 뒤 stale access 없음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `interestUpdateRollbackTest` | update failure가 send false와 map/backend removal로 수렴 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `queueLimitCloseTest` | queue rejection close가 event/map entry를 남기지 않음 |

- 조사 방법: GitHub에서 `928594ec160c`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | connection map, event registration, descriptor lifetime이 함께 전진·정리되고 callback 뒤 stale pointer를 재사용하지 않아야 합니다. |
| 재현 failure / boundary | event add/update exception, connect/line callback self-disconnect+throw, outbound queue rejection입니다. |
| test technique | injected FakeEventManager + real loopback accepted socket + direct `pollOnce()` deterministic lifetime unit/integration입니다. |
| 통과하는 production path | accept/read/send → Server map/backend → callback/refreshInterest → disconnect/erase/destruction입니다. |
| 이 test가 증명하는 것 | server-level ownership rollback과 callback relookup rule을 timing accident 없이 검증합니다. |
| 이 test가 증명하지 않는 것 | native epoll/kqueue kernel behavior, application aggregate cleanup, 모든 OS resource failure는 증명하지 않습니다. |
| test 성격 | deterministic server lifetime regression |
| 막는 회귀 | self-disconnect use-after-free와 partial event registration/interest update leak을 막습니다. |

실행 기록:

- 실행 SHA: `928594ec160c`
- repository에 정의된 실행 명령: `make unit` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `728aaabc4012` — `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인`. 원문 역할은 “Propagates output failure and requires application state to be re-resolved after any send that can disconnect.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `728aaabc4012` — `fix(app): 응답 실패 뒤 클라이언트 상태 다시 확인`

| 항목 | 값 |
| --- | --- |
| Importance | S |
| Tags | DEBUG, LIFECYCLE, RISK |
| 원문 확정 역할 | Propagates output failure and requires application state to be re-resolved after any send that can disconnect. |
| 학습 깊이 | 프로젝트 핵심 architecture/invariant입니다. 이전 상태, failure sequence, 핵심 결정, ownership/lifecycle, 남은 한계와 후속 fix/test를 모두 복원합니다. |

#### Source에서 확정된 역할

> Propagates output failure and requires application state to be re-resolved after any send that can disconnect.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 상태 | Server.sendTo가 queue/interest 실패 시 synchronous disconnect와 application disconnect callback을 실행할 수 있었지만 handlers는 send 뒤 기존 ClientState, Channel reference, iterator를 계속 사용했습니다. |
| failure / boundary | queue limit 또는 interest update failure가 `Server::disconnect()` → application `onDisconnect()` → registry membership/channel erase를 동기 실행할 수 있으므로 false 또는 missing relookup에서 command를 종료합니다. |
| 핵심 결정 | `sendRaw`, `sendNumeric`, `sendNumericRaw`, `sendTopicReply`, `sendNames`, `broadcastMode`가 bool을 반환하도록 전파했습니다. handler는 send 전에 channel name/nick/prefix 같은 stable value를 복사하고, send/broadcast 뒤 `_clients.contains/find`와 `_channels.find`로 다시 resolve하거나 실패 즉시 return하도록 바꿨습니다. |
| 변경 파일·symbol | `src/IrcApplication.hpp/.cpp — bool send helpers and support methods`<br>`src/ChannelCommands.cpp — JOIN/PART/KICK/INVITE/MODE revalidation`<br>`src/RegistrationCommands.cpp — registration response checks`<br>`src/ApplicationSupport.cpp — fan-out/relookup` |
| ownership / lifecycle / state transition | send는 단순 출력 append가 아니라 actor lifecycle을 끝낼 수 있는 mutation boundary입니다. pointer/reference/iterator 대신 fd와 copied channel name/nick이 boundary를 넘는 stable identity가 됩니다. |
| 이 commit이 보장하는 것 | 응답 실패 뒤 삭제된 client/channel state에 접근하지 않고 후속 response/mutation을 기존 borrowed object로 수행하지 않습니다. |
| 아직 보장하지 않는 것 | LIST/NAMES 반복과 compound MODE 일부 branch는 아직 send 실패 뒤 loop를 계속할 수 있어 d48e1f1f8c04에서 추가 수정됩니다. 이미 성공한 mutation을 transaction처럼 rollback하지는 않습니다. |
| 후속 fix/test 연결 | 5edcafda8a4d가 one-byte queue로 registration failure cleanup/log ordering을 검증하고 d48e1f1f8c04/aee5edebe294가 stop-after-failure semantics를 완성합니다. |

#### S-level invariant 재구성

| 순서 | 상태 / 판단 |
| --- | --- |
| 1. 이전 전제 | Server.sendTo가 queue/interest 실패 시 synchronous disconnect와 application disconnect callback을 실행할 수 있었지만 handlers는 send 뒤 기존 ClientState, Channel reference, iterator를 계속 사용했습니다. |
| 2. failure conditions / boundary | queue limit 또는 interest update failure가 `Server::disconnect()` → application `onDisconnect()` → registry membership/channel erase를 동기 실행할 수 있으므로 false 또는 missing relookup에서 command를 종료합니다. |
| 3. 선택한 표현과 순서 | `sendRaw`, `sendNumeric`, `sendNumericRaw`, `sendTopicReply`, `sendNames`, `broadcastMode`가 bool을 반환하도록 전파했습니다. handler는 send 전에 channel name/nick/prefix 같은 stable value를 복사하고, send/broadcast 뒤 `_clients.contains/find`와 `_channels.find`로 다시 resolve하거나 실패 즉시 return하도록 바꿨습니다. |
| 4. authoritative state | send는 단순 출력 append가 아니라 actor lifecycle을 끝낼 수 있는 mutation boundary입니다. pointer/reference/iterator 대신 fd와 copied channel name/nick이 boundary를 넘는 stable identity가 됩니다. |
| 5. 결과 invariant | 응답 실패 뒤 삭제된 client/channel state에 접근하지 않고 후속 response/mutation을 기존 borrowed object로 수행하지 않습니다. |
| 6. 후속 보강 | 5edcafda8a4d가 one-byte queue로 registration failure cleanup/log ordering을 검증하고 d48e1f1f8c04/aee5edebe294가 stop-after-failure semantics를 완성합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `728aaabc4012` | `src/IrcApplication.cpp` | `sendRaw/sendNumeric/sendNumericRaw` | Server send 결과를 application caller에 bool로 전파 |
| `728aaabc4012` | `src/ChannelCommands.cpp` | `handleJoin/part/kick/invite` | stable string copy와 send 뒤 client/channel relookup |
| `728aaabc4012` | `src/IrcApplication.cpp` | `maintainClient` | heartbeat state를 send 전에 설정하고 실패 시 removed state를 재사용하지 않음 |

- 조사 방법: GitHub에서 `728aaabc4012`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | response enqueue는 local operation이라 caller가 보유한 client/channel reference를 무효화하지 않는다는 가정이었습니다. |
| 실제 failure / risk | queue/interest failure는 synchronous disconnect callback을 통해 registry, memberships, empty channel을 지워 handler의 pointer/iterator를 dangling으로 만들었습니다. |
| root cause | cross-layer call stack의 lifecycle side effect가 send API의 void signature와 handler continuation에 숨겨져 있었습니다. |
| 수정된 invariant / decision | send failure를 bool로 끝까지 전파하고 stable id/value만 boundary를 넘기며 모든 이후 state는 authoritative containers에서 다시 resolve합니다. |
| regression evidence | 5edcafda8a4d가 registration response failure를, aee5edebe294가 compound MODE sender removal을 재현합니다. |

#### 다음 연결

- 다음 Thread commit: `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`. 원문 역할은 “Forces registration response failure and proves application cleanup and logging order.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, LIFECYCLE, RISK |
| 원문 확정 역할 | Forces registration response failure and proves application cleanup and logging order. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Forces registration response failure and proves application cleanup and logging order.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | application revalidation fix가 welcome 응답 실패로 connection이 제거되는 실제 cross-layer stack에서 state/log를 올바르게 정리한다는 test가 없었습니다. |
| 주요 문제와 설계 판단 | FakeEventManager와 captured stderr를 사용하는 `application_lifetime_test`를 추가하고 Server `maxPendingBytes=1`로 설정해 NICK/USER 등록 중 첫 welcome frame admission을 강제로 실패시켰습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — CapturedStderr, FakeEventManager, registrationQueueFailureTest`<br>`Makefile — application-test` |
| state / ownership 변화 | real accepted socket과 application callbacks를 사용하되 tiny queue limit가 deterministic disconnect seam이며 test가 server connection count, running state와 log text를 관찰합니다. |
| failure 또는 boundary | welcome queue failure는 Server disconnect와 application onDisconnect cleanup을 동기 실행합니다. |
| 보장 / 비보장 | 보장: 실패 뒤 connection count가 0이고 server는 계속 실행되며 제거된 client를 `event=client_registered`로 기록하지 않습니다.<br>비보장: 이 최초 application test는 channel/mode의 multi-step continuation을 검사하지 않습니다. |
| 다음 관련 변화 | d48e1f1f8c04/aee5edebe294가 LIST/NAMES/MODE continuation과 부분 완료 semantics를 추가로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5edcafda8a4d` | `tests/application_lifetime_test.cpp` | `registrationQueueFailureTest` | 1-byte limit → welcome send failure → connection cleanup, server running, registered log 부재 |

- 조사 방법: GitHub에서 `5edcafda8a4d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | registration response failure로 actor가 제거되면 application state와 lifecycle log가 제거 결과와 일치해야 합니다. |
| 재현 failure / boundary | `maxPendingBytes=1`에서 001 welcome frame queue rejection입니다. |
| test technique | tiny real Server queue limit + fake event backend + real loopback connection + captured stderr입니다. |
| 통과하는 production path | NICK/USER → maybeRegister → sendNumeric → Server.sendTo → queue rejection/refresh/disconnect → onDisconnect cleanup입니다. |
| 이 test가 증명하는 것 | cross-layer synchronous removal 뒤 phantom registered state/log가 없고 server가 process-wide failure로 멈추지 않음을 증명합니다. |
| 이 test가 증명하지 않는 것 | 모든 handler, compound iteration, native backend failure는 증명하지 않습니다. |
| test 성격 | deterministic application lifetime regression |
| 막는 회귀 | welcome 실패 후 removed client를 계속 등록 처리·로그하는 use-after-lifecycle 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `5edcafda8a4d`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `d48e1f1f8c04` — `fix(app): 응답 실패 뒤 명령 처리를 중단`. 원문 역할은 “Stops LIST, NAMES, and compound MODE after response failure.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `d48e1f1f8c04` — `fix(app): 응답 실패 뒤 명령 처리를 중단`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | DEBUG, LIFECYCLE, RISK |
| 원문 확정 역할 | Stops LIST, NAMES, and compound MODE after response failure. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Stops LIST, NAMES, and compound MODE after response failure.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 주요 handlers는 send 결과를 확인했지만 LIST/NAMES 반복과 compound MODE는 중간 response/broadcast 실패 뒤 다음 iteration/mode mutation을 계속할 수 있었습니다. |
| 주요 문제와 설계 판단 | LIST header/item, NAMES channel/item, MODE i/t/o/unknown response의 bool 결과를 즉시 검사해 false면 return하도록 했습니다. channel name과 target nick을 stable copy하고 broadcast 뒤 continuation 전에 actor/channel 존재를 확인했습니다. |
| 변경 파일·symbol | `src/ChannelCommands.cpp — handleList, handleNames, handleChannelMode` |
| state / ownership 변화 | compound command는 앞서 완료한 mutation을 유지하지만 actor lifecycle이 끝난 정확한 boundary 이후에는 cursor를 더 전진시키지 않습니다. |
| failure 또는 boundary | 첫 mode mutation의 broadcast가 sender disconnect를 유발하면 뒤 mode 문자는 처리되지 않습니다. 반복 reply도 첫 failure에서 끝나 stale iterator/reference를 재사용하지 않습니다. |
| 보장 / 비보장 | 보장: output failure 이후 command continuation은 즉시 중단됩니다.<br>비보장: transactional rollback을 제공하지 않으므로 failure 이전에 성공한 mutation은 남습니다. 이것이 선택된 semantics입니다. |
| 다음 관련 변화 | aee5edebe294가 `MODE #room +it`에서 +i는 유지되고 +t는 적용되지 않는 state assertion으로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `d48e1f1f8c04` | `src/ChannelCommands.cpp` | `handleList/handleNames` | 각 send failure에서 반복 즉시 return |
| `d48e1f1f8c04` | `src/ChannelCommands.cpp` | `handleChannelMode` | mode별 broadcast/send false 뒤 cursor continuation 중단 |

- 조사 방법: GitHub에서 `d48e1f1f8c04`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Fix chain

| 단계 | 역사 복원 |
| --- | --- |
| 기존 가정 | handler entry에서 revalidation하면 같은 command 안의 이후 send/iteration도 안전하다는 가정이었습니다. |
| 실제 failure / risk | 각 send가 actor를 제거할 수 있어 list loop나 mode cursor가 다음 transition을 삭제된 sender 권한으로 계속 수행했습니다. |
| root cause | failure propagation은 추가되었지만 모든 continuation point가 return 조건으로 연결되지 않았습니다. |
| 수정된 invariant / decision | 각 output boundary의 bool 결과를 그 자리에서 검사하고 compound operation은 non-transactional partial completion 후 즉시 종료합니다. |
| regression evidence | aee5edebe294가 첫 mode만 남고 later mode는 적용되지 않는 exact state를 검증합니다. |

#### 다음 연결

- 다음 Thread commit: `aee5edebe294` — `test(app): 연결 정리 뒤 모드 변경 중단 검증`. 원문 역할은 “Proves a compound MODE retains completed work but performs no later transition after sender removal.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `aee5edebe294` — `test(app): 연결 정리 뒤 모드 변경 중단 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, LIFECYCLE, CHANNEL_STATE |
| 원문 확정 역할 | Proves a compound MODE retains completed work but performs no later transition after sender removal. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Proves a compound MODE retains completed work but performs no later transition after sender removal.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | stop-after-failure semantics는 코드에 반영됐지만 compound MODE의 partial completion과 unrelated peer 보존을 state level에서 확인하지 않았습니다. |
| 주요 문제와 설계 판단 | white-box application fixture와 fd-specific update failure를 추가했습니다. 두 registered client가 있는 `#room`에서 sender를 operator로 두고 topic protection을 끈 뒤 `MODE #room +it`를 실행하며 첫 `+i` broadcast interest update에서 sender를 제거했습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — FakeEventManager::failNextUpdateFor, modeStopsAfterSenderCleanupTest` |
| state / ownership 변화 | test는 `_clients`와 `_channels`를 직접 구성·관찰하고 failure 전/후 channel flags와 두 client 존재를 각각 저장한 뒤 cleanup합니다. |
| failure 또는 boundary | `+i` mutation은 broadcast 전에 이미 완료되지만 update failure가 sender disconnect를 유발하므로 handler가 return해 다음 `+t` mutation은 실행하지 않습니다. |
| 보장 / 비보장 | 보장: channel은 peer 때문에 유지되고 `inviteOnly=true`, `topicProtected=false`, sender removed, peer present라는 non-transactional stop semantics가 고정됩니다.<br>비보장: white-box setup은 public IRC parser/authorization 전체를 통과하지 않으며 한 compound mode pattern만 검사합니다. |
| 다음 관련 변화 | 416efc91e580가 application lifetime suite를 두 OS와 sanitizer에서 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `aee5edebe294` | `tests/application_lifetime_test.cpp` | `modeStopsAfterSenderCleanupTest` | injected sender update failure 뒤 +i 유지, +t 미적용, sender만 제거 |
| `aee5edebe294` | `tests/application_lifetime_test.cpp` | `FakeEventManager::failNextUpdateFor` | 정확한 fd의 다음 interest update만 실패 |

- 조사 방법: GitHub에서 `aee5edebe294`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | actor lifecycle이 output boundary에서 끝나면 완료된 mutation은 유지하되 같은 compound command의 이후 transition은 실행하지 않아야 합니다. |
| 재현 failure / boundary | `MODE #room +it`의 첫 `+i` broadcast에서 sender interest update failure입니다. |
| test technique | white-box application state + fd-specific fake backend failure injection입니다. |
| 통과하는 production path | handleChannelMode → setInviteOnly → broadcastMode → Server.update failure/disconnect → false → handler return입니다. |
| 이 test가 증명하는 것 | partial completion, sender-only cleanup, peer/channel 보존과 later mode suppression을 exact state로 증명합니다. |
| 이 test가 증명하지 않는 것 | 모든 compound MODE 조합이나 public wire path, transactional rollback은 증명하지 않습니다. |
| test 성격 | deterministic command-continuation regression |
| 막는 회귀 | sender 제거 뒤 남은 mode 문자를 실행하거나 unrelated peer/channel을 함께 지우는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `aee5edebe294`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| erase-before-callback baseline | `7a6bc7e1276a` | Server::disconnect | recursive lookup은 removed state를 보지만 caller의 기존 pointer 문제는 남습니다. | 5dcd882f0763/928594ec160c |
| map/backend/fd atomic lifecycle | `5dcd882f0763` | start/accept/update rollback | map owner를 먼저 세우고 backend 실패를 erase/disconnect로 되돌립니다. | 928594ec160c |
| application send boundary | `728aaabc4012` | bool propagation + stable value/relookup | send가 client/channel graph를 지울 수 있음을 API에 반영합니다. | 5edcafda8a4d |
| stop-after-actor-removal | `d48e1f1f8c04` | LIST/NAMES/MODE continuation checks | completed mutation은 유지하고 later transition을 중단합니다. | aee5edebe294 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| single-thread callback은 pointer를 무효화하지 않음 | self-disconnect 뒤 use-after-free | 5dcd882f0763 | fd relookup after every callback/foreign operation | 928594ec160c |
| event add/update는 실패해도 local state와 일치 | map/backend/fd partial ownership | 5dcd882f0763 | map-first+erase rollback, update failure disconnect | 928594ec160c |
| send는 local queue append | sync disconnect가 client/channel/iterator를 제거 | 728aaabc4012 | bool 전파+stable copy+relookup | 5edcafda8a4d |
| 한 번 revalidate하면 compound command 끝까지 안전 | 첫 response failure 뒤 later mode mutation | d48e1f1f8c04 | 각 output boundary에서 즉시 return | aee5edebe294 |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| connection stable identity | raw pointer/reference | fd + Server map relookup | 5dcd882f0763 | path/symbol은 위 commit 증거표에 연결했습니다. |
| event registration rollback | 분산 exception path | Server start/accept/refreshInterest | 5dcd882f0763 | path/symbol은 위 commit 증거표에 연결했습니다. |
| send failure signal | void/hidden lifecycle | bool through Server→IrcApplication helper | 728aaabc4012 | path/symbol은 위 commit 증거표에 연결했습니다. |
| channel/client post-send state | borrowed pointer/iterator | stable copied key + container relookup | 728aaabc4012 | path/symbol은 위 commit 증거표에 연결했습니다. |
| compound failure semantics | 계속 처리 가능 | partial completion + immediate stop | d48e1f1f8c04 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: 여러 event/error/stop 경로에서 connection 제거 순서가 하나의 함수로 고정되지 않았습니다.
- 마지막 commit `aee5edebe294` 시점의 상태: channel은 peer 때문에 유지되고 `inviteOnly=true`, `topicProtected=false`, sender removed, peer present라는 non-transactional stop semantics가 고정됩니다.
- Thread 안에서 강화된 핵심 invariant: erase-before-callback baseline, map/backend/fd atomic lifecycle, application send boundary, stop-after-actor-removal.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: white-box setup은 public IRC parser/authorization 전체를 통과하지 않으며 한 compound mode pattern만 검사합니다. 416efc91e580가 application lifetime suite를 두 OS와 sanitizer에서 반복합니다.
- 최종 설명: `7a6bc7e1276a`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `aee5edebe294`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| server callback boundary | 5dcd882f0763 | `accept/read handler` | callback 후 fd로 map relookup | missing이면 즉시 return |
| event registration/update | 5dcd882f0763 | `start/accept/refreshInterest` | map/backend/fd 함께 성공 | add 실패 erase rollback, update 실패 disconnect |
| application send | 728aaabc4012 | `sendNumeric/broadcast helper` | Server.sendTo true면 authoritative state relookup | false 또는 missing state면 handler return |
| sync cleanup stack | 728aaabc4012 | `Connection queue/update failure` | Server::disconnect → onDisconnect → registry/channel cleanup | caller의 pre-send pointer/iterator는 무효 |
| compound command | d48e1f1f8c04 / aee5edebe294 | `MODE +it` | +i 완료 후 failure 없으면 +t | +i broadcast에서 sender 제거되면 +t 미실행 |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 08-reentrant-server-and-application-cleanup.md =====

===== BEGIN FILE: 09-verification-maturation-and-portability-enforcement.md =====
# Thread 09 — Verification maturation and portability enforcement

부제: 검증 성숙과 이식성 강제

## 1. Thread 목표

broad live-TCP smoke에서 exact public contract, injected transport/server/application failure tests, high-descriptor·slow-reader isolation, Linux/macOS·sanitizer CI로 verification architecture가 성숙하는 과정을 복원합니다.

Source에서 확정된 significance:

> Verification evolves from broad functional integration to exact public contracts, injected transport and event failures, application-lifetime tests, and real-process descriptor pressure. The CI commit then makes the same evidence repeatable for both readiness backends and under dynamic memory and undefined-behavior checks. No individual test proves absolute fairness or every failure class, but together they align the verification structure with the architecture's highest-risk boundaries.

이 문서의 source-confirmed 문장, commit map, SHA, subject, importance, tags와 원문 역할은 변경하지 않았습니다. 아래 학습 기록은 지정 branch의 각 exact SHA에서 확인한 code diff를 기준으로 작성했습니다.

## 2. 이 Thread를 이해하기 위한 핵심 질문

- broad smoke와 exact contract는 assertion precision과 failure localization이 어떻게 다른가?
- Connection, Server, Application lifetime test는 각각 어떤 production invariant와 injection seam을 사용하는가?
- real-process 160-peer/slow-reader test는 무엇을 증명하고 formal fairness·capacity는 왜 증명하지 않는가?
- Linux와 macOS matrix가 실제 epoll/kqueue backend를 같은 suite로 통과시키는가?
- ASan/UBSan job이 unit과 real-network binary 모두를 같은 instrumentation으로 빌드하는가?

### Source와 직접 연결되는 invariant

- 검증 구조는 architecture의 높은 위험 경계인 partial I/O, event/map ownership, application reentrancy, cross-platform backend와 대응해야 합니다.
- public contract test는 CLI, exact IRC frame, CRLF, numeric/order, shutdown, log ordering을 명시적으로 고정해야 합니다.
- rare failure는 timing accident가 아니라 injected operation/backend/limit로 deterministic하게 재현되어야 합니다.

### Source와 직접 연결되는 engineering difficulty

- rare transport/event failure를 deterministic하게 만들면서 production path 자체를 우회하지 않는 test seam 설계.
- 많은 fd와 slow receiver에서도 progress isolation을 보여 주되 benchmark/fairness 보장을 과장하지 않는 검증 범위 설정.
- 두 event backend와 sanitizer suite를 지속적 자동화로 강제하는 문제.

## 3. 완료 기준

- 각 test commit을 production invariant, boundary, technique, code path, proves/does-not-prove, test type, prevented regression으로 분류했습니다.
- smoke → exact contract → deterministic layer tests → stress/isolation → CI의 confidence 증가를 설명합니다.
- test double과 real socket/process를 선택한 이유를 failure 특성에 맞게 구분했습니다.
- CI matrix와 sanitizer flags가 Makefile target 전체에 전달되는지 확인했습니다.
- formal fairness, latency, maximum capacity, 모든 syscall failure class를 과장하지 않았습니다.

## 4. Commit map

| 순서 | SHA | Subject | Importance | Tags | 원문 확정 역할 |
| --- | --- | --- | --- | --- | --- |
| 1 | `6b4a7738a285` | `test(smoke): 실제 TCP 등록과 채널 흐름 검증` | A | VERIFICATION, INTEGRATION, RISK | Adds the first full real-TCP smoke flow. |
| 2 | `e5e6c57db80d` | `test(irc): 실행 조건과 오류 동작 계약 검증` | A | VERIFICATION, IRC_PROTOCOL, RISK | Replaces substring-oriented confidence with exact CLI, frame, and shutdown contracts. |
| 3 | `f34ab135c546` | `test(connection): 부분 송신과 대기열 경계 검증` | A | VERIFICATION, EVENT_IO, RISK | Adds deterministic Connection failure-state tests. |
| 4 | `928594ec160c` | `test(server): 연결 제거와 이벤트 등록 실패 경로 검증` | A | VERIFICATION, LIFECYCLE, RISK | Adds deterministic Server ownership and rollback tests. |
| 5 | `5edcafda8a4d` | `test(app): 작은 송신 한도에서 상태 정리 검증` | A | VERIFICATION, LIFECYCLE, RISK | Adds deterministic application cleanup verification. |
| 6 | `de1dd0fc30d0` | `test(event): 160개 연결과 느린 수신자 처리 공정성 검증` | A | VERIFICATION, EVENT_IO, RISK | Adds 160-peer readiness and slow-receiver isolation evidence. |
| 7 | `416efc91e580` | `ci: Linux·macOS 회귀와 새니타이저 자동화` | A | BUILD, VERIFICATION, RISK | Runs the complete suite on Linux and macOS and under ASan and UBSan. |

Commit 순서는 source에 정의된 순서이며 재정렬하지 않았습니다.

## 5. Commit별 학습 기록

각 기록은 해당 commit의 exact diff에서 확인한 파일·symbol·state와, 필요한 경우 parent/앞선 관련 SHA의 상태 차이를 기준으로 합니다. 후속 commit의 field나 test seam을 이전 commit에 소급하지 않았습니다.

### 5.1 `6b4a7738a285` — `test(smoke): 실제 TCP 등록과 채널 흐름 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, INTEGRATION, RISK |
| 원문 확정 역할 | Adds the first full real-TCP smoke flow. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds the first full real-TCP smoke flow.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | parser·registry·registration·channel 기능이 실제 process/socket 안에서 함께 동작한다는 통합 증거가 없었습니다. |
| 주요 문제와 설계 판단 | Makefile test/smoke, shell runner, Python peer를 추가해 loopback server를 실행하고 다중 client 시나리오를 수행했습니다. |
| 변경 파일·symbol | `Makefile — test, smoke`<br>`tests/irc_smoke.sh`<br>`tools/irc_smoke_client.py` |
| state / ownership 변화 | test harness가 process PID, free port, temp log, peer sockets와 수신 line buffer를 소유하고 EXIT trap으로 정리합니다. |
| failure 또는 boundary | wrong password, fragmented input, case-insensitive collision, channel/mode/message/departure 오류를 실제 wire에서 관찰합니다. |
| 보장 / 비보장 | 보장: transport framing부터 application/channel/output queue까지 주요 정상·일부 오류 경로가 한 process에서 조합됩니다.<br>비보장: assertion은 주로 substring 중심이어서 exact 전체 frame/order/CRLF와 rare syscall failure, capacity/fairness를 증명하지 않습니다. |
| 다음 관련 변화 | e5e6c57db80d가 exact public contract로 정밀도를 높이고 계층별 deterministic tests가 후속됩니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `6b4a7738a285` | `tests/irc_smoke.sh` | `runner and cleanup trap` | 실제 server process 기동·readiness·로그·종료 관리 |
| `6b4a7738a285` | `tools/irc_smoke_client.py` | `IrcPeer and scenario` | fragmented write와 registration/channel/messaging 시나리오 |

- 조사 방법: GitHub에서 `6b4a7738a285`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | Connection framing, parser, registration, nickname index, channel state와 queued send가 실제 process에서 결합됩니다. |
| 재현 failure / boundary | wrong password, fragmented PING, nickname collision, JOIN/TOPIC/PRIVMSG/INVITE/MODE/KICK/PART/QUIT입니다. |
| test technique | compiled server + real loopback TCP + shell process harness + Python peer의 broad end-to-end smoke입니다. |
| 통과하는 production path | process startup → Server event loop → Connection framing → parser → IrcApplication → registry/channel → output queue입니다. |
| 이 test가 증명하는 것 | 대표 정상·오류 흐름이 실제 socket에서 연결되어 진행됨을 증명합니다. |
| 이 test가 증명하지 않는 것 | exact 모든 frame, 모든 ordering, rare send/event failure, formal capacity·fairness는 증명하지 않습니다. |
| test 성격 | broad integration smoke |
| 막는 회귀 | packet boundary를 command boundary로 오인하거나 identity/channel fan-out 통합이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `6b4a7738a285`
- repository에 정의된 실행 명령: `make test` 또는 `make smoke`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`. 원문 역할은 “Replaces substring-oriented confidence with exact CLI, frame, and shutdown contracts.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.2 `e5e6c57db80d` — `test(irc): 실행 조건과 오류 동작 계약 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, IRC_PROTOCOL, RISK |
| 원문 확정 역할 | Replaces substring-oriented confidence with exact CLI, frame, and shutdown contracts. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Replaces substring-oriented confidence with exact CLI, frame, and shutdown contracts.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | broad smoke는 주요 기능 조합을 보여 주었지만 CLI 실패, exact frame/CRLF/numeric order, timeout/rate/metrics/shutdown/log ordering을 정밀하게 고정하지 못했습니다. |
| 주요 문제와 설계 판단 | `tests/irc_contract.py` 중심의 executable contract suite를 추가해 process startup 오류, runtime options, exact/regex wire frames, protection responses, metrics field order, graceful shutdown 및 log order를 manifest와 assertion으로 검사했습니다. |
| 변경 파일·symbol | `tests/irc_contract.py — CLI and wire contract checks`<br>`Makefile — contract/test integration` |
| state / ownership 변화 | test가 server process, sockets, manifest, logs와 expected frames를 소유하며 dynamic fd/token 같은 값만 regex로 허용합니다. |
| failure 또는 boundary | invalid/missing option과 runtime timeout/rate/queue/shutdown boundary를 subprocess return code, exact line, close 및 log event로 관찰합니다. |
| 보장 / 비보장 | 보장: public CLI·wire·shutdown·observability contract의 성공/실패 shape와 ordering을 broad substring 수준보다 정확히 고정합니다.<br>비보장: rare syscall return, event add/update failure, pointer lifetime, formal fairness는 직접 주입하지 않습니다. |
| 다음 관련 변화 | b6c10bc51937/5d1286620994가 parser width 결함을 수정·검증하고 f34ab/928/5ed가 low-level deterministic failure를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `e5e6c57db80d` | `tests/irc_contract.py` | `CLI checks and record_exact/record_regex` | 정적 frame은 exact, 동적 값만 제한된 regex로 검증 |
| `e5e6c57db80d` | `tests/irc_contract.py` | `wire/shutdown/log scenarios` | timeout·rate·metrics·shutdown public ordering |

- 조사 방법: GitHub에서 `e5e6c57db80d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 문서화된 CLI syntax, IRC frame/CRLF/numeric ordering, protection response, metrics와 shutdown/log ordering이 public boundary와 일치합니다. |
| 재현 failure / boundary | missing/invalid args, registration/heartbeat/rate/output limits, exact numerics, signal shutdown과 final ERROR입니다. |
| test technique | compiled server subprocess + real loopback TCP + exact/limited-regex assertions + captured logs/manifest입니다. |
| 통과하는 production path | argv parser → Server/IrcApplication runtime policy → wire output → signal flag → shutdown/drain → logs입니다. |
| 이 test가 증명하는 것 | 외부 사용자가 관찰하는 성공·실패 frame과 ordering을 정확히 고정합니다. |
| 이 test가 증명하지 않는 것 | 모든 syscall/event failure, memory safety, formal latency·fairness, 내부 lifetime 안전성은 증명하지 않습니다. |
| test 성격 | exact process and wire contract integration |
| 막는 회귀 | CLI가 sign/whitespace를 허용하거나 numeric/CRLF/order/shutdown log contract가 변하는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `e5e6c57db80d`
- repository에 정의된 실행 명령: `python3 tests/irc_contract.py ./irc-relay-server` 또는 해당 Make target
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`. 원문 역할은 “Adds deterministic Connection failure-state tests.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.3 `f34ab135c546` — `test(connection): 부분 송신과 대기열 경계 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, EVENT_IO, RISK |
| 원문 확정 역할 | Adds deterministic Connection failure-state tests. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds deterministic Connection failure-state tests.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | Connection fix의 rare syscall/overflow branch는 real socket timing에 맡기면 반복적으로 재현하기 어려웠습니다. |
| 주요 문제와 설계 판단 | `SendStep`과 `ScriptedSender`가 Bytes/Error/Zero sequence를 반환하는 unit binary를 추가하고 production `Connection::flushPending()`에 주입했습니다. Makefile에 `connection-test`를 연결했습니다. |
| 변경 파일·symbol | `tests/connection_test.cpp — SendStep, ScriptedSender and five test groups`<br>`Makefile — connection-test` |
| state / ownership 변화 | script index, captured bytes와 Connection pending/offset/close state를 각 단계 후 동시에 assertion합니다. |
| failure 또는 boundary | EINTR→partial→EAGAIN→partial→EWOULDBLOCK→final sequence, zero then progress, EPIPE, requested size보다 큰 return, size_t max arithmetic와 exact CRLF limit을 결정적으로 재현합니다. |
| 보장 / 비보장 | 보장: successful bytes는 `abcdefghijkl` 순서로 정확히 한 번 전달되고 retryable/terminal/invalid outcome은 아직 안 보낸 bytes를 소비하지 않으며 rejected append는 queue를 변경하지 않습니다.<br>비보장: fake sender는 kernel buffer, actual readiness notification, TCP segmentation이나 peer scheduling을 검증하지 않습니다. |
| 다음 관련 변화 | de1dd0fc30d0가 real process/slow reader에서 connection-local queue isolation을 보완하고 416efc91e580가 sanitizer로 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `f34ab135c546` | `tests/connection_test.cpp` | `testLimitArithmetic/testHardLimitAndLineTerminator` | size_t max와 CRLF 포함 exact-limit mutation |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInterruptedAndPartialSendState` | scripted retry마다 pending offset과 exact output |
| `f34ab135c546` | `tests/connection_test.cpp` | `testZeroAndTerminalErrorPreservePendingBytes` | zero/EPIPE가 suffix를 소비하지 않음 |
| `f34ab135c546` | `tests/connection_test.cpp` | `testInvalidSendCountCannotAdvanceOffset` | oversize return이 offset을 손상시키지 않음 |

- 조사 방법: GitHub에서 `f34ab135c546`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | logical pending limit은 overflow하지 않고, offset은 검증된 successful byte만큼만 전진하며 unsent suffix가 exact order로 보존됩니다. |
| 재현 failure / boundary | size_t max, CRLF exact limit, EINTR, partial, EAGAIN/EWOULDBLOCK, zero, EPIPE, impossible oversize return입니다. |
| test technique | injected send operation을 사용하는 deterministic scripted unit test입니다. |
| 통과하는 production path | queueRaw/queueLine → overflow-safe admission → flushPending → injected send result → offset/close/pending query입니다. |
| 이 test가 증명하는 것 | rare outcome sequence에서 exact byte accounting과 mutation-free rejection을 직접 증명합니다. |
| 이 test가 증명하지 않는 것 | real kernel readiness, TCP delivery acknowledgement, end-to-end fairness는 증명하지 않습니다. |
| test 성격 | deterministic transport boundary regression |
| 막는 회귀 | partial retry 중 byte 중복·유실, unsigned wrap, invalid return으로 offset이 깨지는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `f34ab135c546`
- repository에 정의된 실행 명령: `make connection-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`. 원문 역할은 “Adds deterministic Server ownership and rollback tests.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.4 `928594ec160c` — `test(server): 연결 제거와 이벤트 등록 실패 경로 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, LIFECYCLE, RISK |
| 원문 확정 역할 | Adds deterministic Server ownership and rollback tests. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds deterministic Server ownership and rollback tests.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | server lifetime fix는 있었지만 add/update failure와 callback self-removal을 real backend/timing에 의존하지 않고 재현하는 regression suite가 없었습니다. |
| 주요 문제와 설계 판단 | `FakeEventManager`에 interest map, queued events, fail-next-add/update seam을 두고 loopback ClientSocket과 `Server::pollOnce()`를 결합한 `server_lifetime_test`를 추가했습니다. |
| 변경 파일·symbol | `tests/server_lifetime_test.cpp — FakeEventManager, ClientSocket, five tests`<br>`Makefile — unit/server lifetime target` |
| state / ownership 변화 | fake backend가 등록 fd와 호출 횟수를 관찰하고 test가 real accepted socket과 Server connection count를 함께 assertion합니다. |
| failure 또는 boundary | (1) client add 실패 rollback, (2) connect callback self-disconnect+throw, (3) line callback self-disconnect+throw, (4) write interest update 실패, (5) outbound queue rejection close를 각각 결정적으로 주입합니다. |
| 보장 / 비보장 | 보장: 각 case 뒤 connection map과 fake backend에 client fd가 남지 않고 callback 예외가 cleanup을 무효화하지 않으며 queue/update failure가 false를 반환합니다.<br>비보장: fake backend는 epoll/kqueue native semantics 자체를 검증하지 않고 callback 이후 application channel/client graph도 직접 검사하지 않습니다. |
| 다음 관련 변화 | 728aaabc4012/5edcafda8a4d가 send failure가 application state와 log까지 정리되는 경로를 추가합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `registrationRollbackTest` | injected add failure 뒤 map/backend가 모두 비어 있음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `connectCallbackLifetimeTest/lineCallbackLifetimeTest` | callback self-disconnect와 throw 뒤 stale access 없음 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `interestUpdateRollbackTest` | update failure가 send false와 map/backend removal로 수렴 |
| `928594ec160c` | `tests/server_lifetime_test.cpp` | `queueLimitCloseTest` | queue rejection close가 event/map entry를 남기지 않음 |

- 조사 방법: GitHub에서 `928594ec160c`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | connection map, event registration, descriptor lifetime이 함께 전진·정리되고 callback 뒤 stale pointer를 재사용하지 않아야 합니다. |
| 재현 failure / boundary | event add/update exception, connect/line callback self-disconnect+throw, outbound queue rejection입니다. |
| test technique | injected FakeEventManager + real loopback accepted socket + direct `pollOnce()` deterministic lifetime unit/integration입니다. |
| 통과하는 production path | accept/read/send → Server map/backend → callback/refreshInterest → disconnect/erase/destruction입니다. |
| 이 test가 증명하는 것 | server-level ownership rollback과 callback relookup rule을 timing accident 없이 검증합니다. |
| 이 test가 증명하지 않는 것 | native epoll/kqueue kernel behavior, application aggregate cleanup, 모든 OS resource failure는 증명하지 않습니다. |
| test 성격 | deterministic server lifetime regression |
| 막는 회귀 | self-disconnect use-after-free와 partial event registration/interest update leak을 막습니다. |

실행 기록:

- 실행 SHA: `928594ec160c`
- repository에 정의된 실행 명령: `make unit` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`. 원문 역할은 “Adds deterministic application cleanup verification.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.5 `5edcafda8a4d` — `test(app): 작은 송신 한도에서 상태 정리 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, LIFECYCLE, RISK |
| 원문 확정 역할 | Adds deterministic application cleanup verification. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds deterministic application cleanup verification.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | application revalidation fix가 welcome 응답 실패로 connection이 제거되는 실제 cross-layer stack에서 state/log를 올바르게 정리한다는 test가 없었습니다. |
| 주요 문제와 설계 판단 | FakeEventManager와 captured stderr를 사용하는 `application_lifetime_test`를 추가하고 Server `maxPendingBytes=1`로 설정해 NICK/USER 등록 중 첫 welcome frame admission을 강제로 실패시켰습니다. |
| 변경 파일·symbol | `tests/application_lifetime_test.cpp — CapturedStderr, FakeEventManager, registrationQueueFailureTest`<br>`Makefile — application-test` |
| state / ownership 변화 | real accepted socket과 application callbacks를 사용하되 tiny queue limit가 deterministic disconnect seam이며 test가 server connection count, running state와 log text를 관찰합니다. |
| failure 또는 boundary | welcome queue failure는 Server disconnect와 application onDisconnect cleanup을 동기 실행합니다. |
| 보장 / 비보장 | 보장: 실패 뒤 connection count가 0이고 server는 계속 실행되며 제거된 client를 `event=client_registered`로 기록하지 않습니다.<br>비보장: 이 최초 application test는 channel/mode의 multi-step continuation을 검사하지 않습니다. |
| 다음 관련 변화 | d48e1f1f8c04/aee5edebe294가 LIST/NAMES/MODE continuation과 부분 완료 semantics를 추가로 고정합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `5edcafda8a4d` | `tests/application_lifetime_test.cpp` | `registrationQueueFailureTest` | 1-byte limit → welcome send failure → connection cleanup, server running, registered log 부재 |

- 조사 방법: GitHub에서 `5edcafda8a4d`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | registration response failure로 actor가 제거되면 application state와 lifecycle log가 제거 결과와 일치해야 합니다. |
| 재현 failure / boundary | `maxPendingBytes=1`에서 001 welcome frame queue rejection입니다. |
| test technique | tiny real Server queue limit + fake event backend + real loopback connection + captured stderr입니다. |
| 통과하는 production path | NICK/USER → maybeRegister → sendNumeric → Server.sendTo → queue rejection/refresh/disconnect → onDisconnect cleanup입니다. |
| 이 test가 증명하는 것 | cross-layer synchronous removal 뒤 phantom registered state/log가 없고 server가 process-wide failure로 멈추지 않음을 증명합니다. |
| 이 test가 증명하지 않는 것 | 모든 handler, compound iteration, native backend failure는 증명하지 않습니다. |
| test 성격 | deterministic application lifetime regression |
| 막는 회귀 | welcome 실패 후 removed client를 계속 등록 처리·로그하는 use-after-lifecycle 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `5edcafda8a4d`
- repository에 정의된 실행 명령: `make application-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `de1dd0fc30d0` — `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`. 원문 역할은 “Adds 160-peer readiness and slow-receiver isolation evidence.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.6 `de1dd0fc30d0` — `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | VERIFICATION, EVENT_IO, RISK |
| 원문 확정 역할 | Adds 160-peer readiness and slow-receiver isolation evidence. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Adds 160-peer readiness and slow-receiver isolation evidence.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | deterministic unit/lifetime tests는 rare branch를 고정했지만 많은 live fd와 실제 읽지 않는 recipient 아래 다른 connection progress를 보여 주는 process-level 증거가 없었습니다. |
| 주요 문제와 설계 판단 | `irc_event_fairness.py`와 Make `event-test`를 추가했습니다. 첫 scenario는 160개 non-blocking peer가 각기 다른 PING을 보내 exact PONG을 공통 deadline 안에 받게 하고, 둘째는 receive buffer 1024인 slow peer에게 4096개의 400-byte PRIVMSG를 보내는 동안 unrelated probe의 PING/PONG이 15초 안에 진행되는지 확인했습니다. |
| 변경 파일·symbol | `tests/irc_event_fairness.py — check_many_connections, check_slow_receiver_isolation, process harness`<br>`Makefile — event-test/test integration` |
| state / ownership 변화 | Python selector가 160 peer socket과 expected PONG/buffer를 관리하고 separate slow/sender/probe sockets가 connection-local pressure를 만듭니다. |
| failure 또는 boundary | 어느 peer든 PONG을 못 받거나 slow receiver 때문에 probe가 멈추거나 server가 SIGTERM 후 정상 종료하지 못하면 captured server output과 함께 실패합니다. |
| 보장 / 비보장 | 보장: 160 live descriptors에서 readiness fan-out이 진행되고 한 unread recipient의 pending pressure가 unrelated connection의 기본 progress를 막지 않는다는 관찰 증거를 제공합니다.<br>비보장: formal fairness, latency SLO, 최대 capacity, 모든 fd limit과 scheduler adversary를 증명하지 않으며 한 구성·한 workload의 regression입니다. |
| 다음 관련 변화 | 416efc91e580가 이 real-process suite를 Linux epoll과 macOS kqueue 모두에서 자동 반복합니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `check_many_connections` | 160 exact PONG을 selector와 monotonic deadline으로 수집 |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `check_slow_receiver_isolation` | slow recipient flood 중 unrelated probe progress |
| `de1dd0fc30d0` | `tests/irc_event_fairness.py` | `stop_server` | SIGTERM bounded normal exit contract |

- 조사 방법: GitHub에서 `de1dd0fc30d0`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 높은 fd 수와 connection-local backpressure에서도 unrelated connection의 event-loop progress가 유지되어야 합니다. |
| 재현 failure / boundary | 160 simultaneous peers와 한 unread receiver에 대한 4096×400-byte message flood입니다. |
| test technique | compiled real server process + loopback sockets + Python selectors + small receive buffer + monotonic deadlines입니다. |
| 통과하는 production path | many accept registrations → epoll/kqueue wait → per-connection read/queue/write → slow pending queue while probe PING/PONG progresses입니다. |
| 이 test가 증명하는 것 | 해당 workload에서 fd>select 전통 한계 수준과 slow-reader isolation을 실제 process로 보여 줍니다. |
| 이 test가 증명하지 않는 것 | formal fairness, worst-case latency, maximum connection capacity, 모든 backpressure pattern은 증명하지 않습니다. |
| test 성격 | real-process stress and progress-isolation regression |
| 막는 회귀 | 고정 배열/fd 한계, 한 느린 recipient가 event loop 전체를 정지시키는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `de1dd0fc30d0`
- repository에 정의된 실행 명령: `make event-test` 또는 `make test`
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 다음 Thread commit: `416efc91e580` — `ci: Linux·macOS 회귀와 새니타이저 자동화`. 원문 역할은 “Runs the complete suite on Linux and macOS and under ASan and UBSan.”입니다. 현재 commit의 state/API/미해결 위험은 위의 후속 연결 항목처럼 이어집니다.

### 5.7 `416efc91e580` — `ci: Linux·macOS 회귀와 새니타이저 자동화`

| 항목 | 값 |
| --- | --- |
| Importance | A |
| Tags | BUILD, VERIFICATION, RISK |
| 원문 확정 역할 | Runs the complete suite on Linux and macOS and under ASan and UBSan. |
| 학습 깊이 | 주요 subsystem·boundary·failure path·integration point를 path와 symbol 기준으로 복원합니다. |

#### Source에서 확정된 역할

> Runs the complete suite on Linux and macOS and under ASan and UBSan.

#### 해당 SHA의 역사 복원

| 학습 항목 | 학습 기록 |
| --- | --- |
| 직전 경계/상태 | 모든 test target이 로컬에서 실행 가능했지만 epoll/kqueue 양쪽과 sanitizer instrumentation 아래 반복된다는 repository-level enforcement가 없었습니다. |
| 주요 문제와 설계 판단 | GitHub Actions workflow를 추가해 `ubuntu-latest`와 `macos-latest` matrix에서 `make -j2`와 `make test`를 실행하고, 별도 Ubuntu job에서 ASan+UBSan·frame pointer·leak detection·halt-on-error flags로 동일 binary/tests를 build/run하도록 했습니다. |
| 변경 파일·symbol | `.github/workflows/ci.yml — platform-regression and linux-sanitizers jobs` |
| state / ownership 변화 | matrix OS가 default EventManager factory를 통해 각각 epoll/kqueue backend를 선택하고 sanitizer CXXFLAGS가 production binary와 unit/network test builds에 동일하게 전달됩니다. |
| failure 또는 boundary | 어느 OS build/test 또는 sanitizer report라도 job을 실패시키며 matrix는 fail-fast=false로 다른 platform 결과도 수집합니다. |
| 보장 / 비보장 | 보장: complete Make test graph를 Linux/macOS 및 Linux ASan/UBSan에서 push/PR마다 반복하도록 자동화합니다.<br>비보장: CI runner가 지원하지 않는 OS/compiler, TSan, exhaustive race/fuzz/formal verification은 포함하지 않습니다. workflow 정의의 존재는 현재 run 성공 이력 자체와 동일하지 않습니다. |
| 다음 관련 변화 | Thread 09의 broad→exact→injected→stress evidence를 두 readiness backend와 dynamic checks에 연결하는 최종 enforcement입니다. |

#### 코드 증거

| SHA | File path | Symbol / 함수 | 이 증거가 뒷받침하는 결론 |
| --- | --- | --- | --- |
| `416efc91e580` | `.github/workflows/ci.yml` | `platform-regression matrix` | ubuntu/macos에서 build와 complete make test 실행 |
| `416efc91e580` | `.github/workflows/ci.yml` | `linux-sanitizers job` | ASan+UBSan/leak options를 build와 test 전체에 전달 |

- 조사 방법: GitHub에서 `416efc91e580`의 exact commit diff와 변경 파일을 확인했습니다.
- runtime 결과로 표시한 내용은 없으며 위 표는 code inspection으로 확인한 사실입니다.

#### Test / verification 학습 기록

| 구분 | 역사 복원 기록 |
| --- | --- |
| 대상 production invariant | 동일 regression graph가 Linux epoll, macOS kqueue, ASan/UBSan instrumentation에서 반복 가능해야 합니다. |
| 재현 failure / boundary | cross-platform compilation/warnings, complete unit/network tests, address/undefined/leak failures입니다. |
| test technique | GitHub Actions OS matrix와 separate sanitizer CI job입니다. |
| 통과하는 production path | checkout → make production/tests → make test → backend factory/network process/unit binaries입니다. |
| 이 test가 증명하는 것 | workflow가 두 OS와 sanitizer에서 같은 suite를 실행하도록 구성되었음을 코드로 증명합니다. |
| 이 test가 증명하지 않는 것 | 이 작업 환경에서 실제 CI run이 성공했다는 사실, TSan/Windows/BSD와 모든 UB class는 증명하지 않습니다. |
| test 성격 | continuous cross-platform and dynamic-analysis enforcement |
| 막는 회귀 | 한 backend에서만 compile/test되거나 memory/UB defect가 일반 build에서 숨는 회귀를 막습니다. |

실행 기록:

- 실행 SHA: `416efc91e580`
- repository에 정의된 실행 명령: workflow의 `make -j2`, `make test`, sanitizer CXXFLAGS build/test
- 현재 작업 환경 실행 여부: **미실행**
- 이유: 현재 런타임에서 외부 DNS가 차단되어 지정 branch의 local checkout을 만들 수 없었습니다. GitHub 연결 도구로 해당 SHA의 production/test diff와 test mechanism은 확인했지만, binary/test command의 성공 결과를 만들거나 추정하지 않았습니다.
- 실행 결과: 기록 없음

#### 다음 연결

- 이 commit은 이 Thread의 마지막 항목입니다. 아래 Invariant ledger와 Thread 최종 상태에서 전체 변화를 연결합니다.

## 6. Invariant ledger

| Invariant | 처음 도입 | 강화/적용 | 학습자 최종 기록 | Fix/Test 연결 |
| --- | --- | --- | --- | --- |
| broad live integration | `6b4a7738a285` | real process/TCP smoke | 여러 subsystem의 조합을 넓게 확인하지만 assertion precision은 낮습니다. | e5e6c57db80d |
| exact public contract | `e5e6c57db80d` | CLI/wire/shutdown/log exact assertions | 외부 behavior와 ordering을 정밀하게 고정합니다. | low-level rare failure는 별도 seam 필요 |
| deterministic layer failures | `f34ab135c546` | 928594ec160c, 5edcafda8a4d | send operation, event backend, queue limit로 rare branch를 재현합니다. | 각 architecture risk와 직접 대응 |
| real-process pressure isolation | `de1dd0fc30d0` | 160 peers + slow receiver | 실제 descriptor 수와 backpressure에서 progress evidence를 추가합니다. | formal fairness/capacity는 아님 |
| cross-platform/dynamic enforcement | `416efc91e580` | Linux/macOS matrix + ASan/UBSan | complete make test graph를 두 backend와 sanitizer에 반복합니다. | 현재 run 성공 자체는 workflow inspection과 구분 |

도입과 완성을 같은 의미로 쓰지 않았습니다. 후속 fix가 이전 SHA의 사실을 바꾸지 않도록 각 시점의 보장과 부족함을 분리했습니다.

## 7. Failure → Fix → Test 연결

| 기존 가정 | 실제 failure / risk | Fix 또는 기반 변화 | 수정된 decision / semantics | Test 연결 |
| --- | --- | --- | --- | --- |
| broad substring smoke만 충분 | exact frame/order와 rare branch가 숨음 | e5e6c57db80d + layer tests | contract와 deterministic injection 분리 | f34ab/928/5ed |
| real socket에서 rare syscall/event failure를 기다림 | timing-dependent·불안정 재현 | SendOperation/FakeEventManager/tiny queue | production path에 controlled failure 주입 | f34ab/928/5ed |
| 단일 OS local success면 portable | epoll/kqueue 또는 sanitizer에서만 나타나는 결함 | 416efc91e580 | OS matrix와 same-suite instrumentation | CI jobs |

## 8. Ownership / state / responsibility 변화

| 책임 또는 상태 | 이전 상태 | 이후 authoritative 위치 | 관련 commit | 확인 결과 |
| --- | --- | --- | --- | --- |
| process lifecycle fixture | 없음 | shell/Python smoke harness | 6b4a7738a285 | path/symbol은 위 commit 증거표에 연결했습니다. |
| public contract oracle | substring expectation | exact/regex manifest | e5e6c57db80d | path/symbol은 위 commit 증거표에 연결했습니다. |
| transport failure script | kernel timing | ScriptedSender | f34ab135c546 | path/symbol은 위 commit 증거표에 연결했습니다. |
| server backend failure control | native backend | FakeEventManager | 928594ec160c | path/symbol은 위 commit 증거표에 연결했습니다. |
| application failure trigger | 우연한 send pressure | maxPendingBytes=1 + captured log | 5edcafda8a4d | path/symbol은 위 commit 증거표에 연결했습니다. |
| portable enforcement | developer local environment | GitHub Actions matrix/sanitizer | 416efc91e580 | path/symbol은 위 commit 증거표에 연결했습니다. |

## 9. Thread 최종 상태

- 시작 직전 상태: parser·registry·registration·channel 기능이 실제 process/socket 안에서 함께 동작한다는 통합 증거가 없었습니다.
- 마지막 commit `416efc91e580` 시점의 상태: complete Make test graph를 Linux/macOS 및 Linux ASan/UBSan에서 push/PR마다 반복하도록 자동화합니다.
- Thread 안에서 강화된 핵심 invariant: broad live integration, exact public contract, deterministic layer failures, real-process pressure isolation, cross-platform/dynamic enforcement.
- 남은 한계 또는 후속 Thread에서 보강되는 부분: CI runner가 지원하지 않는 OS/compiler, TSan, exhaustive race/fuzz/formal verification은 포함하지 않습니다. workflow 정의의 존재는 현재 run 성공 이력 자체와 동일하지 않습니다. Thread 09의 broad→exact→injected→stress evidence를 두 readiness backend와 dynamic checks에 연결하는 최종 enforcement입니다.
- 최종 설명: `6b4a7738a285`에서 시작한 책임은 commit map 순서대로 상태 owner, failure branch와 cleanup ordering을 추가했고 `416efc91e580`에서 이 Thread가 정한 검증 또는 운영 상태에 도달합니다. 이전 시점의 미보장은 후속 fix/test SHA를 별도로 연결했으며 final HEAD 상태를 과거 commit에 소급하지 않았습니다.

## 10. 최종 architecture 또는 execution flow 정리

| 단계 | SHA | Caller / callee / state owner | 정상 transition | failure / cleanup transition |
| --- | --- | --- | --- | --- |
| broad integration | 6b4a7738a285 | `make smoke/test` | real server + peers + substring assertions | 실패 시 process/log cleanup |
| exact contract | e5e6c57db80d | `irc_contract.py` | exact frame/CLI/shutdown/log assertions | dynamic value만 limited regex |
| deterministic units | f34ab135c546 / 928594ec160c / 5edcafda8a4d | `injected sender/backend/limit` | production Connection/Server/App paths | rare branch를 fixed outcome으로 재현 |
| stress isolation | de1dd0fc30d0 | `irc_event_fairness.py` | 160 fd exact PONG + slow-reader/probe | deadline miss 또는 abnormal exit는 failure |
| CI enforcement | 416efc91e580 | `GitHub Actions` | ubuntu/macos make test + Linux ASan/UBSan | 한 job failure가 regression signal |

## 11. 학습 완료 자가 점검

- [x] Commit map의 모든 SHA, subject, importance, tags를 source와 대조했습니다.
- [x] 모든 commit의 exact SHA diff와 변경 파일·symbol을 확인했습니다.
- [x] final HEAD의 함수나 field를 과거 commit 설명에 소급하지 않았습니다.
- [x] S commit은 architecture/invariant, failure, ownership/lifecycle, 후속 fix/test까지 기록했습니다.
- [x] A commit은 주요 subsystem/boundary/failure path와 설계 판단을 기록했습니다.
- [x] B commit은 Thread 흐름에서 맡는 구현 역할과 state 변화를 기록했습니다.
- [x] fix commit의 기존 가정, root cause, 수정 invariant, regression test를 연결했습니다.
- [x] test commit의 production path, technique, 증명/비증명 범위를 구분했습니다.
- [x] Invariant ledger와 responsibility 표를 path/symbol 증거에 연결했습니다.
- [ ] production build/test command를 이 작업 환경에서 직접 실행했습니다. local checkout을 만들 수 없어 code/test inspection과 실행 결과를 분리했으며 pass 결과를 기록하지 않았습니다.
===== END FILE: 09-verification-maturation-and-portability-enforcement.md =====

===== BEGIN FILE: README.md =====
# ft_irc Development Thread 학습 골격

## 목적

이 문서 세트는 완성형 프로젝트 해설서가 아닙니다. 학습자가 실제 commit history와 각 SHA 시점의 코드를 직접 읽고 증거를 기록하면서 설계, 구현, 실패 처리, 수정, 검증의 발전 과정을 복원하기 위한 골격입니다.

문서 구조와 commit 관계는 `commit-importance.md`의 Development Threads를 따르며, commit의 구현 의도와 failure handling 확인 항목은 `commit-bodies.md`를 기준으로 작성되었습니다.

## 권장 학습 순서

| 순서 | Thread 문서 | 학습 초점 |
| --- | --- | --- |
| 1 | [Portable readiness and non-blocking transport](01-portable-readiness-and-nonblocking-transport.md) | 이벤트 준비 상태와 논블로킹 전송 |
| 2 | [Protocol boundary, identity, and registration](02-protocol-boundary-identity-and-registration.md) | 프로토콜 경계, 식별자와 등록 |
| 3 | [Channel authority, fan-out, and cleanup](03-channel-authority-fanout-and-cleanup.md) | 채널 권한, 팬아웃과 정리 |
| 4 | [Operational protections and controlled shutdown](04-operational-protections-and-controlled-shutdown.md) | 운영 보호와 제어된 종료 |
| 5 | [Strict runtime configuration boundaries](05-strict-runtime-configuration-boundaries.md) | 엄격한 런타임 구성 경계 |
| 6 | [Heartbeat liveness correctness](06-heartbeat-liveness-correctness.md) | 하트비트 생존성 정확성 |
| 7 | [Output-queue correctness under partial failure](07-output-queue-correctness-under-partial-failure.md) | 부분 실패에서의 송신 대기열 정확성 |
| 8 | [Reentrant server and application cleanup](08-reentrant-server-and-application-cleanup.md) | 재진입 가능한 서버와 애플리케이션 정리 |
| 9 | [Verification maturation and portability enforcement](09-verification-maturation-and-portability-enforcement.md) | 검증 성숙과 이식성 강제 |

위 순서는 source에 정의된 Development Thread 순서입니다. 같은 commit이 여러 문서에 등장해도 제거하지 않습니다. 각 문서에서 서로 다른 invariant와 학습 관점으로 다시 확인합니다.

## Thread 문서 사용법

1. 문서의 Commit map에서 현재 확인할 SHA와 importance를 확인합니다.
2. repository를 해당 SHA로 이동한 뒤 그 시점의 변경 파일과 실제 symbol을 찾습니다.
3. 문서에 미리 적힌 source-confirmed role과 implementation anchor를 기준으로 코드 증거를 수집합니다.
4. 학습 기록란에는 path, symbol, caller/callee, state mutation 순서, failure branch, cleanup 경로를 직접 채웁니다.
5. fix commit은 기존 가정 → 실제 위험 → root cause → 수정 invariant → 수정 코드 → regression test 순서로 연결합니다.
6. test commit은 production invariant, 재현 boundary, technique, 통과하는 production path, 증명/비증명 범위를 분리합니다.
7. Thread 마지막에 invariant ledger, failure-fix-test, responsibility 변화, execution flow를 자신의 코드 증거로 완성합니다.

## 해당 SHA 코드 확인 원칙

- 반드시 현재 학습 중인 commit SHA의 tree를 확인합니다.
- 기본 비교는 `<SHA>^`와 `<SHA>`입니다. Thread에서 직전 관련 SHA가 따로 제시되면 두 시점도 함께 비교합니다.
- final HEAD의 코드를 과거 commit 설명에 소급 적용하지 않습니다.
- 후속 fix에서 생긴 함수, field, test seam을 이전 commit에 존재했던 것처럼 기록하지 않습니다.
- source가 확정하지 않은 invariant를 새 사실처럼 추가하지 않습니다. 코드에서 직접 확인한 해석은 path와 symbol 증거를 붙여 학습자 결론으로 구분합니다.
- commit subject, SHA, importance, tags, Thread 순서는 변경하지 않습니다.

권장 확인 명령 예시는 다음과 같습니다. 실제 repository 상태와 작업 방식에 맞게 사용하되, 확인 대상 SHA는 바꾸지 않습니다.

```sh
git switch --detach <SHA>
git show --stat --oneline <SHA>
git diff <SHA>^ <SHA> -- <path>
git show <SHA>:<path>
```

## S/A/B/C별 학습 깊이

| Importance | 학습 깊이 |
| --- | --- |
| S | 프로젝트 핵심 architecture/invariant로 취급합니다. 직전 상태, failure 가능성, 핵심 결정, 실제 코드, ownership/lifecycle/state transition, 후속 fix/test까지 깊게 기록합니다. |
| A | 주요 subsystem·boundary·failure path·integration point를 설명할 수 있도록 핵심 코드와 설계 판단을 기록합니다. |
| B | thread 흐름에서 맡는 구현 역할과 필요한 코드·상태 변화를 확인합니다. S/A와 같은 분량을 강제하지 않습니다. |
| C | thread 이해에 필요한 맥락만 기록합니다. 이 프로젝트의 Development Threads에는 C commit이 포함되지 않습니다. |

## 실제 코드 삽입 기준

- 문서에는 해당 SHA에서 직접 확인한 최소 코드 조각만 삽입합니다.
- 코드 조각마다 SHA, file path, symbol 또는 함수 범위를 함께 적습니다.
- 구현 전체를 복사하지 않고 invariant, ordering, ownership transfer, failure branch를 증명하는 부분만 남깁니다.
- 변경 전/후 비교가 핵심이면 parent와 current SHA의 대응 조각을 나란히 기록합니다.
- line number는 tree나 formatter에 따라 변할 수 있으므로 path와 symbol을 기본 식별자로 사용합니다.
- 실제 코드를 확인하지 못한 항목은 추측으로 채우지 않고 “확인하지 못한 path/symbol과 이유”를 기록합니다.

## Test commit 학습 방법

각 test commit에서 다음을 분리해 기록합니다.

- 대상 production invariant
- 재현하는 failure 또는 boundary
- test technique: real process/socket, fake backend, injected operation, white-box setup, sanitizer 등
- 실제 통과하는 production code path
- test가 증명하는 것
- test가 증명하지 않는 것
- broad integration인지 deterministic regression인지
- 후속 변경에서 막는 회귀
- 실행 환경, 명령, 결과와 실패 transcript/log

테스트가 통과했다는 사실만 적지 않습니다. 어떤 production branch를 어떤 입력과 seam으로 통과시켰는지 설명할 수 있어야 완료입니다.

## 문서 완료 기준

- 모든 Thread의 commit을 source 순서대로 확인했습니다.
- 모든 기록이 해당 SHA 코드에 근거하며 final HEAD를 소급 사용하지 않았습니다.
- S/A/B/C 중요도에 맞게 학습 깊이를 구분했습니다.
- 각 중요한 commit에서 실제 path, symbol, state, caller/callee, failure/cleanup 근거를 남겼습니다.
- fix와 regression test를 하나의 원인-수정-검증 흐름으로 연결했습니다.
- invariant ledger에 도입, 강화, 부족함 발견, fix, regression test를 구분했습니다.
- Thread 최종 architecture 또는 execution flow를 자신의 코드 증거로 설명할 수 있습니다.
- 별도 프로젝트 재학습 없이 commit history에 근거해 설계 → 구현 → 실패 → 수정 → 검증 과정을 다시 설명할 수 있습니다.
===== END FILE: README.md =====

